---
title: "knowgrph Mobile-First PWA — Requirements"
doc_type: "Requirements"
version: "1.1.0"
date: "2026-08-20"
lang: "en-US"
frontmatter_contract: "required"
owner: "solo-founder-ai-orchestrator"
local_rung: "spec-complete"
delivered_rung: "undocumented"
lane: "authoring"
universal_scope: "false"
lifecycle_status: "proposed"
source_prd: "joohwee/prd-tad-ard/knowgrph-mobile-first-pwa-prd-tad-adr.md"
governing_guidelines:
  - "huijoohwee.github.io/guidelines/agentic-sdlc-guidelines.md"
  - "agentic-canvas-os/docs/START-WORKFLOW.md"
---

# Requirements Document

## Introduction

This feature converts `knowgrph` from a browser-only, desktop-oriented surface into an installable,
offline-tolerant, touch-native Progressive Web App delivered through the existing
Authoring → Mirror → Delivery lane (`GitHub/knowgrph` → `GitHub/huijoohwee/content/knowgrph` →
Cloudflare `airvio.co/knowgrph`). Scope is derived from the source PRD/TAD/ADR document named in
frontmatter: Must-tier is a cached installable shell, an install affordance, and a local-first
read/write store with a bounded replay queue. Should-tier is iOS install guidance, touch
ergonomics, and a critical-path payload budget. Could-tier push re-engagement is deferred.

The feature adds no backend service, no provisioned runtime, and no paid dependency. It declares no
new `/`, `#`, or `@` invocation routes; the Replay_Queue consumes existing MCP-routed commands whose
harness contracts remain owned by `agentic-canvas-os/docs`.

This document owns the requirements phase only. Design and task decomposition are downstream
phases, and every requirement below carries a Verifiable Completion Condition so the downstream
task list can be derived rather than re-authored.

## Glossary

- **Shell_Cache**: The client-runtime component that stores and serves the application shell, route
  chunks, and static assets from the browser Cache Storage API (component C1 in the source TAD).
- **Install_Handler**: The component that publishes installation metadata and captures the deferred
  browser install prompt to present a first-party install affordance (component C2).
- **Local_Store**: The client-side persistent store holding last-synced graph records as the
  device-local source of truth (component C3).
- **Replay_Queue**: The persistent client-side queue that records offline write intents and replays
  them in order against the existing API surface on reconnect (component C4).
- **Install_Overlay**: The dismissible instructional surface that describes manual Add-to-Home-Screen
  steps on browsers with no native install prompt (component C5).
- **Touch_Layer**: The layout and interaction layer responsible for tap-target sizing, safe-area
  insets, and touch responsiveness.
- **Payload_Gate**: The build-time check that compares critical-path JavaScript against a stated
  budget and confirms route-level code splitting.
- **Delivery_Lane**: The ordered publication path Authoring → Mirror → Delivery, where Authoring is
  the local `knowgrph` development checkout, Mirror is `GitHub/huijoohwee/content/knowgrph`, and
  Delivery is Cloudflare `airvio.co/knowgrph`.
- **Authoring_Surface**: The first surface of the Delivery_Lane: the local `knowgrph` development
  checkout on which authoring and verification checks run.
- **Mirror_Surface**: The second surface of the Delivery_Lane: the mirror repository
  `GitHub/huijoohwee/content/knowgrph`.
- **Delivery_Surface**: The third surface of the Delivery_Lane: the published Cloudflare surface
  `airvio.co/knowgrph`.
- **Dependency_Manifest**: The recorded list of dependencies added by this feature, holding for each
  entry the dependency name, its pinned exact version, and its license identifier.
- **Invocation_Register**: The register of declared `/`, `#`, and `@` invocation routes, owned by
  `agentic-canvas-os/docs`.
- **Demo_Guide**: The authored walkthrough artifact `demo.md` in this spec directory that scripts a
  reproducible first-run and offline demonstration of the shipped feature.
- **Replay_Record**: A typed, serialized representation of one offline write intent, persisted by the
  Replay_Queue and reconstructible into the in-memory intent it was created from.
- **Conflict_Marker**: A typed state attached to a Replay_Record whose replay target changed since
  the local edit, requiring explicit user resolution.
- **Parked_State**: A typed terminal state attached to a Replay_Record after its retry bound is
  exhausted, visible to the user as needing attention.
- **Offline_Mode**: The condition in which the browser reports no network connectivity or every
  network request to the Delivery surface fails.
- **Standalone_Mode**: The condition in which the application is launched from an installed
  home-screen entry and reports display mode `standalone`.
- **FOSS_Dependency**: A runtime or build dependency distributed under an OSI-approved license with
  no per-seat, per-request, or subscription cost.
- **Evidence_Reference**: A named, invocable check paired with its recorded result and the surface it
  ran on, as defined by the governing agentic SDLC guidelines.

## Requirements

### Requirement 1: Cached Application Shell

**User Story:** As a mobile knowledge worker, I want the application shell and static assets cached on first visit, so that repeat visits and offline opens do not re-fetch the whole application.

#### Acceptance Criteria

1. WHEN a first online visit to the Delivery surface completes, THE Shell_Cache SHALL store the application shell document, the initial route chunks, and every static asset referenced by that document within 10 seconds of visit completion, and SHALL report the stored entry count as greater than zero.
2. WHILE Offline_Mode is active and at least one prior online visit has completed, THE Shell_Cache SHALL serve the cached application shell such that the shell reaches first render within 3 seconds with zero network responses received.
3. WHEN a new Shell_Cache version activates, THE Shell_Cache SHALL delete all cache entries belonging to superseded versions within 5 seconds of activation and before serving the next request, leaving zero superseded entries.
4. WHEN a request targets a read API route and a cached response for that route exists with an age of 24 hours or less, THE Shell_Cache SHALL serve the cached response within 1 second and revalidate that entry in the background.
5. IF a network fetch for a cached asset fails or does not complete within 5 seconds, THEN THE Shell_Cache SHALL serve the cached copy of that asset.
6. WHEN a user session ends by sign-out, THE Shell_Cache SHALL delete every cache entry keyed to that session's authorization scope within 5 seconds, leaving zero authorization-scoped entries, and SHALL retain shell and static asset entries that carry no authorization scope.
7. IF a network fetch fails and no cached copy of the requested resource exists, THEN THE Shell_Cache SHALL return a failure result indicating that the resource is unavailable offline and SHALL leave all existing cache entries unchanged.
8. IF storing new entries would exceed the available storage allowance, THEN THE Shell_Cache SHALL delete read API cache entries in oldest-first order until the store succeeds, SHALL retain the application shell document and initial route chunks, and SHALL report a storage-limit condition.

> **VCC**: `Verify the shell renders with the network adapter disabled after one prior online visit, that zero cache entries from a superseded version remain after activation, and that sign-out leaves zero authorization-scoped entries`

### Requirement 2: Installability

**User Story:** As a mobile knowledge worker, I want to install knowgrph to my home screen, so that it opens like a native application without browser chrome.

#### Acceptance Criteria

1. THE Install_Handler SHALL publish installation metadata declaring standalone display, a theme color, a background color, a launch URL scoped under the /knowgrph base path on the Delivery surface, and maskable icons at exactly 192x192 and 512x512 pixels, with every one of these fields present and non-empty.
2. WHEN the browser emits its deferred install prompt event, THE Install_Handler SHALL suppress the browser's default prompt, retain the event for the lifetime of the current document, and render a first-party install affordance that is visible and activatable within 1 second of the event.
3. WHEN the user activates the install affordance, THE Install_Handler SHALL invoke the retained prompt exactly once, record the outcome as either accepted or dismissed, discard the consumed event, and hide the affordance.
4. WHEN the application launches from an installed home-screen entry, THE Install_Handler SHALL report Standalone_Mode within 2 seconds of first render, and WHEN the application launches inside browser chrome, THE Install_Handler SHALL report a non-standalone display mode.
5. IF the browser emits no deferred install prompt event within 5 seconds of first render, THEN THE Install_Handler SHALL leave the install affordance hidden and SHALL NOT block any other application function.
6. IF the user dismisses the install prompt, THEN THE Install_Handler SHALL keep the affordance hidden for the remainder of the session and SHALL retain all existing application state unchanged.
7. IF invoking the retained prompt fails or the retained event is no longer valid, THEN THE Install_Handler SHALL hide the install affordance, present a message indicating installation is unavailable, and SHALL NOT reload or reset the application.
8. WHILE the application is running in Standalone_Mode, THE Install_Handler SHALL keep the install affordance hidden.

> **VCC**: `Verify an automated installability audit of the Delivery surface passes, the published metadata contains every named field, and a launch from the installed entry reports display mode standalone`

### Requirement 3: Local-First Read Path

**User Story:** As a mobile knowledge worker, I want previously synced graph content to render without a network, so that a dropped connection never blanks the screen.

#### Acceptance Criteria

1. WHEN a graph record of up to 1 MB is received from the existing API surface, THE Local_Store SHALL persist that record as a typed local record within 500 ms of receipt and SHALL confirm persistence to the caller.
2. WHILE Offline_Mode is active, WHEN a read is requested for a graph record that has a local copy, THE Local_Store SHALL return the last-synced copy of that record within 200 ms without issuing a network request.
3. IF a received payload does not satisfy the typed local record schema, or exceeds the 1 MB per-record limit, THEN THE Local_Store SHALL reject that payload, leave any previously persisted copy of that record unchanged, and return a typed validation error identifying the failed field or the exceeded limit.
4. IF a requested record has no local copy WHILE Offline_Mode is active, THEN THE Local_Store SHALL return a typed absent-record result within 200 ms that the interface renders as an explicit unavailable state naming the requested record.
5. WHEN the persisted schema version differs from the running schema version, THE Local_Store SHALL apply the declared migration for that version transition to completion before serving any read, and SHALL serve reads only after the migration reports success.
6. IF applying a declared migration fails, THEN THE Local_Store SHALL restore the pre-migration persisted state, return a typed migration error to the caller, and continue serving reads at the pre-migration schema version.
7. IF a persist attempt fails because the storage quota is exhausted, THEN THE Local_Store SHALL evict least-recently-read records until at least the space required by the incoming record is free, retry the persist once, and on a second quota failure return a typed storage-exhausted error while leaving existing records intact.
8. WHEN a record is evicted, THE Local_Store SHALL remove that record from the local set so that a subsequent read of it returns the typed absent-record result, and SHALL retain at most 5,000 records or 50 MB of typed local records, whichever limit is reached first.

> **VCC**: `Verify a read of a previously synced record succeeds with the network adapter disabled, an unsynced record returns the typed absent-record result, a malformed payload returns a typed validation error, and a version bump applies its declared migration`

### Requirement 4: Offline Write Queue and Bounded Replay

**User Story:** As a mobile knowledge worker, I want edits made offline to persist and replay when connectivity returns, so that no edit is lost and no edit silently overwrites newer state.

#### Acceptance Criteria

1. WHILE Offline_Mode is active, WHEN the user commits an edit, THE Replay_Queue SHALL persist a Replay_Record for that edit within 1 second and THE Local_Store SHALL apply the edit locally so that it is readable by the next read of that target.
2. THE Replay_Queue SHALL retain at most 500 pending Replay_Records, counting records carrying a Conflict_Marker or a Parked_State.
3. IF the user commits an edit WHILE the Replay_Queue holds 500 pending Replay_Records, THEN THE Replay_Queue SHALL reject the new Replay_Record, leave the existing pending set unchanged, and surface an indication that the offline queue is full.
4. WHEN connectivity returns, THE Replay_Queue SHALL begin replay within 5 seconds and SHALL replay pending Replay_Records in the order they were persisted, replaying no more than one Replay_Record per target concurrently so that records touching the same target replay in persisted order.
5. WHEN a Replay_Record replay returns a success response, THE Replay_Queue SHALL remove that record and record the replay result with its completion time.
6. IF a replay target changed since the local edit, THEN THE Replay_Queue SHALL attach a Conflict_Marker to that Replay_Record, leave the remote state unchanged, retain the local edit in the Local_Store, and continue with the next record.
7. IF a replay attempt fails for a transport or server reason, THEN THE Replay_Queue SHALL retry that record after delays of 1, 2, 4, and 8 seconds, up to a maximum of 5 attempts in total per record.
8. IF a Replay_Record reaches 5 failed attempts, THEN THE Replay_Queue SHALL attach a Parked_State to that record, surface it as needing attention, and continue with the next record.
9. WHEN the user selects a Replay_Record carrying a Conflict_Marker or a Parked_State and chooses to retry or discard it, THE Replay_Queue SHALL either reset that record to 0 failed attempts and requeue it at the end of the pending set, or remove that record from the pending set, according to the choice made.
10. WHERE the browser provides no background synchronization capability, THE Replay_Queue SHALL begin replay on the next application foreground event.
11. IF replay of the pending set is interrupted by loss of connectivity or application termination, THEN THE Replay_Queue SHALL retain every Replay_Record that has not received a success response, preserve its persisted order and attempt count, and resume replay from the earliest such record on the next replay start.
12. WHEN a Replay_Record is replayed more than once for the same edit, THE Replay_Queue SHALL produce the same resulting remote state as a single replay of that record.
13. FOR ALL Replay_Records, THE Replay_Queue SHALL reconstruct the original write intent from its persisted serialization, so that serializing then reading then serializing yields an equivalent Replay_Record (round-trip property).
14. WHEN replay of the pending set completes, THE Replay_Queue SHALL hold zero records other than those carrying a Conflict_Marker or a Parked_State.

> **VCC**: `Verify an offline edit persists a Replay_Record, reconnect drains the pending set in persisted order to zero non-flagged records with a recorded result per record, a changed target yields a Conflict_Marker with unchanged remote state, the fifth consecutive failure yields Parked_State, a repeated replay yields the same remote state, and a round-trip property over generated Replay_Records holds`

### Requirement 5: Manual Install Guidance

**User Story:** As a browser user whose browser provides no native install prompt, I want explicit Add-to-Home-Screen instructions, so that I can install the application anyway.

#### Acceptance Criteria

1. WHERE the browser provides no deferred install prompt event, WHILE the application is not in Standalone_Mode, WHEN 3 seconds have elapsed since the application became interactive, THE Install_Overlay SHALL become visible within 500 milliseconds and present the manual Add-to-Home-Screen steps for the detected browser as an ordered list of 2 to 6 steps.
2. WHEN the user dismisses the Install_Overlay, THE Install_Overlay SHALL become hidden within 300 milliseconds and SHALL remain hidden for the remainder of that browser session, including across in-session navigations and reloads, and SHALL become eligible to appear again in a new browser session.
3. WHILE the application is in Standalone_Mode, THE Install_Overlay SHALL remain hidden and SHALL NOT become visible for any trigger.
4. THE Install_Overlay SHALL expose a dismiss control that is reachable by keyboard within 3 sequential focus moves from the point at which the Install_Overlay receives focus, that carries a non-empty accessible name, and that is activated by both Enter and Space.
5. WHEN the Install_Overlay becomes visible, THE Install_Overlay SHALL move keyboard focus to the Install_Overlay, SHALL confine sequential focus moves to controls inside the Install_Overlay while visible, and WHEN the Install_Overlay becomes hidden, THE Install_Overlay SHALL return focus to the element focused immediately before it became visible.
6. WHEN the user presses the Escape key while the Install_Overlay is visible, THE Install_Overlay SHALL dismiss with the same behavior as criterion 2.
7. IF the browser cannot be matched to a known set of Add-to-Home-Screen steps, THEN THE Install_Overlay SHALL present browser-agnostic Add-to-Home-Screen steps of 2 to 6 steps and SHALL indicate that the steps are generic, rather than remaining empty or hidden.

> **VCC**: `Verify the overlay renders only in the no-native-prompt and non-standalone condition, stays hidden in standalone mode and after dismissal within the same session, and exposes a keyboard-reachable named dismiss control`

### Requirement 6: Touch Ergonomics

**User Story:** As a mobile user, I want tap targets, safe areas, and scrolling tuned for touch, so that the application feels native rather than like a shrunk desktop page.

#### Acceptance Criteria

1. WHILE the viewport width is between 320 and 430 CSS pixels, THE Touch_Layer SHALL render every interactive element with a hit area of at least 44 by 44 CSS pixels and at least 8 CSS pixels of separation from any adjacent interactive element.
2. THE Touch_Layer SHALL apply safe-area inset values to the root layout containers so that no interactive element or text content renders within the device cutout region or within the bottom 34 CSS pixels reserved for the home-indicator region.
3. WHEN a user taps an interactive element, THE Touch_Layer SHALL dispatch the resulting action within 100 milliseconds of touch release, with no double-tap-to-zoom activation delay applied.
4. THE Touch_Layer SHALL express interactive controls as native semantic elements, each carrying a non-empty accessible name of 1 to 100 characters.
5. WHEN the device orientation changes, THE Touch_Layer SHALL re-apply safe-area insets and target sizing within 500 milliseconds without loss of scroll position or entered form input.
6. WHILE a scrollable region is being dragged, THE Touch_Layer SHALL sustain frame presentation at or above 50 frames per second and SHALL confine scrolling to the dragged region without scrolling the page behind it.
7. IF the user agent reports a reduced-motion preference or a text scale factor of up to 200 percent, THEN THE Touch_Layer SHALL suppress non-essential transition animation and SHALL keep all interactive elements reachable and free of clipped or overlapping content.

> **VCC**: `Verify an automated audit reports zero interactive elements below 44 by 44 CSS pixels, root layout containers resolve non-zero-capable safe-area inset values, and zero interactive controls lack an accessible name`

### Requirement 7: Critical-Path Payload Budget

**User Story:** As a mobile user on a metered connection, I want the initial payload minimized, so that first load stays fast on cellular.

#### Acceptance Criteria

1. WHEN a production build completes, THE Payload_Gate SHALL record the total critical-path JavaScript compressed transfer size, measured as brotli-compressed bytes of every JavaScript asset requested during initial page load, within 60 seconds of build completion.
2. IF the recorded critical-path JavaScript compressed transfer size exceeds 180 KB (184,320 brotli-compressed bytes), THEN THE Payload_Gate SHALL fail the build check and report both the recorded compressed size in bytes and the 180 KB budget.
3. WHEN a production build completes, THE Payload_Gate SHALL confirm that at least one route-level chunk is requested on demand after initial load and is absent from the initial payload, and SHALL fail the build check when no such chunk is found.
4. WHEN the build check completes, THE Payload_Gate SHALL write the recorded compressed transfer size, the 180 KB budget, the pass or fail outcome, and the identified on-demand route-level chunk to an Evidence_Reference retained for the build.
5. IF one or more critical-path JavaScript assets cannot be measured, THEN THE Payload_Gate SHALL fail the build check, report an error indicating which assets could not be measured, and SHALL NOT report a partial size as the recorded size.
6. WHERE the active lane is the Authoring lane, THE Payload_Gate SHALL run the build check; in all other lanes THE Payload_Gate SHALL skip the check and record a skipped outcome in the Evidence_Reference.

> **VCC**: `Verify the build check reports the critical-path JavaScript transfer size, fails when that size exceeds the stated budget, and identifies at least one on-demand route-level chunk absent from the initial payload`

### Requirement 8: Zero-Infrastructure, FOSS, and Cost Posture

**User Story:** As the solo founder operating this product, I want the feature to add no paid service, no provisioned runtime, and no model spend, so that monthly total cost of ownership stays at zero.

#### Acceptance Criteria

1. THE knowgrph PWA feature SHALL add zero server-side runtime components beyond the existing static Delivery surface, where a server-side runtime component is any process, function, container, scheduled job, or managed service that executes outside the user's device.
2. THE knowgrph PWA feature SHALL declare every added runtime and build dependency as a FOSS_Dependency in a Dependency_Manifest, recording for each entry the dependency name, pinned exact version, and OSI-approved license identifier.
3. IF an added runtime or build dependency carries a license that is not OSI-approved or whose license cannot be determined, THEN THE knowgrph PWA feature SHALL fail the build with an error indicating the offending dependency and its license status, and SHALL exclude that dependency from the Dependency_Manifest.
4. THE Replay_Queue SHALL complete a full replay cycle of up to 500 Replay_Record entries with zero model invocations attributable to the Replay_Queue itself.
5. WHEN a Replay_Record targets an existing MCP-routed command, THE Replay_Queue SHALL forward that record to the existing MCP dispatcher within 1 second of dequeue and SHALL add zero entries to the Invocation_Register.
6. THE knowgrph PWA feature SHALL keep every persisted user record in the Local_Store on the user's device, and SHALL transmit a record to the existing API surface only as part of an explicit replay initiated by user action or by the Replay_Queue on reconnection.
7. THE knowgrph PWA feature SHALL issue zero network requests to third-party telemetry, analytics, advertising, or crash-reporting endpoints, permitting outbound requests only to the existing Delivery surface and the existing API surface.
8. THE knowgrph PWA feature SHALL add 0.00 USD to the monthly recurring cost of the product, measured as the sum of charges for any newly introduced hosting, storage, model-inference, or third-party service line item over any 30-day billing period.

> **VCC**: `Verify the Dependency_Manifest lists an OSI-approved license for every added dependency, the feature adds zero server-side runtime components, a full replay cycle records zero attributable model invocations, and the Invocation_Register gains zero entries`

### Requirement 9: Delivery Lane and Boundary Discipline

**User Story:** As the operator of the publication path, I want the feature to ship through the existing Authoring → Mirror → Delivery lane under explicit gates, so that no automated step publishes without my decision.

#### Acceptance Criteria

1. THE knowgrph PWA feature SHALL reach the Delivery surface only through the Authoring → Mirror → Delivery order of the Delivery_Lane, and SHALL treat any arrival on Delivery without a preceding recorded Mirror crossing as a lane violation.
2. WHILE the feature is in the Authoring lane, THE build and verification checks SHALL run against the Authoring_Surface only, leaving the Mirror_Surface and Delivery_Surface byte-identical to their state before the checks started.
3. WHERE a lane boundary crossing is requested, THE Delivery_Lane SHALL require a recorded operator decision naming the exact candidate revision, the source surface, and the target surface before that crossing proceeds.
4. IF a lane boundary crossing is requested without a recorded operator decision naming the exact candidate revision, THEN THE Delivery_Lane SHALL block the crossing, leave the target surface unchanged, and report an error indicating that the operator decision is absent.
5. IF a published Delivery revision fails its verification checks, THEN THE Delivery_Lane SHALL restore the prior Delivery revision through the recorded rollback action within 15 minutes of the failing result being recorded, and SHALL record the restored revision identifier.
6. WHEN a Delivery rollback completes, THE Shell_Cache SHALL replace its contents with the restored revision's assets and serve the restored revision to every subsequent load, discarding assets of the rolled-back revision.
7. THE knowgrph PWA feature SHALL record one Evidence_Reference per stated VCC, naming the invocable check, the recorded result, and the surface that check ran on, where that surface is exactly one of Authoring_Surface, Mirror_Surface, or Delivery_Surface.
8. IF an Evidence_Reference names the Authoring_Surface, THEN THE knowgrph PWA feature SHALL label the claim as unverified on Delivery and SHALL NOT count that Evidence_Reference toward a Delivery verification claim.

> **VCC**: `Verify every authoring check leaves the Mirror_Surface and Delivery_Surface unchanged, each boundary register row records an operator decision and a rollback action, and one Evidence_Reference exists per stated VCC`

### Requirement 10: Demonstration Walkthrough Artifact

**User Story:** As the operator validating time-to-value, I want a scripted demonstration walkthrough, so that the first-run and offline behavior can be reproduced and timed on a clean device.

#### Acceptance Criteria

1. THE Demo_Guide SHALL state the ordered steps, numbered 1 through at most 4, that take a mobile browser with no prior Shell_Cache, Local_Store, or Replay_Queue entries for the Delivery_Surface origin from opening the Delivery_Surface to reopening the installed application in Standalone_Mode with the network disabled.
2. THE Demo_Guide SHALL state, for each numbered step, the observable outcome as a screen-visible or device-visible result that a reader can judge pass or fail without inspecting source code.
3. THE Demo_Guide SHALL state the reset preconditions that establish a clean device, covering removal of any installed application instance and clearing of Shell_Cache, Local_Store, and Replay_Queue entries for the Delivery_Surface origin.
4. THE Demo_Guide SHALL state the measured step count and the measured elapsed time in minutes and seconds, from first request of the Delivery_Surface to the reopened application rendering in Standalone_Mode, against the ceilings of 4 steps and 3 minutes on a connection throttled to slow cellular.
5. THE Demo_Guide SHALL state the throttling profile used for the measurement in downlink throughput and round-trip latency terms, and SHALL state that the same profile applies to every timed run.
6. THE Demo_Guide SHALL state an ordered step sequence that performs a write while in Offline_Mode, shows the resulting Replay_Record in the Replay_Queue, restores the network, and shows the Replay_Record leaving the Replay_Queue.
7. THE Demo_Guide SHALL name the invocable check behind each demonstrated requirement so that each demonstrated claim maps to exactly one Evidence_Reference.
8. IF a measured step count exceeds 4 steps or a measured elapsed time exceeds 3 minutes, THEN THE Demo_Guide SHALL record the measured value, mark the run as failing its ceiling, and retain the recorded measurement rather than replacing it with a passing run.
9. IF a demonstrated step depends on a browser capability the demonstration device lacks, THEN THE Demo_Guide SHALL name the missing capability and state the degraded path for that step, including the observable outcome of that degraded path.
10. IF a demonstrated requirement has no invocable check, THEN THE Demo_Guide SHALL mark that claim as unverified and state the manual observation used in place of an Evidence_Reference.

> **VCC**: `Verify the Demo_Guide exists in this spec directory, contains an ordered step list with an observable outcome per step, records a step count of at most 4 and an elapsed time of at most 3 minutes, and names an invocable check per demonstrated requirement`

## Out of Scope

- Native application store distribution wrappers.
- Push re-engagement notifications and any relay component supporting them.
- Server-authoritative concurrent-edit merging; this increment surfaces conflicts explicitly and
  merges nothing.
- Cross-device write coalescing; the Replay_Queue is single-device.
- Full background-synchronization parity on browsers lacking that capability; the foreground retry
  path in Requirement 4 is the accepted degraded mode.
- New Agentic OS, AI Agent discovery, or MCP Gateway federation capability for knowgrph surfaces;
  those dimensions and the invocation register remain owned by `agentic-canvas-os/docs`.
- Any `GameXR` PWA layer.

## Assumptions

- The existing Cloudflare Pages delivery lane for `airvio.co/knowgrph` and the Mirror repository
  remain available and unchanged in structure.
- The existing API surface and its JSON schemas are unchanged by this feature.
- The existing MCP dispatcher accepts forwarded command intents without a new route declaration.
- Success-metric baselines for mobile retention, install rate, and offline failure rate are not yet
  measured; targets stated in the source PRD are tracked there, not verified by this document.

## Open Questions

- Which browser and device matrix must the Requirement 6 tap-target audit and the Requirement 10
  demonstration cover?
