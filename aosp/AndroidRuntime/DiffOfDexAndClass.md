---
layout: default
title: Dex文件和Class文件的区别
nav_order: 3
parent: AndroidRuntime
grand_parent: AOSP
author: Anonymous Committer
---
# .class文件

## 目标定位与运行环境
面向通用 Java 虚拟机（JVM），由 javac 编译生成，每个 .class 文件对应一个 Java 类（或接口）

## 执行模型
JVM 是基于栈的虚拟机，只能通过操作栈访问操作数

##  文件结构

<https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html>

# .dex
Android Runtime字节码文件，可以由d8等工具将若干 .class 转换打包而成。一个 APK 中通常只有一个或多个 classes.dex，内部包含所有类。运行前可进一步编译为 OAT/ODEX文件。


## 执行模型


dalvik虚拟机是基于寄存器的虚拟机。与class基于操作数栈的模型不一样。这里的寄存器是虚拟寄存器，与物理寄存器的概念不一样，在每个方法的描述中有这个方法会用到的虚拟寄存器的个数。

### 寄存器

寄存器机器减少内存访问、执行跳转更直接。

当用于位值（例如整数和浮点数）时，寄存器会被视为宽度为 32 位。如果值是 64 位，则使用两个相邻的寄存器。对于寄存器对，没有对齐要求。

当用于对象引用时，寄存器会被视为其宽度正好能够容纳一个此类引用。


### 指令
Class文件的指令以一个字节为一个单位（一个字节的操作码后面根操作数）
Dex文件的指令以2个字节为一个单位（操作码也是一个字节，但是操作数的分布比较复杂）

### 帧结构


帧的大小在创建时确定后就固定不变。每一帧由特定数量的寄存器（由方法指定）以及执行该方法所需的所有辅助数据构成。

#### 解释器模式

runtime/interpreter/shadow_frame.h 定义了 ShadowFrame，并且解释器入口函数 EnterInterpreterFromEntryPoint 就会接收一个 ShadowFrame* 参数来执行字节码。

```cpp
  ShadowFrame* link_; // 上一个帧
  ArtMethod* method_; // 当前方法数据结构ArtMethod的指针
  LockCountData lock_count_data_;  // This may contain GC roots when lock counting is active.
  const uint32_t number_of_vregs_; // 寄存器个数

  uint32_t dex_pc_; 

  // This is a set of ShadowFrame::FrameFlags which denote special states this frame is in.
  // NB alignment requires that this field takes 4 bytes no matter its size. Only 7 bits are
  // currently used.
  uint32_t frame_flags_;

  // This is a two-part array:
  //  - [0..number_of_vregs) holds the raw virtual registers, and each element here is always 4
  //    bytes.
  //  - [number_of_vregs..number_of_vregs*2) holds only reference registers. Each element here is
  //    ptr-sized.
  // In other words when a primitive is stored in vX, the second (reference) part of the array will
  // be null. When a reference is stored in vX, the second (reference) part of the array will be a
  // copy of vX.
  uint32_t vregs_[0];
```

#### Quick (AOT/JIT) 模式下

方法调用会走quick stubs和本地栈帧，并不分配或使用 ShadowFrame 对象。

在 runtime/entrypoints/quick、runtime/art_method.cc 中通过 OatQuickMethodHeader, callee-save 信息，以及平台原生栈布局来完成，完全靠机器栈和寄存器存储上下文，不涉及ShadowFrame。

只有在deopt时，ART 才会将这些 Quick 帧转成 Interpreter 模式的 ShadowFrame，以便继续在解释器里执行或单步调试。

## 文件结构

<https://source.android.google.cn/docs/core/runtime/dex-format>

dex文件有header，header中有string\type\proto\field\method\class\data字段的大小和偏移
主要区别在于因为一个dex文件包含很多类，所以字符串去重效果会更好，使用LEB128变长编码提高整型存储效率，然后方法会有比class文件更短的shorty的描述符，总的来说存储效率更高。
