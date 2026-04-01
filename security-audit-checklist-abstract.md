# Security Audit Checklist (Abstract, TON/TVM)

This checklist is intentionally **contract-agnostic**. Use it as a baseline for any TON/FunC/Tolk audit, then layer project-specific checks on top.

---

## 1) Access Control

- [ ] Every privileged operation has an explicit authorization check.
- [ ] Role boundaries are clear (owner/admin/operator/emergency roles).
- [ ] No hidden privilege escalation paths across helper dispatchers.
- [ ] Permissionless handlers are read-only or tightly bounded in effect.
- [ ] Admin handover is two-step and claimable only by the designated next admin.

## 2) State & Invariant Integrity

- [ ] Critical invariants are checked at the point of final execution (not only at initiation).
- [ ] State transitions are ordered to remain safe under asynchronous execution.
- [ ] Pending-state structures are used where cross-message reconciliation is required.
- [ ] Counters/maps updated symmetrically on insert/remove/error paths.

## 3) Parsing & Serialization Safety

- [ ] Message parsing is strict and complete (`end_parse`/equivalent).
- [ ] Signed/unsigned serialization is consistent across load/store paths.
- [ ] Optional values are handled explicitly; no unsafe unwrap assumptions.
- [ ] Unknown opcodes/sub-ops fail loudly (no silent catch-all).

## 4) Gas, Fees, and Reserve Safety

- [ ] Fee sufficiency is validated before mutating critical state.
- [ ] Compute + forward + storage impact is included in gas budgeting.
- [ ] Reserve strategy (`raw_reserve` or equivalent) is intentional and tested.
- [ ] Underfunded downstream failures cannot silently corrupt accounting.

## 5) Bounce & Failure Recovery

- [ ] Bounce handlers cover all optimistic state transitions that need rollback.
- [ ] Bounce loss scenarios (insufficient funds to send bounce) are considered.
- [ ] Non-bounceable sends are used only when irrecoverability is acceptable.
- [ ] Action-phase failure semantics are understood and documented.

## 6) Integer & Arithmetic Safety

- [ ] Arithmetic uses overflow-safe patterns (or explicit range checks).
- [ ] Division order preserves precision where economically relevant.
- [ ] Underflow to negative values is prevented for unsigned semantic fields.
- [ ] Error handling is explicit when numeric bounds are exceeded.

## 7) Message Modes & Send Semantics

- [ ] Send modes/flags are documented per message type.
- [ ] `IGNORE_ERRORS` usage is intentional and economically acceptable.
- [ ] Destructive or full-balance modes are authorization-gated.
- [ ] Multi-send handlers are validated for fee sufficiency and ordering hazards.

## 8) Upgrade & Code-Change Safety

- [ ] Upgrade paths are tightly admin-gated and auditable.
- [ ] `set_code` timing assumptions are correct (effective next transaction).
- [ ] Data schema compatibility across upgrades is verified.
- [ ] Untrusted code execution is avoided; if unavoidable, blast radius is constrained.

## 9) External Interface & Replay Safety

- [ ] External message handlers implement signature + nonce/seqno + expiry checks.
- [ ] Replay protection exists for every signed external command.
- [ ] Workchain/address validation is consistently applied where required.
- [ ] Exit codes are reserved, stable, and do not collide with success/system codes.

---

## Suggested Usage

1. Start with this abstract checklist.
2. Derive contract-specific checks from opcodes, roles, and invariants.
3. Add targeted tests for all high-risk async and bounce paths.
