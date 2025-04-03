# Overview

## &#x6E;**-Party MPC**

$$
\mathsf{MPC}_n(f, w_1,  \dots, w_n, x) \rightarrow f(w_1, \dots, w_n, x)
$$

**n-Party MPC (Multi-Party Computation)** is a cryptographic protocol that allows $$n$$ **parties to jointly compute a function** $$f(w_1, \dots, w_n, x)$$ over their respective private inputs **without revealing them to each other**.

In other words:

* For $$1 \le i \le n$$, **each party** $$p_i$$ holds a secret input $$w_i$$
* A shared input $$x$$ is known to every party
* They compute $$f(w_1, \dots, w_n, x)$$ **without disclosing their inputs**

This approach is designed for **privacy-preserving computation in security-sensitive environments**, enabling collaboration without compromising data confidentiality.

Of course, you can also use notation for the $$f$$ without shared input as follows:

$$
\mathsf{MPC}_n(f, w_1,  \dots, w_n) \rightarrow f(w_1, \dots, w_n)
$$
