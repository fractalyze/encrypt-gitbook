# 2-Chain and 2-Cycle of Elliptic Curves

## 2-Chain of Elliptic Curves

#### Definition 1.&#x20;

A **2-chain** of elliptic curves is a list of two distinct curves $$E_1 / \mathbb{F}_{p_1}$$ and $$E_2/ \mathbb{F}_{p_2}$$ where $$p_1$$ divides $$\#E_2(\mathbb{F}_{p_2})$$. The first curve is denoted the **inner curve**, while the second curve whose order is the characteristic of the inner curve, is denoted the **outer curve**.&#x20;

<figure><img src="../../../.gitbook/assets/image (151).png" alt=""><figcaption></figcaption></figure>

SNARK-friendly 2-chains are composed of two curves that have subgroups of orders $$r_1$$ and $$r_2$$ where $$r_1 | \#E_1(\mathbb{F}_{p_1})$$,  $$r_2 | \#E_2(\mathbb{F}_{p_2})$$ and $$r_1 \equiv r_2 \equiv 1 \mod 2^L$$ for a large integer $$L$$ (2-adicity). An example of a 2-chain for SNARK is composed of the inner curve BLS12-377 and the outer curve BW6-761.



## 2-Cycle of Elliptic Curves

#### Definition 2.

A **2-cycle** of elliptic curves is a list of two distinct prime-order curves $$E_1 / \mathbb{F}_{p_1}$$and $$E_2/ \mathbb{F}_{p_2}$$ where $$p_1$$ and $$p_2$$ are large primes where $$p_1 = \#E_2(\mathbb{F}_{p_2})$$ and $$p_2 = \#E_1(\mathbb{F}_{p_1})$$. Similarly to the definition of 2-chain, by SNARK-friendly 2-cycles we mean the cycles composed of two curves whose orders satisfy the following : $$\#E_1(\mathbb{F}_{p_1}) \equiv \#E_2(\mathbb{F}_{p_2}) \equiv 1 \mod 2^L$$ for a large integer $$L$$ (2-adicity).&#x20;

A 2-cycle is a 2-chain where **both curves are inner and outer curves with respect to each other.**&#x20;

<figure><img src="../../../.gitbook/assets/image (152).png" alt=""><figcaption></figcaption></figure>



**Recursive SNARKs** benefit significantly from the structure of a 2-cycle of elliptic curves. In recursive constructions, a proof generated over one elliptic curve needs to be verified inside a SNARK circuit that runs over another curve. A 2-cycle enables this by ensuring that each curve’s scalar field matches the group order of the other, allowing seamless recursion across curves. This mutual compatibility is essential for recursive proof composition and is used in systems like Halo2 and Nova.

***

## Reference

* [https://eprint.iacr.org/2022/1400.pdf#page=6](https://eprint.iacr.org/2022/1400.pdf#page=6)

> Written by [Carson Lee](https://app.gitbook.com/u/Hm5RHrPlu2fbxwXnpUgxmvXVTwB3 "mention") from [A41](https://www.a41.io/)
