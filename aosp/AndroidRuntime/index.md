---
layout: default
title: AndroidRuntime
nav_order: 2
parent: AOSP
has_children: true
---

## 安卓运行时

安卓运行时（Android Runtime, ART）是安卓操作系统中专门为应用程序设计的核心运行环境，它负责将应用的Java或Kotlin代码编译成设备可以直接执行的原生指令。与早期的Dalvik虚拟机不同，ART主要采用预先编译（AOT）和即时编译（JIT）相结合的策略，在应用安装或运行时将其DEX字节码高效地转换为本地机器码，从而显著提升了应用的启动速度、运行性能和电池续航。除了执行代码，ART还全面负责内存管理（如垃圾回收GC）、线程调度以及系统服务的交互，是支撑所有安卓应用运行的基石。

## 顶层总览

```mermaid
flowchart LR
  INP["Input 应用字节码"]
  LNK["Link 类加载与解析"]
  EXE["Exec执行"]
  PGO["PGO优化"]
  AOT["AOT编译"]
  MEM["Memory,GC"]
  NAT["Native Code, JNI"]

  INP -->|"加载dex"| LNK
  LNK -->|"解析,验证"| EXE

  EXE <-->|"对象分配,GC"| MEM
  EXE -->|"热点触发,OSR"| AOT
  EXE -->|"热点计数,画像数据"| PGO

  PGO -->|"指导 JIT"| EXE
  PGO -->|"指导 dex2oat 编译"| AOT

  AOT -->|"产出 oat/odex 与 image"| LNK
  AOT -->|"提供 Quick Code 供执行"| EXE

  EXE <-->|"JNI 调用"| NAT
```

### 执行引擎Exec（执行引擎）

```mermaid
flowchart LR
  subgraph "Exec（执行引擎）"
    INT["Interpreter（mterp）"]
    JIT["JIT Compiler（热点 / OSR）"]
    CODEC["Code Cache（已编译代码）"]
    QCODE["Quick Code(AOT/JIT)"]
    NAT["Native Code（JNI）"]
  end

  INT -->|"热点阈值 / OSR 触发"| JIT
  JIT -->|"编译产物写入"| CODEC -->|"入口桩 / 回边"| QCODE

  QCODE -->|"deopt(异常慢路径,去优化)"| INT

  INT -->|"JNI 调用"| NAT
  QCODE -->|"JNI"| NAT
  NAT -->|"返回到解释帧"| INT
  NAT -->|"返回到已编译帧"| QCODE
```


### 类加载与解析Link（类加载与解析）

```mermaid
flowchart LR
  subgraph "Link（类加载与解析）"
    CL["ClassLoader 栈"]
    DEXF["DexFile（.dex）"]
    VDX["vdex校验"]
    DEXC["DexCache"]
    CLINK["ClassLinker解析连接初始化"]
    VERI["Verifier"]
  end

  DEXF -->|"打开"| CL
  VDX -->|"校验"| CL
  CL -->|"请求解析"| CLINK
  CLINK -->|"填充解析缓存"| DEXC
  CLINK -->|"触发验证"| VERI -->|"通过 / 拒绝"| CLINK
```

---

### AOTdex2oat（编译与镜像）

```mermaid
flowchart LR
  subgraph "AOT（编译与镜像）"
    D2O["dex2oat（安装期，后台）"]
    PROF[PGO 指导]
    OAT["Quick Code，元数据"]
    IMG["app image / boot image"]
    REL["relocation"]
  end

  PROF -->|"方法选择 / 内联 / 热路径"| D2O
  D2O -->|"产出"| OAT
  D2O -->|"产出"| IMG
  OAT -->|"需要时重定位"| REL
  IMG -->|"预热加载（Zygote / App）"| REL
```

### PGO画像与优化闭环

```mermaid
flowchart LR
  subgraph "PGO（画像与优化）"
    HITS["运行时热点计数 / 采样"]
    SAVER["ProfileSaver"]
    BASE["baseline profile随APK"]
    BOOT["boot profile（系统侧）"]
  end

  HITS -->|"写入画像数据"| SAVER
  SAVER -->|"反馈阈值 / 内联 / OSR 策略"| HITS

  SAVER -->|"输入"| D2O["dex2oat"]
  BASE -->|"输入"| D2O
  BOOT -->|"输入"| D2O

  SAVER -->|"动态阈值 / 热点集"| JIT["JIT Compiler"]
```

---

### 内存与 GCMemory（内存 & GC）

```mermaid
flowchart LR
  subgraph "Memory（内存 & GC）"
    IMGSP["Image Space"]
    ZYGSP["Zygote Space（共享）"]
    ALOSP["Alloc Space:Ros,碰撞指针"]
    LOS["Large Object Space"]
    REG["Region Space"]
    GC["GC(CC/CMC,读写屏障)"]
  end

  ALOSP -->|"常规分配"| GC
  LOS -->|"大对象跟踪"| GC
  REG -->|"并发回收 / 压缩"| GC
  ZYGSP -->|"子进程 COW 复用"| IMGSP
```

---

### Native JNI

```mermaid
flowchart LR
  subgraph "Native Code（JNI）"
    JNIB["JNI 桥"]
    CRIT["Critical Native"]
    LIBS[".so 本地库"]
    INTR["intrinsics（内建本地实现）"]
  end

  JNIB -->|"JNIEnv,局部引用"| LIBS
  CRIT -->|"绕过部分 JNI 开销（受限）"| LIBS
  INTR -->|"被 Quick Code 内联调用"| LIBS
```
