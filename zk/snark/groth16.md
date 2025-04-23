---
description: 'Presentation: https://www.youtube.com/watch?v=1upt6GOdYXk'
---

# Groth16

## Introduction

[Groth16](https://eprint.iacr.org/2016/260.pdf) is a zk-SNARK construction introduced in 2016 by Jens Groth, characterized by its compact proof size of just **3 group elements** and exceptionally fast verification speed. It is widely used in frameworks like [circom](https://docs.circom.io/) and [RISC0](https://dev.risczero.com/terminology#groth16-receipt).

## Background

### Quardratic Arithmetic Protocol (QAP)

For a system where the number of instance variables is $$\ell + 1$$, the number of witness variables is $$m - \ell$$ (thus, the total number of variables is $$m + 1$$), and the number of constraints is $$n$$, [R1CS](https://www.notion.so/o/vlMwaGV8Ukn8S4gDaGYp/s/rwz1ZAZJtK5FHz4Y1esA/~/changes/28/zk/arithmetization/r1cs/~/overview) can be represented as follows:

<figure><img src="../../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

This R1CS system can be reduced to a **Quadratic Arithmetic Program (QAP)**:

$$
(\sum_{i = 0}^m a_i(X) \cdot z_i ) (\sum_{i = 0}^m b_i(X) \cdot z_i) - \sum_{i = 0}^m c_i(X) \cdot z_i = 0 \mod t(X)
$$

where

* $$a_i(X)$$: A polynomial such that $$a_i(x_j) = A_{j, i}$$ where $$0 \le i \le m$$ and $$0 \le j < n$$.&#x20;
* $$b_i(X)$$: A polynomial such that $$b_i(x_j) = B_{j, i}$$ where $$0 \le i \le m$$ and $$0 \le j < n$$.&#x20;
* $$c_i(X)$$: A polynomial such that $$c_i(x_j) = C_{j, i}$$ where $$0 \le i \le m$$ and $$0 \le j < n$$.&#x20;
* $$t(X)$$: $$X^n - 1$$.
* $$z_i$$: A variable, where $$0 \le i \le m$$.

Here, $$\deg a_i(X) = \deg b_i(X) = \deg c_i(X) = n - 1$$.

Then we can define $$h(X)$$ as follows:

$$
h(X) = \frac{(\sum_{i = 0}^m a_i(X) \cdot z_i) (\sum_{i = 0}^m b_i(X) \cdot z_i) - \sum_{i = 0}^m c_i(X) \cdot z_i}{t(X)}
$$

Where $$\deg h(X) = n - 2$$.&#x20;

> In [BKSV20](https://eprint.iacr.org/2020/811) and [GR22](https://geometryresearch.xyz/notebook/groth16-malleability) it is shown that for Groth16 there exists potential malleability on the public input unless certain linear independence requirements are met. An easy way to meet these requirements is to add additional constraints $$z_i​⋅0=0$$ for all public inputs. This is done implicitly by the Arkworks and SnarkJS implementations. (it's taken from [https://xn--2-umb.com/22/groth16/](https://xn--2-umb.com/22/groth16/).) See also [https://geometry.xyz/notebook/groth16-malleability](https://geometry.xyz/notebook/groth16-malleability)!

## Protocol Explanation

### Key Generation

1. Sample the toxic wastes $$\alpha, \beta, \gamma, \delta, x \stackrel{\$}{\leftarrow} \mathbb{F}$$ . They are called toxic wastes which can be used to create fake proofs which will be accepted by the verifier. So they should be discarded right after key generation, or generated through [MPC](https://en.wikipedia.org/wiki/Secure_multi-party_computation)(or [Trusted Setup](https://vitalik.eth.limo/general/2022/03/14/trustedsetup.html)).
2. Create a verifying key $$\mathsf{vk}$$.

$$
\mathsf{vk} = \left([\alpha]_1, [\beta]_2, [\gamma]_2, [\delta]_2, \left\{ \left[ \frac{\beta a_i(x) + \alpha b_i(x) + c_i(x)}{\gamma} \right]_1 \right\}_{i = 0}^{\ell} \right)
$$

3. Create a proving key $$\mathsf{pk}$$.

$$
\mathsf{pk} = \mathsf{vk} + 
\left(
\begin{align*}
&  [\beta]_1, [\delta]_1,  \{ [a_i(x)]_1\}_{i=0}^{m},  \{ [b_i(x)]_1\}_{i=0}^{m},  \{ [b_i(x)]_2\}_{i=0}^{m}, \\
& \left\{ \left[\frac{\beta a_i(x) + \alpha b_i(x) + c_i(x)}{\delta} \right]_1 \right\}_{i = \ell + 1}^{m}, \left\{ \left[ \frac{x^i t(x)}{\delta} \right]_1 \right\}_{i = 0}^{n - 2} 
\end{align*}
\right)
$$

### Computing h(X)

1. Create $$a(X), b(X), c(X)$$:

$$
a(X) = \sum_{j = 0}^{n-1}  L_j(X) (\sum_{i = 0}^{m} A_{j, i} \cdot z_i) \\
b(X) = \sum_{j = 0}^{n-1}  L_j(X) (\sum_{i = 0}^{m} A_{j, i} \cdot z_i) \\
c(X) = \sum_{j = 0}^{n-1}  L_j(X) (\sum_{i = 0}^{m} A_{j, i} \cdot z_i) \\
$$

where $$L_j(X)$$ are lagrange basis polynomials.

2. Evaluate $$a(X), b(X), c(X)$$ on a coset domain:

$$
\bm{a} = (a(g^0), \dots, a(g^{n-1})) \\
\bm{b} = (b(g^0), \dots, b(g^{n-1}))  \\
\bm{c} = (c(g^0), \dots, c(g^{n-1}))
$$

where $$g^i = (\eta \omega)^i$$. The reason why we compute on a coset domain is avoiding division by zero since $$t(\omega^j) = 0$$.&#x20;

3. Evaluate $$\bm{h}$$:

$$
\bm{h} = \left(\frac{a(g^0)b(g^0) - c(g^0)}{t(g^0)}, \dots, \frac{a(g^{n - 1})b(g^{n - 1}) - c(g^{n - 1})}{t(g^{n-1})}\right)
$$

4. Compute $$h(X)$$:

$$
h(X) = \sum_{i = 0}^{n-1}  {L'}_i(X)  \frac{a(g^i)b(g^i) - c(g^i)}{t(g^i)}
$$

where $${L'}_i(X)$$ are lagrange basis polynomials on a coset domain.

To compute $$h(X)$$, the costs are:

* **3 IFFT operations** of length $$n$$
* **3 FFT on a coset domain operation** of length $$n$$
* **1 evaluations** of length $$n$$
* **IFFT on a coset domain operations** of length $$n - 2$$ (This step can be removed by using [Jordi's Trick](groth16.md#jordis-trick).)

_NOTE: (I)FFT operations on a coset domain implicitly include additional linear operations proportional to the domain size. The (I)FFT operations on a coset domain can be removed by using_ [_Jordi's Trick_](groth16.md#jordis-trick)_._

### Proof Generation

Sample $$r, s \stackrel{\$}{\leftarrow} \mathbb{F}$$ and generate the proof $$\pi = ([A]_1, [B]_2, [C]_1)$$.

$$
[A]_1 = [\alpha]_1 + \sum_{i=0}^m z_i [a_i(x)]_1 + r [\delta]_1    \\
[B]_2 = [\beta]_2 + \sum_{i=0}^m z_i [b_i(x)]_2 + s [\delta]_2 \\
[C]_1 = \sum_{i = \ell + 1}^{m} z_i \left[ \frac{\beta  a_i(x) + \alpha  b_i(x) + c_i(x)}{\delta} \right]_1 + \sum_{i = 0}^{n - 2} h_i \left[ \frac{x^i t(x)}{\delta} \right]_1 + s[A]_1 + r[B]_1 - rs[\delta]_1
$$

where $$h_i$$ are the coefficients of the $$h(X)$$.

To generate a proof $$\pi$$, the costs are:

* For computing $$[\mathbb{A}]_1$$:
  * **1 MSM operation** of length $$m + 1$$ in $$\mathbb{G}_1$$
* For computing $$[\mathbb{B}]_2$$:
  * **1 MSM operation** of length $$m + 1$$ in $$\mathbb{G}_2$$
* For computing $$[\mathbb{C}]_1$$:
  * **1 MSM operation** of length $$m - \ell$$ in $$\mathbb{G}_1$$
  * **1 MSM operation** of length $$n − 1$$ in $$\mathbb{G}_1$$
  * **1 MSM operation** of length $$m + 1$$ in $$\mathbb{G}_1$$ (for $$[B]_1$$)

### [Jordi](https://x.com/jbaylina)'s Trick

To compute $$\sum_{i = 0}^{n - 2} h_i \left[ \frac{x^i t(x)}{\delta} \right]_1$$ of the proof efficiently, consider the following setup:

$$
\begin{align*}
\sum_{i = 0}^{n - 2} h_i \left[ \frac{x^i t(x)}{\delta} \right]_1 &= \left[ \frac{h(x)t(x)}{\delta} \right]_1 \\
&= \left[ \frac{a(x)b(x) - c(x)}{\delta} \right]_1 \\
&= \sum_{i = 0}^{2n - 2} (a({\omega'}^i)b({\omega'}^i) - c({\omega'}^i)) \left[ \frac{{L'}_i(x)}{\delta} \right]_1
\end{align*}
$$

Where $$\omega'$$ be a $$2n$$-th root of unity and $$\omega$$ an $$n$$-th root of unity. These satisfy:

$$
({\omega'}^i)^{2n} = ({\omega'}^{2i})^n = (\omega^i)^n= 1
$$

and $${L'}_i(X)$$ are lagrange basis polynomials.

**Properties of** $$a(X)b(X) - c(X)$$

$$\sum_{i = 0}^{2n - 2} (a({\omega'}^i)b({\omega'}^i) - c({\omega'}^i)) \left[ \frac{{L'}_i(x)}{\delta} \right]_1$$ can be divided into evaluations at even powers and odd powers.

$$
\begin{align*}
\sum_{i = 0}^{2n - 2} (a({\omega'}^i)b({\omega'}^i) - c({\omega'}^i)) \left[ \frac{{L'}_i(x)}{\delta} \right]_1 &= \sum_{i = 0}^{n - 1} (a({\omega'}^{2i})b({\omega'}^{2i}) - c({\omega'}^{2i})) \left[ \frac{{L'}_{2i}(x)}{\delta} \right]_1 \\
& + \sum_{i = 0}^{n - 2} (a({\omega'}^{2i + 1})b({\omega'}^{2i + 1}) - c({\omega'}^{2i + 1})) \left[ \frac{{L'}_{2i + 1}(x)}{\delta} \right]_1
\end{align*}
$$

1. **Evaluation at even powers of** $$\omega'$$**:** For $$0 \le j < n$$,  evaluates to zero at even powers of $$\omega'$$:

$$
a({\omega'}^{2j})b({\omega'}^{2j}) - c({\omega'}^{2j}) = a({\omega}^{j})b({\omega}^{j}) - c({\omega}^{j}) = 0
$$

This implies:

$$
\sum_{i = 0}^{n - 1} (a({\omega'}^{2i})b({\omega'}^{2i}) - c({\omega'}^{2i})) \left[ \frac{{L'}_{2i}(x)}{\delta} \right]_1 = 0
$$

2. **Evaluation at odd powers of** $$\omega'$$: For $$0 \le j < n$$, given $$p(X) = p_0 + p_1X + \cdots + p_{n-1}X^{n-1}$$, we can compute the evaluations at odd powers of $$\omega'$$:&#x20;

$$
p({\omega'}^{2j + 1}) = \sum_{i =0}^{n-1} p_i ({\omega'}^{2j + 1})^i = \sum_{i=0}^{n-1} p_i {\omega'}^i ({\omega'}^{2j})^i = \sum_{i=0}^{n-1} p_i {\omega'}^i ({\omega}^{j})^i = p'({\omega}^{j})
$$

Where $$p'(X) = p_0 + p_1{\omega'}X + \cdots + p_{n-1}{\omega'}^{n-1}X^{n-1}$$.

This implies:

$$
\sum_{i = 0}^{n - 2} (a({\omega'}^{2i + 1})b({\omega'}^{2i + 1}) - c({\omega'}^{2i + 1})) \left[ \frac{{L'}_{2i + 1}(x)}{\delta} \right]_1 = \sum_{i = 0}^{n - 2} (a'({\omega}^{i})b'({\omega}^{i}) - c'({\omega}^{i})) \left[ \frac{{L'}_{2i + 1}(x)}{\delta} \right]_1
$$

Thus, the evaluations of $$a(X)b(X) - c(X)$$ at odd powers of $$\omega'$$ can be efficiently computed by combining FFT results of $$a'(X), b'(X), c'(X)$$.

**Optimizing** $$h(X)$$ **in the Proving Key**

To eliminate the IFFT of $$h(X)$$, the term $$\left\{ \left[ \frac{x^i t(x)}{\delta} \right]_1 \right\}_{i = 0}^{n - 2}$$ in the proving key (pk) is replaced with $$\left\{ \left[ \frac{{L'}_{2i + 1}(x)}{\delta} \right] \right\}_{i = 0}^{n - 2}$$:

$$
\sum_{i = 0}^{n - 2} h_i \left[ \frac{x^i t(x)}{\delta} \right]_1 \rightarrow \sum_{i = 0}^{n - 2} (a'({\omega}^i)b'({\omega}^i) - c'({\omega}^i)) \left[ \frac{{L'}_{2i + 1}(x)}{\delta} \right]_1
$$

Then computing $$h(X)$$ is performed as follows:

1. Create $$a(X),b(X),c(X)$$:&#x20;

$$
a(X) = \sum_{j = 0}^{n-1}  L_j(X) (\sum_{i = 0}^{m} A_{j, i} \cdot z_i) \\
b(X) = \sum_{j = 0}^{n-1}  L_j(X) (\sum_{i = 0}^{m} A_{j, i} \cdot z_i) \\
c(X) = \sum_{j = 0}^{n-1}  L_j(X) (\sum_{i = 0}^{m} A_{j, i} \cdot z_i) \\
$$

where $$L_j(X)$$ are lagrange basis polynomials.

2. Create $$a'(X),b'(X),c'(X)$$:

$$
a'(X) = \sum_{i =0}^{n-1} a_i {\omega'}^i X^i \\ b'(X) = \sum_{i =0}^{n-1} b_i {\omega'}^i X^i \\ c'(X) = \sum_{i =0}^{n-1} c_i {\omega'}^i X^i
$$

where $$\omega'$$ is $$2n$$-th root of unity.

3. Evaluate $$a'(X), b'(X), c'(X)$$:

$$
\bm{a'} = (a'({\omega}^0), \dots, a'({\omega}^{n-1})) \\\bm{b'} = (b'({\omega}^0), \dots, b'({\omega}^{n-1})) \\\bm{c'} = (c'({\omega}^0), \dots, c'({\omega}^{n-1}))
$$

where $$\omega$$ is $$n$$-th root of unity.

4. Evaluate $$\bm{h}$$:

$$
\bm{h} = (a'({\omega}^0)b'({\omega}^0) - c'({\omega}^0), \dots, a'({\omega}^{n - 1})b'({\omega}^{n - 1}) - c'({\omega}^{n - 1}))
$$

To compute $$\bm{h}$$, the costs are:

* **3 IFFT operations** of length $$n$$
* **3 FFT operation** of length $$n$$
* **1 evaluations** of length $$n$$

### Proof Verification

The verifier checks the proof using the pairing equation: (Note that the right hand side will be precomputed.)

$$
e([A]_1, [B]_2) \cdot e(\sum_{i=0}^\ell z_i \left[ \frac{\beta a_i(x) + \alpha b_i(x) + c_i(x)}{\gamma} \right]_1, [-\gamma]_2) \cdot e([C]_1, [-\delta]_2) \stackrel{?}=e([\alpha]_1, [\beta]_2)
$$

**Proof.** Considering only the exponent part, the equation simplifies as follows:

$$
\begin{align*}
AB &= (\alpha + \sum_{i = 0}^m z_i a_i(x) + r \delta) (\beta + \sum_{i = 0}^m z_i b_i(x) + s \delta) \\
 &= \alpha \beta +  \sum_{i = 0}^m z_i (\beta a_i(x) + \alpha b_i(x) ) + \underbrace{(\sum_{i = 0}^m z_i a_i(x))(\sum_{i = 0}^m z_i b_i(x))}_{\sum_{i=0}^m z_i c_i(x) + h(x) t(x)} + s \delta A + r \delta B - rs\delta^2 \\
 &= \alpha \beta +  \sum_{i = 0}^m z_i (\beta a_i(x) + \alpha b_i(x) + c_i(x)) + h(x)t(x) + s \delta A + r \delta B - rs\delta^2 \\
 &= \alpha \beta +  \sum_{i = 0}^l z_i (\beta a_i(x) + \alpha b_i(x) + c_i(x)) + \sum_{i = l + 1}^m z_i (\beta a_i(x) + \alpha b_i(x) + c_i(x)) + h(x) t(x) + s \delta A + r \delta B - rs\delta^2  \\
 &= \alpha \beta +  \sum_{i = 0}^l z_i (\beta a_i(x) + \alpha b_i(x) + c_i(x)) + \delta \underbrace{\left( \frac{\sum_{i = l + 1}^{m} z_i (\beta a_i(x) + \alpha b_i(x) + c_i(x))}{\delta} + \frac{h(x)t(x)}{\delta} + s A + r B - rs\delta \right)}_{C} 
\end{align*}
$$

To verify a proof, the costs are:

* **1 MSM operation** of length $$\ell$$ in $$\mathbb{G}_1$$
* **3 pairing operations**

#### Batch Proof Verification

Assume there are $$n$$ proofs. We denote each proof $$\pi_i$$ for $$0 \leq i < n$$ as follows:

$$
\pi_i = ([A_i]_1, [B_i]_2, [C_i]_1)
$$

Using the random linear combination trick with $$\theta_i \stackrel{\$}\leftarrow \mathbb{F}$$, you can verify proofs in a batch:

$$
\prod_{i=0}^{n-1} \left(e(\theta_i[A_i]_1, [B_i]_2) \right) \cdot e(\sum_{i=0}^{n-1} \theta_i \left(\sum_{j=0}^{\ell}z_j \left[ \frac{\beta a_j(x) + \alpha b_j(x) + c_j(x)}{\gamma} \right]_1\right), [\gamma]_2) \cdot e(\sum_{i=0}^{n-1}\theta_i[C_i]_1, [-\delta]_2) \stackrel{?}= \left(\prod_{i=0}^{n-1}\theta_i\right) e([\alpha]_1, [\beta]_2)
$$

**Proof.**

From the single verification case, for $$0 \leq i < n$$, we know that the following equation holds:

$$
A_iB_i = \alpha \beta +  \sum_{j = 0}^l z_j (\beta a_j(x) + \alpha b_j(x) + c_j(x)) + \delta C_i
$$

Multiplying both sides by $$\theta_i$$ gives:

$$
\theta_iA_iB_i = \theta_i \alpha \beta +  \theta_i \left(\sum_{j = 0}^l z_j (\beta a_j(x) + \alpha b_j(x) + c_j(x))\right) + \theta_i \delta C_i
$$

Summing over $$i = 0$$ to $$n - 1$$, we obtain:

$$
\sum_{i=0}^{n-1}\theta_i A_iB_i = \left(\sum_{i=0}^{n-1}\theta_i\right) \alpha \beta + \sum_{i=0}^{n-1}\theta_i \left(\sum_{j = 0}^l z_j (\beta a_j(x) + \alpha b_j(x) + c_j(x))\right) + \sum_{i=0}^{n-1}\theta_i \delta C_i
$$

These are precisely the exponent parts in the batch verification equation.

To verify $$n$$ proofs, the costs are:

* $$n$$ **MSM operations** of length $$\ell$$ in $$\mathbb{G}_1$$
* $$3n + 1$$ **scalar multiplications** in $$\mathbb{G}_1$$
* $$n - 1$$ **field multiplications**
* $$n + 2$$ **pairing operations**

If each proof were verified individually, the process would require $$3n$$ expensive pairing operations. However, batch verification significantly reduces this cost.

### Malleability

**Malleability** is a property of cryptographic systems or protocols where an attacker can modify a valid message or ciphertext into another valid message or ciphertext without needing to know the original plaintext. The modified message still appears valid under the cryptographic scheme, which can lead to vulnerabilities and unintended consequences.

#### **Attack 1**

An attacker selects $$y \stackrel{\$}{\leftarrow} \mathbb{F}$$ and modifies the proof $$\pi' = ([A']_1, [B']_2, [C']_1)$$ as follows:

$$
[A']_1 = y[A]_1 \\ [B']_2 = y^{-1}[B]_2 \\ [C']_1 = [C]_1
$$

Even though the modified proof $$\pi'$$ differs from the original, it remains valid because $$A'B' = AB$$.

#### **Attack 2**

An attacker selects $$y \stackrel{\$}{\leftarrow} \mathbb{F}$$ and modifies $$\pi' = ([A']_1, [B']_2, [C']_1)$$ as folows:

$$
[A']_1 = [A]_1 \\ [B']_2 = [B]_2 + y \delta \\ [C']_1 = [C]_1 + y [A]_1
$$

**Proof.** The validity of the modified proof $$\pi'$$ can be shown as:

$$
\begin{align*} A'B' &= A(B + y\delta) \\ &= AB + y\delta A \\ &= \alpha\beta + \sum_{i = 0}^l z_i (\beta a_i(x) + \alpha b_i(x) + c_i(x)) + \delta C + y \delta A \\ &= \alpha\beta + \sum_{i = 0}^l z_i (\beta a_i(x) + \alpha b_i(x) + c_i(x)) + \delta \underbrace{(C + y A)}_{C'} \end{align*}
$$

#### **Combining Attacks**

By combining the two attacks above, a fresh proof for the same statement can be constructed, which is indistinguishable from the existing proofs. (The attacks above contains the same part of the proof.)  The modified proof $$\pi' = ([A']_1, [B']_2, [C']_1)$$ is computed as follows (see [GM17](https://eprint.iacr.org/2017/540), [BKSV20](https://eprint.iacr.org/2020/811.pdf) and the [Tachyon implementation](https://github.com/kroma-network/tachyon/blob/e7b13063c6f110b9851655cca740ee62ffb41137/tachyon/zk/r1cs/groth16/prove.h#L240-L295)):

$$
[A']_1 = \frac{1}{r_1}[A]_1 \\ [B']_2 = r_1 [B]_2 + r_1r_2[\delta]_2 \\ [C']_1 = [C]_1 +r_2 [A]_1
$$

**Proof.** The modified proof $$\pi'$$ remains valid:

$$
\begin{align*} A'B' &= \frac{1}{r_1}A(r_1B + r_1r_2\delta) \\ &= AB + r_2 \delta A \\ &= \alpha\beta + \sum_{i = 0}^l z_i (\beta a_i(x) + \alpha b_i(x) + c_i(x)) + \delta C + r_2 \delta A \\ &= \alpha\beta + \sum_{i = 0}^l z_i (\beta a_i(x) + \alpha b_i(x) + c_i(x)) + \delta \underbrace{(C + r_2 A)}_{C'} \end{align*}
$$

## Conclusion

Groth16 is a highly efficient zk-SNARK protocol that combines succinctness, fast verification, and strong zero-knowledge guarantees. Its compact proof size of just 3 group elements and (almost) constant-time verification make it a preferred choice for various privacy-preserving applications or proof compressions(STARK to SNARK).

## References

* [https://ethresear.ch/t/transaction-malleability-attack-of-groth16-proof/15881](https://ethresear.ch/t/transaction-malleability-attack-of-groth16-proof/15881)
* [https://xn--2-umb.com/22/groth16/](https://xn--2-umb.com/22/groth16/)
* [https://geometry.xyz/notebook/the-hidden-little-secret-in-snarkjs](https://geometry.xyz/notebook/the-hidden-little-secret-in-snarkjs)

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
