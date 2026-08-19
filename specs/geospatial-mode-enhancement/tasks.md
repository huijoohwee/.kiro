# Geospatial Mode Enhancement Tasks

## Status

**Runtime-ready-dev on protected main
`6b381860cb2abd26cc2e37b84fd1bbc9cfa93896`.**

This ledger separates implementation, source proof, the unsealed owned-worktree
browser diagnostic, protected integration, and exact-main proof. The feature
is `runtime-ready-dev` because every required checkbox below is complete with
evidence from the exact revision named by the protected gate.

## A. Canonical contract

- [x] Define the structured Floating Panel Geo **Enhanced layers** catalog.
- [x] Define Environment, Local, and Empty source-badge semantics.
- [x] Define Add/Edit/Remove, per-layer Toggle, and
  **Reset to environment defaults** behavior.
- [x] Define atomic persistence, mutation-free field validation, same-tab
  without-reload updates, and keyboard/mobile requirements.
- [x] Extend correctness properties from 38 to 44.
- [x] Separate source readiness, candidate browser proof, exact-main proof, and
  deployment authority.

## B. Runtime controls

- [x] Split `GeospatialPanelHost.tsx` so it and every new UI owner remain below
  600 lines.
- [x] Implement `canvas/src/features/geospatial/enhancedLayerEditorModel.ts` as
  the owner of draft conversion and field validation.
- [x] Implement `canvas/src/features/geospatial/useEnhancedLayerCatalog.ts` as
  the owner of source projection and typed editor actions.
- [x] Implement
  `canvas/src/features/geospatial/EnhancedLayerCatalogPanel.tsx` and
  `EnhancedLayerEditorForm.tsx` with the source badge and accessible rows/forms.
- [x] Implement structured Add/Edit forms for building extrusion, road
  extrusion, and 3D asset entries.
- [x] Implement atomic Remove with sibling retention and stale per-ID visibility
  cleanup.
- [x] Implement live per-layer Toggle through the persistence owner and shared
  change event.
- [x] Implement **Reset to environment defaults** by removing catalog and
  visibility overrides, never by writing `[]`.
- [x] Confirm every successful action updates the mounted runtime in the same
  tab without reload.

## C. Executable source proof

- [x] Add focused tests for effective source badge and catalog projection.
- [x] Assert `data-kg-geo-enhanced-config-source` and the stable row action
  accessible names in panel integration tests.
- [x] Add focused tests for atomic Add/Edit/Remove and persistence after reload.
- [x] Add focused tests for isolated Toggle writes and same-tab runtime events.
- [x] Add focused tests for reset-to-environment precedence and default
  visibility restoration.
- [x] Add focused tests proving invalid fields show actionable errors and
  perform no storage, event, or runtime mutation.
- [x] Add keyboard-native and mobile-width panel integration coverage.
- [x] Inventory every editor/UI owner in the readiness file-size and
  hardcoded-URL guard.
- [x] Update the ordered readiness manifest to Properties 1-44 using markers
  from tests that the source-readiness command actually executes.
- [x] Run the focused tests, production build, and final
  `check-geospatial-mode-readiness.mjs` guard at the owned candidate
  SHA.

## D. Owned task-worktree browser diagnostic

- [x] Record the owned branch, fence SHA, and worktree; start from a fresh origin
  seeded with one environment extrusion and one environment asset.
- [x] Verify the Environment badge and both effective rows.
- [x] Add and Edit a local entry through structured controls; verify the Local
  badge, same-tab readiness, and persistence after reload.
- [x] Hide and show exactly one layer with its row Toggle; verify runtime and UI
  update within 500 milliseconds without reload.
- [x] Remove one entry; verify its sibling remains and reload preserves the
  result.
- [x] Submit duplicate/invalid input; verify field-level errors, first-invalid
  focus, unchanged storage after reload, and retained rendered layers.
- [x] Reset to environment defaults; verify the environment catalog and default
  visibility return without reload and remain after reload.
- [x] Confirm native labelled controls and repeat interaction at a 390 × 844
  floating-panel viewport.
- [x] Record native extrusion/asset readiness markers, active 3D/globe
  selection, and zero critical browser logs.

This diagnostic used
`agent/huis-macbook-pro-3/geospatial-layer-controls`, worktree
`/Users/huijoohwee/Documents/GitHub/.worktrees/knowgrph/geospatial-layer-controls`,
and the repository-owned fence lineage ending at `67073ce4f85ee1eac7728df168d4e19d01eda2bd`.
It remains unsealed evidence and does not establish runtime readiness.

## E. Protected integration and exact-main gate

- [x] Integrate through the repository-owned protected workflow without a
  manual commit, push, or deployment shortcut.
- [x] Record the exact integrated main SHA.
- [x] Repeat every Section D interaction against that exact main SHA.
- [x] Confirm the exact-main proof is not a relabelled candidate artifact.
- [x] Keep production, Cloudflare, public-route, and physical-device proof
  explicitly unclaimed unless separately authorized.

PR `#448` merged the feature at `78df3a5544a7e042e9701fbb226a291adf713258`.
The protected build then exposed the repository-wide 2 GiB Vite heap cliff.
PR `#452` fixed that upstream pipeline owner with a bounded 4 GiB heap, and the
exact-main Integration Gate passed at
`6b381860cb2abd26cc2e37b84fd1bbc9cfa93896`.

The repository-owned runtime supervisor then proved Apex `5173`, storage
`8787`, and the storage proxy at HTTP 200 from that exact SHA. A fresh browser
flow added `exact-main-building`, observed
`data-kg-enhanced-layer-ids="exact-main-building"` on the mounted map, hid and
showed the row within 500 milliseconds, rejected a duplicate with first-invalid
focus and no second row, retained the row and runtime marker after reload,
repeated access and Toggle at 390 × 844, and removed the row and marker through
the two-step reset. Reload preserved the Empty source. There were no browser
errors; one non-critical MapLibre pre-style warning was recorded after reset.

## Runtime-ready transition

- [x] All Sections B-E are complete and evidence-linked.
- [x] `requirements.md`, design splits, this ledger, and the Kiro projection
  record `runtime-ready-dev` with 44 properties. The canonical protected-main
  document retains its pre-proof candidate boundary at the verified SHA rather
  than introducing an unproved post-proof source revision.
- [x] Change status to `runtime-ready-dev` only after the exact-main browser gate
  passes with no required work remaining.
