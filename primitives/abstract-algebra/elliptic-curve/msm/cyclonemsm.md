---
description: 'Presentation: https://youtu.be/djA3mzn7BPg'
---

# CycloneMSM

## Introduction

**CycloneMSM** is an FPGA-accelerated architecture for Multi-Scalar Multiplication (MSM) over the BLS12-377 elliptic curve, developed by [Jump Trading](https://www.jumptrading.com/) and [Jump Crypto](https://jumpcrypto.com/). Optimized for the case of a **fixed base point**, this implementation achieves **high-performance execution**, completing 4 million-point MSMs in under one second and 64 million-point MSMs in approximately six seconds—delivering over **6× speedup** compared to other state-of-the-art software. At its core, CycloneMSM leverages the **bucket method of the Pippenger algorithm**, and applies aggressive **pipeline optimization and scheduling techniques** to perform curve operations without bottlenecks.

## Background

### FPGA

To dramatically accelerate MSM computation, **CycloneMSM leverages Field Programmable Gate Arrays (FPGAs)**. Unlike CPUs or GPUs, FPGAs offer a fundamentally different computation model, making them particularly well-suited for exploiting the **parallelism and pipeline structure** inherent in MSM.

#### Comparison: CPU vs GPU vs FPGA

<table><thead><tr><th width="112.8134765625">Architecture</th><th width="442.5908203125">Characteristics</th><th>Suitability for MSM</th></tr></thead><tbody><tr><td><strong>CPU</strong></td><td>General-purpose processor; optimized for sequential tasks</td><td>❌ Too slow</td></tr><tr><td><strong>GPU</strong></td><td>Designed for parallel computation with many execution units</td><td>⭕ Reasonable performance</td></tr><tr><td><strong>FPGA</strong></td><td>Custom logic design and deep pipelining capabilities</td><td>✅ Ideal choice</td></tr></tbody></table>

* **CPUs** excel at complex control flow and branching, but are inefficient for highly repetitive operations like point additions and multiplications.
* **GPUs** offer strong parallelism but struggle with **fine-grained scheduling and memory access control**.
* **FPGAs**, on the other hand, allow direct **circuit-level customization**, enabling **deeply pipelined designs** that sustain a continuous flow of operations with minimal control overhead.

### Coordinate Systems

MSM involves millions of point additions, so the **efficiency of the addition operation** plays a crucial role in overall performance. In particular, [**affine coordinates**](../weierstrass-curve/coordinate-forms.md#affine) require an inverse operation per addition, which can significantly increase the total computation cost. (Reference: [Explicit Formulas Database](https://www.hyperelliptic.org/EFD))

<figure><img src="../../../../.gitbook/assets/image (125).png" alt=""><figcaption></figcaption></figure>

While [**Extended Jacobian coordinates**](../weierstrass-curve/coordinate-forms.md#xyzz-extended-jacobian) for [**Weierstrass Curves**](../weierstrass-curve/) enable inverse-free operations, the formulas used depend on the context (e.g., addition vs doubling). This results in **variable computational cost**, making it difficult to pipeline efficiently in hardware.

In contrast, [**Extended Projective coordinates**](../edwards-curve/coordinate-forms.md#extended-projective) for [**Twisted Edwards**](../edwards-curve/) form of elliptic curves support both addition and doubling with a **single unified formula**, resulting in **constant computational cost**. This property makes them highly suitable for **hardware pipeline design**.

Accordingly, CycloneMSM adopts the following coordinate system strategy:

* **Software implementation**: Uses **Extended Jacobian coordinates** for inverse-free operations
* **FPGA hardware implementation**: Uses **Extended projective coordinates** for pipelining optimization

This coordinate system selection is a key architectural decision aimed at maximizing computational efficiency in each environment.

### Pippenger Algorithm

The [**Pippenger algorithm**](pippengers-algorithm.md) is a classic technique for efficiently performing MSM (Multi-Scalar Multiplication). Given $$N$$ pairs of (point, scalar) and a window size $$c$$, each window $$W$$ requires the following operations:

* **Bucket Accumulation**: $$N \times \mathsf{MixedAdd}$$
* **Bucket Reduction**: $$2^c \times \mathsf{Add}$$
* **Window Reduction**: $$c \times \mathsf{Double} + 1 \times \mathsf{Add}$$

Among these, **Bucket Accumulation accounts for the majority of the total computation**. Therefore, optimizing this stage is critical for improving overall MSM performance.

## Protocol Explanation

### Scheduler: Optimizing Operation Order to Avoid Pipeline Conflicts

CycloneMSM is designed around an **FPGA-based architecture**, enabling **fully pipelined execution** of computations. In particular, the **Bucket Accumulation phase**, which dominates MSM computation, is highly accelerated by this pipelining structure.

#### How the Pipeline Works

The **Curve Adder**, a core computational unit of CycloneMSM, is a pipelined structure designed to efficiently perform point addition on Twisted Edwards curves. This module supports both $$\mathsf{MixedAdd}$$ and $$\mathsf{Add}$$ operations and is optimized for execution on FPGA hardware.

The Curve Adder takes $$T$$ cycles to complete a single addition. In CycloneMSM, $$T = 96$$. A new operation is fed into the pipeline every cycle, allowing multiple point additions across different buckets to run in parallel, as illustrated below:

<table data-header-hidden data-full-width="true"><thead><tr><th></th><th></th><th></th><th></th><th></th><th></th><th></th></tr></thead><tbody><tr><td></td><td><span class="math">t</span></td><td><span class="math">t + 1</span></td><td><span class="math">t + 2</span></td><td><span class="math">\dots</span></td><td><span class="math">t + {T-1}</span></td><td><span class="math">t + T</span></td></tr><tr><td><span class="math">B_{k^{(t)}}</span></td><td><span class="math">P^{(t)}</span>﻿</td><td></td><td></td><td></td><td></td><td><span class="math">P^{(t + T)}</span>﻿</td></tr><tr><td><span class="math">B_{k^{(t + 1)}}</span></td><td></td><td><span class="math">P^{(t + 1)}</span></td><td></td><td></td><td></td><td></td></tr><tr><td><span class="math">B_{k^{(t + 2)}}</span></td><td></td><td></td><td><span class="math">P^{(t + 2)}</span>﻿</td><td></td><td></td><td></td></tr><tr><td><span class="math">\vdots</span></td><td></td><td></td><td></td><td><span class="math">\ddots﻿</span></td><td></td><td></td></tr><tr><td><span class="math">B_{k^{(t + T - 1)}}</span>​﻿</td><td></td><td></td><td></td><td></td><td><span class="math">P^{(t + T - 1)}</span>﻿</td><td></td></tr></tbody></table>

The addition at each cycle is defined as:

$$
B_{k^{(t)}} \leftarrow B_{k^{(t)}} + P^{(t)}
$$

**Definition of Terms**

* $$t$$: The pipeline time step (i.e., cycle index). At each $$t$$, a new operation enters the pipeline.
* $$P^{(t)}$$: The point being added at time $$t$$.
* $$k^{(t)}$$: The reduced scalar index corresponding to point $$P^{(t)}$$. It determines the bucket.
* $$B_{k^{(t)}}$$​: The bucket that accumulates all points whose reduced scalar index is $$k^{(t)}$$.

However, for this pipeline to operate correctly, **no two additions in the pipeline should access the same bucket** $$B_k$$ **within a** $$T$$**-cycle window**. This is enforced by the following constraint:

$$
k^{(t_1)} \ne k^{(t_2)} \quad \text{ if } |t_1 - t_2| < T
$$

#### Design Goals of the Scheduler

Based on this, the Scheduler must satisfy two key objectives:

1. **Correctness**: Prevent pipeline conflicts by adhering to the constraint above.
2. **Efficiency**: Avoid excessive reordering of points; most operations should proceed in near-original order.

#### Scheduler Implementation Strategies

CycloneMSM explores two strategies to meet these goals:

1. **Delayed Scheduler**

* **Idea**: When a conflict is detected, defer processing of that point by placing it in a **FIFO queue** and retry it later.
* **Estimated conflict count**: Approximately $$\frac{NT}{2^{c-1}}$$.
* Example: With $$N = 2^{26}$$, $$c = 16$$, $$T = 100$$, about **204,800** conflicts arise (\~0.3% of points).
* **In practice**: Most points are processed successfully within 2–3 passes.
* **Advantages**: Simple architecture and easy to implement in hardware
* **Drawbacks**:
  * Requires maintaining a queue of size proportional to the number of conflicts, which can be memory-intensive
  * In worst-case scenarios (e.g., non-uniform scalars), frequent collisions could degrade performance

> CycloneMSM adopts this approach in its actual FPGA implementation.

2. **Greedy Scheduler**

* **Idea**: Attempt to process delayed points at the earliest valid time (e.g., $$t + T + 1$$), even if conflicts occurred earlier.
* **Advantages**: Queue is drained quickly, leading to **smaller queue size**
* **Drawbacks**:
  * Requires tracking last access times for all buckets, which complicates state management
  * Increased memory updates and lookups each cycle place **heavy demands on hardware resources**

> Due to memory access overhead, this strategy was **not adopted** in CycloneMSM’s hardware environment.

### **Software-Side Affine Optimization**

When implementing MSM in software, **non-Affine coordinate systems** such as **Extended Jacobian** are typically used. The main reason is that **point addition in Affine coordinates requires field inversion**, which is computationally expensive. For instance, consider a typical Bucket Accumulation where multiple points are added sequentially:

$$
P \leftarrow P_1 + P_2 + \dots + P_{n-1} + P_n
$$

If Affine coordinates were used, each addition would involve an inverse operation, leading to significant performance overhead.&#x20;

{% hint style="info" %}
[BATZORIG ZORIGOO](https://app.gitbook.com/u/lqk5Tx9zY4XYVRfF3ReRDiOhbxG3 "mention") mentioned that batch inversion can be applied using a tree-structured addition approach, which reduces the number of operations to $$\log_2 n \times \mathsf{Inv}$$
{% endhint %}

#### Scheduler Enables the Use of Affine Coordinates

However, thanks to the **Scheduler** introduced in CycloneMSM, points can now be grouped into batches for addition **without conflicts between buckets**:

$$
B_{k^{(t)}} \leftarrow B_{k^{(t)}} + P^{(t)} \\
B_{k^{(t + 1)}} \leftarrow B_{k^{(t + 1)}} + P^{(t + 1)} \\
\vdots \\
B_{k^{(t + T - 1)}} \leftarrow B_{k^{(t + T - 1)}} + P^{(t + T - 1)} \\
$$

This structure allows for **batch addition of** $$T$$ **Affine points**, making it possible to apply **batch inversion** and significantly reduce the overall computation cost.

#### Comparison of Operation Costs

* Affine addition (1 time): $$3 \times \mathsf{Mul} + 1 \times \mathsf{Inv}$$
* $$T$$ Affine additions (naive): $$3T \times \mathsf{Mul} + T \times \mathsf{Inv}$$
* $$T$$ Affine additions (with batch inversion): $$6T \times \mathsf{Mul} + 1 \times \mathsf{Inv}$$
* $$T$$ Extended Joaobian addtions: $$10T \times \mathsf{Mul}$$

Thus, **the number of inversion operations is reduced from** $$T$$ **to 1**, while the rest is replaced by multiplications, leading to a dramatic boost in efficiency. If $$4T \times \mathsf{Mul} > 1 \times \mathsf{Inv}$$, then $$T$$ Affine addition (with batch inversion) is more efficient.

#### Real-World Implementations

This Affine optimization has been adopted in actual open-source libraries:

* [gnark-crypto #261](https://github.com/Consensys/gnark-crypto/pull/261)
* [halo2curves #130](https://github.com/privacy-scaling-explorations/halo2curves/pull/130)

Thanks to this optimization, a **10–20% performance improvement** over Extended Jacobian-based implementations has been reported, significantly enhancing MSM computation efficiency in practice.

### FPGA Design

#### Field Arithmetic

<figure><img src="../../../../.gitbook/assets/image (126).png" alt=""><figcaption></figcaption></figure>

Arithmetic in the finite field $$\mathbb{F}_q$$ is performed using 377-bit operations in [**Montgomery representation**](../../../modular-arithmetic/modular-reduction/montgomery-reduction.md#the-montgomery-representation), with a Montgomery parameter of $$R = 2^{384}$$. Field multiplication is implemented using **three 384-bit integer multiplications** based on the [**Karatsuba algorithm**.](../../extension-field/multiplication/karatsuba-multiplication.md)

Additionally, since $$R \ge 4q$$, the system can safely operate on inputs $$a, b \in [0, 2q)$$ for multiplication. This characteristic allows certain modular reductions to be skipped when implementing the Curve Adder module:

1. **When** $$a, b \in [0, q)$$, the sum $$a + b \in [0, 2q)$$ → **modular reduction is unnecessary**
2. **To handle subtraction safely**, compute $$q + a - b \in [0, 2q)$$ → **modular reduction is also unnecessary**

By avoiding modular reductions in these cases, resource usage and processing delays in the pipeline are further minimized.

#### Constant Multiplier

<figure><img src="../../../../.gitbook/assets/image (127).png" alt=""><figcaption></figcaption></figure>

In Montgomery representation, two constant multiplications are required as follows:

$$
a \times (R^2 \mod q) \times (R^{-1} \mod q)
$$

Instead of using the standard double-and-add approach for multiplying constants, **CycloneMSM employs NAF (Non-Adjacent Form)**. This reduces the **Hamming weight** of the constant from $$\frac{1}{2}$$ to approximately $$\frac{1}{3}$$, resulting in several advantages:

* Fewer additions in the double-and-add process → **improved speed**
* Shallower and faster **adder trees** in the FPGA design
  * Specifically, **adder tree depth is reduced from** $$\log_2 W(b)$$ **to** $$\log_3 W(b)$$, where $$W(b)$$ denotes the Hamming weight of the constant $$b$$.

This optimization plays a critical role in improving both performance and resource efficiency in the CycloneMSM hardware pipeline.

#### Curve Arithmetic

<figure><img src="../../../../.gitbook/assets/Screenshot 2025-04-17 at 1.28.31 PM.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/Screenshot 2025-04-17 at 1.44.52 PM.png" alt=""><figcaption></figcaption></figure>

**Pipelined Structure**

* **Clock speed**: 250 MHz → 1 cycle = 4 ns
* **Cycle latency**:
  * $$\mathsf{MixedAdd}$$ outputs are produced **after 96 cycles (= 384 ns)**
  * $$\mathsf{mul}$$ outputs are produced **after 39 cycles (= 156 ns)**

**MixedAdd Operation (Refer to the left side of Figure 4)**

<figure><img src="../../../../.gitbook/assets/image (129).png" alt=""><figcaption></figcaption></figure>

In Twisted Edwards curves, $$\mathsf{MixedAdd}$$ uses the following two inputs:

* $$P_1 = (X_1 : Y_1 : Z_1 : T_1)$$ (extended projective)
* $$P_2 = (x_2, y_2, u_2 = kx_2y_2)$$ (extended affine)

This operation involves **7 field multiplications**, and it is optimized using formulas from \[[HWCD08, Sec. 4.2](https://eprint.iacr.org/2008/522.pdf#page=8)]. Due to its pipelined nature, the adder can accept new points every cycle.

**Full Add Operation (Refer to the right side of Figure 4)**

A general $$\mathsf{Add}$$ operation requires **9 multiplications**. CycloneMSM implements this as a **2-cycle pipelined process**. The processing flow is as follows:

1.  **Stage 0**

    Inputs: $$Z_1$$, $$Z_2$$ → passed to the multiplier
2.  **Stage 1**

    Inputs: $$T_2$$, curve constant $$k$$ → passed to the multiplier
3.  **Stage 40 (After the stages required for a single Montgomery Multiplier)**

    Invoke MixedAdd with outputs below:

$$
\mathsf{MixedAdd}\left(P_1 = (X_1 : Y_1 : r_0 : T_1),\ P_2 = (X_2, Y_2, r_1)\right)
$$

where $$r_0 = Z_1 \cdot Z_2$$ and $$r_1 = k \cdot T_2$$

This design converts a full Add into a MixedAdd call, effectively **reusing the existing pipeline** while precomputing necessary multiplications to maintain efficiency.

#### MSM Acceleration

<figure><img src="../../../../.gitbook/assets/image (130).png" alt=""><figcaption><p>They use <span class="math">S_k</span> for bucket instead of <span class="math">B_k</span>.</p></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (131).png" alt=""><figcaption></figcaption></figure>

The MSM operation in CycloneMSM is executed through coordinated interaction between the FPGA and the host, following the flow of functions outlined below:

* `MSM_INIT()`:
  * The user provides $$N$$ points in **Weierstrass affine coordinates**
  * [The host converts these points to the **Twisted Edwards coordinate system**](../edwards-curve/twisted-edwards-short-weierstrass-transformation.md)
    * Typically affine → extended affine: $$(x, y) \rightarrow (x, y, u = kxy)$$
  * The converted points are transferred to the FPGA and stored in **DDR memory**
  * Since the conversion involves inversion, **batch inversion** is used to optimize performance
* `MSM()`:
  * The host splits each scalar into **16-bit windows (**$$c = 16$$**)** using `PREPROCESS()`
    * A total of $$W = 16$$ windows are created
  * For each window, the following functions are executed:
    1. `ACCUMULATE(j)`: performs bucket accumulation
    2. `AGGREGATE(j)`: performs bucket reduction
  * The final result of all windows is computed by the host via **Window Reduction**
* `ACCUMULATE(j)`:
  * The host invokes `FPGA.ACCUMULATE(j)`
  * Scalars are reduced to **16-bit reduced scalars** and streamed to the FPGA
    * Data is batched and transmitted in 64-bit chunks (8 scalars per batch)
* `FPGA.ACCUMULATE(j)`:
  * The FPGA invokes `SCHED()` to read points from DDR memory
    * If a conflict is detected, the scalar and point are stored back into DDR via FIFO
    * Otherwise, the addition is performed and the result is stored in the corresponding bucket in SRAM
* `AGGREGATE()`:
  * The host invokes `FPGA.AGGREGATE()`
* `FPGA.AGGREGATE()`:
  *   The FPGA accumulates $$2^{c-1}$$ buckets in a sequential manner using the following formula:

      $$R = B_K + (B_K + B_{K-1}) + \dots + (B_1 + \dots + B_K)$$
  * This operation is performed using a **2-cycle** $$\mathsf{Add}$$ **pipeline** implemented on the FPGA

## Conclusion

<figure><img src="../../../../.gitbook/assets/image (133).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (132).png" alt=""><figcaption></figcaption></figure>

CycloneMSM leverages the architectural strengths of FPGAs and the inherent parallelism of MSM operations to deliver a high-performance system that significantly outpaces traditional software-based MSM implementations. The system's achievements are underpinned by the following design philosophies and optimization strategies:

*   **Optimized Coordinate System Selection**:

    On hardware, the Twisted Edwards coordinate system was chosen for its uniform operation count, which maximizes pipelining efficiency. On the software side, performance was improved through affine coordinates combined with batch inversion.
*   **Pipelined Curve Adder**:

    A 96-stage pipeline running at 250MHz accepts new inputs every 4ns, enabling the core operation of MSM—$$\mathsf{MixedAdd}$$—to be processed in every clock cycle.
*   **Conflict-Free Scheduling**:

    By applying a Delayed Scheduler, most points were processed without collisions. This also enabled batch inversion in the affine coordinate system, contributing to performance gains on the software side.
*   **Modular and Parallel MSM Flow**:

    The MSM process—initialization, bucket accumulation, reduction, and window reduction—was modularized and optimized, with roles distributed efficiently between the FPGA and host. This maximized pipeline utilization across the system.
*   **Superior Performance and Energy Efficiency**:

    CycloneMSM completes MSM on 64M points in under 6 seconds, achieving **4–6× speedups** compared to software, while consuming **only one-tenth the power**.

In summary, CycloneMSM is more than just a hardware implementation—it is a **deeply integrated system combining algorithmic and architectural design**, demonstrating the real potential of hardware-accelerated MSM for more complex ZK and large-scale proof systems in the future.

## References

* [https://www.youtube.com/watch?v=aZ3CzKZBK38](https://www.youtube.com/watch?v=aZ3CzKZBK38)
* [https://eprint.iacr.org/2022/1396.pdf](https://eprint.iacr.org/2022/1396.pdf)

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze
