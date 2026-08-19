# Geospatial Correctness Properties

## Evidence policy

These properties are invariants, not a mandate that every invariant use the
same test technique. The readiness workflow combines:

- generative `fast-check` properties where the input space is algebraic;
- deterministic unit and lifecycle tests;
- Canvas registry integration tests;
- MCP contract tests;
- source and build guards for structural boundaries;
- clean-profile browser proof for the rendered runtime.

`scripts/geospatial-readiness-properties.json` is the ordered machine-readable
map from Properties 1-44 to stable evidence markers. The readiness guard fails
if an ID, file, or marker is missing. It does not convert a source marker into a
claim of browser proof.

## Baseline and extrusion

### Property 1: Zero-configuration baseline equivalence

For any absent, empty, or non-array enhanced catalog, normalization returns no
enhanced extrusion or asset. No custom-layer context is allocated.

### Property 2: Failed on-demand load rolls back mode state

If enabling the extracted runtime fails, the previous enabled preference is
restored, the current view-mode key is not written, and an error reaches the
caller.

### Property 3: Extrusions are native

Every configured extrusion render layer has MapLibre type `fill-extrusion`.

### Property 4: Height resolution is total and feature-preserving

For every input polygon feature and any source property value, normalization
returns exactly one feature with a finite height in `[0, 10_000]`.

### Property 5: Roads and buildings share height semantics

Changing `kind` between `road` and `building` does not change named-property
resolution, fallback classification, or feature retention.

### Property 6: Paint follows configuration

Height property, base height, color, opacity, and fallback height resolve from
normalized configuration, using documented neutral defaults only when absent
or invalid.

## Asset projection and lifecycle

### Property 7: Geographic anchoring uses MapLibre's active transform

Each asset frame requests `getMatrixForModel([lng, lat], altitudeMeters)` with
the normalized geographic values.

### Property 8: Camera and projection changes reproject in the same frame

Two render calls with different active model or frame matrices produce matrices
that reflect the respective frame; no previous flat matrix is reused.

### Property 9: Asset load failure is isolated

One failed or malformed asset does not discard valid meshes or loaded
extrusions. Only successfully parsed meshes enter the custom layer.

### Property 10: Unsafe asset input fails closed

Out-of-range coordinates, non-finite transforms, non-positive scale, and
malformed meshes are skipped without allocating or corrupting a GL context.

### Property 11: One custom context per enable cycle

An enable cycle installs at most one asset custom layer. Replacement disposes
the previous handle only after the new handle is created.

### Property 12: Heavy runtime dependencies are prohibited

Readiness fails if a prohibited GIS/rendering engine appears in runtime
dependencies.

## Retrieval, cache, and visibility

### Property 13: A valid cache hit issues no request

When the URL exists in the cache and satisfies the caller's current byte bound,
the loader returns a copy with `fromCache=true` without invoking `fetch`.

### Property 14: Same-origin paths remain same-origin

An absolute same-origin path resolves on the current origin and is never routed
through the localhost remote proxy.

### Property 15: Partial availability renders the successful subset

Independent layer tasks return on their own success or failure path. A failed
task does not clear ready IDs, source data, meshes, or bounds owned by siblings.

### Property 16: Failure status is bounded and explicit

Every enhanced-layer message is no longer than 140 characters. Network
classification contains the literal `network-unavailable`.

### Property 17: Both retrieval bounds are mandatory

Every normalized enhanced request has positive `timeoutMs` and `maxBytes`.

### Property 18: Missing bounds cause no request

If either bound is missing or non-positive, the loader returns
`missing-fetch-bound` before URL resolution or `fetch`.

### Property 19: Oversized payloads leave no partial value

If Content-Length or streamed bytes exceed `maxBytes`, the reader is cancelled,
the partial bytes are discarded, and the cache is unchanged.

### Property 20: Slow retrievals stop at the effective deadline

The result returns at `min(configured timeoutMs, 10_000)` with a typed timeout,
and a late network completion cannot write the cache.

### Property 21: Proxy routing depends on origin and environment

Only a localhost cross-origin resource uses `/__fetch_remote`; same-origin and
production/static retrievals use their direct resolved URL.

## Invocation

### Property 22: All command writes cross the bridge

Chat and MCP execution call only `GeoCommandBridge`; neither receives a map
reference or writes localStorage directly.

### Property 23: At references fit exact graph bounds

For an existing geo-capable node, `@node-id` fits `[lng, lat, lng, lat]`. An
existing node without bounds returns `no-geo-bounds` without camera mutation.

### Property 24: Hash actions affect exactly matched tags

`#tag show|hide` writes one tag command after confirming at least one configured
entry matches the normalized tag. No match performs no write.

### Property 25: Successful writes publish synchronization

Mode and enhanced visibility mutations synchronously publish the shared
geospatial event after their namespaced state write.

### Property 26: Rejection is mutation-free

Unknown actions, targets, bounds, and tags produce actionable rejections and
zero bridge writes.

## Optional authoring

### Property 27: Invalid input spends no model calls

Input validation failure returns before invoking the adapter.

### Property 28: Completed calls produce valid cost logs

Every completed adapter response creates one schema-valid log with model,
prompt tokens, completion tokens, cache hits, and estimated cost.

### Property 29: Invalid output is never applied

An invalid cost log or draft returns `output-invalid`; `applyDraft` is not
called.

### Property 30: Iteration bounds are clamped and enforced

The configured maximum is clamped to `[1,50]`. Repeated continuation responses
cannot exceed that count and end with `iteration-limit`.

### Property 31: Budget exhaustion is atomic

When cumulative estimated cost reaches the invocation budget, the harness
returns `budget-exceeded` before applying any draft.

### Property 32: Model timeout activates the deterministic fallback

Timeouts are clamped to `[1,300]` seconds and return `model-unavailable` plus a
disabled draft; no apply occurs.

### Property 33: Transport failure is typed and redacted

Raw provider failures are replaced by a bounded typed message and the same
deterministic disabled fallback.

## Host reliability

### Property 34: Load failure is visible and non-destructive

Each load failure produces an in-app error notification and retains
already-loaded layers.

### Property 35: Progress is bounded or explicitly indeterminate

Known-length progress is clamped to `[0,100]`. Unknown length reports
`indeterminate` plus received bytes.

### Property 36: Enabled opacity is nonzero

Enabling with persisted opacity `0` restores the persisted value to `1`.

### Property 37: The map has a drawable surface

Host layout, style/load/resize handling, and blank-map fallback keep a
viewport-sized map surface and prevent a silent zero-height overlay.

### Property 38: Fit follows selection precedence

Selection requests resolve `selected -> graph -> enhanced`; general fit resolves
`graph -> enhanced`. An explicit bounds request uses its exact bounds.

## Operator catalog and proof

### Property 39: The effective catalog is reachable in the Geo panel

For every environment, local, or empty effective source, opening Floating Panel
Geo renders one **Enhanced layers** catalog, the matching source badge, and
exactly one row per effective extrusion or asset. No environment-only or
localStorage-only path is required to reach the catalog. The catalog exposes
`data-kg-geo-enhanced-config-source` for deterministic browser assertions.

### Property 40: Add, edit, and remove are atomic and persistent

Every valid Add/Edit/Remove produces one complete valid local catalog write and
one same-tab change notification. Reload reproduces the saved catalog. Removing
one ID preserves all siblings and clears only that ID's stale visibility
override; intentionally removing the final local entry persists `[]`.

### Property 41: Reset restores source precedence

**Reset to environment defaults** removes both local enhanced-layer keys, never
writes `[]`, and immediately resolves the environment catalog or the empty
catalog. Reload produces the same source badge, entries, and default visibility.

### Property 42: Per-layer Toggle is live and isolated

Changing one row Toggle writes only that ID's visibility override, publishes the
shared change event, and updates the mounted layer and UI status within 500
milliseconds without reload. All non-target layer visibility remains unchanged.

### Property 43: Invalid editor input is visible and mutation-free

Duplicate or blank IDs, invalid URLs, missing or non-positive bounds, and
invalid kind-specific fields produce field-level errors and identify the first
invalid control. Storage, events, and mounted runtime state remain byte-for-byte
unchanged. Editor controls remain labelled, keyboard-operable, and usable at a
mobile floating-panel width.

### Property 44: Browser readiness is exact-revision evidence

The owned fresh-origin task-worktree flow is an unsealed diagnostic and cannot
establish runtime readiness. Runtime readiness requires the complete flow on
the exact integrated main SHA: source badge, Add/Edit/Remove, persistence after
reload, live hide/show Toggle, invalid-draft atomicity, reset,
extrusion/asset readiness, keyboard-native controls, mobile layout, and no
critical page or request failure. Source markers or environment-only rendering
cannot satisfy this property.
