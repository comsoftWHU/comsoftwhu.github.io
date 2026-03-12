---
layout: default
title: 一些可能用到的命令
nav_order: 4
parent: AOSP
author: Anonymous Committer
---
## 设备连接与系统控制（ADB）

### 基础连接与权限

* `adb wait-for-device`：等待设备上线
* `adb devices -l`：列出设备与详细信息
* `adb root`：切 root adbd（userdebug/eng 且设备支持时）
* `adb remount`：重新挂载分区为可写（需要 root/策略允许）
* `adb shell ...`：在设备端执行命令
* `adb push <local> <remote>` / `adb pull <remote> <local>`：推送/拉取文件

> 多设备场景下显式加 `-s <serial>`，避免命令跑到错误设备上。([developer.android.com](https://developer.android.com/studio/debug/bug-report))

### 常见 `adb shell` 子命令

* **日志与报告**

  * `logcat`：清空/抓取日志（见后文）([developer.android.com](https://developer.android.com/tools/logcat))
  * `adb bugreport ...`：生成 bugreport（见后文）([developer.android.com](https://developer.android.com/studio/debug/bug-report))
* **系统属性**

  * `getprop <key>` / `setprop <key> <value>`：读取/修改系统属性（见后文）([source.android.com](https://source.android.com/docs/core/architecture/configuration?utm_source=chatgpt.com))
* **进程与启动**

  * `am force-stop <pkg>`：强杀应用
  * `am set-debug-app -w --persistent <pkg>`：以“等待调试器”方式启动并保留标记（见后文）([android.googlesource.com](https://android.googlesource.com/platform/frameworks/base/%2B/742a67127366c376fdf188ff99ba30b27d3bf90c/cmds/am/src/com/android/commands/am/Am.java?utm_source=chatgpt.com))
  * `monkey -p <pkg> -c android.intent.category.LAUNCHER 1`：触发 launcher 启动（见后文）([developer.android.com](https://developer.android.com/studio/test/other-testing-tools/monkey))
  * `pidof -s <pkg>`：获取 PID
* **包与编译**

  * `pm path <pkg>`：查询 APK 路径
  * `cmd package compile ...`：触发 dexopt/AOT 编译（见后文）([android.googlesource.com](https://android.googlesource.com/platform//art/%2B/c7772140ae0e287efed1731a8bb01d4cf78e8daf/libartservice/service/java/com/android/server/art/ArtShellCommand.java))
* **系统服务状态**

  * `dumpsys <service>`：按系统服务维度 dump 状态（Activity/Package/CPU/Mem 等）([developer.android.com](https://developer.android.com/tools/dumpsys))

---

## LLDB 脚本命令

* `add-dsym`：加载/补充符号（自定义命令）
* `command script import <py>`：导入 python 扩展
* `target stop-hook add ...`：挂 stop hook（过滤信号、自动打印寄存器等）
* `command alias ...`：命令别名
* `c`：continue

---

## AOSP 开发测试流程：目的 -> 可用命令

### 环境与设备准备

目的：确认设备可调试、后续操作具备权限与稳定性。

* 连通设备：`adb wait-for-device`
* 检查设备：`adb devices -l`
* 提升权限（可选）：`adb root`（再 `adb remount`）
* 基础状态验证：

  * `adb shell getprop ro.build.fingerprint`
  * `adb shell getprop ro.zygote`
  * `adb shell getprop ro.product.cpu.abi`

---

### 日志与问题初筛（logcat / bugreport / dumpsys）

目的：复现问题并尽快抓到“第一现场”的关键证据。

#### logcat：建议直接用可搜索的格式

* 清空历史日志：

  ```sh
  adb shell logcat -c
  ```
* 推荐抓取（带时间、线程信息，覆盖常用 buffer）：

  ```sh
  adb logcat -v threadtime -b main,system,crash
  ```
* 只看错误级别（快速扫一遍）：

  ```sh
  adb logcat "*:E"
  ```

logcat 的 buffer（`-b main/system/crash/...`）与输出格式在官方文档里有明确说明；抓日志时不带 `crash` buffer 很容易漏掉关键堆栈。([developer.android.com](https://developer.android.com/tools/logcat))

#### bugreport：一次性打包证据

* 生成 bugreport（保存到本地目录或指定路径）：

  ```sh
  adb bugreport ./bugreports/
  ```

* 设备端默认保存位置与拉取方式（示例）：

  ```sh
  adb shell ls /bugreports/
  adb pull /bugreports/<bugreport-xxx>.zip
  ```

bugreport 的核心价值是：它把 `dumpsys`、`dumpstate`、`logcat` 等诊断信息整合在一起。([developer.android.com](https://developer.android.com/studio/debug/bug-report))

#### dumpsys：按服务维度看状态

* 基本用法：

  ```sh
  adb shell dumpsys <service>
  ```
* 列出全部服务（找 service 名称用）：

  ```sh
  adb shell dumpsys -l
  ```
* 常用示例（按你关心的维度挑）：

  ```sh
  adb shell dumpsys activity
  adb shell dumpsys package <your.package>
  adb shell dumpsys meminfo <pid|process>
  adb shell dumpsys cpuinfo
  adb shell dumpsys batterystats
  adb shell dumpsys gfxinfo <your.package>
  ```

`dumpsys` 的官方文档给出了完整语法（包括 `-l`、超时 `-t` 等），并列举了输入、内存、电量、网络等常见诊断任务。([developer.android.com](https://developer.android.com/tools/dumpsys))

---

### 进程生命周期与启动控制（稳定复现 + 便于 attach）

目的：固定启动时机与 PID，减少“我刚要 attach 它就没了”的痛苦。

* 强制停止目标进程：

  ```sh
  adb shell am force-stop <pkg>
  ```
* 设置“等待调试器”并保留标记：

  ```sh
  adb shell am set-debug-app -w --persistent <pkg>
  ```
* 触发启动（launcher）：

  ```sh
  adb shell monkey -p <pkg> -c android.intent.category.LAUNCHER 1
  ```
* 获取 PID：

  ```sh
  adb shell pidof -s <pkg>
  ```

`am set-debug-app` 的 `-w` 与 `--persistent` 行为在 AOSP `Am.java` 的帮助文本里写得很清楚。([android.googlesource.com](https://android.googlesource.com/platform/frameworks/base/%2B/742a67127366c376fdf188ff99ba30b27d3bf90c/cmds/am/src/com/android/commands/am/Am.java?utm_source=chatgpt.com))
Monkey 的基本语法与常用选项也有官方文档可查。([developer.android.com](https://developer.android.com/studio/test/other-testing-tools/monkey))

---

### Runtime 行为与编译策略干预（ART / dexopt / profile）

目的：控制 JIT/AOT/dexopt 行为，让问题更容易复现或更容易解释。

#### 临时注入运行时参数（谨慎）

* 临时追加 ART 参数（示例）：

  ```sh
  adb shell setprop dalvik.vm.extra-opts "<opts>"
  ```

> 注意：系统属性是否“即时生效”取决于读取时机；很多 ART/zygote 相关配置在进程启动阶段读取，通常需要重启相关进程甚至重启设备才能完全验证。([source.android.com](https://source.android.com/docs/core/architecture/configuration?utm_source=chatgpt.com))

#### 触发 dexopt/AOT：`cmd package compile`

在 **Android 14+**，设备端应用的 AOT/dexopt 由 **ART Service** 负责。([source.android.com](https://source.android.com/docs/core/runtime/configure/art-service))
对使用者来说，常用入口仍然是 `cmd package compile ...`。

* 指定编译过滤器（compiler filter）并强制：

  ```sh
  adb shell cmd package compile -m speed-profile -f <pkg>
  adb shell cmd package compile -m speed -f <pkg>
  adb shell cmd package compile -m verify -f <pkg>
  ```
* 按“编译原因（reason）”触发（让系统按默认策略映射 filter/优先级）：

  ```sh
  adb shell cmd package compile -r bg-dexopt <pkg>
  adb shell cmd package compile -r install <pkg>
  ```
* 重置 dexopt 状态（适合做对照实验）：

  ```sh
  adb shell cmd package compile --reset <pkg>
  ```
* 控制编译范围（主 dex / 次 dex / 依赖 / 全部）：

  ```sh
  adb shell cmd package compile --primary-dex --include-dependencies <pkg>
  adb shell cmd package compile --secondary-dex <pkg>
  adb shell cmd package compile --full <pkg>
  ```

这些参数集合与含义可以直接以 AOSP 的 ART Service shell help 为准。([android.googlesource.com](https://android.googlesource.com/platform//art/%2B/c7772140ae0e287efed1731a8bb01d4cf78e8daf/libartservice/service/java/com/android/server/art/ArtShellCommand.java))

#### 查询/诊断 dexopt 状态

ART Service 提供了 dump/cleanup 等诊断能力，例如：

```sh
adb shell cmd package art dump <pkg>
adb shell cmd package art cleanup
```

同样可参考 ART Service 的 shell help。([android.googlesource.com](https://android.googlesource.com/platform//art/%2B/c7772140ae0e287efed1731a8bb01d4cf78e8daf/libartservice/service/java/com/android/server/art/ArtShellCommand.java))

---

### Native/ART 调试（LLDB）

目的：定位 native crash、ART runtime 行为、方法执行路径。

* 启动 LLDB 并加载脚本：

  ```sh
  lldb -s <script>
  ```
* 直接 attach（示例）：

  ```sh
  lldb -s <script> -o "process attach -p <pid>"
  ```
* 配合“等待调试器”的启动点固定住 attach 时机。([android.googlesource.com](https://android.googlesource.com/platform/frameworks/base/%2B/742a67127366c376fdf188ff99ba30b27d3bf90c/cmds/am/src/com/android/commands/am/Am.java?utm_source=chatgpt.com))

---

### OAT/ODEX 产物分析（oatdump）

目的：分析编译产物与指令级输出，或验证“到底有没有编译到我想要的程度”。

* 查询安装包路径（定位 base.apk 等）：

  ```sh
  adb shell pm path <pkg>
  ```
* 枚举 oat/odex（按你的设备/版本路径调整）：

  ```sh
  adb shell find ... -name "*.odex" -o -name "*.oat"
  ```
* 反汇编（示例）：

  ```sh
  adb shell oatdump --oat-file=<path> ...
  ```
* 压缩并拉取结果：

  ```sh
  gzip <oatdump.txt>
  adb pull <remote.gz> <local.gz>
  ```

---


## 常见系统属性与 Runtime 配置

说明：系统属性（system properties）是跨进程共享的 key/value 配置入口；属性可能分布在`/system/build.prop`,`/system/product/vendor` 等分区的 `build.prop` 中。([source.android.com](https://source.android.com/docs/core/architecture/configuration?utm_source=chatgpt.com))
如果要新增/规范化属性，AOSP 也给出了推荐方法与注意事项。([source.android.com](https://source.android.com/docs/core/architecture/configuration/add-system-properties?utm_source=chatgpt.com))

### ART/Dalvik 堆参数（内存行为）

* `dalvik.vm.heapstartsize`：初始堆
* `dalvik.vm.heapgrowthlimit`：普通增长上限
* `dalvik.vm.heapsize`：最大堆上限
* `dalvik.vm.heaptargetutilization`：目标堆利用率（影响 GC 频率）
* `dalvik.vm.heapminfree` / `dalvik.vm.heapmaxfree`：空闲阈值

用途：调优内存占用与 GC 抖动，常用于性能/卡顿分析。

### JIT / Profile 相关

* `dalvik.vm.usejit`：是否启用 JIT
* `dalvik.vm.usejitprofiles`：是否启用 profile 引导
* `dalvik.vm.jitthreshold` / `dalvik.vm.jitinitialsize`：机型/版本相关（不一定都有）
* `dalvik.vm.extra-opts`：运行时额外参数注入入口

JIT（带 profiling）与 AOT 之间是互补关系；如果你在做冷热路径、启动性能权衡，理解 JIT/ART 的定位会很有帮助。([source.android.com](https://source.android.com/docs/core/runtime/jit-compiler?utm_source=chatgpt.com))

### dex2oat / 编译过滤策略（系统侧）

* `dalvik.vm.dex2oat-filter`
* `dalvik.vm.image-dex2oat-filter`
* `dalvik.vm.dex2oat-threads`
* `dalvik.vm.boot-dex2oat-threads`

用途：控制 AOT 编译强度、过滤级别与线程数，影响首次编译耗时与运行性能。([source.android.com](https://source.android.com/docs/core/runtime/configure))

### `pm.dexopt.*` 策略（Android 14+：由 ART Service 负责）

在 Android 14+，许多默认 dexopt 策略通过 `pm.dexopt.<reason>` 表达，例如 `pm.dexopt.bg-dexopt=speed-profile` 等；ART Service 配置文档给出了标准默认值与含义。([source.android.com](https://source.android.com/docs/core/runtime/configure/art-service))
