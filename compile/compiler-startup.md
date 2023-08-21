---
layout: default
title: 编译器学习材料
nav_order: 1
parent: 编译
author: qingan
---
{% assign author = site.data.authors[page.author] %}
<div> 作者: {{ author.name }}  
 邮箱：{{ author.email }}
</div>


## 编译技术概览
- 理论：本科专业课程《编译原理》([龙书](https://gitee.com/li-qingan/cs-docs/tree/master/ebooks/DragonBook2nd.pdf))
- 实践：
    - [LLVM项目](https://llvm.org)
    - [LLVM cook book](https://gitee.com/li-qingan/cs-docs/tree/master/ebooks/llvmCookBook.pdf)
    - [LLVM讨论资料](https://gitee.com/li-qingan/cs-docs/llvmSlides)

## 前端
- 理论：龙书第2章的例子 ([龙书](https://gitee.com/li-qingan/cs-docs/tree/master/ebooks/DragonBook2nd.pdf))，or 虎书前端部分
- 实践：
    - [LLVM万花筒项目](https://llvm.org/docs/tutorial/)
    - [antrl项目](https://github.com/antlr)

## 中端及优化
- 理论：
    - 数据流分析（龙书第九章）
    - 南京大学《软件分析》（B站有视频）
- 实践
    - [LLVM项目](https://www.llvm.org/docs/Passes.html)

## 后端
- 理论：龙书第8章（代码生成、寄存器分配）、第10章（指令调度）
- 实践：[LLVM CPU0](https://jonathan2251.github.io/lbd/llvmstructure.html)

## 链接过程
- 《深入理解计算机系统》第7章([csapp book](https://gitee.com/li-qingan/cs-docs/tree/master/ebooks/csappBook.pdf))

## 运行时
### 静态内存分配
- [寄存器分配](https://gitee.com/li-qingan/cs-docs/tree/master/ebooks/regAlloc.pptx)

### 动态内存分配
- 《深入理解计算机系统》第9章 ([csapp book](https://gitee.com/li-qingan/cs-docs/tree/master/ebooks/csappBook.pdf))
- 《编译原理》运行时相关章节，龙书第7章