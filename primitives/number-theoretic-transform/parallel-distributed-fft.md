# Parallel/Distributed FFT

## Parallel FFT

Suppose we are given a polynomial $$P(X) = \sum_{i=0}^{2^{2n} - 1} c_i X^i$$. The Discrete Fourier Transform (DFT) of this polynomial is defined as:

$$
\hat{c}_j = \sum_{i=0}^{2^{2n} - 1} c_i \cdot \omega^{ij}, \quad \text{where } 0 \le j < 2^{2n}
$$

Here, $$\omega$$ is a primitive $$2^{2n}$$-th root of unity.

We can decompose this equation using index splitting as follows (where $$0 \le j_0, j_1 < 2^n$$):

$$
\begin{align*}
\hat{c}_{j_1 \cdot 2^n + j_0} 
&= \sum_{i_0 = 0}^{2^n - 1} \sum_{i_1 = 0}^{2^n - 1} c_{i_1 \cdot 2^n + i_0} \cdot \omega^{(i_1 \cdot 2^n + i_0)(j_1 \cdot 2^n + j_0)} \\
&= \underbrace{
\left(\sum_{i_0 = 0}^{2^n - 1} 
\underbrace{\left( \omega^{i_0 j_0} 
\underbrace{
\left( 
\sum_{i_1 = 0}^{2^n - 1} c_{i_1 \cdot 2^n + i_0} \cdot (\omega^{2^n})^{(j_1 \cdot 2^n + j_0)i_1} 
\right)}_{\text{Step 1}}
\right)}_{\text{Step 2}} 
\cdot (\omega^{2^n})^{i_0 j_1}\right)}_{\text{Step 3}}
\end{align*}
$$

### Example ($$n=2$$)

Let $$P(X) = c_0 + c_1 X + \cdots + c_{15} X^{15}$$, and suppose $$\omega^{16} = 1$$. Define the following input vectors:

$$
\begin{aligned} 
\mathbf{c}^{(0)} &= (c_0, c_4, c_8, c_{12}) \\ 
\mathbf{c}^{(1)} &= (c_1, c_5, c_9, c_{13}) \\ 
\mathbf{c}^{(2)} &= (c_2, c_6, c_{10}, c_{14}) \\ 
\mathbf{c}^{(3)} &= (c_3, c_7, c_{11}, c_{15}) \\ 
\end{aligned}
$$

#### **Step 1: Perform FFT on each** $$\mathbf{c}^{(i_0)}$$

$$
\begin{aligned}
\widehat{\mathbf{c}^{(0)}} &= (\widehat{c^{(0)}}_0, \widehat{c^{(0)}}_1, \widehat{c^{(0)}}_2, \widehat{c^{(0)}}_3) \\
\widehat{\mathbf{c}^{(1)}} &= (\widehat{c^{(1)}}_0, \widehat{c^{(1)}}_1, \widehat{c^{(1)}}_2, \widehat{c^{(1)}}_3) \\
\widehat{\mathbf{c}^{(2)}} &= (\widehat{c^{(2)}}_0, \widehat{c^{(2)}}_1, \widehat{c^{(2)}}_2, \widehat{c^{(2)}}_3) \\
\widehat{\mathbf{c}^{(3)}} &= (\widehat{c^{(3)}}_0, \widehat{c^{(3)}}_1, \widehat{c^{(3)}}_2, \widehat{c^{(3)}}_3)
\end{aligned}
$$

Each transform is defined by:

$$
\widehat{c^{(i_0)}}_{j_0} = \sum_{i_1 = 0}^3 c_{4 \cdot i_1 + i_0} \cdot (\omega^4)^{(4j_1 + j_0)i_1}
$$

#### **Step 2: Construct** $$\mathbf{z}^{[j_0]}$$

$$
\begin{aligned}
\mathbf{z}^{[0]} &= (\widehat{c^{(0)}}_0, \widehat{c^{(1)}}_0, \widehat{c^{(2)}}_0, \widehat{c^{(3)}}_0) \\
\mathbf{z}^{[1]} &= (\widehat{c^{(0)}}_1, \omega^1 \widehat{c^{(1)}}_1, \omega^2 \widehat{c^{(2)}}_1, \omega^3 \widehat{c^{(3)}}_1) \\
\mathbf{z}^{[2]} &= (\widehat{c^{(0)}}_2, \omega^2 \widehat{c^{(1)}}_2, \omega^4 \widehat{c^{(2)}}_2, \omega^6 \widehat{c^{(3)}}_2) \\
\mathbf{z}^{[3]} &= (\widehat{c^{(0)}}_3, \omega^3 \widehat{c^{(1)}}_3, \omega^6 \widehat{c^{(2)}}_3, \omega^9 \widehat{c^{(3)}}_3)
\end{aligned}
$$

Each $$\mathbf{z}^{[j_0]}$$ can be defined generally as:

$$
\mathbf{z}^{[j_0]} := \left( \widehat{c^{(0)}}_{j_0}, \omega^{j_0} \widehat{c^{(1)}}_{j_0}, \omega^{2j_0} \widehat{c^{(2)}}_{j_0}, \omega^{3j_0} \widehat{c^{(3)}}_{j_0} \right)
$$

#### **Step 3: Final FFT on each** $$\mathbf{z}^{[j_0]}$$

$$
\begin{aligned}
\widehat{\mathbf{z}^{[0]}} &= (\hat{c}_0, \hat{c}_4, \hat{c}_8, \hat{c}_{12}) \\
\widehat{\mathbf{z}^{[1]}} &= (\hat{c}_1, \hat{c}_5, \hat{c}_9, \hat{c}_{13}) \\
\widehat{\mathbf{z}^{[2]}} &= (\hat{c}_2, \hat{c}_6, \hat{c}_{10}, \hat{c}_{14}) \\
\widehat{\mathbf{z}^{[3]}} &= (\hat{c}_3, \hat{c}_7, \hat{c}_{11}, \hat{c}_{15})
\end{aligned}
$$

Each transform is again a standard FFT:

$$
\widehat{z^{[j_0]}}_{j_1} = \sum_{i_0 = 0}^3 {z^{[j_0]}}_{i_0} \cdot (\omega^4)^{i_0 j_1}
$$

#### **Verification Examples**

* $$\widehat{z^{[1]}}_2 = \hat{c}_9$$ $$(j_1 = 2, j_0 = 1)$$

$$
\begin{align*}
\widehat{z^{[1]}}_2 &= {z^{[1]}}_0  + {z^{[1]}}_1 \omega^{8} + {z^{[1]}}_2 \omega^{16} + {z^{[1]}}_3 \omega^{24} \\ 
&= \widehat{c^{(0)}}_1  + \widehat{c^{(1)}}_1 \omega^{9} + \widehat{c^{(2)}}_1 \omega^{18} + \widehat{c^{(3)}}_1 \omega^{27} \\ 
&= \sum_{i_1=0}^3 c_{4i_1} (\omega^4)^{9i_1}  + \sum_{i_1=0}^3 c_{4i_1  + 1} (\omega^4)^{9i_1}\omega^9 + \sum_{i_1=0}^3 c_{4i_1  + 2} (\omega^4)^{9i_1}\omega^{18}  + \sum_{i_1=0}^3 c_{4i_1  + 3} (\omega^4)^{9i_1}\omega^{27} \\
&= \hat{c}_9
\end{align*}
$$

* $$\widehat{z^{[2]}}_1 = \hat{c}_6$$ $$(j_1 = 1, j_0 = 2)$$

$$
\begin{align*}
\widehat{z^{[2]}}_1 &= {z^{[2]}}_0  + {z^{[2]}}_1 \omega^{4} + {z^{[2]}}_2 \omega^{8} + {z^{[2]}}_3 \omega^{12} \\ 
&= \widehat{c^{(0)}}_2  + \widehat{c^{(1)}}_2 \omega^{6} + \widehat{c^{(2)}}_2 \omega^{12} + \widehat{c^{(3)}}_2 \omega^{18} \\ 
&= \sum_{i_1=0}^3 c_{4i_1} (\omega^4)^{6i_1}  + \sum_{i_1=0}^3 c_{4i_1  + 1} (\omega^4)^{6i_1} \omega^6 + \sum_{i_1=0}^3 c_{4i_1  + 2} (\omega^4)^{6i_1} \omega^{12}  + \sum_{i_1=0}^3 c_{4i_1  + 3} (\omega^4)^{6i_1} \omega^{18} \\ 
&= \hat{c}_6
\end{align*}
$$

The inverse FFT is structurally similar, but we do not cover it in this article.

## Distributed FFT

<figure><img src="../../.gitbook/assets/Screenshot 2025-04-30 at 8.32.32 AM.png" alt=""><figcaption></figcaption></figure>

The Parallel FFT algorithm described above can be implemented as a **Distributed FFT** by mapping it onto the MapReduce model. The diagram above illustrates how the same example from the parallel FFT is executed within the MapReduce paradigm.

* **Map phase:** Each executor performs the first-stage FFTs over $$\mathbf{c}^{(i_0)}$$
* **Shuffle (transpose + twist):** Results are reorganized into $$\mathbf{z}^{[j_0]}$$
* **Reduce phase:** Second-stage FFTs are computed over each $$\mathbf{z}^{[j_0]}$$

This structure enables FFT computations over extremely large input sizes to be parallelized and scaled effectively in distributed systems like [Spark](https://spark.apache.org/) or [Hadoop](https://hadoop.apache.org/).

## Reference

* [https://www.csd.uwo.ca/\~mmorenom/SNC-11-file-for-ACM/p54-Sze.pdf](https://www.csd.uwo.ca/~mmorenom/SNC-11-file-for-ACM/p54-Sze.pdf)

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
