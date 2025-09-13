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

{{< hint info >}}
建议阅读本文前先阅读前文 [llm.c 源码解析 - GPT-2 结构](../analyze-llm-c-part-1)
{{< /hint >}}

llm.c 是一个开源的用 Cuda 与 C 写成的简易 LLM 推理框架，用大约 1700 行代码完成了对于 GPT-2 模型的训练以及推理。

{{< hint info >}}
本文中的下标字母可能会有点奇怪，这是为了与 llm.c 中的循环变量保持一致。
{{< /hint >}}

## Softmax 层反向传播

下面单看一个 token 是怎么计算 Loss的，假设数据集中这个位置的 token 在词表中的下标为 $i_x$, 设以下符号对应:

- $L$：损失函数
- $\mathbf P$：长度为词表大小的向量, 其中 $\mathbf P_i$ 代表这个 token 为词表里面编号为 $i$ 的 token 的概率 (Prob), 通过 Softmax 由 $\mathbf S$ 计算而来
- $\mathbf S$：长度为词表大小的向量, 其中 $\mathbf S_i$ 代表这个 token 为词表里面编号为 $i$ 的 token 的分数 (Score)

GPT-2 采用的 Loss 计算方式是交叉熵损失函数 (Cross Entropy) 即

$$
L=-\log{\mathbf P_{i_x}}
$$

在计算完 Loss 之后就是反向传播。

这一层我们需要计算的是 Score 相对于 Loss 的梯度, 也就是 $\dfrac{\partial L}{\partial \mathbf S}$，由于 $L$ 是一个标量，$S$ 是一个 $1\times V$ 的向量，因此这个梯度应该也是一个 $1\times V$ 的向量。

要计算这个梯度，就需要知道这个向量中每个元素的表达式。根据链式求导法则，这个向量中的第 $i$ 个元素 $\dfrac{\partial L}{\partial \mathbf S_i}$ 的表达式如下： $$\dfrac{\partial L}{\partial \mathbf S_i}=\sum\limits_{j=1}^V \dfrac{\partial L}{\partial \mathbf P_j}\dfrac{\partial \mathbf P_j}{\partial \mathbf S_i}.$$ 因此只需要计算出 $\dfrac{\partial L}{\partial \mathbf P_j}$ 与 $\dfrac{\partial \mathbf P_j}{\partial \mathbf S_i}$ 即可。

对于 $\dfrac{\partial L}{\partial \mathbf P_j}$, 根据交叉熵公式 $L=-\log{\mathbf P_{i_x}}$ 可得:

$$
\dfrac{\partial L}{\partial \mathbf P_j} =
\begin{cases}
-\dfrac{1}{\mathbf P_{i_x}}, &j=t \\\\
0, &j\neq t
\end{cases}
$$

对于 $\dfrac{\partial \mathbf P_j}{\partial \mathbf S_i}$, 根据 Softmax 公式 $P_i=\dfrac{\exp (S_i)}{\sum_{i=1}^V \exp(S_i)}$ 可得:

$$
\dfrac{\partial \mathbf P_j}{\partial \mathbf S_i} =
\begin{cases}
\mathbf P_j(1-\mathbf P_i), &i=j \\\\
-\mathbf P_j\mathbf P_i, &i\neq j
\end{cases}
$$

因此

$$
\begin{aligned}
\dfrac{\partial L}{\partial \mathbf S_i}&=\sum\limits_{j=1}^V \dfrac{\partial L}{\partial \mathbf P_j}\dfrac{\partial \mathbf P_j}{\partial \mathbf S_i} =
\begin{cases}
\mathbf P_i - 1, &i={i_x} \\\\
\mathbf P_i, &i\neq {i_x}
\end{cases}
\end{aligned}
$$

对比 llm.c 中的 C 语言代码：
```c
void crossentropy_softmax_backward(float* dlogits,
                           float* dlosses, float* probs, int* targets,
                           int B, int T, int V, int Vp) {
    // backwards through both softmax and crossentropy
    for (int b = 0; b < B; b++) {
        for (int t = 0; t < T; t++) {
            float* dlogits_bt = dlogits + b * T * Vp + t * Vp;
            float* probs_bt = probs + b * T * Vp + t * Vp;
            float dloss = dlosses[b * T + t];
            int ix = targets[b * T + t];
            // note we only loop to V, leaving the padded dimensions
            // of dlogits untouched, so gradient there stays at zero
            for (int i = 0; i < V; i++) {
                float p = probs_bt[i];
                float indicator = i == ix ? 1.0f : 0.0f;
                dlogits_bt[i] += (p - indicator) * dloss;
            }
        }
    }
}
```

## 线性层反向传播

线性层可以表示为 $\mathbf Y=\mathbf X \mathbf W+\mathbf B$, 其中：
- $\mathbf X$ 为输入，为 $T \times C_1$ 的矩阵
- $\mathbf Y$ 为输出, 为 $T \times C_2$ 的矩阵
- $\mathbf W$ 为权重，为 $C_1\times C_2$ 的矩阵
- $\mathbf B$ 为偏置，为 $1\times C_2$ 的向量

要进行反向传播，还需要有损失值 $L$

需要求取的梯度包括三个:

- $L$ 对 $\mathbf X$ 的梯度 $\dfrac{\partial L}{\partial \mathbf X}$, 用于传入下一层
- $L$ 对 $\mathbf W$ 的梯度 $\dfrac{\partial L}{\partial \mathbf W}$, 用于更新这一层的 $\mathbf W$
- $L$ 对 $\mathbf B$ 的梯度 $\dfrac{\partial L}{\partial \mathbf B}$, 用于更新这一层的 $\mathbf B$

### L 对于 X 的梯度

由于 $L$ 是一个标量，而 $\mathbf X$ 是一个 $T \times C_1$ 的矩阵。因此 $\dfrac{\partial L}{\partial \mathbf X}$ 也应当是一个 $T \times C_1$ 的矩阵。
要求这个矩阵的值，就需要知道这个矩阵中每个元素的表达式, 即 $\dfrac{\partial L}{\partial \mathbf X_{t,i}}$

根据 $\mathbf Y=\mathbf X \mathbf W+\mathbf B$, 有

$$
\mathbf Y_{t_2, o}=\sum\limits_{k=1}^{C_2} \mathbf X_{t_2,k} \mathbf W_{k,o} + \mathbf B_o
$$

因此

$$\dfrac{\partial \mathbf Y_{t_2, o}}{\partial \mathbf X_{t_1,i}}=\dfrac{\partial (\sum\limits_{k=1}^{C_2} \mathbf X_{t_2,k} \mathbf W_{k,o} + \mathbf B_o)}{\partial \mathbf X_{t_1,i}} = 
\begin{cases}
0 &t_1\neq t_2\\\\
\mathbf W_{i, o} &t_1=t_2
\end{cases}
$$

根据链式法则有

$$
\begin{aligned}
\dfrac{\partial L}{\partial \mathbf X_{t_1,i}}
&=
\sum\limits_{t_2=1}^T\sum\limits_{o=1}^{C_2}\dfrac{\partial L}{\partial \mathbf Y_{t_2,o}}\dfrac{\partial \mathbf Y_{t_2,o}}{\partial \mathbf X_{t_1,i}} \\\\
&=
\sum\limits_{o=1}^{C_2}\sum\limits_{t_2=1}^T\dfrac{\partial L}{\partial \mathbf Y_{t_2,o}}\dfrac{\partial \mathbf Y_{t_2,o}}{\partial \mathbf X_{t_1,i}} \\\\
&= \sum_{o=1}^{{C_2}} \dfrac{\partial L}{\partial \mathbf Y_{t_1,o}}\mathbf W_{i, o}
\end{aligned}
$$

也就是：

$$
\begin{aligned}
\dfrac{\partial L}{\partial \mathbf X_{t,i}}
&= \sum_{o=1}^{C_2} \dfrac{\partial L}{\partial \mathbf Y_{t,o}}\mathbf W_{i, o}
\end{aligned}
$$

这里 $C_2$ 对应 $\mathbf Y$ 的维度，也就是输出维度(Output Channel), 记作 OC. 对比 llm.c 里面的 C 语言代码，需要注意的是 llm.c 里面的 `weight` 存的是转置的矩阵：

```c
void matmul_backward(float* dinp, float* dweight, float* dbias,
                     const float* dout, const float* inp, const float* weight,
                     int B, int T, int C, int OC) {
    // most of the running time is spent here and in matmul_forward
    // this backward could be done in a single "round" of loops
    // but that doesn't afford an efficient parallelization strategy

    // backward into inp first, parallelize over B,T
    #pragma omp parallel for collapse(2)
    for (int b = 0; b < B; b++) {
        for (int t = 0; t < T; t++) {
            const float* dout_bt = dout + b * T * OC + t * OC;
            float* dinp_bt = dinp + b * T * C + t * C;
            for (int o = 0; o < OC; o++) {
                const float* wrow = weight + o*C;
                float d = dout_bt[o];
                for (int i = 0; i < C; i++) {
                    dinp_bt[i] += wrow[i] * d;
                }
            }
        }
    }
    ...
}
```

### L 对于 W 的梯度

同样的方法，可以求出 

$$
\begin{aligned}
\dfrac{\partial L}{\partial \mathbf W_{i,o}}
&= \sum\limits_{t=1}^{T} \dfrac{\partial L}{\partial \mathbf Y_{t, o}}\mathbf X_{t, i}
\end{aligned}
$$

对比 llm.c 里面的 C 语言代码：

```c
void matmul_backward(float* dinp, float* dweight, float* dbias,
                     const float* dout, const float* inp, const float* weight,
                     int B, int T, int C, int OC) {
    ...
    // backward into weight/bias, parallelize over output channels OC
    #pragma omp parallel for
    for (int o = 0; o < OC; o++) {
        for (int b = 0; b < B; b++) {
            for (int t = 0; t < T; t++) {
                const float* dout_bt = dout + b * T * OC + t * OC;
                const float* inp_bt = inp + b * T * C + t * C;
                float* dwrow = dweight + o*C;
                float d = dout_bt[o];
                ...
                for (int i = 0; i < C; i++) {
                    dwrow[i] += inp_bt[i] * d;
                }
            }
        }
    }
}
```

### L 对于 B 的梯度

同样的方法，可以求出 

$$
\begin{aligned}
\dfrac{\partial L}{\partial \mathbf B_o}
&=
\sum\limits_{t=1}^{T} \dfrac{\partial L}{\partial \mathbf Y_{t, o}}
\end{aligned}
$$

对比 llm.c 里面的 C 语言代码：

```c
void matmul_backward(float* dinp, float* dweight, float* dbias,
                     const float* dout, const float* inp, const float* weight,
                     int B, int T, int C, int OC) {
    ...
    // backward into weight/bias, parallelize over output channels OC
    #pragma omp parallel for
    for (int o = 0; o < OC; o++) {
        for (int b = 0; b < B; b++) {
            for (int t = 0; t < T; t++) {
                const float* dout_bt = dout + b * T * OC + t * OC;
                ...
                float d = dout_bt[o];
                if (dbias != NULL) { dbias[o] += d; }
                ...
            }
        }
    }
}
```

## 自注意力层反向传播

自注意力公式本质上就是矩阵相乘套一个 Softmax, 所以形式上就是前两者的结合体。反向传播的时候先求 Softmax 的梯度，之后再求矩阵相乘的梯度。这里就不推导具体公式了，直接放对应代码：

```c
void attention_backward(float* dinp, float* dpreatt, float* datt,
                        float* dout, float* inp, float* att,
                        int B, int T, int C, int NH) {
    // inp/dinp are (B, T, 3C) Q,K,V
    // att/datt/dpreatt are (B, NH, T, T)
    // dout is (B, T, C)
    int C3 = C*3;
    int hs = C / NH; // head size
    float scale = 1.f / sqrtf(hs);

    for (int b = 0; b < B; b++) {
        for (int t = 0; t < T; t++) {
            for (int h = 0; h < NH; h++) {
                float* att_bth = att + b*NH*T*T + h*T*T + t*T;
                float* datt_bth = datt + b*NH*T*T + h*T*T + t*T;
                float* dpreatt_bth = dpreatt + b*NH*T*T + h*T*T + t*T;
                float* dquery_t = dinp + b * T * C3 + t * C3 + h * hs;
                float* query_t = inp + b * T * C3 + t * C3 + h * hs;

                ...

                // backward pass 2 & 3, the softmax
                // note that softmax (like e.g. tanh) doesn't need the input (preatt) to backward
                for (int t2 = 0; t2 <= t; t2++) {
                    for (int t3 = 0; t3 <= t; t3++) {
                        float indicator = t2 == t3 ? 1.0f : 0.0f;
                        float local_derivative = att_bth[t2] * (indicator - att_bth[t3]);
                        dpreatt_bth[t3] += local_derivative * datt_bth[t2];
                    }
                }

                // backward pass 1, the query @ key matmul
                for (int t2 = 0; t2 <= t; t2++) {
                    float* key_t2 = inp + b * T * C3 + t2 * C3 + h * hs + C; // +C because it's key
                    float* dkey_t2 = dinp + b * T * C3 + t2 * C3 + h * hs + C; // +C because it's key
                    for (int i = 0; i < hs; i++) {
                        // in the forward pass this was:
                        // preatt_bth[t2] += (query_t[i] * key_t2[i]) * scale;
                        // so now we have:
                        dquery_t[i] += key_t2[i] * dpreatt_bth[t2] * scale;
                        dkey_t2[i] += query_t[i] * dpreatt_bth[t2] * scale;
                    }
                }
            }
        }
    }
}
```

## 参考资料

- [激活函数 softmax 的反向推导](https://blog.csdn.net/bowenlaw/article/details/125237713)
- [cs231n Handout - Backpropagation for a Linear Layer](https://cs231n.stanford.edu/handouts/linear-backprop.pdf)
