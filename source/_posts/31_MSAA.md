---
title: 31_MSAA
date: 2020-01-15 22:15:19
tags:
- Vulkan
- MSAA
- 3D
- Demo
categories:
- Vulkan
---

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

细心的同学会发现之前的Demo里面画面效果不是非常的好，有很多锯齿存在，这里将展示如何利用MSAA来消除这些锯齿。

<!-- more -->

![31_MSAA](https://raw.githubusercontent.com/BobLChen/VulkanDemos/master/preview/29_MSAA.gif)

## MSAA

什么是MSAA？这里有一篇文章讲解的非常详细，大家可以看看。
[MSAA详解](https://www.cnblogs.com/ghl_carmack/p/8245032.html)


## 线条模型

为了凸显出锯齿的情况，先准备一个线条状的圆球出来。创建圆球的代码如下：

```c++
void GenerateLineSphere(std::vector<float>& outVertices, int32 sphslices, float scale)
{
    int32 count  = 0;
    int32 slices = sphslices;
    int32 stacks = slices;

    outVertices.resize((slices + 1) * stacks * 3 * 2);

    float ds = 1.0f / sphslices;
    float dt = 1.0f / sphslices;
    float t  = 1.0;
    float drho   = PI / stacks;
    float dtheta = 2.0 * PI / slices;

    for (int32 i= 0; i < stacks; ++i) 
    {
        float rho = i * drho;
        float s   = 0.0;
        for (int32 j = 0; j<=slices; ++j) {
            float theta = (j == slices) ? 0.0f : j * dtheta;
            float x = -sin(theta) * sin(rho) * scale;
            float z =  cos(theta) * sin(rho) * scale;
            float y = -cos(rho) * scale;

            outVertices[count + 0] = x;
            outVertices[count + 1] = y;
            outVertices[count + 2] = z;
            count += 3;

            x = -sin(theta) * sin(rho+drho) * scale;
            z =  cos(theta) * sin(rho+drho) * scale;
            y = -cos(rho+drho) * scale;

            outVertices[count + 0] = x;
            outVertices[count + 1] = y;
            outVertices[count + 2] = z;
            count += 3;

            s += ds;
        }
        t -= dt;
    }
}
```

圆球的创建代码非常简单，这里不再过多叙述。圆球创建好之后，然后创建出对应的DVKModel即可：

```c++
// LineSphere
std::vector<float> vertices;
GenerateLineSphere(vertices, 40, 1.0f);

// model
m_LineModel = vk_demo::DVKModel::Create(
    m_VulkanDevice,
    cmdBuffer,
    vertices,
    {},
    { VertexAttribute::VA_Position }
);
```

## 创建MSAA

先准备MSAA Texture：

```c++
void CreateMSAATexture()
{
    // msaa color texture
    m_MSAAColorTexture = vk_demo::DVKTexture::CreateRenderTarget(
        m_VulkanDevice,
        PixelFormatToVkFormat(GetVulkanRHI()->GetPixelFormat(), false),
        VK_IMAGE_ASPECT_COLOR_BIT,
        m_FrameWidth, m_FrameHeight,
        VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT | VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT,
        m_MSAACount
    );
    
    // msaa depth texture
    m_MSAADepthTexture = vk_demo::DVKTexture::CreateRenderTarget(
        m_VulkanDevice,
        PixelFormatToVkFormat(m_DepthFormat, false),
        VK_IMAGE_ASPECT_DEPTH_BIT | VK_IMAGE_ASPECT_STENCIL_BIT,
        m_FrameWidth, m_FrameHeight,
        VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT | VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT,
        m_MSAACount
    );
}
```

ColorTexture以及DepthTexture都需要准备一份。

然后准备对应的MSAAFrameBuffer：

```c++
void CreateMSAAFrameBuffers()
{
    DestroyMSAATexture();
    CreateMSAATexture();

    int32 fwidth    = GetVulkanRHI()->GetSwapChain()->GetWidth();
    int32 fheight   = GetVulkanRHI()->GetSwapChain()->GetHeight();
    VkDevice device = GetVulkanRHI()->GetDevice()->GetInstanceHandle();

    std::vector<VkImageView> attachments(4);
    attachments[0] = m_MSAAColorTexture->imageView;
    // attachments[1] = swapchain image
    attachments[2] = m_MSAADepthTexture->imageView;
    attachments[3] = m_DepthStencilView;

    VkFramebufferCreateInfo frameBufferCreateInfo;
    ZeroVulkanStruct(frameBufferCreateInfo, VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO);
    frameBufferCreateInfo.renderPass      = m_RenderPass;
    frameBufferCreateInfo.attachmentCount = attachments.size();
    frameBufferCreateInfo.pAttachments    = attachments.data();
    frameBufferCreateInfo.width			  = fwidth;
    frameBufferCreateInfo.height		  = fheight;
    frameBufferCreateInfo.layers		  = 1;

    const std::vector<VkImageView>& backbufferViews = GetVulkanRHI()->GetBackbufferViews();

    m_FrameBuffers.resize(backbufferViews.size());
    for (uint32 i = 0; i < m_FrameBuffers.size(); ++i) {
        attachments[1] = backbufferViews[i];
        VERIFYVULKANRESULT(vkCreateFramebuffer(device, &frameBufferCreateInfo, VULKAN_CPU_ALLOCATOR, &m_FrameBuffers[i]));
    }
}
```

最后准备MSAARenderPass：

```c++
void CreateMSAARenderPass()
{
    VkDevice device = GetVulkanRHI()->GetDevice()->GetInstanceHandle();
    PixelFormat pixelFormat = GetVulkanRHI()->GetPixelFormat();

    std::vector<VkAttachmentDescription> attachments(4);
    // MSAA Attachment
    attachments[0].format		  = PixelFormatToVkFormat(pixelFormat, false);
    attachments[0].samples		  = m_MSAACount;
    attachments[0].loadOp		  = VK_ATTACHMENT_LOAD_OP_CLEAR;
    attachments[0].storeOp        = VK_ATTACHMENT_STORE_OP_STORE;
    attachments[0].stencilLoadOp  = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    attachments[0].stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    attachments[0].initialLayout  = VK_IMAGE_LAYOUT_UNDEFINED;
    attachments[0].finalLayout	  = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
    // color attachment
    attachments[1].format         = PixelFormatToVkFormat(pixelFormat, false);
    attachments[1].samples        = VK_SAMPLE_COUNT_1_BIT;
    attachments[1].loadOp         = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    attachments[1].storeOp        = VK_ATTACHMENT_STORE_OP_STORE;
    attachments[1].stencilLoadOp  = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    attachments[1].stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    attachments[1].initialLayout  = VK_IMAGE_LAYOUT_UNDEFINED;
    attachments[1].finalLayout    = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
    // MSAA Depth
    attachments[2].format         = PixelFormatToVkFormat(m_DepthFormat, false);
    attachments[2].samples        = m_MSAACount;
    attachments[2].loadOp         = VK_ATTACHMENT_LOAD_OP_CLEAR;
    attachments[2].storeOp        = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    attachments[2].stencilLoadOp  = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    attachments[2].stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    attachments[2].initialLayout  = VK_IMAGE_LAYOUT_UNDEFINED;
    attachments[2].finalLayout    = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
    // depth stencil attachment
    attachments[3].format         = PixelFormatToVkFormat(m_DepthFormat, false);
    attachments[3].samples        = VK_SAMPLE_COUNT_1_BIT;
    attachments[3].loadOp         = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    attachments[3].storeOp        = VK_ATTACHMENT_STORE_OP_STORE;
    attachments[3].stencilLoadOp  = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    attachments[3].stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    attachments[3].initialLayout  = VK_IMAGE_LAYOUT_UNDEFINED;
    attachments[3].finalLayout    = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;

    VkAttachmentReference colorReference;
    colorReference.attachment = 0;
    colorReference.layout     = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;

    VkAttachmentReference depthReference;
    depthReference.attachment = 2;
    depthReference.layout     = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;

    VkAttachmentReference resolveReference;
    resolveReference.attachment = 1;
    resolveReference.layout     = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;

    VkSubpassDescription subpass = {};
    subpass.pipelineBindPoint       = VK_PIPELINE_BIND_POINT_GRAPHICS;
    subpass.colorAttachmentCount    = 1;
    subpass.pColorAttachments       = &colorReference;
    subpass.pResolveAttachments     = &resolveReference;
    subpass.pDepthStencilAttachment = &depthReference;

    std::vector<VkSubpassDependency> dependencies(2);
    dependencies[0].srcSubpass      = VK_SUBPASS_EXTERNAL;
    dependencies[0].dstSubpass      = 0;
    dependencies[0].srcStageMask    = VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT;
    dependencies[0].dstStageMask    = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
    dependencies[0].srcAccessMask   = VK_ACCESS_MEMORY_READ_BIT;
    dependencies[0].dstAccessMask   = VK_ACCESS_COLOR_ATTACHMENT_READ_BIT | VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
    dependencies[0].dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;

    dependencies[1].srcSubpass      = 0;
    dependencies[1].dstSubpass      = VK_SUBPASS_EXTERNAL;
    dependencies[1].srcStageMask    = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
    dependencies[1].dstStageMask    = VK_PIPELINE_STAGE_BOTTOM_OF_PIPE_BIT;
    dependencies[1].srcAccessMask   = VK_ACCESS_COLOR_ATTACHMENT_READ_BIT | VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
    dependencies[1].dstAccessMask   = VK_ACCESS_MEMORY_READ_BIT;
    dependencies[1].dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;

    VkRenderPassCreateInfo renderPassInfo;
    ZeroVulkanStruct(renderPassInfo, VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO);
    renderPassInfo.attachmentCount = attachments.size();
    renderPassInfo.pAttachments    = attachments.data();
    renderPassInfo.subpassCount    = 1;
    renderPassInfo.pSubpasses      = &subpass;
    renderPassInfo.dependencyCount = dependencies.size();
    renderPassInfo.pDependencies   = dependencies.data();
    VERIFYVULKANRESULT(vkCreateRenderPass(device, &renderPassInfo, VULKAN_CPU_ALLOCATOR, &m_RenderPass));
}
```

这里需要注意的就是为了显示，我们需要将MSAATexture Resolve到正常的Texture上，关键代码如下：

```c++
VkSubpassDescription subpass = {};
subpass.pipelineBindPoint       = VK_PIPELINE_BIND_POINT_GRAPHICS;
subpass.colorAttachmentCount    = 1;
subpass.pColorAttachments       = &colorReference;
subpass.pResolveAttachments     = &resolveReference;
subpass.pDepthStencilAttachment = &depthReference;
```

FrameBuffer以及RenderPass都创建完成之后，即可开启MSAA功能。效果也如预览图所示，对锯齿有非常大的提升。