# Requirements Document

## Introduction

This feature is the authoring of one guideline document — `huijoohwee.github.io/docs/documents/git-guidelines.md` — that serves as the git-layer companion to `huijoohwee.github.io/guidelines/agentic-sdlc-guidelines.md`.

The document translates the execution-domain rules of the Agentic SDLC set (collaboration identity, scoped lane admission, cloud-authoritative claims, dependency-ordered integration, release lifecycle) into the concrete git operations a solo operator runs across multiple devices, sessions, and agents against a zero-infrastructure toolchain: git plus the filesystem, no coordination daemon, no database, no paid service.

The document is a **projection**, not a new authority. Claim identity, authority order, write-scope comparison, fence meaning, and handoff semantics stay owned by the canonical collaboration modules; the document names them and shows the git-level operations that satisfy them. Commit/push/deploy mechanics already owned by `commit-push-deploy-guidelines.md` are referenced, not restated.

The document must be loadable by section for a single stage of work, parseable by an agent, discoverable from the Agentic Canvas OS docs control surface, and invocable through the `/`, `@`, and `#` dictionaries plus MCP capability metadata.

## Glossary

- **Git_Guidelines_Document**: the authored artifact at `huijoohwee.github.io/docs/documents/git-guidelines.md`; the system under specification.
- **Execution_Companion**: `huijoohwee.github.io/guidelines/agentic-sdlc-guidelines.md`; owner of execution-domain rules, task model, and collaboration identity.
- **Authoring_Authority**: `huijoohwee.github.io/guidelines/prd-tad-adr-guidelines.md`; owner of frontmatter contract, Rule ID derivation, finding recording contract, and readiness ladder.
- **Collaboration_Module**: `huijoohwee.github.io/guidelines/agentic-sdlc-cloud-collaboration.md`; owner of the protected remote ledger, claim fields, and offline boundary.
- **Lane_Admission_Module**: `huijoohwee.github.io/guidelines/agentic-sdlc-scoped-lane-admission.md`; owner of additive lane admission and preservation proof.
- **Delivery_Guidelines**: `huijoohwee.github.io/guidelines/commit-push-deploy-guidelines.md`; existing owner of commit, push, and deploy command sequences.
- **Canonical_Lane**: the one checkout per repository that tracks the exact protected canonical revision and owns shared refresh.
- **Task_Lane**: an isolated branch and checkout owning exactly one declared write scope, one lease epoch, and one fence revision.
- **Collaboration_Identity**: the tuple Actor ID + Device ID + Session ID + Worktree ID + Branch ID + Scope ID + Lease Epoch + Fence Revision.
- **Coordination_Artifact**: a filesystem JSON record projecting collaboration state — declared write scope, claim request, accepted claim result, or receipt — stored under `.coordination/`.
- **Change_Manifest**: a JSON record of schema `agentic-change-manifest/v1` naming a lane's branch, base revision, and enumerated paths.
- **Bundle_Backup**: a `git bundle` file under `.backups/` preserving a lane or stash as recoverable immutable bytes.
- **Recovery_Capture**: a directory under `.recovery/` holding a manifest, a tracked patch, retained files, and a completion marker for preserved work.
- **Recovery_Handle**: either the exact directory name of a Recovery_Capture under `.recovery/` whose completion marker is present, or the exact filename of a verified Bundle_Backup under `.backups/`.
- **Document_Local_Finding**: a finding type name owned by neither the Execution_Companion nor the Authoring_Authority, introduced by the Git_Guidelines_Document with a `document-local` ownership marker.
- **Dev_Repository**: the development source repository (`knowgrph`).
- **Prod_Mirror**: the production mirror repository path (`huijoohwee/content/knowgrph`).
- **Delivery_Surface**: the public deployed surface (`airvio.co` and `airvio.co/knowgrph`).
- **Promotion_Chain**: the ordered movement Dev_Repository → Prod_Mirror → Delivery_Surface.
- **Docs_Control_Surface**: the docs folder of `agentic-canvas-os`.
- **Docs_Index**: `agentic-canvas-os/docs/README.md`, including its Document Map table.
- **Command_Dictionary**: `agentic-canvas-os/docs/DICTIONARY-COMMAND.md`, owner of `/` tokens.
- **Semantic_Dictionary**: `agentic-canvas-os/docs/DICTIONARY-SEMANTIC.md`, owner of `#` filters.
- **Binding_Dictionary**: `agentic-canvas-os/docs/DICTIONARY-BINDING.md`, owner of `@` bindings.
- **Conformance_Checker**: the deterministic check that reads the Git_Guidelines_Document and its registrations and exits zero only when every stated obligation is satisfied.
- **Reference_Implementation_Block**: a heading or block whose own text contains the words "reference implementation", the only place a brand, repository, host, or command name may appear.
- **Rule_ID**: the identifier derived per Authoring_Authority as section anchor + `#` + ordinal of the rule within that section.

## Requirements

### Requirement 1: Frontmatter and Authoring Conformance

**User Story:** As a solo operator, I want the git guidelines to carry the same machine-readable identity as every other canonical document, so that tooling reads its status without inferring anything from its path.

#### Acceptance Criteria

1. THE Git_Guidelines_Document SHALL begin with a YAML frontmatter block as the first block in the file, with zero bytes of content preceding its opening delimiter.
2. THE Git_Guidelines_Document SHALL declare the baseline keys `title`, `doc_type`, `version`, `date`, and `lang`, each carrying a non-empty scalar, with `title` at or below 120 characters, `doc_type` drawn from the document type enumeration of the Authoring_Authority, `version` carrying two or three dot-separated non-negative integers, `date` carrying a calendar date in `YYYY-MM-DD` form, and `lang` carrying one language tag of 2 to 5 characters.
3. THE Git_Guidelines_Document SHALL declare the conformance keys `owner`, `local_rung`, `delivered_rung`, `lane`, and `universal_scope`, each carrying a non-empty scalar within the domain stated for that key by criteria 4 through 7.
4. THE Git_Guidelines_Document SHALL declare `owner` as exactly one scalar of 1 to 80 characters, and SHALL NOT declare it as a sequence, a mapping, or a delimiter-separated list of two or more values.
5. THE Git_Guidelines_Document SHALL declare `local_rung` and `delivered_rung` as two separate keys, each carrying exactly one rung value drawn from the readiness ladder enumerated by the Authoring_Authority, and SHALL NOT merge them into a single key or a sequence.
6. THE Git_Guidelines_Document SHALL declare `lane` with the exact lowercase value `authoring` and no other value.
7. THE Git_Guidelines_Document SHALL declare `universal_scope` with the boolean value `false`, because the document names concrete repositories and hosts.
8. THE Git_Guidelines_Document SHALL declare a `companion_of` key carrying exactly one scalar that identifies the Execution_Companion by the same identifier the boundary section uses for it.
9. WHERE a frontmatter scalar contains reserved punctuation — a colon followed by a space, a `#` preceded by a space, or a leading `-`, `?`, `:`, `,`, `[`, `]`, `{`, `}`, `&`, `*`, `!`, `|`, `>`, `%`, `@`, backtick, single quote, or double quote — THE Git_Guidelines_Document SHALL quote that scalar.
10. WHEN the frontmatter is parsed by a strict YAML parser, serialized, and parsed again, THE Conformance_Checker SHALL confirm that the two parsed mappings carry identical key sets at every nesting level and identical scalar values for every key (round-trip property).
11. IF a baseline or conformance frontmatter key is absent or carries a value outside the domain stated in criteria 2 through 7, THEN THE Conformance_Checker SHALL exit non-zero and SHALL name every such key together with its expected domain rather than stopping at the first one.
12. IF an optional frontmatter key carries a value outside its declared domain, THEN THE Conformance_Checker SHALL name that key and its expected domain as an advisory finding, SHALL exit zero when every baseline and conformance key is valid, and SHALL leave the bytes of the Git_Guidelines_Document unchanged.
13. THE Conformance_Checker SHALL treat `companion_of`, the invocation token keys required by Requirement 11, and the overrun justification key required by Requirement 3 as the optional frontmatter key set, and SHALL report any key outside the baseline, conformance, and optional key sets as an advisory finding.
14. IF the file carries no frontmatter block, the block is not the first block, or a strict YAML parser rejects the block — including rejection on a duplicate key — THEN THE Conformance_Checker SHALL exit non-zero, name the failure and the line at which it was detected, and evaluate no further criterion of this requirement.
15. WHEN the Conformance_Checker validates the frontmatter, THE Conformance_Checker SHALL emit exactly one verdict and complete within 5 seconds of invocation.

### Requirement 2: Companion Boundary and Non-Duplication

**User Story:** As a solo operator, I want the git guidelines to project existing authority rather than compete with it, so that one rule never has two conflicting homes.

#### Acceptance Criteria

1. THE Git_Guidelines_Document SHALL contain exactly one boundary section, addressable by its own `##` heading anchor, naming the Execution_Companion, the Authoring_Authority, the Collaboration_Module, the Lane_Admission_Module, and the Delivery_Guidelines as the five owners of the rules it projects, each named with a resolvable link to that owner's document.
2. THE Git_Guidelines_Document SHALL contain in the boundary section a table carrying exactly one row per rule family it covers, each row stating the rule family name, exactly one disposition value drawn from `owns` and `consumes`, and, for a `consumes` row, exactly one owner drawn from the five owners named in criterion 1.
3. THE Git_Guidelines_Document SHALL reference the Delivery_Guidelines by resolvable link for commit, push, and deploy command sequences already owned there.
4. THE Git_Guidelines_Document SHALL name lane classes, task states, roles, and receipts using only the exact terminology tokens of the Execution_Companion — including the four lane classes `canonical`, `overlapping`, `disjoint-attributed`, `ambiguous` and the four roles Orchestrator, Implementer, Evaluator, Operator — and no synonym, abbreviation, or alternative spelling of those tokens.
5. THE Git_Guidelines_Document SHALL record claim identity, authority order, write-scope comparison, fence meaning, and handoff semantics as five `consumes` rows naming the Collaboration_Module as owner, and SHALL state no decision rule for those five that the Collaboration_Module does not state.
6. IF a rule family marked `consumes` states a term, value domain, ordering, or outcome that differs from the named owner's statement of that rule, THEN THE Conformance_Checker SHALL exit non-zero and report every divergent rule with its Rule_ID, its rule family, and the named owner, leaving the Git_Guidelines_Document unmodified.
7. WHERE the Git_Guidelines_Document names a brand, host, repository, package manager, or vendor command, THE Git_Guidelines_Document SHALL place that name inside a Reference_Implementation_Block.
8. THE Git_Guidelines_Document SHALL restate no commit, push, or deploy command sequence owned by the Delivery_Guidelines outside a Reference_Implementation_Block.
9. IF the boundary table omits a row for a rule family the Git_Guidelines_Document covers, carries a disposition value outside `owns` and `consumes`, or names an owner absent from the five owners of criterion 1, THEN THE Conformance_Checker SHALL exit non-zero and name the offending rule family with its Rule_ID.
10. IF a brand, host, repository, package manager, or vendor command name appears outside a Reference_Implementation_Block, THEN THE Conformance_Checker SHALL exit non-zero and report each occurrence with its Rule_ID and section anchor.

### Requirement 3: Section Addressability and Load Budget

**User Story:** As an agent operating under a token budget, I want to load only the git rules for my current stage, so that governing the work costs a fraction of doing the work.

#### Acceptance Criteria

1. THE Git_Guidelines_Document SHALL contain a Module Index that lists every `##` section other than the Module Index itself, giving each section exactly one entry on one physical line carrying that section's heading anchor and a role description of at most 120 characters naming the section's rule family and the git stages that load it.
2. THE Git_Guidelines_Document SHALL make each `##` section self-contained, meaning every rule in the section is applicable by a reader who has loaded only that section, the Module Index, the boundary section, and the Glossary, with no reference that requires loading another `##` section of the same document; references to the Execution_Companion, the Authoring_Authority, the Collaboration_Module, the Lane_Admission_Module, and the Delivery_Guidelines remain permitted.
3. THE Git_Guidelines_Document SHALL contain a load budget table with exactly one row per git stage — session start, lane admission, authoring, commit, push, review, integration, promotion, recovery, cleanup — mapping each stage to at least one and at most four `##` sections required for that stage.
4. THE Git_Guidelines_Document SHALL express every rule as either one directive bullet or one table row, each occupying exactly one physical line of at most 200 characters.
5. THE Git_Guidelines_Document SHALL keep total length at or below 400 physical lines, counting every line of the file including the frontmatter block, blank lines, and table rows.
6. THE Git_Guidelines_Document SHALL keep the Module Index and the boundary section at or below 40 physical lines combined, counted for each section from its `##` heading line through the line preceding the next `##` heading, because both load on every stage.
7. THE Git_Guidelines_Document SHALL keep the combined physical line count of the sections required for any single stage — including the Module Index and the boundary section — at or below 150 lines.
8. WHEN a rule is added, THE Git_Guidelines_Document SHALL place it in the section owning the earliest git stage at which the rule applies and SHALL list that section in the load budget row of every other stage at which the rule applies, so stage-scoped loading remains sufficient without restating the rule.
9. WHERE correct stage placement of a rule raises total length above 400 lines, THE Git_Guidelines_Document SHALL keep the placement, record in its frontmatter an overrun entry stating the measured line count and the justification for the overrun, and keep total length at or below 440 physical lines.
10. IF total length exceeds 400 physical lines with no recorded overrun justification in the frontmatter, or exceeds 440 physical lines with or without one, THEN THE Conformance_Checker SHALL exit non-zero and report the measured line count together with the exceeded limit.
11. IF a `##` section has no Module Index entry, or the Module Index names a heading anchor that resolves to no `##` section of the Git_Guidelines_Document, THEN THE Conformance_Checker SHALL exit non-zero and name the unindexed section or the unresolved anchor.
12. IF a stage row of the load budget table names no section, or a `##` section is named by no stage row, THEN THE Conformance_Checker SHALL exit non-zero and name the unmapped stage or the unreferenced section anchor.
13. IF a rule in a `##` section cannot be applied without loading another `##` section of the same document, THEN THE Conformance_Checker SHALL exit non-zero and report the referring Rule_ID with the referenced section anchor.

### Requirement 4: Multi-Device Concurrent Lane Topology

**User Story:** As one human working from several machines, sessions, and agents at once, I want a git lane topology that keeps concurrent writers from colliding, so that parallel work never costs me committed bytes.

#### Acceptance Criteria

1. THE Git_Guidelines_Document SHALL define exactly one Canonical_Lane per repository plus zero or more Task_Lanes, SHALL define an active writer as one Actor ID + Device ID + Session ID triple holding a live lease on that lane, SHALL permit at most one active writer per lane at any instant, and SHALL state that there is no global policy cap on concurrent current authorities whose normalized declared write sets are pairwise disjoint.
2. THE Git_Guidelines_Document SHALL state the branch naming pattern for a Task_Lane as `agent/<device-id>/<semantic-scope>`, with `<device-id>` and `<semantic-scope>` each 1 to 64 characters drawn from lowercase letters, digits, and hyphens, and the complete branch name at or below 200 characters.
3. THE Git_Guidelines_Document SHALL state that the Canonical_Lane's head revision equals the exact protected canonical revision, that the Canonical_Lane carries zero staged changes, zero unstaged tracked changes, and zero untracked files, and that it performs no source authoring.
4. THE Git_Guidelines_Document SHALL enumerate the eight fields of Collaboration_Identity as Actor ID, Device ID, Session ID, Worktree ID, Branch ID, Scope ID, Lease Epoch, and Fence Revision, and SHALL state for each field which git object or Coordination_Artifact projects it.
5. THE Git_Guidelines_Document SHALL define the four lane classes `canonical`, `overlapping`, `disjoint-attributed`, and `ambiguous`, and SHALL state for each class the observable state — branch registration, head revision, declared write scope artifact, and accepted claim result — from which that class is derived.
6. THE Git_Guidelines_Document SHALL state that a lane whose future write scope is undeclared or unparseable is classified `ambiguous`, is treated as overlapping in every write-scope comparison, and therefore resolves as blocked for admission.
7. WHEN a new Task_Lane is requested, THE Git_Guidelines_Document SHALL require all four of the following to be verified before the first write: a canonical base whose head revision equals the exact protected canonical revision with zero staged changes, zero unstaged tracked changes, and zero untracked files; an accepted claim result artifact covering the requested scope; a local lease carrying a stated expiry at or below 24 hours from issue; and a declared write scope proven disjoint from every live claim.
8. WHEN a Task_Lane is admitted, THE Git_Guidelines_Document SHALL require the admitting operation to record each pre-existing lane's head revision, branch registration, index entries, tracked working bytes, and untracked bytes before the operation and to prove each recorded value byte-for-byte equal after the operation.
9. IF a live claim's declared write scope overlaps the requested scope, THEN THE Git_Guidelines_Document SHALL retain the request as a non-writing waiting successor, require authoring admission to resolve as blocked, name every overlapping live claim with its owning Device ID, leave the requesting lane's bytes unchanged, and keep the successor blocked until an authenticated retirement, handoff, or reclaim is recorded in the authoritative claim result or receipt.
10. IF source authoring has already begun on the protected canonical branch, THEN THE Git_Guidelines_Document SHALL require the exact tracked and untracked bytes to be preserved, proven byte-for-byte equal after being moved into an owned Task_Lane, and the protected canonical branch returned to zero uncommitted and zero untracked changes, all before the next commit.
11. WHERE the canonical base carries uncommitted or untracked bytes, THE Git_Guidelines_Document SHALL permit Task_Lane admission once those bytes are captured as a Bundle_Backup or Recovery_Capture, moved into the requesting Task_Lane and proven byte-for-byte equal to the captured bytes, and the exact base revision is recorded, and SHALL require every other admission condition of criterion 7 to hold unchanged.
12. WHERE work originates from a browser or mobile surface, THE Git_Guidelines_Document SHALL state that the surface projects an accepted claim identified by its claim digest and lease epoch, creates no independent authority, and resolves as blocked when it cannot present that claim digest and lease epoch.
13. IF any of the eight Collaboration_Identity fields cannot be projected from a git object or a Coordination_Artifact, THEN THE Git_Guidelines_Document SHALL require the requested lane operation to resolve as blocked, to name each unprojectable field, and to leave all existing committed and uncommitted bytes unchanged.
14. IF a lane's local lease is past its stated expiry or its lease epoch is lower than the lease epoch of the current accepted claim result, THEN THE Git_Guidelines_Document SHALL require every further write in that lane to resolve as blocked until a fresh accepted claim and lease are recorded, and SHALL require the lane's committed and uncommitted bytes to be retained unchanged while blocked.
15. IF any admission condition of criterion 7 fails, THEN THE Git_Guidelines_Document SHALL require the admission to leave the repository in its pre-request state with no branch created, no ref updated, and no working or untracked bytes changed, and to indicate which condition failed.

### Requirement 5: Coordination Artifacts on Git and Filesystem Only

**User Story:** As a zero-infrastructure operator, I want coordination state to live in files and git refs, so that concurrency safety costs nothing to run and works offline.

#### Acceptance Criteria

1. THE Git_Guidelines_Document SHALL specify the `.coordination/` directory at the root of the target repository's working tree as the sole location of Coordination_Artifacts, and SHALL specify each Coordination_Artifact as one UTF-8 encoded JSON file at or below 64 KB.
2. THE Git_Guidelines_Document SHALL specify the declared write scope artifact with schema `agentic-declared-write-scope/v1` carrying exactly one semantic scope of 3 to 64 characters and one path list of 1 to 4096 repository-relative paths, each path at or below 512 characters, sorted in ascending byte order, with no duplicate entry.
3. THE Git_Guidelines_Document SHALL specify the claim request artifact carrying target repository, work item, canonical base revision, lane revision, declared write scope, lease epoch as a non-negative integer that increases monotonically per scope, expiry as an absolute UTC instant no more than 24 hours after issuance, device, session, and actor, with every listed field required and non-empty.
4. THE Git_Guidelines_Document SHALL specify the accepted claim result artifact carrying claim state drawn from the enumerated set `accepted`, `blocked`, `expired`, `released`, plus claim digest, write-set digest, lease epoch, fence revision, ledger revision, and receipt digest, with every listed field required and non-empty, and SHALL require the recorded lease epoch and declared write scope to equal those of the claim request it answers.
5. THE Git_Guidelines_Document SHALL specify the artifact file naming pattern `<semantic-scope>-<artifact-role>.json`, where `<semantic-scope>` matches the semantic scope recorded inside the artifact using lowercase letters, digits, and hyphens only, and `<artifact-role>` is drawn from the enumerated set `write-scope`, `request`, `claim`, `receipt`.
6. THE Git_Guidelines_Document SHALL state that scope comparison is performed on repository-relative paths normalized by resolving `.` and `..` segments and removing trailing separators, and SHALL state that path equality, ancestor and descendant relationships, shared semantic artifacts, wildcard scopes, and any comparison that cannot be decided from the recorded paths all count as overlap.
7. THE Git_Guidelines_Document SHALL state that a lease recorded only on the local filesystem proves no cross-device authority, and SHALL require any operation needing cross-device authority to resolve as blocked until an accepted claim for the same scope and lease epoch is readable from the protected remote.
8. THE Git_Guidelines_Document SHALL state that coordination requires no service beyond git remotes and the local filesystem, and SHALL state that every Coordination_Artifact is readable and writable using git and filesystem access alone, with no daemon, database, or paid dependency.
9. WHILE a device is offline, THE Git_Guidelines_Document SHALL permit local commits in an already owned Task_Lane whose accepted claim has not passed its recorded expiry.
10. WHILE a device is offline and no accepted claim covers the requested scope, THE Git_Guidelines_Document SHALL require new local branch authoring, shared claim acquisition, review dispatch, handoff, and integration to resolve as blocked.
11. WHILE a device is online, THE Git_Guidelines_Document SHALL permit local branch authoring in a Task_Lane whose declared write scope is covered by an accepted claim that has not passed its recorded expiry and whose fence revision equals the current accepted fence.
12. WHEN a device reconnects, THE Git_Guidelines_Document SHALL require fetching the protected heads, confirming the recorded canonical base revision and lane revision still resolve to the same revisions recorded in the lane's Coordination_Artifacts, and obtaining a claim whose state is `accepted`, whose expiry is in the future, and whose fence revision equals the current accepted fence, before publishing lane bytes.
13. IF a Coordination_Artifact is absent, fails to parse as JSON against its declared schema, has passed its recorded expiry, or records a fence revision differing from the current accepted fence, THEN THE Git_Guidelines_Document SHALL require the operation to resolve as blocked.
14. THE Git_Guidelines_Document SHALL define a device as online when a reachability probe against the configured git remote succeeds within 10 seconds, and as offline when that probe fails or does not complete within 10 seconds.
15. IF an operation resolves as blocked under this requirement, THEN THE Git_Guidelines_Document SHALL require the blocking condition and the identity of the Coordination_Artifact or claim that caused it to be surfaced in the acting mechanism's own output, and SHALL require the lane's head, index, working bytes, and untracked bytes to remain unchanged.

### Requirement 6: Preservation, Backup, and Recovery

**User Story:** As a solo operator with work spread across machines, I want every risky git operation to leave recoverable bytes behind, so that no lane is ever the only copy of its work.

#### Acceptance Criteria

1. THE Git_Guidelines_Document SHALL specify the `.backups/` directory as the location of Bundle_Backups, SHALL state that a Bundle_Backup is never modified after it is written, and SHALL require every Bundle_Backup to be retained until a recorded Operator decision names that exact file for removal.
2. THE Git_Guidelines_Document SHALL specify the Bundle_Backup naming pattern `<repository>-<semantic-scope>-<short-revision>-<yyyymmdd>.bundle`, and SHALL require each written filename to be unique within `.backups/` such that writing a Bundle_Backup never overwrites or truncates an existing file, requiring a distinct filename where the pattern would otherwise collide.
3. WHEN an operation that rewrites the working tree is requested — hard reset, a checkout or branch switch that discards uncommitted bytes, rebase, merge, stash apply, stash drop, or untracked-file cleaning — THE Git_Guidelines_Document SHALL require the uncommitted bytes to be stashed with the stash message pattern `WIP: <semantic-scope>` and captured as a Bundle_Backup that is verified readable and verified to contain the captured revision before the requested operation proceeds.
4. THE Git_Guidelines_Document SHALL specify the Change_Manifest schema `agentic-change-manifest/v1` carrying `branch`, `baseSha`, and a `paths` list ordered by ascending lexicographic comparison of the path text, so two writes enumerating the same path set produce identical `paths` ordering.
5. THE Git_Guidelines_Document SHALL specify `.agentic-manifests/` as the location of an active lane's Change_Manifest and `.recovery/` as the location of a Recovery_Capture, and SHALL specify the Recovery_Capture directory naming pattern `<semantic-scope>-<yyyymmdd>T<hhmm>Z`, where the timestamp is UTC at minute precision.
6. THE Git_Guidelines_Document SHALL specify the Recovery_Capture contents as a manifest, a tracked patch, retained untracked files, and a completion marker, and SHALL require the completion marker to be written only after the manifest, the tracked patch, and every retained untracked file are written, so a Recovery_Capture without its completion marker is incomplete.
7. WHEN a lane is parked for handoff, THE Git_Guidelines_Document SHALL require the lane revision to be pushed to a remote before the handoff transition, together with a current Change_Manifest and a recorded Recovery_Handle.
8. WHERE a lane is still active, THE Git_Guidelines_Document SHALL permit a Recovery_Handle to be created and recorded before the lane is parked.
9. WHEN an operation would rewrite history, delete beyond one declared artifact, or discard untracked bytes, THE Git_Guidelines_Document SHALL require exactly one recorded Operator decision per occurrence, naming the operation, the affected lane, and the affected paths, and SHALL state that a decision recorded for one occurrence authorizes no other occurrence.
10. IF work is classified for removal, THEN THE Git_Guidelines_Document SHALL require a `keep` / `port` / `drop` classification with exact identity, evidence, and rationale recorded before removal.
11. WHERE a restore is performed from a Recovery_Capture whose completion marker is present, THE Git_Guidelines_Document SHALL require the restored bytes to be byte-for-byte identical to the captured bytes for every path enumerated in that capture's manifest (round-trip property).
12. THE Git_Guidelines_Document SHALL define a recorded Recovery_Handle as exactly one of two values: the exact directory name of a Recovery_Capture under `.recovery/` whose completion marker is present, or the exact filename of a Bundle_Backup under `.backups/` verified readable and verified to contain the captured revision.
13. IF a Bundle_Backup, Change_Manifest, or recorded Operator decision required by this requirement is absent, or the Recovery_Capture selected for a restore lacks its completion marker, THEN THE Git_Guidelines_Document SHALL require the requested operation to resolve as blocked with the lane's tracked and untracked bytes unchanged, SHALL require the blocking reason to name the absent artifact or the incomplete capture, and SHALL state that an incomplete Recovery_Capture satisfies no Recovery_Handle obligation.
14. IF the restored bytes differ from the captured bytes for any path enumerated in the capture's manifest, THEN THE Git_Guidelines_Document SHALL require the restore to resolve as blocked, SHALL require the Recovery_Capture and the pre-restore bytes to be retained, and SHALL require the report to name every differing path.

### Requirement 7: Agent-Driven Commits and Attribution

**User Story:** As the reviewer of my own agents' work, I want every agentic commit to state its scope, its task, and its author mechanism, so that I can audit a run months later from history alone.

#### Acceptance Criteria

1. THE Git_Guidelines_Document SHALL specify the commit subject format `<type>(<scope>): <summary>` as a single line at or below 72 characters, with `<type>` drawn from the enumeration of permitted commit types, `<scope>` equal to the Task_Lane's admitted semantic scope, and `<summary>` a non-empty string of 1 to 60 characters.
2. THE Git_Guidelines_Document SHALL enumerate the permitted commit types as a closed set of at least 1 and at most 12 types, and SHALL state for each type exactly one selection condition by which a reader assigns a change to that type.
3. THE Git_Guidelines_Document SHALL specify the commit trailer set that attributes an agentic change, carrying the task identifier, the semantic scope, the lease epoch, and the acting agent mechanism, and SHALL require each of those four trailers to appear exactly once as a single line of 1 to 200 characters in the commit message's final block, with the semantic scope and lease epoch values equal to those recorded in the lane's accepted claim artifact.
4. THE Git_Guidelines_Document SHALL require one commit per logical unit, defined as a change set carrying exactly one task identifier and exactly one semantic scope that can be reverted alone without requiring the revert of another commit in the same Task_Lane, and SHALL require staging by explicit path or interactive hunk selection while forbidding directory-wide or repository-wide wildcard staging.
5. THE Git_Guidelines_Document SHALL require every path a commit in a Task_Lane changes to be equal to, or a descendant of, a path enumerated in that lane's declared write scope.
6. THE Git_Guidelines_Document SHALL require the Change_Manifest's sorted `paths` list to be updated before each commit is recorded and to equal exactly the set of paths the lane's commits have changed since its `baseSha`, including incidental changes, with neither omissions nor entries the lane did not change.
7. WHEN an agent commits on behalf of the operator, THE Git_Guidelines_Document SHALL require the commit body to contain at least one statement naming what changed and at least one statement giving why it changed, and SHALL forbid a body whose only content restates the subject line.
8. IF a commit writes a path outside the declared write scope, THEN THE Git_Guidelines_Document SHALL attach an `out-of-scope-write` finding to that commit naming each offending path, SHALL keep the commit itself valid and recoverable, and SHALL require any push of that commit to a shared branch to resolve as blocked until the scope is re-admitted or the paths are removed.
9. WHERE a commit is unpushed and authored in the current lane, THE Git_Guidelines_Document SHALL permit amending that commit.
10. IF a required attribution trailer is absent, duplicated, or carries an empty value, THEN THE Git_Guidelines_Document SHALL classify the commit as an `unattributed-agentic-commit` finding, SHALL keep the commit's bytes recoverable, and SHALL require the attribution to be re-recorded before the commit is pushed.
11. IF a proposed commit subject does not match the specified format or exceeds 72 characters, THEN THE Git_Guidelines_Document SHALL require the commit to resolve as blocked, with an error indication naming the violated constraint, until the subject is corrected.
12. IF a commit is already pushed or was authored in a lane other than the current lane, THEN THE Git_Guidelines_Document SHALL forbid amending that commit and SHALL require the correction to be recorded as a new commit.

### Requirement 8: Verification Gates Before Push and Review

**User Story:** As an operator who cannot review every diff, I want a named check with a recorded result standing between an agent's commit and a shared branch, so that trust is mechanical rather than rhetorical.

#### Acceptance Criteria

1. THE Git_Guidelines_Document SHALL require a named check, stated as it is invocable, to run to a terminal exit status against the exact revision to be pushed before any push to a shared branch, where a shared branch is any branch that exists on a remote plus the protected canonical branch.
2. THE Git_Guidelines_Document SHALL require the recorded result of that check to carry the check name, the exact revision checked, the terminal exit status, the pass and fail counts or test summary, and the completion timestamp, and SHALL require that recorded result to be surfaced in the pushing agent's own output.
3. THE Git_Guidelines_Document SHALL require the verdict on a lane's readiness to be issued by a mechanism whose acting agent mechanism and Session ID pair differs from every acting agent mechanism and Session ID pair recorded on that lane's authoring commits, and SHALL require the verdict record to state the verdict outcome, the verdicting mechanism's acting agent mechanism and Session ID, the exact lane revision judged, and the recorded check results relied upon.
4. THE Git_Guidelines_Document SHALL require every verification command already registered in the repository before that lane's admission to run to a terminal recorded result, in addition to the lane's own new check, before any push to a shared branch.
5. THE Git_Guidelines_Document SHALL require a bug-fixing lane to bind two recorded results for its added check — one bound to the pre-fix revision carrying a failing terminal exit status and one bound to the post-fix revision carrying a passing terminal exit status — before the fix lands on a shared branch.
6. THE Git_Guidelines_Document SHALL require pre-commit and pre-push hooks to run to a terminal exit status on the exact revision being committed or pushed, and SHALL require an Operator decision recorded per bypass naming the bypassed hook and that exact revision.
7. WHEN a review request is opened for a lane, THE Git_Guidelines_Document SHALL require the request to bind the current protected canonical base revision, the exact lane revision, and a scope token equal to the admitted semantic scope, and SHALL require the request to resolve as blocked while any of those three bindings is absent or differs from its current observed value.
8. IF a check result is asserted with no recorded run, or a recorded result relied upon is bound to a revision other than the revision being pushed or reviewed, THEN THE Git_Guidelines_Document SHALL classify the assertion as an `evidence-without-run` finding and SHALL require the operation to resolve as blocked.
9. IF the verdict on a lane is issued by a mechanism whose acting agent mechanism and Session ID pair matches a pair recorded on that lane's authoring commits, THEN THE Git_Guidelines_Document SHALL classify the verdict as a `self-graded-verdict` finding and SHALL require the lane to resolve as blocked pending a verdict from a distinct mechanism.
10. IF the classification of a finding cannot be recorded, THEN THE Git_Guidelines_Document SHALL require every other gate of this requirement to remain enforced and the operation to resolve as blocked.
11. THE Git_Guidelines_Document SHALL require each check named under this requirement to state a maximum run duration in seconds, beyond which the run counts as non-terminating.
12. IF a check required before push or review fails, is absent, or does not reach a terminal exit status within its stated maximum run duration, THEN THE Git_Guidelines_Document SHALL require the push and the review request to resolve as blocked and SHALL require the lane's committed bytes to remain recoverable.
13. IF a pre-commit or pre-push hook is bypassed with no recorded Operator decision, THEN THE Git_Guidelines_Document SHALL require the commit or push to resolve as blocked and SHALL require the lane's committed bytes to remain recoverable.

### Requirement 9: Conflict Avoidance and Resolution

**User Story:** As an operator running concurrent lanes, I want a stated resolution path for drift and overlap, so that a conflict is a procedure rather than an improvisation.

#### Acceptance Criteria

1. THE Git_Guidelines_Document SHALL require every lane to rebase or merge the exact protected canonical revision re-read from the protected remote within the same integration operation, and SHALL require that revision identifier to be recorded in the lane's integration request.
2. THE Git_Guidelines_Document SHALL require conflicts to be resolved in the lane that owns the conflicting source, identified as the lane whose accepted claim covers the conflicting path, and not in a consumer, generated projection, or mirror.
3. THE Git_Guidelines_Document SHALL require lanes with overlapping declared write scopes to serialize, so that at most one of those lanes holds an accepted claim covering the overlapping paths at any time and every other lane's integration request resolves as blocked until the holding lane releases, hands off, or is retired, even where their contents would merge cleanly.
4. THE Git_Guidelines_Document SHALL state that content mergeability proves no ownership safety.
5. THE Git_Guidelines_Document SHALL require integration to follow the stated dependency order — shared control and contract sources, then implementation sources, then consumers, then generated projections, then mirrors — and SHALL require a consumer, generated projection, or mirror integration request to resolve as blocked while an owner lane earlier in that order holds an accepted claim on the same control, contract, or source path and has not integrated.
6. WHEN the protected canonical revision differs from the revision recorded by a lane at rebase, review, or candidate preparation, THE Git_Guidelines_Document SHALL require the waiting run's bytes to be preserved, the run to be retired, and a new run to be resealed from the current protected canonical revision.
7. WHEN the same integration approach — the same named git operation applied to the same lane and the same conflicting path set — fails a second time, THE Git_Guidelines_Document SHALL require a root cause to be recorded before any further attempt, and SHALL require every further attempt of that approach to resolve as blocked until both the root cause is recorded and a different approach is named.
8. IF the fence revision recorded for a lane differs from the fence revision carried by the current accepted claim result artifact, THEN THE Git_Guidelines_Document SHALL classify the state as a `stale-collaboration-fence` finding and require the operation to resolve as blocked.
9. WHERE a lane is retired and its path ownership is serialized to another lane, THE Git_Guidelines_Document SHALL permit the surviving lane's operation to proceed on the fence revision carried by the current accepted claim result artifact.
10. IF two lanes could publish different bytes for one path, determined by comparing their declared write scopes under the stated overlap rules of path equality, ancestor and descendant relationship, shared semantic artifact, and wildcard scope, THEN THE Git_Guidelines_Document SHALL require ownership to be serialized or one lane retired before either publishes.
11. IF a conflict on a path is resolved in a lane other than the lane whose accepted claim covers that path, THEN THE Git_Guidelines_Document SHALL classify the resolution as a `misplaced-conflict-resolution` finding, require the operation to resolve as blocked, and require the resolving lane's bytes to be preserved as recoverable.
12. IF a commit, push, or integration would publish tracked bytes still carrying unresolved conflict content, THEN THE Git_Guidelines_Document SHALL classify the operation as an `unresolved-conflict-publish` finding and require the operation to resolve as blocked with the lane's bytes retained unchanged.
13. WHEN more than one lane holds a pending integration request over overlapping declared write scopes, THE Git_Guidelines_Document SHALL require the serialization order to be determined by the dependency order of this requirement first, then by the lowest lease epoch among the tied requests, then by lexicographic order of Scope ID, so the order is reproducible by any reader of the recorded artifacts.

### Requirement 10: Dev to Prod to Delivery Promotion

**User Story:** As the only person who can authorize a release, I want the promotion path across the three repositories written down with its gates closed by default, so that no green check ever deploys on my behalf.

#### Acceptance Criteria

1. THE Git_Guidelines_Document SHALL define the Promotion_Chain as the ordered movement Dev_Repository → Prod_Mirror → Delivery_Surface.
2. THE Git_Guidelines_Document SHALL state the lane of each stage of the Promotion_Chain and the named boundary between consecutive stages.
3. THE Git_Guidelines_Document SHALL state that every boundary of the Promotion_Chain reads closed unless a Coordination_Artifact records an Operator instruction carrying the acting Operator identity, the named boundary being crossed, the exact candidate identity, the exact target stage, and the instruction's issue time, and SHALL state that an instruction that is absent, unparseable, or non-matching — one whose candidate identity or target stage differs from the operation's — reads as closed.
4. THE Git_Guidelines_Document SHALL state that protected integration in the Dev_Repository grants integration authority only and grants no deployment authority.
5. THE Git_Guidelines_Document SHALL require one immutable candidate sealed from the exact release frontier, binding exactly six identities — source revision, dependency closure, target stage, review verdict, artifact, and rollback point — and SHALL require exactly one candidate per deployment attempt, a new candidate identity for each reseal, and the superseded candidate identity to carry no authority.
6. THE Git_Guidelines_Document SHALL require, before any deployment to the Delivery_Surface, a recorded decision that carries the acting Operator identity, names exactly one candidate identity and exactly one target stage, and is valid for exactly one deployment attempt, and SHALL state that a decision already consumed by an attempt authorizes no further attempt.
7. THE Git_Guidelines_Document SHALL require publication to the Prod_Mirror to follow live verification of the Delivery_Surface, and SHALL bind that publication to three recorded passing claims — the deployed revision claim, the public route reachability claim, and the authoritative state readback claim — each recorded from its own verification run.
8. THE Git_Guidelines_Document SHALL require the deployed revision claim, the public route reachability claim, and the authoritative state readback claim to be recorded as three separate claims, each carrying the recorded result of its own run, and SHALL forbid one aggregate result from standing as the record of more than one of those claims.
9. THE Git_Guidelines_Document SHALL state, for each stage of the Promotion_Chain, the rollback trigger, the restoration target revision or state, the observable outcome that ends the rollback, and the code rollback disposition and the state rollback disposition as two separate dispositions, and SHALL name each disposition that is irreversible.
10. WHEN any bound identity drifts after authorization, THE Git_Guidelines_Document SHALL require the authorization to be invalidated and SHALL forbid retargeting or rebuilding under the invalidated authorization.
11. WHERE an authorization is invalidated and the affected work can continue without a deployment claim, THE Git_Guidelines_Document SHALL permit the run to continue in the `authoring` lane and SHALL require a fresh candidate, review, and authorization before the next deployment attempt.
12. WHERE an error condition other than identity drift invalidates a receipt in the chain, THE Git_Guidelines_Document SHALL permit authorization invalidation and run retirement on the same terms as identity drift.
13. WHERE a local runtime command verifies a stage before promotion, THE Git_Guidelines_Document SHALL name that command inside a Reference_Implementation_Block.
14. IF a lane operation would mutate the Prod_Mirror or the Delivery_Surface during authoring, THEN THE Git_Guidelines_Document SHALL classify the operation as a `deploy-boundary-breach` finding.
15. IF a deployment to the Delivery_Surface proceeds with no recorded decision matching that deployment's candidate identity and target stage, including where the only record present is an automated check result, THEN THE Git_Guidelines_Document SHALL classify the deployment as a `deploy-boundary-breach` finding and SHALL require the operation to resolve as blocked.
16. IF the deployed revision claim, the public route reachability claim, or the authoritative state readback claim fails, is absent, or cannot be recorded, THEN THE Git_Guidelines_Document SHALL require publication to the Prod_Mirror to resolve as blocked and SHALL require the stage to enter its rollback path with the code rollback disposition and the state rollback disposition recorded.

### Requirement 11: Discoverability and Invocation

**User Story:** As an agent starting a git task, I want to reach these rules through the same invocation grammar as every other contract, so that the document is used rather than merely written.

#### Acceptance Criteria

1. THE Docs_Index SHALL contain exactly one Document Map row for the Git_Guidelines_Document carrying, in separate cells, the document path, a one-line role description at or below 120 characters, and the load stage or task condition under which the document is loaded, with every one of those three cells non-empty.
2. THE Command_Dictionary SHALL contain exactly one `/` token that routes to the Git_Guidelines_Document, and SHALL declare for that token exactly one intent statement, exactly one completion signal, at least one required `@` binding, and at least one `#` semantic filter.
3. THE Command_Dictionary SHALL declare the `/` token string for the Git_Guidelines_Document as unique across every token string in the Command_Dictionary, and SHALL list the Git_Guidelines_Document path exactly once in its `source_docs` and that token string exactly once in its `dictionary_entries`.
4. THE Semantic_Dictionary SHALL contain at least one `#` filter selecting the git collaboration rule family and naming the Git_Guidelines_Document path as a source.
5. THE Binding_Dictionary SHALL contain at least one `@` binding resolving to the Git_Guidelines_Document path as a source.
6. THE Git_Guidelines_Document SHALL declare in its frontmatter its `/` token, its `#` filters, and its `@` bindings as strings character-for-character equal to the corresponding registered entries in the Command_Dictionary, the Semantic_Dictionary, and the Binding_Dictionary, so a reader of the document alone can state how to invoke it.
7. WHEN the Command_Dictionary catalog changes, THE Conformance_Checker SHALL recompute the catalog digest and the catalog entry count and compare both recomputed values against the values recorded in the Command_Dictionary metadata.
8. WHEN the Command_Dictionary metadata names an entry, THE Conformance_Checker SHALL compare that entry name against the token strings present in the Command_Dictionary contents.
9. IF the Git_Guidelines_Document is present and the Docs_Index row, the `/` token, or an entry named by the Command_Dictionary metadata is absent from the corresponding artifact's contents, THEN THE Conformance_Checker SHALL exit non-zero, name the missing registration and the artifact it is absent from, and leave the Docs_Index, the dictionaries, and the Git_Guidelines_Document unmodified.
10. IF the Git_Guidelines_Document is present, the Docs_Index row and the `/` token are present, and the `#` filter or the `@` binding is absent, THEN THE Conformance_Checker SHALL name the missing registration as an advisory finding and exit zero.
11. IF a recomputed Command_Dictionary catalog digest or catalog entry count differs from the value recorded in the Command_Dictionary metadata, THEN THE Conformance_Checker SHALL exit non-zero, name the mismatched quantity together with the recorded and recomputed values, and leave the Command_Dictionary and the Git_Guidelines_Document unmodified.
12. IF a frontmatter-declared invocation token of the Git_Guidelines_Document is not character-for-character equal to its registered counterpart, or has no registered counterpart in the Command_Dictionary, the Semantic_Dictionary, or the Binding_Dictionary, THEN THE Conformance_Checker SHALL exit non-zero, name the divergent token and the two artifacts compared, and leave the Git_Guidelines_Document and the dictionaries unmodified.
13. IF a Docs_Index row, a `/` token, a `#` filter, or an `@` binding references a document path that does not exist, THEN THE Conformance_Checker SHALL exit non-zero, name the referencing registration and the referenced path, and leave the Docs_Index, the dictionaries, and the Git_Guidelines_Document unmodified.

### Requirement 12: Findings Vocabulary and Rule Identity

**User Story:** As the operator comparing one run to the next, I want git violations reported in the vocabulary I already use, so that a regression is detectable instead of merely describable.

#### Acceptance Criteria

1. THE Git_Guidelines_Document SHALL derive every Rule_ID as its owning section anchor, followed by the character `#`, followed by the rule's 1-based ordinal counted in document order across the directive bullets and table rows of that section, and SHALL make every derived Rule_ID unique document-wide.
2. THE Git_Guidelines_Document SHALL assign every rule exactly one classification of either artifact-bearing — decidable from a named git object or a named Coordination_Artifact and able to raise at least one finding type listed in the findings table — or advisory — able to raise no finding type and unable to change the Conformance_Checker verdict.
3. THE Git_Guidelines_Document SHALL reuse, without renaming, the finding type names and severities enumerated by the Execution_Companion and the Authoring_Authority, including `out-of-scope-write`, `evidence-without-run`, `self-graded-verdict`, `stale-collaboration-fence`, and `deploy-boundary-breach`.
4. THE Git_Guidelines_Document SHALL contain a findings table in which every row carries the rule family, the Rule_IDs that can raise the finding, the finding type name, the severity, and an ownership marker valued either `inherited` with its named owner or `document-local`.
5. WHEN a run completes, THE Git_Guidelines_Document SHALL require one non-negative integer count for each finding type listed in the findings table, with a count of zero stated explicitly for every listed finding type that raised no finding in that run.
6. IF the findings table lists a finding type that no rule of the Git_Guidelines_Document can raise, THEN THE Conformance_Checker SHALL exit non-zero and name that finding type together with the row that lists it.
7. IF the Git_Guidelines_Document states a finding type name owned by the Execution_Companion or the Authoring_Authority with a trigger condition, a severity, or a raising scope differing from the named owner's, THEN THE Conformance_Checker SHALL exit non-zero and report the redefinition with the finding type name and its owner.
8. WHERE a finding type name is owned by neither the Execution_Companion nor the Authoring_Authority, THE Git_Guidelines_Document SHALL permit that name to be introduced as a Document_Local_Finding — including `unattributed-agentic-commit`, `misplaced-conflict-resolution`, and `unresolved-conflict-publish` — and SHALL require the name to carry the `document-local` ownership marker, a severity drawn from the severity set inherited from the Execution_Companion and the Authoring_Authority, and at least one raising Rule_ID.
9. IF a rule of the Git_Guidelines_Document can raise a finding type that the findings table omits, THEN THE Conformance_Checker SHALL exit non-zero and name that finding type together with the Rule_ID that raises it.
10. IF a rule carries no classification or more than one classification, a Rule_ID is duplicated document-wide, or a Rule_ID's ordinal differs from the rule's counted position within its section, THEN THE Conformance_Checker SHALL exit non-zero and name the offending rule or Rule_ID.

### Requirement 13: Validation and Deterministic Conformance

**User Story:** As an operator who distrusts unverified documents, I want one command that judges whether the git guidelines and their registrations are conformant, so that compliance is a result rather than an opinion.

#### Acceptance Criteria

1. THE Git_Guidelines_Document SHALL contain a validation checklist partitioned into exactly the five gates pre-lane, per-commit, pre-push, pre-promotion, and post-run, with each gate carrying at least one check, each check naming the Rule_ID it enforces and the observable evidence that satisfies it, and every artifact-bearing rule of the Git_Guidelines_Document appearing in at least one gate.
2. THE Conformance_Checker SHALL use `huijoohwee.github.io/scripts/check-git-guidelines.mjs` as its entry point, and THE Git_Guidelines_Document SHALL name the Conformance_Checker as it is invocable inside a Reference_Implementation_Block, stating the invocation, the inputs the invocation reads, and the meaning of its exit status as zero for conformant and non-zero for not conformant.
3. THE Conformance_Checker SHALL exit zero only when every artifact-bearing rule of the Git_Guidelines_Document is satisfied and the Docs_Index row and the `/` token required by Requirement 11 are present, and SHALL keep the exit status zero when the only findings raised are advisory.
4. WHEN the Conformance_Checker is run twice against inputs that are identical after the normalization stated per criterion 5, THE Conformance_Checker SHALL produce two reports whose verdict, total finding count, and ordered sequence of findings are identical, where each finding is compared by Rule_ID, finding type, severity, and location (idempotence property).
5. THE Conformance_Checker SHALL state the normalization it applies before comparison as an exhaustive list covering timestamps, run identifiers, absolute path prefixes, line-ending style, trailing whitespace, and the ordering of ordering-insensitive metadata, and SHALL state that no other normalization is applied.
6. WHEN the Conformance_Checker reports findings, THE Conformance_Checker SHALL order them by ascending Rule_ID, then by ascending location within that Rule_ID, then by ascending finding type name, and SHALL collapse findings sharing one Rule_ID and one location into a single entry retaining the highest severity and a repeat count.
7. THE Conformance_Checker SHALL run with zero model invocations, with zero outbound network connections to any host other than the configured git remotes, and with no dependency beyond git and the local filesystem.
8. IF a required input to the Conformance_Checker is absent, unparseable by its declared schema, stale because its recorded fence revision differs from the current accepted fence revision or its recorded base revision differs from the revision under check, or unreadable within the bound stated in criterion 11, THEN THE Conformance_Checker SHALL exit non-zero, name the input and which of those four conditions applies, and SHALL NOT report a passing verdict.
9. WHEN the Conformance_Checker completes, THE Conformance_Checker SHALL emit a report stating the verdict, the total finding count, the count per severity including zero counts, and the Rule_ID of every unsatisfied artifact-bearing rule.
10. IF a configured git remote required by a check is unreachable within 30 seconds, THEN THE Conformance_Checker SHALL exit non-zero, report a finding naming the unreachable remote and the checks it blocked, and SHALL NOT report a passing verdict.
11. WHEN the Conformance_Checker is invoked against a Git_Guidelines_Document of at most 400 lines together with the registrations required by Requirement 11, THE Conformance_Checker SHALL return a verdict within 60 seconds of wall-clock time.

### Requirement 14: Anti-Patterns and Mantra

**User Story:** As an operator reading fast, I want the failure modes and their corrections stated side by side, so that recognizing a mistake costs one line instead of one section.

#### Acceptance Criteria

1. THE Git_Guidelines_Document SHALL contain one anti-pattern section, addressable by its own `##` heading anchor and listed in the Module Index, holding at least seven and at most twenty anti-pattern pairs and occupying at or below 50 lines in total.
2. THE Git_Guidelines_Document SHALL name, as distinct anti-pattern pairs, at minimum the anti-patterns for authoring on the canonical branch, undeclared write scope, local-only lease as cross-device authority, self-graded lane verdicts, history rewrite without preservation, deploy on green integration, and stale authorization reuse, and SHALL state in each corrective line the Rule_ID or the named rule family that owns the corrective rule.
3. THE Git_Guidelines_Document SHALL contain one mantra section, addressable by its own `##` heading anchor, stating exactly one clause per rule family the Git_Guidelines_Document declares as owned under Requirement 2, with each clause on one line at or below 120 characters and the section occupying at or below 25 lines.
4. THE Git_Guidelines_Document SHALL express each anti-pattern pair as exactly two lines — one prohibited line and one corrective line — each line at or below 120 characters, with no pair spanning a third line and no pair carrying a nested list or an example block.
5. WHERE an anti-pattern corresponds to a finding type listed in the findings table of Requirement 12, THE Git_Guidelines_Document SHALL name that finding type in the pair's corrective line, using the finding type name exactly as enumerated by the Execution_Companion, the Authoring_Authority, or the Document_Local_Finding set.
6. IF any of the seven anti-patterns named in criterion 2 is absent, or a pair exceeds two lines, or a line exceeds 120 characters, THEN THE Conformance_Checker SHALL exit non-zero and name the absent anti-pattern or the Rule_ID of the offending pair.
7. IF the mantra clause count differs from the count of rule families the Git_Guidelines_Document declares as owned, THEN THE Conformance_Checker SHALL exit non-zero and name each rule family that carries no clause and each clause that maps to no owned rule family.
