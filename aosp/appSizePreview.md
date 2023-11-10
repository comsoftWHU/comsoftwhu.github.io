---
layout: default
title: Apk code size 分析
nav_order: 1
parent: aosp
author: qingan
---
{% assign author = site.data.authors[page.author] %}
<div> 作者: {{ author.name }}  
 邮箱：{{ author.email }}
</div>

# 问题及背景
手机应用的Apk文件瘦身优化， 通常优先级排在启动优化、卡顿优化之后。但是，对于稳定期的APP项目，Apk文件瘦身有如下好处：

## 1. 满足强制要求
- 满足应用商城的需求。比如，Google Play对上架的App就有大小限制。
- 满足手机厂商的预装需求。Apk越大，单价成本越高。

## 2. 提升下载转化率
- 如果某App的APK大小相比同类型的App要更小的话，可以预期下载率会上升，使用率也会上升。

## 3. 提升性能、降低成本
- 加快安装时间。Apk变小后，文件拷贝、解压、编译、签名校验等过程都会加快。
- 降低运行时内存占用。
- 降低ROM存储占用。
- 降低下载带宽需求。

# Apk分析
## Apk组成
    Apk文件解压缩后，是一个文件夹，其主要成分包括：classes.dex文件、so文件、资源文件等。其中，

    - dex文件。这是Android系统的可执行文件，包含应用程序的全部操作指令以及运行时数据。通常命名classes.dex文件。
    应用程序员开发的Kotlin或Java代码，先被编译成Java字节码文件格式.class，然后再利用dx工具将所有.class文件合并到一个classes.dex文件。在合并过程中，不仅通过数据共享降低了信息冗余，而且将适合解释执行的Java字节码转化成更适合编译优化的dex字节码。相比传统的jar文件，dex文件的大小能较少约50%。
![Alt text](image-1.png){:width="100" height="100"}

    - so文件。应用程序所依赖的第三方库（native代码）。通常在lib目录下。
    - res。应用程序用到的资源文件。一般包括图像，布局文件，字符串，音频，XML文件，这些资源是可以被Android系统管理的，并且根据配置（比如语言，屏幕）自行选择。
    - assets。应用程序的资产文件。主要存储二进制文件或原始文件，如HTML，JSON，音频文件等。
    - META-INF。元数据信息，包含签名等信息。
    - Resources.arsc。  res资源的ID映射文件。
    - AndroidManifest.xml。 安卓的清单文件，包含了应用程序的基本信息，组件的声明，应用程序的权限要求以及其他重要的元数据。

## Apk大小分析
根据对抖音，快手，微信，今日头条，淘宝等热门app的分析，Apk文件中各成分的存储占比如下：
- Dex  44%
- So   34.36%
- Res   11.38%
- Assets  8.14%                                         
- Resources.arsc 0.74% 
- META-INF 0.74%
- AndroidManifest.xml 0.06%
![Alt text](image-4.png)

## 跨App的重复度分析
1. 分析了11个流行App中so文件的存储占比，平均占比38%。
![Alt text](image-6.png)

2. 分析了11个App间重复的so文件数
![Alt text](image-7.png)

## 建议
1. 优先优化dex, so和资源文件。
2. 处理单App内部的优化，还可以考虑如何消除跨App的重复冗余
# 现有方法
![Alt text](image-5.png)
# 相关链接
- https://juejin.cn/post/6844904103131234311#heading-21
- https://juejin.cn/post/6872920643797680136
- https://juejin.cn/post/7052614577216815134