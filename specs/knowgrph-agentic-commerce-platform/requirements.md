---
title: "Knowgrph Agentic Commerce Platform — Requirements"
doc_type: "Spec Requirements"
schema: "kiro-spec-requirements/v1"
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
source_specification: "joohwee/prd-tad-ard/knowgrph-agentic-commerce-platform-prd-tad-adr.md v0.1.0"
governing_contracts:
  - "huijoohwee.github.io/guidelines/agentic-sdlc-guidelines.md"
  - "agentic-canvas-os/docs/START-WORKFLOW.md"
  - "agentic-canvas-os/docs/AGENTS.md"
---

# Requirements Document

## Introduction

This document derives executable requirements for the Knowgrph Agentic Commerce Platform increment specified in `knowgrph-agentic-commerce-platform-prd-tad-adr.md` v0.1.0 (Phase 1 only). That specification introduces exactly three new components — Agent Registry/Router, Agent Definition Validator, and Marketplace Registry Canvas — and reuses the Guardrail Gate, Shared Canvas Node Store, Issuance Service, Settlement Verifier, and Notification Dispatcher unmodified from `knowgrph-agentic-travel-agencies-prd-tad-adr.md` v0.6.0.

Scope discipline: every requirement below traces to a named PRD user story, ADR, architecture section, or Deploy Boundary Register row. No requirement introduces behavior absent from the source specification; where the source states an honest gap, this document states the same gap as a bounded requirement rather than closing it silently.

Deploy boundary: all requirements are satisfied entirely within the Dev lane (`GitHub/knowgrph`, `npm run dev:apex`, `npm run dev`). The Prod mirror (`GitHub/huijoohwee/content/knowgrph`) and Cloudflare routes (`airvio.co`, `airvio.co/knowgrph`) are gated deploy targets and never acceptance criteria for any requirement in this document.

## Glossary

- **Agent_Registry**: The new Durable Object component that resolves a typed intent to exactly one registered agent and dispatches Discovery to that agent. PRD component: Agent Registry/Router.
- **Definition_Validator**: The new deterministic component that checks a submitted Agent Definition and its declared tool allowlist against the Invocation Surface Contract schema owned by `acos-agentic-runtime-ready-production-verified-prd-tad-adr.md`.
- **Registry_Canvas**: The new operator-scoped Yjs CRDT node that projects registered-agent state for the Platform Operator.
- **Guardrail_Gate**: The reused deterministic component that evaluates an offer against the principal's budget and produces a typed gate result. Reused unmodified.
- **Issuance_Service**: The reused MCP/SSE harness that is the sole caller of the StraitsX Card MCP Gateway. Reused unmodified.
- **Discovery_Harness**: A registered agent that receives a typed intent and returns typed offers. Two exist in this increment: Flight Discovery Harness and Shopping Discovery Harness.
- **Discovery input contract**: The named field set of a Typed_Intent that Agent_Registry is permitted to forward to a Discovery_Harness, defined by the Invocation Surface Contract. Fields outside this set, including any payment credential, card identifier, or card token, are not forwardable.
- **Discovery output contract**: The named offer-data field set a Discovery_Harness is permitted to return in a Typed_Offer, defined by the Invocation Surface Contract. The set excludes credential, card-number, card-token, and account-identifier fields.
- **Agent_Definition**: The typed record describing one registered agent: agent identifier, declared category, declared tool allowlist, and trust/verification status.
- **Agent_Definition_Table**: The authoritative persisted set of Agent_Definition records; the only source from which Agent_Registry may build routing entries.
- **Routing_Table**: The category-to-agent mapping Agent_Registry consults, derived only from Definition_Validator pass results.
- **Typed_Intent**: The Intent Parser output consumed by Agent_Registry, carrying a declared category field.
- **Typed_Offer**: The Discovery_Harness output consumed by Guardrail_Gate.
- **Session_Log**: The append-only ordered record of routing, gate, confirmation, issuance, and settlement events for one shopper session, used as evidence for verification.
- **Gate_Pass_Event**: A Session_Log entry recording that Guardrail_Gate returned a pass result for a specific offer.
- **Human_Confirm_Event**: A Session_Log entry recording an authenticated human confirmation for a specific offer.
- **No_Match_Result**: The explicit typed result Agent_Registry returns when no registered agent's declared category matches a Typed_Intent.
- **Operator_Scope**: The read scope that permits only the Platform Operator client to subscribe to Registry_Canvas state.
- **MCP invocation surface**: The established Model Context Protocol command surface of the Agentic Canvas OS through which Agent_Registry registration and routing operations are exposed. It is the single command registry for this increment; no parallel surface is defined.
- **Edge cache**: The edge-resident replica of Registry_Canvas state that a client synchronizes against through CRDT merge. Named in the PRD Topology data-residency column as "Local (device) + Edge cache".
- **Dev_Lane**: The Knowgrph development runtime at `GitHub/knowgrph`, started through repository-owned scripts.
- **Deploy_Boundary**: A recorded gate between lanes whose state must be `closed` or `pending-protected-integration` absent an explicit operator instruction.
- **Registration Deploy_Boundary**: The Deploy Boundary Register row "Agent Registration: declarative-allowlist → routable". It permits a Routing_Table entry to become routable only on a recorded Definition_Validator pass result for that exact entry; reversal is removal of that entry.
- **Mirror-to-Delivery Deploy_Boundary**: The Deploy Boundary Register row gating publication from the Prod mirror (`GitHub/huijoohwee/content/knowgrph`) to the Cloudflare delivery routes (`airvio.co`, `airvio.co/knowgrph`). It reports `closed` absent a recorded exact-candidate human authorization.
- **VCC**: Verifiable Completion Condition — a named check plus recorded result that an independent evaluator judges from surfaced output.

## Requirements

### Requirement 1: Registration Precedes Routability

**User Story:** As an Agent Builder, I want my Discovery agent registered against the Invocation Surface Contract before any shopper request can route to it, so that an unvetted agent never receives a live, payment-adjacent request.

Traces to: PRD US-1; ADR-2; Deploy Boundary Register row "Agent Registration: declarative-allowlist → routable".

#### Acceptance Criteria

1. WHEN Agent_Registry receives a Typed_Intent, THE Agent_Registry SHALL re-evaluate Agent_Definition_Table membership for the resolved agent identifier on that Typed_Intent, and SHALL dispatch Discovery only if that identifier is present in the Agent_Definition_Table at the moment of the dispatch decision.
2. IF a Typed_Intent resolves to an agent identifier absent from the Agent_Definition_Table, THEN THE Agent_Registry SHALL return No_Match_Result, SHALL NOT invoke Discovery or any downstream payment-adjacent call for that Typed_Intent, and SHALL record one typed rejection entry in the Session_Log carrying the Typed_Intent identifier, the resolved agent identifier, and a reason indicating the identifier is not registered.
3. WHEN an Agent_Definition is submitted for registration, THE Definition_Validator SHALL return, within 5 seconds of submission, exactly one of a pass result or a reject result, where a reject result carries at least one typed schema-violation reason and a pass result carries no violation reason.
4. THE Agent_Registry SHALL add a Routing_Table entry for an Agent_Definition only if that Agent_Definition is bound to a Definition_Validator pass result issued for the identical submitted definition content, and THE Agent_Registry SHALL make the entry routable only after that entry is committed.
5. IF an operator requests a Routing_Table change that is not bound to a Definition_Validator pass result for the same Agent_Definition content, THEN THE Agent_Registry SHALL reject the change, SHALL leave the Routing_Table unmodified, and SHALL record a rejection entry carrying the requesting operator identifier and a reason indicating the missing or non-matching pass result.
6. THE Definition_Validator SHALL evaluate submitted Agent_Definition records against the externally owned Invocation Surface Contract schema retrieved from its owning source at validation time, and SHALL persist no second copy of that schema beyond the duration of the validation it is retrieved for.
7. THE Definition_Validator SHALL record, for every reject result, the schema field identifier of each field that produced a violation, enumerating all violating fields detected in that submission rather than only the first.
8. IF the externally owned Invocation Surface Contract schema cannot be retrieved within 5 seconds of a validation attempt, THEN THE Definition_Validator SHALL return a reject result carrying a reason indicating the schema was unavailable, and THE Agent_Registry SHALL add no Routing_Table entry for that submission.
9. IF an Agent_Definition submission is a resubmission of an agent identifier that already holds a Routing_Table entry, THEN THE Agent_Registry SHALL retain the existing entry unchanged until a pass result for the resubmitted content is committed, and SHALL replace the entry only on that commit.

#### Stated Gap (carried from PRD US-1, not closed here)

10. THE Definition_Validator SHALL evaluate declared schema conformance and declared allowlist membership only, and THE Agent_Registry SHALL report registration status as `declared-and-present` rather than as a runtime behavioral guarantee.

### Requirement 2: Single-Agent Category Routing

**User Story:** As a Shopper-Agent Principal, I want a single free-text intent to route automatically to whichever registered agent's declared category matches, so that I use one interface across verticals instead of a different app per vertical.

Traces to: PRD US-2; ADR-1; Journey → System Mapping row "Discover".

#### Acceptance Criteria

1. WHEN Agent_Registry receives a Typed_Intent whose declared category field is present and contains 1 to 64 characters, THE Agent_Registry SHALL select the registered agent whose declared category matches the Typed_Intent declared category field exactly after trimming leading and trailing whitespace and case-folding both values.
2. WHEN Agent_Registry selects a registered agent for a Typed_Intent, THE Agent_Registry SHALL issue exactly one Discovery dispatch for that Typed_Intent and zero further dispatches for that same Typed_Intent.
3. IF zero registered agents match the Typed_Intent declared category field, THEN THE Agent_Registry SHALL return No_Match_Result to the Edge Orchestrator carrying an unmatched-category reason and SHALL issue zero Discovery dispatches.
4. IF two or more registered agents declare the same category, THEN THE Agent_Registry SHALL return No_Match_Result carrying an ambiguous-category reason and SHALL issue zero Discovery dispatches.
5. WHEN Agent_Registry completes routing for a Typed_Intent, THE Agent_Registry SHALL record exactly one Session_Log routing entry carrying the Typed_Intent identifier, the declared category value as received, and either the selected agent identifier or the No_Match_Result reason.
6. THE Agent_Registry SHALL read its category-to-agent mapping from externalized registration state, and THE Agent_Registry SHALL hold zero per-vertical agent identifiers in source.
7. THE Agent_Registry SHALL resolve routing deterministically without a model call, returning the identical outcome for identical declared category field and registration state, within 200 milliseconds of receiving the Typed_Intent.
8. IF the Typed_Intent declared category field is absent, empty, whitespace-only, or longer than 64 characters, THEN THE Agent_Registry SHALL return No_Match_Result to the Edge Orchestrator carrying an invalid-category reason and SHALL issue zero Discovery dispatches.
9. IF the externalized registration state cannot be read, THEN THE Agent_Registry SHALL return No_Match_Result to the Edge Orchestrator carrying a registration-state-unavailable reason, SHALL issue zero Discovery dispatches, and SHALL leave the received Typed_Intent unmodified.

### Requirement 3: Operator-Visible Marketplace Registry Canvas

**User Story:** As a Platform Operator, I want a Registry_Canvas node listing every registered agent's Agent_Definition, tool allowlist, and trust/verification status, so that I can audit what is live without reading code.

Traces to: PRD US-3; ADR-3; Component Specification "Marketplace Registry Canvas".

#### Acceptance Criteria

1. WHEN the Agent_Definition_Table changes, THE Registry_Canvas SHALL project the changed registration state to every subscribed Operator_Scope client within 2 seconds for registries containing up to 500 Agent_Definitions.
2. THE Registry_Canvas SHALL render, for every Agent_Definition present in the Agent_Definition_Table, the agent identifier, declared category, declared tool allowlist, and trust/verification status, and THE Registry_Canvas SHALL render an explicit "not declared" indicator for any of these four fields whose stored value is empty or absent.
3. THE Registry_Canvas SHALL contain zero rendered entries whose agent identifier is absent from the Agent_Definition_Table for the same read.
4. WHILE a client subscription carries a scope other than Operator_Scope, THE Registry_Canvas SHALL render zero registration entries to that subscription and SHALL present an indication that the registration state is not available to that scope.
5. THE Registry_Canvas SHALL persist state under the established `table_name:record_id` key pattern using the already-adopted Yjs CRDT store, and THE Registry_Canvas SHALL introduce zero additional storage systems.
6. WHEN two Operator_Scope clients observe the same Registry_Canvas revision, THE Registry_Canvas SHALL present identical registration state to both clients.
7. THE Registry_Canvas SHALL render its agent list using semantic HTML list and description elements, with one list item per registration and a non-empty accessible name containing the agent identifier for each registration row.
8. IF the Registry_Canvas cannot read the Agent_Definition_Table, THEN THE Registry_Canvas SHALL retain and continue rendering the last successfully projected registration state, SHALL present an indication that the displayed state may be stale, and SHALL discard zero previously projected entries.
9. IF the Agent_Definition_Table contains zero Agent_Definitions for the current read, THEN THE Registry_Canvas SHALL render zero list items and SHALL present an indication that no agents are registered.
10. THE Registry_Canvas SHALL render every field named in criterion 2 without horizontal scrolling at a viewport width of 320 CSS pixels.

### Requirement 4: Guardrail And Confirmation Parity Across Registered Agents

**User Story:** As a Shopper-Agent Principal, I want the same budget guardrail and human-confirmation gate that protects flight bookings to apply identically regardless of which registered agent produced the offer, so that switching verticals never weakens my protection.

Traces to: PRD US-4; ADR-1; PRD Success Metric "Guardrail/confirmation-gate parity across agents".

#### Acceptance Criteria

1. WHEN a Discovery_Harness returns a Typed_Offer, THE Agent_Registry SHALL route that Typed_Offer to Guardrail_Gate as its first destination and SHALL NOT make that Typed_Offer available to Issuance_Service, Settlement Verifier, or Notification Dispatcher until Guardrail_Gate has returned an evaluation outcome for that offer identifier.
2. THE Issuance_Service SHALL issue a card for a Typed_Offer only when the Session_Log contains both a Gate_Pass_Event and a Human_Confirm_Event carrying the same offer identifier as that Typed_Offer, with the Gate_Pass_Event recorded before the Human_Confirm_Event, and SHALL issue at most one card per offer identifier.
3. IF a Typed_Offer reaches Issuance_Service and the Session_Log lacks either a matching Gate_Pass_Event or a matching Human_Confirm_Event for that offer identifier, THEN THE Issuance_Service SHALL issue no card, return a rejection response indicating that the guardrail or confirmation gate was not satisfied, record a fail-closed reason identifying which event was absent together with the offer identifier and agent identifier, and leave all prior Session_Log entries unchanged.
4. THE Guardrail_Gate SHALL produce the same evaluation outcome for two Typed_Offer records that differ only in agent identifier, and THE Agent_Registry SHALL pass zero agent-specific gate parameters, such that the set of inputs supplied to Guardrail_Gate is identical for every registered agent identifier.
5. THE Agent_Registry SHALL preserve the reused Guardrail_Gate, Shared Canvas Node Store, Issuance_Service, Settlement Verifier, and Notification Dispatcher interfaces unchanged, adding, removing, or altering no operation, parameter, or return value of those interfaces.
6. THE Session_Log SHALL record a non-empty agent identifier that matches a registered Agent_Registry entry on every gate, confirmation, and issuance entry, so that gate, confirmation, and issuance outcomes are retrievable and comparable per agent identifier.
7. IF a Typed_Offer carries an agent identifier that is absent from the Agent_Registry or is empty, THEN THE Agent_Registry SHALL route the Typed_Offer to no downstream component, record a fail-closed reason indicating an unrecognized agent identifier, and return a rejection response.
8. IF a Human_Confirm_Event for an offer identifier is not recorded in the Session_Log within the confirmation window that applies to flight bookings, THEN THE Issuance_Service SHALL treat that offer identifier as unconfirmed, issue no card, and record a fail-closed reason indicating confirmation expiry.

### Requirement 5: Issuance Remains The Sole Payment Caller

**User Story:** As an Agent Builder, I want my registered agent's approved spend automatically scoped to a StraitsX-issued disposable card, so that I do not write my own card-issuance integration to participate in the marketplace.

Traces to: PRD US-5; PRD Topology "Registered-Agent zone (outside Knowgrph trust boundary)".

#### Acceptance Criteria

1. THE Issuance_Service SHALL be the only component holding a configured client for the StraitsX Card MCP Gateway, and every StraitsX Card MCP Gateway request observed at the gateway boundary SHALL carry the Issuance_Service component identifier.
2. IF a component whose identifier is not the Issuance_Service identifier attempts a StraitsX or Avalanche call, THEN THE Dev_Lane SHALL reject the call before any outbound request is issued, SHALL return an error indicating an unauthorized payment-path caller, and SHALL record the calling component identifier with the rejected target service name.
3. WHEN THE Agent_Registry forwards a Typed_Intent to a Discovery_Harness, THE Agent_Registry SHALL include only the Typed_Intent fields enumerated in the Discovery input contract, SHALL omit every field not enumerated in that contract, and SHALL include zero payment credentials, card identifiers, or card tokens.
4. IF a Typed_Offer returned by a Discovery_Harness carries any field outside the Discovery output contract's offer-data field set, or any field whose name or value matches a credential, card-number, card-token, or account-identifier form, THEN THE Agent_Registry SHALL reject the Typed_Offer, SHALL return an error indicating a credential-bearing offer, and SHALL leave the originating Typed_Intent unmodified.
5. THE Dev_Lane SHALL resolve every provider endpoint and credential from environment-provided configuration, and THE Dev_Lane source SHALL contain zero credentials, account identifiers, card identifiers, or developer-specific absolute paths.
6. WHEN THE Dev_Lane starts, IF any required provider endpoint or credential is absent from environment-provided configuration, THEN THE Dev_Lane SHALL refuse to start, SHALL return an error naming each absent configuration key, and SHALL apply no built-in default value for that key.

### Requirement 6: Bounded, Traceable Execution Evidence

**User Story:** As the Orchestrator, I want every implementation task bounded and independently verified, so that a completion claim is earned from surfaced evidence rather than asserted.

Traces to: `agentic-sdlc-guidelines.md` sections Agent Roles & Independence, Per-Task Budgets, Verification Strategy, Execution Contract; PRD Component Specifications "Evidence References: none yet".

#### Acceptance Criteria

1. THE Dev_Lane SHALL state, before dispatching a task, one named check per verifiable completion condition traced to that task, phrased exactly as it is invocable, such that a reader can invoke it without further interpretation.
2. WHEN an implementation task returns, THE Dev_Lane SHALL surface in that return the named check invoked, its exit status, its recorded counts, and the enumerated set of artifacts changed, including incidental changes.
3. THE Dev_Lane SHALL route every completion verdict to an Evaluator mechanism that is a different mechanism from the implementing mechanism and does not modify the artifacts it judges, and SHALL record, for each verdict, which mechanism produced it.
4. IF a verdict for a task is produced by that task's implementing mechanism, or is derived from state absent from the surfaced return, THEN THE Dev_Lane SHALL withhold the terminal success transition and record a blocker-severity self-graded-verdict finding naming the task.
5. IF a required named check is absent, a recorded result is absent, a recorded result was not produced by an invocation within the current task, or an evidence join to its source condition is missing, THEN THE Dev_Lane SHALL report the affected component as not runtime-ready and record which of those four causes applied.
6. THE Dev_Lane SHALL record, for every implementation task and regardless of whether any bound was approached, the consumed token count, iteration count, elapsed wall-clock duration, and peak working-context consumption, each against its stated bound.
7. IF a task is dispatched with any of its four bounds unstated, THEN THE Dev_Lane SHALL refuse the dispatch and record a blocker-severity unbounded-task finding.
8. WHEN a task's named check produces no change in recorded result across two consecutive iterations, THE Dev_Lane SHALL stop retrying, transition the task to failed, and record the last observed result and the reason for the terminal transition.
9. THE Dev_Lane SHALL keep each of Agent_Registry, Definition_Validator, and Registry_Canvas at 600 or fewer authored lines per file, excluding generated content, with each file declaring exactly one stated responsibility.
10. IF an authored file among Agent_Registry, Definition_Validator, and Registry_Canvas exceeds 600 authored lines or declares more than one responsibility, THEN THE Dev_Lane SHALL report that file as not runtime-ready and record its measured authored line count.

### Requirement 7: Dev-Only Deploy Boundary

**User Story:** As the Platform Operator, I want every deploy boundary to remain closed during this increment, so that no marketplace change reaches the Prod mirror or Cloudflare without exact-candidate human authorization.

Traces to: PRD Deploy Boundary Register (all three rows); `START-WORKFLOW.md` deploy gate; `agentic-sdlc-guidelines.md` Tool Permission & Blast Radius.

#### Acceptance Criteria

1. THE Dev_Lane SHALL satisfy every acceptance criterion in this document while leaving the Prod mirror revision and the Cloudflare route set byte-identical to their pre-run state, verified by comparing the recorded pre-run and post-run state of both surfaces.
2. IF no recorded authorization naming the exact candidate identity and issued by an authenticated human operator exists for the Mirror-to-Delivery boundary, THEN THE Deploy_Boundary SHALL report state `closed` and SHALL report zero permitted mutations for that boundary.
3. IF an implementation task requests a Prod mirror mutation or a Cloudflare route mutation, THEN THE Dev_Lane SHALL reject the request before any mutation is issued, SHALL leave both surfaces unchanged, SHALL record the requested operation and the boundary name it targeted, and SHALL return an error indicating that the boundary is closed.
4. WHERE integration to protected `main` is requested, WHEN the request is received, THE Dev_Lane SHALL route the change through the protected integration path and SHALL reject any direct write to `main` that bypasses that path.
5. WHEN a protected integration completes successfully, THE Dev_Lane SHALL record an integration receipt naming the protected path used, the candidate identity integrated, and the integration outcome, and THE Dev_Lane SHALL treat that receipt as Dev integration authority only, deriving zero deployment authority from it.
6. WHEN a Definition_Validator pass result is recorded for an exact Routing_Table entry, THE Registration Deploy_Boundary SHALL permit that entry, and only that entry, to become routable.
7. IF a Definition_Validator result for a Routing_Table entry is `fail` or is not recorded, THEN THE Registration Deploy_Boundary SHALL keep that entry non-routable and SHALL return an error indicating that validation evidence is absent or failing.
8. WHEN reversal of a registered Routing_Table entry is requested, THE Registration Deploy_Boundary SHALL remove that Routing_Table entry and SHALL report the entry as non-routable after removal.
9. THE Dev_Lane SHALL report state `closed` for all three rows of the Deploy Boundary Register at the start and at the end of this increment, and each row SHALL report zero recorded exact-candidate human authorizations.

### Requirement 8: Invocation Surface Consistency

**User Story:** As a Shopper-Agent Principal and Platform Operator, I want the marketplace reachable through the established MCP and token invocation surface, so that no second, divergent command registry appears.

Traces to: PRD Dependencies "Invocation Surface Contract / Agent Definition schema"; `AGENTS.md` owner routing; `START-WORKFLOW.md` dictionary authority.

#### Acceptance Criteria

1. THE Dev_Lane SHALL resolve every `/`, `#`, and `@` token exclusively through the three Agentic Canvas OS invocation dictionaries and their shared runtime projection, and SHALL treat a token as resolved only when exactly one dictionary entry matches that token.
2. IF a token has zero matching entries across the three dictionaries, THEN THE Dev_Lane SHALL reject the invocation without dispatch, record the unresolved token together with its invoking surface, and return an error indicating that the token is unresolved.
3. IF a token matches entries in more than one of the three dictionaries, THEN THE Dev_Lane SHALL reject the invocation without dispatch, record the token with the identity of every matching dictionary, and return an error indicating that the token is ambiguous.
4. THE Agent_Registry SHALL expose its registration and routing operations only through the MCP invocation surface, and THE Agent_Registry SHALL define zero parallel command registry.
5. IF a registration or routing request reaches THE Agent_Registry from any surface other than the MCP invocation surface, THEN THE Agent_Registry SHALL reject the request, leave the Routing_Table and Agent_Definition_Table unchanged, and record the rejected surface.
6. WHEN the Invocation Surface Contract schema revision changes, THE Definition_Validator SHALL revalidate every Agent_Definition in the Agent_Definition_Table against the new revision and record, per Agent_Definition, a pass or fail outcome bound to the new revision identifier.
7. WHILE a schema-revision revalidation pass is in progress, THE Dev_Lane SHALL block dispatch to every Agent_Definition that has no recorded pass outcome for the new revision.
8. IF an Agent_Definition fails revalidation against the new schema revision, THEN THE Agent_Registry SHALL remove its Routing_Table entry, leave the stored Agent_Definition record unchanged, and record the removal reason together with the failing revision identifier.

### Requirement 9: Local-First, Offline-Tolerant Operator And Shopper Surfaces

**User Story:** As a Shopper-Agent Principal on a mobile device, I want the marketplace surfaces to remain usable while connectivity is degraded, so that a dropped connection does not lose my session state.

Traces to: PRD Topology data-residency column ("Local (device) + Edge cache"); ADR-3 CRDT reuse; operating context local-first and mobile-first constraints.

#### Acceptance Criteria

1. WHILE a client is offline, where offline means no successful exchange with the edge cache has completed within the preceding 10 seconds, THE Registry_Canvas SHALL present the last locally held registration state within 1 second of the surface being opened or resumed, and SHALL display a persistent visible indicator that the presented state is not current together with the elapsed time since the last successful synchronization.
2. WHEN a client reconnects, THE Registry_Canvas SHALL converge local and edge state through CRDT merge within 5 seconds of reconnection, without operator intervention, and SHALL remove the not-current indicator once convergence completes.
3. IF CRDT merge does not complete within 5 seconds of reconnection, or the edge exchange fails, THEN THE Registry_Canvas SHALL retain the unmerged local state without discarding any locally recorded change, SHALL keep the not-current indicator visible, and SHALL retry merge at intervals of at most 30 seconds for up to 5 attempts before surfacing a message indicating that synchronization is unavailable.
4. WHILE a client is offline, THE Registry_Canvas SHALL retain locally recorded registration changes for at least 24 hours and for at least 100 pending changes, and SHALL submit them for CRDT merge on reconnection in the order recorded.
5. THE Dev_Lane SHALL render shopper and operator surfaces at a viewport width of 360 CSS pixels with no horizontal overflow, with every interactive control reachable by vertical scrolling only, and with no control having a touch target smaller than 44 by 44 CSS pixels.
6. WHILE a client is offline, THE Issuance_Service SHALL issue zero cards, and SHALL present a message indicating that issuance requires connectivity while preserving the submitted issuance inputs locally for resubmission after reconnection.
7. THE Dev_Lane SHALL run every component of this increment on browser and edge runtimes already provisioned, and THE Dev_Lane SHALL add zero new infrastructure services.

## Correctness Properties

Each property names its class per `agentic-sdlc-guidelines.md` Property-Based Obligations. Each is a candidate property-based test with shrinking enabled and a stated minimum iteration count.

### CP-1 — Routing Exclusivity (Invariant)

Covers: Requirement 2.1, 2.2, 2.3, 2.4, 2.8, 2.9; Requirement 1.1.
For all generated Typed_Intent and Agent_Definition_Table pairs, the number of Discovery dispatches emitted by Agent_Registry is exactly one when precisely one registered agent declares the intent's category, and exactly zero otherwise. Minimum iterations: 200.

### CP-2 — Registration Gate Invariant (Invariant)

Covers: Requirement 1.1, 1.4, 1.9.
For all generated dispatch sequences, every dispatched agent identifier is a member of the Agent_Definition_Table at dispatch time, and every Routing_Table entry has a corresponding Definition_Validator pass result. Minimum iterations: 200.

### CP-3 — Payment Ordering Invariant (Invariant)

Covers: Requirement 4.2, 4.3, 4.8.
For all generated Session_Log sequences over all agent identifiers, every Issuance_Service issuance entry is preceded in the same session by both a Gate_Pass_Event and a Human_Confirm_Event carrying the same offer identifier, with the Gate_Pass_Event ordered before the Human_Confirm_Event, and at most one issuance exists per offer identifier. Minimum iterations: 500.

### CP-4 — Agent-Identifier Parity (Metamorphic)

Covers: Requirement 4.4.
For all generated Typed_Offer records, substituting the agent identifier while holding offer content fixed leaves the Guardrail_Gate result unchanged. Minimum iterations: 300.

### CP-5 — Agent Definition Round Trip (Round Trip)

Covers: Requirement 1.3, 1.6.
For all valid Agent_Definition values, serializing then parsing then serializing yields an equivalent Agent_Definition, and the Definition_Validator verdict is identical across both representations. Minimum iterations: 300.

### CP-6 — Registry Projection Consistency (Invariant)

Covers: Requirement 3.2, 3.3, 3.9.
For all generated Agent_Definition_Table states, the set of agent identifiers rendered by Registry_Canvas equals the set present in the Agent_Definition_Table for the same read revision. Minimum iterations: 200.

### CP-7 — CRDT Merge Confluence (Confluence)

Covers: Requirement 3.6, 9.2.
For all generated pairs of concurrent Registry_Canvas update sequences, applying them in either order yields identical converged registration state. Minimum iterations: 300.

### CP-8 — Registration Idempotence (Idempotence)

Covers: Requirement 1.4, 1.9, 8.6.
For all valid Agent_Definition values, registering the same definition twice yields the same Routing_Table state as registering it once. Minimum iterations: 200.

### CP-9 — Malformed Definition Error Conditions (Error Condition)

Covers: Requirement 1.3, 1.7.
For all generated Agent_Definition values violating at least one schema constraint, Definition_Validator returns a reject result naming every violated field identifier, and no Routing_Table entry is created. Minimum iterations: 400.

### CP-10 — Credential Non-Propagation (Invariant)

Covers: Requirement 5.3, 5.4.
For all generated Typed_Intent and Typed_Offer values, the payloads crossing the Agent_Registry boundary contain zero credential-shaped fields, and every StraitsX or Avalanche call in the recorded tool-call log names Issuance_Service as caller. Minimum iterations: 300.

### CP-11 — No_Match Totality (Invariant)

Covers: Requirement 2.3, 2.5, 2.8, 2.9.
For all generated Typed_Intent values, Agent_Registry returns either exactly one selected agent identifier or a No_Match_Result carrying a reason, and records exactly one Session_Log routing entry. Minimum iterations: 300.

### CP-12 — Unrecognized Agent Identifier Rejection (Error Condition)

Covers: Requirement 4.7.
For all generated Typed_Offer values whose agent identifier is empty or absent from the Agent_Registry, zero downstream components receive the offer and a fail-closed reason naming an unrecognized agent identifier is recorded. Minimum iterations: 300.

### CP-13 — Offline Change Order Preservation (Invariant)

Covers: Requirement 9.4.
For all generated sequences of locally recorded registration changes made while offline, the order in which those changes are submitted for CRDT merge on reconnection equals the order in which they were recorded, and no recorded change is dropped. Minimum iterations: 200.

## Integration And Example Checks

Per the source guidance on when property-based testing does not apply, the following criteria are verified by bounded example or integration checks rather than by property tests.

| Criterion | Check kind | Rationale |
|---|---|---|
| 1.8 Schema retrieval unavailable | Integration, 1 example | External retrieval failure and timeout, not input-varying logic |
| 3.4 Operator_Scope withholding | Integration, 2 examples | Access-scope behavior does not vary meaningfully with generated input |
| 3.7 Semantic HTML and accessible names | Deterministic static and DOM assertion | Structural conformance, not input-varying logic |
| 3.8 Stale-read retention and indicator | Integration, 1 example | Read-failure path against an external store |
| 3.10 Registry_Canvas 320-pixel rendering | Browser assertion, 1 example | Deterministic layout check; the narrower 320px constraint applies to Registry_Canvas specifically, and is stricter than the 360px surface-wide constraint in 9.5 |
| 4.8 Confirmation-window expiry | Integration, 1 example | Time-based gate against the existing flight-booking window |
| 5.5 No persisted credentials or machine paths | Repository scan check | Static repository property, single execution |
| 5.6 Startup refusal on absent configuration | Integration, 1 example per absent key class | Startup configuration gate, single execution |
| 6.1, 6.2 Named check statement and surfaced return | Process assertion, 1 example per task | Execution-contract conformance, not input-varying |
| 6.3, 6.4 Evaluator independence | Process assertion, 1 example per verdict | Mechanism-identity check |
| 6.5 Not-runtime-ready cause recording | Integration, 1 example per cause | Four enumerated causes, bounded example set |
| 6.6, 6.7 Bound recording and unbounded dispatch refusal | Process assertion, 1 example each | Dispatch-time governance check |
| 6.8 Two-iteration no-progress stop | Integration, 1 example | Retry-policy behavior, bounded |
| 6.9, 6.10 File size and single responsibility | Repository scan check | Static repository property |
| 7.1–7.5, 7.9 Deploy boundary states | Integration, 1 example per boundary | External gate state, not input-varying |
| 7.6–7.8 Registration boundary routability and reversal | Integration, 1 example each | Recorded gate transition, not input-varying |
| 8.1, 8.2, 8.3 Dictionary token resolution | Integration, 3 examples | Fixed external registry lookup |
| 8.5 Non-MCP surface rejection | Integration, 2 examples | Surface-identity check |
| 8.7 Dispatch block during revalidation | Integration, 1 example | State-transition window, single execution |
| 9.1, 9.3 Offline indicator and retry policy | Integration, 1 example each | Connectivity-state behavior against the edge cache |
| 9.5 360-pixel viewport rendering and touch targets | Browser assertion, 1 example per surface | Deterministic layout check across shopper and operator surfaces |
| 9.6 Offline issuance suppression | Integration, 1 example | Connectivity-gated external call |
| 9.7 Zero new infrastructure services | Dependency and configuration inventory check | Static property |

## Traceability Matrix

| Requirement | Source | Component(s) | Properties |
|---|---|---|---|
| 1 | PRD US-1, ADR-2, Registration Deploy Boundary | Definition_Validator, Agent_Registry | CP-2, CP-5, CP-8, CP-9 |
| 2 | PRD US-2, ADR-1 | Agent_Registry | CP-1, CP-11 |
| 3 | PRD US-3, ADR-3 | Registry_Canvas | CP-6, CP-7 |
| 4 | PRD US-4 | Guardrail_Gate (reused), Agent_Registry, Issuance_Service (reused) | CP-3, CP-4, CP-12 |
| 5 | PRD US-5, PRD Topology | Issuance_Service (reused), Agent_Registry | CP-10 |
| 6 | agentic-sdlc-guidelines execution seam | all three new components | evidence obligations only |
| 7 | PRD Deploy Boundary Register, START-WORKFLOW deploy gate | Dev_Lane | integration checks |
| 8 | PRD Dependencies, dictionary authority | Agent_Registry, Definition_Validator | CP-8 |
| 9 | PRD Topology data residency, ADR-3 | Registry_Canvas, client surfaces | CP-7, CP-13 |

Bridge coverage: 9 of 9 requirements trace to at least one source artifact; 5 of 5 PRD user stories (US-1 through US-5) are covered by Requirements 1 through 5 respectively.

## Non-Goals

Carried verbatim in intent from the PRD's Out of Scope, MoSCoW "Won't (this increment)", and Platform Roadmap Phase 2 through Phase 4 rows. Each is explicitly outside this requirements set.

1. Runtime capability sandboxing of a registered agent's behavior beyond declared allowlist membership (PRD MoSCoW "Could"; ADR-2; roadmap Phase 2).
2. Public third-party self-serve registration surface (PRD Out of Scope; PRD MoSCoW "Won't").
3. On-chain trust or reputation attestation for registered agents (ADR-2; roadmap Phase 2 Agent Trust & Verification Registry).
4. Marketplace fee, billing, or monetization model (PRD Out of Scope; PRD MoSCoW "Won't").
5. Multi-tenant fund segregation beyond the existing shared and personal CRDT key scoping (PRD Out of Scope).
6. A third net-new vertical Discovery_Harness; this increment proves the pattern with the two already-specified harnesses (PRD Out of Scope; Min-Viable Scope).
7. A generic DOM or web-agent Discovery_Harness (roadmap Phase 2 Agentic Checkout Copilot).
8. Exposing Issuance_Service as a directly callable external MCP tool for third-party agents (roadmap Phase 3).
9. On-chain escrow or programmable spend-policy smart contracts (roadmap Phase 3 Spend-Policy Guardrails Agent).
10. Multi-agent split-pay or group wallet settlement (roadmap Phase 4).
11. Post-hoc spend audit and explainability agent (roadmap Phase 4).
12. Any modification to the reused Guardrail_Gate, Shared Canvas Node Store, Issuance_Service, Settlement Verifier, or Notification Dispatcher interfaces (PRD "Reused components (unchanged)").
13. Prod mirror publication and Cloudflare deployment; both remain gated targets outside this increment's acceptance criteria (PRD Deploy Boundary Register; START-WORKFLOW deploy gate).

## Open Questions Carried From The Source Specification

These remain unresolved in the PRD and are recorded here without being silently decided. Each requires an operator decision before the affected requirement can be finalized.

1. Does category routing need a classifier rather than a fixed enum? Requirement 2.7 currently mandates deterministic, non-model routing within 200 milliseconds, matching the PRD's $0 token-cost claim. Introducing a classifier would change that requirement and the token budget.
2. Where does trust and verification actually enforce — inside the registered agent, pre-dispatch inside Agent_Registry, or through on-chain attestation? Requirement 1.10 records the PRD's declarative-only position as interim.
3. Does each registered agent need its own funding source, or does every agent draw from one operator-controlled wallet? Affects any future multi-tenant fund segregation, currently a non-goal.
4. Marketplace fee model — free infrastructure versus a take rate on settled transactions. Currently a non-goal, deliberately undecided.
