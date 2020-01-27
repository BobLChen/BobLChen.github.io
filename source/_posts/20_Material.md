---
title: 20_Material
date: 2019-09-20 22:45:21
tags:
- Vulkan
- 3D
- Material
- Tutorial
categories:
- Vulkan
---

[20_Material](https://github.com/BobLChen/VulkanDemos/tree/master/examples/20_Material)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

之前的Demo里面做了很多封装，例如对Buffer、Pipeline、Texture等进行简单封装以方便后续的使用。通过上个Demo发现虽然经过了封装，但是还是存在一些问题，主要关于**Shader**和**Material**。我想在这个Demo里面设计出一个非常简单的**Shader**以及**Material**的结构出来以方便后续的Demo。

<!-- more -->

![20_Material](https://raw.githubusercontent.com/BobLChen/VulkanDemos/master/preview/19_OptimizeDeferredShading_1.jpg)

## 20_Material

之前已经对Shader以及Layout进行了封装，现在需要的是在它们的基础上封装出一个简单的Material出来。封装Material主要的难点就是参数如何传递。

### UniformBuffer

UniformBuffer需要额外设计一下，针对[UniformBuffer](http://xiaopengyou.fun/public/2019/06/23/UniformBuffer%E8%AE%BE%E8%AE%A1/#more)的设计单独开了一篇来简单讲解，大家可以通过链接查看。

### Params

确定了UniformBuffer的结构之后，就可以按需设计出相关的结构。

```c++
struct DVKSimulateBuffer
{
	std::vector<uint8>		dataContent;
	bool                    global = false;
	uint32					dataSize = 0;
	uint32					set = 0;
	uint32					binding = 0;
	uint32					dynamicIndex = 0;
	VkDescriptorType		descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
	VkShaderStageFlags		stageFlags = 0;
	VkDescriptorBufferInfo	bufferInfo;
};

struct DVKSimulateTexture
{
	uint32              set = 0;
	uint32              binding = 0;
	VkDescriptorType    descriptorType = VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE;
	VkShaderStageFlags  stageFlags = 0;
	DVKTexture*         texture = nullptr;
};
```

主要的就是RingBuffer的结构:

```c++
class DVKRingBuffer
{
public:
	DVKRingBuffer()
	{

	}

	virtual ~DVKRingBuffer()
	{
		realBuffer->UnMap();
		delete realBuffer;
		realBuffer = nullptr;
	}

	void* GetMappedPointer()
	{
		return realBuffer->mapped;
	}

	uint64 AllocateMemory(uint64 size)
	{
		uint64 allocationOffset = Align<uint64>(bufferOffset, minAlignment);
		
		if (allocationOffset + size <= bufferSize) 
		{
			bufferOffset = allocationOffset + size;
			return allocationOffset;
		}

		bufferOffset = 0;
		return bufferOffset;
	}
	
public:
	VkDevice		device = VK_NULL_HANDLE;
	uint64			bufferSize = 0;
	uint64			bufferOffset = 0;
	uint32			minAlignment = 0;
	DVKBuffer*		realBuffer = nullptr;
};
```

RingBuffer的职责就是在一段超长的Buffer上面分配存储数据提供Offset，分配完了之后又从头开始分配。

### Material

最后我们的Material结构就可以简单的确定出来。

```c++
class DVKMaterial
{
private:

	typedef std::unordered_map<std::string, DVKSimulateBuffer>		BuffersMap;
	typedef std::unordered_map<std::string, DVKSimulateTexture>		TexturesMap;
	typedef std::shared_ptr<VulkanDevice>							VulkanDeviceRef;

	DVKMaterial()
	{
		
	}
	
public:
	virtual ~DVKMaterial();

	static DVKMaterial* Create(std::shared_ptr<VulkanDevice> vulkanDevice, VkRenderPass renderPass, VkPipelineCache pipelineCache, DVKShader* shader);
	
	static DVKMaterial* Create(std::shared_ptr<VulkanDevice> vulkanDevice, DVKRenderTarget* renderTarget, VkPipelineCache pipelineCache, DVKShader* shader);

	void PreparePipeline();

	void BeginObject();

	void EndObject();

	void BeginFrame();

	void EndFrame();

	void BindDescriptorSets(VkCommandBuffer commandBuffer, VkPipelineBindPoint bindPoint, int32 objIndex);

	void SetLocalUniform(const std::string& name, void* dataPtr, uint32 size);
	
	void SetTexture(const std::string& name, DVKTexture* texture);

	void SetGlobalUniform(const std::string& name, void* dataPtr, uint32 size);

	void SetStorageBuffer(const std::string& name, DVKBuffer* buffer);

	void SetInputAttachment(const std::string& name, DVKTexture* texture);

	inline VkPipeline GetPipeline() const
	{
		return pipeline->pipeline;
	}

	inline VkPipelineLayout GetPipelineLayout() const
	{
		return pipeline->pipelineLayout;
	}

	inline std::vector<VkDescriptorSet>& GetDescriptorSets() const
	{
		return descriptorSet->descriptorSets;
	}
	
private:
	static void InitRingBuffer(std::shared_ptr<VulkanDevice> vulkanDevice);

	static void DestroyRingBuffer();

	void Prepare();

private:

	static DVKRingBuffer*	ringBuffer;
	static int32			ringBufferRefCount;

public:

	VulkanDeviceRef			vulkanDevice = nullptr;
	DVKShader*				shader = nullptr;
	
	VkRenderPass            renderPass = VK_NULL_HANDLE;
	VkPipelineCache         pipelineCache = VK_NULL_HANDLE;
	
	DVKGfxPipelineInfo      pipelineInfo;
	DVKGfxPipeline*         pipeline = nullptr;
	DVKDescriptorSet*		descriptorSet = nullptr;

	uint32					dynamicOffsetCount;
	std::vector<uint32>		globalOffsets;
	std::vector<uint32>     dynamicOffsets;
	std::vector<uint32>		perObjectIndexes;
	
	BuffersMap				uniformBuffers;
	BuffersMap				storageBuffers;
	TexturesMap				textures;

	bool                    actived = false;
};
```

这个其实只是一个简单的结构，某些API是一种妥协。在一个成熟的结构里面，Material其实是不需要跟renderPass、pipelineCache等耦合的。这些数据应该有一个专门的管理器来管理，根据运行状态以及参数从管理器里面获取，结合Shader、Material复原出我们在第一个Demo里面所做的事情。

最后将新设计的Material替换进去，在这个Demo里面又精简了一部分代码，在后续的Demo里面就会一直使用它们。