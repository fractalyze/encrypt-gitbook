# Extended Euclidean Algorithm

## Objective

The **Extended Euclidean Algorithm** not only computes $$\gcd(A, B)$$, but also finds **integers** $$x$$ **and** $$y$$ that satisfy:

$$
Ax + By = \gcd(A, B)
$$

The pair $$(x, y)$$ is also known as the [**Bézout coefficients**](https://en.wikipedia.org/wiki/B%C3%A9zout's_identity).

The **Extended Euclidean Algorithm** is used to compute the [**modular inverse**](../modular-arithmetic/modular-inverse/) of an integer, which is essential in many cryptographic algorithms like RSA, ECC, and digital signatures.

## Code (Recursive Version)

```python
def extended_gcd(a, b):
    if b == 0:
        return (a, 1, 0)
    else:
        d, x1, y1 = extended_gcd(b, a % b)
        x = y1
        y = x1 - (a // b) * y1
        return (d, x, y)
```

## Proof

**Base Case:** $$B = 0$$

If $$B = 0$$, then:

$$
\gcd(A, 0) = A \Rightarrow x = 1,\ y = 0
$$

which satisfies the identity:

$$
A \cdot 1 + B \cdot 0
$$

***

**Inductive Hypothesis:**\
Assume the recursive call returns $$(d, x_1, y_1)$$ such that:

$$
d = b \cdot x_1 + (A \bmod B) \cdot y_1
$$

Then:

$$
\begin{align*}
d &= B \cdot x_1+ \left(A - \left\lfloor\frac{A}{B}\right\rfloor \cdot B\right) \cdot y1 \\
&= A \cdot y_1 + B\left(x_1 - \left\lfloor\frac{A}{B}\right\rfloor  \cdot y1 \right)
\end{align*}
$$

Now define:

$$
x = y_1, \quad y = x_1 - \left\lfloor\frac{A}{B}\right\rfloor  \cdot y1
$$

So:

$$
Ax + By = d
$$

which proves that the identity holds at each step.

## Code (Non-Recursive Version)

```python
def extended_gcd(a, b):
    # Initialize: (r, s, t) ← (a, 1, 0), (r', s', t') ← (b, 0, 1)
    r_prev, r = a, b
    s_prev, s = 1, 0
    t_prev, t = 0, 1

    while r != 0:
        q = r_prev // r  # quotient

        # Update (r, s, t)
        r_prev, r = r, r_prev - q * r
        s_prev, s = s, s_prev - q * s
        t_prev, t = t, t_prev - q * t

    # At termination: r_prev = gcd(a, b), s_prev = x, t_prev = y
    return r_prev, s_prev, t_prev
```

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
