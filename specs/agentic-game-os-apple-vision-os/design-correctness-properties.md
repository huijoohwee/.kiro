# Agentic Game OS Apple visionOS Correctness Properties

<!-- Responsibility: Define the complete executable property oracle set for the Agentic Game OS Apple visionOS contract; Exports: none -->

This file is normative. It expands the 35 property candidates in
`requirements.md` into 41 numbered properties while keeping `design.md` below
the authored-file size limit. A property passes only when its generator runs at
least 100 cases in the declared TypeScript or Swift harness, or when both
runtimes replay the same repository-owned vector corpus.

Every implementation test carries the tag:
`Feature: agentic-game-os-apple-vision-os, Property {number}: {property text}`.
Shrinking or replay output records the seed, generated case, source revision,
and runtime. No property may call a paid service or require network egress.

## Scene manifest properties

### Property 1 — TypeScript scene-manifest round trip

- **Class:** Round trip
- **Generator:** Arbitrary schema-valid `Scene_Manifest` values.
- **Covers:** 8.7, 8.8
- **Statement:** For every generated value `m`, parsing the canonical TypeScript print of `m` succeeds and returns a value equal to `m` in every canonical domain field.

### Property 2 — Swift scene-manifest round trip

- **Class:** Round trip
- **Generator:** The Property 1 corpus decoded and encoded by the Swift target.
- **Covers:** 8.7, 8.8, 8.9
- **Statement:** For every generated value `m`, Swift decode followed by canonical encode and decode returns a value equal to `m` in every canonical domain field.

### Property 3 — Scene-manifest printer determinism

- **Class:** Invariant
- **Generator:** Pairs of manifest values equal in every canonical domain field but produced through different accepted input texts.
- **Covers:** 8.6, 8.8
- **Statement:** Canonically equal inputs always print to byte-identical UTF-8 text.

### Property 4 — TypeScript and Swift decode agreement

- **Class:** Model based
- **Generator:** Valid, boundary-valid, and schema-invalid manifest texts from one shared corpus.
- **Covers:** 8.9
- **Statement:** TypeScript and Swift agree on acceptance; accepted values have equal canonical fields and rejected values have the same stable error code and field path.

### Property 5 — Scene-edit rollback preservation

- **Class:** Error condition
- **Generator:** Valid scenes paired with out-of-bounds edits and injected persistence failures.
- **Covers:** 8.11, 8.12
- **Statement:** Every rejected or unpersisted edit returns the required typed error while the live scene, stored revision, and manifest bytes remain equal to their pre-edit values.

### Property 6 — Live-edit application idempotence

- **Class:** Idempotence
- **Generator:** Valid scenes paired with arbitrary schema-valid single-field changes.
- **Covers:** 8.3, 8.10
- **Statement:** Applying one accepted canonical value and then committing that already-active value again produces the same live scene, persisted revision, and observable projection as applying it once; the second commit is an explicit no-op success.

## Ownership and portability properties

### Property 7 — Incomplete ownership audit remains incomplete

- **Class:** Error condition
- **Generator:** Ownership scans with arbitrary unreadable modules, timeouts, or missing consumer evidence.
- **Covers:** 4.9
- **Statement:** An incomplete scan returns `audit-incomplete`, names every unavailable subject, and never reports the scanned capability set as duplicate-free.

### Property 8 — Shared-substrate reuse

- **Class:** Invariant
- **Generator:** Schema-valid and invalid ownership manifests paired with complete static import/export/call graphs, selector matches, dynamic edges, and unreadable files.
- **Covers:** 4.4, 4.5
- **Statement:** Every selector-matched GameXR function has a transitive static call to its manifest-declared unique Knowgrph owner export; missing, dynamic, unresolved, or unreadable proof is incomplete rather than clean.

### Property 9 — Consumer pin mismatch is fail closed

- **Class:** Error condition
- **Generator:** Resolved revision and artifact-digest pairs with one or both pin fields absent or changed.
- **Covers:** 4.6, 4.7, 4.8
- **Statement:** Any absent or mismatched pin returns `pin-mismatch`, names every differing field, retains the completed build outputs with their pin-mismatched marking, and withholds promotion admission.

### Property 10 — Consumer pin integrity

- **Class:** Invariant
- **Generator:** Valid resolved dependency revisions, artifact bytes, and matching pin records.
- **Covers:** 4.6, 4.7
- **Statement:** A consumer is admitted only when the resolved revision and computed artifact digest exactly equal the recorded pin values.

### Property 11 — Frontend divergence confinement

- **Class:** Invariant
- **Generator:** Arbitrary module responsibility assignments plus exact change manifests for adding one Game_Mode.
- **Covers:** 5.1, 5.3, 5.11
- **Statement:** GameXR differences are confined to the four owned concerns; a new mode changes only that mode's modules plus one registration entry point and never a shared identifier list; ownership-manifest selectors find zero downstream duplicate-logic functions.

### Property 12 — Capability-tier totality

- **Class:** Invariant
- **Generator:** Arbitrary browser/native probe sets whose promises fulfill true, false, or indeterminate, reject, or never settle, with completion times around the 1000-ms deadline and arbitrary user agents.
- **Covers:** 6.1, 6.2, 6.3, 6.4, 6.10
- **Statement:** Detection completes within the bound and returns exactly one tier for every matrix; a nonempty determinate set containing no true value returns `flat-fallback` with `feature-detection` evidence.

### Property 13 — Cross-runtime capability parity

- **Class:** Metamorphic
- **Generator:** Shared domain inputs paired with any tier supported and `projected` on both targets and probe-result maps satisfying both targets' required rules.
- **Covers:** 6.8, 6.9
- **Statement:** Replacing the browser adapter with the native adapter preserves every canonical domain output while allowing only parity-matrix-declared presentation differences.

### Property 14 — Capability evidence never elevates a tier

- **Class:** Invariant
- **Generator:** Determinate feature evidence, indeterminate feature evidence paired with arbitrary user-agent strings, and fallback evidence.
- **Covers:** 6.3, 6.4, 6.12
- **Statement:** Determinate feature evidence alone may select a probe-dependent capability; user-agent and fallback evidence always select `flat-fallback` and grant no probe-dependent capability.

### Property 15 — Capability evidence source singularity

- **Class:** Error condition
- **Generator:** Resolution records containing zero, one, or multiple evidence sources for each decision field.
- **Covers:** 6.10, 6.11
- **Statement:** Any invalid evidence record returns `evidence-source` with a `flat-fallback` TierResolution preserving probes; callers use only that fallback, which may dispatch a capability only when it has no unsatisfied target-specific probe rule.

### Property 16 — Unsupported capability dispatch is impossible

- **Class:** Invariant
- **Generator:** Arbitrary capability, target, TierResolution probe-result map, tier, and parity-matrix target-rule combinations.
- **Covers:** 6.6, 6.7
- **Statement:** Dispatch occurs if and only if the tier is supported, the target is projected, and every target-specific probe rule has the required determinate value; every other combination returns the complete typed gap.

## Game-mode registry properties

### Property 17 — Game-mode identity uniqueness

- **Class:** Invariant
- **Generator:** Registration/unregistration sequences of 0 through 33 modes with duplicate identities and changed configurations, paired with arbitrary incumbent active modes.
- **Covers:** 5.2, 5.4, 5.5, 13.1
- **Statement:** The registry holds at most one entry per identity and up to 32 modes; duplicate or changed-config registration returns its named rejection and preserves incumbent config, count, and active mode, while an update is admitted only after explicit unregistration.

### Property 18 — Single active game overlay

- **Class:** Invariant
- **Generator:** Arbitrary missing-field registrations and known/unknown activations with successful, failing, timing-out, cleanup-failing, and restoration-failing handlers, call traces, and times around 2 seconds.
- **Covers:** 5.6, 5.9, 5.10
- **Statement:** Missing-field/unknown-ID requests return named payloads, invoke zero handlers, and preserve state; success calls exit then entry once within 2 seconds; after entry failure/timeout, requested cleanup and incumbent restoration each run once: both succeed yields `entry-failure` with incumbent republished, either fails yields `activation-recovery-failure` with zero published overlays; rejected modes never become active.

### Property 19 — Failed mode exit is atomic

- **Class:** Invariant
- **Generator:** Active-mode transitions with arbitrary failing or timing-out exit handlers.
- **Covers:** 5.7
- **Statement:** A failed exit invokes the incumbent exit handler exactly once, invokes the requested entry handler zero times, and leaves the incumbent mode, registry cardinality, and overlay owner unchanged.

## Continuity properties

### Property 20 — Continuity replay determinism

- **Class:** Invariant
- **Generator:** Seeds paired with ordered action sequences of 1 through 10,000 actions.
- **Covers:** 9.7
- **Statement:** Replaying the same seed and ordered actions produces the same revision index, world digest, and canonical world state.

### Property 21 — Continuity close-and-reopen round trip

- **Class:** Round trip
- **Generator:** Arbitrary committed world states and manifest revisions.
- **Covers:** 9.4
- **Statement:** Closing and reopening the world restores the newest valid committed revision with byte-identical canonical state.

### Property 22 — Durable append precedes acknowledgement

- **Class:** Invariant
- **Generator:** Action streams with arbitrary append delay and injected append failures.
- **Covers:** 9.3, 9.9
- **Statement:** No action is acknowledged before its journal record is durable; a failed append produces no acknowledgement and no revision advance.

### Property 23 — Invalid restore preserves stored bytes

- **Class:** Error condition
- **Generator:** Schema-invalid, digest-mismatched, truncated, and length-mismatched record byte strings.
- **Covers:** 9.5
- **Statement:** Restore returns the required unreadable-record error, preserves every stored byte, and exposes only the explicit reset operation.

### Property 24 — Write-lease exclusivity and expiry

- **Class:** Invariant
- **Generator:** Interleaved session writes and renewals across arbitrary gaps around the 30-second lease boundary.
- **Covers:** 9.6, 9.10
- **Statement:** At most one unexpired session can append; another session is admitted only after operation-derived observation proves the incumbent lease expired.

### Property 25 — Offline intent queue ordering

- **Class:** Invariant
- **Generator:** Intent sequences of 0 through 10,000 entries with arbitrary restart and reconnect points.
- **Covers:** 9.8
- **Statement:** Restart and reconnect preserve FIFO order, monotone sequence identities, canonical payload bytes, and payload digests byte-for-byte.

### Property 26 — Offline intent queue overflow is fail closed

- **Class:** Error condition
- **Generator:** Full queues paired with arbitrary additional intents.
- **Covers:** 9.11
- **Statement:** The 10,001st pending intent returns `queue-full`, names the capacity, and leaves all existing entries byte-identical and ordered.

## Invocation and document properties

### Property 27 — Invocation-token resolution uniqueness

- **Class:** Invariant
- **Generator:** Arbitrary token strings and valid entries across the command, semantic, and binding dictionaries.
- **Covers:** 3.1, 3.2, 3.5, 3.7
- **Statement:** Every declared `/`, `#`, or `@` token resolves to exactly one Invocation_SSOT entry and no undeclared token resolves.

### Property 28 — Duplicate or malformed invocation rejection

- **Class:** Invariant
- **Generator:** Invocation strings with arbitrary duplicated, malformed, unknown, and mixed prefixes.
- **Covers:** 3.3, 3.4, 3.8
- **Statement:** The resolver rejects every invalid prefix form with its stable typed code, reads no unrelated dictionary, and applies no substitution.

### Property 29 — Frontmatter contract validity

- **Class:** Invariant
- **Generator:** Arbitrary authored-document presence, field sets, values, duplicates, extra keys, part headings, and reference placements.
- **Covers:** 1.2, 1.3, 1.4, 1.5, 1.9, 1.10, 1.12
- **Statement:** An absent document returns `document-missing` without creating a file; otherwise validation succeeds exactly when all eleven fields are unique and non-empty, all three parts exist, and every concrete implementation reference is under a reference-implementation heading.

### Property 30 — Readiness-rung ordering

- **Class:** Invariant
- **Generator:** Arbitrary local, Readiness_Subject, and delivered rung values, physical-gate states, 41-property result sets, delivery-verification records, and Composite_Candidate digests, including values outside the closed enumerations.
- **Covers:** 1.7, 14.8
- **Statement:** Every accepted rung is in the closed ordering; production is withheld unless both frontmatter rungs and every subject are runtime-ready and every property, physical matrix, native artifact, and delivery receipt is valid for one identical Composite_Candidate digest.

### Property 31 — Authored path portability

- **Class:** Invariant
- **Generator:** Authored texts containing arbitrary path-like literals, account names, and `$GITHUB_ROOT`-relative references.
- **Covers:** 11.3, 11.4, 11.9
- **Statement:** Admission succeeds only when no machine-root or account-name literal exists and every repository reference resolves from `$GITHUB_ROOT`.

### Property 32 — File-size and responsibility bound

- **Class:** Invariant
- **Generator:** Authored files with arbitrary line counts, normalized responsibility-record counts, declared static export-name sets, discovered static export-name sets, and dynamic exports.
- **Covers:** 1.8, 1.11, 11.5, 11.6, 11.10, 11.11
- **Statement:** Admission succeeds only at 600 lines or fewer, with exactly one grammar-valid responsibility record, no dynamic export, and exact equality between declared and statically discovered export-name sets; every other case returns the typed size or single-responsibility rejection.

## Cost, agent surface, and promotion properties

### Property 33 — Zero-cost play-loop accounting

- **Class:** Invariant
- **Generator:** Arbitrary bounded play-loop action sequences.
- **Covers:** 12.1
- **Statement:** Every loop step produces exactly one complete cost record with null model identity, zero tokens, zero estimated cost, and no paid call.

### Property 34 — Cost-entry completeness

- **Class:** Error condition
- **Generator:** Model-bearing invocation records with arbitrary required fields absent or invalid.
- **Covers:** 12.2, 12.9
- **Statement:** Any incomplete record returns `cost-accounting`, names every absent field in one response, and withholds the invocation completion claim.

### Property 35 — Bounded-loop termination

- **Class:** Invariant
- **Generator:** Harness loops with limits from 1 through 100 and arbitrary never-satisfied circuit breakers.
- **Covers:** 12.3, 12.8
- **Statement:** Every loop terminates at or before its declared limit and returns the typed iteration-limit result when its breaker never succeeds.

### Property 36 — Status read is non-mutating

- **Class:** Invariant
- **Generator:** Arbitrary continuity states paired with status requests.
- **Covers:** 13.1, 13.2, 13.3, 13.7
- **Statement:** One status request returns all five fields within one second, records one zero-cost read, and leaves every continuity byte unchanged.

### Property 37 — Control operation schema is closed

- **Class:** Invariant
- **Generator:** Declared operation identifiers paired with arbitrary valid and invalid argument key sets.
- **Covers:** 13.4
- **Statement:** Exactly one declared operation with schema-valid arguments is admitted; no undeclared identifier or argument key reaches an effect handler.

### Property 38 — Rejected control preserves state

- **Class:** Error condition
- **Generator:** Unknown, multiple, malformed, concurrent, and out-of-scope control requests.
- **Covers:** 13.5, 13.8, 13.9
- **Statement:** Every rejection names the offending operation or key, applies zero effects, and leaves all continuity records byte-identical while an incumbent operation completes unchanged.

### Property 39 — Deploy gate is fail closed

- **Class:** Invariant
- **Generator:** Prod and Delivery requests with absent, mismatched, expired, consumed, erroring, and timing-out authorizations, plus Dev requests from wrong-branch, tracked-dirty, and untracked-dirty worktrees.
- **Covers:** 10.3, 10.4, 10.5, 10.6, 10.10
- **Statement:** Delivery verification occurs only with one valid unconsumed authorization per gated stage; invalid gated requests return `authorization-missing`, apply zero target-stage effects, and retain exactly one outcome audit entry, while unclean Dev requests return `unclean-worktree` and start no process; delivery verification alone never emits a feature-readiness promotion.

### Property 40 — Pipeline digest continuity

- **Class:** Invariant
- **Generator:** Composite_Candidate manifests, four-repository revisions, shared pins, and delivery artifact digests across Dev, mirror, and delivery, including arbitrary missing or changed values and repeated requests per stage.
- **Covers:** 10.1, 10.7, 10.8, 10.11, 10.12
- **Statement:** `delivery-verified` is emitted only when all stages name the same Composite_Candidate manifest digest and byte-identical delivery artifact digest, with exactly one immutable outcome entry per gated request outcome; the pipeline emits no `runtime-ready` or `production-runtime-ready` label.

### Property 41 — Validation coverage completeness

- **Class:** Invariant
- **Generator:** Arbitrary subsets of acceptance-criterion identifiers paired with validation-checklist entries and evidence records.
- **Covers:** 14.2, 14.5, 14.6, 14.7
- **Statement:** Readiness review succeeds only when every acceptance criterion has a re-invocable command and valid exact-revision evidence; otherwise one coverage or evidence error names the complete missing set and lowers the affected rung.

## Cross-runtime execution contract

Properties 1 through 41 are executed from repository-owned commands only.
Properties whose subject has both TypeScript and Swift implementations use the
exact pinned `fast-check` and `SwiftCheck` dependencies, or one shared canonical
vector corpus when the stable Swift baseline cannot run SwiftCheck. Document,
invocation, pipeline, and other single-runtime properties execute only their
named owning runtime and oracle; they never fabricate a Swift peer. A
production-readiness receipt records every property number, iteration count,
seed or corpus digest, Composite_Candidate digest, command, exit status, owning
runtime, and either the applicable TypeScript/Swift agreement result or an
explicit cross-runtime `not-applicable` value.
