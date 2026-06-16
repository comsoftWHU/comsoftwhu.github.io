---
layout: default
title: eBPF机制
nav_order: 5
parent: Linux内核
author:  Anonymous Committer
---

## 1. 背景：BPF 是什么？

**BPF** 全称是 **Berkeley Packet Filter**，最早主要用于网络包过滤。

> 内核提供一个小型、安全的“过滤程序执行环境”，让用户可以把一段规则放进内核里，对网络包进行判断。

早期典型场景是 `tcpdump`。比如你执行：

```bash
tcpdump port 80
```

如果没有 BPF，内核可能要把大量网络包都拷贝到用户态，然后 `tcpdump` 再过滤。这样很浪费。

有了 BPF 后，可以先把“只要 80 端口的数据包”这个过滤逻辑放到内核里执行。内核只把符合条件的包交给用户态程序，效率更高。

Linux 内核文档现在把 BPF 文档重点放在 **extended BPF，也就是 eBPF** 上。

---

## 2. 问题：传统内核扩展方式有什么痛点？

操作系统内核很强大，但也很敏感。

假设你想做这些事情：

* 观察某个进程为什么慢；
* 统计每个容器的网络流量；
* 拦截异常系统调用；
* 做高性能负载均衡；
* 在内核路径上采集指标。

传统方式通常有几个选择。

第一种是**改内核源码**。问题是成本高、风险大、升级困难。

第二种是**写内核模块**。问题是内核模块权限很高，一旦写错，可能导致内核崩溃，甚至带来安全漏洞。

第三种是在**用户态采集数据**。问题是用户态和内核态之间切换成本高，很多细粒度事件也不容易拿到。

所以核心矛盾是：

> 我们想扩展内核能力，但又不想频繁改内核、不想重启、不想引入不安全的内核模块。

---

## 3. 解决方案：eBPF 是什么？

**eBPF** 可以理解为 BPF 的增强版。现在它已经不只是“网络包过滤器”，而是 Linux 内核里的一个通用可编程机制。

官方 eBPF 文档对它的定义大致是：eBPF 可以在操作系统内核这样的特权上下文中运行沙箱程序，用来安全、高效地扩展内核能力，而不需要修改内核源码或加载传统内核模块。

它的工作方式可以简化成这样：

```text
用户写 eBPF 程序
        ↓
编译成 eBPF 字节码
        ↓
加载到内核
        ↓
Verifier 安全检查
        ↓
JIT 编译 / 解释执行
        ↓
挂到某个内核事件点上运行
```

关键组件有几个。

### 1）Hook：挂载点

eBPF 程序不会凭空运行，它要挂在某个事件点上。

常见 hook 包括：

* 网络包进入或离开网卡；
* 某个系统调用发生；
* 某个内核函数被调用；
* 某个用户态函数被调用；
* 性能事件触发。

也就是说，eBPF 像是在内核关键路径上放了一些“可编程探针”。

---

### 2）Verifier：安全检查器

这是 eBPF 很核心的地方。

程序加载进内核前，内核会先验证它是否安全。比如检查它会不会越界访问、会不会无限循环、会不会非法访问内核内存等。eBPF 官方文档也强调，程序加载进内核前会经过 verification 阶段，以确保程序安全运行。

这解决了一个很大的问题：

> 允许用户扩展内核，但不能让用户随便把内核搞崩。

---

### 3）Maps：内核态和用户态共享数据

eBPF 程序运行在内核中，但结果通常要给用户态程序看。

比如：

* 某个进程调用了多少次 `open()`；
* 某个 IP 发了多少包；
* 某个容器的延迟是多少。

这些数据可以存在 **eBPF Map** 里。用户态程序可以读取 map，eBPF 程序也可以更新 map。

可以简单理解成：

```text
eBPF 程序负责在内核里采集 / 处理
用户态程序负责读取 / 展示 / 控制
Map 是两者之间的共享数据结构
```

---

### 4）JIT：接近原生性能

eBPF 字节码可以被 JIT 编译成机器码执行。Linux 内核文档提到，eBPF 的设计目标之一就是支持高效 JIT，使其性能可以接近原生编译代码。

所以它不是简单的“脚本解释执行”，而是面向高性能内核路径设计的。

---

## 4. 效果：eBPF 带来了什么？

### 第一，观测能力大幅增强

以前排查系统问题，很多时候只能看到用户态日志，或者靠比较重的 tracing 工具。

有了 eBPF，可以在内核层面观察：

```text
进程 → 系统调用 → 网络 → 文件 → 调度 → TCP 栈
```

比如你可以回答：

* 哪个进程在疯狂创建连接？
* 哪个系统调用耗时最高？
* 哪个容器丢包严重？
* 某个请求慢到底慢在用户态、内核态还是网络路径？

---

### 第二，网络性能和灵活性提升

eBPF 很常见的一个方向是高性能网络。

比如 Kubernetes 网络组件 Cilium 就是基于 eBPF 做网络、安全和可观测性的开源项目。Cilium 官方描述它用于云原生环境中的 networking、security 和 observability。

在这类场景里，eBPF 可以用于：

* 高性能转发；
* 负载均衡；
* 网络策略；
* 服务可观测性；
* 替代部分传统 iptables 路径。

---

### 第三，安全能力增强

eBPF 可以观察和限制系统行为，例如：

* 监控异常系统调用；
* 检测容器逃逸行为；
* 记录敏感文件访问；
* 结合策略系统做运行时安全。

不过也要注意：eBPF 本身能力很强，所以它通常需要权限控制。官方文档也提到，加载 eBPF 程序需要满足权限要求，非特权 eBPF 是否可用取决于系统配置。

---

### 第四，不需要频繁改内核

这是 eBPF 的核心价值之一。

传统方式：

```text
想扩展内核能力
→ 改内核 / 写内核模块
→ 编译
→ 部署
→ 重启或高风险加载
```

eBPF 方式：

```text
写 eBPF 程序
→ 加载到内核
→ verifier 检查
→ 挂到事件点
→ 运行
```

也就是说，eBPF 让内核变得“可编程”，但又通过 verifier、权限控制、沙箱机制尽量保证安全。

## 5. 简单示例：用 eBPF 观察谁在执行命令

我们用一个很直观的例子：

> 每当系统里有进程执行新命令时，把进程名、PID、执行的文件路径打印出来。

这类事件对应 Linux 的 `execve` 系统调用。比如你在终端执行：

```bash
ls
python app.py
curl example.com
```

这些都会触发 `execve`。

可以用 `bpftrace` 写一个非常短的 eBPF 程序：

```bash
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_execve
{
    printf("process=%s pid=%d exec=%s\n", comm, pid, str(args->filename));
}
'
```

它的含义是：

```text
tracepoint:syscalls:sys_enter_execve
```

表示把 eBPF 程序挂到 `execve` 系统调用入口。

```text
comm
```

表示当前进程名。

```text
pid
```

表示当前进程 ID。

```text
args->filename
```

表示这次要执行的程序路径。

运行之后，另开一个终端执行：

```bash
ls
whoami
date
```

你可能会看到类似输出：

```text
process=bash pid=12345 exec=/usr/bin/ls
process=bash pid=12346 exec=/usr/bin/whoami
process=bash pid=12347 exec=/usr/bin/date
```

这个例子体现了 eBPF 的核心思想：

```text
用户态写一段小程序
        ↓
加载到内核
        ↓
挂到某个内核事件点
        ↓
事件发生时自动执行
        ↓
把结果输出给用户态
```

对应到前面的四点：

```text
背景：我们想观察内核里的系统调用行为
问题：传统方式很难低成本、低侵入地看到这些事件
解决方案：用 eBPF 挂到 execve 这个 tracepoint 上
效果：不用改内核、不用写内核模块，就能实时看到进程执行行为
```

所以这个例子可以总结成一句话：

> eBPF 让我们可以在内核事件发生时，安全地插入一段小程序，实时观察或处理系统行为。

## 6. 生产环境例子：Kubernetes 集群用 eBPF 做网络、负载均衡和可观测性

一个非常典型的生产场景是：

> 公司有一个 Kubernetes 集群，里面跑了几百到几千个 Pod。服务之间调用频繁，网络策略复杂，还需要排查“某个服务为什么访问慢 / 为什么连接失败”。

这时可以使用 **Cilium** 这类基于 eBPF 的 Kubernetes 网络方案。Cilium 官方定位就是为 Kubernetes 等云原生环境提供 networking、security、observability；它使用 eBPF 作为核心数据平面。

---

### 1. 背景

假设线上有这样一套系统：

```text
用户请求
  ↓
Ingress / Gateway
  ↓
order-service
  ↓
payment-service
  ↓
inventory-service
  ↓
database
```

这些服务都跑在 Kubernetes 里，每个服务可能有多个 Pod。

比如：

```text
order-service-1
order-service-2
order-service-3

payment-service-1
payment-service-2
```

Kubernetes 需要解决几个基础问题：

```text
服务发现
负载均衡
Pod 到 Pod 通信
网络策略
流量可观测性
```

传统 Kubernetes 里，很多 Service 转发依赖 `kube-proxy`，常见实现方式是 iptables 或 IPVS。

---

### 2. 问题

当集群规模变大后，传统网络路径会遇到一些问题。

比如，一个线上订单服务偶尔报错：

```text
order-service 调 payment-service 超时
```

运维和开发需要回答几个问题：

```text
请求到底有没有发出去？
被哪个网络策略拦了？
连接到了哪个 payment-service Pod？
延迟发生在客户端、服务端，还是网络路径？
某个节点是不是丢包？
```

如果只看应用日志，通常不够。

因为应用日志只能看到：

```text
payment request timeout
```

但它不一定能告诉你：

```text
包在内核网络栈哪里被处理
Service 转发到了哪个后端 Pod
哪条网络策略放行或拒绝了流量
TCP 重传是否变多
```

另一个问题是性能。

如果集群里 Service、Pod、Endpoint 很多，传统 iptables 规则会变得复杂，网络转发路径和排障难度都会上升。Cilium 文档中就有专门的 “Kubernetes Without kube-proxy” 模式，用 Cilium 替代 kube-proxy。

---

### 3. 解决方案：用 eBPF 接管部分内核网络路径

在这个生产场景里，Cilium 会在每个 Kubernetes 节点上运行 Agent，并把 eBPF 程序加载到 Linux 内核的网络路径中。

简化后是这样：

```text
Pod 发出请求
  ↓
Linux 内核网络路径
  ↓
eBPF 程序执行
  ↓
判断目标 Service
  ↓
选择后端 Pod
  ↓
执行转发 / 策略检查 / 观测记录
```

也就是说，eBPF 程序可以直接在内核里做这些事情：

```text
Service 负载均衡
网络策略判断
流量转发
连接跟踪
指标采集
流量日志
```

Cilium 的 GitHub 描述中也提到，它使用 eBPF-based dataplane，支持跨集群网络、L3/L4/L7 策略，并能用 eBPF 替代 kube-proxy，实现分布式负载均衡。

---

### 4. 效果

#### 效果一：请求路径更清楚

以前排查问题可能是这样：

```text
应用日志 → 容器日志 → 节点日志 → iptables → tcpdump
```

用了 eBPF 之后，可以直接观察服务之间的网络流。

例如：

```text
order-service → payment-service
状态：FORWARDED
协议：HTTP
延迟：35ms
源 Pod：order-service-7b9c
目标 Pod：payment-service-6f2a
```

Cilium 生态里的 Hubble 就是用来做网络可观测性的组件，Cilium 文档也把 observability、flow logs、metrics、tracing 作为其重要能力。

---

#### 效果二：网络策略更直观

比如生产要求：

```text
order-service 可以访问 payment-service
frontend 可以访问 order-service
其他服务不能直接访问 payment-service
```

传统方式可能要写很多网络规则，而且排查困难。

使用 Cilium 后，可以基于服务身份做策略，而不是只依赖 IP。

例如逻辑上是：

```text
允许：
order-service → payment-service

拒绝：
unknown-service → payment-service
```

这对 Kubernetes 很重要，因为 Pod IP 经常变化，但服务身份相对稳定。

---

#### 效果三：可以替代 kube-proxy

在一些生产集群中，会启用 Cilium 的 kube-proxy replacement 模式。

传统路径大概是：

```text
Pod
 ↓
Service IP
 ↓
kube-proxy / iptables
 ↓
后端 Pod
```

使用 eBPF 后可以变成：

```text
Pod
 ↓
eBPF Service 负载均衡
 ↓
后端 Pod
```

Cilium 官方文档明确提供了不用 kube-proxy、由 Cilium 替代它的部署模式。

---

#### 效果四：低侵入

这点很重要。

生产环境里你通常不希望：

```text
修改业务代码
重启所有服务
给每个服务加 sidecar
手动抓包
加载高风险内核模块
```

eBPF 的好处是，很多观测和网络能力可以发生在内核路径里，对业务应用比较透明。

也就是说，业务代码仍然是：

```python
requests.post("http://payment-service/pay")
```

但底层的服务发现、负载均衡、策略判断、流量观测，可能已经由 eBPF 在内核里完成了。

---

### 5. 一个具体生产故障例子

假设线上出现报警：

```text
payment-service 错误率升高
order-service 调用 payment-service 超时
```

没有 eBPF 时，排查可能是：

```text
1. 看 order-service 日志
2. 看 payment-service 日志
3. 查 Kubernetes Service
4. 查 Endpoint
5. 查节点网络
6. tcpdump 抓包
7. 查 iptables 规则
```

有 eBPF / Cilium / Hubble 后，可以直接看流量路径：

```text
order-service-7b9c → payment-service-6f2a
状态：DROPPED
原因：Policy denied
```

这时问题就很明确：

> 不是 payment-service 挂了，而是网络策略把流量拦掉了。

或者你看到：

```text
order-service-7b9c → payment-service-6f2a
状态：FORWARDED
延迟：900ms
TCP retransmission increased
```

这说明流量没有被拒绝，但网络质量或目标 Pod 处理能力可能有问题。

再比如看到：

```text
order-service-7b9c → payment-service-9a21
状态：FORWARDED
HTTP status：500
```

那就说明网络层正常，问题更可能在 payment-service 应用逻辑里。

---

### 这个例子里的 eBPF 价值

可以总结成一句话：

> 在生产 Kubernetes 集群里，eBPF 可以把原本黑盒的内核网络路径变成可编程、可观测、可控制的数据平面。

对应到实际效果：

```text
网络转发更高效
服务负载均衡更灵活
网络策略更细粒度
故障排查更快
对业务代码侵入更低
```

所以，eBPF 在生产环境里不是单纯“看系统调用”的小工具，而是可以成为云原生网络、安全和可观测性的底层基础设施。