---
description: >-
  This article outlines the goal and process of the STIR (Shift To Improve Rate)
  protocol as intuitively as possible.
---

# STIR

## Introduction

[**STIR**](https://eprint.iacr.org/2024/390) has achieved the fewest number of queries among IOPP (Interactive Oracle Proof of Proximity) systems based on RS codes. To achieve $$\lambda$$-bit security, the query complexity, which was $$O(\lambda \log d)$$ in the traditional FRI, can be reduced to $$O(\log d + \lambda \log \log d)$$. As the number of queries decreases, the verifier time also decreases, and particularly, the number of verifier hash operations is reduced. In the FRI family, proofs are often generated recursively, and reducing the constraints on the verifier hash in recursive proofs is particularly meaningful. Thanks to this optimization, the corresponding research paper was awarded Best Paper at CRYPTO 2024.

## Background

In **coding theory**, a **code** is a set of rules or patterns used to encode information, particularly to detect and correct errors that might occur during data transmission or storage. A **codeword** is an individual element within a code, representing a specific valid encoded message according to the code's rules.

Now, let’s briefly talk about **RS (Reed-Solomon) codes**. RS can be expressed as $$\mathsf{RS}(\mathbb{F}, N, d)$$, where $$\mathbb{F}$$ is the field, $$N$$ is the size of the evaluation domain, and $$d$$ is the maximum degree of the polynomial $$P$$. A codeword consists of the values evaluated from the polynomial $$P$$ over $$N$$.

<figure><img src="../../.gitbook/assets/image (56).png" alt=""><figcaption><p>Fig 1. The blue points represent an <span class="math">\mathsf{RS}(\mathbb{F}, N, 2)</span> codeword, where <span class="math">N</span>, the evaluation domain, has a size of 4. From left to right, if we label the graphs as 1 through 4, then graphs 2 and 3 illustrate possible valid <span class="math">\mathsf{RS}(\mathbb{F}, N, 2)</span> codewords. In contrast, the red points in graph 4 depict an invalid <span class="math">\mathsf{RS}(\mathbb{F}, N, 2)</span> codeword due to the degree constraint. The purple points indicate intersections between the blue and red lines.</p></figcaption></figure>

Codewords must correspond to polynomials of (at most) a prescribed degree d or else be classified as invalid. In the example above, our required $$d$$ is 1. Since the red codeword in the rightmost figure has a degree of 2, it does not satisfy the condition of being a degree of 1 and is therefore invalid; however as the red codewords in the middle two graphs are of degree 1, they are valid. From this, we can infer that given $$N$$ and $$d$$, uniquely valid codewords must differ from each other by at least $$N - d$$ (3 in this example) values.

An **Interactive Oracle Proof of Proximity (IOPP)** is a framework used to verify that a large dataset or computation is close to a specific structure (such as a low-degree polynomial) without requiring an exhaustive check of the entire dataset. In an IOPP, only a few random queries are made, meaning the verifier does not have direct access to the entire function. Instead, it verifies **proximity**—checking whether the function is sufficiently **close** to the intended structure.

Now, imagine that a function is $$\delta$$-far from the code. This means that there is a fraction $$\delta = \frac{N - d}{ N} = 1 - \frac{d}{ N}$$ of the function’s values that deviate from what would be expected if it belonged to the code. The remaining $$1−\delta$$ portion agrees with the code, meaning that these points behave as if the function were correct.

When the verifier makes only a limited number of queries, there’s a chance it might sample only from the portion of the function that agrees with the code—the $$1−\delta$$ part. If the verifier’s queries happen to avoid the parts where the function diverges (the $$\delta$$-far parts), completeness would require the verifier to conclude that the function is close to the code.

This is why, with a limited number of queries, there’s a trade-off: more queries increase confidence that the function is close to the code, while fewer queries make it easier for the function to “pass” without actually being in close proximity. This is why the probability $$(1−\delta)^t$$ becomes significant, where $$t$$ is the number of queries. This probability is known as the soundness error, and reducing it enhances the protocol’s security. Here rate ρ is defined as $$\frac{d + 1}{ N}$$, so $$\delta \approx 1 - \rho$$. Therefore, lowering the rate preserves security while minimizing the number of queries required.

## Protocol Explanation

<figure><img src="../../.gitbook/assets/image (57).png" alt=""><figcaption><p>Fig 2. A simple diagram of the query phase of the FRI(left) and STIR(right) protocol.</p></figcaption></figure>

Here, $$m$$ represents each round, and $$k$$ is a constant that is a power of 2.

STIR implements each round using four different techniques:

1\. $$k$$**-fold**: This method creates a degree $$\frac{d}{ k}$$ polynomial $$g$$ from a degree d polynomial $$f$$ using random values, with the following features.

* The verifier should be able to compute the next round’s evaluations using $$k$$ evaluations obtained from the previous round.
* The distance between $$f$$ and $$g$$ should be preserved with high probability.

When creating the codeword, FRI and STIR use domains of different sizes, resulting in a different rate, as shown in the table below. This difference allows for fewer queries to achieve the same security level.

|                              | FRI                      | STIR                                                                  |
| ---------------------------- | ------------------------ | --------------------------------------------------------------------- |
| folded degree at round $$m$$ | $$\frac{d}{k ^ m}$$      | $$\frac{d}{k ^ m}$$                                                   |
| domain size at round $$m$$   | $$\frac{N}{k ^ m}$$      | $$\frac{N}{2 ^ m}$$                                                   |
| rate at round $$m$$          | $$\frac{d+1}{N} = \rho$$ | $$\frac{d+1} {N} \cdot (\frac{2}{k})^m = \rho \cdot (\frac{2}{k})^m$$ |

2\. **Quotienting**: After creating a polynomial through $$k$$**-folding**, its validity needs to be checked. Since FRI and STIR use domains of different sizes, the method used in FRI cannot be applied here. Instead, as shown in the diagram below, a method called **Quotienting** is used with the following features:

* The verifier should be able to compute $$\mathsf{Quotient}(x)$$ from a sampled point x in the domain excluding $$S$$ in $$\mathbb{F}$$.
* Suppose that every codeword close to f disagrees from $$\hat{p}$$ (which is derived from $$p$$ and $$S$$). Then $$\mathsf{Quotient}(f, S, p)$$ is far from the code.

<figure><img src="../../.gitbook/assets/image (59).png" alt=""><figcaption><p>Fig 3. Description of univariate function quotienting</p></figcaption></figure>

3\. **Out Of Domain Sampling**: Multiple polynomials may satisfy the conditions above, so out of domain sampling narrows it down to the one unique polynomial. To do this, additional random points are added to the subdomain $$S$$ used in **Quotienting**, and values from substituting f at these points are added to $$p$$.

4\. **Degree Correction**: The degree of the Quotient polynomial becomes $$\deg(f) - |S|$$, which isn’t a power of 2 as required at the beginning of each round. Therefore, a technique is needed to adjust it to a power of 2, called **Degree Correction**. It maintains the following characteristics:

* The verifier should be able to compute $$\mathsf{DegCor}(f)$$ from $$f$$.
* The distance between $$f$$ and $$\mathsf{DegCor}(f)$$ should be preserved with high probability.

<figure><img src="../../.gitbook/assets/image (60).png" alt=""><figcaption><p>Fig 4. Description of degree correction</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (61).png" alt=""><figcaption><p>Fig 5. Comparison of FRI and STIR at different <span class="math">\rho</span>. Lower is better.</p></figcaption></figure>

The figure above illustrates the differences in argument size, verifier hash complexity, prover time, and verifier time between the FRI and STIR protocol at different rates. From the above graph, the following facts can be observed:

* The argument size and verifier hash complexity of STIR is smaller than that of FRI, and it decreases further as the degree increases.
* The prover time of STIR is slightly slower than that of FRI.
* When the degree is small, the verifier time of STIR is slower than FRI, but as the degree increases, it becomes faster.

(The argument size can be found at this [link](https://github.com/WizardOfMenlo/stir/blob/main/scripts/protocol.py), and it can be considered equivalent to the proof size.)

## Conclusion

In conclusion, these four techniques collectively reduce the rate, enabling a reduction in the number of queries. This approach is referred to as "Shift," resulting in the protocol name of STIR (Shift To Improve Rate). Recently, the same authors of the STIR paper presented a new technique called [WHIR](https://eprint.iacr.org/2024/1586.pdf), which was introduced at zkSummit 12, so interested readers may consider watching the presentation. I’ve skipped the explanation of how STIR achieves $$O(\log d + \lambda \log \log d)$$. For those interested in the details, please refer to sections C.1 and C.2 of the paper.

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
>
> Reviewed by [Giacomo Fenzi](https://gfenzi.io/) from [EPFL](https://www.epfl.ch/)&#x20;
