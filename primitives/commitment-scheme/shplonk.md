# SHPlonk

## Introduction

The KZG commitment was introduced in [KZG10](https://www.iacr.org/archive/asiacrypt2010/6477178/6477178.pdf). The time required for commitment is proportional to the degree of the polynomial, and the commitment consists of a single $$\mathbb{G}_1$$ element, which is $$C = [P(s)]_1$$ for given $$P(X)$$.

The opening proof $$W$$ for a point $$x$$ is defined as:

$$
W = \left[\frac{P(s) - y}{s - x} \right]_1
$$

Verification is performed using the following equation:

$$
e(W, [s]_2 -[x]_2) \cdot e(C - [y]_1,[1]_2) \stackrel{?}= 1
$$

However, when proving over AIR, multiple polynomials need to be opened at $$x$$ and $$\omega \cdot x$$. In PLONK-based schemes that support custom gates, such as Halo2, polynomials must be opened at various opening points in different combinations. One of the efficient methods for proving in batch settings is **SHPlonk**, which was introduced in [BDFG20](https://eprint.iacr.org/2020/081.pdf).

## Background

We'll now see how [GWC19](https://eprint.iacr.org/2019/953.pdf) opens multiple polynomials at multiple points. Let's start with simple cases.

### Opening Multiple Polynomials at a Single Point

Suppose two polynomials, $$P_0(X)$$ and $$P_1(X)$$, are opened at $$x$$ with corresponding values $$y_0$$ and $$y_1$$.

1. The verifier provides $$\alpha \leftarrow \mathbb{F}$$ to the prover.
2. The prover generates the following opening proof $$W$$:

$$
W = \left[ \frac{P(s) - y}{s- x} \right]_1
$$

Where $$P(X) := P_0(X) + \alpha \cdot P_1(X)$$ and $$y := y_0 + \alpha \cdot  y_1$$.

Verification is performed using the following equation:

$$
e(W, [s]_2 - [x]_2) \cdot e(C- [y]_1, [-1]_2)\stackrel{?}=1
$$

Where $$C := C_0 + \alpha \cdot C_1$$ and $$C_0, C_1$$ are commitments of $$P_0(X), P_1(X)$$ respectively.&#x20;

This approach is almost identical to opening a single polynomial at a single point, with the only difference being the random linear combination applied to the polynomials before constructing the proof.

### Opening a Single Polynomial at Multiple Points

Suppose a polynomial $$P(X)$$ is opened at two points, $$x_0$$ and $$x_1$$, with corresponding values $$y_0$$ and $$y_1$$.

The prover generates the following opening proof $$W$$:

$$
W = \left[ \frac{P(s) - r(s)}{(s- x_0)(s - x_1)} \right]_1
$$

Where $$r(X)$$ is defined as:

$$
r(X) := \begin{cases}
P(x_0) & \text{ if } X = x_0 \\
P(x_1) & \text{ if } X = x_1
\end{cases}
$$

Verification is performed using the following equation:

$$
e(W, [(s- x_0)(s - x_1)]_2) \cdot e(C - [r(s)]_1, [-1]_2) \stackrel{?}= 1
$$

Where $$C$$ is the commitment of $$P(X)$$.

There are two key issues with this naive approach:

1. **Increasing** $$\mathbb{G}_2$$ **​ parameters**: As the number of opening points increases, the required $$\mathbb{G}_2$$ parameters also increase. For instance, in this case, $$[s^2]_2$$ is required.&#x20;
2. **Expensive group operation in** $$\mathbb{G}_2$$: Operations in $$\mathbb{G}_2$$ are computationally expensive, leading to higher gas consumption in on-chain, and inefficient in recursive proof generation.

To solve these issues, we modify the protocol as follows:

The prover generates 2 opening proofs, $$W_0$$ and $$W_1$$:

$$
W_0 = \left[ \frac{Q_0(s)}{s - x_0} \right], \quad W_1 = \left[ \frac{Q_1(s)}{s - x_1} \right]
$$

Where $$Q_0(X) := P(X) - y_0$$ and $$Q_1(X) := P(X) - y_1$$.

The verifier uses $$\alpha \leftarrow \mathbb{F}$$ to verify the following equation:

$$
e([Q_0(s)]_1 + \alpha \cdot [Q_1(s)]_1 + x_0 \cdot W_0 + \alpha \cdot x_1 \cdot W_1, [1]_2) \cdot e(-W_0 - \alpha \cdot W_1, [s]_2) \stackrel{?}= 1
$$

The verifier can obtain $$[Q_0(s)]_1, [Q_1(s)]_1$$ by computing $$C - [y_0]_1$$ and $$C - [y_1]_1$$ respectively.

**Proof.**

By extracting the exponent components, it can be confirmed that the verification equation holds.

$$
\begin{align*}
&Q_0(s) + \alpha \cdot Q_1(s) + (x_0 - s) \cdot \frac{Q_0(s)}{s - x_0} + \alpha \cdot (x_1 - s) \cdot \frac{Q_1(s)}{s - x_1} \\
= &Q_0(s) + \alpha \cdot Q_1(s) - Q_0(s) - \alpha \cdot Q_1(s) \\
= &0
\end{align*}
$$

Now the number of required $$\mathbb{G}_2$$ parameters is now constant, and the group operations in $$\mathbb{G}_2$$ are eliminated.

### Opening Multiple Polynomials at Multiple Points

In [GWC19](https://eprint.iacr.org/2019/953.pdf), a method that combines the previous techniques is proposed. Consider the following scenario:

* $$P_0(X)$$ is opened at $$x_0$$ with value $$y_0$$.
* $$P_1(X)$$ is opened at $$x_0, x_1$$ with values $$y_1, y_2$$.
* $$P_2(X)$$ is opened at $$x_0, x_1$$ with values $$y_3, y_4$$.

1. The verifier provides $$\alpha \leftarrow \mathbb{F}$$ to the prover.
2. The prover generates 2 opening proofs, $$W_0$$ and $$W_1$$:

$$
W_0 = \left[\frac{Q_0(s)}{s - x_0} \right]_1, \quad W_1 = \left[ \frac{Q_1(s)}{s - x_1} \right]_1
$$

Where $$Q_0(X) := P_0(X) - y_0 + \alpha(P_1(X) - y_1) + \alpha^2(P_2(X) - y_3)$$ and $$Q_1(X) := P_1(X) - y_2 + \alpha(P_2(X) - y_4)$$.

Verification is performed with $$\beta \leftarrow \mathbb{F}$$ using the following equation:

$$
e([Q_0(s)]_1 + \beta \cdot [Q_1(s)]_1 + x_0 \cdot W_0 + \beta \cdot x_1 \cdot W_1, [1]_2) \cdot e(-W_0 - \beta \cdot W_1, [s]_2) \stackrel{?}= 1
$$

The verifier can obtain $$[Q_0(s)]_1, [Q_1(s)]_1$$ by computing $$\alpha(\alpha(C_2 - [y_3]_1) + C_1 - [y_1]_1) + C_0 - [y_0]_1$$ and $$\alpha(C_2 - [y_4]_1) + C_1 - [y_2]_1$$ respectively, where $$C_0, C_1, C_2$$ are the commitments of $$P_0(X), P_1(X), P_2(X)$$.

If there are $$n$$ polynomials of degree $$d$$, and each polynomial is opened at $$t$$ points, the following holds:

* $$\mathsf{srs}$$ **size**: $$(d + 1) \mathbb{G}_1, 2\mathbb{G}_2$$
  * $$\mathsf{srs}$$ stands for "structure reference string".
* **Prover work**: $$O( d \cdot t)\mathbb{G}_1, O(t \cdot d \log d) \mathbb{F}$$
  * To compute each division in $$\frac{Q_i(X)}{X - x_i}$$, it takes $$O(d \log d)  \mathbb{F}$$ operations.
  * To compute each proof $$W_i$$, it takes $$O(d) \mathbb{G}_1$$ operations.
* **Proof length**: $$t\mathbb{G}_1$$
* **Verifier work**: $$O(n \cdot t) \mathbb{G}_1, 2 \mathsf{P}$$
  * To compute all of $$[Q_i(s)]_1$$, it takes $$O(n \cdot t) \mathbb{G}_1$$ operations.

Here, $$\mathbb{G}_i$$ represents scalar multiplication in $$\mathbb{G}_i$$​, $$\mathbb{F}$$ represents addition or multiplication in $$\mathbb{F}$$, and $$\mathsf{P}$$ denotes a pairing operation.

SHPlonk enables a smaller proof length and faster verifier work for this case.

## Protocol Explanation

### Version 1

To illustrate this scheme, let's use the same example as before. If we group the opening points, we can form two sets: $$S_0 = \{x_0\}$$ and $$S_1 = \{x_0, x_1\}$$. The complete set of opening points is then $$T = \{x_0, x_1\}$$.&#x20;

The zero polynomial $$Z_{S}(X)$$ on $$S = \{x_0, \dots, x_{n-1}\}$$ is defined as:

$$
Z_S(X) := (X - x_0) \cdots (X - x_{n-1})
$$

The opening proof $$W$$ is constructed as follows:

1. The verifier provides $$\alpha, \beta \leftarrow \mathbb{F}$$ to the prover.
2. The prover computes $$W = [h(s)]_1$$ as follows:

$$
h(X) := \frac{Q_0(X)}{Z_{S_0}(X)} + \beta \cdot \frac{Q_1(X)}{Z_{S_1}(X)}
$$

Where $$Q_0(X) := P_0(X) - y_0, Q_1(X) := P_1(X) - r_0(X) + \alpha(P_2(X) - r_1(X)), r_0(X),$$$$r_0(X)$$ and $$r_1(X)$$ are defined as follows:

$$
r_0(X) := \begin{cases}
P_1(x_0) & \text{ if } X = x_0 \\
P_1(x_1) & \text{ if } X = x_1
\end{cases},
\quad
r_1(X) := \begin{cases}
P_2(x_0) & \text{ if } X = x_0 \\
P_2(x_1) & \text{ if } X = x_1
\end{cases}
$$

To verify the opening proof $$W$$, the following steps are performed:

1. Compute $$Z_0$$ and $$Z_1$$ using $$Z_i := [Z_{T \setminus S_i}(s)]_2$$:

$$
Z_0 = [s]_2 - [x_1]_2, \quad Z_1 =[1]_2
$$

2. Verify using the following equation:

$$
e([Q_0(s)]_1, Z_0)\cdot e(\beta \cdot [Q_1(s)]_1, Z_1) \stackrel{?}= e(W, [Z_T(s)]_2)
$$

The verifier can obtain $$[Q_0(s)]_1, [Q_1(s)]_1$$ by computing $$C_0 - [y_0]_1$$ and $$\alpha(C_2 - [r_1(s)]_1) + C_1 - [r_0(s)]_1$$ respectively, where $$C_0, C_1, C_2$$ are commitments of $$P_0(X), P_1(X), P_2(X)$$.

**Proof.**

Extracting the exponent components from the above equation, we obtain:

$$
\begin{align*}
&Q_0(s) \cdot Z_{T \setminus S_0}(s) + \beta \cdot Q_1(s) \cdot Z_{T \setminus S_1}(s) \\
=&\left(\frac{Q_0(s)}{Z_{S_0}(s)} + \beta \cdot \frac{Q_1(s)}{Z_{S_1}(s)}\right) \cdot Z_T(s) \\
=&h(s) \cdot Z_T(s)
\end{align*}
$$

Thus, the verification equation holds.

If there are $$n$$ polynomials of degree $$d$$, and each polynomial is opened at $$t$$ points, the computational costs are as follows:

* **The number of opening sets** $$s$$: $$1 \le s \le n$$
* $$\mathsf{srs}$$ **size**: $$(d + 1) \mathbb{G}_1, (t + 1)\mathbb{G}_2$$
* **Prover work**: $$O( d )\mathbb{G}_1, O(d \cdot n + s \cdot d \log d) \mathbb{F}$$
  * This doesn't account for computation of each $$r_i(X)$$, since they are normally small operations.
  * To compute all of  $$Q_i(X)$$, it takes $$O(d \cdot n) \mathbb{F}$$ operations.
  * To compute each $$\frac{Q_i(X)}{Z_{S_i}(X)}$$, it takes $$O(d \log d) \mathbb{F}$$ operations.
  * To compute a proof $$W$$, it takes $$O(d) \mathbb{G}_1$$ operations.
* **Proof length**: $$1\mathbb{G}_1$$
* **Verifier work**: $$O(n ) \mathbb{G}_1, O(s \cdot t) \mathbb{G}_2, (s+1 )\mathsf{P}$$
  * To compute all of $$[Q_i(s)]_1$$, it takes $$O(n)\mathbb{G}_1$$ operations.
  * To compute $$[Z_T(s)]_2$$, it takes $$O(t)\mathbb{G}_2$$ operations.
  * To compute each $$Z_i$$, it takes $$O(t)\mathbb{G}_2$$ operations.

Here, $$\mathbb{G}_i$$ represents scalar multiplication in $$\mathbb{G}_i$$​, $$\mathbb{F}$$ represents addition or multiplication in $$\mathbb{F}$$, and $$\mathsf{P}$$ denotes a pairing operation.

Now the same issues mentioned in [Opening a Single Polynomial at Multiple Points](shplonk.md#opening-a-single-polynomial-at-multiple-points) arise again. Since $$[Z_T(s)]_2$$ is the primary cause of these issues, we need to devise a more efficient approach—one that treats the opening as if it were occurring at a single point rather than $$t$$ points.

During the presentation, [BATZORIG ZORIGOO](https://app.gitbook.com/u/lqk5Tx9zY4XYVRfF3ReRDiOhbxG3 "mention") asked a great question: _"What happens if all polynomials are opened at_ $$T$$_?"_ Since this simplifies $$s$$ to 1, both prover and verifier work can be reduced. However, there are two main issues:

1. **Prover Complexity:** The prover must evaluate the polynomials at every point in $$T$$, which increases the number of opening points to $$n \cdot t$$.
2. **ZK Proof Size:** The ZK proof size increases because it must include all the opened values.

It's important to distinguish between two types of proofs here. The **Proof** mentioned earlier refers specifically to proving polynomial openings. However, in this context, the **ZK Proof** not only includes the opening proof but also contains commitments and the opened values themselves.

### Version 2

Version 2 addresses the issues from the previous approach by introducing one additional proof.

The opening proofs $$W , W'$$ are constructed as follows:

1. The verifier provides $$\alpha, \beta \leftarrow \mathbb{F}$$ to the prover.
2. The prover computes $$W = [h(s)]_1$$ as follows:

$$
h(X) := \frac{f(X)}{Z_T(X)}
$$

Where $$f(X)$$ is defined as:

$$
f(X) := Z_{T \setminus S_0}(X) \cdot Q_0(X) + \beta \cdot Z_{T \setminus S_1}(X) \cdot Q_1(X)
$$

where $$Q_0(X) := P_0(X) - y_0, Q_1(X) := P_1(X) - r_0(X) + \alpha(P_2(X) - r_1(X)), r_0(X),$$$$r_0(X)$$ and $$r_1(X)$$ are defined as follows:

$$
r_0(X) := \begin{cases}
P_1(x_0) & \text{ if } X = x_0 \\
P_1(x_1) & \text{ if } X = x_1
\end{cases},
\quad
r_1(X) := \begin{cases}
P_2(x_0) & \text{ if } X = x_0 \\
P_2(x_1) & \text{ if } X = x_1
\end{cases}
$$

3. The verifier provides $$z \leftarrow \mathbb{F}$$ to the prover.
4. The prover computes $$W'$$ as follows:

$$
W' = \left[ \frac{L(s)}{s- z} \right]_1
$$

Where $$L(X)$$ is defined as:

$$
L(X) := f_z(X) - Z_T(z) \cdot h(X)
$$

Here, $$f_z(X)$$ is defined as:

$$
f_z(X) := Z_{T \setminus S_0}(z)(P_0(X) - y_0) + \beta \cdot Z_{T \setminus S_1}(z) (P_1(X) - r_0(z) + \alpha(P_2(X) - r_1(z)))
$$

The polynomial $$L(X)$$ is crucial in solving the issues from the previous approach. Unlike $$Q_i(X)$$, which vanishes at multiple points, this $$L(X)$$ now vanishes at single point at $$z$$ because:&#x20;

$$
L(z) = f(z) - Z_T(z) \cdot h(z) = 0
$$

To verify the opening proofs, the verifier checks the following equation:

$$
e([f_z(s)]_1 - Z_T(z) \cdot W, [1]_2) \stackrel{?}= e(W', [s]_2 - [z]_2)
$$

The verifier can obtain $$[f_z(s)]_1$$ as follows:

$$
[f_z(s)]_1 = \beta \cdot Z_{T \setminus S_1}(z) (\alpha(C_2 - [r_1(z)]_1) + C_1 - [r_0(z)]_1) + Z_{T \setminus S_0}(z)(C_0 - [y_0]_1)
$$

Where $$C_0, C_1, C_2$$ are commitments of $$P_0(X), P_1(X), P_2(X)$$.

**Proof.**

Extracting the exponent terms from the above equation, we obtain:

$$
\begin{align*}
&f_z(s) - Z_T(z) \cdot h(s)  \\
=& L(s) \\
=&\frac{L(s)}{s-z} \cdot (s - z)
\end{align*}
$$

Thus, the verification equation holds.

If there are $$n$$ polynomials of degree $$d$$, and each polynomial is opened at most $$t$$ points, the computational costs are as follows:

* **The number of opening sets** $$s$$: $$1 \le s \le n$$
* $$\mathsf{srs}$$ **size**: $$(d+1) \mathbb{G}_1, 2\mathbb{G}_2$$
* **Prover work**: $$O( d )\mathbb{G}_1, O(d \cdot n + s \cdot d \log d) \mathbb{F}$$
  * To compute all of  $$Q_i(X)$$, it takes $$O(d \cdot n) \mathbb{F}$$ operations.
  * To compute each $$Z_{T\setminus S_{i}} \cdot Q_i(X)$$, it takes $$O(d \log d)\mathbb{F}$$ operations.
  * To compute $$\frac{f(X)}{Z_T(X)}$$ and $$\frac{L(X)}{X - z}$$, it takes $$O(d \log d) \mathbb{F}$$ operations.
  * To compute $$L(X)$$, it takes $$O(d \cdot n)$$ operations.
  * To compute a proof $$W, W'$$, it takes $$O(d) \mathbb{G}_1$$ operations.
* **Proof length**: $$2\mathbb{G}_1$$
* **Verifier work**: $$O(n ) \mathbb{G}_1, O(1)\mathbb{G}_2,2 \mathsf{P}$$
  * To compute all of $$[f(z)]_1$$, it takes $$O(n)\mathbb{G}_1$$ operations.

Here, $$\mathbb{G}_i$$ represents scalar multiplication in $$\mathbb{G}_i$$​, $$\mathbb{F}$$ represents addition or multiplication in $$\mathbb{F}$$, and $$\mathsf{P}$$ denotes a pairing operation.

#### Optimization on Version 2.

When constructing $$W'$$, it can be "**normalized**", to save one of the scalar multiplications like below.

$$
W' = \left[ \frac{L(s)}{Z_{T \setminus S_0}(z) \cdot (s- z)} \right]_1
$$

Then, the verification logic is changed accordingly:

$$
e([\frac{f_z(s)}{Z_{T \setminus S_0}(z)}]_1 - \frac{Z_T(z)}{Z_{T \setminus S_0}(z)} \cdot W, [1]_2) \stackrel{?}= e(W', [s]_2 - [z]_2)
$$

The verifier can obtain $$\frac{[f_z(s)]_1}{Z_{T \setminus S_0}(z)}$$ as follows:

$$
\frac{[f_z(s)]_1}{Z_{T \setminus S_0}(z)} = \beta \cdot \frac{Z_{T \setminus S_1}(z)}{Z_{T \setminus S_0}(z)} (\alpha(C_2 - [r_1(z)]_1) + C_1 - [r_0(z)]_1) +(C_0 - [y_0]_1)
$$

Notice that the rightmost part of the equation doesn't have scalar multiplication any more.

## Conclusion

By summarizing the key metrics in the following table, we can see that SHPlonk demonstrates superior efficiency compared to GWC19.

|                    | SRS size                                     | Prover work                                                        | Prover length     | Verifier work                                                       |
| ------------------ | -------------------------------------------- | ------------------------------------------------------------------ | ----------------- | ------------------------------------------------------------------- |
| GWC19              | $$(d + 1) \mathbb{G}_1, 2\mathbb{G}_2$$      | $$O(d \cdot t)\mathbb{G}_1, O(t \cdot d \log d) \mathbb{F}$$       | $$t\mathbb{G}_1$$ | $$O(n \cdot t) \mathbb{G}_1, 2 \mathsf{P}$$                         |
| SHPlonk version 1. | $$(d + 1) \mathbb{G}_1,(t + 1)\mathbb{G}_2$$ | $$O( d )\mathbb{G}_1, O(d \cdot n + s \cdot d \log d) \mathbb{F}$$ | $$1\mathbb{G}_1$$ | $$O(n ) \mathbb{G}_1, O(s \cdot t) \mathbb{G}_2, (s+1) \mathsf{P}$$ |
| SHPlonk version 2. | $$(d+1) \mathbb{G}_1, 2\mathbb{G}_2$$        | $$O( d )\mathbb{G}_1, O(d \cdot n + s \cdot d \log d) \mathbb{F}$$ | $$2\mathbb{G}_1$$ | $$O(n ) \mathbb{G}_1, O(1)\mathbb{G}_2, \\2 \mathsf{P}$$            |

SHPlonk offers a more efficient proof system, balancing prover and verifier costs while keeping proof sizes minimal. Depending on the use case, Version 1 is preferable for minimizing proof length, whereas Version 2 provides a more scalable solution with constant $$\mathbb{G}_2$$ parameters and a fixed number of pairing operations while removing scalar multiplications in $$\mathbb{G}_2$$.

## References

* [https://eprint.iacr.org/2020/081.pdf](https://eprint.iacr.org/2020/081.pdf)

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze
