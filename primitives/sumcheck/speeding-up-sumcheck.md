---
description: 'Presentation: https://www.youtube.com/watch?v=YXANnCp5118'
---

# Speeding Up Sumcheck

The sum-check protocol is a fundamental tool in the design of modern succinct interactive proofs. It allows us to verify claims of the form

$$
\sum_{\bm{x}\in\{0,1\}^\ell}g(\bm{x})=C_0
$$

for a $$\ell$$-variate polynomial $$g$$ of degree at most $$d$$ in each variable by checking the evaluation of $$g$$ at a random point with a single oracle query at random challenge $$\bm{r}\in\mathbb{F}^\ell$$:

$$
g(\bm{r}) = v
$$

### Recap of the Protocol

**Setup**: Assume that the verifier has oracle access to the polynomial $$g$$, and aims to verify:

$$
\sum_{\bm{x}\in\{0,1\}^\ell}g(\bm{x})=C_0
$$

**For each round** $$i = 1, \dots,\ell$$**:**

1. $$\mathcal{P}$$ sends the univariate polynomial $$s_i(X)$$ claimed to equal:

$$
\sum_{(x_{i+1},...,x_{\ell})\in\{0,1\}^{\ell−i}} g(r_1,\dots, r_{i−1}, X, x_{i+1}, . . . , x_\ell)
$$

&#x20;    $$\mathcal{P}$$ does so by sending the evaluations $$\{s_i(u) : u \in \widehat{U}_d\}$$ to $$\mathcal{V}$$.&#x20;

{% hint style="info" %}
Here, the original protocol typically evaluates from some evaluation domain $$U_d$$ but we can take $$\widehat{U}_d=U_d/\{1\}$$ and since verifier can calculate $$s_i(1)$$ using other evaluations.
{% endhint %}

2. $$\mathcal{V}$$ sends a random challenge $$r_i \leftarrow \mathbb{F}$$ to $$\mathcal{P}$$.
3. $$\mathcal{V}$$ derives $$s_i(1) := C_{i−1} − s_i(0)$$, then sets $$C_i := s_i(r_i)$$. After $$\ell$$ rounds, we reduce to the claim that $$g(r_1,\dots,r_{\ell}) = C_\ell$$. $$\mathcal{V}$$ checks this claim by making a single oracle query to $$g$$.

Typically, the initial polynomial involve "small" values (e.g., from a base field $$\mathbb{F}_p$$), but for security, the protocol's challenges are drawn from a much larger possible values (e.g., from a extension field $$\mathbb{F}_{p^k}$$). Below, I will denote this challenge field as $$\mathbb{F}_c$$. This creates a significant performance gap between different types of multiplication:

* $$\mathfrak{ss}$$(small-small): Base field by base field. **Very fast.**
* $$\mathfrak{sl}$$ (small-large): Base field by extension field. **Fast.**
* $$\mathfrak{ll}$$ (large-large): Extension field by extension field. **Very slow.**

Therefore, the one of the optimization idea is reducing the number of $$\mathfrak{ll}$$ multiplications.

### Existing Algorithms

Consider the case where the polynomial $$g(x)$$ is given as

$$
g(X) = p_1(X) \times p_2(X) \times \cdots \times p_d(X)=\prod^d_{k=1}p_k(X) \tag{9}
$$

where $$p_1, . . . , p_d$$ are $$d$$ multilinear polynomials in $$\ell$$ variables such that $$p_i(\bm{x})$$ has small values for all $$i \in \{1, . . . , d\}$$ and $$\bm{x} \in \{0, 1\}^\ell$$.

#### Algorithm 1:

The prover maintains $$d$$ arrays $$P_1,\dots, P_d$$ which initially store all evaluations of $$p_1, . . . , p_d$$ over $$\{0, 1\}^\ell$$, e.g., we have $$P_k[\bm{x}] = p_k(\bm{x})$$ for all $$k \in [1,d]$$ and $$\bm{x}\in\{0, 1\}^\ell$$. The prover halves the size of the arrays after each round $$i$$, via “binding” the arrays $$\{P_k\}_{k\in[0,d]}$$ to the challenge $$r_i$$ received from the verifier.

{% hint style="info" %}
**Setup:** arrays $$P_1^{(0)},\dots,P_d^{(0)}$$ of size $$2^\ell$$, corresponding to evaluations of each polynomial over the boolean hypercube $$\{0,1\}^\ell$$

**For each round** $$i = 1, \dots,\ell$$**:**

1. For $$u \in \widehat{U}_d$$, compute $$s_i(u)$$ using the formula:

$$
s_i(u) := \sum_{\bm{x}'\in\{0,1\}^{\ell−i}} \prod^d_{k=1}\bigg(\big(P^{(i)}_k[1,\bm{x}'] − P^{(i)}_k [0,\bm{x}']\big)\cdot u + P^{(i)}_k[0, \bm{x}'] \bigg)
$$

2. Send evaluations $$\{s_i(u) : u \in \widehat{U}_d\}$$ to the verifier.
3. Receive challenge $$r_i \in \mathbb{F}_c$$ from the verifier.
4. For $$k = 1, . . . , d$$ and $$\bm{x}' \in \{0, 1\}^{\ell−i}$$, update arrays:

$$
P^{(i+1)}_k[\bm{x}']:=(P^{(i)}_k[1, \bm{x}'] − P^{(i)}_k[0,\bm{x}'])\cdot r_i + P^{(i)}_k[0,\bm{x}']
$$
{% endhint %}

{% hint style="warning" %}
**Cost Analysis:**&#x20;

1. In round 1, since the initial evaluations $${p_k(x) : k \in [0,d], \bm{x} \in \{0, 1\}^{\ell}}$$ are small, all multiplications are $$\mathfrak{ss}$$. Since we need to compute $$d$$ evaluations $${s_1(u) : u \in \widehat{U}_d}$$, and each evaluation is a sum over $$2^{\ell−1}$$ products of $$d$$ small values, the total cost is $$d · (d − 1) · 2^{\ell-1}$$ $$\mathfrak{ss}$$ multiplications. We then take $$d · 2 ^{\ell−1}$$ $$\mathfrak{sl}$$ multiplications for updating the arrays.
2. From round $$i \geq 2$$, all array values are large,  so we incur $$d · (d − 1) · 2^{\ell −i}$$ $$\mathfrak{ll}$$ multiplications for computing all evaluations, and $$d · 2^{\ell−i}$$ $$\mathfrak{ll}$$ multiplications for updating the arrays.
{% endhint %}

**Lemma**. When the evaluations of $$p_1, . . . , p_d$$ are small, Algorithm 1’s dominant cost is $$d\cdot 2\cdot 2^{\ell -1}\mathfrak{ll}$$\
multiplications, followed by $$d · 2^{ℓ−1}\mathfrak{sl}$$ multiplications, $$d(d − 1) · 2^{\ell-1} \mathfrak{ss}$$, and $$O(n)$$ field additions.

#### Algorithm 2: Quasilinear Time and Square-Root Space

The main idea is to avoid updating the arrays $$P_1,\dots, P_d$$ in each round $$i  \leq \ell/2$$, and instead compute\
$$\prod^d_{k=1} p_k(r_1,\dots, r_{i−1}, u, x')$$ directly using the following lemma.

**Lemma**. Let $$p: \mathbb{F}^\ell \rightarrow \mathbb{F}$$ be a multilinear polynomial. Then for any $$0 \leq i \leq \ell$$, any $$(r_1, . . . , r_i) \in \mathbb{F}_c^i$$,\
and any $$x' ∈ \{0, 1\}^{\ell-i}$$,

$$
p(\bm{r}_{[1,i]}, \bm{x}'
) = \sum_{\bm{y}\in\{0,1\}^i}
\widetilde{\sf{eq}}(\bm{r}_{[1,i]}, \bm{y})\cdot p(\bm{y}, \bm{x}')
$$

With this formula, we can calculate $$s_i(u)$$ as follows:

$$
\begin{aligned}
s_i(u)&=\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\prod^d_{k=1} p_k(\bm{r}_{[1,i-1]},u,\bm{x}')\\
&=\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\prod_{k=1}^d\sum_{\bm{y}\in\{0,1\}^i}\widetilde{\sf{eq}}((\bm{r}_{[1,i-1]},u), \bm{y})\cdot p_k(\bm{y}, \bm{x}')\\
&=\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\prod_{k=1}^d\sum_{\bm{y}'\in\{0,1\}^{i-1}}\widetilde{\sf{eq}}((\bm{r}_{[1,i-1]},u), (\bm{y}',u))\cdot p_k(\bm{y}',u, \bm{x}')\\
&=\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\prod_{k=1}^d\sum_{\bm{y}'\in\{0,1\}^{i-1}}\widetilde{\sf{eq}}(\bm{r}_{[1,i-1]}, \bm{y}')\cdot p(\bm{y}',u, \bm{x}')\\
\end{aligned}
$$

After $$\ell/2$$ rounds, as both the time and space needed for the $$\widetilde{\sf{eq}}$$ evaluations goes past $$2^{\ell/2}$$, we will switch to Algorithm 1, which incurs a negligible cost of $$d^2\cdot2^{ℓ/2}$$ $$\mathfrak{ll}$$. Note that in practice, one could switch as soon as the prover is no longer space-constrained, which may happen well before round $$\ell/2$$.

{% hint style="info" %}
**For each round** $$i=1,\dots,\ell/2$$:

1. For $$u\in\widehat{U}_d$$, compute $$s_i(u)$$ using the formula:

$$
s_i(u)=\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\prod_{k=1}^d\sum_{\bm{y}'\in\{0,1\}^{i-1}}\widetilde{\sf{eq}}(\bm{r}_{[1,i-1]}, \bm{y}')\cdot p_k(\bm{y}',u, \bm{x}') \tag{10}
$$

2. Send evaluations $$\{s_i(u) : u \in \widehat{U}_d\}$$ to the verifier.
3. Receive challenge $$r_i \in \mathbb{F}_c$$ from the verifier.

**During round** $$i=\ell/2$$:&#x20;

1. Store the evaluations of $$p_k(\bm{r}_{[1,\ell/2]},b,\bm{x}')$$ for all $$k=1,\dots,d$$, $$b\in\{0,1\}$$ and $$\bm{x}'\in\{0,1\}^{\ell/2}$$, obtained while computing round $$\ell/2$$.

**For the leftover rounds:**

1. Follow algorithm 1.
{% endhint %}

{% hint style="warning" %}
**Cost Analysis**:

1. Inner sum has $$2^{i-1}$$ $$\mathfrak{sl}$$ multiplications between the $$\widetilde{\sf{eq}}$$ terms and the evaluations $$p_k(\bm{y}', u, \bm{x}')$$.
2. $$(d − 1)$$ $$\mathfrak{ll}$$ multiplications for the product $$\prod^d_{k=1} p_k(\bm{r}_{[1,i−1]}, u, \bm{x}')$$.
3. Outer summation then gives us, $$2^{l-i}\cdot(2^{i-1}\mathfrak{sl} + (d-1)\mathfrak{ll})=2^{l-1}\mathfrak{sl}+2^{l-i}(d-1)\mathfrak{ll}$$ multiplications for each $$s_i(u)$$ evaluation and we need to evaluate over $$d$$ points.
4. &#x20;Besides these, we have lower-order costs of $$d(d − 1)· 2^{\ell−1}$$ $$\mathfrak{ss}$$ multiplications to compute the evaluations $$\{p_k(\bm{y}', u, \bm{x}')\}_{k,\bm{y}',u,\bm{x}'}$$ and $$2^{i−1}$$$$\mathfrak{ll}$$ multiplications to compute the evaluations $$\big\{\widetilde{\sf{eq}}(\bm{r}_{[0,i-1]} , \bm{y}')\big\}_{\bm{y}'\in\{0,1\}^{i−1}}$$ and store them in an array.
{% endhint %}

**Lemma**. When the evaluations of $$p_1,\dots, p_d$$ are small, Algorithm 2’s cost is dominated by $$d · \ell_0 \cdot 2^{\ell-1} \mathfrak{sl}$$ multiplications (spread over the first l/2 rounds), and $$d\cdot(d − 1) · 2^\ell \mathfrak{ll}$$ multiplications. The lower-order costs include $$d\cdot \ell_0\cdot(d − 1)\cdot2^{l−1} \mathfrak{ss}$$ multiplications and $$O(n)$$ field additions.

#### Algorithm 3: Warm-up Attempt

In the Algorithm 1, the prover performs exclusively $$\mathfrak{ll}$$ multiplications starting from round 2, due to the need to bind the multilinear polynomials after round 1. If we want to reduce the number of $$\mathfrak{ll}$$ multiplications, we must delay this binding process. A natural attempt at this leads to Algorithm 3, which is typically faster than Algorithm 1 in the first few rounds, and uses significantly less memory.

If we rearrange the Equation (10), we can get:

$$
\begin{aligned}
s_i(u)&=\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\prod_{k=1}^d\sum_{\bm{y}'\in\{0,1\}^{i-1}}\widetilde{\sf{eq}}(\bm{r}_{[1,i-1]}, \bm{y}')\cdot p_k(\bm{y}',u, \bm{x}')\\
&=\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\sum_{\bm{y}'_1,\dots,\bm{y}'_d\in\{0,1\}^{i-1}}\prod_{k=1}^d\widetilde{\sf{eq}}(\bm{r}_{[1,i-1]}, \bm{y}'_k)\cdot p_k(\bm{y}'_k,u, \bm{x}')\\
&=\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\sum_{\bm{y}'_1,\dots,\bm{y}'_d\in\{0,1\}^{i-1}}\prod_{k=1}^d\widetilde{\sf{eq}}(\bm{r}_{[1,i-1]}, \bm{y}'_k)\cdot \prod_{k=1}^dp_k(\bm{y}'_k,u, \bm{x}')\\
&=\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\sum_{\bm{y}'_1,\dots,\bm{y}'_d\in\{0,1\}^{i-1}}\prod_{k=1}^d\bigg(\prod_{j=1}^{i-1}(1-r_j)^{\bm{y}'_k[j]}\cdot r_j^{1-\bm{y}'_k[j]} \bigg)\cdot\prod_{k=1}^dp_k(\bm{y}'_k,u, \bm{x}')\\
&=\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\sum_{\bm{y}'_1,\dots,\bm{y}'_d\in\{0,1\}^{i-1}}\prod_{j=1}^{i-1}\bigg((1-r_j)^{\sum\bm{y}'_k[j]}\cdot r_j^{d-\sum\bm{y}'_k[j]}\bigg)\cdot\prod_{k=1}^dp_k(\bm{y}'_k,u, \bm{x}')\\
&=\sum_{\bm{y}'_1,\dots,\bm{y}'_d\in\{0,1\}^{i-1}}\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\prod_{j=1}^{i-1}\bigg((1-r_j)^{\sum\bm{y}'_k[j]}\cdot r_j^{d-\sum\bm{y}'_k[j]}\bigg)\cdot\prod_{k=1}^dp_k(\bm{y}'_k,u, \bm{x}')\\
&=\sum_{\bm{y}'_1,\dots,\bm{y}'_d\in\{0,1\}^{i-1}}\prod_{j=1}^{i-1}\bigg((1-r_j)^{\sum\bm{y}'_k[j]}\cdot r_j^{d-\sum\bm{y}'_k[j]}\bigg)\cdot\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\prod_{k=1}^dp_k(\bm{y}'_k,u, \bm{x}')\\
\end{aligned}
$$

1. Exchanging product with the inner sum
2. Distribute $$\prod$$
3. Expand eq
4. Switch inner $$\prod$$ with outer $$\prod$$
5. Switch the order of outer sums
6. $$\widetilde{\mathsf{eq}}$$ part is invariant over $$\bm{x}'$$ so we can factor it out.

At this point, we have rewritten the computation of $$s_i(u)$$ as an inner product between two vectors of length $$2^{d(i−1)}$$ (indexed by $$\bm{y}'_1, . . . , \bm{y}'_d$$): one dependent on the challenges $$r_1, . . . , r_{i−1}$$, and the other dependent on the multilinear polynomials $$p_1, . . . , p_d$$.

We can do slightly better by noticing that the challenge-dependent terms only depend on $$\bm{v} \in \{\sum_{k=1}^d \bm{y}'_k[j]\}_{j\in[1,i−1]}$$ and not on the individual $$\{\bm{y}'_k\}_{k\in[1,d]}$$—there are only $$(d + 1)^{i−1}$$ distinct such terms. By re-indexing based on $$\bm{v} \in [0, d]^{i−1}$$, we can further rewrite:

$$
\begin{aligned}
s_i(u)&=\sum_{\bm{y}'_1,\dots,\bm{y}'_d\in\{0,1\}^{i-1}}\prod_{j=1}^{i-1}\bigg((1-r_j)^{\sum\bm{y}'_k[j]}\cdot r_j^{d-\sum\bm{y}'_k[j]}\bigg)\cdot\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\prod_{k=1}^dp_k(\bm{y}'_k,u, \bm{x}')\\
&=\sum_{\bm{v}\in\{0,1\dots,d\}^{i-1}}\prod_{j=1}^{i-1}\bigg((1-r_j)^{\bm{v}_j}\cdot r_j^{d-\bm{v}_j}\bigg)\cdot\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\sum_{\bm{y}'_1,\dots,\bm{y}'_d\in\{0,1\}^{i-1}; \bm{v}=\sum \bm{y}_k}\prod_{k=1}^dp_k(\bm{y}'_k,u, \bm{x}')
\end{aligned}
$$

Notice that above sum over the hypercube $$\bm{v}\in\{0,\dots,d\}^{i-1}$$ can be thought of as a inner product of a two vector of size $$(d+1)^{i-1}$$:&#x20;

$$
\Braket{\bigotimes_{j=1}^{i-1}\bigg((1-r_j)^{\bm{v}_j}\cdot r_j^{d-\bm{v}_j}\bigg)_{v_j=0}^d, \bigg(\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\sum_{\substack{\bm{y}'_1,\dots,\bm{y}'_d\in\{0,1\}^{i-1}\\ \bm{v}=\sum \bm{y}_k}}\prod_{k=1}^dp_k(\bm{y}'_k,u, \bm{x}')\bigg)_{\bm{v}\in[0,d]^{i−1}}}
$$

{% hint style="info" %}
$$(1-r_j)^{\bm{v}_j}\cdot r_j^{d-\bm{v}_j}$$: this can have $$d$$ different values depending on the value of $$v_j$$ which can range from $$0$$ to $$d$$. If we store them in a vector, we can denote as:

$$
\bigg((1-r_j)^{\bm{v}_j}\cdot r_j^{d-\bm{v}_j}\bigg)_{v_j=0}^d
$$

The outer sum and product is then taking product of all possible $$i-1$$ combinations of entries within this vector and summing them. These product combinations can be thought of as [Kronecker product](https://en.wikipedia.org/wiki/Kronecker_product):

$$
\bigotimes_{j=1}^{i-1}\bigg((1-r_j)^{\bm{v}_j}\cdot r_j^{d-\bm{v}_j}\bigg)_{v_j=0}^d
$$
{% endhint %}

{% hint style="info" %}
Let us denote the right hand side of the inner product as:

$$
\mathsf{A}_i(\bm{v},u)=\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\sum_{\substack{\bm{y}'_1,\dots,\bm{y}'_d\in\{0,1\}^{i-1}\\ \bm{v}=\sum \bm{y}_k}}\prod_{k=1}^dp_k(\bm{y}'_k,u, \bm{x}')
$$

Then we have:

$$
s_i(u)=\Braket{\bigotimes_{j=1}^{i-1}\bigg((1-r_j)^{\bm{v}_j}\cdot r_j^{d-\bm{v}_j}\bigg)_{v_j=0}^d, \bigg(\mathsf{A}_i(\bm{v},u)\bigg)_{\bm{v}\in[0,d]^{i−1}}}
$$
{% endhint %}

Since the accumulators $$\mathsf{A}_i(\bm{v}, u)$$ do not depend on the challenges $$r_1, . . . , r_{i−1}$$, the prover can precompute them before the protocol begins. For each of these, we don't have to recompute $$\{p_k(\bm{y}'_k, u, \bm{x}')\}_{k\in[1,d]}$$ and instead reuse these across them. For this, we can do multilinear extension on $$p_k$$ over variable $$u$$ to get:

$$
\begin{aligned}
\mathsf{A}_i(\bm{v},u)&=\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\sum_{\bm{y}'_1,\dots,\bm{y}'_d\in\{0,1\}^{i-1}; \bm{v}=\sum \bm{y}_k}\prod_{k=1}^dp_k(\bm{y}'_k,u, \bm{x}')\\
&=\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\sum_{\bm{y}'_1,\dots,\bm{y}'_d\in\{0,1\}^{i-1}; \bm{v}=\sum \bm{y}_k}\prod_{k=1}^d\sum_{y^*_k\in\{0,1\}}u^{y^*_k}(1-u)^{1-y^*_k}p_k(\bm{y}'_k,y^*_k, \bm{x}')\\
&=\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\sum_{\bm{y}'_1,\dots,\bm{y}'_d\in\{0,1\}^{i-1}; \bm{v}=\sum \bm{y}_k}\sum_{\{y^*_1, \dots, y^*_d\}\in\{0,1\}}u^{\sum y^*_j}(u-1)^{d-\sum y^*_j}\cdot\prod_{k=1}^dp_k(\bm{y}'_k,y^*_k, \bm{x}')
\end{aligned}
$$

and now we only need to evaluations of $$p_k$$ over the boolean hypercube.

**Cost Analysis.** Algorithm 3 uses $$O((d + 1)^{\ell_0} )$$ space for rounds $$1$$ to $$\ell_0$$ (accumulators and challenge vectors) and $$d · 2^{\ell−\ell_0}$$ space thereafter (cached results). Since $$(d + 1)^{\ell_0} \ll d · 2^{\ell−\ell_0}$$ in practice, the overall space complexity is dominated by the latter term, representing a space saving factor of roughly $$2^{\ell_0}$$ compared to Algorithm 1.&#x20;

Regarding runtime, Algorithm 3 is bottlenecked by the $$(d − 1) · 2^{d\ell_0} · 2^{\ell−\ell_0} \mathfrak{ss}$$ multiplications needed to compute the products $$\{\prod_{k=1}^d p_k(\bm{y}'_k, \bm{x}')\}_{\bm{y},\bm{x}'}$$. We also incur roughly $$d · 2^\ell \mathfrak{sl}$$ mults when using Algorithm 2 for the $$(\ell_0 + 1)$$-th round, and roughly $$d^2 · 2^{\ell−\ell_0−1}\mathfrak{ll}$$ mults for the remaining rounds using Algorithm 1.

#### Algorithm 4: Optimizing with Toom-Cook Multiplication

While Algorithm 3 almost eliminates $$\mathfrak{ll}$$ multiplications for the first few rounds and slashes memory usage, we still end up performing a lot of $$\mathfrak{ss}$$ multiplications (namely, by a factor of $$\approx 2^{(d−1)·\ell_0}$$ compared to the number of $$\mathfrak{ll}$$ mults in Algorithm 1). To reduce this cost, we need another way to rewrite the products $$\prod_{k=1}^d p_k(\bm{r}_{[1,i-1]},u, \bm{x}')$$.

The idea is to see the product as the evaluation of the polynomial:

$$
F(Y_1, . . . , Y_{i−1}) = \prod_{k=1}^d p_k(Y_1, . . . , Y_{i−1}, u, \bm{x}')
$$

at the challenge point $$\bm{r}_{[1,i-1]}$$.

Since $$F$$ has individual degree $$d$$, we can apply Lagrange interpolation to each variable to rewrite $$F(Y_1,\dots, Y_{i−1})$$ as

$$
\sum_{v_1\in U_d}\mathcal{L}_{U_d,v_1}(Y_1)\dots\sum_{v_{i-1}\in U_d}\mathcal{L}_{U_d,v_{i-1}}(Y_{i-1})\prod_{k=1}^dp_k(\bm{v},u,\bm{x}')\tag{14}
$$

As a result, when we plug in the evaluation $$(r_1, \dots , r_{i−1})$$, we can rewrite the $$i$$-th sum-check polynomial $$s_i(X)$$ as:

$$
\begin{aligned}
s_i(u)&=\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\sum_{\bm{v}\in U_d}\prod_{j=1}^{i-1}\mathcal{L}_{U_d,\bm{v}_j}(Y_{i-1})\prod_{k=1}^dp_k(\bm{v},u, \bm{x}')\\
&=\sum_{\bm{v}\in U_{d}^{i-1}}\prod_{j=1}^{i-1}\mathcal{L}_{U_d,\bm{v}_j}(r_j)\cdot\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\prod_{k=1}^d p_k(\bm{v},u,\bm{x}')
\end{aligned}
$$

then we have:

$$
s_i(u)=\Braket{\bigotimes_{j=1}^{i-1}\big(\mathcal{L}_{U_d,v_j}(r_j)\big)_{v_j\in U_d}, \bigg(\mathsf{A}_i(\bm{v},u)\bigg)_{\bm{v}\in U_d^{i−1}}}
$$

Just like with Algorithm 3, we can precompute the accumulators $$\mathsf{A}_i(\bm{v}, u)$$ by computing each of the $$\approx (d + 1)^{\ell_0}$$ product

$$
\bigg\{\prod_{k=1}^d p_k(\bm{v},u,\prod{x}'):\bm{v}\bigg\}
$$

over $$\bm{v}\in U_d^{i-1}$$, $$u \in \widehat{U}_d$$ and $$\bm{x}'\in\{0,1\}^{\ell-i}$$ once, then inserting the result into the appropriate $$\mathsf{A}_i(\bm{v}, u)$$ terms over all $$i\in [1,\ell_0]$$ rounds. For details, see Section A.3.

**Cost Analysis**. Similar to Algorithm 3, Algorithm 4 also saves a factor of $$2^{l_0}$$ in space. Unlike Algorithm 3, however, it incurs only $$(d − 1) · (d + 1)^{\ell_0} · 2^{\ell−\ell_0}\mathfrak{ss}$$, instead of $$(d − 1) · 2^{d\cdot \ell_0} · 2^{\ell−\ell_0}$$. This factor can be roughly 5 times smaller for common parameters (e.g. $$d = 2$$ and $$\ell_0 = 5$$). The cost of $$\mathfrak{sl}$$ and $$\mathfrak{ll}$$ are the same as in Algorithm 3.

### Optimizations for Sum-Check involving EqPoly

In this section, the optimization only applies to sum-check where $$g$$ is of the form:

$$
\sum_{\bm{x}\in\{0,1\}^\ell}g(\bm{x})=\sum_{\bm{x}\in\{0,1\}^\ell}\widetilde{\mathsf{eq}}(\bm{w},\bm{x})\cdot\prod_{k=1}^dp_k(\bm{x})
$$

with $$w \in \mathbb{F}_c^\ell$$ a vector of verifier’s challenges, and $$p_1(X), \dots, p_d(X)$$ are multilinear polynomials. (e.g $$Q_{\bm{io}}$$ zero-check in Spartan).

#### Gruen's Optimization

The main idea of this optimization by Gruen et al. is to reduce the computation of the degree-$$(d + 1)$$ polynomial $$s_i(X)$$ sent by the prover in each round $$i$$, to the computation of a degree-$$d$$ factor $$t_i(X)$$ and a linear factor $$\mathsf{l}_i(X)$$. This way, the linear term is “free” to compute, and the prover saves an evaluation when computing $$t_i(X)$$ instead of $$s_i(X)$$.&#x20;

This is possible due to the special structure of the $$\widetilde{\mathsf{eq}}$$ polynomial, which for every $$i \in [1,\ell]$$ can be decomposed as:

$$
\widetilde{\mathsf{eq}}(\bm{w},(\bm{r}_{[1,i-1]},X,\bm{x}'))=\widetilde{\mathsf{eq}}(\bm{w}_{[1,i-1]},\bm{r}_{[1,i-1]})\cdot\widetilde{\mathsf{eq}}(w_i,X)\cdot\widetilde{\mathsf{eq}}(\bm{w}_{[i+1,\ell]},\bm{x}')
$$

If we adapt the original definition for $$s_i(u)$$ to this case, we get:

$$
s_i(u) := \sum_{x'\in\{0,1\}^{\ell−i}}\widetilde{\mathsf{eq}}(\bm{w},\bm{r}_{[1,i-1]},u,\bm{x}') \prod^d_{k=1}\bigg(\big(P^{(i)}_k[1, x'] − P^{(i)}_k [0,x']\big)\cdot u + P^{(i)}_k[0, x'] \bigg)
$$

Thus, if we set $$\mathsf{l}_i(X):=\widetilde{\mathsf{eq}}(\bm{w}_{[1,i-1]},\bm{r}_{[1,i-1]})\cdot\widetilde{\mathsf{eq}}(w_i,X)$$ and

$$
t_i(X):=\sum_{\bm{x}'\in\{0,1\}^{\ell-i}}\widetilde{\mathsf{eq}}(\bm{w}_{[i+1,\ell]},\bm{x}')\cdot\prod_{k=1}^dp_k(\bm{r}_{[1,i-1]},X,\bm{x}')
$$

then we have $$s_i(X)=\mathsf{l}_i(X)\cdot t_i(X)$$.&#x20;

Using this fact, the sum-check prover now instead computes $$d$$ evaluations of $$t_i(u)$$ at $$u\in\widehat{U}_d$$, then rely on the equality $$s_i(0) + s_i(1)=C_i$$ to compute:

$$
t_i(1):=\mathsf{l}_i(1)^{-1}\cdot(C_i-\mathsf{l}_i(0)\cdot t_i(0))
$$

The prover now has $$d+1$$ evaluations of $$t_i$$, and thus can interpolate $$t_i(X)$$ and compute $$s_i(X)$$. Note that computing $$\{t_i(u):u\in\widehat{U}_d\}$$ can be done generically using either Algorithm 1 or 2; since our focus is on optimizing time rather than space, we assume that Algorithm 1 is used.

#### Algorithm 5

We can further reduce $$2^{\ell-1}\mathfrak{ll}$$ associated with the $$\widetilde{\mathsf{eq}}$$ polynomial. Our insight is that we can further split off the $$\widetilde{\mathsf{eq}}$$ terms for the first $$i<\ell/2$$ rounds, so that:

$$
t_i(u)=\sum_{\bm{x}_R\in\{0,1\}^{\ell/2}}\widetilde{\mathsf{eq}}(\bm{w}_R,\bm{x}_R)\cdot\sum_{\bm{x}_L\in\{0,1\}^{\ell/2 -i}}\widetilde{\mathsf{eq}}(\bm{w}_L,\bm{x}_L)\cdot \tag{17}
$$

$$
\cdot \prod^{d}_{k=1}p_k(\bm{r}_{[1,i-1]},u,\bm{x}_L,\bm{x}_R)\tag{18}
$$

for all $$u\in\widehat{U}_d$$, where $$\bm{w}_R=\bm{w}_{[i+1,\ell/2]}$$ and $$\bm{w}_L=\bm{w}_{[\ell/2 + 1,n]}$$. The sum-check prover now computes all inner sums over $$\bm{x}_L$$, then all outer sums over $$\bm{x}_R$$. For rounds $$i\geq \ell/2$$, we no longer have the inner sum, and proceed similarly to Algorithm 1. Note that we put the left part $$\widetilde{\mathsf{eq}}(\bm{w}_L, \bm{x}_L)$$ in the inner sum, since it leads to better memory locality in practice, assuming the evaluations of $$p_k$$ are streamed in big-endian order (so that $$\bm{x}_L$$ are low-order bits compared to $$\bm{x}_R$$, which means their chunks are contiguous in memory).

{% hint style="info" %}
**Initialize:** Length-$$2^\ell$$ arrays $$P^{(0)}_1,\dots, P^{(0)}_d$$ such that&#x20;

$$
P^{(0)}_k[x] = p_k(x) \text{ for all }k = 1,\dots,d \text{ and }\bm{x}\in\{0, 1\}^l
$$

**Pre-computation:** Use Procedure 3 from the paper to compute the evaluations:

$$
\{\widetilde{\mathsf{eq}}(\bm{w}_{[1,i]},\bm{x}_L), \widetilde{\mathsf{eq}}(\bm{w}_{[\ell/2,\ell/2+i]},\bm{x}_R)\}
$$

for all $$i\in[1,\ell/2],\bm{x}_L\in\{0,1\}^{\ell/2-i}$$, and $$\bm{x}_R\in\{0,1\}^{\ell/2-i}$$.



**For each round** $$i=1,\dots,\ell$$**:**

1. For $$u\in\widehat{U}_d$$, if $$i < \ell/2$$, compute $$t_i(u)$$ using the formula

$$
\sum_{\bm{x}_R\in\{0,1\}^{\ell/2}}\widetilde{\mathsf{eq}}(\bm{w}_R,\bm{x}_R)\cdot \sum_{\bm{x}_L\in\{0,1\}^{\ell/2-i}}\widetilde{\mathsf{eq}}(\bm{w}_L,\bm{x}_L)\cdot\prod_{k=1}^{d}\bigg(\big(P_k^{(i)}[1,\bm{x}_L,\bm{x}_R]-P_k^{(i)}[0,\bm{x}_L,\bm{x}_R]\big)\cdot u + P_k^{(i)}[0,\bm{x}_L,\bm{x}_R]\bigg)
$$

&#x20;      If $$i\geq\ell/2$$, compute $$t_i(u)$$ using the formula:

$$
\sum_{\bm{x}_R\in\{0,1\}^{\ell/2-i}}\widetilde{\mathsf{eq}}(\bm{w}_R,\bm{x}_R)\cdot\prod_{k=1}^{d}\bigg(\big(P_k^{(i)}[1,\bm{x}_R]-P_k^{(i)}[0,\bm{x}_R]\big)\cdot u + P_k^{(i)}[0,\bm{x}_R]\bigg)
$$

2. Use Procedure 8 form the paper to compute $$s_i(X)$$ from $$\big(\{t_i(u)\},\bm{w},\bm{r}_{[1,i-1]}\big)$$
3. Send evaluations of $$s_i$$ over $$\widehat{U}_d$$
4. Receive challenge $$r_i\in\mathbb{F}$$ from the verifier.
5. For $$k \in [1,d]$$ and $$\bm{x}' \in \{0, 1\}^{\ell−i}$$, update arrays:

$$
P^{(i+1)}_k[\bm{x}']:=(P^{(i)}_k[1, \bm{x}'] − P^{(i)}_k[0,\bm{x}'])\cdot r_i + P^{(i)}_k[0,\bm{x}']
$$
{% endhint %}

**Cost Analysis.** Our optimization avoids computing the $$2^{\ell−i}$$-sized table of evaluations $$\{\widetilde{\mathsf{eq}}(\bm{w}_{[i+1,\ell]}, \bm{x}') : x' \in \{0, 1\}^{\ell−i}\}$$ for each round $$i$$, and instead only compute two square-root-sized tables, which costs $$2^{\ell/2}\mathfrak{ll}$$ mults for $$\widetilde{\mathsf{eq}}(\bm{w}_R, \bm{x}_R)$$ and $$2^{\ell/2−i}\mathfrak{ll}$$ mults for $$\widetilde{\mathsf{eq}}(\bm{w}_L, \bm{x}_L)$$. With memoization, these evaluations can be computed for all rounds $$i$$ in $$2^{\ell/2}\mathfrak{ll}$$ multiplications total. The (negligible) downside is that we need to compute an extra $$2^{\ell/2}\mathfrak{ll}$$ multiplications when computing the outer sum over $$\bm{x}_R$$.&#x20;

Using the same process for cost analysis as in Lemma 3.2, we have the following cost breakdown for Algorithm 5.

**Lemma 5.1.** Algorithm 5’s dominant cost is $$d(d + 1)/2 · N\ \mathfrak{ll}$$, reducing from Algorithm 1’s cost in the same setting by roughly $$N\ \mathfrak{ll}$$.

#### Algorithm 6: Combining SVO with Eq-Poly optimization

Our starting point is Equation (17) in the exposition of **Algorithm 5**. We then apply the multivariate Lagrange decomposition formula in Equation (14) to the evaluations $$\prod_{k=1}^d p_k(\bm{r}, u, \bm{x}_L, \bm{x}_R)$$, which gives:

$$
\begin{aligned}
t_i(u)&=\sum_{\bm{x}_R\in\{0,1\}^{\ell/2-i}}\widetilde{\mathsf{eq}}(\bm{w}_R,\bm{x}_R)\cdot\sum_{\bm{x}_L\in\{0,1\}^{\ell/2}}\widetilde{\mathsf{eq}}(\bm{w}_L,\bm{x}_L)\cdot \\ 
&\cdot \sum_{\bm{v}\in U_d^{i-1}}\prod_{j=1}^{i-1}\mathcal{L}_{U_d,\bm{v}_j}(r_j)\cdot\prod_{k=1}^d p_k(\bm{v},u,\bm{x}_L,\bm{x}_R) \\
&=\Braket{\bigotimes_{j=1}^{i-1}\big(\mathcal{L}_{U_d,v}(r_j)\big)_{v\in U_d}, \bigg(\mathsf{A}_i(\bm{v},u)\bigg)_{\bm{v}\in U_d^{i−1}}}
\end{aligned}
$$

where the accumulators $$\mathsf{A}_i(\bm{v}, u)$$ are now of the form:

$$
\mathsf{A}_i(\bm{v}, u)=\sum_{\bm{x}_R\in\{0,1\}^{\ell/2-i}}\widetilde{\mathsf{eq}}(\bm{w}_R,\bm{x}_R)\sum_{\bm{x}_L\in\{0,1\}^{\ell/2}}\widetilde{\mathsf{eq}}(\bm{w}_L,\bm{x}_L)\prod_{k=1}^d p_k(\bm{v},u,\bm{x}_L,\bm{x}_R)
$$

Unlike the small-value only case, here because of the presence of the $$\widetilde{\mathsf{eq}}(\bm{w}, \cdot)$$ terms, we need to use $$\mathfrak{sl}$$ (instead of $$\mathfrak{ss}$$) multiplications in computing the accumulators, and the additions are over the big field as well. We also note that we have swapped the lengths of the vectors $$\bm{x}_R$$ and $$\bm{x}_L$$ in the accumulator formula. This is so that we can compute the same inner sum across the accumulators over all rounds, and we can dispatch these inner sums to different accumulators in different rounds afterwards, with negligible cost. See Section A.4 from the paper for more details.

{% hint style="info" %}
**Pre-computation:** Use Procedure 9 from the paper to compute the accumulators:

$$
\mathsf{A}_i(\bm{v}, u)=\sum_{\bm{x}_R\in\{0,1\}^{\ell/2-i}}\widetilde{\mathsf{eq}}(\bm{w}_R,\bm{x}_R)\sum_{\bm{x}_L\in\{0,1\}^{\ell/2}}\widetilde{\mathsf{eq}}(\bm{w}_L,\bm{x}_L)\prod_{k=1}^d p_k(\bm{v},u,\bm{x}_L,\bm{x}_R)
$$

for all $$i\in[1,\ell_0],\bm{v}\in U_d^{i-1}$$, and $$u\in \widehat{U}_d$$.

**Initialize:** $$\mathsf{R}_1=(1)$$.

**For each round** $$i=1,\dots,\ell_0$$**:**

1. For $$u\in\widehat{U}_d$$, compute $$t_i(u)$$ using the formula

$$
t_i(u):=\sum_{\bm{v}\in [0,d]^{i-1}}\mathsf{R}_i[\mathsf{nat}_{d+1}(\bm{v})]\cdot \mathsf{A}_i(\bm{v},u)
$$

2. Use Procedure 8 form the paper to compute $$s_i(X)$$ from $$\big(\{t_i(u)\},\bm{w},\bm{r}_{[1,i-1]}\big)$$
3. Send evaluations of $$s_i$$ over $$\widehat{U}_d$$
4. Receive challenge $$r_i\in\mathbb{F}_c$$ from the verifier.
5. Compute $$\mathsf{R}_{i+1}=\mathsf{R}_i \otimes(\mathcal{L}_{U_d,k}(r_j))^{d}_{k=0}$$.

**For round** $$\ell_0+1$$: Follow Algorithm 2, storing the evaluations $$p_k(\bm{r}_{[1,\ell_0-1]},b,\bm{x}')$$ for all $$k\in[1,d],b\in\{0,1\}$$, and $$\bm{x}'\in\{0,1\}^{\ell-\ell_0}$$.

**Receive challenge** $$r_{\ell+1}\in\mathbb{F}_c$$ from the verifier, and perform updates to get $$p_k(\bm{r}_{[1,\ell_0]},\bm{x}')$$ for all $$k$$ and $$\bm{x}'$$, similar to Algorithm 1.

**For rounds** $$\ell+2,\dots,\ell$$: Follow Algorithm 5.
{% endhint %}

**Cost Analysis.** Besides the extra subtleties as noted above, the cost estimate of Algorithm 6 is fairly straightforward and follows directly from combining the estimates of Algorithm 4 and Algorithm 5. In particular, the small-value precomputation process is now dominated by $$(d + 1)^{\ell_0} · 2^{\ell−\ell_0}\mathfrak{sl}$$. Next, the streaming round takes roughly $$d · 2^\ell \mathfrak{sl}$$, and the rest of the linear-time sum-check rounds cost roughly $$d^2 · 2^{\ell−\ell_0−1}\mathfrak{ll}$$. We thus summarize as follows.

**Theorem 6.1.** Algorithm 6’s cost is dominated by $$((\frac{d+1}{2})^{\ell_0} + d) N\ \mathfrak{sl}$$ and $$d^2/2^{\ell_0+1}N\ \mathfrak{ll}$$.

### Summary

<table><thead><tr><th width="148.484375"></th><th width="421.6875">Multiplications</th><th>Space Complexity</th></tr></thead><tbody><tr><td>Algorithm 1</td><td><span class="math">d · 2^{ℓ−1}\mathfrak{sl}+d\cdot 2^{\ell}\mathfrak{ll}</span></td><td><span class="math">d · 2^{\ell}</span></td></tr><tr><td>Algorithm 2</td><td><span class="math">d · \ell_0 \cdot 2^{\ell-1} \mathfrak{sl}+d\cdot 2^{\ell}\mathfrak{ll}</span></td><td><span class="math">d · 2^{\ell−\ell_0}</span></td></tr><tr><td>Algorithm 3</td><td><span class="math">(d − 1) · 2^{d\ell_0} · 2^{\ell−\ell_0} \mathfrak{ss} +d · 2^\ell \mathfrak{sl}+d^2 · 2^{\ell−\ell_0−1}\mathfrak{ll}</span></td><td><span class="math">d · 2^{\ell−\ell_0}</span></td></tr><tr><td>Algorithm 4</td><td><span class="math">(d − 1) · (d + 1)^{\ell_0} · 2^{\ell−\ell_0}\mathfrak{ss}+d · 2^\ell \mathfrak{sl}+d^2 · 2^{\ell−\ell_0−1}\mathfrak{ll}</span></td><td><span class="math">d · 2^{\ell-\ell_0}</span></td></tr><tr><td>Algorithm 5</td><td><span class="math">d(d + 1)/2 · N\ \mathfrak{ll}</span></td><td><span class="math">d · 2^{\ell}</span></td></tr><tr><td>Algorithm 6</td><td><span class="math">((\frac{d+1}{2})^{\ell_0} + d) N\ \mathfrak{sl} + d^2/2^{\ell_0+1}N\ \mathfrak{ll}</span></td><td><span class="math">d · 2^{\ell-\ell_0}</span></td></tr></tbody></table>

### References:

* [Speeding Up Sum-Check Proving](https://eprint.iacr.org/2025/1117.pdf) by Suyash et al, 2025.
* [Optimizing the Sumcheck Protocol](https://hackmd.io/@tcoratger/SJjJBfWWlg) by Thomas Coratger, 2025.

> Written by [Batzorig Zorigoo](https://app.gitbook.com/u/cO1lUla01ZW0seepO37jMHFTxg42 "mention") of Fractalyze
