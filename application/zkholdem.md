---
description: 'Presentation: https://www.youtube.com/watch?v=vQ5-Sn2dHFE'
---

# zkHoldem

## Introduction

[zkHoldem](https://www.zkholdem.xyz/) is a Texas Hold'em game built on [zkShuffle](https://zkholdem.xyz/wp-content/themes/zkholdem-theme/zkshuffle.pdf) and launched on [Manta Network](https://manta.network/).

The subtitle of zkShuffle is **"Mental Poker on SNARK for Ethereum."** The term _Mental Poker_ was introduced in a [1979 paper](https://people.csail.mit.edu/rivest/pubs/SRA81.pdf) by Adi Shamir, Ron Rivest, and Leonard Adleman. It explores whether a fair poker game can be played online without physical cards while ensuring the following three principles:

1. Hide the card values from all players
2. Ensure that the cards are dealt correctly, with fair randomness
3. Reveal a card to a specific player or group of players

## Background

Please read [ElGamal Encryption](../primitives/encryption-scheme/elgamal-encryption.md) before continuing on.

## Protocol Explanation

### Overview of zkShuffle

zkShuffle is a protocol that distributes and shuffles cards among players. It primarily follows the [_Barnett and Smart_](https://archive.cone.informatik.uni-freiburg.de/teaching/teamprojekt/dog-w10/literature/mentalpoker-revisited.pdf) structure, which has also been improved and implemented by Geometry.

The key **novelty** of zkHoldem compared to Geometry's implementation is that it utilizes **Groth16** to implement the **shuffle argument** more efficiently. For further details, interested readers can refer to the [Geometry blog](https://geometry.xyz/notebook/mental-poker-in-the-age-of-snarks-part-1).

### **Modified ElGamal Encryption**

#### $$\mathsf{ModifiedElGamal.Setup}() \rightarrow (\mathsf{sk}, \mathsf{pk})$$

* Randomly select a **secret key** $$\mathsf{sk}$$ in the range $$1 \leq \mathsf{sk} < q-1$$.
* Compute the **public key** $$\mathsf{pk}$$ as follows:

$$
\mathsf{pk} = \mathsf{sk} \cdot G
$$

where $$G$$ is a generator of an additive group of a large prime size.

#### $$\mathsf{ModifiedElGamal.Encrypt}(m_1: \mathbb{G}, m_2: \mathbb{G}, \mathsf{pk}, r: \mathbb{F}) \rightarrow (c_1: \mathbb{G}, c_2: \mathbb{G})$$

* When called for the first time, $$m_1 = 0$$ and $$m_2 = m$$, the message. This value is used for nested encryption.
* A random scalar $$r$$ where $$1 \leq r < q-1$$ is used to determine the shared secret $$s$$.

$$
s = r \cdot \mathsf{pk}
$$

* Generate the ciphertext $$(c_1, c_2)$$ as follows:

$$
c_1 = m_1 + r\cdot G, \quad c_2 = m_2 + r \cdot \mathsf{pk}
$$

#### $$\mathsf{ModifiedElGamal.Decrypt}(c_1: \mathbb{G}, c_2: \mathbb{G}, \mathsf{sk}: \mathbb{F}, r: \mathbb{F}) \rightarrow m: \mathbb{G}$$

* $$c_1$$ is computed using the value $$r$$ that was used when calling $$\mathsf{ModifiedElGamal.Encrypt}$$. Note that this $$c_1$$ computed individually by each party is different from the iterative $$c_1$$ from $$\mathsf{.Encrypt}$$.

$$
c_1 = r \cdot G
$$

* Compute the plaintext $$m$$ as follows:

$$
m =  c_2 - \mathsf{sk} \cdot c_1
$$

### **Properties of Modified ElGamal**

Consider a scenario where Alice, Bob, and Carol each have private keys $$\mathsf{sk}_0, \mathsf{sk}_1, \mathsf{sk}_2$$ and their corresponding public keys $$\mathsf{pk}_0, \mathsf{pk}_1, \mathsf{pk}_2$$​. Now, let's examine the encryption process for a card $$m$$.

1. **Alice encrypts using** $$r_0$$$$(m_1 = 0)$$**:**

$$
\mathsf{ModifiedElGamal.Encrypt}(0, m, \mathsf{pk}_0, r_0) \rightarrow (r_0 \cdot G, m + r_0 \cdot \mathsf{pk}_0)
$$

2. **Bob encrypts using** $$r_1$$**​:**

$$
\mathsf{ModifiedElGamal.Encrypt}(r_0 \cdot G, m + r_0 \cdot \mathsf{pk}_0, \mathsf{pk}_1, r_1) \rightarrow ((r_0 + r_1) \cdot G, m + r_0 \cdot \mathsf{pk}_0 + r_1 \cdot \mathsf{pk}_1)
$$

3. **Carol encrypts using** $$r_2$$**​:**

$$
\mathsf{ModifiedElGamal.Encrypt}((r_0 + r_1) \cdot G, m + r_0 \cdot \mathsf{pk}_0 + r_1 \cdot \mathsf{pk}_1, \mathsf{pk}_2, r_2) \rightarrow ((r_0 + r_1 + r_2) \cdot G, m + r_0 \cdot \mathsf{pk}_0 + r_1 \cdot \mathsf{pk}_1 + r_2 \cdot \mathsf{pk}_2)
$$

After this process, the final ciphertext $$\bm{c}$$ is generated as follows:

$$
\bm{c} = ((r_0 + r_1 + r_2) \cdot G, m + r_0 \cdot \mathsf{pk}_0 + r_1 \cdot \mathsf{pk}_1 + r_2 \cdot \mathsf{pk}_2)
$$

At this stage, Alice, Bob, and Carol can **decrypt in any order** to retrieve the original message $$m$$. Suppose they decrypt in the order **Bob → Carol → Alice**:

1. **Bob decrypts using** $$r_1$$**​:**

$$
\mathsf{ModifiedElGamal.Decrypt}(r_1 \cdot G, m + r_0 \cdot \mathsf{pk}_0 + r_1 \cdot \mathsf{pk}_1 + r_2 \cdot \mathsf{pk}_2, \mathsf{sk}_1, r_1) \rightarrow m +  r_0 \cdot \mathsf{pk}_0 + r_2 \cdot \mathsf{pk}_2
$$

2. **Carol decrypts using** $$r_2$$**​:**

$$
\mathsf{ModifiedElGamal.Decrypt}(r_2 \cdot G, m + r_0 \cdot \mathsf{pk}_0 + r_2 \cdot \mathsf{pk}_2, \mathsf{sk}_2, r_2) \rightarrow m + r_0 \cdot \mathsf{pk}_0
$$

3. **Alice decrypts using** $$r_0$$**​:**

$$
\mathsf{ModifiedElGamal.Decrypt}(r_0 \cdot G, m + r_0 \cdot \mathsf{pk}_0, \mathsf{sk}_0, r_0) \rightarrow m
$$

#### **Advantages of Modified ElGamal**

One of the key features of Modified ElGamal is its **order-independent decryption** property.

This is **particularly useful in blockchain environments**, where:

* Decryption results need to be submitted as **on-chain transactions (tx)**.
* The ordering of transactions within a block is unpredictable.

Since decryption can be performed in **any order**, this significantly enhances **the usability of the protocol in decentralized systems**.

### **Shuffling**

In Texas Hold'em, a deck of **52 cards** must be shuffled, which needs to be mathematically represented. For example, suppose we start with a set of four cards:

$$
\bm{C_0} = (c_0, c_1, c_2, c_3)
$$

If Alice wants to shuffle them into the following order:

$$
\bm{C_1} = (c_0, c_2, c_1, c_3)
$$

how can we express this mathematically?

This can be represented using a [**permutation matrix**](https://en.wikipedia.org/wiki/Permutation_matrix) as a matrix multiplication:

$$
\bm{C_1} =\begin{bmatrix} c_0 \\c_2 \\c_1 \\c_3 \\\end{bmatrix} =\begin{bmatrix} 1 & 0 & 0 & 0  \\0 & 0 & 1 & 0  \\0 & 1 & 0 & 0  \\0 & 0 & 0 & 1  \\\end{bmatrix} \cdot\begin{bmatrix} c_0 \\c_1 \\c_2 \\c_3 \\\end{bmatrix}
$$

#### **Conditions for a Permutation Matrix**

A matrix $$A$$ is a **permutation matrix** if it satisfies the following three conditions:

1. $$A$$ **must be a square matrix.**

$$
\mathsf{IsSquare}(A) := M.\mathsf{rows} \stackrel{?}{=} M.\mathsf{cols}
$$

2. **All elements of** $$A$$ **must be either 0 or 1.**

$$
\mathsf{HasOnlyBinaryElements}(A) := \bigwedge_{i=0}^{\mathsf{rows}-1} \bigwedge_{j=0}^{\mathsf{cols}-1} \mathsf{IsBinary}(A_{i,j})
$$

The function $$\mathsf{IsBinary}$$ is defined as:

$$
\mathsf{IsBinary}(A_{i,j}) := A_{i, j} \stackrel{?}{=} 0 \lor A_{i, j} \stackrel{?}{=} 1
$$

3. **Each row and each column of** $$A$$ **must sum to 1.**

$$
\mathsf{HasValidRowAndColSums}(A) := \bigwedge_{i=0}^{\mathsf{rows}-1} \mathsf{IsRowSumOne}(A, i) \land \bigwedge_{j=0}^{\mathsf{cols}-1} \mathsf{IsColSumOne}(A, j)
$$

The function $$\mathsf{IsRowSumOne}$$ is defined as:

$$
\mathsf{IsRowSumOne}(A, i) := \sum_{j=0}^{\mathsf{cols}-1}A_{i,j} \stackrel{?}{=} 1
$$

The function $$\mathsf{IsColSumOne}$$ is defined as:

$$
\mathsf{IsColSumOne}(A, j) := \sum_{i=0}^{\mathsf{rows}-1}A_{i,j} \stackrel{?}{=} 1
$$

Thus, the function $$\mathsf{IsPermutationMatrix}$$ can be defined as:

$$
\mathsf{IsPermutationMatrix}(A) := \mathsf{IsSquare}(A) \land \mathsf{HasOnlyBinaryElements}(A) \land \mathsf{HasValidRowAndColSums}(A)
$$

### **zkShuffle Scheme**

Consider a scenario where $$k$$ players $$\bm{P} = \{p_0, \dots, p_{k-1}\}$$ participate, and there are $$n$$ cards in the game. Each player $$p_i$$ is responsible for shuffling an encrypted deck $$\bm{C_i} = (c_{i,0}, \dots, c_{i,n-1}) \in (\mathbb{G} \times \mathbb{G})^n$$. Here, $$n$$ is $$52$$.

Where:

* $$\mathbb{G}$$ is an **elliptic curve group**, such as Baby Jubjub.
* $$\mathbb{F}$$ is the **scalar field** of the elliptic curve.
* $$G$$ is the **generator** of the elliptic curve group.

All cards are given publicly known values, for example:

$$
\bm{C_0} = ((0, 0), (0, G), (0, 2 \cdot G), \dots, (0, (n-1) \cdot G))
$$

Thus, for $$0 \leq j < n$$, if $$j \cdot G$$ is given, the corresponding card can be determined.

#### $$\mathsf{zkShuffle.Setup}$$

* Each player $$p_i$$ randomly generates a **secret key** $$\mathsf{sk}_i \in \mathbb{F}$$.
* The **public key** $$\mathsf{pk}_i \in \mathbb{G}$$ is computed as follows:

$$
\mathsf{pk}_i = \mathsf{sk}_i \cdot G
$$

#### $$\mathsf{zkShuffle.ShuffleEncrypt}$$

<figure><img src="../.gitbook/assets/image (87) (1).png" alt=""><figcaption></figcaption></figure>

* Player $$p_i$$ generates a **random permutation matrix** $$\bm{A_i} \in \mathbb{F}^{n\times n}$$.
* Upon receiving deck $$\bm{C_i} = (c_{i,0}, \dots, c_{i,n-1})$$ from player $$p_{i-1}$$, the shuffled deck $$\bm{S_{i+1}} = (s_{i+1, 0}, \dots , s_{i+1, n-1})$$ is computed as follows:

$$
\bm{S_{i+1}} = \bm{A_i} \cdot \bm{C_i}
$$

* Player $$p_i$$ generates random values $$r_{i,j} \in \mathbb{F}$$ (for $$0 \le j < n$$).
* Player $$p_i$$ encrypts the shuffled deck $$\bm{S_{i+1}}$$ into $$\bm{C_{i+1}}$$ where each card is calculated as follows (for $$0 \le j < n$$):

$$
c_{i+1,j} = \mathsf{ModifiedElGamal.Encrypt}(s_{i+1,j}[0], s_{i+1,j}[1], \mathsf{pk}_i, r_{i,j})
$$

* Player $$p_i$$ generates a **ZK-proof** for the following circuit and submits it along with $$\bm{C}_{i+1}$$ in a transaction:

$$
C_1(x: \{ \bm{C_i}, \mathsf{pk}_i \}, w: \{ \bm{A_i}, \bm{S_{i+1}, \bm{r_i}} \}): \\ 
\mathsf{IsPermutationMatrix}(\bm{A}_i) \land \bm{S_{i+1}} \stackrel{?}= \bm{A_i} \cdot \bm{C_i} \land \bigwedge_{j=0}^{51}\left( c_{i+1, j}\stackrel{?}=\mathsf{ModifiedElGamal.Encrypt}(s_{i+1,j}[0], s_{i+1,j}[1], \mathsf{pk}_i, r_{i,j})\right)
$$

#### $$\mathsf{zkShuffle.Decrypt}$$

<figure><img src="../.gitbook/assets/image (1) (4).png" alt=""><figcaption></figcaption></figure>

* Each player $$p_i$$ decrypts a received encrypted card $$c: \mathbb{G} \times \mathbb{G}$$ to obtain $$m: \mathbb{G}$$ as follows (The reason shuffling does not need to be considered during decryption is that if we assume the cards were shuffled using $$\mathsf{zkShuffle.ShuffleEncrypt}$$, the final recovered message ($$m$$) will always be one of the elements in $$\bm{C_0}$$.):

$$
m = \mathsf{ModifiedElGamal.Decrypt}(c[0], c[1], \mathsf{sk}_i, r)
$$

#### $$\mathsf{zkShuffle.DecryptPost}$$

* Each player $$p_i$$ decrypts an encrypted card $$c: \mathbb{G} \times \mathbb{G}$$ to obtain $$m: \mathbb{G}$$:

$$
m = \mathsf{ModifiedElGamal.Decrypt}(c[0], c[1], \mathsf{sk}_i, r)
$$

* The player generates a **ZK-proof** for the following circuit and submits it along with $$m$$ in a transaction:

$$
C_2(x: \{ c, m \}, w: \{ \mathsf{sk}_i , r\}): \\ 
m \stackrel{?}=\mathsf{ModifiedElGamal.Decrypt}(c[0], c[1], \mathsf{sk}_i, r)
$$

* Since the contract already has the shuffled encrypted cards, it knows which card is at the top of the deck and which card needs to be decrypted. It verifies whether the public input corresponds to the correct card for decryption.

### **Game Flow**

1. **Shuffling Cards**
   1. Cards are given publicly known values.
   2. All players shuffle the cards using $$\mathsf{zkShuffle.ShuffleEncrypt}$$.
2. **Distributing Two Cards to Each Player**
   1. Cards are dealt from the top of the deck to players in a predetermined order.
   2. **For all dealt cards that do not belong to them, players call** $$\mathsf{zkShuffle.DecryptPost}$$.
   3. After step **b** is finished by every player, **players reveal their own cards to themselves** with $$\mathsf{zkShuffle.Decrypt}$$.
3. **Revealing Three Community Cards**
   1. The top three cards from the deck are placed on the table.
   2. All players call $$\mathsf{zkShuffle.DecryptPost}$$ for these cards, revealing them.
4. **Revealing Additional Community Cards (Two Rounds)**
   1. One additional card from the top of the deck is placed on the table in each round.
   2. All players call $$\mathsf{zkShuffle.DecryptPost}$$ for these cards, revealing them.
5. **Revealing Player Cards**
   1. Players who remain until the end reveal their cards using $$\mathsf{zkShuffle.DecryptPost}$$.

### **How Are the Three Principles Ensured?**

1. **Hide the card values from all players.**
   1. Cards are encrypted using $$\mathsf{zkShuffle.ShuffleEncrypt}$$. Therefore, unless all players participate in decryption, they cannot access the original card value.
2. **Ensure that the cards are dealt correctly, with fair randomness.**
   1. Before the game starts, all players contribute randomness to shuffle the cards. Even if some players collude, the deck remains randomly shuffled.
   2. $$\mathsf{zkShuffle.ShuffleEncrypt}$$ ensures that all players have encrypted the shuffled cards according to the shuffling rules using ZK.
   3. From the shuffle ZK proof, the contract knows the order of the deck.
   4. If a player attempts to decrypt a card that is not at "the top of the deck", the contract will reject the request.
3. **Reveal a card to a specific player or group of players.**
   1. **Revealing to one player**
      1. A player's card is revealed to them when calling $$\mathsf{zkShuffle.Decrypt}$$ after all other players have run $$\mathsf{zkShuffle.DecryptPost}$$.
      2. Since $$\mathsf{zkShuffle.Decrypt}$$ does not submit the result to the contract, other players cannot see their cards.
   2. **Revealing to all players**
      1. Once every player calls $$\mathsf{zkShuffle.DecryptPost}$$ on a card, it is revealed.

## **Conclusion**

The **Circom code** for zkHoldem is private, but according to the paper:

* $$\mathsf{zkShuffle.DecryptPost}$$ includes **3,500 constraints**.
* $$\mathsf{zkShuffle.ShuffleEncrypt}$$ includes **170,000 constraints**.

Using **ZK for online Texas Hold'em** is an exciting concept.

However, there are some limitations:

1. **Receiving a single card requires multiple interactions with other players for decryption.**
2. **Even if a player folds, they must stay online to complete the decryption process.**

These challenges highlight key areas for improvement in real-world applications.

## References

* [https://zkholdem.xyz/wp-content/themes/zkholdem-theme/zkshuffle.pdf](https://zkholdem.xyz/wp-content/themes/zkholdem-theme/zkshuffle.pdf)
* [https://geometry.xyz/notebook/mental-poker-in-the-age-of-snarks-part-1](https://geometry.xyz/notebook/mental-poker-in-the-age-of-snarks-part-1)
* [https://geometry.xyz/notebook/mental-poker-in-the-age-of-snarks-part-2](https://geometry.xyz/notebook/mental-poker-in-the-age-of-snarks-part-2)

> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
