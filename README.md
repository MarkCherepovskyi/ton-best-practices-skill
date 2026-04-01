# ton-best-practices

A Claude Code skill for TON blockchain smart contract security auditing, development, and best practices.

**Language**: FunC / TVM
**Based on**: 233 vulnerabilities from 34 professional audits + TON/FunC documentation + real-world audited contract patterns

## Installation

```bash
npx skills add elsvv/ton-best-practices-skill
```

## What's inside

| File | Contents |
|------|----------|
| `SKILL.md` | Entry point — FunC security quick reference and threat model links |
| `func-complete-reference.md` | Complete FunC best-practices reference (vulnerabilities, patterns, checklist, tools) |
| `func-representation.md` | Practical FunC representation of secure contract flow |
| `vulnerabilities.md` | Full 233-vulnerability catalog with TON-specific examples |
| `tvm-async.md` | TVM internals, async model, BounceMode guide (Tolk 1.2 / TVM 12) |
| `ton-tvm-security-concepts.md` | TON/TVM threat-model primer: phases, fees, bounces, OOG, replay, limits |
| `ton-smart-contract-audit-context.md` | AI-agent-ready TON audit context (phases, exits, workflow, evidence artifacts) |
| `security-audit-checklist-abstract.md` | Contract-agnostic TON audit checklist (auth, parsing, gas, bounce, replay) |
| `audit-checklist.md` | 11-phase professional audit checklist (Phase 0 = Tolk config) |

## Key topics

- **Top 10 vulnerabilities**: auth checks, integer safety, async race/desync, message mode misuse, deserialization bugs, bounce recovery gaps, gas exhaustion
- **FunC-specific pitfalls**: missing `impure`, `~` vs `.`, boolean `-1`, load/store width mismatches
- **Async model**: carry-value pattern, bounce handlers, multi-message race conditions
- **Contract structures**: jetton master/wallet split, deterministic wallet derivation, transfer flow validation
- **Access control**: admin patterns, ownership transfer, workchain validation
- **Gas management**: fee estimation, reserve patterns, out-of-gas handling

## Trigger keywords

`FunC`, `TVM`, `TON contract`, `jetton`, `NFT TON`, `TON audit`, `bounce message`, `smart contract security`, `impure`, `recv_internal`

## Sources

- [PositiveSecurity/ton-audit-guide](https://github.com/PositiveSecurity/ton-audit-guide)
- [arXiv:2509.10823](https://arxiv.org/abs/2509.10823) — "From Paradigm Shift to Audit Rift" (233 vulns, 34 audits)
- [docs.ton.org Tolk docs](https://docs.ton.org/languages/tolk/overview) — full Tolk documentation
- [tolk-bench](https://github.com/ton-blockchain/ton) — 42 production Tolk contracts
- GitHub PRs [#1741](https://github.com/ton-blockchain/ton/pull/1741), [#1795](https://github.com/ton-blockchain/ton/pull/1795), [#1886](https://github.com/ton-blockchain/ton/pull/1886) — Tolk 1.0 / 1.1 / 1.2 changelogs
