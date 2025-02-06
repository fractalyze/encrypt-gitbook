# FRI Security Features and Optimizations

## Extension Fields

Soundness error in a cryptographic proof system is the probability that an incorrect proof (or computation) is mistakenly accepted as correct. This error is typically kept small by increasing the field size relative to the evaluation domain size in accordance with the [Schwartz–Zippel lemma](https://en.wikipedia.org/wiki/Schwartz%E2%80%93Zippel_lemma). In addition to choosing a field large enough in relation to the evaluation domain size, extension fields can be used to increase the field size even further.

Let’s see how using an extension field of 4 (Baby Bear 4) can affect the security bits of using the Baby Bear field. We use a Baby Bear evaluation domain size of $$2^{27}$$ knowing that the total field size is $$2^{31}$$ in order to allow for extensive operations to be done on the evaluations. Note that this is an arbitrary number, but is used in Baby Bear implementations in code by projects such as [Plonky3](https://github.com/Plonky3/Plonky3/blob/main/baby-bear/src/baby_bear.rs).

<figure><img src="../../../.gitbook/assets/image (29).png" alt=""><figcaption><p>Figure 8. Calculations of security with Baby Bear and Baby Bear 4</p></figcaption></figure>

Thus, as shown, the security level increases tremendously from 4 security bits to 97 security bits with the introduction of extension fields.

## Proof of Work Witness

Introducing a variable called the “proof of work witness” can increase the cost of being a malicious prover and fill the gap between the target security level and actual security level achieved by FRI. Adding a proof of work witness involves using a “…64-bit nonce that when hashed together with the state of the hash chain, results in a required number of leading zeros…” that follows all the commitments made by the prover ([StarkWare](https://eprint.iacr.org/2021/582.pdf)). An honest prover only needs to run a grind process once to discover this nonce, but a dishonest prover has to grind again for every changed commitment, requiring more work.

Let’s say we want a target security level of 112 bits and we utilize same Baby Bear 4 extension field from before:

<figure><img src="../../../.gitbook/assets/image (30).png" alt=""><figcaption><p>Figure 9. Computation of total security of Baby Bear 4 with proof of work witness</p></figcaption></figure>

Thus, a target security level of 112 bits on Baby Bear 4 could be achieved by checking the leading 15 bits of the proof of work witness hash.

## Number of Queries

The number of queries is another factor that affects the security of the FRI protocol. Imagine the prover sets 3 evaluations out of four total evaluations as faulty values. The prover would then have a $$\frac{1}{4}$$ chance that the verifier randomly selects a value and accepts the prover’s claims if they query once, or a $$\frac{1}{16}$$ chance if they query twice. As such, a higher number of queries increases the soundness of the protocol at the cost of more work being done by the verifier; however, remember that the goal of IOPPs is to reduce the number of queries done. As such, there is an incentive to reduce the number of queries, while still preserving the security level.

## Blow-up Factor

The blow-up factor helps us increase the evaluation domain size. Since a low degree polynomial would have a degree much smaller than the total field size, codewords that are low-degree extensions, which are evaluations of low-degree polynomials on a separate, larger domain, are used instead. Creating these low degree extensions can allow the determination of a faulty low degree polynomial to become easier, as there will now be a larger number of erroneous values, thus allowing for fewer queries needed on the larger domain. This security factor is closely tied to the concept of “rate” in Reed-solomon codes, in which a larger evaluation domain defines a higher rate, which ensures better security. Refer to [here](https://en.wikipedia.org/wiki/Code_rate) for more information on rate.

<figure><img src="../../../.gitbook/assets/image (31).png" alt=""><figcaption><p>Figure 10: Prover time, verifier time, and proof size in relation to varying blowup factors (Courtesy of <a href="https://eprint.iacr.org/2021/582.pdf">StarkWare</a>)</p></figcaption></figure>

However, as [LambdaClass](https://blog.lambdaclass.com/how-to-code-fri-from-scratch/) puts it, “There is a tradeoff between the blowup factor used (which increases the prover’s work and memory use) and the number of queries (which increases the proof size and verifier’s work)” as displayed by Starkware’s graph above.

## Reduced Rounds

Although our previous explanation and example showed that the FRI protocol ended when the polynomial was folded to a constant value, when using a non-zero blowup factor, the protocol can instead be ended on a polynomial and not a single constant evaluation. Converting low degree polynomials to low degree extensions with the blowup factor maintains the degree of the original polynomial, but adds more total evaluations. With an increase in the number of evaluations, it comes to reason that the number of folding rounds appears like it should increase in order to reach a single constant evaluation; however, the number of rounds can continue be constrained to the log of the original maximum degree, since the new final polynomial’s value at zero would evaluate as the constant value instead. As such, the security is still preserved, regardless of the reduced number of rounds in comparison to the degree plus the blowup factor.

## Bit Reversal Storage

As mentioned before, in the query phase, once the verifier sends a challenge $$x$$, the prover must return their evaluations $$f(x)$$ and $$f(-x)$$ for every level. While this would usually require two Merkle tree proofs, one for each evaluation, bit reversal storage allows the prover to send only one Merkle tree proof. This is done by storing the evaluations of $$f(x)$$ and $$f(-x)$$ inside the same leaf of a Merkle tree. Let’s take a look at storing values with bit reversal more closely:

<figure><img src="../../../.gitbook/assets/image (32).png" alt=""><figcaption><p>Figure 11. The process of bit-reversal</p></figcaption></figure>

We know that if we have $$f(\omega^i)$$, then $$f(-\omega^i)=f(\omega^{i+(d/2)})$$, where i is the root of unity index and $$d$$ is the domain size, since $$-1=\omega^{d/2}$$. Therefore, in this case with our domain size of 8, each index $$i$$ is paired up with their respective index $$i+4$$. Looking at our bit-reversed indexes on the table, we can now see that the needed pairs of indexes are right next to each other, allowing for the efficient storage and calculations.

## Combining Polynomials — Batching FRI

What if we wanted to run the fri protocol on multiple polynomials? The naive approach would be to undergo the process individually with each polynomial, leading to expensive runtimes and large proof sizes. Instead, we can use “Batch FRI,” or employ a linear combination of their evaluations to combine all polynomials together and only require one run of the FRI protocol.

Linear combinations require a given challenge or a random factor. Let’s call it $$\alpha$$. In the following diagram, we can see how the four evaluations of four polynomials on the root of unity $$\omega^0$$ could be combined together into one value.

<figure><img src="../../../.gitbook/assets/image (33).png" alt=""><figcaption><p>Figure 12. Combining evaluations across different polynomials</p></figcaption></figure>

In batch FRI, this method of linear combination would be done for all evaluations of all polynomials at every step to compress the data and reduce the required space.

## **Two Adic FRI PCS** <a href="#id-556b" id="id-556b"></a>

While many implementations of the FRI protocol are run with [2-adicity](https://www.cryptologie.net/article/559/whats-two-adicity/), the coded implementations of the FRI protocol with all the aforementioned security features and optimizations are labeled “Two Adic FRI PCS” or “Two Adic FRI” for short. This version of FRI is implemented in Rust by [Plonky3](https://github.com/Plonky3/Plonky3/blob/main/fri/src/two_adic_pcs.rs) and SP1 and in C++ by [Kroma](https://github.com/kroma-network/tachyon/blob/main/tachyon/crypto/commitments/fri/two_adic_fri.h).

A couple of changes are added to “Two Adic FRI” on top of the process of the naive FRI scheme to conform to the security requirements.

### Before the Commit Phase <a href="#e4f6" id="e4f6"></a>

In “Two Adic FRI,” the coset low-degree extensions of the original polynomial’s evaluations are first committed. The driving factor behind committing these coset low-degree extensions is to allow the use of the blowup factor and increase security.

### Running FRI <a href="#d115" id="d115"></a>

The steps of the FRI Protocol as explained earlier now run, but on the quotient polynomial, defined in the following equation. Given a polynomial $$p(x)$$ of degree $$d$$, this quotient polynomial should be guaranteed to be a polynomial of degree $$d-1$$ as long as $$p(z)$$ equals $$v$$.

$$
\frac{p(x)-v}{x-z}
$$

The commitments outlined in the previous section are evaluations on a coset domain, yet the rest of the protocol runs on the domain of an extension field. As such, the quotient polynomial must be used in order to sample $$p(x)$$ in the domain of the extension field which is outside of the original coset domain. Furthermore, we are ensured that if the quotient polynomial is not of low degree, the original polynomial is also not of low degree.

Besides these differences along with the introduction of other features aforementioned such as the proof of work witness and bit reversal storage, Two Adic FRI runs an almost identical folding process to naive FRI.

> Written by [Ashley Jeong](https://app.gitbook.com/u/wyEMbFN1Kybygv7v1pKpc2P8q6D2 "mention") from [A41](https://www.a41.io/)
