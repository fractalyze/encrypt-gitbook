---
title: 'Definition 5: The Number Th...'
---

**Definition 5**: The Number Theoretic Transform (NTT) of a vector of polynomial coefficients $$\boldsymbol{a}$$ is defined as $$\hat{\boldsymbol{a}} = \mathsf{NTT}(\boldsymbol{a})$$, where:

$$
\hat{a}_j=\sum^{n-1}_{i=0}\omega^{ij}a_i\mod q \tag{8}
$$

and $$j=0,1,\dots,n-1$$. Here, $$\hat{\bm{a}}$$ denotes the evaluations of the polynomial defined by $$\bm{a}$$ over $$\{\omega^{0},\omega^{1},\dots,\omega^{n-1}\}$$ which are the roots of $$(x^n-1)$$.
