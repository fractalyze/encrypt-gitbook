# Extended Binary Euclidean Algorithm

## Core Idea

This code operates as follows:

1. It removes the common factor $$2^k$$ from $$A = 2^k a$$ and $$B = 2^k b$$.
2. Then, it applies the Binary GCD rules starting from $$u = a$$ and $$v = b$$, and reduces $$u$$ and $$v$$ until they become equal. The case where both $$u$$ and $$v$$ are even has already been handled in the previous step, so it's excluded here.

$$
\gcd(u, v) = \begin{cases}
\gcd(u, v) & \text{ if } u  = v\\
\gcd \left(\frac{u}{2}, v \right) & \text{ if } u \equiv 0 \pmod 2 \land \ v \equiv 1 \pmod 2 \\
\gcd \left(u, \frac{v}{2} \right) & \text{ if } u \equiv 1 \pmod 2 \land \ v \equiv 0 \pmod 2 \\
\gcd \left(u - v, v \right) & \text{ if } u > v \land u \equiv 1 \pmod 2 \land \ v \equiv 1 \pmod 2 \\
\gcd \left(u, v - u \right) & \text{ if } v > u \land u \equiv 1 \pmod 2 \land \ v \equiv 1 \pmod 2 \\
\end{cases}
$$

Meanwhile, we set the variables as $$x_1 = 1, y_1 = 0, x_2 = 0, y_2 = 1$$, and maintain the following two invariants:

$$
ax_1 + by_1 = u, \quad ax_2 + by_2 = v
$$

At each step, we update the expressions in a way that preserves these invariants. Let’s consider only case 2 and case 4 as examples.

### Case 2: When $$u$$ is even and $$v$$ is odd

If $$x_1$$ and $$y_1$$ are both even, we update as follows:

$$
ax_1 + by_1 = u \Leftrightarrow a\frac{x_1}{2} + b\frac{y_1}{2} = \frac{u}{2}
$$

Otherwise (if either is odd), we update as:

$$
ax_1 + by_1 = u \Leftrightarrow a\frac{(x_1 + b)}{2} + b\frac{(y_1 - a)}{2} = \frac{u}{2}
$$

Why are $$x_1 + b$$ and $$y_1 - a$$ always even?&#x20;

Since $$u$$ is even, the expression $$a x_1 + b y_1$$​ must also be even. A sum is even only if both terms are even or both are odd.

Now, note that $$a$$ and $$b$$ cannot both be even — because we already factored out the common power of 2 beforehand.

| a    | x₁   | a \* x₁ | b    | y₁   | b \* y₁ |
| ---- | ---- | ------- | ---- | ---- | ------- |
| odd  | odd  | odd     | odd  | odd  | odd     |
| even | odd  | even    | odd  | even | even    |
| odd  | even | even    | even | odd  | even    |

In each of these cases, the sum $$a x_1 + b y_1$$ is even, and both $$x_1 + b$$ and $$y_1 - a$$ are even as well.

### Case 4: When $$u > v$$ and both $$u$$ and $$v$$ are odd

$$
ax_1 + by_1 = u,\quad ax_2 + by_2 = v \Leftrightarrow a(x_1 - x_2) + b(y_1 - y_2) = u - v
$$

## Code

```python
def binary_extended_gcd(a, b):
    # base case: if a or b is 0, return the other number as gcd, with appropriate coefficients
    if a == 0: return (b, 0, 1)
    if b == 0: return (a, 1, 0)

    # Step 1: Remove common factors of 2
    # gcd(2a, 2b) = 2 * gcd(a, b), so we factor out all common 2s
    shift = 0
    while a % 2 == 0 and b % 2 == 0:
        a //= 2
        b //= 2
        shift += 1  # keep track of how many times we factored out 2

    # Step 2: Initialize variables for the extended GCD
    u, v = a, b              # Working values to reduce down to GCD
    x1, y1 = 1, 0            # Bézout coefficients for u
    x2, y2 = 0, 1            # Bézout coefficients for v

    # Step 3: Main loop to compute GCD while updating Bézout coefficients
    while u != v:
        if u % 2 == 0:
            # If u is even, divide by 2
            u //= 2
            # Adjust (x1, y1) to keep Bézout identity valid
            if x1 % 2 == 0 and y1 % 2 == 0:
                x1 //= 2
                y1 //= 2
            else:
                # If odd, we adjust using the original a, b to keep x1, y1 as integers
                x1 = (x1 + b) // 2
                y1 = (y1 - a) // 2

        elif v % 2 == 0:
            # Same logic for v if it is even
            v //= 2
            if x2 % 2 == 0 and y2 % 2 == 0:
                x2 //= 2
                y2 //= 2
            else:
                x2 = (x2 + b) // 2
                y2 = (y2 - a) // 2

        elif u > v:
            # Reduce u by subtracting v
            u -= v
            x1 -= x2
            y1 -= y2
        else:
            # Reduce v by subtracting u
            v -= u
            x2 -= x1
            y2 -= y1

    # Step 4: Multiply back the common factor of 2 (if any)
    # At this point, u == v == gcd(a, b)
    return (u << shift, x1, y1)  # gcd * 2^shift, x, y
```

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze
