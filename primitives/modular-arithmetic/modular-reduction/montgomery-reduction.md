# Montgomery Reduction

Montgomery reduction provides an efficient way to perform modular arithmetic, particularly modular multiplication $$a\cdot b \mod n$$. This is crucial when the modulus $$n$$ is large and many operations are needed, as in cryptography. Montgomery reduction excels when $$n$$ is odd and aims to replace slow divisions by n.

## **The Core Idea**

The method shifts calculations into a special "Montgomery domain" defined by an auxiliary modulus $$R$$. Arithmetic happens within this domain, where the reduction step is cleverly transformed into a division by $$R$$. Since $$R$$ is chosen as a power of 2, this division is just a fast bit shift and the remainder is equivalent to its last few bits. Results are converted back to the standard representation when the sequence of operations is complete.

## **The Math Explained**

### The Montgomery Representation

We choose an auxiliary modulus $$R=b^w$$ (where $$b$$ is the base, usually 2) such that $$R>n$$. Critically, $$n$$ and $$R$$ must be coprime: $$\gcd(n,R)=1$$. In this case, this just implies $$n$$ **must be odd**.&#x20;

**Definition 1.** The Montgomery representation $$\bar{x}$$ of an integer $$x$$ (where $$0\leq x<n$$) is $$\bar{x} = x \cdot R \mod n$$.

Since $$n$$ and $$R$$ are coprime, we can define the following conversion functions:

* $$\mathsf{fromMont}(\bar{x})$$: Converts $$\bar{x}$$ back to standard form: $$x=\bar{x}⋅R^{−1} \mod n$$. This operation is also called _Montgomery Reduction._
* $$\mathsf{toMont}(x)$$: Converts $$x$$ into Montgomery form: $$\bar{x}=x⋅b^w \mod n$$. This calculation itself can be optimized with _Montgomery Reduction._

**Lemma 1.** Remainder of the division by $$R=2^w$$ can be efficiently calculated as $$x \mod R =\mathsf{lowbits}_w(x)$$ where $$\mathsf{lowbits}_w(x)$$ takes the least significant $$w$$ bits of $$x$$.

**Lemma 2.** Quotient $$\lfloor x / R\rfloor$$ can be efficiently computed using bit shift: $$x \gg w$$.

### **Montgomery Multiplication**

Given two numbers in Montgomery form, $$\bar{a}=a\cdot R \mod n$$ and $$\bar{b}=b⋅R \mod n$$. We would like to calculate $$\bar{c}\equiv (a\cdot b)\cdot R \mod n$$. This can be calculated using $$\bar{a}\cdot \bar{b} \cdot R^{-1}$$ since:

$$
\bar{a}\cdot\bar{b}\equiv (a\cdot R)\cdot(b\cdot R)\equiv a\cdot b \cdot R^2\equiv \bar{c}\cdot R\mod n
$$

To do the Montgomery Multiplication efficiently, we need to be able to calculate:

$$
\bar{c}=\mathsf{doMontMul}(\bar{a},\bar{b})=\bar{a}\cdot\bar{b}\cdot R^{-1} \mod n
$$

To get the actual $$c=a\cdot  b \mod n$$, we will have to additionally convert back from montgomery form using $$\mathsf{fromMont}(\bar{c})$$.

### Montgomery Reduction

**Definition 2. Montgomery Reduction** is an efficient reduction method to calculate $$T\cdot R^{-1}\mod n$$ for a given $$T < n\cdot R$$. This operation can also be denoted as:

$$
\mathsf{REDC}(T):=T\cdot R^{-1} \mod n
$$

The key idea to perform Montgomery Reduction efficiently is to find an integer $$m$$ such that $$T + m\cdot n$$ is exactly divisible by $$R$$. Then the desired result can be computed as:

$$
T\cdot R^{-1}\equiv\frac{T+m⋅n}{R}\equiv(T+m\cdot n)\gg w \mod n
$$

> What will happen if we precompute $$R^{-1}$$ to compute $$T\cdot R^{-1}$$?

We need to find $$m$$ such that  $$T+m⋅n≡0 \mod R$$

$$
\begin{align*}
T+m\cdot n&\equiv 0 &\mod R\\
m\cdot n&\equiv -T &\mod R\\
m &\equiv -T\cdot n^{-1} &\mod R\\
m &\equiv T\cdot(-n^{-1}) &\mod R
\end{align*}
$$

To calculate $$m$$ efficiently, we use a precomputed value $$n'=−n^{−1} \mod R$$. This $$n'$$ exists because $$\mathsf{gcd}(n,R)=1$$. Now, we have $$m\coloneqq T\cdot n'\mod R$$ and $$\mathsf{REDC}$$ can be defined as:

$$
\mathsf{REDC}(T)=(T+m⋅n)\gg w  \mod n
$$

And we have:

$$
\begin{align*}
0<T&<n\cdot R &\Rightarrow 0&<(T\gg w)=\frac{T}{R}&<n \\
0<m&< R &\Rightarrow 0&<(m\cdot n \gg w)=n\cdot \frac{m}{R}&<n \\
\\&&0&<((T+m\cdot n)\gg w)&<2n
\end{align*}
$$

This means we just need to subtract $$n$$ once if $$((T+m\cdot n)\gg w)\geq n$$ to get the remainder$$\mod n$$.

#### **Overview of `REDC(T)`:**

Given  $$T < n\cdot R$$, and precomputed $$n' \coloneqq −n^{−1} \mod R$$.

1. Calculate $$m=(T⋅n') \mod R=\mathsf{lowbits}_w(\mathsf{lowbits}_w(T)\cdot n')$$. Essentially, a $$w$$-bit non extended multiplication.
2. Compute $$x=\frac{(T+m\cdot n)}{R}$$. (Involves one extended multiplication $$m\cdot n$$, 3 limb additions, and one right shift by $$w$$).
3. Return $$x−n$$ if $$x\geq n$$, otherwise return $$x$$.

#### Subtraction Variant `REDC'(T)`

This is a clever optimization to avoid a carry check when adding  $$T+m\cdot n$$ together. Remember that the range of allowed  $$T$$ is $$[0,nR)$$ and typically stored in 2 limbs $$(T_{low}, T_{high})$$. This is also the same for $$m\cdot n$$; let's denote it as $$(mn_{low},mn_{high})$$. Then, $$x=\frac{(T+mn)}{R}$$ is calculated as following:

1. Sum the low limbs.  $$sum_{low}=T_{low} + mn_{low}$$.
2. Sum the high limbs. $$sum_{high}=T_{high} + mn_{high}$$.
3. Add 1 to $$sum_{high}$$ if there was a carry in the low sum.
4. Set  $$x=(sum_{low} + (sum_{high} \ll w)) \gg w$$ (which is simply $$x=sum_{high}$$)

The sum of low limbs is actually guaranteed to be 0 since $$T+m\cdot n \equiv 0 \mod R$$. But we still need to calculate the sum of low limbs to check if it has a carry or not. Notice that, either the sum can be $$0$$ or $$2^{w}$$ for it to have $$w$$ zero bits.

What if it was subtraction instead? Suppose we use another $$m_{neg}$$ to make $$T-m_{neg}\cdot n \equiv 0 \mod R$$. In this case, the $$x=\frac{(T-m_{neg}n)}{R}$$ will be calculated as follows:

1. Subtract the low limbs. $$sub_{low}=T_{low} - m_{neg}n_{low}$$
2. Subtract the high limbs. $$sub_{high}=T_{high} - m_{neg}n_{high}$$
3. Subtract 1 from the $$sub_{high}$$ if there was an underflow in the low subtraction.
4. Set  $$x=(sub_{low} + (sub_{high} \ll w)) \gg w$$ (which is simply $$x=sub_{high}$$).

Again, the low subtraction should have $$w$$ trailing zero bits, so it can be only 0 because $$-2^{w}$$ is actually impossible. This is due to the simple fact that both operands are $$w$$-bit integer which are in the range $$[0, 2^w)$$. Hence, the subtraction will never underflow. With this observation, the calculation is simplified to:

1. Subtract the high limbs. $$sub_{high}=T_{high} - m_{neg}n_{high}$$
2. Set $$x=(sub_{low} + (sub_{high} \ll w)) \gg w$$ (which is simply $$x=sub_{high}$$)

This effectively replaces 3 addition operations with 1 subtraction and such $$m'$$ can be easily calculated as:

$$
m_{neg}=T\cdot (n^{-1} \mod R) \mod R
$$

Thus, in this scheme, our precomputed value would be $$n^{-1} \mod R$$. Also, the range can be calculated as follows:

$$
\begin{align*}
0\leq T&<n\cdot R &\Rightarrow 0&\leq(T\gg w)=\frac{T}{R}&<n \\
0\leq m_{neg}&< R &\Rightarrow 0&\leq(m_{neg}\cdot n \gg w)=n\cdot \frac{m}{R}&<n \\
\\&&-n&<((T-m_{neg}\cdot n)\gg w)&<n
\end{align*}
$$

**Overview of `REDC'(T)`:**

Given  $$T < n\cdot R$$, and precomputed $$n_{inv} \coloneqq n^{−1} \mod R$$.

1. Calculate $$m=(T⋅n_{inv}) \mod R=\mathsf{lowbits}_w(\mathsf{lowbits}_w(T)\cdot n_{inv})$$. Essentially, a $$w$$-bit non extended multiplication.
2. Compute $$x=(T-m⋅n)/R$$. (Involves one extended multiplication $$m\cdot n$$, 1 limb subtraction, and one right shift by $$k$$).
3. Return $$x+n$$ if $$x< 0$$, otherwise return $$x$$.

{% hint style="warning" %}
Subtraction variant can be only applied to single precision case where $$n < R$$. Otherwise, $$T_{low}$$ is actually multi-precision and $$T_{low} - mn_{low}$$ is non-zero and only the last limb will be zero.
{% endhint %}

### Summary of Montgomery Reduction

With some precomputation and using Montgomery Reduction $$\mathsf{REDC}$$, we can optimize the $$\mathsf{doMontMul}(\bar{a}, \bar{b})$$, $$\mathsf{toMont}(x)$$, and $$\mathsf{fromMont}(\bar{x})$$ operations as following:

Precompute $$Rsq' := R^2 \mod n$$ and $$n':=-(n^{-1})\mod R$$:

$$
\begin{aligned}
\mathsf{toMont}(x)&=x\cdot R \mod n = \mathsf{REDC}(x\cdot R^2) = \mathsf{REDC}(x\cdot Rsq')\\
\mathsf{fromMont}(\bar{x}) &= \mathsf{REDC}(\bar{x}) \\
\mathsf{doMontMul}(\bar{a},\bar{b})&=\mathsf{REDC}(\bar{a}\cdot\bar{b})
\end{aligned}
$$

## Multi-precision Montgomery Reduction

### Separated Operands Scanning (SOS)

#### Iterative Partial Reduction

In a multi-precision setting, suppose the modulo $$n$$ has $$l$$ limbs. Let $$R=B^l$$, where $$w$$ is limb bit width and $$B=2^w$$. To calculate $$T\cdot R^{-1}=T\cdot B^{-l}\mod n$$, the multi-precision Montgomery Reduction operates iteratively on:

$$
T = T_{2l-1}\cdot B^{2l-1} + \dots + T_l\cdot B^l + T_{l-1}\cdot B^{l-1} + \dots + T_0
$$

where $$T_i$$ represents limb values. At each iteration, the limbs of $$T$$ are reduced one by one which is done by partial reduction. The first partial reduction looks like so:

$$
\begin{align*}
T^{(1)}=T\cdot B^{-1} \mod n &= T_{2l-1}\cdot B^{2l-2} + \dots + T_l\cdot B^{l-1} + T_{l-1}\cdot B^{l-2} + \dots + T_1 + (T_0\cdot B^{-1} \mod n) &\mod n\\ 
&= T^{(1)}_{2l-1}B^{2l-2} + \dots + T^{(1)}_l\cdot B^{l-1} + T^{(1)}_{l-1}\cdot B^{l-2} + \dots  + T^{(1)}_1 &\mod n
\end{align*}
$$

And after the next partial reduction, we will have:

$$
\begin{align*}
T^{(2)}=T\cdot B^{-2} \mod n &= T^{(1)}_{2l-1}\cdot B^{2l-3} + \dots + T^{(1)}_l\cdot B^{l-2} + T^{(1)}_{l-1}\cdot B^{l-3} + \dots + T^{(1)}_2+(T^{(1)}_1\cdot B^{-1} \mod n) &\mod n\\
&=T^{(2)}_{2l-1}\cdot B^{2l-3} + \dots + T^{(2)}_l\cdot B^{l-2} + T^{(2)}_{l-1}\cdot B^{l-3} + \dots + T^{(2)}_2 &\mod n\\
\end{align*}
$$

and so on. After $$l$$ iterations, we have $$T\cdot B^{-l}\mod n$$ with $$l$$ limbs. The values of limbs change at each round because $$T^{(i)}_i \cdot B^{-1} \mod n$$ is also a multi-precision integer.

#### How to multiply by the inverse of $$B$$?

At round $$i$$, we need to calculate $$(T^{(i-1)}_{i-1}\cdot B^{-1} \mod n)$$. This can be done efficiently, similar to the single-precision case by adding some multiple of $$n$$ to make it divisible by $$B$$:&#x20;

1. Precompute $$n'\coloneqq -n^{-1}\mod B$$.
2. Calculate $$m_i\coloneqq T^{(i-1)}_{i-1}\cdot n' \mod B$$. Then we have $$T^{(i-1)}_{i-1} + m_i\cdot n \equiv  T^{(i-1)}_{i-1} -T^{(i-1)}_{i-1}\cdot n^{-1}\cdot n \equiv 0\mod B$$
3. Calculate $$T^{(i-1)}_{i-1}\cdot B^{-1}  \equiv (T^{(i-1)}_{i-1} + m_i\cdot n)\cdot B^{-1}\equiv (T^{(i-1)}_{i-1} + m_i\cdot n)\gg w\mod n$$ .

Since $$m_i < B$$ and $$T^{(i-1)}_{i-1} < B$$, we have

$$
(T^{(i-1)}_{i-1}  +m_i\cdot n) / B \leq (B-1 + (B-1)\cdot n)/B=n + (B-1-n)/B < n
$$

which means after step 3, we don't have to worry about subtracting $$n$$.

#### How many limbs are needed?

While trying to reduce, we add $$m_i\cdot n$$ every round which can cause potential overflow in the largest limb. Let's calculate how much we added in total. If we omit dividing by $$B$$, at each round, we can see that at each round $$i$$, we add $$m_i \cdot n\cdot B^{i-1}$$. This can be confusing but if you think of round $$i$$ as adding some value $$m_i\cdot n$$ to convert the $$(i-1)$$-th limb to 0, it can be more easier to see.

$$
\text{Total added}=\sum_{i=1}^lm_i\cdot n \cdot B^{i-1}<n\sum_{i=1}^lB \cdot B^{i-1}=n\cdot B\bigg(\frac{B^l-1}{B-1}\bigg)<n\cdot R
$$

This gives that total sum can be $$T+n\cdot R <R^2+n\cdot R < 2R^2=2\cdot2^{2lw}=2^{2lw+1}$$ which needs $$2l+1$$ limbs. For $$T$$, we already need $$2l$$ limbs and after the first step, the least significant limb will be 0's which can be discarded. In the first round, we have a maximum value of  $$T+n<n\cdot R + n=n(R+1)\leq(R-1)(R+1)=R^2-1$$ so it fits within the $$2l$$ limbs. So, if we discard the least significant limb after first round, we don't actually need extra limb space to handle the overflowing.

#### Summary of the Iteration step

A complete round $$i$$ of partial reduction can be summarized by the following steps:

1. Compute $$m_i=n' \cdot T^{(i-1)}_{i-1} \mod R$$ (which costs $$1\times 1$$ limb multiplication).
2. Compute $$m_i\cdot n$$ (which costs $$1 \times l$$ limb multiplication).
3. Set $$T^{(i)}=(T^{(i-1)} + m_i\cdot n ) \gg w$$.

Now, let's calculate the upper bound on the final result of the reductions. First of all, we saw above that, the total added value has an upper bound of $$n\cdot R$$, and using $$T<n\cdot R$$, we have:&#x20;

$$
T^{(l)} < \frac{T+n\cdot R}{R}<\frac{2n\cdot R}{R}<2n
$$

Therefore, the reduced value may require one additional subtraction to bring the value within the range $$[0,\dots,n-1]$$. Since we need $$l+1$$ limb multiplications for steps 1 and 2, over $$l$$ rounds, we will need a total of $$l^2+l$$ limb multiplications.

<figure><img src="../../../.gitbook/assets/image (137).png" alt=""><figcaption><p><a href="https://www.microsoft.com/en-us/research/wp-content/uploads/1996/01/j37acmon.pdf">Analyzing and Comparing Montgomery Multiplication Algorithms</a></p></figcaption></figure>

### SOS with Ingonyama Optimization

We have seen that we need to do $$l^2+l$$ multiplications per round for multi-precision Montgomery Reduction. However, [Ingonyama showed](https://hackmd.io/@Ingonyama/Barret-Montgomery#Barrett-Montgomery-duality) that we can reduce it to $$l^2+1$$. Let's see what happens if we just precompute $$B'=B^{-1} \mod n$$ and multiply the free coefficients with $$B'$$. Then, the partial reduction round $$i$$ becomes:

1. Compute $$m'_i=T^{(i-1)}_{i-1}\cdot B'$$ which is $$1\times l$$ limb multiplication, resulting in a $$l+1$$ limb integer.
2. Set $$T^{(i)}=(T^{(i-1)}\gg w) + m'_i$$.

Then, similar to the previous calculation, the upper bound on the total addition will become:

$$
\text{Total added}=\sum_{i=1}^l m'_i \cdot B^{i-1}<\sum_{i=1}^l n\cdot B \cdot B^{i-1}=n\cdot B\bigg(\frac{B^l-1}{B-1}\bigg)<n\cdot R
$$

So it hasn't changed. Which means the upper bound on the value after $$l$$ reductions will also be the same:

$$
T^{(l)} < \frac{T+n\cdot R}{R}<\frac{2n\cdot R}{R}<2n
$$

This gives us the total cost of $$l\cdot l =l^2$$  limb multiplications. Ingonyama's article mentions that this reduction cannot be applied in the last step because $$T^{(l)}=(T^{(l-1)}\gg w) + m'_i$$ will result in a $$l+1$$ limb integer while in the traditional case, we have $$l$$ limbs. Indeed it is true since we add it after the bit shift unlike the original version but if the result is less than $$2\cdot n$$, it shouldn't matter.

### Coarsely Integrated Operands Scanning (CIOS)

In the SOS method above, we iteratively divided $$t$$ by $$B$$ $$l$$ times where $$B$$ denotes the limb base ($$2^w$$) and $$l$$ denotes the number of the limbs. $$l$$ iterations gives the desired result $$t \cdot R^{-1} \mod n$$ since $$R = B^l$$. Now we can discard the lower $$l$$ limbs, which is technically equivalent to dividing by $$R$$.  &#x20;

Naively speaking, SOS algorithm follows this sequence:&#x20;

1. Multiply two large integers in Montgomery form using a [textbook method](https://en.wikipedia.org/wiki/Multiplication_algorithm#Long_multiplication!).
2. Perform Montgomery reduction to produce $$\bar{c} = \bar{a} \cdot \bar{b}$$ from $$\bar{a} \cdot \bar{b} \cdot R$$.

which results in its namesake "**separated"** in SOS.&#x20;

CIOS, on the other hand, is different from SOS in that it interleaves the multiplication and reduction steps, limb by limb.&#x20;

#### Pseudocode

Now we alternate between iterations of the outer loops for multiplication and reduction.

```
for i = 0 to l-1
    C := 0
    for j = 0 to l-1                         // a * b[i]
        (C, t[j]) := t[j] + a[j]*b[i] + C  
    (t[l+1], t[l]) := t[l] + C.              
    C := 0
    m := t[0]*n'[0] mod W                    // Partial m for t[0] only
    (C, _) := t[0] + m*n[0]
    for j = 1 to s-1                         // Add(t, m*n) >> w 
        (C, t[j-1]) := t[j] + m*n[j] + C
    (C, t[l-1]) := t[l] + C                   
    t[l] := t[l+1] + C                       
```

Interleaving multiplication and reduction is valid since the value of $$m$$ in the $$i$$-th iteration of the outer loop for reduction depends only on the value $$t[i]$$, which is completely computed by the $$i$$-th iteration of the outer loop for the multiplication.&#x20;

### Cost analysis

|                | CIOS              | SOS               | SOS + Ingonyama    |
| -------------- | ----------------- | ----------------- | ------------------ |
| Addition       | $$4l^2 + 4l + 2$$ | $$4l^2 + 4l + 2$$ | $$4l^2 + 4l + 2$$  |
| Multiplication | $$2l^2 +l$$       | $$2l^2 +l$$       | $$2l^2 +1$$        |
| Memory space   | $$l+3$$           | $$2l+ 2$$         | $$2l+ 2$$          |

SOS with Ingonyama's optimization saves $$l-1$$ multiplications ($$2l^2 + 1$$ vs. $$2l^2 + l$$) whereas CIOS saves $$l-1$$ memory space compared to SOS ($$2l+2$$ vs. $$l+3$$) owing to not storing the total $$t$$. CIOS can be further optimized using [EdMSM's trick](../../abstract-algebra/elliptic-curve/msm/edmsm.md).&#x20;

## References

* [https://en.wikipedia.org/wiki/Montgomery\_modular\_multiplication](https://en.wikipedia.org/wiki/Montgomery_modular_multiplication)
* [https://hackmd.io/@Ingonyama/Barret-Montgomery#Barrett-Montgomery-duality](https://hackmd.io/@Ingonyama/Barret-Montgomery#Barrett-Montgomery-duality)
* [Analyzing and Comparing Montgomery Multiplication Algorithms](https://www.microsoft.com/en-us/research/wp-content/uploads/1996/01/j37acmon.pdf)
* [EdMSM: Multi-Scalar-Multiplication for SNARKs and Faster Montgomery multiplication](https://eprint.iacr.org/2022/1400.pdf)

> Written by [BATZORIG ZORIGOO](https://app.gitbook.com/u/lqk5Tx9zY4XYVRfF3ReRDiOhbxG3 "mention") and [Carson Lee](https://app.gitbook.com/u/Hm5RHrPlu2fbxwXnpUgxmvXVTwB3 "mention") from [A41](https://www.a41.io/)
