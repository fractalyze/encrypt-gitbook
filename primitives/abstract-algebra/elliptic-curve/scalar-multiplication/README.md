# Scalar Multiplication

A point-scalar multiplication for an elliptic curve is defined as follows:

$$
kP = k*(x,y)
$$

Here, $$k$$ is a scalar or base field element of an elliptic curve and $$P$$ or $$(x,y)$$ is a point on an elliptic curve.&#x20;

Optimizing point-scalar multiplication for elliptic curve points is of significant interest. As point addition proves very cheap, we must reduce point multiplications into a series of point additions instead. The simplest and most naive method to achieve this is the [Double-and-add](double-and-add.md) method.&#x20;

Perhaps the more significant problem to solve is the [MSM problem](../msm/) where the question of how to most efficiently optimize scalar multiplication is generalized to how to most efficiently optimize the sum of multiple scalar multiplications.&#x20;
