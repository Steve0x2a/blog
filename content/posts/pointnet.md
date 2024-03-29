---
title: "PointNet系列论文阅读与理解"
date: 2021-10-04T20:29:20+08:00
draft: false
toc: true
---

**PointNet**是斯坦福大学研究人员提出的一种点云处理网络，其可以直接输入无序点云集合进行处理，而不像基于投影的方法需要先对点云进行预处理再输入网络。其可以用作与点云分类和点云分割。由于其可以直接输入无序点云，因此对深度学习点云处理产生了巨大的影响。而同一个作者的进阶版网络**PointNet++** 则解决了PointNet局部特征使用不足的问题，提高了其局部特征的处理能力。

## 点云的特征

点云往往是欧几里德空间点的子集，往往拥有以下三个主要特征：

1. 无序性

   与图像的像素矩阵或体积栅格中的体素阵列不同，点云是一组没有特定顺序的点，也就是说，$N$个3D点集会有$N!$种排列。

2. 点之间相互作用

   这些点来自具有距离度量的空间，这就意味着点与点之间不是鼓励的，相邻点可能形成一个有意义的子集。因此，模型应该捕获局部结构以及局部结构互相之间的特征。

3. 变换不变性

   对点云全集进行刚性变换，点之间的相对位置是不变的。



## PointNet

PointNet的整体架构如下图所示：

![PointNet结构](https://0x2a-1254351943.cos.ap-guangzhou.myqcloud.com/img/image-20211004210924008.png)



下面将分解PointNet的每一部分。

### MLP

共享多层感知机在PointNet的max pool前用了两次，其为共享MLP，第一次是在对原始点云transform后的点将特征升到64维，第二次是在对64维特征transform后再次升维到1024维度，本质上就是使用MLP对输入的点进行两次特征升维提取。

在进行max pool后，根据网络功能的不同，使用的MLP次数也不同。对于分类网络，对池化后的特征进行MLP，最终输出分类分数矩阵。而对于分割网络，通过了两次shared MLP，最后输出一个$n*m$的分数矩阵。



### Max Pool

Max Pool 可以说是PointNet的核心思想了，但细究原理其实很简单。由于PointNet 是直接将点云集合输入到网络中，因此要求点云输入顺序不同，提取出来的特征也是不能改变的，即需要找到一个函数，可以使得输入的顺序改变，结果也不改变。自然而然的可以找到一些可选选项：**取最大值**、取平均值，取和等方法。而PointNet使用的就是取最大值这个方法，也做了相关实验，这些方法都是可行的，并且取最大值这个方法取得了最好的效果。Max Pool方法示意图如下：

![MaxPool原理](https://0x2a-1254351943.cos.ap-guangzhou.myqcloud.com/img/image-20211004213709574.png)

在进行特征升维后，我们分别取各个维度中每个点最大的一个值，将他们组成最后的特征矩阵，就是Max Pool的做法。易得，这种方法不管输入点的顺序是什么，得到的特征矩阵都是一致的。



### T-Net

T-Net用在input transform和feature transform中，其本质也是一个微型的PointNet，作用是将原始点矩阵或特征矩阵转换为标准的朝向，这样的话就可以适应不同朝向的点云了，对应点云特征第三点。

但实际作用并不是太大，可以略微改善效果，聊胜于无。但其也添加了相应的计算量。因此在PointNet++中就完全删除了T-Net。



### Segmentation Network

PointNet用作于分割相对于用于分类稍复杂一些，其结构如下：

![分割网络结构](https://0x2a-1254351943.cos.ap-guangzhou.myqcloud.com/img/image-20211004214341179.png)

由于分割任务需要把每个点都考虑进去，因此最后用于MLP的矩阵不能只含有全局特征（global feature），还需要包含每个点的特征。最左边的矩阵白色部分是经过特征第一次升维和transform的矩阵，右边灰色部分是将全局特征重复n次，将这两个部分拼接到一起，最后再经过两层MLP，最后得到每一个点的分类，以完成分割任务。



以上三个部分是作者认为的PointNet核心的三个思想。下面来谈一下我觉得PointNet比较有意思的点。



### Critical Point Sets

在Max Pool中我们提到了其作用是提取各个点的每个维度中的最大值，如果只保留对最后全局特征有起到作用的点，将他们集合起来，这就称之为Critical Point Sets。

![image-20211004215456290](https://0x2a-1254351943.cos.ap-guangzhou.myqcloud.com/img/image-20211004215456290.png)

可以看到，Critical Point Sets往往是点云的边界，这些点往往定义了点云的形状。网络的鲁棒性就来源于此处，只要保留了关键点，及时点云有变换和扰动结果也不会受到影响。

![image-20211004215229468](https://0x2a-1254351943.cos.ap-guangzhou.myqcloud.com/img/image-20211004215229468.png)





### Voxel Feature Encoding 

目前来说有一种比较好用的特征提取方法，如图所示：

![image-20211004220027494](https://0x2a-1254351943.cos.ap-guangzhou.myqcloud.com/img/image-20211004220027494.png)

其先进行一次PointNet特征提取，然后将获得的全局特征拼接到原有的特征上，在进行一次PointNet特征提取，最后获得一个新的全局特征，这样的取样方式会获得兼具局部和全局的特征，PointNet++也是用了类似的思想进行特征提取。



### 总结

PointNet的创新点如下：

- 直接用点数据进行操作，不损失精度
- 对无序数据的效果比较好
- 证明了PointNet也可以和正常神经网络一样，可以拟合任何函数

同时，局限性如下：

缺陷：

- 缺少了逐层特征提取
- 仅仅是对每个点表征，对局部结构信息整合能力太弱 
- 分割任务的全局特征global feature是直接复制与local feature拼接



## PointNet++

PointNet的一个比较突出的不足就是无法获取局部特征，因为在PointNet中，只存在对点进行1*1的卷积操作，或者全局点进行最大池化操作，因此获得的特征严重损失了大量珍贵的局部特征。这就导致了PointNet在分割，特别是部分分割场景下的效果并不好。

因此，PointNet的作者借鉴了CNN的思想，通过先把点云划分为具有重叠部分的局部区域，再局部区域中使用PointNet提取局部特征，再扩大范围，再局部特征的范围上再次提取更高层次的特征，直到最后提取点云的全局特征。基于这种思想，提出了PointNet++。PointNet++的整体结构如下：

![PointNet++结构](https://0x2a-1254351943.cos.ap-guangzhou.myqcloud.com/img/image-20211005094052197.png)

整个网络可以看作为encoder-decoder的结构，encoder是一个采样的过程，通过多层set abstraction将数据进行局部到全局的采样。decoder分为分类和分割，分类decoder比较简单，将encoder获得的全局特征经过一个PointNet降维，再通过多层感知机获得分类分数。分割decoder相对较复杂，需要通过反向插值的方法实现采样回复点云，再通过一个unit PointNet获得每个点的分类。



### Set abstraction

每个set abstraction包括sampling、grouping和一个PointNet。

- sampling：

  使用farthest point sampling（FPS）选择$N$个点，作者使用FPS而不是随机取样的原因是，FPS更容易囊括整个点云。

- grouping：

  使用Ball Query或KNN的方法，生成在sampling的邻域中选择$K$个点。这一步使用具体哪个方法差别不太大：![bq and knn](https://0x2a-1254351943.cos.ap-guangzhou.myqcloud.com/img/20210118212956787.png)

- PointNet：

  对Sampling+Grouping以后的点云进行**局部的全局特征**提取

整体的流程如图所示：

![set abstraction](https://0x2a-1254351943.cos.ap-guangzhou.myqcloud.com/img/image-20211005095828299.png)

经过多个set abstraction后，可以获得一个兼具局部和全局的特征。

![encouder层](https://0x2a-1254351943.cos.ap-guangzhou.myqcloud.com/img/image-20211005095908192.png)

### MSG&MRG

通过多层的特征提取，可以使得点云分类和分割性能都有所提升，但是也损失了其鲁棒性。激光所收集的点云中，总是有近处密度高，远处密度低的特性，而PointNet对于不均匀点云的处理性能不佳。

在论文中，作者给出了对比试验，在密度不均匀的点云中使用原始PointNet++，其效果甚至不如PointNet。PointNet++提出了两个解决方案：多尺度分组(MSG)和多分辨率分组(MRG)。

![image-20211005101801592](https://0x2a-1254351943.cos.ap-guangzhou.myqcloud.com/img/image-20211005101801592.png)

- MSG

  MSG在每一个分组层都使用多个尺度（半径）来确定领域范围，每一个范围经过PointNet特征提取后再整合起来得到一个多尺度的新特征。

- MRG

  MRG的每一个特征都由两部分组成：本层领域范围经过PointNet获得的的特征，以及上一层领域范围经过PointNet获得的特征。当点云密度不均时，可以通过判断当前patch的点云密度给予左右两个特征向量不同的权重。例如，当patch中密度过小，左边特征向量中包含的点更稀疏，容易受到抽样不足的影响，因此提高右边特征向量的权重。

MSG效果较好，但相对的计算量也较大（经过三次PointNet）。作者在论文中给出了分类实验结果对比图，可以看出多尺度(MSG, MRG)和单一尺度(SSG)相比分类准确率没有什么提升，但当点云很稀疏的时候，使用MSG可以保持很好的鲁棒性。random input dropout(DP)对于鲁棒性提升也很大。

![实验对比](https://0x2a-1254351943.cos.ap-guangzhou.myqcloud.com/img/20210119152938826.png)

random input dropout(DP)指的是输入的时候随机选择原输入点的一部分进行最终输入，也就是随机抛弃掉一些点，在原文中，保留了95%的点。



### Classification

分类网络比较简单，用Encoder层获得的全局特征经过一次PointNet在提取一次全局特征，然后通过全连接网络就可以获得分类结果了，分类网络的结构图如下：

![Classification](https://0x2a-1254351943.cos.ap-guangzhou.myqcloud.com/img/image-20211005101355519.png)

### Segmentation

经过Encoder部分，我们获得的是一个全局的特征，而如果我们需要做分割，需要的是逐点特征（point-wise feature）。PointNet的解决方法很简单除暴，将全局特征copy了和local feature进行拼接，这样获得的特征具有一定的邻域信息，但是辨别性（discriminative）并不好。

而PointNet++使用反向插值+skip connection的方法获得一个兼具全局和局部的特征。分割网络的结构图如下：

![image-20211005103222387](https://0x2a-1254351943.cos.ap-guangzhou.myqcloud.com/img/image-20211005103222387.png)

我们以红框那一层为例，红框内蓝色矩阵的大小为$(N_1,d+C_2+C_1)$，其中$(N_1,d+C_1)$部分是直接使用encoder部分蓝色矩阵，这一步叫做skip connection，那还有$(N_1,C_2)$的部分怎么获得呢，这就要用到反向插值的方法了。

红框内的橘色矩阵是encoder部分获得的最终特征，大小为$(N_1,d+C_2)$，将skip connection获得的每一个点都投影到橘色矩阵中，再使用KNN的方法获得离他最近的$N$个点，作加权平均，其中这个权重是与x距离成反向相关的，意思就是距离越远的点，对x特征的贡献程度越小。用这个方法就可以获得$C_2$大小的特征，具体公式如下：
$$
f^{(j)}(x)=\frac{\sum_{i=1}^{k} w_{i}(x) f_{i}^{(j)}}{\sum_{i=1}^{k} w_{i}(x)} \quad \text { where } \quad w_{i}(x)=\frac{1}{d\left(x, x_{i}\right)^{p}}, j=1, \ldots, C
$$
依次进行采样后，恢复每一个点特征，也就是decoder。



### 总结

PointNet++是PointNet的延续，一定程度上弥补了PointNet的一些缺陷。PointNet++提供了比较好的表征网络，后序的点云处理发展很多论文都是用到了这种表征方式。不过PointNet++相对于PointNet不管是分类还是分割任务，总体的准确率大概只提升了2-4个点。

## 参考资料
- [PointNet论文](https://arxiv.org/abs/1612.00593)
- [PointNet++论文](https://arxiv.org/abs/1706.02413)
- [PointNet作者演讲](https://www.bilibili.com/video/BV1As411377S)
- [三维点云网络——PointNet论文解读](https://blog.csdn.net/u014636245/article/details/82755966)
- [PointNet++论文解读以及代码分析](https://blog.csdn.net/weixin_41317766/article/details/112770272)
- [搞懂PointNet++，这篇文章就够了！](https://zhuanlan.zhihu.com/p/266324173)