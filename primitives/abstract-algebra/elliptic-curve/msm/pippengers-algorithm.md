---
description: Reduction of multi-scalar multiplication (MSM) into double and add operations
---

# Pippenger's Algorithm

> AKA "Bucket" method

## Motivation

Improve upon [Double-and-add](../scalar-multiplication/double-and-add.md) by reducing the total number of double and add operations needed for [MSM](./).&#x20;

## Algorithm Process

### Introduction

**Multi-Scalar Multiplication (MSM)** refers to the efficient computation of expressions in the following form:

$$
S = \sum_{i=1}^{n} k_i P_i
$$

* $$k_i$$: the $$i$$-th $$\lambda$$-bit scalar
* $$P_i$$: the $$i$$-th point on an elliptic curve

**Pippenger's algorithm** speeds up MSM by dividing each scalar $$k_i$$ into multiple $$s$$**-bit chunks**, and then grouping and processing scalars by their corresponding bit positions (windows).

### 1. Scalar Decomposition

First, we decompose the scalars $$k_i$$:

$$
\begin{align*}
k_i &= k_{i,1} + 2^{s}k_{i,2} + ... + 2^{(\lceil\frac{\lambda}{s}\rceil -1)s}k_{i,\lceil\frac{\lambda}{s}\rceil } \\
k_i &= \sum_{j=1}^{\lceil\frac{\lambda}{s}\rceil }2^{(j - 1)s} k_{i,j}
\end{align*}
$$

* $$s$$ : the number of bits per window
* $$\lceil\frac{\lambda}{s}\rceil$$ : the total number of windows

Through this definition, we can rewrite the original MSM summation as follows:

$$
\begin{align*}
S &= \sum_{i=1}^{n} k_i P_i \\
&= \sum_{i=1}^{n} \sum_{j=1}^{\lceil\frac{\lambda}{s}\rceil } 2^{(j-1)s}k_{i,j} P_i \\
&= \sum_{j=1}^{\lceil\frac{\lambda}{s}\rceil } 2^{(j-1)s} \left( \sum_{i=1}^{n} k_{i,j} P_i \right) \\
&= \sum_{j=1}^{\lceil\frac{\lambda}{s}\rceil } 2^{(j-1)s} W_j
\end{align*}
$$

Note that $$W_j$$ is the sum for the $$j$$-th window.

### 2. Bucket Accumulation&#x20;

Given our $$W_j$$, we denote buckets $$B$$ as the following:

$$
B_{\ell, j} = \sum_{i: k_{ij} = \ell} P_i
$$

Or in other words, $$B_{\ell, j}$$ refers to the $$\ell$$ bucket in window $$W_j$$ consisting of the summation of points $$P_i$$ with the same $$k_{i,j}$$.

Note that $$k_{i,j}$$ must be within the range $$[0, 2^w - 1]$$; however, if $$k_{i,j}$$ is 0, as $$k_{i,j}$$ is the scalar multiplier, the result of the bucket will be 0. Therefore, the practical range of $$k_{i,j}$$ is $$[1, 2^w - 1]$$.

### 3. Bucket Reduction

Thus, in total, a window $$W_j$$ will be re-structured as follows:

$$
W_j = \sum_{i=1}^{n} k_{ij}P_i = \sum_{\ell=1}^{2^s - 1} \ell B_{\ell, j}
$$

Note that the computation of a singular $$\ell B_{\ell,j}$$ is computed through a running sum of recursive summations and not through further scalar multiplications.

#### Complexity Cost:

* Original: $$(2^s - 1) \times \mathsf{ScalarMul} + (2^s - 2) \times \mathsf{PointAdd}$$
* Bucketed: $$(2^{s+1} - 2) \times \mathsf{PointAdd}$$

#### C++ Implementation

```cpp
Bucket ReduceBucketPoints(const std::vector<Bucket>& buckets) {
  Bucket res = Bucket::Zero();
  Bucket sum = Bucket::Zero();
  for (size_t i = buckets.size() - 1; i != SIZE_MAX; --i) {
    sum += buckets[i];
    res += sum;
  }
  return res;
}
```

#### Example

$$
W_1 = B_{1,1} + 2B_{2,1} + 3B_{3,1} + 4B_{4,1}
$$

```
// Round 1
sum = B_(4,1);
res = B_(4,1);
// Round 2
sum = B_(3,1) + B_(4,1);
res = B_(4,1) + [B_(3,1) + B_(4,1)];
// Round 3
sum = B_(2,1) + B_(3,1) + B_(4,1);
res = B_(4,1) + [B_(3,1) + B_(4,1)] + [B_(2,1) + B_(3,1) + B_(4,1)];
// Round 3
sum = B_(1,1) + B_(2,1) + B_(3,1) + B_(4,1);
res = B_(4,1) + [B_(3,1) + B_(4,1)] + [B_(2,1) + B_(3,1) + B_(4,1)] + [B_(1,1) + B_(2,1) + B_(3,1) + B_(4,1)];

// Total:
res = B_(1,1) + 2*B_(2,1) + 3*B_(3,1) + 4*B_(4,1);
```

### 4.  Window Reduction - Final MSM Result

With the finished individual window computations, we return to the total MSM summation and add all windows together:

$$
S= \sum_{j=1}^{\lceil\frac{\lambda}{s}\rceil } 2^{(j-1)s} W_j
$$

In practice, the computation is calculated recursively from $$j = \left\lceil \frac{\lambda}{s} \right\rceil$$ to $$j = 1$$ like so:

$$
T_j = \begin{cases}
0 & j = \left\lceil \frac{\lambda}{s} \right\rceil \\
W_j + 2 ^sT_{j+1} & j \ge 1 \\
\end{cases}
$$

#### Complexity Cost

$$
\left\lceil \frac{\lambda}{s} \right\rceil(s \times \mathsf{PointDouble} + \mathsf{PointAdd})
$$

#### C++ Implementation

```cpp
Bucket ReduceWindows(uint32_t lambda, uint32_t s, const std::vector<Bucket>& buckets) {
  Bucket res = Bucket::Zero();
  uint32_t window_num = static_cast<uint32_t>(std::ceil(lambda / s));
  for (uint32_t j = window_num - 1; j != UINT32_MAX; --i) {
    for (uint32_t i = 0; i < s; ++i) {
      res = res * 2;
    }
    res += buckets[j];
  }
  return res;
}
```

## Complexity Analysis

* To compute $$W_j$$, $$n \times \mathsf{PointAdd}$$ is required: $$\left\lceil \frac{\lambda}{s} \right\rceil(n \times \mathsf{PointAdd})$$
* Each $$W_j$$ needs $$(2^{s+1} - 2) \times \mathsf{PointAdd}$$: $$\left\lceil \frac{\lambda}{s} \right\rceil((2^{s+1} - 2) \times \mathsf{PointAdd})$$
* Final $$Q$$ needs: $$\left\lceil \frac{\lambda}{s} \right\rceil(s \times \mathsf{PointDouble} + \mathsf{PointAdd})$$

### Total complexity

$$
\left\lceil \frac{\lambda}{s} \right\rceil(s \times \mathsf{PointDouble} + (n + 2^{s+1} - 1) \times \mathsf{PointAdd}) \approx \\\lambda \times \mathsf{PointDouble} + \left\lceil \frac{\lambda}{s} \right\rceil(n + 2^{s+1} - 1)\times \mathsf{PointAdd}
$$

### Optimization Table (When , $$n = 2^{20}$$)

<table><thead><tr><th width="298.8214111328125">s</th><th>PointAdd Count</th></tr></thead><tbody><tr><td>2</td><td>134,218,624</td></tr><tr><td>4</td><td>67,110,848</td></tr><tr><td>8</td><td>33,570,784</td></tr><tr><td><mark style="color:green;">16</mark></td><td><mark style="color:green;">18,874,352</mark></td></tr><tr><td>32</td><td>68,727,865,336</td></tr><tr><td>64</td><td>147,573,952,589,680,607,228</td></tr><tr><td>128</td><td>1,361,129,467,683,753,853,853,498,429,727,074,942,974</td></tr></tbody></table>



> 🔍 This clearly shows that selecting an optimal $$s$$ value is critical for performance.

## Example

Let's try to run Pippenger's on the following simple example:

$$
\begin{align*}
S &= k_1P_1 + k_2P_2 + k_3P_3\\
&=12P_1 + 9P_2 + 13P_3
\end{align*}
$$

### 1. Scalar Decomposition

Let's define our bits per window $$s$$ to be 2. In this case, since the maximum amount of bits needed to constrain the values 12, 9, and 13 is 4, we will have $$\lceil\frac{\lambda}{s}\rceil  =\mathsf{max\_bits} / s = 4/2 = 2$$ windows.

Note that in binary we have:

$$
k_1 =12 = \underbrace{0}_{2^0}\underbrace{0}_{2^1}\underbrace{1}_{2^2}\underbrace{1}_{2^3}\\
k_2 = 9 = \underbrace{1}_{2^0}\underbrace{0}_{2^1}\underbrace{0}_{2^2}\underbrace{1}_{2^3} \\
k_3 =13 = \underbrace{1}_{2^0}\underbrace{0}_{2^1}\underbrace{1}_{2^2}\underbrace{1}_{2^3}
$$

Decomposing our $$k_0$$, $$k_1$$, and $$k_2$$ results in the following:

$$
\begin{align*}
k_i &= k_{i,1} + 2^{s}k_{i,2} + ... + 2^{(\lceil\frac{\lambda}{s}\rceil -1)s}k_{i,\lceil\frac{\lambda}{s}\rceil } \\

\end{align*}
$$

$$
\begin{align*}
\begin{array}{rlrlrl}
k_1 = 12 &= \overbrace{\underbrace{0}_{2^0}\underbrace{0}_{2^1}}^{\text{window 1}}\overbrace{\underbrace{1}_{2^2}\underbrace{1}_{2^3}}^{\text{window 2}} 
& \quad
k_2 = 9 &= \overbrace{\underbrace{1}_{2^0}\underbrace{0}_{2^1}}^{\text{window 1}}\overbrace{\underbrace{0}_{2^2}\underbrace{1}_{2^3}}^{\text{window 2}} 
& \quad
k_3 = 13 &= \overbrace{\underbrace{1}_{2^0}\underbrace{0}_{2^1}}^{\text{window 1}}\overbrace{\underbrace{1}_{2^2}\underbrace{1}_{2^3}}^{\text{window 2}} \\[1ex]

&= k_{1,1} + 2^sk_{1,2} 
& \quad &= k_{2,1} + 2^sk_{2,2} 
& \quad &= k_{3,1} + 2^sk_{3,2} \\[1ex]

&= 0 + 3(2^2) 
& \quad &= 1 + 2(2^2) 
& \quad &= 1 + 3(2^2) \\[1ex]
\end{array}
\end{align*}
$$

### 2+3. Bucket Accumulation + Reduction

$$
W_j = \sum_{i=1}^{n} k_{ij}P_i = \sum_{\ell=1}^{2^s - 1} \ell B_{\ell, j}
$$

Given our $$s$$ of 2, we now have buckets of $$[B_{1,j}, B_{2,j}, B_{3,j}]$$ for a given window $$W_j$$. Our windows are now re-structured to buckets like so:

$$
\begin{align*}
W_1 &= \overbrace{1}^{k_{2,1}}P_2 + \overbrace{1}^{k_{3,1}}P_3 = 1(P_2+P_3) = B_{1,1}\\
W_2 &= \overbrace{3}^{k_{1,2}}P_1 + \overbrace{2}^{k_{2,2}}P_2 + \overbrace{3}^{k_{3,2}}P_3 = 2(P_2) + 3(P_1 + P_3) = 2B_{2,2} + 3B_{3,2}
\end{align*}
$$

From bottom up, in our **bucket accumulation** stage we compute the buckets:

| Bucket      |  Computation  |
| ----------- | ------------- |
| $$B_{1,1}$$ | $$P_2 + P_3$$ |
| $$B_{2,2}$$ | $$P_2$$       |
| $$B_{3,2}$$ | $$P_1+P_3$$   |

...and in our **bucket reduction** stage we compute the windows:

$$
\begin{align*}
W_1 &= B_{1,1}\\
W_2 &= (B_{2,2} + B_{2,2}) + (B_{3,2} + B_{3,2} + B_{3,2})
\end{align*}
$$

### 4.  Window Reduction - Final MSM Result

Putting all the windows together for the MSM sum, we get the following:

$$
\begin{align*}
S &=\sum_{j=1}^{\lceil\frac{\lambda}{s}\rceil } 2^{(j-1)s} W_j \\
&= W_1 + 2^2W_2 \\
\end{align*}
$$

## References

* [https://hackmd.io/lNOsNGikQgO0hLjYvH6HJA](https://hackmd.io/lNOsNGikQgO0hLjYvH6HJA)
* [https://eprint.iacr.org/2022/1321.pdf](https://eprint.iacr.org/2022/1321.pdf)

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") & [Ashley Jeong](https://app.gitbook.com/u/PNJ4Qqiz7kSxSs58UIaauk95AO43 "mention") of Fractalyze
