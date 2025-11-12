# Edwards Curve

## Definition

An **Edwards Curve** is a special form of elliptic curve introduced by Harold Edwards in 2007, defined over a field $$\mathbb{F}_q$$ by the following equation:

$$
x^2 + y^2 = 1 + dx^2y^2
$$

where $$d \in \mathbb{F}_q \setminus \{0, 1\}$$. This form is called the **(original) Edwards Form**. In practice, a more general and widely used variant is the **Twisted Edwards Form**:

$$
ax^2 + y^2 = 1 + dx^2y^2
$$

where $$a, d \in \mathbb{F}_q$$, $$a \ne 0$$, $$d \ne 0$$, and the curve is non-singular if $$a \ne d$$.

Edwards curves are particularly useful in cryptography because they offer **efficient and complete** point addition formulas and resist many implementation bugs like those caused by exceptions in traditional [Weierstrass addition](../weierstrass-curve/#point-additions-short-weierstrass-form).

## Addition (Twisted Edwards Form)

Let $$P_1 = (x_1, y_1)$$ and $$P_2 = (x_2, y_2)$$ be two points on a **Twisted Edwards curve** defined by:

$$
ax^2 + y^2 = 1 + dx^2y^2
$$

Then the sum $$P_3 = P_1 + P_2 = (x_3, y_3)$$ is given by:

$$
x_3 = \frac{x_1 y_2 + y_1 x_2}{1 + d x_1 x_2 y_1 y_2} \\
y_3 = \frac{y_1 y_2 - a x_1 x_2}{1 - d x_1 x_2 y_1 y_2}
$$

These formulas are **complete** over prime fields if $$d$$ is a non-square, meaning they **work for all inputs**, unlike the Weierstrass formulas which require case distinctions and exception handling (e.g., $$P = Q$$, $$y = 0$$, etc.).

## Negation (Twisted Edwards Form)

The **additive inverse** of a point $$P = (x, y)$$ on a Twisted Edwards curve is:

$$
-P = (-x, y)
$$

This is because:

* The x-coordinate changes sign,
* The y-coordinate remains the same,
* And:

$$
(x, y) + (-x, y) = \mathcal{O}
$$

where $$\mathcal{O} = (0, 1)$$ is the **identity element** of the group (just like $$\mathcal{O}$$ or "point at infinity" in Weierstrass form).

### Why does this work?

Let’s verify algebraically:

Using the addition formula:

* $$x_1 = x$$, $$y_1 = y$$
* $$x_2 = -x$$, $$y_2 = y$$

Then:

$$
x_3 = \frac{x y + y (-x)}{1 + d x (-x) y y} = \frac{0}{1 - d x^2 y^2} = 0 \\
y_3 = \frac{y y - a x (-x)}{1 - d x (-x) y y} = \frac{y^2 + a x^2}{1 - d x^2 y^2}
$$

If you substitute this back into the curve equation, you’ll find that the result corresponds to the **identity point** $$\mathcal{O} = (0,1)$$, confirming that $$(x,y) + (-x,y) = \mathcal{O}$$.

## Benefits of Edwards Curves

* ✅ **Complete addition formulas** (no exceptions)
* ✅ **Efficient computation** (fewer field multiplications than Weierstrass)
* ✅ **Better resistance to side-channel attacks** due to uniform operation patterns
* ✅ **Symmetry** in $$x$$ and $$y$$ makes certain transformations easier

These features make Edwards curves a popular choice in cryptographic systems such as:

* **Ed25519**: widely used digital signature scheme (used in Signal, SSH, OpenSSH, etc.)
* **Curve25519**: used for key exchange (X25519 in TLS, etc.)

## References

* [Daniel J. Bernstein et al., "Twisted Edwards Curves"](https://eprint.iacr.org/2008/013)

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze
