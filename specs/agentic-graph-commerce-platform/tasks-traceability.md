---
title: "AgenticGraph Commerce Platform — Tasks Annex: Traceability and Coverage"
doc_type: "Spec Tasks Annex"
schema: "kiro-spec-tasks-annex/v1"
version: "0.1.0"
date: "2026-08-28"
lang: "en-US"
frontmatter_contract: "required"
feature_name: "agentic-graph-commerce-platform"
parent_document: "tasks.md"
requirements_baseline: ".kiro/specs/agentic-graph-commerce-platform/requirements.md v0.1.0 (128 acceptance criteria, CP-1..CP-24)"
design_baseline: ".kiro/specs/agentic-graph-commerce-platform/design.md v0.1.0 (40 design elements)"
criteria_total: 128
design_elements_total: 40
properties_total: 24
leaf_tasks_total: 100
---

# Tasks Annex — Traceability and Coverage

Sibling annex to `tasks.md`, split along the same owner boundary the design uses: the task list carries the work and its bounds, this annex carries the coverage proof. Split because both files obey Requirement 13.7's 600-line ceiling.

Every table below is **derived from `tasks.md` itself** — the criteria and element references parsed out of each task's `_Requirements:_` and `**Validates: Requirements**` lines — rather than maintained by hand. A criterion or element that appears here but not in `tasks.md` is impossible by construction; re-derive after editing the task list.

## Forward — task → criteria → design elements

100 leaf tasks. `verification` marks a `*` sub-task; skipping one leaves the criteria on its row unverified.

| Task | Kind | Acceptance criteria | Design elements | Property |
|---|---|---|---|---|
| 1.1 Complete the START-WORKFLOW claim, lease, fence, and runtime-identity stages | build | 9.7, 9.8, 12.8 | — | — |
| 1.2 Create the task-bounds schema and its validator | build | 13.1, 13.2, 13.8 | 40 | — |
| 1.3 Register the 15 named checks as fail-closed stubs | build | 13.1, 13.5 | 38, 40 | — |
| 1.4 Implement the evidence emitter | build | 13.3, 13.4, 13.6, 13.11 | 38 | — |
| 1.5 Implement the authored-limits scanner | build | 13.7, 13.8 | 39 | — |
| 1.6 Write generative tests for the evidence emitter | verification | 13.6, 13.11 | 38 | — |
| 2.1 Author the Terminology_Register and its reader | build | 1.1, 1.2, 1.3, 1.7, 1.10 | 1 | — |
| 2.2 Implement the terminology checker | build | 1.4, 1.9 | 2 | — |
| 2.3 Implement the legacy-identity guard | build | 1.6 | 3 | — |
| 2.4 Write property test for legacy identity rejection parity | verification | 1.5, 1.6, 1.8, 1.9 | 2, 3 | CP-1 |
| 2.5 Rename prose and annotate the externally-owned code occurrences | build | 1.1, 1.4, 1.5 | 4, 5 | — |
| 2.6 Record the baseline passing-check set and assert no behavior drift | verification | 1.8 | 1, 2, 5 | — |
| 3.1 Implement Convergence_Evaluator | build | 2.1, 2.2, 2.3, 2.5, 2.6, 2.7, 2.8, 2.10 | 6 | — |
| 3.2 Write property test for surplus tolerance and deficit blocking | verification | 2.1, 2.2, 2.3, 2.7 | 6 | CP-2 |
| 3.3 Write property test for verdict determinism | verification | 2.10 | 6 | CP-3 |
| 3.4 Write property test for revision and identity blocking | verification | 2.3, 2.5, 2.6, 2.8 | 6 | CP-4 |
| 3.5 Convert the upstream evidence verifier into a thin adapter | build | 2.1, 2.2, 2.3, 2.5, 2.6, 2.7, 2.8 | 7 | — |
| 3.6 Compose the split Readiness_Report | build | 2.4, 2.9, 2.11, 5.2, 5.10 | 8 | — |
| 3.7 Write integration examples for provider evidence retrieval and readiness composition | verification | 2.4, 2.9, 2.11 | 8 | — |
| 5.1 Implement Take_Rate_Calculator | build | 3.1, 3.3, 3.4 | 9 | — |
| 5.2 Create the RevenueLedger Durable Object with its migration | build | 3.2, 3.5, 3.9, 3.10, 3.11 | 10 | — |
| 5.3 Write property test for markup arithmetic and non-interference | verification | 3.1, 3.4, 3.6 | 9, 11 | CP-5 |
| 5.4 Write property test for ledger idempotence | verification | 3.2, 3.5 | 10 | CP-6 |
| 5.5 Write property test for period read-back and aggregation | verification | 3.9, 3.10 | 10 | CP-7 |
| 5.6 Add the post-settlement markup hook to CheckoutSession | build | 3.1, 3.2, 3.6, 3.8 | 11 | — |
| 5.7 Add the fail-closed take-rate configuration gate | build | 3.3, 3.7 | 12 | — |
| 5.8 Write integration examples for the configuration gate, deferral, and demand reporting | verification | 3.1, 3.7, 3.8, 3.11 | 10, 11, 12 | — |
| 7.1 Implement the Theme_Manifest schema | build | 4.1, 4.2, 4.4 | 13 | — |
| 7.2 Implement the merchant catalog projection | build | 4.5, 4.8 | 17 | — |
| 7.3 Write property test for theme manifest round trip | verification | 4.1, 4.2, 4.4 | 13 | CP-8 |
| 7.4 Create the ThemeDeployment Durable Object with its migration | build | 4.3, 4.9, 4.10 | 15 | — |
| 7.5 Implement theme deployment orchestration | build | 4.3, 4.9, 4.10 | 14 | — |
| 7.6 Write property test for catalog scope containment | verification | 4.3, 4.5 | 14, 17 | CP-9 |
| 7.7 Make the console renderer manifest-driven, mobile-first, and accessible | build | 4.1, 4.5, 4.6, 4.7, 9.1, 9.6 | 16 | — |
| 7.8 Add the theme deployment and merchant storefront routes | build | 4.1, 4.5, 4.8, 4.11 | 14, 16, 17 | — |
| 7.9 Bootstrap the browser check lane | build | 4.6, 4.7, 7.1, 7.3, 9.1, 9.5, 9.6 | 16, 25, 31 | — |
| 7.10 Write browser measurements for paint, markup, and touch targets | verification | 4.6, 4.7 | 16 | — |
| 7.11 Write integration examples for deployment failure paths and records | verification | 4.3, 4.8, 4.9, 4.10, 4.12 | 14, 15, 16, 17 | — |
| 9.1 Declare exactly two Dev lane entry points and gate the deploy scripts | build | 5.1, 5.2, 5.5 | 18 | — |
| 9.2 Declare the Production delivery route | build | 5.3 | 19 | — |
| 9.3 Implement Release_Controller | build | 5.4, 5.5, 5.6, 5.7, 5.8 | 20 | — |
| 9.4 Write property tests for controller refusal, retirement, and rollback | verification | 5.6, 5.7, 5.8 | 20 | CP-19 |
| 9.5 Report the deploy lane and candidate on the health and console surfaces | build | 5.1, 5.3, 5.4, 5.5, 5.9 | 8, 16, 20 | — |
| 9.6 Write integration examples for Dev egress and split readiness | verification | 5.2, 5.9, 5.10 | 8, 18 | — |
| 10.1 Author the capability→token map | build | 6.2, 6.7 | 21 | — |
| 10.2 Replace the fixed invocation bounds with register-declared bounds | build | 6.1, 6.2, 6.3, 6.8 | 22 | — |
| 10.3 Write property test for resolution totality | verification | 6.1, 6.3, 6.4, 6.6 | 22 | CP-10 |
| 10.4 Write property test for resolution idempotence across revisions | verification | 6.6, 6.8 | 22 | CP-11 |
| 10.5 Mount the operator MCP surface | build | 6.7 | 23 | — |
| 10.6 Write integration examples for catalog hydration failure and MCP reachability | verification | 6.5, 6.7 | 22, 23 | — |
| 12.1 Implement the first-party storefront session route (security-relevant) | build | 7.2, 7.4 | 23, 24 | — |
| 12.2 Narrow the console CSP and add the nonce-bound module tag | build | 7.1 | 16 | — |
| 12.3 Implement the single storefront action set | build | 7.2, 7.4 | 24 | — |
| 12.4 Implement WebMCP registration, the tool-set digest, and drift refusal | build | 7.1, 7.2, 7.3, 7.5, 7.6, 7.7 | 25 | — |
| 12.5 Write property test for dual-path guardrail parity | verification | 7.4, 7.7 | 24, 25 | CP-12 |
| 12.6 Write property test for registration drift refusal | verification | 7.5, 7.6 | 25 | CP-13 |
| 12.7 Write browser integration examples for capable and incapable engines | verification | 7.1, 7.3 | 25 | — |
| 12.8 Record the WebMCP boundary state and the standard's stage | build | 7.8, 7.9 | 36 | — |
| 13.1 Add declared attributes and one declared fallback to the registry projection | build | 8.1, 8.2 | 28 | — |
| 13.2 Migration — drop the one-agent-per-category index and relax `health()` | build | 8.1, 8.8 | 28 | — |
| 13.3 Implement the selection policy | build | 8.1, 8.2, 8.7 | 26 | — |
| 13.4 Write property test for selection determinism and the tie rule | verification | 8.1, 8.2, 8.7 | 26 | CP-15 |
| 13.5 Replace the ambiguity refusal with a scored selection | build | 8.1, 8.3, 8.6 | 27 | — |
| 13.6 Write property test for dispatch count totality | verification | 8.1, 8.3, 8.4, 8.5, 8.6 | 27, 29 | CP-14 |
| 13.7 Implement bounded fallback dispatch | build | 8.4, 8.5 | 29 | — |
| 13.8 Implement Public_Catalog_View and its unauthenticated route | build | 8.8, 8.9, 8.10 | 30 | — |
| 13.9 Write property test for public projection fidelity | verification | 8.8, 8.10 | 30 | CP-16 |
| 13.10 Write the projection latency integration example | verification | 8.9 | 30 | — |
| 15.1 Implement the order-independent sync merge | build | 9.4, 9.10 | 32 | — |
| 15.2 Implement the local change log | build | 9.1, 9.2, 9.3, 9.5 | 31 | — |
| 15.3 Write property test for merge confluence | verification | 9.4 | 32 | CP-17 |
| 15.4 Write property test for offline order preservation | verification | 9.2, 9.3 | 31 | CP-18 |
| 15.5 Create the AuthoringClaim Durable Object with its migration | build | 9.7, 9.8, 9.9, 12.8 | 33 | — |
| 15.6 Write property test for claim admission and byte preservation | verification | 9.7, 9.8, 9.9 | 33 | CP-19 |
| 15.7 Gate every operator mutation route behind claim admission | build | 9.7, 9.8, 9.9 | 33 | — |
| 15.8 Write browser and inventory checks for the offline surface | verification | 9.1, 9.5, 9.6, 9.10 | 16, 31, 32, 33 | — |
| 16.1 Implement Offer_Change_Listener | build | 10.1, 10.2, 10.5, 10.6, 10.7, 10.8 | 34 | — |
| 16.2 Write property test for change-event idempotence | verification | 10.2, 10.7 | 34 | CP-20 |
| 16.3 Add the session alarm and the settlement gate | build | 10.1, 10.3, 10.4 | 35 | — |
| 16.4 Write property test for settlement blocking after change | verification | 10.3, 10.4 | 35 | CP-21 |
| 16.5 Write integration examples for observation scheduling, stop reasons, and retries | verification | 10.1, 10.5, 10.6, 10.8, 10.9 | 34, 35 | — |
| 18.1 Declare the sandbox Worker with an empty privileged binding surface | build | 11.4, 11.6 | 36 | — |
| 18.2 Implement the isolated executor | build | 11.1, 11.2, 11.4, 11.5, 11.8 | 36 | — |
| 18.3 Write property test for isolation and allowlist enforcement | verification | 11.2, 11.3, 11.6 | 36 | CP-22 |
| 18.4 Write property test for resource-limit termination | verification | 11.4, 11.5 | 36 | CP-23 |
| 18.5 Implement scoped preview identities | build | 11.7 | 36 | — |
| 18.6 Wire the registration dry run into the registration path | build | 11.2, 11.3 | 36 | — |
| 18.7 Run the WebMCP lane inside sandbox isolation and record the browser support set | build | 7.8, 11.1, 11.4, 11.7, 11.8 | 25, 36 | — |
| 19.1 Author the merge agent bounds and lane binding | build | 12.5, 12.8 | 37 | — |
| 19.2 Implement review-comment handling | build | 12.1 | 37 | — |
| 19.3 Implement check repair with the two-attempt ceiling | build | 12.2, 12.4 | 37 | — |
| 19.4 Implement write-set-bounded conflict resolution | build | 12.3 | 37 | — |
| 19.5 Implement the run orchestrator and its forbidden-operation assertion | build | 12.6, 12.7, 12.8, 12.9 | 33, 37 | — |
| 19.6 Write property test for bounded merge mutation | verification | 12.3, 12.5, 12.7, 12.8, 12.9 | 37 | CP-24 |
| 19.7 Write process assertions for the merge agent branches | verification | 12.1, 12.2, 12.4, 12.6 | 37 | — |
| 20.1 Wire the aggregate verification gate | build | 13.1, 13.2, 13.5 | 38, 40 | — |
| 20.2 Emit verdict records and evidence references for the increment | build | 13.3, 13.4, 13.5, 13.6, 13.11 | 38 | — |
| 20.3 Write generative tests for reference emission and rung derivation | verification | 13.6, 13.11 | 38 | — |
| 20.4 Sweep authored limits and remediate findings | build | 13.7, 13.8 | 39 | — |
| 20.5 Author `demo.md` | build | 13.9, 13.10 | 8, 11, 14, 16, 18 | — |

## Reverse check 1 — every acceptance criterion has at least one task

Criteria in the requirements baseline: **128**. Criteria with at least one covering task: **128**. Uncovered: **0**.

Criteria 1.5, 3.11, and 13.9 are covered **partially** per Annex B; their covering tasks state the partial disposition rather than claiming full satisfaction.

| Criterion | Covering task(s) |
|---|---|
| 1.1 | 2.1, 2.5 |
| 1.2 | 2.1 |
| 1.3 | 2.1 |
| 1.4 | 2.2, 2.5 |
| 1.5 (PARTIAL) | 2.4, 2.5 |
| 1.6 | 2.3, 2.4 |
| 1.7 | 2.1 |
| 1.8 | 2.4, 2.6 |
| 1.9 | 2.2, 2.4 |
| 1.10 | 2.1 |
| 2.1 | 3.1, 3.2, 3.5 |
| 2.2 | 3.1, 3.2, 3.5 |
| 2.3 | 3.1, 3.2, 3.4, 3.5 |
| 2.4 | 3.6, 3.7 |
| 2.5 | 3.1, 3.4, 3.5 |
| 2.6 | 3.1, 3.4, 3.5 |
| 2.7 | 3.1, 3.2, 3.5 |
| 2.8 | 3.1, 3.4, 3.5 |
| 2.9 | 3.6, 3.7 |
| 2.10 | 3.1, 3.3 |
| 2.11 | 3.6, 3.7 |
| 3.1 | 5.1, 5.3, 5.6, 5.8 |
| 3.2 | 5.2, 5.4, 5.6 |
| 3.3 | 5.1, 5.7 |
| 3.4 | 5.1, 5.3 |
| 3.5 | 5.2, 5.4 |
| 3.6 | 5.3, 5.6 |
| 3.7 | 5.7, 5.8 |
| 3.8 | 5.6, 5.8 |
| 3.9 | 5.2, 5.5 |
| 3.10 | 5.2, 5.5 |
| 3.11 (PARTIAL) | 5.2, 5.8 |
| 4.1 | 7.1, 7.3, 7.7, 7.8 |
| 4.2 | 7.1, 7.3 |
| 4.3 | 7.4, 7.5, 7.6, 7.11 |
| 4.4 | 7.1, 7.3 |
| 4.5 | 7.2, 7.6, 7.7, 7.8 |
| 4.6 | 7.7, 7.9, 7.10 |
| 4.7 | 7.7, 7.9, 7.10 |
| 4.8 | 7.2, 7.8, 7.11 |
| 4.9 | 7.4, 7.5, 7.11 |
| 4.10 | 7.4, 7.5, 7.11 |
| 4.11 | 7.8 |
| 4.12 | 7.11 |
| 5.1 | 9.1, 9.5 |
| 5.2 | 3.6, 9.1, 9.6 |
| 5.3 | 9.2, 9.5 |
| 5.4 | 9.3, 9.5 |
| 5.5 | 9.1, 9.3, 9.5 |
| 5.6 | 9.3, 9.4 |
| 5.7 | 9.3, 9.4 |
| 5.8 | 9.3, 9.4 |
| 5.9 | 9.5, 9.6 |
| 5.10 | 3.6, 9.6 |
| 6.1 | 10.2, 10.3 |
| 6.2 | 10.1, 10.2 |
| 6.3 | 10.2, 10.3 |
| 6.4 | 10.3 |
| 6.5 | 10.6 |
| 6.6 | 10.3, 10.4 |
| 6.7 | 10.1, 10.5, 10.6 |
| 6.8 | 10.2, 10.4 |
| 7.1 | 7.9, 12.2, 12.4, 12.7 |
| 7.2 | 12.1, 12.3, 12.4 |
| 7.3 | 7.9, 12.4, 12.7 |
| 7.4 | 12.1, 12.3, 12.5 |
| 7.5 | 12.4, 12.6 |
| 7.6 | 12.4, 12.6 |
| 7.7 | 12.4, 12.5 |
| 7.8 | 12.8, 18.7 |
| 7.9 | 12.8 |
| 8.1 | 13.1, 13.2, 13.3, 13.4, 13.5, 13.6 |
| 8.2 | 13.1, 13.3, 13.4 |
| 8.3 | 13.5, 13.6 |
| 8.4 | 13.6, 13.7 |
| 8.5 | 13.6, 13.7 |
| 8.6 | 13.5, 13.6 |
| 8.7 | 13.3, 13.4 |
| 8.8 | 13.2, 13.8, 13.9 |
| 8.9 | 13.8, 13.10 |
| 8.10 | 13.8, 13.9 |
| 9.1 | 7.7, 7.9, 15.2, 15.8 |
| 9.2 | 15.2, 15.4 |
| 9.3 | 15.2, 15.4 |
| 9.4 | 15.1, 15.3 |
| 9.5 | 7.9, 15.2, 15.8 |
| 9.6 | 7.7, 7.9, 15.8 |
| 9.7 | 1.1, 15.5, 15.6, 15.7 |
| 9.8 | 1.1, 15.5, 15.6, 15.7 |
| 9.9 | 15.5, 15.6, 15.7 |
| 9.10 | 15.1, 15.8 |
| 10.1 | 16.1, 16.3, 16.5 |
| 10.2 | 16.1, 16.2 |
| 10.3 | 16.3, 16.4 |
| 10.4 | 16.3, 16.4 |
| 10.5 | 16.1, 16.5 |
| 10.6 | 16.1, 16.5 |
| 10.7 | 16.1, 16.2 |
| 10.8 | 16.1, 16.5 |
| 10.9 | 16.5 |
| 11.1 | 18.2, 18.7 |
| 11.2 | 18.2, 18.3, 18.6 |
| 11.3 | 18.3, 18.6 |
| 11.4 | 18.1, 18.2, 18.4, 18.7 |
| 11.5 | 18.2, 18.4 |
| 11.6 | 18.1, 18.3 |
| 11.7 | 18.5, 18.7 |
| 11.8 | 18.2, 18.7 |
| 12.1 | 19.2, 19.7 |
| 12.2 | 19.3, 19.7 |
| 12.3 | 19.4, 19.6 |
| 12.4 | 19.3, 19.7 |
| 12.5 | 19.1, 19.6 |
| 12.6 | 19.5, 19.7 |
| 12.7 | 19.5, 19.6 |
| 12.8 | 1.1, 15.5, 19.1, 19.5, 19.6 |
| 12.9 | 19.5, 19.6 |
| 13.1 | 1.2, 1.3, 20.1 |
| 13.2 | 1.2, 20.1 |
| 13.3 | 1.4, 20.2 |
| 13.4 | 1.4, 20.2 |
| 13.5 | 1.3, 20.1, 20.2 |
| 13.6 | 1.4, 1.6, 20.2, 20.3 |
| 13.7 | 1.5, 20.4 |
| 13.8 | 1.2, 1.5, 20.4 |
| 13.9 (PARTIAL) | 20.5 |
| 13.10 | 20.5 |
| 13.11 | 1.4, 1.6, 20.2, 20.3 |

## Reverse check 2 — every design element is built by at least one task

Design elements in the design baseline: **40**. Elements with at least one building task: **40**. Unbuilt: **0**.

| Element | Building task(s) |
|---|---|
| 1 | 2.1, 2.6 |
| 2 | 2.2, 2.4, 2.6 |
| 3 | 2.3, 2.4 |
| 4 | 2.5 |
| 5 | 2.5, 2.6 |
| 6 | 3.1, 3.2, 3.3, 3.4 |
| 7 | 3.5 |
| 8 | 3.6, 3.7, 9.5, 9.6, 20.5 |
| 9 | 5.1, 5.3 |
| 10 | 5.2, 5.4, 5.5, 5.8 |
| 11 | 5.3, 5.6, 5.8, 20.5 |
| 12 | 5.7, 5.8 |
| 13 | 7.1, 7.3 |
| 14 | 7.5, 7.6, 7.8, 7.11, 20.5 |
| 15 | 7.4, 7.11 |
| 16 | 7.7, 7.8, 7.9, 7.10, 7.11, 9.5, 12.2, 15.8, 20.5 |
| 17 | 7.2, 7.6, 7.8, 7.11 |
| 18 | 9.1, 9.6, 20.5 |
| 19 | 9.2 |
| 20 | 9.3, 9.4, 9.5 |
| 21 | 10.1 |
| 22 | 10.2, 10.3, 10.4, 10.6 |
| 23 | 10.5, 10.6, 12.1 |
| 24 | 12.1, 12.3, 12.5 |
| 25 | 7.9, 12.4, 12.5, 12.6, 12.7, 18.7 |
| 26 | 13.3, 13.4 |
| 27 | 13.5, 13.6 |
| 28 | 13.1, 13.2 |
| 29 | 13.6, 13.7 |
| 30 | 13.8, 13.9, 13.10 |
| 31 | 7.9, 15.2, 15.4, 15.8 |
| 32 | 15.1, 15.3, 15.8 |
| 33 | 15.5, 15.6, 15.7, 15.8, 19.5 |
| 34 | 16.1, 16.2, 16.5 |
| 35 | 16.3, 16.4, 16.5 |
| 36 | 12.8, 18.1, 18.2, 18.3, 18.4, 18.5, 18.6, 18.7 |
| 37 | 19.1, 19.2, 19.3, 19.4, 19.5, 19.6, 19.7 |
| 38 | 1.3, 1.4, 1.6, 20.1, 20.2, 20.3 |
| 39 | 1.5, 20.4 |
| 40 | 1.2, 1.3, 20.1 |

## Reverse check 3 — every correctness property has a task that writes and runs it

Properties in the design baseline: **24** (CP-1..CP-24). Properties with a task: **24**. Missing: **0**.

| Property | Task | Lane |
|---|---|---|
| CP-1 | 2.4 | `npm run test:unit` |
| CP-2 | 3.2 | `npm run test:domain` |
| CP-3 | 3.3 | `npm run test:domain` |
| CP-4 | 3.4 | `npm run test:domain` |
| CP-5 | 5.3 | `npm run test:domain` |
| CP-6 | 5.4 | `npm run test:workers` |
| CP-7 | 5.5 | `npm run test:workers` |
| CP-8 | 7.3 | `npm run test:domain` |
| CP-9 | 7.6 | `npm run test:domain` |
| CP-10 | 10.3 | `npm run test:domain` |
| CP-11 | 10.4 | `npm run test:domain` |
| CP-12 | 12.5 | `npm run test:unit` |
| CP-13 | 12.6 | `npm run test:unit` |
| CP-14 | 13.6 | `npm run test:workers` |
| CP-15 | 13.4 | `npm run test:domain` |
| CP-16 | 13.9 | `npm run test:workers` |
| CP-17 | 15.3 | `npm run test:domain` |
| CP-18 | 15.4 | `npm run test:domain` |
| CP-19 | 9.4, 15.6 | `npm run test:workers` |
| CP-20 | 16.2 | `npm run test:workers` |
| CP-21 | 16.4 | `npm run test:workers` |
| CP-22 | 18.3 | `npm run test:unit` |
| CP-23 | 18.4 | `npm run test:unit` |
| CP-24 | 19.6 | `npm run test:unit` |

## What These Checks Do And Do Not Prove

They prove that every acceptance criterion is named by at least one task, every design element is named by at least one task, and every correctness property has one task that writes and runs it. That is a mapping, not a verification result.

They do not prove:

- **That the mapping is sufficient.** A criterion named by a task whose bounds are exhausted is still uncovered in practice. The verdict record per task (Requirement 13.3) is what turns a mapping into evidence.
- **That partial coverage became full.** Criteria 1.5, 3.11, and 13.9 are marked `(PARTIAL)` above and stay partial for the reasons Annex B records: an upstream rename this repository does not own, a ledger that carries an agent identifier rather than a payer principal, and a Dev-lane-only walkthrough that cannot demonstrate a live release.
- **That a `verification` row ran.** Rows marked `verification` correspond to `*` tasks. Skipping one leaves the criteria on that row unverified; nothing else in this annex changes when it is skipped, which is exactly why the distinction is in the Kind column.
- **That four non-CP obligations are properties in the CP series.** Annex B describes verification for 1.9's injected-occurrence arm, 5.6, 5.7, 5.8, 13.6, and 13.11 as properties, but the design fixes the property set at CP-1..CP-24. Tasks 2.4, 9.4, 1.6, and 20.3 implement them as bounded generative tests labelled non-CP. Reverse check 3 counts CP-1..CP-24 only, so those six obligations appear in reverse check 1 and not in reverse check 3.
