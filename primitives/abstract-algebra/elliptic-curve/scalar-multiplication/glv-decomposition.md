---
description: >-
  Splitting a scalar multiplication into 2 and reducing count of double
  operations
---

# GLV Decomposition

> Note 1. The elliptic curve in use should have efficiently-computable [endomorphisms](../../group/morphisms.md#endomorphism-a-homomorphism-where-domain-and-codomain-are-the-same-object)
>
> Note 2. The elliptic curve in use must have the [equation ](../#definition)$$y^2= x^3 + b$$, where $$a=0$$

## Algorithm Process

We attempt to calculate the following elliptic curve point-scalar multiplication:

$$
kP
$$

...where $$k$$ is the scalar and point $$P$$ is $$(x,y)$$.

### 1. Find a scalar $$\lambda$$

...that satisfies the following:

$$
\lambda P = (\beta x , y) = P'
$$

$$\beta$$ will be a cubic root of the base field or in other words for a base field $$\mathbb{F}_p$$ we have $$\beta^3 \equiv 1 \mod{p}$$

As $$\beta$$ is a 3rd root of unity, the elliptic curve equation (with $$a=0$$) holds:

$$
y^2 =(\beta \cdot x)^3 + b =  x^3 + b
$$

TODO: Explain how to find $$\lambda$$

Note that the property of [endomorphism](../../group/morphisms.md#endomorphism-a-homomorphism-where-domain-and-codomain-are-the-same-object) is used for this process.

### 2. Decompose the scalar $$k$$

$$
k = k_1 + \lambda k_2 \mod{p}
$$

...where $$k_1$$ and $$k_2$$ are around half the bits of $$k$$.

### 3. Calculate the total

This means, by definition, the following is true:

$$
kP = k_1P + \lambda k_2 P = k_1P + k_2P'
$$

## Comparison of Cost

In the [GLV paper](https://www.iacr.org/archive/crypto2001/21390189.pdf), they mention their algorithm reduces the number of double operations by around half in comparison to other algorithms due to the decomposed scalar. However, determining the exact cost benefit of GLV proves quite difficult. The authors of GLV state the following in their [analysis section](https://www.iacr.org/archive/crypto2001/21390189.pdf#page=5\&zoom=100,178,550) (which has been paraphrased for simplicity):

Whereas a current "[highly efficient algorithm](https://link.springer.com/content/pdf/10.1007/3-540-49649-1_6.pdf)" that uses exponent recoding and a sliding window algorithm costs **157 point doubles** and **24 point additions** for **160-bit** scalar multiplication, with GLV, it costs around **79 point doubles** and **38 point additions**. If a point double costs **8 field multiplications** and a point addition costs **11 field multiplication**s (per [Jacobian point form](../coordinate-forms.md#jacobian)), then GLV has a run time of around **66%** the "highly efficient algorithm." With an increase in bit length, the comparative run time percentage gets better. For example, **512-bit** scalar multiplication reduces with percentage to **62%.**

By this description, we can conclude that GLV severely decreases the total number of point doubles, but can slightly increase the number of point additions.

## GLV Advantages

### Parallelization

The separation of one scalar multiplication into smaller parts introduces the possibility for running parallelization.&#x20;

### Reduced overhead from shared memory

Memory for intermediate computations can be shared by first calculating $$k_1P$$ and $$k_2P$$. After calculations are finished, $$\beta$$ can be simply multiplied to $$k_2P$$'s result.

```cpp
Point ScalarMultiply(const Point& p, int k) {
    auto [k1, k2] = Point::Split(k); // GLV scalar decomposition

    Point s = p;
    Point p1, p2;

    int max_bits = std::max(
        static_cast<int>(std::bit_width(static_cast<unsigned int>(k1))),
        static_cast<int>(std::bit_width(static_cast<unsigned int>(k2)))
    );
    
    // Share intermediate computations when running Double-and-add on 2 parts
    for (int i = 0; i < max_bits; ++i) {
        if (k1 & (1 << i)) p1 += s;
        if (k2 & (1 << i)) p2 += s;
        s = s.Double();
    }

    return p1 + Point::Endo(p2);
}
```

### MSM

GLV is essential for optimization of [Multi-scalar multiplication (MSM)](../msm/) as it can be used to decompose MSM multiplications before running separate MSM's on the smaller parts. Moreover, when utilizing a bucket optimization technique (like [Pippenger's](../msm/pippengers-algorithm.md)) for MSM, if GLV is used before [Signed Bucket Indexes](../msm/signed-bucket-index.md), the number of buckets required per window is reduced to $$\frac{1}{4}$$ of the original number.&#x20;

## References

* [https://hackmd.io/@drouyang/glv#GLV-Decomposition-for-Multi-Scalar-Multiplication-MSM](https://hackmd.io/@drouyang/glv#GLV-Decomposition-for-Multi-Scalar-Multiplication-MSM)
* [https://www.iacr.org/archive/crypto2001/21390189.pdf](https://www.iacr.org/archive/crypto2001/21390189.pdf)

> Written by [Ashley Jeong](https://app.gitbook.com/u/Z4DJjtMFfLVXwMaSBRDBJRWVDV12 "mention") and [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") of A41
