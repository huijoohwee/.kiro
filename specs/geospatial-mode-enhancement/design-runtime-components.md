# Geospatial Runtime Components and Interfaces

## Configuration model

The normalized runtime contract remains discriminated and URL-neutral:

```ts
type FetchBound = {
  timeoutMs: number
  maxBytes: number
}

type ExtrusionLayerConfig = {
  id: string
  datasetId: string
  url: string
  kind: 'building' | 'road'
  heightProperty: string
  defaultHeightMeters: number
  baseHeightMeters: number
  fillColor: string
  fillOpacity: number
  tags: readonly string[]
  visible: boolean
  fetchBound: FetchBound
}

type Asset3DConfig = {
  id: string
  url: string
  lat: number
  lng: number
  altitudeMeters: number
  scale: number
  rotationDegrees: number
  tags: readonly string[]
  visible: boolean
  fetchBound: FetchBound
}
```

`normalizeEnhancedConfig` accepts operator-authored records and returns
extrusions, assets, and typed diagnostics. Invalid entries are skipped without
invalidating siblings. Heights, base height, color, opacity, scale, coordinates,
and tags are normalized in their owning package.

## Configuration source

`enhancedLayerConfigSource.ts` is a pure source resolver. It receives the raw
localStorage value and the statically addressable Vite environment value and
returns:

```ts
type EnhancedLayerConfigSource = {
  raw: unknown
  source: 'local-storage' | 'environment' | 'default'
  invalidEnvironmentValue?: string
}
```

`enhancedLayerPersistence.ts` owns browser access and applies visibility
overrides after normalization. The environment key is
`VITE_GEOSPATIAL_DATASETS_JSON`; the local key is
`kg:ui:geospatial:enhancedLayers`.

## Structured catalog editor

`useEnhancedLayerCatalog.ts` adapts the source resolver and normalized runtime
contract for UI use, while `enhancedLayerEditorModel.ts` owns drafts and field
validation:

```ts
type EnhancedLayerEditorState = {
  source: 'environment' | 'local' | 'empty'
  entries: readonly EnhancedLayerEditorEntry[]
  diagnostics: readonly EnhancedLayerEditorDiagnostic[]
}

type EnhancedLayerEditorAction =
  | { kind: 'add'; draft: EnhancedLayerDraft }
  | { kind: 'edit'; originalId: string; draft: EnhancedLayerDraft }
  | { kind: 'remove'; id: string }
  | { kind: 'visibility'; id: string; visible: boolean }
  | { kind: 'reset-environment' }
```

The editor state owner converts every effective extrusion and asset into a
round-trippable draft and validates required IDs, unique IDs, URLs, mandatory
positive fetch bounds, and kind-specific values. It returns field-addressable
errors and the first invalid field. Validation has no storage, event, or runtime
side effect.

`EnhancedLayerCatalogPanel.tsx` and `EnhancedLayerEditorForm.tsx` render the
source badge, catalog rows, and Add/Edit/Remove/Toggle/Reset controls. They
delegate state transitions through `useEnhancedLayerCatalog.ts` and
`gympgrphBridge.ts` and never access localStorage. `GeospatialPanelHost.tsx`
injects the catalog through `GeospatialPanelDatasetControls.tsx`; display
controls and shared KTV UI remain in `GeospatialPanelDisplayControls.tsx` and
`geospatialPanelUi.tsx`. Every runtime file stays below 600 lines.

Valid Add/Edit/Remove writes are atomic full-catalog replacements. Removing the
final local entry intentionally persists `[]`. A visibility action changes only
`kg:ui:geospatial:enhancedLayerVisibility`. Reset is different: it removes the
local catalog and visibility keys, then resolves the environment catalog or
empty source. Every successful action emits the enhanced-layer change event so
the mounted same-tab runtime refreshes without reload.

The source badge values are **Environment**, **Local**, and **Empty**. All
controls have stable accessible names, keyboard operation, and a responsive
single-column form at mobile floating-panel widths.

## Fetch bounds and resource cache

`loadBoundedResource` accepts:

```ts
{
  target: string
  url: string
  bound: FetchBound
  onProgress?: (progress: LoadProgress) => void
  cacheOnly?: boolean
}
```

It returns either copied bytes plus `fromCache`, or a typed failure:

- `missing-fetch-bound`;
- `max-bytes-exceeded`;
- `timeout`;
- `network-unavailable`;
- `parse-failed`.

The loader checks bounds before URL resolution and fetch. It races network work
against the effective readiness deadline, aborts the request, cancels an active
reader, and rejects any post-deadline completion before the cache write.

`ByteBoundedResourceCache` separately owns byte accounting and LRU ordering.
Its global instance is bounded to 32 MiB and 32 entries. A cached payload larger
than a caller's current `maxBytes` returns `max-bytes-exceeded`; it is never
silently reused.

## Extrusion path

`normalizeExtrusionFeatures` copies each feature and writes normalized render
properties. Missing, non-numeric, negative, or over-10,000-meter source heights
use the configured fallback. The feature remains present and a diagnostic is
recorded.

`ensureExtrusionLayer` waits for a loaded style, creates one GeoJSON source, and
uses a native MapLibre layer:

```ts
{
  type: 'fill-extrusion',
  paint: {
    'fill-extrusion-height': ['get', 'kgExtrusionHeightM'],
    'fill-extrusion-base': ['min', baseHeight, ['get', 'kgExtrusionHeightM']],
    'fill-extrusion-color': configuredColor,
    'fill-extrusion-opacity': configuredOpacity
  }
}
```

Road and building entries share the same height and paint owners.

## Asset descriptor

The runtime accepts one source-authored JSON format:

```json
{
  "schemaId": "knowgrph-geo-asset-mesh/v1",
  "positions": [0, 0, 0, 1, 0, 0, 0, 1, 0],
  "indices": [0, 1, 2],
  "color": [0.6, 0.65, 0.7, 1]
}
```

Positions are finite z-up model-space meters. Indices are triangles and remain
within unsigned-16-bit and vertex bounds. Color has four components in `[0,1]`.
Malformed descriptors are rejected before GL allocation.

## Per-frame asset projection

`computeAssetFrameMatrix` accepts the active MapLibre transform, the current
`CustomRenderMethodInput`, and one normalized asset. It validates all inputs,
requests `getMatrixForModel` for the current projection, composes the frame and
z-up local matrices in 64-bit precision, and converts the final finite matrix to
`Float32Array` for WebGL.

No manual Mercator anchor or flat-projection compatibility path remains.

## Custom-layer lifecycle

`createAsset3DCustomLayer` returns `null` when there is no renderable asset.
Otherwise it returns one `CustomLayerInterface` and one handle:

```ts
type Asset3DLayerHandle = {
  readonly id: string
  readonly contextId: string
  setVisible(assetId: string, visible: boolean): void
  dispose(): void
}
```

`onAdd` compiles one program and creates owned buffers. `render` computes every
visible asset's frame matrix and draws it while restoring MapLibre's GL state in
a `finally` block. `onRemove` releases live-context resources. If a context is
lost, references are discarded without invalid delete calls; a later `onAdd`
recreates resources.

## Enhanced-layer orchestration

`useEnhancedGeospatialLayers` owns one enable-cycle:

1. read normalized configuration;
2. emit configuration diagnostics;
3. load each extrusion and asset independently;
4. report determinate or indeterminate progress;
5. retain successful siblings when one fails;
6. install native extrusion layers and one asset custom layer;
7. publish ready IDs and asset context on the map container for diagnostics;
8. combine loaded bounds for fit behavior;
9. dispose custom-layer ownership on disable or teardown.

When the overlay becomes enabled and stored opacity is zero, it writes `1`.
Messages are limited to 140 characters.

## Fit precedence

`applyGeospatialFitRequest` handles explicit bounds and current-location
requests first. Selection fit uses:

```text
selected bounds -> graph bounds -> enhanced bounds
```

General fit uses:

```text
graph bounds -> enhanced bounds
```

All fit calls use zero duration at this owner so tests and invocations remain
deterministic.

## Command model

```ts
type GeoCommand =
  | { kind: 'mode.set'; enabled: boolean }
  | { kind: 'extrusion.visibility'; layerId: string; visible: boolean }
  | { kind: 'asset.visibility'; assetId: string; visible: boolean }
  | { kind: 'tag.visibility'; tag: string; visible: boolean }
  | { kind: 'fit.node'; nodeId: string }
```

The envelope schema is `knowgrph-geospatial-command/v1`.

`geoInvocationDispatcher.ts` parses and validates commands.
`geoInvocationRuntime.ts` resolves graph data and current enhanced config.
`gympgrphBridge.ts` is the sole mutation gateway. It writes through extracted
package actions and shared events; invocation code never receives a MapLibre
map reference.

## Chat collision rules

- `/geo` and `/geospatial` require a command boundary and are claimed locally.
- `#tag` requires exactly `show` or `hide`.
- `@node-id` is claimed only when a graph node with that ID exists.
- Unknown actions, unknown layers, missing bounds, and unmatched tags return
  typed rejections without mutation.
- Unrelated `/`, `@`, and `#` input falls through to existing chat/document
  invocation behavior.

The local geospatial activator runs before generic provider URL resolution, so
valid commands require no provider configuration or network call.

## MCP deep-link flow

The MCP tool exposes exactly the required commands: mode state, extrusion
visibility, and asset visibility. It returns a validated envelope and a local
Canvas URL containing `kgGeo=1` and `kgGeoCommand`.

The Canvas host claims the query once, removes the command payload from browser
history, and runs the envelope through the same graph-aware command runtime.
Malformed queries fail closed. React StrictMode cannot execute the same claim
twice.

## Authoring flow

`runGeoAuthoring` accepts intent, dataset ID, layer kind, iteration bound, cost
budget, and model timeout. It returns one of:

- a validated applied draft;
- a structured validation, budget, iteration, or output error;
- a typed model-unavailable error plus a deterministic disabled fallback.

Each completed model call creates and validates a canonical cost log. Cost and
iteration checks occur before any apply. Model-unavailable paths do not call
`applyDraft`; the operator must explicitly review and commit a fallback.
