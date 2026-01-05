# Notations & Definitions

&#x20;This page is a glossary for notation and concepts present in the documentation.

## Sets and Groups [#](https://www.zkdocs.com/docs/zkdocs/notation/#sets-groups-and-special-functions) <a href="#sets-groups-and-special-functions" id="sets-groups-and-special-functions"></a>

* $$\mathbb{Z}$$ is the set of integers, $$\{\dots, -2, -2, -1, 0, 1, 2, \dots\}$$.&#x20;
* $$\mathbb{N}$$ is the set of integers greater than or equal to 0, $$\{0,1,2,\dots\}$$.
* $$[n]$$ is the finite set of integers $$\{1, \dots ,n\}$$.&#x20;
* $$\mathbb{Z}_n$$ are the integers modulo $$n$$, a set associated with the equivalence classes of integers $$\{0,1, \dots ,n−1\}$$.
* $$\mathbb{Z}_n^∗$$ is the multiplicative group of integers modulo $$n$$: an element $$e$$ from $$\mathbb{Z}_n$$ is in $$\mathbb{Z}_n^∗$$ iff $$\gcd(e,n)=1$$, that is $$\mathbb{Z}_n^* = \{e \in  \mathbb{Z}_n: \gcd(e, n) = 1 \}$$. When $$n$$ is prime, then $$\mathbb{Z}_n^* = \{1, \dots, n-1\}$$.
* $$\mathbb{F}_p$$ is the finite field of order $$p$$; when $$p$$ is a prime number, these are the integers modulo $$p$$, $$\mathbb{Z}_p$$; when $$p$$ is a prime power $$q^k$$, these are [Galois fields](https://en.wikipedia.org/wiki/Finite_field).
* $$|S|$$ is the _order_ of a set $$S$$, i.e., its number of elements. For example, $$|\mathbb{Z}_n^*| = \Phi(n)$$, and for a prime $$n$$, $$|\mathbb{Z}^*_n| = n - 1$$.
* $$\mathcal{L}$$: An **evaluation domain**, typically used in FFT-based polynomial commitment schemes (e.g., domains of size a power of two).
* $$\mathcal{R}$$: A **relation** or constraint system, such as an R1CS (Rank-1 Constraint System).
* $$\mathsf{RS}$$: **Reed-Solomon codes**.

## Vectors [#](https://www.zkdocs.com/docs/zkdocs/notation/#vectors) <a href="#vectors" id="vectors"></a>

* $$\bm{a} \in \mathbb{Z}_q^n$$ is a vector $$(a_1, \dots ,a_n)$$ with $$a_i \in \mathbb{Z}_q$$ for all $$i$$.
* $$c \cdot \bm{a}$$ denotes the scalar product $$(c \cdot a_1, \cdots ,c \cdot a_n)$$.
* $$\lang \bm{a}, \bm{b} \rang$$ denotes the inner product $$\sum_{i = 1}^n = a_i \cdot b_i$$

## Sampling [#](https://www.zkdocs.com/docs/zkdocs/notation/#sampling) <a href="#sampling" id="sampling"></a>

In protocol specifications, we will often need to uniformly sample elements from sets. We will use the following notation:

* $$x \stackrel{\$}{\leftarrow} X$$, where $$x$$ is uniformly sampled from the set $$X$$.

## Assertions [#](https://www.zkdocs.com/docs/zkdocs/notation/#assertions) <a href="#assertions" id="assertions"></a>

We will use assertions in protocol descriptions. When the assertions do not hold, the protocol must abort to avoid leaking secret information.

* $$a\stackrel{?}=b$$, requires $$a=b$$, and aborts otherwise
* $$a \stackrel{?}> b$$, requires $$a>b$$, and aborts otherwise
* $$a \stackrel{?}\in S$$, requires that $$a$$ is in the set $$S$$, and aborts otherwise.

## Hash Functions [#](https://www.zkdocs.com/docs/zkdocs/notation/#hash-functions) <a href="#hash-functions" id="hash-functions"></a>

* $$\mathsf{Hash}(\cdot)$$ is a cryptographically secure domain-separated hash function.
* $$\mathsf{Hash}(\cdot)[0:k]$$ is a cryptographically secure domain-separated hash function with specific output-size of kk-bits.

## Special Functions

* $$\deg(f)$$: The degree of a polynomial $$f$$, i.e., the highest exponent with non-zero coefficient in $$f$$.
* $$\log x$$: The logarithm of $$x$$ (base context-dependent, often base 2 in crypto).
* $$\max(x_1, \dots, x_n)$$: The maximum of a set of values.
* $$\min(x_1, \dots, x_n)$$: The minimum of a set of values.
*   $$\gcd(n,m)$$ is the positive [greatest common divisor](https://en.wikipedia.org/wiki/Greatest_common_divisor) of integers $$n$$ and $$m$$; when $$\gcd(n,m)=1$$

    , $$n$$ and $$m$$ are said to be _coprime_.
* $$\varphi(n)$$ is Euler’s [totient function](https://en.wikipedia.org/wiki/Euler's_totient_function); for $$n \ge 1$$, it is the number of integers in $$\{1, \dots ,n\}$$ coprime with $$n$$.
* $$\operatorname{adj}(M)$$ denotes the adjugate of the matrix $$M$$, which is formed by taking the transpose of the matrix of cofactors of $$M$$.

## Others

* A Montgomery form of $$a \in \mathbb{Z}_N$$ is represented by $$\bar{a} = aR \mod N$$, given a modulus $$N$$ and a Montgomery radix $$R$$ such that $$\gcd (N, R) = 1$$.&#x20;

## References [#](https://www.zkdocs.com/docs/zkdocs/notation/#references)

* [https://www.zkdocs.com/docs/zkdocs/notation/](https://www.zkdocs.com/docs/zkdocs/notation/)
