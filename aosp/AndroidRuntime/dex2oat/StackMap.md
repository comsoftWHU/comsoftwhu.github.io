---
layout: default
title: StackMap：从“为何需要它”到“代码如何组织”
nav_order: 11
parent: dex2oat
author: Anonymous Committer
---

## 背景：编译后世界的“现场可解释性”

在 ART 的 AOT/JIT 编译路径中，Dex 字节码会被编译成 ARM64 等机器码。机器码运行时，栈和寄存器里混合承载了多种语义：整数、浮点数、对象引用（GC roots），以及被内联后的多层调用栈信息。运行时在以下关键点需要“可解释”当前现场：

* GC（需要精确根集合）
* 异常投递与栈回溯（需要 native PC ↔ dex PC 映射与内联链）
* deopt/调试（需要把 dex vregs 恢复回解释器语义）

StackMap/CodeInfo 是编译器为这些关键点提供的紧凑元数据载体：每个方法一个 `CodeInfo`，方法内多个 `StackMap` 条目对应多个关键点。

---


## StackMap目标：运行时需要提供的信息

围绕单个 compiled method，运行时至少需要支持：

1. **GC roots**：当前时刻哪些栈槽/寄存器是对象引用
2. **native PC → dex PC 映射**：异常、调试、栈回溯需要
3. **dex vregs 的位置/值**：deopt 需要恢复 vreg 语义
4. **内联信息**：一个 native PC 可能对应多层 Java 调用栈

ART 将这些能力打包在 `CodeInfo` 中，并以多张 bit-table（位表）组织数据，通过索引关联与去重压缩控制体积。

---

## 底层编码基建：BitMemoryReader/Writer 与 interleaved varints

`CodeInfo` 和 `BitTable` 的序列化并不是“按字节写结构体”，而是**按 bit 紧凑写入**。ART 在`art/libartbase/base/bit_memory_region.h` 为此提供了 `BitMemoryReader`/`BitMemoryWriter`：它们在一个连续 buffer 上维护 bit-level 游标，允许以任意位宽写入/读取整数。

在 bit-level 场景下，最常见的需求是把一组 `uint32_t` 以更小体积写进 buffer。ART 使用一种轻量的“varint（变长整数）”编码：每个整数先写一个固定宽度的前缀（这里是 4 bits），前缀要么直接携带小数值，要么表示“后面还跟着多少字节的 payload”。

### 1）单个 varint 的规则（consecutive varints）

* 固定前缀位宽：`kVarintBits = 4`
* 可直接 inline 的最大值：`kVarintMax = 11`
* 若 `value <= 11`：仅写 4 bits（值本身）
* 若 `value > 11`：4-bit 前缀写入 `kVarintMax + payload_bytes`，随后写入 payload（`payload_bytes * 8` bits），payload_bytes ∈ {1,2,3,4}

读取端对应 `ReadVarint()`：先读 4 bits 决定是否还需要读取额外的 payload。

```cpp
constexpr uint32_t kVarintBits = 4;
constexpr uint32_t kVarintMax = 11;

ALWAYS_INLINE uint32_t ReadVarint() {
  uint32_t x = ReadBits(kVarintBits);
  return (x <= kVarintMax) ? x : ReadBits((x - kVarintMax) * kBitsPerByte);
}
```

> 读取存在 ReadVarint（consecutive），而写入统一走 WriteInterleavedVarints

### 2）interleaved varints：固定字段数量时的“先前缀后 payload”

当有**固定数量 N** 的字段要写入（例如 `CodeInfo` header、`BitTable` header），如果按“consecutive varints”逐个写，会导致读取端频繁执行“读 4 bits → 可能再读 payload”的小碎步操作。

ART 使用 `ReadInterleavedVarints<N>() / WriteInterleavedVarints<N>()` 做优化：

* **先连续写入 N 个 4-bit 前缀**
* 再按同样顺序把所有 `>11` 的字段 payload 依次写出
* 读取端可以一次性读出 `N * 4` bits 的所有前缀（常见 N 不大，可以一次读进 64-bit），然后只为需要的字段读取 payload

```cpp
template<size_t N>
ALWAYS_INLINE std::array<uint32_t, N> ReadInterleavedVarints() {
  static_assert(N * kVarintBits <= sizeof(uint64_t) * kBitsPerByte, "N too big");
  std::array<uint32_t, N> values;
  uint64_t data = ReadBits<uint64_t>(N * kVarintBits);
  for (size_t i = 0; i < N; i++) {
    values[i] = BitFieldExtract(data, i * kVarintBits, kVarintBits);
  }
  for (size_t i = 0; i < N; i++) {
    if (UNLIKELY(values[i] > kVarintMax)) {
      values[i] = ReadBits((values[i] - kVarintMax) * kBitsPerByte);
    }
  }
  return values;
}

template<size_t N>
ALWAYS_INLINE void WriteInterleavedVarints(std::array<uint32_t, N> values) {
  // Write small values (or the number of bytes needed for the large values).
  for (uint32_t value : values) {
    if (value > kVarintMax) {
      WriteBits(kVarintMax + BitsToBytesRoundUp(MinimumBitsToStore(value)), kVarintBits);
    } else {
      WriteBits(value, kVarintBits);
    }
  }
  // Write large values.
  for (uint32_t value : values) {
    if (value > kVarintMax) {
      WriteBits(value, BitsToBytesRoundUp(MinimumBitsToStore(value)) * kBitsPerByte);
    }
  }
}
```

这就是 “interleaved varints” 的含义：**把所有字段的“长度/小值信息”集中在一起写**，payload 则集中放在后面，从而减少 bit-reader 的分支和细粒度读取次数。

---

## BitTable：按列位宽压缩的二维表（CodeInfo 的核心载体）

`CodeInfo` 内部的 `stack_maps_ / register_masks_ / stack_masks_ / ...` 等“表”，都建立在一个通用的数据结构之上：`BitTable`。

`BitTable` 的目标是：用尽可能少的 bit 存储一个 `uint32_t` 二维表，并提供 O(1) 的 (row, column) 访问。核心思路是**按列计算最小位宽**，然后把每个单元用该位宽写入 bit 流；表头存储行数与每列位宽。

### 1）二进制布局

一个 `BitTable` 的编码形式为：

1. **表头 header**（interleaved varints）

   * `num_rows`（行数）
   * `column_bits[c]`（每列位宽，c=0..kNumColumns-1）
2. **表体 table_data**（bit-packed）

   * 按行写入：对每个 row，按列写入 `column_bits[c]` bits

解码端在 `BitTableBase::Decode()` 里体现得很直接：先读 header，再读 `num_rows_ * NumRowBits()` 这么多 bit 作为表体 region。

```cpp
// Generic purpose table of uint32_t values, which are tightly packed at bit level.
// It has its own header with the number of rows and the bit-widths of all columns.
// The values are accessible by (row, column).  The value -1 is stored efficiently.
template<uint32_t kNumColumns>
class BitTableBase {
 public:
  static constexpr uint32_t kNoValue = std::numeric_limits<uint32_t>::max();  // == -1.
  static constexpr uint32_t kValueBias = kNoValue;  // Bias so that -1 is encoded as 0.

  BitTableBase() {}
  explicit BitTableBase(BitMemoryReader& reader) {
    Decode(reader);
  }

  ALWAYS_INLINE void Decode(BitMemoryReader& reader) {
    // Decode row count and column sizes from the table header.
    std::array<uint32_t, 1+kNumColumns> header = reader.ReadInterleavedVarints<1+kNumColumns>();
    num_rows_ = header[0];
    column_offset_[0] = 0;
    for (uint32_t i = 0; i < kNumColumns; i++) {
      size_t column_end = column_offset_[i] + header[i + 1];
      column_offset_[i + 1] = dchecked_integral_cast<uint16_t>(column_end);
    }

    // Record the region which contains the table data and skip past it.
    table_data_ = reader.ReadRegion(num_rows_ * NumRowBits());
  }

  ALWAYS_INLINE uint32_t Get(uint32_t row, uint32_t column = 0) const {
    DCHECK(table_data_.IsValid()) << "Table has not been loaded";
    DCHECK_LT(row, num_rows_);
    DCHECK_LT(column, kNumColumns);
    size_t offset = row * NumRowBits() + column_offset_[column];
    return table_data_.LoadBits(offset, NumColumnBits(column)) + kValueBias;
  }

  ALWAYS_INLINE BitMemoryRegion GetBitMemoryRegion(uint32_t row, uint32_t column = 0) const {
    DCHECK(table_data_.IsValid()) << "Table has not been loaded";
    DCHECK_LT(row, num_rows_);
    DCHECK_LT(column, kNumColumns);
    size_t offset = row * NumRowBits() + column_offset_[column];
    return table_data_.Subregion(offset, NumColumnBits(column));
  }

  uint32_t NumRows() const { return num_rows_; }

  uint32_t NumRowBits() const { return column_offset_[kNumColumns]; }

  constexpr uint32_t NumColumns() const { return kNumColumns; }

  uint32_t NumColumnBits(uint32_t column) const {
    return column_offset_[column + 1] - column_offset_[column];
  }

  size_t DataBitSize() const { return table_data_.size_in_bits(); }

  bool Equals(const BitTableBase& other) const {
    return num_rows_ == other.num_rows_ &&
        std::equal(column_offset_, column_offset_ + kNumColumns, other.column_offset_) &&
        BitMemoryRegion::Equals(table_data_, other.table_data_);
  }

 protected:
  BitMemoryRegion table_data_;
  uint32_t num_rows_ = 0;
  uint16_t column_offset_[kNumColumns + 1] = {};
};
```

这里有两个关键点：

* **表体不是字节对齐**：每列占用多少 bit 完全取决于该列最大值所需位宽。
* `kNoValue`（语义上的 “-1/不存在”）用 `kValueBias` 做偏置，使其能更省 bit：存储时写 `value - kValueBias`，读取时加回 `kValueBias`。

### 2）编码端如何决定每列位宽

编码端在 `BitTableBuilderBase::Measure()` 里扫描所有行，找每列的“最大位宽需求”。随后在 `Encode()` 写入 header，并按 row-major 写表体。

```cpp
  // Calculate the column bit widths based on the current data.
  void Measure(/*out*/ uint32_t* column_bits) const {
    uint32_t max_column_value[kNumColumns];
    std::fill_n(max_column_value, kNumColumns, 0);
    for (uint32_t r = 0; r < size(); r++) {
      for (uint32_t c = 0; c < kNumColumns; c++) {
        max_column_value[c] |= rows_[r][c] - kValueBias;
      }
    }
    for (uint32_t c = 0; c < kNumColumns; c++) {
      column_bits[c] = MinimumBitsToStore(max_column_value[c]);
    }
  }


  // Encode the stored data into a BitTable.
  template<typename Vector>
  void Encode(BitMemoryWriter<Vector>& out) const {
    size_t initial_bit_offset = out.NumberOfWrittenBits();

    // Write table header.
    std::array<uint32_t, 1 + kNumColumns> header;
    header[0] = size();
    uint32_t* column_bits = header.data() + 1;
    Measure(column_bits);
    out.WriteInterleavedVarints(header);

    // Write table data.
    for (uint32_t r = 0; r < size(); r++) {
      for (uint32_t c = 0; c < kNumColumns; c++) {
        out.WriteBits(rows_[r][c] - kValueBias, column_bits[c]);
      }
    }

    // Verify the written data.
    if (kIsDebugBuild) {
      BitTableBase<kNumColumns> table;
      BitMemoryReader reader(out.GetWrittenRegion().Subregion(initial_bit_offset));
      table.Decode(reader);
      DCHECK_EQ(size(), table.NumRows());
      for (uint32_t c = 0; c < kNumColumns; c++) {
        DCHECK_EQ(column_bits[c], table.NumColumnBits(c));
      }
      for (uint32_t r = 0; r < size(); r++) {
        for (uint32_t c = 0; c < kNumColumns; c++) {
          DCHECK_EQ(rows_[r][c], table.Get(r, c)) << " (" << r << ", " << c << ")";
        }
      }
    }
  }
```

### 3）BitTableBuilder 的去重（Dedup）

`CodeInfo` 体积控制的关键并不只靠位宽压缩，还靠 **“表行去重”**：例如很多 stack map 共享相同的 mask/inline 结构。

`BitTableBuilderBase::Dedup()` 为此维护了一个 `hash -> row index` 的 multimap；遇到相同数据时直接返回旧的 index，从而避免重复写入。

```cpp
  // Append given list of values and return the index of the first value.
  // If the exact same set of values was already added, return the old index.
  uint32_t Dedup(Entry* values, size_t count = 1) {
    FNVHash<MemoryRegion> hasher;
    uint32_t hash = hasher(MemoryRegion(values, sizeof(Entry) * count));

    // Check if we have already added identical set of values.
    auto range = dedup_.equal_range(hash);
    for (auto it = range.first; it != range.second; ++it) {
      uint32_t index = it->second;
      if (count <= size() - index &&
          std::equal(values,
                     values + count,
                     rows_.begin() + index,
                     [](const Entry& lhs, const Entry& rhs) {
                       return memcmp(&lhs, &rhs, sizeof(Entry)) == 0;
                     })) {
        return index;
      }
    }

    // Add the set of values and add the index to the dedup map.
    uint32_t index = size();
    rows_.insert(rows_.end(), values, values + count);
    dedup_.emplace(hash, index);
    return index;
  }
```

---

## 数据组织：CodeInfo 的“头 + 多表”布局

### 1）Header：固定字段（varint 编码）

`CodeInfo` 头部字段以 interleaved varints 编码写入，包含 flags、code size、frame size、spill masks、dex vreg 数量、bit-table 存在/去重标志等。运行时构造 `CodeInfo` 时会一次性读出这些 header。

```cpp

  // The CodeInfo starts with sequence of variable-length bit-encoded integers.
  // (Please see kVarintMax for more details about encoding).
  static constexpr size_t kNumHeaders = 7;
  uint32_t flags_ = 0;
  uint32_t code_size_ = 0;  // The size of native PC range in bytes.
  uint32_t packed_frame_size_ = 0;  // Frame size in kStackAlignment units.
  uint32_t core_spill_mask_ = 0;
  uint32_t fp_spill_mask_ = 0;
  uint32_t number_of_dex_registers_ = 0;
  uint32_t bit_table_flags_ = 0;

  // The encoded bit-tables follow the header.  Based on the above flags field,
  // bit-tables might be omitted or replaced by relative bit-offset if deduped.
  static constexpr size_t kNumBitTables = 8;
  BitTable<StackMap> stack_maps_;
  BitTable<RegisterMask> register_masks_;
  BitTable<StackMask> stack_masks_;
  BitTable<InlineInfo> inline_infos_;
  BitTable<MethodInfo> method_infos_;
  BitTable<DexRegisterMask> dex_register_masks_;
  BitTable<DexRegisterMapInfo> dex_register_maps_;
  BitTable<DexRegisterInfo> dex_register_catalog_;
```

### 2）Bit-tables：固定顺序枚举（8 张表）

`CodeInfo` 通过 `ForEachBitTableField()` 固定了 bit-table 的逻辑顺序（也是编码/解码时的顺序）：

* `stack_maps_`
* `register_masks_`
* `stack_masks_`
* `inline_infos_`
* `method_infos_`
* `dex_register_masks_`
* `dex_register_maps_`
* `dex_register_catalog_`

关键实现片段（节选）：

```cpp
  // Invokes the callback with index and member pointer of each BitTable field.
  template<typename Callback>
  ALWAYS_INLINE static void ForEachBitTableField(Callback callback) {
    size_t index = 0;
    callback(index++, &CodeInfo::stack_maps_);
    callback(index++, &CodeInfo::register_masks_);
    callback(index++, &CodeInfo::stack_masks_);
    callback(index++, &CodeInfo::inline_infos_);
    callback(index++, &CodeInfo::method_infos_);
    callback(index++, &CodeInfo::dex_register_masks_);
    callback(index++, &CodeInfo::dex_register_maps_);
    callback(index++, &CodeInfo::dex_register_catalog_);
    DCHECK_EQ(index, kNumBitTables);
  }
```

这套固定枚举顺序的意义在于：编码端与解码端可以用同一套遍历逻辑串行处理全部表；同时允许按需只保留部分表用于 GC 或栈回溯等快速路径（见后文）。

---

## 关键表：StackMap、Mask、InlineInfo、DexRegisterMap

### 1）StackMap：索引中枢

`StackMap` 行本身不存大块数据，而是存“索引列”：dex pc、packed native pc，以及指向 mask、inline、dex vreg map 的 index。运行时通过这些 index 再去对应表取数据。

运行时查找 `StackMap`（按 native PC）采用二分策略，并明确将 Catch 类 stack map 作为“末尾区域”处理，避免影响默认/OSR 查找。关键实现片段（节选）：

```cpp
StackMap CodeInfo::GetStackMapForNativePcOffset(uintptr_t pc, InstructionSet isa) const {
  uint32_t packed_pc = StackMap::PackNativePc(pc, isa);
  // Binary search.  All catch stack maps are stored separately at the end.
  auto it = std::partition_point(
      stack_maps_.begin(),
      stack_maps_.end(),
      [packed_pc](const StackMap& sm) {
        return sm.GetPackedNativePc() < packed_pc && sm.GetKind() != StackMap::Kind::Catch;
      });
  // Start at the lower bound and iterate over all stack maps with the given native pc.
  for (; it != stack_maps_.end() && (*it).GetNativePcOffset(isa) == pc; ++it) {
    StackMap::Kind kind = static_cast<StackMap::Kind>((*it).GetKind());
    if (kind == StackMap::Kind::Default || kind == StackMap::Kind::OSR) {
      return *it;
    }
  }
  return stack_maps_.GetInvalidRow();
}
```

### 2）GC root：RegisterMask 的 value+shift 压缩

寄存器 root mask 往往低位有效、高位大量为 0，因此用 `(value, shift)` 表示 `mask = value << shift` 更节省空间。编译期构建时会取最低有效位作为 shift。

编译期构建片段（节选）：

```cpp
  if (register_mask != 0) {
    uint32_t shift = LeastSignificantBit(register_mask);
    BitTableBuilder<RegisterMask>::Entry entry;
    entry[RegisterMask::kValue] = register_mask >> shift;
    entry[RegisterMask::kShift] = shift;
    current_stack_map_[StackMap::kRegisterMaskIndex] = register_masks_.Dedup(&entry);
  }
```

### 3）GC root：StackMask 的 bit 粒度（kFrameSlotSize字节一个槽）

栈 mask 与 vreg-in-stack 的编码都以“frame slot”为单位，slot 大小由常量kFrameSlotSize定义。对于 in-stack 的位置编码，packed value 会按 slot 大小进行除法压缩，并在解码时乘回。

关键实现片段（节选）：

```cpp
  static uint32_t PackValue(DexRegisterLocation::Kind kind, uint32_t value) {
    uint32_t packed_value = value;
    if (kind == DexRegisterLocation::Kind::kInStack) {
      DCHECK(IsAligned<kFrameSlotSize>(packed_value));
      packed_value /= kFrameSlotSize;
    }
    return packed_value;
  }
```

### 4）InlineInfo + MethodInfo：将“方法索引信息”拆表以利去重

内联信息需要记录每一层的 dex pc、方法标识等。实现上将方法相关信息拆到 `method_infos_` 中，并在 inline 表中仅存 `MethodInfoIndex`，以提高 dedup 的效率。编译期使用 `method_infos_.Dedup(...)` 生成 index。

关键实现片段（节选）：

```cpp
entry[InlineInfo::kMethodInfoIndex] = method_infos_.Dedup({dex_method_index, ...});
```

### 5）DexRegisterMap：delta 压缩（只记录“变化的 vreg”）

dex vreg 的位置（寄存器/栈/常量/none）变化频率通常远低于 stack map 的数量，因此编译期对 vreg map 做增量编码：

* `dex_register_masks_`：标记哪些 vreg 在此 stack map 有更新
* `dex_register_maps_`：对应更新项的“catalog index 序列”
* `dex_register_catalog_`：对 `(kind, value)` 去重后的全方法唯一条目表

编译期核心逻辑：若 vreg 位置变化，或距离上次定义超过上限（确保运行时回溯有界），则将其写入本次的 delta。 

关键实现片段（节选）：

```cpp
// Create delta-compressed dex register map based on the current list of DexRegisterLocations.
// All dex registers for a stack map are concatenated - inlined registers are just appended.
void StackMapStream::CreateDexRegisterMap() {
  // These are fields rather than local variables so that we can reuse the reserved memory.
  temp_dex_register_mask_.ClearAllBits();
  temp_dex_register_map_.clear();

  // Ensure that the arrays that hold previous state are big enough to be safely indexed below.
  if (previous_dex_registers_.size() < current_dex_registers_.size()) {
    previous_dex_registers_.resize(current_dex_registers_.size(), DexRegisterLocation::None());
    dex_register_timestamp_.resize(current_dex_registers_.size(), 0u);
  }

  // Set bit in the mask for each register that has been changed since the previous stack map.
  // Modified registers are stored in the catalogue and the catalogue index added to the list.
  for (size_t i = 0; i < current_dex_registers_.size(); i++) {
    DexRegisterLocation reg = current_dex_registers_[i];
    // Distance is difference between this index and the index of last modification.
    uint32_t distance = stack_maps_.size() - dex_register_timestamp_[i];
    if (previous_dex_registers_[i] != reg || distance > kMaxDexRegisterMapSearchDistance) {
      BitTableBuilder<DexRegisterInfo>::Entry entry;
      entry[DexRegisterInfo::kKind] = static_cast<uint32_t>(reg.GetKind());
      entry[DexRegisterInfo::kPackedValue] =
          DexRegisterInfo::PackValue(reg.GetKind(), reg.GetValue());
      uint32_t index = reg.IsLive() ? dex_register_catalog_.Dedup(&entry) : kNoValue;
      temp_dex_register_mask_.SetBit(i);
      temp_dex_register_map_.push_back({index});
      previous_dex_registers_[i] = reg;
      dex_register_timestamp_[i] = stack_maps_.size();
    }
  }

  // Set the mask and map for the current StackMap (which includes inlined registers).
  if (temp_dex_register_mask_.GetNumberOfBits() != 0) {
    current_stack_map_[StackMap::kDexRegisterMaskIndex] =
        dex_register_masks_.Dedup(temp_dex_register_mask_.GetRawStorage(),
                                  temp_dex_register_mask_.GetNumberOfBits());
  }
  if (!current_dex_registers_.empty()) {
    current_stack_map_[StackMap::kDexRegisterMapIndex] =
        dex_register_maps_.Dedup(temp_dex_register_map_.data(),
                                 temp_dex_register_map_.size());
  }

  if (kVerifyStackMaps) {
    size_t stack_map_index = stack_maps_.size();
    // We need to make copy of the current registers for later (when the check is run).
    auto expected_dex_registers = std::make_shared<dchecked_vector<DexRegisterLocation>>(
        current_dex_registers_.begin(), current_dex_registers_.end());
    dchecks_.emplace_back([=](const CodeInfo& code_info) {
      StackMap stack_map = code_info.GetStackMapAt(stack_map_index);
      uint32_t expected_reg = 0;
      for (DexRegisterLocation reg : code_info.GetDexRegisterMapOf(stack_map)) {
        CHECK_EQ((*expected_dex_registers)[expected_reg++], reg);
      }
      for (InlineInfo inline_info : code_info.GetInlineInfosOf(stack_map)) {
        DexRegisterMap map = code_info.GetInlineDexRegisterMapOf(stack_map, inline_info);
        for (DexRegisterLocation reg : map) {
          CHECK_EQ((*expected_dex_registers)[expected_reg++], reg);
        }
      }
      CHECK_EQ(expected_reg, expected_dex_registers->size());
    });
  }
}
```


---

## 运行时解码：如何查、如何还原、如何按需加载

### 1）CodeInfo 构造：读取 header + 逐表 decode，并支持“表级去重回跳”

运行时解码时，先读 header，再按固定顺序遍历各 bit-table。若某表被标记为 “deduped”，则不会从当前位置 decode，而是读取一个 varint 作为回跳距离，并在更早的 bit offset 上复用同一份表数据。

关键实现片段（节选）：

```cpp
template<typename DecodeCallback>
CodeInfo::CodeInfo(const uint8_t* data, size_t* num_read_bits, DecodeCallback callback) {
  BitMemoryReader reader(data);
  std::array<uint32_t, kNumHeaders> header = reader.ReadInterleavedVarints<kNumHeaders>();
  ForEachHeaderField([this, &header](size_t i, auto member_pointer) ALWAYS_INLINE {
    this->*member_pointer = header[i];
  });
  ForEachBitTableField([this, &reader, &callback](size_t i, auto member_pointer) ALWAYS_INLINE {
    auto& table = this->*member_pointer;
    if (LIKELY(HasBitTable(i))) {
      if (UNLIKELY(IsBitTableDeduped(i))) {
        ssize_t bit_offset = reader.NumberOfReadBits() - reader.ReadVarint();
        BitMemoryReader reader2(reader.data(), bit_offset);  // The offset is negative.
        table.Decode(reader2);
        callback(i, &table, reader2.GetReadRegion());
      } else {
        ssize_t bit_offset = reader.NumberOfReadBits();
        table.Decode(reader);
        callback(i, &table, reader.GetReadRegion().Subregion(bit_offset));
      }
    }
  });
  if (num_read_bits != nullptr) {
    *num_read_bits = reader.NumberOfReadBits();
  }
}
```


### 2）GetStackMapForNativePcOffset：二分定位 + 排除 Catch 区

该函数将 native pc pack 成 `packed_pc`，通过 `partition_point` 做二分定位，并跳过 Catch stack maps（Catch 单独放末尾）。之后在同一 native pc 范围内选择 Default/OSR 类型返回。

```cpp
StackMap CodeInfo::GetStackMapForNativePcOffset(uintptr_t pc, InstructionSet isa) const {
  uint32_t packed_pc = StackMap::PackNativePc(pc, isa);
  // Binary search.  All catch stack maps are stored separately at the end.
  auto it = std::partition_point(
      stack_maps_.begin(),
      stack_maps_.end(),
      [packed_pc](const StackMap& sm) {
        return sm.GetPackedNativePc() < packed_pc && sm.GetKind() != StackMap::Kind::Catch;
      });
  // Start at the lower bound and iterate over all stack maps with the given native pc.
  for (; it != stack_maps_.end() && (*it).GetNativePcOffset(isa) == pc; ++it) {
    StackMap::Kind kind = static_cast<StackMap::Kind>((*it).GetKind());
    if (kind == StackMap::Kind::Default || kind == StackMap::Kind::OSR) {
      return *it;
    }
  }
  return stack_maps_.GetInvalidRow();
}
```

### 3）DecodeDexRegisterMap：向后回溯合成完整 vreg map（有界）

运行时解码某个 stack map 的 vreg 状态时，并不要求该点完整存储所有 vreg，而是从目标 stack map 向前扫描：

* 每遇到一条 dex-register mask，就只填充尚未确定的 vreg
* 利用 `map_index += mask.PopCount(0, first)` 跳过不关注的前缀（处理内联拼接的寄存器区间）
* 若回溯仍未覆盖全部寄存器，则剩余位置默认为 `None`

同时用 `kMaxDexRegisterMapSearchDistance` 保证“不会无界回溯”。

关键实现片段（节选）：

```cpp
// Scan backward to determine dex register locations at given stack map.
// All registers for a stack map are combined - inlined registers are just appended,
// therefore 'first_dex_register' allows us to select a sub-range to decode.
void CodeInfo::DecodeDexRegisterMap(uint32_t stack_map_index,
                                    uint32_t first_dex_register,
                                    /*out*/ DexRegisterMap* map) const {
  // Count remaining work so we know when we have finished.
  uint32_t remaining_registers = map->size();

  // Keep scanning backwards and collect the most recent location of each register.
  for (int32_t s = stack_map_index; s >= 0 && remaining_registers != 0; s--) {
    StackMap stack_map = GetStackMapAt(s);
    DCHECK_LE(stack_map_index - s, kMaxDexRegisterMapSearchDistance) << "Unbounded search";

    // The mask specifies which registers where modified in this stack map.
    // NB: the mask can be shorter than expected if trailing zero bits were removed.
    uint32_t mask_index = stack_map.GetDexRegisterMaskIndex();
    if (mask_index == StackMap::kNoValue) {
      continue;  // Nothing changed at this stack map.
    }
    BitMemoryRegion mask = dex_register_masks_.GetBitMemoryRegion(mask_index);
    if (mask.size_in_bits() <= first_dex_register) {
      continue;  // Nothing changed after the first register we are interested in.
    }

    // The map stores one catalogue index per each modified register location.
    uint32_t map_index = stack_map.GetDexRegisterMapIndex();
    DCHECK_NE(map_index, StackMap::kNoValue);

    // Skip initial registers which we are not interested in (to get to inlined registers).
    map_index += mask.PopCount(0, first_dex_register);
    mask = mask.Subregion(first_dex_register, mask.size_in_bits() - first_dex_register);

    // Update registers that we see for first time (i.e. most recent value).
    DexRegisterLocation* regs = map->data();
    const uint32_t end = std::min<uint32_t>(map->size(), mask.size_in_bits());
    const size_t kNumBits = BitSizeOf<uint32_t>();
    for (uint32_t reg = 0; reg < end; reg += kNumBits) {
      // Process the mask in chunks of kNumBits for performance.
      uint32_t bits = mask.LoadBits(reg, std::min<uint32_t>(end - reg, kNumBits));
      while (bits != 0) {
        uint32_t bit = CTZ(bits);
        if (regs[reg + bit].GetKind() == DexRegisterLocation::Kind::kInvalid) {
          regs[reg + bit] = GetDexRegisterCatalogEntry(dex_register_maps_.Get(map_index));
          remaining_registers--;
        }
        map_index++;
        bits ^= 1u << bit;  // Clear the bit.
      }
    }
  }

  // Set any remaining registers to None (which is the default state at first stack map).
  if (remaining_registers != 0) {
    DexRegisterLocation* regs = map->data();
    for (uint32_t r = 0; r < map->size(); r++) {
      if (regs[r].GetKind() == DexRegisterLocation::Kind::kInvalid) {
        regs[r] = DexRegisterLocation::None();
      }
    }
  }
}
```

### 4）按需解码：GC 与栈回溯的轻量入口

为了避免不必要的解码开销，运行时提供了“只解码必要表”的入口：

* `DecodeGcMasksOnly()`：仅保留 stack_maps/register_masks/stack_masks
* `DecodeInlineInfoOnly()`：仅保留 stack_maps/inline_infos/method_infos（及 dex vreg 数）

这两者都先构造完整 `CodeInfo`，随后拷贝需要的字段到一个新对象，以便编译器做死代码消除，减少后续访问成本。 

```cpp
CodeInfo CodeInfo::DecodeGcMasksOnly(const OatQuickMethodHeader* header) {
  CodeInfo code_info(header->GetOptimizedCodeInfoPtr());
  CodeInfo copy;  // Copy to dead-code-eliminate all fields that we do not need.
  copy.stack_maps_ = code_info.stack_maps_;
  copy.register_masks_ = code_info.register_masks_;
  copy.stack_masks_ = code_info.stack_masks_;
  return copy;
}

CodeInfo CodeInfo::DecodeInlineInfoOnly(const OatQuickMethodHeader* header) {
  CodeInfo code_info(header->GetOptimizedCodeInfoPtr());
  CodeInfo copy;  // Copy to dead-code-eliminate all fields that we do not need.
  copy.number_of_dex_registers_ = code_info.number_of_dex_registers_;
  copy.stack_maps_ = code_info.stack_maps_;
  copy.inline_infos_ = code_info.inline_infos_;
  copy.method_infos_ = code_info.method_infos_;
  return copy;
}
```

---

## 编译期构建：StackMapStream 如何收集并编码

### 1）表构建器：编译期与运行时的“表顺序一致”

`StackMapStream` 内部也按 `CodeInfo::kNumBitTables` 的一致顺序维护各 `BitTableBuilder`，并提供 `ForEachBitTable()` 用于编码时串行写入。


### 2）BeginStackMapEntry：填 StackMap 行，并对 mask 做去重索引化

* `PackedNativePc` 由 `StackMap::PackNativePc()` 生成
* `RegisterMask` 通过 `(value, shift)` 编码后 `register_masks_.Dedup()` 得到 index
* `StackMask` 采用“延迟读取”：先暂存指针，直到 `EndMethod()` 才读取并写入 dedup 后的 index（允许编译器后续修改 bitvector）

```cpp
void StackMapStream::BeginStackMapEntry(
    uint32_t dex_pc,
    uint32_t native_pc_offset,
    uint32_t register_mask,
    BitVector* stack_mask,
    StackMap::Kind kind,
    bool needs_vreg_info,
    const std::vector<uint32_t>& dex_pc_list_for_catch_verification) {
  DCHECK(in_method_) << "Call BeginMethod first";
  DCHECK(!in_stack_map_) << "Mismatched Begin/End calls";
  in_stack_map_ = true;

  DCHECK_IMPLIES(!dex_pc_list_for_catch_verification.empty(), kind == StackMap::Kind::Catch);
  DCHECK_IMPLIES(!dex_pc_list_for_catch_verification.empty(), kIsDebugBuild);

  current_stack_map_ = BitTableBuilder<StackMap>::Entry();
  current_stack_map_[StackMap::kKind] = static_cast<uint32_t>(kind);
  current_stack_map_[StackMap::kPackedNativePc] =
      StackMap::PackNativePc(native_pc_offset, instruction_set_);
  current_stack_map_[StackMap::kDexPc] = dex_pc;
  if (stack_maps_.size() > 0) {
    // Check that non-catch stack maps are sorted by pc.
    // Catch stack maps are at the end and may be unordered.
    if (stack_maps_.back()[StackMap::kKind] == StackMap::Kind::Catch) {
      DCHECK(current_stack_map_[StackMap::kKind] == StackMap::Kind::Catch);
    } else if (current_stack_map_[StackMap::kKind] != StackMap::Kind::Catch) {
      DCHECK_LE(stack_maps_.back()[StackMap::kPackedNativePc],
                current_stack_map_[StackMap::kPackedNativePc]);
    }
  }
  if (register_mask != 0) {
    uint32_t shift = LeastSignificantBit(register_mask);
    BitTableBuilder<RegisterMask>::Entry entry;
    entry[RegisterMask::kValue] = register_mask >> shift;
    entry[RegisterMask::kShift] = shift;
    current_stack_map_[StackMap::kRegisterMaskIndex] = register_masks_.Dedup(&entry);
  }
  // The compiler assumes the bit vector will be read during PrepareForFillIn(),
  // and it might modify the data before that. Therefore, just store the pointer.
  // See ClearSpillSlotsFromLoopPhisInStackMap in code_generator.h.
  lazy_stack_masks_.push_back(stack_mask);
  current_inline_infos_.clear();
  current_dex_registers_.clear();
  expected_num_dex_registers_ = needs_vreg_info  ? num_dex_registers_ : 0u;

  if (kVerifyStackMaps) {
    size_t stack_map_index = stack_maps_.size();
    // Create lambda method, which will be executed at the very end to verify data.
    // Parameters and local variables will be captured(stored) by the lambda "[=]".
    dchecks_.emplace_back([=, this](const CodeInfo& code_info) {
      // The `native_pc_offset` may have been overridden using `SetStackMapNativePcOffset(.)`.
      uint32_t final_native_pc_offset = GetStackMapNativePcOffset(stack_map_index);
      if (kind == StackMap::Kind::Default || kind == StackMap::Kind::OSR) {
        StackMap stack_map = code_info.GetStackMapForNativePcOffset(final_native_pc_offset,
                                                                    instruction_set_);
        CHECK_EQ(stack_map.Row(), stack_map_index);
      } else if (kind == StackMap::Kind::Catch) {
        StackMap stack_map = code_info.GetCatchStackMapForDexPc(
            ArrayRef<const uint32_t>(dex_pc_list_for_catch_verification));
        CHECK_EQ(stack_map.Row(), stack_map_index);
      }
      StackMap stack_map = code_info.GetStackMapAt(stack_map_index);
      CHECK_EQ(stack_map.GetNativePcOffset(instruction_set_), final_native_pc_offset);
      CHECK_EQ(stack_map.GetKind(), static_cast<uint32_t>(kind));
      CHECK_EQ(stack_map.GetDexPc(), dex_pc);
      CHECK_EQ(code_info.GetRegisterMaskOf(stack_map), register_mask);
      BitMemoryRegion seen_stack_mask = code_info.GetStackMaskOf(stack_map);
      CHECK_GE(seen_stack_mask.size_in_bits(), stack_mask ? stack_mask->GetNumberOfBits() : 0);
      for (size_t b = 0; b < seen_stack_mask.size_in_bits(); b++) {
        CHECK_EQ(seen_stack_mask.LoadBit(b), stack_mask != nullptr && stack_mask->IsBitSet(b));
      }
    });
  }
}
```

### 3）CreateDexRegisterMap：生成 delta-compressed vreg map（三表联动）

该函数完成：

* 对变化 vreg 置位 `temp_dex_register_mask_`
* 将变化项的 catalog index 追加到 `temp_dex_register_map_`
* 写入 `DexRegisterMaskIndex` 与 `DexRegisterMapIndex`（均走 Dedup）

其中“距离上次定义超过上限也要强制写一条”的策略，是为了匹配运行时的有界回溯实现。

```cpp
// Create delta-compressed dex register map based on the current list of DexRegisterLocations.
// All dex registers for a stack map are concatenated - inlined registers are just appended.
void StackMapStream::CreateDexRegisterMap() {
  // These are fields rather than local variables so that we can reuse the reserved memory.
  temp_dex_register_mask_.ClearAllBits();
  temp_dex_register_map_.clear();

  // Ensure that the arrays that hold previous state are big enough to be safely indexed below.
  if (previous_dex_registers_.size() < current_dex_registers_.size()) {
    previous_dex_registers_.resize(current_dex_registers_.size(), DexRegisterLocation::None());
    dex_register_timestamp_.resize(current_dex_registers_.size(), 0u);
  }

  // Set bit in the mask for each register that has been changed since the previous stack map.
  // Modified registers are stored in the catalogue and the catalogue index added to the list.
  for (size_t i = 0; i < current_dex_registers_.size(); i++) {
    DexRegisterLocation reg = current_dex_registers_[i];
    // Distance is difference between this index and the index of last modification.
    uint32_t distance = stack_maps_.size() - dex_register_timestamp_[i];
    if (previous_dex_registers_[i] != reg || distance > kMaxDexRegisterMapSearchDistance) {
      BitTableBuilder<DexRegisterInfo>::Entry entry;
      entry[DexRegisterInfo::kKind] = static_cast<uint32_t>(reg.GetKind());
      entry[DexRegisterInfo::kPackedValue] =
          DexRegisterInfo::PackValue(reg.GetKind(), reg.GetValue());
      uint32_t index = reg.IsLive() ? dex_register_catalog_.Dedup(&entry) : kNoValue;
      temp_dex_register_mask_.SetBit(i);
      temp_dex_register_map_.push_back({index});
      previous_dex_registers_[i] = reg;
      dex_register_timestamp_[i] = stack_maps_.size();
    }
  }

  // Set the mask and map for the current StackMap (which includes inlined registers).
  if (temp_dex_register_mask_.GetNumberOfBits() != 0) {
    current_stack_map_[StackMap::kDexRegisterMaskIndex] =
        dex_register_masks_.Dedup(temp_dex_register_mask_.GetRawStorage(),
                                  temp_dex_register_mask_.GetNumberOfBits());
  }
  if (!current_dex_registers_.empty()) {
    current_stack_map_[StackMap::kDexRegisterMapIndex] =
        dex_register_maps_.Dedup(temp_dex_register_map_.data(),
                                 temp_dex_register_map_.size());
  }

  if (kVerifyStackMaps) {
    size_t stack_map_index = stack_maps_.size();
    // We need to make copy of the current registers for later (when the check is run).
    auto expected_dex_registers = std::make_shared<dchecked_vector<DexRegisterLocation>>(
        current_dex_registers_.begin(), current_dex_registers_.end());
    dchecks_.emplace_back([=](const CodeInfo& code_info) {
      StackMap stack_map = code_info.GetStackMapAt(stack_map_index);
      uint32_t expected_reg = 0;
      for (DexRegisterLocation reg : code_info.GetDexRegisterMapOf(stack_map)) {
        CHECK_EQ((*expected_dex_registers)[expected_reg++], reg);
      }
      for (InlineInfo inline_info : code_info.GetInlineInfosOf(stack_map)) {
        DexRegisterMap map = code_info.GetInlineDexRegisterMapOf(stack_map, inline_info);
        for (DexRegisterLocation reg : map) {
          CHECK_EQ((*expected_dex_registers)[expected_reg++], reg);
        }
      }
      CHECK_EQ(expected_reg, expected_dex_registers->size());
    });
  }
}
```

### 4）Encode：写 header，再按顺序编码非空 bit-table

编码过程：

1. 汇总 `flags`（是否有 inline、baseline/debuggable 等）
2. 汇总 `bit_table_flags`（哪些表非空）
3. `WriteInterleavedVarints` 写入 header
4. 依次对每张非空表 `Encode(out)`

```cpp

ScopedArenaVector<uint8_t> StackMapStream::Encode() {
  DCHECK(in_stack_map_ == false) << "Mismatched Begin/End calls";
  DCHECK(in_inline_info_ == false) << "Mismatched Begin/End calls";

  uint32_t flags = 0;
  flags |= (inline_infos_.size() > 0) ? CodeInfo::kHasInlineInfo : 0;
  flags |= baseline_ ? CodeInfo::kIsBaseline : 0;
  flags |= debuggable_ ? CodeInfo::kIsDebuggable : 0;
  flags |= has_should_deoptimize_flag_ ? CodeInfo::kHasShouldDeoptimizeFlag : 0;

  uint32_t bit_table_flags = 0;
  ForEachBitTable([&bit_table_flags](size_t i, auto bit_table) {
    if (bit_table->size() != 0) {  // Record which bit-tables are stored.
      bit_table_flags |= 1 << i;
    }
  });

  ScopedArenaVector<uint8_t> buffer(allocator_->Adapter(kArenaAllocStackMapStream));
  BitMemoryWriter<ScopedArenaVector<uint8_t>> out(&buffer);
  out.WriteInterleavedVarints(std::array<uint32_t, CodeInfo::kNumHeaders>{
    flags,
    code_size_,
    packed_frame_size_,
    core_spill_mask_,
    fp_spill_mask_,
    num_dex_registers_,
    bit_table_flags,
  });
  ForEachBitTable([&out](size_t, auto bit_table) {
    if (bit_table->size() != 0) {  // Skip empty bit-tables.
      bit_table->Encode(out);
    }
  });

  // Verify that we can load the CodeInfo and check some essentials.
  size_t number_of_read_bits;
  CodeInfo code_info(buffer.data(), &number_of_read_bits);
  CHECK_EQ(number_of_read_bits, out.NumberOfWrittenBits());
  CHECK_EQ(code_info.GetNumberOfStackMaps(), stack_maps_.size());
  CHECK_EQ(CodeInfo::HasInlineInfo(buffer.data()), inline_infos_.size() > 0);
  CHECK_EQ(CodeInfo::IsBaseline(buffer.data()), baseline_);
  CHECK_EQ(CodeInfo::IsDebuggable(buffer.data()), debuggable_);
  CHECK_EQ(CodeInfo::HasShouldDeoptimizeFlag(buffer.data()), has_should_deoptimize_flag_);

  // Verify all written data (usually only in debug builds).
  if (kVerifyStackMaps) {
    for (const auto& dcheck : dchecks_) {
      dcheck(code_info);
    }
  }

  return buffer;
}
```

---

## 总结：为何它能“既可用又不臃肿”

* **索引化拆表**：StackMap 行只存索引，数据分散在多张表中
* **方法内去重**：BitTableBuilder 的 `Dedup()` 广泛用于 mask、inline、catalog 等
* **delta 压缩**：dex vreg map 只记录变化项，并强制上限保证运行时回溯有界
* **按需解码**：GC 与栈回溯走不同的轻量入口，避免解码无关表
* **表级去重回跳**：运行时解码支持 “deduped table 复用先前表”的回跳机制，为跨 CodeInfo 的进一步压缩留出空间

以上逻辑共同确保：在保持运行时可恢复性的前提下，StackMap/CodeInfo 的体积与解码成本可控。