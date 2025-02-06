---
description: >-
  This document aims to intuitively explain the goals and processes of the
  Brakedown protocol.
---

# Brakedown

## Introduction

[**Brakedown**](https://eprint.iacr.org/2021/1043) originates from the question, "How can we generate proofs for R1CS as quickly as possible?" It presents a scheme that achieves proof generation with $$O(N)$$ field operations. To accomplish this, it introduces a method of encoding that operates in linear time. While similar techniques were used in[ BCG](https://eprint.iacr.org/2020/1426.pdf), Brakedown distinguishes itself by being more practical. Additionally, it does not require a trusted setup, may offer quantum resistance, and is field-agnostic, making it more practical.

## Background

See [LinearCode](../../primitives/coding-theory/linear-code.md) for more details.

### Why use Merkle Tree operations?

When committing to an $$O(N)$$-length vector, the operations involved generally have the following complexities:

* **FFT** **Operations** → $$O(N \cdot \log N)$$ field operations
* **Merkle Tree** **Operations** → $$O(N)$$ hash operations
* **MSM** **Operations** → Using Pippenger's algorithm, $$O(N \cdot \lambda / \log(N \cdot \lambda))$$ group operations

To achieve $$O(N)$$, FFT operations, which are $$O(N \cdot \log N)$$, cannot be used. Similarly, MSM operations are avoided because group operations are slower than field operations. Instead, **Merkle Tree operations** are utilized for commitment.

The traditional encoding function used in RS-codes involves FFT operations. However, due to the limitations mentioned earlier, alternative encoding methods must be explored.

### Tensor Product

The [**Tensor Product**](https://en.wikipedia.org/wiki/Tensor_product) is an operation that generates all possible combinations of two vectors v and u from different vector spaces. For example, if $$q_1 = [a, b]$$ and $$q_2 = [x, y, z]$$, the tensor product between the two is the following matrix:

$$
q_1 \otimes q_2 =
\begin{bmatrix}
a \cdot x & a \cdot y & a \cdot z \\
b \cdot x & b \cdot y & b \cdot z
\end{bmatrix}
$$

Additionally, the tensor product can be used to express a **Multi Linear Extension (MLE)** evaluation like so:

$$
\text{Given an evaluation point } r \in \mathbb{F}^l\text{ and }z \in \mathbb{F}^{2^l} \\
\begin{align*}
g(r) &= \langle \otimes_{i=1}^{l} (r_1, 1 - r_1), z\rangle \\
&= \langle(r_1, 1 - r_1) \otimes\cdots\otimes(r_l, 1 - r_l), z\rangle
\end{align*}
$$

For example, when $$l = 2$$, the MLE evaluation is calculated as follows:

$$
\begin{align*} 
g(r) &= \mathsf{eq}((0, 0), r) \cdot z_1 + \mathsf{eq}((0, 1), r) \cdot z_2 + \mathsf{eq}((1, 0), r) \cdot z_3 + \mathsf{eq}((1, 1), r) \cdot z_4 \\
&= (1 - r_1)\cdot(1- r_2)\cdot z_1 + (1 - r_1)\cdot r_2\cdot z_2 +  r_1\cdot(1- r_2)\cdot z_3 + r_1 \cdot r_2\cdot z_4  \\
&= \langle((r_1, 1-r_1) \otimes (r_2, 1-r_2)), (z_1, z_2, z_3, z_4)\rangle \\
&= \langle((r_1, 1-r_1) \otimes (r_2, 1-r_2)), z)\rangle 
\end{align*}
$$

## Protocol Explanation

### Brief Overview

The protocol is divided into three phases: Commitment Phase, Testing Phase and Evaluation Phase.

#### Commitment Phase

This phase commits to the input matrix, with the following steps:

1. Each matrix $$u$$ is provided as an input, where $$u_i$$ represents the $$i$$-th row.

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfebaVY1aTpJuoe331PPsIGM0TnmdagxWEITGeKBKtREXgEwzDTOXwNlOpt832g0KA3gGYS7-DSB48tIWNKglmdm80wbpNJqe-fzodOcvBbRqZ3Dl2p_-hvHXeRYrS0ZdPT4Vjq?key=TV6kc-5CN5lS3U7bOqCI-uh5" alt=""><figcaption></figcaption></figure>

$$
u= \{u_i\}_{i \in [r]} \in (\mathbb{F}^c)^r
$$

2.  For a given rate $$ho$$, each row of length $$c$$ is encoded into a row of length $$N$$, producing an $$r \times N$$ matrix $$\hat{u}$$. The prover commits this $$\hat{u}$$ matrix to the verifier using Merkle hash.\\

    <figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXf2SD0w2En_Y9dpLmdKvC2lsARHqwgiPCd4J86ETjfZaV0s3MDRcOyMC0p7blgY9xgNcV0bmWckePT0DqWE4dLXYxyUNjEMyx_3qSZ3VTCq97fr77NohFKtoJArrlN4R1enowku?key=TV6kc-5CN5lS3U7bOqCI-uh5" alt=""><figcaption></figcaption></figure>

$$
\hat{u}= \{\mathsf{Enc}(u_i)\}_{i \in [r]} \in (\mathbb{F}^N)^r
$$

#### Testing Phase

This phase verifies whether the encoding was performed correctly, with the following steps:

1. The verifier samples a random scalar $$\alpha$$ and sends it to the prover.
2.  The prover computes $$u'$$ using the random value $$\alpha$$ and sends it to the verifier:\\

    <div data-full-width="true"><figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfMd0VPOyqQNinoHcDkohIlvmfA6mCjbOLtVe_96Jo8-xC0vj1JQ5A_V8uJr9lw-gj5sCKlwWAraxfriSx8tTKtLkOZJIClX5Ve2hFWxmCjuyaQOKR9puu9Uol89u7CSWldkydA?key=TV6kc-5CN5lS3U7bOqCI-uh5" alt=""><figcaption></figcaption></figure></div>

$$
u' = \mathsf{RLC}(u) =  \sum_{i=1}^r\alpha^{i-1} \cdot u_i \in \mathbb{F}^c
$$

3. The verifier tests the following equality by sampling $$l$$ values between $$1$$ and $$N$$:

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXe1Ul9IcoVYJWYwmA0JO9S0GGXswOV23n56IUzx4kaEaHW0a_LK3Cs1ozPPwEIR8iZ-JtJCEq2cV6TiEqAyohMxaYZL5vtyg9f2BR4tlA65e0vrTmVUo-srm2KH1E8JW79fOYFy?key=TV6kc-5CN5lS3U7bOqCI-uh5" alt=""><figcaption></figcaption></figure>

$$
\sum_{i=1}^r\alpha^{i-1} \cdot \hat{u}_i \stackrel{?}= \mathsf{Enc}(u')
$$

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXc0hhtNYtKjEyLpldChAjuzBO6cC_dtY3g_ALcPWd85zG5YrBjrupHDRBhOrCQfo_PuPyaRnKwgWzjje0XoGHTMEcu2PzDjD-nWoctJhGAgMnuGl0uhcuHctFW6_qHIDAJdBcsJ?key=TV6kc-5CN5lS3U7bOqCI-uh5" alt=""><figcaption></figcaption></figure>

This holds because encoding and **random linear combination (RLC)** are commutative and the codeword is linear code. That is, encoding the rows of the matrix and then performing RLC is equivalent to performing RLC first and then encoding. According to Section 4, "Concrete Optimizations to the Commitment Scheme," the testing phase can be skipped if a Polynomial Commitment Scheme is used.

#### Evaluation Phase

This phase performs identical checks as in the testing phase but without encoding, with the following steps:

1. The verifier samples $$q_1 \in \mathbb{F}^r$$.
2. The prover computes $$u''$$ and sends it to the verifier: (Identical to Testing Phase 2.)

$$
u'' = \sum_{i=1}^rq_{1,i}\cdot u_i \in \mathbb{F}^c
$$

3. The verifier tests the following equality by sampling $$q_2 \in \mathbb{F}^c$$: (Identical to Testing Phase 3.) Here, tensor product between $$q_1$$ and $$q_2$$ are flattened to the vector.\\

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXd50uA_k4bUMg3x27qencQVzh_tGSFU1aX3femSviBEh2CVRLeu7B3dH9y1vmkTqwtdAbTZctLzaZbCjAkrRzePEyA8TX5xNJKxHYGntJPvalmwTcwYvQfUq7cB9lE1MkHgId4?key=TV6kc-5CN5lS3U7bOqCI-uh5" alt=""><figcaption></figcaption></figure>

$$
g(r) = \langle(q_1 \otimes q_2), u)\rangle \stackrel{?}= \langle u'', q_2\rangle
$$

For example, when $$r = 2$$ and $$c = 2$$, the equation holds due to the following reasoning:

$$
\begin{align*}
\langle(q_1 \otimes q_2), u\rangle &= q_{1,1}\cdot q_{2,1} \cdot u_{1,1} + q_{1,1}\cdot q_{2,2}\cdot u_{2,1} + q_{1,2}\cdot q_{2,1}\cdot u_{1,2}+  q_{1,2}\cdot q_{2,2}\cdot u_{2,2} \\
&= (q_{1,1}\cdot u_{1,1} + q_{1,2}\cdot u_{1,2})\cdot q_{2,1} + (q_{1,1}\cdot u_{2,1} + q_{1,2} \cdot u_{2,2})\cdot q_{2,2} \\
&= \langle\sum_{i=1}^{2}q_{1,i}\cdot u_i, q_2\rangle
\end{align*}
$$

#### Analysis

The proof generation time depends on the encoding and commitment operations. Since Merkle Tree is used for commitments, as long as encoding is performed in linear time, the overall execution time is $$O(N)$$.

The proof structure consists of:

* $$c$$: Fields used in Testing Phase Step 2 and Evaluation Phase Step 3.
* $$r\cdot l$$: Fields sampled during Testing Phase Step 3 and Evaluation Phase Step 4.
* $$\alpha$$: Opening proof size required for $$l$$-queries.

Verification time is $$O(r \cdot l)$$. While proof generation is fast, the proof size and verification time is directly proportional with $$r$$.

### Linear-time Encoding

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfZI1M0Vnh7N-GZeYPdCdj_067IKSYhbX-U7hqXYdMB72p4CJERFnaaDI51xsbKH6Tn0K8AXIT2qwmabCMNG02SaQK5k_j1EfxPQ99nFuSwsppcRF8F0oGobx07Z-divHX7o-Bc?key=TV6kc-5CN5lS3U7bOqCI-uh5" alt=""><figcaption></figcaption></figure>

The figure above illustrates the linear-time encoding method used in Brakedown. For a given rate $$ho$$, $$\text{Enc}(x)$$ is a function that generates a vector of length $$n \cdot \rho^{-1}$$ from an input vector $$x$$ of length $$n$$. The output is divided into three parts: $$(x, z, v)$$. The encoding $$\text{Enc}(x)$$ proceeds as follows:

```cpp
Vec Enc(Vec x) {
  if (x.size() < 900) {
    // As mentioned in Section 5 of the paper, 
    // the threshold is set at 900. 
    // Quadratic-time operations are acceptable here 
    // since their time contribution is negligible.
    return RSEnc(x);
  }
  // A is a sparse matrix.
  Matrix A = SampleMatrixA();
  CHECK_EQ(A.rows(), x.size());
  CHECK_EQ(A.cols(), kAlpha * x.size());
  Vec y = x * A ;
  Vec z = Enc(y);
  // B is a sparse matrix.
  Matrix B = SampleMatrixB();
  CHECK_EQ(B.rows(), z.size());
  CHECK_EQ(B.cols(), (kRateInv * - 1 - kAlpha * kRateInv) * x.size()); 
  Vec v = z * B;

  return Concatenate(x, z, v);  
}
```

#### Properties of the Encoded Output

For the encoded result $$w = \mathsf{Enc}(x)$$, given the parameter $$\beta$$, the zero norm $$\|w\|_0$$ and distance $$\delta$$ are given by:

$$
\|w\|_0 > \beta n\text{, } \delta =  (\beta n) / (n\rho^{-1}) = \beta / r
$$

The zero norm, also known as[ Hamming weight](https://en.wikipedia.org/wiki/Hamming_weight), represents the number of non-zero elements in $$x$$:

$$
\|x\|_0 = \#\{i \, | \, x_i \neq 0\}
$$

The execution time of encoding is proportional to the input vector $$x$$'s length. Proofs related to this encoding method are detailed in Section 5.1 of the paper.

### Linear-time Commitments for Sparse Multilinear Polynomials

Using techniques from[ Spartan](https://eprint.iacr.org/2019/550.pdf), the input vector can be committed in linear time.

#### Representing the Vector as a Sparse Matrix

For example, the vector can be converted into a matrix as follows. Here, Rows = 4, Cols = 4, Non-zero Elements (N) = 4, Total Elements (M) = 16:

$$
[0, 2, 0, 0, 3, 0, 0, 0, 0, 0, 4, 0, 0, 0, 0, 5] \rightarrow
\begin{bmatrix}
0 & 2 & 0 & 0 \\
3 & 0 & 0 & 0\\
0 & 0 & 4 & 0\\
0 & 0 & 0 & 5\\
\end{bmatrix}
$$

Simply using a naive polynomial commitment scheme would require a computational complexity of $$O(M)$$.

#### Sparse Representation of the Matrix

The sparse representation is defined using $$R$$, $$C$$ and $$V$$ as the following:\
1\. Each row of $$R$$,$$C$$,$$V$$ defines a single non-zero entry in the sparse matrix.

2\. $$M_{Rᵢ,Cᵢ}=Vᵢ$$

$$
\begin{bmatrix}
0 & 2 & 0 & 0 \\
3 & 0 & 0 & 0\\
0 & 0 & 4 & 0\\
0 & 0 & 0 & 5\\
\end{bmatrix}

\rightarrow 

R = \begin{bmatrix}
0  \\
1 \\
2 \\
3 \\
\end{bmatrix},

C = \begin{bmatrix}
1  \\
0 \\
2 \\
3 \\
\end{bmatrix},

V = \begin{bmatrix}
2  \\
3 \\
4 \\
5 \\
\end{bmatrix}
$$

#### Defining Sparse Polynomial D

The sparse polynomial $$D$$ is defined as:

$$
D(r_x, r_y) = \sum_{k = \{0,1\}^{\log N}} \mathsf{val}(k)\cdot \widetilde{\mathsf{eq}}(b^{-1}(\mathsf{row}(k)), r_x)\cdot \widetilde{\mathsf{eq}}(b^{-1}(\text{col}(k)), r_y), \\
\text{where }\mathsf{val}(k) = V_k, \mathsf{row}(k) = R_k, \mathsf{col}(k) = C_k, \\ b^{-1}: \text{canonical injection from }\mathbb{F} \text{ to } \{0, 1\}^{\log M}, \\
\widetilde{\mathsf{eq}}(x, y) = 
\begin{cases}
    1,& \text{if } x = y\\
    0,& \text{otherwise}
\end{cases}
$$

Here is an example calculation of $$\text{row}$$, $$\text{col}$$, $$\text{val}$$ and $$b^{-1}$$ based on the previous example of a sparse matrix.

$$
\mathsf{row}((0, 0)) = 0, \mathsf{row}((0, 1)) = 1, \mathsf{row}((1, 0)) = 2, \mathsf{row}((1, 1)) = 3 \\
  \mathsf{col}((0, 0)) = 1, \mathsf{col}((0, 1)) = 0, \mathsf{col}((1, 0)) = 2, \mathsf{col}((1, 1)) = 3 \\
  \mathsf{val}((0, 0)) = 2, \mathsf{val}((0, 1)) = 3, \mathsf{val}((1, 0)) = 4, \mathsf{val}((1, 1)) =5 \\
  b^{-1}(0) = (0, 0), b^{-1}(1) = (0, 1), b^{-1}(2) = (1, 0), b^{-1}(3) = (1, 1)
$$

Therefore, $$D$$ will be calculated as so:

$$
D((0, 0), (0, 1)) = 2, 
D((0, 1), (0, 0)) = 3, \\D((1, 0), (1, 0)) = 4, D((1, 1), (1, 1)) = 5
$$

#### Commitment and Opening Steps

1. Given $$r_x$$ and $$r_y$$, the prover provides oracles and $$E_{r_y}$$:

$$
\forall k \in \{0, 1\}^{\log N},  \space E_{r_x}(k) = \widetilde{\mathsf{eq}}(b^{-1}(\mathsf{row}(k)), r_x) \\
\forall k \in \{0, 1\}^{\log N},  \space E_{r_y}(k) = \widetilde{\mathsf{eq}}(b^{-1}(\mathsf{col}(k)), r_y) \\
$$

2. The prover and verifier perform a sumcheck protocol for the equation:

$$
v \stackrel{?}= \sum_{k \in \{0, 1\}^{\log N}} \mathsf{val}(k) \cdot E_{r_x}(k) \cdot E_{r_y}(k)
$$

3. At the final round of sumcheck, the verifier checks the following

$$
v(r_z) \stackrel{?}= v_{\text{val}} \land E_{r_x}(r_z) \stackrel{?}= v_{E_{r_x}} \land E_{r_y}(r_z) \stackrel{?}= v_{E_{r_y}}
$$

To verify the prover's claims of $$E_{r_x}$$ and $$E_{r_y}$$, an **Offline Memory Checking** method is employed (refer to Detour: offline memory checking in Section 6 of the paper for details).

### Reduction of R1CS to Sumcheck

Brakedown is a proving scheme for R1CS. Since the sumcheck protocol is integral to Brakedown, R1CS must be reduced to a sumcheck format. R1CS is defined as:

$$
A\cdot Z \circ B \cdot Z = C \cdot Z\text{, where } A, B, C \in \mathbb{F}_q^{M\times M}\text{ and } Z \in \mathbb{F}_q^M
$$

This can be converted into a sumcheck format as follows (simplified for illustration; see Section 7 for details):

$$
0 \stackrel{?}= \sum_{x \in \{0, 1\}^{\log M}} \widetilde{\text{eq}}(r, x) \cdot [(\sum_{y \in \{0, 1\}^{\log M}} \widetilde{A}(x, y)\cdot \widetilde{Z}(y))\cdot (\sum_{y \in \{0, 1\}^{\log M}} \widetilde{B}(x, y)\cdot \widetilde{Z}(y)) - (\sum_{y \in \{0, 1\}^{\log M}} \widetilde{C}(x, y)\cdot \widetilde{Z}(y))]
$$

## Conclusion

Brakedown introduces linear-time encoding and linear-time commitment for generating proofs for R1CS, enabling proof generation with $$O(N)$$ field operations. However, as mentioned earlier, the proof size is large, and the verification time is slow. These characteristics are evident in the following results:

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXedA4vZNQjx56-U4Evr3f1JI4mU9dlP4Z6vjBEd6u1J8ppSgBrMKoem7fpwCEgH-D1w8v14GWVgiF2N5VYTtc9axULMxLcHtUiTzVaabKXMcrogmQ5uQaacBX_KiXOkwEfdctEe?key=TV6kc-5CN5lS3U7bOqCI-uh5" alt=""><figcaption></figcaption></figure>

The above figure, taken from the paper, benchmarks the performance of Brakedown's Polynomial Commitment Scheme. As shown, while Commit and Open operations are as fast as Ligero, the Verify and Communication stages are the slowest among the compared schemes.

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXe-RIw84V9kggKW_oTs59mpxlzcENzooFqjclmAc13P7ObKJLYUrZI19DZf_zTvcjx9tiD1_Ls0oLZjgH9BwrBhaj0-msVFQSDz5e-rSRF6T9tX46FaeJPBOAlCPGgIoA9hRaI?key=TV6kc-5CN5lS3U7bOqCI-uh5" alt=""><figcaption></figcaption></figure>

This figure, also taken from the paper, shows that Brakedown achieves the fastest Prove time and is among the fastest for Encode time. However, it also confirms that Verify remains slow, and the Proof Size is the largest.

Brakedown has since been improved and used in subsequent works, including[ Orion](https://eprint.iacr.org/2022/1010.pdf),[ Orion+](https://eprint.iacr.org/2022/1355.pdf),[ Vortex](https://eprint.iacr.org/2024/185.pdf), and[ Binius](https://eprint.iacr.org/2023/1784).

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from A41
