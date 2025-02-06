# Sumcheck

The sumcheck protocol is a cryptographic proof protocol designed to efficiently verify whether the sum of a polynomial $$f(X_0, X_1, …, X_{n-1})$$ over a specified domain equals a claimed value.

Given:

* A multivariate polynomial $$f(X_0, X_1, …, X_{n-1})$$
* A finite domain $$D$$, often a Boolean hypercube $$\{0, 1\}^n$$
* A claimed sum $$C = \sum_{X \in D^n}f(X)$$

The goal is for the prover to convince the verifier that the claimed $$C$$ is correct, without the verifier evaluating $$f(X)$$ on all $$D^n$$ elements (which would be computationally expensive).

Protocol Steps

1. The prover claims the equation below and sends this $$C$$ to the verifier.

$$
C = \sum_{X_0, X_1, \dots, X_{n-1} \in D} f(X_0, X_1, \dots, X_{n-1})
$$

2. The prover sends the univariate polynomial below, which reduces the problem from $$n$$ variables to $$n - 1$$.

$$
g_0(X_0) = \sum_{X_0, X_1, \dots, X_{n-1} \in D}f(X_0, X_1, \dots, X_{n-1})
$$

3. The verifier checks the equation below and if this doesn't hold the verifier rejects.

$$
\sum_{X_0 \in D}g_0(X_0) \stackrel{?}= C
$$

4. The verifier chooses a random $$r_0$$ and sends it to the prover.
5. The prover now claims:

$$
g_1(X_1) = \sum_{X_2, \dots, X_{n-1} \in D}f(r_0, X_1, \dots, X_{n-1})
$$

6. The verifier checks:

$$
\sum_{X_1 \in D}g_1(X_1) \stackrel{?} = g_0(r_0)
$$

6. This process continues iteratively, reducing the dimension at each step until reaching a single variable.
7. The prover now claims:

$$
g_{n-2}(X_{n-2}) = \sum_{X_{n-1} \in D}f(r_0, r_1, \dots, X_{n-2}, X_{n-1})
$$

8. The verifier checks:

$$
\sum_{X_{n-1} \in D}g_{n-1}(X_{n-1}) \stackrel{?}=g_{n-2}(r_{n-2})
$$

9. The verifier chooses a random $$r_{n-1}$$ and sends it to the prover.
10. The verifier access to the oracle at random $$r_{n-1}$$ and checks:

$$
g_{n-1}(r_{n-1}) \stackrel{?}= f(r_0, r_1, \dots, r_{n-2}, r_{n-1} )
$$

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from A41
