---
title: 6_ImGUI
date: 2019-08-01 20:09:52
tags:
- Vulkan
- 3D
- IMGUI
- Tutorial
categories:
- Vulkan
---

[ImageGUI](https://github.com/BobLChen/VulkanDemos/tree/master/examples/6_ImageGUI)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

在之前的Demo里面，我们对一些需要大量编码的工作进行了简单封装。为了方便后续**Demo**的进行，现在急需一个**UI**来进行参数或者功能相关的调节。幸运的是，**Github**上有现成的**UI**库可以使用，该项目叫做[**imgui**](https://github.com/ocornut/imgui)。这个UI库有**Vulkan**的实现**[点我查看](https://github.com/ocornut/imgui/blob/master/examples/imgui_impl_vulkan.cpp)**

<!-- more -->

![Preview](https://raw.githubusercontent.com/BobLChen/VulkanTutorials/master/preview/6_ImageGUI.jpg)

## ImGUI

ImGUI的集成其实很简单，我们把它的实现抄过来即可。具体的实现我就不再赘述了，因为跟之前的Demo流程是一致的。这里只阐述一下需要注意的几个地方。

### ShaderCode

ImGUI需要使用到Vulkan的二进制Shader，为了方便起见，避免每次传入Shader路径，可以将Shader的二进制文件编码放到CPP文件里面。这样在创建ImGUI的时候直接从CPP中保存变量中获取。
```c++
static uint32_t g__glsl_shader_vert_spv[] =
{
    0x07230203,0x00010000,0x00080001,0x0000002e,0x00000000,0x00020011,0x00000001,0x0006000b,
    0x00000001,0x4c534c47,0x6474732e,0x3035342e,0x00000000,0x0003000e,0x00000000,0x00000001,
    0x000a000f,0x00000000,0x00000004,0x6e69616d,0x00000000,0x0000000b,0x0000000f,0x00000015,
    0x0000001b,0x0000001c,0x00030003,0x00000002,0x000001c2,0x00040005,0x00000004,0x6e69616d,
    0x00000000,0x00030005,0x00000009,0x00000000,0x00050006,0x00000009,0x00000000,0x6f6c6f43,
    0x00000072,0x00040006,0x00000009,0x00000001,0x00005655,0x00030005,0x0000000b,0x0074754f,
    0x00040005,0x0000000f,0x6c6f4361,0x0000726f,0x00030005,0x00000015,0x00565561,0x00060005,
    0x00000019,0x505f6c67,0x65567265,0x78657472,0x00000000,0x00060006,0x00000019,0x00000000,
    0x505f6c67,0x7469736f,0x006e6f69,0x00030005,0x0000001b,0x00000000,0x00040005,0x0000001c,
    0x736f5061,0x00000000,0x00060005,0x0000001e,0x73755075,0x6e6f4368,0x6e617473,0x00000074,
    0x00050006,0x0000001e,0x00000000,0x61635375,0x0000656c,0x00060006,0x0000001e,0x00000001,
    0x61725475,0x616c736e,0x00006574,0x00030005,0x00000020,0x00006370,0x00040047,0x0000000b,
    0x0000001e,0x00000000,0x00040047,0x0000000f,0x0000001e,0x00000002,0x00040047,0x00000015,
    0x0000001e,0x00000001,0x00050048,0x00000019,0x00000000,0x0000000b,0x00000000,0x00030047,
    0x00000019,0x00000002,0x00040047,0x0000001c,0x0000001e,0x00000000,0x00050048,0x0000001e,
    0x00000000,0x00000023,0x00000000,0x00050048,0x0000001e,0x00000001,0x00000023,0x00000008,
    0x00030047,0x0000001e,0x00000002,0x00020013,0x00000002,0x00030021,0x00000003,0x00000002,
    0x00030016,0x00000006,0x00000020,0x00040017,0x00000007,0x00000006,0x00000004,0x00040017,
    0x00000008,0x00000006,0x00000002,0x0004001e,0x00000009,0x00000007,0x00000008,0x00040020,
    0x0000000a,0x00000003,0x00000009,0x0004003b,0x0000000a,0x0000000b,0x00000003,0x00040015,
    0x0000000c,0x00000020,0x00000001,0x0004002b,0x0000000c,0x0000000d,0x00000000,0x00040020,
    0x0000000e,0x00000001,0x00000007,0x0004003b,0x0000000e,0x0000000f,0x00000001,0x00040020,
    0x00000011,0x00000003,0x00000007,0x0004002b,0x0000000c,0x00000013,0x00000001,0x00040020,
    0x00000014,0x00000001,0x00000008,0x0004003b,0x00000014,0x00000015,0x00000001,0x00040020,
    0x00000017,0x00000003,0x00000008,0x0003001e,0x00000019,0x00000007,0x00040020,0x0000001a,
    0x00000003,0x00000019,0x0004003b,0x0000001a,0x0000001b,0x00000003,0x0004003b,0x00000014,
    0x0000001c,0x00000001,0x0004001e,0x0000001e,0x00000008,0x00000008,0x00040020,0x0000001f,
    0x00000009,0x0000001e,0x0004003b,0x0000001f,0x00000020,0x00000009,0x00040020,0x00000021,
    0x00000009,0x00000008,0x0004002b,0x00000006,0x00000028,0x00000000,0x0004002b,0x00000006,
    0x00000029,0x3f800000,0x00050036,0x00000002,0x00000004,0x00000000,0x00000003,0x000200f8,
    0x00000005,0x0004003d,0x00000007,0x00000010,0x0000000f,0x00050041,0x00000011,0x00000012,
    0x0000000b,0x0000000d,0x0003003e,0x00000012,0x00000010,0x0004003d,0x00000008,0x00000016,
    0x00000015,0x00050041,0x00000017,0x00000018,0x0000000b,0x00000013,0x0003003e,0x00000018,
    0x00000016,0x0004003d,0x00000008,0x0000001d,0x0000001c,0x00050041,0x00000021,0x00000022,
    0x00000020,0x0000000d,0x0004003d,0x00000008,0x00000023,0x00000022,0x00050085,0x00000008,
    0x00000024,0x0000001d,0x00000023,0x00050041,0x00000021,0x00000025,0x00000020,0x00000013,
    0x0004003d,0x00000008,0x00000026,0x00000025,0x00050081,0x00000008,0x00000027,0x00000024,
    0x00000026,0x00050051,0x00000006,0x0000002a,0x00000027,0x00000000,0x00050051,0x00000006,
    0x0000002b,0x00000027,0x00000001,0x00070050,0x00000007,0x0000002c,0x0000002a,0x0000002b,
    0x00000028,0x00000029,0x00050041,0x00000011,0x0000002d,0x0000001b,0x0000000d,0x0003003e,
    0x0000002d,0x0000002c,0x000100fd,0x00010038
};

static uint32_t g__glsl_shader_frag_spv[] =
{
    0x07230203,0x00010000,0x00080001,0x0000001e,0x00000000,0x00020011,0x00000001,0x0006000b,
    0x00000001,0x4c534c47,0x6474732e,0x3035342e,0x00000000,0x0003000e,0x00000000,0x00000001,
    0x0007000f,0x00000004,0x00000004,0x6e69616d,0x00000000,0x00000009,0x0000000d,0x00030010,
    0x00000004,0x00000007,0x00030003,0x00000002,0x000001c2,0x00040005,0x00000004,0x6e69616d,
    0x00000000,0x00040005,0x00000009,0x6c6f4366,0x0000726f,0x00030005,0x0000000b,0x00000000,
    0x00050006,0x0000000b,0x00000000,0x6f6c6f43,0x00000072,0x00040006,0x0000000b,0x00000001,
    0x00005655,0x00030005,0x0000000d,0x00006e49,0x00050005,0x00000016,0x78655473,0x65727574,
    0x00000000,0x00040047,0x00000009,0x0000001e,0x00000000,0x00040047,0x0000000d,0x0000001e,
    0x00000000,0x00040047,0x00000016,0x00000022,0x00000000,0x00040047,0x00000016,0x00000021,
    0x00000000,0x00020013,0x00000002,0x00030021,0x00000003,0x00000002,0x00030016,0x00000006,
    0x00000020,0x00040017,0x00000007,0x00000006,0x00000004,0x00040020,0x00000008,0x00000003,
    0x00000007,0x0004003b,0x00000008,0x00000009,0x00000003,0x00040017,0x0000000a,0x00000006,
    0x00000002,0x0004001e,0x0000000b,0x00000007,0x0000000a,0x00040020,0x0000000c,0x00000001,
    0x0000000b,0x0004003b,0x0000000c,0x0000000d,0x00000001,0x00040015,0x0000000e,0x00000020,
    0x00000001,0x0004002b,0x0000000e,0x0000000f,0x00000000,0x00040020,0x00000010,0x00000001,
    0x00000007,0x00090019,0x00000013,0x00000006,0x00000001,0x00000000,0x00000000,0x00000000,
    0x00000001,0x00000000,0x0003001b,0x00000014,0x00000013,0x00040020,0x00000015,0x00000000,
    0x00000014,0x0004003b,0x00000015,0x00000016,0x00000000,0x0004002b,0x0000000e,0x00000018,
    0x00000001,0x00040020,0x00000019,0x00000001,0x0000000a,0x00050036,0x00000002,0x00000004,
    0x00000000,0x00000003,0x000200f8,0x00000005,0x00050041,0x00000010,0x00000011,0x0000000d,
    0x0000000f,0x0004003d,0x00000007,0x00000012,0x00000011,0x0004003d,0x00000014,0x00000017,
    0x00000016,0x00050041,0x00000019,0x0000001a,0x0000000d,0x00000018,0x0004003d,0x0000000a,
    0x0000001b,0x0000001a,0x00050057,0x00000007,0x0000001c,0x00000017,0x0000001b,0x00050085,
    0x00000007,0x0000001d,0x00000012,0x0000001c,0x0003003e,0x00000009,0x0000001d,0x000100fd,
    0x00010038
};
```

### Subpass
Pipeline如果使用在`Subpass`中，需要指定`Subpass`的索引。所以也需要将`Subpass`传递给ImGUI，告诉ImGUI当前渲染处于哪一个Subpass中。

### RenderPass
考虑到RenderPass可能因为窗口Resize而重建，因此RenderPass也要通知到ImGUI。

### 事件绑定
ImGUI毕竟是一个UI库，类似鼠标移动、点击、滚动等事件也需要一并通知到ImGUI。如果不通知到ImGUI，它是无法处理这些行为的，ImGUI本身不会捕获每个平台的这些事件。

```c++
void ImageGUIContext::StartFrame()
{
	const Vector2& mousePos = InputManager::GetMousePosition();
    ImGuiIO& io = ImGui::GetIO();
	io.MouseWheel  += InputManager::GetMouseDelta();
    io.MousePos     = ImVec2(mousePos.x * io.DisplayFramebufferScale.x, mousePos.y * io.DisplayFramebufferScale.y);
    io.MouseDown[0] = InputManager::IsMouseDown(MouseType::MOUSE_BUTTON_LEFT);
    io.MouseDown[1] = InputManager::IsMouseDown(MouseType::MOUSE_BUTTON_RIGHT);
    io.MouseDown[2] = InputManager::IsMouseDown(MouseType::MOUSE_BUTTON_MIDDLE);

	io.KeyCtrl  = InputManager::IsKeyDown(KeyboardType::KEY_LEFT_CONTROL) || InputManager::IsKeyDown(KeyboardType::KEY_RIGHT_CONTROL);
	io.KeyShift = InputManager::IsKeyDown(KeyboardType::KEY_LEFT_SHIFT) || InputManager::IsKeyDown(KeyboardType::KEY_RIGHT_SHIFT);
	io.KeyAlt   = InputManager::IsKeyDown(KeyboardType::KEY_LEFT_ALT) || InputManager::IsKeyDown(KeyboardType::KEY_RIGHT_ALT);
	io.KeySuper = false;

	ImGui::NewFrame();
}
```

### Update

当使用ImGUI时，如果有对UI进行操作，例如点开一个下拉列表等，这个时候出现了新的UI，也就意味ImGUI产生新的数据。这些UI最终是以三角形组合拼凑而成，因此VertexBuffer和IndexBuffer势必也会产生新的。这个时候就需要我们按需创建出对应的Buffer数据以供ImGUI使用。

```c++
bool ImageGUIContext::Update()
{
    ImDrawData* imDrawData = ImGui::GetDrawData();
	bool updateCmdBuffers  = false;

	if (!imDrawData) { 
		return false;
	};
    
	// Note: Alignment is done inside buffer creation
	VkDeviceSize vertexBufferSize = imDrawData->TotalVtxCount * sizeof(ImDrawVert);
	VkDeviceSize indexBufferSize  = imDrawData->TotalIdxCount * sizeof(ImDrawIdx);

	// Update buffers only if vertex or index count has been changed compared to current buffer size
	if ((vertexBufferSize == 0) || (indexBufferSize == 0)) {
		return false;
	}
	
	// Vertex buffer
	if ((m_VertexBuffer.buffer == VK_NULL_HANDLE) || (m_VertexCount != imDrawData->TotalVtxCount)) {
		m_VertexCount = imDrawData->TotalVtxCount;
		m_VertexBuffer.Unmap();
		m_VertexBuffer.Destroy();
		CreateBuffer(m_VertexBuffer, VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT, vertexBufferSize);
		m_VertexBuffer.Map();
		updateCmdBuffers = true;
	}
    
	// Index buffer
	if ((m_IndexBuffer.buffer == VK_NULL_HANDLE) || (m_IndexCount < imDrawData->TotalIdxCount)) {
		m_IndexCount = imDrawData->TotalIdxCount;
		m_IndexBuffer.Unmap();
		m_IndexBuffer.Destroy();
		CreateBuffer(m_IndexBuffer, VK_BUFFER_USAGE_INDEX_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT, indexBufferSize);
		m_IndexBuffer.Map();
		updateCmdBuffers = true;
	}

	// Upload data
	ImDrawVert* vtxDst = (ImDrawVert*)m_VertexBuffer.mapped;
	ImDrawIdx* idxDst  = (ImDrawIdx*)m_IndexBuffer.mapped;

	for (int n = 0; n < imDrawData->CmdListsCount; n++) {
		const ImDrawList* cmdList = imDrawData->CmdLists[n];
		memcpy(vtxDst, cmdList->VtxBuffer.Data, cmdList->VtxBuffer.Size * sizeof(ImDrawVert));
		memcpy(idxDst, cmdList->IdxBuffer.Data, cmdList->IdxBuffer.Size * sizeof(ImDrawIdx));
		vtxDst += cmdList->VtxBuffer.Size;
		idxDst += cmdList->IdxBuffer.Size;
	}

	m_VertexBuffer.Flush();
	m_IndexBuffer.Flush();

	return updateCmdBuffers || m_Updated;
}
```

### 集成到Demo
集成到Demo其实非常简单，在`Draw`函数内部增加一个`UpdateUI`函数，这个函数里面就负责进行UI逻辑处理。如果有发现修改了渲染数据，就重新录制一下渲染命令即可。
在录制渲染命令的时候，只需要把UI的绘制放到最后。如果放置到前面，可能会出现3D图像遮挡2D UI的情况。代码如下:

```c++
VERIFYVULKANRESULT(vkBeginCommandBuffer(m_CommandBuffers[i], &cmdBeginInfo));
            
vkCmdBeginRenderPass(m_CommandBuffers[i], &renderPassBeginInfo, VK_SUBPASS_CONTENTS_INLINE);
vkCmdSetViewport(m_CommandBuffers[i], 0, 1, &viewport);
vkCmdSetScissor(m_CommandBuffers[i], 0, 1, &scissor);

vkCmdBindDescriptorSets(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_PipelineLayout, 0, 1, &m_DescriptorSet, 0, nullptr);
vkCmdBindPipeline(m_CommandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, m_Pipeline);
vkCmdBindVertexBuffers(m_CommandBuffers[i], 0, 1, &(m_VertexBuffer->buffer), offsets);
vkCmdBindIndexBuffer(m_CommandBuffers[i], m_IndexBuffer->buffer, 0, VK_INDEX_TYPE_UINT16);
vkCmdDrawIndexed(m_CommandBuffers[i], m_IndicesCount, 1, 0, 0, 0);

m_GUI->BindDrawCmd(m_CommandBuffers[i], m_RenderPass);

vkCmdEndRenderPass(m_CommandBuffers[i]);

VERIFYVULKANRESULT(vkEndCommandBuffer(m_CommandBuffers[i]));
```

综上所述就完成了UI的集成，后续的Demo就可以把参数可视化。