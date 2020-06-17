---
title: 25_Bloom
date: 2019-10-26 22:26:12
tags:
- Vulkan
- 3D
- Bloom
- Tutorial
categories:
- Vulkan
---

[25_Bloom](https://github.com/BobLChen/VulkanDemos/tree/master/examples/25_Bloom)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

Bloom是比较常用的后处理特效，实现的方式都大同小异，这次就实现一个非常建议的Bloom后处理效果。

<!-- more -->

![25_Bloom](https://raw.githubusercontent.com/BobLChen/VulkanDemos/master/preview/25_Bloom.jpg)

## 亮度提取

Bloom效果一般是对亮度比较高的地方进行模糊处理，当然具体的实现不一定非得按照这样的方式来，这种特效完全是按照需求来决定的。既然要对亮度比较高的地方进行模糊处理，那显然需要将亮度比较高的地方提取出来。提取的方法很简单：
```glsl
const vec3 W = vec3(0.2125, 0.7154, 0.0721);
vec4 color = texture(diffuseTexture, inUV0);
float luminance = dot(color.rgb, W);
```

## 模糊

亮度提取完成之后，就需要进行模糊处理。模糊处理常用的方式就是降采样的同时进行一次横向或者纵向的模糊处理，然后再进行一次模糊处理。之所以要进行两次模糊处理，主要是为了优化性能分成了两次。想要一次处理完成其实也是可以的，这需要在一次pass里面大量的进行采样，但是真实情况下不容许我们进行大量的采样。一是因为采样的次数有限不能无限次采样，二是因为采样数量过多性能急剧下降。那这样的话在一次pass里面进行10x10(如果允许的)次采样跟两次pass，横向10次+纵向10次的效果是一致的。

## 合并

我们只是对亮度教高得区域进行了模糊，还需要跟原图进行合并，一般都是给个权值用来影响合并图层的程度。
综上所述，所需要创建的RenderTarget代码如下:

```c++
void CreateRenderTarget()
{
    m_RTColor = vk_demo::DVKTexture::CreateRenderTarget(
        m_VulkanDevice,
        PixelFormatToVkFormat(GetVulkanRHI()->GetPixelFormat(), false), 
        VK_IMAGE_ASPECT_COLOR_BIT,
        m_FrameWidth, m_FrameHeight,
        VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_SAMPLED_BIT
    );
    
    m_RTColorQuater0 = vk_demo::DVKTexture::CreateRenderTarget(
        m_VulkanDevice,
        PixelFormatToVkFormat(GetVulkanRHI()->GetPixelFormat(), false), 
        VK_IMAGE_ASPECT_COLOR_BIT,
        m_FrameWidth / 4.0f, m_FrameHeight / 4.0f,
        VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_SAMPLED_BIT
    );
    
    m_RTColorQuater1 = vk_demo::DVKTexture::CreateRenderTarget(
        m_VulkanDevice,
        PixelFormatToVkFormat(GetVulkanRHI()->GetPixelFormat(), false), 
        VK_IMAGE_ASPECT_COLOR_BIT,
        m_FrameWidth / 4.0f, m_FrameHeight / 4.0f,
        VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_SAMPLED_BIT
    );
    
    m_RTDepth = vk_demo::DVKTexture::CreateRenderTarget(
        m_VulkanDevice,
        PixelFormatToVkFormat(m_DepthFormat, false),
        VK_IMAGE_ASPECT_DEPTH_BIT,
        m_FrameWidth, m_FrameHeight,
        VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT | VK_IMAGE_USAGE_SAMPLED_BIT
    );
    
    // 正常渲染场景
    vk_demo::DVKRenderPassInfo rttNormalInfo(
        m_RTColor, VK_ATTACHMENT_LOAD_OP_CLEAR, VK_ATTACHMENT_STORE_OP_STORE,
        m_RTDepth, VK_ATTACHMENT_LOAD_OP_CLEAR, VK_ATTACHMENT_STORE_OP_STORE
    );
    m_RTTNormal = vk_demo::DVKRenderTarget::Create(m_VulkanDevice, rttNormalInfo);

    // 1/4降级渲染
    vk_demo::DVKRenderPassInfo rttQuater0Info(
        m_RTColorQuater0, VK_ATTACHMENT_LOAD_OP_CLEAR, VK_ATTACHMENT_STORE_OP_STORE, nullptr
    );
    m_RTTQuater0 = vk_demo::DVKRenderTarget::Create(m_VulkanDevice, rttQuater0Info);

    // 1/4降级渲染
    vk_demo::DVKRenderPassInfo rttQuater1Info(
        m_RTColorQuater1, VK_ATTACHMENT_LOAD_OP_CLEAR, VK_ATTACHMENT_STORE_OP_STORE, nullptr
    );
    m_RTTQuater1 = vk_demo::DVKRenderTarget::Create(m_VulkanDevice, rttQuater1Info);

}
```

其它的比较简单，不再细说，最后的效果就如预览图所述，由于模型选择的有问题，效果可能不太明显，大家可以注意观察一下角色的大光头，大光头是有辉光的效果的。