# Requirements Document

## Introduction

`knowgrph` (canonical repo `huijoohwee/knowgrph`) runs the `knowgrph.video_remix.run` Director — a native-in-repo, approval-gated Agentic_Loop harness (the `knowgrph-acos-mcp-connector` spec) that now sequences **research → storyboard → render → edit → publish → checkout** under a bounded-retry policy (max 8 iterations, exponential backoff 1s→30s, circuit-breaker on `{blocked, budget_exceeded, approval_required, verification_failed}`). BytePlus ModelArk is an integrated model provider in this monorepo: the Cloudflare AI Gateway client (`mcp/video-remix/ai-gateway-client.js`) exposes `chat` (`agnes/seed`), `image` (`seedream`), and an async `submitVideo`/`pollVideoUntilDone` pair (`seedance`) against BytePlus, and the wider frontend proxy stack (`huijoohwee/functions/api/_integrationHub.js`, `huijoohwee/functions/__chat_proxy/[[path]].js`, provider id `byteplus-modelark`) routes chat traffic to `ark.ap-southeast.bytepluses.com` / `ark.eu-west.bytepluses.com`.

This specification defines the **Video_Agent**: an **enhancement of the existing Video_Remix Director**, not a rebuild. It strengthens the Director's storyboard stage for demonstrable narrative depth, routes the Director's render stage through a live BytePlus video-generation client when configuration and approval are present, and adds exactly one **Editing_Stage** between render and publish so the pipeline demonstrably handles scriptwriting → storyboarding → video generation → editing end to end. Every requirement below is evaluated through the four compounding lenses required by the PRD & TAD Guidelines (`guidelines/prd-tad-guidelines.md`): **min-viable-max-value**, **TCO-zero**, **token economics**, and **harness-first** — and defaults to reusing the existing Director, the existing BytePlus/AI-Gateway integration path, the existing `contracts/approval.schema.js` / `contracts/cost-log.schema.js` shared contracts, and the existing Cost_Ledger_Aggregator and Approval_Gate machinery over introducing any new provider integration, spend-tracking mechanism, or approval mechanism.

Two goals are first-class and independently demonstrable in this increment: (1) **narrative ability** — the storyboard stage produces a coherent, evidence-grounded shot sequence with an observable, non-repetitive narrative arc, and each shot's prompt is traceably handed off unmodified to the video-generation stage (the **multimodal orchestration** handoff: text plan → video artifact); and (2) **output quality under a bounded token budget** — the Video_Agent SHALL operate under a configured per-run token ceiling for its narrative-reasoning (chat) calls and SHALL degrade gracefully (never silently exceed the ceiling) when that budget is tight.

This document is the source requirement artifact for the runtime-ready Video_Agent implementation. The combined PRD/TAD documents this spec informs (`knowgrph/docs/documents/knowgrph-agentic-os-prd-tad.md` and its companion `knowgrph/docs/documents/knowgrph-agentic-os-video-agent-prd-tad.companion.md`) are kept aligned with this source spec.

### Grounding: what already exists vs. what this spec adds

| Capability area | Already exists in `knowgrph` (reuse, do not rebuild) | What this spec adds (net-new Video_Agent glue) |
|---|---|---|
| Pipeline orchestration | `runVideoRemix` (`mcp/video-remix/run-video-remix.js`) sequences `research → storyboard → render → edit → publish → checkout`; bounded retry (max 8 iterations, 1s→30s backoff, `mcp/video-remix/retry.js`); circuit-breaker on `{blocked, budget_exceeded, approval_required, verification_failed}` | The Director's existing stage-sequencing, retry, and circuit-breaker logic is extended, not replaced |
| Narrative / storyboard reasoning | `runStoryboardHarness` (`mcp/video-remix/storyboard-harness.js`) — evidence-grounded shot planning, schema-valid `kgc-computing-flow/v1` `Kgc_Document` emission, single-node reasoning-failure fallback (R7.5), source-referential-integrity gate | A narrative-coherence check on the emitted shot sequence (no two consecutive shots share an identical prompt) and a per-run token-budget ceiling with a defined degraded-mode response, both layered on the existing harness output |
| BytePlus video generation | `createAiGatewayClient().submitVideo`/`pollVideoUntilDone` (`mcp/video-remix/ai-gateway-client.js`, model `seedance`, 5s poll / 600s max); `createBytePlusVideoProvider` (`mcp/video-remix/render-providers.js`) wires that client + a media persister into the `dispatch({runId,stageId,shotId,prompt,model})` shape `runRenderHarnessAsync` expects | `resolveStageClients()` (`mcp/video-remix/live-clients.js`) routes the default live render-client path through `createBytePlusVideoProvider`; `createStrytreeRenderQueueClient` remains reachable only through the explicit provider override |
| Approval / spend gating | Six canonical `Approval_Gate` ids (`contracts/approval.schema.js`), `render-action` guards the render stage, 15-minute `Approval_Token` TTL, R11 Spend_Isolation_Boundary | Exactly one catalog-only zero-spend id (`edit-manifest-assembly`) exists for observability and is never referenced by a spend-bearing path |
| Cost / token accounting | `contracts/cost-log.schema.js` canonical `Cost_Log` (`model, prompt_tokens, completion_tokens, cache_hits, estimated_cost_usd, incomplete`); Director per-stage aggregation (`mcp/video-remix/cost-log.js`); USD budget-cap halt (R4.6) | A parallel **token**-denominated ceiling (distinct from the existing USD `budgetUsd` cap) applied to the narrative-reasoning stage's `prompt_tokens + completion_tokens`, reusing the same `Cost_Log` shape and the same halt-and-record pattern as the USD cap |
| Editing / composition | The Director includes an `edit` stage between `render` and `publish` | An `Edit_Manifest` sequences the run's rendered shot assets in storyboard order with per-shot trim/transition metadata, and an `Edited_Video_Reference` persists through the existing media path |

## Glossary

- **Video_Agent**: The enhancement defined by this specification — the narrative-coherence check, the token-budget ceiling and its degraded mode, the live BytePlus video-generation wiring, and the new Editing_Stage — layered on the existing Video_Remix Director. It introduces no new orchestrator; it extends the Director's existing stage sequence, retry policy, and circuit-breaker.
- **Director**: The existing durable orchestrator (`knowgrph.video_remix.run`, `mcp/video-remix-runtime.js` / `mcp/video-remix/run-video-remix.js`) that sequences stages, enforces budget and approval gates, and applies bounded-retry failure handling. Unmodified in its core contract by this spec except for the stage-sequence extension in Requirement 4.6.
- **Research_Harness**, **Storyboard_Harness**, **Render_Harness**, **Commerce_Harness**, **Evidence_Pack**, **Source_Card**, **Kgc_Document**, **Approval_Gate**, **Approval_Token**, **Cost_Log**, **Budget_Meters**, **Run_Manifest**, **Run_State**, **Dry_Run**, **Live_Mode**: as defined in the `knowgrph-acos-mcp-connector` spec's Glossary; reused verbatim by this specification without redefinition.
- **BytePlus_Video_Provider**: The existing `createBytePlusVideoProvider` factory (`mcp/video-remix/render-providers.js`) that submits an async video-generation task and polls it to completion through the `Ai_Gateway_Client`, then persists the result to R2 through an injected media persister. Reused unmodified by this specification.
- **Ai_Gateway_Client**: The existing `createAiGatewayClient` (`mcp/video-remix/ai-gateway-client.js`) — the single egress for BytePlus chat, image, and async video calls, routed through the Cloudflare AI Gateway. Reused unmodified by this specification.
- **Narrative_Coherence_Check**: A read-time validation applied to a Storyboard_Harness result's `plannedShots[]`: no two consecutive shots (by storyboard order) carry an identical `prompt` string. This is the observable, testable proxy for "the storyboard advances the narrative rather than repeating a beat" used to demonstrate the Video_Agent's narrative ability.
- **Shot_Prompt_Traceability**: The multimodal-orchestration handoff guarantee that the `prompt` string a completed `plannedShots[]` entry carries is passed unmodified as the `prompt` argument to the video-generation dispatch (`BytePlus_Video_Provider.dispatch({ ..., prompt, ... })`) for that same `shotId`, so the text-to-video handoff is observable and auditable end to end.
- **Token_Budget_Ceiling**: A configured, per-run, non-negative integer bound on the sum of `prompt_tokens + completion_tokens` across every narrative-reasoning (chat) call the Storyboard_Harness makes for that run, distinct from the existing USD `budgetUsd` cap. Omitted or zero means no ceiling is enforced.
- **Narrative_Degraded_Mode**: The defined fallback the Storyboard_Harness enters when continuing narrative reasoning would exceed the Token_Budget_Ceiling: it stops issuing further narrative-reasoning calls for that run and completes the storyboard using the shots already planned (a partial, still-schema-valid `Kgc_Document`) rather than exceeding the ceiling or throwing.
- **Editing_Stage**: The new Director stage inserted between `render` and `publish`. Given the run's completed render-stage assets, it produces an `Edit_Manifest` and an `Edited_Video_Reference`.
- **Edit_Manifest**: The ordered, per-shot composition plan the Editing_Stage produces: `{ runId, sequence: [{ shotId, assetUrl, startMs, endMs, transitionToNext }] }`, where `sequence` is ordered identically to the storyboard's `plannedShots[]` order.
- **Edited_Video_Reference**: The single resolvable media reference the Editing_Stage produces for the composed output, in the same resolvable-media-reference shape (`assetUrl` / `durableR2Url`) the Render_Harness already produces for a single shot asset.
- **Token_Emission_Gap**: A narrative-reasoning call that produced a schema-valid `Cost_Log` entry whose `prompt_tokens` or `completion_tokens` field is the existing `"unknown"` indicator rather than a concrete integer, making it unusable for Token_Budget_Ceiling accounting.

## Requirements

### Requirement 1: Narrative Coherence Across a Planned Shot Sequence

**User Story:** As an operator demonstrating the Video_Agent, I want the storyboard stage's planned shots to visibly advance the narrative rather than repeat a beat, so that I can show the agent's narrative ability with an observable, checkable property rather than a subjective claim.

#### Acceptance Criteria

1. WHEN the Storyboard_Harness completes with two or more planned shots, THE Video_Agent SHALL verify that no two consecutive shots (by storyboard order) carry an identical `prompt` string, where "identical" means exact, case-sensitive, character-for-character string equality.
2. IF two or more consecutive planned shots carry an identical `prompt` string (per the exact, case-sensitive, character-for-character equality defined in Criterion 1), THEN THE Video_Agent SHALL record a Narrative_Coherence_Check failure that explicitly reports, in the `repeatedShotIds` field, every offending shot id across all duplicate pairs found (not merely the first pair or the first shot), and SHALL leave the storyboard's existing `schemaValid`, `sourceReferences`, and `shotCount` fields unchanged (Requirement 1 is an additive check, not a replacement for the existing `kgc-computing-flow/v1` validation).
3. THE Video_Agent SHALL report the Narrative_Coherence_Check result as a `narrativeCoherence` field with shape `{ ok: boolean, repeatedShotIds: string[] }`, as part of the storyboard stage's observable result, alongside the existing `schemaValid`, `sourceReferences`, and `shotCount` fields.
4. WHEN the Storyboard_Harness completes with fewer than two planned shots, THE Video_Agent SHALL report the Narrative_Coherence_Check result as `narrativeCoherence: { ok: true, repeatedShotIds: [] }`, applying identically to a single-node fallback storyboard.

> **VCC translation** (Requirement 1.1): `Verify a storyboard result with N>=2 planned shots whose consecutive prompts are pairwise distinct reports narrativeCoherence.ok:true with an empty repeatedShotIds, and a result with any consecutive-duplicate prompt reports ok:false naming both shot ids, with the existing schemaValid/sourceReferences fields unchanged.`

### Requirement 2: Multimodal Handoff Traceability From Storyboard to Video Generation

**User Story:** As an operator demonstrating the Video_Agent, I want each planned shot's storyboard prompt to be traceably the same prompt used to generate that shot's video, so that I can show the agent's multimodal orchestration skill end to end rather than asserting it.

#### Acceptance Criteria

1. WHEN the render stage dispatches a shot for video generation through the BytePlus_Video_Provider, THE Video_Agent SHALL pass that shot's storyboard-planned `prompt` string as the `prompt` argument of the dispatch call, byte-for-byte identical to the storyboard-planned prompt.
2. WHEN a render asset completes, THE Video_Agent SHALL record the `shotId` and the exact `prompt` string used for that shot's dispatch, so the storyboard-to-render handoff is inspectable per shot after the run completes.
3. IF a shot dispatched to the BytePlus_Video_Provider has no corresponding planned shot in the run's storyboard, THEN THE Video_Agent SHALL permit the dispatch to proceed using the caller-supplied prompt for that shot, rather than fabricating a prompt.
4. IF a shot dispatched to the BytePlus_Video_Provider has no corresponding planned shot in the run's storyboard, THEN THE Video_Agent SHALL record an explicit `unplannedShotDispatch` log entry naming the `shotId`, rather than silently dispatching with no trace of the deviation.
5. THE Video_Agent SHALL NOT alter, truncate, or re-summarize a shot's storyboard prompt before passing it to the BytePlus_Video_Provider.
6. IF the BytePlus_Video_Provider or its underlying dispatch mechanism itself rejects a shot because it has no corresponding planned shot in the run's storyboard, THEN THE Video_Agent SHALL respect that rejection and SHALL NOT override it to force the dispatch to proceed; Criterion 3's permit-to-proceed rule applies only when the dispatch mechanism itself does not reject the unplanned shot.

> **VCC translation** (Requirement 2.1): `Verify, for a run with N planned shots each dispatched to the render stage, that every recorded render-dispatch prompt is byte-identical to that shot's storyboard-planned prompt, and that a dispatch for a shotId absent from the storyboard proceeds using the caller-supplied prompt while emitting an unplannedShotDispatch log entry naming that shotId.`

### Requirement 3: Live BytePlus Video-Generation Stage

**User Story:** As an operator, I want the Director's render stage to generate real video assets through the already-integrated BytePlus video model when live mode and an approved render-action Approval_Token are present, so that the pipeline produces an actual video from the planned storyboard prompt.

#### Acceptance Criteria

1. WHEN the render stage executes in Live_Mode with a verified, unexpired `render-action` Approval_Token and no different render provider is explicitly configured for the run, THE Video_Agent SHALL dispatch each planned shot through the BytePlus_Video_Provider (`submitVideo` → `pollVideoUntilDone` → R2 persistence), avoiding the generic `createStrytreeRenderQueueClient` render-queue client.
2. WHERE the run is explicitly configured to use a different, named render provider (Requirement 8), THE Video_Agent SHALL dispatch each planned shot through that explicitly configured provider instead of the BytePlus_Video_Provider.
3. THE Video_Agent SHALL reuse the existing 5-second poll interval and 600-second maximum poll duration already defined by `pollVideoUntilDone` (`mcp/video-remix/ai-gateway-client.js`) without introducing a second polling policy.
4. IF a BytePlus video-generation dispatch fails (submit error, poll error, or a poll that does not complete within the 600-second maximum poll duration defined in Criterion 3), THEN THE Video_Agent SHALL mark that shot's render as failed, record the actual spend incurred as a Credit_Ledger event using the existing Render_Harness ledger contract (R8.4) even though no asset reference is produced for that shot, and leave every other shot asset already rendered for that run unchanged, per the existing Render_Harness failure contract (R8.6).
5. WHEN a BytePlus video-generation dispatch completes successfully, THE Video_Agent SHALL persist the resulting video bytes to the existing R2 media bucket and record exactly one resolvable asset reference under the existing knowgrph media bucket and exactly one Credit_Ledger event for that shot, per the existing Render_Harness contract (R8.3, R8.4).
6. THE Video_Agent SHALL NOT introduce a second video-generation provider integration, a second AI Gateway client, or a second BytePlus endpoint configuration; it SHALL route every video-generation call through the existing `Ai_Gateway_Client`.
7. IF no provider key is available or the run's cumulative provider spend meets or exceeds the configured budget cap, THEN THE Video_Agent SHALL route the shot to the existing deterministic zero-spend mock provider, per the existing routing rule (R8.5), rather than attempting a live BytePlus call.
8. IF the BytePlus provider key becomes unavailable partway through a run after one or more shots have already been dispatched live through the BytePlus_Video_Provider, THEN THE Video_Agent SHALL continue attempting BytePlus dispatch for that run's remaining shots rather than switching those shots to the mock provider mid-run, accepting that a remaining dispatch MAY fail with an authentication error handled per Criterion 4.
9. WHERE the run's BytePlus configuration is present, THE Video_Agent routing a given shot to the mock provider under Criterion 7 (budget cap met/exceeded) SHALL NOT prevent other shots in the same run from being dispatched live through the BytePlus_Video_Provider.

> **VCC translation** (Requirement 3.1): `Verify a Live_Mode run with a verified render-action Approval_Token and a configured BytePlus video endpoint, and no explicitly configured different render provider, dispatches every planned shot through createBytePlusVideoProvider (observable via a mocked Ai_Gateway_Client receiving one submitVideo call per shot), and that mcp/video-remix/live-clients.js resolveStageClients() constructs no second video-generation client.`

### Requirement 4: Editing Stage — Ordered Composition of Rendered Shots

**User Story:** As an end creator, I want the individually generated shots assembled into one ordered, edited video, so that the pipeline output is a single cohesive video rather than a set of disconnected clips.

#### Acceptance Criteria

1. WHEN the render stage completes with one or more shot assets, THE Video_Agent SHALL produce an Edit_Manifest whose `sequence[]` lists every completed shot asset exactly once, ordered identically to the storyboard's `plannedShots[]` order, using, for any `shotId` with more than one completed render asset, only the most recently completed asset for that `shotId`.
2. THE Video_Agent SHALL include, for each `Edit_Manifest.sequence[]` entry, the shot's `shotId`, its resolvable `assetUrl`, and start/end trim points (`startMs`, `endMs`) that default to the full duration of that shot's rendered asset when no trim is specified.
3. IF a `sequence[]` entry's specified `startMs` or `endMs` is negative, `endMs` is less than or equal to `startMs`, or either value exceeds the duration of that shot's rendered asset, THEN THE Video_Agent SHALL reject the Edit_Manifest, record the trim validation error identifying the offending `shotId`, and SHALL NOT produce an Edited_Video_Reference.
4. WHEN the Edit_Manifest is complete and passes trim validation, THE Video_Agent SHALL produce exactly one Edited_Video_Reference for the run, resolvable under the existing knowgrph media bucket in the same shape the Render_Harness already produces for a single shot asset.
5. IF the render stage completes with zero shot assets (all shots failed or none were dispatched), THEN THE Video_Agent SHALL skip the Editing_Stage, record that it was skipped and why, and SHALL NOT produce an Edit_Manifest or an Edited_Video_Reference.
6. IF the Editing_Stage was skipped, THEN THE Video_Agent SHALL block the publish stage from proceeding for that run and record the skip reason as the blocking cause.
7. IF composing the Edited_Video_Reference fails after the Edit_Manifest was produced, THEN THE Video_Agent SHALL mark the edit stage as failed, record the composition error, leave every already-produced per-shot render asset unchanged, and retain the already-produced Edit_Manifest unchanged for diagnostics.
8. THE Editing_Stage SHALL execute after the render stage completes and before the publish stage begins, extending the Director's existing stage order to `research → storyboard → render → edit → publish → checkout`.

> **VCC translation** (Requirement 4.1): `Verify a run with N completed shot assets produces an Edit_Manifest whose sequence has exactly N entries in the same order as the storyboard's plannedShots, and a run with zero shot assets produces no Edit_Manifest and records an explicit skip reason.`

### Requirement 5: Token-Budget Ceiling for Narrative Reasoning

**User Story:** As a solo founder operating under a fixed token budget, I want the storyboard/narrative-reasoning stage to stay within a configured token ceiling per run, so that output quality is maximized without unbounded spend when the budget is tight.

#### Acceptance Criteria

1. WHERE a Token_Budget_Ceiling is configured for a run and that configured value is greater than zero, THE Video_Agent SHALL track the cumulative `prompt_tokens + completion_tokens` across every narrative-reasoning (chat) call the Storyboard_Harness makes for that run, using the existing `Cost_Log` fields without introducing a second token-accounting record. A configured Token_Budget_Ceiling of zero SHALL be treated identically to no Token_Budget_Ceiling being configured, for the purposes of this Requirement 5.
2. THE Video_Agent SHALL always issue the first narrative-reasoning call of a run regardless of the configured Token_Budget_Ceiling, so that Narrative_Degraded_Mode SHALL never be entered before at least one shot has been planned. IF, after a narrative-reasoning call completes, the cumulative token count tracked per Requirement 5.1 is greater than or equal to the configured Token_Budget_Ceiling, THEN THE Video_Agent SHALL enter Narrative_Degraded_Mode: it SHALL stop issuing further narrative-reasoning calls for that run and complete the storyboard using the shots already planned.
3. WHEN Narrative_Degraded_Mode is entered, THE Video_Agent SHALL record the degradation in the storyboard stage's result (`{ degraded: true, reason: "token_budget_ceiling", plannedShotCountAtDegradation: number }`) and SHALL still emit a schema-valid `Kgc_Document` over the shots already planned.
4. THE Video_Agent SHALL NOT exceed a configured Token_Budget_Ceiling under any narrative-reasoning path, including the existing single-node fallback path (Requirement is satisfied vacuously when the fallback's token cost is already at or under the ceiling).
5. WHERE a Token_Budget_Ceiling is configured for a run and that configured value is greater than zero (i.e., budget enforcement is active per Criterion 1), IF a narrative-reasoning call's `Cost_Log` entry is a Token_Emission_Gap (an `"unknown"` token field), THEN THE Video_Agent SHALL treat that call as consuming the full remaining Token_Budget_Ceiling for the purpose of the Requirement 5.2 check, so a token-accounting gap never silently permits unbounded spend when budget enforcement is active.
6. WHERE no Token_Budget_Ceiling is configured for a run, or a configured Token_Budget_Ceiling is zero, THE Video_Agent SHALL apply no token ceiling, SHALL NOT apply the Criterion 5 protective assumption to any Token_Emission_Gap, and SHALL behave exactly as the existing Storyboard_Harness does today.

> **VCC translation** (Requirement 5.2): `Verify a run configured with a Token_Budget_Ceiling greater than zero always issues its first narrative-reasoning call, then after each subsequent call whose post-call cumulative token count is >= the ceiling, halts further narrative reasoning, reports degraded:true with reason "token_budget_ceiling", and still emits a Kgc_Document that passes the existing kgc-computing-flow/v1 validation over the shots planned before degradation.`

### Requirement 6: Reuse of the Existing Approval-Gate and Spend-Isolation Boundary

**User Story:** As a solo founder, I want the new video-generation and editing stages to spend only through the existing approval and spend-isolation machinery, so that adding this capability introduces no new way to spend money without approval.

#### Acceptance Criteria

1. WHEN the Video_Agent dispatches a live BytePlus video-generation call, THE Video_Agent SHALL have first verified the same valid, unexpired `render-action` Approval_Token the existing Render_Harness already requires.
2. WHEN the Editing_Stage is triggered for an action that incurs provider spend, THE Video_Agent SHALL require a verified, unexpired `render-action` Approval_Token before executing that action.
3. WHEN the Editing_Stage is triggered, THE Video_Agent SHALL execute the Editing_Stage's zero-spend actions (Edit_Manifest assembly) regardless of Approval_Token presence; this zero-spend execution guarantee applies independently of Criterion 2, so gating a spend-bearing sub-action of the same Editing_Stage trigger under Criterion 2 SHALL NOT block execution of a zero-spend sub-action of that same trigger.
4. THE Video_Agent SHALL NOT introduce a new Approval_Gate id for any action that incurs provider spend; every spend-bearing boundary this specification adds resolves to one of the six existing canonical gate ids defined by `contracts/approval.schema.js`. WHERE an Editing_Stage action incurs zero provider spend and does not fit an existing gate's `actionKind`, THE Video_Agent SHALL introduce a new, zero-spend-only Approval_Gate id scoped exclusively to that zero-spend action, and SHALL NOT reuse that new id for any spend-bearing action.
5. THE Video_Agent SHALL NOT modify `contracts/approval.schema.js`, the 15-minute Approval_Token TTL, or the R11 Spend_Isolation_Boundary.
6. IF the render-action Approval_Token is missing, is expired, is malformed (fails structural or signature validation against `contracts/approval.schema.js`), or is gate-mismatched (its gate id does not equal the `render-action` gate id required for this dispatch) at the point of a live BytePlus dispatch, THEN THE Video_Agent SHALL reject the dispatch per the existing Render_Harness rejection contract (R8.2), performing zero provider spend for that shot.
7. WHERE the Video_Agent introduces a new zero-spend-only Approval_Gate id under Criterion 4, THE Video_Agent SHALL ensure every Approval_Gate instance carrying that id has an `estimatedCostUsd` value of exactly 0, and SHALL NOT reference that id from any code path that incurs provider spend.

> **VCC translation** (Requirement 6.4, 6.7): `Verify every spend-bearing boundary check added by this spec resolves to a gate id already present in contracts/approval.schema.js APPROVAL_GATE_ID_VALUES, and any newly introduced zero-spend-only gate id is never referenced by a spend-bearing code path (estimatedCostUsd is 0 for every ApprovalGate carrying that id across the test suite).`

### Requirement 7: Cost and Token Accounting Through Existing Ledgers

**User Story:** As a solo founder, I want video-generation and editing spend to appear in the existing cost ledger and budget meters, so that I have one place to see real spend without a second accounting system.

#### Acceptance Criteria

1. THE Video_Agent SHALL emit every video-generation and Editing_Stage `Cost_Log` entry in the existing canonical shape (`model, prompt_tokens, completion_tokens, cache_hits, estimated_cost_usd, incomplete`) defined by `contracts/cost-log.schema.js`.
2. THE Video_Agent SHALL validate every emitted `Cost_Log` entry against the existing `validateCostLog()` before including it in the run's aggregated `Budget_Meters`; a `Budget_Meters` update that includes no new `Cost_Log` entry (e.g., a zero-spend Edit_Manifest assembly step) SHALL require no `validateCostLog()` call for that update.
3. THE Video_Agent SHALL record every spend-bearing event, including video-generation provider spend and any spend-bearing Editing_Stage action, as a Credit_Ledger event using the existing Render_Harness ledger contract (R8.4), without introducing a second ledger.
4. IF an emitted `Cost_Log` entry fails `validateCostLog()`, THEN THE Video_Agent SHALL exclude that entry from the run's aggregated `Budget_Meters`, record the entry as a validation failure, and continue aggregating the run's remaining valid `Cost_Log` entries.
5. WHEN THE Video_Agent attempts to emit a `Cost_Log` entry for any video-generation or Editing_Stage action, THE Video_Agent SHALL call `validateCostLog()` for that entry regardless of the spend amount it records, including an entry recording zero spend; Criterion 2's no-`validateCostLog()`-call allowance applies only to a `Budget_Meters` update for which no `Cost_Log` entry is attempted at all.
6. THE Video_Agent SHALL NOT introduce a second currency, unit, or spend-aggregation mechanism alongside the existing `estimated_cost_usd` / `providerSpendCents` fields.

> **VCC translation** (Requirement 7.1): `Verify every Cost_Log entry emitted by the video-generation and editing stages passes contracts/cost-log.schema.js validateCostLog(), with contracts/cost-log.schema.js unmodified.`
> **VCC translation** (Requirement 7.4): `Verify that a Cost_Log entry failing validateCostLog() is excluded from the run's aggregated Budget_Meters, is recorded as a validation failure, and does not halt aggregation of the run's remaining valid Cost_Log entries.`

### Requirement 8: Reuse of the Existing BytePlus Integration Path

**User Story:** As a solo founder prioritizing the existing tech stack, I want the video-generation stage to route through the BytePlus integration knowgrph already has, so that this feature adds zero new provider-integration surface.

#### Acceptance Criteria

1. THE Video_Agent SHALL route every video-generation model call through the existing `Ai_Gateway_Client` (`mcp/video-remix/ai-gateway-client.js`) methods `submitVideo`, `pollVideo`, and `pollVideoUntilDone`, which route to BytePlus ModelArk through the Cloudflare AI Gateway, and THE Video_Agent SHALL NOT issue any video-generation model call through any other client, module, or code path.
2. THE Video_Agent SHALL NOT add a new BytePlus API key configuration surface, a new BytePlus endpoint host, or a new proxy path distinct from the existing `byteplus-modelark` provider id already proxied by `huijoohwee/functions/api/_integrationHub.js` and `huijoohwee/functions/__chat_proxy/[[path]].js`.
3. WHEN `resolveStageClients()` (`mcp/video-remix/live-clients.js`) constructs a live render client for the video-generation stage, THE Video_Agent SHALL construct it via `createBytePlusVideoProvider` instead of routing through `createStrytreeRenderQueueClient` or defining a second HTTP client for the same provider, so that `createStrytreeRenderQueueClient` is not used for the video-generation stage.
4. IF no live BytePlus video-generation configuration is present at the point `resolveStageClients()` would otherwise construct a live client, meaning the BytePlus endpoint and API key are both absent or empty, THEN THE Video_Agent SHALL default to the existing deterministic mock video provider, making zero live/paid calls, per the existing `resolveStageClients()` fail-safe default.
5. IF the live BytePlus video-generation configuration is absent (endpoint and key both absent or empty) or incomplete (exactly one of endpoint or key is present, but not both) at the point `resolveStageClients()` would otherwise construct a live client, THEN THE Video_Agent SHALL construct no `Ai_Gateway_Client` video call at all for that run and SHALL route every shot to the deterministic mock provider, so an incomplete configuration can never partially or accidentally trigger a live call.
6. WHERE a live BytePlus configuration is present for the run, THE Video_Agent routing an individual shot to the mock provider under Requirement 3's per-shot rules (e.g., a budget cap met/exceeded per Requirement 3 Criterion 7) SHALL NOT be construed as "no live BytePlus video-generation configuration is present" for the purpose of Criteria 4-5, and other shots in that same run remain eligible for live dispatch through the BytePlus_Video_Provider.

> **VCC translation** (Requirement 8.1): `Verify no new outbound host, provider id, or API-key environment variable is introduced by this increment's diff; every live video-generation network call in a test trace originates from Ai_Gateway_Client.submitVideo/pollVideo; and a run with absent/incomplete BytePlus video configuration produces zero Ai_Gateway_Client invocations, routing every shot to the deterministic mock provider instead.`

### Requirement 9: Bounded Orchestration Preserved

**User Story:** As a solo founder, I want the enhanced pipeline to keep the Director's existing bounded-retry and circuit-breaker guarantees, so that adding video-generation and editing never introduces an unbounded loop.

#### Acceptance Criteria

1. THE Video_Agent SHALL execute within the Director's existing maximum-iteration bound (`maxIterations`, default 8, accepted range [1,100]) using a single shared stage retry count for the video-generation and Editing_Stage, treating an Editing_Stage composition failure (as defined in Requirement 4.7) as a retryable stage-attempt failure under this same bounded-retry mechanism, without introducing a second retry counter.
2. WHEN a per-shot video-generation dispatch fails or an Editing_Stage composition attempt fails, THE Video_Agent SHALL increment the single shared stage retry count referenced in Criterion 1 by exactly 1 for that failure, and SHALL NOT maintain a separate per-shot retry counter.
3. WHEN a video-generation or Editing_Stage call is retried, THE Video_Agent SHALL apply the Director's existing exponential backoff (starting at 1 second, capped at 30 seconds per attempt) to that retry.
4. IF the video-generation or Editing_Stage retries exhaust the configured `maxIterations`, THEN THE Video_Agent SHALL fail closed to Run_State `blocked` for the entire run — halting every downstream stage (edit, publish, checkout when the exhausted stage is render; publish, checkout when the exhausted stage is edit) rather than only the exhausted stage — SHALL leave every already-completed upstream stage's status and recorded spend unchanged, SHALL record no new spend for any halted downstream stage, and SHALL append a failure record `{ stageId, finalRetryCount, reason }`, per the Director's existing exhaustion contract (R5.4).
5. THE Video_Agent SHALL trigger the Director's existing circuit-breaker on the same condition set `{blocked, budget_exceeded, approval_required, verification_failed}` without adding a new circuit-breaker condition.

> **VCC translation** (Requirement 9.4): `Verify a video-generation or edit-stage retry sequence that exhausts maxIterations produces Run_State "blocked", a failure record whose finalRetryCount equals maxIterations, using the Director's existing retry.js exhaustion helpers unmodified, and every stage downstream of the exhausted stage remains unexecuted.`

### Requirement 10: Zero New Dependency and Zero New Persistent Datastore (Guardrail)

**User Story:** As a solo founder operating a TCO-zero stack, I want this enhancement to add no new paid dependency and no new datastore, so that the Video_Agent's monthly TCO delta stays at $0 beyond BytePlus's already-accounted per-call spend.

#### Acceptance Criteria

1. THE Video_Agent SHALL introduce zero new npm or pip dependency (whether listed as a production dependency or a devDependency, and whether direct or transitive) in `mcp/`, `contracts/`, or `cloudflare/workers/knowgrph-mcp`, measured against each affected package's dependency manifest state immediately before this increment's changes, beyond what `createBytePlusVideoProvider` and `createAiGatewayClient` already require; this bound is absolute.
2. IF a video-editing/composition capability needed to satisfy Requirement 4 appears to require a capability that neither `createBytePlusVideoProvider` nor `createAiGatewayClient` already provides, THEN THE Video_Agent SHALL satisfy that need using only existing in-repo capabilities (e.g., an Edit_Manifest consumed at playback time by an existing renderer rather than a physical re-encode) rather than adding a new dependency, per Open Question 1.
3. THE Video_Agent SHALL introduce zero new persistent datastore (a new D1 table, a new R2 bucket, a new KV namespace, or a new Durable Object class); the Edit_Manifest and the Edited_Video_Reference SHALL persist exclusively through the existing R2 media bucket already used by the Render_Harness and the existing Run_Manifest storage already used for run state, with no additional storage mechanism introduced.
4. IF a capability gap is found during design where the Editing_Stage's intended behavior cannot be achieved using only the existing in-repo capabilities described in Acceptance Criteria 1-2 (e.g., a video-composition/editing capability BytePlus does not expose), THEN THE Video_Agent's design phase SHALL resolve the Editing_Stage's scope to what is achievable with zero new dependencies (per Acceptance Criterion 1) and SHALL record the unmet capability as a deferred ADR candidate — a written entry containing the problem statement and an explicit comparison against at least one FOSS/free-tier alternative — rather than adding the dependency in this increment.
5. THE Video_Agent MAY ship its initial release with the Editing_Stage's scope limited to only the capabilities resolved achievable under Criterion 4, and SHALL NOT be blocked from launching by an unmet editing capability that has been recorded as a deferred ADR candidate per Criterion 4; every deferred capability SHALL remain visible in the ADR record rather than being silently dropped from the specification.

> **VCC translation** (Requirement 10.1): `Verify package.json/package-lock.json (or equivalent pip lock) manifests under mcp/, contracts/, and cloudflare/workers/knowgrph-mcp show zero added dependencies (production or dev) attributable to this increment's diff relative to the pre-increment manifest state, with no exception for editing/composition needs — any such need is met by an Edit_Manifest/playback-time approach (Acceptance Criterion 2) or deferred as an ADR candidate (Acceptance Criterion 4).`

## Non-Functional Requirements

- **Narrative ability (demonstrability)**: an operator or judge SHALL be able to observe, from a single run's recorded output, that (a) the Narrative_Coherence_Check passed (Requirement 1) and (b) every render dispatch's prompt matches its storyboard-planned prompt (Requirement 2), without inspecting internal logs beyond the Run_Manifest.
- **Multimodal orchestration (demonstrability)**: the same Run_Manifest SHALL make the text (storyboard) → video (render) → composed (edit) handoff traceable shot-by-shot.
- **Token economics**: every narrative-reasoning call's token cost is tracked via the existing `Cost_Log` shape (Requirement 5, 7); a configured Token_Budget_Ceiling is never silently exceeded (Requirement 5.4).
- **TCO-zero**: zero new dependency, zero new datastore, zero new provider-integration surface (Requirements 8, 10); BytePlus per-call spend is the only cost delta, already accounted through the existing Cost_Log/Credit_Ledger.
- **Harness-first**: every new capability (narrative check, editing) is expressed as a typed input/output extension of an existing harness (Storyboard_Harness, Render_Harness) or a small new harness (Editing_Stage) with the same contract shape (typed input → typed output → Cost_Log → fallback), not an ad-hoc prompt call.

## Success Metrics

| Metric | Baseline | Target | Timeline |
|---|---|---|---|
| Narrative_Coherence_Check pass rate on demo runs | Not measured today | 100% of demo runs report `ok:true` or a named, explainable failure | First increment ship |
| Shot_Prompt_Traceability match rate | Not measured today | 100% of completed render assets have a byte-identical prompt match to their storyboard shot | First increment ship |
| Video-generation stage live-wireable | Generic queue client only | Live BytePlus dispatch wired through `createBytePlusVideoProvider` and exercisable end to end in an approval-gated run | This increment |
| Editing_Stage present | 0 (no stage exists between render and publish) | 1 (Editing_Stage produces an Edit_Manifest + Edited_Video_Reference for every run with >=1 completed shot asset) | This increment |
| Token_Budget_Ceiling enforcement | Not tracked (USD cap only) | 100% of runs with a configured ceiling degrade rather than exceed it | This increment |
| New provider-integration surfaces added | 0 target | 0 | This increment (guardrail) |
| New paid dependency / datastore added | 0 target | 0 | This increment (guardrail) |
| Monthly TCO delta (Video_Agent glue itself) | N/A | $0 beyond BytePlus per-call spend, which is already accounted | Ongoing |
| Token cost / month (narrative-reasoning stage) | Unbounded per run | Bounded by the configured Token_Budget_Ceiling × expected demo run volume | Ongoing |
| ROI Score threshold | — | ≥ 40 for any item promoted out of Won't | Per sprint review |

### Time-to-Value: Operator Runs a Narrative-and-Video Demo

| Dimension | Estimate | Target ceiling | Validation method |
|---|---|---|---|
| TTV steps | 3 steps (configure a BytePlus live endpoint + Token_Budget_Ceiling; call `knowgrph.video_remix.run` in dry-run to preview; approve `render-action` and re-run live) | ≤ 3 steps | Walk-through on a clean checkout |
| TTV elapsed time | ~5 minutes (excluding actual BytePlus video-generation poll time, bounded at 600s per shot) | ≤ 10 minutes to first approval-gated live shot | Timed first-run test |
| First-value action | Operator sees one completed shot asset with a matching storyboard prompt and one Edit_Manifest entry | — | Observable Run_Manifest fields |
| Persona | Operator (solo founder demonstrating the agent) | — | Defined above |

## MoSCoW Priority

ROI Score = (User Impact × Reach) / (Build Hours + Monthly TCO + Token Cost/Month); Reach ≈ 40 demo/investor/judge sessions per month (a narrative-and-video demo is lower-frequency than the Agentic OS's operator-visibility calls).

| Tier | Requirement | ROI Score | Rationale |
|---|---|---|---|
| Must | R3 Live BytePlus video-generation stage | 50.0 | Directly demonstrates the headline capability (video generation); reuses 90% of existing code |
| Must | R2 Shot_Prompt_Traceability | 66.7 | Cheapest to build (a recording/assertion over an existing dispatch call); highest demo-credibility value |
| Must | R1 Narrative_Coherence_Check | 57.1 | Small, pure-function check; directly answers "does it demonstrate narrative ability" |
| Must | R6 Approval-gate reuse (guardrail) | n/a (guardrail) | Non-negotiable spend-isolation constraint |
| Must | R8 BytePlus integration reuse (guardrail) | n/a (guardrail) | Non-negotiable TCO-zero / reuse-first constraint |
| Must | R10 Zero new dependency/datastore (guardrail) | n/a (guardrail) | Non-negotiable cost-avoidance constraint |
| Should | R4 Editing_Stage | 33.3 | High demo value ("editing" is explicitly requested); the composition mechanism is resolved in ADR-VA-1 |
| Should | R5 Token_Budget_Ceiling + degraded mode | 36.4 | Directly demonstrates "maximize quality under limited token budget"; moderate build risk (new accounting path) |
| Should | R7 Cost/token accounting reuse | 40.0 | Needed for R5 and R3 to be observable; low build risk |
| Could | R9 Bounded orchestration preserved (explicit re-statement) | 30.0 | Mostly already true by construction; formalizing it as a requirement adds verification value, not new code |
| Won't (this increment) | A general-purpose video-editing UI or timeline editor | — | Out of scope for a headless Director stage; fails min-viable-max-value |
| Won't (this increment) | A second video-generation provider (e.g., PixVerse) as a fallback | — | Fails TCO-zero / reuse-first; BytePlus is the prioritized existing stack |
| Won't (this increment) | Real-time video preview/streaming during generation | — | Not required to demonstrate narrative ability or multimodal orchestration; adds unbounded scope |

## Out of Scope

- A general-purpose, user-facing video-editing timeline or UI; the Editing_Stage is a headless Director stage producing an Edit_Manifest and an Edited_Video_Reference, not an editing product.
- A second video-generation provider integration (PixVerse, Runway, etc.); this increment is scoped to the already-integrated BytePlus path.
- Modifying `contracts/approval.schema.js`, `contracts/cost-log.schema.js`, the six canonical Approval_Gate ids, or the 15-minute Approval_Token TTL.
- Modifying the Director's existing `maxIterations` bound, backoff policy, or circuit-breaker condition set.
- A new persistent datastore dedicated to Edit_Manifest or token-budget state; both persist through the existing Run_Manifest / R2 media bucket.
- Real-time push, streaming, or webhook delivery of render/edit progress; the Director's existing pull/read Run_Manifest model is unchanged.

## Dependencies

- `mcp/video-remix-runtime.js` / `mcp/video-remix/run-video-remix.js` — the existing Director orchestrator this spec extends with one new stage.
- `mcp/video-remix/storyboard-harness.js` — the existing narrative-reasoning harness this spec layers the Narrative_Coherence_Check and Token_Budget_Ceiling onto.
- `mcp/video-remix/render-harness.js`, `mcp/video-remix/render-providers.js` (`createBytePlusVideoProvider`, `createBytePlusImageProvider`) — the existing render harness and BytePlus provider factories this spec wires live.
- `mcp/video-remix/ai-gateway-client.js` (`submitVideo`, `pollVideoUntilDone`) — the existing single egress for BytePlus video calls; not modified by this spec.
- `mcp/video-remix/live-clients.js` (`resolveStageClients`, `resolveGateClientDeps`) — the existing env-gated client resolver this spec extends to construct `createBytePlusVideoProvider` instead of / alongside `createStrytreeRenderQueueClient` for the render stage.
- `mcp/video-remix/director-live-run.js` (`executeLiveStages`) — the existing async live-stage execution layer the Editing_Stage's live wiring composes with.
- `contracts/approval.schema.js`, `contracts/cost-log.schema.js` — existing canonical schemas reused unmodified.
- `huijoohwee/functions/api/_integrationHub.js`, `huijoohwee/functions/__chat_proxy/[[path]].js` (provider id `byteplus-modelark`) — existing BytePlus proxy path this spec's video-generation calls route through at the platform boundary.
- The `knowgrph-acos-mcp-connector` spec's Data Models, R4/R8/R11, and Property 1 invariants — existing normative source for stage ordering, the render Approval_Gate, and the Spend_Isolation_Boundary this spec must not weaken.
- The `knowgrph-agentic-os` spec's Track B (Live_Golden_Path_Run) notes — historical source for the render-stage gap that the runtime-ready Video_Agent closes.

## Resolved Decisions

1. **Editing composition mechanism**: resolved to an `Edit_Manifest` consumed at playback time and persisted through the existing media path; no physical re-encode and no new dependency.
2. **Live render-client routing**: resolved to default BytePlus routing through `AI_GATEWAY_VIDEO_URL` + `BYTEPLUS_API_KEY` (or `AI_GATEWAY_TOKEN`), with `RENDER_PROVIDER=strytree` as the explicit override.
3. **Editing_Stage spend gate**: resolved to zero provider spend for manifest assembly; `edit-manifest-assembly` is catalog-only and never used as a spend-bearing gate.
4. **Token_Budget_Ceiling default value**: resolved to 2000 tokens per run when the caller does not supply a ceiling.
