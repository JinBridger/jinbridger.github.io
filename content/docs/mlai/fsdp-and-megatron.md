---
title: "LLM 分布式训练的几种方案"
weight: 1
date: 2026-03-29
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# LLM 分布式训练的几种方案

## 无分布式

在模型不大, 数据不多的时候, 可以只用一张卡进行训练.

```py
W_full = init_model()
O_full = init_optimizer_state(W_full)

for x, y in dataloader:
    # forward
    loss = model_forward(W_full, x, y)

    # backward
    G_full = backward(loss, W_full)

    # 用完整参数、完整梯度、完整优化器状态更新
    W_full, O_full = optimizer_step(W_full, G_full, O_full)
```

但是随着模型与数据变大, 单卡不够了. 因此需要多卡分布式训练.

## 普通的数据并行: DDP

DDP 全称是 Distributed Data Parallel.
它的核心在于复制模型, 切分数据.
每个 GPU 上面都会放置一份完整的模型, 不同 GPU 处理不同的 batch. 使用 All Reduce 来同步梯度.

```py
# 每张 GPU 上都有完整模型参数 W
W_full = init_model()
O_full = init_optimizer_state(W_full)   # 例如 Adam 的 m, v

for x, y in dataloader(rank, world_size):
    # forward
    loss = model_forward(W_full, x, y)

    # backward: 本地先算出完整梯度
    G_full = backward(loss, W_full)

    # DDP: 所有卡对完整梯度做 all-reduce，然后每张卡都拿到完整梯度
    G_full = all_reduce_mean(G_full)

    # 每张卡都用完整参数、完整梯度、完整优化器状态更新
    W_full, O_full = optimizer_step(W_full, G_full, O_full)
```

可以发现, LLM 在前向计算的时候是按层的. 训练时短时间内只会用到一层的参数. 那么能不能将暂时用不到的层进行切分? 这就是参数切分的思想.

## 带参数切分的数据并行: ZeRO 与 FSDP

ZeRO 的全称是 Zero Redundancy Optimizer.
来自于 DeepSpeed.
它将 DDP 的冗余拆掉, 分为 3 个 stage.

PyTorch FSDP 实际上等价于 ZeRO-3.


### ZeRO-1
切分优化器状态.

```py
# 每张 GPU 仍有完整参数
W_full = init_model()

# 但优化器状态只保存自己负责那一片
W_shard = shard(W_full, rank, world_size)
O_shard = init_optimizer_state(W_shard)

for x, y in dataloader(rank, world_size):
    # 1) forward: 直接用完整参数
    loss = model_forward(W_full, x, y)

    # 2) backward: 本地得到完整梯度
    G_full = backward(loss, W_full)

    # 3) 仍像 DDP 一样对完整梯度 all-reduce
    G_full = all_reduce_mean(G_full)

    # 4) 只更新自己负责的参数分片
    G_shard = shard(G_full, rank, world_size)
    W_shard_new, O_shard = optimizer_step(W_shard, G_shard, O_shard)

    # 5) 把各卡更新后的参数分片同步回来, 拼成完整参数
    W_full = all_gather(W_shard_new)
    W_shard = shard(W_full, rank, world_size)
```

### ZeRO-2
在 ZeRO-1 的基础上对梯度进行切分.

```py
W_full = init_model()

# 每张卡只负责一部分参数对应的 optimizer state
W_shard = shard(W_full, rank, world_size)
O_shard = init_optimizer_state(W_shard)

for x, y in dataloader(rank, world_size):
    # 1) forward: 还是直接用完整参数
    loss = model_forward(W_full, x, y)

    # 2) backward: 本地产生完整梯度（概念上）
    G_full_local = backward(loss, W_full)

    # 3) 不再 all-reduce 完整梯度
    #    而是 reduce-scatter，得到“属于自己负责参数分片”的梯度
    G_shard = reduce_scatter_mean(G_full_local)

    # 4) 用自己的梯度分片 + 优化器分片 更新自己的参数分片
    W_shard_new, O_shard = optimizer_step(W_shard, G_shard, O_shard)

    # 5) 把更新后的参数分片 all-gather，重新组装出完整参数
    W_full = all_gather(W_shard_new)
    W_shard = shard(W_full, rank, world_size)
```

### ZeRO-3 (FSDP)
在 ZeRO-2 的基础上对参数进行切分.


```py
# 初始化时，每张卡只保留自己负责的参数分片
W_shards = init_model_shards(rank, world_size)
O_shards = init_optimizer_state(W_shards)

for x, y in dataloader(rank, world_size):
    activations = []
    full_params_cache = []

    # ========= forward =========
    h = x
    for layer_id in model_layers:
        # 1) 临时收集这一层的完整参数
        W_layer_full = all_gather(W_shards[layer_id])

        # 2) 用完整参数做当前层前向
        h = layer_forward(layer_id, W_layer_full, h)

        activations.append(h)
        full_params_cache.append(W_layer_full)

        # 3) 有些实现会尽快释放 full param，或只保留必要信息
        # del W_layer_full
    
    loss = compute_loss(h, y)

    # ========= backward =========
    grad_h = grad_of_loss(loss)

    for layer_id in reversed(model_layers):
        W_layer_full = full_params_cache[layer_id]

        # 4) 用完整参数做该层反传，得到完整梯度
        grad_h, G_layer_full = layer_backward(layer_id, W_layer_full, grad_h)

        # 5) 对该层梯度做 reduce-scatter
        G_layer_shard = reduce_scatter_mean(G_layer_full)

        # 6) 只更新自己负责的参数分片
        W_shards[layer_id], O_shards[layer_id] = optimizer_step(
            W_shards[layer_id],
            G_layer_shard,
            O_shards[layer_id]
        )

        # 7) 释放完整参数/完整梯度临时副本
        del W_layer_full
        del G_layer_full
```


## 模型并行: Megatron-LM

Megatron-LM 是 NVIDIA 提出的一套分布式训练框架. 它实现了模型并行.

> [!INFO]
> **模型并行与参数切分的区别**
>
> 模型并行: 每张 GPU 只计算模型的一部分
> ```
> GPU0: Y0 = X @ W0
> GPU1: Y1 = X @ W1
>
> Y = concat(Y0, Y1)
> ```
>
> 参数切分: 每张 GPU 仍然计算完整模型
> ```
> GPU0 and GPU1:
> W = all_gather(W_shard)
> Y = X @ W
> ```

### Tensor Parallel (TP)

将一个矩阵乘法拆分到多个 GPU 上.

以 Column Parallel 为例:
```
W = [W1 | W2]

GPU0: Y1 = XW1
GPU1: Y2 = XW2

Y = concat(Y1, Y2)
```

### Pipeline Parallel (PP)

把不同层放到不同 GPU 上.

例如 12 层 Transformer:
```
GPU0: layer 1-4
GPU1: layer 5-8
GPU2: layer 9-12
```

计算时用 micro-batch 流水线:
```
Time:   t1        t2        t3
GPU0: Batch1 -> Batch2 -> Batch3
GPU1:           Batch1 -> Batch2
GPU2:                     Batch1
```

## 真实场景下: Megatron-LM + FSDP

真实场景下一般会将 Megatron-LM 与 FSDP 一起用.
- Megatron-LM 负责把一次前向/反向的计算切开 (TP/PP)
- FSDP 负责将同一种模型副本的参数状态切开 (DP)

举个常见的例子. 假设有 64 张卡, 可以这样设置:
- `TP = 4`
- `PP = 4`
- `DP = 4`

总卡数 = TP * PP * DP = 64

这时每张卡属于三个 group：

- 一个 TP group: 每个 Group 存 1/4 Tensor.
- 一个 PP group: 每个 Group 存 1/4 层.
- 一个 DP group: 将参数切成 4 份保存.

其中：

- 在 TP/PP 维度，每张卡只持有自己那部分计算职责
- 在 DP 维度，FSDP 再把这份「本地该负责的模型 shard」的参数状态继续切开

