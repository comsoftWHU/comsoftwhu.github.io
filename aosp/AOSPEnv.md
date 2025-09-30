---
layout: default
title: AOSP环境搭建
nav_order: 3
parent: AOSP
author: Anonymous Committer
---

参见
<https://source.android.com/docs/setup/build/building>

<https://source.android.com/docs/setup/start/requirements>

<https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/>

## 依赖

```bash
sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev libc6-dev-i386 x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig
```

## 安装 `repo` 工具

```bash
mkdir ~/bin
PATH=~/bin:$PATH
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

或者使用 git-repo 镜像

```bash
curl -L https://mirrors.tuna.tsinghua.edu.cn/git/git-repo -o repo
chmod +x repo
```

repo 的运行过程中会尝试访问官方的 git 源更新自己，如果想使用清华镜像源进行更新，可以将如下内容复制到你的`~/.bashrc`里

```bash
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'
```

## 初始化仓库

<https://source.android.com/docs/setup/reference/build-numbers>

选择要同步的分支，例如 android-15.0.0_r20 ：

```bash
mkdir ~/aosp
cd ~/aosp
repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest -b android-15.0.0_r20
```

## 同步源码

```bash
repo sync -j$(nproc)
```

> 清华源建议sync的时候并发数不宜太高，否则会出现 503 错误，即 -j 后面的数字不能太大，建议选择 4。
> ⚠️ 这是最耗时的一步，视网络情况可能需要数小时甚至更久。

## 配置编译环境

进入源码目录后，执行：

```bash
source build/envsetup.sh
```

选择要编译的目标，例如 Pixel 7 pro (代号 `cheetah`)：

```bash
lunch aosp_cheetah-bp1a-userdebug 
```

后续的刷写见 <https://source.android.com/docs/setup/test/running>

例如cuttlefish模拟器：

```bash
lunch aosp_cf_arm64_only_phone-trunk_staging-userdebug 
```

## 开始编译

在源代码目录使用 `m` 命令启动编译整个镜像：

```bash
m -j$(nproc)
```

编译完成后，生成的系统镜像位于 `out/target/product/<device>/` 目录下。

使用`m`后跟目标名字可以选择编译某个模块，比如ART：

```bash
m com.android.art
```

编译完成后，会在`out/target/product/<device>/system/apex/`生成可以用于更新ART的CAPEX格式的包。

## 生成`compile_commands.json`

比如clangd此类工具需要`compile_commands.json`来获得编译具体某个文件的命令,可以从`build/soong/docs/compdb.md`找到生成方法。

通过设置环境变量，可以开启 compdb 文件的生成：

```bash
export SOONG_GEN_COMPDB=1
export SOONG_GEN_COMPDB_DEBUG=1
```

还可以通过设置环境变量，让 Soong 在生成时创建一个指向 compdb 文件的符号链接：

```bash
export SOONG_LINK_COMPDB_TO=$ANDROID_HOST_OUT
```

然后触发一次空编译即可：

```bash
make nothing
```
