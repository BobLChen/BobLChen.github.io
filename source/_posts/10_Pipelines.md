---
title: 10_Pipelines
date: 2019-08-02 20:01:52
tags:
- Vulkan
- 物理设备
- 3D
- Demo
categories:
- Vulkan
---

[Pipelines](https://github.com/BobLChen/VulkanDemos/tree/master/examples/10_Pipelines)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

在前面的Demo里面，我们封装了很多东西，但是有一个东西一直没有处理，也是因为这个东西在前的这些Demo里面都没有需求。下面我们将着手封装**Pipeline**。

<!-- more -->

![Pipelines](https://raw.githubusercontent.com/BobLChen/VulkanTutorials/master/preview/10_Pipelines.jpg)

## DVKPipelineInfo

通过观察之前Demo里面**Pipeline**创建流程，可以很容易的总结出，Pipeline的创建实质是状态的设置。例如混合、光栅化、深度模板测试等等。既然是状态的设置，那么我们提前准备好一个默认的状态，根据需要修改其中的状态，然后创建出**Pipeline**，不就简化了创建流程嘛？

根据这个思路，我们设计出**PipelineInfo**，里面保存了创建**Pipeline**的必要元素。

```c++
struct DVKPipelineInfo
{
	VkPipelineInputAssemblyStateCreateInfo		inputAssemblyState;
	VkPipelineRasterizationStateCreateInfo		rasterizationState;
	VkPipelineColorBlendAttachmentState			blendAttachmentStates[8];
	VkPipelineDepthStencilStateCreateInfo		depthStencilState;
	VkPipelineMultisampleStateCreateInfo		multisampleState;
    
	VkShaderModule	vertShaderModule = VK_NULL_HANDLE;
	VkShaderModule	fragShaderModule = VK_NULL_HANDLE;
	VkShaderModule	compShaderModule = VK_NULL_HANDLE;
	VkShaderModule	tescShaderModule = VK_NULL_HANDLE;
	VkShaderModule	teseShaderModule = VK_NULL_HANDLE;
	VkShaderModule	geomShaderModule = VK_NULL_HANDLE;

	DVKShader*		shader  = nullptr;
	int32			subpass = 0;
    int32           colorAttachmentCount = 1;

	DVKPipelineInfo()
	{
		ZeroVulkanStruct(inputAssemblyState, VK_STRUCTURE_TYPE_PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO);
		inputAssemblyState.topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST;
		
		ZeroVulkanStruct(rasterizationState, VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO);
		rasterizationState.polygonMode 			   = VK_POLYGON_MODE_FILL;
		rasterizationState.cullMode                = VK_CULL_MODE_BACK_BIT;
		rasterizationState.frontFace               = VK_FRONT_FACE_CLOCKWISE;
		rasterizationState.depthClampEnable        = VK_FALSE;
		rasterizationState.rasterizerDiscardEnable = VK_FALSE;
		rasterizationState.depthBiasEnable         = VK_FALSE;
		rasterizationState.lineWidth 			   = 1.0f;
		
        for (int32 i = 0; i < 8; ++i)
        {
            blendAttachmentStates[i] = {};
            blendAttachmentStates[i].colorWriteMask = (
                VK_COLOR_COMPONENT_R_BIT |
                VK_COLOR_COMPONENT_G_BIT |
                VK_COLOR_COMPONENT_B_BIT |
                VK_COLOR_COMPONENT_A_BIT
            );
            blendAttachmentStates[i].blendEnable = VK_FALSE;
            blendAttachmentStates[i].srcColorBlendFactor = VK_BLEND_FACTOR_ONE;
            blendAttachmentStates[i].dstColorBlendFactor = VK_BLEND_FACTOR_ZERO;
            blendAttachmentStates[i].colorBlendOp        = VK_BLEND_OP_ADD;
            blendAttachmentStates[i].srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE;
            blendAttachmentStates[i].dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO;
            blendAttachmentStates[i].alphaBlendOp        = VK_BLEND_OP_ADD;
        }
        
		ZeroVulkanStruct(depthStencilState, VK_STRUCTURE_TYPE_PIPELINE_DEPTH_STENCIL_STATE_CREATE_INFO);
		depthStencilState.depthTestEnable 		= VK_TRUE;
		depthStencilState.depthWriteEnable 		= VK_TRUE;
		depthStencilState.depthCompareOp		= VK_COMPARE_OP_LESS_OR_EQUAL;
		depthStencilState.depthBoundsTestEnable = VK_FALSE;
		depthStencilState.back.failOp 			= VK_STENCIL_OP_KEEP;
		depthStencilState.back.passOp 			= VK_STENCIL_OP_KEEP;
		depthStencilState.back.compareOp 		= VK_COMPARE_OP_ALWAYS;
		depthStencilState.stencilTestEnable 	= VK_TRUE;
		depthStencilState.front 				= depthStencilState.back;

		ZeroVulkanStruct(multisampleState, VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO);
		multisampleState.rasterizationSamples = VK_SAMPLE_COUNT_1_BIT;
		multisampleState.pSampleMask 		  = nullptr;
	}

	void FillShaderStages(std::vector<VkPipelineShaderStageCreateInfo>& shaderStages)
	{
		if (vertShaderModule != VK_NULL_HANDLE) {
			VkPipelineShaderStageCreateInfo shaderStageCreateInfo;
			ZeroVulkanStruct(shaderStageCreateInfo, VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO);
			shaderStageCreateInfo.stage  = VK_SHADER_STAGE_VERTEX_BIT;
			shaderStageCreateInfo.module = vertShaderModule;
			shaderStageCreateInfo.pName  = "main";
			shaderStages.push_back(shaderStageCreateInfo);
		}

		if (fragShaderModule != VK_NULL_HANDLE) {
			VkPipelineShaderStageCreateInfo shaderStageCreateInfo;
			ZeroVulkanStruct(shaderStageCreateInfo, VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO);
			shaderStageCreateInfo.stage  = VK_SHADER_STAGE_FRAGMENT_BIT;
			shaderStageCreateInfo.module = fragShaderModule;
			shaderStageCreateInfo.pName  = "main";
			shaderStages.push_back(shaderStageCreateInfo);
		}

		if (compShaderModule != VK_NULL_HANDLE) {
			VkPipelineShaderStageCreateInfo shaderStageCreateInfo;
			ZeroVulkanStruct(shaderStageCreateInfo, VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO);
			shaderStageCreateInfo.stage  = VK_SHADER_STAGE_COMPUTE_BIT;
			shaderStageCreateInfo.module = compShaderModule;
			shaderStageCreateInfo.pName  = "main";
			shaderStages.push_back(shaderStageCreateInfo);
		}

		if (geomShaderModule != VK_NULL_HANDLE) {
			VkPipelineShaderStageCreateInfo shaderStageCreateInfo;
			ZeroVulkanStruct(shaderStageCreateInfo, VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO);
			shaderStageCreateInfo.stage  = VK_SHADER_STAGE_GEOMETRY_BIT;
			shaderStageCreateInfo.module = geomShaderModule;
			shaderStageCreateInfo.pName  = "main";
			shaderStages.push_back(shaderStageCreateInfo);
		}

		if (tescShaderModule != VK_NULL_HANDLE) {
			VkPipelineShaderStageCreateInfo shaderStageCreateInfo;
			ZeroVulkanStruct(shaderStageCreateInfo, VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO);
			shaderStageCreateInfo.stage  = VK_SHADER_STAGE_TESSELLATION_CONTROL_BIT;
			shaderStageCreateInfo.module = tescShaderModule;
			shaderStageCreateInfo.pName  = "main";
			shaderStages.push_back(shaderStageCreateInfo);
		}

		if (teseShaderModule != VK_NULL_HANDLE) {
			VkPipelineShaderStageCreateInfo shaderStageCreateInfo;
			ZeroVulkanStruct(shaderStageCreateInfo, VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO);
			shaderStageCreateInfo.stage  = VK_SHADER_STAGE_TESSELLATION_EVALUATION_BIT;
			shaderStageCreateInfo.module = teseShaderModule;
			shaderStageCreateInfo.pName  = "main";
			shaderStages.push_back(shaderStageCreateInfo);
		}
	}

};
```

## DVKPipeline

有了**PipelineInfo**，那我们就可以根据Info信息来创建出对应的**Pipeline**即可。那Pipeline的设计和创建也变得极为简单：

```c++
class DVKPipeline
{
public:
	
	DVKPipeline()
		: vulkanDevice(nullptr)
		, pipeline(VK_NULL_HANDLE)
	{

	}

	~DVKPipeline()
	{
		VkDevice device = vulkanDevice->GetInstanceHandle();
		if (pipeline != VK_NULL_HANDLE) {
			vkDestroyPipeline(device, pipeline, VULKAN_CPU_ALLOCATOR);
		}
	}

	static DVKPipeline* Create(
		std::shared_ptr<VulkanDevice> vulkanDevice,
		VkPipelineCache pipelineCache,
		DVKPipelineInfo& pipelineInfo, 
		const std::vector<VkVertexInputBindingDescription>& inputBindings, 
		const std::vector<VkVertexInputAttributeDescription>& vertexInputAttributs,
		VkPipelineLayout pipelineLayout,
		VkRenderPass renderPass
	);

public:
	
	typedef std::shared_ptr<VulkanDevice> VulkanDeviceRef;

	VulkanDeviceRef		vulkanDevice;
	VkPipeline			pipeline;
	VkPipelineLayout	pipelineLayout;
};
```

在Create函数里面，我们只是简单的把**PipelineInfo**里面的数据填充到**CreateInfo**里面，然后创建出来即可。

```c++
DVKPipeline* DVKPipeline::Create(
		std::shared_ptr<VulkanDevice> vulkanDevice,
		VkPipelineCache pipelineCache,
		DVKPipelineInfo& pipelineInfo, 
		const std::vector<VkVertexInputBindingDescription>& inputBindings, 
		const std::vector<VkVertexInputAttributeDescription>& vertexInputAttributs,
		VkPipelineLayout pipelineLayout,
		VkRenderPass renderPass
	)
	{
		DVKPipeline* pipeline    = new DVKPipeline();
		pipeline->vulkanDevice   = vulkanDevice;
		pipeline->pipelineLayout = pipelineLayout;

		VkDevice device = vulkanDevice->GetInstanceHandle();

		VkPipelineVertexInputStateCreateInfo vertexInputState;
		ZeroVulkanStruct(vertexInputState, VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO);
		vertexInputState.vertexBindingDescriptionCount   = inputBindings.size();
		vertexInputState.pVertexBindingDescriptions      = inputBindings.data();
		vertexInputState.vertexAttributeDescriptionCount = vertexInputAttributs.size();
		vertexInputState.pVertexAttributeDescriptions    = vertexInputAttributs.data();

		VkPipelineColorBlendStateCreateInfo colorBlendState;
		ZeroVulkanStruct(colorBlendState, VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO);
		colorBlendState.attachmentCount = pipelineInfo.colorAttachmentCount;
		colorBlendState.pAttachments    = pipelineInfo.blendAttachmentStates;
		
		VkPipelineViewportStateCreateInfo viewportState;
		ZeroVulkanStruct(viewportState, VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO);
		viewportState.viewportCount = 1;
		viewportState.scissorCount  = 1;
			
		std::vector<VkDynamicState> dynamicStateEnables;
		dynamicStateEnables.push_back(VK_DYNAMIC_STATE_VIEWPORT);
		dynamicStateEnables.push_back(VK_DYNAMIC_STATE_SCISSOR);

		VkPipelineDynamicStateCreateInfo dynamicState;
		ZeroVulkanStruct(dynamicState, VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO);
		dynamicState.dynamicStateCount = 2;
		dynamicState.pDynamicStates    = dynamicStateEnables.data();

		std::vector<VkPipelineShaderStageCreateInfo> shaderStages;
		if (pipelineInfo.shader) {
			shaderStages = pipelineInfo.shader->shaderStageCreateInfos;
		}
		else {
			pipelineInfo.FillShaderStages(shaderStages);
		}
		
		VkGraphicsPipelineCreateInfo pipelineCreateInfo;
		ZeroVulkanStruct(pipelineCreateInfo, VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO);
		pipelineCreateInfo.layout 				= pipelineLayout;
		pipelineCreateInfo.renderPass 			= renderPass;
		pipelineCreateInfo.subpass              = pipelineInfo.subpass;
		pipelineCreateInfo.stageCount 			= shaderStages.size();
		pipelineCreateInfo.pStages 				= shaderStages.data();
		pipelineCreateInfo.pVertexInputState 	= &vertexInputState;
		pipelineCreateInfo.pInputAssemblyState 	= &(pipelineInfo.inputAssemblyState);
		pipelineCreateInfo.pRasterizationState 	= &(pipelineInfo.rasterizationState);
		pipelineCreateInfo.pColorBlendState 	= &colorBlendState;
		pipelineCreateInfo.pMultisampleState 	= &(pipelineInfo.multisampleState);
		pipelineCreateInfo.pViewportState 		= &viewportState;
		pipelineCreateInfo.pDepthStencilState 	= &(pipelineInfo.depthStencilState);
		pipelineCreateInfo.pDynamicState 		= &dynamicState;
		VERIFYVULKANRESULT(vkCreateGraphicsPipelines(device, pipelineCache, 1, &pipelineCreateInfo, VULKAN_CPU_ALLOCATOR, &(pipeline->pipeline)));
		
		return pipeline;
	}
```

这个Pipeline的封装，我们并没有做很多操作，只是把代码搬运到了一起而已。

## Usage

既然封装好了**Pipeline**，那我们肯定得要试用一下。在这个Demo里面，我们将创建三个Pipeline，以实现三种不同的效果。

- 白色带光照的模型
- 黄色带光照的模型
- 紫色可膨胀的模型，并且参数可调

前面两个的Shader很简单，不做说明，最后一个简单说明一下。模型膨胀其实就是按照顶点的法线方向进行膨胀。代码如下：

```glsl
#version 450

layout (location = 0) in vec3 inPosition;
layout (location = 1) in vec3 inNormal;

layout (binding = 0) uniform MVPBlock 
{
	mat4 modelMatrix;
	mat4 viewMatrix;
	mat4 projectionMatrix;
} uboMVP;

layout (binding = 1) uniform ParamBlock 
{
	float intensity;
} uboParam;

layout (location = 0) out vec3 outNormal;

out gl_PerVertex 
{
    vec4 gl_Position;   
};

void main() 
{
    vec3 normal   = normalize(inNormal);
    vec3 position = inPosition + inNormal * uboParam.intensity;
    outNormal     = inNormal;
	gl_Position   = uboMVP.projectionMatrix * uboMVP.viewMatrix * uboMVP.modelMatrix * vec4(position.xyz, 1.0);

```

### 创建Pipelines

我们在**CreatePipelines**函数里面分别创建三个Pipeline。如下所示：

```c++
void CreatePipelines()
{
	m_Pipelines.resize(3);

	VkVertexInputBindingDescription vertexInputBinding = m_Model->GetInputBinding();
	std::vector<VkVertexInputAttributeDescription> vertexInputAttributs = m_Model->GetInputAttributes();
	
	vk_demo::DVKPipelineInfo pipelineInfo0;
    pipelineInfo0.vertShaderModule = vk_demo::LoadSPIPVShader(m_Device, "assets/shaders/10_Pipelines/pipeline0.vert.spv");
	pipelineInfo0.fragShaderModule = vk_demo::LoadSPIPVShader(m_Device, "assets/shaders/10_Pipelines/pipeline0.frag.spv");
	m_Pipelines[0] = vk_demo::DVKPipeline::Create(m_VulkanDevice, m_PipelineCache, pipelineInfo0, { vertexInputBinding }, vertexInputAttributs, m_PipelineLayout, m_RenderPass);
	
	vk_demo::DVKPipelineInfo pipelineInfo1;
    pipelineInfo1.vertShaderModule = vk_demo::LoadSPIPVShader(m_Device, "assets/shaders/10_Pipelines/pipeline1.vert.spv");
	pipelineInfo1.fragShaderModule = vk_demo::LoadSPIPVShader(m_Device, "assets/shaders/10_Pipelines/pipeline1.frag.spv");
	m_Pipelines[1] = vk_demo::DVKPipeline::Create(m_VulkanDevice, m_PipelineCache, pipelineInfo1, { vertexInputBinding }, vertexInputAttributs, m_PipelineLayout, m_RenderPass);

	vk_demo::DVKPipelineInfo pipelineInfo2;
	pipelineInfo2.rasterizationState.polygonMode = VkPolygonMode::VK_POLYGON_MODE_LINE;
    pipelineInfo2.vertShaderModule = vk_demo::LoadSPIPVShader(m_Device, "assets/shaders/10_Pipelines/pipeline2.vert.spv");
	pipelineInfo2.fragShaderModule = vk_demo::LoadSPIPVShader(m_Device, "assets/shaders/10_Pipelines/pipeline2.frag.spv");
	m_Pipelines[2] = vk_demo::DVKPipeline::Create(m_VulkanDevice, m_PipelineCache, pipelineInfo2, { vertexInputBinding }, vertexInputAttributs, m_PipelineLayout, m_RenderPass);
	
	vkDestroyShaderModule(m_Device, pipelineInfo0.vertShaderModule, VULKAN_CPU_ALLOCATOR);
	vkDestroyShaderModule(m_Device, pipelineInfo0.fragShaderModule, VULKAN_CPU_ALLOCATOR);
	vkDestroyShaderModule(m_Device, pipelineInfo1.vertShaderModule, VULKAN_CPU_ALLOCATOR);
	vkDestroyShaderModule(m_Device, pipelineInfo1.fragShaderModule, VULKAN_CPU_ALLOCATOR);
	vkDestroyShaderModule(m_Device, pipelineInfo2.vertShaderModule, VULKAN_CPU_ALLOCATOR);
	vkDestroyShaderModule(m_Device, pipelineInfo2.fragShaderModule, VULKAN_CPU_ALLOCATOR);
}
```

### 调整Projection

在这个Demo我还将演示如何通过Viewport参数来控制渲染区域。三个效果，我们将分成三个区域进行显示。既然渲染区域有调整，那么我们的投影矩阵中的宽高比例也有调整才行。

```c++
void CreateUniformBuffers()
	{
		vk_demo::DVKBoundingBox bounds = m_Model->rootNode->GetBounds();
		Vector3 boundSize   = bounds.max - bounds.min;
        Vector3 boundCenter = bounds.min + boundSize * 0.5f;
		boundCenter.z -= boundSize.Size();

		m_MVPDatas.resize(m_Model->meshes.size());
		m_MVPBuffers.resize(m_Model->meshes.size());

		for (int32 i = 0; i < m_Model->meshes.size(); ++i)
		{
			m_MVPDatas[i].model.SetIdentity();
			m_MVPDatas[i].model.SetOrigin(Vector3(0, 0, 0));
        
			m_MVPDatas[i].view.SetIdentity();
			m_MVPDatas[i].view.SetOrigin(boundCenter);
			m_MVPDatas[i].view.SetInverse();

			m_MVPDatas[i].projection.SetIdentity();
			m_MVPDatas[i].projection.Perspective(MMath::DegreesToRadians(75.0f), (float)GetWidth() / 3.0f, (float)GetHeight(), 0.01f, 3000.0f);
		
			m_MVPBuffers[i] = vk_demo::DVKBuffer::CreateBuffer(
				m_VulkanDevice, 
				VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, 
				VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, 
				sizeof(MVPBlock),
				&(m_MVPDatas[i])
			);
			m_MVPBuffers[i]->Map();
		}

		m_ParamData.intensity = 5.0f;
		m_ParamBuffer = vk_demo::DVKBuffer::CreateBuffer(
			m_VulkanDevice, 
			VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, 
			VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, 
			sizeof(ParamBlock),
			&(m_ParamData)
		);
		m_ParamBuffer->Map();
	}
```

### 录制渲染命令

录制渲染命令时，我们需要注意，我们在绘制每一个模型时，我们都调整了视口的宽度，每个视口占屏幕宽度的1/3，并且还修改了坐标。这样每一个模型都是在单独的一个视口区域内进行绘制。

```c++
for (int32 i = 0; i < m_CommandBuffers.size(); ++i)
{
    renderPassBeginInfo.framebuffer = m_FrameBuffers[i];
    
	VERIFYVULKANRESULT(vkBeginCommandBuffer(m_CommandBuffers[i], &cmdBeginInfo));
	vkCmdBeginRenderPass(m_CommandBuffers[i], &renderPassBeginInfo, VK_SUBPASS_CONTENTS_INLINE);

	for (int32 j = 0; j < 3; ++j)
	{
		int32 ww = 1.0f / 3 * m_FrameWidth;
		int32 tx = j * ww;

		VkViewport viewport = {};
		viewport.x        = tx;
		viewport.y        = m_FrameHeight;
		viewport.width    = ww;
		viewport.height   = -(float)m_FrameHeight;    // flip y axis
		viewport.minDepth = 0.0f;
		viewport.maxDepth = 1.0f;
		
		VkRect2D scissor = {};
		scissor.extent.width  = ww;
		scissor.extent.height = m_FrameHeight;
		scissor.offset.x      = tx;
		scissor.offset.y      = 0;

		vkCmdSetViewport(m_CommandBuffers[i], 0, 1, &viewport);
		vkCmdSetScissor(m_CommandBuffers[i], 0, 1, &scissor);

		vkCmdBindPipeline(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_Pipelines[j]->pipeline);
        vkCmdBindDescriptorSets(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_Pipelines[j]->pipelineLayout, 0, 1, &m_DescriptorSets[i], 0, nullptr);
        
		for (int32 meshIndex = 0; meshIndex < m_Model->meshes.size(); ++meshIndex) {
			m_Model->meshes[meshIndex]->BindDrawCmd(m_CommandBuffers[i]);
		}
	}
	
	m_GUI->BindDrawCmd(m_CommandBuffers[i], m_RenderPass);

	vkCmdEndRenderPass(m_CommandBuffers[i]);
	VERIFYVULKANRESULT(vkEndCommandBuffer(m_CommandBuffers[i]));
}
```

最终的效果就是跟封面图一样。在这个Demo里面，我们封装了**Pipeline**，最终Pipeline的创建代码是急剧减少，尤其特别是需要大量创建Pipeline的时候，代码减少的更多。代码量少有一个好处：排查错误非常方便，避免了因为复制粘贴导致的一些错误。

