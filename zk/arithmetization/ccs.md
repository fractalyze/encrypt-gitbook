# CCS

## Customizable Constraint System

**Definition 2.1 (R1CS).** An R1CS structure $$\mathcal{S}$$ consists of:

* size bounds $$m, n, N,\ell \in \mathbb{N}$$ where $$n > \ell$$
* and three matrices $$\bm{A},\bm{B},\bm{C} \in  \mathbb{F}^{m\times n}$$ with at most $$N = \Omega(\mathsf{max}(m, n))$$ non-zero entries in total.&#x20;

An R1CS instance consists of public input $$\bm{x} \in F^\ell$$. An R1CS witness consists of a vector $$\bm{w} ∈ F^{n−\ell−1}$$. An R1CS structure-instance tuple $$(\mathcal{S}, \bm{x})$$ is satisfied by an R1CS witness $$\bm{w}$$ if:

$$
(\bm{A} \cdot \bm{z}) \circ (\bm{B} \cdot \bm{z}) − \bm{C} \cdot \bm{z} = \bm{0}
$$

where

$$
\bm{z} = (\bm{w}, 1, \bm{x}) ∈ \mathbb{F}^n
$$

&#x20;Here, $$\bm{A}\cdot \bm{z}$$ denotes a matrix-vector multiplication operation, $$\circ$$ denotes the Hadamard (i.e., entry-wise) product between vectors, and $$\bold{0}$$ is a zero vector in $$\mathbb{F}^m$$.&#x20;

**Definition 2.2 (CCS)**. A CCS structure $$\mathcal{S}$$ consists of:

* size bounds $$m, n, N, \ell, t, q, d \in \mathbb{N}$$ where $$n > \ell$$
* a sequence of matrices $$\bm{M}_0,\dots ,\bm{M}_{t−1}\in \mathbb{F}_{m\times n}$$ with at most $$N = \Omega(\mathsf{max}(m, n))$$ non-zero entries in total
* a sequence of $$q$$ multisets $$[S_0,\dots, S_{q−1}]$$, where an element in each multi-set is from the domain $$\{0, . . . , t − 1\}$$ and the cardinality of each multi-set is at most $$d$$
* a sequence of $$q$$ constants $$[c_0,\dots, c_{q−1}]$$, where each constant is from $$\mathbb{F}$$

A CCS instance consists of public input $$\bm{x} \in \mathbb{F}^{\ell}$$. A CCS witness consists of a vector $$\bm{w}\in\mathbb{F}^{n-\ell-1}$$. Then, a CCS structure-instance tuple $$(\mathcal{S}, \bm{x})$$ is satisfied by a CCS witness $$\bm{w}$$ if:

$$
\sum^{q-1}_{i=0}c_i\cdot \bigcirc_{j\in S_{i}}\bm{M}_j\cdot \bm{z}=\bold{0}
$$

where

$$
\bm{z} = (\bm{w}, 1,\bm{x}) \in \mathbb{F}^n
$$

$$\bm{M}_j \cdot \bm{z}$$ denotes matrix-vector multiplication, $$\bigcirc$$ denotes the Hadamard product between vectors, and $$\bold{0}$$ is a zero vector in $$\mathbb{F}^m$$.

{% hint style="info" %}
CCS can be easily converted to R1CS. I will leave it as a thought exercise.
{% endhint %}

### Representing Plonkish with CCS

Let's recall the Plonkish constraint system first.

**Definition 2.3 (Plonkish)**. A Plonkish structure $$\mathcal{S}$$ consists of:

* size bounds $$m, n, \ell, t, q, d, e \in \mathbb{N}$$;
* a multivariate polynomial $$g$$ in $$t$$ variables, where $$g$$ is expressed as a sum of $$q$$ monomials and each monomial has a total degree at most $$d$$;
* a vector of constants called selectors $$\bm{s} \in \mathbb{F}^e$$; and
* a set of $$m$$ constraints. Each constraint $$i$$ is specified via a vector $$\bm{T}_i$$ of length $$t$$, with entries in the range $$\{0, . . . , n + e − 1\}$$. $$\bm{T}_i$$ is interpreted as specifying $$t$$ entries of a purported satisfying assignment $$z$$ to feed into $$g$$.

A Plonkish instance consists of public input $$\bm{x} \in \mathbb{F}^\ell$$. A Plonkish witness consists of a vector $$\bm{w} \in F^{n−\ell}$$. A Plonkish structure-instance tuple $$(\mathcal{S}, \bm{x})$$ is satisfied by a Plonkish witness $$\bm{w}$$ if:

$$
\forall i \in \{0,\dots,m-1\}: g(\bm{z}[\bm{T}_i[1]], . . . , \bm{z}[\bm{T}_i[t]]) = 0
$$

where $$\bm{z} = (\bm{w}, \bm{x}, \bm{s})\in F^{n+e}$$.

### Representing AIR with CCS

**Definition 2.4 (AIR)**. An AIR structure $$\mathcal{S}$$ consists of:

* size bounds $$m, t, q, d \in \mathbb{N}$$, where $$t$$ is even;
* a multivariate polynomial $$g$$ in $$t$$ variables (also known as state transition constraint), where $$g$$ is expressed as a sum of $$q$$ monomials and each monomial has total degree at most $$d$$.

An AIR instance consists of public input and output $$\bm{x} \in \mathbb{F}^t$$. An AIR witness consists of a vector $$\bm{w} ∈ \mathbb{F}^{(m−1)\cdot t/2}$$. Let

$$
\begin{equation}
\bm{z} = (\bm{x}[..t/2], \bm{w}, \bm{x}[t/2 + 1..]) \in (\mathbb{F}^{t/2})^{m+1} \tag{6}
\end{equation}
$$

Notice that, $$\bm{z}$$ can be viewed as $$\mathbb{F}^{(m+1)\times t/2}$$ matrix. Let us index the $$m + 1$$ “rows” of $$\bm{z}$$ as $$0, 1, . . . , m$$. An AIR structure-instance tuple $$(\mathcal{S}, \bm{x})$$ is satisfied by an AIR witness $$w$$ if:

$$
\begin{equation}
\forall i ∈ \{1, . . . , m\}: g(\bm{z}[i − 1], \bm{z}[i]) = 0. \tag{7}
\end{equation}
$$

#### Handling many AIR constraint polynomials

To represent the constraints capturing a single step of a CPU, typically many polynomials $$g$$ will be needed, say $$g_1, . . . , g_k$$. In practice, $$k$$ may be in the many dozens to hundreds \[GPR21, BGtRZt23]. There is, however, a straightforward and standard randomized reduction from AIR with $$k > 1$$ constraint polynomials to AIR with a single constraint polynomial $$g$$: the verifier picks a random $$r \in \mathbb{F}$$ and sends it to the prover, and the prover replaces $$g_1, . . . , g_k$$ with the single constraint polynomial

$$
\begin{equation}
g:=\sum^{k-1}_{i=0}r^i\cdot g_i \tag{8}
\end{equation}
$$

If Equation (7) holds for all $$g_1, \dots , g_k$$ in place of $$g$$, then with probability 1 over the choice of $$r$$ the same will hold for the random linear combination $$g$$. Meanwhile, if it fails to hold for any $$g_i$$ then with probability at least $$1 − \frac{(k − 1)}{|\mathbb{F}|}$$, it will fail to hold for the random linear combination $$g$$. You could also get $$k$$ random values for each polynomial, which improves the soundness error to $$\frac{1}{|\mathbb{F}|}$$.

**Lemma 3** (AIR to CCS transformation). Given an AIR structure $$\mathcal{S}_{AIR} = (m, t, q, d,g)$$, instance $$\mathcal{I}_{AIR} = \bm{x}$$, and witness $$\bm{w}_{AIR} = \bm{w}$$, there exists a CCS structure $$\mathcal{S}_{CCS}$$ such that the tuple $$(\mathcal{S}_{CCS}, \bm{x})$$ is satisfied by $$\bm{w}$$ if and only if $$(\mathcal{S}_{AIR},\bm{x})$$ is satisfied by $$\bm{w}$$. The natural NP checker’s time to verify the tuple $$((\mathcal{S}_{CCS}, \bm{x}), \bm{w})$$ is identical to the runtime of the NP checker that evaluates $$g$$ monomial-by-monomial when checking that $$((\mathcal{S}_{AIR}, \bm{x}), \bm{w})$$ satisfies Equation (7).

_Proof_.

* Let $$\bm{w}_{CCS} = \bm{w}_{AIR}$$ and $$\mathcal{I}_{CCS} = \mathcal{I}_{AIR}$$.
* Let $$\mathcal{S}_{CCS}= (m, n, N, \ell, t, q, d, [\bm{M}_0, \dots , \bm{M}_{t−1}], [s_0, \dots , s_{q−1}], [c_0, \dots , c_{q−1}])$$ where:
  * $$m,t,q,d$$ are from the $$\mathcal{S}_{AIR}$$
  * $$\ell \leftarrow t/2, n\leftarrow (m+1)\cdot t/2$$

_Deriving_ $$\bm{M}_0, \dots , \bm{M}_{t−1}$$_, and_ $$N$$_._&#x20;

Recall that $$g$$ in $$\mathcal{S}_{AIR}$$ is a multivariate polynomial in $$t$$ variables. There is a CCS row for each of the $$m$$  transition constraints in $$\mathcal{S}_{AIR}$$, so, if we index the CCS rows by $$\{0,\dots, m − 1\}$$, it suffices to specify how the $$i$$th row of these matrices is set. We consider three cases:

For all  $$i \in \{0,\dots,m-1\}, j \in \{0, 1 . . . , t − 1\}$$, let $$k_j = i \cdot t/2 + j$$.

* If $$i = 0$$ and $$j < t/2$$, we set $$\bm{M}_j[i][j + |\bm{w}_{AIR}|]=1$$.&#x20;
* If $$i = m − 1$$ and $$j \ge t/2$$, we set $$\bm{M}_j[i][j + |\bm{w}_{AIR}| + t/2] = 1$$.
* Otherwise, we set $$\bm{M}_j[i][k_j] = 1$$.

NOTE: In this version, I think the authors assumed:

$$
\begin{equation}
\bm{z} = (\bm{w}, \bm{x}[..t/2], \bm{x}[t/2 + 1..]) \in \mathbb{F}^{t/2\cdot(m+1)}
\end{equation}
$$
