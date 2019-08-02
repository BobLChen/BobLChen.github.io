---
title: Vulkan Demo 02 Triangle
date: 2019-07-28 00:16:33
tags:
- Vulkan
- 物理设备
- 3D
- Demo
categories:
- Vulkan
---

# [Triangle](https://github.com/BobLChen/VulkanDemos/tree/master/examples/2_Triangle)
[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

## 注意事项
在这之前有一些注意事项需要说明，这些注意事项非常关键，决定着之后一些东西的理解。

<!-- more -->

### NDC坐标系
由于Vulkan使用的NDC坐标系有点奇葩，跟Opengl、DirectX等不太一样，因此需要特别说明一下。

- X轴：[-1, 1]
- Y轴：[-1, 1]
- Z轴：[ 0, 1]

需要注意的是Y轴是跟DirectX或者OpenGL的相反，因此一些在DirectX或者OpenGL适用的方法在Vulkan里面不适合。另外需要注意的是深度值跟OpenGL不一致，OpenGL是[-1, 1]，但是跟DirectX一样，因此在OpenGL中关于深度处理的算法无法适用于Vulkan。

### 3D坐标系
这个Demo今后将会使用左手系，如下图所示：

![0.png](0.png)

- X轴：向右为正向
- Y轴：向上为正向
- Z轴：向前为正向

之所以使用左手系，是因为我之前做的关于引擎的工作使用的是左手系。。。另外Unity3D也是使用的左手系，后面有些算法什么的可以从Unity3D中扒。

### 扩展
在这里大家不需要关心设备扩展是什么，后续会有说明。但是一定要记住，我们使用了左手系，但是呢NDC坐标系中的Y轴翻转了，矩阵通过线性运算又无法有效的解决这个问题(虽然可以一样翻转Y轴，但是会涉及到其它的一些列问题)。因此用上了Vulkan的一个扩展，这个扩展就是专门用来解决这个问题的，即将Y轴[-1, 1]翻转为[1, -1]。扩展名称为"VK_KHR_maintenance1"。

## 窗体
在开始绘制三角形之前，我们需要在不同的平台创建可视化的窗体。目前本教程只支持Windows、MacOS、Ubuntu(其它未测试)，Android其实有计划支持，代码都已经准备好了，由于没有Android设备只能暂时告一段落，IOS之前已经跑通，但是配置较为繁琐，还没时间研究如何脚本产生工程配置。

在不同平台上面如何创建窗体不过多介绍，有兴趣可以Main函数开始一步一步的分析。

## Vulkan初始化

### Instance
Instance可以理解与Vulkan交互的一个桥梁。创建Instance需要像驱动程序提供一些应用层信息。例如版本号，名称等等。
```C++
VkApplicationInfo appInfo;
ZeroVulkanStruct(appInfo, VK_STRUCTURE_TYPE_APPLICATION_INFO);
appInfo.pApplicationName   = Engine::Get()->GetTitle();
appInfo.applicationVersion = VK_MAKE_VERSION(0, 0, 0);
appInfo.pEngineName        = ENGINE_NAME;
appInfo.engineVersion      = VK_MAKE_VERSION(0, 0, 0);
appInfo.apiVersion         = VK_API_VERSION_1_0;
```

#### 扩展
创建Instance之前，我们需要指定扩展信息。扩展可以理解为插件，不同厂商支持不同，另外就是有些Debug插件可以为我们所用。通过vkEnumerateInstanceExtensionProperties和vkEnumerateInstanceLayerProperties接口可以获取到扩展信息。具体实现可以看[这里](https://github.com/BobLChen/VulkanDemos/blob/master/Engine/Monkey/Vulkan/VulkanLayers.cpp#L209)。
以下是我的Mac电脑获取到一些关于扩展的信息。
```C++
LOG:  GetInstanceLayersAndExtensions          :238  - Found instance layer VK_LAYER_GOOGLE_unique_objects
LOG:  GetInstanceLayersAndExtensions          :238  - Found instance layer VK_LAYER_GOOGLE_threading
LOG:  GetInstanceLayersAndExtensions          :238  - Found instance layer VK_LAYER_LUNARG_standard_validation
LOG:  GetInstanceLayersAndExtensions          :238  - Found instance layer VK_LAYER_LUNARG_core_validation
LOG:  GetInstanceLayersAndExtensions          :238  - Found instance layer VK_LAYER_LUNARG_parameter_validation
LOG:  GetInstanceLayersAndExtensions          :238  - Found instance layer VK_LAYER_LUNARG_object_tracker
LOG:  GetInstanceLayersAndExtensions          :242  - Found instance extension VK_KHR_get_physical_device_properties2
LOG:  GetInstanceLayersAndExtensions          :242  - Found instance extension VK_KHR_surface
LOG:  GetInstanceLayersAndExtensions          :242  - Found instance extension VK_MVK_macos_surface
LOG:  GetInstanceLayersAndExtensions          :242  - Found instance extension VK_EXT_debug_report
LOG:  GetInstanceLayersAndExtensions          :242  - Found instance extension VK_EXT_debug_utils
LOG:  GetInstanceLayersAndExtensions          :290  Using instance layers
LOG:  GetInstanceLayersAndExtensions          :292  * VK_LAYER_LUNARG_standard_validation
LOG:  GetInstanceLayersAndExtensions          :292  * VK_LAYER_GOOGLE_unique_objects
LOG:  GetInstanceLayersAndExtensions          :292  * VK_LAYER_GOOGLE_threading
LOG:  GetInstanceLayersAndExtensions          :292  * VK_LAYER_LUNARG_core_validation
LOG:  GetInstanceLayersAndExtensions          :292  * VK_LAYER_LUNARG_parameter_validation
LOG:  GetInstanceLayersAndExtensions          :292  * VK_LAYER_LUNARG_object_tracker
LOG:  GetInstanceLayersAndExtensions          :301  Using instance extensions
LOG:  GetInstanceLayersAndExtensions          :303  * VK_EXT_debug_utils
LOG:  GetInstanceLayersAndExtensions          :303  * VK_EXT_debug_report
LOG:  GetInstanceLayersAndExtensions          :303  * VK_KHR_surface
LOG:  GetInstanceLayersAndExtensions          :303  * VK_MVK_macos_surface
```
从名称中就可以看出，带Debug或者validation字样的都是用来调试程序的。带Surface字样的都是跟窗体相关的，因为Vulkan跟平台不相关，但是确又要跟窗体进行绑定，所以窗体相关的接口就落到了扩展的头上。

#### 创建Instance
有了扩展信息之后，我们就可以创建出Instance。需要注意的是，Vulkan系的各种实例的创建，很少有通过参数完成的，一般都是各种`Info`结构体，然后通过填充结构体，传递结构体指针给相关创建的API来完成创建的功能。
```C++
VkInstanceCreateInfo instanceCreateInfo;
ZeroVulkanStruct(instanceCreateInfo, VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO);
instanceCreateInfo.pApplicationInfo        = &appInfo;
instanceCreateInfo.enabledExtensionCount   = uint32_t(m_InstanceExtensions.size());
instanceCreateInfo.ppEnabledExtensionNames = m_InstanceExtensions.size() > 0 ? m_InstanceExtensions.data() : nullptr;
instanceCreateInfo.enabledLayerCount       = uint32_t(m_InstanceLayers.size());
instanceCreateInfo.ppEnabledLayerNames     = m_InstanceLayers.size() > 0 ? m_InstanceLayers.data() : nullptr;

VkResult result = vkCreateInstance(&instanceCreateInfo, VULKAN_CPU_ALLOCATOR, &m_Instance);
```

#### DebugReportCallback设置
之前提到，在创建Instance时需要配置扩展，如果我们配置了`VK_LAYER_LUNARG_standard_validation、VK_EXT_debug_utils、VK_EXT_debug_report`，那么我们就可以在Instance创建成功之后配置DebugReport。
```C++
#define VK_DESTORY_DEBUG_REPORT_CALLBACK_EXT_NAME "vkDestroyDebugReportCallbackEXT"
#define VK_CREATE_DEBUG_REPORT_CALLBACK_EXT_NAME  "vkCreateDebugReportCallbackEXT"

VKAPI_ATTR VkBool32 VKAPI_CALL VulkanDebugCallBack(
    VkDebugReportFlagsEXT flags,
    VkDebugReportObjectTypeEXT objType,
    uint64_t obj,
    size_t location,
    int32_t code,
    const char* layerPrefix,
    const char* msg,
    void* userData)
{
    std::string prefix("");
    if (flags & VK_DEBUG_REPORT_ERROR_BIT_EXT) {
        prefix += "ERROR:";
    }
    if (flags & VK_DEBUG_REPORT_WARNING_BIT_EXT) {
        prefix += "WARNING:";
    }
    if (flags & VK_DEBUG_REPORT_PERFORMANCE_WARNING_BIT_EXT) {
        prefix += "PERFORMANCE:";
    }
    if (flags & VK_DEBUG_REPORT_INFORMATION_BIT_EXT) {
        prefix += "INFO:";
    }
    if (flags & VK_DEBUG_REPORT_DEBUG_BIT_EXT) {
        prefix += "DEBUG:";
    }
    MLOG("%s [%s] Code %d : %s", prefix.c_str(), layerPrefix, code, msg);
    return VK_FALSE;
}

VkDebugReportCallbackCreateInfoEXT debugInfo;
ZeroVulkanStruct(debugInfo, VK_STRUCTURE_TYPE_DEBUG_REPORT_CALLBACK_CREATE_INFO_EXT);
debugInfo.flags       = VK_DEBUG_REPORT_ERROR_BIT_EXT | VK_DEBUG_REPORT_WARNING_BIT_EXT;
debugInfo.pfnCallback = VulkanDebugCallBack;
debugInfo.pUserData   = this;

auto func    = (PFN_vkCreateDebugReportCallbackEXT)vkGetInstanceProcAddr(m_Instance, VK_CREATE_DEBUG_REPORT_CALLBACK_EXT_NAME);
bool success = true;
if (func != nullptr) {
    success = func(m_Instance, &debugInfo, nullptr, &m_MsgCallback) == VK_SUCCESS;
}
else {
    success = false;
}
```

### 选择GPU物理设备并创建逻辑设备
PC或者移动设备里面可能存在多个GPU，因此使用哪一个或者几个GPU需要我们自行决定。通过枚举设备列表，我们可以获取到当前平台拥有哪些GPU。
```C++
uint32 gpuCount = 0;
VkResult result = vkEnumeratePhysicalDevices(m_Instance, &gpuCount, nullptr);
std::vector<VkPhysicalDevice> physicalDevices(gpuCount);
vkEnumeratePhysicalDevices(m_Instance, &gpuCount, physicalDevices.data());
```
在PC上，有些GPU可能是集成到主板或者CPU上的，有些GPU是独立的，一般来讲独立版本性能会比集成的好一点儿，一般来讲。。。

### 创建逻辑设备
有了物理设备之后，还需要从物理设备中创建出逻辑设备。物理设备中有用不同功能的引擎，我们需要根据需要进行选择不同功能进行组合。在创建逻辑设备之前，还需要指定设备扩展。设备扩展一般都是一些扩展功能。以下API可以获取到设备的扩展信息。

```C++
uint32 count = 0;
vkEnumerateDeviceLayerProperties(m_PhysicalDevice, &count, nullptr);
std::vector<VkLayerProperties> properties(count);
vkEnumerateDeviceLayerProperties(m_PhysicalDevice, &count, properties.data());
```

以下是我的Mac电脑获取到的设备扩展：
```C++
LOG:  GetDeviceExtensionsAndLayers            :341  - Found device layer VK_LAYER_LUNARG_standard_validation
LOG:  GetDeviceExtensionsAndLayers            :341  - Found device layer VK_LAYER_GOOGLE_threading
LOG:  GetDeviceExtensionsAndLayers            :341  - Found device layer VK_LAYER_LUNARG_parameter_validation
LOG:  GetDeviceExtensionsAndLayers            :341  - Found device layer VK_LAYER_LUNARG_object_tracker
LOG:  GetDeviceExtensionsAndLayers            :341  - Found device layer VK_LAYER_LUNARG_core_validation
LOG:  GetDeviceExtensionsAndLayers            :341  - Found device layer VK_LAYER_GOOGLE_unique_objects
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_KHR_dedicated_allocation
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_KHR_descriptor_update_template
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_KHR_get_memory_requirements2
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_KHR_get_physical_device_properties2
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_KHR_image_format_list
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_KHR_maintenance1
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_KHR_maintenance2
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_KHR_push_descriptor
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_KHR_sampler_mirror_clamp_to_edge
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_KHR_shader_draw_parameters
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_KHR_surface
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_KHR_swapchain
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_EXT_shader_viewport_index_layer
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_EXT_vertex_attribute_divisor
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_MVK_macos_surface
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_MVK_moltenvk
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_AMD_negative_viewport_height
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_EXT_debug_marker
LOG:  GetDeviceExtensionsAndLayers            :345  - Found device extension VK_EXT_validation_cache
LOG:  GetDeviceExtensionsAndLayers            :422  Using device extensions
LOG:  GetDeviceExtensionsAndLayers            :424  * VK_KHR_swapchain
LOG:  GetDeviceExtensionsAndLayers            :424  * VK_KHR_sampler_mirror_clamp_to_edge
LOG:  GetDeviceExtensionsAndLayers            :424  * VK_KHR_maintenance1
LOG:  GetDeviceExtensionsAndLayers            :430  Using device layers
LOG:  GetDeviceExtensionsAndLayers            :432  * VK_LAYER_LUNARG_standard_validation
LOG:  GetDeviceExtensionsAndLayers            :432  * VK_LAYER_GOOGLE_unique_objects
LOG:  GetDeviceExtensionsAndLayers            :432  * VK_LAYER_GOOGLE_threading
LOG:  GetDeviceExtensionsAndLayers            :432  * VK_LAYER_LUNARG_core_validation
LOG:  GetDeviceExtensionsAndLayers            :432  * VK_LAYER_LUNARG_parameter_validation
LOG:  GetDeviceExtensionsAndLayers            :432  * VK_LAYER_LUNARG_object_tracker
```
从中我们可以看到一些Mac特有的扩展，例如`VK_MVK_macos_surface`以及`VK_MVK_moltenvk`。

获取到设备信息之后，就可以开始着手选择相应的功能，然后创建逻辑设备。

```C++
VkDeviceCreateInfo deviceInfo;
ZeroVulkanStruct(deviceInfo, VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO);
deviceInfo.enabledExtensionCount   = uint32_t(deviceExtensions.size());
deviceInfo.ppEnabledExtensionNames = deviceExtensions.data();
deviceInfo.enabledLayerCount       = uint32_t(validationLayers.size());
deviceInfo.ppEnabledLayerNames     = validationLayers.data();
deviceInfo.queueCreateInfoCount = uint32_t(queueFamilyInfos.size());
deviceInfo.pQueueCreateInfos    = queueFamilyInfos.data();
deviceInfo.pEnabledFeatures     = &m_PhysicalDeviceFeatures;
VkResult result = vkCreateDevice(m_PhysicalDevice, &deviceInfo, VULKAN_CPU_ALLOCATOR, &m_Device);
```

### 创建SwapChain

Swapchain的创建跟平台相关，因为Swapchain需要跟窗口绑定，需要将图像数据输出到窗体。例如在Mac下的创建代码如下
```C++
VkMacOSSurfaceCreateInfoMVK createInfo;
ZeroVulkanStruct(createInfo, VK_STRUCTURE_TYPE_MACOS_SURFACE_CREATE_INFO_MVK);
createInfo.pView = m_View;
VERIFYVULKANRESULT(vkCreateMacOSSurfaceMVK(instance, &createInfo, VULKAN_CPU_ALLOCATOR, outSurface));
```

### 创建DepthStencil

Depth是用来优化渲染以及确定图像遮挡关系的，Stencil一般用来做一些高级的功能。绘制2D UI可能不需要Depth，因为靠着绘制顺序可以确定遮挡关系，但是3D拥有深度信息，仅靠绘制顺序无法确定遮挡关系，因此一般需要深度信息。创建DepthStencil其实很简单，它们其实就是一个Texture。创建过程大致如下：
- 创建Image
- 分配内存或显存
- Image跟内存或显存绑定
- 创建ImageView

### RenderPass

RenderPass描述了输出信息，例如延迟渲染里面想要输出Albedo、Normal、Depth、Specular等数据，在普通的渲染里面想要输出Albedo、Depth等数据。

输出信息是通过附件来进行描述的，如下：
```C++
std::vector<VkAttachmentDescription> attachments(2);
// color attachment
attachments[0].format         = PixelFormatToVkFormat(pixelFormat, false);
attachments[0].samples        = m_SampleCount;
attachments[0].loadOp         = VK_ATTACHMENT_LOAD_OP_CLEAR;
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
attachments[1].finalLayout    = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
```
第一个附件描述了颜色附件的结构，第二个附件描述了Depth、Stencil附件的结构。只有附件描述信息还不够，因为数值里面的附件可能是其它Pass需要用到的(RenderTarget)，因此还需要另外一个数据结构来描述。
```C++
VkAttachmentReference colorReference = { };
colorReference.attachment = 0;
colorReference.layout     = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;

VkAttachmentReference depthReference = { };
depthReference.attachment = 1;
depthReference.layout     = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
```
这里的`attachment`就指的是attachments里面的索引。

由于有Subpass的存在，因此Subpass的描述信息也需要进行填充。
```C++
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
```
Subpass还有依赖信息，以及隐式的对Texture进行布局转换。
最后我们就可以根据以上信息创建出RenderPass
```
VkRenderPassCreateInfo renderPassInfo;
ZeroVulkanStruct(renderPassInfo, VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO);
renderPassInfo.attachmentCount = static_cast<uint32_t>(attachments.size());
renderPassInfo.pAttachments    = attachments.data();
renderPassInfo.subpassCount    = 1;
renderPassInfo.pSubpasses      = &subpassDescription;
renderPassInfo.dependencyCount = static_cast<uint32_t>(dependencies.size());
renderPassInfo.pDependencies   = dependencies.data();
VERIFYVULKANRESULT(vkCreateRenderPass(device, &renderPassInfo, VULKAN_CPU_ALLOCATOR, &m_RenderPass));
```

### FrameBuffer

创建FrameBuffer非常简单，申请了多少个缓冲区就创建多少个FrameBuffer。
```C++
VkFramebufferCreateInfo frameBufferCreateInfo;
ZeroVulkanStruct(frameBufferCreateInfo, VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO);
frameBufferCreateInfo.renderPass      = m_RenderPass;
frameBufferCreateInfo.attachmentCount = 2;
frameBufferCreateInfo.pAttachments    = attachments;
frameBufferCreateInfo.width           = fwidth;
frameBufferCreateInfo.height          = fheight;
frameBufferCreateInfo.layers          = 1;

const std::vector<VkImageView>& backbufferViews = GetVulkanRHI()->GetBackbufferViews();

m_FrameBuffers.resize(backbufferViews.size());
for (uint32 i = 0; i < m_FrameBuffers.size(); ++i) {
    attachments[0] = backbufferViews[i];
    VERIFYVULKANRESULT(vkCreateFramebuffer(device, &frameBufferCreateInfo, VULKAN_CPU_ALLOCATOR, &m_FrameBuffers[i]));
}
```

## 准备三角形

CPU与GPU是两个独立的设备，因此两个独立的设备协同进行任务时需要进行同步以确保任务的正确性。

### Fence

为了在CPU和GPU之间进行同步，我们可以使用`Fence`来进行同步。Fence的创建也是非常简单：
```C++
VkDevice device  = GetVulkanRHI()->GetDevice()->GetInstanceHandle();
int32 frameCount = GetVulkanRHI()->GetSwapChain()->GetBackBufferCount();

VkFenceCreateInfo fenceCreateInfo;
ZeroVulkanStruct(fenceCreateInfo, VK_STRUCTURE_TYPE_FENCE_CREATE_INFO);
fenceCreateInfo.flags = VK_FENCE_CREATE_SIGNALED_BIT;

m_Fences.resize(frameCount);
for (int32 i = 0; i < m_Fences.size(); ++i) {
    VERIFYVULKANRESULT(vkCreateFence(device, &fenceCreateInfo, VULKAN_CPU_ALLOCATOR, &m_Fences[i]));
}
```

### Semaphore
仅仅只是在CPU和GPU之间进行同步还不够，有时候还需要能够在GPU上进行同步操作。例如只有绘制完成时，才能将数据递交给`Present`引擎进行显示。GPU之间的同步是通过信号量完成，创建如下：

```C++
VkDevice device = GetVulkanRHI()->GetDevice()->GetInstanceHandle();
VkSemaphoreCreateInfo createInfo;
ZeroVulkanStruct(createInfo, VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO);
vkCreateSemaphore(device, &createInfo, VULKAN_CPU_ALLOCATOR, &m_RenderComplete);
```

### CommandBuffer

由于CPU和GPU是两个独立的设备，我们不可能在CPU中对GPU发起一个操作，就让CPU等待GPU完成，然后进行下一个操作。这样并不能让CPU和GPU并行的进行工作，为了能够让它们并行的进行工作，我们需要在CPU里面将一批需要进行的操作录制下来，然后发送给GPU批量执行。CommandBuffer就是用来录制命令的。为了能够高效的创建销毁CommandBUffer，Vulkan设计出了CommandBufferPool，通过Pool的方式可以让我们能够频繁的进行创建和销毁CommandBuffer并且还能保证优异的性能。
```C++
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
```

### 创建Buffer

渲染出一个三角形，需要顶点Buffer以及索引Buffer。Buffer的创建过程其实都是一样的，按照几个步骤完成:
- 创建Buffer
- 创建Buffer所需的内存(显存)
- Buffer与内存绑定

但是需要注意的是我们的数据是在内存里面，那么从内存到达显存就需要一个过程。无论如何，我们都有一个从内存拷贝数据到显存的操作。因此我们在创建VertexBuffer以及IndexBuffer的时候，我们其实是需要准备两个Buffer，一个Buffer在内存中，一个Buffer在显存中，通过CommandBuffer录制拷贝命令，将数据从内存拷贝至显存，最终我们使用的Buffer都是在显存上的。
```C++
// 顶点数据以及索引数据在整个生命周期中几乎不会发生改变，因此最佳的方式是将这些数据存储到GPU的内存中。
// 存储到GPU内存也能加快GPU的访问。为了存储到GPU内存中，需要如下几个步骤。
// 1、在主机端(Host)创建一个Buffer
// 2、将数据拷贝至该Buffer
// 3、在GPU端(Local Device)创建一个Buffer
// 4、通过Transfer簇将数据从主机端拷贝至GPU端
// 5、删除主基端(Host)的Buffer
// 6、使用GPU端(Local Device)的Buffer进行渲染
```

### UniformBuffer

对于Shader参数来讲，随时都可能被修改，如果放置到GPU内部，就需要频繁的从内存往显存传递，因此对于UniformBuffer一般直接在主机对GPU可见的内存区域上创建Buffer即可。[具体架构点我查看](http://xiaopengyou.fun/public/2019/06/07/VulkanBuffer%E4%B8%8E%E5%86%85%E5%AD%98/#more)

### PipelineLayout

在Shader里面会引用很多数据，例如输入的顶点数据，UniformBuffer数据，Texture数据等等。这些数据需要另外的数据结构来进行描述，可以简单的理解为需要一个说明书来指导Shader如何正确的访问数据。

`PipelineLayout`由`DescriptorSetLayout`构成，`DescriptorSetLayout`又由`SetLayoutBinding`构成。其实很好理解，SetLayoutBinding描述了资源的绑定信息。

```C++
VkDescriptorSetLayoutBinding layoutBinding;
layoutBinding.binding 			 = 0;
layoutBinding.descriptorType     = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
layoutBinding.descriptorCount    = 1;
layoutBinding.stageFlags 		 = VK_SHADER_STAGE_VERTEX_BIT;
layoutBinding.pImmutableSamplers = nullptr;
```
如上所以，在位置`0`绑定了一个`UniformBuffer`的数据，它被`Vertex Shader`所使用。为了更好的管理这些绑定信息，有时候会对它们进行分组，例如第一组有3个资源的描述，第二组有5个资源的描述。

```C++
VkDescriptorSetLayoutCreateInfo descSetLayoutInfo;
ZeroVulkanStruct(descSetLayoutInfo, VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO);
descSetLayoutInfo.bindingCount = 1;
descSetLayoutInfo.pBindings    = &layoutBinding;
VERIFYVULKANRESULT(vkCreateDescriptorSetLayout(device, &descSetLayoutInfo, VULKAN_CPU_ALLOCATOR, &m_DescriptorSetLayout));
```
一系列的`layoutBinding`就可以创建出一个`DescriptorSetLayout`。最终所有用到的`DescriptorSetLayout`组合起来，构成了我们最终需要的`PipelineLayout`。
```C++
VkPipelineLayoutCreateInfo pipeLayoutInfo;
ZeroVulkanStruct(pipeLayoutInfo, VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO);
pipeLayoutInfo.setLayoutCount = 1;
pipeLayoutInfo.pSetLayouts    = &m_DescriptorSetLayout;
VERIFYVULKANRESULT(vkCreatePipelineLayout(device, &pipeLayoutInfo, VULKAN_CPU_ALLOCATOR, &m_PipelineLayout));
```

### DescriptorSet

创建好了`Shader`的说明书之后，并不能直接使用，因为到现在为止我们都没有把对应的资源与它进行关联起来。为了完成这个操作，我们需要通过`DescriptorSet`来完成。
```C++
VkDescriptorSetAllocateInfo allocInfo;
ZeroVulkanStruct(allocInfo, VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO);
allocInfo.descriptorPool     = m_DescriptorPool;
allocInfo.descriptorSetCount = 1;
allocInfo.pSetLayouts        = &m_DescriptorSetLayout;
VERIFYVULKANRESULT(vkAllocateDescriptorSets(device, &allocInfo, &m_DescriptorSet));
```

创建好了`DescriptorSet`之后，我们将资源关联进去。
```C++
VkWriteDescriptorSet writeDescriptorSet;
ZeroVulkanStruct(writeDescriptorSet, VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET);
writeDescriptorSet.dstSet          = m_DescriptorSet;
writeDescriptorSet.descriptorCount = 1;
writeDescriptorSet.descriptorType  = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
writeDescriptorSet.pBufferInfo     = &m_MVPDescriptor;
writeDescriptorSet.dstBinding      = 0;
vkUpdateDescriptorSets(device, 1, &writeDescriptorSet, 0, nullptr);
```

### 创建Pipeline

Pipeline用来描述整个渲染流程中的行为，以及一些状态。例如三角形的排列方式、光栅化的方式、混合方式等等。如下所示：
```C++
VkDevice device = GetVulkanRHI()->GetDevice()->GetInstanceHandle();
        
VkPipelineCacheCreateInfo createInfo;
ZeroVulkanStruct(createInfo, VK_STRUCTURE_TYPE_PIPELINE_CACHE_CREATE_INFO);
VERIFYVULKANRESULT(vkCreatePipelineCache(device, &createInfo, VULKAN_CPU_ALLOCATOR, &m_PipelineCache));

VkPipelineInputAssemblyStateCreateInfo inputAssemblyState;
ZeroVulkanStruct(inputAssemblyState, VK_STRUCTURE_TYPE_PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO);
inputAssemblyState.topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST;

VkPipelineRasterizationStateCreateInfo rasterizationState;
ZeroVulkanStruct(rasterizationState, VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO);
rasterizationState.polygonMode 			   = VK_POLYGON_MODE_FILL;
rasterizationState.cullMode                = VK_CULL_MODE_NONE;
rasterizationState.frontFace               = VK_FRONT_FACE_COUNTER_CLOCKWISE;
rasterizationState.depthClampEnable        = VK_FALSE;
rasterizationState.rasterizerDiscardEnable = VK_FALSE;
rasterizationState.depthBiasEnable         = VK_FALSE;
rasterizationState.lineWidth 			   = 1.0f;

VkPipelineColorBlendAttachmentState blendAttachmentState[1] = {};
blendAttachmentState[0].colorWriteMask = (
    VK_COLOR_COMPONENT_R_BIT |
    VK_COLOR_COMPONENT_G_BIT |
    VK_COLOR_COMPONENT_B_BIT |
    VK_COLOR_COMPONENT_A_BIT
);
blendAttachmentState[0].blendEnable = VK_FALSE;

VkPipelineColorBlendStateCreateInfo colorBlendState;
ZeroVulkanStruct(colorBlendState, VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO);
colorBlendState.attachmentCount = 1;
colorBlendState.pAttachments    = blendAttachmentState;

VkPipelineViewportStateCreateInfo viewportState;
ZeroVulkanStruct(viewportState, VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO);
viewportState.viewportCount = 1;
viewportState.scissorCount  = 1;

std::vector<VkDynamicState> dynamicStateEnables;
dynamicStateEnables.push_back(VK_DYNAMIC_STATE_VIEWPORT);
dynamicStateEnables.push_back(VK_DYNAMIC_STATE_SCISSOR);
VkPipelineDynamicStateCreateInfo dynamicState;
ZeroVulkanStruct(dynamicState, VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO);
dynamicState.dynamicStateCount = (uint32_t)dynamicStateEnables.size();
dynamicState.pDynamicStates    = dynamicStateEnables.data();

VkPipelineDepthStencilStateCreateInfo depthStencilState;
ZeroVulkanStruct(depthStencilState, VK_STRUCTURE_TYPE_PIPELINE_DEPTH_STENCIL_STATE_CREATE_INFO);
depthStencilState.depthTestEnable 		= VK_TRUE;
depthStencilState.depthWriteEnable 		= VK_TRUE;
depthStencilState.depthCompareOp		= VK_COMPARE_OP_LESS_OR_EQUAL;
depthStencilState.depthBoundsTestEnable = VK_FALSE;
depthStencilState.back.failOp 			= VK_STENCIL_OP_KEEP;
depthStencilState.back.passOp 			= VK_STENCIL_OP_KEEP;
depthStencilState.back.compareOp 		= VK_COMPARE_OP_ALWAYS;
depthStencilState.stencilTestEnable 	= VK_FALSE;
depthStencilState.front 				= depthStencilState.back;

VkPipelineMultisampleStateCreateInfo multisampleState;
ZeroVulkanStruct(multisampleState, VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO);
multisampleState.rasterizationSamples = m_SampleCount;
multisampleState.pSampleMask 		  = nullptr;

// (triangle.vert):
// layout (location = 0) in vec3 inPos;
// layout (location = 1) in vec3 inColor;
// Attribute location 0: Position
// Attribute location 1: Color
// vertex input bindding
VkVertexInputBindingDescription vertexInputBinding = {};
vertexInputBinding.binding   = 0; // Vertex Buffer 0
vertexInputBinding.stride    = sizeof(Vertex); // Position + Color
vertexInputBinding.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;

std::vector<VkVertexInputAttributeDescription> vertexInputAttributs(2);
// position
vertexInputAttributs[0].binding  = 0;
vertexInputAttributs[0].location = 0; // triangle.vert : layout (location = 0)
vertexInputAttributs[0].format   = VK_FORMAT_R32G32B32_SFLOAT;
vertexInputAttributs[0].offset   = 0;
// color
vertexInputAttributs[1].binding  = 0;
vertexInputAttributs[1].location = 1; // triangle.vert : layout (location = 1)
vertexInputAttributs[1].format   = VK_FORMAT_R32G32B32_SFLOAT;
vertexInputAttributs[1].offset   = 12;

VkPipelineVertexInputStateCreateInfo vertexInputState;
ZeroVulkanStruct(vertexInputState, VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO);
vertexInputState.vertexBindingDescriptionCount   = 1;
vertexInputState.pVertexBindingDescriptions      = &vertexInputBinding;
vertexInputState.vertexAttributeDescriptionCount = 2;
vertexInputState.pVertexAttributeDescriptions    = vertexInputAttributs.data();

std::vector<VkPipelineShaderStageCreateInfo> shaderStages(2);
ZeroVulkanStruct(shaderStages[0], VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO);
ZeroVulkanStruct(shaderStages[1], VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO);
shaderStages[0].stage  = VK_SHADER_STAGE_VERTEX_BIT;
shaderStages[0].module = LoadSPIPVShader("assets/shaders/2_Triangle/triangle.vert.spv");
shaderStages[0].pName  = "main";
shaderStages[1].stage  = VK_SHADER_STAGE_FRAGMENT_BIT;
shaderStages[1].module = LoadSPIPVShader("assets/shaders/2_Triangle/triangle.frag.spv");
shaderStages[1].pName  = "main";

VkGraphicsPipelineCreateInfo pipelineCreateInfo;
ZeroVulkanStruct(pipelineCreateInfo, VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO);
pipelineCreateInfo.layout 				= m_PipelineLayout;
pipelineCreateInfo.renderPass 			= m_RenderPass;
pipelineCreateInfo.stageCount 			= (uint32_t)shaderStages.size();
pipelineCreateInfo.pStages 				= shaderStages.data();
pipelineCreateInfo.pVertexInputState 	= &vertexInputState;
pipelineCreateInfo.pInputAssemblyState 	= &inputAssemblyState;
pipelineCreateInfo.pRasterizationState 	= &rasterizationState;
pipelineCreateInfo.pColorBlendState 	= &colorBlendState;
pipelineCreateInfo.pMultisampleState 	= &multisampleState;
pipelineCreateInfo.pViewportState 		= &viewportState;
pipelineCreateInfo.pDepthStencilState 	= &depthStencilState;
pipelineCreateInfo.pDynamicState 		= &dynamicState;
VERIFYVULKANRESULT(vkCreateGraphicsPipelines(device, m_PipelineCache, 1, &pipelineCreateInfo, VULKAN_CPU_ALLOCATOR, &m_Pipeline));

vkDestroyShaderModule(device, shaderStages[0].module, VULKAN_CPU_ALLOCATOR);
vkDestroyShaderModule(device, shaderStages[1].module, VULKAN_CPU_ALLOCATOR);
```

创建一个Pipeline不易，代码超长，极容易出错。

### 录制绘制命令

Pipeline创建完成之后，就可以开始录制命令。命令的录制也比较简单，我们配置好视口尺寸，Pipeline、DescriptorSets、VertexBuffer、IndexBuffer即可。指定`Pipeline`是因为Pipeline决定着渲染的行为，所以我们必须配置。指定`DescriptorSets`是因为我们的数据通过`DescriptorSets`进行了关联，但是需要注意的是`DescriptorSets`并没有与`Pipeline`关联，所以只指定`Pipeline`还不够，还需要指定`DescriptorSets`。最后的录制代码看起来像这样：
```C++
VkCommandBufferBeginInfo cmdBeginInfo;
ZeroVulkanStruct(cmdBeginInfo, VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO);

VkClearValue clearValues[2];
clearValues[0].color        = { {0.2f, 0.2f, 0.2f, 1.0f} };
clearValues[1].depthStencil = { 1.0f, 0 };

int32 fwidth  = GetVulkanRHI()->GetSwapChain()->GetWidth();
int32 fheight = GetVulkanRHI()->GetSwapChain()->GetHeight();

VkRenderPassBeginInfo renderPassBeginInfo;
ZeroVulkanStruct(renderPassBeginInfo, VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO);
renderPassBeginInfo.renderPass      = m_RenderPass;
renderPassBeginInfo.clearValueCount = 2;
renderPassBeginInfo.pClearValues    = clearValues;
renderPassBeginInfo.renderArea.offset.x = 0;
renderPassBeginInfo.renderArea.offset.y = 0;
renderPassBeginInfo.renderArea.extent.width  = fwidth;
renderPassBeginInfo.renderArea.extent.height = fheight;

for (int32 i = 0; i < m_CommandBuffers.size(); ++i)
{
    renderPassBeginInfo.framebuffer = m_FrameBuffers[i];
    
    VkViewport viewport = {};
    viewport.x        = 0;
    viewport.y        = fheight;
    viewport.width    = (float)fwidth;
    viewport.height   = -(float)fheight;    // flip y axis
    viewport.minDepth = 0.0f;
    viewport.maxDepth = 1.0f;
    
    VkRect2D scissor = {};
    scissor.extent.width  = fwidth;
    scissor.extent.height = fheight;
    scissor.offset.x      = 0;
    scissor.offset.y      = 0;
    
    VkDeviceSize offsets[1] = { 0 };
    
    VERIFYVULKANRESULT(vkBeginCommandBuffer(m_CommandBuffers[i], &cmdBeginInfo));
    
    vkCmdBeginRenderPass(m_CommandBuffers[i], &renderPassBeginInfo, VK_SUBPASS_CONTENTS_INLINE);
    vkCmdSetViewport(m_CommandBuffers[i], 0, 1, &viewport);
    vkCmdSetScissor(m_CommandBuffers[i], 0, 1, &scissor);
    vkCmdBindDescriptorSets(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_PipelineLayout, 0, 1, &m_DescriptorSet, 0, nullptr);
    vkCmdBindPipeline(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_Pipeline);
    vkCmdBindVertexBuffers(m_CommandBuffers[i], 0, 1, &m_VertexBuffer.buffer, offsets);
    vkCmdBindIndexBuffer(m_CommandBuffers[i], m_IndicesBuffer.buffer, 0, VK_INDEX_TYPE_UINT16);
    vkCmdDrawIndexed(m_CommandBuffers[i], m_IndicesCount, 1, 0, 0, 0);
    vkCmdEndRenderPass(m_CommandBuffers[i]);
    
    VERIFYVULKANRESULT(vkEndCommandBuffer(m_CommandBuffers[i]));
}
```
上述代码里面有一个地方需要注意，就是视口的坐标以及高度我们配置的有点奇怪。这是因为我们开启了`设备扩展`，目的是用来翻转`DNC`空间下的Y轴，翻转了之后Viewport的配置就必须得进行相应的更改。
```C++
VkViewport viewport = {};
viewport.x        = 0;
viewport.y        = fheight;
viewport.width    = (float)fwidth;
viewport.height   = -(float)fheight;    // flip y axis
viewport.minDepth = 0.0f;
viewport.maxDepth = 1.0f;
```

以上就是通过Vulkan绘制一个三角形所必须的一些步奏。在这里我并没有详细的罗列解析每一个步骤。其实对照源码即可看出每一个关键步骤的具体情况，另外就是一些博客非常非常详细的解释了如何创建一个三角形，比如这个翻译的国外的教程，非常详细[黑桃花Vulkan翻译教程](https://www.cnblogs.com/heitao/p/6882815.html)。

## 其它
- [内存关系](http://xiaopengyou.fun/public/2019/06/07/VulkanBuffer%E4%B8%8E%E5%86%85%E5%AD%98/#more)
- [物理设备与簇](http://xiaopengyou.fun/public/2019/06/03/%E7%89%A9%E7%90%86%E8%AE%BE%E5%A4%87%E4%B8%8E%E7%B0%87/)