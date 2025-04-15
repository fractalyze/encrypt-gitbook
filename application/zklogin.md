---
description: 'Presentation: https://www.youtube.com/watch?v=vVMS1HbDoEM'
---

# zkLogin

## Introduction

In the traditional Web3 environment, users must manage their mnemonics directly, which poses a challenge since losing or having them stolen makes recovery difficult. Additionally, when users switch devices, they must restore their accounts using their self-managed mnemonics. In contrast, [**zkLogin from Sui**](https://sui.io/zklogin) introduces an OAuth authentication method leveraging ZK to overcome the limitations of conventional Web3 login mechanisms and enhance the user experience. Notably, [**Aptos' Keyless Wallet**](https://aptos.dev/en/build/guides/aptos-keyless) operates in a similar manner.

## Background

### OIDC (OpenID Connect)

[OAuth2](https://oauth.net/2/) is a protocol that grants applications **permission** to access a user's data on their behalf. For example, to obtain file access permissions for Google Drive with OAuth2, the following steps are taken:

1. The user attempts to access Google Drive files through an application (MyApp).
2. MyApp requests user authorization from Google.
3. The user logs in to Google and grants permission.
4. Google issues an Access Token to MyApp.
5. MyApp sends the Access Token to the Google Drive API to access the files.

On the other hand, [OIDC](https://openid.net/developers/how-connect-works/) is a protocol based on OAuth2 that verifies a user's **identity**. In other words, it determines, "Is this person really who they claim to be?" For instance, when logging in using a Google account, the following steps occur:

1. The user clicks the "Login with Google" button on MyApp.
2. MyApp sends a login request to Google.
3. The user completes the Google login process.
4. Google issues an ID Token to MyApp.
5. MyApp verifies the ID Token and processes the user's login.

Entities such as Google, Facebook, and GitHub that provide this protocol are called **OIDC Providers**. The issued **ID Token** is known as a **JWT (JSON Web Token)**.

### **JWT Example**

To run the following script, you need to install the required packages locally:

```bash
pip install cryptography pyjwt 
```

```python
import jwt
import base64
import json
import time
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization

# Example JWT payload with nonce
payload = {
    "iss": "https://accounts.google.com",
    "sub": "1234567890",
    "aud": "4074087",
    "iat": int(time.time()),
    "exp": int(time.time()) + 3600,  
    "name": "Ryan Kim",
    "email": "ryan@a41.io",
    "nonce": "random_nonce_value"  
}

# Generate a new RSA private key
private_key_obj = rsa.generate_private_key(
    public_exponent=65537,
    key_size=2048
)

# Serialize the private key to PEM format
private_key_pem = private_key_obj.private_bytes(
    encoding=serialization.Encoding.PEM,
    format=serialization.PrivateFormat.TraditionalOpenSSL,
    encryption_algorithm=serialization.NoEncryption()
).decode()

# Serialize the public key to PEM format
public_key_pem = private_key_obj.public_key().public_bytes(
    encoding=serialization.Encoding.PEM,
    format=serialization.PublicFormat.SubjectPublicKeyInfo
).decode()

# Generate JWT with RS256 signature using the new private key
jwt_signed = jwt.encode(payload, private_key_pem, algorithm="RS256")

print(jwt_signed)
```

You can decode the JWT at [https://www.jstoolset.com/jwt](https://www.jstoolset.com/jwt), which will display the following structure:

<figure><img src="../.gitbook/assets/Screenshot 2025-03-08 at 10.42.10 PM.png" alt=""><figcaption></figcaption></figure>

Each field in the JWT has the following meaning:

* **`iss` (issuer)**: Represents the entity (Identity Provider, IdP) that issued the JWT. In this case, it indicates that Google issued the token.
* **`sub` (subject)**: The unique ID of the authenticated user. Providers like Google, Twitch, and Slack use the same `sub` across all `aud` values. However, Apple, Facebook, and Microsoft generate a unique `sub` for each `aud`.
* **`aud` (audience)**: Specifies the target service (client ID) for which this JWT is intended.
* **`iat` (issued at)**: The timestamp (Unix timestamp in seconds) when the JWT was issued.
* **`exp` (expiration)**: The expiration timestamp (Unix timestamp in seconds) after which the JWT becomes invalid.
* **`nonce`**: A random value used to prevent replay attacks by ensuring request and response integrity.

Then, the OIDC Provider signs the JSON data using the following method:

$$
\mathsf{JWT.Issue}(\mathsf{sk_{OIDC}, claim}) \rightarrow \mathsf{jwt}
$$

where $$\mathsf{sk_{OIDC}}$$ is the secret key of the OIDC Provider, $$\mathsf{claim} = \{\mathsf{iss, sub, aud, iat, exp, nonce, \dots}\}$$ represents the claims included in the $$\mathsf{jwt}$$, and $$\mathsf{jwt}$$ is the signed token that contains both $$\mathsf{claim}$$ and the signature $$\sigma_{\mathsf{OIDC}}$$.

On the client side, the generated $$\mathsf{jwt}$$ can be verified as follows:

$$
\mathsf{JWT.Verify}(\mathsf{pk_{OIDC}, jwt}) \rightarrow b \in \{0, 1\}
$$

This verification process can be performed by a blockchain validator. Here, the `nonce` can be determined by the audience, i.e., the wallet application, and we aim to leverage this mechanism to improve the existing login process.

For further details on why OIDC was chosen over alternatives such as Passkey, MPC, and HSM, interested readers can refer to [AIP-61](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md#alternative-solutions).

## Protocol Explanation

### Transaction Signature with JWT

To reiterate, our goal is to ensure that users do not need to remember anything. This means we need to replace the traditional method of signing transactions with a private key derived from a mnemonic and the way we generate an on-chain address from it. The new approach should be both efficient and secure, ensuring that the OIDC Provider does not introduce security risks.

We explore various approaches to achieve this goal. The following explanation is inspired by [the method presented by Alin Tomescu, Head of Cryptography at Aptos, at ZKSummit12](https://www.youtube.com/watch?v=sKqeGR4BoI0), as it provides a clear and intuitive understanding.

#### Method 1: Signing Transactions Using OIDC

The first approach is to embed the transaction inside the `nonce`. By doing so, users can sign transactions via Google login (or any OIDC protocol) instead of generating a private key from a mnemonic. The user's on-chain address can then be derived as follows:

$$
\mathsf{addr} = H(\mathsf{iss \| sub \| aud})
$$

Refer to [Appendix E](https://arxiv.org/pdf/2401.11735#page=20) of the [**zkLogin paper**](https://arxiv.org/pdf/2401.11735) for the reason why `aud` is included.

However, this approach has a drawback: the validator can infer the relationship between `sub` and $$\mathsf{addr}$$, potentially exposing user identity information.

#### Method 2: Introducing ZK

To address the issue from **Method 1**, we can design a ZK circuit that hides `sub` while still ensuring its validity.

$$
C(x: \{\mathsf{addr, tx, pk_{OIDC}}\}, w: \{ \mathsf{jwt} \}): \\H(\mathsf{jwt.iss \| jwt.sub \| jwt.aud}) \stackrel{?}= \mathsf{addr} \land \mathsf{JWT.Verify}(\mathsf{pk_{OIDC}, jwt}) \stackrel{?}=1 \land \mathsf{jwt.nonce} \stackrel{?}= \mathsf{tx}
$$

Here, we use the notation from [here](../introduction/notations-and-definitions/zk.md#circuit).

However, this approach still has a flaw: the OIDC Provider can infer **the relationship between `sub` and** $$\mathsf{addr}$$**, which could compromise user privacy.**

#### Method 3: Adding a `salt` to Address Derivation

To further improve privacy, we introduce a $$\mathsf{salt}$$ when deriving the user’s on-chain address:

$$
\mathsf{addr} = H(\mathsf{iss \| sub \| aud \| \textcolor{red}{salt}})
$$

With this modification, unless the OIDC Provider knows the $$\mathsf{salt}$$, it cannot link the user's `sub` to their on-chain activity. The only requirement is ensuring that the user does not lose their $$\mathsf{salt}$$. However, since our goal is for users to remember nothing, we need a mechanism that allows only the user to generate their $$\mathsf{salt}$$ securely. (This document does not cover how to achieve this—refer to [AIP-81](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-81.md) or [Section **4.3.1**](https://arxiv.org/pdf/2401.11735#page=9\&zoom=100,424,735) of the [**zkLogin paper**](https://arxiv.org/pdf/2401.11735) for details.)

TODO(chokobole): add how to ensure that only the user can derive their salt.

Next, we modify the ZK circuit accordingly:

$$
C(x: \{\mathsf{addr, tx, pk_{OIDC}} \}, w: \{ \mathsf{jwt, \textcolor{red}{salt}} \}): \\H(\mathsf{jwt.iss \| jwt.sub \| jwt.aud \| \textcolor{red}{salt}}) \stackrel{?}= \mathsf{addr} \land \mathsf{JWT.Verify}(\mathsf{pk_{OIDC}, jwt}) \stackrel{?}=1 \land \mathsf{jwt.nonce} \stackrel{?}= \mathsf{tx}
$$

However, this approach presents **two major challenges**:

1. **Performance Overhead**: A new ZK proof must be generated for every transaction, and validators must verify these proofs, which adds computational cost.
2. **Potential for Abuse**: Since the OIDC Provider holds the signing private key for transactions, it could be exploited maliciously.

#### Method 4: Signing with an Ephemeral Key

Now, we introduce an **ephemeral key pair** $$(\mathsf{esk}, \mathsf{epk})$$. The transaction is signed using this ephemeral key pair, allowing users to **regenerate** their key pair via Google login or another OIDC provider if they ever lose it. This ensures that users **do not need to remember anything**.

Additionally, by **sending the proof** that includes the new ephemeral public key $$\mathsf{epk}$$, the user **updates their ephemeral key** on-chain, ensuring the system recognizes the new key for future transactions.

$$
\mathsf{Sig.Sign}(\mathsf{esk, tx}) \rightarrow \sigma_{\mathsf{tx}} \\ \mathsf{Sig.Verify}(\mathsf{epk, tx}, \sigma_{\mathsf{tx}}) \rightarrow b \in \{0, 1\}
$$

Next, we embed $$\mathsf{epk}$$ into the `nonce` and modify the ZK circuit accordingly:

$$
C(x: \{\mathsf{addr, \textcolor{red}{epk}, pk_{OIDC}} \}, w: \{ \mathsf{jwt, salt} \}): \\ H(\mathsf{jwt.iss \| jwt.sub \| jwt.aud \| salt}) \stackrel{?}= \mathsf{addr} \land \mathsf{JWT.Verify}(\mathsf{pk_{OIDC}, jwt}) \stackrel{?}=1 \land \mathsf{jwt.nonce} \stackrel{?}= \textcolor{red}{\mathsf{epk}}
$$

**Flow Overview**

1. **Key Generation:** The user generates an ephemeral key pair $$(\mathsf{esk}, \mathsf{epk})$$.
2. **Proof Creation:** The user constructs a **ZK proof** $$\pi$$ proving ownership of their identity.
3. **Tx Submission:** The user signs a transaction $$\mathsf{tx}$$ and signature using $$\mathsf{esk}$$, producing $$\sigma_{\mathsf{tx}}$$.
4. **Submission & Key Update:** The user submits $$(\pi, \mathsf{epk}, \mathsf{tx}, \sigma_{\mathsf{tx}})$$, to the validator. Upon verification, the validator may **update the user's ephemeral public key (**$$\mathsf{epk}$$**) on-chain**.
5. **Subsequent Transactions:** Until the **ephemeral key expires**, the user only needs to submit a new transaction $$\mathsf{tx}'$$ and its signature $$\sigma_{\mathsf{tx'}}$$ to the validator, without regenerating a new proof.

This approach has several **advantages**:

* **Reduced Proof Overhead**: A ZK proof only needs to be generated when updating the ephemeral key pair, reducing computation costs.
* **Enhanced Security**: Users create the signing private key themselves, preventing abuse by the OIDC Provider.

However, one issue remains: the OIDC Provider can still infer **which transactions were signed by a given `sub` through the** $$\mathsf{epk}$$**.**

#### Method 5: Adding Randomness to the `nonce`

A simple solution, such as hashing $$\mathsf{epk}$$ before embedding it into `nonce`, is **insufficient** because $$\mathsf{epk}$$ is public. An adversary could still **brute-force** the mapping between $$\mathsf{epk}$$ and `sub`.

To prevent this, we introduce randomness $$\mathsf{r}$$ and modify the `nonce` by hashing both $$\mathsf{epk}$$ and $$\mathsf{r}$$:

$$
C(x: \{\mathsf{addr, epk, pk_{OIDC}} \}, w: \{ \mathsf{jwt, salt, \textcolor{red}{r}} \}): \\ H(\mathsf{jwt.iss \| jwt.sub \| jwt.aud \| salt}) \stackrel{?}= \mathsf{addr} \land \mathsf{JWT.Verify}(\mathsf{pk_{OIDC}, jwt}) \stackrel{?}=1 \land \mathsf{jwt.nonce} \stackrel{?}= \textcolor{red}{H(\mathsf{epk \| r})}
$$

Now, the OIDC Provider can no longer determine which transactions a given `sub` has signed, ensuring **stronger privacy protection**.

#### Method 6: Updatable Ephemeral Key Expiration

Typically, JWTs are valid for **one hour**, which is too short in the **blockchain domain** and inconvenient due to its reliance on real-time expiration. Instead, we extend the validity based on **block height** rather than real-world time.

For example, if we want the key to be valid for **10 hours** and the blockchain's **block time** is **10 minutes**, we can set:

$$
\mathsf{exp} = \mathsf{block.cur} + 60
$$

We then update the `nonce` with:

$$
\mathsf{nonce} =H(\mathsf{epk \| exp \| r})
$$

Additionally, the validator must check that:

$$
\mathsf{block.cur} < \mathsf{exp}
$$

The updated ZK circuit is as follows:

$$
C(x: \{\mathsf{addr, epk, \textcolor{red}{exp}, pk_{OIDC}} \}, w: \{ \mathsf{jwt, salt, r} \}): \\ H(\mathsf{jwt.iss \| jwt.sub \| jwt.aud \| salt}) \stackrel{?}= \mathsf{addr} \land \mathsf{JWT.Verify}(\mathsf{pk_{OIDC}, jwt}) \stackrel{?}=1 \land \mathsf{jwt.nonce} \stackrel{?}= H(\mathsf{epk \| \textcolor{red}{exp} \| r})
$$

#### Method 7: Adding `iss` to Public Input

Since $$\mathsf{pk_{OIDC}}$$ (OIDC Provider's public key) **changes periodically**, we add `iss` to the **public input**. The validator must then verify that $$\mathsf{pk_{OIDC}}$$ indeed corresponds to the public key of `iss`.

The ZK circuit is further modified as:

$$
C(x: \{\mathsf{addr, epk, exp, \textcolor{red}{iss}, pk_{OIDC}} \}, w: \{ \mathsf{jwt, salt, r} \}): \\ H(\mathsf{jwt.iss \| jwt.sub \| jwt.aud \| salt}) \stackrel{?}= \mathsf{addr} \land \mathsf{JWT.Verify}(\mathsf{pk_{OIDC}, jwt}) \stackrel{?}=1 \land \mathsf{jwt.nonce} \stackrel{?}= H(\mathsf{epk \| exp \| r}) \land \textcolor{red}{\mathsf{jwt.iss \stackrel{?}= iss}}
$$

Additionally, the validator must check that:

$$
\mathsf{isValidPublicKey(iss, \mathsf{pk_{OIDC}}, block.cur)} \stackrel{?}= 1
$$

The following figure, extracted from the [**zkLogin paper**](https://arxiv.org/pdf/2401.11735), provides an overview of the **entire login process**:

<figure><img src="../.gitbook/assets/Screenshot 2025-03-08 at 11.28.48 PM (1).png" alt=""><figcaption></figcaption></figure>

Through these methods, we successfully achieve the **initial goal** of **passwordless Web3 authentication** while maintaining **security and privacy**.

### Transaction Signature with nonce-less JWT

As mentioned in [**Section 4.5**](https://arxiv.org/pdf/2401.11735#page=11\&zoom=100,424,150) of the paper, unfortunately, according to the [OIDC specification 3.1.2.1](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest), the `nonce` field is **not mandatory**.

This raises an important question: **how can we implement zkLogin without a `nonce` in the JWT?**

To answer this, let's recall the role of `nonce`. It was used to **bind** $$\mathsf{epk, exp}$$ to the **JWT**. Thus, by modifying the circuit derived in **Method 7**, we can adapt zkLogin as follows:

$$
C(x: \{\mathsf{addr, epk, exp, iss, \textcolor{red}{h}, pk_{OIDC}} \}, w: \{ \mathsf{jwt, salt} \}): \\ H(\mathsf{jwt.iss \| jwt.sub \| jwt.aud \| salt}) \stackrel{?}= \mathsf{addr} \land \mathsf{JWT.Verify}(\mathsf{pk_{OIDC}, jwt}) \stackrel{?}=1 \land \textcolor{red}{\mathsf{h}} \stackrel{?}= H(\mathsf{epk \| exp \| \textcolor{red}{jwt}}) \land \mathsf{jwt.iss \stackrel{?}= iss}
$$

#### Why Is $$\mathsf{r}$$ No Longer Needed?

Previously, we introduced **randomness** $$\mathsf{r}$$ to prevent the link between **`sub` and** $$\mathsf{epk}$$ through `nonce`. However, in this case, since `nonce` **does not exist**, there is no longer a need for $$\mathsf{r}$$.

#### **Security Concern: Transaction Signature Manipulation**

One key issue arises with this approach.

* In **previous methods**, even if an attacker obtained a **valid** $$\mathsf{jwt}$$, they **could not forge** a transaction signature.
* However, in this case, since we **do not commit** $$\mathsf{epk}$$ via `nonce` an attacker who obtains a valid $$\mathsf{jwt}$$ could use it to generate a **different** ephemeral key pair $$(\mathsf{esk}, \mathsf{epk})$$ and **sign arbitrary transactions**.

This introduces a new **security vulnerability**, which must be addressed before deploying this approach. This could be mitigated to an extent through client-side proof generation, but still poses the same risk if the client themselves leak their valid own $$\mathsf{jwt}$$. Thus, since there is no current true solution to this vulnerability, transaction signatures with nonce-less JWT are not in use commercially today.

### Circuit Implementation

When an **OIDC Provider** signs a JWT, it typically follows this process:

$$
\mathsf{RSA.Sign}(\mathsf{sk_{OIDC}, SHA2(jwt)}) \rightarrow \sigma_{\mathsf{OIDC}}
$$

Thus, $$\mathsf{JWT.Verify}$$ must be implemented as follows:

$$
\mathsf{RSA.Verify(pk_{OIDC}, jwt.}{\sigma_{\mathsf{OIDC}}}\mathsf{,SHA2(jwt)}) \rightarrow b \in \{0, 1\}
$$

However, since $$\mathsf{jwt} = \mathsf{Base64.Encode(claim)}$$, accessing a specific value like $$\mathsf{jwt.x}$$ actually requires $$\mathsf{Base64.Decode(jwt).x}$$.

As a result, the **ZK circuit** must represent **RSA signatures**, **SHA-2 hashing**, and **Base64 decoding**—all of which are **not ZK-friendly**. This was implemented using [**circom**](https://docs.circom.io/), and it introduces the following computational overhead (Reference: [https://youtu.be/IyTQ2FfglFE?t=1473](https://youtu.be/IyTQ2FfglFE?t=1473)):

* **SHA-2 computation** – **74%**
* **RSA signature verification** – **15%** (Utilizes a trick from [the xJsnark paper](https://ieeexplore.ieee.org/document/8418647))
* **JSON parsing, Poseidon hashing, Base64 encoding, and additional constraints** – **11%**

Currently, the **Sui zkLogin** codebase is private. However, the implementation of **Aptos Keyless Wallet** is publicly available at 🔗 [Aptos Keyless ZK Proofs](https://github.com/aptos-labs/keyless-zk-proofs).

#### **JWT parsing**

One novel aspect of zkLogin is **JWT parsing**, with two key optimizations:

1. **Header Parsing Optimization**:
   * Since **JWT headers** are **public**, they are **decoded outside the circuit** to reduce computational overhead.
2. **Selective Payload Parsing**:
   * Based on the following assumptions:
     * The **OIDC Provider** adheres to the **JSON specification**.
     * The **required fields** (e.g., `sub`, `iss`, `aud`, `nonce`) are located at the **top level** of the JSON structure.
     * All **JSON values** are either **strings** (`sub`, `iss`, `aud`, `nonce`) or **booleans** (`email_verified`).
     * JSON **keys do not contain escaped quotes** (e.g., `\"sub\"` is not allowed).

Example JSON:

```json
{"sub":"123","aud":"mywallet","nonce":"ajshda"}
```

#### **Selective Payload Parsing Algorithm**

The function  $$\mathsf{SelectivePayloadParsing}(S, i, \ell, j) \rightarrow (k, v)$$ operates as follows. Assume $$S$$ **represents the above JSON structure**. For simplicity, **whitespace handling** and **boolean values** are omitted from the explanation.

1. Extract the substring $$S' := S[i: i +\ell]$$.&#x20;
   1. Example: If $$i = 1$$ and $$\ell = 12$$, then $$S'$$ becomes `"sub":"123",`.
2. Ensure $$S'$$ **ends** with either `,` or `}`:&#x20;

$$
S'[- 1] \stackrel{?}=\mathsf{Base64.Encode(,)} \lor S'[- 1] \stackrel{?}=\mathsf{Base64.Encode(\})}
$$

3. Verify that $$S'[j]$$ is `:`:&#x20;

$$
S'[j] \stackrel{?}= \mathsf{Base64.Encode(:)}
$$

4. Using column index $$j$$, extract:
   1. **Key**: $$k := S'[0:j]$$
   2. **Value**: $$v:=S'[j + 1:-1]$$
5. Ensure both $$k$$ and $$v$$ **start and end** with `"`:&#x20;

$$
k[0] \stackrel{?}= \mathsf{Base64.Encode(")} \land k[-1] \stackrel{?}= \mathsf{Base64.Encode(")} \land v[0] \stackrel{?}= \mathsf{Base64.Encode(")} \land v[-1] \stackrel{?}= \mathsf{Base64.Encode(")}
$$

#### **How to implement indexing operator in Circuit**

For a string $$S$$ of length $$n$$, where $$0 \le t < n$$, the value $$S[t]$$ can be computed as follows:

$$
S[t] = S_t = S\cdot O_t
$$

For $$0 \le t < n$$, the **indexing operator** $$O_t$$ is defined as:

$$
O_t(x) = \begin{cases}
1 & \text{ if } x = t \\
0 & \text{otherwise}
\end{cases}
$$

For example, to check whether $$S'$$ ends with `,`, we transform it into:

$$
S'[-1] = S'[\ell - 1] = S[i + \ell -1] = S_{i + \ell - 1} \stackrel{?}=\mathsf{Base64.Encode(,)}
$$

Similarly, extracting $$S[i: i + \ell]$$ follows:

$$
S[i: i + \ell] = \{S_t\}_{t \in \{i, \dots, i+\ell-1\}}
$$

Checking whether $$k$$ **starts with `"`** can be rewritten as:

$$
k[0] = S[i] = S_i \stackrel{?}= \mathsf{Base64.Encode(")}
$$

#### **Optimization via Packing**

Since computing $$S[t]$$ requires $$n$$ **multiplications**, extracting $$S[i:i+m]$$ requires $$n \times m$$ **constraints**. However, we can significantly **reduce constraints** by leveraging **field packing**:

* **JWT elements** are **8-bit values**.
* **BN254 scalar field** has a width of **253 bits**.
* This allows us to **pack 16 elements at once**, reducing constraints.

Even after adding **boundary checks** and **unpacking logic**, the **constraint count** drops from $$n \times m$$ **to** $$18m + \frac{n \times m}{32}$$, resulting in a **major efficiency gain**. (The paper doesn't explain how.)

### Proving Scheme

In addition to verifying transaction signatures using [**EdDSA**](https://en.wikipedia.org/wiki/EdDSA), as commonly used in **Sui** and **Aptos**, validators must now also verify **ZK proofs**.

Although a proof is generated **only once**, it is **verified multiple times** by validators. Given that ZK verification must not be significantly slower than traditional **EdDSA signature verification**, [**Groth16**](../zk/snark/groth16.md) was chosen as the proving scheme. Groth16 requires only **three pairing operations** for verification, making it computationally efficient. (There are additional computations using public inputs, but these are omitted for simplicity.)

One advantage of **Groth16** is that it allows [batch verification of $$n$$ **proofs simultaneously**](../zk/snark/groth16.md#batch-proof-verification). However, if at least **one proof is invalid**, all $$n$$ **proofs must be re-verified** individually, leading to a **worst-case scenario** that could be exploited as a **DDoS attack vector**.

In addition, **ZK proof verification** can be **optimized through engineering techniques**, such as **caching** frequently used proofs.

### Prover Service

> _"This drastically reduces the time between when a user logs in with their keyless account to when that user is able to transact, from \~25 seconds to 3 seconds. In turn, this greatly improves the user experience of keyless accounts."_\
> — [AIP-75](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-75.md)

According to **AIP-75**, generating a proof using [Method 7](zklogin.md#method-7-adding-iss-to-public-input) on the **client side** takes approximately **25 seconds**, which is unacceptable from a UX perspective. To address this, both **Sui** and **Aptos** have adopted a **prover service** to generate proofs on the server side.

Beyond performance, there are **security concerns** as well. If there were **bugs in circom or the ZK circuit**, users’ funds could be at risk. By centralizing proof generation on a **server**, such risks can be mitigated.

However, this approach introduces several **risks**:

* **Liveness Issue**: If the prover service goes **down**, users will be **unable to send transactions**.
* **Privacy Issue**: The prover service gains access to $$w: \{\mathsf{jwt, salt, r}\}$$, meaning it can infer the **relationship between `sub` and** $$\mathsf{addr}$$ (similar to the issue in **Method 1**). Fortunately, the prover **does not know** $$\mathsf{esk}$$, preventing transaction signature forgery.
* **Cost Issue**: If the prover service generates a **large number of proofs**, it incurs significant computational **costs**.
* **Centralization Issue**: Users must **trust** the prover service to be honest and reliable.

The image below, taken from the [**zkLogin paper**](https://arxiv.org/pdf/2401.11735), shows that **Sui** has a **server-side proof generation time** of around **3 seconds**, which is similar to Aptos's proof generation time mentioned above:

<figure><img src="../.gitbook/assets/Screenshot 2025-03-09 at 3.08.11 PM (1).png" alt=""><figcaption></figcaption></figure>

## Conclusion

Take a look at this timestamped video to see zkLogin in action.

{% embed url="https://www.youtube.com/watch?t=2127s&v=IyTQ2FfglFE" %}

zkLogin significantly improves **UX** by leveraging **OIDC**, allowing users to **avoid managing mnemonics** and other sensitive information. This innovation is a **step forward** toward **mass adoption** of Web3.

However, there are **trade-offs**:

* Web3 was initially created to **move beyond Web2**. Yet, zkLogin **relies on Web2 services** to enhance the **Web3 experience**, which feels somewhat contradictory.
* **Client-side proof generation** is crucial for fully decentralized authentication, but due to **performance constraints**, zkLogin currently relies on **server-side proof generation**, which is **not ideal**.
* **Validators must verify numerous ZK proofs**, which is why **Groth16** was chosen—it requires only **three pairing operations** per proof. However, instead of optimizing **proof generation**, both zkLogin and Keyless Wallet optimize **circuit design** for performance. This leads to an **inconvenience** in development since **each circuit update requires a new trusted setup**, posing a **DevEx** challenge.

Despite these limitations, **zkLogin represents a meaningful step toward improving Web3 usability**, and with further optimizations, it could **strike a better balance between decentralization, security, and user experience**.

## References

* [https://www.youtube.com/watch?v=IyTQ2FfglFE](https://www.youtube.com/watch?v=IyTQ2FfglFE)
* [https://www.youtube.com/watch?v=sKqeGR4BoI0](https://www.youtube.com/watch?v=sKqeGR4BoI0)
* [https://arxiv.org/abs/2401.11735](https://arxiv.org/abs/2401.11735)
* [https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md#the-keyless-zk-relation-mathcalr](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-61.md#the-keyless-zk-relation-mathcalr)

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
