# Requirements Document

## Introduction

This feature specifies an auditable alignment and conformance capability that answers one question with evidence: do an authoring Guideline_Set and a Runtime_Repository, taken together, enforce a complete from-0-to-1 pipeline that terminates in a verifiable runtime-ready state?

The first audited pair is the authoring guideline document owned by the guidelines repository and the runtime documentation control surface owned by the Agentic Canvas OS repository. The capability itself is defined so that it applies to any guideline set and any runtime repository, because the audited guideline set declares universality, neutrality, agnosticity, and modularity as binding rules and the audit capability must satisfy the same rules it enforces.

Scope for this increment is documentation and audit only. The Alignment_Auditor reads sources, derives models, and writes an Audit_Report. Source mutation and any promotion to a production or edge surface stay outside this increment and require an explicit operator instruction.

Operating context folded into the requirements: a solo-dev, AI-native harness and orchestration stack; priorities of low total cost of ownership, zero-infrastructure and browser-based delivery, mobile reach, local-first and offline-capable behavior, token economics, free and open-source dependencies, minimum-viable-maximum-value scoping, short time-to-value, and high return on investment; invocation through MCP plus `/`, `#`, and `@` surfaces; and a three-lane environment topology of development, production mirror, and edge delivery.

## Glossary

- **Alignment_Auditor**: The system specified by this document. Reads a Guideline_Set and a Runtime_Repository, derives models, evaluates conformance, and emits an Audit_Report.
- **Guideline_Set**: The authored document set that defines authoring rules, phase gates, checklists, templates, anti-pattern guards, readiness dimensions, and the Verifiable_Completion_Condition primitive.
- **Runtime_Repository**: The audited repository that holds runtime documents, contract schemas, validation commands, invocation dictionaries, and declared readiness statuses.
- **Guideline_Parser**: The component that converts Guideline_Set documents into a Guideline_Model.
- **Guideline_Printer**: The component that renders a Guideline_Model into a Guideline_Digest.
- **Guideline_Model**: The parsed representation of a Guideline_Set as a set of Normative_Elements.
- **Guideline_Digest**: The Markdown rendering of a Guideline_Model.
- **Normative_Element**: One binding statement extracted from the Guideline_Set: a directive, a phase-gate condition, a checklist item, a required template field, or an anti-pattern guard.
- **Artifact_Bearing_Element**: One Normative_Element that the Guideline_Parser marks as requiring a corresponding Runtime_Repository artifact.
- **Advisory_Element**: One Normative_Element that the Guideline_Parser marks as guidance without a required corresponding artifact.
- **Universal_Scope_Document**: One audited document whose content declares that the document applies to any product, domain, language, or runtime.
- **Element_Id**: A stable identifier for a Normative_Element derived from the owning section heading anchor and the element text.
- **Artifact_Indexer**: The component that builds the Artifact_Index.
- **Artifact_Index**: The set of audited Runtime_Repository entries, one entry per Markdown document, contract schema, validation command, or declared readiness status.
- **Artifact_Printer**: The component that renders an Artifact_Index into Markdown.
- **Traceability_Mapper**: The component that links Normative_Elements, Artifact_Index entries, and Verifiable_Completion_Conditions.
- **Traceability_Chain**: The linked path from a product requirement, through an architecture element, through a decision record, through a specification, to a Verifiable_Completion_Condition and its Evidence_Reference.
- **Verifiable_Completion_Condition**: One measurable end state plus a stated check plus a scope constraint, evaluable from surfaced output.
- **Evidence_Reference**: A named reproducible check plus a recorded result that supports a readiness claim.
- **Gate_Evaluator**: The component that evaluates Pipeline_Gates.
- **Pipeline_Gate**: One ordered stage of the from-0-to-1 pipeline, carrying one entry condition, one exit condition, and one required evidence type.
- **Gate_State**: One of `unmet`, `partially-met`, `met`.
- **Readiness_Evaluator**: The component that assigns Readiness_Levels.
- **Readiness_Level**: One rung of the ordered ladder `undocumented` < `spec-complete` < `dev-proven` < `runtime-ready` < `production-verified`.
- **Drift_Detector**: The component that classifies divergence between the Guideline_Model and the Artifact_Index.
- **Finding**: One typed audit record carrying a Finding_Type, a severity, a guideline anchor, an artifact reference, an evidence excerpt, and a remediation statement.
- **Finding_Type**: One member of the documented Finding enumeration.
- **Neutrality_Checker**: The component that evaluates universality, neutrality, agnosticity, and modularity conformance.
- **Economics_Checker**: The component that evaluates return-on-investment, total-cost-of-ownership, token-budget, time-to-value, and delivery-reach conformance.
- **Invocation_Checker**: The component that evaluates `/`, `#`, `@`, and MCP surface conformance.
- **Topology_Checker**: The component that evaluates lane topology and deploy-boundary conformance.
- **Report_Writer**: The component that emits the Audit_Report.
- **Audit_Report**: The versioned Markdown artifact that records the alignment summary, readiness gap matrix, Finding table, and Pipeline_Gate state table.
- **Audit_Run**: One complete execution of the Alignment_Auditor over a stated pair of input revisions.
- **Audit_Output_Directory**: The single directory that receives every file the Alignment_Auditor writes.
- **Lane**: One environment stage of the deployment topology: development, production mirror, or edge delivery.
- **Deploy_Boundary**: The gate between adjacent Lanes that requires an explicit operator instruction before promotion.
- **Invocation_Route**: One `/` command route, one `#` semantic tag, one `@` binding, or one MCP tool identity.

## Requirements

### Requirement 1: Non-Mutating Audit Scope

**User Story:** As a solo founder, I want the audit to run without changing source files or touching deployed surfaces, so that I can measure alignment risk at zero blast radius.

#### Acceptance Criteria

1. THE Alignment_Auditor SHALL confine every file write of an Audit_Run to the Audit_Output_Directory.
2. WHEN an Audit_Run completes, THE Alignment_Auditor SHALL report the count of files it modified outside the Audit_Output_Directory as zero.
3. IF a Finding remediation requires a change to a Guideline_Set file or a Runtime_Repository file, THEN THE Alignment_Auditor SHALL record the proposed change as a remediation statement and leave that file byte-identical.
4. THE Alignment_Auditor SHALL record every production-surface and edge-surface mutation as outside the scope of an Audit_Run.
5. IF a Finding remediation requires a production-surface or edge-surface mutation, THEN THE Alignment_Auditor SHALL assign the remediation a `deploy-gated` state and require an explicit operator instruction reference before that state advances.
6. THE Alignment_Auditor SHALL read Guideline_Set and Runtime_Repository content from paths supplied as configuration values.

### Requirement 2: Guideline Element Extraction

**User Story:** As a maintainer, I want every binding statement in the Guideline_Set to become an addressable element, so that conformance can be measured statement by statement instead of by impression.

#### Acceptance Criteria

1. WHEN a Guideline_Set document is supplied, THE Guideline_Parser SHALL produce a Guideline_Model containing one Normative_Element for each directive, phase-gate condition, checklist item, required template field, and anti-pattern guard present in that document.
2. THE Guideline_Parser SHALL derive each Normative_Element from parsed frontmatter and document body content.
3. THE Guideline_Parser SHALL assign each Normative_Element one Element_Id derived from the owning section heading anchor and the element text.
4. THE Guideline_Printer SHALL render a Guideline_Model into a Guideline_Digest.
5. FOR ALL Guideline_Model instances, THE Guideline_Parser SHALL parse the Guideline_Digest produced by the Guideline_Printer into a Guideline_Model equal to the input Guideline_Model.
6. IF a Guideline_Set document omits a required frontmatter key, THEN THE Guideline_Parser SHALL record a Finding of type `missing-frontmatter-key` naming that key and continue with the remaining documents.
7. THE Guideline_Parser SHALL record for each Normative_Element the section heading anchor that owns it.
8. THE Guideline_Parser SHALL mark each Normative_Element as exactly one of Artifact_Bearing_Element and Advisory_Element based on the element text.

### Requirement 3: Runtime Artifact Inventory

**User Story:** As a maintainer, I want a content-derived inventory of the Runtime_Repository, so that audit conclusions rest on what documents declare rather than on where files sit.

#### Acceptance Criteria

1. WHEN an Audit_Run starts, THE Artifact_Indexer SHALL produce an Artifact_Index containing one entry for each audited Markdown document, contract schema, validation command, and declared readiness status.
2. THE Artifact_Indexer SHALL record for each entry the declared status value, the declared runtime scope, the declared owner, and the declared proof reference taken from parsed frontmatter and document body content.
3. THE Artifact_Indexer SHALL derive each entry attribute from document content and exclude file path segments and directory names from that derivation.
4. IF an audited document declares a status value that is absent from the Readiness_Level ladder, THEN THE Artifact_Indexer SHALL record a Finding of type `unknown-status` naming the declared value.
5. THE Artifact_Printer SHALL render an Artifact_Index into Markdown.
6. FOR ALL Artifact_Index instances, THE Artifact_Indexer SHALL parse the Markdown produced by the Artifact_Printer into an Artifact_Index equal to the input Artifact_Index.
7. THE Artifact_Indexer SHALL record for each validation command entry the command text as declared in the audited document.

### Requirement 4: Traceability Chain Closure

**User Story:** As a maintainer, I want bidirectional traceability from guideline statement to runtime evidence, so that orphaned rules and unbacked claims surface as named gaps.

#### Acceptance Criteria

1. THE Traceability_Mapper SHALL construct a Traceability_Chain that links each Normative_Element to its matching Artifact_Index entries and each Artifact_Index entry to its matching Verifiable_Completion_Conditions.
2. THE Traceability_Mapper SHALL record for each link the Element_Id, the artifact reference, and the Evidence_Reference.
3. WHEN an Artifact_Bearing_Element has zero linked Artifact_Index entries, THE Traceability_Mapper SHALL record a Finding of type `unimplemented-guideline`.
4. WHEN an Advisory_Element has zero linked Artifact_Index entries, THE Traceability_Mapper SHALL record that element as advisory coverage and omit a Finding for that element.
5. WHEN an Artifact_Index entry has zero linked Normative_Elements, THE Traceability_Mapper SHALL record a Finding of type `unguided-artifact`.
6. WHEN an Artifact_Index entry declares a Readiness_Level of `runtime-ready` and has zero linked Verifiable_Completion_Conditions, THE Traceability_Mapper SHALL record a Finding of type `unproven-claim` with severity `blocker`.
7. WHERE a linked reference names a path that is absent from the supplied input roots, THE Traceability_Mapper SHALL record a Finding of type `unresolvable-reference` and continue with the remaining links.
8. THE Traceability_Mapper SHALL report the count of Artifact_Bearing_Elements, the count of linked Artifact_Bearing_Elements, and the ratio of linked to total Artifact_Bearing_Elements.

### Requirement 5: From-0-to-1 Pipeline Gate Model

**User Story:** As a solo founder, I want the from-0-to-1 pipeline expressed as ordered gates with entry and exit conditions, so that I can see exactly which stage blocks runtime readiness.

#### Acceptance Criteria

1. THE Gate_Evaluator SHALL define an ordered Pipeline_Gate sequence covering problem validation, requirements authoring, architecture authoring, alignment review, implementation, local proof, and release readiness.
2. THE Gate_Evaluator SHALL derive the Pipeline_Gate sequence and each gate condition from Guideline_Set content.
3. THE Gate_Evaluator SHALL record for each Pipeline_Gate one entry condition, one exit condition, and one required evidence type.
4. WHEN THE Gate_Evaluator evaluates a Pipeline_Gate, THE Gate_Evaluator SHALL assign exactly one Gate_State to that Pipeline_Gate.
5. THE Gate_Evaluator SHALL assign the Gate_State `met` to a Pipeline_Gate only when every Normative_Element mapped to that Pipeline_Gate carries at least one Evidence_Reference.
6. IF the Runtime_Repository documents a stage order that differs from the Guideline_Set-derived Pipeline_Gate order, THEN THE Gate_Evaluator SHALL record a Finding of type `gate-order-drift` naming both orders.
7. WHEN a Pipeline_Gate holds the Gate_State `unmet` and a later Pipeline_Gate holds the Gate_State `met`, THE Gate_Evaluator SHALL record a Finding of type `gate-sequence-violation`.

### Requirement 6: Runtime Readiness As A Verifiable Condition

**User Story:** As an operator, I want runtime readiness defined as an evidence-bound level rather than a narrative claim, so that promotion decisions rest on reproducible checks.

#### Acceptance Criteria

1. THE Readiness_Evaluator SHALL assign each audited capability exactly one Readiness_Level from the ordered ladder `undocumented`, `spec-complete`, `dev-proven`, `runtime-ready`, `production-verified`.
2. THE Readiness_Evaluator SHALL derive each assigned Readiness_Level from Evidence_References recorded in the Traceability_Chain.
3. WHILE a capability holds zero Evidence_References, THE Readiness_Evaluator SHALL assign that capability a Readiness_Level of `spec-complete` or lower.
4. WHEN a capability holds one Evidence_Reference that names a reproducible local check and a recorded result, THE Readiness_Evaluator SHALL assign that capability a Readiness_Level of `dev-proven` or higher.
5. WHEN a capability holds Evidence_References for every Verifiable_Completion_Condition linked to it, THE Readiness_Evaluator SHALL assign that capability the Readiness_Level `runtime-ready`.
6. THE Readiness_Evaluator SHALL assign the Readiness_Level `production-verified` only when the capability holds one recorded production-surface check result and one explicit operator deploy instruction reference.
7. WHEN Evidence_References are added to a capability and every existing Evidence_Reference of that capability is retained, THE Readiness_Evaluator SHALL assign that capability a Readiness_Level greater than or equal to the previously assigned Readiness_Level.
8. THE Readiness_Evaluator SHALL report local readiness and deployed readiness as two separate fields for each audited capability.

### Requirement 7: Gap And Drift Detection

**User Story:** As a maintainer, I want divergence between guidelines and runtime documents classified and prioritized, so that I can fix blockers before cosmetic gaps.

#### Acceptance Criteria

1. THE Drift_Detector SHALL assign each Finding exactly one Finding_Type from the documented Finding enumeration.
2. THE Drift_Detector SHALL assign each Finding exactly one severity from `blocker`, `major`, `minor`.
3. THE Drift_Detector SHALL record for each Finding one guideline anchor, one artifact reference, one evidence excerpt, and one remediation statement.
4. WHEN two audited documents declare conflicting statuses for one capability, THE Drift_Detector SHALL record a Finding of type `status-conflict` naming both documents.
5. WHEN an Evidence_Reference names a validation command that is absent from the Artifact_Index command entries, THE Drift_Detector SHALL record a Finding of type `stale-evidence`.
6. WHEN two audited documents declare ownership of one capability contract, THE Drift_Detector SHALL record a Finding of type `duplicate-owner` naming both documents.
7. WHEN one audited status field combines a local readiness claim and a deployed readiness claim, THE Drift_Detector SHALL record a Finding of type `blended-status`.
8. WHEN an audited document names a required companion document that is absent from the Artifact_Index, THE Drift_Detector SHALL record a Finding of type `missing-companion`.

### Requirement 8: Universality, Neutrality, Agnosticity, And Modularity Conformance

**User Story:** As a maintainer, I want the audit to enforce the same neutrality rules the Guideline_Set declares, so that the guidelines stay liftable into any product context.

#### Acceptance Criteria

1. THE Neutrality_Checker SHALL evaluate each audited document against the universality rule, the neutrality rule, the agnosticity rule, and the modularity rule.
2. WHEN a document that declares universal scope names a brand, product, or vendor outside a labelled reference-implementation block, THE Neutrality_Checker SHALL record a Finding of type `vendor-coupling`.
3. WHEN an audited document states that a normative claim derives from a file path segment or a directory name, THE Neutrality_Checker SHALL record a Finding of type `path-derived-claim` quoting that statement.
4. WHEN a `##` section of a Universal_Scope_Document depends on another section without naming that section, THE Neutrality_Checker SHALL record a Finding of type `non-modular-section`.
5. WHERE an audited document is absent from the Universal_Scope_Document set, THE Neutrality_Checker SHALL omit the modularity rule from the evaluation of that document.
6. FOR ALL audited documents, THE Neutrality_Checker SHALL produce one identical Finding set for one unchanged document content when the containing directory name changes.
7. THE Neutrality_Checker SHALL report the count of `vendor-coupling` Findings for the Guideline_Set separately from the count for the Runtime_Repository.

### Requirement 9: Solo-Dev Economics And Delivery Reach Conformance

**User Story:** As a solo founder, I want cost, token, time-to-value, and reach constraints checked as first-class conformance items, so that alignment never ships an economically unviable pipeline.

#### Acceptance Criteria

1. THE Economics_Checker SHALL verify that each audited feature-bearing document records a return-on-investment statement, a 12-month total-cost-of-ownership statement, a token-budget statement, and a time-to-value statement.
2. WHEN an audited document omits one or more of the four statements named in criterion 9.1, THE Economics_Checker SHALL record one Finding of type `missing-economics-metric` for each omitted statement, naming that statement.
3. WHEN a total-cost-of-ownership statement combines a managed deployment model and a self-managed deployment model into one figure, THE Economics_Checker SHALL record a Finding of type `blended-deployment-tco`.
4. WHEN an audited document names a proprietary dependency without a free-and-open-source alternative comparison, THE Economics_Checker SHALL record a Finding of type `missing-foss-comparison`.
5. WHEN an audited AI pipeline omits a maximum-iteration bound or a circuit-breaker condition, THE Economics_Checker SHALL record a Finding of type `unbounded-loop` with severity `blocker`.
6. WHEN an audited discovery or read view declares a non-zero token cost, THE Economics_Checker SHALL record a Finding of type `paid-read-path`.
7. THE Economics_Checker SHALL verify that each audited user-facing capability records a delivery statement covering browser reach, mobile reach, and offline behavior.

### Requirement 10: Invocation Surface Conformance

**User Story:** As an external agent author, I want every documented invocation route to resolve to exactly one owner, so that I can call the runtime without scraping or guessing.

#### Acceptance Criteria

1. THE Invocation_Checker SHALL verify that each documented `/` command route, `#` semantic tag, and `@` binding resolves to exactly one owner document in the Artifact_Index.
2. WHEN an Invocation_Route resolves to zero owner documents, THE Invocation_Checker SHALL record a Finding of type `orphan-route`.
3. WHEN an Invocation_Route resolves to two or more owner documents, THE Invocation_Checker SHALL record a Finding of type `ambiguous-route`.
4. THE Invocation_Checker SHALL verify that each documented MCP tool identity appears in the federation contract document and in the capability catalog document.
5. WHEN a documented MCP tool identity is absent from the federation contract document, THE Invocation_Checker SHALL record a Finding of type `unfederated-tool`.
6. THE Invocation_Checker SHALL report the count of documented Invocation_Routes and the count of resolved Invocation_Routes.

### Requirement 11: Environment Topology And Deploy Boundary Conformance

**User Story:** As an operator, I want the development, production mirror, and edge lanes documented with explicit gates, so that no development command can silently reach a public surface.

#### Acceptance Criteria

1. THE Topology_Checker SHALL verify that the audited documents record one development Lane, one production mirror Lane, and one edge delivery Lane.
2. THE Topology_Checker SHALL verify that each Lane transition records one named Deploy_Boundary, one Evidence_Reference, and one rollback statement.
3. WHEN an audited document describes a development command as mutating a production surface or an edge surface, THE Topology_Checker SHALL record a Finding of type `deploy-boundary-breach` with severity `blocker`.
4. WHILE one or more Lanes are absent from the audited documents, THE Topology_Checker SHALL continue the deploy-boundary-breach evaluation named in criterion 11.3 across every audited document.
5. IF a Lane transition omits an explicit operator approval statement, THEN THE Topology_Checker SHALL record a Finding of type `ungated-promotion`.
6. THE Topology_Checker SHALL verify that each documented runtime component appears in one topology table with a named connection type and a named data residency.
7. WHEN a documented runtime component omits a connection type or a data residency, THE Topology_Checker SHALL record a Finding of type `incomplete-topology-node`.
8. THE Topology_Checker SHALL record the Deploy_Boundary state as `closed` for every Audit_Run that lacks an explicit operator deploy instruction reference.

### Requirement 12: Audit Report Generation

**User Story:** As a solo founder, I want one versioned report that states alignment, gaps, gate states, and readiness, so that the audit result is reviewable and comparable across runs.

#### Acceptance Criteria

1. WHEN an Audit_Run completes, THE Report_Writer SHALL produce one Audit_Report containing an alignment summary, a readiness gap matrix, a Finding table, and a Pipeline_Gate state table.
2. THE Report_Writer SHALL begin each Audit_Report with a YAML frontmatter block containing `title`, `doc_type`, `version`, `date`, and `lang`.
3. THE Report_Writer SHALL record in each Audit_Report the input revision identifier of the Guideline_Set and the input revision identifier of the Runtime_Repository.
4. THE Report_Writer SHALL order Finding entries by severity and then by Finding_Type.
5. THE Report_Writer SHALL record in the readiness gap matrix one row per audited capability containing the current Readiness_Level, the gap statement, the priority, and the exit criterion expressed as a Verifiable_Completion_Condition.
6. THE Report_Writer SHALL express each remediation statement as a documentation change, a specification change, or a locally reproducible check.
7. THE Report_Writer SHALL apply semantic versioning to each published Audit_Report and retain every prior Audit_Report version.
8. THE Report_Writer SHALL quote each scalar containing reserved punctuation in the Audit_Report frontmatter.

### Requirement 13: Audit Determinism And Robustness

**User Story:** As a maintainer, I want repeated audits over unchanged inputs to produce identical findings, so that the report is trustworthy as a regression signal.

#### Acceptance Criteria

1. WHEN THE Alignment_Auditor processes one input set twice and that input set is unchanged between runs, THE Alignment_Auditor SHALL produce two identical Finding sets.
2. WHEN THE Alignment_Auditor processes the audited documents of one input set in any order, THE Alignment_Auditor SHALL produce one identical Finding set.
3. WHEN one audited document is added to an input set and every other input is unchanged, THE Alignment_Auditor SHALL produce a Finding set that contains every Finding produced for the unchanged documents.
4. THE Alignment_Auditor SHALL produce a Finding count that is less than or equal to the sum of the Normative_Element count and the Artifact_Index entry count.
5. IF an audited document is malformed, THEN THE Alignment_Auditor SHALL record a Finding of type `malformed-document` naming the document and complete the Audit_Run.
6. IF an audited document is unreadable, THEN THE Alignment_Auditor SHALL record a Finding of type `unreadable-input` naming the document and complete the Audit_Run.
7. WHEN an Audit_Run completes, THE Alignment_Auditor SHALL report the count of audited documents, the count of Findings, and the elapsed run time.

## Candidate Correctness Properties

Non-normative. These properties are traced to the acceptance criteria above and are carried forward into the design phase as property-based test candidates.

| Property | Class | Source criteria |
|---|---|---|
| Parsing a printed Guideline_Model returns an equal Guideline_Model | Round trip | 2.4, 2.5 |
| Parsing a printed Artifact_Index returns an equal Artifact_Index | Round trip | 3.5, 3.6 |
| Every Artifact_Bearing_Element is either linked or reported as a Finding | Invariant (closure) | 4.1, 4.3, 4.8 |
| Every `runtime-ready` claim carries at least one Verifiable_Completion_Condition | Invariant | 4.6, 6.5 |
| Adding evidence never lowers an assigned Readiness_Level | Metamorphic (monotonicity) | 6.7 |
| A second audit over unchanged inputs produces an identical Finding set | Idempotence | 12.7, 13.1 |
| Document processing order does not change the Finding set | Confluence | 13.2 |
| Adding one document preserves every Finding for unchanged documents | Metamorphic | 13.3 |
| Finding count stays bounded by element count plus index entry count | Metamorphic bound | 13.4 |
| Renaming a containing directory does not change the Finding set | Invariant (agnosticity) | 3.3, 8.3, 8.6 |
| Malformed and unreadable inputs yield typed Findings and a completed run | Error condition | 13.5, 13.6 |
| No file outside the Audit_Output_Directory changes during a run | Invariant (read-only) | 1.1, 1.2, 1.3 |
| Deploy_Boundary stays `closed` without an explicit operator instruction | Invariant (safety) | 1.5, 11.8 |
