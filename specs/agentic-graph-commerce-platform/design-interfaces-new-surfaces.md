---
title: "AgenticGraph Commerce Platform — Design Annex A2: Typed Contracts, Requirements 7–13"
doc_type: "Spec Design Annex"
schema: "kiro-spec-design-annex/v1"
version: "0.1.0"
date: "2026-08-28"
lang: "en-US"
frontmatter_contract: "required"
feature_name: "agentic-graph-commerce-platform"
parent_document: "design.md"
implementation_baseline: "agentic-commerce-os @ main 2e39e5c43f2f49866f4dd6849994f7bfdf677ec1"
---

# Annex A2 — Typed Contracts, Requirements 7–13 (new surfaces)

Companion to Annex A1 (`design-interfaces-baseline-edits.md`), which covers Requirements 1–6. Split along the owner boundary between edits to built files and genuinely new surfaces, per Requirement 13.7's 600-line ceiling.

Element numbers match the Components and Interfaces table in `design.md`. Every block states the disposition and the target file. Conventions carried from the baseline without change: `Readonly<{...}>` result shapes, `Object.freeze` on returned records, `{ ok: false, code }` rejections, canonical JSON plus SHA-256 for every digest (`src/shared/digest.ts`), zero `any`, zero index signatures on payload types.

## Requirement 7 — WebMCP (elements 24–25)

```ts
// 24 — src/edge/client/storefront-actions.ts (NEW). ~140 authored lines. The single action set.
export type StorefrontActions = Readonly<{
  searchCatalog(input: Readonly<{ query: string; limit: number }>): Promise<CatalogPage>
  selectOffer(input: Readonly<{ listingId: string; offerId: string }>): Promise<SelectedOffer>
  initiateCheckout(input: Readonly<{ offerId: string; amountMinor: number; currency: string }>): Promise<CheckoutPrepared>
  // confirmCheckout is deliberately ABSENT from the action set: human confirmation is obtained
  // through the visual surface only, and no tool-supplied value substitutes for it (7.4).
}>
export function createStorefrontActions(deps: StorefrontDeps): StorefrontActions
// Both the visual controls and the WebMCP tools call these exact functions; zero backend endpoint
// exists solely for the tool surface (7.2).
```

```ts
// 25 — src/edge/client/webmcp-tools.ts (NEW). ~170 authored lines. Browser module.
export const MAXIMUM_REGISTERED_TOOLS = 16
export type RegisteredToolSet = Readonly<{
  tools: readonly Readonly<{ name: string; inputSchema: unknown; outputSchema: unknown }>[]
  toolCount: number
  digest: string     // SHA-256 over names + input schemas + output schemas + count (7.5)
}>
export type RegistrationOutcome =
  | Readonly<{ ok: true; registered: RegisteredToolSet; registeredInMs: number }>
  | Readonly<{ ok: false; code: 'model_context_api_absent'; event: 'webmcp_surface_unavailable' }>  // 7.3
export type ToolRefusal = Readonly<{
  ok: false; code: 'webmcp_registration_drift'; invokedTool: string
  recordedDigest: string; observedDigest: string
}>  // carries zero credential, card identifier, card token, or auth token (7.7)
export function registerStorefrontTools(actions: StorefrontActions): Promise<RegistrationOutcome>
export function verifyBeforeExecute(set: RegisteredToolSet): ToolRefusal | null   // 7.6
```

## Requirement 8 — Routing and public catalog (elements 26–30)

```ts
// 26 — src/domain/selection-policy.ts (NEW). ~110 authored lines. Pure, model-free.
export type DeclaredAttributes = Readonly<{ priceMinor: number; qualityScore: number; latencyMs: number }>
export type SelectionPolicy = Readonly<{
  schema: 'agentic-graph-selection-policy/v1'
  weights: Readonly<{ price: number; quality: number; latency: number }>   // externalized (8.1)
  normalization: 'min-max'
}>
export type Selection = Readonly<{
  selectedAgentId: string; score: number
  consideredAgentIds: readonly string[]                                   // 8.2
  decidingAttributes: Readonly<Record<string, DeclaredAttributes>>        // 8.2
}>
export function selectAgent(
  eligible: readonly Readonly<{ agentId: string; attributes: DeclaredAttributes }>[],
  policy: SelectionPolicy,
): Selection | null      // null → no eligible agent (8.6); score ties resolve by ascending agentId
```

```ts
// 27 — src/domain/exclusive-category-router.ts (MODIFIES). One branch replaced.
// The `candidates.length > 1 → no-dispatch: 'ambiguous-category'` branch becomes a selectAgent call.
// NoDispatchReason drops 'ambiguous-category' and keeps 'invalid-intent', 'unmatched-category',
// 'registry-conflict'. DispatchDecision gains `consideredAgentIds` and `decidingAttributes`.
// Every existing guard — credential screening, JSON compatibility, category normalization,
// registry-conflict detection — is untouched.
```

```ts
// 28 — src/core/agent-registry.ts (MODIFIES). Attributes plus one index drop.
export type CommerceAgentProjection = Readonly<{
  category: string; discoveryTool: string
  declaredAttributes: DeclaredAttributes      // NEW, part of registrationContentHash
  fallbackAgentId: string | null              // NEW, at most one declared fallback (8.4)
}>
// DROP INDEX one_active_admission_per_category — Requirement 8.1 presumes multiple active agents
// per category, which that unique index makes unreachable. health() changes from
// "exactly one verified active admission per required category" to
// "at least one verified active admission per required category, zero stale invocation pins".
// Obliges wrangler.core.jsonc migration tag v2 and a docs/do-storage-compatibility.json update.
```

```ts
// 29 — src/core/intent-route.ts (MODIFIES). Fallback dispatch.
export type DispatchAttempt = Readonly<{
  agentId: string; role: 'selected' | 'fallback'
  outcome: 'completed' | 'timeout' | 'failed'; startedAt: string; completedAt: string | null
}>
// dispatch(): AbortSignal.timeout(30_000) on the selected agent; on timeout, at most one dispatch to
// the declared fallback with reason 'dispatch_timeout' recorded; on fallback timeout or absence,
// { ok: false, code: 'dispatch_exhausted' } and zero further dispatch (8.4, 8.5).
// The DO singleton already guarantees exactly one dispatch per intent identity (8.3).
```

```ts
// 30 — src/core/public-catalog.ts (NEW). ~90 authored lines.
export type PublicAgentEntry = Readonly<{
  agentId: string; declaredCategory: string
  declaredCapabilities: readonly string[]
  trustStatus: 'declared-and-present'        // the only legal value until ADR-2 Phase 2
}>
export const PUBLIC_FIELD_ALLOWLIST: readonly string[]
export function projectPublicCatalog(snapshot: AgentRegistrySnapshot): Readonly<{
  ok: true; revision: number; digest: string; agents: readonly PublicAgentEntry[]
}>
// Constructed from the allowlist rather than by deleting fields, so no operator-only field,
// credential, or internal binding identity can leak by omission (8.8 / CP-16).
// Served unauthenticated at GET /v1/public/agents — read-only, no mutation, no credential.
```

## Requirement 9 — Local-first and claims (elements 31–33)

```ts
// 31 — src/edge/client/local-store.ts (NEW). ~180 authored lines. IndexedDB, no framework.
export const MAXIMUM_PENDING_CHANGES = 500
export type PendingChange = Readonly<{ sequence: number; scope: string; payload: unknown; recordedAtMs: number }>
export type RecordOutcome =
  | Readonly<{ ok: true; sequence: number }>
  | Readonly<{ ok: false; code: 'local_change_capacity_reached'; retained: number }>   // 9.2, never drops
export type ReplayOutcome = Readonly<{
  submittedSequences: readonly number[]   // strictly ascending, equals record order (9.3)
  acknowledgedSequences: readonly number[]
  retainedSequences: readonly number[]    // retained until acknowledged
}>
export function recordChange(change: Omit<PendingChange, 'sequence'>): Promise<RecordOutcome>
export function replayOnReconnect(submit: SubmitFn): Promise<ReplayOutcome>
export function renderableSnapshot(): Promise<LastSyncSnapshot>   // 9.1, zero network request
export function settlementBlockedOffline(): Readonly<{ code: 'connectivity_absent' }>  // 9.5
```

```ts
// 32 — src/core/sync-merge.ts (NEW). ~120 authored lines. Pure.
export type ChangeOrigin = Readonly<{ deviceId: string; sequence: number; recordedAtMs: number }>
export type FieldChange = Readonly<{ scope: string; field: string; value: unknown; origin: ChangeOrigin }>
export function mergeSequences(
  base: MergedState, left: readonly FieldChange[], right: readonly FieldChange[],
): MergedState
// Per-field last-writer-wins over a total order on (recordedAtMs, deviceId, sequence) plus an
// append-only event log for non-field changes. Order-independent by construction: the winner of
// each field is a max over a total order, and log entries are a set union ordered by the same key.
// Therefore mergeSequences(b, l, r) === mergeSequences(b, r, l) and zero change is discarded (CP-17).
// Recorded departure from ADR-3 (Yjs) — see design.md Design Decision 10.
```

```ts
// 33 — src/core/authoring-claim.ts (NEW). SQLite DO. ~170 authored lines.
export type Claim = Readonly<{
  claimId: string; actorId: string; deviceId: string; sessionId: string
  worktree: string; branch: string; semanticScope: string
  declaredWriteSet: readonly string[]
  leaseEpoch: number; leaseExpiresAtMs: number; fenceRevision: string
}>
export type ClaimAdmission =
  | Readonly<{ ok: true; claimId: string; leaseEpoch: number; fenceRevision: string }>
  | Readonly<{ ok: false
      code: 'scope_held' | 'lease_expired' | 'write_set_overlap' | 'fence_stale'
      holdingClaimId: string | null; holdingLeaseEpoch: number | null; holdingFenceRevision: string | null }>
export class AuthoringClaim extends DurableObject<CoreEnv> {
  acquire(claim: Claim): Promise<ClaimAdmission>
  admitMutation(scope: string, claimId: string, fenceRevision: string): Promise<ClaimAdmission>
  release(claimId: string, leaseEpoch: number): Promise<Readonly<{ ok: boolean }>>
}
// Every operator mutation route calls admitMutation before touching state; a refusal writes zero
// bytes (9.7) and the holding identity is returned, never inferred from a local projection.
```

## Requirement 10 — Held-offer change detection (elements 34–35)

```ts
// 34 — src/core/offer-watch.ts (NEW). ~160 authored lines.
export const OBSERVATION_INTERVAL_MS = 60_000
export const MAXIMUM_OBSERVATION_RETRIES = 3
export type ObservedAttributes = Readonly<{ priceMinor: number; available: boolean; agentActive: boolean }>
export type ChangeEvent = Readonly<{
  eventType: 'offer_changed'
  attribute: 'priceMinor' | 'available' | 'agentActive'
  recordedValue: unknown; observedValue: unknown; observedAt: string
}>
export type ObservationOutcome =
  | Readonly<{ kind: 'unchanged' }>                                        // appends nothing (10.7)
  | Readonly<{ kind: 'changed'; events: readonly ChangeEvent[] }>          // one per changed attribute
  | Readonly<{ kind: 'agent-inactive'; agentId: string }>                  // 10.4
  | Readonly<{ kind: 'failed'; attempt: number }>                          // 10.6
  | Readonly<{ kind: 'suspended'; attempts: 3 }>                           // 10.6
export function diffObservation(recorded: ObservedAttributes, observed: ObservedAttributes): readonly ChangeEvent[]
export function observeHeldOffer(env: CoreEnv, held: HeldOffer): Promise<ObservationOutcome>
// One provider read per observation; zero model call, zero additional paid call (10.8)
```

```ts
// 35 — src/core/checkout-session.ts (MODIFIES). Alarm plus settlement gate.
// alarm(): while state is 'confirmation_required' or 'confirming' and an offer is held, run
// observeHeldOffer and append its events to the existing checkout_event table, then reschedule at
// <= 60 s; stop within 60 s of settlement or closure recording stopAt and stopReason (10.1, 10.5).
// confirm(): before any settlement call, refuse when an unresolved 'offer_changed',
// 'offer_observation_suspended', or 'offer_agent_inactive' event exists for the held offer, and
// require a new human confirmation naming that event's sequence (10.3, 10.4 / CP-21).
export type ConfirmInputV2 = CheckoutConfirmInput & Readonly<{ acknowledgedEventSequence?: number }>
```

## Requirement 11 — Isolated execution (element 36)

```ts
// 36 — src/sandbox/executor.ts + wrangler.sandbox.jsonc (NEW). Separate Worker; dev-only.
export type SandboxPurpose = 'theme-build' | 'registration-dry-run' | 'unshipped-surface-build'
export type SandboxLimits = Readonly<{ wallClockSeconds: number; memoryMegabytes: number }>
//   wallClockSeconds <= 300, memoryMegabytes <= 512 (11.4)
export type AttemptedCall = Readonly<{ toolId: string; allowlisted: boolean; outcome: 'executed' | 'refused' }>
export type InstanceRecord = Readonly<{
  instanceId: string; purpose: SandboxPurpose
  startedAt: string; endedAt: string
  outcome: 'completed' | 'refused' | 'limit-exceeded' | 'failed'
  exceededLimit: 'wall-clock' | 'memory' | null; configuredValue: number | null
  attemptedCalls: readonly AttemptedCall[]
}>
export type SandboxResult =
  | Readonly<{ ok: true; record: InstanceRecord }>
  | Readonly<{ ok: false; code: 'sandbox_call_not_allowlisted'; toolId: string }>
  | Readonly<{ ok: false; code: 'sandbox_blocked'; reason: string; rung: 'dev-proven' }>
export function runIsolated(request: SandboxRequest): Promise<SandboxResult>
export function previewIdentity(engagementId: string, lifetimeHours: number): Promise<PreviewIdentity>
//   lifetimeHours <= 24; a later request to a revoked identity returns { code: 'preview_revoked' } (11.7)
// Binding surface is declared empty of payment credentials, settlement bindings, and operator
// authority in wrangler.sandbox.jsonc — capability absence is configuration, not discipline (11.6).
```

## Requirement 12 — Merge automation (element 37)

```ts
// 37 — scripts/merge-agent/*.ts (NEW). Node only. Zero deployment authority.
export type MergeBounds = Readonly<{
  tokenCeiling: number; iterationCeiling: number   // <= 10 mutating actions per run (12.5)
  wallClockMinutes: number                         // <= 30
  circuitBreaker: string                           // observable halting signal
}>
export type MergeReason = 'requires-operator-decision' | 'out-of-write-set' | 'scope-gap'
  | 'repair_approach_exhausted' | 'bound-reached' | 'circuit-breaker'
export type MergeAction = Readonly<{
  sequence: number; actingIdentity: string; lane: string
  action: 'comment-change' | 'check-repair' | 'conflict-resolution' | 'escalation'
  boundConsumed: Readonly<{ iterations: number; wallClockMinutes: number; tokens: number }>
  checkOutcome: 'pass' | 'fail' | 'not-run'
}>
export type MergeRunResult = Readonly<{
  lane: string; claimId: string; leaseEpoch: number; fenceRevision: string
  actions: readonly MergeAction[]        // append-only, ordered (12.9)
  changedPaths: readonly string[]        // subset of the declared write set (12.3 / CP-24)
  endedBecause: MergeReason | 'completed'
  issuedCommands: readonly string[]      // asserted to exclude push --force, reset --hard,
                                         // history rewrite, and every deploy command (12.7)
}>
export function runMergeAgent(bounds: MergeBounds, lane: LaneBinding): Promise<MergeRunResult>
// Calls AuthoringClaim.admitMutation before every mutation and stops the run when the lease or
// fence is no longer current (12.8). Reaches the canonical frontier only through Release_Controller.
```

## Requirement 13 — Evidence and limits (elements 38–40)

```ts
// 38 — scripts/evidence-reference.ts (NEW). ~140 authored lines.
export type CheckResult = Readonly<{
  namedCheck: string          // exact invocation form (13.1)
  ran: boolean; passed: boolean
  recordedResult: string      // the surfaced output, never "a result exists" (13.6)
  readableSurface: string     // path or URL bearing that output
}>
export type EvidenceReference = Readonly<{
  namedCheck: string; recordedResult: string; readableSurface: string; surface: 'authoring'
}>
export type VerdictRecord = Readonly<{
  taskId: string
  performingMechanism: string; verdictIssuingMechanism: string    // must differ (13.3)
  derivedFromSurfacedOutput: boolean
  finding: 'self_graded_verdict' | null                          // 13.4
}>
export function emitReferences(results: readonly CheckResult[]): readonly EvidenceReference[]
//   emitted set === { r | r.ran && r.passed } exactly (13.6)
export function deriveRung(
  references: readonly EvidenceReference[], openFindings: readonly string[],
): Readonly<{ localRung: string; deliveredRung: string; blocked: boolean }>
//   pure function of its inputs; blocked === openFindings.length > 0 (13.11)
```

```ts
// 39 — scripts/validate-authored-limits.ts (NEW). ~120 authored lines.
export const AUTHORED_LINE_CEILING = 600
export type LimitFinding = Readonly<{
  path: string; line: number | null
  finding: 'line-ceiling-exceeded' | 'absolute-developer-path' | 'credential-value' | 'account-identifier'
  detail: string     // names the finding without echoing a secret value
}>
export function validateAuthoredLimits(rootDirectory: string): Readonly<{
  ok: boolean; scannedFileCount: number; findings: readonly LimitFinding[]
}>
// Scans the authored file set only: excludes node_modules/**, src/generated/**, package-lock.json.
// Detail strings name the key or pattern, never the matched secret value.
```

```jsonc
// 40 — config/task-bounds.schema.json (NEW). Enforced against tasks.md before dispatch (13.1, 13.2).
{
  "required": ["taskId", "namedCheck", "tokenCeiling", "iterationCeiling",
               "wallClockMinutes", "contextCeiling", "circuitBreaker"],
  "properties": {
    "namedCheck":       { "type": "string", "minLength": 1 },
    "tokenCeiling":     { "type": "integer", "minimum": 1 },
    "iterationCeiling": { "type": "integer", "minimum": 1 },
    "wallClockMinutes": { "type": "integer", "minimum": 1 },
    "contextCeiling":   { "type": "integer", "minimum": 1 },
    "circuitBreaker":   { "type": "string", "minLength": 1 }
  }
}
```

## Shared Type Reused Across Elements

```ts
type Rejected = Readonly<{ ok: false; code: string }>          // baseline reject()/rejected() shape
type ConsoleMetadata = Readonly<{ lane: string; releaseCandidateSha: string
  version: Readonly<{ id?: string; tag?: string; timestamp?: string }>
  liveReleaseReadiness?: Readonly<{ ok: boolean; reason: string | null }> }>
```

Authored-line estimates above are design intent, not measurements. Every file stays under the 600-line ceiling (Requirement 13.7); the two largest — `theme-manifest.ts` and `release-controller.ts` — are the ones to watch, and each splits along an obvious behavior boundary (schema versus defaults; authorization versus candidate lifecycle) if it approaches the ceiling.
