# Cairo M

## Introduction

Cairo M is a new zkVM proposed to address the structural limitations of the original [**Cairo VM**](cairo.md).

While Cairo was designed to be STARK-friendly, it faces critical constraints in terms of scalability and parallel proving (continuation):

*   **Non-deterministic continuous read-only memory with relocation model**

    In the original Cairo, the runner must perform a memory segment **relocation** after program execution. This design prevents the use of the **continuation** technique—where proofs can be generated as soon as each segment is ready — because the final memory addresses are only known **after** relocation is completed.
*   **Large prime field requirement**

    The design of Cairo requires a base prime field larger than $$2^{63}$$. However, modern STARK provers typically operate over **small prime fields** such as BabyBear or Mersenne31, making Cairo inefficient in contemporary proving frameworks.

***

## Background

### Stwo Lookup

All aspects of Cairo M — including memory consistency, register updates, and hash computations —\
are verified through the **Lookup Component** provided by the **Stwo framework**.

In Stwo, all lookup relations are combined into a **single global fraction sum**:

$$
\sum_i m_i \cdot \mathsf{Relation}(v_{i,0}, \dots, v_{i,n}) = 0
$$

where:

* $$m_i$$ denotes the **multiplicity (number of occurrences)** of the relation
* $$\mathsf{Relation}(v_{i,0}, \dots, v_{i,n})$$ represents the **lookup term** for that relation

When all relations are balanced within this global sum such that the total equals zero,\
it guarantees **global lookup consistency** across the entire proof.

In this model:

* $$+\mathsf{Relation(\dots)}$$ represents an **emit / produce** operation
* $$-\mathsf{Relation(\dots)}$$ represents a **consume / remove** operation

The proof system is designed so that the sum of all emitted and consumed terms equals zero,\
ensuring state consistency across all components.

***

## Description

### Memory

To support **continuation**, Cairo M must ensure that the **final memory state** of stage $$n$$\
is identical to the **initial memory state** of stage $$n + 1$$.

To achieve this, Cairo M commits the memory state using a **Merkle tree** structure.

#### Commitment

The Merkle tree is used for memory state commitment for the following reasons:

1. **Challenge-independent commitment**\
   — The initial and final root hashes alone are sufficient to ensure state consistency.
2. **Sparse memory support**\
   — Partial pruning of unused branches allows for efficient memory handling.

**Features**

* Hash function: **Poseidon2** (a STARK-friendly hash)
* Number of leaves: $$2^{30}$$ — the largest power of two smaller than $$P$$
* Leaf hashing: leaf nodes are **not hashed** to reduce computational overhead
  * Although sibling nodes may be revealed during inclusion proofs, this is not an issue for **state root commitment** purposes.

***

#### Lookup Argument

The Merkle component must prove that the root of the Merkle tree, composed of $$2^{30}$$ leaves, corresponds to the **initial or final root hash**. This is achieved by removing the parent node and adding its child nodes, expressed as a relation within the **global LogUp sum**:

$$
-\mathsf{Merkle(index, depth, parent, root)} \\
+b_{2 \cdot \mathsf{index}} \cdot \mathsf{Merkle(2 \cdot index, depth + 1, child_L, root)} \\
+b_{2 \cdot \mathsf{index + 1}} \cdot \mathsf{Merkle(2 \cdot index + 1, depth + 1, child_R, root)} \\
-\mathsf{Poseidon2(parent)} \\
+\mathsf{Poseidon2(child_L, child_R)}
$$

where:

* $$b_{2 \cdot \mathsf{index}}$$ and $$b_{2 \cdot \mathsf{index} + 1}$$ are binary flags indicating whether the left or right child exists (1 if present, 0 otherwise)
* Unused branches have multiplicity set to 0 and are automatically **pruned**

This relation ensures that the Merkle path is consistently connected through the **Poseidon2** hash function.

***

#### Memory Access

Memory accesses are expressed as a 3-tuple:

$$
\mathsf{(address, clock, value)}
$$

Here, $$\mathsf{clock}$$ is a **monotonically increasing counter** that determines the order of memory accesses.

***

#### Word Size

Although Cairo M’s memory is committed at the level of individual **field elements**, each address can group multiple leaves together to form a **multi-limb word**.

For example, if four leaves are combined into one memory word:

$$
-\mathsf{Merkle}(a, 30, v_0, \mathsf{root}) \\
-\mathsf{Merkle}(a + 1, 30, v_1, \mathsf{root}) \\
-\mathsf{Merkle}(a + 2, 30, v_2, \mathsf{root}) \\
-\mathsf{Merkle}(a + 3, 30, v_3, \mathsf{root}) \\
+\mathsf{Memory}(a, \mathsf{clock}, v_0, v_1, v_2, v_3)
$$

This is merely an example — in Cairo M, **word size is not fixed**. Because the LogUp argument ignores trailing zeros, the number of limbs per address can vary while still preserving consistency.

This concept is referred to as **lazy word size**.

For instance, types such as `u16`, `u32`, and `u64` can share the same memory address:

$$
\mathsf{Memory}(a, \mathsf{clock, value}, 0, \dots, 0)
= \mathsf{Memory}(a, \mathsf{clock, value})
$$

If necessary, a fixed word length can be explicitly defined by adding a $$\mathsf{LEN}$$ term:

$$
\mathsf{Memory}(a, \mathsf{LEN, clock, value}, \dots)
$$

***

#### Read / Write Operations

**Write Operation**

$$
-\mathsf{Memory(address, prev\_clock, prev\_value)} \\
+\mathsf{Memory(address, clock, value)} \\
+\mathsf{RangeCheck20(clock - prev\_clock - 1)}
$$

Here, $$\mathsf{RangeCheck20}(v)$$ represents a component that ensures the input value is constrained to a 20-bit range $$(0 \leq v < 2^{20})$$.

* The first term consumes the previous memory state
* The second term emits the new state
* The third term enforces the monotonic increase of $$\mathsf{clock}$$ (ordering constraint)

**Read Operation**

For a read-only operation, where $$\mathsf{prev\_value = value}$$:

$$
-\mathsf{Memory(address, prev\_clock, value)} \\
+\mathsf{Memory(address, clock, value)} \\
+\mathsf{RangeCheck20(clock - prev\_clock - 1)}
$$

By subtracting and adding memory terms in this way, Cairo M can naturally prove that the transition from the **initial state to the final state** is consistent across the entire memory.

***

#### Clock Update

$$\mathsf{RangeCheck20}$$ can only verify a 20-bit range (i.e., up to $$2^{20}$$). Therefore, if the clock difference $$\delta$$ exceeds this $$\mathsf{RC\_LIMIT}$$([**R**anche](#user-content-fn-1)[^1]**C**heck **LIMIT**), a **clock update** must be performed.

$$
-\mathsf{Memory(address, prev\_clock, prev\_value)} \\
+\mathsf{Memory(address, prev\_clock + RC\_LIMIT, prev\_value)}
$$

This process does not modify the stored value; it simply **splits a long time interval into smaller steps (step-splitting)**. In practice, the prover divides a large clock gap into multiple smaller RangeCheck blocks.

{% hint style="info" %}
I assume that $$\mathsf{RC\_LIMIT}$$ is a negative value.
{% endhint %}

***

#### Cost Analysis

Now, let’s compare the cost between the **read-write memory model** and the **read-only memory model**. Here, cost is measured in units of **trace columns**.

***

**Write Access on Read-Write Memory**

* Main trace: $$(\mathsf{address, prev\_clock, clock, prev\_value, value})$$
*   Lookup:

    $$
    -\mathsf{Memory(address, prev\_clock, prev\_value)} \\
    +\mathsf{Memory(address, clock, value)} \\
    +\mathsf{RangeCheck20(clock - prev\_clock - 1)}
    $$
* **Total:** $$5T + 3L$$

***

**Read Access on Read-Write Memory**

* Main trace: $$(\mathsf{address, prev\_clock, clock, value})$$
*   Lookup:

    $$
    -\mathsf{Memory(address, prev\_clock, value)} \\
    +\mathsf{Memory(address, clock, value)} \\
    +\mathsf{RangeCheck20(clock - prev\_clock - 1)}
    $$
* **Total:** $$4T + 3L$$

***

**Read Access on Read-Only Memory**

* Main trace: $$(\mathsf{address, value})$$
* Lookup: $$-\mathsf{Memory(address, value)}$$
* **Total:** $$2T + 1L$$

***

**Tradeoff**

Lookup columns are defined over the **secure field**, typically $$\mathsf{QM31} \approx 4\mathsf{M31}$$. Thus, we can approximate $$1L \approx 4T$$ and estimate the following overhead:

<table><thead><tr><th width="218.80206298828125">Operation Type</th><th>Overhead per Access</th></tr></thead><tbody><tr><td><strong>Write</strong></td><td><span class="math">(5T + 3L) - (2T + 1L) = 3T + 2L ≈ 11T</span></td></tr><tr><td><strong>Read</strong></td><td><span class="math">(4T + 3L) - (2T + 1L) = 2T + 2L ≈ 10T</span></td></tr></tbody></table>

For example, a `STORE` operation such as `dst = op0 + op1` requires **2 reads + 1 write**, resulting in approximately **31T** of overhead.

***

**Mitigation Strategies**

This overhead can be mitigated in the following ways:

* **In-place opcodes:** Reduce the number of read/write operations (e.g., use `x += y` instead of `x = x + y`)
* **Pre-sum optimization (**[**Section 6.1.2**](https://github.com/kkrt-labs/cairo-m/blob/ba42e4182a4678c5699fa106e4c33c1cd48d852a/docs/design.pdf)**):** Combine multiple lookup terms to cut lookup cost roughly in half

Although the overhead increases, the design offers significant advantages:

* Flexible control flow
* No need for frame duplication
* Simplified program development

Therefore, Cairo M considers this overhead a **worthwhile trade-off**.

Meanwhile, immutable data such as program bytecode can remain in a **read-only segment**, while only dynamic data is stored in **read/write memory**, allowing for more efficient overall design.

***

### Registers

The original **Cairo VM** includes three registers:

<table><thead><tr><th width="138.99481201171875">Name</th><th>Meaning</th></tr></thead><tbody><tr><td><code>pc</code></td><td>Program Counter — the address of the currently executing instruction</td></tr><tr><td><code>fp</code></td><td>Frame Pointer — the base address of the current function frame</td></tr><tr><td><code>ap</code></td><td>Allocation Pointer — a free memory pointer (the end of the contiguous memory region)</td></tr></tbody></table>

However, since Cairo M adopts a **read/write RAM model**, the concept of “free memory” is no longer needed. In other words, there is no need to dynamically append memory, so the `ap` register can be removed.

Cairo M’s initial design therefore includes only the following three registers:

<table><thead><tr><th width="139.5572509765625">Register</th><th>Role</th></tr></thead><tbody><tr><td><code>pc</code></td><td>The address of the current instruction</td></tr><tr><td><code>fp</code></td><td>The base address of the current frame</td></tr><tr><td><code>clock</code></td><td>A monotonically increasing counter used to enforce memory access order and timing constraints</td></tr></tbody></table>

***

#### The Register Model from the Prover’s Perspective

From the ZK prover’s point of view, each register is represented as a **column in the trace table**.

In other words:

* `pc`, `fp`, and `clock` each occupy one column
* Each opcode component (e.g., `StoreAdd`, `JmpRel`, etc.) consumes the current register state and produces the next one

This means that **register state transitions can also be represented using the same LogUp lookup relation** as memory.

$$
-\mathsf{Registers(prev\_pc, prev\_fp, prev\_clock)} \\
+\mathsf{Registers(pc, fp, prev\_clock + 1)}
$$

Here, the $$\mathsf{Registers}$$ relation represents a state transition of the register vector, where each opcode execution “consumes (−)” the previous state and “produces (+)” the next one.

***

#### Register Expansion Trade-off

In Cairo M, registers are grouped together as a **main register stack**. When increasing the number of registers, a trade-off arises:

<table><thead><tr><th width="218.48345947265625">Option</th><th width="230.08074951171875">Advantage</th><th>Disadvantage</th></tr></thead><tbody><tr><td><strong>Expand the main register stack</strong></td><td>Adds one column per component → faster access</td><td>Each component must always use the column → wasted space</td></tr><tr><td><strong>Introduce secondary register stacks</strong></td><td>Access only when needed → saves columns</td><td><strong>Requires two lookup terms (≈ 4 base columns)</strong> per access</td></tr></tbody></table>

Because the values in the secondary stack do not exist directly in the main trace, every access must perform two lookups — one to consume (−) the previous state and another to produce (+) the new state. Thus, Cairo M must balance **space cost (columns)** and **computation cost (lookups)**.

In the current design, only $$\mathsf{(pc, fp, clock)}$$ reside in the **main stack**, while any additional registers will be introduced later via **secondary stacks** if needed.

***

### Opcodes

Cairo M’s instruction set originates from the original Cairo VM, but it has been redesigned to fit the new proving model based on **LogUp** and **component-based AIR**.

The goal of Cairo M’s opcode design is to strike a balance among **performance, simplicity, and parallelism**.

***

#### AIR Basics

AIR (Algebraic Intermediate Representation) expresses computation as a set of **polynomial constraints**.

An AIR can be viewed as a dataframe composed of **columns** (variables) and **rows** (state transition steps). Each operation is expressed as a constraint that must evaluate to zero for every row.

For example, the equation $$c = a + b$$ can be represented in AIR form as:

$$
\mathsf{df}[c] - \mathsf{df}[a] - \mathsf{df}[b] = 0
$$

This constraint must hold for every row, and each column is interpreted as a polynomial. During STARK proving, the polynomial commitments are stored via a Merkle tree.

Thus:

* **More columns** → larger proof size, more verification cost
* **Higher constraint degree** → larger evaluation domain and higher proving cost

Cairo M’s design goal is to **minimize both the number of columns and the degree of constraints for reduced proving and verification costs.**

***

#### Design Principles

Cairo M’s ISA design follows the classical **RISC (Reduced Instruction Set)** vs **CISC (Complex Instruction Set)** trade-off logic:

| Type               | Characteristic         | Trade-off                     |
| ------------------ | ---------------------- | ----------------------------- |
| **RISC-style ISA** | Fewer, simpler opcodes | More cycles but fewer columns |
| **CISC-style ISA** | Many, complex opcodes  | Fewer cycles but more columns |

For long traces, Cairo M can leverage **continuation** to split a large trace into smaller proving segments, and then recursively aggregate the resulting proofs.

If the execution trace is represented as a dataframe with shape $$(n, m)$$ ($$n$$ rows, $$m$$ columns), it can be reshaped into $$(n / k, m \cdot k)$$, but not the other way around — this highlights that increasing column count inherently increases the proof size and verifier complexity, since each additional column introduces another commitment.

{% hint style="info" %}
The paper doesn't explain how reshape is possible and why it's impossible to change the shape from  $$(n/k, m \cdot k)$$ to $$(n, m)$$.
{% endhint %}

Therefore, Cairo M chooses the **RISC approach (a long & thin AIR)**, which keeps columns few and constraints simple.

***

#### Minimal Instruction Set

Cairo M’s **minimal instruction set** is defined as the smallest possible configuration that maintains RAM consistency while still supporting all general-purpose computations.

| Category         | Instruction                                                | Description                         |
| ---------------- | ---------------------------------------------------------- | ----------------------------------- |
| **Control Flow** | `CallRel`, `Ret`                                           | Function calls and returns          |
| **Branching**    | `JmpRel`, `JnzRel`                                         | Conditional and unconditional jumps |
| **Arithmetic**   | `StoreAdd`, `StoreSub`, `StoreMul`, `StoreDiv`             | Field arithmetic and result storage |
| **Memory Move**  | `MoveString`, `MoveStringIndirect`, `MoveStringIndirectTo` | Memory copy and indirect moves      |

This instruction set alone supports field arithmetic, control flow, function calls, and memory manipulation.

The overall AIR footprint for this minimal set is approximately **52T + 39L columns**.

***

#### Extensions

The base instruction set alone is not always efficient in every context. Therefore, Cairo M introduces the concept of **extension opcodes**, which allow certain operations to be handled **natively at the prover level**.

***

**Uint Types**

While the STARK prover’s native type is a **field element**, real-world software typically relies on integer types such as `u8`, `u16`, `u32`, and `u64`. To bridge this gap, Cairo M supports these integer types **directly at the AIR level**, using **range-checked memory segments**.

* Division is defined as **Euclidean division**
* Each limb is range-checked to guarantee that its value lies within $$[0, 2^k)$$
* The largest simple native uint type is **u20** (as determined by the RangeCheck20 component)

However, since `u20` is not software-friendly, the practical implementation uses **u8** or **u16** as the base limb size.

| Type  | RangeCheck Component | Characteristic                                            |
| ----- | -------------------- | --------------------------------------------------------- |
| `u8`  | $$[0, 2^8)$$         | Byte-addressable, compatible with WASM/RISC-V             |
| `u16` | $$[0, 2^{16})$$      | Reduced arithmetic cost, efficient 16-bit limb operations |

This makes Cairo M’s memory layout compatible with byte-level zkVM architectures such as **RISC-V** and **WASM-based** systems.

***

**Built-in Functions (Precompiles)**

Cairo M also supports the concept of **built-in functions (precompiles)**, which define complex operations as dedicated AIR components. Operations such as **Poseidon hashing** or **modular exponentiation** are inefficient to implement as normal opcodes.

By defining them as precompiles, Cairo M gains several advantages:

1. **Fewer lookup arguments** → reduced column count
2. **Support for non-deterministic computations** → the prover can provide intermediate values as hints during witness generation

Precompiles can be integrated in two ways:

| Method                           | Description                                                                     | Trade-off                                  |
| -------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------ |
| **(1) Opcode-level**             | Add a unique opcode ID and include it in the main AIR                           | Increases column count                     |
| **(2) Recursive pre-processing** | Prove the operation in a separate AIR and recursively merge with the main proof | Improves parallelism and memory efficiency |

This design aligns with modern zk compiler trends, where the compiler can automatically generate precompile circuits and handle recursive composition transparently.

***

## Conclusion

The goal of Cairo M is simple —

**“To build the fastest, simplest, and most general-purpose zkVM.”**

Cairo M is not merely an “updated version” of the original Cairo VM; it represents a new design standard for modern zkVM architectures.

| Design Aspect             | Cairo VM                     | Cairo M                              |
| ------------------------- | ---------------------------- | ------------------------------------ |
| **Memory model**          | Relocation-based, read-only  | Merkle committed, read/write capable |
| **Field**                 | Large (2²⁵¹ + 17⋅2¹⁹² + 1)   | Small (M31)                          |
| **Proof style**           | Sequential, monolithic       | Continuation + recursion             |
| **Execution scalability** | Limited (\~10⁵ steps)        | Scalable to 10⁸+ steps               |
| **Implementation focus**  | StarkNet transaction proving | General-purpose computation          |

In other words, Cairo M can be seen as a **zk-friendly CPU architecture** redesigned into a **scalable proving environment**.

Cairo M began as a **minimal felt-based zkVM**, but ultimately reached the conclusion that a shift toward **byte-addressable memory** and a **byte-level ISA** is necessary. From this perspective, it acknowledges that reusing existing ISAs such as **RISC-V** or **WASM** in a ZK-friendly form is the most practical approach. Projects like **RISC Zero** are already leading the way in this direction.

***

The most important insight that emerged from Cairo M is this:

> **“The essence of a zkVM lies not in its instruction set, but in the communication and proof composition between components (continuation + recursion).”**

That is, the future of zkVMs will not be defined by _which instructions they support_, but by _how each AIR component exchanges proof data_ and _how multiple proofs are ultimately combined into a single unified proof_.

## References

* [https://github.com/kkrt-labs/cairo-m/blob/5a0a9609714f84352a649ee5b12b402f803e7802/docs/design.pdf](https://github.com/kkrt-labs/cairo-m/blob/5a0a9609714f84352a649ee5b12b402f803e7802/docs/design.pdf)

> Written by [Ryan Kim](https://app.gitbook.com/u/cPk8gft4tSd0Obi6ARBfoQ16SqG2 "mention") of Fractalyze

[^1]: Range?
