---
title: Vulkan Demo 03 DemoBase
date: 2019-08-01 00:16:33
tags:
- Vulkan
- 3D
- Tutorial
- Triangle
categories:
- Vulkan

---

# [DemoBase](https://github.com/BobLChen/VulkanDemos/tree/master/examples/2_Triangle)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

在[2_Triangle](http://xiaopengyou.fun/public/2019/07/28/2-Triangle/#more)这个Demo里面，会发现为了绘制一个三角形做了太多的准备工作，后面还有大量的Demo，如果每个Demo都来搞一遍就有点儿费时费力了。为了偷懒以及稍微进行结构优化，才有了这个Demo。在这个Demo里面主要演示了如何将一些公用的功能进行简单封装。

<!-- more -->

## 封装FrameBuffer

在上个Demo中，创建了**RenderPass**、**FrameBuffer**、**DepthStencil**，这几个东西其实可以进行适当封装。在**AppModuleBase.h**里面准备一个**Prepare()**函数，用来准备这三个基本的要素。

```c++
virtual void Prepare()
{		
    CreateDepthStencil();
    CreateRenderPass();
    CreateFrameBuffers();
}
```

在这个函数里面，分别创建这三个要素。需要注意的是，以后有可能需要自己创建这三个资源，所以也需要对三个函数能够拥有**Override**权限。

### 创建DepthStenicl

DepthStencil创建如下，其实就是从上个Demo里面复制粘贴过来。

```c++
virtual void CreateDepthStencil()
{
    DestoryDepthStencil();

    int32 fwidth    = GetVulkanRHI()->GetSwapChain()->GetWidth();
    int32 fheight   = GetVulkanRHI()->GetSwapChain()->GetHeight();
    VkDevice device = GetVulkanRHI()->GetDevice()->GetInstanceHandle();

    VkImageCreateInfo imageCreateInfo;
    ZeroVulkanStruct(imageCreateInfo, VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO);
    imageCreateInfo.imageType   = VK_IMAGE_TYPE_2D;
    imageCreateInfo.format      = PixelFormatToVkFormat(m_DepthFormat, false);
    imageCreateInfo.extent      = { (uint32)fwidth, (uint32)fheight, 1 };
    imageCreateInfo.mipLevels   = 1;
    imageCreateInfo.arrayLayers = 1;
    imageCreateInfo.samples     = m_SampleCount;
    imageCreateInfo.tiling      = VK_IMAGE_TILING_OPTIMAL;
    imageCreateInfo.usage       = VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT | VK_IMAGE_USAGE_TRANSFER_SRC_BIT;
    imageCreateInfo.flags       = 0;
    VERIFYVULKANRESULT(vkCreateImage(device, &imageCreateInfo, VULKAN_CPU_ALLOCATOR, &m_DepthStencilImage));

    VkMemoryRequirements memRequire;
    vkGetImageMemoryRequirements(device, m_DepthStencilImage, &memRequire);
    uint32 memoryTypeIndex = 0;
    VERIFYVULKANRESULT(GetVulkanRHI()->GetDevice()->GetMemoryManager().GetMemoryTypeFromProperties(memRequire.memoryTypeBits, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, &memoryTypeIndex));

    VkMemoryAllocateInfo memAllocateInfo;
    ZeroVulkanStruct(memAllocateInfo, VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO);
    memAllocateInfo.allocationSize  = memRequire.size;
    memAllocateInfo.memoryTypeIndex = memoryTypeIndex;
    vkAllocateMemory(device, &memAllocateInfo, VULKAN_CPU_ALLOCATOR, &m_DepthStencilMemory);
    vkBindImageMemory(device, m_DepthStencilImage, m_DepthStencilMemory, 0);

    VkImageViewCreateInfo imageViewCreateInfo;
    ZeroVulkanStruct(imageViewCreateInfo, VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO);
    imageViewCreateInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
    imageViewCreateInfo.format   = PixelFormatToVkFormat(m_DepthFormat, false);
    imageViewCreateInfo.flags    = 0;
    imageViewCreateInfo.image    = m_DepthStencilImage;
    imageViewCreateInfo.subresourceRange.aspectMask     = VK_IMAGE_ASPECT_DEPTH_BIT | VK_IMAGE_ASPECT_STENCIL_BIT;
    imageViewCreateInfo.subresourceRange.baseMipLevel   = 0;
    imageViewCreateInfo.subresourceRange.levelCount     = 1;
    imageViewCreateInfo.subresourceRange.baseArrayLayer = 0;
    imageViewCreateInfo.subresourceRange.layerCount     = 1;
    VERIFYVULKANRESULT(vkCreateImageView(device, &imageViewCreateInfo, VULKAN_CPU_ALLOCATOR, &m_DepthStencilView));
}
```

### 创建RenderPass

仍然从上个Demo中复制过来。

```c++
virtual void CreateRenderPass()
{
    DestoryRenderPass();

    VkDevice device = GetVulkanRHI()->GetDevice()->GetInstanceHandle();
    PixelFormat pixelFormat = GetVulkanRHI()->GetPixelFormat();

    std::vector<VkAttachmentDescription> attachments(2);
    // color attachment
    attachments[0].format		  = PixelFormatToVkFormat(pixelFormat, false);
    attachments[0].samples		  = m_SampleCount;
    attachments[0].loadOp		  = VK_ATTACHMENT_LOAD_OP_CLEAR;
    attachments[0].storeOp        = VK_ATTACHMENT_STORE_OP_STORE;
    attachments[0].stencilLoadOp  = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    attachments[0].stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    attachments[0].initialLayout  = VK_IMAGE_LAYOUT_UNDEFINED;
    attachments[0].finalLayout    = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
    // depth stencil attachment
    attachments[1].format         = PixelFormatToVkFormat(m_DepthFormat, false);
    attachments[1].samples        = m_SampleCount;
    attachments[1].loadOp         = VK_ATTACHMENT_LOAD_OP_CLEAR;
    attachments[1].storeOp        = VK_ATTACHMENT_STORE_OP_STORE;
    attachments[1].stencilLoadOp  = VK_ATTACHMENT_LOAD_OP_CLEAR;
    attachments[1].stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    attachments[1].initialLayout  = VK_IMAGE_LAYOUT_UNDEFINED;
    attachments[1].finalLayout	  = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;

    VkAttachmentReference colorReference = { };
    colorReference.attachment = 0;
    colorReference.layout     = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;

    VkAttachmentReference depthReference = { };
    depthReference.attachment = 1;
    depthReference.layout     = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;

    VkSubpassDescription subpassDescription = { };
    subpassDescription.pipelineBindPoint       = VK_PIPELINE_BIND_POINT_GRAPHICS;
    subpassDescription.colorAttachmentCount    = 1;
    subpassDescription.pColorAttachments       = &colorReference;
    subpassDescription.pDepthStencilAttachment = &depthReference;
    subpassDescription.pResolveAttachments     = nullptr;
    subpassDescription.inputAttachmentCount    = 0;
    subpassDescription.pInputAttachments       = nullptr;
    subpassDescription.preserveAttachmentCount = 0;
    subpassDescription.pPreserveAttachments    = nullptr;

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
    renderPassInfo.attachmentCount = static_cast<uint32_t>(attachments.size());
    renderPassInfo.pAttachments    = attachments.data();
    renderPassInfo.subpassCount    = 1;
    renderPassInfo.pSubpasses      = &subpassDescription;
    renderPassInfo.dependencyCount = static_cast<uint32_t>(dependencies.size());
    renderPassInfo.pDependencies   = dependencies.data();
    VERIFYVULKANRESULT(vkCreateRenderPass(device, &renderPassInfo, VULKAN_CPU_ALLOCATOR, &m_RenderPass));
}
```

### 创建FrameBuffers

仍然从上个Demo中拷贝过来。。。

```c++
virtual void CreateFrameBuffers()
{
    DestroyFrameBuffers();

    int32 fwidth    = GetVulkanRHI()->GetSwapChain()->GetWidth();
    int32 fheight   = GetVulkanRHI()->GetSwapChain()->GetHeight();
    VkDevice device = GetVulkanRHI()->GetDevice()->GetInstanceHandle();

    VkImageView attachments[2];
    attachments[1] = m_DepthStencilView;

    VkFramebufferCreateInfo frameBufferCreateInfo;
    ZeroVulkanStruct(frameBufferCreateInfo, VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO);
    frameBufferCreateInfo.renderPass      = m_RenderPass;
    frameBufferCreateInfo.attachmentCount = 2;
    frameBufferCreateInfo.pAttachments    = attachments;
    frameBufferCreateInfo.width			  = fwidth;
    frameBufferCreateInfo.height		  = fheight;
    frameBufferCreateInfo.layers		  = 1;

    const std::vector<VkImageView>& backbufferViews = GetVulkanRHI()->GetBackbufferViews();

    m_FrameBuffers.resize(backbufferViews.size());
    for (uint32 i = 0; i < m_FrameBuffers.size(); ++i) {
        attachments[0] = backbufferViews[i];
        VERIFYVULKANRESULT(vkCreateFramebuffer(device, &frameBufferCreateInfo, VULKAN_CPU_ALLOCATOR, &m_FrameBuffers[i]));
    }
}
```

这三个功能的封装很简单，就是拷贝粘贴即可，写代码不就是个复制粘贴的过程的嘛。

### 封装常用属性

**AppModuleBase**只提供RenderPass、FrameBuffer、DepthStencil的封装，我不打算在**AppModuleBase**里面进行过度的封装，因为随着这个教程的不断演化，最终我是打算演化成为一个渲染功能比较健全的引擎。所以**Demo**功能我们专门封装到一个类**DemoBase**。

有一些常用属性是需要经常访问的，例如**Device**、**FrameBuffer尺寸**、**Queue**等等。我们为**DemoBase**提供一个**Setup**函数，用于获取保存这些属性。即只要在**Demo**中调用了**Setup**函数，就可以直接获取这些参数。

```c++
void DemoBase::Setup()
{
	auto vulkanRHI    = GetVulkanRHI();
	auto vulkanDevice = vulkanRHI->GetDevice();

	m_SwapChain     = vulkanRHI->GetSwapChain();

	m_Device		= vulkanDevice->GetInstanceHandle();
	m_VulkanDevice	= vulkanDevice;
	m_GfxQueue		= vulkanDevice->GetGraphicsQueue()->GetHandle();
	m_PresentQueue	= vulkanDevice->GetPresentQueue()->GetHandle();
    
	m_FrameWidth	= vulkanRHI->GetSwapChain()->GetWidth();
	m_FrameHeight	= vulkanRHI->GetSwapChain()->GetHeight();
}
```

## 封装常用Vulkan资源

另外有一些功能是比较常用的，例如**Fence**、**CommandPool**、**CommandBuffer**、**PipelineCache**等。因此重写**AppModuleBase**的**Prepare**函数，在里面准备这些资源。

```c++
void Prepare() override
{
    AppModuleBase::Prepare();
    CreateFences();
    CreateCommandBuffers();
    CreatePipelineCache();
    CreateDefaultRes();
}
```

有创建就会有释放，因此提供相应的释放函数。

```c++
void Release() override
{
    AppModuleBase::Release();
    DestroyDefaultRes();
    DestroyFences();
    DestroyCommandBuffers();
    DestroyPipelineCache();
}
```

### CreateFences

**Fence**的创建如下

```c++
void DemoBase::CreateFences()
{
	VkDevice device  = GetVulkanRHI()->GetDevice()->GetInstanceHandle();
    int32 frameCount = GetVulkanRHI()->GetSwapChain()->GetBackBufferCount();
        
	VkFenceCreateInfo fenceCreateInfo;
	ZeroVulkanStruct(fenceCreateInfo, VK_STRUCTURE_TYPE_FENCE_CREATE_INFO);
	fenceCreateInfo.flags = VK_FENCE_CREATE_SIGNALED_BIT;
        
    m_Fences.resize(frameCount);
	for (int32 i = 0; i < m_Fences.size(); ++i) {
		VERIFYVULKANRESULT(vkCreateFence(device, &fenceCreateInfo, VULKAN_CPU_ALLOCATOR, &m_Fences[i]));
	}

	VkSemaphoreCreateInfo createInfo;
	ZeroVulkanStruct(createInfo, VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO);
	vkCreateSemaphore(device, &createInfo, VULKAN_CPU_ALLOCATOR, &m_RenderComplete);
}
```

### CreateCommandBuffers

**CommandBuffer**创建如下：

```c++
void DemoBase::CreateCommandBuffers()
{
    VkDevice device = GetVulkanRHI()->GetDevice()->GetInstanceHandle();

    VkCommandPoolCreateInfo cmdPoolInfo;
    ZeroVulkanStruct(cmdPoolInfo, VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO);
    cmdPoolInfo.queueFamilyIndex = GetVulkanRHI()->GetDevice()->GetPresentQueue()->GetFamilyIndex();
    cmdPoolInfo.flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT;
    VERIFYVULKANRESULT(vkCreateCommandPool(device, &cmdPoolInfo, VULKAN_CPU_ALLOCATOR, &m_CommandPool));

    VkCommandBufferAllocateInfo cmdBufferInfo;
    ZeroVulkanStruct(cmdBufferInfo, VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO);
    cmdBufferInfo.level              = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
    cmdBufferInfo.commandBufferCount = 1;
    cmdBufferInfo.commandPool        = m_CommandPool;

    m_CommandBuffers.resize(GetVulkanRHI()->GetSwapChain()->GetBackBufferCount());
    for (int32 i = 0; i < m_CommandBuffers.size(); ++i) {
        vkAllocateCommandBuffers(device, &cmdBufferInfo, &(m_CommandBuffers[i]));
    }
}
```

创建CommandBuffer的时候顺便提前创建一个**CommandPool**。

### CreatePipelineCache

PipelineCache的创建则比较简单。

```c++
void DemoBase::CreatePipelineCache()
{
	VkDevice device = GetVulkanRHI()->GetDevice()->GetInstanceHandle();

	VkPipelineCacheCreateInfo createInfo;
    ZeroVulkanStruct(createInfo, VK_STRUCTURE_TYPE_PIPELINE_CACHE_CREATE_INFO);
    VERIFYVULKANRESULT(vkCreatePipelineCache(device, &createInfo, VULKAN_CPU_ALLOCATOR, &m_PipelineCache));
```

上述这些资源的创建，其实也都是从[2_Triangle](http://xiaopengyou.fun/public/2019/07/28/2-Triangle/#more)这个Demo里面搬运过来的代码，只是简单对它们进行了封装。

## 封装AcquireBackbufferIndex

Backbuffer的索引获取是通过Swapchain请求到的，为了避免写起来麻烦，对它也简单封装一下。

```c++
int32 DemoBase::AcquireBackbufferIndex()
{
	int32 backBufferIndex = m_SwapChain->AcquireImageIndex(&m_PresentComplete);
	return backBufferIndex;
}
```

## 封装Present

**Present**的调用目前来讲也是异常繁琐，同样对它也进行简单封装一下。

```C++
void DemoBase::Present(int backBufferIndex)
{
	VkSubmitInfo submitInfo = {};
	submitInfo.sType 				= VK_STRUCTURE_TYPE_SUBMIT_INFO;
	submitInfo.pWaitDstStageMask 	= &m_WaitStageMask;									
	submitInfo.pWaitSemaphores 		= &m_PresentComplete;
	submitInfo.waitSemaphoreCount 	= 1;
	submitInfo.pSignalSemaphores 	= &m_RenderComplete;
	submitInfo.signalSemaphoreCount = 1;											
	submitInfo.pCommandBuffers 		= &(m_CommandBuffers[backBufferIndex]);
	submitInfo.commandBufferCount 	= 1;												
	
    vkResetFences(m_Device, 1, &(m_Fences[backBufferIndex]));

	VERIFYVULKANRESULT(vkQueueSubmit(m_GfxQueue, 1, &submitInfo, m_Fences[backBufferIndex]));
    vkWaitForFences(m_Device, 1, &(m_Fences[backBufferIndex]), true, MAX_uint64);
    
    // present
    m_SwapChain->Present(m_VulkanDevice->GetGraphicsQueue(), m_VulkanDevice->GetPresentQueue(), &m_RenderComplete);
}
```

至此为止，我们的**DemoBase**就已经完成了它的使命，**TriangleDemo**只要继承它，就可以拥有这些常用功能。经过封装之后，新的**TriangleDemo**代码变成了500多行。代码越少，越方便理解。

之前的Demo里面初始化以及释放函数长这样：

```C++
virtual bool Init() override
{
    CreateDepthStencil();
    CreateRenderPass();
    CreateFrameBuffers();
    CreateSemaphores();
    CreateFences();
    CreateCommandBuffers();
    CreateMeshBuffers();
    CreateUniformBuffers();
    CreateDescriptorPool();
    CreateDescriptorSetLayout();
    CreateDescriptorSet();
    CreatePipelines();
    SetupCommandBuffers();

    m_Ready = true;

    return true;
}

virtual void Exist() override
{
    DestroyFrameBuffers();
    DestoryRenderPass();
    DestoryDepthStencil();
    DestroyCommandBuffers();
    DestroyDescriptorSetLayout();
    DestroyDescriptorPool();
    DestroyPipelines();
    DestroyUniformBuffers();
    DestroyMeshBuffers();
    DestorySemaphores();
    DestroyFences();
}
```

现在的初始化释放函数变成了这样：

```C++
virtual bool Init() override
{
    DemoBase::Setup();
    DemoBase::Prepare();

    CreateMeshBuffers();
    CreateUniformBuffers();
    CreateDescriptorPool();
    CreateDescriptorSetLayout();
    CreateDescriptorSet();
    CreatePipelines();
    SetupCommandBuffers();

    m_Ready = true;

    return true;
}

virtual void Exist() override
{
    DemoBase::Release();

    DestroyDescriptorSetLayout();
    DestroyDescriptorPool();
    DestroyPipelines();
    DestroyUniformBuffers();
    DestroyMeshBuffers();
}
```

总体来说要清爽很多。