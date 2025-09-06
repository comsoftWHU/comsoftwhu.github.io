---
layout: default
title: dex2oat
nav_order: 0
parent: AndroidRuntime
has_children: true
---

`dex2oat` 用于将 Android 应用的 DEX (Dalvik Executable) 文件转换为 OAT (Optimized Android executable) 文件，从而提升应用在 Android 设备上的执行效率。这个转换过程是 Android 在启动应用时进行的一个重要步骤，通常是在安装应用时或在设备空闲时进行。

1. **优化性能**：`dex2oat` 将 DEX 字节码转换成机器代码，并且进行优化，减少了运行时的解释执行，从而提升应用的启动速度和运行效率。

2. **预编译**：通过提前编译 DEX 文件，`dex2oat` 减少了运行时的开销，使得应用在执行时不再需要经过 JIT。这也有助于节省电池电量，尤其是在设备的低功耗模式下。

3. **OAT 文件**：生成的 OAT 文件包含了经过优化的代码，这些代码是为特定设备架构定制的二进制代码，能够直接在设备上运行。OAT 文件被存储在设备的文件系统中，而原始的 DEX 文件则通常会被压缩或去除。
