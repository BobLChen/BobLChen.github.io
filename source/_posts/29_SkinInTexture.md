---
title: 29_SkinInTexture
date: 2019-12-12 23:12:12
tags:
- Vulkan
- Skeleton
- 3D
- Demo
categories:
- Vulkan
---

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

上个Demo里面对动画数据进行了一定的压缩，但是有个地方问题对性能有一定的冲击，细心的同学也会有所发现...

<!-- more -->

![29_SkinInTexture](https://raw.githubusercontent.com/BobLChen/VulkanDemos/master/preview/28_SkeletonDualQuat.gif)

## Pose计算

细心的同学会发现Pose的计算量其实非常大，这个计算量跟骨骼的数量正相关。Pose的数据计算其实就是下面这个过程：

```c++
UpdateAnimation(time, delta);

// bones data
for (int32 j = 0; j < mesh->bones.size(); ++j) 
{
    int32 boneIndex = mesh->bones[j];
    vk_demo::DVKBone* bone = m_RoleModel->bones[boneIndex];

    // 获取骨骼的最终Transform矩阵
    // 也可以使用对偶四元素来替换矩阵的计算
    Matrix4x4 boneTransform = bone->finalTransform;
    boneTransform.Append(mesh->linkNode->GetGlobalMatrix().Inverse());

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
}
```

注意地方是`UpdateAnimation(time, delta)`，这个函数是根据delta时间对两个关键帧进行插值，然后通过`BindPose`计算出`ModelSpace`下的`Pose`数据。

```c++
void DVKModel::GotoAnimation(float time)
{
    if (animIndex == -1) {
        return;
    }
    
    DVKAnimation& animation = animations[animIndex];
    animation.time = MMath::Clamp(time, 0.0f, animation.duration);
    
    // update nodes animation
    for (auto it = animation.clips.begin(); it != animation.clips.end(); ++it)
    {
        vk_demo::DVKAnimationClip& clip = it->second;
        vk_demo::DVKNode* node = nodesMap[clip.nodeName];
        
        float alpha = 0.0f;
        
        // rotation
        Quat prevRot(0, 0, 0, 1);
        Quat nextRot(0, 0, 0, 1);
        clip.rotations.GetValue(animation.time, prevRot, nextRot, alpha);
        Quat retRot = MMath::Lerp(prevRot, nextRot, alpha);
        
        // position
        Vector3 prevPos(0, 0, 0);
        Vector3 nextPos(0, 0, 0);
        clip.positions.GetValue(animation.time, prevPos, nextPos, alpha);
        Vector3 retPos = MMath::Lerp(prevPos, nextPos, alpha);
        
        // scale
        Vector3 prevScale(1, 1, 1);
        Vector3 nextScale(1, 1, 1);
        clip.scales.GetValue(animation.time, prevScale, nextScale, alpha);
        Vector3 retScale = MMath::Lerp(prevScale, nextScale, alpha);
        
        node->localMatrix.SetIdentity();
        node->localMatrix.AppendScale(retScale);
        node->localMatrix.Append(retRot.ToMatrix());
        node->localMatrix.AppendTranslation(retPos);
    }

    // update bones
    for (int32 i = 0; i < bones.size(); ++i)
    {
        DVKBone* bone = bones[i];
        DVKNode* node = nodesMap[bone->name];
        // 注意行列矩阵的区别
        bone->finalTransform = bone->inverseBindPose;
        bone->finalTransform.Append(node->GetGlobalMatrix());
    }
}
```

从上面的代码可以看出，计算量其实还是比较大的，特别是角色数量非常多的情况下。当然这个得计算过程可以放到子线程里面去完成，分发到多个子线程里面，在提交Command的时间再把数据取回来。但是在某些时候我们不会关心动画的Blend效果如何或者说可以牺牲掉一些动画效果，那么在这种情况下就可以提前将Pose的数据计算出来并加以存储。在这里为了通用，采样了存储到Texture的方式进行演示，当然存储到`StorageBuffer`也是可以的，存储到Texture稍微更通用一点儿。

现在提供一个函数，用来捕获所有帧的动画数据，然后存储到Texture：

```c++
void CreateAnimTexture(vk_demo::DVKCommandBuffer* cmdBuffer)
{
    std::vector<float> animData(64 * 32 * 4); // 21个骨骼 * 30帧动画数据 * 8
    vk_demo::DVKAnimation& animation = m_RoleModel->GetAnimation();
    
    // 获取关键帧信息
    m_Keys.push_back(0);
    for (auto it = animation.clips.begin(); it != animation.clips.end(); ++it)
    {
        vk_demo::DVKAnimationClip& clip = it->second;
        for (int32 i = 0; i < clip.positions.keys.size(); ++i) {
            if (m_Keys.back() < clip.positions.keys[i]) {
                m_Keys.push_back(clip.positions.keys[i]);
            }
        }
        for (int32 i = 0; i < clip.rotations.keys.size(); ++i) {
            if (m_Keys.back() < clip.rotations.keys[i]) {
                m_Keys.push_back(clip.rotations.keys[i]);
            }
        }
        for (int32 i = 0; i < clip.scales.keys.size(); ++i) {
            if (m_Keys.back() < clip.scales.keys[i]) {
                m_Keys.push_back(clip.scales.keys[i]);
            }
        }
    }

    vk_demo::DVKMesh* mesh = m_RoleModel->meshes[0];
    
    // 存储每一帧所对应的动画数据
    for (int32 i = 0; i < m_Keys.size(); ++i)
    {
        m_RoleModel->GotoAnimation(m_Keys[i]);
        // 数据步长，一个节点的动画数据需要两个Vector存储。
        int32 step = i * mesh->bones.size() * 8;
        
        for (int32 j = 0; j < mesh->bones.size(); ++j)
        {
            int32 boneIndex = mesh->bones[j];
            vk_demo::DVKBone* bone = m_RoleModel->bones[boneIndex];
            // 获取骨骼的最终Transform矩阵
            // 也可以使用对偶四元素来替换矩阵的计算
            Matrix4x4 boneTransform = bone->finalTransform;
            boneTransform.Append(mesh->linkNode->GetGlobalMatrix().Inverse());
            // 从Transform矩阵中获取四元数以及位移信息
            Quat quat   = boneTransform.ToQuat();
            Vector3 pos = boneTransform.GetOrigin();
            // 转为使用对偶四元数
            float dx = (+0.5) * ( pos.x * quat.w + pos.y * quat.z - pos.z * quat.y);
            float dy = (+0.5) * (-pos.x * quat.z + pos.y * quat.w + pos.z * quat.x);
            float dz = (+0.5) * ( pos.x * quat.y - pos.y * quat.x + pos.z * quat.w);
            float dw = (-0.5) * ( pos.x * quat.x + pos.y * quat.y + pos.z * quat.z);
            // 计算出当前帧当前骨骼在Texture中的坐标
            int32 index = step + j * 8;
            animData[index + 0] = quat.x;
            animData[index + 1] = quat.y;
            animData[index + 2] = quat.z;
            animData[index + 3] = quat.w;
            animData[index + 4] = dx;
            animData[index + 5] = dy;
            animData[index + 6] = dz;
            animData[index + 7] = dw;
        }
    }
    
    // 创建Texture
    m_AnimTexture = vk_demo::DVKTexture::Create2D(
        (const uint8*)animData.data(), animData.size() * sizeof(float), VK_FORMAT_R32G32B32A32_SFLOAT, 
        64, 32,
        m_VulkanDevice,
        cmdBuffer
    );
    m_AnimTexture->UpdateSampler(
        VK_FILTER_NEAREST, 
        VK_FILTER_NEAREST,
        VK_SAMPLER_MIPMAP_MODE_NEAREST,
        VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE,
        VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE,
        VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE
    );
}
```

先提前把所有关键帧的时间节点都捕获下来，然后让动画驱动到对应时间节点捕获其对应的数据。获取到数据之后将其转化为对偶四元素以节省存储空间，最后创建Float类型的Texture即可。

最后在VertexShader中采样出数据，然后进行蒙皮即可。首先将UniformBuffer更换为Sample2D:
```glsl
layout (binding = 1) uniform sampler2D animMap;
```

然后从Texture中采样出动画数据:
```glsl
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

```

最后按照之前的方式驱动起来即可。

```glsl
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

	if (paramData.animIndex.w > 0)
	{
		// skin info
		ivec4 skinIndex   = UnPackUInt32To4Byte(uint(inSkinPack.x));
		ivec2 skinWeight0 = UnPackUInt32To2Short(uint(inSkinPack.y));
		ivec2 skinWeight1 = UnPackUInt32To2Short(uint(inSkinPack.z));
		vec4  skinWeight  = vec4(skinWeight0 / 65535.0, skinWeight1 / 65535.0);

		mat2x4 dualQuat = CalcDualQuat(skinIndex, skinWeight, int(paramData.animIndex.z));

		position = vec4(DualQuatTransformPosition(dualQuat, inPosition.xyz), 1.0);
		normal   = DualQuatTransformVector(dualQuat, inNormal);
	}
	else
	{
		position = vec4(inPosition, 1.0);
		normal   = inNormal;
	}

	// 转换法线
	mat3 normalMatrix = transpose(inverse(mat3(paramData.modelMatrix)));
	normal = normalize(normalMatrix * normal);

	outUV     = inUV0;
	outNormal = normal;
	
	gl_Position = paramData.projectionMatrix * paramData.viewMatrix * paramData.modelMatrix * position;
}
```

整个流程其实也是非常简单的，不同之处是不在CPU里面计算每个时刻的`Pose`数据了，而是直接采样提前计算好的`Pose`数据。这种优化对于大量重复角色的场景下是非常有效的。下一篇文章就会讲解如何扩大这种优势。
