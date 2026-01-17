+++
date = '2026-01-13T18:22:25+09:00'
draft = false
title = '物理渲染引擎实现：光线追踪'
showTableOfContents = true
tags = ['图形学', '渲染', 'C++', '性能优化', 'HPC']
summary = "在 Berkeley CS184 框架下从零实现的 C++ 物理渲染器。重点讨论了基于 SAH 的 BVH 构建优化、自适应采样算法以及多线程负载均衡下的性能调优。"
description = "本文详细介绍了从底层射线求交到全局光照管线的实现过程，并展示了如何通过统计学原理和启发式算法实现 2.25 倍的渲染加速。"
+++

{{< katex >}}

## 项目综述

<div style="width: 80%; margin: 0 auto;">
    {{< figure src="cornell.png" caption="**图 1:** 直接光照 vs 全局光照 (低采样) vs 全局光照 (高采样)" alt="Result 1" >}}
</div>

本项目是基于 **C++** 开发的物理渲染引擎（Path Tracer），实现了从底层几何求交到高阶全局光照的全栈管线。在完成核心渲染逻辑的基础上，我结合**高性能计算**的思维，针对加速结构效率和采样收敛速度进行了深度优化。

**核心技术亮点：**
- **底层求交引擎：** 实现了基于 **Möller-Trumbore** 算法的高效三角面片求交。
- **BVH 加速结构：** 引入 **SAH (Surface Area Heuristic)** 与 **Binning 策略**，极大提升了复杂场景的遍历速度。
- **方差缩减技术：** 实现了光源重要性采样（Importance Sampling）与俄罗斯轮盘赌（Russian Roulette）机制。
- **性能工程：** 针对多线程负载不均问题优化了 Tile 调度，并引入**自适应采样**实现算力动态分配。

---

## 1. 核心求交与管线构建

渲染器的基础在于将 2D 图像空间映射到 3D 物理世界。在 `Camera::generate_ray` 中，我构建了完整的相机模型，处理了从传感器平面到世界空间的坐标变换。

对于最核心的求交逻辑，我实现了 **Möller-Trumbore** 算法。该算法通过克莱姆法则直接解算重心坐标 \((1-b_1-b_2, b_1, b_2)\) 和射线参数 \(t\)。
- **有效性判定：** 严格限制 \(t\) 在相机的近、远裁剪平面内，并确保重心坐标符合几何约束。

### 1.1 基础测试：法线着色验证

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="CBempty.png" caption="**图 2:** 空 Cornell Box" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="CBspheres.png" caption="**图 3:** 包含球体的 Cornell Box" alt="Result 2" >}}
    </div>
    <div class="flex-1">
        {{< figure src="banana.png" caption="**图 4:** 香蕉模型测试" alt="Result 3" >}}
    </div>
</div>

---

## 2. BVH 加速结构：从 \(O(N)\) 到 \(O(\log N)\)

面对数万面片的模型（如 Dragon），传统的线性求交是极其缓慢的的。我实现了一套基于 **SAH (表面积启发式)** 的 BVH 树。

### 2.1 分割策略优化 (Binning)
为了避免 \(O(N^2)\) 的枚举开销，我采用了 **Binning (桶划分)** 近似方案：
1. 将选定轴均匀划分为 **16 个 Bin**。
2. 统计各 Bin 的边界框（AABB）与图元数量。
3. 利用**前缀/后缀扫描**评估潜在分割点的 SAH 代价。

$$C = C_{trav} + \frac{S_A}{S_{total}} N_A C_{isect} + \frac{S_B}{S_{total}} N_B C_{isect}$$



<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="sah-1.png" caption="**图 5:** SAH 根节点划分" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="sah-2.png" caption="**图 6:** 子节点递归划分" alt="Result 2" >}}
    </div>
    <div class="flex-1">
        {{< figure src="sah-3.png" caption="**图 7:** 叶节点细节" alt="Result 2" >}}
    </div>
</div>

### 2.2 复杂模型渲染结果

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="maxplanck.png" caption="**图 8:** Max Planck (复杂网格)" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="CBlucy.png" caption="**图 9:** Lucy 雕像 (高面片数)" alt="Result 2" >}}
    </div>
</div>

### 2.3 性能分析

我对比了朴素实现与 BVH 加速实现在不同复杂度模型下的渲染性能。所有测试均在搭载 M2 芯片的 MacBook Pro 上以单线程方式运行。

| Scene            | Triangle Count | Render Time (No BVH) | Render Time (With BVH) |
| :--------------- | :------------- | :------------------- | :--------------------- |
| **cow.dae**      | 5,856          | 32.48 s              | 0.0831 s               |
| **beetle.dae**   | 7,512          | 38.96 s              | 0.0728 s               |
| **CBdragon.dae** | 105,120        | > 1 hour             | 0.0899 s               |

---

## 3. 直接光照与采样策略比较

我实现了两种不同的直接光照估算算法：**均匀半球采样**与**光源重要性采样**。

- **光源重要性采样：** 直接在光源表面采样，显著降低了软阴影区域的方差。

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="bunny_H_64_32.png" caption="**图 10:** 均匀半球采样 (噪声较高)" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="bunny_64_32.png" caption="**图 11:** 光源重要性采样 (更平滑)" alt="Result 2" >}}
    </div>
</div>

### 3.1 阴影质量与采样光线数的关系 (\(l\) 参数)

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="bunny_1_1.png" caption="1 条光线 (-l 1)" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="bunny_1_64.png" caption="64 条光线 (-l 64)" alt="Result 2" >}}
    </div>
</div>

---

## 4. 全局光照 (Global Illumination)

通过模拟多次光线弹射，实现了颜色透射 (Color Bleeding) 和软阴影填充。

- **俄罗斯轮盘赌：** 采用概率 \(0.65\) 作为无偏终止条件，在性能与光路深度间取得平衡。

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="CBbunny_1024_4_1.png" caption="**图 12:** 仅直接光照" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="CBbunny_1024_4_100_Russo.png" caption="**图 13:** 全局光照 (多级弹射)" alt="Result 2" >}}
    </div>
</div>

### 4.1 光线弹射分析 (n-th bounce)

<div class="light-bounce-m">
<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="CBbunny_1024_4_0.png" caption="**Fig 1:** 0th Bounce" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="CBbunny_1024_4_1.png" caption="**Fig 2:** 1st Bounce" alt="Result 2" >}}
    </div>
    <div class="flex-1">
        {{< figure src="CBbunny_1024_4_2.png" caption="**Fig 3:** 2nd Bounce" alt="Result 2" >}}
    </div>
</div>
<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="CBbunny_1024_4_3.png" caption="**Fig 4:** 3rd Bounce" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="CBbunny_1024_4_4.png" caption="**Fig 5:** 4th Bounce" alt="Result 2" >}}
    </div>
    <div class="flex-1">
        {{< figure src="CBbunny_1024_4_5.png" caption="**Fig 6:** 5th Bounce" alt="Result 2" >}}
    </div>
</div>
</div>


### 4.2 光线弹射累计结果

<div class="light-bounce-full">
<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="CBbunny_1024_4_0.png" caption="**Fig 1:** 0th Bounce" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="CBbunny_1024_4_1.png" caption="**Fig 2:** 1st Bounce" alt="Result 2" >}}
    </div>
    <div class="flex-1">
        {{< figure src="CBbunny_1024_4_2_full.png" caption="**Fig 3:** 2nd Bounce" alt="Result 2" >}}
    </div>
</div>
<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="CBbunny_1024_4_3_full.png" caption="**Fig 4:** 3rd Bounce" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="CBbunny_1024_4_4_full.png" caption="**Fig 5:** 4th Bounce" alt="Result 2" >}}
    </div>
    <div class="flex-1">
        {{< figure src="CBbunny_1024_4_5_full.png" caption="**Fig 6:** 5th Bounce" alt="Result 2" >}}
    </div>
</div>
</div>


---

## 5. 高性能优化与自适应采样

### 5.1 负载均衡优化
针对多线程环境下 Tile 复杂度不均导致的“长尾效应”，我将 Tile 大小优化为 **16x16**，使 8 线程利用率更加均匀。

<div style="width: 95%; margin: 0 auto;">
    {{< figure src="profile.png" caption="**图 14:** Tray Profiler 可视化：不同区域 Tile 的渲染耗时差异显著20ms vs. 1000ms" alt="Result 1" >}}
</div>

### 5.2 自适应采样 (Adaptive Sampling)
基于统计学 95% 置信区间判断像素收敛：

$$I = 1.96 \cdot \frac{\sigma}{\sqrt{n}} \leq \text{maxTolerance} \cdot \mu$$

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="CBbunny_1024_4_32_rate.png" caption="**图 15:** 无自适应采样热力图" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="CBbunny_1024_4_32_a_rate.png" caption="**图 16:** 自适应采样热力图 (蓝色为早停区域)" alt="Result 2" >}}
    </div>
</div>

**优化结果：** 在 `CBbunny` 场景中，自适应采样将总耗时从 **248s** 压缩至 **110s**，实现了 **2.24x** 的效率提升。

### 5.3 优化总结

通过实现自适应采样，我们可以显著减少渲染时间，特别是在大型均匀区域（如 Cornell Box 的墙壁）上，同时在复杂区域（如兔子的表面）上保持高保真度。

在 CBbunny.dae 场景测试中（参数设置：`max_ray_depth = 32`，`samplesPerPixel = 1024`，`light_ray_per_sample = 4`），自适应采样将总渲染时间从 **248.2378s** 优化至 **110.7405s**。具体的性能画像（Profiling）分析结果详见下表：

| Region Type                       | Before Optimization | After Optimization | Speedup Factor |
| :-------------------------------- | :------------------ | :----------------- | :------------- |
| **Simple Areas (e.g., Walls)**    | 20 ms / tile        | 3 ms / tile        | ~6.67x         |
| **Complex Areas (e.g., Shadows)** | 1,000 ms / tile     | ~630 ms / tile     | ~1.59x         |
| **Total Render Time (CBbunny)**   | 248.2 s             | 110.7 s            | 2.24x          |

---

## 6. 更多 BSDF 材质展示

扩展了对微表面模型 (Microfacet) 及电介质 (Glass) 的支持：

<div class="more-bsdfs">
<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="dragon_cu.png" caption="铜材质 Dragon" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="featured.png" caption="铜材质 Bunny" alt="Result 2" >}}
    </div>
    <div class="flex-1">
        {{< figure src="gems_1024_4_a.png" caption="玻璃宝石" alt="Result 3" >}}
    </div>
</div>
</div>

## 总结与未来展望

通过本项目，我深入掌握了物理渲染管线的底层细节。未来我计划将遍历逻辑迁移至 **GPU (CUDA)**，并集成 **AI 降噪** 技术，探索实时光线追踪的边界。

期待回国后在配置了 **RTX 5070 Ti** 的新机器上，继续我的图形学探索之旅。