# Requirements Document

## Introduction

This document specifies requirements for **enhancing** the existing Knowgrph
Geospatial Mode. The current mode overlays a MapLibre basemap and configurable
geo layers on top of the 2D infinite canvas, supports 2D (MapLibre), 2D (SVG
fallback), and 3D (MapLibre) views, renders dataset points/lines/polygons and
graph nodes/edges, and preserves graph-first defaults (documented in
`docs/documents/knowgrph-geospatial-mode-document.md`).

This enhancement extends that surface with native `fill-extrusion` layers for
buildings and roads and a custom-layer path for arbitrary 3D assets, while
preserving every currently documented behavior. It keeps Knowgrph
browser-based, mobile-first, local-first, offline-first, and zero-infra. The
enhancement draws inspiration only from `opengeos/GeoLibre`, `visgl/deck.gl`,
`tauri-apps/tauri`, and `duckdb/duckdb-wasm`; it copies no code from those
projects and adds none of them as runtime dependencies.

**Implementation status (2026-07-27):** `runtime-ready-dev`. Structured
operator controls, focused source proof, the owned task-worktree diagnostic,
protected Integration Gate, canonical runtime supervision, and the repeated
browser flow pass on exact protected main
`6b381860cb2abd26cc2e37b84fd1bbc9cfa93896`. Production and Cloudflare
deployment remain separately authorized and unclaimed.

### Value lenses (solo-dev, AI-native)

- **Min-viable-max-value**: extend the existing MapLibre runtime path rather
  than add a parallel renderer.
- **TCO-zero**: no new paid infrastructure, no new metered egress, FOSS-first.
- **Token economics**: any AI-assisted geo authoring flows through a bounded
  harness with a per-call cost log.
- **Harness-first**: MCP, `/`, `@`, and `#` invocation surfaces route through
  the existing invocation contract.

### Non-negotiable boundaries

- This spec carries **development-authoring and validation authority only**.
  Production and Cloudflare deploys require a separate explicit instruction.
- The enhancement **must not replace** existing Geospatial Mode behavior; it is
  strictly additive and backward-compatible.
- No heavy GIS engine (server-side or client-side spatial database, routing
  engine, or full geoprocessing stack) is introduced.

## Glossary

- **Geospatial_Mode**: The existing Knowgrph canvas rendering mode that
  overlays a basemap and geo layers on the 2D canvas; the subject of this
  enhancement.
- **Overlay_Runtime**: The MapLibre-based runtime path (loaded on-demand from
  the sibling `gympgrph` module) that owns basemap lifecycle, layers, and
  interaction gating.
- **Extrusion_Layer**: A MapLibre native `fill-extrusion` layer that renders
  polygon features (buildings, road footprints) with height.
- **Custom_Layer**: A MapLibre custom layer (WebGL/WebGL2 draw hook) used to
  render arbitrary 3D assets registered by configuration.
- **Asset_3D**: A configuration-declared 3D asset (for example a mesh or model
  reference) positioned by geographic coordinates and rendered through a
  Custom_Layer.
- **Geo_Feature**: A GeoJSON Feature or record-derived feature carrying
  geometry and properties.
- **Height_Property**: A named numeric feature property that supplies extrusion
  height; resolved by configuration, not hardcoded.
- **Dataset_Config**: Configuration (environment catalog or UI input) that
  declares dataset URLs, layer types, and rendering options without embedding
  values in compiled code.
- **Enhanced_Layer_Editor**: The structured Floating Panel Geo surface that
  displays the effective catalog source and owns Add, Edit, Remove, Toggle, and
  reset actions for extrusion and 3D asset entries.
- **Invocation_Surface**: The MCP tool surface and the `/` (command), `@`
  (mention/reference), and `#` (tag) invocable entrypoints exposed by Knowgrph.
- **Geo_Harness**: A schema-validated wrapper around any AI/model call used for
  geo authoring that emits a per-call cost log and defines a fallback path.
- **Fetch_Bound**: The configurable `timeoutMs` and `maxBytes` limits that cap
  dataset and asset retrieval. Enhanced resources use an effective deadline of
  `min(timeoutMs, 10_000)`.
- **Session_Cache**: The current-tab, in-memory, byte-accounted LRU for enhanced
  resource bytes; bounded to 32 MiB and 32 entries.
- **Operator**: The solo developer or end user interacting with Geospatial_Mode
  in the browser.
- **Reference_Project**: One of `opengeos/GeoLibre`, `visgl/deck.gl`,
  `tauri-apps/tauri`, `duckdb/duckdb-wasm`, used for inspiration only.

## Requirements

### Requirement 1: Preserve existing Geospatial Mode behavior

**User Story:** As a maintainer, I want the enhancement to extend rather than
replace current behavior, so that existing datasets, views, and interactions
keep working unchanged.

#### Acceptance Criteria

1. THE Geospatial_Mode SHALL provide the three documented view selections
   (2D MapLibre, 2D SVG fallback, 3D MapLibre) without removing, renaming, or
   reordering any of them.
2. WHERE no Extrusion_Layer and no Custom_Layer is configured, THE
   Geospatial_Mode SHALL produce the same view selections, layer ordering, and
   camera and interaction controls as those defined in
   `knowgrph-geospatial-mode-document.md`.
3. WHILE no geo import has occurred in the active session, THE Geospatial_Mode
   SHALL keep the overlay disabled.
4. WHEN a geo import completes, THE Geospatial_Mode SHALL auto-enable the
   overlay as defined in `knowgrph-geospatial-mode-document.md`.
5. WHILE Geospatial_Mode is active, THE Geospatial_Mode SHALL suppress
   knowledge-graph rendering as defined in
   `knowgrph-geospatial-mode-document.md`.
6. THE Overlay_Runtime SHALL load on-demand from the extracted `gympgrph`
   module without embedding Geospatial_Mode logic in the host codebase.
7. IF the on-demand load of the `gympgrph` module fails, THEN THE
   Overlay_Runtime SHALL leave the overlay disabled, retain the current view
   selection unchanged, and surface an error indication that the overlay could
   not be loaded.

### Requirement 2: Native fill-extrusion for buildings and roads

**User Story:** As an Operator, I want buildings and roads rendered with height,
so that I can explore 3D geographic context without a separate renderer.

#### Acceptance Criteria

1. WHERE a Dataset_Config declares an Extrusion_Layer for polygon features,
   THE Overlay_Runtime SHALL render those features using a MapLibre native
   `fill-extrusion` layer.
2. WHEN an Extrusion_Layer references a Height_Property, THE Overlay_Runtime
   SHALL read the extrusion height in meters from the named feature property
   resolved by Dataset_Config, accepting numeric values in the range 0 to
   10,000 meters.
3. IF a Geo_Feature lacks the configured Height_Property, THEN THE
   Overlay_Runtime SHALL apply the Extrusion_Layer configured default height and
   render the feature without raising an error and without skipping the feature.
4. IF a Geo_Feature's Height_Property value is non-numeric, negative, or greater
   than 10,000 meters, THEN THE Overlay_Runtime SHALL apply the Extrusion_Layer
   configured default height, render the feature, and record a diagnostic
   indicating the invalid height value while preserving all other features in
   the layer.
5. WHERE a Dataset_Config declares road features for extrusion, THE
   Overlay_Runtime SHALL render road footprints as Extrusion_Layer polygons
   using the same Height_Property resolution and default-height fallback path
   defined for building features.
6. THE Overlay_Runtime SHALL resolve each Extrusion_Layer paint option
   (Height_Property name, base height in meters, fill color, and fill opacity as
   a value from 0.0 to 1.0) from Dataset_Config, using the configured value when
   present and the documented default when absent, without hardcoding
   dataset-specific values in compiled code.

### Requirement 3: Custom-layer path for 3D assets

**User Story:** As an Operator, I want to place arbitrary 3D assets on the map,
so that I can represent landmarks and models beyond extruded polygons.

#### Acceptance Criteria

1. WHERE a Dataset_Config declares an Asset_3D whose latitude is within -90 to
   90 degrees and whose longitude is within -180 to 180 degrees, THE
   Overlay_Runtime SHALL render the asset through a MapLibre Custom_Layer
   positioned at the asset's geographic coordinates.
2. WHEN the map camera changes through a pan, zoom, pitch, or bearing operation,
   THE Overlay_Runtime SHALL reproject each rendered Asset_3D within the same
   render frame so that the asset's rendered screen position corresponds to its
   geographic coordinates within a tolerance of 1 pixel.
3. IF an Asset_3D fails to load within Fetch_Bound, THEN THE Overlay_Runtime
   SHALL surface an error indication identifying the failed asset and the
   failure reason, retain any already-loaded layers, and continue rendering the
   remaining layers.
4. IF an Asset_3D declares coordinates with latitude outside -90 to 90 degrees
   or longitude outside -180 to 180 degrees, THEN THE Overlay_Runtime SHALL skip
   that asset, surface an error indication identifying the asset and the invalid
   coordinate value, and continue rendering the remaining layers.
5. THE Overlay_Runtime SHALL create at most one Custom_Layer render context per
   enable-cycle so that in-flight resource loads are not cancelled.
6. WHERE no Asset_3D is configured, THE Overlay_Runtime SHALL NOT initialize a
   Custom_Layer render context.

### Requirement 4: No heavy GIS engine and inspiration-only dependencies

**User Story:** As a maintainer, I want the enhancement to stay lightweight and
FOSS-first, so that total cost of ownership and bundle weight stay low.

#### Acceptance Criteria

1. THE Geospatial_Mode SHALL implement extrusion and 3D asset rendering using
   the existing MapLibre runtime and source-authored code only, and SHALL NOT
   require any additional rendering-engine runtime dependency.
2. THE Geospatial_Mode SHALL NOT introduce a client-side or server-side spatial
   database, routing engine, or full geoprocessing stack as a runtime dependency
   listed in the project dependency manifest.
3. THE Geospatial_Mode SHALL NOT add `opengeos/GeoLibre`, `visgl/deck.gl`,
   `tauri-apps/tauri`, or `duckdb/duckdb-wasm` to the project dependency manifest
   as a runtime (non-development) dependency.
4. THE Geospatial_Mode SHALL contain only source-authored implementation, and
   SHALL NOT copy code, schemas, or assets from any Reference_Project.
5. WHERE a Reference_Project informs a design choice, THE design documentation
   SHALL record the inspiration in a dedicated references section as a
   non-binding reference rather than as a project dependency.
6. THE Geospatial_Mode SHALL add no more than 250 KB (gzipped) to the production
   client bundle relative to the pre-enhancement baseline bundle size.
7. IF a build or dependency-audit step detects any prohibited runtime dependency
   named in criteria 2 or 3, THEN THE build process SHALL fail and SHALL produce
   an error indicating the prohibited dependency and its source, and SHALL NOT
   publish the affected build artifact.

### Requirement 5: Preserve runtime constraints (browser, mobile, local, offline)

**User Story:** As an Operator on a phone with intermittent connectivity, I want
3D geo rendering to work locally and offline, so that I can use it anywhere.

#### Acceptance Criteria

1. THE Geospatial_Mode SHALL render Extrusion_Layer and Custom_Layer content in
   a browser runtime without requiring any connection to a backend service.
2. WHILE the device viewport width is 768 CSS pixels or less, THE
   Overlay_Runtime SHALL render extrusion and 3D asset layers and SHALL respond
   to touch-based pan, zoom, pitch, and bearing gestures.
3. WHERE the requested datasets and assets are already present in Session_Cache,
   THE Geospatial_Mode SHALL render Extrusion_Layer and Custom_Layer content
   without issuing any network request.
4. WHEN a same-origin dataset path (a path beginning with `/`) is provided, THE
   Overlay_Runtime SHALL load extrusion and asset data using only same-origin
   requests and SHALL NOT issue a cross-origin request.
5. IF a network request required for a layer is unavailable, THEN THE
   Geospatial_Mode SHALL render all layers whose data is available in
   Session_Cache.
6. IF a network request required for a layer is unavailable, THEN THE
   Geospatial_Mode SHALL surface a status message, of at most 140 characters,
   that identifies each unavailable layer and indicates the reason as
   `network-unavailable`, while retaining any already-rendered cached layers.
7. THE Session_Cache SHALL hold no more than 32 MiB and 32 entries, SHALL evict
   least-recently-used entries, and SHALL revalidate each caller's current
   `maxBytes` on a cache hit. Page reload or eviction MAY require retrieval
   again; persistent offline storage is outside this requirement.

### Requirement 6: Configuration-driven and bounded ingestion

**User Story:** As a maintainer, I want extrusion and asset behavior driven by
configuration with bounded fetches, so that no dataset is hardcoded and no fetch
runs unbounded.

#### Acceptance Criteria

1. THE Geospatial_Mode SHALL resolve declarations at initialization using exact
   precedence: a present `kg:ui:geospatial:enhancedLayers` value, otherwise
   `VITE_GEOSPATIAL_DATASETS_JSON`, otherwise an empty catalog. An invalid
   environment value SHALL fail closed with an `invalid-config` diagnostic, and
   THE compiled code SHALL contain zero dataset or asset URLs.
2. WHEN the Overlay_Runtime initiates any dataset or asset retrieval, THE
   Overlay_Runtime SHALL apply configured `maxBytes` and an effective timeout of
   `min(timeoutMs, 10_000)` to that retrieval.
3. IF a required Fetch_Bound value (`timeoutMs` or `maxBytes`) is absent from
   Dataset_Config, THEN THE Overlay_Runtime SHALL abort the affected retrieval
   before any network request and surface an error identifying the missing
   configuration key, retaining the previously loaded overlay state unchanged.
4. IF a dataset or asset retrieval transfers more than the configured
   `maxBytes`, THEN THE Overlay_Runtime SHALL terminate that retrieval before
   completion, discard the partial payload, and surface an error identifying the
   exceeded byte limit and the affected dataset or asset.
5. IF a dataset or asset retrieval does not complete within the effective
   timeout, THEN THE Overlay_Runtime SHALL cancel that retrieval, discard any
   partial payload, prevent a late cache write, and surface an error identifying
   the effective timeout and the affected dataset or asset.
6. WHEN a polygon dataset whose size is within the configured `maxBytes` is
   ingested for extrusion, THE Overlay_Runtime SHALL derive geometry through the
   existing bounded parse path without blocking user interaction with the map
   beyond the effective timeout.
7. WHILE running on localhost, THE Overlay_Runtime SHALL route extrusion and
   asset retrievals through the documented dev-only cross-origin proxy.
8. WHILE running in a production or static deploy, THE Overlay_Runtime SHALL
   load extrusion and asset resources directly without the dev-only cross-origin
   proxy.

### Requirement 7: Invocation surfaces (MCP, `/`, `@`, `#`)

**User Story:** As an Operator, I want to control the enhanced geo layers through
Knowgrph's invocation surfaces, so that geo actions compose with the rest of the
runtime.

#### Acceptance Criteria

1. THE Invocation_Surface SHALL expose MCP tool commands to enable
   Geospatial_Mode, to set an Extrusion_Layer visibility state (visible or
   hidden), and to set an Asset_3D visibility state (visible or hidden).
2. WHERE a `/` command targets Geospatial_Mode, THE Invocation_Surface SHALL
   route the command to the Overlay_Runtime through the existing interaction
   gating path rather than invoking the Overlay_Runtime directly.
3. WHEN an `@` reference resolves to a geo-capable node, THE Invocation_Surface
   SHALL fit the Overlay_Runtime camera to that node's geographic bounds.
4. WHERE a `#` tag selects a dataset or layer group, THE Invocation_Surface
   SHALL toggle the visibility of every tagged Extrusion_Layer and Asset_3D in
   that group.
5. WHEN a layer is toggled or Geospatial_Mode is changed through any
   Invocation_Surface, THE Invocation_Surface SHALL emit the documented
   `GEOSPATIAL_MODE_CHANGED_EVENT` within 500 milliseconds so host UI stays
   synchronized.
6. IF a `/` command targets Geospatial_Mode with an unrecognized action or
   target, THEN THE Invocation_Surface SHALL reject the command with an
   actionable error and SHALL NOT change Overlay_Runtime state.
7. IF an `@` reference resolves to a node without resolvable geographic bounds,
   THEN THE Invocation_Surface SHALL leave the Overlay_Runtime camera unchanged
   and SHALL surface an actionable error.
8. IF a `#` tag matches no Extrusion_Layer or Asset_3D, THEN THE
   Invocation_Surface SHALL make no visibility change and SHALL surface an
   actionable error.

### Requirement 8: AI-assisted geo authoring harness (optional)

**User Story:** As an Operator, I want optional AI assistance for authoring geo
layers, so that I can generate extrusion or asset configuration with bounded,
observable cost.

#### Acceptance Criteria

1. WHERE an AI-assisted geo authoring action is invoked, THE Geo_Harness SHALL
   validate the supplied input against its defined input schema before
   initiating any model call.
2. IF input schema validation fails, THEN THE Geo_Harness SHALL return a
   structured error identifying each failing field and SHALL NOT initiate any
   model call.
3. WHEN the Geo_Harness completes a model call, THE Geo_Harness SHALL validate
   the output against its defined output schema and SHALL emit a cost log
   containing model identifier, prompt token count, completion token count,
   cache hit count, and estimated cost.
4. IF the Geo_Harness output schema validation fails, THEN THE Geo_Harness SHALL
   return a structured error indicating the schema violation and SHALL NOT apply
   the invalid configuration, retaining the prior configuration unchanged.
5. WHERE a Geo_Harness call is an agentic loop, THE Geo_Harness SHALL enforce a
   configured maximum iteration bound in the range 1 to 50 iterations
   (default 10) and SHALL exit via its circuit-breaker when the iteration bound
   is reached.
6. WHILE a Geo_Harness invocation is executing, IF the cumulative estimated cost
   reaches the configured per-invocation cost budget, THEN THE Geo_Harness SHALL
   halt further model calls, return a structured error indicating the budget was
   exceeded, and SHALL NOT apply partial configuration.
7. IF a model call does not return within the configured model-call timeout in
   the range 1 to 300 seconds (default 30 seconds), THEN THE Geo_Harness SHALL
   treat the call as unavailable and SHALL activate its fallback path.
8. IF a model call is unavailable, THEN THE Geo_Harness SHALL return
   `ok:false`, a typed `model-unavailable` error rather than a raw model failure,
   and a deterministic non-null fallback draft whose dataset and render entries
   are disabled and whose source URL is empty.
9. WHEN the unavailable-model fallback is returned, THE Geo_Harness SHALL NOT
   call `applyDraft`; an Operator must explicitly review and apply the draft.

### Requirement 9: Reliability and visibility of enhanced layers

**User Story:** As an Operator, I want enhanced layers to be visible and stable,
so that I never face a silent blank or invisible-layer state.

#### Acceptance Criteria

1. IF an Extrusion_Layer or Custom_Layer does not complete loading within 10
   seconds or returns a load error, THEN THE Overlay_Runtime SHALL surface the
   failure via the documented in-app toast within 1 second, rather than
   rendering silently blank, and SHALL retain any already-loaded layers.
2. WHILE an Extrusion_Layer or Asset_3D is loading and Content-Length is
   available, THE Overlay_Runtime SHALL surface determinate progress as a
   percentage from 0 to 100 of bytes received.
3. WHILE an Extrusion_Layer or Asset_3D is loading and Content-Length is not
   available, THE Overlay_Runtime SHALL surface an indeterminate loading
   indicator.
4. WHEN Geospatial_Mode is enabled, IF the persisted overlay opacity is zero,
   THEN THE Overlay_Runtime SHALL restore an opacity of 1.0 (on a 0.0 to 1.0
   scale) so enhanced layers cannot be enabled but invisible.
5. WHEN a style, load, or resize event occurs, THE Overlay_Runtime SHALL force
   viewport-sized overlay layout and call `map.resize()` within 1 second so that
   extrusion and asset layers have a non-zero drawable area.
6. WHEN a geo-capable graph selection exists in 3D render mode, THE
   Overlay_Runtime SHALL use the selection bounds as the auto-fit target so
   enhanced layers stay aligned with Zoom-to-Selection.
7. WHEN no geo-capable graph selection exists in 3D render mode, THE
   Overlay_Runtime SHALL use the full extent of the loaded layers as the
   auto-fit target.
8. THE development source-readiness command SHALL execute the focused
   geospatial, editor, persistence, Canvas, and MCP suites and SHALL validate an
   ordered evidence manifest for all 44 correctness properties before reporting
   source success. It SHALL NOT report runtime readiness without the browser
   evidence required by Requirement 10.

### Requirement 10: Operator-managed enhanced-layer catalog

**User Story:** As an Operator, I want to configure and toggle enhanced layers
from the Geo panel, so that environment variables and direct localStorage edits
are not required for normal use.

#### Acceptance Criteria

1. WHEN the Operator opens Floating Panel Geo, THE Enhanced_Layer_Editor SHALL
   display an **Enhanced layers** catalog, an effective-source badge whose value
   is **Environment**, **Local**, or **Empty**, and one row per effective
   Extrusion_Layer and Asset_3D. The catalog SHALL expose
   `data-kg-geo-enhanced-config-source` for deterministic browser proof.
2. EACH catalog row SHALL display its stable ID, kind, visibility status, **Edit**,
   **Remove**, and a keyboard-operable visible/hidden **Toggle**, using
   `Toggle enhanced layer <id>`, `Edit enhanced layer <id>`, and
   `Remove enhanced layer <id>` as accessible names.
3. WHEN the Operator selects **Add enhanced layer**, THE
   Enhanced_Layer_Editor SHALL
   support building extrusion, road extrusion, and 3D asset drafts with ID, URL,
   mandatory `timeoutMs` and `maxBytes`, and all kind-specific render fields.
4. WHEN the Operator saves a valid Add or Edit, THE Enhanced_Layer_Editor SHALL
   atomically write the complete catalog to
   `kg:ui:geospatial:enhancedLayers`, report **Local** as the effective source,
   emit the enhanced-layer change event, and update the mounted Overlay_Runtime
   in the same tab without reload.
5. WHEN the Operator confirms **Remove**, THE Enhanced_Layer_Editor SHALL remove
   exactly that entry and its stale visibility override, retain all siblings,
   persist the complete result including `[]` when the final local entry is
   intentionally removed, and update the mounted Overlay_Runtime without
   reload.
6. WHEN the Operator changes a row Toggle, THE Enhanced_Layer_Editor SHALL
   persist only that ID under
   `kg:ui:geospatial:enhancedLayerVisibility`, emit the shared change event, and
   hide or show the mounted layer plus its UI status within 500 milliseconds
   without reload.
7. WHEN the Operator confirms **Reset to environment defaults**, THE
   Enhanced_Layer_Editor SHALL remove the local catalog and all enhanced
   visibility overrides, SHALL NOT write `[]`, SHALL re-resolve environment
   then empty precedence, and SHALL refresh the catalog and mounted runtime
   without reload.
8. IF a draft has a duplicate or blank ID, invalid URL, missing or non-positive
   Fetch_Bound, or invalid kind-specific field, THEN THE Enhanced_Layer_Editor
   SHALL show field-level actionable errors, identify the first invalid field,
   perform no storage, event, or runtime mutation, and retain the previously
   rendered catalog.
9. THE Enhanced_Layer_Editor SHALL remain usable with labelled keyboard
   controls at desktop and mobile floating-panel widths.
10. BEFORE protected integration, THE runtime workflow SHALL record an owned
    fresh-origin task-worktree diagnostic for Add/Edit/Remove,
    persistence-after-reload, Toggle, invalid-draft, reset, and mobile
    behavior. BEFORE the enhancement is marked runtime-ready, THE workflow
    SHALL repeat the required assertions at the exact integrated main SHA with
    no critical page or request failure. The unsealed task-worktree diagnostic
    SHALL NOT establish runtime readiness.
