---
description: 'Presentation: https://youtu.be/zNWatOe9Hmo'
---

# Cairo

## Introduction

Let us assume we are designing a **zkDSL**. There are two major architectural approaches to consider.

***

### 1. ASIC Approach

This is the _Circom_-style approach, where a **program** is taken as input and compiled into a **circuit**.\
The resulting circuit can prove only the execution of that **specific program**, making it essentially an **application-specific circuit (ASIC)**.

In other words, whenever the program changes, a **new circuit** must be generated.

***

### 2. CPU (von Neumann) Approach

This is the _Cairo_-style approach, where a **single universal circuit models a CPU architecture**.\
In this model, a program is loaded into memory, and the program counter (PC) fetches, decodes, and executes instructions sequentially.

Thus, the same circuit can verify the execution of **different programs**.

> The name **Cairo** originates from **C**PU **AIR**&#x20;

***

When designing a DSL for **zero-knowledge proving**, the two most critical factors are **programmability** and **efficiency**.

Let us now explore **why Cairo chose the CPU-based (von Neumann) approach** to achieve this balance.

## Description

### ASIC vs CPU Approach

#### ASIC Approach

For example, to prove that $$x \ne y$$, the statement can be **arithmetized** as follows:

$$
\exists a : (x - y) \cdot a = 1
$$

In this **ASIC approach**, a circuit is generated specifically for a single program, which makes it **highly efficient** for that particular computation.

However, inefficiencies arise when handling **conditional statements** and **loops**.

***

#### Conditional Statement

A conditional statement can be expressed using a **selector variable** $$s$$:

$$
s \times \mathsf{arith_{true}} + (1 - s) \times \mathsf{arith_{false}}
$$

At runtime, only one branch is actually executed, but in the circuit, both branches must be represented as algebraic constraints.

***

#### Loop Statement

A loop is typically unrolled up to a predefined **maximum bound** $$B$$, and the loop body is repeated $$B$$ times during arithmetization.

Even if the loop terminates earlier at runtime, all $$B$$ iterations are still represented as constraints — leading to redundant computations and reduced efficiency.

***

#### Summary

As a result, the **ASIC approach** has the following limitations:

* **Inefficient:** unnecessary constraints arise from conditionals and loops
* **Non-reusable:** every time the program changes, the circuit must be recompiled

***

#### CPU Approach

In contrast, the **CPU approach** introduces some overhead for **instruction fetching and decoding**, but Cairo minimizes this overhead through several architectural design choices.

***

#### 1. Metrics-based Instruction Set Design

Cairo designs its instruction set to minimize **the number of trace cells used**.

* For example, suppose we define two instruction sets: $$\mathsf{ISA_A}$$, which includes a specific instruction $$\mathsf{instr}$$, and $$\mathsf{ISA_B}$$, which does not.\
  Let $$n_i$$ denote the number of **trace cells per cycle** for instruction set $$\mathsf{ISA}_i$$, and $$k_i$$ denote the **total number of cycles** required to execute a program $$P$$. Then, the **total number of trace cells** produced during execution is given by $$n_i \times k_i$$.
* In general, Cairo selects an instruction set such that:

$$
k_A \times n_A < k_B \times n_B
$$

This ensures that, on average, the **overall proof cost** is minimized. In other words, the instruction set is chosen to optimize **average proving efficiency**.

***

#### 2. Low-degree AIR Constraint

Cairo’s AIR (Algebraic Intermediate Representation) constraints are designed to remain mostly **quadratic** (degree 2). When a constraint of degree 3 is unavoidable, additional **trace cells** are introduced to reduce it back to degree 2 through algebraic transformation.

This approach ensures that the **STARK proof system** operates efficiently, maintaining low-degree polynomial constraints throughout the entire trace.

***

#### 3. Builtins

If new instructions are added but rarely used, they introduce unnecessary overhead. Conversely, implementing every primitive operation purely as Cairo instructions would also be inefficient.

To resolve this trade-off, Cairo introduces the concept of **builtins**.

Each builtin has its own **independent memory segment**, and the Cairo program simply **writes values** into this segment to trigger the corresponding operation automatically under the AIR constraints.

For example, using a **range-check builtin**, one can efficiently verify that a value lies within a specific range — without introducing a new instruction.

Thus, Cairo’s builtins enable **operation-level modularity** that allows efficient computation through **memory-based invocation** rather than instruction expansion.

<figure><img src="../.gitbook/assets/Screenshot 2025-10-10 at 4.35.39 PM.png" alt=""><figcaption></figcaption></figure>

***

### Nondeterminism

Consider an NP-complete problem, such as SAT, and consider the following two algorithms:

**Algorithm A** takes a SAT instance and a candidate assignment as input, and returns `True` if the assignment satisfies the formula.

**Algorithm B** takes a SAT instance and iterates over **all possible assignments**. If it finds one that satisfies the formula, it returns `True`; otherwise, it returns `False`.

If a **prover** wants to convince a **verifier** that a certain SAT formula is satisfiable, either algorithm can be used — since if **either one** returns `True`, the formula is indeed satisfiable.

Of course, **Algorithm A** is much more efficient, so in practice, this approach is preferred for proof construction.

This style of proving — where the prover demonstrates that **Algorithm A would return `True`** — is called **nondeterministic programming**.

In other words, instead of showing the **entire computation**, the prover can **guess a potential solution** and only **prove that the guess is correct**. This allows the proof to be much shorter and more efficient.

Cairo supports this style of nondeterministic proving through a mechanism called **“Hint.”**

For example, if the computation involves $$y = \sqrt{x}$$, the prover does not need to show the full algorithm for computing the square root. Instead, the prover can **guess** a value $$y$$ and then **prove** that $$y^2 = x$$.

In other words, Cairo focuses on proving **the correctness of results**, not the **process of computation**, thereby significantly reducing the overall proving cost.

***

### Register

In a typical physical computing system, **memory access is expensive**, so **general-purpose registers** are used to store frequently accessed values.&#x20;

However, in **Cairo**, memory access is extremely cheap (See why it's cheap later), and therefore **no general-purpose registers** are needed. All values are accessed directly through **memory cells**.

Cairo provides only **three address registers**, which serve as pointers:

* **pc**: _Program Counter_ — points to the current instruction.
* **ap**: _Allocation Pointer_ — used to allocate new memory cells during execution.
* **fp**: _Frame Pointer_ — used to access function arguments and local variables. At the start of a function, $$\mathsf{fp}$$ is initialized to the same value as $$\mathsf{ap}$$. During function execution, it remains constant, and when the function returns, it is reset to its previous value.

***

### Instruction

Cairo’s instructions operate according to the rules described in [**Section 4.5**](https://eprint.iacr.org/2021/1063.pdf#page=32\&zoom=100,150,500). An instruction occupies **one word** if it has no immediate value, and **two words** if it includes an immediate value.

<figure><img src="../.gitbook/assets/image (185).png" alt=""><figcaption></figcaption></figure>

To encode an instruction, the following constraint must hold: (I'll skip the definitions of each term)

$$
\mathsf{inst} =
\widetilde{off}_{dst}
+ 2^{16} \cdot \widetilde{off}_{op0}
+ 2^{32} \cdot \widetilde{off}_{op1}
+ 2^{48} \cdot \widetilde{f}_0
$$

Because each term must lie within the range $$[0, 2^{16})$$ a **permutation range-check** (see [Section 9.9](https://eprint.iacr.org/2021/1063.pdf#page=57\&zoom=100,200,500)) is required to enforce this constraint.

For this representation to be valid, Cairo also requires that:

$$
\mathrm{char}(\mathbb{F}) > 2^{63}
$$

This ensures that all 63-bit instruction words fit within the field $$\mathbb{F}$$ without wrap-around or overflow.

***

### Cairo Machine

The **Cairo Machine** is not a general-purpose computer designed to perform arbitrary computation. Instead, its purpose is to **verify** the correctness of a computation.

Cairo defines two kinds of machines:

1. **Deterministic Cairo Machine**
2. **Non-deterministic Cairo Machine**

Here, let $$\mathbb{F}_P = \mathbb{Z}/P$$, and $$\mathbb{F}$$  is an extension field of $$\mathbb{F}_P$$.

***

#### Deterministic Cairo Machine

A Deterministic Cairo Machine takes the following inputs:

1. The number of steps $$T \in \mathbb{N}$$
2. A memory function $$m: \mathbb{F} \to \mathbb{F}$$
3. A sequence of states $$S = (S_0, S_1, \dots, S_T)$$, where each state is defined as $$S_i = (\mathsf{pc}_i, \mathsf{ap}_i, \mathsf{fp}_i) \in \mathbb{F}^3$$

If, for every $$i$$, the transition $$S_i \to S_{i+1}$$ is **valid** according to Cairo’s transition rules, the machine outputs **“accept”**; otherwise, it outputs **“reject.”**

***

#### Example

Suppose the following input is given:

* $$T = 10$$
* Initial state $$S_0 = (0, 5, 5)$$
* Initial memory function $$m$$:

| i | m(i)               |
| - | ------------------ |
| 0 | 0x48307ffe7fff8000 |
| 1 | 0x010780017fff7fff |
| 2 | -1                 |
| 3 | 1                  |
| 4 | 1                  |

***

**1. First Instruction Execution**

$$
m(\mathsf{pc}_0) = m(0) = \text{0x48307ffe7fff8000}
$$

Decoded fields:

| Field                    | Bit Range | Value |
| ------------------------ | --------- | ----- |
| $$\mathsf{off_{dst}}$$   | 0–16      | 0     |
| $$\mathsf{off_{op0}}$$   | 16–32     | -1    |
| $$\mathsf{off_{op1}}$$   | 32–48     | -2    |
| $$\mathsf{dst_{reg}}$$   | 48–49     | 0     |
| $$\mathsf{op0_{reg}}$$   | 49–50     | 0     |
| $$\mathsf{op1_{src}}$$   | 50–53     | 4     |
| $$\mathsf{res_{logic}}$$ | 53–55     | 1     |
| $$\mathsf{pc_{update}}$$ | 55–58     | 0     |
| $$\mathsf{ap_{update}}$$ | 58–60     | 2     |
| $$\mathsf{opcode}$$      | 60–63     | 4     |

According to [Section 4.5](https://eprint.iacr.org/2021/1063.pdf#page=32\&zoom=100,150,500) transition rules:

* $$\mathsf{op0} = m(\mathsf{ap_0 + off_{op0}}) = m(\mathsf{ap_0} - 1) = m(4)$$
* $$\mathsf{op1} = m(\mathsf{ap_1 + off_{op1}}) = m(\mathsf{ap_1} - 2) = m(3)$$
* $$\mathsf{res} = \mathsf{op0} + \mathsf{op1} = m(4) + m(3)$$
* $$\mathsf{dst} = m(\mathsf{ap_0 + off_{dst}}) = m(\mathsf{ap_0} + 0) = m(5)$$
* $$\textcolor{red}{\mathsf{assert(res = dst)}}$$
* $$\mathsf{pc_1} = \mathsf{pc_0} + 1$$
* $$\mathsf{ap_1} = \mathsf{ap_0} + 1$$
* $$\mathsf{fp_1} = \mathsf{fp_0}$$

Hence, $$\textcolor{red}{m(5) = m(4) + m(3)}$$, and the state updates to $$S_1 = (1, 6, 5)$$.

***

**2. Second Instruction Execution**

$$
m(\mathsf{pc}_1) = m(1) = \text{0x010780017fff7fff}
$$

Decoded fields:

| Field                    | Bit Range | Value |
| ------------------------ | --------- | ----- |
| $$\mathsf{off_{dst}}$$   | 0–16      | -1    |
| $$\mathsf{off_{op0}}$$   | 16–32     | -1    |
| $$\mathsf{off_{op1}}$$   | 32–48     | 1     |
| $$\mathsf{dst_{reg}}$$   | 48–49     | 1     |
| $$\mathsf{op0_{reg}}$$   | 49–50     | 1     |
| $$\mathsf{op1_{src}}$$   | 50–53     | 1     |
| $$\mathsf{res_{logic}}$$ | 53–55     | 0     |
| $$\mathsf{pc_{update}}$$ | 55–58     | 2     |
| $$\mathsf{ap_{update}}$$ | 58–60     | 0     |
| $$\mathsf{opcode}$$      | 60–63     | 0     |

Transition rules:

* $$\mathsf{op0} = m(\mathsf{fp_1 + off_{op0}}) = m(\mathsf{fp_1} - 1) = m(4)$$
* $$\mathsf{op1} = m(\mathsf{pc_1 + off_{op1}}) = m(\mathsf{pc_1} + 1) = m(2)$$
* $$\mathsf{res} = \mathsf{op1} = m(2)$$
* $$\mathsf{dst} = m(\mathsf{fp_1 + off_{dst}}) = m(\mathsf{fp_1} - 1) = m(2)$$
* $$\mathsf{pc_2} = \mathsf{pc_1} + \mathsf{res} = \mathsf{pc_1} + m(2)$$
* $$\mathsf{ap_2} = \mathsf{ap_1}$$
* $$\mathsf{fp_2} = \mathsf{fp_1}$$

Since $$m(2) = -1$$, we have $$\mathsf{pc_2} = 0$$, which means the program counter loops back to the beginning.

Thus, this program repeatedly alternates between $$\mathsf{pc} = 0$$ and $$\mathsf{pc} = 1.$$

***

**3. Memory Configuration for “accept”**

To be accepted, memory should be given as follows: (and it's <mark style="color:red;">**(continuous)**</mark> <mark style="color:red;">**read-only**</mark> memory)

| i | m(i)               |
| - | ------------------ |
| 0 | 0x48307ffe7fff8000 |
| 1 | 0x010780017fff7fff |
| 2 | -1                 |
| 3 | 1                  |
| 4 | 1                  |
| 5 | 2                  |
| 6 | 3                  |
| 7 | 5                  |
| 8 | 8                  |
| 9 | 13                 |

***

#### Non-deterministic Cairo Machine

A **Non-deterministic Cairo Machine** takes the following inputs:

1. The number of steps $$T \in \mathbb{N}$$
2. A <mark style="color:red;">**partial**</mark>**&#x20;memory function** $$m^*: A^* \to \mathbb{F}$$, where $$A^* \subseteq \mathbb{F}_P$$
3. Initial and final values of program counter and allocation pointer:

$$
\mathsf{pc_I}, \mathsf{pc_F}, \mathsf{ap_I}, \mathsf{ap_F}
$$

(The initial frame pointer $$\mathsf{fp}$$ is set to $$\mathsf{ap_I}$$.)

***

If we define the initial and final states as:

$$
S_0 = (\mathsf{pc_I}, \mathsf{ap_I}, \mathsf{ap_I}), \quad
S_T = (\mathsf{pc_F}, \mathsf{ap_F}, *)
$$

then, if there exists a **full memory extension** $$m: \mathbb{F} \to \mathbb{F}$$ of $$m^*$$ such that a valid transition sequence $$S_0 \to S_1 \to \dots \to S_T$$ exists, the machine **accepts**; otherwise, it **rejects**. (This is <mark style="color:red;">**non-determinism**</mark>!)

***

Here, $$m^*$$ represents the **public memory**, which includes:

1. **Program bytecode**
   * $$m^*(\mathsf{prog_{base}} + i) = b_i$$ for $$i \in [0, |b|)$$
   * $$\mathsf{pc_I} = \mathsf{prog_{base}} + \mathsf{prog_{start}}$$
   * $$\mathsf{pc_F} = \mathsf{prog_{base}} + \mathsf{prog_{end}}$$
2. **Program input/output** necessary for verification

***

A **STARK prover** then constructs a proof asserting that:

“The nondeterministic Cairo Machine **accepts** given the input $$(T, m^*, \mathsf{pc_I}, \mathsf{pc_F}, \mathsf{ap_I}, \mathsf{ap_F})$$.”

***

### Cairo Runner

The **Cairo Runner** executes a compiled Cairo program and produces the following outputs:

* Inputs that cause the **Non-deterministic Cairo Machine** to accept: $$(T, m^*, \mathsf{pc_I}, \mathsf{pc_F}, \mathsf{ap_I}, \mathsf{ap_F})$$
* Inputs that cause the **Deterministic Cairo Machine** to accept: $$(T, m, S)$$

***

While the Cairo Machine supports **random-access memory** by definition, an efficient **Cairo AIR** implementation requires that **memory accesses be continuous** in address order.

To achieve this, the **Cairo Runner** introduces the concept of **relocatable memory segments**.

***

#### Memory Segment Management

Whenever memory allocation is required during program execution, the Cairo Runner creates and assigns a new **memory segment**.

The **size** of each segment does **not** need to be specified in advance.

After program execution finishes, the Runner performs a **relocation phase**, where each segment is assigned a concrete **base address** so that the entire memory space can be **linearized** into one continuous address range.

***

#### Relocation Is Not Proven

The relocation process is **not included in the proof**. It is performed locally by the Cairo Runner and does not appear in the STARK proof itself.

In a **read-write memory model**, this could be problematic — a malicious prover could intentionally create **overlapping segments**, causing one segment’s write operation to inadvertently change the value in another segment.

***

#### Why This Is Safe in Cairo

In Cairo, memory is **read-only (immutable)**. Once a value is written, it cannot be modified.

Therefore, even if two segments overlap, it does not matter **which segment** a value is read from — as long as the value itself is consistent.

In other words, as far as the Cairo execution model is concerned, **overlapping segments** and **non-overlapping segments** are **indistinguishable**.

***

### Memory Layout

#### Function Call Stack

* **Function arguments** provided by the caller are stored at memory addresses $$[\mathsf{fp} - 3], [\mathsf{fp} - 4], \dots$$
* **Pointer to the caller’s frame** is stored at $$[\mathsf{fp} - 2]$$
* **Return address** — the instruction to be executed after the function returns — is stored at $$[\mathsf{fp} - 1]$$
* **Local variables** allocated by the function are stored at $$[\mathsf{fp}], [\mathsf{fp} + 1], \dots$$

Additionally, **function return values** are stored in memory at $$[\mathsf{ap} - 1], [\mathsf{ap} - 2], \dots$$, where $$\mathsf{ap}$$ is the value of the allocation pointer at the end of the function.

<figure><img src="../.gitbook/assets/image (184).png" alt=""><figcaption></figcaption></figure>

The figure above illustrates the call stack for the following code:

```
f:
  call g
  ret
g:
  call h
  call h
  ret
h:
  ret
```

***

#### Nondeterministic Continuous Read-Only Random-Access Memory

#### **Description**

Cairo implements a **Nondeterministic Continuous Read-Only Random-Access Memory** model. In this model, memory is **read-only** — once written, values cannot be modified — and memory accesses must be **contiguous**.

What does “contiguous” mean? It means:

1. If $$x < y$$,
2. and there is a memory access at $$x$$,
3. and there is a memory access at $$y$$,

then there must also exist a memory access at every address $$a$$ such that $$x < a < y$$.

In this model, the **cost** depends on the **number of memory accesses**, not on how much total memory is used. Therefore, developers do not need to worry about **freeing** or **reusing** memory cells.

***

#### Constraints

Let

$$
L_1 = \{(a_i, v_i)\}_{i=0}^{n-1}, \quad L_2 = \{(a_i', v_i')\}_{i=0}^{n-1}
$$

be two lists of memory accesses.

The following constraints ensure Cairo’s **continuous, read-only memory** behavior:

*   **Continuity**

    $$
    (a_{i+1}' - a_i')(a_{i+1}' - a_i' - 1) = 0
    \quad \text{for all } i \in [0, n-1)
    $$

    → Ensures consecutive addresses differ by exactly 1.
*   **Single-valuedness**

    $$
    (v_{i+1}' - v_i')(a_{i+1}' - a_i' - 1) = 0
    \quad \text{for all } i \in [0, n-1)
    $$

    → Ensures that if two accesses share the same address, their values must be identical.
*   **Permutation**

    The permutation argument enforces that the two memory access lists correspond to the same mapping.

    *   **Initial value**

        $$
        (z - (a_0' + \alpha \times v_0')) \times p_0
          = z - (a_0 + \alpha \times v_0)
        $$
    *   **Final value**

        $$
        p_{n-1} = 1
        $$
    *   **Cumulative product step**

        $$
        (z - (a_{i}' + \alpha \times v_{i}')) \times p_i
          = z - (a_i + \alpha \times v_i)
        \quad \text{for all } i \in [1, n)
        $$

If these constraints hold, $$L_1$$ forms a **continuous read-only memory**.

***

#### Summary

* Each memory access uses **five trace cells**: $$(a_i, v_i, a_{i+1}, v_{i+1}, z)$$
* Each instruction performs **three memory accesses**: $$\mathsf{op0}, \mathsf{op1}, \mathsf{dst}$$

***

### Public Memory

#### Description

The **program’s input, output, and bytecode** that are required for verification must be **shared with the verifier**. These are provided as part of the input $$m^*$$ to the **Non-deterministic Cairo Machine**.

The **prover** must then demonstrate the existence of an **extended memory function** $$m$$ that is consistent with $$m^*$$.

***

#### Constraints

* Append $$|A^*|$$ pairs of $$(0, 0)$$ to $$L_1$$, and append $$\{(a, m^*(a))\}_{a \in A^*}$$ to $$L_2$$.
* Modify the **final value** constraint as follows:

$$
\frac{\prod_{a \in A^*} (z - (a + \alpha \times m^*(a)))}{z^{|A^*|}}
$$

This ensures that the public memory $$m^*$$ is properly incorporated into the permutation argument used to verify memory consistency.

***

### Program Input & Output

* **Program input:** The data that serves as the program’s input. This data **does not need to be shared** with the verifier.
* **Program output:** The data produced during program execution. This data **is shared** with the verifier.

***

For example, suppose the program input is $$n$$, and the goal is to compute the $$n$$-th Fibonacci number. Depending on how the **program output** is defined, the proof statement differs:

| Output Contents      | Meaning of the Statement                                                                          |
| -------------------- | ------------------------------------------------------------------------------------------------- |
| both $$n$$ and $$y$$ | “The $$n$$-th Fibonacci number is $$y$$.”                                                         |
| only $$y$$           | “I know some $$n$$ such that the $$n$$-th Fibonacci number is $$y$$.” (but $$n$$ is not revealed) |
| only $$n$$           | “I have computed the $$n$$-th Fibonacci number.” (but the result is hidden)                       |

***

### Conclusion

Cairo demonstrates how a **universal, proof-friendly CPU architecture** can bridge the gap between programmability and efficiency in zero-knowledge proving systems.

Unlike ASIC-style zkDSLs, which generate a dedicated circuit for each program, Cairo models computation at the CPU level — allowing **any program** to be proven on the **same proving system**.\
This shift from _“program-specific circuits”_ to _“program-verifying machines”_ is what makes Cairo fundamentally scalable and expressive.

## Reference

* [https://eprint.iacr.org/2021/1063](https://eprint.iacr.org/2021/1063)



> Written by [ryan Kim](https://app.gitbook.com/u/FEVExqcoLKVoL5siVqLSKP5TO5V2 "mention") from [A41](https://www.a41.io/)
