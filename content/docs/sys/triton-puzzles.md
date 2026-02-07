---
title: "Triton Puzzles 记录"
weight: 1
date: 2026-02-06
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Triton Puzzles 记录

Triton Puzzles 是一份用来学习 Triton 的练习题.

## 配置

我用的是 [Triton-Puzzles-Lite](https://github.com/SiriusNEO/Triton-Puzzles-Lite).

对应的 triton 版本为 `triton==3.1.0`.

安装时需要注意指定 `numpy<2.0`. 否则运行会有问题.

## 题目

### Puzzle 1: Constant Add

Add a constant to a vector. Uses one program id axis. 
Block size `B0` is always the same as vector `x` with length `N0`.

$$z_i = 10 + x_i \text{ for } i = 1\ldots N_0$$

直接 load 然后加回去就可以.

```py
@triton.jit
def add_kernel(x_ptr, z_ptr, N0, B0: tl.constexpr):
    # We name the offsets of the pointers as "off_"
    off_x = tl.arange(0, B0)
    x = tl.load(x_ptr + off_x)
    output = x + 10
    tl.store(z_ptr + off_x, output)
    return
```

### Puzzle 2: Constant Add Block

Add a constant to a vector. Uses one program block axis (no `for` loops yet). 
Block size `B0` is now smaller than the shape vector `x` which is `N0`.


$$z_i = 10 + x_i \text{ for } i = 1\ldots N_0$$

直接 load 然后加回去就可以.


```py
@triton.jit
def add_mask2_kernel(x_ptr, z_ptr, N0, B0: tl.constexpr):
    pid = tl.program_id(0)
    offset = tl.arange(0, B0) + pid * B0
    x = tl.load(x_ptr + offset, offset < N0)
    output = x + 10
    tl.store(z_ptr + offset, output, offset < N0)
    return
```

### Puzzle 3: Outer Vector Add

Add two vectors.

Uses one program block axis. Block size `B0` is always the same as vector `x` length `N0`.
Block size `B1` is always the same as vector `y` length `N1`.

$$z_{j, i} = x_i + y_j\text{ for } i = 1\ldots B_0,\ j = 1\ldots B_1$$

在 triton 中, 两个不同形状的向量进行操作会触发广播,
从最后一维开始对齐.
对每一维:
- 如果两个维度相等: OK
- 如果其中一个是 1: 可以广播成另一个
- 否则: shape 不兼容（编译期/报错）

这里就用了广播机制.

```python
@triton.jit
def add_vec_kernel(x_ptr, y_ptr, z_ptr, N0, N1, B0: tl.constexpr, B1: tl.constexpr):
    off_x = tl.arange(0, B0)[:, None]
    off_y = tl.arange(0, B1)[None, :]

    x = tl.load(x_ptr + off_x)
    y = tl.load(y_ptr + off_y)

    z = x + y
    off_z = off_y * B0 + off_x
    tl.store(z_ptr + off_z, z)
    return
```

### Puzzle 4: Outer Vector Add Block

Add a row vector to a column vector.

Uses two program block axes. Block size `B0` is always less than the vector `x` length `N0`.
Block size `B1` is always less than vector `y` length `N1`.

$$z_{j, i} = x_i + y_j\text{ for } i = 1\ldots N_0,\ j = 1\ldots N_1$$

简单的 2D tile.

```python
@triton.jit
def add_vec_block_kernel(
    x_ptr, y_ptr, z_ptr, N0, N1, B0: tl.constexpr, B1: tl.constexpr
):
    block_id_x = tl.program_id(0)
    block_id_y = tl.program_id(1)

    off_x = tl.arange(0, B0)[:, None] + B0 * block_id_x
    off_y = tl.arange(0, B1)[None, :] + B1 * block_id_y

    mask_x = off_x < N0
    mask_y = off_y < N1
    mask_z = mask_x & mask_y

    x = tl.load(x_ptr + off_x, mask_x)
    y = tl.load(y_ptr + off_y, mask_y)

    z = x + y
    off_z = off_y * N0 + off_x
    tl.store(z_ptr + off_z, z, mask_z)
    return
```

### Puzzle 5: Fused Outer Multiplication

Multiply a row vector to a column vector and take a relu.

Uses two program block axes. Block size `B0` is always less than the vector `x` length `N0`.
Block size `B1` is always less than vector `y` length `N1`.


$$z_{j, i} = \text{relu}(x_i \times y_j)\text{ for } i = 1\ldots N_0,\ j = 1\ldots N_1$$

仍然是 2D tile. 填充时注意 mask.

```py
@triton.jit
def mul_relu_block_kernel(
    x_ptr, y_ptr, z_ptr, N0, N1, B0: tl.constexpr, B1: tl.constexpr
):
    block_id_x = tl.program_id(0)
    block_id_y = tl.program_id(1)

    off_x = tl.arange(0, B0)[None, :] + B0 * block_id_x
    off_y = tl.arange(0, B1)[:, None] + B1 * block_id_y

    mask_x = off_x < N0
    mask_y = off_y < N1
    mask_z = mask_x & mask_y

    x = tl.load(x_ptr + off_x, mask_x)
    y = tl.load(y_ptr + off_y, mask_y)

    z = x * y
    off_z = off_y * N0 + off_x
    tl.store(z_ptr + off_z, 0, mask_z)
    tl.store(z_ptr + off_z, z, mask_z & (z > 0))
    return
```


### Puzzle 6: Fused Outer Multiplication - Backwards

Backwards of a function that multiplies a matrix with a row vector and take a relu.

Uses two program blocks. Block size `B0` is always less than the vector `x` length `N0`.
Block size `B1` is always less than vector `y` length `N1`. Chain rule backward `dz`
is of shape `N1` by `N0`


$$f(x, y) = \text{relu}(x_{j, i} \times y_j)\text{ for } i = 1\ldots N_0,\ j = 1\ldots N_1$$

$$dx_{j, i} = f_x'(x, y)_{j, i} \times dz_{j, i}$$

反向传播的梯度是 $$dx_{j, i} = \begin{cases}
    dz_{j, i} \times y_j , &x_{j, i} \times y_j > 0,\\
    0, &\text{else}
\end{cases} $$

```py
@triton.jit
def mul_relu_block_back_kernel(
    x_ptr, y_ptr, dz_ptr, dx_ptr, N0, N1, B0: tl.constexpr, B1: tl.constexpr
):
    block_id_i = tl.program_id(0)
    block_id_j = tl.program_id(1)

    off_i = tl.arange(0, B0)[None, :] + B0 * block_id_i
    off_j = tl.arange(0, B1)[:, None] + B1 * block_id_j
    off_ij = off_j * N0 + off_i

    mask_i = off_i < N0
    mask_j = off_j < N1
    mask_ij = mask_i & mask_j

    x = tl.load(x_ptr + off_ij, mask_ij)
    y = tl.load(y_ptr + off_j, mask_j)
    z = x * y

    dz = tl.load(dz_ptr + off_ij, mask_ij)
    dx = dz * y

    tl.store(dx_ptr + off_ij, 0, mask_ij)
    tl.store(dx_ptr + off_ij, dx, mask_ij & (z > 0))
    return
```


### Puzzle 7: Long Sum

Sum of a batch of numbers.

Uses one program blocks. Block size `B0` represents a range of batches of  `x` of length `N0`.
Each element is of length `T`. Process it `B1 < T` elements at a time.  


$$z_{i} = \sum^{T}_j x_{i,j} =  \text{ for } i = 1\ldots N_0$$

Hint: You will need a for loop for this problem. These work and look the same as in Python.

由于 GPU 寄存器有限, 因此对于长向量只能切分成小段依次处理.

```py
@triton.jit
def sum_kernel(x_ptr, z_ptr, N0, N1, T, B0: tl.constexpr, B1: tl.constexpr):
    pid_0 = tl.program_id(0)

    tl.store(z_ptr + pid_0, 0)

    nblocks = tl.cdiv(T, B1)
    for block in range(nblocks):
        offset = tl.arange(0, B1) + block * B1
        mask = offset < T
        offset = offset + pid_0 * T
        x = tl.load(x_ptr + offset, mask)
        sum_x = tl.sum(x, axis=0)
        z = tl.load(z_ptr + pid_0)
        tl.store(z_ptr + pid_0, z + sum_x)
    return
```

### Puzzle 8: Long Softmax

Softmax of a batch of logits.

Uses one program block axis. Block size `B0` represents the batch of `x` of length `N0`.
Block logit length `T`.   Process it `B1 < T` elements at a time.  

$$z_{i, j} = \text{softmax}(x_{i,1} \ldots x_{i, T}) \text{ for } i = 1\ldots N_0$$

Note softmax needs to be computed in numerically stable form as in Python. In addition in Triton 
they recommend not using `exp` but instead using `exp2`. You need the identity

$$\exp(x) = 2^{\log_2(e) x}$$

Advanced: there one way to do this with 3 loops. You can also do it with 2 loops if you are clever. 
Hint: you will find this identity useful:

$$\exp(x_i - m) =  \exp(x_i - m/2 - m/2) = \exp(x_i - m/ 2) /  \exp(m/2)$$

这里用了一个 trick, 用一个循环计算完 $\sum \exp(x_i)$.
具体来说是
$$
\begin{aligned}
\exp(x_i - m_\text{new}) &=  \exp((x_i - m_\text{old}) - (m_\text{new}-m_\text{old})) \\
&= \exp(x_i - m_\text{old}) \times  \exp(m_\text{old}-m_\text{new})
\end{aligned}
$$
计算时保存一个旧的最值 $m_\text{old}$.

```py
@triton.jit
def softmax_kernel(x_ptr, z_ptr, N0, N1, T, B0: tl.constexpr, B1: tl.constexpr):
    """2 loops ver."""
    pid_0 = tl.program_id(0)
    log2_e = 1.44269504

    nblocks = tl.cdiv(T, B1)

    d = tl.full((), 0, tl.float32)
    x_max = tl.full((), 0, tl.float32)
    for block in range(nblocks):
        offset = tl.arange(0, B1) + block * B1
        mask = offset < T
        offset = offset + pid_0 * T
        x = tl.load(x_ptr + offset, mask)

        old_max = x_max
        x_max = max(tl.max(x, axis=0), x_max)
        exp_diff = tl.exp2((old_max - x_max) * log2_e)
        d = d * exp_diff

        d = d + tl.sum(tl.exp2((x - x_max) * log2_e), axis=0)

    for block in range(nblocks):
        offset = tl.arange(0, B1) + block * B1
        mask = offset < T
        offset = offset + pid_0 * T
        x = tl.load(x_ptr + offset, mask)
        x = tl.exp2((x - x_max) * log2_e)
        softmax = x / d
        tl.store(z_ptr + offset, softmax, mask)

    return
```

### Puzzle 9: Simple FlashAttention

A scalar version of FlashAttention.

Uses zero programs. Block size `B0` represent the batches of `q` to process out of `N0`. Sequence length is `T`. Process it `B1 < T` elements (`k`, `v`) at a time for some `B1`.

$$
z_{i} = \sum_{j=1}^{T} \text{softmax}(q_i k_1, \ldots, q_i k_T)_j v_{j} \text{ for } i = 1\ldots N_0
$$

This can be done in 1 loop using a similar trick from the last puzzle.

Hint: Use `tl.where` to mask `q dot k` to -inf to avoid overflow (NaN).

同样的处理.

```py
@triton.jit
def flashatt_kernel(
    q_ptr, k_ptr, v_ptr, z_ptr, N0, T, B0: tl.constexpr, B1: tl.constexpr
):
    block_id_i = tl.program_id(0)
    log2_e = 1.44269504
    myexp = lambda x: tl.exp2(log2_e * x)

    off_qz = tl.arange(0, B0)[:, None] + block_id_i * B0
    mask_qz = off_qz < T
    q = tl.load(q_ptr + off_qz, mask_qz)

    nblk_kv = tl.cdiv(T, B1)
    d = tl.zeros((B0,), dtype=tl.float32)[:, None]
    x_max = tl.zeros((B0,), dtype=tl.float32)[:, None]
    for blk in range(nblk_kv):
        off_kv = tl.arange(0, B1)[None, :] + blk * B1
        mask_kv = off_kv < T

        k = tl.load(k_ptr + off_kv, mask_kv)

        x = q * k

        old_x_max = x_max
        x_max = max(x_max, tl.max(x, axis=1)[:, None])
        exp_diff = myexp(old_x_max - x_max)
        d = d * exp_diff

        d = d + tl.sum(myexp(x - x_max), axis=1)[:, None]

    z = tl.zeros((B0,), dtype=tl.float32)[:, None]
    for blk in range(nblk_kv):
        off_kv = tl.arange(0, B1)[None, :] + blk * B1
        mask_kv = off_kv < T

        k = tl.load(k_ptr + off_kv, mask_kv)
        v = tl.load(v_ptr + off_kv, mask_kv)

        x = q * k

        z = z + tl.sum(myexp(x - x_max) / d * v, axis=1)[:, None]

    tl.store(z_ptr + off_qz, z, mask_qz)

    return
```

### Puzzle 10: Two Dimensional Convolution

A batched 2D convolution.

Uses one program id axis. Block size `B0` represent the batches to process out of `N0`.
Image `x` is size is `H` by `W` with only 1 channel, and kernel `k` is size `KH` by `KW`.


$$z_{i, j, l} = \sum_{oj, ol}^{j+oj\le H, l+ol\le W} k_{oj,ol} \times x_{i,j + oj, l + ol} 
    \text{ for } i = 1\ldots N_0 \text{ for } j = 1\ldots H \text{ for } l = 1\ldots W$$

卷积, 每个线程处理 `B0` 张 image.

```py
@triton.jit
def conv2d_kernel(
    x_ptr, k_ptr, z_ptr, N0, H, W, KH: tl.constexpr, KW: tl.constexpr, B0: tl.constexpr
):
    block_id_i = tl.program_id(0)
    off_blk = block_id_i * B0 * H * W
    for img in range(B0):
        off_img = img * H * W
        for i in range(H):
            for j in range(W):
                off_pixel = i * W + j
                off_total = off_blk + off_img + off_pixel

                off_x_i = tl.arange(i, i + KH)[:, None]
                off_x_j = tl.arange(j, j + KW)[None, :]
                off_x = off_blk + off_img + off_x_i * W + off_x_j
                mask_x = (off_x_i < H) & (off_x_j < W)
                x = tl.load(x_ptr + off_x, mask_x, 0)

                off_k = tl.arange(0, KH)[:, None] * KW + tl.arange(0, KW)[None, :]
                k = tl.load(k_ptr + off_k)

                tl.store(
                    z_ptr + off_total,
                    tl.sum(x * k),
                    off_total < N0 * H * W,
                )
    return
```

### Puzzle 11: Matrix Multiplication

A blocked matrix multiplication.

Uses three program id axes. Block size `B2` represent the batches to process out of `N2`.
Block size `B0` represent the rows of `x` to process out of `N0`. Block size `B1` represent the cols 
of `y` to process out of `N1`. The middle shape is `MID`.

$$z_{i, j, k} = \sum_{l} x_{i,j, l} \times y_{i, l, k} \text{ for } i = 1\ldots N_2, j = 1\ldots N_0, k = 1\ldots N_1$$

You are allowed to use `tl.dot` which computes a smaller mat mul.

Hint: the main trick is that you can split a matmul into smaller parts.

$$z_{i, j, k} = \sum_{l=1}^{L/2} x_{i,j, l} \times y_{i, l, k} +  \sum_{l=L/2}^{L} x_{i,j, l} \times y_{i, l, k}$$

矩阵相乘, 使用 2D tile.

```py
@triton.jit
def dot_kernel(
    x_ptr,
    y_ptr,
    z_ptr,
    N0,
    N1,
    N2,
    MID,
    B0: tl.constexpr,
    B1: tl.constexpr,
    B2: tl.constexpr,
    B_MID: tl.constexpr,
):
    block_id_j = tl.program_id(0)  # Row of X
    block_id_k = tl.program_id(1)  # Col of Y
    block_id_i = tl.program_id(2)  # Batch

    off_i = block_id_i * B2
    off_j = tl.arange(0, B0)[:, None] + block_id_j * B0
    off_k = tl.arange(0, B1)[None, :] + block_id_k * B1

    off_z = off_i * N0 * N1 + off_j * N1 + off_k
    mask_z = (off_j < N0) & (off_k < N1)
    z = tl.zeros((B0, B1), dtype=tl.float32)

    blk_size = 16
    nblk_l = tl.cdiv(MID, blk_size)

    for blk in range(nblk_l):
        off_blk = tl.arange(0, blk_size) + blk * blk_size
        off_x = off_i * N0 * MID + off_j * MID + off_blk[None, :]
        mask_x = (off_j < N0) & (off_blk[None, :] < MID)
        x = tl.load(x_ptr + off_x, mask_x)

        off_y = off_i * MID * N1 + off_blk[:, None] * N1 + off_k
        mask_y = (off_blk[:, None] < MID) & (off_k < N1)
        y = tl.load(y_ptr + off_y, mask_y)

        z = z + tl.dot(x, y)

    tl.store(z_ptr + off_z, z, mask_z)

    return
```

### Puzzle 12: Quantized Matrix Mult

When doing matrix multiplication with quantized neural networks a common strategy is to store the weight matrix in lower precision, with a shift and scale term.

For this problem our `weight` will be stored in 4 bits. We can store `FPINT` of these in a 32 bit integer. In addition for every `group` weights in order we will store 1 `scale` float value and 1 `shift` 4 bit value. We store these for the column of weight. The `activation`s are stored separately in standard floats.

Mathematically it looks like.

$$z_{j, k} = \sum_{l} sc_{j, \frac{l}{g}} (w_{j, l} - sh_{j, \frac{l}{g}}) \times y_{l, k} 
    \text{ for } j = 1\ldots N_0, k = 1\ldots N_1$$

Where `g` is the number of groups (`GROUP`).

However, it is a bit more complex since we need to also extract the 4-bit values into floats to begin.

Note:
- We don't consider batch size, i.e. `i`, in this puzzle.
- Remember to unpack the `FPINT` values into separate 4-bit values. This contains some shape manipulation.

最复杂的一个.
题干讲的有点绕, 我的理解如下:
原始权重可以表示为:
$x=s\times(w-o)$, 其中:
- $x$ 是原始的权重.
  - 大小为 `N0 * MID`.
- $s$ 是缩放因子. 格式为 `float32`.
  - 输入格式是 `float32`.
  - 每 `GROUP` 个原始权重共用一个.
  - 大小为 `N0 * MID / GROUP`
- $w$ 是量化后的权重. 格式为 `int4`.
  - 输入格式是 `int32`.
  - 一个 `int32` 中可以保存 `FPINT` 个权重.
  - 大小为 `N0 * MID / FPINT`
- $o$ 是偏移. 格式为 `int4`.
  - 输入格式是 `int32`.
  - 一个 `int32` 中可以保存 `FPINT` 个权重.
  - 每 `GROUP` 个原始权重共用一个.
  - 大小为 `N0 * MID / FPINT / GROUP`

```py
@triton.jit
def quant_dot_kernel(
    scale_ptr,
    offset_ptr,
    weight_ptr,
    activation_ptr,
    z_ptr,
    N0,
    N1,
    MID,
    B0: tl.constexpr,
    B1: tl.constexpr,
    B_MID: tl.constexpr,
):
    block_id_j = tl.program_id(0)
    block_id_k = tl.program_id(1)

    def extract(x):
        over = tl.arange(0, 8) * 4
        mask = tl.full([8], 0xF, tl.int32)
        return (x >> over) & mask

    off_j = tl.arange(0, B0)[:, None] + block_id_j * B0
    off_k = tl.arange(0, B1)[None, :] + block_id_k * B1
    off_z = off_j * N1 + off_k
    mask_z = (off_j < N0) & (off_k < N1)
    z = tl.zeros((B0, B1), dtype=tl.float32)

    nblk_l = tl.cdiv(MID, B_MID)

    MID_S = MID // GROUP
    MID_W = MID // FPINT
    MID_O = MID // FPINT // GROUP

    B_MID_S = B_MID // GROUP
    B_MID_W = B_MID // FPINT
    B_MID_O = B_MID // FPINT // GROUP

    for blk in range(nblk_l):
        off_s_l = tl.arange(0, B_MID_S) + blk * B_MID_S
        off_s = off_j * MID_S + off_s_l[None, :]
        mask_s = (off_j < N0) & (off_s_l[None, :] < MID_S)
        s = tl.load(scale_ptr + off_s, mask_s)

        off_w_l = tl.arange(0, B_MID_W) + blk * B_MID_W
        off_w = off_j * MID_W + off_w_l[None, :]
        mask_w = (off_j < N0) & (off_w_l[None, :] < MID_W)
        w = tl.load(weight_ptr + off_w, mask_w)
        unpacked_w = extract(w[:, :, None])

        off_o_l = tl.arange(0, B_MID_O) + blk * B_MID_O
        off_o = off_j * MID_O + off_o_l[None, :]
        mask_o = (off_j < N0) & (off_o_l[None, :] < MID_O)
        o = tl.load(offset_ptr + off_o, mask_o)
        unpacked_o = extract(o)

        dequant_w = s[:, :, None] * (unpacked_w - unpacked_o[:, :, None])
        dequant_w = tl.reshape(dequant_w, (dequant_w.shape[0], B_MID))

        off_a_l = tl.arange(0, B_MID) + blk * B_MID
        off_a = off_a_l[:, None] * N1 + off_k
        mask_a = (off_a_l[:, None] < MID) & (off_k < N1)
        a = tl.load(activation_ptr + off_a, mask_a)

        z = z + tl.dot(dequant_w, a)

    tl.store(z_ptr + off_z, z, mask_z)

    return
```
