# Requirements Document

## Introduction

This specification refines the existing PRD/TAD (`knowgrph-mcp-agentic-canvas-os-prd-tad.md`, v0.3.1, status `dev-implemented`) and the operator/judge demo doc (`knowgrph-mcp-agentic-canvas-os-demo.md`) into precise, testable requirements for the **knowgrph ↔ agentic-canvas-os MCP connector**. It is a documentation and planning artifact only; no code is to be changed until the user explicitly instructs.

The feature connects two repositories so a single autonomous agent can take a reference video URL plus a creative brief and drive a full, approval-gated loop (research → storyboard → render → publish → checkout) that ends in a sold asset:

- **knowgrph** (control plane and contract SSOT): Cloudflare Workers + Pages, Cloudflare AI Gateway, an MCP server built on the Agents SDK (`McpAgent`) at `airvio.co/knowgrph/mcp`, BytePlus/ModelArk, Exa, and Stripe. knowgrph also owns the canvas engine.
- **agentic-canvas-os** (realized split product shell): a keyless product tier where the **Vercel** frontend and same-origin serverless Agent-API routes are the **primary/default** product path, and the **AWS** Agent-API (API Gateway + Lambda + S3) is the **fallback** path invoked by the browser only on a primary 5xx or transport error. Both tiers hold no model keys, forward `knowgrph.video_remix.run` to the control plane over MCP Streamable HTTP, and embed the live knowgrph canvas rather than reimplementing the renderer.

Requirements are organized around six concerns: (1) User Flow, (2) Work Flow, (3) Data Flow, (4) Tech Stack boundaries, (5) the MCP Connection Enhancement, and (6) the realized topology (Vercel-primary / AWS-fallback, the embedded knowgrph doc-view canvas, and the Dev → Prod → Cloudflare publish chain). They refine the existing acceptance criteria AC-1..AC-7 into EARS-format requirements and honor the project lenses (min-viable-max-value, tco-zero, token-economics, harness-first) and the seven judging dimensions (Agent Overview, Autonomy & Decision-Making, Actions & Tool Use, Orchestration, Human-in-the-Loop, Failure Handling, Demo & Presentation).

Requirement 15 covers caller authentication and authorization for the network-exposed Agent_Api endpoints (the Vercel `POST /api/auth/session`, `POST /api/run`, `GET /api/runs/{id}` routes and the AWS fallback `POST /auth/session`, `POST /run`, `GET /runs/{id}`, `GET /health` routes), which previously had no defined caller authentication. It establishes that spend-bearing and state endpoints reject unauthenticated requests, that run manifests are accessible only to entitled callers, that the liveness probe remains open without disclosing sensitive data, and that authentication credentials are never exposed to the client bundle or logs.

Requirements 16–19 capture the v0.3.1 topology re-baseline: Requirement 16 makes the Vercel-primary / AWS-fallback relationship and the browser fail-over behavior explicit and testable; Requirement 17 specifies that the product embeds the live knowgrph `doc-view` canvas under cross-origin framing and run-scoped entitlement guarantees; Requirement 18 captures the Dev → Prod → Cloudflare publish chain and `cloud-deploy` deploy gating; and Requirement 19 records that the AWS AgentCore wrapper is an additive deployable-agent lane, not a replacement for the Vercel-primary / AWS-fallback REST path.

## Glossary

- **Control_Plane**: The knowgrph deployment on Cloudflare (Workers + Pages) that hosts the MCP server, model routing, and the video/payment pipeline. Holds all model keys and is the only tier permitted to invoke paid models.
- **Mcp_Agent**: The MCP server built on the Cloudflare Agents SDK (`McpAgent`), exposing approval-gated, durable tools over MCP Streamable HTTP transport at `airvio.co/knowgrph/mcp`.
- **Director**: The durable orchestrator (`knowgrph.video_remix.run`, an Agents SDK `AgentWorkflow`) that sequences stages, enforces budget and approval gates, and applies bounded-retry failure handling.
- **Research_Harness**: The stage tool (`knowgrph.video_remix.research`) that retrieves and cites live web context via Exa.
- **Storyboard_Harness**: The stage tool (`knowgrph.video_remix.storyboard`) that converts an approved brief plus evidence into a KGC canvas shot-plan document.
- **Render_Harness**: The stage tool (`knowgrph.video_remix.render`) that dispatches per-shot generation through the existing Strytree/BytePlus/PixVerse queue and returns stored asset references.
- **Commerce_Harness**: The stage tools (`knowgrph.video_remix.publish` and `knowgrph.video_remix.checkout`) that publish an asset and create a gated Stripe checkout and payout.
- **Hitl_Gate_Service**: The service that issues, stores, and verifies per-gate approval tokens using the Agents SDK `needsApproval` pattern before any paid action.
- **Approval_Gate**: A named spend boundary requiring a verified approval token. The defined gate catalog has exactly six gate ids: `paid-model-call`, `render-action`, `payment-action`, `cloud-deploy`, `consumer-repo-write`, and `authenticated-browser`. The render stage uses its own `render-action` gate, distinct from `paid-model-call`.
- **Approval_Token**: A verifiable credential issued by the Hitl_Gate_Service that authorizes a single Approval_Gate.
- **Auth_Token**: A verifiable, time-bounded caller credential presented to the Agent_Api to authenticate a Caller_Identity, distinct from an Approval_Token. Issued by the configured identity provider and never embedded in the client bundle or written to logs.
- **Caller_Identity**: The authenticated principal associated with a request to the Agent_Api, established from a valid Auth_Token and used to authorize access to run manifests the principal is entitled to.
- **Ai_Gateway**: Cloudflare AI Gateway, the single model control plane providing response caching, token counting, fallback routing, rate limiting, and unified billing.
- **Agent_Api**: The agentic-canvas-os product Agent-API surface that validates submissions, forwards MCP calls to the Mcp_Agent, and exposes run-readback. It is realized by two tiers — the primary Vercel_Agent_Api and the fallback Aws_Agent_Api — that share one platform-neutral Agent-API core. Holds no model keys. Requirements that reference Agent_Api without naming a tier apply to both tiers.
- **Vercel_Agent_Api**: The primary/default Agent_Api tier, implemented as same-origin Vercel serverless routes `POST /api/auth/session`, `POST /api/run`, and `GET /api/runs/{id}`. It is the default browser product path. Holds no model keys.
- **Aws_Agent_Api**: The fallback Agent_Api tier, implemented on AWS API Gateway + Lambda + S3, exposing `POST /auth/session`, `POST /run`, `GET /runs/{id}`, and `GET /health`. The browser invokes it only when the Vercel_Agent_Api is unavailable or returns a 5xx response. Holds no model keys.
- **Agent_Core**: The additive AWS AgentCore wrapper lane that exposes a deployable-agent surface around the same control-plane forwarding seam. It is supplementary to, and not a replacement for, the Vercel_Agent_Api primary / Aws_Agent_Api fallback REST path.
- **Frontend**: The agentic-canvas-os Vercel application that collects intent, mints an Auth_Token via `POST /api/auth/session`, submits and re-submits runs through the Vercel_Agent_Api, renders approval prompts, surfaces the Run_Manifest and assets, and embeds the knowgrph Doc_View canvas.
- **Doc_View**: The knowgrph-hosted canvas route at `airvio.co/knowgrph/doc-view?run=<runId>[&doc=<docId>]` that renders a Kgc_Document live. knowgrph owns the canvas engine; the product does not reimplement the renderer.
- **Canvas_Embed**: The run-scoped embedding of the Doc_View route inside the Frontend via an iframe, established after MCP-backed orchestration. The Canvas_Embed crosses an origin boundary and is governed by framing and entitlement guarantees (frame-ancestors restricted to the Vercel origin; the embedded document scoped to the entitled `runId`).
- **Publish_Chain**: The one-directional artifact flow Dev → Prod → Cloudflare. Dev (`/Users/huijoohwee/Documents/GitHub/knowgrph`) is the source of truth; Prod (`huijoohwee/content/knowgrph`) is the artifact mirror; Cloudflare (`airvio.co`, `airvio.co/knowgrph`, `airvio.co/knowgrph/mcp`) is the live deployment. Deploys are operator-gated behind a `cloud-deploy` Approval_Token.
- **Run_Manifest**: The structured output of a Director run containing `state`, `stages[]`, `approvalGates[]`, `budgetMeters`, and `demoPack`.
- **Run_State**: The Director lifecycle state, one of `running`, `blocked`, `budget_exceeded`, `approval_required`, `verification_failed`, or `completed`.
- **Evidence_Pack**: The Research_Harness output `{ sources[], citations[], summary }` in which each source carries a `sourceId`.
- **Source_Card**: A cited research source identified by `sourceId`, referenced by downstream claims.
- **Kgc_Document**: A canvas document conforming to the `kgc-computing-flow/v1` frontmatter-flow schema, with `flow.nodes[]` and `flow.edges[]`.
- **Credit_Ledger**: The Cloudflare Durable Object (`StrytreeCreditLedgerActor`) that records render spend as ledger events.
- **Cost_Log**: The per-model-call record `{ model, prompt_tokens, completion_tokens, cache_hits, estimated_cost_usd }` emitted by the Ai_Gateway.
- **Budget_Meters**: The Director-tracked budget accounting including `estimatedCostUsd` and provider spend.
- **Demo_Pack**: The judging evidence aggregate containing `urls[]` and the seven judging-dimension sections. Each entry in `urls[]` carries a `kind` of `frontend`, `agent_api`, or `canvas`. A `canvas` entry references the embedded, runId-scoped Doc_View canvas and is verified only when that embedded runId-scoped canvas is reachable.
- **Dry_Run**: An execution mode (`mode:"dry-run"`) in which spend-bearing steps resolve to plan artifacts and perform zero paid actions.
- **Live_Mode**: An execution mode (`mode:"live"`) in which spend-bearing steps execute only after their Approval_Gate is approved.

## Requirements

### Requirement 1: Creator User Flow — Reference URL to Sellable Asset

**User Story:** As an end creator user, I want to paste a reference video URL and a creative brief, review the plan and budget, and approve spend, so that I receive a researched, storyboarded, remixed video I can buy without touching infrastructure.

#### Acceptance Criteria

1. WHEN an end creator user submits a reference video URL, a creative brief, and a budget cap through the Frontend, THE Frontend SHALL obtain an Auth_Token via `POST /api/auth/session` and forward the submission to the primary Vercel_Agent_Api `POST /api/run` endpoint within 2 seconds.
2. IF the submitted reference video URL is empty or not a syntactically valid HTTP/HTTPS URL, OR the creative brief is empty or exceeds 5,000 characters, OR the budget cap is not a positive number between 0.01 and 999,999.99, THEN THE Frontend SHALL reject the submission, display an error message identifying the invalid field, and SHALL NOT forward the submission to the Vercel_Agent_Api `POST /api/run` endpoint.
3. WHEN a run is initiated, THE Frontend SHALL display each planned stage and the budget cap before any Approval_Gate is approved.
4. WHEN the Research_Harness produces an Evidence_Pack, THE Frontend SHALL display every cited source contained in the Evidence_Pack to the end creator user.
5. WHEN the Storyboard_Harness produces a Kgc_Document, THE Frontend SHALL render the shot-plan with exactly one visual node per planned shot defined in the Kgc_Document.
6. WHEN the Director raises an Approval_Gate, THE Frontend SHALL render an approval prompt that displays the gate id and the estimated cost of the gated action.
7. WHEN a render completes and the `payment-action` Approval_Gate is approved, THE Frontend SHALL present the rendered asset and a Stripe checkout entry point to the end creator user.
8. IF the Vercel_Agent_Api returns an error response or no response within 30 seconds for the `POST /api/run` submission and the Aws_Agent_Api fallback also fails to accept the submission, THEN THE Frontend SHALL display an error message indicating the run could not be initiated and SHALL retain the end creator user's submitted inputs.
9. WHEN the Run_Manifest state changes at a stage transition, THE Frontend SHALL display the updated Run_Manifest state to the end creator user.

### Requirement 2: Solo Founder / Orchestrator Flow — One Gated Tool

**User Story:** As a solo founder and AI orchestrator, I want one MCP tool that runs the full research-to-sale pipeline in a live, approval-gated mode with observable spend, so that I can ship and demo an autonomous product without hand-wiring each stage.

#### Acceptance Criteria

1. THE Mcp_Agent SHALL expose `knowgrph.video_remix.run` as a single tool that accepts `{ referenceUrl, brief, budgetUsd, mode, approvals[] }` — where `brief` is 1 to 5,000 characters, `budgetUsd` is a decimal between 0.01 and 100,000.00 inclusive, `mode` is one of `"live"` or `"dry-run"`, and `approvals[]` contains 0 to 50 entries — and SHALL return a Run_Manifest within 5 seconds.
2. IF a `knowgrph.video_remix.run` call omits a required field, supplies an out-of-range `budgetUsd`, or supplies a `mode` value other than `"live"` or `"dry-run"`, THEN THE Mcp_Agent SHALL reject the call, return an error identifying the invalid field, perform zero paid-provider calls, and create no Run_Manifest.
3. WHEN the solo founder calls `knowgrph.video_remix.run` in Live_Mode with an empty `approvals[]` array, THE Director SHALL halt at the first spend boundary, return Run_State `blocked`, emit at least 5 Approval_Gate entries, set `budgetMeters.estimatedCostUsd` to exactly 0, and record exactly 0 paid-provider calls. (Refines AC-1)
4. THE Director SHALL record exactly one Cost_Log entry per model-bearing stage in the Run_Manifest, each entry containing the stage id, the estimated cost in USD, and the actual cost in USD.
5. WHILE Run_State is in-progress, THE Director SHALL update Budget_Meters within 2 seconds of each spend event to reflect cumulative estimated and actual spend.
6. WHERE the caller sets `mode:"dry-run"`, THE Director SHALL resolve every spend-bearing step to a plan artifact, perform exactly zero paid-provider calls, and report `budgetMeters.actualCostUsd` as exactly 0.

### Requirement 3: Hackathon Judge Flow — Evidence Pack

**User Story:** As a hackathon judge, I want an evidence pack mapped to the seven judging dimensions, so that I can verify agent autonomy, tool use, orchestration, human-in-the-loop, failure handling, and demo clarity.

#### Acceptance Criteria

1. WHEN a Director run reaches a terminal Run_State, THE Director SHALL assemble a Demo_Pack containing exactly seven evidence sections, one for each of the seven judging dimensions, where each section is non-empty.
2. THE Demo_Pack SHALL include a `urls[]` collection in which each entry carries a `kind` of `frontend`, `agent_api`, or `canvas`, containing at least one `frontend` URL and at least one `agent_api` endpoint, where each listed `frontend` and `agent_api` URL returns HTTP 200 within 5 seconds of request. (Refines AC-7)
3. IF any `frontend` or `agent_api` URL in the Demo_Pack `urls[]` collection does not return HTTP 200 within 5 seconds, THEN THE Director SHALL mark the corresponding evidence section as unverified and record the failing URL. (Refines AC-7)
4. WHEN the product is deployed and the deploy Approval_Gates are approved, THE Aws_Agent_Api SHALL return HTTP 200 on its `GET /health` route within 5 seconds. (Refines AC-7)
5. IF the Aws_Agent_Api `GET /health` route does not return HTTP 200 within 5 seconds after the deploy Approval_Gates are approved, THEN THE Director SHALL retry the request up to 3 times and, if all retries fail, record a health-check failure indication in the Demo_Pack. (Refines AC-7)
6. WHILE assembling the Demo_Pack, THE Director SHALL reference the Evidence_Pack citations, the rendered asset reference, and the Stripe session identifier for each artifact that exists.
7. IF the Evidence_Pack citations, the rendered asset reference, or the Stripe session identifier do not exist at assembly time, THEN THE Director SHALL mark each missing artifact reference as not available in the Demo_Pack.
8. WHERE the Demo_Pack `urls[]` collection includes a `canvas` entry, THE Director SHALL mark that entry verified only when the embedded runId-scoped Doc_View canvas for the run is reachable, and SHALL otherwise mark the entry unverified and record the unreachable canvas URL. (Refines AC-7)

### Requirement 4: Work Flow — Director Run Sequencing and HITL Gates

**User Story:** As a solo founder, I want the director to sequence research, storyboard, render, publish, and checkout with human-approval gates at every spend boundary, so that no paid action occurs without explicit approval.

#### Acceptance Criteria

1. WHEN a Director run starts in Live_Mode, THE Director SHALL execute stages strictly in the order research, storyboard, render, publish, checkout, beginning each stage only after the preceding stage has reached a completed state.
2. THE Director SHALL require a verified, unexpired Approval_Token for the `render-action` Approval_Gate before executing the render stage, and SHALL NOT execute the render stage otherwise.
3. THE Director SHALL require a verified, unexpired `payment-action` Approval_Gate token before invoking checkout or payout, and SHALL NOT invoke checkout or payout otherwise.
4. IF a spend-bearing stage is reached without a verified Approval_Token, THEN THE Director SHALL set that stage to `approval_required` and resolve the stage to a Dry_Run plan artifact.
5. WHEN the Research_Harness returns fewer than 3 sources, THE Director SHALL mark the research stage `weak_signal` and halt before the storyboard stage, remaining halted until a verified Approval_Token authorizes continuation. (Refines AC-2 alternate path)
6. WHEN Budget_Meters reach or exceed the configured budget cap mid-run, THE Director SHALL record `budget_exceeded`, halt all further spend-bearing stages, and surface a budget-exceeded indication to the operator.
7. THE Hitl_Gate_Service SHALL verify an Approval_Token within the same operation that precedes each paid action, with no intervening spend-bearing operation, and SHALL treat an Approval_Token as valid only within 15 minutes of its issuance.
8. IF the Approval_Token presented before a paid action is absent, invalid, or expired, THEN THE Director SHALL block the paid action, record the rejection reason in the Run_Manifest, preserve the prior Run_State, and perform zero paid-provider calls.

### Requirement 5: Work Flow — Bounded Failure Handling

**User Story:** As a solo founder, I want failures to retry within a bounded limit and otherwise fail closed with evidence, so that the agent never runs away and the demo degrades gracefully.

#### Acceptance Criteria

1. WHEN a single stage tool fails, THE Director SHALL retry the stage using exponential backoff starting at 1 second and capped at 30 seconds per attempt, and SHALL increment the stage retry count by exactly 1 on each retry. (Refines AC-6)
2. THE Director SHALL limit total stage iterations to `maxIterations`, where `maxIterations` is a positive integer between 1 and 100 inclusive.
3. WHILE the stage retry count is less than `maxIterations`, THE Director SHALL keep Run_State set to `running` and continue retrying the failed stage. (Refines AC-6)
4. IF a stage continues to fail after the stage retry count reaches `maxIterations`, THEN THE Director SHALL set Run_State to `blocked`, halt all further stage iterations, and append a failure record to the Run_Manifest containing the stage identifier, final retry count, and failure reason. (Refines AC-6)
5. IF the Ai_Gateway reports that all providers are unavailable, THEN THE affected harness SHALL return a structured degraded error identifying the unavailable providers, and THE Director SHALL set Run_State to `blocked` without consuming additional retries.
6. IF a Stripe checkout webhook does not match a verified session in the Commerce_Harness, THEN THE Commerce_Harness SHALL withhold the payout, leave the payout amount unchanged, and append a reconciliation flag to the Run_Manifest identifying the affected run.

### Requirement 6: Data Flow — Research and Source Attribution

**User Story:** As an end creator user, I want research backed by cited live sources, so that downstream suggestions are grounded in evidence rather than fabricated.

#### Acceptance Criteria

1. WHEN the research stage runs, THE Research_Harness SHALL query Exa through the Ai_Gateway and, within 30 seconds, produce an Evidence_Pack containing at least 3 and at most 50 Source_Cards. (Refines AC-2)
2. WHEN the Research_Harness creates an Evidence_Pack, THE Research_Harness SHALL assign each Source_Card a `sourceId` that is unique within that Evidence_Pack.
3. THE Storyboard_Harness SHALL reference at least one `sourceId`, each resolving to a Source_Card present in the associated Evidence_Pack, for every downstream claim derived from research. (Refines AC-2)
4. IF Exa returns an error or the query does not complete within 30 seconds, THEN THE Research_Harness SHALL return a degraded summary with an empty source list, mark the stage `weak_signal`, retain any partial input data without modification, and SHALL NOT fabricate sources.
5. IF Exa returns fewer than 3 Source_Cards, THEN THE Research_Harness SHALL mark the stage `weak_signal` and SHALL NOT fabricate additional sources to reach the minimum count. (Refines AC-2)
6. IF the Storyboard_Harness produces a downstream claim that references a `sourceId` not present in the associated Evidence_Pack, THEN THE Storyboard_Harness SHALL reject the claim and return an error indicating an unresolved source reference.

### Requirement 7: Data Flow — Storyboard Materialization on Canvas

**User Story:** As an end creator user, I want the plan rendered as a shot-plan on the canvas, so that I can see the structure before approving render spend.

#### Acceptance Criteria

1. WHEN the storyboard stage runs with an approved brief, THE Storyboard_Harness SHALL emit a Kgc_Document that successfully validates against the `kgc-computing-flow/v1` schema, with a non-empty `flow.nodes[]` array. (Refines AC-3)
2. WHEN the storyboard stage emits a Kgc_Document for a plan of N planned shots (where 1 ≤ N ≤ 500), THE Storyboard_Harness SHALL produce exactly one `flow.nodes[]` entry per planned shot, such that the count of `flow.nodes[]` entries equals N. (Refines AC-3)
3. FOR ALL emitted Kgc_Documents, THE Storyboard_Harness SHALL guarantee that parsing the document, then serializing the parsed result, then parsing it again produces an equivalent flow structure, where equivalence means identical node count, identical set of node identifiers, identical node ordering, and identical edge connections between nodes (round-trip property).
4. IF the storyboard stage produces a Kgc_Document that fails validation against the `kgc-computing-flow/v1` schema, THEN THE Storyboard_Harness SHALL reject the document, return an error indication identifying the validation failure, and emit no `flow.nodes[]` entries to the canvas.
5. IF storyboard reasoning fails, THEN THE Storyboard_Harness SHALL emit a fallback Kgc_Document containing exactly one `flow.nodes[]` entry that validates against the `kgc-computing-flow/v1` schema and satisfies the round-trip property of criterion 3, and SHALL return an indication that fallback content was substituted.

### Requirement 8: Data Flow — Render, Asset Storage, and Credit Ledger

**User Story:** As a solo founder, I want render to reuse the existing pipeline and record every spend in the credit ledger, so that asset costs stay observable and bounded.

#### Acceptance Criteria

1. WHEN the render stage runs with a valid, unexpired render Approval_Token, THE Render_Harness SHALL dispatch generation through the existing Strytree/BytePlus queue within 5 seconds of stage invocation. (Refines AC-4)
2. IF the render Approval_Token is missing, expired, or fails validation, THEN THE Render_Harness SHALL reject the render request, perform no provider dispatch, record zero provider spend, and return an error indication identifying the Approval_Token failure. (Refines AC-4)
3. WHEN a shot render completes successfully, THE Render_Harness SHALL return exactly one asset reference resolvable under the knowgrph media bucket and exactly one Credit_Ledger event identifier for that shot. (Refines AC-4)
4. WHEN a shot render completes, THE Render_Harness SHALL record one Credit_Ledger event capturing the provider spend amount and the provider identity for that shot before returning the asset reference.
5. IF no provider key is available OR the cumulative recorded provider spend for the current run meets or exceeds the configured budget cap, THEN THE Render_Harness SHALL route the shot to the deterministic mock provider and record a Credit_Ledger event with provider spend equal to zero.
6. IF the provider dispatch fails or returns no asset within 120 seconds of dispatch, THEN THE Render_Harness SHALL return an error indication identifying the failed shot, record a Credit_Ledger event reflecting the actual provider spend incurred, and leave previously rendered shot assets unchanged.

### Requirement 9: Data Flow — Gated Sale and Payout

**User Story:** As a solo founder, I want checkout and payout to occur only after explicit payment approval, so that funds move only with human consent.

#### Acceptance Criteria

1. WHEN the checkout stage runs and the `payment-action` Approval_Gate state is `approved`, THE Commerce_Harness SHALL create a Stripe checkout session and return a non-empty session identifier within 10 seconds. (Refines AC-5)
2. WHEN the `payment-action` Approval_Gate state is `approved` and a Stripe checkout session exists, THE Commerce_Harness SHALL settle the payout and record a settlement confirmation observable to the caller. (Refines AC-5)
3. IF the `payment-action` Approval_Gate state is any value other than `approved`, THEN THE Commerce_Harness SHALL NOT create a Stripe checkout session and SHALL NOT settle the payout, and SHALL leave the payout in its pre-checkout state. (Refines AC-5)
4. IF creation of the Stripe checkout session or settlement of the payout fails after the `payment-action` Approval_Gate state is `approved`, THEN THE Commerce_Harness SHALL NOT settle the payout, SHALL return an error indication identifying the failed operation, and SHALL preserve the payout in its pre-checkout state. (Refines AC-5)

### Requirement 10: Data Flow — Cost Logging and Token Economics

**User Story:** As a solo founder, I want every model call logged with token counts and cost, so that orchestration spend stays within the token-economics budget.

#### Acceptance Criteria

1. WHEN a model-bearing stage completes a model call, THE Ai_Gateway SHALL emit a Cost_Log within 1 second containing `model` (non-empty string), `prompt_tokens` (integer ≥ 0), `completion_tokens` (integer ≥ 0), `cache_hits` (integer ≥ 0), and `estimated_cost_usd` (decimal ≥ 0.00).
2. IF a model call completes but one or more token counts are unavailable from the provider, THEN THE Ai_Gateway SHALL emit a Cost_Log with each affected token field set to an explicit unknown indicator, mark the entry as incomplete, and retain the entry without discarding it.
3. WHEN a Cost_Log is emitted, THE Director SHALL aggregate the entry into the Run_Manifest Budget_Meters within 1 second of emission.
4. THE Credit_Ledger SHALL remain consistent such that the sum of recorded ledger events equals the total provider spend reported in Budget_Meters within a tolerance of ±0.01 USD.
5. IF the sum of recorded Credit_Ledger events deviates from the total provider spend reported in Budget_Meters by more than ±0.01 USD, THEN THE Director SHALL flag a reconciliation discrepancy and preserve both the Credit_Ledger events and Budget_Meters values without modification.

### Requirement 11: Tech Stack — Per-Repo Boundaries and Spend Isolation

**User Story:** As a solo founder, I want the product tier to never hold model keys or call paid models directly, so that all spend routes through one observable, gated control plane.

#### Acceptance Criteria

1. THE Agent_Api SHALL forward all reasoning and spend-bearing actions to the Mcp_Agent, and neither the Vercel_Agent_Api nor the Aws_Agent_Api SHALL store, reference, or transmit model provider keys in any request, response, configuration, or environment value.
2. Neither the Vercel_Agent_Api nor the Aws_Agent_Api SHALL issue requests to paid model provider endpoints, and any reasoning or spend-bearing action received by the Agent_Api SHALL result in forwarding to the Mcp_Agent rather than direct provider invocation.
3. WHERE the Frontend performs a model call, THE Frontend SHALL route the call through the Ai_Gateway, and THE Frontend SHALL NOT invoke any paid model provider endpoint directly.
4. THE Control_Plane SHALL route all model calls originating from both the control plane and the product through the Ai_Gateway, such that no model call bypasses the Ai_Gateway.
5. THE Control_Plane SHALL be the only tier that holds model provider keys and invokes paid models, and no other tier (Vercel_Agent_Api, Aws_Agent_Api, Agent_Core, Mcp_Agent forwarding clients, Frontend) SHALL hold model provider keys.
6. WHEN any paid action is requested in Live_Mode, THE Hitl_Gate_Service SHALL require an Approval_Token that is unexpired, matches the requested paid action, and has not been previously consumed, before permitting execution.
7. IF a paid action is requested in Live_Mode with an Approval_Token that is missing, expired, already consumed, or does not match the requested action, THEN THE Hitl_Gate_Service SHALL block execution, leave all spend-bearing state unchanged, and return an error indication identifying the failed approval check to the caller.
8. WHEN the Hitl_Gate_Service permits execution against a valid Approval_Token, THE Hitl_Gate_Service SHALL mark that Approval_Token as consumed so it cannot authorize a second paid action.

### Requirement 12: MCP Connection Enhancement — Agent-API Forwarding

**User Story:** As a solo founder, I want the product Agent-API — both the primary Vercel tier and the fallback AWS tier — to forward MCP Streamable HTTP calls to the control plane through one shared core, so that agentic-canvas-os can consume knowgrph capabilities through a thin keyless adapter on either tier.

#### Acceptance Criteria

1. WHEN the Agent_Api receives a run submission (`POST /api/run` on the Vercel_Agent_Api or `POST /run` on the Aws_Agent_Api), THE Agent_Api SHALL validate the request against the schema `{ referenceUrl, brief, budgetUsd, approvals[] }`, where `referenceUrl` is a non-empty absolute URL of at most 2,048 characters, `brief` is a non-empty string of 1 to 10,000 characters, `budgetUsd` is a decimal between 0.01 and 999,999,999.99 inclusive, and `approvals[]` contains 0 to 100 entries.
2. WHEN a run submission passes schema validation, THE Agent_Api SHALL forward a `knowgrph.video_remix.run` MCP call to the Mcp_Agent over MCP Streamable HTTP transport within 2,000 milliseconds of validation completion.
3. IF a run submission fails schema validation, THEN THE Agent_Api SHALL return an HTTP 4xx response that names each invalid field and the reason it failed, and SHALL NOT forward any MCP call to the Mcp_Agent.
4. IF the Agent_Api backend is saturated, defined as the count of in-flight forwarded MCP calls reaching the configured maximum concurrency limit, THEN THE Agent_Api SHALL return an HTTP 503 response with a `retry-after` value between 1 and 120 seconds.
5. WHEN the Agent_Api receives a run-readback request (`GET /api/runs/{id}` on the Vercel_Agent_Api or `GET /runs/{id}` on the Aws_Agent_Api) for a run whose identifier exists, THE Agent_Api SHALL return the current Run_Manifest for that run within 1,000 milliseconds.
6. IF the Agent_Api receives a run-readback request for a run whose identifier does not exist, THEN THE Agent_Api SHALL return an HTTP 404 response indicating the run was not found.
7. IF the Mcp_Agent returns a typed MCP error, THEN THE Agent_Api SHALL map the error either to a gate prompt or to a failure record recorded in the Run_Manifest, and SHALL preserve the existing Run_Manifest state for that run.

### Requirement 13: MCP Connection Enhancement — Frontend Approval and Manifest Rendering

**User Story:** As an end creator user, I want the Vercel frontend to render approval prompts and run manifests, so that I can approve spend and follow run progress in the product UI.

#### Acceptance Criteria

1. WHEN the Run_Manifest contains one or more pending Approval_Gate entries, THE Frontend SHALL render an approval prompt for each pending Approval_Gate within 2 seconds of receiving the Run_Manifest, displaying the gate identifier and the associated spend amount.
2. WHEN the end creator user approves a gate in the Frontend, THE Frontend SHALL re-submit the run with the updated `approvals[]` to the primary Vercel_Agent_Api `POST /api/run` endpoint for forwarding to the Hitl_Gate_Service within 2 seconds of the approval action.
3. IF transmission of an approval re-submission to the Vercel_Agent_Api fails or does not receive a success response within 10 seconds, THEN THE Frontend SHALL retain the pending approval prompt, display an error indication that the approval was not submitted, and allow the end creator user to retry up to 3 times.
4. WHEN the Frontend receives an updated Run_Manifest, THE Frontend SHALL render the current Run_State, the complete stage list, and the Budget_Meters within 2 seconds of receipt.
5. IF the primary Vercel_Agent_Api returns HTTP 503, THEN THE Frontend SHALL fail over to the Aws_Agent_Api in accordance with Requirement 16 and poll run-readback at 5-second intervals for a maximum of 12 attempts, and SHALL resume normal primary Vercel_Agent_Api operation when a non-503 response is received from the primary tier.

### Requirement 14: MCP Connection Enhancement — Durable, Stateful Tool Surface

**User Story:** As a solo founder, I want the control plane MCP server to expose live, durable, stateful, approval-gated tools to remote clients, so that the product can drive long-running runs reliably.

#### Acceptance Criteria

1. THE Mcp_Agent SHALL expose the Director tool and its stage tools to remote clients over MCP Streamable HTTP transport at the control plane MCP endpoint.
2. WHEN a Director run's state changes, THE Mcp_Agent SHALL write the updated Run_Manifest state to durable storage within 2 seconds of the change, such that a subsequent `GET /runs/{id}` query for that run returns the latest persisted state.
3. IF writing the updated Run_Manifest state to durable storage fails, THEN THE Mcp_Agent SHALL retain the most recently persisted state, return a response indicating the persistence failure, and emit an observability diagnostic indicating the persistence failure.
4. WHEN a remote client requests the tool surface, THE Mcp_Agent SHALL return a list that includes `knowgrph.video_remix.run` and each of its stage tools, with each listed tool including its input schema and its output schema.
5. WHEN a Director run transitions from one stage to another, THE Mcp_Agent SHALL emit an observability diagnostic record containing the run identifier, the originating stage identifier, the destination stage identifier, a UTC timestamp, and the transition outcome status.
6. WHERE a stage tool is configured as approval-gated, IF a remote client invokes that tool before approval for the invocation has been granted, THEN THE Mcp_Agent SHALL withhold execution of the tool, leave the Run_Manifest state unchanged, and return a response indicating that approval is required.

### Requirement 15: Agent-API Endpoint Authentication and Authorization

**User Story:** As a solo founder, I want the network-exposed Agent_Api endpoints to authenticate callers and authorize run-manifest access, so that unauthenticated or unentitled callers cannot trigger spend, read another tenant's run data, or extract credentials.

#### Acceptance Criteria

1. WHEN the Agent_Api receives a request to a spend-bearing or state endpoint (the Vercel_Agent_Api `POST /api/run` or `GET /api/runs/{id}`, or the Aws_Agent_Api `POST /run` or `GET /runs/{id}`) without a valid, unexpired Auth_Token, THE Agent_Api SHALL reject the request with HTTP 401, perform no MCP forwarding, and disclose no Run_Manifest data.
2. WHEN the Agent_Api receives a request to a spend-bearing or state endpoint carrying a valid, unexpired Auth_Token, THE Agent_Api SHALL establish the Caller_Identity from the Auth_Token before processing the request.
3. IF the Auth_Token presented to a spend-bearing or state endpoint is malformed, has an invalid signature, or is expired, THEN THE Agent_Api SHALL reject the request with HTTP 401, perform no MCP forwarding, and return an error indication that does not reveal credential contents or internal configuration.
4. WHEN the Agent_Api receives a `GET /runs/{id}` request from an authenticated Caller_Identity that is entitled to that run, THE Agent_Api SHALL return the Run_Manifest for that run.
5. IF an authenticated Caller_Identity requests `GET /runs/{id}` for a run the Caller_Identity is not entitled to, THEN THE Agent_Api SHALL deny access with HTTP 403 or HTTP 404, return no Run_Manifest content, and record the denied access attempt.
6. THE Aws_Agent_Api SHALL serve `GET /health` without requiring an Auth_Token, and THE Aws_Agent_Api SHALL restrict the `GET /health` response to liveness status, disclosing no Run_Manifest data, credentials, or internal configuration values.
7. THE Agent_Api SHALL keep Auth_Token credentials and any authentication secrets server-side only, such that no authentication secret is included in the Frontend client bundle, written to logs, or returned in any Agent_Api response, consistent with the Tech Stack boundary that the product tier holds no model provider keys.
8. WHERE the deployment configures an Auth_Token expiry window, THE Agent_Api SHALL treat an Auth_Token as expired once its issuance age exceeds the configured window, and the configured window SHALL be between 5 minutes and 24 hours inclusive, defaulting to 60 minutes when unset.
9. IF an authenticated request to a spend-bearing endpoint is accepted, THEN THE Agent_Api SHALL continue to enforce all existing Approval_Gate checks, such that authentication SHALL NOT substitute for an Approval_Token at any spend boundary.

### Requirement 16: Topology — Vercel-Primary / AWS-Fallback Agent-API and Browser Fail-Over

**User Story:** As a solo founder, I want the Vercel-hosted Agent-API to be the default product path and the AWS Agent-API to be the fallback, so that the browser uses one keyless primary path and degrades to a proven fallback only when the primary fails.

#### Acceptance Criteria

1. WHEN the Frontend initiates a session or a run, THE Frontend SHALL direct the request to the primary Vercel_Agent_Api same-origin routes (`POST /api/auth/session`, `POST /api/run`, `GET /api/runs/{id}`) before any use of the Aws_Agent_Api.
2. IF a primary Vercel_Agent_Api request returns an HTTP 5xx response, or fails to return any response within 30 seconds (transport error), THEN THE Frontend SHALL retry the equivalent request against the Aws_Agent_Api fallback routes (`POST /auth/session`, `POST /run`, `GET /runs/{id}`) exactly once.
3. THE Frontend SHALL NOT invoke the Aws_Agent_Api for a request that the primary Vercel_Agent_Api has answered with any HTTP response other than 5xx, including any 2xx, 3xx, or 4xx response.
4. WHEN a primary Vercel_Agent_Api request succeeds after one or more prior fail-overs, THE Frontend SHALL resume routing subsequent requests to the primary Vercel_Agent_Api, and SHALL NOT retain a sticky fallback state that bypasses the primary-first routing of criterion 1.
5. WHEN forwarding a run, both the Vercel_Agent_Api and the Aws_Agent_Api SHALL forward the same `knowgrph.video_remix.run` call to the Mcp_Agent at `airvio.co/knowgrph/mcp` over MCP Streamable HTTP, and neither tier SHALL hold model provider keys.
6. WHILE a run is being processed through the Aws_Agent_Api fallback, THE fallback tier SHALL enforce the same Auth_Token authentication, Caller_Identity entitlement, and Approval_Gate checks that apply to the primary Vercel_Agent_Api.
7. IF the Aws_Agent_Api fallback retry also returns an HTTP 5xx response or fails to return any response within 30 seconds (transport error), THEN THE Frontend SHALL perform no further fail-over for that request, display an error indication that the request could not be completed, and retain the caller's submitted inputs unchanged.

### Requirement 17: Topology — Embedded knowgrph Doc-View Canvas

**User Story:** As an end creator user, I want the product to embed the live knowgrph canvas after orchestration rather than reimplement it, so that I view the authoritative shot-plan while the product remains a thin OS shell.

#### Acceptance Criteria

1. WHEN a Run_Manifest whose storyboard Kgc_Document validates against the `kgc-computing-flow/v1` schema is available, THE Frontend SHALL display the canvas by embedding the knowgrph Doc_View route `airvio.co/knowgrph/doc-view?run=<runId>` in a Canvas_Embed iframe within 2 seconds, and SHALL NOT render the Kgc_Document through any alternative or reimplemented canvas renderer.
2. THE Doc_View route SHALL restrict framing through a `frame-ancestors` policy that permits embedding only from the Vercel Frontend origin, and SHALL refuse framing from any other origin.
3. THE Doc_View route SHALL scope the embedded document to the authenticated, entitled `runId`, enforcing the same entitlement boundary that governs the run-readback path in Requirement 15.
4. IF the requesting Caller_Identity is not entitled to the requested `runId`, THEN THE Doc_View route SHALL deny access and SHALL render no Kgc_Document content for that `runId`.
5. WHERE the Frontend supplies the optional `&doc=<docId>` parameter on the Doc_View route, THE Doc_View route SHALL scope the embedded document to that `docId` under the same `runId` entitlement boundary established in criterion 3, and SHALL render no content when the Caller_Identity is not entitled to that `runId`.
6. IF the Canvas_Embed does not load embeddable content within 10 seconds, or framing is refused, or the Doc_View route returns a non-success response, THEN THE Frontend SHALL within 2 seconds fall back to a manifest-only view, indicate that the embedded canvas is unavailable, and preserve the displayed Run_Manifest state unchanged.
7. THE Frontend SHALL treat the Canvas_Embed as reachable only when the Doc_View route returns embeddable content within 10 seconds and framing from the Vercel Frontend origin is permitted, and THE Director SHALL use this reachability determination to verify the Demo_Pack `canvas` url-kind entry per Requirement 3 criterion 8.
8. WHEN the embedded Kgc_Document defines N planned shots (where 1 ≤ N ≤ 500), THE Canvas_Embed SHALL render exactly one visual node per planned shot, such that the count of rendered visual nodes equals the count of `flow.nodes[]` entries, consistent with the storyboard materialization in Requirement 7.

### Requirement 18: Topology — Dev → Prod → Cloudflare Publish Chain and Deploy Gating

**User Story:** As a solo founder, I want knowgrph Dev to be the source of truth and Prod and Cloudflare to receive only synced artifacts behind an operator-gated deploy, so that the connector is never patched in a downstream mirror and no deploy occurs without explicit approval.

#### Acceptance Criteria

1. THE Publish_Chain SHALL treat the Dev knowgrph repository (`/Users/huijoohwee/Documents/GitHub/knowgrph`) as the single source of truth for all connector contracts, implementation, and documentation.
2. THE Prod mirror (`huijoohwee/content/knowgrph`) and the Cloudflare deployment (`airvio.co`, `airvio.co/knowgrph`, `airvio.co/knowgrph/mcp`) SHALL receive only artifacts generated or synced from the Dev source by the Dev → Prod → Cloudflare Publish_Chain.
3. THE Publish_Chain SHALL NOT permit any connector contract, route-specific fix, or other connector change to be authored directly in the Prod mirror or in the Cloudflare artifact, and SHALL require every such change to originate in the Dev source.
4. WHEN a deploy to Cloudflare is initiated, THE Publish_Chain SHALL require a `cloud-deploy` Approval_Token that is verified, unexpired (issued within the 15-minute Approval_Token validity window defined in Requirement 4), and not previously consumed, before the deploy proceeds, and SHALL mark that Approval_Token consumed once the deploy is permitted so it cannot authorize a second deploy.
5. IF a deploy to Cloudflare is attempted with a `cloud-deploy` Approval_Token that is absent, invalid, expired, or already consumed, THEN THE Publish_Chain SHALL block the deploy, leave the live Cloudflare deployment (`airvio.co`, `airvio.co/knowgrph`, `airvio.co/knowgrph/mcp`) unchanged, and return an error indication identifying the failed approval check.
6. WHEN the Publish_Chain syncs artifacts from Dev to Prod, THE Publish_Chain SHALL preserve the Dev contract such that the Mcp_Agent endpoint exposed to the product remains `airvio.co/knowgrph/mcp`.
7. WHEN the Publish_Chain completes a Dev → Prod sync, THE Publish_Chain SHALL verify that every synced Prod artifact corresponds to the output of that sync from the Dev source, and SHALL record a sync-verification result identifying any Prod artifact that does not correspond.
8. IF the Publish_Chain detects drift, defined as a Prod mirror or Cloudflare artifact that diverges from the current Dev → Prod sync output without a corresponding sync from the Dev source, THEN THE Publish_Chain SHALL flag the divergent artifact, block any Cloudflare deploy, and leave the live Cloudflare deployment unchanged until the divergence is reconciled from the Dev source.

### Requirement 19: Topology — Additive AgentCore Lane

**User Story:** As a solo founder, I want the AWS AgentCore wrapper to be an additive deployable-agent surface, so that adopting it does not change the Vercel-primary / AWS-fallback product path.

#### Acceptance Criteria

1. WHEN the Agent_Core lane receives a run submission, THE Agent_Core lane SHALL forward the same `knowgrph.video_remix.run` call to the Mcp_Agent at `airvio.co/knowgrph/mcp` over MCP Streamable HTTP within 2,000 milliseconds of validation completion.
2. WHERE the Agent_Core lane is deployed, THE Vercel_Agent_Api SHALL remain the primary product path and the Aws_Agent_Api SHALL remain the fallback product path, such that enabling the Agent_Core lane does not change the browser primary-first routing and single fail-over defined in Requirement 16.
3. WHERE the Agent_Core lane is absent or not deployed, THE Vercel_Agent_Api primary path and the Aws_Agent_Api fallback path SHALL provide the full product capability without dependence on the Agent_Core lane.
4. THE Agent_Core lane SHALL hold no model provider keys and SHALL NOT issue requests to paid model provider endpoints directly, consistent with the Tech Stack boundary in Requirement 11.
5. WHEN any paid action is requested through the Agent_Core lane in Live_Mode, THE Agent_Core lane SHALL require an Approval_Token that is unexpired, matches the requested paid action, and has not been previously consumed, before the action proceeds, and SHALL otherwise block the action and leave all spend-bearing state unchanged.
6. THE Agent_Core lane SHALL authenticate inbound callers with an Auth_Token at its exposed surface, applying the same authentication boundary that governs the Agent_Api in Requirement 15, such that authentication never substitutes for an Approval_Token at any spend boundary.
