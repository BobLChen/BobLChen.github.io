---
title: 11_Texture
date: 2019-08-02 22:03:40
tags:
- Vulkan
- 物理设备
- 3D
- Demo
categories:
- Vulkan
---

[Texture](https://github.com/BobLChen/VulkanDemos/tree/master/examples/11_Texture)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

上一个Demo里面封装了Pipeline，然后通过Pipeline显示了三种不同的效果，但是至今为止我们的模型都极为单调。在这个Demo里面将使用Texture来丰富效果。

<!-- more -->

![Texture](https://raw.githubusercontent.com/BobLChen/VulkanTutorials/master/preview/11_Texture.jpg)

## Texture

在使用Texture之前，先对Texture进行封装。创建Texture需要几个基本的要素：
- VkImage
- VkDeviceMemory
- VkImageView
以上是创建Texture必备的要素，除此之外还需要一些其它信息。例如：宽高、深度、Mipmap数量、纹理格式等等。那么依据这些条件，可以设计出头文件如下：

```C++
class DVKTexture
{
public:
    DVKTexture()
    {
        
    }
    
    ~DVKTexture()
    {
        if (imageView != VK_NULL_HANDLE) {
            vkDestroyImageView(device, imageView, VULKAN_CPU_ALLOCATOR);
            imageView = VK_NULL_HANDLE;
        }
        
        if (image != VK_NULL_HANDLE) {
            vkDestroyImage(device, image, VULKAN_CPU_ALLOCATOR);
            image = VK_NULL_HANDLE;
        }
        
        if (imageSampler != VK_NULL_HANDLE) {
            vkDestroySampler(device, imageSampler, VULKAN_CPU_ALLOCATOR);
            imageSampler = VK_NULL_HANDLE;
        }
        
        if (imageMemory != VK_NULL_HANDLE) {
            vkFreeMemory(device, imageMemory, VULKAN_CPU_ALLOCATOR);
            imageMemory = VK_NULL_HANDLE;
        }
    }

    void UpdateSampler(
        VkFilter magFilter = VK_FILTER_LINEAR, 
        VkFilter minFilter = VK_FILTER_LINEAR,
        VkSamplerMipmapMode mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR,
        VkSamplerAddressMode addressModeU = VK_SAMPLER_ADDRESS_MODE_REPEAT, 
        VkSamplerAddressMode addressModeV = VK_SAMPLER_ADDRESS_MODE_REPEAT, 
        VkSamplerAddressMode addressModeW = VK_SAMPLER_ADDRESS_MODE_REPEAT
    );
    
    static DVKTexture* Create2D(const uint8* rgbaData, int32 width, int32 height, std::shared_ptr<VulkanDevice> vulkanDevice, DVKCommandBuffer* cmdBuffer); 

    static DVKTexture* Create2D(const std::string& filename, std::shared_ptr<VulkanDevice> vulkanDevice, DVKCommandBuffer* cmdBuffer);

    static DVKTexture* Create2D(std::shared_ptr<VulkanDevice> vulkanDevice, VkFormat format, VkImageAspectFlags aspect, int32 width, int32 height, VkImageUsageFlags usage);

    static DVKTexture* Create2DArray(const std::vector<std::string> filenames, std::shared_ptr<VulkanDevice> vulkanDevice, DVKCommandBuffer* cmdBuffer);
    
    static DVKTexture* Create3D(VkFormat format, const uint8* rgbaData, int32 size, int32 width, int32 height, int32 depth, std::shared_ptr<VulkanDevice> vulkanDevice, DVKCommandBuffer* cmdBuffer);

public:
    VkDevice						device = nullptr;
    
    VkImage                         image = VK_NULL_HANDLE;
    VkImageLayout                   imageLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    VkDeviceMemory                  imageMemory = VK_NULL_HANDLE;
    VkImageView                     imageView = VK_NULL_HANDLE;
    VkSampler                       imageSampler = VK_NULL_HANDLE;
    VkDescriptorImageInfo           descriptorInfo;
    
    int32                           width = 0;
    int32                           height = 0;
    int32							depth = 1;
    int32                           mipLevels = 0;
    int32							layerCount = 1;
    VkFormat                        format = VK_FORMAT_R8G8B8A8_UNORM;
};
```

## 创建Texture

同模型文件一样，Texture也需要从磁盘上创建。

### stb_image

从磁盘加载图像文件，需要用到一些第三方库。加载到的文件数据并不能直接使用，例如JPG、PNG等这些格式都是压缩格式，各个压缩算法还略有不同。后续我们将使用`stb_image`这个库来读取文件格式。为了方便使用，对`stb_image`进行简单封装：
头文件如下:

```c++
#pragma once

#include "Common/Common.h"

class StbImage
{
public:

	static uint8* LoadFromMemory(const uint8* inBuffer, int32 inSize, int32* outWidth, int32* outHeight, int32* outComp, int32 reqComp);

	static void Free(uint8* data);
};
```

CPP文件如下:

```c++
#include "ImageLoader.h"

#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"

#define STB_IMAGE_WRITE_IMPLEMENTATION
#include "stb_image_write.h"

 #define STB_IMAGE_RESIZE_IMPLEMENTATION
#include "stb_image_resize.h"

uint8* StbImage::LoadFromMemory(const uint8* inBuffer, int32 inSize, int32* outWidth, int32* outHeight, int32* outComp, int32 reqComp)
{
	return stbi_load_from_memory(inBuffer, inSize, outWidth,outHeight, outComp, reqComp);
}

void StbImage::Free(uint8* data)
{
	stbi_image_free(data);
}
```

### 加载图片

加载图片时，有一点儿需要注意，图片数据可能不完整，例如只包含两个或者三个通道的数据，所以加载图像数据的时候需要申请四个通道。让`stb_image`补全余下的通道数据。

```c++
uint32 dataSize = 0;
uint8* dataPtr  = nullptr;
if (!FileManager::ReadFile(filename, dataPtr, dataSize)) {
    MLOGE("Failed load image : %s", filename.c_str());
    return nullptr;
}

int32 comp   = 0;
int32 width  = 0;
int32 height = 0;
uint8* rgbaData = StbImage::LoadFromMemory(dataPtr, dataSize, &width, &height, &comp, 4);

delete[] dataPtr;
dataPtr = nullptr;

if (rgbaData == nullptr) {
    MLOGE("Failed load image : %s", filename.c_str());
    return nullptr;
}
```

### 创建Texture

有了rgba数据之后，即可开始着手创建Texture。

#### 计算Mipmap

常规图像格式并不包含Mipmap的数据，所以加载了图像创建Texture之后，还需要创建Mipmap，以便3D模型在不同的距离时也有比较好的显示效果。Mipmap可以通过宽度以及高度计算出来。

```c++
int32 mipLevels = MMath::FloorToInt(MMath::Log2(MMath::Max(width, height))) + 1;
```

#### Staging Buffer

与IndexBuffer、VertexBuffer一样，Texture也是需要从磁盘->内存->显存这样一个过程。所以需要创建一个StagingBuffer用来中转。

```c++
DVKBuffer* stagingBuffer = DVKBuffer::CreateBuffer(vulkanDevice, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, size);
stagingBuffer->Map();
stagingBuffer->CopyFrom((void*)rgbaData, size);
stagingBuffer->UnMap();
```

#### 创建Image

创建Image非常简单，我们填充好对应的参数即可。需要注意的是`usage`参数，Image在使用时，为了对显卡友好，需要将图像布局转换为`VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`，但是原始排列方式是线性排列，逐像素的。所以为了能够以最佳的方式使用，需要进行转换。那么这个Image既是转化的源，也是转化的目标。

```c++
// 创建image
VkImageCreateInfo imageCreateInfo;
ZeroVulkanStruct(imageCreateInfo, VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO);
imageCreateInfo.imageType       = VK_IMAGE_TYPE_2D;
imageCreateInfo.format          = format;
imageCreateInfo.mipLevels       = mipLevels;
imageCreateInfo.arrayLayers     = 1;
imageCreateInfo.samples         = VK_SAMPLE_COUNT_1_BIT;
imageCreateInfo.tiling          = VK_IMAGE_TILING_OPTIMAL;
imageCreateInfo.sharingMode     = VK_SHARING_MODE_EXCLUSIVE;
imageCreateInfo.initialLayout   = VK_IMAGE_LAYOUT_UNDEFINED;
imageCreateInfo.extent          = { (uint32_t)width, (uint32_t)height, 1 };
imageCreateInfo.usage           = VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_TRANSFER_SRC_BIT | VK_IMAGE_USAGE_SAMPLED_BIT;
VERIFYVULKANRESULT(vkCreateImage(device, &imageCreateInfo, VULKAN_CPU_ALLOCATOR, &image));
```

#### 分配显存

跟Buffer的创建流程一样，Image创建好之后也需要为其分配显存。

```c++
// bind image buffer
vkGetImageMemoryRequirements(device, image, &memReqs);
vulkanDevice->GetMemoryManager().GetMemoryTypeFromProperties(memReqs.memoryTypeBits, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, &memoryTypeIndex);
memAllocInfo.allocationSize  = memReqs.size;
memAllocInfo.memoryTypeIndex = memoryTypeIndex;
VERIFYVULKANRESULT(vkAllocateMemory(device, &memAllocInfo, VULKAN_CPU_ALLOCATOR, &imageMemory));
VERIFYVULKANRESULT(vkBindImageMemory(device, image, imageMemory, 0));
```

#### Layout转换

前面提到为了访问读取速度更快，需要进行布局调整。首先将布局调整到`VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`，这个布局作为传输目标时最佳，矩阵转换有个过程，需要确保在布局转换完成之后才进行数据的拷贝操作，这个**保障**就需要**Barrier**来完成。`Barrier`的目的就是为了同步状态，保证下个操作在使用时数据是处于正确的状态，不然就会出现可能布局还未转换完毕，下个操作确已经开始。

```c++
VkImageSubresourceRange subresourceRange = {};
subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
subresourceRange.levelCount = 1;
subresourceRange.layerCount = 1;

{
    VkImageMemoryBarrier imageMemoryBarrier;
    ZeroVulkanStruct(imageMemoryBarrier, VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER);
    imageMemoryBarrier.oldLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    imageMemoryBarrier.newLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
    imageMemoryBarrier.srcAccessMask = 0;
    imageMemoryBarrier.dstAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
    imageMemoryBarrier.image = image;
    imageMemoryBarrier.subresourceRange = subresourceRange;
    vkCmdPipelineBarrier(cmdBuffer->cmdBuffer, VK_PIPELINE_STAGE_HOST_BIT, VK_PIPELINE_STAGE_TRANSFER_BIT, 0, 0, nullptr, 0, nullptr, 1, &imageMemoryBarrier);
}
```

将Image的布局调整完成之后，即可开始进行拷贝操作。

```c++
VkBufferImageCopy bufferCopyRegion = {};
bufferCopyRegion.imageSubresource.aspectMask     = VK_IMAGE_ASPECT_COLOR_BIT;
bufferCopyRegion.imageSubresource.mipLevel       = 0;
bufferCopyRegion.imageSubresource.baseArrayLayer = 0;
bufferCopyRegion.imageSubresource.layerCount     = 1;
bufferCopyRegion.imageExtent.width  = width;
bufferCopyRegion.imageExtent.height = height;
bufferCopyRegion.imageExtent.depth  = 1;

vkCmdCopyBufferToImage(cmdBuffer->cmdBuffer, stagingBuffer->buffer, image, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, 1, &bufferCopyRegion);
```

数据拷贝完成之后，将Image从`VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`转换为`VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`。这个的目的是需要通过GPU生成Mipmap数据。需要注意的是并不是所有的格式都可以生成Mipmap，大部分压缩格式都不能生成Mipmap。。。

```c++
{
    VkImageMemoryBarrier imageMemoryBarrier;
    ZeroVulkanStruct(imageMemoryBarrier, VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER);
    imageMemoryBarrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
    imageMemoryBarrier.newLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL;
    imageMemoryBarrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
    imageMemoryBarrier.dstAccessMask = VK_ACCESS_TRANSFER_READ_BIT;
    imageMemoryBarrier.image = image;
    imageMemoryBarrier.subresourceRange = subresourceRange;
    vkCmdPipelineBarrier(cmdBuffer->cmdBuffer, VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_TRANSFER_BIT, 0, 0, nullptr, 0, nullptr, 1, &imageMemoryBarrier);
}

```

转换完成之后，开始准备生成Mipmap数据。

```c++
// Generate the mip chain
for (uint32_t i = 1; i < mipLevels; i++) {
    VkImageBlit imageBlit = {};
    
    int32 mip0Width  = MMath::Max(width  >> (i - 1), 1);
    int32 mip0Height = MMath::Max(height >> (i - 1), 1);
    int32 mip1Width  = MMath::Max(width  >> (i - 0), 1);
    int32 mip1Height = MMath::Max(height >> (i - 0), 1);

    imageBlit.srcSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    imageBlit.srcSubresource.layerCount = 1;
    imageBlit.srcSubresource.mipLevel   = i - 1;
    imageBlit.srcOffsets[1].x = int32_t(mip0Width);
    imageBlit.srcOffsets[1].y = int32_t(mip0Height);
    imageBlit.srcOffsets[1].z = 1;
    
    imageBlit.dstSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    imageBlit.dstSubresource.layerCount = 1;
    imageBlit.dstSubresource.mipLevel   = i;
    imageBlit.dstOffsets[1].x = int32_t(mip1Width);
    imageBlit.dstOffsets[1].y = int32_t(mip1Height);
    imageBlit.dstOffsets[1].z = 1;
    
    VkImageSubresourceRange mipSubRange = {};
    mipSubRange.aspectMask   = VK_IMAGE_ASPECT_COLOR_BIT;
    mipSubRange.baseMipLevel = i;
    mipSubRange.levelCount   = 1;
    mipSubRange.layerCount   = 1;
    
    {
        VkImageMemoryBarrier imageMemoryBarrier;
        ZeroVulkanStruct(imageMemoryBarrier, VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER);
        imageMemoryBarrier.oldLayout = VK_IMAGE_LAYOUT_UNDEFINED;
        imageMemoryBarrier.newLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
        imageMemoryBarrier.srcAccessMask = 0;
        imageMemoryBarrier.dstAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
        imageMemoryBarrier.image = image;
        imageMemoryBarrier.subresourceRange = mipSubRange;
        vkCmdPipelineBarrier(cmdBuffer->cmdBuffer, VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_TRANSFER_BIT, 0, 0, nullptr, 0, nullptr, 1, &imageMemoryBarrier);
    }
    
    vkCmdBlitImage(cmdBuffer->cmdBuffer, image, VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL, image, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, 1, &imageBlit, VK_FILTER_LINEAR);
    
    {
        VkImageMemoryBarrier imageMemoryBarrier;
        ZeroVulkanStruct(imageMemoryBarrier, VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER);
        imageMemoryBarrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
        imageMemoryBarrier.newLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL;
        imageMemoryBarrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
        imageMemoryBarrier.dstAccessMask = VK_ACCESS_TRANSFER_READ_BIT;
        imageMemoryBarrier.image = image;
        imageMemoryBarrier.subresourceRange = mipSubRange;
        vkCmdPipelineBarrier(cmdBuffer->cmdBuffer, VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_TRANSFER_BIT, 0, 0, nullptr, 0, nullptr, 1, &imageMemoryBarrier);
    }
}

subresourceRange.levelCount = mipLevels;
```

Mipmap生成完毕之后，就可以将图像布局格式调整为着色器读取最佳。

```c++
{
    VkImageMemoryBarrier imageMemoryBarrier;
    ZeroVulkanStruct(imageMemoryBarrier, VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER);
    imageMemoryBarrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL;
    imageMemoryBarrier.newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
    imageMemoryBarrier.srcAccessMask = VK_ACCESS_TRANSFER_READ_BIT;
    imageMemoryBarrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;
    imageMemoryBarrier.image = image;
    imageMemoryBarrier.subresourceRange = subresourceRange;
    vkCmdPipelineBarrier(cmdBuffer->cmdBuffer, VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT, 0, 0, nullptr, 0, nullptr, 1, &imageMemoryBarrier);
}
```

最后提交命令，删除StagingBuffer。

```c++
cmdBuffer->End();
cmdBuffer->Submit();

delete stagingBuffer;
```

#### 创建Sampler

对Texture采样有不同的算法，有时候需要插值，有时候并不需要，可以为其创建一个默认的采样器。

```c++
VkSamplerCreateInfo samplerInfo;
ZeroVulkanStruct(samplerInfo, VK_STRUCTURE_TYPE_SAMPLER_CREATE_INFO);
samplerInfo.magFilter        = VK_FILTER_LINEAR;
samplerInfo.minFilter        = VK_FILTER_LINEAR;
samplerInfo.mipmapMode       = VK_SAMPLER_MIPMAP_MODE_LINEAR;
samplerInfo.addressModeU     = VK_SAMPLER_ADDRESS_MODE_REPEAT;
samplerInfo.addressModeV     = VK_SAMPLER_ADDRESS_MODE_REPEAT;
samplerInfo.addressModeW     = VK_SAMPLER_ADDRESS_MODE_REPEAT;
samplerInfo.compareOp	     = VK_COMPARE_OP_NEVER;
samplerInfo.borderColor      = VK_BORDER_COLOR_FLOAT_OPAQUE_WHITE;
samplerInfo.maxAnisotropy    = 1.0;
samplerInfo.anisotropyEnable = VK_FALSE;
samplerInfo.maxLod           = 1.0f;
VERIFYVULKANRESULT(vkCreateSampler(device, &samplerInfo, VULKAN_CPU_ALLOCATOR, &imageSampler));
```

#### 创建ImageView

ImageView供Shader使用，ImageView描述了Image的状态。

```
VkImageViewCreateInfo viewInfo;
ZeroVulkanStruct(viewInfo, VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO);
viewInfo.image      = image;
viewInfo.viewType   = VK_IMAGE_VIEW_TYPE_2D;
viewInfo.format     = format;
viewInfo.components = { VK_COMPONENT_SWIZZLE_R, VK_COMPONENT_SWIZZLE_G, VK_COMPONENT_SWIZZLE_B, VK_COMPONENT_SWIZZLE_A };
viewInfo.subresourceRange.aspectMask     = aspect;
viewInfo.subresourceRange.layerCount     = 1;
viewInfo.subresourceRange.levelCount     = 1;
viewInfo.subresourceRange.baseMipLevel   = 0;
viewInfo.subresourceRange.baseArrayLayer = 0;
VERIFYVULKANRESULT(vkCreateImageView(device, &viewInfo, VULKAN_CPU_ALLOCATOR, &imageView));

descriptorInfo.sampler     = imageSampler;
descriptorInfo.imageView   = imageView;
descriptorInfo.imageLayout = imageLayout;
```

## Usage

我们在`LoadAssets`加入加载Texture的逻辑。

### 加载Texture

```c++
void LoadAssets()
{
    vk_demo::DVKCommandBuffer* cmdBuffer = vk_demo::DVKCommandBuffer::Create(m_VulkanDevice, m_CommandPool);

    m_Model = vk_demo::DVKModel::LoadFromFile(
        "assets/models/head.obj",
        m_VulkanDevice,
        cmdBuffer,
        { VertexAttribute::VA_Position, VertexAttribute::VA_UV0, VertexAttribute::VA_Normal, VertexAttribute::VA_Tangent }
    );

    m_TexDiffuse       = vk_demo::DVKTexture::Create2D("assets/textures/head_diffuse.jpg", m_VulkanDevice, cmdBuffer);
    m_TexNormal        = vk_demo::DVKTexture::Create2D("assets/textures/head_normal.png", m_VulkanDevice, cmdBuffer);
    m_TexCurvature     = vk_demo::DVKTexture::Create2D("assets/textures/curvatureLUT.png", m_VulkanDevice, cmdBuffer);
    m_TexPreIntegrated = vk_demo::DVKTexture::Create2D("assets/textures/preIntegratedLUT.png", m_VulkanDevice, cmdBuffer);

    delete cmdBuffer;
}
```

### 创建DescriptorSetLayout

我们在Shader中添加了四个Texture资源，相应的也要为`DescriptorSetLayout`添加对应的信息。

```c++
void CreateDescriptorSetLayout()
{
    VkDescriptorSetLayoutBinding layoutBindings[6] = { };
    layoutBindings[0].binding 			 = 0;
    layoutBindings[0].descriptorType     = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    layoutBindings[0].descriptorCount    = 1;
    layoutBindings[0].stageFlags 		 = VK_SHADER_STAGE_VERTEX_BIT;
    layoutBindings[0].pImmutableSamplers = nullptr;

    layoutBindings[1].binding 			 = 1;
    layoutBindings[1].descriptorType     = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    layoutBindings[1].descriptorCount    = 1;
    layoutBindings[1].stageFlags 		 = VK_SHADER_STAGE_FRAGMENT_BIT;
    layoutBindings[1].pImmutableSamplers = nullptr;

    for (int32 i = 0; i < 4; ++i)
    {
        layoutBindings[2 + i].binding 			 = 2 + i;
        layoutBindings[2 + i].descriptorType     = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
        layoutBindings[2 + i].descriptorCount    = 1;
        layoutBindings[2 + i].stageFlags 		 = VK_SHADER_STAGE_FRAGMENT_BIT;
        layoutBindings[2 + i].pImmutableSamplers = nullptr;
    }
    
    VkDescriptorSetLayoutCreateInfo descSetLayoutInfo;
    ZeroVulkanStruct(descSetLayoutInfo, VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO);
    descSetLayoutInfo.bindingCount = 6;
    descSetLayoutInfo.pBindings    = layoutBindings;
    VERIFYVULKANRESULT(vkCreateDescriptorSetLayout(m_Device, &descSetLayoutInfo, VULKAN_CPU_ALLOCATOR, &m_DescriptorSetLayout));
    
    VkPipelineLayoutCreateInfo pipeLayoutInfo;
    ZeroVulkanStruct(pipeLayoutInfo, VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO);
    pipeLayoutInfo.setLayoutCount = 1;
    pipeLayoutInfo.pSetLayouts    = &m_DescriptorSetLayout;
    VERIFYVULKANRESULT(vkCreatePipelineLayout(m_Device, &pipeLayoutInfo, VULKAN_CPU_ALLOCATOR, &m_PipelineLayout));
}
```

### 创建DescriptorSet

同样也要修改对应的`DescriptorSet`，并将四个Texture关联进去。

```c++
void CreateDescriptorSet()
{
    VkDescriptorPoolSize poolSizes[2];
    poolSizes[0].type            = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    poolSizes[0].descriptorCount = 2;
    poolSizes[1].type            = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
    poolSizes[1].descriptorCount = 4;
    
    VkDescriptorPoolCreateInfo descriptorPoolInfo;
    ZeroVulkanStruct(descriptorPoolInfo, VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO);
    descriptorPoolInfo.poolSizeCount = 2;
    descriptorPoolInfo.pPoolSizes    = poolSizes;
    descriptorPoolInfo.maxSets       = 1;
    VERIFYVULKANRESULT(vkCreateDescriptorPool(m_Device, &descriptorPoolInfo, VULKAN_CPU_ALLOCATOR, &m_DescriptorPool));
    
    VkDescriptorSetAllocateInfo allocInfo;
    ZeroVulkanStruct(allocInfo, VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO);
    allocInfo.descriptorPool     = m_DescriptorPool;
    allocInfo.descriptorSetCount = 1;
    allocInfo.pSetLayouts        = &m_DescriptorSetLayout;
    VERIFYVULKANRESULT(vkAllocateDescriptorSets(m_Device, &allocInfo, &m_DescriptorSet));
    
    VkWriteDescriptorSet writeDescriptorSet;
    
    ZeroVulkanStruct(writeDescriptorSet, VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET);
    writeDescriptorSet.dstSet          = m_DescriptorSet;
    writeDescriptorSet.descriptorCount = 1;
    writeDescriptorSet.descriptorType  = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    writeDescriptorSet.pBufferInfo     = &(m_MVPBuffer->descriptor);
    writeDescriptorSet.dstBinding      = 0;
    vkUpdateDescriptorSets(m_Device, 1, &writeDescriptorSet, 0, nullptr);
    
    ZeroVulkanStruct(writeDescriptorSet, VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET);
    writeDescriptorSet.dstSet          = m_DescriptorSet;
    writeDescriptorSet.descriptorCount = 1;
    writeDescriptorSet.descriptorType  = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    writeDescriptorSet.pBufferInfo     = &(m_ParamBuffer->descriptor);
    writeDescriptorSet.dstBinding      = 1;
    vkUpdateDescriptorSets(m_Device, 1, &writeDescriptorSet, 0, nullptr);

    std::vector<vk_demo::DVKTexture*> textures = { m_TexDiffuse, m_TexNormal, m_TexCurvature, m_TexPreIntegrated };
    for (int32 i = 0; i < 4; ++i)
    {
        ZeroVulkanStruct(writeDescriptorSet, VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET);
        writeDescriptorSet.dstSet          = m_DescriptorSet;
        writeDescriptorSet.descriptorCount = 1;
        writeDescriptorSet.descriptorType  = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
        writeDescriptorSet.pBufferInfo     = nullptr;
        writeDescriptorSet.pImageInfo      = &(textures[i]->descriptorInfo);
        writeDescriptorSet.dstBinding      = 2 + i;
        vkUpdateDescriptorSets(m_Device, 1, &writeDescriptorSet, 0, nullptr);
    }
}
```

### VertexShader
```glsl
#version 450

layout (location = 0) in vec3 inPosition;
layout (location = 1) in vec2 inUV;
layout (location = 2) in vec3 inNormal;
layout (location = 3) in vec4 inTangent;

layout (binding = 0) uniform MVPBlock 
{
	mat4 modelMatrix;
	mat4 viewMatrix;
	mat4 projectionMatrix;
} uboMVP;

layout (location = 0) out vec2 outUV;
layout (location = 1) out vec3 outNormal;
layout (location = 2) out vec3 outTangent;
layout (location = 3) out vec3 outBiTangent;

out gl_PerVertex 
{
    vec4 gl_Position;   
};

void main() 
{
	mat3 normalMatrix = transpose(inverse(mat3(uboMVP.modelMatrix)));

	vec3 normal  = normalize(normalMatrix * inNormal.xyz);
	vec3 tangent = normalize(normalMatrix * inTangent.xyz);
	
	outUV        = inUV;
	outNormal    = normal;
	outTangent   = tangent;
	outBiTangent = cross(normal, tangent) * inTangent.w;
	
	gl_Position  = uboMVP.projectionMatrix * uboMVP.viewMatrix * uboMVP.modelMatrix * vec4(inPosition.xyz, 1.0);
}

```

### FragmentShader
```glsl
#version 450

layout (binding = 1) uniform ParamBlock 
{
	vec3 lightDir;
    float curvature;

    vec3 lightColor;
    float exposure;

    vec2 curvatureScaleBias;
    float blurredLevel;
    float padding;
} params;

layout (location = 0) in vec2 inUV;
layout (location = 1) in vec3 inNormal;
layout (location = 2) in vec3 inTangent;
layout (location = 3) in vec3 inBiTangent;

layout (binding = 2) uniform sampler2D diffuseMap;
layout (binding = 3) uniform sampler2D normalMap;
layout (binding = 4) uniform sampler2D curvatureMap;
layout (binding = 5) uniform sampler2D preIntegratedMap;

layout (location = 0) out vec4 outFragColor;

vec3 EvaluateSSSDiffuseLight(vec3 normal, vec3 blurredNormal)
{
    float blurredNDL      = dot(blurredNormal, normalize(params.lightDir));
	float curvatureScaled = params.curvature * params.curvatureScaleBias.x + params.curvatureScaleBias.y;
	vec2 curvatureUV      = vec2(blurredNDL * 0.5 + 0.5, curvatureScaled);
	vec3 curvatureRGB     = texture(curvatureMap, curvatureUV).rgb * 0.5 - 0.25;
	vec3 normalFactor     = vec3(clamp(1.0 - blurredNDL, 0.0, 1.0));
	
    normalFactor *= normalFactor;

	vec3 normal0 = normalize(mix(normal, blurredNormal, 0.3 + 0.7 * normalFactor));
	vec3 normal1 = normalize(mix(normal, blurredNormal, normalFactor));
	float ndl0   = clamp(dot(normal0, normalize(params.lightDir)), 0.0, 1.0);
	float ndl1   = clamp(dot(normal1, normalize(params.lightDir)), 0.0, 1.0);
	vec3 ndlRGB  = vec3(clamp(blurredNDL, 0.0, 1.0), ndl0, ndl1);
	vec3 rgbSSS  = clamp(curvatureRGB + ndlRGB, 0.0, 1.0);

    return rgbSSS * params.lightColor;
}

vec4 SRGBtoLINEAR(vec4 srgbIn)
{
    vec3 bless  = step(vec3(0.04045), srgbIn.xyz);
	vec3 linOut = mix(srgbIn.xyz / vec3(12.92), pow((srgbIn.xyz + vec3(0.055)) / vec3(1.055), vec3(2.4)), bless);
	return vec4(linOut, srgbIn.w);;
}

vec3 Tonemap(vec3 rgb)
{
	rgb *= params.exposure;
	rgb = max(vec3(0), rgb - 0.004);
    rgb = (rgb * (6.2 * rgb + 0.5)) / (rgb * (6.2 * rgb + 1.7) + 0.06);
	return rgb;
}

void main() 
{
    // make tbn
    mat3 TBN = mat3(inTangent, inBiTangent, inNormal);

    // normal
    vec3 normal = texture(normalMap, inUV).xyz;
    normal = normalize(2.0 * normal - vec3(1.0, 1.0, 1.0));
    normal = TBN * normal;

    // blurredNormal
    vec3 blurredNormal = texture(normalMap, inUV, params.blurredLevel).xyz;
    blurredNormal = normalize(2.0 * blurredNormal - vec3(1.0, 1.0, 1.0));
    blurredNormal = TBN * blurredNormal;

    // diffuse
    vec3 diffuseColor = SRGBtoLINEAR(texture(diffuseMap, inUV)).xyz;

    // light
    vec3 diffuseLight = EvaluateSSSDiffuseLight(normal, blurredNormal);

    outFragColor = vec4(Tonemap(diffuseLight * diffuseColor), 1.0);
}

```

最终的效果就是预览图的效果。