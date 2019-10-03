---
title: 15_Texture3D
date: 2019-08-23 23:50:12
tags:
- Vulkan
- 物理设备
- 3D
- Demo
categories:
- Vulkan
---

[15_Texture3D](https://github.com/BobLChen/VulkanDemos/tree/master/examples/15_Texture3D)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

在[14_TextureArray](https://github.com/BobLChen/VulkanDemos/tree/master/examples/14_TextureArray)这个Demo里面，我们了解了如何通过TextureArray技术来实现一个非常简易的地形。如果我们从另外一个维度来想象TextureArray，把**Array**想象成一个新的维度，那之前的TextureArray也就是一个Textue3D的功能。Vulkan API确实给我们提供了Texture3D的功能，在创建Texture3D的时候我们只需要把深度指定上即可。

<!-- more -->

![15_Texture3D](https://raw.githubusercontent.com/BobLChen/VulkanTutorials/master/preview/15_Texture3D.jpg)

## 15_Texture3D

Texture3D一般不会在常规场景中使用。
- 某些设备可能不支持Texture3D
- Texture3D消耗资源太多
- Texture2D可以模拟出Texture3D功能
- 特定需要存储体的数据场景才会使用

虽然使用的情况比较少，但是我们还是有必要来学习一下，因为在某些情况下又是特别有用。如果对摄影比较感兴趣的朋友，可能对LUT这个技术有一定的了解，这个技术在摄影行业里面比较常用。既然在摄影里面比较常用，那么在渲染中一样也是比较常用的技术了，毕竟渲染就是为了模拟现实。LUT说白了就是一个**色调映射**技术，将一个颜色值映射到另一个颜色值上。常规的颜色值由**R、G、B**三个分量组成，每个分量可以由[0, 255]表示，也就是**RGB**可以表示**256x256x256=16777216**，也就是那些手机厂商最喜欢的文案，号称它们的手机支持**1600万**色的原因。

### 色调映射方案

#### 1、直接映射
我们既然想要实现色调映射，那么就必须要解决**16777216**种颜色的映射。我们可以简单粗暴的直接进行映射，用一个Map来进行映射，存储16777216对键值数据。然后我们来计算一下需要的最小内存和显存数据：**16777216x4/1024/=64MB**，由此可以计算出来我们需要64MB的数据。大家可以仔细思考一下，按照我们目前掌握的Demo知识来看，除了Texture可以存储这么庞大的数据以外，并没有其它方案可以解决这个64MB的问题。

### 2、Texture2D
前面提到，根据目前掌握的知识里面只有Texture可以解决这个问题。我们可以通过一张Texture2D来解决这个问题，我们可以对Texture2D进行分块，一小块的尺寸为**256x256**，一行放置**16**块，放置**16**列。这样我们就只需要一张**4096x4096**的图片完成我们的色调映射的功能。即小块**256x256**映射**RG**数据，总的块数量**16x16**映射**B**数据。虽然这样可行，但是大家会发现一个问题，映射起来非常麻烦，得要根据UV值解算出来每一个块的坐标，然后进行映射。

### 3、Texture3D
前面两种方案要么行不通要么麻烦，但是呢我们可以利用Texture3D来简单的实现这个功能。Texture3D就是三个维度**X、Y、Z**也可以理解为**R、G、B**，在Shader里面进行采样的时候，给定的就是三个维度的坐标。那么我们在Shader里面就可以通过texture指令根据**color**值进行采样得到对应存储到Texture3D中的颜色值。

### 准备Texture3D数据
Texture3D数据不能直接从磁盘里面获得了，得要我们自己准备。因为常规的文件格式都是二维的，但是我们现在需要的是三维的数据，在维度上面少了一个维度。

既然我们要做色调映射这个功能，那么我们在准备数据的时候就要准备**RGB**值对应的映射值。我们现在来做一个怀旧风格的映射图。由于**RGB**值的范围是[0, 255]，因此把尺寸设置为**256**，刚好覆盖**RGB**的范围。然后整三个for循环，即可把所有的颜色值的组合都遍历完，在遍历的时候我们进行色调的映射，代码如下：
```c++
int32 lutSize  = 256;
uint8* lutRGBA = new uint8[lutSize * lutSize * 4 * lutSize];
for (int32 x = 0; x < lutSize; ++x)
{
    for (int32 y = 0; y < lutSize; ++y)
    {
        for (int32 z = 0; z < lutSize; ++z)
        {
            int idx = (x + y * lutSize + z * lutSize * lutSize) * 4;
            int32 r = x * 1.0f / (lutSize - 1) * 255;
            int32 g = y * 1.0f / (lutSize - 1) * 255;
            int32 b = z * 1.0f / (lutSize - 1) * 255;
            // 怀旧PS滤镜，色调映射。
            r = 0.393f * r + 0.769f * g + 0.189f * b;
            g = 0.349f * r + 0.686f * g + 0.168f * b;
            b = 0.272f * r + 0.534f * g + 0.131f * b;
            lutRGBA[idx + 0] = MMath::Min(r, 255);
            lutRGBA[idx + 1] = MMath::Min(g, 255);
            lutRGBA[idx + 2] = MMath::Min(b, 255);
            lutRGBA[idx + 3] = 255;
        }
    }
}
```

### 创建Texture3D
我们再来看一下如何创建Textuer3D，创建过程其实跟之前创建Texture2D差异不大，只是多了一个维度。代码如下：
```c++
DVKTexture* DVKTexture::Create3D(VkFormat format, const uint8* rgbaData, int32 size, int32 width, int32 height, int32 depth, std::shared_ptr<VulkanDevice> vulkanDevice, DVKCommandBuffer* cmdBuffer, ImageLayoutBarrier imageLayout)
{
    VkDevice device = vulkanDevice->GetInstanceHandle();

    DVKBuffer* stagingBuffer = DVKBuffer::CreateBuffer(vulkanDevice, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, size);
    stagingBuffer->Map();
    stagingBuffer->CopyFrom((void*)rgbaData, size);
    stagingBuffer->UnMap();
    
    uint32 memoryTypeIndex = 0;
    VkMemoryRequirements memReqs = {};
    VkMemoryAllocateInfo memAllocInfo;
    ZeroVulkanStruct(memAllocInfo, VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO);
    
    // image info
    VkImage                         image = VK_NULL_HANDLE;
    VkDeviceMemory                  imageMemory = VK_NULL_HANDLE;
    VkImageView                     imageView = VK_NULL_HANDLE;
    VkSampler                       imageSampler = VK_NULL_HANDLE;
    VkDescriptorImageInfo           descriptorInfo = {};
    
    // Create optimal tiled target image
    VkImageCreateInfo imageCreateInfo;
    ZeroVulkanStruct(imageCreateInfo, VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO);
    imageCreateInfo.imageType     = VK_IMAGE_TYPE_3D;
    imageCreateInfo.format        = format;
    imageCreateInfo.mipLevels     = 1;
    imageCreateInfo.arrayLayers   = 1;
    imageCreateInfo.samples       = VK_SAMPLE_COUNT_1_BIT;
    imageCreateInfo.tiling        = VK_IMAGE_TILING_OPTIMAL;
    imageCreateInfo.sharingMode   = VK_SHARING_MODE_EXCLUSIVE;
    imageCreateInfo.extent.width  = width;
    imageCreateInfo.extent.height = height;
    imageCreateInfo.extent.depth  = depth;
    imageCreateInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    imageCreateInfo.usage         = VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT;
    VERIFYVULKANRESULT(vkCreateImage(device, &imageCreateInfo, VULKAN_CPU_ALLOCATOR, &image));
    
    // bind image buffer
    vkGetImageMemoryRequirements(device, image, &memReqs);
    vulkanDevice->GetMemoryManager().GetMemoryTypeFromProperties(memReqs.memoryTypeBits, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, &memoryTypeIndex);
    memAllocInfo.allocationSize  = memReqs.size;
    memAllocInfo.memoryTypeIndex = memoryTypeIndex;
    VERIFYVULKANRESULT(vkAllocateMemory(device, &memAllocInfo, VULKAN_CPU_ALLOCATOR, &imageMemory));
    VERIFYVULKANRESULT(vkBindImageMemory(device, image, imageMemory, 0));
    
    cmdBuffer->Begin();
    
    VkImageSubresourceRange subresourceRange = {};
    subresourceRange.aspectMask     = VK_IMAGE_ASPECT_COLOR_BIT;
    subresourceRange.levelCount     = 1;
    subresourceRange.layerCount     = 1;
    subresourceRange.baseMipLevel   = 0;
    subresourceRange.baseArrayLayer = 0;
    
    ImagePipelineBarrier(cmdBuffer->cmdBuffer, image, ImageLayoutBarrier::Undefined, ImageLayoutBarrier::TransferDest, subresourceRange);
    
    VkBufferImageCopy bufferCopyRegion = {};
    bufferCopyRegion.imageSubresource.aspectMask     = VK_IMAGE_ASPECT_COLOR_BIT;
    bufferCopyRegion.imageSubresource.mipLevel       = 0;
    bufferCopyRegion.imageSubresource.baseArrayLayer = 0;
    bufferCopyRegion.imageSubresource.layerCount     = 1;
    bufferCopyRegion.imageExtent.width  = width;
    bufferCopyRegion.imageExtent.height = height;
    bufferCopyRegion.imageExtent.depth  = depth;
    
    vkCmdCopyBufferToImage(cmdBuffer->cmdBuffer, stagingBuffer->buffer, image, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, 1, &bufferCopyRegion);
    
    ImagePipelineBarrier(cmdBuffer->cmdBuffer, image, ImageLayoutBarrier::TransferDest, imageLayout, subresourceRange);
    
    cmdBuffer->End();
    cmdBuffer->Submit();

    delete stagingBuffer;
    
    // Create sampler
    VkSamplerCreateInfo samplerInfo;
    ZeroVulkanStruct(samplerInfo, VK_STRUCTURE_TYPE_SAMPLER_CREATE_INFO);
    samplerInfo.magFilter        = VK_FILTER_LINEAR;
    samplerInfo.minFilter        = VK_FILTER_LINEAR;
    samplerInfo.mipmapMode       = VK_SAMPLER_MIPMAP_MODE_LINEAR;
    samplerInfo.addressModeU     = VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE;
    samplerInfo.addressModeV     = VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE;
    samplerInfo.addressModeW     = VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE;
    samplerInfo.mipLodBias       = 0.0f;
    samplerInfo.compareOp        = VK_COMPARE_OP_NEVER;
    samplerInfo.minLod           = 0.0f;
    samplerInfo.maxLod           = 0.0f;
    samplerInfo.maxAnisotropy    = 1.0;
    samplerInfo.anisotropyEnable = VK_FALSE;
    samplerInfo.borderColor      = VK_BORDER_COLOR_FLOAT_OPAQUE_WHITE;
    VERIFYVULKANRESULT(vkCreateSampler(device, &samplerInfo, VULKAN_CPU_ALLOCATOR, &imageSampler));

    // Create image view
    VkImageViewCreateInfo viewInfo;
    ZeroVulkanStruct(viewInfo, VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO);
    viewInfo.image    = image;
    viewInfo.viewType = VK_IMAGE_VIEW_TYPE_3D;
    viewInfo.format   = format;
    viewInfo.components = {
        VK_COMPONENT_SWIZZLE_R,
        VK_COMPONENT_SWIZZLE_G,
        VK_COMPONENT_SWIZZLE_B,
        VK_COMPONENT_SWIZZLE_A
    };
    viewInfo.subresourceRange.aspectMask     = VK_IMAGE_ASPECT_COLOR_BIT;
    viewInfo.subresourceRange.baseMipLevel   = 0;
    viewInfo.subresourceRange.baseArrayLayer = 0;
    viewInfo.subresourceRange.layerCount     = 1;
    viewInfo.subresourceRange.levelCount     = 1;
    VERIFYVULKANRESULT(vkCreateImageView(device, &viewInfo, VULKAN_CPU_ALLOCATOR, &imageView));

    descriptorInfo.sampler     = imageSampler;
    descriptorInfo.imageView   = imageView;
    descriptorInfo.imageLayout = GetImageLayout(imageLayout);

    DVKTexture* texture   = new DVKTexture();
    texture->descriptorInfo = descriptorInfo;
    texture->format         = format;
    texture->width          = width;
    texture->height         = height;
    texture->depth          = depth;
    texture->image          = image;
    texture->imageLayout    = GetImageLayout(imageLayout);
    texture->imageMemory    = imageMemory;
    texture->imageSampler   = imageSampler;
    texture->imageView      = imageView;
    texture->device			= device;
    texture->mipLevels		= 1;
    texture->layerCount		= 1;

    return texture;
}
```
我们可以看出，只是之前在创建时**depth**设置为了1，现在**depth**设置为了第三个维度的值。

### 其它资源
其它流程跟之前的没有任何区别，都是一样的。只是需要一张Texture2D原始图片用来被映射，一张Texture3D用来做查询图。最终资源的加载代码如下：
```c++
void LoadAssets()
{
    vk_demo::DVKCommandBuffer* cmdBuffer = vk_demo::DVKCommandBuffer::Create(m_VulkanDevice, m_CommandPool);

    m_Model = vk_demo::DVKModel::LoadFromFile(
        "assets/models/plane_z.obj",
        m_VulkanDevice,
        cmdBuffer,
        { 
            VertexAttribute::VA_Position, 
            VertexAttribute::VA_UV0
        }
    );
    
    // 64mb 
    // map image0 -> image1
    int32 lutSize  = 256;
    uint8* lutRGBA = new uint8[lutSize * lutSize * 4 * lutSize];
    for (int32 x = 0; x < lutSize; ++x)
    {
        for (int32 y = 0; y < lutSize; ++y)
        {
            for (int32 z = 0; z < lutSize; ++z)
            {
                int idx = (x + y * lutSize + z * lutSize * lutSize) * 4;
                int32 r = x * 1.0f / (lutSize - 1) * 255;
                int32 g = y * 1.0f / (lutSize - 1) * 255;
                int32 b = z * 1.0f / (lutSize - 1) * 255;
                // 怀旧PS滤镜，色调映射。
                r = 0.393f * r + 0.769f * g + 0.189f * b;
                g = 0.349f * r + 0.686f * g + 0.168f * b;
                b = 0.272f * r + 0.534f * g + 0.131f * b;
                lutRGBA[idx + 0] = MMath::Min(r, 255);
                lutRGBA[idx + 1] = MMath::Min(g, 255);
                lutRGBA[idx + 2] = MMath::Min(b, 255);
                lutRGBA[idx + 3] = 255;
            }
        }
    }
    
    m_TexOrigin = vk_demo::DVKTexture::Create2D("assets/textures/game0.jpg", m_VulkanDevice, cmdBuffer);
    m_Tex3DLut  = vk_demo::DVKTexture::Create3D(VK_FORMAT_R8G8B8A8_UNORM, lutRGBA, lutSize * lutSize * 4 * lutSize, lutSize, lutSize, lutSize, m_VulkanDevice, cmdBuffer);
    
    delete cmdBuffer;
}
```

### Fragment Shader
最后关键的就是Fragment Shader稍微有一点儿区别，我们将采样出来的**颜色值**当成了采样坐标。
```glsl
#version 450

layout (location = 0) in vec2 inUV0;

layout (binding = 1) uniform sampler2D diffuseMap;
layout (binding = 2) uniform sampler3D lutMap;

layout (binding = 3) uniform LutDebugBlock
{
	float   bias;
    vec3    padding;
} uboLutDebug;

layout (location = 0) out vec4 outFragColor;

void main() 
{
    vec4 color = texture(diffuseMap, inUV0);
    outFragColor = texture(lutMap, color.xyz);
}
```
最终的效果就是首页预览图的效果。当然这里有同学会说怀旧的效果完全可以不同这么麻烦，只需要在Fragment Shader里面用数学公式就可以实现。这里只是为了演示这个功能，我们做了一个怀旧的效果。但是真实的做法应该是，我们截一张非常满意的场景图，然后交给美术放到PS里面进行任意的色调修改，最后导出一张色调映射图，我们再进行使用。在这种情况下，色调的映射是按照美术的想法来完成的，绝不是单纯的数学公式可以能够做到的。