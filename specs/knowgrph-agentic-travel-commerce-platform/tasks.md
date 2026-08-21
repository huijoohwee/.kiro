---
title: "Knowgrph Agentic Travel Commerce Platform — Implementation Tasks"
doc_type: "Spec Tasks"
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
implementation_language: "TypeScript"
pbt_library: "fast-check (MIT)"
requirements_baseline: ".kiro/specs/knowgrph-agentic-travel-commerce-platform/requirements.md v1.4.0"
design_baseline: ".kiro/specs/knowgrph-agentic-travel-commerce-platform/design.md v1.4.0"
source_specification: "joohwee/prd-tad-ard/knowgrph-agentic-travel-commerce-platform-prd-tad-adr.md v1.4.0"
governing_contracts:
  - "huijoohwee.github.io/guidelines/agentic-sdlc-guidelines.md"
  - "agentic-canvas-os/docs/START-WORKFLOW.md"
  - "agentic-canvas-os/docs/AGENTS.md"
---

# Implementation Plan

## Status Reconciliation — 2026-08-20

Local implementation and deterministic evidence earn the canonical Readiness Ladder rung `dev-proven`. Readiness PR #818 is merged into protected main at `ed06fb5d56529a23ea03540ae2f3d4bf85410f1e`. The local matrices cover the production service topology, Shared Canvas atomic trigger/cold initialization, strict discovery verification, provider-backed settlement, non-expiring reconciliation custody, executable browser/presenter demo, storage authorization, paged/streamed reads, signed media, publication ACLs, both committed dependency graphs, and Worker packaging. The PR-head receipt and Integration Gate remain valid historical review evidence for `332b4ac87104837e351cf1e38457ebaf1d1753d7`, while protected-main Integration run `32426575086` passes for the squash identity. An isolated two-peer stack also passes for `ed06fb5d` and pinned Agentic Canvas OS/catalog `8233d269b1011d670ff5331dc34a8c13df730861`; only the authoritative exact-main `turn:end` receipt remains open in this review layer. Task 20.4, live capacity monitoring in Task 20.5, protected provider configuration, bootstrap receipt v2, and the seven downstream delivery receipts also remain open. `delivered_rung` remains `undocumented`.

Checkboxes are exact about behavior and evidence: `[x]` means the described outcome is implemented by the current owner map and its named local check has passed. `[ ]` means at least one behavioral or proof obligation remains unimplemented, differs from the current contract, or lacks a passing Evidence Reference. Adjacent behavior and source existence do not close a checkbox. Optional `*` tasks remain optional for a working slice, but they are checked only when their evidence exists. Original `_Scope` lines remain derivation history; where a responsibility was split or an evidence file was consolidated, `design.md` and the Authoring-Surface Evidence Ledger name the current owner.

The implementation split the Bundle Graph Store into schema, record, persistence, validation, observability, reconciliation, and recovery owners so the main Durable Object remains below 600 lines. `bundle-types.ts` is now type-only with branded identities and closed unions; every operational `*Minor` field uses validated `MinorUnits`, and recursive type-level assertions reject future raw-number money fields.

## Authoring-Surface Evidence Ledger

Use a shell rooted at the authoring worktree:

```sh
export KNOWGRPH_ROOT="$GITHUB_ROOT/knowgrph"
cd "$KNOWGRPH_ROOT"
npm run travel-commerce:test -- --disableConsoleIntercept cloudflare/workers/knowgrph-travel-commerce/test/evidence
npm run travel-commerce:typecheck
npm run travel-commerce:demo
npm run travel-commerce:demo:browser
```

Each named requirement check is exactly invocable as `npm --prefix "$KNOWGRPH_ROOT" run <check>`. The check records use schema `knowgrph-travel-commerce-check-evidence/v1` and print as `TRAVEL_COMMERCE_EVIDENCE`; the demo runner uses `knowgrph-travel-commerce-demo-evidence/v1`; the browser verifier uses `knowgrph-travel-commerce-browser-evidence/v1`.

| Evidence ID / command | Recorded authoring result |
|---|---|
| `npm run travel-commerce:test -- --disableConsoleIntercept cloudflare/workers/knowgrph-travel-commerce/test/evidence` | passed 15 files / 17 evidence tests in 132.21 s; all fourteen named checks plus `demo-runner`, with supplemental criterion records for 2.10–2.12 and 4.10–4.14 |
| `check:affected-set` | passed; CP-1 seed `1604337636` / 300 and CP-2 seed `1384604211` / 200; 100-walk p95 `9 ms`, below the 50 ms bound |
| `check:atomic-commit` | passed; CP-3 seed `238718612` / 500, CP-4 `311963087` / 200, CP-9 `493726138` / 200, CP-10 `802819226` / 200; mixed-state observations `0`; rollback snapshot byte-identical; rollback settlement calls `0`; one-active-Cascade/structure fence emits 2.10–2.12 |
| `check:net-settlement` | passed; CP-5 seed `134344832`, 400 runs; one first-definitive-path call for non-zero net, zero calls for zero net, and one stable logical identity across ambiguous retries |
| `check:envelope-atomicity` | passed; CP-6 seed `1223867732`, 600 runs; authoritative in-DO p95 `1 ms`; outer-RPC p95 `2 ms` informational only; 10,000 active zero-value rows, indexed-miss reservation `0 ms`, O(1) revision `12` bytes; authenticated ordinary-offer ingress emits 4.10–4.14 |
| `check:hold-lifecycle` | passed; CP-7/CP-8 seed `1719948052` / 300; CP-11 seed `1262333962` / 200; immediate release visibility and zero cache-derived commit decisions |
| `check:cascade-bounds` | passed; two requotes fan out, zero per-leg retries, timeout fails closed |
| `check:scale-boundary` | passed; CP-13 seed `1458004169` / 400 and CP-15 implemented arm seed `1604493119` / 300; generated rejections: legs `86`, edges `111`, cycles `109`, cross-principal `94` |
| `check:storage-placement` | passed; three hot-path D1 callers rejected; graph and ledger state restored after Durable Object eviction |
| `check:edge-cache` | passed; CP-12 seed `57866863` / 200 and CP-16 seed `235111037` / 200; repeated identical dispatches reduced from 2 to 1 |
| `check:cost-observability` | passed; CP-14 seed `1543679184`, 200 runs; one zero-token orchestration entry per generated Cascade and separate harness attribution |
| `check:inference-license` | passed; direct and overflow Workers AI paths carry permitted licenses, explicit Free-policy metering records, and fail-closed configuration behavior |
| `check:reused-interfaces` | passed for the captured `routeIntent` consumer contract; its own evidence explicitly leaves provider-contract verification to the separate provider integration check |
| `check:deploy-boundary` | passed locally; three boundaries closed and external mutations issued `0` |
| `check:replan-surface` | passed local render/replica assertions; no lost observations and rollback never rendered as committed |
| `travel-commerce:demo` | passed 8/8 deterministic beats; real-runtime 21st-Leg and cyclic-Edge mutations rejected, rejected mutations applied `0`; provider, payment, and Production mutations `0` |
| `travel-commerce:demo:browser` | passed at 320 × 800; exact executable fixture coupled to all 8 beats and retained offline; external requests `0`; horizontal overflow `0 px`, minimum touch target `58 px`, 4 semantic legs, observations `9 → 11 → 12` across offline/reconnect, transitions/reloads/reconnects `1/1/1`, lost observations `0` |
| `travel-commerce:typecheck` | passed for the travel-commerce Worker project and its exact local contracts |
| `check:agentic-travel-commerce-platform` | exact final-byte aggregate passed in the owned task worktree on 2026-08-20: root and standalone Canvas all-dependency/production-only audits each reported `0` vulnerabilities; collaboration 79/79; travel runtime/evidence 25 files / 80 tests; MCP 112/112; flight 11/11; experience 11/11; payment 61/61; net settlement 17/17; Strytree 9/9; settlement executor 9/9; operator 7/7; overflow 5/5; Shared Canvas 14/14 plus PBT 2/2; storage 42/42; Canvas build/smoke; clean-install browser demo; and every configured default/staging/production Worker dry-run. The aggregate now audits both independently committed lockfiles. |
| Protected Integration Gate (parent candidate) | GitHub Actions run `32384293256` passed for review parent `b835ef29c` in 20m26s, including immutable manifest proof, cross-device ownership, canonical integration, and XR v2 browser observation |
| Protected Integration Gate (exact candidate) | GitHub Actions run `32389800051` passed for exact review head `332b4ac87104837e351cf1e38457ebaf1d1753d7`, including immutable manifest proof, canonical integration, and XR v2 review-candidate evidence |
| Protected-main Integration Gate | GitHub Actions [run `32426575086`](https://github.com/huijoohwee/knowgrph/actions/runs/32426575086) passed every step for squash-merged main `ed06fb5d56529a23ea03540ae2f3d4bf85410f1e`, including the XR v2 runtime review-candidate gate and browser-observation upload |
| Release-controller preflight | focused suite passed 10/10; incomplete protected configuration issued `0` remote calls, serialized inventory observed maximum in-flight `1`, and a systemic Cloudflare authentication/rate-limit response stopped inventory after exactly `1` call |
| Protected-main local runtime review | isolated run-scoped stack passed for `ed06fb5d56529a23ea03540ae2f3d4bf85410f1e`: migrations, two authenticated peers, room transports, storage readiness `200`, and pinned Agentic Canvas OS/catalog `8233d269b1011d670ff5331dc34a8c13df730861` fresh in one attempt with one verification digest. Authoritative `turn:end` remains unsealed because the upstream docs main has advanced and the canonical sibling checkout preserves an unrelated dirty TODO file. |
| Read-only public preflight | current configured MCP readyz returned `404`; operator readyz is Cloudflare Access-gated; `storage.airvio.co` did not resolve. No authenticated Production probe, deployment, or public/browser receipt is claimed. |
| Reconciliation custody | focused recovery/ordinary/core/reconciliation 36/36 passed: possible-effect protection before attempt, alarm-before-recovery exclusion, >24h quarantine retention, operator auth/readiness, idempotent commit/release, drift rejection, ledger/graph convergence, later mutation |
| Overflow binding integrity | passed `travel-commerce:overflow:workers-ai:check`: the legacy-named overflow Worker declares remote Workers AI for default/staging/production, zero Containers, provider model `@cf/openai/gpt-oss-20b`, and the explicit 10,000-daily-neuron policy |

These records earn `dev-proven`. The local records alone do not close every original task bullet; the review-ready PR and its exact-head protected check bind the immutable reviewed-candidate receipt, but not the separate protected-integration or delivery receipts.

## Protected And Delivered Receipt Ledger — Intentionally Open

- [x] Historical immutable reviewed-candidate receipt — PR #818 review head `332b4ac87104837e351cf1e38457ebaf1d1753d7`
- [x] Historical protected-integration receipt for PR head `332b4ac87104837e351cf1e38457ebaf1d1753d7` — [Integration Gate run 32389800051](https://github.com/huijoohwee/knowgrph/actions/runs/32389800051) passed
- [x] Protected-main Integration receipt for squash result `ed06fb5d56529a23ea03540ae2f3d4bf85410f1e` — [run `32426575086`](https://github.com/huijoohwee/knowgrph/actions/runs/32426575086)
- [ ] Shared Canvas Node → travel-commerce Worker production binding/trigger receipt
- [ ] Authoritative exact-main `turn:end` local runtime review receipt — isolated two-peer evidence passes for `ed06fb5d56529a23ea03540ae2f3d4bf85410f1e` and pinned catalog `8233d269b1011d670ff5331dc34a8c13df730861`, but the canonical receipt is not sealed
- [ ] Exact-candidate human Production authorization
- [ ] Production deployment receipt
- [ ] Authoritative state-reconciliation receipt
- [ ] Public-route and browser live-verification receipt
- [ ] Publication receipt
- [ ] Exercised rollback receipt

Source configuration and Wrangler dry runs may prove packaging only. They cannot check any item above, raise `delivered_rung`, or establish a live-production claim.

## Technical Task Closure

Every scoped travel authoring task is terminal. Task 20.1–20.3 are now source-complete and locally checked; Task 20.4 and the live monitoring obligation in Task 20.5 remain open and are not waived by local evidence. Task 1.2 brands every operational `*Minor` money field as validated `MinorUnits`, preserves `bundle-types.ts` as a type-only module with zero runtime exports, and adds a recursive type assertion that rejects raw-number money fields.

Top-level Task 1 and checkpoint 3 are therefore closed. Task 5 and checkpoint 6 are closed by the separate `affectedSet`/`isPresent` surface, once-per-wake adjacency cache, CP-2 visit instrumentation, Durable Object timing, and exact per-Cascade Session Log evidence. Checkpoint 19 records the exact final-byte aggregate; it does not promote or deploy the candidate. The protected receipt ledger is intentionally separate and stays open.

## Deploy Boundary Statement — Read Before Dispatching Any Task

Every task in this list executes entirely within `Dev_Lane` (`GitHub/knowgrph`, `npm run dev:apex`, `npm run dev`).

- **No task in this list may mutate the Prod mirror** (`GitHub/huijoohwee/content/knowgrph`).
- **No task in this list may mutate a Cloudflare route** (`airvio.co`, `airvio.co/knowgrph`).
- Every task's capability class is one of `read`, `local write`, or `local execute`. **Zero tasks carry `environment mutate` or any boundary-crossing capability**, with one stated exception: task 1.1 adds a pinned dev-only dependency, which is recorded here in advance rather than escalated mid-task.
- All three Deploy Boundary Register rows report `closed` at the start and at the end of this increment (Requirement 13.1).
- No task issues a real payment call. Every payment path in the recorded evidence uses deterministic local service doubles.

## Order Discipline

Task order follows the source specification's own Next Steps, not convenience:

1. **Envelope Ledger first** — it is the smaller self-contained primitive, and the cascade's real-time settlement claim is only true because the envelope check underneath it is genuinely atomic (PRD Next Step 1).
2. **Bundle Graph Store and Re-optimization Worker next**, against a 2-leg MVP bundle (PRD Next Step 2).
3. **Cache layers after the core hot path is dev-proven**, not before — caching a component that does not exist yet is premature optimization against the source document's own discipline (PRD Next Step 3).
4. **Oracle retirement as an independent workstream** — task group 13 has no dependency edge into the cascade path (PRD Next Step 4, ADR-3).

## Current Implementation Owner Map

The original seventeen-unit plan split further during implementation. This table names the current principal owners; it is authoritative over historical `_Scope` lines in individual tasks.

| Unit | Path | Declared responsibility |
|---|---|---|
| Bundle Types | `src/bundle/bundle-types.ts` | Type-only closed contracts and branded identities; zero runtime values |
| Bundle Runtime | `src/bundle/bundle-runtime.ts` | Runtime constants, bounds, identifiers, decoders, and checked `MinorUnits` construction |
| Envelope Ledger | `src/ledger/envelope-ledger.ts` | `envelope`/`holds` DO storage and atomic check-and-reserve |
| Envelope Ledger Schema | `src/ledger/envelope-ledger-schema.ts` | SQLite schema, indexes, and bounded additive migrations |
| Envelope Ledger Records | `src/ledger/envelope-ledger-records.ts` | Ledger row mapping, money/currency/verification validation, and shared deterministic reservation helpers |
| Envelope Ledger State / Alarms | `src/ledger/envelope-ledger-state.ts`, `src/ledger/envelope-ledger-alarms.ts` | O(1) trigger-maintained active-total/revision reads, non-expiring possible-effect custody, repairable alarm scheduling, and expiry transitions |
| Ordinary Offer Holds | `src/ledger/ordinary-offer-holds.ts` | Stable ordinary reservation identity plus idempotent reserve/commit/release transitions |
| Hold Lifecycle | `src/ledger/hold-lifecycle.ts` | Hold state machine and conservation invariant |
| Bundle Graph Store | `src/bundle/bundle-graph-store.ts` | Durable Object coordinator for structure, Cascades, recovery, alarms, and WebSockets |
| Bundle Graph helpers | `src/bundle/bundle-graph-{schema,records,storage,validation,observability,initialization}.ts` | Split schema, row mapping, persistence, validation, observability, and initialization responsibilities |
| Cascade state/recovery | `src/bundle/bundle-settlement-state.ts`, `src/bundle/cascade-{outcomes,recovery}.ts` | Durable settlement attempt/claim state, terminal outcomes, retry/finalize/rollback/reconciliation |
| Topological Order | `src/bundle/topo-order.ts` | Deterministic topological order, cycle rejection, and affected-set calculation |
| Re-opt Dispatch | `src/bundle/reopt-dispatch.ts` | Concurrent Re_Quote fan-out / fan-in |
| Re-opt Worker | `src/bundle/reopt-worker.ts` | One-pass Cascade orchestration and net settlement |
| Guardrail Envelope Adapter | `src/gate/guardrail-envelope-adapter.ts` | Supply Available_Balance to the reused gate, interface untouched |
| Travel Agency Guardrail Service | `src/gate/travel-agency-guardrail-service.ts` | Named same-lane entrypoint joining inherited Guardrail evaluation to ordinary-offer reservation/lifecycle |
| Balance Cache | `src/cache/balance-cache.ts` | KV read-through cache, non-authoritative by construction |
| Offer Cache | `src/cache/offer-cache.ts` | Cache API TTL + stale-while-revalidate wrapper |
| Provenance Archive | `src/archive/provenance-archive.ts` | R2 write-once snapshot and receipt writer |
| Storage Placement Guard | `src/runtime/storage-placement-guard.ts` | Reject a hot-path D1 call; reject a new storage system |
| Cost Log | `src/runtime/cost-log.ts` | Per-Cascade cost entry emission |
| Model License Filter | `src/runtime/model-license-filter.ts` | Permitted_Model_Set derivation, fail-closed |
| Inference Router | `src/runtime/inference-router.ts` | Direct Workers AI primary / authenticated Workers AI Worker overflow selection |
| Deploy Boundary | `src/runtime/deploy-boundary.ts` | Evidence-derived boundary state, fail-closed |
| Runtime Readiness | `src/runtime/readiness.ts` | Bounded binding and dependency readiness probes |
| Bounded JSON Boundaries | `src/runtime/bounded-json.ts`, `cloudflare/workers/knowgrph-mcp/bounded-json.mjs`, `cloudflare/workers/knowgrph-payment/travelAgency/boundedJson.ts` | Cancel oversized request/response streams before complete buffering at each travel boundary |
| MCP Travel Ingress / Router / Agent Definition Cache | `cloudflare/workers/knowgrph-mcp/travel-commerce-ingress.mjs`, `travel-commerce-router.mjs`, `index.ts` | Authenticate registered-offer ingress, preserve inherited Re_Quote routing, expose separately guarded ordinary offers, dispatch category adapters, and own memory/KV definition caching |
| Storage Public-Route Security, Paging, Media, And Publication | `cloudflare/workers/knowgrph-storage/storage{PublicRouteSecurity,SyncSecurity,SyncCursor,SyncPageRows,SyncReadRows,SyncReadRuntime,DocumentRouteSecurity,DocumentReadBounds,DocumentStream,CrawlerCursor,MediaCapability,Publication}.ts`, `chatRelayBodyBounds.ts`, `cloudflare/workers/shared/publishedDoc.ts`, migration `0015_storage_publication_contract.sql`, and the bounded/paged Canvas storage client | Authenticate before body reads; enforce active-user/workspace roles; stream documents; keyset-page sync/export/crawlers; sign workspace/object/operation/expiry media capabilities; stamp and verify R2 ownership; and expose anonymous content only through an exact current publication record |
| Provider / Operator Service Boundaries | `cloudflare/workers/knowgrph-travel-discovery`, `knowgrph-travel-experience-discovery`, `knowgrph-travel-settlement-executor`, `knowgrph-travel-operator-gateway`, and the isolated payment net-settlement entrypoint | Strict Search→Verify, provider-effect receipt, Access-authenticated reconciliation, and service-only journal boundaries |
| Workers AI Overflow Runtime | `cloudflare/workers/knowgrph-travel-ollama-overflow` (legacy resource name) | Token-authenticated Worker with remote Workers AI binding, bounded inference/readiness surface, strict provider-model allowlist, and zero Container binding |
| Re-Plan Surface | `src/ui/replan-surface.ts` | Mobile-first, local-first Cascade projection and render |
| Demo Evidence Lifecycle | `scripts/travel-commerce/demo-evidence.mjs`, `scripts/travel-commerce/run-demo.mjs`, `scripts/travel-commerce/verify-demo-browser.mjs`, `canvas/src/features/testing/TravelCommerceDemoPage.tsx` | Bounded evidence loading, CLI/headless proof, distinct headed presenter lifecycle, and mobile rendering |
| Protected Travel-Mesh Release Controller | `scripts/travel-mesh-release-{plan,inventory,bindings,probes}.mjs`, `scripts/travel-mesh-release.mjs`, `scripts/install-production-release-dependencies.sh`, `.github/workflows/release.yml`, `.github/workflows/runtime-gate.yml` | Validate dependencies and complete protected configuration before any remote call; serialize exact resource/binding inventory and stop systemic access failures; install locked release dependencies, upgrade an authorized nine-Worker baseline, probe it, and restore prior versions or preserve ambiguous state; first provisioning requires the distinct authorized bootstrap receipt |

`src/bundle/wiring.ts` owns local fixture/wiring assembly. The once-per-wake adjacency cache and the fully branded operational money contract are implemented and evidenced. The protected release-controller row names source ownership only; it cannot close any protected or delivered receipt and cannot raise `delivered_rung`.

**Task marking convention.** Sub-tasks postfixed with `*` are **not required for a working slice** — they are property tests, integration checks, measurements, browser assertions, static scans, and process assertions. Sub-tasks without `*` are **required for a working slice**. Top-level tasks are never postfixed.

**Bounds convention.** Per the governing guidelines' Per-Task Budgets rule, every task states four bounds plus a circuit breaker on one `_Bounds:_` line: token budget · iteration cap · wall-clock cap · peak working-context cap · breaker. The default breaker is: two consecutive iterations with no change in the named check's recorded result → stop retrying, transition the task to `failed`, record the last observed result and a terminal reason. Raising a bound to rescue a failing task is forbidden; re-decompose instead.

---

## Responsibility Shards

- [`tasks-core.md`](./tasks-core.md) owns Tasks 1–10: foundations, graph/ledger correctness, and the Cascade hot path.
- [`tasks-runtime-delivery.md`](./tasks-runtime-delivery.md) owns Tasks 11–20: caches, inference, demo/runtime wiring, checkpoints, and still-open Production lifecycle obligations.
- [`tasks-dependency-graph.md`](./tasks-dependency-graph.md) owns execution notes, dependency graph, dispatch waves, and bridge coverage.

Together these four v1.4.0 files form the complete implementation plan. Checkbox and receipt semantics remain those defined above.
