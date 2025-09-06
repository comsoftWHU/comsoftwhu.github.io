---
layout: default
title: UnstartedRuntime为dex2oat提供迷你运行时
nav_order: 2
parent: dex2oat
author: Anonymous Committer
---

## UnstartedRuntime

UnstartedRuntime 的核心目标：为 AOT 编译提供“迷你运行时”

### 简介

简单来说，`UnstartedRuntime` 是一个**轻量级的、不完整的“迷你”ART运行时**。

它的**唯一目的**是为了支持`dex2oat`在生成系统启动镜像时，能够**执行核心Java类的静态初始化代码块 (`<clinit>`)**。

### 为什么需要它？

1. **AOT编译的需求**：为了提升应用启动速度和运行效率，Android 会在设备首次启动或系统更新时，使用 `dex2oat` 工具将核心库（如`core-oj.jar`）中的Java代码预先编译成本地机器码。在这个过程中，`dex2oat` 需要加载并**初始化**许多核心类（如 `java.lang.Object`, `java.lang.String`, `java.lang.System` 等）。初始化类就意味着要执行它们的静态初始化块 (`<clinit>`)。

2. **完整运行时的复杂性**：一个完整的ART运行时非常复杂，包含了线程管理、垃圾回收（GC）、JIT编译器、信号处理等重量级组件。为了仅仅执行一些类的 `<clinit>` 就启动一个完整的运行时，是极其低效且不必要的。

3. **依赖问题**：很多核心类的初始化代码可能会调用一些`native`方法，或者依赖于其他需要完整运行时才能正常工作的Java方法（比如文件IO、获取系统属性等）。

**UnstartedRuntime 的解决方案**：它提供了一套**手写的、用C++实现的、对部分核心Java方法的“替代品”**。当 `dex2oat` 在这个“未启动的”运行时环境中执行 `<clinit>` 时，如果遇到一个被特殊处理的方法调用，它不会去执行真正的Java代码或JNI代码，而是会调用 `UnstartedRuntime` 中对应的C++“山寨”实现。

### 工作机制：方法拦截与委托

`UnstartedRuntime` 的核心机制是**基于函数指针的查找表（HashMap）进行方法拦截**。

### 方法列表

在 `unstarted_runtime_list.h` 文件中，定义了两个宏列表：
    - `UNSTARTED_RUNTIME_DIRECT_LIST`：列出了需要被拦截的**普通Java方法**。
    - `UNSTARTED_RUNTIME_JNI_LIST`：列出了需要被拦截的**native方法**。

```cpp
V(CharacterToLowerCase, "Ljava/lang/Character;", "toLowerCase", "(I)I") \
```

分别代表`ShortName`, `Descriptor`, `Name`, `Signature`。

```cpp
// Methods that intercept available libcore implementations.
#define UNSTARTED_RUNTIME_DIRECT_LIST(V)    \
  V(CharacterToLowerCase, "Ljava/lang/Character;", "toLowerCase", "(I)I") \
  V(CharacterToUpperCase, "Ljava/lang/Character;", "toUpperCase", "(I)I") \
  V(ClassForName, "Ljava/lang/Class;", "forName", "(Ljava/lang/String;)Ljava/lang/Class;") \
  V(ClassForNameLong, "Ljava/lang/Class;", "forName", "(Ljava/lang/String;ZLjava/lang/ClassLoader;)Ljava/lang/Class;") \
  V(ClassGetPrimitiveClass, "Ljava/lang/Class;", "getPrimitiveClass", "(Ljava/lang/String;)Ljava/lang/Class;") \
  V(ClassClassForName, "Ljava/lang/Class;", "classForName", "(Ljava/lang/String;ZLjava/lang/ClassLoader;)Ljava/lang/Class;") \
  V(ClassNewInstance, "Ljava/lang/Class;", "newInstance", "()Ljava/lang/Object;") \
  V(ClassGetDeclaredField, "Ljava/lang/Class;", "getDeclaredField", "(Ljava/lang/String;)Ljava/lang/reflect/Field;") \
  V(ClassGetPublicDeclaredFields, "Ljava/lang/Class;", "getPublicDeclaredFields", "()[Ljava/lang/reflect/Field;") \
  V(ClassGetDeclaredFields, "Ljava/lang/Class;", "getDeclaredFields", "()[Ljava/lang/reflect/Field;") \
  V(ClassGetDeclaredMethod, "Ljava/lang/Class;", "getDeclaredMethodInternal", "(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;") \
  V(ClassGetDeclaredConstructor, "Ljava/lang/Class;", "getDeclaredConstructorInternal", "([Ljava/lang/Class;)Ljava/lang/reflect/Constructor;") \
  V(ClassGetDeclaringClass, "Ljava/lang/Class;", "getDeclaringClass", "()Ljava/lang/Class;") \
  V(ClassGetEnclosingClass, "Ljava/lang/Class;", "getEnclosingClass", "()Ljava/lang/Class;") \
  V(ClassGetInnerClassFlags, "Ljava/lang/Class;", "getInnerClassFlags", "(I)I") \
  V(ClassGetSignatureAnnotation, "Ljava/lang/Class;", "getSignatureAnnotation", "()[Ljava/lang/String;") \
  V(ClassIsAnonymousClass, "Ljava/lang/Class;", "isAnonymousClass", "()Z") \
  V(ClassLoaderGetResourceAsStream, "Ljava/lang/ClassLoader;", "getResourceAsStream", "(Ljava/lang/String;)Ljava/io/InputStream;") \
  V(ConstructorNewInstance0, "Ljava/lang/reflect/Constructor;", "newInstance0", "([Ljava/lang/Object;)Ljava/lang/Object;") \
  V(VmClassLoaderFindLoadedClass, "Ljava/lang/VMClassLoader;", "findLoadedClass", "(Ljava/lang/ClassLoader;Ljava/lang/String;)Ljava/lang/Class;") \
  V(SystemArraycopy, "Ljava/lang/System;", "arraycopy", "(Ljava/lang/Object;ILjava/lang/Object;II)V") \
  V(SystemArraycopyByte, "Ljava/lang/System;", "arraycopy", "([BI[BII)V") \
  V(SystemArraycopyChar, "Ljava/lang/System;", "arraycopy", "([CI[CII)V") \
  V(SystemArraycopyInt, "Ljava/lang/System;", "arraycopy", "([II[III)V") \
  V(SystemGetSecurityManager, "Ljava/lang/System;", "getSecurityManager", "()Ljava/lang/SecurityManager;") \
  V(SystemGetProperty, "Ljava/lang/System;", "getProperty", "(Ljava/lang/String;)Ljava/lang/String;") \
  V(SystemGetPropertyWithDefault, "Ljava/lang/System;", "getProperty", "(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;") \
  V(SystemNanoTime, "Ljava/lang/System;", "nanoTime", "()J") \
  V(ThreadLocalGet, "Ljava/lang/ThreadLocal;", "get", "()Ljava/lang/Object;") \
  V(MathCeil, "Ljava/lang/Math;", "ceil", "(D)D") \
  V(MathFloor, "Ljava/lang/Math;", "floor", "(D)D") \
  V(MathSin, "Ljava/lang/Math;", "sin", "(D)D") \
  V(MathCos, "Ljava/lang/Math;", "cos", "(D)D") \
  V(MathPow, "Ljava/lang/Math;", "pow", "(DD)D") \
  V(MathTan, "Ljava/lang/Math;", "tan", "(D)D") \
  V(ObjectHashCode, "Ljava/lang/Object;", "hashCode", "()I") \
  V(DoubleDoubleToRawLongBits, "Ljava/lang/Double;", "doubleToRawLongBits", "(D)J") \
  V(MemoryPeekByte, "Llibcore/io/Memory;", "peekByte", "(J)B") \
  V(MemoryPeekShort, "Llibcore/io/Memory;", "peekShortNative", "(J)S") \
  V(MemoryPeekInt, "Llibcore/io/Memory;", "peekIntNative", "(J)I") \
  V(MemoryPeekLong, "Llibcore/io/Memory;", "peekLongNative", "(J)J") \
  V(MemoryPeekByteArray, "Llibcore/io/Memory;", "peekByteArray", "(J[BII)V") \
  V(MethodInvoke, "Ljava/lang/reflect/Method;", "invoke", "(Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;") \
  V(ReferenceGetReferent, "Ljava/lang/ref/Reference;", "getReferent", "()Ljava/lang/Object;") \
  V(ReferenceRefersTo, "Ljava/lang/ref/Reference;", "refersTo", "(Ljava/lang/Object;)Z") \
  V(RuntimeAvailableProcessors, "Ljava/lang/Runtime;", "availableProcessors", "()I") \
  V(StringGetCharsNoCheck, "Ljava/lang/String;", "getCharsNoCheck", "(II[CI)V") \
  V(StringCharAt, "Ljava/lang/String;", "charAt", "(I)C") \
  V(StringDoReplace, "Ljava/lang/String;", "doReplace", "(CC)Ljava/lang/String;") \
  V(StringFactoryNewStringFromBytes, "Ljava/lang/StringFactory;", "newStringFromBytes", "([BIII)Ljava/lang/String;") \
  V(StringFactoryNewStringFromChars, "Ljava/lang/StringFactory;", "newStringFromChars", "(II[C)Ljava/lang/String;") \
  V(StringFactoryNewStringFromString, "Ljava/lang/StringFactory;", "newStringFromString", "(Ljava/lang/String;)Ljava/lang/String;") \
  V(StringFastSubstring, "Ljava/lang/String;", "fastSubstring", "(II)Ljava/lang/String;") \
  V(StringToCharArray, "Ljava/lang/String;", "toCharArray", "()[C") \
  V(ThreadCurrentThread, "Ljava/lang/Thread;", "currentThread", "()Ljava/lang/Thread;") \
  V(ThreadGetNativeState, "Ljava/lang/Thread;", "nativeGetStatus", "(Z)I") \
  V(UnsafeCompareAndSwapLong, "Lsun/misc/Unsafe;", "compareAndSwapLong", "(Ljava/lang/Object;JJJ)Z") \
  V(UnsafeCompareAndSwapObject, "Lsun/misc/Unsafe;", "compareAndSwapObject", "(Ljava/lang/Object;JLjava/lang/Object;Ljava/lang/Object;)Z") \
  V(UnsafeGetObjectVolatile, "Lsun/misc/Unsafe;", "getObjectVolatile", "(Ljava/lang/Object;J)Ljava/lang/Object;") \
  V(UnsafePutObjectVolatile, "Lsun/misc/Unsafe;", "putObjectVolatile", "(Ljava/lang/Object;JLjava/lang/Object;)V") \
  V(UnsafePutOrderedObject, "Lsun/misc/Unsafe;", "putOrderedObject", "(Ljava/lang/Object;JLjava/lang/Object;)V") \
  V(JdkUnsafeCompareAndSetLong, "Ljdk/internal/misc/Unsafe;", "compareAndSetLong", "(Ljava/lang/Object;JJJ)Z") \
  V(JdkUnsafeCompareAndSetReference, "Ljdk/internal/misc/Unsafe;", "compareAndSetReference", "(Ljava/lang/Object;JLjava/lang/Object;Ljava/lang/Object;)Z") \
  V(JdkUnsafeCompareAndSwapLong, "Ljdk/internal/misc/Unsafe;", "compareAndSwapLong", "(Ljava/lang/Object;JJJ)Z") \
  V(JdkUnsafeCompareAndSwapObject, "Ljdk/internal/misc/Unsafe;", "compareAndSwapObject", "(Ljava/lang/Object;JLjava/lang/Object;Ljava/lang/Object;)Z") \
  V(JdkUnsafeGetReferenceVolatile, "Ljdk/internal/misc/Unsafe;", "getReferenceVolatile", "(Ljava/lang/Object;J)Ljava/lang/Object;") \
  V(JdkUnsafePutReferenceVolatile, "Ljdk/internal/misc/Unsafe;", "putReferenceVolatile", "(Ljava/lang/Object;JLjava/lang/Object;)V") \
  V(JdkUnsafePutOrderedObject, "Ljdk/internal/misc/Unsafe;", "putOrderedObject", "(Ljava/lang/Object;JLjava/lang/Object;)V") \
  V(IntegerParseInt, "Ljava/lang/Integer;", "parseInt", "(Ljava/lang/String;)I") \
  V(LongParseLong, "Ljava/lang/Long;", "parseLong", "(Ljava/lang/String;)J") \
  V(SystemIdentityHashCode, "Ljava/lang/System;", "identityHashCode", "(Ljava/lang/Object;)I")

// Methods that are native.
#define UNSTARTED_RUNTIME_JNI_LIST(V)           \
  V(VMRuntimeIs64Bit, "Ldalvik/system/VMRuntime;", "is64Bit", "()Z") \
  V(VMRuntimeNewUnpaddedArray, "Ldalvik/system/VMRuntime;", "newUnpaddedArray", "(Ljava/lang/Class;I)Ljava/lang/Object;") \
  V(VMStackGetCallingClassLoader, "Ldalvik/system/VMStack;", "getCallingClassLoader", "()Ljava/lang/ClassLoader;") \
  V(VMStackGetStackClass2, "Ldalvik/system/VMStack;", "getStackClass2", "()Ljava/lang/Class;") \
  V(MathLog, "Ljava/lang/Math;", "log", "(D)D") \
  V(MathExp, "Ljava/lang/Math;", "exp", "(D)D") \
  V(AtomicLongVMSupportsCS8, "Ljava/util/concurrent/atomic/AtomicLong;", "VMSupportsCS8", "()Z") \
  V(ClassGetNameNative, "Ljava/lang/Class;", "getNameNative", "()Ljava/lang/String;") \
  V(DoubleLongBitsToDouble, "Ljava/lang/Double;", "longBitsToDouble", "(J)D") \
  V(ExecutableGetParameterTypesInternal, "Ljava/lang/reflect/Executable;", "getParameterTypesInternal", "()[Ljava/lang/Class;") \
  V(FloatFloatToRawIntBits, "Ljava/lang/Float;", "floatToRawIntBits", "(F)I") \
  V(FloatIntBitsToFloat, "Ljava/lang/Float;", "intBitsToFloat", "(I)F") \
  V(ObjectInternalClone, "Ljava/lang/Object;", "internalClone", "()Ljava/lang/Object;") \
  V(ObjectNotifyAll, "Ljava/lang/Object;", "notifyAll", "()V") \
  V(StringCompareTo, "Ljava/lang/String;", "compareTo", "(Ljava/lang/String;)I") \
  V(StringFillBytesLatin1, "Ljava/lang/String;", "fillBytesLatin1", "([BI)V") \
  V(StringFillBytesUTF16, "Ljava/lang/String;", "fillBytesUTF16", "([BI)V") \
  V(StringIntern, "Ljava/lang/String;", "intern", "()Ljava/lang/String;") \
  V(ArrayCreateMultiArray, "Ljava/lang/reflect/Array;", "createMultiArray", "(Ljava/lang/Class;[I)Ljava/lang/Object;") \
  V(ArrayCreateObjectArray, "Ljava/lang/reflect/Array;", "createObjectArray", "(Ljava/lang/Class;I)Ljava/lang/Object;") \
  V(ThrowableNativeFillInStackTrace, "Ljava/lang/Throwable;", "nativeFillInStackTrace", "()Ljava/lang/Object;") \
  V(UnsafeCompareAndSwapInt, "Lsun/misc/Unsafe;", "compareAndSwapInt", "(Ljava/lang/Object;JII)Z") \
  V(UnsafeGetIntVolatile, "Lsun/misc/Unsafe;", "getIntVolatile", "(Ljava/lang/Object;J)I") \
  V(UnsafePutObject, "Lsun/misc/Unsafe;", "putObject", "(Ljava/lang/Object;JLjava/lang/Object;)V") \
  V(UnsafeGetArrayBaseOffsetForComponentType, "Lsun/misc/Unsafe;", "getArrayBaseOffsetForComponentType", "(Ljava/lang/Class;)I") \
  V(UnsafeGetArrayIndexScaleForComponentType, "Lsun/misc/Unsafe;", "getArrayIndexScaleForComponentType", "(Ljava/lang/Class;)I") \
  V(JdkUnsafeAddressSize, "Ljdk/internal/misc/Unsafe;", "addressSize", "()I") \
  V(JdkUnsafeCompareAndSetInt, "Ljdk/internal/misc/Unsafe;", "compareAndSetInt", "(Ljava/lang/Object;JII)Z") \
  V(JdkUnsafeCompareAndSwapInt, "Ljdk/internal/misc/Unsafe;", "compareAndSwapInt", "(Ljava/lang/Object;JII)Z") \
  V(JdkUnsafeGetIntVolatile, "Ljdk/internal/misc/Unsafe;", "getIntVolatile", "(Ljava/lang/Object;J)I") \
  V(JdkUnsafePutReference, "Ljdk/internal/misc/Unsafe;", "putReference", "(Ljava/lang/Object;JLjava/lang/Object;)V") \
  V(JdkUnsafeStoreFence, "Ljdk/internal/misc/Unsafe;", "storeFence", "()V") \
  V(JdkUnsafeGetArrayBaseOffsetForComponentType, "Ljdk/internal/misc/Unsafe;", "getArrayBaseOffsetForComponentType", "(Ljava/lang/Class;)I") \
  V(JdkUnsafeGetArrayIndexScaleForComponentType, "Ljdk/internal/misc/Unsafe;", "getArrayIndexScaleForComponentType", "(Ljava/lang/Class;)I") \
  V(FieldGetArtField, "Ljava/lang/reflect/Field;", "getArtField", "()J") \
  V(FieldGetNameInternal, "Ljava/lang/reflect/Field;", "getNameInternal", "()Ljava/lang/String;")
```

### 代码生成

在 `unstarted_runtime.h` 中，利用这两个列表在**编译时**自动声明了所有处理函数的原型。

```cpp
 // Methods that intercept available libcore implementations.
#define UNSTARTED_DIRECT(ShortName, DescriptorIgnored, NameIgnored, SignatureIgnored) \
  static void Unstarted ## ShortName(Thread* self,                                    \
                                     ShadowFrame* shadow_frame,                       \
                                     JValue* result,                                  \
                                     size_t arg_offset)                               \
      REQUIRES_SHARED(Locks::mutator_lock_);
  UNSTARTED_RUNTIME_DIRECT_LIST(UNSTARTED_DIRECT)
#undef UNSTARTED_DIRECT

  // Methods that are native.
#define UNSTARTED_JNI(ShortName, DescriptorIgnored, NameIgnored, SignatureIgnored) \
  static void UnstartedJNI ## ShortName(Thread* self,                              \
                                        ArtMethod* method,                         \
                                        mirror::Object* receiver,                  \
                                        uint32_t* args,                            \
                                        JValue* result)                            \
      REQUIRES_SHARED(Locks::mutator_lock_);
  UNSTARTED_RUNTIME_JNI_LIST(UNSTARTED_JNI)
#undef UNSTARTED_JNI
```

### 初始化

当 `UnstartedRuntime::Initialize()`被调用时，它会：

- 遍历 `UNSTARTED_RUNTIME_DIRECT_LIST` 和 `UNSTARTED_RUNTIME_JNI_LIST`。
- 通过 `FindMethod` 找到每个Java方法对应的 `ArtMethod*` 指针。
- 将 `ArtMethod*` 作为Key，将对应的C++实现函数（如 `&UnstartedRuntime::UnstartedCharacterToLowerCase`）的指针作为Value，存入两个静态的 `HashMap` 中`invoke_handlers_` 和 `jni_handlers_`。

```cpp
void UnstartedRuntime::InitializeInvokeHandlers(Thread* self) {
  ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
#define UNSTARTED_DIRECT(ShortName, Descriptor, Name, Signature) \
  { \
    ArtMethod* method = FindMethod(self, class_linker, Descriptor, Name, Signature); \
    invoke_handlers_.insert(std::make_pair(method, & UnstartedRuntime::Unstarted ## ShortName)); \
  }
  UNSTARTED_RUNTIME_DIRECT_LIST(UNSTARTED_DIRECT)
#undef UNSTARTED_DIRECT
  DCHECK_EQ(invoke_handlers_.NumBuckets(), kInvokeHandlersBufferSize);
}

void UnstartedRuntime::InitializeJNIHandlers(Thread* self) {
  ClassLinker* class_linker = Runtime::Current()->GetClassLinker();
#define UNSTARTED_JNI(ShortName, Descriptor, Name, Signature) \
  { \
    ArtMethod* method = FindMethod(self, class_linker, Descriptor, Name, Signature); \
    jni_handlers_.insert(std::make_pair(method, & UnstartedRuntime::UnstartedJNI ## ShortName)); \
  }
  UNSTARTED_RUNTIME_JNI_LIST(UNSTARTED_JNI)
#undef UNSTARTED_JNI
  DCHECK_EQ(jni_handlers_.NumBuckets(), kJniHandlersBufferSize);
}
```

### 拦截与执行

- 当`dex2oat`中的解释器要执行一个Java方法时，它会调用 `UnstartedRuntime::Invoke()`。
- 这个函数会在`invoke_handlers_`中查找当前要执行的 `ArtMethod*`。
  - **如果找到**：就调用对应的C++“山寨”实现。
  - **如果没找到**：调用`ArtInterpreterToInterpreterBridge`尝试继续使用解释器执行。

```cpp
void UnstartedRuntime::Invoke(Thread* self, const CodeItemDataAccessor& accessor,
                              ShadowFrame* shadow_frame, JValue* result, size_t arg_offset) {
  // In a runtime that's not started we intercept certain methods to avoid complicated dependency
  // problems in core libraries.
  CHECK(tables_initialized_);

  const auto& iter = invoke_handlers_.find(shadow_frame->GetMethod());
  if (iter != invoke_handlers_.end()) {
    // Note: When we special case the method, we do not ensure initialization.
    // This has been the behavior since implementation of this feature.

    // Clear out the result in case it's not zeroed out.
    result->SetL(nullptr);

    // Push the shadow frame. This is so the failing method can be seen in abort dumps.
    self->PushShadowFrame(shadow_frame);

    (*iter->second)(self, shadow_frame, result, arg_offset);

    self->PopShadowFrame();
  } else {
    if (!EnsureInitialized(self, shadow_frame)) {
      return;
    }
    // Not special, continue with regular interpreter execution.
    ArtInterpreterToInterpreterBridge(self, accessor, shadow_frame, result);
  }
}

```

进入`ArtInterpreterToInterpreterBridge`之后，当要执行一个`native`方法时，它会调用 `UnstartedRuntime::Jni()`。因为除非测试和生成镜像文件这两种情况，我们通常不应该用解释器去执行 native 代码，它一般是通过 JNI 编译器生成的stub函数进入的。所以使用 `UnstartedRuntime::Jni()`处理。

```cpp
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

这个函数会在`jni_handlers_`中查找当前要执行的 `ArtMethod*`

```cpp

// Hand select a number of methods to be run in a not yet started runtime without using JNI.
void UnstartedRuntime::Jni(Thread* self, ArtMethod* method, mirror::Object* receiver,
                           uint32_t* args, JValue* result) {
  const auto& iter = jni_handlers_.find(method);
  if (iter != jni_handlers_.end()) {
    // Clear out the result in case it's not zeroed out.
    result->SetL(nullptr);
    (*iter->second)(self, method, receiver, args, result);
  } else {
    Runtime* runtime = Runtime::Current();
    if (runtime->IsActiveTransaction()) {
      runtime->GetClassLinker()->AbortTransactionF(
          self,
          "Attempt to invoke native method in non-started runtime: %s",
          ArtMethod::PrettyMethod(method).c_str());
    } else {
      LOG(FATAL) << "Calling native method " << ArtMethod::PrettyMethod(method)
                 << " in an unstarted non-transactional runtime";
    }
  }
}
```

- **如果找到**：就调用对应的C++“山寨”实现。
- **如果没找到**：对于`Jni`，则会认为这是一个不支持的操作，报错。

### 主要“山寨”实现 (`Unstarted...` 系列函数)

这些函数是 `UnstartedRuntime` 的精髓，它们提供了对核心Java API的有限模拟。

- **`UnstartedClass...` 系列 (如 `UnstartedClassForName`, `UnstartedClassNewInstance`)**:
    **作用**：提供了对 `java.lang.Class` 反射功能的最基本支持。它们允许在没有完整运行时的情况下查找类、创建实例（仅限无参构造函数）、查找字段和方法。这是许多类初始化所必需的。

- **`UnstartedSystem...` 系列 (如 `UnstartedSystemArraycopy`, `UnstartedSystemGetProperty`)**:
    **作用**：模拟 `java.lang.System` 的部分功能。
  - `UnstartedSystemArraycopy` 提供了一个安全的、不依赖底层`memcpy`优化的数组复制实现。
  - `UnstartedSystemGetProperty` 从一个硬编码的列表 (`AndroidHardcodedSystemProperties`) 中查找系统属性，而不是真正地查询操作系统。
  - `UnstartedSystemNanoTime` 则直接中止事务，因为它无法在编译时提供一个有意义且不重复的时间戳，这对于`java.util.Random`等类的初始化至关重要。

- **`UnstartedUnsafe...` 和 `UnstartedJdkUnsafe...` 系列**:
    **作用**：提供了对 `sun.misc.Unsafe` 和 `jdk.internal.misc.Unsafe` 中原子操作（如CAS）的支持。这对于初始化 `java.util.concurrent` 包下的类（如`ConcurrentHashMap`）是必需的。这些C++实现直接映射到底层的原子CPU指令。

- **`UnstartedString...` 系列 (如 `UnstartedStringFactoryNewStringFromChars`, `UnstartedStringCharAt`)**:
    **作用**：支持在编译时创建和操作字符串对象。这对于几乎所有类的初始化都至关重要。

- **`UnstartedRuntimeAvailableProcessors`**:
    **作用**：处理 `Runtime.getRuntime().availableProcessors()`。它不查询真实的CPU核心数，而是根据调用者的上下文（如`ConcurrentHashMap`的初始化），返回一个保守的硬编码值（例如8）。

- **`AbortTransactionOrFail` (静态辅助函数)**:
    **作用**：核心的“失败处理”机制。当遇到无法处理的情况时调用，用于通知`ClassLinker`中止当前类的初始化事务，并将初始化推迟到运行时。
