---
title: "Knowgrph Agentic Travel Commerce Platform — Verification and Delivery Design"
doc_type: "Spec Design Shard"
schema: "kiro-spec-design/v1"
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
parent_design: ".kiro/specs/knowgrph-agentic-travel-commerce-platform/design.md v1.4.0"
responsibility: "verification strategy, economics, deploy boundaries, and operator decisions"
---

# Verification and Delivery Design

## Testing Strategy

Property tests use `fast-check` `3.23.2` (MIT), dev-only and pinned exact. Every payment-path test runs against local service doubles; zero test issues a provider, payment, or Production mutation. The table below describes the fully evidenced local generators; protected integration and live-runtime receipts remain separately open in the task ledger.

### Generators

| Generator | Produces | Used by |
|---|---|---|
| `arbBundle` | acyclic `legs`/`edges` within the scale boundary, single principal, 1–20 legs | CP-1, CP-2, CP-3, CP-4, CP-10, CP-15 |
| `arbBundleWithCycle` | edge sets containing at least one cycle | CP-13 |
| `arbOversizeBundle` | insertion sequences crossing 20 legs or 20 edges | CP-13 |
| `arbMutationSequence` | mutation events including repeats with identical event keys | CP-9 |
| `arbRequoteResults` | per-leg pass / reject / missing / malformed / timeout outcomes | CP-3, CP-5, CP-12 |
| `arbConcurrentOffers` | 2–16 offers for one principal with interleaved arrival orders | CP-6, CP-7, CP-8 |
| `arbHoldTransitions` | legal and illegal transition sequences, including repeats at terminal states | CP-7, CP-8 |
| `arbCacheDivergence` | Balance_Cache / ledger value pairs, agreeing and diverging | CP-11 |
| `arbOfferCacheAges` | entry ages spanning under-TTL, at-TTL, past-TTL, with and without completed revalidation | CP-12 |
| `arbEdgeInsertionOrders` | permutations of one edge set's insertion order | CP-15 |
| `arbArchiveWrites` | repeated writes to identical and distinct keys | CP-16 |

### Run Counts

| Property | numRuns | Why |
|---|---|---|
| CP-6 | 600 | hard invariant: over-envelope commits must be 0; concurrency is where it breaks |
| CP-3 | 500 | hard invariant: partial-commit incidents must be 0 |
| CP-5 | 400 | the literal L4 claim |
| CP-13 | 400 | error-condition totality across four distinct rejection causes |
| CP-1, CP-7/CP-8 combined arm, CP-15 | 300 each | downstream precision, lifecycle, and non-empty insertion confluence |
| CP-2 | 200 | internal single-visit observer across generated reachable DAG walks |
| CP-4, CP-9, CP-11, CP-12 | 200 each | |
| CP-8, CP-10, CP-14, CP-16 | 200 each | |

Shrinking matters most for CP-3, CP-6, and CP-15, where the useful failure report is the minimal interleaving or insertion order that breaks the invariant, not the first random one found.

### Integration And Example Checks

| Criterion | Check | Kind |
|---|---|---|
| Req 1.6 — walk under 50 ms for ≤8 legs, in-DO | `check:affected-set` | measurement, recorded |
| Req 1.9 — one Session_Log entry per Cascade including no-ops | `check:affected-set` | integration |
| Req 2.7, 2.8 — archive exactly once; commit retained on archive failure | `check:atomic-commit` | integration, fault-injected |
| Req 2.10–2.12 — one active Cascade, Leg/Edge structural fence, byte-identical rollback, and pre-committed insertion rejection | `check:atomic-commit` focused structural-fence record | executable assertions and emitted criterion record; not attributed to `check:scale-boundary` |
| Req 3.2 — net amount equals signed sum across the affected set | `check:net-settlement` | example, tabulated |
| Req 3.6 — affected-set size and settlement-call count both recorded | `check:net-settlement` | integration |
| Req 3.8 — every payment request carries the Issuance_Service identifier | `check:net-settlement` | static scan + boundary assertion |
| Req 3.9 — success requires an exact provider-backed receipt, never a journal row alone | `travel-commerce:services:test`, `travel-commerce:settlement-executor:test`, plus focused `core-recovery-regressions.test.ts` | provider contract + fault-injected mismatch assertions; not attributed to the base net-settlement record |
| Req 3.10 — ambiguous effect enters non-expiring authenticated reconciliation custody | focused `reconciliation-custody.test.ts` under `travel-commerce:test` | >24h alarm + authenticated convergence assertions; not attributed to `check:hold-lifecycle` |
| Req 3.11 — possible-effect custody precedes attempt recording/dispatch and survives alarm-before-recovery | focused `ordinary-offer-atomicity.test.ts` plus `core-recovery-regressions.test.ts` under `travel-commerce:test` | forced expiry/alarm ordering, unavailable balance, then atomic quarantine |
| Req 4.7 — check-and-reserve under 10 ms, no D1 hop | `check:envelope-atomicity` | measurement, recorded |
| Req 4.10–4.14 — registered ordinary offers cannot bypass reservation; mixed ordinary/Cascade traffic is atomic, currency/verification fenced, and idempotent | `check:envelope-atomicity`, focused `ordinary-offer-atomicity.test.ts`, plus `mcp:worker:test` | named public-ingress → registry → Guardrail → actual reservation evidence plus lifecycle and same-lane topology regressions |
| Req 5.3 — no observable staleness window beyond DO write-then-read | `check:hold-lifecycle` | integration |
| Req 5.7 — cache invalidated before the transition returns | `check:hold-lifecycle` | integration, ordering assertion |
| Req 6.2 — cascade wall-clock equals slowest leg, not the sum | `check:cascade-bounds` | measurement, recorded |
| Req 6.7 — re-quote count, reject count, abort reason, elapsed recorded | `check:cascade-bounds` | integration |
| Req 7.1, 7.2 — zero graph engine reachable; zero graph traversal call | `check:scale-boundary` | static scan |
| Req 7.8 — declared limits readable as named constants | `check:scale-boundary` | static scan |
| Req 8.1–8.3 — zero hot-path D1 query; guard rejects with caller named | `check:storage-placement` | static scan + integration |
| Req 8.5–8.7 — adjacency built once per wake; state survives hibernation; hibernatable WebSockets in use | `check:storage-placement` | integration |
| Req 8.8 — zero new storage system beyond the declared set | `check:storage-placement` | static scan |
| Req 9.3, 9.5 — TTL within 30–60 s; full-request keying | `check:edge-cache` | integration |
| Req 9.6 — dispatch counts with and without cache recorded | `check:edge-cache` | measurement, recorded |
| Req 9.8 — zero archived-only snapshot retained in DO or D1 | `check:edge-cache` | integration |
| Req 9.10 — registry lookups invalidated only on (de)registration | `check:edge-cache` | integration |
| Req 9.11 — in-flight refresh state is runtime-instance scoped | `check:edge-cache` independent-instance assertion | executable assertion and emitted criterion record |
| Req 10.6 — no model client reachable from the three new modules | `check:cost-observability` | static scan |
| Req 11.1, 11.9 — zero Oracle endpoint, credential key name, or SSH config | `check:inference-license` | static scan |
| Req 11.6 — path, model, license, cost recorded per call | `check:inference-license` | integration |
| Req 11.8 — neither path recorded as free | `check:inference-license` | integration |
| Req 12.1–12.4, 12.6 — reused interface snapshot unchanged; inherited rungs not re-claimed | `check:reused-interfaces` | snapshot + inherited-document assertion |
| Req 12.5 — every current-increment component has local/delivered ownership | PRD Component Inventory plus this owner table and `tasks.md` owner map | authoring-document review; no executable named-record claim |
| Req 13.1–13.4 — three boundaries `closed`; mirror and route untouched | `check:deploy-boundary` | process assertion |
| Req 13.7 — zero developer path, credential, or account identifier in source | `check:deploy-boundary` | static scan |
| Req 14.1–14.3 — 320 px no horizontal scroll; semantic list; 44×44 px targets | `check:replan-surface` | browser assertion |
| Req 14.5, 14.6 — offline retention with elapsed indicator; convergence on reconnect | `check:replan-surface` | browser assertion |
| Req 14.7 — rolled-back Cascade never renders as committed | `check:replan-surface` | browser assertion |
| Req 14.8 — media and icon wrappers keep an accessible name | `check:replan-surface` | accessibility assertion |

### Base Named Invocable Check Per Requirement

These fourteen commands are stable routing anchors for the original requirement groups. They are not blanket evidence for every later additive criterion. The supplemental rows above and each emitted `requirements` array define the exact executable coverage boundary.

| Requirement | Named check |
|---|---|
| 1 — Downstream-Only Affected-Set Precision | `npm run check:affected-set` |
| 2 — Atomic All-Or-Nothing Commit | `npm run check:atomic-commit` |
| 3 — One Logical Net Settlement Per Cascade | `npm run check:net-settlement` |
| 4 — Atomic Check-And-Reserve | `npm run check:envelope-atomicity` |
| 5 — Hold Lifecycle And Release Visibility | `npm run check:hold-lifecycle` |
| 6 — Bounded Cascade Orchestration | `npm run check:cascade-bounds` |
| 7 — Flat Adjacency And Scale Boundary | `npm run check:scale-boundary` |
| 8 — Hot-Path Storage Placement | `npm run check:storage-placement` |
| 9 — Three-Layer Edge Cache Correctness | `npm run check:edge-cache` |
| 10 — Model-Free Determinism And Cost | `npm run check:cost-observability` |
| 11 — Inference Consolidation And License Filter | `npm run check:inference-license` |
| 12 — Reused-Interface Preservation | `npm run check:reused-interfaces` |
| 13 — Dev-Only Deploy Boundary | `npm run check:deploy-boundary` |
| 14 — Mobile-First Local-First Re-Plan Surface | `npm run check:replan-surface` |

## Token Economics And Cost

| Path | Prompt | Completion | Cost per call |
|---|---|---|---|
| Affected_Set BFS | 0 | 0 | $0.00 |
| Atomic commit / rollback | 0 | 0 | $0.00 |
| Net-amount computation | 0 | 0 | $0.00 |
| Envelope check-and-reserve | 0 | 0 | $0.00 |
| Hold lifecycle transitions | 0 | 0 | $0.00 |
| Topological order maintenance | 0 | 0 | $0.00 |
| Re_Quote via Discovery Harness | unchanged from travel doc | unchanged | unchanged, and reduced in frequency by Offer_Cache |

Marginal infrastructure uses no new storage category: Durable Objects, KV, Cache API, R2, and reporting-only D1 are Cloudflare-native. Their Production IDs, namespaces, buckets, migrations, bindings, and routes are provisioning prerequisites rather than assumed existing resources. Free-tier availability is a cost hypothesis, not a readiness receipt. Primary and overflow inference share the explicit Workers AI Free 10,000-daily-neuron policy and fail closed outside it (Requirement 11.8).

One-line version: **the cascade itself is arithmetic, so the L4 claim costs storage reads, not tokens.**

## Deploy Boundary

| Boundary | From | To | Evidence Reference | Rollback | State |
|---|---|---|---|---|---|
| Bundle commit: affected-set → committed | Authoring | Mirror | local `check:atomic-commit` / `check:net-settlement`; no promotion receipt | restore the last Committed_Snapshot | `closed` |
| Envelope mutation: offer → hold | Authoring | Mirror | local `check:envelope-atomicity` / `check:hold-lifecycle`; no promotion receipt | transition the hold to `released` | `closed` |
| Prod mirror → Cloudflare delivery | Mirror | Delivery | none — no authorized candidate | not applicable; nothing published | `closed` |

All three read `closed`. Local correctness Evidence References do not grant promotion; the boundary-opening Integration, Deployment, State Reconciliation, Live Verification, Publication, and Rollback receipts remain absent.

## Implementation Evidence Reference

The exact commands, results, metrics, and property seeds are recorded once in `tasks.md` under **Authoring-Surface Evidence Ledger**. The requirement checks emit `knowgrph-travel-commerce-check-evidence/v1`; the deterministic demo and its browser verifier emit `knowgrph-travel-commerce-demo-evidence/v1` and `knowgrph-travel-commerce-browser-evidence/v1`. The browser record passed 8/8 executable beats at 320 CSS px with 0 px horizontal overflow, a 58 px minimum touch target, executable evidence retained offline, zero lost observations, zero provider requests, zero real payment calls, and zero production mutations.

This evidence is sufficient for `dev-proven` and a locally validated authoring candidate. It is intentionally insufficient for a Production-deployment candidate or `runtime-ready`: the storage functionality contracts are source-complete but unverified live, terminal-receipt capacity telemetry is absent, and no protected live configuration/bootstrap-v2/provider-UAT/deployed-state/effect/rollback receipt chain exists.

## Design Decisions And Rationale

- **Two tables, not a graph engine.** ADR-4, made executable: `check:scale-boundary` statically asserts zero graph engine is reachable. At single-digit-to-20-leg bundles, BFS over an indexed `edges` table is not a compromise — a graph engine would be a second data-access paradigm bought for nothing.
- **Structure read and authorization read are separate calls.** `affectedSet` answers "what could be affected"; `isPresent` answers "may this leg be dispatched now". Fusing them would let a warm adjacency snapshot authorize a removed leg.
- **Commit spans the affected set, not the bundle.** Legs outside the Affected_Set are untouched by construction, which is what makes Requirement 1.2's "no unaffected sibling is touched" a property rather than a promise.
- **Balance_Cache is non-authoritative at the adapter boundary.** `confirmAvailableBalance` always joins the cache read to an authoritative ledger RPC before returning a value; divergence invalidates and refreshes KV. No commit path consumes the KV helper directly.
- **Integer minor units everywhere.** A ledger whose stated invariant is zero over-envelope commits should not carry float representation error.
- **Archive failure does not roll back a commit.** Provenance is bookkeeping; the commit is the principal's booking. The asymmetry is deliberate and recorded (Requirement 2.8).
- **License filtering fails closed.** ADR-3's FOSS-hard-gate flag becomes `model-license-filter.ts`. Unreadable config permits zero model, because a filter that opens under error is decoration.
- **One pass per event, no recursion.** A Cascade cannot trigger a Cascade. This bounds the whole feature to something testable and keeps a delay storm from becoming an unbounded re-planning loop.

## V1 Operator Decisions

1. **Trigger sensitivity** (Requirement 1.10) — every validated, unique Mutation_Event triggers one bounded Cascade. No materiality threshold is configured. `(bundleId, legId, eventId)` idempotence returns the recorded outcome for a duplicate and is the only suppression rule.
2. **Rollback notification path** (Requirement 6.9) — the caller receives the typed rollback outcome; the same outcome is persisted in the Session Log and broadcast on the bundle's hibernatable WebSocket projection. V1 sends no separate Notification Dispatcher message and adds no out-of-band synchronous notification dependency.
3. **Client-facing Available_Balance** (Requirement 5.10) — server-side only. The Worker HTTP surface exposes no balance route, and no Shopper Client auth scope is introduced.

The deterministic tie-break for topological order remains ascending `legId`. Any total order works; it is stated so that CP-15's "identical to full recompute" is a checkable claim. The property evidence now generates non-empty DAG edge sets and proves forward/reverse insertion orders converge with full recomputation.
