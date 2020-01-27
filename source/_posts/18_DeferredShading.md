---
title: 18_DeferredShading
date: 2019-09-9 23:15:23
tags:
- Vulkan
- 3D
- DeferredShading
- Tutorial
categories:
- Vulkan
---

[18_DeferredShading](https://github.com/BobLChen/VulkanDemos/tree/master/examples/18_DeferredShading)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

基于上一个Demo的知识，在这个Demo里面简单的进阶一下，实现一个简易的延迟渲染效果。

<!-- more -->

![18_DeferredShading](https://raw.githubusercontent.com/BobLChen/VulkanDemos/master/preview/18_DeferredShading.jpg)

## 18_DeferredShading

### 输出位置数据
在上一节Demo里面，我们输出了Color、Depth、Normal数据，在这一节Demo里面增加一项位置数据的输出。
采用VK_FORMAT_R16G16B16A16_SFLOAT格式存储即可，如果位置数据范围非常大，可以考虑用32位浮点数来进行存储。
```c++
for (int32 i = 0; i < m_AttachsPosition.size(); ++i)
{
    m_AttachsPosition[i] = vk_demo::DVKTexture::CreateAttachment(
        m_VulkanDevice,
        VK_FORMAT_R16G16B16A16_SFLOAT,
        VK_IMAGE_ASPECT_COLOR_BIT,
        fwidth, fheight,
        VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_INPUT_ATTACHMENT_BIT
    );
}
```

### 准备Light数据
既然是简易的延迟渲染，那就需要准备一些Light的数据，在这个Demo里面我只准备了简单的点光数据。然后将点光的数据存储在UniformBuffer里面，传递给Shader。
Light的数据结构如下：
```c++
struct PointLight
{
    Vector4 position;
    Vector3 color;
    float	radius;
};

struct LightSpawnBlock
{
    Vector3 position[NUM_LIGHTS];
    Vector3 direction[NUM_LIGHTS];
    float speed[NUM_LIGHTS];
};

struct LightDataBlock
{
    PointLight lights[NUM_LIGHTS];
};
```

定义好数据结构之后，在`CreateUniformBuffers`函数里面准备一下Light的数据。其中Light的位置根据场景大小进行随机，Light的颜色则是完全的随机，Light的范围从50-200。为了让Light动起来，还额外存储了Light的初始位置、方向、速度等数据。
```c++
// light datas
for (int32 i = 0; i < NUM_LIGHTS; ++i)
{
    m_LightDatas.lights[i].position.x = MMath::RandRange(bounds.min.x, bounds.max.x);
    m_LightDatas.lights[i].position.y = MMath::RandRange(bounds.min.y, bounds.max.y);
    m_LightDatas.lights[i].position.z = MMath::RandRange(bounds.min.z, bounds.max.z);

    m_LightDatas.lights[i].color.x = MMath::RandRange(0.0f, 1.0f);
    m_LightDatas.lights[i].color.y = MMath::RandRange(0.0f, 1.0f);
    m_LightDatas.lights[i].color.z = MMath::RandRange(0.0f, 1.0f);

    m_LightDatas.lights[i].radius = MMath::RandRange(50.0f, 200.0f);

    m_LightInfos.position[i]  = m_LightDatas.lights[i].position;
    m_LightInfos.direction[i] = m_LightInfos.position[i];
    m_LightInfos.direction[i].Normalize();
    m_LightInfos.speed[i] = 1.0f + MMath::RandRange(0.0f, 5.0f);
}
m_LightBuffer = vk_demo::DVKBuffer::CreateBuffer(
    m_VulkanDevice,
    VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT,
    VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
    sizeof(LightDataBlock),
    &(m_LightDatas)
);
m_LightBuffer->Map();
```

### 更新Light

准备好了Light的数据之后，即可在Draw之前更新Light的数据，让它动起来。为了让运动的速度不受到帧率的影响，我们需要用时间作为参数来调节Light的最终坐标。
在UpdateUniform函数里面增加更新Light的代码，如下所示：
```c++
void UpdateUniform(float time, float delta)
{
    m_ViewProjData.view = m_ViewCamera.GetView();
    m_ViewProjData.projection = m_ViewCamera.GetProjection();
    m_ViewProjBuffer->CopyFrom(&m_ViewProjData, sizeof(ViewProjectionBlock));
    // 更新Light
    for (int32 i = 0; i < NUM_LIGHTS; ++i)
    {
        float bias = MMath::Sin(time * m_LightInfos.speed[i]) / 5.0f;
        m_LightDatas.lights[i].position.x = m_LightInfos.position[i].x + bias * m_LightInfos.direction[i].x * 500.0f;
        m_LightDatas.lights[i].position.y = m_LightInfos.position[i].y + bias * m_LightInfos.direction[i].y * 500.0f;
        m_LightDatas.lights[i].position.z = m_LightInfos.position[i].z + bias * m_LightInfos.direction[i].z * 500.0f;
    }
    m_LightBuffer->CopyFrom(&m_LightDatas, sizeof(LightDataBlock));
}
```

### 准备Pipeline相关的数据

这里我就不在赘述，这个步骤在之前已经设置了很多次，这里只是简单提一下。因为增加了位置的附件数据，那么在输出的地方以及使用的地方都需要增加与之相关的描述或资源。

### Shader

由于增加了位置数据的输出，就需要修改Shader以输出相应的数据，这个过程跟上一个Demo一样，只是额外多了一个位置的数据。既然上个Demo都能输出法线的数据，那么依样画葫芦输出位置数据不难。

其实重点的地方在于后处理`Fragment Shader`，需要在Fragment shader里面进行光照的操作。先来看看fragment shader代码：
```glsl
#version 450

layout (input_attachment_index = 0, set = 0, binding = 0) uniform subpassInput inputColor;
layout (input_attachment_index = 1, set = 0, binding = 1) uniform subpassInput inputNormal;
layout (input_attachment_index = 2, set = 0, binding = 2) uniform subpassInput inputPosition;
layout (input_attachment_index = 3, set = 0, binding = 3) uniform subpassInput inputDepth;

struct PointLight {
	vec4 position;
	vec4 colorAndRadius;
};

#define NUM_LIGHTS 64

layout (binding = 4) uniform ParamBlock
{
	PointLight lights[NUM_LIGHTS];
} lightDatas;

layout (location = 0) in vec2 inUV0;

layout (location = 0) out vec4 outFragColor;

float DoAttenuation(float range, float d)
{
    return 1.0 - smoothstep(range * 0.75, range, d);
}

void main() 
{
	vec4 albedo   = subpassLoad(inputColor);
	vec4 normal   = subpassLoad(inputNormal);
	vec4 position = subpassLoad(inputPosition);
	
	normal.xyz    = normalize(normal.xyz);
	
	vec4 ambient  = vec4(0.20);
	
	outFragColor  = vec4(0.0) + ambient;
	for (int i = 0; i < NUM_LIGHTS; ++i)
	{
		vec3 lightDir = lightDatas.lights[i].position.xyz - position.xyz;
		float dist    = length(lightDir);
		float atten   = DoAttenuation(lightDatas.lights[i].colorAndRadius.w, dist);
		float ndotl   = max(0.0, dot(normal.xyz, normalize(lightDir)));
		vec3 diffuse  = lightDatas.lights[i].colorAndRadius.xyz * albedo.xyz * ndotl * atten;

		outFragColor.xyz += diffuse;
	}
}

```

在Fragment shader里面，写了一个`for`循环进行逐Light计算光照，根据Light的位置与顶点的位置可以计算出Light与顶点的数据，根据Light的`radius`属性可以计算出Light的衰减。其它的按照常规的方式计算diffuse颜色即可，然后应用衰减就可以得到一个简易的`Point Light`的形态。最终将所有灯光的颜色累加起来，得到了最终的效果，也就是预览图的效果。