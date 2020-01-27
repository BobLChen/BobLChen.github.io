---
title: 27_SkeletonPackIndexWeight
date: 2019-11-13 23:22:19
tags:
- Vulkan
- Skeleton
- 3D
- Demo
categories:
- Vulkan
---

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

上一个Demo里面给出了一个骨骼动画的实现，但是并不是最优的状态，在这一节里面给出一些简单的优化。

<!-- more -->

![26_Skeleton](https://raw.githubusercontent.com/BobLChen/VulkanDemos/master/preview/26_Skeleton.gif)

## 优化骨骼索引

上一个Demo里面，采用32位来存储骨骼索引数据，但是在真实的场景中不可能使用到这么多骨骼来驱动，一般使用到256根骨骼就已经是非常精良的制作。基于此其实可以将32bit压缩为8bit，8bit已经能够满足大多数的需求。使用8bit来存储也有两种方式：1、将8个bit打包到32bit里面，一般也就是1个int存储了4个索引的数据。2、打包解包需要计算，其实可以直接设定顶点的数据格式为byte。在这里我采样第一种方式，有两个原因：1、Demo的设计里面没有支持byte的形式；2、旧设备也能得到支持。

先来看看在CPU里面进行pack的代码:
```c++
int32 idx0 = primitive->vertices[idx + 0];
int32 idx1 = primitive->vertices[idx + 1];
int32 idx2 = primitive->vertices[idx + 2];
int32 idx3 = primitive->vertices[idx + 3];
// packIndex
uint32 packIndex = (idx0 << 24) + (idx1 << 16) + (idx2 << 8) + idx3;
primitive->vertices[idx + 0] = packIndex;
primitive->vertices[idx + 1] = packIndex;
primitive->vertices[idx + 2] = packIndex;
primitive->vertices[idx + 3] = packIndex;
```

然后看看shader里面的unpack代码:
```glsl
ivec4 UnPackUInt32To4Byte(uint packIndex)
{
	uint idx0 = (packIndex >> 24) & 0xFF;
	uint idx1 = (packIndex >> 16) & 0xFF;
	uint idx2 = (packIndex >> 8)  & 0xFF;
	uint idx3 = (packIndex >> 0)  & 0xFF;
	return ivec4(idx0, idx1, idx2, idx3);
}
```

## 优化骨骼权值

权重值得优化套路其实跟索引一样，权重值采用16bit来存储，一般情况下这样的精度是可以满足需求的，除非是对动画要求特别高得场景。既然套路都一样，这里也就不再过多解释。看一下代码即可：

```c++
// packWeight
uint16 weight0 = primitive->vertices[idx + 4] * 65535;
uint16 weight1 = primitive->vertices[idx + 5] * 65535;
uint16 weight2 = primitive->vertices[idx + 6] * 65535;
uint16 weight3 = primitive->vertices[idx + 7] * 65535;
uint32 packWeight0 = (weight0 << 16) + weight1;
uint32 packWeight1 = (weight2 << 16) + weight3;
primitive->vertices[idx + 4] = packWeight0;
primitive->vertices[idx + 5] = packWeight1;
primitive->vertices[idx + 6] = packWeight0;
primitive->vertices[idx + 7] = packWeight1;
```

```glsl
ivec2 UnPackUInt32To2Short(uint packIndex)
{
	uint idx0 = (packIndex >> 16) & 0xFFFF;
	uint idx1 = (packIndex >> 0)  & 0xFFFF;
	return ivec2(idx0, idx1);
}
```

这个的优化主要目的是减少模型的数据量。下一个Demo着手来优化动画的数量量。


