---
title: 21_Stencil
date: 2019-09-27 23:24:11
tags:
- Vulkan
- 3D
- Stencil
- Tutorial
categories:
- Vulkan
---

[21_Stencil](https://github.com/BobLChen/VulkanDemos/tree/master/examples/21_Stencil)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

今天来看一个比较好玩儿的东西，模板缓冲技术。模板缓冲技术用的地方比较多。例如UI的遮罩功能，部分渲染的性能优化，特效等等。在这个Demo里面通过一个简单的特效来展示Stencil的用法。

<!-- more -->

![21_Stencil](https://raw.githubusercontent.com/BobLChen/VulkanDemos/master/preview/21_Stencil.jpg)

## 透视特效

正如预览图里面看到的效果，角色没有被遮挡的正常显示，被遮挡的显示特效，这个效果跟火炬之光系列里面的效果非常类似，使用的技术也是非常类似。

被遮挡部分显示的效果效果与正常部分不一样，显然被遮挡的部分需要额外的Shader来进行渲染。仔细观察一下这个特效，其实不难看出，这个特效的实现是非常简单的。回想一下光照的效果，光线方向与法线点乘得到角度值，通过角度值来决定明暗，也就是光线跟法线垂直的地方非常暗，光线平行的地方非常亮。这个效果其实刚好相反，垂直的地方非常亮，平行的地方非常暗，用这个明暗值来调节透明度，光线方向也改为使用视线方向。Shader代码如下:

```glsl
#version 450

layout (location = 0) in vec3 inPosition;
layout (location = 1) in vec2 inUV0;
layout (location = 2) in vec3 inNormal;

layout (binding = 0) uniform MVPBlock 
{
	mat4 modelMatrix;
	mat4 viewMatrix;
	mat4 projectionMatrix;
} uboMVP;

layout (location = 0) out vec2 outUV;
layout (location = 1) out vec3 outNormal;

out gl_PerVertex 
{
    vec4 gl_Position;   
};

void main() 
{
	mat3 normalMatrix = transpose(inverse(mat3(uboMVP.modelMatrix)));
	vec3 normal  = normalize(normalMatrix * inNormal.xyz);
	
	outUV        = inUV0;
	outNormal    = normal;
	
	gl_Position  = uboMVP.projectionMatrix * uboMVP.viewMatrix * uboMVP.modelMatrix * vec4(inPosition.xyz, 1.0);
}
```

```glsl
#version 450

layout (location = 0) in vec2 inUV;
layout (location = 1) in vec3 inNormal;

layout (binding  = 1) uniform ParamBlock
{
    vec3    color;
    float   power;
	vec3    viewDir;
    float   padding;
} rayParam;

layout (location = 0) out vec4 outFragColor;

void main() 
{
    float vdotn = dot(normalize(rayParam.viewDir), inNormal);
    vdotn = 1.0 - clamp(vdotn, 0, 1);
    vdotn = pow(vdotn, rayParam.power);
    outFragColor.xyz = rayParam.color * vdotn;
    outFragColor.w   = 1.0;
}
```

## 模板缓冲

模板缓冲技术其实非常简单，简单的讲就是为每一个像素点都分配了一个8位的数据用于存储模板值。通过存储的值与参考的值进行比较决定这个像素点是否启用。举个例子：

```c++
struct Pixel
{
    uint8 r;
    uint8 g;
    uint8 b;
    uint8 a;
    
    uint8 stencil;
};

Pixel backbuffer[1024 * 768];
```

每个像素点不仅存储了颜色的数据，还存储了Stencil的数据。假设最初的状态什么都没有，默认的stencil值是0，那backbuffer的效果就如下图所示(注意示意图是表达的stencil的值)：

![0.png](0.png)

然后设置一个参考值127，既然有参考值那就有比较条件，假设条件是大于等于的时候就替换backbuffer里面的stencil值为参考值。假如在屏幕中间渲染一个正方形，那结果就如下：

![1.png](1.png)

然后设置一个参考值255，条件任然是大于等于的时候就替换为参考值。假如在其它的地方又画了一个正方形，那结果就如下：

![2.png](2.png)

Vulkan API不仅仅提供了这些功能，还提供了更多的一些功能，主要是跟深度测试一起使用。
- 比较方法：无非就是LESS、EQUAL等之类的方法
- 失败行为：就是**参考值**跟当前**存储的值**按照**比较方法**去进行比较，失败时的行为。例如用参考值替换掉存储的值；保持当前存储的值；当前的值递增/递减等行为，目的就是如何影响存储的值。
- 成功行为：同上
- 深度测试失败行为：同上，稍微不同的是，这里是指深度测试失败的行为。

另外需要注意的是深度测试的最小单位是以**三角形(以设置的值为准)**为准，并不是我们调用DrawCall的一个模型为单位。这里其实是最关键的，稍不注意可能就会掉入坑里。模型是由三角形组成的，在绘制的时候三角形在模型里面的顺序不能得到保证，那么在模板测试的时候就有特别注意，尤其是测试通过/失败的行为。错误的使用可能会导致跟预期的不一致，例如预期的是整个模型统一使用一个stencil值，但是由于错误的使用可能导致模型对应的stencil值完全不一致。

### 设置Stencil

首先设置小房子的Stencil状态，无论情况是怎样的，都希望小房子所占区域的stencil都为1。

```c++
// room material
m_SceneMaterials.resize(m_SceneDiffuses.size());
for (int32 i = 0; i < m_SceneMaterials.size(); ++i)
{
    m_SceneMaterials[i] = vk_demo::DVKMaterial::Create(
        m_VulkanDevice,
        m_RenderPass,
        m_PipelineCache,
        m_DiffuseShader
    );
    VkPipelineDepthStencilStateCreateInfo& depthStencilState = m_SceneMaterials[i]->pipelineInfo.depthStencilState;
    depthStencilState.stencilTestEnable = VK_TRUE;
    depthStencilState.back.reference    = 1;
    depthStencilState.back.writeMask    = 0xFF;
    depthStencilState.back.compareMask  = 0xFF;
    depthStencilState.back.compareOp    = VK_COMPARE_OP_ALWAYS;
    depthStencilState.back.failOp       = VK_STENCIL_OP_REPLACE;
    depthStencilState.back.depthFailOp  = VK_STENCIL_OP_REPLACE;
    depthStencilState.back.passOp       = VK_STENCIL_OP_REPLACE;
    depthStencilState.front             = depthStencilState.back;
    m_SceneMaterials[i]->PreparePipeline();
    m_SceneMaterials[i]->SetTexture("diffuseMap", m_SceneDiffuses[i]);
}
```

然后设置角色的stencil状态，角色的正常状态正常渲染即可。

```c++
// role material
m_RoleMaterial = vk_demo::DVKMaterial::Create(
    m_VulkanDevice,
    m_RenderPass,
    m_PipelineCache,
    m_DiffuseShader
);
m_RoleMaterial->PreparePipeline();
m_RoleMaterial->SetTexture("diffuseMap", m_RoleDiffuse);
```

最好设置角色的特效stencil状态，我们期望的是被房子挡住的部分显示特效，那其实就是跟房子重合的部分显示即可，小房子的stencil值已经统一设置为1，只需要跟1比较即可。

```c++
// ray material
m_RayMaterial = vk_demo::DVKMaterial::Create(
    m_VulkanDevice,
    m_RenderPass,
    m_PipelineCache,
    m_RayShader
);
VkPipelineColorBlendAttachmentState& blendState = m_RayMaterial->pipelineInfo.blendAttachmentStates[0];
blendState.blendEnable         = VK_TRUE;
blendState.srcColorBlendFactor = VK_BLEND_FACTOR_ONE;
blendState.dstColorBlendFactor = VK_BLEND_FACTOR_ONE;
blendState.colorBlendOp        = VK_BLEND_OP_ADD;

VkPipelineDepthStencilStateCreateInfo& depthStencilState = m_RayMaterial->pipelineInfo.depthStencilState;
depthStencilState.stencilTestEnable = VK_TRUE;
depthStencilState.back.reference    = 1;
depthStencilState.back.writeMask    = 0xFF;
depthStencilState.back.compareMask  = 0xFF;
depthStencilState.back.compareOp    = VK_COMPARE_OP_EQUAL;
depthStencilState.back.failOp       = VK_STENCIL_OP_KEEP;
depthStencilState.back.depthFailOp  = VK_STENCIL_OP_KEEP;
depthStencilState.back.passOp       = VK_STENCIL_OP_REPLACE;
depthStencilState.front             = depthStencilState.back;
depthStencilState.depthTestEnable   = VK_FALSE;
m_RayMaterial->PreparePipeline();
```

### 绘制

最后配置一下渲染的流程，代码如下：

```c++
// role
vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, m_RoleMaterial->GetPipeline());
for (int32 meshIndex = 0; meshIndex < m_ModelRole->meshes.size(); ++meshIndex) {
    m_RoleMaterial->BindDescriptorSets(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, meshIndex);
    m_ModelRole->meshes[meshIndex]->BindDrawCmd(commandBuffer);
}

// room
for (int32 i = 0; i < m_SceneMatMeshes.size(); ++i)
{
    vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, m_SceneMaterials[i]->GetPipeline());
    for (int32 j = 0; j < m_SceneMatMeshes[i].size(); ++j) {
        m_SceneMaterials[i]->BindDescriptorSets(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, j);
        m_SceneMatMeshes[i][j]->BindDrawCmd(commandBuffer);
    }
}

// ray
vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, m_RayMaterial->GetPipeline());
for (int32 meshIndex = 0; meshIndex < m_ModelRole->meshes.size(); ++meshIndex) {
    m_RayMaterial->BindDescriptorSets(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, meshIndex);
    m_ModelRole->meshes[meshIndex]->BindDrawCmd(commandBuffer);
}
```

核心思想其实很简单，模板测试失败了就不写入当前像素的数据，测试有条件供我们选择，测试成功/失败之后可以调整stencil值，需要跟深度测试一起联合使用。最终的效果也就是预览图那样，利用得当其实可以做出一些炫酷的效果出来的。


