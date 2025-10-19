---
title: "动手实现自动微分"
weight: 1
date: 2025-10-01
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 动手实现自动微分

学习了一下自动微分. 参考 [karpathy/micrograd](https://github.com/karpathy/micrograd)

## 原理

自动微分的原理是对算子构建一个 DAG.
计算微分的时候从要微分的点出发, 沿着边进行传播.

<div align="center">
	<img id="auto-grad_auto_svg" src="/image/mlai/implement-grad/auto-grad.svg" width="40%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    $c=a\times b$ 对应的 DAG. 反向传播时, 设置 $c$ 的梯度为 1, 然后沿着边进行传递.
    </div>
</div>

## 实现

### 定义 DAG

DAG 的节点包括:
- `data`: 节点前向传播的值
- `grad`: 节点反向传播的梯度
- `_backward`: 节点反向传播的时候调用的函数, 根据节点的运算符来决定
- `_prev`: 节点指向的其他节点
- `_op`: 节点的运算符

```python
class Value:
    def __init__(self, data, _children=(), _op='') -> None:
        self.data = data
        self.grad = 0
        self._backward = lambda: None
        self._prev = set(_children)
        self._op = _op
```

### 运算

以加法为例, 当调用 `c=a+b` 的时候, 会创建一个新的节点 `c`, 并设置其反向传播函数

```python
def __add__(self, other):
    other = other if isinstance(other, Value) else Value(other)
    out = Value(self.data + other.data, (self, other), '+')

    def _backward():
        self.grad += out.grad
        other.grad += out.grad
    out._backward = _backward

    return out
```

其他运算类似.

### 反向传播

反向传播就是从要计算的节点出发, 构建拓扑排序. 之后对其中的节点逐个调用反向传播.

```python
def backward(self):
    topo = []
    visited = set()
    def build_topo(v):
        if v not in visited:
            visited.add(v)
            for child in v._prev:
                build_topo(child)
            topo.append(v)
    build_topo(self)

    self.grad = 1
    for v in reversed(topo):
        v._backward()
```

## 测试

利用 [karpathy/micrograd](https://github.com/karpathy/micrograd) 提供的代码进行测试.

```python
a = Value(-4.0)
b = Value(2.0)
c = a + b
d = a * b + b**3
c += c + 1
c += 1 + c + (-a)
d += d * 2 + (b + a).relu()
d += 3 * d + (b - a).relu()
e = c - d
f = e**2
g = f / 2.0
g += 10.0 / f
print(f'{g.data:.4f}') # prints 24.7041, the outcome of this forward pass
g.backward()
print(f'{a.grad:.4f}') # prints 138.8338, i.e. the numerical value of dg/da
print(f'{b.grad:.4f}') # prints 645.5773, i.e. the numerical value of dg/db
```

## 静态图 or 动态图

值得一提的是 micrograd 是动态图.
因为它是在执行的过程中建图.

而静态图则是先定义一张计算蓝图（只描述操作之间的依赖），
然后在运行阶段输入数据, 执行整张图并输出结果.

对于动态图来说, 每次执行算子时：
- Python 立即执行计算
- 只知道当前算子及其输入输出
- 反向图在运行时动态生成
- 无法在执行前知道整个模型的拓扑和数据流
所以编译器无法提前全局分析依赖, 内存生命周期, 算子融合可能性等.


而对于静态图:
- 节点（算子）、边（张量）都已知
- 控制流、数据流、依赖关系清晰
- 图固定，可以在执行前全局优化、重排、编译成高效内核
编译器能像优化 C/C++ 程序那样，把图整体“编译”成一个高效的执行计划

因此静态图可以做一些优化, 比如算子融合, 内存复用, 常量折叠等.

后来动态图也出现了从动态图编译为静态图的机制, 例如 `torch.compile`.
捕获一次动态执行（trace 出计算图）→ 优化编译成静态图 → 后续直接运行高效内核
