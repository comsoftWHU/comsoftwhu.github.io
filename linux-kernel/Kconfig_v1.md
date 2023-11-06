---
layout: default
title: Kconifg_v1
nav_order: 3
parent: Linux内核
author: plot
---

本文介绍Kconfig的基础知识，故名Kconfig_v1。有关Kconfig的进阶知识会以v2，v3的形式发布。本文的例子基于Linux内核5.10.176。

## Kconfig简介

Kconfig 是 Linux 内核中的配置系统，用于管理和配置内核的各种功能、选项和设备驱动程序。**它提供了一种方便的方式来定制和构建适合特定需求的内核映像**。

Kconfig 系统由以下几个核心组件组成：

1. Kconfig 文件：Kconfig 文件是内核配置的源文件，用于定义各个子系统、选项和功能的配置信息。Kconfig 文件使用类似于 Makefile 的语法和结构，它包含了一系列配置项、菜单以及对应的依赖关系和默认值。
2. 菜单配置工具：Kconfig 提供了一系列菜单配置工具，例如 `make menuconfig`、`make xconfig`、`make gconfig` 等，用于通过图形界面或文本界面的方式进行配置。这些工具使得用户可以方便地在各个菜单选项之间导航、选择和修改配置。
3. 构建系统集成：Kconfig 与内核构建系统（如 Makefile）紧密集成，允许在编译过程中自动应用配置。当用户选择和保存某个配置时，**Kconfig 会生成对应的配置文件（`.config`）**，然后构建系统会根据该配置文件来生成内核映像，只包含用户选择的功能和设备驱动。

Kconfig 系统的优点包括：

- 灵活性：Kconfig 提供了丰富的配置选项，允许用户根据需求定制内核映像。用户可以选择需要的子系统、设备驱动和功能，以及调整各种编译选项来满足特定的需求。
- 可维护性：Kconfig 使用模块化的配置文件结构，使得内核配置更易于管理和维护。每个子系统和功能都有对应的配置文件，有助于组织和维护大规模的配置。
- 用户友好性：Kconfig 提供了多种菜单配置工具，包括图形界面和文本界面，使得配置过程更加直观和易于使用。用户可以通过简单的交互方式完成复杂的配置任务。

总而言之，Kconfig 是 Linux 内核中的强大配置系统，为用户提供了灵活、可维护和用户友好的方式来定制和构建内核映像。它是构建高度定制化的嵌入式系统和服务器系统的重要工具。

## Kconfig简单语法

以下介绍Kconfig的常用语法，为理解后面的联系Makefile和启动流程奠定基础。完整的Kconfig语法可参考内核源码中的`Documentation/kbuild/kconfig-language.rst.` 或者[Kconfig语法](https://docs.kernel.org/kbuild/kconfig-language.html)。

### config条目

```kconfig
config CFG80211
    tristate "cfg80211 - wireless configuration API"
    depends on RFKILL || !RFKILL
    select FW_LOADER
    select CRC32
    select CRYPTO_SHA256 if CFG80211_USE_KERNEL_REGDB_KEYS
    help
      cfg80211 is the Linux wireless LAN (802.11) configuration API.
      Enable this if you have a wireless device.
```
**解析：**
* config是关键字，表示一个配置选项的开始，下面几行定义了该config选项的属性。属性一般包括config类型，默认值，依赖关系，help说明；CFG80211是配置选项的名称，省略了前缀"CONFIG\_"（在kconfig文件中配置选项都忽略了前缀"CONFIG_"，而在.config和makefile中，配置选项都会添加前缀"CONFIG\_"）
* tristate表示变量类型，即"CONFIG_CFG80211"的类型。config一共有**有5种类型**：bool、tristate、string、hex和int。其中bool表示config有y和n两种取值，而tristate有y，m和n三种取值，取值就决定了它们是否编译进内核，或者作为内核模块。
* "cfg80211 - wireless configuration API"是config说明，在选择config时显示
* depends on表明CONFIG_CFG80211依赖于其他config，只有当(RFKILL || !RFKILL) = y或m时，CONFIG_CFG80211才能选择。
* select也表明一种依赖性，不过是其他config依赖于CONFIG_CFG80211。当CONFIG_CFG80211=y时，CONFIG_FW_LOADER=y。前者=m时，后者也=m。其余select语句亦然。`select CRYPTO_SHA256 if CFG80211_USE_KERNEL_REGDB_KEYS`存在着if语句的条件限制。
* help属性，用于介绍config。

### menu条目

```kconfig
menu "Network testing"

config NET_PKTGEN
...

config NET_DROP_MONITOR
...

endmenu
```

**解析：**
"menu"..."endmenu"用于创建一个子目录。在本例中，CONFIG_NET_PKTGEN与CONFIG_NET_DROP_MONITOR被放在"Network testing"的子目录中。所有子目录都继承这个menu的依赖性(即menu被if控制，那么子目录也被if控制)。

### source

```kconfig
## 当前目录为net/Kconfig
source "net/wireless/Kconfig"
```
**解析：**
读取并解析Kconfig文件。该例中，net/Kconfig文件中读取解析net/wireless/Kconfig文件。source通常用于构建整个内核Kconfig系统，在后续的Kconfig启动流程中会体现这一点。

## Kconfig与Makefile

我们通过Kconfig系统配置特定的CONFIG，即(一般)将CONFIG_XXX赋值为y，m或者n。在Makefile中，通过CONFIG_XXX来控制编译行为。

```Makefile
obj-$(CONFIG_CFG80211) += cfg80211.o
...
cfg80211-y += core.o sysfs.o radiotap.o util.o reg.o scan.o nl80211.o
cfg80211-y += mlme.o ibss.o sme.o chan.o ethtool.o mesh.o ap.o trace.o ocb.o
cfg80211-y += pmsr.o
cfg80211-$(CONFIG_OF) += of.o
cfg80211-$(CONFIG_CFG80211_DEBUGFS) += debugfs.o
cfg80211-$(CONFIG_CFG80211_WEXT) += wext-compat.o wext-sme.o
```

当CONFIG_CFG80211=y,m,n时：
```
obj-$(CONFIG_CFG80211) += cfg80211.o
-->
obj-y += cfg80211.o ## 编译进内核
obj-m += cfg80211.o ## 编译为内核模块
obj-n += cfg80211.o ## 不编译进内核
```

cfg80211-$(CONFIG_XXX)同理，根据CONFIG_XXX的值，选择是否将特定的.o编译进cfg80211.o对象中。
## Kconfig启动流程

一般编译内核时，会使用`make menuconfig`命令。其重要操作如下：

* 构建Kconfig系统：读取最外层的Kconfig文件，构建整个内核的Kconfig系统。

```kconfig
mainmenu "Linux/$(ARCH) $(KERNELVERSION) Kernel Configuration"
source "scripts/Kconfig.include"
source "init/Kconfig"
source "kernel/Kconfig.freezer"
...
source "Documentation/Kconfig"
```

最外层的Kconfig包含多个source语句，负责将各个子系统的顶层Kconfig包含进来，而各个子系统的顶层Kconfig也使用source组织该子系统下所有Kconfig文件。

* 将.config作为输入，菜单可视化当前的配置信息。若顶层目录下没有.config文件，默认读取arch/(当前架构)/configs下的defconfig作为初始config。
* 输出.config文件。`make -j32`编译内核使用.config作为配置文件进行编译。

