# Requirements Document

## Introduction

This feature enhances the Knowgrph repository (Dev SSOT at `/Users/huijoohwee/Documents/GitHub/knowgrph`) into an agentic widget canvas with end-to-end rich media generation, durable persistence, and cross-environment sync. A user supplies a brief; the system orchestrates text, image, and video generation through a single default provider routed via an AI gateway, persists every generated artifact to durable object storage, renders the artifacts as distinct widgets and Rich Media Panels on a mobile-first responsive canvas, and lets the user replay generated media from durable storage without invoking any model. Canvas state and artifacts auto-save, are retrievable, and sync across the fixed Dev → Prod mirror → Cloudflare topology backed by a shared remote document store.

The platform target is Cloudflare exclusively. The system persists only durable storage URLs (never ephemeral provider URLs), gates paid actions behind explicit human approval, links every artifact to its provenance chain, and remains fully testable offline using deterministic mock providers. This document is a planning artifact only; it does not change code.

## Glossary

- **Knowgrph_System**: The Knowgrph application and runtime under enhancement, rooted at the Dev SSOT `/Users/huijoohwee/Documents/GitHub/knowgrph`.
- **Widget_Canvas**: The agentic canvas surface that hosts Text widgets, Image widgets, Video widgets, Rich Media Panels, and Edges in a balanced 16:9 (1920x1080) logical layout.
- **Rich_Media_Panel**: A canvas panel type that renders a generated artifact (text, image, or video) and supports replay. Image Rich Media Panels and Video Rich Media Panels are distinct panel/widget types.
- **Image_Panel**: The Rich Media Panel type and widget type dedicated to image artifacts.
- **Video_Panel**: The Rich Media Panel type and widget type dedicated to video artifacts.
- **Edge**: A typed data-flow/provenance connection between widgets and Rich Media Panels on the Widget_Canvas.
- **AI_Gateway**: The Cloudflare AI Gateway through which all provider model calls are routed.
- **Media_Provider**: The default media/model provider, BytePlus ModelArk, accessed through the AI_Gateway. Chat uses model `agnes/seed`; image uses model `seedream`; video uses model `seedance`.
- **Mock_Provider**: A deterministic, offline implementation of the text/image/video provider contracts that requires no network and no paid services, used for baseline validation.
- **Ephemeral_Provider_URL**: A temporary media URL or payload returned by the Media_Provider that expires approximately 24 hours after generation.
- **R2_Store**: The Cloudflare R2 object storage used for durable artifact persistence (account id `170e89fdb8679ff2fcc2900e25ed04f4`, bucket `knowgrph-media`).
- **R2_Key_Scheme**: The R2 object key pattern `runs/{runId}/{stageId}/{shotId}.{ext}`.
- **Durable_R2_URL**: The persistent storage URL pointing to an artifact stored in the R2_Store.
- **D1_Store**: The Cloudflare D1 database used as the shared remote store for workspace and document state.
- **Sync_Topology**: The fixed environment chain Dev SSOT (`/Users/huijoohwee/Documents/GitHub/knowgrph`) → Prod artifact mirror (`/Users/huijoohwee/Documents/GitHub/huijoohwee/content/knowgrph`) → Cloudflare route (`airvio.co/knowgrph`).
- **Auto_Save**: The behavior that persists Widget_Canvas state and artifacts without explicit user save action.
- **Provenance_Chain**: The linked record connecting an artifact to its goal, source brief, plan, tool calls, and verification checks.
- **Approval_Token**: A verified per-gate authorization that permits a single category of paid or external action.
- **Spend_Gate**: A checkpoint that blocks a paid or external action until a matching Approval_Token is verified.
- **Payments_Provider**: Stripe, the provider used for payment and checkout actions.
- **Run**: A single execution identified by a unique `runId` that produces artifacts, trace, and a final result.
- **MCP_Surface**: The Model Context Protocol interface through which the pipeline is runnable and inspectable.
- **CLI_Surface**: The command-line interface through which the pipeline is runnable and inspectable.
- **App_UI_Surface**: The application user interface through which the pipeline is runnable and inspectable.
- **Responsive_Proof_Class**: One of the required viewport validation targets: 320x640, 390x844, 768x1024, 1366x768, and 1920x1080.
- **Mobile_Viewport_Threshold**: The viewport width boundary of 768 pixels; viewports narrower than this threshold are treated as mobile-class.
- **Runtime_Test_Gate**: The offline baseline validation command `npm run runtime:test` that must pass without network or paid services.

## Requirements

### Requirement 1: Agentic Widget Canvas Surface and Mobile-First Layout

**User Story:** As a creator, I want an agentic widget canvas that works from a narrow phone screen up to a wide desktop, so that I can orchestrate and inspect rich media generation on any device.

#### Acceptance Criteria

1. THE Knowgrph_System SHALL render a Widget_Canvas that hosts Text widgets, Image_Panel widgets, Video_Panel widgets, Rich_Media_Panel widgets, and Edges within a 16:9 logical presentation frame of 1920x1080 logical pixels.
2. THE Knowgrph_System SHALL derive the canvas layout for each Responsive_Proof_Class (320x640, 390x844, 768x1024, 1366x768, and 1920x1080) from shared layout metadata rather than from per-viewport hardcoded coordinates.
3. WHILE the active viewport width is below the Mobile_Viewport_Threshold of 768 pixels, THE Knowgrph_System SHALL present exactly one primary task surface at a time with every interactive control positioned within the visible viewport bounds and reachable without horizontal scrolling.
4. THE Knowgrph_System SHALL render every Text widget, Image_Panel widget, Video_Panel widget, and Rich_Media_Panel widget with explicit x, y, and z-index values, with each widget either fully contained within the active viewport bounds or reachable through an explicit fit control.
5. THE Knowgrph_System SHALL maintain zero overlapping pixel area between any two widgets in the active layout, with each widget's rendered area within plus or minus 2 percent of its layout-metadata-defined proportional size.
6. WHEN an alternative route exists between an Edge source port and an Edge target port, THE Knowgrph_System SHALL route the Edge so that it does not pass through the center 50 percent (by width and height) of any widget or Rich_Media_Panel content region.
7. IF no alternative route exists that avoids the center content region of every widget, THEN THE Knowgrph_System SHALL route the Edge along the shortest available port-to-port route and indicate the unavoidable crossing to the creator.
8. THE Knowgrph_System SHALL expose the canvas surface contract MainPanel Integrations, FloatingPanel Chat UI, Editor Workspace, Canvas, and balanced 16:9 widget layout as the navigable path for the pipeline.
9. THE Knowgrph_System SHALL persist responsive layout metadata for each artifact such that the stored layout can be inspected and re-rendered identically for each of the five Responsive_Proof_Class sizes.
10. IF persisted responsive layout metadata for a requested Responsive_Proof_Class is missing or fails validation, THEN THE Knowgrph_System SHALL retain the artifact's last valid layout, decline to render the invalid layout, and present an indication that the layout could not be loaded.

### Requirement 2: BytePlus Media Generation Through the AI Gateway

**User Story:** As a creator, I want text, image, and video generated through the default provider behind the AI gateway, so that I get rich media from a single governed integration path.

#### Acceptance Criteria

1. THE Knowgrph_System SHALL route every Media_Provider model call through the AI_Gateway.
2. WHEN the Knowgrph_System performs a chat generation, THE Knowgrph_System SHALL invoke the Media_Provider chat model `agnes/seed` through the AI_Gateway.
3. WHEN the Knowgrph_System performs an image generation, THE Knowgrph_System SHALL invoke the Media_Provider image model `seedream` through the AI_Gateway.
4. WHEN the Knowgrph_System performs a video generation, THE Knowgrph_System SHALL invoke the Media_Provider video model `seedance` through the AI_Gateway.
5. WHEN the Knowgrph_System submits a video generation request, THE Knowgrph_System SHALL treat the request as asynchronous by submitting a task, polling for completion at a fixed interval of 5 seconds for up to a maximum total duration of 600 seconds, and retrieving the resulting media reference upon completion.
6. IF a video generation task does not report completion within the maximum polling duration of 600 seconds, THEN THE Knowgrph_System SHALL stop polling, mark the generation step as failed, and report a timeout error indicating the video task did not complete, without recording a media reference.
7. IF a Media_Provider model call routed through the AI_Gateway returns an error or fails to return a result, THEN THE Knowgrph_System SHALL mark the generation step as failed and report a descriptive error indicating the failed model call, without rendering a partial or placeholder artifact.
8. THE Knowgrph_System SHALL render image artifacts in Image_Panel widgets and video artifacts in Video_Panel widgets as distinct panel and widget types.
9. WHEN a generation step produces an artifact, THE Knowgrph_System SHALL render the artifact in its matching widget within 1 second of artifact creation.
10. WHERE a node defines a local model override, THE Knowgrph_System SHALL use the node-local model in place of the matching group default for that node.
11. THE Knowgrph_System SHALL exclude Vercel and AWS from every generation, routing, storage, and deployment path, including the internal infrastructure used to reach the AI_Gateway and Media_Provider.

### Requirement 3: Persist-On-Generate to Durable R2 Storage

**User Story:** As a creator, I want every generated image and video copied to durable storage at generation time, so that my artifacts remain available after the provider's temporary URLs expire.

#### Acceptance Criteria

1. WHEN the Media_Provider returns a generated image as base64 or as an Ephemeral_Provider_URL, THE Knowgrph_System SHALL store the image in the R2_Store before completing the generation step.
2. WHEN the Media_Provider video task completes and returns an Ephemeral_Provider_URL, THE Knowgrph_System SHALL store the video in the R2_Store before completing the generation step.
3. THE Knowgrph_System SHALL store each artifact in the R2_Store using the R2_Key_Scheme `runs/{runId}/{stageId}/{shotId}.{ext}` in bucket `knowgrph-media` under account `170e89fdb8679ff2fcc2900e25ed04f4`.
4. THE Knowgrph_System SHALL record the Durable_R2_URL as the stored media reference for each artifact.
5. THE Knowgrph_System SHALL exclude the Ephemeral_Provider_URL from all persisted artifact records, canvas state, and synced documents.
6. IF the R2_Store persistence step fails after the configured maximum R2 write-attempt bound is reached, THEN THE Knowgrph_System SHALL mark the generation step as failed and report a descriptive error that identifies the failed artifact by `runId`, `stageId`, and `shotId`, while preserving state by writing no partial artifact record and retaining no Ephemeral_Provider_URL.
7. WHEN the R2_Store reports a successful write, THE Knowgrph_System SHALL verify the stored object is retrievable at the Durable_R2_URL within 10 seconds before recording the artifact as persisted.
8. IF the stored object is not retrievable at the Durable_R2_URL within the 10-second verification window, THEN THE Knowgrph_System SHALL treat the artifact as not persisted, mark the generation step as failed, and report a verification error, without recording the artifact as persisted.
9. WHEN two generation outputs produce identical content, THE Knowgrph_System SHALL store exactly one R2_Store object deduplicated by content hash and reference that single object from both artifacts.

### Requirement 4: Replay From R2 Without Model Invocation

**User Story:** As a viewer, I want to replay generated images and videos from durable storage, so that I can review artifacts repeatedly without triggering regeneration or new spend.

#### Acceptance Criteria

1. WHEN a user replays an Image_Panel artifact, THE Knowgrph_System SHALL load the image from the Durable_R2_URL through an embed or iframe surface and render it within 10 seconds.
2. WHEN a user replays a Video_Panel artifact, THE Knowgrph_System SHALL load the video from the Durable_R2_URL through an embed or iframe surface and present it as ready to play within 10 seconds.
3. WHEN a user replays any stored artifact, THE Knowgrph_System SHALL complete the replay with zero outbound requests to the Media_Provider, zero outbound requests to the AI_Gateway, and zero model calls.
4. IF the Durable_R2_URL does not return retrievable content within 10 seconds after a maximum of 2 retry attempts during replay, THEN THE Knowgrph_System SHALL display an explicit fallback state indicating the artifact is unavailable, SHALL NOT attempt regeneration, and SHALL retain the existing stored artifact reference unchanged.
5. WHILE the requesting user is entitled to the associated Run, THE Knowgrph_System SHALL grant replay access to the artifacts of that Run.
6. IF the requesting user is not entitled to the associated Run, THEN THE Knowgrph_System SHALL deny replay access, SHALL display an access-denied indication to the requester, and SHALL NOT load the artifact from the Durable_R2_URL.

### Requirement 5: Auto-Save, Retrieval, and Sync Across Dev → Prod → Cloudflare

**User Story:** As a creator, I want my canvas state and artifacts to auto-save, be retrievable, and sync across environments, so that my work is durable and consistent from development through production.

#### Acceptance Criteria

1. WHEN a node edit, run completion, approval, or artifact-ready event occurs, THE Knowgrph_System SHALL auto-save the Widget_Canvas state and artifact references to the D1_Store, debouncing edit events at 500 milliseconds and completing the D1_Store write within 2 seconds of the triggering event.
2. WHEN a workspace or document is opened or requested, THE Knowgrph_System SHALL retrieve previously saved Widget_Canvas state and artifact references from the D1_Store within 3 seconds.
3. THE Knowgrph_System SHALL use the D1_Store as the shared remote store for workspace and document state across the Sync_Topology.
4. THE Knowgrph_System SHALL originate state and source changes in the Dev SSOT, propagate generated artifacts to the Prod artifact mirror, and serve them from the Cloudflare route without route-specific overrides.
5. WHEN an auto-save write does not conflict with a concurrent edit, THE Knowgrph_System SHALL complete the auto-save without user intervention.
6. IF an auto-save write conflicts with a concurrent edit, THEN THE Knowgrph_System SHALL preserve the existing edit before any overwrite is considered, surface the conflict, and require an explicit user action to replace, merge, or discard.
7. WHEN multiple auto-save writes for the same canvas are in flight, THE Knowgrph_System SHALL apply latest-result ownership using a monotonically increasing version identifier derived from the most-recent originating event, so that a write with a lower sequence identifier never overwrites newer state.
8. THE Knowgrph_System SHALL anchor the Dev SSOT, the Prod mirror, and the Cloudflare route `airvio.co/knowgrph` to identical D1_Store records with no route-specific overrides.
9. IF a D1_Store auto-save write fails or is not acknowledged within 10 seconds, THEN THE Knowgrph_System SHALL retry up to 3 times and, if all retries fail, preserve unsaved local state and report a save-failure indication.
10. IF retrieval from the D1_Store fails, THEN THE Knowgrph_System SHALL report a retrieval-failure indication and SHALL NOT render partial or empty state as authoritative.

### Requirement 6: Provenance Linking

**User Story:** As a reviewer, I want every generated artifact linked to its origin and verification trail, so that I can audit how each artifact was produced.

#### Acceptance Criteria

1. WHEN the Knowgrph_System produces an artifact, THE Knowgrph_System SHALL record the complete Provenance_Chain linking the artifact to all of its goal, source brief, plan, tool calls, and verification checks before recording the artifact as persisted.
2. THE Knowgrph_System SHALL link each canvas artifact to its originating Run identified by `runId`.
3. THE Knowgrph_System SHALL persist the Provenance_Chain alongside the Durable_R2_URL in the saved artifact record.
4. WHEN a user inspects an artifact, THE Knowgrph_System SHALL present the artifact's Provenance_Chain within 2 seconds, including all of the goal, source brief, plan, tool calls, and verification checks.
5. THE Knowgrph_System SHALL keep provenance metadata after auto-save, retrieval, and sync across the Sync_Topology field-for-field identical to the recorded Provenance_Chain, with no loss or alteration of any component.
6. IF recording the Provenance_Chain fails or a required provenance component is missing, THEN THE Knowgrph_System SHALL mark the step as failed, report a descriptive error, and SHALL NOT persist the artifact.
7. IF an inspected artifact has no available or complete Provenance_Chain, THEN THE Knowgrph_System SHALL present an explicit fallback state indicating that provenance is unavailable.
8. THE Knowgrph_System SHALL restrict Provenance_Chain inspection to users entitled to the associated Run.

### Requirement 7: Stripe-Gated Paid Actions and Human Approval Before Spend

**User Story:** As an account owner, I want every paid or external action to require explicit approval, so that no spend or payment occurs without my authorization.

#### Acceptance Criteria

1. THE Knowgrph_System SHALL default each new Run to dry-run mode, in which zero paid or external actions are performed.
2. IF a Run is in live mode and a Spend_Gate lacks a verified Approval_Token, THEN THE Knowgrph_System SHALL halt at that Spend_Gate without performing the paid action, SHALL preserve all prior Run state with no rollback and no partial spend, and SHALL record the blocked state with an indication identifying the Spend_Gate and the reason for the block.
3. WHEN a verified Approval_Token is present for a Spend_Gate, THE Knowgrph_System SHALL permit exactly one execution of the single matching paid or external action category for that gate.
4. WHEN a paid or external action authorized by an Approval_Token completes, THE Knowgrph_System SHALL mark that Approval_Token as consumed so that it cannot authorize any subsequent action.
5. IF an Approval_Token is missing, has exceeded its configured validity window of 15 minutes, is malformed, or is otherwise unverifiable, THEN THE Knowgrph_System SHALL deny the matching action and SHALL record the denial with an indication of the verification-failure reason.
6. WHEN the Knowgrph_System creates a checkout or payment session, THE Knowgrph_System SHALL use the Payments_Provider (Stripe).
7. THE Knowgrph_System SHALL require a verified Approval_Token before any paid model call, render action, payment action, cloud deploy, consumer-repo write, or authenticated-browser action.
8. WHILE a Run is in live mode, THE Knowgrph_System SHALL record reconciled spend against the configured budget cap for the Run, expressed in the configured currency within the range 0.00 to 999,999,999.99.
9. IF estimated spend for a pending paid action would cause cumulative Run spend to exceed the configured budget cap, THEN THE Knowgrph_System SHALL halt the paid action, SHALL perform no partial spend, and SHALL record the budget breach, independently of any verified Approval_Token.

### Requirement 8: Failure Handling and Offline Mock-Provider Testability

**User Story:** As a developer, I want the pipeline to be testable offline with deterministic mocks and to recover from failures, so that I can validate behavior without network or paid services.

#### Acceptance Criteria

1. WHEN the Runtime_Test_Gate command `npm run runtime:test` runs, THE Knowgrph_System SHALL complete the Mock_Provider baseline run within 120 seconds, make zero outbound network requests, perform zero paid actions, and return a success exit code only when every baseline check passes.
2. IF any test issues an outbound network request or performs a paid action during the Runtime_Test_Gate run, THEN THE Knowgrph_System SHALL fail the run and report an offline-only violation.
3. THE Knowgrph_System SHALL exclude hardcoded absolute filesystem paths and demo-specific content, including fixed sample briefs and machine-specific identifiers, from all tests, and SHALL keep tests working-directory independent.
4. WHEN a Media_Provider generation step fails, THE Knowgrph_System SHALL apply a configured recovery action of retry, degrade, skip-with-evidence, or resume.
5. THE Knowgrph_System SHALL constrain the configured maximum recovery attempt count to the inclusive integer range 0 to 10.
6. WHERE the configured maximum recovery attempt count is zero, THE Knowgrph_System SHALL still apply non-retry recovery actions such as resume, degrade, or skip-with-evidence.
7. IF a generation step remains blocked after configured recovery attempts, THEN THE Knowgrph_System SHALL stop deterministically, preserve already-persisted artifacts, and record a blocker entry identifying the failed step and the failure reason.
8. WHEN two separate Runs use identical Mock_Provider inputs, THE Knowgrph_System SHALL produce byte-for-byte identical artifact references and identical content hashes across those Runs.
9. WHEN a Run is resumed after interruption, THE Knowgrph_System SHALL restore prior state from the D1_Store and continue without regenerating already-persisted artifacts.
10. IF restoring prior state from the D1_Store during resume fails, THEN THE Knowgrph_System SHALL report a restore-failure indication and SHALL NOT regenerate already-persisted artifacts.

### Requirement 9: Multi-Surface Runnability and Endpoint Access Control

**User Story:** As an operator, I want to run and inspect the pipeline from MCP, CLI, and the app UI with secured endpoints, so that the pipeline is usable across surfaces without exposing unauthenticated access.

#### Acceptance Criteria

1. WHEN a Run is initiated from the MCP_Surface, the CLI_Surface, or the App_UI_Surface, THE Knowgrph_System SHALL make the pipeline runnable and inspectable through that surface.
2. WHEN identical inputs are provided, THE Knowgrph_System SHALL produce the same final Run status, the same per-stage outcomes, and identical artifacts with identical Durable_R2_URLs and identical Provenance_Chain, regardless of which surface initiates the Run, except for surface identifiers, Run identifiers, and timestamps.
3. WHERE a network-exposed endpoint is provided, THE Knowgrph_System SHALL require a verified authentication credential and a Run-scoped authorization check before serving the request.
4. IF an unauthenticated request reaches a network-exposed endpoint, THEN THE Knowgrph_System SHALL deny the request, return an access-control error indicating that authentication is required, perform no Run action, and return no Run state or artifact data.
5. IF a request is authenticated but lacks authorization for the requested Run or resource, THEN THE Knowgrph_System SHALL deny the request with a distinct access-control error and disclose no Run state or artifact data.
6. WHEN a user inspects a Run through a supported surface, THE Knowgrph_System SHALL present Run state, the stage list with per-stage status, approval-gate states, budget-meter values, and artifact references through each supported surface.
