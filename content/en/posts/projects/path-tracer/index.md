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

## Part 2: Bounding Volume Hierarchy (BVH)

### 2.1 BVH Construction Algorithm

My BVH construction algorithm is a recursive process (implemented using std::stack and loop) that organizes scene primitives into to a binary tree structure for efficient ray intersection. The core function, `construct_bvh`, takes a list of primitives and recursively divides them until a leaf node is reached.

The detailed steps are as follows:

1. **Bounding Box Calculation:** First, I compute the bounding box of the current set of primitives. If the number of primitives is small enough (less than or equal to `max_leaf_size`), I immediately create a leaf node and return.

2. **Heuristic Selection (SAH):** I employ the **Surface Area Heuristic (SAH)** to determine the optimal splitting plane. I calculate the bounding box of all primitive centroids and choose the axis with the largest extent as the splitting axis.

3. **Binning Strategy:** To avoid expensive \(O(N^2)\) cost of testing every possible split, I implemented the "Binning" approximation. I divide the chosen axis into **16 uniform bins** and then iterate through all primitives, calculate which bin their centroid falls into, and update the bounding box and count for each bin.

4. **Cost Evaluation:** I evaluate the SAH cost for the 15 possible split plances between these buckets. The SAH cost function is:
   $$ C = C_{trav} + \frac{S_A}{S_{total}} N_A C_{isect} + \frac{S_B}{S_{total}} N_B C_{isect} $$
   where \(S\) represents surface area and \(N\) represents number of primitives count. I use efficient forward and backward scans (prefix/suffix sums) to compute the surface areas and counts for the left and right partitions in linear time.

5. **Partitioning & Recursion:** After finding the split index with the minimum cost, I check if splitting is actually compared to creating a leaf. If it is, I use `std::partition` with a lambda function to reorder the primitives in-place: those falling into buckets left of the split point move to the front, and the rest move to the back. Finally, I recursively construct the left and right children of the current node.

### 2.2 Results: Rendering Large Models

With BVH acceleration, I can now render complex geometries with tens of thousands of triangles, which would have been impossible (or infinitely slow) with the naive implementation. Blow are images of complex `.dae` files rendered with normal shading.

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="maxplanck.png" caption="**Fig 1:** maxplanck.dae (Complex Mesh)" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="CBlucy.png" caption="**Fig 2:** CBlucy.dae (High Poly Count)" alt="Result 2" >}}
    </div>
</div>

### 2.3 Performance Analysis

I compared the rendering performance of the naive and BVH-accelerated implementations using model of different complex level. The tests were run on a single thread on Macbook Pro with an M2 Chip.

| Scene            | Triangle Count | Render Time (No BVH) | Render Time (With BVH) |
| :--------------- | :------------- | :------------------- | :--------------------- |
| **cow.dae**      | 5,856          | 32.48 s              | 0.0831 s               |
| **beetle.dae**   | 7,512          | 38.96 s              | 0.0728 s               |
| **CBdragon.dae** | 105,120        | > 1 hour             | 0.0899 s               |

The results demonstrate a drastic improvement in rendering
speed. Without BVH, the intersection complexity is \(O(N)\) per
ray. For models like the Cow or Beetle, this is noticeably
slow. With BVH, the complexity drops to \(O(\log N)\) as rays
only traverse relevant nodes of the tree.
<br>
On the other hand, I found that in BVH, the render time does not
increase significantly with the number of triangles, showing
that **the bottle neck has shifted from CPU computing
power to memory bandwidth and cache efficiency**,
which is a direction that can be further optimized in future
work.

## Part 3: Direct Illumination

### 3.1 Walkthrough of Direct Lighting Implementations

In this part, I implemented two distinct algorithms for estimating direct illumination: **Uniform Hemisphere Sampling** and **Importance Light Sampling**.

**Uniform Hemisphere Sampling:**

In the `estimate_direct_lighting_hemisphere` function, I estimate the radiance at an intersection point by uniformly sampling directions over the heimisphere:

- **Sampling Process:** I used the `hemisphereSampler` to generate a random direction in object space, and then transformed it to world space using the `o2w` matrix constructed from the surface normal.

- **Ray Casting:** A new ray is cast from the hit point in the sampled direction. To prevent self-intersection, the ray's `min_t` is set to `EPS_D`.

- **Radiance Accumulation:** If the ray hits a light source, its contribution is added. This depends on the light's radiance, the surface's BSDF evaluation, and the cosine of the angle between the sample direction and the normal.

- **Monte Carlo Integration:** The final result is the average of all samples, scaled by the inverse of the PDF for uniform hemisphere sampling, which is \(2\pi\).

**Importance Light Sampling:**

In the `estimate_direct_lighting_importance` function, I sample directions directly toward the light sources, which significantly reduces variance and noise.

- **Iterating lights:** The function iterates through all light sources in `scene->lights`. For delta lights (like point lights), only one sample is taken; for area lights, `ns_area_light` samples are taken.

- Visibility Testing:\*\* For each sampled direction, a shadow ray is cast. The `max_t` of this ray is set to `distToLight - EPS_D` to ensure the ray does not erroneously intersect with the light source itself.

- **Occlusion Check:** I used `bvh->has_intersection(shadow_ray)` for an efficient visibility check. If the path is unoccluded, the contribution of the light is added based on the lights's PDF, the BSDF, and the cosine term.

### 3.2 Sampling Results Comparison

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="bunny_H_64_32.png" caption="**Fig 1:** Uniform Hemisphere Sampling" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="bunny_64_32.png" caption="**Fig 2:** Importance Light Sampling" alt="Result 2" >}}
    </div>
</div>

### 3.3 Noise Levels in Soft Shadows (Importance Light Sampling)

To demonstrate the effect of importance sampling in rendering soft shadows, I rendered the `CBbunny.dae` scene with 1 sample per pixel (`-s 1`) while varying the number of light rays per pixel (`-l x`).

<div class="CBbunny_lx">
<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="bunny_1_1.png" caption="**Fig 1:** 1 Light Ray (-l 1)" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="bunny_1_4.png" caption="**Fig 2:** 4 Light Rays (-l 4)" alt="Result 2" >}}
    </div>
</div>

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="bunny_1_16.png" caption="**Fig 1:** 16 Light Rays (-l 16)" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="bunny_1_64.png" caption="**Fig 2:** 64 Light Rays (-l 64)" alt="Result 2" >}}
    </div>
</div>
</div>

**Analysis:**
As the number of light rays increases
from 1 to 64, the noise (graininess) in the soft shadow regions
is drastically reduced. At \(l=1\), the shadows appear as a
scattered collection of dots. By \(l=64\), the gradient of the
soft shadow becomes very smooth as the Monte Carlo estimator
gains more samples. Because we are sampling the lights directly,
we achieve relatively stable results even with a low number of
samples per pixel.

## Part 4: Global Illumination


