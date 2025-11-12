# Euclidean Algorithm

## Objective

The **Euclidean Algorithm** efficiently computes the **Greatest Common Divisor (GCD)** of two integers $$A$$ and $$B$$.

The core idea is based on the following identity:

$$
\gcd(A, B) = \gcd(B, A \bmod B)
$$

## Code

```python
def gcd(a, b):
    if b == 0:
        return a
    else:
        return gcd(b, a % b)
```

## Proof

Let $$G = \gcd(A, B)$$. Then, we can write $$A=aG$$, $$B=bG$$, where $$a$$ and $$b$$ are coprime integers.

Dividing $$A$$ by $$B$$ gives:

$$
A = Bq + R
$$

Then:

$$
R = A - Bq = (a - bq)G = rG
$$

So $$R = rG$$, and we want to prove that $$\gcd(B, R) = G$$ is true if and only if $$b$$ and $$r$$ are coprime.

Suppose, for contradiction, that $$b$$ and $$r$$ are **not** coprime. Then there exists a common divisor $$m > 1$$ such that:

$$
b = mb', \quad r = mr'
$$

where $$b'$$ and $$r'$$ are coprime. Then:

$$
\begin{align*}
a &= bq + r \\
&= mb'q + mr' \\
&= m(b'q + r')
\end{align*}
$$

This means $$a$$ is also divisible by $$m$$, implying $$a$$ and $$b$$ share a common factor $$m$$, contradicting the assumption that $$a$$ and $$b$$ are coprime. Thus, $$b$$ and $$r$$ must be coprime.

Proving the reverse is trivial, so we don't prove it here.

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze
