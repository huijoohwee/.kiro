# Requirements Document: Knowgrph Agentic ECS

## Introduction

Knowgrph needs a small, native Entity Component System (ECS) for deterministic, data-oriented state updates over KGC Markdown documents. The ECS is an in-process Dev runtime owned by Knowgrph. It reuses the repository's KGC, cost-log, MCP, and Canvas owners; it does not create a second graph store, model router, renderer, tool server, deployment surface, or compatibility layer.

The design may use neutral data-oriented ECS principles as inspiration. It SHALL NOT copy bitECS code, prose, schemas, tests, fixtures, or examples and SHALL NOT depend on bitECS at runtime, build time, or test time.

The sibling Agentic Canvas OS repository owns only the source-backed invocation tokens. Knowgrph owns all executable behavior.

## Glossary

- **World**: Opaque process-local ECS state. A private MCP World is session-bounded; a directly created caller-held World follows its embedding host's lifetime.
- **Entity**: Monotonic integer identity allocated atomically with its initial components.
- **Component**: Named structure-of-arrays store whose fields use supported numeric typed arrays.
- **System**: Ordered function that reads or mutates a World during one tick.
- **Decision executor**: Optional injected reasoning boundary; it is not a model client or provider owned by ECS.
- **KGC document**: Repository-owned Markdown whose YAML frontmatter contains `@node` records.
- **EcsComponentSchema**: KGC node with `properties.ecsComponent = { name, fields }`.
- **EcsEntity**: KGC node with `properties.ecsEntity = { entityRef, components }`.
- **EcsDecision**: KGC node with `properties.ecsDecision = { decisionId, decisionType, entityRef, payload, producedAt }`.
- **Pending decision**: Decision produced by the session and not yet persisted successfully.
- **ECS session**: Opaque id mapped privately to one hydrated World, source path, and pending decisions.
- **Execution boundary**: Always `dev-only`; the canonical no-system/no-executor stdio construction has no Prod, Cloudflare, network, or deployment capability. An embedding host exclusively owns any external behavior and cost of injected Systems and its optional executor.

## Requirement 1: Native, Minimal Public Runtime

**User story:** As a maintainer, I want one small native ECS owner so that runtime state has a clear source and no external engine dependency.

### Acceptance criteria

1. THE `ecs` package SHALL publicly export exactly `createWorld`, `allocateEntity`, `registerComponent`, `query`, and `worldTick`.
2. THE World SHALL be opaque to callers; internal stores, journals, sentinels, and system registration details SHALL NOT become public compatibility APIs.
3. `createWorld` SHALL accept optional ordered systems, an optional decision executor, and an optional clock through construction options.
4. THE implementation SHALL contain no bitECS dependency, copied source, alias facade, or stale alternate ECS owner.
5. THE implementation SHALL remain split by responsibility and every authored source file SHALL remain below 600 lines.

**VCC:** Import the package, compare its export names to the exact five-name set, construct an opaque World, scan manifests/source for bitECS, and run hygiene checks.

## Requirement 2: Typed Component Storage and Atomic Entities

**User story:** As a system author, I want predictable numeric component storage so that queries and ticks are deterministic and cache-friendly.

### Acceptance criteria

1. `registerComponent` SHALL validate a non-empty unique name and a non-empty field specification before mutating the World.
2. Supported field types SHALL be `f32`, `f64`, `i8`, `i16`, `i32`, `u8`, `u16`, and `u32`; unsupported types SHALL fail atomically.
3. Each field SHALL use the matching JavaScript typed-array constructor and SHALL grow without changing existing values.
4. `allocateEntity` SHALL validate every referenced component and field value before reserving the next entity id.
5. Successful allocation SHALL attach all requested components atomically; failure SHALL leave entity count, next id, and every store unchanged.
6. Missing component membership SHALL resolve through one private unique sentinel and SHALL render as `[absent]`; numeric `0`, `NaN`, and all valid stored values SHALL remain distinguishable from absence.
7. `query` SHALL return the ascending intersection of entities that own every requested component and SHALL reject unknown components without mutation.

**VCC:** Property tests cover typed-array selection, write/read round trips, growth preservation, absence, atomic rejection, and ascending query intersections for at least 100 generated cases per property.

## Requirement 3: Deterministic KGC Hydration

**User story:** As an operator, I want a session rebuilt from KGC so that the Markdown document remains authoritative.

### Acceptance criteria

1. Hydration SHALL validate `EcsComponentSchema`, `EcsEntity`, and `EcsDecision` nodes, index persisted Decisions, and SHALL ignore unrelated KGC nodes.
2. Component schemas and entities SHALL be materialized in a stable sorted order independent of authored node order.
3. At least one valid component schema and one valid entity SHALL be required; duplicate schema names, entity references, or Decision ids, unknown component references, missing required fields, unsupported field types, invalid values, or invalid Decisions SHALL return a structured error before exposing a session.
4. Byte-identical valid KGC input SHALL hydrate to observably equal query results and component values.
5. Hydration SHALL create no KGC writes, model calls, network calls, source-file aliases, or fabricated default entities.
6. A failed session start SHALL retain no private session entry.
7. `EcsDecision.decisionType` SHALL be exactly `dialogue_outcome`, `quest_flag`, or `world_tick_result`.

**VCC:** Unit and property tests permute equivalent node order, compare hydrated observations, and assert malformed inputs produce typed failures with zero persistent or session mutation.

## Requirement 4: Ordered Transactional World Ticks

**User story:** As a system author, I want one bounded asynchronous tick so that committed state and failure behavior are unambiguous.

### Acceptance criteria

1. `worldTick` SHALL be asynchronous and SHALL execute systems sequentially in construction order.
2. Mutations from each successful system SHALL commit before the next system and SHALL be visible to it.
3. A failing system SHALL roll back only that system's journal, preserve all earlier committed systems, skip later systems and reasoning, and return a structured failure.
4. A System SHALL receive only the frozen restricted context `query`, `read`, `write`, `setComponent`, `emitDecision`, and `requestReasoning`; an invalid emitted Decision SHALL fail inside that System transaction and roll back its writes.
5. Public query, snapshot/projection, and concurrent tick reads SHALL fail with `ECS_TICK_IN_PROGRESS` while a tick is open; only the restricted System context may observe provisional state.
6. Equal initial World state, systems, input, clock behavior, and decision-executor result SHALL yield equal observable output.
7. A successful tick whose Systems emit no reasoning request SHALL emit exactly one canonical zero-cost record created by the shared cost-log owner; failed or concurrent ticks SHALL NOT fabricate that record.
8. Each executor request SHALL return exactly one valid non-`none` entry in plural `cost_logs`; canonical extra fields SHALL be dropped, and valid usage SHALL remain reported even when the paired Decision is invalid/deferred.
9. Reasoning SHALL receive an `AbortSignal` and SHALL be bounded to 30 seconds.
10. Timeout, unavailable reasoning, or invalid usage SHALL return a sanitized deferred outcome, preserve committed non-agentic work, and SHALL NOT create a fake decision, fake usage, or persistent write.
11. Produced decisions SHALL validate against the exact `EcsDecision` contract before becoming pending session state.
12. Once a structured tick commits, any pending-retention failure caused by a conflicting or invalid/noncanonical Decision SHALL return `tickCommitted: true`, the incremented tick count, canonical cost/deferred evidence, and unchanged prior pending state so clients do not blindly replay committed work. A structured System failure MAY preserve prior System commits and increment the tick count without this post-tick marker.

**VCC:** Focused unit and property tests prove order, visibility, per-system rollback, determinism, timeout/unavailable deferral, canonical zero cost, and non-duplicated real cost logs.

## Requirement 5: Decision-Only Atomic Persistence

**User story:** As an operator, I want only agent decisions written back so that runtime component storage never replaces authored KGC state.

### Acceptance criteria

1. Persistence SHALL accept a session id, not caller-supplied decisions, and SHALL persist only that session's pending validated decisions.
2. Each persisted node SHALL use `EcsDecision` and contain `decisionId`, `decisionType`, `entityRef`, `payload`, and `producedAt` under `properties.ecsDecision`.
3. Persistence SHALL be idempotent by `decisionId`; retry SHALL NOT duplicate an existing decision.
4. Existing frontmatter records, Markdown body, ordering outside the inserted decision records, and unrelated authored bytes SHALL be preserved.
5. Persistence SHALL write a unique sibling temporary file exclusively, complete an atomic rename, and remove temporary residue where possible without replacing the primary failure.
6. Any validation, write, or rename failure SHALL leave the source unchanged and keep pending decisions available for retry.
7. Successful persistence, including a zero-pending terminal result, SHALL dispose the session.
8. Raw component stores, entity snapshots, deferred outcomes, and cost logs SHALL never be persisted as KGC decisions.
9. Concurrent persistence calls in one process for the same canonical KGC path SHALL serialize so both non-conflicting batches survive; cross-process mutation SHALL remain governed by the repository collaboration lease rather than an unproven file-lock claim.
10. A session SHALL bind the start-time canonical source path and device/inode identity. Persistence SHALL revalidate root containment, path, and identity inside the serialized turn, read through a no-follow file handle, and repeat validation before its temporary write and rename. Only the session identity or a bounded replacement lineage recorded by prior same-process queued ECS renames SHALL be trusted; unrecognized swaps SHALL fail closed.
11. If source persistence succeeds but session disposal fails, the session SHALL retain pending Decisions plus the replacement file identity for an idempotent terminal retry.

**VCC:** Integration tests compare pre/post bytes, read back the written YAML, retry the same decision ids, inject write/rename failures, and verify session retention or disposal at the correct terminal boundary.

## Requirement 6: Thin Existing-Owner Canvas Projection

**User story:** As a Canvas user, I want ECS state shown through the existing KGC path so that there is no renderer or graph-store fork.

### Acceptance criteria

1. Projection SHALL derive a KGC Markdown text representation from a read-only internal World snapshot.
2. Projection SHALL call the extracted existing-owner seam `applyChatKgcDocumentTextToCanvas`.
3. Projection SHALL NOT create a temporary Source File, directly mutate Canvas stores, or introduce an ECS-specific renderer.
4. Missing component fields SHALL display `[absent]` and SHALL NOT mutate World state.
5. Apply-path failure SHALL return a structured projection error without changing World state.
6. Snapshot/apply error metadata SHALL be sanitized and SHALL NOT reflect source bytes, credentials, or thrown messages.
7. Projection SHALL remain an in-process host seam over a caller-held World and SHALL NOT become a fourth MCP tool or gain access to the private MCP session registry.

**VCC:** Focused Canvas tests prove the existing text seam is called, no temporary source document is created, absent values degrade visibly, and before/after World observations are equal.

## Requirement 7: Exact MCP Tools and Private Session Lifecycle

**User story:** As an external agent, I want three bounded MCP operations over the existing stdio server so that ECS is discoverable without a new transport.

### Acceptance criteria

1. The existing official MCP SDK server SHALL list exactly these new tools: `knowgrph.ecs.session_start`, `knowgrph.ecs.world_tick`, and `knowgrph.ecs.decision_persist`.
2. Tool arguments SHALL be:
   - session start: `{ kgcPath, scope?, binding? }`;
   - world tick: `{ sessionId, input?, scope?, binding? }`;
   - decision persist: `{ sessionId, scope?, binding? }`.
3. Invocation grammar SHALL resolve exactly:
   - `/ecs.session-start #agentic-ecs @source.frontmatter`;
   - `/ecs.world-tick #agentic-ecs @ecs-session`;
   - `/ecs.decision-persist #agentic-ecs @ecs-session`.
4. Unknown scope, binding, extra arguments, unsafe path, unknown session, expired session, or malformed payload SHALL return a structured MCP error without uncaught exceptions or mutation.
5. `kgcPath` SHALL resolve through realpath to a `.md` file inside the configured repository root; root-escaping traversal, symlink escape, and non-Markdown targets SHALL fail closed, while normalized paths that remain inside the root MAY resolve.
6. Session start SHALL bind the validated source identity and read it through a no-follow file handle; persistence SHALL repeat containment/path/identity validation inside its serialized operation.
7. Session ids SHALL be unguessable opaque UUIDs and the private store SHALL enforce a TTL, maximum count, and lazy expiry sweep.
8. Successful or zero-pending persistence SHALL dispose the session; persistence failure SHALL retain it. An inactive expired session SHALL be removed only after successful disposal without source mutation; expiry disposal failure SHALL retain the session, extend its TTL, and surface a retryable disposal error. A post-commit disposal failure SHALL retain state for idempotent close retry.
9. Every tool result SHALL state `execution_boundary: "dev-only"` and SHALL expose no deploy, network, credential, Prod, or Cloudflare capability.
10. Tool input schemas SHALL be closed. The shared output envelope SHALL require `ok` and `execution_boundary` while admitting documented tool-specific result fields.
11. The implementation SHALL add no fourth ECS tool, alternate transport, or public session registry.
12. The canonical stdio server SHALL invent no default system, decision, or model route. Its default zero-system tick SHALL remain valid and model-free; reviewed embedding code MAY inject systems and an executor only through runtime construction, never through MCP arguments.
13. MCP output SHALL allowlist public error/deferred metadata and canonicalize returned Decisions and costs. Allowlists SHALL constrain field shape but SHALL NOT claim content redaction for retained identifiers or canonical values. Embedding hosts SHALL NOT place secrets in Decision identifiers/fields/payload, deferred request identifiers, or canonical Cost_Log fields. Sanitized thrown-error/projection metadata and World internals SHALL NOT reflect source bytes, executor prompts/arbitrary codes, or function names.

**VCC:** An official MCP SDK stdio client performs `initialize`, `tools/list`, and the three calls; tests prove exact discovery, schemas, safe-path enforcement, bounded lifecycle, terminal disposal, retry retention, and absence of a new transport.

## Requirement 8: Source Contracts and Validation

**User story:** As a maintainer, I want source, runtime, and proof to agree so that runtime-ready claims are reviewable.

### Acceptance criteria

1. The canonical KGC schema owner SHALL preserve JSON-safe node `properties`; the ECS node-contract owner SHALL strictly validate all three ECS node shapes without weakening existing KGC behavior.
2. The shared collaboration contract SHALL select ECS unit and property tests when `ecs/**` changes.
3. Focused validation SHALL include ECS unit/PBT, KGC schema, MCP stdio/session, Canvas projection, Canvas typecheck, collaboration selection, hygiene, and diff checks.
4. The PRD/TAD and this Kiro specification SHALL describe the implemented API, persistence, session, cost, and Dev-only boundaries without stale FastMCP, deploy-lane, singular-cost-log, or compatibility claims.
5. Runtime proof SHALL distinguish source tests from browser proof, protected integration, and deployment; no deployment is part of this increment.

**VCC:** All focused commands exit zero, changed authored files stay under 600 lines, the protected Integration Gate passes the exact commit, and final handoff explicitly states that Prod and Cloudflare were untouched.

## Out of Scope

- Physics, collision, rendering engines, multiplayer, durable World snapshots, or server-hosted ECS.
- An ECS-owned model client, provider credentials, prompt registry, agent loop, graph orchestrator, or cost estimator.
- New MCP transports, HTTP routes, browser tool surfaces, or Cloudflare bindings.
- Production promotion, mirror mutation, Cloudflare deployment, or a mechanism to selectively open deployment lanes.
- Backward-compatibility aliases for the rejected draft API.
