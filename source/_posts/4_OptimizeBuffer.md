---
title: Vulkan Demo 4_OptimizeBuffer
date: 2019-08-01 18:25:06
tags:
- Vulkan
- 物理设备
- 3D
- Demo
categories:
- Vulkan
---

[OptimizeBuffer](https://github.com/BobLChen/VulkanDemos/tree/master/examples/4_OptimizeBuffer)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

在[3_DemoBase](http://xiaopengyou.fun/public/2019/08/01/3_DemoBase/#more)这个Demo里面，我们对一些常用的功能和属性进行简单的封装，但是回过头来看一下，会发现整个创建流程还是异常的繁琐，特别是**Buffer**的创建、使用、销毁。仍然本着能偷懒就偷懒的原则，打着优化的旗号实则为了偷懒，有必要对**Buffer**也进行一下简单的封装。

<!-- more -->

## DVKBuffer

为了区分是否属于Demo的功能，统一加一个**DVK**标志，并且放置到**vk_demo**命名空间下。那么头文件代码如下：

```C++
class DVKBuffer
{
    private:
    DVKBuffer()
    {

    }
    public:
    ~DVKBuffer()
    {
        if (buffer != VK_NULL_HANDLE) {
            vkDestroyBuffer(device, buffer, nullptr);
            buffer = VK_NULL_HANDLE;
        }
        if (memory != VK_NULL_HANDLE) {
            vkFreeMemory(device, memory, nullptr);
            memory = VK_NULL_HANDLE;
        }
    }
    public:

    VkDevice				device = VK_NULL_HANDLE;

    VkBuffer				buffer = VK_NULL_HANDLE;
    VkDeviceMemory			memory = VK_NULL_HANDLE;

    VkDescriptorBufferInfo	descriptor;

    VkDeviceSize			size = 0;
    VkDeviceSize			alignment = 0;

    void*					mapped = nullptr;

    VkBufferUsageFlags		usageFlags;
    VkMemoryPropertyFlags	memoryPropertyFlags;

    public:

    static DVKBuffer* CreateBuffer(std::shared_ptr<VulkanDevice> device, VkBufferUsageFlags usageFlags, VkMemoryPropertyFlags memoryPropertyFlags, VkDeviceSize size, void *data = nullptr);

    VkResult Map(VkDeviceSize size = VK_WHOLE_SIZE, VkDeviceSize offset = 0);

    void UnMap();

    VkResult Bind(VkDeviceSize offset = 0);

    void SetupDescriptor(VkDeviceSize size = VK_WHOLE_SIZE, VkDeviceSize offset = 0);

    void CopyFrom(void* data, VkDeviceSize size);

    VkResult Flush(VkDeviceSize size = VK_WHOLE_SIZE, VkDeviceSize offset = 0);

    VkResult Invalidate(VkDeviceSize size = VK_WHOLE_SIZE, VkDeviceSize offset = 0);
};
```

### 屏蔽构造函数

屏蔽构造函数的目的就是为了防止使用者自行的**new**进行创建，要求使用我们提供的**Create**函数进行创建，因为会在这个函数里面做大量的工作。

### 析构时释放资源

既然创建了**Buffer**相关的资源，那么也有义务释放相应的资源，释放的工作就放到析构函数里面。

```c++
~DVKBuffer()
		{
			if (buffer != VK_NULL_HANDLE) {
				vkDestroyBuffer(device, buffer, nullptr);
				buffer = VK_NULL_HANDLE;
			}
			if (memory != VK_NULL_HANDLE) {
				vkFreeMemory(device, memory, nullptr);
				memory = VK_NULL_HANDLE;
			}
		}
```

### Map

如果**Buffer**是在**Host**端且对**GPU**可见，那么就有将数据拷贝到**Buffer**以供GPU使用的需求，在拷贝之前需要将**Buffer**的指针获取到，以便我们能够将数据拷贝进去。

```c++
VkResult DVKBuffer::Map(VkDeviceSize size, VkDeviceSize offset)
	{
		if (mapped) {
			return VK_SUCCESS;
		}
		return vkMapMemory(device, memory, offset, size, 0, &mapped);
	}
```

### UnMap

既然有**Map**操作，那么也就会有**UnMap**操作了。

```C++
void DVKBuffer::UnMap()
	{
		if (!mapped) {
			return;
		}
		vkUnmapMemory(device, memory);
		mapped = nullptr;
	}
```

### CopyFrom

既然有了**Map**操作之后，那么也应该提供一个能够拷贝数据的接口。

```C++
void DVKBuffer::CopyFrom(void* data, VkDeviceSize size)
	{
		if (!mapped) {
			return;
		}
		memcpy(mapped, data, size);
	}
```

### Create

关键的接口已经封装完成，现在需要干的事情就是把前面Demo里面创建Buffer的代码复制粘贴过来。创建Buffer需要**VkDevice**、**VkBufferUsageFlags**、**VkMemoryPropertyFlags**以及**VkDeviceSize**信息。这些信息通过参数传递进来。

```c++
DVKBuffer* DVKBuffer::CreateBuffer(std::shared_ptr<VulkanDevice> vulkanDevice, VkBufferUsageFlags usageFlags, VkMemoryPropertyFlags memoryPropertyFlags, VkDeviceSize size, void *data)
	{
		DVKBuffer* dvkBuffer = new DVKBuffer();
		dvkBuffer->device = vulkanDevice->GetInstanceHandle();
		
		VkDevice vkDevice = vulkanDevice->GetInstanceHandle();
		
		uint32 memoryTypeIndex = 0;
		VkMemoryRequirements memReqs = {};
		VkMemoryAllocateInfo memAlloc;
		ZeroVulkanStruct(memAlloc, VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO);
		
		VkBufferCreateInfo bufferCreateInfo;
		ZeroVulkanStruct(bufferCreateInfo, VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO);
		bufferCreateInfo.usage = usageFlags;
		bufferCreateInfo.size  = size;
		vkCreateBuffer(vkDevice, &bufferCreateInfo, nullptr, &(dvkBuffer->buffer));

		vkGetBufferMemoryRequirements(vkDevice, dvkBuffer->buffer, &memReqs);
		vulkanDevice->GetMemoryManager().GetMemoryTypeFromProperties(memReqs.memoryTypeBits, memoryPropertyFlags, &memoryTypeIndex);
		memAlloc.allocationSize  = memReqs.size;
		memAlloc.memoryTypeIndex = memoryTypeIndex;
		
		vkAllocateMemory(vkDevice, &memAlloc, nullptr, &dvkBuffer->memory);

		dvkBuffer->size       = memAlloc.allocationSize;
		dvkBuffer->alignment  = memReqs.alignment;
		dvkBuffer->usageFlags = usageFlags;
		dvkBuffer->memoryPropertyFlags = memoryPropertyFlags;

		if (data != nullptr)
		{
			dvkBuffer->Map();
			memcpy(dvkBuffer->mapped, data, size);
			if ((memoryPropertyFlags & VK_MEMORY_PROPERTY_HOST_COHERENT_BIT) == 0) {
				dvkBuffer->Flush();
			}
			dvkBuffer->UnMap();
		}

		dvkBuffer->SetupDescriptor();
		dvkBuffer->Bind();

		return dvkBuffer;
	}
```

## Usage

有了封装之后的**DVKBuffer**之后，就可以去掉之前大段重复性的代码。比如**UniformBuffer**的创建就可以优化为:

```c++
void CreateUniformBuffers()
	{
		m_MVPData.model.SetIdentity();
		m_MVPData.model.SetOrigin(Vector3(0, 0, 0));
        
		m_MVPData.view.SetIdentity();
		m_MVPData.view.SetOrigin(Vector4(0, 0, -2.5f));
		m_MVPData.view.SetInverse();
        
		m_MVPData.projection.SetIdentity();
		m_MVPData.projection.Perspective(MMath::DegreesToRadians(75.0f), (float)GetWidth(), (float)GetHeight(), 0.01f, 3000.0f);

		m_MVPBuffer = vk_demo::DVKBuffer::CreateBuffer(
			m_VulkanDevice, 
			VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, 
			VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, 
			sizeof(UBOData),
			&m_MVPData
		);
		m_MVPBuffer->Map();
	}
```

对于**CreateMeshBuffers**则可以优化成如下代码：

```c++
void CreateMeshBuffers()
	{
		std::vector<Vertex> vertices = {
			{ {  1.0f,  1.0f, 0.0f }, { 1.0f, 0.0f, 0.0f } },
			{ { -1.0f,  1.0f, 0.0f }, { 0.0f, 1.0f, 0.0f } },
			{ {  0.0f, -1.0f, 0.0f }, { 0.0f, 0.0f, 1.0f } }
		};
        
		std::vector<uint16> indices = { 0, 1, 2 };
		m_IndicesCount = (uint32)indices.size();
        
		// staging buffer
		vk_demo::DVKBuffer* vertStaging = vk_demo::DVKBuffer::CreateBuffer(
			m_VulkanDevice, 
			VK_BUFFER_USAGE_TRANSFER_SRC_BIT, 
			VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, 
			vertices.size() * sizeof(Vertex), 
			vertices.data()
		);

		vk_demo::DVKBuffer* idexStaging = vk_demo::DVKBuffer::CreateBuffer(
			m_VulkanDevice, 
			VK_BUFFER_USAGE_TRANSFER_SRC_BIT, 
			VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, 
			indices.size() * sizeof(uint16), 
			indices.data()
		);
		
		// reeal buffer
		m_VertexBuffer = vk_demo::DVKBuffer::CreateBuffer(
			m_VulkanDevice, 
			VK_BUFFER_USAGE_VERTEX_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT, 
			VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, 
			vertices.size() * sizeof(Vertex)
		);

		m_IndexBuffer = vk_demo::DVKBuffer::CreateBuffer(
			m_VulkanDevice, 
			VK_BUFFER_USAGE_INDEX_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT, 
			VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, 
			indices.size() * sizeof(uint16)
		);

		VkCommandBuffer xferCmdBuffer;
		// gfx queue自带transfer功能，为了优化需要使用专有的xfer queue。这里为了简单，先将就用。
		VkCommandBufferAllocateInfo xferCmdBufferInfo;
		ZeroVulkanStruct(xferCmdBufferInfo, VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO);
        xferCmdBufferInfo.commandPool        = m_CommandPool;
		xferCmdBufferInfo.level              = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
		xferCmdBufferInfo.commandBufferCount = 1;
		VERIFYVULKANRESULT(vkAllocateCommandBuffers(m_Device, &xferCmdBufferInfo, &xferCmdBuffer));
        
		// 开始录制命令
		VkCommandBufferBeginInfo cmdBufferBeginInfo;
		ZeroVulkanStruct(cmdBufferBeginInfo, VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO);
		VERIFYVULKANRESULT(vkBeginCommandBuffer(xferCmdBuffer, &cmdBufferBeginInfo));
		
		VkBufferCopy copyRegion = {};
		copyRegion.size = vertices.size() * sizeof(Vertex);
		vkCmdCopyBuffer(xferCmdBuffer, vertStaging->buffer, m_VertexBuffer->buffer, 1, &copyRegion);
		
		copyRegion.size = indices.size() * sizeof(uint16);
		vkCmdCopyBuffer(xferCmdBuffer, idexStaging->buffer, m_IndexBuffer->buffer, 1, &copyRegion);
        
		// 结束录制
		VERIFYVULKANRESULT(vkEndCommandBuffer(xferCmdBuffer));
		
		// 提交命令，并且等待命令执行完毕。
		VkSubmitInfo submitInfo;
		ZeroVulkanStruct(submitInfo, VK_STRUCTURE_TYPE_SUBMIT_INFO);
		submitInfo.commandBufferCount = 1;
		submitInfo.pCommandBuffers    = &xferCmdBuffer;

		VkFenceCreateInfo fenceInfo;
		ZeroVulkanStruct(fenceInfo, VK_STRUCTURE_TYPE_FENCE_CREATE_INFO);
		fenceInfo.flags = 0;
        
		VkFence fence = VK_NULL_HANDLE;
		VERIFYVULKANRESULT(vkCreateFence(m_Device, &fenceInfo, VULKAN_CPU_ALLOCATOR, &fence));
		VERIFYVULKANRESULT(vkQueueSubmit(m_GfxQueue, 1, &submitInfo, fence));
		VERIFYVULKANRESULT(vkWaitForFences(m_Device, 1, &fence, VK_TRUE, MAX_int64));
        
		vkDestroyFence(m_Device, fence, VULKAN_CPU_ALLOCATOR);
		vkFreeCommandBuffers(m_Device, m_CommandPool, 1, &xferCmdBuffer);
        
		delete vertStaging;
		delete idexStaging;
	}
```

总体来将，经过了优化的之后的**Buffer**使用起来变得更加简洁，这对于**Buffer**的创建来说，大大的减少了今后的工作量。



