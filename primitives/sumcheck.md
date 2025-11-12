# Sumcheck

## Overview

The sumcheck protocol is a cryptographic proof protocol designed to efficiently prove and verify whether the sum of a multivariate polynomial over a specified domain equals a claimed value, given :

* A multivariate polynomial $$g(X_1,X_2, \dots, X_{v})$$
* A finite domain $$B^v$$ for any $$B \subseteq \mathbb{F}$$
* A claimed sum&#x20;

$$
C_1 := \sum_{b_1 \in B}\sum_{b_2 \in B} \dots \sum_{b_v \in B} g(b_1, \dots, b_v)
$$

or in short $$C_1 := \sum_{b \in B^v}g(b)$$. We denote the true correct sum by $$H$$.&#x20;

Throughout this article, we assume that $$B = \{0,1\}$$ which is the case where the evaluation domain is a boolean hypercube. $$\deg_j(g)$$ denotes the degree of $$g$$ in variable $$X_j$$. We denote the prover and the verifier by $$\mathcal{P}$$ and $$\mathcal{V}$$.&#x20;

The goal is for $$\mathcal{P}$$ to convince $$\mathcal{V}$$ that the claimed $$C_1 = H$$, without having $$\mathcal{V}$$ evaluate $$g$$ on all $$\{0,1\}^v$$ elements (which would be computationally expensive).

## Protocol

1. In the first round, $$\mathcal{P}$$ sends the univariate polynomial $$g_1(X_1)$$ and claims it equals the following:

$$
\sum_{(x_2, \dots, x_v) \in \{0,1\}^{v-1}}g(X_1, x_2, \dots, x_{v})
$$

Note that $$g_1(0) + g_1(1) = C_1$$ if $$\mathcal{P}$$ is honest.&#x20;

2. $$\mathcal{V}$$ checks if $$g_1(0) + g_1(1) = C_1$$. If not, $$\mathcal{V}$$ rejects.&#x20;
3. $$\mathcal{V}$$ checks if $$\deg(g_1) \le \deg_1(g)$$. If not, $$\mathcal{V}$$ rejects.
4. $$\mathcal{V}$$ chooses a random $$r_1 \in \mathbb{F}$$  and sends it to $$\mathcal{P}$$. Both $$\mathcal{V}$$ and $$\mathcal{P}$$ set $$C_2 := g_1(r_1)$$.&#x20;
5. Now in the $$j$$th round, for $$1<j<v$$, $$\mathcal{P}$$ sends the univariate polynomial $$g_j(X_j)$$ and claims it equals the following:

$$
\sum_{(x_{j+1}, \dots, x_v) \in \{0,1\}^{v-j}}g(r_1,\dots,r_{j-1}, X_j, x_{j+1}, \dots, x_{v})
$$

Note that $$g_j(0) + g_j(1) = C_j$$ if $$\mathcal{P}$$ is honest.&#x20;

6. $$\mathcal{V}$$ checks if $$g_j(0) + g_j(1) = C_j$$. If not, $$\mathcal{V}$$ rejects.&#x20;
7. $$\mathcal{V}$$ checks if $$\deg(g_j) \le \deg_j(g)$$. If not, $$\mathcal{V}$$ rejects.
8. $$\mathcal{V}$$ chooses a random $$r_j \in \mathbb{F}$$  and sends it to $$\mathcal{P}$$. Both $$\mathcal{V}$$ and $$\mathcal{P}$$ set $$C_{j+1} := g_j(r_j)$$.

This process continues iteratively, reducing the dimension at each step until reaching a single variable.

9. In round $$v$$, $$\mathcal{P}$$ sends $$g_v(X_v)$$ to $$\mathcal{V}$$ claiming it equals:

$$
g(r_1, \dots, r_{v-1}, X_v)
$$

10. $$\mathcal{V}$$ checks if $$g_{v}(0) + g_{v}(1) = C_v$$. If not, $$\mathcal{V}$$ rejects.&#x20;
11. $$\mathcal{V}$$ chooses a random $$r_v \in \mathbb{F}$$ and evaluates $$g(r_1, \dots, r_v)$$ with a single oracle query to $$g$$. $$\mathcal{V}$$ checks that $$g_v(r_v) = g(r_1, \dots, r_v)$$. If not, $$\mathcal{V}$$ rejects.

### Completeness

Let $$s_i(X)$$ be the polynomial that $$\mathcal{P}$$ would have sent in round $$i$$ if it were honest :

$$
s_i(X_i) = \sum_{(x_{i+1}, \dots, x_v) \in \{0,1\}^{v-i}}g(r_1,\dots,r_{i-1}, X_i, x_{i+1}, \dots, x_{v})
$$

If $$C_1$$ and $$g_1(X)$$ that $$\mathcal{P}$$ provided in the first round is equal to $$H$$ and $$s_1(X)$$, respectively, then $$g_1(0) + g_1(1)$$ is equal to $$C_1$$. $$\mathcal{V}$$ checks if this holds, then randomly samples $$r_1$$ and sends it to $$\mathcal{P}$$. The only remaining part is merely a recursive invocation of another sumcheck protocol on $$s_1(r_1)$$ where the claimed value is $$C_2 := g_1(r_1)$$. At the bottom of the recursion, which is the last round of the protocol, $$\mathcal{P}$$ passes the verification as $$g_v(r_v) = s_v(r_v) = g(r_1, \dots, r_v)$$. Thus, the sumcheck protocol has perfect completeness.&#x20;

### Soundness

What is the soundness error, that is, what is the probability that $$\mathcal{P}$$ passes the protocol even though the first value given in the first step is wrong, i.e. $$C_1 \neq H$$?&#x20;

If the following occurs at any round of the protocol, $$\mathcal{P}$$ can pass the verification even if the first claimed value is wrong:&#x20;

* For the challenge $$r_i \stackrel{\$}{\leftarrow} \mathbb{F}$$ sampled by $$\mathcal{V}$$, luckily $$g_i(r_i) = s_i(r_i)$$ for the $$g_i \neq s_i$$ that $$\mathcal{P}$$ had sent in the previous round.&#x20;

Once the scenario outlined above occurs, $$\mathcal{P}$$ can proceed onto the  remaining rounds with a valid transcript to deceive $$\mathcal{V}$$. The soundness error $$\delta_s$$ is bounded by $$vd / |\mathbb{F}|$$, which can be calculated by the Schwartz-Zippel lemma.  In other words, $$\mathcal{V}$$ can be convinced that $$g_i(X) = s_i(X)$$ if $$g_i(r_i) = s_i(r_i)$$ for a random sampled $$r_i$$.&#x20;

### Cost

<figure><img src="../.gitbook/assets/image (173).png" alt=""><figcaption></figcaption></figure>

We denote the cost of evaluation oracle query to $$g$$ by $$T$$. In reality, $$T$$ is equal to the cost to evaluate $$g$$ at a single input in $$\mathbb{F}^v$$.&#x20;

#### Communication cost.&#x20;

At each round, $$\mathcal{P}$$ generates an univariate polynomial $$g_i(X)$$ whose degree is at most $$\deg_i(g)$$. To uniquely satisfy such polynomial, $$\mathcal{P}$$ needs to send $$\deg_i(g)$$ evaluations on $$g$$.&#x20;

#### Prover cost.

The cost of computing such prescribed messages at each round is equal to evaluating $$g_i(X)$$ at $$\deg_i(g)$$ points. Letting the oracle query cost be $$T$$ and assuming $$\deg_i(g) = O(1)$$, the prover's runtime cost at round $$i$$ is $$O(2^v \cdot T)$$.&#x20;

#### Verifier cost. &#x20;

Also assuming $$\deg_i(g) = O(1)$$, the total verifier cost is dominated by $$T$$; the cost of evaluating $$g$$ only once.&#x20;

Combining with Fiat-Shamir transform, we have successfully reduced a complicated interactive proof into a simple evaluation task for $$\mathcal{V}$$. Unfortunately, in many real world problems, $$\mathcal{V}$$ does not have access to a constant cost oracle query to $$g$$, or even a single evaluation is prohibitively expensive.&#x20;

The key part of sumcheck-based protocols is about how we will grant $$\mathcal{V}$$ oracle access. There should be either a way $$\mathcal{V}$$ can efficiently evaluate $$g$$, or we should run another protocol for the value $$g(r)$$ (refer to [polynomial commitment scheme](commitment-scheme/)).

## Example&#x20;

### #SAT

For a $$n$$-variate boolean formula $$\phi$$, whose size $$S$$ is $$\mathsf{poly} (n)$$, calculate&#x20;

$$
\sum_{x \in \{0,1\}^n} \phi(x)
$$

The fastest algorithm ever known to solve $$\#\mathsf{SAT}$$ is exponential, which means it's not much better than brute-forcing it.&#x20;

In addition to that, there is no polynomial time deterministic & non-interactive algorithm $$\mathcal{V}$$. This is because even if $$\mathcal{P}$$ provides every valid input it knows as a witness, there is no way $$\mathcal{V}$$ can check in polynomial time whether there are more valid inputs. $$\#\mathsf{SAT} \notin \mathsf{NP}$$.&#x20;

However there exists an interactive proof for this, which enables $$\mathcal{V}$$ to verify if the claimed value is correct within polynomial time.  $$\#\mathsf{SAT} \in \mathsf{IP}$$.&#x20;

This implies that the interactive proof is more powerful than merely proving an NP statement. It is known that $$\mathsf{NP} \sube \mathsf{PSPACE} = \mathsf{IP}$$, where $$\mathsf{PSPACE}$$ denotes the set of all problems that can be solved by Turing machines with polynomial space complexity.

#### Protocol

Transform $$\phi$$ into an arithmetic circuit $$\psi(x)$$. AND gate can be extended to $$y \cdot z$$, OR gate to $$y + z - y\cdot z$$, NOT gate to $$1-y$$. Now extend circuit $$\psi$$ to a $$n$$-variate polynomial $$g$$, which means that $$\sum_{x\in\{0,1\}^{n}} g(x) = \sum_{x\in\{0,1\}^{n}} \phi(x)$$.&#x20;

Degree of $$g$$ is $$\mathsf{poly} (S)$$. Run the sumcheck protocol on $$g$$.&#x20;

***

## References

* [https://people.cs.georgetown.edu/jthaler/ProofsArgsAndZK.pdf](https://people.cs.georgetown.edu/jthaler/ProofsArgsAndZK.pdf)

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") and [Carson Lee](https://app.gitbook.com/u/Hm5RHrPlu2fbxwXnpUgxmvXVTwB3 "mention") from [A41](https://www.a41.io/)
