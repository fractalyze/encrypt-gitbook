# Generalized Shanks Algorithm for Quadratic Extension Field

This section explains how to compute square roots in the quadratic extension field $$\mathbb{F}_{p^2}$$, particularly when the base field $$\mathbb{F}_p$$ satisfies $$p \equiv 3 \pmod 4$$. The method is based on _Algorithm 9_ from [ePrint 2012/685](https://eprint.iacr.org/2012/685.pdf).

***

## Goal

Given $$a \in \mathbb{F}_{p^2}$$, find $$x \in \mathbb{F}_{p^2}$$ such that:

$$
x^2 = a
$$

where $$x = x_0 + x_1 \cdot i$$ and $$i^2 = -1$$.

***

## Algorithm

The algorithm proceeds as follows:

1. Compute $$a_1 = a^{(p - 3)/4}$$
2. Compute $$\alpha = a_1^2 \cdot a = a^{(p-1)/2}$$
3. Check whether $$\alpha^{p + 1} = a^{(p^2-1)/2} = -1$$. If so, then $$a$$ is a **non-residue**, and no square root exists.
4. Compute $$\beta = a_1 \cdot a = a^{(p+1)/4}$$
5. If $$\alpha = -1$$, return $$x = \beta \cdot i = a^{(p+1)/4} \cdot i$$
6. Otherwise:
   * Compute $$b = (\alpha + 1)^{(p - 1)/2} =  \left(1 + a^{(p-1)/2} \right)^{(p - 1)/2}$$
   * Return $$x = b \cdot \beta = \left( 1 + a^{(p-1)/2} \right)^{(p - 1)/2} \cdot a^{(p+1)/4}$$

***

### Intuition Behind the Algorithm

To compute a square root of $$a \in \mathbb{F}_{p^2}$$, we can find an element $$b \in \mathbb{F}_{p^2}$$ and an **odd integer** $$s$$ such that:

$$
b^2 \cdot a^s = 1
$$

In this case, a square root of $$a$$ is given by:

$$
x = b \cdot a^{(s + 1)/2}
$$

because:

$$
x^2 = \left( b \cdot a^{(s+1)/2} \right)^2 = b^2 \cdot a^{s+1} = (b^2 \cdot a^s) \cdot a = a
$$

To find such $$b$$ and $$s$$, we set:

* $$s = \frac{p - 1}{2}$$
* $$b = \left( 1 + a^{(p-1)/2} \right)^{(p - 1)/2}$$

#### Case 1: $$b \ne 0$$

We can verify:

$$
\begin{aligned}
b^2 \cdot a^s
&= \left(1 + a^{(p-1)/2}\right)^{(p-1)} \cdot a^{(p-1)/2} \\
&= \left(1 + a^{(p-1)/2}\right)^p \cdot \left(1 + a^{(p-1)/2}\right)^{-1} \cdot a^{(p-1)/2} \\
&= \left(1 + a^{p(p-1)/2}\right) \cdot \left(1 + a^{(p-1)/2}\right)^{-1} \cdot a^{(p-1)/2} \\
&= \left(a^{(p-1)/2} + 1\right) \cdot \left(1 + a^{(p-1)/2}\right)^{-1} = 1
\end{aligned}
$$

So, $$x = b \cdot a^{(p+1)/4}$$ is a valid square root of $$a$$.

**Explanations**

This step works due to the properties of the [Frobenius endomorphism](https://en.wikipedia.org/wiki/Frobenius_endomorphism), which states that:

$$
\left(1 + a^{(p-1)/2}\right)^p = 1 + a^{p \cdot (p-1)/2}
$$

since exponentiation by $$p$$ acts as an [automorphism](../../abstract-algebra/group/morphisms.md#automorphism-an-isomorphism-from-an-object-to-itself.-equivalently-it-is-a-bijective-endomorphism) over $$\mathbb{F}_{p^2}$$, and in particular, fixes elements in the base field $$\mathbb{F}_p$$.

In addition, every nonzero element $$a \in \mathbb{F}_{p^2}$$ satisfies:

$$
a^{p^2 - 1} = 1
$$

because the multiplicative group $$\mathbb{F}_{p^2}^*$$ has order $$p^2 - 1$$. This ensures that all powers used in the algorithm, including $$a^{(p^2 - 1)/2}$$, are well-defined and lie within the group.

#### Case 2: $$b = 0$$

Then, by definition of $$b$$, we must have:

$$
1 + a^{(p-1)/2} = 0 \quad \Rightarrow \quad a^{(p-1)/2} = -1
$$

In this case, define $$x = a^{(p+1)/4} \cdot i$$, where $$i^2 = -1$$. Then:

$$
\begin{aligned}
x^2 &= \left( a^{(p+1)/4} \cdot i \right)^2 = a^{(p+1)/2} \cdot i^2 \\
&= a^{(p+1)/2} \cdot (-1) = a \cdot a^{(p-1)/2} \cdot (-1) = a \cdot (-1) \cdot (-1) = a
\end{aligned}
$$

Thus, $$x$$ is again a valid square root of $$a$$.

***

### Final Formula

We can summarize the square root of $$a \in \mathbb{F}_{p^2}$$, for $$p \equiv 3 \pmod 4$$, as:

$$
x =
\begin{cases}
a^{(p+1)/4} \cdot i & \text{if } a^{(p-1)/2} = -1 \\
\left( 1 + a^{(p-1)/2} \right)^{(p - 1)/2} \cdot a^{(p+1)/4} & \text{otherwise}
\end{cases}
$$

This elegant formula generalizes the classic Shanks algorithm from $$\mathbb{F}_p$$ to quadratic extensions $$\mathbb{F}_{p^2}$$, and avoids iterative procedures entirely.

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze
