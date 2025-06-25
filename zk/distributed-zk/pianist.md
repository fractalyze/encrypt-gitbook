# Pianist

## Introduction

**Pianist (Plonk vIA uNlimited dISTribution)** is a **distributed proof generation system** based on Plonk.

## Protocol Explanation

### Notation

* $$M$$: number of machines
* $$N$$: size of the entire circuit
* $$T$$: size of each sub-circuit, where $$T = \frac{N}{M}$$, a power of 2

### Arithmetic Constraint System for Each Party

Each participant is responsible for a portion $$C_i$$ of the global circuit $$C$$, and constructs a local constraint system to prove correctness of that sub-circuit.

This system consists of two main parts:

1. **Gate Constraint** — ensures that gate computations are correct
2. **Copy Constraint** — ensures that wire connections are consistent

***

#### Gate Constraint

Plonk targets **fan-in 2 arithmetic circuits**, where each gate has at most two inputs (left and right) and one output.

For each sub-circuit $$C_i$$ assigned to machine $$P_i$$, the following **univariate polynomials** are defined for a circuit size $$T$$:

* $$a_i(X)$$: left input
* $$b_i(X)$$: right input
* $$o_i(X)$$: output

Each of these is expressed in the **Lagrange basis** as:

$$
a_i(X) = \sum_{j=0}^{T-1} a_{i,j} L_j(X)
$$

where $$L_j(X)$$ is a Lagrange polynomial, and can be computed in $$O(T \log T)$$ time using the [Number Theoretic Transform (NTT)](../../primitives/number-theoretic-transform.md).

All gate operations are encoded in a single polynomial equation:

$$
g_i(X) := q_{a,i}(X)a_i(X) + q_{b,i}(X)b_i(X) + q_{o,i}(X)o_i(X) + q_{ab,i}(X)a_i(X)b_i(X) + q_{c,i}(X)
$$

The $$q$$ coefficients vary depending on the gate type:

*   **Addition Gate**

    $$
    (q_{a,i}, q_{b,i}, q_{o,i}, q_{ab,i}, q_{c,i}) = (1, 1, -1, 0, 0)
    \quad \Rightarrow \quad g_i(X) = a_i(X) + b_i(X) - o_i(X)
    $$
*   **Multiplication Gate**

    $$
    (q_{a,i}, q_{b,i}, q_{o,i}, q_{ab,i}, q_{c,i}) = (0, 0, -1, 1, 0)
    \quad \Rightarrow \quad g_i(X) = a_i(X) b_i(X) - o_i(X)
    $$
*   **Public Input Gate**

    $$
    (q_{a,i}, q_{b,i}, q_{o,i}, q_{ab,i}, q_{c,i}) = (0, 0, -1, 0, \mathsf{in}_{i,j})
    \quad \Rightarrow \quad g_i(X) = -o_i(X) + q_{c,i}(X)
    $$

Finally, the gate constraint must hold at all evaluation points in the domain:

$$
g_i(X) = 0 \quad \forall X \in \Omega_X
$$

where $$\Omega_X = \{ \omega_X^0, \dots, \omega_X^{T-1} \}$$.

***

#### Copy Constraint

One of the core ideas in Plonk is modeling **wire consistency** using a **permutation argument**. To enable this, the verifier sends two random field elements $$\eta, \gamma \in \mathbb{F}$$. The prover then constructs the following **running product polynomial**:

$$
z_i(\omega_X^j) = 
\begin{cases} 
1 & \text{if } j = 0 \\
\prod_{k=0}^{j-1} \frac{f_i(\omega_X^k)}{f'_i(\omega_X^k)} & \text{otherwise}
\end{cases}
$$

Where:

*   $$f_i(X)$$: actual wiring values

    $$
    f_i(X) := (a_i(X) + \eta \cdot \sigma_{a,i}(X) + \gamma)(b_i(X) + \eta \cdot \sigma_{b,i}(X) + \gamma)(o_i(X) + \eta \cdot \sigma_{c,i}(X) + \gamma)
    $$
*   $$f'_i(X)$$: permuted reference wiring

    $$
    f'_i(X) := (a_i(X) + \eta k_A X + \gamma)(b_i(X) + \eta k_B X + \gamma)(o_i(X) + \eta k_O X + \gamma)
    $$

Here, $$k_A = 1$$, and $$k_B, k_O$$ are distinct **non-residue constants**.

{% hint style="success" %}
**What are non-residue constants?**\
In general, when constructing permutation or lookup arguments over a finite field $$\mathbb{F}$$, non-residue constants are used to apply distinct "shifts". Specifically, if $$\omega$$ is a primitive $$n$$-th root of unity generating a multiplicative subgroup $$H \subset \mathbb{F}$$, then any $$k \notin H$$ is considered a **non-residue**, or a **coset shift**. One can then define a new **coset domain** $$kH = \{ kh \mid h \in H \}$$ that is disjoint from $$H$$.
{% endhint %}

The prover must prove the following holds:

$$
z_i(\omega_X^T) = \prod_{k=0}^{T-1} \frac{f_i(\omega_X^k)}{f'_i(\omega_X^k)} = 1
$$

This is enforced via the following constraints:

$$
p_{i,0}(X) := L_0(X) \cdot (z_i(X) - 1)
$$

$$
p_{i,1}(X) := z_i(X) \cdot f_i(X) - z_i(\omega_X \cdot X) \cdot f'_i(X)
$$

***

#### Quotient Polynomial

To consolidate all constraints into a single polynomial relation, we prove the existence of a polynomial $$h_i(X)$$ satisfying:

$$
g_i(X) + \lambda \cdot p_{i,0}(X) + \lambda^2 \cdot p_{i,1}(X) = V_X(X) \cdot h_i(X)
$$

where the vanishing polynomial over the evaluation domain is defined as:

$$
V_X(X) = X^T - 1
$$

### Constraint System for Data-parallel Circuit

This section presents the **core technique** that aggregates locally proven gate and copy constraints from each machine into a **single unified SNARK proof**.

Assuming the sub-circuits $$C_0, C_1, \dots, C_{M-1}$$ are independent, the goal is to:

* **Aggregate local proof polynomials into a single bivariate polynomial**, and
* **Transform the distributed proof system into a globally verifiable SNARK**

***

#### Naive Approach

The most straightforward idea is to aggregate sub-polynomials using powers of $$Y$$:

$$
A(Y, X) := \sum_{i=0}^{M-1} Y^i a_i(X)
$$

A multiplication gate would then be expressed as:

$$
A(Y, X) B(Y, X) - C(Y, X)
= \left( \sum_{i=0}^{M-1} Y^i a_i(X) \right)
  \left( \sum_{i=0}^{M-1} Y^i b_i(X) \right)
  - \left( \sum_{i=0}^{M-1} Y^i c_i(X) \right)
$$

This introduces **cross-terms** such as $$Y^{i+j} a_i(X) b_j(X)$$, making it difficult to reason about and distribute the proof efficiently.

***

#### Bivariate Lagrange Polynomial Aggregation

To address this, inspired by [Caulk](https://eprint.iacr.org/2022/621), we use Lagrange basis polynomials in $$Y$$:

$$
A(Y, X) := \sum_{i=0}^{M-1} R_i(Y) a_i(X)
$$

Now, a multiplication gate becomes:

$$
A(Y, X) B(Y, X) - C(Y, X)
= \left( \sum_{i=0}^{M-1} R_i(Y) a_i(X) \right)
  \left( \sum_{i=0}^{M-1} R_i(Y) b_i(X) \right)
  - \left( \sum_{i=0}^{M-1} R_i(Y) c_i(X) \right)
$$

This removes any cross-terms and allows clean aggregation of constraints.

***

#### Global Constraint Definitions

* **Gate Constraint**:

$$
G(Y, X) := Q_a(Y, X) A(Y, X)
+ Q_b(Y, X) B(Y, X)
+ Q_o(Y, X) O(Y, X)
+ Q_{ab}(Y, X) A(Y, X) B(Y, X)
+ Q_c(Y, X)
$$

* **Copy Constraint**:

$$
P_0(Y, X) := L_0(X) \cdot (Z(Y, X) - 1)
$$

$$
P_1(Y, X) := Z(Y, X)
\prod_{S \in \{A, B, O\}} (S(Y, X) + \eta \cdot \sigma_s(Y, X) + \gamma)
- Z(Y, \omega_X X)
\prod_{S \in \{A, B, O\}} (S(Y, X) + \eta \cdot k_s X + \gamma)
$$

* **Quotient Polynomial**:

$$
G(Y, X) + \lambda P_0(Y, X) + \lambda^2 P_1(Y, X) = V_X(X) \cdot H_X(Y, X)
$$

***

#### Verifier Strategy

The verifier checks that for every $$i \in [0, M-1]$$, the constraint holds at $$Y = \omega_Y^i$$:

$$
G(Y, X) + \lambda P_0(Y, X) + \lambda^2 P_1(Y, X) - V_X(X) H_X(Y, X) = V_Y(Y) H_Y(Y, X)
$$

where $$V_Y(Y) = Y^M - 1$$ is the vanishing polynomial for the $$Y$$-domain.

### Constraint System for General Circuit

In the **data-parallel setting**, there are no connections between sub-circuits, so each machine can independently prove its portion. However, real-world circuits such as those in zkEVMs have **highly interconnected gates**.

In this **general setting**, the same wire value may appear across multiple sub-circuits, requiring a **cross-machine copy constraint**. To support this in a distributed setting, we must modify the permutation argument. The key idea is to include not just the position within a sub-circuit (X), but also the machine index (Y).

***

#### Modified Copy Constraint

The global permutation check requires each machine to construct a running product that satisfies:

$$
\prod_{i=0}^{M-1} \prod_{j=0}^{T-1} \frac{f_i(\omega_X^j)}{f'_i(\omega_X^j)} \stackrel{?}{=} 1
$$

Where:

*   $$f_i(X)$$: actual wire expression

    $$
    \begin{aligned}
    f_i(X) = & (a_i(X) + \eta_Y \delta_{Y,a,i}(X) + \eta_X \delta_{X,a,i}(X) + \gamma) \\
             & (b_i(X) + \eta_Y \delta_{Y,b,i}(X) + \eta_X \delta_{X,b,i}(X) + \gamma) \\
             & (o_i(X) + \eta_Y \delta_{Y,o,i}(X) + \eta_X \delta_{X,o,i}(X) + \gamma)
    \end{aligned}
    $$
*   $$f'_i(X)$$: sorted reference wiring

    $$
    \begin{aligned}
    f'_i(X) = & (a_i(X) + \eta_Y Y + \eta_X k_A X + \gamma) \\
              & (b_i(X) + \eta_Y Y + \eta_X k_B X + \gamma) \\
              & (o_i(X) + \eta_Y Y + \eta_X k_O X + \gamma)
    \end{aligned}
    $$

Now, the final value of the running product for each machine,

$$
z_i^* = z_i(\omega_X^{T-1}) \cdot \frac{f_i(\omega_X^{T-1})}{f'_i(\omega_X^{T-1})}
$$

is no longer guaranteed to be 1. Thus, the constraints must be modified as follows:

$$
p_{i,0}(X) := L_0(X) \cdot (z_i(X) - 1)
$$

$$
p_{i,1}(X) := (1 - L_{T-1}(X)) \cdot (z_i(X) f_i(X) - z_i(\omega_X X) f'_i(X))
$$

The master node then collects the final values from each machine $$(z_0^*, \dots, z_{M-1}^*)$$ and checks:

$$
\prod_{i=0}^{M-1} \prod_{j=0}^{T-1} \frac{f_i(\omega_X^j)}{f'_i(\omega_X^j)} = \prod_{i=0}^{M-1} z_i^* \stackrel{?}{=} 1
$$

To do this, a new polynomial $$z_{\mathsf{last}}(X)$$ is constructed as:

$$
z_{\mathsf{last}}(\omega_Y^i) = \omega_i =
\begin{cases}
1 & \text{if } i = 0 \\
\prod_{k=0}^{i-1} z_k^* & \text{otherwise}
\end{cases}
$$

This leads to the following constraints:

$$
p_{i,2}(X) := \omega_0 - 1
$$

$$
p_{i,3}(X) := L_{T-1}(X) \cdot (\omega_i z_i(X) f_i(X) - \omega_{(i+1) \mod M} f_i'(X))
$$

***

#### Quotient Polynomial

All constraints are combined into a single relation by proving the existence of $$h_i(X)$$ such that:

$$
g_i(X) + \lambda p_{i,0}(X) + \lambda^2 p_{i,1}(X) + \lambda^3 p_{i,2}(X) = V_X(X) \cdot h_i(X)
$$

where $$V_X(X) = X^T - 1$$ is the vanishing polynomial for the X-domain.

***

#### Global Constraint Definitions

* **Copy Constraints**:

$$
\begin{aligned}
P_0(Y, X) &:= L_0(X) \cdot (Z(Y, X) - 1) \\
P_1(Y, X) &:= (1 - L_{T-1}(X)) \cdot \left[ Z(Y, X) F(Y, X) - Z(Y, \omega_X X) F'(Y, X) \right] \\
P_2(Y) &:= R_0(Y) \cdot (W(Y) - 1) \\
P_3(Y, X) &:= L_{T-1}(X) \cdot \left[ W(Y) Z(Y, X) F(Y, X) - W(\omega_Y Y) F'(Y, X) \right]
\end{aligned}
$$

Here, $$F(Y, X)$$ and $$F'(Y, X)$$ are defined as:

$$
F(Y, X) := \prod_{S \in \{A, B, O\}} (S(Y, X) + \eta_Y \delta_{Y,S}(Y, X) + \eta_X \delta_{X,S}(Y, X) + \gamma)
$$

$$
F'(Y, X) := \prod_{S \in \{A, B, O\}} (S(Y, X) + \eta_Y Y + \eta_X k_S X + \gamma)
$$

* **Quotient Polynomial**: All constraints are bundled into a single bivariate relation using $$H_Y(Y, X)$$:

$$
G(Y, X) + \lambda P_0(Y, X) + \lambda^2 P_1(Y, X) + \lambda^3 P_2(Y) + \lambda^4 P_3(Y, X) - V_X(X) H_X(Y, X) = V_Y(Y) H_Y(Y, X)
$$

### Polynomial IOP for Data-parallel and General Circuits

1. $$\mathcal{P} \to \mathcal{V}$$: Send oracles for $$\{A(Y, X), B(Y, X), C(Y, X)\}$$
2. $$\mathcal{V} \to \mathcal{P}$$: Send random challenges $$\textcolor{orange}{\eta_Y}, \eta_X, \gamma$$
3. $$\mathcal{P} \to \mathcal{V}$$: Send oracles for $$\{Z(Y, X), \textcolor{orange}{W(Y)}\}$$
4. $$\mathcal{V} \to \mathcal{P}$$: Send random challenge $$\lambda$$
5.  $$\mathcal{P} \to \mathcal{V}$$: Send oracle for $$H_X(Y, X)$$

    $$
    H_X(Y, X) = \sum_{i=0}^{M-1} R_i(Y) \cdot \frac{g_i(X) + \lambda p_{i,0}(X) + \lambda^2 p_{i,1}(X) + \textcolor{orange}{\lambda^4 p_{i,3}(X)}}{V_X(X)}
    $$
6. $$\mathcal{V} \to \mathcal{P}$$: Send random challenge $$\alpha$$
7.  $$\mathcal{P} \to \mathcal{V}$$: Send oracle for $$H_Y(Y, \alpha)$$

    $$
    H_Y(Y, \alpha) = \frac{G(Y, \alpha) + \lambda P_0(Y, \alpha) + \lambda^2 P_1(Y, \alpha) + \textcolor{orange}{\lambda^3 P_2(Y) + \lambda^4 P_3(Y, \alpha)} - V_X(\alpha) H_X(Y, \alpha)}{V_Y(Y)}
    $$
8.  $$\mathcal{V}$$ queries all oracles at $$X = \alpha, Y = \beta$$ and checks:

    $$
    G(\beta, \alpha) + \lambda P_0(\beta, \alpha) + \lambda^2 P_1(\beta, \alpha) + \textcolor{orange}{\lambda^3 P_2(\beta) + \lambda^4 P_3(\beta, \alpha)} - V_X(\alpha) H_X(\beta, \alpha) \stackrel{?}{=} V_Y(\beta) H_Y(\beta, \alpha)
    $$

> 🔶 The terms marked in orange are only required in the **General Circuit** case.

The soundness of this protocol is formally proven in [Appendix A](https://eprint.iacr.org/2023/1271.pdf#page=15\&zoom=100,72,513).

***

#### Communication Analysis

Except for $$H_Y(Y, X)$$ and $$W(Y)$$, all other polynomials can be distributed across $$M$$ machines, with only their commitments needing to be sent.

* For $$W(Y)$$, it can be computed by the master node using the final values $$(z_0^*, \dots, z_{M-1}^*)$$ received from all machines.
*   For $$H_Y(Y, X)$$, the prover does not need to send the full polynomial. Instead, the prover collects evaluations at $$X = \alpha$$ from each machine and reconstructs $$H_Y(Y, \alpha)$$. For example:

    $$
    A(Y, \alpha) = \sum_{i=0}^{M-1} R_i(Y) \cdot a_i(\alpha)
    $$

Therefore, the overall communication complexity is $$O(M)$$.

***

#### Proving Time Analysis

* Each prover $$P_i$$ computes $$z_i(X)$$ and $$h_i(X)$$ in $$O(T \log T)$$
* The master node $$P_0$$ computes $$W(Y)$$ and $$H_Y(Y, \alpha)$$ in $$O(M \log M)$$
* So the maximum latency per machine is $$O(T \log T + M \log M)$$
* The total proving time across all machines is $$O(N \log T + M \log M)$$

This is a practical improvement over the original Plonk prover time complexity of $$O(N \log N)$$.

<figure><img src="../../.gitbook/assets/image (117).png" alt=""><figcaption></figcaption></figure>

### Distributed KZG

The original KZG commitment scheme is designed for **univariate polynomials**. Since Pianist models the entire proof system as a **bivariate polynomial** $$f(Y, X)$$, we need to extend KZG to support multiple variables. Furthermore, due to the **distributed nature** of the system, each machine must compute only a portion of the commitment and proof, while the **master node assembles the full result**.

Suppose we have the following bivariate polynomial:

$$
f(Y, X) = \sum_{i=0}^{M-1} \sum_{j=0}^{T-1} f_{i,j} R_i(Y) L_j(X)
\quad \text{(where } f_{i,j} = f_i(\omega^j) \text{)}
$$

Each machine holds a slice $$f_i(X)$$ defined as:

$$
f_i(X) = \sum_{j=0}^{T-1} f_{i,j} L_j(X)
$$

***

#### $$\mathsf{DKZG.KeyGen}(1^\lambda, M, T)$$

Generates the public parameters $$\mathsf{pp}$$:

$$
\mathsf{pp} = \left(
[1]_1, 
\left(\left( [R_i(\tau_Y) L_j(\tau_X)]_1 \right)_{i=0}^{M-1}\right) _{j=0}^{T-1}, 
[1]_2, 
[\tau_X]_2, 
[\tau_Y]_2 
\right)
$$

***

#### $$\mathsf{DKZG.Commit}(f, \mathsf{pp})$$

Each prover $$\mathcal{P}_i$$ computes:

$$
\mathsf{com}_{f_i} = \sum_{j=0}^{T-1} f_{i,j} \cdot [R_i(\tau_Y) L_j(\tau_X)]_1
$$

The master node $$\mathcal{P}_0$$ aggregates:

$$
\mathsf{com}_f = \sum_{i=0}^{M-1} \mathsf{com}_{f_i} = [f(\tau_Y, \tau_X)]_1
$$

***

#### $$\mathsf{DKZG.Open}(f, \beta, \alpha, \mathsf{pp})$$

1.  Each $$\mathcal{P}_i$$ computes:

    * $$f_i(\alpha)$$
    * $$q_0^{(i)}(X) = \frac{f_i(X) - f_i(\alpha)}{X - \alpha}$$
    * $$\pi_0^{(i)} = [R_i(\tau_Y) \cdot q_0^{(i)}(\tau_X)]_1$$

    Then sends $$f_i(\alpha), \pi_0^{(i)}$$ to $$\mathcal{P}_0$$
2. $$\mathcal{P}_0$$ computes:
   * $$\pi_0 = \sum_{i=0}^{M-1} \pi_0^{(i)}$$
   * $$f(Y, \alpha) = \sum_{i=0}^{M-1} R_i(Y) f_i(\alpha)$$
3.  Then computes:

    * $$f(\beta, \alpha)$$
    * $$q_1(Y) = \frac{f(Y, \alpha) - f(\beta, \alpha)}{Y - \beta}$$
    * $$\pi_1 = [q_1(\tau_Y)]_1$$

    Sends $$(z = f(\beta, \alpha), \pi_f = (\pi_0, \pi_1))$$ to $$\mathcal{V}$$

***

#### $$\mathsf{DKZG.Verify}(\mathsf{com}_f, \beta, \alpha, z, \pi_f, \mathsf{pp})$$

The verifier $$\mathcal{V}$$ parses $$\pi_f = (\pi_0, \pi_1)$$ and checks:

$$
e(\mathsf{com}_f - [z]_1, [1]_2) \stackrel{?}{=} e(\pi_0, [\tau_X - \alpha]_2) \cdot e(\pi_1, [\tau_Y - \beta]_2)
$$

**Proof:**

* The exponent in $$e(\mathsf{com}_f - [z]_1, [1]_2)$$ is $$f(\tau_Y, \tau_X) - f(\beta, \alpha)$$
*   The exponent in $$e(\pi_0, [\tau_X - \alpha]_2)$$ is:

    $$
    \sum_{i=0}^{M-1} R_i(\tau_Y) \cdot (f_i(\tau_X) - f_i(\alpha)) = f(\tau_Y, \tau_X) - f(\tau_Y, \alpha)
    $$
*   The exponent in $$e(\pi_1, [\tau_Y - \beta]_2)$$ is:

    $$
    f(\tau_Y, \alpha) - f(\beta, \alpha)
    $$

### Robust Collaborative Proving System for Data-parallel Circuit

Pianist introduces a new proof model to ensure security even in **malicious settings**.

This is called the **Robust Collaborative Proving System (RCPS)** — a protocol that ensures the final proof can still be correctly constructed and verified **even when some machines are faulty or adversarial**.

> 📌 For technical details, see [Section 5 of the paper](https://eprint.iacr.org/2023/1271.pdf#page=8\&zoom=100,420,728).

## Conclusion

<figure><img src="../../.gitbook/assets/image (114).png" alt=""><figcaption></figcaption></figure>

While standard Plonk (on a single machine) hits a memory limit at **32 transactions**, Pianist can scale up to **8192 transactions** simply by increasing the number of machines. With 64 machines, Pianist is able to prove 8192 transactions in just **313 seconds**. This demonstrates that **the number of transactions included in a single proof scales proportionally with the number of machines**. Moreover, the time taken by the master node to gather and finalize the proof is just **2–16 ms**, which is **negligible compared to the overall proving time**. This highlights that **communication overhead in Pianist's parallel architecture is minimal**.

***

<figure><img src="../../.gitbook/assets/image (115).png" alt=""><figcaption></figcaption></figure>

While Figure 1 shows the performance on **data-parallel circuits (e.g., zkRollups)**, Figure 2 extends the evaluation to **general circuits with arbitrary subcircuit connections**. In all circuit sizes tested, **the proving time decreases nearly linearly with the number of machines**, demonstrating **strong scalability even in general circuits**. In smaller circuits, the marginal benefit of scaling beyond 4–8 machines may diminish, but for larger circuits, **the benefits of parallelization become increasingly significant**.\
In other words, Pianist **performs even better on larger circuits**, showcasing a desirable scalability property.

***

<figure><img src="../../.gitbook/assets/image (116).png" alt=""><figcaption></figcaption></figure>

As shown above, **the memory usage per machine decreases significantly as the number of machines increases**. For small circuits, memory savings are minor since the total memory requirement is already low, but for large circuits, **scaling the number of machines directly determines whether proof generation is even feasible**.

***

**In conclusion**, Pianist transforms the original Plonk proving system into a fully distributed, scalable framework that:

* Enables **parallel proving of large circuits across multiple machines**,
* Maintains **constant proof size, verifier time, and communication overhead (all** $$O(1)$$**)**, and
* Offers **high practical utility for real-world zk systems like zkRollups and zkEVMs**.

## References

* [https://eprint.iacr.org/2023/1271](https://eprint.iacr.org/2023/1271)

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from A41
