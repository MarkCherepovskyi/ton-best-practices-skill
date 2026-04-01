# ton-smart-contract-audit-context.md

Version: 1.0  
Last-updated: 2026-03-31  
Purpose: AI-agent-ready context for automated/assisted TON smart-contract audits.

## Executive summary
TON audits should be **phase-aware** (storage → credit → compute → action → bounce), **message-mode-aware** (flags alter rollback and failure behavior), and **cell/slice/builder-aware** (many failures are serialization/data-layout issues).

## Prioritized ingest sources
1. TON docs LLM index: https://docs.ton.org/llms.txt
2. TVM exit codes: https://docs.ton.org/tvm/exit-codes
3. Transaction phases: https://docs.ton.org/foundations/phases
4. Messages:
   - https://docs.ton.org/foundations/messages/overview
   - https://docs.ton.org/foundations/messages/internal
   - https://docs.ton.org/foundations/messages/modes
5. Debugging + retracer:
   - https://docs.ton.org/contract-dev/debug
   - https://retracer.ton.org/
6. Traces + APIs:
   - https://docs.ton.org/foundations/traces
   - https://docs.ton.org/ecosystem/api/toncenter/v3/actions-and-traces/get-actions
7. TVM registers (C4/C5): https://docs.ton.org/tvm/registers
8. Account lifecycle: https://docs.ton.org/foundations/status
9. Fees/gas:
   - https://docs.ton.org/foundations/fees
   - https://docs.ton.org/contract-dev/gas
10. Security best practices: https://docs.ton.org/contract-dev/security
11. Tooling:
   - https://docs.ton.org/contract-dev/blueprint/overview
   - https://github.com/ton-org/blueprint
   - https://github.com/ton-org/sandbox
   - https://github.com/nowarp/misti
   - https://github.com/espritoxyz/tsa

## Agent-first triage model
Normalize every failure as:
1) phase, 2) exit/result code, 3) bounce/mode context, 4) C5 action evidence.

### Practical exit-code subset
| Code | Phase | Meaning | Typical causes |
|---:|---|---|---|
| 4 | Compute | Integer overflow/div-by-zero | Unsafe arithmetic, sign mistakes |
| 5 | Compute | Integer out of range | Width mismatch in serialization |
| 8 | Compute | Cell overflow | >1023 bits or >4 refs in cell builders |
| 9 | Compute | Cell underflow | Reading more bits/refs than available |
| 10 | Compute | Dictionary error | Invalid dict ops/layout assumptions |
| 13/-14 | Compute | Out of gas | Unbounded loops/heavy ops |
| 33 | Action | Too many actions | >255 outbound actions |
| 37 | Action | Not enough Toncoin | Fee/balance underestimation |
| 39/40 | Action | Message packing/processing failure | Oversized/deep messages or low funds |
| 50 | Action | Account state too large | Unbounded storage growth |

## Severity-oriented audit checklist

### Critical / High
- Message mode misuse (`IGNORE_ERRORS`, destroy/account modes) causing state/value inconsistencies.
- Missing or unsafe bounce recovery for optimistic accounting updates.
- Access control flaws on value-moving or upgrade operations.
- Async race/ordering bugs across multi-message traces.

### Medium
- Numeric bounds and signed/unsigned issues.
- Slice/cell/dict parsing robustness (malformed input handling).
- Account lifecycle risks (nonexist/uninit/frozen/deleted transitions).

### Low / Informational
- Error-code consistency and observability.
- Documentation and invariant clarity for operators/integrators.

## Recommended workflow (CI-friendly)
1. Compile (BoC + ABI artifacts).
2. Run static analysis (Misti where applicable).
3. Run symbolic execution (TSA).
4. Run Blueprint/Sandbox unit + adversarial tests.
5. Run gas/coverage regressions.
6. For on-chain incidents: collect trace + retracer logs + C5 action evidence.

## Evidence bundle requirements
- Build artifacts (BoC/ABI/compiler versions)
- Reproducer tests and seeds
- Trace + tx receipt JSON
- Retracer step logs
- C5 actions dump
- Inbound/outbound message BoCs
- Patch + regression tests

## Prompt templates for agents

### Triage prompt
"Classify by phase and exit/result code first. Then map likely root cause, exploitability, fix, and minimal reproducer. If evidence is missing, list required artifacts."

### Structured finding format
- Title
- Severity/category
- Affected components
- Evidence
- Root cause
- Exploit/impact
- Reproduction
- Remediation
- Regression tests
- Confidence + assumptions

## Limitations
- Exit codes are failure symptoms, not full impact assessments.
- Many TON bugs are cross-contract and only visible in trace-level analysis.
- Tool coverage differs by language (e.g., Misti is Tact-centric; TSA is bytecode-centric).
