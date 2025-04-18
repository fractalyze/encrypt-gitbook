# MSM

Multi-scalar multiplication (MSM) refers to calculating the following summation:

$$
S = \sum_{i=1}^{n}k_iP_i
$$

Or in other words, MSM calculates the **sum of&#x20;**_**groups elements**_ $$P$$ **multiplied by&#x20;**_**scalars**_ $$k$$, where $$n$$ is the **total number of&#x20;**_**scalar**_ $$*$$ _**element**_**&#x20;multiplications**.

In terms of elliptic curve operations, we use MSM to calculate the **sum of point-scalar multiplications**. Since **point addition operations are cheap** based on [the definition of elliptic curves](../), operation on points are best reduced to a series of additions (and/or doubles since they are also additions). The most naive reduction method for this purpose is "[Double-and-add](../scalar-multiplication/double-and-add.md)," though numerous additional approaches have been discovered to further reduce the total number of addition and double operations.

#### Here is a list of current optimizations done on MSM:

1. [Pippenger's Algorithm](pippengers-algorithm.md) (MSM optimization)
   1. Reduces number of additions and doubles in comparison to "[Double-and-add](../scalar-multiplication/double-and-add.md)"
2. [Signed Bucket Index](signed-bucket-index.md) (optimization on [Pippenger's](pippengers-algorithm.md) or other naive bucket methods)
   1. Using [NAF](../../../naf-non-adjacent-form.md), the number of required buckets per window is reduced in half
3. [Batch Inversion for Batch Additions](../batch-inverse-for-batch-point-additions.md) (optimization for batch elliptic curve point additions)&#x20;
   1. When calculating batches of point additions at once, batch inversion can be utilized to reduce the number of required field inverse operations to 1.&#x20;
4. Montgomery Multiplication (optimization on field multiplications)
   1. Efficiently computes modular arithmetic using [montgomery reduction](../../../modular-arithmetic/modular-reduction/montgomery-reduction.md)
5. [GLV Decomposition](../scalar-multiplication/glv-decomposition.md) (optimization for point-scalar multiplication)
   1. GLV is often used to decompose MSM multiplications before running separate MSM's on the smaller parts. GLV generally reduces the total number of doubles operations needed; however, more significantly, if GLV is used before [Signed Bucket Indexes](signed-bucket-index.md), the number of buckets required is reduced to $$\frac{1}{4}$$ of the original number.&#x20;

I also recommend reading [section 1.1](https://tches.iacr.org/plugins/generic/pdfJsViewer/pdf.js/web/viewer.html?file=https%3A%2F%2Ftches.iacr.org%2Findex.php%2FTCHES%2Farticle%2Fdownload%2F10287%2F9736%2F9159#subsection.1.1) of the paper [Speeding Up Multi-Scalar Multiplication over Fixed Points Towards Efficient zkSNARKs](https://tches.iacr.org/index.php/TCHES/article/view/10287/9736). This section provides a very solid overview on lines of research for MSM optimization as of 2023.

> Written by [Ashley Jeong](https://app.gitbook.com/u/Z4DJjtMFfLVXwMaSBRDBJRWVDV12 "mention") of A41
