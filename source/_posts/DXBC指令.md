---
title: DXBC指令
date: 2021-01-16 21:53:13
tags:
- Shader
- 3D
categories:
- Shader
---

# DXBC指令

DXBC指令是D3D着色器语言使用的指令，HLSL高级着色语言经过编译器编译之后，会生成相应的DXBC指令。DXBC指令可以理解为GPU需要真正执行的指令。OpengGL或者说是其它GPU厂商，他们提供的指令其实跟DXBC大同小异，略微有差异的也只是某些特殊的指令不是硬件支持而已。虽然编译器大部分情况下可以帮助我们优化代码，但是由于编译器特别智能，在某些情况下并不能保证代码是最优的方式。了解DXBC指令可以帮助我们在编写Shader时，能够写出更加可靠、性能优异的代码。另外了解DXBC指令，在某些情况下可以帮助我们更好的逆向其它游戏的一些材质效果。

<!-- more -->

DXBC指令其实非常简单，学习起来也非常容易，它的套路可以说是非常的单一，大多数指令可以归纳为这样的形式：

```assembly
op dest.[mask], src0.[swizzle], src1.[swizzle]
```

需要注意的是`[mask]`与`[swizzle]`这两个的差异。

- **swizzle**
指将源寄存器的任何组件复制到任何临时寄存器组件的能力。`swizzle`不会影响源寄存器的值，它只是把值拷贝给了另外一个**临时**的寄存器。举个例子：`.zxxy`意思就是`.x=.z; .y=.x; .z=.x; .w=.x;`。如果组件数量不够，则以重复最后一个，例如：`.xy = .xyyy; .wzx = .wzxx; .z = .zzzz;`。
- **mask**
作用在目标寄存器上的，称为`mask`，跟`swizzle`不一样，虽然看起来都是一样的用法。`mask`规则很简单，只能使用`xyzw`，例如：`mul r0.xy, c10, c10; mul r0.xyz, c10, c10; mul r0.xyzw, c10, c10;`。

现在介绍一些常用的指令：

## dcl_constantBuffer

```assembly
dcl_constantBuffer cbN[size], AccessPattern
```

声明一个Constant buffer，例如：

```
dcl_constantbuffer cb0[19], immediateIndexed
```

- immediateIndexed：常数值作为索引
- dynamic_indexed ：表达式作为索引

## dcl_globalFlags

```assembly
dcl_globalFlags flags
```

定义Shader的全局Flag，例如：

```assembly
dcl_globalFlags refactoringAllowed
```

这个Flag用于标识出驱动是不是可以对算术操作进行重排优化。例如下所示：

```hlsl
// 原始代码
a = b * c + b * d + b * e + b * f
// 优化之后的代码
a = dot4((b, b, b, b), (c, d, e, f))

// 原始代码
mul r0.x, cb0[0].x, r1.x
mul r0.y, cb0[0].y, r1.x
mul r0.z, cb0[0].z, r1.x
mul r0.w, cb0[0].w, r1.x
add r0.x, r0.x, r0.y
add r0.y, r0.z, r0.w
add r0.x, r0.x, r0.y
// 优化之后的代码
dot r0.x, r1.xxxx, cb0[0].xyzw
```

## dcl_immediateConstantBuffer 

```
dcl_immediateConstantBuffer value
```

声明一个`immediateConstantBuffer`。`immediateConstantBuffer`其实就是我们在HLSL里面写的一些常量的数据。`immediateConstantBuffer`是有数量限制的，不能超过4096。在以前的经验中，代码里面写的常数其实都算进去了，后面就不太清楚了。如下所示：

```hlsl
// HLSL代码
static const float4 BlurWeights[]={
	float4(0.002216,0.008764,0.026995,0.064759),
	float4(0.120985,0.176033,0.199471,0.176033),
    ...
};
// DXBC
dcl_immediateConstantBuffer {{0.002216,0.008764,0.026995,0.064759}, {0.120985,0.176033,0.199471,0.176033} ... }
// PS:以前的经验中，在其它平台，类似float4 a = b * float4(1, 2, 3, 4)中的float4(1, 2, 3, 4)会算进到immediateConstantBuffer中。
```

## dcl_indexableTemp 

```assembly
dcl_indexableTemp xN[size], ComponentCount
```

声明临时的可索引的数组。例如下所示：

```assembly
// HLSL代码
int statusStack[40];
// DXBC代码
dcl_indexableTemp x0[40], 4
```

其中`40`表示长度，`4`表示分量数。

## dcl_input 

```assembly
dcl_input vN[.mask][, interpolationMode]
```

声明输入寄存器，其中`N`是寄存器的编号，`interpolationMode`可选参数，当用作`PS`输入时，可以设置插值方式。如下所示：

```assembly
// HLSL代码
struct VS_INPUT
{
    float4 vPosition    : POSITION;
    float3 vNormal      : NORMAL;
    float2 vTexcoord    : TEXCOORD0;
};

// DXBC代码
dcl_input v0.xyzw
dcl_input v1.xyz
dcl_input v2.xy
```

## dcl_output 

```assembly
dcl_output oN[.mask]
```

声明输出的寄存器，其中N是寄存器编号。如下所示：

```assembly
// HLSL代码
struct VS_OUTPUT
{
    float4 viewPosition   : POSITION;
    float3 vNormal        : NORMAL;
    float2 vTexcoord      : TEXCOORD0;
    float4 vPosition      : SV_POSITION;
};
// DXBC代码
dcl_output o0.xyzw
dcl_output o1.xyz
dcl_output o2.xy
dcl_output_siv o3.xyzw, position
```

## dcl_output_siv

```assembly
dcl_output_siv oN[.masks], systemValue
```

声明一个包含系统值得输出寄存器。例如：

```assembly
// HLSL
struct VS_OUTPUT
{
    float4 Position   : SV_Position;
};

// DXBC
dcl_output_siv o0.xyzw, position
```

## dcl_resource

```assembly
dcl_resource tN, resourceType, returnType(s)
```

定义一个着色器输入资源。例如：

```assembly
// HLSL
Texture2D TextureBase;

// DXBC
dcl_resource_texture2d (float,float,float,float) t0
```

## dcl_sampler

```assembly
dcl_sampler sN, mode
```

定义一个采样寄存器。例如：

```assembly
// HLSL
SamplerState TextureSampler
{
    Filter = MIN_MAG_MIP_LINEAR;
    AddressU = Wrap;
    AddressV = Wrap;
};

// DXBC
dcl_sampler s0, mode_default

```

## dcl_temps

```assembly
dcl_temps N
```

声明临时寄存器，其中的`N`表示多少个。例如：

```assembly
dcl_temps 10; // 表示声明了r0,r1,r2,r3,r4,r5,r6,r7,r8,r9这些临时寄存器
```

## discard

``` assembly
discard{_z|_nz} src0.select_component
```

有条件的标记像素着色结果被丢弃，例如：

```assembly
// HLSL
if (input.pos.w == 1.0)
		discard;
// DXBC
eq r0.x, v0.w, l(1.000000) // true:0xFFFFFFFF false:0x0000000
discard_nz r0.x
```

其中`discard_nz`表示不为0时，丢弃；`discard_z`表示为0时，丢弃。
 
## add iadd uadd

```assembly
add dest.[mask], src0.[swizzle], src1.[swizzle]
```

`add`用于加法运算，`dest`为存储结果的变量，`src0`以及`src1` 为俩加数，例如：

```assembly
// HLSL
float4 pos = in_pos + vars;
// DXBC
add r0.xyzw, v0.xyzw, cb0[0].xyzw
```

## and

```assembly
and dest[.mask], src0[.swizzle], src1[.swizzle]
```

按位和，例如:

```assembly
// HLSL
float4 value;
if (vPos.x == 1 && vPos.y == 1)
    value = 0;
else
    value = 1;
oPosition = value;
// DXBC
eq r0.xy, v0.xyxx, l(1.000000, 1.000000, 0.000000, 0.000000)
and r0.x, r0.y, r0.x
// movc按条件赋值，r0.xxxx不为0，就用l(0,0,0,0)，否者就用l(1.000000,1.000000,1.000000,1.000000)
movc o0.xyzw, r0.xxxx, l(0,0,0,0), l(1.000000,1.000000,1.000000,1.000000)
```

## abs

```assembly
abs dest[.mask], src0[.swizzle]
```

计算绝对值。

## div

```assembly
div[_sat] dest[.mask], [-]src0[_abs][.swizzle] [-]src1[_abs][.swizzle]
```

除法指令，例如：

```assembly
// HLSL
uniform float2 uvOrig;
uniform float4 _CenterRadius;

float2 offset = uvOrig;
float angle = 1.0 - length(offset / _CenterRadius.zw);

// DXBC
div r0.xy, cb0[0].xyxx, cb0[1].zwzz
```

## dp2 dp3 dp4

```assembly
dpN[_sat] dest[.mask], [-]src0[_abs][.swizzle], [-]src1[_abs][.swizzle]
```

点乘指令，算式为`dest=src0.x * src1.x + src0.y * src1.y ... `。例如：

```assembly
// HLSL
float4 pos = in_pos + vars;
out_pos = mul(pos, world_view_proj);

// DXBC
add r0.xyzw, v0.xyzw, cb0[0].xyzw
dp4 o0.x, r0.xyzw, cb1[0].xyzw
dp4 o0.y, r0.xyzw, cb1[1].xyzw
dp4 o0.z, r0.xyzw, cb1[2].xyzw
dp4 o0.w, r0.xyzw, cb1[3].xyzw
```

以上我们可以发现，在HLSL里面，`mul`函数其实是由四个`dp4`指令完成的。

## eq ieq ueq

```assembly
eq dest[.mask], [-]src0[_abs][.swizzle], [-]src1[_abs][.swizzle]
```

按照分量进行浮点数比较，如果比较结果为true，则返回`0xFFFFFFFF `到dest，否者返回`0x0000000`。例如：

```assembly
// HLSL
if (input.pos.w == 1.0)
		discard;
// DXBC
eq r0.x, v0.w, l(1.000000) // true:0xFFFFFFFF false:0x0000000
discard_nz r0.x
```

## exp

```assembly
exp[_sat] dest[.mask], [-]src0[_abs][.swizzle]
```

以e为底数，src0为指数的指数运算。

## frc

```assembly
frc[_sat] dest[.mask], [-] src0[_abs][.swizzle]
```

提取小数部分，算式为`dest = src0 - floor(src0)`。例如：

```assembly
// HLSL
frac(colorScroll.z * gameTime.w);

// DXBC
mul r0.x, cb0[11].z, cb1[69].w
frc r0.x, r0.x
```

## ftoi ftuo itof utof

```assembly
ftoi dest[.mask], [-]src0[_abs][.swizzle]
ftuo dest[.mask], [-]src0[_abs][.swizzle]
itof dest[.mask], [-]src0[_abs][.swizzle]
utof dest[.mask], [-]src0[_abs][.swizzle]
```

浮点与整数之间的转换。

## ge ige uge

```assembly
ge dest[.mask], [-]src0[_abs][.swizzle], [-]src1[_abs][.swizzle]
```

大于等于指令，如果为`true`，则结果为0xFFFFFFFF；如果为`false`，则结果为0x0000000。

## mad

```assembly
:mad[_sat] dest[.mask], [-]src0[_abs][.swizzle], [-]src1[_abs][.swizzle], [-]src2[_abs][.swizzle]
```

乘加指令，算式为`dest = src0 * src1 + src2`。例如：

```assembly
// HLSL
tex2D( g_samScene, vTex0 ) * vColor + ColOffset;

// DXBC
texld r0, t0, s0
mad r0, r0, v0, c0
```

## max min imax imin

```assembly
max[_sat] dest[.mask], [-]src0[_abs][.swizzle], [-]src1[_abs][.swizzle],
```

max指令，算式为`dest = src0 >= src1 ? src0 : src1`。例如：

```assembly
// HLSL
Output.iClamped = max(min(iValue, 7), 1);
Output.fClamped = max(min(fValue, 8.1), 0.1);

// DXBC
imin r0.x, cb0[0].x, l(7)
imax o1.x, r0.x, l(1)
min r0.x, cb0[0].y, l(8.100000)
max o2.x, r0.x, l(0.100000)
```

## mov

```assembly
mov[_sat] dest[.mask], [-]src0[_abs][.swizzle]
```

赋值指令，等效于`dest = src0`。例如：

```
// HLSL
float x = 1.0;

// DXBC
mov r0.x, l(1.00000)
```

## movc 

```assembly
movc[_sat] dest[.mask], src0[.swizzle], [-]src1[_abs][.swizzle], [-]src2[_abs][.swizzle],
```

按条件赋值，等效于`If src0, then dest = src1 else dest = src2`。例如：

```assembly
// r0.xxxx不为0，就用l(0,0,0,0)，否者就用l(1.000000,1.000000,1.000000,1.000000)
movc o0.xyzw, r0.xxxx, l(0,0,0,0), l(1.000000,1.000000,1.000000,1.000000)
```

## mul imul umul

```assembly
mul[_sat] dest[.mask], [-]src0[_abs][.swizzle], [-]src1[_abs][.swizzle]
```

乘法指令，算式为`dest = src0 * src1`。例如：

```assembly
// HLSL
Output.Position * float4(2, 1, 0.5, 1)

// DXBC
mul r0.xy, r0, c0
```

## sqrt

```assembly
sqrt[_sat] dest[.mask], [-]src0[_abs][.swizzle]
```

开平方根，算式为`dest = sqrt(src0)`。例如：

```assembly
// HLSL
length(a.x)

// DXBC
sqrt r0.x, r0.x
```

## rsq

```assembly
rsq[_sat] dest[.mask], [-]src0[_abs][.swizzle]
```

等效于`dest = 1.0f / sqrt(src0)`。例如：

```assembly
// HLSL
float3 faceNormal = normalize(normal);

// DXBC
dp3 r0.w, r0.xyzx, r0.xyzx
rsq r0.w, r0.w
mul r0.xyz, r0.wwww, r0.xyzx
```

## sample sample_l sample_b sample_c sample_d

```assembly
sample[_aoffimmi(u,v,w)] dest[.mask], srcAddress[.swizzle], srcResource[.swizzle], srcSampler
```

采样指令。例如：

```assembly
// HLSL
float4 TopLeftTex = tex2D(Texture, In.Tex0.xz);

// DXBC
dcl_sampler s0, mode_default
dcl_resource_texture2d (float,float,float,float) t0
sample r0.xyzw, r0.xyxx, t0.xyzw, s0
```

# 示例

了解了上述指令之后，尝试来人肉逆向一些简单的Shader。

## 示例1

```assembly
// 声明：Pixel Shader，Shader Model 4.0
ps_4_0
// 声明一个默认的采样器：SamplerState TextureSampler
dcl_sampler s0, mode_default
// 声明一个Texture2D类型的贴图
dcl_resource_texture2d (float,float,float,float) t0
// 声明输出寄存器o0
dcl_output o0.xyzw
// 声明只有一个临时寄存器
dcl_temps 1
// 对贴图采样，UV为(0.5, 0.5)，采样器为s0，结果存储到r0临时寄存器。
sample r0.xyzw, l(0.500000, 0.500000, 0.000000, 0.000000), t0.xyzw, s0
// 将采样出来的R通道结果存储到o0.xyzw
mov o0.xyzw, r0.xxxx
ret 
```

逆向出来的HLSL代码如下：
```hlsl
Texture2D Texture0;
SamplerState TextureSampler
{
    Filter = MIN_MAG_MIP_LINEAR;
    AddressU = Wrap;
    AddressV = Wrap;
};
float4 main(void) : SV_Target
{
    return Texture0.Sample(TextureSampler, float2(0.5f, 0.5f) ).rrrr;
}
```

## 示例2

```hlsl
struct PS_INPUT
{
    uint PrimID : SV_PrimitiveID;
};

float4 main( PS_INPUT input ) : SV_Target
{
    float4 colour;
    float fPrimID = input.PrimID;
    colour.r = fPrimID;
    colour.g = 2 * fPrimID;
    colour.b = (1.0/64.0) * fPrimID;
    colour.a = (1.0/32.0) * fPrimID;
    return colour;
}
```

人肉导出一个DXBC指令出来：

```assembly
ps_4_0
dcl_input_ps_sgv v0.x, primitive_id
dcl_output o0.xyzw
dcl_temps 1
utof r0.x, v0.x
mul o0.yzw, r0.xxxx, l(0.000000, 2.000000, 0.015625, 0.031250)
mov o0.x, r0.x
ret 
```


