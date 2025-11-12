# DEEP FRI

## Brief recap of FRI

In FRI, our starting point is an input function $$f^{(0)} : \mathcal{L}^{(0)} \rightarrow \mathbb{F}$$ where $$\mathbb{F}$$ is a ﬁnite ﬁeld, $$\mathcal{L}^{(0)} \subset \mathbb{F}$$ is the evaluation domain. The FRI protocol is a two-phase protocol (the two phases are called COMMIT and QUERY) that convinces a veriﬁer that $$f^{(0)}$$ is close to the Reed-Solomon code $$\mathsf{RS}[\mathbb{F}, \mathcal{L}^{(0)}, \rho]$$.

### COMMIT phase:

1. For $$i = 0$$ to $$r - 1$$:
   1. The veriﬁer picks uniformly random $$\beta_i \in \mathbb{F}$$ and sends it to the prover.
   2. The prover sends a merkle proof of a function $$f^{(i+1)} : \mathcal{L}^{(i+1)} \rightarrow \mathbb{F}$$. (In the case of an honest prover, $$f^{(i+1)} = f^{(i)}_{\mathsf{even}} + \beta_if^{(i)}_{\mathsf{odd}}$$).
2. The prover sends a value $$C \in \mathbb{F}_q$$. (In the case of an honest prover, $$f^{(r)}$$ is the constant function with value = $$C$$).

### QUERY phase: (executed by the Veriﬁer)

1. Repeat $$l$$ times:
   1. Pick $$x_0 \in \mathcal{L}^{(0)}$$ uniformly at random.
   2. For $$i = 0$$ to $$r - 1$$:
      1. Deﬁne $$x_{i+1} \in \mathcal{L}^{(i+1)}$$ by $$x_{i+1} = x_i^2$$.
      2. Query $$f^{(i)}$$ at $$x_i$$ and $$-x_i$$ to compute $$f^{(i)}_{\mathsf{even}}(x_{i+1})$$ and $$f^{(i)}_{\mathsf{odd}}(x_{i+1})$$.
      3. Query $$f^{(i+1)}(x_{i+1})$$ and if $$f^{(i)}_{\mathsf{even}}(x_{i+1}) + \beta_i f^{(i)}_{\mathsf{odd}}(x_{i+1})\neq f^{(i+1)}(x_{i+1})$$, then REJECT.
2. If $$f^{(r)}(x_r) \neq C$$, then REJECT.
3. ACCEPT

### Properties

For $$V=\mathsf{RS}[\mathbb{F}, D, \rho]$$ where $$D$$ is an FFT-friendly domain (subgroup of size $$n=2^k$$),[ BBHR17](https://eccc.weizmann.ac.il/report/2017/134) showed that FRI satisfies the following:

* Prover time $$\leq 6n$$, proof length $$\leq \frac{n}{3}$$
* Verifier time $$\leq 21 \log n$$, query complexity $$\leq 2 \log n$$
* Round complexity = $$\frac{\log n}{2}$$
* Soundness: Rejection probability of $$\delta$$-far words $$\approx\delta-o(1)$$ (which basically means it is a bit smaller but that is ignorable), for sufficiently small $$\delta < \delta_0$$
  * A word $$w$$ is said to be $$\delta$$-far from the code $$\mathcal{C}$$ if its Hamming distance from every codeword in $$\mathcal{C}$$ is at least $$\delta n$$)
  * $$\delta_0$$ is a positive constant that depends mainly on the code rate.
* In other words: Probability of rejecting $$\delta$$-far word is approximately $$\min(\delta, \delta_0)$$.

### Understanding the Soundness

The soundness of a proximity testing protocol can be described by a soundness function $$s(\cdot)$$ that takes as input a proximity parameter $$\delta$$, and outputs the minimum rejection probability of the verifier, where this minimum is taken over all words that are $$\delta$$-far from the code.

$$
s(\delta)= \min\{\forall w \text{ s.t }\Delta(w, \mathcal{C})\geq \delta: \mathsf{Pr}_{rej}(w)\}
$$

where $$Pr_{rej}(w)$$ is the rejection probability of the word $$w$$.

$$
\min(\delta_0, \delta) - o(1) \leq s(\delta)\leq \delta
$$

In the case of FRI soundness for a single test, an upper bound $$s(\delta) \leq \delta$$ is easy to establish but the more important part is establishing a lower bound. The initial analysis in[ BBHR18b](https://drops.dagstuhl.de/storage/00lipics/lipics-vol107-icalp2018/LIPIcs.ICALP.2018.14/LIPIcs.ICALP.2018.14.pdf) showed a nearly matching lower bound for sufficiently small values of $$\delta$$. In particular, the bound obtained there gives $$s(\delta) \geq \min\{\delta,\delta_0\} - o(1)$$ where $$\delta_0 \approx \frac{1-3 \rho}{4}$$.

For codes of $$ho \geq 1/3$$, $$\delta_0$$ becomes non-positive so the FRI protocol becomes meaningless. Even when considering the ideal case of $$ho \rightarrow 0\Rightarrow \delta_0 \rightarrow 1/4$$. This rather low soundness means that many tests must be applied in order to reach a target soundness error. For example, to achieve a target soundness error of $$2^{-\lambda}$$, then how many tests do we need?

$$
\text{min rejection probability } s(\delta)\geq 1/4 \Rightarrow \text{max acceptance probability after }l \text{ queries } \leq(1-\frac{1}{4})^l
$$

$$
(1-\frac{1}{4})^l \leq 2^{-\lambda}\Rightarrow l\cdot log_2\frac{3}{4}\leq-\lambda\Rightarrow l \geq-\frac{\lambda}{\log_2{\frac{3}{4}}}\approx2.4\lambda
$$

This shows that for 128-bit security, we need more than 300 queries. So to reduce this number, we need to have a better soundness lower bound.

## Improvements in the soundness lower bound

Following the limitations of FRI, there were some works that improved upon the $$\delta_0$$.

1. Original FRI paper \[[BBHR17](https://eccc.weizmann.ac.il/report/2017/134/)]:

$$
\delta_0 \approx \frac{1-3\rho}{4} \Rightarrow l \geq 2.4\lambda
$$

2. Worst-to-average case reduction \[[BKS18](https://doi.org/10.4230/)]:

$$
\delta_0 \approx 1-\sqrt[4]{\rho}\Rightarrow l\geq\frac{4\lambda}{\log\frac{1}{\rho}}
$$

3. Improved bound from Lemma 3.1 of \[BGKS20]:

$$
\delta_0\approx 1-\sqrt[3]{\rho}\Rightarrow l\geq \frac{3\lambda}{\log\frac{1}{\rho}}
$$

From the second result, we can see that when $$ho \rightarrow 0$$, $$\delta_0 \rightarrow 1$$ which is a big improvement for the soundness of FRI. Building on top of that, Lemma 3.1 improved the number of tests required for $$\lambda$$ bit security from $$\frac{4\lambda}{\log\frac{1}{\rho}}$$ to $$\frac{3\lambda}{\log\frac{1}{\rho}}$$ which is \~25% less.

In addition, Lemma 3.1 is shown to be tight under a special case:

1. $$\mathbb{F}$$ is a binary field
2. $$ho = 1/8$$ precisely
3. Evaluation domain $$D$$ equals all of $$\mathbb{F}$$

Therefore, we needed something different to break this soundness barrier and the DEEP (Domain Extension for Eliminating Pretenders) method was introduced.

## DEEP FRI

First, what do we mean by “pretenders”? Let's say a malicious prover has a very large degree polynomial that agrees on all of $$\mathcal{L}^{(0)}$$ with a low-degree polynomial. Then how does the verifier differentiate between such pretenders and an actual low-degree polynomial? If we sample $$x_0\in \mathcal{L}^{(0)}$$, we won't find any discrepancies. Therefore, we attempt to sample outside the evaluation domain using quotient functions.

### Quotienting

If $$f^{(0)}$$ is indeed evaluations of low-degree polynomial $$P(X)$$ on $$D$$ then verifier should be able to query $$P$$ on an arbitrary $$z\in\mathbb{F}$$. Suppose the prover claims $$P(z)=a$$, then $$P(X) - a$$ is divisible by $$X-z$$ so let's denote it as:

$$
\mathsf{QUOTIENT}(P, z, a) = \frac{P(X) - a}{X-z}
$$

### COMMIT phase:

1. For $$i = 0$$ to $$r - 1$$:
   1. The verifier picks uniformly random $$\beta_i, z_i \in \mathbb{F}$$.
   2. The prover sends $$f^{(i)}_{even}(z_i) + \beta_i f^{(i)}_{odd}(z_i)=a$$.
   3. The prover sends a merkle proof of a function $$f^{(i+1)} : L^{(i+1)} \rightarrow \mathbb{F}$$. (In the case of an honest prover, $$f^{(i+1)}(X)=\mathsf{QUOTIENT}(f^{(i)}_{\mathsf{even}}(X) + \beta_i f^{(i)}_{\mathsf{odd}}(X), z_i, a))$$.
2. The prover sends a value $$C \in \mathbb{F}_q$$. (In the case of an honest prover, $$f^{(r)}$$ is the constant function with value = $$C$$).

### QUERY phase: (executed by the Veriﬁer)

1. Repeat $$l$$ times:
   1. Pick $$x_0 \in \mathcal{L}^{(0)}$$ uniformly at random.
   2. For $$i = 0$$ to $$r - 1$$:
      1. Deﬁne $$x_{i+1} \in \mathcal{L}^{(i+1)}$$ by $$x_{i+1} = x_i^2$$.
      2. Query $$f^{(i)}$$ at $$x_i$$ and $$-x_i$$ to compute $$f^{(i)}_{\mathsf{odd}}(x_{i+1})$$ and $$f^{(i)}_{\mathsf{even}}(x_{i+1})$$.
      3. If $$\frac{(f^{(i)}_{\mathsf{even}}(x_{i+1}) + \beta_i f^{(i)}_{\mathsf{odd}}(x_{i+1})) - a}{(x_{i+1} - z_i)}\neq f^{(i+1)}(x_{i+1})$$, then REJECT.
2. If $$f^{(r)}(x_r) \neq C$$, then REJECT.
3. ACCEPT

### Resulting soundness

As a result of applying the DEEP technique, the soundness claim was improved to:

$$
\delta_0 \approx 1-\sqrt{\rho}\Rightarrow l\geq \frac{2\lambda}{\log\frac{1}{\rho}}
$$

## Final Remarks

The special case for tightness of the lower bound in Lemma 3.1 is something we can ignore. If we assume that the length of the code $$n$$ is much smaller than the size of the field $$q$$ (which is the case for practical STARK implementations), this bound can be improved significantly. Investigating this further, the authors of[ BCIKS2023](https://dl.acm.org/doi/10.1145/3614423) figured that when working with RS code over a sufficiently large field—polynomially large in the code’s blocklength—and when $$\delta \in (0, 1 - \sqrt{\rho})$$, the average-case distance is almost the same as the worst-case distance. With this, they claim that FRI can achieve the same soundness as DEEP FRI without its added prover complexities. However, the out-of-domain sampling method introduced in this paper is still used widely to improve the soundness of the underlying protocol (for example STIR).

## References:

| BBHR17    | Eli Ben-Sasson, Iddo Bentov, Ynon Horesh, and Michael Riabzev. 2018. Fast Reed-Solomon interactive oracle proofs of proximity. In Proceedings of the 45th International Colloquium on Automata, Languages, and Programming (ICALP 2018). Retrieved from[ https://eccc.weizmann.ac.il/report/2017/134](https://eccc.weizmann.ac.il/report/2017/134)                                                                                                                                                          |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| BBHR18b   | <p>Eli Ben-Sasson, Iddo Bentov, Ynon Horesh, and Michael Riabzev. Fast Reed-Solomon Interactive Oracle Proofs of Proximity. In Proceedings of the 45th International Colloquium on Automata, Languages, and Programming (ICALP 2018). Retreived from<br><a href="https://drops.dagstuhl.de/storage/00lipics/lipics-vol107-icalp2018/LIPIcs.ICALP.2018.14/LIPIcs.ICALP.2018.14.pdf">https://drops.dagstuhl.de/storage/00lipics/lipics-vol107-icalp2018/LIPIcs.ICALP.2018.14/LIPIcs.ICALP.2018.14.pdf</a></p> |
| BKS18     | Eli Ben-Sasson, Swastik Kopparty, and Shubhangi Saraf. 2018. Worst-case to average case reductions for the distance to a code. In Proceedings of the 33rd Computational Complexity Conference. 24:1–24:23. DOI:[https://doi.org/10.4230/](https://doi.org/10.4230/) LIPIcs.CCC.2018.24                                                                                                                                                                                                                      |
| BGKS20    | Eli Ben-Sasson, Lior Goldberg, Swastik Kopparty, and Shubhangi Saraf. 2020. DEEP-FRI: Sampling outside the box improves soundness. In Proceedings of the 11th Innovations in Theoretical Computer Science Conference (LIPIcs), Thomas Vidick (Ed.), Vol. 151. Schloss Dagstuhl - Leibniz-Zentrum für Informatik, 5:1–5:32. DOI:[https://doi.org/10.4230/LIPIcs](https://doi.org/10.4230/LIPIcs). ITCS.2020.5                                                                                                |
| BCIKS2023 | Eli Ben-Sasson, Dan Carmon, Yuval Ishai, Swastik Kopparty, and Shubhangi Saraf. 2023. Proximity Gaps for Reed–Solomon Codes. J. ACM 70, 5, Article 31 (October 2023), 57 pages.[ https://doi.org/10.1145/3614423](https://doi.org/10.1145/3614423)                                                                                                                                                                                                                                                          |

> Written by [Batzorig Zorigoo](https://app.gitbook.com/u/cO1lUla01ZW0seepO37jMHFTxg42 "mention") of Fractalyze
