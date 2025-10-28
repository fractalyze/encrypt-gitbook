# LogUp\*

## Introduction

[LogUp](logup-gkr.md#logup) was originally designed as a protocol to prove **non-indexed lookup arguments**. In contrast, permutation arguments are defined in the **indexed lookup form**, and one typically reduces indexed lookup to non-indexed lookup in order to apply LogUp.

A non-indexed lookup argument deals with the problem of proving that, for two arrays $$X, T$$ with $$|X| = n, |T| = m$$, the relation $$X \subseteq T$$ holds. In contrast, an indexed lookup argument introduces an **index array** $$I$$ in place of $$X$$ to carry out the proof. (Let's see how $$I$$ looks like later!)

LogUp\* focuses on the following setting:

1. $$m \ll n$$ ⇒ small table
2. $$I$$ consists of $$n$$ small elements.&#x20;
3. $$T$$ consists of $$m$$ large elements ≈ 128-bit

In this situation, the standard reduction requires committing to the array $$I^*T$$ (the [pullback](logup.md#pullback-and-pushforward)). While elliptic curve–based commitments offer some solutions, hash-based commitments do not provide an efficient method. The problem becomes particularly severe when the elements of $$I^*T$$ must be defined over an extension field in order to achieve 128-bit security; the commitment cost grows prohibitively large according to the paper, roughly 6× the cost compared to plain LogUp.

LogUp\* addresses this challenge by proposing an efficient alternative that is **agnostic to both the underlying field and the commitment scheme**.

***

## Background

### Non-indexed lookup

* Pre-commitment: $$X, T, \mathsf{acc}$$ ($$\mathsf{acc}$$, also called the **multiplicity** , indicates how many times each table element in $$T$$ is looked up by $$X$$).
* Claim: $$X \subseteq T$$
* Challenge: $$c$$
* Verification:

$$
\sum_{0 \le i < n}\frac{1}{c - X[i]} = \sum_{0 \le j < m} \frac{\mathsf{acc}[j]}{c - T[j]}
$$

***

### Indexed lookup

* Claim:

$$
(I^* T)[i] \stackrel{?}{=} T[I[i]]
$$

* Example: If $$X = (1,2,1,3,4)$$ and $$T = (1, 2, 3, 4)$$, then in the indexed lookup setting we have $$I = (0,1,0,2,3)$$, so that $$I^*T = X$$.

***

### Reduction: indexed to non-indexed lookup

* Pre-commitment: $$I, T, \textcolor{red}{I^*T}, \mathsf{acc}$$
* Claim: $$(I^*T, I) \subseteq (T, [0, |T|))$$ (or you can say that $$I^*T$$ is claimed to be pullback)
* Challenge: $$c, \gamma$$
* Verification:

$$
\sum_{0 \le i < n}\frac{1}{c - (I^*T[i] + \gamma I[i])}
= \sum_{0 \le j < m} \frac{\mathsf{acc}[j]}{c - (T[j] + \gamma j)}
$$

LogUp\* aims to eliminate the need for committing to $$\textcolor{red}{I^*T}$$ in this reduction.

***

## Protocol Explanation

### Pullback and pushforward

#### Notations

* $$A: \mathbf{n} = [0,n) \to \mathbb{F}$$
* $$B: \mathbf{m} = [0,m) \to \mathbb{F}$$
* $$I: \mathbf{n} \to \mathbf{m}$$
* Pullback:

$$
I^*B[i] = B[I[i]]
$$

* Pushforward:

$$
I_*A[j] = \sum_{i \mid I[i] = j} A[i]
$$

***

#### Lemma 1. Duality

$$
\langle I_*A, B \rangle = \langle A, I^*B \rangle
$$

**Proof**

$$
\langle I_*A, B \rangle = \sum_{0 \le j < m} I_*A[j]B[j]
   = \sum_{0 \le j < m}\left(\sum_{i \mid I[i] = j} A[i]\right)B[j]    = \sum_{0 \le i < n} A[i]B[I[i]] = \langle A, I^*B \rangle
$$

***

**Intuitive example**

Let $$A = (1, 2, 1, 3, 4), B = (1, 2, 3, 4), I = (0, 1, 0, 2, 3)$$. Then $$I^*B = A$$.

If $$A = (a_0, \dots, a_{n-1})$$, then the right hand is computed as follows:

$$
\sum_{0 \le i < n} a_i^2 = 1^2 + 2^2 + 1^2 + 3^2 + 4^2 = 31
$$

If $$B = (b_0, \dots, b_{m-1})$$, then the left hand is computed as follows:

$$
\sum_{0 \le j < m} \left(\sum_{i \mid I[i] = j} a_i\right)b_j = (1+1)\cdot 1 + 2\cdot 2 + 3\cdot 3 + 4\cdot 4 = 31
$$

Both sides match, confirming the duality.

***

### Trading pullback for pushforward

* Pre-commitment: $$I, T$$
* Claim: $$I^*T(r) \stackrel{?}{=} e$$

**Protocol**

1. Commit to $$I_*\mathsf{eq}_r$$, where&#x20;

$$
\mathsf{eq}_r(x) = \prod_{0 \le i < k}(r_i x_i + (1-r_i)(1-x_i)), \quad n < 2^k
$$

2. Observe that&#x20;

$$
I^*T(r) = \langle I^*T, \mathsf{eq}_r \rangle = \langle T, I_* \mathsf{eq}_r \rangle
$$

{% hint style="info" %}
Evaluation of a polynomial $$P$$ in a point $$r = (r_0, \dots, r_{k−1})$$ can be computed as an inner product with the multilinear Lagrange kernel: $$P(r) = \lang P, \mathsf{eq}_r \rang$$.
{% endhint %}

3. Run sumcheck for the claim $$\langle T, I_* \mathsf{eq}_r \rangle \stackrel{?}{=} e$$, obtaining additional evaluation claims $$T(r') \stackrel{?}{=} c, \; I_*\mathsf{eq}_r(r') \stackrel{?}{=} c'$$.
4. Open $$T$$ and $$I_* \mathsf{eq}_r$$.

This protocol proves the evaluation of $$I^*T(r)$$, provided that $$I_*\mathsf{eq}_r$$ was committed correctly. Note that the commitment to $$\textcolor{red}{I^*T}$$ is not performed.

Then we need to check whether $$I_*\mathsf{eq}_r$$ indeed is pushforward.

***

### Pushforward proof with LogUp\*

#### General Case

* Pre-commitment: $$I, T, X, Y$$
* Claim: $$Y$$is claimed to be pushforward as follows:

$$
Y[j] \stackrel{?}{=} I_*X[j] = \sum_{i \mid I[i] = j} X[i]
$$

* Challenge: $$c$$
* Verification:

$$
\sum_{0 \le i < n} \frac{X[i]}{c - I[i]} = \sum_{0 \le j < m} \frac{Y[j]}{c - j}
$$

Intuitively, if $$I[i = 1,5,7] \mapsto j=3$$, then the left-hand side contributes $$\frac{X[1] + X[5] + X[7]}{c-3}$$, while the right-hand side uses $$\frac{Y[3]}{c-3}$$ to represent the same value.

***

#### Our Case

* Pre-commitment: $$I, T, Y$$ (here $$X = \mathsf{eq}_r$$, and since the verifier can compute $$\mathsf{eq}_r$$ directly, there is no need to commit to $$X$$).
* Challenge: $$c$$
* Verification:

$$
\sum_{0 \le i < n}\frac{\mathsf{eq}_r[i]}{c - I[i]}
  = \sum_{0 \le j < m} \frac{I_*\mathsf{eq}_r[j]}{c - j}
$$

Thus, the prover only commits to $$I, T, Y$$, while the verifier computes $$\mathsf{eq}_r$$ on its own to complete the verification.

## References

* [https://eprint.iacr.org/2025/946.pdf](https://eprint.iacr.org/2025/946.pdf)

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
