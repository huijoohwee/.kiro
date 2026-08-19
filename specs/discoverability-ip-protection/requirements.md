# Requirements Document

## Introduction

This feature defines a single, auditable policy for balancing **discoverability** (AI agents, search engines, and human collaborators can find and correctly interpret the work) against **IP protection** (core source, orchestration internals, and secrets are not exposed) across the cross-repository estate.

The estate already carries both halves of the tension in production form. The publish repository serves `robots.txt`, `sitemap.xml`, `llms.txt`, `openapi.json`, `auth.md`, `.well-known/api-catalog`, and an MCP transport route, while the intended artifact boundary publishes bundled build output under `knowgrph/assets/` and an explicit allowlist of `grph-shared/dist/*`. The implemented audit currently detects additional tracked source and unclassified paths in that public estate, so the boundary is blocked rather than assumed. The invocation dictionaries in the Agentic Canvas OS docs already carry `publish_policy: "Dev-only until explicit operator approval"` on a per-document basis. The normative classification now exists locally; its remaining job is to keep publication fail-closed until the public estate conforms.

The inbound interface is settled: the Prod_Repository (`huijoohwee`) is the sole public origin serving `airvio.co` and the sole inbound interface for external agents and users, and MCP is the only inbound protocol boundary. Two MCP endpoints are already deployed — a public read endpoint carrying read-only tools, and a control-plane endpoint carrying every approval-gated and spend-bearing capability. The `/`, `@`, and `#` grammars are published as Invocation_Registry metadata only and execute through an app-owned forwarder that targets the control-plane endpoint, so they add no second inbound authenticated protocol.

Scope of this specification:

1. A four-tier **surface taxonomy** covering every publishable artifact, with fail-closed defaults.
2. **Discoverability mechanisms** for search engines, human collaborators, and AI agents (sitemap, robots, structured data, `llms.txt`, service description, agent card, MCP manifest, and the `/`, `@`, `#` invocation registries).
3. **IP protection mechanisms**: source withholding, artifact-only publishing, secret and config hygiene, and license posture consistent with the FOSS lean.
4. **Reconciliation rules** that resolve conflicts between the two goals deterministically, with protection as the default winner.
5. Conformance of the combined PRD/TAD document at `huijoohwee.github.io/docs/documents/hjh-discoverability-ip-protection-prd-tad.md` to `huijoohwee.github.io/guidelines/prd-tad-guidelines.md`. The document is authored and has an executable Site_Repository conformance check; readiness claims must remain aligned with the current gate evidence.

This specification governs the local policy runtime and its documentation. It authorises no promotion to the production repository, edge deployment, or mutation of the public-origin working tree.

## Glossary

- **Surface_Registry**: The single source of truth that assigns exactly one Surface_Tier to every publishable artifact in the estate, with per-artifact license id and publish policy.
- **Surface_Tier**: One of four values — `public-discoverable`, `public-artifact`, `gated`, `private`.
  - `public-discoverable`: Human- and agent-readable descriptive content indexed by search engines and listed in agent discovery files (marketing copy, published documents, capability descriptions, API descriptions).
  - `public-artifact`: Executable or renderable build output served to browsers without accompanying source (bundled JavaScript, CSS, static assets, allowlisted `dist` modules).
  - `gated`: Reachable at a stable address but requiring an authorisation credential or operator approval before content or execution is returned (spend-bearing tools, control-plane routes, private workspace content).
  - `private`: Never published to any public origin or public repository (application source modules, prompt internals, orchestration wiring, secrets, unpublished specifications, local scratch).
- **Publication_Gate**: The decision component that evaluates an artifact against the Surface_Registry and either permits or blocks publication, and that resolves Discoverability_Protection_Conflict cases.
- **Discovery_Surface**: The generated set of machine-readable discovery files served from a public origin: `robots.txt`, `sitemap.xml`, `llms.txt`, structured data blocks, service description, agent card, and `.well-known` entries.
- **Invocation_Registry**: The published catalog of invocable tokens across the four invocable surfaces — MCP tool ids, `/` command tokens, `@` binding tokens, and `#` semantic tokens — expressed as metadata only.
- **IP_Boundary**: The rule set that determines which representation of an artifact crosses from the development repository to a public origin: descriptive metadata, build artifact, or nothing.
- **Secret_Hygiene_Check**: The pre-publication scan that rejects candidate publications containing credential material, private endpoints, signed URLs, or local absolute paths.
- **License_Registry**: The mapping from Surface_Tier and artifact class to a declared license identifier and reuse grant.
- **Discoverability_Protection_Conflict**: A case in which making an artifact discoverable would expose material classified as `private` or `gated`.
- **Metadata_Stub**: A `public-discoverable` descriptive record that names, summarises, and addresses a non-public capability without exposing its implementation, inputs, or internal wiring.
- **Dev_Repository**: The development source repository (`knowgrph`), which holds application source and is the authoring origin for all artifacts.
- **Prod_Repository**: The publish repository (`huijoohwee`), whose tracked contents are served at the public origin, and which is the sole inbound interface for external agents and users.
- **Worker_Repository**: The Cloudflare Worker repository (`agentic-canvas-os`), which holds worker source, Durable Object bindings, and the required-secrets list, and which acts as a client of the Control_Plane_MCP_Endpoint.
- **Site_Repository**: The static site repository (`huijoohwee.github.io`), which holds the PRD_TAD_Document and the PRD_TAD_Guidelines.
- **Edge_Runtime**: The Cloudflare-hosted runtime serving `airvio.co` and `airvio.co/knowgrph`.
- **Edge_Route_Include_List**: The enumerated set of routed paths declared in the Edge_Runtime route manifest (`_routes.json`) of the Prod_Repository.
- **Public_Read_MCP_Endpoint**: The unauthenticated MCP endpoint at `https://airvio.co/knowgrph/mcp`, carrying read-only tools only.
- **Control_Plane_MCP_Endpoint**: The credentialed MCP endpoint at `https://airvio.co/knowgrph/control-plane/mcp`, carrying approval-gated, spend-bearing, and orchestration capabilities.
- **Invocation_Forwarder**: The app-owned forwarder that executes `/`, `@`, and `#` tokens by routing to a Control_Plane_MCP_Endpoint tool, and which exposes no inbound protocol boundary of its own.
- **Operator**: The solo developer, who is the sole authority for approving publication and deployment.
- **PRD_TAD_Document**: The combined product and technical architecture document at `huijoohwee.github.io/docs/documents/hjh-discoverability-ip-protection-prd-tad.md`.
- **PRD_TAD_Guidelines**: The authoring contract at `huijoohwee.github.io/guidelines/prd-tad-guidelines.md`.
- **Discovery_Generator**: The component that serialises the Surface_Registry into Discovery_Surface files.
- **Discovery_Parser**: The component that reads a Discovery_Surface file back into a set of entry records.
- **Operator_Instruction**: A recorded authorisation stating an instruction identifier, every authorised artifact identifier, exactly one authorised destination (Prod_Repository or Edge_Runtime), and the instruction timestamp.
- **Override_Record**: An Operator-authored record naming exactly one Discoverability_Protection_Conflict identifier, the override author, the scope, and the justification.
- **Publish_Allowlist**: The enumerated set of distribution module identifiers permitted to be published as `public-artifact`.

## Requirements

### Requirement 1: Surface Classification Taxonomy

**User Story:** As the Operator, I want every publishable artifact assigned to exactly one surface tier, so that publication decisions are mechanical rather than case-by-case judgement.

#### Acceptance Criteria

1. THE Surface_Registry SHALL assign to each registered artifact exactly one Surface_Tier drawn from the values `private`, `gated`, `public-artifact`, and `public-discoverable`.
2. WHEN the Publication_Gate evaluates an artifact that has no Surface_Registry entry, THE Publication_Gate SHALL treat the artifact as `private` and block publication.
3. WHEN the Publication_Gate evaluates an artifact whose Surface_Registry entry names two or more Surface_Tier values, THE Publication_Gate SHALL block publication and report the conflicting entry identifier.
4. THE Surface_Registry SHALL record, for each artifact, a non-empty value for each of the artifact identifier, the Surface_Tier, the license identifier, the publish policy, and the owning repository.
5. WHEN an artifact classified `private` is contained within a directory classified `public-discoverable`, THE Publication_Gate SHALL apply the artifact-level Surface_Tier and exclude that artifact from publication.
6. THE Surface_Registry SHALL classify application source modules, prompt template internals, orchestration wiring, credential material, and unpublished specifications as `private`.
7. THE Surface_Registry SHALL classify bundled browser build output and distribution modules named in the Publish_Allowlist as `public-artifact`.
8. THE Surface_Registry SHALL classify published documents, capability descriptions, and service descriptions as `public-discoverable`.
9. THE Surface_Registry SHALL classify spend-bearing tool routes and control-plane routes as `gated`.
10. WHEN an artifact matches the criteria of two or more artifact classes named in acceptance criteria 6 through 9, THE Surface_Registry SHALL assign the most restrictive matching Surface_Tier in the order `private`, `gated`, `public-artifact`, `public-discoverable`.
11. IF a Surface_Registry entry assigns an artifact class named in acceptance criterion 6 to any Surface_Tier other than `private`, THEN THE Publication_Gate SHALL reject the Surface_Registry entry and report the rejected artifact identifier and the mandatory Surface_Tier.
12. IF a Surface_Registry entry names a Surface_Tier value outside the four values listed in acceptance criterion 1, THEN THE Publication_Gate SHALL block publication of that artifact and report the artifact identifier and the unrecognised Surface_Tier value.
13. IF a Surface_Registry entry omits, or records an empty value for, any field required by acceptance criterion 4, THEN THE Publication_Gate SHALL block publication of that artifact and report the artifact identifier and the names of the missing fields.
14. THE Surface_Registry SHALL record the Dev_Repository and the Worker_Repository as private-visibility repositories, and SHALL classify the source modules of the Dev_Repository and the Worker_Repository as `private`.
15. THE Surface_Registry SHALL classify the Worker_Repository worker source modules, Durable Object bindings, and required-secrets list as orchestration wiring, and therefore as `private` under acceptance criterion 6.
16. THE Surface_Registry SHALL record the Prod_Repository and the Site_Repository as public-visibility repositories.
17. IF a Surface_Registry entry records the Dev_Repository or the Worker_Repository as a public-visibility repository, THEN THE Publication_Gate SHALL block publication and report the affected repository identifier and the mandatory visibility value.
18. WHERE the Discovery_Surface states a provenance record, THE Discovery_Generator SHALL permit that record to name the Dev_Repository identifier, the Worker_Repository identifier, and a commit revision for each, and SHALL exclude every source module, binding name, and secret name of those repositories from that record.

### Requirement 2: Search Engine and Human Discoverability

**User Story:** As a human collaborator or search engine, I want published work to be indexable and canonically addressed, so that the work can be found without prior knowledge of its location.

#### Acceptance Criteria

1. THE Discovery_Generator SHALL emit a `sitemap.xml` document containing one entry per distinct canonical absolute URL derived from the artifacts classified `public-discoverable` after applying the representing-page substitution in acceptance criteria 8 and 10, containing zero duplicate URL entries, and containing zero entries for artifacts classified `gated` or `private`.
2. THE Discovery_Generator SHALL emit a `robots.txt` document that allows crawling of every `public-discoverable` path, disallows crawling of every `gated` path, and contains zero paths of artifacts classified `private`.
3. WHEN the Discovery_Generator emits `robots.txt`, THE Discovery_Generator SHALL include an absolute sitemap reference and a content-signal declaration that states, for each of search indexing, agent input, and model training, whether that use is permitted or not permitted.
4. THE Discovery_Generator SHALL emit exactly one canonical absolute URL per `public-discoverable` artifact, rooted at the public origin served by the Prod_Repository, and SHALL emit no canonical absolute URL that addresses two or more distinct `public-discoverable` artifacts.
5. THE Discovery_Generator SHALL emit a structured data block for each `public-discoverable` document that states the document title, description, canonical URL, license identifier, and last-modified date.
6. IF a `public-discoverable` artifact has no canonical absolute URL, THEN THE Publication_Gate SHALL block publication of that artifact, report the artifact identifier with an indication that the canonical absolute URL is missing, and leave the previously published state of that artifact unchanged.
7. WHERE an artifact is classified `public-artifact`, THE Discovery_Generator SHALL omit that artifact from `sitemap.xml`.
8. WHERE a `public-discoverable` artifact declares a representing human-readable page in the Surface_Registry, THE Discovery_Generator SHALL list the representing page in `sitemap.xml` in place of the artifact.
9. IF an artifact requires a representing human-readable page under acceptance criterion 8 or acceptance criterion 10 and no representing page is declared in the Surface_Registry, or the declared page path resolves to no published file, THEN THE Publication_Gate SHALL block publication of that artifact and report the declared or missing page path.
10. WHERE an artifact is classified `public-artifact` and the Surface_Registry declares a human-readable page that loads that artifact, THE Discovery_Generator SHALL list that page in `sitemap.xml`.
11. WHERE the Surface_Registry contains zero artifacts classified `public-discoverable`, THE Discovery_Generator SHALL emit a `sitemap.xml` document containing the document header and zero entries.

### Requirement 3: AI Agent Discoverability

**User Story:** As an external AI agent, I want to discover the available capabilities and their contracts from machine-readable files without scraping rendered pages, so that I can select the correct surface on the first attempt.

#### Acceptance Criteria

1. THE Discovery_Generator SHALL emit an `llms.txt` document containing one entry for every artifact classified `public-discoverable` and no entry for any artifact classified `gated`, where each entry states a title of 1 to 80 characters, a single-line summary of 1 to 200 characters containing no line breaks, and a canonical absolute URL.
2. WHERE the Surface_Registry contains zero artifacts classified `public-discoverable`, THE Discovery_Generator SHALL emit an `llms.txt` document containing the document header and zero entries.
3. THE Discovery_Generator SHALL emit a service description document enumerating each publicly reachable route with its method, a request schema reference expressed as a resolvable absolute URL, and a response schema reference expressed as a resolvable absolute URL.
4. THE Discovery_Generator SHALL emit a `.well-known` API catalog entry that references the service description document by absolute URL.
5. THE Discovery_Generator SHALL emit an agent card document stating the agent identifier, the supported transports, the authorisation metadata location, and the trust boundary of each transport.
6. WHEN an agent requests any Discovery_Surface file, THE Edge_Runtime SHALL return the file with zero model invocations and a recorded model cost of 0.00 USD.
7. WHEN an agent requests a Discovery_Surface file, THE Edge_Runtime SHALL respond within 500 ms at the 95th percentile measured at the origin over any rolling 24-hour window containing at least 100 such requests.
8. THE Discovery_Surface SHALL state, for each enumerated transport, exactly one classification from the set `public-discoverable`, `public-artifact`, `gated`.
9. IF an agent requests a `gated` route without a valid authorisation credential, THEN THE Edge_Runtime SHALL return an authorisation error and a pointer to the authorisation metadata location, SHALL invoke zero models, and SHALL exclude every portion of the gated route's response payload from the returned response.
10. WHEN the classification of any artifact in the Surface_Registry changes, THE Discovery_Generator SHALL regenerate every Discovery_Surface file so that each emitted file reflects the changed classification.
11. IF an agent requests a Discovery_Surface file that the Surface_Registry does not contain, THEN THE Edge_Runtime SHALL return a not-found error and SHALL invoke zero models.
12. IF an artifact classified `public-discoverable` lacks a title, a summary, or a canonical absolute URL, THEN THE Discovery_Generator SHALL omit that artifact from `llms.txt`, emit the document containing the remaining valid entries, and report a generation error identifying the omitted artifact.
13. THE Surface_Registry SHALL classify the Public_Read_MCP_Endpoint as `public-discoverable`.
14. THE Surface_Registry SHALL classify the Control_Plane_MCP_Endpoint as `gated`.
15. THE Discovery_Generator SHALL publish on the Public_Read_MCP_Endpoint only tools that the Surface_Registry records as read-only.
16. THE Surface_Registry SHALL record the Control_Plane_MCP_Endpoint as the sole execution route of every spend-bearing capability, including media generation and video generation.
17. IF a tool declared for publication on the Public_Read_MCP_Endpoint is recorded as other than read-only, THEN THE Publication_Gate SHALL block publication and report the tool identifier and the Control_Plane_MCP_Endpoint as the required execution route.
18. IF a spend-bearing capability declares an execution route other than the Control_Plane_MCP_Endpoint, THEN THE Publication_Gate SHALL block publication and report the capability identifier and the declared route.
19. THE Discovery_Generator SHALL emit MCP as the only inbound protocol boundary of the Prod_Repository in the agent card document required by acceptance criterion 5.

### Requirement 4: Invocation Registry Publication

**User Story:** As an external AI agent, I want the invocable tokens across MCP, `/`, `@`, and `#` surfaces published as metadata, so that I can discover what is callable without access to the implementations.

#### Acceptance Criteria

1. THE Invocation_Registry SHALL publish, for each invocable token, the token string, the prefix role drawn from exactly one of the four invocable surfaces (MCP tool id, `/` command token, `@` binding token, `#` semantic token), a display label of 1 to 60 characters, a one-line intent summary of 1 to 120 characters containing no line breaks, and exactly one declared Surface_Tier value for the token's execution route.
2. THE Invocation_Registry SHALL withhold implementation owner file paths, internal module identifiers, and prompt template bodies from every published entry.
3. WHILE the Invocation_Registry is assembling the published catalog, IF a token's source document declares a dev-only publish policy and carries no recorded Operator approval, THEN THE Publication_Gate SHALL exclude that token and all of its metadata from the published catalog and from the input to the catalog digest.
4. THE Invocation_Registry SHALL publish exactly one catalog digest computed over the full set of published entries after all exclusions, such that any two assemblies of an identical published entry set produce an identical digest value regardless of the order in which entries were assembled.
5. IF the published catalog digest differs from the digest recomputed from the Surface_Registry, THEN THE Publication_Gate SHALL block publication, SHALL report both digest values, and SHALL leave the previously published catalog unchanged.
6. THE Invocation_Registry SHALL retain exactly one entry per token string, comparing token strings exactly and case-sensitively after trimming leading and trailing whitespace, and SHALL name every contributing source catalog for each retained entry.
7. IF a contributing source catalog returns no response within 10 seconds, THEN THE Invocation_Registry SHALL treat that catalog as unreachable, SHALL emit the catalog assembled from the reachable sources, and SHALL name each unreachable catalog in an explicit unreachable-source field.
8. IF a candidate entry omits the token string, the prefix role, the display label, the intent summary, or the declared Surface_Tier value, or exceeds a stated length bound, THEN THE Publication_Gate SHALL exclude that entry from the published catalog and SHALL report a validation failure identifying the token string and the missing or out-of-bound field name.
9. THE Invocation_Registry SHALL publish each `/` command token, each `@` binding token, and each `#` semantic token as metadata carrying zero directly invocable public endpoint addresses.
10. THE Invocation_Registry SHALL declare, for each `/` command token, `@` binding token, and `#` semantic token, the execution route as the Invocation_Forwarder targeting a Control_Plane_MCP_Endpoint tool.
11. IF a candidate `/`, `@`, or `#` token entry states a directly invocable public endpoint address, or declares an execution route other than the Invocation_Forwarder, THEN THE Publication_Gate SHALL exclude that entry from the published catalog and SHALL report the token string and the declared route.

### Requirement 5: Source Withholding and Artifact-Only Publishing

**User Story:** As the Operator, I want the public origin to serve only build output and descriptions, so that core implementation remains unavailable for direct reuse.

#### Acceptance Criteria

1. THE IP_Boundary SHALL permit publication of build output and descriptive metadata for artifacts classified `public-artifact` and `public-discoverable`.
2. IF a publication candidate contains an application source module classified `private`, THEN THE Publication_Gate SHALL block publication of that candidate and report the `private` source path in its blocking report.
3. IF a publication candidate contains a source map that references `private` source content, THEN THE Publication_Gate SHALL block publication of that candidate and report the source map path in its blocking report.
4. THE IP_Boundary SHALL permit publication of a distribution module only where that module identifier appears in the Publish_Allowlist and every other criterion in this requirement permits publication.
5. IF the Publish_Allowlist and the Surface_Registry disagree about a module identifier, THEN THE Publication_Gate SHALL block publication and report every disagreeing identifier in its blocking report.
6. WHEN the Publish_Allowlist and the Surface_Registry agree about a module identifier, THE Publication_Gate SHALL omit that identifier from the disagreement report.
7. THE Prod_Repository SHALL contain zero tracked files whose Surface_Tier is `private`, where the Surface_Tier of each tracked path is determined by lookup of that path in the Surface_Registry.
8. WHERE a capability is classified `gated` or `private`, THE Publication_Gate SHALL permit a Metadata_Stub describing that capability to be published as `public-discoverable` only where that Metadata_Stub contains no source content and no `private` source path.
9. WHEN the Publication_Gate evaluates a publication candidate, THE Publication_Gate SHALL resolve the Surface_Tier of every tracked path in that candidate against the Surface_Registry before permitting publication.
10. IF a tracked path in a publication candidate has no Surface_Tier entry in the Surface_Registry, THEN THE Publication_Gate SHALL block publication and report that unclassified path in its blocking report.
11. IF THE Publication_Gate blocks publication, THEN THE Publication_Gate SHALL leave the Prod_Repository tracked contents identical to their state immediately before the blocked publication attempt.

### Requirement 6: Secret and Configuration Hygiene

**User Story:** As the Operator, I want publication to fail before any credential or private endpoint reaches a public origin, so that a publish action cannot become a disclosure event.

#### Acceptance Criteria

1. WHEN the Publication_Gate evaluates a publication candidate, THE Secret_Hygiene_Check SHALL scan every file in the publication candidate set, before any candidate file is transmitted to the public origin, for all four match categories: credential material, signed access URLs, private host names, and local absolute filesystem paths.
2. IF the Secret_Hygiene_Check detects a match in any of the four match categories in a publication candidate, THEN THE Publication_Gate SHALL block publication, report the file path and the match category for every match, and omit the matched value and any substring of the matched value from the report.
3. WHEN the Secret_Hygiene_Check completes with zero matches, THE Publication_Gate SHALL record the scan result, the candidate file count, and the scan timestamp.
4. IF the Secret_Hygiene_Check does not return a scan result for every file in the publication candidate set, or does not return a scan result within 300 seconds of scan start, THEN THE Publication_Gate SHALL treat the scan outcome as a detection, block publication, and report the cause of the incomplete scan.
5. IF the Secret_Hygiene_Check completes with zero matches and any other criterion in this specification blocks the candidate, THEN THE Publication_Gate SHALL block publication and report every blocking criterion identifier.
6. THE Surface_Registry SHALL classify runtime configuration values, provider keys, and local convenience files as `private`.
7. IF a file or value in the publication candidate set has no classification in the Surface_Registry, THEN THE Publication_Gate SHALL treat that file or value as `private`, block publication, and report the unclassified file path.
8. WHERE a published document requires a runtime endpoint value, THE Discovery_Generator SHALL emit a placeholder token containing no substring of the endpoint value in place of that endpoint value.
9. IF the Discovery_Generator cannot emit a placeholder token for a required runtime endpoint value, THEN THE Publication_Gate SHALL block publication and report the affected document path and field name.
10. WHEN the Publication_Gate blocks publication, THE Publication_Gate SHALL leave the content of the public origin unchanged from its state before the evaluation started.

### Requirement 7: License Posture

**User Story:** As an external collaborator, I want each published artifact to carry an explicit license, so that permitted reuse is unambiguous and the FOSS position is legible.

#### Acceptance Criteria

1. THE License_Registry SHALL assign exactly one license identifier to each Surface_Tier and artifact class combination present in the Surface_Registry.
2. WHEN an artifact classified `public-discoverable` or `public-artifact` is registered in the Surface_Registry with no license identifier, THE Surface_Registry SHALL reject the registration, retain any previously recorded entry for that artifact identifier unchanged, and report the unlicensed artifact identifier before the artifact reaches Publication_Gate evaluation.
3. WHEN the Publication_Gate evaluates a `public-discoverable` or `public-artifact` artifact with no license identifier, THE Publication_Gate SHALL block publication and report the unlicensed artifact identifier.
4. THE Prod_Repository SHALL contain a license declaration file at its root stating the license identifier for each published artifact class.
5. THE Discovery_Generator SHALL emit the license identifier of each `public-discoverable` artifact in that artifact's structured data block.
6. WHERE an artifact class is published under a permissive open-source license, THE License_Registry SHALL record the license identifier, the covered artifact class, and the excluded artifact classes.
7. WHERE an artifact class is published as a viewable build artifact without a reuse grant, THE License_Registry SHALL record an explicit no-reuse-grant identifier for that class.
8. THE License_Registry SHALL assign exactly one permissive open-source license identifier per artifact class, drawn from the identifiers recorded for the permissive-license category, to published documents, guidelines, specifications, and machine-readable metadata artifact classes.
9. THE License_Registry SHALL assign the single no-reuse-grant identifier recorded for the no-reuse-grant category to bundled browser build output and Publish_Allowlist distribution modules.
10. THE License_Registry SHALL record each licensed artifact class in exactly one of the permissive-license category and the no-reuse-grant category.
11. IF a licensed artifact class appears in neither the permissive-license category nor the no-reuse-grant category, THEN THE Publication_Gate SHALL block publication of that artifact class and report the uncategorised class identifier.
12. IF a licensed artifact class appears in both the permissive-license category and the no-reuse-grant category, THEN THE Publication_Gate SHALL block publication of that artifact class and report the double-categorised class identifier and both recorded license identifiers.
13. IF the license declaration file at the root of the Prod_Repository omits a published artifact class recorded in the License_Registry, THEN THE Publication_Gate SHALL block publication and report the omitted artifact class identifier.
14. IF the license declaration file at the root of the Prod_Repository states a license identifier for a published artifact class that differs from the identifier recorded in the License_Registry for that class, THEN THE Publication_Gate SHALL block publication and report the artifact class identifier and both license identifiers.
15. THE License_Registry SHALL record the prose artifact classes as published documents, guidelines, and specifications, and SHALL assign the license identifier `CC-BY-4.0` to each prose artifact class.
16. THE License_Registry SHALL record the machine-readable metadata artifact classes as invocation dictionary metadata, the service description document (`openapi.json`), the MCP manifest (`.well-known/mcp.json`), the agent card document, the API catalog entry, and `llms.txt`, and SHALL assign the license identifier `Apache-2.0` to each machine-readable metadata artifact class.
17. THE License_Registry SHALL record the no-reuse-grant identifier assigned under acceptance criterion 9 as an SPDX `LicenseRef-` identifier.
18. IF a prose artifact class is assigned a license identifier other than `CC-BY-4.0`, or a machine-readable metadata artifact class is assigned a license identifier other than `Apache-2.0`, THEN THE Publication_Gate SHALL block publication of that artifact class and report the artifact class identifier, the assigned license identifier, and the mandatory license identifier.

### Requirement 8: Conflict Reconciliation

**User Story:** As the Operator, I want conflicts between discoverability and protection resolved by a stated precedence rule, so that ambiguous cases resolve the same way every time.

#### Acceptance Criteria

1. WHEN the Publication_Gate detects a Discoverability_Protection_Conflict, THE Publication_Gate SHALL resolve the conflict in favour of protection and SHALL block publication of the conflicting representation.
2. WHEN the Publication_Gate blocks a representation under a Discoverability_Protection_Conflict, THE Publication_Gate SHALL emit exactly one Metadata_Stub for the affected capability in place of the blocked representation.
3. THE Metadata_Stub SHALL contain the capability name, a single-line summary of at most 200 characters, the declared Surface_Tier, and exactly one contact or authorisation route stated as a canonical absolute URL.
4. THE Metadata_Stub SHALL withhold input schemas, output schemas, implementation paths, and prompt bodies for capabilities classified `private` or `gated`.
5. WHEN the Publication_Gate resolves a Discoverability_Protection_Conflict, THE Publication_Gate SHALL record the conflict identifier, the affected artifact identifier, the resolution outcome stated as exactly one of blocked-with-stub, blocked-without-stub, and override-applied, and the resolution timestamp in UTC.
6. WHERE the Operator records an Override_Record that names exactly one Discoverability_Protection_Conflict identifier, THE Publication_Gate SHALL apply the override to that conflict only and SHALL record the override author, the scope, and the justification.
7. IF any Metadata_Stub field required by acceptance criterion 3 is absent from the Surface_Registry for the affected capability, THEN THE Publication_Gate SHALL emit no Metadata_Stub, keep the conflicting representation blocked, and report the affected capability identifier and each missing field.
8. IF an applied Override_Record would publish material classified `private`, THEN THE Publication_Gate SHALL reject the override, keep the conflicting representation blocked, and report the conflict identifier and the rejected override author.
9. IF an Override_Record omits the conflict identifier, the override author, the scope, or the justification, THEN THE Publication_Gate SHALL reject the override, resolve the conflict in favour of protection, and report each missing field.

### Requirement 9: Publication and Deployment Gating

**User Story:** As the Operator, I want publication to the production repository and the edge runtime to require my explicit instruction, so that no automated step can widen the public surface without my decision.

#### Acceptance Criteria

1. THE Publication_Gate SHALL promote an artifact from the Dev_Repository to the Prod_Repository only where a recorded Operator_Instruction authorises that artifact identifier, names the Prod_Repository as the destination, and carries a timestamp earlier than the promotion attempt.
2. THE Publication_Gate SHALL deploy an artifact to the Edge_Runtime only where a recorded Operator_Instruction authorises that artifact identifier, names the Edge_Runtime as the destination, and carries a timestamp earlier than the deployment attempt.
3. WHILE a document declares a dev-only publish policy and carries no recorded Operator approval, THE Publication_Gate SHALL restrict that document to the Dev_Repository and SHALL block every promotion of that document.
4. WHEN the Publication_Gate promotes an artifact, THE Publication_Gate SHALL record the artifact identifier, the source path, the destination path, the approving Operator_Instruction, and the promotion timestamp.
5. IF a publication or deployment is attempted with no recorded Operator_Instruction, THEN THE Publication_Gate SHALL block the attempt, report the missing approval, and leave the Prod_Repository tracked contents and the Edge_Runtime served contents unchanged.
6. IF promotion to the Prod_Repository and deployment to the Edge_Runtime are attempted together with no recorded Operator_Instruction, THEN THE Publication_Gate SHALL block both the promotion and the deployment.
7. THE Publication_Gate SHALL perform classification, scanning, and reporting on the Operator's local machine and SHALL issue zero outbound network requests during classification, scanning, and reporting.
8. THE Publication_Gate SHALL treat an Operator_Instruction as recorded only where that instruction states an instruction identifier, every authorised artifact identifier, exactly one authorised destination selected from the Prod_Repository and the Edge_Runtime, and the instruction timestamp.
9. IF a promotion or deployment attempt names an artifact identifier or a destination that the recorded Operator_Instruction does not authorise, THEN THE Publication_Gate SHALL block that attempt and report the unauthorised artifact identifier and the unauthorised destination.
10. IF the Publication_Gate cannot write the promotion record named in acceptance criterion 4, THEN THE Publication_Gate SHALL block the promotion, report the failed record write, and leave the Prod_Repository tracked contents unchanged.

### Requirement 10: Discovery Artifact Generation Fidelity

**User Story:** As the Operator, I want the discovery files to be an exact, reproducible projection of the surface registry, so that drift between what is registered and what is advertised is detectable.

#### Acceptance Criteria

1. WHEN the Discovery_Generator runs, THE Discovery_Generator SHALL derive every field of every emitted Discovery_Surface file from the Surface_Registry and SHALL read zero inputs other than the Surface_Registry.
2. WHEN the Discovery_Parser parses a Discovery_Surface file that lists entries, THE Discovery_Parser SHALL produce one entry record per listed entry containing the entry identifier, the canonical absolute URL, and the summary, and SHALL produce an empty summary value for entry records read from a Discovery_Surface file whose format carries no summary field.
3. FOR ALL Surface_Registry states, spanning at least 100 generated states with registered artifact counts from 0 to 1,000 inclusive, parsing each generated entry-listing Discovery_Surface file SHALL produce an entry set whose entry identifiers equal, as an unordered set comparison, the entry identifiers of the artifacts classified `public-discoverable` in the Surface_Registry that the file is specified to list, with zero additional and zero missing entry identifiers (round-trip property).
4. WHEN the Discovery_Generator runs twice against a Surface_Registry in which zero entries were added, removed, or modified between the two runs, THE Discovery_Generator SHALL produce the identical set of Discovery_Surface file names and byte-identical content for each of those files on both runs (idempotence property).
5. FOR ALL Surface_Registry states, spanning at least 100 generated states with registered artifact counts from 0 to 1,000 inclusive, the entry count of each generated Discovery_Surface file SHALL be less than or equal to the total registered artifact count, and adding one artifact classified `public-discoverable` to the Surface_Registry SHALL not decrease the entry count of any generated Discovery_Surface file (metamorphic property).
6. IF the Discovery_Parser encounters a Discovery_Surface file that it cannot parse into entry records under acceptance criterion 2, because a required entry field is absent or the file violates the syntax of its declared format, THEN THE Discovery_Parser SHALL return a parse error naming the file and the 1-based line number of the first offending line, SHALL return zero entry records for that file, and SHALL leave the file unmodified.
7. WHEN a Discovery_Surface file lists an entry that resolves to an artifact classified `private` or `gated`, THE Publication_Gate SHALL block publication of every Discovery_Surface file in the candidate and report the leaked entry identifier and its recorded Surface_Tier.
8. IF the entry identifiers parsed from a Discovery_Surface file in a publication candidate differ from the entry identifiers of the artifacts classified `public-discoverable` that the file is specified to list, THEN THE Publication_Gate SHALL block publication and report each entry identifier present in the file but absent from the Surface_Registry and each entry identifier present in the Surface_Registry but absent from the file.
9. IF the Discovery_Parser returns a parse error for any Discovery_Surface file in a publication candidate, THEN THE Publication_Gate SHALL block publication of that candidate and report the file name and the reported line number.

### Requirement 11: PRD/TAD Document Conformance

**User Story:** As the Operator, I want the combined PRD/TAD document authored against the guidelines contract, so that the policy is captured in the estate's standard documentation form and can drive implementation.

#### Acceptance Criteria

1. THE PRD_TAD_Document SHALL open with a YAML frontmatter block as the first block in the file, stating the document title, document type, version, date, language, and the frontmatter contract field required by the PRD_TAD_Guidelines, with every scalar value containing reserved punctuation quoted so that a strict YAML parser reads the block without error.
2. THE PRD_TAD_Document SHALL contain a product requirements section with a problem statement, personas, user journey stages, user stories in role-capability-benefit form each anchored to one named user journey stage, Given-When-Then acceptance criteria, MoSCoW prioritisation carrying one ROI score per Must-tier and Should-tier item, a minimum-viable scope statement, out-of-scope items, dependencies, and open questions.
3. THE PRD_TAD_Document SHALL contain a success metrics table with named rows for time-to-value steps, time-to-value elapsed time, token cost per month, monthly total cost of ownership, and ROI score, each row stating a baseline value, a target value, and a timeline, with the time-to-value rows stating a maximum step count and a maximum elapsed time in minutes.
4. THE PRD_TAD_Document SHALL contain a technical architecture section with a journey-to-system mapping, a topology table carrying a version identifier and a version note, stating per node its role, its connection type, and, for every storage node, its data residency, with every topology node mapped to one component specification, plus component specifications assigning exactly one responsibility per component, integration contracts, quality attribute scenarios, a deployment strategy, and a component inventory table.
5. THE PRD_TAD_Document SHALL render every diagram of more than five nodes in Mermaid syntax, with the topology diagram grouped into one subgraph per stated boundary.
6. THE PRD_TAD_Document SHALL contain one architectural decision record per decision that selects a dependency, an infrastructure component, or a deployment model for any component named in the component inventory, each record stating status, date, context, decision, alternatives including at least one open-source alternative, rationale, a total-cost-of-ownership comparison with one column per deployment model offered by each candidate and an ops-burden rating per column, and consequences.
7. THE PRD_TAD_Document SHALL contain at least one architectural decision record.
8. THE PRD_TAD_Document SHALL express each acceptance criterion as a Verifiable Completion Condition stating one measurable end state, the check that demonstrates that end state in the executing agent's own surfaced output, and the constraint that must hold, and SHALL record each condition in the component specification of the component that implements it.
9. THE PRD_TAD_Document SHALL name each agent-platform readiness dimension defined by the PRD_TAD_Guidelines as either in scope or explicitly excluded, assign each in-scope dimension exactly one readiness tier of Must, Follow-on, or Won't, and state the execution order across tiers.
10. THE PRD_TAD_Document SHALL establish bidirectional traceability from each requirement identifier in this specification to a component identifier, an interface identifier, and a Verifiable Completion Condition, using the traceability pattern defined by the PRD_TAD_Guidelines.
11. THE PRD_TAD_Document SHALL place the product requirements content and the technical architecture content in separate top-level sections, with the product requirements section ending at its acceptance criteria and every component, interface, protocol, and deployment statement confined to the technical architecture section.
12. WHEN the PRD_TAD_Document is validated against the PRD_TAD_Guidelines pre-implementation checklist, THE PRD_TAD_Document SHALL record every checklist item as either satisfied with a stated location in the document or not applicable with a stated reason, leaving no checklist item unmarked.
13. IF an in-scope agent-platform readiness dimension is not runtime-ready when the document version is stamped, THEN THE PRD_TAD_Document SHALL contain a readiness gap matrix row for that dimension stating its current state, its gap, its priority, and its exit criteria as a Verifiable Completion Condition, reporting local status and deployed status as separate values.
14. IF an open question remains unresolved when the document version is stamped, THEN THE PRD_TAD_Document SHALL retain that question in the open questions section with a named owner and a tracking status.
15. IF a requirement identifier in this specification has no corresponding component identifier or Verifiable Completion Condition, THEN THE PRD_TAD_Document SHALL list that identifier in the traceability record as unmapped with a stated reason.

### Requirement 12: Verification and Audit

**User Story:** As the Operator, I want a single local check that proves the estate matches the declared policy, so that I can confirm compliance before any publication decision.

#### Acceptance Criteria

1. WHEN the Operator invokes the audit, THE Publication_Gate SHALL emit an audit report containing exactly one entry per registered artifact, each entry naming the artifact identifier, its Surface_Tier, its license identifier, and its publication state as either permitted or blocked.
2. WHEN the audit report is generated, THE Publication_Gate SHALL report a count of artifacts for each of the four Surface_Tier values `public-discoverable`, `public-artifact`, `gated`, and `private`, reporting a count of zero for any tier holding no artifact, and SHALL report the count of blocked publication candidates.
3. WHEN the audit completes and every registered artifact is located inside the repository permitted by its Surface_Tier, THE Publication_Gate SHALL exit with a success status.
4. IF any registered artifact is located outside the repository permitted by its Surface_Tier, THEN THE Publication_Gate SHALL exit with a failure status irrespective of any warning raised under acceptance criterion 5, and SHALL report the artifact identifier, the permitted repository, and the containing repository.
5. WHEN a registered artifact is located inside the repository permitted by its Surface_Tier and its recorded tier differs from the tier derived from its artifact class, THE Publication_Gate SHALL report a warning naming the artifact identifier, the recorded Surface_Tier, and the derived Surface_Tier, and SHALL exit with a success status.
6. THE Publication_Gate SHALL complete a full-estate audit of up to 5,000 registered artifacts within 60 seconds of elapsed wall-clock time, measured from Operator invocation to process exit, on the Operator's local machine.
7. WHEN the audit runs, THE Publication_Gate SHALL leave every scanned file unmodified, verified by comparing the per-file digest recorded for each scanned file before the audit with the per-file digest recorded for that same file after the audit.
8. IF the Surface_Registry cannot be read when the audit is invoked, THEN THE Publication_Gate SHALL emit no audit report, SHALL report an error indicating that the Surface_Registry is unreadable, and SHALL exit with a failure status.
9. IF any per-file digest recorded after the audit differs from the digest recorded for that file before the audit, THEN THE Publication_Gate SHALL exit with a failure status and SHALL report the file path of every differing file.
10. IF the audit exceeds 60 seconds of elapsed wall-clock time, THEN THE Publication_Gate SHALL stop the audit, SHALL exit with a failure status, and SHALL report the elapsed time and the count of registered artifacts left unevaluated.

### Requirement 13: Gated Route Inventory

**User Story:** As the Operator, I want every routed path on the public origin to carry a declared Surface_Tier, so that no spend-bearing route is silently crawlable.

#### Acceptance Criteria

1. THE Surface_Registry SHALL classify as `gated` each of the routed paths `/api/llm/responses`, `/api/llm/chat/completions`, `/api/llm/models`, `/api/payments/commerce/x402`, `/knowgrph/control-plane/mcp`, and `/__chat_proxy/*`.
2. THE Surface_Registry SHALL classify as `gated` and SHALL record a rate limit for each of the fetch-on-behalf proxy routes `/api/link-proxy`, `/api/link-preview`, `/api/oembed`, `/__youtube_transcript`, and `/__video_frame`, on the recorded grounds that each of those routes carries both egress cost and server-side request forgery exposure.
3. THE Surface_Registry SHALL classify as `public-discoverable` each of `/knowgrph/mcp`, `/health`, every `.well-known/` entry, `llms.txt`, `sitemap.xml`, `robots.txt`, and `openapi.json`.
4. THE Discovery_Generator SHALL emit in `robots.txt` a disallow directive for every routed path classified `gated`.
5. IF `robots.txt` in a publication candidate omits a disallow directive for a routed path classified `gated`, THEN THE Publication_Gate SHALL block publication and report each omitted routed path.
6. THE Surface_Registry SHALL contain one entry for every path in the Edge_Route_Include_List.
7. IF a path in the Edge_Route_Include_List has no Surface_Registry entry, THEN THE Publication_Gate SHALL treat that path as `private` and block publication, consistent with Requirement 1 acceptance criterion 2, and SHALL report the unclassified routed path.
8. THE Discovery_Generator SHALL emit zero entries for routed paths classified `gated` in `sitemap.xml` and zero entries for routed paths classified `gated` in `llms.txt`.
9. IF `sitemap.xml` or `llms.txt` in a publication candidate lists a routed path classified `gated`, THEN THE Publication_Gate SHALL block publication and report the listed routed path, the containing file name, and the recorded Surface_Tier.
10. IF a routed path classified `gated` under acceptance criterion 2 has no recorded rate limit, THEN THE Publication_Gate SHALL block publication and report the routed path and the missing rate-limit field.

## Assumptions

Repository visibility, the content-signal stance, the inbound agent interface, and the license identifiers are no longer assumptions. Each is settled and recorded in Resolved Decisions below.

1. The existing production discovery files (`robots.txt`, `sitemap.xml`, `llms.txt`, `openapi.json`, `auth.md`, `.well-known/api-catalog`, `.well-known/mcp.json`, `.well-known/agent-card.json`) are the baseline implementation of the Discovery_Surface and are to be governed by the Surface_Registry rather than replaced.
2. The existing `grph-shared/dist` allowlist pattern in the Prod_Repository ignore file is the baseline Publish_Allowlist referenced in Requirement 5.
3. The `_routes.json` include list in the Prod_Repository is the authoritative Edge_Route_Include_List referenced in Requirement 13.
4. The PRD_TAD_Document is authored in the Site_Repository and validated by its repository-owned static conformance check; the Kiro plan records its source/proof boundary.

## Out of Scope

1. Deployment to the Prod_Repository or the Edge_Runtime. This specification defines gating rules only.
2. Changes to pre-existing application or edge request handlers. Local policy scripts, registries, and their focused tests are in scope.
3. Authentication and authorisation implementation for `gated` routes. This specification classifies those routes and requires the pointer to authorisation metadata; the credential mechanism itself is specified elsewhere.
4. Trademark registration, patent filing, and contractual licensing negotiation.
5. Takedown, enforcement, and monitoring of third-party reuse of published artifacts.
6. Search engine ranking optimisation beyond emitting canonical URLs, sitemap entries, and structured data.

## Resolved Decisions

1. **License split — resolved.** Docs and specs open, code closed. Published documents, guidelines, specifications, and machine-readable metadata carry a permissive open-source license; bundled build output and allowlisted distribution modules carry an explicit no-reuse grant. Captured in Requirement 7, criteria 8 and 9. A class belongs to exactly one category (Requirement 7 criterion 10); the permissive category holds one identifier per artifact class rather than one identifier overall.

2. **Model training stance — resolved.** `ai-train=no, search=yes, ai-input=yes` is confirmed as durable policy. Training refusal and agent-input permission are separate axes, so refusing training costs nothing in agent discoverability. Content signals are advisory only; the enforceable half of the posture is the license declaration in Requirement 7.

3. **Repository visibility — resolved.** `knowgrph` (Dev_Repository) is private. `agentic-canvas-os` (Worker_Repository) is private, because worker source, Durable Object wiring, and the required-secrets list are orchestration wiring, which Requirement 1 criterion 6 mandates as `private`. `huijoohwee` (Prod_Repository) and `huijoohwee.github.io` (Site_Repository) are public. The public provenance reference in `.well-known/runtime-readiness.json`, which names both private repositories with commit revisions, is retained deliberately: repository existence is not confidential and the provenance record supports trust. Captured in Requirement 1, criteria 14 through 18.

4. **Inbound agent interface — resolved.** The Prod_Repository `huijoohwee` is the sole inbound interface for external agents and users, and MCP is the only inbound protocol boundary. Two endpoints are already deployed: the public read endpoint `/knowgrph/mcp` is `public-discoverable` and read-only; the control-plane endpoint `/knowgrph/control-plane/mcp` is `gated` and carries every approval-gated, spend-bearing, and orchestration capability, including media and video generation. The `/`, `@`, and `#` grammars are published as Invocation_Registry metadata only and are executed through an app-owned forwarder that routes to the control-plane tool, never through a second inbound authenticated protocol. Captured in Requirement 3, criteria 13 through 19, and Requirement 4, criteria 9 through 11.

5. **License identifiers — resolved.** Three identifiers. `CC-BY-4.0` for prose (published documents, guidelines, specifications), because attribution routes citations back. `Apache-2.0` for machine-readable metadata (invocation dictionaries, `openapi.json`, `mcp.json`, `llms.txt`, agent card, api-catalog), because that content is copied into third-party code where a patent grant and a clean SPDX identifier are the norm and CC-BY attribution terms are impractical. `LicenseRef-airvio-no-reuse-1.0` is the SPDX `LicenseRef-` identifier for bundled browser build output and Publish_Allowlist distribution modules. Captured in Requirement 7, criteria 15 through 18.

6. **Fetch-on-behalf proxy rate limit — resolved.** Every proxy route in Requirement 13 criterion 2 is gated at 20 requests per 60 seconds. The checked Surface_Registry is the operational authority.

## Open Questions

1. **Control-plane credential mechanism.** Whether the Control_Plane_MCP_Endpoint credential mechanism is a bearer token, a signed request, or OAuth is specified elsewhere (see Out of Scope item 3) and must be cross-referenced once located.

## Known Defects

Drift observed in the live estate at the time of this revision. Each entry names the criterion that detects it.

1. **Sitemap advertises unrouted paths.** `sitemap.xml` advertises `/api/storage/source-files`, `/api/storage/llms.txt`, and `/api/storage/content-manifest.json`, none of which appear in the Edge_Route_Include_List. Detected by Requirement 10 criterion 8 and Requirement 2 criterion 9.
2. **Robots disallow list is incomplete.** `robots.txt` disallows only `/api/payments/`, leaving the spend-bearing model routes crawlable, because robots defaults to allow. Detected by Requirement 13, criteria 4 and 5.
3. **No license declaration file.** No license declaration file exists at the root of the Prod_Repository. Detected by Requirement 7 criterion 13.
