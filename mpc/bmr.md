---
description: 'Presentation: https://www.youtube.com/watch?v=8fvH4GM_A5k'
---

# BMR

Created by Beaver, Micali, and Rogaway in 1990, the BMR protocol is an multi-party computation (MPC) for boolean circuits for greater than 2 parties. It extends upon the core concept of Yao’s Garbled Circuits with its garbled tables and also carries a constant communication cost similar to Yao’s. Note that explanations online on the specifics of the protocol vary due to the more conceptual nature of the [original paper](https://www.cs.ucdavis.edu/~rogaway/papers/bmr90), so we will focus mostly on the explanation outlined [here](https://securecomputation.org/docs/ch3-fundamentalprotocols.pdf). It is heavily recommended you read [Yao's Garbled Circuits ](yaos-garbled-circuits.md)as a prerequisite.

## Definitions

Let’s start with the basics: the definitions.&#x20;

### A wire sublabel for a given value, gate, and party:

$$
w^{b}_{i,j} = \left( k^{b}_{i,j}, p^{b}_{i,j} \right) \in_{\mathbb{R}} \{0,1\}^{\kappa+1}
$$

$$w$$ = the wire\
$$b$$ = the true value of the wire (0 or 1)\
$$i$$ = the unique wire ID\
$$j$$ = the unique party ID\
$$k$$ = the key to a given value of the wire (length $$\kappa$$)\
$$p$$ = the pointer bit of the given value of the wire (length 1)

Thus, the wire sublabel for a gate by a given party is a set of $$\kappa + 1$$ random bits, $$\kappa$$ of which are the key bits corresponding to a value and 1 of which is the pointer bit used later for shuffling the encrypted tables. Additionally note that the final wire label $$w_i^b = w_{i,1}^b || w_{i,2}^b || \cdots || w_{i,n-1}^b || w_{i,n}^b$$, where 1 to $$n$$ denote the parties.

#### More on Pointer Bits

Let’s expand some more on pointer bits. Each party holds a pointer bit share for a wire’s input value. Additionally, all parties’ pointer bit shares are XOR-ed together should create the final desired pointer bit for that wire input value. AKA $$p_a^0 = p_{a,1}^0 \oplus p_{a,2}^0 \oplus p_{a,3}^0 \oplus \dots \oplus p_{a,n}^0$$ where $$p_a^0$$  is the pointer bit for input value 0 on wire $$a$$ and 1 to $$n$$ are the parties. Subsequently, the pointer bit for the opposite input value should be the opposite bit (AKA $$p_a^1 = 1- p_a^0$$ or $$p_a^1 \oplus p_a^0 = 1$$)

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXeTAKg-kfXlLSBS3suAd005dIvxYTCIV_wOaLx-KxJW1RiwU5L4QJAfgPveVC0HKIN-c6XnxsFBQAahAd0L2g-Yh3F00prL08AVnr7V3BqOmdrVO_gwxK4ZtRCgWzJDRIemIAii4OwXhz_3cpXCqKw?key=-khGs0T8zP29k5iyMMtkN113" alt=""><figcaption><p>The pointer bits for a given gate with 3 wires a, b, and c</p></figcaption></figure>

### Each party’s underlying-MPC input per wire $$w$$:

$$
I_{i,j} = \left( F(i, w_{i,j}^{0}), F(i, w_{i,j}^{1}), p_{i,j}^{0}, f_{i,j} \right)
$$

$$I$$ = the gate input\
$$F$$ = A pseudo-random function (PRF) defined as the following:

$$
F : id, \{0,1\}^{\kappa+1} \mapsto \{0,1\}^{n \cdot (\kappa+1)}
$$

$$f$$ = the flip bit

For each gate, the input for each party will be the two PRFs of the gate index with the sublabels corresponding to input of 0 or 1, the pointer bit of the true input value, and the flip bit (more on this later).

### Naively encrypted row of table:

$$
e_{v_a, v_b} = w^{v_c}_{c} \bigoplus\limits_{j=1..n} \left( F(i, w^{v_a}_{a,j}) \oplus F(i, w^{v_b}_{b,j}) \right)
$$

$$e_{v_a, v_b}$$ = encrypted row on input values $$v_a$$ and $$v_b$$\
$$v_a$$ = value of input wire $$a$$\
$$v_b$$ = value of input wire $$b$$\
$$v_c$$ = value of output wire $$c$$

Note that the first subscript of the wires $$w$$ also indicate the wire ID, where $$a$$ and $$b$$ refer to the input wires of a given gate and $$c$$ refers to the output wire. Therefore, $$w_c^{v_c}$$ is the output label of wire $$c$$ of true output value $$v_c$$, while $$w_{a,j}^{v_a}$$ is the sublabel for party $$j$$ of input wire $$a$$ with the true value $$v_a$$. This encrypted row is a naive implementation not including the use of flip bits to further secure the encryption.

## Creating the Garbled Table Naively

Let’s take another look at the formula to naively creating the encrypted rows of our gate tables from the previous section:

$$
e_{v_a, v_b} = w^{v_c}_{c} \bigoplus\limits_{j=1..n} \left( F(i, w^{v_a}_{a,j}) \oplus F(i, w^{v_b}_{b,j}) \right)
$$

With the 4 possible permutations of $$v_a$$ and $$v_b$$ , we get the 4 rows as follows for an AND gate:

$$
\begin{aligned}
e_{0,0} &= w_c^{0} \oplus \left[ F(i, w_{a,1}^{0}) \oplus F(i, w_{b,1}^{0}) \right] \oplus \dots \oplus \left[ F(i, w_{a,n}^{0}) \oplus F(i, w_{b,n}^{0}) \right] \\
e_{0,1} &= w_c^{0} \oplus \left[ F(i, w_{a,1}^{0}) \oplus F(i, w_{b,1}^{1}) \right] \oplus \dots \oplus \left[ F(i, w_{a,n}^{0}) \oplus F(i, w_{b,n}^{1}) \right] \\
e_{1,0} &= w_c^{0} \oplus \left[ F(i, w_{a,1}^{1}) \oplus F(i, w_{b,1}^{0}) \right] \oplus \dots \oplus \left[ F(i, w_{a,n}^{1}) \oplus F(i, w_{b,n}^{0}) \right] \\
e_{1,1} &= w_c^{1} \oplus \left[ F(i, w_{a,1}^{1}) \oplus F(i, w_{b,1}^{1}) \right] \oplus \dots \oplus \left[ F(i, w_{a,n}^{1}) \oplus F(i, w_{b,n}^{1}) \right]
\end{aligned}
$$

Note that the output labels are defined as such as well:

$$
\begin{aligned}
w_c^{0} &= w_{c,1}^{0} \parallel \dots \parallel w_{c,n}^{0} \parallel p_c^{0} \\
w_c^{1} &= w_{c,1}^{1} \parallel \dots \parallel w_{c,n}^{1} \parallel p_c^{1}
\end{aligned}
$$

Similar to Yao’s, we now “garble” the table. For example, if $$p_a^0 = 1$$, $$p_b^1 = 0$$ meaning $$p_a^1 = 0$$, $$p_b^0 = 1$$, our table is shuffled as so:

<figure><img src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfoEVSh5OBc1sJROR9wKj-Jqwc59ucrPFfcC3EBle1K4IaiYpwybNspkZHgkd5TTiE4L-d6WyQMc0hbv582hDgYSxbfAZK_vCFKh_Yc43mODFrqdODU1jOlcttScQeI0_JmPuHFMLcC2CLFvBVxSRo?key=-khGs0T8zP29k5iyMMtkN113" alt=""><figcaption><p>Shuffling of encrypted table</p></figcaption></figure>

Now, this is the naive way to create a garbled encrypted table; however, a major flaw that exposes the input values to the evaluator can occur. To prevent this information from being leaked, we use “flip bits.”

TODO(ashjeong): Add explanation of a possible attack for this naive case.

## Creating the Garbled Table Securely with Flip Bits

Flip bits obscure the true input values for the evaluator. Similar to pointer bits, each party creates their own flip bit share for each wire such that for a given wire, the total flip bit is  or all flip bit shares XOR-ed together. Since wire labels and PRF functions result in random strings that are consistent across a binary value, switching the correlations of said labels to their opposite value is fine. Therefore, we instead **encrypt rows** of the encrypted table while obscuring input values with flip bits as shown below:

$$
e_{v_a, v_b} = w_c^{v_c \oplus f_c} \bigoplus\limits_{j=1..n} \left( F(i, w_{a,j}^{v_a \oplus f_a}) \oplus F(i, w_{b,j}^{v_b \oplus f_b}) \right) \\
$$

Ex. $$f_a = 0$$, $$f_b = 1$$, $$f_c = 1$$ for an AND gate:

$$
\begin{aligned}
e_{0,0} &= w_c^{0 \oplus 1} \oplus \left[ F(i, w_{a,1}^{0 \oplus 0}) \oplus F(i, w_{b,1}^{0 \oplus 1}) \right] \oplus \dots \oplus \left[ F(i, w_{a,n}^{0 \oplus 0}) \oplus F(i, w_{b,n}^{0 \oplus 1}) \right]  \\
e_{0,1} &= w_c^{0 \oplus 1} \oplus \left[ F(i, w_{a,1}^{0 \oplus 0}) \oplus F(i, w_{b,1}^{1 \oplus 1}) \right] \oplus \dots \oplus \left[ F(i, w_{a,n}^{0 \oplus 0}) \oplus F(i, w_{b,n}^{1 \oplus 1}) \right]  \\
e_{1,0} &= w_c^{0 \oplus 1} \oplus \left[ F(i, w_{a,1}^{1 \oplus 0}) \oplus F(i, w_{b,1}^{0 \oplus 1}) \right] \oplus \dots \oplus \left[ F(i, w_{a,n}^{1 \oplus 0}) \oplus F(i, w_{b,n}^{0 \oplus 1}) \right]  \\
e_{1,1} &= w_c^{1 \oplus 1} \oplus \left[ F(i, w_{a,1}^{1 \oplus 0}) \oplus F(i, w_{b,1}^{1 \oplus 1}) \right] \oplus \dots \oplus \left[ F(i, w_{a,n}^{1 \oplus 0}) \oplus F(i, w_{b,n}^{1 \oplus 1}) \right] 
\end{aligned}
$$

If the flip bit of a wire is 0 such as for wire a shown above, the wire sublabels refer to their corresponding input value (e.g. $$w_{a,1}^1$$ refers to input value 1). Else, if the flip bit is 1 as in wire $$b$$ above, the wire sublabel refers to the opposite input value (e.g. $$w_{b,1}^0$$ refers to input value 1).

Flip bits introduce more constrained chaos to prevent values from being exposed. Only 4 possible label permutations of each wire $$a$$, $$b$$, and $$c$$ for its 4 possibilities of input values exist when not using flip bits, yet with its inclusion, the total number of label permutations jumps to 32. Take a look below showing off all possible label permutations for a single input case of a AND gate where the inputs are 1.

<figure><img src="../.gitbook/assets/Screenshot 2025-02-19 at 6.28.55 PM.png" alt=""><figcaption><p>AND gate possibilities for (1,1) -> 1</p></figcaption></figure>

## BMR Protocol Summary

All Parties:

1. Create their own keys for all possible wires values
2. Create their own pointer bit shares for each wire value (1 for each wire since the other can be simply calculated)
3. Create their own flip bit shares for each wire

For n-MPC:

4. Receive parties’s private input bits
5. Receive MPC inputs of each party  $$I_{i,j}$$
6. For each gate
   1. Compute total flip bits $$f$$ for each wire
   2. Compute total pointer bits $$p$$ for each wire value
   3. Create the garbled table

One party who is set to be the evaluator:

7. Receive the garbled tables and active wire labels
8. Evaluate garbled circuit re-creating the garbled inputs themselves

## References

* [https://securecomputation.org/docs/ch3-fundamentalprotocols.pdf](https://securecomputation.org/docs/ch3-fundamentalprotocols.pdf)
* [https://eprint.iacr.org/2016/1066.pdf](https://eprint.iacr.org/2016/1066.pdf)
* [https://www.cs.ucdavis.edu/\~rogaway/papers/bmr90](https://www.cs.ucdavis.edu/~rogaway/papers/bmr90)

> Written by [Ashley Jeong](https://app.gitbook.com/u/wyEMbFN1Kybygv7v1pKpc2P8q6D2 "mention") from A41
