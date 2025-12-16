# Lean VM

### 1. System Objective

Lean Ethereum targets a transition from BLS to a post-quantum signature path using stateful hash-based signatures (XMSS), with aggregation handled by a hash-based SNARK. The dominant cost is proving many hash evaluations, and a minimal [cairo.md](../zkdsl/cairo.md "mention")-inspired zkVM expresses the control logic and hashing.

Two operations are supported:

* Aggregation of XMSS signatures
* Merging of aggregate signatures via recursive SNARK verification

### 2. Protocol

#### 2.1 Statement of Computation: $$\text{AggregateMerge}$$

All aggregation and merge behavior is unified into a single program, $$\text{AggregateMerge}$$, which is the only program whose execution must be proven.

**2.1.1 Inputs**

* Public inputs: `pub_keys`, `bitfield`, `msg`
* Private inputs: `s`, `sub_bitfields`, `aggregate_proofs`, `signatures`

**2.1.2 Semantics**

1.  Bitfield consistency:

    $$
    \texttt{bitfield} = \bigcup_i \texttt{sub\_bitfields}[i]
    $$
2.  For $$i = 0..s - 2$$: verify recursive proofs with inner public input

    $$
    (\texttt{pub_keys},\ \texttt{sub_bitfields}[i],\ \texttt{msg})
    $$
3.  For the last sub-bitfield: verify actual XMSS signatures for indices selected by

    $$
    \texttt{sub_bitfields}[s-1]
    $$

#### 2.2 VM Execution Model

**2.2.1 Field and Extension Field**

*   Base field: KoalaBear prime

    $$
    p = 2^{31} - 2^{24} + 1
    $$

    chosen for fewer Poseidon rounds and an efficient Poseidon2 S-box.
* Extension field dimension: 5 or 6 (for security/performance trade-offs).

**2.2.2 Memory and Registers**

<figure><img src="../.gitbook/assets/image (187).png" alt=""><figcaption></figcaption></figure>

*   Memory is read-only, word-aligned to base-field elements, size

    $$
    M = 2^m,\quad 16 \le m \le 29
    $$

    and the memory size is fixed at proof start.
*   The first

    $$
    M' = 2^{m'}
    $$

    memory cells contain public input written by the verifier.
* Registers: `pc`, `fp`; there is no `ap` register (no tracked allocation pointer in the VM state).

**2.2.3 ISA and Precompiles**

Core instructions:

* `add/mul`
* `deref`
* conditional `jump`

Precompiles:

1. `POSEIDON_16`: Poseidon2 permutation over 16 field elements
2. `POSEIDON_24`: Poseidon2 permutation over 24 field elements
3. `DOT_PRODUCT`: Computes a dot product between two slices of extension field elements.
4. `MULTILINEAR EVAL`: interpret a memory chunk as a multilinear polynomial over the base field and evaluate at an extension-field point; implemented via sparse equality constraints in WHIR, with no dedicated AIR table.

#### 2.3 Reduced Commitments

**2.3.1 Execution Table Commitments**

Per cycle, the execution table commits five base-field elements:

* $$\text{pc}, \text{fp}, \text{addr}_a, \text{addr}_b, \text{addr}_c$$

**2.3.2 Instruction Encoding and Bytecode Lookup**

Each instruction is described by 15 field elements:

*   operands:

    $$
    \mathrm{operand}_a,\ \mathrm{operand}_b,\ \mathrm{operand}_c
    $$
*   flags:

    $$
    \mathrm{flag}_a,\ \mathrm{flag}_b,\ \mathrm{flag}_c
    $$
*   opcode flags: 8 one-bit selectors

    $$
    \texttt{add},\ \texttt{mul},\ \texttt{deref},\ \texttt{jump},\
    \texttt{poseidon16},\ \texttt{poseidon24},\ \texttt{dot_product},\ \texttt{multilinear_eval}
    $$
*   auxiliary field:

    $$
    \mathrm{aux}
    $$

Instead of committing these 15 elements per cycle, the execution table commits only `pc`, and a [logup.md](../zk/lookup/logup.md "mention")-based indexed lookup proves that the correct instruction row in the bytecode table is used.

#### 2.4 Precompile Interaction Model: Two Buses

Precompiles live in dedicated tables and are connected to the execution table and memory via two logical “buses”.

**2.4.1 Bus 1: Execution → Precompile Tables**

For each precompile call, the tuple

$$
(i_{\mathrm{precompile}},\ \nu_a,\ \nu_b,\ \nu_c)
$$

appearing in the execution table must also appear in the corresponding precompile table.

A permutation/grand-product argument enforces this inclusion: the multiset of tuples requested in the execution table must match the multiset of tuples present in precompile tables.

**2.4.2 Bus 2: Precompile Tables → Memory**

Each precompile row accesses memory and enforces correctness of the corresponding computation (for example, Poseidon over a block, or a dot product over an extension-field vector).

This is done by:

* Using the addresses recorded in the precompile table
* Performing memory lookups
* Enforcing that the precompile outputs match the specified function of the fetched inputs

When memory accesses are aligned, power-of-two blocks (e.g., Poseidon16 with alignment 8), lookups can be optimized by folding memory regions to reduce lookup cost.

#### 2.5 Commitment and Opening Schedule (Two-Phase)

The commitment scheme operates in two phases:

1. **Phase 1**: Commit to:
   * Memory
   * Execution table
   * Precompile tables for `poseidon16`, `poseidon24`, and `dot_product`
2.  **Phase 2**: After sampling random challenges (Fiat–Shamir), commit to pushforwards for the two `logup*` lookups:

    $$
    \mathrm{memory}_{\mathrm{lookup}},\quad
    \mathrm{bytecode}_{\mathrm{lookup}}
    $$

Finally, all required values are opened at sampled points across these commitments, and all AIR, lookup, and bus constraints are verified at those points.

#### 2.6 Polynomial Packing

The system uses multilinear polynomials (primarily AIR columns and table columns). Instead of committing each polynomial separately, multiple polynomials $$P_i$$ are packed into a single polynomial $$P$$, reducing both:

* The number of commitments
* The number of openings

Evaluation claims on each $$P_i$$ at some point are reduced to evaluation claims on $$P$$ via:

* Chunking each $$P_i$$ into a small number of power-of-two blocks
* Treating evaluation of $$P_i$$ as a combination of “inner evaluations” within these blocks
* Encoding these inner evaluations as sparse equality constraints on the packed polynomial $$P$$

This avoids running a separate sumcheck over each $$P_i$$ and amortizes costs.

### 3. Implementation of Prover

See [prove\_execution in leanMultisig](https://github.com/leanEthereum/leanMultisig/blob/main/crates/lean_prover/src/prove_execution.rs#L20)

#### 3.1 Execution witness generation

The system begins by executing a program written in lean lang ISA, capturing the entire execution trace of the virtual machine. This includes the state of key registers (e.g., `pc`, `fp`) at each step, memory read operations, and invocations of any precompile operations. Memory is modeled as immutable input and logged during execution, forming the primary witness.

#### 3.2 AIR Table Construction

From the execution witness, the system constructs:

* **Execution Trace Table**: Tracks VM state evolution (registers, flags) across cycles.
* **Memory Table**: Records accessed memory cells and their contents.
* **Precompile Tables**: For Poseidon16, Poseidon24, records full round-by-round computation of trace of that hash and records for dot-product operations as well.

Each table is represented as a multilinear polynomial over a hypercube domain.

#### 3.3 Phase 1: WHIR Commitments

Using the WHIR commitment scheme, the prover commits to five polynomial tables:

1. Execution trace
2. Memory table
3. Poseidon16 table
4. Poseidon24 table
5. Dot-product table

These commitments are absorbed into a Fiat–Shamir transcript and fix the witness data.

#### 3.4 Phase 2: Logup\* Lookup Arguments

The system performs two indexed lookups:

* **Memory Lookup**:
  * Merges all (address, value) pairs from CPU and precompile access logs.
  * Verifies consistency against the committed memory table.
  * Uses random challenges to compress each pair to a field element (e.g., $$a + \gamma v$$).
  * Constructs pushforward polynomial $$I_*$$ that is essentially the interpolation of all those combined values from the _access logs._
* **Bytecode Lookup**:
  * Captures all `(pc, opcode)` pairs from the execution trace.
  * Validates against the committed bytecode ROM.
  * Applies similar Logup\* compression and commitment as with memory.

#### 3.6 WHIR polynomial IOP

With all commitments, the system proceeds to generate the final proof. In the concluding phase, it invokes the WHIR interactive proof protocol (a multilinear polynomial IOP) to produce the proof. Two batched `open` are done for memory and bytecode. At this point, lookup arguments are represented as evaluations by inner GKR of Logup\*.

### 4. Reference

* [Minimal VM by EF](https://github.com/leanEthereum/leanMultisig/blob/main/minimal_zkVM.pdf)
* [leanMultisig (implementation)](https://github.com/leanEthereum/leanMultisig/tree/main/crates)

> Written by [Soowon Jeong](https://app.gitbook.com/u/gkru4r7198XKD3pCvVYtqakZB3Z2 "mention") of Fractalyze
