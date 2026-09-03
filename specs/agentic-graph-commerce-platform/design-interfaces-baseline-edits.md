---
title: "AgenticGraph Commerce Platform — Design Annex A1: Typed Contracts, Requirements 1–6"
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

# Annex A1 — Typed Contracts, Requirements 1–6 (edits to built files)

Companion to Annex A2 (`design-interfaces-new-surfaces.md`), which covers Requirements 7–13. Split along the owner boundary between edits to built files and genuinely new surfaces, per Requirement 13.7's 600-line ceiling.

Element numbers match the Components and Interfaces table in `design.md`. Every block states the disposition and the target file. Conventions carried from the baseline without change: `Readonly<{...}>` result shapes, `Object.freeze` on returned records, `{ ok: false, code }` rejections, canonical JSON plus SHA-256 for every digest (`src/shared/digest.ts`), zero `any`, zero index signatures on payload types.

## Requirement 1 — Terminology (elements 1–5)

Requirement 1 is deliberately the smallest edit in the increment. **Zero `src/` value changes.** The two `src/invocation/catalog.ts` occurrences and the six `wrangler.core.jsonc` occurrences name upstream-owned targets and are dispositioned `externally-owned`; the one `test/invocation/client.test.ts` fixture mirrors the upstream endpoint under test and follows that disposition.

```ts
// 1 — config/terminology-register.json, typed by src/shared/terminology-register.ts (NEW)
export type Disposition = 'renamed' | 'externally-owned' | 'historical-record'
export type RegisterOccurrence = Readonly<{
  path: string; line: number
  legacyIdentifier: string; supersedingIdentifier: string
  disposition: Disposition
  owningSystem?: string; reason?: string        // required when 'externally-owned' (1.2)
  preservingArtifact?: string; readOnly?: true  // required when 'historical-record' (1.3)
}>
export type TerminologyRegister = Readonly<{
  schema: 'agentic-graph-terminology-register/v1'
  fileScope: Readonly<{ include: readonly string[]; exclude: readonly string[] }>
  matcher: Readonly<{
    boundaryAware: true
    caseSensitiveTerms: readonly string[]     // 'KG_', 'kgc'
    caseInsensitiveTerms: readonly string[]   // 'knowgrph', 'knowledgegraph', 'knowledge-graph'
  }>
  occurrences: readonly RegisterOccurrence[]
}>
export function readTerminologyRegister(value: unknown): TerminologyRegister | null
export function requiredDispositionFields(entry: RegisterOccurrence): readonly string[]
```

```ts
// 2 — scripts/validate-terminology-register.ts (NEW)
// Modeled on the existing scripts/validate-do-storage-compatibility.ts: Node only, no model call,
// no paid call, deterministic, exits non-zero with per-occurrence file:line naming (1.9).
export type TerminologyFinding = Readonly<{
  path: string; line: number; column: number; matchedTerm: string
  finding: 'undispositioned' | 'renamed-occurrence-remains' | 'disposition-fields-missing'
}>
export type TerminologyVerdict = Readonly<{
  ok: boolean
  scannedFileCount: number
  matchedOccurrenceCount: number
  dispositionCounts: Readonly<Record<Disposition, number>>
  findings: readonly TerminologyFinding[]   // empty iff ok
}>
export function validateTerminology(rootDirectory: string): TerminologyVerdict
// Matcher: /(?<![A-Za-z0-9])(term)(?![A-Za-z0-9])/ per term, case-sensitive for KG_ and kgc.
// Boundary awareness is what eliminates the two package-lock.json base64 false positives; file
// scope excludes them regardless, so the two guards are independent.
```

```ts
// 3 — src/shared/terminology-guard.ts (NEW). ~40 authored lines.
export type LegacyIdentityRejection = Readonly<{
  ok: false; code: 'legacy_identifier_rejected'
  presented: string; supersedingIdentifier: string
}>
// The rejection table is derived from register entries whose disposition is 'renamed' and whose
// kind is 'endpoint' or 'tool'. At first build that table is EMPTY, because no baseline endpoint or
// tool identity carries a legacy form. The guard is the mechanism; CP-1 exercises it over generated
// register entries. Called from src/edge/index.ts route resolution and from src/core/index.ts
// before token resolution; returns null when the presented identity is not a legacy form.
export function rejectLegacyIdentity(
  presented: string, table: readonly Readonly<{ legacy: string; superseding: string }>[],
): LegacyIdentityRejection | null
```

Element 4 (`src/invocation/catalog.ts`): no value change; the file is listed as MODIFIES only because its two occurrences gain register entries and a `// externally-owned per terminology-register.json` comment. Element 5 (`README.md`, `docs/production-runtime.md`): prose renames `Knowgrph` → `AgenticGraph` and the PRD filename → `agentic-graph-commerce-platform-prd-tad-adr.md`, except the two lines recording the baseline PRD revision, which stay `historical-record`.

## Requirement 2 — Convergence (elements 6–8)

Minimal edit, stated precisely. In `verifyUpstreamRuntimeEvidence` exactly three expressions change: the envelope key-set equality checks become required-key presence plus an unnamed-field count; `exactChecks` becomes required-set satisfaction plus a surplus list; `evidence.prdRevision === COMMERCE_PRD_REVISION` becomes the major/minor rule. `digestUpstreamRuntimeEvidence` is untouched, so its fixed six-field input keeps the integrity binding intact under envelope surplus.

```ts
// 6 — src/core/convergence-evaluator.ts (NEW). Pure; one SHA-256; no I/O; no model call.
export type ConvergenceState = 'converged' | 'converged-with-surplus' | 'blocked'
export type ContractRevision = Readonly<{ major: number; minor: number; patch: number }>
export type DeclaredRequirements = Readonly<{
  provider: string
  expectedContract: string
  requiredCheckSet: readonly string[]         // CHECKOUT_ or MARKETPLACE_EVIDENCE_CHECKS
  declaredContractRevision: ContractRevision  // parsed from COMMERCE_PRD_REVISION
  pin: UpstreamEvidencePin                    // identity fields, exact equality (2.8)
}>
export type ConvergenceVerdict = Readonly<{
  provider: string
  state: ConvergenceState
  reason: string | null
  requiredSatisfied: readonly string[]
  requiredAbsentOrFailing: readonly string[]        // 2.3
  surplusChecks: readonly string[]                  // 2.2
  unnamedEnvelopeFieldCount: number                 // 2.7
  declaredContractRevision: string
  advertisedContractRevision: string | null         // 2.6 records both
  identityFailures: readonly Readonly<{ field: string; reason: string }>[]  // 2.8
}>
export function parseContractRevision(value: unknown): ContractRevision | null
export function revisionForwardCompatible(declared: ContractRevision, advertised: ContractRevision): boolean
//   advertised.major === declared.major && advertised.minor >= declared.minor   (2.5 / 2.6)
export function evaluateConvergence(
  advertised: unknown, declared: DeclaredRequirements,
): Promise<ConvergenceVerdict>
```

```ts
// 7 — src/core/upstream-evidence.ts (MODIFIES). Exported names and pin reader unchanged.
export async function verifyUpstreamRuntimeEvidence(
  value: unknown, expectedContract: string,
  pin: UpstreamEvidencePin | null, requiredChecks: readonly string[],
): Promise<Readonly<Record<string, unknown>>>
// Now a thin adapter: builds DeclaredRequirements, delegates to evaluateConvergence, and returns
// { ok, code, verdict, ...existing detail fields } so every existing caller keeps its shape.
// ok === (state !== 'blocked'). code === null when ok, else the blocking reason.
```

```ts
// 8 — src/core/index.ts + src/edge/index.ts (MODIFIES). Readiness composition only.
export type ReadinessReport = Readonly<{
  ok: boolean                    // sourceReadiness.ok && liveReleaseReadiness.ok
  contract: 'commerce.core-readiness/v2'
  lane: string; releaseCandidateSha: string; version: WorkerVersionMetadata
  sourceReadiness: Readonly<{ ok: boolean; checks: readonly unknown[] }>          // 2.11 / 5.10
  liveReleaseReadiness: Readonly<{ ok: boolean; reason: string | null; servingCandidateSha: string | null }>
  convergence: readonly ConvergenceVerdict[]   // exactly one per declared provider (2.9)
}>
// Dev lane: liveReleaseReadiness = { ok: false, reason: 'delivery_route_unauthorized_in_dev',
// servingCandidateSha: null } and issues zero request, preserving 5.2.
```

## Requirement 3 — Take-rate (elements 9–12)

```ts
// 9 — src/core/take-rate.ts (NEW). ~60 authored lines. Integer-only. No I/O.
export const MAXIMUM_RATE_BASIS_POINTS = 1_000
export function readRateBasisPoints(value: unknown): number | null
// integer, 1..1000 inclusive; null for absent, non-numeric, negative, zero, or above the ceiling (3.7)
export function computeMarkupMinor(settledAmountMinor: number, rateBasisPoints: number): number
// half-up on minor units: Math.floor((amount * bp + 5_000) / 10_000)
// guarantees 0 <= result <= settledAmountMinor for bp <= 10_000 (3.4)
```

```ts
// 10 — src/core/revenue-ledger.ts (NEW). SQLite Durable Object. ~180 authored lines.
export type RevenueLine = Readonly<{
  settlementId: string; agentId: string
  settledAmountMinor: number; currency: string
  appliedRateBasisPoints: number; markupMinor: number
  recordedAtMs: number; recordedAt: string
}>
export class RevenueLedger extends DurableObject<CoreEnv> {
  // INSERT ... ON CONFLICT(settlement_id) DO NOTHING → idempotent (3.5 / CP-6)
  appendLine(line: RevenueLine): Promise<Readonly<{ ok: true; idempotent: boolean }> | Rejected>
  readPeriod(startInclusiveMs: number, endExclusiveMs: number, limit: number): Promise<Readonly<{
    ok: true
    lines: readonly RevenueLine[]           // ORDER BY recorded_at_ms ASC, settlement_id ASC (3.10)
    summedMarkupMinor: number               // folded from `lines`, never a maintained total
    lineCount: number
  }> | Rejected>
  demandEvidence(): Promise<Readonly<{
    ok: true
    principalsWithTwoOrMoreSettlements: number   // 3.11 — agent identifiers, see Annex B partial note
    reportedAs: 'capability'
  }>>
}
```

```ts
// 11 — src/core/checkout-session.ts (MODIFIES). One hook after `settlement_recorded`.
// Inserted immediately after the existing transactionSync that sets state = 'settled':
//   1. computeMarkupMinor(settledAmountMinor, rate)
//   2. env.REVENUE_LEDGER.getByName(env.REGISTRY_ID).appendLine(line)
//   3. append 'markup_recorded' evidence event
// On three consecutive failures within 5 s: append 'markup_deferred' { settlementId, failingStage }
// and return the unchanged settled result (3.8). The settled row is never rewritten, which is what
// makes 3.6 structural rather than asserted.
type MarkupOutcome =
  | Readonly<{ stage: 'recorded'; markupMinor: number; idempotent: boolean }>
  | Readonly<{ stage: 'deferred'; failingStage: 'compute' | 'append'; attempts: 3 }>
```

```ts
// 12 — src/core/index.ts (MODIFIES). Fail-closed configuration gate (3.7).
// Added to readiness() as check('take_rate_configuration', ...) and as an early return in fetch():
//   { ok: false, code: 'take_rate_configuration_invalid',
//     key: 'AG_TAKE_RATE_BASIS_POINTS', condition: 'absent' | 'non_numeric' | 'negative'
//       | 'zero' | 'above_maximum' }  → HTTP 503, zero ledger append
```

## Requirement 4 — Template Pack (elements 13–17)

```ts
// 13 — src/shared/theme-manifest.ts (NEW). ~190 authored lines. Shared by edge and core.
export type ThemeManifest = Readonly<{
  merchantId: string
  palette: Readonly<{ ink: string; muted: string; line: string; panel: string; accent: string; background: string }>
  logo: Readonly<{ href: string | null; alt: string }>
  copy: Readonly<{ brand: string; headline: string; subhead: string; footer: string }>
  catalogScope: readonly string[]     // 1..500 agent identifiers
  locale: string
}>
export type ThemeViolation = Readonly<{ field: string; reason: string }>
export type ThemeManifestVerdict =
  | Readonly<{ ok: true; manifest: ThemeManifest; defaultedFields: readonly string[]; digest: string }>
  | Readonly<{ ok: false; violations: readonly ThemeViolation[] }>
export const THEME_MANIFEST_DEFAULTS: ThemeManifest            // baseline console values
export const THEME_MANIFEST_MAX_BYTES = 65_536
export function validateThemeManifest(value: unknown): Promise<ThemeManifestVerdict>
// rejects unknown keys; bounds every text field at 280 chars; every violating field is named (4.2)
export function serializeThemeManifest(manifest: ThemeManifest): string   // canonical, round-trip stable (CP-8)
```

```ts
// 14 — src/core/theme-deployment.ts (NEW). ~150 authored lines.
export type AssetFetchFailure = Readonly<{ href: string; attempts: 3; lastStatus: number | null }>
export type DeploymentResult =
  | Readonly<{ ok: true; merchantId: string; manifestDigest: string; deployedAt: string
      resolvedCatalogScope: readonly string[]; defaultedFields: readonly string[] }>   // 4.10
  | Readonly<{ ok: false; code: 'theme_manifest_invalid'; violations: readonly ThemeViolation[] }>
  | Readonly<{ ok: false; code: 'theme_scope_agent_not_registered'; agentIds: readonly string[] }>  // 4.3
  | Readonly<{ ok: false; code: 'theme_asset_unreachable'; failures: readonly AssetFetchFailure[] }> // 4.9
export function deployTheme(env: CoreEnv, value: unknown): Promise<DeploymentResult>
// Order: validate → registry scope check → asset fetch (3 attempts / 30 s) → activate.
// Every failure path leaves the prior activation serving unchanged.
```

```ts
// 15 — src/core/theme-deployment-store.ts (NEW). SQLite DO, one row per merchant.
export class ThemeDeployment extends DurableObject<CoreEnv> {
  activate(record: ActivatedTheme): Promise<Readonly<{ ok: true; idempotent: boolean }> | Rejected>
  current(): Promise<Readonly<{ ok: true; record: ActivatedTheme | null }>>
}
```

```ts
// 16 — src/edge/dashboard.ts (MODIFIES). One renderer, two surfaces (4.1).
export function consoleResponse(
  metadata: ConsoleMetadata, manifest: ThemeManifest = THEME_MANIFEST_DEFAULTS,
): Response
// With the default manifest the rendered bytes equal the baseline operator console, which is how the
// existing test/workers/edge.test.ts assertions survive the change.
// Adds: 360px breakpoint rules with zero horizontal overflow (9.6); every interactive control a
// native element with a non-empty accessible name and min 44x44 CSS px (4.7); an offline indicator
// region (9.1); one nonce-bound same-origin module tag when the WebMCP surface is enabled (7.1).
// CSP becomes: default-src 'none'; style-src 'unsafe-inline'; script-src 'nonce-<per-response>';
// connect-src 'self'; img-src 'self' https:; base-uri 'none'; form-action 'none'; frame-ancestors 'none'
export function dashboardResponse(metadata: ConsoleMetadata): Response   // retained, delegates
```

```ts
// 17 — src/core/merchant-catalog.ts (NEW). ~90 authored lines.
export function projectMerchantCatalog(
  listings: readonly Listing[], catalogScope: readonly string[],
): readonly Listing[]      // pure filter; every result's owning agent is in scope (CP-9)
export function readListing(
  listings: readonly Listing[], catalogScope: readonly string[], listingId: string,
): Listing | null          // null → typed not-found at the route (4.5)
```

## Requirement 5 — Environment chain (elements 18–20)

```jsonc
// 18 — package.json (MODIFIES). Exactly two Dev entry points (5.1).
"dev":      "wrangler dev -c wrangler.edge.jsonc -c wrangler.core.jsonc -c wrangler.dev-provider.jsonc --env dev",
"dev:apex": "wrangler dev -c wrangler.edge.jsonc -c wrangler.core.jsonc -c wrangler.dev-provider.jsonc --env dev --port 5173"
// deploy:production:* become controller-only: each gains a guard that exits non-zero unless invoked
// by scripts/release-controller.ts with a verified authorization record.
```

```jsonc
// 19 — wrangler.edge.jsonc (MODIFIES). Production env only (5.3).
"routes": [{ "pattern": "airvio.co/agentic-commerce-os", "zone_name": "airvio.co" }]
```

```ts
// 20 — scripts/release-controller.ts (NEW). ~280 authored lines, Node only.
export type ReleaseAuthorization = Readonly<{
  candidateSha: string            // 40-hex, sealed candidate identity
  target: 'prod-mirror' | 'delivery-route'
  humanIdentity: string           // authenticated reviewer
  authorizedAtMs: number          // must be within 24 h (5.6)
  rollbackDisposition: Readonly<{ priorCandidateSha: string; restoreAction: string }>  // 5.8
}>
export type ControllerVerdict =
  | Readonly<{ ok: true; advanced: 'prod-mirror' | 'delivery-route'; candidateSha: string }>
  | Readonly<{ ok: false; code: 'release_authorization_incomplete'; missingElement: string
      requestedCandidateSha: string | null; requestedTarget: string | null }>
  | Readonly<{ ok: false; code: 'release_candidate_retired'; candidateSha: string; frontierSha: string }>
  | Readonly<{ ok: false; code: 'release_rollback_disposition_required'; candidateSha: string }>
export function authorizeAndAdvance(request: unknown): Promise<ControllerVerdict>
export function sealCandidate(sha: string, frontierSha: string): Promise<SealResult>
export function retireOnFrontierAdvance(frontierSha: string): Promise<readonly string[]>   // 5.7
```

## Requirement 6 — Invocation surface (elements 21–23)

```ts
// 21 — config/capability-token-map.json (NEW), typed by src/invocation/capability-map.ts (NEW)
export type CapabilityTokens = Readonly<{
  capabilityAction: string      // e.g. 'revenue.period.read'
  commandToken: string          // exactly one '/' token (6.2)
  semanticTokens: readonly string[]   // at least one '#'
  bindingTokens: readonly string[]    // at least one '@'
  mcpTool: string               // the tool that action is reachable through (6.7)
  httpRoute: string | null      // never the only reachable path
}>
export function readCapabilityMap(value: unknown): readonly CapabilityTokens[] | null
export function coverageFindings(
  map: readonly CapabilityTokens[], catalog: InvocationCatalogSnapshot,
): readonly string[]            // empty iff every action's tokens resolve in the pinned catalog
```

```ts
// 22 — src/core/index.ts and src/core/agent-registry.ts (MODIFIES). Bounds only.
// readTokenArray: the hard cap of 12 becomes the register-declared required-token count
//   (config/capability-token-map.json length, bounded at CATALOG_LIMIT).
// validInvocationProof: `requiredTokens.length === 3` becomes
//   `requiredTokens.length === declaredRequiredTokenCount && length >= 3`.
// resolvePinnedInvocations already returns sourceRevision, catalogDigest, routingSchema,
// routingDigest, counts, and invocations — 6.3 is satisfied at the baseline and is asserted,
// not built. Revision transitions add: { priorSourceRevision, sourceRevision } before serving (6.8).
```

```ts
// 23 — src/edge/index.ts (MODIFIES). Second mount, one catalog.
// POST /mcp          → MCP_BEARER_TOKEN, agent-facing tools (unchanged set plus public catalog,
//                      merchant catalog, revenue period read)
// POST /mcp/operator → OPERATOR_BEARER_TOKEN, operator capability tools (agent registration,
//                      deregistration, registry events, vendor transition, theme deployment,
//                      release boundary read, claim acquire/release)
// Both mounts resolve every action through the one Invocation_Catalog. No second token registry.
```

