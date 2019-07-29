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
在这之前有一些注意事项需要说明，这些注意事项非常关键，决定着之后一些东西的理解。

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
```
uint32 gpuCount = 0;
VkResult result = vkEnumeratePhysicalDevices(m_Instance, &gpuCount, nullptr);
std::vector<VkPhysicalDevice> physicalDevices(gpuCount);
vkEnumeratePhysicalDevices(m_Instance, &gpuCount, physicalDevices.data());
```
在PC上，有些GPU可能是集成到主板或者CPU上的，有些GPU是独立的，一般来讲独立版本性能会比集成的好一点儿，一般来讲。。。

### 创建逻辑设备
