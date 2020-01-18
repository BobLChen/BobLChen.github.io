---
title: 17_InputAttachments
date: 2019-09-2 23:25:41
tags:
- Vulkan
- 物理设备
- 3D
- Demo
categories:
- Vulkan
---

[17_InputAttachments](https://github.com/BobLChen/VulkanDemos/tree/master/examples/17_InputAttachments)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

在这个Demo里面我们来了解如何使用InputAttachment(附件)。InputAttachment我们会在后面大量用到，特别是后期效果的处理流程中，离不开这个功能。如何使用附件是非常简单的，我们将这个Demo里面了解如何在一个Pass里面进行对附件的读取操作。

<!-- more -->

![17_InputAttachments](https://raw.githubusercontent.com/BobLChen/VulkanTutorials/master/preview/17_InputAttachments.jpg)

## 17_InputAttachments

在传统的流程里面，我们一般是使用多Pass来完成一些后期特效。但是对于**tile-based**的GPU架构来说，这个过程会造成性能损失，因为如果使用多Pass的流程，就需要被写入的对象完全写入完成之后才能切换Pass，然后完成后续的操作。这个流程对于移动平台来讲，带宽、拷贝速度等的压力极大，同时也完全没有利用上**tile-based**的优化。

Vulkan给我们提供了一个**Subpass**的概念，就是我们可以在同一个**Pass**里面读取之前写入到附件的数据。但是**Subpass**也不是说完美的，它也有一些限制：只能读取当前像素的数据。

只能读取当前像素的数据，这个可能比较难理解，其实简单的来讲，就是只能在**Subpass**里面读取之前写入的数据。假设我们的**FrameBuffer**的尺寸是**4x4**,我们就只能在Subpass里面(0, 0)位置处读取(0, 0)写入的数据，在(1, 1)处读取(1, 1)写入的数据，依次类推。根据这个特性，大家就可以发现它的劣势了，无法读取其它位置的像素数据包括邻接的数据，也就是一些需要周围像素参与计算的操作是无法完成的了。

大家有必要去了解一下**tile-based**的架构，我觉得**Subpass**跟这个相关性很大。具体情况查看这里[TileBasedArchitectures](https://www.seas.upenn.edu/~pcozzi/OpenGLInsights/OpenGLInsights-TileBasedArchitectures.pdf)

我们简单的来模拟一下**tile-based**流程，假如第一个处理的块是从(0, 0)到(1, 1)这个2x2的块。其它三个块还未开始渲染，这个时候已经可以走到我们的**Subpass**流程里面，在**Subpass**里面，我们已经可以在对应位置读取到之前的结果了，但是我们无法读取到其它位置的数据，因为其它的位置的数据压根都还未开始处理，这个其实也是造成上述的限制的原因。但是优势大家也可以看出，我们完全利用到了**tile-based**这个架构的优势，

## 创建InputAttachments

我们需要准备好附件，准备好了附件之后才能给framebuffer使用。在这个Demo里面，我们会把深度、颜色、法线的数据输出出来，为我们后面的简易延迟着色做准备。

创建InputAttachments很简单，它们其实就是Texture，只是格式稍微有点儿不一样。另外需要注意的是Texture的参数，需要指定Texture的用途以及功能。代码如下:
```c++
void CreateAttachments()
{
    auto swapChain  = GetVulkanRHI()->GetSwapChain();
    int32 fwidth    = swapChain->GetWidth();
    int32 fheight   = swapChain->GetHeight();
    int32 numBuffer = swapChain->GetBackBufferCount();
    
    m_AttachsColor.resize(numBuffer);
    m_AttachsNormal.resize(numBuffer);
    m_AttachsDepth.resize(numBuffer);
    
    for (int32 i = 0; i < m_AttachsColor.size(); ++i)
    {
        m_AttachsColor[i] = vk_demo::DVKTexture::CreateAttachment(
            m_VulkanDevice,
            PixelFormatToVkFormat(GetVulkanRHI()->GetPixelFormat(), false), 
            VK_IMAGE_ASPECT_COLOR_BIT,
            fwidth, fheight,
            VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_INPUT_ATTACHMENT_BIT
        );
    }
    
    for (int32 i = 0; i < m_AttachsNormal.size(); ++i)
    {
        m_AttachsNormal[i] = vk_demo::DVKTexture::CreateAttachment(
            m_VulkanDevice,
            VK_FORMAT_R8G8B8A8_UNORM,
            VK_IMAGE_ASPECT_COLOR_BIT,
            fwidth, fheight,
            VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_INPUT_ATTACHMENT_BIT
        );
    }
    
    for (int32 i = 0; i < m_AttachsDepth.size(); ++i)
    {
        m_AttachsDepth[i] = vk_demo::DVKTexture::CreateAttachment(
            m_VulkanDevice,
            PixelFormatToVkFormat(m_DepthFormat, false), 
            VK_IMAGE_ASPECT_DEPTH_BIT,
            fwidth, fheight,
            VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT | VK_IMAGE_USAGE_INPUT_ATTACHMENT_BIT
        );
    }
}
```

## 创建FrameBuffer
附件最终会被配置到FrameBuffer，我们在创建FrameBuffer的时候就需要指定出绑定关系。代码如下：
```c++
void CreateFrameBuffers() override
{
    DestroyFrameBuffers();
    
    int32 fwidth    = GetVulkanRHI()->GetSwapChain()->GetWidth();
    int32 fheight   = GetVulkanRHI()->GetSwapChain()->GetHeight();
    VkDevice device = GetVulkanRHI()->GetDevice()->GetInstanceHandle();

    VkImageView attachments[4];

    VkFramebufferCreateInfo frameBufferCreateInfo;
    ZeroVulkanStruct(frameBufferCreateInfo, VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO);
    frameBufferCreateInfo.renderPass      = m_RenderPass;
    frameBufferCreateInfo.attachmentCount = 4;
    frameBufferCreateInfo.pAttachments    = attachments;
    frameBufferCreateInfo.width			  = fwidth;
    frameBufferCreateInfo.height		  = fheight;
    frameBufferCreateInfo.layers		  = 1;

    const std::vector<VkImageView>& backbufferViews = GetVulkanRHI()->GetBackbufferViews();

    m_FrameBuffers.resize(backbufferViews.size());
    for (uint32 i = 0; i < m_FrameBuffers.size(); ++i) {
        attachments[0] = backbufferViews[i];
        attachments[1] = m_AttachsColor[i]->imageView;
        attachments[2] = m_AttachsNormal[i]->imageView;
        attachments[3] = m_AttachsDepth[i]->imageView;
        VERIFYVULKANRESULT(vkCreateFramebuffer(device, &frameBufferCreateInfo, VULKAN_CPU_ALLOCATOR, &m_FrameBuffers[i]));
    }
}
```

## 创建RenderPass
RenderPass的创建就稍微有点儿繁琐了，RenderPass需要非常具体的描述出每个附件的信息以及关系，同时也要指定出Subpass的关系以及信息。我们先来看代码：
```c++
void CreateRenderPass() override
{
    DestoryRenderPass();
    DestroyAttachments();
    CreateAttachments();

    VkDevice device = GetVulkanRHI()->GetDevice()->GetInstanceHandle();
    PixelFormat pixelFormat = GetVulkanRHI()->GetPixelFormat();

    std::vector<VkAttachmentDescription> attachments(4);
    // swap chain attachment
    attachments[0].format		  = PixelFormatToVkFormat(pixelFormat, false);
    attachments[0].samples		  = m_SampleCount;
    attachments[0].loadOp		  = VK_ATTACHMENT_LOAD_OP_CLEAR;
    attachments[0].storeOp        = VK_ATTACHMENT_STORE_OP_STORE;
    attachments[0].stencilLoadOp  = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    attachments[0].stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    attachments[0].initialLayout  = VK_IMAGE_LAYOUT_UNDEFINED;
    attachments[0].finalLayout	  = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
    // color attachment
    attachments[1].format         = PixelFormatToVkFormat(pixelFormat, false);
    attachments[1].samples        = m_SampleCount;
    attachments[1].loadOp         = VK_ATTACHMENT_LOAD_OP_CLEAR;
    attachments[1].storeOp        = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    attachments[1].stencilLoadOp  = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    attachments[1].stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    attachments[1].initialLayout  = VK_IMAGE_LAYOUT_UNDEFINED;
    attachments[1].finalLayout    = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
    // normal attachment
    attachments[2].format         = VK_FORMAT_R8G8B8A8_UNORM;
    attachments[2].samples        = m_SampleCount;
    attachments[2].loadOp         = VK_ATTACHMENT_LOAD_OP_CLEAR;
    attachments[2].storeOp        = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    attachments[2].stencilLoadOp  = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    attachments[2].stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    attachments[2].initialLayout  = VK_IMAGE_LAYOUT_UNDEFINED;
    attachments[2].finalLayout    = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
    // depth stencil attachment
    attachments[3].format         = PixelFormatToVkFormat(m_DepthFormat, false);
    attachments[3].samples        = m_SampleCount;
    attachments[3].loadOp         = VK_ATTACHMENT_LOAD_OP_CLEAR;
    attachments[3].storeOp        = VK_ATTACHMENT_STORE_OP_STORE;
    attachments[3].stencilLoadOp  = VK_ATTACHMENT_LOAD_OP_CLEAR;
    attachments[3].stencilStoreOp = VK_ATTACHMENT_STORE_OP_STORE;
    attachments[3].initialLayout  = VK_IMAGE_LAYOUT_UNDEFINED;
    attachments[3].finalLayout    = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
    
    VkAttachmentReference colorReferences[2];
    colorReferences[0].attachment = 1;
    colorReferences[0].layout     = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
    colorReferences[1].attachment = 2;
    colorReferences[1].layout     = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
    
    VkAttachmentReference swapReference = { };
    swapReference.attachment = 0;
    swapReference.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;

    VkAttachmentReference depthReference = { };
    depthReference.attachment = 3;
    depthReference.layout     = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
    
    VkAttachmentReference inputReferences[3];
    inputReferences[0].attachment = 1;
    inputReferences[0].layout     = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
    inputReferences[1].attachment = 2;
    inputReferences[1].layout     = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
    inputReferences[2].attachment = 3;
    inputReferences[2].layout     = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
    
    std::vector<VkSubpassDescription> subpassDescriptions(2);
    subpassDescriptions[0].pipelineBindPoint       = VK_PIPELINE_BIND_POINT_GRAPHICS;
    subpassDescriptions[0].colorAttachmentCount    = 2;
    subpassDescriptions[0].pColorAttachments       = colorReferences;
    subpassDescriptions[0].pDepthStencilAttachment = &depthReference;
    
    subpassDescriptions[1].pipelineBindPoint       = VK_PIPELINE_BIND_POINT_GRAPHICS;
    subpassDescriptions[1].colorAttachmentCount    = 1;
    subpassDescriptions[1].pColorAttachments       = &swapReference;
    subpassDescriptions[1].inputAttachmentCount    = 3;
    subpassDescriptions[1].pInputAttachments       = inputReferences;
    
    std::vector<VkSubpassDependency> dependencies(3);
    dependencies[0].srcSubpass      = VK_SUBPASS_EXTERNAL;
    dependencies[0].dstSubpass      = 0;
    dependencies[0].srcStageMask    = VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT;
    dependencies[0].dstStageMask    = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
    dependencies[0].srcAccessMask   = VK_ACCESS_MEMORY_READ_BIT;
    dependencies[0].dstAccessMask   = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
    dependencies[0].dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;
    
    dependencies[1].srcSubpass      = 0;
    dependencies[1].dstSubpass      = 1;
    dependencies[1].srcStageMask    = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
    dependencies[1].dstStageMask    = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
    dependencies[1].srcAccessMask   = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
    dependencies[1].dstAccessMask   = VK_ACCESS_SHADER_READ_BIT;
    dependencies[1].dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;

    dependencies[2].srcSubpass      = 1;
    dependencies[2].dstSubpass      = VK_SUBPASS_EXTERNAL;
    dependencies[2].srcStageMask    = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
    dependencies[2].dstStageMask    = VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT;
    dependencies[2].srcAccessMask   = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
    dependencies[2].dstAccessMask   = VK_ACCESS_MEMORY_READ_BIT;
    dependencies[2].dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;
    
    VkRenderPassCreateInfo renderPassInfo;
    ZeroVulkanStruct(renderPassInfo, VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO);
    renderPassInfo.attachmentCount = attachments.size();
    renderPassInfo.pAttachments    = attachments.data();
    renderPassInfo.subpassCount    = subpassDescriptions.size();
    renderPassInfo.pSubpasses      = subpassDescriptions.data();
    renderPassInfo.dependencyCount = dependencies.size();
    renderPassInfo.pDependencies   = dependencies.data();
    VERIFYVULKANRESULT(vkCreateRenderPass(device, &renderPassInfo, VULKAN_CPU_ALLOCATOR, &m_RenderPass));
}
```

### VkAttachmentDescription
描述了附件的信息
- format:附件的格式
- samples:MSAA数量
- loadOp:附件Load时的行为，注意这个对深度模板附件是无效的。
- storeOp:附件Store时的行为，对深度模板附件无效。
- stencilLoadOp:既然上面的操作对深度/模板无效，这个对它们才有效。
- stencilStoreOp:同上
- initialLayout:附件初始状态的布局
- finalLayout:附件最后的布局状态
上面布局状态大家需要着重注意，指定了布局状态，就Pass里面就会隐式的进行布局关系的转换。

### VkAttachmentReference
描述了附件的引用关系，我们在FrameBuffer里面指定了附件以及关系，但是在创建RenderPass的时候，RenderPass并不知道具体的对应关系，也不知道具体的功能作用。为了正确的把对应关系映射上，我们需要填充VkAttachmentReference信息，以描述出常规的Color附件和深度/模板附件以及它们的映射关系。
- attachment：这个是FrameBuffer里面的**pAttachments**索引。
- layout:指定了用途

### VkSubpassDescription
上述的VkAttachmentDescription和VkAttachmentReference只是填充了附件的信息以及它们的映射关系。但是并没有把它们组织到一起，而且由于有**Subpass**的关系，我们也没有为subpass指定信息。VkSubpassDescription其实就是把这些数据组织起来，给真正的Subpass使用。
- pipelineBindPoint:指定了Pipeline的类型
- colorAttachmentCount:指定了附件的数量
- pColorAttachments:指定了附件具体信息
- pDepthStencilAttachment:指定深度/模板附件的信息

### VkSubpassDependency
填充完了上述的信息之后，我们就可以开始指定Subpass的依赖信息。从字面意思就可以看出来它们的依赖行为，不在细述。
```c++
std::vector<VkSubpassDependency> dependencies(3);
dependencies[0].srcSubpass      = VK_SUBPASS_EXTERNAL;
dependencies[0].dstSubpass      = 0;
dependencies[0].srcStageMask    = VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT;
dependencies[0].dstStageMask    = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependencies[0].srcAccessMask   = VK_ACCESS_MEMORY_READ_BIT;
dependencies[0].dstAccessMask   = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
dependencies[0].dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;

dependencies[1].srcSubpass      = 0;
dependencies[1].dstSubpass      = 1;
dependencies[1].srcStageMask    = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependencies[1].dstStageMask    = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
dependencies[1].srcAccessMask   = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
dependencies[1].dstAccessMask   = VK_ACCESS_SHADER_READ_BIT;
dependencies[1].dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;

dependencies[2].srcSubpass      = 1;
dependencies[2].dstSubpass      = VK_SUBPASS_EXTERNAL;
dependencies[2].srcStageMask    = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependencies[2].dstStageMask    = VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT;
dependencies[2].srcAccessMask   = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
dependencies[2].dstAccessMask   = VK_ACCESS_MEMORY_READ_BIT;
dependencies[2].dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;
```

### 创建RenderPass
根据上述的信息，就可以创建出我们必要的RenderPass。
```c++
VkRenderPassCreateInfo renderPassInfo;
ZeroVulkanStruct(renderPassInfo, VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO);
renderPassInfo.attachmentCount = attachments.size();
renderPassInfo.pAttachments    = attachments.data();
renderPassInfo.subpassCount    = subpassDescriptions.size();
renderPassInfo.pSubpasses      = subpassDescriptions.data();
renderPassInfo.dependencyCount = dependencies.size();
renderPassInfo.pDependencies   = dependencies.data();
VERIFYVULKANRESULT(vkCreateRenderPass(device, &renderPassInfo, VULKAN_CPU_ALLOCATOR, &m_RenderPass));
```

## Pipeline
由于多了几个附件，Pipeline里面的光栅化的一些操作对附件也是有效的，例如混合模式。因此混合模式那里我们需要特殊指定一下：
```c++
vk_demo::DVKGfxPipelineInfo pipelineInfo0;
pipelineInfo0.shader = m_Shader0;
// 注意这句话
pipelineInfo0.colorAttachmentCount = 2;
m_Pipeline0 = vk_demo::DVKGfxPipeline::Create(
    m_VulkanDevice, 
    m_PipelineCache, 
    pipelineInfo0, 
    { 
        m_Model->GetInputBinding()
    }, 
    m_Model->GetInputAttributes(), 
    m_Shader0->pipelineLayout, 
    m_RenderPass
);
```

Subpass的Pipeline需要特殊指定一下subpass的索引。
```c++
vk_demo::DVKGfxPipelineInfo pipelineInfo1;
pipelineInfo1.depthStencilState.depthTestEnable   = VK_FALSE;
pipelineInfo1.depthStencilState.depthWriteEnable  = VK_FALSE;
pipelineInfo1.depthStencilState.stencilTestEnable = VK_FALSE;
pipelineInfo1.shader  = m_Shader1;
// 注意这句话
pipelineInfo1.subpass = 1;
m_Pipeline1 = vk_demo::DVKGfxPipeline::Create(
    m_VulkanDevice, 
    m_PipelineCache, 
    pipelineInfo1, 
    { 
        m_Quad->GetInputBinding()
    }, 
    m_Quad->GetInputAttributes(), 
    m_Shader1->pipelineLayout, 
    m_RenderPass
);
```

## 录制命令
其它的步骤其实没有任何变化，只是录制命令的时候我们需要额外指定一下。
第一个是Clear的操作，我们需要指定每一个附件Clear时的默认值:
```c++
VkClearValue clearValues[4];
clearValues[0].color        = { { 0.2f, 0.2f, 0.2f, 0.0f } };
clearValues[1].color        = { { 0.2f, 0.2f, 0.2f, 0.0f } };
clearValues[2].color        = { { 0.2f, 0.2f, 0.2f, 0.0f } };
clearValues[3].depthStencil = { 1.0f, 0 };
```

然后我们在录制命令的时候，就可以开始指定每个Pass需要完成的操作，这个Pass就是我们正常的绘制流程。
```c++
// pass0
{
    vkCmdBindPipeline(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_Pipeline0->pipeline);
    for (int32 meshIndex = 0; meshIndex < m_Model->meshes.size(); ++meshIndex) {
        uint32 offset = meshIndex * modelAlign;
        vkCmdBindDescriptorSets(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_Pipeline0->pipelineLayout, 0, m_DescriptorSet0->descriptorSets.size(), m_DescriptorSet0->descriptorSets.data(), 1, &offset);
        m_Model->meshes[meshIndex]->BindDrawCmd(m_CommandBuffers[i]);
    }
}
```

然后我们录制切换Subpass的命令。
```
vkCmdNextSubpass(m_CommandBuffers[i], VK_SUBPASS_CONTENTS_INLINE);
```

最后在Subpass里面读取附件的信息并显示。
```c++
// pass1
{
    vkCmdBindPipeline(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_Pipeline1->pipeline);
    vkCmdBindDescriptorSets(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_Pipeline1->pipelineLayout, 0, m_DescriptorSets[i]->descriptorSets.size(), m_DescriptorSets[i]->descriptorSets.data(), 0, nullptr);
    for (int32 meshIndex = 0; meshIndex < m_Quad->meshes.size(); ++meshIndex) {
        m_Quad->meshes[meshIndex]->BindDrawCmd(m_CommandBuffers[i]);
    }
}
```

另外需要注意的是UI的绘制，UI的绘制是在Subpass里面的完成的，因此UI的代码我有稍稍的调整适配。
```c++
// 注意这个1，就是我们的Subpass的索引，也跟前面创建Shader时里面的PipelineInfo的subpass一样。
m_GUI->BindDrawCmd(m_CommandBuffers[i], m_RenderPass, 1);
```

## Shader
我们来看一下Shader的代码，主要是fragment shader有些许变化，先来看**obj.frag**:
```glgl
#version 450

layout (location = 0) in vec3 inNormal;
layout (location = 1) in vec3 inPosition;

layout (location = 0) out vec4 outFragColor;
layout (location = 1) out vec4 outNormal;

void main() 
{
    vec3 normal = normalize(inNormal);
    float NDotL = clamp(dot(normal, vec3(0, 0, -1)), 0, 1.0);

    outNormal    = vec4(inNormal, 1.0);
    outFragColor = vec4(NDotL, NDotL, NDotL, 1.0);
}

```
我们指定了两个输出，
- `layout (location = 0) out vec4 outFragColor;`：是我们正常的颜色输出。
- `layout (location = 1) out vec4 outNormal;`：我们输出了法线的数据。
- 还有一个隐式的Depth/Stencil的输出不会在Shader中体现出来。

最后来看一下如何在Subpass里面读取附件的数据。
```glsl
#version 450

layout (input_attachment_index = 0, set = 0, binding = 0) uniform subpassInput inputColor;
layout (input_attachment_index = 1, set = 0, binding = 1) uniform subpassInput inputNormal;
layout (input_attachment_index = 2, set = 0, binding = 2) uniform subpassInput inputDepth;

layout (binding = 3) uniform ParamBlock
{
	int attachmentIndex;
	vec3 padding;
} param;

layout (location = 0) in vec2 inUV0;

layout (location = 0) out vec4 outFragColor;

void main() 
{
	if (param.attachmentIndex == 0) {
		outFragColor = subpassLoad(inputColor);
	} 
	else if (param.attachmentIndex == 1) {
		outFragColor = vec4(subpassLoad(inputDepth).r);
	} 
	else if (param.attachmentIndex == 2) {
		outFragColor = subpassLoad(inputNormal);
	} else {
		outFragColor = vec4(inUV0, 0.0, 1.0);
	}
}

```

`layout (input_attachment_index = 0, set = 0, binding = 0) uniform subpassInput inputColor;`我们需要通过**input_attachment_index**参数指定出附件的索引。读取附件时需要使用特殊的**subpassLoad**函数，大家会发现这个函数除了需要附件参数以外，不再需要任何其它的参数。对比之前使用的**texture**函数，不仅需要传入Texture，还需要传入uv参数。

## Debug信息
最后我们增加一下Debug的选项进来，可以根据Index显示Color、Normal、Depth的信息。最后效果就是预览图那样，在后面的Demo里面，我们会来尝试实现一下简单的延迟光照。

