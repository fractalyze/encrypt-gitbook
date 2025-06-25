# Ryan's Trick for Distributed Groth16

Please read [Groth16](../snark/groth16.md) beforehand!

## Bottlenecks

These are the three most computation-intensive parts that must be distributed:

#### 1. **Witness Reduction**

This is required to compute $$a(X), b(X), c(X)$$:

$$
\sum_{i=0}^m (A_{j, i} \cdot z_i), \quad \sum_{i=0}^m (B_{j, i} \cdot z_i), \quad \sum_{i=0}^m (C_{j, i} \cdot z_i)
$$

#### 2. **FFT / Inverse FFT**

This is needed to compute $$h_i$$:

$$
\begin{aligned}
a(X) &= \sum_{j = 0}^{n-1} L_j(X) \left(\sum_{i = 0}^{m} A_{j, i} \cdot z_i\right),\quad  \bm{a} = \left(a'(\omega^0), \dots, a'(\omega^{n-1})\right) \\
b(X) &= \sum_{j = 0}^{n-1} L_j(X) \left(\sum_{i = 0}^{m} B_{j, i} \cdot z_i\right),\quad  \bm{b} = \left(b'(\omega^0), \dots, b'(\omega^{n-1})\right) \\
c(X) &= \sum_{j = 0}^{n-1} L_j(X) \left(\sum_{i = 0}^{m} C_{j, i} \cdot z_i\right),\quad  \bm{c} = \left(c'(\omega^0), \dots, c'(\omega^{n-1})\right)
\end{aligned}
$$

#### 3. **MSM**

This is used to compute $$([A]_1, [B]_2, [C]_1)$$, involving five MSMs, one of which depends on $$h_i$$:

$$
\begin{aligned}
&\sum_{i=0}^m z_i [a_i(x)]_1, \quad \sum_{i=0}^m z_i [b_i(x)]_1, \quad \sum_{i=0}^m z_i [b_i(x)]_2, \\
&\sum_{i = \ell + 1}^{m} z_i \left[ \frac{\beta  a_i(x) + \alpha  b_i(x) + c_i(x)}{\delta} \right]_1, \\
&\sum_{i = 0}^{n - 2} h_i \left[ \frac{{L'}_{2i + 1}(x)}{\delta} \right]_1
\end{aligned}
$$

## Partial Witness Reduction + Partial Inverse FFT

### Motivation

The primary motivation for performing **Partial Witness Reduction** and **Partial FFT** is to **partition the data structures** $$A, B, C, \bm{z}$$ across ddd devices. By doing so, each device only stores a fraction of the full data, significantly **reducing memory overhead** and enabling large-scale proving even when the entire witness cannot fit into a single GPU’s memory.

In **Partial Witness Reduction** and **Partial Inverse FFT**, the matrices $$A, B, C$$ are typically sparse, making it difficult to precisely estimate the memory savings from their partitioning. However, the vector $$\bm{z}$$ is dense and evenly split across ddd devices, so its memory overhead is **reduced by a factor of** $$d$$.

### Protocol

Assuming $$m + 1$$ is divisible by $$d$$ (the number of devices), the polynomials can be decomposed as follows:

$$
\begin{aligned}
a(X) &= \sum_{k = 0}^{d-1} \underbrace{\left(\sum_{j = 0}^{n-1} L_j(X) \left(\sum_{i = k(m + 1) / d}^{(k + 1)(m + 1) / d} A_{j, i} \cdot z_i\right)\right)}_{a^{(k)}(X)} \\
b(X) &= \sum_{k = 0}^{d-1} \underbrace{\left(\sum_{j = 0}^{n-1} L_j(X) \left(\sum_{i = k(m + 1) / d}^{(k + 1)(m + 1) / d} B_{j, i} \cdot z_i\right)\right)}_{b^{(k)}(X)} \\
c(X) &= \sum_{k = 0}^{d-1} \underbrace{\left(\sum_{j = 0}^{n-1} L_j(X) \left(\sum_{i = k(m + 1) / d}^{(k + 1)(m + 1) / d} C_{j, i} \cdot z_i\right)\right)}_{c^{(k)}(X)}
\end{aligned}
$$

Thus, matrices and vectors can be partitioned into $$d$$ parts:

* $$A \to (A_k)_{k=0}^{d-1} \in \left(\mathbb{F}^{n \times \frac{m+1}{d}}\right)^d$$
* $$B \to (B_k)_{k=0}^{d-1} \in \left(\mathbb{F}^{n \times \frac{m+1}{d}}\right)^d$$
* $$C \to (C_k)_{k=0}^{d-1} \in \left(\mathbb{F}^{n \times \frac{m+1}{d}}\right)^d$$
* $$\bm{z} \to (z_k)_{k=0}^{d-1} \in \left(\mathbb{F}^{\frac{m + 1}{d}}\right)^d$$

### Examples

Suppose we are given the following matrices $$A, B, C$$, and vector $$\bm{z}$$:

$$
A = \begin{bmatrix}
A_{0, 0} & A_{0, 1} \\
A_{1, 0} & A_{1, 1} \\
A_{2, 0} & A_{2, 1} \\
A_{3, 0} & A_{3, 1} \\
\end{bmatrix}, \quad
B = \begin{bmatrix}
B_{0, 0} & B_{0, 1} \\
B_{1, 0} & B_{1, 1} \\
B_{2, 0} & B_{2, 1} \\
B_{3, 0} & B_{3, 1} \\
\end{bmatrix}, \quad
C = \begin{bmatrix}
C_{0, 0} & C_{0, 1} \\
C_{1, 0} & C_{1, 1} \\
C_{2, 0} & C_{2, 1} \\
C_{3, 0} & C_{3, 1} \\
\end{bmatrix}, \quad \bm{z} = (z_0, z_1)
$$

Then the polynomials $$a(X), b(X), c(X)$$ are constructed as follows:

$$
\begin{align*}
a(X) &= L_0(X)(A_{0, 0}z_0 + A_{0, 1}z_1) + L_1(X)(A_{1, 0}z_0 + A_{1, 1}z_1) + L_2(X)(A_{2, 0}z_0 + A_{2, 1}z_1) + L_3(X)(A_{3, 0}z_0 + A_{3, 1}z_1) \\
&= \underbrace{L_0(X)(A_{0, 0}z_0) + L_1(X)(A_{1, 0}z_0) + L_2(X)(A_{2, 0}z_0) + L_3(X)(A_{3, 0}z_0)}_{a^{(0)(X)}} \\
&\quad + \underbrace{L_0(X)(A_{0, 1}z_1) + L_1(X)(A_{1, 1}z_1) + L_2(X)(A_{2, 1}z_1) + L_3(X)(A_{3, 1}z_1)}_{a^{{(1)(X)}}} \\
b(X) &= L_0(X)(B_{0, 0}z_0 + B_{0, 1}z_1) + L_1(X)(B_{1, 0}z_0 + B_{1, 1}z_1) + L_2(X)(B_{2, 0}z_0 + B_{2, 1}z_1) + L_3(X)(B_{3, 0}z_0 + B_{3, 1}z_1) \\
&= \underbrace{L_0(X)(B_{0, 0}z_0) + L_1(X)(B_{1, 0}z_0) + L_2(X)(B_{2, 0}z_0) + L_3(X)(B_{3, 0}z_0)}_{b^{(0)(X)}} \\
&\quad + \underbrace{L_0(X)(B_{0, 1}z_1) + L_1(X)(B_{1, 1}z_1) + L_2(X)(B_{2, 1}z_1) + L_3(X)(B_{3, 1}z_1)}_{b^{{(1)(X)}}} \\
c(X) &= L_0(X)(C_{0, 0}z_0 + C_{0, 1}z_1) + L_1(X)(C_{1, 0}z_0 + C_{1, 1}z_1) + L_2(X)(C_{2, 0}z_0 + C_{2, 1}z_1) + L_3(X)(C_{3, 0}z_0 + C_{3, 1}z_1) \\
&= \underbrace{L_0(X)(C_{0, 0}z_0) + L_1(X)(C_{1, 0}z_0) + L_2(X)(C_{2, 0}z_0) + L_3(X)(C_{3, 0}z_0)}_{c^{(0)(X)}} \\
&\quad + \underbrace{L_0(X)(C_{0, 1}z_1) + L_1(X)(C_{1, 1}z_1) + L_2(X)(C_{2, 1}z_1) + L_3(X)(C_{3, 1}z_1)}_{c^{{(1)(X)}}} \\
\end{align*}
$$

## Full FFT

### Motivation

<figure><img src="../../.gitbook/assets/image (102).png" alt=""><figcaption></figcaption></figure>

In protocols like DIZK, distributing FFT requires 3 communications per FFT, which leads to substantial overhead. However, according to data from [ICICLE-Snark](https://medium.com/@ingonyama/icicle-snark-the-fastest-groth16-implementation-in-the-world-00901b39a21f), the dominant cost in Groth16 proving is **MSM**, not FFT.\
Therefore, we choose **not to split the FFT**, and instead perform a **Full FFT after reconstructing**. This design reduces communication complexity while focusing optimization efforts on the actual bottleneck.

### Protocol

After the partial inverse FFT, each device performs a [`Reduce`](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/usage/collectives.html#reduce) operation to reconstruct the full polynomials $$a(X), b(X), c(X)$$.&#x20;

We use `Reduce` instead of `AllReduce` for the Full FFT step in order to minimize data communication cost. As a result, a single device is responsible for performing the element-wise multiplication and the forward FFT on the fully reduced polynomials.

### Example

Suppose we have 4 devices, and each device holds polynomials $$a^{(k)}(X), b^{(k)}(X), c^{(k)}(X)$$ for $$k = 0, 1, 2, 3$$. To proceed with proof generation, every device must eventually obtain the full sums:

$$
a(X) = a^{(0)}(X) + a^{(1)}(X) + a^{(2)}(X) + a^{(3)}(X) \\
b(X) = b^{(0)}(X) + b^{(1)}(X) + b^{(2)}(X) + b^{(3)}(X) \\
c(X) = c^{(0)}(X) + c^{(1)}(X) + c^{(2)}(X) + c^{(3)}(X) \\
$$

## Partial MSM

### Motivation

Each MSM can be intuitively split across devices. When performing the 5 MSMs involved in Groth16 proving, both the scalar inputs $$z_i$$​ and $$h_i$$ are partitioned across ddd devices, reducing their memory footprint by a factor of $$d$$. As a result, the corresponding **base points** used in each MSM are also reduced proportionally, leading to a significant decrease in **per-device memory usage** and enabling more efficient multi-GPU computation.

### Protocol

Assuming $$n - 1$$ is divisible by $$d$$:

$$
\begin{aligned}
&\sum_{i=0}^m z_i [a_i(x)]_1 = \sum_{k=0}^{d-1} \left( \sum_{i=k(m+1)/d}^{(k+1)(m+1)/d} z_i [a_i(x)]_1 \right) \\
&\sum_{i=0}^m z_i [b_i(x)]_1 = \sum_{k=0}^{d-1} \left( \sum_{i=k(m+1)/d}^{(k+1)(m+1)/d} z_i [b_i(x)]_1 \right) \\
&\sum_{i=0}^m z_i [b_i(x)]_2 = \sum_{k=0}^{d-1} \left( \sum_{i=k(m+1)/d}^{(k+1)(m+1)/d} z_i [b_i(x)]_2 \right) \\
&\sum_{i = \ell + 1}^{m} z_i \left[ \frac{\beta  a_i(x) + \alpha  b_i(x) + c_i(x)}{\delta} \right]_1 = \sum_{k=0}^{d-1} \left( \sum_{i= \max(k(m+1)/d, \ell + 1)}^{(k+1)(m+1)/d} \cdots \right) \\
&\sum_{i = 0}^{n - 2} h_i \left[ \frac{{L'}_{2i + 1}(x)}{\delta} \right]_1 = \sum_{k=0}^{d-1} \left( \sum_{i=k(n-1)/d}^{(k+1)(n-1)/d} \cdots \right) \\
\end{aligned}
$$

The first four MSMs can be computed independently. The last one requires a precomputed $$h_i$$, which each device can hold after the [`Send`](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/api/p2p.html#ncclsend) & [`Recv`](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/api/p2p.html#ncclsend) operations.

### Example

```cpp
const int field_element_size = 32; // 32 bytes per BN254 field element
const int num_coeffs_per_gpu = ...; // e.g., 2^23 / num_gpus
const size_t chunk_size = num_coeffs_per_gpu * field_element_size;

if (rank == 0) {
    // `full_poly_data` points to the full polynomial coefficients (uint8_t*)
    for (int i = 0; i < num_gpus; ++i) {
        if (i ! = rank) {
            uint8_t* chunk_ptr = full_poly_data + i * chunk_size;
            ncclSend(chunk_ptr, chunk_size, ncclUint8, i, comm, stream);
        }
    }
} else {
    // `my_poly_chunk` will hold this GPU's assigned coefficients
    ncclRecv(my_poly_chunk, chunk_size, ncclUint8, 0, comm, stream);
}
```

## Put Together

In the following expressions, red highlights indicate the parts that must be reduced across devices. The remaining parts can be computed on the host to finalize the Groth16 proof:

$$
\begin{aligned}
[A]_1 &= [\alpha]_1 + \textcolor{red}{\underbrace{\sum_{i=0}^m z_i [a_i(x)]_1}_{A}} + r [\delta]_1 \\
[B]_2 &= [\beta]_2 + \textcolor{red}{\underbrace{\sum_{i=0}^m z_i [b_i(x)]_2}_{B}} + s [\delta]_2 \\
[C]_1 &= \textcolor{red}{\underbrace{\sum_{i = \ell + 1}^{m} z_i \left[ \frac{\beta  a_i(x) + \alpha  b_i(x) + c_i(x)}{\delta} \right]_1}_{C_1} + \underbrace{\sum_{i = 0}^{n - 2} h_i \left[ \frac{{L'}_{2i + 1}(x)}{\delta} \right]_1}_{C_2}} + s[A]_1 + r\textcolor{red}{\underbrace{[B]_1}_{C_3}} - rs[\delta]_1
\end{aligned}
$$

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-22 at 5.57.37 PM.png" alt=""><figcaption></figcaption></figure>

In protocols like [DIZK](dizk.md), distributing FFT requires many communications per FFT. In contrast, our protocol incurs less communications only during the `Reduce`, `Send` and `Recv` steps, while the subsequent `AllReduce` step involves only a constant number of group elements. As a result, the total communication cost is significantly lower than that of DIZK. Moreover, this distribution strategy can be efficiently implemented using [SPMD](https://arxiv.org/abs/2105.04663).

| Communication Step | Communication Cost                             |
| ------------------ | ---------------------------------------------- |
| `Reduce`           | $$3 \times (d - 1) \times n$$                  |
| `Send` & `Recv`    | $$(d - 1) \times \frac{n}{d}$$                 |
| `AllReduce`        | $$O(d)$$                                       |
| DIZK's `AllToAll`  | $$3 \times 3 \times 2\times (d - 1) \times n$$ |

## Analysis: RISC0 Groth16 proof

Currently, the [stark\_verify.circom](https://github.com/risc0/risc0/blob/4ac16240c8e76583ee6a68a3b71a21c82f37313c/groth16_proof/groth16/stark_verify.circom) used in RISC0 has the following characteristics:

```bash
> circom --r1cs stark_verify.circom 
template instances: 349
non-linear constraints: 5676573
linear constraints: 0
public inputs: 0
private inputs: 25749 (22333 belong to witness)
public outputs: 5
wires: 5635930
labels: 10298490
Written successfully: ./stark_verify.r1cs
Everything went okay
```

The number of rows in the $$A, B, C$$ matrices is **5,676,573**, which means an SRS of size $$2^{23}$$ is required. The number of columns is **5,635,930**.

## Reduce Analysis

We now analyze the cost of the first and most expensive communication step in our distributed system: the `Reduce` operation.

Assume 4 GPUs each compute and store partial results of the polynomials $$a^{(k)}(X), b^{(k)}(X), c^{(k)}(X)$$, and together, they must aggregate them into full polynomials of size $$2^{23}$$ over the BN254 field. The total volume of data involved is:

* **3 polynomials per GPU × 256 MB each to be sent to the leader GPU = 768 MB per GPU**
* **Total communication volume on receiver side of leader GPU = 3 GPUs × 768 MB = 2304 MB**

The goal of `Reduce` is to **sum these partial polynomials in the leader device**, so that 1 leader GPU retains the complete 768 MB result (i.e., full $$a(X), b(X), c(X)$$).

### Computation Cost

Field additions in BN254 are extremely lightweight on GPUs. Each operation for $$a_i(X), b_i(X), c_i(X)$$ takes less than 1 ms and is negligible in the overall runtime.

### Communication Cost

#### **PCIe Transfer Speed**

| Item                           | Value                                       |
| ------------------------------ | ------------------------------------------- |
| Interface                      | PCIe Gen4 x16                               |
| Theoretical Bandwidth          | 32 GB/s                                     |
| Measured Bandwidth (via NCCL)  | 24 \~ 28 GB/s                               |
| Receive time by the leader GPU | 2304 MB / (24 \~ 28) GB/s ≈ **82 \~ 96 ms** |
| Estimated with NCCL overhead   | **92 \~ 106 ms**                            |

<figure><img src="../../.gitbook/assets/image (103).png" alt=""><figcaption><p>Source: <a href="https://www.marvell.com/content/dam/marvell/en/blogs/2024/01/PCIe-Gen6-IO-Bandwidth.png">https://www.marvell.com/content/dam/marvell/en/blogs/2024/01/PCIe-Gen6-IO-Bandwidth.png</a></p></figcaption></figure>

We chose **PCIe Gen4 x16** because it offers a well-balanced trade-off between **performance, cost, and ecosystem stability**. Gen4 provides up to **32 GB/s bidirectional bandwidth**, which is sufficient for most real-world proving workloads, especially when combined with smart overlapping strategies between computation and communication.

While **bandwidth can become a performance bottleneck** in some high-throughput scenarios, upgrading to newer generations like **PCIe Gen5 or Gen6** introduces significant trade-offs in **cost, complexity, and platform requirements**. For now, Gen4 remains the **most practical and widely supported option**, but we remain open to adopting higher PCIe generations if communication overhead proves to be a critical limiting factor.

#### **Performance Summary**

| Task                          | Time             |
| ----------------------------- | ---------------- |
| Field addition                | < 3 ms           |
| PCIe communication (`Reduce`) | **92 \~ 106 ms** |
| **Total Execution Time**      | **95 \~ 109 ms** |

According to ICICLE-Snark, a polynomial of degree $$2^{23}$$ takes approximately **774 ms** for MSM. With four devices, each handling a degree $$2^{21}$$ polynomial, the per-device MSM time is around **193 ms**. If we overlap the `Reduce` step with two of these MSMs, the communication overhead can be effectively hidden.

## Estimation

Let’s perform a simple estimation based on the data from [**ICICLE-Snark**](https://medium.com/@ingonyama/icicle-snark-the-fastest-groth16-implementation-in-the-world-00901b39a21f).

### **Assumptions**

1. $$n$$ and $$m$$ are $$2^{22}$$
2. The runtime for MSM over $$\mathbb{G}_2$$ is the same as that for $$\mathbb{G}_1$$.
3. The runtime for $$\mathbb{G}_1$$ MSM scales linearly with the degree.
4. The total proving time for Groth16 is the sum of the time taken for 5 MSMs (387 ms), 1 FFT ( 10 ms), and 1 IFFT (10 ms).
5. The time for `Reduce`, `Recv` and `Send` is negligible.

### Computation

If everything is computed **serially**, the total time is:

$$
5 \times 387 + 10 + 10 = 1955\ \text{ms}
$$

If we instead use the proposed scheme across **4 GPUs**, the time becomes:

$$
5 \times (387 / 4) + 10 + 10 \approx 504\ \text{ms}
$$

This shows that the proving time is reduced by approximately a **factor of 4**.

### Input Size

We do not include $$A, B, C$$ in our input size estimation, as their sparsity makes it difficult to quantify preciesly. However, they will also contribute to reducing the overall memory requirement.

If everything is computed on a single device, the input size is around $$1664$$ MB. Each component will consume memory as follows:

* Witness vector $$\bm{z}$$: $$2^{22} \times 32$$ B $$= 128$$ MB
* MSM base point $$\left([a_i(x)]_1\right)_{i = 0}^{m}$$: $$2^{22} \times 64$$ B $$= 256$$ MB
* MSM base point $$\left([b_i(x)]_1\right)_{i = 0}^{m}$$: $$2^{22} \times 64$$ B $$= 256$$ MB
* MSM base point $$\left([b_i(x)]_2\right)_{i = 0}^{m}$$: $$2^{22} \times 128$$ B $$= 512$$ MB
* MSM base point $$\left(\left[ \frac{\beta  a_i(x) + \alpha  b_i(x) + c_i(x)}{\delta} \right]_1\right)_{i = \ell + 1}^{m}$$: $$2^{22} \times 64$$ B $$\approx 256$$ MB
* MSM base point $$\left(\left[ \frac{{L'}_{2i + 1}(x)}{\delta} \right]_1\right)_{i = 0}^{n-2}$$: $$2^{22} \times 64$$ B $$\approx 256$$ MB&#x20;

If we instead use the proposed scheme across **4 GPUs**, the input size becomes around $$416$$ MB.

### Runtime Memory Size

{% hint style="warning" %}
Here, we estimate only the memory required for MSM itself, **excluding the additional memory needed for buckets in the** [**Pippenger**](../../primitives/abstract-algebra/elliptic-curve/msm/pippengers-algorithm.md) **algorithm**.
{% endhint %}

If everything is computed on a single device, and intermediate memory is released immediately after use, the main bottleneck becomes the **MSM in** $$\mathbb{G}_2$$, which requires $$128 + 512 = 640$$ MB of memory.

However, with the proposed scheme using **4 GPUs**, this memory requirement is reduced to $$32 + 128 = 160$$ MB. The **Full FFT**, including twiddle factors, requires an additional $$256$$ MB. Therefore, under this setup, **the total memory required per device is approximately** $$256$$ MB.

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
