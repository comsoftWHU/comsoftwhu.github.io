---
layout: default
title: AOSP启动流程
nav_order: 1
parent: AndroidRuntime
grand_parent: AOSP
author: Anonymous Committer
---
# AOSP启动流程

我们只关注怎么启动ART虚拟机运行APP的。
<!-- TOC -->

- [AOSP启动流程](#aosp%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B)
    - [内核启动之后](#%E5%86%85%E6%A0%B8%E5%90%AF%E5%8A%A8%E4%B9%8B%E5%90%8E)
        - [init进程](#init%E8%BF%9B%E7%A8%8B)
            - [FirstStageMain](#firststagemain)
            - [SetupSelinux](#setupselinux)
            - [SecondStageMain](#secondstagemain)
            - [LoadBootScripts](#loadbootscripts)
        - [Zygote进程](#zygote%E8%BF%9B%E7%A8%8B)
    - [运行App](#%E8%BF%90%E8%A1%8Capp)
        - [app_process进程](#app_process%E8%BF%9B%E7%A8%8B)
            - [AppRuntime](#appruntime)
            - [startVm函数](#startvm%E5%87%BD%E6%95%B0)
            - [JNI_CreateJavaVM函数](#jni_createjavavm%E5%87%BD%E6%95%B0)
        - [ART虚拟机](#art%E8%99%9A%E6%8B%9F%E6%9C%BA)
            - [JNI_CreateJavaVM in ART](#jni_createjavavm-in-art)
            - [Runtime::Create](#runtimecreate)
            - [创建Heap](#%E5%88%9B%E5%BB%BAheap)
            - [Runtime::start](#runtimestart)
            - [执行Java业务代码](#%E6%89%A7%E8%A1%8Cjava%E4%B8%9A%E5%8A%A1%E4%BB%A3%E7%A0%81)
            - [ArtMethod::Invoke](#artmethodinvoke)
            - [解释器入口 art::interpreter::EnterInterpreterFromInvoke](#%E8%A7%A3%E9%87%8A%E5%99%A8%E5%85%A5%E5%8F%A3-artinterpreterenterinterpreterfrominvoke)
            - [解释器实际执行](#%E8%A7%A3%E9%87%8A%E5%99%A8%E5%AE%9E%E9%99%85%E6%89%A7%E8%A1%8C)

<!-- /TOC -->

## 内核启动之后

从PID为1的第一个进程init开始

### init进程
system/core/init/main.cpp 中的 main() 函数就是 Android系统 用户态的第一个进程 init的主函数。

```cpp
int main(int argc, char** argv) {
    // Boost prio which will be restored later
    setpriority(PRIO_PROCESS, 0, -20);
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }
    if (argc > 1) {
        if (!strcmp(argv[1], "subcontext")) {
            android::base::InitLogging(argv, &android::base::KernelLogger);
            const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();
            return SubcontextMain(argc, argv, &function_map);
        }
        if (!strcmp(argv[1], "selinux_setup")) {
            return SetupSelinux(argv);
        }
        if (!strcmp(argv[1], "second_stage")) {
            return SecondStageMain(argc, argv);
        }
    }
    return FirstStageMain(argc, argv);
}
```

#### FirstStageMain
FirstStageMain所实现的 第一阶段init，主要负责在用户态做最最基础的环境准备和挂载一些文件系统（如 /vendor、/product、/mnt/vendor、/mnt/product 等）。

```cpp
    const char* path = "/system/bin/init";
    const char* args[] = {path, "selinux_setup", nullptr};
    execv(path, const_cast<char**>(args));
```
FirstStageMain 又会通过execv系统调用重新执行init进入selinux_setup阶段


#### SetupSelinux
SetupSelinux函数负责SELinux（Security-Enhanced Linux）的策略设置。
和ART基本上无关
SetupSelinux中又会通过init进入second_stage阶段。

#### SecondStageMain

SecondStageMain是Android 用户空间启动流程中真正完成系统初始化、服务管理和事件循环的核心入口。主要工作可以归纳为以下几个方面：


1. 环境与信号准备
2. 启动状态标记与调试支持
3. **挂载额外分区，比如使用APEX包热更新art的路径/apex/**
4. Property 服务
5. SELinux 支持
6. **脚本与服务管理**
 - **解析并加载所有 init.rc 及 \*.rc 脚本，创建 Action（事件-动作）和 Service（服务）对象**
7. 事件排队
8. 主循环：
    - I/O 多路复用与事件分发，通过 epoll 监听：子进程退出（SIGCHLD via signalfd）initctl socket（ctl.start、ctl.stop、status 等外部命令）property socket（属性变化触发）
在循环中：
	1. 回收已退出子进程
	2. **执行一个脚本命令（ActionManager::ExecuteOneCommand）**
	3. 处理关机命令
	4. 调度进程重启等后台操作
	5. 根据下一次待执行动作时间计算 epoll 超时，然后继续等待

#### LoadBootScripts

关键在于LoadBootScripts函数加载启动脚本, 在后续主循环中会运行其中的命令。

```cpp
static void LoadBootScripts(ActionManager& action_manager, ServiceList& service_list) {
    Parser parser = CreateParser(action_manager, service_list);

    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    if (bootscript.empty()) {
        parser.ParseConfig("/system/etc/init/hw/init.rc");
        if (!parser.ParseConfig("/system/etc/init")) {
            late_import_paths.emplace_back("/system/etc/init");
        }
        // late_import is available only in Q and earlier release. As we don't
        // have system_ext in those versions, skip late_import for system_ext.
        parser.ParseConfig("/system_ext/etc/init");
        if (!parser.ParseConfig("/vendor/etc/init")) {
            late_import_paths.emplace_back("/vendor/etc/init");
        }
        if (!parser.ParseConfig("/odm/etc/init")) {
            late_import_paths.emplace_back("/odm/etc/init");
        }
        if (!parser.ParseConfig("/product/etc/init")) {
            late_import_paths.emplace_back("/product/etc/init");
        }
    } else {
        parser.ParseConfig(bootscript);
    }
}
```

对于ART相关的启动脚本，有init.zygote32.rc（用于32位的机器）, init.zygote64.rc,init.zygote64_32.rc, 
其中init.zygote64_32.rc内容如下

```bash
import /system/etc/init/hw/init.zygote64.rc

service zygote_secondary /system/bin/app_process32 -Xzygote /system/bin --zygote --socket-name=zygote_secondary --enable-lazy-preload
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote_secondary stream 660 root system
    socket usap_pool_secondary stream 660 root system
    onrestart restart zygote
    task_profiles ProcessCapacityHigh MaxPerformance
```

首先会import  /system/etc/init/hw/init.zygote64.rc

```bash 
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    onrestart exec_background - system system -- /system/bin/vdc volume abort_fuse
    onrestart write /sys/power/state on
    # NOTE: If the wakelock name here is changed, then also
    # update it in SystemSuspend.cpp
    onrestart write /sys/power/wake_lock zygote_kwl
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart --only-if-running media.tuner
    onrestart restart netd
    onrestart restart wificond
    task_profiles ProcessCapacityHigh MaxPerformance
    critical window=${zygote.critical_window.minute:-off} target=zygote-fatal
```

这里的脚本是启动了zygote进程， 运行命令是

```bash
/system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
```
然后启动zygote_secondary，这是32位的zygote,用于兼容32位的APP，运行命令为
```bash
/system/bin/app_process32 -Xzygote /system/bin --zygote --socket-name=zygote_secondary --enable-lazy-preload
```

### Zygote进程

Zygote 是系统启动后由 init 进程启动的第一个ART实例，它会预加载常用类和资源，启动时就把大量系统框架的 Java 类、资源、JNI 库等一次性加载到内存。这样做可以：
 1. 加快后续应用启动：应用进程 fork 之后无需重新加载这些内容；
 2. 节省物理内存：通过 Linux 的 COW（Copy-On-Write）机制，多进程共享同一份只读数据。

## 运行App

### app_process进程

app_process简化的主函数如下
```cpp
......
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
......
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (!className.empty()) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
......
```

AppRuntime包装了java进程，runtime.start()中启动java方法。


#### AppRuntime

AppRuntime继承自AndroidRuntime， start()定义在AndroidRuntime中。

核心流程包括调用jni的接口创建虚拟机，然后根据传入类名，找到类，使用JNI的CallStaticVoidMethod的接口调用main方法。

```cpp
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ......
    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) {
        return;
    }
    onVmCreated(env);

    /*
     * Register android functions.
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    /*
     * We want to call main() with a String array with arguments in it.
     * At present we have two arguments, the class name and an option string.
     * Create an array to hold them.
     */
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);

    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).c_str());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }

    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    // 找到我们启动的类
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            // 使用JNI接口invoke启动类里的main方法
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
        }
    }
    free(slashClassName);

    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}
```


对于Host端来说这个函数的实现会简单一些, 在onStarted()中调用callMain()函数，也是上述找到类，然后调用main方法流程。
```cpp
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote) {
    JNIEnv* env = AndroidRuntime::getJNIEnv();

    auto method_binding_format = getJavaProperty(env, "method_binding_format");

    setJniMethodFormat(method_binding_format);

    // Register native functions.
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android native methods\n");
    }
    onStarted();
}
```

#### startVm函数

AndroidRuntime::startVm 是 Android 真正“拉起”ART/Dalvik 虚拟机的核心函数，它主要挖出所有可配置的启动选项（堆、GC、AOT/JIT、调试、JNI、locale、fingerprint 等）；把它们包装成 JNI 认识的 JavaVMInitArgs；然后调用 JNI_CreateJavaVM，在这个进程内启动一颗全功能的 ART 虚拟机。

预分配各种配置缓冲区
```cpp
char propBuf[PROPERTY_VALUE_MAX];
......
char heapstartsizeOptsBuf[sizeof("-Xms")-1 + PROPERTY_VALUE_MAX];
char heapsizeOptsBuf[sizeof("-Xmx")-1 + PROPERTY_VALUE_MAX];
// … 一大堆类似的 buffer …
```

这些局部数组保证能容纳来自系统属性（property_get）或配置标志（server-configurable flags）的字符串，比如堆大小、JIT 参数、JNI 检查选项等。

读取 profile/classpath/JIT/AOT 相关开关
```cpp
std::string profile_boot_class_path_flag = GetServerConfigurableFlag(...);
bool profile_boot_class_path = ParseBool(...);
if (profile_boot_class_path) {
  addOption("-Xcompiler-option");
  addOption("--count-hotness-in-compiled-code");
  // …
}
```
根据 APEX、properties 或运行时 flag 决定是否开启“Profile Boot Class Path”、是否使用 JIT Zygote 镜像等策略。

处理 JNI、调试和锁分析选项
```cpp
const bool checkJni = GetBoolProperty("dalvik.vm.checkjni", false);
if (checkJni) {
  addOption("-Xcheck:jni");
}
// …
if (zygote) {
  addOption("-XjdwpOptions:suspend=n,server=y");
  parseRuntimeOption("dalvik.vm.opaque-jni-ids", opaqueJniIds, "-Xopaque-jni-ids:", "swapable");
}
```

Heap/Garbage-Collection/Tuning 参数
```cpp
parseRuntimeOption("dalvik.vm.heapstartsize", heapstartsizeOptsBuf, "-Xms", "4m");
parseRuntimeOption("dalvik.vm.heapsize",        heapsizeOptsBuf,    "-Xmx", "16m");
parseRuntimeOption("dalvik.vm.heapgrowthlimit", heapgrowthlimitOptsBuf, "-XX:HeapGrowthLimit=");
// 还有 BackgroundGC、HeapTargetUtilization、FinalizerTimeout 等
```
这些参数控制 ART 堆的初始大小、最大大小、增长策略，以及 GC 行为。

Dex2Oat/Compiler/AOT 选项
```cpp
parseCompilerOption("dalvik.vm.dex2oat-filter", dex2oatCompilerFilterBuf, "--compiler-filter=", "-Xcompiler-option");
parseCompilerOption("dalvik.vm.dex2oat-threads", dex2oatThreadsBuf, "-j", "-Xcompiler-option");
// …
parseCompilerRuntimeOption("dalvik.vm.image-dex2oat-Xms", dex2oatXmsImageFlagsBuf, "-Xms", "-Ximage-compiler-option");
// …
```

这些指令影响在运行时（或生成 boot image 时）如何对 class/dex 文件进行 AOT 编译：线程数、CPU 限制、编译 filter等。

Locale、Fingerprint、NativeBridge

```cpp
// 设置 -Duser.locale=<locale>
strcpy(localeOption, "-Duser.locale=");
strncat(localeOption, readLocale().c_str(), PROPERTY_VALUE_MAX);
addOption(localeOption);

// 传递 system fingerprint 给 VM，用于 ANR 等日志
std::string fingerprint = GetProperty("ro.build.fingerprint", "");
fingerprintBuf = "-Xfingerprint:" + fingerprint;
addOption(fingerprintBuf.c_str());

// 如果配置了 NativeBridge，还要加上 -XX:NativeBridge=<library>
```
让 VM 知道当前语言环境、设备指纹，以及是否启用 Native Bridge（如 Houdini）支持。

组装 JavaVMInitArgs 并调用 JNI_CreateJavaVM

```cpp
initArgs.version          = JNI_VERSION_1_4;
initArgs.options          = mOptions.editArray();
initArgs.nOptions         = mOptions.size();
initArgs.ignoreUnrecognized = JNI_FALSE;

if (JNI_CreateJavaVM(pJavaVM, pEnv, &initArgs) < 0) {
  ALOGE("JNI_CreateJavaVM failed");
  return -1;
}
```

把之前 addOption(...) 收集的一系列 -X、--XX:、-D 等 VM 启动参数传给 JNI 层，真正初始化一个 JavaVM 实例，并获取主线程的 JNIEnv*。

返回与错误处理
  - 如果 JNI_CreateJavaVM 成功，startVm 返回 0，后续会调用 onVmCreated(env) 继续注册 native 方法、调用 main() 等。
	- 失败则打印错误并返回非零，让调用者（AndroidRuntime::start）可以进行适当的退出或报警。

#### JNI_CreateJavaVM函数

JNI_CreateJavaVM函数在哪呢？
在libnativehelper/JniInvocation.c中，JNI_CreateJavaVM实际上调用的JniInvocationImpl的JNI_CreateJavaVM函数指针成员。
```cpp
static struct JniInvocationImpl g_impl;
......
jint JNI_CreateJavaVM(JavaVM** p_vm, JNIEnv** p_env, void* vm_args) {
  ALOG_ALWAYS_FATAL_IF(NULL == g_impl.JNI_CreateJavaVM, "Runtime library not loaded.");
  return g_impl.JNI_CreateJavaVM(p_vm, p_env, vm_args);
}
```
在JniInvocation的Init函数中，使用dlopen手动加载libart.so, 找到一些函数符号，然后JniInvocationImpl的JNI_CreateJavaVM成员函数被赋值为了libart.so中的JNI_CreateJavaVM函数。

```cpp
// Name the default library providing the JNI Invocation API.
static const char* kDefaultJniInvocationLibrary = "libart.so";
static const char* kDebugJniInvocationLibrary = "libartd.so";
......
bool JniInvocationInit(struct JniInvocationImpl* instance, const char* library_name) {
  library_name = JniInvocationGetLibrary(library_name, buffer);
  DlLibrary library = DlOpenLibrary(library_name);
  ......
  DlSymbol JNI_GetDefaultJavaVMInitArgs_ = FindSymbol(library, "JNI_GetDefaultJavaVMInitArgs");
  if (JNI_GetDefaultJavaVMInitArgs_ == NULL) {
    return false;
  }

  DlSymbol JNI_CreateJavaVM_ = FindSymbol(library, "JNI_CreateJavaVM");
  if (JNI_CreateJavaVM_ == NULL) {
    return false;
  }

  DlSymbol JNI_GetCreatedJavaVMs_ = FindSymbol(library, "JNI_GetCreatedJavaVMs");
  if (JNI_GetCreatedJavaVMs_ == NULL) {
    return false;
  }

  instance->jni_provider_library_name = library_name;
  instance->jni_provider_library = library;
  instance->JNI_GetDefaultJavaVMInitArgs = (jint (*)(void *)) JNI_GetDefaultJavaVMInitArgs_;
  instance->JNI_CreateJavaVM = (jint (*)(JavaVM**, JNIEnv**, void*)) JNI_CreateJavaVM_;
  instance->JNI_GetCreatedJavaVMs = (jint (*)(JavaVM**, jsize, jsize*)) JNI_GetCreatedJavaVMs_;

  return true;
}
```
所以从这里控制流就到了art的范围了。

### ART虚拟机

#### JNI_CreateJavaVM in ART

art中JNI_CreateJavaVM函数在art/runtime/jni/java_vm_ext.cc中

```cpp
// JNI Invocation interface.
extern "C" EXPORT jint JNI_CreateJavaVM(JavaVM** p_vm, JNIEnv** p_env, void* vm_args) {
    ......
  const JavaVMInitArgs* args = static_cast<JavaVMInitArgs*>(vm_args);
    ......
  RuntimeOptions options;
  for (int i = 0; i < args->nOptions; ++i) {
    JavaVMOption* option = &args->options[i];
    options.push_back(std::make_pair(std::string(option->optionString), option->extraInfo));
  }
  bool ignore_unrecognized = args->ignoreUnrecognized;
  if (!Runtime::Create(options, ignore_unrecognized)) {
    return JNI_ERR;
  }
   ......
  android::InitializeNativeLoader();
  Runtime* runtime = Runtime::Current();
  ......
  bool started = runtime->Start();
  if (!started) {
    delete Thread::Current()->GetJniEnv();
    delete runtime->GetJavaVM();
    LOG(WARNING) << "CreateJavaVM failed";
    return JNI_ERR;
  }
  *p_env = Thread::Current()->GetJniEnv();
  *p_vm = runtime->GetJavaVM();
  return JNI_OK;
}

```

这个函数干的主要的事情就是
- 解析并设置启动参数
- 构建ART Runtime，（Runtime::Create）并启动 (Runtime::Start)
- 初始化 native loader（Android 的 “Native Loader” 就是 ART 为了让 Java 层能安全、可控、高效地加载和调用本地 .so 库，而在底层封装的一套动态链接器管理机制）
- 最终把可用的 JavaVM* 与 JNIEnv* 返回给上层，让后续 Java 代码真正跑起来。

#### Runtime::Create()

Runtime::Create()核心是Runtime::Init()函数，这根据传入的参数初始化Runtime各种成员

```cpp


bool Runtime::Init(RuntimeArgumentMap&& runtime_options_in) {
  // (b/30160149): protect subprocesses from modifications to LD_LIBRARY_PATH, etc.
  // Take a snapshot of the environment at the time the runtime was created, for use by Exec, etc.
  env_snapshot_.TakeSnapshot();

#ifdef ART_PAGE_SIZE_AGNOSTIC
  gPageSize.AllowAccess();
#endif

  using Opt = RuntimeArgumentMap;
  Opt runtime_options(std::move(runtime_options_in));
  ScopedTrace trace(__FUNCTION__);
  CHECK_EQ(static_cast<size_t>(sysconf(_SC_PAGE_SIZE)), gPageSize);

  // Reload all the flags value (from system properties and device configs).
  ReloadAllFlags(__FUNCTION__);

  deny_art_apex_data_files_ = runtime_options.Exists(Opt::DenyArtApexDataFiles);
  if (deny_art_apex_data_files_) {
    // We will run slower without those files if the system has taken an ART APEX update.
    LOG(WARNING) << "ART APEX data files are untrusted.";
  }

  // Early override for logging output.
  if (runtime_options.Exists(Opt::UseStderrLogger)) {
    android::base::SetLogger(android::base::StderrLogger);
  }

  MemMap::Init(); // MemMap是Runtime管理所有内存映射的数据结构（包括堆内存空间）

  verifier_missing_kthrow_fatal_ = runtime_options.GetOrDefault(Opt::VerifierMissingKThrowFatal);
  force_java_zygote_fork_loop_ = runtime_options.GetOrDefault(Opt::ForceJavaZygoteForkLoop);
  perfetto_hprof_enabled_ = runtime_options.GetOrDefault(Opt::PerfettoHprof);
  perfetto_javaheapprof_enabled_ = runtime_options.GetOrDefault(Opt::PerfettoJavaHeapStackProf);

  // Try to reserve a dedicated fault page. This is allocated for clobbered registers and sentinels.
  // If we cannot reserve it, log a warning.
  // Note: We allocate this first to have a good chance of grabbing the page. The address (0xebad..)
  //       is out-of-the-way enough that it should not collide with boot image mapping.
  // Note: Don't request an error message. That will lead to a maps dump in the case of failure,
  //       leading to logspam.
  // 保留一页(Sentinel) PROT_NONE，用于捕获寄存器泄漏或堆栈溢出时的故障。
  {
    const uintptr_t sentinel_addr =
        RoundDown(static_cast<uintptr_t>(Context::kBadGprBase), gPageSize);
    protected_fault_page_ = MemMap::MapAnonymous("Sentinel fault page",
                                                 reinterpret_cast<uint8_t*>(sentinel_addr),
                                                 gPageSize,
                                                 PROT_NONE,
                                                 /*low_4gb=*/ true,
                                                 /*reuse=*/ false,
                                                 /*reservation=*/ nullptr,
                                                 /*error_msg=*/ nullptr);
    if (!protected_fault_page_.IsValid()) {
      LOG(WARNING) << "Could not reserve sentinel fault page";
    } else if (reinterpret_cast<uintptr_t>(protected_fault_page_.Begin()) != sentinel_addr) {
      LOG(WARNING) << "Could not reserve sentinel fault page at the right address.";
      protected_fault_page_.Reset();
    }
  }

  VLOG(startup) << "Runtime::Init -verbose:startup enabled";

  // “准原子”（quasi‐atomic）操作的抽象，专门用来在那些不完全支持 64 位原子操作或者老旧架构上，保证对 64 位整型变量的 “不撕裂” 读、写和比较并交换（CAS）行为。
  QuasiAtomic::Startup();

  // 在 ART 中，所有的 .dex 文件最终都会或多或少地关联到一个或多个“OAT”（Optimized ART）文件——这些文件包含了 AOT-或 JIT-编译后的本地代码、Profile 信息、校验依赖（VerifierDeps）以及常用数据结构的预先布局。OatFileManager 就是把这套“打开/注册/查找/卸载 OAT 文件”流程全部 统一管理起来 的核心类。
  oat_file_manager_ = new OatFileManager();

  // JniIdManager 是 ART 在 JNI 层用来 管理和翻译 Java 侧的 jmethodID 和 jfieldID（这两个在 JNI 中分别代表方法和字段的句柄）与 ART 内部的 ArtMethod*、ArtField* 指针之间的双向映射工具。
  jni_id_manager_.reset(new jni::JniIdManager());

  Thread::SetSensitiveThreadHook(runtime_options.GetOrDefault(Opt::HookIsSensitiveThread));

  // Monitor 就是用来实现 Java 里每个对象的内置锁（即 synchronized(obj){…}、obj.wait()/notify() 等那些语义）
  Monitor::Init(runtime_options.GetOrDefault(Opt::LockProfThreshold),
                runtime_options.GetOrDefault(Opt::StackDumpLockProfThreshold));

  image_locations_ = runtime_options.ReleaseOrDefault(Opt::Image);

  SetInstructionSet(runtime_options.GetOrDefault(Opt::ImageInstructionSet));

  // 加载boot_class
  boot_class_path_ = runtime_options.ReleaseOrDefault(Opt::BootClassPath);
  boot_class_path_locations_ = runtime_options.ReleaseOrDefault(Opt::BootClassPathLocations);
  DCHECK(boot_class_path_locations_.empty() ||
         boot_class_path_locations_.size() == boot_class_path_.size());
  if (boot_class_path_.empty()) {
    LOG(ERROR) << "Boot classpath is empty";
    return false;
  }

  boot_class_path_files_ =
      FileFdsToFileObjects(runtime_options.ReleaseOrDefault(Opt::BootClassPathFds));
  if (!boot_class_path_files_.empty() && boot_class_path_files_.size() != boot_class_path_.size()) {
    LOG(ERROR) << "Number of FDs specified in -Xbootclasspathfds must match the number of JARs in "
               << "-Xbootclasspath.";
    return false;
  }

  boot_class_path_image_files_ =
      FileFdsToFileObjects(runtime_options.ReleaseOrDefault(Opt::BootClassPathImageFds));
  boot_class_path_vdex_files_ =
      FileFdsToFileObjects(runtime_options.ReleaseOrDefault(Opt::BootClassPathVdexFds));
  boot_class_path_oat_files_ =
      FileFdsToFileObjects(runtime_options.ReleaseOrDefault(Opt::BootClassPathOatFds));
  CHECK(boot_class_path_image_files_.empty() ||
        boot_class_path_image_files_.size() == boot_class_path_.size());
  CHECK(boot_class_path_vdex_files_.empty() ||
        boot_class_path_vdex_files_.size() == boot_class_path_.size());
  CHECK(boot_class_path_oat_files_.empty() ||
        boot_class_path_oat_files_.size() == boot_class_path_.size());

  class_path_string_ = runtime_options.ReleaseOrDefault(Opt::ClassPath);
  properties_ = runtime_options.ReleaseOrDefault(Opt::PropertiesList);

  compiler_callbacks_ = runtime_options.GetOrDefault(Opt::CompilerCallbacksPtr);
  must_relocate_ = runtime_options.GetOrDefault(Opt::Relocate);
  is_zygote_ = runtime_options.Exists(Opt::Zygote);
  is_primary_zygote_ = runtime_options.Exists(Opt::PrimaryZygote);
  is_explicit_gc_disabled_ = runtime_options.Exists(Opt::DisableExplicitGC);
  is_eagerly_release_explicit_gc_disabled_ =
      runtime_options.Exists(Opt::DisableEagerlyReleaseExplicitGC);
  image_dex2oat_enabled_ = runtime_options.GetOrDefault(Opt::ImageDex2Oat);
  dump_native_stack_on_sig_quit_ = runtime_options.GetOrDefault(Opt::DumpNativeStackOnSigQuit);
  allow_in_memory_compilation_ = runtime_options.Exists(Opt::AllowInMemoryCompilation);

  if (is_zygote_ || runtime_options.Exists(Opt::OnlyUseTrustedOatFiles)) {
    oat_file_manager_->SetOnlyUseTrustedOatFiles();
  }

  vfprintf_ = runtime_options.GetOrDefault(Opt::HookVfprintf);
  exit_ = runtime_options.GetOrDefault(Opt::HookExit);
  abort_ = runtime_options.GetOrDefault(Opt::HookAbort);

  default_stack_size_ = runtime_options.GetOrDefault(Opt::StackSize);

  compiler_executable_ = runtime_options.ReleaseOrDefault(Opt::Compiler);
  compiler_options_ = runtime_options.ReleaseOrDefault(Opt::CompilerOptions);
  for (const std::string& option : Runtime::Current()->GetCompilerOptions()) {
    if (option == "--debuggable") {
      SetRuntimeDebugState(RuntimeDebugState::kJavaDebuggableAtInit);
      break;
    }
  }
  image_compiler_options_ = runtime_options.ReleaseOrDefault(Opt::ImageCompilerOptions);

  finalizer_timeout_ms_ = runtime_options.GetOrDefault(Opt::FinalizerTimeoutMs);
  max_spins_before_thin_lock_inflation_ =
      runtime_options.GetOrDefault(Opt::MaxSpinsBeforeThinLockInflation);

  monitor_list_ = new MonitorList;
  monitor_pool_ = MonitorPool::Create();
  thread_list_ = new ThreadList(GetThreadSuspendTimeout(&runtime_options));


  // InternTable 在 ART（Android Runtime）中专门用来实现 Java 里 字符串驻留（string interning） 机制的：
	// 1. 保证“相同内容”字符串对象唯一
	//	Java 规范要求，凡是通过字面量 "foo" 或者调用 String.intern() 得到的字符串，如果内容相同，它们的引用必须相同，方便快速比较和减少内存占用。
	// InternTable 维护了一个全局的哈希集合，只要有新字符串需要驻留，就先查表：
	// - 如果已存在，直接返回表中已有的 String 对象；
	// - 否则就把新字符串插入表里，再返回它自身。
  intern_table_ = new InternTable;
  
  monitor_timeout_enable_ = runtime_options.GetOrDefault(Opt::MonitorTimeoutEnable);
  int monitor_timeout_ms = runtime_options.GetOrDefault(Opt::MonitorTimeout);
  if (monitor_timeout_ms < Monitor::kMonitorTimeoutMinMs) {
    LOG(WARNING) << "Monitor timeout too short: Increasing";
    monitor_timeout_ms = Monitor::kMonitorTimeoutMinMs;
  }
  if (monitor_timeout_ms >= Monitor::kMonitorTimeoutMaxMs) {
    LOG(WARNING) << "Monitor timeout too long: Decreasing";
    monitor_timeout_ms = Monitor::kMonitorTimeoutMaxMs - 1;
  }
  monitor_timeout_ns_ = MsToNs(monitor_timeout_ms);

  verify_ = runtime_options.GetOrDefault(Opt::Verify);

  target_sdk_version_ = runtime_options.GetOrDefault(Opt::TargetSdkVersion);

  // Set hidden API enforcement policy. The checks are disabled by default and
  // we only enable them if:
  // (a) runtime was started with a command line flag that enables the checks, or
  // (b) Zygote forked a new process that is not exempt (see ZygoteHooks).
  hidden_api_policy_ = runtime_options.GetOrDefault(Opt::HiddenApiPolicy);
  DCHECK_IMPLIES(is_zygote_, hidden_api_policy_ == hiddenapi::EnforcementPolicy::kDisabled);

  // Set core platform API enforcement policy. The checks are disabled by default and
  // can be enabled with a command line flag. AndroidRuntime will pass the flag if
  // a system property is set.
  core_platform_api_policy_ = runtime_options.GetOrDefault(Opt::CorePlatformApiPolicy);
  if (core_platform_api_policy_ != hiddenapi::EnforcementPolicy::kDisabled) {
    LOG(INFO) << "Core platform API reporting enabled, enforcing="
        << (core_platform_api_policy_ == hiddenapi::EnforcementPolicy::kEnabled ? "true" : "false");
  }

  // Dex2Oat's Runtime does not need the signal chain or the fault handler
  // and it passes the `NoSigChain` option to `Runtime` to indicate this.
  no_sig_chain_ = runtime_options.Exists(Opt::NoSigChain);
  force_native_bridge_ = runtime_options.Exists(Opt::ForceNativeBridge);

  Split(runtime_options.GetOrDefault(Opt::CpuAbiList), ',', &cpu_abilist_);

  fingerprint_ = runtime_options.ReleaseOrDefault(Opt::Fingerprint);

  // 设置解释器模式
  if (runtime_options.GetOrDefault(Opt::Interpret)) {
    GetInstrumentation()->ForceInterpretOnly();
  }

  zygote_max_failed_boots_ = runtime_options.GetOrDefault(Opt::ZygoteMaxFailedBoots);
  experimental_flags_ = runtime_options.GetOrDefault(Opt::Experimental);
  is_low_memory_mode_ = runtime_options.Exists(Opt::LowMemoryMode);
  madvise_willneed_total_dex_size_ = runtime_options.GetOrDefault(Opt::MadviseWillNeedVdexFileSize);
  madvise_willneed_odex_filesize_ = runtime_options.GetOrDefault(Opt::MadviseWillNeedOdexFileSize);
  madvise_willneed_art_filesize_ = runtime_options.GetOrDefault(Opt::MadviseWillNeedArtFileSize);

  jni_ids_indirection_ = runtime_options.GetOrDefault(Opt::OpaqueJniIds);
  automatically_set_jni_ids_indirection_ =
      runtime_options.GetOrDefault(Opt::AutoPromoteOpaqueJniIds);

  plugins_ = runtime_options.ReleaseOrDefault(Opt::Plugins);
  agent_specs_ = runtime_options.ReleaseOrDefault(Opt::AgentPath);
  // TODO Add back in -agentlib
  // for (auto lib : runtime_options.ReleaseOrDefault(Opt::AgentLib)) {
  //   agents_.push_back(lib);
  // }

  float foreground_heap_growth_multiplier;
  if (is_low_memory_mode_ && !runtime_options.Exists(Opt::ForegroundHeapGrowthMultiplier)) {
    // If low memory mode, use 1.0 as the multiplier by default.
    foreground_heap_growth_multiplier = 1.0f;
  } else {
    // Extra added to the default heap growth multiplier for concurrent GC
    // compaction algorithms. This is done for historical reasons.
    // TODO: remove when we revisit heap configurations.
    foreground_heap_growth_multiplier =
        runtime_options.GetOrDefault(Opt::ForegroundHeapGrowthMultiplier) + 1.0f;
  }

  // 读取GC的设置
  XGcOption xgc_option = runtime_options.GetOrDefault(Opt::GcOption);

  // Generational CC collection is currently only compatible with Baker read barriers.
  bool use_generational_gc = (kUseBakerReadBarrier || gUseUserfaultfd) &&
                             xgc_option.generational_gc && ShouldUseGenerationalGC();

  // Cache the apex versions.
  InitializeApexVersions();

  BackgroundGcOption background_gc =
      gUseReadBarrier ? BackgroundGcOption(gc::kCollectorTypeCCBackground) :
                        (gUseUserfaultfd ? BackgroundGcOption(gc::kCollectorTypeCMCBackground) :
                                           runtime_options.GetOrDefault(Opt::BackgroundGc));

  // 创建堆
  heap_ = new gc::Heap(runtime_options.GetOrDefault(Opt::MemoryInitialSize),
                       runtime_options.GetOrDefault(Opt::HeapGrowthLimit),
                       runtime_options.GetOrDefault(Opt::HeapMinFree),
                       runtime_options.GetOrDefault(Opt::HeapMaxFree),
                       runtime_options.GetOrDefault(Opt::HeapTargetUtilization),
                       foreground_heap_growth_multiplier,
                       runtime_options.GetOrDefault(Opt::StopForNativeAllocs),
                       runtime_options.GetOrDefault(Opt::MemoryMaximumSize),
                       runtime_options.GetOrDefault(Opt::NonMovingSpaceCapacity),
                       GetBootClassPath(),
                       GetBootClassPathLocations(),
                       GetBootClassPathFiles(),
                       GetBootClassPathImageFiles(),
                       GetBootClassPathVdexFiles(),
                       GetBootClassPathOatFiles(),
                       image_locations_,
                       instruction_set_,
                       // Override the collector type to CC if the read barrier config.
                       gUseReadBarrier ? gc::kCollectorTypeCC : xgc_option.collector_type_,
                       background_gc,
                       runtime_options.GetOrDefault(Opt::LargeObjectSpace),
                       runtime_options.GetOrDefault(Opt::LargeObjectThreshold),
                       runtime_options.GetOrDefault(Opt::ParallelGCThreads),
                       runtime_options.GetOrDefault(Opt::ConcGCThreads),
                       runtime_options.Exists(Opt::LowMemoryMode),
                       runtime_options.GetOrDefault(Opt::LongPauseLogThreshold),
                       runtime_options.GetOrDefault(Opt::LongGCLogThreshold),
                       runtime_options.Exists(Opt::IgnoreMaxFootprint),
                       runtime_options.GetOrDefault(Opt::AlwaysLogExplicitGcs),
                       runtime_options.GetOrDefault(Opt::UseTLAB),
                       xgc_option.verify_pre_gc_heap_,
                       xgc_option.verify_pre_sweeping_heap_,
                       xgc_option.verify_post_gc_heap_,
                       xgc_option.verify_pre_gc_rosalloc_,
                       xgc_option.verify_pre_sweeping_rosalloc_,
                       xgc_option.verify_post_gc_rosalloc_,
                       xgc_option.gcstress_,
                       xgc_option.measure_,
                       runtime_options.GetOrDefault(Opt::EnableHSpaceCompactForOOM),
                       use_generational_gc,
                       runtime_options.GetOrDefault(Opt::HSpaceCompactForOOMMinIntervalsMs),
                       runtime_options.Exists(Opt::DumpRegionInfoBeforeGC),
                       runtime_options.Exists(Opt::DumpRegionInfoAfterGC));

  dump_gc_performance_on_shutdown_ = runtime_options.Exists(Opt::DumpGCPerformanceOnShutdown);

  // JDWP 是 “Java Debug Wire Protocol” 的缩写，也就是 Java 调试线协议。它定义了一套在调试器（比如 Android Studio、jdb、DDMS）和 JVM/ART 之间交换调试命令和事件的网络协议。通过 JDWP，调试器可以远程：
	// • 设置断点、单步执行
	// • 查询和修改 变量值
	// • 捕获 异常、线程状态
	// • 监控 类加载、方法调用等
  bool has_explicit_jdwp_options = runtime_options.Get(Opt::JdwpOptions) != nullptr;
  jdwp_options_ = runtime_options.GetOrDefault(Opt::JdwpOptions);
  jdwp_provider_ = CanonicalizeJdwpProvider(runtime_options.GetOrDefault(Opt::JdwpProvider),
                                            IsJavaDebuggable());
  switch (jdwp_provider_) {
    case JdwpProvider::kNone: {
      VLOG(jdwp) << "Disabling all JDWP support.";
      if (!jdwp_options_.empty()) {
        bool has_transport = jdwp_options_.find("transport") != std::string::npos;
        std::string adb_connection_args =
            std::string("  -XjdwpProvider:adbconnection -XjdwpOptions:") + jdwp_options_;
        if (has_explicit_jdwp_options) {
          LOG(WARNING) << "Jdwp options given when jdwp is disabled! You probably want to enable "
                      << "jdwp with one of:" << std::endl
                      << "  -Xplugin:libopenjdkjvmti" << (kIsDebugBuild ? "d" : "") << ".so "
                      << "-agentpath:libjdwp.so=" << jdwp_options_ << std::endl
                      << (has_transport ? "" : adb_connection_args);
        }
      }
      break;
    }
    case JdwpProvider::kAdbConnection: {
      constexpr const char* plugin_name = kIsDebugBuild ? "libadbconnectiond.so"
                                                        : "libadbconnection.so";
      plugins_.push_back(Plugin::Create(plugin_name));
      break;
    }
    case JdwpProvider::kUnset: {
      LOG(FATAL) << "Illegal jdwp provider " << jdwp_provider_ << " was not filtered out!";
    }
  }
  callbacks_->AddThreadLifecycleCallback(Dbg::GetThreadLifecycleCallback());

  jit_options_.reset(jit::JitOptions::CreateFromRuntimeArguments(runtime_options));
  if (IsAotCompiler()) {
    // If we are already the compiler at this point, we must be dex2oat. Don't create the jit in
    // this case.
    // If runtime_options doesn't have UseJIT set to true then CreateFromRuntimeArguments returns
    // null and we don't create the jit.
    jit_options_->SetUseJitCompilation(false);
    jit_options_->SetSaveProfilingInfo(false);
  }

  // Use MemMap arena pool for jit, malloc otherwise. Malloc arenas are faster to allocate but
  // can't be trimmed as easily.
  const bool use_malloc = IsAotCompiler();
  if (use_malloc) {
    arena_pool_.reset(new MallocArenaPool());
    jit_arena_pool_.reset(new MallocArenaPool());
  } else {
    arena_pool_.reset(new MemMapArenaPool(/* low_4gb= */ false));
    jit_arena_pool_.reset(new MemMapArenaPool(/* low_4gb= */ false, "CompilerMetadata"));
  }

  // For 64 bit compilers, it needs to be in low 4GB in the case where we are cross compiling for a
  // 32 bit target. In this case, we have 32 bit pointers in the dex cache arrays which can't hold
  // when we have 64 bit ArtMethod pointers.
  const bool low_4gb = IsAotCompiler() && Is64BitInstructionSet(kRuntimeISA);
  if (gUseUserfaultfd) {
    linear_alloc_arena_pool_.reset(new GcVisitedArenaPool(low_4gb, IsZygote()));
  } else if (low_4gb) {
    linear_alloc_arena_pool_.reset(new MemMapArenaPool(low_4gb));
  }
  linear_alloc_.reset(CreateLinearAlloc());
  startup_linear_alloc_.store(CreateLinearAlloc(), std::memory_order_relaxed);

  small_lrt_allocator_ = new jni::SmallLrtAllocator();

  BlockSignals();
  InitPlatformSignalHandlers();

  // Change the implicit checks flags based on runtime architecture.
  switch (kRuntimeQuickCodeISA) {
    case InstructionSet::kArm64:
      implicit_suspend_checks_ = true;
      FALLTHROUGH_INTENDED;
    case InstructionSet::kArm:
    case InstructionSet::kThumb2:
    case InstructionSet::kRiscv64:
    case InstructionSet::kX86:
    case InstructionSet::kX86_64:
      implicit_null_checks_ = true;
      // Historical note: Installing stack protection was not playing well with Valgrind.
      implicit_so_checks_ = true;
      break;
    default:
      // Keep the defaults.
      break;
  }

#ifdef ART_USE_RESTRICTED_MODE
  // TODO(Simulator): support signal handling and implicit checks.
  implicit_suspend_checks_ = false;
  implicit_null_checks_ = false;
#endif  // ART_USE_RESTRICTED_MODE

  // fault_manager 让 ART 能在遇到内存访问错误时，优雅地抛出 Java 异常或执行虚拟机内部逻辑，而不是直接让进程崩溃。
  fault_manager.Init(!no_sig_chain_);
  if (!no_sig_chain_) {
    if (HandlesSignalsInCompiledCode()) {
      // These need to be in a specific order.  The null point check handler must be
      // after the suspend check and stack overflow check handlers.
      //
      // Note: the instances attach themselves to the fault manager and are handled by it. The
      //       manager will delete the instance on Shutdown().
      if (implicit_suspend_checks_) {
        new SuspensionHandler(&fault_manager);
      }

      if (implicit_so_checks_) {
        new StackOverflowHandler(&fault_manager);
      }

      if (implicit_null_checks_) {
        new NullPointerHandler(&fault_manager);
      }

      if (kEnableJavaStackTraceHandler) {
        new JavaStackTraceHandler(&fault_manager);
      }

      if (interpreter::CanRuntimeUseNterp()) {
        // Nterp code can use signal handling just like the compiled managed code.
        OatQuickMethodHeader* nterp_header = OatQuickMethodHeader::NterpMethodHeader;
        fault_manager.AddGeneratedCodeRange(nterp_header->GetCode(), nterp_header->GetCodeSize());
      }
    }
  }

  verifier_logging_threshold_ms_ = runtime_options.GetOrDefault(Opt::VerifierLoggingThreshold);

  std::string error_msg;
  java_vm_ = JavaVMExt::Create(this, runtime_options, &error_msg);
  if (java_vm_.get() == nullptr) {
    LOG(ERROR) << "Could not initialize JavaVMExt: " << error_msg;
    return false;
  }

  // Add the JniEnv handler.
  // TODO Refactor this stuff.
  java_vm_->AddEnvironmentHook(JNIEnvExt::GetEnvHandler);

  Thread::Startup();

  // ClassLinker needs an attached thread, but we can't fully attach a thread without creating
  // objects. We can't supply a thread group yet; it will be fixed later. Since we are the main
  // thread, we do not get a java peer.
  Thread* self = Thread::Attach("main", false, nullptr, false, /* should_run_callbacks= */ true);
  CHECK_EQ(self->GetThreadId(), ThreadList::kMainThreadId);
  CHECK(self != nullptr);

  self->SetIsRuntimeThread(IsAotCompiler());

  // Set us to runnable so tools using a runtime can allocate and GC by default
  self->TransitionFromSuspendedToRunnable();

  // Now we're attached, we can take the heap locks and validate the heap.
  GetHeap()->EnableObjectValidation();

  CHECK_GE(GetHeap()->GetContinuousSpaces().size(), 1U);

  if (UNLIKELY(IsAotCompiler())) {
    class_linker_ = compiler_callbacks_->CreateAotClassLinker(intern_table_);
  } else {
    class_linker_ = new ClassLinker(
        intern_table_,
        runtime_options.GetOrDefault(Opt::FastClassNotFoundException));
  }
  if (GetHeap()->HasBootImageSpace()) {
    bool result = class_linker_->InitFromBootImage(&error_msg);
    if (!result) {
      LOG(ERROR) << "Could not initialize from image: " << error_msg;
      return false;
    }
    if (kIsDebugBuild) {
      for (auto image_space : GetHeap()->GetBootImageSpaces()) {
        image_space->VerifyImageAllocations();
      }
    }
    {
      ScopedTrace trace2("AddImageStringsToTable");
      for (gc::space::ImageSpace* image_space : heap_->GetBootImageSpaces()) {
        GetInternTable()->AddImageStringsToTable(image_space, VoidFunctor());
      }
    }

    const size_t total_components = gc::space::ImageSpace::GetNumberOfComponents(
        ArrayRef<gc::space::ImageSpace* const>(heap_->GetBootImageSpaces()));
    if (total_components != GetBootClassPath().size()) {
      // The boot image did not contain all boot class path components. Load the rest.
      CHECK_LT(total_components, GetBootClassPath().size());
      size_t start = total_components;
      DCHECK_LT(start, GetBootClassPath().size());
      std::vector<std::unique_ptr<const DexFile>> extra_boot_class_path;
      if (runtime_options.Exists(Opt::BootClassPathDexList)) {
        extra_boot_class_path.swap(*runtime_options.GetOrDefault(Opt::BootClassPathDexList));
      } else {
        ArrayRef<File> bcp_files = start < GetBootClassPathFiles().size() ?
                                       ArrayRef<File>(GetBootClassPathFiles()).SubArray(start) :
                                       ArrayRef<File>();
        OpenBootDexFiles(ArrayRef<const std::string>(GetBootClassPath()).SubArray(start),
                         ArrayRef<const std::string>(GetBootClassPathLocations()).SubArray(start),
                         bcp_files,
                         &extra_boot_class_path);
      }
      class_linker_->AddExtraBootDexFiles(self, std::move(extra_boot_class_path));
    }
    if (IsJavaDebuggable() || jit_options_->GetProfileSaverOptions().GetProfileBootClassPath()) {
      // Deoptimize the boot image if debuggable  as the code may have been compiled non-debuggable.
      // Also deoptimize if we are profiling the boot class path.
      ScopedThreadSuspension sts(self, ThreadState::kNative);
      ScopedSuspendAll ssa(__FUNCTION__);
      DeoptimizeBootImage();
    }
  } else {
    std::vector<std::unique_ptr<const DexFile>> boot_class_path;
    if (runtime_options.Exists(Opt::BootClassPathDexList)) {
      boot_class_path.swap(*runtime_options.GetOrDefault(Opt::BootClassPathDexList));
    } else {
      OpenBootDexFiles(ArrayRef<const std::string>(GetBootClassPath()),
                       ArrayRef<const std::string>(GetBootClassPathLocations()),
                       ArrayRef<File>(GetBootClassPathFiles()),
                       &boot_class_path);
    }
    if (!class_linker_->InitWithoutImage(std::move(boot_class_path), &error_msg)) {
      LOG(ERROR) << "Could not initialize without image: " << error_msg;
      return false;
    }

    // TODO: Should we move the following to InitWithoutImage?
    SetInstructionSet(instruction_set_);
    for (uint32_t i = 0; i < kCalleeSaveSize; i++) {
      CalleeSaveType type = CalleeSaveType(i);
      if (!HasCalleeSaveMethod(type)) {
        SetCalleeSaveMethod(CreateCalleeSaveMethod(), type);
      }
    }
  }

  // Now that the boot image space is set, cache the boot classpath checksums,
  // to be used when validating oat files.
  ArrayRef<gc::space::ImageSpace* const> image_spaces(GetHeap()->GetBootImageSpaces());
  ArrayRef<const DexFile* const> bcp_dex_files(GetClassLinker()->GetBootClassPath());
  boot_class_path_checksums_ = gc::space::ImageSpace::GetBootClassPathChecksums(image_spaces,
                                                                                bcp_dex_files);

  CHECK(class_linker_ != nullptr);

  if (runtime_options.Exists(Opt::MethodTrace)) {
    trace_config_.reset(new TraceConfig());
    trace_config_->trace_file = runtime_options.ReleaseOrDefault(Opt::MethodTraceFile);
    trace_config_->trace_file_size = runtime_options.ReleaseOrDefault(Opt::MethodTraceFileSize);
    trace_config_->trace_mode = Trace::TraceMode::kMethodTracing;
    trace_config_->trace_output_mode = runtime_options.Exists(Opt::MethodTraceStreaming) ?
                                           TraceOutputMode::kStreaming :
                                           TraceOutputMode::kFile;
    trace_config_->clock_source = runtime_options.GetOrDefault(Opt::MethodTraceClock);
  }

  if (GetHeap()->HasBootImageSpace()) {
    const ImageHeader& image_header = GetHeap()->GetBootImageSpaces()[0]->GetImageHeader();
    ObjPtr<mirror::ObjectArray<mirror::Object>> boot_image_live_objects =
        ObjPtr<mirror::ObjectArray<mirror::Object>>::DownCast(
            image_header.GetImageRoot(ImageHeader::kBootImageLiveObjects));
    pre_allocated_OutOfMemoryError_when_throwing_exception_ = GcRoot<mirror::Throwable>(
        boot_image_live_objects->Get(ImageHeader::kOomeWhenThrowingException)->AsThrowable());
    DCHECK(pre_allocated_OutOfMemoryError_when_throwing_exception_.Read()->GetClass()
               ->DescriptorEquals("Ljava/lang/OutOfMemoryError;"));
    pre_allocated_OutOfMemoryError_when_throwing_oome_ = GcRoot<mirror::Throwable>(
        boot_image_live_objects->Get(ImageHeader::kOomeWhenThrowingOome)->AsThrowable());
    DCHECK(pre_allocated_OutOfMemoryError_when_throwing_oome_.Read()->GetClass()
               ->DescriptorEquals("Ljava/lang/OutOfMemoryError;"));
    pre_allocated_OutOfMemoryError_when_handling_stack_overflow_ = GcRoot<mirror::Throwable>(
        boot_image_live_objects->Get(ImageHeader::kOomeWhenHandlingStackOverflow)->AsThrowable());
    DCHECK(pre_allocated_OutOfMemoryError_when_handling_stack_overflow_.Read()->GetClass()
               ->DescriptorEquals("Ljava/lang/OutOfMemoryError;"));
    pre_allocated_NoClassDefFoundError_ = GcRoot<mirror::Throwable>(
        boot_image_live_objects->Get(ImageHeader::kNoClassDefFoundError)->AsThrowable());
    DCHECK(pre_allocated_NoClassDefFoundError_.Read()->GetClass()
               ->DescriptorEquals("Ljava/lang/NoClassDefFoundError;"));
  } else {
    CreatePreAllocatedExceptions(self);
  }

  // Class-roots are setup, we can now finish initializing the JniIdManager.
  GetJniIdManager()->Init(self);

  // Initialize metrics only for the Zygote process or
  // if explicitly enabled via command line argument.
  if (IsZygote() || gFlags.MetricsForceEnable.GetValue()) {
    LOG(INFO) << "Initializing ART runtime metrics";
    InitMetrics();
  }

  // Runtime initialization is largely done now.
  // We load plugins first since that can modify the runtime state slightly.
  // Load all plugins
  {
    // The init method of plugins expect the state of the thread to be non runnable.
    ScopedThreadSuspension sts(self, ThreadState::kNative);
    for (auto& plugin : plugins_) {
      std::string err;
      if (!plugin.Load(&err)) {
        LOG(FATAL) << plugin << " failed to load: " << err;
      }
    }
  }

  // Look for a native bridge.
  //
  // The intended flow here is, in the case of a running system:
  //
  // Runtime::Init() (zygote):
  //   LoadNativeBridge -> dlopen from cmd line parameter.
  //  |
  //  V
  // Runtime::Start() (zygote):
  //   No-op wrt native bridge.
  //  |
  //  | start app
  //  V
  // DidForkFromZygote(action)
  //   action = kUnload -> dlclose native bridge.
  //   action = kInitialize -> initialize library
  //
  //
  // The intended flow here is, in the case of a simple dalvikvm call:
  //
  // Runtime::Init():
  //   LoadNativeBridge -> dlopen from cmd line parameter.
  //  |
  //  V
  // Runtime::Start():
  //   DidForkFromZygote(kInitialize) -> try to initialize any native bridge given.
  //   No-op wrt native bridge.
  {
    std::string native_bridge_file_name = runtime_options.ReleaseOrDefault(Opt::NativeBridge);
    is_native_bridge_loaded_ = LoadNativeBridge(native_bridge_file_name);
  }

  // Startup agents
  // TODO Maybe we should start a new thread to run these on. Investigate RI behavior more.
  for (auto& agent_spec : agent_specs_) {
    // TODO Check err
    int res = 0;
    std::string err = "";
    ti::LoadError error;
    std::unique_ptr<ti::Agent> agent = agent_spec.Load(&res, &error, &err);

    if (agent != nullptr) {
      agents_.push_back(std::move(agent));
      continue;
    }

    switch (error) {
      case ti::LoadError::kInitializationError:
        LOG(FATAL) << "Unable to initialize agent!";
        UNREACHABLE();

      case ti::LoadError::kLoadingError:
        LOG(ERROR) << "Unable to load an agent: " << err;
        continue;

      case ti::LoadError::kNoError:
        break;
    }
    LOG(FATAL) << "Unreachable";
    UNREACHABLE();
  }
  {
    ScopedObjectAccess soa(self);
    callbacks_->NextRuntimePhase(RuntimePhaseCallback::RuntimePhase::kInitialAgents);
  }

  if (IsZygote() && IsPerfettoHprofEnabled()) {
    constexpr const char* plugin_name = kIsDebugBuild ?
        "libperfetto_hprofd.so" : "libperfetto_hprof.so";
    // Load eagerly in Zygote to improve app startup times. This will make
    // subsequent dlopens for the library no-ops.
    dlopen(plugin_name, RTLD_NOW | RTLD_LOCAL);
  }

  VLOG(startup) << "Runtime::Init exiting";

  return true;
}
```
总结一下在Runtime::Create()的过程中，进行了以下事情：

RuntimeOption读取 → 内存映射Memmap::Init & 分配哨兵页 → OatFileManager, JniIDManager, Monitor等核心组件创建 → 创建Heap, 设置GC, 设置sigchain → 创建JavaVM数据结构, 主线程Attach → 类加载器 → 插件/Native Bridge

遇到任何关键错误（如 BootClassPath 为空、创建 JavaVM 失败、ClassLinker 初始化失败等）都会 LOG(ERROR) 并返回 false，否则最终返回 true。

其中涉及到的数据结构：
- 环境快照 env_snapshot_
- MemMap
- protected_fault_page_ 哨兵页
- QuasiAtomic 准原子操作
- oat_file_manager_
- jni_id_manager_
- Monitor
- intern_table_
- monitor_list_ 
- monitor_pool_
- thread_list_
- gc::Heap 
- fault_manager
- JavaVMExt/JNIEnv
- 各种插件agent



#### 创建Heap 

```cpp
Heap::Heap(size_t initial_size,
           size_t growth_limit,
           size_t min_free,
           size_t max_free,
           double target_utilization,
           double foreground_heap_growth_multiplier,
           size_t stop_for_native_allocs,
           size_t capacity,
           size_t non_moving_space_capacity,
           const std::vector<std::string>& boot_class_path,
           const std::vector<std::string>& boot_class_path_locations,
           ArrayRef<File> boot_class_path_files,
           ArrayRef<File> boot_class_path_image_files,
           ArrayRef<File> boot_class_path_vdex_files,
           ArrayRef<File> boot_class_path_oat_files,
           const std::vector<std::string>& image_file_names,
           const InstructionSet image_instruction_set,
           CollectorType foreground_collector_type,
           CollectorType background_collector_type,
           space::LargeObjectSpaceType large_object_space_type,
           size_t large_object_threshold,
           size_t parallel_gc_threads,
           size_t conc_gc_threads,
           bool low_memory_mode,
           size_t long_pause_log_threshold,
           size_t long_gc_log_threshold,
           bool ignore_target_footprint,
           bool always_log_explicit_gcs,
           bool use_tlab,
           bool verify_pre_gc_heap,
           bool verify_pre_sweeping_heap,
           bool verify_post_gc_heap,
           bool verify_pre_gc_rosalloc,
           bool verify_pre_sweeping_rosalloc,
           bool verify_post_gc_rosalloc,
           bool gc_stress_mode,
           bool measure_gc_performance,
           bool use_homogeneous_space_compaction_for_oom,
           bool use_generational_gc,
           uint64_t min_interval_homogeneous_space_compaction_by_oom,
           bool dump_region_info_before_gc,
           bool dump_region_info_after_gc)
    : non_moving_space_(nullptr),
      rosalloc_space_(nullptr),
      dlmalloc_space_(nullptr),
      main_space_(nullptr),
      collector_type_(kCollectorTypeNone),
      foreground_collector_type_(foreground_collector_type),
      background_collector_type_(background_collector_type),
      desired_collector_type_(foreground_collector_type_),
      pending_task_lock_(nullptr),
      parallel_gc_threads_(parallel_gc_threads),
      conc_gc_threads_(conc_gc_threads),
      low_memory_mode_(low_memory_mode),
      long_pause_log_threshold_(long_pause_log_threshold),
      long_gc_log_threshold_(long_gc_log_threshold),
      process_cpu_start_time_ns_(ProcessCpuNanoTime()),
      pre_gc_last_process_cpu_time_ns_(process_cpu_start_time_ns_),
      post_gc_last_process_cpu_time_ns_(process_cpu_start_time_ns_),
      pre_gc_weighted_allocated_bytes_(0.0),
      post_gc_weighted_allocated_bytes_(0.0),
      ignore_target_footprint_(ignore_target_footprint),
      always_log_explicit_gcs_(always_log_explicit_gcs),
      zygote_creation_lock_("zygote creation lock", kZygoteCreationLock),
      zygote_space_(nullptr),
      large_object_threshold_(large_object_threshold),
      disable_thread_flip_count_(0),
      thread_flip_running_(false),
      collector_type_running_(kCollectorTypeNone),
      last_gc_cause_(kGcCauseNone),
      thread_running_gc_(nullptr),
      last_gc_type_(collector::kGcTypeNone),
      next_gc_type_(collector::kGcTypePartial),
      capacity_(capacity),
      growth_limit_(growth_limit),
      initial_heap_size_(initial_size),
      target_footprint_(initial_size),
      // Using kPostMonitorLock as a lock at kDefaultMutexLevel is acquired after
      // this one.
      process_state_update_lock_("process state update lock", kPostMonitorLock),
      min_foreground_target_footprint_(0),
      min_foreground_concurrent_start_bytes_(0),
      concurrent_start_bytes_(std::numeric_limits<size_t>::max()),
      total_bytes_freed_ever_(0),
      total_objects_freed_ever_(0),
      num_bytes_allocated_(0),
      native_bytes_registered_(0),
      old_native_bytes_allocated_(0),
      native_objects_notified_(0),
      num_bytes_freed_revoke_(0),
      num_bytes_alive_after_gc_(0),
      verify_missing_card_marks_(false),
      verify_system_weaks_(false),
      verify_pre_gc_heap_(verify_pre_gc_heap),
      verify_pre_sweeping_heap_(verify_pre_sweeping_heap),
      verify_post_gc_heap_(verify_post_gc_heap),
      verify_mod_union_table_(false),
      verify_pre_gc_rosalloc_(verify_pre_gc_rosalloc),
      verify_pre_sweeping_rosalloc_(verify_pre_sweeping_rosalloc),
      verify_post_gc_rosalloc_(verify_post_gc_rosalloc),
      gc_stress_mode_(gc_stress_mode),
      /* For GC a lot mode, we limit the allocation stacks to be kGcAlotInterval allocations. This
       * causes a lot of GC since we do a GC for alloc whenever the stack is full. When heap
       * verification is enabled, we limit the size of allocation stacks to speed up their
       * searching.
       */
      max_allocation_stack_size_(kGCALotMode ? kGcAlotAllocationStackSize
                                 : (kVerifyObjectSupport > kVerifyObjectModeFast)
                                     ? kVerifyObjectAllocationStackSize
                                     : kDefaultAllocationStackSize),
      current_allocator_(kAllocatorTypeDlMalloc),
      current_non_moving_allocator_(kAllocatorTypeNonMoving),
      bump_pointer_space_(nullptr),
      temp_space_(nullptr),
      region_space_(nullptr),
      min_free_(min_free),
      max_free_(max_free),
      target_utilization_(target_utilization),
      foreground_heap_growth_multiplier_(foreground_heap_growth_multiplier),
      stop_for_native_allocs_(stop_for_native_allocs),
      total_wait_time_(0),
      verify_object_mode_(kVerifyObjectModeDisabled),
      disable_moving_gc_count_(0),
      semi_space_collector_(nullptr),
      active_concurrent_copying_collector_(nullptr),
      young_concurrent_copying_collector_(nullptr),
      concurrent_copying_collector_(nullptr),
      is_running_on_memory_tool_(Runtime::Current()->IsRunningOnMemoryTool()),
      use_tlab_(use_tlab),
      main_space_backup_(nullptr),
      min_interval_homogeneous_space_compaction_by_oom_(
          min_interval_homogeneous_space_compaction_by_oom),
      last_time_homogeneous_space_compaction_by_oom_(NanoTime()),
      gcs_completed_(0u),
      max_gc_requested_(0u),
      pending_collector_transition_(nullptr),
      pending_heap_trim_(nullptr),
      use_homogeneous_space_compaction_for_oom_(use_homogeneous_space_compaction_for_oom),
      use_generational_gc_(use_generational_gc),
      running_collection_is_blocking_(false),
      blocking_gc_count_(0U),
      blocking_gc_time_(0U),
      last_update_time_gc_count_rate_histograms_(  // Round down by the window duration.
          (NanoTime() / kGcCountRateHistogramWindowDuration) * kGcCountRateHistogramWindowDuration),
      gc_count_last_window_(0U),
      blocking_gc_count_last_window_(0U),
      gc_count_rate_histogram_("gc count rate histogram", 1U, kGcCountRateMaxBucketCount),
      blocking_gc_count_rate_histogram_(
          "blocking gc count rate histogram", 1U, kGcCountRateMaxBucketCount),
      alloc_tracking_enabled_(false),
      alloc_record_depth_(AllocRecordObjectMap::kDefaultAllocStackDepth),
      backtrace_lock_(nullptr),
      seen_backtrace_count_(0u),
      unique_backtrace_count_(0u),
      gc_disabled_for_shutdown_(false),
      dump_region_info_before_gc_(dump_region_info_before_gc),
      dump_region_info_after_gc_(dump_region_info_after_gc),
      boot_image_spaces_(),
      boot_images_start_address_(0u),
      boot_images_size_(0u),
      pre_oome_gc_count_(0u) {
  if (VLOG_IS_ON(heap) || VLOG_IS_ON(startup)) {
    LOG(INFO) << "Heap() entering";
  }

  // foreground_collector当前应用在前台（用户可见、交互中）时用哪个 GC 算法。
	// background_collector当应用被通知进入后台（不再和用户直接交互）时切换到哪个 GC 算法。

  // 检测各种配置是否能和GC算法对应上

  LOG(INFO) << "Using " << foreground_collector_type_ << " GC.";
  if (gUseUserfaultfd) {
    // userfaultfd 允许用户态捕获某块内存区域的「缺页」或「保护错误」事件，自己在用户态决定要怎么填页／拷贝／唤醒访问线程。
    CHECK_EQ(foreground_collector_type_, kCollectorTypeCMC);
    CHECK_EQ(background_collector_type_, kCollectorTypeCMCBackground);
  } else {
    // This ensures that userfaultfd syscall is done before any seccomp filter is installed.
    // TODO(b/266731037): Remove this when we no longer need to collect metric on userfaultfd
    // support.
    auto [uffd_supported, minor_fault_supported] = collector::MarkCompact::GetUffdAndMinorFault();
    // The check is just to ensure that compiler doesn't eliminate the function call above.
    // Userfaultfd support is certain to be there if its minor-fault feature is supported.
    CHECK_IMPLIES(minor_fault_supported, uffd_supported);
  }

  if (gUseReadBarrier) {
    CHECK_EQ(foreground_collector_type_, kCollectorTypeCC);
    CHECK_EQ(background_collector_type_, kCollectorTypeCCBackground);
  } else if (background_collector_type_ != gc::kCollectorTypeHomogeneousSpaceCompact) {
    CHECK_EQ(IsMovingGc(foreground_collector_type_), IsMovingGc(background_collector_type_))
        << "Changing from " << foreground_collector_type_ << " to "
        << background_collector_type_ << " (or visa versa) is not supported.";
  }
  verification_.reset(new Verification(this));
  CHECK_GE(large_object_threshold, kMinLargeObjectThreshold);
  ScopedTrace trace(__FUNCTION__);
  Runtime* const runtime = Runtime::Current();
  // If we aren't the zygote, switch to the default non zygote allocator. This may update the
  // entrypoints.
  const bool is_zygote = runtime->IsZygote();
  if (!is_zygote) {
    // Background compaction is currently not supported for command line runs.
    if (background_collector_type_ != foreground_collector_type_) {
      VLOG(heap) << "Disabling background compaction for non zygote";
      background_collector_type_ = foreground_collector_type_;
    }
  }
  ChangeCollector(desired_collector_type_);

  // 自上次垃圾回收（GC）周期以来被认为仍然存活对象所对应的位图。
  live_bitmap_.reset(new accounting::HeapBitmap(this));
  // 在当前垃圾回收周期中已标记对象所对应的位图。
  mark_bitmap_.reset(new accounting::HeapBitmap(this));

  // 在压缩回收时，不区分老年代／新生代／非移动区，所有 region 都用同一种方式对待：扫描出所有可达对象，然后把它们搬到一端或连续几个 region 里，把剩下的 region 释放或留作下一次分配。这就保证了 heap 中存活对象布局是“homogeneous”的、连续的，彻底消除内部碎片。
  // We don't have hspace compaction enabled with CC.
  // CC 和 CMC 本身就是基于 region 的压缩式收集器——它们在后台就会把存活对象搬到新的 regions 上，自动消除碎片
  if (foreground_collector_type_ == kCollectorTypeCC
      || foreground_collector_type_ == kCollectorTypeCMC) {
    use_homogeneous_space_compaction_for_oom_ = false;
  }
  bool support_homogeneous_space_compaction =
      background_collector_type_ == gc::kCollectorTypeHomogeneousSpaceCompact ||
      use_homogeneous_space_compaction_for_oom_;
  // We may use the same space the main space for the non moving space if we don't need to compact
  // from the main space.
  // This is not the case if we support homogeneous compaction or have a moving background
  // collector type.
  bool separate_non_moving_space = is_zygote ||
      support_homogeneous_space_compaction || IsMovingGc(foreground_collector_type_) ||
      IsMovingGc(background_collector_type_);

  // Requested begin for the alloc space, to follow the mapped image and oat files
  uint8_t* request_begin = nullptr;
  // Calculate the extra space required after the boot image, see allocations below.
  size_t heap_reservation_size = 0u;
  if (separate_non_moving_space) {
    heap_reservation_size = non_moving_space_capacity;
  } else if (foreground_collector_type_ != kCollectorTypeCC && is_zygote) {
    heap_reservation_size = capacity_;
  }
  heap_reservation_size = RoundUp(heap_reservation_size, gPageSize);
  // Load image space(s).
  // 加载BootImage
  std::vector<std::unique_ptr<space::ImageSpace>> boot_image_spaces;
  MemMap heap_reservation;
  if (space::ImageSpace::LoadBootImage(boot_class_path,
                                       boot_class_path_locations,
                                       boot_class_path_files,
                                       boot_class_path_image_files,
                                       boot_class_path_vdex_files,
                                       boot_class_path_oat_files,
                                       image_file_names,
                                       image_instruction_set,
                                       runtime->ShouldRelocate(),
                                       /*executable=*/!runtime->IsAotCompiler(),
                                       heap_reservation_size,
                                       runtime->AllowInMemoryCompilation(),
                                       runtime->GetApexVersions(),
                                       &boot_image_spaces,
                                       &heap_reservation)) {
    DCHECK_EQ(heap_reservation_size, heap_reservation.IsValid() ? heap_reservation.Size() : 0u);
    DCHECK(!boot_image_spaces.empty());
    request_begin = boot_image_spaces.back()->GetImageHeader().GetOatFileEnd();
    DCHECK_IMPLIES(heap_reservation.IsValid(), request_begin == heap_reservation.Begin())
        << "request_begin=" << static_cast<const void*>(request_begin)
        << " heap_reservation.Begin()=" << static_cast<const void*>(heap_reservation.Begin());
    for (std::unique_ptr<space::ImageSpace>& space : boot_image_spaces) {
      boot_image_spaces_.push_back(space.get());
      AddSpace(space.release());
    }
    boot_images_start_address_ = PointerToLowMemUInt32(boot_image_spaces_.front()->Begin());
    uint32_t boot_images_end =
        PointerToLowMemUInt32(boot_image_spaces_.back()->GetImageHeader().GetOatFileEnd());
    boot_images_size_ = boot_images_end - boot_images_start_address_;
    if (kIsDebugBuild) {
      VerifyBootImagesContiguity(boot_image_spaces_);
    }
  } else {
    if (foreground_collector_type_ == kCollectorTypeCC) {
      // Need to use a low address so that we can allocate a contiguous 2 * Xmx space
      // when there's no image (dex2oat for target).
      request_begin = kPreferredAllocSpaceBegin;
    }
    // Gross hack to make dex2oat deterministic.
    if (foreground_collector_type_ == kCollectorTypeMS && Runtime::Current()->IsAotCompiler()) {
      // Currently only enabled for MS collector since that is what the deterministic dex2oat uses.
      // b/26849108
      request_begin = reinterpret_cast<uint8_t*>(kAllocSpaceBeginForDeterministicAoT);
    }
  }

  /*
  requested_alloc_space_begin ->     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-
                                     +-  nonmoving space (non_moving_space_capacity)+-
                                     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-
                                     +-????????????????????????????????????????????+-
                                     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-
                                     +-main alloc space / bump space 1 (capacity_) +-
                                     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-
                                     +-????????????????????????????????????????????+-
                                     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-
                                     +-main alloc space2 / bump space 2 (capacity_)+-
                                     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-
  */

  MemMap main_mem_map_1;
  MemMap main_mem_map_2;

  std::string error_str;
  MemMap non_moving_space_mem_map;
  if (separate_non_moving_space) {
    ScopedTrace trace2("Create separate non moving space");
    // If we are the zygote, the non moving space becomes the zygote space when we run
    // PreZygoteFork the first time. In this case, call the map "zygote space" since we can't
    // rename the mem map later.
    const char* space_name = is_zygote ? kZygoteSpaceName : kNonMovingSpaceName;
    // Reserve the non moving mem map before the other two since it needs to be at a specific
    // address.
    DCHECK_EQ(heap_reservation.IsValid(), !boot_image_spaces_.empty());
    if (heap_reservation.IsValid()) {
      non_moving_space_mem_map = heap_reservation.RemapAtEnd(
          heap_reservation.Begin(), space_name, PROT_READ | PROT_WRITE, &error_str);
    } else {
      non_moving_space_mem_map = MapAnonymousPreferredAddress(
          space_name, request_begin, non_moving_space_capacity, &error_str);
    }
    CHECK(non_moving_space_mem_map.IsValid()) << error_str;
    DCHECK(!heap_reservation.IsValid());
    // Try to reserve virtual memory at a lower address if we have a separate non moving space.
    request_begin = non_moving_space_mem_map.Begin() == kPreferredAllocSpaceBegin
                        ? non_moving_space_mem_map.End()
                        : kPreferredAllocSpaceBegin;
  }
  // Attempt to create 2 mem maps at or after the requested begin.
  if (foreground_collector_type_ != kCollectorTypeCC) {
    ScopedTrace trace2("Create main mem map");
    if (separate_non_moving_space || !is_zygote) {
      main_mem_map_1 = MapAnonymousPreferredAddress(
          kMemMapSpaceName[0], request_begin, capacity_, &error_str);
    } else {
      // If no separate non-moving space and we are the zygote, the main space must come right after
      // the image space to avoid a gap. This is required since we want the zygote space to be
      // adjacent to the image space.
      DCHECK_EQ(heap_reservation.IsValid(), !boot_image_spaces_.empty());
      main_mem_map_1 = MemMap::MapAnonymous(
          kMemMapSpaceName[0],
          request_begin,
          capacity_,
          PROT_READ | PROT_WRITE,
          /* low_4gb= */ true,
          /* reuse= */ false,
          heap_reservation.IsValid() ? &heap_reservation : nullptr,
          &error_str);
    }
    CHECK(main_mem_map_1.IsValid()) << error_str;
    DCHECK(!heap_reservation.IsValid());
  }
  if (support_homogeneous_space_compaction ||
      background_collector_type_ == kCollectorTypeSS ||
      foreground_collector_type_ == kCollectorTypeSS) {
    ScopedTrace trace2("Create main mem map 2");
    main_mem_map_2 = MapAnonymousPreferredAddress(
        kMemMapSpaceName[1], main_mem_map_1.End(), capacity_, &error_str);
    CHECK(main_mem_map_2.IsValid()) << error_str;
  }

  // Create the non moving space first so that bitmaps don't take up the address range.
  if (separate_non_moving_space) {
    ScopedTrace trace2("Add non moving space");
    // Non moving space is always dlmalloc since we currently don't have support for multiple
    // active rosalloc spaces.
    const size_t size = non_moving_space_mem_map.Size();
    const void* non_moving_space_mem_map_begin = non_moving_space_mem_map.Begin();
    non_moving_space_ = space::DlMallocSpace::CreateFromMemMap(std::move(non_moving_space_mem_map),
                                                               "zygote / non moving space",
                                                               GetDefaultStartingSize(),
                                                               initial_size,
                                                               size,
                                                               size,
                                                               /* can_move_objects= */ false);
    CHECK(non_moving_space_ != nullptr) << "Failed creating non moving space "
        << non_moving_space_mem_map_begin;
    non_moving_space_->SetFootprintLimit(non_moving_space_->Capacity());
    AddSpace(non_moving_space_);
  }
  // Create other spaces based on whether or not we have a moving GC.
  if (foreground_collector_type_ == kCollectorTypeCC) {
    CHECK(separate_non_moving_space);
    // Reserve twice the capacity, to allow evacuating every region for explicit GCs.
    MemMap region_space_mem_map =
        space::RegionSpace::CreateMemMap(kRegionSpaceName, capacity_ * 2, request_begin);
    CHECK(region_space_mem_map.IsValid()) << "No region space mem map";
    region_space_ = space::RegionSpace::Create(
        kRegionSpaceName, std::move(region_space_mem_map), use_generational_gc_);
    AddSpace(region_space_);
  } else if (IsMovingGc(foreground_collector_type_)) {
    // Create bump pointer spaces.
    // We only to create the bump pointer if the foreground collector is a compacting GC.
    // TODO: Place bump-pointer spaces somewhere to minimize size of card table.
    bump_pointer_space_ = space::BumpPointerSpace::CreateFromMemMap("Bump pointer space 1",
                                                                    std::move(main_mem_map_1));
    CHECK(bump_pointer_space_ != nullptr) << "Failed to create bump pointer space";
    AddSpace(bump_pointer_space_);
    // For Concurrent Mark-compact GC we don't need the temp space to be in
    // lower 4GB. So its temp space will be created by the GC itself.
    if (foreground_collector_type_ != kCollectorTypeCMC) {
      temp_space_ = space::BumpPointerSpace::CreateFromMemMap("Bump pointer space 2",
                                                              std::move(main_mem_map_2));
      CHECK(temp_space_ != nullptr) << "Failed to create bump pointer space";
      AddSpace(temp_space_);
    }
    CHECK(separate_non_moving_space);
  } else {
    CreateMainMallocSpace(std::move(main_mem_map_1), initial_size, growth_limit_, capacity_);
    CHECK(main_space_ != nullptr);
    AddSpace(main_space_);
    if (!separate_non_moving_space) {
      non_moving_space_ = main_space_;
      CHECK(!non_moving_space_->CanMoveObjects());
    }
    if (main_mem_map_2.IsValid()) {
      const char* name = kUseRosAlloc ? kRosAllocSpaceName[1] : kDlMallocSpaceName[1];
      main_space_backup_.reset(CreateMallocSpaceFromMemMap(std::move(main_mem_map_2),
                                                           initial_size,
                                                           growth_limit_,
                                                           capacity_,
                                                           name,
                                                           /* can_move_objects= */ true));
      CHECK(main_space_backup_.get() != nullptr);
      // Add the space so its accounted for in the heap_begin and heap_end.
      AddSpace(main_space_backup_.get());
    }
  }
  CHECK(non_moving_space_ != nullptr);
  CHECK(!non_moving_space_->CanMoveObjects());
  // Allocate the large object space.
  if (large_object_space_type == space::LargeObjectSpaceType::kFreeList) {
    large_object_space_ = space::FreeListSpace::Create("free list large object space", capacity_);
    CHECK(large_object_space_ != nullptr) << "Failed to create large object space";
  } else if (large_object_space_type == space::LargeObjectSpaceType::kMap) {
    large_object_space_ = space::LargeObjectMapSpace::Create("mem map large object space");
    CHECK(large_object_space_ != nullptr) << "Failed to create large object space";
  } else {
    // Disable the large object space by making the cutoff excessively large.
    large_object_threshold_ = std::numeric_limits<size_t>::max();
    large_object_space_ = nullptr;
  }
  if (large_object_space_ != nullptr) {
    AddSpace(large_object_space_);
  }
  // Compute heap capacity. Continuous spaces are sorted in order of Begin().
  CHECK(!continuous_spaces_.empty());
  // Relies on the spaces being sorted.
  uint8_t* heap_begin = continuous_spaces_.front()->Begin();
  uint8_t* heap_end = continuous_spaces_.back()->Limit();
  size_t heap_capacity = heap_end - heap_begin;
  // Remove the main backup space since it slows down the GC to have unused extra spaces.
  // TODO: Avoid needing to do this.
  if (main_space_backup_.get() != nullptr) {
    RemoveSpace(main_space_backup_.get());
  }
  // Allocate the card table.
  // We currently don't support dynamically resizing the card table.
  // Since we don't know where in the low_4gb the app image will be located, make the card table
  // cover the whole low_4gb. TODO: Extend the card table in AddSpace.
  UNUSED(heap_capacity);
  // Start at 4 KB, we can be sure there are no spaces mapped this low since the address range is
  // reserved by the kernel.
  static constexpr size_t kMinHeapAddress = 4 * KB;
  card_table_.reset(accounting::CardTable::Create(reinterpret_cast<uint8_t*>(kMinHeapAddress),
                                                  4 * GB - kMinHeapAddress));
  CHECK(card_table_.get() != nullptr) << "Failed to create card table";
  if (foreground_collector_type_ == kCollectorTypeCC && kUseTableLookupReadBarrier) {
    rb_table_.reset(new accounting::ReadBarrierTable());
    DCHECK(rb_table_->IsAllCleared());
  }
  if (HasBootImageSpace()) {
    // Don't add the image mod union table if we are running without an image, this can crash if
    // we use the CardCache implementation.
    for (space::ImageSpace* image_space : GetBootImageSpaces()) {
      accounting::ModUnionTable* mod_union_table = new accounting::ModUnionTableToZygoteAllocspace(
          "Image mod-union table", this, image_space);
      CHECK(mod_union_table != nullptr) << "Failed to create image mod-union table";
      AddModUnionTable(mod_union_table);
    }
  }
  if (collector::SemiSpace::kUseRememberedSet && non_moving_space_ != main_space_) {
    accounting::RememberedSet* non_moving_space_rem_set =
        new accounting::RememberedSet("Non-moving space remembered set", this, non_moving_space_);
    CHECK(non_moving_space_rem_set != nullptr) << "Failed to create non-moving space remembered set";
    AddRememberedSet(non_moving_space_rem_set);
  }
  // TODO: Count objects in the image space here?
  num_bytes_allocated_.store(0, std::memory_order_relaxed);
  mark_stack_.reset(accounting::ObjectStack::Create("mark stack", kDefaultMarkStackSize,
                                                    kDefaultMarkStackSize));
  const size_t alloc_stack_capacity = max_allocation_stack_size_ + kAllocationStackReserveSize;
  allocation_stack_.reset(accounting::ObjectStack::Create(
      "allocation stack", max_allocation_stack_size_, alloc_stack_capacity));
  live_stack_.reset(accounting::ObjectStack::Create(
      "live stack", max_allocation_stack_size_, alloc_stack_capacity));
  // It's still too early to take a lock because there are no threads yet, but we can create locks
  // now. We don't create it earlier to make it clear that you can't use locks during heap
  // initialization.
  gc_complete_lock_ = new Mutex("GC complete lock");
  gc_complete_cond_.reset(new ConditionVariable("GC complete condition variable",
                                                *gc_complete_lock_));

  thread_flip_lock_ = new Mutex("GC thread flip lock");
  thread_flip_cond_.reset(new ConditionVariable("GC thread flip condition variable",
                                                *thread_flip_lock_));
  task_processor_.reset(new TaskProcessor());
  reference_processor_.reset(new ReferenceProcessor());
  pending_task_lock_ = new Mutex("Pending task lock");
  if (ignore_target_footprint_) {
    SetIdealFootprint(std::numeric_limits<size_t>::max());
    concurrent_start_bytes_ = std::numeric_limits<size_t>::max();
  }
  CHECK_NE(target_footprint_.load(std::memory_order_relaxed), 0U);
  CreateGarbageCollectors(measure_gc_performance);
  if (!GetBootImageSpaces().empty() && non_moving_space_ != nullptr &&
      (is_zygote || separate_non_moving_space)) {
    // Check that there's no gap between the image space and the non moving space so that the
    // immune region won't break (eg. due to a large object allocated in the gap). This is only
    // required when we're the zygote.
    // Space with smallest Begin().
    space::ImageSpace* first_space = nullptr;
    for (space::ImageSpace* space : boot_image_spaces_) {
      if (first_space == nullptr || space->Begin() < first_space->Begin()) {
        first_space = space;
      }
    }
    bool no_gap = MemMap::CheckNoGaps(*first_space->GetMemMap(), *non_moving_space_->GetMemMap());
    if (!no_gap) {
      PrintFileToLog("/proc/self/maps", LogSeverity::ERROR);
      MemMap::DumpMaps(LOG_STREAM(ERROR), /* terse= */ true);
      LOG(FATAL) << "There's a gap between the image space and the non-moving space";
    }
  }
  // Perfetto Java Heap Profiler Support.
  if (runtime->IsPerfettoJavaHeapStackProfEnabled()) {
    // Perfetto Plugin is loaded and enabled, initialize the Java Heap Profiler.
    InitPerfettoJavaHeapProf();
  } else {
    // Disable the Java Heap Profiler.
    GetHeapSampler().DisableHeapSampler();
  }

  instrumentation::Instrumentation* const instrumentation = runtime->GetInstrumentation();
  if (gc_stress_mode_) {
    backtrace_lock_ = new Mutex("GC complete lock");
  }
  if (is_running_on_memory_tool_ || gc_stress_mode_) {
    instrumentation->InstrumentQuickAllocEntryPoints();
  }
  if (VLOG_IS_ON(heap) || VLOG_IS_ON(startup)) {
    LOG(INFO) << "Heap() exiting";
  }
}
```

Heap构造函数初始化堆的主要步骤：

1. 初始化成员变量
  - 各种配置参数（初始大小、增长上限、并发 GC 线程数、是否开启 TLAB、OOM 时是否做同质空间压缩…）直接保存在对应字段里。
  - 前台/后台 GC 算法类型分别保存在 foreground_collector_type_、background_collector_type_，并且初始时把“下一次”要用的收集器设为前台那种。
	- 统计数据（CPU 时间、已分配字节、已回收字节、GC 次数等）清零或初始化。
2. 校验和环境准备
	- 根据 gUseUserfaultfd（是否启用 userfaultfd 驱动 pagefault）检查只能用 CMC/CMCBackground，或者查询内核是否支持。
	- 如果启用了读屏障（Read Barrier），则要使用 CC/CCBackground；否则必须保证前后台两种收集器都是“移动”或都是“非移动”的，不可混用。
	-	生成一个 Verification 对象（仅在 debug 或验证模式下跑完整一致性检查）。
3. 选择当前收集器
	- 非 Zygote 进程若后台算法与前台不同，强制后台也用前台同样的算法。
	- 调用 ChangeCollector(desired_collector_type_)，安装初始的 GC stub（entrypoint）和收集器状态。
4. 创建位图
	- live_bitmap_：回收前“存活”对象集合的位图。
	- mark_bitmap_：当前 GC 标记阶段“已标记”对象集合的位图。
5. 处理同质空间压缩（HSpace Compact）
	- 对于 CC/CMC 本身就是 region‐based 压缩式收集器，不要再额外在 OOM 时做 homogeneous space compaction，于是禁止该选项。
	- 判断是否需要一个独立的“非移动区”（non-moving space）：
	  -	Zygote 进程总要独立；
	  -	如果后台算法是 HomogeneousSpaceCompact，或前后台任一是“移动 GC”，也要独立；否则可把主区当成非移动区共用。
6. 加载 BootImage
	-	调用 ImageSpace::LoadBootImage(...)，映射并加入所有包含 AOT 代码的 image 空间。
	-	记下它们的起止地址并设定好后续堆内存的“请求起点” (request_begin)。
7. 在低地址区域预留非移动区和主分配区
	- 如果要独立非移动区，就先用 MapAnonymousPreferredAddress（或 remap）映射一块大小为 non_moving_space_capacity 的地址。
	- 然后从 request_begin 开始，映射一到两个连续的主分配区内存区（根据是否要两段并行 bump-pointer 或 region），共计 capacity_ 大小。
8. 创建各个分区空间 (Spaces)
	-	非移动区：用 DlMallocSpace（或 RosAlloc），标记为不可移动。主要是Classes, ArtMethods, ArtFields, and non moving objects.
	- 主分配区：
	  -	如果前台是 CC（region GC），用 RegionSpace；
	  - 否则如果是移动 GC（MS/CMC），用两个 BumpPointerSpace；
	  -	否则（Non-moving 或 semi-space GC），用 malloc‐based 空间（dlmalloc/rosalloc）。
	-	大对象区：根据配置选用 free-list 或 mmap‐backed。
9. 设置堆边界与CardTable
	-	把所有连续空间（image + non-moving + main + backup + large）按地址排序，算出 heap_begin/heap_end，然后创建整个 4 GB 低内存范围的写屏障CardTable。写屏障打脏卡时记录哪一块内存页（card）可能包含跨代／跨区引用。

	-	当启用 Baker-style 读屏障时，用一张rb_table_来快速判断某个对象是否已经被“访问过”或“复制过”。
10.	初始化 Root/Mod-Union/RememberedSet
  - 对于包含 ImageSpace 的，把各个 image 空间加上一个 Mod-Union table；记录它内部所有指向其他 Space（如非移动区）的引用。

  - 如果 semi-space GC 且有专门的非移动区，为它建一个 RememberedSet。
  对于半空间复制式 GC（semi-space collector）或者分代 GC，ART 通常会把“新分配的 bump-pointer 区”（young generation）和一个“非移动区”（常驻区）分开管理。
      - 非移动区存放老对象（或者 zygote 预载对象），不会在普通 GC 里被 compact。
      - 但新生代里对象里的引用，可能会指向非移动区，或者反过来。

  - 为了 GC 不必全堆扫描就能找到这些跨空间的引用，ART 为「非移动区」维护了一个 Remembered Set。对每个老年代（或 region）Space 维护一张集合，精确记录哪些具体对象字段里存了指向新生代（或其它目标区）的引用。在分代 GC／Region GC 时只扫描这些记录过的引用，而不用遍历整个老年代。

11.	准备标记和分配栈
  用于 GC 标记／并发分配阶段。
	- mark_stack_: 标记—清扫或压缩 GC 时，把“已发现但字段还没扫”的对象推上来，用来做递归／迭代扫描。
  - allocation_stack_: 在并行／并发 GC（特别是并发压缩）里，新分配对象也要被视作“活对象”并在 GC 结束前扫描到，所以每次分配都会把对象推到这张栈上。
  - live_stack_: 和allocation_stack_交换内容，gc扫描live_stack_，避免影响线程分配对象。

12.	其它启动工作
	-	创建各种 GC 完成锁、条件变量、后台任务处理器、ReferenceProcessor；
	-	如果忽略目标占用（ignore_target_footprint_），把理想占用调到极大；
	-	最后调用 CreateGarbageCollectors(...) 真正实例化各类 GC 对象；
	-	如果是 Perfetto heap profiler，也在这里初始化；
	-	如果是内存分析工具模式或 GC 压力模式，给 allocation entrypoints 安装 instrumentation。

整个流程下来，ART 就把 BootImage、非移动区、主分配区、大对象区、屏障表、位图、GC 对象和所有数据结构都配置好了，堆就能对外提供分配和触发垃圾回收服务了。

总结一下：
在 Heap::Heap(…) 构造函数里，除了各类 Space（分代区／非移动区／大对象区）本身，还会创建和初始化一系列配合 GC 运行的辅助数据结构，主要包括：
1. Bitmap
	-	live_bitmap_：记录上次 GC 后「被认为仍存活」对象的位置。
	-	mark_bitmap_：当前 GC 标记阶段「已标记」对象的位置。
2.	屏障表（Write & Read Barriers）
	-	card_table_：对整个低 4 GB 地址空间的写屏障卡表，用于追踪跨区引用。
	-	rb_table_（可选，CC + table-lookup 模式）：读屏障辅助表。
3. 空间列表
	-	boot_image_spaces_：所有映射的 AOT image 区，用来存放引导类与它们的编译代码。
	-	non_moving_space_：专门的「不可移动区」（Zygote / 非移动区）。
	-	main_space_（或 bump-pointer / region / malloc 空间）：用于大多数对象的分配。
	-	large_object_space_：大对象区（free-list 或 mmap backing）。
4. 栈／队列结构
	-	mark_stack_：标记期的对象工作栈，用来做递归扫描。
	-	allocation_stack_、live_stack_：并发 GC 时用于分配／存活对象的临时栈。
5. Mod-Union / RememberedSet
	-	对每个 ImageSpace 建立一个 Mod-Union table，用于增量／并发标记阶段跟踪跨区指针。
	-	如果 semi-space GC 且有独立非移动区，则为其创建一个 RememberedSet。
6. 线程同步／调度
	-	gc_complete_lock_ + gc_complete_cond_：阻塞式 GC 完成通知。
	-	thread_flip_lock_ + thread_flip_cond_：并发复制GC中切换「谁在跑」时的同步。
	-	pending_task_lock_：保护异步挂起任务队列。
	-	task_processor_、reference_processor_：后台 GC 任务与 ReferenceQueue 处理器。
7. 统计与调试
	-	各种计数器／直方图（GC 次数、暂停时长、CPU 时间、分配量等）。
	-	verification_：（debug 下）堆结构一致性验证工具。
	-	Perfetto Java Heap Profiler（可选）或者 HeapSampler。

这些结构配合起来，才能让 ART 在不同 GC 算法之间切换、支持并发／并行回收、屏障同步、增量标记，以及在各类内存异常时做完善的诊断和恢复。



#### Runtime::start()

Runtime类就是ART java虚拟机包装了运行时信息的数据结构，包括堆。Runtime::start()包含了ART初始化运行时的流程。


```cpp

bool Runtime::Start() {
  // Restore main thread state to kNative as expected by native code.
  Thread* self = Thread::Current();
  started_ = true;
  // Before running any clinit, set up the native methods provided by the runtime itself.
  // 注册 Runtime 自带的 native 方法：把 ART/C++ 端提供的 native 接口（如 JNI_OnLoad、信号处理、内存分配监控等）注册到对应的 Java 类上。
  RegisterRuntimeNativeMethods(self->GetJniEnv());

  
  class_linker_->RunEarlyRootClinits(self);
```
ClassLinker::RunEarlyRootClinits 在 ART 初始化阶段负责提前把几个“根”Java 类的静态初始化（<clinit>）跑一遍，确保后面运行时这些核心类已经就绪，避免出现“类还没初始化就被用到”的问题。

```cpp
void ClassLinker::RunEarlyRootClinits(Thread* self) {
  StackHandleScope<1u> hs(self);
  Handle<mirror::ObjectArray<mirror::Class>> class_roots = hs.NewHandle(GetClassRoots());
  EnsureRootInitialized(this, self, GetClassRoot<mirror::Class>(class_roots.Get()));
  EnsureRootInitialized(this, self, GetClassRoot<mirror::String>(class_roots.Get()));
  // `Field` class is needed for register_java_net_InetAddress in libcore, b/28153851.
  EnsureRootInitialized(this, self, GetClassRoot<mirror::Field>(class_roots.Get()));

  WellKnownClasses::Init(self->GetJniEnv());

  // `FinalizerReference` class is needed for initialization of `java.net.InetAddress`.
  // (Indirectly by constructing a `ObjectStreamField` which uses a `StringBuilder`
  // and, when resizing, initializes the `System` class for `System.arraycopy()`
  // and `System.<clinit> creates a finalizable object.)
  EnsureRootInitialized(
      this, self, WellKnownClasses::java_lang_ref_FinalizerReference_add->GetDeclaringClass());
}
```
继续runtime::start()，接下来会加载一些Intrinsic方法，加载其他会用到的so库，初始化2个runtime中ThreadGroup类型的成员 main_thread_group_ 和system_thread_group_。

```cpp
  InitializeIntrinsics(); //把一大批“内置加速”（intrinsic）方法绑定到对应的 Java 实现上，以便在 JIT/AOT 时能够生成更高效的本地机器码。

  self->TransitionFromRunnableToSuspended(ThreadState::kNative); // 切换Java线程状态为Native

  // InitNativeMethods needs to be after started_ so that the classes
  // it touches will have methods linked to the oat file if necessary.
  {
    ScopedTrace trace2("InitNativeMethods");
    InitNativeMethods();
    // 把几个核心的 JNI 本地库（.so）“手动”加载进来，libicu_jni.so，libjavacore.so，libopenjdk.so
  }

  // InitializeCorePlatformApiPrivateFields() needs to be called after well known class
  // initializtion in InitNativeMethods().
  art::hiddenapi::InitializeCorePlatformApiPrivateFields();

  // Initialize well known thread group values that may be accessed threads while attaching.
  // 
  InitThreadGroups(self);

  Thread::FinishStartup();
```

在FinishStartup函数中，把当前主线程从“裸的 native 线程”提升为一个完整的 Java 线程，就是说在 Java 世界中创建并绑定一个 Thread 类型的对象，运行所有剩余的核心类初始化，将本线程登记到主线程组，以便后续所有基于 ThreadGroup 的管理（如未捕获异常处理、线程统计、按组 interrupt 等）都能正常工作。

继续runtime::start()
```cpp
  // Create the JIT either if we have to use JIT compilation or save profiling info. This is
  // done after FinishStartup as the JIT pool needs Java thread peers, which require the main
  // ThreadGroup to exist.
  //
  // TODO(calin): We use the JIT class as a proxy for JIT compilation and for
  // recoding profiles. Maybe we should consider changing the name to be more clear it's
  // not only about compiling. b/28295073.
  if (jit_options_->UseJitCompilation() || jit_options_->GetSaveProfilingInfo()) {
    CreateJit();
    // 这里会创建并启动JIT相关的线程池
  }

  // Send the start phase event. We have to wait till here as this is when the main thread peer
  // has just been generated, important root clinits have been run and JNI is completely functional.
  {
    ScopedObjectAccess soa(self);
    callbacks_->NextRuntimePhase(RuntimePhaseCallback::RuntimePhase::kStart);
  }

// 在 ART 运行时初始化完成后，从 Java 层获取并缓存 “系统类加载器”（SystemClassLoader），然后把它设置成当前主线程的上下文类加载器（contextClassLoader）
  system_class_loader_ = CreateSystemClassLoader(this);

```


**什么是类加载？**

“类加载”（Class Loading）是各种JVM将磁盘上的字节码（.class 或 .dex 文件）变成运行时存在于内存之中可操作的 Class 对象的整个过程。

**父委托模型**
Java 的标准类加载机制是 父委托模型：
- 当一个 ClassLoader 收到加载请求时，先委托其父加载器加载；
- 只有父加载器无法加载时，才自己去查找类文件。

这保证了核心 Java/Android 类只能由引导加载器或系统加载器来加载，避免重复或隔离加载。

java 提供了很多服务提供者接口（Service Provider Interface，SPI），允许第三方为这些接口提供实现。常见的 SPI 有 JDBC、JCE、JNDI、JAXP 和 JBI 等。

这些 SPI 的接口由 Java 核心库来提供，而这些 SPI 的实现代码则是作为 Java 应用所依赖的 jar 包被包含进类路径（CLASSPATH）里。

SPI接口中的代码经常需要加载具体的实现类。 并且我们知道一个类由类加载器A加载,那么这个类依赖类也应该由相同的类加载器加载.那么问题来了，引导类加载器无法找到 SPI 的实现类的，因为它只加载 Java 的核心库。它也不能代理给系统类加载器，因为它是系统类加载器的祖先类加载器。
此时就用 contextClassLoader 来打破父委托，让代码按线程绑定的加载器来加载特定类。


继续runtime::start()
```cpp
  if (!is_zygote_) {
    if (is_native_bridge_loaded_) {
      PreInitializeNativeBridge(".");
    }
    NativeBridgeAction action = force_native_bridge_
        ? NativeBridgeAction::kInitialize
        : NativeBridgeAction::kUnload;
    InitNonZygoteOrPostFork(self->GetJniEnv(),
                            /* is_system_server= */ false,
                            /* is_child_zygote= */ false,
                            action,
                            GetInstructionSetString(kRuntimeISA));
  }

 ```
 如果加载了 Native Bridge，则预初始化它。调用 InitNonZygoteOrPostFork() 完成类路径注册、Native Bridge 卸载或初始化等。

 “Native Bridge” 是 Android Runtime（ART）里用来让应用加载和执行为不同 CPU 架构编译的本地库（.so 文件）的一套兼容层机制。简单来说，它充当了这样一个“翻译器”：

- 在 x86 安卓设备上，绝大多数第三方应用或系统组件包含的是为 ARM 架构编译的本地代码。因为 x86 CPU 不能原生执行 ARM 指令，就需要一个Native Bridge（也称作“翻译库”或“兼容层”，典型例子是 Intel 的 libhoudini）来做动态二进制翻译，把 ARM 指令实时转换成 x86 指令并执行。

继续runtime::start()
 ```cpp
  {
    ScopedObjectAccess soa(self);
    StartDaemonThreads();
    self->GetJniEnv()->AssertLocalsEmpty();

    // Send the initialized phase event. Send it after starting the Daemon threads so that agents
    // cannot delay the daemon threads from starting forever.
    callbacks_->NextRuntimePhase(RuntimePhaseCallback::RuntimePhase::kInit);
    self->GetJniEnv()->AssertLocalsEmpty();
  }
```
在 StartDaemonThreads()中，启动的是这4种Daemon线程：

1. HeapTaskDaemon 完成HeapTask的线程，HeapTask 有一下几种
    - ConcurrentGCTask 
    - CollectorTransitionTask 
    - HeapTrimTask
    - TriggerPostForkCCGcTask
    - ReduceTargetFootprintTask 
    - ClearedReferenceTask 
    - RecursiveTask
    - TestOrderTask 
    - MapBootImageMethodsTask
    - StartupCompletedTask
    - TraceStopTask
2. ReferenceQueueDaemon 
    - 监视 GC 清理出的引用，把它们从内部链表转到正式的 ReferenceQueue，并在必要时触发底层 “post-cleanup” 回调。
    - 为何：Java 的弱/软/虚引用机制要求在对象回收后把引用入队，以便开发者拿到回调或在 ReferenceQueue 上做清理工作；ReferenceQueueDaemon 就是负责驱动这整个“入队 → 回调”流程的那根主心骨。）
3. FinalizerDaemon 
    - 从 FinalizerReference.queue（即 ReferenceQueue）里取出已经被 GC 判定为可回收的引用，依次调用它们对应对象的 finalize() 方法（或 Cleaner.clean()），然后从内部链表里删除引用，确保后续不会再次执行。
4. FinalizerWatchdogDaemon
    - 监控 FinalizerDaemon（和前面提到的 ReferenceQueueDaemon）是否“卡住”——即某次 finalize() 调用或队列转移花费时间过长。若检测到超时且无调试器连接，就触发收集堆栈、发送 SIGQUIT、抛出或者上报超时异常，甚至终止进程，防止系统因未完成的清理而死锁或崩溃。


继续Runtime.start()
```cpp
  VLOG(startup) << "Runtime::Start exiting";
  finished_starting_ = true;
  // 标记 finished_starting_ = true，表示运行时的所有核心服务都已就绪。

  // 可选的 Method Tracing 与 Profile 注册： 如果命令行或 VM 选项指定要开始方法跟踪（method trace），在此刻调用 Trace::Start() 打开跟踪文件。

  if (trace_config_.get() != nullptr && trace_config_->trace_file != "") {
    ScopedThreadStateChange tsc(self, ThreadState::kWaitingForMethodTracingStart);
    int flags = 0;
    if (trace_config_->clock_source == TraceClockSource::kDual) {
      flags = Trace::TraceFlag::kTraceClockSourceWallClock |
              Trace::TraceFlag::kTraceClockSourceThreadCpu;
    } else if (trace_config_->clock_source == TraceClockSource::kWall) {
      flags = Trace::TraceFlag::kTraceClockSourceWallClock;
    } else if (TraceClockSource::kThreadCpu == trace_config_->clock_source) {
      flags = Trace::TraceFlag::kTraceClockSourceThreadCpu;
    } else {
      LOG(ERROR) << "Unexpected clock source";
    }
    Trace::Start(trace_config_->trace_file.c_str(),
                 static_cast<int>(trace_config_->trace_file_size),
                 flags,
                 trace_config_->trace_output_mode,
                 trace_config_->trace_mode,
                 0);
  }

 // 如果要保存 Profile，将当前 classpath 注册到 JIT 的 Profile Saver，方便后续 AOT 编译或分析。
  // In case we have a profile path passed as a command line argument,
  // register the current class path for profiling now. Note that we cannot do
  // this before we create the JIT and having it here is the most convenient way.
  // This is used when testing profiles with dalvikvm command as there is no
  // framework to register the dex files for profiling.
  if (jit_.get() != nullptr && jit_options_->GetSaveProfilingInfo() &&
      !jit_options_->GetProfileSaverOptions().GetProfilePath().empty()) {
    std::vector<std::string> dex_filenames;
    Split(class_path_string_, ':', &dex_filenames);

    // We pass "" as the package name because at this point we don't know it. It could be the
    // Zygote or it could be a dalvikvm cmd line execution. The package name will be re-set during
    // post-fork or during RegisterAppInfo.
    //
    // Also, it's ok to pass "" to the ref profile filename. It indicates we don't have
    // a reference profile.
    RegisterAppInfo(
        /*package_name=*/ "",
        dex_filenames,
        jit_options_->GetProfileSaverOptions().GetProfilePath(),
        /*ref_profile_filename=*/ "",
        kVMRuntimePrimaryApk);
  }

  return true;
}
```

总结：
Runtime::Start() 按顺序做了：
	1.	注册本地方法和信号处理；
	2.	执行必要的静态初始化和 Intrinsics 设置；
	3.	启动 JIT 与后台Daemon线程；
	4.	回调各阶段，让监控工具抓取 VM 启动事件；
	5.	生成 SystemClassLoader 并启动所有守护线程；
	6.	根据配置启动方法跟踪与 Profile 注册。



#### 执行Java业务代码

回到AndroidRuntime::start()函数
```cpp
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
        }
    }
```
在创建完ART虚拟机之后，使用JNI的CallStaticVoidMethod调用主方法，从这里开始Java业务代码的执行。

```cpp
// 告诉编译器在这个函数里不要插入栈保护（stack canary）检查，避免在可变参数处理时出问题。
 NO_STACK_PROTECTOR
  static void CallStaticVoidMethod(JNIEnv* env, jclass, jmethodID mid, ...) {
    // 初始化 C 语言的可变参数处理，从 mid 之后开始读。
    va_list ap;
    va_start(ap, mid);
    // 一个 RAII 对象，保证函数返回时会自动调用 va_end(ap)，避免忘记清理。
    ScopedVAArgs free_args_later(&ap);
    CHECK_NON_NULL_ARGUMENT_RETURN_VOID(mid);
    // 把当前线程切换到可以操作 mirror::Object* 等内部结构的模式，并且确保线程在退出这个作用域时会正确地清理／恢复状态。
    ScopedObjectAccess soa(env);
    //第三个参数是jmethodID类型
    InvokeWithVarArgs(soa, nullptr, mid, ap);
  }
```

这个函数会用jni::DecodeArtMethod(mid)把jmethodID转化成ArtMethod的结构。
```cpp
template <>
NO_STACK_PROTECTOR
JValue InvokeWithVarArgs(const ScopedObjectAccessAlreadyRunnable& soa,
                         jobject obj,
                         jmethodID mid,
                         va_list args) REQUIRES_SHARED(Locks::mutator_lock_) {
  DCHECK(mid != nullptr) << "Called with null jmethodID";
  return InvokeWithVarArgs(soa, obj, jni::DecodeArtMethod(mid), args);
}
```

实际的InvokeWithVarArgs函数

```cpp
template <>
NO_STACK_PROTECTOR
JValue InvokeWithVarArgs(const ScopedObjectAccessAlreadyRunnable& soa,
                         jobject obj,
                         ArtMethod* method,
                         va_list args) REQUIRES_SHARED(Locks::mutator_lock_) {
  // We want to make sure that the stack is not within a small distance from the
  // protected region in case we are calling into a leaf function whose stack
  // check has been elided.
  if (UNLIKELY(__builtin_frame_address(0) < soa.Self()->GetStackEnd<kNativeStackType>())) {
    ThrowStackOverflowError<kNativeStackType>(soa.Self());
    return JValue();
  }
  // Java 里 new String(...) 最后底层会调用 <init> 构造器，但 ART 里为了性能和去重，实际上会把它替换成一个工厂方法 StringFactory.createFrom(...)。
	// 如果 method 真的是 String 的 <init>，就重定向成对应的 factory 方法。
  bool is_string_init = method->IsStringConstructor();
  if (is_string_init) {
    // Replace calls to String.<init> with equivalent StringFactory call.
    method = WellKnownClasses::StringInitToStringFactory(method);
  }
  // 这里的 receiver 就是 this引用所指向的对象
  ObjPtr<mirror::Object> receiver = method->IsStatic() ? nullptr : soa.Decode<mirror::Object>(obj);
  uint32_t shorty_len = 0;
  // proxy method 就是那些 ART 运行时自动生成（而不是你写在 .java/.dex 里的）的方法实现
  const char* shorty =
      method->GetInterfaceMethodIfProxy(kRuntimePointerSize)->GetShorty(&shorty_len);
  JValue result;
  ArgArray arg_array(shorty, shorty_len);
  arg_array.BuildArgArrayFromVarArgs(soa, receiver, args);
  InvokeWithArgArray(soa, method, &arg_array, &result, shorty);
  if (is_string_init) {
    // For string init, remap original receiver to StringFactory result.
    UpdateReference(soa.Self(), obj, result.GetL());
  }
  return result;
}
```
在做完参数准备后实际调用方法是通过ArtMethod的Invoke成员函数实现的。
```cpp
void InvokeWithArgArray(const ScopedObjectAccessAlreadyRunnable& soa,
                               ArtMethod* method, ArgArray* arg_array, JValue* result,
                               const char* shorty)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  uint32_t* args = arg_array->GetArray();
  if (UNLIKELY(soa.Env()->IsCheckJniEnabled())) {
    CheckMethodArguments(soa.Vm(), method->GetInterfaceMethodIfProxy(kRuntimePointerSize), args);
  }
  method->Invoke(soa.Self(), args, arg_array->GetNumBytes(), result, shorty);
}
```

#### ArtMethod::Invoke

```cpp

NO_STACK_PROTECTOR
void ArtMethod::Invoke(Thread* self, uint32_t* args, uint32_t args_size, JValue* result,
                       const char* shorty) {
  if (UNLIKELY(__builtin_frame_address(0) < self->GetStackEnd<kNativeStackType>())) {
    ThrowStackOverflowError<kNativeStackType>(self);
    return;
  }

  if (kIsDebugBuild) {
    // 确保当前线程处于“可执行 Java 代码”状态（没有被暂停、没有在 GC 中等）。
    self->AssertThreadSuspensionIsAllowable();
    CHECK_EQ(ThreadState::kRunnable, self->GetState());
    CHECK_STREQ(GetInterfaceMethodIfProxy(kRuntimePointerSize)->GetShorty(), shorty);
  }

  // Push a transition back into managed code onto the linked list in thread.
  ManagedStack fragment;
  self->PushManagedStackFragment(&fragment);

  Runtime* runtime = Runtime::Current();
  // Call the invoke stub, passing everything as arguments.
  // If the runtime is not yet started or it is required by the debugger, then perform the
  // Invocation by the interpreter, explicitly forcing interpretation over JIT to prevent
  // cycling around the various JIT/Interpreter methods that handle method invocation.
  if (UNLIKELY(!runtime->IsStarted() ||
               (self->IsForceInterpreter() && !IsNative() && !IsProxyMethod() && IsInvokable()))) {
    // 解释器模式
    if (IsStatic()) {
      art::interpreter::EnterInterpreterFromInvoke(
          self, this, nullptr, args, result, /*stay_in_interpreter=*/ true);
    } else {
      mirror::Object* receiver =
          reinterpret_cast<StackReference<mirror::Object>*>(&args[0])->AsMirrorPtr();
      art::interpreter::EnterInterpreterFromInvoke(
          self, this, receiver, args + 1, result, /*stay_in_interpreter=*/ true);
    }
  } else {
    DCHECK_EQ(runtime->GetClassLinker()->GetImagePointerSize(), kRuntimePointerSize);
    // 调用编译后quick代码
    constexpr bool kLogInvocationStartAndReturn = false;
    bool have_quick_code = GetEntryPointFromQuickCompiledCode() != nullptr;
    if (LIKELY(have_quick_code)) {
      if (kLogInvocationStartAndReturn) {
        LOG(INFO) << StringPrintf(
            "Invoking '%s' quick code=%p static=%d", PrettyMethod().c_str(),
            GetEntryPointFromQuickCompiledCode(), static_cast<int>(IsStatic() ? 1 : 0));
      }

      // Ensure that we won't be accidentally calling quick compiled code when -Xint.
      if (kIsDebugBuild && runtime->GetInstrumentation()->IsForcedInterpretOnly()) {
        CHECK(!runtime->UseJitCompilation());
        const void* oat_quick_code =
            (IsNative() || !IsInvokable() || IsProxyMethod() || IsObsolete())
            ? nullptr
            : GetOatMethodQuickCode(runtime->GetClassLinker()->GetImagePointerSize());
        CHECK(oat_quick_code == nullptr || oat_quick_code != GetEntryPointFromQuickCompiledCode())
            << "Don't call compiled code when -Xint " << PrettyMethod();
      }

      // 调用stub函数
      if (!IsStatic()) {
        (*art_quick_invoke_stub)(this, args, args_size, self, result, shorty);
      } else {
        (*art_quick_invoke_static_stub)(this, args, args_size, self, result, shorty);
      }
      if (UNLIKELY(self->GetException() == Thread::GetDeoptimizationException())) {
        // Unusual case where we were running generated code and an
        // exception was thrown to force the activations to be removed from the
        // stack. Continue execution in the interpreter.
        self->DeoptimizeWithDeoptimizationException(result);
      }
      if (kLogInvocationStartAndReturn) {
        LOG(INFO) << StringPrintf("Returned '%s' quick code=%p", PrettyMethod().c_str(),
                                  GetEntryPointFromQuickCompiledCode());
      }
    } else {
      LOG(INFO) << "Not invoking '" << PrettyMethod() << "' code=null";
      if (result != nullptr) {
        result->SetJ(0);
      }
    }
  }

  // Pop transition.
  self->PopManagedStackFragment(fragment);
}
```

#### 解释器入口 art::interpreter::EnterInterpreterFromInvoke

```cpp
void EnterInterpreterFromInvoke(Thread* self,
                                ArtMethod* method,
                                ObjPtr<mirror::Object> receiver,
                                uint32_t* args,
                                JValue* result,
                                bool stay_in_interpreter) {
  DCHECK_EQ(self, Thread::Current());
  bool implicit_check = Runtime::Current()->GetImplicitStackOverflowChecks();
  if (UNLIKELY(__builtin_frame_address(0) < self->GetStackEndForInterpreter(implicit_check))) {
    ThrowStackOverflowError<kNativeStackType>(self);
    return;
  }

  // This can happen if we are in forced interpreter mode and an obsolete method is called using
  // reflection.
  if (UNLIKELY(method->IsObsolete())) {
    ThrowInternalError("Attempting to invoke obsolete version of '%s'.",
                       method->PrettyMethod().c_str());
    return;
  }

  const char* old_cause = self->StartAssertNoThreadSuspension("EnterInterpreterFromInvoke");
  CodeItemDataAccessor accessor(method->DexInstructionData());
  uint16_t num_regs;
  uint16_t num_ins;
  // 从 Dex CodeItem 里读出本方法声明时在 ART 里的总寄存器数 num_regs 和输入寄存器数 num_ins。
  if (accessor.HasCodeItem()) {
    num_regs =  accessor.RegistersSize();
    num_ins = accessor.InsSize();
  } else if (!method->IsInvokable()) {
    self->EndAssertNoThreadSuspension(old_cause);
    method->ThrowInvocationTimeError(receiver);
    return;
  } else {
    DCHECK(method->IsNative()) << method->PrettyMethod();
    // 如果是 native，就根据 shorty 自动算出要多少寄存器。
    num_regs = num_ins = ArtMethod::NumArgRegisters(method->GetShortyView());
    if (!method->IsStatic()) {
      num_regs++;
      num_ins++;
    }
  }
  // shadow frame 就是解释器在执行时维护的那一帧虚拟栈帧
  // Set up shadow frame with matching number of reference slots to vregs.
  ShadowFrameAllocaUniquePtr shadow_frame_unique_ptr =
      CREATE_SHADOW_FRAME(num_regs, method, /* dex pc */ 0);
  ShadowFrame* shadow_frame = shadow_frame_unique_ptr.get();

  // 接下来的逻辑作用是将 args 拷贝到 ShadowFrame
  size_t cur_reg = num_regs - num_ins;
  // 先把非 static 的 receiver （this）放到第一个 vreg。
  if (!method->IsStatic()) {
    CHECK(receiver != nullptr);
    shadow_frame->SetVRegReference(cur_reg, receiver);
    ++cur_reg;
  }
  uint32_t shorty_len = 0;
  const char* shorty = method->GetShorty(&shorty_len);
  // 再按 shorty 的顺序
  for (size_t shorty_pos = 0, arg_pos = 0; cur_reg < num_regs; ++shorty_pos, ++arg_pos, cur_reg++) {
    DCHECK_LT(shorty_pos + 1, shorty_len);
    switch (shorty[shorty_pos + 1]) {
      case 'L': {
        // 'L' 就当引用处理，读成 mirror::Object* 后 SetVRegReference
        ObjPtr<mirror::Object> o =
            reinterpret_cast<StackReference<mirror::Object>*>(&args[arg_pos])->AsMirrorPtr();
        shadow_frame->SetVRegReference(cur_reg, o);
        break;
      }
      case 'J': case 'D': {
        // 'J'/'D'（long/double）占两个 vreg，要合并两次 arg 才算一个 64 位
        uint64_t wide_value = (static_cast<uint64_t>(args[arg_pos + 1]) << 32) | args[arg_pos];
        shadow_frame->SetVRegLong(cur_reg, wide_value);
        cur_reg++;
        arg_pos++;
        break;
      }
      default:
        // 其他原始类型直接 SetVReg
        shadow_frame->SetVReg(cur_reg, args[arg_pos]);
        break;
    }
  }
  self->EndAssertNoThreadSuspension(old_cause);
  if (!EnsureInitialized(self, shadow_frame)) {
    return;
  }
  self->PushShadowFrame(shadow_frame);
  if (LIKELY(!method->IsNative())) {
    // 对于普通 Java 方法，调用解释器核心 Execute(...)，一路走字节码逐条执行到 return。
    JValue r = Execute(self, accessor, *shadow_frame, JValue(), stay_in_interpreter);
    if (result != nullptr) {
      *result = r;
    }
  } else {
    // We don't expect to be asked to interpret native code (which is entered via a JNI compiler
    // generated stub) except during testing and image writing.
    // Update args to be the args in the shadow frame since the input ones could hold stale
    // references pointers due to moving GC.
    // 如果方法是 native，ART 会调用一个专门的“解释器版 JNI”入口 InterpreterJni，模拟 JNI 调用约定，但仍在解释器栈框架里执行。
    args = shadow_frame->GetVRegArgs(method->IsStatic() ? 0 : 1);
    if (!Runtime::Current()->IsStarted()) {
      UnstartedRuntime::Jni(self, method, receiver.Ptr(), args, result);
    } else {
      InterpreterJni(self, method, shorty, receiver, args, result);
    }
  }
  self->PopShadowFrame();
}
```

#### 解释器实际执行 

```cpp
NO_STACK_PROTECTOR
static inline JValue Execute(
    Thread* self,
    const CodeItemDataAccessor& accessor,
    ShadowFrame& shadow_frame,
    JValue result_register,
    bool stay_in_interpreter = false,
    bool from_deoptimize = false) REQUIRES_SHARED(Locks::mutator_lock_) {
  DCHECK(!shadow_frame.GetMethod()->IsAbstract());
  DCHECK(!shadow_frame.GetMethod()->IsNative());

  // We cache the result of NeedsDexPcEvents in the shadow frame so we don't need to call
  // NeedsDexPcEvents on every instruction for better performance. NeedsDexPcEvents only gets
  // updated asynchronoulsy in a SuspendAll scope and any existing shadow frames are updated with
  // new value. So it is safe to cache it here.
  shadow_frame.SetNotifyDexPcMoveEvents(
      Runtime::Current()->GetInstrumentation()->NeedsDexPcEvents(shadow_frame.GetMethod(), self));

  if (LIKELY(!from_deoptimize)) {  // Entering the method, but not via deoptimization.
    if (kIsDebugBuild) {
      CHECK_EQ(shadow_frame.GetDexPC(), 0u);
      self->AssertNoPendingException();
    }
    ArtMethod *method = shadow_frame.GetMethod();

    // If we can continue in JIT and have JITed code available execute JITed code.
    // 看看是否可以直接跳到 JIT 编译后的机器码
    if (!stay_in_interpreter &&
        !self->IsForceInterpreter() &&
        !shadow_frame.GetForcePopFrame() &&
        !shadow_frame.GetNotifyDexPcMoveEvents()) {
      jit::Jit* jit = Runtime::Current()->GetJit();
      if (jit != nullptr) {
        jit->MethodEntered(self, shadow_frame.GetMethod());
        if (jit->CanInvokeCompiledCode(method)) {
          JValue result;

          // Pop the shadow frame before calling into compiled code.
          self->PopShadowFrame();
          // Calculate the offset of the first input reg. The input registers are in the high regs.
          // It's ok to access the code item here since JIT code will have been touched by the
          // interpreter and compiler already.
          uint16_t arg_offset = accessor.RegistersSize() - accessor.InsSize();
          // 从解释器切到 JIT 机器码
          ArtInterpreterToCompiledCodeBridge(self, nullptr, &shadow_frame, arg_offset, &result);
          // Push the shadow frame back as the caller will expect it.
          self->PushShadowFrame(&shadow_frame);

          return result;
        }
      }
    }

    // 拦截方法入口做 entry/unwind 回调、支持调试器强制 pop frame。
    instrumentation::Instrumentation* instrumentation = Runtime::Current()->GetInstrumentation();
    if (UNLIKELY(instrumentation->HasMethodEntryListeners() || shadow_frame.GetForcePopFrame())) {
      instrumentation->MethodEnterEvent(self, method);
      if (UNLIKELY(shadow_frame.GetForcePopFrame())) {
        // The caller will retry this invoke or ignore the result. Just return immediately without
        // any value.
        DCHECK(Runtime::Current()->AreNonStandardExitsEnabled());
        JValue ret = JValue();
        PerformNonStandardReturn(self,
                                 shadow_frame,
                                 ret,
                                 instrumentation,
                                 /* unlock_monitors= */ false);
        return ret;
      }
      if (UNLIKELY(self->IsExceptionPending())) {
        instrumentation->MethodUnwindEvent(self,
                                           method,
                                           0);
        JValue ret = JValue();
        if (UNLIKELY(shadow_frame.GetForcePopFrame())) {
          DCHECK(Runtime::Current()->AreNonStandardExitsEnabled());
          PerformNonStandardReturn(self,
                                   shadow_frame,
                                   ret,
                                   instrumentation,
                                   /* unlock_monitors= */ false);
        }
        return ret;
      }
    }
  }

  ArtMethod* method = shadow_frame.GetMethod();

  DCheckStaticState(self, method);

  // Lock counting is a special version of accessibility checks, and for simplicity and
  // reduction of template parameters, we gate it behind access-checks mode.
  DCHECK_IMPLIES(method->SkipAccessChecks(), !method->MustCountLocks());

  VLOG(interpreter) << "Interpreting " << method->PrettyMethod();

  // 字节码执行循环
  return ExecuteSwitch(self, accessor, shadow_frame, result_register);
}
```

ExecuteSwitch 函数本身并不是真正的字节码分发循环，它只是做了一个“分流”，把后续的工作交给了真正的 interpreter 实现函数 ExecuteSwitchImpl：

```cpp
NO_STACK_PROTECTOR
static JValue ExecuteSwitch(Thread* self,
                            const CodeItemDataAccessor& accessor,
                            ShadowFrame& shadow_frame,
                            JValue result_register) REQUIRES_SHARED(Locks::mutator_lock_) {
  Runtime* runtime = Runtime::Current();
  // ART 支持 在运行时做类的热重定义／回滚（transactional class loading），这时候解释器的行为要稍微不一样（要维护旧版／新版类元信息的隔离），因此需要一个事务安全版本的解释器。
  // 把“选哪个版本的 interpreter”这件事提取到最外层，核心的字节码分发逻辑不用重复维护。
  // 但其实就模版变量的值不一样，ExecuteSwitchImplCpp<true> 与 ExecuteSwitchImplCpp<false>
  auto switch_impl_cpp = runtime->IsActiveTransaction()
      ? runtime->GetClassLinker()->GetTransactionalInterpreter()
      : reinterpret_cast<const void*>(&ExecuteSwitchImplCpp</*transaction_active=*/ false>);
  return ExecuteSwitchImpl(
      self, accessor, shadow_frame, result_register, switch_impl_cpp);
}

// Wrapper around the switch interpreter which ensures we can unwind through it.
ALWAYS_INLINE inline JValue ExecuteSwitchImpl(Thread* self,
                                              const CodeItemDataAccessor& accessor,
                                              ShadowFrame& shadow_frame,
                                              JValue result_register,
                                              const void* switch_impl_cpp)
  REQUIRES_SHARED(Locks::mutator_lock_) {
  SwitchImplContext ctx {
    .self = self,
    .accessor = accessor,
    .shadow_frame = shadow_frame,
    .result_register = result_register,
    .result = JValue(),
  };
  const uint16_t* dex_pc = ctx.accessor.Insns();
  ExecuteSwitchImplAsm(&ctx, switch_impl_cpp, dex_pc);
  return ctx.result;
}


// Hand-written assembly method which wraps the C++ implementation,
// while defining the DEX PC in the CFI so that libunwind can resolve it.
extern "C" void ExecuteSwitchImplAsm(
    SwitchImplContext* ctx, const void* impl, const uint16_t* dexpc)
    REQUIRES_SHARED(Locks::mutator_lock_);
```

ExecuteSwitchImplAsm的定义
```arm
// Wrap ExecuteSwitchImpl in assembly method which specifies DEX PC for unwinding.
//  Argument 0: x0: The context pointer for ExecuteSwitchImpl.
//  Argument 1: x1: Pointer to the templated ExecuteSwitchImpl to call.
//  Argument 2: x2: The value of DEX PC (memory address of the methods bytecode).
ENTRY ExecuteSwitchImplAsm
    SAVE_TWO_REGS_INCREASE_FRAME x19, xLR, 16
    mov x19, x2                                   // x19 = DEX PC
    CFI_DEFINE_DEX_PC_WITH_OFFSET(0 /* x0 */, 19 /* x19 */, 0)
    // x1 是第二个参数：指向实际的 ExecuteSwitchImplCpp<…> 函数地址。
    // blr x1 就是“branch with link to register x1”，等同于调用那个 C++ 模板函数。
    blr x1                                        // Call the wrapped method.
    RESTORE_TWO_REGS_DECREASE_FRAME x19, xLR, 16
    ret
END ExecuteSwitchImplAsm
```

包装调用 ExecuteSwitchImplCpp，在 unwind 信息里注入“当前 DEX PC”，以便遇到异常或需要栈回溯时，能把解释器层面的字节码位置打印出来，而不是机器指令地址。


真正解释执行的部分
```cpp

template<bool transaction_active>
NO_STACK_PROTECTOR
void ExecuteSwitchImplCpp(SwitchImplContext* ctx) {
  Thread* self = ctx->self;
  const CodeItemDataAccessor& accessor = ctx->accessor;
  ShadowFrame& shadow_frame = ctx->shadow_frame;
  self->VerifyStack();

  uint32_t dex_pc = shadow_frame.GetDexPC();
  const auto* const instrumentation = Runtime::Current()->GetInstrumentation();
  const uint16_t* const insns = accessor.Insns();
  // 根据dex_pc获取指令
  const Instruction* next = Instruction::At(insns + dex_pc);

  DCHECK(!shadow_frame.GetForceRetryInstruction())
      << "Entered interpreter from invoke without retry instruction being handled!";

  while (true) {
    const Instruction* const inst = next;
    dex_pc = inst->GetDexPc(insns);
    shadow_frame.SetDexPC(dex_pc);
    TraceExecution(shadow_frame, inst, dex_pc);
    // inst_data 是指令编码，后面用来提取具体的 Opcode。
    uint16_t inst_data = inst->Fetch16(0);
    bool exit = false;
    bool success;  // Moved outside to keep frames small under asan.
    if (InstructionHandler<transaction_active, Instruction::kInvalidFormat>(
            ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit).
            Preamble()) {
      DCHECK_EQ(self->IsExceptionPending(), inst->Opcode(inst_data) == Instruction::MOVE_EXCEPTION);
      // Opcode 分发
      // 	ART 为每个 DEX Opcode（如 OP_MOVE,OP_INVOKE_VIRTUAL 等）都生成一个 OP_<NAME> 模板函数。
	    // 调用它时传入当前上下文、帧、指令等，它会执行真正的指令语义，然后返回 success：
	    // 如果返回 true，说明指令执行完毕，直接 continue 下一循环；
      // 如果返回 false，则跳出 switch，交给后续的异常或退出处理。
      switch (inst->Opcode(inst_data)) {
#define OPCODE_CASE(OPCODE, OPCODE_NAME, NAME, FORMAT, i, a, e, v)                                \
        case OPCODE: {                                                                            \
          next = inst->RelativeAt(Instruction::SizeInCodeUnits(Instruction::FORMAT));             \
          success = OP_##OPCODE_NAME<transaction_active>(                                         \
              ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit);     \
          if (success) {                                                                          \
            continue;                                                                             \
          }                                                                                       \
          break;                                                                                  \
        }
  DEX_INSTRUCTION_LIST(OPCODE_CASE)
#undef OPCODE_CASE
      }
    }
    if (exit) {
      shadow_frame.SetDexPC(dex::kDexNoIndex);
      return;  // Return statement or debugger forced exit.
    }
    if (self->IsExceptionPending()) {
      if (!InstructionHandler<transaction_active, Instruction::kInvalidFormat>(
              ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit).
              HandlePendingException()) {
        shadow_frame.SetDexPC(dex::kDexNoIndex);
        return;  // Locally unhandled exception - return to caller.
      }
      // Continue execution in the catch block.
    }
  }
}  // NOLINT(readability/fn_size)
```

InstructionHandler的Preamble函数进行一些前置检查，查看是否存在debug或者异常等其他情况导致这条指令不用被执行。

```cpp
  // Code to run before each dex instruction.
  HANDLER_ATTRIBUTES bool Preamble() {
    /* We need to put this before & after the instrumentation to avoid having to put in a */
    /* post-script macro.                                                                 */
    // 如果在 debug 或者事务里，有某个 listener/事务强制让当前方法“立即返回”，我们不应该再执行这条指令，而是要直接走一遍非标准退出（PerformNonStandardReturn），然后立刻跳出整个解释器循环，返回给调用者。
    if (!CheckForceReturn()) {
      return false;
    }
    if (UNLIKELY(InstrumentationHandler::NeedsDexPcEvents(shadow_frame_))) {
      // 如果有人在这个方法上设了断点或者做了单步（NeedsDexPcEvents==true），每当我们要“执行”到新的 DexPC，就得先跑一遍 DoDexPcMoveEvent，让调试器或 profiler 知道“我现在要走到这个字节码了”。
	    // 特殊处理 MOVE_RESULT_OBJECT：如果上条指令是对象结果搬运，需要先把之前的结果存到 listener 能看到的位置。
      uint8_t opcode = inst_->Opcode(inst_data_);
      bool is_move_result_object = (opcode == Instruction::MOVE_RESULT_OBJECT);
      JValue* save_ref = is_move_result_object ? &ctx_->result_register : nullptr;
      if (UNLIKELY(!InstrumentationHandler::DoDexPcMoveEvent(Self(),
                                                             Accessor(),
                                                             shadow_frame_,
                                                             DexPC(),
                                                             Instrumentation(),
                                                             save_ref))) {
        // 调用 DoDexPcMoveEvent()，它会通知所有注册的 DexPc 移动 listener
        DCHECK(Self()->IsExceptionPending());
        // Do not raise exception event if it is caused by other instrumentation event.
        shadow_frame_.SetSkipNextExceptionEvent(true);
        return false;  // Pending exception.
      }
      if (!CheckForceReturn()) {
        return false;
      }
    }

    // Call any exception handled event handlers after the dex pc move event.
    // The order is important to see a consistent behaviour in the debuggers.
    // See b/333446719 for more discussion.
    // 当上一次异常被 catch 掉（解释器跳到某个 catch 分支），我们要在执行下一个普通指令前先通知 “异常已处理” 的 listener，确保 debugger 或者其他 instrumentation 能看到“catch 到异常”这个时刻。
    if (UNLIKELY(shadow_frame_.GetNotifyExceptionHandledEvent())) {
      shadow_frame_.SetNotifyExceptionHandledEvent(/*enable=*/ false);
      bool is_move_exception = (inst_->Opcode(inst_data_) == Instruction::MOVE_EXCEPTION);

      if (!InstrumentationHandler::ExceptionHandledEvent(
              Self(), is_move_exception, Instrumentation())) {
        DCHECK(Self()->IsExceptionPending());
        // TODO(375373721): We need to set SetSkipNextExceptionEvent here since the exception was
        // thrown by an instrumentation handler.
        return false;  // Pending exception.
      }

      if (!CheckForceReturn()) {
        return false;
      }
    }
    return true;
  }
```

DEX_INSTRUCTION_LIST 定义在 art/libdexfile/dex/dex_instruction_list.h中，列举了所有的Dex指令

```cpp

// V(opcode, instruction_code, name, format, index, flags, extended_flags, verifier_flags);
#define DEX_INSTRUCTION_LIST(V) \
  V(0x00, NOP, "nop", k10x, kIndexNone, kContinue, 0, kVerifyNothing) \
  V(0x01, MOVE, "move", k12x, kIndexNone, kContinue, 0, kVerifyRegA | kVerifyRegB) \
  V(0x02, MOVE_FROM16, "move/from16", k22x, kIndexNone, kContinue, 0, kVerifyRegA | kVerifyRegB) \
  V(0x03, MOVE_16, "move/16", k32x, kIndexNone, kContinue, 0, kVerifyRegA | kVerifyRegB) \
  ......
```
每种指令对应的解释运行的函数也是使用宏进行定义的，调用的是InstructionHandler对应的OPCODE_NAME的成员函数。

```cpp

#define OPCODE_CASE(OPCODE, OPCODE_NAME, NAME, FORMAT, i, a, e, v)                                \
template<bool transaction_active>                                                                 \
ASAN_NO_INLINE NO_STACK_PROTECTOR static bool OP_##OPCODE_NAME(                                   \
    SwitchImplContext* ctx,                                                                       \
    const instrumentation::Instrumentation* instrumentation,                                      \
    Thread* self,                                                                                 \
    ShadowFrame& shadow_frame,                                                                    \
    uint16_t dex_pc,                                                                              \
    const Instruction* inst,                                                                      \
    uint16_t inst_data,                                                                           \
    const Instruction*& next,                                                                     \
    bool& exit) REQUIRES_SHARED(Locks::mutator_lock_) {                                           \
  InstructionHandler<transaction_active, Instruction::FORMAT> handler(                            \
      ctx, instrumentation, self, shadow_frame, dex_pc, inst, inst_data, next, exit);             \
  return LIKELY(handler.OPCODE_NAME());                                                           \
}

```

我们来看一下对于INVOKE_DIRECT指令的解释执行

```cpp
  HANDLER_ATTRIBUTES bool INVOKE_DIRECT() {
    return HandleInvoke<kDirect, /*is_range=*/ false>();
  }

    template<InvokeType type, bool is_range>
  HANDLER_ATTRIBUTES bool HandleInvoke() {
    bool success = DoInvoke<type, is_range>(
        Self(), shadow_frame_, inst_, inst_data_, ResultRegister());
    return PossiblyHandlePendingExceptionOnInvoke(!success);
  }

// Handles all invoke-XXX/range instructions except for invoke-polymorphic[/range].
// Returns true on success, otherwise throws an exception and returns false.
template<InvokeType type, bool is_range>
static ALWAYS_INLINE bool DoInvoke(Thread* self,
                                   ShadowFrame& shadow_frame,
                                   const Instruction* inst,
                                   uint16_t inst_data,
                                   JValue* result)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  // Make sure to check for async exceptions before anything else.
  if (UNLIKELY(self->ObserveAsyncException())) {
    return false;
  }
  const uint32_t vregC = is_range ? inst->VRegC_3rc() : inst->VRegC_35c();
  ObjPtr<mirror::Object> obj = type == kStatic ? nullptr : shadow_frame.GetVRegReference(vregC);
  ArtMethod* sf_method = shadow_frame.GetMethod();
  bool string_init = false;
  ArtMethod* called_method = FindMethodToCall<type>(
      self, sf_method, &obj, *inst, /* only_lookup_tls_cache= */ false, &string_init);
  if (called_method == nullptr) {
    DCHECK(self->IsExceptionPending());
    result->SetJ(0);
    return false;
  }

  return DoCall<is_range>(
      called_method, self, shadow_frame, inst, inst_data, string_init, result);

  
template<bool is_range>
NO_STACK_PROTECTOR
bool DoCall(ArtMethod* called_method,
            Thread* self,
            ShadowFrame& shadow_frame,
            const Instruction* inst,
            uint16_t inst_data,
            bool is_string_init,
            JValue* result) {
  // Argument word count.
  const uint16_t number_of_inputs =
      (is_range) ? inst->VRegA_3rc(inst_data) : inst->VRegA_35c(inst_data);

  // TODO: find a cleaner way to separate non-range and range information without duplicating
  //       code.
  uint32_t arg[Instruction::kMaxVarArgRegs] = {};  // only used in invoke-XXX.
  uint32_t vregC = 0;
  if (is_range) {
    vregC = inst->VRegC_3rc();
  } else {
    vregC = inst->VRegC_35c();
    inst->GetVarArgs(arg, inst_data);
  }

  return DoCallCommon<is_range>(
      called_method,
      self,
      shadow_frame,
      result,
      number_of_inputs,
      arg,
      vregC,
      is_string_init);
}


template <bool is_range>
static inline bool DoCallCommon(ArtMethod* called_method,
                                Thread* self,
                                ShadowFrame& shadow_frame,
                                JValue* result,
                                uint16_t number_of_inputs,
                                uint32_t (&arg)[Instruction::kMaxVarArgRegs],
                                uint32_t vregC,
                                bool string_init) {
  // Compute method information.
  CodeItemDataAccessor accessor(called_method->DexInstructionData());
  // Number of registers for the callee's call frame.
  uint16_t num_regs;
  // Test whether to use the interpreter or compiler entrypoint, and save that result to pass to
  // PerformCall. A deoptimization could occur at any time, and we shouldn't change which
  // entrypoint to use once we start building the shadow frame.

  const bool use_interpreter_entrypoint = ShouldStayInSwitchInterpreter(called_method);
  if (LIKELY(accessor.HasCodeItem())) {
    // When transitioning to compiled code, space only needs to be reserved for the input registers.
    // The rest of the frame gets discarded. This also prevents accessing the called method's code
    // item, saving memory by keeping code items of compiled code untouched.
    if (!use_interpreter_entrypoint) {
      DCHECK(!Runtime::Current()->IsAotCompiler()) << "Compiler should use interpreter entrypoint";
      num_regs = number_of_inputs;
    } else {
      num_regs = accessor.RegistersSize();
      DCHECK_EQ(string_init ? number_of_inputs - 1 : number_of_inputs, accessor.InsSize());
    }
  } else {
    DCHECK(called_method->IsNative() || called_method->IsProxyMethod());
    num_regs = number_of_inputs;
  }

  // Hack for String init:
  //
  // Rewrite invoke-x java.lang.String.<init>(this, a, b, c, ...) into:
  //         invoke-x StringFactory(a, b, c, ...)
  // by effectively dropping the first virtual register from the invoke.
  //
  // (at this point the ArtMethod has already been replaced,
  // so we just need to fix-up the arguments)
  //
  // Note that FindMethodFromCode in entrypoint_utils-inl.h was also special-cased
  // to handle the compiler optimization of replacing `this` with null without
  // throwing NullPointerException.
  uint32_t string_init_vreg_this = is_range ? vregC : arg[0];
  if (UNLIKELY(string_init)) {
    DCHECK_GT(num_regs, 0u);  // As the method is an instance method, there should be at least 1.

    // The new StringFactory call is static and has one fewer argument.
    if (!accessor.HasCodeItem()) {
      DCHECK(called_method->IsNative() || called_method->IsProxyMethod());
      num_regs--;
    }  // else ... don't need to change num_regs since it comes up from the string_init's code item
    number_of_inputs--;

    // Rewrite the var-args, dropping the 0th argument ("this")
    for (uint32_t i = 1; i < arraysize(arg); ++i) {
      arg[i - 1] = arg[i];
    }
    arg[arraysize(arg) - 1] = 0;

    // Rewrite the non-var-arg case
    vregC++;  // Skips the 0th vreg in the range ("this").
  }

  // Parameter registers go at the end of the shadow frame.
  DCHECK_GE(num_regs, number_of_inputs);
  size_t first_dest_reg = num_regs - number_of_inputs;
  DCHECK_NE(first_dest_reg, (size_t)-1);

  // Allocate shadow frame on the stack.
  const char* old_cause = self->StartAssertNoThreadSuspension("DoCallCommon");
  ShadowFrameAllocaUniquePtr shadow_frame_unique_ptr =
      CREATE_SHADOW_FRAME(num_regs, called_method, /* dex pc */ 0);
  ShadowFrame* new_shadow_frame = shadow_frame_unique_ptr.get();

  // Initialize new shadow frame by copying the registers from the callee shadow frame.
  if (!shadow_frame.GetMethod()->SkipAccessChecks()) {
    // Slow path.
    // We might need to do class loading, which incurs a thread state change to kNative. So
    // register the shadow frame as under construction and allow suspension again.
    ScopedStackedShadowFramePusher pusher(self, new_shadow_frame);
    self->EndAssertNoThreadSuspension(old_cause);

    // ArtMethod here is needed to check type information of the call site against the callee.
    // Type information is retrieved from a DexFile/DexCache for that respective declared method.
    //
    // As a special case for proxy methods, which are not dex-backed,
    // we have to retrieve type information from the proxy's method
    // interface method instead (which is dex backed since proxies are never interfaces).
    ArtMethod* method =
        new_shadow_frame->GetMethod()->GetInterfaceMethodIfProxy(kRuntimePointerSize);

    // We need to do runtime check on reference assignment. We need to load the shorty
    // to get the exact type of each reference argument.
    const dex::TypeList* params = method->GetParameterTypeList();
    uint32_t shorty_len = 0;
    const char* shorty = method->GetShorty(&shorty_len);

    // Handle receiver apart since it's not part of the shorty.
    size_t dest_reg = first_dest_reg;
    size_t arg_offset = 0;

    if (!method->IsStatic()) {
      size_t receiver_reg = is_range ? vregC : arg[0];
      new_shadow_frame->SetVRegReference(dest_reg, shadow_frame.GetVRegReference(receiver_reg));
      ++dest_reg;
      ++arg_offset;
      DCHECK(!string_init);  // All StringFactory methods are static.
    }

    // Copy the caller's invoke-* arguments into the callee's parameter registers.
    for (uint32_t shorty_pos = 0; dest_reg < num_regs; ++shorty_pos, ++dest_reg, ++arg_offset) {
      // Skip the 0th 'shorty' type since it represents the return type.
      DCHECK_LT(shorty_pos + 1, shorty_len) << "for shorty '" << shorty << "'";
      const size_t src_reg = (is_range) ? vregC + arg_offset : arg[arg_offset];
      switch (shorty[shorty_pos + 1]) {
        // Handle Object references. 1 virtual register slot.
        case 'L': {
          ObjPtr<mirror::Object> o = shadow_frame.GetVRegReference(src_reg);
          if (o != nullptr) {
            const dex::TypeIndex type_idx = params->GetTypeItem(shorty_pos).type_idx_;
            ObjPtr<mirror::Class> arg_type = method->GetDexCache()->GetResolvedType(type_idx);
            if (arg_type == nullptr) {
              StackHandleScope<1> hs(self);
              // Preserve o since it is used below and GetClassFromTypeIndex may cause thread
              // suspension.
              HandleWrapperObjPtr<mirror::Object> h = hs.NewHandleWrapper(&o);
              arg_type = method->ResolveClassFromTypeIndex(type_idx);
              if (arg_type == nullptr) {
                CHECK(self->IsExceptionPending());
                return false;
              }
            }
            if (!o->VerifierInstanceOf(arg_type)) {
              // This should never happen.
              std::string temp1, temp2;
              self->ThrowNewExceptionF("Ljava/lang/InternalError;",
                                       "Invoking %s with bad arg %d, type '%s' not instance of '%s'",
                                       new_shadow_frame->GetMethod()->GetName(), shorty_pos,
                                       o->GetClass()->GetDescriptor(&temp1),
                                       arg_type->GetDescriptor(&temp2));
              return false;
            }
          }
          new_shadow_frame->SetVRegReference(dest_reg, o);
          break;
        }
        // Handle doubles and longs. 2 consecutive virtual register slots.
        case 'J': case 'D': {
          uint64_t wide_value =
              (static_cast<uint64_t>(shadow_frame.GetVReg(src_reg + 1)) << BitSizeOf<uint32_t>()) |
               static_cast<uint32_t>(shadow_frame.GetVReg(src_reg));
          new_shadow_frame->SetVRegLong(dest_reg, wide_value);
          // Skip the next virtual register slot since we already used it.
          ++dest_reg;
          ++arg_offset;
          break;
        }
        // Handle all other primitives that are always 1 virtual register slot.
        default:
          new_shadow_frame->SetVReg(dest_reg, shadow_frame.GetVReg(src_reg));
          break;
      }
    }
  } else {
    if (is_range) {
      DCHECK_EQ(num_regs, first_dest_reg + number_of_inputs);
    }

    CopyRegisters<is_range>(shadow_frame,
                            new_shadow_frame,
                            arg,
                            vregC,
                            first_dest_reg,
                            number_of_inputs);
    self->EndAssertNoThreadSuspension(old_cause);
  }

  PerformCall(self,
              accessor,
              shadow_frame.GetMethod(),
              first_dest_reg,
              new_shadow_frame,
              result,
              use_interpreter_entrypoint);

  if (string_init && !self->IsExceptionPending()) {
    SetStringInitValueToAllAliases(&shadow_frame, string_init_vreg_this, *result);
  }

  return !self->IsExceptionPending();
}


inline void PerformCall(Thread* self,
                        const CodeItemDataAccessor& accessor,
                        ArtMethod* caller_method,
                        const size_t first_dest_reg,
                        ShadowFrame* callee_frame,
                        JValue* result,
                        bool use_interpreter_entrypoint)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  if (UNLIKELY(!Runtime::Current()->IsStarted())) {
    interpreter::UnstartedRuntime::Invoke(self, accessor, callee_frame, result, first_dest_reg);
    return;
  }

  if (!EnsureInitialized(self, callee_frame)) {
    return;
  }

  // 根据被调用方法使用不同的Bridge函数
  if (use_interpreter_entrypoint) {
    interpreter::ArtInterpreterToInterpreterBridge(self, accessor, callee_frame, result);
  } else {
    interpreter::ArtInterpreterToCompiledCodeBridge(
        self, caller_method, callee_frame, first_dest_reg, result);
  }
}



NO_STACK_PROTECTOR
void ArtInterpreterToInterpreterBridge(Thread* self,
                                       const CodeItemDataAccessor& accessor,
                                       ShadowFrame* shadow_frame,
                                       JValue* result) {
  bool implicit_check = Runtime::Current()->GetImplicitStackOverflowChecks();
  if (UNLIKELY(__builtin_frame_address(0) < self->GetStackEndForInterpreter(implicit_check))) {
    ThrowStackOverflowError<kNativeStackType>(self);
    return;
  }

  self->PushShadowFrame(shadow_frame);

  if (LIKELY(!shadow_frame->GetMethod()->IsNative())) {
    // 在这里又回到了Execute函数
    result->SetJ(Execute(self, accessor, *shadow_frame, JValue()).GetJ());
  } else {
    // We don't expect to be asked to interpret native code (which is entered via a JNI compiler
    // generated stub) except during testing and image writing.
    CHECK(!Runtime::Current()->IsStarted());
    bool is_static = shadow_frame->GetMethod()->IsStatic();
    ObjPtr<mirror::Object> receiver = is_static ? nullptr : shadow_frame->GetVRegReference(0);
    uint32_t* args = shadow_frame->GetVRegArgs(is_static ? 0 : 1);
    UnstartedRuntime::Jni(self, shadow_frame->GetMethod(), receiver.Ptr(), args, result);
  }

  self->PopShadowFrame();
}


```

如果是方法有被编译过的quick代码呢？

```cpp


NO_STACK_PROTECTOR
void ArtInterpreterToCompiledCodeBridge(Thread* self,
                                        ArtMethod* caller,
                                        ShadowFrame* shadow_frame,
                                        uint16_t arg_offset,
                                        JValue* result)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  ArtMethod* method = shadow_frame->GetMethod();
  // Basic checks for the arg_offset. If there's no code item, the arg_offset must be 0. Otherwise,
  // check that the arg_offset isn't greater than the number of registers. A stronger check is
  // difficult since the frame may contain space for all the registers in the method, or only enough
  // space for the arguments.
  if (kIsDebugBuild) {
    if (method->GetCodeItem() == nullptr) {
      DCHECK_EQ(0u, arg_offset) << method->PrettyMethod();
    } else {
      DCHECK_LE(arg_offset, shadow_frame->NumberOfVRegs());
    }
  }
  jit::Jit* jit = Runtime::Current()->GetJit();
  if (jit != nullptr && caller != nullptr) {
    jit->NotifyInterpreterToCompiledCodeTransition(self, caller);
  }
  // 使用Artmethod的Invoke成员函数
  method->Invoke(self, shadow_frame->GetVRegArgs(arg_offset),
                 (shadow_frame->NumberOfVRegs() - arg_offset) * sizeof(uint32_t),
                 result, method->GetInterfaceMethodIfProxy(kRuntimePointerSize)->GetShorty());
}

```

如果是被编译过的话，会调用或者 art_quick_invoke_stub, art_quick_invoke_static_stub
```cpp
// Called by art::ArtMethod::Invoke to do entry into a static method.
// TODO: migrate into an assembly implementation as with ARM64.
NO_STACK_PROTECTOR
extern "C" void art_quick_invoke_static_stub(ArtMethod* method, uint32_t* args,
                                             uint32_t args_size, Thread* self, JValue* result,
                                             const char* shorty) {
  quick_invoke_reg_setup<true>(method, args, args_size, self, result, shorty);
}
```

```cpp
template <bool kIsStatic>
NO_STACK_PROTECTOR
static void quick_invoke_reg_setup(ArtMethod* method, uint32_t* args, uint32_t args_size,
                                   Thread* self, JValue* result, const char* shorty) {
  // Note: We do not follow aapcs ABI in quick code for both softfp and hardfp.
  uint32_t core_reg_args[4];  // r0 ~ r3
  uint32_t fp_reg_args[16];  // s0 ~ s15 (d0 ~ d7)
  uint32_t gpr_index = 1;  // Index into core registers. Reserve r0 for ArtMethod*.
  uint32_t fpr_index = 0;  // Index into float registers.
  uint32_t fpr_double_index = 0;  // Index into float registers for doubles.
  uint32_t arg_index = 0;  // Index into argument array.
  const uint32_t result_in_float = (shorty[0] == 'F' || shorty[0] == 'D') ? 1 : 0;

  if (!kIsStatic) {
    // Copy receiver for non-static methods.
    core_reg_args[gpr_index++] = args[arg_index++];
  }

  for (uint32_t shorty_index = 1; shorty[shorty_index] != '\0'; ++shorty_index, ++arg_index) {
    char arg_type = shorty[shorty_index];
    switch (arg_type) {
      case 'D': {
        // Copy double argument into fp_reg_args if there are still floating point reg arguments.
        // Double should not overlap with float.
        fpr_double_index = std::max(fpr_double_index, RoundUp(fpr_index, 2));
        if (fpr_double_index < arraysize(fp_reg_args)) {
          fp_reg_args[fpr_double_index++] = args[arg_index];
          fp_reg_args[fpr_double_index++] = args[arg_index + 1];
        }
        ++arg_index;
        break;
      }
      case 'F':
        // Copy float argument into fp_reg_args if there are still floating point reg arguments.
        // If fpr_index is odd then its pointing at a hole next to an existing float argument. If we
        // encounter a float argument then pick it up from that hole. In the case fpr_index is even,
        // ensure that we don't pick up an argument that overlaps with with a double from
        // fpr_double_index. In either case, take care not to go beyond the maximum number of
        // floating point arguments.
        if (fpr_index % 2 == 0) {
          fpr_index = std::max(fpr_double_index, fpr_index);
        }
        if (fpr_index < arraysize(fp_reg_args)) {
          fp_reg_args[fpr_index++] = args[arg_index];
        }
        break;
      case 'J':
        if (gpr_index == 1) {
          // Don't use r1-r2 as a register pair, move to r2-r3 instead.
          gpr_index++;
        }
        if (gpr_index < arraysize(core_reg_args)) {
          // Note that we don't need to do this if two registers are not available
          // when using hard-fp. We do it anyway to leave this
          // code simple.
          core_reg_args[gpr_index++] = args[arg_index];
        }
        ++arg_index;
        FALLTHROUGH_INTENDED;  // Fall-through to take of the high part.
      default:
        if (gpr_index < arraysize(core_reg_args)) {
          core_reg_args[gpr_index++] = args[arg_index];
        }
        break;
    }
  }

  art_quick_invoke_stub_internal(method, args, args_size, self, result, result_in_float,
      core_reg_args, fp_reg_args);
}
```


```arm

    /*
     * Quick invocation stub internal.
     * On entry:
     *   r0 = method pointer
     *   r1 = argument array or null for no argument methods
     *   r2 = size of argument array in bytes
     *   r3 = (managed) thread pointer
     *   [sp] = JValue* result
     *   [sp + 4] = result_in_float
     *   [sp + 8] = core register argument array
     *   [sp + 12] = fp register argument array
     *  +-------------------------+
     *  | uint32_t* fp_reg_args   |
     *  | uint32_t* core_reg_args |
     *  |   result_in_float       | <- Caller frame
     *  |   Jvalue* result        |
     *  +-------------------------+
     *  |          lr             |
     *  |          r11            |
     *  |          r9             |
     *  |          r4             | <- r11
     *  +-------------------------+
     *  | uint32_t out[n-1]       |
     *  |    :      :             |        Outs
     *  | uint32_t out[0]         |
     *  | StackRef<ArtMethod>     | <- SP  value=null
     *  +-------------------------+
     */
ENTRY art_quick_invoke_stub_internal
    SPILL_ALL_CALLEE_SAVE_GPRS             @ spill regs (9)
    mov    r11, sp                         @ save the stack pointer
    .cfi_def_cfa_register r11

    mov    r9, r3                          @ move managed thread pointer into r9

    add    r4, r2, #4                      @ create space for method pointer in frame
    sub    r4, sp, r4                      @ reserve & align *stack* to 16 bytes: native calling
    and    r4, #0xFFFFFFF0                 @ convention only aligns to 8B, so we have to ensure ART
    mov    sp, r4                          @ 16B alignment ourselves.

    mov    r4, r0                          @ save method*
    add    r0, sp, #4                      @ pass stack pointer + method ptr as dest for memcpy
    bl     memcpy                          @ memcpy (dest, src, bytes)
    mov    ip, #0                          @ set ip to 0
    str    ip, [sp]                        @ store null for method* at bottom of frame

    ldr    ip, [r11, #48]                  @ load fp register argument array pointer
    vldm   ip, {s0-s15}                    @ copy s0 - s15

    ldr    ip, [r11, #44]                  @ load core register argument array pointer
    mov    r0, r4                          @ restore method*
    add    ip, ip, #4                      @ skip r0
    ldm    ip, {r1-r3}                     @ copy r1 - r3

    REFRESH_MARKING_REGISTER

    @ 这里是实际的调用入口，根据quick_code 的偏移加上 ArtMethod地址算出编译好的函数的位置，然后跳转
    ldr    ip, [r0, #ART_METHOD_QUICK_CODE_OFFSET_32]  @ get pointer to the code
    blx    ip                              @ call the method

    mov    sp, r11                         @ restore the stack pointer
    .cfi_def_cfa_register sp

    ldr    r4, [sp, #40]                   @ load result_is_float
    ldr    r9, [sp, #36]                   @ load the result pointer
    cmp    r4, #0
    ite    eq
    strdeq r0, [r9]                        @ store r0/r1 into result pointer
    vstrne d0, [r9]                        @ store s0-s1/d0 into result pointer

    pop    {r4, r5, r6, r7, r8, r9, r10, r11, pc}               @ restore spill regs
END art_quick_invoke_stub_internal
```

