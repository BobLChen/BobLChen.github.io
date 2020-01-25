---
title: 8_OptimizeVertexIndexBuffer
date: 2019-08-02 01:02:32
tags:
- Vulkan
- 物理设备
- 3D
- Demo
categories:
- Vulkan
---

[Github项目](https://github.com/BobLChen/VulkanDemos)

[Demo源码地址](https://github.com/BobLChen/VulkanDemos/tree/master/examples/8_OptimizeVertexIndexBuffer)

在前面的Demo里面，模型数据的创建全是靠手工编写。对于非常复杂的模型，例如上万面或上万顶点，我们不太可能也进行手工编写。后续的Demo里面势必会有从外部加载模型文件的需求。为了后续Demo在加载外部模型文件时更加方便快捷，在这个Demo里面，对**VertexBuffer**以及**IndexBuffer**进行封装。

<!-- more -->

## IndexBuffer

**IndexBuffer**非常简单，只需要简单封装一下索引的数量、类型、Buffer即可。头文件如下：

```c++
class DVKIndexBuffer
{
    private:
    DVKIndexBuffer()
    {

    }

    public:
    ~DVKIndexBuffer()
    {
        if (dvkBuffer) {
            delete dvkBuffer;
        }
        dvkBuffer = nullptr;
    }

    void BindDraw(VkCommandBuffer cmdBuffer)
    {
        vkCmdBindIndexBuffer(cmdBuffer, dvkBuffer->buffer, 0, indexType);
        vkCmdDrawIndexed(cmdBuffer, indexCount, 1, 0, 0, 0);
    }

    static DVKIndexBuffer* Create(std::shared_ptr<VulkanDevice> vulkanDevice, DVKCommandBuffer* cmdBuffer, std::vector<uint16> indices, VkIndexType type = VK_INDEX_TYPE_UINT16);

    public:
    VkDevice		device = VK_NULL_HANDLE;
    DVKBuffer*		dvkBuffer = nullptr;
    int32			indexCount = 0;
    VkIndexType		indexType = VK_INDEX_TYPE_UINT16;
};
```

需要注意的是**VkIndexType**需要封装进来，因为并不是所有的设备都支持32位的索引类型的。低端设备会支持**VK_INDEX_TYPE_UINT16**，无符号的16位索引最大可以支持到65535个索引，也就是65535/3个三角形。IndexBuffer封装完成了之后，创建的代码我从前面的Demo里面拷贝过来。如下：

```c++
DVKIndexBuffer* DVKIndexBuffer::Create(std::shared_ptr<VulkanDevice> vulkanDevice, DVKCommandBuffer* cmdBuffer, std::vector<uint16> indices, VkIndexType type)
{
    VkDevice device = vulkanDevice->GetInstanceHandle();

    DVKIndexBuffer* indexBuffer = new DVKIndexBuffer();
    indexBuffer->device = device;
    indexBuffer->indexCount = indices.size();

    vk_demo::DVKBuffer* indexStaging = vk_demo::DVKBuffer::CreateBuffer(
        vulkanDevice, 
        VK_BUFFER_USAGE_TRANSFER_SRC_BIT, 
        VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, 
        indices.size() * sizeof(uint16), 
        indices.data()
    );

    indexBuffer->dvkBuffer = vk_demo::DVKBuffer::CreateBuffer(
        vulkanDevice, 
        VK_BUFFER_USAGE_INDEX_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT, 
        VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, 
        indices.size() * sizeof(uint16)
    );

    cmdBuffer->Begin();

    VkBufferCopy copyRegion = {};
    copyRegion.size = indices.size() * sizeof(uint16);

    vkCmdCopyBuffer(cmdBuffer->cmdBuffer, indexStaging->buffer, indexBuffer->dvkBuffer->buffer, 1, &copyRegion);

    cmdBuffer->End();
    cmdBuffer->Submit();

    delete indexStaging;

    return indexBuffer;
}
```

## VertexBuffer

VertexBuffer需要提供的功能会比较多一些，之前的Demo里面在创建**Pipeline**的时候，会有这样一段代码用来描述**顶点**数据结构。

```c++
// (triangle.vert):
// layout (location = 0) in vec3 inPos;
// layout (location = 1) in vec3 inColor;
// Attribute location 0: Position
// Attribute location 1: Color
// vertex input bindding
VkVertexInputBindingDescription vertexInputBinding = {};
vertexInputBinding.binding   = 0; // Vertex Buffer 0
vertexInputBinding.stride    = sizeof(Vertex); // Position + uv
vertexInputBinding.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;

std::vector<VkVertexInputAttributeDescription> vertexInputAttributs(2);
// position
vertexInputAttributs[0].binding  = 0;
vertexInputAttributs[0].location = 0; // diffuse.vert : layout (location = 0)
vertexInputAttributs[0].format   = VK_FORMAT_R32G32B32_SFLOAT;
vertexInputAttributs[0].offset   = 0;
// uv
vertexInputAttributs[1].binding  = 0;
vertexInputAttributs[1].location = 1; // diffuse.vert : layout (location = 1)
vertexInputAttributs[1].format   = VK_FORMAT_R32G32_SFLOAT;
vertexInputAttributs[1].offset   = 12;

VkPipelineVertexInputStateCreateInfo vertexInputState;
ZeroVulkanStruct(vertexInputState, VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO);
vertexInputState.vertexBindingDescriptionCount   = 1;
vertexInputState.pVertexBindingDescriptions      = &vertexInputBinding;
vertexInputState.vertexAttributeDescriptionCount = 2;
vertexInputState.pVertexAttributeDescriptions    = vertexInputAttributs.data();
```

那么在封装**VertexBuffer**的时候，就要考虑到**VertexBuffer**定义了顶点的数据结构，那么它就有必要提供顶点数据结构。

### 顶点数据类型

为了方便处理顶点数据结构，预先定义好一套顶点数据结构的标准出来。

```c++
enum VertexAttribute
{
	VA_None = 0,
	VA_Position,
	VA_UV0,
	VA_UV1,
	VA_Normal,
	VA_Tangent,
	VA_Color,
	VA_SkinWeight,
	VA_SkinIndex,
	VA_Custom0,
	VA_Custom1,
	VA_Custom2,
	VA_Custom3,
	VA_Count,
};
```

定义好顶点数据类型之后，就可以提供相关的工具函数，例如获取顶点数据的长度、顶点数据的类型等等。如下所示：

```c++
inline int32 VertexAttributeToSize(VertexAttribute attribute)
	{
		// count * sizeof(float)
		if (attribute == VertexAttribute::VA_Position) {
			return 3 * sizeof(float);
		}
		else if (attribute == VertexAttribute::VA_UV0) {
			return 2 * sizeof(float);
		}
		else if (attribute == VertexAttribute::VA_UV1) {
			return 2 * sizeof(float);
		}
		else if (attribute == VertexAttribute::VA_Normal) {
			return 3 * sizeof(float);
		}
		else if (attribute == VertexAttribute::VA_Tangent) {
			return 4 * sizeof(float);
		}
		else if (attribute == VertexAttribute::VA_Color) {
			return 3 * sizeof(float);
		}
		else if (attribute == VertexAttribute::VA_SkinWeight) {
			return 4 * sizeof(float);
		}
		else if (attribute == VertexAttribute::VA_SkinIndex) {
			return 4 * sizeof(float);
		}
        else if (attribute == VertexAttribute::VA_Custom0 ||
                 attribute == VertexAttribute::VA_Custom1 ||
                 attribute == VertexAttribute::VA_Custom2 ||
                 attribute == VertexAttribute::VA_Custom3
        )
        {
            return 4 * sizeof(float);
        }
		return 0;
	}
    
	inline VkFormat VertexAttributeToVkFormat(VertexAttribute attribute)
	{
		VkFormat format = VK_FORMAT_R32G32B32_SFLOAT;
		if (attribute == VertexAttribute::VA_Position) {
			format = VK_FORMAT_R32G32B32_SFLOAT;
		}
		else if (attribute == VertexAttribute::VA_UV0) {
			format = VK_FORMAT_R32G32_SFLOAT;
		}
		else if (attribute == VertexAttribute::VA_UV1) {
			format = VK_FORMAT_R32G32_SFLOAT;
		}
		else if (attribute == VertexAttribute::VA_Normal) {
			format = VK_FORMAT_R32G32B32_SFLOAT;
		}
		else if (attribute == VertexAttribute::VA_Tangent) {
			format = VK_FORMAT_R32G32B32A32_SFLOAT;
		}
		else if (attribute == VertexAttribute::VA_Color) {
			format = VK_FORMAT_R32G32B32_SFLOAT;
		}
		else if (attribute == VertexAttribute::VA_SkinWeight) {
			format = VK_FORMAT_R32G32B32A32_SFLOAT;
		}
		else if (attribute == VertexAttribute::VA_SkinIndex) {
			format = VK_FORMAT_R32G32B32A32_SFLOAT;
		}
        else if (attribute == VertexAttribute::VA_Custom0 ||
                 attribute == VertexAttribute::VA_Custom1 ||
                 attribute == VertexAttribute::VA_Custom2 ||
                 attribute == VertexAttribute::VA_Custom3
        )
        {
            format = VK_FORMAT_R32G32B32A32_SFLOAT;
        }
		return format;
	}
```

### 顶点数据结构

顶点的数据可以任意排列，并不影响我们的使用，只要我们提供正确的**VkVertexInputBindingDescription**即可。例如顶点数据可以排列成：**VA_Position**+**VA_Normal**+**VA_UV0**；也可以排列成：**VA_Position**+**VA_UV0**+**VA_Normal**。为了完成这个需求，可以通过使用一个数组来存储顶点数据类型，数组的顺序就是它们的排列方式：

```c++
std::vector<VertexAttribute>	attributes;
```

### VkVertexInputAttributeDescription

既然有了数据类型的信息，那么就可以根据数据类型来产生顶点的描述配置信息。先看一下**VkVertexInputAttributeDescription**的数据结构：

```c++
typedef struct VkVertexInputAttributeDescription {
    uint32_t    location;
    uint32_t    binding;
    VkFormat    format;
    uint32_t    offset;
} VkVertexInputAttributeDescription;
```

其中有**location**和**binding**信息，那么它们指代的是什么意思呢？这里其实隐含了另外一个信息，那就是顶点数据没有要求必须存储到一个**Buffer**里面，它们可以分开存储到多个Buffer之上。例如**VA_Position**存储到**Buffer0**，**VA_Normal**存储到Buffer1，这样也是可以的。由于可以存储到不同的Buffer，所以就需要有信息来指定顶点数据位于哪一个Buffer。**location**其实就是用来干这事的。**binding**很好理解，跟Shader里面的顶点输入数据一一对应。

在本Demo里面，为了方便，不会将数据存储到多个**Buffer**之上。因此**attributes**就描述了一个**Buffer**的结构。那么VkVertexInputAttributeDescription信息也比较好产生了：

```c++
std::vector<VkVertexInputAttributeDescription> DVKVertexBuffer::GetInputAttributes(const std::vector<VertexAttribute>& shaderInputs)
{
    std::vector<VkVertexInputAttributeDescription> vertexInputAttributs;
    int32 offset = 0;
    for (int32 i = 0; i < shaderInputs.size(); ++i)
    {
        VkVertexInputAttributeDescription inputAttribute = {};
        inputAttribute.binding  = 0;
        inputAttribute.location = i;
        inputAttribute.format   = VertexAttributeToVkFormat(shaderInputs[i]);
        inputAttribute.offset   = offset;
        offset += VertexAttributeToSize(shaderInputs[i]);
        vertexInputAttributs.push_back(inputAttribute);
    }
    return vertexInputAttributs;
}
```

这里还有一个需要注意的地方是，本Demo默认**shaderInputs**与**VertexBuffer**中**attributes**一一对应。

### VkVertexInputBindingDescription

虽然提供了VertexBuffer的数据结构信息，上面也提到顶点数据可以分别存储到不同的Buffer，那么每个Buffer上面存储的数据信息也需要进行描述。

```c++
VkVertexInputBindingDescription DVKVertexBuffer::GetInputBinding()
{
    int32 stride = 0;
    for (int32 i = 0; i < attributes.size(); ++i) {
        stride += VertexAttributeToSize(attributes[i]);
    }

    VkVertexInputBindingDescription vertexInputBinding = {};
    vertexInputBinding.binding   = 0;
    vertexInputBinding.stride    = stride;
    vertexInputBinding.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;

    return vertexInputBinding;
}
```

由于我们限制了只是有一个Buffer，并且attributes属性对Shader中的输入数据对应，那么这个的信息也就可以简单的生成了。

### 创建VertexBuffer

创建VertexBuffer的函数没有特别之处，从之前的Demo里面拷贝过来即可。

```c++
DVKVertexBuffer* DVKVertexBuffer::Create(std::shared_ptr<VulkanDevice> vulkanDevice, DVKCommandBuffer* cmdBuffer, std::vector<float> vertices, const std::vector<VertexAttribute>& attributes)
{
    VkDevice device = vulkanDevice->GetInstanceHandle();

    DVKVertexBuffer* vertexBuffer = new DVKVertexBuffer();
    vertexBuffer->device	 = device;
    vertexBuffer->attributes = attributes;

    vk_demo::DVKBuffer* vertexStaging = vk_demo::DVKBuffer::CreateBuffer(
        vulkanDevice, 
        VK_BUFFER_USAGE_TRANSFER_SRC_BIT, 
        VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, 
        vertices.size() * sizeof(float), 
        vertices.data()
    );

    vertexBuffer->dvkBuffer = vk_demo::DVKBuffer::CreateBuffer(
        vulkanDevice, 
        VK_BUFFER_USAGE_VERTEX_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT, 
        VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, 
        vertices.size() * sizeof(float)
    );

    cmdBuffer->Begin();

    VkBufferCopy copyRegion = {};
    copyRegion.size = vertices.size() * sizeof(float);
    vkCmdCopyBuffer(cmdBuffer->cmdBuffer, vertexStaging->buffer, vertexBuffer->dvkBuffer->buffer, 1, &copyRegion);

    cmdBuffer->End();
    cmdBuffer->Submit();

    delete vertexStaging;

    return vertexBuffer;
}
```

## Usage

经过了封装之后，对之前Demo中**CreateMeshBuffer**函数进行优化，如下：

```c++
void CreateMeshBuffers()
{
    std::vector<float> vertices = {
        1.0f,   1.0f, 0.0f, 1.0f, 0.0f, 0.0f,
        -1.0f,  1.0f, 0.0f, 0.0f, 1.0f, 0.0f,
        0.0f,  -1.0f, 0.0f, 0.0f, 0.0f, 1.0f
    };

    std::vector<uint16> indices = { 0, 1, 2 };

    vk_demo::DVKCommandBuffer* cmdBuffer = vk_demo::DVKCommandBuffer::Create(m_VulkanDevice, m_CommandPool);

    m_VertexBuffer = vk_demo::DVKVertexBuffer::Create(m_VulkanDevice, cmdBuffer, vertices, { VertexAttribute::VA_Position, VertexAttribute::VA_Color });
    m_IndexBuffer  = vk_demo::DVKIndexBuffer::Create(m_VulkanDevice, cmdBuffer, indices);

    delete cmdBuffer;
}
```

**Pipeline**创建代码中关于顶点数据的说明也可以从之前的：

```c++
// (triangle.vert):
		// layout (location = 0) in vec3 inPos;
		// layout (location = 1) in vec3 inColor;
		// Attribute location 0: Position
		// Attribute location 1: Color
		// vertex input bindding
		VkVertexInputBindingDescription vertexInputBinding = {};
		vertexInputBinding.binding   = 0; // Vertex Buffer 0
		vertexInputBinding.stride    = sizeof(Vertex); // Position + uv
		vertexInputBinding.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;
        
		std::vector<VkVertexInputAttributeDescription> vertexInputAttributs(2);
		// position
		vertexInputAttributs[0].binding  = 0;
        vertexInputAttributs[0].location = 0; // diffuse.vert : layout (location = 0)
		vertexInputAttributs[0].format   = VK_FORMAT_R32G32B32_SFLOAT;
		vertexInputAttributs[0].offset   = 0;
		// uv
		vertexInputAttributs[1].binding  = 0;
		vertexInputAttributs[1].location = 1; // diffuse.vert : layout (location = 1)
		vertexInputAttributs[1].format   = VK_FORMAT_R32G32_SFLOAT;
		vertexInputAttributs[1].offset   = 12;
		
		VkPipelineVertexInputStateCreateInfo vertexInputState;
		ZeroVulkanStruct(vertexInputState, VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO);
		vertexInputState.vertexBindingDescriptionCount   = 1;
		vertexInputState.pVertexBindingDescriptions      = &vertexInputBinding;
		vertexInputState.vertexAttributeDescriptionCount = 2;
		vertexInputState.pVertexAttributeDescriptions    = vertexInputAttributs.data();
```

优化成如下：

```c++
VkVertexInputBindingDescription vertexInputBinding = m_VertexBuffer->GetInputBinding();
		std::vector<VkVertexInputAttributeDescription> vertexInputAttributs = m_VertexBuffer->GetInputAttributes({ VertexAttribute::VA_Position, VertexAttribute::VA_Color });
		
		VkPipelineVertexInputStateCreateInfo vertexInputState;
		ZeroVulkanStruct(vertexInputState, VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO);
		vertexInputState.vertexBindingDescriptionCount   = 1;
		vertexInputState.pVertexBindingDescriptions      = &vertexInputBinding;
		vertexInputState.vertexAttributeDescriptionCount = 2;
		vertexInputState.pVertexAttributeDescriptions    = vertexInputAttributs.data();
```

总体来说，代码量进一步减少。