# Chinese Remainder Theorem (CRT)

Given $$N = \prod_{i=1}^{k}n_i$$ where each moduli(or divisor) $$n_i$$ is a **pairwise** [**coprime**](https://en.wikipedia.org/wiki/Coprime_integers), and the set of $$a_i$$ is given as below:

$$
x \equiv a_1 \pmod {n_1} \\ \vdots \\ x \equiv a_k \pmod {n_k}
$$

Then the system has a unique solution $$x$$ under modulo $$N$$ such that:

$$
x \equiv s_1 \cdot \prod_{i \ne 1} n_i + s_2 \cdot \prod_{i \ne 2} n_i + \cdots + s_k \cdot \prod_{i \ne k} n_i \pmod N
$$

where each $$s_i$$ satisfies:

$$
s_i \cdot \prod_{j \ne i} n_j = a_i \pmod {n_i}
$$

In other words, a set of modulus statements can be reduced to a single statement.

Take the example below:

$$
x = \begin{cases} 2 \pmod 3 \\ 3 \pmod 5 \\ 2 \pmod 7 \end{cases}
$$

$$
35 \cdot s_1 \equiv 2 \cdot s_1 \equiv 2 \pmod 3 \\ 21 \cdot s_2 \equiv s_2 \equiv 3 \pmod 5 \\ 15 \cdot s_3 \equiv s_3 \equiv 2 \pmod 7 \\
$$

Solving these gives $$s_1 = 1, s_2 = 3$$ and $$s_3 = 2$$. Then the solution is

$$
x \equiv 1 \cdot 35 + 3 \cdot 21 + 2 \cdot 15 \equiv 128 \equiv 23 \pmod {105}
$$

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from A41
