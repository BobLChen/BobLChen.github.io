---
title: 14_TextureArray
date: 2019-08-19 23:21:32
tags:
- Vulkan
- 物理设备
- 3D
- Demo
categories:
- Vulkan
---

[14_TextureArray](https://github.com/BobLChen/VulkanDemos/tree/master/examples/14_TextureArray)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

在[11_Texture](https://github.com/BobLChen/VulkanDemos/tree/master/examples/11_Texture)这个Demo里面，我们了解到如何从磁盘加载图片并赋予给模型使用。在那个Demo里面，我使用的是多张Texture2D，但是其实可以像数组那样来使用Texture。在某些情况下可以通过这种方式来进行一些优化，例如地形系统。地形一般会使用到许多图片来进行互相融合以丰富地表，但是并不是是说可以使用任意数量的Texture，因为Texture的寄存器根据设备不同只有8个或者16个。在这种情况下，通过TextureArray可以满足这种特定的需求。

<!-- more -->

![14_TextureArray](https://raw.githubusercontent.com/BobLChen/VulkanTutorials/master/preview/14_TextureArray.jpg)

## TextureArray

TextureArray也并不是毫无限制的进行使用。

- 它的数量根据设备不同有不同的数量限制，具体数量限制可以通过**VkPhysicalDeviceLimits.maxImageArrayLayers**查询得到，我的设备是可以使用到2048个的，这个数量看起来是非常可观的。
- 尺寸必须要求全部保持一致。

这些限制其实算不上一些特殊限制，在大多数情况下都可以利用上TextuerArray，例如地形、角色贴图、UI图集等。

## 创建TextuerArray

在之前就已经简单封装过Texture2D的功能，TextureArray其实只是比Texture2D多了一个维度，继续在**DVKTexture**里面进行封装TextuerArray功能。

```c++
static DVKTexture* Create2DArray(
    const std::vector<std::string> filenames, 
    std::shared_ptr<VulkanDevice> vulkanDevice, 
    DVKCommandBuffer* cmdBuffer,
    ImageLayoutBarrier imageLayout = ImageLayoutBarrier::PixelShaderRead
);
```

因为是Array，所以传入的文件也从之前的单文件变成了文件数组。首先将所有的文件都加载进来：

```c++
struct ImageInfo
{
    int32   width  = 0;
    int32   height = 0;
    int32   comp   = 0;
    uint8*  data   = nullptr;
    uint32  size   = 0;
};

// 加载图集数据
std::vector<ImageInfo> images(filenames.size());
for (int32 i = 0; i < filenames.size(); ++i) 
{
    uint32 dataSize = 0;
    uint8* dataPtr  = nullptr;
    if (!FileManager::ReadFile(filenames[i], dataPtr, dataSize)) 
    {
        MLOGE("Failed load image : %s", filenames[i].c_str());
        return nullptr;
    }

    ImageInfo& imageInfo = images[i];
    imageInfo.data = StbImage::LoadFromMemory(dataPtr, dataSize, &imageInfo.width, &imageInfo.height, &imageInfo.comp, 4);
    imageInfo.comp = 4;
    imageInfo.size = imageInfo.width * imageInfo.height * imageInfo.comp;

    delete[] dataPtr;
    dataPtr = nullptr;

    if (!imageInfo.data) 
    {
        MLOGE("Failed load image : %s", filenames[i].c_str());
        return nullptr;
    }
}
```

加载完毕之后，获取一下图片的尺寸，然后准备一下Array的长度，数据格式以及Mipmap数量。

```c++
// 图片信息，TextureArray要求尺寸一致
int32 width     = images[0].width;
int32 height    = images[0].height;
int32 numArray  = images.size();
VkFormat format = VK_FORMAT_R8G8B8A8_UNORM;
int32 mipLevels = MMath::FloorToInt(MMath::Log2(MMath::Max(width, height))) + 1;
VkDevice device = vulkanDevice->GetInstanceHandle();
```

接下来的工作其实跟之前的很类似，就是创建一个足够容纳数据的**staging buffer**用来做中转。由于我们这里是TextureArray，那么**staging buffer**也要足够长，并且也要分段保存每一个Texture的图像数据。

```c++
// 准备stagingBuffer
DVKBuffer* stagingBuffer = DVKBuffer::CreateBuffer(
    vulkanDevice, 
    VK_BUFFER_USAGE_TRANSFER_SRC_BIT, 
    VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, 
    width * height * 4 * numArray
);

for (int32 i = 0; i < images.size(); ++i) 
{
    uint8* src  = images[i].data;
    uint32 size = width * height * 4;
    stagingBuffer->Map(size, size * i);
    stagingBuffer->CopyFrom(src, size);
    stagingBuffer->UnMap();

    StbImage::Free(src);
}
```

然后创建Image，创建Image的时候需要指定一下Array的长度。

```c++
// 创建image
VkImageCreateInfo imageCreateInfo;
ZeroVulkanStruct(imageCreateInfo, VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO);
imageCreateInfo.imageType       = VK_IMAGE_TYPE_2D;
imageCreateInfo.format          = format;
imageCreateInfo.mipLevels       = mipLevels;
imageCreateInfo.arrayLayers     = numArray;
imageCreateInfo.samples         = VK_SAMPLE_COUNT_1_BIT;
imageCreateInfo.tiling          = VK_IMAGE_TILING_OPTIMAL;
imageCreateInfo.sharingMode     = VK_SHARING_MODE_EXCLUSIVE;
imageCreateInfo.initialLayout   = VK_IMAGE_LAYOUT_UNDEFINED;
imageCreateInfo.extent          = { (uint32_t)width, (uint32_t)height, 1 };
imageCreateInfo.usage           = VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_TRANSFER_SRC_BIT | VK_IMAGE_USAGE_SAMPLED_BIT;
VERIFYVULKANRESULT(vkCreateImage(device, &imageCreateInfo, VULKAN_CPU_ALLOCATOR, &image));
```

后面就是一些分配Memory，绑定等工作跟之前的一样。我们从staging buffer拷贝到Image的时候，需要指定一下每一个Layer的尺寸以及Offset。

```c++
// start record
cmdBuffer->Begin();

VkImageSubresourceRange subresourceRange = {};
subresourceRange.aspectMask     = VK_IMAGE_ASPECT_COLOR_BIT;
subresourceRange.levelCount     = 1;
subresourceRange.layerCount     = numArray;
subresourceRange.baseMipLevel   = 0;
subresourceRange.baseArrayLayer = 0;

ImagePipelineBarrier(cmdBuffer->cmdBuffer, image, ImageLayoutBarrier::Undefined, ImageLayoutBarrier::TransferDest, subresourceRange);

std::vector<VkBufferImageCopy> bufferCopyRegions;
for (int32 i = 0; i < images.size(); ++i)
{
    VkBufferImageCopy bufferCopyRegion = {};
    bufferCopyRegion.imageSubresource.aspectMask     = VK_IMAGE_ASPECT_COLOR_BIT;
    bufferCopyRegion.imageSubresource.mipLevel       = 0;
    bufferCopyRegion.imageSubresource.baseArrayLayer = i;
    bufferCopyRegion.imageSubresource.layerCount     = 1;
    bufferCopyRegion.imageExtent.width  = width;
    bufferCopyRegion.imageExtent.height = height;
    bufferCopyRegion.imageExtent.depth  = 1;
    bufferCopyRegion.bufferOffset       = width * height * 4 * i;
    bufferCopyRegions.push_back(bufferCopyRegion);
}

vkCmdCopyBufferToImage(cmdBuffer->cmdBuffer, stagingBuffer->buffer, image, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, bufferCopyRegions.size(), bufferCopyRegions.data());

ImagePipelineBarrier(cmdBuffer->cmdBuffer, image, ImageLayoutBarrier::TransferDest, ImageLayoutBarrier::TransferSource, subresourceRange);
```

拷贝完成之后，继续Mipmap的生成，这个过程跟之前的一样。

```c++
// Generate the mip chain
for (uint32_t i = 1; i < mipLevels; i++) 
{
    VkImageBlit imageBlit = {};

    imageBlit.srcSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    imageBlit.srcSubresource.layerCount = numArray;
    imageBlit.srcSubresource.mipLevel   = i - 1;
    imageBlit.srcOffsets[1].x = int32_t(width  >> (i - 1));
    imageBlit.srcOffsets[1].y = int32_t(height >> (i - 1));
    imageBlit.srcOffsets[1].z = 1;

    imageBlit.dstSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    imageBlit.dstSubresource.layerCount = numArray;
    imageBlit.dstSubresource.mipLevel   = i;
    imageBlit.dstOffsets[1].x = int32_t(width  >> i);
    imageBlit.dstOffsets[1].y = int32_t(height >> i);
    imageBlit.dstOffsets[1].z = 1;

    VkImageSubresourceRange mipSubRange = {};
    mipSubRange.aspectMask     = VK_IMAGE_ASPECT_COLOR_BIT;
    mipSubRange.baseMipLevel   = i;
    mipSubRange.levelCount     = 1;
    mipSubRange.layerCount     = numArray;
    mipSubRange.baseArrayLayer = 0;

    ImagePipelineBarrier(cmdBuffer->cmdBuffer, image, ImageLayoutBarrier::Undefined, ImageLayoutBarrier::TransferDest, mipSubRange);

    vkCmdBlitImage(cmdBuffer->cmdBuffer, image, VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL, image, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, 1, &imageBlit, VK_FILTER_LINEAR);

    ImagePipelineBarrier(cmdBuffer->cmdBuffer, image, ImageLayoutBarrier::TransferDest, ImageLayoutBarrier::TransferSource, mipSubRange);
}

subresourceRange.aspectMask   = VK_IMAGE_ASPECT_COLOR_BIT;
subresourceRange.levelCount   = mipLevels;
subresourceRange.layerCount   = numArray;
subresourceRange.baseMipLevel = 0;

ImagePipelineBarrier(cmdBuffer->cmdBuffer, image, ImageLayoutBarrier::TransferSource, imageLayout, subresourceRange);

cmdBuffer->End();
cmdBuffer->Submit();
```

## 创建ImageView

ImageView的创建过程也跟之前的类似，只是Layer的数量需要进行修正。

```c++
VkImageViewCreateInfo viewInfo;
ZeroVulkanStruct(viewInfo, VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO);
viewInfo.viewType = VK_IMAGE_VIEW_TYPE_2D_ARRAY;
viewInfo.image    = image;
viewInfo.format   = format;
viewInfo.components = { VK_COMPONENT_SWIZZLE_R, VK_COMPONENT_SWIZZLE_G, VK_COMPONENT_SWIZZLE_B, VK_COMPONENT_SWIZZLE_A };
viewInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
viewInfo.subresourceRange.layerCount = numArray;
viewInfo.subresourceRange.levelCount = mipLevels;
VERIFYVULKANRESULT(vkCreateImageView(device, &viewInfo, VULKAN_CPU_ALLOCATOR, &imageView));
```

## 设置混合数据

光有贴图还不行，还需要额外的数据进行混合的操作，在这个Demo里面，我就偷懒一下，把混合数据存储到顶点属性里面。但是在真实开发场景中，绝大多数都是通过一张或者多张额外的贴图来进行混合，将不同层级的混合数据存储到贴图的不同通道里面，虽然多了一张贴图，但是非常易于修改。顶点属性设置如下：

```c++
// 为顶点数据生成混合数据。PS:这里一般使用贴图来进行混合，而不是顶点数据。
for (int32 i = 0; i < m_Model->meshes.size(); ++i)
{
    vk_demo::DVKMesh* mesh = m_Model->meshes[i];
    for (int32 j = 0; j < mesh->primitives.size(); ++j)
    {
        vk_demo::DVKPrimitive* primitive = mesh->primitives[j];
        int32 stride = primitive->vertices.size() / primitive->vertexCount;
        for (int32 v = 0; v < primitive->vertexCount; ++v)
        {
            float vy = primitive->vertices[v * stride + 1];

            float& tex0Index = primitive->vertices[v * stride + stride - 4];
            float& tex0Alpha = primitive->vertices[v * stride + stride - 3];
            float& tex1Index = primitive->vertices[v * stride + stride - 2];
            float& tex1Alpha = primitive->vertices[v * stride + stride - 1];

            const float snowLine    = 0.8f;
            const float terrainLine = 0.3f;
            const float rocksLine   = -0.1f;

            const float snowTerrainBandSize  = 3.0f;
            const float terrainRocksBandSize = 0.5f;
            const float rocksLavaBandSize    = 1.f;

            if (vy >= snowLine)
            {
                tex0Index = 0.0f;
                tex1Index = 1.0f;
                tex0Alpha = MMath::Min(1.0f, (vy - snowLine) / snowTerrainBandSize);
                tex1Alpha = 1.0f - tex0Alpha;
            }
            else if (vy >= terrainLine)
            {
                tex0Index = 2.0f;
                tex1Index = 1.0f;
                tex1Alpha = MMath::Min(1.0f, (vy - terrainLine) / terrainRocksBandSize);
                tex0Alpha = 1.0f - tex1Alpha;
            }
            else if (vy >= rocksLine)
            {
                tex0Index = 2.0f;
                tex1Index = 3.0f;
                tex0Alpha = MMath::Min(1.0f, (vy - rocksLine) / rocksLavaBandSize);
                tex1Alpha = 1.0f - tex0Alpha;
            }
            else
            {
                tex0Index = 2.0f;
                tex1Index = 3.0f;
                tex0Alpha = 0.0f;
                tex1Alpha = 1.0f;
            }
        }
    }
}
```

## 在Shader中混合

准备好以上的数据之后，只需要根据混合权值在Fragment Shader中进行混合即可。

```c++
#version 450

layout (location = 0) in vec2 inUV0;
layout (location = 1) in vec3 inNormal;
layout (location = 2) in vec3 inTangent;
layout (location = 3) in vec3 inBiTangent;
layout (location = 4) in vec4 inCustom;

layout (binding = 2) uniform sampler2DArray diffuseMap;

layout (binding = 3) uniform ParamBlock
{
    float step;
    float debug;
    float padding0;
    float padding1;
} params;

layout (location = 0) out vec4 outFragColor;

void main() 
{
    // make tbn
    mat3 TBN = mat3(inTangent, inBiTangent, inNormal);

    vec4 normal0 = texture(diffuseMap, vec3(inUV0, inCustom.x + params.step * 1)) * inCustom.y;
    vec4 normal1 = texture(diffuseMap, vec3(inUV0, inCustom.z + params.step * 1)) * inCustom.w;
    vec3 normal  = normal0.xyz + normal1.xyz;
    normal = normalize(2.0 * normal - vec3(1.0, 1.0, 1.0));
    normal = TBN * normal;

    vec4 finalColor = vec4(0, 0, 0, 0);

    if (params.debug <= 0)
    {
        vec4 albedo0 = texture(diffuseMap, vec3(inUV0, inCustom.x)) * inCustom.y;
        vec4 albedo1 = texture(diffuseMap, vec3(inUV0, inCustom.z)) * inCustom.w;
        finalColor = albedo0 + albedo1;
    }
    else
    {
        vec4 albedo0 = texture(diffuseMap, vec3(inUV0, inCustom.x + params.step * 2)) * inCustom.y;
        vec4 albedo1 = texture(diffuseMap, vec3(inUV0, inCustom.z + params.step * 2)) * inCustom.w;
        finalColor = albedo0 + albedo1;
    }

    float NDotL = clamp(dot(normal, vec3(0, 1, 0)), 0, 1.0);
    finalColor = finalColor * NDotL;
    outFragColor = finalColor;
}
```

最终的效果也就是封面图的效果，我省略了一些步骤，这些步骤都是之前Demo里面的，查看源码即可。