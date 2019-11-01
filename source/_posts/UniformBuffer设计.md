---
title: UniformBuffer设计
date: 2019-06-23 11:51:02
tags:
- Vulkan
- 物理设备
- 3D
categories:
- Vulkan
---

# Vulkan Uniform Buffer设计

最近我一直在思考关于UniformBuffer的结构如何设计。由于工作太忙，只能断断续续的设计它的结构以便于能够更好的匹配绘制流程。对D3D12或者Vulkan有过了解的同学应该也会有这个疑问，就是它们的UniformBuffer如何才能使用起来更加方便。

<!-- more -->

例如**D3D12**关于UniformBuffer的使用

[Github直达](https://github.com/microsoft/DirectX-Graphics-Samples/blob/master/Samples/Desktop/D3D12Bundles/src/FrameResource.cpp#L31)

```C++
// Create an upload heap for the constant buffers.
ThrowIfFailed(pDevice->CreateCommittedResource(
    &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD),
    D3D12_HEAP_FLAG_NONE,
    &CD3DX12_RESOURCE_DESC::Buffer(sizeof(SceneConstantBuffer) * m_cityRowCount * m_cityColumnCount),
    D3D12_RESOURCE_STATE_GENERIC_READ,
    nullptr,
    IID_PPV_ARGS(&m_cbvUploadHeap)));
```

或者**Vulkan**关于UniformBuffer的创建。

[Github直达](https://github.com/SaschaWillems/Vulkan/blob/master/examples/triangle/triangle.cpp#L1023)
```C++
// Create a new buffer
VK_CHECK_RESULT(vkCreateBuffer(device, &bufferInfo, nullptr, &uniformBufferVS.buffer));
vkGetBufferMemoryRequirements(device, uniformBufferVS.buffer, &memReqs);
allocInfo.allocationSize = memReqs.size;
allocInfo.memoryTypeIndex = getMemoryTypeIndex(memReqs.memoryTypeBits, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT);
VK_CHECK_RESULT(vkAllocateMemory(device, &allocInfo, nullptr, &(uniformBufferVS.memory)));
VK_CHECK_RESULT(vkBindBufferMemory(device, uniformBufferVS.buffer, uniformBufferVS.memory, 0));
```

从以上代码我们可以看出，如果渲染固定数量的对象，以上方式完全没有任何问题。但是我们现在设想如下两种情况：
- 多个Mesh共用一个材质
- 多个动态创建的Mesh

## 多个Mesh共用一个材质
这种情况在引擎里面非常非常常见，美术制作好一个材质，调节好参数，然后赋予给多个相同或者不同的Mesh。在渲染的时候，**常规**情况下除了Mesh的**World矩阵**会发生变化，其它材质参数在**当前批次**里面是不会发生变化的。

我之前设想的是，针对**World矩阵**创建出多个UniformBuffer或者创建一个Dynamic属性的UniformBuffer。但是这种方式不易于管理，因为我们要针对不同的材质、不同的**Backbuffer**分别创建出UniformBuffer或者DynamicUniformBuffer，并且这种方式会导致碎片化非常严重。

## 动态创建的Mesh
无论是否为动态创建的物体，最终都是被收集到RenderList里面，按照远近、材质进行归类。这样在真正录制渲染命令的时候，就可以根据这个列表进行UniformBuffer的动态创建以及分配。

如下：

[0, 0, 0] [1, 1, 1, 1] [2, 2]

一共有3 + 4 + 2个Mesh需要渲染，它们的材质ID分别为0，1，2。按照这样的方式，我们就可以在渲染ID为0的材质的时候，为它分配3个**Matrix4x4 UniformBuffer**用于存放**World矩阵**数据，为它分配**1**个**Material Param UniformBuffer**用于存放其材质参数。

这种方式其实需要我们有一个健壮强大动态的分配器以及回收器，同时还要能够规避碎片化问题。最终我放弃了这种方式，觉得难以管理以及实现。

## 利用Dynamic属性的UniformBuffer

**Dynamic UniformBuffer**其实也是**UniformBuffer**，它们并没有任何不同，也没有性能上的差异。但是可以利用在创建**DescriptorSetLayout**时指定为**VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC**，在录制绘制命令时，指定**vkCmdBindDescriptorSets**的Offset，以达到每个Drawcall读取不同的UniformBuffer数据，且都是在一个UniformBuffer上面操作的目的。

基于上述信息，其实可以将所有的**DescriptorSetLayout**中的**UniformBuffer**都设置为**Dynamic**。然后准备一个**虚拟**的**UniformBuffer**提供给**材质**以及**着色器**使用。

```c++
class VulkanEmulatedUniformBuffer
{
public:
	VulkanEmulatedUniformBuffer(const void* contents);

	std::Vector<uint8> constantData;

	void UpdateConstantData(const void* contents, int32 contentsSize);
};
```

既然我们将所有的UniformBuffer都设置为了Dynamic属性，那么我们可以将所有的UniformBuffer数据都存储到一个超大的UniformBuffer上面，通过Offset来完成数据的映射。

```c++
class VulkanRingBuffer
{
public:
	VulkanRingBuffer(uint64 totalSize, VkFlags usage, VkMemoryPropertyFlags memPropertyFlags);
    
	~VulkanRingBuffer();
	
	inline uint64 AllocateMemory(uint64 size, uint32 alignment)
	{
		alignment = max(alignment, minAlignment);
		uint64 allocationOffset = Align<uint64>(bufferOffset, alignment);
		if (allocationOffset + size <= bufferSize)
		{
			bufferOffset = allocationOffset + size;
			return allocationOffset;
		}

		return WrapAroundAllocateMemory(size, alignment);
	}

protected:
	uint64 bufferSize;
	uint64 bufferOffset;
	uint32 minAlignment;
	VkBuffer buffer;
    
	uint64 WrapAroundAllocateMemory(uint64 size, uint32 alignment)
	{
		bufferOffset = size;
	}
};
```

如上所示，准备一个RingBuffer，这个Buffer的容量预估在**32MB**或者**48MB**，这个RingBuffer直接与**VkBuffer**绑定，也就是Vulkan的UniformBuffer容量在**32MB**或者**48MB**。

有了这个Buffer之后，我们就可以通过**VulkanEmulatedUniformBuffer**为它从**VulkanRingBuffer**中分配一段真正的**VulkanBuffer**存储Uniform数据。这个RingBuffer其实也解决了**多缓冲区**的问题。一般来讲我们需要为每个缓冲区都分配UniformBuffer(其实就是每一次绘制命令从提交到执行完毕这段时间，UniformBuffer要保证不能被修改)，保证每个缓冲区使用到的UniformBuffer都是有效的。

之所以RingBuffer可以解决这个问题，是因为每次绘制都是为其分配的**新的段**用于写入数据。一般缓冲区也就2级或者3级，加上实时帧率一般都是在30帧以上，也就是3帧之前的数据其实已经没有保留的必要。因为这个RingBuffer足够长，当它**分配完**的时候其实已经过了3帧，这个时候可以放心的从头(**bufferOffset=0)**开始重新分配。

## DynamicOffsets

在绘制阶段，一般**Renderable**和**Material**绑定在一起。**Renderable**负责提供**VertexInputAttribute**，**Shader**负责提供**DescriptorSetLayout**以及**VertexBindingState**。

从**DescriptorSetLayout**中我们可以获取到整个结构，从**DescriptorSetLayout**中我们可以提前分配好**DynamicOffsets**数据以及**VkWriteDescriptorSet**。当在录制每个DrawCall的时候，我们将**VulkanEmulatedUniformBuffer**数据写入到**VulkanRingBuffer**同时记录**DynamicOffsets**数据，最终在**vkCmdBindDescriptorSets**阶段予以提供**DynamicOffsets**信息。

## SingleDraw 

SingleDraw意味着Uniform数据只在当次DrawCall中有效，例如**World**矩阵数据（骨骼数据等等），每一个渲染对象对应的**World**都是不同的，不能共享。因此我把这种数据类型称之为**SingleDraw**。

## MultiDraw

能够多个渲染对象共享的Uniform数据类型，我称之为**MultiDraw**。例如**Material**参数，虽然参数是作用到Shader上的，但是没有必要为每一个渲染对象都分配Uniform数据，可以为**Material**分配**Uniform**数据，使用到这个材质的渲染对象都共享**Uniform**数据。例如**Scene**的数据**ViewMatrix**、**ProjectionMatrix**、**CurrentTime**、**DeltaTime**、**Light**等数据，这些数据不仅可以在多个Material之间共享，还能贯穿当前帧。简单点对于**Material**，我们可以前10个物体是红色，后10个物体是绿色。它们的VkPipeline都是相同的，唯一不同的就是Color。如下代码：

[Github直达](<https://github.com/BobLChen/VulkanTutorials/blob/master/examples/6_DynamicUniformBuffer/DynamicUniformBuffer.cpp#L409>)

```c++
uint32 bufferAlignment = GetVulkanRHI()->GetDevice()->GetLimits().minUniformBufferOffsetAlignment;

// model matrix dynamic uniform buffer
uint32 modelAlign = Align(sizeof(ModelUBOData), bufferAlignment);
uint32 modelSize  = modelAlign * CUBE_COUNT;
m_ModelBuffers.resize(GetFrameCount());
for (int32 i = 0; i < GetFrameCount(); ++i) 
{
    m_ModelBuffers[i] = VulkanBuffer::CreateBuffer(VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT, modelSize);
}

uint32 vpAlign    = Align(sizeof(MaterialUBOData), bufferAlignment);
uint32 vpSize     = vpAlign * MATERIAL_COUT;
m_ViewProjectionBuffers.resize(GetFrameCount());
for (int32 i = 0; i < GetFrameCount(); ++i) 
{
    m_ViewProjectionBuffers[i] = VulkanBuffer::CreateBuffer(VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT, vpSize);
}
```

针对以上情况，我打算使用三个**VulkanRingBuffer**来存储Uniform数据，一个用来存储**SingleDraw**、一个用来存储**MultiDraw**、一个用来存储**Global**。其中**SingleDraw**和**MultiDraw**可以适当容量大一点儿，**Global**可以适当小一点儿。

这样无论是动态亦或者静态3D对象，我们都可以让其UniformBuffer的分配自动化起来，避免我们手动对其进行分配管理。

