---
title: "AgenticGraph Commerce Platform — Design Annex B: Criterion Coverage"
doc_type: "Spec Design Annex"
schema: "kiro-spec-design-annex/v1"
version: "0.1.0"
date: "2026-08-28"
lang: "en-US"
frontmatter_contract: "required"
feature_name: "agentic-graph-commerce-platform"
parent_document: "design.md"
requirements_baseline: ".kiro/specs/agentic-graph-commerce-platform/requirements.md v0.1.0"
criteria_total: 128
---

# Annex B — Criterion Coverage Map

One row per acceptance criterion, all 128. **Element #** refers to the numbered rows in the Components and Interfaces table in `design.md`. **Verification** names the property (CP-n) or the check kind. Criteria covered only partially are marked `PARTIAL` and explained in the last section.

## Requirement 1 — Terminology Supersession (10)

| Criterion | Element # | File | Verification |
|---|---|---|---|
| 1.1 | 1, 2, 5 | `config/terminology-register.json`, `scripts/validate-terminology-register.ts` | Repository scan, 1 execution |
| 1.2 | 1 | `config/terminology-register.json` | Schema assertion per `externally-owned` entry |
| 1.3 | 1 | `config/terminology-register.json` | Schema assertion per `historical-record` entry |
| 1.4 | 1, 2, 5 | register + checker + prose renames | Repository scan, residual set empty |
| 1.5 `PARTIAL` | 4 | `src/invocation/catalog.ts` | Export-surface assertion (one endpoint, one tool) + register cross-check |
| 1.6 | 3 | `src/shared/terminology-guard.ts` | CP-1 |
| 1.7 | 1, 9 | register + `AG_TAKE_RATE_BASIS_POINTS` and siblings | Key inventory scan over wrangler configs and generated env types |
| 1.8 | 1, 2, 5 | whole repository | `npm run check`, passing-check-name diff against baseline record |
| 1.9 | 2 | `scripts/validate-terminology-register.ts` | CP-1 determinism arm + injected-occurrence property |
| 1.10 | 1 | register | Scan asserting one dictionary source, zero alias module |

## Requirement 2 — Tolerant Upstream Convergence (11)

| Criterion | Element # | File | Verification |
|---|---|---|---|
| 2.1 | 6, 7 | `src/core/convergence-evaluator.ts` | CP-2 (exact-set arm) |
| 2.2 | 6 | same | CP-2 (surplus arm), recorded surplus names asserted |
| 2.3 | 6 | same | CP-2 (deficit arm), CP-4 |
| 2.4 | 8 | `src/core/index.ts` | Integration, 1 example unretrievable + 1 incomplete |
| 2.5 | 6 | same | CP-4 (accept region) |
| 2.6 | 6 | same | CP-4 (block region, both values recorded) |
| 2.7 | 6 | same | CP-2 (metamorphic surplus-field arm), count asserted |
| 2.8 | 6, 7 | evaluator + `readUpstreamEvidencePin` | CP-4 (identity and digest mutation arms) |
| 2.9 | 8 | `src/core/index.ts`, `src/edge/index.ts` | Integration, 1 example per verdict combination |
| 2.10 | 6 | same | CP-3 + one latency measurement (≤200 ms) in the domain lane |
| 2.11 | 8 | readiness composition | Integration, 1 example per field-presence case |

## Requirement 3 — First Settled Markup (11)

| Criterion | Element # | File | Verification |
|---|---|---|---|
| 3.1 | 9, 11 | `src/core/take-rate.ts`, `checkout-session.ts` | CP-5 + one 500 ms measurement in the workers lane |
| 3.2 | 10, 11 | `revenue-ledger.ts` | CP-6 |
| 3.3 | 9, 12 | `take-rate.ts`, `src/core/index.ts` | Repository scan (zero numeric rate literal) + `AG_` key assertion |
| 3.4 | 9 | `take-rate.ts` | CP-5 |
| 3.5 | 10 | `revenue-ledger.ts` | CP-6 |
| 3.6 | 11 | `checkout-session.ts` | CP-5 (non-interference arm) |
| 3.7 | 12 | `src/core/index.ts` | Integration, 1 example per invalid class (absent, non-numeric, negative, zero, >1000 bp) |
| 3.8 | 11 | `checkout-session.ts` | Integration, 1 example with a forced append fault |
| 3.9 | 10 | `revenue-ledger.ts` | CP-7 (field completeness arm) |
| 3.10 | 10 | `revenue-ledger.ts` | CP-7 (order and recomputed-sum arms) |
| 3.11 `PARTIAL` | 10 | `revenue-ledger.ts` | Integration, 1 example asserting the reported count field |

## Requirement 4 — White-Label Template Pack (12)

| Criterion | Element # | File | Verification |
|---|---|---|---|
| 4.1 | 13, 16 | `theme-manifest.ts`, `dashboard.ts` | Module-count scan + default-manifest render equality with the baseline console |
| 4.2 | 13 | `theme-manifest.ts` | CP-8 + one 5 s measurement |
| 4.3 | 14, 15 | `theme-deployment.ts` | CP-9 (reject arm) |
| 4.4 | 13 | `theme-manifest.ts` | CP-8 (defaulted-field arm) |
| 4.5 | 16, 17 | `merchant-catalog.ts`, `dashboard.ts` | CP-9 (containment arm) |
| 4.6 | 16 | `dashboard.ts` | Browser measurement, 5 cold loads at 360 px / 1.6 Mbps, median reported |
| 4.7 | 16 | `dashboard.ts` | DOM + static assertion on default and one themed render |
| 4.8 | 16, 17 | edge routes + `checkout-session.ts` | Integration, 1 example asserting the shared core route |
| 4.9 | 14 | `theme-deployment.ts` | Integration, 1 example with a failing asset fetch |
| 4.10 | 15 | `theme-deployment-store.ts` | Integration, 1 example asserting each recorded field |
| 4.11 | 13–17 | `wrangler.*.jsonc`, `package.json` | Binding + dependency inventory diff against the baseline category set |
| 4.12 | 15 | deployment record | Integration, 1 example asserting capability is not reported as demand |

## Requirement 5 — Environment Chain and Deploy Boundaries (10)

| Criterion | Element # | File | Verification |
|---|---|---|---|
| 5.1 | 18 | `package.json` | Script inventory assertion (exactly `dev`, `dev:apex`) |
| 5.2 | 18 | `package.json`, wrangler dev configs | Integration, 1 example with an egress recorder around startup |
| 5.3 | 19 | `wrangler.edge.jsonc` | Configuration assertion, exactly one Production route |
| 5.4 | 20 | controller config + docs | Declaration assertion (mirror is generated output) |
| 5.5 | 20 | `scripts/release-controller.ts`, `package.json` | Script-surface scan + boundary register assertion |
| 5.6 | 20 | controller | CP-19-style admission property specialised to authorization records (missing/expired element arms) |
| 5.7 | 20 | controller | Property over seal/advance sequences: a retired candidate is always refused |
| 5.8 | 20 | controller | Property: refusal exactly when no rollback disposition is recorded |
| 5.9 | 8, 16 | `src/edge/index.ts`, `dashboard.ts` | Workers-lane example per surface + secret-shaped-field scan |
| 5.10 | 8 | readiness composition | Integration, 1 example per readiness combination |

## Requirement 6 — One Invocation Surface (8)

| Criterion | Element # | File | Verification |
|---|---|---|---|
| 6.1 | 21, 22 | `src/invocation/*`, `capability-token-map.json` | Import-graph and module scan (one registry) + CP-10 |
| 6.2 | 21 | `capability-token-map.json` | Coverage scan against the hydrated catalog (one `/`, ≥1 `#`, ≥1 `@` per action) |
| 6.3 | 22 | `src/core/index.ts` | CP-10 (five-field arm) |
| 6.4 | 22 | `src/core/index.ts` | CP-10 (unresolved arm) |
| 6.5 | 22 | `src/core/index.ts` | Integration, 1 example with a failing docs MCP binding |
| 6.6 | 22 | `src/core/index.ts` | CP-11 + one 50 ms measurement + import scan for model clients |
| 6.7 | 23 | `src/edge/index.ts` (`/mcp`, `/mcp/operator`) | Integration, 1 example per capability comparing MCP and HTTP reachability |
| 6.8 | 22 | `src/core/index.ts` | CP-11 (revision-transition arm) |

## Requirement 7 — WebMCP Storefront Tool Surface (9)

| Criterion | Element # | File | Verification |
|---|---|---|---|
| 7.1 | 25 | `src/edge/client/webmcp-tools.ts` | Browser integration, 1 example on a capable engine (3 tool kinds, 16 cap, 2000 ms) |
| 7.2 | 24, 25 | `storefront-actions.ts` | Import-graph scan + route inventory diff (zero tool-only endpoint) |
| 7.3 | 25 | `webmcp-tools.ts` | Browser integration, 1 example with the registration API removed |
| 7.4 | 24, 25 | shared client actions | CP-12 |
| 7.5 | 25 | `webmcp-tools.ts` | CP-13 (digest stability arm) |
| 7.6 | 25 | `webmcp-tools.ts` | CP-13 (drift refusal arm) |
| 7.7 | 25 | `webmcp-tools.ts` | CP-12 (credential-absence arm) |
| 7.8 | 36 | boundary register + `wrangler.sandbox.jsonc` | Boundary state assertion + recorded browser support set |
| 7.9 | — | `design.md` frontmatter, register | Documentation assertion (origin-trial stage, rung reported) |

## Requirement 8 — Neutral Aggregation Routing (10)

| Criterion | Element # | File | Verification |
|---|---|---|---|
| 8.1 | 26, 27, 28 | `selection-policy.ts`, `exclusive-category-router.ts` | CP-14, CP-15 |
| 8.2 | 26, 28 | `agent-registry.ts` route record | CP-15 (recording arm) |
| 8.3 | 27, 29 | `intent-route.ts` | CP-14 |
| 8.4 | 29 | `intent-route.ts` | CP-14 (fallback arm, injected clock) |
| 8.5 | 29 | `intent-route.ts` | CP-14 (exhaustion arm) |
| 8.6 | 27 | `exclusive-category-router.ts` | CP-14 (no-match arm) |
| 8.7 | 26 | `selection-policy.ts` | CP-15 + one 200 ms measurement + import scan |
| 8.8 | 30 | `public-catalog.ts` | CP-16 (field-subset arm) |
| 8.9 | 30 | `public-catalog.ts` | Integration, 1 example at 500 agents (≤5 s) |
| 8.10 | 30 | `public-catalog.ts` | CP-16 (set-equality arm) |

## Requirement 9 — Local-First, Offline-First, Multi-Device (10)

| Criterion | Element # | File | Verification |
|---|---|---|---|
| 9.1 | 16, 31 | `local-store.ts`, `dashboard.ts` | Browser integration, 1 example with the network disabled (indicator ≤1 s) |
| 9.2 | 31 | `local-store.ts` | CP-18 (capacity and retention arms) |
| 9.3 | 31 | `local-store.ts` | CP-18 (order and acknowledgement arms) + one 5 s measurement |
| 9.4 | 32 | `sync-merge.ts` | CP-17 |
| 9.5 | 31 | `local-store.ts` | Browser integration, 1 example asserting zero payment-adjacent call |
| 9.6 | 16 | `dashboard.ts` | Browser integration, 1 example per primary surface at 360 px |
| 9.7 | 33 | `authoring-claim.ts` | CP-19 |
| 9.8 | 33 | `authoring-claim.ts` | CP-19 (disjointness arm) |
| 9.9 | 33 | `authoring-claim.ts` | CP-19 (expiry arm, injected clock) |
| 9.10 | 31–33 | `wrangler.*.jsonc`, `package.json` | Binding + dependency inventory |

## Requirement 10 — Held-Offer Change Detection (9)

| Criterion | Element # | File | Verification |
|---|---|---|---|
| 10.1 | 34, 35 | `offer-watch.ts`, `checkout-session.ts` alarm | Integration, 1 example with a controlled alarm clock |
| 10.2 | 34 | `offer-watch.ts` | CP-20 (event-content arm) |
| 10.3 | 35 | `checkout-session.ts` | CP-21 |
| 10.4 | 35 | `checkout-session.ts` | CP-21 (inactive-agent arm) |
| 10.5 | 34 | `offer-watch.ts` | Integration, 1 example per stop reason |
| 10.6 | 34 | `offer-watch.ts` | Integration, 1 example with injected observation faults |
| 10.7 | 34 | `offer-watch.ts` | CP-20 (idempotence arm) |
| 10.8 | 34 | `offer-watch.ts` | Import scan + per-observation call-count assertion |
| 10.9 | — | `design.md`, register | Documentation assertion (deferral recorded) |

## Requirement 11 — Isolated Execution (8)

| Criterion | Element # | File | Verification |
|---|---|---|---|
| 11.1 | 36 | `src/sandbox/executor.ts` | Integration, 1 example per completed and failed result |
| 11.2 | 36 | `executor.ts` | CP-22 (recording arm) |
| 11.3 | 36 | `executor.ts` | CP-22 (allowlist arm) |
| 11.4 | 36 | `executor.ts`, `wrangler.sandbox.jsonc` | Integration, 1 example per exceeded limit |
| 11.5 | 36 | `executor.ts` | CP-23 (record completeness arm) |
| 11.6 | 36 | `wrangler.sandbox.jsonc` bindings | CP-22 (capability-absence arm) |
| 11.7 | 36 | `executor.ts` | Integration, 1 example with a compressed lifetime |
| 11.8 | 36 | `executor.ts` | Integration, 1 example with provisioning disabled + rung assertion |

## Requirement 12 — Bounded Agent Merge Automation (9)

| Criterion | Element # | File | Verification |
|---|---|---|---|
| 12.1 | 37 | `scripts/merge-agent/review.ts` | Process assertion, 1 example per outcome branch |
| 12.2 | 37 | `scripts/merge-agent/repair.ts` | Process assertion, 1 example per attempt-count path |
| 12.3 | 37 | `scripts/merge-agent/conflict.ts` | CP-24 |
| 12.4 | 37 | `repair.ts` | Process assertion, 1 example per branch |
| 12.5 | 37 | `config/merge-agent-bounds.json` | CP-24 (bound-breach arm) |
| 12.6 | 37 | `scripts/merge-agent/index.ts` | Process assertion, 1 example |
| 12.7 | 37 | `scripts/merge-agent/*` | CP-24 (forbidden-operation arm) |
| 12.8 | 33, 37 | `authoring-claim.ts` + merge agent | CP-24 (lane arm) + CP-19 instantiation |
| 12.9 | 37 | merge agent action record | CP-24 (append-only arm) |

## Requirement 13 — Execution Evidence and Demo Walkthrough (11)

| Criterion | Element # | File | Verification |
|---|---|---|---|
| 13.1 | 40 | `tasks.md`, `config/task-bounds.schema.json` | Task-document scan |
| 13.2 | 40 | same | Task-document schema scan (five positive-integer bounds + halting signal) |
| 13.3 | 38 | `scripts/evidence-reference.ts` | Process assertion, 1 example per verdict record |
| 13.4 | 38 | same | Process assertion, 1 example (self-graded rejection) |
| 13.5 | 38 | `package.json` `check` + named checks | Process gate, 1 execution per task |
| 13.6 | 38 | `evidence-reference.ts` | Property over generated check-result sets: emitted set equals passing subset |
| 13.7 | 39 | `scripts/validate-authored-limits.ts` | Line-count scan over the authored file set |
| 13.8 | 39 | same | Path, credential, and account-identifier scan |
| 13.9 `PARTIAL` | — | `demo.md` (tasks phase) | Operator-run walkthrough, 1 execution + command-target scan |
| 13.10 | — | `demo.md` | Document completeness scan |
| 13.11 | 38 | `evidence-reference.ts` | Property over reference and finding sets: rung is a pure function, no advance while a finding is open |

## Non-Property Check Design — Consolidated

Kinds, mapped to the exact command that runs them. Every row is a single or prescribed-count execution, never a randomized property.

| Kind | Criteria | Command | Iterations |
|---|---|---|---|
| Repository scan | 1.1–1.4, 1.7, 1.10, 3.3, 4.1, 4.11, 6.1, 6.2, 7.2, 9.10, 10.8, 13.7, 13.8 | `npm run check:terminology`, `npm run check:authored-limits`, `npm run check:invocation-surface` | 1 |
| Aggregate lane diff | 1.8, 13.5 | `npm run check` | 1 per task |
| Provider integration (`workerd` + fake services) | 2.4, 2.9, 2.11, 3.7, 3.8, 4.3 reject example, 4.8–4.10, 4.12, 5.2, 5.10, 6.5, 6.7, 8.9, 10.1, 10.5, 10.6, 11.1, 11.4, 11.7, 11.8 | `npm run test:workers` | 1–2 per case |
| Configuration assertion | 5.1, 5.3, 5.4, 5.5, 7.8, 7.9, 10.9 | `npm run check:deploy-boundary` | 1 |
| Response-shape assertion | 5.9, 3.11 | `npm run test:workers` | 1 per surface |
| DOM / static markup assertion | 4.7 | `npm run check:browser` | 1 default + 1 themed |
| Browser measurement | 4.6, 7.1, 7.3, 9.1, 9.5, 9.6 | `npm run check:browser` | 5 cold loads for 4.6, 1 each otherwise |
| Latency measurement | 2.10, 3.1, 6.6, 8.7 | lane of the owning property | 1 recorded median |
| Process assertion | 12.1, 12.2, 12.4, 12.6, 13.1–13.5, 13.10 | `npm run check:evidence`, `npm run check:merge-agent` | 1 per path |
| Operator walkthrough | 13.9 | `demo.md` steps, Dev lane only | 1 |

## Partial Coverage — Stated, Not Asserted

**1.5 — one endpoint and one tool identity in superseding form.** Satisfiable only for repository-owned identity. The endpoint address `https://airvio.co/knowgrph/control-plane/mcp` and the tool name `knowgrph.agentic_canvas_os.docs.invoke` in `src/invocation/catalog.ts` are owned by the upstream Agentic Canvas OS docs MCP service. Requirement 1.10 dispositions them `externally-owned` and forbids an alias layer, so the superseding-form half of 1.5 cannot be reached from this repository. Coverage: singularity is verified, superseding form is verified for every repository-owned identity, and the two upstream values are recorded as blocked on an upstream rename.

**3.11 — external-principal demand evidence.** The ledger records `agent_id`, which identifies the registered agent, not an external payer principal. The design can report the count of distinct agent identifiers with two or more settlements; it cannot distinguish an external principal from the Platform Operator without a payer identity the baseline does not carry. Coverage: the reporting field and its shape are verified; the semantic claim "external principal" remains unproven and Stream 1's success metric stays unmet by construction.

**13.9 — live-release distinction inside the Demo_Walkthrough.** The walkthrough is Dev-lane only by criterion, so it can demonstrate source-level readiness and both revenue paths but cannot demonstrate a live release. Coverage: every Dev-lane step is verifiable; the live-release half is reported as `not-ready` through the split readiness field (5.10) rather than demonstrated.

No other criterion is partially covered. Every remaining criterion in the tables above names one design element and one verification mechanism.
