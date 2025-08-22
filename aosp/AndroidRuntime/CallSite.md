---
layout: default
title: CallSite是什么
nav_order: 8
parent: AndroidRuntime
grand_parent: AOSP
author: Anonymous Committer
---

## Call Site

在DEX文件中，`Call Site`（调用点）是一个核心概念，它与 `invoke-dynamic` 和 `invoke-custom` 这两条指令紧密相关，是实现**动态语言特性**和**现代Java/Kotlin高级功能**的关键机制。

简单来说，一个 `Call Site` 就像一个**可编程的、延迟绑定的“插座”**。普通的函数调用就像一个焊死的连接，编译时就确定了要调用哪个具体方法。而一个 `Call Site` 则是一个插座，第一次使用时，它会通过一个特殊的“引导方法”（Bootstrap Method）来决定到底应该“插入”哪个具体的方法，并在后续调用中直接使用这个方法。

---

## Call Site 的详细解析

### 1. 它是做什么的？

`Call Site` 的主要目的是**将方法调用的“解析”过程从编译时推迟到运行时**。对于一个普通的 `invoke-virtual` 或 `invoke-static` 调用，编译器在生成DEX文件时就已经将调用的目标方法链接好了。

但对于 `invoke-dynamic` 指令，DEX文件中只记录了一个 `Call Site` 的索引。这个 `Call Site` 包含了解析调用所需的所有元信息，但不包含最终要执行的方法。

### 2. Call Site 在DEX文件中的结构

一个 `Call Site` 在DEX文件（版本 038+）中由 `call_site_id_item` 结构体表示。它本质上是一个指向 `encoded_array_item`（编码数组）的索引，这个数组包含了以下关键信息：

1. **引导方法句柄 (Bootstrap Method Handle)**：这是一个指向特殊静态方法（即引导方法）的方法句柄（`MethodHandle`）。
2. **动态方法名 (Dynamic Method Name)**：要调用的方法名，例如 `lambda$main$0`。
3. **动态方法类型 (Dynamic Method Type)**：要调用的方法的类型签名（即参数和返回值的类型）。
4. **额外参数 (Optional Arguments)**：传递给引导方法的一些静态参数。

---

## 结合代码示例：Lambda表达式的实现

为了更具体地理解，我们来看一个简单的Lambda表达式是如何通过 `Call Site` 实现的。

### 1. Java 源码

假设我们有这样一段简单的Java代码：

```java
public class Test {
    public static void main(String[] args) {
        // 一个简单的Lambda表达式，实现了Runnable接口
        Runnable r = () -> System.out.println("Hello, Call Site!");
        r.run();
    }
}
```

### 2. DEX 字节码 (Smali 格式)

编译器不会为这个Lambda生成一个单独的 `Test$1.class` 匿名内部类文件。取而代之，它会在 `main` 方法中生成一条 `invoke-dynamic` 指令：

```smali
.method public static main([Ljava/lang/String;)V
    .registers 2

    # 这就是 invoke-dynamic 指令
    # 它引用了一个 Call Site (call_site_0) 和一个引导方法 (bsm_0)
    invoke-dynamic {}, LTest;->bsm_0()Ljava/lang/Runnable;@call_site_0

    move-result-object v0
    
    # v0 现在是实现了 Runnable 接口的对象
    # 调用 run() 方法
    invoke-interface {v0}, Ljava/lang/Runnable;->run()V

    return-void
.end method

# 编译器会生成一个私有的静态方法作为Lambda的实际执行体
.method private static synthetic lambda$main$0()V
    .registers 2
    
    sget-object v0, Ljava/lang/System;->out:Ljava/io/PrintStream;
    const-string v1, "Hello, Call Site!"
    invoke-virtual {v0, v1}, Ljava/io/PrintStream;->println(Ljava/lang/String;)V
    
    return-void
.end method
```

在这个字节码中，`invoke-dynamic` 指令是核心。它不指向任何具体的方法，而是指向一个 `call_site_0`。

```smali
invoke-dynamic {}, LTest;->bsm_0()Ljava/lang/Runnable;@call_site_0
```

- `{}` 参数寄存器：花括号内是传递给“动态调用”的参数列表。在这里， `{}` 表示空，意味着我们创建这个 Lambda 实例 r 的过程，并不需要从当前 main 方法的上下文中捕获任何变量。如果 Lambda 捕获了一个局部变量（例如，一个在 main 方法中定义的 int），那么存放那个变量的寄存器（如 v1）就会出现在这里，形式如 `invoke-dynamic {v1}, ...`。

- `LTest;->bsm_0()Ljava/lang/Runnable;` 调用点的“外观”签名 (Call Site Specifier)：这部分看起来像一个普通的方法签名，但它描述的不是引导方法，而是这个 `invoke-dynamic` 指令本身的“外观”。它为虚拟机的类型检查系统提供了必要信息。

- `LTest;->bsm_0`: `bsm_0` (bootstrap method 0) 是编译器为这个动态调用点生成的一个唯一的、符号性的名称。这个名字本身没有特殊含义，只是用来在 DEX 文件内部进行引用。LTest; 指明了这个符号属于 Test 类。

- `()Ljava/lang/Runnable;`: 这是该调用的方法类型签名。() 表示这个调用不接受参数（与第2点的 {} 对应），`Ljava/lang/Runnable`; 表示这个调用的返回值必须是一个实现了 Runnable 接口的对象。ART 在执行前会验证这一点。

- `@call_site_0` Call Site 索引：这是最关键的部分。它是一个直接的引用，指向 DEX 文件中定义的 call_site_id_item 列表的第 0 个条目（即 call_site_0）。ART 虚拟机就是通过这个索引找到 Call Site 的所有元数据——包括真正的引导方法 (LambdaMetafactory.metafactory)、要实现的方法名 ("run")、以及传递给引导方法的额外参数等。

### 3\. Call Site 和引导方法

`call_site_0` 在DEX文件中的定义会指向一个**引导方法**（Bootstrap Method, BSM）。对于Java Lambda，这个引导方法通常是 `java.lang.invoke.LambdaMetafactory.metafactory`。

DEX文件中的 `call_site_0` 条目大致包含了这些信息：

- **引导方法句柄**: 指向 `LambdaMetafactory.metafactory(...)`。
- **动态方法名**: `"run"` (要实现的接口方法名)。
- **动态方法类型**: `()Ljava/lang/Runnable;` (表示这个调用会返回一个`Runnable`对象)。
- **额外参数**:
  - 要实现的函数式接口 (`java.lang.Runnable`)。
  - Lambda方法体 (`lambda$main$0`) 的方法句柄。

## 工作流程：`invoke-dynamic` 的“解析与调用”两步走

现在，我们将代码示例和工作流程联系起来：

1. **解析 (Resolution)**：

      - ART第一次执行到 `invoke-dynamic` 指令。
      - 它发现 `call_site_0` 尚未链接，于是调用其指定的引导方法 `LambdaMetafactory.metafactory(...)`。
      - `LambdaMetafactory` 就像一个**工厂**，它会接收DEX中记录的元信息，然后在内存中**动态地生成一个轻量级的类**。这个类实现了 `Runnable` 接口，并且其 `run` 方法直接调用我们真正的逻辑 `lambda$main$0`。
      - 最后，工厂返回一个 `java.lang.invoke.CallSite` 对象，这个对象“包裹”了刚刚创建的这个动态类的实例。

2. **链接 (Linking)**：

      - ART将 `invoke-dynamic` 指令与这个返回的 `Call Site` 对象进行“链接”。现在，这个 `Call Site` 就像一个已经插入了正确电器的“插座”。

3. **调用 (Invocation)**：

      - `invoke-dynamic` 指令执行的结果，就是那个动态生成的、实现了`Runnable`接口的对象。这个对象被 `move-result-object v0` 存入了`v0`寄存器。
      - 后续 `invoke-interface {v0}` 就能正确执行Lambda的逻辑了。
      - 如果代码**第二次**执行到同一条 `invoke-dynamic` 指令，ART会直接返回**第一次链接好的那个 `Call Site` 对象**，跳过所有解析步骤，效率极高。

## Call Site 的实际用途是什么？

`Call Site` 和 `invoke-dynamic` 机制是ART（以及JVM）支持现代编程语言特性的基石。

1. **实现 Lambda 表达式**: 如上例所示，编译器不再为每个Lambda生成一个匿名内部类（这会增加APK体积），而是通过`invoke-dynamic`在运行时动态生成。这大大减少了APK中的类文件数量，优化了性能。

2. **支持动态语言**: 对于像JRuby、Jython这类运行在Java虚拟机上的动态语言，`Call Site` 允许它们实现自己的动态方法分发和缓存策略，极大地提升了性能。

3. **Java 9+ 的字符串拼接**: 从Java 9开始，`javac` 会将字符串拼接操作（如 `"a" + b + "c"`）编译成 `invoke-dynamic` 指令。其引导方法会使用 `StringConcatFactory` 在运行时选择最高效的拼接策略（如 `StringBuilder`），比旧版本的实现更灵活、性能更好。
