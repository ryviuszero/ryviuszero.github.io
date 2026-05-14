---
layout: post
title: Perlin Noise - Principles and Implementation
date:   2025-11-07 16:34:01 +0800
lang: en
permalink: /en/posts/perlin-noise/
translation_key: perlin-noise
---

> "There is no true randomness in nature, only patterns we haven't understood yet." — Philosophy of Perlin Noise

---

## Table of Contents

1. [What is Perlin Noise?](#1-what-is-perlin-noise)
2. [Why Do We Need Perlin Noise?](#2-why-do-we-need-perlin-noise)
3. [Core Principles](#3-core-principles)
4. [1D Perlin Noise Explained](#4-1d-perlin-noise-explained)
5. [2D Perlin Noise Explained](#5-2d-perlin-noise-explained)
6. [Fractal Noise (Fractal Noise / fBm)](#6-fractal-noise-fractal-noise--fbm)
7. [Simplex Noise](#7-simplex-noise)
8. [Code Implementation](#8-code-implementation)
9. [Common Applications](#9-common-applications)
10. [Parameter Tuning Tips](#10-parameter-tuning-tips)
11. [Common Misconceptions](#11-common-misconceptions)
12. [References](#12-references)

---

## 1. What is Perlin Noise?

**Perlin Noise** is a **gradient noise algorithm** developed by Ken Perlin in 1983 for the film *Tron*, and formally published in 1985 in the SIGGRAPH paper *An Image Synthesizer*. Perlin received an Academy Award for this contribution.

It is a type of **coherent noise** with these characteristics:

- **Smooth transitions** between neighboring sample points (no sudden jumps)
- **Statistically isotropic** (no preferred direction)
- Output values in a defined range (typically `-1` to `1`, or normalized to `0` to `1`)
- **Deterministic**: same input always produces same output

---

## 2. Why Do We Need Perlin Noise?

### The Problem with Pure Randomness

```
Pure Random (White Noise):  9 4 1 7 3 8 2 5 6 ...
Perlin Noise:              3 4 5 6 5 4 3 2 3 ...
```

Pure random noise (white noise) treats each pixel independently, creating a "TV static" effect that **almost never appears in nature**.

Real natural phenomena (mountains, clouds, flames, water surfaces) exhibit **local coherence**: nearby locations have similar values, but with overall macroscopic variation. Perlin Noise mimics exactly this property.

### Comparison

| Property | White Noise | Perlin Noise |
|----------|-----------|-------------|
| Smooth neighbors | ❌ | ✅ |
| Natural looking | ❌ | ✅ |
| Controllability | Low | High |
| Computational complexity | O(1) | O(2ⁿ), n = dimensions |

---

## 3. Core Principles

Perlin Noise follows three core steps:

```
① Create grid, assign random gradient vectors to each grid point
          ↓
② Compute distance vectors from sample point to surrounding points
       Dot product with gradient vectors
          ↓
③ Use smooth interpolation to blend contributions from surrounding points
```

### Key Concept: Gradient Vectors

At each integer coordinate (grid point), pre-assign a **random unit direction vector**—this is the "gradient."

Unlike Value Noise (storing random values at points), Perlin Noise stores **directions, not values**. This produces smoother, more natural output.

### Key Concept: Smooth Interpolation Curve

Simple linear interpolation (`lerp`) produces linear artifacts. Perlin uses a **quintic smoothing curve**:

$$f(t) = 6t^5 - 15t^4 + 10t^3$$

Properties:
- $f(0) = 0$, $f(1) = 1$
- $f'(0) = 0$, $f'(1) = 0$ (zero derivatives at endpoints for smooth joining)
- $f''(0) = 0$, $f''(1) = 0$ (zero second derivatives for extra smoothness)

> Early versions used cubic curve $3t^2 - 2t^3$ (Smoothstep); improved versions use quintic (Smootherstep).

---

## 4. 1D Perlin Noise Explained

For a 1D sample point $x = 2.3$:

### Step 1: Find the Grid Cell

```
Left point:  x0 = floor(2.3) = 2
Right point: x1 = x0 + 1 = 3
Local coordinate: t = 2.3 - 2 = 0.3
```

### Step 2: Get Gradients at Endpoints

In 1D, gradients are just two directions: `+1` or `-1`, determined via **permutation table**:

```
grad(2) = +1
grad(3) = -1
```

### Step 3: Calculate Dot Products

Distance vector × gradient vector (multiplication in 1D):

```
Left contribution: dot(grad(x0), x - x0) = (+1) × (2.3 - 2) = +0.3
Right contribution: dot(grad(x1), x - x1) = (-1) × (2.3 - 3) = +0.7
```

### Step 4: Smooth Interpolation

```
u = fade(t) = fade(0.3) ≈ 0.163  (quintic curve)
result = lerp(u, 0.3, 0.7) = 0.3 + 0.163 × (0.7 - 0.3) ≈ 0.365
```

---

## 5. 2D Perlin Noise Explained

In 2D, sample point $(x, y)$ is surrounded by four grid points:

```
(x0,y1) -------- (x1,y1)
   |         .        |
   |      (x,y)       |
   |                  |
(x0,y0) -------- (x1,y0)
```

### Gradient Vectors (2D)

2D gradients typically chosen from 4 (or 8) vectors:

```
(1,0), (-1,0), (0,1), (0,-1)
(1,1), (-1,1), (1,-1), (-1,-1)  ← normalized version
```

### Computation Flow

```
① Calculate distance vectors from sample to 4 corners
② Dot each with corner gradient → 4 values
③ Interpolate along x using fade(tx), then along y with fade(ty)
```

```python
# Pseudocode
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

## 6. Fractal Noise (Fractal Noise / fBm)

Single-layer Perlin Noise produces overly "smooth and monotonous" terrain lacking detail. Natural landscapes (mountains, clouds) are **multi-scale layered** phenomena.

### fBm (Fractional Brownian Motion)

Layer multiple noise octaves with different frequencies and amplitudes:

$$\text{fBm}(x) = \sum_{i=0}^{n} \text{amplitude}_i \cdot \text{noise}(\text{frequency}_i \cdot x)$$

### Key Parameters

| Parameter | Meaning | Typical Value |
|-----------|---------|---------------|
| **Octaves** | Number of layers; more = richer detail | 4 ~ 8 |
| **Lacunarity** | Frequency multiplier per layer | 2.0 |
| **Persistence / Gain** | Amplitude reduction per layer | 0.5 |

### Example (4 Layers)

```
Layer 0: frequency=1,  amplitude=1.0   → Large terrain outline
Layer 1: frequency=2,  amplitude=0.5   → Mid-scale hills
Layer 2: frequency=4,  amplitude=0.25  → Small-scale rocks
Layer 3: frequency=8,  amplitude=0.125 → Fine texture
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

Ken Perlin proposed **Simplex Noise** in 2001 as an improved version.

### Improvements

| Property | Perlin Noise | Simplex Noise |
|----------|------------|--------------|
| Interpolation corners | 2ⁿ corners | n+1 vertices |
| Complexity (high-dim) | O(2ⁿ) | O(n²) |
| Directional artifacts | Yes (axis-aligned) | Less |
| Implementation | Simpler | Complex |
| Patent issues | None | 3D+ patented (use OpenSimplex) |

For high-dimensional scenarios (3D, 4D), **Simplex Noise offers dramatic performance gains**.

> For patent-free implementation, use **OpenSimplex2** or **SuperSimplex**.

---

## 8. Code Implementation

### Python Implementation (Simplified 2D Perlin Noise)

```python
import math
import random

def fade(t):
    """Quintic smoothing curve"""
    return t * t * t * (t * (t * 6 - 15) + 10)

def lerp(t, a, b):
    """Linear interpolation"""
    return a + t * (b - a)

def grad(hash_val, x, y):
    """Select gradient direction and compute dot product"""
    h = hash_val & 3
    if h == 0: return  x + y
    if h == 1: return -x + y
    if h == 2: return  x - y
    return              -x - y

class PerlinNoise2D:
    def __init__(self, seed=None):
        random.seed(seed)
        # Build permutation table
        self.perm = list(range(256))
        random.shuffle(self.perm)
        self.perm += self.perm  # Double for easier indexing

    def noise(self, x, y):
        # Grid coordinates
        xi = int(math.floor(x)) & 255
        yi = int(math.floor(y)) & 255
        # Local coordinates
        xf = x - math.floor(x)
        yf = y - math.floor(y)
        # Smooth
        u = fade(xf)
        v = fade(yf)
        # Hash values for 4 corners
        p = self.perm
        aa = p[p[xi    ] + yi    ]
        ab = p[p[xi    ] + yi + 1]
        ba = p[p[xi + 1] + yi    ]
        bb = p[p[xi + 1] + yi + 1]
        # Interpolate
        x1 = lerp(u, grad(aa, xf,   yf  ), grad(ba, xf-1, yf  ))
        x2 = lerp(u, grad(ab, xf,   yf-1), grad(bb, xf-1, yf-1))
        return lerp(v, x1, x2)

# Usage
pn = PerlinNoise2D(seed=42)
for y in range(5):
    row = [pn.noise(x * 0.1, y * 0.1) for x in range(10)]
    print([f"{v:.2f}" for v in row])
```

### GLSL Shader Implementation (2D)

```glsl
// Hash function (approximation)
float hash(vec2 p) {
    p = fract(p * vec2(127.1, 311.7));
    p += dot(p, p + 19.19);
    return fract(p.x * p.y);
}

// Gradient vectors
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

## 9. Common Applications

### Terrain Generation

```
Height map = fBm(x, z)
Texture blend weight = noise(x, z)  → grass/dirt/rock
```

### Clouds and Sky

```
Cloud density = smoothstep(0.4, 0.6, fBm(x, y, t))
```

### Water Surface and Waves

```
Water height(x, z, t) = noise(x + t, z) + noise(x, z + t * 0.7) * 0.5
Surface normal = gradient of height field
```

### Procedural Textures

- **Marble**: `sin(x + noise(x, y) * 10)`
- **Wood grain**: `sin(sqrt(x² + y²) + noise(x, y) * 5)`
- **Fire effect**: Dynamic noise with color gradient

### Animation and Motion

- **Camera shake**: Low-frequency noise drives camera offset, simulating handheld effect
- **NPC pathfinding**: Noise generates natural wandering directions
- **Particle systems**: Noise field controls particle forces

---

## 10. Parameter Tuning Tips

### Frequency (Frequency / Scale)

```
Low frequency (large scale) → Slow variation, large-scale features (mountains, continents)
High frequency (small scale) → Fast variation, fine details (stone texture, grass)
```

### Amplitude

Controls noise value intensity, typically normalized:

```python
# Normalize to [0, 1]
normalized = (noise_value + 1) / 2
```

### Octave Selection

```
2 ~ 3 layers: Smooth, simple (clouds, water)
4 ~ 6 layers: Natural terrain (mountains, hills)
7 ~ 8 layers: Rich detail (rocks, soil)
```

> **Note:** More layers have diminishing visual returns but linear computation cost.

### Domain Warping

Advanced technique—warp coordinates using noise for highly natural flow effects:

```python
def warped_noise(x, y):
    # Use noise to offset input coordinates
    dx = fbm(x + 1.7, y + 9.2)
    dy = fbm(x + 8.3, y + 2.8)
    return fbm(x + 4.0 * dx, y + 4.0 * dy)
```

This is the core technique behind many Shadertoy works by Inigo Quilez.

---

## 11. Common Misconceptions

### Misconception 1: Perlin Noise is Random

Perlin Noise is **not random**. Same input always returns same output. It's **deterministic pseudo-randomness**. True randomness comes from shuffling the permutation table at initialization (`seed`).

### Misconception 2: Output Range is [-1, 1]

Theoretical range is $[-\sqrt{n/4}, \sqrt{n/4}]$ (n = dimensions), but actual implementations vary. **Don't assume output strictly constrained to any range**—normalize or clamp in practice.

### Misconception 3: Higher Frequency = Better Detail

High-frequency noise causes **aliasing**, producing flickering in dynamic scenes. Choose max frequency based on sampling rate (consult Nyquist theorem).

### Misconception 4: Using random() Directly

Many beginners regenerate random gradients on each call, breaking coherence. **Pre-build permutation tables**, query via lookup.

---

## 12. References

- Ken Perlin's improved version: [Improving Noise (SIGGRAPH 2002)](https://mrl.cs.nyu.edu/~perlin/paper445.pdf)
- Inigo Quilez on Domain Warping: [iquilezles.org/articles/warp](https://iquilezles.org/articles/warp/)
- The Book of Shaders (highly recommended): [thebookofshaders.com/11/](https://thebookofshaders.com/11/)
- Wikipedia - Perlin Noise: [en.wikipedia.org/wiki/Perlin_noise](https://en.wikipedia.org/wiki/Perlin_noise)
- OpenSimplex2 (patent-free alternative): [github.com/KdotJPG/OpenSimplex2](https://github.com/KdotJPG/OpenSimplex2)

---
