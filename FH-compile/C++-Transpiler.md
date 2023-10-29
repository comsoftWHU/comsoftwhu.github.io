---
layout: default
title: C++ Transpiler
nav_order: 2
parent: FH-compile
author: qingan
---

Google开源了一个面向全同态加密的C++转译器——[Transpiler](https://github.com/google/fully-homomorphic-encryption/tree/main/transpiler)。FHE C++ Transpiler 是一项开源技术，允许任何 C++ 开发人员直接对加密数据进行操作。
- [项目链接](https://github.com/google/fully-homomorphic-encryption/tree/main/transpiler)
- [论文链接](https://arxiv.org/abs/2106.07893)

该转译器将 Google 的 XLS库连接到多个FHE后端（当前是TFHE 库和 OpenFHE的BinFHE 库）。它将允许开发人员（包括那些没有密码学专业知识的软件开发人员）编写在加密数据上进行操作的代码，而不会泄露数据内容或计算结果。该系统应该有助于为实用 FHE 系统的进一步发展奠定基础。

该系统目前仅支持 Linux，并且需要 GCC 版本 9（或更高版本）和 Bazel 4.0.0。

目前该项目一个探索性的概念验证。虽然该项目可以在实践中部署，但 FHE-C++ 的运行时间可能太长，以至于不实用。该转译器严重依赖所选的FHE 库来保证安全。由于 OpenFHE 和 TFHE 库都相对较新，因此还没有针对它们的强大的、广泛接受的密码分析。因此，在将该系统的输出集成到实时生产部署之前，请注意这两个库中可能存在尚未发现的漏洞。

## 当前支持的demo
### 1. 计算器
这个演示例子可以直接对两个加密后的短整型进行加法、减法或乘法运算，因此服务器不需要知道原始的加数，以及最终的结果。
- 基础的FHE-C++版本的翻译命令：
    - 使用TFHE：shell bazel run //transpiler/examples/calculator:calculator_tfhe_testbench
    - 使用OpenFHE：shell bazel run //transpiler/examples/calculator:calculator_openfhe_testbench
- 多核版本的解释器命令：
    - 使用TFHE：shell bazel run //transpiler/examples/calculator:calculator_interpreted_tfhe_testbench
    - 使用OpenFHE：shell bazel run //transpiler/examples/calculator:calculator_interpreted_openfhe_testbench

