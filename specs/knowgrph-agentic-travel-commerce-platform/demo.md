---
title: "Knowgrph Agentic Travel Commerce Platform — Demo Walkthrough"
doc_type: "Spec Demo"
schema: "kiro-spec-demo/v1"
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
demo_runtime: "Dev only — GitHub/knowgrph via npm run dev:apex"
demo_duration_target: "under 7 minutes"
requirements_baseline: ".kiro/specs/knowgrph-agentic-travel-commerce-platform/requirements.md v1.0.0"
design_baseline: ".kiro/specs/knowgrph-agentic-travel-commerce-platform/design.md v1.0.0"
tasks_baseline: ".kiro/specs/knowgrph-agentic-travel-commerce-platform/tasks.md v1.0.0"
source_specification: "joohwee/prd-tad-ard/knowgrph-agentic-travel-commerce-platform-prd-tad-adr.md v1.0.0"
governing_contracts:
  - "huijoohwee.github.io/guidelines/agentic-sdlc-guidelines.md"
  - "agentic-canvas-os/docs/START-WORKFLOW.md"
  - "agentic-canvas-os/docs/AGENTS.md"
---

# Demo Walkthrough

A presenter script for the consolidation increment, run entirely in the Dev lane. Requirement, property, and task numbers reference `requirements.md`, `design.md`, and `tasks.md`; nothing is restated here.

The through-line: a traditional OTA makes the human re-plan when a flight slips. This demo shows the platform doing it — only the causally affected legs, all or nothing, and one settlement instead of four.

## What This Demo Proves

| Observable claim | Independently verified by |
|---|---|
| A change to one leg re-plans exactly the causally downstream legs, and touches no unaffected sibling | `npm run check:affected-set` |
| The whole affected set commits or none of it does, and a rollback restores the prior state exactly | `npm run check:atomic-commit` |
| One cascade costs one settlement call, whatever the affected-set size | `npm run check:net-settlement` |
| Two concurrent agent offers can never jointly overspend one budget | `npm run check:envelope-atomicity` |
| Released funds are available to other in-flight agents with no observable staleness window | `npm run check:hold-lifecycle` |
| The re-plan surface stays legible and offline-tolerant at phone width | `npm run check:replan-surface` |

What the demo does **not** prove:

- **Nothing here is a load or performance result.** The 50 ms walk and 10 ms envelope check are asserted by measurement in checks, with recorded values. What the audience sees on screen is not the evidence.
- **The scale boundary is real.** Twenty legs and twenty edges per bundle. Beyond that, insertion is rejected with the observed count. We show that rejection on purpose (Beat 6) rather than hoping nobody asks.
- **No Prod mirror or Cloudflare surface is touched.** Everything runs against `GitHub/knowgrph`. The Prod mirror and the `airvio.co` routes are byte-identical before and after; all three deploy boundaries read `closed` for the whole session.
- **Inference is not free.** Workers AI is metered beyond its free allocation and Containers overflow is metered compute. Beat 7 says so out loud, because the ADR does.

## Preconditions

Minimum non-optional tasks that must be `verified` before the demo can run:

`1.1`, `1.2`, `2.1`, `2.3`, `4.1`, `4.2`, `5.1`, `7.1`, `8.1`, `9.1`, `9.2`, `9.3`, `11.1`, `11.3`, `11.5`, `12.1`, `12.2`, `13.1`, `13.2`, `14.1`, `15.1`, `15.2`, `16.1`, `17.1`.

Optional (`*`) tasks are not required to run the demo, but the Evidence Appendix rows they back cannot be shown without them.

Required environment configuration, **by key name only** — no value appears in this document or in the repository:

- `STRAITSX_MCP_GATEWAY_ENDPOINT`
- `STRAITSX_MCP_GATEWAY_CREDENTIAL`
- `AVALANCHE_DATA_API_ENDPOINT`
- `TELEGRAM_BOT_TOKEN`
- `EDGE_ORCHESTRATOR_ENDPOINT`
- `MODEL_LICENSE_CONFIG_URL`
- `WORKERS_AI_BINDING`
- `CONTAINERS_OLLAMA_ENDPOINT`

Startup command:

```
npm run dev:apex
```

Startup **fails closed**: if any required key is absent, the process refuses to start and names each absent key. No built-in default is applied. If you see that error on stage, read the named key aloud and set it — the refusal is the intended behavior.

Seed the 2-leg MVP bundle from task 16.1's fixture before starting: one flight leg, one downstream local-experience leg whose start time depends on arrival, one edge between them, one principal, one envelope with a known `total_budget`. Beats 2 through 4 need one more downstream leg attached to the experience, so seed a 3-leg chain and demote it mentally to the 2-leg claim if time runs short.

## Demo Script

### Beat 1 — The dependency structure is a real object

- **Presenter:** open the re-plan surface at phone width. Show the seeded bundle's legs and the edge between them.
- **Viewer sees:** legs rendered as a semantic list, each row naming its leg identifier, its committed offer, and its committed amount. The downstream leg is visibly downstream — the edge is data, not a diagram someone drew.
- **Spec claim:** Requirement 14.1, 14.2 — legible at 320 CSS px with an accessible name per row. Requirement 7.1 — two flat tables, no graph engine anywhere.
- **Say this out loud:** everything after this beat is a consequence of the platform knowing that arrow exists.

### Beat 2 — One change, only the downstream legs

- **Presenter:** delay the flight leg. Do not touch anything else.
- **Viewer sees:** the affected set appears, containing exactly the legs reachable downstream of the flight — and not the flight itself. An unrelated sibling leg, seeded deliberately, stays untouched with its original offer.
- **Spec claim:** Requirement 1.1–1.3 — the affected set equals exactly the downstream-reachable set, excluding the changed leg. Property CP-1.
- **Say this out loud:** this is the difference between "suggest an alternative for the leg that broke" and "re-plan the legs that broke *because of it*."

### Beat 3 — All of it, or none of it

- **Presenter:** let the cascade complete cleanly. Then repeat the same delay with one downstream harness rigged to reject its re-quote.
- **Viewer sees:** first run — every affected leg moves to its new offer together. Second run — nothing moves. Both legs sit on their prior offers, the rollback reason reads `requote-rejected`, and the surface labels the cascade rolled back rather than partially applied.
- **Spec claim:** Requirement 2.1–2.4 — one atomic transaction across the affected set, byte-identical restoration on rollback. Properties CP-3 and CP-4.
- **Say this out loud:** there is no partial-retry path. One rejection aborts the set, because partial retry is exactly how you end up half-booked.

### Beat 4 — One cascade, one settlement

- **Presenter:** run the delay against the 3-leg chain so the affected set has more than one leg. Open the cascade record.
- **Viewer sees:** affected-set size of 2 or 3, settlement-call count of exactly **1**, and the net amount as the signed sum of new minus prior across the whole set. Then run a cascade whose new offers net to zero: settlement-call count **0**, recorded as zero-net.
- **Spec claim:** Requirement 3.1–3.3, 3.6 — one call regardless of affected-set size, zero for a zero net, with both numbers recorded rather than inferred. Property CP-5.
- **Say this out loud:** this is the beat that makes the L4 claim literal. Drop it and this is a nicer L3.

### Beat 5 — Two agents, one budget, no overspend

- **Presenter:** submit two offers concurrently from two registered agents, each individually inside the remaining budget, together above it.
- **Viewer sees:** one reserved, one rejected with `insufficient-envelope`. Then release the reserved hold and immediately resubmit the rejected offer: it now passes, with no waiting period.
- **Spec claim:** Requirement 4.1–4.6 — check-and-reserve is one indivisible operation against true available balance. Requirement 5.3 — released funds reappear with no observable staleness window. Properties CP-6 and CP-7.
- **Say this out loud:** the old guardrail checked a number once. This one accounts for every hold in flight, which is the only reason the settlement claim in Beat 4 is true in real time rather than in a script.

### Beat 6 — The scale boundary, shown rather than hidden

- **Presenter:** attempt to insert a 21st leg. Then attempt an edge that would introduce a cycle.
- **Viewer sees:** both rejected, each naming the limit and the observed count, and the bundle unchanged. `check:scale-boundary` reads the limit from the exported constant rather than a number the test invented.
- **Spec claim:** Requirement 7.3–7.5, 7.8 — the boundary is a declared, readable constant, and crossing it is a typed rejection. Property CP-13.
- **Say this out loud:** "no graph database" is an honest constraint because its limit is written down. If we ever hit it, the fix is sharding the flat tables, not adopting a graph engine that is still the wrong tool at twenty rows.

### Beat 7 — Cost, caches, and the one honest non-zero

- **Presenter:** show the cascade's Cost_Log entry. Then re-run the same delay inside the offer cache TTL window.
- **Viewer sees:** orchestration prompt and completion tokens at `0`, dollar cost `0.00`. On the second run, fewer Discovery dispatches, with the with-cache and without-cache counts both shown. Then the inference log: selected path, model identifier, declared license, recorded cost — and the recorded note that neither Workers AI beyond its free allocation nor Containers overflow is free.
- **Spec claim:** Requirement 10.2, 9.6 — model-free orchestration with recorded dispatch reduction. Requirement 11.6, 11.8 — per-call license and cost recorded, neither path claimed free. Property CP-14.
- **Say this out loud:** the cascade is arithmetic. It costs storage reads, not tokens. Inference costs money and we are not going to pretend otherwise.

### Beat 8 — Offline on a phone

- **Presenter:** stay at 360 CSS px, enable airplane mode, reload the surface, then reconnect.
- **Viewer sees:** the last projected cascade renders immediately with no network round trip, a not-current indicator showing elapsed time since sync, no horizontal overflow, then convergence and the indicator disappearing on reconnect. A rolled-back cascade still reads rolled back throughout.
- **Spec claim:** Requirement 14.4–14.7 — local replica first, edge as convergence peer, rolled-back never rendered as committed. Requirement 8.7 — hibernatable WebSockets let the idle Durable Object release memory without losing state.

## Evidence Appendix

| Beat | Invocable check | Properties | Where recorded evidence lands |
|---|---|---|---|
| 1 | `npm run check:replan-surface`, `npm run check:scale-boundary` | CP-13 | Rendered row set, accessible names, overflow assertion; static scan result for zero graph engine; exit status and seed |
| 2 | `npm run check:affected-set` | CP-1, CP-2 | Session_Log entry per cascade with `bundle_id`, changed leg, affected identifiers in traversal order, outcome; measured walk latency; exit status, counts, seed |
| 3 | `npm run check:atomic-commit` | CP-3, CP-4, CP-10 | Pre- and post-cascade snapshot digests; rollback reason; mixed-state assertion count; exit status, seed |
| 4 | `npm run check:net-settlement` | CP-5 | Per-cascade affected-set size, settlement-call count, net amount, gateway caller identity; exit status, seed |
| 5 | `npm run check:envelope-atomicity`, `npm run check:hold-lifecycle` | CP-6, CP-7, CP-8 | Available-balance-at-check per accepted offer, hold state transitions, conservation identity after each transition, measured reserve latency; exit status, seed |
| 6 | `npm run check:scale-boundary` | CP-13, CP-15 | Rejection reasons with observed counts; declared-limit constant read; topological order comparison; exit status, seed |
| 7 | `npm run check:cost-observability`, `npm run check:edge-cache`, `npm run check:inference-license` | CP-11, CP-12, CP-14, CP-16 | One Cost_Log entry per cascade; dispatch counts with and without cache; per-call path, model, license, cost; archive write-once result; exit status, seed |
| 8 | `npm run check:replan-surface`, `npm run check:storage-placement` | — | Offline retention and `sinceLastSyncMs`; convergence timing; hibernation state-restore assertion; exit status, seed |

Every check exits non-zero on failure and prints its recorded counts, measured values, and property run seeds, so a failing beat is reproducible from the printed seed alone.

## Cost And Token Economics Talking Point

BFS, atomic commit, rollback, net computation, hold arithmetic, and topological order maintenance are deterministic. Both new pipelines carry a `$0.00/call` token budget. The only token spend in a cascade belongs to the reused Discovery Harnesses re-quoting downstream legs — and the offer cache cuts how often that happens, because most re-plans re-query the same route or item within seconds of the original quote.

Marginal infrastructure: zero new vendor, zero new storage category. Durable Objects and D1 were already provisioned; KV, Cache API, and R2 sit inside free tiers at MVP scale, and R2 has no egress fee to Workers.

One-sentence version: **re-planning a four-leg cascade costs four storage reads and one settlement call, because the reasoning is arithmetic and the settlement is netted before it is sent.**

## Failure Modes During Live Demo

| Live failure | Observed symptom | Why it is correct fail-closed behavior | Recovery line |
|---|---|---|---|
| Envelope ledger unreachable | Every offer rejected with `envelope-unavailable`; zero holds reserved; nothing reaches a downstream component | The balance is an authoritative read, not an optimistic assumption. Unreadable means unauthorized, not "probably fine" | "It can't confirm the remaining budget, so it approves nothing." |
| Stale cached balance | Outcome identical to the uncached path; the divergent entry is invalidated | The KV cache returns `mustConfirm: true`, so a commit decision structurally cannot be reached without confirming against the ledger | "The cache is a speed-up, never an authority. The type signature enforces that, not a code comment." |
| Offer cache entry past TTL, revalidation incomplete | Cascade rolls back with `stale-offer-cache-entry`; zero commit; zero settlement | Committing at a price whose quote expired is the one thing a cache could plausibly buy you, so it is refused explicitly | "The quote aged out mid-cascade. It refuses to buy at a price it can't currently confirm." |
| One downstream re-quote times out | Whole affected set aborts with `cascade-timeout`; snapshot restored; holds released | A cascade is one bounded pass with a stated cap. A partial retry would be how a mixed state gets created | "One leg stalled, so the whole re-plan backs out. Nothing half-changed." |
| Settlement call fails | Commit rolled back, holds released, `settlement-failed` recorded; zero cards issued | The commit and the settlement are one atomic outcome from the principal's point of view | "The money move failed, so the booking change unwound with it." |
| Archive write fails after commit | Commit **retained**; `archive-deferred` recorded | Provenance is bookkeeping; the commit is the principal's real booking state. Rolling back a correct booking for a failed archival write would trade a real outcome for tidy records | "The booking stands. Only the archive copy is deferred, and it's recorded as deferred." |
| Cycle in the seeded fixture | Cascade rejected with `cyclic-dependency`; nothing mutated | A cycle means "A depends on B depends on A", which has no correct re-plan order. Failing beats looping | "The fixture has a circular dependency. It refuses to guess an order. Let me reseed." |
| Model license config unreachable | Every inference route rejected with `license-configuration-unavailable` | A license filter that opens under error is decoration, not a filter | "It can't verify the model licenses, so it routes to nothing." |
| One required config key missing | Startup refuses and names the absent key | No built-in default exists for any provider endpoint or credential, so a partially configured run cannot start | "It won't start half-configured. It's naming the exact key to set." |

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
- **Prod mirror or Cloudflare deployment?** No. All three boundaries stay `closed` for the whole session, and both surfaces are byte-identical before and after.

## Known Gaps Called Out Live

Three gaps, all inherited from the source specification's own Open Questions. Each is a **blocked** task, not a skipped one, and each is pending exactly one operator decision.

**1. No materiality threshold on the trigger (task 18.1, Requirement 1.10).** Every mutation event currently triggers a cascade. The source specification itself warns that an overly sensitive trigger could cascade on noise. We have not invented a threshold, because a check written today could only assert against a number it had itself made up. `check:affected-set` reports Requirement 1.10 as not runtime-ready with cause "operator decision absent."

**2. Rollback notifies nobody yet (task 18.2, Requirement 6.9).** A rolled-back cascade is written to the Session_Log and emits no notification. Whether the principal is told synchronously or through the existing Notification Dispatcher queue is an open decision. Either answer is additive, since the rollback record already exists.

**3. `available_balance` is server-side only (task 18.3, Requirement 5.10).** Whether the Shopper Client should see its own remaining budget is undecided, and it determines whether the KV-cached read needs any client-facing auth scope at all. We expose it server-side only, so choosing transparency later adds a scope rather than reworking one.

Say it plainly on stage: three decisions are open, all three are recorded as blocked, and none of them has been closed by picking a plausible default to make a green check appear.
