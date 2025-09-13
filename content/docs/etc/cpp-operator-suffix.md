---
title: "C++ 重载字面量运算符"
weight: 1
date: 2024-09-10
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# C++ 重载字面量运算符

## 背景

在群里看见这样一段代码：

<div align="center">
	<img src="/image/etc/cpp-operator-suffix/a.png" width="80%">
</div>
<div align="center">
	<img src="/image/etc/cpp-operator-suffix/b.png" width="80%">
</div>

里面有一句代码很神奇: `1.0_kmphps`. `0.1_kmphps` 的 `_kmphps` 是一个后缀运算符，
作用是用 0.1 构建一个 `kmphps` 结构体，
这样在 C++ 里实现一个强类型的单位字面量。然后各个单位之间的转换提前定义了构造函数，在代码里以 `static_cast` 体现

查阅了一下，这是 C++11 引入的新特性：[重载字面运算符常量](https://en.cppreference.com/w/cpp/language/user_literal)。

## 用法

类似于 `ull` `f` 这种后缀, 可以实现对字面量的后缀的重载。这种形式的后缀重载可以使用任意标准未规定的后缀，但是不能重载或重定义已经使用的后缀形式，比如前面提到的 `ull` `f` 等后缀。 `operator""` 支持四种格式的重载：

- **整型字面量**
重载字面量运算符时使用 `unsigned long long`、`const char *`、或者模板字面量运算符，比如：`123m`，`1234567890123456789x`。

- **浮点型字面量**
重载字面量运算符时使用 `long double`、`const char *`、或者模板字面量运算符，比如：`10.0s`, `4567.891234567x`。

- **字符串字面量**
重载字面量运算符时使用 `(const char*, size_t)` 参数，比如：`"string"s`， `"Foobar"_path`。

- **字符**
重载字面量运算符时使用 `char`, `wchar_t`, `char16_t`, `char32_t` 参数，比如： `'f'_runic`, `u'BEEF'_w`。

{{< hint warning >}}
C++ 标准规定保留所有非下划线开头的字面量后缀形式，重载字面量运算符时建议使用下划线开头。
如果使用了非下划线开头的字面量运算符重载形式，在 GCC 编译器中也会有警告信息。
{{< /hint >}}

C++11 中提供字面量运算符的重载形式，给字面常量的处理带来很大的便利性和可定制化处理，比如可以在 C++ 中支持任意进制的数据输入、支持大数处理（不用通过先保存为字符串，然后预处理的机制）等。

## 参考资料

- User-defined literals - cppreference https://en.cppreference.com/w/cpp/language/user_literal
- User-Defined-Literal 自定义字面量 https://www.cnblogs.com/tocy/p/cpp11-user-defined-literal.html
