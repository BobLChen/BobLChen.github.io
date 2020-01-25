---
title: 7_UniformBuffer
date: 2019-08-02 00:09:52
tags:
- Vulkan
- 物理设备
- 3D
- Demo
categories:
- Vulkan
---

[Github项目](https://github.com/BobLChen/VulkanDemos)

[Demo源码地址](https://github.com/BobLChen/VulkanDemos/tree/master/examples/7_UniformBuffer)

在之前的几个Demo里面，我们都是在为后续的Demo做准备，包括封装CommandBuffer、封装Buffer、集成ImGUI等等。做了这么久的准备工作，暂时不想再进行相关的封装工作，现在迫切的需要一个新的Demo来验证之前的封装是否合理。在这个Demo里面，我们将传递一些参数给Shader，通过控制不同的参数来达到一些需要的效果。

<!-- more -->

![Preview](https://raw.githubusercontent.com/BobLChen/VulkanTutorials/master/preview/7_UniformBuffer.jpg)

### [Pre-Integrated Skin Shading](https://zhuanlan.zhihu.com/p/56052015)

皮肤材质是大家都比较关心，希望解决的一个问题。常规的模拟方法是需要大量的计算，然后才能得到一个比较近似的结果。然后就出现了众多大佬经过多年的潜心研究，发明了一种计算量少但是效果还不错的方法。主要的原理还请参考这篇文章[Pre-Integrated Skin Shading](https://zhuanlan.zhihu.com/p/56052015)。在这篇文章里面，大佬将一些计算预先计算好，然后存储到一张贴图里面，然后在Shader里面通过这张查询图+曲率贴图以低成本的方式实现一个不成的效果。

在这里不细说它是如何实现的，我也不在这个Demo里面实现这个皮肤效果。另辟蹊径，在Shader里面通过函数来简单的拟合这张`LUT`查询图。当然只是拟合，并不能保证一模一样，但是也不要认为这个没有任何作用。1、首先节省了一张贴图，一张贴图无论是内存显存亦或者寄存器的占用，这些资源都是宝贵的；2、其次可以通过参数调节，来深层次的丰富效果。需要明确的是，有时候在做一些效果的时候，其实并不需要保证物理正确，反而只要保证`看起来非常不错`即可。

### 创建UniformBuffer

Shader参数一般是通过UniformBuffer来进行传递，由于UniformBuffer可变，因此一般将UniformBuffer放置到`Host`端对GPU可见。虽然访问速度慢了一点儿，但是更新速度变快了。在之前的Demo里面封装过Buffer，因此UniformBuffer的创建变得极为简单。

```c++
m_Params.omega   = 0.25 * PI;
m_Params.k       = 10;
m_Params.cutoff  = 0.57;
m_Params.padding = 0.0f;
m_ParamsBuffer = vk_demo::DVKBuffer::CreateBuffer(
    m_VulkanDevice,
    VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT,
    VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
    sizeof(UBOParams),
    &m_Params
);
m_ParamsBuffer->Map();

```
需要注意的是`VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT`表明该Buffer是用于`UniformBuffer`,而`VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,`则表示Buffer没有创建到GPU的存储之上。

### 设置VkDescriptorPoolSize

从之前的Demo中可以发现，`DescriptorSet`是用来关联资源到Shader。而`DescriptorSet`的创建是通过Pool来分配的。现在多了一个UniformBuffer，那么也需要扩充Pool的容量。

```c++
void CreateDescriptorPool()
{
    VkDescriptorPoolSize poolSize = {};
    poolSize.type            = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    poolSize.descriptorCount = 2;
    
    VkDescriptorPoolCreateInfo descriptorPoolInfo;
    ZeroVulkanStruct(descriptorPoolInfo, VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO);
    descriptorPoolInfo.poolSizeCount = 1;
    descriptorPoolInfo.pPoolSizes    = &poolSize;
    descriptorPoolInfo.maxSets       = 1;
    VERIFYVULKANRESULT(vkCreateDescriptorPool(m_Device, &descriptorPoolInfo, VULKAN_CPU_ALLOCATOR, &m_DescriptorPool));
}
```

在这里指定`PoolSize`的类型为`VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER`，它的数量有两个(1个ModelViewProject矩阵+1个参数)。设置好Size之后我们即可创建出对应的Pool。

### 创建DescriptorSetLayout

之前提到`DescriptorSetLayout`用来描述Shader的资源布局，多个`DescriptorSetLayout`构成一个`PipelineLayout`，真正使用的是`PipelineLayout`。在本Demo里面，使用到的资源有两个，`DescriptorSetLayout`只需要一个即可。注意在Shader里面的参数如下：

```GLSL
layout (set = 0, binding = 1) uniform UBOParam 
{
	float omega;
	float k;
	float cutoff;
	float padding;
} params;
```

注意观察`layout (set = 0, binding = 1)`，其中指定了`set`为0，`binding`为1。`set`就与`DescriptorSetLayout`对应。`binding`指的是在这个`set`里面的绑定位置。在创建`DescriptorSetLayout`的时候就需要跟Shader里面的参数对应起来。例如在VertexShader以及FragmentShader里面的参数如下：

```GLSL
// vertex shader
layout (set = 0, binding = 0) uniform UBO 
{
	mat4 modelMatrix;
	mat4 viewMatrix;
	mat4 projectionMatrix;
} uboMVP;
// fragment shader
layout (set = 0, binding = 1) uniform UBOParam 
{
	float omega;
	float k;
	float cutoff;
	float padding;
} params;
```

在创建`DescriptorSetLayout`时需要配置相应的信息。如下所示：

```c++
std::vector<VkDescriptorSetLayoutBinding> layoutBindings(2);
layoutBindings[0].binding 			 = 0;
layoutBindings[0].descriptorType     = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
layoutBindings[0].descriptorCount    = 1;
layoutBindings[0].stageFlags 		 = VK_SHADER_STAGE_VERTEX_BIT;
layoutBindings[0].pImmutableSamplers = nullptr;

layoutBindings[1].binding            = 1;
layoutBindings[1].descriptorType     = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
layoutBindings[1].descriptorCount    = 1;
layoutBindings[1].stageFlags         = VK_SHADER_STAGE_FRAGMENT_BIT;
layoutBindings[1].pImmutableSamplers = nullptr;

VkDescriptorSetLayoutCreateInfo descSetLayoutInfo;
ZeroVulkanStruct(descSetLayoutInfo, VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO);
descSetLayoutInfo.bindingCount = 2;
descSetLayoutInfo.pBindings    = layoutBindings.data();
VERIFYVULKANRESULT(vkCreateDescriptorSetLayout(m_Device, &descSetLayoutInfo, VULKAN_CPU_ALLOCATOR, &m_DescriptorSetLayout));
```
DescriptorSetLayout创建完成之后，我们即可创建出对应的`PipelineLayout`。
```c++
VkPipelineLayoutCreateInfo pipeLayoutInfo;
ZeroVulkanStruct(pipeLayoutInfo, VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO);
pipeLayoutInfo.setLayoutCount = 1;
pipeLayoutInfo.pSetLayouts    = &m_DescriptorSetLayout;
VERIFYVULKANRESULT(vkCreatePipelineLayout(m_Device, &pipeLayoutInfo, VULKAN_CPU_ALLOCATOR, &m_PipelineLayout));
```

### 创建DescriptorSet

DescriptorSet用于关联资源信息，需要注意的是，一个`DescriptorSet`对应一个`DescriptorSetLayout`。代码如下：

```c++
VkDescriptorSetAllocateInfo allocInfo;
ZeroVulkanStruct(allocInfo, VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO);
allocInfo.descriptorPool     = m_DescriptorPool;
allocInfo.descriptorSetCount = 1;
allocInfo.pSetLayouts        = &m_DescriptorSetLayout;
VERIFYVULKANRESULT(vkAllocateDescriptorSets(m_Device, &allocInfo, &m_DescriptorSet));
```

如果有多个`DescriptorSetLayout`，就需要创建对应的多个`DescriptorSet`。虽然这多个`DescriptorSetLayout`创建出了`PipelineLayout`，但是任然需要使用`DescriptorSetLayout`信息来创建出`DescriptorSet`。创建完成之后，即可将资源关联进去：

```c++
std::vector<VkWriteDescriptorSet> writeDescriptorSets(2);
ZeroVulkanStruct(writeDescriptorSets[0], VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET);
writeDescriptorSets[0].dstSet          = m_DescriptorSet;
writeDescriptorSets[0].descriptorCount = 1;
writeDescriptorSets[0].descriptorType  = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
writeDescriptorSets[0].pBufferInfo     = &(m_MVPBuffer->descriptor);
writeDescriptorSets[0].dstBinding      = 0;

ZeroVulkanStruct(writeDescriptorSets[1], VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET);
writeDescriptorSets[1].dstSet          = m_DescriptorSet;
writeDescriptorSets[1].descriptorCount = 1;
writeDescriptorSets[1].descriptorType  = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
writeDescriptorSets[1].pBufferInfo     = &(m_ParamsBuffer->descriptor);
writeDescriptorSets[1].dstBinding      = 1;

vkUpdateDescriptorSets(m_Device, 2, writeDescriptorSets.data(), 0, nullptr);
```

例如`Binding=0`关联了VertexShader的`MVP`矩阵数据；`Binding=1`关联了FragmentShader的`Param`数据。

### 设置DescriptorSet

最后在录制命令的时候，设置好`DescriptorSet`即可。

```c++
VERIFYVULKANRESULT(vkBeginCommandBuffer(m_CommandBuffers[i], &cmdBeginInfo));
            
vkCmdBeginRenderPass(m_CommandBuffers[i], &renderPassBeginInfo, VK_SUBPASS_CONTENTS_INLINE);
vkCmdSetViewport(m_CommandBuffers[i], 0, 1, &viewport);
vkCmdSetScissor(m_CommandBuffers[i], 0, 1, &scissor);

vkCmdBindDescriptorSets(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_PipelineLayout, 0, 1, &m_DescriptorSet, 0, nullptr);
vkCmdBindPipeline(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_Pipeline);
vkCmdBindVertexBuffers(m_CommandBuffers[i], 0, 1, &(m_VertexBuffer->buffer), offsets);
vkCmdBindIndexBuffer(m_CommandBuffers[i], m_IndexBuffer->buffer, 0, VK_INDEX_TYPE_UINT16);
vkCmdDrawIndexed(m_CommandBuffers[i], m_IndicesCount, 1, 0, 0, 0);

m_GUI->BindDrawCmd(m_CommandBuffers[i], m_RenderPass);

vkCmdEndRenderPass(m_CommandBuffers[i]);

VERIFYVULKANRESULT(vkEndCommandBuffer(m_CommandBuffers[i]));
```

最终效果就如封面图所示，我们可以通过调节参数来近似的拟合那张贴图。

### VertexShader
```glsl
#version 450

layout (location = 0) in vec3 inPosition;
layout (location = 1) in vec2 inUV0;

layout (set = 0, binding = 0) uniform UBO 
{
	mat4 modelMatrix;
	mat4 viewMatrix;
	mat4 projectionMatrix;
} uboMVP;

layout (location = 0) out vec2 outUV0;

out gl_PerVertex 
{
    vec4 gl_Position;   
};

void main() 
{
	outUV0 = inUV0;
	gl_Position = uboMVP.projectionMatrix * uboMVP.viewMatrix * uboMVP.modelMatrix * vec4(inPosition.xyz, 1.0);
}

```
### Fragment Shader
```glsl
#version 450

layout (set = 0, binding = 1) uniform UBOParam 
{
	float omega;
	float k;
	float cutoff;
	float padding;
} params;

layout (location = 0) in vec2 inUV0;

layout (location = 0) out vec4 outFragColor;

float PreCompute(float omega, float k, float CutOff, float u, float v)
{
	float left      = 0.36 * cos(omega * v) + 0.1;
	float inArea1   = max(sign(left - u), 0.0);
	float inArea3   = max(sign(k * (u - CutOff) - v), 0.0);
	float inArea2   = 1.0 - min(inArea1 + inArea3, 1.0);
	float value3    = 1;//0.5*u+0.38;
	float right     = v / k + CutOff;
	float amplitude = 0.5;//0.5 * (0.5 * right + 0.38);
	float omeg      = 3.1415926 / (right - left);
	float value2    = amplitude + amplitude * sin(omeg * (u - left - 0.5 * (right - left)));
	return (inArea2 * value2 + inArea3 * value3);
}

void main() 
{
	float bias = PreCompute(params.omega, params.k, params.cutoff, inUV0.x, inUV0.y);
	outFragColor.rgba = vec4(bias, bias, bias, 1.0);
}

```