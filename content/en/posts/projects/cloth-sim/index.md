+++
date = '2026-01-10T01:20:57+09:00'
draft = false
title = 'Cloth Sim'
tags = ['Graphics', 'Simulation']
summary = "Berkeley CS184 cloth simulation project, extra exploration on XPBD and performance optimization."
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
- *Initial Approach*: Applying wind forces directly to vertices, which resulted in unnatural, uniform translation.

- *Advanced Model*: Implemented a face-normal-based wind model. By calculating forces based on triangle orientations and relative velocity, and integrating a **turbulence function**, I achieved significantly more organic and complex cloth motion.

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

