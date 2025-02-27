---
description: 'Presentation: https://www.youtube.com/watch?v=-BT-Oyqy5vc'
---

# Yao's Garbled Circuits

Though Yao’s Garbled Circuits was first introduced in 1986, it still stands as the base for numerous multi-party computation (MPC) methods used today (i.e. [ABY3](https://eprint.iacr.org/2018/403.pdf)). We'll go over the original concept of Yao’s Garbled Circuits and introduce a couple popular optimizations done on top of this protocol. [Lúcás’s article on Yao’s Garbled Circuits](https://cronokirby.com/posts/2022/05/explaining-yaos-garbled-circuits/) and the [Mina book](https://o1-labs.github.io/proof-systems/fundamentals/zkbook_2pc/overview.html) with their reference links to papers served as valuable resources.

Now let’s get into it!

**Yao’s Garbled Circuits** is a 2 party MPC (2PC) method using only boolean gates. As the descriptor “MPC” suggests, Yao’s Garbled Circuits allow both parties to calculate a result without each party knowing the other’s private inputs. It works like this:

We split the two parties into the **Garbler** and the **Evaluator**. The two parties are semi-honest, meaning they will follow the protocol as outlined, but are still interested in knowing the other party’s input. While it doesn’t matter which party takes which role, the actions performed by each role is widely different. As a short introductory summary, the Garbler creates the circuit with garbled gates, and the evaluator uses said circuit to evaluate the final answer.

### Boolean Gates

The Garbler takes the role of creating boolean garbled gates such as AND, OR, or XOR gates. The basis of a boolean gate lies in its 2 binary inputs and 1 binary output as illustrated below:

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 11.45.57 AM.png" alt="" width="375"><figcaption><p>A regular boolean gate</p></figcaption></figure>

We’ll start with thinking about a simple case for a circuit: a single AND gate. An AND gate would have the 4 potential cases below:

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 11.47.13 AM.png" alt="" width="256"><figcaption><p>An AND gate</p></figcaption></figure>

How can the inputs of each party be hidden from each other?

The Garbler creates a label for each input and output case of $$\textcolor{red}a=0,\textcolor{red}a=1,\textcolor{green}b=0,\textcolor{green}b=1,\textcolor{blue}c=0,\textcolor{blue}c=1$$ as $$\textcolor{red}{X_a^0}, \textcolor{red}{X_a^1}, \textcolor{green}{X_b^0}, \textcolor{green}{X_b^1}, \textcolor{blue}{X_c^0}, \textcolor{blue}{X_c^1}$$ , respectively. Labels should not reveal the underlying value (0 or 1) of the input. Now, boolean gates will look like this:

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 11.48.51 AM.png" alt="" width="373"><figcaption><p>A boolean gate with labels</p></figcaption></figure>

Furthermore, a pair of $$\textcolor{red}a$$ and $$\textcolor{green}b$$ labels will be used to encrypt the outcome label of a given gate as shown below:

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 11.51.21 AM.png" alt=""><figcaption><p>An AND gate encrypted by labels</p></figcaption></figure>

This encrypted gate is sent to the Evaluator, but the Evaluator also needs the $$\textcolor{red}a$$ and $$\textcolor{green}b$$ labels corresponding to the Garbler’s and the Evaluator’s inputs to decrypt a row.

### How can we send the labels corresponding to the parties' inputs to the Evaluator without exposing inputs to each other?

Let’s assume that $$\textcolor{red}a$$ is the private input of the Garbler and $$\textcolor{green}b$$ is the private input of the Evaluator.

#### _Case 1._ Sending $$\textcolor{red}{X_a^0}$$ or $$\textcolor{red}{X_a^1}$$ from the Garbler to the Evaluator

This case is very simple: just send it! The Garbler knows $$\textcolor{red}{X_a^0}$$ and $$\textcolor{red}{X_a^1}$$ since it made them, and since the labels should hide the input values, it’s safe to send one of the values as it is.

#### _Case 2_. Sending $$\textcolor{green}{X_b^0}$$ or $$\textcolor{green}{X_b^1}$$ from the Garbler to the Evaluator

Now this case is a bit tricky. Since the Garbler should not know the Evaluator’s private input $$\textcolor{green}b$$, the Garbler can’t simply ask the Evaluator for their input and send the corresponding label. Additionally, the Garbler shouldn’t send both labels to the Evaluator since the Evaluator could then decrypt two values in the gate. Thus, two constraints must be met to prevent exposed inputs: 1) the Garbler should not know what label the Evaluator picked and 2) the Evaluator should not know the other label. A process called Oblivious Transfer (OT) perfectly satisfies these conditions. (To understand how a simple implementation of 2-to-1 OT works, [watch this video](https://www.youtube.com/watch?v=wE5cl8J27Is). Note that an error occurs at the end of the video, where $$m^*$$ should equal $$m'_b -k$$ instead)

<div align="center"><figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 2.36.38 PM.png" alt="" width="358"><figcaption><p>Oblivious Transfer (OT) used in Yao’s with the example where the Evaluator’s input is 1</p></figcaption></figure></div>

With OT, the corresponding labels can now be securely sent from the Garbler to the Evaluator.

### How do both parties get the outcome?

Let’s look at the case where $$\textcolor{red}a=0$$ and $$\textcolor{green}b=1$$ for simplicity. As shown below, the Garbler will create the labels and the encrypted gate. Next, the Garbler will send the gate and $$\textcolor{red}{X_a^0}$$. Additionally, they will send $$\textcolor{green}{X_b^0}$$ and $$\textcolor{green}{X_b^1}$$ through the oblivious transfer protocol such that the Evaluator only receives $$\textcolor{green}{X_b^1}$$.

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 2.40.45 PM.png" alt=""><figcaption><p>Items sent by Garbler to Evaluator for a single AND gate where <span class="math">\textcolor{red}a =0</span> &#x26; <span class="math">\textcolor{green}b =1</span></p></figcaption></figure>

Finally, with $$\textcolor{red}{X_a^0}$$ and $$\textcolor{red}{X_b^1}$$, the Evaluator tries decrypting all 4 rows of the gate. The Evaluator will successfully only decrypt the row $$\mathsf{Enc}_{\textcolor{red}{X_a^0},\textcolor{green}{X_b^1}}(\textcolor{blue}{X_c^0})$$ to receive $$\textcolor{blue}{X_c^0}$$. As the final step, the Evaluator relays $$\textcolor{blue}{X_c^0}$$ back to the Garbler, therein which the two communicate and the Garbler sends the corresponding $$\textcolor{blue}c=0$$ value of the label $$\textcolor{blue}{X_c^0}$$ back to the Evaluator, so both parties finally know the total result.

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 3.19.56 PM.png" alt=""><figcaption><p>Total Yao protocol for a single AND gate where <span class="math">\textcolor{red}a =0</span> &#x26; <span class="math">\textcolor{green}b =1</span></p></figcaption></figure>

Now a couple additional questions will probably come to mind:

### How does the Evaluator know a value has been properly decrypted?

In the naive version of Yao’s Garbled Circuits, the Evaluator doesn’t specifically know which row in a gate to decrypt, but instead tries decrypting all 4 rows of a gate and settles on the one value that is decrypted properly. This insinuates that the Evaluator knows if they have decrypted a message properly. This can be done as per the method described in the [Mina book](https://o1-labs.github.io/proof-systems/fundamentals/zkbook_2pc/basics.html) which pads the original message with 0s. For example, say our original message b was padded with 4 zeros to be $$00004312$$, and we encoded $$b$$ with $$a$$ like so: $$\mathsf{Enc}_a(b)$$. If we then tried decrypting this encrypted message with a different key than $$a$$, say $$c$$, we would instead get something like $$\mathsf{Dec}_c(\mathsf{Enc}_a(b)) = 15819739$$ instead of $$\mathsf{Dec}_a(\mathsf{Enc}_a(b)) = 00004312$$. Thus, we would be able to tell if the decryption succeeded or not by looking at whether or not the decryption result achieves the given amount of zero padding.

### How can multiple gates be implemented?

We can delegate each gate’s outcome label as an input label for the next gate. For example, if we had $$(\textcolor{red}a \land \textcolor{green}b)\oplus \textcolor{red}a$$, we’d have the following encrypted gates:

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 3.31.14 PM.png" alt=""><figcaption><p>Encrypted gates for <span class="math">(\textcolor{red}a \land \textcolor{green}b)\oplus \textcolor{red}a</span></p></figcaption></figure>

### Couldn’t the output value expose the Evaluator’s input value to the Garbler?

Both our simple example of a single AND gate and the multiple gate instance above could expose the Evaluator’s input to the Garbler. For example, for a single AND gate, if the Garbler’s input was 1, and they received a result of 0 from the Evaluator, they can now tell that the Evaluator’s input value was 0. Now, how is this mitigated? The simple answer is it’s not explicitly addressed in the naive version of Yao’s Garbled Circuits with semi-honest parties, but is solved through optimizations done on top of Yao to protect against malicious parties.

### Can’t the Evaluator determine the Garbler’s input based on the index of the encrypted row they try to decrypt?

Now what do I mean by this? Well, if the Garbler naively sends over gates every time with indexes and key orders like so…

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 3.35.31 PM.png" alt=""><figcaption><p>Encrypted AND gate with ordered indexes</p></figcaption></figure>

…once the Evaluator successfully decrypts, say the row with index 2, they can immediately determine the two inputs are $$\textcolor{red}{a}=1$$ and $$\textcolor{green}{b}=0$$. To prevent this leak from occurring, we rely on the process of garbling gates with the “point-and-permute” method mentioned later.

## Optimizations on Yao

In this section, I’ll illustrate optimizations done onto the naive version of Yao’s garbled circuits, as outlined in the [Mina Book](https://o1-labs.github.io/proof-systems/fundamentals/zkbook_2pc/overview.html). More specifically, we’ll go over “Point and Permute,” “Free-XOR,” “Row Reduction,” and “Half Gates.” Note that a full set of boolean gates can be created by using AND and XOR gates, which are the gates that these optimizations target.

<figure><img src="../.gitbook/assets/image (1).png" alt="" width="360"><figcaption><p>Construction of all other boolean gates with AND &#x26; XOR</p></figcaption></figure>

### Point-and-Permute

The “pointer bits” in [the given Yao article](https://cronokirby.com/posts/2022/05/explaining-yaos-garbled-circuits/), are instead referred to as “[Color bits](https://o1-labs.github.io/proof-systems/fundamentals/zkbook_2pc/pap.html)” by Mina. Mina cites the [BMR paper](https://www.cs.ucdavis.edu/~rogaway/papers/bmr90) for this optimization, which does not name the bits themselves. Moving on, we’ll refer to these bits as “pointer bits” as well.

In a naive setting, the evaluator would need to try decrypting all 4 rows of an encrypted gate to obtain the gate outcome; however, a more ideal solution is to ensure the evaluator pinpoints and decrypts the one desired row in a gate every time. This can be done through “pointer bits.” This method also shuffles the circuit to prevent leaking the Garbler’s inputs to the Evaluator by way of gate row index.

A pointer bit is a random $$0$$ or $$1$$ bit paired with a given label. As in the Mina book, let’s set up the labels with pointer bits as such: $$\textcolor{red}{X_a^0}$$ with $$0$$, $$\textcolor{red}{X_a^1}$$ with $$1$$, $$\textcolor{green}{X_b^0}$$ with $$1$$, and $$\textcolor{green}{X_b^1}$$ with $$0$$. Next, the rows are ordered by the random pointer bits, effectively shuffling the encrypted gates.

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 4.01.20 PM.png" alt=""><figcaption><p>Garbled rows using pointer bits</p></figcaption></figure>

Finally, when labels are tossed around from the Garbler to the Evaluator, the corresponding pointer bit must be passed around as well. With our previous example of $$\textcolor{red}a=0$$ and $$\textcolor{green}b=1$$, the Garbler would send $$\textcolor{red}{X_a^0}$$ with a $$0$$ pointer bit and the Evaluator would receive $$\textcolor{green}{X_b^1}$$ with a $$0$$ pointer bit, telling the Evaluator to decrypt the first row.

Since the pointer bits are chosen completely at random, the Evaluator can no longer know which garbled row corresponds to which inputs. Additionally, from point-and-permute, the Evaluator only needs to decrypt one value per gate, instead of potentially trying all 4.

### Free-XOR

We can free the cost of XOR gates. Shocking, right?

For this, we need to change our input and output labels. First, the Garbler creates a random bit string $$R$$. Next, we define $$\textcolor{red}{X_a^1} = \textcolor{red}{X_a^0}\oplus R$$, $$\textcolor{green}{X_b^1} = \textcolor{green}{X_b^0}\oplus R$$, $$\textcolor{blue}{X_c^1} = \textcolor{blue}{X_c^0}\oplus R$$ where $$\textcolor{blue}{X_c^0} = \textcolor{red}{X_a^0}\oplus \textcolor{green}{X_b^0}$$. This means that the superscript of labels that denote their true values can be removed such that the gates will change like so:

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 4.08.08 PM.png" alt=""><figcaption><p>How all gates change with the Free-XOR optimization added to the circuit</p></figcaption></figure>

With the given constraints, the Evaluator can simply XOR the input labels to get the outcome label instead of relying on a Garbler’s gate with encrypted rows. This is shown below:

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 4.14.15 PM.png" alt=""><figcaption><p>Free-XOR XOR gate(Takes inspiration from the <a href="https://o1-labs.github.io/proof-systems/fundamentals/zkbook_2pc/freexor.html">Free-XOR table illustrated in the Mina Book</a>)</p></figcaption></figure>

Since the XOR is done on labels, the Garbler does not have to send over the encrypted XOR gate thus, there is no risk of leaking values. Therefore, XOR gates become free, where the Garbler can just send labels with no encrypted gate and the Evaluator can determine the output label themselves.

Note that the change in labels doesn’t just affect XOR gates, but affects all gates within the circuit. This means, of course, AND Gates will still require an encrypted gate to be sent by the Garbler to the Evaluator as we can’t do the same magic for this different computation.

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 4.19.49 PM.png" alt=""><figcaption><p>Free-XOR AND gate</p></figcaption></figure>

### Row Reduction

As a quick interlude on encryption to prep for the next section, I’ll explain a little about the actual encryption algorithm. Previously, I depicted an output gate value encrypted by two values through something like $$\mathsf{Enc}_{\textcolor{red}{X_a},\textcolor{green}{X_b}}(\textcolor{blue}{X_c})$$; however, in the [Mina book](https://o1-labs.github.io/proof-systems/fundamentals/zkbook_2pc/pap.html), they explain more specifically that the encryption algorithm is a hash function in addition to a one-time pad such that $$\mathsf{Enc}_{\textcolor{red}{X_a},\textcolor{green}{X_b}}(\textcolor{blue}{X_c}) = H(\textcolor{red}{X_a}, \textcolor{green}{X_b})\oplus\textcolor{blue}{X_c} = X_{\mathsf{enc}}$$ and $$\mathsf{Dec}_{\textcolor{red}{X_a},\textcolor{green}{X_b}}(X_{\mathsf{enc}}) =  H(\textcolor{red}{X_a}, \textcolor{green}{X_b})\oplus X_{enc} = H(\textcolor{red}{X_a}, \textcolor{green}{X_b})\oplus H(\textcolor{red}{X_a}, \textcolor{green}{X_b})\oplus \textcolor{blue}{X_c} = \textcolor{blue}{X_c}$$. Now that we’ve shown this, let’s get into Row Reduction!

The concept of row reduction is fairly simple. Let’s first take our Free-XOR AND gate and apply our encryption algorithm to it.

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 4.31.34 PM.png" alt=""><figcaption><p>Free-XOR AND gate with encryption algorithm</p></figcaption></figure>

With row reduction, we can remove the first encrypted row by setting it to “$$0$$.” How can we do this? If we set $$\textcolor{blue}{X_c} = H(\textcolor{red}{X_a}, \textcolor{green}{X_b})$$, we can see that the first row becomes $$H(\textcolor{red}{X_a}, \textcolor{green}{X_b})\oplus H(\textcolor{red}{X_a}, \textcolor{green}{X_b}) = 0$$! Of course, we have to uphold this equality across all rows now which will result in:

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 4.37.41 PM.png" alt=""><figcaption><p>Free-XOR AND gate with encryption algorithm and row reduction</p></figcaption></figure>

Now, you might be wondering how the outcome label of the first row case could be evaluated. Easy! It’s just the hash of the two given input labels.

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 4.39.11 PM.png" alt=""><figcaption><p>“Hidden” first row of the Free-XOR AND gate</p></figcaption></figure>

In the end, the row reduction process reduces the gate overhead by one row. Additionally, the Evaluator can determine the output label of the first “hidden” row through hashing the input labels themselves.

### Half Gates

Let’s now try to optimize AND gates with “Half Gates.” We’ll take a look at the 3 cases where only the Garbler knows one input ([Case 1](yaos-garbled-circuits.md#case-1.-the-garbler-knows-one-input-to-an-and-gate-but-doesnt-know-the-other-input)), only the Evaluator knows one input ([Case 2](yaos-garbled-circuits.md#case-2.-the-evaluator-knows-one-input-to-an-and-gate-but-doesnt-know-the-other-input)), or neither knows the inputs of a gate ([Case 3](yaos-garbled-circuits.md#case-3-two-halves-make-a-whole-.-neither-the-garbler-nor-the-evaluator-know-any-of-the-inputs-to-the)). The two halves of [Case 1](yaos-garbled-circuits.md#case-1.-the-garbler-knows-one-input-to-an-and-gate-but-doesnt-know-the-other-input) and [Case 2](yaos-garbled-circuits.md#case-2.-the-evaluator-knows-one-input-to-an-and-gate-but-doesnt-know-the-other-input) will be merged to solve [Case 3](yaos-garbled-circuits.md#case-3-two-halves-make-a-whole-.-neither-the-garbler-nor-the-evaluator-know-any-of-the-inputs-to-the). We call back to the second gate in the figure “Encrypted gates for " as an example of [Case 1](yaos-garbled-circuits.md#case-1.-the-garbler-knows-one-input-to-an-and-gate-but-doesnt-know-the-other-input) if $$\textcolor{red}{a}$$ is the Garbler’s private input and $$\textcolor{green}b$$ is the Evaluator’s private input.

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 4.53.50 PM.png" alt=""><figcaption><p>“Encrypted gates for <span class="math">(\textcolor{red}a\land\textcolor{green}b)\oplus\textcolor{red}a</span>” adapted to Free-XOR</p></figcaption></figure>

Remember that with Free-XOR $$X_i$$ is the label for $$i=0$$ and $$X_i \oplus R$$ is the label for $$i=1$$.

#### Case 1. The Garbler knows one input ($$\textcolor{red}a$$) to an AND gate, but doesn’t know the other input ($$\textcolor{green}b$$)

Let’s dive into the two possible subcases of $$\textcolor{green}b$$:

_Subcase 1_. $$\textcolor{green}b=0$$

No matter what $$\textcolor{red}a$$ is, the gate output should be $$0$$ ($$\textcolor{blue}{X_c}$$). Additionally, we only need to encrypt with the $$\textcolor{green}b$$ label ($$\textcolor{green}{X_b}$$). As such, we use the encryption $$\mathsf{Enc}_{\textcolor{green}{X_b}}(\textcolor{blue}{X_c})$$ if $$\textcolor{green}b=0$$.

<figure><img src="../.gitbook/assets/Screenshot 2025-02-24 at 1.33.00 PM.png" alt=""><figcaption><p><a href="yaos-garbled-circuits.md#case-1.-the-garbler-knows-one-input-to-an-and-gate-but-doesnt-know-the-other-input">Case 1</a> where input <span class="math">\textcolor{green}b=0</span></p></figcaption></figure>

_Subcase 2._ $$\textcolor{green}b=1$$

This subcase requires some more thought as different values of$$\textcolor{red}a$$ should result in different results. If $$\textcolor{red}a = 0$$, the result should be $$0$$ ($$\textcolor{blue}{X_c}$$), and if $$\textcolor{red}a=1$$, the result should be $$1$$ ($$\textcolor{blue}{X_c}\oplus R$$); however, similar to _Subcase 1_, we only need to encrypt with the $$\textcolor{green}b$$ label ($$\textcolor{green}{X_b \oplus R}$$). Therefore, we use the encryption $$\mathsf{Enc}_{\textcolor{green}{X_b\oplus R}}(\textcolor{blue}{X_c\oplus[\textcolor{red}a\cdot R]})$$ if $$\textcolor{green}b=1$$. In the case $$\textcolor{red}a=0$$, we use $$\mathsf{Enc}_{\textcolor{green}{X_b\oplus R}}(\textcolor{blue}{X_c})$$ and in the case $$\textcolor{red}a=1$$, we use $$\mathsf{Enc}_{\textcolor{green}{X_b\oplus R}}(\textcolor{blue}{X_c\oplus R})$$, fulfilling our desire of having two different outputs for the two different cases of our known input $$\textcolor{red}a$$.

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 6.12.50 PM.png" alt=""><figcaption><p><a href="yaos-garbled-circuits.md#case-1.-the-garbler-knows-one-input-to-an-and-gate-but-doesnt-know-the-other-input">Case 1 </a>where input <span class="math">\textcolor{green}b=1</span></p></figcaption></figure>

In total, the two rows in this AND gate will be $$\mathsf{Enc}_{\textcolor{green}{X_b}}(\textcolor{blue}{X_c})$$ and $$\mathsf{Enc}_{\textcolor{green}{X_b\oplus R}}(\textcolor{blue}{X_c\oplus[\textcolor{red}a\cdot R]})$$. But hold on… Remember row reduction? By additionally applying row reduction to our two rows, we can get **one final ciphertext!**

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 6.13.25 PM.png" alt=""><figcaption><p>Total resulting ciphertexts of <a href="yaos-garbled-circuits.md#case-1.-the-garbler-knows-one-input-to-an-and-gate-but-doesnt-know-the-other-input">Case 1</a></p></figcaption></figure>

#### **Case 2. The Evaluator knows one input (**$$\textcolor{red}a$$**) to an AND gate, but doesn’t know the other input (**$$\textcolor{green}b$$**)**

Let’s dive into the two possible subcases of $$\textcolor{red}a$$:

_Subcase 1._ $$\textcolor{red}a=0$$

If $$\textcolor{red}a=0$$, the result will always be $$0$$ ($$\textcolor{blue}{X_c}$$). This is simple enough for the Garbler to create with the Evaluator’s label ($$\textcolor{red}{X_a}$$) as the encryption key: $$\mathsf{Enc}_{\textcolor{red}{X_a}}(\textcolor{blue}{X_c})$$.

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 6.26.26 PM.png" alt=""><figcaption><p><a href="yaos-garbled-circuits.md#case-2.-the-evaluator-knows-one-input-to-an-and-gate-but-doesnt-know-the-other-input">Case 2</a> where input <span class="math">\textcolor{red}a=0</span></p></figcaption></figure>

_Subcase 2_. $$\textcolor{red}a=1$$

Once again, the latter subcase, $$\textcolor{red}a=1$$, becomes complicated due to the possible varying outcomes dependent on $$\textcolor{green}b$$. So how can this subcase be resolved? We’ll use the Evaluation’s label ($$\textcolor{red}{X_a\oplus R}$$) as the encryption key again and create $$\mathsf{Enc}_{\textcolor{red}{X_a \oplus R}}(\textcolor{blue}{X_c}\oplus \textcolor{green}{X_b})$$. Let’s take a look at how this can be used to create both outcomes:

First, the Evaluator decrypts the ciphertext to obtain $$\textcolor{blue}{X_c}\oplus \textcolor{green}{X_b}$$. Next, the Evaluator XORs this value with the pertaining $$\textcolor{green}b$$ label. That is, if $$\textcolor{green}b=0$$ ($$\textcolor{green}{X_b}$$), the Evaluator computes $$\textcolor{blue}{X_c}\oplus \textcolor{green}{X_b}\oplus(\textcolor{green}{X_b})=\textcolor{blue}{X_c}=$$the outcome label for $$0$$, exactly how we want it! On the other hand, if $$\textcolor{green}b=1$$ ($$\textcolor{green}{X_b\oplus R}$$), the Evaluator computes $$\textcolor{blue}{X_c}\oplus \textcolor{green}{X_b}\oplus(\textcolor{green}{X_b\oplus R}) = \textcolor{blue}{X_c\oplus R} =$$ the outcome label for 1. Thus, we have successfully created a ciphertext for $$\textcolor{red}a=1$$ that can be used to generate our desired outcome labels.

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 6.39.09 PM.png" alt=""><figcaption><p><a href="yaos-garbled-circuits.md#case-2.-the-evaluator-knows-one-input-to-an-and-gate-but-doesnt-know-the-other-input">Case 2</a> where input <span class="math">\textcolor{red}a=1</span></p></figcaption></figure>

In total, we get $$\mathsf{Enc}_{\textcolor{red}{X_a}}(\textcolor{blue}{X_c})$$ and $$\mathsf{Enc}_{\textcolor{red}{X_a \oplus R}}(\textcolor{blue}{X_c}\oplus \textcolor{green}{X_b})$$ for this AND gate. Just like in [Case 1](yaos-garbled-circuits.md#case-1.-the-garbler-knows-one-input-to-an-and-gate-but-doesnt-know-the-other-input), we can also apply row reduction to reduce this to one ciphertext row!

<figure><img src="../.gitbook/assets/Screenshot 2025-01-13 at 6.42.48 PM.png" alt=""><figcaption><p>Total resulting ciphertexts of <a href="yaos-garbled-circuits.md#case-2.-the-evaluator-knows-one-input-to-an-and-gate-but-doesnt-know-the-other-input">Case 2</a></p></figcaption></figure>

#### Case 3 (“Two halves make a whole”). Neither the Garbler nor the Evaluator know any of the inputs to the AND gate.

To optimize this case, we’ll employ a little trick:

<figure><img src="../.gitbook/assets/Screenshot 2025-02-24 at 5.30.19 PM.png" alt=""><figcaption><p><a href="yaos-garbled-circuits.md#case-3-two-halves-make-a-whole-.-neither-the-garbler-nor-the-evaluator-know-any-of-the-inputs-to-the">Case 3</a> trick (Reference: <a href="https://eprint.iacr.org/2014/756.pdf">“Two Halves Make a Whole” Paper</a>)</p></figcaption></figure>

If the $$\textcolor{orchid}r$$ is a random bit chosen by the Garbler, this means that the $$(\textcolor{red}a\land \textcolor{orchid}r)$$ AND gate can be optimized via [Case 1](yaos-garbled-circuits.md#case-1.-the-garbler-knows-one-input-to-an-and-gate-but-doesnt-know-the-other-input), since the Garbler knows . Furthermore, if the Garbler can leak $$(\textcolor{orchid}r\oplus \textcolor{green}b)$$ to the Evaluator, the $$(\textcolor{red}a\land (\textcolor{orchid}r \oplus \textcolor{green}b))$$ AND gate can the optimized via [Case 2](yaos-garbled-circuits.md#case-2.-the-evaluator-knows-one-input-to-an-and-gate-but-doesnt-know-the-other-input), since the Evaluator will know an input. Finally, all together, the XOR gate between $$(\textcolor{red}a\land \textcolor{orchid}r)$$ and $$(\textcolor{red}a\land (\textcolor{orchid}r \oplus \textcolor{green}b))$$ would be free from the Free-XOR optimization. Therefore, in total, an AND gate where neither input is known by the Garbler or Evaluator can be reduced down to $$1((\textcolor{red}a\land \textcolor{orchid}r)$$ AND gate$$) + 1((\textcolor{red}a\land (\textcolor{orchid}r\oplus \textcolor{green}b))$$ AND gate$$)$$ **2 ciphertexts**!

Now, this is banking on the fact that the Garbler can leak $$(\textcolor{orchid}r\oplus \textcolor{green}b)$$ to the Evaluator, but how can this be done? First of all, $$(\textcolor{orchid}r\oplus \textcolor{green}b)$$ is safe to leak as the Evaluator would not know the random value $$\textcolor{orchid}r$$ and therefore would not be able to discern the private input $$\textcolor{green}b$$. So how’s it done? By setting $$(\textcolor{orchid}r\oplus \textcolor{green}b)$$as the pointer bit corresponding to the given $$\textcolor{green}b$$ value!

<figure><img src="../.gitbook/assets/Screenshot 2025-02-24 at 5.38.06 PM.png" alt=""><figcaption><p>Resulting label and pointer bit dependent on <span class="math">\textcolor{orchid}r</span> and <span class="math">\textcolor{green}b</span> value</p></figcaption></figure>

Since the random $$\textcolor{orchid}r$$ ensures the pointer bit stays random, the Evaluator shouldn’t be able to discern the value of $$\textcolor{green}b$$ from the label or the pointer bit.

Thus, we finally have our final structure for an AND gate where neither the Garbler or Evaluator knows the inputs. Below, we try to show this scheme with a diagram and additionally include an example of how it would run with the values $$\textcolor{red}a=1$$, $$\textcolor{green}b=1$$, and $$\textcolor{orchid}r=0$$.

<figure><img src="../.gitbook/assets/Screenshot 2025-02-24 at 5.42.17 PM.png" alt=""><figcaption><p>Total structure of an AND gate, where neither the Garbler or Evaluator know the inputs</p></figcaption></figure>

<figure><img src="../.gitbook/assets/Screenshot 2025-02-24 at 5.50.04 PM.png" alt=""><figcaption><p>Example for the total structure of an AND gate, where neither the Garbler or Evaluator know the inputs <span class="math">\textcolor{red}a=1</span>, <span class="math">\textcolor{green}b=1</span>, <span class="math">\textcolor{orchid}r=0</span></p></figcaption></figure>

Therefore, [Case 3](yaos-garbled-circuits.md#case-3-two-halves-make-a-whole-.-neither-the-garbler-nor-the-evaluator-know-any-of-the-inputs-to-the) can be implemented through [Case 1](yaos-garbled-circuits.md#case-1.-the-garbler-knows-one-input-to-an-and-gate-but-doesnt-know-the-other-input) and [Case 2](yaos-garbled-circuits.md#case-2.-the-evaluator-knows-one-input-to-an-and-gate-but-doesnt-know-the-other-input), costing it only two rows.

## Final Words

In the end, I showed how Yao’s Garbled Circuits runs as an MPC protocol and introduced a couple optimizations that can be added on top of Yao’s Garbled Circuits to drastically reduce the communication overhead between the Garbler and Evaluator by reducing the number of gate rows. The total results of these optimizations are shown below:

<figure><img src="../.gitbook/assets/image (7).png" alt=""><figcaption><p>Resulting gate rows from the naive case to the optimized case</p></figcaption></figure>

## References

* [https://cronokirby.com/posts/2022/05/explaining-yaos-garbled-circuits/](https://cronokirby.com/posts/2022/05/explaining-yaos-garbled-circuits/)
* [https://o1-labs.github.io/proof-systems/fundamentals/zkbook\_2pc/overview.html](https://o1-labs.github.io/proof-systems/fundamentals/zkbook_2pc/overview.html)
* [https://www.youtube.com/watch?v=wE5cl8J27Is](https://www.youtube.com/watch?v=wE5cl8J27Is)
* [https://eprint.iacr.org/2014/756.pdf](https://eprint.iacr.org/2014/756.pdf)

> Written by [Ashley Jeong](https://app.gitbook.com/u/wyEMbFN1Kybygv7v1pKpc2P8q6D2 "mention") from [A41](https://www.a41.io/)
