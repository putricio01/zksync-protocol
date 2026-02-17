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

## Deep-Dive: Can Refund Inflation Bypass Any Hard Per-Block Bound?

A targeted investigation was conducted to determine whether inflated warm/cold refunds
could bypass any hard or implicit per-block capacity limit in the proof system.

### Hard bounds identified

| Bound | Value | Location | Enforced how |
|-------|-------|----------|-------------|
| SCHEDULER_CAPACITY | 28,000 total circuit instances | `circuit_definitions/recursion_layer/mod.rs:38` | Scheduler loop count baked into proving key (`scheduler/mod.rs:1056`), with hard assertion all types must complete (`scheduler/mod.rs:1266`) |
| Per-VM-instance cycles | 5,390 (production) | `GeometryConfig.cycles_per_vm_snapshot` | Fixed loop in VM circuit |
| RECURSION_TIP_ARITY | 32 | `recursion/recursion_tip/input.rs:24` | Fixed-size array for per-type queues |
| NUM_BASE_LAYER_CIRCUITS | 20 types | `recursion/mod.rs:10` | Enum + fixed-size arrays throughout |

### How the scheduler creates a hard per-block bound

The scheduler circuit (`scheduler/mod.rs:1056-1266`) processes **one base-layer circuit instance
per iteration** for exactly `config.capacity = SCHEDULER_CAPACITY = 28,000` iterations. Every VM
snapshot, storage sorter instance, storage applicator instance, etc. consumes one slot. After the
loop, the circuit enforces `execution_flag == false` (line 1266) — meaning all 20 circuit types
must have completed. If a block requires > 28,000 total instances, the proof is unsatisfiable.

This value is chosen to fit the scheduler circuit within a 2^20 trace size (comment at lines 32-37).

### The gas-to-circuit coupling design

`ERGS_PER_CIRCUIT = 80,000` (`system_params.rs:12`) is the designed invariant: 80K ergs of
computation should fill approximately one circuit instance. The pricing generator
(`circuit_pricing_generator/main.rs`) derives all opcode costs from this:

- Cold SLOAD: 2,008 ergs = 4 (VM) + 1 (RAM) + 1 (demuxer) + 2 (sorter) + 2,000 (cold access)
- Warm SLOAD: 38 ergs = 4 (VM) + 1 (RAM) + 1 (demuxer) + 2 (sorter) + 30 (warm access)
- Refund delta: 1,970 ergs per inflated SLOAD

### Why inflation CANNOT bypass SCHEDULER_CAPACITY

The decisive factor is **VM_INITIAL_FRAME_ERGS = u32::MAX ≈ 4.3 billion ergs**
(`system_params.rs:9`, enforced at `loading.rs:45-46`).

For N unique cold SLOADs, the circuit instances required (using production geometry):

```
Instances(N) ≈ N/5390 [VM] + N/33 [storage_app] + N/44171 [sorter] + N/58125 [demuxer]
             ≈ N × 0.0305
```

**Gas-bounded maximum (no inflation):**
```
N_gas = u32::MAX / 2008 ≈ 2,139,000 SLOADs
Instances = 2,139,000 × 0.0305 ≈ 65,239  →  ALREADY EXCEEDS 28,000
```

**Circuit-bounded maximum:**
```
N_circuit = 28,000 / 0.0305 ≈ 918,000 SLOADs
```

Even **without** inflated refunds, the bootloader's 4.3B ergs can sustain ~2.14M SLOADs —
far more than the ~918K that circuit capacity allows. **The circuit capacity is already the
binding constraint.** Refund inflation cannot push past a bound that is already reached under
honest accounting.

**With inflation:** u32::MAX / 38 ≈ 113M SLOADs → still bounded by circuit capacity at ~918K.
The inflation is immaterial.

### On the pricing-model numbers (optimistic case for attacker)

The pricing generator uses larger capacities than actual geometry (e.g., `CYCLES_PER_STORAGE_APPLICATION = 118`
vs production `33`). Using pricing-model numbers:

- Without inflation: ~2.14M ops → ~18,524 instances (66% of 28,000)
- With inflation: ~3.23M ops → ~28,000 instances (100% of 28,000)

Even in this most-favorable framing, the operator needs inflation only to fill the last ~34% of
scheduler capacity — and can already fill 66% without it. Moreover, the operator can achieve the
same result by simply including more transactions with legitimate gas consumption, since they
control transaction inclusion.

### Recursion structure: no additional bound

- **Leaf layer**: Tree recursion per circuit type — unbounded instances per type.
- **Node layer**: Intermediate tree aggregation — unbounded.
- **Recursion tip**: `ARITY=32` bounds circuit **types** (20 ≤ 32), not instances per type.
- **Public inputs**: `keccak256(prev_hash || this_hash)` → 4 field elements (`scheduler/mod.rs:1507-1527`). No commitment to circuit counts.

### Verdict on escalation

**No present-day escalation.** The warm/cold refund inflation cannot bypass `SCHEDULER_CAPACITY`
because the bootloader's u32::MAX ergs already provides enough gas to exceed circuit capacity
without inflation. The circuit-instance bound — not the gas bound — is the operational limit
for an adversarial operator.

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
| `main_vm/opcodes/log.rs` | 779-819 | Queue push enables gated by `should_apply` |
| `main_vm/cycle.rs` | 454-469 | ergs_remaining propagation across cycles |
| `main_vm/cycle.rs` | 493-533 | Pubdata counter constraints (for comparison) |
| `main_vm/loading.rs` | 45-46 | VM_INITIAL_FRAME_ERGS = u32::MAX (circuit-enforced) |
| `main_vm/pre_state.rs` | 257-261 | ergs_remaining read at cycle start |
| `base_structures/log_query/mod.rs` | 32-44 | LogQuery struct (no refund field) |
| `main_vm/mod.rs` | 102-124 | Fixed cycle count, completion check |
| `fsm_input_output/circuit_inputs/main_vm.rs` | 47-51 | VmOutputData (no gas output) |
| `scheduler/mod.rs` | 1056-1263 | Scheduler loop — 28,000 iterations processing circuit instances |
| `scheduler/mod.rs` | 1266 | Hard assertion: all circuit types must complete |
| `circuit_definitions/recursion_layer/mod.rs` | 32-38 | SCHEDULER_CAPACITY = 28,000 |
| `recursion/recursion_tip/input.rs` | 24, 30-33 | RECURSION_TIP_ARITY = 32, fixed-size arrays |
| `zkevm_opcode_defs/src/system_params.rs` | 7, 9, 12 | MAX_TX_ERGS_LIMIT, VM_INITIAL_FRAME_ERGS, ERGS_PER_CIRCUIT |
| `zkevm_opcode_defs/src/circuit_pricing_generator/main.rs` | 9-33, 78-156 | Gas-to-circuit coupling design |
| `zkevm_opcode_defs/src/circuit_prices.rs` | 4-28 | Generated per-operation ergs costs |
| `zkevm_opcode_defs/src/definitions/log.rs` | 103-118 | StorageRead/Write ergs_price (cold costs) |
| `boojum/src/gadgets/queue/mod.rs` | 594-611 | QueueState includes length (UInt32) |
| `circuit_definitions/aux_definitions/witness_oracle.rs` | 204-236 | get_cold_warm_refund implementation |
