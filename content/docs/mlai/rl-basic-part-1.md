---
title: "强化学习入门 - 基础定义"
weight: 1
date: 2025-09-13
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 强化学习入门: 基础定义

强化学习的理论基础来自于马尔可夫决策过程.
我们先学习一下马尔可夫决策过程是什么.

## 随机过程

随机过程是指随着时间演化的一系列随机变量的集合, 形式上可以写作:
$$\\{X_t\\}_{t\in T}$$
其中 $T$ 代表时间集合, $X_t$ 代表在时间 $t$ 时刻的随机变量.

我们把 $X$ 所有可能的取值范围称为是状态空间, 记作 $S$, 其中的取值 $s$ 称为一个状态.

如果 $T$ 是离散的, 比如是一个集合 $\\{0, 1, 2, ...\\}$, 那么它就是一个离散随机过程.
反之, 如果 $T$ 是连续的, 比如是一个区间 $[0, 1]$, 那么它就是一个连续随机过程.
由于计算机只能处理离散值, 因此我们只讨论离散随机过程.

## 马尔可夫链

在随机过程中, 马尔可夫性质是指一个随机过程在给定现在状态及所有过去状态情况下, 其未来状态的条件概率分布仅依赖于当前状态.

> [!NOTE]
> 其实马尔科夫性质就是在动态规划中常说的 **「无后效性」**, 一句话概括就是「未来的状态只依赖于当前状态，而与过去的状态无关」

对于一个离散随机过程 $\\{X_t\\}_{t \in T}$, 如果对于任意的时刻

$$t_0 < t_1 < ... < t_n < t \in T$$
以及任意的状态
$$x_0, x_1, ..., x_n, x \in S$$
都有
$$P(X_t=x|X_{t_n} = x_n, X_{t_{n-1}}=x_{n-1}, ..., X_{t_0}=x_0)=P(X_t=x|X_{t_n} = x_n)$$
则称其满足马尔可夫性质.

我们把满足马尔可夫性质的离散随机过程称为马尔可夫过程, 也可称作**马尔可夫链**. 可以用有向图来表示，用节点代表状态, 用有向边表示节点之间的状态转移. 比如下图就是一个马尔可夫链, 其状态空间 $S=\\{s_1, s_2, s_3, s_4\\}$.

<div align="center">
	<img id="markov_auto_svg"  src="/image/mlai/rl-basic-part-1/markov.svg" width="35%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    用有向图表示马尔可夫链
    </div>
</div>

我们可以把这个有向图表示为一个矩阵, 矩阵的第 $i$ 行第 $j$ 列代表状态 $s_i$ 转移到状态 $s_j$ 的概率 $p(s_j|s_i)$. 假设有 $N$ 个状态, 那么写成矩阵就是

$$\mathbf{P}=
\begin{bmatrix}
p(s_1|s_1) & p(s_2|s_1) & \cdots & p(s_N|s_1) \\\\
p(s_1|s_2) & p(s_2|s_2) & \cdots & p(s_N|s_2) \\\\
\vdots & \vdots & \ddots & \vdots \\\\
p(s_1|s_N) & p(s_2|s_N) & \cdots & p(s_N|s_N)
\end{bmatrix}
$$

例如上图的马尔可夫链就可以写成

$$\mathbf{P}=
\begin{bmatrix}
0.1 & 0.2 & 0.7 & 0   \\\\
1   & 0   & 0   & 0   \\\\
0   & 0.3 & 0.5 & 0.2 \\\\
0   & 1   & 0   & 0
\end{bmatrix}
$$

## 马尔可夫奖励过程

强化学习的本质是趋利避害, 现在状态有了, 状态之间的转移也有了, 我们还要定义什么是「利」, 什么是「害」, 也就是哪些状态是好的, 哪些状态是不好的. 因此我们引入「奖励」的概念.

我们用 $r$ 表示即时奖励, 它依赖于转移前后的状态. 当我们从状态 $s_t$ 转移到 $s_{t+1}$ 的时候, 用 $r_{t+1}$ 来表示获取的即时奖励.

> [!IMPORTANT]
> $r$ 是一个随机变量而不是一个具体的值.

> [!NOTE]
> 有些教材认为即时奖励只与当前或者下一个状态有关, 这里为了更具有一般性, 我们认为即时奖励与当前状态以及下一个状态都有关, 即依赖于状态转移本身.

除此之外, 我们用 $R(s, s^\prime)$ 来表示状态 $s$ 转移到 $s^\prime$ 时可以获得的即时奖励的期望. 也就是
$$R(s, s^\prime)=E(r_{t+1}|s_t=s, s_{t+1}=s^\prime)$$

同样的, $R$ 也可以写成矩阵形式:
$$
\mathbf R = 
\begin{bmatrix}
R(s_1, s_1) & R(s_1, s_2) & \cdots & R(s_1, s_N) \\\\
R(s_2, s_1) & R(s_2, s_2) & \cdots & R(s_2, s_N) \\\\
\vdots & \vdots & \ddots & \vdots \\\\
R(s_N, s_1) & R(s_N, s_2) & \cdots & R(s_N, s_N)
\end{bmatrix}
$$


除此以外, 我们还想定义一个折扣因子 $\gamma$. 这样就构成了一个**马尔可夫奖励过程 (MRP)**, 可以用四元组表示:
$$\text{MRP} = (S, \mathbf P, \mathbf R, \gamma)$$

> [!NOTE]
> 我们使用折扣因子 $\gamma$ 的原因如下:
> 1. 有些马尔可夫过程是带环的, 它并不会终结, 我们想避免无穷的奖励.
> 2. 我们并不能建立完美的模拟环境的模型, 我们对未来的评估不一定是准确的, 我们不一定完全信任模型, 因为这种不确定性, 所以我们对未来的评估增加一个折扣. 我们想把这个不确定性表示出来, 希望尽可能快地得到奖励, 而不是在未来某一个点得到奖励.
> 3. 如果奖励是有实际价值的, 我们可能更希望立刻就得到奖励, 而不是后面再得到奖励
>     - 有些时候可以把折扣因子设为 0, 代表只关注当前的奖励.
>     - 也可以把折扣因子设为 1, 表示未来获得的奖励与当前获得的奖励是一样的
> 
> 折扣因子可以作为强化学习智能体的一个超参数来进行调整, 通过调整折扣因子, 我们可以得到不同动作的智能体.

在一个马尔可夫奖励过程中, 定义某一个时刻 $t$ 开始的回报 $G_t$ 为:
$$G_t=\sum_{k=0}^\infty \gamma^k r_{t+k+1}$$

> [!IMPORTANT]
> $G_t$ 同样也是一个随机变量而不是具体的值.

假设时刻 $t$ 的时候我们处于状态 $s$, 那么就可以定义状态 $s$ 在时刻 $t$ 的状态价值函数为
$$
V_t(s)=E(G_t|s_t=s)
$$

> [!NOTE]
> 对于无限时域马尔可夫奖励过程 (比如可以无限走动的迷宫), 状态价值函数只与状态 $s$ 有关, 而与 $t$ 无关. 因此可以省略 $t$ 写为 $V(s)$.
> 
> 但对于有限时域的马尔可夫奖励过程 (比如限制步数的游戏), 状态价值函数与 $t$ 有关. 举个简单的例子:
> - 在第 1 步到达某个状态, 未来还能拿 9 步奖励;
> - 在第 9 步到达同样的状态, 未来最多只剩 1 步奖励.
> 
> 因此必须写为 $V_t(s)$

接下来的问题就是如何计算 $V_t(s)$. 这里我们可以用贝尔曼方程来进行迭代计算. 贝尔曼方程的定义如下:
$$V_t(s) =\sum_{s^\prime \in S} p(s^\prime |s) R(s, s^\prime) + \gamma \sum_{s^\prime \in S} p(s^\prime |s)V_{t+1}(s^\prime)$$


可以这样理解贝尔曼方程: 
- 第一项 $\sum_{s^\prime \in S} p(s^\prime |s) R(s, s^\prime)$ 代表下一步的奖励期望
- 第二项 $\gamma \sum_{s^\prime \in S} p(s^\prime |s)V_{t+1}(s^\prime)$ 代表对下一步转移到对所有状态的状态价值函数求期望.

贝尔曼方程的证明如下:
$$
\begin{aligned}
V_t(s) &= E(G_t|s_t=s)\\\\
&= E(\sum_{k=0}^\infty \gamma^k r_{t+k+1} | s_t=s)\\\\
&= E(r_{t+1} + \gamma\sum_{k=0}^\infty \gamma^k r_{(t+1)+k+1} | s_t=s)\\\\
&= E(r_{t+1} + \gamma G_{t+1} | s_t=s)\\\\
&= E(r_{t+1} | s_t=s) + \gamma E(G_{t+1} | s_t=s)\\\\
&= E(r_{t+1} | s_t=s) + \gamma E(E(G_{t+1}|s_{t+1}=s^\prime) | s_t=s)\\\\
&= \sum_{s^\prime \in S} p(s^\prime |s) R(s, s^\prime) + \gamma \sum_{s^\prime \in S} p(s^\prime |s)V_{t+1}(s^\prime)
\end{aligned}
$$

> [!TIP]
> 倒数第 3 行的推导用了叠期望公式. 这是因为 $s_t$ 与 $G_{t+1}$ 并不直接相关.

有了贝尔曼方程之后就可以求解 $V_t(s)$.

对于**有限期的马尔可夫奖励过程**, 
贝尔曼方程可以写成矩阵的形式 (假设 $\mathbf V$ 是列向量):
$$
\mathbf V_t = \text{diag}(\mathbf P \mathbf R^\top) + \gamma \mathbf P \mathbf V_{t+1} 
$$

求解时先写出最后一个时刻的 $\mathbf V$, 之后向前递推就可以.

对于**无限期的马尔可夫奖励过程**, 对应的矩阵形式为
$$
\mathbf V = \text{diag}(\mathbf P \mathbf R^\top) + \gamma \mathbf P \mathbf V
$$
移项后可得
$$
\mathbf V = (\mathbf I - \gamma \mathbf P)^{-1}\text{diag}(\mathbf P \mathbf R^\top) 
$$
直接求解 $\mathbf V$ 即可.

## 马尔可夫决策过程

马尔可夫决策过程在马尔可夫奖励过程的基础上引入了动作与决策.
强化学习的数学基础也来自于此.
如果把马尔可夫决策过程比做是一艘船在根据海面的状况有选择的行驶,
那么马尔可夫奖励过程就是一艘船在海面上随波漂流.

> [!TIP]
> 为方便起见, 接下来我们只讨论无限时域下的场景.

我们先引入动作这一概念, 我们用 $a$ 来表示动作. 所有可以做的动作构成了动作空间 $A$.

与此同时, 状态转移的概率不再仅仅依赖于前后状态, 而是将动作也考虑了进来, 也就是说状态转移概率的表达式变成了 $p(s^\prime |s, a)$, 表示当采用动作 $a$ 的时候从 $s$ 转移到 $s^\prime$ 的概率.

同时, 奖励函数也将动作考虑了进来, 变成了 $R(s, s^\prime , a)$, 表示当采用动作 $a$ 的时候, 从 $s$ 转移到 $s^\prime$ 所获得的奖励期望.

这样就构成了一个**马尔可夫决策过程 (MRP)**, 可以用五元组表示:
$$\text{MDP} = (S, A, \mathbf P, \mathbf R, \gamma)$$

下一步就是引入策略这一概念.
策略定义了在某一个状态应该采取什么动作, 得知当前状态后, 可以用策略函数来计算采取不同行动的概率. 我们用 $\pi(a|s)$ 来表示当处于状态 $s$ 时, 采取动作 $a$ 的概率.

假设我们严格遵守策略 $\pi$, 那么马尔可夫决策过程就退化成了马尔可夫奖励过程.

在这种情况下从 $s$ 转移到 $s^\prime$ 的概率可以写作:
$$
p^\pi(s^\prime | s) = \sum_{a\in A}\pi(a | s) p(s^\prime | s, a) 
$$

同样的, 在这种情况下从 $s$ 转移到 $s^\prime$ 的奖励函数可以写作:
$$
R^\pi (s, s^\prime) = \sum_{a\in A} \pi(a|s) R(s, s^\prime, a)
$$

同样的, 在这种情况下 $t$ 时刻状态 $s$ 对应的状态价值函数写作:
$$
V^\pi(s) = E_\pi(G_t|s_t=s)
$$

接下来, 我们引入一个函数 $Q$, 称为动作价值函数, 定义如下:
$$
Q^\pi(s, a) = E_\pi(G_t|s_t=s, a_t=a)
$$
它描述的是：在状态 $s$ 下选择某个动作 $a$, 并随后按照策略 $\pi$ 来行动时, 获得的累积奖励的期望.

> [!TIP]
> $Q^\pi(s, a)$ 提供了一个比较同一状态下不同动作的价值的方法. 有了 $Q^\pi(s, a)$ 之后, 我们就可以比较状态 $s$ 下哪个动作更好, 从而更新策略.

接下来就是怎么算 $Q^\pi(s, a)$ 与 $V^\pi(s)$. 类似于前面马尔可夫奖励过程的递推公式, 这里的 $Q^\pi(s, a)$ 与 $V^\pi(s)$ 也可以用递推的方式算出来. 证明如下:

仔细观察 $Q^\pi(s, a)$ 与 $V^\pi(s)$ 的定义, 不难发现:
$$
V^\pi(s) = \sum_{a\in A} \pi(a|s) Q^\pi(s, a)
\tag{1}
$$

将 $Q$ 函数用前面马尔可夫奖励过程推导 $V(s)$ 的贝尔曼方程的方式展开:

$$
Q^\pi(s,a) = \sum_{s^\prime \in S} p(s^\prime |s, a) R(s, s^\prime, a) + \gamma \sum_{s^\prime \in S} p(s^\prime |s, a)V^\pi(s^\prime)
\tag{2}
$$

将 $(1)$ 式代入 $(2)$ 式可得 $Q^\pi(s, a)$ 的递推公式:
$$
Q^\pi(s,a) = \sum_{s^\prime \in S} p(s^\prime |s, a) R(s, s^\prime, a) + \gamma \sum_{s^\prime \in S} p(s^\prime |s, a)\sum_{a\in A} \pi(a|s) Q^\pi(s^\prime, a)
$$

将 $(2)$ 式代入 $(1)$ 式可得 $V^\pi(s)$ 的递推公式:
$$
V^\pi(s) = \sum_{a\in A} \pi(a|s) \sum_{s^\prime \in S} p(s^\prime |s, a) (R(s, s^\prime, a) + \gamma V^\pi(s^\prime))
$$

直接推式子可能不太好理解, 我们用回溯图 (backup diagram) 来形象的解释一下 $(1)$ 式与 $(2)$ 式.

<div align="center">
	<img id="backup_diagram_auto_svg"  src="/image/mlai/rl-basic-part-1/backup-diagram.svg" width="100%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    回溯图
    </div>
</div>

上面的回溯图中:
- 空心圆圈代表 $V^\pi$
- 实心圆圈代表 $Q^\pi$.
- 从空心圆圈指向实心圆圈的有向边代表 $\pi(a|s)$
- 从实心圆圈指向空心圆圈的有向边代表 $p(s^\prime|s, a)$.

> [!IMPORTANT]
> $\gamma$ 与 $R(s, s^\prime, a)$ 在图中没有画出.

对于每个节点, 其值由该节点出发的有向边与其指向的节点的值决定. 红框代表了 $(1)$ 式的计算过程. 蓝框代表了 $(2)$ 式的计算过程.

通过迭代的方式, 我们就可以计算出所有的 $Q^\pi(s, a)$ 与 $V^\pi(s)$.
