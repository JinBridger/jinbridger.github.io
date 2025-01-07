---
title: "llm.c 源码解析 - 反向传播"
weight: 1
date: 2025-01-07
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# llm.c 源码解析 - 反向传播

llm.c 是一个开源的用 Cuda 与 C 写成的简易 LLM 推理框架，用大约 1700 行代码完成了对于 GPT-2 模型的训练以及推理。

## Softmax 层反向传播

下面单看一个 token 是怎么计算 Loss的，假设数据集中这个位置的 token 在词表中的下标为 $t$, 设以下符号对应:

- $L$：损失函数
- $\mathbf P$：长度为词表大小的向量, 其中 $\mathbf P_i$ 代表这个 token 为词表里面编号为 $i$ 的 token 的概率 (Prob), 通过 Softmax 由 $\mathbf S$ 计算而来
- $\mathbf S$：长度为词表大小的向量, 其中 $\mathbf S_i$ 代表这个 token 为词表里面编号为 $i$ 的 token 的分数 (Score)

GPT-2 采用的 Loss 计算方式是交叉熵损失函数 (Cross Entropy) 即

$$
L=-\log{\mathbf P_t}
$$

在计算完 Loss 之后就是反向传播。

这一层我们需要计算的是 Score 相对于 Loss 的梯度, 也就是 $\dfrac{\partial L}{\partial \mathbf S}.$ 根据链式求导法则 $$\dfrac{\partial L}{\partial \mathbf S}=\dfrac{\partial L}{\partial \mathbf P}\dfrac{\partial \mathbf P}{\partial \mathbf S}.$$ 因此只需要计算出 $\dfrac{\partial L}{\partial \mathbf P}$ 与 $\dfrac{\partial \mathbf P}{\partial \mathbf S}$ 即可。

对于 $\dfrac{\partial L}{\partial \mathbf P}$:

$$
\dfrac{\partial L}{\partial \mathbf P}=
\begin{bmatrix}
\dfrac{\partial L}{\partial \mathbf P_1} & \dfrac{\partial L}{\partial \mathbf P_2} & \cdots & \dfrac{\partial L}{\partial \mathbf P_{V}} 
\end{bmatrix}
$$

其中

$$
\dfrac{\partial L}{\partial \mathbf P_i} =
\begin{cases}
-\dfrac{1}{\mathbf P_i}, &i=t \\\\
0, &i\neq t
\end{cases}
$$

对于 $\dfrac{\partial \mathbf P}{\partial \mathbf S}$:

$$
\dfrac{\partial \mathbf P}{\partial \mathbf S}=
\begin{bmatrix}
\dfrac{\partial \mathbf P_1}{\partial \mathbf S_1} & \dfrac{\partial \mathbf P_2}{\partial \mathbf S_1} & \cdots & \dfrac{\partial \mathbf P_{V}}{\partial \mathbf S_1} \\\\
\dfrac{\partial \mathbf P_1}{\partial \mathbf S_2} & \dfrac{\partial \mathbf P_2}{\partial \mathbf S_2} & \cdots & \dfrac{\partial \mathbf P_{V}}{\partial \mathbf S_2} \\\\
\vdots & \vdots & \ddots & \vdots \\\\
\dfrac{\partial \mathbf P_1}{\partial \mathbf S_{V}} & \dfrac{\partial \mathbf P_2}{\partial \mathbf S_{V}} & \cdots & \dfrac{\partial \mathbf P_{V}}{\partial \mathbf S_{V}} \\\\
\end{bmatrix}
$$

其中

$$
\dfrac{\partial \mathbf P_i}{\partial \mathbf S_j} =
\begin{cases}
\mathbf P_i(1-\mathbf P_i), &i=j \\\\
-\mathbf P_i\mathbf P_j, &i\neq j
\end{cases}
$$

因此

$$
\begin{aligned}
\dfrac{\partial L}{\partial \mathbf S}&=\dfrac{\partial L}{\partial \mathbf P}\dfrac{\partial \mathbf P}{\partial \mathbf S} \\\\
&=
\begin{bmatrix}
\dfrac{\partial L}{\partial \mathbf P_1} & \dfrac{\partial L}{\partial \mathbf P_2} & \cdots & \dfrac{\partial L}{\partial \mathbf P_{V}} 
\end{bmatrix}
\begin{bmatrix}
\dfrac{\partial \mathbf P_1}{\partial \mathbf S_1} & \dfrac{\partial \mathbf P_2}{\partial \mathbf S_1} & \cdots & \dfrac{\partial \mathbf P_{V}}{\partial \mathbf S_1} \\\\
\dfrac{\partial \mathbf P_1}{\partial \mathbf S_2} & \dfrac{\partial \mathbf P_2}{\partial \mathbf S_2} & \cdots & \dfrac{\partial \mathbf P_{V}}{\partial \mathbf S_2} \\\\
\vdots & \vdots & \ddots & \vdots \\\\
\dfrac{\partial \mathbf P_1}{\partial \mathbf S_{V}} & \dfrac{\partial \mathbf P_2}{\partial \mathbf S_{V}} & \cdots & \dfrac{\partial \mathbf P_{V}}{\partial \mathbf S_{V}} \\\\
\end{bmatrix}\\\\
&=
\begin{bmatrix}
\mathbf P_1 & \mathbf P_2 & \cdots & \mathbf P_t - 1 & \cdots & \mathbf P_{V}
\end{bmatrix}
\end{aligned}
$$


## 线性层反向传播

线性层可以表示为 $\mathbf Y=\mathbf X \mathbf W+\mathbf B$, 其中：
- $\mathbf Y$ 为输出, $\mathbf X$ 为输入，均为 $T \times C$ 的向量
- $\mathbf W$ 为权重，为 $C\times C$ 的矩阵
- $\mathbf B$ 为偏置，为 $1\times C$ 的向量

要进行反向传播，还需要有损失值 $L$

需要求取的梯度包括三个:

- $L$ 对 $\mathbf X$ 的梯度 $\dfrac{\partial L}{\partial \mathbf X}$, 用于传入下一层
- $L$ 对 $\mathbf W$ 的梯度 $\dfrac{\partial L}{\partial \mathbf W}$, 用于更新这一层的 $\mathbf W$
- $L$ 对 $\mathbf B$ 的梯度 $\dfrac{\partial L}{\partial \mathbf B}$, 用于更新这一层的 $\mathbf B$

### L 对于 X 的梯度

对于第一个梯度 $\dfrac{\partial L}{\partial \mathbf X}$, 求取的方法如下：

$$
\begin{aligned}
\dfrac{\partial L}{\partial \mathbf X}
&=
\begin{bmatrix}
\dfrac{\partial L}{\partial \mathbf X_{1,1}} & \dfrac{\partial L}{\partial \mathbf X_{1,2}} & \cdots & \dfrac{\partial L}{\partial \mathbf X_{1,C}} \\\\
\dfrac{\partial L}{\partial \mathbf X_{2,1}} & \dfrac{\partial L}{\partial \mathbf X_{2,2}} & \cdots & \dfrac{\partial L}{\partial \mathbf X_{2,C}} \\\\
\vdots & \vdots & \ddots & \vdots \\\\
\dfrac{\partial L}{\partial \mathbf X_{T,1}} & \dfrac{\partial L}{\partial \mathbf X_{T,2}} & \cdots & \dfrac{\partial L}{\partial \mathbf X_{T,C}} \\\\
\end{bmatrix} \\\\
&=
\begin{bmatrix}
\dfrac{\partial L}{\partial \mathbf Y}\dfrac{\partial \mathbf Y}{\partial \mathbf X_{1,1}} & \dfrac{\partial L}{\partial \mathbf Y}\dfrac{\partial \mathbf Y}{\partial \mathbf X_{1,2}} & \cdots & \dfrac{\partial L}{\partial \mathbf Y}\dfrac{\partial \mathbf Y}{\partial \mathbf X_{1,C}} \\\\
\dfrac{\partial L}{\partial \mathbf Y}\dfrac{\partial \mathbf Y}{\partial \mathbf X_{2,1}} & \dfrac{\partial L}{\partial \mathbf Y}\dfrac{\partial \mathbf Y}{\partial \mathbf X_{2,2}} & \cdots & \dfrac{\partial L}{\partial \mathbf Y}\dfrac{\partial \mathbf Y}{\partial \mathbf X_{2,C}} \\\\
\vdots & \vdots & \ddots & \vdots \\\\
\dfrac{\partial L}{\partial \mathbf Y}\dfrac{\partial \mathbf Y}{\partial \mathbf X_{T,1}} & \dfrac{\partial L}{\partial \mathbf Y}\dfrac{\partial \mathbf Y}{\partial \mathbf X_{T,2}} & \cdots & \dfrac{\partial L}{\partial \mathbf Y}\dfrac{\partial \mathbf Y}{\partial \mathbf X_{T,C}} \\\\
\end{bmatrix}
\end{aligned}
$$

其中 
$$
\begin{aligned}
\dfrac{\partial L}{\partial \mathbf Y}
&=
\begin{bmatrix}
\dfrac{\partial L}{\partial \mathbf Y_{1,1}} & \dfrac{\partial L}{\partial \mathbf Y_{1,2}} & \cdots & \dfrac{\partial L}{\partial \mathbf Y_{1,C}} \\\\
\dfrac{\partial L}{\partial \mathbf Y_{2,1}} & \dfrac{\partial L}{\partial \mathbf Y_{2,2}} & \cdots & \dfrac{\partial L}{\partial \mathbf Y_{2,C}} \\\\
\vdots & \vdots & \ddots & \vdots \\\\
\dfrac{\partial L}{\partial \mathbf Y_{T,1}} & \dfrac{\partial L}{\partial \mathbf Y_{T,2}} & \cdots & \dfrac{\partial L}{\partial \mathbf Y_{T,C}} \\\\
\end{bmatrix}
\end{aligned}
$$
$$
\begin{aligned}
\dfrac{\partial \mathbf Y}{\partial \mathbf X_{i,j}}
&=
\begin{bmatrix}
\dfrac{\partial \mathbf Y_{1,1}}{\partial \mathbf X_{i,j}} & \dfrac{\partial \mathbf Y_{1,2}}{\partial \mathbf X_{i,j}} & \cdots & \dfrac{\partial \mathbf Y_{1,C}}{\partial \mathbf X_{i,j}} \\\\
\dfrac{\partial \mathbf Y_{2,1}}{\partial \mathbf X_{i,j}} & \dfrac{\partial \mathbf Y_{2,2}}{\partial \mathbf X_{i,j}} & \cdots & \dfrac{\partial \mathbf Y_{2,C}}{\partial \mathbf X_{i,j}} \\\\
\vdots & \vdots & \ddots & \vdots \\\\
\dfrac{\partial \mathbf Y_{T,1}}{\partial \mathbf X_{i,j}} & \dfrac{\partial \mathbf Y_{T,2}}{\partial \mathbf X_{i,j}} & \cdots & \dfrac{\partial \mathbf Y_{T,C}}{\partial \mathbf X_{i,j}} \\\\
\end{bmatrix}
\end{aligned}
$$

又有
$$\dfrac{\partial \mathbf Y_{m, n}}{\partial \mathbf X_{i,j}}=\dfrac{\partial (\sum \mathbf X_{m,k} \mathbf W_{k,n} + \mathbf B_n)}{\partial \mathbf X_{i,j}} = 
\begin{cases}
0 &i\neq m\\\\
\mathbf W_{j, n} &i=m
\end{cases}
$$

根据链式法则有

$$
\begin{aligned}
\dfrac{\partial L}{\partial \mathbf X_{i,j}}
&=
\sum\limits_{m=1}^T\sum\limits_{n=1}^C\dfrac{\partial L}{\partial \mathbf Y_{m,n}}\dfrac{\partial \mathbf Y_{m,n}}{\partial \mathbf X_{i,j}} \\\\
&=
\sum\limits_{n=1}^C\sum\limits_{m=1}^T\dfrac{\partial L}{\partial \mathbf Y_{m,n}}\dfrac{\partial \mathbf Y_{m,n}}{\partial \mathbf X_{i,j}} \\\\
&=
\sum\limits_{k=1}^C\sum\limits_{m=1}^T\dfrac{\partial L}{\partial \mathbf Y_{m,k}}\dfrac{\partial \mathbf Y_{m,k}}{\partial \mathbf X_{i,j}} \\\\
&= \sum_{k=1}^{C} \dfrac{\partial L}{\partial \mathbf Y_{i,k}}\mathbf W_{j, k}
\end{aligned}
$$

因此

$$
\begin{aligned}
\dfrac{\partial L}{\partial \mathbf X}
&=
\begin{bmatrix}
\sum\limits_{k=1}^{C} \dfrac{\partial L}{\partial \mathbf Y_{1,k}}\mathbf W_{1, k} & \sum\limits_{k=1}^{C} \dfrac{\partial L}{\partial \mathbf Y_{1,k}}\mathbf W_{2, k} & \cdots & \sum\limits_{k=1}^{C} \dfrac{\partial L}{\partial \mathbf Y_{1,k}}\mathbf W_{C, k} \\\\
\sum\limits_{k=1}^{C} \dfrac{\partial L}{\partial \mathbf Y_{2,k}}\mathbf W_{1, k} & \sum\limits_{k=1}^{C} \dfrac{\partial L}{\partial \mathbf Y_{2,k}}\mathbf W_{2, k} & \cdots & \sum\limits_{k=1}^{C} \dfrac{\partial L}{\partial \mathbf Y_{2,k}}\mathbf W_{C, k} \\\\
\vdots & \vdots & \ddots & \vdots \\\\
\sum\limits_{k=1}^{C} \dfrac{\partial L}{\partial \mathbf Y_{T,k}}\mathbf W_{1, k} & \sum\limits_{k=1}^{C} \dfrac{\partial L}{\partial \mathbf Y_{T,k}}\mathbf W_{2, k} & \cdots & \sum\limits_{k=1}^{C} \dfrac{\partial L}{\partial \mathbf Y_{T,k}}\mathbf W_{C, k} \\\\
\end{bmatrix} \\\\
&= \dfrac{\partial L}{\partial \mathbf Y} \mathbf W^\top
\end{aligned}
$$

### L 对于 W 的梯度

同样的方法，可以求出 
$$
\begin{aligned}
\dfrac{\partial L}{\partial \mathbf W}
&=
\begin{bmatrix}
\sum\limits_{k=1}^{T} \dfrac{\partial L}{\partial \mathbf Y_{k, 1}}\mathbf X_{k, 1} & \sum\limits_{k=1}^{T} \dfrac{\partial L}{\partial \mathbf Y_{k, 2}}\mathbf X_{k, 1} & \cdots & \sum\limits_{k=1}^{T} \dfrac{\partial L}{\partial \mathbf Y_{k, C}}\mathbf X_{k, 1} \\\\
\sum\limits_{k=1}^{T} \dfrac{\partial L}{\partial \mathbf Y_{k, 1}}\mathbf X_{k, 2} & \sum\limits_{k=1}^{T} \dfrac{\partial L}{\partial \mathbf Y_{k, 2}}\mathbf X_{k, 2} & \cdots & \sum\limits_{k=1}^{T} \dfrac{\partial L}{\partial \mathbf Y_{k, C}}\mathbf X_{k, 2} \\\\
\vdots & \vdots & \ddots & \vdots \\\\
\sum\limits_{k=1}^{T} \dfrac{\partial L}{\partial \mathbf Y_{k, 1}}\mathbf X_{k, C} & \sum\limits_{k=1}^{T} \dfrac{\partial L}{\partial \mathbf Y_{k, 2}}\mathbf X_{k, C} & \cdots & \sum\limits_{k=1}^{T} \dfrac{\partial L}{\partial \mathbf Y_{k, C}}\mathbf X_{k, C} \\\\
\end{bmatrix} \\\\
&= \mathbf X^\top \dfrac{\partial L}{\partial \mathbf Y}
\end{aligned}
$$

### L 对于 B 的梯度

同样的方法，可以求出 

$$
\begin{aligned}
\dfrac{\partial L}{\partial \mathbf B}
&=
\begin{bmatrix}
\sum\limits_{k=1}^{T} \dfrac{\partial L}{\partial \mathbf Y_{k, 1}} & \sum\limits_{k=1}^{T} \dfrac{\partial L}{\partial \mathbf Y_{k, 2}} & \cdots & \sum\limits_{k=1}^{T} \dfrac{\partial L}{\partial \mathbf Y_{k, C}}
\end{bmatrix} \\\\
&= \mathbf 1_{1\times (T)} \dfrac{\partial L}{\partial \mathbf Y}
\end{aligned}
$$

## 自注意力层反向传播

之后写。

## 参考资料

- [激活函数 softmax 的反向推导](https://blog.csdn.net/bowenlaw/article/details/125237713)
- [cs231n Handout - Backpropagation for a Linear Layer](https://cs231n.stanford.edu/handouts/linear-backprop.pdf)
