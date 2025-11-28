---
layout: default
title: Vscode配置LLDB调试环境指南
parent: 异构推理
grand_parent: AI编译
author: Yan Li
---


# 一、环境准备
## 1. 基本工具下载
1. cmake
2. ninja
3. [adb](https://developer.android.google.cn/tools/adb)

## 2. 主机端下载lldb
使用apt安装lldb或从源码中安装lldb（apt安装的lldb版本较旧，建议从源码安装）

源码安装参考过程：
```shell
git clone https://github.com/llvm/llvm-project.git
cd llvm-project
mkdir build
cd build
cmake -G Ninja ../llvm \
    -DCMAKE_BUILD_TYPE=Release \
    -DLLVM_ENABLE_PROJECTS="lldb;clang" \
    -DLLVM_TARGETS_TO_BUILD="X86" \
    -DLLVM_ENABLE_ASSERTIONS=OFF \
    -DCMAKE_INSTALL_PREFIX=/path/to/your/install/dir
```

## 3. 设备端lldb-server
在Android NDK中找到lldb-server，并adb push到设备端，记下地址

参考地址：

`${ANDROID_SDK_ROOT}/ndk/{version}/toolchains/llvm/prebuilt/linux-x86_64/lib64/clang/{version}/lib/linux/aarch64/lldb-server`

添加权限

`adb shell chmod 755 /path/lldb-server`


# 二、Vscode 环境配置
Vscode中安装 `CodeLLDB`插件
## 1. 配置Launch.json
在项目根目录下创建`.vscode/`目录，在`.vscode/`目录下新建文件`launch.json`
```cpp
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Android CodeLLDB Attach",
            "type": "lldb",
            "request": "attach",
            "pid": "${input:pid}",
            "initCommands": [
                "platform select remote-android",   //选择 Android 远端平台
                //"settings set target.source-map /llama_cpp /home/comsoft/dev/llama-cpp-qnn-builder/llama.cpp", // 若需要修改二进制文件的代码源路径
                "platform connect unix-abstract-connect:///data/local/tmp/debug.sock"  //连接设备上的 `lldb-server`
            ],
            "preRunCommands": [
                "breakpoint set --name common_init"     //在函数处设置断点
            ]
        }
    ],
    "inputs": [
        {
            "id": "pid",
            "type": "promptString",
            "description": "Enter process ID to attach to"
        }
    ] 
}
```
- 调试原理为使用lldb在主机端连接设备端中启动的lldb-server，通过`platform process attach -p <PID>`连接到设备启动进程，因此需要获取设备中启动进程的pid

以下给出一种获取pid的方法：在代码main函数开头加上
```c
LOG_INF("Process ID (PID): %d\n", (int)getpid());
sleep(10); //等待10s
```


## 2. 配置tasks.json
在llama.cpp下新建`.vscode/` ，`.vscode/`目录下新建`tasks.json`文件
```cpp
{
    "version": "2.0.0",
    "tasks": [
         {
            "label": "Make",   //编译源代码并adb push到手机目录下
            "type": "shell",
            "command": "sh",
            "args": [
                "-c",
                "cd ${workspaceFolder}/build && ninja && adb push ./bin/llama-cli /data/local/llama_tests/"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
        },
        {
            "label": "start-lldb-server",  //启动手机上的lldb-server
            "type": "shell",
            "command": "adb", 
            "args": [
                "shell",
                "setsid",
                "/data/local/tmp/lldb-server",
                "p",
                "--server",
                "--listen",
                "unix-abstract:///data/local/tmp/debug.sock"
            ],
        },
        {
            "label": "stop-lldb-server",  //停止手机上的lldb-server
            "type": "shell",
            "command": "adb",
            "args": [
                "shell",
                "pkill",
                "-f",
                "lldb-server"
            ]
        },
        {
            "label": "Run-app",  //启动adb shell , cd 到目标目录，设置环境变量并运行llama-cli
            "type": "shell",
            "command": "adb",
            "args": [
                "shell",
                "cd /data/local/llama_tests && export LD_LIBRARY_PATH=/vendor/lib64:$LD_LIBRARY_PATH && ./llama-cli -m /data/local/llama_tests/llama-3.2-1b-instruct-q4_0.gguf -b 128 -c 2048 -ngl 0 -n 1 -p \"Hello, who are you?\" --no-warmup --temp 0 -no-cnv"
            ]
        }
    ]
}
```

- `Make`为编译任务， 将args内容修改为项目编译命令，并修改adb push的源地址和目标地址
- `start-lldb-server`为启动设备上的lldb-server任务，将地址修改为设备手机中的lldb-server地址
- `Run-app`为运行任务，使用adb shell启动项目，需要将args中的内容修改为项目启动命令


# 三、调试流程
## 1. Vscode调试流程
vscode启动任务方法：Ctrl + Shift + P ，选择Tasks : Run Task，此时出现的列表即对应tasks.json中定义的任务

1. 在`main()`函数开始添加打印pid并等待10s的函数，或其他方法等待lldb连接
2. 选择`Make`任务编译源代码并push到手机中
3. 选择`start-lldb-server`任务启动手机上的lldb-server
4. 选择`Run-app`任务，运行llama.cpp（注意不能关闭lldb-server，新开一个终端并继续）
5. 从终端中读出llama.cpp进程的Pid
6. F5启动调试，输入llama.cpp进程的pid
7. 此时进程即会命中已经设置的断点，即可开始调试

## 2. 命令行调试流程
### 主机端
```
# 设备上 以 unix-abstract socket 方式监听
adb shell '/data/local/tmp/lldb-server p --server --listen unix-abstract:///data/local/tmp/debug.sock'

# 新开终端
lldb
platform select remote-android
platform connect unix-abstract-connect:///data/local/tmp/debug.sock

# 设备上启动进程 attach到进程
platform process attach -p  <PID>

# 设置断点
breakpoint set --name common_init

```

### 设备端
```
# 设置环境变量
export LD_LIBRARY_PATH=/vendor/lib64:$LD_LIBRARY_PATH

# 启动进程
cd data/local/llama_tests/
./llama-cli -m /data/local/llama_tests/llama-3.2-1b-instruct-q4_0.gguf -b 128 -c 2048 -ngl 0 -n 1 -p "Hello, who are you?" --no-warmup --temp 0 -no-cnv
```
