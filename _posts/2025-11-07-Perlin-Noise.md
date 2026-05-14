---
layout: post
title: Perlin Noise 原理与实现
date:   2025-11-07 16:34:01 +0800
lang: zh
permalink: /zh/posts/perlin-noise/
translation_key: perlin-noise
---

> "自然界中没有真正的随机，只有我们尚未理解的规律。" —— Perlin Noise 的哲学

---

## 目录

1. [什么是 Perlin Noise？](#1-什么是-perlin-noise)
2. [为什么需要 Perlin Noise？](#2-为什么需要-perlin-noise)
3. [核心原理](#3-核心原理)
4. [一维 Perlin Noise 详解](#4-一维-perlin-noise-详解)
5. [二维 Perlin Noise 详解](#5-二维-perlin-noise-详解)
6. [分形噪声（Fractal Noise / fBm）](#6-分形噪声fractal-noise--fbm)
7. [Simplex Noise](#7-simplex-noise)
8. [代码实现](#8-代码实现)
9. [常见应用场景](#9-常见应用场景)
10. [参数调节技巧](#10-参数调节技巧)
11. [常见误区](#11-常见误区)
12. [参考资料](#12-参考资料)

---

## 1. 什么是 Perlin Noise？

**Perlin Noise**（柏林噪声）是由 Ken Perlin 在 1983 年为电影《创：战纪》（Tron）开发的一种**梯度噪声算法**，并于 1985 年在 SIGGRAPH 论文 *An Image Synthesizer* 中正式发表。Ken Perlin 也因此获得了奥斯卡科学技术奖。

它是一种**相干噪声（Coherent Noise）**，具有以下特点：

- 相邻采样点之间**平滑过渡**，不会突变
- 在统计上是**各向同性**的（没有明显方向偏好）
- 输出值在一定范围内（通常 `-1` 到 `1`，或归一化到 `0` 到 `1`）
- **可重复**：相同输入总是产生相同输出

---

## 2. 为什么需要 Perlin Noise？

### 纯随机的问题

```
纯随机（White Noise）：9 4 1 7 3 8 2 5 6 ...
Perlin Noise：        3 4 5 6 5 4 3 2 3 ...
```

纯随机噪声（白噪声）每个像素完全独立，产生的是"电视雪花"效果，**在自然界中几乎不存在**。

真实的自然现象（山脉、云朵、火焰、水面）都具有**局部相似性**：靠近的地方高度/颜色差异不大，但整体上又有宏观的起伏变化。Perlin Noise 正是为了模拟这种特性而生。

### 对比总结

| 特性 | 白噪声 | Perlin Noise |
|------|--------|--------------|
| 相邻平滑 | ❌ | ✅ |
| 自然感 | ❌ | ✅ |
| 可控性 | 低 | 高 |
| 计算复杂度 | O(1) | O(2ⁿ)，n 为维度 |

---

## 3. 核心原理

Perlin Noise 的核心思路分三步：

```
① 建立网格，为每个格点分配随机梯度向量
        ↓
② 计算采样点到周围格点的距离向量，与梯度向量做点积
        ↓
③ 用平滑插值函数，将周围格点的点积值融合成最终输出
```

### 关键概念：梯度向量（Gradient Vector）

在每个整数坐标（格点）上，预先分配一个**随机的单位方向向量**，这就是"梯度"。

与 Value Noise（直接在格点上存随机值）不同，Perlin Noise 存的是**方向**，而不是**值**。这使得它的输出更加平滑、视觉上更自然。

### 关键概念：平滑插值函数

普通线性插值（`lerp`）会产生折线感。Perlin 使用了**五次平滑曲线（Quintic Curve）**：

$$f(t) = 6t^5 - 15t^4 + 10t^3$$

这个函数的特点：
- $f(0) = 0$，$f(1) = 1$
- $f'(0) = 0$，$f'(1) = 0$（端点处导数为 0，保证平滑衔接）
- $f''(0) = 0$，$f''(1) = 0$（二阶导也为 0，更加平滑）

> 早期版本使用的是三次曲线 $3t^2 - 2t^3$（Smoothstep），改进版使用五次曲线（Smootherstep）。

---

## 4. 一维 Perlin Noise 详解

以一维为例，假设采样点 $x = 2.3$：

### Step 1：找到所属格子

```
左格点：x0 = floor(2.3) = 2
右格点：x1 = x0 + 1 = 3
局部坐标：t = 2.3 - 2 = 0.3
```

### Step 2：获取两端梯度

一维中，梯度只有两个方向：`+1` 或 `-1`，通过**置换表（Permutation Table）**伪随机确定：

```
grad(2) = +1
grad(3) = -1
```

### Step 3：计算点积

距离向量 × 梯度向量（一维中即乘法）：

```
左端点贡献：dot(grad(x0), x - x0) = (+1) × (2.3 - 2) = +0.3
右端点贡献：dot(grad(x1), x - x1) = (-1) × (2.3 - 3) = +0.7
```

### Step 4：平滑插值

```
u = fade(t) = fade(0.3) ≈ 0.163  （五次曲线）
result = lerp(u, 0.3, 0.7) = 0.3 + 0.163 × (0.7 - 0.3) ≈ 0.365
```

---

## 5. 二维 Perlin Noise 详解

二维时，采样点 $(x, y)$ 被四个格点围绕：

```
(x0,y1) -------- (x1,y1)
   |         .        |
   |      (x,y)       |
   |                  |
(x0,y0) -------- (x1,y0)
```

### 梯度向量（2D）

二维梯度通常从以下 4 个（或 8 个）向量中随机选取：

```
(1,0), (-1,0), (0,1), (0,-1)
(1,1), (-1,1), (1,-1), (-1,-1)  ← 归一化后使用
```

### 计算流程

```
① 计算采样点到 4 个角的距离向量
② 分别与各角的梯度做点积，得到 4 个值
③ 用 fade(tx) 和 fade(ty) 先沿 x 插值，再沿 y 插值
```

```python
# 伪代码
a = dot(grad(x0, y0), dx,   dy  )
b = dot(grad(x1, y0), dx-1, dy  )
c = dot(grad(x0, y1), dx,   dy-1)
d = dot(grad(x1, y1), dx-1, dy-1)

u = fade(dx)
v = fade(dy)

ab = lerp(u, a, b)
cd = lerp(u, c, d)
result = lerp(v, ab, cd)
```

---

## 6. 分形噪声（Fractal Noise / fBm）

单层 Perlin Noise 产生的地形过于"平滑单调"，缺乏细节。自然界的地貌（山脉、云朵）是**多尺度叠加**的结果。

### fBm（Fractional Brownian Motion）

通过叠加多个频率和振幅不同的 Noise 层（**倍频程，Octaves**）：

$$\text{fBm}(x) = \sum_{i=0}^{n} \text{amplitude}_i \cdot \text{noise}(\text{frequency}_i \cdot x)$$

### 关键参数

| 参数 | 说明 | 典型值 |
|------|------|--------|
| **Octaves（倍频程数）** | 叠加的层数，越多细节越丰富 | 4 ~ 8 |
| **Lacunarity（缺陷度）** | 每层频率的倍增系数 | 2.0 |
| **Persistence / Gain** | 每层振幅的衰减系数 | 0.5 |

### 示例（4 层叠加）

```
Layer 0: frequency=1,  amplitude=1.0   → 大尺度地形轮廓
Layer 1: frequency=2,  amplitude=0.5   → 中尺度丘陵
Layer 2: frequency=4,  amplitude=0.25  → 小尺度岩石
Layer 3: frequency=8,  amplitude=0.125 → 细节纹理
```

```python
def fbm(x, y, octaves=6, lacunarity=2.0, gain=0.5):
    value = 0.0
    amplitude = 1.0
    frequency = 1.0
    for _ in range(octaves):
        value += amplitude * perlin(x * frequency, y * frequency)
        amplitude *= gain
        frequency *= lacunarity
    return value
```

---

## 7. Simplex Noise

Ken Perlin 于 2001 年提出了 **Simplex Noise**，作为 Perlin Noise 的改进版本。

### 改进点

| 特性 | Perlin Noise | Simplex Noise |
|------|-------------|---------------|
| 插值方向数 | 2ⁿ 个角 | n+1 个顶点 |
| 计算复杂度（高维）| O(2ⁿ) | O(n²) |
| 方向性伪影 | 有（轴对齐） | 较少 |
| 实现复杂度 | 较简单 | 较复杂 |
| 专利问题 | 无 | 3D+ 版本有专利（OpenSimplex 可替代） |

在高维（3D、4D）场景中，**Simplex Noise 的性能优势非常显著**。

> 如果需要无专利约束的实现，可以使用 **OpenSimplex2** 或 **SuperSimplex**。

---

## 8. 代码实现

### Python 实现（简化版 2D Perlin Noise）

```python
import math
import random

def fade(t):
    """五次平滑曲线"""
    return t * t * t * (t * (t * 6 - 15) + 10)

def lerp(t, a, b):
    """线性插值"""
    return a + t * (b - a)

def grad(hash_val, x, y):
    """根据 hash 值选择梯度方向并计算点积"""
    h = hash_val & 3
    if h == 0: return  x + y
    if h == 1: return -x + y
    if h == 2: return  x - y
    return              -x - y

class PerlinNoise2D:
    def __init__(self, seed=None):
        random.seed(seed)
        # 构建置换表
        self.perm = list(range(256))
        random.shuffle(self.perm)
        self.perm += self.perm  # 复制一份，方便索引

    def noise(self, x, y):
        # 格点坐标
        xi = int(math.floor(x)) & 255
        yi = int(math.floor(y)) & 255
        # 局部坐标
        xf = x - math.floor(x)
        yf = y - math.floor(y)
        # 平滑
        u = fade(xf)
        v = fade(yf)
        # 四个角的 hash
        p = self.perm
        aa = p[p[xi    ] + yi    ]
        ab = p[p[xi    ] + yi + 1]
        ba = p[p[xi + 1] + yi    ]
        bb = p[p[xi + 1] + yi + 1]
        # 插值
        x1 = lerp(u, grad(aa, xf,   yf  ), grad(ba, xf-1, yf  ))
        x2 = lerp(u, grad(ab, xf,   yf-1), grad(bb, xf-1, yf-1))
        return lerp(v, x1, x2)

# 使用示例
pn = PerlinNoise2D(seed=42)
for y in range(5):
    row = [pn.noise(x * 0.1, y * 0.1) for x in range(10)]
    print([f"{v:.2f}" for v in row])
```

### GLSL Shader 实现（2D）

```glsl
// 置换函数（近似）
float hash(vec2 p) {
    p = fract(p * vec2(127.1, 311.7));
    p += dot(p, p + 19.19);
    return fract(p.x * p.y);
}

// 梯度向量
vec2 gradient(vec2 p) {
    float h = hash(p);
    float angle = h * 6.2832; // 0 ~ 2π
    return vec2(cos(angle), sin(angle));
}

// 2D Perlin Noise
float perlin(vec2 p) {
    vec2 i = floor(p);
    vec2 f = fract(p);
    vec2 u = f * f * f * (f * (f * 6.0 - 15.0) + 10.0); // fade

    float a = dot(gradient(i + vec2(0,0)), f - vec2(0,0));
    float b = dot(gradient(i + vec2(1,0)), f - vec2(1,0));
    float c = dot(gradient(i + vec2(0,1)), f - vec2(0,1));
    float d = dot(gradient(i + vec2(1,1)), f - vec2(1,1));

    return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
}
```

---

## 9. 常见应用场景

### 地形生成

```
高度图 = fBm(x, z)
纹理混合权重 = noise(x, z)  → 草地/泥土/岩石
```

### 云朵与天空

```
云密度 = smoothstep(0.4, 0.6, fBm(x, y, t))
```

### 水面与波浪

```
水面高度(x, z, t) = noise(x + t, z) + noise(x, z + t * 0.7) * 0.5
法线方向 = gradient of height field
```

### 程序化纹理

- 大理石纹理：`sin(x + noise(x, y) * 10)`
- 木纹纹理：`sin(sqrt(x² + y²) + noise(x, y) * 5)`
- 火焰效果：动态 noise 配合颜色渐变

### 动画与运动

- **相机抖动**：用低频 noise 驱动相机偏移，模拟手持效果
- **NPC 行走路径**：noise 生成自然的漫游方向
- **粒子系统**：用 noise 场控制粒子受力方向

---

## 10. 参数调节技巧

### 频率（Frequency / Scale）

```
频率低（Scale 大）→ 变化缓慢，大尺度特征（山脉、大陆）
频率高（Scale 小）→ 变化快速，细节特征（石头纹理、草地）
```

### 振幅（Amplitude）

控制 noise 值的强度范围，通常配合归一化：

```python
# 归一化到 [0, 1]
normalized = (noise_value + 1) / 2
```

### Octaves 的选择

```
2 ~ 3 层：平滑、简洁（适合云朵、水面）
4 ~ 6 层：自然地形（适合山脉、丘陵）
7 ~ 8 层：极丰富细节（适合岩石、土壤）
```

> 注意：层数过多在视觉上收益递减，但计算量线性增加。

### 域变形（Domain Warping）

一种高级技巧，用 noise 对坐标本身进行扭曲，产生极度自然的流动感：

```python
def warped_noise(x, y):
    # 用 noise 偏移输入坐标
    dx = fbm(x + 1.7, y + 9.2)
    dy = fbm(x + 8.3, y + 2.8)
    return fbm(x + 4.0 * dx, y + 4.0 * dy)
```

这正是 Inigo Quilez 等人创作的许多 Shadertoy 作品的核心技巧。

---

## 11. 常见误区

### 误区 1：Perlin Noise 是随机的

Perlin Noise **不是随机的**。相同的输入坐标永远返回相同的值。它是**确定性的伪随机过程**。真正的随机来自初始化时打乱的置换表（`seed`）。

### 误区 2：输出范围是 [-1, 1]

理论范围是 $[-\sqrt{n/4}, \sqrt{n/4}]$（n 为维度），实际实现中因梯度选取不同而略有差异。在实践中，**不要假设输出严格限制在某个范围内**，应对输出进行归一化或 clamp 处理。

### 误区 3：频率越高细节越好

高频 noise 容易产生**走样（Aliasing）**，在动态场景中会出现闪烁。应根据采样率选择合适的最高频率（参考奈奎斯特定理）。

### 误区 4：用 `random()` 直接生成梯度

不少初学者在每次调用时重新生成随机梯度，导致输出不连贯或不可重复。正确做法是**预先构建置换表**，通过查表确定梯度。

---

## 12. 参考资料


- Ken Perlin 改进版：[Improving Noise (SIGGRAPH 2002)](https://mrl.cs.nyu.edu/~perlin/paper445.pdf)
- Inigo Quilez 的 Domain Warping：[iquilezles.org/articles/warp](https://iquilezles.org/articles/warp/)
- The Book of Shaders（强烈推荐）：[thebookofshaders.com/11/](https://thebookofshaders.com/11/)
- Wikipedia - Perlin Noise：[en.wikipedia.org/wiki/Perlin_noise](https://en.wikipedia.org/wiki/Perlin_noise)
- OpenSimplex2（无专利替代方案）：[github.com/KdotJPG/OpenSimplex2](https://github.com/KdotJPG/OpenSimplex2)

---
