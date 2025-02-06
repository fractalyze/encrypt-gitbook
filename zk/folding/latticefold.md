---
description: >-
  This article aims to intuitively explain the goals and processes of the
  LatticeFold protocol.
---

# LatticeFold

## Introduction

[**LatticeFold**](https://eprint.iacr.org/2024/257) is **the first lattice-based folding scheme**. Folding schemes require a **Homomorphic Additive Commitment Scheme**, which is why Pedersen commitments are commonly used. These elliptic curve-based commitments are vulnerable to quantum computing attacks, require large-sized fields, and rely on non-native field arithmetic for folding verification. In contrast, LatticeFold uses **Ajtai commitments** based on the **Module SIS problem**, which is known to be quantum-resistant, supports small-sized fields, and is cost-efficient to verify.

## Background

### Folding Schemes: Motivation and Challenges

A circuit is typically expressed as $$(x; w)$$, where the public input x and witness w represent a certain relation $$\mathcal{R}$$. For example, a circuit representing $$w^2 = x$$ could have relations such as $$(9; 3)$$ and $$(16; 4)$$ satisfying it. While public inputs act as constraints on the relation, since witnesses must be kept hidden, a ZK SNARK proof ensures the relation is held accordingly.

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXd7lFQXL0Qgqzcy5fguy0I73HB4BLI6FN8GBzq6LNgIIDYAeEAhHd2ayTvMs45VxQ1nrdA45Nyc4TLFXNldXWeCh7Yc1tQmP6q2CRXQHyYz8xYIv4yn8wdol5lUDNXJdmfOzimk?key=fr56y9m2lAA1NJbFGYUN2Zfo" alt=""><figcaption><p>Fig 1. State machine with multiple steps</p></figcaption></figure>

As shown in Fig. 1, some cases require expressing sequences of computations. For example, $$(2^{16}; 2^8),(2^8; 2^4), (2^4; 2^2)$$ and $$(2^2; 2)$$ represent the four steps in computing the square root of $$2^{16}$$. Such computational sequences are common, with zkEVMs and zkVMs being other notable examples.

The computational expense of generating a SNARK proof for $$(x; w)$$ is a major challenge. This inefficiency is compounded when there are multiple instances satisfying the relation, as generating separate proofs for each instance is costly. For example, creating two separate SNARK proofs for $$(x_1; w_1)$$ and $$(x_2; w_2)$$, both satisfying the same relation $$\mathcal{R}$$, can be prohibitively expensive.

In comparison, a folding scheme creates a single new pair $$(x_3; w_3)$$ from $$(x_1; w_1)$$ and $$(x_2; w_2)$$ that also satisfies the relation $$\mathcal{R}$$. Then, the SNARK proof for the folded $$(x_3; w_3)$$ not only validates the new relation but also proves the relations for the original pairs $$(x_1; w_1)$$ and $$(x_2; w_2)$$. Generating and verifying folding proofs is faster and cheaper than doing the same with SNARK proofs. Therefore, instead of recursively generating and verifying SNARK proofs, the process is optimized by using folding proofs for intermediate steps before creating a final SNARK proof. The example shown here demonstrates 2-to-1 folding, but some folding schemes support n-to-1 folding.\\

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXd_YRlnMoqOxkXt0VWvW6Dg_KmC9YFo7YWe-W5Ljc5HYoi3wKBFeTtvF0AgFTAJSTa2bGEb2nFMb5zpUqRycfo0LatalMtdopLM-cNoOifMfx9_MYoCuwRG8wzkr9FmRlc0dmFW?key=fr56y9m2lAA1NJbFGYUN2Zfo" alt=""><figcaption><p>Fig 2. Proof generation using recursion</p></figcaption></figure>

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfEUdKRDQj-J9iX51CxSBP9UaMo_McmM6lGbf4fxLzO17Fpb7f8olnuxvMlknW6hkUzmosY5RebctP3xcIKiV3C2gA_eckOUycXYf_ykO31oUCMJ-KYDssLE4DgHq7rINKHRuA?key=fr56y9m2lAA1NJbFGYUN2Zfo" alt=""><figcaption><p>Fig 3. Proof generation using folding scheme</p></figcaption></figure>

Folding generally employs a technique called **Random Linear Combination (RLC**) where values are combined together with a random value. To ensure succinctness and to hide the witness, the prover uses commitments of the witness to apply RLC. This means that the commitment must support **additive homomorphism**.

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXc8uFRsd5JHe1MfwVYZFhrsG0vy3T40HOzQXEijN8tZxXw4EeQsrJEQWRbJ0Za32unyidBacSf5ifGLcj8CYNMVB3P_1-Bt6vmINci-viyml_EBVnPsB8kXucfVrUHn3tUXGt5t?key=fr56y9m2lAA1NJbFGYUN2Zfo" alt=""><figcaption><p>Fig 4. 2-to-1 folding example</p></figcaption></figure>

Since FRI-based STARKs use Merkle Trees as commitments, which do not satisfy the additive homomorphism property, folding schemes cannot be used in FRI-based STARK systems.

### Module SIS problem

A **lattice** refers to a combination of points following a repetitive pattern, defined as:

$$
L(\mathbf{b}_1, \mathbf{b}_2, \dots, \mathbf{b}_n) = \left\{ \sum_{i=1}^n x_i \mathbf{b}_i : x_i \in \mathbb{Z} \right\}
$$

Here, a set of $$\{b_1, b_2, \dots, b_n\} \in \mathbb{R}^d$$ is called the **basis**, $$n$$ is the **rank**, and $$d$$ is the **dimension**, satisfying $$n \le d$$. The basis vectors must be linearly independent, defined as:

$$
x_1b_1 + x_2b_2 + \cdots + x_nb_n = 0 \implies x_1 = x_2 = \cdots = x_n = 0
$$

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXf3E4M_wKVpQuUPsORP-7CTR3irTvKeDuoYRrVGro9v_8ouhIN43ufrpOLIMsJyQFtj4u930gex4wL6ucPWwLoJHyxqm5eZAk4PZxnPqgYEYWkw7RqdPACmXme_HT1PPLrNXQg0?key=fr56y9m2lAA1NJbFGYUN2Zfo" alt=""><figcaption><p>Fig 5. Examples of various bases, source:<a href="https://www.esat.kuleuven.be/cosic/blog/introduction-to-lattices/"> https://www.esat.kuleuven.be/cosic/blog/introduction-to-lattices/</a></p></figcaption></figure>

A given lattice can be created with different bases. One notable problem in lattices is the **Shortest Vector Problem (SVP)**, which asks for the shortest vector that can be formed given a basis. For instance:

$$
\vec{b}_1 = (64, 218, 133), \quad \vec{b}_2 = (71, 205, 111) \quad \vec{b}_3 = (28, -48, -84).
\\
\text{Lattice } = \{ x_1\vec{b}_1 + x_2\vec{b}_2 + x_3\vec{b}_3 \mid x_1, x_2, x_3 \in \mathbb{Z} \}.
$$

Using the [$$L^2$$-norm](https://en.wikipedia.org/wiki/Norm_\(mathematics\)#Euclidean_norm) of all the basis vectors, the solution to the SVP for this basis is found to be $$(1,3,−1)$$ formed by $$x_1 =−322, x_2=323, x_3=−83$$ ([Reference](https://youtu.be/AMqdppb-oPU?t=56)). This problem becomes significantly harder with poorly chosen bases and higher dimensions, posing challenges even for quantum computers.

The **Short Integer Solution (SIS)** problem was introduced in[ Ajtai’s 1999 paper](https://people.csail.mit.edu/vinodv/CS294/ajtai99.pdf), defined as:

Given a uniformly random integer matrix $$A \in \mathbb{Z}_q^{\kappa \times m}$$ and $$B \in \mathbb{Z}$$, find a non-zero integer vector ($$\mathbf{x} \in \mathbb{Z}^m$$) such that $$A \cdot \mathbf{x} = 0 \in \mathbb{Z}^{\kappa}_q$$ and $$0 < \|\mathbf{x}\| \le B$$.

Ajtai demonstrated a **worst-case-to-average-case reduction**, proving that solving SIS implies solving SVP. The key insight here is that the random matrix $$A$$ allows uniform sampling of bases, retaining the complexity of hard lattice problems and making them practical for cryptographic applications. This was a pivotal step in advancing lattice-based cryptography.

The **Module SIS (M-SIS)** with $$B$$ problem, a generalization of SIS, serves as the foundation for LatticeFold. It is defined as:

Given $$R = \mathbb{Z}[X] / (X^d + 1)$$ and $$R_q = R / qR$$, with a uniformly random matrix $$A \in R^{\kappa \times m}_q$$ and $$B \in \mathbb{R}_q$$, find a non-zero vector $$\mathbf{x} \in R^m_q$$ such that $$A \cdot \mathbf{x} = 0 \in R^\kappa _q$$ and $$0 < \|\mathbf{x} \| \le B$$. Here, $$X^d + 1$$ is a cyclotomic polynomial ([Reference](https://en.wikipedia.org/wiki/Cyclotomic_polynomial)), specifically one where $$d$$ is a power of 2.

### Ajtai Commitment

In LatticeFold, the matrix $$A$$ and witness vector $$\mathbf{w}$$ create a commitment , which is called an Ajtai commitment. For a commitment scheme, the following properties are required:

* **Hiding**: The original message cannot be inferred from the commitment. This is satisfied if $$A$$ is randomly sampled.
* **Binding**: Each commitment maps to one original message with high probability. If the same commitment is created from two different $$x_1$$ and $$x_2$$, it is equivalent to solving the M-SIS with $$2B$$, which is probabilistically very difficult.

$$
A \cdot \mathbf{x}_1 = A \cdot \mathbf{x}_2 \text{ for } \mathbf{x}_1 \ne \mathbf{x}_2 \leftrightarrow
A \cdot (\mathbf{x}_1 - \mathbf{x}_2) = 0 \text{ where } 0 < \| \mathbf{x}_1 - \mathbf{x}_2 \| \le 2B
$$

* **Compression**: The commitment size is smaller than the original message. This is achieved if $$\kappa < m$$.

Additionally, unlike SIS, M-SIS leverages rings instead of fields, introducing challenges in proving **knowledge soundness**. Specifically, not all ring elements are invertible, and even if they are, their norms can become excessively large, necessitating solutions to address this.

The **Ajtai commitment** supports **additive homomorphism**, making it suitable for folding schemes, which use random linear combinations. To preserve the binding property, the norm of the random linear combination of two witness vectors must stay within a certain boundary $$B$$. If performed naively like shown below, the norm may become excessively large, exceeding $$B$$.

$$
A \cdot \mathbf{w_1} + A \cdot \alpha \cdot \mathbf{w_2} = A \cdot (\mathbf{w_1} + \alpha \cdot \mathbf{w_2})\text{, } \text{where } 0 < \|\mathbf{w_1} + \alpha \cdot \mathbf{w_2}\| < (1 + \alpha) \cdot B
$$

In LatticeFold, the $$L ^{\infin}$$(infinity norm) is used for norm calculations. For a vector $$\mathbf{v}=[v_1,v_2,\dots,v_n]$$, the infinity norm is defined as:

$$
\|\mathbf{v}\|_\infty = \max_{i} |v_i|
$$

From this point onward, we use the infinity norm even without an infinity symbol.

## Protocol Explanation

LatticeFold is divided into three steps: **Expansion**, **Decomposition**, and **Fold**.

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfWYL4Mkj5q736qHcPvU2SdytRkSHjcssD_rSOjTVdgZLA2-Rh0Obu0VQ5PysmRIapQPIP0HFv29ubWLQgihTd42wD3vNk_Av1gfHVGdIgVRL4mOTba1mC2oWLBmL0CZZ86f112?key=fr56y9m2lAA1NJbFGYUN2Zfo" alt=""><figcaption><p>Fig 6. LatticeFold folding process, where comp stands for computation and acc stands for accumulation</p></figcaption></figure>

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfeUrQJs_ee4DFUemKh7BKM4WBkd8jmSOjSDhLWGpVLK2W9mUGnVdbDz14hUK7to11eWXwBIKQsUaa2bbEJak1mD9ol6nK99c6ncEp-cmq2Lv1xj0fsvE12XrCcZNrW9IncScgK?key=fr56y9m2lAA1NJbFGYUN2Zfo" alt=""><figcaption></figcaption></figure>

This image above is taken from [the LatticeFold paper](https://eprint.iacr.org/2024/257). As shown above, we first define the multilinear extension (MLE), and then define the following two relations.

$$
\mathcal{R}_{comp}^B = \{(A \cdot w; w):  A \in R_q^{\kappa \times m} \land w \in R_q^m \land 0 \le \|w\| < B \ \}   \\

\mathcal{R}_{acc}^B = \Big\{ 
\begin{aligned}
    &(A\cdot w, r, v; w)
\end{aligned}
:
\begin{aligned}
    &(A\cdot w; w) \in \mathcal{R}_{comp}^B \\
    & r \in R_q^{\log m} \land v \in  R_q  \wedge \mathsf{mle}[w](r) = v
\end{aligned}

\Big\}
$$

The difference between the two relations lies with $$(r, v)$$. For simplicity, the public value $$x$$ will be omitted.

### Expansion

In the Expansion step, $$\mathcal{R}_{comp}^B$$ ​ is transformed into $$\mathcal{R}_{acc}^B$$ using the zero vector. This transformation is necessary because, in the **Fold** step, the structure must conform to the format of $$\mathcal{R}_{acc}$$.

$$
(A \cdot w; w) \rightarrow (A \cdot w, \vec{0}, \mathsf{mle}[w](\vec{0}); w)
$$

### Decomposition

Remember that this takes 2 $$\mathcal{R}_{acc}^B$$ as input and produces $$2k \times \mathcal{R}_{acc}^b$$​. See Fig 6. above

$$
2 \times (A \cdot w_i, \vec{r}_i, \mathsf{mle}[w](\vec{r}_i); w_i) \rightarrow 2k \times (A \cdot w_j, \vec{r}_j, \mathsf{mle}[w_j](\vec{r}_j); w_j)
$$

The Decomposition proceeds through the steps below for each relation:

1. The prover provides $$cm_0, \dots, cm_{k-1}, v_0, \dots, v_{k-1}$$ to the verifier, satisfying the following conditions:

$$
w = \sum_{i = 0}^{k-1}b^i\cdot w_i\text{, where } 0 \le \|w_i\| < b \\
cm_i = A \cdot w_i \\
v_i = \mathsf{mle}[w_i](r)
$$

2. The verifier checks the following conditions

$$
A\cdot w \stackrel{?}= \sum_{i=0}^{k-1}b^i \cdot cm_i \space \land v \stackrel{?}= \sum_{i = 0}^{k-1}b^i\cdot v_i
$$

If the conditions above are satisfied, each $$\mathcal{R}_{acc}^B$$ is transformed into $$k$$ instances of $$\mathcal{R}_{acc}^b$$.

$$
(A \cdot w, \vec{r}, \mathsf{mle}[w](\vec{r}); w) \rightarrow k \times (A \cdot w_i, \vec{r}, \mathsf{mle}[w_i](\vec{r}); w_i)
$$

By applying this to the $$2 \times \mathcal{R}_{acc}^B$$, the number increases from $$2 \times \mathcal{R}_{acc}^B$$ to $$2k \times \mathcal{R}_{acc}^b$$​. This is done to restrict the norm size to $$b$$, which should be smaller than $$B$$, preventing the norm from growing excessively during the **Fold** step. However, note that the range check for each $$w_i$$ has not yet been performed.

### Fold

To verify that the $$w_i$$​ values from the previous step lie within the range $$(−b, b)$$, the following polynomial will be used.

$$
P(X) = \prod_{i = -(b-1)}^{b- 1}(X - i)
$$

If a value is sampled within $$(−b, b)$$ and evaluated with $$P(X)$$, the result must be 0 for the range check of $$w_i$$ to succeed (This is used in Step 2 below.).

As mentioned earlier, it is also crucial to sample random values effectively. For this purpose, we sample from $$C_{small}$$, known as a strong sampling set. Even when multiplied by $$ho$$ sampled from this set, the norm increases by at most a factor of $$C$$, known as the expansion factor. If $$C \cdot b \cdot 2k < B$$, the norm of the folded witnesses will remain less than $$B$$. It can be formally described as:

$$
\|\sum_{i=0}^{2k-1}\rho_i \cdot w_i\| = \sum_{i=0}^{2k-1}\|\rho_i \cdot w_i\| \le \sum_{i=0}^{2k-1} C \cdot \| w_i\| \le \sum_{i=0}^{2k-1} C \cdot b< B
$$

This takes $$2k \times \mathcal{R}_{acc}^b$$ as input and produces $$\mathcal{R}_{acc}^B$$.

$$
2k \times (A \cdot w_i, \vec{r}_i, \mathsf{mle}[w_i](\vec{r}); w_i) \rightarrow (A \cdot w', \vec{r}_{out}, \mathsf{mle}[w'](\vec{r}_{out}); w'), \\
\text{where } w' = \sum_{i=0}^{2k-1}\rho_i \cdot w_i
$$

The Fold proceeds through the steps below:

1. The verifier samples $$\alpha_0, \dots, \alpha_{2k-1}, \mu_0, \dots, \mu_{2k - 1}, \beta$$ and sends them to the prover.
2. The prover performs the sumcheck protocol to prove the following. The $$g_{norm}$$ term is constrained to compute as 0 for all $$b$$ in $$\{0,1\}^{\log⁡ m}$$.

$$
\sum_{\vec{x} \in \{0, 1\}^{\log m}} g(\vec{x}) = \sum_{i=0}^{2k-1} \alpha_i \cdot v_i \\
\text{where } g(\vec{x}) = g_{eval}(\vec{x}) + g_{norm}(\vec{x}) \\
g_{eval}(\vec{x}) = \sum_{i=0}^{2k-1}\alpha_i \cdot \mathsf{eq}(\vec{r}_i, \vec{x})\cdot \mathsf{mle}[w_i](\vec{x}) \\
g_{norm}(\vec{x}) = \sum_{i=0}^{2k-1}\mu_i \cdot \mathsf{eq}(\vec{\beta}, \vec{x})\cdot \prod_{j = -(b - 1)}^{b -1}(\mathsf{mle}[w_i](\vec{x}) - j)
$$

3. At the end of the sumcheck protocol, the following evaluation claim is obtained:

$$
g(\vec{r}_{out}) \stackrel{?}= s
$$

4. The prover sends $$\theta_0, \dots, \theta_{2k - 1}$$ to the verifier, where:

$$
\theta_i = \mathsf{mle}[w_i](\vec{r}_{out})
$$

5. The verifier checks the following condition:

$$
s \stackrel{?}= \sum_{i = 0}^{2k - 1}\alpha_i \cdot \mathsf{eq}(\vec{r}_i, \vec{r}_{out})\cdot \theta_i + \mu_i \cdot \mathsf{eq}(\vec{\beta}, \vec{r}_{out})\cdot \prod_{j = -(b - 1)}^{b-1}(\theta_i - j)
$$

6. The verifier samples $$ho_0, \dots, \rho_{2k-1}$$ from $$C_{small}$$ and sends them to the prover. Using these, a new relation is created.

$$
(A \cdot w', \vec{r}_{out}, \mathsf{mle}[w'](\vec{r}_{out}); w')\text{, where } w' = \sum_{i=0}^{2k-1}\rho_i \cdot w_i
$$

## Conclusion

Although not covered in detail here, the paper also discusses the use of extension fields to enable smaller field sizes and methods to apply this approach to [HyperNova](https://eprint.iacr.org/2023/573)’s [CCS](https://eprint.iacr.org/2023/552.pdf). However, some aspects of LatticeFold limit its application to Protostar's, leaving it an open problem. The significance of LatticeFold lies in it being the first to apply lattice-based cryptography to folding schemes, paving the way for incorporating lattice-based folding schemes into STARKs.

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from A41
