---
layout: default
title: AOSP
nav_order: 5
has_children: true
---

## AOSP 简介

Android Open Source Project（AOSP）是安卓操作系统的开源上游代码库与项目生态，提供从框架到系统服务、工具链与测试套件的完整基础，实现“可编译、可启动、可定制”的 Android。

### 包含什么

* **运行时与核心**：ART、libcore、system/core（init、adb、logcat 等）
* **框架层**：frameworks/base（Activity/Window/Package 等系统服务）
* **应用与组件**：packages/apps、packages/modules（可独立更新的 Mainline 模块）
* **硬件抽象**：hardware/interfaces（AIDL/HIDL HAL）、vendor 接口
* **构建系统**：Soong/Blueprint + Ninja，build/make
* **测试与兼容**：CTS/VTS、Tradefed、perfetto、simpleperf 等

### 基本流程

```bash
# 下载源代码
repo init -u <manifest-url> -b android-<version>_rXX
repo sync -j
# 准备环境
source build/envsetup.sh
# 选择具体设备/目标
lunch           
# 编译生成 out/target/product/<target>/... 镜像               
m -j                         
```

### 代码组织

* **`art/`**
  Android Runtime（ART）与相关工具链：解释器（mterp/nterp）、JIT/AOT、`dex2oat`，更改这里会影响字节码执行、内存管理与性能/兼容性。

* **`frameworks/`**
  框架层与系统服务的“大本营”。
  常见子目录：

  * `frameworks/base/`：Java/Kotlin 系统 API、Activity/Window/Package/Content 等服务实现，系统 UI 基础组件；
  * `frameworks/native/`：native Binder、`libbinder`、`libui`、`libutils` 等基建；
  * `frameworks/av/`：多媒体（MediaServer、AudioFlinger、Camera service）；
  * `frameworks/ml/`、`frameworks/opt/`：ML 与可选特性。
    扩展系统行为、添加/调整公开 API 多在此进行。

* **`system/`**
  系统“地基”和核心守护进程：

  * `system/core/`：`init`、`adbd`、`logd`、`toybox` 子集、`libbase` 等；
  * `system/sepolicy/`：AOSP 侧 SEPolicy；
  * 其他如 `vold/`（存储）、`netd/`（网络）。
    设备启动、init 脚本、系统日志等都与此强相关。

* **`hardware/`**
  HAL 接口与参考实现：

  * `hardware/interfaces/`：AIDL/HIDL HAL 规范（IDL）；
  * `hardware/libhardware/`：传统 HAL 头文件与抽象；
  * `hardware/google/`、`hardware/qcom/`：参考/平台相关实现（开源部分）。
    驱动在内核，用户态与内核之间的稳定契约通常由这里的 HAL 定义。

* **`packages/`**
  系统应用与模块：

  * `packages/apps/`：如 Settings、Launcher、DeskClock 等系统 App；
  * `packages/modules/`：Mainline 可独立更新模块（如 `Permission`, `NetworkStack`, `Media`），含各自的服务、API、测试与构建脚本。
    想定制或替换系统应用、调整 Mainline 组件，主要看这里。

* **`external/`**
  第三方上游依赖的镜像/移植，如 `protobuf/`、`boringssl/`、`jsoncpp/`、`skia/`。
  一般不要随意改动（需与上游同步），但可为 AOSP 构建集成做最小改造。

* **`build/`**
  构建系统定义与全局配置：

  * `build/soong/`（Blueprint/Soong）与 `build/make/`（兼容层）；
  * 产品/目标配置、通用规则、打包脚本、镜像产物布局规则等。
    新增模块、写 `Android.bp`/`Android.mk`、调整编译选项都会经过这里。

* **`prebuilts/`**
  预编译工具/SDK/NDK/编译器（如 clang、javac、host 工具）、`qemu-kernel` 等。
  只读为主，用于确保可复现构建环境；不要在这里开发功能。

* **`device/<vendor>/<device>/`**
  设备树（Device Tree）：

  * 板级与产品配置：`BoardConfig.mk`、`device.mk`、`vendor_required/`、分区与映像定义；
  * 设备专属 `init.<device>.rc`、触摸屏/相机/音频等配置、资源 overlay；
  * 公版驱动接口匹配（与 `vendor/` 专有实现配合）。
    **示例**

  ```bash
  device/acme/phoenix/
  ├─ AndroidProducts.mk
  ├─ BoardConfig.mk
  ├─ device.mk
  ├─ init.phoenix.rc
  ├─ overlay/ (资源覆盖)
  └─ sepolicy/ (设备侧策略片段)
  ```

* **`vendor/`**
  厂商闭源或半闭源部分（通常不在纯 AOSP 清单中）：专有 HAL 实现、固件/二进制、厂商侧 SEPolicy、`proprietary/` 资源、产品定制应用等。与 `device/` 协同，满足实际硬件落地与合规（如 VNDK、Treble 分区边界）。
