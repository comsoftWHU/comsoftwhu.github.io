---
layout: default
title: 安卓延迟绑定?
nav_order: 4
parent: AOSP
author: Anonymous Committer
---


Android 不支持 RTLD_LAZY，总是用 RTLD_NOW，这样可以在 dlopen 返回前就把所有 undefined 符号解析完，从而立刻对 PLT 做 RELRO 保护。

## 延迟绑定

符号的地址不是在程序启动或库加载时一次性解析完，而是等到“第一次真的调用它”时再去解析。

（动态链接器层面的 lazy binding和面向对象里虚函数的lazy binding不是一回事）

<https://www.openeuler.org/en/blog/lijiajie128/2020-11-10-%E5%8A%A8%E6%80%81%E9%93%BE%E6%8E%A5%E4%B8%AD%E7%9A%84PLT%E4%B8%8EGOT>

### 绑定

在使用动态库时，编译好的程序里会有很多“外部符号引用”，比如：

```c
extern int puts(const char*);
...
puts("hello");
```

二进制里，这个调用点并不知道 `puts` 的真实地址，只知道“要调一个叫 puts 的符号”。

**绑定（binding）** = 把这个“符号引用”真正变成一个 **具体地址**：

* 找到这个符号在哪个 so 里
* 算出它在内存中的真实地址
* 把这个地址写到合适的位置

什么时候做这件事，就有两种策略：

* 立即绑定：程序 / so 一加载完，所有相关符号一次性解析、重定位。
* 延迟绑定：先不全解析，只在第一次调用某个函数时，才为这个函数做解析和重定位。

### 为什么要延迟

主要是为了减少启动时开销：

* 动态库很多，符号也很多，全部一次性解析会拖慢启动。
* 有些函数可能整个运行过程中一次都不会用到，提前给它们做重定位是浪费。

所以 ELF 的动态链接机制设计了一个优化：**函数调用的重定位可以等到第一次真正调用时再做**

### 怎么实现

延迟绑定主要就是围绕：`.plt`,`.got.plt` + 动态段 `.dynamic` 里的几类 `DT_*` 项 + PLT 重定位表 `.rel(a).plt`（类型是 `R_*_JUMP_SLOT`）+ 符号表 `.dynsym`, `.dynstr` 这一整套东西在协作。

关键角色有两个：

* **PLT（Procedure Linkage Table，过程链接表）**
* **GOT（Global Offset Table，全局偏移表）**

编译器不会直接 `call puts@绝对地址`，而是：

1. 对外部函数，生成 `call puts@PLT`，跳到当前模块的一个 **PLT stub**。
2. 每个外部函数会有一个 GOT 项，对应这个函数真正的地址。
<!-- 
#### PLT/GOT 相关的 `DT_*`

##### DT_PLTGOT

指向与 PLT 相关的 GOT 区域，典型就是 .got.plt 的起始地址。

动态链接器通过这个指针，知道“PLT/GOT 跳板”在哪；

##### DT_JMPREL

指向 PLT 重定位表（`.rel.plt` 或 `.rela.plt`）；

这些重定位条目就是“函数跳转槽”（`JUMP_SLOT`）对应的 relocation。

##### DT_PLTREL

表示 DT_JMPREL 指向的是 Rel 还是 Rela 表（DT_REL / DT_RELA）；

###### DT_PLTRELSZ

PLT 重定位表的总大小（字节），用来算出条目个数。 -->

### ELF 源文件视角

顶层 ELF 文件

```mermaid
flowchart LR
    ELF[ELF 文件] --> PHDR[Program Header 表]
    ELF --> SECT["Section 表（.plt / .got.plt / .dynsym / .dynstr / .rel.plt 等）"]
```

运行时动态链接器首先要通过 `PT_DYNAMIC` 找到内存里的 `.dynamic`，再从里面的 `DT_PLTGOT` / `DT_JMPREL` / `DT_PLTREL` / `DT_PLTRELSZ` 等信息，定位 `.got.plt` 和 `PLT` 重定位表，最后才能对 `.got.plt` 里的条目做修改。

Program Header 视角

```mermaid
flowchart LR
    PHDR[Program Header 表] --> PT_DYNAMIC[PT_DYNAMIC 段<br/>指向 .dynamic]
    PT_DYNAMIC --> DYNAMIC[".dynamic<br/>(ElfW(Dyn) 数组)"]

    %% .dynamic 里的关键 DT_* 项
    subgraph DYN[".dynamic 动态段（DT_*）"]
        DYNAMIC --> DT_PLTGOT[DT_PLTGOT<br/>= .got.plt 起始地址]
        DYNAMIC --> DT_JMPREL["DT_JMPREL<br/>= .rel[a].plt 起始地址"]
        DYNAMIC --> DT_PLTREL[DT_PLTREL<br/>= .rel / .rela 类型]
        DYNAMIC --> DT_PLTRELSZ["DT_PLTRELSZ<br/>= .rel[a].plt 大小"]
        DYNAMIC --> DT_SYMTAB[DT_SYMTAB<br/>= .dynsym 起始地址]
        DYNAMIC --> DT_STRTAB[DT_STRTAB<br/>= .dynstr 起始地址]
        DYNAMIC --> DT_STRSZ[DT_STRSZ<br/>= 字符串表大小]
        DYNAMIC --> DT_HASH[DT_HASH / DT_GNU_HASH<br/>= 符号 hash 表]
        DYNAMIC --> DT_FLAGS[DT_FLAGS / DT_FLAGS_1<br/>含 DF_BIND_NOW/DF_1_NOW<br/>控制是否允许 lazy]
    end
```

Section 视角

```mermaid
flowchart LR
    %% Section 视角：和 lazy binding 直接相关的区
    subgraph SECTIONS[ELF Sections]
        PLT[.plt / .plt.sec<br/>PLT 入口代码（foo@plt, PLT0）]
        GOTPLT[.got.plt<br/>PLT 使用的 GOT 槽]
        RELPLT[.rel.plt / .rela.plt<br/>PLT 重定位表<br/>R_*_JUMP_SLOT]
        DYNSYM[.dynsym<br/>动态符号表]
        DYNSTR[.dynstr<br/>符号名字符串表]
        HASH[.hash / .gnu.hash<br/>符号 hash 表]
    end


    %% lazy binding 动态行为（概念连线）
    subgraph RUNTIME[运行时]
        CALL[调用 foo@plt] --> PLT
        PLT --> GOTREAD[读取 .got.plt 中 foo 槽<br/>第一次：指向 PLT0 / 解析入口]
        GOTREAD --> PLT0[PLT0：统一入口<br/>跳入动态链接器]
        PLT0 --> RESOLVE[运行时动态链接器]
    end

    PLT -. 运行时执行 .-> RUNTIME
    GOTPLT -. 运行时读/写 .-> RUNTIME
    RELPLT -. 提供 JUMP_SLOT 重定位条目 .-> RESOLVE
    DYNSYM -. 符号表 .-> RESOLVE
    DYNSTR -. 符号名 .-> RESOLVE
    HASH -. 按名查符号加速 .-> RESOLVE
```

### 流程

```mermaid
flowchart TD
    A[程序执行<br/>call puts@PLT] --> B[进入 .plt 段中<br/>puts 对应的 PLT 入口]

    B --> C{"GOT[puts]<br/>是否已包含真实地址?"}

    %% 第一次调用（lazy 绑定路径）
    C -- 否，第一次调用 --> D[PLT 入口代码将<br/>重定位索引等信息压栈]
    D --> E[跳转到通用入口 PLT0]
    E --> F[PLT0 调用动态链接器<br/>_dl_runtime_resolve]
    F --> G[动态链接器查重定位表/符号表<br/>找到 puts 的真实地址]
    G --> H["将真实地址写回 GOT[puts]"]
    H --> I[跳转到真实的 puts 函数执行]

    %% 后续调用（已绑定）
    C -- 是，已解析过 --> J["从 GOT[puts]<br/>直接取出真实地址"]
    J --> I
```

## 安卓的延迟绑定

### TL;DR

Android 的 bionic 链接器直接**禁用** lazy binding, 因为 lazy binding 会阻碍像 RELRO 这样的加固特性。

### RELRO

RELRO（Relocation Read-Only）就是“把做完重定位后还需要保持不变的那块内存设成只读”，主要是保护 GOT 等表，防止被覆盖劫持控制流。

ELF 程序启动时，动态链接器要做各种重定位，在重定位完成之后理论上`GOT`类似的结构就不该再被改动。
但默认情况下，这些区段是可写的，攻击者一旦拿到任意写，就可以改 `GOT` 某个条目，然后劫持控制流。

RELRO 的核心思路：

* 让链接器把这些需要保护的区段单独放进一个段
* 程序启动时动态链接器先应用所有重定位
* 然后对这个段 `mprotect(PROT_READ)`

<https://www.redhat.com/ja/blog/hardening-elf-binaries-using-relocation-read-only-relro>

### Bionic 实现

<https://cs.android.com/android/platform/superproject/main/+/main:bionic/libc/include/dlfcn.h;l=188>

在头文件`bionic/libc/include/dlfcn.h`这里定义了 `dlopen` 的 flag（`RTLD_NOW`, `RTLD_LAZY` 等）。Android 在注释里直接写了一直使用 `RTLD_NOW` 出于安全原因，不支持 `RTLD_LAZY`。

API 层把 `RTLD_LAZY` 这个宏暴露出来是为了兼容 POSIX，但实现上直接当作 `RTLD_NOW` 处理。

```cpp
/**
 * A dlopen() flag to not make symbols from this library available to later
 * libraries. See also RTLD_GLOBAL.
 */
#define RTLD_LOCAL    0

/**
 * Not supported on Android. Android always uses RTLD_NOW for security reasons.
 * Resolving all undefined symbols before dlopen() returns means that RELRO
 * protections can be applied to the PLT before dlopen() returns.
 */
#define RTLD_LAZY     0x00001

/** A dlopen() flag to resolve all undefined symbols before dlopen() returns. */
#define RTLD_NOW      0x00002

/**
 * A dlopen() flag to not actually load the given library;
 * used to test whether the library is already loaded.
 */
#define RTLD_NOLOAD   0x00004

/**
 * A dlopen() flag to make symbols from this library available to later
 * libraries. See also RTLD_LOCAL.
 */
#define RTLD_GLOBAL   0x00100

/**
 * A dlopen() flag to ignore later dlclose() calls on this library.
 */
#define RTLD_NODELETE 0x01000
```

在`bionic/libdl/libdl.cpp`中

```cpp
__attribute__((__weak__))
void* dlopen(const char* filename, int flag) {
  const void* caller_addr = __builtin_return_address(0);
  return __loader_dlopen(filename, flag, caller_addr);
}
```

dlopen调用`bionic/linker/dlfcn.cpp`中的

```cpp
void* __loader_dlopen(const char* filename, int flags, const void* caller_addr) {
  return dlopen_ext(filename, flags, nullptr, caller_addr);
}

// .....

static void* dlopen_ext(const char* filename,
                        int flags,
                        const android_dlextinfo* extinfo,
                        const void* caller_addr) {
  ScopedPthreadMutexLocker locker(&g_dl_mutex);
  g_linker_logger.ResetState();
  void* result = do_dlopen(filename, flags, extinfo, caller_addr);
  if (result == nullptr) {
    __bionic_format_dlerror("dlopen failed", linker_get_error_buffer());
    return nullptr;
  }
  return result;
}
```

`do_dlopen`是核心执行代码，在`bionic/linker/linker.cpp`中

<https://cs.android.com/android/platform/superproject/main/+/main:bionic/linker/linker.cpp;l=2116>

大致的流程如图

```mermaid
flowchart TD
    A["调用 do_dlopen(name, flags, extinfo, caller_addr)"] --> B["find_containing_library(caller_addr) <br> "]
    B --> C[get_caller_namespace]
    C --> D["translateSystemPathToApexPath <br/> if SDK < 29"]
    D --> E["find_library(namespace, translated_name, flags, extinfo, caller)"]

    %% find_library / find_libraries 内部
    subgraph LOAD_LINK
        E --> F{"soinfo 已存在(是否已加载)?"}
        F -- 是，增加引用计数 --> G[返回已存在的 soinfo 指针]
        F -- 否，需要加载 --> H[find_libraries<br/>解析依赖 & 装载 ELF]

        H --> H1[通过 open/mmap 加载 ELF 映像<br/>填充 soinfo 结构]
        H1 --> H2["soinfo::prelink_image()<br>解析 .dynamic, 构建重定位表"]
        H2 --> H3["soinfo::link_image()<br/>重定位(包括 PLT/JUMP_SLOT)"]
        H3 --> H4["phdr_table_protect_gnu_relro()<br/>对 PT_GNU_RELRO 段 mprotect 只读"]
        H4 --> H5[更新依赖关系 & 引用计数]
        H5 --> G
    end

    %% 回到 do_dlopen
    G --> I{soinfo 是否为 null?}
    I -- 是，加载/链接失败 --> J[设置 dlerror 信息<br/>返回 nullptr]
    I -- 否，加载/链接成功 --> K["si->call_constructors()<br/>执行 .preinit_array / .init_array 等构造器"]
    K --> L["返回 si->to_handle()<br/>dlopen() 返回句柄给调用方"]

```

### Linker Namespace

Android之前的状态大概是

* 整个进程里只有一个全局库空间，所有 so 都在一起；

* 任意 so 可以随便 `dlopen/DT_NEEDED` 依赖任何路径下的 so；

* vendor so 大量直接 link framework 的私有 so（libandroid_runtime.so 之类），
系统一升级符号就崩；同时还容易出现“同名不同 ABI 的 so 冲突”。

#### 解决办法

namespace 隔离: 每个 namespace 有独立的搜索路径, 允许加载的目录；
isolated=true 时，动态链接器会严格检查 so 路径，不在这些目录就不让装；只能对白名单中这些库名走跨 namespace 解析。

### soinfo

所有被加载的 native 库都会对应一个 soinfo 结构，里面记录这个库的名字、路径、ASLR 后的基址、所属 namespace 等信息。

#### `soinfo::prelink_image`

##### 核心作用

读取当前 so 的 PT_DYNAMIC 段，解析各种 DT_* 动态表项，把 soinfo 里和符号表、重定位、构造/析构、版本、TLS、Memtag 等相关的字段全部填好，并做一轮合法性校验，最后打上 FLAG_PRELINKED 标记。
真正“改内存、做重定位”的是在后面的 link_image()。

##### 源代码

 <https://cs.android.com/android/platform/superproject/main/+/main:bionic/linker/linker.cpp;l=2852>

##### 流程图

```mermaid
flowchart TD
    A["prelink_image()"] --> B{FLAG_PRELINKED?}
    B -- 是 --> B1[直接返回 true]
    B -- 否 --> C[通过 PHDR 找到 PT_DYNAMIC]

    C --> D{dynamic 为空?}
    D -- 是 --> D1[报错: 缺少 PT_DYNAMIC<br/>return false]
    D -- 否 --> E[解析 TLS / ARM_exidx 等附加信息]

    E --> F[第一轮遍历 .dynamic<br/>收集: STRTAB/SYMTAB/HASH<br/>REL/RELA/RELR/ANDROID_REL*<br/>INIT/FINI/数组构造器<br/>版本信息 DT_VER* 等]

    F --> G[做合法性检查<br/>确保 hash/strtab/symtab 等存在<br/>linker 不允许有 DT_NEEDED]

    G --> H[第二轮遍历 .dynamic<br/>解析 DT_SONAME / DT_RUNPATH]

    H --> I["兼容老 app: 无 SONAME 时用 basename(realpath_)"]

    I --> J[validate_verdef_section<br/>必要时处理 Memtag globals 重映射]

    J --> K[置 FLAG_PRELINKED<br/>return true]

```

#### `soinfo::link_image`

<https://cs.android.com/android/platform/superproject/main/+/main:bionic/linker/linker.cpp;l=3357>

link_image = 应用重定位 + 段保护（包括 RELRO）+ 一些调试/MTE/RELRO 共享的收尾工作

```mermaid
flowchart TD
    A["link_image()"] --> B{已标记为 linked?}
    B -- 是 --> B1[return true]
    B -- 否 --> C[设置 local_group_root_ 等<br/>必要时记录 target_sdk]

    C --> D{32 位 && has_text_relocations?}
    D -- 是 --> D1[发 warning/按 SDK 判错<br/>临时解除 PT_LOAD 段只读] --> E
    D -- 否 --> E[跳过 text reloc 解保护]

    E --> F{是否 vdso?}
    F -- 是 --> G["跳过 relocate()"] --> I
    F -- 否 --> H["调用 relocate(lookup_list)<br/>执行所有重定位"]
    H --> H1{成功?}
    H1 -- 否 --> H2[return false]
    H1 -- 是 --> I

    I --> J{32 位 && has_text_relocations?}
    J -- 是 --> J1["恢复 PT_LOAD 段保护(r-x/r--)"] --> K
    J -- 否 --> K

    K --> L{当前 so 是 linker 本体?}
    L -- 是 --> L1[跳过 protect_relro <br/>由 linker 启动后期再做] --> M
    L -- 否 --> L2["protect_relro()<br/>对 PT_GNU_RELRO 段 mprotect 只读"]
    L2 --> L3{成功?}
    L3 -- 否 --> L4[return false]
    L3 -- 是 --> M

    M --> N{需要 Memtag globals?}
    N -- 是 --> N1["name_memtag_globals_segments()"] --> O
    N -- 否 --> O

    O --> P{extinfo 要求共享/复用 RELRO?}
    P -- WRITE_RELRO --> P1[serialize_gnu_relro 到 fd]
    P -- USE_RELRO --> P2[map_gnu_relro 从 fd 映射]
    P1 --> P1e{成功?}
    P1e -- 否 --> P1f[return false]
    P2 --> P2e{成功?}
    P2e -- 否 --> P2f[return false]

    P1e -- 是 --> Q
    P2e -- 是 --> Q
    P -- 否 --> Q[不做 RELRO 共享]

    Q --> R[模块计数++，notify_gdb_of_load]
    R --> S[标记 is_image_linked<br/>return true]

```

#### `soinfo::relocate`

<https://cs.android.com/android/platform/superproject/main/+/main:bionic/linker/linker_relocate.cpp;l=592>

```mermaid
flowchart TD
    A["relocate()"] --> B{g_is_ldd?}
    B -- 是 --> B1[仅做依赖分析<br/>不打 reloc<br/>return true]
    B -- 否 --> C[初始化 VersionTracker]

    C --> C1{init 失败?}
    C1 -- 是 --> C2[return false]
    C1 -- 否 --> D[构造 Relocator<br/>设置 si/strtab/symtab<br/>TLS 相关字段]

    D --> E{存在 RELR && 不是 linker?}
    E -- 是 --> E1["relocate_relr()<br/>应用压缩 RELR 相对重定位"]
    E1 --> E1e{成功?}
    E1e -- 否 --> E1f[return false]
    E1e -- 是 --> F
    E -- 否 --> F

    F --> G{存在 ANDROID_REL/RELA?}
    G -- 是 --> G1["检查 'APS2' magic<br/>packed_relocate&lt;Typical&gt;()"] 
    G1 --> G1e{成功?}
    G1e -- 否 --> G1f[return false]
    G1e -- 是 --> H
    G -- 否 --> H

    H --> I{USE_RELA?}
    I -- 是 --> I1["plain_relocate&lt;Typical&gt;() 处理 .rela.dyn<br/>plain_relocate&lt;JumpTable&gt;() 处理 .rela.plt"]
    I -- 否 --> I2["plain_relocate&lt;Typical&gt;() 处理 .rel.dyn<br/>plain_relocate&lt;JumpTable&gt;() 处理 .rel.plt"]

    I1 --> I1e{成功?}
    I1e -- 否 --> I1f[return false]
    I2 --> I2e{成功?}
    I2e -- 否 --> I2f[return false]

    I1e -- 是 --> J
    I2e -- 是 --> J

    J --> K{架构为 arm64/riscv64?}
    K -- 是 --> K1[遍历 deferred_tlsdesc_relocs<br/>填充 TlsDescriptor.func/arg] --> L
    K -- 否 --> L

    L[return true]

```

### lazy binding?

* 头文件里说明：RTLD_LAZY 在 Android 上“不被支持”，行为等价于 RTLD_NOW；传这个 flag 只是为了兼容 POSIX 接口签名。

* prelink_image 里：DT_PLTGOT 明确被Ignored，链接器不会利用 PLT/GOT 机制去搭建 _dl_runtime_resolve 之类的 lazy 路径。

* relocate() 里：所有 `.rel[a].plt` 条目都走 `plain_relocate<RelocMode::JumpTable>`，`process_relocation_impl()` 会立刻查符号并把真实地址写进 GOT 槽，不保留“跳回 PLT0”的中间状态。

* link_image() 里：relocate() 结束后立即对 PT_GNU_RELRO 做 mprotect，只读，把包含 GOT/PLT 在内的一整块内存锁死，后续运行期就算想改 GOT 也改不了了，更不可能再 lazy binding。
