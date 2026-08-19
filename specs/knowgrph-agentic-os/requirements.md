# Requirements Document

## Introduction

`knowgrph` (canonical repo `huijoohwee/knowgrph`, deployed at `airvio.co/knowgrph`) is a solo-dev, AI-native knowledge graph and media intelligence platform that already runs six independent AI/automation harnesses — FloatingPanel Chat → KGC (multi-provider LLM harness), the AI Agents Memory Layer (`mcp/memory-layer-runtime.js`), the Visual Annotation Engine (Transformers.js + Florence-2-base, in-browser), the HTML Video Renderer, the Video Intelligence pipeline (VideoDB Director MCP), the SuperAgent Harness (`knowgrph_parser/superagent_harness.py`), the AI Showrunner (`mcp/showrunner-runtime.js`), and the `knowgrph.video_remix.run` Director (the `knowgrph-acos-mcp-connector` spec) — each exposed through its own MCP tool surface, its own run-state shape, and its own circuit-breaker.

This specification defines the **knowgrph Agentic OS**: a thin, OS-level unification layer over these *already-existing* harnesses and contracts — not a replacement for any of them. "Agentic OS" here means the smallest set of cross-harness primitives (process visibility, capability discovery, cost/token accounting, approval-gate consistency, circuit-breaker observability) that lets an operator or an external agent reason about *all* running knowgrph agent work through one surface, instead of learning each harness's bespoke shape individually. Every requirement below is evaluated through the four compounding lenses required by the PRD & TAD Guidelines (`guidelines/prd-tad-guidelines.md`): **min-viable-max-value**, **TCO-zero**, **token economics**, and **harness-first** — and defaults to reusing existing contracts (`contracts/*.schema.js`), existing registries (`knowgrph.vdeoxpln.list`, `mcp/local-tool-contract.js`), existing storage (Cloudflare D1/R2/KV/Durable Objects, local JSON state files), and existing MCP surfaces (`mcp/server.js`, the Cloudflare `McpAgent`) over introducing any new dependency, datastore, or process runtime. Per the native-in-repo consolidation decision (ADR-3 in the combined PRD/TAD), the Vercel product tier, the AWS Agent-API fallback, and the AWS AgentCore wrapper lane are removed from the runtime topology; Supabase is permanently excluded. The Cloudflare `McpAgent` is the sole remote MCP surface.

This document is the requirements artifact for the runtime-ready Agentic OS increment. The combined PRD/TAD (`docs/documents/knowgrph-agentic-os-prd-tad.md` in the `knowgrph` repo) weaves these requirements together with the TAD Core Template, following the existing combined-doc conventions established by `knowgrph-tech-stack-document.md` and `knowgrph-ai-agents-memory-layer-prd-tad.md`.

**This revision** folds three further increments into the same spec, evaluated through the same four lenses and preserving Requirements 1–7 verbatim: (a) the three follow-on tracks already scoped in `docs/documents/knowgrph-agentic-os-follow-on-prd-tad.md` — a durable HITL Approval_Token store on the Cloudflare control-plane Worker (Track A), one deployed golden-path live Exa research + storyboard proof (Track B), and the Agentic Canvas OS dashboard document rendered on Canvas (Track C); (b) MCP Gateway maturity — consistent error/timeout semantics and a documented surface-selection contract across the existing four MCP surfaces (local stdio, Pages HTTP MCP, browser WebMCP, Cloudflare `McpAgent` control plane), without adding a fifth proxy tier (ADR-4/ADR-006 preserved); and (c) AI Agent tool ergonomics — per-tool usage guidance surfaced through `knowgrph.os.status`, a consistent structured error shape across harness tool calls, and one documented agent onboarding sequence. None of Requirements 8–15 below weaken the R11 Spend_Isolation_Boundary, the Approval_Gate_Invariant (Property 1, `knowgrph-acos-mcp-connector`), ADR-3 (no Vercel/Supabase/AWS tier), or ADR-4/ADR-006 (no fifth MCP proxy tier).

### Grounding: what already exists vs. what this spec adds

| Capability area | Already exists in `knowgrph` (reuse, do not rebuild) | What this spec adds (net-new OS glue) |
|---|---|---|
| Per-harness run state | Showrunner `Pipeline_Run` (`queued→running→awaiting_review→complete`, `showrunner/runs/<run_id>/state.json`); Video_Remix `Run_Manifest` (`state`, `stages[]`, `approvalGates[]`, `budgetMeters`); SuperAgent `state.json` + `trace.jsonl` + `harness-proof.json` | A read-only **Process_Registry** that aggregates these three existing state shapes into one normalized listing — no new persistent store |
| Cost/token accounting | `contracts/cost-log.schema.js` (canonical raw `Cost_Log` — `model, prompt_tokens, completion_tokens, cache_hits, estimated_cost_usd, incomplete`); Director per-stage aggregation (`mcp/video-remix/cost-log.js`); `Credit_Ledger` (`StrytreeCreditLedgerActor` / render ledger events) | A **Cost_Ledger_Aggregator** that normalizes whatever each harness already emits into the canonical `Cost_Log` shape and reports which harnesses do **not** yet emit one (coverage gap), rather than a second cost-tracking system |
| Capability / tool discovery | `knowgrph.vdeoxpln.list` (the vdeoxpln skill registry, `canvas/src/features/agent-ready/knowgrphVdeoxplnContract.mjs`); `mcp/local-tool-contract.js` (local stdio MCP tool catalog); the Cloudflare `McpAgent` `tools/list` | A **Capability_Registry** that unions these three existing catalogs into one queryable surface — no parallel skill registry (the former AWS AgentCore `TOOL_CATALOG` is removed from the topology per ADR-3) |
| Approval / spend gating | `contracts/approval.schema.js` (`ApprovalGate`, `Approval_Token`, six canonical gate ids, 15-minute TTL, R11 spend-isolation boundary, Property 1 invariant) — used by `knowgrph.video_remix.run`; Showrunner's ad hoc `knowgrph.showrunner.approve_stage` (no `Approval_Token` today) | A **Gate_Catalog** extension so Showrunner stage approval is expressible under the *same* `ApprovalGate`/`Approval_Token` contract, without touching the R11 boundary or Property 1 |
| Bounded loops / circuit-breakers | VideoDB async poll (36 × 10 s); SuperAgent retry loop (`max_steps`, `max_retries_per_task`); Showrunner per-role `max_retries` / `token_budget`; Director bounded backoff (1 s → 30 s cap) | A **Circuit_Breaker_Registry**: a read-only catalog of each harness's already-configured bound and current iteration count — not a shared dispatcher (the harnesses run on three different runtimes: Cloudflare Workers, Node stdio, Python subprocess, and cannot share one in-process scheduler without a rewrite that fails min-viable-max-value) |
| Operator/agent visibility | None today — each harness must be queried individually via its own MCP tool | An **Os_Status_Tool**: one MCP tool surface exposing all five Agentic OS read views |
| HITL token durability | `mcp/video-remix/approval-token-issuer.js` — Approval_Token issuance, single-use `consumeSeam()`, and an injectable `store` interface, backed today by an in-memory store local to the Worker isolate | A **Durable_Approval_Token_Store** Worker KV adapter implementing the same `store` interface — Track A |
| Live stage clients | `mcp/video-remix/live-clients.js` — env-gated `resolveStageClients()`; Exa research client wireable now; storyboard client wireable now (`AI_GATEWAY_CHAT_URL`); render/commerce clients scaffolded (`requiresAsyncHarness`) | One deployed, approval-gated golden-path proof exercising the already-wireable research + storyboard clients — Track B |
| Dashboard/lane plan | `knowgrph.agentic_canvas_os.plan` (`mcp/local-tool-contract.js`) — dry-run Dashboard_Document + typed run manifest across lanes; existing `applyChatKgcWorkspaceDocumentToCanvas` frontmatter-flow apply path and Storyboard Widget | Rendering the dry-run plan's Dashboard_Document on Canvas through the existing apply path — Track C, no new renderer |
| MCP Gateway federation | Four existing surfaces (local stdio `mcp/server.js`, Pages HTTP MCP, browser WebMCP, Cloudflare `McpAgent` control plane) unified by shared contracts (`contracts/*.schema.js`, `mcp/local-tool-contract.js`, `canvas/src/features/agent-ready/`, `cloudflare/workers/knowgrph-mcp/tool-registry.mjs`); `.well-known/mcp/server-card.json` | A documented surface-selection contract (Capability_Class + recommended surface per capability) and a shared error/timeout envelope across the four surfaces — no fifth proxy tier |
| Tool ergonomics | `mcp/local-tool-contract.js` descriptor pattern (`name`, `inputSchema`, `outputSchema`); per-harness ad hoc error shapes (`{ok:false, error:{code,message}}` in Showrunner vs `{ok:false, errorCode, message}` in `knowgrph.os.status` vs unstructured `Error: ...` text in the `mcp/server.js` top-level catch) | Optional `usageExample` descriptor field, one normalized `{ok:false, errorCode, message}` error shape across all harness tool calls, and one documented onboarding sequence |

## Glossary

- **Agentic_OS**: The unification layer defined by this specification — the Process_Registry, Capability_Registry, Cost_Ledger_Aggregator, Circuit_Breaker_Registry, Gate_Catalog extension, and the Os_Status_Tool that exposes them. It is additive to, and holds no duplicate state store from, the existing Harnesses it aggregates.
- **Harness**: Any existing typed knowgrph pipeline component with input/output schemas and a fallback path — FloatingPanel Chat → KGC, the AI Agents Memory Layer, the Visual Annotation Engine, the HTML Video Renderer, the Video Intelligence pipeline, the SuperAgent Harness, the AI Showrunner, and the Video_Remix Director.
- **Process_Registry**: The Agentic_OS read-only aggregation surface that lists in-flight and recently completed units of harness work (Process_Entry records) by reading each Harness's existing state source, without persisting a second copy of that state.
- **Process_Entry**: A normalized record `{ processId, harness, status, startedAt, sourceRef }` derived at read time from a Pipeline_Run, a Run_Manifest, or a Superagent_Run.
- **Pipeline_Run**: The existing Showrunner run-lifecycle record (`mcp/showrunner-runtime.js`), one of `queued`, `running`, `awaiting_review`, `complete`.
- **Run_Manifest**: The existing Video_Remix Director run output (`knowgrph-acos-mcp-connector` spec Data Models), containing `state`, `stages[]`, `approvalGates[]`, `budgetMeters`.
- **Superagent_Run**: The existing SuperAgent Harness run record persisted as `state.json` / `trace.jsonl` / `harness-proof.json` under the run's output directory (`knowgrph_parser/superagent_harness.py`).
- **Capability_Registry**: The Agentic_OS read-only surface that unions the Vdeoxpln_Registry, the Local_Mcp_Tool_Catalog, and the Cloudflare_Mcp_Agent tool catalog into one queryable list of callable capabilities, each with its id, owning Harness, and input/output schema reference.
- **Vdeoxpln_Registry**: The existing skill/capability registry exposed by `knowgrph.vdeoxpln.list` (`canvas/src/features/agent-ready/knowgrphVdeoxplnContract.mjs`).
- **Local_Mcp_Tool_Catalog**: The existing local stdio MCP tool descriptor set built by `mcp/local-tool-contract.js` and served by `mcp/server.js`.
- **Cloudflare_Mcp_Agent**: The existing Cloudflare `McpAgent` worker (`cloudflare/workers/knowgrph-mcp`) exposing `tools/list` over MCP Streamable HTTP — the sole remote MCP surface after the ADR-3 native-in-repo consolidation (the former AWS AgentCore forwarder is removed from the runtime topology).
- **Cost_Log**: The existing canonical per-model-call record defined by `contracts/cost-log.schema.js` — `{ model, prompt_tokens, completion_tokens, cache_hits, estimated_cost_usd, incomplete }`.
- **Cost_Ledger_Aggregator**: The Agentic_OS read-only surface that collects the Cost_Log entries and Credit_Ledger events already emitted (or, where a Harness has an identified gap, reports the absence) across all Harnesses and reports totals normalized to the canonical Cost_Log schema.
- **Cost_Emission_Gap**: A Harness identified by the Cost_Ledger_Aggregator as not yet emitting a schema-valid Cost_Log for one or more of its model-bearing calls.
- **Approval_Gate**: The existing named spend boundary defined by `contracts/approval.schema.js`, identified by one of the six canonical gate ids.
- **Approval_Token**: The existing verifiable, time-bounded (15-minute), single-use credential defined by `contracts/approval.schema.js` that authorizes one Approval_Gate.
- **Gate_Catalog**: The Agentic_OS-maintained enumeration of every Approval_Gate in effect across all Harnesses, extending the existing six-gate catalog to additionally list the Showrunner stage-approval boundary under the same Approval_Token contract.
- **Circuit_Breaker_Registry**: The Agentic_OS read-only catalog of each Harness's already-configured max-iteration bound and circuit-breaker exit condition, together with that Harness's current iteration count for any in-flight Process_Entry.
- **Os_Status_Tool**: The MCP tool surface, reachable through the existing Local_Mcp_Tool_Catalog (and, WHERE deployed, the Cloudflare_Mcp_Agent), through which an Operator or External_Agent reads the Process_Registry, Capability_Registry, Cost_Ledger_Aggregator, and Circuit_Breaker_Registry.
- **Operator**: The solo founder/developer operating the knowgrph deployment.
- **External_Agent**: Any MCP client — a human-operated tool, an autonomous agent, or an evaluation harness — that calls the knowgrph MCP surface.
- **Spend_Isolation_Boundary**: The existing R11 invariant (established in `knowgrph-acos-mcp-connector`) that only the Cloudflare control plane holds model keys and invokes paid models. After the ADR-3 native-in-repo consolidation there are no AWS or Vercel tiers; the invariant is preserved by a strictly smaller surface. The Agentic_OS SHALL NOT weaken this boundary.
- **Approval_Gate_Invariant**: The existing Property 1 invariant that a paid/spend-bearing action executes only against a verified, unexpired, unconsumed Approval_Token, and an Auth_Token never substitutes for one. The Agentic_OS SHALL NOT weaken this invariant.
- **Durable_Approval_Token_Store**: A Cloudflare Worker KV-backed adapter implementing the same `store` interface (`save`, `get`, `has`, `delete`, `list`, `clear`, `size`) already defined by `mcp/video-remix/approval-token-issuer.js`'s in-memory store seam, so an Approval_Token minted before a Worker restart still verifies and single-use-consumes after that restart, within its 15-minute TTL. It is a drop-in adapter, not a new issuance or verification mechanism, and it does not alter `APPROVAL_GATE_IDS`, `APPROVAL_TOKEN_TTL_MS`, or the Approval_Gate_Invariant.
- **Live_Golden_Path_Run**: One operator-approved Video_Remix Director run, on the deployed Cloudflare control-plane Worker, in which the research and storyboard stages execute against their already-wireable live clients (`mcp/video-remix/live-clients.js` `exaClient` and `storyboardClient`) because `KNOWGRPH_LIVE_CLIENTS` and the corresponding provider secrets are set, an Approval_Token is presented for each gated stage, and the resulting Run_Manifest persists with a schema-valid Cost_Log. Unset env or a withheld Approval_Token SHALL yield zero paid-provider calls per the existing ADR-FO-2 mock-default behavior.
- **Dashboard_Document**: The Source Files Markdown document produced by rendering the dry-run manifest output of `knowgrph.agentic_canvas_os.plan` (`mcp/local-tool-contract.js`) as a `kgc-computing-flow/v1` frontmatter-flow document, opened in Canvas through the existing `applyChatKgcWorkspaceDocumentToCanvas` apply path and the Storyboard Widget. It is read-only projection of the dry-run manifest; it introduces no new renderer, parser, or write-capable mutation path.
- **Mcp_Gateway_Surface**: One of the four existing MCP transports that together constitute the MCP Gateway federation contract: the Local_Mcp_Tool_Catalog (local stdio, `mcp/server.js`), the Pages HTTP MCP surface (`/knowgrph/mcp`, read-only), the browser WebMCP surface (in-page `navigator.modelContext`), and the Cloudflare_Mcp_Agent control-plane surface (`/knowgrph/control-plane/mcp`, orchestration/spend). Per ADR-4/ADR-006, these four surfaces are the complete gateway topology; no fifth surface or monolithic proxy tier SHALL be introduced.
- **Capability_Class**: A classification assigned to a Capability_Registry entry — either `read_only_discovery` (safe on any Mcp_Gateway_Surface, zero spend) or `approval_gated_spend` (routes exclusively to the Cloudflare_Mcp_Agent control-plane surface, subject to the Approval_Gate_Invariant) — that determines which Mcp_Gateway_Surface an Operator or External_Agent should select for that capability.
- **Gateway_Routing_Contract**: The documented mapping, surfaced through the Capability_Registry, from each Capability_Class to its recommended Mcp_Gateway_Surface(s), together with a shared error/timeout envelope (`{ok:false, errorCode, message}`, a stated per-surface timeout bound) applied consistently across all four Mcp_Gateway_Surfaces. It is a documentation and response-shape contract over the existing four surfaces, not a new dispatcher process.
- **Tool_Usage_Example**: An optional, static, per-capability usage guidance string or example-arguments object attached to a Capability_Entry (sourced from an existing tool descriptor's own documentation, e.g. `mcp/local-tool-contract.js` or `mcp/README.md`, never fabricated), surfaced through `knowgrph.os.status` (`view:"capabilities"`) so an External_Agent can see how to call a capability without a separate documentation fetch.
- **Harness_Tool_Error**: The normalized structured error shape `{ ok: false, errorCode: string, message: string }` that every harness-dispatched local MCP tool call (reachable through `mcp/server.js`'s `CallToolRequestSchema` handler) SHALL return on failure, replacing any unstructured text-only error content and unifying the per-harness ad hoc shapes already in use (e.g. Showrunner's `{ok:false, error:{code,message}}`, `knowgrph.os.status`'s `{ok:false, errorCode,message}`) under one `errorCode`-bearing envelope.
- **Agent_Onboarding_Sequence**: The documented, four-step sequence an External_Agent SHOULD follow to move from discovery to spend: (1) discover via DNS-AID/`.well-known`/`knowgrph.os.status` (`view:"capabilities"`); (2) call a `read_only_discovery` Capability_Class capability on its recommended Mcp_Gateway_Surface; (3) request an Approval_Token for any `approval_gated_spend` capability; (4) present the token and invoke the capability on the Cloudflare_Mcp_Agent control-plane surface. This sequence documents the existing four-surface topology; it introduces no new surface or token-issuance mechanism.

## Requirements

### Requirement 1: Operator Visibility Across In-Flight Harness Work (Process_Registry)

**User Story:** As an Operator, I want to see all in-flight and recently completed harness runs (Showrunner Pipeline_Run, Video_Remix Run_Manifest, SuperAgent Superagent_Run) from one place, so that I do not have to individually query each Harness's own tool to know what is running.

#### Acceptance Criteria

1. WHEN an Operator or External_Agent calls the Os_Status_Tool's process-list capability, THE Process_Registry SHALL return a Process_Entry for every Pipeline_Run, Run_Manifest, and Superagent_Run whose underlying state source exists and is readable at call time.
2. THE Process_Registry SHALL normalize each Process_Entry to the fields `{ processId, harness, status, startedAt, sourceRef }` regardless of which Harness produced the underlying state.
3. IF a Harness's underlying state source (Pipeline_Run manifest, Run_Manifest, or Superagent_Run `state.json`) is missing or unreadable, THEN THE Process_Registry SHALL omit that run from the response, SHALL record that Harness's state source as unavailable in a response metadata field, and SHALL NOT fail the overall call.
4. THE Process_Registry SHALL NOT persist a second copy of any Harness's run state; each call SHALL read the existing state sources directly.
5. WHEN the Process_Registry response would exceed 200 Process_Entry records, THE Process_Registry SHALL return the 200 most recently started entries and SHALL indicate that the result is truncated.

> **VCC translation** (Requirement 1.1): `Verify the Os_Status_Tool process-list response contains one Process_Entry per readable Pipeline_Run/Run_Manifest/Superagent_Run state source found on disk at call time, with no existing Harness state file modified.`

### Requirement 2: Unified Capability/Tool Discovery (Capability_Registry)

**User Story:** As an External_Agent, I want one query that lists every capability/tool the knowgrph deployment exposes, so that I can discover them without separately calling `knowgrph.vdeoxpln.list`, the local MCP tool catalog, and the Cloudflare McpAgent `tools/list`.

#### Acceptance Criteria

1. WHEN an Operator or External_Agent calls the Os_Status_Tool's capability-list capability, THE Capability_Registry SHALL return the union of the Vdeoxpln_Registry, the Local_Mcp_Tool_Catalog, and, WHERE the corresponding runtime is deployed and reachable, the Cloudflare_Mcp_Agent tool catalog.
2. THE Capability_Registry SHALL report, for each capability entry, at minimum its tool id, its owning Harness, and a reference to its input/output schema as already declared by the source catalog.
3. IF two source catalogs declare a capability with the same tool id, THEN THE Capability_Registry SHALL report both source catalogs against that single tool id rather than duplicating the entry.
4. IF the Cloudflare_Mcp_Agent catalog is unreachable at call time, THEN THE Capability_Registry SHALL return the remaining reachable catalogs' entries and SHALL indicate which catalog was unreachable.
5. THE Capability_Registry SHALL introduce no new capability-catalog storage; its output SHALL be derived at call time from the three existing catalogs.

> **VCC translation** (Requirement 2.1): `Verify the Capability_Registry response tool-id set equals the union of the three existing catalogs' tool-id sets recorded in the test fixture, with none of the three source catalog files modified.`

### Requirement 3: Unified Cost/Token Ledger Visibility With Coverage-Gap Detection (Cost_Ledger_Aggregator)

**User Story:** As an Operator, I want one view of Cost_Log totals across every Harness, and a flag on any Harness that is not emitting one, so that I can see real spend and close accounting gaps without redesigning any harness's internals.

#### Acceptance Criteria

1. WHEN an Operator calls the Os_Status_Tool's cost-summary capability, THE Cost_Ledger_Aggregator SHALL return, for each Harness, the sum of `estimated_cost_usd` across every Cost_Log entry or Credit_Ledger event available from that Harness's existing emission point (`contracts/cost-log.schema.js` consumers, the Director per-stage aggregation, render ledger events, or `Credit_Ledger` JSONL sources).
2. THE Cost_Ledger_Aggregator SHALL validate every aggregated Cost_Log-shaped entry against `validateCostLog()` from `contracts/cost-log.schema.js` before including it in a total, and SHALL validate Credit_Ledger source events against `validateCreditLedgerEvent()` before normalizing them to Cost_Log.
3. IF an entry fails `validateCostLog()`, THEN THE Cost_Ledger_Aggregator SHALL exclude that entry from the total, SHALL record it as a validation failure, and SHALL NOT halt aggregation of the remaining entries.
4. IF a Harness has at least one model-bearing call in its Process_Registry history with no corresponding schema-valid Cost_Log or Credit_Ledger entry, THEN THE Cost_Ledger_Aggregator SHALL report that Harness as a Cost_Emission_Gap, regardless of whether that Harness has never emitted a schema-valid cost entry for any of its model-bearing calls or has emitted one for some calls and not others; THE Cost_Ledger_Aggregator SHALL NOT distinguish "never emitted" from "stopped emitting" when classifying a Cost_Emission_Gap.
5. THE Cost_Ledger_Aggregator SHALL report totals using the existing `estimated_cost_usd` field semantics and SHALL NOT introduce a second currency or unit representation.

> **VCC translation** (Requirement 3.2): `Verify every Cost_Log-shaped entry included in a reported total passes contracts/cost-log.schema.js validateCostLog(), every Credit_Ledger source event passes contracts/credit-ledger.schema.js validateCreditLedgerEvent(), and every entry that fails is present in the validation-failures list instead, with schema files unmodified.`

### Requirement 4: Approval-Gate Consistency Across Harnesses (Gate_Catalog Extension)

**User Story:** As an Operator, I want Showrunner's existing `approve_stage` action to be describable under the same Approval_Gate / Approval_Token contract already used by the Video_Remix Director, so that "what needs my approval" has one shape across harnesses without changing how either harness enforces its own gate today.

#### Acceptance Criteria

1. THE Gate_Catalog SHALL list every Approval_Gate already defined by `contracts/approval.schema.js` (the six canonical gate ids) plus one additional entry representing the Showrunner stage-approval boundary.
2. WHEN a Showrunner Pipeline_Run reaches `awaiting_review`, OR WHEN any Harness run (regardless of its own pipeline/run state) carries an Approval_Gate whose `approvalState` is `pending`, THE Gate_Catalog SHALL report a pending Approval_Gate entry for that run using the existing `ApprovalGate` shape (`gateId`, `approvalState`, `estimatedCostUsd`, `token`).
3. THE Gate_Catalog extension SHALL NOT modify the six existing canonical gate ids, the 15-minute Approval_Token TTL, or the Spend_Isolation_Boundary.
4. IF the Gate_Catalog cannot determine an `estimatedCostUsd` for the Showrunner stage-approval boundary from existing data, THEN THE Gate_Catalog SHALL report that field using the existing Cost_Log unknown indicator rather than a fabricated number.
5. THE Agentic_OS SHALL NOT introduce any new mechanism capable of approving a spend-bearing action; the Gate_Catalog is read/describe-only and defers all approval enforcement to the existing Hitl_Gate_Service and the Showrunner `approve_stage` handler.

> **VCC translation** (Requirement 4.3): `Verify contracts/approval.schema.js APPROVAL_GATE_ID_VALUES, APPROVAL_TOKEN_TTL_MS, and the R11 secret-scan smoke tests are byte-identical before and after this increment.`

### Requirement 5: Circuit-Breaker Bound Observability (Circuit_Breaker_Registry)

**User Story:** As an Operator, I want to see each Harness's already-configured iteration bound and its current iteration count for any in-flight run, so that I can tell whether a run is approaching its circuit-breaker without inspecting each harness's logs individually.

#### Acceptance Criteria

1. THE Circuit_Breaker_Registry SHALL report, for each Harness, its already-configured max-iteration bound and circuit-breaker exit condition exactly as declared in that Harness's existing configuration (VideoDB 36×10s poll, SuperAgent `max_steps`/`max_retries_per_task`, Showrunner per-role `max_retries`/`token_budget`, Director bounded backoff 1s→30s cap).
2. WHEN a Process_Entry is in-flight, THE Circuit_Breaker_Registry SHALL always report a current-iteration-count field for that entry alongside its Harness's configured bound, regardless of whether the underlying count is available.
3. THE Circuit_Breaker_Registry SHALL NOT alter, pause, or reset any Harness's iteration count or circuit-breaker state; it is read-only.
4. IF a Harness's current iteration count is unavailable at call time, THEN THE Circuit_Breaker_Registry SHALL include that Harness in the `breakers` report with the configured bound together with the current-iteration-count field explicitly marked `"unavailable"`, and SHALL NOT omit the Harness from `breakers` for that reason.

> **VCC translation** (Requirement 5.3): `Verify no Harness configuration file, retry counter, or circuit-breaker state file changes value as a result of a Circuit_Breaker_Registry read call (before/after snapshot diff is empty).`

### Requirement 6: One MCP Surface for OS-Level Visibility (Os_Status_Tool)

**User Story:** As an Operator or External_Agent, I want one MCP tool group I can call locally today (and, where deployed, over the Cloudflare McpAgent) to reach the Process_Registry, Capability_Registry, Cost_Ledger_Aggregator, Gate_Catalog, and Circuit_Breaker_Registry, so that I have a single entry point for OS-level visibility instead of multiple harness-specific calls.

#### Acceptance Criteria

1. THE Os_Status_Tool SHALL be reachable through the existing Local_Mcp_Tool_Catalog (`mcp/server.js`) as one new tool descriptor following the existing `mcp/local-tool-contract.js` descriptor pattern, with `view` values `process_list`, `capabilities`, `cost_summary`, `gate_catalog`, and `circuit_breakers`.
2. THE Os_Status_Tool SHALL expose the same capabilities through the Cloudflare_Mcp_Agent tool surface using the existing keyless MCP Streamable HTTP forwarding pattern regardless of whether that runtime is currently deployed or running, thereby preserving the Spend_Isolation_Boundary.
3. WHEN any underlying registry call fails, THE Os_Status_Tool SHALL return a structured error result rather than throwing an uncaught exception.
4. THE Os_Status_Tool SHALL emit a Cost_Log entry with `estimated_cost_usd`, `prompt_tokens`, and `completion_tokens` each computed as exactly `0` for every one of its own calls, because the Os_Status_Tool performs zero model calls; the Agentic_OS SHALL NOT add a suppression mechanism that artificially clamps a non-zero value to `0`. IF a non-zero `estimated_cost_usd`, `prompt_tokens`, or `completion_tokens` value is computed for an Os_Status_Tool call, THEN THE Os_Status_Tool SHALL fail that call with a Harness_Tool_Error rather than return a successful result carrying the non-zero value, so the underlying defect cannot pass silently as a successful read.
5. THE Os_Status_Tool SHALL be listed as its own entry in the Capability_Registry it exposes, so that its own presence is discoverable the same way as any other Harness capability.

> **VCC translation** (Requirement 6.3): `Verify every Os_Status_Tool integration test that induces an underlying registry failure receives a { ok:false, errorCode } result and the test process exits 0 with no unhandled rejection logged.`

### Requirement 7: TCO-Zero and No New Dependency by Default (Guardrail)

**User Story:** As an Operator applying the TCO-zero and FOSS-first lenses, I want the Agentic OS to introduce zero new paid services and zero new runtime dependencies unless explicitly justified, so that unifying visibility does not add ongoing cost or maintenance burden disproportionate to its value.

#### Acceptance Criteria

1. THE Agentic_OS SHALL be implemented using only libraries and services already present in the knowgrph dependency tree (`@modelcontextprotocol/sdk`, existing Cloudflare primitives, existing `contracts/*` schemas). IF any future increment of the Agentic_OS proposes introducing a dependency not already present in that tree, THEN THE Agentic_OS SHALL require that dependency be explicitly flagged and justified in the spec/design ADR trail per the FOSS-first ADR requirement *before* that dependency is introduced into any Agentic_OS module, and SHALL NOT introduce it on the basis of a post-hoc justification recorded after the dependency is already in use.
2. THE Agentic_OS SHALL introduce zero new persistent datastore; all five read views SHALL be computed at read time from existing state sources (files, D1 rows, Durable Object state, in-memory Director state) rather than written to a new table, bucket, or KV namespace.
3. IF a future increment of the Agentic_OS requires a new dependency or datastore, THEN the design document for that increment SHALL record an ADR with a FOSS-alternative comparison and a 12-month TCO estimate before implementation.
4. THE Agentic_OS SHALL preserve the Spend_Isolation_Boundary and the Approval_Gate_Invariant exactly as defined by `knowgrph-acos-mcp-connector`; no requirement in this document SHALL be interpreted to permit a new tier to hold a model key or to bypass a verified Approval_Token.
5. THE Agentic_OS SHALL introduce no external infrastructure tier: no Vercel, Supabase, or AWS runtime, host, or datastore SHALL be referenced by any Agentic_OS module (native-in-repo consolidation, ADR-3).

> **VCC translation** (Requirement 7.2): `Verify package manifests and lockfiles (package.json/package-lock.json where present) across canvas/, mcp/, contracts/, and cloudflare/workers/knowgrph-mcp show zero added dependencies attributable to this increment; no new D1 migration, R2 bucket, KV namespace, or Durable Object class is present in the diff; and no aws/, vercel*, or supabase* runtime configuration is referenced by any Agentic OS module.`

### Requirement 8: Durable HITL Approval_Token Store on the Control-Plane Worker (Track A)

**User Story:** As an Operator, I want an Approval_Token minted on the deployed control-plane Worker to still verify and single-use-consume after the Worker restarts, so that remote approval is trustworthy rather than reset by every isolate recycle.

#### Acceptance Criteria

1. THE Cloudflare_Mcp_Agent SHALL persist every minted Approval_Token to a Durable_Approval_Token_Store that implements the same `store` interface (`save`, `get`, `has`, `delete`, `list`, `clear`, `size`) already defined by `mcp/video-remix/approval-token-issuer.js`'s in-memory store seam.
2. WHEN the Cloudflare_Mcp_Agent Worker restarts within an Approval_Token's 15-minute TTL, THE Durable_Approval_Token_Store SHALL return that Approval_Token, unconsumed, on the next `verify` call for its `gateId`.
3. WHEN a stage tool call on the Cloudflare_Mcp_Agent presents a valid, unexpired, unconsumed Approval_Token stored in the Durable_Approval_Token_Store, THE Cloudflare_Mcp_Agent SHALL permit the gated stage exactly as it does today against the in-memory store, and SHALL mark that Approval_Token consumed in the Durable_Approval_Token_Store immediately after the permitted spend completes.
4. IF a stage tool call or a standalone verification request presents, or references by id, an Approval_Token already marked consumed in the Durable_Approval_Token_Store, THEN THE Cloudflare_Mcp_Agent SHALL reject that call or request with reason `consumed` and SHALL NOT execute the gated action, identically to the existing in-memory single-use behavior, regardless of whether a gated stage tool call is actually attempted.
5. THE Durable_Approval_Token_Store SHALL NOT modify `APPROVAL_GATE_IDS`, `APPROVAL_TOKEN_TTL_MS`, or any field of the existing `ApprovalGate`/`Approval_Token` shape defined by `contracts/approval.schema.js`.
6. THE Durable_Approval_Token_Store SHALL be implemented using a Cloudflare Worker KV namespace binding already available to the `knowgrph-mcp` Worker's deployment target, introducing zero new Durable Object class and zero new external vendor, consistent with Requirement 7.
7. IF marking an Approval_Token consumed in the Durable_Approval_Token_Store fails after its permitted spend has already executed, THEN THE Cloudflare_Mcp_Agent SHALL NOT retry the marking operation as part of that spend's request/response cycle, and SHALL accept that the token may remain valid for a subsequent verify call rather than blocking or reversing the already-completed spend.

> **VCC translation** (Requirement 8.2): `Verify a token minted before a simulated Worker restart (isolate recycle / KV-backed store re-instantiation) still passes verifyGateToken() for its gateId after the restart, and that mcp/__tests__/approval-token-single-use.test.mjs and mcp/__tests__/approval-rejection-path.test.mjs continue to pass unmodified against the KV-backed store adapter.`

### Requirement 9: One Deployed Golden-Path Live Research + Storyboard Proof (Track B)

**User Story:** As an Operator, I want one approved live Director run on the deployed control-plane Worker to complete research and storyboard against real providers, so that the env-gated live path is proven end-to-end rather than only unit-tested locally.

#### Acceptance Criteria

1. WHERE `KNOWGRPH_LIVE_CLIENTS` is set truthy or an `EXA_API_KEY` is present on the deployed Cloudflare_Mcp_Agent Worker, THE Video_Remix Director SHALL resolve the research stage's client through `resolveStageClients()` to the live Exa client rather than the deterministic mock.
2. WHEN a Live_Golden_Path_Run's research stage executes against the live Exa client with a verified `paid-model-call` Approval_Token, THE Research_Harness SHALL return at least 3 cited sources and THE Video_Remix Director SHALL emit a Cost_Log entry for that stage that passes `validateCostLog()`.
3. WHEN a Live_Golden_Path_Run's storyboard stage executes with `AI_GATEWAY_CHAT_URL` configured and a verified `paid-model-call` Approval_Token, THE Storyboard_Harness SHALL emit a Kgc_Document that validates against the `kgc-computing-flow/v1` schema with `flow.nodes[]` count equal to the planned shot count. WHEN the storyboard stage's execution completes successfully, THE Video_Remix Director SHALL count that stage as a Live_Golden_Path_Run stage success regardless of whether the emitted Kgc_Document passes that schema validation or `flow.nodes[]` count check; a schema-validation failure or a `flow.nodes[]` count mismatch on the emitted Kgc_Document SHALL be handled as a storyboard-harness-level output-validation/fallback concern (governed by the existing `knowgrph-acos-mcp-connector` Requirement 7.4/7.5 fallback behavior) and SHALL NOT be treated as a Live_Golden_Path_Run stage-failure condition.
4. IF `KNOWGRPH_LIVE_CLIENTS` is unset and no `EXA_API_KEY` is present, THEN THE Video_Remix Director SHALL make zero calls to any live provider client and SHALL resolve every spend-bearing step to the existing deterministic mock, exactly as it does today, regardless of whether an Approval_Token is presented. IF `KNOWGRPH_LIVE_CLIENTS` is set truthy or an `EXA_API_KEY` is present but no verified, unexpired, unconsumed Approval_Token is presented for a gated stage, THEN `resolveStageClients()` MAY still succeed in resolving a live client descriptor for that stage, but THE Video_Remix Director SHALL prevent that stage from proceeding to any live provider call: successful client resolution alone SHALL NOT constitute or authorize a live provider call, and THE Video_Remix Director SHALL route that stage to the deterministic mock rather than allow the call to proceed to the live provider and fail there.
5. THE Cloudflare_Mcp_Agent SHALL persist the Live_Golden_Path_Run's Run_Manifest through the existing Run_Manifest Durable Object store, retrievable via `GET /knowgrph/control-plane/mcp/runs/{id}` after the run reaches a terminal Run_State.
6. THE Video_Remix Director SHALL NOT invoke the render or commerce live clients as part of this requirement; render and commerce remain scaffolded (`requiresAsyncHarness`) and are explicitly out of scope for this increment (see Out of Scope).
7. THE Video_Remix Director SHALL make a live provider call for a gated stage if and only if both `KNOWGRPH_LIVE_CLIENTS` is set truthy or the relevant provider API key is present, AND a verified, unexpired, unconsumed Approval_Token for that stage's gate is presented; a defect in evaluating either condition SHALL fail closed to the deterministic mock rather than permit a live call. Live client resolution succeeding (per 9.4) is not equivalent to the call proceeding: WHEN live client resolution succeeds but the Approval_Token condition is not satisfied, THE Video_Remix Director SHALL prevent the live provider call entirely and route to the deterministic mock rather than allow the call to reach the provider and fail there.

> **VCC translation** (Requirement 9.2): `Verify mcp/__tests__/research-harness.test.mjs and mcp/__tests__/director-live-run.test.mjs pass with a live-mode fixture (injected fetch or fixture Exa client) that returns >=3 sources and a validateCostLog()-passing Cost_Log entry, and that an unset KNOWGRPH_LIVE_CLIENTS / missing EXA_API_KEY fixture run records zero live-client invocations.`

### Requirement 10: Agentic Canvas OS Dashboard Rendered on Canvas (Track C)

**User Story:** As an Operator, I want the `knowgrph.agentic_canvas_os.plan` dry-run output visible as a dashboard on Canvas, so that I can inspect cross-repo build state visually instead of reading a raw JSON manifest.

#### Acceptance Criteria

1. WHEN `knowgrph.agentic_canvas_os.plan` produces a dry-run manifest, THE Dashboard_Document SHALL render that manifest as a `kgc-computing-flow/v1` frontmatter-flow Markdown document with one `flow.nodes[]` entry per lane present in the manifest (e.g. `market_radar`, `browser_evidence`, `market_to_artifact`, `learning_loop`, `starter_repo`).
2. WHEN an Operator opens the Dashboard_Document in Canvas, THE existing `applyChatKgcWorkspaceDocumentToCanvas` apply path SHALL project the Dashboard_Document's `flow.nodes[]` onto the Storyboard Widget without invoking a second or parallel apply pipeline.
3. THE Dashboard_Document SHALL be derived at generation time from the `knowgrph.agentic_canvas_os.plan` dry-run manifest and SHALL NOT expose file-write, deploy, paid-call, or payment mutation actions from the Canvas rendering surface. WHERE the underlying frontmatter-flow schema or an existing Canvas component — regardless of whether that rendering component is newly built for this requirement or reused from an existing Canvas pipeline — would otherwise carry a file-write, deploy, paid-call, or payment mutation affordance, THE Dashboard_Document rendering path SHALL actively filter that affordance out before the document is opened in Canvas, rather than relying on the affordance being absent by construction or exempting a newly-built component from filtering on the assumption it would not carry that affordance in the first place.
4. IF the `knowgrph.agentic_canvas_os.plan` manifest for a lane is absent or unreadable, THEN THE Dashboard_Document SHALL omit that lane's node and SHALL NOT fabricate a placeholder node in its place.
5. THE Dashboard_Document rendering path SHALL introduce no new Canvas renderer, parser, or graph-apply component; it SHALL reuse the existing frontmatter-flow schema, `applyChatKgcWorkspaceDocumentToCanvas`, and Storyboard Widget exactly as they exist today.
6. WHEN the Dashboard_Document rendering path filters out a file-write, deploy, paid-call, or payment mutation affordance per Acceptance Criterion 3, THE Dashboard_Document rendering path SHALL log or alert that filtering event rather than silently dropping the affordance with no record.

> **VCC translation** (Requirement 10.2): `Verify a fixture dashboard.agentic-os.md document generated from a knowgrph.agentic_canvas_os.plan dry-run manifest applies through applyChatKgcWorkspaceDocumentToCanvas with node count equal to the manifest's lane count, and that no second Canvas apply function is introduced or invoked for this document type.`

### Requirement 11: MCP Gateway Surface-Selection Contract (Capability_Class Routing)

**User Story:** As an External_Agent, I want a documented mapping from each capability's spend class to the correct one of the four existing MCP surfaces, so that I do not have to guess or trial-and-error which surface to call.

#### Acceptance Criteria

1. THE Capability_Registry SHALL assign exactly one Capability_Class — `read_only_discovery` or `approval_gated_spend` — to every Capability_Entry it returns, derived from the owning Harness's already-declared approval-gate association (present when the capability appears in the Gate_Catalog) or its absence (read-only otherwise).
2. THE Capability_Registry response SHALL report, for every Capability_Entry with Capability_Class `approval_gated_spend`, exactly one recommended Mcp_Gateway_Surface value of `cloudflare_mcp_agent`, and for every Capability_Entry with Capability_Class `read_only_discovery`, at least one recommended Mcp_Gateway_Surface drawn from the sourceCatalogs already reported for that entry.
3. THE Gateway_Routing_Contract SHALL be expressed entirely as data returned by the existing Capability_Registry and documented in the design/TAD; it SHALL NOT be implemented as a new dispatcher process, proxy, or fifth Mcp_Gateway_Surface (ADR-4/ADR-006 preserved).
4. WHERE an External_Agent calls a capability whose Capability_Class is `approval_gated_spend` on a Mcp_Gateway_Surface other than the Cloudflare_Mcp_Agent control-plane surface, THE owning Harness's existing approval-gate enforcement SHALL reject the call exactly as it does today for any unapproved spend attempt. IF that existing approval-gate enforcement fails to reject such a call, THEN THE Gateway_Routing_Contract SHALL be documented to require an explicit surface-boundary check at the Mcp_Gateway_Surface's own dispatch point (not a new dispatcher process) that rejects any `approval_gated_spend` capability call arriving on a Mcp_Gateway_Surface other than `cloudflare_mcp_agent`, so the call is never routed to the owning Harness at all. THE surface-boundary check's own failure mode SHALL be fail-closed (reject the call) rather than fail-open (permit the call), so that no scenario exists in which both the owning Harness's approval-gate check and the surface-boundary check fail and the call is nonetheless permitted to proceed. THE Capability_Registry SHALL classify every `read_only_discovery` capability's `recommendedSurfaces` as drawn only from the Local_Mcp_Tool_Catalog, the Pages HTTP MCP surface, or the browser WebMCP surface, and SHALL NOT recommend the Cloudflare_Mcp_Agent control-plane surface for a `read_only_discovery` capability. This surface reservation is a routing recommendation steering `read_only_discovery` calls toward the other three surfaces, not a hard block on the control-plane surface: a `read_only_discovery` capability MAY still be successfully called on the Cloudflare_Mcp_Agent control-plane surface where no rejection mechanism activates for it. THE enforcement and defense-in-depth guarantee in this Acceptance Criterion applies specifically to `approval_gated_spend` calls arriving on a Mcp_Gateway_Surface other than `cloudflare_mcp_agent`, and SHALL NOT be read to require rejection of a `read_only_discovery` call arriving on the Cloudflare_Mcp_Agent control-plane surface.
5. THE `.well-known/mcp/server-card.json` metadata SHALL reference the Gateway_Routing_Contract by linking to, or embedding a summary of, the Capability_Class/Mcp_Gateway_Surface mapping already exposed by `knowgrph.os.status` (`view:"capabilities"`), so an External_Agent can resolve routing from either the server card or the capabilities view.

> **VCC translation** (Requirement 11.1): `Verify every Capability_Entry returned by knowgrph.os.status (view:"capabilities") carries a capabilityClass field whose value is exactly "read_only_discovery" or "approval_gated_spend", with every gate-catalog-associated tool id classified "approval_gated_spend" and every other tool id classified "read_only_discovery", against a fixture union of the three source catalogs.`

### Requirement 12: Consistent Error and Timeout Semantics Across MCP Gateway Surfaces

**User Story:** As an External_Agent, I want every one of the four MCP surfaces to fail in the same shape and within a stated timeout, so that my error-handling code does not need a special case per surface.

#### Acceptance Criteria

1. WHEN any of the four Mcp_Gateway_Surfaces returns a tool-call failure, THE responding surface SHALL shape that failure as `{ ok: false, errorCode: string, message: string }`, matching the Harness_Tool_Error shape already used by `knowgrph.os.status`.
2. THE design/TAD for this increment SHALL document one stated per-surface timeout bound for each of the four Mcp_Gateway_Surfaces (local stdio, Pages HTTP MCP, browser WebMCP, Cloudflare_Mcp_Agent control plane), reusing whichever bound each surface already enforces today rather than introducing a new timer.
3. IF a Mcp_Gateway_Surface cannot reach an underlying dependency (e.g. an unreachable Cloudflare_Mcp_Agent catalog from Capability_Registry, or an unreadable Harness state source from Process_Registry), THEN THE surface SHALL return `ok:true` with the failure named in the existing `unavailableSources`/`unreachableCatalogs` metadata field when the overall call can still partially succeed, and SHALL return the Harness_Tool_Error shape only when the call cannot proceed at all. WHEN the call cannot proceed at all because a dependency is unreachable, THE returned Harness_Tool_Error SHALL still include whatever `unavailableSources`/`unreachableCatalogs` metadata is determinable at the point of failure, rather than omitting that metadata solely because the overall call failed.
4. THE Gateway_Routing_Contract documentation SHALL record, for each of the four Mcp_Gateway_Surfaces, which of the two failure modes in Acceptance Criterion 3 applies to which capability, so an External_Agent can distinguish "partial degradation" from "call failed" without probing empirically. THE design/TAD for this increment SHALL NOT be considered complete for Requirement 12 until this per-surface, per-capability failure-mode documentation exists in full; a surface whose error handling works correctly SHALL NOT be exempted from this documentation requirement.

> **VCC translation** (Requirement 12.1): `Verify one example test per Mcp_Gateway_Surface asserts that an induced tool-call failure on that surface returns an object with ok:false, a non-empty string errorCode, and a non-empty string message, using each surface's own existing test fixtures.`

### Requirement 13: Per-Tool Usage Guidance in the Capability_Registry

**User Story:** As an External_Agent, I want to see how to call a capability directly from the capabilities view, so that I can proceed without a separate documentation fetch.

#### Acceptance Criteria

1. WHERE an existing tool descriptor (in `mcp/local-tool-contract.js`, `mcp/README.md`, or an equivalent already-published source) already documents a usage example for a capability, THE Capability_Registry SHALL surface that example as a Tool_Usage_Example field on the corresponding Capability_Entry.
2. THE Capability_Registry SHALL NOT fabricate a Tool_Usage_Example for a capability that has no existing documented example; the field SHALL be omitted for that entry rather than populated with invented content, and the response SHALL carry no additional field or message suggesting an example might exist elsewhere.
3. THE Tool_Usage_Example field, WHEN present, SHALL be a static string or example-arguments object copied from its existing documented source, computed at read time, and SHALL introduce no new persistent documentation store.
4. IF the Capability_Registry's extraction of a Tool_Usage_Example from an existing documented source fails to parse that source's documented example, THEN THE Capability_Registry SHALL omit the Tool_Usage_Example field for that entry and SHALL record the parse failure in response metadata rather than silently returning a partial or malformed example.

> **VCC translation** (Requirement 13.1): `Verify every Capability_Entry whose owning tool id has a documented example in mcp/README.md's tool-call examples carries a non-empty usageExample field equal to that documented example, and every Capability_Entry with no documented example has no usageExample field.`

### Requirement 14: Consistent Structured Error Shape Across Harness Tool Calls

**User Story:** As an External_Agent, I want every local MCP tool call to fail with the same structured error shape, so that I can write one error-handling path instead of one per harness.

#### Acceptance Criteria

1. WHEN a local MCP tool call dispatched through `mcp/server.js`'s `CallToolRequestSchema` handler fails for any reason (validation error, unknown tool, harness-internal error), THE handler SHALL return a Harness_Tool_Error (`{ ok: false, errorCode, message }`) rather than an unstructured text-only error content block.
2. THE Agentic_OS SHALL NOT alter the specific `errorCode` values or success-path response shapes any existing harness (Showrunner, Video_Remix Director, SuperAgent Harness, Memory Layer, HTML Video Renderer) already returns; this requirement normalizes the failure-path envelope only, wrapping each harness's existing error information into the `errorCode`/`message` fields without discarding it. Regardless of whether a given harness's current failure path is already structured or is unstructured text, THE `mcp/server.js` dispatch boundary SHALL standardize every harness's failure-path response to the Harness_Tool_Error envelope, so no harness is left on an inconsistent (structured-for-some-calls, unstructured-for-others) error shape after this requirement is implemented.
3. IF an existing harness tool already returns a structured `{ok:false, error:{code, message}}` shape (e.g. Showrunner), THEN THE Harness_Tool_Error normalization SHALL map that shape's `error.code` to `errorCode` and `error.message` to `message` without requiring a change to that harness's internal handler. IF an existing harness tool returns an unstructured text-only error, THEN THE Harness_Tool_Error normalization SHALL wrap that text as `message` with a generic non-empty `errorCode` (e.g. `harness_error`), again without requiring a change to that harness's internal handler.
4. THE Capability_Registry SHALL report, for every Capability_Entry, that its owning Harness now returns the Harness_Tool_Error shape on failure, so an External_Agent can rely on one error contract across the full capability union.

> **VCC translation** (Requirement 14.1): `Verify one induced-failure example test per existing harness handler (Showrunner, Video_Remix Director, SuperAgent, Memory Layer, HTML Video Renderer) dispatched through mcp/server.js's CallToolRequestSchema receives a result containing ok:false, a non-empty string errorCode, and a non-empty string message, with each harness's existing success-path response shape unchanged.`

### Requirement 15: Documented Agent Onboarding Sequence (Discovery → Capability → Approval → Spend)

**User Story:** As an External_Agent operator, I want one documented sequence describing how to move from discovery to an approved spend-bearing call, so that I do not have to reverse-engineer the four-surface topology from source.

#### Acceptance Criteria

1. THE design/TAD for this increment SHALL document the Agent_Onboarding_Sequence as four ordered steps: discover capabilities, call a `read_only_discovery` capability, request an Approval_Token for an `approval_gated_spend` capability, and present that token on the Cloudflare_Mcp_Agent control-plane surface.
2. THE Agent_Onboarding_Sequence documentation SHALL reference only Mcp_Gateway_Surfaces, capabilities, and approval mechanisms that already exist at the time this requirement is implemented; it SHALL NOT describe a step that requires a new surface, a new token-issuance mechanism, or a new dependency.
3. THE `knowgrph.os.status` (`view:"capabilities"`) response SHALL remain sufficient, on its own, to identify every capability an External_Agent needs to complete Agent_Onboarding_Sequence step 2 (at least one `read_only_discovery` entry) and, WHERE at least one `approval_gated_spend`-classified capability exists in the deployment, step 3 (at least one `approval_gated_spend` entry with a named gate id resolvable in the Gate_Catalog). A deployment with zero `approval_gated_spend`-classified capabilities in the Gate_Catalog SHALL NOT be treated as failing this Acceptance Criterion; per Acceptance Criterion 4, that condition instead marks step 3 inapplicable.
4. THE Agent_Onboarding_Sequence documentation SHALL always describe all four steps in full, for every deployment, regardless of whether the current deployment has any `approval_gated_spend`-classified capability. IF the Gate_Catalog reports zero `approval_gated_spend`-classified capabilities at call time, THEN step 3 and step 4 of the documented sequence SHALL be marked inapplicable for that deployment rather than omitted from the documentation, and no error SHALL be raised solely because no spend-bearing capability exists.

> **VCC translation** (Requirement 15.3): `Verify a fixture External_Agent walkthrough test resolves at least one read_only_discovery Capability_Entry and, when the Gate_Catalog is non-empty, at least one approval_gated_spend Capability_Entry with a gateId present in the gate_catalog view's gates[] set, entirely from knowgrph.os.status responses with no other tool call required.`

## MoSCoW Priority (with ROI Scoring)

ROI Score = (User Impact × Reach) / (Build Hours + Monthly TCO + Token Cost/Month), per the PRD & TAD Guidelines ROI template. Reach is estimated at ~150 harness-invoking sessions/month (Operator sessions plus agent/evaluation runs), consistent with the platform-wide ~200 MAU baseline in `knowgrph-tech-stack-document.md` scoped down to sessions that actually touch a Harness. Token Cost/Month is $0 for every item in Requirements 1–8 and 10–15 because every capability is read-only and performs zero model calls; Requirement 9 (Track B) is the sole exception and is explicitly gated/bounded (est. < $0.03/live golden-path run per the follow-on PRD/TAD's Orchestration/Harness Flow token budget).

| Tier | Requirement | User Impact (1–5) | Reach | Build Hours | Monthly TCO | Token Cost/Mo | ROI Score | Rationale |
|---|---|---|---|---|---|---|---|---|
| Must | R6 Os_Status_Tool (entry point) | 5 | 150 | 10 | $0 | $0 | 75.0 | Delivery vehicle for every other item; zero value without it |
| Must | R1 Process_Registry | 4 | 150 | 8 | $0 | $0 | 75.0 | Highest-frequency Operator pain today: no single place to see what is running |
| Must | R2 Capability_Registry | 4 | 150 | 6 | $0 | $0 | 100.0 | Cheapest to build (pure union of 3 existing catalogs); high discoverability value for External_Agent onboarding |
| Must | R7 TCO-zero guardrail | — | — | 2 | $0 | $0 | n/a (guardrail) | Non-negotiable cost-avoidance constraint, not a value-scored feature |
| Must | R8 Durable HITL Approval_Token store (Track A) | 4 | 150 | 6 | $0 | $0 | 100.0 | Smallest deploy diff (KV adapter behind an existing interface); unblocks trustworthy remote approval for every other spend-bearing capability |
| Should | R3 Cost_Ledger_Aggregator | 4 | 150 | 10 | $0 | $0 | 60.0 | High value (spend visibility) but requires per-Harness gap analysis; slightly more build risk than Must-tier items |
| Should | R4 Gate_Catalog extension | 3 | 150 | 8 | $0 | $0 | 56.25 | Improves consistency but Showrunner already has a working (if bespoke) approval action; not blocking |
| Should | R11 MCP Gateway surface-selection contract | 4 | 150 | 8 | $0 | $0 | 75.0 | Closes the ambiguity that makes external agents guess which of the four surfaces to call; pure data/documentation over existing catalogs |
| Should | R14 Consistent structured error shape across harness tool calls | 3 | 150 | 6 | $0 | $0 | 75.0 | One error contract instead of N ad hoc shapes; cheap normalization at the existing mcp/server.js dispatch boundary |
| Could | R5 Circuit_Breaker_Registry | 2 | 150 | 6 | $0 | $0 | 50.0 | Nice-to-have observability; lower incident frequency than the Must-tier items |
| Could | R9 Live golden-path research + storyboard proof (Track B) | 3 | 50 | 10 | $0 | ~$0.03/run | 15.0 | Env-gated clients already exist; needs an operator deploy proof, not new code; lower reach than Must-tier (one-time proof, not a recurring session) |
| Could | R10 Dashboard Canvas render (Track C) | 3 | 100 | 8 | $0 | $0 | 37.5 | Dry-run plan exists; reuses existing frontmatter-flow apply path; visual polish rather than a new capability |
| Could | R12 Consistent error/timeout semantics across MCP Gateway surfaces | 3 | 150 | 6 | $0 | $0 | 75.0 | Documentation + response-shape normalization over surfaces that already enforce their own timeouts |
| Could | R13 Per-tool usage guidance in Capability_Registry | 2 | 150 | 4 | $0 | $0 | 75.0 | Cheap read-time projection of already-documented examples; nice-to-have onboarding polish |
| Could | R15 Documented agent onboarding sequence | 2 | 150 | 3 | $0 | $0 | 100.0 | Pure documentation over existing capabilities; near-zero build cost |
| Won't (this increment) | Shared in-process scheduler/dispatcher across Harnesses | — | — | — | — | — | — | Harnesses run on three incompatible runtimes (Cloudflare Workers, Node stdio, Python subprocess); forcing one dispatcher fails min-viable-max-value and risks the Spend_Isolation_Boundary |
| Won't (this increment) | New persistent OS-level datastore beyond the Durable_Approval_Token_Store KV adapter | — | — | — | — | — | — | Violates R7 TCO-zero guardrail; every registry remains computable at read time from existing state; R8's KV adapter is the sole, explicitly justified exception |
| Won't (this increment) | Visual dashboard/UI for the Process/Capability/Cost/Gate/Circuit-Breaker registries | — | — | — | — | — | — | Out of scope for an MCP-tool-surface requirements increment; distinct from R10's Agentic Canvas OS Dashboard_Document, which renders a different, already-existing dry-run plan artifact through an already-existing Canvas pipeline |
| Won't (this increment) | Automatic/auto-continuing approval of any Approval_Gate | — | — | — | — | — | — | Would directly weaken the Approval_Gate_Invariant; explicitly forbidden |
| Won't (this increment) | A fifth MCP proxy tier or monolithic gateway process | — | — | — | — | — | — | ADR-4/ADR-006 forbid a fifth surface; R11/R12 mature the existing four surfaces instead |
| Won't (this increment) | Async live render + commerce clients on the deployed Worker | — | — | — | — | — | — | Requires the render/commerce harnesses to go async first (`requiresAsyncHarness` scaffold); deferred to a later increment per the follow-on PRD/TAD execution sequence |
| Won't (this increment) | Full Market Radar / browser-evidence live capture, Starter Repo file writes, or Learning Loop skill auto-promotion | — | — | — | — | — | — | Explicitly excluded by the follow-on PRD/TAD's Min-Viable Scope; each requires either live browser capture, file-write mutation, or auto-promotion that is not min-viable for this increment |
| Won't (this increment) | New Vercel/AWS/Supabase runtime tier for any Track A/B/C or gateway-maturity capability | — | — | — | — | — | — | ADR-3 forbids; every new capability in this revision is implemented on the existing Cloudflare + local stack |

## Min-Viable Scope

The runtime-ready deliverable is the Os_Status_Tool (Requirement 6) exposing all five read capabilities — `process_list`, `capabilities`, `cost_summary`, `gate_catalog`, and `circuit_breakers` — through the existing Local_Mcp_Tool_Catalog, with the same tool name and zero-token read-view contract advertised by the Cloudflare_Mcp_Agent where deployed. The surface remains read-only and subject to the Requirement 7 guardrails.

This revision's Min-Viable Scope for the folded-in Must/Should tiers additionally requires:

1. A Durable_Approval_Token_Store KV adapter on the `knowgrph-mcp` Worker implementing the existing `store` interface (Requirement 8), with `mcp/__tests__/approval-token-single-use.test.mjs` and `mcp/__tests__/approval-rejection-path.test.mjs` passing unmodified against it.
2. The Capability_Registry reporting a Capability_Class and recommended Mcp_Gateway_Surface for every capability entry (Requirement 11), and every local MCP tool-call failure normalized to the Harness_Tool_Error shape (Requirement 14).
3. Requirements 9 (Track B live proof), 10 (Track C dashboard render), 5 (Circuit_Breaker_Registry), 12 (surface timeout documentation), 13 (usage guidance), and 15 (onboarding sequence) are Could-tier and MAY ship in a later sprint without blocking this increment's Must/Should deliverable.

Explicitly excludes (Could-tier, deferred): async live render/commerce on the Worker, full Market Radar/browser-evidence live capture, Starter Repo file writes, and Learning Loop skill auto-promotion — consistent with the follow-on PRD/TAD's own Min-Viable Scope exclusions.

## Success Metrics

| Metric | Baseline | Target | Timeline |
|---|---|---|---|
| Operator time to answer "what is running right now?" | Manual query across 3+ harness-specific tools (~3–5 min) | ≤ 1 Os_Status_Tool call (~10 s) | First increment ship |
| Time-to-value (TTV steps) | N/A (capability does not exist) | ≤ 2 steps (start local MCP server if not already running; call `process-list`) | First increment ship |
| Time-to-value (TTV elapsed) | N/A | ≤ 1 minute on a clean checkout with local MCP server already running for other work | Validated before Phase 3 sign-off |
| HITL Approval_Token survives Worker restart (Track A) | Not durable; local in-memory store only | Verified via KV-backed store adapter; existing single-use + rejection tests pass unmodified | This revision, Must tier |
| Live golden-path research + storyboard proof (Track B) | Proven only in local unit tests | One deployed, approved Live_Golden_Path_Run with a persisted Run_Manifest | This revision, Could tier |
| Agentic Canvas OS dashboard visibility (Track C) | Dry-run JSON manifest only | Dashboard_Document renders on Canvas via the existing apply path | This revision, Could tier |
| MCP Gateway surfaces with a documented Capability_Class routing contract | 0 of 4 | 4 of 4, expressed via `knowgrph.os.status` (`view:"capabilities"`) | This revision, Should tier |
| Harness tool-call failure shapes unified | 3+ ad hoc shapes across harnesses | 1 (`Harness_Tool_Error`) across every local MCP tool call | This revision, Should tier |
| Harnesses covered by Process_Registry | 0 of 3 run-state sources unified | 3 of 3 (Pipeline_Run, Run_Manifest, Superagent_Run) | First increment ship |
| Capability catalogs unified | 0 of 3 (each queried separately) | 3 of 3 (Vdeoxpln_Registry, Local_Mcp_Tool_Catalog, Cloudflare_Mcp_Agent) | First increment ship |
| Cost_Emission_Gap harnesses identified | Unknown before Agentic OS | 100% of model-bearing Harnesses classified as covered or gapped | Runtime-ready increment |
| Token cost / month (Os_Status_Tool itself) | N/A | $0 (read-only, zero model calls) | Ongoing |
| Monthly TCO (Agentic_OS itself) | N/A | $0 (no new datastore, no new paid service) | Ongoing |
| HITL Approval_Token survives Worker restart (Track A) | Not durable; local in-memory store only | Verified via KV-backed store adapter; existing single-use + rejection tests pass unmodified | This revision, Must tier |
| Live golden-path research + storyboard proof (Track B) | Proven only in local unit tests | One deployed, approved Live_Golden_Path_Run with a persisted Run_Manifest | This revision, Could tier |
| Agentic Canvas OS dashboard visibility (Track C) | Dry-run JSON manifest only | Dashboard_Document renders on Canvas via the existing apply path | This revision, Could tier |
| MCP Gateway surfaces with a documented Capability_Class routing contract | 0 of 4 | 4 of 4, expressed via `knowgrph.os.status` (`view:"capabilities"`) | This revision, Should tier |
| Harness tool-call failure shapes unified | 3+ ad hoc shapes across harnesses | 1 (`Harness_Tool_Error`) across every local MCP tool call | This revision, Should tier |
| Token cost / month (Requirements 8, 10–15) | N/A | $0 (read-only / documentation-only) | Ongoing |
| Token cost / month (Requirement 9, Track B) | N/A | ≤ $25 at demo load, bounded by Approval_Gates + mock default | Ongoing, Could tier |
| Monthly TCO delta (this revision) | $0 | $0 (Worker KV reuses an existing binding category; no new vendor) | Ongoing |
| ROI Score threshold | — | ≥ 50 (per table above) for any item promoted out of Won't | Per sprint review |

## Time-to-Value

### Time-to-Value: knowgrph Agentic OS (Must-tier increment)

| Dimension | Estimate | Target ceiling | Validation method |
|---|---|---|---|
| TTV steps | 2 steps (ensure local MCP server running; call `process-list`) | ≤ 2 steps | Walk-through on a clean `knowgrph` checkout |
| TTV elapsed time | ~1 min | ≤ 2 min | Timed first-run test against `mcp/server.js` |
| First-value action | Operator receives a normalized Process_Entry list spanning all 3 existing run-state sources in one response | — | Observable JSON output from the Os_Status_Tool call |
| Persona | Operator (solo founder) | — | Defined above in Glossary |

### Time-to-Value: Durable HITL Approval_Token Store (Track A, Requirement 8)

| Dimension | Estimate | Target ceiling | Validation method |
|---|---|---|---|
| TTV steps | 3 steps (bind a KV namespace to the `knowgrph-mcp` Worker; deploy; mint + verify one token across a simulated restart) | ≤ 3 steps | Deploy runbook walk-through against a staging Worker |
| TTV elapsed time | ~5 min | ≤ 10 min | Timed deploy + verification test |
| First-value action | Operator confirms a minted Approval_Token still verifies after a Worker restart | — | `mcp/__tests__/approval-token-single-use.test.mjs` passing against the KV adapter |
| Persona | Operator (solo founder) | — | Defined above in Glossary |

### Time-to-Value: Live Golden-Path Proof (Track B, Requirement 9)

| Dimension | Estimate | Target ceiling | Validation method |
|---|---|---|---|
| TTV steps | 5 steps (set `KNOWGRPH_LIVE_CLIENTS` + Exa/AI Gateway env; deploy; mint approval tokens; submit run; read manifest) | ≤ 5 steps | Follow-on PRD/TAD Min-Viable Scope walk-through |
| TTV elapsed time | ~15 min | ≤ 30 min | Timed deploy + one approved run |
| First-value action | Operator reads a persisted Run_Manifest with cited research sources and a validated storyboard Kgc_Document | — | `GET /knowgrph/control-plane/mcp/runs/{id}` response |
| Persona | Operator (solo founder) | — | Defined above in Glossary |

### Time-to-Value: Dashboard Canvas Render (Track C, Requirement 10)

| Dimension | Estimate | Target ceiling | Validation method |
|---|---|---|---|
| TTV steps | 2 steps (run `knowgrph.agentic_canvas_os.plan`; open the generated Dashboard_Document in Canvas) | ≤ 2 steps | Walk-through on a clean `knowgrph` checkout |
| TTV elapsed time | ~2 min | ≤ 5 min | Timed dry-run + Canvas open |
| First-value action | Operator sees one Storyboard Widget node per plan lane | — | Observable Canvas render |
| Persona | Operator (solo founder) | — | Defined above in Glossary |

### Time-to-Value: MCP Gateway Surface-Selection Contract (Requirements 11–15)

| Dimension | Estimate | Target ceiling | Validation method |
|---|---|---|---|
| TTV steps | 1 step (call `knowgrph.os.status` with `view:"capabilities"`) | ≤ 1 step | Walk-through against local or deployed `knowgrph.os.status` |
| TTV elapsed time | ~10 s | ≤ 1 min | Timed capabilities-view call |
| First-value action | External_Agent resolves a Capability_Class and recommended Mcp_Gateway_Surface for every capability without a second documentation fetch | — | Observable JSON output from the capabilities view |
| Persona | External_Agent | — | Defined above in Glossary |

## Out of Scope

- A new persistent OS-level database, table, KV namespace, R2 bucket, or Durable Object class dedicated to storing registry state, beyond the Durable_Approval_Token_Store KV adapter explicitly justified by Requirement 8.
- A shared in-process scheduler or dispatcher that runs Showrunner, the Video_Remix Director, and the SuperAgent Harness on one common runtime.
- A visual dashboard or web UI for the Process_Registry, Capability_Registry, Cost_Ledger_Aggregator, Gate_Catalog, or Circuit_Breaker_Registry read views — those remain an MCP tool surface only; distinct from Requirement 10's Dashboard_Document, which renders the separate, already-existing `knowgrph.agentic_canvas_os.plan` dry-run artifact.
- Automatic approval, auto-continuation, or bypass of any existing Approval_Gate; the Agentic_OS is read/describe-only with respect to approvals.
- Modifying the internal state machine, retry policy, schema, or persistence format of any existing Harness (Showrunner, Video_Remix Director, SuperAgent Harness, Memory Layer, Visual Annotation Engine, HTML Video Renderer, Video Intelligence pipeline, FloatingPanel Chat → KGC).
- Real-time push, streaming, or webhook delivery of registry updates; every registry is pull/read-only per call.
- Any Vercel frontend tier, Supabase datastore, AWS Agent-API tier, or AWS AgentCore wrapper lane — removed from the runtime topology per the native-in-repo consolidation (ADR-3 in the combined PRD/TAD); the Cloudflare `McpAgent` is the sole remote MCP surface.
- Issuing an actual Approval_Token for the Showrunner stage-approval boundary (Requirement 4 is catalog/description only; token issuance for Showrunner is a candidate for a later increment, see Open Questions).
- A fifth MCP proxy tier or any new monolithic gateway process; Requirements 11 and 12 mature routing/error semantics over the existing four Mcp_Gateway_Surfaces only (ADR-4/ADR-006).
- Async live render or commerce clients on the deployed control-plane Worker; the render/commerce harnesses remain `requiresAsyncHarness` scaffolds until a later increment (see follow-on PRD/TAD execution sequence).
- Full Market Radar / real-browser evidence live capture, Starter Repo file writes, and Learning Loop skill auto-promotion — each remains a dry-run/inspection-only capability per the follow-on PRD/TAD's own Min-Viable Scope exclusions.
- A new caller-authentication or session tier for remote `knowgrph.os.status` or Capability_Registry access; this revision documents routing and error shape only, and does not add Auth_Token/Caller_Identity handling.
- Fabricating a Tool_Usage_Example, Capability_Class, or Mcp_Gateway_Surface value for a capability that has no existing documented source; every field added by Requirements 11–15 SHALL be derived from data or documentation that already exists.

## Dependencies

- `contracts/cost-log.schema.js`, `contracts/approval.schema.js`, `contracts/run-manifest.schema.js`, `contracts/credit-ledger.schema.js` — existing canonical schemas the Agentic_OS reads and validates against; not modified by this spec.
- `mcp/local-tool-contract.js`, `mcp/server.js` — existing local MCP tool descriptor pattern and stdio server the Os_Status_Tool extends; `mcp/server.js`'s `CallToolRequestSchema` handler is the normalization point for Requirement 14.
- `mcp/showrunner-runtime.js` (Pipeline_Run state, `knowgrph.showrunner.approve_stage`) — existing state source read by the Process_Registry and Gate_Catalog.
- `mcp/video-remix-runtime.js` / the Video_Remix Director's Run_Manifest state — existing state source read by the Process_Registry and Cost_Ledger_Aggregator.
- `knowgrph_parser/superagent_harness.py` (`state.json`, `trace.jsonl`, `harness-proof.json`) — existing state source read by the Process_Registry.
- `canvas/src/features/agent-ready/knowgrphVdeoxplnContract.mjs` (`knowgrph.vdeoxpln.list`) — existing capability registry unioned by the Capability_Registry.
- The Cloudflare `McpAgent` worker (`cloudflare/workers/knowgrph-mcp`) — the sole existing remote MCP surface the Os_Status_Tool optionally forwards through, WHERE deployed (the former AWS AgentCore forwarder is removed per ADR-3; `aws/agentcore` is archival reference only).
- The `knowgrph-acos-mcp-connector` spec's Data Models and R11/Property 1 invariants — existing normative source for the Spend_Isolation_Boundary and Approval_Gate_Invariant this spec must not weaken.
- `mcp/video-remix/approval-token-issuer.js` (`store` interface, `consumeSeam()`, `APPROVAL_GATE_IDS`, `APPROVAL_TOKEN_TTL_MS`) — existing issuance/verification module the Durable_Approval_Token_Store (Requirement 8) adapts without modification.
- `mcp/video-remix/live-clients.js` (`resolveStageClients()`, `createLiveArgsResolver()`) — existing env-gated client resolver the Live_Golden_Path_Run (Requirement 9) exercises without modification.
- `mcp/local-tool-contract.js`'s `knowgrph.agentic_canvas_os.plan` descriptor and its dry-run manifest output — existing source the Dashboard_Document (Requirement 10) renders.
- The existing frontmatter-flow (`kgc-computing-flow/v1`) schema, `applyChatKgcWorkspaceDocumentToCanvas`, and the Storyboard Widget — existing Canvas apply path the Dashboard_Document (Requirement 10) reuses.
- `cloudflare/workers/knowgrph-mcp/tool-registry.mjs`, `cloudflare/workers/knowgrph-mcp/os-status-tool.mjs`, `cloudflare/pages/knowgrph-agent-ready.mjs`'s `mcpServerCard` — existing MCP Gateway surface definitions Requirements 11 and 12 document routing/error semantics over.
- `docs/documents/knowgrph-agentic-os-follow-on-prd-tad.md` (PRD-FO-1..6, ADR-FO-1, ADR-FO-2) — existing normative source for Track A/B/C scope, MoSCoW tiers, and Out-of-Scope items this revision formalizes as EARS requirements.

## Resolved and Deferred Questions

1. SuperAgent multi-run enumeration is resolved: enumerate subdirectories of `data/superagent-runs/` and include each readable `state.json`; missing or unreadable states are recorded in `unavailableSources`.
2. Tool granularity is resolved: expose one combined MCP tool, `knowgrph.os.status`, with `view` values `process_list`, `capabilities`, `cost_summary`, `gate_catalog`, and `circuit_breakers`.
3. Retention remains deferred: this increment uses the 200-record cap from Requirement 1.5; a time-based window is a later ADR if needed.
4. Showrunner Approval_Token issuance remains deferred: the Gate_Catalog is read/describe-only for this increment.
5. Remote External_Agent auth remains governed by the Cloudflare_Mcp_Agent surface where deployed; this Agentic OS increment adds no new auth/session tier.
6. Cost emission coverage is handled by runtime classification: `costEmissionGaps` reports model-bearing harnesses without schema-valid Cost_Log or Credit_Ledger entries.
7. Durable Approval_Token storage is resolved: implement as a Worker KV namespace adapter behind the existing `store` interface (ADR-FO-1 in the follow-on PRD/TAD), not a new Durable Object class — smallest diff, zero new DO class.
8. Render/commerce live clients remain deferred: they stay `requiresAsyncHarness` scaffolds; Requirement 9 (Track B) exercises only the already-wireable research and storyboard clients.
9. MCP Gateway maturity is resolved as documentation/data over the existing four surfaces (Capability_Class routing, shared error/timeout envelope), never a fifth surface — no design alternative to ADR-4/ADR-006 was considered viable given the native-in-repo and harness-first constraints.
10. Tool usage guidance and error-shape normalization (Requirements 13–14) are resolved as read-time projections of already-existing documentation and already-existing per-harness error information; neither requirement introduces a new documentation store or changes any harness's success-path contract.
