---
title: SnapdragonProfiler工具
date: 2020-08-22 10:42:13
tags:
- 3D
categories:
- 调试工具
---

# Snapdragon Profiler 工具

Snapdragon Profiler工具非常强大，适用于骁龙系列的芯片，这个工具如果使用恰当，对于性能分析及其有用，远胜于Unity的FrameDebugger。

<!-- more -->

## 前置条件

- Android、OpenGL ES、JNI
- 两个充满屏幕的三角形
- 骁龙CPU
- Snapdragon Profiler工具的Snapshot功能

## 计算量

运算量是衡量APP性能优异的一个点，**一般**运算量越大所消耗的性能就更多。恰好，`Snapdragon Profiler`这个工具可以获取到APP在一帧里面每次操作的运算量。

以下图为例：

注意看红框所圈出来的数据，其中`Vertices Shaded`表示提交的顶点数量，这里数据为`6`刚好是我在OpenGL里面提交的数量。注意看`Vertex Instructions`为`0`，这个也是符合预期的，因为在VertexShader里面没有做任何计算。

```glsl
attribute vec4 vPosition;
void main()
{
    gl_Position = vPosition;
}
```

![1597568596337](1597568596337.png)



为了更好的展示这个效果，将VertexShader做一些修改，然后重新打包捕获，如下所示：

我们发现`Vertex Instructions`多了16次，但是`Vertex Shader`里面其实只多了两次数学运算，`2 * 6 = 12`跟`16`也无法匹配。这个我也不知道为啥，我猜测可能是由于`GPU`架构原因导致多执行了几次，这个我在PC上面核对了一下，PC的数据是正常的。

![20200816202650](20200816202650.png)



然后我们在`Vertex Shader`里面增加一次数学运算，然后看一下结果：

我们会发现运算量变为了`24`次，这个结果也跟预期符合。

![1597580956384](1597580956384.png)

由上述可知，我们可以通过观测`Vertices Shaded`以及`Vertex Instructions`来衡量`Vertex Shader`的运算量。



`Vertex Shader`完了之后，还有`Pixel Shader`，这个才是大头，还是先从最简单的状态开始，如下所示：

注意红框标识出来的信息，分辨率为：1080x2184，绘制了两个三角形，`Pixel Shader`非常简单，只是一个赋值操作。

![1597567606004](1597567606004.png)

首先计算分辨率`1080x2184=2358720`，然后观测`Fragments Shaded`数据为`2362544`，这个数值跟`2358720`非常接近，我**怀疑**多的一部分数据可能是对角线的像素量，但是具体是什么情况不清楚。

然后注意看`Fragment Instructions`数据为`9447936`，差不多刚好是提交的4倍，这个的差异运算量也不太清楚具体是干什么去了，但是并不影响我们使用。



然后我们来看下面这幅图，观测一下里面的数据：

还是看红框数据，同样的是一次`DrawCall`，但是我们发现`Fragments Shaded`以及`Fragment Instructions`数据都为`0`。之所以出现这样的情况，只是我把深度测试打开了，然后第二次`DrawCall`由于深度测试失败，导致`Fragment Shader`不需要执行，因此数据都为`0`。

![1597568596337](1597568596337.png)



然后改变一下Shader，在`VertexShader`里面传递一个Vector4数据到`FragmentShader`。`VertexShader`传递给`FragmentShader`的数据，如果没有特殊指明，都会被重新插值输出给`FragmentShader`。

所以我们会发现`Fragment ALU Instructions(Half)`多了`9447936`，这个的数量刚好跟`Fragment ALU Instructions(Full)`吻合。

`ALU`指代简单的逻辑算术运算，类似加减乘除。`EFU`指代比较特殊的运算，类似`Sin`、`Cos`等之类，`Full=Half * 2`。

然后看一下`Fragment Instructions`数据`14171904`刚好等于`9447936`乘以`1.5`。

![1597582722035](1597582722035.png)



上述是插值引起的计算量，那么我们给`Fragment Shader`增加一次数学运算，然后看一下结果如何：

我们发现`Fragment ALU Instructions(Half)`数据变为`11809920`，与`9447936`之差为`2361984`。

![1597583017674](1597583017674.png)



如果再增加一次数学运算，增加的计算量保持一致，并且跟提交的`2362544`非常接近：14171904-11809920=2361984`。

![1597587095959](1597587095959.png)



最后看一下`AlphaTest`功能对运算量的影响：

注意看`Fragment ALU Instructions(Half)`数据跟我们上述增加一次数学运算之后的运算量保持一致。

![1597588094619](1597588094619.png)



然后看一下开启深度测试之后，第一次正常绘制，第二次开启`AlphaTest`功能的状态：

会发现`Fragment Shader`执行次数都为0，这个其实很好理解，`AlphaTest功能`只是屏蔽了一部分`Fragment`的写入，但是它并没有改变`Fragment`固有的深度值，所以深度测试仍然提前过滤掉一批像素。PS：注意深度测试条件。

![1597589633278](1597589633278.png)



最后试一下关闭深度测试之后，第一次正常绘制，第二次开启`AlphaTest`功能并将`Discard`条件前置，观测一下有没有变化：

我们发现`Fragment ALU Instructions(Half)`运算量跟上述两次数学运算量保持一致，也比较符合我们的预期，既条件前置会屏蔽后续的计算。

![1597588617676](1597588617676.png)