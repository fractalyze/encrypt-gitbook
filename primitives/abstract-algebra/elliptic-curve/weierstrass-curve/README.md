# Weierstrass Curve

## Definition

An elliptic curve over a field is commonly defined using the **Weierstrass equation**, which appears in multiple forms. The **general Weierstrass form** is:

$$
y^2 + a_1xy + a_3y = x^3 + a_2x^2 + a_4x + a_6
$$

This equation defines a **non-singular cubic curve**, given certain conditions on the coefficients $$a_1, a_2, a_3, a_4, a_6$$ to ensure smoothness (i.e., the curve has no cusps or self-intersections).

However, when the field has characteristic not equal to 2 or 3, we can simplify this equation via a change of variables into a more convenient form, known as the **Short Weierstrass Form**:

$$
y^2 = x^3 + ax + b
$$

This is the form most commonly used in cryptography. In this case, the curve is uniquely determined by the values of $$a$$ and $$b$$, and the non-singularity condition becomes:

$$
4a^3 + 27b^2 \ne 0
$$

The set of points on an elliptic curve forms an [additive abelian group](../../group/#properties), with a well-defined addition operation.

Note that $$\mathcal{O} = (0,1,0)$$ refers to the **point at infinity**, or the **identity point**, as defined in [projective form](coordinate-forms.md#projective).

## Addition (Short Weierstrass Form)

We define point addition geometrically through the relation:

$$
P + Q + R = \mathcal{O}
$$

That is, three colinear points on an elliptic curve (in affine form) sum to the identity. Hence, the sum of two points is the reflection of the third point over the x-axis:

$$
P + Q = -R
$$

<figure><img src="../../../../.gitbook/assets/a.png" alt="" width="324"><figcaption></figcaption></figure>

### **Calculating Point Additions**

Let $$P = (x_P, y_P)$$, $$Q = (x_Q, y_Q)$$, and $$-R = (x_R, y_R)$$. The formulas depend on whether we are **adding** or **doubling** points.

#### **Case 1. Adding:** $$P \ne Q \to P + Q = (x_P,y_P) + (x_Q,y_Q) = (x_R,y_R) = -R$$&#x20;

$$
m = \frac{y_Q - y_P}{x_Q - x_P} \\
x_R = m^2 - x_P - x_Q \\
y_R = m(x_P - x_R) - y_P
$$

#### **Case 2. Doubling:** $$P = Q \to 2P = 2(x_P,y_P)= (x_P,y_P) + (x_P,y_P) =(x_R,y_R) = -R$$

$$
m = \frac{3x_P^2 + a}{2y_P} \\
x_R = m^2 - 2x_P \\
y_R = m(x_P - x_R) - y_P
$$

## Negation (Short Weierstrass Form)

The **additive inverse** of a point $$P = (x, y)$$ on a Short Weierstrass curve is:

$$
-P = (x, -y)
$$

### **Why?**

The curve is symmetric about the **x-axis**, because the equation contains $$y^2$$ (even power), so flipping the sign of $$y$$ still satisfies the curve equation:

$$
(-y)^2 = y^2 = x^3 + ax + b
$$

So:

$$
P + (-P) = \mathcal{O}
$$

This makes point negation simple and geometric: just reflect the point across the x-axis.

## **Confirming Additive Abelian Group Properties**

* **Closure**: If $$P$$, $$Q$$ are on the curve, then $$P + Q$$ is also on the curve ✅
* **Associativity**: $$P + (Q + R) = (P + Q) + R$$ ✅
* **Identity**: $$P + \mathcal{O} = P$$ ✅
* **Inverse**: $$P + (-P) = \mathcal{O}$$ ✅
* **Commutativity**: $$P + Q = Q + P$$ ✅

## References

* [Elliptic curves: A gentle Intro](https://andrea.corbellini.name/2015/05/17/elliptic-curve-cryptography-a-gentle-introduction/)
* [A (Relatively Easy To Understand) Primer on Elliptic Curve Cryptography](https://blog.cloudflare.com/a-relatively-easy-to-understand-primer-on-elliptic-curve-cryptography/)

> Written by [Ashley Jeong](https://app.gitbook.com/u/Z4DJjtMFfLVXwMaSBRDBJRWVDV12 "mention") of A41
