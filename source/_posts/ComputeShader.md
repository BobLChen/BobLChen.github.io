---
title: ComputeShader调度
date: 2020-08-22 09:41:13
tags:
- Vulkan
- 物理设备
- 3D
categories:
- Vulkan
- Shader
---

# Compute Shader 调度

有些时候在决定ComputeShader尺寸的时候只能靠拍脑袋，具体数值应该怎样确定不得而知，为了帮助大家理解如何确定数值，对GPU的调度做了一个简要的分析。

<!-- more -->

## GPU架构简单说明

先来看一张架构图，以下是**NVIDIA**的**Pascal**或**Turing**架构的总览图：

![0](0.png)

下面是实物图，将实物图跟上面的示意图对照看，可以发现结构几乎一致。

![1](1.png)

### GigaThread Engine

首先来看一下顶部，第一个是**PCI-E 3.0**接口。第二个是GPU内部的调度引擎，称为**GigaThread Engine**，具体怎么工作的不得而知，可以知道的是所有的工作都是由它进行调度。

![2](2.png)

### GPC

然后一大块一大块的**GPC**模块，是如下图红箭头所示。

![3](3.png)

**GPC**是`Graphics Processing Cluster`的简称，可以理解为处理单元簇，GPU被划分为几个这样的簇。低中高端显卡决定了簇的数量。

### SM

注意看上图，上图里面一个**GPC**又被划分为很多**SM**单元。**SM**是`Streaming Multiprocessor`的简称，称为流处理器。如下图所示：

![4](4.png)

注意看黄色的小字：**Warp Scheduler + Dispatch (32 thread/clk)**，SM里面有一个**Warp**调度器。

### Warp

**Warp**是什么呢？Warp是Nvidia用来描述线程束(集合)取得一个名词，一个**Warp**里面有**32**个线程，一个Warp是同步执行相同指令的。

目前为止，一个**SM**里面包含了**32个Warp**也就是**1024**个线程。

## 调度(匹配)

最关键的地方来了，**PixelShader**的调度无法由我们控制，但是**ComputeShader**的调度是可以被我控制的。为了让**ComputeShader**拥有更好的性能，我们需要调整ComputeShader的线程组大小以及调度数量来迎合GPU。

为了更好的说明，我们以`GTX 1650 Max-Q`显卡为例，该显卡有**16**个**SM**单元，每个单元有**32**个**Warp**。恰好，通过**Nvidia**的[扩展](<https://www.khronos.org/registry/OpenGL/extensions/NV/NV_shader_thread_group.txt>)，可以在Shader里面拿到这些量，既：

- `gl_SMIDNV`为`SM ID`，范围在**[0,15]**
- `gl_WarpIDNV` 为`Warp ID`，范围在**[0,31]**
- `gl_WarpsPerSMNV`为每个**SM**的**Warp**最大数量
- `gl_SMCountNV`为GPU的最大**SM**数量

然后在**ComputeShader**里面把这些数据输出显示到一张**256x256**的图片。

```glsl
outColor = vec3(
	gl_SMIDNV   / float(gl_SMCountNV),    // 红色通道
	gl_WarpIDNV / float(gl_WarpsPerSMNV), // 绿色通道
	0
);
```

我们知道**SM=32Warp=1024Thread**，因此线程组的大小设置为**32x32x1**，刚好一个SM可以搞定。然后派发**8*8=64**个线程组出去，这样就刚好覆盖了`256x256`这张图片。

以下是输出结果：

![dispatch-32](dispatch-32.gif)![dispatch-32-red](dispatch-32-red.gif)![dispatch-32-green](dispatch-32-green.gif)

第一张图片是红色通道跟绿色通道的混合结果，第二张是拆出来的红色通道，第三张是拆出来的绿色通道。

### 红色通道

![dispatch-32-zoom-red](dispatch-32-red.gif)![dispatch-32-zoom-red](dispatch-32-zoom-red.gif)

首先注意看**第一张**图片，我们可以发现第一排跟第二排保持不变，颜色红浅到深。这个其实应该很好理解，显卡一共只有**16**个**SM**单元，那么前16个线程组刚好可以被调度完，这个其实就是它颜色保持不变的原因。

然后看一下后面的**6**排图像，我们可以发现这些图像一直在闪动，颜色从浅到深随机出现。这个其实也比较好理解，因为一共只有**16**个**SM**单元，发出了**64**个线程组，势必会有**48**个线程组需要排队被调度。那这48个就只能等待哪个**SM**有空了就被调度，所以颜色呈不规律变化。

### 绿色通道

![dispatch-32-green](dispatch-32-green.gif)![dispatch-32-zoom-green](dispatch-32-zoom-green.gif)

红色通道说明了**SM**的调度策略，绿色通道就说明了**Warp**的调度策略。注意看第二张图，放大版本。

我们可以观察到在水平线上，颜色保持一致。水平线上保持一致，是因为**32**个线程被归类为了一组(**Warp**)，所以水平线颜色保持一致。

但是垂直线颜色比较**规律**或者**随机**，这个是因为**Warp Scheduler**的调度机制决定的导致**Warp ID**不一致，具体如何调度的不得而知。

## 调度(不匹配)

现在把线程组大小设置为**13x13**而不是**32x32**，为了覆盖**256x256**这张图片需要发出**20x20**个线程组。结果如下所示：

![dispatch13](dispatch13.gif)![dispatch13-green](dispatch13-green.gif)![dispatch13-red](dispatch13-red.gif)

![dispatch13-zoom](dispatch13-zoom.gif)![dispatch13-zoom-red](dispatch13-zoom-red.gif)![dispatch13-zoom-green](dispatch13-zoom-green.gif)

我们就可以发现调度变得混乱起来。大家可以算一下上面这种情况线程的利用率会是多少。
