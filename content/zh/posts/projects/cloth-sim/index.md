+++
title = '进阶布料仿真：从质点弹簧到 XPBD 动力学'
date = '2026-01-10T01:20:57+09:00'
draft = false
tags = ['图形学', '物理仿真', '性能优化', 'C++']
summary = "加州大学伯克利分校基于C++实现的CS184布料仿真项目，重点拓展 XPBD 算法实现与系统级性能优化。"
description = "深度解析高性能物理仿真工程：基于 C++ 实现扩展基于位置的动力学（XPBD），并针对缓存局部性与并行化处理进行极致优化。"
+++

{{< katex >}}

## 项目概览

下方是使用 **扩展基于位置的动力学 (XPBD)** 算法实现的最终仿真结果。

{{< video src="xpbd-better.webm" autoplay="true" loop="true" muted="true" preload="auto" playsinline="true" caption="极强风力环境下的 XPBD 布料仿真（风力强度：2500）。" >}}

本项目中，我首先构建了一个基础的**质点弹簧系统**，实现了布料与外部物体的碰撞检测及自碰撞处理。同时，我利用 **GLSL** 编写了多款着色器以增强视觉表现力。

### 技术核心：进阶探索与性能优化
我认为本项目的核心竞争力在于**额外技术探索**阶段。在该阶段，我完成了从“功能实现”到“工业级优化”的跨越：

1.  **空气动力学模型改良**：
    * **初始方案**：直接将风力作用于顶点，导致布料运动过于僵硬且缺乏真实感。
    * **改进方案**：实现了基于三角形面片法线的风力模型。通过计算面片朝向与风速的相对关系，并引入**紊流 (Turbulence)** 算法，使布料产生了逼真的起伏和褶皱效果。
2.  **性能诊断与瓶颈分析**：
    基于我的高性能计算 (HPC) 背景，我通过 Profiler 对仿真流程进行了细粒度分析，识别出以下瓶颈：
    * **风力计算开销**：大量向量叉积运算占据了显著的 CPU 耗时。
    * **内存分配瓶颈**：使用 `std::unordered_map` 构建空间哈希时产生的**堆内存分配开销**。
    * **缓存命中率**：空间哈希遍历时的随机内存访问导致了严重的 **Cache Miss (缓存缺失)**。
3.  **针对性优化 (整体性能提升 30%)**：
    * **多线程并行**：将 **OpenMP** 整合到项目中，利用 **OpenMP** 对风力累加和自碰撞检测逻辑进行了并行化改造。
    * **数据结构重构**：将基于节点的哈希表重构为 **Flat Spatial Grid (扁平空间网格)**。通过使用排序后的连续 `std::vector` 存储空间索引，消灭了每帧的堆内存申请，并极大提升了 CPU 缓存局部性。
4.  **XPBD 算法实现**：
    为了解决强风场或高刚度 (\\(k_s\\)) 条件下的数值爆炸问题，我将求解器升级为 **XPBD** 架构。 该算法将约束视为位置投影而非显式力，确保了仿真在极端物理环境下依然具备极高的稳定性。

## 第一部分：质点弹簧系统 (Mass-Spring System)

在布料仿真的第一阶段，我构建了布料的物理骨架——**质点弹簧系统**。该系统通过将布料离散化为相互连接的质点网格，并利用弹簧约束模拟织物的抗拉、抗剪及抗折叠特性。

### 1.1 网格构建与空间分布

在 `Cloth::buildGrid` 的实现中，我通过双重循环生成质点。根据场景配置文件，布料分为水平和竖直两种初始分布逻辑：

* **水平分布 (Horizontal)：** 质点均匀展布于 XZ 平面，其 \( Y \) 轴坐标固定为 \( 1.0 \)。
* **竖直分布 (Vertical)：** 质点分布于 XY 平面。为了打破计算中的完美对称性、降低数值不稳定性（Numerical Instability），我对每个质点的 \( Z \) 坐标添加了极小的随机扰动（范围在 \( 0 \) 至 \( 0.001 \) 之间）。
* **约束固定 (Pinned)：** 系统通过读取 `pinned` 向量，将对应坐标 \( (x, y) \) 的质点标记为固定状态，这些质点在后续动力学计算中将保持静止。

### 1.2 弹簧约束及其物理意义

为了模拟真实织物的物理响应，我为每对质点建立了三种类型的弹簧约束。这些约束共同定义了布料的杨氏模量与抗形变能力：

1. **结构约束 (Structural)：** 连接当前质点与其左侧及上方相邻质点，模拟布料纤维的基本拉伸抗力。
2. **剪切约束 (Shearing)：** 连接当前质点与其左上及右上对角线的质点，防止网格在平面内发生过度平行四边形化形变。
3. **弯曲约束 (Bending)：** 连接当前质点与其间隔一个单位（距离为 2）的质点，用于限制布料的折叠角度，模拟织物的硬挺度。



### 1.3 线框模式可视化 (Wireframe Visualizations)

通过可视化线框结构，我们可以直观观察到不同弹簧类型叠加后的拓扑结构变化（点击图片可放大）：

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="no_sheering.png" caption="**图 1:** 无剪切约束 (No Shearing)" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="sheering.png" caption="**图 2:** 仅开启剪切约束" alt="Result 2" >}}
    </div>
    <div class="flex-1">
        {{< figure src="all.png" caption="**图 3:** 全约束开启 (完整拓扑)" alt="Result 3" >}}
    </div>
</div>

---

{{< katex >}}

## 第二部分：数值积分与参数分析 (Numerical Integration)

### 2.1 Verlet 积分实现
为了模拟布料的动态运动，我实现了基于 **Verlet 积分**的物理引擎。相比于欧拉积分，Verlet 积分在处理约束时具有更好的稳定性。

### 2.2 物理参数对布料表现的影响
通过调整模拟参数，布料表现出截然不同的材质感：

- **弹性系数 \(k_s\)**：低 \(k_s\) 使布料柔软易拉伸，褶皱细碎；高 \(k_s\) 使布料僵硬如帆布。
- **密度 (\(\rho\))**：高密度增加重力负载，导致布料产生更明显的下垂与拉伸。
- **阻尼 (Damping)**：模拟能量损耗（如空气阻力）。低阻尼导致持续摆动，高阻尼使运动平滑且迅速静止。



<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="ks_50.png" caption="\(k_s=50\) (极其柔软)" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="ks_50000.png" caption="\(k_s=50000\) (较为坚挺)" alt="Result 2" >}}
    </div>
</div>

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="density_5.png" caption="低密度 (\(5g/m^2\))" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="density_5000.png" caption="高密度 (\(5000g/m^2\))" alt="Result 2" >}}
    </div>
</div>

---

## 第三部分：物体碰撞处理 (Collisions)

我实现了布料与几何原语（球体与平面）的交互逻辑，通过计算修正向量将质点推回物体表面。

- **球体碰撞**：检测质点到球心的距离。若小于半径，则沿着质点当前位置与球心的连线方向，将质点修正至球面上。

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="sphere_ks_500.png" caption="\(k_s=500\) 碰撞表现" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="sphere_ks_5000.png" caption="\(k_s=5000\) 碰撞表现" alt="Result 2" >}}
    </div>
    <div class="flex-1">
        {{< figure src="sphere_ks_50000.png" caption="\(k_s=50000\) 碰撞表现" alt="Result 3" >}}
    </div>
</div>

- **平面碰撞**：通过点积判断质点是否穿过平面。若发生碰撞，计算交点并在法线方向添加微小的 `SURFACE_OFFSET` 偏移量。

<div style="width: 50%; margin: 0 auto;">
    {{< figure src="plane.png" caption="布料在平面上的最终静止状态" alt="Result 1" >}}
</div>

---

## 第四部分：自碰撞处理 (Self-Collisions)

为了防止布料穿模并模拟真实的折叠效果，我引入了基于**空间哈希 (Spatial Hashing)** 的自碰撞检测算法。

- **空间哈希**：将 3D 空间划分为网格。每帧将质点映射到哈希桶中，仅对同一桶内的质点进行距离检测，将复杂度从 \(O(N^2)\) 优化至接近线性。
- **碰撞消除**：若质点间距小于 \(2 \times thickness\)，则施加斥力修正向量。



<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="early_collision.png" caption="初始降落" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="middle_collision.png" caption="开始产生自碰撞" alt="Result 2" >}}
    </div>
    <div class="flex-1">
        {{< figure src="final_collision.png" caption="完全折叠状态" alt="Result 3" >}}
    </div>
</div>

---


## 第五部分：着色器 (Shaders)

着色器是运行在 GPU 上的并行程序，用于高效处理图形渲染管线。本项目使用 **GLSL** 编写，主要包含两个核心组件：

- **顶点着色器 (Vertex Shader)：** 处理单个顶点。负责应用模型-视图-投影（MVP）变换，计算顶点在屏幕空间的最终位置 `gl_Position`，并将位置、法线等属性传递给片元着色器。
- **片元着色器 (Fragment Shader)：** 处理光栅化生成的像素（片元）。利用顶点着色器插值后的数据计算像素最终颜色，实现复杂的动态光照与材质效果。

### 5.1 Blinn-Phong 着色模型

Blinn-Phong 模型通过组合以下三个分量来计算表面颜色：

1. **环境光 (Ambient)：** 模拟环境中的间接光照，赋予表面基础底色。
2. **漫反射 (Diffuse)：** 表现粗糙表面的反光，取决于光源方向与表面法线的夹角（Lambertian 反射）。
3. **高光 (Specular)：** 表现平滑表面的亮斑。利用视角方向与光源方向的“半向量 (Half-vector)”进行计算，性能优于经典的 Phong 模型。



<div class="phong-model">
<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="ambient.png" caption="**图 1:** 仅环境光分量" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="diffuse.png" caption="**图 2:** 仅漫反射分量" alt="Result 2" >}}
    </div>
</div>

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="specular.png" caption="**图 3:** 仅高光分量" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="phong.png" caption="**图 4:** 完整 Blinn-Phong 着色" alt="Result 2" >}}
    </div>
</div>
</div>

---

### 5.2 纹理映射 (Texture Mapping)

纹理映射通过 **UV 坐标** 将 2D 图像采样并映射至 3D 表面，显著增强布料的细节表现力。

<div style="width: 50%; margin: 0 auto;">
    {{< figure src="texture_map.png" caption="**图 5:** 纹理映射效果演示" alt="Result 1" >}}
</div>

---

### 5.3 凹凸映射 (Bump) vs. 位移映射 (Displacement)

我实现了两种增强表面细节的技术：

- **凹凸映射 (Bump Mapping)：** 基于高度图的导数（\(du, dv\)）修改表面法线。它通过光照计算产生深度错觉，但**不改变**物体的实际几何形状，轮廓线依然平滑。
- **位移映射 (Displacement Mapping)：** 在顶点着色器中根据高度图沿法线方向**实际偏移**顶点位置。

**对比分析：** 凹凸映射计算开销小，适合表现微小细节；位移映射能提供真实的物理轮廓和阴影变化，但需要极高的网格细分程度以防止锯齿。

<div class="bump-displace">
<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="bump_128.png" caption="**图 6:** 布料凹凸映射" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="disp_128.png" caption="**图 7:** 布料位移映射" alt="Result 2" >}}
    </div>
</div>

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="bump_sphere.png" caption="**图 8:** 球体凹凸映射" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="disp_sphere.png" caption="**图 9:** 球体位移映射" alt="Result 2" >}}
    </div>
</div>
</div>

---

### 5.4 环境反射 (Mirror Shader)

环境反射着色器使用**立方体贴图 (Cubemap)** 来模拟完美的镜面效果。通过计算视角相对于表面法线的反射向量，对环境图进行采样获取颜色。

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="mirror_cloth.png" caption="**图 10:** 布料环境反射" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="mirror_sphere.png" caption="**图 11:** 球体环境反射" alt="Result 2" >}}
    </div>
</div>

---
## 第六部分：进阶探索 (Extra Exploration)

### 6.1 动态风场模拟

我实现了一套更符合物理规律的风场系统。最初采用简单的正弦力场，但视觉效果欠佳。随后我升级为基于**空气动力学投影**的模型：

- **空气动力学投影 (Aerodynamic Projection)：** 阻力根据相对速度 \(\mathbf{v}_{rel} = \mathbf{v}_{wind} - \mathbf{v}_{cloth}\) 在三角形法线方向的投影计算。通过点积提取垂直于表面的风力分量，并结合受力面积 \(\mathbf{n}_{area}\) 确定最终大小：
$$
\mathbf{F}_{drag} = \mathbf{n}_{area} \cdot \left(\mathbf{v}_{rel} \cdot \frac{\mathbf{n}_{area}}{\|\mathbf{n}_{area}\|} \right)
$$

- **程序化湍流 (Procedural Turbulence)：** 为了模拟自然界阵风的不规则感，我使用多频正弦函数叠加产生空间和时间相关的调制系数，使风力波动更自然：
$$T(s, t) = \sum \alpha_i \sin(\omega_i s + \phi_i t)$$
$$\mathbf{F}_{final} = \mathbf{F}_{drag} \cdot (1.0 + 0.3 T)$$

{{< video src="xpbd-better.webm" autoplay="true" loop="true" muted="true" preload="auto" playsinline="true" caption="**图 1:** 实时风场模拟效果" >}}

---

### 6.2 性能画像与优化 (Profiling & Optimization)

#### 6.2.1 **瓶颈分析**
为了提升实时性，我编写了自定义 Profiler 来监测每帧各模块的耗时。结果显示，**自碰撞检测**和**风力计算**是核心瓶颈，占据了总耗时的 80.3%。

| 物理模块 | 平均耗时 (ms/步) | 占比 |
| :--- | :--- | :--- |
| **自碰撞检测 (Self-collision)** | 0.187 | **50.9%** |
| **风场模拟 (Wind)** | 0.108 | **29.4%** |
| **弹簧约束 (Springs)** | 0.044 | 11.9% |
| 其他 (积分/重力等) | 0.029 | 7.8% |

#### 6.2.2 **硬件级优化策略**
针对上述瓶颈，我实施了以下底层优化：
- **OpenMP 并行计算：** 对力累加和约束投影进行了多线程化，特别是在风力计算中利用原子操作保证线程安全，显著提升了多核 CPU 的吞吐量。
- **有序连续数据结构 (Sorted Continuum)：** 将传统的 `std::unordered_map` 替换为紧凑的 `std::vector`。通过对哈希值进行排序，使空间邻近的质点在内存地址上也尽可能相邻，极大提高了 **CPU 缓存命中率 (Cache Locality)**。

**优化成果：总体性能提升 16.3%**。其中风力计算由于高度并行化，性能提升达 **38.9%**。

---

### 6.3 XPBD 算法集成 (能量守恒优化)

传统的 Verlet 积分在 \(k_s\) 极大或风力过强时容易出现数值爆炸（Explode）。为此，我集成了工业界主流的 **XPBD (Extended Position Based Dynamics)** 算法。

<div class="flex flex-row gap-4">
    <div class="flex-1">
        {{< figure src="spring_high_ks.png" caption="**图 2:** Verlet 积分在高 \(k_s\) 下崩溃" alt="Result 1" >}}
    </div>
    <div class="flex-1">
        {{< figure src="xpbd_high_ks.png" caption="**图 3:** XPBD 在相同参数下依然稳定" alt="Result 2" >}}
    </div>
</div>

#### 6.3.1 XPBD 核心原理
XPBD 将约束视为几何目标而非单纯的力。对于距离约束 \(C = \|\mathbf{p}_1 - \mathbf{p}_2\| - L\)，其位置修正量 \(\Delta \lambda\) 计算如下：
$$\Delta \lambda = \frac{-C - \tilde{\alpha} \lambda}{\sum w_i + \tilde{\alpha}}$$
其中 \(\tilde{\alpha} = \alpha / \Delta t^2\)。这种方法保证了即使在高密度网格下，布料也不会表现得像“拉面”一样过度拉伸。

#### 6.3.2 深度优化：基于“图着色”的并行化 (Graph Coloring)
XPBD 最大的瓶颈在于迭代约束投影，直接并行会导致数据竞争（Data Race）。
- **解决方案：** 我实现了**图着色算法**，将互相独立的弹簧约束分为不同的“颜色批次（Color Batches）”。
- **执行逻辑：** 在同一个批次内，没有任何两个弹簧共享同一个质点。这使得我们可以安全地使用 OpenMP 进行并行加速，无需加锁。

**并行化成果：** 约束求解阶段实现了 **2.77x** 的加速，带动模拟单步总耗时降低了 **47.3%**，使 \(128 \times 128\) 高密度网格的实时仿真成为可能。

---

## 第七部分：项目总结与展望

本项目不仅是布料仿真的练习，更是一次对**实时物理工程约束**的深度探索。从基础的 Verlet 积分进化到并行化的 XPBD 求解器，我深刻理解了物理精度与硬件效率之间的权衡。

### 核心亮点总结
- **卓越的性能飞跃：** 通过图着色并行化实现了约束求解 **2.77x** 的加速。
- **极致的稳定性：** XPBD 的集成解决了高刚度系数下的数值爆炸问题。
- **底层优化思维：** 利用空间哈希排序优化内存布局，展示了对缓存友好型代码的理解。

### 未来改进方向
1. **架构演进 (AoS to SoA)：** 目前采用结构体数组 (AoS)，未来计划迁移至数组化的结构体 (SoA)，以更好地利用 **SIMD (AVX/SSE)** 指令集进行向量化加速。
2. **GPU 算力迁移：** 受限于交换期间的硬件环境，目前主要在 CPU 侧优化。回国后我计划在 **RTX 5070 Ti** 平台上利用 **CUDA** 重构自碰撞检测，预计能将毫秒级的耗时降至微秒级。
3. **前沿算法探索：** 研究隐式欧拉法（Implicit Euler）与投影动力学（Projective Dynamics），进一步探索仿真刚度与计算成本的最优解。