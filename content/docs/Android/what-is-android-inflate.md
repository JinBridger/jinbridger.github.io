---
title: "Android inflate 的简单分析"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Android inflate 的简单分析

最近学了一些Android开发，但是对于inflate这一概念还是云里雾里不甚理解，每次也都是靠着Android Studio的补全来写。今天就仔细研究一下inflate的原理。

## 什么是 inflate

一般而言，inflate的作用是将一个xml对象转换为View对象。在这之前，我想简单的谈一下View是什么。

我们都知道例如Button, TextView等控件，而View是这些控件的共同基类。它是一种抽象的存在，意为这些控件，同样的还有ViewGroup，也就是控件组，表示一组控件。

Android 程序运行时，是通过获取View来展示对应的画面的，而我们一般的都是写xml文件，因此需要有一个将xml转换为View的工具，也就是inflate.

## 常见的 inflate

举个最简单的例子，一般的，在 Android 开发中，如果启用viewBinding的话，我们刚开始会从下面的代码开始：

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        val view = binding.root
        setContentView(view)
    }
}
```

那么，什么是 LayoutInflater? inflate方法又是干嘛的？

## LayoutInflater 是什么

LayoutInflater是一个用于将xml布局文件加载为View或者ViewGroup对象的工具，我们可以称之为布局加载器。

它的作用就是接收一个xml文件，并将对应的xml布局转换为View对象。

LayoutInflater本身是一个抽象类，不能使用常规的方法获取。对于每个Activity，都有对应自己的LayoutInflater.一般的，获取LayoutInflater的方法是调用Activity的get方法获取。

一般的，我们都是调用它的inflate方法加载xml布局。

## inflate 方法是什么

通过查阅源码，可以发现inflate方法有以下四个重载：

```java
inflate(int resource, ViewGroup root)
inflate(int resource, ViewGroup root, boolean attachToRoot)
inflate(XmlPullParser parser, ViewGroup root)
inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot)
```

前两种调用会分别调用后两种重载。系统会将resourceId解析为对应的xml。

通过查阅资料得知，如果：

- __`root != null, attachToRoot == true`__：传进来的布局会被加载成为一个View并作为子View添加到root中，最终返回root；而且这个布局根节点的android:layout_xxx参数会被解析用来设置View的大小。

- __`root != null， attachToRoot == false`__：传进来的布局会被加载成为一个View并直接返回。布局根View的android:layout_xxx属性会被解析成LayoutParams并保留。(root只用来参与生成布局根View的LayoutParams)

- __`root == null, attachToRoot` 未传__：当root为空时，attachToRoot是什么都没有意义，此时传进来的布局会被加载成为一个View并直接返回；布局根View的android:layout_xxx属性会被忽略。

也就是说，下面两种写法是等价的：

```kotlin
layoutInflater.inflate(R.layout.test_textview, binding.textLayout, true)
```

```kotlin
val newView = layoutInflater.inflate(R.layout.test_textview, binding.textLayout, false)
binding.textLayout.addView(newView)
```

## 一个简单的例子

下面的例子通过调用inflate实现更换RelativeLayout内容：

__MainActivity.kt:__
```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        val view = binding.root
        setContentView(view)

        binding.loadButton.setOnClickListener {
            loadView()
        }
        binding.removeButton.setOnClickListener {
            removeView()
        }
    }

    private fun loadView() {
        val newView = layoutInflater.inflate(R.layout.test_textview, binding.textLayout, false)
        binding.textLayout.addView(newView)

        // also can written in
        // layoutInflater.inflate(R.layout.test_textview, binding.textLayout, true)
    }

    private fun removeView() {
        binding.textLayout.removeView(findViewById(R.id.test_textview))
    }
}
```

__test_textview.xml:__
```xml
<?xml version="1.0" encoding="utf-8"?>
<TextView xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/test_textview"
    android:background="#00ff00"
    android:text="This is test textview."
    android:layout_width="match_parent"
    android:layout_height="match_parent">

</TextView>
```

__activity_main.xml:__
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >

    <Button
        android:id="@+id/loadButton"
        android:text="load"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

    <Button
        android:id="@+id/removeButton"
        android:text="remove"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

    <RelativeLayout
        android:id="@+id/textLayout"
        android:layout_width="300dp"
        android:layout_height="300dp" />

</LinearLayout>
```
