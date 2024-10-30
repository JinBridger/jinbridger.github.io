---
title: "Python 实现单例模式的误区"
weight: 1
date: 2024-08-09
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Python 实现单例模式的误区

## 遇到的问题

用 Python 实现单例模式并不困难，有一种实现是通过重载类的 `__new__` 方法实现单例模式：

```python
class Student:
    _instance = None

    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __new__(cls, *args, **kwargs):
        if not cls._instance:
            cls._instance = super().__new__(cls)
        return cls._instance


stu1 = Student('jack', 18)
stu2 = Student('jack', 18)
print(stu1 is stu2)
```

上面的代码是我在 Google 上随意找到的[一篇阅读量 11 万的博客](https://www.cnblogs.com/the3times/p/12755968.html)提供的代码. GPT-4 提供的代码也基本类似。这段代码输出为 `True`, 也确实说明了创建的两个对象是在一个地址上。

但是这段代码实际上有一点问题, 如果我们在 `__init__` 中加一句 `print` 语句用来打印执行次数就会发现不太对劲：

```python
...
    def __init__(self, name, age):
        print("Call __init__")
        self.name = name
        self.age = age
...
```

输出变成了:

```
Call __init__
Call __init__
True
```

虽然是单例，但是却调用了两次 `__init__` 方法！

{{< hint info >}}
根据 [Python 官方文档](https://docs.python.org/3/reference/datamodel.html#object.__new__)的描述, 除非 `__new__` 方法不返回实例, 否则会调用 `__init__` 方法进行初始化.

> If `__new__()` is invoked during object construction and it returns an instance of *cls*, then the new instance’s `__init__()` method will be invoked like `__init__(self[, ...])`, where *self* is the new instance and the remaining arguments are the same as were passed to the object constructor.
{{< /hint >}}

## 正确实现单例模式

我在 stackoverflow 上找到了对于[这个问题的解决方案](https://stackoverflow.com/questions/65195333/why-is-singleton-class-init-method-called-twice)

解决方法很简单：**不要使用 `__init__` 方法即可**

```python
class Student:
    _instance = None

    def __init__(self, name, age):
        pass    # 不要使用 __init__ 方法

    def __new__(cls, *args, **kwargs):
        if not cls._instance:
            cls._instance = super().__new__(cls)
            cls._instance.name = name  # 而是在这里进行初始化
            cls._instance.age = age
        return cls._instance


stu1 = Student('jack', 18)
stu2 = Student('jack', 18)
print(stu1 is stu2)
```

## 参考资料

- [Python的6种方式实现单例模式](https://www.cnblogs.com/the3times/p/12755968.html)
- [Python 3.12.4 documentation](https://docs.python.org/3/reference/datamodel.html#object.__new__)
- [Why is `__init__()` always called after `__new__()`?](https://stackoverflow.com/questions/65195333/why-is-singleton-class-init-method-called-twice)
