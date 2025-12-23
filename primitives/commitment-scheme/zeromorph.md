# Zeromorph

## Introduction

Aztec uses [Sumcheck](../sumcheck/) to prove the proof of **circuit satisfiability**. In Sumcheck, a multilinear polynomial is used. Therefore, it is necessary to efficiently commit to and open this multilinear polynomial. Additionally, since Aztec is a privacy-focused Layer 2, an efficient way to achieve zero-knowledge (ZK) is also required. To address this, Aztec introduces a method called Zeromorph, which is currently implemented in [Aztec Zeromorph](https://github.com/AztecProtocol/aztec-packages/tree/98cd1bb/barretenberg/cpp/src/barretenberg/commitment_schemes/zeromorph).

## Background

### KZG

1. $$\mathsf{KZG.Setup}(1^\lambda, N_{\mathsf{max}}) \rightarrow ([1]_1, [s]_1, \dots, [s^{N_{\mathsf{max}}}]_1, [1]_2, [s]_2)$$
2. $$\mathsf{KZG.Commit}(f \in F[X]^{N_{\mathsf{max}}}) \rightarrow C = [f(s)]_1$$
3. $$\mathsf{KZG.ProveEval}(f, z, v) \rightarrow W = \left[\frac{f(s) - v}{s - z}\right]_1$$
4. $$\mathsf{KZG.VerifyEval}(C, W, z, v) \rightarrow b \in \{0, 1\}$$:

The verifier checks the following equation:

$$
e(C - [v]_1, [1]_2) \stackrel{?}=e(W, [s]_2 - [z]_2)
$$

Here, the verifier learns the information $$(C, W, v)$$, which presents two issues:

1. Since KZG is a **computationally hiding commitment**, if quantum computing becomes feasible, revealing $$(C, W)$$ may no longer satisfy the ZK property.
2. While it proves **circuit satisfiability**, it also reveals information of the polynomial with $$P(z) = v$$.

### Hiding KZG

Instead of the naive KZG scheme outlined above, Zeromorph uses a more secure variant named "Hiding KZG."

#### $$\mathsf{HidingKZG.Setup}(1^\lambda, N_{\mathsf{max}}) \rightarrow ([1]_1, [s]_1, \dots, [s^{N{\mathsf{max}}}]_1, [\xi]_1, [1]_2, [s]_2, [\xi]_2)$$

#### $$\mathsf{HidingKZG.Commit}(f \in \mathsf{F}[X]^{<N_{\mathsf{max}}}) \rightarrow (C, r)$$:

1. The prover samples $$r \leftarrow \mathbb{F}$$.
2. The prover constructs $$C = [f(s)]_1 + r \cdot [\xi]_1$$.
3. The prover can send only $$C$$ to the verifier.

#### $$\mathsf{HidingKZG.Open}(C, f, r) \rightarrow b \in \{0, 1\}$$:

This checks the following equation:

$$
C \stackrel{?}= [f(s)]_1 + r \cdot [\xi]_1
$$

#### $$\mathsf{HidingKZG.ProveEval}(C, r, z, v) \rightarrow (W, \delta)$$:

1. The prover samples $$\alpha \leftarrow \mathbb{F}$$.
2. The prover provides the verifier with $$W$$ and $$\delta$$:

$$
W = \left[\frac{f(s) - v}{s - z}\right]_1 + \alpha \cdot [\xi]_1, \quad \delta = [r - \alpha (s - z)]_1
$$

#### $$\mathsf{HidingKZG.VerifyEval}(C, W, \delta, z, v) \rightarrow b \in \{0, 1\}$$:

The verifier checks the following equation:

$$
e(C - [v]_1, [1]_2) \stackrel{?}=e(W, [s]_2 - [z]_2) \cdot e(\delta, [\xi]_2)
$$

**Proof.**

$$
\begin{align*}
f(s) + r \xi - v&= f(s) -v + \alpha \xi(s-z) + (r- \alpha(s-z))\xi \\
&= f(s) - v + r\xi
\end{align*}
$$

This is only satisfied iff $$\mathsf{HidingKZG.Open}(C, f, r) = 1 \land f(z) = v$$.

In comparison to naive KZG, **HidingKZG**'s verifier learns of an additional $$\delta$$ that allows **HidingKZG** to stand up against the issues of naïve KZG in the following ways:

1. Since **HidingKZG** is a **perfectly hiding** commitment, even if quantum computing becomes feasible, revealing $$(C, W, \delta)$$ still satisfies the ZK property.
2. However, it still reveals $$P(z) = v$$ while proving **circuit satisfiability**.

Later in this document, we explain an additional trick that ensures that only information about **circuit satisfiability** is revealed.

During the presentation, [이태훈](https://app.gitbook.com/u/r90jaIdBK3e1D7UbVBxgVIP7qIf1 "mention")asked a great question: "Why isn't the blinding technique introduced in the Plonk paper sufficient for the ZK property?". After internal discussion, we concluded that it requires more randomness and does not provide protection against post-quantum threats.

### Multilinear Polynomial Identity

Zeromorph requires the use of the Sumcheck protocol, which is reduced to an evaluation claim for a multilinear polynomial $$f \in \mathbb{F}[X_0, \dots, X_{n-1}]^{\le1}$$:

$$
\sum_{\bm{b} \in \{0, 1\}^n} f(\bm{b}) = 0 \rightarrow f(\bm{z}) = v
$$

where $$\bm{z} = (z_0, \dots, z_{n-1}) \in \mathbb{F}^n$$ and $$v \in \mathbb{F}$$.

By [**Lemma 2.3.1**](https://eprint.iacr.org/2023/917.pdf#page=9\&zoom=100,180,400), The **multilinear polynomial identity** is defined as follows:

$$
\begin{align*}
f - v = \sum_{i=0}^{n-1}(X_i - z_i)q_i  && (1)
\end{align*}
$$

If there exist $$q_i$$ as defined below that satisfies the above, then $$f(\bm{z}) = v$$.

* For $$i = 0$$: $$q_i \in \mathbb{F}$$
* For $$0 < i < n$$: $$q_i \in \mathbb{F}[X_0, \dots, X_{i-1}]^{\le1}$$

**Proof:**

If $$n = 1$$, equation $$(1)$$ reduces to the following univariate polynomial identity:

$$
f - v = (X_0 - z_0)q_0
$$

Thus, if $$q_0 \in \mathbb{F}$$, then $$f(z_0) = v$$.

If $$n > 1$$, $$f$$ is divided by $$(X_{n-1} - z_{n-1})$$ results in quotient $$q_{n-1}$$ with remainder $$f(X_0, \dots, X_{n-2}, z_{n-1})$$ as follows:

$$
f(X_0, \dots, X_{n-1}) = (X_{n-1} - z_{n-1})q_{n-1} + f(X_0, \dots, X_{n-2}, z_{n-1})
$$

The polynomial $$f(X_0, \dots, X_{n-1}) - f(X_0, \dots, X_{n-2}, z_{n-1})$$ has a degree of at most 1 in all variables. Therefore, $$q_{n-1}$$ must have a degree of at most 1 in $$X_0, \dots, X_{n-2}$$, and a degree of at most 0 in $$X_{n-1}$$. This implies $$q_{n-1} \in \mathbb{F}[X_0, \dots ,X_{n-2}]^{\le1}$$. Using this information and recursive decomposition, we derive the following:

$$
f(X_0, \dots, X_{n-2}, z_{n-1}) = (X_{n-2} - z_{n-2})q_{n-2} + f(X_0, \dots, X_{n-3}, z_{n-2}, z_{n-1}) \\
\dots \\
f(X_0, z_1, \dots z_{n-1}) = (X_0 - z_0) q_0 + f(z_0, \dots, z_{n-1})
$$

Iteratively substituting all the equations above yields:

$$
f  = \sum_{i=0}^{n-1}(X_i - z_i)q_i + f(z_0, \dots, z_{n-1})
$$

Therefore, by equation $$(1)$$, we confirm $$f(z_0, \dots, z_{n-1}) = f(\bm{z}) =v$$.&#x20;

#### **Computing** $$q_i$$

According to [**Lemma 2.3.1**](https://eprint.iacr.org/2023/917.pdf#page=9\&zoom=100,180,400) in [the Zeromorph paper](https://eprint.iacr.org/2023/917.pdf), $$q_i$$ can be computed as:

$$
q_i = f(X_0, \dots, X_{i-1}, z_i + 1, z_{i+1}, \dots, z_{n-1}) - f(X_0, \dots, X_{i-1}, z_i, z_{i+1}, \dots, z_{n-1})
$$

**Example:**

Let's calculate $$q_1$$, the quotient from dividing $$f(X_0, X_1, z_2)$$ by $$(X_1 - z_1)$$:

$$
q_1 = f(X_0, z_1 + 1, z_2) - f(X_0, z_1, z_2)
$$

$$f(X_0, z_1 + 1, z_2)$$ will be evaluated as follows:

$$
\begin{align*}f(X_0, z_1 + 1, z_2) = c_0X_0(z_1 + 1)z_2 + c_1X_0(z_1 + 1) + c_2X_0z_2 + c_3(z_1 + 1)z_2 + c_4X_0 + c_5(z_1 + 1) + c_6z_2 + c_7  && (a)\end{align*}
$$

$$f(X_0, z_1, z_2)$$ will be evaluated as follows:

$$
\begin{align*}f(X_0, z_1, z_2) = c_0X_0z_1z_2 + c_1X_0z_1 + c_2X_0z_2 + c_3z_1z_2 + c_4X_0 + c_5z_1 + c_6z_2 + c_7  && (b)\end{align*}
$$

Subtracting equation $$(b)$$ from equation $$(a)$$ allows us to compute $$q_1$$ as the following:

$$
\begin{aligned}
    f(X_0, z_1 + 1, z_2) - f(X_0, z_1, z_2) &= c_0X_0(z_1 + 1)z_2 + c_1X_0(z_1 + 1) + c_2X_0z_2 + c_3(z_1 + 1)z_2 \\
    &\quad + c_4X_0 + c_5(z_1 + 1) + c_6z_2 + c_7 \\
    &\quad - c_0X_0z_1z_2 - c_1X_0z_1 - c_2X_0z_2 - c_3z_1z_2 \\
    &\quad - c_4X_0 - c_5z_1 - c_6z_2 - c_7 \\
    &= c_0X_0z_2 + c_1X_0 + c_3z_2 + c_5.
\end{aligned}
$$

Now we can see that $$f(X_0, X_1, z_2)$$ can be decomposed as the following:

$$
\begin{align*}
    f(X_0, X_1, z_2) &= (X_1 - z_1) \underbrace{(c_0X_0z_2 + c_1X_0 + c_3z_2 + c_5)}_{q_1 } + f(X_0, z_1, z_2) \\
\end{align*}
$$

The validity of this statement can be ensured by multiplying all terms out

$$
\begin{align*}
    f(X_0, X_1, z_2) &= c_0X_0X_1z_2 + c_1X_0X_1 + c_3X_1z_2 + c_5X_1 \\
    &\quad - c_0X_0z_1z_2 - c_1X_0z_1 - c_3z_1z_2 - c_5z_1 \\
    &\quad + c_0X_0z_1z_2 + c_1X_0z_1 + c_2X_0z_2 + c_3z_1z_2 \\
    &\quad + c_4X_0 + c_5z_1 + c_6z_2 + c_7 \\
    &= c_0X_0X_1z_2 + c_1X_0X_1 + c_2X_0z_2 + c_3X_1z_2 \\
    &\quad + c_4X_0 + c_5X_1 + c_6z_2 + c_7.
\end{align*}
$$

#### **Verifying the Multilinear Polynomial Identity**

Verifying equation $$(1)$$ requires $$n$$ pairing operations, which imposes a computational overhead on the verifier:

$$
e([f(s_0, \dots, s_{n-1})]_1 - [v]_1,[1]_2) \stackrel{?}= \prod_{i = 0}^{n-1} e([q_i(s_0, \dots, s_{i-1})]_1, [s_i]_2 - [z_i]_2)
$$

Additionally, $$q_i \in \mathbb{F}[X_0, \dots ,X_{i-1}]^{\le1}$$ must be verified, which will be checked using the [**degree check protocol**](zeromorph.md#degree-check-protocol) introduced later.

## Protocol Explanation

### Multilinear Polynomial Identity using Univariate PCS

To efficiently verify the multilinear polynomial identity, we first define the following mapping:

$$
\mathcal{U}_n: \mathbb{F}[X_0, \dots, X_{n-1}]^{\le1} \rightarrow \mathbb{F}[X]^{<2^n}
$$

Given $$f(\bm{X})$$ defined as:

$$
f(\bm{X}) := \sum_{\bm{b} \in \{0, 1\}^n}v_{\bm{b}} \cdot \mathsf{\widetilde{eq}}(\bm{X}, \bm{b})
$$

the function $$\mathcal{U}_n(f)$$ transforms the following expression:

$$
\mathsf{\widetilde{eq}}(\bm{X}, \bm{b}) =\prod_{i = 0}^{n-1} (b_i  X_i +(1-b_i)(1 - X_i))  \rightarrow \prod_{i=0}^{n-1} \left(X^{2^i}\right)^{b_i}
$$

This seemingly expensive operation actually introduces no computational overhead. This is because the witness generation process for constructing a ZK proof already outputs polynomial evaluations, making this mapping essentially free.

If $$f = c \in \mathbb{F}$$ is a constant function, we can compute:

$$
\mathcal{U}_n(f) = c \cdot \Phi_n(X)
$$

where $$\Phi_n(X)$$ is defined as:

$$
\Phi_n(X) :=\sum_{i = 0}^{2^n - 1}X^i
$$

This satisfies the following equation, meaning that evaluating $$\Phi_n(X)$$ at any point $$x$$ requires only $$n + 1$$ multiplications and $$2$$ additions:

$$
\Phi_n(X) = \frac{X^{2^n} - 1}{X - 1}
$$

For example, if $$n = 2$$, we can compute:

$$
\mathcal{U}_2(f) = f(0, 0) + f(1, 0)X + f(0, 1)X^2 + f(1, 1)X^3
$$

Applying this mapping to both sides of equation $$(1)$$, we transform it into:

$$
\begin{align*}
\mathcal{U}_n(f) - v \cdot \mathcal{U}_n(1) = \sum_{i=0}^{n-1}\mathcal{U}_n(X_iq_i)- z_i \cdot \mathcal{U}_n(q_i) && (2)
\end{align*}
$$

Since $$\mathcal{U}_n(f)$$ is a linear isomorphism, satisfying equation $$(2)$$ implies equation $$(1)$$ is also satisfied. Instead of using a multilinear PCS to commit to $$f$$, we can now use a univariate PCS via $$\mathcal{U}_n(f)$$.&#x20;

Next, we analyze how the verifier can efficiently verify the claim. The prover will commit to $$\mathcal{U}_n(f)$$ and $$\mathcal{U}_n(q_i)$$, while $$\mathcal{U}_n(1) = \Phi_n(X)$$ can be easily computed. The key question now is: how can we efficiently compute $$\mathcal{U}_n(X_iq_i)$$?

By [**Lemma 2.5.2**](https://eprint.iacr.org/2023/917.pdf#page=11\&zoom=100,180,669), let $$\hat{f} \in \mathbb{F}^{\le2^n -1}$$ and define $$f := \mathcal{U}_n^{-1}\left(\hat{f}\right)$$. For $$0 \le i < n$$, if $$f \in \mathbb{F}[X_0, \dots, X_{i-1}]^{\le 1}$$, then $$\hat{f}(X) = \Phi_{n-i}\left(X^{2^i} \right) \hat{f}^{<2^i}$$, where $$\hat{f}^{<2^i} =\mathcal{U}_i(f)$$.

For example, if $$i = 2, n = 3$$ and $$f = 2X_0 + X_1 + 0X_2$$​, then $$\mathcal{U}_3(f)$$ can be computed as:

$$
\begin{align*}
\mathcal{U}_3(f) &= \textcolor{red}{f(0, 0, 0) + f(1, 0, 0) X + f(0,1,0)X^2 + f(1,1,0) X^3} + f(0, 0, 1)X^4 + f(1,0,1)X^5 + f(0,1,1)X^6  + f(1,1,1)X^7 \\
&= \textcolor{red}{0 + 2X + 1X^2 + 3X^3} + 0X^4 + 2X^5 + 1X^6 + 3X^7 \\ &= \underbrace{(X^4 + 1)}_{\Phi_1(X^4)}\underbrace{\textcolor{red}{(0 + 2X + 1X^2 + 3X^3)}}_{ \mathcal{U}_2(f)^{<4}} 
\end{align*}
$$

By [**Lemma 2.5.3**](https://eprint.iacr.org/2023/917.pdf#page=12\&zoom=100,180,565), if $$f \in \mathbb{F}[X_0, \dots, X_{i-1} ]^{\le1}$$, we can compute:

$$
\mathcal{U}_n(X_if) = \frac{X^{2^i}}{X^{2^i} + 1}\mathcal{U}_n(f)
$$

For example, if $$i = 2, n = 3$$, and $$f = 2X_0 + X_1$$​, then multiplying by $$X_2$$​ gives $$f' = X_2f = 2X_0X_2 + X_1X_2$$.

Thus, when $$X_2 = 0$$, both $$f$$ and $$f'$$ evaluate to 0. When $$X_2 = 1$$, $$f'$$ takes the same value as $$f$$.

$$
\begin{align*}
\mathcal{U}_3(f') &= f'(0, 0, 0) + f'(1, 0, 0) X + f'(0,1,0)X^2 + f'(1,1,0) X^3 + f'(0, 0, 1)X^4 + f'(1,0,1)X^5 + f'(0,1,1)X^6  + f'(1,1,1)X^7 \\
&= f(0, 0, 1)X^4 + f(1,0,1)X^5 + f(0,1,1)X^6  + f(1,1,1)X^7 \\
&= \frac{X^4}{X^4 + 1}\underbrace{\left({f(0, 0, 0) + f(1, 0, 0) X + f(0,1,0)X^2 + f(1,1,0) X^3} + f(0, 0, 1)X^4 + f(1,0,1)X^5 + f(0,1,1)X^6  + f(1,1,1)X^7\right)}_{\mathcal{U}_3(f)}
\end{align*}
$$

Applying [**Lemma 2.5.3**](https://eprint.iacr.org/2023/917.pdf#page=12\&zoom=100,180,565) to equation $$(2)$$, we transform it into:

$$
\begin{align*}
\mathcal{U}_n(f) - v \cdot \mathcal{U}_n(1) = \sum_{i=0}^{n-1}\left(\frac{X^{2^i}}{X^{2^i} + 1}- z_i\right) \mathcal{U}_n(q_i) && (3)
\end{align*}
$$

At this stage, the verifier must check whether $$q_i \in \mathbb{F}[X_0, \dots, X_{i-1} ]^{\le1}$$. We can verify this using $$\mathcal{U}_n(q_i) \in \mathbb{F}[X]^{<2^i}$$ through a **degree check protocol**. by substituting **Lemma 2.5.2** into equation $$(3)$$, which results in:

$$
\begin{align*}
\mathcal{U}_n(f) - v \cdot \mathcal{U}_n(1) = \sum_{i=0}^{n-1}\left(\frac{X^{2^i}}{X^{2^i} + 1}- z_i\right) \Phi_{n-i}\left(X^{2^i}\right)\mathcal{U}_n(q_i)^{<2^i} && (4)
\end{align*}
$$

Since $$\Phi_{n-i}\left(X^{2^i}\right)$$ is defined as:

$$
\begin{align*}
\Phi_{n-i}\left(X^{2^i}\right) &= 1 + X^{2^i} + \left(X^{2^i}\right)^2 + \dots + \left(X^{2^i}\right)^{2^{n-i} - 1} \\
&= \frac{X^{2^n} - 1}{X^{2^i} - 1}
\end{align*}
$$

we can simplify:

$$
\frac{X^{2^i}}{X^{2^i} + 1}\Phi_{n-i}\left(X^{2^i}\right) = \frac{X^{2^i}}{X^{2^i} + 1}\frac{X^{2^n} - 1}{X^{2^i} - 1} = X^{2^i}\frac{X^{2^n} - 1}{X^{2^{i + 1}} - 1}  = X^{2^i}\Phi_{n-i-1}\left(X^{2^{i+1}}\right)
$$

Applying this to equation $$(4)$$, we obtain:

$$
\begin{align*}
  \mathcal{U}_n(f) - v \cdot \mathcal{U}_n(1) = \sum_{i=0}^{n-1}\left(X^{2^i}\Phi_{n-i-1}\left(X^{2^{i+1}}\right)- z_i \cdot \Phi_{n-i}\left(X^{2^i}\right)\right) \mathcal{U}_n(q_i)^{<2^i} && (5)
  \end{align*}
$$

Now we can define $$\zeta_x(X)$$ using equation (5), which is partially evaluated at $$x$$ on $$\Phi_n(X)$$ as:

$$
\zeta_x(X) := \mathcal{U}_n(f) - v \cdot \Phi_n(x) - \sum_{i=0}^{n-1}\left(x^{2^i}\Phi_{n-i-1}\left(x^{2^{k+1}}\right)- z_i \cdot \Phi_{n-i} \left( x^{2^i} \right) \right)\mathcal{U}_n(q_i)^{<2^i}(X) \\
$$

Then the prover needs to commit to this polynomial $$(C_{\zeta_x}, r) = \mathsf{HidingKZG.Commit}(\zeta_x)$$. If the prover sends $$(W, \delta) = \mathsf{HidingKZG.ProveEval}(C_{\zeta_x}, r, x, 0)$$ to the verifier, then the verifier can verify using 3 pairing operations with $$\mathsf{HidingKZG.VerifyEval}(C_{\zeta_x},  W, \delta, x, 0)$$.

Note that since only $$\zeta_x(x) = 0$$ is opened for $$\zeta_x(X)$$, it is now possible to prove **circuit satisfiability** **without revealing any additional information.**

### Degree Check Protocol

#### Single KZG Degree Check

Let's construct a protocol based on $$\mathsf{HidingKZG}$$ that verifies whether a given polynomial $$f \in \mathbb{F}[X]$$ satisfies $$\deg(f) \le d < N_{\mathsf{max}}$$.

$$\mathsf{HidingKZG.ProveOpenWithDegreeCheck}(C, r, d) \rightarrow (W, \delta)$$:

1. The prover samples $$\alpha \leftarrow \mathbb{F}$$.
2. The prover provides the verifier with $$W, \delta$$:

$$
W = \left[f(s)s^{N_{\mathsf{max}} - 1 -d} \right]_1 + \alpha\cdot [\xi]_1, \quad \delta = r \cdot \left[s^{N_{\mathsf{max}}-1-d}\right]_1 - [\alpha]_1
$$

$$\mathsf{HidingKZG.VerifyOpenWithDegreeCheck}(C, W, \delta, d) \rightarrow b \in \{0, 1\}$$:

The verifier checks the following equation:

$$
e(C, \left[ s^{N_{\mathsf{max}}-1-d} \right]_2) \stackrel{?}= e(W, [1]_2) \cdot e(\delta, [\xi]_2)
$$

This is satisfied iff $$\mathsf{HidingKZG.Open}(C, f, r) = 1 \land f \in \mathbb{F}[X]^{\le d}$$.

**Soundness Proof:**

Suppose $$\deg(f) > d$$. Then, the degree of $$f(X) \cdot X^{N_{\mathsf{max}}-1-d}$$ is at least $$N_{\mathsf{max}}$$, making it impossible to commit to $$W$$.

**Completeness Proof:**

$$
\begin{align*}
C \cdot s^{N_{\mathsf{max} - 1 - d}} =(f(s) + r\xi) \cdot s^{N_{\mathsf{max} - 1 - d}} &= f(s)s^{N_{\mathsf{max}}-1-d} +\alpha\xi + (r\cdot s^{N_{\mathsf{max}} - 1 - d} - \alpha)\xi   \\
&= f(s)s^{N_{\mathsf{max}}-1-d} + r \xi \cdot s^{N_{\mathsf{max}} - 1 - d}
\end{align*}
$$

$$\mathsf{HidingKZG.ProveEvalWithDegreeCheck}(C, r, d, z, v) \rightarrow (W, \delta)$$:

1. The prover samples $$\alpha \leftarrow \mathbb{F}$$.
2. The prover provides the verifier with $$W, \delta$$:

$$
W = \left[\frac{f(s) - v}{s - z}s^{N_{\mathsf{max}}-d} \right]_1 + \alpha\cdot [\xi]_1, \quad \delta = r \cdot \left[s^{N_{\mathsf{max}}-d} \right]_1 - [\alpha(s - z)]_1
$$

$$\mathsf{HidingKZG.VerifyEvalWithDegreeCheck}(C, W, \delta, d, z, v) \rightarrow b \in \{0, 1\}$$:

The verifier checks the following equation:

$$
e(C - [v]_1, \left[ s^{N_{\mathsf{max}}-d} \right]_2) \stackrel{?}= e(W, [s]_2 - [z]_2) \cdot e(\delta, [\xi]_2)
$$

This is satisfied iff $$\mathsf{HidingKZG.Open}(C, f, r) = 1 \land f \in \mathbb{F}[X]^{\le d} \land f(z) = v$$.

**Soundness Proof:**

Suppose $$\deg(f) > d$$. Then, the degree of $$\frac{f(X) - v}{X - z}X^{N_{\mathsf{max}}-d}$$ is at least $$N_{\mathsf{max}}$$, making it impossible to commit to $$W$$.

**Completeness Proof:**

$$
\begin{align*}
(C -v ) \cdot s^{N_{\mathsf{max}} - d} =(f(s) + r\xi -v ) \cdot s^{N_{\mathsf{max}} - d} &= (f(s) - v)s^{N_{\mathsf{max}}-d} + \alpha\xi(s - z) + (r\cdot s^{N_{\mathsf{max}} - d} - \alpha(s - z))\xi   \\
&= (f(s) - v)s^{N_{\mathsf{max}}-d} + r\xi \cdot s^{N_{\mathsf{max}}-d}
\end{align*}
$$

#### Batch Degree Check

Now, let’s consider a set of polynomials $$\bm{f} = \{f_0, \dots, f_{k-1}\}$$ and their respective degree bounds $$\bm{d} = \{d_0, \dots, d_{k-1}\}$$. For $$0 \le i < k$$, each $$f_i$$ belongs to $$\mathbb{F}[X]$$, and we aim to construct a batch protocol to verify whether $$\deg(f_i) \le d_i < N_{\mathsf{max}}$$.

First, we choose $$\max d_i \le d^* \le N_{\mathsf{max}} - 2$$. Let $$\bm{C} = \{C_0, \dots, C_{k-1}\}$$ and $$\bm{r} =\{r_0, \dots, r_{k-1}\}$$ be the set of commitments and randomness corresponding to $$\bm{f}$$.

$$\mathsf{HidingKZG.ProveOpenWithBatchDegreeCheck}(\bm{f}, \bm{r}, \bm{d}) \rightarrow (C_f, W, \delta)$$:

1. The verifier samples $$y \leftarrow \mathbb{F}$$ and sends it to the prover.
2. The prover computes $$(C_f, r) = \mathsf{HidingKZG.Commit}(f)$$ and sends it to the verifier:

$$
f := \sum_{i=0}^{k-1}y^iX^{d^*-d_i+1}f_i
$$

3. The verifier samples $$x \leftarrow \mathbb{F}^*$$ (where $$x \ne 0$$) and sends it to the prover.
4. The prover samples $$\alpha \leftarrow \mathbb{F}$$.
5. The prover sends $$W, \delta$$ to the verifier:

$$
W = \left[\frac{\zeta_x(s)}{s - x} s^{N_{\mathsf{max}} - d^* - 1}\right]_1 + \alpha \cdot [\xi]_1 \\ \delta = \left(r - \sum_{i=0}^{k-1}y^ix^{d^*-d_i+1}r_i\right)\cdot\left[ s^{N_{\mathsf{max}}-d*-1}\right]_1 - [\alpha(s - x)]_1
$$

Here, $$\zeta_x(X)$$ is defined as:

$$
\zeta_x(X) := f(X) - \sum_{i = 0}^{k-1}y^ix^{d^*-d_i+1}f_i(X)
$$

$$\mathsf{HidingKZG.VerifyOpenWithBatchDegreeCheck}(\bm{C}, C_f, W, \delta, \bm{d}) \rightarrow b \in \{0, 1\}$$

The verifier checks the following equation:

$$
e(C_{\zeta_x},\left[ s^{N_{\mathsf{max}} - d^* - 1} \right]_2) \stackrel{?}= e(W, [s]_2 - [x]_2) \cdot e(\delta, [\xi]_2)
$$

The commitment $$C_{\zeta_x}$$ is defined as:

$$
C_{\zeta_X} := C_f - \sum_{i=0}^{k-1}y^ix^{d^*-d_i+1}C_i
$$

This is satisfied iff $$\mathsf{HidingKZG.Open}(C_i, f_i, r_i) = 1 \land f_i \in  \mathbb{F}[X]^{\le d_i}$$ for $$0 \le i < k$$.

**Soundness Proof:**

Suppose that for some $$0 \le i < k$$, we have $$\deg(f_i) > d_i$$. Then the degree of $$f$$ becomes:

$$
\deg(\zeta_x(X)) = \deg (f)  \ge d^* + 2
$$

which leads to:

$$
\deg\left(\frac{\zeta_x(X)}{X - x}X^{N_{\mathsf{max}} - d^* -1}\right) \ge N_{\mathsf{max}}
$$

making it impossible to commit to $$W$$.

**Completeness Proof:**

$$
\begin{align*}
\left(\zeta_x(s) + \left(r - \sum_{i=0}^{k-1}y_ix^{d^*-d_i+1}r_i\right)\xi\right)\cdot s^{N_{\mathsf{max}} - d^* - 1} &= \zeta_x(s) \cdot s^{N_{\mathsf{max}} - d^* - 1} + \alpha\xi(s - x) + \left(\left(r - \sum_{i=0}^{k-1}y_ix^{d^*-d_i+1}r_i\right) \cdot s^{N_{\mathsf{max}} - d^* - 1} - \alpha(s - x)\right)\xi \\
&= \zeta_x(s)\cdot s^{N_{\mathsf{max}} - d^* - 1} + \left(r - \sum_{i=0}^{k-1}y_ix^{d^*-d_i+1}r_i\right)\xi \cdot s^{N_{\mathsf{max}} - d^* - 1}
\end{align*}
$$

In the above protocol, we designed the batch degree check using the $$\mathsf{HidingKZG}$$ protocol. However, as discussed in [Section 5.4](https://eprint.iacr.org/2023/917.pdf#page=28\&zoom=100,180,362) of the paper, this method is applicable in a more general setting, provided that:

* The commitment scheme satisfies additive homomorphism.
* The evaluation protocol is knowledge-sound.

These conditions allow the batch degree check to be used in other polynomial commitment schemes as well.

#### Batch Evaluation with Degree Check

Now, let’s consider a set of polynomials $$\bm{f} = \{f_0, \dots, f_{k-1}\}$$ and their respective evaluation values $$\bm{v} = \{v_0, \dots, v_{k-1}\}$$. For $$0 \le i < k$$, each $$f_i$$ belongs to $$\mathbb{F}[X]$$, and we aim to construct a batch protocol to verify whether $$\deg(f_i) \le d < N_{\mathsf{max}}$$ and $$f_i(z) = v_i$$.

Let $$\bm{C} = \{C_0, \dots, C_{k-1}\}$$ and $$\bm{r} =\{r_0, \dots, r_{k-1}\}$$ be the set of commitments and randomness corresponding to $$\bm{f}$$.

$$\mathsf{HidingKZG.ProveBatchEvalWithDegreeCheck}(\bm{C}, \bm{r}, d, z, \bm{v}) \rightarrow (W, \delta)$$:

1. The verifier samples $$y \leftarrow \mathbb{F}$$ and sends it to the prover.
2. The prover samples $$\alpha \leftarrow \mathbb{F}$$.
3. The prover provides the verifier with $$W, \delta$$:

$$
W = \left[\frac{\sum_{i = 0}^{k-1}y^i(f_i(s) - v_i)}{s - z} s^{N_{\mathsf{max}} - d}\right]_1 + \alpha \cdot [\xi]_1 \\ 
\delta = \left(\sum_{i=0}^{k-1}y^ir_i\right)\cdot\left[ s^{N_{\mathsf{max}}-d}\right]_1 - [\alpha(s - z)]_1
$$

$$\mathsf{HidingKZG.VerifyBatchEvalWithDegreeCheck}(\bm{C}, W, \delta, d, z, \bm{v}) \rightarrow b \in \{0, 1\}$$:

The verifier checks the following equation:

$$
e(\sum_{i=0}^{k-1}y_iC_i - \left[\sum_{i=0}^{k-1}y^iv_i\right]_1,\left[ s^{N_{\mathsf{max}} - d} \right]_2) \stackrel{?}= e(W, [s]_2 - [x]_2) \cdot e(\delta, [\xi]_2)
$$

This is satisfied iff $$\mathsf{HidingKZG.Open}(C_i, f_i, r_i) = 1 \land f_i \in \mathbb{F}[X]^{\le d} \land f_i(z) = v_i$$ for $$0 \le i < k$$.

**Soundness Proof:**

Suppose that for some $$0 \le i < k$$, we have $$\deg(f_i) > d$$. Then the degree of the expression:

$$
\deg\left(\frac{\sum_{i=0}^{k-1}y^i(f_i(X) - v)}{X - x}X^{N_{\mathsf{max}} - d}\right) \ge N_{\mathsf{max}}
$$

making it impossible to commit to $$W$$.

**Completeness Proof:**

$$
\begin{align*}
\sum_{i=0}^{k-1}y_i\left(f_i(s) + r_i\xi - v_i\right) \cdot s^{N_{\mathsf{max}} - d} &= \sum_{i=0}^{k-1}y_i(f_i(s) - v_i) \cdot s^{N_{\mathsf{max}} - d} + \alpha\xi(s - z) + \left(\sum_{i=0}^{k-1}y^ir_i \cdot s^{N_{\mathsf{max}} - d} - \alpha(s - z)\right)\xi \\
&=\sum_{i=0}^{k-1}y_i(f_i(s) - v_i) \cdot s^{N_{\mathsf{max}} - d} + \sum_{i=0}^{k-1}y^ir_i\xi \cdot s^{N_{\mathsf{max}} - d}
\end{align*}
$$

### Put Together

To summarize, in order to prove $$f(\bm{z}) \stackrel{?}= v$$, we transformed the problem as follows:

$$
\mathcal{U}_n(f) - v \cdot \Phi_n(X) = \sum_{i=0}^{n-1}\left(X^{2^i}\Phi_{n-i-1}\left(X^{2^{i+1}}\right)- z_i \cdot \Phi_{n-i}\left(X^{2^i}\right)\right) \mathcal{U}_n(q_i)^{<2^i}
$$

Now, let's prove the above equation using $$\mathsf{HidingKZG}$$.

1. The prover computes commitments $$(C_i, r_i) := \mathsf{HidingKZG.commit}\left(\mathcal{U}_n(q_i)^{<2^i}\right)$$ for $$0 \le i < n$$, and sends them to the verifier.
2. The verifier randomly samples $$y \leftarrow \mathbb{F}$$.
3. The prover computes $$(C_{\hat{q}}, \hat{r}) := \mathsf{HidingKZG.commit}\left(\hat{q}\right)$$ for $$0 \le i < n$$ and sends the commitments to the verifier, where:

$$
\hat{q}(X) := \sum_{i=0}^{n-1}y^iX^{2^n-d_i-1}\mathcal{U}_n(q_i)^{<2^i}
$$

4. The verifier randomly samples $$x \leftarrow \mathbb{F}^*, z\leftarrow \mathbb{F}$$ (where $$x \neq 0$$).&#x20;
5. The prover samples $$\alpha \leftarrow \mathbb{F}$$.
6. The prover provides the verifier with $$W, \delta$$:

$$
W = \left[\left(\frac{\zeta_x(s) + z \cdot Z_x(s)}{s- x}\right)s^{N_{\mathsf{max}} - (2^n - 1)} \right]_1 + \alpha \cdot [\xi]_1 \\
 \delta = \left[ (r_\zeta + z r_Z) s^{N_{\mathsf{max}} - (2^n - 1)} -\alpha (s - x)\right]_1
$$

Where the values are defined as follows:

$$
\zeta_x(X) := \hat{q}(X) - \sum_{i=0}^{n-1}y^ix^{2^n-d_i-1}\mathcal{U}_n(q_i)^{<2^i}\\
Z_x(X) := \mathcal{U}_n(f) - v \cdot \Phi_n(x) - \sum_{i=0}^{n-1}\left(x^{2^i}\Phi_{n-i-1}\left(x^{2^{k+1}}\right)- z_i \cdot \Phi_{n-i} \left( x^{2^i} \right) \right)\mathcal{U}_n(q_i)^{<2^i} \\
r_\zeta := r - \sum_{i=0}^{n-1}\left(x^{2^i}\Phi_{n-i-1}\left(x^{2^{i+1}}\right)-z_i \cdot \Phi_{n-i}\left(x^{2^i}\right)\right) \cdot r_i \\
r_Z := \hat{r} - \sum_{i=0}^{n-1}y^ix^{2^n-d_i-1}r_i
$$

7. The verifier checks the following equation:

$$
e(C_{\zeta_x} + z \cdot C_{Z_x}, [s^{N_{\mathsf{max}}-(2^n-1)}]_2) \stackrel{?}= e(W, [s - x]_2) \cdot e(\delta, [\xi]_2)
$$

Where the values are defined as follows:

$$
C_{\zeta_x} := C_{\hat{q}} - \sum_{i=0}^{n-1}y^ix^{2^n-d_i-1}C_i \\
C_{Z_x} := [\mathcal{U}_n(f)(s)]_1 - [v \cdot \Phi_n(x)]_1 - \sum_{i=0}^{n-1}\left(x^{2^i}\Phi_{n-i-1}\left(x^{2^{i+1}}\right) - z_i \cdot \Phi_{n-i}\left(x^{2^i})\right)\right)\cdot C_i
$$

**Completeness Proof:**

$$
\begin{align*}
(\zeta_x(s) + r_{\zeta} + z \cdot (Z_x(s) + r_Z))s^{N_{\mathsf{max}} - (2^n - 1)} &= (\zeta_x(s) + z \cdot Z_x(s))s^{N_{\mathsf{max}} - (2^n - 1)} + \alpha\xi(s- x) + ((r_\zeta + z r_Z) s^{N_{\mathsf{max}} - (2^n - 1)}
- \alpha(s - x))\xi \\
&= (\zeta_x(s) + z \cdot Z_x(s))s^{N_{\mathsf{max}} - (2^n - 1)} + (r_\zeta + z r_Z) s^{N_{\mathsf{max}} - (2^n - 1)}

\end{align*}
$$

**Analysis:**

* $$\mathsf{srs}$$(structure reference string) **size**: $$N^{\mathsf{max}}+1\mathbb{G}_1, N^{\mathsf{max}}+1\mathbb{G}_2$$
* **Prover work**: $$O(n)\ \mathsf{MSM}$$
  * To compute each $$C_i$$, it takes $$2^i + 1 \ \mathsf{MSM}$$ operations.
  * To compute $$C_{\hat{q}}$$, it takes at most $$2^{n-1}\ \mathsf{MSM}$$ operations. This is because $$\hat{q}$$ ​ has nonzero coefficients ranging from a maximum degree of $$2^{n-1}=2^n- d_{n-1}-1$$ to $$2^{n}-1 = 2^n - d_0 - 1$$.
  * To compute $$W$$, it takes at most $$2^n\ \mathsf{MSM}$$ operations.
  * To compute $$\delta$$, it takes $$3\ \mathsf{MSM}$$ operations: $$(r_\zeta + z r_Z) \cdot [ s^{N_{\mathsf{max}} - (2^n - 1)}]_1 -\alpha \cdot [s]_1 + \alpha x \cdot [1]_1$$.
* **Proof length**: $$n+3\mathbb{G}_1,3\mathbb{F}$$
  * $$n + 3\mathbb{G}_1$$: $$(C_0, \dots, C_{n-1}, C_{\hat{q}}, W, \delta)$$
  * $$3\mathbb{F}$$: $$(y, x, z)$$
* **Verifier work**: $$2n+6 \mathsf{MSM}_1, 2\mathbb{G}_2, 3\mathsf{P}$$
  * To compute $$[v \cdot \Phi_n(x)]_1$$, it takes $$1\ \mathsf{MSM}$$ operation.
  * To compute $$C_{Zx}$$, it takes $$n + 2\ \mathsf{MSM}$$ operations.
  * To compute $$C_{\zeta_x}$$, it takes $$n + 1\ \mathsf{MSM}$$ operations.
  * To compute $$C_{\zeta_x} + z \cdot C_{Z_x}$$, it takes $$2\ \mathsf{MSM}$$ operations.
  * To compuete $$[s]_2 - [x]_2$$, it takes $$2 \mathbb{G}_2$$ operations.

Here, $$\mathsf{MSM}$$ represents MSM operations in $$\mathbb{G}_1$$, $$\mathbb{F}$$ represents addition or multiplication in $$\mathbb{F}$$, and $$\mathsf{P}$$ denotes a pairing operation.

### Shift Evaluation

In permutation arguments or lookup arguments like Plookup, computing the evaluations of shifted polynomials is necessary. For example, if $$f \in \mathbb{F}[X_0, \dots, X_{n-1}]^{\le 1}$$ has the following evaluations:

$$
\bm{v} = \{ v_0, v_1, \dots, v_{2^n-2}, v_{2^n-1} \}
$$

then the evaluations of $$\mathsf{shift}(f)$$ are:

$$
\bm{v'} = \{ v_1, v_2, \dots, v_{2^n-1}, v_0 \}
$$

Applying the $$\mathcal{U}_n$$ mapping results in:

$$
\mathcal{U}_n(f) = v_0 + v_1X  \dots + v_{2^n-2}X^{2^n-2} + v_{2^n-1}X^{2^n-1} \\
\mathcal{U}_n(\mathsf{shift}(f) ) =v_1 + v_2X +  \dots + v_{2^n-1}X^{2^n -2} + v_0X^{2^n -1}
$$

To prove $$\mathsf{shift}(f)(\bm{z}) = v$$, we construct the following equation:

$$
\mathsf{shift}(f) - v = \sum_{i=0}^{n-1}(X_i - z_i)q_i
$$

and demonstrate the existence of a suitable $$q_i$$ satisfying:

* $$i = 0 \Rightarrow q_0 \in \mathbb{F}$$
* $$0 < i < n \Rightarrow q_i \in \mathbb{F}[X_0, \dots, X_{i-1}]^{\le 1}$$

Applying the transformation from equation $$(1)$$ to $$(5)$$, we obtain:

$$
\begin{align*}
  \mathcal{U}_n(\mathsf{shift}(f)) - v \cdot \Phi_n(X) = \sum_{i=0}^{n-1}\left(X^{2^i}\Phi_{n-i-1}\left(X^{2^{i+1}}\right)- z_i \cdot \Phi_{n-i}\left(X^{2^i}\right)\right) \mathcal{U}_n(q_i)^{<2^i} && (6)
  \end{align*}
$$

Multiplying $$\mathcal{U}_n(\mathsf{shift}(f))$$ by $$X$$, we get:

$$
\begin{align*}
X\mathcal{U}_n(\mathsf{shift}(f)) &= v_1X + v_2X^2 + \dots +v_{2^n-1}X^{2^n-1} + v_0X^{2^n} \\
&= v_0 + v_1X + v_2X^2 + \dots +v_{2^n-1}X^{2^n-1} + v_0X^{2^n} - v_0 \\
&= \mathcal{U}_n(f) + v_0(X^{2^n} - 1)
\end{align*}
$$

Thus, multiplying both sides of equation $$(6)$$ by $$X$$, we obtain:

$$
\begin{align*}
  \mathcal{U}_n(f) + v_0(X^{2^n} - 1)
 - v \cdot X \Phi_n(X) = X \cdot \sum_{i=0}^{n-1}\left(X^{2^i}\Phi_{n-i-1}\left(X^{2^{i+1}}\right)- z_i \cdot \Phi_{n-i}\left(X^{2^i}\right)\right) \mathcal{U}_n(q_i)^{<2^i} && (7)
  \end{align*}
$$

Since equation $$(7)$$ holds, we can prove that $$\mathsf{shift}(f)$$ is indeed a shifted version of $$f$$ by committing only once to $$\mathcal{U}_n(f)$$ and then using it to verify the evaluation claim of $$\mathsf{shift}(f)$$.

If $$v_0 = 0$$, the protocol simplifies even further! While the referenced paper discusses high-degree shifts and batched verification methods, these are omitted here for brevity. Interested readers are encouraged to refer to [**Sections 7**](https://eprint.iacr.org/2023/917.pdf#page=39\&zoom=100,180,786) and [**8**](https://eprint.iacr.org/2023/917.pdf#page=41\&zoom=100,180,693) for more details!

## Conclusion

<figure><img src="../../.gitbook/assets/Screenshot 2025-02-28 at 9.24.46 AM.png" alt=""><figcaption></figcaption></figure>

The multilinear polynomial identity originally required $$n$$ pairing operations. However, thanks to  $$\mathcal{U}_n$$ mapping, the verifier can now perform the expensive pairing computation in just three operations.

Meanwhile, to prove that $$q_i$$ is bounded by a certain degree, the degree check protocol must be executed, which requires an $$\mathsf{srs}$$ proportional to $$N_{\mathsf{max}}$$ in $$\mathbb{G}_1$$ and $$\mathbb{G}_2$$. Fortunately, the verifier does not need to perform the costly $$\mathbb{G}_2$$ operations.

Additionally, an important point is that for hiding, only $$\log N + 5$$ elements in $$\mathbb{G}_1$$ are required.

## References

* [https://eprint.iacr.org/2023/917](https://eprint.iacr.org/2023/917)
* [https://www.youtube.com/watch?v=W950pkm3IL0](https://www.youtube.com/watch?v=W950pkm3IL0)

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze
