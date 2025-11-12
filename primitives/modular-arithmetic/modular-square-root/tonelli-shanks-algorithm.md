# Tonelli-Shanks Algorithm

The **Tonelli–Shanks algorithm** is a classical and efficient method to compute modular square roots over finite fields $$\mathbb{F}_{p^m}$$, especially when $$m$$ is odd and $$p \equiv 1 \pmod 4$$, where simpler methods like Shanks do not apply. The method is based on _Algorithm 5_ from [ePrint 2012/685](https://eprint.iacr.org/2012/685.pdf).

***

## Preconditions

To use Tonelli–Shanks:

* $$m$$ must be **odd**.
* $$p$$ must be an **odd prime**
* $$a$$ is a **quadratic residue**.

We express $$p - 1$$ as:

$$
p - 1 = 2^s \cdot T
$$

where $$T$$ is odd.

This decomposition is known as the **2-adic decomposition** of $$p−1$$, and $$s$$ is the **2-adicity**.

***

## Algorithm Steps (High-level)

Let’s outline the algorithm assuming $$a \in \mathbb{F}_{p^m}^*$$:

1. **Precomputation**
   * Write $$p - 1 = 2^s \cdot T$$
   * Find a known **quadratic non-residue** $$c \in \mathbb{F}_{p^m}$$
   * Compute $$z = c^T$$ — a $$2^s$$-th primitive root of unity
2. **Initial values**
   * $$x = a^{(T+1)/2}$$
   * $$b = a^T$$
   * $$v = s$$
3. **Main loop**
   * While $$b \ne 1$$, do:
     * Find the smallest $$k$$ such that $$b^{2^k} \equiv 1$$
     * Let $$w = z^{2^{v - k - 1}}$$
     *   Update:

         $$
         x \leftarrow x \cdot w,\quad
         b \leftarrow b \cdot w^2,\quad
         z \leftarrow w^2,\quad
         v \leftarrow k
         $$
4. **Output**
   * $$x$$ is the square root of $$a$$

***

### Intuition Behind the Updates in Tonelli–Shanks

The Tonelli–Shanks algorithm iteratively computes the square root of a quadratic residue $$a \in \mathbb{F}_p$$, given the decomposition $$p - 1 = 2^s \cdot T$$, with $$T$$ odd. Its central mechanism relies on maintaining a key invariant and gradually reducing the complexity of the problem step by step.

***

#### Invariant: $$x_i^2 = a \cdot b_i$$

At each iteration $$i$$, the algorithm maintains:

$$
x_i^2 = a \cdot b_i
$$

We define:

* $$x_0 = a^{(T+1)/2}$$
* $$b_0 = a^T$$
*   Then:

    $$
    x_0^2 = a^{T+1} = a \cdot a^T = a \cdot b_0
    $$

**Inductive step**: Suppose at iteration $$i$$, the invariant holds:

$$
x_i^2 = a \cdot b_i
$$

Then the algorithm updates:

$$
\begin{aligned}
w_i &= z^{2^{v_i - k_i - 1}} \\
x_{i+1} &= x_i \cdot w_i \\
b_{i+1} &= b_i \cdot w_i^2
\end{aligned}
$$

Then we have:

$$
x_{i+1}^2 = (x_i w_i)^2 = x_i^2 \cdot w_i^2 = (a \cdot b_i) \cdot w_i^2 = a \cdot (b_i \cdot w_i^2) = a \cdot b_{i+1}
$$

Therefore, the invariant is preserved at every step:

$$
x_i^2 = a \cdot b_i
$$

***

### Why $$b_i \to 1$$: Convergence of the Loop

At each iteration of Tonelli–Shanks, the algorithm updates:

$$
b_{i+1} = b_i \cdot w_i^2
$$

To show that $$b_i \to 1$$, we analyze how the order of $$b_i$$ decreases over time. Suppose that at iteration $$i$$, the smallest integer $$k$$ such that $$b_i^{2^k} = 1$$, and $$b_i^{2^{k-1}} \ne 1$$. Then $$b_i$$ is a **primitive**  $$2^k$$**-th root of unity**.

We now prove that the update:

$$
b_{i+1} = b_i \cdot w_i^2
\quad \text{with} \quad
w_i = z^{2^{v - k - 1}}
$$

forces $$b_{i+1}$$ to have strictly lower order.

Let’s compute:

$$
\begin{aligned}
(b_{i+1})^{2^{k-1}} &= (b_i \cdot w_i^2)^{2^{k-1}} \\
&= b_i^{2^{k-1}} \cdot (w_i^2)^{2^{k-1}} \\
&= (-1) \cdot (-1) = 1
\end{aligned}
$$

***

#### Why does this work?

* Since $$b_i^{2^k} = 1$$ and $$b_i^{2^{k-1}} \ne 1$$, we know by the **Halving Lemma** that $$b_i^{2^{k-1}} = -1$$
*   Now for $$w_i$$, recall that:

    $$
    w_i = z^{2^{v - k - 1}} \Rightarrow w_i^2 = z^{2^{v - k}}
    $$

    Then:

    $$
    (w_i^2)^{2^{k-1}} = z^{2^{v - k} \cdot 2^{k-1}} = z^{2^{v - 1}} = -1
    $$

    because $$z$$ is a **quadratic non-residue**, and by construction(when $$i = 0, v= s$$):

    $$
    z^{2^{s-1}} = \left(c^T\right)^{2^{s-1}} = c^{2^{s-1}\cdot T} = -1
    $$

    This identity continues to hold at every step due to the invariant:

    $$
    z_i^{2^{v - 1}} = -1
    $$

***

#### Result

Since both $$b_i^{2^{k - 1}} = -1$$ and $$(w_i^2)^{2^{k - 1}} = -1$$, their product is 1:

$$
(b_i \cdot w_i^2)^{2^{k - 1}} = 1
$$

Thus, the order of $$b_{i+1}$$ is now at most $$2^{k - 1}$$, which is strictly less than the order of $$b_i$$. As a result, the order of $$b$$ decreases with each iteration, and within at most $$s$$ steps, we reach:

$$
b_{\text{last}} = 1
$$

Once this happens, the invariant $$x^2 = a \cdot b$$ gives $$x^2 = a$$, completing the algorithm.

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze
