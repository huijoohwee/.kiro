# Flow Editor Run Button Regression — Bugfix Design

## Overview

The Flow Editor's widget card action buttons have regressed: clicking "Run" (or any non-download
action button) triggers a `*.json` file download containing serialized node/graph frontmatter
instead of dispatching the button's declared `trigger` to the action handler layer.

The fix is fully generic. It operates at the canonical action dispatcher layer and the widget card
button renderer — neither layer may contain hardcoded trigger names, node IDs, or widget types.
Every action button is driven entirely by its `trigger` field value sourced from
`canvas:widgetCard.actions[]`. The download path (Blob URL + `<a download>`) is structurally
removed from all non-download trigger flows, not masked by a downstream patch.

Renderer context: `kgCanvas2dRenderer: "flowEditor"` (2D Flow Editor).
Validation anchor: `knowgrph-flow-editor-computing-flow-template.md`.

---

## Glossary

- **Bug_Condition (C)**: The condition that triggers the bug — a widget card action button whose
  `trigger` field is NOT a download-type trigger is clicked, yet the browser initiates a `*.json`
  file download instead of dispatching the declared action.
- **Property (P)**: The desired post-fix behavior — for any clicked action button, only the handler
  keyed to `button.trigger` is invoked; no file download, no default browser navigation occurs.
- **Preservation**: All existing behaviors for non-buggy paths (Edit, Reset, downstream compute
  invocation, side effects, button rendering) that must remain unchanged by this fix.
- **`canvas:widgetCard.actions[]`**: The data-driven list of action button descriptors on a widget
  card node. Each entry has at minimum `{ id, label, trigger }`. The `trigger` field is the sole
  dispatch key — no other field drives handler selection.
- **Action Dispatcher**: The shared, headless, unopinionated layer that receives a `(trigger, context)`
  pair and routes it to the correct handler function via a generic dispatch table. It MUST NOT
  contain hardcoded conditionals per trigger name or node type.
- **Dispatch Table**: A data-driven map `Record<string, ActionHandler>` populated at registration
  time. The dispatcher looks up `trigger` in the table and calls the resolved function. Unknown
  triggers are handled by a registered fallback or error handler — never silently routed to download.
- **Download-type trigger**: A trigger explicitly declared as producing a file download (e.g.,
  `trigger: "download"` with an explicit `mimeType` / `filename` in the action descriptor). Only
  these triggers may ever produce a file download. All others are structurally blocked from the
  download path.
- **`buildScopedGraphSemanticKey`**: The existing shared semantic-key helper used for scoping
  action contexts to the correct graph/node. Reused as-is; not modified by this fix.
- **`frontmatter:primitive`**: Node-level field (`"node"` or `"graph"`) that identifies the
  frontmatter scope for a given node. Referenced for context construction; never used as a dispatch
  discriminator.

---

## Bug Details

### Bug Condition

The bug manifests when any widget card action button whose `trigger` value is not a download-type
trigger is clicked. The system incorrectly produces a `*.json` file download. Two structural
candidate paths explain the failure:

**Candidate A — Parent `<a download>` propagation:**
The action button is rendered as a child (or descendant) of an `<a>` element that has a `download`
attribute and a JSON Blob `href`. When the button is clicked, the event bubbles up to the anchor,
and the browser's native download behavior fires. Neither `event.preventDefault()` nor
`event.stopPropagation()` is called at the button handler level, so propagation is not blocked.

**Candidate B — Dispatcher Blob URL branch:**
The action dispatcher contains a code path that serializes the node or graph frontmatter into a
JSON Blob, constructs a temporary object URL, and programmatically triggers `<a download>` click
navigation. This branch fires for all triggers (or for triggers it fails to match against a
handler), meaning non-download triggers fall through into the Blob download path instead of
resolving to their registered handlers.

In either case, the structural point of failure is the same: **the download path is reachable from
a non-download trigger click**. The fix must make this structurally impossible.

**Formal Specification:**

```
FUNCTION isBugCondition(action, event, result)
  INPUT:
    action  — an entry from canvas:widgetCard.actions[]
    event   — the DOM click event fired when the button is clicked
    result  — the observable outcome after the click is processed
  OUTPUT: boolean

  RETURN action.trigger NOT IN downloadTypeTriggerSet(action)
         AND result.fileDownloadInitiated = true
END FUNCTION

FUNCTION downloadTypeTriggerSet(action)
  // Only triggers with an explicit download declaration in their descriptor
  RETURN { t | action.trigger = t AND action.mimeType IS DEFINED AND action.filename IS DEFINED }
END FUNCTION
```

### Examples

| Widget | Button | `trigger` | Current (buggy) | Expected (correct) |
|--------|--------|-----------|----------------|--------------------|
| `source_input` | Run | `runDownstream` | `*.json` file download | dispatches `runDownstream` → targets `["compute_summary"]` |
| `compute_summary` | Run | `compute` | `*.json` file download | dispatches `compute` on widget |
| `source_input` | Edit | `openFieldEditor` | (would download if bug persists) | opens field editor for `input_query` |
| `compute_summary` | Reset | `clearOutputs` | (would download if bug persists) | clears `output`, `imageUrl`, `outputSrcDoc` |
| Any node | Any download button | `download` (with `mimeType`+`filename`) | file download | file download ✓ (unchanged) |

---

## Expected Behavior

### Preservation Requirements

**Unchanged Behaviors:**

- Edit button (`trigger: "openFieldEditor"`) on `source_input` MUST continue to open the field
  editor for `input_query` without any regression.
- Reset button (`trigger: "clearOutputs"`) on `compute_summary` MUST continue to clear
  `output`, `imageUrl`, and `outputSrcDoc` fields without regression.
- When `runDownstream` is dispatched, the system MUST continue to invoke the compute function
  on target nodes and update their output fields as defined in `canvas:runAction`.
- When `compute` completes successfully, the side effects `run_status → "done"`,
  `template_flow_demo.active_graph_mutated → true`, and `template_flow_demo.run_id` (pattern
  `kgcf_run_yyyyMMddHHmm`) MUST continue to be applied.
- Widget card action buttons MUST continue to render with correct labels, icons, primary styling,
  and ordering as declared in `canvas:widgetCard.actions[]`.
- Any action button whose descriptor explicitly declares a download-type trigger (with `mimeType`
  and `filename`) MUST continue to produce a file download.
- The `buildScopedGraphSemanticKey` helper MUST be reused without modification for context scoping.

**Scope:**

All button interactions that do NOT satisfy `isBugCondition` (i.e., inputs where the trigger is
already dispatching correctly, or where the trigger is a legitimate download-type) are completely
unaffected by this fix. The fix is scoped exclusively to the action dispatcher and the button
renderer's event handling contract.

---

## Hypothesized Root Cause

Based on the bug description and the two candidate structural paths:

1. **`<a download>` DOM wrapping (Candidate A — most likely primary cause):**
   The widget card or node card renderer wraps the entire card (or its action area) in an
   `<a href={blobUrl} download={filename}>` element to support node/frontmatter export. Action
   buttons rendered inside this anchor inherit its click propagation. Since no
   `event.stopPropagation()` is called on the button click handler, the event bubbles to the
   anchor, triggering the download. This is the most common form of this regression in React/DOM
   canvas renderers where export links are added at the card container level.

2. **Dispatcher fallthrough to Blob download branch (Candidate B — secondary candidate):**
   The action dispatcher has a code path — possibly introduced as a "serialize node to JSON" export
   feature — that constructs a Blob from the node's frontmatter and triggers a programmatic
   `<a download>` click. If the trigger lookup fails (missing registration, case mismatch, stale
   trigger name) or if the dispatch table is not consulted before the Blob path, all triggers fall
   through to this download behavior. A stale or legacy trigger name remapping may also cause
   valid triggers to be unresolved in the table.

3. **Missing `event.preventDefault()` at button handler registration:**
   Even if Candidate A is the primary cause, the absence of `preventDefault()` compounds the
   problem. Any anchor-level default navigation behavior would also need to be blocked.

4. **Legacy/stale trigger name aliases in dispatch table:**
   A backward-compatibility remapping (e.g., mapping `"run"` → old download handler, or treating
   unrecognized triggers as "export node") could cause `runDownstream` and `compute` to resolve
   to the download branch even if the dispatch table is otherwise correct.

---

## Correctness Properties

Property 1: Bug Condition — Non-Download Trigger Clicks Never Produce a File Download

_For any_ action in `canvas:widgetCard.actions[]` where `isBugCondition` holds (the trigger is
not a download-type trigger), clicking the rendered button SHALL dispatch only the handler
registered for `action.trigger` in the generic dispatch table. The fixed system SHALL call
`event.preventDefault()` and `event.stopPropagation()` on the click event before handler
resolution, and SHALL NOT initiate a `*.json` (or any) file download, construct a Blob URL,
or invoke any `<a download>` navigation as a side effect of the dispatch.

**Validates: Requirements 1.3, 2.1, 2.2, 2.3**

---

Property 2: Preservation — Non-Buggy Action Paths Unchanged

_For any_ button click where `isBugCondition` does NOT hold (i.e., the trigger is a legitimate
download-type trigger, OR the action is being tested on already-correct code), the fixed system
SHALL produce exactly the same observable behavior as the original correct behavior, preserving all
existing functionality: field editor opening, output clearing, downstream compute invocation, side
effect application, and button rendering.

**Validates: Requirements 3.1, 3.2, 3.3, 3.4, 3.5**

---

## Fix Implementation

### Principle

The fix operates at exactly two layers — no others:

1. **Button renderer event handler** — ensures `preventDefault` + `stopPropagation` are called
   unconditionally on every action button click before dispatching.
2. **Action dispatcher** — ensures the Blob/download path is structurally unreachable for
   non-download triggers; the dispatch table is the sole routing mechanism.

No widget-specific patches. No hardcoded trigger names. No node-type conditionals.

---

### Changes Required

**Layer 1: Widget Card Button Renderer**

File: the shared widget card button renderer component (2D Flow Editor renderer layer).

**Specific Changes:**

1. **Remove `<a download>` wrapper from action button area:**
   If the card container or action button area is wrapped in `<a href={blobUrl} download>`,
   restructure the DOM so action buttons are never descendants of that anchor. The download
   anchor (if needed for legitimate export) MUST be either:
   - a sibling element, not a parent/ancestor of action buttons, OR
   - rendered only when a `download`-type action button is explicitly present in `actions[]`
     and is itself the download anchor, not a wrapper.

2. **Unconditional event guard on every action button click handler:**
   ```
   FUNCTION handleActionButtonClick(event, action, dispatchContext)
     event.preventDefault()
     event.stopPropagation()
     actionDispatcher.dispatch(action.trigger, action, dispatchContext)
   END FUNCTION
   ```
   This guard MUST be applied to every button rendered from `canvas:widgetCard.actions[]`,
   regardless of trigger value. It is not conditional on trigger type.

3. **Button rendered as `<button type="button">`, never as `<a>` or descendant of `<a download>`:**
   Action buttons in `canvas:widgetCard.actions[]` MUST be rendered as `<button type="button">`
   elements (or equivalent non-anchor interactive elements). Using `type="button"` explicitly
   prevents form submission default behavior as a secondary guard.

---

**Layer 2: Action Dispatcher**

File: the shared, headless action dispatch module (canonical, shared across all renderer contexts).

**Specific Changes:**

4. **Remove the Blob URL / `<a download>` branch from the non-download dispatch path:**
   Any code path in the dispatcher that:
   - constructs a `new Blob(...)` from node/graph frontmatter,
   - creates an object URL via `URL.createObjectURL(...)`,
   - appends a temporary `<a download>` element to the DOM,
   - or calls `.click()` on such an element
   
   MUST be fully removed from the general dispatch flow. This path is only permissible inside
   a handler explicitly registered for a download-type trigger (i.e., a handler in the dispatch
   table registered under a trigger descriptor that declares `mimeType` and `filename`).

5. **Generic dispatch table — data-driven, no hardcoded conditionals:**
   The dispatcher resolves handlers via a lookup-only table:
   ```
   FUNCTION dispatch(trigger, action, context)
     handler := dispatchTable.get(trigger)
     IF handler IS UNDEFINED THEN
       fallbackHandler(trigger, action, context)  // logs warning, does NOT download
       RETURN
     END IF
     handler(action, context)
   END FUNCTION
   ```
   The table MUST be populated at registration time by the feature layer, not by hardcoded
   switch/if blocks inside the dispatcher. The dispatcher is unopinionated about trigger semantics.

6. **Remove all legacy trigger name aliases and backward-compatibility remappings:**
   Any entry in the dispatch table (or pre-dispatch transformation) that:
   - remaps a trigger name to a legacy name,
   - treats an unrecognized trigger as "export" or "download",
   - or falls through to a Blob download for unknown triggers
   
   MUST be removed entirely. No aliases. No remapping. Unrecognized triggers go to
   `fallbackHandler`, which logs a warning and returns without side effects.

7. **Context construction via `buildScopedGraphSemanticKey`:**
   When constructing the dispatch context, the existing `buildScopedGraphSemanticKey` helper
   MUST be used to derive the scoped graph/node identity for the action. No inline key
   construction. The context object passed to the handler includes the scoped key, the action
   descriptor, and the node's frontmatter reference — but the dispatcher never serializes
   frontmatter to a Blob or file.

---

### Cleanup Spec — Legacy/Stale Code to Remove (100% Removal)

The following MUST be removed with no backward-compatibility aliases left in place:

| Category | What to Remove |
|----------|---------------|
| Blob serialization | Any `new Blob([JSON.stringify(node)])` or `new Blob([JSON.stringify(frontmatter)])` calls in the dispatcher or card renderer outside of an explicitly registered download handler |
| Object URL construction | Any `URL.createObjectURL(...)` calls in the dispatcher or card renderer outside of an explicitly registered download handler |
| Programmatic `<a download>` injection | Any code that creates a temporary `<a>` element, sets `download` + `href`, appends it to the DOM, calls `.click()`, and then removes it — outside of an explicitly registered download handler |
| `<a download>` card wrapper | The wrapping of action button areas (or entire card containers) in `<a href={blobUrl} download>` elements |
| Legacy trigger name remappings | Any `trigger → oldTriggerName` alias tables, compatibility shims, or fallthrough-to-download paths for unrecognized triggers |
| Hardcoded trigger conditionals | Any `if (trigger === "runDownstream")`, `if (trigger === "compute")`, `switch(trigger)` blocks inside the shared dispatcher |
| Hardcoded node-type conditionals | Any `if (nodeType === "InputWidget")` or `if (nodeType === "ComputeWidget")` blocks inside the shared dispatcher or button renderer |

---

## Testing Strategy

### Validation Approach

Two-phase approach:

1. **Exploratory / Bug Condition Checking** — run tests on UNFIXED code to surface counterexamples
   that confirm the root cause path (Candidate A, B, or both).
2. **Fix Checking + Preservation Checking** — run tests on FIXED code to verify Property 1 and
   Property 2 hold universally.

Runtime-ready validation document: `knowgrph-flow-editor-computing-flow-template.md`
(renderer: `kgCanvas2dRenderer: "flowEditor"`, template `template_flow_demo`).

---

### Exploratory Bug Condition Checking

**Goal:** Surface counterexamples demonstrating the bug on unfixed code. Confirm or refute
root cause hypotheses. If Candidate A is refuted (no `<a download>` ancestor found), escalate
investigation to Candidate B (dispatcher Blob branch). If both are refuted, re-hypothesize.

**Test Plan:** Simulate click events on action buttons rendered for `source_input` and
`compute_summary` nodes from `knowgrph-flow-editor-computing-flow-template.md`. Assert whether
a file download is initiated and/or whether the expected dispatch call is made.

**Test Cases:**

1. **`source_input` Run button — unfixed code** (EXPECTED TO FAIL):
   Simulate click on the `run` button (`trigger: "runDownstream"`) of `source_input`.
   Assert: no file download initiated, `runDownstream` dispatched to `["compute_summary"]`.
   On unfixed code: file download is initiated — confirms bug condition.

2. **`compute_summary` Run button — unfixed code** (EXPECTED TO FAIL):
   Simulate click on the `run` button (`trigger: "compute"`) of `compute_summary`.
   Assert: no file download initiated, `compute` dispatched on widget.
   On unfixed code: file download is initiated — confirms bug condition.

3. **DOM ancestor inspection — Candidate A probe:**
   For each rendered action button in a widget card, traverse `parentElement` chain and assert
   no ancestor has `tagName === "A"` with a `download` attribute set.
   On unfixed code: if an `<a download>` ancestor is found, Candidate A is confirmed.

4. **Dispatcher Blob branch probe — Candidate B probe:**
   Spy on `URL.createObjectURL` and `Blob` constructor in the dispatcher module.
   Simulate click on a non-download action button.
   Assert: neither `Blob` constructor nor `URL.createObjectURL` is called during dispatch.
   On unfixed code: if called, Candidate B is confirmed.

**Expected Counterexamples:**
- File download is initiated for `runDownstream` and `compute` triggers.
- Possible causes confirmed by probes: `<a download>` ancestor (Candidate A) and/or
  `Blob`/`createObjectURL` called in dispatcher (Candidate B).

---

### Fix Checking

**Goal:** Verify that for all inputs where `isBugCondition` holds (non-download trigger clicks),
the fixed function dispatches the correct handler and produces NO file download.

**Pseudocode:**
```
FOR ALL action IN canvas:widgetCard.actions[]
  WHERE action.trigger NOT IN downloadTypeTriggerSet(action)
DO
  event := simulateClick(buttonFor(action))
  result := observeOutcome(event)
  ASSERT result.fileDownloadInitiated = false
  ASSERT result.dispatchedTrigger = action.trigger
  ASSERT result.eventDefaultPrevented = true
  ASSERT result.eventPropagationStopped = true
END FOR
```

**Test Cases:**

1. **`runDownstream` dispatched correctly after fix:**
   Click `source_input` Run → assert `runDownstream` dispatched with `targets: ["compute_summary"]`,
   no Blob created, no download initiated.

2. **`compute` dispatched correctly after fix:**
   Click `compute_summary` Run → assert `compute` dispatched, no download initiated.

3. **`openFieldEditor` dispatched correctly after fix:**
   Click `source_input` Edit → assert `openFieldEditor` dispatched for `targetField: "input_query"`,
   no download initiated.

4. **`clearOutputs` dispatched correctly after fix:**
   Click `compute_summary` Reset → assert `clearOutputs` dispatched with
   `clearFields: ["output","imageUrl","outputSrcDoc"]`, no download initiated.

5. **Generic property — all non-download triggers across arbitrary widget card configurations:**
   Generate action descriptors with varied trigger values (non-download). For each, assert
   Property 1 holds. No hardcoded trigger names in the test generator.

---

### Preservation Checking

**Goal:** Verify that for all inputs where `isBugCondition` does NOT hold, the fixed function
produces exactly the same behavior as the original correct behavior.

**Pseudocode:**
```
FOR ALL action IN canvas:widgetCard.actions[]
  WHERE isBugCondition(action, event, result) = false
DO
  result_original := observeOutcome_originalCorrectBehavior(action)
  result_fixed    := observeOutcome_fixed(action)
  ASSERT result_original = result_fixed
END FOR
```

**Testing Approach:** Property-based testing is used for preservation checking because:
- It generates many varied action descriptors automatically, covering the full domain.
- It catches edge cases (unusual trigger names, empty action arrays, missing optional fields)
  that manual unit tests would miss.
- It provides strong guarantees that the dispatcher and renderer are neutral with respect to
  trigger semantics.

**Test Cases:**

1. **Downstream compute chain preservation:**
   After fix, dispatch `runDownstream` → assert `compute` is invoked on `compute_summary`,
   output fields (`output`, `imageUrl`, `outputSrcDoc`) are updated.

2. **Side effect preservation:**
   After fix, `compute` completion → assert `run_status = "done"`,
   `template_flow_demo.active_graph_mutated = true`,
   `template_flow_demo.run_id` matches pattern `kgcf_run_yyyyMMddHHmm`.

3. **Download-type trigger preservation (if declared):**
   For any action whose descriptor has `mimeType` and `filename` (explicit download-type),
   clicking the button MUST still produce a file download — the fix MUST NOT break legitimate
   download actions.

4. **Button rendering preservation:**
   After fix, all buttons in `canvas:widgetCard.actions[]` render with correct `label`, `icon`,
   `primary` flag, and ordering — unchanged from pre-fix rendering.

---

### Unit Tests

- Test `handleActionButtonClick` calls `event.preventDefault()` and `event.stopPropagation()`
  before `dispatch()` is called, for every trigger value.
- Test that no `<a download>` ancestor exists in the DOM tree above any rendered action button.
- Test that `dispatch(trigger, action, context)` resolves to `dispatchTable.get(trigger)` and
  never to a Blob/download path for non-download triggers.
- Test that `dispatch` with an unrecognized trigger calls `fallbackHandler` (no download, no crash).
- Test that action buttons are rendered as `<button type="button">` elements.

### Property-Based Tests

- **Property 1 (generic, no hardcoded triggers):** For any action descriptor generated with an
  arbitrary `trigger` string not in `downloadTypeTriggerSet`, simulating a button click MUST NOT
  initiate a file download and MUST call the registered handler.
- **Property 2 (preservation, generic):** For any action descriptor where the handler is
  pre-registered and behavior is captured on correct code, the fixed dispatcher MUST produce
  identical observable output.
- **Dispatch table completeness:** For any trigger string registered in the dispatch table,
  `dispatchTable.get(trigger)` MUST return a defined, callable function (never `undefined` falling
  through to download).

### Integration Tests

- Full flow: open `knowgrph-flow-editor-computing-flow-template.md` in the 2D Flow Editor renderer,
  click Run on `source_input`, verify `compute_summary` compute is invoked and output fields are
  populated — no download dialog appears.
- Full flow: click Run on `compute_summary`, verify compute completes, `run_status` transitions
  `idle → running → done`, side effects applied.
- Regression guard: click Edit on `source_input`, verify field editor opens, no download.
- Regression guard: click Reset on `compute_summary`, verify output fields cleared, no download.
- Visual regression: verify all three action buttons render correctly on both widget cards after fix.
