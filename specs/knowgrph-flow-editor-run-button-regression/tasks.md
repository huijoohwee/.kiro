# Implementation Plan

## Overview

Four-phase exploratory bugfix for the Flow Editor Run button regression. Tasks follow the bug condition methodology: explore (confirm bug on unfixed code), preserve (capture baseline non-buggy behavior), implement (two-layer structural fix), validate (all tests pass, no regressions).

## Tasks

- [x] 1. Write bug condition exploration test
  - **Property 1: Bug Condition** - Non-Download Trigger Click Triggers File Download
  - **CRITICAL**: This test MUST FAIL on unfixed code — failure confirms the bug exists
  - **DO NOT attempt to fix the test or the code when it fails**
  - **NOTE**: This test encodes the expected behavior — it will validate the fix when it passes after implementation
  - **GOAL**: Surface counterexamples demonstrating the bug on unfixed code (Candidate A and/or Candidate B)
  - **Scoped PBT Approach**: Scope the property to the two concrete failing cases — `source_input` Run (`trigger: "runDownstream"`) and `compute_summary` Run (`trigger: "compute"`) — plus a DOM ancestor traversal probe and a Blob/`URL.createObjectURL` spy probe
  - Test setup: render widget cards for `source_input` and `compute_summary` nodes from `knowgrph-flow-editor-computing-flow-template.md` (renderer: `kgCanvas2dRenderer: "flowEditor"`)
  - **Candidate A probe**: For each rendered action button, traverse `parentElement` chain and assert no ancestor has `tagName === "A"` with a `download` attribute — on unfixed code this assertion FAILS, confirming Candidate A
  - **Candidate B probe**: Spy on `Blob` constructor and `URL.createObjectURL` in the dispatcher module; simulate click on a non-download action button; assert neither is called — on unfixed code this assertion FAILS, confirming Candidate B
  - **Core assertion**: `isBugCondition(action, event, result)` returns `true` for `runDownstream` and `compute` triggers — i.e., `result.fileDownloadInitiated = true` for those triggers
  - Formal condition: `action.trigger NOT IN downloadTypeTriggerSet(action) AND result.fileDownloadInitiated = true`
  - Run test on UNFIXED code
  - **EXPECTED OUTCOME**: Test FAILS (proves the bug exists; documents counterexamples)
  - Document counterexamples found (e.g., "clicking Run on `source_input` initiates `*.json` download instead of dispatching `runDownstream`")
  - Mark task complete when test is written, run, and failure + counterexamples are documented
  - _Requirements: 1.1, 1.2, 1.3_

- [x] 2. Write preservation property tests (BEFORE implementing fix)
  - **Property 2: Preservation** - Non-Buggy Action Paths Produce Correct Behavior
  - **IMPORTANT**: Follow observation-first methodology — observe actual behavior on UNFIXED code for inputs where `isBugCondition` = false
  - **Scope**: Actions where the trigger is either a legitimate download-type trigger (with `mimeType` + `filename`), or is being tested against already-correct infrastructure (field editor, output clearing, side effects, rendering)
  - **Observe on unfixed code**:
    - `openFieldEditor` (Edit on `source_input`) → opens field editor for `input_query`
    - `clearOutputs` (Reset on `compute_summary`) → clears `output`, `imageUrl`, `outputSrcDoc`
    - `compute` completion → `run_status = "done"`, `template_flow_demo.active_graph_mutated = true`, `template_flow_demo.run_id` matches `kgcf_run_yyyyMMddHHmm`
    - Button rendering → all buttons render with correct `label`, `icon`, `primary` flag, and ordering from `canvas:widgetCard.actions[]`
    - Legitimate download-type trigger (descriptor with `mimeType` + `filename`) → file download is produced
  - **Write property-based tests capturing observed behavior patterns**:
    - For all actions where `isBugCondition` = false: `observeOutcome_fixed(action) = observeOutcome_originalCorrectBehavior(action)`
    - PBT generates arbitrary action descriptors with varied trigger values where `isBugCondition` = false and asserts behavioral identity
    - PBT generates varied non-download trigger strings and asserts dispatch table completeness (`dispatchTable.get(trigger)` returns a defined, callable function)
  - Run tests on UNFIXED code
  - **EXPECTED OUTCOME**: Tests PASS (confirms baseline behavior to preserve after fix)
  - Mark task complete when tests are written, run, and passing on unfixed code
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

- [x] 3. Fix for Run button regression — non-download trigger clicks triggering file download

  - [x] 3.1 Layer 1 — Remove `<a download>` wrapper from widget card action button area
    - Locate the shared widget card button renderer component (2D Flow Editor renderer layer, `kgCanvas2dRenderer: "flowEditor"`)
    - Remove any `<a href={blobUrl} download={filename}>` element wrapping the card container or action button area
    - If a legitimate export anchor is needed, restructure it as a sibling element — never as a parent/ancestor of action buttons
    - Render action buttons as `<button type="button">` elements — never as `<a>` or as descendants of `<a download>`
    - _Bug_Condition: isBugCondition(action, event, result) where action.trigger NOT IN downloadTypeTriggerSet — Candidate A: click event bubbles to parent `<a download>` ancestor_
    - _Expected_Behavior: no `<a download>` ancestor exists in the DOM tree above any rendered action button from `canvas:widgetCard.actions[]`_
    - _Preservation: button labels, icons, primary styling, and ordering from `canvas:widgetCard.actions[]` unchanged_
    - _Requirements: 1.3, 2.3, 3.5_

  - [x] 3.2 Layer 1 — Add unconditional event guard to every action button click handler
    - Apply `event.preventDefault()` and `event.stopPropagation()` unconditionally at the top of every action button click handler, before any dispatch call
    - Guard is not conditional on trigger type — it applies to every button rendered from `canvas:widgetCard.actions[]`
    - Implementation: `handleActionButtonClick(event, action, dispatchContext) { event.preventDefault(); event.stopPropagation(); actionDispatcher.dispatch(action.trigger, action, dispatchContext); }`
    - _Bug_Condition: missing preventDefault/stopPropagation allows event to bubble to `<a download>` ancestor or invoke default browser navigation_
    - _Expected_Behavior: result.eventDefaultPrevented = true AND result.eventPropagationStopped = true for every action button click_
    - _Requirements: 2.3_

  - [x] 3.3 Layer 2 — Remove Blob URL / `<a download>` injection branch from action dispatcher
    - Locate the shared, headless action dispatch module (canonical, shared across all renderer contexts)
    - Remove ALL of the following from the general dispatch flow (permitted only inside an explicitly registered download-type handler):
      - `new Blob([JSON.stringify(node)])` / `new Blob([JSON.stringify(frontmatter)])` calls
      - `URL.createObjectURL(...)` calls
      - Programmatic `<a>` element creation + `download` attribute + `.click()` + DOM append/remove
    - _Bug_Condition: isBugCondition(action, event, result) where action.trigger NOT IN downloadTypeTriggerSet — Candidate B: dispatcher Blob URL branch fires for non-download triggers_
    - _Expected_Behavior: Blob constructor and URL.createObjectURL are never called during dispatch of a non-download trigger_
    - _Preservation: download-type handlers (with mimeType + filename declared) MUST still be able to produce file downloads via their registered handler_
    - _Requirements: 1.3, 2.1, 2.2, 2.3_

  - [x] 3.4 Layer 2 — Replace hardcoded trigger/node-type conditionals with pure dispatch table lookup
    - Remove all `if (trigger === "...")`, `switch(trigger)`, `if (nodeType === "InputWidget")`, `if (nodeType === "ComputeWidget")` blocks from the shared dispatcher
    - Implement generic dispatch: `handler := dispatchTable.get(trigger); if handler IS UNDEFINED then fallbackHandler(trigger, action, context); return; end if; handler(action, context)`
    - `fallbackHandler` logs a warning and returns without side effects — it MUST NOT trigger a download
    - Dispatch table is populated at registration time by the feature layer; dispatcher is unopinionated about trigger semantics
    - _Bug_Condition: legacy trigger alias tables or hardcoded conditionals cause runDownstream/compute to resolve to the download branch_
    - _Expected_Behavior: dispatchTable.get(trigger) is the sole routing mechanism; unknown triggers → fallbackHandler (no download, no crash)_
    - _Requirements: 2.1, 2.2, 2.3_

  - [x] 3.5 Layer 2 — Remove all legacy trigger name aliases and backward-compatibility remappings
    - Remove any `trigger → oldTriggerName` alias tables, compatibility shims, or fallthrough-to-download paths for unrecognized triggers
    - Remove any entry that remaps a trigger name to a legacy name or treats unrecognized triggers as "export" or "download"
    - No backward-compatibility aliases are left in place — 100% removal
    - _Bug_Condition: stale/legacy trigger name remapping causes runDownstream or compute to resolve to the download handler_
    - _Expected_Behavior: trigger strings are matched directly against dispatchTable with no pre-dispatch transformation or aliasing_
    - _Requirements: 2.1, 2.2, 2.3_

  - [x] 3.6 Layer 2 — Use `buildScopedGraphSemanticKey` for dispatch context construction
    - When constructing the dispatch context, reuse the existing `buildScopedGraphSemanticKey` helper to derive the scoped graph/node identity
    - Remove any inline key construction in the dispatcher or renderer
    - Context object passed to the handler includes: scoped key, action descriptor, and node's frontmatter reference — dispatcher MUST NOT serialize frontmatter to a Blob or file
    - `buildScopedGraphSemanticKey` is reused without modification
    - _Bug_Condition: inline key construction or frontmatter serialization in dispatcher contributes to Blob download path_
    - _Expected_Behavior: context construction uses buildScopedGraphSemanticKey; frontmatter is never serialized to Blob in the dispatcher_
    - _Preservation: buildScopedGraphSemanticKey behavior unchanged_
    - _Requirements: 2.3_

  - [x] 3.7 Verify bug condition exploration test now passes
    - **Property 1: Expected Behavior** - Non-Download Trigger Click Dispatches Correct Handler, No File Download
    - **IMPORTANT**: Re-run the SAME test from task 1 — do NOT write a new test
    - The test from task 1 encodes the expected behavior: `result.fileDownloadInitiated = false`, `result.dispatchedTrigger = action.trigger`, `result.eventDefaultPrevented = true`, `result.eventPropagationStopped = true`
    - When this test passes, it confirms Property 1 is satisfied for all non-download trigger clicks
    - Verify for all four concrete cases: `runDownstream` (source_input Run), `compute` (compute_summary Run), `openFieldEditor` (source_input Edit), `clearOutputs` (compute_summary Reset)
    - Verify generic PBT case: for any arbitrary non-download trigger, no file download and correct handler called
    - Run bug condition exploration test from step 1
    - **EXPECTED OUTCOME**: Test PASSES (confirms bug is fixed; Property 1 holds)
    - _Requirements: 1.3, 2.1, 2.2, 2.3 — Expected Behavior Properties from design_

  - [x] 3.8 Verify preservation tests still pass
    - **Property 2: Preservation** - Non-Buggy Action Paths Unchanged
    - **IMPORTANT**: Re-run the SAME tests from task 2 — do NOT write new tests
    - Run preservation property tests from step 2
    - Verify: `openFieldEditor` → field editor opens for `input_query` (Requirement 3.1)
    - Verify: `clearOutputs` → output fields cleared (`output`, `imageUrl`, `outputSrcDoc`) (Requirement 3.2)
    - Verify: `runDownstream` dispatched → compute invoked on `compute_summary`, output fields updated (Requirement 3.3)
    - Verify: `compute` completion → `run_status = "done"`, `template_flow_demo.active_graph_mutated = true`, `template_flow_demo.run_id` matches `kgcf_run_yyyyMMddHHmm` (Requirement 3.4)
    - Verify: all buttons render with correct `label`, `icon`, `primary` flag, ordering (Requirement 3.5)
    - Verify: download-type triggers (with `mimeType` + `filename`) still produce file downloads
    - **EXPECTED OUTCOME**: Tests PASS (confirms no regressions; Property 2 holds)

- [x] 4. Checkpoint — Ensure all tests pass
  - Run all tests (exploration/bug-condition, preservation, unit, and integration)
  - Confirm Property 1 holds: ∀ action where trigger ∉ downloadTypeTriggerSet → click MUST NOT produce file download, MUST dispatch registered handler (validates: 1.3, 2.1, 2.2, 2.3)
  - Confirm Property 2 holds: ∀ action where isBugCondition = false → fixed behavior identical to correct original behavior (validates: 3.1, 3.2, 3.3, 3.4, 3.5)
  - Confirm no legacy/stale code remains: no Blob serialization, no URL.createObjectURL, no programmatic `<a download>` injection, no `<a download>` card wrappers, no legacy trigger alias tables, no hardcoded if/switch on trigger names or node types in dispatcher or renderer
  - Confirm `buildScopedGraphSemanticKey` is used for all dispatch context construction
  - Ensure all tests pass; ask the user if questions arise

## Task Dependency Graph

```json
{
  "waves": [
    {"wave": 1, "tasks": ["1", "2"]},
    {"wave": 2, "tasks": ["3.1", "3.2", "3.3", "3.4", "3.5", "3.6"]},
    {"wave": 3, "tasks": ["3.7", "3.8"]},
    {"wave": 4, "tasks": ["4"]}
  ]
}
```

## Notes

- Renderer context: `kgCanvas2dRenderer: "flowEditor"` (2D Flow Editor)
- Validation anchor: `knowgrph-flow-editor-computing-flow-template.md`
- All changes are at exactly two layers: widget card button renderer (Layer 1) and shared action dispatcher (Layer 2)
- No hardcoded trigger names, node IDs, or widget types in renderer or dispatcher
- `buildScopedGraphSemanticKey` is reused without modification
- 100% removal of legacy/stale code — no backward-compatibility aliases
- Download-type triggers (with `mimeType` + `filename` in action descriptor) are fully preserved
