# Toom-Cook Multiplication

## 1. Generalizing Karatsuba: Toom-$$n$$

The Toom–Cook algorithm generalizes [Karatsuba](karatsuba-multiplication.md) (which is Toom-2, where $$n=2$$). For an input polynomial of degree $$k-1$$, Toom-$$n$$ splits it into $$n$$ **sub-polynomials** and reduces the number of necessary recursive multiplications from $$n^2$$ to $$2n-1$$.

### Complexity Comparison

For multiplying two polynomials of size $$k$$ (degree $$k-1$$), the time complexity is determined by the split count $$n$$:

$$
T(k) = O\left(k^{\log_n (2n-1)}\right)
$$

| Algorithm Name         | Split Count(n) | Recursive Calls(2n -1) | Exponent                   | Complexity       |
| ---------------------- | -------------- | ---------------------- | -------------------------- | ---------------- |
| **Karatsuba (Toom-2)** | $$n=2$$        | 3                      | $$\log_2 3 \approx 1.585$$ | $$O(k^{1.585})$$ |
| **Toom-3**             | $$n=3$$        | 5                      | $$\log_3 5 \approx 1.465$$ | $$O(k^{1.465})$$ |
| **Toom-4**             | $$n=4$$        | 7                      | $$\log_4 7 \approx 1.404$$ | $$O(k^{1.404})$$ |

## 2. Toom–Cook Process in $$\mathbb{F}_{P^k}$$ (Toom-3 Example)

Let $$A(x)$$ and $$B(x)$$ be elements in $$\mathbb{F}_{P^k}$$, represented as polynomials of degree approximately $$k-1$$. We aim to compute the polynomial product $$C'(x) = A(x) \cdot B(x)$$ efficiently.

### Step 1: Splitting (Divide)

Divide $$A(x)$$ and $$B(x)$$ into $$n=3$$ **sub-polynomials** ($$A_2, A_1, A_0$$ and $$B_2, B_1, B_0$$) of degree less than $$m \approx k/3$$:

$$
A(x) = A_2(x) x^{2m} + A_1(x) x^m + A_0(x) \\
B(x) = B_2(x) x^{2m} + B_1(x) x^m + B_0(x)
$$

### Step 2: Evaluation

$$
C'(x) = C_4(x) x^{4m} + C_3(x) x^{3m} + C_2(x) x^{2m} + C_1(x) x^m + C_0(x)
$$

The resulting product $$C'(x)$$ has a maximum degree of $$4m$$, meaning it has $$2n-1 = 5$$ unknown coefficients(from $$C_0$$ to $$C_4$$)that must be solved for. Therefore, we need to evaluate $$A(x)$$ and $$B(x)$$ at 5 distinct, carefully chosen **evaluation points** $$e_i \in \mathbb{F}_P$$ (or a small extension of $$\mathbb{F}_P$$).

Commonly chosen evaluation points are $$\{0, 1, -1, 2, \infty\}$$ for $$n=3$$ (Similarly, for $$n=4$$, we'll evaluate on $$\{0, 1, -1, 2, -2, 3, \infty\}$$).

*   **Compute** $$v_i$$ **and** $$w_i$$**:**

    $$
    v_i = A(e_i) \quad \text{and} \quad w_i = B(e_i) \quad \text{for } i = 0, \dots, 4
    $$
*   **Evaluation at infinity:**

    $$
    A(\infty) = A_{n-1}(x), \quad B(\infty) = B_{n-1}(x)
    $$

### Step 3: Pointwise Multiplication

Perform the $$2n-1 = \mathbf{5}$$ **recursive multiplications** ($$U_0, \dots, U_4$$) on the evaluated results. This is where the bulk of the computational cost lies:

$$
U_i = v_i \cdot w_i = C'(e_i)
$$

* Each $$U_i$$ is a product of polynomials (or numbers) of size $$m$$, and these recursive calls continue until the polynomials are small enough to be handled by a classical or base-case multiplication algorithm.

### Step 4: Interpolation (Solve)

Use the $$2n-1 = 5$$ calculated products $$U_i$$ to find the $$2n-1$$ coefficient blocks ($$C_0, \dots, C_4$$) of the unreduced result $$C'(x)$$.

The relationship between the evaluated values $$U_i = C'(e_i)$$ and the unknown coefficients $$C_j$$ is a **Vandermonde system of linear equations**:

$$
\begin{pmatrix}
e_0^0 & e_0^1 & \cdots & e_0^4 \\
e_1^0 & e_1^1 & \cdots & e_1^4 \\
\vdots & \vdots & \ddots & \vdots \\
e_4^0 & e_4^1 & \cdots & e_4^4
\end{pmatrix}
\begin{pmatrix}
C_0 \\
C_1 \\
\vdots \\
C_4
\end{pmatrix}
=
\begin{pmatrix}
U_0 \\
U_1 \\
\vdots \\
U_4
\end{pmatrix}
$$

By pre-calculating the **inverse of the Vandermonde matrix**, the interpolation is performed quickly using only linear combinations (additions and scalar multiplications) of the $$U_i$$ values over $$\mathbb{F}_P$$.

This above explains well why Karatuba is a specialized version of Toom-2.

$$
\begin{pmatrix}
1 & 0 & 0\\
1 & 1 & 1 \\
0 & 0 & 1
\end{pmatrix}
\begin{pmatrix}
C_0 \\
C_1 \\
C_2
\end{pmatrix}
=
\begin{pmatrix}
a_0b_0 \\
a_0b_0 + a_0b_1 + a_1b_0 + a_1b_1\\
a_1b_1
\end{pmatrix}
$$

### Step 5: Recomposition and Final Reduction

1.  **Recomposition:** Combine the resulting coefficient blocks $$C_j$$ using shifts:

    $$
    C'(x) = C_4(x) x^{4m} + C_3(x) x^{3m} + C_2(x) x^{2m} + C_1(x) x^m + C_0(x)
    $$
2.  **Reduction:** Apply the final modulo reduction to get the element in $$\mathbb{F}_{P^k}$$:

    $$
    C(x) = C'(x) \pmod{f(x)}
    $$

## 3. Practical Considerations for $$\mathbb{F}_{P^k}$$

In specialized hardware and software for finite field cryptography (like pairing-based cryptography which uses very large $$k$$), Toom–Cook is highly favored over Karatsuba because of its lower exponent.

* **Choice of** $$n$$**:** The optimal choice for the splitting factor $$n$$ depends on the size of the extension degree $$k$$ and the hardware architecture. Higher $$n$$ reduces the asymptotic complexity but increases the overhead of the Evaluation and Interpolation steps. Typically, $$n=3$$ or $$n=4$$ provides the best real-world performance gain before the overhead becomes prohibitive.
* **Base Field Operations:** All additions, subtractions, and scalar multiplications are performed within the base field $$\mathbb{F}_P$$, which must itself be computed efficiently (often using Karatsuba or classical methods for the underlying integer arithmetic).

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze
