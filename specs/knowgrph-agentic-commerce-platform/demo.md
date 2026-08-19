---
title: "Knowgrph Agentic Commerce Platform — Demo Walkthrough"
doc_type: "Spec Demo"
schema: "kiro-spec-demo/v1"
version: "0.1.0"
date: "2026-08-19"
lang: "en-US"
frontmatter_contract: "required"
spec_type: "feature"
workflow_type: "requirements-first"
feature_name: "knowgrph-agentic-commerce-platform"
owner: "Solo Founder / AI Orchestrator"
lane: "authoring"
local_rung: "spec-complete"
delivered_rung: "undocumented"
deploy_boundary: "dev-only"
demo_runtime: "Dev only — GitHub/knowgrph via npm run dev:apex"
demo_duration_target: "under 6 minutes"
requirements_baseline: ".kiro/specs/knowgrph-agentic-commerce-platform/requirements.md v0.1.0"
design_baseline: ".kiro/specs/knowgrph-agentic-commerce-platform/design.md v0.1.0"
tasks_baseline: ".kiro/specs/knowgrph-agentic-commerce-platform/tasks.md v0.1.0"
source_specification: "joohwee/prd-tad-ard/knowgrph-agentic-commerce-platform-prd-tad-adr.md v0.1.0"
governing_contracts:
  - "huijoohwee.github.io/guidelines/agentic-sdlc-guidelines.md"
  - "agentic-canvas-os/docs/START-WORKFLOW.md"
  - "agentic-canvas-os/docs/AGENTS.md"
---

# Demo Walkthrough

This is a presenter script for the Phase 1 increment, run entirely in the Dev lane. Requirement and property numbers reference `requirements.md` and `design.md`; nothing is restated here.

## What This Demo Proves

| Observable claim | Independently verified by |
|---|---|
| One free-text interface routes to two different verticals, one dispatch each, with no per-vertical app | `npm run check:routing` |
| The same budget guardrail and human-confirmation gate applies whichever agent produced the offer, and outcomes are comparable per agent identifier | `npm run check:payment-ordering` |
| An unregistered, unmatched, or ambiguous agent gets zero Discovery dispatches and zero payment-adjacent calls | `npm run check:registration-gate`, `npm run check:routing` |
| The operator surface stays usable offline at mobile width and converges on reconnect without losing a queued change | `npm run check:offline-surfaces` |

What the demo does **not** prove:

- **Registration is `declared-and-present` only.** It is a schema-conformance and allowlist-membership fact. There is no runtime sandboxing of registered-agent behavior. The trust status column shows that literal term for exactly this reason.
- **No Prod mirror or Cloudflare surface is touched.** Everything runs against `GitHub/knowgrph`. The Prod mirror and `airvio.co` routes are byte-identical before and after; the boundary is closed for the whole session.
- **Nothing here is a performance or load result.** Latency and projection bounds are asserted by checks, not by what the audience sees on screen.

## Preconditions

Minimum non-optional tasks that must be `verified` before the demo can run:

`1.1`, `1.2`, `2.1`, `2.3`, `2.5`, `4.1`, `5.1`, `5.4`, `5.7`, `7.1`, `7.2`, `7.3`, `8.1`, `8.2`, `10.1`, `10.2`, `11.1`, `11.2`, `12.1`, `13.1`.

Optional (`*`) tasks are not required to run the demo, but the Evidence Appendix rows they back cannot be shown without them.

Required environment configuration, **by key name only** — no value appears in this document or in the repository:

- `STRAITSX_MCP_GATEWAY_ENDPOINT`
- `STRAITSX_MCP_GATEWAY_CREDENTIAL`
- `AVALANCHE_DATA_API_ENDPOINT`
- `TELEGRAM_BOT_TOKEN`
- `INVOCATION_SURFACE_CONTRACT_SCHEMA_URL`
- `EDGE_ORCHESTRATOR_ENDPOINT`

Startup command:

```
npm run dev:apex
```

Startup **fails closed**: if any required key is absent, the process refuses to start and names each absent key. No built-in default is applied. If you see that error on stage, read the named key aloud and set it — that refusal is the intended behavior, not a broken build.

Seed both harnesses as registered agents with distinct declared categories before starting (task 12.1 fixture). A mis-seeded duplicate category produces the ambiguity path, which is correct but not the beat you want in slot 2.

## Demo Script

### Beat 1 — Operator registry canvas

- **Presenter:** open the operator surface under Operator_Scope.
- **Viewer sees:** two rows, one per registered agent, each showing agent identifier, declared category, declared tool allowlist, and trust status rendered as the literal string `declared-and-present`.
- **Spec claim:** Requirement 3.2 — the operator audits what is live without reading code. The literal status term is Requirement 1.10: registration is declarative, and the surface is not permitted to overstate it.

### Beat 2 — One interface, two verticals

- **Presenter:** type a flight-shaped free-text request. Then, in the same input, a shopping-shaped request.
- **Viewer sees:** the first resolves to the Flight Discovery Harness, the second to the Shopping Discovery Harness. One dispatch each. Same input box, same session, no app switch.
- **Spec claim:** Requirement 2.1, 2.2 — deterministic category routing to exactly one registered agent. Property 1.

### Beat 3 — Guardrail and confirmation parity

- **Presenter:** submit an over-budget offer from the flight harness, then the equivalent over-budget offer from the shopping harness. Open the Session_Log view.
- **Viewer sees:** both stopped by the same gate with the same outcome. Each `gate-fail` entry carries a non-empty registered agent identifier, so the two outcomes are directly comparable per agent.
- **Spec claim:** Requirement 4.4 and 4.6 — the input set supplied to Guardrail_Gate is identical for every agent identifier, and outcomes are retrievable per agent. Properties 3 and 4.

### Beat 4 — Fail-closed routing

- **Presenter:** submit a request in a category no agent declares. Then de-register one harness and immediately submit a request that previously routed to it.
- **Viewer sees:** `No_Match_Result` both times — `unmatched-category` for the first, `agent-not-registered` for the second. Zero Discovery dispatches. Exactly one routing entry per intent in the Session_Log.
- **Spec claim:** Requirement 2.3 and 1.1 — membership is re-read at the moment of the dispatch decision, not trusted from a warm index. Properties 2 and 11.
- **Say this out loud:** this is the boring beat and it is the one that matters. Nothing happened, on purpose, on a payment-adjacent path.

### Beat 5 — Issuance scoped by card

- **Presenter:** take an in-budget offer through gate pass, then human confirmation.
- **Viewer sees:** exactly one disposable card issued for that offer identifier. The recorded tool-call log names Issuance_Service as the caller of every payment-path call.
- **Spec claim:** Requirement 4.2 and 5.1 — gate pass before human confirm before issuance, at most one card per offer, and Issuance_Service is the only component holding a configured StraitsX client. Properties 3 and 10.

### Beat 6 — Offline resilience on mobile

- **Presenter:** narrow the operator surface to 360 CSS px, enable airplane mode, reload the surface, queue a registration change, then reconnect.
- **Viewer sees:** last locally held state within a second, a persistent not-current indicator with elapsed-since-sync, no horizontal overflow, the queued change retained while offline, then CRDT convergence and the indicator disappearing after reconnect.
- **Spec claim:** Requirement 9.1, 9.2, 9.4, 9.5 — local-first read, bounded retry, record-order submission. Properties 7 and 13.

## Evidence Appendix

| Beat | Invocable check | Properties | Where recorded evidence lands |
|---|---|---|---|
| 1 | `npm run check:registry-canvas` | 6, 7 | Projection revision and rendered row set; check exit status, rendered-row and not-declared counts, run seed |
| 2 | `npm run check:routing` | 1, 11 | Session_Log `routing` entries (intentId, categoryAsReceived, agentId); dispatch count per intent; exit status, counts, seed |
| 3 | `npm run check:payment-ordering` | 3, 4 | Session_Log `gate-pass` / `gate-fail` entries carrying agentId; parity comparison counts; exit status, seed |
| 4 | `npm run check:routing`, `npm run check:registration-gate` | 1, 2, 11 | Session_Log `routing` entries with No_Match reason; zero-dispatch assertion counts; exit status, seed |
| 5 | `npm run check:payment-ordering`, `npm run check:payment-isolation` | 3, 10 | Session_Log `human-confirm`, `issuance`, `fail-closed` entries; caller-identity tool-call log; exit status, per-offer issuance counts, seed |
| 6 | `npm run check:offline-surfaces` | 7, 13 | Pending-queue order record, `sinceLastSyncMs`, convergence timing; exit status, submitted-vs-recorded order counts, seed |

Every check exits non-zero on failure and prints its recorded counts and the run seed, so a failing beat is reproducible from the printed seed alone.

## Cost And Token Economics Talking Point

Routing, validation, and registry projection are model-free: string normalization, schema evaluation, and CRDT projection. Intent Parser is the only model call in the path and it existed before this increment. Marginal cost of adding a registered agent is one Agent_Definition record plus one routing entry — no token cost, no vendor onboarding, no new infrastructure service.

One-sentence version: **this increment adds a second vertical for the price of one database row, because routing never calls a model.**

## Failure Modes During Live Demo

| Live failure | Observed symptom | Why it is correct fail-closed behavior | Recovery line |
|---|---|---|---|
| Schema source unreachable | Registration submission returns a reject carrying a `schema-unavailable` violation within 5 seconds; no routing entry appears | The Invocation Surface Contract schema is externally owned and never cached locally, so an unavailable source cannot be papered over with a stale copy | "The schema lives elsewhere and we don't keep a second copy. No copy, no registration." |
| Registration state unreadable | Every request returns `No_Match_Result` with `registration-state-unavailable`; zero dispatches; the submitted intent is unmodified | Routing authorization is a read, not a cached assumption. Unreadable state means unauthorized, not "probably fine" | "It can't confirm who's registered, so it dispatches to nobody." |
| Ambiguous category from a mis-seeded fixture | `No_Match_Result` with `ambiguous-category`; zero dispatches | Ambiguity is never resolved by tie-break, because that would make routing depend on invisible registration order | "Two agents claim the same category, so it refuses to guess. Let me reseed." |
| Offline at issuance | Zero cards issued, a connectivity-required message, issuance inputs preserved locally for resubmission | Issuance is a real external payment call and is never speculatively queued | "Card issuance needs the network. Nothing is lost — the inputs are held for resubmission." |
| One required config key missing | Startup refuses and names the absent key | No built-in default exists for any provider endpoint or credential, so a partially configured run can never start | "It won't start half-configured. It's telling us exactly which key to set." |

## Not In This Demo

Questions a viewer will reasonably ask, with the honest answer:

- **Runtime sandboxing of registered agents?** No. Declared allowlist membership only. Phase 2.
- **Public self-serve registration?** No. Registration is operator-driven in this increment.
- **On-chain trust or reputation attestation?** No. `declared-and-present` is the whole claim today.
- **Marketplace fee or take rate?** Deliberately undecided. Nothing in the design assumes either.
- **A third vertical?** No. Two harnesses are enough to prove the pattern; a third proves nothing new.
- **Multi-agent split-pay or group wallets?** No. Phase 4.
- **Prod mirror or Cloudflare deployment?** No. Both boundaries stay closed for this entire session, and both surfaces are byte-identical before and after.

## Known Gap Called Out Live

Task 14.1 is **blocked**, not skipped. Requirement 4.8's confirmation-window expiry cannot be independently checked yet: the window is owned by the reused Issuance_Service and is not exposed as a named readable constant. A check written today could only assert against a value it had itself hardcoded, which is a test asserting against itself.

The gap is pending one operator decision — whether Issuance_Service may expose a read-only constant such as `CONFIRMATION_WINDOW_MS`. Our reading is that a read-only export is additive and alters no operation, parameter, or return value, so it does not conflict with Requirement 4.5 or Non-Goal 12. Until that is confirmed, `npm run check:payment-ordering` reports Requirement 4.8 as not runtime-ready with cause "required named check absent."

Say it plainly on stage: the expiry path is designed and unverified, and we are not inventing a window value to make a green check appear.
