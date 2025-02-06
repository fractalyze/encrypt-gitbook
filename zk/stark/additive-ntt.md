---
description: >-
  This article aims to intuitively explain the goals and process of the Additive
  NTT protocol.
---

# Additive NTT

## Introduction

[**Binius**](https://eprint.iacr.org/2023/1784) is a SNARK constructed using a [**characteristic**](https://en.wikipedia.org/wiki/Characteristic_\(algebra\))**-2** finite field. In Binius, RS-encoding (Reed-Solomon encoding) is employed, but there is a challenge: the conventional Radix-2 FFT cannot be utilized in characteristic-2 finite fields. This paper addresses that challenge.

The title of the paper is _"Novel Polynomial Basis and Its Application to Reed-Solomon Erasure Codes."_ A characteristic-2 finite field is an extension field of the form $$\mathbb{F}_{2^r}$$​, which can be constructed as a polynomial ring. Traditionally, polynomials are represented using the Monomial Basis $$1, X, X^2, \dots$$which is a natural choice. However, this paper introduces an ingenious **new polynomial basis** and proposes an encoding algorithm that achieves performance comparable to the Radix-2 FFT.

## Background

### Radix-2 FFT

The **coefficient form** of a univariate polynomial of degree $$n−1$$ is defined as:

$$
P(X) = c_0 + c_1X + c_2X^2 + \cdots + c_{n-1}X^{n-1}
$$

where the basis is $$\{1, X, X^2, \dots, X^{n-1}\}$$.

On the other hand, the **evaluation form** of a univariate polynomial is defined as:

$$
P(X) = P(x_0)\cdot L_0(X) + P(x_1)\cdot L_1(X) + P(x_2)\cdot L_2(X) + \cdots + P(x_{n-1})\cdot L_{n-1}(X)
$$

where the basis is $$\{L_0(X), L_1(X), L_2(X), \dots, L_{n-1}(X)\}$$, called the **Lagrange basis**.

In **Radix-2 FFT (Fast Fourier Transform)**, a Lagrange basis is constructed using the [roots of unity](https://en.wikipedia.org/wiki/Root_of_unity). Each basis polynomial $$L_i(X)$$ is defined as:

$$
L_i(X) = \prod_{j \ne i}\frac{X - \omega^j}{\omega^i - \omega^j} = \begin{cases} 1 &\text{ if }X = \omega^i \\ 0 &\text{ otherwise} \end{cases}
$$

Since we use finite fields instead of complex numbers, this is technically a **Radix-2 NTT (Number Theoretic Transform)**, but for simplicity, we'll refer to it as FFT. Radix-2 FFT transforms a polynomial in coefficient form into its evaluation form. The matrix used to multiply the coefficient vector is called a [**Vandermonde matrix**](https://en.wikipedia.org/wiki/Vandermonde_matrix):

$$
\begin{bmatrix} P(1)\\ P(\omega)\\ P(\omega^2)\\ \vdots \\ P(\omega^{n-1}) \end{bmatrix} = \begin{bmatrix} 1 & 1 & 1 & \cdots & 1 \\ 1 & \omega & \omega^2 & \cdots & \omega^{n-1} \\ 1 & \omega^2 & \omega^4 & \cdots & \omega^{2(n-1)} \\ \vdots & \vdots & \vdots & \ddots & \vdots \\ 1 & \omega^{n-1} & \omega^{2(n-1)} & \cdots & \omega^{(n-1)^2} \end{bmatrix} \cdot \begin{bmatrix} c_0 \\ c_1 \\ c_2 \\ \vdots \\ c_{n-1} \end{bmatrix}
$$

The $$n$$-th roots of unity satisfy:

$$
\omega^n - 1 = (\omega^{n/2} - 1)(\omega^{n/2} + 1) = 0
$$

Since $$\omega^{n/2}$$ cannot equal $$1$$, it must equal $$-1$$.

Using the property of $$\omega^{n/2} = -1$$, Radix-2 FFT is efficiently computed via the [**Cooley-Tukey algorithm**](https://en.wikipedia.org/wiki/Cooley%E2%80%93Tukey_FFT_algorithm). The polynomial $$P(X)$$ is divided into two parts:

* $$P_E​(X^2)$$: even powers of $$X$$.
* $$P_O(X^2)$$: odd powers of $$X$$.

This gives:

$$
P(X) = P_{E}(X^2) + X \cdot P_{O}(X^2)
$$

For $$X = -X$$,substituting $$X \cdot \omega^{n/2}$$, we have:

$$
P(-X) = P(X \cdot \omega^{n/2}) = P_E(X^2) - X\cdot P_O(X^2)
$$

This transforms an $$n$$-point FFT problem into two $$n / 2$$-point FFT problems.

Define $$\Delta^m_i(X)$$ as follows:

$$
\Delta^m_i(X) = \begin{cases} \Delta^m_{i+1}(X) + X^{2^i}\Delta^{m + 2^i }_{i+1}(X) & \text{ if }0 \le i < \log_2 n \\ c_m & \text{ if }i = \log_2 n\\ \end{cases}
$$

For example, for $$n=8$$, the recursive computation proceeds as such:

$$
\begin{align*} P(X) &= \sum_{i = 0}^{7}c_iX_i \\ &= c_0 + c_4X^4 + X^2(c_2 + c_6X^4) + X(c_1 + c_5X^4 + X^2(c_3 + c_7X^4)\\ &= \Delta^0_2(X) + X^2\Delta^2_2(X) + X(\Delta^1_2(X) + X^2\Delta^3_2(X)) \\ &=\Delta^0_1(X) + X\Delta^1_1(X) \\ &=\Delta^0_0(X) \end{align*}
$$

By recursively applying this process, the computational complexity of FFT becomes $$O(n \cdot \log n)$$.

The **Inverse FFT (IFFT)** converts a polynomial from its evaluation form back to its coefficient form by using the inverse of the **Vandermonde matrix**:

$$
\begin{bmatrix} c_0 \\ c_1 \\ c_2 \\ \vdots \\ c_{n-1} \end{bmatrix} = \begin{bmatrix} 1 & 1 & 1 & \cdots & 1 \\ 1 & \omega & \omega^2 & \cdots & \omega^{n-1} \\ 1 & \omega^2 & \omega^4 & \cdots & \omega^{2(n-1)} \\ \vdots & \vdots & \vdots & \ddots & \vdots \\ 1 & \omega^{n-1} & \omega^{2(n-1)} & \cdots & \omega^{(n-1)^2} \end{bmatrix}^{-1} \cdot \begin{bmatrix} P(1)\\ P(\omega)\\ P(\omega^2)\\ \vdots \\ P(\omega^{n-1}) \end{bmatrix}
$$

The IFFT can also be computed in $$O(n \cdot \log n)$$ using a similar recursive approach to FFT.

### 2-adicity

When the modulus of a prime field is of the form $$2^S\cdot T + 1$$, a $$2^k$$-th root of unity $$\omega$$ for $$k < S$$ can be determined as follows.

$$
g^{p-1} = g^{2^S \cdot T} = (g^{2^{S-k} \cdot T})^{2^k} = \omega^{2^k} \equiv 1 \pmod p
$$

In other words, if the order of the multiplicative subgroup $$\mathbb{F}_p^\times$$​ of a prime field is of the form $$2^S \cdot T$$, and $$S$$ (also known as the **2-adicity**) is large, FFT can be used effectively. Fields with this structure are referred to as **FFT-friendly fields**.

What about characteristic-2 finite fields, such as $$\mathbb{F}_{2^r}$$​? For these fields, the order of the multiplicative subgroup $$\mathbb{F}_{2^r}^{\times}$$ is $$2^r - 1$$. Since the 2-adicity of $$2^r - 1$$ is 0, Radix-2 FFT cannot be used.

This naturally raises the question: when the characteristic is 2, what new polynomial basis would be suitable? Moreover, to achieve fast computation similar to Radix-2 FFT, the method must recursively reduce the problem into smaller subproblems.Protocol Explanation

**NOTE:** From this point onward, $$\omega$$ is no longer a root of unity, and $$\Delta$$ will be newly defined.

### Polynomial Basis

The elements of $$\mathbb{F}_{2^r}$$ can be represented as the set $$\{\omega_i\}_{i=0}^{2^r - 1}$$, where:

$$
i = \sum_{k = 0}^{r-1}i_k \cdot 2^k, \forall i_k \in {0,1}
$$

Given a basis vector $$v = \{v_0, \dots, v_{r-1}\}$$ composed of elements from $$\mathbb{F}_{2^r}$$, each $$\omega_i$$ can be expressed as:

$$
\omega_i = \sum_{k = 0}^{r-1}i_k \cdot v_k
$$

We define the **subspace vanishing polynomial** as:

$$
W_j(X) = \prod_{i=0}^{2^j-1}(X + \omega_i)\text{, where } 0 \le j \le r - 1
$$

This mean the full form of the vanishing polynomials $$W_j(X)$$ are as follows:

$$
W_0(X) = X + \omega_0, \\ W_1(X) = (X + \omega_0)(X + \omega_1) ,\\ W_2(X) = (X + \omega_0)(X + \omega_1)(X + \omega_2)(X + \omega_3) ,\\
$$

and so on.

Thus, $$\deg(W_j(X)) = 2^j$$ and the roots of $$W_2(X)$$ are $$\omega_0, \omega_1, \omega_2, \omega_3$$ . This follows from the fact that addition in $$\mathbb{F}_{2^r}$$​ is XOR, so $$\omega_i + \omega_i = 0$$, a direct result of the field’s characteristic being 2.

**Lemma 1.** $$W_j(X)$$ is a $$\mathbb{F}_2$$-linearized polynomial.

$$
W_j(X) = \sum_{i = 0}^{j}a_{j,i}X^{2^i} \text{, where }a_{j,i} \in \mathbb{F}_{2^r}
$$

Furthermore, $$W_j(X)$$ satisfies the following property:

$$
W_j(X + Y) = W_j(X) + W_j(Y), \forall X,Y \in \mathbb{F}_{2^r}
$$

We now define a new **polynomial basis** $$X_i(X)$$ as:

$$
X_i(X) = \prod_{j= 0}^{r-1}(\frac{W_j(X)}{W_j(\omega_{2^j})})^{i_j}
$$

where:

$$
i = \sum_{j = 0}^{r-1}i_j \cdot 2^j, \forall i_j \in {0,1}
$$

The new polynomial basis $$X_i(X)$$ is as follows:

$$
X_0(X) = 1 ,
\\ X_1(X) = \frac{W_0(X)}{W_0(\omega_1)} = \frac{X + \omega_0}{\omega_1 + \omega_0} ,
\\ X_2(X) = \frac{W_1(X)}{W_1(\omega_2)} = \frac{(X + \omega_0)(X + \omega_1)}{(\omega_2 + \omega_0)(\omega_2 + \omega_1)},
\\ X_3(X) = \frac{W_0(X)W_1(X)}{W_0(\omega_1)W_1(\omega_2)} = \frac{(X + \omega_0)^2(X + \omega_1)}{(\omega_1 + \omega_0)(\omega_2 + \omega_0)(\omega_2 + \omega_1)} ,
\\ X_4(X) = \frac{W_2(X)}{W_2(\omega_4)} = \frac{(X + \omega_0)(X + \omega_1)(X + \omega_2)(X + \omega_3)}{(\omega_4 + \omega_0)(\omega_4 + \omega_1)(\omega_4 + \omega_2)(\omega_4 + \omega_3)},
$$

and so on.

Thus, $$\deg(X_i(X)) = i$$.

The **coefficient form** of an $$n-1$$-degree univariate polynomial $$[D_n](X) \in \mathbb{F}_{2^r}[X] / (X^{2^r} - 1)$$ is expressed as:

$$
[D_n](X) = d_0 + d_1X_1(X) + d_2X_2(X) + \dots + d_{n-1}X_{n-1}(X)
$$

The **evaluation form** of this polynomial is:

$$
[D_n](X) = [D_n](\omega_0 + \omega_l) \cdot L^l_0(X) + [D_n](\omega_1 + \omega_l) \cdot L^l_1(X) + [D_n](\omega_2 + \omega_l) \cdot L^l_2(X) + \cdots + [D_n](\omega_{n-1} + \omega_l) \cdot L^l_{n-1}(X)
$$

where $$L^l_i(X)$$ is defined as:

$$
L^l_i(X) = \prod_{j \ne i}\frac{X - \omega_j}{\omega_i + \omega_l - \omega_j} = \begin{cases} 1 &\text{ if }X = \omega_i + \omega_l \\ 0 &\text{ otherwise} \end{cases}
$$

We denote the set of evaluations as $$\hat{D}^l_n$$, defined as such:

$$
\hat{D}^l_n = \{[D_n](\omega_0 + \omega_l), [D_n](\omega_1 + \omega_l),[D_n](\omega_2 + \omega_l),\dots, [D_n](\omega_{n-1} + \omega_l) \}
$$

### Delta Polynomial

The recursive calculation of $$[D_n](X)$$ can be performed by defining $$\Delta^m_i(X)$$ as follows:

$$
\Delta^m_i(X) = \begin{cases}
\Delta^m_{i+1}(X) + \frac{W_i(X)}{W_i(\omega_{2^i})}\Delta^{m + 2^i}_{i+1}(X) & \text{ if }0 \le i < \log_2 n \\
d_m & \text { if } i = \log_2n \end{cases}
$$

where $$0 \le m < 2^i$$.

For $$n = 8$$, $$\Delta$$ exists as follows:

$$
\Delta^0_3(X) = d_0,\Delta^1_3(X) = d_1, \Delta^2_3(X) = d_2, \dots, \Delta^7_3(X) = d_7, \\
\Delta^0_2(X) = \Delta^0_3(X) + \frac{W_2(X)}{W_2(\omega_4)} \Delta^4_3(X) = d_0 + \frac{W_2(X)}{W_2(\omega_4)}d_4, \\
\Delta^1_2(X) = \Delta^1_3(X) + \frac{W_2(X)}{W_2(\omega_4)} \Delta^5_3(X) = d_1 + \frac{W_2(X)}{W_2(\omega_4)}d_5, \\
\Delta^2_2(X) = \Delta^2_3(X) + \frac{W_2(X)}{W_2(\omega_4)} \Delta^6_3(X) = d_2 + \frac{W_2(X)}{W_2(\omega_4)}d_6, \\
\Delta^3_2(X) = \Delta^3_3(X) + \frac{W_2(X)}{W_2(\omega_4)} \Delta^7_3(X) = d_3 + \frac{W_2(X)}{W_2(\omega_4)}d_7, \\
\Delta^0_1(X) = \Delta^0_2(X) + \frac{W_1(X)}{W_1(\omega_2)} \Delta^2_2(X), \\
\Delta^1_1(X) = \Delta^1_2(X) + \frac{W_1(X)}{W_1(\omega_2)} \Delta^3_2(X), \\
\Delta^0_0(X) = \Delta^0_1(X) + \frac{W_0(X)}{W_0(\omega_1)} \Delta^1_1(X)
$$

$$[D_n](X)$$ is equivalent to $$\Delta^0_0(X)$$. For example, $$[D_8](X)$$ is recursively computed as follows:

$$
\begin{align*} [D_8](X) &= \sum_{i = 0}^{7} d_iX_i(X) \\ &=d_0 + d_1\frac{W_0(X)}{W_0(\omega_1)} + d_2\frac{W_1(X)}{W_1(\omega_2)} + d_3\frac{W_0(X)W_1(X)}{W_0(\omega_1)W_1(\omega_2)} + d_4\frac{W_2(X)}{W_2(\omega_4)} \\ &\space\space\space\space\space d_5\frac{W_0(X)W_2(X)}{W_0(\omega_1)W_2(\omega_4)} + d_6\frac{W_1(X)W_2(X)}{W_1(\omega_2)W_2(\omega_4)} + d_7\frac{W_0(X)W_1(X)W_2(X)}{W_0(\omega_1)W_1(\omega_2)W_2(\omega_4)} \\ &=\bigg(d_0 + d_4\frac{W_2(X)}{W_2(\omega_4)} + \frac{W_1(X)}{W_1(\omega_2)}\bigg( d_2 + d_6\frac{W_2(X)}{W_2(\omega_4)}\bigg)\bigg) \\ &\space\space\space\space\space + \frac{W_0(X)}{W_0(\omega_1)}\bigg(d_1 + d_5\frac{W_2(X)}{W_2(\omega_4)} + \frac{W_1(X)}{W_1(\omega_2)}\bigg(d_3 + d_7\frac{W_2(X)}{W_2(\omega_4)} \bigg) \bigg) \\ &=\bigg(\Delta^0_2(X) + \frac{W_1(X)}{W_1(\omega_2)}\Delta^2_2(X) \bigg) + \frac{W_0(X)}{W_0(\omega_1)}\bigg( \Delta^1_2(X) + \frac{W_1(X)}{W_1(\omega_2)}\Delta^3_2(X) \bigg) \\ &= \Delta^0_1(X) + \frac{W_0(X)}{W_0(\omega_1)}\Delta^1_1(X) \\ &= \Delta^0_0(X) \end{align*}
$$

**Lemma 2.** $$\Delta^m_i(X + Y) = \Delta^m_i(X), \forall Y \in \{\omega_b\}_{b = 0}^{2^i - 1}$$

**Proof:**

**Case 1:** $$i = \log_2 n$$, both $$\Delta^m_i(X + Y)$$ and $$\Delta^m_i(X)$$ equal $$d_m$$, so the equality holds trivially.

**Case 2:** $$i = \log_2 n - 1$$, $$\Delta^m_i(X)$$ is given as:

$$
\begin{align*} \Delta^m_{\log_2 n - 1}(X) &= \Delta^m_{\log_2 n}(X) + \frac{W_{\log_2 n - 1}(X)}{W_{\log_2 n - 1}(\omega_{2^{\log_2 n - 1}})} \Delta^{m + 2^{\log_2 n - 1}}_{\log_2 n}(X) \\ 
&= d_m + \frac{W_{\log_2 n - 1}(X)}{W_{\log_2 n - 1}(\omega_{2^{\log_2 n - 1}})} d_{m + 2^{\log_2 n - 1}} \end{align*}
$$

which simplifies as follows:

$$
\Delta^m_{\log_2 n - 1}(X + Y) 
= d_m + \frac{W_{\log_2 n - 1}(X + Y)}{W_{\log_2 n - 1}(\omega_{2^{\log_2 n - 1}})} d_{m + 2^{\log_2 n - 1}}
$$

Using **Lemma 1**:

$$
W_{\log_2 n - 1}(X + Y) = W_{\log_2 n - 1}(X) + W_{\log_2 n - 1}(Y)
$$

Since $$Y \in \{\omega_b\}_{b=0}^{2^i - 1}$$, we know $$W_{\log_2 n - 1}(Y) = 0$$, so:

$$
\begin{align*}
\Delta^m_{\log_2 n - 1}(X + Y) 
&= d_m + \frac{W_{\log_2 n - 1}(X)}{W_{\log_2 n - 1}(\omega_{2^{\log_2 n - 1}})} d_{m + 2^{\log_2 n - 1}} \\
&= \Delta^m_{\log_2 n -1}(X)
\end{align*}
$$

**Case 3:** Inductive Step

Assume the equality holds for $$i = c+ 1$$. At , we have:

$$
\Delta^m_{c}(X + Y)  = \Delta^m_{c + 1}(X + Y) + \frac{W_{c}(X + Y)}{W_{c}(\omega_{2^c})} \Delta^{m + 2^c}_{c + 1}(X + Y)
$$

By **Lemma 1**:

$$
W_c(X + Y) = W_c(X) + W_c(Y)
$$

Since $$Y \in \{\omega_b\}_{b=0}^{2^c - 1}$$, we know $$W_{c}(Y) = 0$$, thus:

$$
\begin{align*}
\Delta^m_{c}(X + Y)  &= \Delta^m_{c + 1}(X + Y) + \frac{W_{c}(X)}{W_{c}(\omega_{2^c})} \Delta^{m + 2^c}_{c + 1}(X + Y)  \\
&= \Delta^m_c(X)
\end{align*}
$$

### Transform

Define the transform $$\Psi$$ as:

$$
\Psi(i, m, l) = \begin{cases} \{ \Delta^m_i(\omega_c + \omega_l)| c \in \{b \cdot 2^i\}^{n/2^i - 1}_{b = 0} \} & \text{ if } 0 \le i < \log_2 n \\ \{d_m\} & \text{ if } i = \log_2n \end{cases}
$$

where $$0 \le m < 2^i$$.

The transform computes $$\{\Psi(\log_2 n, k, l)|k \in [0, n-1]\} \rightarrow \Psi(0, 0, l)$$. For $$n = 8$$, this becomes:

$$
\{d_0, d_1, d_2 \dots, d_{7}\} = \Psi(3, 0, l) \cup \Psi(3, 1, l) \cup \Psi(3, 2, l) \cup \cdots \cup \Psi(3, 7, l) \\ \rightarrow \{\Delta^0_0(\omega_0 + \omega_l),\Delta^0_0(\omega_1 + \omega_l), \Delta^0_0(\omega_2 + \omega_l), \dots, \Delta^0_0(\omega_{7} + \omega_l)\} = \Psi(0, 0, l)
$$

Remember that $$\Delta^0_0(X)$$ is equivalent to $$[D_n](X)$$.

$$\Psi(i, m, l)$$ can be divided into two parts as follows:

* $$\{\Delta^m_i(\omega_c + \omega_l)| c \in \{b \cdot 2^{i+1}\}^{n/2^{i+1} - 1}_{b = 0}\}$$ which can be computed from $$\Psi(i + 1, m, l)$$.
* $$\{\Delta^m_i(\omega_c + \omega_l + \omega_{2^i})| c \in \{b \cdot 2^{i+1}\}^{n/2^{i+1} - 1}_{b = 0}\}$$ which can be computed from $$\Psi(i + 1, m +2^i, l)$$.

**Part 1.** $$\Psi(i + 1, m, l)$$

$$
\Delta^m_i(\omega_c + \omega_l) = \Delta^m_{i+1}(\omega_c + \omega_l) + \frac{W_i(\omega_c + \omega_l)}{W_i(\omega_{2^i})}\Delta^{m + 2^i}_{i +1}(\omega_c + \omega_l)
$$

Since all the right-hand terms are known, the left-hand terms can be determined.

**Part 2.** $$\Psi(i + 1, m + 2^i, l)$$

$$
\begin{align*} 
\Delta^m_i(\omega_c + \omega_l + \omega_{2^i}) &= \Delta^m_{i+1}(\omega_c + \omega_l + \omega_{2^i}) + \frac{W_i(\omega_c + \omega_l + \omega_{2^i})}{W_i(\omega_{2^i})}\Delta^{m + 2^i}_{i +1}(\omega_c + \omega_l + \omega{2^i}) 
\\ &= \Delta^m_{i+1}(\omega_c + \omega_l) + \frac{W_i(\omega_c + \omega_l + \omega_{2^i})}{W_i(\omega_{2^i})}\Delta^{m + 2^i}_{i +1}(\omega_c + \omega_l) \because \text{Lemma } 2. 
\\ &= \Delta^m_{i+1}(\omega_c + \omega_l) + \frac{W_i(\omega_c + \omega_l) + W_i(\omega_{2^i})}{W_i(\omega_{2^i})}\Delta^{m + 2^i}_{i +1}(\omega_c + \omega_l) \because \text{Lemma } 1.
\\ &= \Delta^m_{i+1}(\omega_c + \omega_l) + \frac{W_i(\omega_c + \omega_l)}{W_i(\omega_{2^i})}\Delta^{m + 2^i}_{i +1}(\omega_c + \omega_l) + \Delta^{m + 2^i}_{i +1}(\omega_c + \omega_l)
\\ &= \Delta^m_i(\omega_c + \omega_l) + \Delta^{m + 2^i}_{i+1}(\omega_c + \omega_l) \end{align*}
$$

Since $$\Delta^m_i(\omega_c + \omega_l)$$ can be computed from the Part 1, the left-hand terms can be determined.

When $$n=8, i=2$$, for $$\Psi(2, m, l)$$, it can be split into $$\Psi(3,m , l) = \{d_m\}$$ and $$\Psi(3, m + 4, l) = \Delta^{m+4}_3(X) =\{d_{m+4}\}$$.

$$
\begin{align*}
\Psi(2, m, l) &= \{\Delta^m_2(\omega_0 + \omega_l),\Delta^m_2(\omega_4 + \omega_l)\} 
\\ &= \{\Delta^m_3(\omega_0 + \omega_l) + \frac{W_2(\omega_0 + \omega_l)}{W_2(\omega_4)}\Delta^{m + 4}_3(\omega_0 + \omega_l), \Delta^m_3(\omega_4 + \omega_l) + \frac{W_2(\omega_4 + \omega_l)}{W_2(\omega_4)}\Delta^{m + 4}_3(\omega_4 + \omega_l) \} 
\\ &= \{d_m + \frac{W_2(\omega_0 + \omega_l)}{W_2(\omega_4)}d_{m+4}, d_m + \frac{W_2(\omega_4 + \omega_l)}{W_2(\omega_4)}d_{m+4} \} 
\end{align*}
$$

When $$n=8, i=1$$, for $$\Psi(1, m, l)$$, it can be split into$$\Psi(2, m, l) = \{\Delta^m_2(\omega_0 + \omega_l), \Delta^m_2(\omega_4 + \omega_l)\}$$ and $$\Psi(2, m + 2, l) = \{\Delta^{m + 2}_2(\omega_0 + \omega_l), \Delta^{m + 2}_2(\omega_4 + \omega_l)\}$$.

$$
\begin{align*}\Psi(1, m, l) &= \{\Delta^m_1(\omega_0 + \omega_l),\Delta^m_1(\omega_2 + \omega_l),\Delta^m_1(\omega_4 + \omega_l),\Delta^m_1(\omega_6 + \omega_l)\} \\ &= \{\Delta^m_2(\omega_0 + \omega_l) + \frac{W_1(\omega_0 + \omega_l)}{W_1(\omega_2)}\Delta^{m + 2}_2(\omega_0 + \omega_l), \Delta^m_2(\omega_2 + \omega_l) + \frac{W_1(\omega_2 + \omega_l)}{W_1(\omega_2)}\Delta^{m + 2}_2(\omega_2 + \omega_l), \Delta^m_2(\omega_4 + \omega_l) + \frac{W_1(\omega_4 + \omega_l)}{W_1(\omega_2)}\Delta^{m + 2}_2(\omega_4 + \omega_l), \Delta^m_2(\omega_6 + \omega_l) + \frac{W_1(\omega_6 + \omega_l)}{W_1(\omega_2)}\Delta^{m + 2}_2(\omega_6 + \omega_l) \} \\ &= \{\Delta^m_2(\omega_0 + \omega_l) + \frac{W_1(\omega_0 + \omega_l)}{W_1(\omega_2)}\Delta^{m + 2}_2(\omega_0 + \omega_l), \Delta^m_2(\omega_0 + \omega_l + \omega_2) + \frac{W_1(\omega_0 + \omega_l + \omega_2)}{W_1(\omega_2)}\Delta^{m + 2}_2(\omega_0 + \omega_l + \omega_2), \Delta^m_2(\omega_4 + \omega_l) + \frac{W_1(\omega_4 + \omega_l)}{W_1(\omega_2)}\Delta^{m + 2}_2(\omega_4 + \omega_l), \Delta^m_2(\omega_4 + \omega_l + \omega_2) + \frac{W_1(\omega_4 + \omega_l + \omega_2)}{W_1(\omega_2)}\Delta^{m + 2}_2(\omega_4 + \omega_l + \omega_2) \} \\ &= \{\Delta^m_2(\omega_0 + \omega_l) + \frac{W_1(\omega_0 + \omega_l)}{W_1(\omega_2)}\Delta^{m + 2}_2(\omega_0 + \omega_l), \Delta^m_2(\omega_0 + \omega_l) + \frac{W_1(\omega_0 + \omega_l)}{W_1(\omega_2)}\Delta^{m + 2}_2(\omega_0 + \omega_l), \Delta^m_2(\omega_4 + \omega_l) + \frac{W_1(\omega_4 + \omega_l)}{W_1(\omega_2)}\Delta^{m + 2}_2(\omega_4 + \omega_l), \Delta^m_2(\omega_4 + \omega_l) + \frac{W_1(\omega_4 + \omega_l)}{W_1(\omega_2)}\Delta^{m + 2}_2(\omega_4 + \omega_l) \} \\ \end{align*}
$$

### Inverse Transform

To compute $$\Psi(i + 1, m, l)$$ and $$\Psi(i + 1, m + 2^i, l)$$ from $$\Psi(i, m, l)$$:

**Part 1.** $$\Psi(i + 1, m + 2^i, l)$$

$$
\Psi(i + 1, m + 2^i, l) = \{\Delta^{m + 2^i}_{i+1}(\omega_c + \omega_l)| c \in \{b \cdot 2^{i+1}\}^{n/2^{i+1} - 1}_{b = 0}\}
$$

where: (Refer to Part 2. in Transform section above.)

$$
\Delta^{m + 2^i}_{i+1}(\omega_c + \omega_l) = \Delta^m_i(\omega_c + \omega_l) + \Delta^m_i(\omega_c + \omega_l + \omega_{2^i})
$$

Since all the right-hand terms are known, the left-hand terms can be determined.

**Part 2.** $$\Psi(i + 1, m, l)$$

$$
\Psi(i + 1, m, l) = \{\Delta^{m }_{i+1}(\omega_c + \omega_l)| c \in \{b \cdot 2^{i+1}\}^{n/2^{i+1} - 1}_{b = 0}\}
$$

where (Refer to Part 1. in Transform section above):

$$
\Delta^{m}_{i+1}(\omega_c + \omega_l) = \Delta^m_i(\omega_c + \omega_l) + \frac{W_i(\omega_c + \omega_l)}{W_i(\omega_{2^i})}\Delta^{m + 2^i}_{i+1}(\omega_c + \omega_l)
$$

Since $$\Delta^m_i(\omega_c + \omega_l)$$ can be computed from the Part 1, the left-hand terms can be determined.

<figure><img src="../../.gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

The diagram above illustrates the data dependencies for transforming and inversely transforming a polynomial of size 8. Each eleme[nt is computed as the sum of a solid-line element and a dashed-line element, where the la](#user-content-fn-1)[^1]tter represents scalar multiplication by $$\widehat{W}^j_i$$.

$$
\widehat{W}^j_i = \frac{W_i(\omega_j)}{W_i(\omega_{2^i})}
$$

## Conclusion

The method for performing a formal derivative on the **Novel Polynomial Basis** and its use in the RS erasure decoding algorithm is omitted here due to space constraints. For those interested, please refer to the original paper.

<figure><img src="../../.gitbook/assets/image (9).png" alt=""><figcaption><p>Complexities of Previous n-point FFT Algorithms</p></figcaption></figure>

Using the newly proposed Novel Polynomial Basis, the Additive NTT achieves $$O(n \log n)$$ additive complexity and $$O(n \log n)$$ multiplicative complexity without any constraints. Consequently, for the first time, RS-encoding over characteristic-2 finite fields achieves $$O(n \log n)$$ complexity.

<figure><img src="../../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

Additionally, it is well-known that polynomial multiplication can be optimized using FFT. Similarly, by leveraging the Novel Polynomial Basis, multiplication in characteristic-2 finite fields can also be optimized.

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)

[^1]: n
