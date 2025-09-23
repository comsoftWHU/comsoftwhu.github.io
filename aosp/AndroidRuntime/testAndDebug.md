---
layout: default
title: 使用lldb调试ART
nav_order: 0
parent: AndroidRuntime
grand_parent: AOSP
author: Anonymous Committer
---

## 1. 环境准备

确保具备：

* host 端 LLDB：[https://lldb.llvm.org/index.html](https://lldb.llvm.org/index.html)
* 已解锁 Bootloader 的 Android 设备（需可 `adb root`）
* ADB：[https://developer.android.google.cn/tools/adb](https://developer.android.google.cn/tools/adb)
* 已编译好的 ART（`com.android.art.capex`）
* 设备端 `lldb-server` 二进制（Android NDK，示例路径：`${ANDROID_SDK_ROOT}/ndk/{version}/toolchains/llvm/prebuilt/linux-x86_64/lib64/clang/{version}/lib/linux/aarch64/lldb-server`）
* （可选）host 上的 unstripped 符号文件：`out/target/product/*/symbols/apex/com.android.art`

---

## 2. 设备准备

进入 `adb root` 模式并解除安全限制：

```bash
adb root
adb shell disable-verity
adb shell setenforce 0
```

重新挂载 `/system`，为后续替换 ART 做准备：

```bash
adb shell remount
adb shell mkdir -p /system/art_wrap/ # art_wrap 可以是其他路径
adb shell stop
```

---

## 3. 推送自定义 ART

编译 `com.android.art` 后，在 `out/target/product/*/system/apex` 中可得到 `com.android.art.capex`。解包路径说明：

1. 解压 `com.android.art.capex` 得到 `original_apex`
2. 再解压 `original_apex` 得到 `apex_payload.img`
3. 挂载/解包 `apex_payload.img`，即可拿到 `bin/`, `lib64/`, `javalib/`, `etc/` 等最终文件夹（需要 push 到设备）

目录示例：

```text
bin/
etc/
javalib/
lib64/
```

推送与绑定挂载：

```bash
# 将 /system/art_wrap 绑定到正式 APEX 挂载点
adb shell mount /system/art_wrap/ /apex/com.android.art/

# 推送自定义 art_wrap（注意保持子目录结构）
adb push art_wrap /system/

# 赋权（至少保证 bin 可执行）
adb shell chmod -R 0755 /system/art_wrap/bin
```

> 说明：使用 **bind mount** 替换 APEX 内容，不需要破坏性地重打包/签名 APEX，且可快速回滚（umount或重启即可）。

---

## 4. 部署并启动 lldb-server（设备端）

```bash
adb push {Sdk_Path}/ndk/{version}/toolchains/llvm/prebuilt/linux-x86_64/lib64/clang/{version}/lib/linux/aarch64/lldb-server /data/local/tmp
adb shell chmod 755 /data/local/tmp/lldb-server

# 以 unix-abstract socket 方式监听
adb shell '/data/local/tmp/lldb-server p --server --listen unix-abstract:///data/local/tmp/debug.sock'
```

---

## 5. LLDB 连接与调试（主机端）

```bash
lldb
```

选择 Android 远端平台：

```bash
platform list
platform select remote-android
platform status
```

连接设备上的 `lldb-server`：

```bash
platform connect unix-abstract-connect:///data/local/tmp/debug.sock
```

列出与附加进程：

```bash
platform process list
platform process attach -p <PID>
```

一般来说, 第一个`app_process64`的进程就是`zygote`进程,可以attach到上面debug ART。

加载符号：

```bash
# 为指定 so/可执行添加符号
target symbols add /absolute/path/to/out/target/product/*/symbols/apex/com.android.art/lib64/libart.so
```

常用断点与检查：

```bash
# 示例：在 ART 关键函数上断点
br set -s libart.so -n art::ThreadList::SuspendAll
br set -s libart.so -n art::gc::collector::MarkCompact::MarkCompact

# 线程与回溯
thread list
bt

# 表达式求值（查看 PrettyMethod）
expression -- reinterpret_cast<art::ArtMethod *>(0x00000000702bd9b8)->PrettyMethod(true)
```

---

## 6. 运行与日志

```bash
adb shell start                        # 恢复系统服务
adb shell logcat "*:F"                 # 仅致命级别
adb shell artd&                        # 后台运行 artd（可选）
```

---

## 7. 如何把 Java 编译成 DEX 字节码

使用 `javac + d8`

`d8` 是 Android Build-Tools 提供的 DEX 编译器（替代老旧的 `dx`）。

**步骤示例：**

1）准备 Java 源码（`HeapAllocationTest.java`）

```java
public class HeapAllocationTest {
    public static void main(String[] args) {
        byte[] buf = new byte[1024 * 1024];
        System.out.println("ok: " + buf.length);
    }
}
```

2）使用 `javac` 编译成 `.class`（需要合适的 JDK，建议 8 或 11）：

```bash
# 输出目录 classes/
javac -source 1.8 -target 1.8 -d classes HeapAllocationTest.java
```

3）用 `d8` 生成 `classes.dex`（Build-Tools 目录中）：

```bash
# 假设 ANDROID_SDK_ROOT 指向 Android SDK 根目录
${ANDROID_SDK_ROOT}/build-tools/<build-tools-version>/d8  --release  --min-api 26  --output out_dex classes
```

生成目录 `out_dex/` 下会包含 `classes.dex`。将其推送至设备：

```bash
adb push out_dex/classes.dex /data/local/tmp/
```

## 8. 测试 Dalvik 虚拟机

假设 `classes.dex` 中包含测试类 `HeapAllocationTest`：

```bash
adb push classes.dex /data/local/tmp/
```

运行 `dalvikvm64`：

```bash
/data/local/tmp/art_wrap/bin/dalvikvm64 -cp /data/local/tmp/classes.dex HeapAllocationTest main
```

比如

```bash
1|cheetah:/ $ dalvikvm64 -cp /data/local/tmp/classes.dex HeapAllocationTest main
========== 测试场景1: 大量分配后回收 ==========
[开始前] 内存状态: 可用=1.9MB, 已分配=2.0MB, 最大=256.0MB
[0] 内存状态: 可用=1.9MB, 已分配=2.0MB, 最大=256.0MB
[100000] 内存状态: 可用=0.0MB, 已分配=102.3MB, 最大=256.0MB
[200000] 内存状态: 可用=0.0MB, 已分配=204.7MB, 最大=256.0MB
Exception in thread "main" java.lang.OutOfMemoryError: Failed to allocate a 1040 byte allocation with 126288 free bytes and 123KB until OOM, target footprint 268435456, growth limit 268435456; giving up on allocation because <1% of heap free after GC.
        at HeapAllocationTest.testMassiveAllocationWithGC(HeapAllocationTest.java:27)
        at HeapAllocationTest.main(HeapAllocationTest.java:10)
```

---
