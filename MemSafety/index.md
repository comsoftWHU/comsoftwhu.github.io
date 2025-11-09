---
layout: default
title: 内存安全
nav_order: 6
has_children: true
---


## 软件插桩类（Sanitizers）

* **ASan（AddressSanitizer）**
  Shadow Memory + Redzone + Quarantine 检测 **OOB/UAF/DF/栈溢出**；CPU~1.5–3×、内存~1.5–2×；开发/测试期常用。
* **HWASan（Hardware-assisted ASan）**
  AArch64 基于 **TBI** 给指针/内存打随机 tag（16 种），**概率**发现 OOB/UAF；比 ASan 更省内存；仅 64 位 ARM 可用。
* **UBSan（UndefinedBehaviorSanitizer）**
  捕获 **未定义行为**（整数溢出、错位类型转换、越界访问等）。
* **TSan（ThreadSanitizer）**
  侦测 **数据竞争/死锁**。
* **MSan（MemorySanitizer）**
  侦测 **未初始化读**。
* **KASAN（Kernel ASan）**
  内核版 ASan。

## 硬件/架构级防护

* **MTE（Memory Tagging Extension）**
  Armv8.5-A：地址顶字节携带 **4-bit tag**，内存以 **16B granule** 存 tag；访问时硬件比对。**同步/异步** 两类模式。
* **PA/PAC（Pointer Authentication）**
  用密钥对指针/返回地址签名（如 **PACIASP/RETAA**）；阻断 ROP/劫持，但**不检测**越界/悬挂。
* **CFI（Control-Flow Integrity）**
  约束间接跳转目标（编译期 + 运行时检查）；降低 JOP/ROP 成功率；需 LTO/类型信息较完整。
* **SCS（Shadow Call Stack）**
  返回地址保存在独立只写栈；抵御覆盖返回地址类攻击。
* **BTI（Branch Target Identification）**
  标记合法落点（BTI 指令）；拒绝跳入指令中间；与 PAC 协同提升控制流安全。

## 平台/机制相关概念

* **TBI（Top Byte Ignore）**
  AArch64 忽略虚拟地址顶字节，用作 **tag**（HWASan/MTE 的前置条件）。
* **Shadow Memory**
  将真实内存按比例映射到“影子区”存放 **poison/元信息**（ASan/KASAN）。
* **Redzone / Poison / Unpoison**
  分配边界留缓冲区并置毒；访问命中即报错；释放后延迟回收到 **Quarantine**。
* **Granule（MTE/HWASan 粒度）**
  标记的最小内存单位，MTE/HWASan 典型为 **16 字节**。
* **同步/异步（MTE）**
  同步：错误在访问点抛出；异步：不立即抛错。
* **Interceptor**
  对 `memcpy/malloc/pthread` 等库函数的替换/封装，让运行时能追踪分配与访问。
* **ASLR / NX(W^X)**
  地址随机化、可执行/可写互斥的基础防护。

## 常见缺陷名词

* **OOB（Out-of-Bounds）**：越界读写（数组/指针算术错误）。
* **UAF（Use-After-Free）**：释放后继续使用。
* **DF（Double Free）**：重复释放同一块内存。
* **Leak（内存泄漏）**：分配未释放导致增长。
* **Race（数据竞争）**：多线程未同步导致并发错误。
