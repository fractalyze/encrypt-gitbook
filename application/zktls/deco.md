---
description: 'Presentation: https://www.youtube.com/watch?v=cTBqiwAx5jA'
---

# DECO

## Introduction

Thanks to TLS, users can access their private data securely with guarantees of end-to-end confidentiality and integrity. However, in the traditional TLS model, it is **impossible to prove this data to a third party without relying on trust assumptions**. In other words, a user’s private data remains **trapped at its origin**, with no cryptographically verifiable way to share or prove it externally.

<figure><img src="../../.gitbook/assets/image (2) (4).png" alt=""><figcaption></figcaption></figure>

**DECO** (**DEC**entralized **O**racle) is a protocol proposed by researchers at Cornell in 2019 that addresses this problem. It enables **privacy-preserving and selective disclosure** of data transmitted over TLS, allowing users to prove facts about their data to third parties **without revealing the underlying content**.

Moreover, DECO is the first oracle protocol to support both TLS 1.2 and TLS 1.3, making it compatible with modern web security standards **without requiring any modifications to the server**. Prior to DECO, such functionality required either **modifying the server** or relying on **Trusted Execution Environments (TEEs)**.

## Background

### ECDH (Elliptic Curve Diffie-Hellman)

**Elliptic Curve Diffie-Hellman (ECDH)** is a key exchange protocol built on **elliptic curve cryptography (ECC)**. It enables two parties to **safely derive a shared secret key** over an insecure channel using only public keys.

This is an elliptic-curve variant of the classic **Diffie-Hellman Key Exchange (DHKE)** protocol.

Let $$G$$ be a generator point on the elliptic curve:

1. **Private key generation**:\
   Alice and Bob independently sample secret keys $$a, b$$.
2. **Public key computation**:
   * Alice: $$A = a \cdot G$$
   * Bob: $$B = b \cdot G$$
3. **Key exchange**:
   * Alice sends $$A$$ to Bob.
   * Bob sends $$B$$ to Alice.
4. **Shared secret key computation**:
   * Alice computes $$S = a \cdot B = a \cdot (b \cdot G)$$
   * Bob computes $$S = b \cdot A = b \cdot (a \cdot G)$$

Thus, both parties derive the same shared key without revealing their private inputs.

### MAC-then-Encrypt

<figure><img src="../../.gitbook/assets/image (3) (4).png" alt=""><figcaption><p>Source: <a href="https://en.wikipedia.org/wiki/Authenticated_encryption#MAC-then-Encrypt_(MtE)">https://en.wikipedia.org/wiki/Authenticated_encryption#MAC-then-Encrypt_(MtE)</a></p></figcaption></figure>

**MAC-then-Encrypt** is the authenticated encryption approach used in **TLS 1.2 and earlier**. It ensures both **confidentiality** and **integrity** by:

1. Computing a message authentication code (MAC) with the key $$k^\mathsf{MAC}$$ and the message $$m$$:

$$
\sigma = \mathsf{MAC}(k^{\mathsf{MAC}}, m)
$$

2. Encrypting the message and MAC with the key $$k^{\mathsf{Enc}}$$:

$$
C = \mathsf{Encrypt}(k^\mathsf{Enc}, m \| \sigma)
$$

To decrypt and verify $$C$$:

1. Decrypt the ciphertext:

$$
(m, \sigma) = \mathsf{Decompose(Decrypt}(k^\mathsf{Enc},C))
$$

2. Verify the MAC:

$$
\sigma \stackrel{?}= \mathsf{MAC}(k^\mathsf{MAC}, m)
$$

### HMAC (Hash-based Message Authentication Code)

**HMAC** is a type of **Message Authentication Code (MAC)** that uses a **hash function** to ensure both **integrity** and **authentication** of data.

The core idea is to **combine a cryptographic hash function with a secret key** to generate an authentication code (MAC) for a message. This allows the recipient to verify that the data has **not been tampered with** and that it was sent by a **trusted sender**.

HMAC is computed as follows:

$$
\mathsf{HMAC}(k, m) \rightarrow H((k \oplus \mathsf{opad}) \| (H(k \oplus \mathsf{ipad}) \| m))
$$



* $$\mathsf{opad}$$: outer padding (0x5c)
* $$\mathsf{ipad}$$: inner padding (0x36)

### AES-CBC-HMAC

**AES-CBC-HMAC** is a scheme that combines **AES encryption in CBC (Cipher Block Chaining) mode** with **HMAC-based authentication** to ensure both **confidentiality** and **integrity** of the message. It follows the **MAC-then-Encrypt** paradigm.

#### Encryption

1. Generate HMAC:

$$
\sigma = \mathsf{HMAC}(k^{\mathsf{MAC}} , m)
$$

2. Encrypt the message and MAC using AES in CBC mode:

$$
C = \mathsf{AES\text{-}CBC.Encrypt}(k^\mathsf{Enc}, m \| \sigma, IV)
$$

* $$IV$$ is the **Initialization Vector**, which is sent along with the ciphertext $$C$$.

#### Decryption

1. Decrypt and split the message:

$$
(m, \sigma) = \mathsf{Decompose(AES\text{-}CBC.Decrypt}(k^\mathsf{Enc}, C, IV))
$$

2. Verify the MAC:

$$
\sigma \stackrel{?}= \mathsf{HMAC}(k^{\mathsf{MAC}} , m)
$$

TODO(chokobole): Add AES-GCM

### TLS

#### Handshake Protocol

The **TLS handshake** is the process by which a client and a server establish a secure connection. It typically involves the following steps:

1. **Cipher suite negotiation**: The client and server agree on which cryptographic algorithms (cipher suite) to use.
2. **Server authentication:** The server presents a digital certificate (usually [X.509](https://en.wikipedia.org/wiki/X.509)) to prove its identity.\
   Client authentication is optional (e.g., in non-mutual TLS setups).
3. **Session key generation:** The client and server derive symmetric keys:
   * $$k^{\mathsf{Enc}}$$: symmetric encryption key
   * $$k^{\mathsf{MAC}}$$: symmetric authentication key

These keys are used in the following **Record Protocol** for data transmission.

DECO supports **ECDHE (Elliptic Curve Diffie-Hellman Ephemeral)** as the key exchange method.

#### Record Protocol

The **TLS Record Protocol** operates in the following steps:

1. **Data fragmentation**: The sender splits application data $$\bm{D}$$ into **fixed-size plaintext records**\
   $$\bm{D} = (D_1, D_2, \dots, D_n)$$. The last block $$D_n$$​ is **padded** if necessary.
2. **Optional compression**: The data may optionally be compressed before encryption.
3. **Authenticated encryption**: Each record is encrypted and authenticated using one of the following modes:
   * TLS ≤ 1.2: **MAC-then-Encrypt** (e.g., AES-CBC-HMAC)
   * TLS ≥ 1.3: **AEAD modes** (e.g., AES-[GCM](https://en.wikipedia.org/wiki/Galois/Counter_Mode), [ChaCha20-Poly1305](https://en.wikipedia.org/wiki/ChaCha20-Poly1305))
4. **Decryption and integrity check (receiver side)**: The receiver decrypts the record and verifies integrity by recomputing the MAC.
5. **Optional decompression and reassembly**: If compression was applied, the data is decompressed and multiple records are combined to reconstruct the original application data.

DECO supports both **CBC-HMAC** and **GCM mode** for AES-based encryption.

## Protocol Explanation

```json
{
  "accounts": [
    { "account_id": 1, "balance": 500 },
    { "account_id": 2, "balance": 2000 },
    { "account_id": 3, "balance": 5000 }
  ]
}

```

Let’s assume the above JSON message was received from a bank server over TLS. Our goal is to prove in zero-knowledge (ZK) that the account with `account_id = 2` has a balance greater than or equal to 1000, without revealing any information about the other accounts.

For simplicity, we assume that the TLS session uses **AES-CBC-HMAC** (as in TLS 1.2), though this can be extended to **AES-GCM** as well.

TODO(chokobole): Add how DECO is instantiated with AES-GCM

### Method 1: Naive Approach

The most straightforward idea is to construct a circuit like the one below and generate a ZK proof.\
(We omit **AES-CBC's** IV for clarity.)

$$
C(x: \{ C, q \}, w: \{ k^\mathsf{Enc}, k^\mathsf{MAC}, m \}) : \\ \sigma \stackrel{?}= \mathsf{HMAC}(k^{\mathsf{MAC}} , m) \land C \stackrel{?}= \mathsf{AES\text{-}CBC.Encrypt}(k^\mathsf{Enc}, m \| \sigma) \land \mathsf{jq.exec}(m, q) \ge 1000
$$

Here, we use the notation from [the circuit overview](../../zk/overview.md#circuit).

* $$q$$ is the [jq](https://jqlang.org/) query: `.accounts[] | select(.account_id == 2) | .balance`
* $$\mathsf{jq.exec}(m, q)$$ denotes applying the jq query $$q$$ to message $$m$$

<figure><img src="../../.gitbook/assets/image (91).png" alt=""><figcaption></figcaption></figure>

#### ❗ Problem

This naive approach has a critical security flaw: **The client already knows the MAC key** $$k^{\mathsf{MAC}}$$.

This means the client can **forge a valid MAC for a tampered message**. For example, even if the real balance is only 500, the client could create a fake message like:

```json
{
  "accounts": [
    { "account_id": 1, "balance": 500 },
    { "account_id": 2, "balance": 9999 },
    { "account_id": 3, "balance": 5000 }
  ]
}
```

and generate a valid MAC on it—making it appear as though the server authenticated this falsified message.

Thus, performing MAC verification entirely inside the ZK circuit **fails to prevent forgery when the client knows the MAC key**.

### Method 2: Three-Party Handshake

The limitation of the naive approach lies in the fact that, under the standard TLS model, the **client learns the MAC key** $$k^{\mathsf{MAC}}$$ **before receiving the server’s response** $$R$$.

Because the client directly knows the MAC key, it can **forge a message and still generate a valid ZK proof**, introducing a serious vulnerability.

To resolve this, **DECO introduces a Three-Party Handshake** involving:

* $$\mathcal{P}$$: TLS client and ZK prover
* $$\mathcal{V}$$: Verifier
* $$\mathcal{S}$$: TLS server

#### :bulb: Core Idea

* The **MAC key** $$k^{\mathsf{MAC}}$$ is split between $$\mathcal{P}$$ and $$\mathcal{V}$$, so that $$\mathcal{P}$$ **cannot compute it alone**.
* $$\mathcal{P}$$ first **commits** to the received TLS message.
* Only then does $$\mathcal{V}$$ reveal their MAC key share, preventing the prover from tampering with the message (At this moment, $$\mathcal{P}$$ and $$\mathcal{V}$$ know the full MAC key $$k^\mathsf{MAC}$$).

#### Step 1: Key Exchange

<figure><img src="../../.gitbook/assets/Screenshot 2025-03-27 at 5.53.37 PM.png" alt=""><figcaption></figcaption></figure>

#### Step 2: Key Derivation

_<mark style="background-color:yellow;">TODO(chokobole):</mark>  Add details on how the MAC key is derived from the shared secrets_ $$Z_\mathcal{P}$$_​ and_ $$Z_\mathcal{V}$$_._

#### ❗ Problem

In **Method 1**, the prover $$\mathcal{P}$$ knows the full MAC key $$k^{\mathsf{MAC}}$$, allowing them to compute HMAC and all related operations **locally**.

However, from **Method 2 onward**, the MAC key is only **additively shared** between $$\mathcal{P}$$ and $$\mathcal{V}$$, meaning:

* $$\mathcal{P}$$ cannot reconstruct the full key.
* **HMAC must be computed via a Two-Party MPC (2PC)** protocol. In order to send a query, the prover $$\mathcal{P}$$ and the verifier $$\mathcal{V}$$ compute the MAC together as follows(We use the notation defined in [here](../../mpc/overview.md#n-party-mpc)) :

$$
\mathsf{MPC}_2(\mathsf{HMAC}, k^\mathsf{MAC}_\mathcal{P}, k^\mathsf{MAC}_\mathcal{V}, m) \rightarrow \mathsf{HMAC}(k^\mathsf{MAC}_\mathcal{P} + k^\mathsf{MAC}_\mathcal{V}, m)
$$

This introduces a performance challenge: **Naively computing HMAC via 2PC is expensive**, as HMAC internally invokes the hash function **twice**, each requiring complex boolean circuits with a high gate count.

### Method 3: Query Optimization

Now in order to send a query, the prover $$\mathcal{P}$$ and the verifier $$\mathcal{V}$$ compute the MAC together as follows:

To address the performance bottleneck in Method 2, Method 3 applies an **optimization strategy that leverages the structural properties of HMAC and the internal workings of SHA-256**.

The goal is to **minimize the scope of 2PC computations** by offloading as much of the work as possible to local computation.

$$
\mathsf{HMAC}(k, m) \rightarrow \underbrace{H((k \oplus \mathsf{opad}) \| \underbrace{H\left((k \oplus \mathsf{ipad}) \| m\right)}_{\text{inner hash}})}_{\text{outer hash}}
$$

SHA-256 follows the [Merkle–Damgård](https://en.wikipedia.org/wiki/Merkle%E2%80%93Damg%C3%A5rd_construction) construction, processing messages in blocks. It computes hashes in the following recursive manner:

$$
H(m_1 \| m_2) = f_H(f_H(IV, m_1), m_2)
$$

where:

* $$f_H$$ is the compression function of the hash
* $$IV$$ is the initial vector.

Using this structure, HMAC can be split and optimized as follows:

1. **Initial compression via 2PC**:

$$
s_0 = f_H(IV, k \oplus \mathsf{ipad})
$$

2. **Compute inner hash locally**:

$$
s_1 = H(s_0 \| m)
$$

As $$s_0$$​ is already known, the prover $$\mathcal{P}$$ can compute the **inner hash** $$s_1$$​ locally using the full message $$m$$.

3. **Compute outer hash via 2PC only**:

$$
\mathsf{MPC}_2(\mathsf{HMAC\text{-}opt}, k^\mathsf{MAC}_\mathcal{P}, k^\mathsf{MAC}_\mathcal{V}, s_1) \rightarrow \mathsf{HMAC\text{-}opt}(k^\mathsf{MAC}_\mathcal{P} + k^\mathsf{MAC}_\mathcal{V}, s_1)
$$

where $$\mathsf{HMAC\text{-}opt}$$ is defined as:

$$
\mathsf{HMAC\text{-}opt}(k, s_1) = H((k \oplus \mathsf{opad}) \| s_1)
$$

By computing the most expensive part of the computation (inner hash) locally, this optimization significantly reduces the cost of 2PC.

### Method 4: AES-CBC-HMAC Optimization

As in Method 1, verifying the entire AES-CBC-HMAC process **inside a ZK circuit introduces significant computational overhead**. To address this, we can apply a circuit-level optimization that leverages the **CBC** structure and **MAC-then-Encrypt** semantics.

Assume that the message $$m$$ consists of 1024 AES blocks, and the MAC $$\sigma$$ consists of 3 fixed-size AES blocks:

$$
\bm{M} = (B_1, \dots, B_{1024}, \sigma), \quad \bm{\sigma} = (B_{1025}, B_{1026}, B_{1027})
$$

After **CBC** encryption, it transforms into:

$$
\hat{\bm{M}} = (\hat{B}_1, \dots, \hat{B}_{1024}, \hat{\sigma}), \quad \bm{\hat{\sigma}} = (\hat{B}_{1025}, \hat{B}_{1026}, \hat{B}_{1027})
$$

If we naively construct the circuit, we would need to perform **1027 AES encryptions inside the circuit**:

$$
C(x: \{ \bm{\hat{M}} \}, w: \{ k^\mathsf{Enc}, \bm{M} \}) : \bigwedge_{i=1}^{1027} \hat{B}_i \stackrel{?}= \mathsf{AES\text{-}CBC.Encrypt}(k^\mathsf{Enc}, B_i)
$$

#### Optimization 1: Verifying MAC Blocks Only

Using the MAC-then-Encrypt structure, we can **only verify the MAC portion of the CBC-encrypted message** in the circuit. The HMAC itself can be checked outside the circuit:

$$
C(x: \{ \textcolor{red}{\bm{\sigma}, \bm{\hat{\sigma}}} \}, w: \{ k^\mathsf{Enc} \}) : \textcolor{red}{\bigwedge_{i=1025}^{1027}}\hat{B}_i \stackrel{?}= \mathsf{AES\text{-}CBC.Encrypt}(k^\mathsf{Enc}, B_i)
$$

Then the verifier $$\mathcal{V}$$ checks the HMAC externally:

$$
\sigma \stackrel{?}= \mathsf{HMAC}(k^\mathsf{MAC}, m)
$$

This is secure under the assumption that HMAC is collision-resistant.

#### Optimization 2: Include Only Sensitive Block in the Circuit

If the $$i$$-th block (where $$i \in [1024]$$) contains sensitive data, we can selectively include it in the circuit:

$$
C(x: \{ \bm{\sigma}, \bm{\hat{\sigma}}, \textcolor{red}{\bm{B_{i-}}, \bm{B_{i+}}} \}, w: \{ k^\mathsf{Enc}, \textcolor{red}{k^{\mathsf{MAC}}, B_i} \}) : \\ \bigwedge_{j=1025}^{1027} \hat{B}_j \stackrel{?}= \mathsf{AES\text{-}CBC.Encrypt}(k^\mathsf{Enc}, B_j) \land \textcolor{red}{\sigma \stackrel{?}= \mathsf{HMAC}(k^\mathsf{MAC}, \bm{B_{i-}}\|B_i\|\bm{B_{i+}})}
$$

Where:

* $$\bm{B_{i-}} = (B_1, \dots, B_{i-1})$$
* $$\bm{B_{i+}} = (B_{i+1}, \dots, B_{1024})$$

#### Optimization 3: Apply HMAC Optimization from Method 3

We can further reduce the circuit by applying the HMAC optimization described in Method 3:

$$
C(x: \{ \bm{\sigma}, \bm{\hat{\sigma}}, \textcolor{red}{s_{i-1}, s_i} \}, w: \{ k^\mathsf{Enc}, B_i \}) : \\ \bigwedge_{j=1025}^{1027} \hat{B}_j \stackrel{?}= \mathsf{AES\text{-}CBC.Encrypt}(k^\mathsf{Enc}, B_j) \land \textcolor{red}{s_i \stackrel{?}= f_H(s_{i-1}, B_i)}
$$

Then, outside the circuit:

1. Compute $$s_{i-1} = H(\bm{B_{i-}})$$
2. Verify $$s_{i-1} \stackrel{?}{=} x.s_{i-1}$$
3. Verify $$\sigma \stackrel{?}{=} H((k \oplus \mathsf{opad}) \| H(s_i \| \bm{B_{i+}}))$$

#### Final Circuit Structure

$$
C(x: \{ \textcolor{red}{\bm{\sigma}, \bm{\hat{\sigma}}, s_{i-1}, s_i} \}, w: \{ k^\mathsf{Enc}, \textcolor{red}{B_i} \}) : \\ \textcolor{red}{\bigwedge_{j=1025}^{1027} \hat{B}_j \stackrel{?}= \mathsf{AES\text{-}CBC.Encrypt}(k^\mathsf{Enc}, B_j) \land s_i \stackrel{?}= f_H(s_{i-1}, B_i) \land \mathsf{extractBalance(}B_i) \ge 1000}
$$

#### ❗ Problem

However, there's still a critical issue: **How do we know that** $$B_i$$**​ actually corresponds to the field we intend to prove?**

For example, if we're trying to prove the balance of `account_id = 2`, there's nothing in the above circuit that guarantees $$B_i$$​ isn't actually from account 1 or 3.

Thus, we need an additional mechanism to ensure:

* **Context Integrity** — the value is taken from the correct location

### Method 5: Context Integrity via Structured Commitments

This method is based on the [Chainlink blog post on DECO](https://blog.chain.link/deco-parsing-the-response/#soundness_checks) and aims to resolve a key limitation identified in Method 4—namely, **the inability to verify that a block truly corresponds to the intended field**.

To address this, the prover $$\mathcal{P}$$ constructs a sanitized version of the TLS response message $$R$$, denoted $$R'$$, where **sensitive data fields are removed**.

For example, suppose the original message looks like this:

```json
{
  "accounts": [
    { "account_id": "", "balance": "" },
    { "account_id": "", "balance": "" },
    { "account_id": "", "balance": "" }
  ]
}
```

The actual values that were removed are committed separately as a vector of scalars:

$$
Z = (1, 500, 2, 2000, 3, 5000)
$$

#### Verification Process

The verifier proceeds as follows:

Each $$Z_k$$​ represents a redacted numeric value from the response, and the remaining static content is encoded as a list of **fixed string tokens** $$P$$, representing the prefix/suffix structure of the original JSON:

```json
P = [
  "{\n  \"accounts\": [\n    { \"account_id\": ",
  ", \"balance\": ",
  " }, \"account_id\": ",
  ", \"balance\": ",
  " }, \"account_id\": ",
  ", \"balance\": ",
  "}\n  ]}"
]
```

Using this setup, the verifier $$\mathcal{V}$$ checks the following conditions:

1. $$R'$$ is a syntactically valid JSON.
2. The following circuit holds:

$$
C(x: \{ [P_k]_{0 \le k \le n}, [Z_k]_{0 \le k < n}, [R] \}, w: \{ \{Z_k\}_{0 \le k < n} \}) : \\ 
[R] \stackrel{?}= [P_0 \| Z_0 \| \dots \|P_{n-1}\|Z_{n-1}\|P_n] \land \bigwedge_{k=0}^{n-1} [Z_k] \stackrel{?}= \mathsf{Commit}(Z_k)
$$

* $$[X]$$: commitment of $$X$$
* The commitment scheme used must support **homomorphism under string concatenation**.

In the above example, $$Z_3$$​ corresponds to the balance for `account_id = 2`, solving the ambiguity issue from [Method 4](deco.md#method-4-aes-cbc-hmac-optimization).

_<mark style="background-color:yellow;">TODO(chokobole):</mark> Add details on commitment schemes that support this homomorphic property._

#### Forgery Prevention Examples

If a malicious prover tampers with the response structure, the verification fails due to a **mismatch in the number of commitments or string layout**:

❌ **Missing fields**

```json
{
  "accounts": [
    { "account_id": "", "balance": "" },
    { "account_id": "", "balance": "" }
  ]
}
```

❌ **Sensitive data not redacted**

```json
{
  "accounts": [
    { "account_id": 1, "balance": 500 },
    { "account_id": "", "balance": "" },
    { "account_id": "", "balance": "" }
  ]
}
```

#### Final Circuit Structure

$$
C(x: \{ \bm{\sigma}, \bm{\hat{\sigma}}, s_{i-1}, s_i, \textcolor{red}{\{[P_k]\}_{0 \le k \le n}, \{[Z_k]\}_{0 \le k < n}, [R], Z_m} \}, w: \{ k^\mathsf{Enc}, B_i, \textcolor{red}{\{Z_k\}_{0 \le k < n}} \}) : \\ \bigwedge_{j=1025}^{1027} \hat{B}_j \stackrel{?}= \mathsf{AES\text{-}CBC.Encrypt}(k^\mathsf{Enc}, B_j) \land s_i \stackrel{?}= f_H(s_{i-1}, B_i) \land \textcolor{red}{\mathsf{isCorrectBlock}(B_j, Z_m) \land [R] \stackrel{?}= [P_0 \| Z_0 \| \dots \|P_{n-1}\|Z_{n-1}\|P_n] \land \bigwedge_{k=0}^{n-1} [Z_k] \stackrel{?}= \mathsf{Commit}(Z_k) \land  Z_m \ge 1000}
$$

* $$m$$: the expected index of the scalar vector $$\bm{Z}$$.

_<mark style="background-color:yellow;">TODO(chokobole)</mark>: The paper and Chainlink blog do not provide detailed implementation of how_ $$\mathsf{isCorrectBlock}$$ _is constructed._

## Conclusion

**DECO** enables **verifiable web data disclosure**, a capability that traditional TLS protocols do not support. It overcomes the inherent limitation of web data being **"private-by-design" and trapped at the origin**, using zero-knowledge proofs (ZK).

Its core contributions are as follows:

* Through a **Three-Party Handshake** architecture, DECO **securely splits the TLS session key (especially the MAC key)** between the prover $$\mathcal{P}$$ and verifier $$\mathcal{V}$$, preventing the prover from forging TLS responses.
* It is designed to **verify both TLS message integrity and application-level data predicates inside a ZK circuit**, enabling selective and private disclosure.
* DECO leverages the [**Merkle–Damgård**](https://en.wikipedia.org/wiki/Merkle%E2%80%93Damg%C3%A5rd_construction) **structure of SHA-256**, the **MAC-then-Encrypt construction**, and **parsing-aware validation techniques** to minimize 2-Party Computation (2PC) overhead and optimize ZK proof efficiency.
* The **context integrity** problem for DECO is solved through grammar-aware parsing and enabling **position binding** proofs in ZK—for example, proving a specific value comes from a structured key-value JSON field without revealing the entire context.

## References

* [https://arxiv.org/pdf/1909.00938](https://arxiv.org/pdf/1909.00938)
* [https://blog.chain.link/deco-introduction/](https://blog.chain.link/deco-introduction/)
* [https://blog.chain.link/deco-provenance-and-authenticity/](https://blog.chain.link/deco-provenance-and-authenticity/)
* [https://blog.chain.link/deco-parsing-the-response/](https://blog.chain.link/deco-parsing-the-response/)
* [https://blog.chain.link/hiding-secret-lengths/](https://blog.chain.link/hiding-secret-lengths/)

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
