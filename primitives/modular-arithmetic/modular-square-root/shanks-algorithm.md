# Shanks Algorithm

When the prime modulus $$p$$ satisfies $$p \equiv 3 \pmod 4$$, computing the square root of a quadratic residue $$a \in \mathbb{F}_{p^m}$$ becomes remarkably simple when $$m$$ is **odd**. This is because the field size $$q = p^m$$ then satisfies $$q \equiv 3 \pmod 4$$, enabling a shortcut formula for extracting square roots. The method is based on Algorithm 2 from [ePrint 2012/685](https://eprint.iacr.org/2012/685.pdf).

***

## Goal

Find $$x \in \mathbb{F}_{p^m}$$ such that:

$$
x^2 = a
$$

if $$a$$ is a quadratic residue.

***

## Original Algorithm

The algorithm consists of the following steps:

1. Compute $$a_1 = a^{(p - 3)/4}$$
2. Compute $$a_0 = a_1^2 \cdot a = a^{(p-1) / 2}$$
3. If $$a_0 = -1$$, then $$a$$ is a quadratic nonresidue
4. Otherwise, return $$x = a_1 \cdot a = a^{(p + 1)/4}$$

This version performs:

* 1 exponentiation,
* 1 squaring,
* 2 multiplications.

***

## Modified Algorithm

A simplified version is often used in practice:

1. Compute $$x = a^{(p + 1)/4}$$
2. If $$x^2 = a$$, return $$x$$
3. Otherwise, $$a$$ is a quadratic nonresidue

This version only requires:

* 1 exponentiation,
* 1 squaring

and defers the residuosity check to a final comparison.

***

### Why It Works (when $$m = 1$$)

To understand why this shortcut is valid, consider the case where $$a \in \mathbb{F}_p$$ and $$p \equiv 3 \pmod 4$$. Suppose $$x$$ is a square root of $$a$$, i.e.,

$$
x^2 = a
$$

Then,

$$
x^4 = (x^2)^2 = a^2
$$

We know:

$$
a^2 = a^{p+1} \mod p
$$

But since $$p \equiv 3 \pmod 4$$,  taking the 4th root of both sides gives:

$$
x = a^{(p+1)/4}
$$

Therefore, $$x = a^{(p+1)/4}$$ is indeed a valid square root of $$a \mod p$$ — if $$a$$ is a quadratic residue.

***

### Why It Requires $$m$$ to Be Odd

In the general case over $$\mathbb{F}_{p^m}$$, we require the field size $$q = p^m$$ to satisfy $$q \equiv 3 \pmod 4$$ for the exponent $$(q+1)/4$$ to be an integer and the formula to hold. This congruence only holds when:

* $$p \equiv 3 \pmod 4$$, and
* $$m$$ is odd

For example:

* If $$p = 3$$, then:
  * $$3^1 \equiv 3 \pmod 4$$ ✅ (odd $$m = 1$$)
  * $$3^2 = 9 \equiv 1 \pmod 4$$ ❌ (even $$m = 2$$)

So this algorithm is only valid when both conditions are met.

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze
