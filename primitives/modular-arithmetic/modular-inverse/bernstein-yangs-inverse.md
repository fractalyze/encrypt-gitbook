---
description: 'Presentation: https://youtu.be/t0_iII-nKNw'
---

# Bernstein-Yang's Inverse

## Introduction

GCD-based algorithms are widely used as an efficient method for computing modular inverses. However, most of them involve **input-dependent branching**, which makes them vulnerable to **side-channel attacks**.

For example, an implementation of [**Stein’s binary GCD algorithm**](../../euclidean-algorithm/binary-euclidean-algorithm.md) in OpenSSL was shown to be susceptible to an **RSA key extraction attack**. This kind of attack demonstrates that the execution path of conditional branches can leak secret information.

A simple countermeasure is to use [**Fermat’s Little Theorem**](https://en.wikipedia.org/wiki/Fermat's_little_theorem), which computes the inverse as $$a^{-1} \equiv a^{p-2} \mod p$$. This method can be implemented in **constant time**, but it is generally **less efficient**.

This raises the question: Is there a way to compute the modular inverse that is both **constant-time secure** and **faster than the Fermat-based approach**?

## Background

### Limitations of Existing Constant-Time GCD Algorithms

The Binary GCD algorithm (also known as Stein’s Algorithm) involves **branching based on the parity** of the input integers or **comparisons between them**, which causes the execution path to vary. For example:

* If $$a$$ is even → right shift
* If $$b$$ is even → right shift
* If $$a > b$$ → $$a := a - b$$

Branching based on whether $$a$$ or $$b$$ is even is relatively safe since it only inspects the least significant bit. However, branching on comparisons like $$a > b$$ often requires **examining multiple bits (or limbs)**, which makes it difficult to ensure **constant-time execution**.

## Protocol Explanation

The original paper also covers the case of computing inverses of polynomials, but for simplicity, we will restrict ourselves to the case of integers. The explanation closely follows the implementation described in [https://github.com/bitcoin-core/secp256k1/blob/master/doc/safegcd\_implementation.md](https://github.com/bitcoin-core/secp256k1/blob/master/doc/safegcd_implementation.md), while adhering to the original paper's formulation of the `divstep` update as much as possible.

### GCD with `divstep`

When $$a$$ is an **odd integer** and $$b$$ is **any integer**, the `divstep` function is defined as follows:

$$
\mathsf{divstep}(\delta, a, b) = 
\begin{cases}
(1 - \delta,\ b,\ \frac{(b \bmod 2)\cdot a - (a \bmod 2)\cdot b}{2}) & \text{if } \delta > 0 \land b \bmod 2 = 1 \\
(1 + \delta,\ a,\ \frac{(a \bmod 2)\cdot b - (b \bmod 2)\cdot a}{2}) & \text{otherwise}
\end{cases}
$$

This operation is designed to execute in **constant-time**, relying only on fixed-bit values like $$\delta$$ and the least significant bit of $$b$$ to determine behavior, ensuring that only a single bit needs to be checked.

#### Stages

The `divstep` operation can be viewed as a **composition of two functions**:

1.  **Conditional Swap**

    * Condition: $$\delta > 0$$ and $$b$$ is odd
    * Operation: $$(\delta, a, b) \rightarrow (-\delta, b, a)$$

    TODO(chokobole): explain the reason why $$\delta$$ helps to reduce $$b$$ to 0.
2.  **Elimination**

    * Always performed
    * Purpose: to reduce the size of $$b$$
    * Operation: $$(\delta, a, b) \rightarrow (1 + \delta, a, \frac{(a \bmod 2)\cdot b - (b \bmod 2)\cdot a}{2})$$

    Since $$a$$ is always odd, $$a \bmod 2 = 1$$ is guaranteed, allowing simplification:

    $$
    (\delta, a, b) \rightarrow (1 + \delta, a, \frac{b - (b \bmod 2)\cdot a}{2})
    $$

    This division by 2 is always valid because:

    * If $$b$$ is odd, then $$b - a$$ is even (odd - odd = even)
    * If $$b$$ is even, the result is trivially divisible by 2

#### GCD Preservation

This logic is rooted in the classical Euclidean algorithm:

$$
\gcd(a, b) = \gcd(b, a) = \gcd(a, b - a) = \gcd(a, b/2) \quad \text{(if } b \text{ is even)}
$$

Thus, `divstep` preserves the GCD while continually reducing the size of $$b$$. Eventually, $$b$$ reaches 0, and the GCD remains in $$a$$.

#### Simplified `divstep`

The behavior of `divstep` can be more concretely classified as:

$$
\mathsf{divstep}(\delta, a, b) =
\begin{cases}
a & \text{if } b = 0 \\
(1 - \delta, b, \frac{a - b}{2}) & \text{if } \delta > 0 \land b \bmod 2 = 1 \\
(1 + \delta, a, \frac{b - a}{2}) & \text{if } \delta \le 0 \land b \bmod 2 = 1 \\
(1 + \delta, a, \frac{b}{2}) & \text{if } b \bmod 2 = 0
\end{cases}
$$

As previously discussed, this operation ultimately returns the GCD.

#### Code

```python
def gcd(a, b):
    """Compute the GCD of an odd integer a and another integer b."""
    assert a & 1  # require a to be odd
    delta = 1     # additional state variable
    while b != 0:
        assert a & 1  # a will be odd in every iteration
        if delta > 0 and b & 1:
            delta, a, b = 1 - delta, b, (a - b) // 2
        elif b & 1:
            delta, a, b = 1 + delta, a, (b - a) // 2
        else:
            delta, a, b = 1 + delta, a, b // 2
    return abs(a)
```

### Modular Inverse with `divstep`

The idea of using `divstep` to compute the **modular inverse** becomes most clear when we assume that the modulus `p` is an **odd prime**. In this section, we explain the procedure for computing the inverse of an integer $$a \in \mathbb{F}_p$$, i.e., $$a^{-1} \mod p$$.

#### Initial Setup

We initialize the variables as $$u = p$$, $$v = a$$, $$x_1 = 0$$, and $$x_2 = 1$$, and maintain the following **invariants** throughout:

$$
ax_1 \equiv u \mod p, \quad ax_2 \equiv v \mod p
$$

At each iteration, we apply a `divstep` transformation to the pair $$(u, v)$$. Since $$p$$ is a prime and $$a \in \mathbb{F}_p^\times$$, we know that $$\gcd(p, a) = 1$$. This guarantees that repeated applications of `divstep` will eventually reduce $$v$$ to $$0$$, and $$u$$ to either $$1$$ or $$-1$$.

Formally, after a sufficient number of iterations, we reach:

$$
(u, v) = (\pm 1, 0)
$$

At this point, because the algorithm maintains the invariant $$a x_1 \equiv u \mod p$$, we conclude:

$$
x_1 \cdot u \equiv a^{-1} \mod p
$$

Since $$u = \pm1$$, multiplying $$x_1$$ by $$u$$ gives the correct modular inverse of $$a$$.

The `divstep` function $$\mathsf{divstep}(\delta, u, v)$$ branches into three cases depending on $$\delta$$ and the parity of $$v$$. We analyze how the invariants are preserved in each case.

#### Case 1: $$\delta > 0$$ and $$v$$ is odd

We use the rule $$\mathsf{divstep}(\delta, a, b) = (1 - \delta, b, \frac{a - b}{2})$$ for **Case 1**. Accordingly, the values of $$u$$ and $$v$$ are updated as:

$$
u \leftarrow v \\
v \leftarrow \frac{u - v}{2}
$$

Since $$u$$ is guaranteed to be odd, the subtraction $$u - v$$ is always even, making the division by 2 valid.

We apply the same update rule to $$x_1$$ and $$x_2$$ as follows:

$$
x_1 \leftarrow x_2 \\
x_2 \leftarrow 
\begin{cases}
\frac{x_1 - x_2}{2} & \text{if } (x_1 - x_2) \bmod 2 = 0 \\
\frac{x_1 - x_2 + p}{2} & \text{otherwise}
\end{cases}
$$

#### Case 2: $$\delta \le 0$$ and $$v$$ is odd

We use the rule $$\mathsf{divstep}(\delta, a, b) = (1 + \delta, a, \frac{b - a}{2})$$ for **Case 2** Accordingly, the values of $$u$$ and $$v$$ are updated as:

$$
u \leftarrow u\\
v \leftarrow \frac{v - u}{2}
$$

Since $$u$$ is guaranteed to be odd, the subtraction $$u - v$$ is always even, making the division by 2 valid.

We apply the same update rule to $$x_1$$ and $$x_2$$ as follows:

$$
x_1 \leftarrow x_1 \\
x_2 \leftarrow 
\begin{cases}
\frac{x_2 - x_1}{2} & \text{if } (x_2 - x_1) \bmod 2 = 0 \\
\frac{x_2 - x_1 + p}{2} & \text{otherwise}
\end{cases}
$$

#### Case 3: $$v$$ is even

We use the rule $$\mathsf{divstep}(\delta, a, b) = (1 + \delta, a, \frac{b}{2})$$ for **Case 3** Accordingly, the values of $$u$$ and $$v$$ are updated as:

$$
u \leftarrow u\\
v \leftarrow \frac{v}{2}
$$

We apply the same update rule to $$x_1$$ and $$x_2$$ as follows:

$$
x_1 \leftarrow x_1 \\
x_2 \leftarrow 
\begin{cases}
\frac{x_2}{2} & \text{if } x_2 \bmod 2 = 0 \\
\frac{x_2 + p}{2} & \text{otherwise}
\end{cases}
$$

#### Code

```python
def div2(p, a):
    """Helper routine to compute a/2 mod p (where p is odd)."""
    assert p & 1
    if a & 1:  # If a is odd, make it even by adding p.
        a += p
    # a must be even now, so clean division by 2 is possible.
    return a // 2

def modinv(p, a):
    """Compute the inverse of a mod p (given that it exists, and p is odd)."""
    assert p & 1
    delta, u, v = 1, p, a
    x1, x2 = 0, 1
    while v != 0:
        # Note that while division by two for u and v is only ever done on even inputs,
        # this is not true for x1 and x2, so we need the div2 helper function.
        if delta > 0 and v & 1:
            delta, u, v = 1 - delta, v, (u - v) // 2
            x1, x2 = x2, div2(p, x1 - x2)
        elif v & 1:
            delta, u, v = 1 + delta, u, (v - u) // 2
            x1, x2 = x1, div2(p, x2 - x1)
        else:
            delta, u, v = 1 + delta, u, v // 2
            x1, x2 = x1, div2(p, x2)
        # Verify that the invariants x1=u/a mod p, x2=v/a mod p are maintained.
        assert u % p == (a * x1) % p
        assert v % p == (a * x2) % p
    assert u == 1 or u == -1  # |u| is the GCD, it must be 1
    # Because of invariant x1 = u/a mod p, 1/a = x1/u mod p. Since |u| = 1, x1/u = x1 * u.
    return (x1 * u) % p
```

### Batching Multiple Divsteps

#### Motivation

The `divstep` operation involves division by 2 for the variables $$u, v, x_1, x_2$$. Instead of performing division by 2 on $$x_1$$​ and $$x_2$$​ at every iteration, we can batch $$N$$ `divstep`s together and apply them to $$x_1$$ and $$x_2$$ all at once. This reduces $$N$$ individual divisions into a single division.

#### Transition Matrix

A single `divstep` can be represented using a matrix multiplication. The one-step update is expressed as:

$$
\begin{bmatrix}x_1^{(1)} \\x_2^{(1)}\end{bmatrix} \equiv \frac{1}{2} \cdot \left( \mathcal{T}(\delta, u, v) \cdot \begin{bmatrix}x_1 \\x_2\end{bmatrix} + \begin{bmatrix} 0 \\ \alpha \end{bmatrix} \right) \mod p
$$

Here, $$\alpha$$ is $$0$$ or $$p$$ depending on $$x_1$$ and $$x_2$$ that make sure the expression inside the parentheses becomes divisible by 2.

{% hint style="info" %}
Note that $$u$$ and $$v$$ are standard integers, while $$x_1$$ and $$x_2$$ are integers modulo $$p$$
{% endhint %}

The transition matrix $$\mathcal{T}$$ used to update $$u, v$$ is the same one applied to $$x_1, x_2$$. The key idea is to compute the transition matrix from $$u$$ and $$v$$ over $$N$$ steps, and then apply it to $$x_1, x_2$$ using matrix-vector multiplication as follows:

$$
\begin{bmatrix}x_1^{(N)} \\x_2^{(N)}\end{bmatrix} \equiv \frac{1}{2^N} \cdot \left( \mathcal{T}^N(\delta, u, v) \cdot \begin{bmatrix}x_1 \\x_2\end{bmatrix} - \begin{bmatrix}m_1p \\ m_2p\end{bmatrix} \right) \mod p
$$

Here, $$m_1p$$ and $$m_2p$$ are correction terms chosen so that the expression inside the parentheses becomes divisible by $$2^N$$. This will be explained in [here](bernstein-yangs-inverse.md#why-div2n-makes-divisible-by).

The transition matrix $$\mathcal{T}(\delta, u, v)$$ is defined as follows:

$$
\mathcal{T}(\delta, u, v) =
\begin{cases}
\begin{pmatrix}0 & 2 \\1 & -1\end{pmatrix} & \text{if } \delta > 0 \land v \bmod 2 = 1 \\
\begin{pmatrix}2 & 0 \\-1 & 1\end{pmatrix} & \text{if } \delta \le 0 \land v \bmod 2 = 1 \\
\begin{pmatrix}2 & 0 \\0 & 1\end{pmatrix} & \text{otherwise}
\end{cases}
$$

#### Upper Bound on Iteration Count: Theorem 11.2

Let $$b = \max(\text{bit\_length}(u), \text{bit\_length}(v))$$, which can usually be considered the bit length of the modulus. Then, the number of N-step matrix applications is bounded by:

$$
N =
\begin{cases}
\left\lfloor \frac{49b + 80}{17} \right\rfloor & \text{if } b < 46 \\
\left\lfloor \frac{49b + 57}{17} \right\rfloor & \text{otherwise}
\end{cases}
$$

**Example:** For the BN254 curve where the prime has 254 bits:

$$
\left\lfloor \frac{49 \cdot 254 + 57}{17} \right\rfloor = 735 \\
\Rightarrow 12 \cdot 62 = 744 > 735 \Rightarrow \boxed{N = 62}
$$

So if we set $$N$$ to 62, it has been proven that a modular inverse can be computed with at most 12 iterations of the loop.

### Code

1. **`transition_matrix`**: Computes the matrix after `N` divsteps, this also computes $$u^{(N)}, v^{(N)}$$

```python
def transition_matrix(delta, u, v):
    """Compute delta and transition matrix t after N divsteps (multiplied by 2^N)."""
    m00, m01, m10, m11 = 1, 0, 0, 1  # start with identity matrix
    for _ in range(N):
        if delta > 0 and v & 1:
            delta, u, v, m00, m01, m10, m11 = 1 - delta, v, (u - v) // 2, 2*m10, 2*m11, m00 - m10, m01 - m11
        elif v & 1:
            delta, u, v, m00, m01, m10, m11 = 1 + delta, u, (v - u) // 2, 2*m00, 2*m01, m10 - m00, m11 - m01
        else:
            delta, u, v, m00, m01, m10, m11 = 1 + delta, u, v // 2, 2*m00, 2*m01, m10, m11
    return delta, u, v, (m00, m01, m10, m11)
```

2. **`div2n` &** `update_x1x2`: Applies the matrix to $$[x_1, x_2]$$ to compute $$x_1^{(N)}, x_2^{(N)}$$

```python
def div2n(p, p_inv, x):
    """Compute x/2^N mod p, given p_inv = 1/p mod 2^N."""
    assert (p * p_inv) % 2**N == 1
    m = (x * p_inv) % 2**N
    x -= m * p
    assert x % 2**N == 0
    return (x >> N) % p

def update_x1x2(x1, x2, t, p, p_inv):
    """Multiply matrix t/2^N with [x1, x2], modulo p."""
    m00, m01, m10, m11 = t
    x1n, x2n = m00*x1 + m01*x2, m10*x1 + m11*x2
    return div2n(p, p_inv, x1n), div2n(p, p_inv, x2n)
```

#### **Why `div2n` makes** $$x$$ **divisible by** $$2^N$$**:**

We can express an integer $$x$$ as:

$$
x = 2^N \cdot q + r
$$

where $$r$$ is the **remainder mod** $$2^N$$. To make $$x$$ divisible by $$2^N$$, we need to **eliminate the** $$r$$ **part**.

To do this, we first compute:

$$
m = (x \cdot p^{-1}) \mod 2^N = r \cdot p^{-1} \mod 2^N \Rightarrow m \cdot p \equiv r \mod  2^N
$$

Then, we subtract $$m \cdot p$$ from $$x$$:

$$
x - m \cdot p = 2^N \cdot q + r - m \cdot p\equiv2^N +r - r \equiv 2^Nq \equiv 0\mod 2^N
$$

which is now **cleanly divisible by** $$2^N$$.

Additionally, since $$m \cdot p \equiv r \mod 2^N$$, the subtraction step preserves the modular equivalence:

$$
x - m \cdot p \equiv x \mod p
$$

3. **`modinv`**: Combines all components to compute the modular inverse

```python
def modinv(p, p_inv, x):
    """Compute the modular inverse of x mod p, given p_inv=1/p mod 2^N."""
    assert p & 1
    delta, u, v, x1, x2 = 1, p, x, 0, 1
    while v != 0:
        delta, u, v, t = transition_matrix(delta, u, v)
        x1, x2 = update_x1x2(x1, x2, t, p, p_inv)
    assert u == 1 or u == -1  # |u| must be 1
    return (x1 * u) % p
```

### Avoiding Modulus Operations

The original `modinv` function already involved a modulus operation, but with the introduction of batched `divstep`, expensive modular operations now occur at every loop iteration. In this section, we aim to eliminate those costly modulus computations.

#### Removing Modulus Operation in `div2n`

**Original Purpose:**\
The `div2n` function is used to reduce results into the range $$[0, p)$$.

**Optimization Insight:**\
Instead of restricting outputs to modulo $$p$$, we can eliminate the `mod` operation if we ensure that all intermediate values remain within a wider range, such as $$(-2p, p)$$.

**✅ Version 1: Allowing Range Expansion**

```python
def update_x1x2_optimized_ver1(x1, x2, t, p, p_inv):
    """Multiply matrix t/2^N with [x1, x2], modulo p, given p_inv = 1/p mod 2^N."""
    m00, m01, m10, m11 = t
    x1n, x2n = m00*x1 + m01*x2, m10*x1 + m11*x2
    # Cancel out bottom N bits of x1n and x2n.
    mx1n = ((x1n * p_inv) % 2**N)
    mx2n = ((x2n * p_inv) % 2**N)
    x1n -= mx1n * p
    x2n -= mx2n * p
    return x1n >> N, x2n >> N  
```

**🔍 Range Analysis of Version 1**

Each transition matrix $$\mathcal{T}$$ acts on the vector $$[x_1, x_2]^T$$ in one of the following ways:

$$
\mathcal{T}(\delta, u, v)\begin{pmatrix} x_1 \\ x_2 \end{pmatrix} =
\begin{cases}
\begin{pmatrix}2x_2 \\ x_1 - x_2\end{pmatrix} & \text{if } \delta > 0 \land v \bmod 2 = 1 \\
\begin{pmatrix}2x_1 \\ x_2 - x_1\end{pmatrix} & \text{if } \delta \le 0 \land v \bmod 2 = 1 \\
\begin{pmatrix}2x_1 \\ x_2\end{pmatrix} & \text{otherwise}
\end{cases}
$$

After $$N$$ iterations, we get:

$$
-2^N x_1 < x_1^{(N)} < 2^N x_1, \quad -2^N x_2 < x_2^{(N)} < 2^N x_2
$$

Since $$x_1, x_2 \in (-p, p)$$ initially:

$$
-2^N p < x_1^{(N)} < 2^N p, \quad -2^N p < x_2^{(N)} < 2^N p
$$

In `update_x1x2_optimized_ver1`, we subtract a multiple of $$p$$ before division:

$$
x_1^{(N)} \leftarrow x_1^{(N)} - m_{x_1} \cdot p, \quad x_2^{(N)} \leftarrow x_2^{(N)} - m_{x_2} \cdot p
$$

With $$0 \le m_{x_1}, m_{x_2} < 2^N$$, we obtain:

$$
-2^{N+1}p < x_1^{(N)} - m_{x_1}p < 2^N p, \quad -2^{N+1}p < x_2^{(N)} - m_{x_2}p < 2^N p
$$

After division by $$2^N$$:

$$
-2p < \frac{x_1^{(N)} - m_{x_1}p}{2^N} < p, \quad -2p < \frac{x_2^{(N)} - m_{x_2}p}{2^N} < p
$$

Hence, the result stays within $$(-2p, p)$$.

**⚠️ Drawback**

After each `divstep`, the range expands:

$$
(-p, p) \rightarrow (-2p, p) \rightarrow (-3p, 2p) \rightarrow ...
$$

This increasing range leads to:

* Unnecessary arithmetic overhead
* Larger intermediate values
* Potential memory inefficiencies

**✅ Version 2: Pre-Clamping Strategy**

To prevent range blow-up, we can add $$p$$ to negative values to bring the range back to $$(-p, p)$$:

```python
def update_x1x2_optimized_ver2(x1, x2, t, p, p_inv):
    """Multiply matrix t/2^N with [x1, x2], modulo p, given p_inv = 1/p mod 2^N."""
    m00, m01, m10, m11 = t
    # x1, x2 in (-2*p, p)
    if x1 < 0:
        x1 += p
    if x2 < 0:
        x2 += p
    # x1, x2 in (-p, p)
    x1n, x2n = m00*x1 + m01*x2, m10*x1 + m11*x2
    mx1n = -((p_inv * x1n) % 2**N)
    mx2n = -((p_inv * x2n) % 2**N)
    x1n += mx1n * p
    x2n += mx2n * p
    return x1n >> N, x2n >> N
```

#### Avoiding Modulus Operation in `modinv`

After the final iteration, the results $$x_1^{(N)}, x_2^{(N)}$$ lie in $$(-2p, p)$$. We want to map $$x_1$$ into the range $$[0, p)$$ using the following normalization:

```python
def normalize(sign, v, p):
    """Compute sign*v mod p, where v is in range (-2*p, p); output in [0, p)."""
    assert sign == 1 or sign == -1
    if v < 0:
        v += p
    if sign == -1:
        v = -v
    if v < 0:
        v += p
    return v
```

<table data-full-width="true"><thead><tr><th></th><th>sign == -1 and v &#x3C; 0</th><th>sign == -1 and v >= 0</th><th>sign == 1 and v &#x3C; 0</th><th>sign == 1 and v >= 0</th></tr></thead><tbody><tr><td><code>if v &#x3C; 0: v += p</code></td><td><span class="math">-p &#x3C; v &#x3C; p</span></td><td><span class="math">0 \le v &#x3C; p</span></td><td><span class="math">-p &#x3C; v &#x3C; p</span></td><td><span class="math">0 \le v &#x3C; p</span></td></tr><tr><td><code>if sign == -1: v = -v</code></td><td><span class="math">0 &#x3C; v &#x3C; p</span></td><td><span class="math">-p &#x3C; v \le 0</span></td><td><span class="math">-p &#x3C; v &#x3C; p</span></td><td><span class="math">0 \le v &#x3C; p</span></td></tr><tr><td><code>if v &#x3C; 0: v += p</code></td><td><span class="math">0 &#x3C; v &#x3C; p</span></td><td><span class="math">0 \le v &#x3C; p</span></td><td><span class="math">0 &#x3C; v &#x3C; p</span></td><td><span class="math">0 \le v &#x3C; p</span></td></tr></tbody></table>

#### Final Implementation

```python
def modinv(p, p_inv, x):
    """Compute the modular inverse of x mod p, given p_inv=1/p mod 2^N."""
    assert p & 1
    delta, u, v, x1, x2 = 1, p, x, 0, 1
    while v != 0:
        delta, u, v, t = transition_matrix(delta, u, v)
        x1, x2 = update_x1x2_optimized_ver2(x1, x2, t, p, p_inv)
    assert u == 1 or u == -1  
    return normalize(u, x1, p)
```

## Conclusion

Traditional GCD-based modular inverse algorithms are fast but suffer from a major drawback: they are **vulnerable to side-channel attacks** due to conditional branches and comparison operations. This vulnerability can be critical in environments where sensitive key material is involved, such as RSA or ECC.

This article introduced a robust alternative: the **Bernstein–Yang `divstep`-based modular inverse algorithm**. This approach offers several compelling advantages:

* ✅ **Constant-time execution**: Designed to ensure execution time does not vary based on input values
* ✅ **High performance**: Faster than Fermat’s method and comparable to (or better than) classical GCD-based methods
* ✅ **Optimized computation**: Improves efficiency through batch processing via matrix multiplication and by avoiding modulus operations\
  ↪ For additional optimizations, refer to [Bitcoin Core’s safegcd implementation guide](https://github.com/bitcoin-core/secp256k1/blob/master/doc/safegcd_implementation.md)

Ultimately, the `divstep`-based algorithm strikes an excellent balance between **performance and security**. It is especially well-suited for **environments that demand resistance to timing attacks**, such as cryptographic protocol implementations or hardware acceleration contexts.

## References

* [https://eprint.iacr.org/2019/266](https://eprint.iacr.org/2019/266)
* [https://github.com/bitcoin-core/secp256k1/blob/master/doc/safegcd\_implementation.md](https://github.com/bitcoin-core/secp256k1/blob/master/doc/safegcd_implementation.md)

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
