---
title: 13_DynamicUniformBuffer
date: 2019-08-15 22:49:17
tags:
- Vulkan
- 物理设备
- 3D
- Demo
categories:
- Vulkan
---

[DynamicUniformBuffer](https://github.com/BobLChen/VulkanDemos/blob/master/examples/13_DynamicUniformBuffer/DynamicUniformBuffer.cpp)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

上一个Demo里面我们通过PushConstant实现了使用同一个Shader绘制不同模型的需求。但是PushConstant有弊端，它使用的量非常少，在我的RTX2080TI上面只有256字节可以使用，我相信在其它常规设备上面也只会有这个量。所以我们要来了解一下DynamicUniformBuffer。

<!-- more -->

![DynamicUniformBuffer](https://raw.githubusercontent.com/BobLChen/VulkanTutorials/master/preview/13_DynamicUniformBuffer.jpg)

DynamicUniformBuffer的创建过程跟普通的UniformBuffer的创建过程一样，因为它们都是UniformBuffer，数据一样，用途大致一样，功能稍微有点儿区别而已。只是有个小地方需要注意一下，因为是Dynamic的形式，可以在上面自由存储数据，所以需要注意对齐，迎合GPU。我们可以在一个大的Buffer上面存储若干个我们需要的数据，然后通过Offset偏移的方式来使用对应的数据。

### 创建DynamicUniformBuffer

在本Demo里面，我们将创建三个UniformBuffer，其中两个用于Dynamic以更新模型的**颜色**以及**矩阵**信息。另外一个还是常规的UniformBuffer，因为有些数据只需要**每帧**更新**一次**，**当帧**里面所有的渲染物体**都适用**，例如**Light**、**Camera**等信息。

```c++
void CreateUniformBuffers()
	{
		vk_demo::DVKBoundingBox bounds = m_Model->rootNode->GetBounds();
		Vector3 boundSize   = bounds.max - bounds.min;
        Vector3 boundCenter = bounds.min + boundSize * 0.5f;
        boundCenter.z -= boundSize.Size() * 0.5f;
        
		uint32 alignment  = m_VulkanDevice->GetLimits().minUniformBufferOffsetAlignment;
        // world matrix dynamicbuffer
		uint32 modelAlign = Align(sizeof(ModelBlock), alignment);
        m_ModelDatas.resize(modelAlign * m_Model->meshes.size());
        for (int32 i = 0; i < m_Model->meshes.size(); ++i)
        {
            ModelBlock* modelBlock = (ModelBlock*)(m_ModelDatas.data() + modelAlign * i);
            modelBlock->model = m_Model->meshes[i]->linkNode->GetGlobalMatrix();
        }
        
		m_ModelBuffer = vk_demo::DVKBuffer::CreateBuffer(
			m_VulkanDevice, 
			VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, 
			VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, 
			m_ModelDatas.size(),
			m_ModelDatas.data()
		);
		m_ModelBuffer->Map();
        
		// view projection buffer
		m_ViewProjData.view.SetIdentity();
		m_ViewProjData.view.SetOrigin(boundCenter);
        m_ViewProjData.view.AppendRotation(30, Vector3::RightVector);
		m_ViewProjData.view.SetInverse();

		m_ViewProjData.projection.SetIdentity();
		m_ViewProjData.projection.Perspective(MMath::DegreesToRadians(75.0f), (float)GetWidth(), (float)GetHeight(), 100.0f, 1000.0f);
		
		m_ViewProjBuffer = vk_demo::DVKBuffer::CreateBuffer(
			m_VulkanDevice, 
			VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, 
			VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, 
			sizeof(ViewProjectionBlock),
			&(m_ViewProjData)
		);
		m_ViewProjBuffer->Map();
        
		// color per object
		uint32 colorAlign = Align(sizeof(ColorBlock), alignment);
		m_ColorDatas.resize(colorAlign * m_Model->meshes.size());
		for (int32 i = 0; i < m_Model->meshes.size(); ++i)
		{
            float r = MMath::RandRange(0.0f, 1.0f);
            float g = MMath::RandRange(0.0f, 1.0f);
            float b = MMath::RandRange(0.0f, 1.0f);
            ColorBlock* colorBlock = (ColorBlock*)(m_ColorDatas.data() + colorAlign * i);
			colorBlock->color.Set(r, g, b, 1.0f);
		}
		m_ColorBuffer = vk_demo::DVKBuffer::CreateBuffer(
			m_VulkanDevice, 
			VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, 
			VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, 
			m_ColorDatas.size(),
			m_ColorDatas.data()
		);
		m_ColorBuffer->Map();
        
		for (int32 i = 0; i < m_Model->meshes.size(); ++i) {
			m_MeshNames.push_back(m_Model->meshes[i]->linkNode->name.c_str());
		}
	}
```

### 设置DescriptorSetLayout

在前面的Demo里面，我们了解到**DescriptorSetLayout**是用来描述资源信息。既然我们有了一个新的类型的资源，那么也要信息其对应的**DescriptorSetLayout**。

```c++
void CreateDescriptorSetLayout()
	{
		VkDescriptorSetLayoutBinding layoutBindings[3] = { };
		layoutBindings[0].binding 			 = 0;
		layoutBindings[0].descriptorType     = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
		layoutBindings[0].descriptorCount    = 1;
		layoutBindings[0].stageFlags 		 = VK_SHADER_STAGE_VERTEX_BIT;
		layoutBindings[0].pImmutableSamplers = nullptr;

		layoutBindings[1].binding 			 = 1;
		layoutBindings[1].descriptorType     = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC;
		layoutBindings[1].descriptorCount    = 1;
		layoutBindings[1].stageFlags 		 = VK_SHADER_STAGE_VERTEX_BIT;
		layoutBindings[1].pImmutableSamplers = nullptr;

		layoutBindings[2].binding 			 = 2;
		layoutBindings[2].descriptorType     = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC;
		layoutBindings[2].descriptorCount    = 1;
		layoutBindings[2].stageFlags 		 = VK_SHADER_STAGE_FRAGMENT_BIT;
		layoutBindings[2].pImmutableSamplers = nullptr;

		VkDescriptorSetLayoutCreateInfo descSetLayoutInfo;
		ZeroVulkanStruct(descSetLayoutInfo, VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO);
		descSetLayoutInfo.bindingCount = 3;
		descSetLayoutInfo.pBindings    = layoutBindings;
		VERIFYVULKANRESULT(vkCreateDescriptorSetLayout(m_Device, &descSetLayoutInfo, VULKAN_CPU_ALLOCATOR, &m_DescriptorSetLayout));
        
		VkPipelineLayoutCreateInfo pipeLayoutInfo;
		ZeroVulkanStruct(pipeLayoutInfo, VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO);
		pipeLayoutInfo.setLayoutCount = 1;
		pipeLayoutInfo.pSetLayouts    = &m_DescriptorSetLayout;
		VERIFYVULKANRESULT(vkCreatePipelineLayout(m_Device, &pipeLayoutInfo, VULKAN_CPU_ALLOCATOR, &m_PipelineLayout));
	}
```

注意看**VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC**这个类型对应的描述信息。

### 设置DescriptorSet

有了DescriptorSetLayout之后，我们就可以根据DescriptorSetLayout绑定对应的资源。所以DescriptorSet也要随着DescriptorSetLayout的调整而进行调整。

```c++
void CreateDescriptorSet()
	{
		VkDescriptorPoolSize poolSizes[2];
		poolSizes[0].type            = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
		poolSizes[0].descriptorCount = 1;
		poolSizes[1].type            = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC;
		poolSizes[1].descriptorCount = 2;
        
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
        writeDescriptorSet.pBufferInfo     = &(m_ViewProjBuffer->descriptor);
        writeDescriptorSet.dstBinding      = 0;
        vkUpdateDescriptorSets(m_Device, 1, &writeDescriptorSet, 0, nullptr);

		VkDescriptorBufferInfo bufferInfo;
		bufferInfo.buffer = m_ModelBuffer->buffer;
		bufferInfo.offset = 0;
		bufferInfo.range  = sizeof(ModelBlock);

		ZeroVulkanStruct(writeDescriptorSet, VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET);
        writeDescriptorSet.dstSet          = m_DescriptorSet;
        writeDescriptorSet.descriptorCount = 1;
        writeDescriptorSet.descriptorType  = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC;
        writeDescriptorSet.pBufferInfo     = &bufferInfo;
        writeDescriptorSet.dstBinding      = 1;
        vkUpdateDescriptorSets(m_Device, 1, &writeDescriptorSet, 0, nullptr);

		bufferInfo.buffer = m_ColorBuffer->buffer;
		bufferInfo.offset = 0;
		bufferInfo.range  = sizeof(ColorBlock);
    
		ZeroVulkanStruct(writeDescriptorSet, VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET);
        writeDescriptorSet.dstSet          = m_DescriptorSet;
        writeDescriptorSet.descriptorCount = 1;
        writeDescriptorSet.descriptorType  = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC;
        writeDescriptorSet.pBufferInfo     = &bufferInfo;
        writeDescriptorSet.dstBinding      = 2;
        vkUpdateDescriptorSets(m_Device, 1, &writeDescriptorSet, 0, nullptr);
	}
```

这里有一个非常非常需要注意的地方，就是**VkDescriptorBufferInfo**的描述信息要正确。在一些**其它**的**Demo**里面，它们一般直接将整个Buffer的**size**当成了**range**传递给了**VkDescriptorBufferInfo**。这个行为其实是错误的，给了我们一种感觉，就是**VkDescriptorBufferInfo**的range描述的是Buffer的长度，但是其实不是这样。我们回想一下之前提到的，**DescriptorSet**用于关联资源，我们的**Shader**里面用到的**UniformBuffer**数据只有一个**mat4**或者**vec4**，如下所示：

```glsl
// Vertex
layout (binding = 1) uniform ModelBlock
{
	mat4 modelMatrix;
} uboModel;

// Fragment
layout (binding = 2) uniform ColorBlock
{
	vec4 color;
} uboColor;
```

所有这里我们要设置为我们使用到的真实UniformBuffer的长度。如果不注意这一点儿，可能后面就会遇到不能理解的错误。例如我们创建了一个32MB的Buffer，然后在这个Buffer上面存储数据，然后我们把这个32MB的长度设置到**VkDescriptorBufferInfo**，Vulkan的校验层就会抛出一个错误，UniformBuffer的最大长度只能为65535。

### 绑定使用

做好了准备工作之后，我们即可以在录制命令阶段将它启用起来。启用起来也非常简单，只需要设置一下偏移量即可。

```c++
void SetupCommandBuffers()
	{
		VkCommandBufferBeginInfo cmdBeginInfo;
		ZeroVulkanStruct(cmdBeginInfo, VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO);

		VkClearValue clearValues[2];
		clearValues[0].color        = { {0.2f, 0.2f, 0.2f, 1.0f} };
		clearValues[1].depthStencil = { 1.0f, 0 };

		VkRenderPassBeginInfo renderPassBeginInfo;
		ZeroVulkanStruct(renderPassBeginInfo, VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO);
        renderPassBeginInfo.renderPass      = m_RenderPass;
		renderPassBeginInfo.clearValueCount = 2;
		renderPassBeginInfo.pClearValues    = clearValues;
		renderPassBeginInfo.renderArea.offset.x = 0;
		renderPassBeginInfo.renderArea.offset.y = 0;
        renderPassBeginInfo.renderArea.extent.width  = m_FrameWidth;
        renderPassBeginInfo.renderArea.extent.height = m_FrameHeight;
        
		uint32 alignment  = m_VulkanDevice->GetLimits().minUniformBufferOffsetAlignment;
		uint32 modelAlign = Align(sizeof(ModelBlock), alignment);
		uint32 colorAlign = Align(sizeof(ColorBlock), alignment);

		for (int32 i = 0; i < m_CommandBuffers.size(); ++i)
		{
            renderPassBeginInfo.framebuffer = m_FrameBuffers[i];
            
			VERIFYVULKANRESULT(vkBeginCommandBuffer(m_CommandBuffers[i], &cmdBeginInfo));
			vkCmdBeginRenderPass(m_CommandBuffers[i], &renderPassBeginInfo, VK_SUBPASS_CONTENTS_INLINE);
            
            VkViewport viewport = {};
            viewport.x        = 0;
            viewport.y        = m_FrameHeight;
            viewport.width    = m_FrameWidth;
            viewport.height   = -(float)m_FrameHeight;    // flip y axis
            viewport.minDepth = 0.0f;
            viewport.maxDepth = 1.0f;
            
            VkRect2D scissor = {};
            scissor.extent.width  = m_FrameWidth;
            scissor.extent.height = m_FrameHeight;
            scissor.offset.x      = 0;
            scissor.offset.y      = 0;
            
            vkCmdSetViewport(m_CommandBuffers[i], 0, 1, &viewport);
            vkCmdSetScissor(m_CommandBuffers[i], 0, 1, &scissor);
            
            vkCmdBindPipeline(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_Pipeline->pipeline);
            
            for (int32 meshIndex = 0; meshIndex < m_Model->meshes.size(); ++meshIndex)
            {
				uint32 dynamicOffsets[2] = {
					meshIndex * modelAlign,
					meshIndex * colorAlign
				};
				vkCmdBindDescriptorSets(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_Pipeline->pipelineLayout, 0, 1, &m_DescriptorSet, 2, dynamicOffsets);
                m_Model->meshes[meshIndex]->BindDrawCmd(m_CommandBuffers[i]);
            }
			
			m_GUI->BindDrawCmd(m_CommandBuffers[i], m_RenderPass);

			vkCmdEndRenderPass(m_CommandBuffers[i]);
			VERIFYVULKANRESULT(vkEndCommandBuffer(m_CommandBuffers[i]));
		}
	}
```

### 修改数据

为了验证我们的UniformBuffer确实是动态可修改的。我们在UI那里增加两个小功能：

- 修改Alpha值，以改变透明度。
- 修改选择的Mesh对应的颜色值。

```c++
void UpdateUI(float time, float delta)
	{
		m_GUI->StartFrame();
        
		{
			ImGui::SetNextWindowPos(ImVec2(0, 0));
			ImGui::SetNextWindowSize(ImVec2(0, 0), ImGuiSetCond_FirstUseEver);
            ImGui::Begin("DynamicUniformBuffer", nullptr, ImGuiWindowFlags_AlwaysAutoResize | ImGuiWindowFlags_NoResize | ImGuiWindowFlags_NoMove);
			ImGui::Text("Renderabls");
            
			ImGui::Checkbox("AutoRotate", &m_AutoRotate);
            
            uint32 alignment  = m_VulkanDevice->GetLimits().minUniformBufferOffsetAlignment;
            uint32 colorAlign = Align(sizeof(ColorBlock), alignment);
            
			if (ImGui::SliderFloat("Alpha", &m_GlobalAlpha, 0.0f, 1.0f)) {
				for (int32 i = 0; i < m_Model->meshes.size(); ++i) {
                    ColorBlock* colorBlock = (ColorBlock*)(m_ColorDatas.data() + colorAlign * i);
					colorBlock->color.w = m_GlobalAlpha;
				}
			}
            
			ImGui::Combo("Select Mesh", &m_Selected, m_MeshNames.data(), m_MeshNames.size());
            
            ColorBlock* selectedColorBlock = (ColorBlock*)(m_ColorDatas.data() + m_Selected * colorAlign);
			ImGui::ColorEdit4("Mesh Color", (float*)&(selectedColorBlock->color), ImGuiColorEditFlags_AlphaBar);

			ImGui::Text("%.3f ms/frame (%.1f FPS)", 1000.0f / ImGui::GetIO().Framerate, ImGui::GetIO().Framerate);
            ImGui::End();
		}
        
		m_GUI->EndFrame();
        
		if (m_GUI->Update()) {
			SetupCommandBuffers();
		}
	}
```

整个流程其实还是非常简单的，只是需要稍微注意几个地方而已，不要被其它的Demo给带偏了。