---
schema_id: "agentic-rag-document-v1"
id: "knowgrph-geospatial-mode"
title: "Knowgrph Geospatial Mode"
status: "runtime-ready-dev"
updated_at: "2026-07-27"
source_repository: "huijoohwee/knowgrph"
source_path: "docs/documents/knowgrph-geospatial-mode-document.md"
authority: "canonical source document in knowgrph"
runtime_scope: "structured operator controls, protected Integration Gate, canonical local supervision, and exact-main browser proof pass at 6b381860cb2abd26cc2e37b84fd1bbc9cfa93896"
deployment_scope: "no production or Cloudflare authority"
commands:
  - "/geo"
  - "/geospatial"
hash_tags:
  - "#geospatial"
at_tokens:
  - "@geospatial"
---

# Knowgrph Geospatial Mode — Kiro Projection

This file is a concise projection for the Kiro specification. The canonical
runtime and operator documentation is:

`/Users/huijoohwee/Documents/GitHub/knowgrph/docs/documents/knowgrph-geospatial-mode-document.md`

Detailed design is split by responsibility:

- `design.md`;
- `design-runtime-components.md`;
- `design-correctness-properties.md`;
- `design-verification-adrs.md`;
- `requirements.md`;
- `tasks.md`.

This projection records `runtime-ready-dev` after the exact protected-main
browser gate passed at
`6b381860cb2abd26cc2e37b84fd1bbc9cfa93896`.

## Runtime status

The development candidate preserves the existing three view selections:

- 2D MapLibre;
- 2D SVG fallback;
- 3D MapLibre globe.

Enhanced rendering is additive:

- polygons use native MapLibre `fill-extrusion`;
- `knowgrph-geo-asset-mesh/v1` descriptors use one MapLibre custom layer;
- zero enhanced configuration allocates no enhanced layer or custom context.

The runtime adds no GIS engine, model loader, spatial database, routing engine,
desktop shell, or paid service dependency.

## Configuration

Enhanced declarations resolve in this order:

1. present localStorage key `kg:ui:geospatial:enhancedLayers`;
2. `VITE_GEOSPATIAL_DATASETS_JSON`;
3. empty catalog.

An invalid environment value fails closed with an `invalid-config` diagnostic.
A present local value, including `[]`, overrides the environment. Per-layer
visibility remains under
`kg:ui:geospatial:enhancedLayerVisibility`.

### Operator catalog

Floating Panel Geo exposes a structured **Enhanced layers** catalog with an
**Environment**, **Local**, or **Empty** source badge. Each extrusion and asset
row exposes its ID, kind, status, **Edit**, **Remove**, and a labelled
visible/hidden **Toggle**. The catalog publishes
`data-kg-geo-enhanced-config-source`; row actions use the accessible names
`Toggle enhanced layer <id>`, `Edit enhanced layer <id>`, and
`Remove enhanced layer <id>`.

**Add enhanced layer** and Edit expose URL, mandatory `timeoutMs`/`maxBytes`, and
kind-specific fields. Valid Add/Edit/Remove actions atomically persist the full
local catalog and update the same-tab runtime without reload. Toggle persists
only that layer's visibility override and updates the mounted layer within 500
milliseconds.

**Reset to environment defaults** removes the local catalog and visibility
keys, never writes `[]`, and immediately restores the environment or empty
source. Invalid drafts show field-level errors and cause no storage, event, or
runtime mutation. Controls remain keyboard-operable and usable at mobile panel
widths.

Every entry requires:

```json
{
  "fetchBounds": {
    "timeoutMs": 20000,
    "maxBytes": 26214400
  }
}
```

The effective enhanced-resource deadline is still
`min(timeoutMs, 10_000)`, so the example above has a ten-second readiness
ceiling.

## Loading and cache

- Missing bounds abort before a request.
- Content-Length and streamed bytes are checked against `maxBytes`.
- Oversized and timed-out partial payloads are discarded.
- Late timed-out work cannot write the cache.
- Cache hits issue no request and recheck the current caller byte limit.
- The current-tab LRU owns at most 32 MiB and 32 entries.
- Eviction or page reload can require retrieval again.
- Failures retain successful siblings.
- Network failures surface the literal `network-unavailable`.

## Extrusion

Building and road footprints share one feature normalizer. The named height
property accepts finite values in `[0,10_000]` meters. Invalid or missing values
use the configured neutral fallback without dropping the feature. Color,
opacity, base height, and fallback height are configuration-owned.

## Custom-layer assets

The runtime validates finite positions, triangle indices, and RGBA color before
allocating GL resources. Every draw computes:

```text
defaultProjectionData.mainMatrix
  * map.transform.getMatrixForModel([lng, lat], altitudeMeters)
  * z-up local scale and rotation
```

This per-frame path follows the current MapLibre globe/Mercator projection and
camera. The former manual flat-Mercator matrix is removed.

The custom layer restores host GL state, releases owned buffers/programs/vertex
arrays, and recreates them after context restoration.

## Invocation

The local runtime supports:

- `/geo on|off`;
- `/geo extrusion <id> show|hide`;
- `/geo asset <id> show|hide`;
- `@<existing-geo-node-id>`;
- `#<tag> show|hide`;
- MCP tool `knowgrph.geospatial.command`.

Collision rules are strict. Bare or unrelated `@` and `#` tokens fall through
to the existing chat/document invocation system. Unknown targets and actions
return bounded actionable errors and perform no write.

Chat and MCP converge on the same typed command runtime and
`gympgrphBridge`. MCP query envelopes are validated, consumed once, and removed
from browser history. A failed on-demand package import restores the prior mode
preference without changing the view mode.

## Optional authoring

The model harness is off by default. It validates input before any model call,
enforces 1-50 iterations, enforces a 1-300-second model timeout, validates every
completed-call cost log, and applies only a valid draft through an injected
callback.

If no adapter exists, the model times out, or transport fails, it returns a
typed `model-unavailable` error plus a deterministic disabled draft. The draft
has an empty URL and invisible render entry. It is never auto-applied.

There is no claimed `/geo.author` production caller or model-authoring button.
The manual structured Enhanced layers catalog is independent of this optional
model harness.

## Readiness

The owner command is:

```bash
npm run geospatial-mode:check
```

It performs shared and extracted-package builds, Canvas TypeScript validation,
all focused geospatial Node tests, MCP tests, filtered Canvas
geospatial-invocation tests, a production build, and the final readiness guard.

The source guard must verify:

- focused renderer, editor, persistence, Canvas, and MCP test owners;
- ordered executable evidence for all 44 correctness properties;
- prohibited dependency absence;
- zero compiled dataset/asset URLs;
- runtime owner files below 600 lines;
- required canonical-document markers;
- production gzip delta within 250 KiB.

Browser evidence began with an owned fresh-origin task-worktree diagnostic,
which remains intentionally unsealed. Runtime readiness required the same
source-badge transitions, Add/Edit/Remove, persistence after reload, live
Toggle, mutation-free field validation, Reset to environment defaults,
keyboard-native controls, mobile use, extrusion and asset readiness, and no
critical page or request failures on the exact integrated main SHA. Those Dev
gates passed at `6b381860cb2abd26cc2e37b84fd1bbc9cfa93896`, so status is
`runtime-ready-dev`. This proof does not claim production, Cloudflare,
public-route, or physical-device validation.
