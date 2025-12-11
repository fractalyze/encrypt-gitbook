---
description: 'Presentation: https://youtu.be/jtImWo3CmZM'
---

# Karatsuba Multiplication

## 1. Field Context: $$\mathbb{F}_{P^k}$$ Multiplication

In the extension field $$\mathbb{F}_{P^k}$$, an element $$A$$ is represented as a polynomial of degree up to $$k-1$$ with coefficients in the base field $$\mathbb{F}_P$$:

$$
A(x) = a_{k-1}x^{k-1} + \dots + a_1 x + a_0
$$

Multiplying two elements $$A$$ and $$B$$ in $$\mathbb{F}_{P^k}$$ involves two steps:

1. **Polynomial Multiplication:** Compute $$C'(x) = A(x) \cdot B(x)$$.
2. **Reduction:** Compute $$C(x) = C'(x) \pmod{f(x)}$$, where $$f(x)$$ is the irreducible polynomial of degree $$k$$ defining the extension.

The **Karatsuba algorithm** is used to speed up the **Polynomial Multiplication** step (Step 1).

## 2. Karatsuba Algorithm for Polynomials

We aim to compute $$C'(x) = A(x) \cdot B(x)$$, where $$A(x)$$ and $$B(x)$$ are polynomials of degree $$N=k-1$$. We assume $$k$$ is an even number for simplicity, and let $$m = k/2$$.

### Step 1: Splitting (Divide)

The polynomials $$A(x)$$ and $$B(x)$$ are split into two halves, where $$A_1(x), A_0(x), B_1(x), B_0(x)$$ are polynomials of degree less than $$m$$

$$
A(x) = A_1(x) \cdot x^m + A_0(x), \quad B(x) = B_1(x) \cdot x^m + B_0(x)
$$

The product $$C'(x) = A(x)B(x)$$ expands algebraically as:

$$
C'(x) = A_1(x) B_1(x) \cdot x^{2m} + A_1(x) B_0(x) + A_0(x) B_1(x)) \cdot x^m + A_0(x) B_0(x)
$$

The classical method would require **four** recursive polynomial multiplications of size $$m$$.

### Step 2: Three Recursive Multiplications

Karatsuba reduces this to **three** recursive polynomial multiplications, where the coefficients of the polynomials are in $$\mathbb{F}_P$$ (or another base field).

1.  $$M_1(x)$$ **(High Part):**

    $$
    M_1(x) = A_1(x) \cdot B_1(x)
    $$
2.  $$M_2(x)$$ **(Low Part):**

    $$
    M_2(x) = A_0(x) \cdot B_0(x)
    $$
3.  $$M_3(x)$$ **(Middle Part – The Karatsuba Trick):**

    $$
    M_3(x) = (A_1(x) + A_0(x)) \cdot (B_1(x) + B_0(x))
    $$

### Step 3: Combination (Conquer)

The middle term is extracted by performing polynomial additions and subtractions over $$\mathbb{F}_P$$:

$$
A_1(x) B_0(x) + A_0(x) B_1(x) = M_3(x) - M_1(x) - M_2(x)
$$

The final unreduced product $$C'(x)$$ is then constructed:

$$
C'(x) = M_1(x) \cdot x^{2m} + (M_3(x) - M_1(x) - M_2(x)) \cdot x^m + M_2(x)
$$

(Multiplication by $$x^m$$ or $$x^{2m}$$ is a simple coefficient shift.)

### Step 4: Final Reduction

After obtaining the unreduced product $$C'(x)$$, the result must be reduced modulo the defining polynomial $$f(x)$$ to get the final result $$C(x) \in \mathbb{F}_{P^k}$$:

$$
C(x) = C'(x) \pmod{f(x)}
$$

## 3. Complexity in $$\mathbb{F}_{P^k}$$ Context

For a base field $$\mathbb{F}_P$$, the Karatsuba algorithm reduces the complexity of polynomial multiplication from $$O(k^2)$$ base field operations (classical) to $$O(k^{\log_2 3})$$ base field operations.

* **Recursive Call Size:** The size of the polynomials in the recursive calls is reduced from degree $$k-1$$ to approximately degree $$k/2 - 1$$.
* **Base Operations:** The "cost" of addition/subtraction ($$O(k)$$) and the recursive multiplications ($$3 \cdot T(k/2)$$) is measured in terms of operations within the base field $$\mathbb{F}_P$$.

The overall time complexity for the polynomial multiplication step remains:

$$
T(k) = O(k^{\log_2 3}) \approx O(k^{1.585})
$$

Karatsuba is highly effective in cryptographic contexts (like Elliptic Curve Cryptography) that extensively rely on fast finite field arithmetic in $$\mathbb{F}_{P^k}$$ or $$\mathbb{F}_{2^k}$$.

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze
