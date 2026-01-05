---
description: >-
  Presentation: https://youtu.be/9wpJk-wUa-w,
  https://www.youtube.com/watch?v=54I7Rd9_9_s
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

## 5. Relative Frobenius Mapping

In a tower of extensions $$\mathbb{F}_{P^n} / \mathbb{F}_{P^m} / \mathbb{F}_P$$ (where $$n = m \cdot k$$), the **Relative Frobenius Mapping** $$\Phi_{rel}$$ on an element $$a \in \mathbb{F}_{P^n}$$ with respect to the base field $$\mathbb{F}_{P^m}$$ is defined as:

$$
\Phi_{rel}(a) = a^{P^m}
$$

Unlike the absolute Frobenius map $$\Phi(a) = a^P$$, this mapping specifically fixes the elements of the intermediate field $$\mathbb{F}_{P^m}$$. That is, for any $$x \in \mathbb{F}_{P^m}$$, $$\Phi_{rel}(x) = x$$.

## 6. Relative Norm

The **Relative Norm** of an element $$a \in \mathbb{F}_{P^n}$$ over $$\mathbb{F}_{P^m}$$, denoted as $$N_{\mathbb{F}_{P^n}/\mathbb{F}_{P^m}}(a)$$, is defined as the product of $$a$$ and its images under the iterated relative Frobenius mappings:

$$
N_{\mathbb{F}_{P^n}/\mathbb{F}_{P^m}}(a) = \prod_{i=0}^{k-1} \Phi_{rel}^i(a) = a \times a^{P^m} \times a^{P^{2m}} \times \cdots \times a^{P^{(k-1)m}}
$$

### 6.1. Properties and Transitivity (Tower Property)

The norm in a tower structure exhibits the property of **Transitivity**, which allows for step-by-step reduction across multiple layers of extension:

* **Mapping to Base Field:** The result of the relative norm is always an element of the immediate base field: $$N_{\mathbb{F}_{P^n}/\mathbb{F}_{P^m}}(a) \in \mathbb{F}_{P^m}$$.
*   **Absolute Norm**: The norm of an element relative to the prime field $$\mathbb{F}_P$$ (the absolute norm) can be computed by taking the norm of the norm:

    $$
    N_{\mathbb{F}_{P^n}/\mathbb{F}_P}(a) = N_{\mathbb{F}_{P^m}/\mathbb{F}_P}(N_{\mathbb{F}_{P^n}/\mathbb{F}_{P^m}}(a))
    $$

### 6.2. Mathematical Derivation of Transitivity

To compute the absolute norm $$N_{\mathbb{F}_{P^n}/\mathbb{F}_P}(a)$$ using the tower property, we follow a stepwise reduction process through the intermediate field $$\mathbb{F}_{P^m}$$.

#### Step 1: Relative Norm from $$\mathbb{F}_{P^n}$$ to $$\mathbb{F}_{P^m}$$

First, we reduce the element $$a \in \mathbb{F}_{P^n}$$ to an element $$b \in \mathbb{F}_{P^m}$$ by applying the relative norm:

$$
b = N_{\mathbb{F}_{P^n}/\mathbb{F}_{P^m}}(a) = \prod_{i=0}^{k-1} a^{(P^m)^i} = a \times a^{P^m} \times a^{P^{2m}} \times \cdots \times a^{P^{(k-1)m}}
$$

#### Step 2: Absolute Norm from $$\mathbb{F}_{P^m}$$ to $$\mathbb{F}_P$$

Next, we treat $$b$$ as an element of the base field and apply its own norm mapping toward the prime field:

$$
N_{\mathbb{F}_{P^m}/\mathbb{F}_P}(b) = \prod_{j=0}^{m-1} b^{P^j} = b \times b^P \times b^{P^2} \times \cdots \times b^{P^{m-1}}
$$

#### Step 3: Final Consolidation

By substituting the definition of $$b$$ into the second equation, we obtain the double product:

$$
N_{\mathbb{F}_{P^n}/\mathbb{F}_P}(a) = \prod_{j=0}^{m-1} \left( \prod_{i=0}^{k-1} a^{P^{mi}} \right)^{P^j} = \prod_{j=0}^{m-1} \prod_{i=0}^{k-1} a^{P^{mi + j}}
$$

This double product covers all exponents $$P^0, P^1, \dots, P^{n-1}$$ exactly once, proving that:

$$
N_{\mathbb{F}_{P^n}/\mathbb{F}_P}(a) = a \times a^P \times a^{P^2} \times \cdots \times a^{P^{n-1}}
$$

### 6.3. Practical Example: $$\mathbb{F}_{P^4} / \mathbb{F}_{P^2} / \mathbb{F}_P$$

For an element $$a \in \mathbb{F}_{P^4}$$ where $$k=2$$ and $$m=2$$:

1. **Relative Norm:** $$b = N_{\mathbb{F}_{P^4}/\mathbb{F}_{P^2}}(a) = a \cdot a^{P^2}$$.
2. **Base Norm:** $$N_{\mathbb{F}_{P^2}/\mathbb{F}_P}(b) = b \cdot b^P$$.
3. **Result:** $$(a \cdot a^{P^2}) \cdot (a \cdot a^{P^2})^P = a \cdot a^P \cdot a^{P^2} \cdot a^{P^3}$$, which is the absolute norm.

## 7. Recursive Inversion in Tower Extensions

The tower property of the norm provides an efficient recursive algorithm for computing the inverse $$x^{-1}$$ in high-degree extension fields.

1. **Compute Relative Norm:** Calculate $$N(x) = x^{r} \in \mathbb{F}_{P^m}$$ using the pre-computed relative Frobenius coefficients, where $$r = \frac{(P^m)^k - 1}{P^m - 1}$$.
2. **Recursive Call:** Compute the relative inverse of this norm, $$N(x)^{-1}$$, within the base field $$\mathbb{F}_{P^m}$$. If $$\mathbb{F}_{P^m}$$ is itself an extension field, repeat the norm-based inversion process.
3.  **Final Reconstruction**: Multiply the partial product of Frobenius images by the inverse obtained from the base field:

    $$
    x^{-1} = \left(\prod_{i=1}^{k-1}\Phi_{rel}^i(x)\right) \cdot N_{\mathbb{F}_{P^n}/\mathbb{F}_{P^m}}(x)^{-1}
    $$

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze
