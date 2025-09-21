---
title: "强化学习入门 - DQN"
weight: 1
date: 2025-09-20
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 强化学习入门: DQN

这一部分讲一下 DQN 算法. DQN 算法是一种值函数方法.
与参数化策略不同, 值函数方法不需要策略梯度. 做个简单的比喻就是:
- **参数化策略**: 直接教 Actor 怎么演
  - 必须用策略梯度来调整动作.
- **值函数方法**: 只管训练一个 Critic 去预测动作好不好
  - 策略从评论家口中间接产生, 不需要策略梯度.


## 基本概念

DQN 的出发点来自于 Q-learning.
在表格型策略中的 Q-learning 中,
我们存储了一个 Q 表格用来更新策略.
当状态跟动作空间都有限的时候, 这样做是可行的.
但当状态与动作空间都非常大的时候就没办法处理了.
比如在玩 ATARI 游戏的时候, 屏幕的每帧画面都是一个状态.
显然这是不可能用表格存下的. 因此需要找个方法.


DQN 的思路就是把表格替换成一个函数.
也就是说, 我们不再用表格来存储 $Q(s, a)$,
而是训练一个参数为 $\theta$ 的函数 $Q_\theta(s, a)$.

我们先看一下最优的 $Q^\ast(s, a)$ 满足什么性质.
根据 $Q(s, a)$ 定义, 假设最优的策略是 $\pi^\ast$, 那么有:
$$
Q^\ast(s_t, a_t) = r_{t+1} + \gamma Q^\ast(s_{t+1}, \pi^\ast(s_{t+1}))
$$
根据贪心策略, 又有
$$
\pi^\ast(s) = \argmax_{a\in A} Q^\ast(s, a)
$$
因此有
$$
Q^\ast(s_t, a_t) = r_{t+1} + \gamma \max_{a\in A} Q^\ast(s_{t+1}, a)
$$
这就是一阶差分. 我们可以根据这个进行拟合. 不断的拉近等式左边与等式右边之间的距离就可以.

具体到每一步来说,
假设我们采样了一个状态转移 $(s_t, a_t, r_{t+1}, s_{t+1}, a_{t+1})$,
那么我们想尽可能的拉近这两个的距离:
$$
Q_\theta(s_t, a_t) \longleftrightarrow r_{t+1} + \gamma \max_{a\in A} Q_\theta(s_{t+1}, a)
$$
可以构建均方差作为损失函数进行优化:
$$
J(\theta) = \frac{1}{2} (r_{t+1} + \gamma \max_{a\in A} Q_\theta(s_{t+1}, a) - Q_\theta(s_t, a_t))^2
$$

求梯度可得
$$
\nabla_\theta J(\theta) = (r_{t+1} + \gamma \max_{a\in A} Q_\theta(s_{t+1}, a) - Q_\theta(s_t, a_t)) \nabla_\theta Q_\theta(s_t, a_t)
$$

> [!IMPORTANT]
> 这里也用了半梯度方法, 我们没有对 $y$ 中的 $\max_{a\in A} Q_\theta(s_{t+1}, a)$ 求梯度.

之后利用梯度下降公式对 $\theta$ 进行更新:
$$
\theta \leftarrow \theta - \alpha \nabla_\theta J(\theta)
$$

> [!IMPORTANT]
> **这里的梯度不是策略梯度**, 因为它不是用来更新策略 $\pi$ 的, 而是用来更新 $Q$ 的.

最后, 当我们训练完 $Q_\theta$ 之后, 推理时利用下面的公式选取策略即可:
$$
a = \argmax_{a\in A} Q_\theta(s, a)
$$

## 目标网络与在线网络

注意到刚才我们求梯度的时候用了半梯度方法.
也就是说, 其实我们在训练过程中分成了两个网络: 在线网络 (参数用 $\theta$ 表示) 与离线网络 (用 $\theta^-$ 表示)

严谨的梯度公式如下:
$$
\nabla_\theta J(\theta) = (r_{t+1} + \gamma \max_{a\in A} Q_{\theta^-}(s_{t+1}, a) - Q_\theta(s_t, a_t)) \nabla_\theta Q_\theta(s_t, a_t)
$$

具体到一步上的流程如下, 假设当前的状态是 $s_t$:
1. 通过在线网络计算当前采取的动作 $a_t$
2. 将 $a_t$ 输入到环境获取到对应的奖励 $r_{t+1}$ 与下一个状态 $s_{t+1}$
3. 用离线网络计算 $\max_{a\in A} Q_{\theta^-}(s_{t+1}, a)$
4. 根据梯度计算公式更新在线网络
5. 在一定步数之后用当前的在线网络替换掉离线网络

> [!IMPORTANT]
> 要注意一点的是生成 $a_t$ 的时候应该用 $1-\varepsilon$ 贪心策略来保证探索性

用一张图来表示就是:
<div align="center">
	<img id="dqn_auto_svg"  src="/image/mlai/rl-basic-part-4/dqn.svg" width="70%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    用一张图来表示 DQN 的训练过程
    </div>
</div>

## 经验回放 (Experience Replay)

由于 DQN 是值函数方法而非基于策略梯度的方法, 因此可以是异策略的.
因此我们可以采样大量数据并重复利用. 这被称为是 **经验回放**.

> [!INFO]
> 上一部分中我们提到, **完全**基于策略梯度的算法都是同策略的.
> 这是因为梯度下降要求优化的函数与采样的函数必须一致.
> 但 DQN 是值函数方法, 其优化的对象是动作价值函数 $Q$ 而非策略本身.
> 因此可以是异策略的.
>
> 举个比分来说:
> 假设我们在开餐厅, 优化的目标是改进「菜单」
>
> 基于策略梯度的做法类似如下:
> - 每天你只能根据顾客点的菜来收集反馈
> - 如果今天的菜单换了, 顾客点的东西也随之改变
> - 所以你只能根据当下这份菜单收集到的评价来更新它, 因而必须是**同策略**的
>
> 而 DQN 的做法如下:
> - 不直接优化菜单本身, 而是写美食测评指南
> - 无论顾客随便点什么，你都能记录并更新每道菜的评分
> - 最终, 你直接按照评分表挑选最高分的菜, 生成最好的菜单

经验回放会构建一个回放缓冲区, 回放缓冲区又被称为是回放内存.
当我们去交互的时候, 它会把每个 $(s_t, a_t, s_{t+1}, r_{t+1})$ 放进缓冲区.
训练的时候会从中抽数据进行训练.

举个例子, Google 在 DQN 的原始论文中是这样设计的:
- 回放缓冲区大小: 1e6 条记录
- 每次从回放缓冲区抽 32 条记录计算梯度并更新
- 每采集 4 次数据更新一次梯度

带有经验回放的 DQN 算法如下:
- 初始化 $Q_\theta = Q_{\theta^-} = 0$
- 不断进行如下的操作:
  - 采样 N 次数据:
    - 假设当前的状态是 $s_t$:
    - 通过在线网络 $Q_{\theta}$ 计算当前采取的动作 $a_t$
    - 将 $a_t$ 输入到环境获取到对应的奖励 $r_{t+1}$ 与下一个状态 $s_{t+1}$
    - 将 $(s_t, a_t, s_{t+1}, r_{t+1})$ 存入回放缓冲区
  - 更新梯度:
    - 从回放缓冲区中抽一个 batch 的数据基于 $Q_{\theta^-}$ 对 $Q_{\theta}$ 进行优化
  - 在更新 C 次梯度后更新 $Q_{\theta^-}$ 为 $Q_{\theta}$

## 优先经验回放 (Prioritized ER)

在原始的 DQN 中, 我们会从回放缓冲区中均匀的抽取更新用的经验样本.
但实际上, 有些经验样本对于模型训练的更新贡献较小, 特别是当 Q 值已经很接近真实值时, 这些经验样本对网络更新的帮助有限.
而一些重要的经验样本（例如, Q 值估计误差较大的样本）则可能被随机采样的机会较低, 从而导致学习进展缓慢. 
因此, 我们需要给经验样本加权值.

我们用 TD 误差来当作权值, 用 $\delta$ 来表示, TD 误差越大的样本权值越高:
$$
\delta = |r^\prime + \gamma \max_{a\in A} Q_\theta(s^\prime, a) - Q_\theta(s, a)|
$$
这样, 每次抽取的时候就考虑进去权值的因素.
每次更新完参数 $\theta$ 后, 都重新计算权值.

## 噪声网络 (Noisy Nets)

我们还可以改进探索. 目前的探索方式是 $\varepsilon-$贪心.
这可以保证我们能收敛到最优策略, 但还有个更好的办法是噪声网络.
通过给 $Q$ 加一个噪声再取贪心策略来代替原来的 $\varepsilon-$贪心.

> [!IMPORTANT]
> 需要注意的是, 参数虽然会被加上噪声, 但在同一个回合里面参数是固定的. 我们在换回合、玩另一场新的游戏的时候, 才会重新采样噪声.
>
> **这一点非常重要.** 因为这导致了噪声网络与原来的 $\varepsilon-$贪心相比有本质上的差异.
> 
> 原始的 $\varepsilon-$贪心 对于同一个输入, 输出可能是不同的.
> 这其实违背常理, 因为对于一个确定的策略与一个确定的输入, 它的输出就应该是确定的.
> 而噪声网络就可以确保这一点.
> 也就是说, 原来的 $\varepsilon-$贪心是随机乱尝试, 而噪声网络则是系统性的尝试.


## 双深度 Q 网络 (Double DQN)

考虑一个场景:
假设我们有四个动作 $A = \\{a_1, a_2, a_3, a_4\\}$.
当前的状态是 $s_t$, 用在线网络选出的 $a_t$ 是 $a_1$.
之后的下一个状态是 $s_{t+1}$, 用离线网络选出的 $a_{t+1}$ 是 $a_2$.
那我们想要减小这两个东西的差距:
$$
Q_\theta(s_t, a_1) \longleftrightarrow r_{t+1} + \gamma \max_{a\in A} Q_{\theta^-}(s_{t+1}, a_2)
$$

由于网络估计肯定是不准的, 我们不如假设它高估了 $Q(s_{t+1}|a_2)$ (高估的部分用绿色标出).
那我们就会发现, 我们在优化 $Q_\theta(s_t, a_1)$ 的时候会继续高估. 
<div align="center">
	<img id="ddqn_auto_svg"  src="/image/mlai/rl-basic-part-4/ddqn.svg" width="80%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    高估的传递
    </div>
</div>

> [!TIP]
> 由于选择时用的是贪心策略, 因此低估不会被传递.

因此原始的 DQN 在训练的过程中经常会出现高估 $Q$ 的问题.

解决方法也很简单, 既然用一个网络同时做选择+生成预测值会高估, 那么把做选择与生成预测值这两个工作分给不同的网络就可以.
这样, 即使原始的估计有偏差, 由于我们做选择的网络与生成预测值的网络不一致, 因此被传递的那个选项不一定是被高估的.

我们把做选择的工作交给在线网络, 生成预测值的工作仍然是离线网络. 也就是改成拉近这两个的距离:
$$
Q_\theta(s_t, a_t) \longleftrightarrow r_{t+1} + \gamma Q_{\theta^-}(s_{t+1}, \argmax_{a\in A} Q_\theta(s_{t+1}, a))
$$
这就是双深度 Q 网络.

## 对偶深度 Q 网络 (Dueling DQN)

对偶深度 Q 网络的思路是将 $Q(s, a)$ 分成两部分: 状态价值 $V(s)$ 与相对优势 $A(s, a)$:
$$
Q(s, a) = V(s) + A(s, a)
$$
把原先网络的单个输出层换成两个输出层, 分别输出 $V(s)$ 与 $A(s, a)$.

这样做有一个好处就是只需要修改 $V(s)$ 就可以更新这个 $s$ 下所有的 $Q(s, a)$.
这样即使在某一个状态下没采样到某个动作, 也可以更改这个动作的 $Q(s, a)$.

<div align="center">
	<img id="dueling-dqn_auto_svg"  src="/image/mlai/rl-basic-part-4/dueling-dqn.svg" width="80%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    DQN 对比 Dueling DQN
    </div>
</div>

但直接这样做可能会有一个问题: 最后学习到的 $V(s)$ 全都是 0, 而 $A(s, a)$ 是原来的 $Q(s, a)$.
为了避免这个问题, 我们需要给 $A(s, a)$ 加个约束, 使得一般情况下模型更倾向于更新 $V(s)$.

一个直觉上的做法是约束 $\sum_{s} A(s, a) = 0$.
这样做的话, 最后输出 $Q$ 的时候就不是直接 $Q=V+A$ 了,
而是 $Q=V+(A - \bar A)$.
这样网络在更新的时候, 就会更倾向于更新 $V$ 而不是 $A$.

## 分布式 Q 函数 (Distributional Q-func)

这个方法是用一个分布来代替原来的单一输出.
原来的网络是对于每个 $(s, a)$ 输出对应的 $Q(s, a)$.
而分布式 Q 函数不再输出一个值, 而是输出一个分布 $r\sim Q_{(s, a)}$.
通过计算每个 $(s, a)$ 的期望来计算对应的 $Q(s, a)$.

> [!TIP]
> 这样做的好处是我们能拟合一些额外的信息.
> 比如有的动作对应的奖励期望很高, 但是方差也很高 (高风险高收益).
> 如果我们想利用方差信息尽量避开这种动作, 就可以用分布式 Q 函数来建模.
>
> 例如在下图中, 分布式 Q 函数输出的三个分布可以看出:
> - $a_1$ 具有高风险高收益
> - $a_2$ 具有低风险低收益
> - $a_3$ 具有随机风险随机收益

<div align="center">
	<img id="distri-q-func_auto_svg"  src="/image/mlai/rl-basic-part-4/distri-q-func.svg" width="100%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    普通的 Q 函数对比分布式 Q 函数.
    </div>
</div>

## 彩虹 (Rainbow)

彩虹方法就是把上面所有的优化方法都缝合起来, 组成一个究极 DQN.
可以达到 1+1>2 的效果.

<div align="center">
	<img id="rainbow_svg"  src="/image/mlai/rl-basic-part-4/rainbow.svg" width="50%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    实验数据对比. 可以说做到了 1+1>2 的效果.
    </div>
</div>