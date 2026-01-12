+++
date = '2026-01-10T01:20:57+09:00'
draft = false
title = 'Cloth Sim'
showTableOfContents = true
tags = ['Graphics', 'Simulation', 'C++', 'XPBD', 'Performance Optimization']
summary = "Berkeley CS184 - C++ based cloth simulation project, extra exploration on XPBD and performance optimization."
description = "A deep dive into high-performance physics engineering: implementing Extended Position Based Dynamics and optimizing cache locality and parallelism in C++."
+++

{{< katex >}}

## Project Overview

In this project, I first implemented a basic mass-spring cloth simulation system. Including collision with other objects and self-collision. Then, I implemented several shaders through GLSL.

{{< video src="xpbd-better.webm" autoplay="true" loop="true" muted="true" preload="auto" playsinline="true" caption="Cloth simulation (XPBD) under extreme wind condition (Strength=2500)." >}}

In this project, I initially developed a foundational cloth simulator based on a **Mass-Spring system**. This included implementing robust object-environment interactions and self-collision handling, complemented by a suite of custom **GLSL shaders** for realistic material rendering.

### The Technical "Deep Dive"

The true technical depth of this project lies in the **External Exploration** phase, where I transitioned from basic functional implementation to high-performance physical engineering:

1. **Aerodynamic Refinement**

- _Initial Approach_: Applying wind forces directly to vertices, which resulted in unnatural, uniform translation.

- _Advanced Model_: Implemented a face-normal-based wind model. By calculating forces based on triangle orientations and relative velocity, and integrating a **turbulence function**, I achieved significantly more organic and complex cloth motion.

2. **Performance Profiling & Bottleneck Analysis**:
   (Drawing on my High-Performance Computing (HPC) background, I utilized detailed profiling to identify critical performance sinks:)

- **Vector Operations**: High computational cost of per-face aerodynamic calculations.
- **Memory Management**: Excessive **heap allocation overhead** during spatial hash construction.
- **Data Locality**: High **cache miss rates** during spatial map traversal for self-collision detection.

3. **Optimization Strategies (30% Aggregate Speedup)**:

- **Multi-threading**: Integrated **OpenMP** to parallelize wind force accumulation and self-collision impulse resolution.
- **Cache-Friendly Data Structures**: Replaced `std::unordered_map` with a **Flat Spatial Grid** (a sorted `std::vector`). This approach utilizes a contiguous memory block to drastically reduce cache misses and eliminate per-frame heap allocations.

4. **XPBD Integration**:

- To ensure numerical stability under extreme conditions (e.g., hurricane-force winds or near-infinite stiffness \\( k_s \\) ), I upgraded the solver to **Extended Position Based Dynamics**. Unlike traditional force-based models, XPBD treats constraints as position projections, ensuring the simulation remains unconditionally stable without "exploding."

## Part 1: Mass-Spring System

In this part, I implemented the physical construction of the cloth using a **mass-spring system**. The process involved two main steps:
generating a grid of `PointMass` objects and connecting them with `Spring` contraints to simulate the physical properties of the fabric.

### 1. Grid Construction

I implemented `Cloth::buildGrid` by iterating through `num_height_points` and `num_width_points`. For each point, I calculate its position based on the cloth's orientation:

- Horizontal: The points are spread across the XZ plane with a constant Y-coordinate of \\( 1.0 \\).

- Vertical: The points are spread across the XY plane with a small random Z-offset (between \( 0 \) and \( 0.001 \)) to break symmetry and prevent numerical instability.

I also cross-referenced each \( (x, y) \) coordinate with the `pinned` vector to set the `pinned` boolean for specific point masses, ensuring they remain stationary during the simulation.

### 2. Spring Construction

To give the cloth its structure integrity, I implemented three types of spring constraints:

- **Structural:** Connects a point mass to its neighbor directly to the left and directly above.

- **Shearing:** Connects a point mass to its neighbor diagonally (upper-left and upper-right).

- **Bending:** Connects a point mass to neighbors two units away (to the left and above) to provide resistance against folding.

### 3. Wireframe Visualizations

The following images show the wireframe structure of `scene/pinned2.json`. By toggling specific constraints, we can clearly see the different topologies created by the spirng types. (Click to zoom in)

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="no_sheering.png" caption="**Fig 1:** No Sheering Cloth" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="sheering.png" caption="**Fig 2:** Only Sheering Cloth" alt="Result 2" >}}
    </div>
    <div class="flex-1">
        {{< figure src="all.png" caption="**Fig 3:** All Constraints Cloth" alt="Result 3" >}}
    </div>
</div>

## Part 2: Simulation via numerical integration

In this part, I implemented the physical movement of the cloth using **Verlet Integration** and handled constraints to ensure the cloth doesn't over-stretch. By modifying parameters like `ks`, `density`, and `damping`, I observed how the cloth's material properties change.

### 1. Effects of Spring Constant `ks`

The `ks` value represents the stiffness of the springs.

- **Low** `ks`: With a very low `ks` (e.g. \(50N/m\)), the cloth behaves like a very soft, stretchy fabric (like thin spandex or silk). It exhibits significant stretch and many small, fine wrinkles as it falls to its resting state.
- **High** `ks`: With a high `ks` (e.g. \(50000N/m\)), the cloth becomes much stiffer, behaving more like a thin metal sheet or heavy canvas. It resists stretching and folding, and a flatter appearance is observed. (Note that for a very large `ks`, the spring based simulation may become unstable and explode.)

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="ks_50.png" caption="**Fig 1:** \(k_s=50\)" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="ks_50000.png" caption="**Fig 2:** \(k_s=50000\)" alt="Result 2" >}}
    </div>
</div>

### 2. Effects of Density

Density affects the mass of each `PointMass`, which in turn influences blow gravity pulls on the cloth.

- **Low Density:** The cloth feels "lighter". It sags very little and reaches its resting state quickly with minimal downward pull.

- **High Density:** The cloth feels "heavier". The gravitational force (`mass \* g) overpowers the spring forces, causing the cloth to sag more and stretch the springs further before reaching equilibrium.

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="density_5.png" caption="**Fig 1:** \(\rho=5g/m^2\)" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="density_5000.png" caption="**Fig 2:** \(\rho=5000g/m^2\)" alt="Result 2" >}}
    </div>
</div>

### 3. Effects of Damping

Damping simulated energy loss (like air resistance or internal friction). In my code, this is implemented as `(1 - damping / 100)` applied to the velocity term.

- **Low Damping:** The cloth is extremely "bouncy" and takes a long time to come to rest.
- **High Damping:** The cloth moves as if it is submerged in a thick iguid (like oil). It falls slowly and gracefully, coming to rest almost immediateky without any oscillation.

### 4. Final Resting State

Below is the shaded cloth from `scene/pinned4.json` with `ks=1500`, `density=15g/m^2`, and `damping=20`.

<div style="width: 50%; margin: 0 auto;">
    {{< figure src="resting.png" caption="**Fig 1:** Final Resting State" alt="Result 1" >}}
</div>

## Part 3: Handling collisions with other objects

In this part, I implemented the interaction between the cloth and other 3D primitives (spheres and planes) by calculating sollision responses that push the cloth's point masses back to the surface of the object.

### 1. Implementation details

- **Sphere Collision:** For each point mass, I check if its distance from the sphere's origin is less than or equal to the radius. If a collision is detected, I calculate the **tangent point** on the sphere's surface by extending the vector from the origin through the current position. THe point mass is then moved toward this tangent vector using a correction vector originating from its `last_position`, scaled by `(1 - friction)`.

- **Plane Collision:** I use the dot product of the point mass's position (relative to a point on the plane) and the plane's normal to determine if the cloth has crossed from one side to the other. Upon crossing, I calculate the intersection point (tangent point) and offset it slightly by `SURFACE_OFFSET` to prevent the cloth from getting stuck inside the plane. The final position is updated similarly using a friction-scaled correction vector from the `last_position`.

### 2. Sphere Collision Examples

Varing the spring constant `ks` significantly alters the cloth's stiffness and how it drapes over the sphere:

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="sphere_ks_500.png" caption="**Fig 1:** \(k_s=50\)" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="sphere_ks_5000.png" caption="**Fig 2:** \(k_s=50000\)" alt="Result 2" >}}
    </div>
    <div class="flex-1">
        {{< figure src="sphere_ks_50000.png" caption="**Fig 3:** \(k_s=500000\)" alt="Result 3" >}}
    </div>
</div>

### 3. Plane Collision Example

The following image show the shaded cloth lying at rest on the plance. The collision response ensures the cloth remains on the surface without clipping.

<div style="width: 50%; margin: 0 auto;">
    {{< figure src="plane.png" caption="**Fig 1:** Shaded cloth lying at rest on the plane." alt="Result 1" >}}
</div>

## Part 4: Handling self-collisions

To simulate realistic fabric folding without the cloth passing through itself, I implemented a **spatial hashing** algorithm to efficiently detect and resolve self-collisions.

### 1. Implementation details

- **Spatial Hashing:** I implemented `hash_position` to partition 3D space into uniform grid boxes. Each `PointMass` is mapped to a unique grid box based on its position. In each step, `build_spatial_map` populates an `unordered_map`, allowing me to only check collisions between points in the same box.

- **Collision Resolution:** For each point mass, I iterate through other points in its hash bucket. If two points are closer than `2 * thickness`, I apply a correction vector to push them aprat. These corrections are averaged and applied to the position, scaled down by `simalation_steps` to ensure stability.

### 2. Cloth falling and folding

The sequence below captures the cloth as it falls, makes initial contact with itself, and eventually settles into a folded state.

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="early_collision.png" caption="**Fig 1:** Early collision" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="middle_collision.png" caption="**Fig 2:** Initial self collision" alt="Result 2" >}}
    </div>
    <div class="flex-1">
        {{< figure src="final_collision.png" caption="**Fig 3:** Final self collision" alt="Result 3" >}}
    </div>
</div>

### Parameter Analysis

- **Density:** Increasing density makes the cloth heavier. Higher ensity leads to more compression and more frequent self-collisions.
- **Spring Constant (ks):** A higher `ks` increases the cloth's resistance to bending and stretching. When `ks` is high, the cloth maintains larger, more open folds and resists overlapping.

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="self_col_den_1500.png" caption="**Fig 1:** High density" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="self_col_ks_50000.png" caption="**Fig 2:** High spring constant" alt="Result 2" >}}
    </div>
</div>

## Part 5: Shaders

### 1. Shader Programs

A shader program is a set of instructions that runs on the GPU to handle the graphics pipeline efficiently. In this project, shader programs are written in **GLSL** and consists two main components: **Vertex Shader** and **Fragment Shader**.

- **Vertex Shader:** These process individual vertices. They take in per-vertex properties like positions and normals, apply transformations (such as model and view-projection matrices), and compute the final screen-space `gl_Position`. They also pass data to the fragment shader through "out" variables.

- **Fragment Shader:** These process the pixels (fragments) generated by the rasterizer. They take the interpolated data from the vertex shader to compute the final pixel color, enabling complex lighting and material effects like specular highlights and textures.

### 2. Blinn-Phong Shading Model

The Blinn-Phong shading model calculates a surface's color by combining three components:

- **Ambient:** A constant base color representing indirect light.
- **Diffuse:** Matte-like reflection that depends on the angle between the light source and the surface normal (Lambertian reflection).
- **Specular:** Bright highlights that appear when the viewer is near the path of light reflection. It uses a "half-vector" between the view and light directions for better performance than standard Phong model.

<div class="phong-model">
<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="ambient.png" caption="**Fig 1:** Ambient Only" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="diffuse.png" caption="**Fig 2:** Diffuse Only" alt="Result 2" >}}
    </div>
</div>

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="specular.png" caption="**Fig 1:** Specular Only" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="phong.png" caption="**Fig 2:** Phong Shading" alt="Result 2" >}}
    </div>
</div>
</div>

### 3. Textures Mapping

Texture mapping allows us to map a 2D image onto a 3D surface using UV coordinates. The shader samples the albedo from the texture and integrates it into the lighting calculations.

<div style="width: 50%; margin: 0 auto;">
    {{< figure src="texture_map.png" caption="**Fig 1:** Textures Mapping" alt="Result 1" >}}
</div>

### 4. Bump vs. Displacement Mapping

I implemented both bump and displacement mapping to add surface details:

- **Bump Mapping:** Modifies the surface normals based on the derivatives (du, dv) of a height map. This creates the illusion of depth and detail through lighting without changing the actual geometry of the surface.
- **Displacement Mapping:** Actually modifies the vertex positions in the vertax shader by shifting them along their normals according to the height map.

\*Comparison:\*\* Bump mapping is computationally efficient and provides convincing surface details, but the sihouette remains perfectly smooth. Displacement mapping provides a realistic sihouette and physical geometry changes, but it requires a high mesh resolution to avoid blockiness.

<div class="bump-displace">
<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="bump_128.png" caption="**Fig 1:** Bump Mapping" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="disp_128.png" caption="**Fig 2:** Displacement Mapping" alt="Result 2" >}}
    </div>
</div>

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="bump_sphere.png" caption="**Fig 1:** Bump Mapping on Sphere" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="disp_sphere.png" caption="**Fig 2:** Displacement Mapping on Sphere" alt="Result 2" >}}
    </div>
</div>
</div>

### 5. Mirror Shader

The mirror shader uses an environment cubemap to simulate perfect reflection. By calculating a reflection vector based on the camera position and surface normal, we sample the environment map to get the final color.

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="mirror_cloth.png" caption="**Fig 1:** Mirror Shader on Cloth" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="mirror_sphere.png" caption="**Fig 2:** Mirror Shader on Sphere" alt="Result 2" >}}
    </div>
</div>

## Part 6: Extra Exploration

### 1. Wind Simulation

I implemented a wind simulation system to simulate the effect of wind on the cloth. at first, I used a simple technique that just add a sinusoidal force to every point mass in the cloth. However, this technique is not realistic enough. 

Then I implemented a more physics-based approach using those metrics:
- Aerodynamic Projection: The drag force is calculated by projecting the relative velocity \(\mathbf{v}_{rel} = \mathbf{v}_{wind} - \mathbf{v}_{cloth}\) onto the triangle's unit normal. By using a dot product, we extract only the wind component perpendicular to the surface and scale it by the triangle's area \(\mathbf{n}_{area}\) to determine the final force:
$$
\mathbf{F}_{drag} = \mathbf{n}_{area} \cdot \left(\mathbf{v}_{rel} \cdot \frac{\mathbf{n}_{area}}{\|\mathbf{n}_{area}\|} \right)
$$

- **Procedural Turbulence:** To simulate organic, gusty wind, I used a multi-frequency sine function that modulates the force based on space and time. This prevents the cloth from moving too uniformly by summing waves with different amplitudes and frequencies:
$$T(s, t) = \sum \alpha_i \sin(\omega_i s + \phi_i t)$$

This creates a pseudo-random scaler \(T\) used to perturb the base force by up to \(30\%\):
$$\mathbf{F}_{final} = \mathbf{F}_{drag} \cdot (1.0 + 0.3 T)$$

{{< video src="xpbd-better.webm" autoplay="true" loop="true" muted="true" preload="auto" playsinline="true" caption="**Fig1:** Wind Simulation" >}}

### 2. Profiling and Optimization

#### 2.1 **Profiling** 
To optimize the performance of the cloth simulation, the first step is to profile the code to **identify the bottlenecks**. I wrote my own profiler to measure the time taken by each function in the simulation, which updated every frame step and printed the results to the console:

| Physics Module | Avg (ms/step) | Percentage |
| :--- | :--- | :--- |
| **self_collision** | 0.187 | 50.9% |
| **wind** | 0.108 | 29.4% |
| **springs** | 0.044 | 11.9% |
| Provot | 0.021 | 5.8% |
| Verlet | 0.005 | 1.2% |
| gravity | 0.003 | 0.8% |
| collision | 0.000 | 0.0% |

The results show that self_collision and wind are the most time-consuming functions, accounting for 50.9% and 29.4% of the total time, respectively, which is the main part we will be working on.

#### 2.2 **Optimization**

To achieve real-time performance, I implemented several hardware-level optimizations focusing on parallel throughput and memory locality:

- **Parallel Computing with OpenMP:** I parallelized the force calculations and constraint projections. By using #pragma omp parallel for with thread-safe atomic operations for wind forces, I significantly reduced the per-step computation time across multiple CPU cores.Data 

- **Locality & Spatial Hashing:** I replaced the traditional `std::unordered_map` with a `Sorted Continuum Data Structure`. By flattening the spatial map into a single contiguous std::vector and sorting it by hash value, I minimized Cache Misses. This allows the CPU's prefetcher to efficiently load neighboring particles during collision detection, as they are now stored in adjacent memory addresses.

| Physics Module          | Baseline (Avg ms/step) | Optimized (Avg ms/step) | Performance Gain |
| :---------------------- | :--------------------- | :---------------------- | :--------------- |
| **Self-Collision (Total)**| 0.187                  | **0.159** | **+15.0%** |
| **Wind Force** | 0.108                  | **0.066** | **+38.9%** |
| **Spring Constraints** | 0.044                  | 0.050                   | -13.6%           |
| **Numerical Integration**| 0.021                  | 0.025                   | -19.0%           |
| **Verlet / Gravity** | 0.008                  | 0.008                   | 0%               |
| **Total Step Time** | **0.368** | **0.308** | **+16.3%** |

Note That `Spring` and `Provot` Avg time increased, this (+0.005ms) may due to OpenMP context switching, which can be accounted to system error.

### 3. XPBD Integration

So far, the Verlet integration is fast but has a major flaw: it is not energy conserving. For example, when the `ks` of the cloth is very large, the cloth would explode. And for strong wind, the cloth will be stretching in a way it shouldn't. To address those flaws, I searched for a better integration method and found XPBD widely used in game industry. 

#### 2.1 Extended Position Based Dynamics (XPBD)

Unlike force-based solvers (Like Mass-Spring systems) which can explode if the `ks` is too large, XPBD treats constraints as geometric targets. It essentially asks: "Where should these two points move so that the distance between them is exactly `L`?

**The XPBD algorithm loop** follows a specific four-stage pipeline for every time step:

1. **Prediction:** We ignore all constraints and apply external forces (Gravity, Wind). The mass `m` is moved to a temporary position predicted position `p` based on its current velocity:

$$\mathbf{p} = \mathbf{x} + \mathbf{v} \Delta t + \mathbf{a}_{ext} \Delta t^2$$

2. **Constraints Projection:** The is the core of the solver. We iterate through all constraints and adjust the predicted positions to satisfy the constraints. For a distance constraint \(C = \|\mathbf{p}_1 - \mathbf{p}_2\| - L\), the correction \(\Delta \lambda\) is calculated as:$$\Delta \lambda = \frac{-C - \tilde{\alpha} \lambda}{\sum w_i + \tilde{\alpha}}$$

Where \(\tilde{\alpha} = \alpha / \Delta t^2\). This ensures that even with a high density like \(128 \times 128\), the cloth doesn't look like rubber.

3. **Collision Handling:** After spring constraints are solved, we check for environment and self-collisions. Any points penetrating objects or other parts of the cloth are projected back to the surface.

4. **Update:** Finally, the velocity is updated based on the difference between the old position and the new, constraint-satisfied predicted position.
$$\mathbf{v}_{new} = (\mathbf{p}_{final} - \mathbf{x}) / \Delta t$$
$$\mathbf{x}_{new} = \mathbf{p}_{final}$$