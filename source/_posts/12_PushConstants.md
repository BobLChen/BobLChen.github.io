---
title: 12_PushConstants
date: 2019-08-12 23:39:07
tags:
- Vulkan
- 物理设备
- 3D
- Demo
categories:
- Vulkan
---

[Texture](https://github.com/BobLChen/VulkanDemos/tree/master/examples/12_PushConstants)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

上一个Demo里面我们了解到如何将贴图与模型结合起来显示，再通过一些复杂的Shader。我们已经可以实现许多特殊的效果，但是存在一个问题，不知道大家有没有想过，那就是如何显示多个模型，并且这些模型在位于不同的坐标。在之前的Demo中，我们处理的都是一个模型，一个模型的显示非常方便，但是多个模型的显示可能就会有一种束手束脚的感觉。

<!-- more -->

![PushConstants](https://raw.githubusercontent.com/BobLChen/VulkanTutorials/master/preview/12_PushConstants.jpg)

结合之前我们学到UniformBuffer，我们其实也可以实现这个功能，那就是为每一个模型都创建一个UniformBuffer用来存放**MVP**矩阵数据。这种方法也不是不可以，但是有一些缺点，比如我们必须为每一个**Backbuffer**都创建一个UniformBuffer。注意这里我指的是每一个**Backbuffer**，之前的Demo里面，我们只创建了一个UniformBuffer来供两个或者三个**Backbuffer**使用，这种方式其实是不对的。例如：在第一帧画面的时候，坐标在原点；在第二帧画面的时候坐标在(1, 0, 0)点；但是我们的UniformBuffer只有一个，只能存一份数据。还有一些其它缺陷比如会造成内存和显存不连续等等。

## PushConstants

PushConstants可以解决一部分上述的问题，但是它也有一些缺陷。例如容量一般只有256字节，存储的数据量较少。虽然有缺陷，但是目前我们可以使用这个技术来实现我们的需求。

### Shader
首先我们需要修改我们的**Shader**，在Vertex Shader里面指定数据结构为PushConstants。
```glsl
#version 450

layout (location = 0) in vec3 inPosition;
layout (location = 1) in vec2 inUV0;
layout (location = 2) in vec3 inNormal;

layout (binding = 0) uniform UBO 
{
	mat4 viewMatrix;
	mat4 projectionMatrix;
} uboMVP;

layout(push_constant) uniform PushConsts {
    layout(offset = 0) mat4 modelMatrix;
} pushConsts;

layout (location = 0) out vec2 outUV0;
layout (location = 1) out vec3 outNormal;

out gl_PerVertex 
{
    vec4 gl_Position;   
};

void main() 
{
	mat3 normalMatrix = transpose(inverse(mat3(pushConsts.modelMatrix)));
	vec3 normal  = normalize(normalMatrix * inNormal);

	outUV0 = inUV0;
	outNormal = normal;

	gl_Position = uboMVP.projectionMatrix * uboMVP.viewMatrix * pushConsts.modelMatrix * vec4(inPosition.xyz, 1.0);
}
```

即将`Model`矩阵从`uboMVP`拆出来。

```glsl
layout(push_constant) uniform PushConsts {
    layout(offset = 0) mat4 modelMatrix;
} pushConsts;
```

在进行对顶点投影映射转换的时候，变为如下：

```glsl
gl_Position = uboMVP.projectionMatrix * uboMVP.viewMatrix * pushConsts.modelMatrix * vec4(inPosition.xyz, 1.0);
```

其它的没有过多变化，修改的不多，只是修改了Shader的数据类型。

### DescriptorSetLayout

既然我们的Shader的数据结构发生了变化，那么我们的**DescriptorSetLayout**也要随着进行修改以匹配**Shader**的数据结构。
```c++
void CreateDescriptorSetLayout()
{
    VkDescriptorSetLayoutBinding layoutBindings[1] = { };
    layoutBindings[0].binding 			 = 0;
    layoutBindings[0].descriptorType     = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    layoutBindings[0].descriptorCount    = 1;
    layoutBindings[0].stageFlags 		 = VK_SHADER_STAGE_VERTEX_BIT;
    layoutBindings[0].pImmutableSamplers = nullptr;

    VkDescriptorSetLayoutCreateInfo descSetLayoutInfo;
    ZeroVulkanStruct(descSetLayoutInfo, VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO);
    descSetLayoutInfo.bindingCount = 1;
    descSetLayoutInfo.pBindings    = layoutBindings;
    VERIFYVULKANRESULT(vkCreateDescriptorSetLayout(m_Device, &descSetLayoutInfo, VULKAN_CPU_ALLOCATOR, &m_DescriptorSetLayout));
    
    VkPushConstantRange pushConstantRange = {};
    pushConstantRange.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;
    pushConstantRange.offset     = 0;
    pushConstantRange.size       = sizeof(ModelPushConstantBlock);
    
    VkPipelineLayoutCreateInfo pipeLayoutInfo;
    ZeroVulkanStruct(pipeLayoutInfo, VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO);
    pipeLayoutInfo.setLayoutCount = 1;
    pipeLayoutInfo.pSetLayouts    = &m_DescriptorSetLayout;
    pipeLayoutInfo.pushConstantRangeCount = 1;
    pipeLayoutInfo.pPushConstantRanges    = &pushConstantRange;
    VERIFYVULKANRESULT(vkCreatePipelineLayout(m_Device, &pipeLayoutInfo, VULKAN_CPU_ALLOCATOR, &m_PipelineLayout));
}
```
多了一个关于PushConstantRange的描述。
```c++
VkPushConstantRange pushConstantRange = {};
pushConstantRange.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;
pushConstantRange.offset     = 0;
pushConstantRange.size       = sizeof(ModelPushConstantBlock);
```

### Draw

我们只是修改了Shader的数据结构，随后我们也相应正确的修改了Layout，其它的地方不需要修改。我们只需要在录制绘制命令的时候，在录制Draw命令之前将相应的数据给到即可。我们可以通过`vkCmdPushConstants`来完成。
```c++
vkCmdPushConstants(
    m_CommandBuffers[i], m_PipelineLayout, VK_SHADER_STAGE_VERTEX_BIT,
    0, sizeof(ModelPushConstantBlock), &globalMatrix
);
```
第一个参数是CommandBuffer，第二个参数指定我们的PipelineLayout，第三个参数说明我们的PushConstants作用到哪一个**Stage**，最后几个参数则指定我们需要关联的具体数据。

完整代码如下：
```c++
void SetupCommandBuffers()
{
    VkCommandBufferBeginInfo cmdBeginInfo;
    ZeroVulkanStruct(cmdBeginInfo, VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO);

    VkClearValue clearValues[2];
    clearValues[0].color        = { {0.2f, 0.2f, 0.2f, 1.0f} };
    clearValues[1].depthStencil = { 1.0f, 0 };

    VkRenderPassBeginInfo renderPassBeginInfo;
    ZeroVulkanStruct(renderPassBeginInfo, VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO);
    renderPassBeginInfo.renderPass      = m_RenderPass;
    renderPassBeginInfo.clearValueCount = 2;
    renderPassBeginInfo.pClearValues    = clearValues;
    renderPassBeginInfo.renderArea.offset.x = 0;
    renderPassBeginInfo.renderArea.offset.y = 0;
    renderPassBeginInfo.renderArea.extent.width  = m_FrameWidth;
    renderPassBeginInfo.renderArea.extent.height = m_FrameHeight;
    
    for (int32 i = 0; i < m_CommandBuffers.size(); ++i)
    {
        renderPassBeginInfo.framebuffer = m_FrameBuffers[i];
        
        VERIFYVULKANRESULT(vkBeginCommandBuffer(m_CommandBuffers[i], &cmdBeginInfo));
        vkCmdBeginRenderPass(m_CommandBuffers[i], &renderPassBeginInfo, VK_SUBPASS_CONTENTS_INLINE);
        
        VkViewport viewport = {};
        viewport.x        = 0;
        viewport.y        = m_FrameHeight;
        viewport.width    = m_FrameWidth;
        viewport.height   = -(float)m_FrameHeight;    // flip y axis
        viewport.minDepth = 0.0f;
        viewport.maxDepth = 1.0f;
        
        VkRect2D scissor = {};
        scissor.extent.width  = m_FrameWidth;
        scissor.extent.height = m_FrameHeight;
        scissor.offset.x      = 0;
        scissor.offset.y      = 0;
        
        vkCmdSetViewport(m_CommandBuffers[i], 0, 1, &viewport);
        vkCmdSetScissor(m_CommandBuffers[i], 0, 1, &scissor);
        
        vkCmdBindPipeline(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_Pipeline->pipeline);
        vkCmdBindDescriptorSets(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_Pipeline->pipelineLayout, 0, 1, &m_DescriptorSet, 0, nullptr);
        
        for (int32 meshIndex = 0; meshIndex < m_Model->meshes.size(); ++meshIndex)
        {
            const Matrix4x4& globalMatrix = m_Model->meshes[meshIndex]->linkNode->GetGlobalMatrix();
            vkCmdPushConstants(
                m_CommandBuffers[i], m_PipelineLayout, VK_SHADER_STAGE_VERTEX_BIT,
                0, sizeof(ModelPushConstantBlock), &globalMatrix
            );
            m_Model->meshes[meshIndex]->BindDrawCmd(m_CommandBuffers[i]);
        }
        
        m_GUI->BindDrawCmd(m_CommandBuffers[i], m_RenderPass);

        vkCmdEndRenderPass(m_CommandBuffers[i]);
        VERIFYVULKANRESULT(vkEndCommandBuffer(m_CommandBuffers[i]));
    }
}
```

由于我们之前封装了很多功能，今天这个Demo到现在为此我们都只需要改动简单的几行代码即可，之前封装的优势已经开始逐步体现出来。最终效果就如封面图所示，大家也可以尝试用一些复杂的场景来体会Demo中的场景。如果大家使用的是一个比较标准的场景，那么当去掉Pushconstants这个功能之后，大家可能会发现所有的物体都处于原点位置，只有加入Pushconstants这个功能，物体才会正确的摆放。

