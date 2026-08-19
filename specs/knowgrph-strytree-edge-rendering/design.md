# Knowgrph Strytree Edge Rendering Bugfix Design

## Overview

This is a **two-document contract edit**, not a code change. The bug is a contract gap: the Strytree PRD/TAD (`knowgrph/docs/documents/knowgrph-strytree-prd-tad.md`, v0.2.1, `status: implementation-contract`) only *describes* story edges in prose and prototype-calculation terms, and the anchor/demo doc (`huijoohwee/docs/knowgrph-agentic-canvas-os-demo.md`) pins a single active 2D renderer. Neither document binds edge rendering to **one shared, renderer-agnostic contract**. Because the PRD/TAD is the implementation contract, the document gap is what *permits* non-neutral, hardcoded, or downstream-patched edge rendering "in the first place" — that permission is the defect.

The fix neutralizes the root cause at the source: both documents are finetuned so every edge-rendering concern is bound to ONE shared contract surface already carried by the anchor doc:

- `kgSharedRendererContract@shared-renderer-contract/v1` (semantic identity, edge model, renderer policy)
- `buildScopedGraphSemanticKey` (the shared semantic-key helper / edge identity rule)
- `socket_types` (typed edge/socket model)
- the `flow` port/handle model (`flow.edgeType`, `flow.direction`, per-node `handles` + `flow:portTypes`)

Strytree's `parent_node_id`-derived edges and the demo's flow nodes/handles are projected through that **same** contract, **renderer-agnostically** across `flowEditor | Storyboard | Strybldr`. The fix is surgical: it adds contract blocks, reclassifies prototype edge prose as historical source-analysis, and removes 100% of conflicting/legacy edge specification with NO backward-compatibility remapping — while leaving every non-edge epic and all non-edge demo content byte-for-identical-intent.

The strategy is deliberately **additive-at-source + reclassify + neutralize**, never per-renderer fork, never local/downstream patch, never hardcode, and introduces no new graph-rendering dependency, staying inside the Cloudflare-only topology and the `Dev -> Prod -> Cloudflare` hosting path.

## Glossary

- **Bug_Condition (C)**: An edge-rendering concern (a clause/section of either fix-target doc that governs how edges are rendered) that is NOT bound to the shared renderer contract + shared semantic-key helper, OR permits a forbidden non-neutral mechanism, OR is not renderer-agnostic across the three 2D renderers.
- **Property (P)**: The desired finetuned state — edges sourced upstream (`parent_node_id` for Strytree, flow nodes+handles for the demo) and projected identically through the shared contract for every supported renderer.
- **Preservation**: Existing non-edge contract content (non-edge epics, prototype source-analysis, all non-edge demo content) and existing edge-derivation invariants (one edge per non-null `parent_node_id`, no edge table) that must remain unchanged by the fix.
- **kgSharedRendererContract**: The document-owned, renderer-agnostic rendering contract block (`version: shared-renderer-contract/v1`) carried in the anchor doc frontmatter; `semanticIdentity: buildScopedGraphSemanticKey`, `edgeModel: "active graph edges from the selected source graph"`, `rendererPolicy: "frontmatter and source payloads own data; renderers project view state only"`.
- **buildScopedGraphSemanticKey**: The shared semantic-key helper that assigns one scoped, reusable identity per edge/graph entity so edges are never re-derived or duplicated downstream.
- **socket_types**: Typed edge/socket model (`idea_signal`, `evidence_signal`, `approval_signal`, `artifact_signal`), each declaring `color`, `edgeWidthPx`, `handleStrokeWidthPx`, `accepts`.
- **flow port/handle model**: `flow.edgeType` (`smoothstep`), `flow.direction`, per-node `handles`, and per-node `flow:portTypes` — the shared, agnostic edge-projection driver.
- **kgCanvas2dRenderer**: The frontmatter field naming the active 2D renderer (currently `"flowEditor"` in the demo doc).
- **rendererAgnostic**: The predicate that edges project identically for every renderer in `{flowEditor, Storyboard, Strybldr}`, with the renderer set declared as *data the renderers project*, not branch on — no renderer-specific edge path, no per-renderer hardcode, no per-renderer fork.
- **F**: The current pair of documents (edge rendering only described locally; contract not declared renderer-agnostic; demo pinned to a single renderer).
- **F'**: The finetuned pair of documents (edge rendering sourced upstream and projected identically through the shared contract, renderer-agnostically).

## Bug Details

### Bug Condition

The bug manifests in any clause or section of either fix-target document that *governs edge rendering* while failing to bind that rendering to the shared, neutral renderer contract — or while permitting a forbidden non-neutral mechanism (hardcode, local/downstream patch, alias stacking, re-computation, legacy/stale remapping) — or while tying edge rendering to a single 2D renderer instead of projecting edges identically across `flowEditor | Storyboard | Strybldr`. The PRD/TAD currently does this descriptively (prototype Bezier prose, parent-derived edge prose with no contract binding, no typed socket model, no `flow` projection fields, no semantic-key identity, no renderer policy, no forbid/cleanup rules), and the anchor/demo doc does this by pinning `kgCanvas2dRenderer: "flowEditor"` with no declared renderer-set capability.

**Formal Specification:**
```
FUNCTION isBugCondition(X)
  INPUT: X of type EdgeRenderingConcern   // a clause/section of either fix-target doc governing edge rendering
  OUTPUT: boolean

  RETURN governsEdgeRendering(X)
         AND ( ( NOT boundToSharedRendererContract(X)      // kgSharedRendererContract@shared-renderer-contract/v1
                 AND NOT usesSharedSemanticKeyHelper(X) )  // buildScopedGraphSemanticKey
             OR permitsForbiddenMechanism(X)               // hardcode | local/downstream patch | alias | re-compute | legacy remap
             OR NOT rendererAgnostic(X) )                  // renderer-specific edge path / per-renderer hardcode / fork per renderer
END FUNCTION

FUNCTION rendererAgnostic(X)
  INPUT: X of type EdgeRenderingConcern
  OUTPUT: boolean

  RETURN projectsIdenticallyFor(X, {"flowEditor", "Storyboard", "Strybldr"})
         AND NOT hasRendererSpecificEdgePath(X)
         AND NOT hasPerRendererHardcode(X)
         AND NOT forksEdgeLogicPerRenderer(X)
END FUNCTION
```

### Examples

- **Prototype Bezier prose (PRD/TAD Part A, A1 "Prototype Calculation Engine" table row "Edge shape | SVG cubic Bezier from parent card right edge to child card left edge")** — *Expected*: documented as historical source analysis only. *Actual (F)*: sits in the implementation contract with no marker separating it from the runtime contract, so it reads as a permitted target rendering rule (`isBugCondition = true` if treated as contract; neutralized by reclassification).
- **Parent-derived edge clause (PRD/TAD PRD-STR-E02-AC-03 + Part C "Edge Strategy")** — *Expected*: derive one edge per non-null `parent_node_id` AND project it through `kgSharedRendererContract` + `buildScopedGraphSemanticKey`. *Actual (F)*: derives the edge but does not bind the projection to the shared contract, leaving room for local/downstream/hardcoded edge logic (`isBugCondition = true`).
- **No typed socket model (PRD/TAD frontmatter)** — *Expected*: typed `socket_types` carried as the neutral edge-typing source. *Actual (F)*: absent, so edge typing has no neutral source contract (`isBugCondition = true`).
- **Single-renderer pin (demo doc frontmatter `kgCanvas2dRenderer: "flowEditor"`)** — *Expected*: a capability field declaring the supported 2D renderer set as projected data so edges render identically for `Storyboard`/`Strybldr` too. *Actual (F)*: pinned to one renderer with no agnostic capability declared (`isBugCondition = true`).
- **Edge case — non-edge content (e.g., PRD-STR-E03 wallet, demo `compute` functions)**: does NOT govern edge rendering, so `governsEdgeRendering(X) = false` ⇒ `isBugCondition = false` ⇒ MUST be preserved unchanged.

## Expected Behavior

### Preservation Requirements

**Unchanged Behaviors (¬C(X) — must satisfy F(X) = F'(X)):**
- Strytree edge derivation invariant: one edge per node from non-null `parent_node_id`, NO separate edge table (PRD-STR-E02-AC-03; Part C "Edge Strategy"; the `constraints` line "story edges derive from parent_node_id unless a later graph index is justified").
- The "no new external graph-rendering dependency for the Strytree workbench" constraint (PRD-STR-E08-AC-04 + matching frontmatter constraint).
- The "no hosted database dependency outside the Cloudflare topology" constraint.
- The "no alternate app hosting path outside `Dev -> Prod -> Cloudflare`" constraint and `deployment_topology`/`cloudflare_route`.
- All non-edge epics: PRD-STR-E01 (access/identity), E03 (wallet), E04 (payment), E05 (unlock/split), E06 (PixVerse harness), E07 (observability), E08 (ForkCompare beyond its UI-surface clause).
- The existing Strybldr/Storyboard/Strytree SVG/HTML card surface and Cloudflare-native bindings (PRD-STR-E08-AC-04); no alternate renderer surface introduced.
- The Part A prototype source-analysis content stays as historical source analysis (not target contract).
- All non-edge demo content: `agentic_canvas_os_demo`, `flow_diagrams`, node payloads, `compute` functions, `outputSrcDoc` previews, `socket_types` *values*, and every other non-edge field — byte-for-identical-intent. The demo body sections (`## Response`, `## Rich Media Outputs`, `## Inputs`, `## Guardrails`) are untouched.
- Switching the active 2D renderer among `flowEditor`/`Storyboard`/`Strybldr` MUST leave all non-edge behavior identical (node content, card surfaces, compute/run actions, timeline/transport surfaces, every non-edge field).

**Scope:**
All concerns that do NOT govern edge rendering are completely unaffected by this fix. This includes wallet/ledger/payment/unlock/harness/observability epics, prototype source-analysis prose (except its reclassification marker), the demo's compute pipeline and body, and all topology/hosting/dependency constraints.

**Note:** The actual expected correct edge-rendering behavior is defined in the Correctness Properties section (Property 1). This section enumerates what must NOT change.

## Hypothesized Root Cause

Based on the bug analysis, the contract gap has these root causes:

1. **Missing shared-contract binding in the PRD/TAD**: The implementation contract carries no `kgSharedRendererContract`, no `buildScopedGraphSemanticKey` identity rule, no `socket_types`, and no `flow` projection fields. Edge rendering is therefore specified only in prose, which *permits* a local/downstream/hardcoded edge renderer to exist without violating the contract.
   - Prototype prose ("SVG cubic Bezier...") is co-located with the runtime contract with no reclassification marker.
   - PRD-STR-E02-AC-03 derives edges from `parent_node_id` but does not bind the *projection* to the shared contract.

2. **No renderer-projection policy / responsibility assignment**: Neither document carries `rendererPolicy: "frontmatter and source payloads own data; renderers project view state only"` for Strytree edges, so a renderer could own data, re-calculate, re-compute, or re-render edges instead of projecting view state.

3. **Single-renderer pin in the anchor/demo doc**: `kgCanvas2dRenderer: "flowEditor"` with no capability field declaring the supported 2D renderer set. The renderer set is not declared as *data the renderers project*, so edge rendering is implicitly tied to one renderer and not stated to project identically under `Storyboard`/`Strybldr`.

4. **No forbid/cleanup rules**: Neither document forbids backfill, churn, conflict, duplicate, freeze, infinite loop, hardcode, legacy, re-calculation, re-computation, re-rendering, or stale state, nor mandates source/upstream neutralization with removal of legacy/stale/conflicting edge spec and NO backward-compat remapping. This permits downstream patching instead of source neutralization.

5. **No unifying cross-document rule**: Strytree's `parent_node_id`-derived edges and the demo's flow nodes/handles are not explicitly bound to ONE shared, renderer-agnostic edge projection, so the same logical edge could project differently (or recompute) per active renderer.

## Correctness Properties

Property 1: Bug Condition - Shared, Renderer-Agnostic Edge Projection

_For any_ edge-rendering concern X where the bug condition holds (`isBugCondition(X)` returns true), the finetuned documents (F') SHALL bind X so that `boundToSharedRendererContract(X)` AND `usesSharedSemanticKeyHelper(X)` (`buildScopedGraphSemanticKey`) AND `derivesEdgesFrom(X, "parent_node_id")` (Strytree, at source/upstream) AND `projectsViewStateOnly(X)` (renderers do not own data, no re-compute) AND `rendererAgnostic(X)` (identical projection for `flowEditor | Storyboard | Strybldr`) AND `forbidsNonNeutralMechanisms(X)` (no hardcode/legacy/alias/patch/re-render/stale) all hold.

**Validates: Requirements 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 2.7, 2.8, 2.9, 2.10**

Property 2: Preservation - Non-Edge Contract and Demo Content

_For any_ concern X where the bug condition does NOT hold (`isBugCondition(X)` returns false — non-edge epics, prototype source-analysis, non-edge demo content, topology/hosting/dependency constraints, and the existing `parent_node_id`/no-edge-table derivation invariant), the finetuned documents SHALL produce the same content as the original, i.e. `F(X) = F'(X)`, preserving all non-edge contract clauses and all non-edge demo content byte-for-identical-intent, including under any active-renderer switch among `flowEditor | Storyboard | Strybldr`.

**Validates: Requirements 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 3.9, 3.10**

## Fix Implementation

### Changes Required

Assuming the root cause analysis is correct, the fix is a set of surgical, source-level contract edits. Two principles govern every edit: **(a)** edges remain a *pure projection* of source-owned data (`parent_node_id` for Strytree; flow nodes/handles for the demo); **(b)** the renderer set is declared as *data the renderers project*, never branched on.

---

#### File 1: `knowgrph/docs/documents/knowgrph-strytree-prd-tad.md` (v0.2.1 → v0.2.2)

**1A. Frontmatter — add the shared renderer contract binding** (insert after the `related:` block, keeping all existing frontmatter keys intact; bump `version` to `0.2.2` and `updated`):

```yaml
kgCanvas2dRendererCapability:
  supportedRenderers: ["flowEditor", "Storyboard", "Strybldr"]
  selectionModel: "projected-data"          # renderers project this set; they do not branch on it
  edgeProjectionInvariance: "identical-across-supportedRenderers"
kgSharedRendererContract:
  version: "shared-renderer-contract/v1"
  semanticIdentity: "buildScopedGraphSemanticKey"
  edgeModel: "active graph edges from the selected source graph"
  edgeSource: "strytree_nodes.parent_node_id"   # source/upstream derivation, no edge table
  rendererPolicy: "frontmatter and source payloads own data; renderers project view state only"
socket_types:
  idea_signal: {color: "#14b8a6", edgeWidthPx: 2, handleStrokeWidthPx: 2, accepts: [idea_signal]}
  evidence_signal: {color: "#22c55e", edgeWidthPx: 2, handleStrokeWidthPx: 2, accepts: [evidence_signal]}
  approval_signal: {color: "#f59e0b", edgeWidthPx: 3, handleStrokeWidthPx: 3, accepts: [approval_signal]}
  artifact_signal: {color: "#8b5cf6", edgeWidthPx: 2, handleStrokeWidthPx: 2, accepts: [artifact_signal]}
flow:
  direction: "LR"
  edgeType: "smoothstep"
  # Per-node handles + flow:portTypes are the shared, agnostic edge-projection driver.
  # For Strytree, each node carries a single inbound handle keyed to its parent_node_id-derived edge.
  storyEdgeProjection:
    handleModel: "per-node source/target handles derived from parent_node_id"
    portTypeDefault: "idea_signal"            # story-edge semantic mapping (single typed projection)
    semanticKeyRule: "buildScopedGraphSemanticKey(storyId, parentNodeId, childNodeId)"
edgeContractForbid:
  - "backfill"
  - "churn"
  - "conflict"
  - "duplicate"
  - "freeze"
  - "infinite-loop"
  - "hardcode"
  - "legacy"
  - "re-calculation"
  - "re-computation"
  - "re-rendering"
  - "stale-state"
  - "renderer-specific-edge-path"
  - "per-renderer-hardcode"
  - "alias-stacking"
  - "local-or-downstream-patch"
  - "backward-compat-remap"
edgeContractCleanup:
  rule: "root/source/upstream neutralization; remove 100% of legacy/stale/conflicting edge spec, hardcoded fixtures, and tests; NO backward-compatibility remapping"
edgeContractPrinciples: ["universality", "neutrality", "agnosticity", "modularity", "spec-complete-runtime-ready", "forbid-hardcode-in-repo"]
```

> **Why frontmatter:** the contract must be machine-projectable and document-owned (matching the anchor doc), so the renderer projects view state from declared data rather than from prose. The `parent_node_id`-derived edge maps onto the shared port/handle/socket projection via `storyEdgeProjection` — it does NOT invent a parallel local edge model and does NOT add a separate edge table.

**1B. Part A — reclassify prototype edge prose as historical source analysis** (do NOT delete; mark it):
- Add a one-line banner at the top of `## A1. Observed Strytree Prototype`: *"Part A records observed prototype behavior as historical source analysis only. It is NOT the target implementation contract; the runtime edge contract is defined in the frontmatter `kgSharedRendererContract` + `flow.storyEdgeProjection` and in Part C 'Edge Rendering Contract'."*
- In the `### Prototype Calculation Engine` table, append "(historical prototype only)" to the `Edge existence` and `Edge shape | SVG cubic Bezier...` rows so they can never be read as the runtime contract.

**1C. Part B — bind PRD-STR-E02-AC-03 to the shared contract** (extend, do not weaken the existing derivation invariant):
- Keep the existing AC text verbatim (one edge per non-null `parent_node_id`, no edge table — this is preserved per 3.1).
- Add a sentence: *"The derived edge SHALL be projected through `kgSharedRendererContract@shared-renderer-contract/v1` using `buildScopedGraphSemanticKey` for identity and the `flow` port/handle/`socket_types` model for typing; local/downstream patches, alias stacking, and hardcoded edge logic are forbidden."*

**1D. Part C — add a new "Edge Rendering Contract" subsection** under C4, immediately after `### Edge Strategy` (the `parent_node_id` SSOT statement is preserved verbatim above it):
- State the renderer-projection policy (renderers project view state only; no re-calculation/re-computation/re-rendering of source-owned edges).
- State the renderer-agnostic rule: the same shared `kgSharedRendererContract` + `buildScopedGraphSemanticKey` + `socket_types` + flow port/handle model drives projection regardless of the active 2D renderer; forbid any renderer-specific edge code path, per-renderer hardcode, or per-renderer fork.
- State the cross-document unification rule: Strytree `parent_node_id`-derived edges and the demo flow nodes/handles resolve to the SAME shared edge projection across `flowEditor | Storyboard | Strybldr` with no recompute/re-render/duplicate/stale on renderer switch.

**1E. Part C — update C3 "Storytree SVG Renderer" responsibility row**: change "Renders nodes, parent-derived edges, pan, zoom..." to clarify it *projects* parent-derived edges through the shared contract (view state only), not owns/recomputes them. (Edge-governing row only; all other C3 rows untouched.)

**1F. Frontmatter `constraints` — add neutralization/agnosticity constraints** (append; keep all existing constraints byte-identical):
- `"story edge rendering must bind to kgSharedRendererContract@shared-renderer-contract/v1 and buildScopedGraphSemanticKey; no local/downstream/hardcoded edge logic"`
- `"story edge projection must be renderer-agnostic across flowEditor | Storyboard | Strybldr; no per-renderer edge path, hardcode, or fork"`

---

#### File 2: `huijoohwee/docs/knowgrph-agentic-canvas-os-demo.md`

**2A. Frontmatter — add the renderer-agnostic capability field** (insert immediately after `kgCanvas2dRenderer: "flowEditor"`; the existing `kgCanvas2dRenderer` line is preserved as the *default active* renderer, not the *only* renderer):

```yaml
kgCanvas2dRendererCapability:
  supportedRenderers: ["flowEditor", "Storyboard", "Strybldr"]
  selectionModel: "projected-data"          # renderers project this set; they do not branch on it
  edgeProjectionInvariance: "identical-across-supportedRenderers"
```

**2B. Frontmatter — extend the existing `kgSharedRendererContract` block** with one line declaring renderer-agnostic edge projection (all existing keys preserved byte-for-identical-intent):

```yaml
  rendererAgnosticEdges: "edges project identically for every supportedRenderer; no renderer-specific edge path"
```

> **No other demo edits.** The `flow.edges` list, all `flow.nodes`, `handles`, `flow:portTypes`, `socket_types` values, `agentic_canvas_os_demo`, `flow_diagrams`, `compute` functions, `outputSrcDoc` previews, and the entire document body (`## Response`, `## Rich Media Outputs`, `## Inputs`, `## Guardrails`) remain unchanged. The capability is declared as *projected data*, not a branch, so the existing edge list resolves identically regardless of the active `kgCanvas2dRenderer`.

### Legacy / Stale / Conflicting Cleanup Inventory

| Location | Current text | Classification | Action |
|---|---|---|---|
| PRD/TAD Part A, `Delivery Shape` row "Graph rendering \| SVG story tree with HTML cards via `foreignObject`" | Descriptive prototype | Historical source analysis | Keep; covered by Part A reclassification banner (3.7) |
| PRD/TAD Part A, `Prototype Calculation Engine` row "Edge existence \| `parentId` exists" | Prototype calc | Historical source analysis | Mark "(historical prototype only)" |
| PRD/TAD Part A, `Prototype Calculation Engine` row "Edge shape \| SVG cubic Bezier from parent card right edge to child card left edge" | Prototype calc prose | Historical source analysis (the prime conflicting/legacy edge spec) | Mark "(historical prototype only)"; runtime contract is now `flow.storyEdgeProjection` |
| PRD/TAD PRD-STR-E02-AC-03 | Parent-derived edge, no edge table | Runtime contract (incomplete binding) | Extend with shared-contract binding (1C); derivation invariant preserved |
| PRD/TAD Part C `Edge Strategy` | `parent_node_id` SSOT, no `story_edges` table | Runtime contract (preserved) | Keep verbatim; add "Edge Rendering Contract" subsection beneath it |
| PRD/TAD C3 "Storytree SVG Renderer" responsibility | "Renders ... parent-derived edges" | Runtime contract (ownership ambiguous) | Reword to "projects ... through shared contract (view state only)" |
| Demo doc `kgCanvas2dRenderer: "flowEditor"` | Single-renderer pin | Runtime contract (incomplete) | Keep as default; add capability field (2A) + agnostic line (2B) |

**100% of conflicting/legacy edge *specification* is neutralized** by reclassification (Part A prototype rows become explicitly historical) and by replacement of prose-only runtime edge spec with the frontmatter contract. There is **NO backward-compatibility remapping**: the prototype Bezier rule is not carried forward as an alternate runtime path, and no alias bridges old prose to new contract.

## Testing Strategy

### Validation Approach

Because both fix targets are documents (contracts), "tests" are **document contract checks**: deterministic predicates evaluated over the document AST/frontmatter and text. The approach is two-phase: first surface counterexamples that demonstrate the contract gap on the UNFIXED documents, then verify the fix closes every gap (Fix Checking) and leaves all non-edge content unchanged (Preservation Checking).

### Exploratory Bug Condition Checking

**Goal**: Surface counterexamples on the UNFIXED documents that prove edge-rendering concerns are not bound to the shared, renderer-agnostic contract. Confirm or refute the root cause analysis; if refuted, re-hypothesize.

**Test Plan**: Parse both documents' frontmatter/text and assert, for each edge-rendering concern, that it is bound to `kgSharedRendererContract` + `buildScopedGraphSemanticKey`, derives from `parent_node_id` at source, projects view state only, is renderer-agnostic, and forbids non-neutral mechanisms. Run on UNFIXED docs to observe failures.

**Test Cases**:
1. **PRD/TAD shared-contract presence** — assert `kgSharedRendererContract` + `socket_types` + `flow` exist in PRD/TAD frontmatter (will fail on unfixed code).
2. **PRD/TAD semantic-key binding** — assert PRD-STR-E02-AC-03 references `buildScopedGraphSemanticKey` (will fail on unfixed code).
3. **PRD/TAD forbid/cleanup rules** — assert the forbid list + cleanup rule exist (will fail on unfixed code).
4. **Demo renderer-agnostic capability** — assert the demo doc declares a supported-renderer-set capability and `rendererAgnosticEdges` (will fail on unfixed code).
5. **Cross-renderer projection invariance** — assert both docs state identical projection across `{flowEditor, Storyboard, Strybldr}` (will fail on unfixed code).
6. **Prototype reclassification** — assert Part A Bezier/edge-existence rows are marked historical (may fail on unfixed code).

**Expected Counterexamples**:
- PRD/TAD frontmatter has no `kgSharedRendererContract` / `socket_types` / `flow` keys.
- PRD-STR-E02-AC-03 does not reference the shared semantic-key helper.
- Demo doc pins `kgCanvas2dRenderer: "flowEditor"` with no capability field.
- Possible causes (per hypothesis): missing contract binding, missing renderer policy, single-renderer pin, missing forbid/cleanup rules, missing unifying rule.

### Fix Checking

**Goal**: Verify that for all edge-rendering concerns where the bug condition holds, the finetuned documents produce the expected behavior.

**Pseudocode:**
```
FOR ALL X WHERE isBugCondition(X) DO
  result := F_prime(X)
  ASSERT boundToSharedRendererContract(result)
     AND usesSharedSemanticKeyHelper(result)         // buildScopedGraphSemanticKey
     AND derivesEdgesFrom(result, "parent_node_id")  // Strytree, at source/upstream
     AND projectsViewStateOnly(result)               // no re-compute / no data ownership
     AND rendererAgnostic(result)                    // identical for flowEditor | Storyboard | Strybldr
     AND forbidsNonNeutralMechanisms(result)         // no hardcode/legacy/alias/patch/re-render/stale
END FOR
```

### Preservation Checking

**Goal**: Verify that for all concerns where the bug condition does NOT hold, the finetuned documents produce the same content as the originals.

**Pseudocode:**
```
FOR ALL X WHERE NOT isBugCondition(X) DO
  ASSERT F(X) = F_prime(X)
END FOR
```

**Testing Approach**: Property-based testing is recommended for preservation checking because it generates many concerns across both documents' input domain, catches edge cases manual checks miss, and gives strong guarantees that non-edge content is unchanged. The `rendererAgnostic` predicate is exercised across all three renderers `{flowEditor, Storyboard, Strybldr}` so projection invariance is checked, not assumed.

**Test Plan**: Snapshot the UNFIXED non-edge content (non-edge epics, prototype source-analysis text, all non-edge demo fields and body), then assert byte-for-identical-intent after the fix; additionally assert the derivation invariant (one edge per non-null `parent_node_id`, no edge table) is unchanged.

**Test Cases**:
1. **Non-edge epic preservation** — snapshot PRD-STR-E01/E03/E04/E05/E06/E07 and E08 (non-UI clauses) on unfixed docs, assert unchanged after fix.
2. **Derivation invariant preservation** — assert PRD-STR-E02-AC-03 still derives one edge per non-null `parent_node_id` with no edge table; assert Part C "Edge Strategy" unchanged.
3. **Topology/hosting/dependency preservation** — assert the three constraints (no new graph-rendering dep; no DB outside Cloudflare; no hosting outside `Dev -> Prod -> Cloudflare`) are unchanged.
4. **Demo non-edge preservation** — snapshot `agentic_canvas_os_demo`, `flow_diagrams`, node payloads, `compute`, `outputSrcDoc`, `socket_types` values, and the body sections; assert unchanged.
5. **Renderer-switch non-edge invariance** — assert both docs state non-edge behavior is identical across renderer switches.

### Unit Tests

- Frontmatter contract presence: PRD/TAD carries `kgSharedRendererContract`, `socket_types`, `flow`, `kgCanvas2dRendererCapability`, `edgeContractForbid`, `edgeContractCleanup`.
- PRD-STR-E02-AC-03 references `buildScopedGraphSemanticKey` and preserves the no-edge-table derivation invariant.
- Part A prototype edge rows are marked historical.
- Demo doc carries `kgCanvas2dRendererCapability.supportedRenderers == ["flowEditor","Storyboard","Strybldr"]` and `rendererAgnosticEdges`.

### Property-Based Tests

- For randomly sampled edge-rendering concerns across both docs, assert Property 1 (bound to shared contract, semantic-key identity, parent-derived, view-state-only, renderer-agnostic, forbids non-neutral mechanisms).
- For randomly sampled non-edge concerns, assert Property 2 (`F(X) = F'(X)`).
- For the `rendererAgnostic` predicate, generate the renderer set `{flowEditor, Storyboard, Strybldr}` and assert projection invariance for every member; assert no renderer-specific path/hardcode/fork is introduced.

### Integration Tests

- Cross-document unification: assert Strytree `parent_node_id`-derived edges and the demo flow nodes/handles both resolve to the SAME shared edge projection contract (`shared-renderer-contract/v1` + `buildScopedGraphSemanticKey` + `socket_types` + flow port/handle model).
- Renderer-switch flow: assert that switching the declared active renderer among the three supported values changes no edge projection and no non-edge content in either document.
- Spec-complete → runtime-ready: assert the PRD/TAD frontmatter now carries every field a renderer needs to project Strytree edges without prose interpretation, with no new graph-rendering dependency and within the Cloudflare-only / `Dev -> Prod -> Cloudflare` topology.
