# GMW

Created by Oded **Goldreid**, Silvio **Micali**, and Avi **Wigderson**, **GMW** is a **multi-party computation (MPC)** protocol designed for boolean or arithmetic circuits for $$n$$-parties where $$n \ge 2$$. We’ll focus on the creation of boolean circuits in GMW for simplicity.

## 2-party GMW <a href="#d3bb" id="d3bb"></a>

Let’s start with 2-party GMW. Say we have two parties: <mark style="color:red;">Alice</mark> and <mark style="color:green;">Bob</mark>, and we have one boolean gate with 2 input bits $$\textcolor{red}i$$ (known by <mark style="color:red;">Alice</mark>) and $$\textcolor{green}j$$ (known by <mark style="color:green;">Bob</mark>) and one output bit $$k$$.

<figure><img src="../.gitbook/assets/image (8).png" alt=""><figcaption><p>A boolean gate</p></figcaption></figure>

In the GMW protocol, every party first undergoes a process of secret sharing with the other parties on the input bits they know:

For her input bit $$\textcolor{red}i$$, <mark style="color:red;">Alice</mark> chooses a random bit share $$\textcolor{green}{s_i^b}$$ to send to <mark style="color:green;">Bob</mark>. Meanwhile, she creates a personal bit share $$\textcolor{red}{s_i^a}$$ for herself such that $$\textcolor{red}{s_i^a}\oplus\textcolor{green}{s_i^b}=\textcolor{red}i$$.

The same, but opposite, occurs for <mark style="color:green;">Bob</mark>. <mark style="color:green;">Bob</mark> chooses a random bit share $$\textcolor{red}{s^a_j}$$ to send to <mark style="color:red;">Alice</mark> and then creates a personal bit share $$\textcolor{green}{s^b_j}$$ such that $$\textcolor{red}{s^a_j}\oplus \textcolor{green}{s^b_j}=\textcolor{green}j$$.

Therefore a single gate will be portrayed as so by each party:

<figure><img src="../.gitbook/assets/image (9).png" alt=""><figcaption><p>A single boolean gate implemented across 2 parties <mark style="color:red;">Alice</mark> and <mark style="color:green;">Bob</mark></p></figcaption></figure>

Here, we can see that the output of all parties will be XOR-ed together for the final result between all parties.

### Boolean Gates <a href="#id-66c1" id="id-66c1"></a>

Since [a full boolean circuit can be created with XOR and AND gates](yaos-garbled-circuits.md#optimizations-on-yao), let’s see how XOR and AND gates can be implemented in the 2-party GMW system.

### XOR <a href="#id-1e57" id="id-1e57"></a>

A XOR gate is extremely easy to implement in this scenario. A XOR gate means that our $$k$$ output above is $$(\textcolor{red}i\oplus\textcolor{green}j)=(\textcolor{red}{s_i^a}\oplus \textcolor{green}{s_i^b})\oplus(\textcolor{red}{s_j^a}\oplus \textcolor{green}{s_j^b})=(\textcolor{red}{s_i^a}\oplus \textcolor{red}{s_j^a})\oplus(\textcolor{green}{s_i^b}\oplus \textcolor{green}{s_j^b})$$. This means that each party can compute the XOR on their own shares and the final output of each party can be XOR-ed together to get the total XOR result.

<figure><img src="../.gitbook/assets/image (10).png" alt=""><figcaption><p>A single XOR gate implemented in 2-party GMW</p></figcaption></figure>

### AND

Implementing an AND gate is a bit more complicated. Since the $$k$$ output for an AND gate should be $$(\textcolor{red}i\land\textcolor{green}j)=(\textcolor{red}{s_i^a}\oplus \textcolor{green}{s_i^b})\land(\textcolor{red}{s_j^a}\oplus \textcolor{green}{s_j^b})$$, this means that each party can’t compute a portion of the AND gate output by themselves. Instead, we’ll first transform $$(\textcolor{red}{s_i^a}\oplus \textcolor{green}{s_i^b})\land(\textcolor{red}{s_j^a}\oplus \textcolor{green}{s_j^b})$$ as per the following logic equivalence statement:

$$
\begin{align*}k &=(\textcolor{red}i\land\textcolor{green}j)\\&=(\textcolor{red}{s_i^a}\oplus \textcolor{green}{s_i^b})\land(\textcolor{red}{s_j^a}\oplus \textcolor{green}{s_j^b})\\&=\bm((\textcolor{red}{s_i^a}\land \textcolor{red}{s_j^a})\oplus(\textcolor{green}{s_i^b}\land \textcolor{green}{s_j^b})\bm)\oplus\bm((\textcolor{red}{s_i^a}\land \textcolor{green}{s_j^b})\oplus(\textcolor{green}{s_i^b}\land \textcolor{red}{s_j^a})\bm)\end{align*}
$$

Looking closely at this resulting statement, we see that the former half’s parts $$(\textcolor{red}{s_i^a}\land \textcolor{red}{s_j^a})$$ and $$(\textcolor{green}{s_i^b}\land \textcolor{green}{s_j^b})$$ can be easily computed by the respective parties <mark style="color:red;">Alice</mark> and <mark style="color:green;">Bob</mark> individually, yet the latter half $$(\textcolor{red}{s_i^a}\land \textcolor{green}{s_j^b})\oplus(\textcolor{green}{s_i^b}\land \textcolor{red}{s_j^a})$$ appears to require some communication between the two parties.

To communicate the latter half between the parties, we set one party as the sender (<mark style="color:red;">Alice</mark>) and one party as the receiver (<mark style="color:green;">Bob</mark>).

<mark style="color:red;">Alice</mark> knows her own shares $$\textcolor{red}{s_i^a}$$ and $$\textcolor{red}{s_j^a}$$, but doesn’t know <mark style="color:green;">Bob’s</mark> shares $$\textcolor{green}{s_i^b}$$ and $$\textcolor{green}{s_j^b}$$; however, <mark style="color:red;">Alice</mark> still knows that each of <mark style="color:green;">Bob’s</mark> shares is either $$0$$ or $$1$$. Knowing this, <mark style="color:red;">Alice</mark> can create 4 potential outcomes of $$(\textcolor{red}{s_i^a}\land \textcolor{green}{s_j^b})\oplus(\textcolor{green}{s_i^b}\land \textcolor{red}{s_j^a})$$ using the different permutations of $$\textcolor{green}{s_i^b}$$ and $$\textcolor{green}{s_j^b}$$ as $$(0,0),(0,1),(1,0),(1,1)$$. The following table shows the possible outcomes if <mark style="color:red;">Alice’s</mark> shares of $$\textcolor{red}{s_i^a}$$ and $$\textcolor{red}{s_j^a}$$ were $$0$$ and $$1$$, respectively.

<figure><img src="../.gitbook/assets/image (11).png" alt=""><figcaption><p>Potential outcomes of the logical statement given <span class="math">\textcolor{red}{s_i^a}=0</span> and <span class="math">\textcolor{red}{s_j^a}=1</span></p></figcaption></figure>

Next, <mark style="color:red;">Alice</mark> creates a random bit $$\textcolor{red}r$$ (let’s say $$\textcolor{red}r=0$$ in our example) and encrypts all the possible outcomes by computing them with $$\oplus \textcolor{red}r$$.

<figure><img src="../.gitbook/assets/image (12).png" alt=""><figcaption><p>Potential outcomes of the logical statement <span class="math">\oplus \textcolor{red}r</span> given <span class="math">\textcolor{red}{s_i^a}=0</span> and <span class="math">\textcolor{red}{s_j^a}=1</span></p></figcaption></figure>

Finally, <mark style="color:red;">Alice</mark> sends this “Encrypted Possible Outcomes” table to a 1-out-of-4 oblivious transfer (OT) protocol. On the other side, <mark style="color:green;">Bob</mark> inputs their shares $$\textcolor{green}{s_i^b}=0$$ and $$\textcolor{green}{s_j^b}=1$$ to the same OT protocol. Through OT, <mark style="color:green;">Bob</mark> obtains the value of $$\textcolor{red}r\oplus\bm((\textcolor{red}{s_i^a}\land \textcolor{green}{s_j^b})\oplus(\textcolor{green}{s_i^b}\land \textcolor{red}{s_j^a})\bm)$$ specific to his shares’ values without learning of the other three encrypted possible outcomes, and <mark style="color:red;">Alice</mark> does not know which value <mark style="color:green;">Bob</mark> has obtained.

<figure><img src="../.gitbook/assets/Screenshot 2025-01-17 at 3.12.45 PM.png" alt=""><figcaption><p>1-out-of-4 OT protocol between <mark style="color:red;">Alice</mark> and <mark style="color:green;">Bob</mark> for <span class="math">\textcolor{red}r\oplus\bm((\textcolor{red}{s_i^a}\land \textcolor{green}{s_j^b})\oplus(\textcolor{green}{s_i^b}\land \textcolor{red}{s_j^a})\bm)</span></p></figcaption></figure>

As a result of <mark style="color:red;">Alice’s</mark> computation and interaction with <mark style="color:green;">Bob</mark> in a 1-out-of-4 OT protocol, the AND gate results of both parties can now be determined. <mark style="color:red;">Alice</mark> sets her AND gate result as $$(\textcolor{red}{s_i^a}\land \textcolor{red}{s_j^a})$$ XOR-ed her random bit $$\textcolor{red}r$$, and <mark style="color:green;">Bob</mark> sets his AND gate output as $$(\textcolor{green}{s_i^b}\land \textcolor{green}{s_j^b})$$ XOR-ed the OT result $$\textcolor{red}r\oplus\bm((\textcolor{red}{s_i^a}\land \textcolor{green}{s_j^b})\oplus(\textcolor{green}{s_i^b}\land \textcolor{red}{s_j^a})\bm)$$ he received. Therefore, as seen below, if the results of both parties’s AND gates are now XOR-ed together for the final logical total, the random bit $$\textcolor{red}r$$ is cancelled out, and $$(\textcolor{red}i\land\textcolor{green}j)$$ is successfully computed.

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption><p>A single AND gate implemented in 2-party GMW</p></figcaption></figure>

## Boolean GMW Protocol Summary <a href="#a7fc" id="a7fc"></a>

Thus, the whole process of GMW can be summed up as follows:

1. All parties share their input bits’ secret shares with each other.
2. All parties compute the circuit separately.\
   a. If a XOR gate is met, each party XORs their own input shares.\
   b. If an AND gate is met, each party communicates with all other parties with 1-out-of-4 OT and XORs the results with the AND of their own input shares.
3. Once all parties are finished computing the circuit, the end results of each party are all XOR-ed together for the final multi-party computation result.

This step-by-step process holds even when GMW is implemented for $$n$$-parties (where $$n \geq 2$$).

## $$n$$-parties Boolean GMW <a href="#id-36eb" id="id-36eb"></a>

Let’s view how the steps would look for the $$n$$-party version of GMW.

### Section 1. <a href="#id-4890" id="id-4890"></a>

> _"All parties share their input bits’ secret shares with each other."_

Here, we follow the same process of how <mark style="color:red;">Alice</mark> and <mark style="color:green;">Bob</mark> shared their secret bit shares with each other.

Given a random input bit $$b$$ known by Party 1 ($$P_1$$), $$P_1$$ creates $$n-1$$ random bit shares $$s_b^2,s_b^3, \dots,s_b^{n-1},s_b^n$$ and sends these to the other parties $$P_2,P_3, \dots ,P_{n-1},P_n$$. Subsequently, $$P_1$$ creates their own bit share $$s_b^1$$ such that $$b=s_b^1\oplus s_b^2\oplus s_b^3\oplus \cdots \oplus s_b^{n-1}\oplus s_b^n$$.

This bit sharing process is thus done for all input bits from all parties, ensuring that every party holds a share of an input bit in order to compute the circuit individually.

Our diagrams will now extend on our 2-party example and add in <mark style="color:purple;">Carol</mark> as a 3rd party to <mark style="color:red;">Alice</mark> and <mark style="color:green;">Bob</mark>.

<figure><img src="../.gitbook/assets/image (14).png" alt=""><figcaption><p>A single boolean gate implemented across n parties <mark style="color:red;">Alice</mark>, <mark style="color:green;">Bob</mark>, <mark style="color:purple;">Carol</mark>,...</p></figcaption></figure>

### Section 2a.

> "If a XOR gate is met, each party XORs their own input shares."

Indeed, like the 2-party explanation, each party will XOR their own shares together for their personal XOR operation result. Subsequently, if all parties’ results are XOR-ed together, the final XOR multi-party computation result is revealed.

<figure><img src="../.gitbook/assets/image (15).png" alt=""><figcaption><p>A single XOR gate implemented in <span class="math">n</span>-parties GMW</p></figcaption></figure>

### Section 2b.

> "If an AND gate is met, each party communicates with all other parties with 1-out-of-4 OT and XORs the results with the AND of their own input shares."

Our logical equivalence statement used in the 2-party version can be generalized to the following (inspiration from [here](https://securecomputation.org/docs/ch3-fundamentalprotocols.pdf)):

$$
\begin{align*}
k &= (\textcolor{red}{i} \land \textcolor{green}{j}) \\
  &= (\textcolor{red}{s_i^a} \oplus \textcolor{green}{s_i^b} \oplus \textcolor{blueviolet}{s_i^c} \oplus \cdots)\land (\textcolor{red}{s_j^a} \oplus \textcolor{green}{s_j^b} \oplus \textcolor{blueviolet}{s_j^c} \oplus \cdots) \\
  &= \left( \bigoplus_{x}^n ({s_i^x} \land {s_j^x}) \right)\oplus \left( \bigoplus_{x \neq y} ({s_i^x} \land {s_j^y}) \right)\\
  &= \underline{(\textcolor{red}{s_i^a} \land \textcolor{red}{s_j^a})
}\oplus (\textcolor{red}{s_i^a} \land \textcolor{green}{s_j^b})
\oplus (\textcolor{red}{s_i^a} \land \textcolor{blueviolet}{s_j^c})
\oplus \cdots
\oplus (\textcolor{green}{s_i^b} \land \textcolor{red}{s_j^a})
\oplus \underline{(\textcolor{green}{s_i^b} \land \textcolor{green}{s_j^b})}
\oplus (\textcolor{green}{s_i^b} \land \textcolor{blueviolet}{s_j^c})
\oplus \cdots
\oplus (\textcolor{blueviolet}{s_i^c} \land \textcolor{red}{s_j^a})
\oplus (\textcolor{blueviolet}{s_i^c} \land \textcolor{green}{s_j^b})
\oplus \underline{(\textcolor{blueviolet}{s_i^c} \land \textcolor{blueviolet}{s_j^c})}
\oplus \cdots
\end{align*}
$$

As with our 2-party example, we can see that for the last statement, the sections underlined can be computed by an individual party on their own (“...AND of their own input shares”), while the sections not underlined require communication between multiple parties. More specifically, we can see that the latter sections can be grouped together like so:

$$
\left((\textcolor{red}{s_i^a}\land \textcolor{green}{s_j^b})\oplus(\textcolor{green}{s_i^b}\land \textcolor{red}{s_j^a})\right)\oplus\left((\textcolor{green}{s_i^b}\land \textcolor{blueviolet}{s_j^c})\oplus(\textcolor{blueviolet}{s_i^c}\land \textcolor{green}{s_j^b})\right)\oplus\left((\textcolor{red}{s_i^a}\land \textcolor{blueviolet}{s_j^c})\oplus(\textcolor{blueviolet}{s_i^c}\land \textcolor{red}{s_j^a})\right)\oplus\cdots
$$

The first portion $$(\textcolor{red}{s_i^a}\land \textcolor{green}{s_j^b})\oplus(\textcolor{green}{s_i^b}\land \textcolor{red}{s_j^a})$$ is the same statement used for OT between <mark style="color:red;">Alice</mark> and <mark style="color:green;">Bob</mark> in our 2-party example. Subsequently, $$(\textcolor{green}{s_i^b}\land \textcolor{blueviolet}{s_j^c})\oplus(\textcolor{blueviolet}{s_i^c}\land \textcolor{green}{s_j^b})$$ requires OT between <mark style="color:green;">Bob</mark> and <mark style="color:purple;">Carol</mark>, while $$(\textcolor{red}{s_i^a}\land \textcolor{blueviolet}{s_j^c})\oplus(\textcolor{blueviolet}{s_i^c}\land \textcolor{red}{s_j^a})$$ requires OT between <mark style="color:red;">Alice</mark> and <mark style="color:purple;">Carol</mark> (“...each party communicates with all other parties with 1-out-of-4 OT…”).

Therefore, the final AND result between $$n$$-parties will be each parties’ individual calculations XOR-ed with all OT results with all other parties. Since displaying all OT pairs between all parties proves difficult through a diagram, I’ve shown only 3 parties instead in the following figure:

<figure><img src="../.gitbook/assets/image (16).png" alt=""><figcaption><p>A single AND gate implemented for 3-party GMW</p></figcaption></figure>

### Total AND Gate Costs

Going on a slight tangent, let’s think of the communication cost between parties for a whole circuit. Remember that XOR gates don’t have any communication cost.

<figure><img src="../.gitbook/assets/image (18).png" alt=""><figcaption><p>Example circuit with XOR and AND gates</p></figcaption></figure>

Since all AND gates at the same level can be run in parallel (such as AND gates 1 and 2 in the figure above), yet all AND gates in succession must be run in order (as in AND gate 1 must be computed before AND gate 3 in the figure above), the total number of rounds of communication between parties equals the depth of the circuit in terms of AND gates (depth = 2 in the figure above). Additionally, each AND gate requires $$C_2^n = \frac{n(n-1)}{2}$$ total OT processes to take place (total number of unique party pairs). This means that in total, there are (depth of the circuit in terms of AND gates) $$\times \frac{n(n-1)}{2}$$ total OTs occurring in a given circuit, a significant communication cost for a circuit with many AND gates.

### Section 3.

> "Once all parties are finished computing the circuit, the end results of each party are all XOR-ed together for the final multi-party computation result."

As explained with this sentence, the final results of the circuit for every party are XOR-ed for the total result. Note that this means each party does not expose their intermediate results to the other parties.

## Arithmetic Circuits

While I showed the boolean version of GMW, this protocol also works for **arithmetic circuits**. Instead of XOR and AND gates, arithmetic circuits use **addition** and **multiplication** gates. Additionally, while the secret shares for an input $$b$$ are calculated as $$b=s_b^1\oplus s_b^2\oplus s_b^3\oplus\cdots\oplus s_b^{n-1}\oplus s_b^n$$ for $$n$$ parties for the boolean version, for the arithmetic version, $$b$$ equals $$s_b^1+ s_b^2+ s_b^3+ \cdots + s_b^{n-1}+ s_b^n$$. Therefore, the addition operation will become free (like how XOR gates are free) and the multiplication operation requires communication between parties (similar to AND gates).

## Conclusion <a href="#id-4f71" id="id-4f71"></a>

In total, GMW stands as a notable $$n$$-party MPC protocol based around the concept of secret shares with free XOR gates and costly AND gates. Communication between parties must occur during the start of the protocol, at every AND gate, and at the end of the circuit computation.

> Written by [Ashley Jeong](https://app.gitbook.com/u/wyEMbFN1Kybygv7v1pKpc2P8q6D2 "mention") from [A41](https://www.a41.io/)
