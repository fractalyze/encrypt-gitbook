# Modular Square Root

## Introduction

Suppose we want to find the square root $$x^2 \equiv a \mod p$$, where $$p$$ is an odd prime. That is, we seek some $$x \in \mathbb{Z}_p$$ such that the square of $$x$$ is congruent to $$a$$ modulo $$p$$.

If such an $$x$$ exists, we say that $$a$$ is a [**quadratic residue**](https://mathworld.wolfram.com/QuadraticResidue.html) modulo $$p$$; otherwise, it is a [**quadratic nonresidue**](https://mathworld.wolfram.com/QuadraticNonresidue.html). The trivial case $$a = 0$$ is typically excluded when enumerating quadratic residues, since $$0$$ has a unique square root (itself), and we are often more interested in the structure of the nonzero squares in $$\mathbb{Z}_p^*$$.

For example, when $$p = 7$$, the quadratic residues modulo 7 are:

$$
\begin{align*}
1^2 &\equiv 6^2 \equiv 1 \pmod 7 \\
2^2 &\equiv 5^2 \equiv 4 \pmod 7 \\
3^2 &\equiv 4^2 \equiv 2 \pmod 7
\end{align*}
$$

Thus, the quadratic residues modulo 7 are $$\{1, 2, 4\}$$, and the quadratic nonresidues are $$\{3, 5, 6\}$$.

Computing square roots modulo a prime or composite number is a fundamental operation in number theory with important applications in cryptography and computational mathematics. One example is when constructing elliptic curves or decompressing elliptic curve points from compressed form, one must compute square roots modulo a prime field to recover the y-coordinate from the x-coordinate.

To efficiently determine whether a square root exists and to compute it when it does, number theorists use symbolic tools such as the **Legendre symbol**, **Jacobi symbol**, and **Euler’s criterion**, which we now briefly introduce.

***

### Legendre Symbol

The **Legendre symbol** $$\left( \frac{a}{p} \right)$$ is a notation that encodes whether $$a$$ is a quadratic residue modulo an odd prime $$p$$. It is defined as:

$$
\left( \frac{a}{p} \right) =
\begin{cases}
\;\;\, 0 & \text{if } a \equiv 0 \mod p \\
\;\;\, 1 & \text{if } a \text{ is a quadratic residue mod } p \\
-1 & \text{if } a \text{ is a quadratic nonresidue mod } p
\end{cases}
$$

The Legendre symbol is a **multiplicative function**, and is a key component in [quadratic reciprocity](https://en.wikipedia.org/wiki/Quadratic_reciprocity) and in algorithms like the [Tonelli–Shanks](https://en.wikipedia.org/wiki/Tonelli%E2%80%93Shanks_algorithm) square root algorithm.

***

### Euler’s Criterion

Euler’s criterion provides a direct method to evaluate the Legendre symbol using modular exponentiation:

$$
\left( \frac{a}{p} \right) \equiv a^{\frac{p-1}{2}} \mod p
$$

This gives a simple way to test whether $$a$$ is a square modulo $$p$$:

* If $$a^{\frac{p-1}{2}} \equiv 1 \mod p$$, then $$a$$ is a quadratic residue.
* If $$a^{\frac{p-1}{2}} \equiv -1 \mod p$$, then $$a$$ is a nonresidue.

***

### Generalization to Extension Fields

Euler’s criterion generalizes naturally to extension fields $$\mathbb{F}_{q}$$, where $$q = p^m$$, as follows:

$$
a^{(q - 1)/2} =
\begin{cases}
\;\;\,1 & \text{if } a \in \mathbb{F}_q^* \text{ is a square in } \mathbb{F}_q \\
-1 & \text{otherwise}
\end{cases}
$$

That is, an element $$a \in \mathbb{F}_q^*$$ is a quadratic residue if and only if:

$$
a^{(q - 1)/2} \equiv 1
$$

***

### Jacobi Symbol

The **Jacobi symbol** $$\left( \frac{a}{n} \right)$$ is a generalization of the Legendre symbol to odd composite moduli $$n$$, where $$n = p_1^{e_1} p_2^{e_2} \cdots p_k^{e_k}$$ is the product of odd primes. It is defined as:

$$
\left( \frac{a}{n} \right) = \prod_{i=1}^k \left( \frac{a}{p_i} \right)^{e_i}
$$

Note that, unlike the Legendre symbol, the Jacobi symbol **does not** determine whether $$a$$ is a square modulo $$n$$ when $$n$$ is composite. For example, it is possible that $$\left( \frac{a}{n} \right) = 1$$ even though $$a$$ is not a square mod $$n$$. Nonetheless, the Jacobi symbol is widely used in square root algorithms in composite fields.

***

### Square Root Algorithm Branch

<figure><img src="../../../.gitbook/assets/Screenshot 2025-07-18 at 9.31.25 PM.png" alt=""><figcaption></figcaption></figure>

The diagram above is taken from [https://eprint.iacr.org/2012/685.pdf](https://eprint.iacr.org/2012/685.pdf) and illustrates how different square root algorithms can be selected based on the modulus and the degree of the extension field. In the following subsections, we introduce several of these algorithms in more detail.

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze
