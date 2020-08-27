---
title: LinearizeDepth
date: 2020-08-27 22:15:19
tags:
- Vulkan
- MSAA
- 3D
- Demo
categories:
- Vulkan
---

```c++

n   = near
f   = far
z   = depth buffer Z-value
EZ  = eye Z value
LZ  = depth buffer Z-value remapped to a linear [0..1] range (near plane to far plane)
LZ2 = depth buffer Z-value remapped to a linear [0..1] range (eye to far plane)

DX:
EZ  = (n * f) / (f - z * (f - n))
LZ  = (eyeZ - n) / (f - n) = z / (f - z * (f - n))
LZ2 = eyeZ / f = n / (f - z * (f - n))


GL:
EZ  = (2 * n * f) / (f + n - z * (f - n))
LZ  = (eyeZ - n) / (f - n) = n * (z + 1.0) / (f + n - z * (f - n))
LZ2 = eyeZ / f = (2 * n) / (f + n - z * (f - n))

LZ2 = 1.0 / (c0 * z + c1)

DX:
  c1 = f / n
  c0 = 1.0 - c1

GL:
  c0 = (1 - f / n) / 2
  c1 = (1 + f / n) / 2

```