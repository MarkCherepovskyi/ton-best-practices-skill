# TON Smart Contract Best Practices (FunC) — Complete Reference

This file is the FunC-oriented context base for this skill. It intentionally complements, not replaces, the deeper catalogs in `vulnerabilities.md`, `audit-checklist.md`, and `tvm-async.md`.

## 1) Overview and TON vs EVM
- TON is asynchronous and message-driven; cross-contract execution is never atomic.
- Use carry-value flows and bounce-aware accounting.
- Replay protection is manual for external messages.

## 2) Top Critical Vulnerabilities (FunC-centric)
1. Missing auth checks on state mutation.
2. Integer range violations caught late during serialization.
3. Async race/desync across multi-message flows.
4. Missing `impure` on side-effect checks.
5. `~` vs `.` mutation confusion.
6. Unsafe send mode combinations.
7. Missing `end_parse()`.
8. Missing bounce recovery logic.
9. Bounce payload truncation assumptions.
10. Underestimated gas / reserve requirements.

## 3) TVM Internals You Must Model
- Compute/action/bounce separation.
- c4/c5 commit behavior.
- Cell and action-list limits.

## 4) Transaction Phases
`Storage -> Credit -> Compute -> Action -> Bounce`

Security rule: validate and fail in compute phase whenever possible.

## 5) FunC Language Pitfalls
- Every side-effect function should be `impure`.
- Prefer `equal_slices` for address equality.
- Use `~load_*` / `~udict_*` when you intend mutation.
- Treat booleans as non-zero checks (`true` is `-1`).

## 6) Message Handling Patterns
- Check bounced flag early.
- Parse op/query_id consistently.
- Reject unknown opcodes loudly.
- Keep handlers idempotent where possible.

## 7) Bounce Handling
- Handle only expected bounced ops.
- Restore optimistic accounting on bounce.
- Include enough value for destination to afford returning bounce.

## 8) Gas and Reserve Safety
- Validate fee sufficiency before mutation.
- Reserve minimum storage balance before aggressive sends.
- Recompute gas constants when logic changes.

## 9) Arithmetic and Serialization Safety
- Enforce explicit bounds for stored widths.
- Keep `load_*` and `store_*` widths aligned.
- Use `muldiv` for precision-sensitive math.

## 10) Upgrade Safety
- Upgrade paths must be strictly admin-gated.
- Assume `set_code` applies next transaction.
- Keep data schema migration explicit and testable.

## 11) External Message Security
- Signature + seqno + expiration are mandatory.
- `accept_message()` only after validation.
- Consider immediate seqno commit before risky branches.

## 12) Audit Checklist (Abstract)
Use `security-audit-checklist-abstract.md` as baseline, then layer contract-specific checks from opcodes/roles/invariants.

## 13) Tools
- FunC compiler
- Misti static analyzer
- @ton/sandbox transaction/gas tests
- TSA symbolic execution

## 14) Sources
- PositiveSecurity TON audit guide
- TON docs (FunC + TVM)
- arXiv 2509.10823 (233 vulnerabilities)

## 15) Contract Structures and Patterns (from TON docs)

### Jetton structure (master + per-owner wallet)
- A Jetton has one **master/minter** contract.
- Every owner has a separate **jetton wallet** contract at a deterministic address derived from `(owner, master, wallet_code)`.
- Transfer flow pattern: `transfer` -> `internal_transfer` -> optional `transfer_notification` (when `forward_ton_amount > 0`).

### Recommended interface patterns
- Jetton wallet should expose `get_wallet_data()` and return:
  - balance,
  - owner address,
  - master address,
  - wallet code.
- Jetton master should expose `get_wallet_address(owner)` for deterministic derivation.

### Receiver-side validation pattern
When processing jetton deposits/notifications:
1. Verify sender wallet is expected for the trusted master and owner/deposit wallet.
2. Verify opcode and body format before accounting.
3. Keep an allowlist of trusted jetton masters.

### External-message wallet pattern
For contracts that accept external signed commands, the robust sequence is:
1. Parse and validate fields,
2. Verify signature,
3. Check seqno/nonce and expiration,
4. `accept_message()`,
5. Persist replay guard, then execute actions.

### Gas-excess pattern
For non-consumed value, use explicit excess-return messaging (commonly opcode `0xd53276db`) and document send modes.
