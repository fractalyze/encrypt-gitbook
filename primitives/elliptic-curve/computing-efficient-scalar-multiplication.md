# Computing Efficient Scalar Multiplication

## $$\text{End}: (X, Y) \rightarrow (e_q\cdot X, Y)$$

Given the elliptic curve $$E(\mathbb{F}_q): Y^2 = X^3 +b$$, the 3-rd root of unity $$e_q \in \mathbb{F_q}$$, and $$P = (x, y) \in E$$, the following equation holds.

$$
y^2 = x^3 + b = (e_q \cdot x)^3 + b
$$

Then we can find $$e_r$$ that satisfies the following equation:

$$
e_r \cdot (x, y) = (e_q \cdot x, y) \in E
$$

This is the process to find such $$e_r$$:

1. Find a base field $$e_q \in \mathbb{F}_q$$ such that $${e_q}^3 = 1 \land e_q \ne 1$$.
2. Find a scalar field $$e_r \in \mathbb{F}_r$$ such that $${e_r}^3 = 1 \land e_r \ne 1$$.
3. Test if $$e_r \cdot G \stackrel{?}= (e_q \cdot G_x, G_y)$$ where $$G = (G_x, G_y)$$ is a generator.
4. If the both hands are different, $$e_r \leftarrow e_r^2$$.

We can compute scalar multiplication $$k \cdot G$$ more efficiently. e.g, let’s say $$e_r = 7$$ and $$k = 15$$.

$$
\begin{align*} 15 \cdot G &= 2 \cdot (7 \cdot G) + G \\ &= 2 \cdot (e_k \cdot G_x, G_y) + G \end{align*}
$$

Note that the property of [endomorphism](../group-theory.md#endomorphism-a-homomorphism-where-domain-and-codomain-are-the-same-object) is used for this process.

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from A41
