# Coordinate Forms

## Affine $$(x,y)$$

Affine form refers to the regular $$(x,y)$$ form of coordinates on an $$xy$$-plane.

### **Problem**

For elliptic curves, point addition is done very often as it is considered cheap. To [calculate point addition in affine form](./#calculating-point-additions), we must compute the slope $$m = \frac{y_Q-y_P}{x_Q-x_P}$$. However, calculating field inverses or the $$(x_Q-x_P)^{-1}$$ in this case is expensive, so we want to prevent the calculation of inverses as much as possible.

### **Solution**

When calculating a **series of point operations**, we convert points to a different intermediate form that does not require field inverses for point operations and for the final result, we revert back to affine. Calculating through these intermediate forms does not cost any inverse operations; however, converting the forms back to affine will **cost one inverse operation**.

The reasoning for why our intermediate forms do not calculate inverses is quite simple: when calculating an operation, the resulting formulas of the final $$x$$ and $$y$$ are manipulated to use the same denominator ([projective](coordinate-forms.md#projective)) or related denominators ([Jacobian](coordinate-forms.md#jacobian) & [XYZZ](coordinate-forms.md#xyzz-extended-jacobian)). This denominator will then be calculated and stored as a continuous separate value $$Z$$.

> We will provide an explanation of projective coordinates to ease our way into understanding Jacobian coordinates, yet in reality for our elliptic curves, we use Jacobian coordinates as it yields much simpler results to work with.

## Projective $$(\frac{X}{Z}, \frac{Y}{Z}) : (X,Y,Z)$$&#x20;

In projective form, affine points of $$(x,y)$$ are redefined as $$(\frac{X}{Z}, \frac{Y}{Z})$$. In practice, projective form is stored as $$(X,Y,Z)$$. With this conversion, the elliptic curve equation is changed to:

$$
Y^2Z =X^3 + aXZ^2 + bZ^3
$$

### Affine $$\Longrightarrow$$ Projective

$$
(x,y) \Longrightarrow (\frac{x}{1},\frac{y}{1}) :(x, y, 1)
$$

### Projective $$\Longrightarrow$$ Affine

$$
(X, Y,Z) \Longrightarrow (\frac{X}{Z},\frac{Y}{Z})
$$

### Calculating Operations

Refer to [Hyperelliptic for Projective points](https://www.hyperelliptic.org/EFD/g1p/auto-shortw-projective.html) on the best practices on how to calculate addition and double operations on projective coordinates.

## Jacobian $$(\frac{X}{Z^2},\frac{Y}{Z^3}) :(X, Y, Z)$$

In Jacobian form, affine points of $$(x,y)$$ are redefined as $$(\frac{X}{Z^2}, \frac{Y}{Z^3})$$. In practice, Jacobian form is stored as $$(X,Y,Z)$$. With this conversion, the elliptic curve equation is changed to:

$$
Y^2 =X^3 + aXZ^4 + bZ^6
$$

### Affine $$\Longrightarrow$$ Jacobian

$$
(x,y) \Longrightarrow (\frac{x}{1^2},\frac{y}{1^3}) :(x, y, 1)
$$

### Jacobian $$\Longrightarrow$$ Affine

$$
(X, Y,Z) \Longrightarrow (\frac{X}{Z^2},\frac{Y}{Z^3})
$$

### Calculating Operations

Refer to [Hyperelliptic for Jacobian points](https://www.hyperelliptic.org/EFD/g1p/auto-shortw-jacobian.html) on the best practices on how to calculate addition and double operations on Jacobian coordinates.

## XYZZ - Extended Jacobian$$(\frac{X}{Z^2},\frac{Y}{Z^3}) : (X,Y,Z^2,Z^3)$$

XYZZ form also known as "Extended Jacobian form" follows the same calculation and reasoning as Jacobian, but with one minor difference: Extended Jacobian is instead stored as $$(x, y, z^2, z^3)$$ to provide better performance of computations with a slight tradeoff in memory.

### Affine $$\Longrightarrow$$ XYZZ

$$
(x,y) \Longrightarrow (\frac{x}{1^2},\frac{y}{1^3}) :(x, y, 1^2, 1^3)
$$

### Jacobian $$\Longrightarrow$$ XYZZ

$$
(X, Y,Z^2, Z^3) \Longrightarrow (\frac{X}{Z^2},\frac{Y}{Z^3})
$$

### Calculating Operations

Refer to [Hyperelliptic for XYZZ points](https://www.hyperelliptic.org/EFD/g1p/auto-shortw-xyzz.html) on the best practices on how to calculate addition and double operations on XYZZ coordinates.

## Summary of all forms:

<table><thead><tr><th width="161.3671875">Form</th><th width="188.3984375">Point Form in Affine Form</th><th width="152.1328125">Storage of values</th><th>Elliptic Curve Equation</th></tr></thead><tbody><tr><td>Affine</td><td><span class="math">(x,y)</span></td><td><span class="math">(x,y)</span></td><td><span class="math">y^2 = x^3 + ax + b</span></td></tr><tr><td>Projective</td><td><span class="math">(\frac{X}{Z}, \frac{Y}{Z})</span></td><td><span class="math">(X,Y,Z)</span></td><td><span class="math">Y^2Z = X^3 + aXZ^2 + Z^3</span></td></tr><tr><td>Jacobian</td><td><span class="math">(\frac{X}{Z^2}, \frac{Y}{Z^3})</span></td><td><span class="math">(X,Y,Z)</span></td><td><span class="math">Y^2 =X^3+aXZ^4+bZ^6</span></td></tr><tr><td>Extended Jacobian (XYZZ)</td><td><span class="math">(\frac{X}{Z^2}, \frac{Y}{Z^3})</span></td><td><span class="math">(X,Y,Z^2, Z^3)</span></td><td><span class="math">Y^2 =X^3+aXZ^4+bZ^6</span></td></tr></tbody></table>

## References

* [https://hackmd.io/@cailynyongyong/HkuoMtz6o#Projective-Coordinates-in-Elliptic-Curves](https://hackmd.io/@cailynyongyong/HkuoMtz6o#Projective-Coordinates-in-Elliptic-Curves)
* [https://www.hyperelliptic.org/EFD/g1p/auto-shortw.html](https://www.hyperelliptic.org/EFD/g1p/auto-shortw.html)

> Written by [Ashley Jeong](https://app.gitbook.com/u/Z4DJjtMFfLVXwMaSBRDBJRWVDV12 "mention") of A41
