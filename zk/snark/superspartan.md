---
description: 'Presentation: https://youtu.be/4Qvj2ME-Xyg'
---

# SuperSpartan

## Introduction

The SuperSpartan paper introduces [**customizable constraint systems (CCS)**](https://fractalyze.gitbook.io/intro/~/revisions/GBuVDfZJZBNDtoleSgnx/zk/arithmetization/ccs), a generalization of [R1CS](https://fractalyze.gitbook.io/intro/~/revisions/WK2i8xxbUNf7f7bV0E9h/zk/arithmetization/r1cs) that can simultaneously capture [R1CS](https://fractalyze.gitbook.io/intro/~/revisions/WK2i8xxbUNf7f7bV0E9h/zk/arithmetization/r1cs), [Plonkish](https://fractalyze.gitbook.io/intro/~/revisions/WK2i8xxbUNf7f7bV0E9h/zk/arithmetization/plonk), and [AIR](https://fractalyze.gitbook.io/intro/~/revisions/WK2i8xxbUNf7f7bV0E9h/zk/arithmetization/air) without incurring overhead. Unlike existing formulations of Plonkish and AIR, CCS is not tied to any specific proof system. The paper shows that the linear-time polynomial IOP for R1CS from [Spartan](https://fractalyze.gitbook.io/intro/~/revisions/6ILOvDSvW4zAg8fUzupj/zk/snark/spartan) extends naturally to CCS, and that combining this with a polynomial commitment scheme yields a family of SNARKs for CCS, referred to as SuperSpartan.

SuperSpartan supports high-degree constraints without imposing cryptographic costs that scale with constraint degree; only the underlying field operations scale with degree. As in Spartan, it does not rely on superlinear-time or hard-to-distribute operations such as FFTs. While HyperPlonk achieves similar properties for Plonkish through a different technique, the authors observe that it remains unclear how to prove CCS instances—or even R1CS instances—using HyperPlonk or Plonk itself without overhead.

In contrast, SuperSpartan can prove uniform CCS instances, including AIR, without requiring linear-time preprocessing for the verifier and offers “free” addition gates in those settings. **SuperSpartan applied to AIR yields the first SNARK for AIR that features a linear-time prover, transparent and sublinear-time preprocessing, polylogarithmic proof sizes, and plausible post-quantum security**. Notably, SuperSpartan provides a faster prover than existing transparent SNARKs for AIR, often referred to as STARKs.

## Background

1. Multilinear Extensions
2. [CCS](https://fractalyze.gitbook.io/intro/~/revisions/GBuVDfZJZBNDtoleSgnx/zk/arithmetization/ccs)
3. [Sumcheck](../../primitives/sumcheck/)
4. [PCS](../../primitives/commitment-scheme/)
5. Polynomial IOP
6. [Spartan](https://fractalyze.gitbook.io/intro/~/revisions/auBV3JinIQSOc57um3fH/zk/snark/spartan)

## Protocol Explanation

### Polynomial IOP for CCS

This section describes SuperSpartan’s polynomial IOP for CCS. It is a straightforward generalization of polynomial IOP for R1CS in Spartan. As per definition of [CCS](https://fractalyze.gitbook.io/intro/~/revisions/6ILOvDSvW4zAg8fUzupj/zk/arithmetization/ccs),  suppose we are given a CCS structure-instance tuple with size bounds:

$$
((m, n, l, t, q, d), (\bm{M}_0, . . . , \bm{M}_{t−1}, \bm{s}_0, . . . , \bm{s}_{q−1}), \bm{x}).
$$

{% hint style="info" %}
For simplicity, we describe our protocol assuming m and n are powers of two (if this is not the case, we can pad each matrix $$\bm{M}_{i}$$ and the witness vector $$\bm{z}$$ with zeros to extend $$m$$ and $$n$$ to a power of 2 without increasing the prover or verifier’s time in the SNARKs resulting from the protocol below).
{% endhint %}

Interpret the matrices $$\bm{M}_0, . . . , \bm{M}_{t−1}$$ as functions mapping domain $$\{0, 1\}^{\log m} \times\{0, 1\}^{\log n}$$ to $$\mathbb{F}$$ in the natural way. That is, an input in $$\{0, 1\}^{\log m} \times \{0, 1\}^{\log n}$$ is interpreted as the binary representation of an index $$(i, j) \in \{0, 1, . . . , m − 1\} \times \{0, 1, . . . , n − 1\}$$, and the function outputs the $$(i, j)$$th entry of the matrix.\
\
**Theorem 1 (A generalization of Spartan from R1CS to CCS).** For any finite field $$\mathbb{F}$$, there exists a polynomial IOP for $$\mathcal{R}_{CCS}$$, with the following parameters, where $$m \times n$$ denotes the dimensionality of the CCS coefficient matrices, and $$N$$ denotes the total number of non-zero entries across all of the matrices:

* soundness error is $$O((t + d) \log m)/|\mathbb{F}|$$
* round complexity is $$O(\log m + \log n)$$;
* communication complexity is $$O(d \log m + \log n)$$ elements of $$\mathbb{F}$$;
* at the start of the protocol, the prover sends a single $$(\log m − 1)$$-variate multilinear polynomial $$\widetilde{W}$$, and the verifier has a query access to $$t$$ additional $$2 \log m$$-variate multilinear polynomials $$\widetilde{M}_0, \dots , \widetilde{M}_{t−1}$$
* the verifier makes a single evaluation query to each of the polynomials $$\widetilde{W}, \widetilde{M}_{0}, \dots ,\widetilde{M}_{t−1}$$, and otherwise performs $$O(dq + d\log m + \log n)$$ operations over $$\mathbb{F}$$;&#x20;
* the prescribed prover performs $$O(N + tm + qmd \log^2 d)$$ operations over F to compute its messages over the course of the polynomial IOP (and to compute answers to the verifier’s four queries to $$\widetilde{W}, \widetilde{M}_{0}, \dots ,\widetilde{M}_{t−1}$$).

_Remark 10_. The $$O(qmd \log^2 d)$$ field operations term in the prover’s runtime from Theorem 1 involves using FFTs to multiply together $$d$$ different degree-1 polynomials. FFTs are only practical over some finite fields. Moreover, in CCS/AIR/Plonkish instances that arise in practice, $$d$$ rarely, if ever, exceeds 100 and is often as low as $$5$$ or even $$2$$ \[GPR21, BGtRZt23]. Hence, these $$O(d \log^2 d)$$-time FFT algorithms will typically be much slower than the naive $$O(d^2)$$-time algorithm for multiplying together $$d$$ degree-1 polynomials. Using Karatsuba’s algorithm in place of FFTs would also yield a sub-quadratic-in-$$d$$ time algorithm that works over any field.

**Having the verifier evaluate** $$\widetilde{Z}$$ **efficiently**. Let us first assume that the CCS witness $$\bm{w}$$ and $$(1, \bm{x})$$ both have the same length $$n/2$$. Then it is easy to check that the MLE $$\widetilde{Z}$$  of  Z satisfies:

$$
\widetilde{Z}(X_0,\cdot, X_{\log n−1}) = (1 − X_0) \cdot \widetilde{W}(X_1,\cdot, X_{\log n−1}) + X_0 \cdot \widetilde{(1, x)}(X_1,\cdot, X_{\log n−1}) \tag{13}
$$

If the length of $$(1, \bm{x})$$ is less than that of $$\bm{w}$$ (as is the case in the CCS instances resulting from the AIR → CCS transformation of Lemma 3), we replace $$\bm{x}$$ with a padded vector $$\bm{v}$$ that appends zeros to $$\bm{x}$$ until it has the same length as $$\bm{w}$$ (and we henceforth use $$n$$ to denote twice the length of this padded vector). Replacing $$(1, \bm{x})$$ with the padded vector does increase the time required for the verifier to compute $$\widetilde{(1, \bm{x})}(X_1,\dots X_{\log n-1})$$ by more than an additive $$O(\log n)$$ field operations.

**The protocol.** Have the verifier pick $$\bm{\tau}\in\mathbb{F}^{\log m}$$ at random. Similar to [Spartan](spartan/) Theorem 4.1, checking if $$(\mathcal{I}, \bm{w} ) \in \mathcal{R}_{CCS}$$ is equivalent, except for a soundness error of $$\log m/|\mathbb{F}|$$ over the choice of  $$\bm{\tau}\in\mathbb{F}^{\log m}$$, to checking if the following identity holds:

$$
\sum_{\bm{a}\in\{0,1\}^{\log m}}\widetilde{\sf{eq}}(\bm{\tau},\bm{a})\cdot \sum_{i=0}^{q-1}c_i\cdot\prod_{j\in S_i}\bigg(\sum_{\bm{y}\in\{0,1\}^{\log n}}\widetilde{M_j}(\bm{a},\bm{y})\cdot \widetilde{Z}(\bm{y})\bigg)\stackrel{?}=0 \tag{14}
$$

If we denote the inner part as $$g(a) : \mathbb{F}^{\log m}\to\mathbb{F}$$:

$$
g(\bm{a})=\widetilde{\sf{eq}}(\bm{\tau},\bm{a})\cdot \sum_{i=0}^{q-1}c_i\cdot\prod_{j\in S_i}\bigg(\sum_{\bm{y}\in\{0,1\}^{\log n}}\widetilde{M_j}(\bm{a},\bm{y})\cdot \widetilde{Z}(\bm{y})\bigg)
$$

Then the identity becomes a classic sum-check:

$$
\sum_{\bm{a}\in\{0,1\}^{\log m}}g(\bm{a})\stackrel{?}=0
$$

which reduces the task to verifying $$g(\bm{r}_a)$$ at random $$\bm{r}_a\in\mathbb{F}^{\log m}$$. Note that the verifier can evaluate $$\widetilde{\sf{eq}}(\bm{\tau},\bm{r}_a)$$ unassisted in $$O(\log m)$$ field operations with:

$$
\widetilde{\sf{eq}}(\bm{\tau},\bm{r}_a)=\prod_{i=1}^{\log m}\big(\tau_i r_{a,i} +(1-\tau_i)(1-r_{a,i})\big)
$$

With $$\widetilde{\sf{eq}}(\bm{\tau},\bm{r}_a)$$ in hand, $$g(\bm{r}_a)$$ can be computed in $$O(dq)$$ time given the following $$t$$ quantities for $$j\in\{0,1,\dots,t-1\}$$:

$$
\sum_{\bm{y}\in\{0,1\}^{\log n}}\widetilde{M_j}(\bm{a},\bm{y})\cdot \widetilde{Z}(\bm{y})
$$

These $$t$$ quantities can be computed by applying the sum-check protocol $$t$$ more times in parallel, once to each of the following polynomials, where $$j\in\{0, 1, \dots , t − 1\}$$ (to reduce communication costs by a factor of $$t$$, pick a random $$\gamma\in\mathbb{F}$$ and take linear combination with weights given by $$[\gamma^0,\dots, \gamma^{t−1}]$$):

$$
\widetilde{M_j}(\bm{r}_a,\bm{y})\cdot \widetilde{Z}(\bm{y})
$$

To perform the verifier’s final check in this invocation of the sum-check protocol, it suffices for the verifier to evaluate each of the above $$t$$ polynomials at the random vector $$\bm{r}_y$$, which means it suffices for the verifier to evaluate $$\widetilde{M}_j(\bm{r}_a, \bm{r}_y)$$. These evaluations can be obtained via the verifier’s assumed query access to $$\widetilde{M}_{0}, \dots ,\widetilde{M}_{t−1}$$. $$\widetilde{Z}(\bm{r}_y)$$ can be obtained from one query to $$\widetilde{W}$$ and one query to $$\widetilde{(1,\bm{x})}$$ via Equation (13).

<figure><img src="../../.gitbook/assets/image (186).png" alt=""><figcaption></figcaption></figure>

## References

* [Srinath Setty. Spartan: Efficient and general-purpose zkSNARKs without trusted setup. In Proceedings of the International Cryptology Conference (CRYPTO), 2020](https://eprint.iacr.org/2019/550.pdf).
* [Customizable constraint systems for succinct arguments](https://eprint.iacr.org/2023/552) (CCS and SuperSpartan)



> Written by [Batzorig Zorigoo](https://app.gitbook.com/u/cO1lUla01ZW0seepO37jMHFTxg42 "mention") of Fractalyze
