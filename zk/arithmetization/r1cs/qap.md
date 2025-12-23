# QAP

## Introduction

Given a R1CS instance $$\bm{A},\bm{B},\bm{C}\in \mathbb{F}^{m\times n}$$, the verifier has to check if it holds:

$$
\bm{Az}\circ\bm{Bz}\stackrel{?}=\bm{Cz}
$$

which is a $$O(m)$$ complexity task so we want to somehow reduce this to $$O(1)$$.

### Preliminaries

1. [Number Theoretic Transform](https://fractalyze.gitbook.io/intro/~/revisions/0AAov1j5GF4J6Ca62R1w/primitives/number-theoretic-transform)
2. [Schwartz-Zippel Lemma](https://en.wikipedia.org/wiki/Schwartz%E2%80%93Zippel_lemma)

## Protocol Explanation

Let's consider a simpler case where we want to check if $$\bm{Au}=\bm{Bv}$$. Denote $$i$$th column vectors of $$\bm{A}$$ and $$\bm{B}$$ as $$\bm{a}_i$$ and $$\bm{b}_i$$ respectively. Then, what we want to check becomes:

$$
\sum_{i=1}^n\bm{a}_iu_i\stackrel{?}=\sum_{i=1}^n\bm{b}_iv_i
$$

So if we consider polynomials $$f_i=\mathsf{INTT}(\bm{a}_i)$$ and $$g_i=\mathsf{INTT}(\bm{b}_i)$$ then, it is equivalent of checking if this two are same:

$$
\sum_{i=1}^nf_iu_i\stackrel{?}=\sum_{i=1}^n g_iv_i
$$

Since the sum itself is another polynomial, if we denote them as $$f$$ and $$g$$, we have simple equality check:

$$
f\stackrel{?}=g
$$

This check can be done using a single random challenge following the Schwartz-Zippel Lemma.

### Applying to R1CS

Similarly, let's denote the polynomial interpolation of $$i$$th column of $$\bm{A}$$, $$\bm{B}$$, $$\bm{C}$$ as $$f_{a,i},f_{b,i}, f_{c,i}$$. Then, the R1CS check becomes:

$$
\sum_{i=1}^nf_{a,i}z_i\sum_{i=1}^nf_{b,i}z_i\stackrel{?}=\sum_{i=1}^n f_{c,i}z_i
$$

Since each sum results in a single polynomial, we can simplify this equation as:

$$
f_a\circ f_b\stackrel{?}=f_c
$$

### Degree imbalance

The equality above is not as simple as it looks since the $$\deg (f_a\circ f_b)=2m =2\deg(f_c)$$. Because of this imbalance, our polynomials are not necessarily equal but if we think about the $$\mathsf{INTT}$$, they must be equal over the evaluation domain:

$$
\forall i =0,\dots,m-1: (f_a\circ f_b)(\omega^i)= f_c(\omega^{i})
$$

which is equivalent to checking:

$$
\forall i =0,\dots,m-1: (f_a\circ f_b-f_c)(\omega^i)\stackrel{?}=0
$$

In other words, $$f_a\circ f_b - f_c$$ should be divisible by the degree $$m$$ **zero polynomial**:

$$
t(X)=(X-w^0)(X-\omega^1)\dots(X-w^{m-1})=X^m-1
$$

Therefore, the **quotient polynomial** must be well defined:

$$
h(X)=\frac{f_a\circ f_b - f_c}{t}(X)
$$

### Final formula for QAP

Prover commits to $$f_a,f_b,f_c,h$$ polynomials and verifier checks with random challenge $$\tau \in_R \mathbb{F}$$:

$$
f_a(\tau)\cdot f_b(\tau)=f_c(\tau) + h(\tau)\cdot t(\tau)
$$

## References

1. [https://rareskills.io/post/quadratic-arithmetic-program#qap-end-to-end](https://rareskills.io/post/quadratic-arithmetic-program#qap-end-to-end)
