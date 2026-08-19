---
title: "Knowgrph Agentic Travel Commerce Platform — Requirements"
doc_type: "Spec Requirements"
schema: "kiro-spec-requirements/v1"
version: "1.0.0"
date: "2026-08-19"
lang: "en-US"
frontmatter_contract: "required"
spec_type: "feature"
workflow_type: "requirements-first"
feature_name: "knowgrph-agentic-travel-commerce-platform"
owner: "Solo Founder / AI Orchestrator"
lane: "authoring"
local_rung: "spec-complete"
delivered_rung: "undocumented"
deploy_boundary: "dev-only"
source_specification: "joohwee/prd-tad-ard/knowgrph-agentic-travel-commerce-platform-prd-tad-adr.md v1.0.0"
inherited_specifications:
  - ".kiro/specs/knowgrph-agentic-travel-agencies/requirements.md (travel doc v0.6.0)"
  - ".kiro/specs/knowgrph-agentic-commerce-platform/requirements.md (commerce doc v0.1.0)"
governing_contracts:
  - "huijoohwee.github.io/guidelines/agentic-sdlc-guidelines.md"
  - "agentic-canvas-os/docs/START-WORKFLOW.md"
  - "agentic-canvas-os/docs/AGENTS.md"
---

# Requirements Document

## Introduction

This document derives executable requirements for the consolidation increment specified in `knowgrph-agentic-travel-commerce-platform-prd-tad-adr.md` v1.0.0. That specification adds exactly three things neither prior document had: the L4 **Bundle Re-Optimization Engine**, the **Budget Envelope Ledger** primitive underneath it, and a dedicated **cache / computation / memory** performance strategy — plus an infrastructure ADR retiring Oracle Always Free ARM from the critical path.

Scope discipline: every requirement below traces to a named PRD user story and its VCC translation, an ADR, a row of the Journey → System Mapping or Topology table, a Performance Enhancement recommendation, or a Deploy Boundary Register row. No requirement introduces behavior absent from the source specification. Where the source states an honest gap or an Open Question, this document states the same gap as a bounded requirement or a blocked operator decision rather than closing it silently.

Reuse discipline: Shared Canvas Node Store, Agent Registry/Router, Discovery Harnesses, Issuance Service, Settlement Verifier, Notification Dispatcher, and Marketplace Registry Canvas are consumed at their existing interfaces. Guardrail Gate gains **one new data dependency** (Envelope Ledger) and **no interface change**. Those components keep whatever readiness rung they already earned; nothing here re-claims a rung on their behalf.

Deploy boundary: every requirement is satisfied entirely within the Dev lane (`GitHub/knowgrph`, `npm run dev:apex`, `npm run dev`). The Prod mirror (`GitHub/huijoohwee/content/knowgrph`) and the Cloudflare routes (`airvio.co`, `airvio.co/knowgrph`) are gated deploy targets and are never acceptance criteria for any requirement in this document.

## Glossary

- **Bundle**: One multi-leg itinerary owned by exactly one principal, identified by `bundle_id`.
- **Leg**: One committed or candidate unit of a Bundle (a flight, a hotel night, a transfer, a local experience), identified by `leg_id`, holding exactly one current `offer_id` when committed.
- **Edge**: A directed causal dependency from an upstream Leg to a downstream Leg, stored as a row in the flat `edges` table. Direction means "changing the source may invalidate the target".
- **Bundle_Graph_Store**: The new Durable Object holding `legs`/`edges` for one `bundle_id`, exposing bounded downstream BFS plus atomic commit and rollback. PRD Topology node "Bundle Graph Store".
- **Affected_Set**: The set of Legs reachable by following outgoing Edges from a Changed_Leg, excluding the Changed_Leg itself. The subject of PRD US-1's VCC.
- **Changed_Leg**: The single Leg named by a Mutation_Event as having changed.
- **Mutation_Event**: A Shared Canvas Node mutation notification carrying exactly one `leg_id`. The only trigger for re-optimization in this increment.
- **Reopt_Worker**: The new Worker orchestrating one cascade: BFS, parallel re-quote, envelope check, atomic commit, one net settlement call, archive write. PRD Topology node "Re-optimization Worker".
- **Cascade**: One complete Reopt_Worker execution for one Mutation_Event.
- **Re_Quote**: One Discovery dispatch, issued through the reused Agent Registry/Router, requesting a replacement offer for one Leg in the Affected_Set.
- **Committed_Snapshot**: The persisted `legs`/`edges` state of one Bundle at its last successful commit; the sole rollback target.
- **Envelope_Ledger**: The new Durable Object holding `envelope`/`holds` for one `principal_id`, exposing atomic check-and-reserve. PRD Topology node "Envelope Ledger".
- **Hold**: One record in the `holds` table in state `reserved`, `committed`, or `released`, carrying an amount and an `offer_id`.
- **Available_Balance**: `total_budget` minus the sum of all `reserved` and `committed` Hold amounts for that principal at that instant.
- **Guardrail_Gate**: The reused deterministic component evaluating one offer against the principal's budget, now reading Available_Balance from Envelope_Ledger. Interface unchanged.
- **Issuance_Service**: The reused MCP/SSE harness; the sole caller of the StraitsX Card MCP Gateway. Reused unmodified.
- **Net_Settlement_Call**: The single Issuance_Service call carrying the net signed delta of one Cascade.
- **Balance_Cache**: The Cloudflare KV read-through cache of Available_Balance. Never a source of truth.
- **Offer_Cache**: The Cloudflare Cache API cache of Discovery Harness offers, short TTL with stale-while-revalidate.
- **Provenance_Archive**: The R2 write-once store of committed Bundle snapshots and provenance receipts.
- **Scale_Boundary**: The declared per-Bundle limit of 20 Legs and 20 Edges, stated openly per ADR-4 rather than silently assumed.
- **Cost_Log**: The recorded per-Cascade cost entry naming prompt tokens, completion tokens, and dollar cost.
- **Inference_Router**: The component selecting Cloudflare Workers AI (primary) or Cloudflare Containers running Ollama (overflow) for a probe-tree L1 inference call, per ADR-3.
- **Permitted_Model_Set**: The subset of Workers AI catalog entries whose weights are licensed Apache-2.0 or MIT. The only set Inference_Router may route to on the primary path.
- **Dev_Lane**: The Knowgrph development runtime at `GitHub/knowgrph`, started through repository-owned scripts.
- **Deploy_Boundary**: A recorded gate between lanes whose state must read `closed` absent an explicit recorded operator instruction.
- **Bundle_Commit_Deploy_Boundary**: Deploy Boundary Register row "Bundle commit: affected-set → committed".
- **Envelope_Mutation_Deploy_Boundary**: Deploy Boundary Register row "Envelope mutation: offer → hold".
- **Mirror_Delivery_Deploy_Boundary**: The gate from the Prod mirror to the Cloudflare delivery routes. Reports `closed` absent a recorded exact-candidate human authorization.
- **VCC**: Verifiable Completion Condition — a named check plus a recorded result an independent evaluator judges from surfaced output.

## Requirements

### Requirement 1: Downstream-Only Affected-Set Precision

**User Story:** As a Shopper-Agent Principal, I want a change to one committed leg to automatically trigger re-planning of only the legs causally downstream of it, so that I do not have to notice and fix every knock-on effect myself.

Traces to: PRD Bundle Re-Optimization US-1 and its VCC; Computation recommendation "Affected-set walk"; ADR-4.

#### Acceptance Criteria

1. WHEN Reopt_Worker receives a Mutation_Event naming a Changed_Leg present in Bundle_Graph_Store, THE Reopt_Worker SHALL compute an Affected_Set equal to exactly the set of Legs reachable by following one or more outgoing Edges from that Changed_Leg.
2. THE Reopt_Worker SHALL include zero Legs in the Affected_Set that are not reachable from the Changed_Leg by outgoing Edges, and SHALL omit zero Legs that are so reachable.
3. THE Reopt_Worker SHALL exclude the Changed_Leg itself from the Affected_Set, and SHALL issue zero Re_Quote calls for the Changed_Leg.
4. THE Reopt_Worker SHALL traverse each Leg at most once per Cascade, terminating the walk when no unvisited outgoing Edge remains.
5. WHEN the Changed_Leg has zero outgoing Edges, THE Reopt_Worker SHALL return a no-op result, SHALL issue zero Re_Quote calls, SHALL issue zero Net_Settlement_Calls, and SHALL leave the Committed_Snapshot unmodified.
6. THE Reopt_Worker SHALL complete the Affected_Set walk within 50 milliseconds for a Bundle of 8 or fewer Legs, measured inside the Durable Object with no intervening network hop.
7. IF the Changed_Leg named by a Mutation_Event is absent from Bundle_Graph_Store, THEN THE Reopt_Worker SHALL return a typed rejection carrying an unknown-leg reason, SHALL issue zero Re_Quote calls, and SHALL leave the Committed_Snapshot unmodified.
8. IF the `edges` table for a Bundle contains a cycle reachable from the Changed_Leg, THEN THE Reopt_Worker SHALL return a typed rejection carrying a cyclic-dependency reason, SHALL issue zero Re_Quote calls, and SHALL leave the Committed_Snapshot unmodified.
9. THE Reopt_Worker SHALL record, for every Cascade including no-op and rejected ones, one Session_Log entry carrying the `bundle_id`, the Changed_Leg identifier, the resulting Affected_Set identifiers in traversal order, and the outcome.

#### Stated Gap (carried from PRD Open Question, not closed here)

10. THE Reopt_Worker SHALL treat every Mutation_Event as trigger-worthy without applying a materiality threshold, and SHALL record that no threshold is configured, until an operator decision defines one. THE Reopt_Worker SHALL NOT apply an invented default threshold.

### Requirement 2: Atomic All-Or-Nothing Affected-Set Commit

**User Story:** As a Shopper-Agent Principal, I want the entire affected set re-quoted, approved, and settled as one atomic operation, so that I never end up in a half-changed state.

Traces to: PRD Bundle Re-Optimization US-2 and its VCC; Success Metrics "Partial-commit incidents = 0" and "Rollback correctness = 100%"; Bundle_Commit_Deploy_Boundary.

#### Acceptance Criteria

1. WHEN every Leg in the Affected_Set has a Re_Quote result that passed Guardrail_Gate, THE Bundle_Graph_Store SHALL commit the new `offer_id` for every Leg in that Affected_Set in one atomic storage transaction.
2. IF any single Leg in the Affected_Set has a Re_Quote result that Guardrail_Gate rejected, is absent, or is malformed, THEN THE Bundle_Graph_Store SHALL commit zero new `offer_id` values for that Cascade and SHALL restore the Committed_Snapshot.
3. THE Bundle_Graph_Store SHALL expose zero externally observable states in which some Legs of one Affected_Set carry a new `offer_id` while other Legs of the same Affected_Set carry a stale `offer_id`.
4. WHEN a Cascade rolls back, THE Bundle_Graph_Store SHALL restore every Leg and Edge row to a state byte-identical to the Committed_Snapshot that preceded that Cascade.
5. WHEN a Cascade rolls back, THE Reopt_Worker SHALL transition every Hold it reserved for that Cascade to `released`, and SHALL leave zero Holds for that Cascade in state `reserved`.
6. THE Reopt_Worker SHALL treat a repeated Mutation_Event carrying the same `bundle_id`, Changed_Leg identifier, and event identifier as idempotent, producing at most one commit and at most one Net_Settlement_Call across all repeats.
7. WHEN a commit succeeds, THE Reopt_Worker SHALL write the resulting Bundle snapshot and its provenance receipt to Provenance_Archive exactly once for that Cascade.
8. IF the Provenance_Archive write fails after a successful commit, THEN THE Reopt_Worker SHALL retain the commit, SHALL record a typed archive-deferred entry naming the Cascade, and SHALL NOT roll back the committed Legs.
9. THE Bundle_Commit_Deploy_Boundary SHALL report `closed` at the start and at the end of every Cascade, and THE Reopt_Worker SHALL commit only through the transactional path named in criterion 1.

### Requirement 3: One Net Settlement Call Per Cascade

**User Story:** As a Shopper-Agent Principal, I want one net settlement call for the whole cascade instead of one per changed leg, so that re-planning a 4-leg cascade costs one card top-up or refund action, not four.

Traces to: PRD Bundle Re-Optimization US-3 and its VCC; MoSCoW "Must — single net settlement call per cascade — this is the literal L4 claim".

#### Acceptance Criteria

1. WHEN a Cascade commits, THE Reopt_Worker SHALL issue exactly 1 Issuance_Service settlement call for that Cascade, regardless of the Affected_Set size.
2. THE Reopt_Worker SHALL compute the Net_Settlement_Call amount as the signed sum of every committed Leg's new offer amount minus that Leg's prior committed offer amount, across the whole Affected_Set.
3. WHEN the computed net amount is zero, THE Reopt_Worker SHALL issue zero Issuance_Service settlement calls and SHALL record a zero-net entry naming the Cascade.
4. IF a Cascade rolls back, THEN THE Reopt_Worker SHALL issue zero Issuance_Service settlement calls for that Cascade.
5. THE Reopt_Worker SHALL be the only component issuing a settlement call arising from a Cascade, and SHALL issue that call through the reused Issuance_Service interface unchanged.
6. THE Reopt_Worker SHALL record, per Cascade, the Affected_Set size and the settlement-call count, so that the ratio asserted in criterion 1 is retrievable from surfaced output rather than inferred.
7. IF the Issuance_Service call fails, THEN THE Reopt_Worker SHALL roll back the commit per Requirement 2.4, SHALL release the Cascade's Holds per Requirement 2.5, and SHALL record a typed settlement-failed reason.
8. THE Reopt_Worker SHALL hold zero configured StraitsX or Avalanche client of its own, and every payment-path request arising from a Cascade SHALL carry the Issuance_Service component identifier at the gateway boundary.

### Requirement 4: Atomic Check-And-Reserve Against True Available Balance

**User Story:** As a Shopper-Agent Principal, I want every offer from any registered agent to atomically check and reserve against my true remaining budget, accounting for every concurrent hold, so that two simultaneous agent offers can never jointly overspend my limit.

Traces to: PRD Budget Envelope Ledger US-1 and its VCC; Success Metrics "Over-envelope commits = 0"; ADR-1.

#### Acceptance Criteria

1. WHEN Guardrail_Gate evaluates an offer, THE Envelope_Ledger SHALL compute Available_Balance as `total_budget` minus the sum of all Hold amounts in state `reserved` or `committed` for that `principal_id` at that instant.
2. THE Envelope_Ledger SHALL perform the balance check and the Hold reservation as one indivisible operation, such that no other reservation for the same `principal_id` can interleave between them.
3. THE Envelope_Ledger SHALL accept an offer only when its amount is less than or equal to the Available_Balance computed in the same indivisible operation, and SHALL reject it otherwise.
4. THE Envelope_Ledger SHALL serialize every mutation for one `principal_id` through exactly one Durable Object instance keyed on that `principal_id`.
5. WHEN two or more offers for the same `principal_id` are submitted concurrently and their amounts sum above Available_Balance, THE Envelope_Ledger SHALL accept only the subset whose cumulative amount remains within Available_Balance, and SHALL reject the remainder with a typed insufficient-envelope reason.
6. THE Envelope_Ledger SHALL contain zero committed Hold sets whose summed amount exceeds `total_budget` for that `principal_id`, at any observable instant.
7. THE Envelope_Ledger SHALL complete a check-and-reserve within 10 milliseconds measured inside the Durable Object with no D1 hop.
8. IF the Envelope_Ledger cannot be reached or its state cannot be read, THEN THE Guardrail_Gate SHALL reject the offer with a typed envelope-unavailable reason, SHALL reserve zero Holds, and SHALL make the offer available to zero downstream components.
9. THE Guardrail_Gate SHALL retain its existing operation names, parameters, and return values unchanged, adding Envelope_Ledger as a data dependency only.

### Requirement 5: Hold Lifecycle And Immediate Release Visibility

**User Story:** As a Shopper-Agent Principal, I want a released or rolled-back leg's held funds to become available to other in-flight agents immediately, so that a rejected re-plan does not leave my budget artificially locked.

Traces to: PRD Budget Envelope Ledger US-2 and its VCC; MoSCoW "Must — hold lifecycle"; Should "KV-cached available_balance read".

#### Acceptance Criteria

1. THE Envelope_Ledger SHALL transition a Hold only along `reserved → committed` or `reserved → released`, and SHALL reject every other transition with a typed illegal-transition reason.
2. THE Envelope_Ledger SHALL treat a repeated transition request for a Hold already in the requested terminal state as a no-op returning the current state, mutating nothing.
3. WHEN a Hold transitions to `released`, THE Envelope_Ledger SHALL exclude its amount from the next computed Available_Balance for that `principal_id`, with no externally observable staleness window beyond the Durable Object's own write-then-read consistency.
4. THE Envelope_Ledger SHALL maintain the invariant `total_budget = Available_Balance + sum(reserved) + sum(committed)` after every transition.
5. THE Guardrail_Gate SHALL read Available_Balance from Balance_Cache on its fast path, and SHALL fall back to the Envelope_Ledger on a cache miss.
6. THE Guardrail_Gate SHALL treat Balance_Cache as non-authoritative: WHEN a Balance_Cache value and an Envelope_Ledger value diverge for the same `principal_id`, THE Guardrail_Gate SHALL use the Envelope_Ledger value and SHALL invalidate the cached entry.
7. THE Envelope_Ledger SHALL invalidate the Balance_Cache entry for a `principal_id` on every Hold transition for that principal, before returning the transition result.
8. THE Envelope_Ledger SHALL reserve, commit, and release Holds only through the serialized path named in Requirement 4.2, and SHALL expose zero direct `holds` table write operation outside it, satisfying the Envelope_Mutation_Deploy_Boundary operator instruction.
9. THE Envelope_Mutation_Deploy_Boundary SHALL report `closed` at the start and at the end of this increment.

#### Stated Gap (carried from PRD Open Question, not closed here)

10. THE Envelope_Ledger SHALL expose Available_Balance to server-side callers only, and SHALL expose zero client-facing read scope for it, until an operator decision defines whether a Shopper Client may read it. THE Envelope_Ledger SHALL NOT introduce a client-facing auth scope for Available_Balance in advance of that decision.

### Requirement 6: Bounded Cascade Orchestration And Fail-Closed Circuit Breaker

**User Story:** As a Platform Operator, I want one cascade to be a single bounded pass with a stated breaker, so that a re-optimization can never recurse, retry unboundedly, or partially retry its way into a mixed state.

Traces to: PRD Orchestration/Harness Flows "Bundle Re-Optimization Pipeline"; Computation recommendation "Re-dispatch via `Promise.all`"; PRD fail-closed posture inherited from the travel document.

#### Acceptance Criteria

1. THE Reopt_Worker SHALL execute exactly 1 pass per Mutation_Event, and SHALL trigger zero further Cascades from within that pass, including from Legs its own commit changed.
2. THE Reopt_Worker SHALL dispatch every Re_Quote for one Affected_Set concurrently and SHALL wait for all of them before evaluating any of them, so that Cascade wall-clock time is bounded by the slowest single Re_Quote rather than their sum.
3. WHEN any Re_Quote in a Cascade is rejected, THE Reopt_Worker SHALL abort the whole Affected_Set, SHALL roll back per Requirement 2.4, and SHALL issue zero partial retries of individual Legs.
4. THE Reopt_Worker SHALL bound each Cascade by a stated wall-clock cap and SHALL abort with a typed cascade-timeout reason on exceeding it, rolling back per Requirement 2.4.
5. THE Reopt_Worker SHALL resolve every ambiguous or unrepresentable Re_Quote result as a rejection, and SHALL treat zero ambiguous result as a pass.
6. THE Reopt_Worker SHALL issue every Re_Quote through the reused Agent Registry/Router interface, and SHALL hold zero per-vertical harness identifier in source.
7. THE Reopt_Worker SHALL record, per Cascade, the Re_Quote count, the reject count, the abort reason where present, and the elapsed wall-clock time.
8. IF Bundle_Graph_Store is unreachable at the start of a Cascade, THEN THE Reopt_Worker SHALL return a typed store-unavailable rejection, SHALL issue zero Re_Quote calls, and SHALL mutate nothing.

#### Stated Gap (carried from PRD Open Question, not closed here)

9. WHEN a Cascade rolls back, THE Reopt_Worker SHALL record the rollback in the Session_Log and SHALL emit zero principal notification, until an operator decision defines whether rollback notification is synchronous or queued through the reused Notification Dispatcher. THE Reopt_Worker SHALL NOT choose a notification path by default.

### Requirement 7: Flat Adjacency Storage And Explicit Scale Boundary

**User Story:** As a Platform Operator, I want the bundle dependency structure stored as flat relational tables with a written-down scale limit, so that "no graph database" is an honest engineering constraint rather than an unexamined assumption.

Traces to: ADR-4 (Accepted); PRD Component Configuration "scale boundary asserted at ~20 legs/edges per bundle"; Memory recommendation "Explicit scale boundary".

#### Acceptance Criteria

1. THE Bundle_Graph_Store SHALL persist Bundle structure as two flat tables, `legs` and `edges`, and SHALL introduce zero dedicated graph engine, graph query language, or graph-specific storage system at any layer.
2. THE Bundle_Graph_Store SHALL walk Edges in memory over rows read from those two tables, and SHALL execute zero graph-engine traversal call.
3. THE Bundle_Graph_Store SHALL reject a Leg insertion that would take a Bundle above 20 Legs with a typed scale-boundary reason naming the limit and the observed count, and SHALL leave the Bundle unmodified.
4. THE Bundle_Graph_Store SHALL reject an Edge insertion that would take a Bundle above 20 Edges with a typed scale-boundary reason naming the limit and the observed count, and SHALL leave the Bundle unmodified.
5. THE Bundle_Graph_Store SHALL reject an Edge insertion that would introduce a cycle into a Bundle with a typed cyclic-dependency reason, and SHALL leave the Bundle unmodified.
6. THE Bundle_Graph_Store SHALL scope exactly one Durable Object instance per `bundle_id`, and SHALL hold Legs of two different `bundle_id` values in zero shared instance.
7. THE Bundle_Graph_Store SHALL reject a Bundle whose Legs reference more than one `principal_id` with a typed cross-principal reason, consistent with the PRD's explicit out-of-scope statement for cross-principal bundles.
8. THE Bundle_Graph_Store SHALL surface the configured Scale_Boundary values as readable named constants, so that a check asserts against the declared limit rather than against a value it hardcoded itself.
9. THE Bundle_Graph_Store SHALL maintain a topological order of Legs incrementally on Edge insertion, and SHALL recompute a full topological sort zero times per Mutation_Event.
10. THE Bundle_Graph_Store SHALL produce, for any sequence of Edge insertions yielding the same Edge set, a topological order consistent with that Edge set and identical to the order a full recompute over that Edge set would yield under the same deterministic tie-break rule.

### Requirement 8: Hot-Path Storage Placement And Aggregate-Only D1

**User Story:** As a Platform Operator, I want hot-path ledger and graph state colocated with its actor and D1 restricted to aggregate reporting, so that every hot mutation avoids a network hop and gets atomicity from the actor model instead of hand-written transactions.

Traces to: ADR-1; Cache recommendation "Durable Object storage (SQLite)".

#### Acceptance Criteria

1. THE Bundle_Graph_Store SHALL persist `legs` and `edges` in its own Durable Object SQLite storage, and SHALL issue zero D1 query on the Cascade hot path.
2. THE Envelope_Ledger SHALL persist `envelope` and `holds` in its own Durable Object SQLite storage, and SHALL issue zero D1 query on the check-and-reserve path.
3. THE Dev_Lane SHALL restrict D1 access to cross-key aggregate and reporting queries, and SHALL reject a D1 read or write issued from the Cascade or check-and-reserve path with a typed storage-placement reason naming the calling component.
4. THE Bundle_Graph_Store and THE Envelope_Ledger SHALL each rely on their Durable Object's single-threaded per-key execution for atomicity, and SHALL introduce zero external lock, lease, or advisory-locking mechanism for hot-path mutation.
5. THE Bundle_Graph_Store SHALL build its in-memory adjacency list once per Durable Object wake and SHALL hold it for that instance's active lifetime, rebuilding it zero times per Mutation_Event within one wake.
6. THE Bundle_Graph_Store and THE Envelope_Ledger SHALL restore correct state from their SQLite storage after hibernation, with zero cold-rebuild logic required by a caller.
7. THE Shared Canvas Node client push SHALL use hibernatable WebSockets, so that an idle Durable Object releases from memory without losing state.
8. THE Dev_Lane SHALL introduce zero storage system beyond the Durable Object SQLite storage, KV, Cache API, R2, and the already-provisioned D1 and Yjs store.

### Requirement 9: Three-Layer Edge Cache Correctness

**User Story:** As a Platform Operator, I want each cache matched to its data's actual access pattern with correctness stated as a requirement, so that cache invalidation is a tested surface rather than an assumed free win.

Traces to: ADR-2; Cache recommendations for KV, Cache API, and R2; ADR-2 Consequences "cache invalidation logic is a real correctness surface to test, not free".

#### Acceptance Criteria

1. THE Balance_Cache SHALL store only Available_Balance keyed by `principal_id`, and SHALL be read only by Guardrail_Gate's fast path.
2. THE Balance_Cache SHALL never be treated as a source of truth, per Requirement 5.6, and THE Dev_Lane SHALL reject any commit decision derived from a Balance_Cache read that was not confirmed against the Envelope_Ledger.
3. THE Offer_Cache SHALL cache Discovery Harness offers with a time-to-live between 30 and 60 seconds inclusive, and SHALL serve a stale entry only under stale-while-revalidate while a revalidation is in flight.
4. THE Reopt_Worker SHALL commit zero Leg against an Offer_Cache entry whose age exceeds its stated time-to-live and whose revalidation has not completed.
5. THE Offer_Cache SHALL key entries on the full Re_Quote request identity, such that two Re_Quotes differing in any request field resolve to different entries.
6. THE Offer_Cache SHALL reduce Discovery Harness dispatches for repeated identical Re_Quotes within the time-to-live window, and THE Dev_Lane SHALL record dispatch counts with and without the cache so the reduction is retrievable from surfaced output.
7. THE Provenance_Archive SHALL write each committed Bundle snapshot and provenance receipt exactly once under a Cascade-derived key, and SHALL reject an overwrite of an existing key with a typed archive-immutable reason.
8. THE Dev_Lane SHALL keep active Bundle state in Durable Object storage and archived Bundle state in Provenance_Archive, and SHALL retain zero archived-only snapshot in Durable Object or D1 storage.
9. THE Dev_Lane SHALL introduce zero vendor outside Cloudflare for any of the three cache layers.
10. THE Reopt_Worker SHALL cache Agent Definition lookups in Worker memory and Balance_Cache-class KV, and SHALL invalidate those entries only on agent registration or de-registration.

### Requirement 10: Model-Free Determinism And Cost Observability

**User Story:** As a Solo Founder, I want the cascade and ledger arithmetic to be provably model-free with a recorded per-cascade cost, so that the L4 claim does not quietly add a token bill.

Traces to: PRD Orchestration/Harness Flows token budgets "0 prompt + 0 completion = $0.00/call" for both new pipelines; operating priority "token economics".

#### Acceptance Criteria

1. THE Reopt_Worker, THE Bundle_Graph_Store, and THE Envelope_Ledger SHALL import zero model client and SHALL issue zero model call.
2. THE Reopt_Worker SHALL emit one Cost_Log entry per Cascade recording prompt tokens, completion tokens, and dollar cost attributable to the Cascade's own orchestration, and SHALL record `0`, `0`, and `0.00` for a Cascade whose Re_Quotes returned from cache or from a deterministic harness path.
3. THE Cost_Log SHALL attribute Discovery Harness token cost to the harness that incurred it, and SHALL attribute zero harness token cost to the Reopt_Worker itself.
4. THE Reopt_Worker SHALL produce an identical Affected_Set, commit decision, and net amount for identical Bundle state, Mutation_Event, and Re_Quote results.
5. THE Envelope_Ledger SHALL produce an identical accept or reject outcome for identical `total_budget`, Hold set, and offer amount.
6. THE Dev_Lane SHALL reject a build in which a model client is reachable from the Reopt_Worker, Bundle_Graph_Store, or Envelope_Ledger module graph, naming the importing module.
7. THE Cost_Log SHALL be emitted for rejected and rolled-back Cascades as well as committed ones, so that a failed Cascade's cost is not invisible.

### Requirement 11: Inference Provider Consolidation With Explicit FOSS License Filtering

**User Story:** As a Solo Founder, I want probe-tree L1 inference consolidated onto Cloudflare with a real per-model license filter, so that retiring Oracle does not silently import a non-OSI-licensed model into a FOSS-first stack.

Traces to: ADR-3 (Proposed) including its explicit FOSS-hard-gate flag; ADR-3 Consequences.

#### Acceptance Criteria

1. THE Inference_Router SHALL route probe-tree L1 inference calls to Cloudflare Workers AI as its primary path, and SHALL issue zero call to an Oracle-hosted inference endpoint.
2. THE Inference_Router SHALL route to Cloudflare Containers running self-hosted Ollama only as an overflow path, and only for a model absent from the Permitted_Model_Set on the primary path.
3. THE Inference_Router SHALL restrict primary-path model selection to the Permitted_Model_Set, defined as catalog entries whose weights are licensed Apache-2.0 or MIT.
4. IF a requested model's declared license is neither Apache-2.0 nor MIT, THEN THE Inference_Router SHALL reject the primary-path route with a typed license-excluded reason naming the model identifier and its declared license, and SHALL issue zero primary-path call for that model.
5. THE Inference_Router SHALL read model license declarations from externalized configuration, and SHALL hold zero model identifier or license string hardcoded in source.
6. THE Inference_Router SHALL record, per inference call, the selected path, the model identifier, the declared license, and the recorded cost, so that license compliance is auditable from surfaced output.
7. IF the license configuration cannot be read, THEN THE Inference_Router SHALL reject every route with a typed license-configuration-unavailable reason, and SHALL fall back to zero unfiltered model.
8. THE Dev_Lane SHALL record that Workers AI is metered beyond its free allocation and that Containers overflow is metered compute, and SHALL claim zero-cost inference for neither path.
9. THE Dev_Lane SHALL hold zero Oracle endpoint, credential key name, or SSH configuration for an inference path.

### Requirement 12: Reused-Component Interface Preservation

**User Story:** As a Solo Founder, I want the reused components consumed at their existing interfaces, so that this increment inherits their earned rungs instead of invalidating them.

Traces to: PRD "Reused components (unchanged interfaces, extended data dependency only for Guardrail Gate)"; PRD Component Inventory rung inheritance.

#### Acceptance Criteria

1. THE Dev_Lane SHALL preserve the Shared Canvas Node Store, Agent Registry/Router, Discovery Harnesses, Issuance_Service, Settlement Verifier, Notification Dispatcher, and Marketplace Registry Canvas interfaces unchanged, adding, removing, or altering zero operation, parameter, or return value.
2. THE Guardrail_Gate SHALL gain exactly one new data dependency, the Envelope_Ledger, and SHALL change zero element of its own interface.
3. THE Dev_Lane SHALL detect a change to any reused interface as a typed reused-interface-changed finding naming the interface and the changed element.
4. THE Reopt_Worker SHALL consume the Shared Canvas Node mutation event at its existing shape, and SHALL require zero new field from it beyond the `leg_id` the PRD names.
5. THE Component Inventory SHALL record, for every component in this increment, its local rung and delivered rung and the document it inherits them from, and SHALL re-claim a rung for zero inherited component.
6. THE Dev_Lane SHALL record `delivered_rung: undocumented` for every component in this increment until a delivered Evidence Reference exists.

### Requirement 13: Dev-Only Deploy Boundary Discipline

**User Story:** As a Solo Founder, I want every deploy boundary in this increment to read closed and to be reversible on paper before anything runs, so that no cascade or ledger mutation can leak past the Dev lane.

Traces to: PRD Deploy Boundary Register (both rows, both `closed`); governing guidelines' Tool Permission & Blast Radius; START-WORKFLOW deploy gates.

#### Acceptance Criteria

1. THE Dev_Lane SHALL report the Bundle_Commit_Deploy_Boundary, the Envelope_Mutation_Deploy_Boundary, and the Mirror_Delivery_Deploy_Boundary as `closed` at the start and at the end of this increment.
2. THE Dev_Lane SHALL derive each Deploy_Boundary state from a recorded Evidence Reference, and SHALL report `closed` when the corresponding Evidence Reference is absent.
3. THE Dev_Lane SHALL mutate the Prod mirror path zero times and SHALL issue zero request to a Cloudflare delivery route during any task in this increment.
4. IF a component attempts a Prod mirror write or a Cloudflare delivery-route mutation, THEN THE Dev_Lane SHALL reject it before the request is issued and SHALL record the requesting component identifier and the target boundary.
5. THE Dev_Lane SHALL state a rollback statement for each Deploy_Boundary row: reverting Bundle_Graph_Store to its last Committed_Snapshot for bundle commit, and transitioning the Hold to `released` for envelope mutation.
6. THE Dev_Lane SHALL require a recorded exact-candidate human authorization before the Mirror_Delivery_Deploy_Boundary may report anything other than `closed`, and SHALL infer, default, schedule, or simulate that authorization zero times.
7. THE Dev_Lane SHALL persist zero developer-specific absolute path, credential value, account identifier, or environment-specific default in source, fixtures, tests, or generated assets.

### Requirement 14: Mobile-First, Local-First Re-Plan Surface

**User Story:** As a Shopper-Agent Principal on a phone, I want to see what a cascade changed and what it cost, offline-tolerant and without horizontal scrolling, so that autonomous re-planning stays legible rather than opaque.

Traces to: operating priorities "browser-based, mobile-first, local-first, offline-first"; PRD reuse of Shared Canvas Node for client projection; Memory recommendation "Hibernatable WebSockets".

#### Acceptance Criteria

1. THE re-plan surface SHALL render every Cascade's Changed_Leg, Affected_Set, per-Leg prior and new offer amount, net amount, and outcome, without horizontal scrolling at a viewport width of 320 CSS pixels.
2. THE re-plan surface SHALL render its Leg list using semantic HTML list and description elements, with one list item per Leg and a non-empty accessible name containing the Leg identifier.
3. THE re-plan surface SHALL present every interactive control with a touch target of at least 44 by 44 CSS pixels.
4. THE re-plan surface SHALL read from its local replica first and SHALL treat the edge as a convergence peer, requiring zero network round trip to render the last known Cascade state.
5. WHILE the surface is offline, THE re-plan surface SHALL retain and render the last projected Cascade state, SHALL present a not-current indicator carrying elapsed time since last synchronization, and SHALL discard zero previously projected Cascade.
6. WHEN connectivity returns, THE re-plan surface SHALL converge with the edge replica and SHALL remove the not-current indicator, losing zero locally recorded observation.
7. THE re-plan surface SHALL render a rolled-back Cascade as rolled back with its recorded reason, and SHALL render a rolled-back Cascade as committed zero times.
8. THE re-plan surface SHALL keep media and icon wrappers visible to selection tooling, retaining an accessible name at the owning semantic element rather than marking selectable visual structure as decorative.

## Non-Goals

Stated so that a task cannot quietly adopt one:

1. Cross-principal or group bundles, and any pooled or shared envelope — the Split-Pay roadmap item's data model, deliberately not built early.
2. Bundles beyond the declared Scale_Boundary of 20 Legs and 20 Edges.
3. Any dedicated graph database, graph query language, or graph-native engine, at any layer (ADR-4).
4. Speculative or predictive re-optimization ahead of a Mutation_Event.
5. Re-optimization triggered by anything other than a Shared Canvas Node Mutation_Event.
6. Multi-currency envelope netting.
7. Dispute or chargeback-triggered hold adjustment.
8. Recursive re-optimization within one Cascade.
9. Any modification to a reused component's interface.
10. Any Oracle-hosted inference path.
11. Any Prod mirror publication or Cloudflare deployment.
12. Any claim that Workers AI or Containers overflow is zero-cost.

## Bridge Coverage Statement

14 requirements derive from 5 PRD user stories across 2 features, 4 ADRs, 3 Performance Enhancement tables, 2 Deploy Boundary Register rows, and the platform's stated operating priorities. Three PRD Open Questions are carried as explicit Stated Gaps (Requirements 1.10, 5.10, 6.9) rather than closed with invented values; each becomes a blocked task in `tasks.md` pending one operator decision.
