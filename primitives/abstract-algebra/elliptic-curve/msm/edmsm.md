---
description: ZPrize "Accelerating MSM on Mobile" winner
---

# EdMSM

The two main optimizations contributed by this work are listed as follows:&#x20;

1. Improvement in [CIOS (Coarsely Integrated Operand Scanning)](../../../modular-arithmetic/modular-reduction/montgomery-reduction.md#coarsely-integrated-operands-scanning-cios) modular multiplication which is one method of multi-precision Montgomery multiplication.&#x20;
2. Variant of [Pippenger MSM](edmsm.md#pippengers-algorithm-bucket-method) with optimizations tailored for [twisted Edwards curves](../edwards-curve/).&#x20;

This article will first introduce the optimization trick applicable to CIOS when the MSB of the modulus' most significant word is 0 and not all of the remaining bits are set (with 64-bit machine words, the most significant word of the modulus should be at most 0x7FFFFFFFFFFFFFFE).

## Multiplication Optimization

For an explanation of the CIOS method of multi-precision Montgomery multiplication, refer to the [page ](../../../modular-arithmetic/modular-reduction/montgomery-reduction.md#coarsely-integrated-operands-scanning-cios)written by us.&#x20;

### Normal CIOS

#### Implementation in Python

```python
def cios(a, b, n, n_prime, w, t, l):
    """
    Montgomery Multiplication Step.
    
    Args:
        a (list[int]): \bar{a} = aR
        b (list[int]): \bar{b} = bR
        n (list[int]): modulus
        n_prime (list[int]): n' = -inv(n) mod R
        w (int): bit size of limb
        t (list[int]): REDC(toMont(a) * toMont(b)) = toMont(ab)
        l (int): number of limbs
    
    Returns:
        list[int]: result after multiplication using CIOS
    """
    for i in range(l):
        carry = 0

        # Step 1: t[j] = t[j] + a[j] * b[i] + carry
        for j in range(l):
            total = t[j] + a[j] * b[i] + carry
            # divmod(p, q) divides p by q and returns a tuple (quotient, remainder)
            carry, t[j] = divmod(total, w)
            
        # (1)
        t[l + 1], t[l] = divmod(t[l] + carry, w)

        # Step 2: Compute m
        m = (t[0] * n_prime[0]) % w

        # (carry, _) := t[0] + m * n[0]
        total = t[0] + m * n[0]
        carry, _ = divmod(total, w)

        # (t + m * n) >> w
        for j in range(1, l):
            total = t[j] + m * n[j] + carry
            carry, t[j - 1] = divmod(total, w)

        # (carry, t[l - 1]) := t[l] + carry
        total = t[l] + carry
        carry, t[l - 1] = divmod(total, w)

        # (2) 
        t[l] = t[l + 1] + carry

    return t
```

### Optimization

Now we will show that when the highest word of the modulus $$p$$ is at most $$\frac{(B-1)}{2} - 1$$ (which holds when for 64-bit machine words, the significant word of the modulus is at most 0x7FFFFFFFFFFFFFFE), the two highlighted **additions (1) and (2) can be safely omitted.**&#x20;

#### Proof Sketch

Observe that for the last iterations of two inner loops, the results have the form of $$\mathsf{(carry\_out}, t_{new}) := t_{old} + x_1 \cdot x_2 + \mathsf{carry\_in}$$. We can derive the lemma below with a simple calculation.

**Lemma 1**. If one of $$x_1$$ and $$x_2$$$$x_1$$ is at most $$\frac{(B-1)}{2} - 1$$, then $$\mathsf{carry\_out} \le \frac{(B-1)}{2}$$.&#x20;

From this lemma, we can derive that if the highest word of $$n$$ is at most $$\frac{(B-1)}{2} - 1$$ then the variables $$t[l]$$ and $$t[l+1]$$ always store the value $$0$$ at the beginning of the iteration. This is shown by a proof of induction.

The base case ($$i=0$$) is trivial ($$t$$ is always initialized to $$0$$).&#x20;

At iteration $$i$$, suppose that $$t[l] = t[l+1] = 0$$ and trace the first inner loop execution through the iteration. As $$\bar{a} < n$$ and the highest word of $$n$$ is smaller than $$\frac{(B-1)}{2}$$, Lemma 1 can be used to see that the carry out from the last iteration of the first inner loop is at most $$\frac{(B-1)}{2}$$. Then line (1) sets

$$
t[l] =  \mathsf{carry\_out}\\
t[l+1] = 0
$$

Hence we can remove the addition from line (1).&#x20;

The same reasoning applies for the second inner loop as the highest word of $$n$$ is smaller than $$\frac{(B-1)}{2}$$. As $$t[l] + \mathsf{carry\_out}$$ has the maximum value of $$\frac{(B-1)}{2} + \frac{(B-1)}{2} = (B-1)$$, we can assume that it falls into one limb and the updated $$\mathsf{carry\_out}$$ is $$0$$, and we know that $$t[l+1]$$ is untouched during the loop, which altogether keeps the invariant true going through the line (2) :

$$
t[l+1] = 0 \\
t[l] = t[l+1] + 0 = 0
$$

Hence we can remove the line (2) as well.&#x20;

#### Performance improvement

Experiments show that the new algorithm yields 5-10% improvement over the normal CIOS algorithm.&#x20;

|                | CIOS              | CIOS + EdMSM | SOS               | SOS + Ingonyama    |
| -------------- | ----------------- | ------------ | ----------------- | ------------------ |
| Addition       | $$4l^2 + 4l + 2$$ | $$4l^2 - l$$ | $$4l^2 + 4l + 2$$ | $$4l^2 + 4l + 2$$  |
| Multiplication | $$2l^2 +l$$       | $$2l^2 +l$$  | $$2l^2 +l$$       | $$2l^2 +1$$        |
| Memory space   | $$l+3$$           | $$l+ 3$$     | $$2l+ 2$$         | $$2l+ 2$$          |

SOS saves $$l-1$$ multiplications whereas CIOS saves $$5l + 2$$ additions and $$l-1$$ memory space.&#x20;

## Pippenger optimization

### 2-chain and 2-cycle of elliptic curves

Refer to the [page](../2-chain-and-2-cycle-of-elliptic-curves.md) for the explanation of 2-chain and 2-cycle of elliptic curves.&#x20;

The key property of such **2-chains** of elliptic curves that we are going to exploit, in particular for those whose inner curve is a BLS curve, is shown as a lemma below:

#### Lemma 2.

All inner BLS curves admit a Short Weierstrass form of : $$y^2 = x^3 + 1$$

and they always admit a [twisted Edwards form](../edwards-curve/twisted-edwards-short-weierstrass-transformation.md) : $$ay^2 + x^2 = 1 + dx^2y^2$$ (with $$a = 2\sqrt{3} - 3$$ and $$d = - 2\sqrt{3} - 3$$ over $$\mathbb{F}_p$$).&#x20;

Refer to the [Section 3.1 of the paper](https://eprint.iacr.org/2022/1400.pdf#page=6) for the proof. Note that this implies both curves in a **2-cycle** admit a twisted Edwards form following the same reasoning.&#x20;

### Pippenger's Algorithm (Bucket method)

Refer to the [page](pippengers-algorithm.md) written by us for the full background on Pippenger's.

Let $$n$$ be the size of MSM, $$\lambda$$ be the maximum bit length of scalars, $$s$$ be the window size, $$P_i$$ be the bases points ($$i = [n]$$), $$W_i$$ be the window sum ($$i = [\lambda / s]$$), and $$B_i$$ be the bucket sums ($$i = [2^{s-1}]$$).

### Known optimizations

Using **parallelism**, we can achieve best improvement when we have $$\lambda/s$$ cores and let each core compute each window sum $$W_i$$ in Bucket Reduction step. But this increases memory usage as each core needs to keep $$2^s - 1$$ buckets.&#x20;

**Precomputation** also may improve the performance when the bases $$P_1, \ ..., \ P_n$$ are known in advance. For example, for each base $$P_i$$ we can choose $$k$$ as big as the storage allows and precompute $$k$$ points $$[2^s - k]P, \ ..., \ [2^s-1]P$$ so that can skip first $$k$$ buckets. However large MSM instances already utilize their memory to extreme. Hence the precomputation approach yields negligible improvement in our case.&#x20;

The number of buckets can be reduced by adopting the [NAF approach](signed-bucket-index.md). This method decreases the bucket number by half, hence giving the final total cost $$\approx \frac{\lambda}{s}(n+2^{s-1})$$ group operations.&#x20;

[GLV decomposition](../scalar-multiplication/glv-decomposition.md) is an additional optimization that can be applied only when the curves of interest also have the endomorphism property.&#x20;

### Curve forms and coordinate systems

To minimize the overall cost of storage but also run time, one can store the bases $$P_i$$ in affine coordinates (in tuples $$(x_i, y_i)$$).&#x20;

As Bucket Accumulation is the only step whose cost includes a factor $$n$$, for large MSM instances the dominating cost is in **summing up each bucket**. Note that the additions here are **mixed addition**, which refers to the addition between a affine point ($$P_i$$) and a projective point (partial sum of $$B_i$$), which is faster than normal projective-projective addition.&#x20;

Over the short Weierstrass (SW) form, we commonly choose extended Jacobian coordinates $$\{X, Y, Z^2, Z^3\}$$ where $$x = \frac{X}{Z^2}$$ and $$y = \frac{Y} {Z^3}$$, trading-off memory for run time compared to the usual Jacobian coordinates.&#x20;

However, it appears that a twisted Edwards (tEd) form is appealing for the bucket method since it has the lowest cost for the mixed addition in extended coordinates. Furthermore, the arithmetic on this form is **complete**, i.e. it eliminates the need of branching in case of adding or doubling compared to a SW form. **We showed in Lemma 2 that all inner BLS curves admit a tEd form.**

<figure><img src="../../../../.gitbook/assets/image (141).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (142).png" alt=""><figcaption></figcaption></figure>

Dedicated addition can be applied only when whether the operands in the addition is the same (doubling) or not (addition) is precisely known in advance.&#x20;

We take the example of BLS12-377 for which $$a = -1$$ :

* For Window Reduction step, they uses unified additions (9m) and dedicated doublings (4m + 4s).
* For Bucket Reduction step, they used unified additions (9m) to keep track of the running sum and unified mixed additions (8m) to keep track of the total sum.
* For Bucket Accumulation step, they used unified mixed additions with some precomputations. Instead of storing $$P_i$$ in affine coordinates they store them in a custom coordinates system $$(X, Y, T)$$ where $$y − x = X, y + x = Y$$ and $$2d' \cdot x \cdot y = T$$.

The conversion of all the $$P_i$$ points given on a SW curve with affine coordinates to points on a tEd curve with the custom coordinates $$(X, Y, T )$$ is a **one-time computation dominated by a single inverse** using the Montgomery batch trick.

#### Performance improvement

The implementation of EdMSM shows that an MSM instance of size $$2^{16}$$ on the BLS12-377 curve is **30% faster** when $$P_i$$ points are given on a tEd curve with the custom coordinates compared to the extended-Jacobian-based version which takes points in affine coordinates on a SW curve.&#x20;

<figure><img src="../../../../.gitbook/assets/image (143).png" alt=""><figcaption></figcaption></figure>

***

## References

* [Analyzing and Comparing Montgomery Multiplication Algorithms](https://www.microsoft.com/en-us/research/wp-content/uploads/1996/01/j37acmon.pdf)
* [EdMSM: Multi-Scalar-Multiplication for SNARKs and Faster Montgomery multiplication](https://eprint.iacr.org/2022/1400.pdf)
* [https://www.hyperelliptic.org/EFD/g1p/auto-shortw-xyzz.html](https://www.hyperelliptic.org/EFD/g1p/auto-shortw-xyzz.html)

> Written by [Carson Lee](https://app.gitbook.com/u/Hm5RHrPlu2fbxwXnpUgxmvXVTwB3 "mention") from [A41](https://www.a41.io/)
