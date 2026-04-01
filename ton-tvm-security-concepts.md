# TON/TVM Security Concepts

Essential TON-specific concepts for security reasoning. These differ significantly from EVM patterns and are a major source of TON-specific vulnerabilities.

---

## 1) Execution model (async by design)

- **One message per transaction**: each transaction processes exactly one inbound message.
- **No atomic cross-contract execution**: all interactions are asynchronous message passing.
- **Delivery is guaranteed, timing is not**: messages may arrive many blocks later, so invariants checked early can be invalid by the time downstream messages execute.

### Transaction phases
1. **Storage phase** — storage fees deducted from balance
2. **Credit phase** — inbound message value credited
3. **Compute phase** — TVM executes contract logic
4. **Action phase** — outbound actions/messages are formed
5. **Bounce phase** — bounce is generated (if bounceable + failure)

---

## 2) Fee model: compute + storage + forward

```text
Compute fee  = comp_fee_price × gas_consumed
Storage fee  = storage_fee_price × contract_size_bits × seconds_elapsed
Forward fee  = fwd_fee_price × message_size
```

Security rule of thumb before risky state transitions:

```text
msg_value >= expected_compute_fee + expected_forward_fee (+ safety margin)
```

Underfunded message flows can create logical inconsistencies when downstream sends fail.

---

## 3) Silent bounce-failure trap

Dangerous scenario:
1. Contract A sends bounceable message to contract B
2. B fails
3. TVM attempts to send bounce to A
4. B cannot afford bounce forwarding fee
5. Bounce is dropped, and A never receives recovery signal

Implication: optimistic state changes in A may stay inconsistent unless protected by explicit pending-state accounting and timeout/reconciliation logic.

**TVM 12 note:** richer bounce payload behavior can increase fee requirements versus older assumptions.

---

## 4) Out-of-gas cannot be caught

- `out_of_gas` aborts execution immediately.
- Do not rely on generic catch patterns for OOG recovery.
- Prefer pre-checks, conservative gas budgeting, and reserve patterns (`raw_reserve`) when maintaining post-failure liveness is critical.

---

## 5) Untrusted code + `COMMIT` risk

If untrusted code is ever executed (e.g., unsafe upgrade flow), that code may persist changes via `COMMIT` and then trigger failure conditions. Restrict code-upgrade paths to strict admin control and validate code provenance.

---

## 6) Integer model in FunC

- FunC arithmetic uses signed 257-bit integers.
- `load_uint(n)` loads an unsigned bitstring into a 257-bit int value.
- Mixing signed/unsigned store-load conventions for the same field can introduce corruption or invariant breaks.
- Enforce explicit range checks at arithmetic boundaries.

---

## 7) Cell/storage limits matter for security

```text
Per cell: up to 1023 data bits + 4 refs
Per account state: up to 65536 unique cells
```

Large dictionaries and unbounded growth can trigger action/storage failures. Bound map sizes and design pruning/compaction paths.

---

## 8) `set_code` timing

`set_code(new_code)` applies after current transaction completion. Statements after `set_code` still execute under old code. Keep upgrade handlers minimal and deterministic.

---

## 9) Exit-code hygiene

- Avoid TVM-success codes (`0`, `1`) for thrown application errors.
- Keep a documented project-specific error range.
- Preserve stable, meaningful codes for monitoring and incident response.

---

## 10) Async race conditions

Checks at the start of a message cascade are not enough. Validate critical invariants at the point of final execution (receiver side), and use carry-value + pending-state patterns where needed.

---

## 11) Replay surface for external messages

External messages have no implicit nonce protection. If `recv_external` is used, include at least:
- signature verification,
- seqno/nonce tracking,
- expiry (`valid_until`) validation.

Without these, signed messages can be replayed.
