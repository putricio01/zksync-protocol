# Finding: Warm/Cold Storage Refund Is Unverifiable in the Proof System

**Severity: LOW / INFORMATIONAL**
**Category: Operator Trust Assumption — Gas Accounting**
**Affected Component: VM Circuit (main_vm/opcodes/log.rs)**

---

## Summary

The warm/cold storage access gas refund in the zkSync Era VM circuit is a prover-supplied witness value that is not verified by the storage sorter, storage applicator, or any downstream circuit. The `LogQuery` struct committed to the log queue contains no refund or warm/cold status field, making the refund invisible to all circuits outside the VM.

While this is a real verification gap, three structural properties of the current proof system limit its impact to **informational/low severity**.

---

## Technical Details

### How the refund works in the VM circuit

**File:** `crates/zkevm_circuits/src/main_vm/opcodes/log.rs`

1. **Witness allocation (lines 387-405):** The refund is allocated from a prover-supplied witness oracle:
   ```rust
   let cold_warm_access_ergs_refund = UInt32::allocate_from_closure_and_dependencies(
       cs, move |inputs| { guard.get_cold_warm_refund(...) }, &dependencies);
   ```

2. **Masking (line 408-409):** Applied only to state storage accesses:
   ```rust
   let cold_warm_access_ergs_refund =
       cold_warm_access_ergs_refund.mask(cs, is_state_storage_access);
   ```

3. **Bounding (lines 411-417):** Constrained to not exceed the opcode's base cost:
   ```rust
   let _ = max_refund.sub_no_overflow(cs, cold_warm_access_ergs_refund);
   ```
   This `sub_no_overflow` ensures `refund <= max(sload_cost, sstore_cost)`.

4. **Application (line 688):** Added to ergs_remaining AFTER the initial cost check:
   ```rust
   let ergs_remaining = ergs_remaining.add_no_overflow(cs, refund_value);
   ```

### What the storage sorter does NOT verify

**File:** `crates/zkevm_circuits/src/storage_validity_by_grand_product/mod.rs`

The storage sorter processes `LogQuery` entries that contain:
```rust
pub struct LogQuery<F: SmallField> {
    pub address: UInt160<F>,
    pub key: UInt256<F>,
    pub read_value: UInt256<F>,
    pub written_value: UInt256<F>,
    pub aux_byte: UInt8<F>,
    pub rw_flag: Boolean<F>,
    pub rollback: Boolean<F>,
    pub is_service: Boolean<F>,
    pub shard_id: UInt8<F>,
    pub tx_number_in_block: UInt32<F>,
    pub timestamp: UInt32<F>,
}
```
*(File: `crates/zkevm_circuits/src/base_structures/log_query/mod.rs`, lines 32-44)*

**No refund field. No is_warm field.** The storage sorter enforces sorting, deduplication, and value consistency — it has zero concept of warm/cold access patterns or gas refunds.

---

## Impact Assessment: Three Decider Questions

### 1. Is there a circuit-enforced per-block ergs budget?

**NO.**

- `MAX_TX_ERGS_LIMIT` (80M) in `zkevm_opcode_defs/src/system_params.rs` is software-only.
- `VmOutputData` contains only queue final states — no gas commitment:
  ```rust
  pub struct VmOutputData<F: SmallField> {
      pub log_queue_final_state: QueueState<F, QUEUE_STATE_WIDTH>,
      pub memory_queue_final_state: QueueState<F, FULL_SPONGE_QUEUE_STATE_WIDTH>,
      pub decommitment_queue_final_state: QueueState<F, FULL_SPONGE_QUEUE_STATE_WIDTH>,
  }
  ```
  *(File: `fsm_input_output/circuit_inputs/main_vm.rs`, lines 47-51)*
- The VM runs a fixed number of cycles (`main_vm/mod.rs:102`), not gas-bounded.
- Completion check only verifies callstack empty + bootloader exit (`main_vm/mod.rs:114-122`).
- No public input commits to total gas consumed.

**Implication:** The warm/cold refund gap cannot bypass a circuit-enforced gas budget because no such budget exists. Gas limits are purely sequencer software policy.

### 2. Does ergs_remaining gate anything beyond control flow?

**NO.**

- The charge-then-refund flow ensures the initial cost check (`overflowing_sub` at line 459) happens BEFORE the refund.
- `should_apply` (line 472) — the gate for all material effects — depends on `have_enough_ergs`, not on the refund amount.
- Queue lengths, log entries, and register updates are conditional on `should_apply` / `should_apply_io`.
- `ergs_remaining` is stored in `ExecutionContextRecord` and propagated to the next cycle, but only affects whether future operations succeed or fail.
- It does not influence: queue state, pubdata counters, or any public output.

**Implication:** An inflated refund gives the VM more gas for future operations, but those operations are still individually correct. No state corruption possible.

### 3. Is there an external consistency requirement between op count and charged gas?

**NO.**

- Public inputs contain only queue commitments, start/completion flags, and FSM state hashes.
- No gas-used counter appears in any public output or inter-circuit commitment.
- The VM runs exactly `limit` cycles regardless of meaningful operations.
- No receipt or gas total is verified against an external anchor.

**Implication:** There is nothing in the proof system that says "this block consumed X gas," so there's nothing for a gas inflation attack to violate.

---

## Comparison: Pubdata IS Constrained (Asymmetry)

Pubdata accounting has genuine circuit enforcement, confirming this is a deliberate asymmetry:

- `io_pubdata_cost` validated against `PubdataCostValidityTable` lookup (`log.rs:384`)
- Global `pubdata_revert_counter` accumulated in `cycle.rs:493-525`
- Non-negativity constraint enforced at `cycle.rs:528-533`:
  ```rust
  let le_bytes = new_state.pubdata_revert_counter.to_le_bytes(cs);
  let is_negative = test_if_bit_is_set(cs, &le_bytes[3], 7);
  Boolean::enforce_equal(cs, &is_negative, &boolean_false);
  ```

**Pubdata = circuit-constrained. Gas/ergs = software policy only.**

---

## Escalation Analysis

| Escalation Path | Result |
|-----------------|--------|
| Bypass initial cost check via refund? | **Not possible** — refund applied after cost check |
| Inflate ergs to overflow circuit capacity? | **Self-limiting** — fixed cycle count per circuit |
| Corrupt state via inflated ergs? | **Not possible** — ergs only gate control flow |
| Circumvent pubdata constraints? | **Not possible** — independent computation path |
| Interact with tx_number overflow? | **Not practical** — would need billions of txs per block |

No escalation path to state corruption or soundness violation was identified.

---

## Severity Justification

**LOW / INFORMATIONAL** because:

1. The operator is already a trusted centralized sequencer with full control over block contents
2. Gas is fundamentally not a circuit invariant — it's software policy
3. The refund cannot bypass the initial per-operation cost check
4. State transition soundness is maintained (storage values, pubdata, Merkle proofs all verified)
5. No escalation path to state corruption

**What the operator CAN do:** Claim warm refunds for cold accesses, inflating in-circuit ergs, allowing more operations per proof execution than gas accounting intends.

**What the operator CANNOT do:** Forge state transitions, avoid L1 pubdata costs, cause arithmetic unsoundness, or corrupt queue state.

---

## Recommendation

- **Current architecture:** Informational — falls within existing operator trust model.
- **Before decentralizing the sequencer:** Must be addressed. Options:
  1. Commit warm/cold status to the log queue and verify in the storage sorter
  2. Derive warm/cold inside the circuit via a per-tx access set accumulator
  3. Remove warm/cold refunds from the circuit entirely and handle at the software level
  4. Add a circuit-enforced block gas budget as a public input

---

## References

| File | Lines | What |
|------|-------|------|
| `main_vm/opcodes/log.rs` | 387-417 | Refund witness allocation and bounding |
| `main_vm/opcodes/log.rs` | 459-472 | Cost check (before refund) |
| `main_vm/opcodes/log.rs` | 668-688 | Refund application |
| `main_vm/cycle.rs` | 454-469 | ergs_remaining propagation |
| `main_vm/cycle.rs` | 493-533 | Pubdata counter constraints (for comparison) |
| `base_structures/log_query/mod.rs` | 32-44 | LogQuery struct (no refund field) |
| `main_vm/mod.rs` | 102-124 | Fixed cycle count, completion check |
| `fsm_input_output/circuit_inputs/main_vm.rs` | 47-51 | VmOutputData (no gas output) |
| `zkevm_opcode_defs/src/system_params.rs` | — | MAX_TX_ERGS_LIMIT (software-only) |
| `circuit_definitions/aux_definitions/witness_oracle.rs` | 204-236 | get_cold_warm_refund implementation |
