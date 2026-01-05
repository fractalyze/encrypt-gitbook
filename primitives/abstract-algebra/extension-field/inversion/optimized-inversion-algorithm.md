---
description: 'Presentation: https://www.youtube.com/watch?v=QNxgKYcc-A0'
---

# Optimized Inversion Algorithm

Finite-field inversion can be optimized when the extension degree is small and the field's algebraic structure is well-defined. Specifically, extension fields of degrees three and four enable inversion to be formulated in closed form, leveraging low-dimensional linear algebra and quadratic subextensions.

For the quadratic extension ($$\mathbb{F}_{p^2}$$), inversion admits a particularly simple closed form. Writing an element as $$x = x_0 + x_1 u$$, the inverse is obtained by first forming a base-field denominator $$x_0^2 - \eta x_1^2$$ and then multiplying its inverse by the conjugate $$x_0 - x_1 u$$.

For the cubic extension ($$\mathbb{F}_{p^3}$$), multiplication by a fixed element defines an $$\mathbb{F}_p$$ -linear endomorphism of a three-dimensional vector space. Inversion can therefore be obtained by inverting this linear transformation, leading to explicit formulas involving determinants and adjugate matrices.

For the quartic extension ($$\mathbb{F}_{p^4}$$), the field admits a natural representation as a quadratic extension over $$\mathbb{F}_{p^2}$$. This tower structure enables inversion via conjugation with respect to the intermediate field, reducing the problem to the inversion of a single element in $$\mathbb{F}_{p^2}$$.

### 1. Inversion in $$\mathbb{F}_{p^3}$$

#### 1.1 Field Definition

Let

$$
\mathbb{F}_{p^3} = \mathbb{F}_p[u] / (u^3 - \xi),
$$

where $$\xi \in \mathbb{F}_p$$ is fixed. Every element $$x \in \mathbb{F}_{p^3}$$ is written uniquely as

$$
x = x_0 + x_1 u + x_2 u^2, \qquad x_0, x_1, x_2 \in \mathbb{F}_p.
$$

#### 1.2 Multiplication as a Linear Operator

Multiplication by $$x$$ defines an $$\mathbb{F}_p$$-linear map $$L_x : \mathbb{F}_{p^3} \to \mathbb{F}_{p^3}$$ given by $$L_x(y) = x y$$. With respect to the basis $$\{1, u, u^2\}$$, this map is represented by

$$
M_x =
\begin{pmatrix}
x_0 & \xi x_2 & \xi x_1 \\
x_1 & x_0     & \xi x_2 \\
x_2 & x_1     & x_0
\end{pmatrix}.
$$

#### 1.3 Inversion via Linear Algebra

Assume $$x \neq 0$$. Then $$L_x$$ is invertible and the inverse element satisfies $$x^{-1} = L_x^{-1}(1)$$. By [Cramer’s rule](https://en.wikipedia.org/wiki/Cramer's_rule),

$$
M_x^{-1} = \det(M_x)^{-1} \operatorname{adj}(M_x),
$$

so the coordinates of $$x^{-1}$$ are given by the first column of $$\operatorname{adj}(M_x)$$.

#### 1.4 Explicit Inversion Formula

1. $$t_0, t_1, t_2$$: The First Column of the Adjugate Matrix

* $$t_0 = x_0^2 - \xi x_1 x_2$$
  * This is the cofactor of the $$M_x[1, 1]$$, where $$M_x[r, c]$$ is the element at row $$r$$ and coloum $$c$$.
  * It is calculated as the determinant of the $$2 \times 2$$ matrix $$\begin{pmatrix} x_0 & \xi x_2 \\ x_1 & x_0 \end{pmatrix}$$ remaining after crossing out the 1st row and 1st column.
* $$t_1 = \xi x_2^2 - x_0 x_1$$
  * This is the cofactor for the $$M_x[1, 2]$$.
  * The sign $$(-)$$ is already integrated into the simplified formula in the code.
* $$t_2 = x_1^2 - x_0 x_2$$
  * This is the cofactor for the $$M_x[1, 3]$$.

These $$t_0, t_1, t_2$$ constitute the **first column of the adjugate matrix** $$\text{adj}(M_x)$$. In extension fields, the matrix has a **circulant** structure, meaning that knowing the first column is sufficient to determine the entire inverse.<br>

2. $$t_3$$: The Determinant

$$
t_3 = x_0 t_0 + \xi (x_2 t_1 + x_1 t_2)
$$

* This is the [**Laplace expansion**](https://en.wikipedia.org/wiki/Laplace_expansion) along the first column of matrix $$M_x$$.
* It performs $$M_x[1, 1] \cdot t_0 + M[1, 2] \cdot t_1 + M[1,3] \cdot t_3$$

2. $$y = (y_0, y_1, y_2)$$: The Final Inverse

$$
(y_0, y_1, y_2) = t_3^{-1} \cdot (t_0, t_1, t_2)
$$

***

### 2. Inversion in the Quartic Extension Field

Direct inversion via a $$4 \times 4$$ matrix is inefficient. Instead, $$\mathbb{F}_{p^4}$$ is viewed as a quadratic extension over $$\mathbb{F}_{p^2}$$:

$$
\mathbb{F}_p
\;\xrightarrow{\, v^2 = \xi \,}\;
\mathbb{F}_{p^2}
\;\xrightarrow{\, u^2 = v \,}\;
\mathbb{F}_{p^4}.
$$

#### 2.1 Tower Construction

Let $$\mathbb{F}_{p^2} = \mathbb{F}_p[v]/(v^2 - \xi)$$ and $$\mathbb{F}_{p^4} = \mathbb{F}_{p^2}[u]/(u^2 - v)$$.

Every element $$x \in \mathbb{F}_{p^4}$$ can be written as $$x = A + B u$$ with $$A, B \in \mathbb{F}_{p^2}$$.

$$
\begin{align*}
x &= x_0 + x_1u + x_2u^2 + x_3u^3 \\
&= (x_0 + x_2u^2) + (x_1 + x_3u^2)u \\
&= (x_0 + x_2v) + (x_1 + x_3v)u
\end{align*}
$$

#### 2.2 Quadratic Conjugation

Define the conjugate of $$x$$ over $$\mathbb{F}_{p^2}$$ by $$\bar{x} = A - B u$$. This operation satisfies $$x \bar{x} \in \mathbb{F}_{p^2}$$.

#### 2.3 First Level of Rationalization (Removing $$u$$)

Our goal is to find $$x^{-1} = \frac{1}{A+Bu}$$. To remove $$u$$ from the denominator, we multiply the top and bottom by the conjugate $$\bar{x} = A - Bu$$:

$$
x^{-1} = \frac{1}{A+Bu} \cdot \frac{A-Bu}{A-Bu} = \frac{A-Bu}{A^2 - B^2u^2}
$$

Since $$u^2 = v$$, the denominator becomes $$D = A^2 - B^2v$$.

Now, $$u$$ has been eliminated from the denominator, leaving only $$v$$.

#### 2.4 Second Level of Rationalization (Removing $$v$$)

Let's look at the new denominator $$D$$. Since $$A$$ and $$B$$ contain $$v$$, $$D$$ can be expanded into a linear form of $$v$$: $$D = D_0 + D_1v$$. To remove $$v$$, we multiply by its conjugate $$(D_0 - D_1v)$$:

$$
x^{-1} = \frac{A - Bu}{D_0 + D_1v} \cdot \frac{D_0 - D_1v}{D_0 - D_1v} = \frac{(A - Bu) \cdot (D_0 - D_1v)}{D_0^2 - D_1^2v^2}
$$

Since $$v^2 = \xi$$, the final denominator becomes $$N = D_0^2 - D_1^2\xi$$.

This value $$N$$ (the Norm) is now a pure scalar (BaseField), free of both $$u$$ and $$v$$.

#### 2.5 Inversion Formula

Assume $$x \neq 0$$. Then $$D \neq 0$$, and the inverse of $$x$$ is

$$
x^{-1} = D^{-1} (A - B u).
$$

Thus inversion in $$\mathbb{F}_{p^4}$$ reduces to inversion in $$\mathbb{F}_{p^2}$$.

#### 2.6 Inversion in the Quadratic Subfield

Writing $$D = D_0 + D_1 v$$ with $$D_0, D_1 \in \mathbb{F}_p$$, one has

$$
D^{-1} = (D_0^2 - \xi D_1^2)^{-1} (D_0 - D_1 v).
$$

This completes the inversion algorithm using only base-field arithmetic.

### 3. Performance Comparison with the [Itoh–Tsujii Algorithm](../inversion.md)

This section compares the specialized inversion methods for low-degree extension fields with the Itoh–Tsujii algorithm from the perspective of arithmetic cost. The comparison is expressed in terms of operations over the prime field $$\mathbb{F}_p$$.

Throughout, it is assumed that:

* Frobenius applications are free or negligible compared to multiplication,
* squaring and multiplication in $$\mathbb{F}_p$$ are counted separately

***

#### 3.1 Arithmetic Cost Summary

<table><thead><tr><th width="82.76171875">Field</th><th width="125.5078125">Method</th><th>Square</th><th>Multiplication</th><th>Inversion</th><th>Structural Cost</th></tr></thead><tbody><tr><td><span class="math">\mathbb{F}_{p^3}</span></td><td>Matrix-based inversion</td><td><span class="math">3</span></td><td><span class="math">9</span></td><td><span class="math">1</span></td><td>Linear algebra (determinant + adjugate)</td></tr><tr><td><span class="math">\mathbb{F}_{p^3}</span></td><td>Itoh–Tsujii</td><td><span class="math">3</span></td><td><span class="math">\approx 12</span></td><td><span class="math">1</span></td><td>Extension-field multiplications</td></tr><tr><td><span class="math">\mathbb{F}_{p^4}</span></td><td>Quadratic tower inversion</td><td><span class="math">6</span></td><td><span class="math">12</span></td><td><span class="math">1</span></td><td>Two successive quadratic reductions</td></tr><tr><td><span class="math">\mathbb{F}_{p^4}</span></td><td>Itoh–Tsujii</td><td><span class="math">\approx 8</span></td><td><span class="math">\approx 18</span></td><td><span class="math">1</span></td><td>Flat exponentiation in full extension</td></tr></tbody></table>

#### 3.2 Interpretation

For both cubic and quartic extensions, the specialized inversion formulas reduce the number of base-field multiplications by avoiding full extension-field multiplication.

* In the cubic case, inversion is obtained via explicit determinant and adjugate computation, keeping all arithmetic in $$\mathbb{F}_p$$.
* In the quartic case, inversion proceeds through a quadratic tower, so that all intermediate inversions occur in proper subfields.

In contrast, the Itoh–Tsujii algorithm operates uniformly on the full extension field $$\mathbb{F}_{p^k}$$ and requires additional extension-field multiplications, which expand to a larger number of base-field operations.

#### 3.3 Asymptotic Perspective

However, for small values of $$k$$, the constant factors dominate:

* Specialized methods achieve lower arithmetic cost by exploiting explicit low-degree structure.
* Itoh–Tsujii trades higher constant factors for generality and uniformity across extension degrees.

As a result, for extension degrees $$k = 3$$ and $$k = 4$$, the specialized algorithms are typically preferred.

### A. Possibility of Inversion over a Quadratic Tower

#### A.1 Setting

Consider an extension field of degree $$2^n$$ over the prime field $$\mathbb{F}_p$$. Assume that the field admits a recursive quadratic tower representation of the form

$$
\mathbb{F}_{p^{2^n}}
=
\mathbb{F}_{p^{2^{n-1}}}[u_{n-1}] / (u_{n-1}^2 - \eta_{n-1}),
$$

where $$\eta_{n-1} \in \mathbb{F}_{p^{2^{n-1}}}$$ is a non-square.

#### A.2 Quadratic Reduction Step

At each level $$i$$ of the tower, an element of $$\mathbb{F}_{p^{2^{i+1}}}$$ can be written as $$x = x_0 + x_1 u_i$$ with $$x_0, x_1 \in \mathbb{F}_{p^{2^i}}$$. The inverse of $$x$$ is computed by:

* forming the denominator $$x_0^2 - \eta_i x_1^2 \in \mathbb{F}_{p^{2^i}}$$,
* multiplying its inverse by the conjugate $$x_0 - x_1 u_i$$.

This reduces inversion in $$\mathbb{F}_{p^{2^{i+1}}}$$ to a single inversion in $$\mathbb{F}_{p^{2^i}}$$.

#### A.3 Recursive Structure

By recursively applying the quadratic reduction step, inversion in $$\mathbb{F}_{p^{2^n}}$$ is reduced along the tower

$$
\mathbb{F}_{p^{2^n}} \rightarrow \mathbb{F}_{p^{2^{n-1}}} \rightarrow \cdots \rightarrow \mathbb{F}_{p^2} \rightarrow \mathbb{F}_p.
$$

As a result, exactly one inversion in the base field $$\mathbb{F}_p$$ is required, together with a bounded number of multiplications and squarings in intermediate subfields.

#### A.4 Applicability Conditions

This approach is applicable if and only if:

* the extension field is explicitly represented as a quadratic tower,
* a suitable non-residue $$\eta_i$$ is available at each level of the tower.

If the field is instead given as a flat extension $$\mathbb{F}_p[x]/(f(x))$$ with $$\deg f = 2^n$$, a compatible quadratic tower representation may not be immediately available and may require nontrivial basis transformations.

#### A.5 Limitations

Although quadratic tower inversion is structurally simple and implementation-friendly, it is not universally optimal. For large values of $$n$$, the accumulated cost of intermediate multiplications may outweigh the benefits of avoiding extension-field exponentiation. Moreover, the existence of suitable non-residues is field-dependent and must be ensured by construction.

> Written by [Soowon Jeong](https://app.gitbook.com/u/gkru4r7198XKD3pCvVYtqakZB3Z2 "mention") of Fractalyze
