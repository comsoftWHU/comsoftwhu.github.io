---
layout: default
title: FHE-compile
nav_order: 8
has_children: true
---

# 全同态加密
- https://zhuanlan.zhihu.com/p/149812445
- https://zhuanlan.zhihu.com/p/150920501
- https://zhuanlan.zhihu.com/p/156786436
- https://zhuanlan.zhihu.com/p/260033204

# 全同态加密编译器

随着云计算服务的不断广泛采用，客户要求服务提供商保证其数据的安全和隐私。全同态
加密（FHE）是一种密码技术，由于服务器端仅有权访问加密后的数据，因此可提供强大的安全保证。
FHE 迄今为止尚未得到广泛应用，主要原因有两个：计算成本仍然太高而无法实用，且开发FHE应用程序需要大量的密码专业知识。幸运的是，由于硬件加速、高效优化和底层实现等方面的重大进展，在过去的几年里，FHE的计算量已经大大减少。然而，FHE的广泛采用仍然亟需友好的开发工具来允许
没有密码学专业知识的软件开发人员可以将FHE集成到他们的应用程序中。

Google开源了一个面向全同态加密的C++转译器——[Transpiler](https://github.com/google/fully-homomorphic-encryption/tree/main/transpiler)。FHE C++ Transpiler 是一项开源技术，允许任何 C++ 开发人员直接对加密数据进行操作。

该转译器将 Google 的 XLS库连接到多个FHE后端（当前是TFHE 库和 OpenFHE的BinFHE 库）。它将允许开发人员（包括那些没有密码学专业知识的软件开发人员）编写在加密数据上进行操作的代码，而不会泄露数据内容或计算结果。该系统应该有助于为实用 FHE 系统的进一步发展奠定基础。

该系统目前仅支持 Linux，并且需要 GCC 版本 9（或更高版本）和 Bazel 4.0.0。

目前该项目一个探索性的概念验证。虽然该项目可以在实践中部署，但 FHE-C++ 的运行时间可能太长，以至于不实用。该转译器严重依赖所选的FHE 库来保证安全。由于 OpenFHE 和 TFHE 库都相对较新，因此还没有针对它们的强大的、广泛接受的密码分析。因此，在将该系统的输出集成到实时生产部署之前，请注意这两个库中可能存在尚未发现的漏洞。