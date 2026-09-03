---
title: "AgenticGraph Commerce Platform — Requirements"
doc_type: "Spec Requirements"
schema: "kiro-spec-requirements/v1"
version: "0.1.0"
date: "2026-08-28"
lang: "en-US"
frontmatter_contract: "required"
spec_type: "feature"
workflow_type: "requirements-first"
feature_name: "agentic-graph-commerce-platform"
owner: "Solo Founder / AI Orchestrator"
lane: "authoring"
local_rung: "spec-complete"
delivered_rung: "undocumented"
deploy_boundary: "dev-only"
source_specification: "joohwee/prd-tad-ard/agentic-graph-commerce-platform-prd-tad-adr.md v0.12.0 (2026-08-26)"
implementation_baseline: "agentic-commerce-os @ main 2e39e5c43f2f49866f4dd6849994f7bfdf677ec1"
governing_contracts:
  - "huijoohwee.github.io/guidelines/agentic-sdlc-guidelines.md v1.23.0"
  - "agentic-canvas-os/docs/START-WORKFLOW.md (knowgrph-start-workflow/v2)"
  - "agentic-canvas-os/docs/AGENTS.md (agentic-os-agents/v1)"
supersedes: ".kiro/specs/knowgrph-agentic-commerce-platform (naming and PRD revision superseded)"
deliverable_set: ["requirements.md", "design.md", "tasks.md", "demo.md"]
naming_authority: "agenticgraph | agentic-graph | agentic_graph | agentic.graph | AG_ | agc supersede knowgrph | knowledge-graph | knowledgegraph | KG_ | kgc"
---

# Requirements Document

## Introduction

This document derives executable requirements for the AgenticGraph Commerce Platform increment specified in `agentic-graph-commerce-platform-prd-tad-adr.md` v0.12.0, grounded in the code that already exists in `agentic-commerce-os` at the implementation baseline named in the frontmatter.

The source specification adds one new primitive — an Agent Registry/Router over a reused commerce chain — and then layers four monetization streams, a WebMCP tool-exposure surface, a white-label storefront template pack, and an isolated-execution layer on top of it. The implementation baseline already contains a substantial part of that shape: an authenticated edge Worker exposing `/mcp` plus `/v1/*` routes, a private core Worker owning `AgentRegistry`, `IntentRoute`, and `CheckoutSession` SQLite Durable Objects, a deterministic exclusive-category router, an ACOS admission receipt store, a pinned `/`, `#`, `@` invocation catalog, a read-only mobile-first console at `GET /`, and a three-lane test harness (`node --test` domain tests, Vitest unit tests, and Vitest workers-pool tests against real `workerd`).

Requirements are therefore **ranked by proximity to the built baseline**: Requirement 1 changes identifiers only, Requirement 2 relaxes one already-written verification function, and each later requirement adds progressively more new surface. This ordering is the ranking obligation itself, not a presentation choice — see Capability Ranking and Reasoning Grounding Chain below.

Scope discipline: every requirement traces to a named PRD user story, monetization stream, ADR, architecture row, or governing-contract rule. Where the source specification states an honest gap, this document states the same gap as a bounded requirement rather than closing it silently. No requirement introduces behavior absent from the source specification or the governing contracts.

Deploy boundary: every acceptance criterion in this document is satisfiable entirely inside the Dev lane. The Prod mirror and the Cloudflare delivery route are gated targets named in Requirement 5 and are never acceptance criteria for any other requirement.

## Session Start Declaration

Recorded per `START-WORKFLOW.md`. This pass authored specification documents only; it performed no fetch, no lease claim, no ledger transition, and no application-code edit. Values below are inspected, not assumed, and the honest gaps are stated rather than implied.

```yaml
action: /session.start
semantics: ["#multi-agent-collaboration", "#runtime-ready", "#no-hardcode"]
bindings: ["@operator", "@working-directory", "@runtime-proof"]
semantic_scope: agentic-graph-commerce-platform
base_ref: origin/main
observed_source_branch: main
observed_source_sha: 2e39e5c43f2f49866f4dd6849994f7bfdf677ec1
authoring_status: ready
write_scope: [".kiro/specs/agentic-graph-commerce-platform/**"]
application_code_mutation: forbidden-in-this-phase
deploy_boundary: dev-only
fetch_performed: false
writer_lease_claimed: false
ledger_claim_recorded: false
runtime_identity_verified: false
declaration_gap: "Ownership claim, writer lease, fencing SHA, and runtime identity are not established by this documentation pass; a code-bearing task derived from this document must complete the full claim and activate stages before mutation."
```

## Deliverable Set

The complete deliverable set for this feature is four artifacts. This phase produces the first only; the remaining three are recorded here so no artifact is silently dropped.

| Artifact | Phase | Owner obligation |
|---|---|---|
| `requirements.md` | this phase | EARS + INCOSE compliant acceptance criteria, correctness properties, traceability |
| `design.md` | next phase | Component design covering every acceptance criterion; no uncovered criterion |
| `tasks.md` | after design | Executable task list where every task traces to at least one VCC and every VCC is covered |
| `demo.md` | after tasks | Operator-runnable walkthrough of the shipped increment, including the first-dollar path in Requirement 3 and the merchant happy path in Requirement 4 |

## Glossary

- **AG_Edge**: The authenticated edge Worker. Exists at baseline (`src/edge/index.ts`); serves `GET /`, `GET /livez`, `GET /readyz`, `POST /mcp`, the `/v1/*` MCP-authority routes, and the `/v1/operator/*` operator routes.
- **AG_Core**: The private core Worker reached only over a Service Binding. Exists at baseline (`src/core/index.ts`); owns every Durable Object and every provider call.
- **Agent_Registry**: The `AgentRegistry` SQLite Durable Object that stores admission receipts and registered agent records. Exists at baseline (`src/core/agent-registry.ts`).
- **Intent_Router**: The deterministic component that resolves one typed intent to at most one registered agent and persists the routing decision. Exists at baseline (`src/domain/exclusive-category-router.ts`, `src/core/intent-route.ts`).
- **Checkout_Session**: The Durable Object enforcing routing evidence, guardrail evaluation, human confirmation, and one settlement call per session. Exists at baseline (`src/core/checkout-session.ts`).
- **Invocation_Catalog**: The pinned, revision- and digest-bound projection of the `/`, `#`, and `@` token dictionaries. Exists at baseline (`src/invocation/catalog.ts`).
- **Invocation_Token**: One `/`, `#`, or `@` token resolvable through Invocation_Catalog. `/` denotes a command, `#` a semantic, `@` a binding.
- **Readiness_Report**: The fail-closed dependency and candidate report returned by `GET /readyz`. Exists at baseline (`src/core/index.ts`).
- **Convergence_Evaluator**: The component that decides whether an upstream provider's advertised evidence satisfies this platform's declared requirement set. Exists at baseline in exact-parity form (`src/core/upstream-evidence.ts`); Requirement 2 replaces the parity rule with a tolerant rule.
- **Required_Check_Set**: The named set of upstream evidence checks this platform declares it depends on, per contract (checkout and marketplace sets exist at baseline).
- **Convergence_Verdict**: The typed outcome of Convergence_Evaluator for one provider: `converged`, `converged-with-surplus`, or `blocked` plus a reason.
- **Take_Rate_Calculator**: The deterministic component computing the platform markup on one settled amount. PRD component, not yet built.
- **Settled_Amount**: The provider-confirmed amount for one confirmed checkout, as recorded by Checkout_Session.
- **Revenue_Ledger**: The relational record of customers, invoices, and settled markup lines, per ADR-8. Not yet built.
- **Storefront_Console**: The read-only, mobile-first browser surface served at `GET /`. Exists at baseline (`src/edge/dashboard.ts`).
- **Template_Pack**: A themeable, per-merchant deployment configuration over Storefront_Console — palette, logo, copy, and catalog scope — sold under Monetization Stream 4. Not yet built.
- **Theme_Manifest**: The typed, validated per-merchant configuration record a Template_Pack deployment is built from.
- **WebMCP_Tool_Surface**: The browser-native tool registration surface exposing platform actions through `navigator.modelContext` or `document.modelContext` to any visiting WebMCP-capable agent. Not yet built.
- **Public_Catalog_View**: The unauthenticated, agent-facing and shopper-facing projection of registered agents and their declared capabilities, per ADR-10. Not yet built.
- **Offer_Change_Listener**: The component that observes a held offer's price, availability, and agent-registration state after initial dispatch and emits a typed change event. PRD component "Agent State-Change Listener", not yet built.
- **Held_Offer**: An offer already shown to or selected by a shopper in an open Checkout_Session and not yet settled.
- **Sandbox_Executor**: The isolated-container execution layer used for untrusted merchant input, pre-registration dry runs, and build-from-zero work on unshipped surfaces, per ADR-13. Not yet built.
- **Merge_Agent**: The bounded automation that responds to review comments, repairs failing required checks, and resolves merge conflicts on one owned lane. Not yet built.
- **Legacy_Identifier**: Any occurrence of `knowgrph`, `knowledge-graph`, `knowledgegraph`, `KG_`, or `kgc` in a repository-authored identifier, path, service name, endpoint, tool name, environment key, or document body.
- **Superseding_Identifier**: The corresponding `agenticgraph`, `agentic-graph`, `agentic_graph`, `agentic.graph`, `AG_`, or `agc` form that replaces a Legacy_Identifier.
- **Terminology_Register**: The single authored mapping from each Legacy_Identifier to its Superseding_Identifier, plus the disposition of each occurrence as `renamed`, `externally-owned`, or `historical-record`.
- **Dev_Lane**: The development runtime at `GitHub/agentic-commerce-os`, started through repository-owned scripts (`npm run dev`, and the Home Apex entry point `npm run dev:apex`).
- **Prod_Mirror**: The generated release output at `GitHub/huijoohwee/content/agentic-commerce-os`. Never a default edit target.
- **Delivery_Route**: The Cloudflare delivery target `airvio.co/agentic-commerce-os`.
- **Release_Controller**: The repository-owned protected controller that is the only mechanism permitted to advance Prod_Mirror or Delivery_Route.
- **Deploy_Boundary**: A recorded gate between lanes whose state reads `closed` or `pending-protected-integration` absent an explicit recorded operator authorization.
- **Evidence_Event**: One append-only, ordered record of a routing, guardrail, confirmation, settlement, change-detection, or registration decision, readable for verification.
- **Authoring_Lane**: A registered task worktree bound to one branch, one semantic scope, one declared write set, and one current lease, per `START-WORKFLOW.md`.
- **Concurrent_Author**: One writer identified by actor, device, session, worktree, branch, semantic scope, lease epoch, and fence revision.
- **Demo_Walkthrough**: The `demo.md` artifact in the deliverable set.
- **VCC**: Verifiable Completion Condition — a named check plus a recorded result that an evaluator distinct from the implementer judges from surfaced output.

## Capability Ranking and Reasoning Grounding Chain

Every capability below carries the full chain the operating context requires: pain point with its demand evidence, solution, feature, and monetization. The rank column is the build order obligation — lower rank means closer to the built baseline and therefore cheaper in time, tokens, and total cost of ownership. Demand evidence is recorded at its actual strength; `unvalidated` appears where no paying customer exists yet, because an unspecified demand claim cannot be refuted.

| Rank | Requirement | Pain point (demand evidence) | Solution | Feature proximity to baseline | Monetization |
|---:|---|---|---|---|---|
| 1 | R1 Terminology supersession | Two live vocabularies for one platform break token resolution, service binding lookup, and operator trust (provable: baseline carries Legacy_Identifiers in the invocation endpoint, the invocation tool name, three Production service bindings, README, and runtime docs) | One Terminology_Register plus mechanical rename with an enforcement check | Identifier-only; zero behavior change | Enabler — every paid surface below resolves tokens and bindings through the renamed identity |
| 2 | R2 Tolerant convergence | Exact-parity readiness blocks Production on a benign upstream advance, so no release ever reaches a customer (provable: baseline requires exact set equality of check names, exact revision equality, and exact digest equality) | Convergence_Evaluator returns `converged-with-surplus` when the Required_Check_Set is satisfied | One function's rule, one report field | Enabler — Streams 1 and 4 cannot bill from a lane that cannot release |
| 3 | R3 Take-rate on confirmed checkout | Platform routes value and captures none (Stream 1; demand `unvalidated` — zero external principals have transacted twice) | Take_Rate_Calculator on the existing confirm path plus a Revenue_Ledger line | One deterministic module on an existing route and Durable Object | **Stream 1**, markup on routed volume; the first-dollar mechanism |
| 4 | R4 White-label Template Pack | Small merchants want a polished multi-vendor storefront and will not build one (Stream 4; demand `unvalidated`, outreach unblocked) | Theme_Manifest plus per-merchant deployment over Storefront_Console | Configuration and asset substitution over a built console | **Stream 4**, one-time setup fee plus recurring hosting fee — fastest realistic path to a first dollar (ADR-12) |
| 5 | R5 Environment chain and boundaries | Work that cannot reach `airvio.co/agentic-commerce-os` cannot be sold (provable: the edge route list is empty at baseline) | Named Dev → Prod_Mirror → Delivery_Route chain behind Release_Controller | Configuration plus controller wiring | Enabler — the delivery surface every stream bills through |
| 6 | R6 Invocation surface universality | A second command registry fragments agent and operator entry (provable: one pinned catalog exists and must stay singular) | One Invocation_Catalog owning `/`, `#`, `@` for every new surface | Extension of a built catalog | Enabler — agent-facing discovery for Streams 1 and 3 |
| 7 | R7 WebMCP tool surface | Bespoke per-agent backend integration is the cost that stops third-party agents arriving (ADR-11; demand `unvalidated`, standard is origin-trial stage) | Register platform actions as browser-native tools reusing existing client functions | New browser surface over built client actions | Stream 1 volume expansion; Stream 3 groundwork |
| 8 | R8 Neutral aggregation routing | A fixed-enum router gives an agent no reason to route through rather than around this platform (ADR-10) | Price, quality, and latency aware selection with typed fallback, plus Public_Catalog_View | New selection policy over a built deterministic router | Stream 1 — markup only scales with routed volume |
| 9 | R9 Local-first, offline-first, multi-device | A dropped connection on a phone loses shopper and operator state, and two devices authoring at once corrupt it | Local-first state with ordered replay, and one active writer per Authoring_Lane | New client state layer; lane rules already contracted | Enabler — mobile-first is where Stream 4 merchants demo |
| 10 | R10 Held-offer change detection | Nothing watches a held offer after dispatch, so a shopper can confirm a stale price (provable: the PRD scores its own domain object below rubric L1) | Offer_Change_Listener emitting typed change events into Checkout_Session | New component; reuses the built session evidence log | Protects Stream 1 revenue integrity |
| 11 | R11 Isolated execution | Untrusted merchant theme input and unvetted agent tool calls have nowhere safe to run (ADR-2 gap; ADR-13 partial answer) | Sandbox_Executor for merchant builds, pre-registration dry runs, and unshipped-surface work | New layer; consumes R4 and R7 outputs | Stream 4 support cost containment |
| 12 | R12 Agent merge automation | Review comments, failing checks, and conflicts consume the solo operator's scarcest resource | Merge_Agent acting on one owned lane within stated bounds | New automation over the contracted lane model | Enabler — cadence, not revenue |
| 13 | R13 Execution evidence and Demo_Walkthrough | An asserted completion earns no readiness rung and demos nothing | Named check plus independent verdict per task, and a Demo_Walkthrough per increment | Process obligation over the built three-lane test harness | Enabler — evidence is what a buyer and an auditor both read |

## Requirements

### Requirement 1: Terminology Supersession and Compatibility Migration

**User Story:** As the Platform Operator, I want one vocabulary for this platform across code, configuration, and documentation, so that a token, a binding, or a service name resolves the same way everywhere.

Traces to: naming authority in this document's frontmatter; `AGENTS.md` rule against downstream aliases and compatibility remaps; baseline Legacy_Identifier occurrences in `src/invocation/catalog.ts`, `wrangler.core.jsonc`, `docs/production-runtime.md`, `README.md`, and `test/invocation/client.test.ts`.

Grounding chain: pain point — two vocabularies for one platform, provable from the baseline occurrences named above; solution — one Terminology_Register plus mechanical rename; feature — identifier-only change at rank 1; monetization — enabler for every stream below.

#### Acceptance Criteria

1. THE Terminology_Register SHALL record, for every Legacy_Identifier occurrence in a version-controlled repository-authored file — excluding dependency directories, build output, and generated Prod_Mirror output — exactly one Superseding_Identifier and exactly one disposition drawn from `renamed`, `externally-owned`, or `historical-record`.
2. WHEN a Legacy_Identifier occurrence is dispositioned `externally-owned`, THE Terminology_Register SHALL record the owning system that requires the legacy form and the recorded reason the repository cannot change that form.
3. WHEN a Legacy_Identifier occurrence is dispositioned `historical-record`, THE Terminology_Register SHALL record the artifact that must preserve the legacy form for provenance and SHALL record that the occurrence is read-only.
4. THE repository SHALL contain, within the file scope stated in criterion 1, zero Legacy_Identifier occurrences carrying no Terminology_Register disposition and zero Legacy_Identifier occurrences dispositioned `renamed`.
5. WHEN AG_Core resolves an Invocation_Token, THE AG_Core SHALL resolve that token through exactly one endpoint identity and exactly one tool identity, both expressed in Superseding_Identifier form.
6. IF a request presents a Legacy_Identifier form of an endpoint or tool identity, THEN THE AG_Core SHALL return a typed rejection naming the Superseding_Identifier form, SHALL resolve zero requests to the legacy target, and SHALL leave stored state unmodified.
7. THE repository SHALL define exactly one environment-key prefix, `AG_`, for repository-authored configuration keys, and SHALL contain zero repository-authored keys carrying a `KG_` prefix other than keys the Terminology_Register dispositions `externally-owned` or `historical-record`.
8. WHEN the terminology migration completes, THE existing verification lane SHALL report the identical set of passing checks recorded at the implementation baseline revision named in this document's frontmatter, with zero behavior-bearing assertions changed.
9. THE repository SHALL provide one deterministic check that returns the identical verdict for identical repository content, that fails when an undispositioned Legacy_Identifier occurrence is introduced, that names each failing occurrence by file path and line number, and that executes without a model call and without a paid call.

#### Stated Gap

10. THE Terminology_Register SHALL record every upstream Cloudflare service target still bearing a Legacy_Identifier as `externally-owned` until that upstream rename lands, and THE repository SHALL introduce zero alias layers, zero duplicated invocation dictionaries, and zero compatibility remaps to hide the difference.

### Requirement 2: Tolerant Upstream Convergence Instead of Exact Parity

**User Story:** As the Platform Operator, I want Production readiness to require semantic convergence rather than byte-exact upstream parity, so that a benign upstream advance does not indefinitely block a release a customer is waiting for.

Traces to: operating context rule that Production releases must not rely on brittle exact-parity checks; baseline `verifyUpstreamRuntimeEvidence` exact set, revision, and digest equality; `agentic-sdlc-guidelines.md` Runtime Readiness Enforcement (fail-closed on missing evidence, not on surplus evidence).

Grounding chain: pain point — an unreleasable lane, provable from the baseline equality rules; solution — Required_Check_Set satisfaction with declared tolerance; feature — one function's rule at rank 2; monetization — enabler for Streams 1 and 4.

#### Acceptance Criteria

1. WHEN Convergence_Evaluator evaluates one provider's advertised evidence whose passing checks contain every member of the Required_Check_Set and zero check outside the Required_Check_Set, THE Convergence_Evaluator SHALL return `converged`.
2. WHEN the advertised passing checks contain every member of the Required_Check_Set and at least one check outside the Required_Check_Set, THE Convergence_Evaluator SHALL return `converged-with-surplus` and SHALL record each surplus check name in the Convergence_Verdict.
3. IF at least one member of the Required_Check_Set is absent from the advertised evidence or is advertised with a failing result, THEN THE Convergence_Evaluator SHALL return `blocked` and SHALL record every such member by name.
4. IF one provider's advertised evidence remains unretrievable or incomplete 5 seconds after the readiness request, THEN THE Convergence_Evaluator SHALL return `blocked` and SHALL record the provider name and the absent-evidence reason.
5. WHEN Convergence_Evaluator compares an advertised contract revision against the declared revision, THE Convergence_Evaluator SHALL return `converged` for an advertised revision whose major component equals the declared major component and whose minor component is equal to or greater than the declared minor component.
6. IF an advertised contract revision carries a major component differing from the declared major component, or carries a minor component lower than the declared minor component, THEN THE Convergence_Evaluator SHALL return `blocked` and SHALL record both revision values.
7. WHEN an advertised evidence envelope presents every Required_Check_Set member and every declared identity field as present and satisfied, THE Convergence_Evaluator SHALL return the verdict determined by the advertised passing checks alone, regardless of the count of envelope fields the declared schema leaves unnamed, and SHALL record that count in the Convergence_Verdict.
8. IF an advertised evidence envelope omits a declared identity field, carries a declared identity field whose value fails the declared format, or fails the envelope's own integrity binding, THEN THE Convergence_Evaluator SHALL return `blocked` and SHALL record each such field name with the recorded failure reason.
9. WHEN Readiness_Report is produced, THE Readiness_Report SHALL carry exactly one Convergence_Verdict per declared provider, SHALL report ready only where every carried Convergence_Verdict reads `converged` or `converged-with-surplus`, and SHALL report not-ready where at least one carried Convergence_Verdict reads `blocked`.
10. THE Convergence_Evaluator SHALL return the identical Convergence_Verdict for identical advertised evidence and identical declared requirements, within 200 milliseconds of receiving that advertised evidence.
11. THE Readiness_Report SHALL carry the source-level Convergence_Verdict and the live-release Convergence_Verdict as two separately named fields per declared provider, and SHALL report not-ready where the live-release field is absent.

### Requirement 3: First Settled Markup on the Confirmed-Checkout Path

**User Story:** As the Platform Operator, I want a configured markup computed and recorded on every confirmed checkout, so that the platform earns a first real dollar from volume it already routes.

Traces to: PRD Monetization Model Stream 1; ADR-5; ADR-8 Revenue_Ledger on the relational store; PRD Component "Take-Rate Calculator"; PRD Success Metric "Take-rate computation correctness"; baseline `POST /v1/checkouts/{id}/confirm` and `src/core/checkout-session.ts`.

Grounding chain: pain point — the platform routes value and captures none, demand `unvalidated`; solution — Take_Rate_Calculator plus a Revenue_Ledger line; feature — one deterministic module on an existing route at rank 3; monetization — Stream 1, the first-dollar mechanism.

#### Acceptance Criteria

1. WHEN Checkout_Session records a confirmed settlement, THE Take_Rate_Calculator SHALL compute exactly one markup amount from that settlement's Settled_Amount and the configured rate within 500 milliseconds of that record.
2. WHEN Take_Rate_Calculator returns a markup amount for one settlement, THE Checkout_Session SHALL append exactly one Revenue_Ledger line carrying that settlement identifier.
3. THE Take_Rate_Calculator SHALL read the markup rate as an integer count of basis points from one externalized configuration key carrying the `AG_` prefix, and THE repository SHALL contain zero hardcoded markup rate values in source.
4. THE Take_Rate_Calculator SHALL compute the markup amount in the settlement currency's minor units using integer arithmetic, SHALL round the computed value half up to the nearest whole minor unit, and SHALL produce a markup amount no less than 0 minor units and no greater than the Settled_Amount.
5. WHEN the same settlement identifier is presented more than once, THE Checkout_Session SHALL retain exactly one Revenue_Ledger line for that settlement identifier and SHALL leave the recorded markup amount and the recorded applied rate unchanged.
6. WHEN Take_Rate_Calculator computes the markup amount for one checkout, THE AG_Core SHALL leave the amount requested from the settlement provider for that checkout equal to the amount recorded for that checkout before the computation.
7. IF the configured markup rate is absent, non-numeric, negative, zero, or greater than 1000 basis points, THEN THE AG_Core SHALL refuse to start, SHALL report the configuration key name and the failing condition, and SHALL append zero Revenue_Ledger lines.
8. IF the markup computation or the Revenue_Ledger append for one settlement fails on 3 consecutive attempts within 5 seconds, THEN THE Checkout_Session SHALL record the settlement as settled, SHALL record one typed markup-deferred Evidence_Event naming the settlement identifier and the failing stage, and SHALL leave the settlement outcome unchanged.
9. THE Revenue_Ledger SHALL record, per line, the settlement identifier, the agent identifier, the Settled_Amount in minor units, the settlement currency, the applied rate in basis points, the computed markup amount in minor units, and the recording instant in UTC at millisecond resolution.
10. WHEN the Platform Operator reads the Revenue_Ledger for a stated period bounded by an inclusive start instant and an exclusive end instant, THE AG_Core SHALL return every line whose recording instant falls within that period, ordered by recording instant ascending then settlement identifier ascending, and SHALL return the summed markup amount recomputed from those returned stored lines alone.

#### Stated Gap

11. THE Success Metric for Stream 1 SHALL remain unmet until at least one external principal distinct from the Platform Operator settles two or more checkouts, and THE AG_Core SHALL report the count of external principals who have settled two or more checkouts as the sole demand evidence for Stream 1.

### Requirement 4: White-Label Storefront Template Pack

**User Story:** As a small merchant, I want a polished, themed storefront deployed for my own catalog scope, so that I sell through a multi-vendor storefront without building one.

Traces to: PRD Monetization Model Stream 4; ADR-12 sequencing; PRD Component "Storefront Template Pack (White-Label)"; PRD "Merchant Happy-Path Journey"; PRD Success Metric "First Storefront Template Pack sold to a non-Joohwee merchant"; baseline `src/edge/dashboard.ts`.

Grounding chain: pain point — merchants want the storefront look and will not build it, demand `unvalidated` with outreach unblocked; solution — Theme_Manifest plus per-merchant deployment; feature — configuration and asset substitution over the built console at rank 4; monetization — Stream 4, setup fee plus recurring hosting fee, the fastest realistic path to a first dollar.

#### Acceptance Criteria

1. THE Template_Pack SHALL derive every merchant-visible difference — palette, logo, copy, and catalog scope — from a Theme_Manifest, and THE repository SHALL contain exactly one Storefront_Console implementation serving both the operator surface and every Template_Pack deployment.
2. WHEN a Theme_Manifest is submitted to AG_Core, THE AG_Core SHALL validate that Theme_Manifest within 5 seconds against the declared Theme_Manifest schema, which bounds each text field at 280 characters or fewer, the declared catalog scope at 1 through 500 agent identifiers, and the whole manifest at 64 kilobytes or smaller, and SHALL return exactly one of a pass result or a reject result naming every violating field.
3. IF a Theme_Manifest declares a catalog scope that includes an agent identifier absent from Agent_Registry, THEN THE AG_Core SHALL return a reject result naming that agent identifier, SHALL create no deployment, and SHALL leave any existing Template_Pack deployment for the submitting merchant identifier serving unchanged.
4. WHERE a Theme_Manifest omits an optional field, THE Template_Pack SHALL apply the schema-declared default for that field, and THE AG_Core SHALL record every field name resolved to a default.
5. WHEN a Template_Pack deployment serves a catalog request, THE Template_Pack SHALL return only listings whose owning agent identifier appears in the catalog scope recorded for that deployment, and SHALL return a not-found result for a request naming a listing outside that catalog scope.
6. THE Template_Pack SHALL complete first contentful paint within 2 seconds on a 360-pixel-wide viewport over a connection limited to 1.6 megabits per second, measured as the median of 5 consecutive cold loads.
7. THE Template_Pack SHALL express every interactive control as a native semantic HTML element carrying a non-empty accessible name, and SHALL present every touch target at 44 by 44 CSS pixels or larger.
8. THE Template_Pack SHALL expose exactly one checkout path per listing, and THE Template_Pack SHALL reach settlement through the same Checkout_Session route as the operator surface.
9. IF retrieval of a Theme_Manifest asset reference fails on 3 attempts within 30 seconds at build time, THEN THE AG_Core SHALL fail the deployment build, SHALL name every unreachable reference, and SHALL leave any previously deployed Template_Pack for the submitting merchant identifier serving the last successfully built configuration.
10. WHEN a Template_Pack deployment build succeeds, THE AG_Core SHALL record the merchant identifier, the Theme_Manifest digest, the resolved catalog scope, every field name resolved to a default, and the deployment instant in UTC.
11. THE Template_Pack SHALL introduce zero new infrastructure service categories beyond those already provisioned in the Dev_Lane configuration.

#### Stated Gap

12. THE Success Metric for Stream 4 SHALL remain unmet until at least one merchant who is not the Platform Operator pays for a Template_Pack deployment, and THE existence of a themeable deployment SHALL be reported as capability rather than as demand evidence.

### Requirement 5: Environment Chain and Deploy Boundaries

**User Story:** As the Platform Operator, I want one named path from Dev to the Cloudflare delivery route behind a protected controller, so that a sellable surface can ship and no unauthorized change can.

Traces to: operating-context environment chain Dev → Prod → Cloudflare; `START-WORKFLOW.md` deploy gate; `agentic-sdlc-guidelines.md` Global Release-Control Rule; PRD Deploy Boundary Register; baseline empty `routes` array in `wrangler.edge.jsonc`.

Grounding chain: pain point — an unreachable delivery surface cannot be sold, provable from the empty route list; solution — a named chain behind Release_Controller; feature — configuration plus controller wiring at rank 5; monetization — enabler for every stream's delivery surface.

#### Acceptance Criteria

1. THE repository SHALL expose exactly two Dev_Lane entry points, named `dev` and `dev:apex`, and THE Dev_Lane SHALL start through exactly those two entry points.
2. WHEN the Dev_Lane starts, THE Dev_Lane SHALL issue zero write requests to Prod_Mirror, zero write requests to Delivery_Route, and zero live-mode requests to any payment provider.
3. THE delivery configuration for the Production environment SHALL declare exactly one Delivery_Route entry reading `airvio.co/agentic-commerce-os`, and THE Production route list SHALL read non-empty.
4. THE Prod_Mirror SHALL be declared as `GitHub/huijoohwee/content/agentic-commerce-os`, and THE repository SHALL record Prod_Mirror as generated release output carrying zero authored edit targets.
5. THE Release_Controller SHALL be the only mechanism permitted to advance Prod_Mirror or Delivery_Route, and every Deploy_Boundary SHALL read `closed` or `pending-protected-integration` until a valid authorization for that exact Deploy_Boundary is recorded.
6. IF a deployment is requested absent a recorded authorization naming the requested candidate identity, the requested target, an authenticated human identity, and an authorization instant within the preceding 24 hours, THEN THE Release_Controller SHALL refuse the deployment, SHALL leave Prod_Mirror and Delivery_Route serving unchanged, and SHALL record the requested candidate, the requested target, and the missing authorization element.
7. WHEN the canonical source frontier advances after a candidate is sealed, THE Release_Controller SHALL mark that sealed candidate `retired`, SHALL refuse every deployment naming a `retired` candidate, and SHALL require a newly sealed candidate bound to the advanced frontier.
8. THE Release_Controller SHALL record one rollback disposition per deployment, naming the prior released candidate identity and the restore action, before that deployment proceeds, and SHALL refuse a deployment carrying zero recorded rollback disposition.
9. THE AG_Edge SHALL report the deploy lane name and the release candidate identity of the serving deployment on `GET /livez` and on Storefront_Console, and SHALL report zero secret values on either surface.
10. THE Readiness_Report SHALL report source-level readiness and live-release readiness as two separately named fields, and SHALL report live-release readiness as not-ready until Delivery_Route serves the release candidate named by the most recent authorized deployment.

### Requirement 6: One Invocation Surface for Every New Capability

**User Story:** As an agent and as the Platform Operator, I want every new capability reachable through the one existing `/`, `#`, `@` invocation surface, so that no second, divergent command registry appears.

Traces to: `AGENTS.md` rule that the three dictionaries are the only token authority; operating-context MCP-native and WebMCP-native requirement; baseline `src/invocation/catalog.ts` and `src/invocation/index.ts`; PRD Dependencies "Invocation Surface Contract".

Grounding chain: pain point — a fragmented entry surface, provable from the singular pinned catalog that must stay singular; solution — one Invocation_Catalog owning every new token; feature — extension of a built catalog at rank 6; monetization — enabler for agent-facing discovery in Streams 1 and 3.

#### Acceptance Criteria

1. THE Invocation_Catalog SHALL be the only source from which AG_Core resolves an Invocation_Token, and THE repository SHALL contain zero token registry other than the three `/`, `#`, and `@` dictionaries projected by Invocation_Catalog.
2. WHEN a capability in this document gains an operator or agent action, THE Invocation_Catalog SHALL carry exactly one `/` command token, at least one `#` semantic token, and at least one `@` binding token for that action before that action becomes reachable on any surface.
3. WHEN AG_Core resolves an Invocation_Token, THE AG_Core SHALL return, within the same read, the resolved entry, the catalog source revision, the catalog digest computed over the served entry set, and three per-sigil counts covering `/`, `#`, and `@`.
4. IF a presented Invocation_Token is absent from the Invocation_Catalog or carries a leading character other than `/`, `#`, or `@`, THEN THE AG_Core SHALL return a typed unresolved result naming the presented token, SHALL invoke zero downstream capability, and SHALL leave stored state unmodified.
5. IF the Invocation_Catalog cannot be hydrated from the pinned catalog source, THEN THE Readiness_Report SHALL read not-ready, SHALL record a catalog-hydration reason naming the pinned catalog source, and THE AG_Core SHALL return a typed catalog-unavailable result for every presented Invocation_Token until hydration succeeds.
6. THE AG_Core SHALL resolve an Invocation_Token deterministically with zero model call, SHALL return the identical result for the identical token and the identical catalog revision, and SHALL complete each resolution within 50 milliseconds of receiving the token.
7. THE AG_Edge SHALL expose every capability in this document through the MCP tool surface at `POST /mcp` in addition to any HTTP route, such that every capability action is reachable through the `/` command token recorded in Invocation_Catalog and zero capability action is reachable only through an HTTP route.
8. WHEN the Invocation_Catalog source revision changes, THE AG_Core SHALL recompute the catalog digest and the three per-sigil counts, and SHALL report the prior revision and the new revision, before serving any resolution against the new revision.

### Requirement 7: WebMCP-Native Storefront Tool Surface

**User Story:** As a visiting WebMCP-capable agent, I want to discover and call the storefront's own tools from the page, so that I transact without a bespoke backend integration.

Traces to: PRD ADR-11; PRD Component "Storefront WebMCP Tool Surface"; PRD ADR-13 update decoupling build from ship; PRD Success Metric "Storefront tools discoverable by a WebMCP-capable agent"; PRD MoSCoW row placing production shipping in `Won't (this increment)`.

Grounding chain: pain point — bespoke per-agent integration cost blocks third-party agents; solution — browser-native tool registration reusing existing client functions; feature — new browser surface over built client actions at rank 7; monetization — Stream 1 volume expansion and Stream 3 groundwork.

#### Acceptance Criteria

1. WHERE the loading browser exposes a model-context registration API, WHEN Storefront_Console completes loading, THE WebMCP_Tool_Surface SHALL register at least one catalog-search tool, at least one offer-selection tool, and at least one guardrailed-checkout-initiation tool, SHALL cap the registered tool set at 16 tools, and SHALL complete registration within 2000 milliseconds of load completion.
2. THE WebMCP_Tool_Surface SHALL implement every registered tool by calling the same client function the visual control calls, and SHALL introduce zero backend endpoint that exists only for the tool surface.
3. IF the loading browser exposes no model-context registration API, THEN THE Storefront_Console SHALL render every visual control that Storefront_Console renders in a registration-capable browser, SHALL keep every rendered visual control operable, SHALL surface zero registration-failure notice to the shopper, and SHALL record one typed unavailable-surface Evidence_Event naming the absent registration API.
4. WHEN a registered tool is invoked, THE WebMCP_Tool_Surface SHALL evaluate the identical guardrail set in the identical order the visual surface evaluates for the same action, SHALL obtain human confirmation through the visual surface before every payment-adjacent call, and SHALL accept zero tool-supplied value in place of that human confirmation.
5. THE WebMCP_Tool_Surface SHALL compute the registered tool set digest over every registered tool name, every registered input schema, every registered output schema, and the registered tool count, SHALL record that digest at registration, and SHALL verify, before each registered tool executes, that the currently computed digest equals the recorded digest.
6. IF the currently computed registered tool set digest differs from the digest recorded at registration, THEN THE WebMCP_Tool_Surface SHALL refuse the invocation, SHALL return to the calling agent a typed registration-drift refusal naming the invoked tool, SHALL record one typed registration-drift Evidence_Event, SHALL invoke zero payment-adjacent call, and SHALL leave every Checkout_Session record unchanged.
7. THE WebMCP_Tool_Surface SHALL expose zero credential value, zero card identifier, zero card token, and zero authentication token through any registered tool's input schema, output schema, or refusal payload.
8. THE WebMCP_Tool_Surface SHALL be developed and tested inside Sandbox_Executor isolation, and THE Deploy_Boundary for the WebMCP_Tool_Surface SHALL read `closed` for Delivery_Route until the refusal behavior in criterion 6 is recorded as passing under Sandbox_Executor and the supported browser set is recorded by browser name and minimum browser version.

#### Stated Gap

9. THE repository SHALL record the model-context standard as origin-trial stage with partial, evolving browser support, and SHALL report the WebMCP_Tool_Surface readiness rung as `spec-complete` or `dev-proven` rather than as delivered until the Deploy_Boundary in criterion 8 reads open for Delivery_Route.

### Requirement 8: Neutral Aggregation Routing and Public Catalog

**User Story:** As an Agent Builder, I want the platform to route on price, quality, and latency and to list my agent publicly, so that routing through the platform is worth more to me than routing around it.

Traces to: PRD ADR-10; PRD ADR-1; PRD Component Inventory rows "Agent Registry/Router — smart/neutral routing + fallback dispatch" and "Marketplace Registry Canvas — public catalog view"; PRD Open Question on classifier versus fixed enum; baseline `src/domain/exclusive-category-router.ts`.

Grounding chain: pain point — a fixed-enum router gives an agent no reason to route through this platform; solution — declared-attribute selection with typed fallback plus a public catalog; feature — new selection policy over a built deterministic router at rank 8; monetization — Stream 1, since markup scales only with routed volume.

#### Acceptance Criteria

1. WHEN Intent_Router resolves a typed intent to two or more eligible registered agents, THE Intent_Router SHALL select exactly one agent by applying the externalized selection policy over the eligible agents' declared price, quality, and latency attributes, and SHALL resolve a tie in policy score by ascending agent identifier.
2. WHEN Intent_Router completes a routing decision, THE Intent_Router SHALL record the eligible agent identifiers, the selected agent identifier, and the declared attribute values that produced the selection.
3. THE Intent_Router SHALL issue exactly one dispatch per typed intent, and SHALL issue zero dispatch to a non-selected eligible agent for that same typed intent.
4. IF the selected agent returns no result within 30 seconds of dispatch, THEN THE Intent_Router SHALL dispatch to at most one declared fallback agent, SHALL record the fallback decision with a timeout reason, and SHALL issue zero further dispatch to the selected agent for that typed intent.
5. IF the declared fallback agent returns no result within 30 seconds of dispatch, or zero fallback agent is declared, THEN THE Intent_Router SHALL return a typed failure result carrying an exhausted-dispatch reason and SHALL issue zero further dispatch for that typed intent.
6. IF zero eligible agents match a typed intent, THEN THE Intent_Router SHALL return a typed no-match result carrying an unmatched reason and SHALL issue zero dispatch.
7. WHEN Intent_Router receives a typed intent, THE Intent_Router SHALL resolve the selection without a model call within 200 milliseconds, and SHALL return the identical selection for identical declared attributes and identical registration state.
8. THE Public_Catalog_View SHALL project, without authentication, each registered agent whose registration state is active by identifier, declared category, declared capabilities, and declared trust status, and SHALL project zero operator-only field, zero credential value, and zero internal binding identity.
9. WHEN Agent_Registry state changes, THE Public_Catalog_View SHALL project the changed state within 5 seconds for registries containing up to 500 registered agents.
10. THE Public_Catalog_View SHALL report the projected set of agent identifiers as equal to the set of agent identifiers whose registration state is active in Agent_Registry for the same read revision.

### Requirement 9: Local-First, Offline-First, and Multi-Device Concurrent Authoring

**User Story:** As a shopper on a phone and as an operator on a second device, I want the surfaces to keep working while connectivity is degraded and to converge without losing my work, so that a dropped connection or a second device never destroys state.

Traces to: operating-context browser-based, mobile-first, on-device, local-first, offline-first, and multi-device concurrent authoring requirements; PRD Topology data residency "Local (device) + Edge cache"; PRD ADR-3 CRDT reuse; `START-WORKFLOW.md` one-active-writer-per-worktree and fencing rules.

Grounding chain: pain point — lost state on a dropped mobile connection and corrupted state across two devices; solution — local-first state with ordered replay and one active writer per Authoring_Lane; feature — new client state layer over already-contracted lane rules at rank 9; monetization — enabler for the mobile demos Stream 4 sells through.

#### Acceptance Criteria

1. WHILE a client has no network connectivity, THE Storefront_Console SHALL render the state of the last completed synchronization from local storage without issuing a network request, and SHALL present a typed offline indicator within 1 second of the first failed request.
2. WHILE a client has no network connectivity, THE Storefront_Console SHALL record each locally originated change in local storage in origination order for at least 500 such changes, and SHALL refuse a further locally originated change with a typed capacity reason rather than discard a recorded change.
3. WHEN network connectivity returns, THE Storefront_Console SHALL begin submitting locally recorded changes within 5 seconds in recorded first-to-last order, and SHALL retain each recorded change in local storage until that change is acknowledged.
4. WHEN two synchronization sequences carrying changes for the same state are applied in either relative order, THE AG_Core SHALL produce identical resulting state and SHALL discard zero change carried by either sequence.
5. WHILE a client has no network connectivity, THE Storefront_Console SHALL invoke zero settlement call against any payment provider and SHALL present a typed reason naming absent connectivity as the blocking cause.
6. THE Storefront_Console SHALL render every primary-surface control and text region at a viewport width of 360 CSS pixels with zero horizontal scrolling, zero clipped content, and zero overlapping text.
7. WHEN a second Concurrent_Author attempts to mutate a semantic scope already held by a current claim, THE AG_Core SHALL apply zero byte of that mutation, SHALL return a typed conflict result to that Concurrent_Author, and SHALL record the holding claim identity, the holding lease epoch, and the holding fence revision.
8. THE AG_Core SHALL permit concurrent mutation across Concurrent_Authors only where the declared write sets share zero element and each Concurrent_Author holds one claim whose lease is unexpired at the current fence revision.
9. IF a Concurrent_Author's lease has passed the declared lease duration for that Concurrent_Author, THEN THE AG_Core SHALL refuse that author's mutation, SHALL leave that author's recorded bytes unchanged, SHALL return a typed expired-lease reason, and SHALL require a new claim before accepting a further mutation from that author.
10. THE AG_Core SHALL introduce zero new infrastructure service category to satisfy this requirement beyond those already provisioned in the Dev_Lane configuration.

### Requirement 10: Held-Offer Change Detection

**User Story:** As a shopper, I want to be told before I confirm when a price, an availability, or an agent's registration has changed since the offer was shown, so that I never confirm a stale offer.

Traces to: PRD Breakthrough Rubric Assessment (pre-L1 finding for this document's own domain object); PRD Component "Agent State-Change Listener"; PRD ADR-6 reuse of the dependency-graph primitives; baseline Checkout_Session evidence event log.

Grounding chain: pain point — nothing watches a held offer after dispatch, provable from the PRD's own rubric self-assessment; solution — Offer_Change_Listener emitting typed change events into Checkout_Session; feature — new component reusing the built session evidence log at rank 10; monetization — protects Stream 1 revenue integrity.

#### Acceptance Criteria

1. WHILE a Held_Offer exists in an open Checkout_Session, THE Offer_Change_Listener SHALL observe that offer's recorded price, recorded availability, and originating agent registration state, beginning within 60 seconds of the offer being shown and repeating at an interval of at most 60 seconds.
2. WHEN an observed Held_Offer attribute value differs from the value recorded for that attribute when the offer was shown, THE Offer_Change_Listener SHALL append exactly one typed change Evidence_Event to that Checkout_Session naming the attribute, the recorded value, the observed value, and the observation instant.
3. WHEN a typed change Evidence_Event or a typed observation-suspended Evidence_Event exists for a Held_Offer, THE Checkout_Session SHALL invalidate every prior human confirmation for that offer and SHALL refuse settlement for that offer until a new human confirmation naming that Evidence_Event is recorded.
4. IF the originating agent of a Held_Offer becomes inactive in Agent_Registry, THEN THE Checkout_Session SHALL refuse settlement for that offer, SHALL record exactly one typed agent-inactive Evidence_Event naming that agent identifier, and SHALL retain that offer's recorded attribute values unchanged.
5. WHEN a Held_Offer is settled or that offer's Checkout_Session closes, THE Offer_Change_Listener SHALL stop observing that offer within 60 seconds and SHALL record the observation stop instant and the typed stop reason.
6. IF an observation attempt fails, THEN THE Offer_Change_Listener SHALL retain the last recorded attribute values, SHALL record exactly one typed observation-failed Evidence_Event, and SHALL retry at most 3 times at an interval of at most 60 seconds before recording one typed observation-suspended Evidence_Event.
7. THE Offer_Change_Listener SHALL record at most one change Evidence_Event per attribute per distinct observed value, such that an observation returning the most recently observed value appends zero further Evidence_Event.
8. THE Offer_Change_Listener SHALL observe deterministically and SHALL issue zero model call and zero paid call per observation other than the provider read required for the three observed attributes.

#### Stated Gap

9. THE Offer_Change_Listener SHALL satisfy detection and re-confirmation only, and THE repository SHALL record autonomous re-derivation of dependent cart lines and real-time re-settlement of the markup delta as unbuilt, deferred to the reused dependency-graph primitives named in ADR-6.

### Requirement 11: Isolated Execution for Untrusted Input and Pre-Registration Dry Runs

**User Story:** As the Platform Operator, I want untrusted merchant input and unvetted agent tool calls to run in isolation, so that neither reaches the shared runtime before it is trusted.

Traces to: PRD ADR-13; PRD Component "Cloudflare Sandbox SDK Build/QA Layer"; PRD ADR-2 stated runtime-sandboxing gap; `agentic-sdlc-guidelines.md` Tool Permission and Blast Radius.

Grounding chain: pain point — untrusted theme input and unvetted tool calls have nowhere safe to run; solution — Sandbox_Executor for merchant builds, dry runs, and unshipped-surface work; feature — new layer consuming Requirement 4 and Requirement 7 outputs at rank 11; monetization — Stream 4 support cost containment.

#### Acceptance Criteria

1. WHEN a Theme_Manifest build is requested, THE Sandbox_Executor SHALL execute that build inside one isolated instance, SHALL grant that instance zero write access to shared runtime state, and SHALL return one typed build result reading `completed` or `failed`.
2. WHEN a registration dry run is requested for a submitted agent definition, THE Sandbox_Executor SHALL execute the declared tool calls inside one isolated instance and SHALL record each attempted call, the target tool identifier, and the outcome of that call.
3. IF a dry run attempts a call outside the submitted definition's declared allowlist, THEN THE Sandbox_Executor SHALL refuse that call, SHALL record the attempted call, and SHALL return a reject result for the registration.
4. THE Sandbox_Executor SHALL enforce per instance one configured wall-clock limit of at most 300 seconds and one configured memory ceiling of at most 512 megabytes, and SHALL terminate an instance exceeding either limit within 5 seconds of the excess while recording the exceeded limit and the configured value.
5. WHEN an isolated instance terminates, THE Sandbox_Executor SHALL record the instance identifier, the purpose, the start instant, the end instant, and one outcome reading `completed`, `refused`, `limit-exceeded`, or `failed`.
6. THE Sandbox_Executor SHALL grant an isolated instance zero access to any payment credential, any settlement provider binding, and any operator authority.
7. WHERE a preview surface is requested for a merchant engagement, THE Sandbox_Executor SHALL expose one addressable preview identity scoped to that engagement, and SHALL revoke that identity at the configured lifetime of at most 24 hours such that a later request to that identity returns a typed revoked result.
8. IF an isolated instance fails to provision or fails during execution, THEN THE Sandbox_Executor SHALL return one typed blocked result for the dependent build, SHALL leave shared runtime state unchanged, and SHALL report the Sandbox_Executor readiness rung as `dev-proven` rather than as delivered.

### Requirement 12: Bounded Agent Merge Automation

**User Story:** As the solo operator, I want an agent to address review comments, repair failing required checks, and resolve conflicts on my own lane, so that my scarcest resource is spent on decisions rather than mechanics.

Traces to: operating-context agent merge requirement; `agentic-sdlc-guidelines.md` Autonomous Continuation and Interaction Economy, Human-in-the-Loop Gates, Per-Task Budgets, and Global Release-Control Rule; `START-WORKFLOW.md` one-active-writer and fencing rules.

Grounding chain: pain point — review, check-repair, and conflict mechanics consume the operator's scarcest resource; solution — Merge_Agent acting on one owned lane within stated bounds; feature — new automation over the contracted lane model at rank 12; monetization — enabler for sprint cadence rather than revenue.

#### Acceptance Criteria

1. WHEN a review comment is recorded on an Authoring_Lane whose current lease and current fence revision are held by the Merge_Agent, THE Merge_Agent SHALL produce at most one change addressing that comment within the same run, or SHALL record one typed reason drawn from `requires-operator-decision`, `out-of-write-set`, or `scope-gap`.
2. WHEN a required check fails on that Authoring_Lane, THE Merge_Agent SHALL record the failing check identity and the observed failure output, SHALL produce one repair per attempt, SHALL rerun that same check after each attempt, and SHALL perform at most two attempts for that failing check.
3. WHEN a merge conflict exists between that Authoring_Lane and the canonical frontier, THE Merge_Agent SHALL resolve that conflict inside the declared write set of that Authoring_Lane only, SHALL preserve every byte outside that write set unchanged, and SHALL leave the conflict unresolved with a recorded `out-of-write-set` reason where resolution requires a change outside that write set.
4. IF two consecutive attempts using the same repair approach leave the same required check failing, THEN THE Merge_Agent SHALL stop that approach, SHALL record the diagnosed root cause, and SHALL either apply exactly one different approach or record one escalation to the Platform Operator and end the run.
5. THE Merge_Agent SHALL record, before the first repair of a run, one numeric token bound, one iteration bound of at most 10 mutating actions per run, one wall-clock bound of at most 30 minutes per run, and one circuit-breaker condition, and SHALL end the run with zero further mutation when any recorded bound is reached or the circuit-breaker condition holds.
6. IF a review comment, a check failure, or a conflict requires behavior absent from this document, THEN THE Merge_Agent SHALL end the run with zero further mutation, SHALL record one typed `scope-gap` reason naming the absent behavior, and SHALL return that gap to the requirements phase.
7. THE Merge_Agent SHALL perform zero canonical write, zero force update, zero history rewrite, and zero deployment, and SHALL reach the canonical frontier only through Release_Controller.
8. THE Merge_Agent SHALL mutate exactly one Authoring_Lane per run, SHALL hold that lane's current lease and current fence revision for the duration of every mutation, SHALL refuse every mutation once that lease or fence revision is no longer current, and SHALL leave every other lane's recorded bytes unchanged.
9. THE Merge_Agent SHALL record, per action and in append-only recorded order, the acting identity, the target lane, the action taken, the bound consumed, and the resulting check outcome.

### Requirement 13: Bounded, Independently Verified Execution Evidence and Demo Walkthrough

**User Story:** As the Orchestrator, I want every implementation task bounded, independently verified, and demonstrable, so that a completion claim is earned from surfaced evidence rather than asserted.

Traces to: `agentic-sdlc-guidelines.md` Execution Contract, Verification Strategy, Per-Task Budgets, Independence Rule, and Validation Checklist; this document's Deliverable Set; PRD Readiness rung reporting discipline.

Grounding chain: pain point — an asserted completion earns no readiness rung and demonstrates nothing to a buyer; solution — named check plus independent verdict per task and one Demo_Walkthrough per increment; feature — process obligation over the built three-lane test harness at rank 13; monetization — enabler, since evidence is what a buyer and an auditor both read.

#### Acceptance Criteria

1. THE task list derived from this document SHALL state, per task, one named check phrased in the exact form the check is invoked, before that task is dispatched.
2. THE task list SHALL state, per task, one token bound as a positive integer ceiling, one iteration bound as a positive integer ceiling, one wall-clock bound as a positive integer ceiling in minutes, one context bound as a positive integer ceiling, and one circuit-breaker condition expressed as an observable halting signal.
3. WHEN a task completes, THE verdict for that task SHALL be issued by a mechanism distinct from the mechanism that performed the task, and THE verdict record SHALL name the performing mechanism identity and the verdict-issuing mechanism identity.
4. IF a verdict for a task is derived from output the performing mechanism did not surface, THEN THE Orchestrator SHALL reject that verdict and SHALL record a self-graded-verdict finding.
5. WHEN a task completes, THE Orchestrator SHALL run the repository's existing verification lane in addition to that task's own named check, and SHALL withhold the completion claim for that task until the existing verification lane and that task's own named check each record a pass.
6. THE Orchestrator SHALL emit one Evidence Reference per satisfied VCC, carrying the named check, the recorded result, and the readable surface bearing that recorded result, and SHALL emit zero Evidence Reference for a check that was not run and zero Evidence Reference for a check that recorded a failure.
7. THE repository SHALL keep every authored file at 600 lines or fewer, and SHALL split a file exceeding that ceiling along one owner boundary or one behavior boundary such that the behavior of every split part taken together equals the behavior recorded before the split.
8. THE repository SHALL persist zero developer-specific absolute path, zero credential value, and zero account identifier in source, fixtures, tests, generated assets, and documentation.
9. THE Demo_Walkthrough SHALL present one runnable path through the shipped increment, covering the settled-markup path in Requirement 3 and the merchant deployment path in Requirement 4, using Dev_Lane commands only and carrying zero command targeting Prod_Mirror and zero command targeting Delivery_Route.
10. THE Demo_Walkthrough SHALL record, per step, the command in the exact form the command is invoked, the expected observable outcome, and the Deploy_Boundary state recorded at that step.
11. THE Orchestrator SHALL report the readiness rung derived solely from emitted Evidence References, SHALL report a source-level pass as distinct from a live Production release, and SHALL advance zero readiness rung while an unresolved self-graded-verdict finding recorded under criterion 4 exists.

## Correctness Properties

Each property names its class per `agentic-sdlc-guidelines.md` Property-Based Obligations, states its minimum iteration count, and keeps shrinking enabled. A property run once is an example test wearing a costume, so a single-execution obligation appears in Integration and Example Checks instead.

| ID | Class | Covers | Property statement | Min iterations |
|---|---|---|---|---:|
| CP-1 | Metamorphic | 1.5, 1.6, 1.8 | For all requests exercising a renamed endpoint or tool identity, substituting the Superseding_Identifier for the Legacy_Identifier while holding request content fixed leaves the response body and status unchanged, and the legacy form yields a typed rejection naming the superseding form | 200 |
| CP-2 | Invariant | 2.1, 2.2, 2.3, 2.7 | For all advertised check sets, adding a passing check outside the Required_Check_Set never turns `converged` into `blocked`, and removing any Required_Check_Set member always yields `blocked` naming that member | 500 |
| CP-3 | Invariant | 2.10 | For all advertised evidence envelopes and declared requirement sets, two evaluations of the same pair yield the identical Convergence_Verdict including identical recorded reasons in identical order | 300 |
| CP-4 | Error condition | 2.3, 2.5, 2.6, 2.8 | For all advertised envelopes and declared revisions, Convergence_Evaluator returns a convergent verdict only where the advertised major component equals the declared major component and the advertised minor component is no lower than the declared minor component, and otherwise returns `blocked` naming every absent or failing Required_Check_Set member, every violated declared identity or integrity field, and both revision values | 400 |
| CP-5 | Invariant | 3.1, 3.4, 3.6 | For all Settled_Amount and rate pairs, the markup is an integer in minor units no less than 0 and no greater than the Settled_Amount, matches half-up rounding of the basis-point rate applied to the Settled_Amount, and the provider-requested amount equals the amount recorded before the computation | 500 |
| CP-6 | Idempotence | 3.2, 3.5 | For all settlement identifiers, recording the same settlement twice yields the identical Revenue_Ledger state as recording it once, with exactly one line and an unchanged markup amount and applied rate | 300 |
| CP-7 | Invariant | 3.9, 3.10 | For all settlement sequences and period bounds, the returned line set equals the stored lines whose recording instant falls in that period in the declared order, the summed markup equals the sum recomputed from those returned lines, and every returned line carries a complete field set | 300 |
| CP-8 | Round trip | 4.1, 4.2, 4.4 | For all valid Theme_Manifest values, serialize-parse-serialize yields an equivalent manifest, the validation verdict is identical across both representations, and the defaulted-field set is identical | 300 |
| CP-9 | Invariant | 4.3, 4.5 | For all Theme_Manifest and Agent_Registry pairs, every listing a Template_Pack deployment returns lies inside that merchant's declared catalog scope, and a manifest naming an absent agent identifier yields a reject result and zero deployment | 300 |
| CP-10 | Invariant | 6.1, 6.3, 6.4, 6.6 | For all token strings, AG_Core returns either exactly one resolved entry carrying catalog revision, digest, and per-sigil counts, or exactly one typed unresolved result naming the token, invoking zero downstream capability in the unresolved case | 400 |
| CP-11 | Idempotence | 6.6, 6.8 | For all tokens, resolving the same token twice against the same catalog revision yields the identical result including identical digest and counts | 200 |
| CP-12 | Metamorphic | 7.4, 7.7 | For all checkout actions, invoking through WebMCP_Tool_Surface and invoking the equivalent visual-surface action produce identical guardrail and confirmation event ordering, and neither payload carries a credential-shaped field | 300 |
| CP-13 | Error condition | 7.5, 7.6 | For all tool-set mutations after registration, an invocation whose current tool-set digest differs from the registration digest is refused, records a registration-drift event, and issues zero payment-adjacent call | 300 |
| CP-14 | Invariant | 8.1, 8.3, 8.4, 8.5, 8.6 | For all typed intents and registration states, dispatch count is exactly one when an agent is eligible and no fallback fires, exactly two when one declared fallback fires, and exactly zero when no agent is eligible, and an exhausted or unmatched outcome yields exactly one typed result | 500 |
| CP-15 | Invariant | 8.1, 8.2, 8.7 | For all eligible-agent attribute sets, two selections over identical attributes and identical policy yield the identical selected agent identifier and identical recorded attribute values, and equal policy scores resolve to the lowest agent identifier in ascending order | 300 |
| CP-16 | Invariant | 8.8, 8.10 | For all Agent_Registry states, the identifier set Public_Catalog_View projects equals the active identifier set for the same read revision, and the projection carries zero operator-only, credential, or internal binding field | 300 |
| CP-17 | Confluence | 9.4 | For all pairs of concurrent synchronization sequences over the same state, applying the pair in either order yields identical converged state | 300 |
| CP-18 | Invariant | 9.2, 9.3 | For all sequences of changes recorded while offline, the submission order on reconnection equals the recorded order and zero recorded change is dropped | 300 |
| CP-19 | Invariant | 9.7, 9.8, 9.9 | For all pairs of Concurrent_Author claims, a mutation is admitted only where declared write sets are disjoint and the acting author holds a current claim, and a refused mutation leaves the holding author's recorded bytes unchanged | 300 |
| CP-20 | Idempotence | 10.2, 10.7 | For all observation sequences over a Held_Offer, repeating an unchanged observation appends zero further change Evidence_Event, and each distinct attribute difference appends exactly one | 300 |
| CP-21 | Invariant | 10.3, 10.4 | For all Checkout_Session event sequences, zero settlement call is preceded by an unresolved change Evidence_Event for that offer, and zero settlement call exists for an offer whose originating agent is inactive | 500 |
| CP-22 | Invariant | 11.2, 11.3, 11.6 | For all agent definitions and attempted call sequences, every call executed inside an isolated instance is a member of the submitted declared allowlist, any out-of-allowlist attempt yields a reject result, and zero payment credential or settlement binding is reachable from the instance | 400 |
| CP-23 | Error condition | 11.4, 11.5 | For all workloads exceeding a configured wall-clock or resource limit, the isolated instance terminates, the exceeded limit is recorded, and a complete instance record is written | 200 |
| CP-24 | Invariant | 12.3, 12.7, 12.8 | For all conflict states, every byte the Merge_Agent changes lies inside the declared write set, every byte outside that write set is unchanged, and zero canonical write, force update, history rewrite, or deployment occurs | 300 |

## Integration And Example Checks

Per the source guidance on where property-based testing does not apply, the following criteria are verified by bounded example, integration, static, or process checks rather than by property tests. Behavior that does not vary meaningfully with input, or that tests an external service rather than this platform's own logic, belongs here.

| Criterion | Check kind | Rationale |
|---|---|---|
| 1.1, 1.2, 1.3, 1.4, 1.7, 1.9, 1.10 Terminology register scope and completeness, disposition detail, `renamed` end-state, prefix singularity with dispositioned exceptions, enforcement check determinism and per-occurrence naming, externally-owned upstream targets | Repository scan check, single execution | Static repository property; 100 iterations find nothing 1 cannot |
| 2.4 Unretrievable or incomplete provider evidence at the 5-second bound | Integration, 1 example per unretrievable and per incomplete case | External provider retrieval timing |
| 2.9, 2.11 Readiness_Report per-provider verdict composition and source-level versus live-release field separation | Integration, 1 example per provider verdict | External provider probe composition |
| 3.3, 3.7 Basis-point rate read from an `AG_`-prefixed key, zero hardcoded rate, and startup refusal per invalid-rate class | Repository scan plus integration, 1 example per invalid class | Startup configuration gate, single execution |
| 3.8 Markup-deferred path after 3 consecutive failures | Integration, 1 example | Failure path against a bounded computation |
| 3.11 External-principal count reporting | Integration, 1 example | Metric reporting shape, not input-varying |
| 4.6 Template_Pack first contentful paint | Browser measurement, 5 cold loads, median reported | Deterministic performance measurement |
| 4.7 Semantic elements, non-empty accessible names, touch targets | Static and DOM assertion | Structural conformance, not input-varying |
| 4.8 Exactly one checkout path per listing through the shared Checkout_Session route | Integration, 1 example | Route-identity check |
| 4.9 Unreachable asset build failure after 3 attempts, prior deployment preserved | Integration, 1 example | External retrieval failure |
| 4.10, 4.11 Deployment record fields and zero new service category | Integration plus dependency inventory, 1 each | Recorded fact and static inventory |
| 4.12 Stream 4 metric reporting | Integration, 1 example | Metric reporting shape |
| 5.1–5.10 Two named Dev_Lane entry points, zero Prod_Mirror and Delivery_Route writes, declared non-empty Production route, generated-output disposition, boundary states, authorization refusals, candidate retirement, rollback disposition, lane and candidate reporting, readiness field separation | Integration, 1 example per boundary and per refusal path | External gate state; not input-varying |
| 6.2 Token coverage per capability action | Repository scan check | Static catalog-coverage property |
| 6.5 Catalog hydration failure and catalog-unavailable results | Integration, 1 example | External hydration failure |
| 6.7 MCP tool exposure per capability with zero HTTP-only action | Integration, 1 example per capability | Surface-identity check |
| 7.1 Tool registration, 16-tool cap, and 2000-millisecond registration bound on a capable browser | Browser integration, 1 example | External browser capability |
| 7.2 Zero tool-only backend endpoint | Repository scan check | Static repository property |
| 7.3 Full visual-control parity and unavailable-surface event on an incapable browser | Browser integration, 1 example | External browser capability |
| 7.8, 7.9 Sandbox-only development, recorded browser support set, and closed delivery boundary | Integration plus boundary state assertion, 1 each | Recorded gate state |
| 8.9 Public catalog projection latency | Integration, 1 example at 500 agents | Deterministic latency measurement |
| 9.1, 9.5, 9.6 Offline indicator within 1 second, settlement suppression, 360-pixel rendering | Browser integration, 1 example each | Connectivity state and deterministic layout |
| 9.10 Zero new service category | Dependency and configuration inventory check | Static property |
| 10.1 Observation start and repeat interval bounds | Integration, 1 example | Scheduling behavior, single execution |
| 10.5, 10.6 Observation stop with recorded reason and bounded retry to observation-suspended | Integration, 1 example each | Lifecycle and retry policy, bounded |
| 10.8, 10.9 Zero model call and zero extra paid call, and stated deferral | Cost assertion plus documentation assertion | Static properties |
| 11.1, 11.7, 11.8 Isolated build with typed result, scoped preview identity revoked at lifetime, provision-failure blocked result and rung disposition | Integration, 1 example each | External isolation layer behavior |
| 12.1, 12.2, 12.4, 12.5, 12.6, 12.9 Merge_Agent review handling with typed reasons, two-attempt check repair, repeated-failure stop, recorded bounds and circuit breaker, scope-gap stop, append-only action record | Process assertion, 1 example per path | Execution-contract conformance, not input-varying |
| 13.1–13.6, 13.11 Named check statement, positive-integer bounds and halting signal, dual mechanism identity in the verdict record, self-graded rejection, dual-pass completion claim, Evidence Reference emission with readable surface, rung derivation blocked by an unresolved finding | Process assertion, 1 example per task or per verdict | Mechanism-identity and dispatch-time governance |
| 13.7, 13.8 File size ceiling with behavior-preserving split, and absence of persisted paths, credentials, account identifiers | Repository scan check | Static repository property |
| 13.9, 13.10 Demo_Walkthrough runnability with zero Prod_Mirror and Delivery_Route commands, and per-step Deploy_Boundary state | Operator-runnable walkthrough, 1 execution | Single end-to-end demonstration |

## Traceability Matrix

| Requirement | Source | Baseline proximity | Component(s) | Properties |
|---|---|---|---|---|
| 1 | Naming authority; `AGENTS.md` no-alias rule; baseline legacy occurrences | Identifier-only | Invocation_Catalog, AG_Core, delivery configuration, docs | CP-1 |
| 2 | Operating-context tolerant-convergence rule; baseline exact-parity function; SDLC Runtime Readiness Enforcement | One function's rule | Convergence_Evaluator, Readiness_Report | CP-2, CP-3, CP-4 |
| 3 | PRD Stream 1; ADR-5; ADR-8; Take-Rate Calculator; Stream 1 Success Metric | One module on a built route | Take_Rate_Calculator, Checkout_Session, Revenue_Ledger | CP-5, CP-6, CP-7 |
| 4 | PRD Stream 4; ADR-12; Storefront Template Pack; merchant happy path; Stream 4 Success Metric | Configuration over a built console | Template_Pack, Storefront_Console, Theme_Manifest | CP-8, CP-9 |
| 5 | Operating-context environment chain; START-WORKFLOW deploy gate; SDLC Global Release-Control Rule; PRD Deploy Boundary Register | Configuration plus controller | Dev_Lane, Prod_Mirror, Delivery_Route, Release_Controller | integration checks |
| 6 | `AGENTS.md` dictionary authority; MCP-native and WebMCP-native context; PRD Invocation Surface Contract | Extension of a built catalog | Invocation_Catalog, AG_Edge, AG_Core | CP-10, CP-11 |
| 7 | PRD ADR-11; ADR-13 build/ship decoupling; WebMCP Success Metric; MoSCoW ship gate | New browser surface over built actions | WebMCP_Tool_Surface, Storefront_Console | CP-12, CP-13 |
| 8 | PRD ADR-10; ADR-1; Component Inventory smart-routing and public-catalog rows | New policy over a built router | Intent_Router, Public_Catalog_View, Agent_Registry | CP-14, CP-15, CP-16 |
| 9 | Operating-context local-first, offline-first, mobile-first, multi-device; PRD Topology residency; ADR-3; START-WORKFLOW lane rules | New client state layer | Storefront_Console, AG_Core, Authoring_Lane rules | CP-17, CP-18, CP-19 |
| 10 | PRD Breakthrough Rubric Assessment; Agent State-Change Listener; ADR-6 | New component on a built event log | Offer_Change_Listener, Checkout_Session, Agent_Registry | CP-20, CP-21 |
| 11 | PRD ADR-13; Sandbox Build/QA Layer; ADR-2 gap; SDLC blast radius | New layer over R4 and R7 outputs | Sandbox_Executor | CP-22, CP-23 |
| 12 | Operating-context agent merge; SDLC autonomous continuation, gates, budgets, release control; START-WORKFLOW fencing | New automation over contracted lanes | Merge_Agent | CP-24 |
| 13 | SDLC Execution Contract, Verification Strategy, Independence Rule, Validation Checklist; Deliverable Set | Process over a built test harness | Orchestrator, Demo_Walkthrough | evidence obligations only |

Coverage: 13 of 13 requirements trace to at least one source artifact. The five PRD user stories are covered as follows — US-1 registration-before-routing by Requirements 8 and 11, US-2 single-agent routing by Requirement 8, US-3 operator-visible registry by Requirement 8, US-4 guardrail and confirmation parity across agents by Requirements 7 and 10, US-5 issuance as sole payment caller by Requirements 3, 7, and 11. All four monetization streams are represented: Stream 1 by Requirement 3, Stream 4 by Requirement 4, and Streams 2 and 3 as recorded Non-Goals below.

The PRD user stories US-1 through US-3 were already reduced to acceptance criteria in the superseded spec at `.kiro/specs/knowgrph-agentic-commerce-platform` and are partially realized in the baseline (`AgentRegistry`, `IntentRoute`, and the ACOS admission path). This document restates only the increments those built surfaces do not yet satisfy — selection policy, public projection, and fallback dispatch — rather than re-deriving satisfied criteria.

## Non-Goals

Carried in intent from the PRD Out of Scope, MoSCoW `Won't (this increment)`, and Platform Roadmap Phase 2 and Phase 3 rows. Each is explicitly outside this requirements set.

1. Public third-party self-serve registration for strangers. The increment proves the primitive with operator-controlled agents; opening registration is a trust and abuse-surface question the source specification does not resolve.
2. On-chain trust or reputation attestation (PRD ADR-2; Platform Roadmap Phase 2).
3. Agent Builder registration or listing fee, Monetization Stream 2. Blocked on a customer segment that does not exist until third-party registration opens.
4. Issuance-as-a-Service usage pricing, Monetization Stream 3. Blocked on a committed customer that does not exist.
5. Production shipping of the WebMCP_Tool_Surface to Delivery_Route. Requirement 7 covers build, test, and isolation only; the delivery boundary stays closed.
6. Autonomous re-derivation of dependent cart lines and real-time re-settlement of the markup delta. Requirement 10 covers detection and re-confirmation only.
7. Migration of the Revenue_Ledger off the relational store named in ADR-8. The stated migration trigger is not met by this increment.
8. Multi-tenant fund segregation beyond the existing key-scoping. Named as an open question in the source specification.
9. A third net-new vertical. The increment deliberately proves the pattern with the verticals already contracted.
10. Any change to the settlement, issuance, or guardrail providers themselves. Every provider remains externally owned and is called, not modified.

## Open Questions Carried From The Source Specification

These are recorded rather than resolved. Each is an operator decision or an external dependency, and none is silently defaulted inside an acceptance criterion.

1. Does routing-by-declared-category need a model-based classifier, or is declared-attribute selection sufficient at the current agent count? Requirement 8 requires deterministic, model-free selection, so adopting a classifier is a scope decision that returns to this phase.
2. Where does the trust and verification boundary enforce — inside the registered agent, inside the router before dispatch, or through an external attestation? Requirement 11 provides isolated dry-run enforcement only.
3. Does each registered agent need its own funding source, or do all registered agents draw from one operator-controlled wallet? Unresolved; affects fund segregation, which is a Non-Goal here.
4. Will any Agent Builder pay a listing fee or accept a revenue share before third-party registration exists? Unresolved; Stream 2 is a Non-Goal until answered.
5. How far does browser support for the model-context standard extend by the time the WebMCP_Tool_Surface would ship, and is partial single-engine support acceptable for a first release? Unresolved; Requirement 7 keeps the delivery boundary closed until recorded.
6. Does the registration-overwrite exposure require mitigation on this platform's side beyond the digest check in Requirement 7.5, or is it accepted as a disclosed inherited risk? Requirement 7 mandates the digest check and leaves the broader disposition open.
7. Should the discovery adapter probe for browser-native tools first and fall back to a bespoke integration, or should per-merchant routing be configured explicitly? Unresolved; Requirement 8 requires explicit configuration, so runtime probing is a scope decision.
