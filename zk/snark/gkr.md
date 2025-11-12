---
description: 'Presentation: https://youtu.be/OTCxQ9qIzDY'
---

# GKR

## Overview

Although interactive proofs like [Sumcheck](../../primitives/sumcheck.md) enabled the verifier to efficiently check the validity of solutions to problems in complexity classes beyond NP, real world computing entities cannot solve a large instance of problems beyond NP practically.&#x20;

Still, it can be pretty useful regarding more solvable problems in the real world (even for the verifier) **if the verifier's cost is much less than solving it by itself** and the **prover's work is not increased asymptotically** compared to what it takes to merely solve it.&#x20;

Let $$\mathcal{C}$$ be the circuit in layered form whose number of gates (or circuit size) is $$S$$ and circuit depth is $$d$$. The GKR protocol is important in a sense that :&#x20;

* $$\mathcal{P}$$-cost is _almost_ $$O(S)$$; it is not much more than the cost to solve the circuit.
* $$\mathcal{V}$$-cost is $$O(d \log S)$$; it's even less than just reading through the entire circuit.&#x20;
  * Note: This needs the assumption that the circuit is [logspace uniform](https://en.wikipedia.org/wiki/Circuit_complexity#Logspace_uniform), which implies that it has a succinct implicit description. That's why $$\mathcal{V}$$ can verify even without looking into the entire $$\mathcal{C}$$.&#x20;
* It can function as a general-purpose protocol for any $$\mathcal{P}$$ and $$\mathcal{V}$$ to run an IP on any arithmetic circuit evaluation instance.

The protocol starts from $$\mathcal{P}$$ sending the claimed output value to $$\mathcal{V}$$ as the first message.&#x20;

In a high level view, the protocol iterates from the first layer (output layer) to the bottom (input layer), where in each iteration $$i$$ the goal is to **reduce the claim of the output values of the gates in layer** $$i$$ **to the claim of the output values of the gates in layer** $$i+1$$.

Going through all the way down, the first claim is reduced to that of input values, which are known to $$\mathcal{V}$$ and thus can be checked by itself. Reduction in each iteration is done by the [sumcheck protocol](../../primitives/sumcheck.md).&#x20;

<figure><img src="../../.gitbook/assets/image (174).png" alt=""><figcaption></figcaption></figure>

> When a circuit is in layered form, it means every leaf appears in the last layer (at depth $$d$$). For a circuit that is not layered, there is a transformation with a blowup in size of factor $$d$$.&#x20;

***

## Protocol details

Let $$\mathcal{C}$$ be a fan-in 2 circuit over a finite field $$\mathbb{F}$$ in a layered form, with

* $$S$$ : the size of the circuit (number of total gates)
* $$S_i$$ : the number of gates in layer $$i$$ (hence each gates in layer $$i$$ is labeled by $$0, \dots, S_i - 1$$)
* $$d$$ : the depth
* $$n$$ : the input length (i.e. $$S_d$$)&#x20;

and each gate is either addition or multiplication.

Without loss of generality let $$S_i = 2^{k_i}$$. Let $$W_i : \{0,1\}^{k_i} \rightarrow \mathbb{F}$$ to be a mapping which takes as input the binary representation of gate labels in layer $$i$$ and gives the gate output.&#x20;

Define "wiring predicate" $$\mathsf{in}_{1,i}, \mathsf{in}_{2,i} : \{0,1\}^{k_i} \rightarrow \{0,1\}^{k_{i+1}}$$, which takes as input the gate label and gives the label of one of its input gates in the lower layer.&#x20;

Lastly, define "gate switch" $$\mathsf{add}_i, \mathsf{mult}_i : \{0,1\}^{k_i + 2k_{i+1}} \rightarrow \{0,1\}$$ which takes as input the in/out gate labels and gives its instruction type. For example, supposing that gate $$a$$ in layer $$i$$ is an addition gate,

$$
\mathsf{add}_i(a,b,c) = 
\begin{cases} 
1 & \text{if } (b,c) = (\mathsf{in}_{1,i}(a), \mathsf{in}_{2,i}(a)) 
\\ 0 & \text{otherwise} 
\end{cases}
$$

Functions with a tilde on top of the function name indicates the multilinear extension of the mapping.

(TODO: Make polynomial extension page and set backlinks to it in pages [Spartan](spartan/), [Lasso](../lookup/lasso.md), etc.)

<figure><img src="../../.gitbook/assets/image (175).png" alt=""><figcaption></figcaption></figure>

### Lemma 1

The multilinear extension of the gate output polynomial can be represented as below.&#x20;

$$
\widetilde{W}_i(z) = \sum_{b,c \in \{0,1\}^{k_{i+1}}} \widetilde{\mathsf{add}}_i(z,b,c)(\widetilde{W}_{i+1}(b) +\widetilde{W}_{i+1}(c)) + \widetilde{\mathsf{mult}}_i(z,b,c)(\widetilde{W}_{i+1}(b) \cdot \widetilde{W}_{i+1}(c))
$$

#### Proof.&#x20;

Obviously both hands match at $$\{0,1\}^{k_i}$$. As both hands are multilinear in $$z$$ and a multlinear extension of such mapping is uniquely defined at domain $$\{0,1\}^{k_i}$$, both hands are identical.

<figure><img src="../../.gitbook/assets/image (176).png" alt=""><figcaption><p>Circuit over <span class="math">\mathbb{F}_5</span> with only multiplication gates. </p></figcaption></figure>

### Protocol

1. $$\mathcal{P}$$ sends a function $$D : \{0,1\}^{k_0} \rightarrow \mathbb{F}$$ claimed to equal $$W_0$$ (the function that maps output gate labels to output values).&#x20;
2. $$\mathcal{V}$$ picks a random $$r_0 \stackrel{\$}{\leftarrow} \mathbb{F}^{k_0}$$, and lets $$m_0 \leftarrow \widetilde{D}(r_0)$$. The remainder of the protocol is devoted to confirming that $$m_0 = \widetilde{W}_0(r_0)$$.&#x20;
3. Now iterate for $$i = 0, 1, \dots, d-1$$. Each iteration is devoted to reduce the claim on $$\widetilde{W}_i(r_i)$$ to the claim on $$\widetilde{W}_{i+1}(r_{i+1})$$.&#x20;
   1. Define the $$(2k_{i+1})$$-variate polynomial:$$f_{r_i}^{(i)}(b,c) = \widetilde{\mathsf{add}}_i(r_i, b, c) \left( \widetilde{W}_{i+1}(b) + \widetilde{W}_{i+1}(c) \right) + \widetilde{\mathsf{mult}}_i(r_i, b, c) \left( \widetilde{W}_{i+1}(b) \cdot \widetilde{W}_{i+1}(c) \right)$$
   2. $$\mathcal{P}$$'s claim is identical to that $$\sum_{b, c \in \{0,1\}^{k_{i+1}}} f_{r_i}^{(i)}(b, c) = m_i$$.
   3. So that $$\mathcal{V}$$ may check this claim, $$\mathcal{P}$$ and $$\mathcal{V}$$ apply the sumcheck protocol to $$f_{r_i}^{(i)}$$, up until $$\mathcal{V}$$'s final check in that protocol, where $$\mathcal{V}$$ must evaluate $$f_{r_i}^{(i)}$$ at a randomly chosen point $$(b^*, c^*) \in \mathbb{F}^{k_{i+1}} \times \mathbb{F}^{k_{i+1}}$$.&#x20;
      1. Note that $$\mathcal{V}$$ only needs to know $$\widetilde{\mathsf{add}}_i$$ and $$\widetilde{\mathsf{mult}}_i$$ and not the entire polynomial $$f_{r_i}^{(i)}$$.
   4. Now we need to reduce the two claims on $$\widetilde{W}_{i+1}(b^*)$$ and $$\widetilde{W}_{i+1}(c^*)$$ to one claim. Let $$\ell$$ be the unique line satisfying $$\ell(0) = b^*$$ and $$\ell(1) = c^*$$. $$\mathcal{P}$$ sends a univariate polynomial $$q$$ of degree at most $$k_{i+1}$$ to $$\mathcal{V}$$, claimed to equal $$\widetilde{W}_{i+1}$$ restricted to $$\ell$$.&#x20;
   5. $$\mathcal{V}$$ now performs the final check in the sumcheck protocol, using $$q(0)$$ and $$q(1)$$ in place of $$\widetilde{W}_{i+1}(b^*)$$ and $$\widetilde{W}_{i+1}(c^*)$$.&#x20;
      1. Note we assume here that for each layer $$i$$, $$\mathcal{V}$$ can evaluate the multilinear extensions $$\widetilde{\mathsf{add}}_i$$ and $$\widetilde{\mathsf{mult}}_i$$ in polylogarithmic time. This is necessary for keeping the verifier cost sublinear to circuit size $$S$$.&#x20;
   6. $$\mathcal{V}$$ chooses $$r^* \in \mathbb{F}$$ at random and sets $$r_{i+1} \leftarrow \ell(r^*)$$ and $$m_{i+1} \leftarrow q(r_{i+1})$$.
      1. Note that $$q(r_{i+1}) = \widetilde{W}_{i+1}(r_{i+1})$$. The claim is successfully reduced to that of the next layer.&#x20;
4. After the final round, $$\mathcal{V}$$ directly checks $$m_d = \widetilde{W}_d(r_d)$$. As $$\mathcal{V}$$ knows the input vector, it can evaluate $$\widetilde{W}_d$$ at any point with $$O(n)$$ time complexity where $$n = 2^{k_d}$$, i.e. the length of input.

<figure><img src="../../.gitbook/assets/image (177).png" alt=""><figcaption></figcaption></figure>

For a more detailed explanation and analysis on costs and soundness, please refer to [Section 4.1 and 4.2 of the original paper](https://people.cs.georgetown.edu/jthaler/gkrnotes.pdf).

In particular, the prover cost is reduced from $$O(S^3)$$ of the naive method of sumcheck protocol to $$O(S^2)$$ of the optimized method which exploits the fact that $$\widetilde{\mathsf{add}}_i$$ and $$\widetilde{\mathsf{mult}}_i$$ are sparse and structured and thus can be evaluated very efficiently. That is, as most of the evaluations are guaranteed to be zero, instead of passing over the entire evaluation domain, it passes only over the values that are known to be nonzero. Furthermore, the prover cost can be reduced to $$O(S)$$ when the computation can be parallelized by data, as shown in the next section.

Similarly in $$\mathcal{V}$$'s cost analysis, the cost for evaluating $$\widetilde{\mathsf{add}}_i$$ and $$\widetilde{\mathsf{mult}}_i$$ is considered to be lower order and omitted. This owes to repeatedly structured wiring patterns that commonly arise in circuits of real world problems in interest. Even without structured wiring patterns, we also have an option to let the verifier itself commit to $$\widetilde{\mathsf{add}}_i$$ and $$\widetilde{\mathsf{mult}}_i$$ (with $$O(S)$$ preprocessing time) using polynomial commitment scheme and let the prover open it each time in need, which turns the interactive proof to an interactive argument.&#x20;

## Leveraging Data Parallelism&#x20;

When the circuit has the data parallelism property; that is, the same sub-computation is applied independently on different pieces of data and then possibly aggregated, the performance of GKR is greatly improved. Fortunately, data parallel computation is pervasive in real-world computing, including the use cases in ZK era such as [LogUp-GKR](../lookup/logup-gkr.md), [Lasso](../lookup/lasso.md), [Libra](https://eprint.iacr.org/2019/317.pdf) and [Ceno](https://eprint.iacr.org/2024/387.pdf).

<figure><img src="../../.gitbook/assets/image (178).png" alt=""><figcaption></figcaption></figure>

Let $$\mathcal{C}$$ be a circuit of size $$S$$ with an arbitrary wiring pattern, and let $$\mathcal{C}'$$ be a "super-circuit" of size $$B \cdot S$$ that applies $$\mathcal{C}$$ independently to $$B = 2^b$$ input data before aggregating results. Labels of gates consists of two indices, one for the location in the layer and the other for which circuit among the copies to choose. For example, $$a = (a_1, a_2) \in \{0,1\}^{k_{i}} \times \{0,1\}^b$$.&#x20;

Let $$h$$ denote the polynomial $$\mathbb{F}^{k_i \times b} \rightarrow \mathbb{F}$$ defined by&#x20;

$$
h(a_1, a_2) :=\sum_{b_1, c_1 \in {0,1}^{k_{i+1}}} g(a_1, a_2, b_1, c_1)
$$

where&#x20;

$$
g(a_1, a_2, b_1, c_1) :=\widetilde{\mathsf{add}}_i(a_1, b_1, c_1) \left( \widetilde{W}_{i+1}^\prime(b_1, a_2) + \widetilde{W}_{i+1}^\prime(c_1, a_2) \right)+ \widetilde{\mathsf{mult}}_i(a_1, b_1, c_1) \cdot \widetilde{W}_{i+1}^\prime(b_1, a_2) \cdot \widetilde{W}_{i+1}^\prime(c_1, a_2)
$$

Then $$h$$ extends $$W_i'$$, just as in the single data case. However since $$h$$ is not multilinear like before, we multi-linearize $$h$$, and represent the multilinear extension of $$W_i'$$ as below. For any $$z \in \{0,1\}^{k_i + b}$$,&#x20;

$$
\widetilde{W}_i^\prime(z) =
\sum_{(a_1, a_2, b_1, c_1) \in \{0,1\}^{k_i + b + 2k_{i+1}}} g_z^{(i)}(a_1, a_2, b_1, c_1) \\
$$

where

$$
g_z^{(i)}(a_1, a_2, b_1, c_1) :=
\widetilde{\mathsf{eq}}_{k_i + b}(z, (a_1, a_2)) \cdot \left[
\widetilde{\textsf{add}}_i(a_1, b_1, c_1) \left( \widetilde{W}_{i+1}^\prime(b_1, a_2) + \widetilde{W}_{i+1}^\prime(c_1, a_2) \right) +
\widetilde{\textsf{mult}}_i(a_1, b_1, c_1) \cdot \widetilde{W}_{i+1}^\prime(b_1, a_2) \cdot \widetilde{W}_{i+1}^\prime(c_1, a_2)
\right]
$$

Note that as the same circuit structure is repeated, $$\widetilde{\mathsf{add}}_i$$ and $$\widetilde{\mathsf{mult}}_i$$ do not depend on $$a_2$$, which significantly reduces the cost of evaluating them for both parties and mitigates the bottleneck.&#x20;

The protocol is the same as the single data version, except for that at layer $$i$$, the sumcheck protocol is applied to $$g_{r_i}^{(i)}$$ instead.&#x20;

<figure><img src="../../.gitbook/assets/image (179).png" alt=""><figcaption></figcaption></figure>

Check [Section 5 of the paper](https://people.cs.georgetown.edu/jthaler/gkrnotes.pdf) for detailed cost analysis. &#x20;

***

## References

* [https://www.microsoft.com/en-us/research/wp-content/uploads/2016/12/2008-DelegatingComputation.pdf](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/12/2008-DelegatingComputation.pdf)
* [https://people.cs.georgetown.edu/jthaler/gkrnotes.pdf](https://people.cs.georgetown.edu/jthaler/gkrnotes.pdf)
* [https://people.cs.georgetown.edu/jthaler/ProofsArgsAndZK.pdf](https://people.cs.georgetown.edu/jthaler/ProofsArgsAndZK.pdf)
