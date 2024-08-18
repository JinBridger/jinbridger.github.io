---
title: "llm.c 源码解析 - GPT-2 结构"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# llm.c 源码解析 - GPT-2 结构

llm.c 是一个开源的用 Cuda 与 C 写成的简易 LLM 推理框架，用大约 1700 行代码完成了对于 GPT-2 模型的训练以及推理。在解析 llm.c 的源码之前，最好先了解一下 GPT-2 的结构。下文中 GPT-2 一般代指 GPT-2 Small, 也就是 124M 参数量的 GPT-2.

## GPT-2 概览

GPT-2 模型主要的结构由三部分组成：
- 嵌入层：将 Token 从文本变成矩阵组成的隐藏表示
- Transformer 层：对隐藏表示进行注意力计算
- 输出层：将隐藏表示转为文本输出

下图列出了 GPT-2 的主要结构以及对应的参数量，右侧一些字母的含义如下：

- `V`: Vocabs, 模型词表大小
- `C`: Channel, 模型隐藏层大小
- `T`: Max Token, 模型最大输入长度
- `L`: Layers, Transformer 层的数量


<div align="center">
<img src="/image/mlsys/analyze-llm-c-part-1/GPT-2-overview.svg" width="100%">
    <br>
    <div style="border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">GPT-2 概览</div>
	<!-- <img src="" >
    msdos -->
</div>

{{< hint info >}}
下面的图示中, 矩阵的大小会标在矩阵的左上角，矩阵乘法与加法会用 ⨂ 与 ⨁ 表示。
{{< /hint >}}


## 嵌入层 - Text & Position Embed

嵌入层的工作是将输入的 Token 转为可以进行计算的矩阵。

第一步要做的事情就是将输入的序列转为一个个 index. 比如 How are you 这句话会被转为 2437, 389, 345 这三个数, 之后用 One Hot 编码为三个 1 * 50257 的向量。

在编码为三个向量以后，就是对每个向量进行词嵌入与位置嵌入。

先说词嵌入，词嵌入就是将矩阵与词嵌入权重 `wte` 进行相乘，得到词嵌入向量。

而位置嵌入，则是取出这个位置对应的 位置嵌入矩阵 `wpe` 的行. 比如如果这个词是第三个 Token, 就取出 `wpe[3]`

最后将词嵌入得到的向量与位置嵌入得到的向量相加就可以得到最后的向量。对每个向量都这样做，就可以得到嵌入完成的矩阵。

<div align="center">
<img src="/image/mlsys/analyze-llm-c-part-1/embedding.svg" width="100%">
    <br>
    <div style="border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">嵌入层 - Text & Position Embed</div>
	<!-- <img src="" >
    msdos -->
</div>

## Transformer 层 - Transformer

### 层归一化 - Layer Norm

层归一化就是对上一层输入的矩阵中的每个向量进行归一化并乘以 weight $\gamma$ 和 bias $\beta$

<div align="center">
<img src="/image/mlsys/analyze-llm-c-part-1/layer-norm.svg" width="90%">
    <br>
    <div style="border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">层归一化 - Layer Norm</div>
	<!-- <img src="" >
    msdos -->
</div>

### 自注意力层 - Self Attention

GPT-2 的自注意力层并没有掩码, 这一点跟一些 LLM 不太一样。剩下的就是经典的 Attention 公式: $$\text{Attention} = \text{softmax}(\frac{QK^T}{\sqrt{d_k}})V$$
最后还要把输出的矩阵再做一次线性的投射，投射到隐藏层的维度上面。

<div align="center">
<img src="/image/mlsys/analyze-llm-c-part-1/attention.svg" width="90%">
    <br>
    <div style="border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">自注意力层 - Self Attention</div>
	<!-- <img src="" >
    msdos -->
</div>

自注意力层的输出还要与最开始的输入做一个加法，从而组成残差网络。（概览图里面 Self Attention 与 Layer Norm 之间还有个加法）

### 前馈网络 - Feed Forward

在自注意力层与前馈网络之间还有一个层归一化，上面已经提到过不再赘述。

<div align="center">
<img src="/image/mlsys/analyze-llm-c-part-1/feed-forward.svg" width="90%">
    <br>
    <div style="border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">前馈网络 - Feed Forward</div>
	<!-- <img src="" >
    msdos -->
</div>

这一层的输出还要与之前的输入做一个加法，从而组成残差网络。（概览图里面 Feed-Forward 下面还有个加法）

至此就走完了一个完整的 Transformer 块的流程，对于 GPT-2 需要重复 12 次 Transformer 块。

## 输出层

### 线性层与 Softmax 层 - Linear & Softmax

线性层的功能是将隐藏层投射回 Vocab 大小。注意到线性层之前还有一个 Layer Norm 层，上面已经提到过不再赘述。最后将线性层的输出的每一行做 Softmax 就可以得到最终的输出概率。

<div align="center">
<img src="/image/mlsys/analyze-llm-c-part-1/output.svg" width="70%">
    <br>
    <div style="border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">输出层</div>
	<!-- <img src="" >
    msdos -->
</div>

## 参考资料

- llm.c https://github.com/karpathy/llm.c
- llm-viz https://bbycroft.net/llm
- The Illustrated GPT-2 (Visualizing Transformer Language Models) https://jalammar.github.io/illustrated-gpt2/