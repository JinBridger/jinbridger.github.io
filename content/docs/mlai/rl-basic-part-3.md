---
title: "强化学习入门 - 参数化策略"
weight: 1
date: 2025-09-18
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

> [!INFO]
> 本文为强化学习入门笔记的其中一部分, 其他笔记参见:
> 1. [强化学习入门 - 基础定义]({{< ref "/docs/mlai/rl-basic-part-1" >}})
> 2. [强化学习入门 - 表格型策略]({{< ref "/docs/mlai/rl-basic-part-2" >}})
> 3. [强化学习入门 - 参数化策略]({{< ref "/docs/mlai/rl-basic-part-3" >}})
> 4. [强化学习入门 - DQN]({{< ref "/docs/mlai/rl-basic-part-4" >}})


# 强化学习入门: 参数化策略

这一部分讨论一下比较复杂的一种策略, 也是最常用的一种策略: 参数化策略.
与表格型策略不同, 参数化策略用函数而不是离散的表格来表示策略.
这使得它能够应用于大规模的连续空间, 也是目前应用中最主流的策略.

> [!IMPORTANT]
> 这一部分我们就只讨论 **免模型+带有终止状态的无限时域 MDP** 的情况.

参数化策略中的策略用 $\pi_\theta$ 来表示. 其中的 $\theta$ 即参数.
在状态 $s$ 下采用动作 $a$ 的概率记为 $\pi_\theta(a|s)$.

随着智能体与环境不断的交互, 会产生一条轨迹. 用 $\tau$ 来表示:
$$
\tau = \\{s_1, a_1, s_2, a_2, ..., s_T\\}
$$
这个轨迹的意思是智能体最开始在 $s_1$ 的状态下, 之后做出了 $a_1$ 的动作.
智能体的状态也随之更新到了 $s_2$, 如此不断重复直到到达终止状态 $s_T$.
用 $r(\tau)$ 代表轨迹 $\tau$ 获得的总共的奖励.

对于策略 $\pi_\theta$, 可以用 $p_\theta(\tau)$ 表示轨迹 $\tau$ 发生的可能性:
$$
\begin{aligned}
p_\theta(\tau) &= p(s_1)\pi_\theta(a_1|s_1)p(s_2|a_1, s_1)\pi_\theta(a_2|s_2)\cdots p(s_T|a_{T-1}, s_{T-1})\\\\
&= p(s_T|a_{T-1}, s_{T-1})\prod_{i=1}^{T-1} p(s_i)\pi_\theta(a_i|s_i)
\end{aligned}
$$

由于环境是随机的, 因此我们优化的目标是尽可能的让策略 $\pi_\theta$ 能够在所有可能性下尽量都能取到比较高的奖励.
因此, 我们优化的是奖励的期望 $R_\theta$:
$$
R_\theta = E_\theta(r(\tau)) = \sum_\tau r(\tau)p_\theta(\tau)
$$

## 策略梯度定理

跟其他机器学习方法一样, 这里同样是求梯度然后优化.

我们先对 $R_\theta$ 求梯度:
$$
\begin{aligned}
\nabla R_\theta &= \sum_\tau r(\tau) \nabla p_\theta(\tau)\\\\
&= \sum_\tau r(\tau) p_\theta(\tau) \nabla \log  p_\theta(\tau)\\\\
&= E_\theta (r(\tau) \nabla \log  p_\theta(\tau))
\end{aligned}
$$

> [!NOTE]
> 这里用了一个公式:
> $$
> \nabla f(x) = f(x) \nabla \log (f(x))
> $$
> 变形一下就是对数函数求导公式+链式法则:
> $$
> \nabla \log(f(x)) = \frac{\nabla f(x)}{f(x)}
> $$


我们没办法直接求期望, 但我们可以故技重施: 用采样求均值的方式估计期望. 假设我们采样 $N$ 次就有:
$$
\begin{aligned}
E_\theta (r(\tau) \nabla \log  p_\theta(\tau)) &= \frac{1}{N} \sum_{k=1}^N r(\tau^k) \nabla \log p_\theta(\tau^k)\\\\
&= \frac{1}{N} \sum_{k=1}^N \sum_{i=1}^{T^k - 1} r(\tau^k) \nabla \log \pi_\theta(a_i^k|s_i^k)
\end{aligned}
$$
也就是:
$$
\nabla R_\theta = \frac{1}{N} \sum_{k=1}^N \sum_{i=1}^{T^k - 1} r(\tau^k) \nabla \log \pi_\theta(a_i^k|s_i^k)
$$


这样只要我们采样足够多, 就可以算出梯度, 然后用梯度上升法更新 $\theta$:
$$
\theta \leftarrow \theta + \alpha \nabla R_\theta
$$

这就是最简单的策略梯度定理.

### 改进: 调整分配

原始的策略梯度定理很好, 但是还是有一个问题.

$$
\nabla R_\theta = \frac{1}{N} \sum_{k=1}^N \sum_{i=1}^{T^k - 1} r(\tau^k) \nabla \log \pi_\theta(a_i^k|s_i^k)
$$
仔细观察可以发现, 我们在同一个路径 $\tau$ 里面, 对于所有的动作都是用的同样的 $r(\tau)$.
这显然是不合理的, 因为一个路径可以有很多个动作.
这些动作里面有好有坏.
不应该用路径总的奖励来作为所有动作的奖励.

这个问题的解决方案就是把 $r(\tau)$ 换掉:
$$
\nabla R_\theta = \frac{1}{N} \sum_{k=1}^N \sum_{i=1}^{T^k - 1} G^k_i  \nabla \log \pi_\theta(a_i^k|s_i^k)
$$
其中
$$
G^k_i = \sum_{j=i}^{T^k - 1} \gamma^{j - i} r^k_j
$$
把原来的 $r(\tau)$ 换成当前状态后的奖励 $G^k_i(s_i)$, 并且带有折扣因子.

这样, 每个动作对应的权值就由一整条路径的奖励变成了这个动作之后的奖励. 而且随着时间的增加, 后面的奖励越来越少, 对应着该动作的影响力越来越小.

### 改进: 添加基线

一般场景下, 我们会设定奖励函数有正有负.
- 如果模型得到了正向奖励, 那么在梯度上升的时候会让这个模型更高概率去执行这个操作.
- 反之, 如果模型得到的奖励是负的, 那么在梯度上升的时候就会让模型尽量不要去执行这个操作.

但有些场景下, 我们会设置奖励函数都是正数. 比如我们在玩乒乓球游戏时, 得到的奖励总是正的.
那这样的话, 相当于模型都要提高这些操作的概率.
但这些操作的概率总和是 1. 因此:
- 模型会提高那些获得奖励更多的操作的概率
- 降低那些获得奖励相对较低的操作的概率

对应到公式上,
如果 $a_i^k$ 对应的 $G^k_i \nabla \log \pi_\theta(a_i^k|s_i^k)$ 比较大, 那么梯度上升的时候就会更多的偏向这个操作的方向.

如果我们的采样数量非常多, 每种情况都能采样到, 那么就不存在什么问题. 但是我们的采样不一定能覆盖到所有情况. 举个具体的例子:

假设在某个状态 $s$ 下有三个操作可以选:
- 操作 $a_1$: 执行后的路径获得的奖励是 +1
- 操作 $a_2$: 执行后的路径获得的奖励是 +3
- 操作 $a_3$: 执行后的路径获得的奖励是 +5

那梯度上升的时候可能就会提高 $a_2$ 与 $a_3$ 的概率.
但有可能我们根本没采样到 $a_3$ 这个操作的路径, 结果导致模型提升了 $a_1$ 与 $a_2$ 的概率.
$a_3$ 的概率反而被降低了.

也就是说, 我们希望模型能够知道: **没有被采样到的操作 ≠ 不好的操作**.
为了实现这一点, 我们就引入了一个技巧: 添加基线.

我们修改公式, 加入一个基线 $b(s_i)$:
$$
\nabla R_\theta = \frac{1}{N} \sum_{k=1}^N \sum_{i=1}^{T^k - 1} (G^k_i - b(s_i)) \nabla \log \pi_\theta(a_i^k|s_i^k)
$$

这个基线 $b(s_i)$ 就相当于一个预期.
比如刚才的场景, $b(s)$ 可能就是 3. 那么:
- 操作 $a_1$: 执行后的路径获得的奖励是 +1, 减掉基线后变成 -2
- 操作 $a_2$: 执行后的路径获得的奖励是 +3, 减掉基线后变成 0

模型就会去提高 $a_2$ 与 $a_3$ 的概率.
这样, 即使我们没有采样到 $a_3$, 模型仍然会提高 $a_3$ 的概率.

> [!TIP]
> 其实就相当于模型对状态有一个 **预期**. 让梯度更新只关注「比平均水平好 / 差多少」，减少了回报数值本身的波动对学习的影响

改进后最终的策略梯度公式如下:
$$
\nabla R_\theta = \frac{1}{N} \sum_{k=1}^N \sum_{i=1}^{T^k - 1} (\sum_{j=i}^{T^k - 1} \gamma^{j - i} r^k_j - b(s_i)) \nabla \log \pi_\theta(a_i^k|s_i^k)
$$
其中的 $(\sum_{j=i}^{T^k - 1} \gamma^{j - i} r^k_j - b(s_i))$ 又被称为**相对优势函数**, 用 $A_\theta(s, a)$ 表示. 从名字可以看出, 它表示一个在状态 $s$ 下, 相较于其他可能的动作, 执行动作 $a$ 的优势有多大.

## REINFORCE

REINFORCE 算法基于没有基线的策略梯度定理, 具体流程如下:

- 假设要跑 $N$ 个回合, 对于其中的回合 $k$:
   - 根据策略 $\pi_\theta$ 生成路径 $\tau^k$
   - 对 $\tau^k$ 中每一步, 计算 $\nabla R_\theta$
   - 根据 $\nabla R_\theta$ 更新 $\theta$

> [!TIP]
> 也有改进版带有基线的 REINFORCE with Baseline 算法.
> 不过由于种种原因, REINFORCE 算法本身用的并不多, 因此就不展开叙述了.

> [!INFO]
> REINFORCE 算法是同策略算法.


## Actor-Critic

REINFORCE 算法更新时依赖于蒙特卡洛回报，方差很大, 采集到的数据可能会背离真实值.
Actor-Critic 算法的出发点就是为了解决 REINFORCE 算法方差过大的问题.
与 REINFORCE 算法不同, Actor-Critic 算法不再基于真实路径回报进行优化,
而是用一个网络去估计每个状态的回报期望.
这样就避免了数据方差大的问题.

具体而言, 整个算法包括三部分: Actor, Critic, 环境.
- Actor 负责根据状态以及策略生成动作, 并根据 Critic 提供的状态价值对策略进行优化
- Critic 负责根据奖励与状态估计状态价值, 并在估计的过程中不断优化自身估计的精确度
- 环境负责接收 Actor 提供的动作, 并输出获得的奖励与下一个状态

<div align="center">
	<img id="ac_auto_svg"  src="/image/mlai/rl-basic-part-3/ac.svg" width="40%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    Actor-Critic 示意图
    </div>
</div>

> [!INFO]
> 这里讨论 V-based Critic.


我们先看 Critic 是怎么更新的.
我们用 $w$ 来表示 Critic 网络的参数.
直接用一阶时序差分的均方差来当作损失函数:
$$
J(w) = \frac{1}{2} (r_{t+1} + \gamma V_w(s_{t+1}) - V_w(s_t))^2
$$
求梯度可得
$$
\nabla_w J(w) = (r_{t+1} + \gamma V_w(s_{t+1}) - V_w(s_t))^2 \nabla_w V_w(s_t)
$$
之后应用梯度下降即可.

> [!IMPORTANT]
> 这里求梯度的时候我们没有对 $V_w(s_{t+1})$ 应用链式法则. 这是因为如果对 $V_w(s_{t+1})$ 求梯度, 那么更新的时候会同时修改 $V_w(s_{t+1})$ 与 $V_w(s_{t})$. 就像是教师批改作业的时候同时修改参考答案跟学生的答案, 会产生震荡. 因此这里只对 $V_w(s_t)$ 应用链式法则.
>
> 这种求梯度的方式也被称作**半梯度方法**.

对应的 Actor 的损失函数梯度就是将权值换成价值函数的一阶时序差分:
$$
\nabla_\theta J(\theta) = (r_{t+1} + \gamma V_w(s_{t+1}) - V_w(s_t)) \nabla \log \pi_\theta(a_t|s_t)
$$

这样每走一步都会进行更新. 具体的算法如下:
- 假设要跑 $N$ 个回合, 对于其中的每个回合:
  - 对于回合中的每一步 $t$:
    - Actor 根据当前的状态 $s_t$ 给出对应的动作 $a_t$
    - Critic 输出当前状态 $s_t$ 的状态价值 $V(s_t)$
    - Actor 根据 $V(s_t)$ 优化自己的参数 $\theta$
    - 将 $a_t$ 输入到环境中获得对应的奖励 $r_{t+1}$ 与下一个状态 $s_{t+1}$
    - Critic 根据 $r_{t+1}$ 与下一个状态 $s_{t+1}$ 更新自己的参数 $w$

> [!INFO]
> Actor-Critic 也是同策略算法.

## 重要性采样

由于同策略的原因, REINFORCE 有以下的缺点:
- **数据利用效率低**: 一旦策略更新，旧数据就不再符合分布，不能再次使用
- **探索受限**: 探索完全由 $\pi_\theta$ 控制. 如果 $\pi_\theta$ 在早期陷入某些“坏动作”的区域, 探索就会非常慢

一个简单的改进思路是换用异策略来解决同策略的缺点.
但直接使用异策略会导致分布不匹配的问题.

举个简单的例子, 假设有一个双臂老虎机, 有以下两个动作:
- 动作 $a_1$: 平均回报为 1.0
- 动作 $a_2$: 平均回报为 2.0

现在我们有两个策略: 行为策略 $\mu$ 与目标策略 $\pi$:
- 行为策略 $\mu$ 为了保证探索性, 有 0.8 的概率选择 $a_1$, 有 0.2 的概率选择 $a_2$
- 目标策略 $\pi$ 则需要保证最优, 有 0.2 的概率选择 $a_1$, 有 0.8 的概率选择 $a_2$

由于我们采样时用的是 $\mu$, 因此我们只能得到 $\mu$ 的奖励期望.
而我们在优化梯度时, 是用 $\pi$ 的奖励期望来优化的:
$$
\nabla R_\pi = E_\pi (r(\tau) \nabla \log  p_\pi(\tau))
$$
如果我们直接把 $\mu$ 的奖励期望拿来来优化 $\pi$, 那么就会导致分布不匹配.

解决分布不匹配的一个思路是进行重要性采样.
重要性采样要解决的问题就是怎么用 $\mu$ 来计算 $E_\pi$.

根据期望计算公式, 有:
$$
E_\pi(r(a)) = \sum_{a\in A} r(a) \pi(a) = \sum_{a\in A} r(a) \mu(a) \frac{\pi(a)}{\mu(a)} = E_\mu(r(a)\frac{\pi(a)}{\mu(a)})
$$

因此, 我们计算 $r(a)\frac{\pi(a)}{\mu(a)}$ 在 $\mu$ 下面的期望就可以得到 $E_\pi$. 这就是重要性采样. 其中的 $\frac{\pi(a)}{\mu(a)}$ 被称为是重要性权重. 它衡量了一个动作在 $\pi$ 中相对于 $\mu$ 的重要性.

重要性采样提供了在行为策略与目标策略之间转换的工具, 
但在应用中要注意一点: 两个策略不能差距过大. 我们比较一下原始的方差 $\text{var}\_\pi(r(a))$ 与估算得到的方差 $\text{var}\_\mu(r(a)\dfrac{\pi(a)}{\mu(a)})$ 就可以理解:

$$
\text{var}\_\mu (r(a)\frac{\pi(a)}{\mu(a)}) - \text{var}\_\pi(r(a))
= E\_\mu(r^2(a)(\frac{\pi(a)}{\mu(a)})^2) - E\_\pi(r^2(a)) \geq 0
$$

其中 $\dfrac{\pi(a)}{\mu(a)}$ 差的越大, 这两个方差之间差的就越大, 我们采样得到的结果的差距就越大.

> [!IMPORTANT]
> 看到这里可能就有点疑惑了, 我们的出发点是换用异策略, 但是这里又要求 $\mu$ 跟 $\pi$ 不能差的过大.
> 那不就又回到同策略了吗?
>
> 确实, 重要性采样只是**在形式上**变得像是异策略, 但它并没有改变同策略的本质. 不过它允许在 $\mu$ 采集到的同一批数据上对新策略做更新 (前提是采样用的策略跟要优化的策略不能差太多). 比起 REINFORCE 一个数据只能用一次还是有所改进的.

> [!IMPORTANT]
> 这里额外说一句, **完全**基于策略梯度的算法都是同策略的.
> 这是因为梯度下降要求优化的函数与采样的函数必须一致.

## 信任区域策略优化 (TRPO)

TRPO 基于重要性采样, 因此是同策略的.

假设行动策略对应的参数为 $\theta^\prime$, 目标策略对应的参数为 $\theta$
$$
R_\theta = E_{\theta^\prime} (A_\theta(s, a) \frac{p_\theta(a|s)}{p_{\theta^\prime}(a|s)})
$$
由于重要性采样的要求, 需要加一个约束项来约束这两个策略之间的差距.
于是用 KL 散度来约束:
$$
D_{\text{KL}}(\theta \parallel \theta^\prime) < \delta 
$$

不过这个约束项不好实现.
因此我们一般会用 TRPO 改进版, 也就是 PPO.

## 近端策略优化 (PPO)

PPO 的改进之处在于把约束项与奖励合并成了一个优化目标:
$$
J_{\text{PPO}}(\theta, \theta^\prime) = R_\theta - \beta D_{\text{KL}}(\theta \parallel \theta^\prime)
$$
这样我们只需要优化 $J_{\text{PPO}}$ 即可.

> [!INFO]
> 值得一提的是, PPO 中的 Proximal (近端) 即来自于这个约束项, 利用与要优化的函数的相近的函数来采样.

PPO 又有两个主要的变种: 近端策略优化惩罚 (PPO-penalty) 与近端策略优化裁切 (PPO-clip).

### 近端策略优化惩罚 (PPO-penalty)

这里的惩罚就是指约束项 $D_{\text{KL}}(\theta \parallel \theta^\prime)$.
原始的 PPO 中, 约束项前面的 $\beta$ 是一个常数.
但实际中我们希望它是一个变量, 能够自动调整.
因此我们引入两个阈值: $D_{\max}$ 与 $D_{\min}$:
- 当 $D_{\text{KL}}(\theta \parallel \theta^\prime) > D_{\max}$ 的时候, 就调大 $\beta$
  - 这说明两个策略已经差很多了, 我们希望先提高两个策略的相似程度
- 当 $D_{\text{KL}}(\theta \parallel \theta^\prime) < D_{\min}$ 的时候, 就调小 $\beta$
  - 这说明两个策略已经很相似了, 我们希望避免只提高两个策略的相似程度

### 近端策略优化裁切 (PPO-clip)

我们定义裁切函数 $\text{clip}(x, a, b)$ 如下, 其中 $a > b$:
$$
\text{clip}(x, a, b) =
\begin{cases}
a, &x \leq a \\\\
x, &a < x \leq b \\\\
b, &b < x
\end{cases}
$$

PPO-clip 用裁切函数代替了约束项.
这也是名字中「裁切」的来源.
其优化目标如下:
$$
J_{\text{PPO-clip}}(\theta, \theta^\prime) = R_\theta = E_{\theta^\prime} (A_\theta(s, a) \text{clip}(\frac{p_\theta(a|s)}{p_{\theta^\prime}(a|s)}, 1-\varepsilon, 1+\varepsilon))
$$

通过约束比值在 $\[1-\varepsilon, 1+\varepsilon\]$, 使得目标函数不再随差距继续增大而增加收益. 从而强制了策略更新不会过度远离旧策略.
