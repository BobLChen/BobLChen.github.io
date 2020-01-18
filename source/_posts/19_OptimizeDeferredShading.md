---
title: 19_OptimizeDeferredShading
date: 2019-09-14 23:55:29
tags:
- Vulkan
- 物理设备
- 3D
- Demo
categories:
- Vulkan
---

[19_OptimizeDeferredShading](https://github.com/BobLChen/VulkanDemos/tree/master/examples/19_OptimizeDeferredShading)

[项目Github地址请戳我](https://github.com/BobLChen/VulkanDemos)

虽说上个Demo是个非常简易版本的延迟渲染案例，但是其中也是有部分可以优化的地方。为了这个优化，我又新增了一个Demo用于说明如何进行优化。之所新增一个Demo，是因为这个优化在后面会大量用到，这个优化看似很小但是其实很常用，我觉得你们应该会大量遇到。

<!-- more -->

![19_OptimizeDeferredShading](https://raw.githubusercontent.com/BobLChen/VulkanDemos/master/preview/19_OptimizeDeferredShading_1.jpg)

## 19_OptimizeDeferredShading

### 带宽压力
回想一下在上个Demo里面我们输出了多少数据？我们可以列举一下：Color、Depth、Normal、Position，虽然看起来才四样，但是其实数据量是非常恐怖的。

- Color：VK_FORMAT_R8G8B8A8_UNORM 32位
- Depth：VK_FORMAT_D24_UNORM_S8_UINT 24位以上
- Normal：VK_FORMAT_R16G16B16A16_SFLOAT 64位
- Position：VK_FORMAT_R16G16B16A16_SFLOAT 64位

对于每一个像素，我们一共用了`(32 + 24 + 64 + 64)位数据`，现在2K的显示器已经是常规显示器了，像素总量是`2560 * 1080`，最终呈现完整个图像所需的数据就是`(32 + 24 + 64 + 64) * (2560 * 1080) / 8 / 1024 / 1024 = 60MB`。如果我们的目标是144FPS，这个量可能承受不起。。。

观察我们的所需数据其实不难发现，有一个明显的地方是可以优化掉的，即法线数据。

### 优化法线

我们之所以采用`VK_FORMAT_R16G16B16A16_SFLOAT`来存储法线，是因为法线数据带有方向。既然是万向的数据，每个分量的数据范围就是[-1, 1]。由于16Bit的精度已经可以容纳[-1, 1]，我们采用了`VK_FORMAT_R16G16B16A16_SFLOAT`这样一种格式。

仔细观察[-1, 1]其实不难发现，我们可以通过简单的数学运算，将[-1, 1]范围缩放到[0, 1]范围，即`x * 0.5 + 0.5`。从[0, 1]范围恢复到[-1, 1]范围也不难，`x * 2.0 - 1.0`即可。既然可以通过这种方式进行转化，那么我们就可以用无符号8Bit的格式来进行存储，即`VK_FORMAT_R8G8B8A8_UNORM`数据格式。

在fragment shader里面代码如下：
```glsl
#version 450

layout (location = 0) in vec3 inNormal;
layout (location = 1) in vec3 inColor;

layout (location = 0) out vec4 outFragColor;
layout (location = 1) out vec4 outNormal;

void main() 
{
    vec3 normal = normalize(inNormal);
    // float NDotL = clamp(dot(normal, normalize(vec3(0, 1, -1))), 0, 1.0);

    // 这里进行了转化
    // [-1, 1] -> [0, 1]
    normal       = (normal + 1) / 2;
    outNormal    = vec4(normal, 1.0);

    outFragColor = vec4(inColor, 1.0);
}

```

### 优化Position

法线数据优化完成之后，我们再来看看Position数据是否也可以优化掉。回想一下射线方程的定义，我们知道一个坐标点可以通过起点+射线+距离计算出来，即`pos = start + ray * t`。

由于vertex shader传递给fragment shader的数据会被插值，根据这个特性，我们可以把`Quad`的坐标数据传递给Fragment shader，这样在任意一个像素点上都能得到一个[-1, 1]之间的数值。又因为在`View空间(Camera)`下，相机的坐标是在(0, 0, 0)点，将插值之后的点跟相机原点连接，即可得到每个像素点的射线数据。现在唯一缺少是数据`t`，这个数据其实可以通过深度数据计算出来。有了`ray`、`t`以及`start`数据之后，我们可以很轻易的计算出某个像素点在世界空间的坐标点。

### 非线性深度转线性深度

为什么会有非线性深度和线性深度的区分？回想一下`Projection`矩阵的构建过程，深度的输出结果并不是单纯的一个`z`，而是加了一些其它数据进去。因此这个深度并不能直接拿来当成`z`使用，须得进行转化，计算出`z`。这个转化过程跟我们使用的轴`NDC`空间有关，例如在Z轴方向范围是[0, 1]与[-1, 1]的计算方式是不一样的。这里我就不再推导如何还原出来线性深度的过程，这个过程很简单，只是简单的加减乘除运算。计算过程如下：

```c++
m_VertFragParam.zNear   = 10.0f;
m_VertFragParam.zFar    = 3000.0f;
m_VertFragParam.one     = 1.0f;
m_VertFragParam.yMaxFar = m_VertFragParam.zFar * MMath::Tan(MMath::DegreesToRadians(45.0f) / 2);
m_VertFragParam.xMaxFar = m_VertFragParam.yMaxFar * (float)GetWidth() / (float)GetHeight();

Matrix4x4 modelViewProj;
modelViewProj.Append(m_ViewProjData.view);
modelViewProj.Append(m_ViewProjData.projection);

Vector4 realPos(10, 20, 30, 1.0f);
Vector4 posView = m_ViewProjData.view.TransformPosition(realPos);
Vector4 posProj = modelViewProj.TransformVector4(realPos);
float depth = posProj.z / posProj.w;

float zc0 = 1.0 - m_VertFragParam.zFar / m_VertFragParam.zNear;
float zc1 = m_VertFragParam.zFar / m_VertFragParam.zNear; 
float depth01 = 1.0 / (zc0 * depth + zc1);
```

### 组装射线

组装射线就变得非常简单了，根据前面提到的，我们可以很轻易的组装出`View sapce`下的射线，代码如下：

```c++
// ndc space
float u = posProj.x / posProj.w;
float v = posProj.y / posProj.w;
// view space ray
Vector3 viewRay = Vector3(m_VertFragParam.xMaxFar * u, m_VertFragParam.yMaxFar * v, m_VertFragParam.zFar);
Vector3 viewPos = viewRay * depth01;
```

由代码可以看出，可以根据`u、v`以及视椎体的尺寸组织处`ray`。在gpu里面同样可以根据这样的方式组织出来，并且由于在fragment shader里面`u、v`会被插值，我们可以组织出每一个像素的的`ray`以及`view space position`。

有了`view space`下的坐标点数据只会，根据`view`的逆矩阵轻易的可以得到`世界空间`下的坐标点。

完整的推导代码如下：
```c++
{
    // fast reconstruct position from eye linear depth
    Matrix4x4 modelViewProj;
    modelViewProj.Append(m_ViewProjData.view);
    modelViewProj.Append(m_ViewProjData.projection);

    // real world position
    Vector4 realPos(10, 20, 30, 1.0f);
    // view space position
    Vector4 posView = m_ViewProjData.view.TransformPosition(realPos);
    // projection space position
    Vector4 posProj = modelViewProj.TransformVector4(realPos);
    // none linear depth
    float depth = posProj.z / posProj.w;
    // linear depth
    float zc0 = 1.0 - m_VertFragParam.zFar / m_VertFragParam.zNear;
    float zc1 = m_VertFragParam.zFar / m_VertFragParam.zNear; 
    float depth01 = 1.0 / (zc0 * depth + zc1);
    MLOG("LinearDepth:%f,%f", posProj.w / m_VertFragParam.zFar, depth01);
    // ndc space
    float u = posProj.x / posProj.w;
    float v = posProj.y / posProj.w;
    // view space ray
    Vector3 viewRay = Vector3(m_VertFragParam.xMaxFar * u, m_VertFragParam.yMaxFar * v, m_VertFragParam.zFar);
    Vector3 viewPos = viewRay * depth01;

    MLOG("posView:(%f,%f,%f) - (%f,%f,%f)", posView.x, posView.y, posView.z, viewPos.x, viewPos.y, viewPos.z);
}
```

### Shader应用

在Shader里面，可以按照上述C++的代码流程实现一遍。
`obj.vert`的代码如下：
```glsl
#version 450

layout (location = 0) in vec3 inPosition;
layout (location = 1) in vec3 inNormal;
layout (location = 2) in vec3 inColor;

layout (binding = 0) uniform ViewProjBlock 
{
	mat4 viewMatrix;
	mat4 projectionMatrix;
} uboViewProj;

layout (binding = 1) uniform ModelDynamicBlock
{
	mat4 modelMatrix;
} uboModel;

layout (location = 0) out vec3 outNormal;
layout (location = 1) out vec3 outColor;

out gl_PerVertex 
{
    vec4 gl_Position;   
};

void main() 
{
	mat3 normalMatrix = transpose(inverse(mat3(uboModel.modelMatrix)));
	vec3 normal = normalize(normalMatrix * inNormal);

	gl_Position = uboViewProj.projectionMatrix * uboViewProj.viewMatrix * uboModel.modelMatrix * vec4(inPosition.xyz, 1.0);
	outNormal   = normal;
	outColor	= inColor;
}

```

`obj.frag`代码如下：
```glsl
#version 450

layout (location = 0) in vec3 inNormal;
layout (location = 1) in vec3 inColor;

layout (location = 0) out vec4 outFragColor;
layout (location = 1) out vec4 outNormal;

void main() 
{
    vec3 normal = normalize(inNormal);
    // float NDotL = clamp(dot(normal, normalize(vec3(0, 1, -1))), 0, 1.0);

    // [-1, 1] -> [0, 1]
    normal       = (normal + 1) / 2;
    outNormal    = vec4(normal, 1.0);

    outFragColor = vec4(inColor, 1.0);
}

```

`quad.vert`代码如下：
```glsl
#version 450

layout (location = 0) in vec3 inPosition;
layout (location = 1) in vec2 inUV0;

layout (location = 0) out vec2 outUV0;
layout (location = 1) out vec4 outRay;

layout (binding = 3) uniform ParamBlock
{
	vec4 param0;	// (attachmentIndex, zNear, zFar, one)
	vec4 param1;	// (xMaxFar, yMaxFar, padding, padding)
	mat4 invView;
} paramData;

out gl_PerVertex {
	vec4 gl_Position;
};

void main() 
{
	gl_Position = vec4(inPosition, 1.0f);

	float xMaxFar = paramData.param1.x;
	float yMaxFar = paramData.param1.y;
	float zFar    = paramData.param0.z;
	
	outRay.x = inPosition.x * xMaxFar;
	outRay.y = inPosition.y * yMaxFar;
	outRay.z = zFar;
	outRay.w = 1.0;
	
	outUV0   = inUV0;
}

```

`quad.frag`代码如下：
```glsl
#version 450

layout (input_attachment_index = 0, set = 0, binding = 0) uniform subpassInput inputColor;
layout (input_attachment_index = 1, set = 0, binding = 1) uniform subpassInput inputNormal;
layout (input_attachment_index = 2, set = 0, binding = 2) uniform subpassInput inputDepth;

layout (binding = 3) uniform ParamBlock
{
	vec4 param0;	// (attachmentIndex, zNear, zFar, one)
	vec4 param1;	// (xMaxFar, yMaxFar, padding, padding)
	mat4 invView;
} paramData;

#define NUM_LIGHTS 64
struct PointLight {
	vec4 position;
	vec4 colorAndRadius;
};

layout (binding = 5) uniform LightDataBlock
{
	PointLight lights[NUM_LIGHTS];
} lightDatas;

layout (location = 0) in vec2 inUV0;
layout (location = 1) in vec4 inRay;

layout (location = 0) out vec4 outFragColor;

vec4 zBufferParams;

float Linear01Depth(float z)
{
	return 1.0 / (zBufferParams.x * z + zBufferParams.y);
}

float LinearEyeDepth(float z)
{
	return 1.0 / (zBufferParams.z * z + zBufferParams.w);
}

float DoAttenuation(float range, float d)
{
    return 1.0 - smoothstep(range * 0.75, range, d);
}

void main() 
{
	int attachmentIndex = int(paramData.param0.x);
	float zNear = paramData.param0.y;
	float zFar  = paramData.param0.z;
	float xMaxFar = paramData.param1.x;
	float yMaxFar = paramData.param1.y;

	float zc0 = 1.0 - zFar / zNear;
	float zc1 = zFar / zNear;
	zBufferParams = vec4(zc0, zc1, zc0 / zFar, zc1 / zFar);

	// world position
	float depth   = subpassLoad(inputDepth).r;
	float realZ01 = Linear01Depth(depth);
	vec4 position = vec4(inRay.xyz * realZ01, 1.0);
	position = paramData.invView * position;

	// normal [0, 1] -> [-1, 1]
	vec4 normal  = subpassLoad(inputNormal);
	normal = normal * 2 - 1;

	// albedo color
	vec4 albedo  = subpassLoad(inputColor);

	if (attachmentIndex == 0) {
		vec4 ambient  = vec4(0.20);
		outFragColor  = vec4(0.0) + ambient;
		for (int i = 0; i < NUM_LIGHTS; ++i)
		{
			vec3 lightDir = lightDatas.lights[i].position.xyz - position.xyz;
			float dist    = length(lightDir);
			float atten   = DoAttenuation(lightDatas.lights[i].colorAndRadius.w, dist);
			float ndotl   = max(0.0, dot(normal.xyz, normalize(lightDir)));
			vec3 diffuse  = lightDatas.lights[i].colorAndRadius.xyz * albedo.xyz * ndotl * atten;

			outFragColor.xyz += diffuse;
		}
	}
	else if (attachmentIndex == 1) {
		outFragColor = albedo;
	} 
	else if (attachmentIndex == 2) {
		outFragColor = position / 1500;
	}
	else if (attachmentIndex == 3) {
		outFragColor = normal;
	}
	else {
		// undefined
		outFragColor = vec4(1, 0, 0, 1.0);
	}
}

```

整个优化其实改动非常小，但是数据量可以减少很多。在完善的延迟渲染流程里面，就是用到的上述的优化手段，根据项目需求不同，其实还可以进一步的优化。