---
title: "RM 算法部分代码逻辑"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< hint info >}}
本文为东南大学 3SE 算法组寒假培训第一部分作业。[代码仓库地址](https://gitee.com/liu-zhiqiu/3-se-suanfa)
{{< /hint >}}

# RM 算法部分代码逻辑

## 运动预测

__运动预测主要是由 `Motion_Predict` 类完成。位于 Pose/MotionPredictor.cpp 中。__

__主要调用的是 `predict` 函数，该函数接收当前帧的陀螺仪 pitch 与 yaw 数据以及目标在相机坐标系下的坐标。__

predict 函数在接收到参数之后，首先会判断要预测的装甲是否是已经挂了的机器人的装甲。如 35 行：

```cpp
if(target_armor->color != GREEN && target_armor->is_in_dead_robot != true) {...}
else if(target_armor->color==GREEN && target_armor->is_in_dead_robot==false) {...}
```

下面 else if 的情况是因为击中装甲板可能会识别到暗的那一帧，这时 color 是 GREEN 但是车并不是死的。

若为挂了的机器人，则会直接返回，并且取上一次的 yaw 轴数据覆盖当前的 yaw 轴数据。

否则，会根据下面的公式将相机坐标系下的坐标转换到世界坐标系下。

{{< katex display >}}
\begin{bmatrix}
x_W \\
y_W \\
z_W
\end{bmatrix}
=
R_{3 \times 3}(
\begin{bmatrix}
x_C \\
y_C \\
z_C
\end{bmatrix}
+ T_{3\times 1}
)
{{< /katex >}}

也就是代码中的

```cpp
p_abs = _RVec * (camPoint3D + _TVec);
```

其中，{{< katex >}}T_{3\times 1}{{< /katex >}} 代表平移矩阵 (`_TVec`)，也就是枪管与相机之间的距离。而
{{< katex >}}R_{3\times 3}{{< /katex >}} 为旋转矩阵 (`_RVec`)，
由 pitch 轴与 yaw 轴两个旋转矩阵{{< katex >}}R_{pitch}{{< /katex >}} (`R_pitch`)与{{< katex >}}R_{yaw}{{< /katex >}} (`R_yaw`)相乘得来。矩阵如下：

{{< katex display >}}
R_{pitch} = 
\begin{bmatrix}
1 & 0 & 0 \\
0 & \cos \alpha & -\sin \alpha \\
0 & \sin \alpha & \cos \alpha
\end{bmatrix}
~~~~
R_{yaw} = 
\begin{bmatrix}
\cos \beta & 0 & \sin \beta \\
0 & 1 & 0 \\
-\sin \beta & 0 & \cos \beta
\end{bmatrix}
{{< /katex >}}

也就是代码中的

```cpp
R_yaw << cos(poseEuler_gimbal_yaw/180*CV_PI), 0, sin(poseEuler_gimbal_yaw/180*CV_PI),
         0,                                   1, 0,
		-sin(poseEuler_gimbal_yaw/180*CV_PI), 0, cos(poseEuler_gimbal_yaw/180*CV_PI);
R_pitch << 1, 0,                                     0,
           0, cos(poseEuler_gimbal_pitch/180*CV_PI), -sin(poseEuler_gimbal_pitch/180*CV_PI),
		   0, sin(poseEuler_gimbal_pitch/180*CV_PI), cos(poseEuler_gimbal_pitch/180*CV_PI);
_RVec = R_yaw * R_pitch;
```

在转换坐标系后，会判断一下装甲板与上一次预测的装甲板是否为同一个，如果改变，就清空预测的队列，重新预测，否则会使用之前的数据。

如果转换完的坐标有效（三维不全为 0 ），就会计算目标的各方向上的速度以及总的速度。
也就是这一行：

```cpp
velocity_vector = (p_abs - p_abs_bef) / (timeStamp - last_timestamp);
double v_predict = velocity_vector.norm();
```

`v_predict` 就是目标的总的速度。`velocity_vector` 为目标各方向上的速度。

如果目标的速度大于 0.05 就会开始判断目标移动的方向。(`l56`)

{{< hint warning >}}
__这一部分代码逻辑可能有问题 (`l56-80`)__
{{< /hint >}}

判断方向是由 `direction_index` 完成，如果该值为 2 代表目标向右移动，如果该值为 -2 则代表目标向左移动。

根据刚才算出的目标各个方向上的速度，
- 如果目标在 x 轴方向上速度大于 0，则 `direction_index` 增加 1。
- 如果目标在 x 轴方向上速度小于 0，则 `direction_index` 减小 1。
- `direction_index` 不会超过上下限 2 与 -2。

之后，将各方向上的速度保存到队列中。

如果当前时间戳与上一个时间戳间隔达到了 200ms，就会触发预测。预测的流程如下：

- 取出之前保存在队列的速度。(`l101`)
- 过滤掉与当前方向相反的速度。(`l107`)
- 求出速度的平均值。(`l113`) 
- 更新时间戳。(`l119`)

{{< hint warning >}}
__这一部分代码逻辑可能有问题 (`l101-121`)__
{{< /hint >}}

之后将预测出来的速度进行低通滤波：

```cpp
p_abs_lp(0) = Fx.LpFilter(p_abs(0));
p_abs_lp(1) = Fy.LpFilter(p_abs(1));
p_abs_lp(2) = Fz.LpFilter(p_abs(2));
```

用子弹滞空时间 `time_to_fly_delay` +通信延迟 `time_to_stm` 计算出子弹打击到目标需要的时间 `time_predict` ，并用这个时间乘以计算出来的速度得到世界坐标系下目标的预测坐标 `p_abs_pre`。也就是：

```cpp
p_abs_pre = p_abs_lp + vel_mean * time_predict * 1000;
```

如果目标预测坐标在 z 轴上与本体距离不为 0，就进行如下步骤：

- 反推相机坐标系下的目标预测坐标。(`l135`)
- 绘制预测点。(`l136`)
- 计算为了将枪口指向预测点的偏转角度 `yaw_pre` 与 `pitch_pre`。(`l137-138`)
- 记录刚才计算的偏转角度。(`l139-145`)

最后更新时间戳与之前的预测坐标。(`l148-149`)

## 反陀螺

反陀螺中最简单的便是敌方车辆原地旋转的时候，此时的击打策略为：

- 击打敌方车辆中心点。原因在于由于车辆的原地小陀螺确实速度会很快。甚至超过了识别和子弹滞空时间，所以没办法进行预测，就算预测也效率不高。
- 取敌方车辆中心点的方法为：取每一个周期的平均值即可。

在 Main/test_infantry.cpp 中可以看到这部分逻辑，对于低速或者开陀螺的装甲车，是不会使用目标预测的，转而打击目标的中心点。(`l439`)

__检测机器人是否旋转主要使用 `SpinningChecker` 类的 `checkSpinning` 函数实现。位于 Armor/SpinningChecker.cpp 中。__

`checkSpinning` 函数接收一个 `Robot` 对象并判断该对象是否在旋转。

函数首先会判断 `spinning_trend` 数组内数据数量是否足够判断，如果数据不够，则会直接判断为不旋转并返回。(`l33`)

这里先简要看下 `spinning_trend` 数组的作用。这个数组存储着三元组 {{< katex >}}(a, b, c){{< /katex >}}, 其中 
{{< katex display >}}
a=\frac{x}{X}
,~~~
b=\frac{y}{Y}
{{< /katex >}}

{{< katex >}}c{{< /katex >}}的意义暂且略过不提，如下图：

<div align="center">
	<img src="/image/other/rm-code-analyze/relcoord.png" width="50%">
</div>

也就是说，这个三元组存储了 `robot.main_armor` 在 `robot.rect` 的相对坐标。这也是 `getRelativeCoord()` 所做的事情，如下代码所示：(`l12`)

```cpp
c1.x = robot.main_armor->center.x - robot.rect.tl().x;
c1.y = robot.main_armor->center.y - robot.rect.tl().y;
c1.x = c1.x / robot.rect.width;
c1.y = c1.y / robot.rect.height;
```

回到刚才的正题，如果 `spinning_trend` 数组内数据数量足够，就会开始计算每两帧之间相对坐标的变化值 `delta_RelativePos_x` 与 `delta_RelativePos_y`. (`l45-46`)

之后会计算装甲板旋转的速度。这里首先要取相对靠中间的装甲板计算速度 (`l51`), 将所有的差值求平均数即为最终的装甲板运动速度 `armor_relative_speed` 并将其保存起来 。(`l65, l72`)

根据最后两帧装甲板移动的速度，可更新装甲板旋转程度 `spinning_score`。(`l79-88`)

{{< hint warning >}}
__这一部分代码逻辑可能有问题 (`l69-72`)__
{{< /hint >}}

- 如果速度大于阈值，则旋转程度 `spinning_score` +1
- 如果速度大于阈值，则旋转程度 `spinning_score` -1
- 旋转程度 `spinning_score` 不会超过 0 和 15

如果旋转程度大于 5 则判定为在旋转。更新 `is_spinning` 并返回。(`l95`)

## 打能量机关

__打能量机关（打符）主要由 `Windmill` 类实现。位于 Windmill/Windmill.cpp 内。__

整个打符的流程大致如下：

1. 串口获得当前任务为打符，进入打符流程。(`Main/test_infantry.cpp:250`)
2. 加载当前帧的图像以及时间戳。(`Main/test_infantry.cpp:258`)
3. __检测能量机关。__(`Main/test_infantry.cpp:265`)
4. __进行重力补偿，计算角度数据并回传。__(`Main/test_infantry.cpp:270-316`)

核心步骤为 3 与 4.

### 检测能量机关

__检测能量机关主要由 `Windmill` 类的 `Detect()` 函数实现。位于 Windmill/Windmill.cpp 内。__

1. 首先，分离三通道的图像 (`l260-267`)，并根据设定的风车颜色使用阈值进行二值化。(`l268-279`)
2. 合并二值化图像，取图像交集。(`l313-339`)
3. 对图像进行膨胀处理。(`l349-353`)
4. 对膨胀处理后的图像寻找轮廓。将要打击的装甲板的颜色要比已经打击的装甲板更亮，所以经过滤波之后已经打击过的装甲板会有一部分成分的残缺，这不用管，可以还会对后面的识别有所帮助。(`l363`)
5. 进行扇叶的识别 (`Fan_recognition()` 函数)
	- 根据轮廓大小进行过滤，剔除过大/过小的扇叶。(`l818`)
	- 如果大小符合要求，进行匹配，将匹配度保存到 `comres_swatter` 数组中。(`l824`)
	- 选出匹配度最高的扇叶，如果该扇叶匹配度仍不达标则视为没有找到扇叶并返回。(`l855`)
	- 否则，计算扇叶的质心 `_fanCenter`。(`l887`)
	- 之后寻找距离质心最远的点 `newDirection`，该点由距离质心最远的点与相邻两点取平均数得到。(`l893-916`)
	- 将 `newDirection` 与 `_fanCenter` 加上偏移值，得到在原图中的坐标。标记找到了扇叶并返回(`l918-921`)
6. 识别风扇中间的 R 标志 (`R_recognition()` 函数)
	- 大体步骤与扇叶识别一致，最终返回 R 标志中心坐标 `_center`。(`l978-979`)
7. 调用 `returnNextPoint()` 计算打击预测点。
	- 调用 `forecastNextPoint()` 计算打击预测点
		- 为了获知装置旋转的方向，这个函数在前十次调用不会返回预测点，而是通过这前十次计算调用计算装置旋转的方向。计数器为 `_framecounter`。
		- 每次调用会将当前由扇叶指向中心的向量与上一次由扇叶指向中心叉乘，根据正负号判断方向。如果为负，则 `for_clock` 增加。(`l457-465`)
		- 前十次调用结束后，如果 `for_clock` 大于 5 则为顺时针方向。(`l602`)
		- 小大符思路基本一致，大符可能判断次数多一些。
		- 根据弹速计算出预测点与当前点偏移的圆心角 `forecast_angle`。(`l600`)
		- 根据偏移的圆心角计算出预测点 `forecastPointPlus` 并返回。(`l694-718`)
	- 检查刚才预测的是否合理。
		- 计算扇叶重心与预测点距离，并对其进行过滤，过大或过小均视为未找到并返回。(`l751`)
		- 计算扇叶最远点与装置中心点距离，并对其进行过滤，过大或过小均视为未找到并返回。(`l757`)
		- 计算扇叶重心与装置中心点距离，并对其进行过滤，过大或过小均视为未找到并返回。(`l764`)
	- 如果合理，返回刚才预测的打击点。
8. 调用 `Whether_To_Hit()` 判断是否击打目标。如果轮廓大小合适则予以打击。如果五个扇叶全亮则停止打击。

### 进行重力补偿

__调用 `gravityOffset()` 函数实现，参见群文件 技术文档_位姿解算.pdf__

最后将补偿完的角度加到 pitch 轴上 (`l292`) 并返回控制数据。