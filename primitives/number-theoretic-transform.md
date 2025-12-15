---
description: 'Presentation: https://youtu.be/jeHQtLduHN8'
---

# Number Theoretic Transform

## Preliminaries

### Discrete Linear Convolution

> **Convolution** is a mathematical operation that blends two functions (or signals) to produce a third function, which shows how one function modifies or overlaps with the other.

A convolution is **linear** if it satisfies the properties of additivity and homogeneity. This means that the convolution of a linear combination of sequences is the same as the corresponding linear combination of their convolutions. For example, $$(A+B)\circ C= A\circ C + B\circ C$$

So **discrete linear convolution** is when it is working on sequences of integers rather than on continuous functions.

### Polynomial Multiplication

**Definition 1**: Suppose that $$G(x)$$ and $$H(x)$$ are polynomials of degree $$n − 1$$ in the ring $$\mathbb{Z}_q[x]$$ where $$q \in \mathbb{Z}$$ and $$x$$ is the polynomial variable, a polynomial multiplication of $$G(x)$$ and $$H(x)$$ is defined as:

$$
Y(x)=G(x)\cdot H(x)=\sum_{k=0}^{2(n-1)}y_k x^k \tag{1}
$$

where $$y_k=\sum_{i=0}^{k}g_i h_{k-i} \mod q$$, $$\bm{g}$$ and $$\bm{h}$$ are the polynomial coefficients of $$G(x)$$ and $$H(x)$$ respectively.

Polynomial multiplication is equivalent to a **discrete linear convolution** between the coefficients' vectors $$\bm{g}=(g_0,\dots,g_n)$$ and $$\bm{h}=(h_0,\dots,h_n)$$.

### Cyclic Convolution

**Definition 2**: Suppose that $$G(x)$$ and $$H(x)$$ are polynomials of degree $$n − 1$$ in the quotient ring $$\mathbb{Z}_q[x]/(x^n − 1)$$ where $$q\in\mathbb{Z}$$. A **cyclic convolution** or **positive wrapped convolution (PWC)** is defined as:

$$
\mathsf{PWC}(x)=\sum^{n-1}_{k=0}c_kx^k\tag{2}
$$

where $$c_k=\sum^{k}_{i=0}g_i h_{k-i} + \sum^{n-1}_{i=k+1}g_i h_{k+n-i} \mod q$$. If $$Y(x)$$ from equation $$(1)$$ is the result of their linear convolution in the ring $$\mathbb{Z}_q[x]$$, it can also be defined as:

$$
\mathsf{PWC}(x)=Y(x) \mod (x^n-1) \tag{3}
$$

Naive method to calculate the $$\mathsf{PWC}$$ is through a polynomial multiplication followed by a long division. This method has $$O(n^2)$$ complexity.

> The term "cyclic" stems from the fact that $$x^n$$ _cycles back to_ $$1$$ in this quotient ring. For example, $$F(x)=x^n$$ will map to $$F(x)=1$$ in this quotient ring.

### Negacyclic convolution

**Definition 3**: Suppose that $$G(x)$$ and $$H(x)$$ are polynomials of degree $$n − 1$$ in the quotient ring $$\mathbb{Z}[x]/(x^n + 1)$$ where $$q \in \mathbb{Z}$$. A **negacyclic convolution** or **negative wrapped convolution**, $$\mathsf{NWC}(x)$$ is defined as:

$$
\mathsf{NWC}(x)=\sum_{k=0}^{n-1}c_k x^k \tag{4}
$$

where $$c_k=\sum_{i=0}^k g_i h_{k-i} - \sum_{i=k+i}^{n-1}g_i h_{k+n-i}\mod q$$. If $$Y(x)$$ from equation $$(1)$$ is the result of their linear convolution in the ring $$\mathbb{Z}[x]$$, it also can be defined as:

$$
\mathsf{NWC}(x)=Y(x) \mod (x^n + 1) \tag{5}
$$

Note that the only difference between cyclic and negacyclic convolution is the divisor.

> The term "negacyclic" stems from the fact that $$x^n$$ _cycles back to_ $$-1$$ in this quotient ring. For example, $$F(x)=x^n$$ will map to $$F(x)=-1$$ in this quotient ring.

## Number Theoretic Transform

> Many researchers do not differentiate the term NTT and FFT-based algorithms to calculate NTT, which creates confusion when understanding the topic. This report refers to the transformation itself as NTT and the FFT-like algorithms as fast-NTT

The classical NTT has quadratic complexity of $$O(n^2)$$ while fast-NTT algorithms have a more efficient quasi-linear complexity $$O(n\log n).$$&#x20;

> Remember that for NTT, we are particularly interested in the evaluations at the roots of the quotient polynomial. ($$A\cdot B= C + \mathsf{QuotientPoly} \cdot c$$ and we want the $$\mathsf{QuotientPoly}$$ to vanish).

**Definition 4**: Let $$\mathbb{Z}_q$$ be an integer ring modulo $$q$$, and $$n − 1$$ is the polynomial degree of $$G(x)$$ and $$H(x)$$. Such rings have a multiplicative identity (unity) of 1. Define $$\omega$$ as primitive $$n$$-th root of unity in $$\mathbb{Z}_q$$ if and only if:

$$
\omega^n\equiv 1 \mod q \tag{6}
$$

and

$$
\omega^k\not\equiv 1\mod q\tag{7}
$$

for all $$k=1,\dots,n-1$$.

One thing to note is that the primitive $$n$$-th root of unity in a ring $$\mathbb{Z}_q$$ might not be unique. For example, in a ring $$\mathbb{Z}_{7}$$, primitive 3rd root of unity can be $$2$$ and $$4$$. The choice of value of $$\omega$$ will be important in calculating $$\mathsf{NWC}$$ and $$\mathsf{PWC}$$.

{% include "../.gitbook/includes/definition-5-the-number-th... (1).md" %}

**Definition 6**: The Inverse of Number Theoretic Transform (INTT) of an NTT vector $$\hat{\boldsymbol{a}}$$ is defined as $$\boldsymbol{a} = \mathsf{INTT}(\hat{\boldsymbol{a}})$$, where:

$$
a_i=n^{-1}\sum^{n-1}_{j=0}w^{-ij}\hat{a}_j \mod q\tag{9}
$$

and $$j=0,1,\dots,n-1$$.

> Note that the INTT has a very similar formula to NTT. The only differences are $$\omega$$ replaced by its inverse in $$\mathbb{Z}_q$$ and a $$n^{−1}$$ scaling factor. It always holds that $$\bm{a} = \mathsf{INTT}(\mathsf{NTT}(\bm{a}))$$.

In the ZK scene, it is common to work on a polynomial with some max degree $$n-1$$. This just means that we are working on polynomials from the quotient ring $$\mathbb{Z}_q[x]/(x^n − 1)$$. In such rings, we must utilize positive-wrapped convolution.

In the context of PQC and HE, the chosen ring is mostly $$\mathbb{Z}_q[x]/(x^n + 1)$$ instead of $$\mathbb{Z}_q[x]/(x^n − 1)$$. One must calculate the polynomial multiplications via the negative-wrapped convolution in such rings. To calculate negative-wrapped convolution, one needs the primitive $$2n$$-th root of unity, $$\psi$$.

**Definition 7**: Let $$\mathbb{Z}_q$$ be an integer ring modulo $$q$$, and $$n - 1$$ is the polynomial degree of $$G(x)$$ and $$H(x)$$ and $$\omega$$ is its primitive $$n$$-th root of unity. Define $$\psi$$ as the primitive $$2n$$-th root of unity if and only if:

$$
(\psi^2\equiv\omega\mod q)\land (\psi^n \equiv -1 \mod q) \tag{10}
$$

**Example.** In a ring $$\mathbb{Z}_{7681}$$ and $$n = 4$$, when $$\omega = 3383$$, the value of $$\psi$$ can be 1925 or 5756 as $$1925^2 \equiv 5756^2 \equiv 3383 \mod 7681$$ and $$1925^4 \equiv 5756^4 \equiv 7680 \equiv −1 \mod 7681$$. Therefore, one can choose the value of $$\psi = 1925$$ or $$\psi = 5756$$.

**Definition 8**: The Negative-Wrapped Number Theoretic Transform ($$\mathsf{NTT}^\psi$$) of a vector of polynomial coefficients $$\bm{a}$$ is defined as $$\hat{\boldsymbol{a}} = \mathsf{NTT^\psi}(\boldsymbol{a})$$, where:

$$
\hat{a}_j = \sum^{n−1}_{i=0} \psi^{2ij+i}a_i \mod q \tag{11}
$$

and $$j=0,1,\dots,n-1$$.  Here, $$\hat{\bm{a}}$$ denotes the evaluations of the polynomial defined by $$\bm{a}$$ over $$\{\psi^{1},\psi^{3},\dots,\psi^{2n-1}\}$$ which are the roots of $$(x^n+1)$$.

**Definition 9**: The Negative-Wrapped Inverse of Number Theoretic Transform (INTT) of an NTT vector $$\hat{\bm{a}}$$ is defined as $$\bm{a} = \mathsf{INTT}^{\psi^{-1}}(\hat{\bm{a}})$$, where:

$$
a_i=n^{-1}\sum_{j=0}^{n-1}\psi^{-i}\omega^{-ij}\hat{a}_j \mod q \tag{12}
$$

and $$i=0,1,\dots,n-1$$. Substituting $$\omega = \psi^2$$ yields:

$$
a_i=n^{-1}\sum^{n-1}_{j=0}\psi^{-(2ij+i)}\hat{a}_j\mod q \tag{17}
$$

To reduce the complexity and fasten the process of the matrix multiplication needed for the NTT transformation, one can use ‘‘divide and conquer’’ techniques by utilizing the periodicity and symmetry property of:

$$
\begin{aligned}\text{periodicity:}\ \ \ &\psi^{k+2n}& &=&\psi^k \\ \text{symmetry:}\ \ \ &\psi^{k+n}& &=&-\psi^k\end{aligned} \tag{18}
$$

### Cooley-Tukey (CT) Algorithm for Fast NTT

#### PWC case

From equation (8) for $$\mathsf{NTT}$$, one can separate the summation into two parts, based on odd and even indices:

$$
\begin{aligned}
\hat{a}_j& = \sum_{i=0}^{n-1}\omega^{ij}a_i\mod q \\ 
&=\sum_{i=0}^{n/2-1}\omega^{2ij}a_{2i} + \sum_{i=0}^{n/2-1}\omega^{2ij + j}a_{2i+1}\mod q\\
&=\sum_{i=0}^{n/2-1}\omega^{2ij}a_{2i} + \omega^{j}\sum_{i=0}^{n/2-1}\omega^{2ij}a_{2i+1}\mod q
\end{aligned}\tag{19}
$$

Similarly, we can get the equation for $$\hat{\bm{a}}_{j+\frac{n}{2}}$$ by using the fact that $$\omega^{\frac{n}{2}}\equiv-1 \mod q$$ :

$$
\begin{aligned}
\hat{a}_{j+\frac{n}{2}}& = \sum_{i=0}^{n-1}\omega^{i(j+\frac{n}{2})}a_i &\mod q \\
&=\sum_{i=0}^{n/2-1}\omega^{2i(j+\frac{n}{2})}a_{2i} + \sum_{i=0}^{n/2-1}\omega^{(2i+1)(j + \frac{n}{2})}a_{2i+1} &\mod q\\
&=\sum_{i=0}^{n/2-1}\omega^{2ij + {ni}}a_{2i} + \sum_{i=0}^{n/2-1}\omega^{2ij + j + ni + \frac{n}{2}}a_{2i+1} &\mod q \\ &=\sum_{i=0}^{n/2-1}\omega^{2ij}a_{2i} + \omega^{j+\frac{n}{2}}\sum_{i=0}^{n/2-1}\omega^{2ij}a_{2i+1} &\mod q \\
&=\sum_{i=0}^{n/2-1}\omega^{2ij}a_{2i} - \omega^{j}\sum_{i=0}^{n/2-1}\omega^{2ij}a_{2i+1} &\mod q\end{aligned}\tag{20}
$$

If we denote, $$A_j=\sum^{n/2-1}_{i=0}\omega^{2ij}a_{2i}$$ and $$B_j=\sum^{n/2-1}_{i=0}\omega^{2ij}a_{2i+1}$$, the equation $$(19)$$ and $$(20)$$ becomes:



$$
\begin{aligned}\hat{\boldsymbol{a}}_j&=A_j + \omega^j B_j \mod q\\\hat{\boldsymbol{a}}_{j+n/2}&=A_j-\omega^jB_j \mod q \end{aligned}\tag{21}
$$

With this finding, we can calculate $$A_j$$ and $$B_j$$ for $$j=0,\dots,\frac{n}{2}$$ and calculate $$\hat{\bm{a}}$$ by reusing them for the upper half. Notice that $$A_j$$ and $$B_j$$ can be calculated using $$\frac{n}{2}$$ point NTT so if we repeat recursively, we can achieve the best case scenario $$O(n\log n)$$ which occurs when $$n$$ is a power of 2.&#x20;

**Example.** Let's consider the case when $$\bm{a}=(1,2,3,4)$$.

$$
\hat{\bm{a}}=
   \begin{bmatrix} 
   \omega^{0} & \omega^{0} & \omega^{0} & \omega^{0}  \\
   \omega^{0} & \omega^{1} & \omega^{2} & \omega^{3}   \\
  \omega^{0} & \omega^{2} & \omega^{4} & \omega^{6}   \\
 \omega^{0} & \omega^{3} & \omega^{6} & \omega^{9}    
\end{bmatrix} 
\cdot 
\begin{bmatrix} 
   1  \\
   2  \\
   3  \\
   4  \\
   \end{bmatrix}
$$

$$
\hat{\bm{a}}=
   \begin{bmatrix} 
   \omega^{0} & \omega^{0} & \omega^{0} & \omega^{0}  \\
   \omega^{0} & \omega^{1} & \omega^{2} & \omega^{3}   \\
  \omega^{0} & \omega^{2} & \omega^{0} & \omega^{2}   \\
 \omega^{0} & \omega^{3} & \omega^{2} & \omega^{1}    
\end{bmatrix} 
\cdot 
\begin{bmatrix} 
   1  \\
   2  \\
   3  \\
   4  \\
   \end{bmatrix}
$$

$$
\hat{a}_0=\omega^0\cdot 1 + \omega^0\cdot 2 + \omega^0\cdot 3 + \omega^0\cdot 4 \\ 
\hat{a}_1=\omega^0\cdot 1 + \omega^1\cdot 2 + \omega^2\cdot 3 + \omega^3\cdot 4 \\
\hat{a}_2=\omega^0\cdot 1 + \omega^2\cdot 2 + \omega^0\cdot 3 + \omega^2\cdot 4 \\
\hat{a}_3=\omega^0\cdot 1 + \omega^3\cdot 2 + \omega^2\cdot 3 + \omega^1\cdot 4
$$

$$
\hat{a}_0=\omega^0\cdot (1 + 3) + \omega^0\cdot (2 + 4) \\ 
\hat{a}_1=\omega^0\cdot (1 -  3) + \omega^1\cdot (2 - 4) \\
\hat{a}_2=\omega^0\cdot (1 + 3) -\omega^0\cdot (2 + 4) \\
\hat{a}_3=\omega^0\cdot (1 - 3) - \omega^1\cdot (2 - 4)
$$

The idea is to calculate similar terms in the brackets once and then distribute the results instead of calculating them multiple times.

#### NWC case

From equation $$(11)$$ for $$\mathsf{NTT}^\psi$$, one can separate the summation into two parts similarly into following:

$$
\begin{aligned}\hat{\boldsymbol{a}}_j& = \sum_{i=0}^{n-1}\psi^{2ij+i}\boldsymbol{a}_i\mod q \\ & = \sum_{i=0}^{n/2-1}\psi^{4ij+2i}\boldsymbol{a}_{2i} + \sum_{i=0}^{n/2-1}\psi^{4ij + 2j + 2i + 1}\boldsymbol{a}_{a_{2i+1}}&\mod q\\ &=\sum_{i=0}^{n/2-1}\psi^{4ij+2i}\boldsymbol{a}_{2i} + \psi^{2j+1}\sum_{i=0}^{n/2-1}\psi^{4ij + 2i}\boldsymbol{a}_{a_{2i+1}}&\mod q\end{aligned}\tag{22}
$$

Based on the $$\psi$$'s symmetry properties:

$$
\hat{\boldsymbol{a}}_{j+n/2}=\sum_{i=0}^{n/2-1}\psi^{4ij+2i}\boldsymbol{a}_{2i} - \psi^{2j + 1}\sum_{i=0}^{n/2-1}\psi^{4ij + 2i}\boldsymbol{a}_{a_{2i+1}}\mod q \tag{23}
$$

Let $$A_j=\sum^{n/2-1}_{i=0}\psi^{4ij + 2i}a_{2i}$$ and $$B_j=\sum^{n/2-1}_{i=0}\psi^{4ij+2i}a_{2i+1}$$, equations $$(22)$$ and $$(23)$$ become:

$$
\begin{aligned}\hat{\boldsymbol{a}}_j &= A_j + \psi^{2j+1}B_j\mod q\\ \hat{\boldsymbol{a}}_{j+n/2}&=A_j-\psi^{2j+1}B_j\mod q\end{aligned}\tag{24}
$$

With this finding, we can calculate $$A_j$$ and $$B_j$$ for $$j=0,\dots,\frac{n}{2}$$ and calculate $$\hat{\bm{a}}$$ by reusing them for the upper half. Notice that $$A_j$$ and $$B_j$$ can be calculated using $$\frac{n}{2}$$ point NTT so if we repeat recursively, we can achieve the best case scenario $$O(n\log n)$$ which occurs when $$n$$ is a power of 2.&#x20;

<figure><img src="../.gitbook/assets/image (112).png" alt=""><figcaption><p>Figure 1. Cooley-Tukey Butterfly graph notation</p></figcaption></figure>

Figure 1 illustrates how this butterfly operation is denoted as a graph. This figure represents the computation of $$A+\psi^kB$$ and $$A-\psi^k B$$.

### Gentleman-Sande (GS) Algorithm for Fast INTT

#### PWC case

For the INTT, instead of dividing the summation by even and odd indices, it is separated by the lower and upper half of the summation. From equation $$(10)$$ for $$\mathsf{INTT}$$ and ignoring $$n^{-1}$$ term:&#x20;

$$
\begin{aligned}
a_i&=\sum^{n-1}_{j=0}\omega^{-ij}\hat{a}_j &\mod q \\
&=\sum^{\frac{n}{2}-1}_{j=0}\omega^{-ij}\hat{a}_j+\sum^{\frac{n}{2}-1}_{j=0}\omega^{-i(j+\frac{n}{2})}\hat{a}_{j+\frac{n}{2}} &\mod q\\
&=\sum^{\frac{n}{2}-1}_{j=0}\omega^{-ij}\hat{a}_j+\omega^{-\frac{ni}{2}}\sum^{\frac{n}{2}-1}_{j=0}\omega^{-ij}\hat{a}_{j+\frac{n}{2}} &\mod q \\
&=\sum^{\frac{n}{2}-1}_{j=0}(\hat{a}_j+\omega^{-\frac{ni}{2}}\hat{a}_{j+\frac{n}{2}})\cdot\omega^{-ij} &\mod q
\end{aligned}\tag{25}
$$

If we consider the odd and even term separately, we get:

$$
\begin{align*}
a_{2i}&=\sum^{\frac{n}{2}-1}_{j=0}(\hat{a}_j+\omega^{-ni}\hat{a}_{j+\frac{n}{2}})\cdot\omega^{-2ij} &\mod q \\
&=\sum^{\frac{n}{2}-1}_{j=0}(\hat{a}_j+\hat{a}_{j+\frac{n}{2}})\cdot\omega^{-2ij} &\mod q
\end{align*} \tag{26}
$$

$$
\begin{aligned}
a_{2i+1}&=\sum^{\frac{n}{2}-1}_{j=0}(\hat{a}_j+\omega^{-ni-\frac{n}{2}}\hat{a}_{j+\frac{n}{2}})\cdot\omega^{-2ij -j} &\mod q \\
&=\sum^{\frac{n}{2}-1}_{j=0}((\hat{a}_j-\hat{a}_{j+\frac{n}{2}})\cdot\omega^{-j})\cdot\omega^{-2ij} &\mod q
\end{aligned}  \tag{27}
$$

Let $$A_{i}=\sum^{\frac{n}{2}-1}_{j=0}(\hat{a}_j+\hat{a}_{j+\frac{n}{2}})\omega^{-2ij}$$ and $$B_{i}=\sum^{\frac{n}{2}-1}_{j=0}\bigg((\hat{a}_j-\hat{a}_{j+\frac{n}{2}})\omega^{-j}\bigg)\omega^{-2ij}$$, equation $$(26)$$ and $$(27)$$ become:

$$
\begin{aligned}
a_{2i}&=A_{i}&\mod q\\
a_{2i+1}&=B_i &\mod q  \tag{27}
\end{aligned}
$$

With this finding, we just need to calculate $$A_i$$ and $$B_i$$ for $$0\leq i \leq \frac{n}{2}-1$$. Notice that $$A_i$$ and $$B_i$$ can be calculated using $$\frac{n}{2}$$ point INTT so if we repeat recursively, we can achieve the best case scenario $$O(n\log n)$$ which occurs when $$n$$ is a power of 2.&#x20;

#### NWC case

Similarly, continuing from equation $$(17)$$ for $$\mathsf{INTT}^\psi$$,  ignoring $$n^{−1}$$ term, we got:

$$
\begin{aligned}
a_i& = \sum_{j=0}^{n-1}\psi^{-(2ij+i)}\hat{a}_j&\mod q \\ 
& = \psi^{-i}\sum_{j=0}^{n-1}\psi^{-2ij}\hat{a}_j&\mod q \\ 
&=\psi^{-i}\Bigg(\sum_{j=0}^{\frac{n}{2}-1}\psi^{-2ij}\hat{a}_{j} + \sum_{j=0}^{\frac{n}{2}-1}\psi^{-2i(j+\frac{n}{2})}\hat{a}_{j+\frac{n}{2}}\Bigg)&\mod q\\
\end{aligned}\tag{28}
$$

If we consider the odd and even term, we get:

$$
\begin{aligned}
a_{2i}&=\psi^{-2i}\Bigg(\sum^{\frac{n}{2}-1}_{j=0}\psi^{-4ij}\hat{a}_j+\sum^{\frac{n}{2}-1}_{j=0}\psi^{-4i(j+\frac{n}{2})}\hat{a}_{j+\frac{n}{2}}\Bigg) &\mod q \\
&=\psi^{-2i}\sum^{\frac{n}{2}-1}_{j=0}\bigg(\hat{a}_j + \hat{a}_{j+\frac{n}{2}}\Bigg)\psi^{-4ij} &\mod q  \\
\end{aligned} \tag{29}
$$

$$
\begin{aligned}
a_{2i+1}&=\psi^{-2i-1}\Bigg(\sum^{\frac{n}{2}-1}_{j=0}\psi^{-2(2i+1)j}\hat{a}_j+\sum^{\frac{n}{2}-1}_{j=0}\psi^{-2(2i+1)(j+\frac{n}{2})}\hat{a}_{j+\frac{n}{2}}\Bigg) &\mod q \\
&=\psi^{-2i-1}\Bigg(\sum^{\frac{n}{2}-1}_{j=0}\psi^{-4ij-2j}\hat{a}_j+\sum^{\frac{n}{2}-1}_{j=0}\psi^{-4ij-2j-2ni - n}\hat{a}_{j+\frac{n}{2}}\Bigg) &\mod q \\
&=\psi^{-2i-1}\sum^{\frac{n}{2}-1}_{j=0}\bigg((\hat{a}_j - \hat{a}_{j+\frac{n}{2}})\psi^{-2j}\Bigg)\psi^{-4ij} &\mod q
\end{aligned}\tag{30}
$$

Let $$A_{i}=\sum^{\frac{n}{2}-1}_{j=0}\psi^{-(4ij+2i)}(\hat{a}_j + \hat{a}_{j+\frac{n}{2}})$$ and $$B_i=\sum^{\frac{n}{2}-1}_{j=0}\psi^{-(4ij+2i)}\bigg((\hat{a}_j - \hat{a}_{j+\frac{n}{2}})\psi^{-2j}\Bigg)$$, then equation $$(29)$$ and $$(30)$$ becomes:

$$
\begin{aligned}
a_{2i}&=A_i &\mod q\\
a_{2i+1}&=\psi^{-1}B_i &\mod q
\end{aligned}  \tag{31}
$$

With this finding, we just need to calculate $$A_i$$ and $$B_i$$ for $$0\leq i \leq \frac{n}{2}-1$$. Notice that $$A_i$$ and $$B_i$$ can be calculated using $$\frac{n}{2}$$ point INTT so if we repeat recursively, we can achieve the best case scenario $$O(n\log n)$$ which occurs when $$n$$ is a power of 2. <br>

<div align="center" data-full-width="false"><figure><img src="../.gitbook/assets/image (114).png" alt=""><figcaption><p>Figure 2. Gentleman-Sande (GS) butterfly graph notation</p></figcaption></figure></div>

Figure 2 illustrates how this butterfly operation is denoted as a graph. This figure represents the computation of $$(A+B)$$ and $$(A-B)\psi^k$$.

## References:

* [Conceptual Review on Number Theoretic&#x20;  Transform and Comprehensive  &#x20;Review on Its Implementations](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=\&arnumber=10177902) by A. Satriawan et al (2023)
* [Lifting a butterfly – A component-based FFT](https://www.researchgate.net/publication/220060688_Lifting_a_butterfly_-_A_component-based_FFT) by Sibylle Schupp (2003)

> Written by [Batzorig Zorigoo](https://app.gitbook.com/u/cO1lUla01ZW0seepO37jMHFTxg42 "mention") of Fractalyze
