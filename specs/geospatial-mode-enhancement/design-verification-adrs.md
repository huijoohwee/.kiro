# Geospatial Verification, Decisions, and Traceability

## Source-readiness command

The development source/build owner is:

```bash
npm run geospatial-mode:check
```

It runs these gates in order:

1. build `grph-shared`;
2. build `gympgrph`;
3. run the Canvas TypeScript check;
4. execute every `scripts/__tests__/geospatial-*.test.ts` suite, including
   structured editor/persistence tests, and the geo-authoring fallback suite;
5. execute MCP geospatial contract/runtime tests;
6. execute filtered Canvas geospatial-invocation registry tests;
7. create a production Canvas build;
8. run the dependency, URL, source-size, document, 44-property manifest, and
   gzip source-readiness guard.

The ordered manifest must contain Properties 1-44 and at least one existing,
stable evidence marker for every property. The focused suites themselves are
executed; a manifest marker alone is not reported as a passing runtime test.
This command cannot by itself mark the feature runtime-ready because Property
44 requires separately recorded browser evidence at two exact revisions.

## Test technique

Generative properties use `fast-check` with 120 runs for height totality,
road/building equivalence, valid projection coordinates, and mutation-free
target rejection. Deterministic tests cover projection composition, GL state
and context restoration, LRU eviction, per-hit bounds, hard deadlines,
progress, source precedence, disabled authoring fallback, and fit precedence.
Structured UI tests cover source-badge projection, draft validation, atomic
Add/Edit/Remove, visibility isolation, reset semantics, and same-tab event
publication.

Canvas registry tests cover invocation collision and bridge integration. MCP
tests cover the published tool schema and command envelope. Clean-profile
browser proof covers rendered host behavior and same-origin fixtures; it is
recorded separately from source/build proof.

## Runtime proof boundary

Candidate browser readiness requires:

- owned task-worktree provenance: branch, fence SHA, and worktree;
- all source/build/readiness gates green;
- one clean-profile browser flow beginning with the environment catalog;
- visible Environment/Local/Empty source transitions;
- structured Add, Edit, Remove, and invalid-draft interactions;
- persistence proven through reload;
- per-layer hide/show proven without reload;
- Reset to environment defaults proven without writing an empty local catalog;
- native extrusion and custom-layer readiness markers;
- a nonzero MapLibre canvas;
- an explicit 3D/globe view check;
- keyboard and mobile viewport interaction checks;
- no critical page or request failure.

After protected integration, the same assertions must pass against the exact
integrated main SHA. The unsealed task-worktree diagnostic cannot be relabelled
as exact-main evidence. Only then may the spec status become
`runtime-ready-dev`. This is
deterministic desktop browser proof, not physical-device, production,
Cloudflare, or public-route proof. Those require separate authority.

Recorded Dev proof passed at protected main
`6b381860cb2abd26cc2e37b84fd1bbc9cfa93896`: the Integration Gate succeeded,
the repository supervisor proved Apex, storage, and proxy health, and the
repeated browser flow proved mounted-layer readiness, configure/toggle/reload,
invalid-submit stability, 390 × 844 access, and reset persistence.

## Error classification

| Failure | Runtime result |
| --- | --- |
| Missing `timeoutMs` or `maxBytes` | abort before fetch; identify key |
| Payload exceeds current `maxBytes` | cancel/discard; retain siblings |
| Effective deadline reached | `timeout` with effective milliseconds |
| URL/fetch unavailable | literal `network-unavailable` |
| Invalid GeoJSON or mesh | `parse-failed`; skip only target |
| Invalid asset coordinate/matrix | diagnostic or frame skip; continue |
| Unknown invocation action/target | actionable typed rejection; no write |
| Invalid editor field or duplicate ID | field-level error; no storage/event/runtime write |
| Missing graph geo bounds | `no-geo-bounds`; no camera write |
| Unmatched tag | `no-tag-match`; no visibility write |
| Runtime dynamic import failure | restore prior mode preference |
| Invalid authoring input/output | structured error; no model/apply as applicable |
| Model timeout/transport/absence | typed error plus disabled fallback |
| Cost/iteration circuit breaker | structured error; no partial apply |

Messages exposed by enhanced-layer loading are bounded to 140 characters.
Detailed provider errors are not passed through authoring fallback results.

## Bundle and source budgets

- No additional GIS, GPU, model-loader, database, routing, or desktop-shell
  runtime dependency.
- Gzipped production JavaScript may grow by at most 250 KiB over the
  deterministic readiness baseline.
- Runtime, UI, and readiness owner files, including `GeospatialPanelHost.tsx`,
  must be at most 600 lines.
- Compiled runtime source must contain no dataset or asset URL.
- Enhanced cache: at most 32 MiB and 32 entries.
- Enhanced resource readiness: at most 10 seconds.
- Authoring loop: at most 50 completed model iterations.
- Authoring model timeout: 1-300 seconds.

The readiness command reports the current gzip bytes and delta. A prior number
in prose is historical unless it matches the latest successful command output.

## ADR-1: Use native `fill-extrusion`

Decision: use MapLibre's existing style layer for polygon height.

Why: it reuses the current map, projection, interaction, ordering, and
dependency graph. A second GPU overlay would duplicate camera and lifecycle
ownership.

Consequence: input must be polygonal GeoJSON and height remains a style
property. This is accepted.

## ADR-2: Use MapLibre's custom-layer context

Decision: draw source-authored mesh descriptors through one custom layer that
shares MapLibre's WebGL context.

Why: arbitrary assets need a draw hook, but not a second engine.

Consequence: Knowgrph owns shader, buffer, GL-state restoration, context-loss
handling, and cleanup. Focused lifecycle tests protect this boundary.

## ADR-3: Use the active MapLibre projection matrix

Decision: compose `defaultProjectionData.mainMatrix`,
`getMatrixForModel`, and the local z-up transform on every frame.

Why: a manual Mercator matrix cannot follow globe rendering or projection
transition state.

Consequence: MapLibre 5.24's transform API is the projection SSOT. Missing or
non-finite frame data skips the affected draw without a stale fallback.

References:

- [MapLibre 3D model on globe](https://maplibre.org/maplibre-gl-js/docs/examples/add-a-3d-model-to-globe-using-threejs/)
- [MapLibre simple custom layer on globe](https://maplibre.org/maplibre-gl-js/docs/examples/add-a-simple-custom-layer-on-a-globe/)
- [CustomRenderMethodInput API](https://maplibre.org/maplibre-gl-js/docs/API/type-aliases/CustomRenderMethodInput/)

## ADR-4: Use a source-authored JSON mesh

Decision: accept the minimal `knowgrph-geo-asset-mesh/v1` descriptor.

Why: a model-format loader adds weight, parser surface, and external asset
semantics that the Must scope does not need.

Consequence: operators preprocess other formats outside the runtime.

## ADR-5: Bound both individual and aggregate memory

Decision: require request bounds and add a byte-accounted, entry-bounded LRU.

Why: per-request limits do not prevent aggregate tab memory growth. A plain
unbounded `Map` is not runtime-ready.

Consequence: cache eviction or page reload may require network retrieval again.
Offline support is current-tab best effort, not persistent offline storage.

## ADR-6: Keep config precedence explicit

Decision: a present local key overrides the environment catalog; the
environment initializes only a clean profile.

Why: operator edits must not be overwritten on reload, while demos and
deployments need URL-neutral configuration and normal users need a structured
enhanced-layer form.

Consequence: an explicit local `[]` disables the environment catalog until the
key is removed. Add/Edit/Remove write the local catalog; per-layer Toggle uses a
separate visibility key; **Reset to environment defaults** removes both keys,
never writes `[]`, and re-resolves environment then empty precedence.

## ADR-7: Converge invocation on one bridge

Decision: chat and MCP parse to the same command model and mutate only through
`gympgrphBridge`.

Why: direct map references, duplicate storage writes, and surface-specific
behavior would violate interaction and synchronization ownership.

Consequence: `@` resolution requires current GraphData; enhanced visibility
commands load the current normalized config before mutation.

## ADR-8: Return but never auto-apply fallback drafts

Decision: unavailable models produce a deterministic disabled draft and typed
error.

Why: a useful editable starting point improves local resilience, while
automatic writes would hide model failure and risk partial or unintended
configuration.

Consequence: a caller must present and explicitly apply the draft later.

## Requirements traceability

| Requirement | Primary properties | Main executable surfaces |
| --- | --- | --- |
| R1 preserve existing mode | 1-2, 36-38 | Canvas filtered tests, bridge rollback, browser |
| R2 native extrusion | 3-6 | shared generative/unit suite |
| R3 custom assets | 7-11 | projection/GL lifecycle suite, browser |
| R4 lightweight/no-copy | 12 | dependency and URL guard |
| R5 browser/mobile/local/offline | 13-16, 37 | cache/loader tests, browser |
| R6 bounded config ingestion | 17-21 | config, loader, proxy tests |
| R7 invocation surfaces | 22-26 | Canvas invocation and MCP suites |
| R8 optional authoring | 27-33 | authoring and fallback suites |
| R9 visibility/reliability | 34-38 | loader, host tests, fit test, browser |
| R10 operator catalog | 39-44 | editor/persistence tests, panel integration, candidate and exact-main browser proof |

## Inspiration-only references

The following projects supplied neutral design questions only:

- GeoLibre: local-first geospatial workflows;
- deck.gl: declarative layer composition;
- Tauri: explicit capability boundaries;
- DuckDB-Wasm: bounded in-browser processing.

No implementation, prose, schemas, examples, or dependency configuration is
copied from these projects.
