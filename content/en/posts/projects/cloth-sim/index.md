+++
date = '2026-01-10T01:20:57+09:00'
draft = false
title = 'Cloth Sim'
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

1.**Aerodynamic Refinement**

- _Initial Approach_: Applying wind forces directly to vertices, which resulted in unnatural, uniform translation.

- _Advanced Model_: Implemented a face-normal-based wind model. By calculating forces based on triangle orientations and relative velocity, and integrating a **turbulence function**, I achieved significantly more organic and complex cloth motion.

  1.**Performance Profiling & Bottleneck Analysis**:
  (Drawing on my High-Performance Computing (HPC) background, I utilized detailed profiling to identify critical performance sinks:)

* **Vector Operations**: High computational cost of per-face aerodynamic calculations.
* **Memory Management**: Excessive **heap allocation overhead** during spatial hash construction.
* **Data Locality**: High **cache miss rates** during spatial map traversal for self-collision detection.

  2.**Optimization Strategies (30% Aggregate Speedup)**:

* **Multi-threading**: Integrated **OpenMP** to parallelize wind force accumulation and self-collision impulse resolution.
* **Cache-Friendly Data Structures**: Replaced `std::unordered_map` with a **Flat Spatial Grid** (a sorted `std::vector`). This approach utilizes a contiguous memory block to drastically reduce cache misses and eliminate per-frame heap allocations.

  2.**XPBD Integration**:

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

### 3. Wireframe Visualizations (pinned2.json)

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
- **High** `ks`: With a high `ks` (e.g. \(50000N/m\)), the cloth becomes much stiffer, behaving more like a thin metal sheet or heavy canvas. It resists stretching and folding, and a flatter ap

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

### 4. Final Resting State (pinned4.json)

Below is the shaded cloth from `scene/pinned4.json` with `ks=1500`, `density=15g/m^2`, and `damping=20`.

<div style="width: 50%; margin: 0 auto;">
    {{< figure src="resting.png" caption="Final Resting State" >}}
</div>