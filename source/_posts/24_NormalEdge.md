---
title: 24_NormalEdge
date: 2019-10-19 23:51:01
tags:
- Vulkan
- 3D
- Edge
- Tutorial
categories:
- Vulkan
---

[24_NormalEdge](https://github.com/BobLChen/VulkanDemos/tree/master/examples/24_NormalEdge)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

在这个Demo里面加强一下RenderTarget的使用，顺便试试输出多个数据是否可以很好的工作。在这个Demo里面输出Color以及Normal数据，利用Normal数据来检测边缘。

<!-- more -->

![24_NormalEdge](https://raw.githubusercontent.com/BobLChen/VulkanDemos/master/preview/24_NormalEdge.jpg)

## 描边
描边的方法很多，不同的需求有着不同的实现。在这个Demo里面我将展示一种非常规方式实现的描边效果。但是还是需要说明一下常规的一些描边方法：

### 膨胀顶点
这种方式其实是比较常用的方式，毕竟这种方式消耗比较少，效果还不错。实现方式主要是通过Stencil来实现，思路其实跟之前**21_Stencil**这个Demo一样，在第二遍绘制的时候把模型放大，然后利用Stencil剔除掉原先的模型，这样边缘就出现了。放大的方法一般有两种，一种是直接缩放模型，这种方式效果非常差，但是好在不需要额外的Shader，直接缩放即可。另外一种是在顶点Shader里面按照顶点法线的方向进行缩放，这种效果在圆润的物体上面效果不错，但是在尖锐的物体上面可能表现的比较差。

### RimLight
这种方式我个人觉得其实不算是描边，而是一种边缘发光的特效。这个的实现方式跟**21_Stencil**里面的特效一样，利用**View**的方向与法线的夹角来确定边缘亮度，这种方式不能在模型周围描出边缘。

### 后处理抠图
这种方式应该是效果最好的，跟模型的形状无关。这种方式实现起来也比较简单，第一步先用纯白色绘制物体到RenderTarget，第二步对RenderTarget进行模糊处理，横向一遍，纵向一遍。第三步用第二步的图减去第一步的图，就得到了边缘，边缘的粗细主要由第二步模糊的程度来决定。

### 利用法线描边
这种方式需要在后处理里面做，第一步先输出每个像素的法线数据，第二步采样像素附近的法线，然后看一下附近的法线与当前的法线夹角是否过大，如果夹角非常大就说明是边缘，如果夹角较小说明是连续的。Shader代码如下：
```glsl
#version 450

layout (location = 0) in vec3 inPosition;
layout (location = 1) in vec2 inUV0;

layout (location = 0) out vec2 outUV0;

out gl_PerVertex 
{
    vec4 gl_Position;
};

void main() 
{
    outUV0 = inUV0;
	gl_Position = vec4(inPosition, 1.0);
}
```

```glsl
#version 450

layout (location = 0) in vec2 inUV0;

layout (binding  = 1) uniform sampler2D diffuseTexture;
layout (binding  = 2) uniform sampler2D normalsTexture;

layout (binding = 0) uniform FilterParam 
{
    vec4 size;
} param;

layout (location = 0) out vec4 outFragColor;

void main() 
{

    const vec2 PixelKernel[4] =
    {
        { 0,  1 },
        { 1,  0 },
        { 0, -1 },
        {-1,  0 }
    };
    
    vec4 oriNormal = texture(normalsTexture, inUV0);
    oriNormal = oriNormal * 2 - 1;

    vec4 sum = vec4(0);
    for (int i = 0; i < 4; ++i)
    {
        vec4 dstNormal = texture(normalsTexture, inUV0 + PixelKernel[i] / param.size.xy);
        dstNormal = dstNormal * 2 - 1;

        float odotd = dot(oriNormal.xyz, dstNormal.xyz);
        odotd = min(odotd, 1.0);
        odotd = max(odotd, 0.0);
        sum += 1.0 - odotd;
    }

    vec4 diffuse = texture(diffuseTexture, inUV0);
    if (param.size.z > 1.0) {
        outFragColor = sum * diffuse;
    } else {
        outFragColor = sum;
    }
}
```

其它的就大同小异了，无非就是配置一下RenderTarget之类的，就不再细说了。
