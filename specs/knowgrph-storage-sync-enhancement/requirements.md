# Requirements Document

## Introduction

This feature updates and enhances the KnowGrph storage and sync layer for a solo-developer, AI-native, zero-infrastructure, browser-based, mobile-first product built on local-first and offline-first principles. The enhancement grounds itself in the existing storage/sync documentation family (the combined Storage & Sync PRD/TAD and its companion, the storage sync ADRs, the storage schemas and their deferred extensions, the spreadsheet storage surface, and the source-files import workflow).

The goal is to harden and extend five areas without introducing new infrastructure or cost: local-first and offline-first persistence, sync behavior and conflict handling, storage schemas and their extensions, spreadsheet storage, and source-file import. Every enhancement must honor the product constraints: zero incremental total cost of ownership (D1 free tier, FOSS PocketBase, Cloudflare Worker free tier), browser-based and mobile-first delivery, offline-first and local-first authority, token performance and economics (chunk reuse, hash-based dedupe, bounded pull/push), and FOSS-first tooling. No requirement in this document authorizes deployment to Production or mutation of Cloudflare resources; all acceptance criteria are satisfiable within local and Dev runtime scope.

## Glossary

- **Storage_Sync_System**: The overall KnowGrph storage and sync feature comprising the client sync engine, the browser persisted cache, the storage contract, and the (locally verifiable) Worker route contracts.
- **Client_Sync_Engine**: The browser-side sync loop that enqueues mutations, pushes them, pulls remote changes, and manages the cursor (`knowgrphStorageClientSync.ts`).
- **Persisted_Cache**: The bounded browser-local IndexedDB/Dexie working store for documents, chunks, graph snapshots, the sync outbox, and the sync cursor (`knowgrphStorageDb.ts`).
- **Outbox**: The queue of pending local mutations awaiting successful push, retained across reloads and offline periods.
- **Storage_Worker**: The Cloudflare Worker exposing the storage route contract (push, pull, export, doc view, blob, collab save). Referenced for contract shape only; no deployment is authorized here.
- **Conflict_UX**: The shared toast-and-log surface that surfaces and resolves storage conflicts (`ToastHost.tsx`, `HistoryView.tsx`), exposing Keep Local, Accept Remote, and Review Log actions.
- **Source_Files**: The canonical ingest and workspace file surface persisted via IndexedDB (Dexie), acting as a local file-system abstraction.
- **Import_Pipeline**: The source-file import subsystem covering local files, URL sources, GitHub repositories, webpages, websites, and YouTube.
- **Spreadsheet_Surface**: The spreadsheet-like editing and curation surface that projects active graph rows through the Graph Data Table and reuses the shared storage contract.
- **Storage_Schema**: The D1 table set, the browser-local cache shapes, and the shared contract record types.
- **Schema_Extension**: The deferred authenticated-collaboration, chat-relay, and PostgreSQL schema material that is spec-complete but not runtime-ready.
- **Storage_Mode**: The MainPanel `Document Storage & Sync` selection of `Online` or `Offline only`.
- **Content_Hash**: The stable hash recorded per document revision and per chunk used for dedupe and reuse.
- **Canonical_Path**: The workspace-relative path that, with the workspace id, uniquely identifies a document.
- **Token_Budget**: The bounded set of rules that minimize resent bytes and LLM/context token spend (chunk reuse, snapshot reuse, delta application).
- **Git_Engine**: The KnowGrph-owned, browser-and-Worker git read/write engine (an original implementation inspired by isomorphic-git but sharing no source and adding no external runtime dependency) that reads from and writes to a Local_Git_Repository and that fetches from and pushes to a Git_Remote within offline-first constraints.
- **Local_Git_Repository**: The git object store and reference set that the Git_Engine materializes and maintains inside the Persisted_Cache.
- **Git_Remote**: A remote git repository endpoint that the Git_Engine fetches from and pushes to through the Storage_Worker.
- **Git_Operation**: A single Git_Engine action of type clone, fetch, commit, or push that can be queued in the Outbox when offline.
- **File_Sync_Engine**: The KnowGrph-owned multi-provider file synchronization engine (an original implementation inspired by rclone but sharing no source and adding no external runtime dependency or external binary) that performs bidirectional synchronization of files and directories through File_Sync_Provider instances.
- **Provider_Interface**: The KnowGrph-owned standard interface contract that every File_Sync_Provider implements, exposing list, read, write, and delete operations for a storage backend.
- **File_Sync_Provider**: A pluggable implementation of the Provider_Interface for one specific cloud storage backend.
- **Sync_Transfer**: A single File_Sync_Engine transfer of one file in a pull or push direction through a File_Sync_Provider.

## Requirements

### Requirement 1: Local-First and Offline-First Persistence

**User Story:** As a solo developer working across devices and connectivity states, I want my workspace to persist locally and remain fully editable offline, so that I never lose work and can resume from any browser.

#### Acceptance Criteria

1. WHEN a Source_Files change occurs, THE Storage_Sync_System SHALL persist the change to the Persisted_Cache and confirm the durable write before any remote transport is attempted.
2. WHILE the Storage_Mode is `Offline only`, THE Storage_Sync_System SHALL retain all local edits and queued mutations in the Persisted_Cache and Outbox without discarding local work.
3. WHILE the Storage_Mode is `Offline only`, THE Client_Sync_Engine SHALL pause D1 and PocketBase transport while keeping IndexedDB persistence active.
4. WHEN the browser reloads, THE Persisted_Cache SHALL restore documents, chunks, graph snapshots, the Outbox, and the sync cursor to their last committed pre-reload values within 3 seconds of reload.
5. IF a persisted-cache write fails due to a concurrent write conflict, THEN THE Storage_Sync_System SHALL retry the write one time, and IF the retry fails, THEN THE Storage_Sync_System SHALL degrade to the in-memory file system and surface an observable error indication.
6. THE Persisted_Cache SHALL store one raw markdown copy per document revision, SHALL store chunks and graph snapshots as separate records, and SHALL retain at least the 10 most recent revisions per document.
7. WHERE `VITE_KNOWGRPH_STORAGE_WORKSPACE_ID` is unset, THE Storage_Sync_System SHALL use the canonical workspace identity `kgws:canonical-docs` for local persistence.
8. IF IndexedDB initialization fails or the storage quota is exhausted, THEN THE Storage_Sync_System SHALL degrade to the in-memory file system, surface an observable error indication, and preserve in-session edits.
9. IF restore of one record type fails on reload, THEN THE Persisted_Cache SHALL restore all unaffected record types and surface an indication identifying which record type failed to restore.

### Requirement 2: Sync Behavior

**User Story:** As a developer editing on multiple devices, I want bounded, token-economical push and pull sync, so that my workspace state converges across devices without unbounded cost or polling.

#### Acceptance Criteria

1. WHEN an autosave boundary fires and the Storage_Mode is `Online`, THE Client_Sync_Engine SHALL enqueue the document mutation in the Outbox and push it to the push route within a 30-second request timeout.
2. WHEN a push succeeds for a mutation, THE Client_Sync_Engine SHALL remove the corresponding Outbox entry within the same sync cycle.
3. WHEN the Client_Sync_Engine pulls remote changes, THE Client_Sync_Engine SHALL request only records newer than the stored cursor and SHALL apply returned records to the Persisted_Cache.
4. WHILE the Storage_Mode is `Online`, THE Client_Sync_Engine SHALL run the workspace-scoped polling loop at an interval of 120 seconds.
5. IF a push receives a Worker 5xx response, THEN THE Client_Sync_Engine SHALL retain the Outbox entry and SHALL retry using a bounded backoff with a base delay of 1 second doubling on each attempt, capped at 30 seconds, for a maximum of 3 retry attempts.
6. IF a pull request fails, THEN THE Client_Sync_Engine SHALL keep the last stored cursor and SHALL NOT discard any Outbox mutation.
7. WHEN pulled records contain Content_Hash values matching existing local chunks, THE Client_Sync_Engine SHALL reuse the cached chunks instead of re-fetching unchanged chunk bytes.
8. WHEN remote changes are available, THE Client_Sync_Engine SHALL apply pulled deltas rather than reloading the full workspace snapshot.
9. WHEN the Client_Sync_Engine pulls and no changes exist since the cursor, THE Client_Sync_Engine SHALL complete the pull without triggering a Persisted_Cache write.
10. IF a push fails due to a non-5xx transport failure such as a network error or timeout, THEN THE Client_Sync_Engine SHALL retain the Outbox entry and SHALL retry using a bounded backoff with a base delay of 1 second doubling on each attempt, capped at 30 seconds, for a maximum of 3 retry attempts.
11. WHEN the maximum retry attempts are exhausted for a push, THE Client_Sync_Engine SHALL retain the Outbox entry, surface a failure indication through the Conflict_UX, and discard no mutation.

### Requirement 3: Conflict Handling

**User Story:** As a developer whose edits may collide with remote changes, I want conflicts surfaced and resolved through one predictable path, so that I retain control without a separate conflict system.

#### Acceptance Criteria

1. WHEN a push mutation carries a base revision older than the stored server revision, THE Storage_Worker SHALL reject the mutation, preserve the stored server revision unchanged, and return a conflict acknowledgement that identifies the affected record and the current stored server revision.
2. WHEN a conflict acknowledgement is received, THE Conflict_UX SHALL surface the conflict through the shared toast and history log surfaces within 1 second, identifying the affected record by its Canonical_Path and presenting the Keep Local, Accept Remote, and Review Log actions.
3. WHEN a pull returns a server revision greater than or equal to the local Outbox revision for the same record, THE Client_Sync_Engine SHALL remove the stale Outbox conflict entry without user action.
4. WHERE an Outbox conflict entry is not stale, THE Storage_Sync_System SHALL retain the entry until the user selects a resolution action or a later retry succeeds.
5. WHEN the user selects `Keep Local`, THE Storage_Sync_System SHALL re-read the Outbox entry, increment its revision, and retry the push.
6. WHEN the user selects `Accept Remote`, THE Storage_Sync_System SHALL apply the remote record to the Persisted_Cache, align the local revision to the remote server revision, and clear the corresponding Outbox entry only after the Persisted_Cache write succeeds.
7. THE Conflict_UX SHALL expose the Keep Local, Accept Remote, and Review Log actions through the shared action-descriptor runtime and SHALL NOT introduce a separate storage-only modal, drawer, or panel.
8. WHILE two or more collaborators edit the same JSON document in `Online` mode, THE Storage_Sync_System SHALL route edits through CRDT-backed structured controls and SHALL make the raw JSON editor read-only.
9. WHEN a concurrent JSON document is saved without accompanying CRDT state, THE Storage_Worker save bridge SHALL reject the save with a conflict response.
10. IF a `Keep Local` retry push again receives a conflict acknowledgement, THEN THE Storage_Sync_System SHALL retain the Outbox entry, re-surface the conflict through the Conflict_UX, and SHALL NOT automatically retry the push again without a user resolution action.
11. IF the Persisted_Cache write fails while applying `Accept Remote`, THEN THE Storage_Sync_System SHALL retain the Outbox entry unchanged and surface an error through the Conflict_UX indicating the remote record was not applied.

### Requirement 4: Document Write Authority and Save Bridge

**User Story:** As a developer whose documents span product, collaborative, and invocation roots, I want writes routed to the correct GitHub docs root, so that authority stays path-scoped and never drifts to a database-first workflow.

#### Acceptance Criteria

1. WHEN a document save resolves its repository target for a `knowgrph/docs/**` path or a `/docs/workspace-seeds/**` path, THE Storage_Sync_System SHALL select the `knowgrph-docs` repository target.
2. WHEN a document save resolves its repository target for a collaborative workspace path, THE Storage_Sync_System SHALL select the `workspace-docs` repository target.
3. IF a save targets an `agentic-canvas-os/**` path or the duplicate `huijoohwee/docs/workspace-seeds/**` root, THEN THE Storage_Sync_System SHALL reject the write, SHALL return a rejection response identifying the unsupported path, and SHALL NOT perform any partial GitHub or D1 write.
4. WHEN a browser save request is issued, THE Storage_Sync_System SHALL include the resolved repository target, and THE Storage_Worker SHALL re-derive the target from the document path before selecting repository configuration.
5. WHEN an explicit Source_Files cloud upload is performed, THE Storage_Sync_System SHALL write to the resolved GitHub root first, push identical text to D1 only after the GitHub write succeeds, and mark the row cloud-synced only after the public document read-back returns byte-identical text within a maximum of 3 read-back attempts.
6. IF the GitHub write fails during an explicit cloud upload, THEN THE Storage_Sync_System SHALL skip the D1 push and SHALL keep the Source_Files row in a retryable failure state that retains the row text, surfaces a failure indication, and retries up to 3 attempts.
7. IF the public document read-back does not return byte-identical text, THEN THE Storage_Sync_System SHALL keep the row in the retryable failure state and SHALL NOT mark the row cloud-synced.
8. THE Storage_Sync_System SHALL keep GitHub repository credentials confined to the Storage_Worker and SHALL NOT store repository tokens or secrets in browser settings or the Persisted_Cache.

### Requirement 5: Storage Schemas

**User Story:** As a developer maintaining the storage contract, I want the shipped schema baseline to stay stable and token-economical, so that documents, chunks, and snapshots stay consistent across the client and shared store.

#### Acceptance Criteria

1. THE Storage_Schema SHALL identify each document uniquely by the composite key of workspace id and Canonical_Path, where Canonical_Path is a non-empty workspace-relative string of at most 1024 characters.
2. WHEN a document write resolves against an existing workspace-id-and-Canonical_Path pair, THE Storage_Sync_System SHALL upsert the existing row within the same write operation and SHALL preserve that row's identity rather than inserting a duplicate row.
3. WHEN a document revision or a chunk record is written, THE Storage_Schema SHALL record a Content_Hash derived from that record's content, and WHEN the record's content changes, THE Storage_Schema SHALL recompute and update its Content_Hash.
4. THE Storage_Schema SHALL keep the shipped D1 baseline limited to exactly the `workspaces`, `documents`, `document_chunks`, `graph_snapshots`, `sync_devices`, and `sync_events` tables and no additional tables.
5. THE Storage_Schema SHALL preserve the browser-local field names `documentRevision` and `isDeleted` where the remote contract uses `revision` and `deleted`, mapping each browser-local field to its remote counterpart on every read and write across the contract boundary.
6. WHEN a push prunes the audit log, THE Storage_Schema SHALL delete `sync_events` entries whose age exceeds a 24-hour time-to-live window and SHALL retain every `sync_events` entry within that window.
7. WHERE a generated binary artifact is persisted, THE Storage_Schema SHALL store the artifact bytes as an R2 object and SHALL store a sibling markdown manifest as a normal D1 document.
8. IF a document revision or chunk record is submitted without a derivable Content_Hash, THEN THE Storage_Schema SHALL reject the write, SHALL leave any existing row unchanged, and SHALL return an error response indicating the missing Content_Hash.
9. IF a write targets a table outside the shipped D1 baseline set defined in criterion 4, THEN THE Storage_Schema SHALL reject the write and SHALL return an error response indicating the unsupported table.
10. IF the R2 object write for a generated binary artifact fails, THEN THE Storage_Schema SHALL NOT persist the sibling markdown manifest and SHALL leave the artifact in a retryable failure state.

### Requirement 6: Schema Extensions

**User Story:** As a developer planning future collaboration and scale, I want deferred schema extensions kept spec-complete and clearly gated, so that they do not leak into the shipped baseline before their owners exist.

#### Acceptance Criteria

1. THE Storage_Sync_System SHALL treat the authenticated-collaboration, chat-relay, and PostgreSQL Schema_Extension material as deferred and SHALL exclude every Schema_Extension table from the shipped D1 baseline defined in Requirement 5.
2. IF a Schema_Extension lacks any of a named Worker owner, an applied migration, or a passing focused test, THEN THE Storage_Sync_System SHALL NOT expose the extension through an active runtime route and SHALL NOT report the extension as runtime-ready.
3. THE Schema_Extension SHALL exclude repository credentials, local mirror paths, and online/offline preferences from its D1 tables.
4. WHERE the authenticated relay `serverManaged` mode is requested and the workspace provider policy does not explicitly allow server-managed relay, THE Storage_Sync_System SHALL reject the request, SHALL NOT establish the relay, and SHALL return an error indicating server-managed relay is not permitted while retaining the prior configuration.
5. WHERE a BYOK provider key is supplied, THE Storage_Sync_System SHALL treat the key as a per-request browser input, SHALL NOT persist the key to D1 or the Persisted_Cache, and SHALL discard the key after the request completes.
6. WHEN the documentation describes PostgreSQL adoption, THE Schema_Extension SHALL present PostgreSQL as a deferred migration path gated by concurrent multi-user editing, server-side retrieval scale, vector-search runtime need, or tenancy and audit requirements.

### Requirement 7: Spreadsheet Storage

**User Story:** As a developer curating graph data in a spreadsheet view, I want spreadsheet edits to persist through the shared storage contract, so that no spreadsheet-specific database or sync loop is introduced.

#### Acceptance Criteria

1. THE Spreadsheet_Surface SHALL persist authored source through the same Source_Files and Storage_Sync_System contracts used by every other workspace document.
2. THE Spreadsheet_Surface SHALL project the graph rows currently composed in the shared Graph Data Table and SHALL NOT maintain a spreadsheet-specific database, route, or sync loop.
3. WHILE the Storage_Mode is `Offline only`, THE Spreadsheet_Surface SHALL retain IndexedDB state and queued mutations under the shared storage contract such that the retained state is restored after a browser reload.
4. THE Spreadsheet_Surface SHALL keep spreadsheet domain types source-owned in the shared spreadsheet types module.
5. THE Spreadsheet_Surface SHALL NOT request repository credentials from the user.
6. IF a spreadsheet persist through the shared storage contract fails, THEN THE Spreadsheet_Surface SHALL retry the persist one time, and IF the retry fails, THEN THE Spreadsheet_Surface SHALL degrade to the in-memory file system and surface an observable error indication consistent with Requirement 1.

### Requirement 8: Source-File Import

**User Story:** As a developer importing content from many sources, I want bounded, local-first import that composes into the graph, so that ingest stays token-economical and never discards imported content offline.

#### Acceptance Criteria

1. THE Import_Pipeline SHALL treat Source_Files as the canonical ingest surface for text-like and document-like sources and for URL sources.
2. WHEN a remote URL source is fetched, THE Import_Pipeline SHALL bound the fetch by a 30-second timeout and a maximum of 10,485,760 bytes.
3. WHEN a local file is imported, THE Import_Pipeline SHALL bound the import by the same maximum of 10,485,760 bytes used for URL sources.
4. WHILE the Storage_Mode is `Offline only`, THE Import_Pipeline SHALL preserve previously imported content without discarding it.
5. WHEN a change to Source_Files occurs, including add, remove, clear, toggle, or parsed-hash update, THE Import_Pipeline SHALL trigger a recomposition of the active graph data.
6. WHEN Source_Files becomes empty, THE Import_Pipeline SHALL make the composed graph data empty with no stale rows retained.
7. WHEN a local file import changes the content of an existing same-name document, THE Import_Pipeline SHALL upsert the canonical same-name workspace document rather than retaining a duplicate replacement file.
8. WHEN a valid source in a supported format is parsed and then printed back to that format, THE Import_Pipeline SHALL produce a re-parsed result that is structurally identical to the original parsed result, preserving every field, value, and element ordering (round-trip property).
9. IF an imported source is malformed for its inferred format, THEN THE Import_Pipeline SHALL record a per-source error that identifies the source and the failure reason, SHALL leave prior imports unchanged, and SHALL continue processing remaining sources.
10. WHEN an editor save, Source_Files writeback, or GitGraph document change occurs, THE Import_Pipeline SHALL record a local document version snapshot through the shared document-versioning utility and SHALL retain at most the 50 most recent snapshots per document, discarding the oldest snapshots beyond that limit.
11. IF a URL fetch exceeds the 30-second timeout or the 10,485,760-byte maximum, THEN THE Import_Pipeline SHALL record a per-source limit-exceeded error that identifies the source and SHALL continue processing remaining sources.
12. IF a URL source is unreachable or the fetch fails, THEN THE Import_Pipeline SHALL record a per-source fetch-failure error that identifies the source and the failure reason and SHALL continue processing remaining sources.

### Requirement 9: Token Performance and Economics

**User Story:** As a solo developer optimizing for cost and context budget, I want the storage and sync path to minimize resent bytes and avoid token spend, so that operation stays within zero-TCO and token-economical constraints.

#### Acceptance Criteria

1. THE Storage_Sync_System SHALL complete every push and pull sync operation without invoking any LLM or model-inference call, such that the count of LLM calls originating from the sync path is zero.
2. WHEN a chunk's Content_Hash matches the Content_Hash already held by the sync peer for the same semantic chunk key, THE Storage_Sync_System SHALL transmit zero bytes of that chunk's content and SHALL send only the chunk key and its Content_Hash reference.
3. THE Storage_Sync_System SHALL address chunks by semantic chunk keys rather than by byte offsets.
4. WHEN a document revision's Content_Hash equals the Content_Hash of the stored revision for the same workspace-id-and-Canonical_Path pair, THE Storage_Sync_System SHALL reuse the stored raw markdown and the stored graph snapshot without regenerating either artifact.
5. BEFORE writing a `documents`, `document_chunks`, or `graph_snapshots` record to D1, THE Storage_Sync_System SHALL read the existing record and SHALL issue the write only when at least one persisted field value differs from the record being written.
6. WHEN a pull returns no records newer than the stored cursor, THE Storage_Sync_System SHALL complete the pull without issuing any D1 write.

### Requirement 10: Deployment and Cost Boundary

**User Story:** As a solo developer avoiding infrastructure cost and risk, I want the enhancement to stay within local and Dev scope at zero incremental cost, so that no Production or Cloudflare mutation is triggered by this work.

#### Acceptance Criteria

1. THE Storage_Sync_System SHALL satisfy every enhancement acceptance criterion using only local and Dev runtime scope, and SHALL NOT perform a Production mirror write or a Cloudflare resource create, update, or delete operation.
2. IF a mutating Source_Files action would target the production default origin and no explicitly configured local Worker origin is present, THEN THE Storage_Sync_System SHALL reject the mutation, preserve the local Source_Files state without modification, and surface an error indicating that a configured local Worker origin is required.
3. WHEN a mutating Source_Files action executes and an explicitly configured local Worker origin is present, THE Storage_Sync_System SHALL route the mutation to that local Worker origin rather than to the production default origin.
4. THE Storage_Sync_System SHALL keep monthly total cost of ownership at zero (0.00 in any currency) by operating within the D1 free tier, the FOSS PocketBase and Yjs stack, and the Cloudflare Worker free tier, and SHALL NOT provision any paid or metered resource beyond these free tiers.
5. THE Storage_Sync_System SHALL NOT store credentials of any kind, including repository tokens, provider keys, and secrets, in browser settings, browser local or session storage, or the Persisted_Cache.
6. IF an enhancement operation would create, update, or delete a Cloudflare resource or write to a Production mirror, THEN THE Storage_Sync_System SHALL reject the operation and SHALL NOT modify any remote resource.

### Requirement 11: Git-Remote Read/Write Engine

**User Story:** As a solo developer working offline-first from the browser, I want a KnowGrph-owned git engine that reads and writes local repositories and fetches and pushes remotes, so that I can version my workspace against git remotes without infrastructure and without leaking credentials to the browser.

#### Acceptance Criteria

1. THE Git_Engine SHALL be a KnowGrph-owned implementation and SHALL NOT introduce isomorphic-git or any other external git library as a runtime dependency.
2. WHEN the Git_Engine clones or fetches a Git_Remote, THE Git_Engine SHALL materialize the fetched git objects and references into the Local_Git_Repository within the Persisted_Cache before reporting the operation complete.
3. WHEN the Git_Engine commits local changes, THE Git_Engine SHALL route every resulting document write through the path-scoped Document Write Authority and Save Bridge defined in Requirement 4 and SHALL NOT bypass or replace that authority.
4. IF a Git_Operation targets an `agentic-canvas-os/**` path or the duplicate `huijoohwee/docs/workspace-seeds/**` root, THEN THE Git_Engine SHALL reject the operation, SHALL return a rejection response identifying the unsupported path, and SHALL NOT perform any partial write.
5. THE Git_Engine SHALL keep Git_Remote authentication credentials confined to the Storage_Worker and SHALL NOT store git tokens, keys, or secrets in browser settings, browser local or session storage, or the Persisted_Cache.
6. WHEN the Git_Engine pushes to a Git_Remote, THE Git_Engine SHALL send the push request through the Storage_Worker so that the Storage_Worker supplies the Git_Remote credentials.
7. WHILE the Storage_Mode is `Offline only`, THE Git_Engine SHALL enqueue each requested Git_Operation in the Outbox and SHALL retain the enqueued operation without discarding it.
8. WHEN the Storage_Mode transitions to `Online`, THE Git_Engine SHALL drain queued Git_Operation entries from the Outbox in first-in-first-out order.
9. WHEN the Git_Engine performs a clone, fetch, or push, THE Git_Engine SHALL bound the operation by a 30-second timeout and a maximum transfer size of 10,485,760 bytes.
10. IF a Git_Operation exceeds the 30-second timeout or the 10,485,760-byte maximum, THEN THE Git_Engine SHALL abort the operation, retain the corresponding Outbox entry, and surface a limit-exceeded indication that identifies the affected Git_Operation.
11. WHEN a push to a Git_Remote is rejected because the remote reference advanced beyond the local base reference, THE Git_Engine SHALL surface the conflict through the shared Conflict_UX defined in Requirement 3 and SHALL NOT introduce a separate git-only conflict surface.
12. IF a Git_Operation receives a Storage_Worker 5xx response or a network transport failure, THEN THE Git_Engine SHALL retain the Outbox entry and SHALL retry using a bounded backoff with a base delay of 1 second doubling on each attempt, capped at 30 seconds, for a maximum of 3 retry attempts.
13. WHEN a fetched git object carries a Content_Hash matching an object already held in the Local_Git_Repository, THE Git_Engine SHALL reuse the cached object and SHALL NOT re-fetch the matching object bytes.
14. WHEN the maximum retry attempts are exhausted for a Git_Operation, THE Git_Engine SHALL retain the corresponding Outbox entry, surface a failure indication through the shared Conflict_UX defined in Requirement 3, and discard no Git_Operation.
15. IF the Storage_Worker cannot supply valid Git_Remote credentials or the Git_Remote rejects authentication, THEN THE Git_Engine SHALL abort the Git_Operation, retain the corresponding Outbox entry, and surface an authentication-failure indication that does not expose any credential value.

### Requirement 12: Multi-Provider File Sync

**User Story:** As a solo developer syncing files across cloud storage backends, I want a KnowGrph-owned pluggable provider engine for bidirectional file and directory sync, so that I can move content between providers within zero-cost, offline-first, and no-deployment constraints.

#### Acceptance Criteria

1. THE File_Sync_Engine SHALL be a KnowGrph-owned implementation and SHALL NOT introduce an rclone binary or any other external file-sync library or binary as a runtime dependency.
2. THE File_Sync_Engine SHALL access every storage backend exclusively through a File_Sync_Provider that implements the Provider_Interface.
3. WHEN a new File_Sync_Provider that implements the Provider_Interface is registered with a unique provider identifier, THE File_Sync_Engine SHALL make the provider available for synchronization without modifying the File_Sync_Engine core.
4. WHEN a pull synchronization runs for a selected File_Sync_Provider, THE File_Sync_Engine SHALL transfer files and directories from the provider into the Persisted_Cache under the shared Storage_Sync_System contract.
5. WHEN a push synchronization runs for a selected File_Sync_Provider, THE File_Sync_Engine SHALL transfer files and directories from the Persisted_Cache to the provider.
6. WHEN a Sync_Transfer's Content_Hash matches the Content_Hash already held by the destination for the same file key, THE File_Sync_Engine SHALL skip transferring that file's content bytes and SHALL record the file as already synchronized.
7. WHEN a Sync_Transfer's Content_Hash does not match the Content_Hash held by the destination for the same file key, THE File_Sync_Engine SHALL transfer that file's content bytes to the destination.
8. WHEN the File_Sync_Engine performs a Sync_Transfer, THE File_Sync_Engine SHALL bound the transfer by a 30-second timeout and a maximum transfer size of 10,485,760 bytes.
9. IF a single Sync_Transfer fails or exceeds the 30-second timeout or the 10,485,760-byte maximum, THEN THE File_Sync_Engine SHALL record a per-file error that identifies the file and the failure reason, SHALL leave already-transferred files unchanged, and SHALL continue processing the remaining files.
10. WHILE the Storage_Mode is `Offline only`, THE File_Sync_Engine SHALL enqueue each requested Sync_Transfer in the Outbox up to a maximum of 10,000 queued entries and SHALL retain each enqueued transfer without discarding it.
11. IF the Outbox already holds 10,000 queued Sync_Transfer entries when a new Sync_Transfer is requested, THEN THE File_Sync_Engine SHALL reject the new Sync_Transfer, retain the existing queued entries, and surface a queue-capacity indication identifying the rejected transfer.
12. IF draining a queued Sync_Transfer fails, THEN THE File_Sync_Engine SHALL retry using a bounded backoff with a base delay of 1 second doubling on each attempt, capped at 30 seconds, for a maximum of 3 retry attempts, and SHALL record a per-file error identifying the transfer when the final attempt fails.
13. WHEN the Storage_Mode transitions to `Online`, THE File_Sync_Engine SHALL drain queued Sync_Transfer entries from the Outbox in first-in-first-out order.
14. THE File_Sync_Engine SHALL keep File_Sync_Provider credentials confined to the Storage_Worker and SHALL NOT store provider tokens, keys, or secrets in browser settings, browser local or session storage, or the Persisted_Cache.
15. THE File_Sync_Engine SHALL operate within local and Dev runtime scope only and SHALL NOT perform a Production mirror write or a Cloudflare resource create, update, or delete operation.
16. IF a File_Sync_Engine operation would create, update, or delete a Cloudflare resource or write to a Production mirror, THEN THE File_Sync_Engine SHALL reject the operation and SHALL NOT modify any remote resource.
