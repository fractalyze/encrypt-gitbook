# Fflonk

## Introduction

When opening at a single point for $$n$$ polynomials, the common technique is to use a random linear combination, which requires $$n - 1$$ **scalar multiplications. Eliminating these operations could lead to more efficient verification**. Meanwhile, in SHPlonk, when opening KZG in batch, the verifier randomly samples a point $$z \leftarrow \mathbb{F}$$ and constructs a vanishing polynomial $$L(X)$$ for this point. This method allows an opening at $$t$$ points to be converted into an opening at a single point. **However, the verifier still needs to perform** $$O(n)$$ **computations, and reducing this cost could improve verification efficiency.**

A natural question arises: can _"the problem of opening at a single point for multiple polynomials"_ be transformed into _"the problem of opening at multiple points for a single polynomial"_? If possible, then leveraging SHPlonk’s method could enable more efficient batch opening for the verifier.

## Background

Please read [SHPlonk](shplonk.md) beforehand!

## Protocol Explanation

When performing Radix-2 FFT, $$P(X)$$ is decomposed into even and odd parts:

$$
P(X) = P_E(X^2) + X \cdot P_O(X^2)
$$

Now, consider a simple problem of opening at a single point $$x$$ for multiple polynomials $$P_0(X), P_1(X)$$. We define $$P(X)$$ as follows:

$$
P(X) = P_0(X^2) + X \cdot P_1(X^2)
$$

Let $$x = z^2$$, and open $$P(X)$$ at $$\bm{S} = \{z, -z\}$$:

$$
P(z) = P_0(x) + z \cdot P_1(x) \\
P(-z) = P_0(x) - z \cdot P_1(x)
$$

By performing the following computations, we can retrieve $$P_0(x)$$ and $$P_1(x)$$:

$$
P_0(x) = \frac{P(z) + P(-z)}{2} \\ P_1(x) = \frac{P(z) - P(-z)}{2z}
$$

Thus, the original problem is transformed into opening at multiple points $$\bm{S} = \{z, -z\}$$ for a single polynomial $$P(X)$$, allowing us to apply SHPlonk’s batch opening method.

To generalize this approach to $$n$$ polynomials, we need a way to combine $$P_0(X), \dots, P_{n-1}(X)$$ into a single polynomial and derive $$\bm{S}$$ from $$x$$.

### Polynomial Combination and Decomposition

To merge multiple polynomials into one, we define the following functions:

* $$\mathsf{combine}_n(\bm{P}): \mathbb{F}[X]^n \to \mathbb{F}[X]$$:  This constructs $$P(X)$$ from $$P_0(X), \dots, P_{n-1}(X)$$:

$$
P(X) := \sum_{i < n} X^i \cdot P_i(X^n)
$$

* $$\mathsf{decompose}_n(P): \mathbb{F}[X] \rightarrow \mathbb{F}[X]^n$$: This extracts $$P_0(X), \dots, P_{n-1}(X)$$ from $$P(X)$$.

By definition, we have:

$$
\mathsf{decompose}_n(\mathsf{combine}_n(\bm{P})) = \bm{P}
$$

By the way, $$\mathsf{decompose}_n(P)$$ is just introduced, but not actually used in this protocol!

### Finding $$\bm{S}$$ from $$x$$

Next, we define a function to derive the set of opening points $$\bm{S}$$ from $$x$$:

* $$\mathsf{roots}_n(x): \mathbb{F} \to \mathbb{F}^n$$: This determines the point set $$\bm{S}$$:

$$
\bm{S} := (z\omega^i)_{i < n}
$$

where $$z \in \mathbb{F}$$ satisfies:

* $$z^n = x$$
* $$z^i \neq x$$ for $$i < n$$

$$n$$ must be a divisor of $$p - 1$$, and $$\omega$$ is the $$n$$-th root of unity.

For example, if $$n = 4$$, meaning $$\bm{P} = \{P_0(X), P_1(X), P_2(X), P_3(X)\}$$, and we want to open at $$x$$, we construct:

$$
P(X) = P_0(X^4) + X \cdot P_1(X^4) + X^2 \cdot  P_2(X^4) + X^3 \cdot P_3(X^4)
$$

The corresponding root set is:

$$
\bm{S} = \{z, z\omega, -z, -z\omega \}
$$

This is because $$\omega^2 = -1$$ since $$\omega^4 = 1$$.

Opening at $$\bm{S}$$ gives:

$$
P(z) = P_0(x) + z \cdot P_1(x) + z^2 \cdot P_2(x) + z^3 \cdot P_3(x) \\P(z\omega) = P_0(x) + z\omega \cdot P_1(x) - z^2 \cdot P_2(x) - z^3\omega \cdot P_3(x) \\P(-z) = P_0(x) - z \cdot P_1(x) + z^2 \cdot P_2(x) - z^3 \cdot P_3(x) \\P(-z\omega) = P_0(x) - z\omega \cdot P_1(x) - z^2 \cdot P_2(x) + z^3\omega \cdot P_3(x) \\
$$

Remember that $$x = z^4$$ by definition!

By computing:

$$
P_0(x) = \frac{P(z) + P(z\omega) + P(-z) + P(-z\omega)}{4} \\
P_1(x) = \frac{P(z) + P(z\omega)  - P(-z) - P(-z\omega)}{4z} \\  
P_2(x) = \frac{P(z) - P(z\omega) + P(-z) - P(-z\omega)}{4z^2} \\
P_3(x) = \frac{P(z) - P(z\omega)  - P(-z) + P(-z\omega)}{4z^3} \\
$$

we can recover $$P_0(x), P_1(x), P_2(x), P_3(x)$$.

### Opening Multiple Polynomials at Multiple Points

Consider the same example from SHPlonk:

* $$P_0(X)$$ is opened at $$x_0$$ with value $$y_0$$.
* $$P_1(X)$$ is opened at $$x_0, x_1$$ with values $$y_1, y_2$$.
* $$P_2(X)$$ is opened at $$x_0, x_1$$ with values $$y_3, y_4$$.

Let $$\bm{P} = \{P_0, P_1, P_2\}$$, then we compute $$P(X) = \mathsf{combine}_4(\bm{P})$$, determine $$\bm{S}_0 = \mathsf{roots}_4(x_0)$$ and $$\bm{S}_1 = \mathsf{roots}_4(x_1)$$, and perform a batch opening with $$S_0 \cup S_1$$.

Let's set the maximum allowable number of polynomials as $$A$$. Then, if there are $$n < A$$ polynomials of degree $$d$$, and each polynomial is opened at $$t$$ points, the computational costs are as follows:

* **The number of opening sets** $$s$$: $$1 \le s \le n$$
* $$\mathsf{srs}$$ **size**: $$A \cdot (d + 1) \mathbb{G}_1, 2\mathbb{G}_2$$
* **Prover work**: $$O(d)\mathbb{G}_1, O(s \cdot d \log d) \mathbb{F}$$
* **Proof length**: $$2\mathbb{G}_1$$
* **Verifier work**: $$O(1)\mathbb{G}_1, O(n)\mathbb{F}, 2\mathsf{P}$$

Here, $$\mathbb{G}_i$$ represents scalar multiplication in $$\mathbb{G}_i$$​, $$\mathbb{F}$$ represents addition or multiplication in $$\mathbb{F}$$, and $$\mathsf{P}$$ denotes a pairing operation.

## Conclusion

**Fflonk** further optimizes batch opening protocols using a **trick inspired by FFT**, (that's why the name starts with ff, which denotes "fast fourier") reducing verifier work while maintaining prover efficiency and proof size. This makes it particularly well-suited for large-scale applications where minimizing verifier overhead is critical. In [https://ethresear.ch/t/on-the-gas-efficiency-of-the-whir-polynomial-commitment-scheme/21301](https://ethresear.ch/t/on-the-gas-efficiency-of-the-whir-polynomial-commitment-scheme/21301), it says verifying **groth16** and **fflonk** proofs costs are almost identical.

TODO([Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") ): compare the prover performance between groth16 and fflonk.

## References

* [https://eprint.iacr.org/2021/1167](https://eprint.iacr.org/2021/1167)
* [https://ethresear.ch/t/on-the-gas-efficiency-of-the-whir-polynomial-commitment-scheme/21301](https://ethresear.ch/t/on-the-gas-efficiency-of-the-whir-polynomial-commitment-scheme/21301)
