# Bugfix Requirements Document

## Introduction

The Knowgrph Strytree PRD/TAD (`knowgrph/docs/documents/knowgrph-strytree-prd-tad.md`, v0.2.1) only *describes* story edges in prose and prototype-calculation terms ("story edges derive from `parent_node_id`"; "Edge shape: SVG cubic Bezier from parent card right edge to child card left edge"). It does **not** carry the shared, neutral edge-rendering contract that the reference anchor doc (`huijoohwee/docs/knowgrph-agentic-canvas-os-demo.md`) carries.

The anchor doc defines a reusable, document-owned rendering contract:

- `kgSharedRendererContract` (`version: shared-renderer-contract/v1`, `semanticIdentity: buildScopedGraphSemanticKey`, `edgeModel: "active graph edges from the selected source graph"`, `rendererPolicy: "frontmatter and source payloads own data; renderers project view state only"`).
- Typed `socket_types` (`idea_signal` / `evidence_signal` / `approval_signal` / `artifact_signal`, each with `color`, `edgeWidthPx`, `handleStrokeWidthPx`, `accepts`).
- `flow.edgeType: smoothstep`, `flow.direction`, and per-node `handles` + `flow:portTypes` that drive edge projection.

**Root cause to neutralize (at source/upstream, not downstream):** in the Strytree workbench, edges must be **derived from `parent_node_id` at the source/upstream** and **projected through the SHARED renderer contract + the shared semantic-key helper (`buildScopedGraphSemanticKey`)** — never through local/downstream patches, stacking aliases, hardcoded edge logic, or re-computation in the renderer. Because the PRD/TAD is the implementation contract (`status: implementation-contract`), the contract gap in the document is the bug: it permits non-neutral, hardcoded, or downstream-patched edge rendering to exist "in the first place."

**Renderer-agnostic root cause (cross-renderer scope):** the anchor/demo doc currently declares a single active 2D renderer (`kgCanvas2dRenderer: "flowEditor"`), and neither document states that the shared edge contract must project edges **identically across all 2D renderers**. The strytree workbench already renders on the Strybldr/Storyboard SVG/HTML card surface (PRD-STR-E08-AC-04), so edges must render the same way whether the active 2D renderer is `flowEditor`, `Storyboard`, or `Strybldr`. The shared edge-rendering contract must therefore be **renderer-agnostic**: the renderer set / capability is declared as data the renderers *project*, not branch on, and there must be NO renderer-specific edge code path, NO per-renderer hardcode, and NO aliasing/forking of edge logic per renderer. This directly reinforces the existing universality / neutrality / agnosticity / modularity principles.

**Fix targets (two implementation/anchor contracts):**

1. **The Strytree PRD/TAD** (`knowgrph/docs/documents/knowgrph-strytree-prd-tad.md`). The document must be finetuned so it is *spec-complete → runtime-ready*: it carries the shared renderer contract binding, the typed edge/socket model, the `flow` edge-projection fields, the `buildScopedGraphSemanticKey` semantic identity, the renderer-projection policy, the renderer-agnostic cross-renderer edge-projection rule, and explicit forbid/cleanup rules — while keeping every existing Strytree constraint intact.
2. **The anchor/demo doc** (`huijoohwee/docs/knowgrph-agentic-canvas-os-demo.md`). The demo doc must carry whatever contract field makes edge rendering renderer-agnostic (the supported 2D renderer set / capability declared as data the renderers project, not branch on), so its `flow` nodes/handles resolve to the SAME shared edge projection regardless of the active 2D renderer, while leaving all of its non-edge demo content unchanged.

Both documents must unify onto **one renderer-agnostic edge contract**: Strytree's `parent_node_id`-derived edges and the demo's flow nodes/handles must resolve to the SAME shared edge projection, so switching the active 2D renderer (`flowEditor` | `Storyboard` | `Strybldr`) never recomputes, re-renders, duplicates, or staleifies edges.

### Bug Condition Methodology

**F** = the current pair of documents, where (a) edge rendering is only described locally/descriptively and is not bound to the shared renderer contract, and (b) the edge contract is not declared renderer-agnostic across `flowEditor` / `Storyboard` / `Strybldr` (the demo doc pins a single `kgCanvas2dRenderer: "flowEditor"`).
**F'** = the finetuned pair of documents, where edge rendering is sourced from `parent_node_id` (Strytree) / flow nodes+handles (demo) upstream and projected identically through the shared renderer contract + `buildScopedGraphSemanticKey`, renderer-agnostically across `flowEditor` / `Storyboard` / `Strybldr`.

```pascal
FUNCTION isBugCondition(X)
  INPUT: X of type EdgeRenderingConcern   // a clause/section of either fix-target doc that governs how edges are rendered
  OUTPUT: boolean

  // X is buggy when it specifies or permits edge rendering WITHOUT binding it to the
  // shared, neutral renderer contract carried by the anchor doc, OR when it permits a
  // forbidden non-neutral mechanism (hardcode, local/downstream patch, alias stacking,
  // re-computation, legacy/stale remapping), OR when it ties edge rendering to a single
  // 2D renderer instead of projecting edges identically across flowEditor | Storyboard | Strybldr.
  RETURN governsEdgeRendering(X)
         AND ( ( NOT boundToSharedRendererContract(X)        // kgSharedRendererContract@shared-renderer-contract/v1
                 AND NOT usesSharedSemanticKeyHelper(X) )    // buildScopedGraphSemanticKey
             OR permitsForbiddenMechanism(X)                 // hardcode | local/downstream patch | alias | re-compute | legacy remap
             OR NOT rendererAgnostic(X) )                    // renderer-specific edge path / per-renderer hardcode / fork per renderer
END FUNCTION

FUNCTION rendererAgnostic(X)
  INPUT: X of type EdgeRenderingConcern
  OUTPUT: boolean

  // True only when edges project identically for every supported 2D renderer,
  // with the renderer set declared as data the renderers project, not branch on.
  RETURN projectsIdenticallyFor(X, {"flowEditor", "Storyboard", "Strybldr"})
         AND NOT hasRendererSpecificEdgePath(X)
         AND NOT hasPerRendererHardcode(X)
         AND NOT forksEdgeLogicPerRenderer(X)
END FUNCTION
```

```pascal
// Property: Fix Checking — every edge-rendering concern is sourced upstream and projected via the
// shared contract, renderer-agnostically across flowEditor | Storyboard | Strybldr
FOR ALL X WHERE isBugCondition(X) DO
  result <- F'(X)
  ASSERT boundToSharedRendererContract(result)
     AND usesSharedSemanticKeyHelper(result)          // buildScopedGraphSemanticKey
     AND derivesEdgesFrom(result, "parent_node_id")   // Strytree, at source/upstream
     AND projectsViewStateOnly(result)                // renderers do not own data, no re-compute
     AND rendererAgnostic(result)                     // identical projection for flowEditor | Storyboard | Strybldr
     AND forbidsNonNeutralMechanisms(result)          // no hardcode/legacy/alias/patch/re-render/stale
END FOR

// Property: Preservation Checking — non-edge-contract content (including all non-edge demo content
// and non-edge epics) is unchanged, and switching the active 2D renderer never alters non-edge behavior
FOR ALL X WHERE NOT isBugCondition(X) DO
  ASSERT F(X) = F'(X)
END FOR
```

## Bug Analysis

### Current Behavior (Defect)

What currently happens in the Strytree PRD/TAD (`F`) for edge-rendering concerns.

1.1 WHEN the document specifies story-edge rendering THEN it describes edges only descriptively ("SVG cubic Bezier from parent card right edge to child card left edge") and does NOT reference the shared `kgSharedRendererContract` (`shared-renderer-contract/v1`) carried by the anchor doc.

1.2 WHEN the document defines edge derivation (PRD-STR-E02-AC-03) THEN it states the client derives an edge from `parent_node_id` but does NOT bind that projection to the shared renderer contract or the shared semantic-key helper, leaving room for local/downstream edge logic, alias stacking, or hardcoded edge construction.

1.3 WHEN the document models edges THEN it carries NO typed edge/socket model (no `socket_types` with `color` / `edgeWidthPx` / `handleStrokeWidthPx` / `accepts`), so edge typing has no neutral source contract.

1.4 WHEN the document specifies edge projection mechanics THEN it carries NO `flow.edgeType` (`smoothstep`), NO `flow.direction`, and NO per-node `handles` + `flow:portTypes`, so there is no shared, agnostic projection driver.

1.5 WHEN the document identifies graph/edge entities THEN it carries NO `buildScopedGraphSemanticKey` semantic-identity rule, so edge identity is undefined and may be re-derived or duplicated downstream.

1.6 WHEN the document assigns rendering responsibility THEN it carries NO `rendererPolicy` ("frontmatter and source payloads own data; renderers project view state only"), so a renderer could own data, re-calculate, or re-render edges rather than project view state.

1.7 WHEN the document constrains how edge rendering may be implemented THEN it does NOT explicitly forbid backfill, churn, conflict, duplicate, freeze, infinite loop, hardcode, legacy, re-calculation, re-computation, re-rendering, or stale state, and does not mandate removal of legacy/stale/conflicting code, hardcoded fixtures, and tests — permitting the bug to be patched downstream instead of neutralized at the source.

1.8 WHEN the anchor/demo doc declares the active 2D renderer THEN it pins a single `kgCanvas2dRenderer: "flowEditor"` and carries NO contract field declaring the supported 2D renderer set / capability, so edge rendering is implicitly tied to one renderer and is not stated to project identically when the active 2D renderer is `Storyboard` or `Strybldr`.

1.9 WHEN either document specifies edge projection across renderers THEN it permits (does not forbid) a renderer-specific edge code path, per-renderer hardcode, or aliasing/forking of edge logic per renderer, so switching the active 2D renderer (`flowEditor` | `Storyboard` | `Strybldr`) could recompute, re-render, duplicate, or staleify edges instead of projecting the same shared edge model.

1.10 WHEN the two documents are read together THEN Strytree's `parent_node_id`-derived edges and the demo's flow nodes/handles are NOT bound to one shared, renderer-agnostic edge projection, so the same logical edge could be projected differently (or recomputed) depending on which 2D renderer is active.

### Expected Behavior (Correct)

What the finetuned Strytree PRD/TAD (`F'`) must specify. Each clause corresponds to the same-numbered defect above.

2.1 WHEN the document specifies story-edge rendering THEN it SHALL bind edge rendering to the shared `kgSharedRendererContract` at `version: shared-renderer-contract/v1` (with `edgeModel: "active graph edges from the selected source graph"`), reusing the anchor doc's contract rather than redefining a local edge renderer.

2.2 WHEN the document defines edge derivation THEN it SHALL require edges to be derived from `parent_node_id` at the source/upstream and projected through the shared renderer contract + the shared semantic-key helper `buildScopedGraphSemanticKey`, and SHALL forbid local/downstream patches, alias stacking, and hardcoded edge logic.

2.3 WHEN the document models edges THEN it SHALL carry a typed edge/socket model equivalent to the anchor `socket_types` (each socket declaring `color`, `edgeWidthPx`, `handleStrokeWidthPx`, `accepts`), reused as the neutral edge-typing source.

2.4 WHEN the document specifies edge projection mechanics THEN it SHALL carry `flow.edgeType: smoothstep`, `flow.direction`, and per-node `handles` + `flow:portTypes` as the shared, agnostic edge-projection driver.

2.5 WHEN the document identifies graph/edge entities THEN it SHALL define edge/graph semantic identity via `buildScopedGraphSemanticKey`, so each edge has one scoped, reusable key and is never re-derived or duplicated downstream.

2.6 WHEN the document assigns rendering responsibility THEN it SHALL carry the `rendererPolicy` "frontmatter and source payloads own data; renderers project view state only", so the renderer projects view state only and performs no edge re-calculation, re-computation, or re-rendering of source-owned data.

2.7 WHEN the document constrains how edge rendering may be implemented THEN it SHALL explicitly forbid backfill, churn, conflict, duplicate, freeze, infinite loop, hardcode, legacy, re-calculation, re-computation, re-rendering, and stale state; SHALL require reuse of shared heuristics, the shared semantic-key helper, and headless/unopinionated renderer projection; SHALL mandate root/source/upstream neutralization with NO backward-compatibility remapping of legacy and removal of 100% of legacy/stale/conflicting code, hardcoded fixtures, and tests; and SHALL assert the "spec-complete → runtime-ready" + forbid-hardcode-in-repo principle under universality, neutrality, agnosticity, and modularity.

2.8 WHEN the anchor/demo doc declares the active 2D renderer THEN it SHALL carry a contract field that makes edge rendering renderer-agnostic — declaring the supported 2D renderer set / capability (`flowEditor` | `Storyboard` | `Strybldr`) as data the renderers project, not branch on — so that edge rendering is no longer tied to a single `kgCanvas2dRenderer` value and is stated to project identically for every supported 2D renderer.

2.9 WHEN either document specifies edge projection across renderers THEN it SHALL require edge projection to be driven by the same shared `kgSharedRendererContract` + `buildScopedGraphSemanticKey` + `socket_types` + flow port/handle model regardless of the active 2D renderer, and SHALL forbid any renderer-specific edge code path, per-renderer hardcode, or aliasing/forking of edge logic per renderer — so switching the active 2D renderer never recomputes, re-renders, duplicates, or staleifies edges.

2.10 WHEN the two documents are read together THEN they SHALL bind Strytree's `parent_node_id`-derived edges and the demo's flow nodes/handles to ONE shared, renderer-agnostic edge projection, so the same logical edge resolves to the same projected edge across `flowEditor` | `Storyboard` | `Strybldr` with no recomputation, re-rendering, duplication, or staleness on renderer switch.

### Unchanged Behavior (Regression Prevention)

Existing Strytree contract content that is NOT an edge-rendering-contract gap (`¬C(X)`) and MUST be preserved byte-for-identical-intent (`F(X) = F'(X)`).

3.1 WHEN the document defines edge derivation source THEN it SHALL CONTINUE TO derive one edge per node from non-null `parent_node_id` without a separate edge table (PRD-STR-E02-AC-03), and SHALL CONTINUE TO honor the constraint "story edges derive from `parent_node_id` unless a later graph index is justified".

3.2 WHEN the document constrains Strytree workbench dependencies THEN it SHALL CONTINUE TO forbid any new external graph-rendering dependency for the Strytree workbench (PRD-STR-E08-AC-04 and the matching frontmatter constraint).

3.3 WHEN the document constrains data storage THEN it SHALL CONTINUE TO forbid any hosted database dependency outside the Cloudflare topology.

3.4 WHEN the document constrains hosting THEN it SHALL CONTINUE TO forbid any alternate app hosting path outside `Dev -> Prod -> Cloudflare`.

3.5 WHEN the document specifies non-edge epics (PRD-STR-E01 access/identity, E03 wallet, E04 payment, E05 unlock/split, E06 PixVerse harness, E07 observability, E08 ForkCompare beyond its UI-surface clause) THEN it SHALL CONTINUE TO specify them exactly as in v0.2.1 with no behavioral change.

3.6 WHEN the document specifies the rendering surface THEN it SHALL CONTINUE TO use the existing Strybldr/Storyboard/Strytree SVG/HTML card surface and Cloudflare-native bindings (PRD-STR-E08-AC-04), introducing no alternate renderer surface while adopting the shared contract.

3.7 WHEN the document records the observed prototype (Part A source analysis) THEN it SHALL CONTINUE TO document the prototype's static edge behavior as historical source analysis, without that section becoming the target implementation contract.

3.8 WHEN the active 2D renderer is switched among `flowEditor`, `Storyboard`, and `Strybldr` THEN both documents SHALL CONTINUE TO leave all non-edge behavior unchanged — node content, card surfaces, compute/run actions, timeline/transport surfaces, and every non-edge field SHALL behave identically regardless of the active renderer.

3.9 WHEN the anchor/demo doc adopts the renderer-agnostic edge contract THEN it SHALL CONTINUE TO preserve all of its non-edge demo content exactly as it exists today — `agentic_canvas_os_demo`, `flow_diagrams`, node payloads, `compute` functions, `outputSrcDoc` previews, `socket_types` values, and every other non-edge field SHALL remain byte-for-identical-intent.

3.10 WHEN either document is finetuned for the renderer-agnostic edge contract THEN it SHALL CONTINUE TO preserve `parent_node_id` edge derivation with no edge table, introduce no new graph-rendering dependency, stay within the Cloudflare-only topology, keep the `Dev -> Prod -> Cloudflare` hosting path, and leave all non-edge epics unchanged.
