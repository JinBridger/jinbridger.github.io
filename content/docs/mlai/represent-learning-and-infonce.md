---
title: "表征学习与 InfoNCE"
weight: 1
date: 2025-09-07
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# 表征学习与 InfoNCE

学习 Representation Learning with Contrastive Predictive Coding 的笔记.

## 表征学习

表征学习是机器学习中的一个重要概念, 核心目标是让模型自动学习出数据的有效表示, 而不是完全依赖人工设计特征. 比如训练一个模型, 输入为数据 (文本, 图像等), 输出为特征向量 (例如 embedding).

表征学习可以使用自监督的方式进行训练. 自监督分为两种方式, 一种是 BERT 的掩码预测, 通过扣掉一部分文本让模型来预测从而学习到表征, 训练完成之后丢掉最后的输出层就得到了表征模型. 另一种方式就是对比学习, 同样的, 对比学习也是通过预测进行训练, 但是区别是通过提供正负样本来学习表征. 用一个形象的说法就是, 掩码预测相当于做填空题, 而对比学习相当于做选择题.

如果下游任务做的是分类/检索/聚类型的任务, 那一般对比学习训练的效果会更好. 但如果是生成/理解型的任务, 那么一般掩码预测会更好.


<div align="center">
	<img id="contrastive_repr_auto_svg" src="/image/mlai/represent-learning-and-infonce/contrastive_repr.svg" width="100%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    论文的图 1
    </div>
</div>

这篇论文里面提到的方法就是对比学习. 论文的图 1 展示了整体的框架. 通过训练一个编码器 $g_\text{enc}$ 来将输入进行编码, 然后用一个自回归模型 $g_\text{ar}$ 处理编码后的输入. 图里面的 $g_\text{ar}$ 用的是 GRU, 接收前一个编码出来的隐状态 (图里面的水平箭头) 与当前的 $z_t$ 作为输入, 输出表征 $c_t$.

通过训练预测任务, 最终得到的 $g_\text{ar}$ 加上 $g_\text{enc}$ 就是表征模型. 推理的时候将要提取表征的对象输入进去, 输出的 $c$ 即为提取的表征.

## 绕过互信息

论文的出发点是互信息. 互信息的推导过程可以参考[信息论入门笔记]({{< ref "/docs/mlai/information-theory-basic.md" >}})

论文中提到的互信息公式为
$$I(x;c)=\sum_{x, c} p(x, c)\log\frac{p(x|c)}{p(x)}$$

其中 $x$ 是要预测的位置的值. $c$ 为之前的输入生成的表征. 为了尽可能提高学习表征的效果, 我们应该尽可能的最大化要预测的位置的 ground truth 与我们之前的输入生成的表征之间的互信息.

但是这个是基本没办法计算的. 因为我们不知道 $x$ 与 $c$ 的联合概率分布 $p(x, c)$. 因此需要找个办法绕过去. 

我们观察互信息的公式, 发现有这样一项
$$\log\frac{p(x|c)}{p(x)}$$

这一项可以写成
$$\log\frac{p(x|c)}{p(x)}=-\log p(x) - (-\log p(x|c))=I(p(x)) - I(p(x|c))$$

也就是用 $p(x)$ 的自信息减去 $p(x|c)$ 的自信息. 不难想到, 如果两个东西的相似度很高, 那么它俩的自信息重合度一定很高, 那么这个值算出来一定也很高. 也就是说, **两个东西的相似度跟互信息中的这一项是成正比的.** 互信息没办法计算, 但是相似度是可以建模计算的. 因此就用一个打分函数 $f$ 来拟合这个东西. 也就是论文中的
$$f_k(x_{t_k}, c_t)\propto \frac{p(x_{t+k}|c_t)}{p(x_{t+k})}$$


互信息是计算这个东西的期望, 那我们就不妨直接把这玩意当成是均匀分布的 (这样做会导致算出来的值比真实的互信息要小, 因为真实的分布一定不是均匀的), 从而得到一个下界. 如果当成均匀分布的计算的值都很大, 那么真实值一定会更大. 因此优化这个下界就是优化互信息. 这个下界就是 InfoNCE, 论文中的公式为:
$$\mathcal L_\text N = -\mathbb E_X \log \frac{f_k(x_{t+k}, c_t)}{\sum_{x_j \in X} f_k(x_j, c_t)}$$

其中 $X=\\{x_1, x_2, ..., x_N\\}$ 是我们构造的样本, 其中只有一个样本是真的, 也就是 $x_{t+k}$, 剩下的都是随机抽的噪声. 这也就是 InfoNCE 的名字的来源: Info 代表它的出发点, 也就是信息论. NCE 代表它借鉴了 NCE 的用噪声样本来构造对比的思路.

> [!NOTE]
> 但实际应用中一般用这个公式:
> 
> $$\mathcal L_\text N = -\log \frac{f_k(x_{t+k}, c_t)}{\sum_{x_j \in X} f_k(x_j, c_t)}$$
> 这是因为从 $X$ 中随机抽取是均匀分布, 直接求和的结果跟期望就差除以一个数量, 为了节省计算就直接省掉了除以数量这一步.

## 应用场景

论文里面提到了一些对比学习的实验.

### 语音

任务是对说话人进行分类.

方法是在 TTS 数据集上训练表征模型, 之后在表征模型上在基于分类数据集单独训练一个分类器.

### 视觉任务

<div align="center">
	<img id="contrastive_vision_auto_svg"  src="/image/mlai/represent-learning-and-infonce/contrastive_vision.svg" width="100%">
    <br>
    <div style="display: inline-block; color: #999; padding: 2px;">
    对比学习应用在视觉任务
    </div>
</div>

任务是对图片进行分类.

方法是先在 ImageNet 上做无监督学习, 将每张图片裁切到 256 * 256 再切成 7 * 7 的 patch. 用 ResNet 作为 $g_\text{enc}$ 对原图像的每个 patch 编码为 1024 维的表征, 用自回归模型预测后面 patch 的表征. 值得注意的是这里自回归模型的输入是：前几行 patch 的表征, 输出是该 patch 下面的 patch 的表征.

之后在 7 * 7 * 1024 的表征上做均值池化, 作为整体的表征. 再单独训练一个分类器作分类.

### NLP

任务是对句子进行分类, 例如评价的好坏, 意见的主观与客观等.

方法是在文本数据集上训练表征模型. 之后在表征模型上在基于分类数据集单独训练一个分类器.

