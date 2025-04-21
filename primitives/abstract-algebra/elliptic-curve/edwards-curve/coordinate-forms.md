# Coordinate Forms

## Affine $$(x, y)$$

Affine form refers to the regular $$(x,y)$$ form of coordinates on an $$xy$$-plane.

### **Problem**

In affine form, every addition or doubling requires field inversion when computing fractions (e.g., $$\frac{1}{1 + dx_1x_2y_1y_2}$$). Like with Weierstrass curves, this becomes the bottleneck for performance.

## Extended Affine $$(x, y, xy)$$

**Extended Affine form** is a lightweight optimization over the basic affine coordinates. It explicitly stores the product $$xy$$, which is used repeatedly in twisted Edwards point addition formulas. This avoids redundant multiplications and slightly speeds up scalar multiplication loops.

### Point Representation:

A point is stored as:

$$
(x, y, xy)
$$

This means we store:

* $$x$$: the affine x-coordinate
* $$y$$: the affine y-coordinate
* $$t = xy$$: the product of x and y

### **Problem**

In standard affine form, when computing twisted Edwards additions, the term $$xy$$ appears frequently. Without optimization, it must be recomputed each time.

### **Solution**

By storing $$t = xy$$ explicitly as a third coordinate, we reduce the number of field multiplications per operation. Though this doesn't eliminate field inversions like projective coordinates do, it provides a minimal memory overhead with noticeable performance gain in systems where full projective coordinates are unnecessary or costly.

### Affine $$\Longrightarrow$$ Extended Affine

$$
(x, y) \Longrightarrow (x, y, xy)
$$

### Extended Affine $$\Longrightarrow$$ Affine

Simply discard the third value:

$$
(x, y, t) \Longrightarrow (x, y)
$$

## Extended Projective $$(X : Y : Z : T)$$

To eliminate the need for inverses during addition and doubling, **extended projective coordinates** are widely used. This is the most efficient coordinate system for twisted Edwards curves.

### Point Representation:

$$
x = \frac{X}{Z}, \quad y = \frac{Y}{Z}, \quad T = \frac{XY}{Z}
$$

So, the point is stored as:

$$
(X, Y, Z, T)
$$

The extra coordinate $$T$$ allows for more efficient addition formulas and better reuse of intermediate terms.

### Elliptic Curve Equation:

In this form, the twisted Edwards curve becomes:

$$
(aX^2 + Y^2)Z^2 = Z^4 + dX^2Y^2
$$

However, in practice, point operations are implemented through optimized formulas that directly compute the result without explicitly evaluating this equation.

### Affine $$\Longrightarrow$$ Extended Projective

$$
(x, y) \Longrightarrow (X, Y, Z, T) = (x, y, 1, xy)
$$

### Extended Projective $$\Longrightarrow$$ Affine

$$
(X, Y, Z, T) \Longrightarrow \left( \frac{X}{Z}, \frac{Y}{Z} \right)
$$

### Calculating Operations

Refer to [Hyperelliptic for Extended Projective points](https://www.hyperelliptic.org/EFD/g1p/auto-twisted-extended.html) on the best practices on how to calculate addition and double operations on Projective coordinates.

## Summary of all forms:

| Form                    | Point Form in Affine Form                  | Storage of values | Elliptic Curve Equation             |
| ----------------------- | ------------------------------------------ | ----------------- | ----------------------------------- |
| **Affine**              | $$(x, y)$$                                 | $$(x, y)$$        | $$ax^2 + y^2 = 1 + dx^2y^2$$        |
| **Extended Affine**     | $$(x, y)$$                                 | $$(x, y, t)$$     | $$ax^2 + y^2 = 1 + dx^2y^2$$        |
| **Extended Projective** | $$\left( \frac{X}{Z}, \frac{Y}{Z}\right)$$ | $$(X, Y, Z, T)$$  | $$(aX^2 + Y^2)Z^2 = Z^4 + dX^2Y^2$$ |

## References

* [https://www.hyperelliptic.org/EFD/g1p/auto-twisted.html](https://www.hyperelliptic.org/EFD/g1p/auto-twisted.html)
