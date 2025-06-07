---
layout: default
title: C++多线程编程&AOSP环境下的多线程
nav_order: 1
parent: AndroidRuntime
grand_parent: AOSP
author: Anonymous Committer
---

# C++多线程编程&AOSP环境下的多线程

<!-- TOC -->

- [C++多线程编程&AOSP环境下的多线程](#c%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BC%96%E7%A8%8Baosp%E7%8E%AF%E5%A2%83%E4%B8%8B%E7%9A%84%E5%A4%9A%E7%BA%BF%E7%A8%8B)
    - [C++ 多线程基础](#c-%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%9F%BA%E7%A1%80)
        - [线程概念](#%E7%BA%BF%E7%A8%8B%E6%A6%82%E5%BF%B5)
        - [std::thread 的创建与管理](#stdthread-%E7%9A%84%E5%88%9B%E5%BB%BA%E4%B8%8E%E7%AE%A1%E7%90%86)
        - [线程参数和返回值](#%E7%BA%BF%E7%A8%8B%E5%8F%82%E6%95%B0%E5%92%8C%E8%BF%94%E5%9B%9E%E5%80%BC)
        - [线程的生命周期](#%E7%BA%BF%E7%A8%8B%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F)
    - [同步原语](#%E5%90%8C%E6%AD%A5%E5%8E%9F%E8%AF%AD)
        - [互斥量（std::mutex）与锁类型](#%E4%BA%92%E6%96%A5%E9%87%8Fstdmutex%E4%B8%8E%E9%94%81%E7%B1%BB%E5%9E%8B)
            - [死锁示例](#%E6%AD%BB%E9%94%81%E7%A4%BA%E4%BE%8B)
        - [条件变量（std::condition_variable）](#%E6%9D%A1%E4%BB%B6%E5%8F%98%E9%87%8Fstdcondition_variable)
            - [wait](#wait)
            - [notify_one](#notify_one)
            - [notify_all](#notify_all)
            - [小结](#%E5%B0%8F%E7%BB%93)
        - [原子操作（std::atomic）](#%E5%8E%9F%E5%AD%90%E6%93%8D%E4%BD%9Cstdatomic)
            - [支持的类型](#%E6%94%AF%E6%8C%81%E7%9A%84%E7%B1%BB%E5%9E%8B)
            - [构造与赋值接口](#%E6%9E%84%E9%80%A0%E4%B8%8E%E8%B5%8B%E5%80%BC%E6%8E%A5%E5%8F%A3)
            - [赋值运算符与拷贝/移动](#%E8%B5%8B%E5%80%BC%E8%BF%90%E7%AE%97%E7%AC%A6%E4%B8%8E%E6%8B%B7%E8%B4%9D%E7%A7%BB%E5%8A%A8)
            - [基本读写接口：load / store](#%E5%9F%BA%E6%9C%AC%E8%AF%BB%E5%86%99%E6%8E%A5%E5%8F%A3load--store)
            - [读—改—写接口（RMW）](#%E8%AF%BB%E6%94%B9%E5%86%99%E6%8E%A5%E5%8F%A3rmw)
            - [示例与注意事项](#%E7%A4%BA%E4%BE%8B%E4%B8%8E%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9)
        - [读写锁（std::shared_mutex / std::shared_timed_mutex）](#%E8%AF%BB%E5%86%99%E9%94%81stdshared_mutex--stdshared_timed_mutex)
        - [其他同步工具](#%E5%85%B6%E4%BB%96%E5%90%8C%E6%AD%A5%E5%B7%A5%E5%85%B7)
            - [std::promise / std::future / std::packaged_task](#stdpromise--stdfuture--stdpackaged_task)
            - [线程局部存储（Thread-Local Storage）](#%E7%BA%BF%E7%A8%8B%E5%B1%80%E9%83%A8%E5%AD%98%E5%82%A8thread-local-storage)
    - [执行模型（Execution Model）](#%E6%89%A7%E8%A1%8C%E6%A8%A1%E5%9E%8Bexecution-model)
        - [单线程执行流程](#%E5%8D%95%E7%BA%BF%E7%A8%8B%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B)
        - [多线程执行流程](#%E5%A4%9A%E7%BA%BF%E7%A8%8B%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B)
            - [同步先行（Happens-Before）关系](#%E5%90%8C%E6%AD%A5%E5%85%88%E8%A1%8Chappens-before%E5%85%B3%E7%B3%BB)
        - [线程与可见性、重排序](#%E7%BA%BF%E7%A8%8B%E4%B8%8E%E5%8F%AF%E8%A7%81%E6%80%A7%E9%87%8D%E6%8E%92%E5%BA%8F)
    - [内存模型（Memory Model）](#%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8Bmemory-model)
        - [数据竞争（Data Race）与未定义行为](#%E6%95%B0%E6%8D%AE%E7%AB%9E%E4%BA%89data-race%E4%B8%8E%E6%9C%AA%E5%AE%9A%E4%B9%89%E8%A1%8C%E4%B8%BA)
        - [内存序（Memory Order）分类](#%E5%86%85%E5%AD%98%E5%BA%8Fmemory-order%E5%88%86%E7%B1%BB)
        - [顺序一致性（Sequential Consistency）](#%E9%A1%BA%E5%BA%8F%E4%B8%80%E8%87%B4%E6%80%A7sequential-consistency)
        - [获取-释放（Acquire-Release）语义](#%E8%8E%B7%E5%8F%96-%E9%87%8A%E6%94%BEacquire-release%E8%AF%AD%E4%B9%89)
        - [松散内存序（Relaxed Ordering）](#%E6%9D%BE%E6%95%A3%E5%86%85%E5%AD%98%E5%BA%8Frelaxed-ordering)
        - [内存屏障（Barrier / Fence）](#%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9Cbarrier--fence)
        - [“释放-一致性” 总结](#%E9%87%8A%E6%94%BE-%E4%B8%80%E8%87%B4%E6%80%A7-%E6%80%BB%E7%BB%93)
    - [常见模式与实践](#%E5%B8%B8%E8%A7%81%E6%A8%A1%E5%BC%8F%E4%B8%8E%E5%AE%9E%E8%B7%B5)
        - [双重检查锁（Double-Checked Locking）](#%E5%8F%8C%E9%87%8D%E6%A3%80%E6%9F%A5%E9%94%81double-checked-locking)
        - [生产者-消费者模式](#%E7%94%9F%E4%BA%A7%E8%80%85-%E6%B6%88%E8%B4%B9%E8%80%85%E6%A8%A1%E5%BC%8F)
        - [线程池（Thread Pool）](#%E7%BA%BF%E7%A8%8B%E6%B1%A0thread-pool)
        - [并发队列（Concurrent Queue）](#%E5%B9%B6%E5%8F%91%E9%98%9F%E5%88%97concurrent-queue)
        - [读者-写者锁模式](#%E8%AF%BB%E8%80%85-%E5%86%99%E8%80%85%E9%94%81%E6%A8%A1%E5%BC%8F)
        - [原子升级（Atomic Upgrade）与 ABA 问题](#%E5%8E%9F%E5%AD%90%E5%8D%87%E7%BA%A7atomic-upgrade%E4%B8%8E-aba-%E9%97%AE%E9%A2%98)
    - [调试与性能优化](#%E8%B0%83%E8%AF%95%E4%B8%8E%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96)
        - [数据竞争检测（Thread Sanitizer）](#%E6%95%B0%E6%8D%AE%E7%AB%9E%E4%BA%89%E6%A3%80%E6%B5%8Bthread-sanitizer)
        - [内存屏障性能影响](#%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C%E6%80%A7%E8%83%BD%E5%BD%B1%E5%93%8D)
        - [伪共享（False Sharing）与对齐优化](#%E4%BC%AA%E5%85%B1%E4%BA%ABfalse-sharing%E4%B8%8E%E5%AF%B9%E9%BD%90%E4%BC%98%E5%8C%96)
        - [原子 vs 互斥对比](#%E5%8E%9F%E5%AD%90-vs-%E4%BA%92%E6%96%A5%E5%AF%B9%E6%AF%94)
    - [示例与案例分析](#%E7%A4%BA%E4%BE%8B%E4%B8%8E%E6%A1%88%E4%BE%8B%E5%88%86%E6%9E%90)
        - [简单的多线程加法示例](#%E7%AE%80%E5%8D%95%E7%9A%84%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%8A%A0%E6%B3%95%E7%A4%BA%E4%BE%8B)
        - [发布-订阅示例](#%E5%8F%91%E5%B8%83-%E8%AE%A2%E9%98%85%E7%A4%BA%E4%BE%8B)
    - [AOSP里的多线程](#aosp%E9%87%8C%E7%9A%84%E5%A4%9A%E7%BA%BF%E7%A8%8B)
        - [Atomic](#atomic)
        - [AtomicPair](#atomicpair)
        - [锁层次](#%E9%94%81%E5%B1%82%E6%AC%A1)
            - [LockLevel 枚举](#locklevel-%E6%9E%9A%E4%B8%BE)
        - [基类 BaseMutex](#%E5%9F%BA%E7%B1%BB-basemutex)
        - [互斥锁 Mutex](#%E4%BA%92%E6%96%A5%E9%94%81-mutex)
        - [读写锁 ReaderWriterMutex & 特化的 MutatorMutex](#%E8%AF%BB%E5%86%99%E9%94%81-readerwritermutex--%E7%89%B9%E5%8C%96%E7%9A%84-mutatormutex)
        - [条件变量 ConditionVariable](#%E6%9D%A1%E4%BB%B6%E5%8F%98%E9%87%8F-conditionvariable)
        - [Scoped Locker 辅助类](#scoped-locker-%E8%BE%85%E5%8A%A9%E7%B1%BB)
        - [QuasiAtomic](#quasiatomic)
        - [Futex](#futex)
            - [基本原理](#%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86)
            - [常用 Futex 操作](#%E5%B8%B8%E7%94%A8-futex-%E6%93%8D%E4%BD%9C)
            - [在 ART 中的应用](#%E5%9C%A8-art-%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8)

<!-- /TOC -->

## C++ 多线程基础
### 1.1 线程概念

- **线程（Thread）**：操作系统分配 CPU 时间的最小单元。一个进程（Process）可以包含多个线程，共享进程的地址空间、文件描述符等资源。  
- **并发（Concurrency）**：程序在逻辑上可以有多个任务同时进行，但在单核上可能是时间片轮转在不同任务间切换。  
- **并行（Parallelism）**：多个任务在物理上同时运行（多核 CPU 同时执行）。  

### 1.2 `std::thread` 的创建与管理

C++11 引入了 `<thread>` 库，使用 `std::thread` 可以方便地创建和管理线程：
```cpp
#include <iostream>
#include <thread>

// 无参数函数
void print_hello() {
    std::cout << "Hello from thread!\n";
}

int main() {
    // 1. 创建线程并立即启动
    std::thread t1(print_hello);

    // 2. 传递带参数的函数或 lambda
    std::thread t2([](int x) {
        std::cout << "Lambda received: " << x << "\n";
    }, 42);

    // 3. 主线程等待 t1 和 t2 完成
    t1.join();
    t2.join();

    return 0;
}
```


- std::thread t(func, args...)：通过可调用对象 func 创建一个新线程，线程函数执行 func(args...)。
- 可调用对象：可以是普通函数、函数对象（重载 operator() 的类）、lambda、成员函数指针（结合 std::bind 或 std::mem_fn 使用）等。
- .join()：阻塞当前线程，直到目标线程执行完毕。如果某个 std::thread 对象被销毁时仍处于可 join 状态，会导致程序 std::terminate()，因此必须在销毁前调用 .join() 或 .detach()。
- .detach()：将线程设置为“分离”（detached）状态，线程会在后台继续执行，程序不再等待它结束，也无法获取其返回值。使用时务必保证线程中不访问已析构的对象。

### 1.3 线程参数和返回值

通过传值或引用，将参数传递给线程函数。例如：

```cpp
void func_by_value(int x) { /* ... */ }
void func_by_reference(int& x) { /* ... */ }

int main() {
    int a = 10;
    std::thread t1(func_by_value, a);      // 传值
    std::thread t2(func_by_reference, std::ref(a));  // 传引用，需要 std::ref
    t1.join();
    t2.join();
}
```

- 无法直接从线程函数中返回值，可以使用以下方式获取：
    1. 全局/共享变量 + 同步：线程将结果写入共享变量并用锁保护。
	2. std::promise / std::future —— C++ 标准库提供的一对同步原语：线程内部 promise.set_value(value)，主线程通过 future.get() 获取返回值。
	3. 返回值包装到动态分配的对象/自定义回调

### 1.4 线程的生命周期
 1. 创建：调用 std::thread 构造函数。
 2. 执行：新线程开始执行指定的函数体。
 3.	同步或分离：调用 .join() 或 .detach()，确定线程的结束方式。
	- joinable()：判断线程是否可 join。
	- 如果在对象析构时依然可 join，会抛出 std::terminate()。
 4. 结束：线程函数执行完毕或者调用 std::exit()/std::terminate()。


## 同步原语

多线程访问共享资源时需要同步原语来保证数据一致性，避免数据竞争（Data Race）和混乱状态。

### 2.1 互斥量（std::mutex）与锁类型
 - std::mutex：最基本的互斥量，用于保护临界区。
 - `std::lock_guard<std::mutex>`：RAII 方式加锁/解锁。

```cpp
#include <mutex>

std::mutex mtx;
int shared_data = 0;

void thread_func() {
    std::lock_guard<std::mutex> lock(mtx);
    // 临界区：只有一个线程能执行到这里
    shared_data++;
}
```

 - `std::unique_lock<std::mutex>`：功能更灵活，可延迟加锁、提前解锁、与条件变量配合使用。
 - `std::timed_mutex` 在 std::mutex 基础上增加了“定时尝试加锁”（`try_lock_for`、`try_lock_until`）接口。
 - `std::recursive_mutex` 可重入互斥锁，同一个线程可以重复获得锁。
    - 内部维护一个计数器：首次 lock() 时计数器从 0 → 1，二次 lock() 时计数器 1 → 2，以此类推。只有当调用 unlock() 把计数器减到 0 时，才真正释放底层互斥资源。
 - `std::recursive_timed_mutex` 在 std::recursive_mutex 基础上增加定时加锁接口。
 - `std::shared_timed_mutex（C++14）／std::shared_mutex（C++17）`允许多读（shared）或单写（exclusive），并支持定时加锁。
    - 共享锁（读）lock_shared()：阻塞直到没有写锁存在，此时可以并发获取读锁（多个线程同时持有读锁）。
    - 独占锁（写）lock()：阻塞直到没有其他线程持有（无论是读锁还是写锁），然后以写权限加锁。

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <chrono>

std::timed_mutex tmtx;

void worker(int id) {
    // 尝试在 200ms 内获取锁
    if (tmtx.try_lock_for(std::chrono::milliseconds(200))) {
        std::cout << "Thread " << id << " got the lock, working...\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(300)); 
        tmtx.unlock();
        std::cout << "Thread " << id << " released the lock.\n";
    } else {
        std::cout << "Thread " << id << " timeout, giving up.\n";
    }
}

int main() {
    // 线程 1 先获取锁并持续 300ms
    std::thread t1([]{
        tmtx.lock();
        std::cout << "Thread 1 acquired lock, sleeping 300ms...\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(300));
        tmtx.unlock();
        std::cout << "Thread 1 released lock.\n";
    });
    // 等 50ms 再启动线程 2，演示超时效果
    std::this_thread::sleep_for(std::chrono::milliseconds(50));
    std::thread t2(worker, 2);

    t1.join();
    t2.join();
    return 0;
}
```

#### 死锁示例

两个线程同时锁定两个互斥量，互相等待对方释放：

```cpp
std::mutex m1, m2;

void thread1() {
    std::lock_guard<std::mutex> lock1(m1);
    std::this_thread::sleep_for(std::chrono::milliseconds(50));
    std::lock_guard<std::mutex> lock2(m2); // 老板：等待 m2，但 m2 可能已被 thread2 锁住
}

void thread2() {
    std::lock_guard<std::mutex> lock2(m2);
    std::this_thread::sleep_for(std::chrono::milliseconds(50));
    std::lock_guard<std::mutex> lock1(m1); // 老板：等待 m1，但 m1 已被 thread1 锁住
}
```
 - 解决方案：规范锁顺序，或者使用 std::lock(m1, m2); 一次性锁定多个互斥量避免死锁。

### 2.2 条件变量（std::condition_variable）
 - 用于在线程间建立“等待—通知”机制。
 - `std::unique_lock<std::mutex>` + `std::condition_variable::wait/notify_one/notify_all`

在多线程编程中，常常需要一个线程等待某个条件满足，另一个线程在状态变化后通知等待线程继续执行。std::condition_variable 正是为此设计的同步原语。它通常与 std::mutex（或 `std::unique_lock<std::mutex>`） 配合使用，实现类似于“生产者—消费者”模型中的线程阻塞与唤醒。

- 场景示例：
    - 生产者线程向队列中加入数据后，通过 notify_one 或 notify_all 通知消费者线程；
    - 消费者线程在队列为空时，通过 wait 阻塞；当生产者把数据加入队列并 notify，消费者被唤醒并消费数据。

#### wait

原型（常用重载）
```cpp
void wait(std::unique_lock<std::mutex>& lock);
template< class Predicate >
void wait(std::unique_lock<std::mutex>& lock, Predicate pred);
```
语义
1. 解锁并阻塞
 - 当调用 cond.wait(lock) 时，当前线程会自动释放传入的 `std::unique_lock<std::mutex>`所持有的互斥锁，然后进入阻塞状态，直到满足两种情况之一才会醒来：
    - 其他线程对该条件变量调用了 notify_one() 或 notify_all()；
    - 出现“虚假唤醒”（spurious wakeup）。
2. 重新加锁
 - 被唤醒后，wait 会自动重新获取 lock，然后函数返回。
3. 谓词版本
 - wait(lock, pred) 等价于：
```cpp
while (!pred()) {
    wait(lock);
}
```
也就是说，如果传入一个返回 bool 的谓词，wait 会在每次被唤醒后重新检查 pred()，只有当谓词为 true 时才真正退出 wait，否则继续阻塞。这种方式可以避免“虚假唤醒”导致的错误。

**虚假唤醒**

TL;DR: 操作系统底层实现：Futex/Wait-Queue 导致的非确定性

大多数现代 Unix / Linux 系统，std::condition_variable 底层都会映射到某种Futex（Fast Userspace muTEX）或Wait-Queue机制上。其基本思路是：
1. 用户线程调用 wait(lock) 时，先在用户态登记“我要阻塞、并且用这个锁关联”；
2. 调用系统调用（syscall）陷入内核，将自己挂到一个等待队列里，然后切换到睡眠状态；
3. 未来如果有其他线程调用 notify_one()，它会向内核发出“唤醒这个等待队列上某个线程”的请求；
4. 内核把线程从“睡眠队列”里面标记为“可运行”（runnable），调度器在某个时间点把它放到可执行队列里，让它重新竞争 CPU；

上面这套机制虽然“看似”是：只有 notify 才会让线程从“等待队列”中唤醒，但其实在实际内核里，由于各种原因，线程在睡眠期间也可能被意外唤醒，比如：
- 定时器中断 / 信号：如果线程收到了某个信号（signal），或者内核在内部做一些超时检查，就可能会把线程从睡眠中“先醒来”，去处理信号、错误码或者做调试报告。
- 内核住在等待队列上的线程被错误地通知：在极少数情况下，内核的 Futex 实现可能因为竞争或者资源回收的缘故，将不该唤醒的线程也标记为“可运行”。
- 诊断/调试工具的干预：比如 GDB 之类的调试器，或者覆盖率工具、内存检查工具插桩，也可能让睡眠线程“看似被唤醒”来做必要的检查。
- 虚拟化/容器环境下的调度延迟：在虚拟机或容器化平台里，内核的 Futex PP（称为“被动页抢占”）可能产生额外中断。这些中断也能让 Futex 调用“误以为要唤醒线程”。

因此，在操作系统层面，单纯“有线程正在 wait() 上睡”，并不意味着“只有出自 notify() 的唤醒才会让它返回”，它可能因为上述各种“系统中断或竞态”而先被唤醒一次。底层实现通常无法也不想为“避免虚假唤醒”而在每次 wait 前后都做更复杂、更昂贵的检查／屏障。这样做的成本远高于“在用户态让线程简单回到队列／再回去睡”。

- C++ 标准允许实现抛弃一次“真实的 notify”或“合并唤醒事件”

在多核系统里，如果有两次相近时间的 notify_one() 操作，中间可能没有足够时间让用户线程切换回来并实际阻塞到 Futex 等待队列里，这时“相当于一次唤醒信号把多次 notify 合并”了。如此一来，被 notify 的线程不一定能“每次都醒，而且醒了就有用”——底层给它一个“可运行”状态后，它很快又会去到 wait() 再睡下去，或者先和另一个线程抢锁，然后马上继续睡。

- “一次 notify，却唤醒了多个线程，导致竞争后只有一个线程真正采到锁”

如果调用 notify_all()，底层可能一次性把所有等待线程都从 Futex 队列里挑出来放到就绪队列，然后它们会并发尝试 lock（或者继续检查谓词），只有第一个拿到锁或谓词为真的线程能继续做，其他线程又可能立刻 wait。这类唤醒-重睡循环，也从使用者角度看，像是“唤醒了无效的线程”（也是一种“虚假唤醒”）。

总之，由于“内核调度”、“Futex/Wait-Queue 实现细节”以及“合并/剔除”、以及“竞态”等原因，C++ 标准特意规定：
```
任何对 condition_variable::wait(...) 的唤醒，都 可能 是“虚假的”，它不保证对应某一次 notify 就一定意味着条件成立。
```

严格来说，只要“唤醒”这件事是交给操作系统内核来做，就无法保证“我只在我想要的时候唤醒”。notify 本质上只是“把线程从 Futex 队列里挑出来，标记为 runnable”，但从用户线程再次进入 wait() 到重新完全阻塞在 Futex/Wait-Queue 的过程，中间可能会发生交叉调度、信号、诊断中断等，导致：
1. 某一次 notify 看似没起作用
- 线程被唤醒后，可能还没有来得及继续往下执行，已经被抢去重新阻塞回 wait()——这对使用者而言就是“唤醒+再睡”两步连在一起，感觉上就像“虚假唤醒”。
- 这在高并发抢锁或多线程同时 notify_all() 时尤其明显。多线程进来拿锁/检查谓词，只有第一个线程真正继续工作，其他线程立刻发现自己不满足谓词又得重新 wait。
2. 在使用 wait_for(...) / wait_until(...) 带超时版时，notify 也可能唤醒得早/唤醒得晚
- 如果一个线程正在做 cv.wait_for(lk, std::chrono::seconds(1), predicate)，中间收到一个 notify()，它就会醒来，但如果 predicate() 仍为 false，wait_for 会再继续“短暂休眠”，直到超时后再返回 false。这段过程也看起来像“notify 也没用”或“虚假唤醒”。
3. 标准没有对“正真对应某个 notify”做排他性保证
- 也就是说，你不能指望“一次 notify_one() 就只能唤醒一个线程；其他线程就永远不会醒”。操作系统有可能先把多个线程都标为“可运行”，让它们纷纷去尝试拿锁或检查条件，然后其中大部分线程一下子就会因为条件不满足又卡回去睡。
- 这种“先把所有睡着的线程都从队列里拉出来，再让它们去竞争，只有一两个赢家”的行为，就很容易被用户误认为“notify 也出现了虚假唤醒”。


#### notify_one

原型
```cpp
void notify_one() noexcept;
```
语义
 - 通知一个（任意一个）正在等待该 condition_variable 的线程，唤醒它（如果没有线程在等待，则什么也不做）。
 - 被唤醒的线程将从 wait 中返回，并重新尝试获取与之关联的 mutex。例如：

```cpp
// 线程 A 正在 wait(lock);
cv.notify_one();  // 线程 A 被唤醒，但在返回 wait 之前，会先尝试重新获得 lock。
```


#### notify_all

原型
```cpp
void notify_all() noexcept;
```
语义
- 通知所有正在等待该 condition_variable 的线程都被唤醒。所有被唤醒的线程会竞争获取与之关联的 mutex，只有获得锁的线程才能真正从 wait 返回。


```cpp
#include <condition_variable>
#include <mutex>
#include <queue>

std::queue<int> data_queue;
std::mutex queue_mutex;
std::condition_variable queue_cv;
bool finished = false; // 生产者完成标志

// 生产者
void producer() {
    for (int i = 0; i < 10; ++i) {
        std::lock_guard<std::mutex> lock(queue_mutex);
        data_queue.push(i);
        queue_cv.notify_one();
    }
    {
        std::lock_guard<std::mutex> lock(queue_mutex);
        finished = true;
    }
    queue_cv.notify_all();
}

// 消费者
void consumer() {
    std::unique_lock<std::mutex> lock(queue_mutex);
    while (!finished || !data_queue.empty()) {
        queue_cv.wait(lock, [] {
            return !data_queue.empty() || finished;
        });
        while (!data_queue.empty()) {
            int val = data_queue.front();
            data_queue.pop();
            lock.unlock(); // 临界区结束，解锁以便生产者继续
            // 处理数据
            std::cout << "Consumed: " << val << std::endl;
            lock.lock();
        }
    }
}
```
 - wait(lock, predicate)：等价于 while (!predicate()) wait(lock);，避免了虚假唤醒（Spurious Wakeup）。

#### 小结
- wait(lock)：让当前线程在释放 lock 后阻塞，直至收到 notify_one/notify_all 或者虚假唤醒，再重新获得 lock 并返回。
- wait(lock, pred)：不断循环调用 wait(lock)，直到谓词 pred() 为真，才返回。推荐用于避免虚假唤醒带来的不确定性。
- notify_one()：唤醒一个正在等待的线程，让它重新尝试获取锁并返回 wait。
- notify_all()：唤醒所有正在等待的线程，让它们都重新尝试获取锁并返回 wait。

在使用时，应当将“修改共享状态”与“通知”这两个操作封装在同一个临界区（持锁状态）中完成，或紧接在持锁状态下完成修改后立即 notify，以确保等待线程不会错过通知。同时，不要忽视“虚假唤醒”的可能性，务必使用带谓词的 wait 或者在 wait 后重新检查条件。

通过上述用法，std::condition_variable 能够帮助你在多线程环境中高效地实现“等待／唤醒”逻辑，协调不同线程对共享资源的访问与处理顺序。


### 2.3 原子操作（std::atomic）
 - 提供了无需互斥锁的原子读写、更新操作，对性能敏感的场景可以避免锁开销。
 - 支持整型、指针、布尔值等原子类型，还可以 `std::atomic<std::shared_ptr<T>>、std::atomic<foo*>` 等。
 - 原子变量保证对读写操作具有同步性，无数据竞争。

#### 支持的类型

C++ 标准对 std::atomic 进行了以下划分：
1. 整型与指针特化
 - `std::atomic<bool>`
 - `std::atomic<char> / std::atomic<signed char> / std::atomic<unsigned char>`
 - `std::atomic<short> / std::atomic<unsigned short>`
 - `std::atomic<int> / std::atomic<unsigned int>`
 - `std::atomic<long> / std::atomic<unsigned long>`
 - `std::atomic<long long> / std::atomic<unsigned long long>`
 - `std::atomic<char16_t>、std::atomic<char32_t>、std::atomic<wchar_t>`

 - `std::atomic<T*>` 对任意指针类型均有专门支持
这些特化类型都保证了“对其上读、写、RMW、比较并交换等操作”都有硬件级别的原子性，并提供了最丰富的操作集（例如 fetch_add、fetch_or 等）。


2. 通用类型特化 `std::atomic<T>`
对用户自定义类型或非整型/非指针类型，也可以用 `std::atomic<T>`。
但前提是 T 必须满足 “可原子化” 的要求：
- 对象大小不超过实现所支持的原子宽度，或者编译器会退化为“锁+内存屏障”的实现。
- T 必须是可拷贝和可赋值的，并且尽可能满足“标准布局类型”或者“平凡拷贝可移植”要求。
- 这一特化只保证对 load()、store()、exchange()、compare_exchange_…() 等少量接口提供原子保证，且 并不一定支持整型那样的算术型专用接口（如 fetch_add、fetch_and 等），除非 T 自身定义了对应的原子算术行为。
3. 原子标志 std::atomic_flag
- std::atomic_flag 是最轻量的原子类型，仅支持两项操作：
    - test_and_set(std::memory_order) —— 将标志设为 true，并返回先前的值
    - clear(std::memory_order) —— 将标志设为 false

atomic_flag 通常用于实现基于自旋的互斥锁或一次性标记。

```cpp
// 在 C++ 中，“可原子化”往往要求类型满足某些底层内存模型和 ABI 规则，以便编译器/平台能够 “对该类型的存取或复制”生成等价于硬件原子指令或较低开销的指令序列。常见的两个关键术语是 “标准布局类型”（standard-layout type） 和 “平凡拷贝可移植类型”（trivially copyable type）。下面分别解释它们的含义、要求和用途。


// 一个类型被称为标准布局类型（standard‐layout type），其目的是确保满足以下几点：
// 1.所有对象的布局在各个编译单元或 ABI 下具有一致性，即具有可预测的、可互操作的内存布局；
// 2.能够安全地将对象所占内存视为“C 风格结构体”，并且可以在不同模块、不同语言（如 C 与 C++）之间互相传递。

// 它的成员从第一个非静态成员到最后一个成员，布局是线性的、无额外填充 “隐藏父类成员” 的插入。
// 如果有多个基类或派生关系，只要它满足“单一的基类”或“所有基类都没有非静态数据成员”等约束，就可以保证基类与派生类成员在内存中的相对偏移是可预测的。
// 可以放心地用 reinterpret_cast、memcpy 或者与 C 结构体混合使用，而不担心布局不一致导致的未定义行为。

// C++17 一个类型 T（class、struct、union）要成为标准布局类型，必须全部满足以下条件（下面把“非静态数据成员”简称为 NSDM）：
// 1. T 没有虚基类；
// 2. T 的所有非静态数据成员及其所有基类都必须是“同一个最底层类”直接或间接定义的，即不能有两个不同的基类拥有相同的内存地址起点。换言之，不允许多重继承导致派生类拥有多个同类型或具有非静态数据成员的基类。或者更精确地说：T 只能有 at most 一个直接或间接含有非静态数据成员的基类。
// 3. T 的所有非静态数据成员（NSDM）都具有相同的访问控制属性（全部 public 或全部 protected，不能部分是 private），或者当且仅当它们都在同一个基类中声明。也就是说，不能在派生类中同时出现 public、private 之类的不同访问权限的 NSDM；否则布局可能在不同编译器或不同模式下不一致。
// 4. T 本身不能有虚函数；其所有基类也不能有虚函数。
// 5. 如果 T 有非静态数据成员，那么第一个这样的成员在内存中的偏移必须为零（例如没有编译器插入额外的前置字节对齐）；否则就不是标准布局类型。
// 6. 如果 T 派生自某个基类 B，那么 B 必须本身是标准布局类型，且 B 中的所有非静态成员与 T 中的成员不“重名”（不能有相同名称的成员跨层次，否则布局不再简单可预测）。

// 总结起来，“标准布局类型”要保证：
// 没有虚继承、没有虚函数；
// 如果存在基类，只有一个直接含有 NSDM 的基类；
// 所有 NSDM 的访问控制修饰符相同，且布局顺序跟声明顺序一致；
// 类没有二义性继承或多重基类中含有数据成员的情况；
// 第一个 NSDM 的偏移为 0。

// 标准布局类型示例 1：简单的 POD struct
struct A {
    int x;       // NSDM
    double y;    // NSDM
    // 没有虚函数、没有基类、所有成员都是 public
};

// 标准布局类型示例 2：单一基类
struct B {
    int b1;
};

struct C : B {   // 派生自标准布局类型 B
    double c1;
    // 所有 NSDM（b1, c1）都来自同一个继承链，不存在多继承或不同访问权限
};

// 非标准布局类型示例：多重继承且都有成员
struct D1 { int d1; };
struct D2 { int d2; };

struct E : D1, D2 {  // E 布局下 D1::d1 与 D2::d2 无法保证线性排列
    int e1;
};
// 由于多重基类 D1 和 D2 都含有 NSDM，E 并不符合“只能有一个带数据成员的基类”约束

// 非标准布局类型示例：不同访问权限的 NSDM
struct F {
public:
    int f1;
private:
    int f2;  // 因为 f1 是 public，f2 是 private，它们的访问权限不同
};  // F 不是标准布局类型

// 非标准布局类型示例：带虚函数
struct G {
    virtual void foo();  // 虚函数插入 vptr，导致布局不透明
    int g1;
};  // G 不是标准布局类型


// “平凡拷贝可移植类型”（Trivially Copyable Type）

// 一个类型若要成为 平凡拷贝可移植类型（trivially copyable type），意味着：
// 1.该类型在内存层面可以直接用 memcpy 或类似方式拷贝，而不会破坏其内部状态或存在潜在未定义行为；
// 2.类型的所有拷贝构造、移动构造、拷贝赋值、移动赋值、析构器 都要么是编译器自动生成的“平凡（trivial）”版本，要么被明确删除。也就是没有用户自定义的拷贝/移动/析构；
// 3.其所有非静态数据成员本身也要都是平凡拷贝可移植类型；
// 4.对于联合（union）或类（class/struct）来说，它不能含有虚函数、虚继承等会导致内部状态需要更复杂初始化/清理的成员。

// 换言之，如果一个类型是平凡拷贝可移植的，就可在内存里做按位拷贝（bitwise copy），然后得到一个等价的对象，无需调用任何构造、析构或赋值函数。这样，对于原子类型而言：
// 编译器可以安全地在底层将一个平凡拷贝可移植类型的大小与对齐信息适配到硬件支持的原子宽度（如 32 位、64 位、128 位等），或退化到用“内部互斥+加载/存储”来保证原子性；无需担心拷贝过程中会调用用户代码或破坏对象不透明的内部状态。


// 一个类型 T（可以是 class、struct、union、或基元类型）要成为平凡拷贝可移植类型（trivially copyable），必须：
// 1. 具有平凡的拷贝构造/移动构造/拷贝赋值/移动赋值/析构
// 平凡的拷贝构造：就是编译器自动生成，不带任何用户自定义逻辑；如果用户声明了拷贝构造就不是平凡的；
// 平凡的移动构造：同样要求编译器自动生成；
// 平凡的拷贝赋值、移动赋值：编译器自动生成；
// 平凡的析构：如果用户没有自定义析构，且成员本身析构平凡，则析构就是平凡的；
// 2. 所有非静态成员和直接基类也都必须是平凡拷贝可移植类型. 这保证“所有子对象”都能安全按位拷贝，不需要调用自定义的拷贝或析构。
// 3. 对数组类型的要求
// 如果 T 是数组，那么 T 的元素类型也必须是平凡拷贝可移植的。
// 4. 不存在虚函数与虚继承
// 虚函数会在对象内部插入额外的 vptr 信息，按位拷贝后可能打乱虚表指针；所以含有虚函数/虚继承的类型不满足“按位复制就能保留行为”的要求。

// 若一个类型满足上述所有条件，则 std::is_trivially_copyable<T>::value 在编译时为 true，并可安全地把对象当做“仅有数据成员，无需特殊构造、赋值或析构逻辑”的平坦结构来拷贝。

// 平凡拷贝可移植类型示例 1：基元类型
int x = 42;     // int 本身就是平凡拷贝可移植

// 示例 2：简单的 POD struct
struct A {
    int a;
    double b;
};  // A 拷贝构造、析构均由编译器生成，且成员都是基元类型 => trivially copyable

// 示例 3：含有数组的平凡拷贝可移植类型
struct B {
    char buf[16];
    int  len;
};  // B 的成员都是平凡可拷贝类型，B 自身也是平凡拷贝可移植

// 非平凡拷贝可移植类型示例：含有用户自定义析构
struct C {
    int* p;
    ~C() { delete p; }  // 用户自定义析构，析构不再是平凡
};  // C 不是 trivially copyable

// 非平凡拷贝可移植类型示例：含有虚函数
struct D {
    virtual void foo();  // 虚函数会插入 vptr，需要在构造时初始化
    int val;
};  // D 不是 trivially copyable

// 非平凡拷贝可移植类型示例：自定义拷贝构造
struct E {
    int x;
    E(const E& o) : x(o.x + 1) {}  // 使用者定义了拷贝构造，非平凡
};

// 为什么在 std::atomic<T> 中要求它们？
// 硬件原子指令（如 x86 的 LOCK CMPXCHG、ARM 的 LDREX/STREX）一般只能保证“对 1、2、4、8（或更大）字节宽度的数据”做原子读写；
// 如果一个类型在内存中“内部结构”过于复杂（包含非平凡析构、虚表指针、二义性基类等），则编译器无法简单地映射到“单条硬件原子指令”上。为了保证安全，编译器/运行时可能会退化为“内部用 mutex + 存储屏障”来实现原子性，这对性能有较大影响。
// 如果类型是平凡拷贝可移植的，就意味着可以安全地把它当作“一个连续、不含隐藏指针的内存块”来读写/交换；编译器可以直接用 memcpy、bitwise 操作或硬件原子指令对它做 RMW，而不需要调用构造/析构。
// 对于 std::atomic<T>，如果 T 是标准布局类型，则 T 在不同编译单元、不同模块之间的布局是一致的，读写一个原子 T 不会因为编译器在不同位置插入隐藏填充或把基类成员放到不同偏移而导致不一致。这样才能保证多个线程在同一个机器/同一地址空间共享同一块内存时，对该原子类型的读写或比较并交换能够得到一致结果。
```

#### 构造与赋值接口

默认构造与显式初始化

- 无参构造
```cpp
std::atomic<int> a;    // 未初始化——处于“不确定状态（indeterminate）”
```
无参构造并不会将其内部值设置为 0，通常会产生“未定义的初始值”。在使用前，必须 store(...) 或通过赋值进行初始化，否则 load() 是未定义行为。

- 带参数构造
```cpp
std::atomic<int> a{0};         // 初始化为 0
std::atomic<long>  b(123456L); // 初始化为 123456
std::atomic<MyType> c(obj);    // 若 MyType 可原子化，则可以这样直接初始化
```
以上等价于调用 `a.store(0, std::memory_order_seq_cst)`，在构造时就把值写入原子对象，保证原子一致性（采用顺序一致性 `memory_order_seq_cst`）。

#### 赋值运算符与拷贝/移动
 - 拷贝构造与拷贝赋值：被显式删除 `std::atomic<T>` 不可拷贝。
 - 移动构造与移动赋值：同样被删除 `std::atomic<T> `不可移动。
 - 赋值运算符（operator=）
```cpp
std::atomic<int> a{10};
std::atomic<int> b;
b = 5;           // 等价于 b.store(5, memory_order_seq_cst)
a = b.load();    // 读取 b 的值并写入 a，其实更好的写法是 a.store(b.load(), mo)
```
虽然“赋值”能接受一个 int，也可以接受另一个 `atomic<int>` 的 load() 值，但不能写成 b = a;（编译错误，不能拷贝 atomic 对象）。

#### 基本读写接口：load / store

**load**
```cpp
T load(std::memory_order order = std::memory_order_seq_cst) const noexcept;
```
 - 功能：以指定的内存序从原子对象中“原子地”读取一个值，并返回该值。
 - 返回值：读取时刻的“快照”——在多线程同时修改时，你会得到某个时刻的一致值。
 - 参数：内存序（memory_order）
 - 默认 memory_order_seq_cst（顺序一致性，最强保证）。
 - 也可选 memory_order_acquire、memory_order_relaxed 等，后文会详细说明。
 - 异常保证：noexcept。如果对象未初始化，行为未定义。

**store**
```cpp
void store(T desired, std::memory_order order = std::memory_order_seq_cst) noexcept;
```
- 功能：以指定的内存序“原子地”将 desired 写入原子对象。
- 参数：内存序（memory_order）
- 默认 memory_order_seq_cst。
- 如果只需“发布写”（Release 语义），可改为 memory_order_release。
- 返回值：无。
- 注意：store 不返回旧值；若想获取旧值，请使用 exchange 或 fetch_xxx 系列。

#### 读—改—写接口（RMW）

RMW（Read-Modify-Write）接口在单个原子操作里完成“读取旧值、计算新值并写回”，并返回“旧值”或“新值”（视具体接口而定）。RMW 天然避免了 “load + 修改 + store” 之间被其它线程插入竞争。

**exchange**
```cpp
T exchange(T desired, std::memory_order order = std::memory_order_seq_cst) noexcept;
```
- 将 desired 写入原子对象，并返回写操作前的旧值。
- 等价于下列伪代码，但是真实执行是原子的：
```cpp
T old = x;
x = desired;
return old;
```
返回的是旧值。若要同时获取新值与旧值，可以在返回后自行对比：
```cpp
int old = a.exchange(42);
int now = a.load();  // now == 42
```
**fetch_add / fetch_sub / fetch_or / fetch_and / fetch_xor …**

对于整型或枚举类型，标准提供一系列 “按位／算术” RMW 方法，接口定义为：
```cpp
T fetch_add(T arg, std::memory_order order = std::memory_order_seq_cst) noexcept;
T fetch_sub(T arg, std::memory_order order = std::memory_order_seq_cst) noexcept;
T fetch_and(T arg, std::memory_order order = std::memory_order_seq_cst) noexcept;
T fetch_or (T arg, std::memory_order order = std::memory_order_seq_cst) noexcept;
T fetch_xor(T arg, std::memory_order order = std::memory_order_seq_cst) noexcept;
```
功能：
- fetch_add(arg)：执行原子加法
```cpp 
old = x; 
x = old + arg; 
return old;
```
- fetch_sub(arg)：执行原子减法 
```cpp
old = x;
x = old - arg; 
return old;
```
- fetch_and(arg)：按位与
- fetch_or(arg)：按位或
- fetch_xor(arg)：按位异或

返回值：写操作之前的“旧值”。


示例：
```cpp
std::atomic<int> cnt{10};
int prev = cnt.fetch_add(5);  // prev == 10，cnt 变为 15
prev = cnt.fetch_sub(3);      // prev == 15，cnt 变为 12
prev = cnt.fetch_and(0x0F);   // prev == 12，cnt 变为 12 & 0x0F == 12
```
- 注意：
    - fetch_add 等仅在整型、枚举或对应的原子特化上可用；若对 `std::atomic<MyStruct>`，若未定义“＋”操作则编译错误。
    - 若仅需要“修改后”值，也可写成：
```cpp
int newVal = cnt.fetch_add(1) + 1;  // 先获取旧值，再加 1
```

**比较并交换接口：compare_exchange_strong / compare_exchange_weak**

“比较并交换”（Compare-and-Swap，CAS）是底层并发编程中最关键的原子操作，用于实现 lock-free 算法。

```cpp
bool compare_exchange_weak(
    T& expected,
    T desired,
    std::memory_order success = std::memory_order_seq_cst,
    std::memory_order failure = std::memory_order_seq_cst
) noexcept;

bool compare_exchange_strong(
    T& expected,
    T desired,
    std::memory_order success = std::memory_order_seq_cst,
    std::memory_order failure = std::memory_order_seq_cst
) noexcept;
```
参数说明：
- T& expected：首先读取原子对象当前值，与之比较。如果 “当前值 == expected” 成立，则原子将 desired 写入；否则不写入。
    - 如果写入成功，函数返回 true，原子对象被设置为 desired，expected 保持不变。
    - 如果写入失败（表示其他线程已修改过），函数返回 false，并将原子当前值写回到 expected 中（此时 expected 会被更新为最新的“原子当前值”）。调用者可以通过再次检查 expected 决定如何重试。
- T desired：希望写入的新值。
- success：当 CAS 成功时采用的内存序（通常可用 memory_order_acq_rel 或 memory_order_seq_cst）。
- failure：当 CAS 失败时采用的内存序，通常必须是“release 或者更弱”。常见写法：


```cpp
a.compare_exchange_weak(exp, newVal,
                         std::memory_order_acq_rel,
                         std::memory_order_acquire);

```


**Weak vs Strong**

- compare_exchange_weak 允许“虚假失败（spurious failure）”：即使原子值与 expected 相等，也有可能返回 false（对于底层硬件 CAS 指令可能因冲突而重试）。
- compare_exchange_strong 保证“只要原子值等于 expected，就一定返回成功”，即没有“虚假失败”。
- 在循环 “重试 CAS” 时，通常用 compare_exchange_weak，因为它在失败时底层可能会降低开销。
- 如果不在循环里使用，而希望一次性成功 / 失败给出明确反馈，则用 compare_exchange_strong。

**虚假失败**

许多现代 CPU（如 ARM、PowerPC、RISC-V 等）并不直接提供“原子比较并交换（CAS）”指令，而是提供一对叫作 Load-Link (LL) 与 Store-Conditional (SC) 的指令序列来实现原子更新：
1. LL (Load-Link)：
- 读取一个内存地址的值，同时“在处理器内部”记住这个地址（建立一个监视点、reservation）。
2. SC (Store-Conditional)：
- 尝试将一个新值写回到这同一地址，前提是自从 LL 之后，没有其他核心（core）/线程写过这个地址。
- 如果自 LL 以来，地址被改写，则 SC 会失败，不会写入任何内容；如果地址未被修改，则 SC 成功并完成写入。
- SC 会返回一个状态码（通常是 0 表示失败、1 表示成功）。

在这种模型下，就可能出现多种“虚假”或“无关操作”导致 SC 失败的情况。例如：
- 缓存行被驱逐：即使其它核心没有显式写入那个地址，只要该地址所在的缓存行被处理器“逐出”（evict）或“失去独占状态”，SC 也可能认为“有人改过它”而失败。
- 系统监视点超时：某些处理器会在 LL 与 SC 之间加上时间或访问次数限制，如果这一段时间里缓存与监视点发生了某些硬件层面的冲刷，就算没有真正写入同一地址，SC 也会返回失败。
- 编译器插入优化指令：在编译为汇编时，有些平台会为安全或同步目的插入额外的指令序列，这也可能让 SC 误判“中间发生了修改”。

这些因素都会导致 SC（Store-Conditional）“看似”失败，从而让 compare_exchange_weak 也出现“当前值与预期值相等，却返回失败”的情况。

相比之下，某些架构（如 x86）直接提供了 LOCK CMPXCHG 这样的原子比较并交换指令，这种指令在“当前值与预期值相等”时一定会成功，不会有“虚假失败”的可能。这也是为什么在 x86 平台上，compare_exchange_strong 模式下的实现往往直接做一次单条指令，而不会出现弱失败。但在 LL/SC 架构上，就只能用“弱失败”来向用户暴露“需要重试”的信号。


```cpp
std::atomic<int> a{5};
int expected = 5;
int newValue = 10;

// 情况 A：a 当前值 == expected
bool ok = a.compare_exchange_strong(expected, newValue);
// ok == true；
// a 变为 10；
// expected 仍为 5

// 情况 B：a 当前值 != expected
a.store(7);
expected = 5;
ok = a.compare_exchange_strong(expected, newValue);
// ok == false；
// a 维持为 7；
// expected 被更新为 7（原子当前值）
```

注意：失败时 expected 会被重写为原子当前值，调用者若要继续重试，需要重新设置新的 desired 或继续使用 expected。

**CAS 循环示例（无锁栈/队列中常见）**
```cpp

#include <atomic>
#include <iostream>

// 假设单向链表节点
struct Node {
    int data;
    Node* next;
};

// 头指针
std::atomic<Node*> head{nullptr};

// 推入新节点（无锁栈）
void push(int value) {
    Node* new_node = new Node{value, nullptr};
    Node* old_head;
    do {
        old_head = head.load(std::memory_order_acquire);
        new_node->next = old_head;
        // 尝试将 head 从 old_head 改为 new_node
    } while (!head.compare_exchange_weak(
                 old_head,        // expected
                 new_node,        // desired
                 std::memory_order_release,
                 std::memory_order_relaxed));  // 失败后仅做 acquire 语义
    // 如果 CAS 失败，old_head 会被更新为最新的 head，
    // 并继续循环重试
}

// 弹出节点
Node* pop() {
    Node* old_head;
    do {
        old_head = head.load(std::memory_order_acquire);
        if (!old_head) return nullptr; // 空栈
    } while (!head.compare_exchange_weak(
                 old_head,
                 old_head->next,
                 std::memory_order_acq_rel,
                 std::memory_order_acquire));
    return old_head;
}
```
- 上面 push 中，compare_exchange_weak 在多线程竞争场景下可能因为其他线程成功修改而失败，然后“重置 expected 再次重试”。
- memory_order 的选取要符合集合：成功时至少是 release，失败时至少是 acquire，以保证可见性和顺序性。

**is_lock_free**
```cpp
bool is_lock_free() const noexcept;
```
- 返回 true 表示该原子类型在本平台／本实现下“永远”无锁实现（利用硬件指令完成全部操作）。
- 返回 false 则表示可能会退化到“内部加锁”或“库实现的互斥”来保证原子性。
- 例如某些平台上，`std::atomic<long long>`可能是“无锁的”，而 `std::atomic<long double>` 可能是“有锁的”。
- 示例：
```cpp
std::atomic<int> a{0};
if (a.is_lock_free()) {
    // a 上的操作都完全靠硬件指令。高性能。
} else {
    // 可能内部会用到互斥量来保证原子性。
}
```


**C++20 新增：wait 与 notify_one/notify_all**

C++20 在 std::atomic<T> 上新增了“等待/通知”功能，用于无锁编程中让线程等待某个原子值变化，而无需自己手动轮询或使用条件变量。
```cpp
void wait(T old, std::memory_order order = std::memory_order_seq_cst) const noexcept;
void notify_one() noexcept;
void notify_all() noexcept;
```
`wait(old)`
- 如果当前原子值 x.load(order) != old，则立即返回；
- 否则“挂起”线程，直到其他线程通过 store、fetch_*、exchange、compare_exchange_* 等原子操作将其修改为不同于 old 的新值后，再通知唤醒。
- 实际底层可实现为“等待一个内部的 futex 或条件变量”，避免了用户自行轮询：
```cpp
int expected = 0;
// 当 atomic_val 的值仍为 0 时，挂起当前线程
atomic_val.wait(expected);
// 唤醒后，atomic_val 变为了不同于 0 的值
```
`notify_one() / notify_all()`
- 会唤醒至少一个（或全部）调用过 wait(old) 但尚未被唤醒的线程。
- 仅当与之关联的原子值在某次修改后与 old 不同时，这些等待中的线程才会真正返回。
- 相当于为原子对象维护了一个“阻塞队列”，在修改原子值后由系统自动 notify。

使用示例（C++20）：
```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> flag{0};

void waiter() {
    int old = 0;
    std::cout << "Waiter: 等待 flag 变为非 0...\n";
    flag.wait(old);  // 如果 flag==0，挂起直到 flag != 0
    std::cout << "Waiter: 检测到 flag 已更改为 " 
              << flag.load() << "\n";
}

void notifier() {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    flag.store(42, std::memory_order_release);
    flag.notify_one();
    std::cout << "Notifier: 设置 flag=42 并唤醒 waiter。\n";
}

int main() {
    std::thread t1(waiter), t2(notifier);
    t1.join();
    t2.join();
    return 0;
}
```
- waiter 中 flag.wait(old)：如果 flag.load() 返回 0，则挂起当前线程；直到其他线程写入了非 0 值并调用 notify_one()。

注意：C++20 的 wait/notify_* 仅在部分标准库实现中才可用（如 libstdc++ ≥ 11、MSVC、libc++ 等），且内部会根据类型大小决定是否使用 futex 等高效原语。



#### 示例与注意事项

下面归纳各个接口在实际场景中的典型用法及注意要点。

```cpp

std::atomic<int> data{0};

// 示例一：顺序一致性
void writer() {
    data.store(100, std::memory_order_seq_cst);
}

void reader() {
    int v = data.load(std::memory_order_seq_cst);
    // v 要么是 0，要么是 100，不会观测到“部分写入”的数据
}

// 示例二：Acquire / Release 语义
std::atomic<bool> ready{false};
int shared_value;

void producer() {
    shared_value = 123;                                 // 普通写
    ready.store(true, std::memory_order_release);       // 发布写
}

void consumer() {
    while (!ready.load(std::memory_order_acquire));     // 获取读
    // 确保 “shared_value = 123” 一定在 ready.store 之前完成并对本线程可见
    std::cout << shared_value << "\n";  // 必然看到 123
}
```
memory_order_release 和 memory_order_acquire 联合使用能建立“写-读”屏障，保证对 shared_value 的正常读写顺序。

```cpp
std::atomic<int> flag{0};

// 单次交换并获取旧值
int old = flag.exchange(1, std::memory_order_acq_rel);
// old 表示交换前 flag 的值
if (old == 0) {
    // 成功从 0 → 1；可做类似 “唯一初始化” 的逻辑
}
```

fetch_add 等示例

```cpp
std::atomic<int> counter{0};

// 并发累加示例
void inc_worker() {
    for (int i = 0; i < 1000; ++i) {
        // 不需要同步任何其它数据，只需对 counter 原子增加
        counter.fetch_add(1, std::memory_order_relaxed);
    }
}

// main 中 spawn 多个线程调用 inc_worker()
// 最终 counter.load() == 线程数 * 1000
```
使用 memory_order_relaxed 时，只保证“每个操作对 counter 本身是原子可见”的语义，不会对其它内存访问产生同步或屏障作用。适用于只关心“结果正确累加”而不关心跨变量可见性的场景。

```cpp
std::atomic<int> lock_flag{0};

// 简易自旋互斥（非递归）
void spin_lock() {
    int expected = 0;
    // 一旦成功将 0 → 1，就获得锁；否则循环重试
    while (!lock_flag.compare_exchange_weak(
               expected,   // expected 初始为 0
               1,          // desired
               std::memory_order_acq_rel,
               std::memory_order_acquire)) {
        // CAS 失败后，expected 会被更新为 lock_flag 当前值
        expected = 0; // 一定重置 expected 为 0，再次尝试
        // 或者 std::this_thread::yield() 让出
    }
}

void spin_unlock() {
    lock_flag.store(0, std::memory_order_release);
}
```
这里用 compare_exchange_weak 放在循环里，允许“虚假失败”，既简化了失败处理，也降低了某些平台 CAS 指令的重试开销。

```
std::atomic<long long> a64{0};
std::atomic<double> d64{0.0};

std::cout << std::boolalpha
          << "atomic<long long> 无锁？ " << a64.is_lock_free() << "\n"
          << "atomic<double> 无锁？     " << d64.is_lock_free() << "\n";

```
如果 is_lock_free() 返回 false，说明对该类型的所有原子操作可能由库内部借助“互斥”或“其他手段”实现，不完全是硬件原子指令。性能可能低于“真正无锁实现”的同类原子类型。

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <chrono>

std::atomic<int> sig{0};

void waiter() {
    int old = 0;
    // 当 sig == old 时，挂起等待
    sig.wait(old);
    std::cout << "waiter: sig 已被修改为 " << sig.load() << "\n";
}

void notifier() {
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    sig.store(123, std::memory_order_release);
    sig.notify_one();
    std::cout << "notifier: 已将 sig=123 并通知 waiter\n";
}

int main() {
    std::thread t1(waiter), t2(notifier);
    t1.join();
    t2.join();
    return 0;
}
```
如果不调用 sig.notify_one()，则即便 sig.store(123)，此时 waiter 也可能一直被挂起。
wait 必须与 “原子修改 + notify” 配对使用。




### 2.4 读写锁（std::shared_mutex / std::shared_timed_mutex）
 - 读-写锁允许多个读者同时持有共享锁，而写者需要独占锁。
 - C++17 引入了 std::shared_mutex，C++14 引入了 std::shared_timed_mutex（支持超时）。

```cpp
#include <shared_mutex>
std::shared_mutex rw_mutex;
int shared_data = 0;

void reader() {
    std::shared_lock<std::shared_mutex> lock(rw_mutex);
    // 多个读者可以并行读取 shared_data
    std::cout << "Read: " << shared_data << std::endl;
}

void writer() {
    std::unique_lock<std::shared_mutex> lock(rw_mutex);
    // 写者独占访问
    shared_data += 1;
    std::cout << "Wrote: " << shared_data << std::endl;
}

```

### 2.5 其他同步工具
#### std::promise / std::future / std::packaged_task
- `std::promise<T>`：在线程 A 创建，将值或异常传给 future。
- `std::future<T>`：在线程 B 等待并获取 promise 设置的结果。
- `std::packaged_task<F>`：把可调用对象 “打包” 为任务，返回一个 future 给调用者，任务执行后将结果填入 future。

在多线程/异步编程中，经常会遇到“一个线程发起任务，另一个线程等待并获取执行结果”的需求。C++11 标准库引入了一套基于 std::promise、std::future、std::packaged_task、std::async 等组件的机制，用于实现跨线程的值传递、异常传递以及任务封装。它们之间的核心思路如下：
- `std::promise<T>`
是“生产者端”，在某个线程中创建，持有一个内部的共享状态（shared state）。生产者（Promise 持有者）可以向这个共享状态中存入一个类型为 T 的值，或者存入一个异常。
- `std::future<T>`
是“消费者端”，通过 `std::promise<T>::get_future()` 或者通过 `std::packaged_task<T()>`、std::async 等接口获得。Future 也持有指向同一个共享状态的引用（引用计数式）。当 future 的持有线程调用 future.get() 时，如果共享状态中的值尚未就绪，则会阻塞等待；一旦 promise.set_value(...) 或者任务执行结束（由 packaged_task 或 async 填充），共享状态就绪，future.get() 返回对应的值（或抛出异常）。
- `std::packaged_task<R()>`
可将任意可调用对象（函数指针、函数对象、lambda、std::function 等）与一个 `std::future<R>` 关联起来。打包后，包裹的可调用对象便成为“任务”，你可以在某个线程中直接调用 task()，此时它会执行内部函数并把返回值/异常存入共享状态，此后与之关联的 `std::future<R>` 就可以通过 get() 拿到结果。



#### 线程局部存储（Thread-Local Storage）
- C++11 引入 thread_local 关键字，实现每个线程拥有各自独立实例的全局／静态变量。
```cpp
thread_local int tls_var = 0;  // 每个线程各自独立
```
## 执行模型（Execution Model）

C++ 标准定义了一套执行模型来规范多线程的行为，并将程序行为分为“程序顺序语义（sequenced-before）”与“同步操作（synchronization operations）”之间的可见性约束。执行模型决定了编译器和硬件对指令的重排序、寄存器优化等行为边界。

### 3.1 单线程执行流程

在单线程环境下，编译器和处理器可以自由地对指令进行重排序、寄存器缓存、内存分块写回等操作，只要最终从单线程可见的行为与程序顺序等价即可。即：
- “序”关系（Sequenced-before）：在同一个线程中，如果表达式 A 在程序顺序上出现在 B 之前，则 A 被称为 “sequenced-before” B。编译器与硬件必须保证 A 的效果对 B 可见。
- 在单线程下，所有编译器和硬件优化都不能改变程序在该线程内部的顺序语义。

### 3.2 多线程执行流程

在多线程环境中，多核处理器和缓存系统会带来更复杂的可见性与重排序问题。C++ 执行模型定义了一些关键概念：
- 线程间 “同步操作（synchronization operations）”：包含原子操作、锁操作、条件变量通知等。
- “跨线程可见性（Visibility）”：一个线程对共享变量的写操作对另一个线程可见，只有当它们之间建立了某种内存序约束（比如获取-释放语义、锁释放-获取、条件变量 notify/wait）后才保证。
- “数据竞争（Data Race）”：如果两个线程访问同一个内存位置，并至少有一个是写操作，且它们之间没有“同步先行（synchronizes-with）”关系，就会产生未定义行为。

#### 同步先行（Happens-Before）关系
 - Sequenced-before：同一线程内的先后顺序。
 - Inter-thread “同步先行”：
	- 如果操作 A “释放-序（release sequence）” 操作 B，则 A 在其他线程中 “获取-序（acquire sequence）” B 后，A “synchronizes-with” B；B “synchronizes-before” 其后续操作。
	- mutex.unlock() “synchronizes-with” 相应的 mutex.lock()，使得 unlock 前的写操作对后续 lock 后可见。
	- condition_variable.notify_one()/all() “synchronizes-with” 在 wait() 重新返回之后的操作，建立可见性。
	- std::promise.set_value() “synchronizes-with” 相应的 std::future.get() 返回操作。
 - “Happens-before”：若 A “sequenced-before” B，或者 A “synchronizes-before” B，则 A “happens-before” B。
 - 在两个线程间，只有“happens-before” 关系下，才保证读操作能够看到写操作的结果，否则形成数据竞争，程序行为未定义。

### 3.3 线程与可见性、重排序
- 编译器重排序：只要在单个线程内部不破坏“sequenced-before”关系，就可以对指令进行重排序。
- 硬件重排序：多核 CPU 会对读写指令做乱序执行、缓存合并、写缓冲等优化，只保证在执行屏障（Fence）或特殊原子操作时的可见性。

示例重排序场景
```cpp
int x = 0, y = 0;
bool ready = false;

// 线程 A
x = 42;         // (1)
ready = true;   // (2)

// 线程 B
if (ready) {    // (3)
    std::cout << x << std::endl; // (4)
}
```
在没有任何同步机制的情况下，处理器/编译器可能将 (1) 和 (2) 重排序。当线程 B 观察到 ready == true（在(3)），理论上 (1) 可能还没有对 B 可见，导致打印出 0。为了避免这种行为，必须使用原子变量或锁保证“释放-获取”顺序。

## 内存模型（Memory Model）

C++11 内存模型详细定义了多线程下各种原子操作的内存序（Memory Order），并严格区分何时允许编译器和 CPU 做重排序以及如何建立跨线程的可见性。理解内存模型能够帮助我们编写高效且正确的并发程序。

### 4.1 数据竞争（Data Race）与未定义行为
- 数据竞争（Data Race）：在两个或多个线程中同时访问同一内存位置，且至少一个访问是写操作，且这些访问之间没有“Happens-Before”关系。
- 如果存在数据竞争，C++ 标准将产生未定义行为（Undefined Behavior）。因此要么使用原子操作，要么保护共享数据访问使用互斥量或其他同步机制。

### 4.2 内存序（Memory Order）分类

std::atomic 操作可以指定以下几种内存序参数（std::memory_order）：
1. memory_order_relaxed
- 松散顺序（Relaxed Ordering）：仅保证原子操作本身原子性，对其他读写操作不建立同步关系。
- 不会产生“happens-before”关系，不保证可见性顺序。
- 适合只需要原子自增、自减计数，不需要跨线程同步的场合。
2. memory_order_consume（C++20 标准基本移除）
- 消费顺序（Consume Ordering）：一种比 acquire 更弱的获取-释放语义，适合依赖数据流读取。由于在大多数实现中退化成 acquire，现已不常使用。
3. memory_order_acquire
- 获取语义（Acquire Semantics）：保证此原子读及之后所有读写不会重排到该操作之前。
- 与 “释放-获取” 结合能够建立可见性边界。
4. memory_order_release
- 释放语义（Release Semantics）：保证此原子写及之前所有读写不会重排到该操作之后。
- 与 acquire 结合，保证在 release 操作之前的所有写对 acquire 操作后的线程可见。
5. memory_order_acq_rel
- 获取-释放语义（Acquire-Release Semantics）：当执行原子读-改-写（如 fetch_add、compare_exchange）时，既具备 acquire 语义又具备 release 语义。
6. memory_order_seq_cst
- 顺序一致性（Sequential Consistency）：最强内存序，所有带 seq_cst 的原子操作在全局层面呈现一个单一的全序（所有线程都观察到相同的顺序）。
- load 带 acquire 语义，store 带 release 语义，同时禁止或者限制更多的重排序。

### 4.3 顺序一致性（Sequential Consistency）
- C++ 标准要求所有使用 memory_order_seq_cst 的原子操作在所有线程中看起来像在一个全局顺序中执行。
- 采用 seq_cst 可以最简单地推理多线程程序行为，但性能可能受限。

示例：顺序一致性
```cpp
#include <atomic>
#include <thread>
#include <cassert>

std::atomic<int> A(0), B(0);
int r1 = 0, r2 = 0;

void thread1() {
    A.store(1, std::memory_order_seq_cst); // (1)
    r1 = B.load(std::memory_order_seq_cst); // (2)
}

void thread2() {
    B.store(1, std::memory_order_seq_cst); // (3)
    r2 = A.load(std::memory_order_seq_cst); // (4)
}

int main() {
    std::thread t1(thread1);
    std::thread t2(thread2);
    t1.join();
    t2.join();
    // seq_cst 保证不可能同时 r1 == 0 且 r2 == 0
    assert(!(r1 == 0 && r2 == 0));
    return 0;
}
```
- 在 (1) 和 (3) 之间任意顺序，若 (1) 先于 (3)，则 (4) 会看到 A==1，r2==1；
若 (3) 先于 (1)，则 (2) 会看到 B==1，r1==1。不会出现两者都读到 0。

### 4.4 获取-释放（Acquire-Release）语义
- 释放操作 （Release）：某线程 X 在写某原子变量时使用 memory_order_release，并且在此写操作之前的普通写操作都会对随后“获取”此原子变量的线程可见。
- 获取操作 （Acquire）：某线程 Y 在读同一个原子变量时使用 memory_order_acquire，该原子读操作及之后的普通读写不会重排到 acquire 之前。
- Release- Acquire 建立“同步先行（synchronizes-with）”关系，从而产生“happens-before”关系。

示例：Acquire-Release
```cpp
#include <atomic>
#include <thread>
#include <cassert>

std::atomic<bool> ready(false);
int data = 0;

void producer() {
    data = 100;                              // (1) 普通写
    ready.store(true, std::memory_order_release); // (2) release
}

void consumer() {
    while (!ready.load(std::memory_order_acquire)); // (3) acquire 等待
    // 保证 (1) 的写对 consumer 可见
    assert(data == 100);
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);
    t1.join();
    t2.join();
    return 0;
}
```
- 在 producer 线程中，(1) data=100 在 (2) release 之前，硬件/编译器不会把 (1) 和 (2) 的顺序颠倒。
- consumer 线程在 (3) acquire 之后，保证能看到 data=100。

### 4.5 松散内存序（Relaxed Ordering）
- memory_order_relaxed 只保证原子性，不建立“happens-before”关系。适合于对数据一致性要求不高的场景，例如原子统计计数、唯一 ID 生成等。
- 不保证可见性顺序，无法用于同步。

示例：计数器
```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> counter(0);

void worker() {
    for (int i = 0; i < 1000000; ++i) {
        counter.fetch_add(1, std::memory_order_relaxed);
    }
}

int main() {
    std::thread t1(worker), t2(worker);
    t1.join();
    t2.join();
    std::cout << "Final counter: " << counter.load(std::memory_order_relaxed) << "\n";
    return 0;
}
```
- 虽然使用 relaxed，仍保证最后值一定是 2,000,000，因为 fetch_add 本身是原子操作，杜绝写时数据竞争。
- 但不能用于需要线程之间的先行关系或保证可见性的场景。

### 4.6 内存屏障（Barrier / Fence）
- std::atomic_thread_fence()：显式插入内存屏障，禁止编译器/处理器在屏障两侧对读写重排序。
- std::atomic_thread_fence(std::memory_order_acq_rel) 可用来在非原子操作场景下补充同步边界，但现代 C++ 多倾向于使用原子变量和锁，而不是手动屏障。


```cpp
#include <atomic>

std::atomic<int> X, Y;
int a, b;

void thread1() {
    a = 1;                                      // (1)
    std::atomic_thread_fence(std::memory_order_release); // release 屏障
    X.store(1, std::memory_order_relaxed);      // (2)
}

void thread2() {
    while (X.load(std::memory_order_relaxed) != 1); // (3)
    std::atomic_thread_fence(std::memory_order_acquire); // acquire 屏障
    b = Y.load(std::memory_order_relaxed);       // (4)
    // 保证 (4) 在 (1) 之后可见
}
```


### 4.7 “释放-一致性” 总结

“Release Consistency” 是一种比顺序一致性更弱、更灵活的内存一致性模型，只需在建立明确的同步操作（Acquire-Release）时才保证可见性顺序。

C++ 内存模型通过多种 memory_order 选项给出了强度不同的同步约束：
1. seq_cst：最强一致性，全序列一致。
2. acquire / release：只要在写和读间使用“获取-释放”就能建立“happens-before”可见性。
3. relaxed：仅保证原子性，不保证任何跨线程可见性。


## 常见模式与实践

### 5.1 双重检查锁（Double-Checked Locking）
- 用于实现线程安全的懒汉式单例，但需注意 C++11 之后必须使用 std::atomic 和 memory_order 保证正确。


```cpp
#include <atomic>
#include <mutex>

class Singleton {
public:
    static Singleton* getInstance() {
        // First check (不加锁)
        Singleton* tmp = instance.load(std::memory_order_acquire);
        if (tmp == nullptr) {
            std::lock_guard<std::mutex> lock(init_mutex);
            tmp = instance.load(std::memory_order_relaxed);
            if (tmp == nullptr) {
                tmp = new Singleton();
                instance.store(tmp, std::memory_order_release);
            }
        }
        return tmp;
    }

private:
    Singleton() = default;
    ~Singleton() = default;
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;

    static std::atomic<Singleton*> instance;
    static std::mutex init_mutex;
};

std::atomic<Singleton*> Singleton::instance(nullptr);
std::mutex Singleton::init_mutex;
```


- 原理：
 1. 先读取原子指针 instance，若非空则直接返回。
 2. 否则进入锁内，再次检查，若仍然为空则执行初始化，并以 release 语义写入原子指针；
 3. 外层读取使用 acquire 保证在拿到非空指针后，能看到完整构造完成的对象状态。

### 5.2 生产者-消费者模式
- 常见场景：一个或多个生产者线程往队列推送数据，若队列满则等待；一个或多个消费者线程从队列取数据，若队列空则等待。
- 推荐使用 std::mutex + std::condition_variable：


```cpp
#include <condition_variable>
#include <mutex>
#include <queue>
#include <thread>

template <typename T>
class BlockingQueue {
public:
    BlockingQueue(size_t capacity) : capacity_(capacity) {}

    void push(const T& item) {
        std::unique_lock<std::mutex> lock(mutex_);
        cond_not_full_.wait(lock, [this] { return queue_.size() < capacity_; });
        queue_.push(item);
        cond_not_empty_.notify_one();
    }

    T pop() {
        std::unique_lock<std::mutex> lock(mutex_);
        cond_not_empty_.wait(lock, [this] { return !queue_.empty(); });
        T item = queue_.front();
        queue_.pop();
        cond_not_full_.notify_one();
        return item;
    }

private:
    size_t capacity_;
    std::queue<T> queue_;
    std::mutex mutex_;
    std::condition_variable cond_not_empty_;
    std::condition_variable cond_not_full_;
};
```

### 5.3 线程池（Thread Pool）
- 预先创建固定数量的线程，从任务队列中取出任务并执行，减少线程创建销毁开销。
- 基本结构：
 1.	一个线程安全的任务队列（`BlockingQueue<std::function<void()>>`）。
 2.	N 个工作线程，循环从队列中取任务并执行。
 3.	提供 submit() 接口，提交一个 `std::function<void()>` 到队列。
 4.	线程池析构时发送停止信号（如特殊任务），并 join() 所有线程。

 
```cpp
#include <vector>
#include <thread>
#include <functional>
#include <future>
#include <atomic>

class ThreadPool {
public:
    ThreadPool(size_t num_threads) : stop_flag(false) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(queue_mutex);
                        cond_var.wait(lock, [this] {
                            return stop_flag || !tasks.empty();
                        });
                        if (stop_flag && tasks.empty())
                            return;
                        task = std::move(tasks.front());
                        tasks.pop();
                    }
                    task();
                }
            });
        }
    }

    template <class F, class... Args>
    auto submit(F&& f, Args&&... args)
        -> std::future<typename std::invoke_result<F, Args...>::type> 
    {
        using RetType = typename std::invoke_result<F, Args...>::type;
        auto task_ptr = std::make_shared<std::packaged_task<RetType()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );
        std::future<RetType> result = task_ptr->get_future();
        {
            std::lock_guard<std::mutex> lock(queue_mutex);
            tasks.emplace([task_ptr] { (*task_ptr)(); });
        }
        cond_var.notify_one();
        return result;
    }

    ~ThreadPool() {
        {
            std::lock_guard<std::mutex> lock(queue_mutex);
            stop_flag = true;
        }
        cond_var.notify_all();
        for (std::thread& worker : workers) {
            if (worker.joinable())
                worker.join();
        }
    }

private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queue_mutex;
    std::condition_variable cond_var;
    bool stop_flag;
};
```

### 5.4 并发队列（Concurrent Queue）
- 基于无锁（Lock-Free）或细粒度锁的实现：
- 无锁队列使用 std::atomic 和 CAS 实现，复杂度高，但性能好。
- Boost、TBB、folly 都提供了成熟的并发队列实现。

示例（简化版，单生产者单消费者环形缓冲区）：


```cpp
template <typename T, size_t N>
class SPSCQueue {
public:
    SPSCQueue(): head(0), tail(0) {
        static_assert((N & (N - 1)) == 0, "N must be a power of 2");
        buffer = new T[N];
    }
    ~SPSCQueue() { delete[] buffer; }

    bool push(const T& item) {
        size_t next_head = (head + 1) & (N - 1);
        if (next_head == tail.load(std::memory_order_acquire))
            return false; // full
        buffer[head] = item;
        head = next_head;
        return true;
    }

    bool pop(T& item) {
        size_t curr_tail = tail.load(std::memory_order_relaxed);
        if (curr_tail == head) 
            return false; // empty
        item = buffer[curr_tail];
        tail.store((curr_tail + 1) & (N - 1), std::memory_order_release);
        return true;
    }

private:
    T* buffer;
    size_t head;
    std::atomic<size_t> tail;
};
```

### 5.5 读者-写者锁模式
- 当读操作远多于写操作时，通过 std::shared_mutex 提高并发度：


```cpp
#include <shared_mutex>
#include <map>

std::map<int, std::string> data_map;
std::shared_mutex map_mutex;

// 读者
std::string read_data(int key) {
    std::shared_lock<std::shared_mutex> lock(map_mutex);
    auto it = data_map.find(key);
    return (it != data_map.end()) ? it->second : "";
}

// 写者
void write_data(int key, const std::string& value) {
    std::unique_lock<std::shared_mutex> lock(map_mutex);
    data_map[key] = value;
}
```

- 注意：过多的读-写转换也会引起性能问题，必须根据实际场景权衡使用。

### 5.6 原子升级（Atomic Upgrade）与 ABA 问题
- ABA 问题：在无锁数据结构中，如果线程 A 读取某个原子指针值为 A，然后线程 B 将其改为 B，又将其改回 A，导致线程 A 认为没有变化而误用。
- 解决方案：带版本号的 CAS 或指针标记（Tagged Pointer），或者使用 `std::atomic<std::uintptr_t>` 存储指针和低位标记。
- 示例（简化，带版本号的指针）：


```cpp
struct TaggedPtr {
    T* ptr;
    uint64_t tag;
};

std::atomic<TaggedPtr> head;

bool cas_head(TaggedPtr& expected, TaggedPtr desired) {
    return head.compare_exchange_strong(expected, desired,
                                        std::memory_order_acq_rel,
                                        std::memory_order_acquire);
}

```

## 调试与性能优化

多线程程序往往难以调试和优化，以下是一些常见工具和策略：

### 6.1 数据竞争检测（Thread Sanitizer）

ThreadSanitizer (TSAN)：Clang/GCC 提供的线程数据竞争检测工具，可以在编译时加上 -fsanitize=thread -fPIE -pie，运行时会报告潜在的数据竞争和其他并发错误。

<https://clang.llvm.org/docs/ThreadSanitizer.html>

### 6.2 内存屏障性能影响
- memory_order_seq_cst：最强一致性，需要 CPU 在指令间插入 MFENCE（或类似）指令，可能导致性能下降。
- memory_order_acquire / release：比 seq_cst 更弱，通常足够构建可见性，可减少屏障开销。
- memory_order_relaxed：无同步开销，仅保证原子性。尽可能使用 relaxed（在逻辑允许的情况下）提高性能。

### 6.3 伪共享（False Sharing）与对齐优化
- 伪共享：**多个线程频繁修改**不同缓存行内同一缓存块（Cache Line）上的不同变量，导致缓存一致性协议频繁失效，严重拖慢性能。
- 解决方案：
    - 为频繁并发访问的变量使用对齐（alignas(64)）隔离开。
    - 避免给频繁写入的变量放在同一缓存行内。

```cpp
struct alignas(64) PaddedCounter {
    std::atomic<int> counter;
    // 填充对齐，避免与其他变量共享缓存行
    char pad[64 - sizeof(std::atomic<int>)];
};
```

### 6.4 原子 vs 互斥对比
- 互斥（Mutex）：开销较大（系统调用、上下文切换），但编程更直观，适用于临界区较大或复杂操作同步。
- 原子（Atomic）：开销小（CPU 原子指令），但只适合较简单的同步场景（单个变量更新、无锁队列等）。
- 选择原则：若同步逻辑简单且性能要求高，可优先考虑原子操作；若涉及复杂数据结构或多步骤操作，使用互斥更容易保证正确性。

## 示例与案例分析

### 7.1 简单的多线程加法示例


```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <atomic>

std::atomic<long> counter(0);

void add_work(int num) {
    for (int i = 0; i < num; ++i) {
        counter.fetch_add(1, std::memory_order_relaxed);
    }
}

int main() {
    const int N = 4;      // 线程数量
    const int ITER = 10000000;
    std::vector<std::thread> threads;

    for (int i = 0; i < N; ++i) {
        threads.emplace_back(add_work, ITER);
    }
    for (auto& t : threads) {
        t.join();
    }
    std::cout << "Expected: " << static_cast<long>(N) * ITER
              << ", Actual: " << counter.load() << std::endl;
    return 0;
}
```
### 7.2 发布-订阅示例
- 使用 std::promise / std::future：



```cpp
#include <future>
#include <iostream>
#include <thread>

int main() {
    std::promise<int> prom;
    std::future<int> fut = prom.get_future();

    std::thread producer([&prom] {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        prom.set_value(42);
    });

    std::thread consumer([&fut] {
        int value = fut.get(); // 阻塞直到 promise 设置了值
        std::cout << "Got value: " << value << std::endl;
    });

    producer.join();
    consumer.join();
    return 0;
}
```

## AOSP里的多线程

### 1. Atomic<T>

封装了 `std::atomic<T>`


```cpp
template<typename T>
class Atomic : public std::atomic<T> {
 public:
  // 构造
  Atomic() : std::atomic<T>(T()) {}
  explicit Atomic(T value) : std::atomic<T>(value) {}

  // Java Data 语义的 load/store（memory_order_relaxed）
  // Load data from an atomic variable with Java data memory order semantics.
  //
  // Promises memory access semantics of ordinary Java data.
  // Does not order other memory accesses.
  // Long and double accesses may be performed 32 bits at a time.
  // There are no "cache coherence" guarantees; e.g. loads from the same location may be reordered.
  // In contrast to normal C++ accesses, racing accesses are allowed.
  T LoadJavaData() const {
    return this->load(std::memory_order_relaxed);
  }

  // Store data in an atomic variable with Java data memory ordering semantics.
  //
  // Promises memory access semantics of ordinary Java data.
  // Does not order other memory accesses.
  // Long and double accesses may be performed 32 bits at a time.
  // There are no "cache coherence" guarantees; e.g. loads from the same location may be reordered.
  // In contrast to normal C++ accesses, racing accesses are allowed.
  void StoreJavaData(T desired_value) {
    this->store(desired_value, std::memory_order_relaxed);
  }

  // 强/弱、顺序一致或 relaxed 的 Compare-And-Set
  bool CompareAndSetStrongSequentiallyConsistent(T expected, T desired);
  bool CompareAndSetWeakSequentiallyConsistent(T expected, T desired);
  bool CompareAndSetStrongRelaxed(T expected, T desired);
  bool CompareAndSetStrongRelease(T expected, T desired);
  bool CompareAndSetWeakRelaxed(T expected, T desired);
  bool CompareAndSetWeakAcquire(T expected, T desired);
  bool CompareAndSetWeakRelease(T expected, T desired);

  // 返回旧值版的 compare-exchange
  T CompareAndExchangeStrongSequentiallyConsistent(T expected, T desired);

  // 统一接口
  bool CompareAndSet(T expected, T desired,
                     CASMode mode,
                     std::memory_order memory_order);

  // 获取底层地址（用于 futex 等）
  volatile T* Address();

  // 类型上限
  static T MaxValue();
};
using AtomicInteger = Atomic<int32_t>;

// Increment a debug- or statistics-only counter when there is a single writer, especially if
// concurrent reads are uncommon. Usually appreciably faster in this case.
// NOT suitable as an approximate counter with multiple writers.
template <typename T>
void IncrementStatsCounter(std::atomic<T>* a) {
  a->store(a->load(std::memory_order_relaxed) + 1, std::memory_order_relaxed);
}

```
- LoadJavaData / StoreJavaData
与 Java 操作相同的松弛语义。
- CompareAndSetXxx
支持强弱 CAS、顺序一致或 release/acquire/relaxed 等多种内存序。


### 2. AtomicPair<IntType>

可做为原子性加载/存储的宽度超过 CPU 原生支持的大小的读写。
比如NativeDexCache数据结构会使用到

**注意！**
- This uses top 4-bytes of the key as version counter and lock bit, which means the stored pair key can not use those bytes.
对于16字节大小版本的AtomicPair的第一个成员key只能使用4字节

```cpp
// 16 字节版：基于 seq-lock
// key 的高 32 位作为版本号和锁标志
// 读：循环加载 key,val,key → 校验版本号未锁定且不变
// 写：CAS 把锁位设为 1 → store val → store 新版本号&解锁

// Implement 16-byte atomic pair using the seq-lock synchronization algorithm.
// This is currently only used for DexCache.
//
// This uses top 4-bytes of the key as version counter and lock bit,
// which means the stored pair key can not use those bytes.
//
// This allows us to read the cache without exclusive access to the cache line.
//
// The 8-byte atomic pair uses the normal single-instruction implementation.
//
static constexpr uint64_t kSeqMask = (0xFFFFFFFFull << 32);
static constexpr uint64_t kSeqLock = (0x80000000ull << 32);
static constexpr uint64_t kSeqIncr = (0x00000001ull << 32);
static constexpr uint kAtomicPairMaxSpins = 10'000u; // 自旋等待次数
static constexpr uint kAtomicPairSleepNanos = 5'000u;

// std::pair<> is not trivially copyable and as such it is unsuitable for atomic operations.
template <typename IntType>
struct PACKED(2 * sizeof(IntType)) AtomicPair {
  static_assert(std::is_integral_v<IntType>);

  AtomicPair(IntType f, IntType s) : key(f), val(s) {}

  IntType key;
  IntType val;
};

// 简单实现（仅限可被 std::atomic 支持的大小）
template <typename IntType>
ALWAYS_INLINE static inline AtomicPair<IntType> AtomicPairLoadAcquire(AtomicPair<IntType>* pair) {
  static_assert(std::is_trivially_copyable<AtomicPair<IntType>>::value);
  auto* target = reinterpret_cast<std::atomic<AtomicPair<IntType>>*>(pair);
  return target->load(std::memory_order_acquire);
}

template <typename IntType>
ALWAYS_INLINE static inline void AtomicPairStoreRelease(AtomicPair<IntType>* pair,
                                                        AtomicPair<IntType> value) {
  static_assert(std::is_trivially_copyable<AtomicPair<IntType>>::value);
  auto* target = reinterpret_cast<std::atomic<AtomicPair<IntType>>*>(pair);
  target->store(value, std::memory_order_release);
}

// IntType 为 uint64_t时, 即pair一共16字节的模版特化版本
ALWAYS_INLINE static inline AtomicPair<uint64_t> AtomicPairLoadAcquire(AtomicPair<uint64_t>* pair) {
  auto* key_ptr = reinterpret_cast<std::atomic_uint64_t*>(&pair->key);
  auto* val_ptr = reinterpret_cast<std::atomic_uint64_t*>(&pair->val);
  for (uint i = 0;; ++i) {
    uint64_t key0 = key_ptr->load(std::memory_order_acquire);
    uint64_t val = val_ptr->load(std::memory_order_acquire);
    uint64_t key1 = key_ptr->load(std::memory_order_relaxed);
    uint64_t key = key0 & ~kSeqMask;
    // 我们认为读取key和value中间没有发生写入
    if (LIKELY((key0 & kSeqLock) == 0 && key0 == key1)) {
      return {key, val};
    }
    if (UNLIKELY(i > kAtomicPairMaxSpins)) {
      NanoSleep(kAtomicPairSleepNanos);
    }
  }
}

ALWAYS_INLINE static inline void AtomicPairStoreRelease(AtomicPair<uint64_t>* pair,
                                                        AtomicPair<uint64_t> value) {
  DCHECK((value.key & kSeqMask) == 0) << "Key=0x" << std::hex << value.key;
  auto* key_ptr = reinterpret_cast<std::atomic_uint64_t*>(&pair->key);
  auto* val_ptr = reinterpret_cast<std::atomic_uint64_t*>(&pair->val);
  uint64_t key = key_ptr->load(std::memory_order_relaxed);
  for (uint i = 0;; ++i) {
    key &= ~kSeqLock;  // Ensure that the CAS below fails if the lock bit is already set.
    if (LIKELY(key_ptr->compare_exchange_weak(key, key | kSeqLock))) {
      break;
    }
    if (UNLIKELY(i > kAtomicPairMaxSpins)) {
      NanoSleep(kAtomicPairSleepNanos);
    }
  }
  // 新版本号, 解锁, key的值
  key = (((key & kSeqMask) + kSeqIncr) & ~kSeqLock) | (value.key & ~kSeqMask);
  val_ptr->store(value.val, std::memory_order_release);
  key_ptr->store(key, std::memory_order_release);
}
```

- 小于等于 8 字节 可直接 `std::atomic<AtomicPair<IntType>>`
- 16 字节 采用顺序锁（seq-lock）算法：
    1. 载入 key0 → 载入 val → 再次载入 key1
    2. 如果 key0==key1 且“锁位”未置位，则返回 {key0,val}
    3. 否则自旋/睡眠后重试
    4. 存储时先 CAS 上锁、写入 val、写版本号并解锁

### 3. 锁层次

#### LockLevel 枚举
- 定义了一个 锁等级体系，用于在运行时检查拿锁顺序，避免死锁。
- 每种锁在构造时绑定一个 LockLevel，同线程内只允许从低到高的顺序获取。

```cpp
// LockLevel is used to impose a lock hierarchy [1] where acquisition of a Mutex at a higher or
// equal level to a lock a thread holds is invalid. The lock hierarchy achieves a cycle free
// partial ordering and thereby cause deadlock situations to fail checks.
//
// [1] http://www.drdobbs.com/parallel/use-lock-hierarchies-to-avoid-deadlock/204801163
enum LockLevel : uint8_t {
  kLoggingLock = 0,
  kSwapMutexesLock,
  kUnexpectedSignalLock,
  kThreadSuspendCountLock,
  kAbortLock,
  kJniIdLock,
  kNativeDebugInterfaceLock,
  kSignalHandlingLock,
  // A generic lock level for mutexes that should not allow any additional mutexes to be gained
  // after acquiring it.
  kGenericBottomLock,
  // Tracks the second acquisition at the same lock level for kThreadWaitLock. This is an exception
  // to the normal lock ordering, used to implement Monitor::Wait - while holding one kThreadWait
  // level lock, it is permitted to acquire a second one - with internal safeguards to ensure that
  // the second lock acquisition does not result in deadlock. This is implemented in the lock
  // order by treating the second acquisition of a kThreadWaitLock as a kThreadWaitWakeLock
  // acquisition. Thus, acquiring kThreadWaitWakeLock requires holding kThreadWaitLock. This entry
  // is here near the bottom of the hierarchy because other locks should not be
  // acquired while it is held. kThreadWaitLock cannot be moved here because GC
  // activity acquires locks while holding the wait lock.
  kThreadWaitWakeLock,
  kJdwpAdbStateLock,
  kJdwpSocketLock,
  kRegionSpaceRegionLock,
  kMarkSweepMarkStackLock,
  // Can be held while GC related work is done, and thus must be above kMarkSweepMarkStackLock
  kThreadWaitLock,
  kJitCodeCacheMutatorAndCHALock,
  kRosAllocGlobalLock,
  kRosAllocBracketLock,
  kRosAllocBulkFreeLock,
  kAllocSpaceLock,
  kTaggingLockLevel,
  kJitCodeCacheLock,
  kTransactionLogLock,
  kCustomTlsLock,
  kJniFunctionTableLock,
  kJniWeakGlobalsLock,
  kJniGlobalsLock,
  kReferenceQueueSoftReferencesLock,
  kReferenceQueuePhantomReferencesLock,
  kReferenceQueueFinalizerReferencesLock,
  kReferenceQueueWeakReferencesLock,
  kReferenceQueueClearedReferencesLock,
  kReferenceProcessorLock,
  kJitDebugInterfaceLock,
  kBumpPointerSpaceBlockLock,
  kArenaPoolLock,
  kInternTableLock,
  kOatFileSecondaryLookupLock,
  kHostDlOpenHandlesLock,
  kVerifierDepsLock,
  kOatFileManagerLock,
  kTracingUniqueMethodsLock,
  kTracingStreamingLock,
  kJniLoadLibraryLock,
  kClassLoaderClassesLock,
  kDefaultMutexLevel,
  kDexCacheLock,
  kDexLock,
  kMarkSweepLargeObjectLock,
  kJdwpObjectRegistryLock,
  kModifyLdtLock,
  kAllocatedThreadIdsLock,
  kMonitorPoolLock,
  kClassLinkerClassesLock,  // TODO rename.
  kSubtypeCheckLock,
  kBreakpointLock,
  kMonitorListLock,
  kThreadListLock,
  kAllocTrackerLock,
  kDeoptimizationLock,
  kProfilerLock,
  kJdwpShutdownLock,
  kJdwpEventListLock,
  kJdwpAttachLock,
  kJdwpStartLock,
  kRuntimeThreadPoolLock,
  kRuntimeShutdownLock,
  kTraceLock,
  kHeapBitmapLock,
  // This is a generic lock level for a lock meant to be gained after having a
  // monitor lock.
  kPostMonitorLock,
  kMonitorLock,
  // This is a generic lock level for a top-level lock meant to be gained after having the
  // mutator_lock_.
  kPostMutatorTopLockLevel,

  kMutatorLock,
  kInstrumentEntrypointsLock,
  // This is a generic lock level for a top-level lock meant to be gained after having the
  // UserCodeSuspensionLock.
  kPostUserCodeSuspensionTopLevelLock,
  kUserCodeSuspensionLock,
  kZygoteCreationLock,

  // The highest valid lock level. Use this for locks that should only be acquired with no
  // other locks held. Since this is the highest lock level we also allow it to be held even if the
  // runtime or current thread is not fully set-up yet (for example during thread attach). Note that
  // this lock also has special behavior around the mutator_lock_. Since the mutator_lock_ is not
  // really a 'real' lock we allow this to be locked when the mutator_lock_ is held exclusive.
  // Furthermore, the mutator_lock_ may not be acquired in any form when a lock of this level is
  // held. Since the mutator_lock_ being held strong means that all other threads are suspended this
  // will prevent deadlocks while still allowing this lock level to function as a "highest" level.
  kTopLockLevel,

  kLockLevelCount  // Must come last.
};
```

### 4. 基类 BaseMutex
```cpp
class BaseMutex {
 public:
  const char* GetName() const;
  virtual void Dump(std::ostream& os) const = 0;
  virtual void WakeupToRespondToEmptyCheckpoint() = 0;
 protected:
  BaseMutex(const char* name, LockLevel level);
  ~BaseMutex();
  void RegisterAsLocked(Thread* self, bool check);
  void RegisterAsUnlocked(Thread* self);
  void CheckSafeToWait(Thread* self);
  // contention log 用于调试等待和持锁情况
};
```
- 记录锁名、等级、竞争日志
- 在 safepoint（排空检查）时唤醒持锁线程
- 管理死锁检测的“注册/注销”


### 5. 互斥锁 Mutex
```cpp
class Mutex : public BaseMutex {
 public:
  explicit Mutex(const char* name,
                 LockLevel level = kDefaultMutexLevel,
                 bool recursive = false);
  ~Mutex();

  void ExclusiveLock(Thread* self);
  bool ExclusiveTryLock(Thread* self);
  bool ExclusiveTryLockWithSpinning(Thread* self);
  void ExclusiveUnlock(Thread* self);

  pid_t GetExclusiveOwnerTid() const;
  unsigned int GetDepth() const;
};
```

Linux (ART_USE_FUTEXES=1)
- 用单个 AtomicInteger state_and_contenders_ 存储“持有标志 + 等待者计数”
- 轻度竞争时只做原子 CAS；高竞争时调用 futex() 挂起/唤醒
非 Linux
- 退回到 pthread_mutex_t
特性
- 支持自旋重试 (TryLockWithSpinning)
- 可选递归模式
- lock-level 检测


### 6. 读写锁 ReaderWriterMutex & 特化的 MutatorMutex


```cpp
class ReaderWriterMutex : public BaseMutex {
 public:
  explicit ReaderWriterMutex(const char* name, LockLevel level);
  void ExclusiveLock(Thread* self);
  void SharedLock(Thread* self);
  void ExclusiveUnlock(Thread* self);
  void SharedUnlock(Thread* self);
};

// MutatorMutex is a special kind of ReaderWriterMutex created specifically for the
// Locks::mutator_lock_ mutex. The behaviour is identical to the ReaderWriterMutex except that
// thread state changes also play a part in lock ownership. The mutator_lock_ will not be truly
// held by any mutator threads. However, a thread in the kRunnable state is considered to have
// shared ownership of the mutator lock and therefore transitions in and out of the kRunnable
// state have associated implications on lock ownership. Extra methods to handle the state
// transitions have been added to the interface but are only accessible to the methods dealing
// with state transitions. The thread state and flags attributes are used to ensure thread state
// transitions are consistent with the permitted behaviour of the mutex.
//
// *) The most important consequence of this behaviour is that all threads must be in one of the
// suspended states before exclusive ownership of the mutator mutex is sought.
//
class MutatorMutex : public ReaderWriterMutex {
  // 对 Java mutator 线程状态的特殊处理
};
```
- 读写分离：允许多个读者并发，也支持单个写者独占
底层实现
- Linux 下用两个原子计数：state_（-1 表示写锁，≥0 表示读锁数），num_contenders_
非 Linux 下退回到 pthread_rwlock_t

MutatorMutex 是专门用于 GC/线程挂起场景的特化，结合线程状态做共享/独占判断


### 7. 条件变量 ConditionVariable


```cpp
class ConditionVariable {
 public:
  ConditionVariable(const char* name, Mutex& guard);
  void Wait(Thread* self);
  bool TimedWait(Thread* self, int64_t ms, int32_t ns);
  void Signal(Thread* self);
  void Broadcast(Thread* self);
};
```


配合 Mutex 使用，内部：
- Linux 下用一个 AtomicInteger sequence_ + futex 进行等待/重排
- 非 Linux 下用 pthread_cond_t

用途：线程间通知、阻塞与唤醒

### 8. Scoped Locker 辅助类
- MutexLock / ReaderMutexLock / WriterMutexLock
- 构造即加锁，析构即解锁
- 利用 Clang Thread-Safety 注解（ACQUIRE / RELEASE）做静态检查


```cpp
// RAII 示例
{
  MutexLock mu(self, my_mutex);
  // … 临界区 …
} // ~MutexLock() 自动解锁
```

### 9. QuasiAtomic

历史遗留的“准原子”支持，主要用于在不支持原子 64 位操作的平台上：


```cpp
class QuasiAtomic {
 public:
  // 无撕裂读写 64 位
  static int64_t Read64(volatile const int64_t* addr);
  static void Write64(volatile int64_t* addr, int64_t value);

  // 强序 compare-and-swap（仅支持顺序一致）
  static bool CompareAndSwap64SeqCst(volatile int64_t* addr,
                                     int64_t old_value,
                                     int64_t new_value);
  // ...
};
```


- ARM32/i386 上用 ldrexd/strexd 或 movq + 互斥
- 逐步被 C++11 原子 Atomic<T> 取代

### 10. Futex

Futex（Fast Userspace muTEX）是 Linux 为了实现高性能、低开销的用户态锁而提供的一种机制。它的核心思想是在“绝大多数”不发生冲突的情况下，只在用户态完成原子操作；一旦发生冲突（需要等待或唤醒其他线程），再借助一次轻量的内核调用来处理阻塞和唤醒。

#### 基本原理
1. 用户态自旋／原子操作
- 锁定和解锁时，线程首先在用户空间对一个整数（通常是 32 位或 64 位）做原子比较与交换（CAS）。
- 如果 CAS 成功，说明没有竞争，直接返回，整个流程都在用户空间完成，无需系统调用。
2. 内核态等待／唤醒
如果 CAS 失败（说明已经被其他线程持有），线程会调用 futex(addr, FUTEX_WAIT, expected, timeout)：
- addr：指向用户空间的那个整数。
- FUTEX_WAIT：如果 *addr == expected，则将当前线程挂起到内核等待队列，否则立即返回错误。
持有锁的线程在解锁时，会原子地把整数改为 0，然后调用 futex(addr, FUTEX_WAKE, count)：
- FUTEX_WAKE：唤醒等待队列中至多 count 个线程。

这样，只有在真正的竞争（冲突）情况下，才会发生一次内核切换，其它情况都在用户态完成，从而大幅降低上下文切换和调度开销。


#### 常用 Futex 操作
- FUTEX_WAIT

```cpp
int futex(int *uaddr, FUTEX_WAIT, int val, const struct timespec *timeout);
```

如果 *uaddr == val，线程进入睡眠；否则立即返回 -EWOULDBLOCK。

- FUTEX_WAKE

```cpp
int futex(int *uaddr, FUTEX_WAKE, int num_wake);
```

唤醒最多 num_wake 个在该 uaddr 上等待的线程。

- FUTEX_REQUEUE、FUTEX_CMP_REQUEUE 等高级操作，用于在两个 futex 地址之间重排等待队列，常用于实现读写锁、条件变量等。


#### 在 ART 中的应用

在 ART 的 Mutex 实现里，如果检测到 ART_USE_FUTEXES 为真（Linux 平台），它就使用 futex 来管理加锁和等待：

```cpp
void Mutex::ExclusiveUnlock(Thread* self) {
  if (kIsDebugBuild && self != nullptr && self != Thread::Current()) {
    std::string name1 = "<null>";
    std::string name2 = "<null>";
    if (self != nullptr) {
      self->GetThreadName(name1);
    }
    if (Thread::Current() != nullptr) {
      Thread::Current()->GetThreadName(name2);
    }
    LOG(FATAL) << GetName() << " level=" << level_ << " self=" << name1
               << " Thread::Current()=" << name2;
  }
  AssertHeld(self);
  DCHECK_NE(GetExclusiveOwnerTid(), 0);
  recursion_count_--;
  if (!recursive_ || recursion_count_ == 0) {
    if (kDebugLocking) {
      CHECK(recursion_count_ == 0 || recursive_) << "Unexpected recursion count on mutex: "
          << name_ << " " << recursion_count_;
    }
    RegisterAsUnlocked(self);
#if ART_USE_FUTEXES
    bool done = false;
    do {
      int32_t cur_state = state_and_contenders_.load(std::memory_order_relaxed);
      if (LIKELY((cur_state & kHeldMask) != 0)) {
        // We're no longer the owner.
        exclusive_owner_.store(0 /* pid */, std::memory_order_relaxed);
        // Change state to not held and impose load/store ordering appropriate for lock release.
        uint32_t new_state = cur_state & ~kHeldMask;  // Same number of contenders.
        done = state_and_contenders_.CompareAndSetWeakRelease(cur_state, new_state);
        if (LIKELY(done)) {  // Spurious fail or waiters changed ?
          if (UNLIKELY(new_state != 0) /* have contenders */) {
            futex(state_and_contenders_.Address(), FUTEX_WAKE_PRIVATE, kWakeOne,
                  nullptr, nullptr, 0);
          }
          // We only do a futex wait after incrementing contenders and verifying the lock was
          // still held. If we didn't see waiters, then there couldn't have been any futexes
          // waiting on this lock when we did the CAS. New arrivals after that cannot wait for us,
          // since the futex wait call would see the lock available and immediately return.
        }
      } else {
        // Logging acquires the logging lock, avoid infinite recursion in that case.
        if (this != Locks::logging_lock_) {
          LOG(FATAL) << "Unexpected state_ in unlock " << cur_state << " for " << name_;
        } else {
          LogHelper::LogLineLowStack(__FILE__,
                                     __LINE__,
                                     ::android::base::FATAL_WITHOUT_ABORT,
                                     StringPrintf("Unexpected state_ %d in unlock for %s",
                                                  cur_state, name_).c_str());
          _exit(1);
        }
      }
    } while (!done);
#else
    exclusive_owner_.store(0 /* pid */, std::memory_order_relaxed);
    CHECK_MUTEX_CALL(pthread_mutex_unlock, (&mutex_));
#endif
  }
}
```

简化版本

```cpp
void Mutex::ExclusiveUnlock(Thread* self) {
  // 1. 原子地把 state_and_contenders_ 的低位（held 标志）清零
  state_and_contenders_.fetch_and(~kHeldMask, std::memory_order_release);
  // 2. 如果有等待者，则调用 futex 唤醒一个线程
  if (get_contenders() > 0) {
    syscall(SYS_futex, &state_and_contenders_, FUTEX_WAKE, 1, nullptr, nullptr, 0);
  }
}

// 简化伪代码：ExclusiveLockUncontendedFor()
void Mutex::ExclusiveLockUncontendedFor(Thread* self) {
  // 已被持有时自旋，若超时则：
  syscall(SYS_futex, &state_and_contenders_, FUTEX_WAIT, kHeldMask, nullptr, nullptr, 0);
}
```

- state_and_contenders_：用一个 AtomicInteger 存放“锁持有状态”与“等待者计数”。
- 持有者释放锁：清除 held 标志后，如果有等待者，就 FUTEX_WAKE(1) 唤醒一个线程。
- 等待者进入睡眠：读取到 held 标志后，自旋一定次数，仍未获取则 FUTEX_WAIT，直到被唤醒。
