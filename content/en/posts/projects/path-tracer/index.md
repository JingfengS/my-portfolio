+++
date = '2026-01-13T18:22:25+09:00'
draft = false
title = 'Path Tracer: Physically Based Rendering'
showTableOfContents = true
tags = ['Graphics', 'Rendering', 'C++', 'Path Tracing', 'Acceleration Structures']
summary = "Berkeley CS184 - A physically-based renderer in C++, featuring BVH acceleration with SAH and deep performance profiling."
description = "Implementing a core rendering engine from scratch: from ray-scene intersection to global illumination. Highlights include Surface Area Heuristic (SAH) for BVH construction and micro-benchmarking performance bottlenecks."
+++

{{< katex >}}

## Project Overview

<div style="width: 80%; margin: 0 auto;">
    {{< figure src="cornell.png" caption="**Fig 1:** Direct Illumination vs Global Illumination (Low Sampling rate) vs Global Illumination (High Sampling rate)" alt="Result 1" >}}
</div>

This project involves the implementation of a physically-based **Path Tracer** as part of the Berkeley CS184 (Spring 2025) curriculum. The goal was to build a rendering engine from scratch capable of simulating the complex behavior of light using Monte Carlo integration.

Beyond the core requirements, I focused heavily on **acceleration structure efficiency** by implementing a Surface Area Heuristic (SAH) for BVH construction and conducted **deep performance profiling** to optimize the C++ rendering pipeline.

**Core Technical Features**

- **Ray-Scene Intersection Engine:** Implemented highly optimized intersection primitives for triangles (using the Möller-Trumbore algorithm) and spheres, serving as the foundation for the entire visibility system.

- **BVH Acceleration Structure:** Utilized a **Surface Area Heuristic (SAH)** to construct an optimized BVH for efficient ray tracing. This structure allowed for rapid intersection tests and improved performance by reducing the number of intersection tests required.

- **Direct Illumination:** Implemented both Uniform Hemisphere Sampling and Light Source Importance Sampling. The latter significantly reduces variance (noise) by focusing samples on emissive surfaces.

- **Global Illumination:** Implemented a full path tracing loop that accounts for indirect lighting. I used **Russian Roulette** as a stochastic termination criterion, ensuring the estimator remains unbiased while keeping the computation efficient.

- **Profiling with `tray`/Tracy:** Used instrumentation and sampling profilers to visualize thread utilization and identify bottlenecks.

- **Multithreading:** Leveraged a thread-pool architecture to parallelize the rendering process across image tiles, ensuring near-linear speedup on multi-core processors.

- **Adaptive Sampling:** Integrated a statistical confidence-based sampling method. By tracking the mean and variance of pixel radiance, the renderer dynamically stops sampling converged pixels, focusing computational power on high-variance (noisy) regions.

## Part 1: Ray Generation and Scene Intersection

### 1.1 Ray Generation Pipline

The ray generation process bridges the 2D image space and the 3D world space. The goal is to take a normalized image coordinate \( (x, y) \)--where \((0, 0)\) is the bottom-left and \((1, 1)\) is the top-right--and transform it into a ray originating from the camera and shooting into the scene.

My implementation in `Camera::generate_ray` follows these steps:

1. **Camera Space Transformation:** We define a virtual sensor place at \(Z = -1\) in camera space. The size of this sensor is determined by the horizontal (_hFov_) and vertical (_vFov_) fields of view. I calculated the sensor dimensions using the tangent of half the FOV angles.

2. **Coordinate Mapping:** Since the input coordinated \((x, y)\) are normalized to \([0, 1]\), I mapped them to the sensor's coordinate system, which ensures he center of the image \((0.5, 0.5)\) aligns with the camera's optical axis \((0, 0, -1)\).

```cpp
// Transform normalized coordinates to sensor space
double sensor_x = (2 * x - 1) * tan(radians(hFov) / 2);
double sensor_y = (2 * y - 1) * tan(radians(vFov) / 2);
```

3. **World Space Transformation:** The ray's direction vector in camera space is \((sensor_x, sensor_y, -1)\). I then multiplied this vector by the camera-to-world retation matrix(`c2w`) to orient the ray correctly in the world.

4. **Ray Creation:** Finally, the ray is created with its origin at the camera's position and the calculated normalized direction. I also initialized the ray's `min_t` and `max_t` using the near and far clipping planes.

To perform supersampling (antialiasing), the raytrace_pixel function calls `generate_ray` multiple times, it uses a `gridSampler` to generate random offsets \((\Delta x, \Delta y)\) to sample the pixel, averaging the radiance results to produce a smoother image.

### 1.2 Intersection

**Triangle Intersection:** For ray-triangle intersection, I implemented **Möller-Trumbore algorithm**, which solves for the barycentric coordinates \((1-b_1-b_2, b_1, b_2)\) and the ray parameter \(t\) simultaneously using Cramer's rule.

An intersection is valid only if:

- The time \(t\) is within the range \([min_t, max_t]\)
- The barycentric coordinates \((1-b_1-b_2, b_1, b_2)\) are within the range \([0, 1]\)

**Sphere Intersection:** I used the analytic geometric approach. I substituted the ray equation \(P(t) = O + tD\) into the sphere equation \((P - C)^2 - R^2 = 0\), resulting in a quadratic equation \(At^2 + Bt + C = 0\). I solved for \(t\) using the quadratic formula. If the discriminant is non-negative, the ray intersects the sphere, and I selected the closest valid \(t\) within the clipping planes.

### 1.3 Results: Normal Shading
Below are images of several simple `.dae` files rendered with normal shading to verify the implementation.

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="CBempty.png" caption="**Fig 1:** Empty Cornell Box" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="CBspheres.png" caption="**Fig 2:** Cornell Box with Spheres" alt="Result 2" >}}
    </div>
    <div class="flex-1">
        {{< figure src="banana.png" caption="**Fig 3:** Banana" alt="Result 3" >}}
    </div>
</div>