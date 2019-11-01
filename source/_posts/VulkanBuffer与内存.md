---
title: Vulkan Buffer与内存
date: 2019-06-07 23:43:40
tags:
- Vulkan
- 内存
- 3D
categories:
- Vulkan
---

# Vulkan Buffer与内存

### Buffer

在Vulkan里面，所有需要存储的资源都视为Buffer。其实这个不难理解，因为无论是Texture、VertexBuffer、IndexBuffer或者UniformBuffer等等，其实都最终都是一段内存，因此Vulkan将这些资源都视为Buffer。创建Buffer时就需要指定Buffer的大小、用途、共享模式等等。如下所示：

```c++
VkBufferCreateInfo vertexBufferInfo;
ZeroVulkanStruct(vertexBufferInfo, VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO);
vertexBufferInfo.size  = vertices.size() * sizeof(Vertex);
vertexBufferInfo.usage = VK_BUFFER_USAGE_TRANSFER_SRC_BIT;
VERIFYVULKANRESULT(vkCreateBuffer(m_Device, &vertexBufferInfo, VULKAN_CPU_ALLOCATOR, &tempVertexBuffer.buffer));
```

<!-- more -->

### 内存

但是Buffer并不会自己分配内存，如果Buffer自己分配了内存，那么我们无法做到自己管理内存。由于在GPU中需要内存对齐，因此我们的Buffer大小与实际内存可能并不一致。为了获取不同资源对应的内存对齐大小以及需要实际分配的内存大小，Vulkan提供了vkGetBufferMemoryRequirements函数供我们使用。如下所示：

```c++
vkGetBufferMemoryRequirements(m_Device, tempVertexBuffer.buffer, &memReqInfo);
uint32 memoryTypeIndex = 0;
GetVulkanRHI()->GetDevice()->GetMemoryManager().GetMemoryTypeFromProperties(memReqInfo.memoryTypeBits, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, &memoryTypeIndex);
memAllocInfo.allocationSize  = memReqInfo.size;
memAllocInfo.memoryTypeIndex = memoryTypeIndex;
VERIFYVULKANRESULT(vkAllocateMemory(m_Device, &memAllocInfo, VULKAN_CPU_ALLOCATOR, &tempVertexBuffer.memory));
```

### Buffer与内存绑定

分配好内存之后，就可以将Buffer与内存绑定到一起。

```c++
VERIFYVULKANRESULT(vkBindBufferMemory(m_Device, tempVertexBuffer.buffer, tempVertexBuffer.memory, 0));
```

我们也可以将多个Buffer绑定到一段分配好的内存。

### 数据传输

在设备上，由于GPU、CPU是两个独立的单元。它们之间使用的是被称为SMP架构模型进行互联并相互协作的，SMP即共享存储型多处理机(Shared Memory mulptiProcessors),）也称为对称型多处理机（Symmetry MultiProcessors)。

或者直白的说CPU和GPU间的交互就是通过共享内存这种方式来进行的，但他们各自又有各自的内存控制器和管理器，甚至各自还有自己的片上高速缓存，因此最终要共享内存就需要一些额外的通信控制方式来进行。

更进一步的SMP架构又被细分为：均匀存储器存取（Uniform-Memory-Access，简称UMA）模型、非均匀存储器存取（Nonuniform-Memory-Access，简称NUMA）模型和高速缓存相关的存储器结构（cache-coherent Memory Architecture，简称CC-UMA）模型，这些模型的区别在于存储器和外围资源如何共享或分布。

**UMA架构模型示意图如下：**

![smp_0](smp_0.png)

从图中可以看出UMA架构中物理存储器被所有处理机均匀共享。所有处理机对所有存储器具有相同的存取时间，这就是为什么称它为均匀存储器存取的原因。每个处理器（CPU或GPU）可以有私有高速缓存,外围设备也以一定形式共享（GPU因为没有访问外围其他设备的能力，实质就不共享外围设备了，这里主要指多个CPU的系统共享外围设备）。实质上UMA方式是目前已经很少见的主板集成显卡的方式之一。需要注意的是这里只是一个简化的示意图，里面只示意了一个CPU和GPU的情况，实质上它是可以扩展到任意多个CPU或GPU互通互联的情况的。

**NUMA架构的示意图如下：**

![smp_1](smp_1.png)

从NUMA的示意图中可以看出，其存储器物理上是分布在所有处理器的本地存储器上。本地存储器的一般具有各自独立的地址空间，因此一般不能直接互访问各自的本地存储。而处理器（CPU或GPU）访问本地存储器是比较快的，但要访问属于另一个处理器的远程存储器则比较慢，并且需要额外的方式和手段，因此其性能也是有额外的牺牲的。其实这也就是我们现在常见的“独显”的架构。当然一般来说现代GPU访问显存的速度是非常高的，甚至远高于CPU访问内存的速度。所以在编程中经常要考虑为了性能，而将尽可能多的纯GPU计算需要的数据放在显存中，从而提高GPU运算的效率和速度。

**CC-UMA架构的示意图如下：**

![smp_2](smp_2.png)

如图所示，CC-UMA是一种只用高速缓存互联互通的多处理器系统。CC-UMA模型是NUMA机的一种特例，只是将后者中分布主存储器换成了高速缓存, 在每个处理器上没有存储器层次结构,全部高速缓冲存储器组成了全局地址空间。通常这是现代CPU中集显最容易采取的架构方式。当然高速缓存共享或直连的方式拥有最高的互访性能。但其缺点就是高速缓存因为高昂的价格，所以往往空间很小，目前的集显上还只有几兆，最多到几十兆高速缓冲的样子，所以对于现代的渲染来说这点存储量实在是少的可怜了。另外因为高速缓存是在不同的处理器（CPU或GPU）之间直接共享或互联的，因此还有一个额外的问题就是存储一致性的问题，就是说高速缓冲的内容跟实质内存中的内容是否一致，比如CPU实质是将数据先加载进内存中然后再加载进高速缓冲的，而GPU在CPU还没有完成从内存到高速缓冲的加载时，就直接访问高速缓冲中的数据就会引起错误了，反之亦然。因此就需要额外的机制来保证存储一致性，当然这就导致一些额外的性能开销。具体的关于存储一致性的内容，我就不多讲了，我们主要还是要靠独显来干活。进一步的知识大家有兴趣的可以百度一下相关资料。

具体来说，比如我的笔记本电脑上就有一个“集显”也有一个“独显”，集显跟CPU形成了CC-UMA架构，并且它独占了128M的内存当做显存，而独显则与CPU形成了NUMA架构，独显上有2G的独立显存，两个GPU都与CPU共享了8149M的内存，作为统一的共享内存。其实这也可以看出实际的系统中往往是上述架构混用的形式。

**大家可以通行Dxdiag程序看到这些信息。**

综上，实质上这些架构之间的主要区别是在各处理器访问存储的速度上，简言之就是说使用高速缓存具有最高的访问速度。其次就是访问各自独占的存储，而最慢的就是访问共享内存了，当然对于CPU来说访问共享内存与自己独占的内存在性能是基本没有差异的。这里的性能差异主要是从GPU的角度来说的。因此我们肯定愿意将一些CPU或GPU专有的数据首先考虑放在各自的独占存储中，其次需要多方来访问的数据就放在共享内存中。

说了这么废话，就是为了给数据传输做铺垫。综上我们其实可以知道，对于UniformBuffer，我们可能更希望将它放置于共享内存中，对于Texture、Vertex、Index等等我们更希望将它们放置于GPU的独立内存中。因此，对于UniformBuffer，我们只需要在共享内存或者高速缓存上面分配内存，绑定到Buffer。

对于其它数据，我们则需要先在共享内存或者高速缓存上分配**临时内存**，绑定**临时Buffer**，然后将数据拷贝至于该块内存，最后则创建真正的Buffer以及在GPU上分配独立的内存，通过**Transfer Command**将数据从共享内存或者高速缓存拷贝至GPU内存。









