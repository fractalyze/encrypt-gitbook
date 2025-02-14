---
description: 'Presentation: https://youtu.be/JdHqAIMihxg'
---

# WHIR

## Introduction

[**WHIR**](https://eprint.iacr.org/2024/1586.pdf) is a follow-up paper to [STIR](https://eprint.iacr.org/2024/390), and it has advanced in 2 key aspects: **verification speed** and use **of multilinear polynomials** over univariate polynomials. WHIR can replace existing protocols such as FRI, [STIR](https://eprint.iacr.org/2024/390), and [Basefold](https://eprint.iacr.org/2023/1705.pdf).&#x20;

## Background

Please refer to [Basefold](basefold.md) and [STIR](stir.md) in advance.

### Constrained RS (CRS) Code&#x20;

An **RS (Reed-Solomon) code** with field $$\mathbb{F}$$, evaluation domain $$\mathcal{L} \subseteq \mathbb{F}$$, and degree $$d$$ can be interpreted   as evaluations over either a univariate polynomial or a multilinear polynomial as follows:

$$
\begin{align*}
\mathsf{RS}[\mathbb{F}, \mathcal{L}, m] :&= \{ f:\mathcal{L} \rightarrow \mathbb{F} : \exist \hat{g} \in \mathbb{F}^{2^m}[X] \text{ s.t. }\forall x \in \mathcal{L}, f(x) = \hat{g}(x) \} \\
&= \{ f:\mathcal{L} \rightarrow \mathbb{F} : \exist \hat{f} \in \mathbb{F}^{2}[X_1, \dots, X_m] \text{ s.t. }\forall x \in \mathcal{L}, f(x) = \hat{f}(\mathsf{pow}(x, m)) \}
\end{align*}
$$

where $$\mathsf{pow}$$ is defined as:

$$
\mathsf{pow}(x, m) = ({x}^{2^0}, \space \dots, \space {x}^{2^{m - 1}})
$$

A **CRS (Constrained Reed-Solomon) code** extends the traditional RS code by introducing additional evaluation constraint $$\hat{f}(\bm{z})=\sigma$$.

Recall that $$\hat{f}$$ is defined as:

$$
\hat{f}(\bm{X}) = \sum_{\bm{b}\in \{0, 1\}^m} \hat{f}(\bm{b}) \cdot \mathsf{eq}(\bm{b}, \bm{X})
$$

To incorporate the additional constraint $$\hat{f}(\bm{z})=\sigma$$,&#x20;

$$
\hat{f}(\bm{z}) = \sum_{\bm{b} \in \{0, 1\} ^m } \hat{f}(\bm{b}) \cdot \mathsf{eq}(\bm{b}, \bm{z}) = \sum_{\bm{b} \in \{0, 1\}^m} \hat{w}(\hat{f}(\bm{b}), \bm{b})  = \sigma
$$

where $$\hat{w}$$ is defined as: (and we call this weight polynomial, which we'll explain later).

$$
\hat{w}(Z, \bm{X}) = Z \cdot \mathsf{eq}(\bm{X}, \bm{z})
$$

Hence, **CRS (Constrained Reed-Solomon) code** code can be written as:

$$
\mathsf{CRS}[\mathbb{F}, \mathcal{L}, m, \hat{w}, \sigma] := \left\{ f \in \mathsf{RS}[\mathbb{F}, \mathcal{L}, m] : \sum_{\bm{b} \in \{0, 1\}^m} \hat{w}_z(\hat{f}(\bm{b}), \bm{b}) = \sigma \right\}
$$

## Protocol Explanation&#x20;

### Brief Overview

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfCRE8M13zX-eZfRO_qbnfFRFK5x0_xNCHbqTlrx7LXHKfXq-jQm15W9OOOTD92_QAskkJP2oiQzuNzgPS3TO0mtm891xaeF_ZwMhT4LQxhvOv6UI0vnM5l_sIuc-4N3you-D_t?key=tVMHVzVEPMgQ0q8cYYDOHXxW" alt=""><figcaption><p>Fig 1. Simple Diagram of the Query Phase in the STIR (left) and WHIR (right) Protocols</p></figcaption></figure>

The **query complexity** in WHIR remains the same as in STIR because the same idea of reducing the rate is applied. WHIR also uses **Out-Of-Domain Sampling**, which is employed in STIR. However, WHIR does not use **Quotienting**, or **Degree Correction**. Instead, it introduces two new methods, described below:

### Sumcheck

For each round in the [**Sumcheck protocol**](../../primitives/sumcheck.md), the verifier provides a random value, and the prover reduces the number of variables by one. With each variable reduction, the degree is also reduced by one. After $$k$$ rounds of sumcheck, $$k$$ variables can be eliminated, reducing the degree by $$k$$. This is the folding method used in WHIR, differing from the $$k$$-fold approach in STIR.

For example, when $$k = 2$$, the process operates as follows. First, let us assume that the evaluation constraint claimed by the CRS in the previous round is as follows:

$$
\sum_{\bm{b} \in \{0, 1\}^m} \hat{w}(\hat{f}(\bm{b}), \bm{b}) = \sigma
$$

1. The prover provides the verifier with a univariate polynomial $$\hat{h}_0$$ defined as follows to prove the constraint:

$$
\hat{h}_{0}(X) = \sum_{\bm{b} \in \{0, 1\}^{m-1}} \hat{w}(\hat{f}(X, \bm{b}), X, \bm{b})
$$

2. The verifier then checks the following condition and rejects if the two sides are not equal:

$$
\hat{h}_0(0) + \hat{h}_0(1) \stackrel{?}= \sigma
$$

3. The verifier samples a random $$\alpha_0$$​ and sends it to the prover.
4. The prover, using the random value $$\alpha_0$$, provides the verifier with another univariate polynomial $$\hat{h}_1$$₁ defined as follows:

$$
\hat{h}_1(X) = \sum_{\bm{b} \in \{0, 1\}^{m-2}} \hat{w}(\hat{f}(\alpha_0, X, \bm{b}), \alpha_0, X, \bm{b})
$$

5. The verifier checks the following condition and rejects if the two sides are not equal:

$$
\hat{h}_1(0) + \hat{h}_1(1) \stackrel{?}= \hat{h}_0(\alpha_0)
$$

5. The verifier samples a random value $$\alpha_1$$​ and sends it to the prover.
6. The folded polynomial $$\hat{g}$$ can then be constructed as follows:

$$
\hat{g}(\bm{X}) = \hat{f}(\alpha_0, \alpha_1, \bm{X})
$$

### Weight Polynomial

#### Out-Of-Domain Sampling

1. The verifier samples random value $$z_0 \stackrel{\$}\leftarrow \mathbb{F}$$.
2. The prover sends $$y_0 = \hat{g}(\mathsf{pow}(z_0, m - k))$$.

#### Shift queries

1. The verifier samples random values $$z_1, \dots, z_{t_i} \stackrel{\$}\leftarrow \mathcal{L}_i$$. The $$\mathcal{L}_i$$ is the evaluation domain in each round $$i$$, and the value $$t$$ represents the number of queries required in each round $$i$$, determined by the security parameter $$\lambda$$.&#x20;
2. For each $$i \in [t]$$, the verifier obtains $$y_i$$ by querying $$f$$:

$$
y_i = \mathsf{Fold}(f, \alpha_0, \alpha_1 )(z_i)
$$

and $$\mathsf{Fold}$$ is defined as:

$$
\mathsf{Fold}(f, \alpha_i, \dots, \alpha_{j})(y) = \begin{cases}
\mathsf{Fold}(\mathsf{Fold}(f, \alpha_i), \alpha_{i + 1}, \dots, \alpha_{j})(y^2) &\text{ if } i < j \\
\frac{f(x)  + f(-x)}{2} +\alpha_i \cdot \frac{f(x) - f(-x)}{2 \cdot x} & \text { if } i = j
\end{cases}
$$

where $$y = x^2 = (-x)^2$$.

#### Recursive Claims

1. The verifier samples random value $$\gamma \stackrel{\$}\leftarrow \mathbb{F}$$.
2. Using the random values and the computed results, the prover and the verifier constructs $$\hat{w}'$$ and $$\sigma'$$, which will be used in the new $$\mathsf{CRS}[\mathbb{F}, \mathcal{L}, m - k, \hat{w}', \sigma']$$.

$$
\hat{w}'(Z, \bm{X}) := \hat{w}(Z, \alpha_0, \dots, \alpha_{k-1},  \bm{X}) + Z \cdot \sum_{i = 0}^t \gamma^{i+1}\cdot \mathsf{eq}(\bm{X}, \bm{z_i}) \\
\sigma' := \hat{h}_{k-1}(\alpha_{k-1}) + \sum_{i = 0}^t \gamma^{i+1} \cdot y_i
$$

### The Full WHIR Protocol

#### Parameters

* a constrained Reed-Solomon code $$\mathsf{CRS}[\mathbb{F}, \mathcal{L}_0, m_0, \hat{w}_0, \sigma_0]$$;
* an iteration count $$M \in \mathbb{N}$$;
* folding parameter $$k_0, \dots, k_{M-1}$$ such that $$\sum_{i=0}^{M-1}k_i \le m$$;
* evaluation domain $$\mathcal{L}_1, \dots, \mathcal{L}_{M-1} \sube \mathbb{F}$$ where $$\mathcal{L}_i$$ is a smooth coset of $$\mathbb{F}^*$$ with order $$|\mathcal{L}_i| \ge 2^{m_i}$$;
* repetition parameters $$t_0, \dots, t_{M-1}$$ with $$t_i \le |\mathcal{L}_i|$$;
* define $$m_0 := m$$ and $$m_i := m - \sum_{j<i}k_j$$;
* define $$d^* := 1 + \deg_Z(\hat{w}_0) + \max_{i \in [m_0]} \deg_{X_i}(\hat{w}_0)$$ and $$d := \max \{ d^*, 3\}$$.

#### Inputs

The verifier has oracle access to $$f_0: \mathcal{L}_0 \rightarrow \mathbb{F}$$. In the honest case, $$f_0 \in \mathsf{CRS}[\mathbb{F}, \mathcal{L}_0, m_0, \hat{w}_0, \sigma_0]$$ and the prover receives $$\hat{f}_0 \in \mathbb{F}^{< 2} [X_1, \dots, X_m]$$ such that $$f_0 = \hat{f}_0(\mathcal{L}_0)$$ and $$\sum_{\bm{b} \in \{0,1\}^{m_0}} \hat{w}_0(\hat{f}_0(\bm{b}), \bm{b} ) = \sigma_0$$.

#### Query(Prover side)

1. **Initial sumcheck:** Set $$\pmb{\alpha}_0 := \emptyset$$. For $$\ell = 1, \dots, k_0$$:&#x20;
   1. The prover sends $$\hat{h}_{0, \ell} \in \mathbb{F}^{< d^*}[X]$$. In the honest case, $$\hat{h}_{0, \ell} := \sum_{\bm{b} \in \{0, 1\}^{m_0 - \ell - 1}} \hat{w}_0(\hat{f}_0(\alpha_0, X, \bm{b}), \bm{\alpha}_0, X, \bm{b})$$.\

   2. The verifier samples $$\alpha_{0, \ell} \leftarrow \mathbb{F}$$. Append $$\alpha_{0, \ell}$$ to set $$\bm{\alpha}_0$$.
2. **Main loop**: For $$i = 1, \dots, M - 1$$:&#x20;
   1. **Send folded function**: The prover sends $$f_i: \mathcal{L}_i \rightarrow \mathbb{F}$$. In the honest case, $$f_i$$ is the evaluation of $$\hat{f}_i := \hat{f}_{i-1}(\bm{\alpha}_{i-1}, \cdot)$$ over $$\mathcal{L}_i$$.
   2. **Out-of-domain sample**: The verifier sends $$z_{i, 0} \stackrel{\$}\leftarrow \mathbb{F}$$. Set $$\bm{z}_{i, 0} := \mathsf{pow}(z_{i, 0}, m_i)$$.
   3. **Out-of-domain reply**: The prover sends $$y_{i, 0} \in \mathbb{F}$$. In the honest case, $$y_{i, 0} := \hat{f}_i(\mathbb{z}_{i, 0})$$.
   4. **Shift message**: The verifier samples $$z_{i, 1}, \dots, z_{i, t_ {i- 1}} \stackrel{\$}\leftarrow \mathcal{L}_{i-1}^{(2^{k_{i - 1}})}$$ and $$\gamma_i \stackrel{\$}\leftarrow \mathbb{F}$$. Set $$\bm{z}_{i, j} := \mathsf{pow}(z_{i, j}, m_i)$$.
   5. **Sumcheck rounds:** Set $$\pmb{\alpha}_0 := \emptyset$$. For $$\ell = 1, \dots, k_i$$:&#x20;
      1. The prover sends $$\hat{h}_{i, \ell} \in \mathbb{F}^{< d}[X]$$. In the honest case, $$\hat{h}_{i, \ell}(X) := \sum_{\bm{b} \in \{0, 1\}^{m_i - \ell - 1}} \hat{w}_i(\hat{f}_i(\alpha_i, X, \bm{b}), \bm{\alpha}_i, X, \bm{b})$$ where $$\hat{w}_i(Z, X_1, \dots, X_{m_i}) := \hat{w}_{i -1}(Z, \bm{\alpha}_{i-1}, X_1, \dots, X_{m_i}) + Z \cdot \sum_{j = 0}^{t_{i-1}}\cdot \mathsf{eq}(\bm{z}_{i, j} , (X_1, \dots, X_{m_i}))$$.
      2. The verifier samples $$\alpha_{i, \ell} \leftarrow \mathbb{F}$$. Append $$\alpha_{i, \ell}$$ to set $$\bm{\alpha}_i$$.
3. **Send final polynomial**: The prover sends $$\hat{f}_M \in \mathbb{F}^{<2}[X_1, \dots, X_{m_M}]$$. In the honest case $$\hat{f}_M := \hat{f}_{M-1}(\bm{\alpha}_{M-1}, \cdot)$$.
4. **Sample final randomness**: The verifier samples $$r_1^{\mathsf{fin}}, \dots, r_{t_{M-1}}^{\mathsf{fin}} \stackrel{\$}{\leftarrow} \mathcal{L}_{M - 1}^{(2^{k _{M - 1}})}$$.

#### Query(Verifier side)

1. **Check initial sumcheck**:
   1. Check that $$\sum_{b \in \{0, 1\}} \hat{h}_{0,1}(b) = \sigma_0$$.
   2. Check that $$\sum_{b \in \{0, 1\}} \hat{h}_{0, \ell}(b) = \hat{h}_{0, \ell-1}(\alpha_{0, \ell-1})$$ for $$\ell \in \{2, \dots, k_0\}$$.
2. **Check main loop**: For $$i = 1, \dots, M - 1$$:&#x20;
   1. Let $$g_{i-1} := \mathsf{Fold}(f_{i-1}, \bm{\alpha}_{i-1})$$.
   2. Compute the points $$\{g_{i-1}(z_{i, j})\}_{j \in [t_{i-1}]}$$ by querying $$f_{i-1}$$ at the appropriate locations.
   3. Check that $$\sum_{b \in \{0, 1\}} \hat{h}_{i,1}(b) = \hat{h}_{i-1, k_{i-1}}(\alpha_{i-1, k_{i - 1}}) + \gamma_i \cdot y_{i, 0} + \sum_{j=1}^{t_{i-1}}  \cdot g_{i-1}(z_{i, j})$$.
   4. Check that $$\sum_{b \in \{0, 1\}} \hat{h}_{i, \ell}(b) = \hat{h}_{i, \ell -1}(\alpha_{i, \ell - 1})$$ for every $$\ell \in \{2, \dots, k_i\}$$.
3. **Check final polynomial**:
   1. Check that, for every $$\ell \in [t_{M-1}]$$, $$\hat{f}_M(\bm{r}^{\mathsf{fin}}_\ell) = g_{M-1}(r^{\mathsf{fin}}_\ell)$$ where $$\bm{r}_\ell^{\mathsf{fin}} := \mathsf{pow}(r^\mathsf{fin}_\ell, m_M)$$.
   2. For $$i = 1, \dots, M - 1$$ set $$\hat{w}_i(Z, X_1, \dots, X_{m_i}) := \hat{w}_{i-1}(Z, \bm{\alpha}_{i-1}, X_1, \dots, X_{m_i}) + Z \cdot \sum_{j=0}^{t_{i-1}} \gamma_i^{j+1} \cdot \mathsf{eq}(\bm{z}_{i,j}, X_1, \dots, X_{m_i})$$
   3. Check that $$\sum_{\bm{b} \in \{0, 1\}^{m_M}} \hat{w}_{M-1}(\hat{f}_M(\bm{b}), \bm{\alpha}_{M-1}, \bm{b}) = \hat{h}_{M-1, k_{M-1}}(\alpha_{M-1, k_{M-1}})$$.

## Conclusion

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXf8MIKbqeEjTXWgH5mDJLGp42iGfMYRg_tngPKqytFGf-QY2aRFp_zz7pexhst-kzPl-dV5ZsviwQLLWbXQkdwIt1whbOis3FlbDceP7PP6AGJVC4UfzqxX7Vwu-778SnHnLCEL?key=tVMHVzVEPMgQ0q8cYYDOHXxW" alt=""><figcaption><p>Fig 2. Comparison Table among BaseFold, FRI, STIR and WHIR</p></figcaption></figure>

The figure above is a table comparing BaseFold, FRI, STIR, and WHIR. It shows that WHIR, like STIR, has **the lowest query complexity**. Additionally, WHIR also achieves **the lowest verifier time complexity**. (For details on how WHIR's verifier time complexity is calculated, refer to Section 2.1.4, "Verifier Efficiency," in the paper.)&#x20;

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXefS0md7Fw24T3Pg5tHmKDOzCahUz1rL_frT0uZApKor_9l3Jb8A3SBYlxWVuQ5R8KrkKZLP9gyNfS6KZF7vGjEgzqthDY_01ie5FHnZKq7yPs8BS6Dp7pce2Vf6lurDNJNbo0w?key=tVMHVzVEPMgQ0q8cYYDOHXxW" alt=""><figcaption><p>Fig 3. Benchmark Result among FRI, STIR and WHIR</p></figcaption></figure>

The benchmarking results, as derived from the table above, show a similar trend. In the [ZK Summit 12 presentation](https://www.youtube.com/watch?v=iPKzmxLDdII), Eylon Yogev highlighted that WHIR has **lower on-chain gas costs** than **Groth16**. Given that Groth16 is widely recognized for its low on-chain gas costs, this suggests an exciting possibility: we may not need to combine STARK-based proof generation with Groth16 verification in the future. This could mark the beginning of a more efficient era, though significant progress is still needed to make it a reality. For now, Groth16 verification remains the most cost-effective option. For more details, see [this discussion](https://ethresear.ch/t/on-the-gas-efficiency-of-the-whir-polynomial-commitment-scheme/21301).

## References

* [https://gfenzi.io/papers/whir/](https://gfenzi.io/papers/whir/)
