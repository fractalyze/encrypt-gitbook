# NTT Over Extension Field

### The Core Concept

When computing a NTT over an **Extension Field** (e.g., $$\mathbb{F}_{p^2}$$), we can avoid doing operations over the extension field. Instead, we can **"flatten"** the data into the **base field** ($$\mathbb{F}_p$$), run a standard NTT on the base components independently, and then reconstitute the result.

***

### 1. The Transformation

Imagine we are working in a Quadratic Extension Field, where every element $$e$$ is represented as $$a + b \cdot u$$.

* $$a, b$$: Coefficients in the **base field**.
* $$u$$: The extension basis (imaginary unit).

Instead of processing one column of "extension elements," we process two columns of "base elements."

**The Algorithm:**

1. **Split:** Take the input vector of elements $$(a_i + b_i u)$$ and split it into two vectors: $$\bm{A} = [a_0, a_1...]$$ and $$\bm{B} = [b_0, b_1...]$$.
2. **Compute:** Run the standard base field NTT on $$\bm{A}$$ and $$\bm{B}$$ separately.
3. **Merge:** The result for index $$k$$ is simply $$\sf{NTT}(\bm{A})_k + \sf{NTT}(\bm{B})_k \cdot u$$.

***

### 2. Why this works

The validity of this optimization relies on one critical constraint: **The Root of Unity (**$$\omega$$**) must exist in the base field.**

If $$\omega$$ is in the base field, multiplying an extension element by $$\omega$$ scales the components **independently**.

#### The Proof

Let us show the proof in the quadratic extension field case but it scales to arbitrary degree of extension field using the same logic.

{% include "../../.gitbook/includes/definition-5-the-number-th... (1).md" %}

Let our vector of coefficients in extension field be $$\bm{x}$$, where each element is defined by:

$$
x_i = a_i + b_i \cdot u
$$

The definition of the NTT for output $$\hat{\bm{x}}$$ is:

$$
\hat{x}_j=\sum^{n-1}_{i=0}\omega^{ij}x_i
$$

Substitute $$x_i$$ and since $$\omega$$ is in the base field, we can distribute it:

$$
\hat{x}_j=\sum^{n-1}_{i=0}\omega^{ij}x_i=\sum^{n-1}_{i=0}\omega^{ij}(a_i + b_iu)=\sum^{n-1}_{i=0}\omega^{ij}a_i + \sum^{n-1}_{i=0}\omega^{ij}b_iu
$$



And if we think about the NTT over the base field, we have:

$$
\hat{a}_j=\sum^{n-1}_{i=0}\omega^{ij}a_i
$$

$$
\hat{b}_j=\sum^{n-1}_{i=0}\omega^{ij}b_i
$$

Now, we can rewrite the $$\hat{x}_j$$ as:

$$
\hat{x}_j=\hat{a}_j + \hat{b}_ju
$$

### 3. The Constraint (When this fails)

This optimization **only** works if the twiddle factor $$\omega$$ is in the base field.

If $$\omega$$ were in the extension field (e.g., $$\omega = c + du$$), the multiplication would "mix" the components:

$$
\omega \cdot (a + bu) = (c + du)(a + bu) = (ca - db\dots) + (cb + da) \cdot u
$$

Because the new "real" part $$(ca - db)$$ depends on both $$a$$ and $$b$$, you could not process the columns independently.

### **Summary**

* **Standard NTT:** $$O(N \log N)$$ operations using expensive extension field arithmetic.
* **Flattened NTT:** $$d \times O(N \log N)$$ operations using base field arithmetic where $$d$$ is extension degree.
