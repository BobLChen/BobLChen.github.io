---
title: 6_ImGUI
date: 2019-08-01 20:09:52
tags:
- Vulkan
- 物理设备
- 3D
- Demo
categories:
- Vulkan
---

[ImageGUI](https://github.com/BobLChen/VulkanDemos/tree/master/examples/6_ImageGUI)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

在之前的Demo里面，我对一些需要大量编码的供进行简单封装。为了方便后续**Demo**的进行，我们急需一个**UI**来进行参数或者功能相关的调节。幸运的是，**Github**上有现成的**UI**库可以使用，该项目叫做[**imgui**](https://github.com/ocornut/imgui)。这个UI库有**Vulkan**的实现[点我点我](https://github.com/ocornut/imgui/blob/master/examples/imgui_impl_vulkan.cpp)



<!-- more -->

<!-- more -->

## ImGUI

ImGUI的集成其实很简单，我们把它的实现抄过来即可。具体的实现我就不再赘述了，因为跟之前的Demo流程是一致的。这里只阐述一下需要注意的几个地方。

### ShaderCode

ImGUI同样需要使用到Vulkan的二进制Shader。

