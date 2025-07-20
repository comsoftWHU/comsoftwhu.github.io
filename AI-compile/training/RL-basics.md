---
layout: default
title: 强化学习基础
parent: 深度学习训练框架
grand_parent: AI编译
author: junhuihe
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

最佳排板请看原文：[论文笔记](https://www.notion.so/225d011c2523807d8a3ecdcc3be006d7?source=copy_link)

# 强化学习基础

## 梯度策略定理

Definitions:

策略Policy $\pi_\theta(a\|s)$ ：一个由参数$\theta$控制的策略 $\theta$ ，表示在状态 $s$ 下采取动作 $a$ 的概率：

$$
\pi_\theta (a\|s) = P(A_t=a\|s_t=s; \theta)
$$

轨迹Trajectory $\tau$ ：一个从开始到结束的状态-动作-奖励序列：

$$
\tau = (s_0, a_0, r_0, s_a_r_2, s_2, a_2, \dots, s_T, a_T, r_T) \\
p_\theta(\tau) = p(s_0) \prod_{t=0^T} \pi_\theta(a_t|s_t)p(s_{t+1}|s_t, a_t)
$$

回报Return $R(\tau)$ ：

$$
R(\tau) = \sum_{t=0}^T r_{t+1}
$$

目标函数Objective Function $J(\theta)$ ：

$$
J(\theta) = \mathbb E_{r \sim p_\theta(\tau)} [R(\tau)] = \sum_{\tau} p_\theta (\tau) R(\tau)
$$

Theorem:

对于任意可微策略 $\pi_\theta(a\|s)$ ，其目标函数 $J(\theta)$ 关于策略参数 $\theta$ 的梯度可以表示为：

$$
\begin{aligned}
\nabla_\theta J(\theta) 
& = \mathbb E_{\tau \sim \pi_\theta}[\nabla_\theta \log P_\theta (\tau) R(\tau)] \\ 
& = \mathbb E_{\tau \sim \pi_\theta}[(\sum_{t=0}^T \nabla_\theta \log \pi_\theta (a_t|s_t)) R(\tau)]
\end{aligned}
$$

为什么需要形式二？因为不包含环境动态 $p(s_{t+1}\|s_t, a_t)$ 项，适用于model-free场景

Prove:

$$
\begin{aligned}
\nabla_\theta J(\theta) &= \nabla_\theta \mathbb E_{\tau \sim \pi_\theta} [R(\tau)] \\
& = \nabla_\theta \sum_{\tau}  P_\theta (\tau) R(\tau) \\
& = \sum_{\tau} \nabla_\theta P_\theta (\tau) R(\tau)
\end{aligned}
$$

由于无法直接从 $\nabla_\theta \pi_\theta$中采样，因此使用对数导数技巧：

$$
\nabla_\theta p_\theta(x) = p_\theta(x) \nabla_\theta \log p_\theta(x)
$$

因此

$$
\begin{aligned}
\nabla_\theta J(\theta)
& = \sum_{\tau} (\nabla_\theta P_\theta (\tau)) R(\tau) \\
& = \sum_{\tau} P_\theta(\tau) \nabla_\theta \log P_\theta(\tau) R(\tau) \\
& = \mathbb E_{\tau \sim \pi_\theta}[\nabla_\theta \log P_\theta(\tau) R(\tau)]
\end{aligned}
$$

得到形式一。

形式一到形式二的证明暂不深究，直觉的理解是环境动态和初始状态分布与策略无关，因此梯度为0。

Proxy objective:

对梯度进行积分，可以得到REINFORCE算法的proxy objective:

$$
\begin{aligned}

J(\theta) 
& = \mathbb E_{\tau \sim \pi_\theta}[\log P_\theta(\tau) R(\tau)] \\
& = \mathbb E_{s_t, a_t \sim \pi_\theta}[\log \pi_\theta (a_t|s_t) R(\tau)]

\end{aligned}
$$

## 常见算法

### REINFORCE

策略梯度理论的直接优化

$$
J(\theta) = \mathbb E_{\tau \sim \pi_\theta} [ \sum_{t=0}^T \log \pi_\theta(a_t|s_t) G_t ] \\
G_t = \sum_{t'=t}^{\infty} \gamma^{t'-t} r_{t'}
$$

其中 $\eta$ 为折扣因子。使用未来回报 $G_t$ 替代整条轨迹的回报 $R(\tau)$ 可以降低梯度的方差。

### Actor-Critic

定义优势函数为，在状态 $s_t$ 时，某个动作 $a_t$ 的比较优势

$$
A^\pi (s_t, a_t) = Q^\pi(a_t, s_t) - V^\pi(s_t)
$$

用含参的 $V_\phi$ 来近似价值函数 $V^\pi$ ， $\hat A(s_t, a_t) = r_t + \gamma V_\phi(s_{t+1}) - V_\phi(s_t)$ 来近似优势函数，得到actor model的目标函数为：

$$
J(\theta) = \mathbb E_{s_t, a_t \sim \pi_\theta} [\log \pi_\theta(a_t|s_t) \hat A(s_t, a_t)]
$$

同时使用TD-Error (temporal-difference error)作为标签来训练value model：

$$
\mathcal L = \mathbb E_t [(r_{t} + \gamma \operatorname{sg} [V_\phi(s_{t+1})] - V_\phi(s_t))^2]
$$

直观上，TD-Error衡量的是：我们对 $V_\phi(s_t)$ 的预期，和我们走了一步之后看到的“现实”之间的差距。不直接使用未来收益的蒙特卡洛采样作为训练的ground truth，是为了降低方差。

值得注意的是，value model与actor model要同步训练和更新；value model只能预测**采用当前actor model的前提下**，某个状态的价值。

### PPO

首先通过importance sampling来实现off-policy update：

$$
J(\theta) = \mathbb E_{s_t, a_t \sim \pi_{\theta}} [\hat A(a_t, s_t)] 
= \mathbb E_{s_t, a_t \sim \pi_{\theta_\text{old}}} [\frac{\pi_\theta(a_t, s_t)}{\pi_{\theta_\text{old}}(a_t, s_t)} \hat A(a_t, s_t)]
$$

令 $r_t(\theta) = \frac{\pi_\theta(a_t, s_t)}{\pi_{\theta_\text{old}}(a_t, s_t)}$ ，加入clipping，得到PPO的目标函数：

$$
J(\theta) = \mathbb E_{s_t, a_t \sim \pi_{\theta_\text{old}}} \left[\min \left( r_t(\theta) \hat A(a_t, s_t), \operatorname{clip}[r_t(\theta), 1-\epsilon, 1+\epsilon]  \hat A(a_t, s_t)  \right)\right]
$$

其中clip是为了保证正向更新（增加reward的更新）不要超出信任域，避免在一次采样中进行过于激进的更新；min是为了保证假如出现了负向的更新（减少reward的更新），梯度不要受到clip影响进入死区。