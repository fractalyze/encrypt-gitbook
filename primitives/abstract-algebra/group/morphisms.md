# -Morphisms

## **Homomorphism**: A **structure preserving map** between 2 objects.

* e.g., $$f: \mathbb{Z} \rightarrow \mathbb{Z}_4, f(x) = x \mod 4$$
* homomorphism:\
  $$\begin{align*} f(m + n) &= (m + n) \mod 4\\ &=(m \mod 4) + (n \mod 4) \\ &= f(m) + f(n) \end{align*}$$

## **Endomorphism**: A **homomorphism** where **domain and codomain are the same object**.

* e.g., $$f: \mathbb{Z} \rightarrow \mathbb{Z}, f(x) = 2\cdot x$$
* homomorphism:\
  $$\begin{align*} f(m + n) &= 2(m + n) \\ &= 2m + 2n \\ &= f(m) + f(n) \end{align*}$$

## **Isomorphism**: A **bijective homomorphism**. (i.e., it has an inverse that is an homomorphism).

* e.g., $$f: \mathbb{Z}_4 \rightarrow \mathbb{Z}_4, f(x) = 3 \cdot x \mod 4$$
* homomorphism:\
  $$\begin{align*} f(m + n) &= 3(m + n) \mod 4 \\ &= (3m + 3n) \mod 4 \\ &= (3m \mod 4) + (3n \mod 4) \\ &= f(m) + f(n) \end{align*}$$
* bijection: Since $$\gcd(3, 4) = 1$$, multiplication by $$3 \mod 4$$ is invertible. Its inverse is multiplication by $$3$$ itself, because $$3 \cdot 3 = 9 \equiv 1\mod 4$$.

## **Automorphism**: An **isomorphism** from an object **to itself**. Equivalently, it is a **bijective endomorphism**.

* e.g., the example above is also an example of automorphism. TODO: Find a better example.

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze
