---
description: >-
  This article aims to provide an intuitive explanation of the goals and
  processes of the Basefold protocol.
---

# Basefold

## Introduction

[**BaseFold**](https://eprint.iacr.org/2023/1705.pdf) is a new Polynomial Commitment Scheme that is field-agnostic, with verifier complexity of $$O(\log^2n)$$ and prover complexity of $$O(n \log n)$$. While the RS-code used in FRI-IOPP requires an FFT-friendly field, BaseFold eliminates this constraint by introducing the concept of **random foldable code**, making it usable with sufficiently large fields.

## Background

### Constant Polynomial in FRI-IOPP

In FRI, the following polynomial is folded using a random value $$\alpha_0$$.

$$
P_0(X) = c_0 + c_1X + c_2X^2 + \cdots + c_{2^k-1}X^{2^k-1}
$$

After the first round, the polynomial becomes:

$$
P_1(X) = c_0 + c_1 \cdot \alpha_0 + (c_2 + c_3 \cdot \alpha_0) X + \cdots + (c_{2^k-2} + c_{2^k-1} \cdot\alpha_0) X^{2^{k-1}- 1}
$$

If this folding process is repeated $$k+1$$ times, the resulting polynomial will eventually reduce to a constant polynomial, which looks like:

$$
P_{k+1}(X) = c_0 + c_1\cdot \alpha_0 + c_2 \cdot \alpha_1 + c_3 \cdot \alpha_1 \cdot \alpha_0 + \cdots + c_{2^k-1} \cdot \alpha_0 \cdot \alpha_1 \cdot \cdots \cdot \alpha_k
$$

This constant polynomial is identical to the evaluation of the multilinear polynomial $$Q$$ at point $$(\alpha_0, \alpha_1, \alpha_2, \dots, \alpha_k)$$.

$$
Q(X_0, X_1, X_2, \dots,X_k) = c_0+ c_1X_0 + c_2X_1 + \cdots + c_{2^{k-1} -1}X_0X_1\cdots X_{k}
$$

## Protocol Explanation

### Foldable Code

It is well-known that RS (Reed-Solomon) codes are a type of [linear code](https://en.wikipedia.org/wiki/Linear_code), and the encoding of linear codes can be represented as a matrix-vector multiplication. For example, the encoding of an $$[n, k, d]$$-RS code, when $$n = 16$$ and $$k = 8$$, can be represented as follows:

<figure><img src="../../.gitbook/assets/Screenshot 2024-12-23 at 3.52.34 PM.png" alt=""><figcaption></figcaption></figure>

The matrix on the left can be transformed into the matrix on the right by applying a row permutation.

<figure><img src="../../.gitbook/assets/Screenshot 2024-12-23 at 3.57.32 PM.png" alt=""><figcaption></figcaption></figure>

If we divide the matrix on the right into four parts, the red and yellow-colored sections are identical and correspond to the matrix used in RS encoding when $$n = 8$$ and $$k = 4$$. The green-colored section is equivalent to multiplying the red matrix by a [diagonal matrix](https://en.wikipedia.org/wiki/Diagonal_matrix) $$T = \mathsf{diag}(1, \omega, \omega^2, \dots, \omega^7)$$. Similarly, the blue-colored section is equivalent to multiplying the red matrix by another diagonal matrix $$T' = \mathsf{diag}(\omega^8, \omega^9, \omega^{10}, \dots, \omega^{15})$$.

If the matrix used in the encoding of a linear code takes the form above, it is referred to as a **foldable** linear code. Let us define this more formally.

Let $$c, k_0, d \in \mathbb{N}$$, $$k_i = k_0 \cdot 2^{i-1}, n_i = c \cdot k_i$$ for every $$i \in [1,d]$$. A $$(c, k_0,d)$$-**foldable linear code** is a [linear code](https://en.wikipedia.org/wiki/Linear_code) with a generator matrix $$G_d$$ and rate $$\frac{1}{c}$$ with the following propertie&#x73;**:**

* the diagonal matrices $$T_{i-1}, T'_{i-1} \in \mathbb{F}^{n_{i-1} \times n_{i-1}}$$ satisfies that $$\mathsf{diag}(T_{i-1})[j] \ne \mathsf{diag}(T'_{i-1})[j]$$ for every $$j \in [0,n_{i-1}-1]$$.
* the matrix $$G_i \in \mathbb{F}^{k_i \times n_i}$$ equals (up to row permutation)\\

$$
G_i = \begin{bmatrix}
G_{i−1} & G_{i−1} \\
G_{i−1} \cdot T_{i-1} & G_{i−1} \cdot T'_{i-1} \\
\end{bmatrix}
$$

#### Random Foldable Code

To design a linear code that is foldable and efficiently encodable regardless of its finite field, BaseFold uses this algorithm:

1. Set $$G_0 \leftarrow D_0$$​, where $$D_0$$​ is a distribution that outputs with 100% probability. Such a $$D_i$$​ is referred to as a $$(c, k_0)$$**-foldable distribution**.
2. Sample $$T_i \leftarrow_\$ (\mathbb{F}^\times)^{n_i}$$ and set $$T'_i = -T_i$$.
3. Generate $$G_{i+1}$$​ using $$G_i, T_i, T'_i$$.
4. Repeat steps 2 and 3 until a $$G_i$$​ of the desired size is obtained.

This type of code is called a **Random Foldable Code (RFC)**. The paper demonstrates that this code has good relative minimum distance. For those interested, refer to Section 3.1 of the paper. Additionally, it is noted that this is a special case of punctured [Reed-Muller codes](https://en.wikipedia.org/wiki/Reed%E2%80%93Muller_code). For further details, consult Appendix D of the paper.

#### Encoding Algorithm

The encoding can be computed recursively as follows:

1. If $$i = 0$$, return $$\mathsf{Enc}_0(m)$$.
2. Unpack $$1 \times k_i$$ matrix $$m$$ into a pair of $$1 \times k_i / 2$$ matrix $$(m_l, m_r)$$.
3. Set $$l = \mathsf{Enc}_{i-1}(m_l)$$, $$t = \mathsf{diag}(T_{i-1})$$, and then pack a pair of $$k_i  \times n_i / 2$$ matrices $$(l + t \circ r, l - t \circ r)$$ to $$k_i \times n_i$$ matrix.

This approach is valid and can be proven inductively:

* If $$i = 0$$, $$\mathsf{Enc}_0(m) = m \cdot G_0$$​, which holds by definition.
* If $$i>0$$, $$\mathsf{Enc}_i(m) = m \cdot G_i$$ holds because of the recursive construction of $$G_i$$.

$$
\begin{align*}
l + t \circ r &= \mathsf{Enc}_{i-1}(m_l) + \mathsf{diag}(T_{i-1}) \circ \mathsf{Enc}_{i-1}(m_r) \\ 
&= m_l \cdot G_{i-1} + \mathsf{diag}(T_{d-1}) \circ (m_r \cdot G_{i-1}) \\
&= m_l\cdot G_{i-1} + m_r \cdot G_{i-1} \cdot T_{i-1}
\end{align*} \\

\begin{align*}
l - t \circ r &= \mathsf{Enc}_{i-1}(m_l)-  \mathsf{diag}(T_{i-1}) \circ \mathsf{Enc}_{i-1}(m_r )\\ 
&= m_l \cdot G_{i-1} - \mathsf{diag}(T_{i-1}) \circ (m_r \cdot G_{i-1}) \\
&= m_l\cdot G_{i-1} - m_r \cdot G_{i-1} \cdot T_{i-1}
\end{align*} \\

(l+t\circ r, l-t \circ r) = (m_l, m_r) \cdot \begin{bmatrix}
G_{i-1} &G_{i-1} \\
G_{i-1} \cdot T_{i-1} & G_{i-1} \cdot -T_{i-1}
\end{bmatrix} = m \cdot G_i
$$

### BaseFold IOPP

To use this in IOPP, we need to ensure that $$\pi_i$$ is indeed an encoded random linear combination derived from $$m_{i+1}$$. To demonstrate this, let $$m_i$$ ​represent the message at the $$i$$-th layer and $$\pi_i$$ ​ the oracle (or block) at the $$i$$-th layer. Then, the following holds:

$$
\begin{align*}
\pi_{i+1} &= m_{i+1}\cdot G_{i+1} \\
&= m_{i+1} \cdot \begin{bmatrix}
G_i & G_i \\
G_i \cdot T_i & G_i \cdot T_i'
\end{bmatrix} \\
&= [\mathsf{Enc}_i(m_{i+1, l}) + \mathsf{diag}(T_i) \circ \mathsf{Enc}_i(m_{i+1, r}) || \mathsf{Enc}_i(m_{i+1, l}) + \mathsf{diag}(T_i') \circ \mathsf{Enc}_i(m_{i+1, r}) ]
\end{align*}
$$

Thus, for $$j \in [0, n_i - 1]$$, the following conditions are satisfied, where $$n_i$$ ​ is the length of $$\pi_i$$:

$$
\pi_{i+1}[j] = \mathsf{Enc}_i(m_{i+1, l})[j] + \mathsf{diag}(T_i)[j] \cdot \mathsf{Enc}_i(m_{i+1, r})[j]  \\
\pi_{i+1}[j + n_i] = \mathsf{Enc}_i(m_{i+1, l})[j] + \mathsf{diag}(T_i')[j] \cdot \mathsf{Enc}_i(m_{i+1, r})[j]
$$

The values $$\pi_{i+1}[j]$$ and $$\pi_{i+1}[j + n_i]$$ are evaluations of the polynomial $$f(X) = \mathsf{Enc}_i(m_{i+1, l})[j] + X \cdot \mathsf{Enc}_i(m_{i+1, r})[j]$$ at $$\mathsf{diag}(T_i)[j]$$ and $$\mathsf{diag}(T_i')[j]$$, respectively. When using the sampled value $$\alpha_i$$ at each $$i$$-th layer, $$f(\alpha_i)$$ can be constructed as follows, ensuring that $$\pi_i$$ encodes the message $$m_{i,l} + \alpha_i \cdot m_{i, r}$$:

$$
\begin{align*}
\pi_i[j] &= f(\alpha_i) \\
&= \mathsf{Enc}_i(m_{i+1, l})[j] + \alpha_i \cdot \mathsf{Enc}_i(m_{i+1, r})[j] \\
&=\mathsf{Enc}_i(m_{i+1, l} + \alpha_i \cdot m_{i+1, r})[j]
\end{align*}
$$

Using this property, an IOPP (Interactive Oracle Proof of Proximity) can be designed. Here, $$\text{interpolate}((x_1, y_1), (x_2, y_2))$$ refers to the computation of a degree-1 polynomial $$P$$ such that $$P(x_1) = y_1, P(x_2) = y_2$$.

$$\mathsf{IOPP.Commit}: \pi_d := \mathsf{Enc}_d(f) \in \mathbb{F}^{n_d} \rightarrow (\pi_{d - 1}, \dots, \pi_0) \in \mathbb{F}^{n_{d-1}} \times \cdots \times \mathbb{F}^{n_0}$$

Prover witness: the polynomial $$f \in \mathbb{F}[X_0, X_1, \dots, X_{d-1}]$$

For $$i$$ from $$d − 1$$ to $$0$$:

1. The verifier samples $$\alpha_i \leftarrow_\$ \mathbb{F}$$ and sends it to the prover.
2. For each index $$j \in [0, n_i - 1]$$, the prover
   1. sets $$f(X) := \mathsf{interpolate}((\mathsf{diag}(T_i)[j], π_{i+1}[j]),(\mathsf{diag}(T'_i)[j], π_{i+1}[j + n_i]))$$.
   2. sets $$\pi_i[j] = f(\alpha_i)$$.
3. The prover outputs oracle $$\pi_i \in \mathbb{F}$$.

$$\mathsf{IOPP.Query}: (\pi_d, \dots, \pi_0) \rightarrow \text{accept or reject}$$

1. The verifier samples an index $$\mu \leftarrow_\$ [0, n_{d-1}-1]$$.
2. For $$i$$ from $$d − 1$$ to $$0$$, the verifier
   1. queries oracle entries $$\pi_{i+1}[\mu], \pi_{i+1}[\mu + n_i]$$.
   2. computes $$p(X) := \mathsf{interpolate}((\mathsf{diag}(T_i)[\mu], π_{i+1}[\mu]),(\mathsf{diag}(T'_i)[\mu], π_{i+1}[\mu + n_i]))$$.
   3. checks that $$p(α_i) = \pi_i[\mu]$$.
   4. if $$i > 0$$ and $$\mu > n_{i−1}$$, update $$\mu \leftarrow \mu − n_{i−1}$$.
3. If $$\pi_0$$ is a valid codeword w.r.t. generator matrix $$G_0$$, output **accept**, otherwise output **reject**.

### Multilinear PCS based on interleaving BaseFold IOPP

When performing the sumcheck protocol, the problem is reduced to an evaluation claim. Interestingly, the BaseFold IOPP shares a similar structure: given an input oracle that encodes a polynomial $$f \in \mathbb{F}[X_0, X_1, \dots, X_{d-1}]$$, the final oracle sent by the honest prover in the IOPP protocol is precisely an encoding of a random evaluation $$f(\bm{r})$$, where $$\bm{r} = (r_0, r_1, \dots, r_{d-1})$$.

Thus, the sumcheck protocol and the IOPP protocol can be executed interleaved, sharing the same random challenge. At the end, the verifier needs to check whether the evaluation claim $$f(\bm{r})=y$$, obtained through the sumcheck protocol, matches the final prover message from the IOPP protocol. It performs as follows:

$$\mathsf{PC.Eval}$$

Public input: oracle $$\pi_f := \mathsf{Enc}_d(f) \in \mathbb{F}^{n_d}$$, point $$\bm{r}$$, claimed evaluation $$y = f(\bm{r}) \in \mathbb{F}$$

Prover witness: the polynomial $$f \in \mathbb{F}[X_0, X_1, \dots, X_{d-1}]$$

1. The prover sends the following to the verifier:

$$
h_d(X) =\sum_{\bm{b} \in \{0, 1\}^{d-1}} f(\bm{b}, X) \cdot \widetilde{\mathsf{eq}}_r(\bm{b}, X), \\
\widetilde{\mathsf{eq}}_r(X_0, X_1, \dots, X_{d-1}) = \prod_{i = 0}^{d-1}(r_i \cdot X_i + (1 - r_i)(1 - X_i))
$$

3. For $$i$$ from $$d − 1$$ to $$0$$:
4. Verifier samples and sends $$r_i \leftarrow_\$ \mathbb{F}$$ to the prover.
5. For each $$j \in [0, n_i - 1]$$, the prover
   1. sets $$g_j(X) := \mathsf{interpolate}((\mathsf{diag}(T_i)[j], π_{i+1}[j]),(\mathsf{diag}(T'_i)[j], π_{i+1}[j + n_i]))$$.
   2. sets $$\pi_i[j] = g_j(r_i)$$.
6. The prover outputs oracle $$\pi_i ∈ \mathbb{F}^{n_i}$$.
7. if $$i > 0$$, the prover sends verifier.

$$
h_i(X) =\sum_{\bm{b} \in \{0, 1\}^{i-1}} f(\bm{b}, X, r_i, \dots, r_{d-1}) \cdot \widetilde{\mathsf{eq}}_z(\bm{b}, X, r_i, \dots, r_{d-1})
$$

8. The verifier checks that
   1. $$\mathsf{IOPP.query}^{(\pi_d, \dots, \pi_0)}$$ outputs accept.
   2. $$h_d(0) + h_d(1) \stackrel{?}= y$$ and for every $$i \in [1, d - 1]$$, $$h_i(0) + h_i(1) \stackrel{?}= h_{i+1}(r_i)$$.
   3. $$\mathsf{Enc}_0(h_1(r_0) / \widetilde{\mathsf{eq}}_z(r_0, \dots, r_{d-1})) \stackrel{?}= \pi_0$$.

## Conclusion

<figure><img src="../../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

In a 256-bit PCS:

* BaseFold compared to field-agnostic Brakedown:
  * **Prover time**: Slower
  * **Verifier time**: Faster
  * **Proof size**: Bigger with a small number of variables but smaller with a large number of variables.
* BaseFoldFri compared to the non-field-agnostic ZeromorphFri:
  * **Prover time**: Faster
  * **Verifier time**: Faster
  * **Proof size**: Larger

<figure><img src="../../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

In a 64-bit PCS:

* BaseFold compared to field-agnostic Brakedown:
  * **Prover time**: Slower
  * **Verifier time**: Faster
  * **Proof size**: Smaller
* BaseFoldFri compared to the non-field-agnostic ZeromorphFri:
  * **Prover time**: Faster
  * **Verifier time**: Slower
  * **Proof size**: Larger

<figure><img src="../../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

In a 256-bit SNARK:

* BaseFold compared to field-agnostic Brakedown:
  * **Prover time**: Faster with a small number of variables but similar with a large number of variables.
  * **Verifier time**: Faster
  * **Proof size**: Smaller
* BaseFoldFri compared to the non-field-agnostic ZeromorphFri:
  * **Prover time**: Similar
  * **Verifier time**: Similar
  * **Proof size**: Larger

<figure><img src="../../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

In a 64-bit SNARK:

* Both BaseFold and BaseFoldFri, compared to Zeromorph:
  * **Prover time**: Similar
  * **Verifier time**: Slower with a small number of variables but faster with a large number of variables.
  * **Proof size**: Larger

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
