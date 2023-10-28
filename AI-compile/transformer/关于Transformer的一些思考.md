---
layout: default
title: Transformer入门
nav_order: 2
parent: Transformer
grand_parent: AI编译
author: zeonfaiho
---

{% assign author = site.data.authors[page.author] %}
<div> 作者: {{ author.name }}  
 邮箱：{{ author.email }}
</div>

<script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.7/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

<script type="text/x-mathjax-config">
  MathJax.Hub.Config({
    tex2jax: {
      inlineMath: [['$','$'], ['\\(','\\)']],
      processEscapes: true
    }
  });
</script>

# Transformer入门

## 模型结构

Transformer模型大致分为encoder only、decoder only、encoder-decoder这三类，其中encoder only模型主要用于处理图像和音频；decoder用于生成文字。我们主要讨论decoder模型。

decoder only模型结构如下图所示，由input、output和若干个串联的decoder block组成，

<img src="./%E5%85%B3%E4%BA%8ETransformer%E7%9A%84%E4%B8%80%E4%BA%9B%E6%80%9D%E8%80%83.assets/image-20231026001827850.png" alt="image-20231026001827850" style="zoom: 50%;" />

Transformer一个很重要的特点是其中间张量具有dynamic shape，因此在不同的工作状态下，张量的形状会有所不同。因此我们需要分别讨论不同的工作状态。

decoder only模型有两种工作状态：

- Prefill phase：称为预处理或encoding；读取一段prompt，计算并缓存每一层的key和value，这个缓存称为KV Cache
- Decode phase：生成新的token的阶段，每次生成一个token；具体来说，将上次生成的token用作这次推理的输入，得到新的一个token；新的token作为输入再进行推理，得到下一个token；直到获得<|end_of_text|>

首先讨论更为典型的decode phase

- input只包含一个矩阵乘法算子，输入用1-hot编码表示上一个生成的token，大小是$\text{n\_vocab} \times 1$，；输出形状是$d \times 1$
- 每个decoder block的输入和输出都是形状为$d \times 1$
- output是input的逆过程，输入是$d \times 1$，输出是$\text{n\_vocab} \times 1$，表示下一个token的概率分布；采样程序从概率高的vocab中选择一个作为模型输出

![image-20231025234640693](./%E5%85%B3%E4%BA%8ETransformer%E7%9A%84%E4%B8%80%E4%BA%9B%E6%80%9D%E8%80%83.assets/image-20231025234640693.png)

decoder block

- 每个decoder block的输入是一个$[d \times 1]$的张量
- 首先进入Multi-Head Attention模块，这个张量与$W_q$、$W_k$、$W_v$相乘，得到3个$d \times 1$的张量；接着这三个矩阵都进行垂直的切分，各自变为$h$个$\frac d h \times 1$的张量（$h$称为注意力头数量），其中第$i$个分别为$q_i$、$k_i$、$v_i$
- 假设之前序列长度为$l-1$，则kv cache中，第$i$个注意力头对应的k cache和v cache形状均为$\frac d h \times (l-1)$；$k_i$与kv cache进行拼接，得到$\frac d h \times l$的张量，称为$K_i$和$V_i$
- $\text{softmax}(q_iK_i)$得到形状为$l \times 1$的张量，称为attention score
- $\text{attn\_score}_i \times V_i$得到$output_i$，形状为$\frac d h \times 1$；连接得到$output$（$d \times 1$）
- 乘上$W_\text{out}$、归一化、残差连接

FFN

- 向高维度投影后再投影回低纬度：$[d \times 1] \to [n\_ff \times 1] \to [d \times 1]$
- 残差连接

分析

- FLOPs: 浮点操作数量
- MOPs: 内存操作数量

$$
\text{MOPs} = \sum_{op}\left(\sum_{in\_tensor \in op}|in\_tensor| + \sum_{out\_tensor \in op} |out\_tensor|\right)
$$

- Arithmetic intensity

  ![image-20231027143428285](./%E5%85%B3%E4%BA%8ETransformer%E7%9A%84%E4%B8%80%E4%BA%9B%E6%80%9D%E8%80%83.assets/image-20231027143428285.png)

decode phaseFLOPs和MOPs分析

![image-20231027143212851](./%E5%85%B3%E4%BA%8ETransformer%E7%9A%84%E4%B8%80%E4%BA%9B%E6%80%9D%E8%80%83.assets/image-20231027143212851.png)

![image-20231026004231902](./%E5%85%B3%E4%BA%8ETransformer%E7%9A%84%E4%B8%80%E4%BA%9B%E6%80%9D%E8%80%83.assets/image-20231026004231902.png)

注意到在decode phase，模型的arithmetic intensity都在2左右；对比常见GPU的浮点性能和访存性能，以RTX4060为例：

- 浮点性能：15.39TFLOPs
- 访存带宽：使用GDDR6显存，带宽在768GBps左右

浮点性能比访存带宽高出20-30倍，因此使用GPU进行decode phase推理的性能瓶颈在于IO

接下来讨论prefill phase，假设当前要解码的序列长度为$n$，计算流程和上面的decode phase没有差异，唯一的区别在于所有值为1的张量维度变成$n$；其结果是：
$$
\text{arithmetic\_intensity} \approx n
$$
因此encode阶段处理每个token的平均时间比decode阶段短得多

![image-20231027154753947](./%E5%85%B3%E4%BA%8ETransformer%E7%9A%84%E4%B8%80%E4%BA%9B%E6%80%9D%E8%80%83.assets/image-20231027154753947.png)

## 主要困难

- Latency
  - IO
    - Flash attention（主要用于训练）：
      - V1：https://proceedings.neurips.cc/paper_files/paper/2022/file/67d57c32e20fd0a7a302cb81d36e40d5-Paper-Conference.pdf
      - V2：https://arxiv.org/pdf/2307.08691.pdf
    - Speculative decoding：[Accelerating large language model decoding with speculative sampling](https://arxiv.org/abs/2302.01318)
    - Continuous batching：[Orca: A Distributed Serving System for Transformer-Based Generative Models](https://www.usenix.org/conference/osdi22/presentation/yu)
  - Parallization
    - Flash decoding：https://crfm.stanford.edu/2023/10/12/flashdecoding.html
- Memory
  - Quantization
    - 4-bit GPTQ：https://arxiv.org/abs/2210.17323
  - Sparsification
    - ANT & Olive：[OliVe: Accelerating Large Language Models via Hardware-friendly Outlier-Victim Pair Quantization](https://dl.acm.org/doi/abs/10.1145/3579371.3589038)
  - Paging：[Efficient memory management for large language model serving with pagedattention](https://arxiv.org/abs/2309.06180)
- Long context
  - Sparse attention：https://arxiv.org/abs/1904.10509

---

以下是私货部分

### Long Context

为什么关注long context

- 随着sequence length的增长，MHA (act-to-act)的FLOPs和MOPs会成正比增长
  - kv cache也会成正比增长：在llama-2-7b上，一个token对应512K参数（256KB内存）；32k token对应16B参数（8GB内存），GPU memory装不下

- 优化latency需要非常熟悉GPU或者加速硬件的底层细节
- 优化memory离不开量化和剪枝，需要非常了解模型的特性

| Sequence Length | Operator                 | FLOPs per token (× 10^6) | MOPs per token (× 10^6) |
| --------------- | ------------------------ | ------------------------ | ----------------------- |
| 128             | MHA (projections)        | 56.64                    | 28.36                   |
|                 | MHA (act-to-act matmuls) | 2.34                     | 1.25                    |
|                 | FFN (projections)        | 113.28                   | 56.72                   |
|                 | Other                    | 0.55                     | 0.23                    |
|                 | Total                    | 172.81                   | 86.56                   |
| 512             | MHA (projections)        | 56.62                    | 28.39                   |
|                 | MHA (act-to-act matmuls) | 9.43                     | 4.79                    |
|                 | FFN (projections)        | 113.25                   | 56.73                   |
|                 | Other                    | 0.68                     | 0.27                    |
|                 | Total                    | 179.98                   | 90.17                   |
| 4096            | MHA (projections)        | 56.65                    | 28.39                   |
|                 | MHA (act-to-act matmuls) | 75.50                    | 38.04                   |
|                 | FFN (projections)        | 113.28                   | 56.72                   |
|                 | Other                    | 1.71                     | 0.79                    |
|                 | Total                    | 246.13                   | 123.86                  |
| 32768           | MHA (projections)        | 56.7                     | 28.39                   |
|                 | MHA (act-to-act matmuls) | 604                      | 302                     |
|                 | FFN (projections)        | 113.3                    | 56.7                    |
|                 | Other                    | 3                        | 1.5                     |
|                 | Total                    | 777                      | 388.6                   |

---

想法：如何尽可能高效地利用硬件？在本地部署transformer的一个问题是，当一轮对话结束后，需要等待用户的下一次输入，这期间的空闲时间能否利用起来？

 [Lora: Low-rank adaptation of large language models](https://arxiv.org/abs/2106.09685)：finetune的$\Delta W$是low rank（低秩）的

[Why can gpt learn in-context? language models secretly perform gradient descent as meta optimizers](https://arxiv.org/abs/2212.10559)：finetune和ICL本质上统一

猜想：context的某种形式也是低秩的

- 实验1：kv cache

  - 结果：k和v都不是low rank的

- 实验2：QK（attention score）

  - 结果：QK是低秩的

    <img src="./%E5%85%B3%E4%BA%8ETransformer%E7%9A%84%E4%B8%80%E4%BA%9B%E6%80%9D%E8%80%83.assets/file-cAGu5Ym9DMQTqJqv75OA7b8y" alt="img" style="zoom: 33%;" />

  - 初步想法：令$X=QK$，$X'=AB$，则$K'=Q^+AB=(Q^+A)(B)$实现近似；同理可以对$V$近似：$O=VX \approx VAB$，所以$V'=(AB)^+O=(B^+A^+)O=(B^+)(A^+O)$实现近似

  - 问题：

    - $X$和$K$都是变长的，每次推理后都进行分解的计算开销很大
      - work around：执行完一轮对话之后进行一次压缩
    - 与sparsification相比优势不显著
      - 不会完全丢弃信息，只会引入误差（是否更好？）
      - 位置编码：ALiBi等相对位置编码需要知道token之间的距离，假如使用稀疏注意力需要额外保存这一信息，导致开销

---

退而求其次：类似于speculative decoding的方法

- 先用更小的context length执行若干次推理
- 再用full context进行一次验证
  - 为什么模型天然可以执行validation？因为输入$l$个token时，模型的输出是$l \times \text{n\_vocab}$，其中第$(i, j)$个元素表示第$i$个token的下一个token是$j$的概率
  - 因此只需要用某个token和上一个输出对比即可，假如相应的概率很小，说明两次推理结果发生冲突，采用validation的结果，并从抛弃这个token之后的结果，重新推理
  - 可以在异构设备上进行，比方说CPU进行inference，GPU进行validation，充分利用GPU浮点性能强的优点

---

是否可以用StreamingLLM实现infinite context？https://arxiv.org/abs/2309.17453