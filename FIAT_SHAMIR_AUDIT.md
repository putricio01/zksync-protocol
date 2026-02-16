# Fiat-Shamir Security Audit: zkSync Era Protocol Layer

**Date:** February 2026
**Scope:** Recursive verification circuits (leaf, node, recursion tip, scheduler, compression)

---

## 1. Summary

This document covers the Fiat-Shamir audit of the recursive verification layer in `zksync-protocol`. The actual transcript implementation lives in `zksync-crypto/crates/boojum`, but the recursive circuits in this repo instantiate and use those transcripts for in-circuit proof verification.

**Assessment: PASS — No Fiat-Shamir vulnerabilities detected in the recursive verification layer.**

---

## 2. Recursive Verification Pipeline

```
Base Layer Proof (boojum native)
  └─> Leaf Layer (crates/zkevm_circuits/src/recursion/leaf_layer/mod.rs)
       └─> Node Layer (crates/zkevm_circuits/src/recursion/node_layer/mod.rs)
            └─> Recursion Tip (crates/zkevm_circuits/src/recursion/recursion_tip/mod.rs)
                 └─> Scheduler (crates/zkevm_circuits/src/scheduler/mod.rs)
                      └─> Compression (circuit_definitions/src/circuit_definitions/aux_layer/)
                           └─> SNARK Wrapper (zksync-crypto snark-wrapper)
```

Each layer calls `verifier.verify::<H, TR, CTR, POW>()` which internally performs the Fiat-Shamir transcript operations using the circuit-compatible transcript (`CircuitTranscript`).

---

## 3. Key Security Properties Verified

### 3.1 Transcript Trait Consistency

The recursive verifier uses `RecursiveTranscript<F>` and `CircuitTranscript<F>` traits that mirror the native `Transcript<F>` trait:

| Native Trait Method | Circuit Trait Method | Purpose |
|---|---|---|
| `witness_field_elements(&[F])` | `witness_field_elements(cs, &[Variable])` | Absorb data |
| `witness_merkle_tree_cap(&[Cap])` | `witness_merkle_tree_cap(cs, &[Cap])` | Absorb commitment |
| `get_challenge() -> F` | `get_challenge(cs) -> Variable` | Squeeze challenge |

The circuit versions perform identical absorb/squeeze sequences but operate on circuit variables instead of field elements.

### 3.2 VK Commitment Binding

In recursive verification, each layer receives a verification key that must be committed to the transcript. The VK is committed as a hash of 4 Goldilocks field elements (`VK_COMMITMENT_LENGTH = 4`, defined in `crates/zkevm_circuits/src/recursion/mod.rs:9`).

The leaf layer verifies proofs against a fixed VK for each circuit type, and the VK commitment is part of the public output used by the node layer.

### 3.3 Cross-Layer Input/Output Binding

Each recursive circuit uses a finite state machine (FSM) input/output pattern:
- Inputs and outputs are committed via `commit_variable_length_encodable_item()` using the Poseidon2 sponge.
- The input/output commitment length is 4 field elements.
- These commitments are checked for consistency across layers.

This prevents cross-layer value manipulation because the outputs of one layer become the committed inputs of the next.

---

## 4. Audit Checklist for Recursive Layer

| # | Check | Status |
|---|-------|--------|
| 1 | Leaf layer calls `verifier.verify()` with correct transcript sequence | PASS |
| 2 | Node layer calls `verifier.verify()` with correct transcript sequence | PASS |
| 3 | Recursion tip calls `verifier.verify()` with correct transcript sequence | PASS |
| 4 | Scheduler calls `verifier.verify()` with correct transcript sequence | PASS |
| 5 | VK commitments are bound to circuit outputs | PASS |
| 6 | FSM input/output commitments use proper hashing | PASS |
| 7 | No hardcoded challenge values in recursive circuits | PASS |
| 8 | Circuit transcript matches native transcript operations | PASS |

---

## 5. Relevant File Paths

- `crates/zkevm_circuits/src/recursion/leaf_layer/mod.rs` — Leaf layer entry point
- `crates/zkevm_circuits/src/recursion/node_layer/mod.rs` — Node layer entry point
- `crates/zkevm_circuits/src/recursion/recursion_tip/mod.rs` — Recursion tip entry point
- `crates/zkevm_circuits/src/scheduler/mod.rs` — Scheduler circuit
- `crates/zkevm_circuits/src/recursion/mod.rs` — VK_COMMITMENT_LENGTH, NUM_BASE_LAYER_CIRCUITS
- `crates/zkevm_circuits/src/fsm_input_output/circuit_inputs.rs` — FSM I/O commitment
- `crates/circuit_definitions/src/circuit_definitions/recursion_layer/mod.rs` — Circuit type aliases
- `crates/circuit_definitions/src/lib.rs` — Proof config, security parameters

---

## 6. Cross-Reference

The detailed Fiat-Shamir audit of the underlying proof system (boojum, bellman, snark-wrapper) is in `zksync-crypto/FIAT_SHAMIR_AUDIT.md`.
