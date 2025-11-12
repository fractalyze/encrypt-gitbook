---
description: >-
  This article aims to intuitively explain the goals and processes of the
  HyperPlonk protocol.
---

# HyperPlonk

## Introduction

[Plonk](https://eprint.iacr.org/2019/953.pdf) is a powerful tool that can be tweaked to support custom gates ([TurboPlonk](https://docs.zkproof.org/pages/standards/accepted-workshop3/proposal-turbo_plonk.pdf)) and lookup tables ([Plonkup](https://eprint.iacr.org/2022/086)). However, as circuits grow larger, FFT becomes a bottleneck. [**HyperPlonk**](https://eprint.iacr.org/2022/1355) modifies Plonk by using multivariate polynomials over a boolean hypercube. This achieves two main advantages:

1. Eliminating the need for FFT.
2. Enabling efficient computation of high-degree custom gates without significant overhead, resulting in faster proving times.

## Background

### Why is FFT a bottleneck?

<figure><img src="../../.gitbook/assets/image (75).png" alt=""><figcaption><p>Source:<a href="https://math.stackexchange.com/questions/4441884/fft-stride-pattern-formula"> https://math.stackexchange.com/questions/4441884/fft-stride-pattern-formula</a></p></figcaption></figure>

The diagram above illustrates the FFT process for a polynomial of size 8. First, as shown on the left side of the diagram, a[ bit reversal](https://en.wikipedia.org/wiki/Bit-reversal_permutation) operation is performed. Then, [butterfly](https://en.wikipedia.org/wiki/Butterfly_diagram) operations are executed at each stage. A bit reversal operation takes $$O(N)$$, and a butterfly operation also requires $$O(N)$$. Since the number of stages is $$\log N$$, the overall complexity is $$O(N \cdot \log N)$$. Furthermore, FFT is notoriously difficult to parallelize. This is because, as the stages progress, the memory locations required to compute the outputs for the next stage become increasingly scattered. If calculations must be performed on hardware like GPUs with limited memory, these constraints can be critical.

### Modern SNARK

**Polynomial IOP (Interactive Oracle Proof)** refers to a verification method where the prover provides an oracle to the verifier. The verifier interacts by sampling random points, and in the final round, accesses the oracle to verify computations on these random points.

Here, the oracle can be thought of as a polynomial. However, since polynomials often have high degrees, they are not succinct. To address this, a **PCS (Polynomial Commitment Scheme)** is used. Instead of sending the full polynomial, the prover sends a commitment that represents the polynomial. During the final round of verification, the prover can reveal (or "open") the evaluations at the random points along with a proof, enabling succinct verification. This combination allows for succinct SNARKs.

Modern SNARKs are built by combining Polynomial IOP with PCS, structured as follows:

* Univariate IOP + Univariate PCS: Examples include[ Sonic](https://eprint.iacr.org/2019/099.pdf),[ Marlin](https://eprint.iacr.org/2019/1047.pdf), and[ Plonk](https://eprint.iacr.org/2019/953.pdf).
* Multilinear IOP + Multilinear PCS: Examples include[ Hyrax](https://eprint.iacr.org/2017/1132.pdf),[ Libra](https://eprint.iacr.org/2019/317.pdf),[ Spartan](https://eprint.iacr.org/2019/550.pdf),[ Quarks](https://eprint.iacr.org/2020/1275.pdf), and[ Gemini](https://eprint.iacr.org/2022/420.pdf).
* Vector IOP + Vector PCS: Examples include[ STARK](https://eprint.iacr.org/2018/046.pdf),[ Aurora](https://eprint.iacr.org/2018/828.pdf),[ Virgo](https://people.eecs.berkeley.edu/~kubitron/courses/cs262a-F19/projects/reports/project5_report_ver2.pdf),[ Brakedown](https://eprint.iacr.org/2021/1043.pdf), and[ Orion](https://eprint.iacr.org/2022/1010.pdf).

### When is FFT used in Plonk?

|              | A | B | C |
| ------------ | - | - | - |
| $$\omega^0$$ | 1 | 1 | 2 |
| $$\omega^1$$ | 1 | 2 | 3 |
| $$\omega^2$$ | 2 | 3 | 5 |
| $$\omega^3$$ | 3 | 5 | 8 |

Let's walk through a simple circuit as an example. The circuit above represents a Fibonacci sequence starting with $$(1, 1)$$. Each column in the circuit is interpreted as a univariate polynomial. Since Plonk implements **Univariate IOP + Univariate PCS**, the protocol proceeds as follows:

<figure><img src="../../.gitbook/assets/image (82).png" alt=""><figcaption></figcaption></figure>

For simplicity, the **Wiring Check (Permutation Argument)** is omitted in this example, as the focus is on why FFT is used.

1. The prover commits to the polynomials A, B, and C to assert that the following relationship holds. This step uses **PCS.BatchCommit** and corresponds to **Oracle**:

$$
\mathsf{PCS.BatchCommit}(\{A, B,C\}) \rightarrow \{C_A, C_B, C_C\}
$$

2. After **Poly IOP for Wiring Identity**, to assert that the gate identity is satisfied, the prover commits to $$Q$$. Since $$t$$ is a known polynomial (pre-shared between prover and verifier), it does not require a separate commitment. This step corresponds to **Poly IOP for Gate Identity → Poly IOP for Quotient Check**:

$$
\mathsf{PCS.Commit}(Q) \rightarrow C_Q \\
\text { where } A(X) + B(X) - C(X) = t(X) \cdot Q(X) \text { and } t(X) = \omega^4 - 1
$$

3. The verifier samples a random $$x$$ from the entire field space.
4. Since $$x$$ is sampled randomly from the field, the prover must interpolate the polynomials $$A, B,$$ and $$C$$ to their Lagrange basis using IFFT:

$$
A(X) = L_0(X)\cdot 1 + L_1(X) \cdot1 + L_2(X) \cdot 2 + L_3(X) \cdot 3 \\
B(X) = L_0(X)\cdot 1 + L_1(X) \cdot 2+ L_2(X) \cdot 3 + L_3(X) \cdot 5\\
C(X) = L_0(X)\cdot 2 + L_1(X) \cdot 3+ L_2(X) \cdot 5 + L_3(X) \cdot 8
$$

5. The prover sends the evaluations $$A(x), B(x), C(x),$$ and $$Q(x)$$ along with the opening proof:

$$
\mathsf{PCS.BatchOpen}(\{A, B, C, Q\}, x) \rightarrow \\(\{A(x), B(x), C(x),Q(x) \}, \pi)
$$

6. The verifier checks that the gate identity holds at the random point $$x$$:

$$
\frac{A(x) + B(x) - C(x)}{t(x)} \stackrel{?}= Q(x)
$$

$$
\mathsf{PCS.BatchVerify}(S_{\mathsf{commit}}, S_{\mathsf{open}}, x, \pi) \stackrel{?}= 1 \\ 
S_{\mathsf{commit}} = \{C_A, C_B, C_C, C_Q\} \text{, } \\
S_{\mathsf{open}} = \{A(x), B(x), C(x), \frac{A(x) + B(x) - C(x)}{t(x)}\}
$$

### High-degree custom gates in Plonk

This section is based on the[ Halo2 book](https://zcash.github.io/halo2/design/proving-system/vanishing.html). Custom gates enable representation of complex operations that cannot easily be expressed with simple addition and multiplication. Let’s explore how they are implemented in Plonk.

Consider a polynomial of degree $$d$$ (or a table with $$d + 1$$ rows). In the earlier example, we dealt with a simple addition operation, where the degree of $$A + B - C$$ was $$d$$. This simplicity made computation straightforward.

When dealing with more complex circuits, custom gates can be represented as $$f_i$$, with selectors $$s_i$$. For a given domain $$H$$, each custom gate $$f_i$$ must satisfy:

$$
s_i(X) \cdot f_i(X) = t(X) \cdot Q_i(X) \text {, } \forall X \in H\text{,} \\
\text{where } s_i(X) =\begin{cases}
1 & \text{if }f_i(X) \text { is enabled}\\
0 & \text{otherwise}
\end{cases}
$$

Checking each gate $$f_i$$ individually is inefficient for the verifier. Instead, we use a common ZK technique: random linear combination. Starting from step 2 of the protocol:

1. The verifier samples a random y.
2. Using $$y$$, the prover combines all custom gates into a single equation:

$$
\sum_{i = 0}^{m - 1}y^i \cdot s_i(X)\cdot f_i(X) = t(X) \cdot Q(X)\text {, } \forall X \in H
$$

Here, $$m$$ is the number of gates.\
The degree of $$Q$$ is calculated as:

$$
\deg(Q(X)) = \max_{0 \le i < m} \deg(s_i(X) \cdot f_i(X)) - \deg(t(X)) = \max_{0 \le i < m} \deg( f_i(X)) - 1 \\
\because \deg(s_i(X)) = d \space \land \space \deg(t(X)) = d + 1
$$

For example, if $$m = 3$$, the custom gates $$f_i$$ are the following:

$$
f_0(X) = A(X) \cdot B(X) \cdot C(X) - 1\\
f_1(X) = A(X) \cdot B(X) - C(X) \\ 
f_2(X) = A(X) + B(X) - C(X)
$$

Then the degree of $$Q$$ is $$n \cdot d - 1$$, where $$n$$ is the max degree of the custom gates. (i.e., the maximum number of polynomial multiplications). Here, $$n$$ is 3. Since the degree of $$Q$$ can be much higher than $$d$$, it needs to be decomposed into $$n$$ smaller polynomials $$q_i$$ to fit within the commitment scheme:

$$
\begin{align*}
Q(X) &= \frac{\sum_{i = 0}^{m - 1}y^i \cdot s_i(X)\cdot f_i(X)}{t(X)} = \sum_{i=0}^{n-1}X^{i(d+1)} \cdot q_i(X) \\
&= q_0(X) + X^{d + 1}\cdot  q_1(X) + \dots + X^{(n - 1)\cdot(d + 1)}\cdot q_{n-1}(X)
\end{align*}
$$

Before committing to $$q_i$$, the prover must compute $$f_i$$. For high-degree gates, this involves the following:

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXc9hU5kVrV1gR78N1OhHivtpcp1UCyu9WKjNa5r2BaK7VaNMrm-SDbJbVv-t0BibB2FK1CFb__J-MFGefnXSZbi1ygh-ZqaUJpDwk-O_pYj5K6SV2_-l9HVkF3hqFfZewIoELMt?key=co-yKRJV7WOYUZIzIkM_ZekV" alt=""><figcaption></figcaption></figure>

The degree becomes $$n \cdot d$$ for the sections highlighted in green, which significantly slows down computation. If the $$n$$ of the custom gate is set too small, flexibility to create a diverse range of computations decreases. On the other hand, if $$n$$ is set too large, it introduces significant overhead in High-Degree FFT, Gate Evaluation, and High-Degree Inverse FFT. Therefore, $$n$$ must be set to an appropriate value.[ For example, $$f(x) = x * e^{2 pi i \xi x}$$n is 9 in Polygon Miden](https://0xpolygonmiden.github.io/miden-vm/design/main.html?highlight=9#design). On the other hand, there is no such limitation in Halo2.

In summary, in a simple protocol as discussed earlier, $$Q$$ could be committed before conducting the **Inverse FFT**. However, when custom gates are introduced, committing $$Q$$ requires going through the following steps:

**Inverse FFT → High Degree FFT → Gate Evaluation → High Degree Inverse FFT**.

3. The prover commits to $$q_0, q_1, q_2$$.

$$
\mathsf{PCS.BatchCommit}(\{q_0, q_1, q_2\}) \rightarrow \{C_{q_0}, C_{q_1}, C_{q_2}\}
$$

4. The verifier samples a random point $$x$$.
5. The prover sends $$A(x), B(x), C(x)$$ and $$q(x)$$ along with the opening proof $$\pi$$ to the verifier:

$$
\mathsf{PCS.BatchOpen}(\{A, B, C, Q\}, x) \rightarrow \\(\{A(x), B(x), C(x), q_0(x) + x^{d+1} \cdot q_1(x) +  x^{2(d+1)} \cdot q_2(x) \}, \pi)
$$

6. The verifier checks whether the following equation holds at the random point $$x$$. Here, $$s_i$$ is assumed to be a polynomial already known to both the prover and the verifier:

$$
\frac{s_0(x)\cdot f_0(x) + y\cdot s_1(x)\cdot f_1(x) + y^2\cdot s_2(x)\cdot f_2(x)}{t(x)} \stackrel{?}= \\
q_0(x) + x^{d + 1}\cdot q_1(x) + x^{2(d + 1)}\cdot q_2(x) = q(x)
$$

7. To validate the proof, the verifier performs the following test. If the test is passed, the verifier accepts the proof:

$$
\mathsf{PCS.BatchVerify}(S_{\mathsf{commit}}, S_{\mathsf{open}}, x, \pi) \stackrel{?}= 1 \\ 
S_{\mathsf{commit}} = \{C_A, C_B, C_C, C_Q\} \text{, } \\
\begin{align*}
S_{\mathsf{open}} = \big\{A(x), B(x), C(x), \frac{s_0(x)\cdot f_0(x) + y\cdot s_1(x) \cdot f_1(x) + y^2 \cdot s_2(x) \cdot f_2(x)}{t(x)}\big\} 
\end{align*}
$$

From the above, the approximate total execution time for the Plonk prover can be calculated as follows:

$$
\begin{align*}
  T_{\mathsf{total}} = &N_{\mathsf{poly}} \cdot T_{\mathsf{commit}} + N_{\mathsf{poly}} \cdot (T_{\mathsf{IFFT}} + T_{\mathsf{HighDegreeFFT}}) +  \\ &N_{\mathsf{gate}} \cdot (T_{\mathsf{gateeval}} + T_{\mathsf{HighDegreeIFFT}}) + T_{\mathsf{batchopen}}
  \end{align*}
$$

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXcmQUXXFSDkqOaN9gJKcfTqGm879yEZTjw2FpXE9oXZV2yIwe7HYCJ-TyL1C4pyyh3zUYrtJjobXjd79VFnHbpU9aJ4-RGQRfJ7MLECinv9GXU3pUxn93PUclFWHMc3QkCRDNrn?key=co-yKRJV7WOYUZIzIkM_ZekV" alt=""><figcaption></figcaption></figure>

Furthermore, according to the [Halo2 benchmarks from the Tachyon repository](https://github.com/kroma-network/tachyon/tree/main/tachyon/zk/plonk/halo2#7-benchmarks-for-each-circuit), it can be observed that the majority of the time is spent on the build $$h$$ poly step, which is directly related to $$Q$$.

## Protocol Explanation

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXeJb6FQ8fGc2akPyAVnJbNE5-kkoPfi7rYav23CraWFTwgoHSVDDwfl59iNiEKRDGAQEaRJgh59HoPVnJ73v1RQVXyx6awftVhy5L4THLHmW4u4HCStrdAHTMsSMyINb42qy0w?key=co-yKRJV7WOYUZIzIkM_ZekV" alt=""><figcaption></figcaption></figure>

This image is taken from the paper, and describes HyperPlonk and HyperPlonk+.

### ZeroCheck → SumCheck

In HyperPlonk, the way constraints are imposed is not fundamentally different from Plonk. For example, to represent an addition gate, the following expression can be used:

$$
A(\bm{X}) + B(\bm{X}) - C(\bm{X})
$$

Plonk uses a **ZeroCheck** followed by a **QuotientCheck**, as shown below:

$$
P(X) = 0 \space \forall X \in H \rightarrow P(X) = Q(X)\cdot T(X)
$$

where $$H := \{ \omega^i | i = 0, 1, \dots, d-1 \}$$ and $$\omega$$ is $$d$$-th root of unity.

HyperPlonk instead uses a **ZeroCheck** followed by a **SumCheck**:

$$
P(\bm{X}) = 0 \space \forall \bm{X} \in B_\mu \rightarrow \sum_{\bm{b} \in B_\mu}\widetilde{\mathsf{eq}}(\bm{X}, \bm{b})\cdot P(\bm{b}) = 0
$$

where $$B_\mu := \{0, 1\}^\mu$$ is the boolean hypercube.

### High-degree custom gates in HyperPlonk

|        | A | B | C |
| ------ | - | - | - |
| (0, 0) | 1 | 1 | 2 |
| (0, 1) | 1 | 2 | 3 |
| (1, 0) | 2 | 3 | 5 |
| (1, 1) | 3 | 5 | 8 |

Why does using SumCheck eliminate the overhead for high-degree custom gates? Using the same example as above, the prover's claim can be expressed as follows:

$$
\sum_{i = 0}^2 y^i \cdot \hat{s}_i(\bm{X}) \cdot \hat{f}_i(\bm{X}) \stackrel{?}= 0\mathsf{, where}\\
\hat{s}_i(\bm{X}) = \sum_{\bm{b} \in \{0, 1\}^{\mu}}\widetilde{\mathsf{eq}}(\bm{X}, \bm{b})\cdot s_i(\bm{b}), \\
\begin{align*}
\hat{f}_0(\bm{X}) = (\sum_{\bm{b} \in \{0, 1\}^{\mu}}\widetilde{\mathsf{eq}}(\bm{X}, \bm{b})\cdot A(\bm{b}))(\sum_{\bm{b} \in \{0, 1\}^{\mu}}\widetilde{\mathsf{eq}}(\bm{X}, \bm{b})\cdot B(\bm{b}))(\sum_{\bm{b} \in \{0, 1\}^{\mu}}\widetilde{\mathsf{eq}}(\bm{X}, \bm{b})\cdot C(\bm{b})) - 1\mathsf{,}\\

\hat{f}_1(\bm{X}) = (\sum_{\bm{b} \in \{0, 1\}^{\mu}}\widetilde{\mathsf{eq}}(\bm{X}, \bm{b})\cdot A(\bm{b}))(\sum_{\bm{b} \in \{0, 1\}^{\mu}}\widetilde{\mathsf{eq}}(\bm{X}, \bm{b})\cdot B(\bm{b}))-\sum_{\bm{b} \in \{0, 1\}^{\mu}}\widetilde{\mathsf{eq}}(\bm{X}, \bm{b})\cdot C(\bm{b})\mathsf{,}\\

\hat{f}_2(\bm{X}) = \sum_{\bm{b} \in \{0, 1\}^{\mu}}\widetilde{\mathsf{eq}}(\bm{X}, \bm{b})\cdot A(\bm{b}) + \sum_{\bm{b} \in \{0, 1\}^{\mu}}\widetilde{\mathsf{eq}}(\bm{X}, \bm{b})\cdot B(\bm{b}) -\sum_{\bm{b} \in \{0, 1\}^{\mu}}\widetilde{\mathsf{eq}}(\bm{X}, \bm{b})\cdot C(\bm{b})\\
\end{align*}
$$

In the SumCheck protocol, the univariate polynomial sent by the prover to the verifier in each round has a degree that is less than or equal to the degree of the multivariate polynomial. Thus, the prover only needs to send **univariate polynomials with a maximum degree equal to the degree of the custom gate + 1.**

Consider how a univariate polynomial can be constructed from a 1st-degree multivariate polynomial. For a 1st-degree polynomial in 2 variables, it can be expressed as:

$$
\begin{align*}
P(X_0, X_1) = &(1 - X_0)\cdot(1 - X_1)\cdot v_0 + (1 - X_0)\cdot X_1 \cdot v_1 + \\ 
&X_0\cdot (1-X_1) \cdot v_2 + X_0 \cdot X_1 \cdot v_3 
\end{align*}
$$

Now let’s create an degree 1 univariate polynomial $$P_0$$ based on the multilinear polynomial above.

$$
\begin{align*}
P_0(X) &= \sum_{X_1 \in \{0, 1\}} P(X, X_1)\\
 &= (1 - X)\cdot v_0 + (1 - X) \cdot v_1 + X \cdot v_2 + X \cdot 
v_3  \\
&= v_0 + v_1 + (v_2 + v_3 - v_0 - v_1) \cdot X
\end{align*}
$$

The complexity for this computation in round $$i$$ (starting from 0) is $$O(2^{μ - i - 1})$$.

For higher-degree multivariate polynomials, the same method is applied to each component, and the results are multiplied. For example, with $$f_0$$:

$$
f_0(X) = A_0(X)\cdot B_0(X) \cdot C_0(X) - 1\text{,} \\
\text{where } A_0(X) = 3X + 2, B(X) = 5X + 3, C(X) = 8X + 5
$$

Let’s analyze complexity for each round $$i$$ (starting from 0):

* **Univariate Polynomial Construction**: $$O(n\cdot 2^{μ − i −1})$$, where $$n$$ is the number of components.
* **Polynomial Multiplication**: $$O(n^2)$$

Since the number of total rounds is $$\mu$$, the total complexity is computed to $$O(n \cdot 2μ + μ \cdot n^2)$$ as follows:

$$
\sum_{i = 0}^{\mu - 1}O(n \cdot 2^{\mu - i - 1}) + O(n^2) = O(n \cdot (2^{\mu} - 1)) + O(\mu \cdot n^2) = O(n \cdot 2^{\mu} + \mu \cdot n^2)
$$

If $$n$$ is treated as a constant, the SumCheck protocol allows **ZeroCheck to be performed in linear time**. For reference, the complexity calculation here differs from the one provided in the paper. For more details, refer to Section 3.1 (Computing SumCheck for high-degree polynomials) and Section 3.8 of the original paper, which introduce further optimizations for the SumCheck protocol.

### ProductCheck

**ProductCheck** is a method to verify whether the product of a polynomial over a boolean hypercube equals a constant $$c$$:

$$
\prod_{\bm{X} \in \{0, 1\}^\mu}h(\bm{X}) = c\text{, where } h(\bm{X}) = \frac{f(\bm{X})}{ g(\bm{X})}
$$

For univariate polynomials, this is typically computed using a running product column, as explained in [From Halo2, LogUp to LogUp-GKR](https://blog.kroma.network/from-halo2-lookup-logup-to-logup-gkr-4af3bf143d38). For multivariate polynomials, the process involves the following steps:

1. Define $$v$$ to satisfy the following conditions:

$$
v(0, \bm{X}) = h(\bm{X}), \space v(1, \bm{X}) = v(\bm{X}, 0) \cdot v(\bm{X}, 1)\text{, where } v(1, \dots, 1) = 0
$$

This allows $$v$$ to be computed as shown in the diagram below:

|               | f(X)        | g(X)        | v(X)                                                                                                         |
| ------------- | ----------- | ----------- | ------------------------------------------------------------------------------------------------------------ |
| $$(0, 0, 0)$$ | $$f(0, 0)$$ | $$g( 0,0)$$ | $$\frac{f( 0,0)}{g(0, 0)}$$                                                                                  |
| $$(0, 0, 1)$$ | $$f(0, 1)$$ | $$g(0, 1)$$ | $$\frac{f( 0,1)}{g(0, 1)}$$                                                                                  |
| $$(0, 1, 0)$$ | $$f(1, 0)$$ | $$g(1, 0)$$ | $$\frac{f(1,0)}{g(1,0)}$$                                                                                    |
| $$(0, 1, 1)$$ | $$f(1, 1)$$ | $$g(1,1)$$  | $$\frac{f(1,1)}{g(1,1)}$$                                                                                    |
| $$(1, 0, 0)$$ |             |             | $$\frac{f(0,0)\cdot f(0,1)}{g(0, 0) \cdot g(0, 1)}$$                                                         |
| $$(1, 0, 1)$$ |             |             | $$\frac{f(1,0)\cdot f(1,1)}{g(1, 0) \cdot g(1, 1)}$$                                                         |
| $$(1, 1, 0)$$ |             |             | $$\frac{f(0, 0) \cdot f(0, 1) \cdot f(1,0)\cdot f(1,1)}{g(0, 0) \cdot g(0, 1) \cdot g(1, 0) \cdot g(1, 1)}$$ |
| $$(1, 1,1)$$  |             |             | $$0$$                                                                                                        |

2. Define a new function $$\hat{h}$$ as follows:

$$
\begin{align*}
  \hat{h}(X_0, \bm{X}) = (1 - X_0)\cdot(v(1, \bm{X}) - v(\bm{X}, 0)\cdot v(\bm{X}, 1)) + X_0\cdot (g(\bm{X})\cdot v(0, \bm{X}) - f(\bm{X}))
  \end{align*}
$$

3. Using $$\hat{h}$$, perform **ZeroCheck → SumCheck** as described earlier. Since $$\hat{h}$$ evaluates to zero for all points in $$\{0,1\}^3$$, this check ensures the validity of $$v$$.
4. Finally, the verifier queries $$v$$ at $$(1, \dots, 1, 0)$$ to ensure the products of $$h$$ are equal to $$c$$.

### Wiring Identity → MultiSetEquality

In univariate polynomials, verifying Wiring Identity involves constructing $$s_i^\sigma$$ as shown in the figure below. The green arrows denote how the label is moved.

<figure><img src="../../.gitbook/assets/Screenshot 2025-02-10 at 4.35.40 PM.png" alt=""><figcaption></figcaption></figure>

This is then transformed into a **MultiSetEquality** using a random value $$\alpha$$:

$$
\frac{\prod_{i = 0}^3(X - \alpha s_0(\omega^i )- A(\omega^i))(X - \alpha s_1(\omega^i )- B(\omega^i))(X - \alpha  s_2(\omega^i ) - C(\omega^i))}{
\prod_{i = 0}^3(X - \alpha s^{\sigma}_0(\omega^i) - A(\omega^i))(X - \alpha  s^{\sigma}_1(\omega^i) - B(\omega^i))(X - \alpha s^{\sigma}_2(\omega^i) - C(\omega^i))} \stackrel{?}= 1
$$

In multivariate polynomials, $$s_i^\sigma$$ is similarly constructed as shown in the figure below.

<figure><img src="../../.gitbook/assets/Screenshot 2025-02-10 at 4.36.26 PM.png" alt=""><figcaption></figcaption></figure>

Using the same random value $$\alpha$$, this is transformed into a **MultiSetEquality** as follows:

$$
\frac{\prod_{\bm{b} \in \{0, 1\}^2}(\bm{X} - \alpha s_0(\bm{b} )- A(\bm{b}))(\bm{X} - \alpha s_1(\bm{b} )- B(\bm{b}))(\bm{X} - \alpha  s_2(\bm{b} )  - C(\bm{b}))}{
\prod_{\bm{b} \in \{0, 1\}^2}(\bm{X} - \alpha s^{\sigma}_0(\bm{b}) - A(\bm{b}))(\bm{X} - \alpha  s^{\sigma}_1(\bm{b}) - B(\bm{b}))(\bm{X} - \alpha s^{\sigma}_2(\bm{b}) - C(\bm{b}))} \stackrel{?}= 1
$$

**MultiSetEquality** can now be replaced by the **ProductCheck** protocol described earlier. This enables efficient verification using the established steps for product consistency.

For handling **Permutation PIOP** on small fields, Section 3.6 of the paper provides a detailed explanation. Interested readers are encouraged to explore that section for additional insights.

### Shift Function for Lookup Argument

Incorporating[ **Plookup**](https://eprint.iacr.org/2020/315.pdf) into HyperPlonk creates **HyperPlonk+**.

$$
F(\beta, \gamma) := (1 + \beta) ^n \cdot \prod_{i \in [n] }(\gamma + f_i) \cdot \prod_{i \in [d-1]} (\gamma(1 + \beta) + t_i + \beta \cdot t_{i + 1}) \\
G(\beta, \gamma) := \prod_{i \in [n + d -1]} (\gamma(1 + \beta) + s_i + \beta \cdot s_{i + 1})
$$

where $$s$$ is  $$(f, t)$$ sorted by $$t$$.

The equation above, taken from the Plookup paper, shows how values $$f_i, s_i, t_{i+1}, s_{i+1}$$ ​ are queried at both the $$i$$-th and $$i+1$$-th indices. For univariate polynomials in Plonk, this can be easily expressed as:

$$
f_i = f(x), s_i = s(x), t_{i+1} = t(\omega \cdot x), s_{i+1} = s(\omega \cdot x)
$$

This "shift through the domain" is achieved using a **shift function**, which for univariate polynomials is defined as:

$$
\mathsf{next}(X) = \omega \cdot X
$$

For multivariate polynomials, the shift function can be represented using a mechanism similar to the one illustrated below (also from the paper):

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXdPN_ezj6UdlLhDqW9zZ-UTPQXuiPFcq2cRdFxk7eZBkIAPrHc9_TXA2cI_a58DAVx_opx5KWAsiGcjIhB8VY8VkE_FGhhBlTOshlndJlFRSbg3u04Zcicj0N1aAJoc4FwnM8X1?key=co-yKRJV7WOYUZIzIkM_ZekV" alt=""><figcaption></figcaption></figure>

Consider $$g_4(b_1, b_2, b_3, b_4)$$ in the context of a 4-dimensional boolean hypercube. The function $$g_4$$ operates as follows:

$$
\begin{align*}
g_4(b_1, b_2, b_3, b_4) &= (b_1 + b_2X + b_3X^2 + b_4X^3) \cdot X \\
&= b_1X + b_2X^2 + b_3X^3 + b_4X^4 \\
&= -b_4 + (b_1 - b_4)X + b_2X^2 + b_3X^3 \\
&= b_4 + (b_1 + b_4)X + b_2X^2 + b_3X^3 \\
&= b_4 + (b_1 \oplus b_4)X + b_2X^2 + b_3 X^3 \\
&= (b_4, b_1 \oplus b_4, b_2, b_3) \\
&= (b_4, b_1', b_2', b_3'), 
\end{align*} \\
\\
\text{where } p_4 = X^4 + X + 1
$$

This cyclically shifts all non-zero points in the boolean hypercube.

$$
1: g_4(0, 0, 0, 1) = (1, 1, 0, 0) \\
2: g_4(1, 1, 0, 0) = (0, 1, 1, 0) \\
3: g_4(0, 1, 1, 0) = (0, 0, 1, 1) \\
4: g_4(0, 0, 1, 1) = (1, 1, 0, 1) \\
5: g_4(1, 1, 0, 1) = (1, 0, 1, 0) \\
6:g_4(1, 0, 1, 0) = (0, 1, 0, 1) \\
7:g_4(0, 1, 0, 1) = (1, 1, 1, 0)\\
8:g_4(1, 1, 1, 0) = (0, 1, 1, 1)\\
9:g_4(0, 1, 1, 1) = (1, 1, 1, 1)\\
10: g_4(1, 1, 1, 1) = (1, 0, 1, 1)\\
11: g_4(1, 0, 1, 1) = (1, 0, 0, 1)\\
12: g_4(1, 0, 0, 1) = (1, 0, 0, 0)\\
13: g_4(1, 0, 0, 0) = (0, 1, 0, 0)\\
14: g_4(0, 1, 0, 0) = (0, 0, 1, 0)\\
15:g_4(0, 0, 1, 0) = (0, 0, 0, 1)\\
$$

The XOR operation can be algebraically expressed as:

$$
X \oplus Y = X + Y - 2 \cdot X \cdot Y
$$

However, this representation is quadratic, leading to an explosion in degree when using nested functions like $$f(g(X))$$.

Luckily, $$g_4$$ can be simplified based on specific rules:

$$
g_4(b_1, b_2, b_3, b_4) = (b_1', b_2', b_3', b_4') =  \begin{cases} 
		(1, 1 - b_1, b_2, b_3) & \text{if } b_4 = 1 \\ 
         (0, b_1, b_2, b_3) & \text{if } b_4 = 0
     \end{cases}
$$

This reduces the complexity, allowing $$g_4$$ to be expressed as a linear function rather than a **quadratic** one. This is depicted in the formula below:

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXclE1urNJXSmB8GKG-D-XZvMV2AGXPRx-W8GRTlleXarE1rplM52TUqKz_mGTOjeElPh8NxyT1ML9qMxC0eG1lhxoYqdp8wRLcQRdQYhnqX4OM29sqmR32HY7IP21FxKolhi8cd?key=co-yKRJV7WOYUZIzIkM_ZekV" alt=""><figcaption></figcaption></figure>

Using this linearized shift function, **MultiSetEquality Check** can be efficiently performed. By leveraging the defined **Polynomial IOP**, this shift function enables the protocol to verify lookup arguments in HyperPlonk+ without significant overhead.

This also implies that the domain in the circuit must be represented differently. Specifically, the point $$(0, 0)$$ cannot be included because it is not cyclic. Instead, the cyclic domain should exclude $$(0, 0)$$ and follow this order in the case of 4 or 8 rows:

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfNgCUi6h4mFFNTjnmzVhjc6CH9MVcQnl5dMsTBH7JwfzSpGpOtgdURjku-yVT_49PXFOlaYB44SbczaTcil5KfmfG_s8qtcK8-D3c_jhWQdf3UkBTvgv_mhZCr_6sxAUmeeA4?key=co-yKRJV7WOYUZIzIkM_ZekV" alt=""><figcaption></figcaption></figure>

## Conclusion

We have explored how **HyperPlonk** operates through examples. Unlike Plonk, HyperPlonk introduces a method to leverage the **boolean hypercube** for proofs. **This approach eliminates the computational overhead caused by FFTs as the maximum degree of custom gates increases, resulting in faster proof generation.**

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXddGizbQPFh5UndkXf1giZdHN-IXepDW6G5uVRuQJWFRngtLjn4jgAb9CUUnD53eMEaYDNV-Qx9azNJBeDjzdn4tiat-WqCjTn3LV_M8d_mYyjnz8sR56T0rwBBQbID5VqOdgQ?key=co-yKRJV7WOYUZIzIkM_ZekV" alt=""><figcaption></figcaption></figure>

The figure above is a capture from the presentation by **Benedikt Bünz** at **ZK Summit 8** ([YouTube link](https://www.youtube.com/watch?v=2JDBD5oMS0w)):

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfh9LmTcxRG2VUX9-tC_2G7kGwXVcu_J3iyju4dAHPhuH4obMamPEPaUCxzDCdmBVSWBQ0PIA5ksIHIQ5FsI0XSDRdRmLPFZi2xYDvLWSrBNRMq96wz1aSLnV2mY7r8vK0gTnQ?key=co-yKRJV7WOYUZIzIkM_ZekV" alt=""><figcaption></figcaption></figure>

Graph b above and the video screenshot both support this observation. While not covered, HyperPlonk has also contributed to the optimization of multivariate polynomial commitments, as seen in systems like Orion+. Despite these significant contributions, the Conclusions and Open Problems section of the paper highlights several areas for future work, introducing ideas for continued improvements and research.

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
