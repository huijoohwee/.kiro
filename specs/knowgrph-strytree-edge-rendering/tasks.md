# Implementation Plan

## Overview

This is a **two-document contract edit** (no application code). The two fix targets are:

- **File 1 (PRD/TAD):** `knowgrph/docs/documents/knowgrph-strytree-prd-tad.md` (v0.2.1 → v0.2.2)
- **File 2 (demo/anchor):** `huijoohwee/docs/knowgrph-agentic-canvas-os-demo.md`

Because both targets are documents, "tests" are **deterministic document contract checks** evaluated over each document's frontmatter/AST and text (presence/binding/forbid predicates, snapshot byte-for-identical-intent comparisons, and the `rendererAgnostic` projection-invariance predicate across `{flowEditor, Storyboard, Strybldr}`). Honored invariants: no new graph-rendering dependency, Cloudflare-only topology, `Dev -> Prod -> Cloudflare`, spec-complete → runtime-ready, forbid-hardcode-in-repo.

Tasks 1–2 run on the **UNFIXED** documents (Task 1 must FAIL, Task 2 must PASS). Task 3 applies the surgical edits (1:1 with design "Fix Implementation > Changes Required" and the Cleanup Inventory). Task 4 re-runs the same checks. Task 5 implements the remaining verification strategy. Task 6 is the checkpoint.

---

## Tasks

- [x] 1. Write bug condition exploration test (BEFORE any edit)
  - **Property 1: Bug Condition** - Shared, Renderer-Agnostic Edge Projection
  - **CRITICAL**: This test MUST FAIL on the unfixed documents - failure confirms the contract gap exists
  - **DO NOT attempt to fix the test or the documents when it fails**
  - **NOTE**: This test encodes the expected finetuned behavior - it will validate the fix when it passes after Task 3
  - **GOAL**: Surface counterexamples that demonstrate edge-rendering concerns are not bound to the shared, renderer-agnostic contract
  - **Scoped PBT Approach**: For this deterministic document gap, scope the property to the concrete edge-rendering concerns enumerated in the design "Exploratory Bug Condition Checking" so failures are reproducible
  - Parse both documents' frontmatter/text and assert, for each edge-rendering concern X where `isBugCondition(X)` holds, that the finetuned predicate set holds: `boundToSharedRendererContract(X)` AND `usesSharedSemanticKeyHelper(X)` (`buildScopedGraphSemanticKey`) AND `derivesEdgesFrom(X, "parent_node_id")` AND `projectsViewStateOnly(X)` AND `rendererAgnostic(X)` across `{flowEditor, Storyboard, Strybldr}` AND `forbidsNonNeutralMechanisms(X)`
  - Concrete checks (from design Test Cases): (a) PRD/TAD frontmatter carries `kgSharedRendererContract` + `socket_types` + `flow`; (b) PRD-STR-E02-AC-03 references `buildScopedGraphSemanticKey`; (c) PRD/TAD carries `edgeContractForbid` + `edgeContractCleanup`; (d) demo doc declares a supported-renderer-set capability + `rendererAgnosticEdges`; (e) both docs state identical projection across `{flowEditor, Storyboard, Strybldr}`; (f) Part A Bezier / edge-existence rows are marked historical
  - Run test on UNFIXED documents
  - **EXPECTED OUTCOME**: Test FAILS (this is correct - it proves the contract gap exists)
  - Document counterexamples found (expected: PRD/TAD frontmatter has no `kgSharedRendererContract`/`socket_types`/`flow`; PRD-STR-E02-AC-03 has no semantic-key reference; demo pins `kgCanvas2dRenderer: "flowEditor"` with no capability field; Part A Bezier prose unmarked)
  - Mark task complete when the test is written, run, and the failure is documented
  - _Implements: design "Exploratory Bug Condition Checking" + "Fix Checking" + Correctness Property 1_
  - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 2.7, 2.8, 2.9, 2.10_

- [x] 2. Write preservation property tests (BEFORE any edit)
  - **Property 2: Preservation** - Non-Edge Contract and Demo Content
  - **IMPORTANT**: Follow observation-first methodology - capture real current content, not assumed content
  - Observe on UNFIXED docs and snapshot: PRD/TAD non-edge epics PRD-STR-E01, E03, E04, E05, E06, E07, and E08 non-UI clauses
  - Observe on UNFIXED docs and snapshot: PRD-STR-E02-AC-03 derivation invariant (one edge per non-null `parent_node_id`, NO separate edge table) and Part C "Edge Strategy" text
  - Observe on UNFIXED docs and snapshot: the three constraints (no new graph-rendering dep; no DB outside Cloudflare; no hosting outside `Dev -> Prod -> Cloudflare`) plus `deployment_topology`/`cloudflare_route`
  - Observe on UNFIXED demo doc and snapshot ALL non-edge demo content: `agentic_canvas_os_demo`, `flow_diagrams`, every `flow.nodes` payload, `handles`, `flow:portTypes`, `compute` functions, `outputSrcDoc` previews, `socket_types` *values*, and the body sections `## Response`, `## Rich Media Outputs`, `## Inputs`, `## Guardrails`
  - Write property-based tests asserting `F(X) = F'(X)` (byte-for-identical-intent) for all sampled non-edge concerns `X` where `isBugCondition(X)` is false
  - Write the **renderer-switch invariance** check: assert both documents state that switching the active 2D renderer among `{flowEditor, Storyboard, Strybldr}` leaves all non-edge behavior identical (node content, card surfaces, compute/run actions, timeline/transport surfaces, every non-edge field)
  - Run tests on UNFIXED documents
  - **EXPECTED OUTCOME**: Tests PASS (this confirms the baseline non-edge content to preserve)
  - Mark task complete when tests are written, run, and passing on the unfixed documents
  - _Implements: design "Preservation Checking" + "Property-Based Tests" + Correctness Property 2_
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 3.9, 3.10_

- [x] 3. Apply the two-document contract edits (the fix)

  - [x] 3.1 File 1 (PRD/TAD) edit 1A — add shared renderer contract + capability + socket_types + flow.storyEdgeProjection + forbid/cleanup/principles, bump version/updated
    - In `knowgrph/docs/documents/knowgrph-strytree-prd-tad.md` frontmatter, insert after the `related:` block (keep all existing keys intact): `kgCanvas2dRendererCapability` (`supportedRenderers: ["flowEditor","Storyboard","Strybldr"]`, `selectionModel: "projected-data"`, `edgeProjectionInvariance: "identical-across-supportedRenderers"`)
    - Add `kgSharedRendererContract` (`version: shared-renderer-contract/v1`, `semanticIdentity: buildScopedGraphSemanticKey`, `edgeModel`, `edgeSource: strytree_nodes.parent_node_id`, `rendererPolicy`)
    - Add `socket_types` (`idea_signal`/`evidence_signal`/`approval_signal`/`artifact_signal` with `color`/`edgeWidthPx`/`handleStrokeWidthPx`/`accepts` matching the anchor doc)
    - Add `flow` (`direction: LR`, `edgeType: smoothstep`, `storyEdgeProjection` with `handleModel`, `portTypeDefault: idea_signal`, `semanticKeyRule: buildScopedGraphSemanticKey(storyId, parentNodeId, childNodeId)`)
    - Add `edgeContractForbid` (full forbid list), `edgeContractCleanup` (root/source neutralization, NO backward-compat remap), `edgeContractPrinciples`
    - Bump `version` `0.2.1` → `0.2.2` and update `updated`
    - _Implements: design Fix Implementation §1A_
    - _Bug_Condition: isBugCondition(X) for PRD/TAD frontmatter lacking shared contract / socket_types / flow_
    - _Expected_Behavior: boundToSharedRendererContract ∧ usesSharedSemanticKeyHelper ∧ projectsViewStateOnly ∧ rendererAgnostic ∧ forbidsNonNeutralMechanisms_
    - _Preservation: all existing frontmatter keys byte-for-identical-intent (3.x)_
    - _Requirements: 2.1, 2.3, 2.4, 2.5, 2.6, 2.7, 2.8_

  - [x] 3.2 File 1 (PRD/TAD) edit 1B — reclassify Part A prototype edge prose as historical source analysis
    - Add a one-line banner at the top of `## A1. Observed Strytree Prototype` stating Part A records observed prototype behavior as historical source analysis only and is NOT the target implementation contract; runtime edge contract lives in frontmatter `kgSharedRendererContract` + `flow.storyEdgeProjection` and Part C "Edge Rendering Contract"
    - In the `### Prototype Calculation Engine` table, append "(historical prototype only)" to the `Edge existence` row and the `Edge shape | SVG cubic Bezier...` row
    - Do NOT delete prototype prose (preserve as historical per 3.7)
    - _Implements: design Fix Implementation §1B + Cleanup Inventory rows for Part A edge rows_
    - _Bug_Condition: isBugCondition(X) where prototype Bezier prose reads as permitted runtime contract_
    - _Expected_Behavior: prototype edge prose neutralized via reclassification (forbidsNonNeutralMechanisms)_
    - _Preservation: Part A text otherwise unchanged (3.7)_
    - _Requirements: 2.7_

  - [x] 3.3 File 1 (PRD/TAD) edit 1C — bind PRD-STR-E02-AC-03 to the shared contract (preserve derivation invariant)
    - Keep the existing AC text verbatim (one edge per non-null `parent_node_id`, no edge table — preserved per 3.1)
    - Append: the derived edge SHALL be projected through `kgSharedRendererContract@shared-renderer-contract/v1` using `buildScopedGraphSemanticKey` for identity and the `flow` port/handle/`socket_types` model for typing; local/downstream patches, alias stacking, and hardcoded edge logic are forbidden
    - _Implements: design Fix Implementation §1C + Cleanup Inventory row PRD-STR-E02-AC-03_
    - _Bug_Condition: isBugCondition(X) where AC derives edge but does not bind projection to shared contract_
    - _Expected_Behavior: derivesEdgesFrom(parent_node_id) ∧ boundToSharedRendererContract ∧ usesSharedSemanticKeyHelper_
    - _Preservation: derivation invariant (one edge per non-null parent_node_id, no edge table) preserved verbatim (3.1)_
    - _Requirements: 2.1, 2.2, 2.5_

  - [x] 3.4 File 1 (PRD/TAD) edit 1D — add new Part C "Edge Rendering Contract" subsection under C4
    - Insert immediately after `### Edge Strategy` (preserve the `parent_node_id` SSOT statement verbatim above it)
    - State the renderer-projection policy (renderers project view state only; no re-calculation/re-computation/re-rendering of source-owned edges)
    - State the renderer-agnostic rule (same `kgSharedRendererContract` + `buildScopedGraphSemanticKey` + `socket_types` + flow port/handle model drives projection regardless of active 2D renderer; forbid renderer-specific edge path, per-renderer hardcode, per-renderer fork)
    - State the cross-document unification rule (Strytree `parent_node_id`-derived edges and demo flow nodes/handles resolve to the SAME shared edge projection across `flowEditor | Storyboard | Strybldr` with no recompute/re-render/duplicate/stale on renderer switch)
    - _Implements: design Fix Implementation §1D_
    - _Bug_Condition: isBugCondition(X) where no renderer-projection policy / renderer-agnostic rule / unifying rule exists_
    - _Expected_Behavior: projectsViewStateOnly ∧ rendererAgnostic ∧ cross-document one-contract unification_
    - _Preservation: Part C "Edge Strategy" SSOT text preserved verbatim (3.1, 3.6)_
    - _Requirements: 2.6, 2.9, 2.10_

  - [x] 3.5 File 1 (PRD/TAD) edit 1E — reword C3 "Storytree SVG Renderer" responsibility row
    - Change "Renders nodes, parent-derived edges, pan, zoom..." so it *projects* parent-derived edges through the shared contract (view state only), not owns/recomputes them
    - Edit the edge-governing row only; leave all other C3 rows untouched
    - _Implements: design Fix Implementation §1E + Cleanup Inventory row C3 "Storytree SVG Renderer"_
    - _Bug_Condition: isBugCondition(X) where renderer responsibility is ownership-ambiguous (could own/recompute edges)_
    - _Expected_Behavior: projectsViewStateOnly (renderer projects, does not own/recompute)_
    - _Preservation: all non-edge C3 rows unchanged (3.5, 3.6)_
    - _Requirements: 2.6_

  - [x] 3.6 File 1 (PRD/TAD) edit 1F — add two neutralization/agnosticity frontmatter constraints
    - Append to frontmatter `constraints` (keep all existing constraints byte-identical): "story edge rendering must bind to kgSharedRendererContract@shared-renderer-contract/v1 and buildScopedGraphSemanticKey; no local/downstream/hardcoded edge logic"
    - Append: "story edge projection must be renderer-agnostic across flowEditor | Storyboard | Strybldr; no per-renderer edge path, hardcode, or fork"
    - _Implements: design Fix Implementation §1F_
    - _Bug_Condition: isBugCondition(X) where constraints permit non-neutral / per-renderer edge logic_
    - _Expected_Behavior: forbidsNonNeutralMechanisms ∧ rendererAgnostic_
    - _Preservation: all existing constraints byte-identical (3.2, 3.3, 3.4)_
    - _Requirements: 2.2, 2.7, 2.9_

  - [x] 3.7 File 2 (demo doc) edit 2A — add kgCanvas2dRendererCapability field
    - In `huijoohwee/docs/knowgrph-agentic-canvas-os-demo.md` frontmatter, insert immediately after `kgCanvas2dRenderer: "flowEditor"` (preserve that line as the *default active* renderer, not the *only* renderer): `kgCanvas2dRendererCapability` (`supportedRenderers: ["flowEditor","Storyboard","Strybldr"]`, `selectionModel: "projected-data"`, `edgeProjectionInvariance: "identical-across-supportedRenderers"`)
    - _Implements: design Fix Implementation §2A + Cleanup Inventory row demo `kgCanvas2dRenderer`_
    - _Bug_Condition: isBugCondition(X) where demo pins a single 2D renderer with no capability set_
    - _Expected_Behavior: rendererAgnostic (renderer set declared as projected data, not branched on)_
    - _Preservation: `kgCanvas2dRenderer: "flowEditor"` kept as default; all other frontmatter keys unchanged (3.9)_
    - _Requirements: 2.8_

  - [x] 3.8 File 2 (demo doc) edit 2B — add rendererAgnosticEdges line to existing kgSharedRendererContract
    - Extend the existing `kgSharedRendererContract` block with one line: `rendererAgnosticEdges: "edges project identically for every supportedRenderer; no renderer-specific edge path"` (all existing keys preserved byte-for-identical-intent)
    - **Explicitly NO other demo edits**: `flow.edges`, all `flow.nodes`, `handles`, `flow:portTypes`, `socket_types` values, `agentic_canvas_os_demo`, `flow_diagrams`, `compute` functions, `outputSrcDoc` previews, and the entire body (`## Response`, `## Rich Media Outputs`, `## Inputs`, `## Guardrails`) remain unchanged
    - _Implements: design Fix Implementation §2B (+ "No other demo edits" note)_
    - _Bug_Condition: isBugCondition(X) where demo contract does not declare renderer-agnostic edge projection_
    - _Expected_Behavior: rendererAgnostic ∧ projectsIdenticallyFor({flowEditor, Storyboard, Strybldr})_
    - _Preservation: all non-edge demo content byte-for-identical-intent (3.9)_
    - _Requirements: 2.8, 2.9, 2.10_

  - [x] 3.9 Cleanup — neutralize legacy/stale/conflicting edge specification per the Cleanup Inventory (NO backward-compat remap)
    - Confirm each Cleanup Inventory row is addressed: Part A `Graph rendering` row kept under the reclassification banner (3.7); Part A `Edge existence` + `Edge shape` rows marked "(historical prototype only)" (done in 3.2); PRD-STR-E02-AC-03 extended (3.3); Part C `Edge Strategy` kept verbatim with new subsection beneath (3.4); C3 renderer row reworded (3.5); demo `kgCanvas2dRenderer` kept as default + capability/agnostic lines added (3.7, 3.8)
    - Verify there is **NO backward-compatibility remapping**: the prototype Bezier rule is NOT carried forward as an alternate runtime path; no alias bridges old prose to the new contract; 100% of conflicting/legacy edge *specification* is neutralized by reclassification + replacement of prose-only runtime spec with the frontmatter contract
    - _Implements: design "Legacy / Stale / Conflicting Cleanup Inventory"_
    - _Bug_Condition: isBugCondition(X) where legacy/stale edge spec permits downstream patching_
    - _Expected_Behavior: forbidsNonNeutralMechanisms (root/source neutralization, no remap)_
    - _Preservation: prototype prose retained as historical source analysis only (3.7)_
    - _Requirements: 2.7_

  - [x] 3.10 Verify bug condition exploration test now passes
    - **Property 1: Expected Behavior** - Shared, Renderer-Agnostic Edge Projection
    - **IMPORTANT**: Re-run the SAME test from Task 1 - do NOT write a new test
    - The test from Task 1 encodes the expected behavior; when it passes it confirms every edge-rendering concern is now bound to the shared, renderer-agnostic contract
    - Run the bug condition exploration test from Task 1 against the edited documents
    - **EXPECTED OUTCOME**: Test PASSES (confirms the contract gap is fixed)
    - _Implements: design "Fix Checking"_
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 2.7, 2.8, 2.9, 2.10_

  - [x] 3.11 Verify preservation tests still pass
    - **Property 2: Preservation** - Non-Edge Contract and Demo Content
    - **IMPORTANT**: Re-run the SAME tests from Task 2 - do NOT write new tests
    - Run the preservation property tests + renderer-switch invariance check from Task 2 against the edited documents
    - **EXPECTED OUTCOME**: Tests PASS (confirms no regressions — non-edge epics + all non-edge demo content remain byte-for-identical-intent; derivation invariant unchanged)
    - _Implements: design "Preservation Checking"_
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 3.9, 3.10_

- [x] 4. Implement remaining verification strategy (unit / property / integration)

  - [x] 4.1 Unit checks — frontmatter contract presence & binding
    - Assert PRD/TAD carries `kgSharedRendererContract`, `socket_types`, `flow`, `kgCanvas2dRendererCapability`, `edgeContractForbid`, `edgeContractCleanup`
    - Assert PRD-STR-E02-AC-03 references `buildScopedGraphSemanticKey` and preserves the no-edge-table derivation invariant
    - Assert Part A prototype edge rows are marked historical
    - Assert demo doc carries `kgCanvas2dRendererCapability.supportedRenderers == ["flowEditor","Storyboard","Strybldr"]` and `rendererAgnosticEdges`
    - _Implements: design "Unit Tests"_
    - _Requirements: 2.1, 2.3, 2.4, 2.5, 2.7, 2.8_

  - [x] 4.2 Property-based checks — Property 1 and Property 2 over sampled concerns
    - For randomly sampled edge-rendering concerns across both docs, assert Property 1 (bound to shared contract, semantic-key identity, parent-derived, view-state-only, renderer-agnostic, forbids non-neutral mechanisms)
    - For randomly sampled non-edge concerns, assert Property 2 (`F(X) = F'(X)`)
    - _Implements: design "Property-Based Tests"_
    - _Requirements: 2.1, 2.2, 2.6, 3.5, 3.9_

  - [x] 4.3 Property-based check — rendererAgnostic predicate across the renderer set
    - Generate the renderer set `{flowEditor, Storyboard, Strybldr}` and assert projection invariance for every member
    - Assert no renderer-specific edge path / per-renderer hardcode / per-renderer fork is introduced in either document
    - _Implements: design "Property-Based Tests" (rendererAgnostic) + "Preservation Checking" (rendererAgnostic exercised across all three renderers)_
    - _Requirements: 2.8, 2.9, 2.10, 3.8_

  - [x] 4.4 Integration check — cross-document unification onto one shared contract
    - Assert Strytree `parent_node_id`-derived edges and the demo flow nodes/handles both resolve to the SAME shared edge projection contract (`shared-renderer-contract/v1` + `buildScopedGraphSemanticKey` + `socket_types` + flow port/handle model)
    - _Implements: design "Integration Tests" (cross-document unification)_
    - _Requirements: 2.10_

  - [x] 4.5 Integration check — renderer-switch flow invariance
    - Assert that switching the declared active renderer among the three supported values changes no edge projection and no non-edge content in either document
    - _Implements: design "Integration Tests" (renderer-switch flow)_
    - _Requirements: 2.9, 3.8_

  - [x] 4.6 Integration check — spec-complete → runtime-ready, within Cloudflare-only topology
    - Assert PRD/TAD frontmatter now carries every field a renderer needs to project Strytree edges without prose interpretation
    - Assert no new graph-rendering dependency is introduced; assert the Cloudflare-only / `Dev -> Prod -> Cloudflare` topology and `forbid-hardcode-in-repo` principle hold
    - _Implements: design "Integration Tests" (spec-complete → runtime-ready)_
    - _Requirements: 2.7, 3.2, 3.3, 3.4_

- [x] 5. Checkpoint - Ensure all tests pass
  - Ensure the Task 1 exploration test PASSES, the Task 2 preservation tests PASS, and all Task 4 unit/property/integration checks PASS
  - Confirm both documents are spec-complete → runtime-ready with no new graph-rendering dependency, Cloudflare-only topology, `Dev -> Prod -> Cloudflare` hosting, and no forbidden mechanisms
  - Ask the user if questions arise

---

## Task Dependency Graph

```json
{
  "waves": [
    {
      "wave": 1,
      "tasks": ["1"],
      "dependsOn": [],
      "description": "Bug condition exploration test — MUST FAIL on unfixed documents (confirms the contract gap)."
    },
    {
      "wave": 2,
      "tasks": ["2"],
      "dependsOn": ["1"],
      "description": "Preservation property tests + renderer-switch invariance — MUST PASS on unfixed documents (baseline to preserve)."
    },
    {
      "wave": 3,
      "tasks": ["3.1", "3.2", "3.7", "3.8"],
      "dependsOn": ["2"],
      "description": "Independent edits: PRD/TAD frontmatter contract (3.1) and Part A reclassification (3.2); demo capability field (3.7) and rendererAgnosticEdges line (3.8). File 1 and File 2 groups run in parallel."
    },
    {
      "wave": 4,
      "tasks": ["3.3", "3.4", "3.5", "3.6"],
      "dependsOn": ["3.1"],
      "description": "PRD/TAD edits that reference the frontmatter contract added in 3.1: AC-03 binding, Part C subsection, C3 reword, frontmatter constraints."
    },
    {
      "wave": 5,
      "tasks": ["3.9"],
      "dependsOn": ["3.2", "3.3", "3.4", "3.5", "3.6", "3.7", "3.8"],
      "description": "Cleanup per the Cleanup Inventory; verify NO backward-compat remapping."
    },
    {
      "wave": 6,
      "tasks": ["3.10", "3.11"],
      "dependsOn": ["3.9"],
      "description": "Re-run the SAME Task 1 test (now PASSES) and Task 2 tests (still PASS). No new tests written."
    },
    {
      "wave": 7,
      "tasks": ["4.1", "4.2", "4.3", "4.4", "4.5", "4.6"],
      "dependsOn": ["3.10", "3.11"],
      "description": "Remaining verification strategy: unit, property-based (Property 1/2 + rendererAgnostic), and integration checks."
    },
    {
      "wave": 8,
      "tasks": ["5"],
      "dependsOn": ["4.1", "4.2", "4.3", "4.4", "4.5", "4.6"],
      "description": "Checkpoint — ensure all tests pass."
    }
  ]
}
```

```
1. Bug Condition exploration test (FAILS on unfixed docs)
        │
2. Preservation property tests + renderer-switch invariance (PASS on unfixed docs)
        │
        ▼
3. Apply two-document contract edits (the fix)
   ├─ File 1 (PRD/TAD)
   │   ├─ 3.1 edit 1A  frontmatter: shared contract + capability + socket_types + flow.storyEdgeProjection + forbid/cleanup/principles + version bump
   │   ├─ 3.2 edit 1B  Part A reclassification banner + "(historical prototype only)" markers      ── independent text edit
   │   ├─ 3.3 edit 1C  PRD-STR-E02-AC-03 shared-contract binding (preserve derivation invariant)   ── depends on 3.1
   │   ├─ 3.4 edit 1D  new Part C "Edge Rendering Contract" subsection                               ── depends on 3.1
   │   ├─ 3.5 edit 1E  C3 "Storytree SVG Renderer" responsibility reword                             ── depends on 3.1
   │   └─ 3.6 edit 1F  two new frontmatter constraints                                               ── depends on 3.1
   ├─ File 2 (demo doc)
   │   ├─ 3.7 edit 2A  kgCanvas2dRendererCapability field
   │   └─ 3.8 edit 2B  rendererAgnosticEdges line in existing kgSharedRendererContract (NO other demo edits)
   ├─ 3.9 Cleanup per Cleanup Inventory (NO backward-compat remap)   ── depends on 3.2–3.8
   ├─ 3.10 Re-run Task 1 test → PASSES                                ── depends on 3.1–3.9
   └─ 3.11 Re-run Task 2 tests → still PASS                           ── depends on 3.1–3.9
        │
        ▼
4. Remaining verification strategy
   ├─ 4.1 Unit checks            ── depends on 3
   ├─ 4.2 PBT Property 1 & 2     ── depends on 3
   ├─ 4.3 PBT rendererAgnostic   ── depends on 3
   ├─ 4.4 Integration: cross-doc unification   ── depends on 3
   ├─ 4.5 Integration: renderer-switch flow    ── depends on 3
   └─ 4.6 Integration: spec-complete→runtime-ready / topology  ── depends on 3
        │
        ▼
5. Checkpoint - all tests pass   ── depends on 3, 4
```

## Notes

- Tasks 1 and 2 are STANDALONE and run BEFORE the fix (Task 1 must FAIL; Task 2 must PASS).
- File 1 and File 2 edit groups (3.1–3.6 vs 3.7–3.8) are independent of each other and may proceed in parallel.
- Within File 1, 3.1 (frontmatter contract) is the prerequisite for 3.3, 3.4, 3.5, 3.6; 3.2 is independent.
- 3.10/3.11 re-run the SAME tests from Tasks 1/2 — no new tests are written.
- Because both fix targets are documents, every "test" is a deterministic document contract check (presence/binding/forbid predicates, snapshot byte-for-identical-intent comparisons, and the `rendererAgnostic` projection-invariance predicate across `{flowEditor, Storyboard, Strybldr}`).
- Honored invariants throughout: no new graph-rendering dependency, Cloudflare-only topology, `Dev -> Prod -> Cloudflare`, spec-complete → runtime-ready, forbid-hardcode-in-repo, and NO backward-compatibility remapping.
