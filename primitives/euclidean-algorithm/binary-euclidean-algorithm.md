# Binary Euclidean Algorithm

## Objective

The **Binary Euclidean Algorithm**, also known as **Stein’s algorithm**, is an efficient method for computing the **greatest common divisor (GCD)** of two integers. Unlike the standard Euclidean algorithm, which relies on division, the binary version replaces division with more efficient **bitwise operations** such as shifts and subtractions. This makes it highly suitable for implementation in both hardware and low-level software.

## Core Idea

$$
\gcd(A, B) = \begin{cases}
A & \text{ if } B = 0 \\
B & \text{ if } A = 0 \\
2 \times \gcd \left(\frac{A}{2}, \frac{B}{2} \right) & \text{ if } A  \equiv 0 \pmod 2 \land \ B \equiv 0 \pmod 2\\
\gcd \left(\frac{A}{2}, B \right) & \text{ if } A \equiv 0 \pmod 2 \land \ B \equiv 1 \pmod 2 \\
\gcd \left(A, \frac{B}{2} \right) & \text{ if } A \equiv 1 \pmod 2 \land \ B \equiv 0 \pmod 2 \\
\gcd \left(\frac{|A - B|}{2}, \min(A, B) \right) & \text{ if } A \equiv 1 \pmod 2 \land \ B \equiv 1 \pmod 2 \\
\end{cases}
$$

$$\gcd(A,B)$$ is transformed according to the following rules:

1. If $$B = 0$$, then $$\gcd(A, B) = A$$
2. If $$A = 0$$, then $$\gcd(A, B) = B$$
3. If both $$A$$ and $$B$$ are even, then $$\gcd(A, B) = 2 \times \gcd\left(\frac{A}{2}, \frac{B}{2}\right)$$ because 2 is a common factor
4. If only $$A$$ is even, then $$\gcd(A, B) = \gcd\left(\frac{A}{2}, B\right)$$, because $$B$$ does not have a factor of 2
5. If only $$B$$ is even, then $$\gcd(A, B) = \gcd\left(A, \frac{B}{2}\right)$$, because $$A$$ does not have a factor of 2
6. If both $$A$$ and $$B$$ are odd, then $$\gcd(A, B) = \gcd\left(\frac{|A - B|}{2}, \min(A, B)\right)$$ because $$\gcd(A, B) = \gcd(B, A)$$, and if $$A > B$$, then $$\gcd(A, B) = \gcd(A - B, B)$$. Since $$A−B$$ is even, we divide it by 2.

## Code

```python
def binary_gcd(a, b):
    # Base case
    if a == 0: return b
    if b == 0: return a

    # Case: both even
    if a % 2 == 0 and b % 2 == 0:
        return 2 * binary_gcd(a // 2, b // 2)
    # Case: a even, b odd
    elif a % 2 == 0:
        return binary_gcd(a // 2, b)
    # Case: a odd, b even
    elif b % 2 == 0:
        return binary_gcd(a, b // 2)
    # Case: both odd
    elif a >= b:
        return binary_gcd((a - b) // 2, b)
    else:
        return binary_gcd((b - a) // 2, a)
```

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
