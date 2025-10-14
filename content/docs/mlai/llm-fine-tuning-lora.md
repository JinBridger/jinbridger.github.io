---
title: "LLM 微调学习笔记: LoRA"
weight: 1
date: 2025-10-14
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# LLM 微调学习笔记: LoRA

LoRA (Low-Rank Adaption) 是一种常用的 LLM 微调方法.

## 秩, 满秩分解与低秩分解

LoRA 的 R 指的就是秩.
因此我们有必要复习一下秩是什么.

根据维基百科的定义,
在线性代数中, 一个矩阵 $A$ 的列秩是列向量生成的最大线性无关组的向量个数.
类似地, 行秩是矩阵 $A$ 的线性无关的横行的个数.
矩阵的列秩和行秩总是相等的, 因此它们可以简单地称作矩阵 $A$ 的秩.
计作 $\text r(A)$

直观上讲, 一个矩阵的秩反映了这个矩阵的信息维度.
如果一个矩阵存在线性相关的行/列,
那么就意味着这个矩阵损失了高维的信息.
对应的, 
我们无法再从这个矩阵构造一组正交基.

举个最简单的例子, 假设我们有一个二维方阵 $A$, 如果它存在线性相关的列:
$$
A=
\begin{bmatrix}
    1 & 2\\\\
    2 & 4\\\\
\end{bmatrix}
$$
无论你怎么尝试对它的行向量/列向量做线性变换,
你都只能得到一个在 $(1, 2)$ 方向上的向量.
因此这个矩阵相当于丢失了二维的信息, 只能表示一维信息.

一个直观的想法是,
如果一个矩阵 $A$ 的秩是 $\text r(A)$,
那么我们可以从 $\text r(A)$ 个向量组成的基来构造这个矩阵.
这个过程就是满秩分解.
具体来说, 假设矩阵 $A\in \mathbb R^{m\times n}$.
其秩为 $r=\text r(A)$.
满秩分解就是找到矩阵 $B\in \mathbb R^{m\times r}$ 与 $B\in \mathbb R^{n\times r}$,
使得 $$A=BC^\top$$
其中 $B$ 与 $C$ 均为列满秩. 
不难发现, 满秩分解是无损的 (是严格的等号).

接下来让我们把这种思想推远一点,
能不能用一个低维度的基来近似表示 $A$?
这种想法就是低秩分解.
假设我们有矩阵 $A\in \mathbb R^{m\times n}$.
低秩分解做的就是将其拆成两个矩阵的乘积:
$$
A\approx UV^\top
$$
其中 $U\in \mathbb R^{m\times k}$, $V\in \mathbb R^{n\times k}$.
而这个 $k < \text r(A)$.

你可能已经注意到了, 低秩分解是约等于符号.
这是由于丢失了维度信息, 我们无法完全重建原来的矩阵.

## LoRA

对于一个全连接参数层 $W^{n\times n}$.
一般的微调是单独建一个 $\Delta W$.
推理时计算 $(W+\Delta W)X$.
反向传播时只修改 $\Delta W$.
这意味着对于一个 4096 大小的参数矩阵,
我们要保存约 1600 万个参数.
这导致训练成本非常高.

LoRA 正是将低秩分解应用于微调的方法.
它将 $\Delta W^{n\times n}$ 拆成 $A^{n\times r}$ 与 $B^{n\times r}$.
也就是 $\Delta W=AB^\top$.

对于一个 4096 大小的 $W$,
$A$ 与 $B$ 的大小可能是 4096 * 8.
这样参数量就骤降到了原来的千分之二.
训练时, 由于输出为 $Y=WX + AB^\top X$,
因此对应的梯度为:
$$
\frac{\partial Y}{\partial A} = X^\top B~~~~\frac{\partial Y}{\partial B} = X^\top A\\\\
$$

## 为什么 LoRA 的效果不差?

这是一个很有意思的问题.
理论上来说, 既然我们做了低秩分解,
照前文所述, 性能应该会损失才对.
但实际上性能并没有损失多少.

根据 LoRA 论文中的解释是:
「the low-rank adaptation matrix potentially *amplifies the important features for specific downstream tasks that were learned but not emphasized in the general pre-training model.*」
也就是说,
特定的下游任务的 $\Delta W$ 常常是不满秩的.
因此可以进行低秩分解而几乎不损失性能.

作为对比, 预训练要求模型在各个方向上的能力都要得到训练,
因此也就没办法进行低秩分解.
