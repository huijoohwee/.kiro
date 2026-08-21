---
title: "Knowgrph Agentic Travel Commerce Platform — Requirements"
doc_type: "Spec Requirements"
schema: "kiro-spec-requirements/v1"
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
source_specification: "joohwee/prd-tad-ard/knowgrph-agentic-travel-commerce-platform-prd-tad-adr.md v1.4.0"
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

This document derives executable requirements for the consolidation increment specified in `knowgrph-agentic-travel-commerce-platform-prd-tad-adr.md` v1.4.0. That specification adds exactly three things neither prior document had: the L4 **Bundle Re-Optimization Engine**, the **Budget Envelope Ledger** primitive underneath it, and a dedicated **cache / computation / memory** performance strategy — plus an infrastructure ADR retiring Oracle Always Free ARM from the critical path.

Scope discipline: every requirement below traces to a named PRD user story and its VCC translation, an ADR, a row of the Journey → System Mapping or Topology table, a Performance Enhancement recommendation, a post-audit production contract, or a Deploy Boundary Register row. Version 1.4.0 reconciles the bounded storage/media/publication/retention contracts, Workers AI Free overflow policy, clean-install demo, and bootstrap receipt v2 without granting deployment authority.

Reuse discipline: every previously published Shared Canvas Node Store, Agent Registry/Router, Discovery Harness, Issuance Service, Settlement Verifier, Notification Dispatcher, Marketplace Registry Canvas, and Guardrail Gate operation remains compatible. Production wiring is additive and isolated behind travel-specific service contracts: Shared Canvas accepted-event outbox delivery, Router travel dispatch, provider-backed net settlement, Discovery adapters, and reconciliation operator ingress. Guardrail Gate gains **one new data dependency** (Envelope Ledger) and **no change to its inherited operation contract**. Inherited components keep whatever readiness rung they already earned; nothing here re-claims a rung on their behalf.

Deploy boundary: every requirement is satisfied entirely within the Dev lane (`GitHub/knowgrph`, `npm run dev:apex`, `npm run dev`). The Prod mirror (`GitHub/huijoohwee/content/knowgrph`) and the Cloudflare routes (`airvio.co`, `airvio.co/knowgrph`) are gated deploy targets and are never acceptance criteria for any requirement in this document.

**Implementation status (2026-08-20):** `dev-proven`. The protected source frontier contains merged travel integration PR #814; clean-install demo, upstream-deadline corrections, and nested-lock supply-chain closure are review-ready in PR #818. The source includes the production service-binding topology, atomic Shared Canvas trigger/outbox and cold initialization, provider-backed settlement executor, strict Search→Verify identity, durable reconciliation custody, keyset-paged sync/export/crawlers, streamed documents, signed workspace-bound media with R2 ownership metadata, exact-revision publication ACLs, and a versioned terminal-Hold receipt-compaction contract. It is not yet a Production-deployment candidate because protected live provider configuration, bootstrap receipt v2, provider UAT, deployed-state, live effect/latency, capacity telemetry, and rollback receipts remain absent. `delivered_rung` therefore remains `undocumented`.

### Post-Audit Production Contracts

1. Storage page cursors SHALL be opaque, workspace/snapshot bound, monotonically advancing keysets; clients SHALL apply each bounded page and persist the sync cursor only after the terminal page.
2. Authenticated document delivery SHALL stream bounded database segments; crawler delivery SHALL use deterministic bounded pages rather than materializing an entire workspace.
3. Production raw media SHALL require an unforgeable short-lived capability binding subject, workspace, object, operation, issue/expiry, and nonce; R2 objects SHALL carry matching workspace ownership metadata.
4. Anonymous document/crawler delivery SHALL require a non-revoked publication whose workspace, document, canonical path, revision, and content hash exactly match the current record.
5. Released terminal Holds MAY compact into versioned minimal replay receipts only when exact replay/conflict semantics remain indefinite; hot balance/revision/overlap paths SHALL remain O(1) or indexed, and Production promotion SHALL include capacity/cold-start telemetry.

## Glossary

- **Bundle**: One multi-leg itinerary owned by exactly one principal, identified by `bundle_id`.
- **Leg**: One committed or candidate unit of a Bundle (a flight, a hotel night, a transfer, a local experience), identified by `leg_id`, holding exactly one current `offer_id` when committed.
- **Edge**: A directed causal dependency from an upstream Leg to a downstream Leg, stored as a row in the flat `edges` table. Direction means "changing the source may invalidate the target".
- **Bundle_Graph_Store**: The new Durable Object holding `legs`/`edges` for one `bundle_id`, exposing bounded downstream BFS plus atomic commit and rollback. PRD Topology node "Bundle Graph Store".
- **Affected_Set**: The set of Legs reachable by following outgoing Edges from a Changed_Leg, excluding the Changed_Leg itself. The subject of PRD US-1's VCC.
- **Changed_Leg**: The single Leg named by a Mutation_Event as having changed.
- **Mutation_Event**: The internal travel-service envelope `{bundle_id, leg_id, event_id}` derived from one accepted Shared Canvas Node mutation. The caller supplies the inherited node mutation fields only: the immutable operator map resolves `bundle_id`, the accepted node carries exactly one `leg_id`, and its inherited `transactionId` becomes `event_id`. The only trigger for re-optimization in this increment.
- **Reopt_Worker**: The new Worker orchestrating one Cascade: BFS, parallel re-quote, envelope check, durable prepare, one idempotent net-settlement operation, atomic finalize, and archive write. PRD Topology node "Re-optimization Worker".
- **Cascade**: One complete Reopt_Worker execution for one Mutation_Event.
- **Re_Quote**: One Discovery dispatch, issued through the reused Agent Registry/Router, requesting a replacement offer for one Leg in the Affected_Set.
- **Committed_Snapshot**: The persisted `legs`/`edges` state of one Bundle at its last successful commit; the sole rollback target.
- **Envelope_Ledger**: The new Durable Object holding `envelope`/`holds` for one `principal_id`, exposing atomic check-and-reserve. PRD Topology node "Envelope Ledger".
- **Hold**: One record in the `holds` table in semantic state `reserved`, `quarantined`, `committed`, or `released`, carrying an amount and an `offer_id`; quarantined is reserved custody excluded from TTL release.
- **Available_Balance**: `total_budget` minus the sum of all `reserved`, `quarantined`, and `committed` Hold amounts for that principal at that instant.
- **Guardrail_Gate**: The reused deterministic component evaluating one offer against the principal's budget, now reading Available_Balance from Envelope_Ledger. Interface unchanged.
- **Issuance_Service**: The inherited provider-effect boundary reached through the new service-only net-settlement executor; its existing public operations remain compatible, and a journal alone is never proof of effect.
- **Net_Settlement_Call**: The single Issuance_Service call carrying the net signed delta of one Cascade.
- **Balance_Cache**: The Cloudflare KV read-through cache of Available_Balance. Never a source of truth.
- **Offer_Cache**: The Cloudflare Cache API cache of Discovery Harness offers, short TTL with stale-while-revalidate.
- **Provenance_Archive**: The R2 write-once store of committed Bundle snapshots and provenance receipts.
- **Scale_Boundary**: The declared per-Bundle limit of 20 Legs and 20 Edges, stated openly per ADR-4 rather than silently assumed.
- **Cost_Log**: The recorded per-Cascade cost entry naming prompt tokens, completion tokens, and dollar cost.
- **Inference_Router**: The component selecting the direct Cloudflare Workers AI binding (primary) or the separately authenticated Workers AI overflow Worker for a probe-tree L1 inference call, per ADR-3.
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

#### V1 Operator Decision

10. THE Reopt_Worker SHALL trigger exactly one bounded Cascade for every validated, unique Mutation_Event and SHALL apply no materiality threshold. THE Reopt_Worker SHALL use the event identity only for idempotent duplicate suppression and SHALL NOT treat duplicate suppression as a materiality rule.

### Requirement 2: Atomic All-Or-Nothing Affected-Set Commit

**User Story:** As a Shopper-Agent Principal, I want the entire affected set re-quoted, approved, and settled as one atomic operation, so that I never end up in a half-changed state.

Traces to: PRD Bundle Re-Optimization US-2 and its VCC; Success Metrics "Partial-commit incidents = 0" and "Rollback correctness = 100%"; Bundle_Commit_Deploy_Boundary.

#### Acceptance Criteria

1. WHEN every Leg in the Affected_Set has a Re_Quote result that passed Guardrail_Gate, THE Bundle_Graph_Store SHALL durably prepare the complete quote set while leaving the prior projection visible; after zero-net handling or a definitive successful settlement, it SHALL commit every new `offer_id` in one atomic storage transaction.
2. IF any single Leg in the Affected_Set has a Re_Quote result that Guardrail_Gate rejected, is absent, or is malformed, or IF settlement returns a definitive non-retryable pre-effect rejection, THEN THE Bundle_Graph_Store SHALL commit zero new `offer_id` values for that Cascade and SHALL retain or restore the Committed_Snapshot. A retryable or ambiguous settlement result SHALL remain pending under Requirement 3.7 instead of being mislabeled as rollback.
3. THE Bundle_Graph_Store SHALL expose zero externally observable states in which some Legs of one Affected_Set carry a new `offer_id` while other Legs of the same Affected_Set carry a stale `offer_id`.
4. WHEN a Cascade rolls back, THE Bundle_Graph_Store SHALL restore every Leg and Edge row to a state byte-identical to the Committed_Snapshot that preceded that Cascade.
5. WHEN a Cascade rolls back, THE Reopt_Worker SHALL transition every Hold it reserved for that Cascade to `released`, and SHALL leave zero Holds for that Cascade in state `reserved`.
6. THE Reopt_Worker SHALL treat a repeated Mutation_Event carrying the same `bundle_id`, Changed_Leg identifier, and event identifier as idempotent, producing at most one commit and at most one logical Net_Settlement identity across all repeats; recovery transport retries SHALL reuse that identity.
7. WHEN a commit succeeds, THE Reopt_Worker SHALL write the resulting Bundle snapshot and its provenance receipt to Provenance_Archive exactly once for that Cascade.
8. IF the Provenance_Archive write fails after a successful commit, THEN THE Reopt_Worker SHALL retain the commit, SHALL record a typed archive-deferred entry naming the Cascade, and SHALL NOT roll back the committed Legs.
9. THE Bundle_Commit_Deploy_Boundary SHALL report `closed` at the start and at the end of every Cascade, and THE Reopt_Worker SHALL commit only through the transactional path named in criterion 1.
10. THE Bundle_Graph_Store SHALL permit at most one non-terminal Cascade for one `bundle_id`; while one Cascade is active, a distinct Mutation_Event SHALL return a typed pending `bundle-busy` outcome without starting another Cascade.
11. WHILE a Cascade is non-terminal, THE Bundle_Graph_Store SHALL reject every Leg or Edge insertion with typed reason `bundle-busy`. Therefore the Edge set and all unaffected Legs remain byte-identical for the Cascade lifetime, and rollback SHALL restore the captured affected Legs while observing the same Edge rows that preceded the Cascade.
12. THE Bundle_Graph_Store SHALL reject insertion of a new Leg that already carries a committed offer/amount with typed reason `committed-leg-insertion-unsupported`; committed positions SHALL enter only through bundle initialization or the atomic Cascade finalize path.

### Requirement 3: One Logical Net Settlement Per Cascade

**User Story:** As a Shopper-Agent Principal, I want one net settlement call for the whole cascade instead of one per changed leg, so that re-planning a 4-leg cascade costs one card top-up or refund action, not four.

Traces to: PRD Bundle Re-Optimization US-3 and its VCC; MoSCoW "Must — one idempotent net-settlement operation per cascade".

#### Acceptance Criteria

1. WHEN a Cascade with a non-zero net amount commits on its first definitive Issuance_Service response, THE Reopt_Worker SHALL issue exactly 1 transport call for that Cascade, regardless of the Affected_Set size; recovery retries SHALL reuse the same Cascade idempotency key and identical settlement identity, so they remain one logical Net_Settlement operation.
2. THE Reopt_Worker SHALL compute the Net_Settlement_Call amount as the signed sum of every committed Leg's new offer amount minus that Leg's prior committed offer amount, across the whole Affected_Set.
3. WHEN the computed net amount is zero, THE Reopt_Worker SHALL issue zero Issuance_Service settlement calls and SHALL record a zero-net entry naming the Cascade.
4. IF a Cascade rolls back before settlement is attempted, THEN THE Reopt_Worker SHALL issue zero Issuance_Service settlement calls; a definitive non-retryable settlement rejection MAY produce a rolled-back Cascade after exactly one recorded attempt.
5. THE Reopt_Worker SHALL be the only component creating a logical settlement operation arising from a Cascade, and SHALL call the isolated net-settlement boundary, which delegates provider effect through the inherited Issuance_Service capability without changing any pre-existing public operation.
6. THE Reopt_Worker SHALL record, per Cascade, the Affected_Set size, durable settlement-attempt count, and one stable idempotency key, so the success-path ratio and any recovery retries are retrievable from surfaced output rather than inferred.
7. IF Issuance_Service returns a definitive, non-retryable pre-effect rejection, THEN THE Reopt_Worker SHALL roll back per Requirement 2.4, SHALL release the Cascade's Holds per Requirement 2.5, and SHALL record a typed settlement-failed reason. IF the result is retryable or ambiguous — including timeout, abort, network failure, HTTP 408/429/5xx, an untyped conflict, or a malformed success body — THEN THE Reopt_Worker SHALL leave the prior graph projection visible, SHALL protect the reserved Holds, SHALL persist a pending recovery disposition, and SHALL retry only with the same idempotency key and settlement request identity. A typed idempotency-conflict HTTP 409 proves semantic identity disagreement and SHALL enter reconciliation rather than loop. IF later recovery cannot safely prove that an ambiguous attempt had no effect, THEN THE Reopt_Worker SHALL require operator reconciliation and SHALL NOT roll back.
8. THE Reopt_Worker SHALL hold zero configured StraitsX or Avalanche client of its own, and every payment-path request arising from a Cascade SHALL carry the Issuance_Service component identifier at the gateway boundary.
9. THE Issuance_Service SHALL treat a journal record as zero proof of financial effect and SHALL return settlement success only after a provider-backed receipt exactly matches the signed amount, currency, Cascade, Bundle, Principal, effect direction, settlement identifier, and provider reference.
10. WHEN an attempted settlement cannot be safely finalized, THE Envelope_Ledger SHALL quarantine its Holds without TTL expiry, SHALL keep their value unavailable, and SHALL release or commit them only through a separately authenticated, audited, idempotent operator decision that converges both ledger and graph state.
11. BEFORE the Reopt_Worker durably records or dispatches a possible external settlement effect, THE Envelope_Ledger SHALL durably mark that Cascade's reserved Holds as non-expiring possible-effect custody. Expiry alarms SHALL exclude those Holds; a crash, delay, or alarm before recovery SHALL NOT return their value. A definitive no-effect path MAY explicitly release them, a definitive success MAY commit them, and an ambiguous path SHALL atomically convert them to the quarantine state in criterion 10.

### Requirement 4: Atomic Check-And-Reserve Against True Available Balance

**User Story:** As a Shopper-Agent Principal, I want every offer from any registered agent to atomically check and reserve against my true remaining budget, accounting for every concurrent hold, so that two simultaneous agent offers can never jointly overspend my limit.

Traces to: PRD Budget Envelope Ledger US-1 and its VCC; Success Metrics "Over-envelope commits = 0"; ADR-1.

#### Acceptance Criteria

1. WHEN Guardrail_Gate evaluates an offer, THE Envelope_Ledger SHALL compute Available_Balance as `total_budget` minus the sum of all Hold amounts in state `reserved`, `quarantined`, or `committed` for that `principal_id` at that instant.
2. THE Envelope_Ledger SHALL perform the balance check and the Hold reservation as one indivisible operation, such that no other reservation for the same `principal_id` can interleave between them.
3. THE Envelope_Ledger SHALL accept an offer only when its amount is less than or equal to the Available_Balance computed in the same indivisible operation, and SHALL reject it otherwise.
4. THE Envelope_Ledger SHALL serialize every mutation for one `principal_id` through exactly one Durable Object instance keyed on that `principal_id`.
5. WHEN two or more offers for the same `principal_id` are submitted concurrently and their amounts sum above Available_Balance, THE Envelope_Ledger SHALL accept only the subset whose cumulative amount remains within Available_Balance, and SHALL reject the remainder with a typed insufficient-envelope reason.
6. THE Envelope_Ledger SHALL contain zero committed Hold sets whose summed amount exceeds `total_budget` for that `principal_id`, at any observable instant.
7. THE Envelope_Ledger SHALL complete a check-and-reserve within 10 milliseconds measured inside the Durable Object with no D1 hop.
8. IF the Envelope_Ledger cannot be reached or its state cannot be read, THEN THE Guardrail_Gate SHALL reject the offer with a typed envelope-unavailable reason, SHALL reserve zero Holds, and SHALL make the offer available to zero downstream components.
9. THE Guardrail_Gate SHALL retain its existing operation names, parameters, and return values unchanged, adding Envelope_Ledger as a data dependency only.
10. THE Agent Registry/Router SHALL make a registered-agent offer available to an Edge_Orchestrator only after the same-lane Guardrail_Gate service has returned success and the Envelope_Ledger has atomically created the corresponding ordinary-offer Hold; the unguarded route operation SHALL remain an internal Reopt_Worker Re_Quote contract and SHALL NOT be an ordinary-offer return path.
11. THE Envelope_Ledger SHALL serialize ordinary-offer Holds and Cascade Holds through the same Durable Object keyed by `principal_id`; mixed concurrent reservations SHALL share one Available_Balance and SHALL preserve criteria 3, 5, and 6.
12. THE Envelope_Ledger SHALL accept only non-negative safe-integer minor-unit budgets and offer amounts. Every offer SHALL carry one uppercase three-letter currency equal to the envelope settlement currency; a mismatch SHALL reject with `quote-currency-mismatch` or `envelope-currency-conflict` and reserve zero value.
13. IN Production_Lane, THE Envelope_Ledger SHALL accept only offers whose price verification is `verified`; `deterministic-demo` SHALL be accepted only outside Production_Lane, and any other or disallowed proof SHALL reject with `quote-unverified` and reserve zero value.
14. THE ordinary-offer reservation identity SHALL be stable and idempotent: an exact repeat while the Hold remains `reserved` or `committed` SHALL return the existing Hold; an exact reservation replay after release SHALL reject with `offer-reservation-released` and SHALL NOT resurrect value; and a repeat of the same operation identity with different agent, offer, amount, currency, or verification SHALL reject with a typed idempotency-conflict reason and mutate nothing.

### Requirement 5: Hold Lifecycle And Immediate Release Visibility

**User Story:** As a Shopper-Agent Principal, I want a released or rolled-back leg's held funds to become available to other in-flight agents immediately, so that a rejected re-plan does not leave my budget artificially locked.

Traces to: PRD Budget Envelope Ledger US-2 and its VCC; MoSCoW "Must — hold lifecycle"; Should "KV-cached available_balance read".

#### Acceptance Criteria

1. THE Envelope_Ledger SHALL transition a normal Hold only along `reserved → committed` or `reserved → released`; after an attempted ambiguous settlement it MAY transition `reserved → quarantined`, and only an authenticated operator reconciliation decision MAY transition `quarantined → committed` or `quarantined → released`; it SHALL reject every other transition with a typed illegal-transition reason.
2. THE Envelope_Ledger SHALL treat a repeated transition request for a Hold already in the requested terminal state as a no-op returning the current state, mutating nothing.
3. WHEN a Hold transitions to `released`, THE Envelope_Ledger SHALL exclude its amount from the next computed Available_Balance for that `principal_id`, with no externally observable staleness window beyond the Durable Object's own write-then-read consistency.
4. THE Envelope_Ledger SHALL maintain the invariant `total_budget = Available_Balance + sum(reserved) + sum(quarantined) + sum(committed)` after every transition.
5. THE Guardrail_Gate SHALL read Available_Balance from Balance_Cache on its fast path, and SHALL fall back to the Envelope_Ledger on a cache miss.
6. THE Guardrail_Gate SHALL treat Balance_Cache as non-authoritative: WHEN a Balance_Cache value and an Envelope_Ledger value diverge for the same `principal_id`, THE Guardrail_Gate SHALL use the Envelope_Ledger value and SHALL invalidate the cached entry.
7. THE Envelope_Ledger SHALL invalidate the Balance_Cache entry for a `principal_id` on every Hold transition for that principal, before returning the transition result.
8. THE Envelope_Ledger SHALL reserve, commit, and release Holds only through the serialized path named in Requirement 4.2, and SHALL expose zero direct `holds` table write operation outside it, satisfying the Envelope_Mutation_Deploy_Boundary operator instruction.
9. THE Envelope_Mutation_Deploy_Boundary SHALL report `closed` at the start and at the end of this increment.

#### V1 Operator Decision

10. THE Envelope_Ledger SHALL expose Available_Balance to server-side callers only, SHALL expose zero client-facing read route or auth scope for it, and SHALL require the Shopper Client to consume only the bounded re-plan projection rather than the ledger value.
11. THE same-lane Guardrail_Gate service SHALL expose ordinary-offer `commit` and `release` operations that bind the transition to the original `principal_id`, operation identity, and registered `agent_id`; an exact terminal repeat SHALL be idempotent, and a mismatched or illegal transition SHALL mutate nothing.

### Requirement 6: Bounded Cascade Orchestration And Fail-Closed Circuit Breaker

**User Story:** As a Platform Operator, I want one cascade to be a single bounded pass with a stated breaker, so that a re-optimization can never recurse, retry unboundedly, or partially retry its way into a mixed state.

Traces to: PRD Orchestration/Harness Flows "Bundle Re-Optimization Pipeline"; Computation recommendation "Re-dispatch via `Promise.all`"; PRD fail-closed posture inherited from the travel document.

#### Acceptance Criteria

1. THE Reopt_Worker SHALL execute exactly 1 pass per Mutation_Event, and SHALL trigger zero further Cascades from within that pass, including from Legs its own commit changed.
2. THE Reopt_Worker SHALL dispatch every Re_Quote for one Affected_Set concurrently and SHALL wait for all of them before evaluating any of them, so that Cascade wall-clock time is bounded by the slowest single Re_Quote rather than their sum.
3. WHEN any Re_Quote in a Cascade is rejected, THE Reopt_Worker SHALL abort the whole Affected_Set, SHALL roll back per Requirement 2.4, and SHALL issue zero partial retries of individual Legs.
4. THE Reopt_Worker SHALL bound each Cascade by a stated wall-clock cap. IF the cap expires before any settlement attempt, it SHALL abort with a typed cascade-timeout reason and roll back per Requirement 2.4. IF the cap expires after settlement may have been attempted, it SHALL preserve the prior visible projection and protected Holds, then remain pending or quarantine for reconciliation under Requirement 3.7/3.10 rather than claim rollback.
5. THE Reopt_Worker SHALL resolve every ambiguous or unrepresentable Re_Quote result as a rejection, and SHALL treat zero ambiguous result as a pass.
6. THE Reopt_Worker SHALL issue every Re_Quote through the reused Agent Registry/Router interface, and SHALL hold zero per-vertical harness identifier in source.
7. THE Reopt_Worker SHALL record, per Cascade, the Re_Quote count, the reject count, the abort reason where present, and the elapsed wall-clock time.
8. IF Bundle_Graph_Store is unreachable at the start of a Cascade, THEN THE Reopt_Worker SHALL return a typed store-unavailable rejection, SHALL issue zero Re_Quote calls, and SHALL mutate nothing.

#### V1 Operator Decision

9. WHEN a Cascade rolls back, THE Reopt_Worker SHALL return the typed rollback outcome to the caller, SHALL record the same outcome in the Session_Log, and SHALL make that record available through the bundle's hibernatable WebSocket projection. THE Reopt_Worker SHALL emit zero separate Notification Dispatcher message and SHALL create zero synchronous out-of-band notification dependency.

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
8. THE Dev_Lane SHALL introduce zero storage category beyond Durable Object SQLite storage, KV, Cache API, R2, and the existing D1 and Yjs categories; Production resource provisioning remains a separate receipt.

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
10. THE Agent Registry/Router SHALL cache Agent Definition lookups in Worker memory and its bound Agent Definition KV namespace, and SHALL invalidate those entries only on agent registration or de-registration; Reopt_Worker SHALL consume the Router result and SHALL own zero registry cache.
11. THE Offer_Cache SHALL scope in-flight refresh coalescing to one runtime cache instance. Two independent request/runtime instances SHALL dispatch independently even when their Re_Quote keys are byte-identical, while calls sharing one instance MAY coalesce the identical refresh.

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

Traces to: ADR-3 (Accepted) including its explicit FOSS-hard-gate flag; ADR-3 Consequences.

#### Acceptance Criteria

1. THE Inference_Router SHALL route probe-tree L1 inference calls to Cloudflare Workers AI as its primary path, and SHALL issue zero call to an Oracle-hosted inference endpoint.
2. THE Inference_Router SHALL route to a separately authenticated Cloudflare Worker using a remote Workers AI binding only as the bounded overflow path; that Worker SHALL declare zero Container binding and SHALL accept only the same permitted provider-model identity.
3. THE Inference_Router SHALL restrict primary-path model selection to the Permitted_Model_Set, defined as catalog entries whose weights are licensed Apache-2.0 or MIT.
4. IF a requested model's declared license is neither Apache-2.0 nor MIT, THEN THE Inference_Router SHALL reject the primary-path route with a typed license-excluded reason naming the model identifier and its declared license, and SHALL issue zero primary-path call for that model.
5. THE Inference_Router SHALL read model license declarations from externalized configuration, and SHALL hold zero model identifier or license string hardcoded in source.
6. THE Inference_Router SHALL record, per inference call, the selected path, the model identifier, the declared license, and the recorded cost, so that license compliance is auditable from surfaced output.
7. IF the license configuration cannot be read, THEN THE Inference_Router SHALL reject every route with a typed license-configuration-unavailable reason, and SHALL fall back to zero unfiltered model.
8. THE Dev_Lane SHALL enforce the explicit 10,000-daily-neuron Workers AI Free policy across primary and overflow paths, fail closed when the policy is unavailable or exhausted, and make no zero-cost claim outside that bounded allocation.
9. THE Dev_Lane SHALL hold zero Oracle endpoint, credential key name, or SSH configuration for an inference path.

### Requirement 12: Reused-Component Interface Preservation

**User Story:** As a Solo Founder, I want the reused components consumed at their existing interfaces, so that this increment inherits their earned rungs instead of invalidating them.

Traces to: PRD "Reused components (unchanged interfaces, extended data dependency only for Guardrail Gate)"; PRD Component Inventory rung inheritance.

#### Acceptance Criteria

1. THE Dev_Lane SHALL preserve every previously published Shared Canvas Node Store, Agent Registry/Router, Discovery Harness, Issuance_Service, Settlement Verifier, Notification Dispatcher, and Marketplace Registry Canvas operation, parameter, and return value unchanged; any travel-specific operation SHALL live on a separately named adapter/route contract and SHALL NOT shadow or mutate an inherited operation.
2. THE Guardrail_Gate SHALL gain exactly one new data dependency, the Envelope_Ledger, and SHALL change zero element of its own interface.
3. THE Dev_Lane SHALL detect a change to any reused interface as a typed reused-interface-changed finding naming the interface and the changed element.
4. THE Shared Canvas integration SHALL consume the accepted node mutation at its inherited shape and SHALL require zero new caller field: it SHALL resolve `bundle_id` from the immutable operator map, copy the single travel `leg_id`, derive `event_id` from the inherited accepted-node `transactionId`, and send only that internal envelope to Reopt_Worker.
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
12. Any claim that Workers AI is unconditionally zero-cost or available beyond the explicit free-policy ceiling.

## Bridge Coverage Statement

14 requirements derive from 5 PRD user stories across 2 features, 4 ADRs, 3 Performance Enhancement tables, 2 Deploy Boundary Register rows, and the platform's stated operating priorities. Requirements 1.10, 5.10, and 6.9 carry the three recorded v1 Operator decisions. They are no longer blocked gaps and are covered by the corresponding local checks and Task 18 decision records.

## Local Evidence References

Every entry below is an authoring-surface record, not delivery evidence. Commands run from `$GITHUB_ROOT/knowgrph`; each `check:*` emits schema `knowgrph-travel-commerce-check-evidence/v1`. Exact metrics and replay seeds are recorded in `tasks.md` under **Authoring-Surface Evidence Ledger**.

A requirement-level named check is a routing anchor, not a claim that every later acceptance criterion appears in that check's emitted requirement array. The emitted array is authoritative for the named record. Supplemental executable evidence and authoring-document-only obligations are stated explicitly below so a top-level mapping cannot silently over-claim them.

| Requirement | Base named Evidence Reference | Supplemental criterion evidence or explicit boundary |
|---|---|---|
| 1 | `npm run check:affected-set` | — |
| 2 | `npm run check:atomic-commit` | The named command emits a focused 2.10–2.12 record proving one active Cascade, the Leg/Edge structural fence, byte-identical rollback, and rejection of pre-committed insertion. `check:scale-boundary` continues to emit Requirement 7 criteria only. |
| 3 | `npm run check:net-settlement` | Criterion 3.9: `npm run travel-commerce:services:test`, `npm run travel-commerce:settlement-executor:test`, plus the provider-receipt regression in `core-recovery-regressions.test.ts`. Criterion 3.10: focused `reconciliation-custody.test.ts` under `npm run travel-commerce:test`. Neither criterion is attributed to the base record. |
| 4 | `npm run check:envelope-atomicity` | The named command emits its 4.1–4.7 mixed-channel ledger record and a 4.10–4.14 authenticated public-ingress → registry → same-lane Guardrail → actual reservation record; focused `ordinary-offer-atomicity.test.ts` and `npm run mcp:worker:test` remain supplemental lifecycle/topology regressions. |
| 5 | `npm run check:hold-lifecycle` | Ordinary-offer terminal operations in criterion 5.11 use the same focused evidence as Requirement 4.10–4.14. |
| 6 | `npm run check:cascade-bounds` | — |
| 7 | `npm run check:scale-boundary` | — |
| 8 | `npm run check:storage-placement` | — |
| 9 | `npm run check:edge-cache` | The emitted requirement array includes 9.11 and the record reports two independent request-instance dispatches, proving in-flight refresh state is not shared across runtime instances. |
| 10 | `npm run check:cost-observability` | — |
| 11 | `npm run check:inference-license` | — |
| 12 | `npm run check:reused-interfaces` | The check snapshots inherited interfaces and their inherited inventory rows. Criterion 12.5's inventory for this increment is an authoring-document review across the PRD Component Inventory and the design/task owner maps; no executable named-record claim is made for that current-increment inventory. |
| 13 | `npm run check:deploy-boundary` | — |
| 14 | `npm run check:replan-surface` plus `npm run travel-commerce:demo:browser` | — |

The locally validated source includes deployable Worker topology, exact fail-closed dependency probes, clean-install demo entrypoints, passing default/staging/production dry-run packaging, and `0`-finding audits for both the root and independently committed Canvas dependency graphs. Protected live configuration, authorized `knowgrph-travel-mesh-bootstrap-receipt/v2`, provider UAT, human Production authorization, Deployment, State Reconciliation, Live Verification, Publication, and Rollback receipts remain absent; therefore the aggregate local rung is `dev-proven` and `delivered_rung` is `undocumented`.
