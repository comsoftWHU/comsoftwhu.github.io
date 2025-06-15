---
layout: default
title: Memmap数据结构
nav_order: 1
parent: AndroidRuntime
grand_parent: AOSP
author: Anonymous Committer
---

# Memmap数据结构

用来管理通过系统调用`mmap`获得的内存段。

## mmap 系统调用

<https://linux.die.net/man/2/mmap>

`mmap`（memory map）类Unix系统中的一个重要系统调用，用于在进程的地址空间中创建内存映射。它允许程序将文件或设备直接映射到内存中，从而实现高效的文件访问和内存管理。

### 基本概念

`mmap`系统调用在进程的虚拟地址空间中创建一个新的映射，可以是：
- **文件映射**：将文件的一部分映射到内存
- **匿名映射**：不关联任何文件的内存区域
    - 匿名映射用于实现malloc的大内存分配


工作原理

1. **建立映射**：
- 内核在进程的虚拟地址空间中分配区域
- 对于文件映射，建立虚拟地址到文件块的映射关系
- 实际物理内存分配延迟到首次访问时（按需分页）

2. **访问机制**：
- 进程访问映射区域时触发缺页异常
- 内核处理异常，加载对应文件内容到物理内存
- 建立页表项，指向实际物理页

3. **写回机制**：
- `MAP_SHARED`：修改会写入页缓存，最终由内核写回文件
- `MAP_PRIVATE`：修改触发写时复制，创建私有副本


### 函数原型

```c
#include <sys/mman.h>

void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

参数说明

| 参数     | 说明                                                                 |
|----------|----------------------------------------------------------------------|
| `addr`   | **建议**的映射起始地址（hint）                       |
| `length` | 映射区域的长度（字节数）                                             |
| `prot`   | 内存保护标志（读/写/执行权限）                                       |
| `flags`  | 映射类型和选项                                                       |
| `fd`     | 文件描述符（匿名映射时设为-1）                                       |
| `offset` | 文件映射的起始偏移量（通常为0）                                      |

保护标志（prot）

| 标志          | 说明                     |
|---------------|--------------------------|
| `PROT_READ`   | 页面可读                 |
| `PROT_WRITE`  | 页面可写                 |
| `PROT_EXEC`   | 页面可执行               |
| `PROT_NONE`   | 页面不可访问             |
| ...... | ......|

映射标志（flags）

| 标志             | 说明                                                                 |
|------------------|----------------------------------------------------------------------|
| `MAP_SHARED`     | 共享映射：表示映射区域是共享的。对映射区域的修改会被写回到文件（如果是文件映射），并且其他映射了同一文件的进程可以看到这些修改（但是修改不一定会及时写回文件）。在匿名映射的情况下，`MAP_SHARED`可以用于实现父子进程之间的共享内存                             |
| `MAP_PRIVATE`    | 私有映射：写时复制（Copy-on-Write），修改不会影响原文件              |
| `MAP_ANONYMOUS`  | 匿名映射：不关联文件（fd应为-1）                                     |
| `MAP_FIXED`      | 使用精确指定的地址。addr 必须是页面大小的整数倍。如果由 addr 和 len 指定的内存区域与任何现有映射的页面重叠，则现有映射的重叠部分将被丢弃。如果无法使用指定地址，mmap() 将失败。由于要求固定地址的映射方式可移植性较差，不建议使用此选项。                             |
| ...... | ......|
### 相关系统调用

`munmap`解除映射

```c
int munmap(void *addr, size_t length);
```
- 移除指定地址范围的映射
- `addr`必须是`mmap`返回的地址
- `length`应与原映射长度一致

`msync`同步内存与文件

```c
int msync(void *addr, size_t length, int flags);
```
- 正常用来把用户空间的内存页与底层文件或设备同步。
  - 但如果 addr 指向的是 未映射 的地址范围，msync 会立刻返回 -1 并把 errno 设为 ENOMEM。
- `flags`选项：
  - `MS_SYNC`：同步写入，等待完成
  - `MS_ASYNC`：异步写入，仅安排写操作
  - `MS_INVALIDATE`：使其他映射失效
- “安全页”（safe page）
	- 就是指那些调用 msync 得到 ENOMEM 错误的页，这表明在当前进程的虚拟地址空间里，这个页根本不存在任何映射——也就是说它真的是一片“空白区”，可以放心拿来做 mmap。

### 示例
 文件映射示例
```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    int fd = open("example.txt", O_RDWR);
    struct stat sb;
    fstat(fd, &sb);
    
    // 创建文件映射
    char *mem = mmap(NULL, sb.st_size, 
                    PROT_READ | PROT_WRITE,
                    MAP_SHARED, fd, 0);
    
    // 通过指针访问文件内容
    printf("First character: %c\n", mem[0]);
    
    // 修改文件内容
    mem[0] = 'A';
    
    // 同步到磁盘
    msync(mem, sb.st_size, MS_SYNC);
    
    // 解除映射
    munmap(mem, sb.st_size);
    close(fd);
    return 0;
}
```
匿名映射示例（进程间共享）

```c
#include <sys/mman.h>
#include <sys/wait.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    // 创建共享匿名映射
    int *shared = mmap(NULL, sizeof(int),
                      PROT_READ | PROT_WRITE,
                      MAP_SHARED | MAP_ANONYMOUS,
                      -1, 0);
    
    *shared = 0;
    
    pid_t pid = fork();
    if (pid == 0) {
        // 子进程
        (*shared)++;
        _exit(0);
    } else {
        // 父进程
        wait(NULL);
        printf("Shared value: %d\n", *shared); // 输出1
        munmap(shared, sizeof(int));
    }
    return 0;
}
```


### AOSP里的封装

```cpp

void* MemMap::TargetMMap(void* start, size_t len, int prot, int flags, int fd, off_t fd_off) {
  return mmap(start, len, prot, flags, fd, fd_off);
}

int MemMap::TargetMUnmap(void* start, size_t len) {
  return munmap(start, len);
}
```

## AOSP中的MemMap

在不支持 MAP_32BIT 的64位系统上，  MemMap的实现将进行**线性扫描**以查找空闲页面。出于安全考虑，**扫描的起始位置使用动态初始化器随机化**。不能有其他静态初始化器访问MemMap，否则，调用时可能会看到未初始化的值。

### 宏USE_ART_LOW_4G_ALLOCATOR

```cpp

/*
 * 内存分配器架构选择逻辑：
 * 
 * 在64位系统（非Fuchsia）且满足以下条件时启用低4G内存分配器：
 * 1. ARM64架构（__aarch64__）或
 * 2. RISC-V架构（__riscv）或 
 * 3. Apple平台（__APPLE__）
 * 
 */

#if defined(__LP64__) && !defined(__Fuchsia__) && \
    (defined(__aarch64__) || defined(__riscv) || defined(__APPLE__))
#define USE_ART_LOW_4G_ALLOCATOR 1
#else
#if defined(__LP64__) && !defined(__Fuchsia__) && !defined(__x86_64__)
// 如果遇到未识别的64位架构（非x86_64），编译时报错
#error "Unrecognized 64-bit architecture."
#endif
#define USE_ART_LOW_4G_ALLOCATOR 0
#endif

// Linux系统特有优化：
// 1. 支持madvise(MADV_DONTNEED)清零内存页
// 2. 提供mremap系统调用支持内存区域替换
#ifdef __linux__
static constexpr bool kMadviseZeroes = true;
#define HAVE_MREMAP_SYSCALL true
#else
static constexpr bool kMadviseZeroes = false;
// We cannot ever perform MemMap::ReplaceWith on non-linux hosts since the syscall is not present.
#define HAVE_MREMAP_SYSCALL false
#endif
```

### 成员变量

```cpp
class MemMap {
......
  std::string name_;
  uint8_t* begin_ = nullptr;    // Start of data. May be changed by AlignBy.
  size_t size_ = 0u;            // Length of data.

  void* base_begin_ = nullptr;  // Page-aligned base address. May be changed by AlignBy.
  size_t base_size_ = 0u;       // Length of mapping. May be changed by RemapAtEnd (ie Zygote).
  int prot_ = 0;                // Protection of the map.

  // When reuse_ is true, this is a view of a mapping on which
  // we do not take ownership and are not responsible for
  // unmapping.
  bool reuse_ = false;

  // When already_unmapped_ is true the destructor will not call munmap.
  bool already_unmapped_ = false;

  size_t redzone_size_ = 0u;

#if USE_ART_LOW_4G_ALLOCATOR
  static uintptr_t next_mem_pos_;   // Next memory location to check for low_4g extent.

  static void* TryMemMapLow4GB(void* ptr,
                               size_t page_aligned_byte_count,
                               int prot,
                               int flags,
                               int fd,
                               off_t offset);
#endif

  static void TargetMMapInit();
  static void* TargetMMap(void* start, size_t len, int prot, int flags, int fd, off_t fd_off);
  static int TargetMUnmap(void* start, size_t len);

  static std::mutex* mem_maps_lock_;

#ifdef ART_PAGE_SIZE_AGNOSTIC
  static size_t page_size_;
#endif

  friend class MemMapTest;  // To allow access to base_begin_ and base_size_.
};
......
}
```


### 成员函数

#### MapAnonymousPreferredAddress
```cpp
non_moving_space_mem_map = MapAnonymousPreferredAddress(
          space_name, request_begin, non_moving_space_capacity, &error_str);
```

```cpp
 main_mem_map_1 = MapAnonymousPreferredAddress(
          kMemMapSpaceName[0], request_begin, capacity_, &error_str);
```

在heap初始化流程中，使用MapAnonymousPreferredAddress尝试紧接着request_begin之后分配页面（request_begin如果LoadImage成功则会设置为image space的末尾）

方法实现：

```cpp
MemMap Heap::MapAnonymousPreferredAddress(const char* name,
                                          uint8_t* request_begin,
                                          size_t capacity,
                                          std::string* out_error_str) {
  while (true) {
    MemMap map = MemMap::MapAnonymous(name,
                                      request_begin,
                                      capacity,
                                      PROT_READ | PROT_WRITE,
                                      /*low_4gb=*/ true,
                                      /*reuse=*/ false,
                                      /*reservation=*/ nullptr,
                                      out_error_str);
    if (map.IsValid() || request_begin == nullptr) {
      return map;
    }
    // Retry a  second time with no specified request begin.
    request_begin = nullptr;
  }
}
```
该方法其实就是第一次尝试一下在request_begin的位置MapAnonymous，如果失败了则清空request_begin，然后再调用一次MapAnonymous。
所以并不能保证一定能在request_begin拿到mmap。

#### MapAnonymous

```cpp
main_mem_map_1 = MemMap::MapAnonymous(
          kMemMapSpaceName[0],
          request_begin,
          capacity_,
          PROT_READ | PROT_WRITE,
          /* low_4gb= */ true,
          /* reuse= */ false,
          heap_reservation.IsValid() ? &heap_reservation : nullptr,
          &error_str);
```

也会直接使用MapAnonymous方法。

方法实现：

```cpp
MemMap MemMap::MapAnonymous(const char* name,
                            uint8_t* addr,
                            size_t byte_count,
                            int prot,
                            bool low_4gb,
                            bool reuse,
                            /*inout*/MemMap* reservation,
                            /*out*/std::string* error_msg,
                            bool use_debug_name) {
#ifndef __LP64__
  UNUSED(low_4gb);
#endif
  //不允许申请 0 大小的映射，直接记录错误信息并返回一个无效的 MemMap。
  if (byte_count == 0) {
    *error_msg = "Empty MemMap requested.";
    return Invalid();
  }
  // 对齐到页大小
  size_t page_aligned_byte_count = RoundUp(byte_count, GetPageSize());

  // MAP_ANONYMOUS 表示不关联任何文件，MAP_PRIVATE 表示写时拷贝
  int flags = MAP_PRIVATE | MAP_ANONYMOUS;
 
  if (reuse) {
    // 表示调用者自己已经在 addr 地址预留了一段空白区域，这里只是重用那片地址空间
    // reuse means it is okay that it overlaps an existing page mapping.
    // Only use this if you actually made the page reservation yourself.
    CHECK(addr != nullptr);
    DCHECK(reservation == nullptr);

    // 确保这片内存确实可用
    DCHECK(ContainedWithinExistingMap(addr, byte_count, error_msg)) << *error_msg;
    // 所以会使用 MAP_FIXED 标志
    flags |= MAP_FIXED;
  } else if (reservation != nullptr) {
    // 已经提前在 reservation 中保留好一块连续地址
    CHECK(addr != nullptr);
    if (!CheckReservation(addr, byte_count, name, *reservation, error_msg)) {
      return MemMap::Invalid();
    }
    flags |= MAP_FIXED;
  }

  unique_fd fd;

  // We need to store and potentially set an error number for pretty printing of errors
  int saved_errno = 0;

  void* actual = nullptr;

#if defined(__linux__)
  // 在内核 ≥4.17 时，用 MAP_FIXED_NOREPLACE
	// 它的含义是“要么在 addr 映射成功，要么直接失败，不走随机地址”
  // Recent kernels have a bug where the address hint might be ignored.
  // See https://lore.kernel.org/all/20241115215256.578125-1-kaleshsingh@google.com/
  // We use MAP_FIXED_NOREPLACE to tell the kernel it must allocate at the address or fail.
  // If the fixed-address allocation fails, we fallback to the default path (random address).
  // Therefore, non-null 'addr' still behaves as hint-only as far as ART api is concerned.
  if ((flags & MAP_FIXED) == 0 && addr != nullptr && IsKernelVersionAtLeast(4, 17)) {
    actual = MapInternal(
        addr, page_aligned_byte_count, prot, flags | MAP_FIXED_NOREPLACE, fd.get(), 0, low_4gb);
  }
#endif  // __linux__

  if (actual == nullptr || actual == MAP_FAILED) {
    actual = MapInternal(addr, page_aligned_byte_count, prot, flags, fd.get(), 0, low_4gb);
  }
  saved_errno = errno;

  if (actual == MAP_FAILED) {
    if (error_msg != nullptr) {
      PrintFileToLog("/proc/self/maps", LogSeverity::WARNING);
      *error_msg = StringPrintf("Failed anonymous mmap(%p, %zd, 0x%x, 0x%x, %d, 0): %s. "
                                    "See process maps in the log.",
                                addr,
                                page_aligned_byte_count,
                                prot,
                                flags,
                                fd.get(),
                                strerror(saved_errno));
    }
    return Invalid();
  }
  if (!CheckMapRequest(addr, actual, page_aligned_byte_count, error_msg)) {
    return Invalid();
  }

  if (use_debug_name) {
    SetDebugName(actual, name, page_aligned_byte_count);
  }

  if (reservation != nullptr) {
    // Re-mapping was successful, transfer the ownership of the memory to the new MemMap.
    DCHECK_EQ(actual, reservation->Begin());
    reservation->ReleaseReservedMemory(byte_count);
  }
  // 将所有参数，名字、起始地址、长度、权限、是否复用等打包成一个 MemMap。
  return MemMap(name,
                reinterpret_cast<uint8_t*>(actual),
                byte_count,
                actual,
                page_aligned_byte_count,
                prot,
                reuse);
}
```

#### MapInternal

```cpp
void* MemMap::MapInternal(void* addr,
                          size_t length,
                          int prot,
                          int flags,
                          int fd,
                          off_t offset,
                          bool low_4gb) {
#ifdef __LP64__
  // When requesting low_4g memory and having an expectation, the requested range should fit into
  // 4GB.
  // 当low_4gb==true时，保证所给的 addr（以及 addr+length）都不会越过 32 位地址空间。
  if (low_4gb && (
      // Start out of bounds.
      (reinterpret_cast<uintptr_t>(addr) >> 32) != 0 ||
      // End out of bounds. For simplicity, this will fail for the last page of memory.
      ((reinterpret_cast<uintptr_t>(addr) + length) >> 32) != 0)) {
    LOG(ERROR) << "The requested address space (" << addr << ", "
               << reinterpret_cast<void*>(reinterpret_cast<uintptr_t>(addr) + length)
               << ") cannot fit in low_4gb";
    return MAP_FAILED;
  }
#else
  UNUSED(low_4gb);
#endif
  DCHECK_ALIGNED_PARAM(length, GetPageSize());
  // TODO:
  // A page allocator would be a useful abstraction here, as
  // 1) It is doubtful that MAP_32BIT on x86_64 is doing the right job for us
  void* actual = MAP_FAILED;
#if USE_ART_LOW_4G_ALLOCATOR
  // MAP_32BIT only available on x86_64.
  if (low_4gb && addr == nullptr) {
    // The linear-scan allocator has an issue when executable pages are denied (e.g., by selinux
    // policies in sensitive processes). In that case, the error code will still be ENOMEM. So
    // the allocator will scan all low 4GB twice, and still fail. This is *very* slow.
    //
    // To avoid the issue, always map non-executable first, and mprotect if necessary.
    const int orig_prot = prot;
    const int prot_non_exec = prot & ~PROT_EXEC;
    // Android 中某些进程（受 SELinux 策略限制）不能直接在低 4 GiB 范围分配可执行页面。这时，会先用 “不带可执行位” 的分配器找到地址。
    actual = MapInternalArtLow4GBAllocator(length, prot_non_exec, flags, fd, offset);

    if (actual == MAP_FAILED) {
      return MAP_FAILED;
    }

    // 再用 mprotect 打开 PROT_EXEC
    // See if we need to remap with the executable bit now.
    if (orig_prot != prot_non_exec) {
      if (mprotect(actual, length, orig_prot) != 0) {
        PLOG(ERROR) << "Could not protect to requested prot: " << orig_prot;
        TargetMUnmap(actual, length);
        errno = ENOMEM;
        return MAP_FAILED;
      }
    }
    return actual;
  }

  actual = TargetMMap(addr, length, prot, flags, fd, offset);
#else
#if defined(__LP64__)
  // 支持MAP_32BIT的平台，直接使用系统 mmap + MAP_32BIT
  if (low_4gb && addr == nullptr) {
    flags |= MAP_32BIT;
  }
#endif
  actual = TargetMMap(addr, length, prot, flags, fd, offset);
#endif
  return actual;
}
```

#### MapInternalArtLow4GBAllocator

`MapInternalArtLow4GBAllocator`是ART自己实现的一个线性分配器用来管理低32位的地址空间。

```cpp

void* MemMap::MapInternalArtLow4GBAllocator(size_t length,
                                            int prot,
                                            int flags,
                                            int fd,
                                            off_t offset) {
#if USE_ART_LOW_4G_ALLOCATOR
  void* actual = MAP_FAILED;

  bool first_run = true;

  std::lock_guard<std::mutex> mu(*mem_maps_lock_);
  // 从 next_mem_pos_ 开始，线性扫描整个低 4GB 空间
  for (uintptr_t ptr = next_mem_pos_; ptr < 4 * GB; ptr += GetPageSize()) {
    // Use gMaps as an optimization to skip over large maps.
    // Find the first map which is address > ptr.
    // 利用 gMaps 跳过已映射区间
    auto it = gMaps->upper_bound(reinterpret_cast<void*>(ptr));
    if (it != gMaps->begin()) {
      auto before_it = it;
      --before_it;
      // Start at the end of the map before the upper bound.
      ptr = std::max(ptr, reinterpret_cast<uintptr_t>(before_it->second->BaseEnd()));
      CHECK_ALIGNED_PARAM(ptr, GetPageSize());
    }
    while (it != gMaps->end()) {
      // 检查空间是足够
      // How much space do we have until the next map?
      size_t delta = reinterpret_cast<uintptr_t>(it->first) - ptr;
      // If the space may be sufficient, break out of the loop.
      if (delta >= length) {
        break;
      }
      // Otherwise, skip to the end of the map.
      ptr = reinterpret_cast<uintptr_t>(it->second->BaseEnd());
      CHECK_ALIGNED_PARAM(ptr, GetPageSize());
      ++it;
    }
    // 试一试这个地址能否直接 mmap 成功
    // Try to see if we get lucky with this address since none of the ART maps overlap.
    actual = TryMemMapLow4GB(reinterpret_cast<void*>(ptr), length, prot, flags, fd, offset);
    if (actual != MAP_FAILED) {
      // 成功后更新 next_mem_pos_，并立即返回
      next_mem_pos_ = reinterpret_cast<uintptr_t>(actual) + length;
      return actual;
    }

    // 如果距离 4GB 剩余空间不足，尝试从底部重来一次
    if (4U * GB - ptr < length) {
      // Not enough memory until 4GB.
      if (first_run) {
        // Try another time from the bottom;
        ptr = LOW_MEM_START - GetPageSize();
        first_run = false;
        continue;
      } else {
        // Second try failed.
        break;
      }
    }

    uintptr_t tail_ptr;

    // Check pages are free. 通过 msync 判断该页是否真的未被映射
    // 纯靠 mmap 试探可能因为内核策略或预留区域而失败，不一定能区分“真正被其它映射占用” vs “位置不合适”两种情况。
    bool safe = true;
    for (tail_ptr = ptr; tail_ptr < ptr + length; tail_ptr += GetPageSize()) {
      if (msync(reinterpret_cast<void*>(tail_ptr), GetPageSize(), 0) == 0) {
        safe = false;
        break;
      } else {
        DCHECK_EQ(errno, ENOMEM);
      }
    }

    next_mem_pos_ = tail_ptr;  // update early, as we break out when we found and mapped a region

    // safe == true 代表从底部开始存在一个大小为length的空余区间，再试一试TryMemMapLow4GB
    if (safe == true) {
      actual = TryMemMapLow4GB(reinterpret_cast<void*>(ptr), length, prot, flags, fd, offset);
      if (actual != MAP_FAILED) {
        return actual;
      }
    } else {
      // Skip over last page.
      ptr = tail_ptr;
    }
  }

  if (actual == MAP_FAILED) {
    LOG(ERROR) << "Could not find contiguous low-memory space.";
    errno = ENOMEM;
  }
  return actual;
#else
  // 不使用USE_ART_LOW_4G_ALLOCATOR的情况下不可能运行到这个方法
  UNUSED(length, prot, flags, fd, offset);
  LOG(FATAL) << "Unreachable";
  UNREACHABLE();
#endif
}
```


#### TryMemMapLow4GB
```cpp

#if USE_ART_LOW_4G_ALLOCATOR
void* MemMap::TryMemMapLow4GB(void* ptr,
                                    size_t page_aligned_byte_count,
                                    int prot,
                                    int flags,
                                    int fd,
                                    off_t offset) {
  void* actual = TargetMMap(ptr, page_aligned_byte_count, prot, flags, fd, offset);
  if (actual != MAP_FAILED) {
    // Since we didn't use MAP_FIXED the kernel may have mapped it somewhere not in the low
    // 4GB. If this is the case, unmap and retry.
    // 如果分配到的地址空间超过了4 * GB，munmap然后返回MAP_FAILED
    if (reinterpret_cast<uintptr_t>(actual) + page_aligned_byte_count >= 4 * GB) {
      TargetMUnmap(actual, page_aligned_byte_count);
      actual = MAP_FAILED;
    }
  }
  return actual;
}
#endif

```