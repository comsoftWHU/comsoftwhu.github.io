---
layout: default 
title: torch.fx源码剖析 01 - 简介
nav_order: 0 
parent: torch.fx 
grand_parent: PyTorch
grand_grand_parent: AI编译
author: DimancheH 
---

{% assign author = site.data.authors[page.author] %}

 <div> 
     作者: {{ author.name }}   邮箱：{{ author.email }}
 </div>


# torch.fx源码剖析 01 - 简介

`torch.fx` 用于对 `torch.nn.Module` 做图变换。它包含三部分：

- 符号跟踪器（symbolic tracer）：用于捕获 module 的语义，它以符号的方式执行 Python 代码（symbolic execution），通过给 module 提供虚假值（Proxies，也就是placeholder）并记录涉及到的运算；
- 中间表示（intermediate representation）：IR 是在 tracing 期间所记录算子的图，包含一系列节点（Node），节点代表输入（placeholder）、函数（get_attr, call_function, call_module, call_method）、输出（output），IR 是用 `torch.fx` 进行图变换（transformation）的基石；
- Python 代码生成（code generation）：直接生成 Python 代码使得 `torch.fx` 可以进行 Python-Python 或 Module-Module 的一一变换，此功能包含在 `torch.fx.GraphModule` 中，它是 `torch.nn.Module` 的实例，并保留有 `torch.fx.Graph`；

总而言之，`torch.fx` 的流程是：符号跟踪，中间表示，代码变换，Python 代码生成。

FX与dynamo的关系：dynamo将动态图切分成静态图，静态图交由FX解决。