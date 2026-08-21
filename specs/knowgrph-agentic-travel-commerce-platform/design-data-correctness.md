---
title: "Knowgrph Agentic Travel Commerce Platform — Data and Correctness Design"
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
responsibility: "data models, correctness properties, and error disposition"
---

# Data and Correctness Design

## Data Models

### Durable Object SQLite — Bundle Graph Store

`src/bundle/bundle-graph-schema.ts` is the executable and authoritative schema/migration source. The review copy below mirrors its current create-table contract; if prose or this copy ever differs from the executable migration, the source file wins and the documentation must be reconciled before a readiness claim.

```sql
CREATE TABLE IF NOT EXISTS _sql_schema_migrations (
  id INTEGER PRIMARY KEY, applied_at INTEGER NOT NULL
);
CREATE TABLE IF NOT EXISTS bundle_meta (
  bundle_id TEXT PRIMARY KEY, principal_id TEXT NOT NULL,
  total_budget_minor INTEGER NOT NULL CHECK (total_budget_minor >= 0),
  initialization_state TEXT NOT NULL DEFAULT 'ready'
    CHECK (initialization_state IN ('pending', 'ready')),
  seed_fingerprint TEXT NOT NULL DEFAULT ''
);
CREATE TABLE IF NOT EXISTS legs (
  leg_id TEXT PRIMARY KEY, principal_id TEXT NOT NULL, category TEXT NOT NULL,
  committed_offer_id TEXT,
  committed_amount_minor INTEGER CHECK (committed_amount_minor >= 0),
  last_cascade_id TEXT
);
CREATE TABLE IF NOT EXISTS edges (
  from_leg_id TEXT NOT NULL, to_leg_id TEXT NOT NULL,
  PRIMARY KEY (from_leg_id, to_leg_id)
);
CREATE TABLE IF NOT EXISTS topology (
  position INTEGER PRIMARY KEY, leg_id TEXT NOT NULL UNIQUE
);
CREATE TABLE IF NOT EXISTS cascades (
  cascade_id TEXT PRIMARY KEY, event_id TEXT NOT NULL, bundle_id TEXT NOT NULL,
  principal_id TEXT NOT NULL, changed_leg_id TEXT NOT NULL,
  phase TEXT NOT NULL CHECK (phase IN (
    'quoting', 'settlement_pending', 'settling', 'finalizing', 'archiving',
    'committed', 'archive_failed', 'reconciliation_required',
    'rolled_back', 'no_op', 'rejected'
  )),
  affected_json TEXT NOT NULL, prior_legs_json TEXT NOT NULL,
  changes_json TEXT NOT NULL, net_amount_minor INTEGER NOT NULL,
  outcome_json TEXT, started_at INTEGER NOT NULL, updated_at INTEGER NOT NULL,
  recovery_attempts INTEGER NOT NULL DEFAULT 0,
  settlement_attempts INTEGER NOT NULL DEFAULT 0,
  next_recovery_at INTEGER, archive_snapshot_json TEXT
);
CREATE TABLE IF NOT EXISTS settlement_claims (
  cascade_id TEXT PRIMARY KEY, owner TEXT NOT NULL, expires_at INTEGER NOT NULL
);
CREATE TABLE IF NOT EXISTS reconciliation_decisions (
  cascade_id TEXT PRIMARY KEY, decision_id TEXT NOT NULL UNIQUE,
  decision TEXT NOT NULL CHECK (decision IN ('commit', 'release')),
  operator_id TEXT NOT NULL, reason TEXT NOT NULL,
  requested_at INTEGER NOT NULL, completed_at INTEGER
);
CREATE TABLE IF NOT EXISTS session_log (
  seq INTEGER PRIMARY KEY AUTOINCREMENT,
  cascade_id TEXT NOT NULL, event_type TEXT NOT NULL,
  bundle_id TEXT NOT NULL, changed_leg_id TEXT NOT NULL, affected_json TEXT NOT NULL,
  outcome TEXT, reason TEXT, recorded_at INTEGER NOT NULL
);
CREATE TABLE IF NOT EXISTS cost_log (
  seq INTEGER PRIMARY KEY AUTOINCREMENT,
  cascade_id TEXT NOT NULL, component TEXT NOT NULL,
  prompt_tokens INTEGER NOT NULL, completion_tokens INTEGER NOT NULL,
  dollar_cost REAL NOT NULL, recorded_at INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_cascades_event ON cascades(event_id);
CREATE UNIQUE INDEX IF NOT EXISTS idx_cost_cascade_component
  ON cost_log(cascade_id, component);
```

The migration also owns backward-compatible `ensureColumn` repairs and the unique Session Log / recovery indexes; they remain executable details in the authoritative source rather than a second handwritten migration in this document. `bundle_id` is stored only in the one-row bundle metadata and cascade/session evidence, not duplicated into every Leg or Edge. The prior Leg set captured in `cascades.prior_legs_json` is the rollback subject; the runtime does not maintain the originally proposed separate `snapshots` table. This is why the exact snapshot-table task language remains reconciled rather than silently claimed.

### Durable Object SQLite — Envelope Ledger

```sql
CREATE TABLE envelope (
  principal_id TEXT PRIMARY KEY,
  total_budget_minor INTEGER NOT NULL CHECK (total_budget_minor >= 0),
  currency TEXT NOT NULL
);
CREATE TABLE holds (
  hold_id TEXT PRIMARY KEY, cascade_id TEXT NOT NULL, bundle_id TEXT NOT NULL,
  leg_id TEXT NOT NULL, offer_id TEXT NOT NULL,
  amount_minor INTEGER NOT NULL CHECK (amount_minor >= 0),
  target_amount_minor INTEGER NOT NULL CHECK (target_amount_minor >= 0),
  prior_hold_id TEXT,
  state TEXT NOT NULL CHECK (state IN ('reserved','committed','released')),
  expires_at INTEGER NOT NULL,
  quarantined INTEGER NOT NULL DEFAULT 0 CHECK (quarantined IN (0,1)),
  custody_pending INTEGER NOT NULL DEFAULT 0 CHECK (custody_pending IN (0,1)),
  quarantine_reason TEXT, quarantined_at INTEGER,
  reconciliation_decision_id TEXT, reconciliation_decision TEXT,
  reconciliation_operator_id TEXT, reconciliation_reason TEXT, reconciled_at INTEGER,
  reservation_kind TEXT NOT NULL DEFAULT 'cascade' CHECK (reservation_kind IN ('cascade','ordinary')),
  operation_id TEXT, agent_id TEXT, price_verification TEXT,
  UNIQUE (cascade_id, leg_id)
);
CREATE TABLE envelope_ledger_state (
  singleton INTEGER PRIMARY KEY CHECK (singleton = 1),
  active_total_minor INTEGER NOT NULL
    CHECK (active_total_minor >= 0 AND active_total_minor <= 9007199254740991),
  revision INTEGER NOT NULL
    CHECK (revision >= 0 AND revision <= 9007199254740991)
);
CREATE INDEX idx_holds_active ON holds(state, expires_at);
CREATE INDEX idx_holds_balance ON holds(state, amount_minor);
CREATE INDEX idx_holds_cascade ON holds(cascade_id);
CREATE INDEX idx_holds_active_position
  ON holds(bundle_id, leg_id, reservation_kind, state);
CREATE UNIQUE INDEX idx_holds_committed_position
  ON holds(bundle_id, leg_id)
  WHERE state = 'committed' AND bundle_id != 'legacy' AND reservation_kind = 'cascade';
CREATE UNIQUE INDEX idx_holds_ordinary_operation
  ON holds(operation_id) WHERE reservation_kind = 'ordinary';
```

Available_Balance is computed as `total_budget_minor - active_total_minor`; it is never a separately writable balance. SQLite insert/update/delete triggers maintain `active_total_minor` and the O(1) revision in the same principal-DO transaction as each Hold mutation, and migration validates that aggregate against canonical Hold rows before serving. KV remains non-authoritative. Exact terminal identity/payload rows are retained because the idempotency contract has no expiry, but availability, conservation, revision, and overlap hot paths use the singleton aggregate or indexed predicates and never materialize lifetime history. Before a possible external settlement effect, `custody_pending=1` makes the target non-expiring; quarantine maps that protected Hold to `state='reserved', quarantined=1, custody_pending=0`, after which only authenticated reconciliation can clear custody.

### D1 — Aggregate Only

D1 holds cross-key rollups for the operator dashboard and platform audit: cascade counts, settlement-call ratios, rollback rates, envelope utilization distributions. No Cascade or check-and-reserve path reads or writes it (Requirement 8.1–8.3).

## Correctness Properties

Sixteen properties define the target contract, each with its class named so coverage gaps are visible by class rather than by count. Local evidence closes all sixteen: CP-2 instruments the implementation observer and asserts one visit per reachable leg; CP-15 generates non-empty DAG edge sets and compares two insertion orders with full recomputation. Every recorded property arm uses shrinking and prints its replayable seed.

| # | Property | Class | Validates |
|---|---|---|---|
| CP-1 | Affected_Set equals exactly the downstream-reachable set, excluding the changed leg | Invariant | Req 1.1–1.3 |
| CP-2 | BFS visits each leg at most once and terminates on every acyclic graph within the scale boundary | Invariant | Req 1.4, 1.6 |
| CP-3 | No observable state has some affected legs on new offers and others on stale offers | Invariant | Req 2.1–2.3 |
| CP-4 | Rollback restores state byte-identical to the preceding Committed_Snapshot | Round Trip | Req 2.4 |
| CP-5 | First-definitive-path call count is 1 for any non-zero net and any affected-set size; 0 for zero net or a pre-settlement rollback; recovery retries preserve one logical settlement identity | Metamorphic | Req 3.1, 3.3, 3.4 |
| CP-6 | No accepted offer exceeds Available_Balance at accept-time under arbitrary concurrent interleaving | Invariant | Req 4.1, 4.3, 4.5, 4.6 |
| CP-7 | `total_budget = available + Σreserved + Σquarantined + Σcommitted` after every transition | Invariant | Req 5.4 |
| CP-8 | Repeated commit or release of the same hold is a no-op returning the current state | Idempotence | Req 5.2 |
| CP-9 | A repeated Mutation_Event with the same event key produces at most one commit and one logical settlement identity; recovery transport attempts reuse identical identity/payload | Idempotence | Req 2.6 |
| CP-10 | Bundle serialize → deserialize yields an identical leg and edge set | Round Trip | Req 2.4, 8.6 |
| CP-11 | A divergent Balance_Cache value never changes a commit decision; the ledger value wins and the entry is invalidated | Metamorphic | Req 5.6, 9.2 |
| CP-12 | No commit occurs against an Offer_Cache entry past TTL whose revalidation has not completed | Invariant | Req 9.3, 9.4 |
| CP-13 | Insertions past 20 legs or 20 edges, cycles, and cross-principal legs are rejected with the correct typed reason and mutate nothing | Error Condition | Req 7.3–7.5, 7.7 |
| CP-14 | Every Cascade emits exactly one Cost_Log entry, and orchestration token counts are zero | Invariant | Req 10.1, 10.2, 10.7 |
| CP-15 | Incremental topological order converges to the full-recompute order for any insertion interleaving of the same edge set | Confluence | Req 7.9, 7.10 |
| CP-16 | An identical Provenance_Archive replay is idempotent, a differing digest at the same key is rejected, and archive state after N identical writes equals state after 1 | Idempotence | Req 2.7, 9.7 |

Class coverage: 6 invariant, 2 round trip, 3 metamorphic, 3 idempotence, 1 error condition, 1 confluence. CP-6 and CP-3 carry the highest run counts because they are the two hard invariants the PRD states as `0` rather than as targets to approach.

## Error Handling

Fail-closed throughout, inherited from the travel document's posture. Every failure resolves to a typed reason, never to a permissive default.

| Failure | Detected by | Resolution | Observable result |
|---|---|---|---|
| Changed leg absent | `affectedSet` | reject, mutate nothing | `rejected: unknown-leg` |
| Cycle reachable from changed leg | `affectedSet` | reject, mutate nothing | `rejected: cyclic-dependency` |
| Bundle Graph Store unreachable | `reopt-worker` preflight | reject before any Re_Quote | `rejected: store-unavailable` |
| One Re_Quote rejected / missing / malformed | `reopt-dispatch` fan-in | abort whole set, restore snapshot, release holds | `rolled-back: requote-*` |
| Cascade wall-clock cap exceeded before settlement attempt | `reopt-dispatch` | abort, restore, release | `rolled-back: cascade-timeout` |
| Cascade wall-clock cap exceeded after settlement may have been attempted | `cascade-recovery` | retain prior projection and protected Holds; defer or quarantine, never claim rollback | `pending` or `reconciliation_required` |
| Envelope insufficient for any leg | `envelope-ledger` | abort, restore, release | `rolled-back: insufficient-envelope` |
| Envelope unreachable | `guardrail-envelope-adapter` | gate rejects offer, zero holds, zero downstream | `rejected: envelope-unavailable` |
| Offer_Cache entry passes soft TTL on commit path | `offer-cache` | await the in-flight/new refresh; do not return stale data to `requote` | fresh result or typed refresh rejection |
| Issuance_Service definitively rejects before effect | `cascade-recovery` | roll back prepared graph state and release holds | `rolled-back: settlement-failed-*` |
| Issuance_Service response is retryable, timed out, malformed, or otherwise ambiguous | `cascade-recovery` | retain prior visible graph projection and protected holds; persist recovery and retry the same idempotency key/body | `pending: settlement-*` |
| An ambiguous settlement may have moved money or safe recovery cannot prove no effect | `cascade-recovery`, `reconciliation-custody` | quarantine Holds without TTL release; require separately authenticated audited operator commit/release | terminal `reconciliation_required` until converged |
| Provenance_Archive write fails after commit | `provenance-archive` | **retain the commit**, record archive-deferred | committed + `archive-deferred` record |
| Archive key already exists | `provenance-archive` | reject the overwrite | `archive-immutable` |
| Hot-path D1 call | `storage-placement-guard` | reject, name the caller | `storage-placement` violation |
| Model license neither Apache-2.0 nor MIT | `model-license-filter` | reject primary route | `license-excluded` + model + license |
| License config unreadable | `model-license-filter` | permit zero model | `license-configuration-unavailable` |
| Prod mirror or Cloudflare mutation attempted | `deploy-boundary` | reject before the request | boundary violation + component + target |

One asymmetry is deliberate: an archive failure after a successful commit does **not** roll back (Requirement 2.8). The commit is the principal's real booking state; the archive is provenance. Rolling back a correct booking because a write-behind archival call failed would trade a real outcome for a bookkeeping convenience.
