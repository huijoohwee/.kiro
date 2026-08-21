---
title: "Knowgrph Agentic Travel Commerce Platform — Core Tasks"
doc_type: "Spec Task Shard"
schema: "kiro-spec-tasks/v1"
version: "1.4.0"
date: "2026-08-20"
lang: "en-US"
frontmatter_contract: "required"
spec_type: "feature"
workflow_type: "requirements-first"
feature_name: "knowgrph-agentic-travel-commerce-platform"
owner: "Solo Founder / AI Orchestrator"
lane: "authoring"
universal_scope: false
local_rung: "dev-proven"
delivered_rung: "undocumented"
deploy_boundary: "dev-only"
parent_tasks: ".kiro/specs/knowgrph-agentic-travel-commerce-platform/tasks.md v1.4.0"
responsibility: "Tasks 1–10: foundations, graph and ledger correctness, and the Cascade hot path"
---

# Core Tasks

## Tasks 1–10

- [x] 1. Foundation — property-test harness and closed typed contracts

  - [x] 1.1 Configure the fast-check property-test harness
    - Add `fast-check` (MIT) as a dev dependency at a pinned exact version; add zero runtime dependency
    - Use the shared evidence helper to keep shrinking enabled, derive a deterministic replay seed per named check, print that seed, and set `numRuns` explicitly at 200 or above
    - Identify each generated property by its CP number in the check name and emitted evidence record
    - Keep payment clients and discovery/issuance adapters mocked or deterministic so zero property test issues a real payment call
    - Exercise concurrency through the local Cloudflare Vitest Durable Object SQLite pool rather than a deployed runtime
    - _Requirements: 10.2, 13.7_
    - _Properties: harness for CP-1..CP-16_
    - _Check: `npm run travel-commerce:test`_
    - _Scope: `package.json`, `cloudflare/workers/knowgrph-travel-commerce/vitest.config.mts`, `cloudflare/workers/knowgrph-travel-commerce/test/evidence/_support.ts`, local service doubles and payment mocks_
    - _Capability: local write, local execute, environment mutate (one pinned dev dependency, declared in advance)_
    - _Bounds: 25k tokens · 3 iterations · 30 min · 35% context · breaker: default_

  - [x] 1.2 Author `bundle-types.ts` — type declarations only, zero logic
    - Declare branded identities: `BundleId`, `LegId`, `OfferId`, `PrincipalId`, `HoldId`, `CascadeId`, `EventId`, `SnapshotId`, `ModelId`, `MinorUnits`
    - Declare `LegRow` and `EdgeRow` as closed readonly interfaces with no index signature, so an excess field is a compile error rather than a lint warning
    - Declare `CascadeOutcome` as a discriminated union over `no-op | committed | rolled-back | rejected`, so "committed carrying a rollback reason" is unrepresentable
    - Declare `RejectReason` and `RollbackReason` as closed string unions covering every cause in the design's Error Handling table
    - Declare `HoldState`, `Hold`, and `ReserveResult` such that a terminal hold state carries no transition target and a rejected reserve carries no hold
    - Represent every monetary value as `MinorUnits` integer; declare zero `number` money field
    - Emit zero runtime value from this module
    - _Requirements: 1.7, 1.8, 2.3, 5.1, 7.3, 7.4, 7.5, 7.7_
    - _Properties: foundation for CP-3, CP-8, CP-13_
    - _Check: `npm run travel-commerce:typecheck`_
    - _Scope: `src/bundle/bundle-types.ts`_
    - _Capability: local write_
    - _Bounds: 30k tokens · 3 iterations · 35 min · 40% context · breaker: default_

  - [x]* 1.3 Write type-level assertions for the closed contract types
    - Assert an excess field on `LegRow` or `EdgeRow` fails type-checking (expect-error assertions)
    - Assert a `committed` `CascadeOutcome` cannot carry a `RollbackReason`, and a `rolled-back` outcome cannot carry `settlementCalls: 1`
    - Assert a float cannot be assigned to `MinorUnits`
    - Assert `BundleId` is not assignable to `LegId`
    - _Requirements: 2.3, 3.4_
    - _Properties: CP-3 (structural arm)_
    - _Check: `npm run check:atomic-commit`_
    - _Scope: `tests/unit/bundle-types.types.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 15k tokens · 2 iterations · 20 min · 30% context · breaker: default_

- [x] 2. Envelope Ledger — the primitive the L4 claim rests on (PRD Next Step 1)

  - [x] 2.1 Implement `envelope-ledger.ts` — one DO per principal, atomic check-and-reserve
    - Create the current `envelope` and position-aware `holds` SQLite tables, including non-negative money CHECKs, currency, reservation kind, ordinary operation/agent/verification identity, `bundle_id`, `leg_id`, target amount, prior hold, expiry, the `state` CHECK constraint, and active/Cascade/ordinary-operation indexes
    - Compute Available_Balance rather than storing it; expose zero stored-balance column
    - Implement `checkAndReserveCascade` as one indivisible replacement operation: compute Available_Balance including released prior commitments, compare, insert the Cascade's `reserved` holds, and return `ReserveResult`
    - Key exactly one Durable Object instance per `principal_id`; hold two principals in zero shared instance
    - Implement `releaseCascade(cascadeId)` returning the released count, for the rollback path
    - Expose `getAvailableBalance()` to server-side callers only; expose zero client-facing read scope
    - Issue zero D1 query from any method in this file
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 5.10, 8.2; ordinary-offer criteria 4.11–4.14 are owned by Task 2.7 and the additional record emitted by the same named check_
    - _Properties: CP-6, CP-7_
    - _Check: `npm run check:envelope-atomicity`_
    - _Scope: `src/ledger/envelope-ledger.ts`_
    - _Capability: local write_
    - _Bounds: 40k tokens · 4 iterations · 50 min · 45% context · breaker: default_

  - [x]* 2.2 Write property test for envelope non-overdraft under concurrency
    - **Property CP-6: No accepted offer exceeds Available_Balance at accept-time under arbitrary concurrent interleaving (Invariant)**
    - Generator: `arbConcurrentOffers` — 2..16 offers for one principal across interleaved arrival orders, including amounts that individually pass but jointly overspend
    - `numRuns: 600` — highest count in the set: the PRD states over-envelope commits as `0`, a hard invariant, not a target
    - Assert every accepted offer satisfies `amount ≤ available_at_check`, and no committed hold set sums above `total_budget`
    - Shrinking enabled — the useful report is the minimal interleaving that overspends
    - **Validates: Requirements 4.1, 4.3, 4.5, 4.6**
    - _Check: `npm run check:envelope-atomicity`_
    - _Scope: `tests/props/cp-06-envelope-non-overdraft.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 35k tokens · 4 iterations · 45 min · 50% context · breaker: default_

  - [x] 2.3 Implement `hold-lifecycle.ts` and ledger integration — legal transitions and conservation
    - Permit normal `reserved → committed` and `reserved → released`; after an attempted ambiguous effect permit `reserved → quarantined`, and permit `quarantined → committed|released` only through authenticated reconciliation; reject every other transition with `illegal-transition`
    - Treat a repeated transition to a hold's current terminal state as a no-op returning that state, mutating nothing
    - Assert `total_budget = available + Σreserved + Σquarantined + Σcommitted` after every transition, in the same operation, not in a background reconciler
    - Have the Envelope Ledger invalidate the Balance_Cache entry for the principal **before** returning a Hold mutation result
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.7, 5.8_
    - _Properties: CP-7, CP-8_
    - _Check: `npm run check:hold-lifecycle`_
    - _Scope: `src/ledger/hold-lifecycle.ts`_
    - _Capability: local write_
    - _Bounds: 30k tokens · 3 iterations · 35 min · 40% context · breaker: default_

  - [x]* 2.4 Write property test for envelope conservation
    - **Property CP-7: `total_budget = available + Σreserved + Σquarantined + Σcommitted` after every transition (Invariant)**
    - Generator: budgets and multiple generated reserved Holds; exercise their deterministic terminal transitions
    - `numRuns: 300`
    - Assert the identity holds after every single transition, not only at sequence end
    - **Validates: Requirements 5.4**
    - _Check: `npm run check:hold-lifecycle`_
    - _Scope: `tests/props/cp-07-envelope-conservation.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 25k tokens · 3 iterations · 30 min · 35% context · breaker: default_

  - [x]* 2.5 Assert hold transition idempotence in the combined lifecycle property
    - **Property CP-8: Repeated commit or release of the same hold is a no-op returning the current state (Idempotence)**
    - The combined CP-7/CP-8 property repeats generated `reserved → committed → committed` arms for 300 runs; a concrete ledger arm repeats `releaseCascade` and observes `idempotent`
    - Assert state after an identical terminal transition equals state after the first
    - **Validates: Requirements 5.2**
    - _Check: `npm run check:hold-lifecycle`_
    - _Scope: `tests/props/cp-08-hold-idempotence.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 20k tokens · 3 iterations · 25 min · 35% context · breaker: default_

  - [x]* 2.6 Measure and record check-and-reserve latency and release visibility
    - Measure `checkAndReserve` inside the Durable Object double and assert under 10 ms with zero D1 hop; record the measured value, not a pass/fail alone
    - Assert a released hold's amount reappears in the next computed Available_Balance with no staleness window beyond write-then-read consistency
    - Assert the Balance_Cache invalidation is ordered before the transition return
    - _Requirements: 4.7, 5.3, 5.7, 8.2_
    - _Check: `npm run check:envelope-atomicity`, `npm run check:hold-lifecycle`_
    - _Scope: `tests/integration/envelope-latency.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 25k tokens · 3 iterations · 30 min · 35% context · breaker: default_

  - [x]* 2.7 Integrate registered ordinary offers with the same principal ledger
    - Add `checkAndReserveOffer`, `commitOffer`, and `releaseOffer` as server-side Envelope Ledger RPCs; bind one stable ordinary reservation identity to principal, operation, registered agent, offer, amount, currency, and verification proof
    - Export `TravelAgencyGuardrailService` as a named same-lane Worker entrypoint, reuse the inherited Guardrail Gate signature unchanged, and bind Agent Registry/Router to that entrypoint in Dev, Staging, and Production
    - Preserve the inherited service-bound Reopt_Worker `routeIntent` request, but require Edge_Orchestrator to use the distinct guarded operation; assert no ordinary offer-return branch can bypass successful atomic reservation
    - Exercise mixed concurrent ordinary and Cascade reservations against one principal, expiry, exact idempotent replay while reserved/committed, fail-closed replay after release, conflicting replay, commit/release lifecycle, failed dependency, caller authorization, and Production rejection of non-verified prices
    - Enforce the request body ceiling while streaming and cancel on overflow before buffering the complete body
    - _Requirements: 4.8, 4.9, 4.10, 4.11, 4.12, 4.13, 4.14, 5.11, 12.1, 12.2_
    - _Check: `npm run check:envelope-atomicity` for the mixed ledger and authenticated public-ingress reservation records; `npm run travel-commerce:test -- --disableConsoleIntercept cloudflare/workers/knowgrph-travel-commerce/test/ordinary-offer-atomicity.test.ts`; `npm run mcp:worker:test` for lifecycle and same-lane topology regressions_
    - _Scope: `src/gate/travel-agency-guardrail-service.ts`, `src/gate/guardrail-envelope-adapter.ts`, `src/ledger/ordinary-offer-holds.ts`, `src/ledger/envelope-ledger{,-state,-alarms}.ts`, `cloudflare/workers/knowgrph-mcp/travel-commerce-router.mjs`, lane configs and focused tests_
    - _Capability: local write, local execute_
    - _Bounds: 45k tokens · 5 iterations · 60 min · 55% context · breaker: default_

- [x] 3. Checkpoint — foundation and envelope primitive
  - Ensure all tests pass; ask the user if questions arise
  - Confirm `npm run travel-commerce:typecheck` is clean, each authored file is ≤600 authored lines with one declared responsibility, CP-6/CP-7/CP-8 all report recorded seeds, and Task 2.7's ordinary-offer/MCP evidence is evaluated before this gate is treated as terminal
  - _Requirements: 4.6, 5.4_
  - _Check: `npm run travel-commerce:test && npm run check:envelope-atomicity && npm run check:hold-lifecycle`; `npm run travel-commerce:test -- --disableConsoleIntercept cloudflare/workers/knowgrph-travel-commerce/test/ordinary-offer-atomicity.test.ts`; `npm run mcp:worker:test`_
  - _Scope: none (read only)_
  - _Capability: read, local execute_
  - _Bounds: 10k tokens · 2 iterations · 15 min · 25% context · breaker: default_

- [x] 4. Bundle structure — flat adjacency tables and the declared scale boundary

  - [x] 4.1 Implement `bundle-graph-store.ts` structure half — schema, insertion, limits
    - Create the current `bundle_meta`, `legs`, `edges`, `topology`, `cascades`, `settlement_claims`, `reconciliation_decisions`, `session_log`, and `cost_log` schema through the split schema/storage helpers; keep rollback input in each Cascade's immutable `prior_legs_json` rather than claim a separate snapshots table
    - Implement `insertLeg` and `insertEdge`, each rejecting rather than truncating: over 20 legs, over 20 edges, cycle-introducing edge, cross-principal leg — each with its typed reason plus the observed count, mutating nothing on rejection
    - Expose `limits` as readable named constants `{ maxLegs: 20, maxEdges: 20 }` so a check asserts against the declaration rather than a value it hardcoded
    - Scope exactly one DO instance per `bundle_id`
    - Admit at most one non-terminal Cascade per bundle; return `bundle-busy` for a distinct Cascade and for every Leg/Edge insertion while it is active, and reject pre-committed Leg insertion with `committed-leg-insertion-unsupported`
    - Keep the evidence boundary explicit: `check:scale-boundary` emits Requirement 7 criteria, while `check:atomic-commit` emits the focused 2.10–2.12 structural-fence record
    - Introduce zero graph engine, graph query language, or graph-specific storage; issue zero D1 query
    - _Requirements: 2.10, 2.11, 2.12, 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 7.7, 7.8, 8.1_
    - _Properties: CP-13_
    - _Check: `npm run check:scale-boundary`; `npm run check:atomic-commit` for the emitted 2.10–2.12 structural-fence record_
    - _Scope: `src/bundle/bundle-graph-store.ts`, `cloudflare/workers/knowgrph-travel-commerce/test/core-runtime.test.ts`, `cloudflare/workers/knowgrph-travel-commerce/test/core-recovery-regressions.test.ts`_
    - _Capability: local write_
    - _Bounds: 40k tokens · 4 iterations · 50 min · 45% context · breaker: default_

  - [x] 4.2 Implement `topo-order.ts` — deterministic order on structure insertion
    - Compute and persist topological order on bundle initialization and structure insertion; recompute a full sort zero times per Mutation_Event
    - Use the stated deterministic ready-set tie-break: ascending `legId`
    - Reject a cycle-introducing insertion, returning the typed reason the store surfaces
    - _Requirements: 7.5, 7.9, 7.10_
    - _Properties: CP-15_
    - _Check: `npm run check:scale-boundary`_
    - _Scope: `src/bundle/topo-order.ts`_
    - _Capability: local write_
    - _Bounds: 30k tokens · 3 iterations · 40 min · 40% context · breaker: default_

  - [x]* 4.3 Write property test for scale-boundary and structural error conditions
    - **Property CP-13: Insertions past 20 legs or 20 edges, cycles, and cross-principal legs are rejected with the correct typed reason and mutate nothing (Error Condition)**
    - Generators: `arbOversizeBundle`, `arbBundleWithCycle`, plus a cross-principal arm
    - `numRuns: 400` — four distinct rejection causes, totality matters more than depth here
    - Assert each rejection carries the correct reason and the observed count, and that store state is byte-identical before and after every rejected insertion
    - **Validates: Requirements 7.3, 7.4, 7.5, 7.7**
    - _Check: `npm run check:scale-boundary`_
    - _Scope: `tests/props/cp-13-scale-boundary.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 30k tokens · 4 iterations · 40 min · 45% context · breaker: default_

  - [x]* 4.4 Write property test for topological order confluence
    - **Property CP-15: Incremental topological order converges to the full-recompute order for any insertion interleaving of the same edge set (Confluence)**
    - Generator: `arbEdgeInsertionOrders` — permutations of one edge set's insertion order
    - `numRuns: 300`. Shrinking enabled — the useful report is the minimal insertion order that diverges
    - Assert the incrementally maintained order equals the full-recompute order under the same tie-break, for every permutation
    - **Validates: Requirements 7.9, 7.10**
    - _Check: `npm run check:scale-boundary`_
    - _Scope: `tests/props/cp-15-topo-confluence.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 30k tokens · 4 iterations · 40 min · 45% context · breaker: default_

  - [x]* 4.5 Static scan — zero graph engine, declared limits readable
    - Scan the module graph reachable from `bundle-graph-store.ts` and assert zero graph database client, graph query language driver, or graph-native traversal library is reachable
    - Assert `limits` is exported as named constants and that no test hardcodes 20 independently of it
    - _Requirements: 7.1, 7.2, 7.8_
    - _Check: `npm run check:scale-boundary`_
    - _Scope: `tests/scans/no-graph-engine.test.ts`_
    - _Capability: read, local execute_
    - _Bounds: 15k tokens · 2 iterations · 20 min · 30% context · breaker: default_

- [x] 5. Affected-set walk — downstream-only precision

  - [x] 5.1 Implement `affectedSet` and `isPresent` in `bundle-graph-store.ts`
    - Implement queue-based BFS over outgoing edges from the changed leg with a visited set; visit each leg at most once; terminate when no unvisited outgoing edge remains
    - Exclude the changed leg from the returned set; return traversal order, not an unordered set
    - Return `no-op` for a changed leg with zero outgoing edges; reject an absent leg with `unknown-leg`; reject a reachable cycle with `cyclic-dependency`
    - Implement `isPresent` as a separate authorization read; do not fuse it into `affectedSet`
    - Build the in-memory adjacency list once per DO wake and hold it for the instance's active lifetime; cache edge structure only, never leg membership, offer identity, or committed amount
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.7, 1.8, 8.5_
    - _Properties: CP-1, CP-2_
    - _Check: `npm run check:affected-set`_
    - _Scope: `src/bundle/bundle-graph-store.ts`_
    - _Capability: local write_
    - _Bounds: 35k tokens · 4 iterations · 45 min · 45% context · breaker: default_

  - [x]* 5.2 Write property test for affected-set precision
    - **Property CP-1: Affected_Set equals exactly the downstream-reachable set, excluding the changed leg (Invariant)**
    - Generator: `arbBundle` — acyclic single-principal bundles of 1..20 legs with arbitrary edge density
    - `numRuns: 300`
    - Assert set equality against an independently computed reachability closure, so the test does not re-implement the BFS under test; assert no unaffected sibling appears and no downstream leg is missing
    - **Validates: Requirements 1.1, 1.2, 1.3**
    - _Check: `npm run check:affected-set`_
    - _Scope: `tests/props/cp-01-affected-set-precision.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 30k tokens · 4 iterations · 40 min · 45% context · breaker: default_

  - [x]* 5.3 Write property test for walk termination and single visit
    - **Property CP-2: BFS visits each leg at most once and terminates on every acyclic graph within the scale boundary (Invariant)**
    - Generator: `arbBundle`, including diamond and wide fan-out shapes where a naive walk revisits
    - `numRuns: 200`
    - Assert visit count per leg ≤ 1 and that the walk terminates for every generated case
    - **Validates: Requirements 1.4**
    - _Check: `npm run check:affected-set`_
    - _Scope: `tests/props/cp-02-walk-termination.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 20k tokens · 3 iterations · 25 min · 35% context · breaker: default_

  - [x]* 5.4 Measure walk latency and assert per-Cascade logging
    - Measure the walk for bundles of ≤8 legs inside the DO double and assert under 50 ms; record the measured value
    - Assert exactly one Session_Log entry per Cascade including no-op and rejected ones, carrying `bundle_id`, changed leg, affected identifiers in traversal order, and outcome
    - _Requirements: 1.6, 1.9_
    - _Check: `npm run check:affected-set`_
    - _Scope: `tests/integration/affected-set-latency.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 25k tokens · 3 iterations · 30 min · 35% context · breaker: default_

- [x] 6. Checkpoint — graph structure and walk
  - Ensure all tests pass; ask the user if questions arise
  - Confirm CP-1, CP-2, CP-13, CP-15 report recorded seeds and that `check:scale-boundary` and `check:affected-set` both exit zero
  - _Requirements: 1.2, 7.10_
  - _Check: `npm run travel-commerce:test && npm run check:affected-set && npm run check:scale-boundary`_
  - _Scope: none (read only)_
  - _Capability: read, local execute_
  - _Bounds: 10k tokens · 2 iterations · 15 min · 25% context · breaker: default_

- [x] 7. Atomic commit and rollback

  - [x] 7.1 Implement durable prepare, atomic affected-set finalize, and rollback in `bundle-graph-store.ts`
    - Capture the exact prior Leg set in the durable Cascade record before preparation; keep it as the rollback target
    - Prepare the complete quote set without changing the visible graph projection, then finalize every affected Leg in one storage transaction after settlement disposition permits it; expose zero partially committed state
    - Implement `rollbackCascade` such that captured affected Legs are restored while the active-Cascade structural fence proves Edge rows and unaffected Legs could not change, making the visible Leg/Edge snapshot byte-identical to the preceding committed state
    - Reject a commit whose leg set is not exactly the affected set for that Cascade
    - Commit only through this transactional path; expose zero direct row-write operation outside it
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.9, 2.10, 2.11_
    - _Properties: CP-3, CP-4, CP-10_
    - _Check: `npm run check:atomic-commit` for 2.1–2.4/2.9 and the emitted 2.10–2.12 structural-fence record_
    - _Scope: `src/bundle/bundle-graph-store.ts`, `cloudflare/workers/knowgrph-travel-commerce/test/core-runtime.test.ts`_
    - _Capability: local write_
    - _Bounds: 40k tokens · 4 iterations · 50 min · 45% context · breaker: default_

  - [x]* 7.2 Write property test for commit atomicity
    - **Property CP-3: No observable state has some affected legs on new offers and others on stale offers (Invariant)**
    - Generators: `arbBundle` × `arbRequoteResults` — per-leg pass / reject / missing / malformed / timeout arms
    - `numRuns: 500` — the PRD states partial-commit incidents as `0`, a hard invariant
    - Assert, at every observation point including mid-transaction, that the affected set is wholly new or wholly stale
    - Shrinking enabled — the useful report is the minimal result vector that produces a mixed state
    - **Validates: Requirements 2.1, 2.2, 2.3**
    - _Check: `npm run check:atomic-commit`_
    - _Scope: `tests/props/cp-03-commit-atomicity.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 35k tokens · 4 iterations · 45 min · 50% context · breaker: default_

  - [x]* 7.3 Write property test for rollback fidelity
    - **Property CP-4: Rollback restores state byte-identical to the preceding Committed_Snapshot (Round Trip)**
    - Generators: `arbBundle` × `arbRequoteResults` with at least one rejecting leg
    - `numRuns: 200`
    - Assert byte-identical restoration of every leg and edge row, and that zero Cascade hold remains `reserved`
    - **Validates: Requirements 2.4, 2.5**
    - _Check: `npm run check:atomic-commit`_
    - _Scope: `tests/props/cp-04-rollback-fidelity.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 25k tokens · 3 iterations · 30 min · 35% context · breaker: default_

  - [x]* 7.4 Write property test for bundle serialization round trip
    - **Property CP-10: Bundle serialize → deserialize yields an identical leg and edge set (Round Trip)**
    - Generator: `arbBundle`
    - `numRuns: 200`
    - Assert round-trip identity by property, not by example, including empty-edge and maximum-size arms; assert state survives a simulated hibernation cycle
    - **Validates: Requirements 2.4, 8.6**
    - _Check: `npm run check:atomic-commit`_
    - _Scope: `tests/props/cp-10-bundle-round-trip.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 20k tokens · 3 iterations · 25 min · 35% context · breaker: default_

- [x] 8. Guardrail envelope adapter — one data dependency, zero interface change

  - [x] 8.1 Implement `guardrail-envelope-adapter.ts`
    - Supply Available_Balance to the reused Guardrail Gate without altering any gate operation, parameter, or return value
    - Read Balance_Cache on the fast path; fall back to the Envelope Ledger on miss
    - On divergence between cache and ledger, use the ledger value and invalidate the cached entry
    - Reject the offer with `envelope-unavailable` when the ledger cannot be read; reserve zero holds; make the offer available to zero downstream component
    - Pass zero agent-specific gate parameter, preserving the commerce increment's per-agent parity guarantee
    - _Requirements: 4.8, 4.9, 5.5, 5.6, 12.2_
    - _Properties: CP-11_
    - _Check: `npm run check:reused-interfaces`_
    - _Scope: `src/gate/guardrail-envelope-adapter.ts`_
    - _Capability: local write_
    - _Bounds: 30k tokens · 3 iterations · 40 min · 40% context · breaker: default_

  - [x]* 8.2 Snapshot the reused interfaces and assert they are unchanged
    - Capture the public surface of Shared Canvas Node Store, Agent Registry/Router, Discovery Harnesses, Issuance Service, Settlement Verifier, Notification Dispatcher, Marketplace Registry Canvas, and Guardrail Gate
    - Assert zero operation, parameter, or return value changed; report a typed `reused-interface-changed` finding naming the interface and element on any difference
    - Assert the inherited Component Inventory records a local and delivered rung per inherited component and re-claims a rung for zero inherited component
    - Review the current increment's PRD Component Inventory plus the design/task owner maps for Requirement 12.5; that authoring-document obligation is not emitted by `check:reused-interfaces`
    - _Requirements: 12.1, 12.2, 12.3, 12.4, 12.5, 12.6_
    - _Check: `npm run check:reused-interfaces` for 12.1–12.4/12.6; authoring-document inventory review for 12.5_
    - _Scope: `tests/snapshots/reused-interfaces.test.ts`_
    - _Capability: read, local execute_
    - _Bounds: 25k tokens · 3 iterations · 30 min · 35% context · breaker: default_

- [x] 9. Cascade orchestration and net settlement

  - [x] 9.1 Implement `reopt-dispatch.ts` — concurrent fan-out / fan-in with a wall-clock cap
    - Issue every Re_Quote for one affected set before awaiting any, and await them all together, so Cascade wall-clock is the slowest single leg rather than their sum
    - Issue every Re_Quote through the reused Agent Registry/Router interface; hold zero per-vertical harness identifier in source
    - Enforce a stated wall-clock cap; abort with `cascade-timeout` on exceeding it
    - Inspect every settled result after all branches settle; issue zero per-leg retry
    - Resolve every ambiguous or unrepresentable result as a rejection
    - _Requirements: 6.2, 6.3, 6.4, 6.5, 6.6_
    - _Properties: CP-3 (dispatch arm)_
    - _Check: `npm run check:cascade-bounds`_
    - _Scope: `src/bundle/reopt-dispatch.ts`_
    - _Capability: local write_
    - _Bounds: 35k tokens · 4 iterations · 45 min · 45% context · breaker: default_

  - [x] 9.2 Implement `reopt-worker.ts` — one pass per event, all outcome branches
    - Handle one Mutation_Event as exactly one pass; trigger zero further Cascade from within it, including from legs its own commit changed
    - Preflight: reject `store-unavailable` before any Re_Quote when the store is unreachable; propagate `unknown-leg` and `cyclic-dependency` unchanged
    - Return the `no-op` outcome for a changed leg with zero outgoing edges, issuing zero Re_Quote and zero settlement call
    - Key idempotence on `(bundleId, legId, eventId)`; return the recorded outcome for a repeat rather than re-running
    - On Re_Quote, guardrail, or definitive pre-effect settlement rejection: roll back and release the Cascade's holds; on a retryable or ambiguous settlement result: keep the prior projection visible, protect the holds, and persist `pending` recovery under the same idempotency key
    - Import zero model client; hold zero payment client and zero storage client of its own
    - Record per Cascade: Re_Quote count, reject count, abort reason where present, elapsed wall-clock
    - _Requirements: 2.5, 2.6, 6.1, 6.7, 6.8, 10.1_
    - _Properties: CP-9, CP-14_
    - _Check: `npm run check:cascade-bounds`_
    - _Scope: `src/bundle/reopt-worker.ts`_
    - _Capability: local write_
    - _Bounds: 45k tokens · 4 iterations · 55 min · 50% context · breaker: default_

  - [x] 9.3 Implement net settlement in `reopt-worker.ts`
    - Compute the net amount as the signed sum of new minus prior committed amount across the whole affected set, in integer minor units
    - Use one logical Issuance Service settlement identity per non-zero Cascade, regardless of affected-set size; the first definitive path issues one transport call and any ambiguous recovery retry reuses the same idempotency key and request identity
    - Issue zero settlement call for a zero net, recording a zero-net entry; issue zero settlement call for Cascades rolled back before the settlement phase
    - Roll back and release holds on a definitive non-retryable pre-effect rejection; retain protected holds and durable pending recovery for timeout, network, retryable status, or ambiguous response; never roll back an outcome that may have moved money
    - Record affected-set size, durable transport-attempt count, and the stable Cascade idempotency key so the success-path 1-per-cascade ratio and recovery retries are retrievable rather than inferred
    - Issue the operation through the isolated travel net-settlement boundary, which delegates to the inherited Issuance Service capability without altering any pre-existing public operation; hold zero StraitsX or Avalanche client
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8_
    - _Properties: CP-5_
    - _Check: `npm run check:net-settlement`_
    - _Scope: `src/bundle/reopt-worker.ts`_
    - _Capability: local write_
    - _Bounds: 35k tokens · 4 iterations · 45 min · 45% context · breaker: default_

  - [x]* 9.4 Write property test for the single-net-settlement claim
    - **Property CP-5: First-definitive-path call count is 1 for any non-zero net and any affected-set size; 0 for zero net or a pre-settlement rollback (Metamorphic)**
    - Generators: `arbBundle` × `arbRequoteResults`, varying affected-set size from 1 to 19 while holding the pass/reject pattern fixed
    - `numRuns: 400` — this is the literal L4 claim
    - Assert first-definitive-path call count is invariant to affected-set size and that the net equals the independently computed signed sum
    - **Validates: Requirements 3.1, 3.2, 3.3, 3.4**
    - _Check: `npm run check:net-settlement`_
    - _Scope: `tests/props/cp-05-net-settlement.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 30k tokens · 4 iterations · 40 min · 45% context · breaker: default_

  - [x]* 9.5 Write property test for cascade idempotence
    - **Property CP-9: A repeated Mutation_Event with the same event key produces at most one commit and one logical settlement identity; recovery transport attempts reuse the same identity and payload (Idempotence)**
    - Generator: `arbMutationSequence` — event sequences including exact repeats and near-repeats differing in one key field
    - `numRuns: 200`
    - Assert commit count ≤ 1 and settlement-call count ≤ 1 per event key, and that a near-repeat is treated as a distinct event
    - **Validates: Requirements 2.6**
    - _Check: `npm run check:cascade-bounds`_
    - _Scope: `tests/props/cp-09-cascade-idempotence.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 25k tokens · 3 iterations · 30 min · 35% context · breaker: default_

  - [x]* 9.6 Measure cascade bounds and assert recorded orchestration evidence
    - Assert Cascade wall-clock approximates the slowest single Re_Quote rather than their sum, with both values recorded
    - Assert the per-Cascade record carries Re_Quote count, reject count, abort reason where present, and elapsed time
    - Assert one pass per event: zero recursive Cascade is triggered by a commit
    - Assert every payment-path request carries the Issuance Service component identifier at the mocked gateway boundary
    - _Requirements: 3.8, 6.1, 6.2, 6.7_
    - _Check: `npm run check:cascade-bounds`, `npm run check:net-settlement`_
    - _Scope: `tests/integration/cascade-bounds.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 30k tokens · 3 iterations · 35 min · 40% context · breaker: default_

  - [x]* 9.7 Prove provider-backed success and reconciliation custody
    - Assert the isolated settlement boundary treats a journal row as zero proof of effect and returns success only when the executor receipt exactly matches signed amount, currency, Cascade, Bundle, Principal, effect direction, settlement identifier, and provider reference
    - Fault-inject mismatched, missing, malformed, retryable, timeout, and typed idempotency-conflict responses; assert none can become settlement success and ambiguous outcomes preserve identical identity for recovery
    - Assert ambiguous attempted effect enters non-expiring quarantine, remains unavailable across a >24h alarm, and is reachable only through the separately authenticated operator path
    - Assert possible-effect custody is durable before attempt recording/dispatch, expiry alarms exclude it, and a forced alarm-before-recovery leaves value unavailable until atomic quarantine
    - Assert an exact audited idempotent commit or release decision converges both ledger and graph, rejects decision drift, and permits a later unique mutation
    - _Requirements: 3.7, 3.9, 3.10, 3.11, 5.1, 5.2, 5.3_
    - _Check: `npm run travel-commerce:services:test` and `npm run travel-commerce:settlement-executor:test` for the exact provider-effect contract; `npm run travel-commerce:test -- --disableConsoleIntercept cloudflare/workers/knowgrph-travel-commerce/test/core-recovery-regressions.test.ts cloudflare/workers/knowgrph-travel-commerce/test/reconciliation-custody.test.ts` for journal distrust, custody, and convergence_
    - _Scope: payment/provider-executor net-settlement contract tests, `cloudflare/workers/knowgrph-travel-commerce/test/core-recovery-regressions.test.ts`, and `cloudflare/workers/knowgrph-travel-commerce/test/reconciliation-custody.test.ts`_
    - _Capability: local execute_
    - _Bounds: 30k tokens · 3 iterations · 40 min · 40% context · breaker: default_

- [x] 10. Checkpoint — cascade hot path dev-proven before any cache lands
  - Ensure all tests pass; ask the user if questions arise
  - Confirm CP-3, CP-4, CP-5, CP-9, CP-10 report recorded seeds, and that the 2-leg cascade commits and rolls back correctly against the DO double
  - This checkpoint is the gate the source specification names: caches land only after the core hot path is dev-proven
  - _Requirements: 2.3, 3.1_
  - _Check: `npm run travel-commerce:test && npm run check:atomic-commit && npm run check:net-settlement`_
  - _Scope: none (read only)_
  - _Capability: read, local execute_
  - _Bounds: 10k tokens · 2 iterations · 15 min · 25% context · breaker: default_
