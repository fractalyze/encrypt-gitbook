---
description: Reduction of Hamming weight to ~1/3
---

# NAF (Non-adjacent form)

## Motivation

The cost of [Double-and-add](abstract-algebra/elliptic-curve/scalar-multiplication/double-and-add.md) is directly proportional to the [Hamming weight](https://en.wikipedia.org/wiki/Hamming_weight) or **the number of non-zero digits** of the scalar's binary form. NAF attempts to reduce the Hamming weight in comparison to unsigned bit representation.

## Definition

Non-adjacent form is defined as a unique signed-bit representation of a positive number where non-zero values are non-adjacent. Note that unsigned-bit representation results in an average Hamming weight of around $$\frac{1}{2}$$ the total bits. In comparison, for NAF, the property for having non-zero values be non-adjacent reduces the average Hamming weight to around $$\frac{1}{3}$$ the total bits.

## Examples

$$
\begin{array}{cll}
& \quad \text{unsigned binary}
& \quad \quad \quad \text{NAF Form}\\
11 &= \underbrace{1}_{2^3}\underbrace{0}_{2^2}\underbrace{1}_{2^1}\underbrace{1}_{2^0}
\quad \quad &= \underbrace{1}_{2^4}\underbrace{0}_{2^3}\underbrace{-1}_{2^2}\underbrace{0}_{2^1}\underbrace{-1}_{2^0} = 16 - 4-1\\
12 &= \underbrace{1}_{2^3}\underbrace{1}_{2^2}\underbrace{0}_{2^1}\underbrace{0}_{2^0}
&= \underbrace{1}_{2^4}\underbrace{0}_{2^3}\underbrace{-1}_{2^2}\underbrace{0}_{2^1}\underbrace{0}_{2^0} = 16 - 4\\
13 &= \underbrace{1}_{2^3}\underbrace{1}_{2^2}\underbrace{0}_{2^1}\underbrace{1}_{2^0}
&= \underbrace{1}_{2^4}\underbrace{0}_{2^3}\underbrace{-1}_{2^2}\underbrace{0}_{2^1}\underbrace{1}_{2^0} = 16 - 4 + 1\\
14 &= \underbrace{1}_{2^3}\underbrace{1}_{2^2}\underbrace{1}_{2^1}\underbrace{0}_{2^0}
&= \underbrace{1}_{2^4}\underbrace{0}_{2^3}\underbrace{0}_{2^2}\underbrace{-1}_{2^1}\underbrace{0}_{2^0} = 16 - 2\\
15 &= \underbrace{1}_{2^3}\underbrace{1}_{2^2}\underbrace{1}_{2^1}\underbrace{1}_{2^0}
&= \underbrace{1}_{2^4}\underbrace{0}_{2^3}\underbrace{0}_{2^2}\underbrace{0}_{2^1}\underbrace{-1}_{2^0} = 16 - 1\\
\end{array}
$$

## References

* [https://hackmd.io/HkVWGwsRTM2HeBL1VN0lAw](https://hackmd.io/HkVWGwsRTM2HeBL1VN0lAw)

> Written by [Ashley Jeong](https://app.gitbook.com/u/Z4DJjtMFfLVXwMaSBRDBJRWVDV12 "mention") of A41
