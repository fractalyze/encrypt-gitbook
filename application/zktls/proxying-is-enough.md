---
description: 'Presentation: https://youtu.be/OL5rqvLkJfE'
---

# Proxying is enough

## Introduction

[DECO](deco.md) achieves selective data provenance from a TLS server by enforcing a server-client-verifier communication pattern. However, DECO introduces significant performance overhead. This raises the question: is it possible to achieve the same functionality by placing the verifier as a proxy between the client and server? Naturally, the participant acting as the proxy must not be able to learn any sensitive information. While DECO's strength lies in requiring no modifications to the server side, a proxy-based approach offers the additional benefit of not requiring any modifications on the client side either.

## Background

### AEAD (Authenticated Encryption with Associated Data)

**AEAD** is a widely used encryption method in modern cryptographic systems, offering two key security guarantees simultaneously:

1. **Confidentiality (Encryption)**: It encrypts the plaintext so that third parties cannot read its content.
2. **Integrity and Authentication**: It ensures that any tampering with the ciphertext or related information can be detected.

AEAD provides stronger security than simple encryption by also covering various types of metadata encountered in real-world scenarios. This is where the concept of **Associated Data (AD)** comes into play.

#### What is Associated Data (AD)?

**Associated Data** is data that is not encrypted but is still included in the authentication scope.

* AD remains in plaintext and is visible to anyone.
* However, during decryption, if the provided AD does not match the original AD used during encryption, the authentication fails.

> ✉️ Think of it like the address on an envelope — it’s visible to everyone, but **it must not be altered**.

#### Why is AD needed?

Traditional encryption schemes (e.g., AES-CBC) only encrypt the plaintext and leave other parts of the message (headers, identifiers, etc.) unprotected.

In real-world communication systems, the existence of unauthenticated metadata leads to several potential attacks:

<table><thead><tr><th width="169.7275390625">Attack Type</th><th>Description</th></tr></thead><tbody><tr><td><strong>Reordering</strong></td><td>Changing the order of messages to alter meaning or induce errors</td></tr><tr><td><strong>Replay</strong></td><td>Re-sending the same message repeatedly to confuse the system</td></tr><tr><td><strong>Re-targeting</strong></td><td>Forging a message to appear as if intended for a different recipient</td></tr></tbody></table>

→ These attacks arise because the **context surrounding the ciphertext** is not protected.

#### Effects of Including AD in AEAD

AEAD generates an authentication tag that includes both the ciphertext and the AD. This leads to the following outcomes:

* The plaintext is encrypted,
* The AD is authenticated but not encrypted,
* If the AD does not match during decryption, **the authentication fails and decryption is rejected.**

<table><thead><tr><th width="249.33203125">Effect</th><th>Description</th></tr></thead><tbody><tr><td>✅ Header forgery prevention</td><td>Protects metadata outside the encrypted portion</td></tr><tr><td>✅ Receiver verification</td><td>Allows messages to be accepted only by specific recipients</td></tr><tr><td>✅ Context binding</td><td>Cryptographically binds key, nonce, and AD into a secure “context”</td></tr></tbody></table>

#### AEAD Operations

An AEAD scheme defines the following interfaces:

* $$\mathsf{AEAD.Gen}(\tau) \rightarrow (k, n)$$: Generates a secret key $$k$$ and nonce $$n$$ from transcript $$\tau$$. We refer to the pair $$(k, n)$$ as the **context**.&#x20;
* $$\mathsf{AEAD.Enc}_k(n, a, m) \rightarrow (c, \sigma)$$: Encrypts plaintext $$m$$ using key $$k$$, nonce $$n$$, and associated data $$a$$, producing ciphertext $$c$$ and authentication tag $$\sigma$$.
* $$\mathsf{AEAD.Dec}_k(n, a, c, \sigma) \rightarrow m\ \text{or}\ \mathsf{FAILED}$$: Verifies $$\sigma$$ using $$k$$, $$n$$, and $$a$$. If valid, decrypts $$c$$ to recover plaintext $$m$$; otherwise, returns $$\mathsf{FAILED}$$.

### AES-GCM: Galois/Counter Mode

**AES-GCM** is an **AEAD (Authenticated Encryption with Associated Data)** algorithm that combines the following two components:

1. [**AES**](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard)**-**[**CTR (Counter Mode)**](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#CTR): A stream cipher method for encrypting plaintext.
2. [**GHASH (Galois Hash)**](https://en.wikipedia.org/wiki/Galois/Counter_Mode#Mathematical_basis): A [universal hash](https://en.wikipedia.org/wiki/Universal_hashing) function used to compute an authentication tag from the ciphertext and associated data.

The plaintext and associated data are divided into blocks:

* $$m = (m_1, \dots, m_{\ell_m})$$
* $$a = (a_1, \dots, a_{\ell_a})$$

where $$\ell_m$$ and $$\ell_a$$ are the number of blocks in the plaintext and AD, respectively.

#### $$\mathsf{AES\text{-}GCM.Enc}_k(n, a, m) \rightarrow (c, \sigma)$$

1. **Encrypt plaintext using AES-CTR**:

$$
c_i = m_i \oplus \mathsf{AES}_k(n + i)
$$

2. **Generate authentication tag using GHASH**:

$$
\sigma =\mathsf{AES}_k(n) \oplus \mathsf{GHASH}(a, c)
$$

#### $$\mathsf{AES\text{-}GCM.Dec}_k(n, a, c, \sigma) \rightarrow m \text{ or } \mathsf{FAILED}$$

1. **Verify authentication tag using GHASH**:

$$
\sigma \stackrel{?}=\mathsf{AES}_k(n) \oplus \mathsf{GHASH}(a, c)
$$

2. **Decrypt ciphertext using AES-CTR**:

$$
m_i = c_i \oplus \mathsf{AES}_k(n + i)
$$

### ChaCha20-Poly1305

[**ChaCha20-Poly1305**](https://en.wikipedia.org/wiki/ChaCha20-Poly1305) is an **AEAD (Authenticated Encryption with Associated Data)** algorithm that combines the following two components:

1. [**ChaCha20**](https://en.wikipedia.org/wiki/Salsa20#ChaCha_variant): A fast stream cipher used for encrypting plaintext.
2. [**Poly1305**](https://en.wikipedia.org/wiki/Poly1305): A universal hash function that generates an authentication tag from the ciphertext and associated data.

#### $$\mathsf{ChaCha20\text{-}Poly1305.Enc}_k(n, a, m) \rightarrow (c, \sigma)$$

1. **Encrypt the plaintext using ChaCha20**:

$$
c_i = m_i \oplus \mathsf{ChaCha20}_k(n, i)
$$

2. **Generate the authentication tag using Poly1305**:

$$
\sigma = s + \mathsf{Poly1305}_{r}(a, c)
$$

Here, $$r$$ and $$s$$ are derived as follows:

$$
(r, s) = \mathsf{ChaCha20}_k(n, 0)
$$

#### $$\mathsf{ChaCha20\text{-}Poly1305.Dec}_k( n, a, c, \sigma) \rightarrow m \text{ or } \mathsf{FAILED}$$

1. **Verify the authentication tag using Poly1305**:

$$
\sigma \stackrel{?}= s + \mathsf{Poly1305}_{r}(a, c)
$$

2. **Decrypt the ciphertext using ChaCha20**:

$$
m_i = c_i \oplus \mathsf{ChaCha20}_k(n, i)
$$

### Context Attacks of AEAD

#### Context Discovery Attack (CDY)

A **Context Discovery Attack (CDY)** refers to an adversary’s ability to find a **valid decryption context** $$(k, n)$$ for a given $$(c, a, \sigma)$$, **even if it is not the original context** used during encryption.

* The goal is not to recover the plaintext itself, but rather to find **any context** $$(k, n)$$ for which decryption **succeeds without error**.
* This attack evaluates **how strongly an AEAD scheme binds a ciphertext to a specific context**.

#### Context Commitment Attack (CMT)

A **Context Commitment Attack (CMT)** occurs when an adversary can find **two different contexts**—say $$(k_1, n_1)$$ and $$(k_2, n_2)$$ with $$(k_1, n_1) \ne (k_2, n_2)$$—**that both successfully decrypt the same** $$(c, a,\sigma)$$.

* The success of this attack implies that **a ciphertext is not uniquely bound to a single context**.
* It exploits the lack of **Key Commitment** in the AEAD scheme.

#### CDY vs. CMT: Relationship and Analogy

<table><thead><tr><th width="253.5947265625">Attack Type</th><th>Hash Function Analogy</th></tr></thead><tbody><tr><td>CDY Attack</td><td><strong>Preimage Attack</strong> (ciphertext → context)</td></tr><tr><td>CMT Attack</td><td><strong>Collision Attack</strong> (same ciphertext → two valid contexts)</td></tr></tbody></table>

<figure><img src="../../.gitbook/assets/image (88).png" alt=""><figcaption></figcaption></figure>

This mirrors the security properties of cryptographic hash functions: **If a hash function is collision-resistant, it is also preimage-resistant.** Similarly, **if an AEAD scheme is secure against CMT attacks, it is also secure against CDY attacks.** (We will discuss _Context Unforgeability_ in a later section.)

### Lack of Key Commitment in AES-GCM and ChaCha20-Poly1305

Although **AES-GCM** and **ChaCha20-Poly1305** are widely used AEAD schemes, they **do not guarantee the Key Commitment property**. That is, the **same ciphertext may be valid under different (key, nonce) combinations**, which introduces a potential vulnerability. Here's why:

* In AES-GCM, the authentication tag is computed as:

$$
\sigma = \mathsf{AES}_k(n) \oplus \mathsf{GHASH}(a, c)
$$

Even if $$\mathsf{AES}_{k_1}(n_1) = \mathsf{AES}_{k_2}(n_2)$$, it’s possible that $$(k_1, n_1) \ne (k_2, n_2)$$.

* In ChaCha20-Poly1305, the tag is computed as:

$$
\sigma = s + \mathsf{Poly1305}_{r}(a, c)
$$

It is also possible that $$\mathsf{ChaCha20}_{k_1}(n_1, 0) = \mathsf{ChaCha20}_{k_2}(n_2, 0)$$, even if $$(k_1, n_1) \ne (k_2, n_2)$$.

According to the referenced paper, in AEAD schemes like AES-GCM and ChaCha20-Poly1305, an adversary $$\mathcal{A}$$ making at most $$q$$ queries to the oracle can **succeed in a CMT (Context Commitment) attack with probability at least** $$\frac{q}{2^{32}}$$**.**

## Protocol Explanation

### Proxy-based zkTLS

<figure><img src="../../.gitbook/assets/image (89).png" alt=""><figcaption></figcaption></figure>

As illustrated above, when the verifier $$\mathcal{V}$$ acts as a proxy between the prover (or TLS client) $$\mathcal{P}$$ and the TLS server $$\mathcal{S}$$, the following advantages arise:

1. There is no 2PC (two-party computation) overhead, offering a performance benefit.
2. Since $$\mathcal{V}$$ only relays messages, this design maintains compatibility with various versions of TLS.
3. Unlike DECO, no modifications are required on either the server or the client side.

***

In DECO, since $$\mathcal{P}$$ knows $$k^\mathsf{MAC}$$, it can forge a ciphertext $$c'$$ instead of relaying the actual server response $$c$$ from $$\mathcal{S}$$ using $$k^\mathsf{MAC}$$. To prevent this, DECO splits $$k^\mathsf{MAC}$$ between $$\mathcal{P}$$ and $$\mathcal{V}$$, and only allows $$\mathcal{V}$$ to reveal its share after $$\mathcal{P}$$ commits to $$c$$.

This issue can also be addressed in the proxy-based approach. Because $$\mathcal{V}$$, acting as a proxy, observes all encrypted requests and responses exchanged between $$\mathcal{P}$$ and $$\mathcal{S}$$, it can detect if $$\mathcal{P}$$ attempts to tamper with the response and use a forged $$c'$$ instead of the genuine $$c$$.

In DECO, the issue of forgery arises from $$\mathcal{P}$$’s knowledge of $$k^\mathsf{MAC}$$, allowing it to manipulate the ciphertext using that key. To mitigate this, the key was split between $$\mathcal{P}$$ and $$\mathcal{V}$$, and $$\mathcal{V}$$ would only reveal its key share after receiving a commitment to $$c$$ from $$\mathcal{P}$$.

The proof in this design can be structured as follows:

$$
C(x:\{ \hat{m}, a, c, \sigma \}, w: \{ \textcolor{red}{\tau}, k, n, m \}): \\
\textcolor{red}{\mathsf{AEAD.Gen}(\tau)  \stackrel{?}= (k, n)} \land \mathsf{AEAD.Enc}_k(n, a, m) \stackrel{?}= (c, \sigma) \land \hat{m} \in m  \land \mathsf{Pred}(\hat{m})
$$

Here, $$\mathsf{Pred}$$ is a user-defined predicate representing the condition the user wants to prove about $$\hat{m}$$.

***

However, the constraint $$\mathsf{AEAD.Gen(\tau)} \stackrel{?}= (k, n)$$ introduces significant complexity. To simplify the circuit, we may omit this constraint and instead define the circuit as:

$$
C(x:\{ \hat{m}, a, c, \sigma \}, w: \{ k, n, m \}): \\
 \mathsf{AEAD.Enc}_k(n, a, m) \stackrel{?}= (c, \sigma) \land \hat{m} \in m  \land \mathsf{Pred}(\hat{m})
$$

then a serious security vulnerability may arise.

This is because AEADs used in TLS 1.2/1.3 **do not satisfy Key Commitment**, making it possible to find two different combinations $$(k_1, n_1, m_1)$$ and $$(k_2, n_2, m_2)$$ that decrypt the same triple $$(a, c, \sigma)$$. For example:

* $$r_1 = (x: \{  \hat{m}_1, a, c, \sigma \}, w: \{k_1, n_1, m_1\}) \in R$$
* $$r_2 = (x: \{ \hat{m}_2, a, c, \sigma \}, w: \{k_2, n_2, m_2\}) \in R$$

If $$\hat{m}_1 \ne \hat{m}_2$$, and particularly if $$\mathsf{Pred}(\hat{m}_1) \ne \mathsf{Pred}(\hat{m}_2)$$, then **multiple conflicting claims can be proven over the same ciphertext**, leading to **critical security failures**.

This vulnerability stems from the structural properties of TLS itself and **may affect systems like DECO in a similar way**.

However, if the underlying AEAD scheme **does satisfy Key Commitment**, then **secure zk proofs can still be constructed in the proxy-based model**, as proven in _Theorem 5.2_ of the referenced paper.

### Case 1: If You Are Using HTTPS

#### Fixed Padding

According to the paper “[How to Abuse and Fix Authenticated Encryption Without Key Commitment](https://eprint.iacr.org/2020/1456),” the key commitment issue can be mitigated using fixed padding. For example, consider prepending 128 bytes of `0x00` to the plaintext as padding. Decrypting to exactly 128 bytes of zeros is extremely difficult. The decryption process of AES-GCM works as follows:

$$
m_i = c_i \oplus \mathsf{AES}_k(n + i)
$$

Assume that $$b$$ fixed padding blocks are inserted at the beginning of the plaintext. For the decrypted output to exactly match the padding, the following condition must hold:

$$
\forall\, 1 \le i \le b,\quad \mathsf{AES}_{k_1}(n_1 + i) = \mathsf{AES}_{k_2}(n_2 + i)
$$

In other words, different key/nonce pairs $$(k_1, n_1) \ne (k_2, n_2)$$ must produce identical key streams for $$b$$ blocks—an extremely unlikely event.

This reasoning is justified when modeling AES as an **ideal cipher**.

***

**📌 Definition: Ideal Cipher**

* An ideal cipher is defined as a collection of functions $$\mathcal{E} = \{ E_k : \mathcal{M} \to \mathcal{C} \}$$ from a message space $$\mathcal{M}$$ to a ciphertext space $$\mathcal{C}$$.
* Each $$E_k$$ is a **random permutation** over $$\mathcal{M}$$.
* That is, for each key $$k$$, the function $$E_k$$ is assumed to be **independent and completely random**.

> 💡 In other words, under the ideal cipher model, even when the same input is used with different keys $$k_1$$ and $$k_2$$, the outputs are almost certainly unrelated.

***

Then a natural question arises: **do we have this kind of padding structure in TLS?**

#### Variable Padding

<figure><img src="../../.gitbook/assets/image (90).png" alt=""><figcaption><p>Source: <a href="https://youtu.be/xcA3o2NGlfw?t=845">https://youtu.be/xcA3o2NGlfw?t=845</a></p></figcaption></figure>

As shown in the image above, many web servers—including those of Google and Twitter—prepend HTTP responses with consistently structured data such as the `HTTP status code` and `Date` fields. This formatting is considered **best practice** under [RFC 7231](https://datatracker.ietf.org/doc/html/rfc7231), and widely adopted by major web servers like **Apache** and **Nginx**. According to the paper, more than **85% of webpages** begin with this kind of pattern.

Although these values are not fixed, their possible combinations are **highly constrained**, offering useful—though weaker—security compared to fixed padding. This is referred to as **Variable Padding**.

For example:

* There are approximately **63** `HTTP status codes`, based on the [official IANA registry](https://www.iana.org/assignments/http-status-codes/http-status-codes.xhtml) (excluding unassigned codes). (Interestingly, when I counted them myself, there were 64 `HTTP status codes`. :thinking:)
* The `Date` field can vary per second, but within a 1-hour time window, it can take on **3600 values**.

Thus, the total number of possible combinations is:

$$
63 \times 3600 = 226800
$$

If the `HTTP status code`  and `Date` header together form a 54-byte string, this effectively corresponds to having $$2^{54 \times 8}$$ possible padding values. It means that **forcing a decryption result to match one of these valid patterns is statistically infeasible for an attacker**. In this sense, such structural padding acts as a practical substitute for full key commitment and can offer meaningful protection. (For a detailed analysis, refer to [Section 6.2](https://eprint.iacr.org/2024/733.pdf#page=11\&zoom=100,65,675) of the paper.)

However, this security assumption relies on the presence of HTTPS. **What happens if HTTPS is not available?**

### Case 2: If HTTPS Is Not Used

#### Context Forgery Attack (CFY)

A **CFY attack** aims to find a **different context** $$(k_2, n_2)$$ such that a given $$(c, a, \sigma)$$, originally generated under a key and nonce $$(k_1, n_1)$$, can still be **successfully decrypted**.

* The constraint is that $$(k_1, n_1) \ne (k_2, n_2)$$.
* In other words, the attacker tries to determine whether the ciphertext can be decrypted **under a different** $$(k, n)$$ **pair** than the one originally used.
* This attack model lies **between** the Context Discovery (CDY) and Context Commitment (CMT) models, and it evaluates whether an AEAD scheme enforces **decryption only under a unique context**.

#### CFY ↔ Second Preimage Attack Analogy

The **structure of a CFY attack closely resembles that of a Second Preimage Attack** in cryptographic hash functions. In a Second Preimage Attack, the attacker is given an input $$x$$ and seeks a different input $$x' \ne x$$ that hashes to the same value:

$$
x \ne x' \quad \land \quad H(x) = H(x')
$$

This is different from a Preimage Attack, where one is given only a hash output $$h$$ and must find **any** corresponding input $$x$$:

$$
h = H(x)
$$

In this analogy, the relationship is:

* **Preimage Attack ↔ CDY Attack**
* **Second Preimage Attack ↔ CFY Attack**

#### Comparison table of context attack on AEAD

The table below compares different types of context attacks on AEAD.

| Attack Type                                         | What the Attacker Knows      | What the Attacker Finds                                                            | Analogous Hash Attack      |
| --------------------------------------------------- | ---------------------------- | ---------------------------------------------------------------------------------- | -------------------------- |
| <p><strong>CDY</strong><br>(Context Discovery)</p>  | $$(c, a, \sigma)$$           | One valid $$(k, n)$$ that produces $$(c, a, \sigma)$$                              | **Preimage** attack        |
| <p><strong>CFY</strong><br>(Context Forgery)</p>    | $$(c, a, \sigma, k_1, n_1)$$ | A second valid $$(k_2, n_2)$$ that produces $$(c, a, \sigma)$$                     | **Second Preimage** attack |
| <p><strong>CMT</strong><br>(Context Commitment)</p> |                              | Two valid $$(k_1, n_1)$$ and $$(k_2, n_2)$$ that produce a same $$(c, a, \sigma)$$ | **Collision** attack       |

The zkTLS attacker must perform a **CFY attack**, not a **CDY attack**. This is because $$(k_1, n_1)$$ is already a valid decryption context, and the attacker’s goal is to find a **different** context $$(k_2, n_2)$$ that also successfully decrypts the ciphertext.

#### AES-GCM CFY Attack Analysis

For simplicity, let’s assume the ciphertext $$c$$ consists of a single block and there is no associated data $$a$$. Under these conditions, the AES-GCM tag $$\sigma$$ is computed as follows:

$$
\sigma = \mathsf{AES}_k(n) \oplus \mathsf{GHASH}(c) = \mathsf{AES}_k(n) \oplus \mathsf{AES}_k(0)^2c \oplus \mathsf{AES}_k(0)
$$

Now, suppose an attacker randomly samples a different key $$k' \ne k$$, and attempts to generate the same tag $$\sigma$$ using a new context $$(k', n')$$:

$$
\sigma = \mathsf{AES}_{k'}(n') \oplus \mathsf{GHASH}(c) = \mathsf{AES}_{k'}(n') \oplus \mathsf{AES}_{k'}(0)^2c \oplus \mathsf{AES}_{k'}(0)
$$

Assuming AES behaves as an **ideal cipher**, each key defines a **reversible** permutation. Thus, the attacker can rearrange the above equation to solve for $$n'$$:

$$
n' = \mathsf{AES}_{k'}^{-1}(\sigma \oplus\mathsf{AES}_{k'}(0)^2c \oplus \mathsf{AES}_{k'}(0) )
$$

In other words, given $$\sigma$$, $$c$$, and $$\mathsf{AES}_{k'}(0)$$, the attacker can **compute a candidate nonce** $$n'$$ that would yield the same tag.

***

⚠️ However, in TLS, nonces must follow a specific format: among the 64-bit nonce, the **last 32 bits must equal** $$(0^{31}‖1)$$. Therefore, the probability that a randomly generated $$n'$$ satisfies this constraint **by chance** is:

$$
\frac{1}{2^{32}}
$$

***

Each attempt with a new $$k'$$ requires two AES invocations: one for $$\mathsf{AES}_{k'}(0)$$ _and one for_ $$\mathsf{AES}_{k'}(n')$$. If the attacker is limited to a total of $$q$$ AES queries, they can make approximately $$\frac{q}{2}$$ attempts.

Thus, the **upper bound on the attack success probability** is:

$$
\frac{q}{2} \cdot \frac{1}{2^{32}} = \frac{q}{2^{33}}
$$

#### ChaCha20-Poly1305 CFY Attack Analysis

To simplify the discussion, assume the ciphertext $$c$$ consists of a single block and there is no associated data $$a$$. Under this assumption, the ChaCha20-Poly1305 tag $$\sigma$$ is computed as:

$$
\sigma = s + \mathsf{Poly1305}_{r}(a)
$$

Here, $$r$$ and $$s$$ are computed as follows:

$$
(r, s) = \mathsf{ChaCha20}_k(n, 0)
$$

The attacker randomly samples a different key and nonce pair $$(k', n') \ne (k, n)$$ and attempts to generate the same tag $$\sigma$$ using this new context. For the attack to succeed, the following condition must hold:

$$
\sigma = s + \mathsf{Poly1305}_{r}(a) = s' + \mathsf{Poly1305}_{r'}(a) \\
$$

Rearranging this equation, we obtain:

$$
s -s' =\mathsf{Poly1305}_{r'}(a) - \mathsf{Poly1305}_{r}(a)
$$

***

Let’s analyze the distribution of both sides.

* Since $$\mathsf{ChaCha20}$$ behaves as a [Pseudorandom Function (PRF)](https://en.wikipedia.org/wiki/Pseudorandom_function_family), different $$(k, n)$$ pairs will produce independent and uniformly random $$(r, s)$$ values. Therefore, $$s - s'$$ is uniformly distributed over $$\mathbb{Z}/2^{128}\mathbb{Z}$$.
* $$\mathsf{Poly1305}$$ is a [Universal Hash](https://en.wikipedia.org/wiki/Universal_hashing), meaning that its output is also uniformly distributed. Hence, if we define:

$$
\Delta p(r') := \mathsf{Poly1305}_{r'}(a) - \mathsf{Poly1305}_r(a)
$$

then $$\Delta p(r')$$ is uniformly distributed over $$\mathbb{Z}/2^{128}\mathbb{Z}$$ as well.

Thus, the probability that the equation holds — i.e., the probability that the attack succeeds — is:

$$
\begin{align*}
\Pr(\mathsf{BAD}) &= \sum_{i \in \mathbb{Z}/2^{128}\mathbb{Z}} \Pr(s' - s = i) \Pr(\Delta p(r') = i) \\
&= \frac{1}{2^{128}}\sum_{i \in \mathbb{Z}/2^{128}} \Pr(\Delta  p(r') = i) = \frac{1}{2^{128}}
\end{align*}
$$

Therefore, if the attacker is allowed to make $$q$$ queries, the **upper bound on the total success probability** is:

$$
\frac{q}{2^{128}}
$$

#### The Necessity of Key Commitment for Fixed Data

Whether AEAD schemes require the **Key Commitment** property depends on the nature of the data being protected.

* **When Key Commitment is essential**: For sensitive information such as **bank account balances**, Key Commitment is critical. If the ciphertext is decrypted under a forged context, it may result in **misinterpreted financial data**, which could lead to serious financial incidents. Therefore, decryption must succeed **only under a single, correct context**, a property guaranteed when AEAD satisfies **Key Commitment**.
* **When Key Commitment is not required**: Data such as **age, account numbers, or national ID numbers** are examples of **Fixed Data**, which do not change across sessions. Even if an attacker manipulates the context to alter the age from 20 to 80, for instance, **proper external safeguards** can **detect or prevent** such tampering effectively.

Even in environments where **HTTPS cannot be used**, and therefore **Variable Padding is unavailable**, there is **no need for despair**.

As discussed earlier, when using **ChaCha20-Poly1305** as the AEAD scheme, even though it lacks Key Commitment, the **probability of a successful CFY attack is extremely low**, and a **zkTLS protocol based on Fixed Data** can still be implemented **safely and practically**.

## Conclusion

This article examined the **Key Commitment issue of AEAD schemes** in the context of selective data disclosure over TLS. In particular, it highlighted the limitations of the existing DECO protocol and explored how a **proxy-based approach** could overcome them. The proxy-based design has the advantage of requiring **no modifications to either the server or the client**, while still enabling selective disclosure through zero-knowledge proofs.

However, since **AES-GCM and ChaCha20-Poly1305**, the AEAD schemes commonly used in TLS, **do not guarantee the Key Commitment property**, any such protocol must ensure that **AEAD decryption is tightly bound to a unique context** to be considered secure.

This article proposed two practical solutions to address the issue:

1. In an **HTTPS environment**, the use of **Variable Padding** can enforce a predictable structure in ciphertexts, helping to mitigate the lack of key commitment.
2. In **non-HTTPS environments**, the **statistical security guarantees** of AEAD schemes like **ChaCha20-Poly1305** make it **feasible to build a secure zkTLS protocol**, especially for **Fixed Data**.

Therefore, while enforcing full Key Commitment in all scenarios may be **unrealistic**, by carefully analyzing the **data characteristics**, the **structure of the AEAD scheme**, and the **capabilities of potential adversaries**, it is **entirely feasible to design a secure and practical zkTLS system**.

That said, one possible drawback of the proxy-based approach is that the proxy has visibility into all communications, which could be viewed as a trade-off.

## References

* [https://www.youtube.com/watch?v=xcA3o2NGlfw](https://www.youtube.com/watch?v=xcA3o2NGlfw)
* [https://eprint.iacr.org/2024/733.pdf](https://eprint.iacr.org/2024/733.pdf)
