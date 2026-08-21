---
title: "Knowgrph Agentic Travel Commerce Platform — Runtime and Delivery Tasks"
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
responsibility: "Tasks 11–20: caches, inference, UI/runtime wiring, checkpoints, and Production lifecycle obligations"
---

# Runtime and Delivery Tasks

## Tasks 11–20

- [x] 11. Cache layers — after the hot path, not before (PRD Next Step 3)

  - [x] 11.1 Implement `balance-cache.ts` — non-authoritative by construction
    - Store only Available_Balance keyed `available-balance:{principalId}`; expose the read only through the Guardrail Gate adapter
    - Make `confirmAvailableBalance` call the authoritative Envelope Ledger on every decision; invalidate divergence and write the current authoritative revision back
    - Expose invalidation used on Hold mutation before the mutation returns
    - _Requirements: 5.5, 5.6, 9.1, 9.2_
    - _Properties: CP-11_
    - _Check: `npm run check:edge-cache`_
    - _Scope: `src/cache/balance-cache.ts`_
    - _Capability: local write_
    - _Bounds: 25k tokens · 3 iterations · 30 min · 35% context · breaker: default_

  - [x]* 11.2 Write property test for cache non-authority
    - **Property CP-11: A divergent Balance_Cache value never changes a commit decision; the ledger value wins and the entry is invalidated (Metamorphic)**
    - Generator: `arbCacheDivergence` — agreeing and diverging cache/ledger pairs, including a stale cache that would wrongly permit an over-envelope commit
    - `numRuns: 200`
    - Assert the accept/reject outcome is identical whether the cache agrees, diverges high, diverges low, or misses entirely
    - **Validates: Requirements 5.6, 9.2**
    - _Check: `npm run check:edge-cache`_
    - _Scope: `tests/props/cp-11-cache-non-authority.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 25k tokens · 3 iterations · 30 min · 35% context · breaker: default_

  - [x] 11.3 Implement `offer-cache.ts` — TTL, stale-while-revalidate, full-request keying
    - Cache Discovery Harness offers with a TTL between 30 and 60 seconds inclusive
    - Serve a stale entry only while a revalidation is in flight
    - Key entries on the full Re_Quote request identity, so any request-field difference resolves to a different entry
    - Carry each entry's fetch timestamp and TTL so a caller can evaluate staleness rather than assume freshness
    - Scope the in-flight refresh map to one `OfferCache` runtime instance; hold zero module-global refresh promise so independent requests/isolates cannot share transient caller state
    - _Requirements: 9.3, 9.5, 9.11_
    - _Properties: CP-12_
    - _Check: `npm run check:edge-cache`; the emitted requirement array includes 9.11 and the record reports independent request-instance dispatches_
    - _Scope: `src/cache/offer-cache.ts`_
    - _Capability: local write_
    - _Bounds: 30k tokens · 3 iterations · 40 min · 40% context · breaker: default_

  - [x]* 11.4 Write property test for commit-path offer freshness
    - **Property CP-12: No commit occurs against an Offer_Cache entry past TTL whose revalidation has not completed (Invariant)**
    - Generator: `arbOfferCacheAges` — ages spanning under-TTL, at-TTL, past-TTL, with and without completed revalidation
    - `numRuns: 200`
    - Assert the commit-path `requote` operation never returns a soft-stale entry: it awaits the in-flight/new refresh, while only the separate advisory operation may serve stale-within-hard-TTL data
    - **Validates: Requirements 9.3, 9.4**
    - _Check: `npm run check:edge-cache`_
    - _Scope: `tests/props/cp-12-stale-offer-refusal.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 25k tokens · 3 iterations · 30 min · 35% context · breaker: default_

  - [x] 11.5 Implement `provenance-archive.ts` — R2 write-once
    - Write each committed snapshot and provenance receipt under `provenance/{encodedBundleId}/{encodedCascadeId}.json`
    - Treat an identical same-digest replay as idempotent and reject a conflicting overwrite with `archive-immutable`
    - On write failure after a successful commit: retain the commit and record a typed `archive-deferred` entry naming the Cascade; do not roll back
    - Retain zero archived-only snapshot in Durable Object or D1 storage
    - _Requirements: 2.7, 2.8, 9.7, 9.8_
    - _Properties: CP-16_
    - _Check: `npm run check:edge-cache`_
    - _Scope: `src/archive/provenance-archive.ts`_
    - _Capability: local write_
    - _Bounds: 30k tokens · 3 iterations · 40 min · 40% context · breaker: default_

  - [x]* 11.6 Write property test for archive write-once idempotence
    - **Property CP-16: An identical same-key write is idempotent, a differing digest is rejected, and archive state after N identical writes equals state after 1 (Idempotence)**
    - Generator: `arbArchiveWrites` — repeated writes to identical and distinct keys, interleaved
    - `numRuns: 200`
    - Assert rejection carries `archive-immutable` and that no existing object's bytes change
    - **Validates: Requirements 2.7, 9.7**
    - _Check: `npm run check:edge-cache`_
    - _Scope: `tests/props/cp-16-archive-write-once.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 20k tokens · 3 iterations · 25 min · 35% context · breaker: default_

  - [x]* 11.7 Integration — cache effectiveness, registry invalidation, archive-deferred fault
    - Record Discovery dispatch counts with and without Offer_Cache for repeated identical Re_Quotes inside the TTL window; surface both numbers
    - Assert Agent Registry/Router owns Agent Definition caching in Worker memory and its bound KV namespace, invalidates only on registration or de-registration, and Reopt Worker owns zero registry cache
    - Assert two independent `OfferCache` instances dispatch byte-identical refreshes independently, while identical calls sharing one instance may coalesce
    - Record that the independent-instance assertion executes inside `check:edge-cache`; emit 9.11 and report independent request-instance dispatches
    - Fault-inject an archive failure after a successful commit; assert the commit is retained and `archive-deferred` is recorded
    - Assert zero archived-only snapshot remains in DO or D1 storage after archival
    - _Requirements: 2.8, 9.6, 9.8, 9.9, 9.10, 9.11_
    - _Check: `npm run check:edge-cache`, `npm run check:atomic-commit`_
    - _Scope: `cloudflare/workers/knowgrph-travel-commerce/test/evidence/edge-cache.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 30k tokens · 3 iterations · 40 min · 40% context · breaker: default_

- [x] 12. Storage placement and cost observability

  - [x] 12.1 Implement `storage-placement-guard.ts`
    - Wrap the D1 client; reject a call carrying a hot-path caller tag with a typed `storage-placement` reason naming the calling component
    - Permit only cross-key aggregate and reporting queries
    - Reject introduction of a storage system outside the declared set: DO SQLite, KV, Cache API, R2, D1, Yjs store
    - Rely on the DO's single-threaded per-key execution for hot-path atomicity; introduce zero external lock, lease, or advisory-locking mechanism
    - _Requirements: 8.3, 8.4, 8.8_
    - _Properties: none (enforced by scan)_
    - _Check: `npm run check:storage-placement`_
    - _Scope: `src/runtime/storage-placement-guard.ts`_
    - _Capability: local write_
    - _Bounds: 25k tokens · 3 iterations · 30 min · 35% context · breaker: default_

  - [x] 12.2 Implement `cost-log.ts`
    - Emit exactly one Cost_Log entry per Cascade recording prompt tokens, completion tokens, and dollar cost attributable to the Cascade's own orchestration
    - Record `0`, `0`, `0.00` for a Cascade whose Re_Quotes returned from cache or a deterministic harness path
    - Attribute Discovery Harness token cost to the harness that incurred it; attribute zero harness cost to the Worker
    - Emit the entry for rejected and rolled-back Cascades too, so a failed Cascade's cost is not invisible
    - _Requirements: 10.2, 10.3, 10.7_
    - _Properties: CP-14_
    - _Check: `npm run check:cost-observability`_
    - _Scope: `src/runtime/cost-log.ts`_
    - _Capability: local write_
    - _Bounds: 25k tokens · 3 iterations · 30 min · 35% context · breaker: default_

  - [x]* 12.3 Write property test for cost-log emission and zero-model discipline
    - **Property CP-14: Every Cascade emits exactly one Cost_Log entry, and orchestration token counts are zero (Invariant)**
    - Generators: `arbBundle` × `arbRequoteResults` covering committed, rolled-back, rejected, and no-op outcomes
    - `numRuns: 200`
    - Assert one entry per Cascade across all four outcome kinds, with orchestration prompt and completion counts of zero
    - **Validates: Requirements 10.2, 10.7**
    - _Check: `npm run check:cost-observability`_
    - _Scope: `tests/props/cp-14-cost-log.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 25k tokens · 3 iterations · 30 min · 35% context · breaker: default_

  - [x]* 12.4 Static scans and hibernation integration
    - Scan the module graphs reachable from `reopt-worker.ts`, `bundle-graph-store.ts`, and `envelope-ledger.ts`; assert no model client is reachable, naming the importing module on failure
    - Assert zero hot-path D1 query is issued during a Cascade or a check-and-reserve, and that the guard rejects one with the caller named
    - Assert the adjacency list is built once per DO wake and zero times per Mutation_Event within that wake
    - Assert both DOs restore correct state after a simulated hibernation cycle with zero caller-side cold-rebuild logic
    - Assert Shared Canvas Node client push uses hibernatable WebSockets
    - Assert determinism: identical inputs produce identical affected set, commit decision, net amount, and envelope outcome
    - _Requirements: 8.1, 8.2, 8.3, 8.5, 8.6, 8.7, 8.8, 10.1, 10.4, 10.5, 10.6_
    - _Check: `npm run check:storage-placement`, `npm run check:cost-observability`_
    - _Scope: `tests/scans/model-free.test.ts`, `tests/integration/storage-placement.test.ts`_
    - _Capability: read, local write, local execute_
    - _Bounds: 30k tokens · 3 iterations · 40 min · 40% context · breaker: default_

- [x] 13. Oracle retirement — independent workstream (ADR-3, PRD Next Step 4)

  - [x] 13.1 Implement `model-license-filter.ts` — fail-closed Permitted_Model_Set
    - Read model license declarations from externalized configuration; hold zero model identifier or license string in source
    - Derive the Permitted_Model_Set as entries licensed Apache-2.0 or MIT
    - Reject a model whose declared license is neither, with `license-excluded` naming the model identifier and its declared license
    - On unreadable configuration, permit zero model with `license-configuration-unavailable`; do not degrade to allow-all
    - _Requirements: 11.3, 11.4, 11.5, 11.7_
    - _Properties: none (enforced by scan and integration)_
    - _Check: `npm run check:inference-license`_
    - _Scope: `src/runtime/model-license-filter.ts`_
    - _Capability: local write_
    - _Bounds: 25k tokens · 3 iterations · 30 min · 35% context · breaker: default_

  - [x] 13.2 Implement `inference-router.ts` — direct Workers AI primary, Workers AI Worker overflow
    - Route probe-tree L1 inference to Cloudflare Workers AI as primary, restricted to the Permitted_Model_Set
    - Route overflow only through the token-authenticated Worker using a remote Workers AI binding; allow only the same permitted provider-model identity and declare zero Container binding
    - Issue zero call to an Oracle-hosted inference endpoint
    - Record per call: selected path, model identifier, declared license, recorded cost
    - Enforce and record the explicit 10,000-daily-neuron Workers AI Free policy across both paths; make no unlimited zero-cost claim
    - _Requirements: 11.1, 11.2, 11.6, 11.8_
    - _Properties: none (enforced by scan and integration)_
    - _Check: `npm run check:inference-license`_
    - _Scope: `src/runtime/inference-router.ts`_
    - _Capability: local write_
    - _Bounds: 30k tokens · 3 iterations · 40 min · 40% context · breaker: default_

  - [x]* 13.3 Static scan and integration for license filtering and Oracle absence
    - Scan the repository and assert zero Oracle endpoint, credential key name, or SSH configuration exists on an inference path
    - Assert a non-Apache-2.0/MIT model is rejected with the model and license named
    - Assert unreadable license configuration permits zero model
    - Assert path, model, license, and cost are recorded per call, and neither path is recorded as free
    - _Requirements: 11.1, 11.4, 11.6, 11.7, 11.8, 11.9_
    - _Check: `npm run check:inference-license`_
    - _Scope: `tests/scans/no-oracle-path.test.ts`, `tests/integration/inference-license.test.ts`_
    - _Capability: read, local write, local execute_
    - _Bounds: 25k tokens · 3 iterations · 30 min · 35% context · breaker: default_

- [x] 14. Deploy boundary discipline

  - [x] 14.1 Implement `deploy-boundary.ts`
    - Derive each of the three boundary states from a recorded Evidence Reference; report `closed` when the reference is absent
    - Reject a Prod mirror write or Cloudflare delivery-route mutation before the request is issued, recording the requesting component identifier and target boundary
    - State the rollback statement per row: restore the last Committed_Snapshot for bundle commit; transition the hold to `released` for envelope mutation
    - Require a recorded exact-candidate human authorization before the mirror-to-delivery boundary may report anything other than `closed`; infer, default, schedule, or simulate that authorization zero times
    - _Requirements: 2.9, 5.9, 13.1, 13.2, 13.3, 13.4, 13.5, 13.6_
    - _Properties: none (enforced by process assertion)_
    - _Check: `npm run check:deploy-boundary`_
    - _Scope: `src/runtime/deploy-boundary.ts`_
    - _Capability: local write_
    - _Bounds: 30k tokens · 3 iterations · 40 min · 40% context · breaker: default_

  - [x]* 14.2 Process assertion and no-hardcode scan
    - Assert all three boundaries report `closed` before and after the full check sweep
    - Assert the Prod mirror path is byte-identical before and after, and zero Cloudflare route request was issued
    - Scan source, fixtures, tests, and generated assets for a developer-specific absolute path, credential value, account identifier, provider catalog, or environment-specific default; assert zero occurrence
    - Assert every authored file is ≤600 lines with one declared responsibility
    - _Requirements: 13.1, 13.3, 13.4, 13.7_
    - _Check: `npm run check:deploy-boundary`_
    - _Scope: `tests/scans/no-hardcode.test.ts`, `tests/integration/deploy-boundary.test.ts`_
    - _Capability: read, local write, local execute_
    - _Bounds: 25k tokens · 3 iterations · 30 min · 35% context · breaker: default_

- [x] 15. Re-plan surface — mobile-first, local-first

  - [x] 15.1 Implement `replan-surface.ts` projection and render
    - Project Cascade outcomes through the reused Shared Canvas Node Store; require zero new field from the mutation event beyond `leg_id`
    - Render changed leg, affected set, per-leg prior and new amount, net amount, and outcome using semantic list and description elements, one list item per leg, each with a non-empty accessible name containing the leg identifier
    - Render without horizontal scrolling at 320 CSS px; present every interactive control at ≥44×44 CSS px
    - Render a rolled-back Cascade as rolled back with its recorded reason; render it as committed zero times
    - Keep media and icon wrappers visible to selection tooling with an accessible name at the owning semantic element
    - _Requirements: 12.4, 14.1, 14.2, 14.3, 14.7, 14.8_
    - _Properties: none (enforced by browser assertion)_
    - _Check: `npm run check:replan-surface`_
    - _Scope: `src/ui/replan-surface.ts`_
    - _Capability: local write_
    - _Bounds: 35k tokens · 4 iterations · 45 min · 45% context · breaker: default_

  - [x] 15.2 Implement local-first read and offline retention in `replan-surface.ts`
    - Read the local replica first; treat the edge as a convergence peer, requiring zero network round trip to render the last known Cascade state
    - While offline: retain and render the last projected state, present a not-current indicator carrying elapsed time since last synchronization, discard zero previously projected Cascade
    - On reconnect: converge with the edge replica, remove the indicator, lose zero locally recorded observation
    - _Requirements: 14.4, 14.5, 14.6_
    - _Properties: none (enforced by browser assertion)_
    - _Check: `npm run check:replan-surface`_
    - _Scope: `src/ui/replan-surface.ts`_
    - _Capability: local write_
    - _Bounds: 30k tokens · 3 iterations · 40 min · 40% context · breaker: default_

  - [x]* 15.3 Browser and accessibility assertions
    - Assert no horizontal overflow at 320 CSS px and ≥44×44 px touch targets
    - Assert semantic list and description elements with a non-empty accessible name per leg row
    - Assert offline retention, elapsed-since-sync indicator, and post-reconnect convergence with zero lost observation
    - Assert a rolled-back Cascade never renders as committed
    - Assert media and icon wrappers retain an accessible name and are not marked decorative
    - _Requirements: 14.1, 14.2, 14.3, 14.5, 14.6, 14.7, 14.8_
    - _Check: `npm run check:replan-surface`_
    - _Scope: `tests/browser/replan-surface.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 30k tokens · 3 iterations · 40 min · 40% context · breaker: default_

- [x] 16. End-to-end wiring — the 2-leg MVP bundle the PRD names

  - [x] 16.1 Implement `wiring.ts` and the minimum-viable fixture
    - Wire the deterministic local fixture through Shared Canvas-shaped input → Re-opt Worker → Bundle Graph Store → dispatch → Agent Registry/Router double → Guardrail Gate → Envelope Ledger → Issuance Service double → Provenance Archive → re-plan surface
    - Seed the min-viable scope from the source specification: one flight leg plus one downstream local-experience leg whose start time depends on arrival, one edge between them, one principal, one envelope
    - Trigger one upstream change producing one downstream re-plan and one net settlement call
    - Keep the production Shared Canvas Node → travel-commerce Worker live receipt as a protected runtime prerequisite; source wiring now atomically persists accepted node + outbox, derives `event_id` from `transactionId`, resolves an immutable operator map, cold-seeds the bundle/envelope, and retries the service-only mutation contract
    - Introduce zero external vendor integration beyond those already spec'd
    - _Requirements: 3.1, 6.6, 12.1, 12.4_
    - _Properties: end-to-end arm for CP-1, CP-3, CP-5_
    - _Check: `npm run travel-commerce:test`_
    - _Scope: `src/bundle/wiring.ts`, `tests/fixtures/two-leg-bundle.ts`_
    - _Capability: local write_
    - _Bounds: 35k tokens · 4 iterations · 45 min · 45% context · breaker: default_

  - [x]* 16.2 End-to-end test over the 2-leg bundle
    - Assert the full chain: upstream change → affected set of exactly one downstream leg → one Re_Quote → gate pass with a reserved hold → durable prepare → exactly one first-path settlement call → atomic finalize → one archive write → committed projection on the surface
    - Assert the rollback path end to end: a rejected downstream Re_Quote leaves both legs on their prior offers, zero holds reserved, zero settlement calls, and a rolled-back projection
    - Assert the no-op path: a change to the downstream leg, which has no outgoing edges, produces zero Re_Quote and zero settlement call
    - _Requirements: 1.5, 2.2, 2.5, 3.1, 3.4_
    - _Check: `npm run travel-commerce:test`_
    - _Scope: `tests/e2e/two-leg-cascade.test.ts`_
    - _Capability: local write, local execute_
    - _Bounds: 35k tokens · 4 iterations · 45 min · 50% context · breaker: default_

- [x] 17. Named invocable checks

  - [x] 17.1 Wire the fourteen `check:*` scripts
    - Add one base script per requirement, exactly as the design's Base Named Invocable Check table names them: `check:affected-set`, `check:atomic-commit`, `check:net-settlement`, `check:envelope-atomicity`, `check:hold-lifecycle`, `check:cascade-bounds`, `check:scale-boundary`, `check:storage-placement`, `check:edge-cache`, `check:cost-observability`, `check:inference-license`, `check:reused-interfaces`, `check:deploy-boundary`, `check:replan-surface`; keep supplemental criterion commands explicit rather than pretending each base record emits every additive criterion
    - Each exits non-zero on failure and prints its recorded counts, measured values, and property run seeds, so a failure is reproducible from printed output alone
    - Each names the requirement identifiers it evaluates, so a check's coverage is readable without reading its source
    - _Requirements: 1.6, 3.6, 4.7, 6.7, 9.6, 10.2, 11.6_
    - _Check: `npm run check:affected-set && npm run check:atomic-commit && npm run check:net-settlement && npm run check:envelope-atomicity && npm run check:hold-lifecycle && npm run check:cascade-bounds && npm run check:scale-boundary && npm run check:storage-placement && npm run check:edge-cache && npm run check:cost-observability && npm run check:inference-license && npm run check:reused-interfaces && npm run check:deploy-boundary && npm run check:replan-surface`_
    - _Scope: `package.json` (`scripts` only), `scripts/checks/*`_
    - _Capability: local write, local execute_
    - _Bounds: 35k tokens · 3 iterations · 45 min · 40% context · breaker: default_

- [x] 18. V1 operator decisions — recorded and testable

  - [x]* 18.1 RESOLVED — every validated unique mutation triggers (Requirement 1.10)
    - **State: `verified`. Decision: no materiality threshold in v1.**
    - Every validated, unique Mutation_Event triggers one bounded Cascade
    - `(bundleId, legId, eventId)` idempotence suppresses exact duplicates only; duplicate suppression is not a materiality rule
    - _Requirements: 1.10_
    - _Check: `npm run check:affected-set`, `npm run check:atomic-commit`_
    - _Scope: specification decision plus existing `src/bundle/reopt-worker.ts` event path_
    - _Capability: read, local execute_
    - _Bounds: decision recorded; no new dispatch required_

  - [x]* 18.2 RESOLVED — rollback outcome uses caller + Session Log/WebSocket only (Requirement 6.9)
    - **State: `verified`. Decision: no separate Notification Dispatcher message in v1.**
    - The caller receives the typed rollback outcome; the same outcome is persisted in the Session Log and available on the bundle's hibernatable WebSocket projection
    - No synchronous out-of-band notification dependency is added
    - _Requirements: 6.9_
    - _Check: `npm run check:cascade-bounds`, `npm run check:replan-surface`, `npm run travel-commerce:test`_
    - _Scope: specification decision plus existing Bundle Graph observability path_
    - _Capability: read, local execute_
    - _Bounds: decision recorded; no new dispatch required_

  - [x]* 18.3 RESOLVED — Available_Balance is server-side only (Requirement 5.10)
    - **State: `verified`. Decision: zero Shopper Client balance-read route or auth scope in v1.**
    - The Guardrail Gate adapter consumes the authoritative Envelope Ledger RPC; the browser receives only the bounded re-plan projection
    - _Requirements: 5.10_
    - _Check: `npm run check:envelope-atomicity`, `npm run check:reused-interfaces`, `npm run travel-commerce:test`_
    - _Scope: specification decision plus existing server-only DO RPC surface_
    - _Capability: read, local execute_
    - _Bounds: decision recorded; no new dispatch required_

- [x] 19. Final checkpoint — full sweep
  - Ensure all tests pass; ask the user if questions arise
  - Run every named check; confirm all fourteen exit zero and print recorded counts, measurements, and seeds
  - Confirm scoped Tasks 1–19 are terminal, Task 20 source contracts and remaining live obligations are recorded exactly, and the three v1 Operator decisions remain unchanged
  - Confirm all three Deploy Boundary rows still read `closed` and `delivered_rung` remains `undocumented`
  - Compare per-run token consumption against the source specification's stated $0.00 orchestration budget and record the comparison
  - _Requirements: 10.2, 12.6, 13.1_
  - _Check: `npm run check:agentic-travel-commerce-platform`_
  - _Scope: none (read only)_
  - _Capability: read, local execute_
  - _Bounds: 15k tokens · 2 iterations · 25 min · 30% context · breaker: default_

- [ ] 20. Production functionality and long-run lifecycle prerequisites — source functionality closed; live delivery and monitoring open

  - [x] 20.1 Define bounded pagination/streaming contracts for storage reads above the prior safe caps
    - Pull/export now use restartable, workspace/snapshot-bound opaque keyset cursors; the Canvas client applies every page immediately and advances the durable cursor only after the final page
    - Authenticated documents stream bounded `substr`/chunk segments; crawler pages use stable opaque cursors and deterministic ordering without full-workspace materialization
    - Storage 42/42 covers multi-page sync/export, >1 MiB documents, >100 chunks, and >100 crawler rows; direct Canvas checks cover cursor advancement, convergence, and incremental response cancellation

  - [x] 20.2 Define and implement signed workspace-bound media capabilities and R2 ownership metadata
    - HMAC-SHA256 capabilities bind subject, workspace, object key, read/write operation, issue/expiry, and nonce; maximum TTL is 15 minutes
    - R2 writes stamp workspace/owner/schema metadata; reads and media-asset persistence verify that metadata, and the server replaces any caller-supplied asset URL with its own signed read capability
    - Production readiness requires `KNOWGRPH_STORAGE_SIGNING_SECRET`; missing/short configuration fails closed and the protected release inventories the secret

  - [x] 20.3 Define and implement publication authorization for anonymous document/crawler delivery
    - `document_publications` pins workspace, document, canonical path, revision, and content hash; publish/revoke requires an authenticated writer role
    - Anonymous document and crawler reads succeed only while the current record exactly matches the published revision/hash; later edits or revocation fail closed
    - Focused storage evidence covers private denial, publish, anonymous document/crawler visibility, automatic revision invalidation, and revoke

  - [ ] 20.4 Deploy and live-probe the authorized storage security candidate
    - The currently served runtime is not credited with the local authorization, bounds, or alias-disable changes
    - Close only with the exact-candidate Deployment, State Reconciliation, Public/Browser Live Verification, Publication, and exercised Rollback receipts from the protected ledger

  - [ ] 20.5 Operate the versioned terminal-Hold retention/compaction and cold-start capacity contract
    - Source contract `knowgrph-envelope-hold-retention/v1` compacts released Hold payloads into indexed ordinary/cascade replay receipts while preserving indefinite exact replay/conflict semantics and imposing no lifetime operation cap
    - Availability, conservation, revision, and overlap are O(1)/indexed; `getHolds` observes active Holds only; full legacy validation occurs once per schema migration before the version marker is committed
    - Local source/type/high-cardinality evidence is complete. Close this Production obligation only when deployed telemetry proves Durable Object storage growth, migration duration, alarm health, and cold-start headroom on the exact candidate

Task 20 records inherited and long-run Production prerequisites discovered by the runtime audit. It is deliberately outside the original authoring wave graph and does not retroactively widen Requirements 1–14; it prevents local travel evidence from being misread as end-to-end Production readiness.
