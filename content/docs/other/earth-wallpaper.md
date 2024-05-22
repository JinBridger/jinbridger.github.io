---
title: "用实时地球图像做壁纸"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 用实时地球图像做壁纸

之前桌面的壁纸用了一年看腻了，打算换个新壁纸。决定用实时的地球影像作为动态壁纸。有人在 Wallpaper Engine 上面做过[这种动态壁纸](https://github.com/qcmiao1998/LiveEarthWeather/)。但考虑到 Wallpaper Engine 会带来一定程度的性能开销，所以决定写个脚本用任务计划的方式实现动态壁纸。

{{< hint info >}}
**优劣**

- ✅ 不会占用太多资源。Wallpaper Engine本质上就相当于开了个浏览器当桌面。
- ✅ 可以方便的自定义。
- ❌ 不能开箱即用。
{{< /hint >}}

具体的代码扔到了[这里](https://gist.github.com/JinBridger/82936502c80130fbe1f59fa33e192d08)。只需要添加到任务计划里面定时每隔 10 分钟执行即可。
<div align="center">
	<img src="/image/other/earth-wallpaper/wallpaper.png" width="100%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    最终的效果。实际上是会随着时间的改变而变化的，这里截取了一张图片作为展示。
    </div>
</div>

## 校正 Himawari-8 的颜色

这个工具采用了向日葵 8 号作为图源。但是向日葵 8 号的图像是明显偏红的，并不太符合真实的地球样子。所以我在脚本里面采用了一点 OpenCV 进行颜色的校正。

具体的过程可以参考这一篇[博客](https://loneskyimages.blogspot.com/2017/05/himawari-8-color-correction.html)。简单来说，就是：


{{< hint info >}}
1. 色阶提高到 1.30
2. 增加 15% 的饱和度
3. 调整通道颜色，具体如下：
   - <div style="color: red; display:inline-block;">红色</div>通道添加 25% 的<div style="color: green; display:inline-block;">绿色</div>通道分量
   - <div style="color: green; display:inline-block;">绿色</div>通道添加 50% 的<div style="color: red; display:inline-block;">红色</div>通道分量
   - <div style="color: blue; display:inline-block;">蓝色</div>通道添加 25% 的<div style="color: red; display:inline-block;">红色</div>通道分量
4. 色阶提高到 1.40
{{< /hint >}}

<div style="display: flex; justify-content: center;">
<div style="width: 50%" align="center">
	<img src="/image/other/earth-wallpaper/himawari-original.png" width="100%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    原图像。
    </div>
</div>

<div style="width: 50%" align="center">
	<img src="/image/other/earth-wallpaper/himawari-correction.png" width="100%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    校正后的图像。
    </div>
</div>
</div>
