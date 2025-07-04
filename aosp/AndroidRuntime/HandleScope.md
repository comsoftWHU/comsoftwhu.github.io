---
layout: default
title: ART的C++代码对Java堆对象的访问--HandleScope
nav_order: 1
parent: AndroidRuntime
grand_parent: AOSP
author: Anonymous Committer
---


# HandleScope

`HandleScope` 的核心作用是**在 ART 的 C++ Native 代码中安全地管理对 Java 对象的引用，确保这些对象不会被GC错误地回收**。

## 问题

GC 通过追踪从根 (GCRoot)开始的所有可达对象来判断哪些对象是“存活”的。

当 ART 自身的 C++ 代码需要操作一个 Java 对象时，它会持有一个指向该对象的指针。但 GC 并不知道这些 C++ 代码中的“裸指针”。如果在 GC 运行时，它认为没有任何 Java 代码引用这个对象，就会将其回收。此时，C++ 代码中的指针就变成了“野指针”，再次使用会导致程序崩溃。

## 解决方案

### Handle
ART 引入了 `Handle` 的概念。一个 `Handle` 是一个间接的、GC 可知的引用。当 GC 运行时，它会扫描所有活跃的 `Handle`，将它们指向的对象标记为存活。如果 GC 移动了对象，它还会**自动更新 `Handle` 中存储的地址**。


### HandleScope

`Handle` 解决了安全引用的问题，但它们的生命周期管理又成了新问题。如果忘记释放一个不再需要的 `Handle`，它所引用的对象将永远无法被回收，导致内存泄漏。各种 `HandleScope` 就是用来**自动化管理 `Handle` 生命周期**的机制。

### 工作流程


1.  **进入作用域**：当一段 C++ 代码需要操作 Java 对象时，它会在栈上创建一个 `HandleScope` 对象。
2.  **创建句柄**：通过 `scope.NewHandle(obj)` 在该作用域内为目标对象创建一个 `Handle`。
3.  **安全操作**：在 `HandleScope` 的生命周期内，所有通过它创建的 `Handle` 都是有效的 GC Roots，它们引用的对象是绝对安全的。
4.  **离开作用域**：当 C++ 代码块执行完毕，栈上的 `HandleScope` 对象被自动销毁。在其析构函数中，它会**自动释放其管理的所有 `Handle`**，解除对这些对象的保护。

## Code

### Handle

Handle 内部只有一个成员reference_，为StackReference的指针，当 GC 移动了堆上的 `Object` 时，它只需要找到并更新栈上的 `StackReference` 的值即可。而 `Handle` 和 C++ 代码本身持有的指针地址完全不受影响，从而实现了对 GC 的透明。

```cpp
template<class T>
class Handle : public ValueObject {
  ......
  StackReference<mirror::Object>* reference_;
  ......
};
```


MutableHandle可以改变reference_指向的StackReference中存储的指针

```cpp
// Handles that support assignment.
template<class T>
class MutableHandle : public Handle<T> {
 public:
  ALWAYS_INLINE T* Assign(T* reference) REQUIRES_SHARED(Locks::mutator_lock_) {
    StackReference<mirror::Object>* ref = Handle<T>::GetReference();
    T* old = down_cast<T*>(ref->AsMirrorPtr());
    ref->Assign(reference);
    return old;
  }

  ALWAYS_INLINE T* Assign(ObjPtr<T> reference) REQUIRES_SHARED(Locks::mutator_lock_) {
    StackReference<mirror::Object>* ref = Handle<T>::GetReference();
    T* old = down_cast<T*>(ref->AsMirrorPtr());
    ref->Assign(reference.Ptr());
    return old;
  }
};
```

### HandleScope

Basic handle scope 组织结构为链表。包含一些必要的统计的字段，但是没有实际的存储的位置

这个是为了后面给 FixedSizeHandleScope VariableSizedHandleScope 提供一个统一的接口

```cpp
// Basic handle scope, tracked by a list. May be variable sized.
class PACKED(4) BaseHandleScope {

  static constexpr int32_t kNumReferencesVariableSized = -1;

  // Link-list of handle scopes. The root is held by a Thread.
  BaseHandleScope* const link_;

  // Number of handlerized references. -1 for variable sized handle scopes.
  const int32_t capacity_;
};
```

HandleScope在BaseHandleScope后面多一个uint32_t成员size_，但是没有实际存储reference的地方。
提供了一个GetReferences方法，返回对象基地址 + sizeof(link_) + sizeof(capacity_) + sizeof(size_),然后把这个地址转换成 `StackReference<mirror::Object>*`

```cpp
class PACKED(4) HandleScope : public BaseHandleScope {
 public:

  static constexpr size_t ReferencesOffset(PointerSize pointer_size) {
    return CapacityOffset(pointer_size) + sizeof(capacity_) + sizeof(size_);
  }

  ALWAYS_INLINE StackReference<mirror::Object>* GetReferences() const {
    uintptr_t address = reinterpret_cast<uintptr_t>(this) + ReferencesOffset(kRuntimePointerSize);
    return reinterpret_cast<StackReference<mirror::Object>*>(address);
  }
  // Position new handles will be created.
  uint32_t size_ = 0;

  // Storage for references is in derived classes.
  // StackReference<mirror::Object> references_[capacity_]
};
```

FixedSizeHandleScope在后面跟了一个固定大小的 `StackReference<mirror::Object>` 数组，storage_可以直接分配在栈上，在FixedSizeHandleScope被析构的时候自动被释放

```cpp
// Fixed size handle scope that is not necessarily linked in the thread.
template<size_t kNumReferences>
class PACKED(4) FixedSizeHandleScope : public HandleScope {
 private:
  explicit ALWAYS_INLINE FixedSizeHandleScope(BaseHandleScope* link)
      REQUIRES_SHARED(Locks::mutator_lock_);
  ALWAYS_INLINE ~FixedSizeHandleScope() REQUIRES_SHARED(Locks::mutator_lock_) {}

  // Reference storage.
  StackReference<mirror::Object> storage_[kNumReferences];

  template<size_t kNumRefs> friend class StackHandleScope;
  friend class VariableSizedHandleScope;
};
```
StackHandleScope 在 FixedSizeHandleScope 基础上增加了  Thread*成员，这是因为创建和释放的时候需要更新线程局部存储的 tlsPtr_.top_handle_scope ，

```cpp
// Scoped handle storage of a fixed size that is stack allocated.
template<size_t kNumReferences>
class PACKED(4) StackHandleScope final : public FixedSizeHandleScope<kNumReferences> {
 public:
  template<size_t kNumReferences>
  inline StackHandleScope<kNumReferences>::StackHandleScope(Thread* self)
      : FixedSizeHandleScope<kNumReferences>(self->GetTopHandleScope()),
        self_(self) {
    DCHECK_EQ(self, Thread::Current());
    if (kDebugLocking) {
      Locks::mutator_lock_->AssertSharedHeld(self_);
    }
    self_->PushHandleScope(this);
  }    

  ALWAYS_INLINE ~StackHandleScope() REQUIRES_SHARED(Locks::mutator_lock_);

  Thread* Self() const {
    return self_;
  }

 private:
  // The thread that the stack handle scope is a linked list upon. The stack handle scope will
  // push and pop itself from this thread.
  Thread* const self_;
};
```

VariableSizedHandleScope 是 FixedSizeHandleScope 的链表


```cpp

// Utility class to manage a variable sized handle scope by having a list of fixed size handle
// scopes.
// Calls to NewHandle will create a new handle inside the current FixedSizeHandleScope.
// When the current handle scope becomes full a new one is created and put at the front of the
// list.
class VariableSizedHandleScope : public BaseHandleScope {
  static constexpr size_t kLocalScopeSize = 64u;
  static constexpr size_t kSizeOfReferencesPerScope =
      kLocalScopeSize
          - /* BaseHandleScope::link_ */ sizeof(BaseHandleScope*)
          - /* BaseHandleScope::capacity_ */ sizeof(int32_t)
          - /* HandleScope<>::size_ */ sizeof(uint32_t);
  static constexpr size_t kNumReferencesPerScope =
      kSizeOfReferencesPerScope / sizeof(StackReference<mirror::Object>);

  Thread* const self_;

  // Linked list of fixed size handle scopes.
  using LocalScopeType = FixedSizeHandleScope<kNumReferencesPerScope>;
  static_assert(sizeof(LocalScopeType) == kLocalScopeSize, "Unexpected size of LocalScopeType");
  LocalScopeType* current_scope_;
  LocalScopeType first_scope_;

  DISALLOW_COPY_AND_ASSIGN(VariableSizedHandleScope);
};

```

### 遍历Handle

HandleScope 提供了VisitRoots和VisitHandles 2个使用 **访问者模式**的接口


```cpp
template <typename Visitor>
inline void HandleScope::VisitRoots(Visitor& visitor) {
  for (size_t i = 0, size = Size(); i < size; ++i) {
    // GetReference returns a pointer to the stack reference within the handle scope. If this
    // needs to be updated, it will be done by the root visitor.
    visitor.VisitRootIfNonNull(GetHandle<mirror::Object>(i).GetReference());
  }
}

template <typename Visitor>
inline void HandleScope::VisitHandles(Visitor& visitor) {
  for (size_t i = 0, size = Size(); i < size; ++i) {
    if (GetHandle<mirror::Object>(i) != nullptr) {
      visitor.Visit(GetHandle<mirror::Object>(i));
    }
  }
}
```

GC MarkingPhase的阶段会对每个线程调用 Thread::VisitRoots 方法,其中会HandleScopeVisitRoots(visitor, thread_id)


```cpp

// FIXME: clang-r433403 reports the below function exceeds frame size limit.
// http://b/197647048
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wframe-larger-than="
template <bool kPrecise>
void Thread::VisitRoots(RootVisitor* visitor) {
  const uint32_t thread_id = GetThreadId();
  visitor->VisitRootIfNonNull(&tlsPtr_.opeer, RootInfo(kRootThreadObject, thread_id));
  if (tlsPtr_.exception != nullptr && tlsPtr_.exception != GetDeoptimizationException()) {
    visitor->VisitRoot(reinterpret_cast<mirror::Object**>(&tlsPtr_.exception),
                       RootInfo(kRootNativeStack, thread_id));
  }
  if (tlsPtr_.async_exception != nullptr) {
    visitor->VisitRoot(reinterpret_cast<mirror::Object**>(&tlsPtr_.async_exception),
                       RootInfo(kRootNativeStack, thread_id));
  }
  visitor->VisitRootIfNonNull(&tlsPtr_.monitor_enter_object, RootInfo(kRootNativeStack, thread_id));
  tlsPtr_.jni_env->VisitJniLocalRoots(visitor, RootInfo(kRootJNILocal, thread_id));
  tlsPtr_.jni_env->VisitMonitorRoots(visitor, RootInfo(kRootJNIMonitor, thread_id));
  HandleScopeVisitRoots(visitor, thread_id);
  // Visit roots for deoptimization.
  if (tlsPtr_.stacked_shadow_frame_record != nullptr) {
    RootCallbackVisitor visitor_to_callback(visitor, thread_id);
    ReferenceMapVisitor<RootCallbackVisitor, kPrecise> mapper(this, nullptr, visitor_to_callback);
    for (StackedShadowFrameRecord* record = tlsPtr_.stacked_shadow_frame_record;
         record != nullptr;
         record = record->GetLink()) {
      for (ShadowFrame* shadow_frame = record->GetShadowFrame();
           shadow_frame != nullptr;
           shadow_frame = shadow_frame->GetLink()) {
        mapper.VisitShadowFrame(shadow_frame);
      }
    }
  }
  for (DeoptimizationContextRecord* record = tlsPtr_.deoptimization_context_stack;
       record != nullptr;
       record = record->GetLink()) {
    if (record->IsReference()) {
      visitor->VisitRootIfNonNull(record->GetReturnValueAsGCRoot(),
                                  RootInfo(kRootThreadObject, thread_id));
    }
    visitor->VisitRootIfNonNull(record->GetPendingExceptionAsGCRoot(),
                                RootInfo(kRootThreadObject, thread_id));
  }
  if (tlsPtr_.frame_id_to_shadow_frame != nullptr) {
    RootCallbackVisitor visitor_to_callback(visitor, thread_id);
    ReferenceMapVisitor<RootCallbackVisitor, kPrecise> mapper(this, nullptr, visitor_to_callback);
    for (FrameIdToShadowFrame* record = tlsPtr_.frame_id_to_shadow_frame;
         record != nullptr;
         record = record->GetNext()) {
      mapper.VisitShadowFrame(record->GetShadowFrame());
    }
  }
  // Visit roots on this thread's stack
  RuntimeContextType context;
  RootCallbackVisitor visitor_to_callback(visitor, thread_id);
  ReferenceMapVisitor<RootCallbackVisitor, kPrecise> mapper(this, &context, visitor_to_callback);
  mapper.template WalkStack<StackVisitor::CountTransitions::kNo>(false);
}
#pragma GCC diagnostic pop


void Thread::HandleScopeVisitRoots(RootVisitor* visitor, uint32_t thread_id) {
  BufferedRootVisitor<kDefaultBufferedRootCount> buffered_visitor(
      visitor, RootInfo(kRootNativeStack, thread_id));
  for (BaseHandleScope* cur = tlsPtr_.top_handle_scope; cur; cur = cur->GetLink()) {
    cur->VisitRoots(buffered_visitor);
  }
}

```

我们看到这里实际上是转换成了BaseHandleScope的指针，但是BaseHandleScope并没有size成员，BaseHandleScope并不知道自己有多少个有效的handle


```cpp
template <typename Visitor>
inline void BaseHandleScope::VisitRoots(Visitor& visitor) {
  if (LIKELY(!IsVariableSized())) {
    AsHandleScope()->VisitRoots(visitor);
  } else {
    AsVariableSized()->VisitRoots(visitor);
  }
}
```

他这里根据是否IsVariableSized，会把this 使用 down_cast 转换成 VariableSizedHandleScope 或者 HandleScope类型的指针。

```cpp
template <typename Visitor>
inline void BaseHandleScope::VisitHandles(Visitor& visitor) {
  if (LIKELY(!IsVariableSized())) {
    AsHandleScope()->VisitHandles(visitor);
  } else {
    AsVariableSized()->VisitHandles(visitor);
  }
}

inline VariableSizedHandleScope* BaseHandleScope::AsVariableSized() {
  DCHECK(IsVariableSized());
  return down_cast<VariableSizedHandleScope*>(this);
}

inline HandleScope* BaseHandleScope::AsHandleScope() {
  DCHECK(!IsVariableSized());
  return down_cast<HandleScope*>(this);
}

```

### down_cast的问题

down_cast 并不是C++的标准，在AOSP 的实现中 会使用 C++ 11 的**类型萃取**特性，先判断To是不是From的子类

<https://en.cppreference.com/w/cpp/header/type_traits.html>


```cpp
template<typename To, typename From>
inline To down_cast(From* f) {
  // This is the important compile-time safety check
  static_assert(std::is_base_of_v<From, std::remove_pointer_t<To>>,
                "down_cast unsafe as To is not a subtype of From");

  // This is the actual cast being performed
  return static_cast<To>(f);
}
```

但是虽然加了std::is_base_of_v编译时检查，但是仍然不能查找出所有的潜在风险，因为这个操作在运行时仍然允许把一个父类的指针转换成子类的指针，在用实际指向的父类指针访问子类的成员时会导致未定义的行为。
