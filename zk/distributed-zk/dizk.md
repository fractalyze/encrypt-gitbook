# DIZK

## Introduction

Generating a zero-knowledge proof (ZKProof) is computationally expensive and involves significant memory overhead. DIZK addresses this by leveraging [Apache Spark](https://spark.apache.org/) to generate [Groth16](../snark/groth16.md) proofs across multiple machines in a distributed cluster environment.

## Background

Please read [**Distributed FFT**](../../primitives/number-theoretic-transform/parallel-distributed-fft.md)**!**

## Protocol Explanation

<figure><img src="../../.gitbook/assets/image (170).png" alt=""><figcaption></figcaption></figure>

### Distributed Setup

In the Groth16 proving system, the setup phase is critical for generating the proving and verification keys. The most computationally intensive part of this process is evaluating the QAP polynomials $$(a_i(X), b_i(X), c_i(X))$$ at a specific point $$x$$. DIZK is designed to perform this operation efficiently in a **distributed environment using** [**Spark**](https://spark.apache.org/).

#### Step 1: Distributed Lagrange Evaluation

As part of the QAP instance reduction, we must evaluate the Lagrange basis polynomials at the point $$x$$:

$$
L_i(X) = \frac{1}{n} \cdot \frac{\omega^i}{X - \omega^i} \cdot (X^n - 1)
$$

Each executor computes $$L_i(x)$$ for its assigned index in parallel, and stores the results as an [Resilient Distributed Dataset (RDD)](https://spark.apache.org/docs/latest/rdd-programming-guide.html#resilient-distributed-datasets-rdds).

#### Step 2: QAP Instance Reduction

* **Input:** $$A \in \mathbb{F}^{n \times (m + 1)}$$, $$x \in \mathbb{F}$$
* **Output:** $$\left(a_i(x)\right)_{i=0}^{m}$$, where $$a_i(x) := \sum_{j=0}^{n-1} A_{j,i} \cdot L_j(x)$$

***

**Naive (Strawman) Approach:**

1. Join $$A_{j,i}$$ with $$L_j(x)$$ by index $$j$$
2. Map each pair $$(A_{j,i}, L_j(x))$$ to the product $$A_{j,i} \cdot L_j(x)$$
3. Reduce by $$i$$ to compute $$\left( a_i(x) = \sum_{j=0}^{n-1} A_{j,i} \cdot L_j(x) \right)_{i=0}^m$$

This join operation would produce a structure like:

| $$(A_{0,0}, L_0(x))$$       | $$(A_{0,1}, L_0(x))$$       | $$\cdots$$ | $$(A_{0,m}, L_0(x))$$       |
| --------------------------- | --------------------------- | ---------- | --------------------------- |
| $$(A_{1,0}, L_1(x))$$       | $$(A_{1,1}, L_1(x))$$       | $$\cdots$$ | $$(A_{1,m}, L_1(x))$$       |
| $$\vdots$$                  | $$\vdots$$                  | $$\ddots$$ | $$\vdots$$                  |
| $$(A_{n-1,0}, L_{n-1}(x))$$ | $$(A_{n-1,1}, L_{n-1}(x))$$ | $$\cdots$$ | $$(A_{n-1,m}, L_{n-1}(x))$$ |

However, the actual matrix $$A$$ is only **almost sparse**. For example:

| $$(991, L_0(x))$$   | $$(681, L_0(x))$$      | $$\cdots$$ | $$(517, L_0(x))$$   |
| ------------------- | ---------------------- | ---------- | ------------------- |
| $$(0, L_1(x))$$     | $$(2476, L_1(x))$$     | $$\cdots$$ | $$(0, L_1(x))$$     |
| $$\vdots$$          | $$\vdots$$             | $$\ddots$$ | $$\vdots$$          |
| $$(0, L_{n-1}(x))$$ | $$(8629, L_{n-1}(x))$$ | $$\cdots$$ | $$(0, L_{n-1}(x))$$ |

As a result, certain executors—such as those handling the first row in the example—become overloaded, causing **out-of-memory errors or straggler tasks**.

***

To address this imbalance, Spark offers techniques such as `blockjoin` and `skewjoin`.

* **Blockjoin** replicates the smaller RDD (typically $$L_j(x)$$) across all executors so that tasks can be executed evenly. This results in a join structure like:

| $$(A_{0,0}, L_0(x))$$       | $$(A_{0,1}, L_0(x))$$       | $$\cdots$$ | $$(A_{0,m}, L_0(x))$$      |
| --------------------------- | --------------------------- | ---------- | -------------------------- |
| $$\vdots$$                  | $$\vdots$$                  | $$\ddots$$ | $$\vdots$$                 |
| $$(A_{0,0}, L_{n-1}(x))$$   | $$(A_{0,1}, L_{n-1}(x))$$   | $$\cdots$$ | $$(A_{0,m}, L_{n-1}(x))$$  |
| $$\vdots$$                  | $$\vdots$$                  | $$\ddots$$ | $$\vdots$$                 |
| $$(A_{n-1,0}, L_0(x))$$     | $$(A_{n-1,1}, L_0(x))$$     | $$\cdots$$ | $$(A_{n-1,m}, L_0(x)$$     |
| $$\vdots$$                  | $$\vdots$$                  | $$\ddots$$ | $$\vdots$$                 |
| $$(A_{n-1,0}, L_{n-1}(x))$$ | $$(A_{n-1,1}, L_{n-1}(x))$$ | $$\cdots$$ | $$(A_{n-1,m}, L_{n-1}(x)$$ |

Then each machine is assigned the set of pairs $$\left(A_{i, 0}, L_i(x)\right)_{i=0}^{n-1}$$. While this approach ensures even data distribution, it significantly increases memory usage for the executor performing the join.

{% hint style="warning" %}
[Carson Lee](https://app.gitbook.com/u/Hm5RHrPlu2fbxwXnpUgxmvXVTwB3 "mention") mentioned that while this approach resolves the join imbalance issue, it may also include unnecessary data in the final result.
{% endhint %}

* **Skewjoin** identifies skewed keys and splits them across multiple partitions, replicating the corresponding RDD entries. However, since each $$A_{j,i}$$ is used only once, this ends up being equivalent to a naive join, offering no real benefit.

***

#### Solution: Hybrid Join Strategy

**Phase 1: Identify Dense Rows**

* Tag any row with more than $$\sqrt{n}$$ non-zero entries as a **dense row**
* Let $$J_A$$ be the set of all such dense row indices

**Phase 2: Execute Hybrid Join**

* **Dense rows** $$( j \in J_A )$$:
  * Join $$A_{j,i}$$ with $$L_j(x)$$
  * Map each pair to $$A_{j,i} \cdot L_j(x)$$
* **Sparse rows** $$(j \notin J_A)$$:
  * Same join and map as strawman
* **Final aggregation:**
  * Union both results: $$(A_{j,i} \cdot L_j(x)) = (A_{j,i} \cdot L_j(x))_{j \in J_A} \cup (A_{j,i} \cdot L_j(x))_{j \notin J_A}$$
  * Reduce by $$i$$ to compute each $$\left( a_i(x) = \sum_{j=0}^{n-1} A_{j,i} \cdot L_j(x) \right)_{i=0}^m$$

#### Step 3: Distributed fixMSM

In the final part of the setup, the evaluated $$a_i(x)$$ values must be converted into group elements on an elliptic curve. This is done using **Fixed-base Multi-Scalar Multiplication (fixMSM)**.

{% hint style="info" %}
In `fixMSM` , given a vector of scalars $$\bm{s} \in \mathbb{F}^n$$ and a group element $$P \in \mathbb{G}$$, the goal is to compute the vector $$\bm{s} \cdot P \in \mathbb{G}^n$$, where each entry is the scalar multiple $$s_i \cdot P$$.&#x20;
{% endhint %}

1. **Lookup Table Generation:** Precompute scalar multiples of a fixed base point $$P$$:

$$
( P, 2P, 4P, \dots, 2^kP )
$$

2. **Broadcast:** Distribute this lookup table to all executors
3. **Parallel Evaluation:** Each executor uses the lookup table to compute scalar multiplications for its assigned chunk

This method aligns well with Spark's execution model, ensuring balanced workload across executors. By precomputing and broadcasting the table, DIZK avoids redundant computation and achieves both **memory and runtime efficiency**.

### Distributed Prover

In Groth16, the prover takes as input the proving key $$\mathsf{pk}$$, public input $$x \in \mathbb{F}^k$$, and witness $$w \in \mathbb{F}^{m-k}$$, and generates the proof $$\pi$$. This process consists of two main stages:

1. Generate the QAP witness vector $$\bm{h}$$ based on $$x$$ and $$w$$
2. Construct the proof $$\pi$$ by combining $$x, w, \bm{h}$$ with $$\mathsf{pk}$$ through linear combinations

#### Step 1: QAP Witness $$\bm{h}$$ Generation

In Groth16’s QAP-based construction, the witness $$\bm{h}$$ corresponds to the coefficient vector of the rational function $$h(X)$$, defined as:

$$
h(X) = \frac{\left( \sum_{i=0}^m a_i(X) z_i \right) \cdot \left( \sum_{i=0}^m b_i(X) z_i \right) - (\sum_{i=0}^m c_i(X) z_i)}{t(X)}
$$

* $$t(X)$$ is a vanishing polynomial over domain $$D$$, carefully chosen so that it does not vanish over an auxiliary domain $$D'$$.
* The numerator consists of three composite polynomial terms, which are computationally intensive to evaluate.

To compute this efficiently, DIZK performs the following steps:

1. Evaluate the numerator polynomials over the domain $$D$$
2. Convert the evaluations to coefficients via inverse FFT
3. Re-evaluate over a disjoint domain $$D'$$ via forward FFT
4. Perform component-wise division by $$t(X)$$, followed by interpolation to recover $$h(X)$$

These FFT operations are implemented using [**Distributed FFT**](../../primitives/number-theoretic-transform/parallel-distributed-fft.md).

***

The most computationally demanding part is the evaluation of the first polynomial term:

$$
a_z(X) = \sum_{i=0}^m a_i(X) \cdot z_i
$$

* **Input:** $$A \in \mathbb{F}^{n \times (m + 1)}$$, $$\bm{z \in \mathbb{F}^{m+1}}$$
* **Output:** $$\left( \sum_{i=0}^m A_{j,i} \cdot z_i \right)_{j=0}^{n-1}$$

***

**Naive (Strawman) Approach:**

1. Join $$A_{j,i}$$ with $$z_i$$ on key $$i$$
2. Map each pair $$(A_{j,i}, z_i)$$ to the product $$a_{j,i} \cdot z_i$$
3. Reduce by $$j$$ to compute $$\left( \sum_{i=0}^m A_{j,i} \cdot z_i \right)_{j=0}^{n-1}$$

This results in the same type of computational skew encountered during QAP instance reduction in the setup phase. However, the key difference here is that the bottleneck occurs across **columns** rather than **rows**.

Fortunately, the matrix $$A$$'s sparsity structure is fixed for a given circuit, even though the witness vector $$\bm{z}$$ changes with each input. Thus, DIZK can **reuse the sparsity analysis from the setup phase** to optimize this join.

***

**Solution: Hybrid Join Strategy**

**Phase 1: Identify Dense Columns**

* Mark any column with $$\ge \sqrt{n}$$ non-zero entries as a **dense column**
* Let $$I_A$$ denote the set of all such column indices

**Phase 2: Hybrid Join Execution**

* **Dense columns** ($$i \in I_A$$):
  * Join $$A_{j,i}$$ with $$z_i$$ on key $$i$$
  * Map to $$A_{j,i} \cdot z_i$$
* **Sparse columns** ($$i \notin I_A$$):
  * Same join and map as strawman
* **Merge and Reduce:**
  * Combine both outputs: $$(A_{j,i} \cdot z_i) = (A_{j,i} \cdot z_i)_{i \in I_A} \cup (A_{j,i} \cdot z_i)_{i \notin I_A}$$
  * Reduce by $$j$$ to obtain $$\left( \sum_{i=0}^m A_{j,i} \cdot z_i \right)_{j=0}^{n-1}$$

#### Step 2: Distributed varMSM

In the final step of proof generation, the prover performs an inner product between a scalar vector and a vector of elliptic curve base points. DIZK distributes this **Variable-base multi-scalar multiplication (varMSM)** using [**Pippenger’s algorithm**](../../primitives/abstract-algebra/elliptic-curve/msm/pippengers-algorithm.md), which is well-suited for this scenario.

{% hint style="info" %}
In `varMSM`, given a vector of scalars $$\bm{s} \in \mathbb{F}^n$$ and a vector of group elements $$\bm{P} \in \mathbb{G}^n$$, the goal is to compute the sum $$\sum_{i=1}^n{s_i \cdot P_i} \in \mathbb{G}$$.&#x20;
{% endhint %}

1. **Chunk Distribution:** The set of scalar-base point pairs $$(s_i, P_i)$$ is partitioned evenly across a set of executors $$S_j$$
2. **Local Computation:** Each executor computes a partial result using Pippenger's locally:

$$
Q_j = \sum_{i \in S_j} s_i \cdot P_i
$$

3. **Global Reduction:** The driver aggregates the partial results from each executor:

$$
Q = \sum_j Q_j
$$

This design allows DIZK to scale to large witness sizes and high-throughput proof generation with minimal overhead and excellent parallelism.

## Conclusion

<figure><img src="../../.gitbook/assets/image (171).png" alt=""><figcaption></figcaption></figure>

As shown in the figure above, we observe that increasing the number of executors allows the system to support larger instance sizes.

<figure><img src="../../.gitbook/assets/image (172).png" alt=""><figcaption></figcaption></figure>

From the graph above, we can observe the following:

* **When the number of executors is fixed**, the setup time and prover time increase linearly with the instance size.
* **When the instance size is fixed**, the setup time and prover time decrease linearly as the number of executors increases.

DIZK shows that zkSNARKs, which were traditionally constrained by memory and computational limits, can be effectively scaled using distributed computing frameworks like Apache Spark. By parallelizing key components such as FFT, Lagrange evaluation, and MSM, DIZK enables proof generation for circuits with billions of constraints, significantly expanding the practical applicability of zero-knowledge proofs. Its modular design also provides reusable distributed primitives that can benefit other proof systems like STARKs.

## References

* [https://eprint.iacr.org/2018/691](https://eprint.iacr.org/2018/691)
* [https://www.youtube.com/watch?v=dJn6bb\_gjoA](https://www.youtube.com/watch?v=dJn6bb_gjoA)

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze
