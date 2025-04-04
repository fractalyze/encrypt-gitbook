# Lasso

[Lasso](https://eprint.iacr.org/2023/1216) by Srinath Setty, Justin Thaler and Riad Wahby

***

## Overview

This paper introduces $$\mathsf{Lasso}$$, which is a lookup argument that allows an untrusted prover to commit to a vector $$a \in \mathbb{F}^m$$ and prove that all entries of $$a$$ reside in some predetermined table $$t \in \mathbb{F}^n$$.

$$\mathsf{Lasso}$$'s performance characteristics unlock the so-called "[Lookup singularity](https://zkresear.ch/t/lookup-singularity/65)".

A key difference of $$\mathsf{Lasso}$$ from other lookup arguments is that, if the table $$t$$ is _structured(as we will explain later)_, then no party needs to commit to $$t$$, enabling the use of enormous tables (e.g. of size $$2^{128}$$ or larger). Most common use cases of lookup argument, such as range checks, bitwise operations or big-number arithmetic, obtains huge benefits in cost from this characteristic of $$\mathsf{Lasso}$$.&#x20;

### Lookup arguments.

Imagine that a prover wishes to establish that at no point in a program's execution did any integer ever exceed $$2^{128}$$.

A naive approach to accomplish this inside a circuit-SAT instance is to have the circuit take 128 field elements as its advice inputs, which constitute the binary representation of $$x$$. The circuit checks that all 128 advice elements are in $$\{0,1\}$$ and they indeed equal to the binary representation of $$x$$, i.e., $$x = \sum^{127}_{i=0} 2^i \cdot b_i$$. A single overflow check turns into at least 129 constraints and additional 128 field elements in witness.

Lookup tables offer a better approach. The verifier initializes a lookup table containing all integers between $$0$$ and $$2^{128} - 1$$. Then the overflow check above amounts to simply confirming that $$x$$ _is in the table_. Of course a table of size $$2^{128}$$ is way too large to be represented, even for the prover. This paper is devoted to describe the techniques to enable such a table lookup **without requiring the table to ever be explicitly materialized** by either of them. Indeed, the table in this example is highly structured in a way that it can be optimized using the techniques we will introduce.

$$\mathsf{Lasso}$$'s starting point is $$\mathsf{Spark}$$, which is an optimal **polynomial commitment scheme for sparse multilinear polynomials** from Spartan. Just as Spartan, $$\mathsf{Lasso}$$ can be instantiated with any multilinear PCS. Particularly for SNARKs that have the prover commit to the witness using a multilinear PCS, $$\mathsf{Lasso}$$ can be applied seamlessly to provide a lookup argument in either R1CS or Plonkish circuit satisfiability.

### Jolt and Lasso.&#x20;

$$\mathsf{Jolt}$$ is a RISC-V-powered zkVM which achieved substantial improvements in front-end design by exploiting $$\mathsf{Lasso}$$. $$\mathsf{Jolt}$$ replaces the instruction execution part of a zkVM circuit with $$\mathsf{Lasso}$$ by letting it look up into a gigantic evaluation table rather than evaluating it in a circuit. For each of the primitive RISC-V instructions, the resulting evaluation table has the structure that we require to apply $$\mathsf{Lasso}$$ for a significant reduce in the cost. Thaler states that $$\mathsf{Lasso}$$ and $$\mathsf{Jolt}$$ together essentially achieve a vision outlined by Barry WhiteHat called the [_Lookup singularity_](https://zkresear.ch/t/lookup-singularity/65) - seeking to transform arbitrary computer program into circuits that only perform lookups.

***

## Preliminiaries

As they already explained in previous presentations, I skip the parts for Multilinear extensions, SNARKs, Polynomial commitment scheme, Polynomial IOP and the [sumcheck protocol](../../primitives/sumcheck.md).

### Sparsity.

$$m$$-sparse refers to multilinear polynomials $$g: \mathbb{F}^l \rightarrow \mathbb{F}$$ in $$l$$ variables such that $$g(x) \neq 0$$ for at most $$m$$ values of $$x \in \{0,1\}^l$$. In other words, $$g$$ has at most $$m$$ non-zero coefficients in its multilinear Lagrange polynomial basis. If $$m \ll 2^l$$, only a tiny fraction of the possible coefficients are non-zero. If $$m = \Theta(2^l)$$, we refer to such $$g$$ as a _dense_ polynomial.

### Dense representation of multilinear polynomials.

The fact that the MLE of a function is unique offers the following method to represent any multilinear polynomial, which is briefer than its point-value representation or coefficient representation.

A multilinear polynomial $$g: \mathbb{F}^l \rightarrow \mathbb{F}$$ can be represented uniquely by the list of tuples $$L$$ such that for all $$i \in \{0,1\}^l, (\mathsf{to}\text{-}\mathsf{field}(i), g(i)) \in L$$ if and only if $$g(i) \neq 0$$.&#x20;

Here, $$\mathsf{to}\text{-}\mathsf{field}$$ is the canonical injection from $$\{0,1\}^l$$ to $$\mathbb{F}$$ and $$\mathsf{to}\text{-}\mathsf{bits}$$ is the inverse mapping. You can think of them as transformations between a field element and its binary representation.&#x20;

We refer to $$L$$ as the dense representation of a sparse polynomial $$g$$.&#x20;

#### Definition 1.&#x20;

A multilinear polynomial $$g$$ in $$l$$ variables is a sparse multilinear polynomial if $$|\mathsf{DenseRepr}(g)|$$ is sub-linear in $$\Theta(2^l)$$. Otherwise, it is a dense multilinear polynomial.&#x20;

***

## Technical overview

For a lookup argument from $$a \in \mathbb{F}^m$$ to $$t \in \mathbb{F}^n$$, it suffices for a prover to show that it knows a sparse matrix $$M \in \mathbb{F}^{m \times n}$$ such that (1) for each row of $$M$$, only one cell has a value of 1 and the rest are zeroes and that (2) $$M \cdot t = a$$.&#x20;

Below demonstrates a simple example of demonstrating that $$a =  [2, 3, 0]$$ is a valid lookup into the table $$t = [0, 1, 2, 3]$$. You can see that $$M$$ is indeed sparse.&#x20;

$$
\begin{bmatrix} 
   0 & 0 & 1 & 0  \\
   0 & 0 & 0 & 1 \\
   1 & 0 & 0 & 0  \\
   \end{bmatrix} 
\cdot 
\begin{bmatrix} 
   0  \\
   1  \\
   2  \\
   3    \\
   \end{bmatrix}
   =
  \begin{bmatrix} 
   2  \\
   3  \\
   0  \\
   \end{bmatrix}
$$

Proving the matrix-vector multiplication above is equivalent to confirming that

Equation (1):

$$
\sum_{{y} \in \{0,1\}^{\log n}} \widetilde{M}({r},{y}) \cdot \tilde{t}({y}) = \tilde{a}({r})
$$

for an $${r} \in \mathbb{F}^{\log m}$$ randomly sampled by the verifier.

To run a sumcheck protocol on this equation to prove it, the verifier should be able to evaluate $$\widetilde{M}({r},{y}) \cdot \tilde{t}({y})$$ at some random $$y$$ succintly, i.e. with a cost independent of $$n$$, regarding that the size of lookup table might be very large. Clearly, we cannot achieve this by committing to $$\widetilde{M}$$ and $$\tilde{t}$$ using normal PCS.

$$M$$ is a sparse matrix where only $$m$$ elements out of $$m \times n$$ are non-zero. Hence, the above is efficiently provable by having the prover commit to the sparse polynomial $$\widetilde{M}$$ using $$\mathsf{Surge}$$, which is a generalization of $$\mathsf{Spark}$$ for $$\mathsf{Lasso}$$, and instead of commiting to the enormous $$t$$, sending only its succint description assuming that it is structured in a way we define. Once we have such a commitment scheme, we instantly obtain the desired lookup argument.

***

## Spark

$$\mathsf{Spark}$$, as explained above, is a polynomial commitment scheme for sparse polynomials. Let $$g$$ be a ($$\log N$$)-variate multilinear polynomial with sparsity $$m$$. The prover commits to a unique dense representation of the sparse polynomial $$g$$, which is the list that specifies only the multilinear Lagrange bases with non-zero coefficients. When the verifier requests an evaluation $$g(r)$$, the prover returns the claimed evaluation $$v$$ along with the evaluation proof.&#x20;

Let $$c$$ be such that $$N = m^c$$ without loss of generality. There is a simple algorithm that takes as input the dense representation of $$g$$ and outputs $$g(r)$$ in $$O(c \cdot m)$$ time.&#x20;

#### Algorithm 1.&#x20;

Decompose the $$\log N = c \log m$$ variables of $$r$$ into $$c$$ blocks, each of size $$\log m$$, writing $$r = (r_1, ..., r_c) \in (\mathbb{F}^{\log m})^c$$. Then **any (**$$\log N$$**)-variate Lagrange basis polynomial evaluated at** $$r$$ **can be expressed as a product of** $$c$$ **smaller Lagrange basis polynomials.** With $$O(c\cdot m)$$ cost of pre-computing a write-once memory, each basis polynomial can be calculated by $$c$$ memory look-ups and extra field operations.&#x20;

#### Example.&#x20;

Let $$N = 2^6$$, $$c = 3$$ and $$m = 2^2$$.

Let $$g(x) = 1$$ for $$x = 000001, ..., 000100$$. Otherwise, $$g(x) = 0$$ for $$x \in \{0,1\}^6$$. Subscript 1, 2, and 3 each denotes the 2-bits length partition of the 6-bits binary string. $$\mathcal{X}_i(r)$$ is equal to $$\widetilde{\mathsf{eq}}(i, r)$$.&#x20;

Observe that the evaluation of $$g(x)$$ on some $$r$$ can be decomposed in this way:&#x20;

$$
g({r}) = \sum^{4}_{i=1} 1 \cdot \mathcal{X}_{\mathsf{to}\text{-}\mathsf{bits}(i)}({r}) \\
= \sum^4_{i=1} 1\cdot (\mathcal{X}_{\mathsf{to}\text{-}\mathsf{bits}(i)_{1}}({r_{1}}) \cdot \mathcal{X}_{\mathsf{to}\text{-}\mathsf{bits}(i)_{2}}({r_{2}}) \cdot
\mathcal{X}_{\mathsf{to}\text{-}\mathsf{bits}(i)_{3}}({r_{3}}))
$$

The memory is composed of $$c(=3)$$ subtables $$t_1, t_2$$ and $$t_3$$ where each $$t_i$$ is a table of size $$m$$ populated by the values from $$\mathcal{X}_{0,0}({r_i})$$ to $$\mathcal{X}_{1,1}({r_i})$$. Each subtable can be computed by the prover in $$O(m)$$, leading to a total computation cost of $$O(c \cdot m)$$.

| t₁                           | t₂                           | t₃                           |
| ---------------------------- | ---------------------------- | ---------------------------- |
| $$\mathcal{X}_{0,0}({r_1})$$ | $$\mathcal{X}_{0,0}({r_2})$$ | $$\mathcal{X}_{0,0}({r_3})$$ |
| $$\mathcal{X}_{0,1}({r_1})$$ | $$\mathcal{X}_{0,1}({r_2})$$ | $$\mathcal{X}_{0,1}({r_3})$$ |
| $$\mathcal{X}_{1,0}({r_1})$$ | $$\mathcal{X}_{1,0}({r_2})$$ | $$\mathcal{X}_{1,0}({r_3})$$ |
| $$\mathcal{X}_{1,1}({r_1})$$ | $$\mathcal{X}_{1,1}({r_2})$$ | $$\mathcal{X}_{1,1}({r_3})$$ |

Notice that this is cheaper with a factor of $$\log m$$ than a naive approach, whose cost is $$O(m \log N)$$ since there are $$m$$ Lagrange basis polynomials with $$\log N$$ variables.&#x20;

The efficiency of the algorithm above comes from the fact that $$g$$ is sparse. $$\mathsf{Spark}$$ **is merely a SNARK as an argument that the prover correctly ran this algorithm** on the committed description of $$g$$. For simplicity, in the demonstrations below, we talk about the special case where $$c = 2$$ and then expand it for a general result.&#x20;

***

### Detailed explanation

Let $$D$$ be a $$m$$-sparse ($$2\log m$$)-variate multilinear polynomial, as we fixed the value of $$c$$ to be 2. For now, you can think of $$D$$ as an $$m \times m$$ matrix with only $$m$$ nonzero elements. &#x20;

When its dense representation is given, one can express its evaluation as below, by decomposing the ($$2\log m$$)-variate Lagrange basis polynomial into two ($$\log m$$)-variate Lagrange basis polynomial.

$$
D({r_x}, {r_y}) = \sum_{({i},{j})\in\{0,1\}^{\log \mathsf{m}} \times \{0,1\}^{\log \mathsf{m}} : D({i},{j}) \neq 0} D({i},{j}) \cdot \widetilde{\mathsf{eq}}({i}, {r_x}) \cdot \widetilde{\mathsf{eq}}({j}, {r_y})
$$

Let's introduce three ($$\log m$$)-variable multilinear polynomials $$\mathsf{val}$$, $$\mathsf{row}$$ and $$\mathsf{col}$$, which extend the dense representation of $$D$$. That is, $$k \in \{0,1\}^{\log m}$$ iterates the m nonzero elements in $$D$$ in some canonical order, and for each $$k$$, there exist some $$i, j$$ such that $$\mathsf{row}(k) = \mathsf{to}\text{-}\mathsf{field}(i)$$, $$\mathsf{col}(k) = \mathsf{to}\text{-}\mathsf{field}(j)$$ and $$\mathsf{val}(k) = D(i, j)$$.&#x20;

$$\mathsf{val}$$ maps $$k$$ to each value of the nonzero elements and $$\mathsf{row}$$ and $$\mathsf{col}$$ maps $$k$$ to a corresponding location of it in $$D$$(or, equivalently, to the index of each two subtables in **Algorithm 1**).&#x20;

Above equation can be simplified to:

Equation (2):

$$
D(r_x, r_y) = \sum_{k\in\{0,1\}^{\log m}} \mathsf{val}(k) \cdot \widetilde{\mathsf{eq}}(\mathsf{to}\text{-}\mathsf{bits}(\mathsf{row}(k)), r_x) \cdot \widetilde{\mathsf{eq}}(\mathsf{to}\text{-}\mathsf{bits}(\mathsf{col}(k)), r_y)
$$



Let $$E_{rx}(k) = \widetilde{\mathsf{eq}}(\mathsf{to}\text{-}\mathsf{bits}(\mathsf{row}(k)), r_x)$$ and $$E_{ry}(k) = \widetilde{\mathsf{eq}}(\mathsf{to}\text{-}\mathsf{bits}(\mathsf{col}(k)), r_y)$$ each of which are ($$\log m$$)-variate. We define a PIOP for proving the evaluation of $$D$$ at $$(r_x, r_y)$$. We assume that $$\mathsf{val}$$ is committed to beforehand.

***

1. $$\mathcal{P} \rightarrow \mathcal{V}$$ : $$E_{rx}(k)$$ and $$E_{ry}(k)$$ as oracles.&#x20;
2. $$\mathcal{P} \leftrightarrow \mathcal{V}$$ : run the sumcheck reduction to reduce this statement

$$
v = \sum_{k\in\{0,1\}^{\log m}} \mathsf{val}(k) \cdot E_{rx}(k) \cdot E_{ry}(k)
$$

to the following checks:

* $$\mathsf{val}(r_z) \stackrel{?}{=} v_{val}$$
* $$E_{rx}(r_z) \stackrel{?}{=} v_{E_{rx}}$$ and $$E_{ry}(r_z) \stackrel{?}{=} v_{E_{ry}}$$.&#x20;

where the claimed values are provided by $$\mathcal{P}$$ and $$r_z$$ is randomly sampled by $$\mathcal{V}$$ at the end of the sumcheck protocol.

3. $$\mathcal{V}$$ : check if the three equalities above hold with an oracle query to each of the polynomials.&#x20;

***

A key part of achieving a sound argument with the PIOP above is to enable the verifier to check that the committed $$E_{rx}$$ and $$E_{ry}$$ are built correctly, i.e.&#x20;

* $$\forall k \in \{0,1\}^{\log m}, E_{rx}(k) = \widetilde{\mathsf{eq}}(\mathsf{to}\text{-}\mathsf{bits}(\mathsf{row}(k)), r_x)$$&#x20;
* $$\forall k \in \{0,1\}^{\log m}, E_{ry}(k) = \widetilde{\mathsf{eq}}(\mathsf{to}\text{-}\mathsf{bits}(\mathsf{col}(k)), r_y)$$.

Checking the above is equivalent to checking if an untrusted prover ran **Algorithm 1** above (**which is the case where** $$c = 2$$ **and the two subtables correspond to** $$E_{rx}$$ **and** $$E_{ry}$$), i.e., that the prover initialized the memory with correct values and the subsequent memory read operations occurred in a correct manner. We introduce offline memory checking, which is a protocol that reduces such memory consistency argument to a multiset equality check.&#x20;

***

## Offline memory checking

Offline memory checking is a protocol between an untrusted memory and a trusted checker, which is the prover and the verifier in our protocol's context, respectively.&#x20;

The checker maintains two local state sets : $$\mathsf{RS}$$ and $$\mathsf{WS}$$. $$\mathsf{RS}$$ is initially empty. For $$\mathsf{M}$$-sized memory, $$\mathsf{WS}$$ is initialized as following : for all $$i \in [N^{1/c}]$$, the tuple $$(i, v_i, 0)$$ is included. Here, the first element is the index, second one is the value and the third one is the accessed count of each memory slot.&#x20;

For each read operation at address $$a$$, suppose that the untrusted memory responds with a value-count pair $$(v, t)$$. Then the checker updates its local state as follows:&#x20;

1. $$\mathsf{RS} \leftarrow \mathsf{RS} \cup \{(a, v, t)\}$$
2. store $$(v, t+1)$$ at address $$a$$ in the untrusted memory, and&#x20;
3. $$\mathsf{WS} \leftarrow \mathsf{WS} \cup \{(a,v,t+1)\}$$.&#x20;

#### Claim 1.&#x20;

If for every read operation, the untrusted memory returns the tuple last written to that location, then there exists a set $$\mathsf{S}$$ with cardinality $$\mathsf{M}$$ consisting of tuples of the form $$(k, v_k, t_k)$$ for all $$k \in [\mathsf{M}]$$ such that $$\mathsf{WS} = \mathsf{RS} \cup \mathsf{S}$$, where $$\mathsf{S}$$ is simply the current memory view. Conversely, if the untrusted memory ever returns a value $$v$$ for a memory read $$k \in [\mathsf{M}]$$ which does not equal the value initially written to cell $$k$$, then there does not exist any set $$\mathsf{S}$$ such that $$\mathsf{WS} = \mathsf{RS} \cup \mathsf{S}$$.&#x20;

***

I skip the proof of the claim, but the example below demonstrates how the first direction of the claim holds. Three memory slots with addresses 1, 2 and 3 are initialized with values 100, 101 and 102. Three read operations at address 2, 2 and 3.

<table data-full-width="true"><thead><tr><th>RS</th><th>WS</th><th>Read address 2.</th><th>RS</th><th>WS</th><th>Read address 2.</th><th>RS</th><th>WS</th><th>Read address 3.</th><th>RS</th><th>WS</th><th>S</th></tr></thead><tbody><tr><td></td><td>(1, 100, 0)</td><td></td><td></td><td>(1, 100, 0)</td><td></td><td></td><td>(1, 100, 0)</td><td></td><td></td><td>(1, 100, 0)</td><td>(1, 100, 0)</td></tr><tr><td></td><td>(2, 101, 0)</td><td></td><td>(2, 101, 0)</td><td>(2, 101, 0)<br>(2, 101, 1)</td><td></td><td>(2, 101, 0)<br>(2, 101, 1)</td><td>(2, 101, 0)<br>(2, 101, 1)<br>(2, 101, 2)</td><td></td><td>(2, 101, 0)<br>(2, 101, 1)</td><td>(2, 101, 0)<br>(2, 101, 1)<br>(2, 101, 2)</td><td>(2, 101, 2)</td></tr><tr><td></td><td>(3, 102, 0)</td><td></td><td></td><td>(3, 102, 0)</td><td></td><td></td><td>(3, 102, 0)</td><td></td><td>(3, 102, 0)</td><td>(3, 102, 0)<br>(3, 102, 1)</td><td>(3, 102, 1)</td></tr></tbody></table>

You can see that there exist an $$\mathsf{S}$$ such that $$\mathsf{WS} = \mathsf{RS} \cup \mathsf{S}$$, and the $$\mathsf{S}$$ is equal to the final memory view.&#x20;

***

Now we define the **counter polynomials**. Given the size $$\mathsf{M}$$ memory and a list of $$m$$ addresses involved in read operations, one can compute two vectors $$C_r \in \mathbb{F}^m$$ and $$C_f \in \mathbb{F}^{\mathsf{M}}$$ where $$C_r[k]$$ stores the 'correct' count that would have been returned for $$k$$-th read operation and $$C_f[j]$$ stores the 'correct' final count stored at memory location $$j$$ after the $$m$$ read operations. Computing these requires $$O(m)$$ operations over $$\mathbb{F}$$.&#x20;

Let $$\textsf{read\_ts} = \widetilde{C_r}$$, $$\textsf{write\_cts} = \widetilde{C_r} + 1$$, and $$\mathsf{final\_cts} = \widetilde{C_f}$$. We refer to these polynomials as counter polynomials.&#x20;

### The evaluation proof

#### Claim 2.&#x20;

Regarding a ($$2 \log M$$)-variate multilinear polynomial, suppose that ($$\mathsf{row, col, val}$$) is committed in advance, and

$$
E_{rx}, E_{ry}, \textsf{read\_ts}_{\mathsf{row}}, \textsf{final\_cts}_{\mathsf{row}}, \textsf{read\_ts}_{\mathsf{col}}, \textsf{final\_cts}_{\mathsf{col}}
$$

are the polynomials sent by the prover at the beginning of the evaluation proof. Notice that the verifier can manually compute $$\mathsf{write\_cts}_{\mathsf{row}}$$ and $$\mathsf{write\_cts}_{\mathsf{col}}$$.&#x20;

If $$E_{rx}$$ is correctly computed, i.e.  $$\forall k \in \{0,1\}^{\log m}, E_{rx}(k) = \widetilde{\mathsf{eq}}(\mathsf{to}\text{-}\mathsf{bits}(\mathsf{row}(k)), r_x)$$,&#x20;

then the following holds : $$\mathsf{WS} = \mathsf{RS} \cup \mathsf{S}$$, where&#x20;

* $$\mathsf{WS} = \{(\textsf{to-field}(i), \widetilde{\mathsf{eq}}(i, r_x), 0): i \in \{0,1\}^{\log \mathsf{M}} \} \cup \{ (\mathsf{row}(k), E_{rx}(k), \textsf{write\_cts}_{\mathsf{row}}(k)) : k \in \{0,1\}^{\log m} \} ;$$
* $$\mathsf{RS} = \{(\mathsf{row}(k), E_{rx}(k), \textsf{read\_ts}_{\mathsf{row}}(k)) : k \in \{0,1\}^{\log m}\};$$ and&#x20;
* $$\mathsf{S} = \{(\textsf{to-field}(i), \widetilde{\mathsf{eq}}(i, r_x), \textsf{final\_cts}_{\textsf{row}}(i)): i \in \{0,1\}^{\log {\mathsf{M}}} \}.$$



The high-level conclusion of the memory checking above is that when $$E_{rx}$$ is the polynomial that is produced by an untrusted memory, the checker (verifier) initializes the memory with $$\widetilde{\mathsf{eq}}(i, r_x)$$ which are the expected 'correct' values, and then by checking if the memory lookups are correctly done for each of the $$E_{rx}(k)$$, the verifier is convinced that the provided $$E_{rx}$$ is correct. The same logic is applied for $$E_{ry}$$.&#x20;

In reality, the verifier doesn't perform the check above manually. Instead, to check that $$\mathcal{H}_{\tau, \gamma}(\mathsf{WS}) = \mathcal{H}_{\tau, \gamma}(\mathsf{RS}) \cdot \mathcal{H}_{\tau, \gamma}(\textsf{S})$$, two parties run a **grand product argument**, which is a multiset equality check that is performed via a GKR protocol whose circuit is simply a binary tree of multiplication gates. Since running a GKR protocol is reduced to a single evaluation of the input vector on a random value sampled by the verifier, and the verifier virtually has an oracle access to all of the required polynomials assuming they are committed by the prover (or is able to evaluate by himself), the memory check above can be done efficiently with a few evaluation proofs. For a more detailed explanation on the grand product argument, refer to [Appendix E](https://eprint.iacr.org/2023/1216.pdf#page=37\&zoom=100,96,642) of the paper.&#x20;

***

## Generalization and Specialization

We will generalize the above protocol, $$\mathsf{Spark}$$, so that it supports an arbitrary value for $$c$$ rather than a fixed value of 2, which implies that the gigantic lookup table can be decomposed into even smaller subtables of size $$N^{1/c}$$.&#x20;

Subsequently we will specialize $$\mathsf{Spark}$$ for $$\mathsf{Lasso}$$ with some cost optimization exploiting the properties of lookup arguments. Eventually, we will further generalize $$\mathsf{Spark}$$ by relaxing the conditions that make a table 'structured' for $$\mathsf{Lasso}$$ to be applied. With these modifications altogether, we introduce $$\mathsf{Surge}$$, as a sparse polynomial commitment scheme for $$\mathsf{Lasso}$$ and its final PIOP.&#x20;



1. Supporting $$\log N = c \log m$$ variables, rather than $$2 \log m$$.&#x20;

$$c$$ can be interpreted as the number of the subtables which are the result of decomposition of the original table of size $$N$$. We can simply factor $$\widetilde{\mathsf{eq}}_{c \log \mathsf{m}}$$ into a product of $$c$$ terms, instead of two terms, each of which are a $$(\log m)$$-variate multilinear Lagrange basis polynomial. Hence, from this point, we use the notation of $$\mathsf{dim_i}(x)$$ for $$i \in [c]$$, for representing the injection from a boolean hypercube to an index in each subtable, instead of $$\mathsf{row}(x)$$ and $$\mathsf{col}(x)$$.&#x20;



2. Specializing for Lasso

As we saw in Equation (1), the sparse $$m \times N$$ matrix $$M$$ that we want to commit to can only have value 1 as its non-zero element. It means that the polynomial $$\mathsf{val}(x)$$ is also fixed to 1 and the commitment to it is unnecessary. We can save one polynomial to commit to with this per each lookup argument.&#x20;

Furthermore, if we let $$c = 2$$ and sample $$r_x$$ from $$\mathbb{F}^{\log m}$$ and $$r_y$$ from $$\mathbb{F}^{\log N}$$ (this means that each $$r_x$$ and $$r_y$$ represents the row index and column index of matrix $$M$$), since each row has exactly one non-zero element, $$\textsf{to-bits}(\textsf{row}(k))$$ is simply $$k$$. $$E_{rx}(k) = \widetilde{\mathsf{eq}}(k, r_x)$$ can be evaluated by verifier on its own, thus there is no need for committing to it or proving its well-formness. &#x20;

$$
\begin{bmatrix} 
   0 & 0 & 1 & 0  \\
   0 & 0 & 0 & 1 \\
   1 & 0 & 0 & 0  \\
   1 & 0 & 0 & 0 
   \end{bmatrix} 
\cdot 
\begin{bmatrix} 
   0  \\
   1  \\
   2  \\
   3    \\
   \end{bmatrix}
=   
  \begin{bmatrix} 
   2  \\
   3  \\
   0  \\
   0  \\
   \end{bmatrix}
$$

> $$M$$ is a 4 x 4 matrix with sparsity of 4. Let $$k$$ be a two bits binary string for indexing 4 nonzero elements, i.e. $$k = 00, 01, 10, 11$$. The table below is the description of mapping $$\mathsf{row}$$ and $$\mathsf{col}$$. It is obvious that the mapping $$\mathsf{row}(k)$$ is simply an iteration from 0 to 3, which is virtually equal to $$k$$. This is because there are exactly one nonzero element per each row. Therefore, it is unnecessary to commit to the polynomial $$E_{rx}(k)$$, as well as $$\mathsf{val}(k)$$. Check the table below.

| k  | row(k) | col(k) | val(k) |
| -- | ------ | ------ | ------ |
| 00 | 0      | 2      | 1      |
| 01 | 1      | 3      | 1      |
| 10 | 2      | 0      | 1      |
| 11 | 3      | 0      | 1      |



3. Generalizing the struct of lookup tables&#x20;

We redefine the term 'structured' to be more comprehensive and well-defined for the properties that the lookup tables may have. If (1) a lookup table $$t$$ can be expressed as a tensor product of $$c \geq 2$$ smaller tables (i.e. decomposible), and (2) for each subtable its multilinear extension polynomial can be evaluated quickly (i.e. MLE-structured), we regard such table $$t$$ as **structured** and Lasso can be applied.&#x20;

Suppose that there is an integer $$k \geq 1$$ and $$\alpha = k \cdot c$$ tables $$T_1, ..., T_{\alpha}$$ of size $$N^{1/c}$$ along with an $$\alpha$$-variate multilinear polynomial $$g$$ such that the following holds.&#x20;

$$
T[r] = g(T_1[r_1], ..., T_k[r_1], T_{k+1}[r_2], ..., T_{2k}[r_2], ..., T_{\alpha-k+1}[r_c], ..., T_{\alpha}[r_c]).
$$

It means that each element of $$T$$ can be briefly expressed by each corresponding element in $$T_1, ..., T_{\alpha}$$.&#x20;

***

#### Example.&#x20;

&#x20;A table for a range check in the first example ($$[1, ..., 2^{128} ]$$) can be easily expressed in the way above with $$k = 1$$ and $$c = 32$$, for instance.&#x20;

$$
T = [1, ..., 2^{128}] \\
T_1 = T_2 = ... = T_{32} = [1, ..., 2^4] \\
T[r] = 2^{124} \cdot T_1[r_1] + 2^{120} \cdot T_2[r_2] + ... + T_{32}[r_{32}]
$$

***

Going back, we rewrite the Equation (2) as below:&#x20;

$$
\sum_{j \in \{0,1\}^{\log N}} \widetilde{M}(r, j) \cdot T[j] = \tilde{a}(r)
$$

If and only if $$M \cdot t = a$$, the equation above holds, with a soundness error $$\log m / |\mathbb{F}|$$. In $$\mathsf{Lasso}$$, after committing to $$\widetilde{M}$$(by committing to $$\mathsf{dim_1}, ..., \mathsf{dim_c}$$) and $$\tilde{a}$$, the verifier sends random sampled $$r$$ to see if above holds.&#x20;

&#x20;Let $$\mathsf{nz}(i)$$ denote the unique nonzero column in row $$i$$ of $$M$$. We can rewrite the LHS as following:&#x20;

$$
\sum_{j \in \{0,1\}^{\log N}} \widetilde{M}(r, j) \cdot T[j]\\
= \sum_{i \in \{0,1\}^{\log m}} \widetilde{\mathsf{eq}}(i, r) \cdot T[\mathsf{nz}(i)]
$$

And with the decomposibility of $$T$$,&#x20;

$$
\sum_{i \in \{0,1\}^{\log m}} \widetilde{\mathsf{eq}}(i, r) \cdot T[\mathsf{nz}(i)] \\
= \sum_{i \in \{0,1\}^{\log m}} \widetilde{\mathsf{eq}}(i, r) \cdot g(T_1[\mathsf{dim}_1(i)], ..., T_k[\mathsf{dim}_1(i)], T_{k+1}[\mathsf{dim}_2(i)], ..., T_{2k}[\mathsf{dim}_2(i)], ..., T_{\alpha-k+1}[\mathsf{dim}_c(i)], ..., T_{\alpha}[\mathsf{dim}_c(i)])
$$

Now the prover should prove that&#x20;

* for some $$E_1, ..., E_{\alpha}$$, above is equal to $$\tilde{a}(r)$$, using the sum-check protocol and&#x20;
* $$E_1, ..., E_{\alpha}$$ are built with correct lookups into the subtables $$T_1, ..., T_{\alpha}$$, using the offline memory checking.

Assuming that the verifier can evaluates the MLE of each subtables quickly, both parties can participate in the protocol only with the short descriptions about the subtables, without ever fully materializing the entire lookup table $$T$$ throughout the protocol.&#x20;

Final PIOP is described below. &#x20;

***

$$\mathcal{P}$$ has committed to $$c$$ multilinear polynomials $$\mathsf{dim_1}, ..., \mathsf{dim_c}$$, each over $$\log m$$ variables.&#x20;

1.  $$\mathcal{P} \rightarrow \mathcal{V}$$ : 2$$\alpha$$ different ($$\log m$$)-variate multilinear polynomials $$E_1, ..., E_{\alpha}$$, $$\textsf{read\_ts}_1$$,..., $$\textsf{read\_ts}_{\alpha}$$ and $$\alpha$$ different ($$\log N / c$$)-variate multilinear polynomials $$\textsf{final\_cts}_1$$, ..., $$\textsf{final\_cts}_{\alpha}$$.&#x20;

    // $$E_i$$ is purported to specify the values of each of the $$m$$ reads into $$T_i$$.

    // $$\textsf{read\_ts}$$ and $$\textsf{final\_cts}$$ are the "counter polynomials" for each memory slot of the subtables after the computation.&#x20;
2. $$\mathcal{V}$$ and $$\mathcal{P}$$ apply the sumcheck protocol to the polynomial $$h(k) := \widetilde{\mathsf{eq}}(r,k) \cdot g(E_1(k), ..., E_{\alpha}(k))$$ which reduces the check of the sum of $$h(k)$$ on the boolean hypercube to : $$E_i(r_z) \stackrel{?}{=} v_{E_i}$$ for $$i = 1, ..., \alpha$$.&#x20;
3.  $$\mathcal{V}$$ : check if the above equalities hold with one oracle query to each $$E_i$$.&#x20;

    // Up to this point, the correctness of $$E_i$$ is are not guaranteed yet. $$\mathcal{V}$$ and $$\mathcal{P}$$ runs the offline memory checking to check that $$E_i(j)$$ equals $$T_i[\mathsf{dim}_i(j)]$$ for all $$j \in \{0,1\}^{\log m}$$.&#x20;
4.  $$\mathcal{V} \rightarrow \mathcal{P}$$ : $$\tau, \gamma \in_{R} \mathbb{F}$$.&#x20;

    // In practice, $$c$$ instances of the sumcheck protocol below can be reduced to a single run of a sumcheck applied to random linear combinations of the polynomials.&#x20;
5. $$\mathcal{V} \leftrightarrow \mathcal{P}$$ : For $$i = 1, ..., {\alpha}$$, run a GKR-based grand products protocol to reduce the check $$\mathcal{H}_{\tau, \gamma}(\mathsf{WS}) \stackrel{?}{=} \mathcal{H}_{\tau, \gamma}(\mathsf{RS}) \cdot \mathcal{H}_{\tau, \gamma}(\textsf{S})$$ to the following checks:&#x20;

$$
E_i(r_i') \stackrel{?}{=} v_{E_i} \\
\mathsf{dim}_i(r_i')  \stackrel{?}{=} v_i,  \textsf{read\_ts}_i(r_i')  \stackrel{?}{=} v_{\textsf{read\_ts}}, \textsf{final\_cts}_i(r_i'')  \stackrel{?}{=} v_{\textsf{final\_cts}}
$$

6. $$\mathcal{V}$$ : check the equalities above with an oracle query to each of $$E_i$$, $$\mathsf{dim}_i$$, $$\textsf{read\_ts}_i$$ and $$\textsf{final\_cts}_i$$.&#x20;



* Input : A polynomial commitment to the multilinear polynomials $$\tilde{a} : \mathbb{F}^{\log m} \rightarrow \mathbb{F}$$, and a description of a structured table $$T$$ of size $$N$$.
* &#x20;$$\mathcal{P}$$ sends a $$\textsf{Surge}$$-commitment to the multilinear extension $$\widetilde{M}$$ of a matrix $$M \in \{0,1\}^{m \times N}$$. This consists of $$c$$ different ($$\log m$$)-variate multilinear polynomials $$\mathsf{dim}_1, ..., \mathsf{dim}_c$$.&#x20;
* $$\mathcal{V}$$ picks a random $$r \in \mathbb{F}^{\log m}$$ and sends $$r$$ to $$\mathcal{P}$$. $$\mathcal{V}$$ makes one evaluation query to $$\tilde{a}$$ to learn $$\tilde{a}(r)$$.&#x20;
* $$\mathcal{P}$$ and $$\mathcal{V}$$ apply $$\textsf{Surge}$$, allowing $$\mathcal{P}$$ to prove that $$\sum_{y \in \{0,1\}^{log N}} \widetilde{M}(r,y)\cdot T[y] = \tilde{a}(r)$$.&#x20;

***

## Cost

### Cost of Spark

<figure><img src="../../.gitbook/assets/image (86) (1).png" alt=""><figcaption></figcaption></figure>

It's worth of noting that while at first glance it seems like increasing $$c$$ incurs an increase in the commitment size and verification cost in a factor linear to $$c$$, since most of the PCS have efficient batching properties for evaluation proofs, the factor $$c$$ can be omitted (i.e. the prover and verifier costs for verifying polynomial evaluations do not grow with $$c$$).

### Prover time

The prover can compute its message for the sumcheck protocol with $$O(b \cdot k \cdot \alpha \cdot m)$$ field operations where $$\alpha = k \cdot c$$ and $$b$$ is the number of monomials in $$g$$. For many tables of practical interest, $$b$$ and $$k$$ factor can be eliminated (where $$g$$ has a 1 or 2 total degree, which is also the case of the range check example above), which gives $$O(c \cdot m)$$ total cost.&#x20;

For the offline memory checking argument, the costs for the prover is similar to $$\mathsf{Spark}$$ : $$O(\alpha \cdot m + \alpha \cdot N^{1/c})$$ field operations plus committing to a low-order number of field elements.&#x20;

### Verifier cost

The verifier performs $$O(k \cdot \log m)$$ field operations. The costs of the memory checking argument, which can be batched, are identical to $$\mathsf{Spark}$$.

***

### References

* [https://eprint.iacr.org/2023/1216.pdf](https://eprint.iacr.org/2023/1216.pdf)
* [https://people.cs.georgetown.edu/jthaler/ProofsArgsAndZK.pdf](https://people.cs.georgetown.edu/jthaler/ProofsArgsAndZK.pdf)&#x20;
* [https://eprint.iacr.org/2019/550.pdf](https://eprint.iacr.org/2019/550.pdf)&#x20;

> Written by [Carson Lee](https://app.gitbook.com/u/Hm5RHrPlu2fbxwXnpUgxmvXVTwB3 "mention")
