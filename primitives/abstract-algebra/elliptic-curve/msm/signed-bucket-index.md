# Signed Bucket Index

> AKA "[NAF (non-adjacent form)](../../../naf-non-adjacent-form.md) method"

> Note: this method is best used when negation for a group is cheap such as for an elliptic curve.

## Motivation

When following a bucket-based method like [Pippenger's algorithm](pippengers-algorithm.md), the amount of memory required for the buckets is demanding. How can we reduce the number of buckets per window from $$2^s-1$$ to $$2^{s-1}$$, where $$s$$ is the number of bits per window, and reduce overhead?

## Explanation

### Naive [Pippenger's](pippengers-algorithm.md)

Let's take a look at an example of creating buckets with [Pippenger's algorithm](pippengers-algorithm.md). Let's say the number of windows $$\lceil\frac{\lambda}{s}\rceil=2$$ and the bit size per window $$s = 3$$. We'll look at the following example where $$k$$ denotes scalars and $$P$$ refers to points on an elliptic curve for the [MSM](./) problem.

$$
\begin{align*}
S &= k_1P_1 + k_2P_2 + k_3P_3+ k_4P_4 + k_5P_5 + k_6P_6 + k_7P_7 \\
&= 57P_1 + 50P_2 + 43P_3 + 36P_4 + 29P_5 + 22P_6 + 15P_7  \\
\end{align*}
$$

$$
\begin{align*}
\begin{array}{ll}
\begin{aligned}
k_1 = 57 &= 
\overbrace{\underbrace{1}_{2^0}\underbrace{0}_{2^1}\underbrace{0}_{2^2}}^{\mathclap{\text{window 1}}}
\overbrace{\underbrace{1}_{2^3}\underbrace{1}_{2^4}\underbrace{1}_{2^5}}^{\mathclap{\text{window 2}}}
\\[1ex]
&= k_{1,1} + k_{1,2}2^s \\
&= 1 + 7(2^3)
\end{aligned} \quad\quad
&
\begin{aligned}
k_2 = 50 &= 
\overbrace{\underbrace{0}_{2^0}\underbrace{1}_{2^1}\underbrace{0}_{2^2}}^{\mathclap{\text{window 1}}}
\overbrace{\underbrace{0}_{2^3}\underbrace{1}_{2^4}\underbrace{1}_{2^5}}^{\mathclap{\text{window 2}}}
\\[1ex]
&= k_{2,1} + k_{2,1}2^s \\
&= 2 + 6(2^3)
\end{aligned}

\\[8ex]
\begin{aligned}
k_3 = 43 &= 
\overbrace{\underbrace{1}_{2^0}\underbrace{1}_{2^1}\underbrace{0}_{2^2}}^{\mathclap{\text{window 1}}}
\overbrace{\underbrace{1}_{2^3}\underbrace{0}_{2^4}\underbrace{1}_{2^5}}^{\mathclap{\text{window 2}}}
\\[1ex]
&= k_{3,1} + k_{3,2}2^s \\
&= 3 + 5(2^3) 
\end{aligned}
&
\begin{aligned}
k_4 = 36 &= 
\overbrace{\underbrace{0}_{2^0}\underbrace{0}_{2^1}\underbrace{1}_{2^2}}^{\mathclap{\text{window 1}}}
\overbrace{\underbrace{0}_{2^3}\underbrace{0}_{2^4}\underbrace{1}_{2^5}}^{\mathclap{\text{window 2}}}
\\[1ex]
&= k_{4,1} + k_{4,2}2^s \\
&= 4 + 4(2^3)
\end{aligned}

\\[8ex]
\begin{aligned}
k_5 = 29 &= 
\overbrace{\underbrace{1}_{2^0}\underbrace{0}_{2^1}\underbrace{1}_{2^2}}^{\mathclap{\text{window 1}}}
\overbrace{\underbrace{1}_{2^3}\underbrace{1}_{2^4}\underbrace{0}_{2^5}}^{\mathclap{\text{window 2}}}
\\[1ex]
&= k_{5,1} + k_{5,2}2^s \\
&= 5 + 3(2^3)
\end{aligned}
&
\begin{aligned}
k_6 = 22 &= 
\overbrace{\underbrace{0}_{2^0}\underbrace{1}_{2^1}\underbrace{1}_{2^2}}^{\mathclap{\text{window 1}}}
\overbrace{\underbrace{0}_{2^3}\underbrace{1}_{2^4}\underbrace{0}_{2^5}}^{\mathclap{\text{window 2}}}
\\[1ex]
&= k_{6,1} + k_{6,2}2^s \\
&= 6 + 2(2^3)
\end{aligned}

\\[8ex]
\begin{aligned}
k_7 = 15 &= 
\overbrace{\underbrace{1}_{2^0}\underbrace{1}_{2^1}\underbrace{1}_{2^2}}^{\mathclap{\text{window 1}}}
\overbrace{\underbrace{1}_{2^3}\underbrace{0}_{2^4}\underbrace{0}_{2^5}}^{\mathclap{\text{window 2}}}
\\[1ex]
&= k_{7,1} + k_{7,2}2^s \\
&= 7 + 1(2^3)
\end{aligned}
\end{array}
\end{align*}
$$

With buckets $$B_{\text{bucket_multiplier, window_index}}$$, we create our windows $$W$$:

$$
\begin{align*}
W_1 &= P_1 + 2P_2 + 3P_3 + 4P_4 + 5P_5 +6P_6 + 7P_7 = B_{1,1} + 2B_{2,1} + 3B_{3,1} + 4B_{4,1} + 5B_{5,1} + 6B_{6,1} + 7B_{7,1}\\
W_2 &= P_7 + 2P_6 + 3P_5+4P_4 + 5P_3 + 6P_2 + 7P_1 = B_{1,2} + 2B_{2,2} + 3B_{3,2} + 4B_{4,2} + 5B_{5,2} + 6B_{6,2} + 7B_{7,2}\\
\end{align*}
$$

### Introducing NAF for buckets

Note that any computation $$kP$$ can be reduced to $$(k-2^{s})P + 2^{s}P$$ where $$s$$ is the bit width of $$k$$.

This means we can define the following NAF bucket method $$k' = \mathsf{NAF}(k, P)$$ for window $$W_j$$ as such:

1. If $$0 < k < 2^{s-1}$$:
   1. $$k' = k$$
   2. Accumulate $$P$$ into the $$k$$-th bucket $$B_{k,j}$$
2. If $$2^{s-1} \le k < 2^s$$:
   1. $$k' = k - 2^s$$, then $$kP = -k'(-P)  + 2^sP$$
   2. Accumulate $$-P$$ into the $$-k'$$-th bucket $$B_{-k', j}$$
   3. Add $$P$$ into the next window

Step 2c computes the $$2^{s}P$$ part of the $$kP$$ reduction and is also known as the "carry."

Let's change our naive Pippenger's to follow this method and generate our buckets and windows:\


<figure><img src="../../../../.gitbook/assets/image.avif" alt=""><figcaption><p>Stage 1: Separating previous window <span class="math">W_1</span> into new buckets</p></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/Screenshot 2025-04-18 at 10.44.12 AM.png" alt=""><figcaption><p>Stage 2: Separating previous window <span class="math">W_2</span> into new buckets</p></figcaption></figure>

> Notice that this method requires the presence of an extra carry bucket $$B_{1,3}$$. This extra carry bucket should not exist. We'll take a look later at how to prevent this after this Naive Pippenger's to NAF method is fully shown.

{% hint style="warning" %}
[Ashley Jeong](https://app.gitbook.com/u/Z4DJjtMFfLVXwMaSBRDBJRWVDV12 "mention"): I'm unsure why they don't just add $$k_{i,j} = 2^{s-1}$$ into the $$B_{k,j}$$ bucket. Perhaps this is a source of optimization when implementing MSM.
{% endhint %}

#### Bucket Accumulation

| Buckets     | Computation               |
| ----------- | ------------------------- |
| $$B_{1,1}$$ | $$P_1-P_7$$               |
| $$B_{2,1}$$ | $$P_2-P_6$$               |
| $$B_{3,1}$$ | $$P_3-P_5$$               |
| $$B_{4,1}$$ | $$-P_4$$                  |
| $$B_{1,2}$$ | $$-P_1$$                  |
| $$B_{2,2}$$ | $$P_7-P_2$$               |
| $$B_{3,2}$$ | $$P_6-P_4-P_3$$           |
| $$B_{4,2}$$ | $$-P_5$$                  |
| $$B_{1,3}$$ | $$P_5+P_4 + P_3+P_2+P_1$$ |

#### Bucket Reduction

$$
\begin{align*}
W_1 &= B_{1,1} + 2B_{2,1} + 3B_{3,1}  + 4B_{4,1} \\
W_2 &= B_{1,2} + 2B_{2,2} + 3B_{3,2} + 4B_{4,2} \\
&\sim* B_{1,3}*
\end{align*}
$$

#### Window Reduction

$$
S = W_1 + 2^2W_2+ 2^4(B_{1,3})
$$

### Preventing the formation of an extra bucket

#### 1. Ensuring last window bit size is within limit

The formation of an extra carry bucket depends on the true bit size $$s_\mathsf{last}$$ of the final window in relation to $$\lambda$$, the total bits of $$k$$, and $$s$$, the bits per windows.

#### **Safe Case:** $$s_\mathsf{last} < s - 1$$&#x20;

* Example: $$\lambda = 8$$, $$s = 5$$ ⇒ $$k = k_1 + 2^5 k_2$$
* Then $$s_\mathsf{last} = 3$$, and $$0 \le k_2 < 2^3$$
* Since $$k_2 + 1 < 2^4$$, **carry cannot occur** ⇒ It’s safe to assume the final carry is 0.

#### **Carry Case:** $$s_\mathsf{last} \ge s - 1$$

* Example: $$\lambda = 8$$, $$s = 4$$ ⇒ $$k = k_1 + 2^4 k_2$$
* Here $$s_\mathsf{last} = 4$$, and $$0 \le k_2 < 2^4$$
* Since it is possible that $$k_2 + 1 \ge 2^3$$, **carry can occur**

Thus, if the restriction $$s_\mathsf{last} < s - 1$$ is held, we can ensure there will be no new carry bucket created.

#### 2. Naive Fix: Use Larger Bucket Only for Final Window

{% @github-files/github-code-block url="https://github.com/kroma-network/tachyon/blob/e7b13063c6f110b9851655cca740ee62ffb41137/tachyon/math/elliptic_curves/msm/algorithms/pippenger/pippenger.h#L119-L123" %}

* Use a larger bit-width bucket for the final window to absorb potential carry.
* Cons: This is more complex and can be inefficient due to extra bucket space.

#### 3. Use a Larger $$\lambda$$ Than the Actual Modulus Bit Size

Let’s say we are working with BN254, where $$\lambda = 256$$ and $$s = 16$$. The actual scalar field is 254 bits.

We express $$k$$ as:

$$
k = k_1 + 2^{16}k_2 + \dots + 2^{16\times15}k_{16}
$$

Let $$M$$ be the modulus of BN254. Since the maximum value of $$k$$ is $$M - 1$$, we have:

$$
1 \le k_{16} + 1 = \lfloor \frac{M-1}{2^{16\times15}} \rfloor + 1 <2^{14}
$$

This ensures that **no overflow occurs** in the final window.

```python
M = 21888242871839275222246405745257275088548364400416034343698204186575808495617
print((M - 1) // (2**(16 * 15)) + 1 < 2 ** 14) # True
```

#### 4. Use [Yrrid](https://www.yrrid.com/)’s Trick

Yrrid, the Zprize winner in 2022 for the category "_Prize 1a: Accelerating MSM Operations on GPU_" explains one more optimization they used for their solution to **guarantee that no carry occurs in the final window**. When the **MSB of** $$k$$ **is set to 1**, they apply the following transformation to **reset the MSB to 0**, thereby preventing any carry:

$$
kP = (-k)(-P) = (M - k)(-P)
$$

Here, $$M$$ is the modulus of the scalar field (i.e., curve order), and $$k \in [0, M)$$.

Let $$m_\mathsf{last}$$ denote the $$s$$-bit chunk in the final window. Yrrid's trick is said to ensure that we don't overflow to a $$m_\mathsf{last} + 1$$ chunk.

For example, in $$\mathbb{F}_{11}$$ with $$\lambda = 4$$, the values where the MSB of $$k$$ is set to 1 are 8, 9, and 10. This means that -8, -9, and -10 will be transformed into 3, 2, and 1, respectively. With this transformation, the new maximum value of $$k$$ becomes $$2^3 - 1 = 7$$. If $$s = 2$$, then:

$$
k = k_1 + 2^2k_2
$$

In this case with the maximum value of $$k$$ as 7, $$k_2$$ will be at most 1.

## Code Example

Here's a code snippet from [Tachyon](https://github.com/kroma-network/tachyon) implementing NAF for bucket MSM:

<pre class="language-cpp"><code class="lang-cpp"><strong>void FillDigits(uint32_t kLambda, uint32_t s, const ScalarField&#x26; f, std::vector&#x3C;int64_t>* digits) {
</strong>  // The size of `digits` must be equal to ceil(lambda / s)
  // (lambda: total number of bits, s: window size in bits)
  CHECK_EQ(digits->size(), static_cast&#x3C;size_t>(std::ceil(kLambda / s)));
  
  // `radix` is 2^s — the maximum range per window
  uint64_t radix = uint64_t{1} &#x3C;&#x3C; s;

  // `carry` to be propagated to the next window
  uint64_t carry = 0;
  size_t bit_offset = 0;
  for (size_t i = 0; i &#x3C; digits->size(); ++i) {
      // mᵢ: s-bit value extracted from the current window, including previous carry
    uint64_t m_i = f.ExtractBits(bit_offset, s) + carry;
  
    // Compute carry: if mᵢ is greater than or equal to radix / 2, carry = 1; 
    //.               otherwise carry = 0
    // This is based on the NAF rounding threshold (radix / 2)
    carry = (m_i + (radix >> 1)) >> s;
    
    // Convert mᵢ to its NAF form: mᵢ - carry * 2^s
    (*digits)[i] = static_cast&#x3C;int64_t>(m_i) - static_cast&#x3C;int64_t>(carry &#x3C;&#x3C; s);
    bit_offset += s;
  }
  
  // Apply the final carry to the last digit
  // This adjustment preserves the total scalar value
  digits->back() += (carry &#x3C;&#x3C; s);
}

void AddBasesToBuckets(uint32_t s, const std::vector&#x3C;Point> bases, 
    const std::vector&#x3C;std::vector&#x3C;int64_t>>&#x26; digits,
    size_t window_index, std::vector&#x3C;Bucket>&#x26; buckets) {
  for (size_t i = 0; i &#x3C; scalar_digits.size(); ++i) {
    int64_t scalar = scalar_digits[i][window_index];
    if (0 &#x3C; scalar) {
      // If the scalar is positive:
      // Place the corresponding base into the (scalar - 1)-th bucket
      //
      // Why `scalar - 1`?
      // Because scalars range from 1 to 2^{s-1} - 1 in NAF,
      // but bucket indices are 0-based (0 to 2^{s-1} - 2)
      // so we subtract 1 to get the correct index.
      buckets[static_cast&#x3C;uint64_t>(scalar - 1)] += bases[i];
    } else if (0 > scalar) {
      // If the scalar is negative:
      // Place the *negated* base into the (-scalar - 1)-th bucket
      //
      // Why `-scalar - 1`?
      // Because negative scalars range from -1 to -2^{s-1},
      // and we want to map -1 → index 0, -2 → index 1, ..., just like for positives.
      buckets[static_cast&#x3C;uint64_t>(-scalar - 1)] -= bases[i];
    }
  }
}
</code></pre>

## References

* [https://hackmd.io/jNXoVmBaSSmE1Z9zgvRW\_w](https://hackmd.io/jNXoVmBaSSmE1Z9zgvRW_w)

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") & [Ashley Jeong](https://app.gitbook.com/u/PNJ4Qqiz7kSxSs58UIaauk95AO43 "mention") of Fractalyze
