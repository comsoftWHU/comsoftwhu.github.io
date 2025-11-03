---
layout: default
title: simpleperf
nav_order: 3
parent: AOSP
author: Anonymous Committer
---

<https://android.googlesource.com/platform/system/extras/+/master/simpleperf/doc/README.md>

<https://android.googlesource.com/platform/system/extras/+/refs/heads/main/simpleperf/doc/executable_commands_reference.md>

<https://android.googlesource.com/platform/system/extras/+/main/simpleperf/doc/android_platform_profiling.md>

<https://android.googlesource.com/platform/system/extras/+/refs/heads/main/simpleperf/doc/view_the_profile.md>

# Simpleperf

## 🧩 一、总体概念：什么是 Simpleperf

**Simpleperf 是 Android 平台上的原生 CPU 性能分析工具**，类似于 Linux 上的 `perf`。
它的主要作用是：

* 采样 **CPU 使用情况、函数调用栈**，生成性能报告；
* 能分析 **C/C++ 原生代码** 和 **Java 层代码（>= Android P）**；
* 可以在 **设备端记录数据**，再在 **主机端分析报告**。

👉 它由两个部分组成：

1. **simpleperf 可执行文件**：采样工具，运行在设备端；
2. **Python 脚本集**：报告和分析工具，运行在主机端。

---

## 🧰 二、工具组成与路径

Simpleperf 的文件结构如下：

```
simpleperf/
├── bin/
│   ├── android/<arch>/simpleperf         # 设备端静态可执行文件
│   ├── host/<arch>/simpleperf            # 主机端报告工具
│   ├── host/<arch>/libsimpleperf_report  # 供脚本调用的报告库
├── scripts/
│   ├── app_profiler.py                   # 分析应用的脚本
│   ├── report_html.py                    # 生成网页报告
│   ├── report.py                         # 命令行报告
│   ├── binary_cache_builder.py           # 构建符号缓存
│   ├── inferno, purgatorio               # 可视化脚本
```

这些文件在：

* AOSP 源码中位于 `system/extras/simpleperf/`
* NDK 发布包中位于 `simpleperf/`

---

## 🚀 三、功能与特性

Simpleperf 相比于 Linux 的 `perf`，主要多了以下适配 Android 的特性：

1. **记录更多上下文信息**（设备型号、符号表、时间等），便于“设备采样—主机分析”；
2. **支持 Java JIT/interpreter 调用栈采样**（Android P 及以上）；
3. **支持 .gnu_debugdata**（O 以后系统库都带该调试信息）；
4. **支持从 APK 内的 so 中读取符号表**；
5. **使用与 Android 一致的栈展开器**；
6. **提供静态可执行文件**，可以推送到任意设备直接运行；
7. **跨平台主机报告工具**（Linux/macOS/Windows）。

---

## 🧪 四、两种典型使用场景

### 1️⃣ Android 应用分析

针对 App，使用 Python 脚本 `app_profiler.py` 辅助采样：

```bash
./app_profiler.py -p com.example.app -r "-g --duration 10"
```

采样 10 秒后自动拉取数据，生成报告。

### 2️⃣ Android 平台代码分析

用于分析系统进程或 native 服务：

```bash
adb shell su 0 simpleperf record -a -g --duration 10
adb pull /data/local/tmp/perf.data .
./report_html.py
```

生成网页形式的火焰图报告。

---

## 📈 五、采样方式：DWARF vs Stack Frame

Simpleperf 支持两种方式记录调用栈：

| 方式                    | 优点                    | 缺点                 | 适用场景           |
| --------------------- | --------------------- | ------------------ | -------------- |
| **DWARF-based**       | 能解析 Java/C++ 调用栈，信息完整 | 开销大，<=4000Hz       | ARM、Java、混合代码  |
| **Stack frame-based** | 开销低，能高频采样（>10kHz）     | ARM 上不可靠，Java 无法解析 | ARM64 C++ 原生程序 |

> DWARF 基于调试信息 `.eh_frame` 展开栈；
> Stack frame 依赖函数栈寄存器帧。

---

## ⚙️ 六、常见问题与修复

### 🧩 1. **DWARF 调用栈不完整**

可能原因：

* 栈数据超过 64KB；
* 缺少 `.eh_frame` 或 `.debug_frame` 段。

解决：

* 使用 `--no-cut-samples` 或增加 `--user-buffer-size`；
* 用未剥离符号表的 so 进行分析：

  ```bash
  adb push unstripped_libs/*.so /data/local/tmp/native_libs
  adb shell simpleperf record ... --symfs /data/local/tmp/native_libs
  ```

---

### 🧩 2. **报告中出现大量 unknown symbol**

原因：设备上库文件被 strip 掉。

解决：
在主机上构建符号缓存：

```bash
./binary_cache_builder.py -lib path/to/unstripped_libs
./report.py --symfs binary_cache
# 或使用 report_html.py 自动搜索 binary_cache/
```

---

### 🧩 3. **采样丢失或栈截断**

原因：内核缓冲区或用户缓冲区满。

解决：

* 增大内核 buffer：`-m <size>`；
* 增大用户 buffer：`--user-buffer-size <size>`；
* 或降低采样频率：`-f <Hz>`。

输出示例：

```
Samples lost: 2,129 (kernelspace: 18, userspace: 2,111)
→ 建议增大 buffer 或降低采样频率
```

---

## 🔍 七、查看源代码与构建

如果要从源码构建 Simpleperf：

```bash
. build/envsetup.sh
lunch aosp_arm64-trunk_staging-userdebug
mmma system/extras/simpleperf -j30
```

生成：

* `out/target/product/generic_arm64/system/bin/simpleperf`
* `out/target/product/generic_arm64/system/bin/simpleperf32`

更新脚本依赖的二进制：

```bash
cp out/host/linux-x86/lib64/libsimpleperf_report.so system/extras/simpleperf/scripts/bin/linux/x86_64/
cp out/target/product/generic_arm64/system/bin/simpleperf system/extras/simpleperf/scripts/bin/android/arm64/
```

---

## 🖼️ 八、报告查看方式

### 1️⃣ HTML 报告

```bash
./report_html.py
# 输出 report.html，可用火焰图查看热点函数
```

### 2️⃣ pprof 报告

```bash
./pprof_proto_generator.py
pprof -http=:8080 perf.proto
```

通过网页交互式查看源码级热点。

---

## 🔧 九、版本支持差异总结

| Android 版本 | 可分析语言          | 支持情况                                      |
| ---------- | -------------- | ----------------------------------------- |
| < N (7.0)  | 仅 C++          | 内核太旧，不支持 DWARF                            |
| M–O (6–8)  | C++ + 已编译 Java | 不支持解释器栈                                   |
| ≥ P (9.0)  | Java + C++     | 支持 JIT / interpreter                      |
| ≥ Q (10)   | 允许分析已发布 App    | 可用 `<profileable android:shell="true" />` |

---

## 💡 十、核心理解总结

| 关键点                             | 说明                    |
| ------------------------------- | --------------------- |
| **Simpleperf = Android 版 perf** | 可记录 native & Java 调用栈 |
| **record + report**             | 设备采样 + 主机报告           |
| **DWARF 调用栈**                   | 准确但慢                  |
| **Stack frame 调用栈**             | 快但不稳                  |
| **.gnu_debugdata 支持**           | 系统库符号可自动解码            |
| **buffer 调整重要**                 | 影响采样丢失与准确率            |
| **binary_cache**                | 解决 unknown symbol 问题  |
| **>= Android P**                | 才能分析 Java 调用栈         |

# Android platform profiling

## 🧩 一、总体背景：Android 平台级性能分析

如果你是系统开发者（能编译并刷入 AOSP 镜像、设备 root），你可以用 Simpleperf 直接分析：

* 系统服务（如 system_server、surfaceflinger、mediaserver）
* native 守护进程（如 servicemanager、vold）
* 甚至 **系统启动阶段（boot-time profiling）**

---

## 🧠 二、General Tips — 基础建议

这部分列出平台开发者的使用建议和注意事项：

### ✅ 1. Root 权限

> “After running adb root, simpleperf can profile any process or system wide.”

要分析系统级进程，你必须运行：

```bash
adb root
```

这样 simpleperf 就能访问任意进程的 perf 事件。

---

### ✅ 2. 使用最新版本

建议使用 **AOSP main 分支** 的最新版 simpleperf：

```bash
system/extras/simpleperf/scripts/
```

因为旧版可能缺少 DWARF unwind、off-cpu tracing、或 JIT 支持。

---

### ✅ 3. 推荐使用 Python 封装脚本

官方推荐：

```bash
./app_profiler.py  # 用于录制
./report_html.py   # 用于报告
```

示例：

```bash
# 对 surfaceflinger 采样 10 秒
./app_profiler.py -np surfaceflinger -r "-g --duration 10"

# 生成 HTML 火焰图报告
./report_html.py
```

---

### ✅ 4. 使用符号与源码信息

Android O 及以后版本自带系统库符号表，因此报告时可以直接解析符号；
但如果你想在报告中显示 **源码与反汇编（带行号）**，仍需要使用未剥离的 so。

示例：

```bash
# 从 AOSP 构建目录收集符号
./binary_cache_builder.py -lib $ANDROID_PRODUCT_OUT/symbols

# 或从 symbol 压缩包（如 Pixel OTA 提供的 symbols.zip）收集
unzip comet-symbols-12488474.zip
./binary_cache_builder.py -lib out
```

> 这些操作会生成一个 `binary_cache/` 文件夹，里面是所有带 debug section 的 so。

---

### ✅ 5. 生成带源码+反汇编的 HTML 报告

```bash
./report_html.py --add_source_code \
  --source_dirs $ANDROID_BUILD_TOP \
  --add_disassembly \
  --binary_filter surfaceflinger.so
```

说明：

* `--add_source_code`：显示源代码；
* `--add_disassembly`：显示汇编；
* `--binary_filter`：仅对特定库反汇编（防止生成太慢）。

---

## ⚙️ 三、Start simpleperf from system_server process

> **场景：** 当你只想在某个“特定事件发生时”自动启动性能分析。

例如，你在 system_server 内部检测到卡顿、崩溃、或者 GC 频繁，就想立刻启动 simpleperf。

### ✅ 实现思路

1. **关闭 SELinux：**

   ```bash
   adb shell setenforce 0
   ```

   因为 system_server 用户无法直接运行 simpleperf。

2. **在检测到特定情况时执行以下代码：**

   ```java
   try {
     // 提升 CAP_SYS_PTRACE 权限，允许采样自己
     Os.prctl(OsConstants.PR_CAP_AMBIENT, OsConstants.PR_CAP_AMBIENT_RAISE,
              OsConstants.CAP_SYS_PTRACE, 0, 0);
     // 启动 simpleperf 记录当前进程
     Runtime.getRuntime().exec(
       "/system/bin/simpleperf record -g -p " + Process.myPid()
       + " -o /data/perf.data --duration 30 --log-to-android-buffer --log verbose");
   } catch (Exception e) {
     Slog.e(TAG, "error while running simpleperf");
   }
   ```

   含义：

   * `-g`：记录调用栈；
   * `-p`：采样当前 PID；
   * `--duration 30`：采样 30 秒；
   * `-o /data/perf.data`：输出结果；
   * `--log-to-android-buffer`：把 simpleperf 日志写到 logcat；
   * `--log verbose`：详细输出。

> ⚠️ 注意：`/data/local/tmp` 对 system_user 不可写，所以这里改为 `/data/`。

这可以帮你做“自触发式分析”——即在触发点启动性能采样。

---

## 💻 四、Hardware PMU counter limit

> **主题：** 硬件性能计数器（PMU）的限制。

每个 CPU 核心上都有若干个硬件计数器（PMU counters），用来追踪事件：

* 指令数
* Cache miss
* 分支预测错误 等等

---

### ✅ 限制原因

每个核心可同时启用的事件数量受限。

例如在 Pixel 上：

* 每核一般有 **7 个 PMU 计数器**
* 其中 **4 个被内核占用**
* 剩下 **3 个可用给 simpleperf**

如果你一次监控超过 3 个事件，simpleperf 会 **在事件间复用计数器（multiplex）**，导致精度下降。

---

### ✅ 查看设备支持

```bash
simpleperf stat --print-hw-counter
```

### ✅ 解决方案

如果确实要监控 >3 个事件：

```bash
simpleperf stat --use-devfreq-counters
```

这样可以借用设备频率计数器（devfreq counters）。

---

## 🚀 五、Get boot-time profile（启动阶段分析）

这个功能非常强大：
**在 Android 启动阶段自动启动 simpleperf**，采样 init、zygote、system_server 等所有进程。

---

### ✅ Step 1：准备配置

默认配置文件：
`system/extras/simpleperf/simpleperf.rc`

它定义了：

* 从 early-init 阶段启动；
* 排除 simpleperf 自身；
* 当 `sys.boot_completed`=true 时自动停止。

你可以修改触发条件或命令行参数。

---

### ✅ Step 2：设置 kernel 命令行参数

在 fastboot 下添加：

```bash
fastboot oem cmdline add androidboot.simpleperf.boot_record=1
```

表示“让 init 在 early-init 阶段启动 simpleperf”。

---

### ✅ Step 3：重启设备

重启后：

* init 在 early-init 阶段发现此参数；
* fork simpleperf 后台进程；
* 记录启动过程（从 init → system_server）性能。

---

### ✅ Step 4：导出结果

启动完成后，数据保存在：

```
/tmp/boot_perf.data
```

导出到主机分析：

```bash
adb pull /tmp/boot_perf.data .
./report_html.py --add_source_code
```

> 第一个采样通常出现在开机约 4.5 秒后。
> 这能帮助你定位“启动卡顿”或“早期初始化过慢”的模块。

---

## 🧾 六、整体总结表

| 功能模块     | 作用                              | 常用命令 / API                                                       |
| -------- | ------------------------------- | ---------------------------------------------------------------- |
| 普通系统进程采样 | 分析 surfaceflinger、system_server | `app_profiler.py -np surfaceflinger`                             |
| 报告分析     | 生成 HTML 火焰图                     | `report_html.py --add_source_code`                               |
| 内嵌触发采样   | 在 Java 中启动 simpleperf           | `Runtime.getRuntime().exec("/system/bin/simpleperf record ...")` |
| PMU 限制   | 每核约 3 个计数器                      | `simpleperf stat --print-hw-counter`                             |
| 启动期采样    | 自动记录 boot 启动性能                  | `androidboot.simpleperf.boot_record=1`                           |

---

## 💡 七、总结一句话理解

> **Simpleperf for platform developers** 是一个可嵌入系统、支持早期引导和自触发分析的性能采样框架。
> 它能记录从 **init → system_server → app 进程** 的完整调用栈信息，用于找启动瓶颈、卡顿点或性能异常。

# Executable commands reference

## 一、Simpleperf 工作原理（超简版）

* CPU 里有 **PMU 硬件计数器**。内核把它们抽象成 **perf events**（再加上软件事件、tracepoints），通过 `perf_event_open` 暴露给用户态。
* **三大命令**：

  * `stat`：**聚合计数**（多少 cycles/misses/…）——只给总量，不保存样本。
  * `record`：**采样**（PC、调用栈、上下文）——写到 `perf.data`。
  * `report`：离线解析 `perf.data`（聚合、排序、火焰图/HTML 报告）。
* 采样流转（`record`）：开启事件 → 建立内核/用户映射缓冲区 → 事件发生到阈值时，**内核写样本** → simpleperf 读出并落盘。

---

## 二、命令族概览（你会常用的就这几类）

* `list`：列出设备支持的事件（hardware/software/hw-cache/raw）。
* `stat`：打印事件计数（支持按进程/线程/系统、按核、按线程输出）。
* `record`：录制样本（事件/目标/频率/时长/输出路径/是否记录调用栈…）。
* `report` / `report-sample`：离线报告（聚合/过滤/排序/调用栈）。
* 调试类（了解即可）：`dump`（读 perf.data）、`debug-unwind`（DWARF 离线回溯）、`kmem`（将由脚本替代）。

---

## 三、`list`：看设备能数什么

```bash
simpleperf list
```

* 常见硬件事件：`cpu-cycles`, `instructions`, `cache-misses` …
* ARM/ARM64 还会列 **raw 事件**（如 `raw-l3d-cache-refill`），当某些 PMU 事件没被内核“映射成标准名”时，可直接用 raw。

---

## 四、`stat`：快速“看量”的瑞士军刀

### 1) 选择事件

```bash
# 单事件
simpleperf stat -e cpu-cycles -p 11904 --duration 10
# 复合事件
simpleperf stat -e cache-references,cache-misses -p 11904 --duration 10
```

> ⚠️ **硬件计数器数目有限** → 事件多于计数器会 **multiplexing**（复用），导致统计变少、误差大。
> 用 `simpleperf stat --print-hw-counter` 看每核可用计数器数，**事件数 ≤ 计数器数** 最稳。
> 确保**某几组事件同时数**：用 `--group e1,e2` 成组。

### 2) 选择目标

```bash
# 进程 / 名称正则
simpleperf stat -p 11904,11905 --duration 10
simpleperf stat -p chrome --duration 10
simpleperf stat -p "chrome:(privileged|sandboxed)" --duration 10
# 线程
simpleperf stat -t 11904,11905 --duration 10
# 启子进程并统计
simpleperf stat ls
# 应用（debuggable/profileable from shell）
simpleperf stat --app simpleperf.example.cpp --duration 10
# 全系统
su 0 simpleperf stat -a --duration 10
```

### 3) 时长与打印间隔

```bash
simpleperf stat -p 11904 --duration 10
simpleperf stat -p 11904 --duration 10 --interval 300   # 每 300ms 打印一次
```

### 4) 按线程 / 按核展开

```bash
# 找忙线程
simpleperf stat --per-thread -p 11904 --duration 1
# 全系统每秒打印各线程值（仅运行中的线程会报）
su 0 simpleperf stat --per-thread -a --interval 1000 --interval-only-values
# 按核分布（big.LITTLE 观测常用）
simpleperf stat -e cpu-cycles --per-core -p 1057 --duration 3
su 0 simpleperf stat --per-core -a --duration 1
```

### 5) 不同核监控不同事件（大/小核分流）

```bash
# 缺省（所有核都数这两个事件）
su 0 simpleperf stat -e cpu-cycles,instructions -a --duration 1 --per-core
# 只在核 0-3,8 数 cycles，再在所有核数 instructions
su 0 simpleperf stat -e cpu-cycles --cpu 0-3,8 -e instructions -a --duration 1 --per-core
# 0-3 数 raw-l3d-cache-refill-rd；4-8 数 raw-l3d-cache-refill
su 0 simpleperf stat --cpu 0-3 -e raw-l3d-cache-refill-rd --cpu 4-8 -e raw-l3d-cache-refill -a --duration 1 --per-core
```

### 6) 把计数打进 systrace（联动可视化）

```bash
su 0 simpleperf stat -e instructions:k,cache-misses -a --interval 300 --duration 15
# 主机上同时采 trace（示例路径基于源码树）
HOST$ external/chromium-trace/systrace.py --time=10 -o new.html sched gfx view
```

---

## 五、`record`：录制样本（做火焰图/函数热点）

### 1) 最小示例

```bash
simpleperf record -p 7394 --duration 10
# 默认事件 cpu-cycles，默认频率 ~4000/s（线程在“运行”时间内）
```

### 2) 选事件/目标/时长/输出

```bash
simpleperf record -e instructions -p 11904 --duration 10
simpleperf record -e task-clock -p 11904 --duration 10              # 以 CPU 时间为准（ns）
simpleperf record -p 11904,11905 --duration 10
simpleperf record -p "chrome:(privileged|sandboxed)" --duration 10
simpleperf record -t 11904,11905 --duration 10
simpleperf record --app simpleperf.example.cpp --duration 10
simpleperf record -a --duration 10
simpleperf record -p 11904 -o data/perf2.data --duration 10
```

### 3) 采样频率/周期与 CPU 占用上限

```bash
# 频率：每秒（线程“在跑”的时间里）采多少次
simpleperf record -f 1000 -p 11904,11905 --duration 10
# 周期：每发生多少事件采一次
simpleperf record -c 100000 -t 11904,11905 --duration 10
# 控制内核为采样花的 CPU 上限（≥Android Q 或 root）
simpleperf record -f 10000 -p 11904 --duration 10 --cpu-percent 50
```

### 4) 记录调用栈（火焰图必要）

```bash
# DWARF（默认 -g）：跨 Java/C++、ARM/ARM64 更稳，但开销更大
simpleperf record -p 11904 -g --duration 10
# 栈帧（--call-graph fp）：开销低、频率高，ARM64 C++ 稳，ARM/Java 不稳
simpleperf record -p 11904 --call-graph fp --duration 10
```

### 5) 记录 on/off-CPU（定位“卡在等待/IO/互斥”的时间）

> 只允许 `cpu-clock`/`task-clock` 事件；simpleperf 同时订阅 `sched:sched_switch`，计算 **off-CPU 段** 的时间。

```bash
# 检查内核是否支持
simpleperf list --show-features     # 看到 trace-offcpu 即支持（kernel ≥ 4.2）
# 录制 on+off
simpleperf record -g -p 11904 --duration 10 --trace-offcpu -e cpu-clock
# 系统级
simpleperf record -a -g --duration 3 --trace-offcpu -e cpu-clock
```

---

## 六、`report`：离线报告与过滤聚合

### 1) 基本用法

```bash
# 默认读取 ./perf.data
simpleperf report
# 指定 data
simpleperf report -i data/perf2.data
```

### 2) 补符号/调试信息（报告能看到函数名、行号/源码/反汇编）

```bash
# 引导 simpleperf 去指定目录找二进制（未剥离 so）
simpleperf report --symfs /path/to/debug_dir
# AOSP 本地构建的系统符号
simpleperf report --symfs $ANDROID_PRODUCT_OUT/symbols
```

> 生成 HTML 火焰图与源码/反汇编：
> `report_html.py --add_source_code --source_dirs $ANDROID_BUILD_TOP --add_disassembly --binary_filter yourlib.so`
> （你也可以先用 `binary_cache_builder.py -lib <dirs>` 把未剥离 so 收集到 `binary_cache/`，`report_html.py` 会自动用）

### 3) 过滤样本（按进程/线程/库/名称）

```bash
simpleperf report --comms sudogame
simpleperf report --pids 7394,7395
simpleperf report --tids 7394,7395
simpleperf report --dsos /data/app/.../libsudo-game-jni.so
```

### 4) 聚合与排序（决定 report 行的“维度”）

```bash
# 仅按进程
simpleperf report --sort pid
# 按 线程ID + 线程名
simpleperf report --sort tid,comm
# 按 二进制 + 函数
simpleperf report --sort dso,symbol
# 默认：--sort comm,pid,tid,dso,symbol
simpleperf report
```

### 5) 展示调用栈

```bash
simpleperf report -g
# HTML 里用 report_html.py 更直观
```

### 6) off-CPU 报告模式（和 record 的 --trace-offcpu 配套）

```bash
# 只看 on-CPU
./report_html.py --trace-offcpu on-cpu
# 只看 off-CPU
./report_html.py --trace-offcpu off-cpu
# on/off 分成两类事件
./report_html.py --trace-offcpu on-off-cpu
# on/off 合并同一事件名下（默认）
./report_html.py --trace-offcpu mixed-on-off-cpu
```

---

## 七、典型场景配方（可直接抄）

### A) “我就想看这进程 10 秒里在干嘛”（函数热点 + 火焰图）

```bash
# 录制（DWARF 调用栈，稳）
simpleperf record -p $PID -g --duration 10
# 生成 HTML + 源码/反汇编（可选）
binary_cache_builder.py -i perf.data -lib $ANDROID_PRODUCT_OUT/symbols
report_html.py --add_source_code --source_dirs $ANDROID_BUILD_TOP --add_disassembly
```

### B) “只看每核/每线程的计数分布”（排查大核是否被打满）

```bash
su 0 simpleperf stat -e cpu-cycles --per-core -a --duration 3
su 0 simpleperf stat --per-thread -a --interval 1000 --interval-only-values
```

### C) “我怀疑卡顿在等待 IO/锁”（off-CPU 定位）

```bash
simpleperf record -g -p $PID --duration 10 --trace-offcpu -e cpu-clock
report_html.py --trace-offcpu on-off-cpu
```

### D) “事件太多数不过来”（避免 multiplex）

```bash
simpleperf stat --print-hw-counter
# 控制监控事件 ≤ 可用计数器；需要分组同时性就用 --group
```

---

## 八、关键坑位与应对

* **硬件计数器不够 → multiplex**：先查 `--print-hw-counter`，超限就减事件或用 `--group` 管控同时性。
* **ARM/Java 栈展开不稳**：优先 `-g`（DWARF）；要高频/低开销再试 `--call-graph fp`（ARM64 C++ 效果好）。
* **报告里一堆 unknown**：离线补符号 + 调试段

  * 用 `binary_cache_builder.py -lib <unstripped_dirs>` 收集；
  * `report_html.py` 自动搜 `binary_cache/`；
  * 或 `report --symfs <debug_dir>`.
* **想看源码/行号/反汇编**：必须有 `.debug_*` 或 `.gnu_debugdata`；用 `--add_source_code --add_disassembly`。
* **频率太高占用过大**：降低 `-f` 或设 `--cpu-percent`；或换 `--call-graph fp`。
* **目标选择**：进程名正则很好用（`-p "chrome:(privileged|sandboxed)"`）；系统范围用 `-a` 需 root。
