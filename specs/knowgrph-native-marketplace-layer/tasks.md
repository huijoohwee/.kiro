---
title: "Knowgrph Clean-Room Native Marketplace Layer — Implementation Tasks"
doc_type: "Spec Tasks"
schema: "kiro-spec-tasks/v1"
version: "0.1.0"
date: "2026-08-22"
lang: "en-US"
frontmatter_contract: "required"
spec_type: "feature"
workflow_type: "requirements-first"
feature_name: "knowgrph-native-marketplace-layer"
owner: "Solo Founder / AI Orchestrator"
lane: "authoring"
local_rung: "spec-complete"
delivered_rung: "undocumented"
deploy_boundary: "closed"
implementation_language: "JavaScript (.mjs) for domain modules; TypeScript for typed contracts"
pbt_library: "fast-check (MIT) — already pinned in this repository; no second library introduced"
requirements_baseline: ".kiro/specs/knowgrph-native-marketplace-layer/requirements.md v0.1.0"
design_baseline: ".kiro/specs/knowgrph-native-marketplace-layer/design.md v0.1.0"
source_specification: "knowgrph/docs/documents/knowgrph-agentic-commerce-platform-prd-tad-adr.md v0.3.0"
governing_contracts:
  - "huijoohwee.github.io/guidelines/agentic-sdlc-guidelines.md"
  - "agentic-canvas-os/docs/START-WORKFLOW.md"
  - "agentic-canvas-os/docs/AGENTS.md"
  - "knowgrph/AGENTS.md"
---

# Implementation Plan

## Deploy Boundary Statement — Read Before Dispatching Any Task

Every task in this list executes entirely within the `authoring` lane, in `GitHub/knowgrph`, exercised through `npm run dev:apex` or `npm run dev`.

- **No task may mutate the Prod mirror** (`GitHub/huijoohwee/content/knowgrph`).
- **No task may mutate a Cloudflare route** (`airvio.co`, `airvio.co/knowgrph`).
- **No task may apply the D1 migration remotely.** Local application only. Remote application is an irreversible operator-gated operation with its own Deploy Boundary row *(Req 9.3)*.
- **No task may issue a real outward payout movement.** Task 9 stubs the rail port for exactly this reason *(Req 9.4)*.
- Every task's capability class is one of `read`, `local write`, or `local execute`. **Zero tasks carry `environment mutate`, `irreversible`, or `boundary-crossing`.** A task needing a wider class returns `blocked` with the requested operation and target boundary recorded, and does not widen its own grant *(Req 9.2, 9.6)*.
- All four Deploy Boundary Register rows added in source specification v0.3.0 read `closed` at the start and at the end of this increment *(Req 9.5)*.

## Conventions

**Task marking.** Sub-tasks postfixed with `*` are **not required for a working slice** — property tests, scans, integration checks, browser assertions, process assertions. Sub-tasks without `*` are required. Top-level tasks are never postfixed.

**Bounds.** Per the governing Per-Task Budgets rule, every task states four bounds plus a circuit-breaker on one `_Bounds:_` line: token budget · iteration cap · wall-clock cap · peak working-context cap · breaker. The default breaker is Requirement 8's derivation of the guideline: two consecutive iterations with no change in the named check's recorded result → stop retrying, transition the task to `failed`, record the last observed result and the terminal reason. A bound is never raised to rescue a failing task; the task is re-decomposed instead.

**Evaluator independence.** The named check is the Evaluator for every task in this list. No task's implementer marks its own task `verified`. `verified` is set only on a recorded check result, and it is the only success state.

**Return obligations.** Every task surfaces: the named check exactly as invocable, its recorded result (exit code plus counts or test summary), the enumerated artifacts changed including incidental changes, consumption against all four bounds, and any constraint violation observed — including ones the task itself caused.

## Dependency Graph

```
1 (clean-room scan)  ──►  2 (contracts + keys)  ──►  3 (vendor lifecycle)
                                              ├──►  4 (minor-unit allocation)
                                              └──►  11 (session log extension)

3 ──┐
4 ──┴──►  5 (commission rule schema + evaluator)  ──►  6 (vendor registry)  ──►  7 (split projector)

7 ──►  8 (payout state)  ──►  9 (payout rail port + coordinator)

6 ──►  10 (vendor settlement canvas)

7, 9, 10, 11 ──►  12 (migration)  ──►  13 (wiring + barrel)  ──►  14 (gate integration)  ──►  15 (doc rung update)
```

Acyclic. Task 1 precedes everything by Requirement 7.9. Task 3 and Task 4 are the only pair that may run in one wave — they write disjoint files and share no artifact *(no concurrent-write conflict)*. Every other pair either shares a file or has a real dependency edge, so no other wave contains two tasks.

---

- [ ] 1. Land the clean-room dependency scan before any component
  - Author `tests/scans/no-foreign-commerce-dependency.test.mjs` in the same shape as the existing `tests/scans/no-schema-retention.test.mjs` — a test asserting the absence of a pattern in source.
  - Fail on any `@medusajs/` or `@mercurjs/` specifier in any manifest in the repository, including every workspace manifest and the lockfile.
  - Fail on any import of those namespaces anywhere in source.
  - Fail on a dependency named `json-rules-engine` or `xstate` in any manifest.
  - Fail on any hosted Mercur or Medusa hostname appearing in source or configuration.
  - Assert the scan itself fails correctly by exercising it against a synthetic fixture containing a forbidden specifier, so the check is proven able to fail rather than merely observed passing.
  - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.7, 7.8, 7.9_
  - _Named check:_ `node --test tests/scans/no-foreign-commerce-dependency.test.mjs`
  - _Capability:_ local write, local execute
  - _Write scope:_ `tests/scans/no-foreign-commerce-dependency.test.mjs`
  - _Bounds:_ 40k tokens · 3 iterations · 20 min · 30% context · default breaker
  - _Why first:_ ADR-4's boundary is a stated intention until a check can fail on it. Building components first would mean claiming an enforced clean-room boundary on the evidence of nobody having violated it yet.

- [ ] 2. Extend the typed contracts and scope keys
  - [ ] 2.1 Add branded primitives `VendorId`, `CommissionRuleId`, `CommissionRuleRevisionId`, `SplitId`, `PayoutId` to `src/registry/typed-contracts.ts`, alongside the existing brands rather than in a new module.
    - Add `VendorLifecycleState`, `PayoutState`, `CommissionRuleKind`, `VendorSplitRow`, `PayoutRecord`, `CommissionRule`, and the closed rejection-reason unions for vendor, commission, split, and payout outcomes.
    - Keep the module type declarations only, with no runtime code, matching its current form.
    - _Requirements: 10.4, 6.1_
  - [ ] 2.2 Add `vendorKey`, `commissionRuleKey`, `vendorSplitKey`, `payoutKey`, and `vendorSettlementCanvasOperatorKey` to `src/registry/scope-keys.mjs`.
    - Follow the module's existing shape: `assertNonEmptyString` for internal invariants, `{ ok, value }` results for caller-supplied input.
    - Reuse the existing `OPERATOR_SCOPE` guard for the canvas key; do not introduce a second scope check.
    - _Requirements: 10.5, 5.3_
  - [ ] 2.3* Extend `tests/unit/scope-keys.test.mjs` with the five new key builders, including the operator-scope refusal path.
    - _Requirements: 5.3, 10.5_
  - _Named check:_ `npx tsc --noEmit -p tsconfig.json && node --test tests/unit/scope-keys.test.mjs`
  - _Capability:_ local write, local execute
  - _Write scope:_ `src/registry/typed-contracts.ts`, `src/registry/scope-keys.mjs`, `tests/unit/scope-keys.test.mjs`
  - _Bounds:_ 45k tokens · 3 iterations · 25 min · 30% context · default breaker

- [ ] 3. Build the vendor lifecycle transition table
  - [ ] 3.1 Author `src/marketplace/vendor-lifecycle-state.mjs` with the frozen transition table exactly as the design specifies, including `suspended → approved` rather than `suspended → active`, and no transition back into `pending_review`.
    - Export one pure decision function returning the next state or `{ ok:false, reason, currentState, requestedTransition }`.
    - Freeze the exported table. No storage access, no clock access.
    - Record a one-line lineage comment naming the pattern as independently derived, per the clean-room protocol — no link to, and no quotation of, any external source.
    - _Requirements: 1.4, 1.5, 7.5, 7.6, 7.7, 10.8_
  - [ ] 3.2* Author `tests/unit/vendor-lifecycle-state.test.mjs` covering each legal transition and each state's rejection set.
    - _Requirements: 1.4, 1.5_
  - [ ] 3.3* Author `tests/props/cp-21-vendor-lifecycle-totality.test.mjs` — error-condition class, shrinking enabled, minimum iteration count set. Assert that every transition absent from the frozen table is rejected and leaves state unchanged, generated exhaustively over `states × transitions`.
    - _Requirements: 1.5, 8.3, 8.8_
  - _Named check:_ `node --test tests/unit/vendor-lifecycle-state.test.mjs tests/props/cp-21-vendor-lifecycle-totality.test.mjs`
  - _Capability:_ local write, local execute
  - _Write scope:_ `src/marketplace/vendor-lifecycle-state.mjs`, `tests/unit/vendor-lifecycle-state.test.mjs`, `tests/props/cp-21-vendor-lifecycle-totality.test.mjs`
  - _Bounds:_ 45k tokens · 3 iterations · 25 min · 30% context · default breaker
  - _Why here:_ zero dependencies, exhaustively testable, and the payout precondition everything downstream reads. The source addendum orders it first for the same reason.

- [ ] 4. Build deterministic minor-unit allocation
  - [ ] 4.1 Author `src/commission/minor-unit-allocation.mjs` implementing largest-remainder allocation exactly as the design defines: floor every proportional share, distribute remaining units one each to the largest fractional remainders, break ties by ascending identifier.
    - Compare remainders as exact integer cross-products. **No floating-point operation anywhere in this module.**
    - Guarantee `sum(shares) === totalMinor` by construction rather than by post-hoc correction.
    - Reject a non-integer, unsafe-integer, zero, or negative total with a typed reason rather than coercing.
    - _Requirements: 2.7, 2.8, 10.2, 10.3_
  - [ ] 4.2* Author `tests/unit/minor-unit-allocation.test.mjs` covering exact division, single-unit remainder, all-remainder-tied, and rejection paths.
    - _Requirements: 2.7, 2.8_
  - [ ] 4.3* Author `tests/props/cp-20-allocation-order-invariance.test.mjs` — metamorphic class. Assert that permuting the input weight order does not change the allocated integers.
    - _Requirements: 2.7, 8.3, 8.7_
  - [ ] 4.4* Author `tests/props/cp-17-integer-only-amounts.test.mjs` — invariant class. Assert no allocated share is non-integer or outside safe-integer range, over generated totals and weight vectors.
    - _Requirements: 2.8, 8.3, 8.5_
  - _Named check:_ `node --test tests/unit/minor-unit-allocation.test.mjs tests/props/cp-20-allocation-order-invariance.test.mjs tests/props/cp-17-integer-only-amounts.test.mjs`
  - _Capability:_ local write, local execute
  - _Write scope:_ `src/commission/minor-unit-allocation.mjs`, `tests/unit/minor-unit-allocation.test.mjs`, `tests/props/cp-20-allocation-order-invariance.test.mjs`, `tests/props/cp-17-integer-only-amounts.test.mjs`
  - _Bounds:_ 50k tokens · 3 iterations · 30 min · 35% context · default breaker
  - _Note:_ this task is the mechanism behind Requirement 2.3's zero Residual. If allocation is correct, conservation cannot fail; if it is wrong, every split in the system is wrong. It is small and it is the highest-leverage code in this increment.

- [ ] 5. Build the commission rule schema and evaluator
  - [ ] 5.1 Author `src/commission/commission-rule-schema.mjs` defining the flat and tiered rule shapes, with rates in basis points constrained to `[0, 10000]`.
    - Validate tiered rules for ascending order, inclusive `upToMinor` boundaries, no overlap, and no gap, so two adjacent tiers can never both match one gross.
    - Return violation lists in the repository's existing violation-collecting idiom; never throw for expected input.
    - _Requirements: 3.6, 3.7, 3.8, 10.3_
  - [ ] 5.2 Author `src/commission/commission-evaluator.mjs` as a pure function over `{ grossMinor, rule, currency }`.
    - `commission = floor(gross × bps / 10000)` using integer arithmetic only; basis points keep the rate itself an integer.
    - Define `net = gross − commission`, so `gross = commission + net` holds by construction.
    - Return a typed rejection for an unresolvable, malformed, or out-of-range rule. **Never substitute a zero commission or a default rate** — a silent zero is a revenue defect that looks like a successful settlement.
    - No storage access, no clock access, no network access. No rate or boundary constant in the module.
    - Record a one-line independently-derived lineage comment per the clean-room protocol.
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.6, 3.8, 3.9, 7.6, 7.7_
  - [ ] 5.3* Author `tests/unit/commission-evaluator.test.mjs` covering flat, each tier of a tiered rule, both tier boundaries, zero-rate, full-rate, and every rejection reason.
    - _Requirements: 3.1, 3.6, 3.7_
  - [ ] 5.4* Author `tests/props/cp-16-commission-decomposition.test.mjs` — invariant class. Assert `gross = commission + net`, `0 ≤ commission ≤ gross`, and `net ≥ 0` over generated gross amounts and valid rules.
    - _Requirements: 3.2, 3.3, 8.3, 8.5_
  - [ ] 5.5* Author `tests/props/cp-24-commission-rule-round-trip.test.mjs` — round-trip class. Assert that re-evaluating a stored rule revision against a stored gross reproduces the stored commission bit-for-bit.
    - _Requirements: 3.4, 3.5, 8.3_
  - _Named check:_ `node --test tests/unit/commission-evaluator.test.mjs tests/props/cp-16-commission-decomposition.test.mjs tests/props/cp-24-commission-rule-round-trip.test.mjs`
  - _Capability:_ local write, local execute
  - _Write scope:_ `src/commission/commission-rule-schema.mjs`, `src/commission/commission-evaluator.mjs`, `tests/unit/commission-evaluator.test.mjs`, `tests/props/cp-16-commission-decomposition.test.mjs`, `tests/props/cp-24-commission-rule-round-trip.test.mjs`
  - _Bounds:_ 60k tokens · 3 iterations · 35 min · 40% context · default breaker
  - _Note:_ this task must be `verified` before Task 7 is dispatched. Per ADR-6, a commission defect aborts a bundle commit, so an unproven evaluator is a settlement outage waiting to happen.

- [ ] 6. Build the vendor registry
  - [ ] 6.1 Author `src/marketplace/vendor-schema.mjs` with the required-field set, closed lifecycle and currency enums, and violation reason codes matching the repository's existing reason vocabulary.
    - _Requirements: 1.1, 1.2, 10.3_
  - [ ] 6.2 Author `src/marketplace/vendor-records.mjs` for row ↔ domain mapping and content hashing, following the existing `hashDefinition` approach over key-sorted JSON.
    - _Requirements: 6.1, 10.3_
  - [ ] 6.3 Author `src/marketplace/vendor-registry.mjs` exposing `register`, `transition`, and `dispatchVerdict`.
    - `register` writes one complete row or nothing, returning a typed violation list on rejection; it forces `lifecycle_state = 'pending_review'` and ignores any caller-supplied state; it rejects a candidate whose commission rule reference does not resolve.
    - `transition` delegates the decision entirely to the lifecycle module and writes only what that module returned; it requires an actor reference so an activation is attributable.
    - `dispatchVerdict` returns `allowed:true` only for exactly `active`, with a distinct reason for each other state so a blocked payout can say why.
    - Suspension must not delete or alter existing splits — the freeze is the absence of a dispatch permission, never a mutation.
    - Do not add a KYC, attestation, screening, or verification field, and do not use compliance vocabulary in any identifier or message.
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.6, 1.7, 1.9, 1.10, 9.7_
  - [ ] 6.4* Author `tests/unit/vendor-registry.test.mjs` covering registration accept and reject, forced initial state, unresolvable rule reference, each `dispatchVerdict` state and reason, and the suspension freeze leaving splits untouched.
    - _Requirements: 1.1, 1.2, 1.3, 1.6, 1.7, 1.10_
  - _Named check:_ `node --test tests/unit/vendor-registry.test.mjs`
  - _Capability:_ local write, local execute
  - _Write scope:_ `src/marketplace/vendor-schema.mjs`, `src/marketplace/vendor-records.mjs`, `src/marketplace/vendor-registry.mjs`, `tests/unit/vendor-registry.test.mjs`
  - _Bounds:_ 60k tokens · 3 iterations · 35 min · 40% context · default breaker

- [ ] 7. Build the vendor ledger split projector
  - [ ] 7.1 Author `src/ledger/vendor-split-records.mjs` for the split row shape, covered-leg encoding as a canonical ascending array, and deterministic field ordering.
    - _Requirements: 2.6, 6.1_
  - [ ] 7.2 Author `src/ledger/vendor-split-projector.mjs` as a pure function of its inputs, taking `vendorLookup` and `evaluate` by injection rather than by import so its invariants are property-testable without a database.
    - Sequence: group legs by vendor → resolve each vendor → allocate gross per group → evaluate commission per group → assemble rows → assert every invariant → return or abort.
    - Assert and abort — never repair — on each of: gross sum equals settled total with zero Residual; every bundle leg in exactly one split; exactly one row per `(bundle, vendor)`; every amount a safe integer; one identical currency across all splits; every vendor resolves; `gross = commission + net` per row.
    - Order vendor groups by ascending vendor identifier and covered-leg identifiers in ascending order, so identical inputs give byte-identical output.
    - _Requirements: 2.2, 2.3, 2.4, 2.5, 2.6, 2.8, 2.10, 2.11, 3.2_
  - [ ] 7.3 Wire the projector into the bundle-commit transaction so the complete Split_Set is written in the same committed state as the bundle, and an invariant violation aborts the enclosing commit.
    - Change the call site only. Do not change the Bundle Graph Store's interface, schema, or contract.
    - Append one `split-committed` event per bundle, carrying bundle identity and split count — one per bundle, not per split, because the atomic unit is the Split_Set.
    - _Requirements: 2.1, 2.2, 2.9_
  - [ ] 7.4* Author `tests/unit/vendor-split-projector.test.mjs` covering single-vendor, multi-vendor, multi-leg-per-vendor, unresolvable vendor, mixed currency, and each abort path.
    - _Requirements: 2.5, 2.10, 2.11_
  - [ ] 7.5* Author `tests/props/cp-14-split-conservation.test.mjs` — invariant class. Assert gross sum equals settled total with Residual exactly zero, over generated leg breakdowns and vendor groupings.
    - _Requirements: 2.3, 8.3, 8.5_
  - [ ] 7.6* Author `tests/props/cp-15-leg-partition.test.mjs` — invariant class. Assert every leg appears in exactly one split, none duplicated and none dropped.
    - _Requirements: 2.4, 8.3, 8.5_
  - [ ] 7.7* Author `tests/props/cp-18-split-reprojection-idempotence.test.mjs` — idempotence class. Assert re-projecting the same committed bundle yields a byte-identical Split_Set including covered-leg ordering.
    - _Requirements: 2.6, 8.3, 8.6_
  - _Named check:_ `node --test tests/unit/vendor-split-projector.test.mjs tests/props/cp-14-split-conservation.test.mjs tests/props/cp-15-leg-partition.test.mjs tests/props/cp-18-split-reprojection-idempotence.test.mjs`
  - _Capability:_ local write, local execute
  - _Write scope:_ `src/ledger/vendor-split-records.mjs`, `src/ledger/vendor-split-projector.mjs`, the bundle-commit call site, and the four test files named above
  - _Bounds:_ 80k tokens · 4 iterations · 50 min · 50% context · default breaker
  - _Note:_ 7.3 is the only sub-task in this increment that touches a reused component. It changes a call site, not an interface. If it cannot be done without altering the Bundle Graph Store's contract, return `blocked` — that would be a scope change, not an implementation detail.

- [ ] 8. Build the payout state table
  - [ ] 8.1 Author `src/payout/payout-state.mjs` with the frozen payout transition table exactly as the design specifies.
    - `settled` and `failed` are terminal — no transition leaves either. `blocked` is deliberately non-terminal, because a blocked payout waits on a precondition while a failed one waits on an operator.
    - Export terminal-state predicates rather than requiring callers to compare against literals.
    - _Requirements: 4.10, 10.8_
  - [ ] 8.2* Author `tests/unit/payout-state.test.mjs` covering each legal transition, each terminal state's rejection of every further transition, and the `blocked → dispatched` recovery path.
    - _Requirements: 4.10_
  - _Named check:_ `node --test tests/unit/payout-state.test.mjs`
  - _Capability:_ local write, local execute
  - _Write scope:_ `src/payout/payout-state.mjs`, `tests/unit/payout-state.test.mjs`
  - _Bounds:_ 35k tokens · 3 iterations · 20 min · 25% context · default breaker

- [ ] 9. Build the payout rail port and dispatch coordinator
  - [ ] 9.1 Author `src/payout/payout-rail-port.mjs` as the single outward-call seam, backed by a service binding to the existing in-repo net-settlement route.
    - Perform no arithmetic; the amount arrives already computed and validated.
    - Introduce no external payout provider and add no Queues binding.
    - Ship the stub implementation used by every task in this increment, so no task issues a real movement.
    - _Requirements: 4.12, 9.4, 9.8_
  - [ ] 9.2 Author `src/payout/payout-dispatch-coordinator.mjs` with the gate order the design specifies, injecting `clock` so retry and breaker behaviour is testable without waiting.
    - Read the settlement-verified event for the split's bundle; absent → `blocked` with a recorded reason.
    - Read the vendor dispatch verdict; not `active` → `blocked` with a recorded reason.
    - Already `settled` → return the prior recorded result with no outward call.
    - Derive the idempotency key deterministically from `split_id` and reuse that exact key on every retry.
    - Import retry bounds from the existing pending-queue module rather than re-declaring them.
    - Two consecutive attempts with an unchanged recorded result → stop, terminal `failed`, record last observed result and terminal reason, no further automatic attempts.
    - Never default, infer, or assume either precondition.
    - _Requirements: 4.1, 4.2, 4.3, 4.5, 4.6, 4.7, 4.8, 4.9, 4.10, 4.11, 6.3_
  - [ ] 9.3 Wire the alarm trigger so dispatch runs outside the bundle-commit transaction, following the repository's existing alarm pattern.
    - _Requirements: 4.12, 4.13_
  - [ ] 9.4* Author `tests/unit/payout-dispatch-coordinator.test.mjs` covering each blocked reason, the already-settled short circuit, idempotency-key stability across retries, bound exhaustion, and circuit-breaker trip.
    - _Requirements: 4.1, 4.2, 4.3, 4.5, 4.8, 4.9_
  - [ ] 9.5* Author `tests/props/cp-19-payout-dispatch-idempotence.test.mjs` — idempotence class. Assert that repeated dispatch for one `split_id` yields at most one settled movement and the prior result thereafter.
    - _Requirements: 4.5, 4.6, 8.3, 8.6_
  - [ ] 9.6* Author `tests/props/cp-23-payout-ordering.test.mjs` — invariant class. Assert no dispatch attempt precedes the settlement-verified event and no dispatch reaches a non-`active` vendor, over generated event interleavings and vendor states.
    - _Requirements: 1.8, 4.1, 4.2, 4.4, 8.3_
  - _Named check:_ `node --test tests/unit/payout-dispatch-coordinator.test.mjs tests/props/cp-19-payout-dispatch-idempotence.test.mjs tests/props/cp-23-payout-ordering.test.mjs`
  - _Capability:_ local write, local execute
  - _Write scope:_ `src/payout/payout-rail-port.mjs`, `src/payout/payout-dispatch-coordinator.mjs`, the alarm wiring site, and the three test files named above
  - _Bounds:_ 75k tokens · 4 iterations · 45 min · 45% context · default breaker
  - _Note:_ last of the logic tasks by design. This is the only component that can move real money, and it is built against the other components once they are `verified`, exactly as the source addendum's build sequence orders it.

- [ ] 10. Build the vendor settlement canvas
  - [ ] 10.1 Author `src/marketplace/vendor-settlement-canvas.mjs` with project, render, and merge functions directly parallel to the existing registry canvas.
    - Refuse any scope other than `Operator_Scope`, through the existing key guard rather than a new check.
    - Merge deterministically, last-write-wins on content hash with a total ordering, so the result is order-independent and idempotent.
    - Render every vendor's identifier, lifecycle state, referenced commission rule revision, and outstanding payout position.
    - Use the shared Key-Type-Value row contract and semantic HTML elements; do not introduce a private table or list layout.
    - Keep every row readable at mobile width without horizontal scrolling.
    - Reuse the existing offline pending queue; do not add a second queue.
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7_
  - [ ] 10.2* Author `tests/unit/vendor-settlement-canvas.test.mjs` covering projection-to-record parity, the non-operator scope refusal, and render field completeness.
    - _Requirements: 5.1, 5.2, 5.3_
  - [ ] 10.3* Author `tests/props/cp-22-settlement-canvas-confluence.test.mjs` — confluence class. Assert merge converges identically for any interleaving of the same updates.
    - _Requirements: 5.4, 8.3, 8.9_
  - _Named check:_ `node --test tests/unit/vendor-settlement-canvas.test.mjs tests/props/cp-22-settlement-canvas-confluence.test.mjs`
  - _Capability:_ local write, local execute
  - _Write scope:_ `src/marketplace/vendor-settlement-canvas.mjs`, `tests/unit/vendor-settlement-canvas.test.mjs`, `tests/props/cp-22-settlement-canvas-confluence.test.mjs`
  - _Bounds:_ 55k tokens · 3 iterations · 30 min · 35% context · default breaker

- [ ] 11. Extend the session log vocabulary
  - [ ] 11.1 Add `vendor-activated`, `split-committed`, `payout-dispatched`, `payout-settled`, and `payout-failed` to the existing closed event set in `src/registry/session-log.mjs`.
    - Extend the identifier-required rule: the three payout events require a non-empty vendor identifier, mirroring the existing agent-identifier rule.
    - Do not create a second log store, and do not change the existing sequence-assignment behaviour.
    - _Requirements: 4.11, 6.4, 6.5_
  - [ ] 11.2 Add `payoutOrderingVerdict(entries, splitId)` alongside the existing `paymentOrderingVerdict`, returning `{ settlementVerifiedBeforeFirstDispatch, atMostOneSettledPayout, dispatchAllowed }`.
    - _Requirements: 4.4, 4.6, 6.4_
  - [ ] 11.3* Extend `tests/unit/session-log.test.mjs` with the five new event types, the vendor-identifier rejection path, and the ordering verdict across permuted append orders.
    - _Requirements: 4.4, 6.4, 6.5_
  - _Named check:_ `node --test tests/unit/session-log.test.mjs`
  - _Capability:_ local write, local execute
  - _Write scope:_ `src/registry/session-log.mjs`, `tests/unit/session-log.test.mjs`
  - _Bounds:_ 40k tokens · 3 iterations · 25 min · 30% context · default breaker

- [ ] 12. Author the D1 migration and apply it locally only
  - [ ] 12.1 Author `cloudflare/d1/migrations/0016_native_marketplace_settlement.sql` with the four tables and five indexes from the design.
    - Follow the established conventions: snake_case identifiers, TEXT primary keys, ISO TEXT timestamps, inline CHECK enums, explicit composite UNIQUE, prefixed index names.
    - Keep `gross_amount_minor > 0` and `net_payout_amount_minor >= 0` as storage-layer backstops; the evaluator remains the enforcement point.
    - Comment the split projection table as explicitly non-authoritative — the Durable Object row committed with the bundle is the authority.
    - Store rule bodies as canonical key-sorted JSON. No expression language, no dynamic evaluation path anywhere.
    - _Requirements: 10.6, 2.1, 3.3, 6.1_
  - [ ] 12.2 Apply the migration **locally only**.
    - _Requirements: 9.3_
  - [ ] 12.3* Assert the local schema matches the migration, and assert no task in this increment references a remote migration command.
    - _Requirements: 9.3_
  - _Named check:_ `npm run storage:d1:migrate:local`
  - _Capability:_ local write, local execute
  - _Write scope:_ `cloudflare/d1/migrations/0016_native_marketplace_settlement.sql`
  - _Bounds:_ 45k tokens · 3 iterations · 25 min · 30% context · default breaker
  - _Gate:_ remote application is **out of scope for this task and every other task in this list**. It is an irreversible operation requiring an explicit per-occurrence operator decision, and it has its own Deploy Boundary row. A forward migration is not rolled back by a local revert.

- [ ] 13. Wire the composition root and the public barrel
  - [ ] 13.1 Extend the existing agentic-commerce runtime composition to assemble the vendor registry, commission evaluator, split projector, payout coordinator, and settlement canvas, following the existing composition-root shape.
    - _Requirements: 10.9_
  - [ ] 13.2 Author `src/travel-commerce/marketplace.mjs` as a bounded re-export barrel, matching the existing barrel pattern, so tests and workers consume a stable surface.
    - _Requirements: 10.1_
  - [ ] 13.3* Author `tests/integration/marketplace-wiring.test.mjs` exercising the full path: register a vendor → activate it → project a two-vendor split → verify settlement → dispatch payouts → read the operator canvas.
    - Assert the whole chain is reconstructible from stored rows alone, with zero fields requiring recomputation from live external state.
    - _Requirements: 6.1, 6.2, 6.3_
  - [ ] 13.4* Extend `tests/process/deploy-boundary.test.mjs` to assert that no module in this increment references a Prod mirror path or Cloudflare route, that the payout rail port is the only outward-call site, and that all four new Deploy Boundary rows read `closed`.
    - _Requirements: 9.1, 9.2, 9.5_
  - _Named check:_ `node --test tests/integration/marketplace-wiring.test.mjs tests/process/deploy-boundary.test.mjs`
  - _Capability:_ local write, local execute
  - _Write scope:_ the composition root, `src/travel-commerce/marketplace.mjs`, `tests/integration/marketplace-wiring.test.mjs`, `tests/process/deploy-boundary.test.mjs`
  - _Bounds:_ 65k tokens · 4 iterations · 40 min · 45% context · default breaker

- [ ] 14. Integrate the focused sub-gate
  - [ ] 14.1 Add a `check:marketplace-settlement` script running the eight unit files, the clean-room scan, `cp-14` through `cp-24`, and the integration test, as the design lists them.
    - _Requirements: 8.12_
  - [ ] 14.2 Append `&& npm run check:marketplace-settlement` as the final clause of `check:agentic-commerce-platform`. Do not stand up a parallel check pipeline.
    - _Requirements: 8.12_
  - [ ] 14.3 Run the repository hygiene gate for every changed file and resolve findings at the owning source rather than by adding a mask, alias, or remap.
    - _Requirements: 10.1, 10.9, 10.10_
  - [ ] 14.4* Run the full aggregate commerce gate and record its result, not only the new sub-gate.
    - _Requirements: 8.11_
  - _Named check:_ `npm run check:marketplace-settlement && npm run hygiene:check && npm run check:agentic-commerce-platform`
  - _Capability:_ local write, local execute
  - _Write scope:_ `package.json` scripts only
  - _Bounds:_ 50k tokens · 3 iterations · 40 min · 35% context · default breaker

- [ ] 15. Re-derive rungs from emitted evidence
  - [ ] 15.1 Update each of the six new components' `local_rung` in the source specification's Component Inventory from `spec-complete` to the rung its emitted Evidence References actually support — no rung authored by hand, and no rung claimed beyond the checks that ran.
    - _Requirements: 8.13_
  - [ ] 15.2 Record each satisfied VCC's Evidence Reference with its named check, recorded result, and surface (`authoring`).
    - _Requirements: 8.13_
  - [ ] 15.3 Confirm and record that all four new Deploy Boundary Register rows still read `closed`, and that `delivered_rung` remains `undocumented`.
    - _Requirements: 9.5_
  - [ ] 15.4 Update the source specification's clean-room conformance paragraph from "stated intention" to "enforced", **only if** Task 1's scan is present and passing. If it is not, leave the paragraph unchanged.
    - _Requirements: 7.8, 7.9_
  - _Named check:_ `npm run check:marketplace-settlement && npm run docs:qa`
  - _Capability:_ local write, local execute
  - _Write scope:_ `knowgrph/docs/documents/knowgrph-agentic-commerce-platform-prd-tad-adr.md`
  - _Bounds:_ 45k tokens · 3 iterations · 30 min · 35% context · default breaker

---

## Bridge Coverage

| Requirement | Covered by task |
|---|---|
| 1.1–1.3, 1.9, 1.10 | 6.1, 6.2, 6.3 |
| 1.4, 1.5 | 3.1, 3.3* |
| 1.6–1.8 | 6.3, 9.2, 9.6* |
| 2.1, 2.2, 2.9 | 7.2, 7.3 |
| 2.3, 2.4 | 7.2, 7.5*, 7.6* |
| 2.5, 2.6 | 7.1, 7.2, 7.7* |
| 2.7, 2.8 | 4.1, 4.3*, 4.4* |
| 2.10, 2.11 | 7.2, 7.4* |
| 3.1–3.4, 3.6–3.9 | 5.1, 5.2, 5.4* |
| 3.5 | 5.5* |
| 4.1–4.4 | 9.2, 9.6*, 11.2 |
| 4.5–4.7 | 9.2, 9.5* |
| 4.8–4.10 | 8.1, 9.2, 9.4* |
| 4.11 | 11.1 |
| 4.12, 4.13 | 9.1, 9.3 |
| 5.1–5.7 | 10.1, 10.2*, 10.3* |
| 6.1–6.3 | 2.1, 6.2, 7.1, 9.2, 13.3* |
| 6.4, 6.5 | 11.1, 11.2, 11.3* |
| 7.1–7.9 | 1, plus lineage comments in 3.1, 5.2 |
| 8.1–8.13 | named checks on every task; 14.1, 14.2, 15.2 |
| 9.1–9.8 | boundary statement above; 9.1, 12.2, 13.4* |
| 10.1–10.10 | 2.1, 2.2, 12.1, 13.2, 14.3 |

**Coverage ratio**: 10 requirements / 10 covered = **10/10**. Every task traces to at least one requirement; no task introduces behaviour absent from `requirements.md`.

## Wave Plan

| Wave | Tasks | Write disjointness |
|---|---|---|
| 1 | 1 | single task |
| 2 | 2 | single task |
| 3 | 3, 4 | disjoint: `src/marketplace/` vs `src/commission/`, no shared test file |
| 4 | 5 | shares `src/commission/` with Task 4 |
| 5 | 6 | shares `src/marketplace/` with Task 3 |
| 6 | 7 | depends on 5 and 6 |
| 7 | 8, 11 | disjoint: `src/payout/` vs `src/registry/session-log.mjs` |
| 8 | 9 | shares `src/payout/` with Task 8 |
| 9 | 10 | shares `src/marketplace/` with Task 6 |
| 10 | 12 | single task |
| 11 | 13 | single task |
| 12 | 14 | single task |
| 13 | 15 | single task |

No wave contains two tasks writing the same artifact. Wave 3 and Wave 7 are the only concurrent pairs, and both are file-disjoint by construction.

## Escalation Triggers

Return `blocked` rather than proceeding, in these cases specifically:

- **Task 7.3 cannot write splits in the bundle-commit transaction without changing the Bundle Graph Store's interface.** That is a scope change against the source specification's "reused, call site extended only" claim, and it belongs back in the authoring loop.
- **A commission rule shape is needed that the flat/tiered schema cannot express.** ADR-5's stated revisit condition. Do not grow the hand-rolled evaluator unbounded; a hand-rolled evaluator that grows without limit is worse than the library it replaced.
- **Payout dispatch needs at-least-once delivery guarantees an alarm loop cannot provide.** ADR-6's stated revisit condition for Queues. Do not add a Queues binding inside a task.
- **Any of the four Open Questions carried in `requirements.md` must be answered to complete a task.** Each requires a recorded operator decision. An absent decision is `blocked`, never an assumed default.
- **A real outward payout movement, or a remote D1 migration, appears necessary.** Both are irreversible and both require an explicit per-occurrence operator decision.
- **The same approach has failed twice.** Diagnose the root cause, state it, and switch approach. Escalate on the third distinct failure rather than continuing to vary details.
