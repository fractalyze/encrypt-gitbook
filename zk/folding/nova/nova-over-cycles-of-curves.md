# Nova over Cycles of Curves

## Cycles of curves

Actual implementation of Nova uses EC group $$\mathbb{G}_p=E(\mathbb{F_q})$$ over a field $$\mathbb{F}_q\neq \mathbb{F}_p$$ defined as:

$$
\mathbb{G}_p=\{(x,y)\in \mathbb{F}_q| E: y^2=x^3 + 5\}
$$

so each element in $$\mathbb{G}_p$$ consists of 2 elements from base field $$\mathbb{F}_q$$. Therefore, constraints in $$\mathbb{F}_p$$ will translate into $$\mathbb{F}_q$$ operations (non-native) which turns out to be quite expensive. So instead, we try to express elliptic curve group operations over the base field only which turns out to be quite efficient. But to incrementally prove and verify, we need something called a **cycle of curves**. In a cycle of curves, there are 2 elliptic curve groups that are base fields of the other. For example,

$$
\mathbb{G}^{(1)}:=\{(x,y)\in \mathbb{F}^{(2)}|\ y^2=x^3+5\}
$$

$$
\mathbb{G}^{(2)}:=\{(x,y)\in \mathbb{F}^{(1)}|\ y^2=x^3+5\}
$$

where $$\mathbb{F}^{(1)}$$ and $$\mathbb{F}^{(2)}$$ are different fields. [Pasta curves](https://o1-labs.github.io/proof-systems/specs/pasta.html) are one example of a cycle of curves.

## Efficient folding over cycles of curves

Let's define functions $$\mathsf{Fold_V}(\mathbb{U_1},\mathbb{U_2}), \mathsf{Fold_P}((\mathbb{U_1},\mathbb{W_1}),(\mathbb{U}_2,\mathbb{W}_2))$$ as the verifier and prover parts of folding committed relaxed R1CS, respectively. Suppose you have a $$\mathsf{R1CS}^{(1)}$$ instance over a certain **scalar field** **of** $$\mathbb{G}^{(1)}$$ which is $$\mathbb{F}^{(1)}$$. To check a folded instance from $$\mathsf{R1CS}^{(1)}$$ we need to check if the folded instance is the result of correct execution $$\mathsf{Fold_V}$$. This constraint can be only be implemented efficiently **over the base field of** $$\mathbb{G}^{(1)}$$ which is $$\mathbb{F}^{(2)}$$.&#x20;

For example, with four $$\mathsf{R1CS}^{(1)}$$ instances, the folding process would occur in two steps:

1. First fold: Fold two pairs of instances, resulting in two $$\mathsf{R1CS}^{(1)}$$ instances but to verify if each of them are constructed correctly, we will need two $$\mathsf{R1CS}^{(2)}$$ instances.
2. Second fold: Fold the $$\mathsf{R1CS}^{(2)}$$ instances into a single $$\mathsf{R1CS}^{(2)}$$ instance but to verify if it is constructed correctly, we will need one $$\mathsf{R1CS}^{(1)}$$ instance.&#x20;

### IVC Construction

<figure><img src="../../../.gitbook/assets/image (83).png" alt=""><figcaption><p>Figure 1: Folding iteration over cycles of curves. Credit: <a href="https://youtu.be/h_PU7FZWiQk?t=583">https://youtu.be/h_PU7FZWiQk?t=583</a></p></figcaption></figure>

To construct the folding described above, the original Nova implementation uses:

1. $$\mathsf{R1CS}^{(1)}$$ to do $$i$$-th step in $$\mathbb{F}^{(1)}$$ and accumulate the instances of $$\mathsf{R1CS}^{(2)}$$ (denoted as $$\mathbb{U}^{(2)}_{acc,i+1}$$).
2. $$\mathsf{R1CS}^{(2)}$$ to do $$i$$-th step in $$\mathbb{F}^{(2)}$$ and accumulate the instances of $$\mathsf{R1CS}^{(1)}$$ (denoted as $$\mathbb{U}^{(1)}_{acc,i+1}$$ ).

So if we denote the step function in $$\mathbb{F}^{(1)}$$ as $$\mathsf{F}_1$$ and $$\mathbb{F}^{(2)}$$ as $$\mathsf{F}_2$$. The high level overview of the IVC will be as shown below:

<figure><img src="../../../.gitbook/assets/image (98).png" alt=""><figcaption><p>Figure 2: High level overview of the IVC over cycles of curves. Credit: <a href="https://www.youtube.com/watch?v=l-F5ykQQ4qw">https://www.youtube.com/watch?v=l-F5ykQQ4qw</a></p></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (99).png" alt=""><figcaption><p>Figure 3: Deeper look at each step of the IVC over cycle of curves. Credit: <a href="https://www.youtube.com/watch?v=l-F5ykQQ4qw">https://www.youtube.com/watch?v=l-F5ykQQ4qw</a></p></figcaption></figure>

This is the high-level view of $$\mathsf{R1CS}^{(1)}$$ but $$\mathsf{R1CS}^{(2)}$$ would also look the same:

<figure><img src="../../../.gitbook/assets/image (101).png" alt=""><figcaption><p>Figure 4: Structure of each R1CS. Credit: <a href="https://www.youtube.com/watch?v=l-F5ykQQ4qw">https://www.youtube.com/watch?v=l-F5ykQQ4qw</a></p></figcaption></figure>

To summarize what we need to do in $$\mathsf{R1CS}^{(1)}$$ is:

1. Check if the public instance $$x_0$$ of $$\mathbb{U}_i^{(2)}$$ is indeed as given.
2. Check $$\mathsf{IsStrict}(\mathbb{U}_i^{(2)})$$ ($$\mathsf{IsStrict}$$ checks if the error term is $$\bm{0}$$ and scalar is 1).
3. Calculate $$z^{(1)}_{i+1}$$ $$F^{(1)}$$ and accumulate $$\mathbb{U}_i^{(2)}$$.
4. Check if the input used in this step matches the output from previous step.
5. Check if the output element hash is done correctly.

**NOTE**: In $$\mathsf{R1CS}^{(2)}$$, it will fold $$\mathbb{U}_{i+1}^{(1)}$$ and $$\mathbb{U}_{acc,i}^{(1)}$$ (instead of instances of same index). Also, hash assignments will be a bit different in terms of indexing:

<figure><img src="../../../.gitbook/assets/Screenshot 2025-02-17 at 4.05.15 PM.png" alt=""><figcaption><p>Figure 5. Input output hashes of each instance.</p></figcaption></figure>

## VC Verification

<figure><img src="../../../.gitbook/assets/image (103).png" alt=""><figcaption><p>Figure 6: IVC Verification. Credit: <a href="https://www.youtube.com/watch?v=l-F5ykQQ4qw">https://www.youtube.com/watch?v=l-F5ykQQ4qw</a></p></figcaption></figure>

### Faulty version

Given a proof $$\pi = (\mathbb{U}_i^{(1)}, \mathbb{U}_i^{(2)}, \mathbb{U}_{acc,i}^{(1)}, \mathbb{U}_{acc,i}^{(2)})$$, $$\mathsf{Verify}(i, (z_0^{(1)}, z_0^{(2)}), (z_i^{(1)}, z_i^{(2)}),\pi)$$ checks the following:

1. Check if $$\mathbb{U}_i^{(1)}$$_,_ $$\mathbb{U}_i^{(2)}$$, $$\mathbb{U}_{acc,i}^{(1)}$$ and $$\mathbb{U}_{acc,i}^{(2)}$$ instances are satisfied.
2. Check if $$\mathbb{U}_i^{(1)}.{\mathsf{x}_1}=\mathsf{hash}(i^{(1)}, z_0^{(1)}, z_i^{(1)}, \mathbb{U}_{acc,i}^{(2)})$$. This ensures that this output hash is derived from inputs $$z_i^{(1)}, \mathbb{U}_{acc,i}^{(2)}.$$&#x20;
3. Check if $$\mathbb{U}_i^{(2)}.{\mathsf{x}_1}=\mathsf{hash}(i^{(2)}, z_0^{(2)}, z_i^{(2)}, \mathbb{U}_{acc,i}^{(1)})$$.

In this version, Wilson Nguyen et al. showed that it is possible to generate proof of evaluation of $$2^{75}$$ rounds of the Minroot VDF in only 1.46 seconds and proposed a fix which has been reflected to Nova implementation.

### Attack overview

This attack mainly stems from the fact that there is no constraints on the $$\mathsf{x}_0$$. Notice that the verification in the faulty version can be split into 2 parts:

1. Check if $$\mathbb{U}_i^{(2)}$$ and $$\mathbb{U}_{acc,i}^{(1)}$$ instances are satisfied.
2. Check $$\mathsf{IsStrict}(\mathbb{U}_i^{(2)})$$
3. Check if $$\mathbb{U}_i^{(2)}.{\mathsf{x}_1}=\mathsf{hash}(i^{(2)}, z_0^{(2)}, z_i^{(2)}, \mathbb{U}_{acc,i}^{(1)})$$.

And the other part would be:

* Check if $$\mathbb{U}_i^{(1)}$$ and $$\$$$$\mathbb{U}_{acc,i}^{(2)}$$ instances are satisfied.
* Check $$\mathsf{IsStrict}(\mathbb{U}_i^{(1)})$$
* Check if $$\mathbb{U}_i^{(1)}.{\mathsf{x}_1}=\mathsf{hash}(i^{(1)}, z_0^{(1)}, z_i^{(1)}, \mathbb{U}_{acc,i}^{(2)})$$.

So they can be from completely 2 different runs and this check will still pass. An actual attack is implemented as follows (Instances highlighted <mark style="color:red;">red</mark> are not satisfied malicious instances and <mark style="color:green;">green</mark> ones are satisfied instances):

**Step 1**: Generate malicious instance $$\textcolor{red}{\mathbb{U}^{(2)}_{i-1}}=(\bm{0},\bm{0},(\mathsf{x}_0,\mathsf{x}_1),1)$$ where $$\mathsf{x}_0=\mathsf{hash}(\cdot,\textcolor{red}{\mathbb{U}_{acc,i-1}^{(2)}})$$ is a trash value and $$\mathsf{x}_1=\mathsf{hash}((i-1)^{(2)},z^{(2)}_0,z^{(2)}_{i-1},\textcolor{green}{\mathbb{U}_\perp^{(1)}})$$.

**Step 2**: Run the honest prover to get $$\textcolor{green}{\mathbb{U}^{(1)}_i}$$ and $$\textcolor{green}{\mathbb{U}^{(2)}_{acc,i}}$$.

**Step 3:** Do step 1 and 2 for the other field to get $$(\textcolor{green}{\mathbb{U}^{(2)}_i},\textcolor{green}{\mathbb{U}^{(1)}_{acc,i}})$$.

It is very well explained in the presentation by Wilson Nguyen [here](https://www.youtube.com/watch?v=l-F5ykQQ4qw\&t=1837s). It is highly recommended to watch it if you want to fully understand how it works. To elaborate a bit more on this video:

* The honest prover will run $$\mathsf{R1CS}^{(1)}$$ with $$\textcolor{red}{\mathbb{U}_{i-1}^{(2)}}$$ and some fake accumulation claim $$\textcolor{red}{\mathbb{U}_{acc,i-1}^{(2)}}$$, and some arbitrary $$z_{i-1}^{(1)}$$.&#x20;
* This will give us:
  * $$\textcolor{red}{\mathbb{U}^{(2)}_{acc,i}}:=\mathsf{Fold_V}(\textcolor{red}{\mathbb{U}_{i-1}^{(2)}},\textcolor{red}{\mathbb{U}_{acc,i-1}^{(2)}})$$
  * $$z_i^{(1)}$$
  * $$\textcolor{green}{\mathbb{U}_i^{(1)}}$$ with $$\textcolor{green}{\mathbb{U}_{i}^{(1)}}.\mathsf{x_0}=\textcolor{red}{\mathbb{U}_{i-1}^{(2)}}.\mathsf{x}_1=\mathsf{hash}((i-1)^{(2)},z^{(2)}_0,z^{(2)}_{i-1},\textcolor{green}{\mathbb{U}_\perp^{(1)}})$$.&#x20;
* This makes it possible to use $$\textcolor{green}{\mathbb{U}_\perp^{(1)}}$$ in $$\mathsf{R1CS}^{(2)}$$ and it will output:
  * $$\textcolor{green}{\mathbb{U}_{acc,i}^{(1)}}:=\mathsf{Fold_V}(\textcolor{green}{\mathbb{U}_i^{(1)}},\textcolor{green}{\mathbb{U}_{\perp}^{(1)}})$$
  * $$\textcolor{green}{\mathbb{U}_i^{(2)}}$$ with $$\textcolor{green}{\mathbb{U}_{i}^{(2)}}.\mathsf{x_0}=\textcolor{red}{\mathbb{U}_{i-1}^{(1)}}.\mathsf{x}_1=\mathsf{hash}(i^{(1)},z^{(1)}_0,z^{(1)}_{i},{\textcolor{red}{\mathbb{U}_{acc,i}^{(2)}}})$$.&#x20;
* With this, we have a valid $$\textcolor{green}{\mathbb{U}_i^{(2)}}$$ and $$\textcolor{green}{\mathbb{U}_{acc,i}^{(1)}}$$.
* The $$\textcolor{red}{\mathbb{U}^{(2)}_{acc,i}}$$ generated here is invalid so $$\textcolor{green}{\mathbb{U}_i^{(1)}}$$ cannot be used as well since the $$\textcolor{green}{\mathbb{U}_i^{(1)}}.\mathsf{x}_1$$ hash has $$\textcolor{red}{\mathbb{U}^{(2)}_{acc,i}}$$. But we will just discard them both and generate them in Step 3.

Now, the faulty verification check will:

1. Check if instances from step 2 and 3 are satisfied. Since they are a result of an honest prover, it passes without problem.
2. Check $$\mathbb{U}_i^{(1)}.{\mathsf{x}_1}=\mathsf{hash}(i^{(1)}, z_0^{(1)}, z_i^{(1)}, \mathbb{U}_{acc,i}^{(2)})$$ passes since this hash was also properly constructed by the honest prover.
3. Check $$\mathbb{U}_i^{(2)}.{\mathsf{x}_1}=\mathsf{hash}(i^{(2)}, z_0^{(2)}, z_i^{(2)}, \mathbb{U}_{acc,i}^{(1)})$$ passes as well similarly.

### Fixed version

Given a proof $$\pi = (\mathbb{U}_i^{(2)}, \mathbb{U}_{acc,i}^{(1)}, \mathbb{U}_{acc,i}^{(2)})$$, $$\mathsf{Verify}(i, (z_0^{(1)}, z_0^{(2)}), (z_i^{(1)}, z_i^{(2)}),\pi)$$ checks the following:

1. Check if $$\mathbb{U}_i^{(2)}$$, $$\mathbb{U}_{acc,i}^{(1)}$$ and $$\mathbb{U}_{acc,i}^{(2)}$$ instances are valid.
2. Check if $$\mathbb{U}_i^{(2)}.{\mathsf{x}_1}=\mathsf{hash}(i^{(2)}, z_0^{(2)}, z_i^{(2)}, \mathbb{U}_{acc,i}^{(1)})$$.
3. Check if $$\mathbb{U}_i^{(2)}.{\mathsf{x}_0}=\mathsf{hash}(i^{(1)}, z_0^{(1)}, z_{i}^{(1)}, \mathbb{U}_{acc,i}^{(2)})$$.

Here, the previous attack passes the check 1 and 2 but for the last check, it becomes tricky since now we can't just discard the invalid instances and generate another one because $$\mathbb{U}_i^{(2)}.{\mathsf{x}_0}$$ contains $$\textcolor{red}{\mathbb{U}_{acc,i}^{(2)}}$$.

## References

* [Revisiting the Nova Proof System on a Cycle of Curves - Wilson Nguyen](https://www.youtube.com/watch?v=l-F5ykQQ4qw)

> Written by [Batzorig Zorigoo](https://app.gitbook.com/u/cO1lUla01ZW0seepO37jMHFTxg42 "mention") of Fractalyze
