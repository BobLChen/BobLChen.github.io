---
title: 28_SkeletonQuat
date: 2019-11-29 23:12:29
tags:
- Vulkan
- Skeleton
- 3D
- Demo
categories:
- Vulkan
---

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

上个Demo里面对骨骼动画做了一些简单的数据压缩，但是在移动设备上，某些极端情况下还能持续的进行压缩以及优化，下面将展示如何利用对偶四元素压缩数据。 

<!-- more -->

![28_SkeletonDualQuat](https://raw.githubusercontent.com/BobLChen/VulkanDemos/master/preview/28_SkeletonDualQuat.gif)

## 关于缩放

在常规的动画中，缩放其实很少用到，大多数情况下不会用到缩放。比如人形角色一般不会制作带有缩放的骨骼动画，因为加了缩放会变得比较奇怪。基于这种情况，可以考虑把缩放因子从动画数据中移除掉，那么参与动画的实际上只有`位移`以及`旋转`数据。

## 数据重组

由于拿掉了缩放数据，就需要考虑到一个问题，就是如何把位移以及旋转数据重组成矩阵数据以方便在Shader中进行蒙皮操作。通过位移以及旋转重组出矩阵这个过程其实还蛮复杂的，参考下面的矩阵：

![0.png](0.png)

上图只是一个旋转的矩阵，位移的矩阵非常简单，就不在说明。主要考察这个旋转的矩阵，会发现要把旋转转化为矩阵这个过程计算量非常大，如果在Shader里面要完成4次这样的转换会感觉有点儿得不偿失，既虽然压缩了数据，但是极大的增加了计算量。为了解决这个问题，可以利用对偶四元素来加速这个过程的计算，总体来说会得到一个比较均衡的状态。另外一个比较有趣的地方是在某些情况下对偶四元素的动画效果要优于矩阵的方式。

## 对偶四元素

对偶四元素的具体推导以及原理这里不再细说，要推导起来估计没个上万字是搞不定的，这里提供一些参考资料。某些时候大家不需要了解其原理，只要能把算法应用起来即可。
[参考链接](https://www.chinedufn.com/dual-quaternion-shader-explained/)

位移+旋转转化为对偶四元素的过程其实非常简单，代码如下所示：
```
// 从Transform矩阵中获取四元数以及位移信息
Quat quat   = boneTransform.ToQuat();
Vector3 pos = boneTransform.GetOrigin();

// 转为使用对偶四元数
float dx = (+0.5) * ( pos.x * quat.w + pos.y * quat.z - pos.z * quat.y);
float dy = (+0.5) * (-pos.x * quat.z + pos.y * quat.w + pos.z * quat.x);
float dz = (+0.5) * ( pos.x * quat.y - pos.y * quat.x + pos.z * quat.w);
float dw = (-0.5) * ( pos.x * quat.x + pos.y * quat.y + pos.z * quat.z);

// 设置参数
m_BonesData.dualQuats[j * 2 + 0].Set(quat.x, quat.y, quat.z, quat.w);
m_BonesData.dualQuats[j * 2 + 1].Set(dx, dy, dz, dw);
```
数据转化完成之后即可上传到GPU，在Shader里面实现蒙皮操作。

## 蒙皮

在Vertex Shader里面定义的Uniform数据格式需要简单调整一下：

```
layout (binding = 1) uniform BonesTransformBlock 
{
	mat2x4 	dualQuats[MAX_BONES];
	vec4 	debugParam;
} bonesData;
```

因为每根骨骼只需要传输两个Vector4数据，因此用mat2x4既2行似列的数据格式来映射。

然后就是在VertexShader里面安装索引以及权重插值计算出最终的对偶四元素。

```
// skin info
ivec4 skinIndex   = UnPackUInt32To4Byte(uint(inSkinPack.x));
ivec2 skinWeight0 = UnPackUInt32To2Short(uint(inSkinPack.y));
ivec2 skinWeight1 = UnPackUInt32To2Short(uint(inSkinPack.z));
vec4  skinWeight  = vec4(skinWeight0 / 65535.0, skinWeight1 / 65535.0);

// dual quats
mat2x4 dualQuat0 = bonesData.dualQuats[skinIndex.x];
mat2x4 dualQuat1 = bonesData.dualQuats[skinIndex.y];
mat2x4 dualQuat2 = bonesData.dualQuats[skinIndex.z];
mat2x4 dualQuat3 = bonesData.dualQuats[skinIndex.w];

if (dot(dualQuat0[0], dualQuat1[0]) < 0.0) {
	dualQuat1 *= -1.0;
}
if (dot(dualQuat0[0], dualQuat2[0]) < 0.0) {
	dualQuat2 *= -1.0;
}
if (dot(dualQuat0[0], dualQuat3[0]) < 0.0) {
	dualQuat3 *= -1.0;
}

mat2x4 blendDualQuat = dualQuat0 * skinWeight.x;
blendDualQuat += dualQuat1 * skinWeight.y;
blendDualQuat += dualQuat2 * skinWeight.z;
blendDualQuat += dualQuat3 * skinWeight.w;
```

最后利用`blendDualQuat`对顶点、法线、切线数据进行转化即可。

```
vec3 DualQuatTransformPosition(mat2x4 dualQuat, vec3 position)
{
	float len = length(dualQuat[0]);
	dualQuat /= len;
	
	vec3 result = position.xyz + 2.0 * cross(dualQuat[0].xyz, cross(dualQuat[0].xyz, position.xyz) + dualQuat[0].w * position.xyz);
	vec3 trans  = 2.0 * (dualQuat[0].w * dualQuat[1].xyz - dualQuat[1].w * dualQuat[0].xyz + cross(dualQuat[0].xyz, dualQuat[1].xyz));
	result += trans;

	return result;
}

vec3 DualQuatTransformVector(mat2x4 dualQuat, vec3 vector)
{
	return vector + 2.0 * cross(dualQuat[0].xyz, cross(dualQuat[0].xyz, vector) + dualQuat[0].w * vector);
}

// 对偶四元素还原Matrix进行计算
if (bonesData.debugParam.x < 1.0)
{
	mat4x4 boneMatrix = DualQuat2Matrix(blendDualQuat[0], blendDualQuat[1]);
	position = boneMatrix * vec4(inPosition.xyz, 1.0);
	normal   = mat3(boneMatrix) * inNormal;

	outColor = vec4(1.1, 1.0, 1.0, 1.0);
}
// 不还原，直接计算
else
{
	position.xyz = DualQuatTransformPosition(blendDualQuat, inPosition.xyz);
	position.w   = 1.0;
	normal = DualQuatTransformVector(blendDualQuat, inNormal);

	outColor = vec4(1.0, 1.1, 1.0, 1.0);
}

// 转换法线
mat3 normalMatrix = transpose(inverse(mat3(uboMVP.modelMatrix)));
normal = normalize(normalMatrix * normal);
```

首先定义出针对顶点的转化函数`DualQuatTransformPosition`以及针对向量的转化函数`DualQuatTransformVector`，然后对输入的顶点以及法线数据转化即可。最终的Shader代码如下所示：
```
#version 450

layout (location = 0) in vec3  inPosition;
layout (location = 1) in vec2  inUV0;
layout (location = 2) in vec3  inNormal;
layout (location = 3) in vec3  inSkinPack;

layout (binding = 0) uniform MVPBlock 
{
	mat4 modelMatrix;
	mat4 viewMatrix;
	mat4 projectionMatrix;
} uboMVP;

#define MAX_BONES 64

layout (binding = 1) uniform BonesTransformBlock 
{
	mat2x4 	dualQuats[MAX_BONES];
	vec4 	debugParam;
} bonesData;

layout (location = 0) out vec2 outUV;
layout (location = 1) out vec3 outNormal;
layout (location = 2) out vec4 outColor;

out gl_PerVertex 
{
    vec4 gl_Position;   
};

ivec4 UnPackUInt32To4Byte(uint packIndex)
{
	uint idx0 = (packIndex >> 24) & 0xFF;
	uint idx1 = (packIndex >> 16) & 0xFF;
	uint idx2 = (packIndex >> 8)  & 0xFF;
	uint idx3 = (packIndex >> 0)  & 0xFF;
	return ivec4(idx0, idx1, idx2, idx3);
}

ivec2 UnPackUInt32To2Short(uint packIndex)
{
	uint idx0 = (packIndex >> 16) & 0xFFFF;
	uint idx1 = (packIndex >> 0)  & 0xFFFF;
	return ivec2(idx0, idx1);
}

mat4x4 DualQuat2Matrix(vec4 real, vec4 dual) {
	
	float len2 = dot(real, real);
	float qx = real.x;
	float qy = real.y;
	float qz = real.z;
	float qw = real.w;

	float tx = dual.x;
	float ty = dual.y;
	float tz = dual.z;
	float tw = dual.w; 
	
	mat4x4 matrix;
	matrix[0][0] = qw * qw + qx * qx - qy * qy - qz * qz;
	matrix[1][0] = 2 * qx * qy - 2 * qw * qz;
	matrix[2][0] = 2 * qx * qz + 2 * qw * qy;
	matrix[0][1] = 2 * qx * qy + 2 * qw * qz;
	matrix[1][1] = qw * qw + qy * qy - qx * qx - qz * qz;
	matrix[2][1] = 2 * qy * qz - 2 * qw * qx;
	matrix[0][2] = 2 * qx * qz - 2 * qw * qy;
	matrix[1][2] = 2 * qy * qz + 2 * qw * qx;
	matrix[2][2] = qw * qw + qz * qz - qx * qx - qy * qy;

	matrix[3][0] = -2 * tw * qx + 2 * qw * tx - 2 * ty * qz + 2 * qy * tz;
	matrix[3][1] = -2 * tw * qy + 2 * tx * qz - 2 * qx * tz + 2 * qw * ty;
	matrix[3][2] = -2 * tw * qz + 2 * qx * ty + 2 * qw * tz - 2 * tx * qy;

	matrix[0][3] = 0;
	matrix[1][3] = 0;
	matrix[2][3] = 0;
	matrix[3][3] = len2;

	matrix /= len2;

	return matrix;
}

vec3 DualQuatTransformPosition(mat2x4 dualQuat, vec3 position)
{
	float len = length(dualQuat[0]);
	dualQuat /= len;
	
	vec3 result = position.xyz + 2.0 * cross(dualQuat[0].xyz, cross(dualQuat[0].xyz, position.xyz) + dualQuat[0].w * position.xyz);
	vec3 trans  = 2.0 * (dualQuat[0].w * dualQuat[1].xyz - dualQuat[1].w * dualQuat[0].xyz + cross(dualQuat[0].xyz, dualQuat[1].xyz));
	result += trans;

	return result;
}

vec3 DualQuatTransformVector(mat2x4 dualQuat, vec3 vector)
{
	return vector + 2.0 * cross(dualQuat[0].xyz, cross(dualQuat[0].xyz, vector) + dualQuat[0].w * vector);
}

void main() 
{
	// skin info
	ivec4 skinIndex   = UnPackUInt32To4Byte(uint(inSkinPack.x));
	ivec2 skinWeight0 = UnPackUInt32To2Short(uint(inSkinPack.y));
	ivec2 skinWeight1 = UnPackUInt32To2Short(uint(inSkinPack.z));
	vec4  skinWeight  = vec4(skinWeight0 / 65535.0, skinWeight1 / 65535.0);

	// dual quats
	mat2x4 dualQuat0 = bonesData.dualQuats[skinIndex.x];
	mat2x4 dualQuat1 = bonesData.dualQuats[skinIndex.y];
	mat2x4 dualQuat2 = bonesData.dualQuats[skinIndex.z];
	mat2x4 dualQuat3 = bonesData.dualQuats[skinIndex.w];
	
	if (dot(dualQuat0[0], dualQuat1[0]) < 0.0) {
		dualQuat1 *= -1.0;
	}
	if (dot(dualQuat0[0], dualQuat2[0]) < 0.0) {
		dualQuat2 *= -1.0;
	}
	if (dot(dualQuat0[0], dualQuat3[0]) < 0.0) {
		dualQuat3 *= -1.0;
	}

	mat2x4 blendDualQuat = dualQuat0 * skinWeight.x;
	blendDualQuat += dualQuat1 * skinWeight.y;
	blendDualQuat += dualQuat2 * skinWeight.z;
	blendDualQuat += dualQuat3 * skinWeight.w;

	vec4 position;
	vec3 normal;

	// 对偶四元素还原Matrix进行计算
	if (bonesData.debugParam.x < 1.0)
	{
		mat4x4 boneMatrix = DualQuat2Matrix(blendDualQuat[0], blendDualQuat[1]);
		position = boneMatrix * vec4(inPosition.xyz, 1.0);
		normal   = mat3(boneMatrix) * inNormal;

		outColor = vec4(1.1, 1.0, 1.0, 1.0);
	}
	// 不还原，直接计算
	else
	{
		position.xyz = DualQuatTransformPosition(blendDualQuat, inPosition.xyz);
		position.w   = 1.0;
		normal = DualQuatTransformVector(blendDualQuat, inNormal);

		outColor = vec4(1.0, 1.1, 1.0, 1.0);
	}

	// 转换法线
	mat3 normalMatrix = transpose(inverse(mat3(uboMVP.modelMatrix)));
	normal = normalize(normalMatrix * normal);

	outUV     = inUV0;
	outNormal = normal;
	
	gl_Position = uboMVP.projectionMatrix * uboMVP.viewMatrix * uboMVP.modelMatrix * position;
}
```

可以发现利用对偶四元素不仅可以压缩掉一部分数据，还能保证动画的效果质量，整个优化的改动其实也比较小，针对移动设备，这种微小的优化在某些时候是非常可取的。