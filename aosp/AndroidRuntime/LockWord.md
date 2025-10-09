---
layout: default
title: 什么是LockWord
nav_order: 6
parent: AndroidRuntime
grand_parent: AOSP
author: Anonymous Committer
---
# LockWord

LockWord 是 ART VM 中用来表示 Java 对象头（mirror::Object）里锁字段的一种紧凑编码方式。它把thin lock、fat lock、HashCode、forwarding address”等不同含义，通过一个 32 位整数里的若干比特域来区分，并且在同一个 32 位里还嵌入了“读屏障（read barrier）”状态和“标记位（mark bit）”这两种 GC 相关标志。

在Object类的实现中，有monitor的uint32_t成员

```cpp
// The Class representing the type of the object.
  HeapReference<Class> klass_;
  // Monitor and hash code information.
  uint32_t monitor_;
```

这个monitor_成员实际上就会转化成LockWord进行管理，比如在Object类成员函数`GetReadBarrierState`中

```cpp
inline uint32_t Object::GetReadBarrierState() {
  if (!kUseBakerReadBarrier) {
    LOG(FATAL) << "Unreachable";
    UNREACHABLE();
  }
  DCHECK(kUseBakerReadBarrier);
  // 通过MonitorOffset得到偏移然后得到monitor_的字段，构造出LockWord
  LockWord lw(GetFieldPrimitive<uint32_t, /*kIsVolatile=*/false>(MonitorOffset())); 
  // 使用LockWord的ReadBarrierState成员函数
  uint32_t rb_state = lw.ReadBarrierState();
  DCHECK(ReadBarrier::IsValidReadBarrierState(rb_state)) << rb_state;
  return rb_state;
}
```

## 官方注释

LockWord的官方注释如下

```cpp
/* The lock value itself as stored in mirror::Object::monitor_.  The two most significant bits
 * encode the state. The four possible states are fat locked, thin/unlocked, hash code, and
 * forwarding address.
 *
 * When the lock word is in the "thin" state and its bits are formatted as follows:
 *
 *  |33|2|2|222222221111|1111110000000000|
 *  |10|9|8|765432109876|5432109876543210|
 *  |00|m|r| lock count |thread id owner |
 *
 * The lock count is zero, but the owner is nonzero for a simply held lock.
 * When the lock word is in the "fat" state and its bits are formatted as follows:
 *
 *  |33|2|2|2222222211111111110000000000|
 *  |10|9|8|7654321098765432109876543210|
 *  |01|m|r| MonitorId                  |
 *
 * When the lock word is in hash state and its bits are formatted as follows:
 *
 *  |33|2|2|2222222211111111110000000000|
 *  |10|9|8|7654321098765432109876543210|
 *  |10|m|r| HashCode                   |
 *
 * When the lock word is in forwarding address state and its bits are formatted as follows:
 *
 *  |33|2|22222222211111111110000000000|
 *  |10|9|87654321098765432109876543210|
 *  |11|0| ForwardingAddress           |
 *
 * The `r` bit stores the read barrier state.
 * The `m` bit stores the mark bit state.
 */
 ```

总结

- `r`：一个比特，用于 ReadBarrier 状态（读屏障 GC 标志）
- `m`：一个比特，用于 MarkBit 状态（标记位，用于 GC）
- 最高两位（bit 31-30）：表示当前 lock word 的“状态类型”（共 4 种状态）

```plain
| 31 30 | 29 m| 28 r| 27 ...... 12  |  11 ........ 0   |
|  0  0 | 29 m| 28 r|  lock count   | owner thread id  |
|  0  1 | 29 m| 28 r|            MonitorId             |
|  1  0 | 29 m| 28 r|            hashcode              |
|  1  1 | forwarding address (>>kObjectAlignmentShift) |
```

类似的我们可以设计一个64位下的LockWord

```plain
| 63 62 | 61 m| 60 r| 59 ... 28 | 27 ... 12  |.  11 ........ 0  |
|  0  0 | 61 m| 60 r|       lock count       | owner thread id  |
|  0  1 | 61 m| 60 r|           |         MonitorId             |
|  1  0 | 61 m| 60 r|           |         hashcode              |
|  1  1 |      forwarding address (>>kObjectAlignmentShift)     |
```

代码中体现为

```cpp
enum SizeShiftsAndMasks : uint32_t {  // private marker to avoid generate-operator-out.py from processing.
    // Number of bits to encode the state, currently just fat or thin/unlocked or hash code.
    kStateSize = 2,
    kReadBarrierStateSize = 1,
    kMarkBitStateSize = 1,
    // Number of bits to encode the thin lock owner.
    kThinLockOwnerSize = 16,
    // Remaining bits are the recursive lock count. Zero means it is locked exactly once
    // and not recursively.
    kThinLockCountSize = 32 - kThinLockOwnerSize - kStateSize - kReadBarrierStateSize -
        kMarkBitStateSize,

    // Thin lock bits. Owner in lowest bits.
    kThinLockOwnerShift = 0,
    kThinLockOwnerMask = (1 << kThinLockOwnerSize) - 1,
    kThinLockOwnerMaskShifted = kThinLockOwnerMask << kThinLockOwnerShift,
    kThinLockMaxOwner = kThinLockOwnerMask,
    // Count in higher bits.
    kThinLockCountShift = kThinLockOwnerSize + kThinLockOwnerShift,
    kThinLockCountMask = (1 << kThinLockCountSize) - 1,
    kThinLockMaxCount = kThinLockCountMask,
    kThinLockCountOne = 1 << kThinLockCountShift,  // == 65536 (0x10000)
    kThinLockCountMaskShifted = kThinLockCountMask << kThinLockCountShift,

    // State in the highest bits.
    kStateShift = kReadBarrierStateSize + kThinLockCountSize + kThinLockCountShift +
        kMarkBitStateSize,
    kStateMask = (1 << kStateSize) - 1,
    kStateMaskShifted = kStateMask << kStateShift,
    kStateThinOrUnlocked = 0,
    kStateFat = 1,
    kStateHash = 2,
    kStateForwardingAddress = 3,
    kStateForwardingAddressShifted = kStateForwardingAddress << kStateShift,
    kStateForwardingAddressOverflow = (1 + kStateMask - kStateForwardingAddress) << kStateShift,

    // Read barrier bit.
    kReadBarrierStateShift = kThinLockCountSize + kThinLockCountShift,
    kReadBarrierStateMask = (1 << kReadBarrierStateSize) - 1,
    kReadBarrierStateMaskShifted = kReadBarrierStateMask << kReadBarrierStateShift,
    kReadBarrierStateMaskShiftedToggled = ~kReadBarrierStateMaskShifted,

    // Mark bit.
    kMarkBitStateShift = kReadBarrierStateSize + kReadBarrierStateShift,
    kMarkBitStateMask = (1 << kMarkBitStateSize) - 1,
    kMarkBitStateMaskShifted = kMarkBitStateMask << kMarkBitStateShift,
    kMarkBitStateMaskShiftedToggled = ~kMarkBitStateMaskShifted,

    // GC state is mark bit and read barrier state.
    kGCStateSize = kReadBarrierStateSize + kMarkBitStateSize,
    kGCStateShift = kReadBarrierStateShift,
    kGCStateMaskShifted = kReadBarrierStateMaskShifted | kMarkBitStateMaskShifted,
    kGCStateMaskShiftedToggled = ~kGCStateMaskShifted,

    // When the state is kHashCode, the non-state bits hold the hashcode.
    // Note Object.hashCode() has the hash code layout hardcoded.
    kHashShift = 0,
    kHashSize = 32 - kStateSize - kReadBarrierStateSize - kMarkBitStateSize,
    kHashMask = (1 << kHashSize) - 1,
    kMaxHash = kHashMask,

    // Forwarding address shift.
    kForwardingAddressShift = kObjectAlignmentShift,

    kMonitorIdShift = kHashShift,
    kMonitorIdSize = kHashSize,
    kMonitorIdMask = kHashMask,
    kMonitorIdAlignmentShift = 32 - kMonitorIdSize,
    kMonitorIdAlignment = 1 << kMonitorIdAlignmentShift,
    kMaxMonitorId = kMaxHash
  };
```

## 四种状态

### “Thin/Unlocked” 状态（kStateThinOrUnlocked = 0）

当最高两位 = 00 时，表示是“薄锁”或者“Unlocked”或者“HashCode”状态中的一种，要结合 “m/r” 和“count/owner”位来判定。

#### Unlocked（完全未加锁，无 HashCode / 无 GC 标志）

- 此时 全部为 0。
- 最高两位 00，读屏障 bit = 0，标记 bit = 0，thin lock count = 0，owner = 0。
- 任何 LockWord 实例默认构造（LockWord()）就对应 “Unlocked” 状态。

#### Thin Lock（“单线程持有、无竞争”状态）

- 最高两位同样是 00。
- “m” 或 “r” bit 可能依然是 0（如果没参与并发 GC 情况）。
- 低 16 位（kThinLockOwner）保存当前尝试加锁的线程 ID（线程 ID 实际上是一个 16 位以内的 ID）。
- 接下来的 12 位（kThinLockCount）保存重入次数：当线程第一次 lock，count=0；如果同一个线程再次 lock，就 count++。
- 例如：若线程 ID = 0x0012，且重入次数为 0，那么这一时刻 value_ 的二进制就是：

```plain
[state=00][m=0][r=0][count=000000000000][owner=00000000010010]
```

- 当 thin lock 发生竞争或需要 inflate 时，可能需要转为“Fat lock”状态。

### HashCode（kStateHash = 2，两位最高为 10）

当一个对象第一次调用 hashCode()，且它从未加锁且无 hashcode，就会把 lock word 切换到 “HashCode” 状态。

- 此时最高两位 10（即 kStateHash），GC 状态 bit（m/r）通常是 0，HashCode 本身放在低 30 位，具体占用 kHashSize = 32 − 2 − 1 − 1 = 28 位。
- 也就是说 `value_ = (hashCode << kHashShift) | (gc_bits << kGCStateShift) | (kStateHash << kStateShift)。`
- 其中 kHashShift = 0，即低 28 位直接存 hashCode & 0x0FFFFFFF。

取出 HashCode 的方法：

```cpp
inline int32_t LockWord::GetHashCode() const {
  DCHECK_EQ(GetState(), kHashCode);
  CheckReadBarrierState();
  return (value_ >> kHashShift) & kHashMask;  // kHashShift = 0, kHashMask = (1<<28)-1
}
```

- 当第一次调用 hashCode()，并且 LockWord 还是 value_== 0（unlocked），就 CAS 将其改为 FromHashCode(computedHash, gc_bits=0)。之后再调用 hashCode 就直接读取 GetHashCode()，无需额外字典维护。
- 如果后续需要加锁（ThinLock），就要把 HashCode 挪到Monitor中，或者在 ThinLock 状态里把 hash 放到 Monitor 里，“inflate”到FatLock，等解锁后把 hash 还原到 lock word。

### Forwarding Address（kStateForwardingAddress = 3，两位最高为 11）

- 在某些 GC（如复制式 GC、压缩 GC）时，对象被搬迁到新地址后，旧对象头要记录一个“转发地址”，以便后续其它引用可以更新引用指向。
- 此时最高两位 = 11，读屏障和标记 bit 位为 0（因为 forwarding address 一般不会与读屏障/标记位共存）。
- 剩下的 30 位里，真正存储的是：forwarding_address >> kForwardingAddressShift。一般 kForwardingAddressShift = kObjectAlignmentShift，因为对象地址总是对齐到kObjectAlignmentShift，低几位都为 0，可以丢弃。

取出 forwarding address 的方法：

```cpp
inline size_t LockWord::ForwardingAddress() const {
  DCHECK_EQ(GetState(), kForwardingAddress);
  return value_ << kForwardingAddressShift;
}
```

### “Fat Lock”（kStateFat = 1，最高两位 01）

当 lock word 决定要“inflate”到胖锁（如出现竞争时），就把整 32 位转换为：

```plain
[ state = 01 ] [ markBit? ] [ readBarrier? ] [ MonitorId (30-bit) ]
```

- 最高两位 01 表示这是一个“Fat Locked”状态；
- 之后两位 “m/r” 可能顺便携带 GC 状态信息。
- 余下的 30 位就是一个 “MonitorId”，该 ID 指向全局的一个 MonitorPool 中的某个 Monitor 对象。Monitor 结构包含：
  - 监视器所有者（owner），
    - 递归锁计数，
    - 等待在此 Monitor 上的线程队列（含条件变量）等实际同步数据，
    - 还有可能记录用于转入/转出等。

FatLockMonitor 的 C++ 实现如下：

```cpp
inline Monitor* LockWord::FatLockMonitor() const {
  DCHECK_EQ(GetState(), kFatLocked);
  CheckReadBarrierState();
  MonitorId mon_id = (value_ >> kMonitorIdShift) & kMonitorIdMask;
  return MonitorPool::MonitorFromMonitorId(mon_id);
}
```

- MonitorPool::MonitorFromMonitorId(mon_id) 会从全局 MonitorPool 中取出对应 ID 的 Monitor 对象指针。

注意：在 FatLock 状态下，“ThinLockOwner”与“ThinLockCount”域都已失效，把具体的锁定信息让给 Monitor 处理。

## GC 相关：读屏障（ReadBarrier）与标记位（MarkBit）

在上述各状态布局中，还预留了两位来保存 GC 时的额外状态.

LockWord 提供了以下方法来获取/设置这两位：

```cpp
uint32_t ReadBarrierState() const {
  return (value_ >> kReadBarrierStateShift) & kReadBarrierStateMask;
}
void SetReadBarrierState(uint32_t rb_state) {
  // 清掉 old bits
  value_ &= ~(kReadBarrierStateMask << kReadBarrierStateShift);
  // 或上新的 rb_state
  value_ |= (rb_state & kReadBarrierStateMask) << kReadBarrierStateShift;
}

uint32_t MarkBitState() const {
  return (value_ >> kMarkBitStateShift) & kMarkBitStateMask;
}
void SetMarkBitState(uint32_t mark_bit) {
  value_ &= kMarkBitStateMaskShiftedToggled;  // 清掉原来的 mark bit
  value_ |= mark_bit << kMarkBitStateShift;
}
```

- 这些方法都会先 DCHECK 一下当前状态不是 forwarding address（即不允许在转发地址状态下修改 GC state）。

## 从 Thin 转到 Fat（inflate）

- 正常无竞争加锁：第一次 monitorenter 时，ART 会尝试 CAS 将 “unlocked” （value_ = 0） 更新为 “ThinLock” 状态：即 `thread_id << ownerShift) | (0 << countShift) | (state=0)`.
- 重入：如果同一线程再次进入 synchronized，发现自己正是 owner，则仅把 “count” 增加 1。
- 检测到竞争：如果某个线程看到 owner 不为自己，就要 inflate。
- Inflate 的步骤：
    1. 创建或从 MonitorPool 中分配一个新的 Monitor。
    2. 构造 LockWord(Monitor*) 也就是 FatLock：把 state=01、MonitorId 写入；
    3. Monitor 结构里会把当前 ThinLock 的 owner 线程加入 Monitor 的 owner 字段，并且把待竞争线程 enqeue 到等待列表。
- 释放锁：ThinLock 时，owner 线程退出同步块后，(count>0 ? count– : owner=0)。如果 owner 清零，就把 value_ 重置为“纯粹 unlocked + GC bits”状态。
- 如果是 FatLock：调用 MonitorPool 里已分配好的 Monitor，Monitor 自己维护一个引用计数，当真正没人持有时再回收 Monitor 并把 LockWord 重置为 “unlocked”。
