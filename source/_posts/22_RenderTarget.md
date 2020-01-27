---
title: 22_RenderTarget
date: 2019-10-05 22:31:10
tags:
- Vulkan
- 3D
- RenderTarget
- Tutorial
categories:
- Vulkan
---

[22_RenderTarget](https://github.com/BobLChen/VulkanDemos/tree/master/examples/22_RenderTarget)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

今天来看一个比较好玩儿的东西，模板缓冲技术。模板缓冲技术用的地方比较多。例如UI的遮罩功能，部分渲染的性能优化，特效等等。在这个Demo里面通过一个简单的特效来展示Stencil的用法。

<!-- more -->

![22_RenderTarget](https://raw.githubusercontent.com/BobLChen/VulkanDemos/master/preview/22_RenderTarget.gif)

## RenderTarget

附件的用法前面几个Demo已经展示过了，但是我想在这个Demo里面复习一下用法。在下个Demo里面对它进行简单封装一下，方便使用，因为后续的一些Demo可能会大量用到。在3D引擎里面有一个RenderTarget的概念，可以理解为输出目标。附件其实就是RenderTarget，下个Demo就会把附件封装为RenderTarget。

这里我们就把前面Demo创建附件的代码拷贝过来，代码如下：

```c++
void CreateRenderTarget()
{
    m_RenderTarget.device = m_Device;
    m_RenderTarget.width  = m_FrameWidth;
    m_RenderTarget.height = m_FrameHeight;

    m_RenderTarget.color = vk_demo::DVKTexture::CreateRenderTarget(
        m_VulkanDevice,
        PixelFormatToVkFormat(GetVulkanRHI()->GetPixelFormat(), false), 
        VK_IMAGE_ASPECT_COLOR_BIT,
        m_FrameWidth, m_FrameHeight,
        VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_SAMPLED_BIT
    );
    
    m_RenderTarget.depth = vk_demo::DVKTexture::CreateRenderTarget(
        m_VulkanDevice,
        PixelFormatToVkFormat(m_DepthFormat, false),
        VK_IMAGE_ASPECT_DEPTH_BIT,
        m_FrameWidth, m_FrameHeight,
        VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT | VK_IMAGE_USAGE_SAMPLED_BIT
    );
    
    std::vector<VkAttachmentDescription> attchmentDescriptions(2);
    // Color attachment
    attchmentDescriptions[0].format         = m_RenderTarget.color->format;
    attchmentDescriptions[0].samples        = VK_SAMPLE_COUNT_1_BIT;
    attchmentDescriptions[0].loadOp         = VK_ATTACHMENT_LOAD_OP_CLEAR;
    attchmentDescriptions[0].storeOp        = VK_ATTACHMENT_STORE_OP_STORE;
    attchmentDescriptions[0].stencilLoadOp  = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
    attchmentDescriptions[0].stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
    attchmentDescriptions[0].initialLayout  = VK_IMAGE_LAYOUT_UNDEFINED;
    attchmentDescriptions[0].finalLayout    = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
    // Depth attachment
    attchmentDescriptions[1].format         = m_RenderTarget.depth->format;
    attchmentDescriptions[1].samples        = VK_SAMPLE_COUNT_1_BIT;
    attchmentDescriptions[1].loadOp         = VK_ATTACHMENT_LOAD_OP_CLEAR;
    attchmentDescriptions[1].storeOp        = VK_ATTACHMENT_STORE_OP_STORE;
    attchmentDescriptions[1].stencilLoadOp  = VK_ATTACHMENT_LOAD_OP_CLEAR;
    attchmentDescriptions[1].stencilStoreOp = VK_ATTACHMENT_STORE_OP_STORE;
    attchmentDescriptions[1].initialLayout  = VK_IMAGE_LAYOUT_UNDEFINED;
    attchmentDescriptions[1].finalLayout    = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;

    VkAttachmentReference colorReference;
    colorReference.attachment = 0;
    colorReference.layout     = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;

    VkAttachmentReference depthReference;
    depthReference.attachment = 1;
    depthReference.layout     = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;

    VkSubpassDescription subpassDescription = {};
    subpassDescription.pipelineBindPoint       = VK_PIPELINE_BIND_POINT_GRAPHICS;
    subpassDescription.colorAttachmentCount    = 1;
    subpassDescription.pColorAttachments       = &colorReference;
    subpassDescription.pDepthStencilAttachment = &depthReference;

    std::vector<VkSubpassDependency> dependencies(2);
    dependencies[0].srcSubpass      = VK_SUBPASS_EXTERNAL;
    dependencies[0].dstSubpass      = 0;
    dependencies[0].srcStageMask    = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
    dependencies[0].dstStageMask    = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
    dependencies[0].srcAccessMask   = VK_ACCESS_SHADER_READ_BIT;
    dependencies[0].dstAccessMask   = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
    dependencies[0].dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;

    dependencies[1].srcSubpass      = 0;
    dependencies[1].dstSubpass      = VK_SUBPASS_EXTERNAL;
    dependencies[1].srcStageMask    = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
    dependencies[1].dstStageMask    = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
    dependencies[1].srcAccessMask   = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
    dependencies[1].dstAccessMask   = VK_ACCESS_SHADER_READ_BIT;
    dependencies[1].dependencyFlags = VK_DEPENDENCY_BY_REGION_BIT;

    // Create renderpass
    VkRenderPassCreateInfo renderPassInfo;
    ZeroVulkanStruct(renderPassInfo, VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO);
    renderPassInfo.attachmentCount = attchmentDescriptions.size();
    renderPassInfo.pAttachments    = attchmentDescriptions.data();
    renderPassInfo.subpassCount    = 1;
    renderPassInfo.pSubpasses      = &subpassDescription;
    renderPassInfo.dependencyCount = dependencies.size();
    renderPassInfo.pDependencies   = dependencies.data();
    VERIFYVULKANRESULT(vkCreateRenderPass(m_Device, &renderPassInfo, nullptr, &(m_RenderTarget.renderPass)));
    
    VkImageView attachments[2];
    attachments[0] = m_RenderTarget.color->imageView;
    attachments[1] = m_RenderTarget.depth->imageView;

    VkFramebufferCreateInfo frameBufferInfo;
    ZeroVulkanStruct(frameBufferInfo, VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO);
    frameBufferInfo.renderPass      = m_RenderTarget.renderPass;
    frameBufferInfo.attachmentCount = 2;
    frameBufferInfo.pAttachments    = attachments;
    frameBufferInfo.width           = m_RenderTarget.width;
    frameBufferInfo.height          = m_RenderTarget.height;
    frameBufferInfo.layers          = 1;
    VERIFYVULKANRESULT(vkCreateFramebuffer(m_Device, &frameBufferInfo, VULKAN_CPU_ALLOCATOR, &(m_RenderTarget.frameBuffer)));
}
```

## Filter

在这个Demo里面的目的是复习前面的附件知识，所以对附件不在做过多的说明。后处理最简单使用方式就是图像处理，大多数图像处理里面的滤镜都是这样的方式来做的。在这个Demo里面，我将展示大概30种不同的滤镜，这些滤镜也是PS或其它图像软件里面常用的滤镜。

为了方便UI，先将滤镜的种类定义为枚举值，枚举值从0开始，枚举值也当做数组下标使用。定义如下：
```c++
enum ImageFilterType
{
    FilterNormal = 0,
    Filter3x3Convolution,
	FilterBilateralBlur,
	FilterBrightness,
	FilterBulgeDistortion,
	FilterCGAColorspace,
	FilterColorBalance,
	FilterColorInvert,
	FilterColorMatrix,
	FilterContrast,
	FilterCrosshatch,
	FilterDirectionalSobelEdgeDetection,
	FilterExposure,
	FilterFalseColor,
	FilterGamma,
	FilterGlassSphere,
	FilterGrayscale,
	FilterHalftone,
	FilterHaze,
	FilterHighlightShadow,
	FilterHue,
	FilterKuwahara,
	FilterLevels,
	FilterLuminance,
	FilterLuminanceThreshold,
	FilterMonochrome,
	FilterPixelation,
	FilterPosterize,
	FilterSharpen,
	FilterSolarize,
	FilterSphereRefraction,
    FilterCount
};
```

然后在制作滤镜对应的类型，对滤镜进行简单封装一下，可以方便Shader的加载、维护、UI显示。

```c++
struct FilterItem
{
    vk_demo::DVKMaterial*	material;
    vk_demo::DVKShader*		shader;
    ImageFilterType			type;

    void Create(const char* vert, const char* frag, std::shared_ptr<VulkanDevice> vulkanDevice, VkRenderPass renderPass, VkPipelineCache pipelineCache, vk_demo::DVKTexture* rtt)
    {
        shader = vk_demo::DVKShader::Create(
            vulkanDevice,
            true,
            vert,
            frag
        );
        material = vk_demo::DVKMaterial::Create(
            vulkanDevice,
            renderPass,
            pipelineCache,
            shader
        );
        material->PreparePipeline();
        material->SetTexture("inputImageTexture", rtt);
    }
```

最后，就可以加载对应滤镜的Shader，代码如下：

```c++
// ------------------------- Filters -------------------------
m_FilterNames.resize(ImageFilterType::FilterCount);
m_FilterItems.resize(ImageFilterType::FilterCount);
m_FilterSpirvs.resize(ImageFilterType::FilterCount * 2);

#define DefineFilter(FilterType, FilterName) \
m_FilterNames[FilterType] = FilterName; \
m_FilterSpirvs[FilterType * 2 + 0] = "assets/shaders/22_RenderTarget/" FilterName ".vert.spv"; \
m_FilterSpirvs[FilterType * 2 + 1] = "assets/shaders/22_RenderTarget/" FilterName ".frag.spv"; \

DefineFilter(ImageFilterType::FilterNormal,							"Normal");
DefineFilter(ImageFilterType::Filter3x3Convolution,					"Filter3x3Convolution");
DefineFilter(ImageFilterType::FilterBilateralBlur,					"FilterBilateralBlur");
DefineFilter(ImageFilterType::FilterBrightness,						"FilterBrightness");
DefineFilter(ImageFilterType::FilterBulgeDistortion,				"FilterBulgeDistortion");
DefineFilter(ImageFilterType::FilterCGAColorspace,					"FilterCGAColorspace");
DefineFilter(ImageFilterType::FilterColorBalance,					"FilterColorBalance");
DefineFilter(ImageFilterType::FilterColorInvert,					"FilterColorInvert");
DefineFilter(ImageFilterType::FilterColorMatrix,					"FilterColorMatrix");
DefineFilter(ImageFilterType::FilterContrast,						"FilterContrast");
DefineFilter(ImageFilterType::FilterCrosshatch,						"FilterCrosshatch");
DefineFilter(ImageFilterType::FilterDirectionalSobelEdgeDetection,	"FilterDirectionalSobelEdgeDetection");
DefineFilter(ImageFilterType::FilterExposure,						"FilterExposure");
DefineFilter(ImageFilterType::FilterFalseColor,						"FilterFalseColor");
DefineFilter(ImageFilterType::FilterGamma,							"FilterGamma");
DefineFilter(ImageFilterType::FilterGlassSphere,					"FilterGlassSphere");
DefineFilter(ImageFilterType::FilterGrayscale,						"FilterGrayscale");
DefineFilter(ImageFilterType::FilterHalftone,						"FilterHalftone");
DefineFilter(ImageFilterType::FilterHaze,							"FilterHaze");
DefineFilter(ImageFilterType::FilterHighlightShadow,				"FilterHighlightShadow");
DefineFilter(ImageFilterType::FilterHue,							"FilterHue");
DefineFilter(ImageFilterType::FilterKuwahara,						"FilterKuwahara");
DefineFilter(ImageFilterType::FilterLevels,							"FilterLevels");
DefineFilter(ImageFilterType::FilterLuminance,						"FilterLuminance");
DefineFilter(ImageFilterType::FilterLuminanceThreshold,				"FilterLuminanceThreshold");
DefineFilter(ImageFilterType::FilterMonochrome,						"FilterMonochrome");
DefineFilter(ImageFilterType::FilterPixelation,						"FilterPixelation");
DefineFilter(ImageFilterType::FilterPosterize,						"FilterPosterize");
DefineFilter(ImageFilterType::FilterSharpen,						"FilterSharpen");
DefineFilter(ImageFilterType::FilterSolarize,						"FilterSolarize");
DefineFilter(ImageFilterType::FilterSphereRefraction,				"FilterSphereRefraction");

#undef DefineFilter

// 创建Filter
for (int32 i = 0; i < ImageFilterType::FilterCount; ++i)
{
    m_FilterItems[i].Create(
        m_FilterSpirvs[i * 2 + 0],
        m_FilterSpirvs[i * 2 + 1],
        m_VulkanDevice,
        m_RenderPass,
        m_PipelineCache,
        m_RenderTarget.color
    );
}

m_Selected = 0;
```

最后就是一些常规操作，配置一下UI，设定一下参数等等即可。下面放一下常规的一些滤镜的效果：

![0.png](0.png)
![1.png](1.png)
![2.png](2.png)
![3.png](3.png)
![4.png](4.png)
![5.png](5.png)
![6.png](6.png)
![7.png](7.png)
![8.png](8.png)
![9.png](9.png)
![10.png](10.png)
![11.png](11.png)
![12.png](12.png)
![13.png](13.png)
![14.png](14.png)
![15.png](15.png)
![16.png](16.png)
![17.png](17.png)
![18.png](18.png)
![19.png](19.png)
![20.png](20.png)
![21.png](21.png)
![22.png](22.png)
![23.png](23.png)
![24.png](24.png)
![25.png](25.png)
![26.png](26.png)
![27.png](27.png)
![28.png](28.png)
![29.png](29.png)
