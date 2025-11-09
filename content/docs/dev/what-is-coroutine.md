---
title: "什么是协程"
weight: 1
date: 2025-11-07
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 什么是协程

## 定义

「协程」是一种通用、强大且易于理解的控制抽象机制. 特点是:
- 函数在执行中可以主动挂起将控制流转移出去, 然后需要的时候再恢复.
- 控制流的转移由用户态进行控制

可以简单的认为: 协程是可暂停执行的函数, 是普通函数的泛化.
所有的普通函数都是不会暂停的协程.

<div align="center">
<img src="/image/dev/what-is-coroutine/coroutine.png" width="100%">
    <br>
    <div style="
    display: inline-block;
    color: #999;
    padding: 2px;">普通函数 vs 协程</div>
</div>

## 对称与非对称

根据控制流转移的方式可以将实现区分为两种: 对称协程与非对称协程.

一种实现是认为每个协程都是平等的, 可以通过一个操作直接将控制流在协程之间进行转移.
这种方式就是对称协程.

另一种实现则是认为协程之间存在调用者与被调用者的关系, 被调用者只能将控制流转移给调用者.
这种方式就是非对称协程.

值得一提的是, 这两种实现实际上是等价的. 非对称协程加一个中央的调度器就可以模拟对称协程.

主流语言基本都选择了非对称协程. 这是因为对称协程需要程序员手动来处理协程之间的调度.
对比之下, 非对称协程由于记录了主从关系, 因此相当于维护了一种隐式的调度器.
控制流比起对称协程更加清晰.

## 有栈与无栈

根据协程的内存模型可以分为有栈协程与无栈协程.

有自己独立的调用栈被称为是有栈协程.
在切换时, 语言内置的调度器会像线程调度一样去保存 / 恢复寄存器, 栈指针等状态.
每个协程都有自己独立的栈空间.
这样做的好处就是可以从任意深度的函数中调用 `yield`.
但缺点也很明显: 实现复杂, 切换成本稍高.

另一种没有自己独立的栈空间的协程被称为是无栈协程.
协程没有自己的栈空间. 而是将本应该存放在栈上的东西放到堆上.
它通过编译器将函数状态转换为一个状态机.
每次 `yield` 时记录当前位置, 下次从这个位置继续.
举个例子, 假设有个协程如下:
```cpp
void fn(){
	int a, b, c;
	a = b + c;
	yield();
	b = c + a;
	yield();
	c = a + b;
}
```
那么转换为状态机就是:
```cpp
struct fn{
    int a, b, c;
    int __state = 0;
    
    void resume(){
        switch(__state) {
        case 0: return fn1();
        case 1: return fn2();
        case 2: return fn3();
        }
    }
    
    void fn1() { a = b + c; __state ++; }
    void fn2() { b = c + a; __state ++; }
    void fn3() { c = a + b; __state ++; }
};
```
可以看到, 本来应该放在栈上面的 `a`, `b`, `c` 都变成了堆上的东西. 我们将保存这些东西的结构称为**协程帧.**

当然, 也可以把协程帧放在栈上, 只要协程的生命周期严格嵌入调用者的生命周期. 比如 Rust 就会分析协程的 Future 是否有逃逸. 如果没有逃逸 (也就是严格的嵌入了调用者的生命周期), 那么编译器就会把协程帧放到栈空间上.

## 协程的实现 (以 Python 为例)

以一个简单的任务为例.

- **输入:** 一个嵌套的 list, 例如 `[1, [[2, 3], [4, 5]], [6, 7, 8]]`
- **任务:** 按顺序打印里面的元素, 例如上面对应的输出就应该是 `1 2 3 4 5 6 7 8`

### greenlet (有栈对称)

Python 的 `greenlet` 提供了一种有栈对称的协程实现.
通过 `glet = greenlet.greenlet(func)` 可以创建一个执行 `func` 的协程 `glet`.
通过使用 `glet.switch()` 可以将控制流移交给 `glet`.

对应的实现如下:
```python
import greenlet

def traverse(x, parent):
    if isinstance(x, list):
        for elem in x:
            traverse(elem, parent)      # recursive call
    else:
        parent.switch(x)                # can switch at any level of stack
        
def greenlet_traverse(x):
    current = greenlet.getcurrent()     # remember the current coroutine
    def _traverse_x():
        traverse(x, current)            # and pass it as the parent
        current.throw(StopIteration)
    return greenlet.greenlet(_traverse_x)

DATA = [1, [[2, 3], [4, 5]], [6, 7, 8]]

glet = greenlet_traverse(DATA)
try:
    while True:
        v = glet.switch()   # switch to the greenlet
        print(v)
except StopIteration:
    pass
```

可以发现有栈对称协程的弊端: 很容易就会被绕迷糊.
所以有栈对称协程并不是主流的实现方案.

### generator (无栈非对称)

Python 的 generator 提供了一种无栈非对称协程的实现.
- 协程通过使用 `yield` 将控制流转移给调用者
- 调用者通过 `next` 将控制流转移给协程

```python
def traverse(x):
    if isinstance(x, list):
        for elem in x:
            new_gen = traverse(elem)    # Create the next level of generator
            try:
                while True:
                    v = next(new_gen)
                    yield v             # Yield everything the inner generator yields
            except StopIteration:       # until iteration stops.
                pass
    else:
        yield x

DATA = [1, [[2, 3], [4, 5]], [6, 7, 8]]

gen = traverse(DATA)    # The top-level generator
try:
    while True:
        v = next(gen)   # Iterate through it
        print(v)
except StopIteration:   # until iteration stops.
    pass
```


## async, await 与协程

协程在面对**非阻塞的** IO bound 的任务时显得尤为有用.
程序可以在等待 IO 的时候将控制流转移出去.
当 IO 有结果的时候再恢复执行.
因此很多语言都基于无栈协程设计了 `async` 与 `await`.

具体而言, 当我们声明一个函数是 `async` 的时候, 就等价于我们声明这个函数是一个协程函数.
对应的, 里面的 `await` 代表了挂起点. 当执行到这个地方的时候, 函数会被挂起, 运行时给这个地方注册一个回调.
等到这个地方的任务完成的时候, 运行时会根据回调来 resume.

以 JavaScript 为例:
```js
const f = async () => {
  console.log("before await");
  await new Promise(resolve => setTimeout(resolve, 1000));
  console.log("after await");
}

f()
```
执行过程：
1. 正常的调用 `f`
2. 打印 `before await`
3. 执行到 `await` 语句
   - 构造一个 Promise
   - 记录等这个 Promise fulfilled 之后要将控制流返回到这里
   - 将 `f` 挂起
4. 一秒后 Promise 变为 fulfilled
5. Runtime 发现 Promise 已经 fulfilled, 于是将控制流返回到这里
6. 打印 `after await`


## goroutine 与协程

之前见到一种说法, 认为 goroutine 也是一种协程.
但严格来讲, 这俩并不是一个概念.

协程是一种基于**单线程控制流**的抽象机制. 

而 goroutine 则是由 Go 运行时管理的轻量级线程. 通过 M:N 将 goroutine 映射到内核线程上.
并且也并非是主动让出, 而是由运行时进行**抢占式**的切换.

## 参考资料

- 协程基础 https://keqing.moe/blog/coroutine-basics/
- Traversing nested lists with coroutines, Rosetta Code style https://wks.github.io/blog/2022/09/22/coroutine-flatten.html
- 协程本质是函数加状态机——零基础深入浅出 C++20 协程 https://www.cnblogs.com/goodcitizen/p/18889661