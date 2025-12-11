---
description: 'Presentation: https://youtu.be/9wpJk-wUa-w'
---

# Inversion

## 1. Frobenius Mapping

Given a prime $$P$$, the **Frobenius Mapping** $$\Phi$$ on an element $$a \in \mathbb{F}_{P^k}$$ is defined as: $$\Phi(a) = a^P$$

This mapping is a crucial [**endomorphism**](../group/morphisms.md#endomorphism-a-homomorphism-where-domain-and-codomain-are-the-same-object) of the field $$\mathbb{F}_{P^k}$$ that fixes the elements of the prime subfield $$\mathbb{F}_P$$.

## 2. Efficient Computation of $$\Phi(a)$$

Let the field element $$a$$ be represented as a polynomial over the prime field: $$a = \sum_{i=0}^{k-1} a_i x^i$$, where $$a_i \in \mathbb{F}_P$$.

### 2.1. Basic Computation using Characteristic $$P$$

Since the characteristic of the field is $$P$$, we can apply the [**Freshman's Dream**](https://en.wikipedia.org/wiki/Freshman's_dream) property:&#x20;

$$
\Phi(a) = a^P = \left(\sum_{i=0}^{k-1} a_i x^i\right)^P = \sum_{i=0}^{k-1} (a_i)^P (x^i)^P
$$

Since $$a_i \in \mathbb{F}_P$$, by [**Fermat's Little Theorem**](https://en.wikipedia.org/wiki/Fermat's_little_theorem), we have $$a_i^P = a_i$$. Thus, the mapping simplifies to:&#x20;

$$
\Phi(a) = \sum_{i=0}^{k-1} a_i x^{iP}
$$

### 2.2. Optimized Computation when $$P \equiv 1 \pmod k$$

If the extension is defined by an irreducible polynomial such that $$x^k = \xi$$ (where $$\xi \in \mathbb{F}_P$$), and $$P \equiv 1 \pmod k$$ (meaning $$P-1$$ is divisible by $$k$$), we can further optimize the expression.

Let $$q = \frac{P-1}{k}$$. We can rewrite $$x^{iP}$$ using the field relation $$x^k = \xi$$:&#x20;

$$
x^{iP} = x^{i(qk+1)} = x^{iqk} \cdot x^i = (x^k)^{iq} \cdot x^i = \xi^{iq} \cdot x^i
$$

Substituting this back into the formula for $$\Phi(a)$$:&#x20;

$$
\begin{align*}
\Phi(a) &= \sum_{i=0}^{k-1} a_i \left(\xi^q\right)^i x^i \\ 
&= \sum_{i=0}^{k-1} a_i \left(\xi^{\frac{P-1}{k}}\right)^i x^i \\
&= a_0 + a_1 \left(\xi^{\frac{P-1}{k}}\right) x + a_2 \left(\xi^{\frac{P-1}{k}}\right)^2 x^2 + \cdots + a_{k-1} \left(\xi^{\frac{P-1}{k}}\right)^{k-1} x^{k-1}
\end{align*}
$$

This demonstrates that if the set of scalar factors $$\left(\xi^{\frac{P-1}{k}}, \dots, \xi^{(k-1){\frac{P-1}{k}}}\right)$$ is **pre-computed**, the Frobenius mapping can be calculated efficiently using only scalar multiplications and polynomial addition.

{% hint style="info" %}
**Pre-computation Note:** Since $$\Phi^k(a) = a$$, we only need to pre-compute the set of iterated mappings $$\left(\Phi(a), \Phi^2(a), \dots, \Phi^{k-1}(a)\right)$$ for use in the Norm and Inversion calculations.
{% endhint %}

## 3. Norm

The **Norm** of an element $$a \in \mathbb{F}_{P^k}$$, denoted $$N(a)$$, is defined as the product of $$a$$ and all its images under the iterated Frobenius mappings:

$$
N(a) = \prod_{i=0}^{k-1} \Phi^i(a) = a \times \Phi(a) \times \Phi^2(a) \times \cdots \times \Phi^{k-1}(a)
$$

### 3.1. Key Property of the Norm

The most important property of the norm is that its result is an element of the **prime subfield** $$\mathbb{F}_P$$. $$\mathbf{N(a) \in \mathbb{F}_P}$$

**Proof:** We apply the Frobenius mapping to $$N(a)$$:

$$
\begin{aligned}
\Phi(N(a)) &= \Phi\left(\prod_{i=0}^{k-1} \Phi^i(a)\right) \\
&= \prod_{i=0}^{k-1} \Phi(\Phi^i(a)) \\
&= \prod_{i=0}^{k-1} \Phi^{i+1}(a) \\
&= \Phi^1(a) \times \Phi^2(a) \times \cdots \times \Phi^{k-1}(a) \times \Phi^k(a)
\end{aligned}
$$

Since $$\Phi^k(a) = a$$, we can replace the last term:

$$
\Phi(N(a)) = \Phi^1(a) \times \Phi^2(a) \times \cdots \times \Phi^{k-1}(a) \times a = N(a)
$$

The condition $$\Phi(N(a)) = N(a)$$ means $$N(a)^P = N(a)$$, which, by Fermat's Little Theorem, implies that $$N(a)$$ must belong to the prime field $$\mathbb{F}_P$$.

## 4. Inversion on the Extension Field $$\mathbb{F}_{P^k}$$

The norm $$N(x)$$ is instrumental in efficiently computing the inverse $$x^{-1}$$ for any non-zero element $$x \in \mathbb{F}_{P^k}$$.

### 4.1. Defining the Factor $$r$$

Let $$r$$ be defined as the sum of powers of $$P$$:

$$
r = \frac{P^k - 1}{P-1} = \sum_{i=0}^{k-1} P^i = 1 + P + P^2 + \cdots + P^{k-1}
$$

We can rewrite $$x^{-1}$$ using $$r$$:

$$
x^{-1} = x^{r-1} \cdot x^{-r}
$$

### 4.2. Calculating $$x^{r-1}$$

The exponent $$r-1$$ can be expressed as a sum of powers of $$P$$:

$$
r-1 = P + P^2 + \cdots + P^{k-1}
$$

This allows $$x^{r-1}$$ to be calculated as a product of iterated Frobenius images:&#x20;

$$
x^{r-1} = x^P \times x^{P^2} \times \cdots \times x^{P^{k-1}} = \Phi(x) \times \Phi^2(x) \times \cdots \times \Phi^{k-1}(x) = \prod_{i=1}^{k-1}\Phi^i(x)
$$

### 4.3. Calculating $$x^{-r}$$

Note that $$x^r = x \cdot x^{r-1}$$. Comparing this to the Norm definition, we see that:&#x20;

$$
x^r = x \cdot \prod_{i=1}^{k-1}\Phi^i(x) = N(x)
$$

Since $$x^r = N(x)$$ **is an element of** $$\mathbb{F}_P$$ (a scalar), its inverse $$x^{-r} = N(x)^{-1}$$ can be computed quickly within the prime field $$\mathbb{F}_P$$.

In practice, $$N(x)$$ is simply the **constant term** of the polynomial resulting from the product $$x^{r-1} \cdot x$$:&#x20;

$$
x^{-r} = \left((x^{r-1} \cdot x)_0\right)^{-1}
$$

where $$(A)_0$$ denotes the constant coefficient of polynomial $$A$$.

### 4.4. Final Inversion Formula

The inverse $$x^{-1}$$ is calculated as the product of the pre-computed Frobenius images $$x^{r-1}$$ and the inverse of the constant term $$N(x)$$:

$$
x^{-1} = x^{r-1} \cdot \left((x^{r-1} \cdot x)_0\right)^{-1}
$$

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze
