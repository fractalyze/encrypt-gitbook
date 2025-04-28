---
description: 'Presentation: https://youtu.be/eEU1v1ZPHe0'
---

# Modular Inverse

## Modular Inverse

Suppose we want to find the **modular inverse** $$x = a^{-1}$$ of some element $$a$$ modulo $$p$$, that is:

$$
ax \equiv 1 \mod p
$$

The inverse **exists if and only if** $$\gcd(a, p) = 1$$, i.e., when $$a$$ is **coprime** with $$p$$.

When $$p$$ is a **prime number**, the field $$\mathbb{F}_p$$ forms a finite field, and every nonzero element has an inverse. In that case, [**Fermat’s Little Theorem**](https://en.wikipedia.org/wiki/Fermat's_little_theorem) can be used to compute the inverse efficiently:

$$
x \equiv a^{p-2} \mod p
$$

{% hint style="warning" %}
Note: This method **only works when** $$p$$ **is prime**. For general modulus $$m$$ (not necessarily prime), the modular inverse of $$a$$ (if it exists) can be computed using the **Extended Euclidean Algorithm**.
{% endhint %}

## Modular Inverse using Binary Extended Euclidean Algorithm

However, a more efficient method is to use the [**Extended Euclidean Algorithm**](../../euclidean-algorithm/#extended-eculidean-algorithm), which solves the following equation:

$$
ax + py = \gcd(a, p) = 1
$$

Taking both sides modulo $$p$$ yields:

$$
ax \equiv 1 \mod p
$$

Therefore, $$x$$ is the **modular inverse** of $$a$$ modulo $$p$$.

### Core Idea

Let $$p$$ be an odd prime, and let $$a$$ and $$p$$ be coprime. Starting with $$u = a$$ and $$v = p$$, we apply the Binary GCD rules and reduce until either $$u$$ or $$v$$ becomes 1. The case where both $$u$$ and $$v$$ are even has already been handled in the previous stage and is thus excluded here.

$$
\gcd(u, v) = 
\begin{cases}
1 & \text{ if } u = 1 \lor v = 1 \\
\gcd \left(\frac{u}{2}, v \right) & \text{ if } u \equiv 0 \pmod 2 \\
\gcd \left(u, \frac{v}{2} \right) & \text{ if } v \equiv 0 \pmod 2 \\
\gcd \left(u - v, v \right) & \text{ if } u \ge v \land u \equiv 1 \pmod 2 \land \ v \equiv 1 \pmod 2 \\
\gcd \left(u, v - u \right) & \text{ if } v > u \land u \equiv 1 \pmod 2 \land \ v \equiv 1 \pmod 2 \\
\end{cases}
$$

Meanwhile, as in the Binary Extended Euclidean Algorithm, we initialize the variables as $$x_1 = 1, y_1 = 0, x_2 = 0, y_2 = 1$$, and maintain the following two invariants:

$$
ax_1 + p y_1 = u, \quad a x_2 + p y_2 = v
$$

If we take both sides modulo $$p$$, we obtain simpler invariants:

$$
a x_1 \equiv u \mod p, \quad a x_2 \equiv v \mod p
$$

At each step, we update the expressions while maintaining these invariants. Let’s examine just case 2 and case 4 as examples.

#### Case 2: $$u$$ is even

* If $$x_1$$​ is even, we update as:

$$
a x_1 \equiv u \mod p \quad \Leftrightarrow \quad a \cdot \frac{x_1}{2} \equiv \frac{u}{2} \mod p
$$

* If $$x_1$$​ is odd, we update as:

$$
a x_1 \equiv u \mod p \quad \Leftrightarrow \quad a \cdot \frac{x_1 + p}{2} \equiv \frac{u}{2} \mod p
$$

#### Case 4: $$u \ge v$$ and both $$u$$ and $$v$$ are odd

We update as:

$$
a x_1 \equiv u \mod p, \quad a x_2 \equiv v \mod p \quad \Leftrightarrow \quad a(x_1 - x_2) \equiv u - v \mod p
$$

### Code

Here is the Python implementation of **Algorithm 16: BEA for Inversion in 𝔽ₚ** (Binary Extended Euclidean Algorithm for computing the modular inverse in a prime field) from [here](https://www.sandeep.de/my/papers/2006_ActaApplMath_EfficientSoftFiniteF.pdf#page=22). Each step is annotated with comments to aid understanding.

```python
def binary_inverse_fp(a: int, p: int) -> int:
    """
    Computes the modular inverse of a modulo prime p using Binary Extended Euclidean Algorithm.
    
    Parameters: a ∈ 𝔽ₚ, prime p greater than 2
    
    Returns: a⁻¹ mod p
    """

    u, v = a, p      # Step 1: Initialize u ← a, v ← p
    b, c = 1, 0      # Initialize Bézout coefficients: b for u, c for v

    while u != 1 and v != 1:  # Step 2
        # Step 3-10: Make u odd
        while u % 2 == 0:
            u //= 2
            if b % 2 == 0:
                b //= 2
            else:
                b = (b + p) // 2

        # Step 11-18: Make v odd
        while v % 2 == 0:
            v //= 2
            if c % 2 == 0:
                c //= 2
            else:
                c = (c + p) // 2

        # Step 19-23: Subtract the smaller from the larger and update coefficients
        if u >= v:
            u -= v
            b -= c
        else:
            v -= u
            c -= b

    # Step 25-29: Return the correct inverse mod p
    if u == 1:
        return b % p
    else:
        return c % p
```

### For Montgomery Domain

A value $$a$$ is transformed to its [**Montgomery representation**](../modular-reduction/montgomery-reduction.md#the-montgomery-representation) as follows:

$$
\mathsf{toMont}(a) = aR = a2^n
$$

Therefore, if we naively compute the modular inverse of $$aR$$, we get:

$$
\mathsf{ModInv}(aR) = a^{-1}R^{-1} = a^{-1}2^{-n}
$$

However, the form we actually want is:

$$
\mathsf{toMont}(a^{-1}) = a^{-1}R
$$

To achieve this, instead of initializing with $$b = 1$$, we can set $$b = R^2 \mod p$$, and then return the result of the inversion algorithm.

$$
\mathsf{ModInv}(aR) \times R^2 = a^{-1}R
$$

## Montgomery Inversion Algorithm

Another efficient algorithm for computing modular Inverse is **Kaliski's Montgomery Inverse algorithm**.

Kaliski’s Montgomery Inverse consists of two phases:

* **Almost Inverse phase**

$$
\mathsf{AlmostInv}(a) = a^{-1} \cdot 2^k
$$

where $$n \le k \le 2n$$

* **Correction phase**

$$
\mathsf{Correction}(a^{-1} \cdot 2^k) = a^{-1} \cdot 2^n
$$

### Core Idea

TODO(chokoble): Add reason why this code works.

### Code

Here is the Python implementation of **Algorithm 17: Montgomery Inversion in 𝔽ₚ** from [here](https://www.sandeep.de/my/papers/2006_ActaApplMath_EfficientSoftFiniteF.pdf#page=23).&#x20;

```python
def montgomery_inverse_fp(a: int, p: int, n: int) -> tuple[int, int]:
    """
    Computes Montgomery Inverse of a mod p.
    Returns r ≡ a⁻¹ · 2^k mod p, where n ≤ k ≤ 2n.
    
    Parameters:
        a (int): element in 𝔽ₚ
        p (int): prime modulus
        n (int): expected bit-length of p (e.g., n = p.bit_length())
    
    Returns:
        (r, k): Montgomery inverse r and the exponent k
    """
    
    # Step 1: Initialization
    u = p
    v = a
    r = 0
    s = 1
    k = 0

    # Step 2–11: Main loop
    while v > 0:
        if u % 2 == 0:
            u //= 2
            s *= 2
        elif u > v:
            u = (u - v) // 2
            r += s
            s *= 2
        else:  # v >= u
            v = (v - u) // 2
            s += r
            r *= 2
        k += 1

    # Step 12–14: Reduce r if needed (Almost Inverse)
    if r >= p:
        r -= p

    # Step 15–21: Final division loop to normalize r
    for _ in range(k - n):
        if r % 2 == 0:
            r //= 2
        else:
            r = (r + p) // 2

    # Step 22: Return Montgomery inverse and k
    return r, k
```

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
