---
layout: default
title: DeepSpeed阅读笔记
parent: 深度学习训练框架
grand_parent: AI编译
author: junhuihe
---

{% assign author = site.data.authors[page.author] %}
<div> 作者: {{ author.name }}  
 邮箱：{{ author.email }}
</div>

[ZeRO: Memory Optimizations Toward Training Trillion Parameter Models](http://arxiv.org/abs/1910.02054)

- DP
    - 每个gpu都维护权重、梯度、优化器的完整副本
    - Memory:
        - Weight: n
        - Activation: x
        - Gradient: n
        - Full precision weight: n
        - First-order momentum: n
        - Second-order momentum: n
    - Communication:
        - Gradient: 2n (all-reduce)
- Zero-1:
    - 每个gpu都维护一个权重和梯度的副本；所有gpu协作维护一个优化器副本；注意优化器的切分是以tp而不是mp的方式进行的，以保证scatter-reduce时每台机器的通讯量相同
    - Memory:
        - Weight: n
        - Activation: x
        - Gradient: n
        - Full precision weight: n/d
        - First-order momentum: n/d
        - Second-order momentum: n/d
    - Communication
        - Gradient: n (scatter-reduce)
        - Updated weight: n (all-broadcast)
- Zero-2:
    - 每个gpu仅维护一个权重副本；梯度在计算之后就立刻发送给它的优化器所在的gpu
    - Memory:
        - Memory:
            - Weight: n
            - Activation: x
            - Gradient: n/d
            - Full precision weight: n/d
            - First-order momentum: n/d
            - Second-order momentum: n/d
        - Communication
            - Gradient: n (per-tensor scatter-reduce)
            - Updated weight: n (all-broadcast)
- Zero-3
    - 连同权重一起tp切分放在对应机器上，forward时all-gather将每一个权重的切片同步给所有机器
    - Memory:
        - Weight: n/d
        - Activation: x
        - Gradient: n/d
        - Full precision weight: n/d
        - First-order momentum: n/d
        - Second-order momentum: n/d
    - Communication
        - Weight: n (all-gather)
        - Gradient: n (scatter-reduce)
        - Updated weight: n (all-broadcast)
- Zero-R
    - 在有tp的情况下，将activation切片保存到tp group的其中一台机器上，避免冗余的副本