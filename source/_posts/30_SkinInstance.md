---
title: 30_SkinInstance
date: 2019-12-27 23:21:39
tags:
- Vulkan
- Skeleton
- 3D
- Demo
categories:
- Vulkan
---

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

上个Demo提到，在某些时候可以扩大上个Demo的优势，那么这里将会展示如何利用Instance来扩大这种优势。

<!-- more -->

![30_SkinInstance](https://raw.githubusercontent.com/BobLChen/VulkanDemos/master/preview/32_SkinInstance.gif)

## InstanceDraw

InstanceDraw又称实例化渲染，这种技术可以加速相同模型的渲染。例如大面积的草地、树木等，只要是模型相同、材质相同的情况下就可以利用`InstanceDraw`来加速渲染。那么就可以利用这项技术来加速大批量重复角色的动画渲染。

InstanceDraw需要额外提供一份InstanceBuffer数据，用来描述每个Instance的特化数据，例如位移、旋转、缩放以及自定义的数据等。在这里我们随机出一些旋转、位移以及不同起始动画的角色出来：

```c++
// animation
SetAnimation(0);
CreateAnimTexture(cmdBuffer);

// instance data
vk_demo::DVKMesh* mesh = m_RoleModel->meshes[0];
Matrix4x4 meshGlobal = mesh->linkNode->GetGlobalMatrix();
vk_demo::DVKPrimitive* primitive = m_RoleModel->meshes[0]->primitives[0];
primitive->instanceDatas.resize(9 * INSTANCE_COUNT);

for (int32 i = 0; i < INSTANCE_COUNT; ++i)
{
    Vector3 translate;
    translate.x = MMath::RandRange(-300.0f, 300.0f);
    translate.y = MMath::RandRange(-180.0f, 180.0f);
    translate.z = MMath::RandRange(-150.0f, 150.0f);

    Matrix4x4 matrix = meshGlobal;
    matrix.AppendRotation(MMath::RandRange(0.0f, 360.0f), Vector3::UpVector);
    matrix.AppendTranslation(translate);

    Quat quat   = matrix.ToQuat();
    Vector3 pos = matrix.GetOrigin();
    float dx = (+0.5) * ( pos.x * quat.w + pos.y * quat.z - pos.z * quat.y);
    float dy = (+0.5) * (-pos.x * quat.z + pos.y * quat.w + pos.z * quat.x);
    float dz = (+0.5) * ( pos.x * quat.y - pos.y * quat.x + pos.z * quat.w);
    float dw = (-0.5) * ( pos.x * quat.x + pos.y * quat.y + pos.z * quat.z);

    int32 index = i * 9;
    primitive->instanceDatas[index + 0] = quat.x;
    primitive->instanceDatas[index + 1] = quat.y;
    primitive->instanceDatas[index + 2] = quat.z;
    primitive->instanceDatas[index + 3] = quat.w;
    primitive->instanceDatas[index + 4] = dx;
    primitive->instanceDatas[index + 5] = dy;
    primitive->instanceDatas[index + 6] = dz;
    primitive->instanceDatas[index + 7] = dw;
    primitive->instanceDatas[index + 8] = MMath::RandRange(0, m_Keys.size()) * mesh->bones.size() * 2;
}

primitive->indexBuffer->instanceCount = 1024;
primitive->instanceBuffer = vk_demo::DVKVertexBuffer::Create(m_VulkanDevice, cmdBuffer, primitive->instanceDatas, m_RoleShader->instancesAttributes);
        
```

需要注意的是`primitive->instanceDatas`里面利用对偶四元素存放了位移+旋转数据，最后一位则存储了动画的起始帧。

最后在提交Command的时候单独设置一下InstanceBuffer即可，这里我Mesh的接口进行了简单封装，提交的代码如下：

```c++
void BindDrawCmd(VkCommandBuffer cmdBuffer)
{
    if (vertexBuffer) {
        vkCmdBindVertexBuffers(cmdBuffer, 0, 1, &(vertexBuffer->dvkBuffer->buffer), &(vertexBuffer->offset));
    }
    
    if (instanceBuffer) {
        vkCmdBindVertexBuffers(cmdBuffer, 1, 1, &(instanceBuffer->dvkBuffer->buffer), &(instanceBuffer->offset));
    }
    
    if (indexBuffer) {
        vkCmdBindIndexBuffer(cmdBuffer, indexBuffer->dvkBuffer->buffer, 0, indexBuffer->indexType);
    }
    
    if (vertexBuffer && !indexBuffer) {
        vkCmdDraw(cmdBuffer, vertexCount, 1, 0, 0);
    }
    else {
        vkCmdDrawIndexed(cmdBuffer, indexBuffer->indexCount, indexBuffer->instanceCount, 0, 0, 0);
    }
}
```

其它地方没有多少变化，最后就是在VertexShader里面有一些细微的改动。主要就是输入的源多了Instance的数据:

```glsl
layout (location = 4) in vec4 inInstanceQuat0;
layout (location = 5) in vec4 inInstanceQuat1; 
layout (location = 6) in float inInstanceAnim;
```

注意这里是两个Vec4 + 一个float刚好跟代码里面的`primitive->instanceDatas`格式对应起来。
然后根据Instance数据取出对应的动画即可。

```glsl
#version 450

layout (location = 0) in vec3 inPosition;
layout (location = 1) in vec2 inUV0;
layout (location = 2) in vec3 inNormal;
layout (location = 3) in vec3 inSkinPack;

layout (location = 4) in vec4 inInstanceQuat0;
layout (location = 5) in vec4 inInstanceQuat1; 
layout (location = 6) in float inInstanceAnim;

layout (binding = 0) uniform MVPBlock 
{
	mat4 modelMatrix;
	mat4 viewMatrix;
	mat4 projectionMatrix;

	vec4 animIndex;
} paramData;

layout (binding = 1) uniform sampler2D animMap;

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

mat2x4 ReadBoneAnim(int boneIndex, int startIndex)
{
	// 计算出当前帧动画的骨骼的索引
	int index = startIndex + boneIndex * 2;
	int animY = index / int(paramData.animIndex.x);
	int animX = index % int(paramData.animIndex.x);

	vec2 index0 = vec2(animX + 0.0f, animY) / paramData.animIndex.xy;
	vec2 index1 = vec2(animX + 1.0f, animY) / paramData.animIndex.xy;

	mat2x4 animData;
	animData[0] = texture(animMap, index0);
	animData[1] = texture(animMap, index1);

	return animData;
}

mat2x4 CalcDualQuat(ivec4 skinIndex, vec4 skinWeight, int startIndex)
{
	mat2x4 dualQuat0 = ReadBoneAnim(skinIndex.x, startIndex);
	mat2x4 dualQuat1 = ReadBoneAnim(skinIndex.y, startIndex);
	mat2x4 dualQuat2 = ReadBoneAnim(skinIndex.z, startIndex);
	mat2x4 dualQuat3 = ReadBoneAnim(skinIndex.w, startIndex);

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

	return blendDualQuat;
}

void main() 
{
	vec4 position;
	vec3 normal;

	int animIndex = int(inInstanceAnim) + int(paramData.animIndex.z);
	animIndex = animIndex % int(paramData.animIndex.w);
	
	// skin info
	ivec4 skinIndex   = UnPackUInt32To4Byte(uint(inSkinPack.x));
	ivec2 skinWeight0 = UnPackUInt32To2Short(uint(inSkinPack.y));
	ivec2 skinWeight1 = UnPackUInt32To2Short(uint(inSkinPack.z));
	vec4  skinWeight  = vec4(skinWeight0 / 65535.0, skinWeight1 / 65535.0);

	mat2x4 dualQuat = CalcDualQuat(skinIndex, skinWeight, animIndex);

	position = vec4(DualQuatTransformPosition(dualQuat, inPosition.xyz), 1.0);
	normal   = DualQuatTransformVector(dualQuat, inNormal);

	mat2x4 instanceDualQuat;
	instanceDualQuat[0] = inInstanceQuat0;
	instanceDualQuat[1] = inInstanceQuat1;
	position = vec4(DualQuatTransformPosition(instanceDualQuat, position.xyz), 1.0);
	normal   = DualQuatTransformVector(instanceDualQuat, normal);

	outUV     = inUV0;
	outNormal = normal;
	
	gl_Position = paramData.projectionMatrix * paramData.viewMatrix * position;
}
```

加入InstanceDraw的功能进来也并不困难，但是效果还是非常可观的，从预览图中可以看出，每个角色的位移、旋转、动画都不相同，这种情况非常适合战场超量士兵等类型游戏。

