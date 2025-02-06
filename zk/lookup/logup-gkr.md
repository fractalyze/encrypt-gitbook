---
description: >-
  This article explores how the Halo2 Lookup, LogUp, and LogUp-GKR protocols
  have evolved.
---

# LogUp-GKR

## Introduction

Demonstrating that all values in Set $$\bm{A}$$ belong to Set $$\bm{S}$$ is the core feature of Lookup. In ZK, this Lookup is used to prove that a value falls within a specific range or to reference a value proven in another circuit. [Since 2022, significant advancements have been made in Lookup techniques](https://ingonyama-zk.github.io/ingopedia/protocolsLookup.html). Recently, it has converged into implementing one of two approaches: LogUp-GKR or Lasso. At ProgCrypto 2023, as seen in [the video by Dohun Kim](https://www.youtube.com/watch?v=AyhU7j2nGGI) from the PSE team, the performance of the two methods is reported to be nearly identical. In this article, we aim to discuss how Halo2 Lookup evolved into LogUp-GKR, as observed in the video.

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXctCTK9dno_sgW663Eph9OiyS-Nq7AHbsyTRdcSVQHoAxCQUOYw8PKIw3OVkyigZaXahjzdi4Fi_DaafBUMyZp9sCoA0Avcm8xATg6Wsyoh7kxjyjgHw3cZ9C7IpezDc0USIBWp?key=qDCl38TNxspCRBTGEVc_9MzT" alt=""><figcaption></figcaption></figure>

## Background

Prerequisites: See [Multiset Check](../../primitives/multiset-check.md) and [Sumcheck](../../primitives/sumcheck.md) for more details

### Polynomial = Column

In the arithmetization of PLONK or AIR, witnesses and public values are commonly filled into a table. Here, let’s assume there are an input polynomial $$A(X)$$ and a table polynomial $$S(X)$$. Typically, the number of the rows is in the form of a power of 2. In this case, the columns of the table are interpreted as polynomials, and the rows are interpreted as the domain. In the example below, the columns of the table are interpreted as univariate polynomials, and $$\omega$$ is the 4-th root of unity. Therefore, $$\omega^4 = 1$$ holds.

|              | A(X) | S(X) |
| ------------ | ---- | ---- |
| $$\omega^0$$ | 1    | 1    |
| $$\omega^1$$ | 2    | 2    |
| $$\omega^2$$ | 1    | 3    |
| $$\omega^3$$ | 3    | 1    |

In the table above, $$A(X)$$ can be expressed as follows, where $$L_i$$​ refers to the Lagrange Basis. (See[ Lagrange Polynomial](https://en.wikipedia.org/wiki/Lagrange_polynomial) for more information.)

$$
A(X) = L_0(X) \cdot 1 + L_1(X) \cdot 2 + L_2(X) \cdot 1 + L_3(X) \cdot 3 \\
L_i(X) = \begin{cases}
1 & \text { if } X = \omega^i \\
0 & \text  { otherwise }
\end{cases}
$$

On the other hand, if the domain is expressed as a boolean hypercube as shown below, it can also be interpreted as a multivariate polynomial. (Here, $$X$$, unlike the previous example, represents a 2-dimensional vector.)

|            | A(X) | S(X) |
| ---------- | ---- | ---- |
| $$(0, 0)$$ | 1    | 1    |
| $$(1, 0)$$ | 2    | 2    |
| $$(0, 1)$$ | 1    | 3    |
| $$(1, 1)$$ | 3    | 1    |

In the table above, $$A(X)$$ can be expressed as follows. (See [eq extension](https://jolt.a16zcrypto.com/background/eq-polynomial.html) for more details)

$$
A(\bm{X}) = \mathsf{eq}(\bm{X}, (0, 0)) \cdot 1 + \mathsf{eq}(\bm{X}, (1, 0)) \cdot 2 + \mathsf{eq}(\bm{X}, (0, 1)) \cdot 1 + \mathsf{eq}(\bm{X}, (1, 1)) \cdot 3 \\
\mathsf{eq}(\bm{X}, \bm{Y}) = \begin{cases}
1 & \text{ if } \bm{X} = \bm{Y} \\
0 & \text{ otherwise }
\end{cases}
$$

## Protocol Explanation

### Halo2 Lookup

Lookup tries to prove that all the unique values in set $$\bm{A}$$ are also in set $$\bm{S}$$. In practice, $$\bm{S}$$ is much smaller and consists of unique, non-repeating values.

$$
\bm{A} = \{1, 2,1,3\} \\
\bm{S} = \{1, 2, 3\}
$$

All rows larger than $$\bm{A}$$ and $$\bm{S}$$ are filled with zeros. In Halo2 Lookup, new polynomials (or auxiliary columns) $$A'(𝑋)$$ and $$S'(𝑋)$$ are created as shown below. Before creating $$A'(𝑋)$$ and $$S'(𝑋)$$, $$A(𝑋)$$ and $$S(𝑋)$$ must be committed. (For convenience, $$L_0(X)$$ is included below, but it is not actually a committed polynomial.)  \


|              | L₀(X) | A(X) | S(X) | A'(X) | S'(X) |
| ------------ | ----- | ---- | ---- | ----- | ----- |
| $$\omega^0$$ | 1     | 1    | 1    | 1     | 1     |
| $$\omega^1$$ | 0     | 2    | 2    | 1     | 0     |
| $$\omega^2$$ | 0     | 1    | 3    | 2     | 2     |
| $$\omega^3$$ | 0     | 3    | 0    | 3     | 3     |
| $$\vdots$$   | 0     | 0    | 0    | 0     | 0     |
| $$\omega^7$$ | 0     | 0    | 0    | 0     | 0     |

$$A'(X)$$ is a polynomial derived from sorting $$A(X)$$ and $$S'(X)$$ must satisfy one of the following conditions:

* If it is the first row, the values of $$A'(X)$$ and $$S'(X)$$ must be the same.
* The values of $$A'(X)$$ and its preceding column $$A'(ω^{-1}X)$$ must be the same.
* If $$A'(X)$$ and $$A'(ω^{-1}X)$$ are not the same, $$A'(X)$$ and $$S'(X)$$ must have the same values.

These constraints can be expressed as follows:

$$
L_0(X)\cdot (A'(X) - S'(X)) = 0 \\
(A'(X) - A'(\omega^{-1}X)) \cdot (A'(X) - S'(X)) = 0
$$

If it can be proven that $$A'(X)$$ and $$S'(X)$$ satisfy the constraints above, the previously discussed MultiSet Check can now be applied. A new polynomial $$Z(X)$$ is then defined to run MultiSet Check, which must satisfy the following constraints. The column representing $$Z(X)$$ is referred to as the running product column.

* The first row must equal 1.
* The next row $$Z(ωX)$$ is the value obtained by multiplying the current $$Z(X)$$ by $$(A(X) + \beta)(S(X) + \gamma)$$ and dividing by $$(A'(X) + \beta)(S'(X) + \gamma)$$.
* The last row must equal either 0 or 1. While a value of 1 is trivial, 0 occurs when $$\beta$$ and $$\gamma$$ are sampled in such a way that either the numerator or denominator becomes zero, causing all subsequent entries to be filled with zeros.

These constraints can be expressed as follows:

$$
L_0(X) \cdot (Z(X) - 1) = 0 \\
(1 - L_{\mathsf{last}}(X)) \cdot (Z(\omega X) \cdot (A'(X) + \beta) \cdot (S'(X) + \gamma) - 
Z(X) \cdot (A(X) + \beta) \cdot (S(X) + \gamma))  = 0 \\
L_{\mathsf{last}}(X) \cdot (Z^2(X) - Z(X)) = 0
$$

For example, if $$\beta = 2$$ and $$\gamma = 3$$, $$Z(X)$$ will be filled as follows. Before constructing $$Z(X)$$, $$A'(X)$$ and $$S'(X)$$ must be committed (For convenience, $$L_{\mathsf{last}}(X)$$ is included below, but it is not actually a committed polynomial). Since the index of the last row is 4, $$L_{\mathsf{last}}(\omega^4) = 1$$.

|              | L₀(X) | Lₗₐₛₜ(X) | A(X) | S(X) | A'(X) | S'(X) | Z(X) |
| ------------ | ----- | -------- | ---- | ---- | ----- | ----- | ---- |
| $$\omega^0$$ | 1     | 0        | 1    | 1    | 1     | 1     | 1    |
| $$\omega^1$$ | 0     | 0        | 2    | 2    | 1     | 0     | 1    |
| $$\omega^2$$ | 0     | 0        | 1    | 3    | 2     | 2     | 20/9 |
| $$\omega^3$$ | 0     | 0        | 3    | 0    | 3     | 3     | 2    |
| $$\omega^4$$ | 0     | 1        | 0    | 0    | 0     | 0     | 1    |
| $$\vdots$$   | 0     | 0        | 0    | 0    | 0     | 0     | 0    |
| $$\omega^7$$ | 0     | 0        | 0    | 0    | 0     | 0     | 0    |

In a Polynomial Interactive Oracle Protocol (PIOP), we use a Polynomial Commitment Scheme (PCS) to transform the problem into an Interactive Oracle Protocol (IOP). The verifier must open several values via the oracle to check the correctness of the relationship above. Assuming we open at $$x$$, the maximum number of values per polynomial to open is 2.

$$
Z(x), Z(\omega x), A(x), S(x), A'(\omega^{-1}x), A'(x), S'(x)
$$

| Polynomial | Number of opening values |
| ---------- | ------------------------ |
| $$Z$$      | 2                        |
| $$A$$      | 1                        |
| $$S$$      | 1                        |
| $$A'$$     | 2                        |
| $$S'$$     | 1                        |

Since this could break the ZK property, additional random values $$a$$ to $$j$$ are added to two columns as shown in the table below. (For convenience, $$\mathsf{Blind}(X)$$ is included but is not actually a committed polynomial.) &#x20;

$$
L_0(X) \cdot (A'(X) - S'(X)) = 0\\
L_0(X) \cdot (Z(X) - 1) = 0\\
L_{\mathsf{last}}(X) \cdot (Z^2(X) - Z(X)) = 0\\
(1 - L_{\mathsf{last}}(X) - \mathsf{Blind}(X)) \cdot (Z(\omega X) \cdot (A'(X) + \beta)\cdot(S'(X) + \gamma) - Z(X) \cdot (A(X) + \beta)\cdot(S(X) + \gamma)) = 0\\
(1 - L_{\mathsf{last}}(X) - \mathsf{Blind}(X)) \cdot (A'(X) - A'({\omega}^{-1}X)) \cdot (A'(X) - S'(X)) = 0\\
$$

<table><thead><tr><th></th><th>L₀(X)</th><th>Lₗₐₛₜ(X)</th><th>Blind(X)</th><th>A(X)</th><th>S(X)</th><th width="99">A'(X)</th><th>S'(X)</th><th>Z(X)</th></tr></thead><tbody><tr><td><span class="math">\omega^0</span></td><td>1</td><td>0</td><td>0</td><td>1</td><td>1</td><td>1</td><td>1</td><td>1</td></tr><tr><td><span class="math">\omega^1</span></td><td>0</td><td>0</td><td>0</td><td>2</td><td>2</td><td>1</td><td>0</td><td>1</td></tr><tr><td><span class="math">\omega^2</span></td><td>0</td><td>0</td><td>0</td><td>1</td><td>3</td><td>2</td><td>2</td><td>20/9</td></tr><tr><td><span class="math">\omega^3</span></td><td>0</td><td>0</td><td>0</td><td>3</td><td>0</td><td>3</td><td>3</td><td>2</td></tr><tr><td><span class="math">\omega^4</span></td><td>0</td><td>1</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>1</td></tr><tr><td><span class="math">\omega^5</span></td><td>0</td><td>0</td><td>1</td><td>a</td><td>b</td><td>c</td><td>d</td><td>e</td></tr><tr><td><span class="math">\omega^6</span></td><td>0</td><td>0</td><td>1</td><td>f</td><td>g</td><td>h</td><td>i</td><td>j</td></tr><tr><td><span class="math">\omega^7</span></td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td></tr></tbody></table>

When using Halo2 Lookup, if there are $$n$$ input polynomials referencing $$S(X)$$, there are $$3n$$ columns to commit, as there are $$n$$ values of each $$A'$$, $$S'$$, and $$Z$$. The downside of this method is inefficiency, as the same $$S(X)$$ is referenced repeatedly, requiring redundant computations with $$S(X)$$ and $$S'(X)$$. Considering that Lookup is often used to verify ranges, this cost cannot be ignored. &#x20;

For more details, refer to [Lookup Argument - The Halo2 Book](https://zcash.github.io/halo2/design/proving-system/lookup.html).

### LogUp

In the [original LogUp](https://eprint.iacr.org/2022/1530), the protocol is described for multivariate polynomials, but here it is described for univariate polynomials as used in Scroll Halo2.

Let us assume there are two polynomials referencing $$\bm{S}$$, as shown below:

$$
\bm{A_0} = \{1,2,1,3\} \\
\bm{A_1} = \{1,1,2,2\} \\
\bm{S} = \{1,2,3\}
$$

To solve the problem above, let us create a polynomial $$m(X)$$ that indicates the degree of multiplicity. Before creating $$m(X)$$, $$A_0(X)$$, $$A_1(X)$$, and $$S(X)$$ must be committed.

|              | A₀(X) | A₁(X) | m(X) | S(X) |
| ------------ | ----- | ----- | ---- | ---- |
| $$\omega^0$$ | 1     | 1     | 4    | 1    |
| $$\omega^1$$ | 2     | 1     | 3    | 2    |
| $$\omega^2$$ | 1     | 2     | 1    | 3    |
| $$\omega^3$$ | 3     | 2     | 0    | 0    |
| $$\vdots$$   | 0     | 0     | 0    | 0    |
| $$\omega^7$$ | 0     | 0     | 0    | 0    |

Based on this, let us assume that, as before, a running product column $$Z(X)$$ is created. However, with $$m(X)$$ expressed as an exponent, it can no longer be represented as a polynomial.

$$
L_0(X) \cdot (Z(X) - 1) = 0 \\
(1 - L_{\mathsf{last}}(X)) \cdot Z(\omega X) \cdot \prod_{i=0}^{1}(A_i(X) + \beta)\cdot(S(X) + \gamma)^{m(X)} - Z(X) \cdot \prod_{i=0}^{1}(A_i(X) + \beta)\cdot(S(X) + \gamma)^{m(X)} = 0 \\
L_{\mathsf{last}}(X) \cdot (Z^2(X) - Z(X)) = 0 \\
$$

Lemma 3 of LogUp summarizes that Multiset Check, based on products, can be replaced with a sum-based method as shown below:

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXedvE49ZaLy8iVCR2bN6s3cydOkaIcgPNvu1_3s1UeUDd9bs_pGQsbVoX69EWyFJA3pVrXDLaMuW4CrviUfLVA7hRh9j6LOntXk9Cc_oz-DzUVdEw-jdbr2UOdM5VS1-4C17Xe_?key=i-_YTrjb0_QJkFzBaKFWlQ0U" alt=""><figcaption></figcaption></figure>

Lemma 5 of LogUp summarizes how multiplicity can be utilized in the sum-based method:

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXe5MeKZd0MzcaXVC8kngLgu_WwdtChx5WWQkziCpjSy01uFJgCJvlGy3Pu0GRDatG3ssIHppimM64JLxNB8A8KxNAZ5c_e8G9I8jgxbskKP4lfGr0uj-00UsrMjS1Mz0LEB5JsU?key=i-_YTrjb0_QJkFzBaKFWlQ0U" alt=""><figcaption></figcaption></figure>

Both lemmas can be derived informally by taking the logarithm on both sides of the Multiset equality based on products followed by taking derivative of both sides. The converse is shown following the reverse process.

By applying this, a new polynomial $$Z(X)$$ can be defined, which must satisfy the following constraints. The column representing this $$Z(X)$$ is referred to as the running sum column.

* The first row must equal 0.
* The next row $$Z(\omega X)$$ is the value obtained by adding $$\frac{1} {(A_0(X) + \beta)} + \frac{1}{(A_1(X) + \beta)}$$ to the current $$Z(X)$$ and subtracting $$\frac{m(X)}{ (S(X) + \beta)}$$.
* The last row must equal 0. (Unlike the running product column, the check for equality to 1 is removed.)

These constraints can be expressed as follows:

$$
L_0(X) \cdot Z(X) = 0 \\
(1 - L_{\mathsf{last}}(X)) \cdot (Z(\omega X) - Z(X) - \sum_{i=0}^{1}(\frac{1}{A_i(X) + \beta}) + \frac{m(X)}{S(X) + \beta} = 0 \\
L_{\mathsf{last}}(X) \cdot Z(X) = 0 \\
$$

Before constructing $$Z(X)$$, $$m(X)$$ must be committed. If $$\beta=2$$, $$Z(X)$$ will be filled as shown below. (For convenience, $$L_0(X)$$ and $$L_{last}(X)$$ are included below but are not actually committed polynomials.)

|              | L₀(X) | Lₗₐₛₜ(X) | A₀(X) | A₁(X) | m(X) | S(X) | Z(X)  |
| ------------ | ----- | -------- | ----- | ----- | ---- | ---- | ----- |
| $$\omega^0$$ | 1     | 0        | 1     | 1     | 4    | 1    | 0     |
| $$\omega^1$$ | 0     | 0        | 2     | 1     | 3    | 2    | -2/3  |
| $$\omega^2$$ | 0     | 0        | 1     | 2     | 1    | 3    | -5/6  |
| $$\omega^3$$ | 0     | 0        | 3     | 2     | 0    | 0    | -9/20 |
| $$\omega^4$$ | 0     | 1        | 0     | 0     | 0    | 0    | 0     |
| $$\vdots$$   | 0     | 0        | 0     | 0     | 0    | 0    | 0     |
| $$\omega^7$$ | 0     | 0        | 0     | 0     | 0    | 0    | 0     |

In Scroll Halo2, constraints are flexibly expressed with fractional sums, whereas in the original LogUp, they are strictly expressed with polynomial sums. This is also the case in [the video](https://www.youtube.com/watch?v=KeuibiW_tRc) and [article](https://hackmd.io/@plafer/HJGhiRIPC/%2FS1X5XFAwR#Transition-constraint) presented by Polygon Miden at ZKSummit 12.

With the strict method, the degree of the constraint polynomial can increase, which is why the original LogUp separately creates a helper function $$h$$ as shown below:

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXdkS8qrqxIPoynXCKZVwI47CSP6HGOYSJYQ7uJs1Z62sSyOvq0fcqsVs8XRjoW2gPs2Z3gjf_X3W8wvd_91KefDgTaRfTz9cmBnDkBHEkAuebiCCDd-1PeIcmCon2MmeEOyOR7Q?key=i-_YTrjb0_QJkFzBaKFWlQ0U" alt=""><figcaption></figcaption></figure>

When using univariate LogUp, if there are $$n$$ input polynomials referencing $$S(X)$$, there are 2 columns to commit: $$m(X)$$ and $$Z(X)$$.

By introducing $$m(X)$$ in LogUp, the computational overhead is reduced. If $$n$$ is large, the reduction becomes even more significant. However, there is still overhead associated with generating and committing $$Z(X)$$. Can this overhead be eliminated?

### LogUp-GKR

In [https://eprint.iacr.org/2023/1284.pdf](https://eprint.iacr.org/2023/1284.pdf), inspired by Lasso, GKR is introduced to further enhance Lookup performance. The height of a layer as 3 can be represented as a layered circuit in GKR as follows:

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfhDhJL2rBhfl9K_IU1W5WxRJU93DkMmRgBrFu4iw0u5WqG-Ay3adfPWTiwnIA0_Mz_QQY8B4D9MUfV6PzLCSHeiuXTNv7kkEfiYzzSs8DuaFCtReNPAHRHaEQ4dfRrjMshn8OK?key=i-_YTrjb0_QJkFzBaKFWlQ0U" alt=""><figcaption></figcaption></figure>

Here, $$+_F$$ represents the fraction sum where $$P_i(X)$$ is the numerator and $$Q_i(X)$$ is the denominator.

$$
\frac{P_i(\bm{X})}{Q_i(\bm{X})} = \frac{P_{i+1}(\bm{X}, 0) \cdot Q_{i+1}(\bm{X}, 1) + P_{i+1}(\bm{X}, 1) \cdot Q_{i+1}(\bm{X}, 0)}{Q_{i+1}(\bm{X}, 0) \cdot Q_{i+1}(\bm{X}, 1)}
$$

Hence, $$+_F$$ is formally defined as below.

$$
(P_{i+1}(\bm{X}, 0), Q_{i+1}(\bm{X}, 0)) +_F (P_{i+1}(\bm{X}, 1), Q_{i+1}(\bm{X}, 1)) \\
 = (P_{i+1}(\bm{X}, 0) \cdot Q_{i+1}(\bm{X}, 1) + P_{i+1}(\bm{X}, 1) \cdot Q_{i+1}(\bm{X}, 0), Q_{i+1}(\bm{X}, 0) \cdot Q_{i+1}(\bm{X}, 1)) \\
 = (P_i(\bm{X}), Q_i(\bm{X}))
$$

The constraints can be expressed as follows:

$$
P_0(0) \cdot Q_0(1) + P_0(1) \cdot Q_0(0) = 0 \\
Q_0(0) \cdot Q_0(1) \neq 0 \\
P_k(\bm{X}) = \sum_{\bm{b} \in \{0, 1\}^{k + 1}} \mathsf{eq}(\bm{X}, \bm{b}) \cdot ( P_{k+1}(\bm{b}, 0) \cdot Q_{k+1}(\bm{b}, 1) + P_{k+1}(\bm{b}, 1) \cdot Q_{k+1}(\bm{b}, 0) ) \\
Q_k(\bm{X}) = \sum_{\bm{b} \in \{0, 1\}^{k + 1}} \mathsf{eq}(\bm{X}, \bm{b}) \cdot Q_{k+1}(\bm{b}, 0) \cdot Q_{k+1}(\bm{b}, 1)
$$

where the first two constraints force the two output fractions to add up to zero.

The interaction between the prover and verifier proceeds in the following sequence:

* The prover sends $$P_0(0)$$, $$Q_0(0)$$, $$P_0(1)$$, $$Q_0(1)$$ to the verifier.
* The verifier checks the validity of these values.
* The verifier samples $$ho_0$$ and $$\lambda_0$$ and sends them to the prover.
* The prover and verifier execute the sumcheck protocol to assert $$S_0 = P_0(\rho_0) + \lambda_0 Q_0(\rho_0)$$ over the equation below until the final layer.

$$
\widetilde{P_i}(\bm{X}) + \lambda_i \widetilde{Q_i}(\bm{X}) = 

\sum_{\bm{b} \in \{0,1\}^{k + 1}} \mathsf{eq}(\bm{X}, \bm{b})[\widetilde{P_{i+1}}(\bm{b}, 0) \cdot \widetilde{Q_{i+1}}(\bm{b}, 1) + \widetilde{P_{i+1}}(\bm{b}, 1) \cdot \widetilde{Q_{i+1}}(\bm{b}, 0) + \lambda_i \widetilde{Q_{i+1}}(\bm{b}, 0) \cdot \widetilde{Q_{i+1}}(\bm{b}, 1) ]
$$

* For the final layer, the prover sends $$P_{\mathsf{last}-1}(\rho_0, \dots, X), Q_{\mathsf{last}-1}(\rho_0, \dots, X)$$ to the verifier.
* The verifier computes $$P_{\mathsf{last}-1}(\rho_0, \dots, 0) + \lambda_{\mathsf{last}-1}  Q_{\mathsf{last}-1}(\rho_0, \dots, 0) + P_{\mathsf{last}-1}(\rho_0, …, 1) + \lambda_{\mathsf{last}-1}  Q_{\mathsf{last}-1}(\rho_0, …, 1)$$and compares it with the claim $$S_{\mathsf{last}-2}$$.
* The verifier samples $$\rho_{\mathsf{last}-1}$$ and sends it to the prover.
* The prover sends $$S_{\mathsf{last} - 1} = P_{\mathsf{last}-1}(\rho_0, \dots, \rho_{n-1}) + \lambda_{\mathsf{last} - 1} Q_{\mathsf{last}-1}(\rho_0, \dots, \rho_{n-1})$$ to the verifier.
* The verifier queries $$P_{\mathsf{last}-1}(\bm{X}) + \lambda_{\mathsf{last} - 1}Q_{\mathsf{last} - 1}(\bm{X})$$ at $$(\rho_0, \dots, \rho_{n-1})$$ and compares the result with the $$S_{\mathsf{last}-1}$$.

$$
\widetilde{P_i}(\rho_i) + \lambda_i \widetilde{Q_i}(\rho_i) = 

\sum_{x \in \{0,1\}^{k_i}} \mathsf{eq}(x, \rho_i)[\widetilde{P_{i+1}}(x, 0) \cdot \widetilde{Q_{i+1}}(x, 1) + \widetilde{P_{i+1}}(x, 1) \cdot \widetilde{Q_{i+1}}(x, 0) + \lambda_i \widetilde{Q_{i+1}}(x, 0) \cdot \widetilde{Q_{i+1}}(x, 1) ]
$$

* For the final layer, the prover sends $$P_{last-1}(\rho_0, \dots, X_{n-1}), Q_{last-1}(\rho_0, \dots, X_{n-1})$$ to the verifier.
* The verifier computes $$P_{last-1}(\rho_0, \dots, 0) + \lambda_{last-1} Q_{last-1}(\rho_0, \dots, 0) + P_{last-1}(\rho_0, …, 1) + \lambda_{last-1} Q_{last-1}(\rho_0, …, 1)$$and compares it with the claim $$S_{last-2}$$.
* The verifier samples $$ho_{last-1}$$ and sends it to the prover.
* The prover sends $$P_{last-1}(\rho_0, \dots, \rho_{n-1})$$ to the verifier.
* The verifier compares $$P_{last-1}(\rho_0, \dots, \rho_{n-1})$$ with the claim $$S_{last-1}$$.

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXdd-I_Ack41UZveDHHW_TsLEX-p4BBp-ZZC0osVlOqB4hmU5skQE_hFCRSXPCXIQb2u1ZTE24cT19PvckbTRC4gF_pySXpkSQmE9fewbralqjKn7TS_VqKqP1OpGAU0F0pSngPy?key=i-_YTrjb0_QJkFzBaKFWlQ0U" alt=""><figcaption></figcaption></figure>

The values in the bottom layer will be filled as described in the LogUp-GKR paper. Using the previous LogUp example, it would look as follows:

| (X₀, X₁, Y₀, Y₁) | Source  | P(X₀, X₁, Y₀, Y₁) | Q(X₀, X₁, Y₀, Y₁) |
| ---------------- | ------- | ----------------- | ----------------- |
| (0, 0, 0, 0)     | $$A_0$$ | 1                 | 1 + $$\beta$$     |
| (1, 0, 0, 0)     | $$A_0$$ | 1                 | 2 + $$\beta$$     |
| (0, 1, 0, 0)     | $$A_0$$ | 1                 | 1 + $$\beta$$     |
| (1, 1, 0, 0)     | $$A_0$$ | 1                 | 3 + $$\beta$$     |
| (0, 0, 1, 0)     | $$A_1$$ | 1                 | 1 + $$\beta$$     |
| (1, 0, 1, 0)     | $$A_1$$ | 1                 | 1 + $$\beta$$     |
| (0, 1, 1, 0)     | $$A_1$$ | 1                 | 2 + $$\beta$$     |
| (1 1, 1, 0)      | $$A_1$$ | 1                 | 2 + $$\beta$$     |
| (0, 0, 1, 1)     | $$S$$   | 4                 | 1 + $$\beta$$     |
| (1, 0, 1, 1)     | $$S$$   | 3                 | 2 + $$\beta$$     |
| (0, 1, 1, 1)     | $$S$$   | 1                 | 3 + $$\beta$$     |
| (1, 1, 1, 1)     | $$S$$   | 0                 | $$\beta$$         |

When using LogUp-GKR, if there are n input polynomials referencing $$S(X)$$, only 1 column $$m(X)$$ needs to be committed. Additionally, the issue in LogUp with excessively high degrees of constraints has also been resolved.

## Conclusion

<figure><img src="../../.gitbook/assets/Screenshot 2025-02-06 at 11.12.03 AM.png" alt=""><figcaption></figcaption></figure>

We have reviewed the evolution from Halo2 Lookup to LogUp-GKR. Initially, the process started with univariate polynomials, but from LogUp onward, multivariate polynomials were used. As the protocol evolved, the number of committed polynomials decreased, leading to improved performance, as illustrated in the examples above.

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
