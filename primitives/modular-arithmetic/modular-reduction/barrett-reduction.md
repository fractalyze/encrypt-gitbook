# Barrett Reduction

,The primary goal of Barrett reduction is to perform modular reduction efficiently. The standard calculation, $$x - \lfloor x / n \rfloor \cdot n$$, involves a division ($$x / n$$), which can be slow. Barrett reduction aims to replace this division.

## The Core Idea

Barrett reduction uses multiplications, subtractions, and bit shifts (fast division by powers of 2) instead of a general division. It achieves this using a precomputed value based on the modulus $$n$$ to approximate the quotient value $$\lfloor x / n \rfloor$$.

## How it works

### Setup and Precomputation

Suppose we are working with integers base $$b$$. (Typically $$b=2$$ in computers). Consider an integer $$k$$ such that $$b^k > n$$. Often, $$k$$ is chosen such that $$b^k$$ is roughly $$n^2$$ (e.g., if $$n$$ fits in $$w$$ bits, $$k = 2w$$ is common). Now, we precompute the value $$m$$:&#x20;

$$
m = \lfloor b^k / n \rfloor
$$

which acts as a scaled approximation of division by $$n$$.&#x20;

### Approximating the Quotient

The true remainder is $$r = x - q \cdot n$$, where the true quotient is $$q = \lfloor x / n \rfloor$$. So in Barrett's method we would like to estimate $$q$$ without dividing $$x$$ by $$n$$.

Consider the product $$x \cdot m$$. Substituting the definition of $$m$$,  we get:

$$
x \cdot m = x \cdot \lfloor b^k / n \rfloor\leq x\cdot (b^k/n)
$$

Dividing both sides by $$b^k$$ (which is a right shift by $$k$$ if $$b=2$$):

$$
(x \cdot m) / b^k \le (x \cdot b^k / n) / b^k = x / n
$$

This gives us the approximation to $$q$$:&#x20;

$$
\lfloor(x \cdot m) / b^k\rfloor \le \lfloor x / n \rfloor = q
$$

Let's denote this approximation as $$\hat{q}$$:&#x20;

$$
\hat{q} = \lfloor (x \cdot m) / b^k \rfloor\le q
$$

### How Close is the Approximation?

Notice that we can write $$b^k=\lfloor b^k/n\rfloor \cdot n + r'$$ for some remainder $$r'<n$$. Solving for $$m$$, we have$$m=(b^k-r')/n$$. Replacing $$m$$ for the $$\hat{q}$$ equation, we get:

$$
\hat{q} = \lfloor (x \cdot m) / b^k \rfloor = \left\lfloor x \cdot \frac{(b^k - r')/n}{b^k} \right\rfloor = \lfloor (x/n) - (x \cdot r' / (n \cdot b^k)) \rfloor
$$

Comparing $$\hat{q}$$ to $$q = \lfloor x/n \rfloor$$, the difference depends on the term $$(x \cdot r' / (n \cdot b^k))$$, which is small if $$b^k$$ is large enough relative to $$x$$.

For example, if $$b^k > n^2$$, then $$(x \cdot r' / (n \cdot b^k)) < 1$$ which gives us a nice tight bound $$q-1\leq \hat{q}$$ so we need only 1 subtraction at the worst case. If we choose a smaller $$k$$ such that $$b^k > n$$, $$\hat{q}$$ can get as low as $$q-2$$, but typically we choose a large $$b^k > n^2$$.

## Final Reduction Algorithm

* Precomputation (once for fixed $$n$$):
  * Choose $$k$$ (e.g., $$k = 2 \cdot \mathsf{bitlength}(n)$$). We assume $$b=2$$.
  * Calculate $$m = \lfloor 2^k / n \rfloor$$.
* Reduction (for each $$x$$):
  * Calculate $$\hat{q} = \lfloor (x \cdot m) / 2^k \rfloor$$ (intermediate product $$x \cdot m$$ might need $$k + \mathsf{bitlength}(x/n)$$ bits; often only the higher bits are needed since we do a $$k$$ bit-shift afterwards).
  * Calculate $$\hat{r} = x - \hat{q} \cdot n$$.
  * Correction: while $$\hat{r} \geq n$$, $$\hat{r}=\hat{r}-n$$

### Cost Analysis of Modular Multiplication

#### Single-Precision case

Suppose the modulus $$n$$ has bit width of $$w< \mathsf{word\_size}$$. Then, the cost looks like:

1. Computing $$x=a\cdot  b:$$ multiplication of bitwidth $$w\times w\rightarrow 2w$$ .&#x20;
2. Computing $$x\cdot m:$$ multiplication of bitwidth $$2w\times (k-w)\rightarrow w+k$$.
3. Computing $$\hat{q}\cdot n:$$ multiplication of bitwidth $$w\times w\rightarrow 2w$$.

TODO([Batzorig Zorigoo](https://app.gitbook.com/u/cO1lUla01ZW0seepO37jMHFTxg42 "mention")): add multi-precision case

> Written by [Batzorig Zorigoo](https://app.gitbook.com/u/cO1lUla01ZW0seepO37jMHFTxg42 "mention") of Fractalyze
