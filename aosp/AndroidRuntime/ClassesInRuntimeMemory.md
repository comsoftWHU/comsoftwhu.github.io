---
layout: default
title: Runtime在内存里怎么存储类信息
nav_order: 3
parent: AndroidRuntime
grand_parent: AOSP
author: Anonymous Committer
---

首先来看一下在内存中构造出相关数据结构之前，在Dex文件中是怎么定义一个类的。

## Dex文件对类信息的组织

### 核心部件池

DEX 文件首先定义了几个大的“池”，里面存放了所有类都会用到的基础部件，实现了最大程度的数据复用。

- string_ids (字符串池): 存放了文件中用到的所有字符串，例如类名 ("java.lang.String")、字段名 ("value")、方法名 ("equals")、源文件名等。每个字符串只存一次。

- type_ids (类型池): 存放了所有类型描述符，如 "Ljava/lang/String;" (类)、`"[I"` (int数组)、"V" (void)。它本身不存字符串，而是存一个指向 string_ids 的索引。

- proto_ids (方法原型池): 存放了所有方法的签名（即参数列表和返回值类型）。例如，boolean equals(Object) 的原型就是 (Ljava/lang/Object;)Z。它通过指向 type_ids 的索引来描述每个类型。

- field_ids (字段ID池): 定义了每一个字段的唯一标识。它由三部分索引组成：

  - 指向 type_ids 的索引，表示该字段所属的类。

  - 指向 type_ids 的索引，表示该字段的类型。

  - 指向 string_ids 的索引，表示该字段的名字。

- method_ids (方法ID池): 定义了每一个方法的唯一标识。它也由三部分索引组成：

  - 指向 type_ids 的索引，表示该方法所属的类。

  - 指向 proto_ids 的索引，表示该方法的签名（参数和返回值）。

  - 指向 string_ids 的索引，表示该方法的名字。

### 类的定义

DEX 文件中有一个 class_defs 列表，其中每个 class_def_item 就代表一个在该 DEX 文件中定义的类。

一个 class_def_item 包含以下关键信息（大部分是索引）：

- class_idx: 指向 type_ids 的索引，说明“我是哪个类”（例如，Lcom/example/MyClass;）。

- access_flags: 访问标志，如 public, final, interface 等。

- superclass_idx: 指向 type_ids 的索引，说明“我的父类是谁”。

- interfaces_off: 一个偏移量，指向一个列表，该列表包含了这个类实现的所有接口（每个接口也是一个指向 type_ids 的索引）。

- source_file_idx: 指向 string_ids 的索引，说明“我的源文件名是什么”。

- annotations_off: 一个偏移量，指向这个类的注解信息。

- class_data_off: 指向了真正定义类成员（字段和方法）的数据区。

- static_values_off: 一个偏移量，指向静态字段的初始值列表。

### 类的成员列表

class_def_item 中的 class_data_off 指向一个 class_data_item。这里才是真正“组装”类的地方。一个 class_data_item 的结构如下：

- 头信息
  - static_fields_size: 静态字段的数量。
  
  - instance_fields_size: 实例字段的数量。
  
  - direct_methods_size: 直接方法（私有方法、静态方法、构造方法）的数  量。
  
  - virtual_methods_size: 虚方法（除了static final 构造函数所有其他方  法）的数量。

- 成员列表

  - static_fields 列表: 每一项都包含一个指向 field_ids 的索引以及访问标志

  - instance_fields 列表: 同上。

  - direct_methods 列表: 每一项都包含一个指向 method_ids 的索引（指明是哪个方法）以及访问标志，加上一个偏移量，指向该方法的实际字节码 (code_item)。

  - virtual_methods 列表: 同上。

## DexCache数据结构

DexCache 是与单个 DexFile 紧密绑定的一个运行时缓存。缓存已经从 .dex 文件中**解析（Resolve）**过的字符串、类型、字段和方法。它的存在是为了避免重复解析 .dex 文件中那些基于索引的定义，从而大幅提升性能。

```java
final class DexCache {
    private ClassLoader classLoader;
    private String location;
    /** Holds C pointer to dexFile. */
    @UnsupportedAppUsage
    private long dexFile;
    private long resolvedCallSites;
    private long resolvedFields;
    private long resolvedFieldsArray;
    private long resolvedMethodTypes;
    private long resolvedMethodTypesArray;
    private long resolvedMethods;
    private long resolvedMethodsArray;

    private long resolvedTypes;
    private long resolvedTypesArray;
    private long strings;
    private long stringsArray;
    private DexCache() {}
}
```

对应的C++ mirror定义

```cpp
// C++ mirror of java.lang.DexCache.
class MANAGED DexCache final : public Object {
  // ......
  HeapReference<ClassLoader> class_loader_;
  HeapReference<String> location_;

  uint64_t dex_file_;                     // const DexFile*
                                          //
  uint64_t resolved_call_sites_;          // Array of call sites
  uint64_t resolved_fields_;              // NativeDexCacheArray holding ArtField's
  uint64_t resolved_fields_array_;        // Array of ArtField's.
  uint64_t resolved_method_types_;        // DexCacheArray holding mirror::MethodType's
  uint64_t resolved_method_types_array_;  // Array of mirror::MethodType's
  uint64_t resolved_methods_;             // NativeDexCacheArray holding ArtMethod's
  uint64_t resolved_methods_array_;       // Array of ArtMethod's
  uint64_t resolved_types_;               // DexCacheArray holding mirror::Class's
  uint64_t resolved_types_array_;         // Array of resolved types.
  uint64_t strings_;                      // DexCacheArray holding mirror::String's
  uint64_t strings_array_;                // Array of String's.
};
```

这里的uint64_t类型实际上都是存储原生指针

resolved_methods_ 和 resolved_methods_array_两个字段缓存已解析的方法。当 ART 需要调用一个在 .dex 文件中索引为 i 的方法时，它会首先检查这个缓存。如果缓存中已有对应的 ArtMethod*，就直接使用；如果没有，ART 会解析方法，创建 ArtMethod 对象，然后将其指针存入缓存中，供下次使用。

**为什么有`*`和`*array_`2个字段呢？**

在commit message 中提到

如果需要缓存的条目数量很多，那么NativeDexCacheArray的管理优势就能体现出来。ART 仍会使用旧的、更复杂的结构，并将指针存放在原始字段中（例如 resolved_methods_）。

如果一个 .dex 文件需要缓存的条目数量很少（低于某个设定的阈值），ART 就会选择使用这个轻便的数组。它会分配一个纯数组，并将指针存放在新增的 _array_ 字段中（例如 resolved_methods_array_）。也意味着 nterp 的汇编代码可以用几条最基础、最快的机器指令来直接计算出缓存项的地址并读取它。

在旧版本中，DexCache 对每种需要缓存的实体（方法、字段、类型等）都只用一种缓存结构 NativeDexCacheArray。

### DexCachePair / NativeDexCachePair

```cpp
template <typename T> struct alignas(8) DexCachePair {
    GcRoot<T> object; // 缓存的对象
    uint32_t index;   // 原始的DEX文件索引
};
```

DexCachePairArray 利用 DexCachePair 实现了一个哈希表，用于解决“用小空间缓存大范围索引”的问题。

当要缓存一个索引为 idx 的条目时，它并不直接存在数组的第 idx 位。相反，它通过取模运算 slot = idx % size 计算出一个“槽位”。

因为多个不同的 idx 可能映射到同一个 slot（哈希冲突），所以每次查找时，除了计算 slot，还必须进行一次关键的比较：if (idx != a_pair.index)。只有当要查找的 idx 和槽位中存储的 index 完全一致时，才证明找到了正确的条目。

访问开销稍高。每次访问都需要一次取模运算和一次索引比较，比直接访问数组要慢一些。

### GcRootArray / NativeArray

```cpp
template <typename T> class GcRootArray {
private:
    Atomic<GcRoot<T>> entries_[0]; // 变长数组技巧
};
```

当要查找索引为 idx 的条目时，它直接访问数组的第 idx 个元素 `entries_[idx]`。访问速度极快、可能浪费内存。

### DEFINE_DUAL_CACHE决策使用哪个方案

```cpp
// NOLINTBEGIN(bugprone-macro-parentheses)
#define DEFINE_ARRAY(name, array_kind, getter_setter, type, ids, alloc_kind) \
  template<VerifyObjectFlags kVerifyFlags = kDefaultVerifyFlags> \
  array_kind* Get ##getter_setter() \
      ALWAYS_INLINE \
      REQUIRES_SHARED(Locks::mutator_lock_) { \
    return GetFieldPtr<array_kind*, kVerifyFlags>(getter_setter ##Offset()); \
  } \
  void Set ##getter_setter(array_kind* value) \
      REQUIRES_SHARED(Locks::mutator_lock_) { \
    SetFieldPtr<false>(getter_setter ##Offset(), value); \
  } \
  static constexpr MemberOffset getter_setter ##Offset() { \
    return OFFSET_OF_OBJECT_MEMBER(DexCache, name); \
  } \
  array_kind* Allocate ##getter_setter(bool startup = false) \
      REQUIRES_SHARED(Locks::mutator_lock_) { \
    return reinterpret_cast<array_kind*>(AllocArray<type>( \
        getter_setter ##Offset(), GetDexFile()->ids(), alloc_kind, startup)); \
  } \
  template<VerifyObjectFlags kVerifyFlags = kDefaultVerifyFlags> \
  size_t Num ##getter_setter() REQUIRES_SHARED(Locks::mutator_lock_) { \
    return Get ##getter_setter() == nullptr ? 0u : GetDexFile()->ids(); \
  } \

#define DEFINE_PAIR_ARRAY(name, pair_kind, getter_setter, type, size, alloc_kind) \
  template<VerifyObjectFlags kVerifyFlags = kDefaultVerifyFlags> \
  pair_kind ##Array<type, size>* Get ##getter_setter() \
      ALWAYS_INLINE \
      REQUIRES_SHARED(Locks::mutator_lock_) { \
    return GetFieldPtr<pair_kind ##Array<type, size>*, kVerifyFlags>(getter_setter ##Offset()); \
  } \
  void Set ##getter_setter(pair_kind ##Array<type, size>* value) \
      REQUIRES_SHARED(Locks::mutator_lock_) { \
    SetFieldPtr<false>(getter_setter ##Offset(), value); \
  } \
  static constexpr MemberOffset getter_setter ##Offset() { \
    return OFFSET_OF_OBJECT_MEMBER(DexCache, name); \
  } \
  pair_kind ##Array<type, size>* Allocate ##getter_setter() \
      REQUIRES_SHARED(Locks::mutator_lock_) { \
    return reinterpret_cast<pair_kind ##Array<type, size>*>( \
        AllocArray<std::atomic<pair_kind<type>>>( \
            getter_setter ##Offset(), size, alloc_kind)); \
  } \
  template<VerifyObjectFlags kVerifyFlags = kDefaultVerifyFlags> \
  size_t Num ##getter_setter() REQUIRES_SHARED(Locks::mutator_lock_) { \
    return Get ##getter_setter() == nullptr ? 0u : size; \
  } \

#define DEFINE_DUAL_CACHE( \
    name, pair_kind, getter_setter, type, pair_size, alloc_pair_kind, \
    array_kind, component_type, ids, alloc_array_kind) \
  DEFINE_PAIR_ARRAY( \
      name, pair_kind, getter_setter, type, pair_size, alloc_pair_kind) \
  DEFINE_ARRAY( \
      name ##array_, array_kind, getter_setter ##Array, component_type, ids, alloc_array_kind) \
  type* Get ##getter_setter ##Entry(uint32_t index) REQUIRES_SHARED(Locks::mutator_lock_) { \
    DCHECK_LT(index, GetDexFile()->ids()); \
    auto* array = Get ##getter_setter ##Array(); \
    if (array != nullptr) { \
      return array->Get(index); \
    } \
    auto* pairs = Get ##getter_setter(); \
    if (pairs != nullptr) { \
      return pairs->Get(index); \
    } \
    return nullptr; \
  } \
  void Set ##getter_setter ##Entry(uint32_t index, type* resolved) \
      REQUIRES_SHARED(Locks::mutator_lock_) { \
    DCHECK_LT(index, GetDexFile()->ids()); \
    auto* array = Get ##getter_setter ##Array(); \
    if (array != nullptr) { \
      array->Set(index, resolved); \
    } else { \
      auto* pairs = Get ##getter_setter(); \
      if (pairs == nullptr) { \
        bool should_allocate_full_array = ShouldAllocateFullArray(GetDexFile()->ids(), pair_size); \
        if (ShouldAllocateFullArrayAtStartup() || should_allocate_full_array) { \
          array = Allocate ##getter_setter ##Array(!should_allocate_full_array); \
          array->Set(index, resolved); \
        } else { \
          pairs = Allocate ##getter_setter(); \
          pairs->Set(index, resolved); \
        } \
      } else { \
        pairs->Set(index, resolved); \
      } \
    } \
  } \
  void Unlink ##getter_setter ##ArrayIfStartup() \
      REQUIRES_SHARED(Locks::mutator_lock_) { \
    if (!ShouldAllocateFullArray(GetDexFile()->ids(), pair_size)) { \
      Set ##getter_setter ##Array(nullptr) ; \
    } \
  }

  DEFINE_ARRAY(resolved_call_sites_,
               GcRootArray<CallSite>,
               ResolvedCallSites,
               GcRoot<CallSite>,
               NumCallSiteIds,
               LinearAllocKind::kGCRootArray)

  DEFINE_DUAL_CACHE(resolved_fields_,
                    NativeDexCachePair,
                    ResolvedFields,
                    ArtField,
                    kDexCacheFieldCacheSize,
                    LinearAllocKind::kNoGCRoots,
                    NativeArray<ArtField>,
                    ArtField*,
                    NumFieldIds,
                    LinearAllocKind::kNoGCRoots)

  DEFINE_DUAL_CACHE(resolved_method_types_,
                    DexCachePair,
                    ResolvedMethodTypes,
                    mirror::MethodType,
                    kDexCacheMethodTypeCacheSize,
                    LinearAllocKind::kDexCacheArray,
                    GcRootArray<mirror::MethodType>,
                    GcRoot<mirror::MethodType>,
                    NumProtoIds,
                    LinearAllocKind::kGCRootArray);

  DEFINE_DUAL_CACHE(resolved_methods_,
                    NativeDexCachePair,
                    ResolvedMethods,
                    ArtMethod,
                    kDexCacheMethodCacheSize,
                    LinearAllocKind::kNoGCRoots,
                    NativeArray<ArtMethod>,
                    ArtMethod*,
                    NumMethodIds,
                    LinearAllocKind::kNoGCRoots)

  DEFINE_DUAL_CACHE(resolved_types_,
                    DexCachePair,
                    ResolvedTypes,
                    mirror::Class,
                    kDexCacheTypeCacheSize,
                    LinearAllocKind::kDexCacheArray,
                    GcRootArray<mirror::Class>,
                    GcRoot<mirror::Class>,
                    NumTypeIds,
                    LinearAllocKind::kGCRootArray);

  DEFINE_DUAL_CACHE(strings_,
                    DexCachePair,
                    Strings,
                    mirror::String,
                    kDexCacheStringCacheSize,
                    LinearAllocKind::kDexCacheArray,
                    GcRootArray<mirror::String>,
                    GcRoot<mirror::String>,
                    NumStringIds,
                    LinearAllocKind::kGCRootArray);
```

宏展开后的代码长成这样

```cpp
//======= 由 DEFINE_PAIR_ARRAY(resolved_fields_, ...) 展开而来 =======
// 
// 这一部分定义了哈希表方案的访问和分配函数。
//
// 哈希表的 size 由 kDexCache*CacheSize确定，默认都是1024
// template<...> pair_kind##Array<type, size>* Get##getter_setter()
template<VerifyObjectFlags kVerifyFlags = kDefaultVerifyFlags>
NativeDexCachePairArray<ArtField, 1024>* GetResolvedFields()
    ALWAYS_INLINE
    REQUIRES_SHARED(Locks::mutator_lock_) {
  return GetFieldPtr<NativeDexCachePairArray<ArtField, 1024>*, kVerifyFlags>(ResolvedFieldsOffset());
}

// void Set##getter_setter(pair_kind##Array<type, size>* value)
void SetResolvedFields(NativeDexCachePairArray<ArtField, 1024>* value)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  SetFieldPtr<false>(ResolvedFieldsOffset(), value);
}

// static constexpr MemberOffset getter_setter##Offset()
static constexpr MemberOffset ResolvedFieldsOffset() {
  return OFFSET_OF_OBJECT_MEMBER(DexCache, resolved_fields_);
}

// pair_kind##Array<type, size>* Allocate##getter_setter()
NativeDexCachePairArray<ArtField, 1024>* AllocateResolvedFields()
    REQUIRES_SHARED(Locks::mutator_lock_) {
  return reinterpret_cast<NativeDexCachePairArray<ArtField, 1024>*>(
      AllocArray<std::atomic<NativeDexCachePair<ArtField>>>(
          ResolvedFieldsOffset(), 1024, LinearAllocKind::kNoGCRoots));
}

// template<...> size_t Num##getter_setter()
template<VerifyObjectFlags kVerifyFlags = kDefaultVerifyFlags>
size_t NumResolvedFields() REQUIRES_SHARED(Locks::mutator_lock_) {
  return GetResolvedFields() == nullptr ? 0u : 1024;
}

//======= 由 DEFINE_ARRAY(resolved_fields_array_, ...) 展开而来 =======
//
// 这一部分定义了“文件柜”（纯数组）方案的访问和分配函数。
//

// template<...> array_kind* Get##getter_setter()
template<VerifyObjectFlags kVerifyFlags = kDefaultVerifyFlags>
NativeArray<ArtField>* GetResolvedFieldsArray()
    ALWAYS_INLINE
    REQUIRES_SHARED(Locks::mutator_lock_) {
  return GetFieldPtr<NativeArray<ArtField>*, kVerifyFlags>(ResolvedFieldsArrayOffset());
}

// void Set##getter_setter(array_kind* value)
void SetResolvedFieldsArray(NativeArray<ArtField>* value)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  SetFieldPtr<false>(ResolvedFieldsArrayOffset(), value);
}

// static constexpr MemberOffset getter_setter##Offset()
static constexpr MemberOffset ResolvedFieldsArrayOffset() {
  return OFFSET_OF_OBJECT_MEMBER(DexCache, resolved_fields_array_);
}

// array_kind* Allocate##getter_setter(bool startup = false)
NativeArray<ArtField>* AllocateResolvedFieldsArray(bool startup = false)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  return reinterpret_cast<NativeArray<ArtField>*>(AllocArray<ArtField*>(
      ResolvedFieldsArrayOffset(), GetDexFile()->NumFieldIds(), LinearAllocKind::kNoGCRoots, startup));
}

// template<...> size_t Num##getter_setter()
template<VerifyObjectFlags kVerifyFlags = kDefaultVerifyFlags>
size_t NumResolvedFieldsArray() REQUIRES_SHARED(Locks::mutator_lock_) {
  return GetResolvedFieldsArray() == nullptr ? 0u : GetDexFile()->NumFieldIds();
}

//======= 由 DEFINE_DUAL_CACHE 直接定义的函数展开而来 =======
//
// 这一部分提供了统一的 Get/Set 接口，内部包含了智能决策逻辑。
//

// type* Get##getter_setter##Entry(uint32_t index)
ArtField* GetResolvedFieldsEntry(uint32_t index) REQUIRES_SHARED(Locks::mutator_lock_) {
  DCHECK_LT(index, GetDexFile()->NumFieldIds());
  // 优先使用“纯数组”方案
  auto* array = GetResolvedFieldsArray();
  if (array != nullptr) {
    return array->Get(index);
  }
  // 否则，使用“哈希表”方案
  auto* pairs = GetResolvedFields();
  if (pairs != nullptr) {
    return pairs->Get(index);
  }
  return nullptr;
}

// void Set##getter_setter##Entry(uint32_t index, type* resolved)
void SetResolvedFieldsEntry(uint32_t index, ArtField* resolved)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  DCHECK_LT(index, GetDexFile()->NumFieldIds());
  // 优先写入“纯数组”
  auto* array = GetResolvedFieldsArray();
  if (array != nullptr) {
    array->Set(index, resolved);
  } else {
    // 如果“纯数组”不存在，再检查“哈希表”
    auto* pairs = GetResolvedFields();
    if (pairs == nullptr) {
      // 如果两者都不存在（首次写入），则进行决策
      bool should_allocate_full_array = ShouldAllocateFullArray(GetDexFile()->NumFieldIds(), 1024);
      if (ShouldAllocateFullArrayAtStartup() || should_allocate_full_array) {
        // 决策结果：使用“纯数组”
        array = AllocateResolvedFieldsArray(!should_allocate_full_array);
        array->Set(index, resolved);
      } else {
        // 决策结果：使用“哈希表”
        pairs = AllocateResolvedFields();
        pairs->Set(index, resolved);
      }
    } else {
      // 如果“哈希表”已存在，则直接写入
      pairs->Set(index, resolved);
    }
  }
}

// void Unlink##getter_setter##ArrayIfStartup()
void UnlinkResolvedFieldsArrayIfStartup()
    REQUIRES_SHARED(Locks::mutator_lock_) {
  if (!ShouldAllocateFullArray(GetDexFile()->NumFieldIds(), 1024)) {
    SetResolvedFieldsArray(nullptr) ;
  }
}
```

`DEFINE_DUAL_CACHE` 这个宏为每种缓存类型（resolved_fields, resolved_methods 等）生成了一套统一的逻辑。

- 首先检查 `*_array_` 是否已经被分配了。如果已经分配，就直接使用它。

- 如果未分配，再检查 哈希表方案 是否已被分配。如果已经分配，就使用它。

- 如果两者都未分配（这是第一次为该类型的缓存设置条目），则进入决策逻辑：
  - ShouldAllocateFullArray 会比较这个 DEX 文件中该类型条目的总数（GetDexFile()->ids()）和设置的哈希表大小。
  - 如果总数 <= 哈希表大小: 说明这是个小 DEX 文件，于是分配一个纯数组 (Allocate...Array)。
  - 如果总数 > 哈希表大小: 说明这是个大 DEX 文件，于是分配一个哈希表 (Allocate...)。

## ArtField数据结构

```cpp
class EXPORT ArtField final {
  // ... ...
  // 指向定义了这个字段的 mirror::Class 对象。
  GcRoot<mirror::Class> declaring_class_;

  // 访问标志
  uint32_t access_flags_ = 0;

  //该字段在 .dex 文件 field_ids 池中的索引
  // Dex cache index of field id
  uint32_t field_dex_idx_ = 0;

  // 对于实例字段  这个 offset_ 是指该字段相对于其所属对象实例起始地址的字节偏移。
  // 对于静态字段 这个 offset_ 是指该字段相对于其所属类对象起始地址的字节偏移。
  // Offset of field within an instance or in the Class' static fields
  uint32_t offset_ = 0;
};
```

ArtField 是 Java 字段在 ART 内存中的 C++ 表示。它本身不存储字段的值，而是告诉虚拟机去哪里找到这个字段的值。

## ArtMethod数据结构

```cpp
class EXPORT ArtMethod final {
 protected:
  // 指向定义了这个方法的 mirror::Class 对象
  GcRoot<mirror::Class> declaring_class_;
  
  // 在类被链接后，虚拟机的不同组件（如 JIT 编译器、校验器）可能会在多线程环境中并发地修改
  // 这些标志位。例如，JIT 可能会把它标记为“已编译”，或者一个优化可能会把它标记为“单实现方
  // 法”。使用原子类型确保了这些并发修改是线程安全的。
  std::atomic<std::uint32_t> access_flags_;
  
  // 一个整数，表示这个方法在原始 .dex 文件 method_ids 池中的索引
  uint32_t dex_method_index_;
  
  // 方法在其所属类的方法列表中的索引。
  // 对于静态方法或私有方法，它是 directMethods 数组的索引。
  // 对于虚方法，它是 vtable（虚方法表）的索引。
  // 对于接口方法，它是实现类的 IfTable（接口方法表）中对应接口方法数组的索引。
  uint16_t method_index_;
  
  union {
    // 对于普通方法，这里存储的是它的热度计数值。当一个方法被频繁调用时，ART 会增加这个计
    // 数。JIT 编译器会周期性地检查这个值，当它超过某个阈值时，JIT 就会将这个“热点方法”
    // 编译成高质量的机器码以提升性能。注释中提到它不是原子的，因为偶尔丢失一两次计数不影
    // 响最终识别出热点方法，这是一种性能上的权衡。
    uint16_t hotness_count_;
    // 对于抽象的接口方法，这里存储的是它在 IMT (Interface Method Table) 中的索引，这是
    //  ART 用于加速接口方法调用的另一项优化。
    uint16_t imt_index_;
  };
  
  // Fake padding field gets inserted here.
  
  // Must be the last fields in the method.
  struct PtrSizedFields {
    // Native 方法: 指向已注册的 C/C++ JNI 函数地址。
    // 需要解析的方法 (Resolution Method): 指向一个运行时的内部函数，该函数负责在首次调
    // 用时解析出真正的目标方法。
    // 冲突方法 (Conflict Method): 当接口方法表（IMT）发生哈希冲突时，它指向一个
    // ImtConflictTable 结构，用于解决冲突。
    // 抽象/接口方法: 如果 ART 的静态分析发现这个抽象方法在整个应用中只有一个实现，这里会
    // 直接指向那个唯一的实现方法，将一个动态调用优化为直接调用。
    // 普通方法: 在 AOT 编译时，它存储的是方法字节码 (CodeItem) 在 .dex 文件中的偏移量。
    // 在 运行时，这个偏移量会被“修复”为指向内存中 CodeItem 的直接指针。
    void* data_;
  
    // 当 AOT/JIT 编译后的代码要调用这个方法时，就会跳转到这个指针所指向的地址。
    // 它不一定总是指向 AOT 代码。例如，如果方法需要被解释执行，这个指针就会指向 
    // quick_to_interpreter_bridge_trampoline，从而将执行流引导到解释器。
    void* entry_point_from_quick_compiled_code_;
  } ptr_sized_fields_;
};

```
  
## Class 数据结构
  
```cpp
class EXPORT MANAGED Class final : public Object {
  // 指向加载了这个类的 ClassLoader。如果是启动类加载器，则为 null
  HeapReference<ClassLoader> class_loader_;
  // 数组元素类型: 仅对数组类有效。普通类此字段为 null。
  HeapReference<Class> component_type_;
  // 指向这个类所属的 DexFile 对应的 DexCache。虚拟机通过它来快速解析方法、字段等。
  HeapReference<DexCache> dex_cache_;
  // 一个指向 ClassExt 对象的指针，用于存储一些不常用或“额外”的数据，比如校验和错误信息。
  // 这个字段是懒分配的，只有在需要时才会创建，以节省内存
  HeapReference<ClassExt> ext_data_;
  // 因为 Java 支持实现多个接口，无法用单一的 vtable 来处理。iftable_ 存储了一系列的键值
  // 对，每个键是一个接口的 Class 对象，值是另一个指针数组，该数组包含了当前类对这个接口所
  // 有方法的具体实现（ArtMethod*）。
  HeapReference<IfTable> iftable_;
  // Descriptor 字符串，懒加载。
  HeapReference<String> name_;
  // 指向该类的直接父类。
  HeapReference<Class> super_class_;
  // 这是一个指针数组，包含了所有可以被子类重写（override）的虚方法的 ArtMethod 指针。
  // 当执行 invoke-virtual 指令时，通过对象的 vtable_ 快速找到并调用正确的方法实现。
  // 它从父类拷贝并根据当前类的方法进行修改。
  HeapReference<PointerArray> vtable_;
  // 指向一个 ArtField 的数组，包含了该类直接声明的所有静态和实例字段。
  // 父类的字段不在这里，而在父类的 fields_ 中
  uint64_t fields_;
  // 方法列表: 指向一个 ArtMethod 的数组，包含了该类逻辑上定义的所有方法。
  // 这是一个非常重要的字段，其内部布局被精心划分为三个部分：
  // [0, virtual_methods_offset_): 直接方法。包括静态方法、私有方法和构造函数.
  // [virtual_methods_offset_, copied_methods_offset_): 
  // 虚方法即该类声明的 public 或 protected 的非 final 方法。
  // [copied_methods_offset_, ...): 
  // 拷贝方法。主要指为了实现接口而从父接口中拷贝过来的默认方法 (default method) 
  // 或 Miranda methods (为了让抽象父类满足接口要求而隐式生成的具体方法)。
  uint64_t methods_;
  // 访问标志
  uint32_t access_flags_;
  //ART 内部使用的一些帮助 GC 等子系统快速判断的标志。
  uint32_t class_flags_;
  // Class 对象自身的大小: mirror::Class 对象本身在堆上占用的总字节数。
  uint32_t class_size_;
  // 当一个线程开始初始化这个类时（执行 <clinit>），会把自己的线程ID记录在这里
  // 用于检测多线程环境下的循环初始化错误
  pid_t clinit_thread_id_;
  // 将这个内存中的 Class 对象链接回它在原始 .dex 文件中定义的索引
  int32_t dex_class_def_idx_;
  int32_t dex_type_idx_;
  // 记录了实例/静态字段中引用类型的数量
  uint32_t num_reference_instance_fields_;
  uint32_t num_reference_static_fields_;
  // 实例对象的大小
  uint32_t object_size_;
  // 当一个类满足某些条件时，ART 会将其实例大小缓存到这个字段中。
  // GC 在分配这个类的实例时，可以直接读取这个字段，走一个“快速分配路径”，
  // 避免了更复杂的尺寸计算和状态检查，从而提升对象创建的速度。
  uint32_t object_size_alloc_fast_path_;
  // 低 16 位: 存储一个 Primitive::Type 枚举值，表示这是哪种原始类型
  // 高 16 位: 存储该原始类型尺寸的位移值 (size shift)。例如，int 是 4 字节位移值就是 2
  uint32_t primitive_type_;
  // 实例引用字段偏移量的位图 (Bitmap)。
  // GC 在扫描一个对象时，需要知道哪些字段是指向其他对象的引用 (指针)。
  // 它的每一位 (bit) 对应一个 32 位的内存“槽位”。
  // 如果第 n 位是 1，就表示从对象起始地址偏移 n * 4 字节的位置上存的是一个引用。
  uint32_t reference_instance_offsets_;
  // 类生命周期状态: 主要部分用于存储类的 ClassStatus
  // 子类型检查位 (Subtype Check Bits): 另一部分位用于一个叫做 “Subtype Check” 的优化。
  // 通过一些预计算的位掩码，ART 可以非常快速地判断 
  // A.class.isInstance(b) 或 b instanceof A 这样的类型检查，
  // 其速度远快于遍历整个继承链。
  uint32_t status_;
  
  uint16_t copied_methods_offset_;
  uint16_t virtual_methods_offset_;
  
  
  // 一些可变大小的数据，则被直接地、连续地附加在这些固定字段的内存之后。
  // The following data exist in real class objects.
  
  
  // Embedded Vtable length, for class object that's instantiable, fixed size.
  // uint32_t vtable_length_;
  
  // 这个字段在代码中并没有体现，ImTableEntry没有这个数据结构
  // 我有理由怀疑这是24年提交“使用可变大小的引用偏移位图来加速 VisitReferences()”这个Commit的作者的
  // 笔误， 当时也许在开发了一部分的ImTableEntry，但是没有完成，在提交这个commit的时候混入了这段注释。
  // Embedded Imtable pointer, for class object that's not an interface, fixed size.
  // ImTableEntry embedded_imtable_;
  
  // vtable_ 字段本身只是一个指针，而真正的虚方法表数组内容就存放在这片嵌入式内存区里。
  // 这减少了一次额外的内存分配和指针解引用，提升了 invoke-virtual 的性能。
  // Embedded Vtable, for class object that's not an interface, variable size.
  // VTableEntry embedded_vtable_[0];
  
  // 所有静态字段 (Static Fields) 的实际值也存放在这里。
  // 当访问一个静态字段时，ART 通过 ArtField 中的偏移量，直接在这个嵌入式区域中定位并读写数据。
  // Static fields, variable size.
  // uint32_t fields_[0];
  
  
  // 如果一个类的实例引用字段太多，一个 32 位的 reference_instance_offsets_ 位图不够用，
  // 那么一个更大 的、可变长度的位图就会被存放在这里。
  // Embedded bitmap of offsets of instance fields, for classes that need more than 31
  // reference-offset bits. 'reference_instance_offsets_' stores the number of
  // 32-bit entries that hold the entire bitmap. We compute the offset of first
  // entry by subtracting this number from class_size_.
  // uint32_t reference_bitmap_[0];
};
```
