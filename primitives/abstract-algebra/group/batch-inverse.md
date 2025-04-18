---
description: >-
  Reduction of the number of field inverse operations when computing batch
  operations.
---

# Batch Inverse

## **What is Batch Inverse?**

**Batch Inverse** is a technique used to efficiently compute the inverses of multiple values $$(v_1, \dots, v_n)$$ at once. Normally, calculating each inverse separately requires $$n \times \mathsf{INV}$$ (inverse operations), but with Batch Inversion, you can compute all the inverses using **just a single inverse operation**.

As inverse operations are significantly more expensive than multiplications, this method is especially useful in performance-sensitive contexts—such as in ZK provers.

***

## **How Does It Work?**

Let’s walk through an example of computing the inverses of four values: $$(a, b, c, d)$$

### 1. Compute Prefix Products

Multiply the values one by one, keeping track of intermediate results:

* $$P_1 = a$$
* $$P_2 = ab$$
* $$P_3 = abc$$
* $$P_4 = abcd$$

So we get:

$$
P = (P_1, P_2, P_3, P_4)
$$

### 2. Compute Inverse of the Final Product

Compute the inverse of the final product:

$$
t = P_4^{-1} = (abcd)^{-1}
$$

This is the **only inverse operation** required.

### 3. Compute Each Inverse in Reverse

Now, traverse the values in reverse to compute individual inverses:

* $$d^{-1} = P_3 \times t$$, then update $$t = t \times d = (abc)^{-1}$$
* $$c^{-1} = P_2 \times t$$, then update $$t = t \times c = (ab)^{-1}$$
* $$b^{-1} = P_1 \times t$$, then update $$t = t \times b = (a)^{-1}$$
* $$a^{-1} = t$$

This phase involves only **multiplication operations**.

***

## **Computational Complexity**

If the vector has length $$n$$, the total computational cost is:

1. **Prefix Product Computation**: $$(n - 1) \times \mathsf{MUL}$$
2. **One Inverse Operation**: $$1 \times \mathsf{INV}$$
3. **Reverse Traversal**: $$2(n - 1) \times \mathsf{MUL}$$

**Total:**

$$
3(n - 1) \times \mathsf{MUL} + 1 \times \mathsf{INV}
$$

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") of [A41](https://www.a41.io/)
