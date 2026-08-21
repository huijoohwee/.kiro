---
title: "Knowgrph Agentic Travel Commerce Platform — Demo Walkthrough"
doc_type: "Spec Demo"
schema: "kiro-spec-demo/v1"
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
demo_runtime: "Dev only — GitHub/knowgrph via npm run travel-commerce:demo:present"
demo_duration_target: "under 7 minutes"
requirements_baseline: ".kiro/specs/knowgrph-agentic-travel-commerce-platform/requirements.md v1.4.0"
design_baseline: ".kiro/specs/knowgrph-agentic-travel-commerce-platform/design.md v1.4.0"
tasks_baseline: ".kiro/specs/knowgrph-agentic-travel-commerce-platform/tasks.md v1.4.0"
source_specification: "joohwee/prd-tad-ard/knowgrph-agentic-travel-commerce-platform-prd-tad-adr.md v1.4.0"
governing_contracts:
  - "huijoohwee.github.io/guidelines/agentic-sdlc-guidelines.md"
  - "agentic-canvas-os/docs/START-WORKFLOW.md"
  - "agentic-canvas-os/docs/AGENTS.md"
---

# Demo Walkthrough

A presenter script for the consolidation increment, run entirely in the Dev lane. Requirement, property, and task numbers reference `requirements.md`, `design.md`, and `tasks.md`; nothing is restated here.

The through-line: a traditional OTA makes the human re-plan when a flight slips. This demo shows the platform doing it — only the causally affected legs, all or nothing, and one idempotent net-settlement operation instead of four independent settlements.

## What This Demo Proves

| Observable claim | Independently verified by |
|---|---|
| A change to one leg re-plans exactly the causally downstream legs, and touches no unaffected sibling | `npm run check:affected-set` |
| The whole affected set commits or none of it does, and a rollback restores the prior state exactly | `npm run check:atomic-commit` |
| One cascade owns one logical settlement operation and one stable idempotency key, whatever the affected-set size; the first definitive path uses one transport call and ambiguous retries preserve identical identity/payload | `npm run check:net-settlement` |
| Two concurrent reservations against one principal ledger can never jointly overspend one budget | `npm run check:envelope-atomicity` |
| A registered ordinary offer traverses the separately named Guardrail route and shares that ledger with Cascades | `npm run travel-commerce:test -- --disableConsoleIntercept cloudflare/workers/knowgrph-travel-commerce/test/ordinary-offer-atomicity.test.ts` plus `npm run mcp:worker:test` (supplemental evidence, not Beat 5's presenter path) |
| Released funds are available to other in-flight agents with no observable staleness window | `npm run check:hold-lifecycle` |
| The re-plan surface stays legible at phone width and survives an actual offline reload/reconnect | `npm run check:replan-surface`, the headed presenter's bounded self-rehearsal, and the separate `npm run travel-commerce:demo:browser` headless context |

What the demo does **not** prove:

- **Nothing here is a load or performance result.** The 50 ms walk and 10 ms envelope check are asserted by measurement in checks, with recorded values. What the audience sees on screen is not the evidence.
- **The scale boundary is real.** Twenty legs and twenty edges per bundle. Beyond that, insertion is rejected with the observed count. We show that rejection on purpose (Beat 6) rather than hoping nobody asks.
- **No provider, payment, deployment, or Production surface is touched.** The runner accepts only an explicit `--local-demo` mode and the browser proof accepts loopback URLs only. Its evidence records zero external requests and zero Production mutations.
- **Inference has a hard free-policy ceiling.** Primary and overflow use Workers AI under the explicit 10,000-daily-neuron policy and fail closed outside it. Beat 7 says so out loud, because the ADR does.

## Preconditions

Run from `$GITHUB_ROOT/knowgrph` after installing the locked workspace dependencies. The browser and presenter scripts prepare their linked Canvas packages themselves after a clean `npm ci`; no manual package build is required. No provider credential, payment credential, Cloudflare login, or Production resource is required or accepted by the deterministic demo path.

The command-line proof is:

```sh
npm run travel-commerce:demo
```

The full browser proof is:

```sh
npm run travel-commerce:demo:browser
```

The headed presenter is:

```sh
npm run travel-commerce:demo:present
```

All three package scripts add the required `--local-demo` flag. Invoking the underlying runner without that flag fails closed. The CLI and browser-proof paths execute and strictly validate the same eight-beat record. The browser proof remains automated and headless: it writes a uniquely named ephemeral readout, starts a temporary Canvas Vite server, opens only `127.0.0.1`, compares every rendered beat and visible evidence group with that exact record at 320 × 800 CSS pixels, performs an actual offline reload and reconnect, prints the browser-session counters, then removes the readout and stops the server.

**Current clean-install rehearsal (2026-08-20):** CLI 8/8; browser 8/8 at 320 CSS px, 0 horizontal overflow, 58 px minimum touch target, one offline transition/reload/reconnect, zero lost observations, zero provider requests, zero real payment calls, and zero Production mutations. The headed presenter independently completed its service-worker rehearsal in about 1.6 seconds within the 15-second bound and selected Beat 8 before announcing readiness.

`travel-commerce:demo:present` is the human presentation lifecycle. It executes and validates a fresh record, starts a strict-port loopback server, and opens a fresh non-persistent headed 320 × 800 presenter context. Before announcing readiness, that same headed context performs one end-to-end 15-second-bounded service-worker rehearsal: it confirms control of the exact demo route, transitions offline, reloads that route through the service worker, returns online, verifies non-zero offline-transition/reload/reconnect counters with zero observation loss, and selects Beat 8. Only after those checks pass does it print `TRAVEL_COMMERCE_DEMO_PRESENTER_REHEARSAL` and the exact `TRAVEL_COMMERCE_DEMO_PRESENTER_URL`. The evidence file and server stay available until the presenter explicitly closes the tab/window or presses q, Enter, or Ctrl+C in the terminal. Only then are the browser context, server, and the exact bounded evidence file cleaned up. The presenter owns its rehearsal counters; it does not inherit state from the separate headless verifier. The headless command independently repeats the browser/offline proof as the automated acceptance artifact.

`KG_TRAVEL_COMMERCE_DEMO_PORT` may select another unoccupied local port. An already-running loopback surface may be selected with `KG_TRAVEL_COMMERCE_DEMO_URL` only when that surface serves this worktree and its generated evidence path; non-loopback URLs are rejected.

The runner creates its own deterministic four-leg fixture: one changed flight, two causally downstream legs, and one unaffected hotel sibling. It also creates isolated fixtures for rollback, zero-net settlement, concurrent envelope reservation, the 21-leg boundary, and cycle rejection. No manual seed step is required.

## Demo Script

### Beat 1 — The dependency structure is a real object

- **Presenter:** open the re-plan surface at phone width and select the already-executed Beat 1 readout. Show the seeded bundle's legs and the edge between them.
- **Viewer sees:** legs rendered as a semantic list, each row naming its leg identifier, its committed offer, and its committed amount. The downstream leg is visibly downstream — the edge is data, not a diagram someone drew.
- **Spec claim:** Requirement 14.1, 14.2 — legible at 320 CSS px with an accessible name per row. Requirement 7.1 — two flat tables, no graph engine anywhere.
- **Say this out loud:** everything after this beat is a consequence of the platform knowing that arrow exists.

### Beat 2 — One change, only the downstream legs

- **Presenter:** select the already-executed Beat 2 delay readout; explain that the runner issued the flight mutation and did not mutate any sibling leg.
- **Viewer sees:** the affected set appears, containing exactly the legs reachable downstream of the flight — and not the flight itself. An unrelated sibling leg, seeded deliberately, stays untouched with its original offer.
- **Spec claim:** Requirement 1.1–1.3 — the affected set equals exactly the downstream-reachable set, excluding the changed leg. Property CP-1.
- **Say this out loud:** this is the difference between "suggest an alternative for the leg that broke" and "re-plan the legs that broke *because of it*."

### Beat 3 — All of it, or none of it

- **Presenter:** select the two already-executed Beat 3 outcomes: the clean cascade, then the same deterministic delay with one downstream harness configured to reject its re-quote.
- **Viewer sees:** first run — every affected leg moves to its new offer together. Second run — nothing moves. Both legs sit on their prior offers, the rollback reason reads `requote-rejected`, and the surface labels the cascade rolled back rather than partially applied.
- **Spec claim:** Requirement 2.1–2.4 — one atomic transaction across the affected set, byte-identical restoration on rollback. Properties CP-3 and CP-4.
- **Say this out loud:** there is no partial-retry path. One rejection aborts the set, because partial retry is exactly how you end up half-booked.

### Beat 4 — One cascade, one settlement

- **Presenter:** select the already-executed Beat 4 multi-leg cascade record, then its zero-net companion readout.
- **Viewer sees:** the two-leg non-zero scenario, its cascade ID repeated as the idempotency key, settlement-call count **1**, and the signed net amount. Its exact replay leaves the call count at **1**. The separately executed zero-net companion shows settlement-call count **0** and a `zero-net` record. Ambiguous-transport retry identity remains covered by `check:net-settlement`, not simulated in the presenter.
- **Spec claim:** Requirement 3.1–3.3, 3.6 — one logical operation regardless of affected-set size, one transport call on the first definitive path, zero for a zero net, and identical identity/payload on any ambiguous recovery retry. Attempts and identity are recorded rather than inferred. Property CP-5.
- **Say this out loud:** this is the beat that makes the L4 claim literal. Drop it and this is a nicer L3.

### Beat 5 — Two concurrent ledger reservations, one budget, no overspend

- **Presenter:** select the already-executed Beat 5 concurrency readout. The runner submitted two simultaneous Cascade-channel reservations directly to the principal's Envelope Ledger; their demo identities are agent-labelled, but this beat does not traverse Agent Registry/Router or the named ordinary-offer Guardrail entrypoint.
- **Viewer sees:** one reserved, one rejected with `insufficient-envelope`. Then release the reserved hold and immediately resubmit the rejected offer: it now passes, with no waiting period.
- **Spec claim:** Requirement 4.1–4.6 — check-and-reserve is one indivisible operation against true available balance. Requirement 5.3 — released funds reappear with no observable staleness window. Properties CP-6 and CP-7.
- **Evidence boundary:** the separately focused ordinary-offer test and `mcp:worker:test` prove the registered offer → named Guardrail → shared principal ledger path for Requirements 4.10–4.14; this presenter beat proves the underlying concurrent ledger invariant only.
- **Say this out loud:** the ledger accounts for every hold in flight. The registered ordinary-offer route is a separate focused contract, not something this card simulates.

### Beat 6 — The scale boundary, shown rather than hidden

- **Presenter:** select the already-executed Beat 6 boundary readout. The runner attempted a real 21st-leg insertion and then a real cycle-forming edge insertion.
- **Viewer sees:** both rejected, each naming the limit and the observed count, and the bundle unchanged. `check:scale-boundary` reads the limit from the exported constant rather than a number the test invented.
- **Spec claim:** Requirement 7.3–7.5, 7.8 — the boundary is a declared, readable constant, and crossing it is a typed rejection. Property CP-13.
- **Say this out loud:** "no graph database" is an honest constraint because its limit is written down. If we ever hit it, the fix is sharding the flat tables, not adopting a graph engine that is still the wrong tool at twenty rows.

### Beat 7 — Cost, caches, and the one honest non-zero

- **Presenter:** select the already-executed Beat 7 Cost_Log and cache readout. It includes the original cascade and the runner's identical second request inside the offer-cache TTL.
- **Viewer sees:** the executed orchestration Cost_Log with prompt tokens `0`, completion tokens `0`, and dollar cost `0`. Two identical quote requests show two baseline dispatches versus one cached dispatch. The eligible model declaration shows its model ID, `workers-ai` path, Apache-2.0 license, metered input/output rates, and the explicit fact that orchestration did not invoke it.
- **Spec claim:** Requirement 10.2, 9.6 — model-free orchestration with recorded dispatch reduction. Requirement 11.6, 11.8 — per-call license and cost recorded, neither path claimed free. Property CP-14.
- **Say this out loud:** the cascade is arithmetic. It costs storage reads, not tokens. Inference costs money and we are not going to pretend otherwise.

### Beat 8 — Offline on a phone

- **Presenter:** the presenter arrives on Beat 8 automatically after its headed self-rehearsal. Point to the deterministic local projection/convergence record and the adjacent `Current browser network-proof session` card.
- **Viewer sees:** the executed fixture's committed projection retained offline and its converged rolled-back observation with zero loss. The same visible headed session shows non-zero offline-transition, offline-reload, and reconnect counters, its before/after observation counts, and lost observations `0`. The separate headless verifier independently repeats the network sequence and prints its own counters.
- **Spec claim:** the deterministic card demonstrates the local projection/convergence fixture. The presenter's headed self-rehearsal and the separate headless command both exercise the browser behavior for Requirement 14.4–14.7. Requirement 8.7's hibernatable-WebSocket persistence remains a storage/runtime check rather than a headed-presenter action.

## Evidence Appendix

| Beat | Invocable check | Properties | Where recorded evidence lands |
|---|---|---|---|
| 1 | `npm run check:replan-surface`, `npm run check:scale-boundary` | CP-13 | Rendered row set, accessible names, overflow assertion; static scan result for zero graph engine; exit status and seed |
| 2 | `npm run check:affected-set` | CP-1, CP-2 | Session_Log entry per cascade with `bundle_id`, changed leg, affected identifiers in traversal order, outcome; measured walk latency; exit status, counts, seed |
| 3 | `npm run check:atomic-commit` | CP-3, CP-4, CP-10 | Pre- and post-cascade snapshot digests; rollback reason; mixed-state assertion count; exit status, seed |
| 4 | `npm run check:net-settlement` | CP-5 | Per-cascade affected-set size, settlement-call count, net amount, gateway caller identity; exit status, seed |
| 5 | `npm run check:envelope-atomicity`, `npm run check:hold-lifecycle` | CP-6, CP-7, CP-8 | Cascade-channel available-balance-at-check, hold transitions, conservation, and reserve latency. Focused ordinary-offer/MCP checks are supplemental and are not simulated by this beat |
| 6 | `npm run check:scale-boundary` | CP-13, CP-15 | Rejection reasons with observed counts; declared-limit constant read; topological order comparison; exit status, seed |
| 7 | `npm run check:cost-observability`, `npm run check:edge-cache`, `npm run check:inference-license` | CP-12, CP-14, CP-16 | One Cost_Log entry per cascade; dispatch counts with and without cache; per-call path, model, license, cost; archive write-once result; exit status, seed |
| 8 | `npm run travel-commerce:demo:present`, `npm run travel-commerce:demo:browser`, `npm run check:replan-surface`, `npm run check:storage-placement` | — | Headed presenter self-rehearses and visibly retains actual offline/reload/reconnect counters with zero loss; headless context independently repeats and prints them; deterministic fixture records projection/convergence; storage check owns hibernation |

Every check exits non-zero on failure and prints its recorded counts, measured values, and property run seeds, so a failing beat is reproducible from the printed seed alone.

## Cost And Token Economics Talking Point

BFS, atomic commit, rollback, net computation, hold arithmetic, and topological order maintenance are deterministic. Both new pipelines carry a `$0.00/call` token budget. The only token spend in a cascade belongs to the reused Discovery Harnesses re-quoting downstream legs — and the offer cache cuts how often that happens, because most re-plans re-query the same route or item within seconds of the original quote.

Marginal infrastructure: zero new vendor and zero new storage category. The local demo uses Miniflare-backed Durable Objects, KV, Cache API, and R2 doubles and requires no remote resource. Production resource IDs, namespaces, buckets, bindings, routes, and credentials remain explicit operator provisioning prerequisites; no free-tier assumption is a readiness receipt.

One-sentence version: **re-planning is deterministic arithmetic with zero orchestration tokens, and a non-zero cascade owns one logical net-settlement operation under one stable idempotency key; only ambiguous transport outcomes may retry that same operation.**

## Failure Modes During Live Demo

| Live failure | Observed symptom | Why it is correct fail-closed behavior | Recovery line |
|---|---|---|---|
| Executable evidence is missing or violates any per-beat relationship | Presenter exits before opening the surface, or the page labels evidence `failed`; no claim cards are rendered | The UI requires edges, offer snapshots, both rollback outcomes, both settlement scenarios, release/resubmission, cost/model/license, and offline/reconnect fields—not merely eight `passed` labels | "The readout and runtime record disagree, so the demo refuses to present it." |
| Presenter port is already owned | Vite exits under `--strictPort`; no alternate or stale server is opened | A silent port fallback could show a different checkout or stale evidence | "This port already belongs to another local runtime; I will choose an explicit unused demo port." |
| Headed Chromium cannot launch | `travel-commerce:demo:present` exits non-zero and removes its bounded evidence file after stopping any server it started | A headless proof is not substituted for the human presenter lifecycle | "The executable record passed, but the headed presentation surface is unavailable on this host." |
| Headed offline/service-worker rehearsal fails or exceeds its bound | No presenter-ready URL is announced; the command exits non-zero, closes its context, stops its local server, and removes its bounded evidence file | The presenter never displays zero or unverified network counters as a ready Beat 8 claim | "The headed browser could not prove its offline reload and reconnect, so it refused to announce the presenter as ready." |
| Presenter closes the tab/window or presses q, Enter, or Ctrl+C | The command records an explicit presenter exit, then stops the local server and removes only its own evidence file | Evidence persists for the whole presentation but is not left stale afterward | "The presentation lifecycle ended explicitly and cleaned up its own local artifacts." |
| Envelope ledger unreachable | Every offer rejected with `envelope-unavailable`; zero holds reserved; nothing reaches a downstream component | The balance is an authoritative read, not an optimistic assumption. Unreadable means unauthorized, not "probably fine" | "It can't confirm the remaining budget, so it approves nothing." |
| Stale cached balance | Outcome identical to the uncached path; the divergent entry is invalidated | The KV cache returns `mustConfirm: true`, so a commit decision structurally cannot be reached without confirming against the ledger | "The cache is a speed-up, never an authority. The type signature enforces that, not a code comment." |
| Offer cache entry past TTL, revalidation incomplete | Cascade rolls back with `stale-offer-cache-entry`; zero commit; zero settlement | Committing at a price whose quote expired is the one thing a cache could plausibly buy you, so it is refused explicitly | "The quote aged out mid-cascade. It refuses to buy at a price it can't currently confirm." |
| One downstream re-quote times out | Whole affected set aborts with `cascade-timeout`; snapshot restored; holds released | A cascade is one bounded pass with a stated cap. A partial retry would be how a mixed state gets created | "One leg stalled, so the whole re-plan backs out. Nothing half-changed." |
| Settlement is known to have failed before any provider effect | Graph commit rolls back and holds remain in release recovery until the ledger confirms release | A definitely effect-free failure can be reversed safely, but release failure is never presented as final | "The provider did not move money, and the booking change remains blocked until its holds are confirmed released." |
| Settlement result is ambiguous after an attempt | The cascade remains pending or enters `reconciliation-required`; no rollback is claimed | Retrying uses one stable idempotency key and payload. Rolling back after money may have moved would create a worse split-brain outcome | "The payment result is ambiguous, so the platform preserves the committed intent and asks recovery or an operator to reconcile it." |
| Archive write fails after commit | Commit **retained**; `archive-deferred` recorded | Provenance is bookkeeping; the commit is the principal's real booking state. Rolling back a correct booking for a failed archival write would trade a real outcome for tidy records | "The booking stands. Only the archive copy is deferred, and it's recorded as deferred." |
| Cycle in the seeded fixture | Cascade rejected with `cyclic-dependency`; nothing mutated | A cycle means "A depends on B depends on A", which has no correct re-plan order. Failing beats looping | "The fixture has a circular dependency. It refuses to guess an order. Let me reseed." |
| Model license config unreachable | Every inference route rejected with `license-configuration-unavailable` | A license filter that opens under error is decoration, not a filter | "It can't verify the model licenses, so it routes to nothing." |
| One required config key missing | The serverless process remains live, but `/readyz` returns 503 and names the absent capability/configuration | No built-in default exists for any provider endpoint or credential, so a partially configured deployment cannot become ready or accept a protected workflow | "It stays fail-closed: live enough to diagnose, unavailable for work, and names the exact operator action." |

## Not In This Demo

Questions a viewer will reasonably ask, with the honest answer:

- **Group trips, split pay, or a shared wallet?** No. Cross-principal bundles are rejected outright. That is the Split-Pay roadmap item and it needs a different envelope data model, deliberately not built early here.
- **Bundles bigger than twenty legs?** No, and Beat 6 shows the rejection. Sharding the flat tables is the fix if we ever need it.
- **A graph database anywhere?** No, at any layer. ADR-4, enforced by static scan.
- **Predictive re-planning ahead of a delay?** No. Re-optimization triggers on an actual Shared Canvas mutation event, never on a forecast.
- **Recursive cascades?** No. One event, one pass. A commit cannot trigger another cascade.
- **Multi-currency netting?** No. One currency per envelope in this increment.
- **Chargeback or dispute-triggered hold adjustment?** No. Later roadmap phase.
- **Any Oracle-hosted inference?** No — retired from the critical path entirely, and a scan asserts no Oracle endpoint, credential key name, or SSH config remains.
- **Prod mirror or Cloudflare deployment?** No. All three boundaries stay `closed` for the whole session. The local process check digests the available Prod-mirror path before and after, while the demo and boundary guard issue zero Cloudflare delivery-route request; no live remote byte-identity claim is inferred.

## Production Truth Boundary Called Out Live

The three v1 decisions are resolved and recorded in tasks 18.1–18.3: every validated unique mutation triggers, rollback is returned to the caller and projected through the Session Log/WebSocket rather than a new notification channel, and `available_balance` remains server-side only.

The deterministic demo proves authoring-surface behavior, repeatability, mobile layout, offline retention, and the absence of accidental external effects. It does **not** establish protected integration or a live Production deployment. Before a Production claim, operators still must:

1. retain the aggregate audit of every independently committed dependency graph and require all-dependency plus production-only audits to remain at `0` findings;
2. provision the named Cloudflare KV/R2/Durable Object and service bindings, a strong `KNOWGRPH_STORAGE_SIGNING_SECRET`, plus the immutable Shared Canvas `{schema, revision, entries}` bundle-seed map and matching service token;
3. configure Atlas and live-experience provider origins/paths, exact route catalogues, credentials, and production/staging secrets, then run both adapters' Search→Verify identity/latency UAT;
4. replace the settlement executor's fail-closed `.invalid` Issuance Service origin, provision its Bearer secret, prove the authenticated provider capability/effect receipts, and provision a distinct Production `RECONCILIATION_OPERATOR_TOKEN`;
5. complete the protected configuration, run the serialized read-only inventory, and obtain separate human authorization for first provisioning; create the exact nine Workers/resources/routes and one 100%-active baseline per Worker, then issue `knowgrph-travel-mesh-bootstrap-receipt/v2` through protected `TRAVEL_MESH_BOOTSTRAP_RECEIPT_JSON`, binding the exact Workers AI Free policy and resource/probe/route/subdomain digests—the ordinary release controller is upgrade-only and refuses an absent baseline;
6. verify the overflow Worker's remote Workers AI binding, strict `@cf/openai/gpt-oss-20b` allowlist, 10,000-daily-neuron policy, and absence of any Container binding, then upgrade the authorized exact candidate in executable dependency order: settlement executor → net settlement → flight discovery → experience discovery → overflow → travel-commerce → MCP → operator gateway → storage trigger;
7. retain PR #818 head `332b4ac87` as historical review evidence and protected-main Integration run `32426575086` as the passing receipt for squash result `ed06fb5d5`; seal the authoritative exact-main `turn:end` receipt, then obtain the remaining seven delivery receipts listed in `tasks.md`: Shared Canvas production binding/trigger, exact-candidate human Production authorization, deployment, state reconciliation, public/browser live verification, publication, and exercised rollback. The bootstrap receipt is a separate prerequisite, not an additional delivery receipt.

Those storage functionality contracts are now source-complete: opaque snapshot-bound keyset pages cover sync/export/crawlers; documents stream bounded database segments; HMAC media capabilities bind subject/workspace/object/operation/expiry and verify R2 ownership metadata; anonymous delivery requires an exact current publication revision/hash; and released terminal Holds compact into versioned minimal replay receipts while retaining indefinite exact replay/conflict semantics. The currently served storage runtime is not credited with any of these local changes until the authorized deployment, state reconciliation, live browser/public-route probes, publication exercise, rollback, and Durable Object capacity/cold-start telemetry produce the receipts above.

Until those receipts and live telemetry exist, the correct readiness statement is `local_rung: dev-proven`, `delivered_rung: undocumented`: demo-ready and locally authoring-gate-ready, but not Production-deployment-ready, Production-live, or Production-verified.
