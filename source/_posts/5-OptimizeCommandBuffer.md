---
title: 5_OptimizeCommandBuffer
date: 2019-08-01 19:45:25
tags:
- Vulkan
- 物理设备
- 3D
- Demo
categories:
- Vulkan
---

[OptimizeCommandBuffer](https://github.com/BobLChen/VulkanDemos/tree/master/examples/5_OptimizeCommandBuffer)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

在[4_OptimizeBuffer](http://xiaopengyou.fun/public/2019/08/01/4_OptimizeBuffer/#more)这个Demo里面，我们对**Buffer**进行了封装。封装过的**Buffer**使用起来非常方便，代码量急剧减少。但是在这个Demo里面，我们在优化**CreateMeshBuffers**时，我们会发现虽然**Buffer**的代码量减少了，但是**CommandBuffer**相关的代码任然非常庞大。为了减少代码量，我们需要对**CommandBuffer**也进行简单的封装。

<!-- more -->

## DVKCommandBuffer

任然跟之前的Demo一样，我们单独建个文件保存CommandBuffer相关的功能。头文件如下所示：

```c++
class DVKCommandBuffer
	{
	public:
		~DVKCommandBuffer();

	private:
		DVKCommandBuffer();

	public:
		void Begin();

		void End();

		void Submit(VkSemaphore* signalSemaphore = nullptr);

		static DVKCommandBuffer* Create(std::shared_ptr<VulkanDevice> vulkanDevice, VkCommandPool commandPool, VkCommandBufferLevel level = VK_COMMAND_BUFFER_LEVEL_PRIMARY);

	public:
		VkCommandBuffer						cmdBuffer = VK_NULL_HANDLE;
		VkFence								fence = VK_NULL_HANDLE;
		VkCommandPool						commandPool = VK_NULL_HANDLE;
		std::shared_ptr<VulkanDevice>		vulkanDevice = nullptr;
		std::vector<VkPipelineStageFlags>	waitFlags;
		std::vector<VkSemaphore>			waitSemaphores;

		bool								isBegun;
	};
```

### Begin

CommandBuffer在使用时需要显示的进行**Begin**以及**End**，但是**Begin**的时候需要**VkCommandBufferBeginInfo**参数，同时为了能够清楚的知道**CommandBuffer**的状态，我们也需要记录它是否开始以及结束，因此我们对**Begin**、**End**简单封装一下。

```c++
void DVKCommandBuffer::Begin()
	{
		if (isBegun) {
			return;
		}
		isBegun = true;

		VkCommandBufferBeginInfo cmdBufBeginInfo;
		ZeroVulkanStruct(cmdBufBeginInfo, VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO);
		cmdBufBeginInfo.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;

		vkBeginCommandBuffer(cmdBuffer, &cmdBufBeginInfo);
	}
```

### End

End则比较简单，我们重置一下状态即可。

```c++
void DVKCommandBuffer::End()
	{
		if (!isBegun) {
			return;
		}

		isBegun = false;
		vkEndCommandBuffer(cmdBuffer);
	}
```

### Submit

比较重要的是提交功能，我们需要将录制好的命名提交到**GPU**进行执行。**GPU**的则需要**Fence**来进行同步状态，因此我们的CommandBuffer里面需要自带一个**Fence**。有了**Fence**之后，我们就可以设计我们的**Submit**接口。

```c++
void DVKCommandBuffer::Submit(VkSemaphore* signalSemaphore)
	{
		End();

		VkSubmitInfo submitInfo;
		ZeroVulkanStruct(submitInfo, VK_STRUCTURE_TYPE_SUBMIT_INFO);
		submitInfo.commandBufferCount   = 1;
		submitInfo.pCommandBuffers      = &cmdBuffer;
		submitInfo.signalSemaphoreCount = signalSemaphore ? 1 : 0;
		submitInfo.pSignalSemaphores    = signalSemaphore;
		
		if (waitFlags.size() > 0) 
		{
			submitInfo.waitSemaphoreCount = waitSemaphores.size();
			submitInfo.pWaitSemaphores    = waitSemaphores.data();
			submitInfo.pWaitDstStageMask  = waitFlags.data();
		}

		vkResetFences(vulkanDevice->GetInstanceHandle(), 1, &fence);
		vkQueueSubmit(vulkanDevice->GetGraphicsQueue()->GetHandle(), 1, &submitInfo, fence);
		vkWaitForFences(vulkanDevice->GetInstanceHandle(), 1, &fence, true, MAX_uint64);
	}
```

Submit提交时，自动进行End操作，防止未**End**的情况进行了提交。另外就是提交的时候，可以设置信号量**VkSemaphore**以便通知GPU的其它操作。

### Create

完成了上述的接口设计之后，我们就可以编写创建的函数。创建函数跟之前Demo里面的创建过程一样，我们把之前的Demo里面的代码拷贝过来即可。

```c++
DVKCommandBuffer* DVKCommandBuffer::Create(std::shared_ptr<VulkanDevice> vulkanDevice, VkCommandPool commandPool, VkCommandBufferLevel level)
	{
		VkDevice device = vulkanDevice->GetInstanceHandle();

		DVKCommandBuffer* cmdBuffer = new DVKCommandBuffer();
		cmdBuffer->vulkanDevice = vulkanDevice;
		cmdBuffer->commandPool  = commandPool;
		cmdBuffer->isBegun      = false;

		VkCommandBufferAllocateInfo cmdBufferAllocateInfo;
		ZeroVulkanStruct(cmdBufferAllocateInfo, VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO);
		cmdBufferAllocateInfo.commandPool = commandPool;
		cmdBufferAllocateInfo.level       = level;
		cmdBufferAllocateInfo.commandBufferCount = 1;
		vkAllocateCommandBuffers(device, &cmdBufferAllocateInfo, &(cmdBuffer->cmdBuffer));

		VkFenceCreateInfo fenceCreateInfo;
		ZeroVulkanStruct(fenceCreateInfo, VK_STRUCTURE_TYPE_FENCE_CREATE_INFO);
		fenceCreateInfo.flags = 0;
		vkCreateFence(device, &fenceCreateInfo, nullptr, &(cmdBuffer->fence));

		return cmdBuffer;
	
```

## Usage

有了封装之后的**DVKCommandBuffer**之后，我们就可以修改上一个Demo的代码，对其中**CreateMeshBuffers()**函数进行优化。

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

		vk_demo::DVKCommandBuffer* cmdBuffer = vk_demo::DVKCommandBuffer::Create(m_VulkanDevice, m_CommandPool);
		cmdBuffer->Begin();

		VkBufferCopy copyRegion = {};
		copyRegion.size = vertices.size() * sizeof(Vertex);
		vkCmdCopyBuffer(cmdBuffer->cmdBuffer, vertStaging->buffer, m_VertexBuffer->buffer, 1, &copyRegion);

		copyRegion.size = indices.size() * sizeof(uint16);
		vkCmdCopyBuffer(cmdBuffer->cmdBuffer, idexStaging->buffer, m_IndexBuffer->buffer, 1, &copyRegion);
        
		cmdBuffer->End();
		cmdBuffer->Submit();
        
		delete cmdBuffer;
		delete vertStaging;
		delete idexStaging;
	}
```

优化之后我们就会发现相对之前代码量又减少了很多。**CommandBuffer**只需要**Begin**、**录制**、**Sumit**即可。

