---
description: 'Presentation: https://www.youtube.com/watch?v=dDsAroTRaFI'
---

# Nova

## Problem Statement

How to prove $$y=F^{(n)}(x)$$? The naïve approach would be unrolling this into a single circuit and proving with some SNARK. But this would require a lot of memory and the proof is not incrementally updatable. Also, the verifier's time may depend on $$n$$. To tackle this problem more efficiently, a new approach called incrementally-verifiable computation was introduced in [Val08](https://iacr.org/archive/tcc2008/49480001/49480001.pdf), [BCTV14](https://eprint.iacr.org/2014/595.pdf). In IVC, an augmented circuit is introduced which ensures correct execution of $$F$$ as well as some verifier circuit which can now be used to inductively prove the correct execution of $$F^{(n)}(x)$$.

## Folding R1CS

### Standard R1CS

Given three $$m\times m$$ matrices $$\bm{A},\bm{B},\bm{C}$$ and public input $$\mathsf{x}$$, does there exist witness $$\bm{W}$$ such that: $$\bm{AZ}\circ \bm{BZ}=\bm{CZ}$$ where $$\bm{Z}=(\bm{W},\mathsf{x},1)$$. Here, we use $$\mathsf{x}$$ instead of vector $$\bm{x}$$ for public input since in Nova, it is typically a hash value.

### An attempt to fold R1CS

Suppose prover $$\mathsf{P}$$ holds $$\bm{W}_1$$ and $$\bm{W}_2$$ and verifier $$\mathsf{V}$$ holds $$U_1=(\bm{W}_1,\mathsf{x}_1)$$ and $$U_2=(\bm{W}_2,\mathsf{x}_2)$$.

1. Verifier picks random $$r$$ and send it to the prover.
2. Verifier computes $$\mathsf{x}\leftarrow \mathsf{x}_1 + r\cdot \mathsf{x}_2$$ so the folded instance becomes $$(\bm{A},\bm{B},\bm{C},\mathsf{x})$$.
3. Prover computes the folded witness $$\bm{W}\leftarrow \bm{W}_1 + r\cdot \bm{W}_2$$.

<figure><img src="../../../.gitbook/assets/image (90).png" alt=""><figcaption><p>Figure 1. Folding R1CS. Credit: <a href="https://www.youtube.com/watch?v=ilrvqajkrYY">https://www.youtube.com/watch?v=ilrvqajkrYY</a></p></figcaption></figure>

Unfortunately there exists many $$r$$ such that $$\bm{AZ}\circ \bm{BZ} \neq \bm{CZ}$$ for $$\bm{Z}=(\bm{W},\mathsf{x},1)$$.

$$
\bm{AZ}\circ \bm{BZ}=(\bm{AZ}_1\circ \bm{BZ}_1) + r^2\cdot(\bm{AZ}_2\circ \bm{BZ}_2) + r\cdot (\bm{AZ}_2\circ \bm{BZ}_1 + \bm{AZ}_1 \circ \bm{BZ}_2)
$$

This shows 3 issues:

1. Need to account the cross-term: $$r\cdot (\bm{AZ}_2\circ \bm{BZ}_1 + \bm{AZ}_1 \circ \bm{BZ}_2)$$
2. Mismatch in coefficient of $$\bm{CZ}_2: r^2\cdot \bm{CZ}_2 \neq r\cdot \bm{CZ}_2$$
3. Even $$\bm{Z}\neq \bm{Z}_1 + r\cdot \bm{Z}_2$$ since $$\bm{Z}_1 + r\cdot \bm{Z}_2=(\bm{W},\mathsf{x},1+r\cdot 1)$$

To address first issue, we add error vector $$\bm{E}\in \mathbb{F}^m$$ which absorbs the cross-terms. To handle the second and third issues, we introduce a scalar $$u$$, which absorbs an extra factor of $$r$$ in $$CZ_2$$ and in $$\bm{Z}=(\bm{W},\mathsf{x},1 + r\cdot 1)$$. We refer to the R1CS with these additional terms as **relaxed R1CS**.

### Relaxed R1CS

**Definition 11 from the paper**. Consider a finite field $$\mathbb{F}$$. Let the public parameters consist of size bounds $$m, n, l \in \mathbb{N}$$ where $$m>l$$. The relaxed R1CS structure consists of sparse matrices $$\bm{A,B,C}\in \mathbb{F}^{m\times m}$$ with at most $$n=\Omega(m)$$ non-zero values in each matrix. A relaxed R1CS instance consists of **an error vector** $$\bm{E} \in \mathbb{F}^m$$, **a scalar** $$u\in \mathbb{F}$$, and public inputs and outputs $$\mathsf{x}\in \mathbb{F}^l$$. An instance $$((\bm{A},\bm{B},\bm{C}),(\bm{E},u,\mathsf{x}))$$ is satisfied by a witness $$\bm{W} \in F^{m-l-1}$$ if

$$
\bm{AZ}\circ \bm{BZ}=u\cdot \bm{CZ} + \bm{E}
$$

where $$\bm{Z}=(\bm{W},\mathsf{x},u)$$. In this setup, notice that if $$u=1$$ and $$\bm{E}=\vec{0}$$ then it is same as standard R1CS.

### Folding relaxed R1CS

To fold relaxed R1CS, the prover and verifier additionally compute:

1. $$u\leftarrow u_1 + r\cdot u_2$$
2. $$\bm{E} \leftarrow \bm{E}_1 + r\cdot (\bm{AZ}_1 \circ \bm{BZ}_2 + \bm{AZ}_2 \circ \bm{BZ}_1 - u_1\bm{CZ}_2 - u_2\bm{CZ}_1) + r^2\cdot \bm{E}_2$$
3. Now, the folded instance is $$(\bm{A,B,C,E},u,\mathsf{x})$$.

<figure><img src="../../../.gitbook/assets/image (92).png" alt=""><figcaption><p>Figure 2. Folding relaxed R1CS. Credit: https://www.youtube.com/watch?v=mY-LWXKsBLc</p></figcaption></figure>

Now, you will be able get $$\bm{AZ}\circ \bm{BZ} = u\cdot\bm{CZ} + \bm{E}$$ where $$\bm{Z}=\bm{Z}_1 + r\cdot \bm{Z}_2$$. But there is still a few issues remaining:

1. The verifier has to compute $$\bm{E}$$ which has multiple large matrix multiplications so it is _non-trivial_.
2. Verifier cannot check if $$W$$ is indeed the outcome of $$\bm{W}_1 + r\cdot \bm{W}_2$$ so it cannot be zero-knowledge because verifier needs to know $$\bm{W}_1$$ and $$\bm{W}_2$$ to fully verify.

To circumvent these issues, we use succinct and hiding additive homomorphic commitments to $$\bm{W}$$ and $$\bm{E}$$ in the instance, and treat both $$\bm{W}$$ and $$\bm{E}$$ as the witness. We refer to this variant as **committed relaxed R1CS**.

### Committed Relaxed R1CS

**Definition 12 from the paper.** Consider a finite field $$\mathbb{F}$$ and a commitment scheme $$\mathsf{Com}$$ over $$\mathbb{F}$$. Let the public parameters consist of size bounds $$m, n, l \in \mathbb{N}$$ where $$m > l$$, and commitment parameters $$\mathsf{pp}_W$$ and $$\mathsf{pp}_E$$ for vectors of size $$m$$ and $$m−l−1$$ respectively. The committed relaxed R1CS structure consists of sparse matrices $$\bm{A, B, C} \in \mathbb{F}^{m\times m}$$ with at most $$n = \Omega(m)$$ non-zero entries in each matrix. A **committed relaxed R1CS instance** is a tuple $$(\bm{A,B,C}, \bar{W},\bar{E},u,\mathsf{x})$$, where $$\bar{W}$$ and $$\bar{E}$$ are commitments, $$u \in \mathbb{F}$$, and $$\mathsf{x} \in \mathbb{F}^l$$ are public inputs and outputs. An instance is satisfied by a witness $$(\bm{W},\bm{E}, r_W, r_E ) \in (\mathbb{F}^{m−l−1},\mathbb{F}^m, \mathbb{F},\mathbb{F})$$ if :

1. $$\bar{E} = \mathsf{Com}(\mathsf{pp}_E, E, r_E)$$
2. $$\bar{W} = \mathsf{Com}(\mathsf{pp}_W , W, r_W )$$
3. $$\bm{AZ} \circ \bm{BZ}=u\cdot \bm{CZ} + \bm{E}$$, where $$\bm{Z} = (\bm{W}, \mathsf{x}, \mathsf{u})$$.

### A Folding Scheme for Committed Relaxed R1CS

Suppose prover holds $$(\bm{W}_1,\bm{E}_1,r_{W_1},r_{E_1})$$ and $$(\bm{W}_2, \bm{E}_2, r_{W_2}, r_{E_2})$$. The verifier takes two committed instances $$(\bar{W_1},\bar{E_1},u_1,\mathsf{x}_1)$$ and $$(\bar{W_2},\bar{E_2},\mathsf{u}_2, \mathsf{x}_2)$$. Let $$\bm{Z}_1=(\bm{W}_1,\mathsf{x}_1,u_1)$$ and $$\bm{Z}_2=(\bm{W}_2,\mathsf{x}_2,u_2)$$.

1. Prover sends $$\bar{T}=\mathsf{Com}(\mathsf{pp}_E,\bm{T},r_T)$$ where $$r_T\leftarrow \mathbb{F}$$ and

$$
\bm{T}=\bm{AZ}_1\circ \bm{BZ}_2 + \bm{AZ}_2\circ \bm{BZ}_1 - u_1\cdot \bm{CZ}_2-u_2\cdot \bm{CZ}_1
$$

2. Verifier sends challenge $$r\leftarrow \mathbb{F}$$.
3. Prover and verifier computes folded instance $$(\bar{W}, \bar{E}, u,\mathsf{x})$$ where

$$
\bar{W}\leftarrow \bar{W_1} + r\cdot \bar{W_2} \\ \bar{E}\leftarrow \bar{E_1} + r\cdot \bar{T} + r^2\cdot \bar{E_2} \\ u\leftarrow u_1 + r\cdot u_2\\ x\leftarrow x_1 + r\cdot x_2
$$

4. Prover computes the folded witness $$(\bm{W},\bm{E},r_W,r_E)$$ where

$$
\bm{W}\leftarrow \bm{W}_1 + r\cdot \bm{W}_2 \\ \bm{E}\leftarrow \bm{E}_1 + r\cdot \bm{T} + r^2\cdot\bm{E}_2 \\ r_E\leftarrow r_{E_1}+r\cdot r_T + r^2\cdot r_{E_2} \\ r_W\leftarrow r_{W_1}+r\cdot r_{W_2}
$$

<figure><img src="../../../.gitbook/assets/image (93).png" alt=""><figcaption><p>Figure 3. Folding Commited Relaxed R1CS. Credit: <a href="https://www.youtube.com/watch?v=ilrvqajkrYY">https://www.youtube.com/watch?v=ilrvqajkrYY</a></p></figcaption></figure>

**Proof of completeness.** To prove completeness, we must show that for any valid witness pairs, the folding verifier will accept the folded instance. For the folded witness to satisfy the constraint, we must have:

$$
\bm{A}(\bm{Z}_1 + r\bm{Z}_2)\circ \bm{B}(\bm{Z}_1 + r\cdot \bm{Z}_2)=(u_1 + r\cdot u_2)\cdot \bm{C}(\bm{Z}_1 + r\cdot \bm{Z}_2) + \bm{E}
$$

Distributing, we get:

$$
\bm{AZ}_1\circ \bm{BZ}_1 + r(\bm{AZ}_1\circ \bm{BZ}_2 + \bm{AZ}_2\circ \bm{BZ}_1) + r^2(\bm{AZ}_2 \circ \bm{BZ}_2)=\\u_1\cdot \bm{CZ}_1 + r(u_1\cdot \bm{CZ}_2 + u_2\cdot \bm{CZ}_1) + r^2 \cdot u_2 \cdot \bm{CZ}_2 + \bm{E}
$$

Aggregating by powers of $$r$$, we must have:

$$
(\bm{AZ}_1\circ \bm{BZ}_1 - u_1 \cdot \bm{CZ}_1) + \\r(\bm{AZ}_1\circ \bm{BZ}_2 + \bm{AZ}_2 \circ \bm{BZ}_1 - u_1\cdot \bm{CZ}_2 - u_2\bm{CZ}_1)+\\r^2(\bm{AZ}_2 \circ \bm{BZ}_2 - u_2 \cdot \bm{CZ}_2)\\=\bm{E}
$$

Since $$W_1$$ and $$W_2$$ are satisfying witnesses, we have:

$$
\bm{E}_1 + r\cdot \bm{T} + r^2\cdot \bm{E}_2=\bm{E}
$$

which holds by construction.&#x20;

**Proof of soundness.** To prove soundness, we must show that for any wrongly constructed folded instance will be rejected by the folding verifier. The intuition is we can extract a satisfying witness from the proof if the verification passes. TODO: elaborate how we can extract it.

**Theorem 3 from the paper**. The above construction is a public-coin folding scheme for committed relaxed R1CS with perfect completeness, knowledge soundness, and zero-knowledge.

## From Folding to IVC

We constructed a way to fold two R1CS instances into one but now what? How do we incrementally prove and verify with this? To prove that $$z_n=F^{n}(z_0)$$, for some count $$n$$, initial input $$z_0$$, and output $$z_n$$, we use an augmented constraint system which in addition to invoking $$F$$, performs additional bookkeeping to fold proofs of prior invocations of itself. A simple version of this augmented constraint system would take two committed relaxed R1CS instances $$\mathbb{U}_\mathsf{acc,i}$$ and $$\mathbb{U}_\mathsf{i}$$ as an input where $$\mathbb{U}_\mathsf{acc,i}$$ represents the correct execution of previous invocations and $$\mathbb{U}_\mathsf{i}$$ represents correct execution of $$i$$-th invocation. Mainly, it performs two tasks:

1. (**State transition**) Execute a step of the incremental computation: instance $$\mathbb{U}_\mathsf{i}$$ contains $$z_i$$ which the augmented constraint system uses to output $$z_{i+1} = F(z_i)$$.
2. **(Folding)** Fold $$\mathbb{U}_\mathsf{acc,i}$$ and $$\mathbb{U}_\mathsf{i}$$ into a single R1CS instance $$\mathbb{U}_\mathsf{acc,i+1}$$.

<figure><img src="../../../.gitbook/assets/image (87).png" alt=""><figcaption><p>Figure 4: Simple outline of IVC. <a href="https://youtu.be/h_PU7FZWiQk?t=784">https://youtu.be/h_PU7FZWiQk?t=784</a></p></figcaption></figure>

### IVC over a single curve

First, let's understand how we can construct it over a single curve. What we need to constrain within this $$\mathsf{R1CS}$$ is:

1. Check previous state transitions are correct
   1. $$\mathsf{IsStrict}(\mathbb{U}_\mathsf{i})=\mathsf{True}$$ (checks if error term is 0 and scalar is 1)
   2. Check if $$\mathbb{U_i}.\mathsf{x}=\mathsf{H}(i,z_0,z_i,\mathbb{U}_\mathsf{acc,i})$$. It might be confusing since we don't use hash here normally. To prove $$F^n(z_0)=z_n$$ we would typically have $$\mathsf{x}=(z_0,z_n,n)$$. But for IVC, we need to keep track of the accumulated claim as well so we use hash here to keep the input size constant.&#x20;
2. Apply the state transition and accumulate instances.
   1. Check if $$z_{i+1}=F(z_i)$$
   2. Accumulate into $$\mathbb{U}_\mathsf{acc,i+1}=\mathsf{Fold_V}(\mathbb{U}_\mathsf{i}, \mathbb{U}_\mathsf{acc,i})$$
3. Generate the instance for next state transitions.
   1. Set $$\mathbb{U}_\mathsf{i+1}.\mathsf{x}=\mathsf{H}(i+1,z_0,z_{i+1},\mathbb{U}_\mathsf{acc,i+1})$$
   2. Generate $$\mathbb{U}_\mathsf{i+1}=(\bar{0},\bar{W}_{i+1},\mathsf{x},1)$$ instance which resembles correct execution of this step. Notice that the error term should be 0 and scalar should be 1 since it is not a folded instance.

<figure><img src="../../../.gitbook/assets/image (104).png" alt=""><figcaption><p>Figure 5. In-depth view of the augmented constraint system. <a href="https://youtu.be/h_PU7FZWiQk?t=978">https://youtu.be/h_PU7FZWiQk?t=978</a></p></figcaption></figure>

### IVC Verification

Given a proof $$\pi=((\mathbb{U}_\mathsf{i}, \mathbb{W}_\mathsf{i}),(\mathbb{U}_\mathsf{acc,i}, \mathbb{W}_\mathsf{acc,i}))$$, the IVC verification will do the following:

$$\mathsf{Verify}((i,z_0,z_i),\pi):$$

1. $$\mathsf{IsStrict}(\mathbb{U}_\mathsf{i})=\mathsf{True}$$
2. $$((\mathbb{U}_\mathsf{i}, \mathbb{W}_\mathsf{i}),(\mathbb{U}_\mathsf{acc,i}, \mathbb{W}_\mathsf{acc,i}))$$ satisfy $$\mathsf{R1CS}$$.
3. $$\mathbb{U}_\mathsf{i}.\mathsf{x}=\mathsf{H}(i,z_0,z_i,\mathbb{U}_\mathsf{acc,i})$$

## Conclusion

In proof-of-concept benchmarks, Nova was shown to be 4x faster at proving 10000 recursive SHA256 hashes than Halo2 (KZG) and memory usage is constant with respect to number of iterations in Nova while in Halo2 (KZG), the memory usage increases almost linearly with respect to number of iterations. This result is taken from: [https://hackmd.io/0gVClQ9IQiSXHYAK0Up9hg?view](https://hackmd.io/0gVClQ9IQiSXHYAK0Up9hg?view)

## References

* [zkStudyClub: Supernova (Srinath Setty - MS Research)](https://www.youtube.com/watch?v=ilrvqajkrYY)
* [Nova: Recursive Zero-Knowledge Arguments from Folding Schemes](https://eprint.iacr.org/2021/370.pdf)
* [Revisiting the Nova Proof System - Wilson Nguyen](https://www.youtube.com/watch?v=h_PU7FZWiQk)
* [Revisiting the Nova Proof System on a Cycle of Curves - Wilson Nguyen](https://www.youtube.com/watch?v=l-F5ykQQ4qw)

> Written by [Batzorig Zorigoo](https://app.gitbook.com/u/cO1lUla01ZW0seepO37jMHFTxg42 "mention") of Fractalyze
