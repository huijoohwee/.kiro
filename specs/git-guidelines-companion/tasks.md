# Implementation Plan: git-guidelines-companion

## Overview

Three coupled deliverables, built in dependency order:

1. **Git_Guidelines_Document** — `huijoohwee.github.io/docs/documents/git-guidelines.md`, 16 `##` sections, 392 allocated lines of a 400 cap.
2. **Registrations** — `agentic-canvas-os/docs/README.md`, `DICTIONARY-COMMAND.md`, `DICTIONARY-SEMANTIC.md`, `DICTIONARY-BINDING.md`.
3. **Conformance_Checker** — `huijoohwee.github.io/scripts/check-git-guidelines.mjs` plus 17 modules under `scripts/lib/git-guidelines/`.

Implementation language is **JavaScript, Node 22 ESM** — fixed by the design's runtime decision. The checker carries **zero runtime dependencies** (`node:fs`, `node:path`, `node:url`, `node:crypto`, `node:child_process` only). `fast-check@3.23.2` and `js-yaml@4.1.1` are dev-only and never imported by shipped checker code.

Sequencing constraints carried from the design:

- `input-resolver`, `normalizer`, and `rule-registry` precede every check family — every finding needs a derived Rule_ID.
- The `Blocked_Outcome` Glossary term precedes every section that references it.
- Registration tasks require the document to exist at its path (R11.13 existence check plus two-directional frontmatter parity).
- All 17 document-authoring tasks write one file, so they are scheduled in successive waves rather than in parallel.

## Tasks

- [x] 1. Operator decisions and project scaffolding

  - [x] 1.1 Record the four resolved Operator decisions
    - Create `.kiro/specs/git-guidelines-companion/open-decisions.md` with the prior default, resolved value, and required implementation consequence for each decision
    - Record exactly: concurrency = protected-upstream unlimited pairwise-disjoint current authorities; lease ceiling = 24 h; divergent artifacts = fix; checker location = `huijoohwee.github.io/scripts/`
    - State that Decision 1 consumes protected upstream and removes the former document-local O1 numeric cap; Decision 2 remains a document-local O2 ceiling
    - Record the owner-led retirement/handoff/reclaim and fresh-lane authorization as provenance only; it does not bypass live claim, lease, fence, preservation, or protected-integration checks
    - _Requirements: 4.1, 4.7, 4.9, 5.3, 5.13, 13.2_

  - [x] 1.2 Scaffold the checker entry point, module directory, and test layout
    - Create `huijoohwee.github.io/scripts/check-git-guidelines.mjs` as a Node ESM entry that initializes `verdict = "not-conformant"` before any family runs and implements the 0/1/2/3 exit-status contract
    - Create empty module directory `huijoohwee.github.io/scripts/lib/git-guidelines/`
    - Create `scripts/__pbt__/` and `scripts/__tests__/` plus `scripts/__tests__/fixtures/conformant/` and `scripts/__tests__/fixtures/mutants/`
    - Add `devDependencies` to `huijoohwee.github.io/package.json`: `fast-check` at exactly `3.23.2`, `js-yaml` at exactly `4.1.1`; add no `dependencies`
    - _Requirements: 13.2, 13.3, 13.7_

- [x] 2. Checker foundation — input resolution, normalization, rule identity, reporting

  - [x] 2.1 Implement `scripts/lib/git-guidelines/input-resolver.mjs`
    - Resolve the five input groups once into frozen objects: document bytes, Docs_Index + 3 dictionaries, 5 owner documents, `.coordination/`/`.agentic-manifests/`/`.recovery/`/`.backups/`, and git facts via `rev-parse`, `status --porcelain`, `symbolic-ref`, `worktree list --porcelain`, `ls-remote`
    - Emit `input-absent`, `input-unparseable`, `input-stale`, `input-unreadable`, `remote-unreachable`; map to exit 2 for the four degraded conditions and exit 3 for the 30 s remote bound and the 60 s verdict bound
    - Implement the 10 s reachability probe defining online/offline
    - _Requirements: 5.14, 13.8, 13.10, 13.11_

  - [x] 2.2 Implement `scripts/lib/git-guidelines/normalizer.mjs`
    - Apply exactly the six normalization classes in order: line endings, trailing whitespace, absolute path prefix stripping, ISO-8601 → `<TS>`, `runId`/`operationId`/`sessionId`/`idempotencyKey` → `<RUN>`, ordering-insensitive metadata sorted ascending by byte order
    - Export the exclusion list as data so the sensitivity half of Property 17 can assert it: no interior whitespace collapsing, case folding, Unicode normalization, comment stripping, quote unification, number reformatting, or separator translation
    - _Requirements: 13.5_

  - [x] 2.3 Implement `scripts/lib/git-guidelines/rule-registry.mjs`
    - Derive every Rule_ID as `<section-anchor>#<1-based ordinal>` counted in document order across the section's directive bullets and table rows; record `ruleText` alongside each ID
    - Emit `rule-id-duplicate`, `rule-ordinal-mismatch`, `rule-unclassified`, `rule-multiclassified`; classify each rule as artifact-bearing or advisory, exactly one
    - Export a Rule_ID index consumed by every other family; this module runs before all of them
    - _Requirements: 12.1, 12.2, 12.10_

  - [ ]* 2.4 Write property test for Rule_ID derivation
    - **Property 6: Rule_ID derivation, uniqueness, and ordinal agreement**
    - File `scripts/__pbt__/git-guidelines-structure.pbt.test.mjs`
    - Generator domain: 1–20 sections, each with 0–40 rules interleaving bullets and table rows, including duplicate rule text within a section, duplicate section titles producing colliding anchors, and rules inserted before existing rules
    - **Validates: Requirements 12.1, 12.10**

  - [x] 2.5 Implement `scripts/lib/git-guidelines/report.mjs`
    - Sort by `[anchor asc byte order, ordinal asc numeric, line asc, column asc, type asc]`; dedup **after** sorting on `(ruleId, location)` folding to max severity plus `repeatCount`; severity order `minor < major < blocker`
    - Emit the `Report` shape: verdict, `findingTotal`, `severityCounts` with explicit zeros, `typeCounts` keyed by the full 12-type registry with explicit zeros, ascending `unsatisfiedRuleIds`, `inputStatus`, `blockedOutcomes`, `exitStatus`
    - Lower `verdict` to `conformant` only after every family has completed and only when zero `blocker`, zero `major`, and the Docs_Index row and `/` token are present
    - Record a `checker-internal-error` finding and discard that family's findings if a family throws
    - _Requirements: 12.5, 13.3, 13.6, 13.9_

  - [ ]* 2.6 Write property test for finding-set totality
    - **Property 3: Finding-set totality**
    - File `scripts/__pbt__/git-guidelines-reporting.pbt.test.mjs`
    - Generator domain: `k` drawn 0–24; defect kinds drawn from missing frontmatter key, out-of-domain value, brand outside a reference-implementation block, unindexed section, unmapped stage, orphan findings-table row, absent anti-pattern, out-of-scope path, differing restored path
    - **Validates: Requirements 1.11, 2.10, 4.13, 6.14, 7.8, 9.13, 11.8, 12.5, 13.9**

  - [ ]* 2.7 Write property test for idempotence and out-of-class sensitivity
    - **Property 17: Checker idempotence under the stated normalization, and sensitivity outside it**
    - File `scripts/__pbt__/git-guidelines-reporting.pbt.test.mjs`; needs a two-run harness with a controlled clock and a perturbation generator
    - Generator domain: inside classes — CRLF and lone-CR rewrites, trailing whitespace, absolute vs relative path prefixes, substituted ISO-8601 instants, substituted run/session/operation/idempotency identifiers, permuted frontmatter key order, permuted unordered sequences. Outside classes — interior whitespace, case changes, NFC/NFD variants, edited comment text, changed quote style around a reserved-punctuation scalar, reordered table rows
    - **Validates: Requirements 13.4, 13.5, 13.6**

- [x] 3. Checkpoint — foundation
  - Ensure all tests pass, ask the user if questions arise.

- [x] 4. Document spine — frontmatter, boundary, Module Index, Glossary, load budget

  - [x] 4.1 Author the frontmatter and title block of `huijoohwee.github.io/docs/documents/git-guidelines.md`
    - Acceptance target: **21 physical lines** — 17 frontmatter (2 delimiters + 5 baseline + 5 conformance + 5 optional) + 4 title block
    - Baseline `title`, `doc_type`, `version`, `date`, `lang`; conformance `owner` (single scalar), `local_rung`, `delivered_rung` (two separate keys), `lane: authoring`, `universal_scope: false`; optional `companion_of`, `invocation_token: "/git.guidelines"`, `semantic_filters: ["#git-collaboration"]`, `bindings: ["@git-guidelines"]`, `frontmatter_contract`
    - Quote every scalar carrying reserved punctuation; frontmatter is the first block with zero preceding bytes; no overrun justification key, since allocation closes at 392
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8, 1.9, 11.6_

  - [x] 4.2 Author the `boundary--ownership` section
    - Acceptance target: **21 lines** — heading 1 + blank 1 + 2 table header + 16 rows + trailing blank 1
    - Exactly 16 rows: the 9 `consumes` rows C1–C9 and the 7 `owns` rows O1–O7 from the design's boundary table; the five Collaboration_Module rows C1–C5 stay separate and are never merged
    - Every one of the five owners appears as the owner of at least one `consumes` row, each with a resolvable relative link; name the Delivery_Guidelines link for commit/push/deploy sequences
    - _Requirements: 2.1, 2.2, 2.3, 2.5, 12.2_

  - [x] 4.3 Author the `module-index` section
    - Acceptance target: **18 lines** — heading 1 + blank 1 + 15 entries + trailing blank 1
    - Exactly 15 entries, one per `##` section other than the Module Index itself; each entry is one physical line ≤ 120 characters carrying the heading anchor, the section's rule family, and the git stages that load it
    - Include the `anti-patterns` entry, which R14.1 requires by name
    - _Requirements: 3.1, 14.1_

  - [x] 4.4 Author the `glossary` section including the `Blocked_Outcome` term
    - Acceptance target: **14 lines** — heading 1 + blank 1 + 2 table header + 9 terms + trailing blank 1
    - `Blocked_Outcome` carries all four obligations once: name the blocking condition from the closed enumeration, name the causing artifact as a path/claim identity/revision, leave head + index + tracked working + untracked bytes unchanged, surface in the acting mechanism's own output
    - This task must complete before any section that says "resolve as a `Blocked_Outcome`" — that is nearly every rule section
    - _Requirements: 3.2, 4.15, 5.15, 6.13, 7.8, 8.12, 8.13, 9.11, 9.12_

  - [x] 4.5 Author the `load-budget` section
    - Acceptance target: **15 lines** — heading 1 + blank 1 + 2 table header + 10 stage rows + trailing blank 1
    - Exactly the 10 stage rows of the design's load budget table, each naming 1–4 sections; every one of the 16 sections is named by at least one row
    - _Requirements: 3.3, 3.12_

- [x] 5. Frontmatter and structural check families

  - [x] 5.1 Implement `scripts/lib/git-guidelines/fm-reader.mjs`
    - Bounded strict-subset YAML reader: block mappings of scalars, single-level nested mappings, block and flow sequences of scalars, plain and quoted scalars — nothing else
    - Reject out-of-subset input and duplicate keys with a typed `frontmatter-unparseable` finding naming the failure and its line; never guess
    - No third-party import
    - _Requirements: 1.10, 1.14_

  - [x] 5.2 Implement `scripts/lib/git-guidelines/frontmatter.mjs`
    - Partition every key into baseline / conformance / optional / unknown; baseline and conformance defects are fatal, optional and unknown defects are `minor`
    - Validate each declared domain: `title` ≤ 120 chars, `doc_type` from the Authoring_Authority enumeration, `version` 2–3 dot-separated non-negative integers, `date` `YYYY-MM-DD`, `lang` 2–5 chars, `owner` single scalar only, `local_rung`/`delivered_rung` separate single rung values, `lane: authoring`, `universal_scope: false`
    - Accumulate every defect with its expected domain rather than stopping at the first; complete within 5 s and emit exactly one verdict; leave document bytes unmodified
    - _Requirements: 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8, 1.11, 1.12, 1.13, 1.15, 11.6_

  - [ ]* 5.3 Write property test for the frontmatter round trip
    - **Property 1: Frontmatter round trip**
    - File `scripts/__pbt__/git-guidelines-frontmatter.pbt.test.mjs`; also run `js-yaml` as a dev-time model oracle and assert agreement with `fm-reader.mjs` on every generated mapping
    - Generator domain: mappings over the baseline, conformance, and optional key sets; scalars over an alphabet including every reserved punctuation character, the empty string, values at 1 / 80 / 120 characters, ASCII and non-ASCII text, and flow sequences of 0–8 scalars
    - **Validates: Requirements 1.9, 1.10**

  - [ ]* 5.4 Write property test for frontmatter key partition and severity routing
    - **Property 2: Frontmatter key partition and severity routing**
    - File `scripts/__pbt__/git-guidelines-frontmatter.pbt.test.mjs`
    - Generator domain: mappings mixing in-domain and out-of-domain values across all four partitions, arbitrary unknown key names, `owner` as scalar / sequence / mapping / delimited list, `local_rung` and `delivered_rung` as separate keys, merged key, and sequence
    - **Validates: Requirements 1.2, 1.3, 1.4, 1.5, 1.12, 1.13, 12.2**

  - [x] 5.5 Implement `scripts/lib/git-guidelines/structure.mjs`
    - Emit `section-unindexed`, `anchor-unresolved`, `stage-unmapped`, `section-unreferenced`, `cross-section-reference`, `rule-line-shape`
    - Enforce one rule per physical line ≤ 200 chars as either a directive bullet or a table row; enforce self-containment, permitting only references to the five owners, `module-index`, `boundary--ownership`, and `glossary`
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.11, 3.12, 3.13_

  - [x] 5.6 Implement `scripts/lib/git-guidelines/line-budget.mjs`
    - Measure total physical lines, each section span from its `##` heading through the line preceding the next `##`, the Module-Index-plus-boundary combined span, and each stage sum including the floor of Module Index + boundary
    - Fatal when `(total > 400 ∧ no frontmatter overrun justification) ∨ total > 440 ∨ indexBoundary > 40 ∨ max(stageSum) > 150`; report the measured count together with the exceeded limit
    - _Requirements: 3.5, 3.6, 3.7, 3.9, 3.10_

  - [ ]* 5.7 Write property test for line-budget measurement and threshold agreement
    - **Property 4: Line-budget measurement and threshold agreement**
    - File `scripts/__pbt__/git-guidelines-structure.pbt.test.mjs`; needs a document synthesizer emitting a known line count and section span vector
    - Generator domain: totals sampled densely across 380–460; Module-Index-plus-boundary spans across 35–45; section length vectors and stage→section maps producing per-stage sums across 140–160; with and without a well-formed frontmatter overrun entry
    - **Validates: Requirements 3.5, 3.6, 3.7, 3.9, 3.10, 14.1, 14.3, 14.4**

  - [ ]* 5.8 Write property test for bidirectional coverage between paired sets
    - **Property 5: Bidirectional coverage between paired sets**
    - File `scripts/__pbt__/git-guidelines-structure.pbt.test.mjs`; one parameterised test over all six pairs — sections↔index entries, sections↔stage rows, findings-table rows↔raisable types, artifact-bearing rules↔gate checks, owned families↔mantra clauses, dictionary metadata entries↔content token strings
    - Generator domain: for each pair, `(A, B)` with random symmetric difference in both directions, including empty sets and total mismatch
    - **Validates: Requirements 3.11, 3.12, 11.8, 12.6, 12.9, 13.1, 14.3, 14.7**

  - [ ]* 5.9 Write property test for section self-containment
    - **Property 9: Section self-containment**
    - File `scripts/__pbt__/git-guidelines-structure.pbt.test.mjs`
    - Generator domain: injected references of each kind — bare anchor link, prose naming another section's heading, "see the … section", a rule whose predicate names a table defined elsewhere — plus permitted references to the five owners and to Glossary terms
    - **Validates: Requirements 3.2, 3.13**

- [x] 6. Boundary and neutrality check families

  - [x] 6.1 Implement `scripts/lib/git-guidelines/boundary.mjs`
    - Require exactly 16 rows, each with one disposition from `owns` / `consumes` and, for `consumes`, one owner from the five; require the five Collaboration_Module rows C1–C5 to be present and separate; resolve every owner link on the filesystem
    - Emit `boundary-row-invalid`, `boundary-family-missing`, `boundary-owner-unknown`, each naming the offending rule family with its Rule_ID
    - _Requirements: 2.1, 2.2, 2.5, 2.9_

  - [x] 6.2 Implement `scripts/lib/git-guidelines/divergence.mjs`
    - Read the five owner documents for divergence comparison only, never to derive an obligation
    - Compare every `consumes`-marked term, value domain, ordering, and outcome against the named owner; emit `owner-divergence`, `terminology-drift`, `finding-type-redefinition` carrying Rule_ID, rule family, and named owner; leave every input unmodified
    - Enforce the exact Execution_Companion tokens `canonical`, `overlapping`, `disjoint-attributed`, `ambiguous` and the four roles Orchestrator / Implementer / Evaluator / Operator with no synonym, abbreviation, or alternative spelling
    - _Requirements: 2.4, 2.6, 12.3, 12.7_

  - [x] 6.3 Implement `scripts/lib/git-guidelines/neutrality.mjs`
    - Build a block index and accept a brand, host, repository, package-manager, or vendor-command token only inside a heading or block whose own text contains the words "reference implementation"; reject near-miss phrases
    - Emit `vendor-coupling` at the exact line and column of each occurrence with its Rule_ID and section anchor
    - _Requirements: 2.7, 2.8, 2.10, 10.13_

  - [ ]* 6.4 Write property test for reference-implementation containment
    - **Property 7: Reference-implementation containment**
    - File `scripts/__pbt__/git-guidelines-neutrality.pbt.test.mjs`
    - Generator domain: a fixed alphabet of brand and vendor-command tokens injected at random line and column positions — immediately before a block's opening fence, immediately after its closing fence, inside a nested fence, inside a table cell, inside the frontmatter, inside the anti-pattern and mantra sections — plus blocks whose heading text carries the near-miss phrases "reference implementations", "for reference", "implementation reference"
    - **Validates: Requirements 2.7, 2.8, 2.10, 10.13, 13.2**

  - [ ]* 6.5 Write property test for divergence from a named owner
    - **Property 8: No divergence from a named owner**
    - File `scripts/__pbt__/git-guidelines-neutrality.pbt.test.mjs`; the five real owner documents are the oracle, so an owner-side edit surfaces as a test failure
    - Generator domain: substitutions into consumed families — lane-class token variants (case, hyphenation, plural, abbreviation), role-name variants, claim-state vocabulary substitutions, reordered authority sequences, inverted outcomes — plus finding-type renames and severity changes for inherited types
    - **Validates: Requirements 2.4, 2.6, 12.3, 12.7**

- [x] 7. Checkpoint — spine and structural families
  - Ensure all tests pass, ask the user if questions arise.

- [x] 8. Document rule sections — lane topology through preservation

  - [x] 8.1 Author the `lane-topology--admission` section
    - Acceptance target: **38 lines** — heading 1 + blank 1 + 10-line identity projection table + 6-line lane class table + 17 directive bullets + trailing blank 1 + 2 spacing
    - Carry the eight-field Collaboration_Identity projection table, the four lane classes with their four observable derivation inputs, `ambiguous` as the fail-closed default treated as overlapping against every peer, the four R4.7 admission preconditions, additive-admission preservation, and the protected-upstream rule of unlimited pairwise-disjoint current authorities with exactly one current writer and only non-writing waiting successors per overlap
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.9, 4.12, 4.13, 4.14, 4.15_

  - [x] 8.2 Author the `coordination-artifacts` section
    - Acceptance target: **36 lines** — heading 1 + blank 1 + 16-line artifact/field table + 15 directive bullets + trailing blank 1 + 2 spacing
    - Carry `.coordination/` as sole location, UTF-8 JSON ≤ 64 KB, the `<semantic-scope>-<artifact-role>.json` naming pattern with roles `write-scope` / `request` / `claim` / `receipt`, the three artifact schemas, path normalization and the overlap rules, local-lease-proves-nothing, the online/offline 10 s probe definition, the offline permission and prohibition sets, the reconnect sequence, and the document-owned lease expiry ceiling resolved in task 1.1
    - State `claim.state` in the Collaboration_Module's six-value vocabulary and `admissionDecision` as derived, never stored
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 5.8, 5.9, 5.10, 5.11, 5.12, 5.13, 5.14, 5.15_

  - [x] 8.3 Author the `authoring--write-scope` section
    - Acceptance target: **20 lines** — heading 1 + blank 1 + 16 directive bullets + trailing blank 1 + 1 spacing
    - Carry one-commit-per-logical-unit, explicit-path or hunk staging with wildcard staging forbidden, scope containment for every changed path, the Change_Manifest update-before-commit obligation, the `out-of-scope-write` consequence, the amend permission and prohibition, and the dirty-canonical-base capture path
    - _Requirements: 4.11, 7.4, 7.5, 7.6, 7.8, 7.9, 7.12_

  - [x] 8.4 Author the `preservation-recovery--cleanup` section
    - Acceptance target: **30 lines** — heading 1 + blank 1 + 7-line artifact-location table + 19 directive bullets + trailing blank 1 + 1 spacing
    - Carry `.backups/` immutability and retention, the Bundle_Backup naming pattern with never-overwrite, the working-tree-rewrite capture obligation with `WIP: <semantic-scope>`, the Change_Manifest schema with sorted `paths`, `.agentic-manifests/` and `.recovery/` locations, the `<semantic-scope>-<yyyymmdd>T<hhmm>Z` capture pattern, the four-step capture with the completion marker written last, the park preconditions, per-occurrence Operator decisions, keep/port/drop classification, the restore round trip, and the two-variant Recovery_Handle
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 6.7, 6.8, 6.9, 6.10, 6.11, 6.12, 6.13, 6.14_

- [x] 9. Document rule sections — commit through promotion

  - [x] 9.1 Author the `commit--attribution` section
    - Acceptance target: **24 lines** — heading 1 + blank 1 + 8-line commit-type table + 6-line trailer table + 6 directive bullets + trailing blank 1 + 1 spacing
    - Carry the `<type>(<scope>): <summary>` subject at ≤ 72 chars, the closed commit-type set with exactly one selection condition each, the four attribution trailers each appearing exactly once in the final block with values equal to the accepted claim's, the what-and-why body obligation, and the `unattributed-agentic-commit` and `subject-format-invalid` outcomes
    - _Requirements: 7.1, 7.2, 7.3, 7.7, 7.10, 7.11_

  - [x] 9.2 Author the `verification-gates` section
    - Acceptance target: **24 lines** — heading 1 + blank 1 + 7-line recorded-result field table + 13 directive bullets + trailing blank 1 + 1 spacing
    - Carry the named check bound to the exact pushed revision, the recorded-result field set surfaced in the pushing agent's own output, verdict independence by (acting agent mechanism, Session ID) disjointness, pre-existing registered checks, the two-revision bug-fix binding, hook terminality with per-bypass Operator decisions, the three review-request bindings, the stated maximum run duration, and the `evidence-without-run` / `self-graded-verdict` / `check-not-terminal` outcomes
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 8.8, 8.9, 8.10, 8.11, 8.12, 8.13_

  - [x] 9.3 Author the `conflict--integration-order` section
    - Acceptance target: **24 lines** — heading 1 + blank 1 + 7-line dependency-order table + 13 directive bullets + trailing blank 1 + 1 spacing
    - Carry rebase on the re-read exact protected revision recorded in the integration request, resolution in the owning lane, serialization of overlapping scopes, "content mergeability proves no ownership safety", the five-class dependency order, preserve-retire-reseal on canonical drift, the repeated-approach root cause obligation, and the total comparator dependency class → lowest lease epoch → lexicographic Scope ID
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6, 9.7, 9.8, 9.9, 9.10, 9.11, 9.12, 9.13_

  - [x] 9.4 Author the `promotion-chain` section
    - Acceptance target: **28 lines** — heading 1 + blank 1 + 5-line chain/boundary table + 5-line rollback table + 8 directive bullets + 6-line reference-implementation block + trailing blank 1 + 1 spacing
    - Carry the three-stage chain with named boundaries, closed-by-default boundaries requiring five instruction elements, integration authority granting no deployment authority, the six-identity single-use candidate, the single-use deployment decision, the three separately recorded publication claims, per-stage rollback with separate code and state dispositions naming each irreversible one, authorization invalidation on drift, and `deploy-boundary-breach`
    - Every host, repository, and command name in this section sits inside the reference-implementation block
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.6, 10.7, 10.8, 10.9, 10.10, 10.11, 10.12, 10.13, 10.14, 10.15, 10.16_

- [x] 10. Document rule sections — findings, checklist, anti-patterns, mantra

  - [x] 10.1 Author the `findings--rule-identity` section
    - Acceptance target: **28 lines** — heading 1 + blank 1 + 13-line findings table (2 header + 11 rows) + 11 directive bullets + trailing blank 1 + 1 spacing
    - Carry the Rule_ID derivation rule, the artifact-bearing / advisory classification rule, and the 12-type registry with severities and `inherited`+owner or `document-local` markers; the three document-local types are `unattributed-agentic-commit`, `misplaced-conflict-resolution`, `unresolved-conflict-publish`; define no new severity
    - _Requirements: 12.1, 12.2, 12.3, 12.4, 12.5, 12.6, 12.7, 12.8, 12.9, 12.10_

  - [x] 10.2 Author the `validation-checklist` section
    - Acceptance target: **22 lines** — heading 1 + blank 1 + 12 lines across the five gate sub-blocks + 6-line reference-implementation block + trailing blank 1 + 1 spacing
    - Exactly five gates — pre-lane, per-commit, pre-push, pre-promotion, post-run — each with ≥ 1 check naming its Rule_ID and the observable evidence; every artifact-bearing rule appears in at least one gate
    - Name the Conformance_Checker invocation, its inputs, and its exit-status meaning inside the reference-implementation block
    - _Requirements: 13.1, 13.2_

  - [x] 10.3 Author the `anti-patterns` section
    - Acceptance target: **19 lines** — heading 1 + blank 1 + 8 pairs × 2 bullets = 16 + trailing blank 1 + 1 spacing; no blank separators, no nesting, no example blocks
    - The 8 pairs must include the seven R14.2 anti-patterns: authoring on the canonical branch, undeclared write scope, local-only lease as cross-device authority, self-graded lane verdicts, history rewrite without preservation, deploy on green integration, stale authorization reuse
    - Each corrective line names the owning Rule_ID or rule family, and where the pair corresponds to a registry finding type, names that type exactly; each line ≤ 120 chars
    - _Requirements: 14.1, 14.2, 14.4, 14.5, 14.6_

  - [x] 10.4 Author the `mantra` section
    - Acceptance target: **10 lines** — heading 1 + blank 1 + 7 clauses + trailing blank 1
    - Exactly 7 clauses, one per `owns` row O1–O7 of the boundary table, each on one line ≤ 120 chars
    - _Requirements: 14.3, 14.7_

- [x] 11. Line budget verification against the delivered file

  - [x] 11.1 Verify the budget arithmetic against the delivered `git-guidelines.md`
    - Add `scripts/__tests__/git-guidelines-budget.test.mjs` measuring the shipped file rather than a fixture
    - Assert total ≤ 400 with 392 allocated and 8 reserve; assert `module-index` + `boundary--ownership` ≤ 40 with 39 measured; assert every per-stage sum ≤ 150 under the strict reading with 127 worst (integration) and under the conservative Glossary-always-loaded reading with 141 worst
    - Assert the section-local caps: anti-pattern section ≤ 50 with 8 pairs of exactly 2 lines each ≤ 120 chars, mantra section ≤ 25 with clause count equal to the owned-family count, Module Index entries one line ≤ 120 chars, every rule line ≤ 200 chars
    - Assert no overrun justification key is present, since allocation closes under 400
    - _Requirements: 3.4, 3.5, 3.6, 3.7, 3.10, 14.1, 14.3, 14.4_

- [x] 12. Content, artifact-schema, findings-table, checklist, and anti-pattern families

  - [x] 12.1 Implement `scripts/lib/git-guidelines/findings-table.mjs`
    - Emit `finding-type-orphan`, `finding-type-unlisted`, `finding-row-invalid`, `document-local-marker-missing`; check both coverage directions between table rows and types raisable by some Rule_ID
    - Require every row to carry rule family, raising Rule_IDs, type name, severity, and an ownership marker; require each document-local type to carry the marker, an inherited-set severity, and ≥ 1 raising Rule_ID
    - _Requirements: 12.4, 12.5, 12.6, 12.8, 12.9_

  - [x] 12.2 Implement `scripts/lib/git-guidelines/content.mjs`
    - For each artifact-bearing document obligation of R4, R6–R10, decide satisfaction from section bodies and emit `unimplemented-guideline` per unmet rule
    - Implement the shared overlap relation once and reuse it for admission, commit scope, and publication; implement the total serialization comparator; implement the closed 27-condition blocking enumeration and the `Blocked_Outcome` record with pre/post four-digest comparison
    - Keep gate evaluation and finding recording as separate stages, so a recording failure cannot skip a gate
    - _Requirements: 4.6, 5.6, 7.5, 8.10, 9.5, 9.10, 9.13, 12.2, 13.1_

  - [x] 12.3 Implement `scripts/lib/git-guidelines/artifact-schema.mjs`
    - Validate the five schemas against the live trees: `agentic-declared-write-scope/v1`, `agentic-cloud-collaboration-request/v1`, `agentic-cloud-collaboration-result/v1`, `agentic-change-manifest/v1`, `agentic-legacy-dirty-lane-recovery/v1`
    - Emit `artifact-schema-invalid`, `artifact-name-mismatch`, `capture-incomplete`, `manifest-order-invalid`; enforce ascending byte order and no duplicates in `paths`, the file-name-to-`semanticScope` match, claim/request `leaseEpoch` and `declaredWriteScope` equality, and the completion marker as the only completeness proof
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 6.4, 6.5, 6.6_

  - [x] 12.4 Implement `scripts/lib/git-guidelines/checklist.mjs`
    - Emit `gate-partition-invalid`, `rule-uncovered-by-gate`, `checker-invocation-unstated`; require exactly the five gates, ≥ 1 check per gate, a Rule_ID and observable evidence per check, and full coverage of the artifact-bearing rule set
    - _Requirements: 13.1, 13.2_

  - [x] 12.5 Implement `scripts/lib/git-guidelines/antipattern.mjs`
    - Emit `antipattern-missing`, `antipattern-shape-invalid`, `mantra-clause-unmapped`, `mantra-family-uncovered`; check the seven mandated anti-patterns by name, the exactly-two-lines-and-≤-120-chars shape, finding-type name exactness against the registry, and mantra clause count against the `owns` row count in both directions
    - _Requirements: 14.1, 14.2, 14.3, 14.4, 14.5, 14.6, 14.7_

- [ ] 13. Lane, recovery, and gate property tests

  - [ ]* 13.1 Write property test for additive lane admission preservation
    - **Property 10: Additive lane admission preserves every pre-existing lane**
    - File `scripts/__pbt__/git-guidelines-lanes.pbt.test.mjs`; runs against real `git init` repositories under `os.tmpdir()`, never a mocked git, and never a network
    - Generator domain: `n` drawn 0–12 with explicit examples at 9 and 12 over pairwise-disjoint scopes; 12 is a finite test-resource bound only, never a policy cap; retain the existing dirt vectors, all 15 admission-precondition failure combinations, and interruption points
    - **Validates: Requirements 4.1, 4.7, 4.8, 4.15**

  - [ ]* 13.2 Write property test for the overlap relation algebra
    - **Property 11: Overlap relation algebra**
    - File `scripts/__pbt__/git-guidelines-lanes.pbt.test.mjs`
    - Generator domain: path sets containing `.`/`..` segments, trailing separators, repeated separators, ancestor/descendant chains of depth 1–6, unicode and percent-like path characters, `semantic:` entries, wildcard entries, absent and unparseable scope artifacts
    - **Validates: Requirements 4.6, 5.6, 7.5, 9.10**

  - [ ]* 13.3 Write property test for monotonic lease epoch with mutual exclusion
    - **Property 12: Monotonic lease epoch per scope with mutual exclusion**
    - File `scripts/__pbt__/git-guidelines-lanes.pbt.test.mjs`
    - Generator domain: epoch sequences per scope (increasing, repeating, decreasing, interleaved across scopes); expiry offsets straddling `now` and the ceiling; overlapping lane sets of size 2–8 whose contents are constructed to merge without conflict
    - **Validates: Requirements 4.14, 5.3, 9.3, 9.5**

  - [ ]* 13.4 Write property test for the recovery capture and restore round trip
    - **Property 13: Recovery capture and restore round trip**
    - File `scripts/__pbt__/git-guidelines-recovery.pbt.test.mjs`; asserts mode bits and binary content, not just text equality
    - Generator domain: trees of 0–200 files mixing text, binary, empty files, unicode and space-bearing filenames, nested directories, mode bits `420`/`493`, symlinks; capture interrupted at each of the four write steps; `k` corrupted paths injected before restore
    - **Validates: Requirements 4.10, 4.11, 6.6, 6.11, 6.12, 6.13, 6.14**

  - [ ]* 13.5 Write property test for serialization order determinism and totality
    - **Property 14: Serialization order determinism and totality**
    - File `scripts/__pbt__/git-guidelines-recovery.pbt.test.mjs`
    - Generator domain: path sets of size 0–500 shuffled `m` times; pending-request sets of size 0–8 with ties engineered at each of the three comparison keys, including full three-way ties and equal Scope IDs
    - **Validates: Requirements 6.4, 7.6, 9.13, 13.6**

  - [ ]* 13.6 Write property test for single-use authorization consumption
    - **Property 15: Single-use authorization is consumed exactly once**
    - File `scripts/__pbt__/git-guidelines-gates.pbt.test.mjs`
    - Generator domain: attempt sequences of length 1–10 per decision; decisions with each of the five required fields present or absent; candidate-identity and target-stage matches and mismatches; reseal sequences producing superseded candidate identities; irreversible-operation occurrence sequences under one decision
    - **Validates: Requirements 6.9, 10.5, 10.6, 10.10**

  - [ ]* 13.7 Write property test for closed-by-default boundaries
    - **Property 16: Boundaries are closed unless exactly authorized**
    - File `scripts/__pbt__/git-guidelines-gates.pbt.test.mjs`
    - Generator domain: instruction records over the power set of the five elements; candidate and target values matching and mismatched; evidence sets containing 0–5 passing check results with no decision; authoring-lane operations targeting the mirror and the delivery surface; the three publication claims sharing or not sharing run identities
    - **Validates: Requirements 10.3, 10.7, 10.8, 10.14, 10.15, 10.16**

  - [ ]* 13.8 Write property test for fail-closed behavior and byte preservation
    - **Property 18: Fail-closed — no passing verdict on degraded input, and every block preserves bytes**
    - File `scripts/__pbt__/git-guidelines-gates.pbt.test.mjs`; the largest domain in the suite
    - Generator domain: the cross product of the full input set × the four degraded conditions (absent, unparseable, stale, unreadable); plus the full enumeration of blocking Rule_IDs, each with a generated triggering state carrying non-empty head, index, working, and untracked bytes
    - Assert no report anywhere in the generated domain carries `verdict = conformant`
    - **Validates: Requirements 1.14, 4.9, 4.13, 5.13, 5.15, 6.13, 7.11, 8.12, 8.13, 9.8, 9.11, 9.12, 11.9, 11.11, 11.12, 11.13, 13.3, 13.8**

- [x] 14. Checkpoint — document complete, checker families complete
  - Ensure all tests pass, ask the user if questions arise.

- [x] 15. Registrations in the Docs_Control_Surface

  - [x] 15.1 Add the Docs_Index Document Map row
    - Edit `agentic-canvas-os/docs/README.md`: exactly one Document Map row with three non-empty cells — path `docs/documents/git-guidelines.md`, role "Git-layer companion to the execution set" at ≤ 120 chars, load condition "any git stage: session start through cleanup"
    - The referenced path lives in `huijoohwee.github.io` and resolves relative to the workspace root, not the dictionary's repository
    - _Requirements: 11.1, 11.13_

  - [x] 15.2 Register the `/git.guidelines` token in the Command_Dictionary
    - Edit `agentic-canvas-os/docs/DICTIONARY-COMMAND.md`: add the token with exactly one intent statement, exactly one completion signal, at least one required `@` binding, and at least one `#` semantic filter
    - List the document path exactly once in `source_docs` and the token string exactly once in `dictionary_entries`; assert the token string is unique across the dictionary
    - _Requirements: 11.2, 11.3_

  - [x] 15.3 Recompute and record the Command_Dictionary catalog digest and entry count
    - Edit `agentic-canvas-os/docs/DICTIONARY-COMMAND.md` metadata after 15.2, 15.4, and 15.5 have landed
    - Compute `catalogEntry = { token, kind, label, summary, sourcePath }`, canonical-encode with keys sorted ascending and no insignificant whitespace, sort entries by token across all three dictionaries, and record `sha256` of the concatenation plus the entry count
    - _Requirements: 11.7, 11.8, 11.11_

  - [x] 15.4 Register the `#git-collaboration` filter in the Semantic_Dictionary
    - Edit `agentic-canvas-os/docs/DICTIONARY-SEMANTIC.md`: add the filter selecting the git collaboration rule family and naming the document path as a source
    - _Requirements: 11.4, 11.13_

  - [x] 15.5 Register the `@git-guidelines` binding in the Binding_Dictionary
    - Edit `agentic-canvas-os/docs/DICTIONARY-BINDING.md`: add the binding resolving to the document path as a source
    - _Requirements: 11.5, 11.13_

  - [x] 15.6 Implement `scripts/lib/git-guidelines/registration.mjs`
    - Emit `registration-missing`, `registration-dangling`, `catalog-digest-mismatch`, `catalog-count-mismatch`, `token-divergence`; row and token gaps are `blocker`, filter and binding gaps are `minor` with exit zero
    - Compare frontmatter token / filters / bindings against registered counterparts byte-for-byte; compare metadata-named entries against content token strings in both directions; resolve every referenced path against the workspace root; leave every input unmodified
    - _Requirements: 11.1, 11.2, 11.3, 11.4, 11.5, 11.6, 11.7, 11.8, 11.9, 11.10, 11.11, 11.12, 11.13_

- [ ] 16. Registration drift tests against the real artifacts

  - [ ]* 16.1 Write the `docs-index-row` drift test
    - `scripts/__tests__/git-guidelines-drift-docs-index-row.test.mjs`: exactly one Document Map row for the document path with three non-empty cells
    - _Requirements: 11.1_

  - [ ]* 16.2 Write the `command-token` drift test
    - `scripts/__tests__/git-guidelines-drift-command-token.test.mjs`: `/git.guidelines` present once in `dictionary_entries` and once in the Commands table, with one intent, one completion signal, ≥ 1 binding, ≥ 1 filter, and a token string unique dictionary-wide
    - _Requirements: 11.2, 11.3_

  - [ ]* 16.3 Write the `source-docs` drift test
    - `scripts/__tests__/git-guidelines-drift-source-docs.test.mjs`: the document path appears exactly once in `source_docs`
    - _Requirements: 11.3_

  - [ ]* 16.4 Write the `frontmatter-parity` drift test
    - `scripts/__tests__/git-guidelines-drift-frontmatter-parity.test.mjs`: frontmatter token, filters, and bindings byte-equal to their registered counterparts, with divergence naming both compared artifacts
    - _Requirements: 11.6, 11.12_

  - [ ]* 16.5 Write the `catalog-digest` drift test
    - `scripts/__tests__/git-guidelines-drift-catalog-digest.test.mjs`: recomputed digest equals recorded; a synthetic entry rename changes it; a mismatch reports both recorded and recomputed values
    - _Requirements: 11.7, 11.11_

  - [ ]* 16.6 Write the `catalog-count` drift test
    - `scripts/__tests__/git-guidelines-drift-catalog-count.test.mjs`: recomputed count equals recorded; a synthetic add changes it; a rename does not, confirming why both quantities are required
    - _Requirements: 11.7, 11.11_

  - [ ]* 16.7 Write the `metadata-entry-parity` drift test
    - `scripts/__tests__/git-guidelines-drift-metadata-entry-parity.test.mjs`: metadata-named entries and content token strings agree in both directions, with the gap direction named
    - _Requirements: 11.8_

  - [ ]* 16.8 Write the `path-existence` drift test
    - `scripts/__tests__/git-guidelines-drift-path-existence.test.mjs`: every path referenced by any of the four registrations resolves on the filesystem, including the cross-repository document path
    - _Requirements: 11.13_

- [ ] 17. Fixture-based conformance tests

  - [ ]* 17.1 Build the conformant fixture and its assertion
    - `scripts/__tests__/fixtures/conformant/git-guidelines.md` plus the real registrations; assert exit 0, zero `blocker`, zero `major`, `minor` findings permitted, and all 12 registry type counts present with explicit zeros
    - _Requirements: 13.3, 13.9, 12.5_

  - [ ]* 17.2 Build the 14 frontmatter mutants and their assertions
    - `scripts/__tests__/fixtures/mutants/frontmatter-*.md` + `git-guidelines-mutants-frontmatter.test.mjs`; each asserts exit non-zero, the **expected Rule_ID** present in `unsatisfiedRuleIds`, and document bytes unchanged — not merely a non-zero exit
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.11, 1.12, 1.13, 1.14_

  - [ ]* 17.3 Build the 7 boundary mutants and their assertions
    - `fixtures/mutants/boundary-*.md` + `git-guidelines-mutants-boundary.test.mjs`; each asserts the expected Rule_ID **and** the named rule family
    - _Requirements: 2.1, 2.2, 2.5, 2.6, 2.9_

  - [ ]* 17.4 Build the 5 neutrality mutants and their assertions
    - `fixtures/mutants/neutrality-*.md` + `git-guidelines-mutants-neutrality.test.mjs`; each asserts `vendor-coupling` at the expected line **and** column
    - _Requirements: 2.7, 2.8, 2.10, 10.13_

  - [ ]* 17.5 Build the 9 structure mutants and their assertions
    - `fixtures/mutants/structure-*.md` + `git-guidelines-mutants-structure.test.mjs`; each asserts the expected coverage-direction finding with its Rule_ID
    - _Requirements: 3.1, 3.2, 3.11, 3.12, 3.13, 3.4_

  - [ ]* 17.6 Build the 6 line-budget mutants and their assertions
    - `fixtures/mutants/line-budget-*.md` + `git-guidelines-mutants-line-budget.test.mjs`; each asserts the measured count **and** the exceeded limit, covering the 400-without-justification, 400-with-justification, 440, 40, and 150 thresholds
    - _Requirements: 3.5, 3.6, 3.7, 3.9, 3.10_

  - [ ]* 17.7 Build the 6 findings-table mutants and their assertions
    - `fixtures/mutants/findings-table-*.md` + `git-guidelines-mutants-findings-table.test.mjs`; each asserts the coverage direction plus the offending finding type or Rule_ID
    - _Requirements: 12.4, 12.6, 12.8, 12.9_

  - [ ]* 17.8 Build the 4 checklist mutants and their assertions
    - `fixtures/mutants/checklist-*.md` + `git-guidelines-mutants-checklist.test.mjs`; each asserts the expected uncovered artifact-bearing Rule_ID or the invalid gate partition
    - _Requirements: 13.1, 13.2_

  - [ ]* 17.9 Build the 8 anti-pattern mutants and their assertions
    - `fixtures/mutants/antipattern-*.md` + `git-guidelines-mutants-antipattern.test.mjs`; each asserts the absent anti-pattern by name or the offending pair's Rule_ID
    - _Requirements: 14.1, 14.2, 14.4, 14.5, 14.6_

  - [ ]* 17.10 Build the 3 mantra mutants and their assertions
    - `fixtures/mutants/mantra-*.md` + `git-guidelines-mutants-mantra.test.mjs`; each asserts the uncovered owned family **and** the unmapped clause
    - _Requirements: 14.3, 14.7_

- [ ] 18. Integration and smoke tests

  - [ ]* 18.1 Write the unreachable-remote integration test
    - `scripts/__tests__/git-guidelines-integration-remote.test.mjs`: a stub remote that never answers yields exit 3, names the unreachable remote and every check it blocked, and reports no passing verdict
    - _Requirements: 13.10_

  - [ ]* 18.2 Write the 60-second verdict bound integration test
    - `scripts/__tests__/git-guidelines-integration-verdict-bound.test.mjs`: the conformant fixture plus the real registration artifacts returns a verdict within 60 s of wall clock
    - _Requirements: 13.11_

  - [ ]* 18.3 Write the 5-second frontmatter bound integration test
    - `scripts/__tests__/git-guidelines-integration-frontmatter-bound.test.mjs`: the largest permitted document (440 lines) yields exactly one verdict object within 5 s
    - _Requirements: 1.15_

  - [ ]* 18.4 Write the 10-second online probe integration test
    - `scripts/__tests__/git-guidelines-integration-probe.test.mjs`: three cases — success at 9 s reads online, failure reads offline, an 11 s timeout reads offline
    - _Requirements: 5.14_

  - [ ]* 18.5 Write the dependency-surface smoke test
    - `scripts/__tests__/git-guidelines-smoke-deps.test.mjs`: walk the checker's module graph from `check-git-guidelines.mjs` and assert zero non-`node:` imports, zero model invocations, and no socket opened to a host other than a configured git remote
    - _Requirements: 5.8, 13.7_

  - [ ]* 18.6 Write the live artifact schema smoke test
    - `scripts/__tests__/git-guidelines-smoke-live-artifacts.test.mjs`: every real artifact under `.coordination/`, `.agentic-manifests/`, and `.recovery/` validates against its declared schema
    - Expected to fail on first run against the two known divergences until task 19 lands
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 6.4, 6.5, 6.6_

- [x] 19. Repair the two divergent live coordination artifacts per resolved Operator Decision 3

  - [x] 19.1 Reorder `paths` in `.coordination/dev-source-resolver-write-scope.json` into ascending byte order
    - `scripts/__tests__/worktree-policy.test.mjs` must precede `scripts/worktree-policy.mjs` (`_` is 0x5F, `w` is 0x77)
    - Decision 3 selects artifact repair: perform the byte-order correction and do not relax R5.2
    - _Requirements: 5.2_

  - [x] 19.2 Add the `schema` key to `.coordination/dev-source-resolver-cloud-request.json`
    - Add `"schema": "agentic-cloud-collaboration-request/v1"`; a declared schema is what makes R5.13's unparseable-against-schema condition decidable
    - Decision 3 selects artifact repair: add the declared schema and do not relax R5.13; this O2 file-layout change touches no claim identity
    - _Requirements: 5.3, 5.13_

- [x] 20. Wire the checker into the existing test lane

  - [x] 20.1 Add the checker and its suites to `huijoohwee.github.io/package.json`
    - Add `"git-guidelines:check": "node scripts/check-git-guidelines.mjs"` and `"git-guidelines:test": "node --test scripts/__tests__/*.test.mjs scripts/__pbt__/*.test.mjs"`; Node 22 treats bare directory operands as modules, so the two broad globs are the executable directory-wide equivalent
    - Extend `"test"` to run alongside the existing lane: `npm run agentic-sdlc:policy:check && npm run agenticrag:guidelines-map:check && npm run git-guidelines:check && npm run git-guidelines:test`
    - Confirm `fast-check` and `js-yaml` remain in `devDependencies` only and that `dependencies` stays absent
    - _Requirements: 13.2, 13.3, 13.7_

- [x] 21. Final checkpoint
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP; the document and its registrations remain deliverable without them, but the conformance guarantee weakens.
- Task 19 is the only task that edits pre-existing coordination artifacts; resolved Operator Decision 3 authorizes the two exact repairs and forbids schema relaxation.
- Task 1.1 remains first because it records the resolved provenance consumed by Tasks 8.1, 8.2, 19, and 20; it is no longer a question gate.
- The 17 document-authoring sub-tasks all write `git-guidelines.md`, so they occupy successive waves. Each is paired with checker work in a different file to keep the waves productive.
- Every property sub-task names its property number, its generator domain, and the requirements it discharges, per the design's 18-property set.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2"] },
    { "id": 1, "tasks": ["2.1", "2.2", "2.3", "4.1"] },
    { "id": 2, "tasks": ["2.4", "2.5", "4.2", "5.1"] },
    { "id": 3, "tasks": ["2.6", "4.3", "5.2", "6.1", "12.3"] },
    { "id": 4, "tasks": ["2.7", "4.4", "5.3", "5.5", "6.2"] },
    { "id": 5, "tasks": ["4.5", "5.4", "5.6", "6.3", "12.1"] },
    { "id": 6, "tasks": ["5.7", "6.4", "8.1", "12.2", "12.4"] },
    { "id": 7, "tasks": ["5.8", "6.5", "8.2", "12.5"] },
    { "id": 8, "tasks": ["5.9", "8.3"] },
    { "id": 9, "tasks": ["8.4", "13.1"] },
    { "id": 10, "tasks": ["9.1", "13.2"] },
    { "id": 11, "tasks": ["9.2", "13.3"] },
    { "id": 12, "tasks": ["9.3", "13.4"] },
    { "id": 13, "tasks": ["9.4", "13.5"] },
    { "id": 14, "tasks": ["10.1", "13.6"] },
    { "id": 15, "tasks": ["10.2", "13.7"] },
    { "id": 16, "tasks": ["10.3", "13.8"] },
    { "id": 17, "tasks": ["10.4"] },
    { "id": 18, "tasks": ["11.1", "15.1", "15.2", "15.4", "15.5", "17.1"] },
    { "id": 19, "tasks": ["15.3", "15.6", "17.2", "17.3", "17.4", "17.5", "17.6", "17.7", "17.8", "17.9", "17.10"] },
    { "id": 20, "tasks": ["16.1", "16.2", "16.3", "16.4", "16.5", "16.6", "16.7", "16.8", "18.1", "18.2", "18.3", "18.4", "18.5", "18.6", "19.1", "19.2"] },
    { "id": 21, "tasks": ["20.1"] }
  ]
}
```
