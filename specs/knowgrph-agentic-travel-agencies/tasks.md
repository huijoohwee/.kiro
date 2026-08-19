# Implementation Plan: Knowgrph Agentic Travel Agencies

## Overview

The plan sequences the **Shared_Canvas_Node_Store and Property 1 before every other component**, following the design's own recommendation and design risk R1: the CRDT-merge-to-Durable-Object join is the single largest risk in the increment, and a checksum-mismatch defect discovered after the gates are built is a rewrite rather than a fix. Nothing else is built until the node-delta contract, the canonical serialization plus `Node_Payload_Checksum`, the persistence and rehydration lifecycle, and Property 1 all hold.

Languages follow the substrate the design fixed: TypeScript in `cloudflare/workers/*`, `.js`/`.mjs` in `mcp/` and `contracts/`, TypeScript in `canvas/src/`. Tests use `node --test` plus `fast-check` 3.23.2 in `__pbt__/` directories, and Vitest plus jsdom for client surfaces. Every property test runs at `numRuns: 200`, matching the existing `RUNS = 200` convention.

Two migrations are treated as blocking work with their own tasks: `0012_travel_agency.sql`, and the chain-id constraint widening that gates any sandbox settlement Evidence Reference and whose SQLite rollback is a table rebuild.

Module decomposition follows the design so no file needs a later split: the CI-enforced caps are 600 lines and 500 KiB per authored file, and `canvasSyncRoom.ts` is already 549 lines, so the shared-node work lands in `sharedCanvasNode/` with a thin dispatch seam in the existing class.

Every configuration value — endpoints, credentials, caps, ceilings, retry bounds — resolves from operator-owned runtime configuration. No task writes such a value into repository source.

## Tasks

- [ ] 1. Shared-canvas node foundation — delta contract, canonical serialization, CRDT merge, persistence
  - [ ] 1.1 Create the typed node-delta contract
    - Create `cloudflare/workers/knowgrph-storage/sharedCanvasNode/nodeDeltaContract.ts` with the `NodeDeltaEnvelope` schema (`schema: "knowgrph-travel-node-delta/v1"`, `nodeId`, `transactionId`, `writerSide`, `clientSeq`, `updateBase64`, `updateByteLength`, `expectedScope`) and the `SharedCanvasNode` expanded-payload schema (`schema: "knowgrph-travel-shared-canvas-node/v1"`) exactly as fixed in the design's Data Models section
    - Validate totally before any partial apply: shape, `updateByteLength` equal to the decoded length and at most the configured `sharedNode.maxDeltaBytes`, well-formed identifiers, `writerSide` overwritten from the resolved session, resulting payload at most the configured `sharedNode.maxPayloadBytes`
    - Return `node-delta-schema-invalid` naming the failing field path, and `delta-limit-exceeded` naming the exceeded limit and its configured value; read both limits from configuration, never from a source literal
    - _Requirements: 1.9, 1.10, 1.16, 16.2, 16.3_

  - [ ] 1.2 Create the canonical serializer, excluded-field constant and checksum
    - Create `cloudflare/workers/knowgrph-storage/sharedCanvasNode/nodeChecksum.ts` exporting exactly one canonical serializer and exactly one digest function, plus the single `NODE_CHECKSUM_EXCLUDED_FIELDS` constant (`viewerSide`, `viewerMembershipId`, `subscriptionId`, `servedAtMs`, `remainingWindowSeconds`, `activePeerCount`)
    - Compose over the existing `grph-shared/src/collaboration/yjsSnapshot.ts` helpers: `serializeCollaborationYDoc({ documentKind: 'json' })` then `omitDeep` then `formatCollaborationJson` then `sha256Hex` via `crypto.subtle.digest('SHA-256', …)`
    - Do not use `contracts/semantic-key.js` `hashSemanticParts` for the checksum — it is FNV-1a 32-bit and not collision-resistant; leave that helper untouched for its existing dedup role
    - Carry monetary amounts as integer minor units only, so no rounding difference can enter the digest
    - _Requirements: 1.3, 16.5_

  - [ ] 1.3 Implement CRDT merge over the existing Yjs helpers
    - Create `cloudflare/workers/knowgrph-storage/sharedCanvasNode/nodeCrdtMerge.ts` holding one `Y.Doc` per node identity in an LRU-bounded `Map` of 64 documents, created through `createCollaborationYDoc({ documentKey: 'txnode/{workspaceId}/{roomId}/{nodeId}.json', documentKind: 'json', initialText: '{}' })`
    - Apply deltas through `applyYjsUpdateBase64({ doc, updateBase64, origin: 'room-delta' })` and treat its `false` return as the zero-length guard rather than a silent accept
    - Import the `grph-shared` helpers and add nothing to them; escalate rather than absorb any change that `yjsSnapshot.ts` itself would need (design risk R9)
    - _Requirements: 1.2, 1.6, 16.5_

  - [ ] 1.4 Implement Durable Object persistence, rehydration and the bounded replay log
    - Create `cloudflare/workers/knowgrph-storage/sharedCanvasNode/nodeStorage.ts` implementing the `{recordKind}:{workspaceId}:{roomId}:{recordId}` key layout for `txnode:`, `txnode-delta:…#{seq}` and `txnode-index:`, introducing no third key pattern
    - Per accepted delta inside one input-gate turn: increment `acceptedSeq`, write the replay entry, recompute the checksum, write the full `txnode` record with a non-empty `yjsStateBase64`, update the index when scope or party identifiers change, prune replay entries past the configured retention window plus margin and past `sharedNode.replayLogMaxEntries`
    - On restart, rehydrate from the persisted state, recompute and compare against the persisted checksum; on inequality record `node-rehydration-checksum-mismatch`, serve the node read-only and reject further deltas — terminal, never auto-repaired
    - _Requirements: 1.1, 1.6, 1.7, 1.14_

  - [ ] 1.5 Register the new `__pbt__` directories in the test script
    - Add `cloudflare/workers/knowgrph-storage/__pbt__`, `cloudflare/workers/knowgrph-payment/__pbt__` and `canvas/src/__pbt__` to the `runtime:test` script in `package.json`, which enumerates directories explicitly rather than globbing
    - Confirm the existing `mcp/__pbt__`, `contracts/__pbt__` and `docs/__pbt__` entries still resolve after the edit
    - _Requirements: 16.5_

  - [ ]* 1.6 Write property test for dual-read checksum identity
    - **Property 1: Dual-read checksum identity**
    - **Validates: Requirements 1.2, 1.3, 1.11, 1.14, 1.15, 8.7, 12.9**
    - File: `cloudflare/workers/knowgrph-storage/__pbt__/shared-canvas-node-checksum.pbt.test.mjs`, 200 runs, at most 40 deltas per run, generator payloads capped at the configured maximum so limit rejection is exercised without dominating the run
    - Drive the Durable Object through the existing `KnowgrphDurableObjectStateLike` fake used by `canvas/src/__tests__/runtimeIdentityCanvasRoom.test.ts`; no Worker, no `wrangler dev`
    - Cover the permutation clause, restart-with-rehydration, snapshot fallback and over-limit rejection clauses in the same property

- [ ] 2. Shared-canvas node store — dispatch seam, rejection semantics, subscription filtering, provenance
  - [ ] 2.1 Implement the zero-model Cost_Log per node-store operation
    - Create `cloudflare/workers/knowgrph-storage/sharedCanvasNode/nodeCostLog.ts` emitting one log per merge, individual write, rejected delta, reconnection replay and snapshot serve
    - Use `createCostLog` from the existing `contracts/cost-log.schema.js` verbatim with `model: "none"` and zero token, cache and cost fields; carry operation counters in a sibling `operationCounters` record so the validated shape stays unforked
    - _Requirements: 1.12, 1.13, 15.2, 16.5_

  - [ ] 2.2 Add the node-message dispatch seam to the existing Durable Object
    - Modify `cloudflare/workers/knowgrph-storage/canvasSyncRoom.ts` to dispatch `node.delta`, `node.subscribe`, `node.resume` and `node.snapshot.request` from `webSocketMessage` after the existing `ping` / `runtime.identity.*` / `presence.update` branches and before the `document.sync` branch, leaving `document.sync` behaviour unchanged for non-transaction documents
    - Emit `node.delta.accepted`, `node.delta.rejected`, `node.state`, `node.replay` and `node.snapshot` frames; attach subscriber-specific fields to the outbound frame after the checksum is computed so they never enter the Yjs document
    - Keep the seam thin — the class is already 549 lines against a 600-line cap, so logic belongs in `sharedCanvasNode/*`
    - _Requirements: 1.1, 1.3, 1.8, 16.4_

  - [ ]* 2.3 Write property test for rejection leaving the record byte-identical
    - **Property 13: A rejected write leaves the record byte-identical**
    - **Validates: Requirements 1.9, 1.10, 1.16, 12.4, 12.7, 13.7, 16.10**
    - File: `cloudflare/workers/knowgrph-storage/__pbt__/shared-canvas-node-rejection.pbt.test.mjs`, 200 runs; assert byte-identity by comparing the serialized stored record before and after each rejection

  - [ ] 2.4 Implement scope promotion, party filtering and the replay-versus-snapshot decision
    - Create `cloudflare/workers/knowgrph-storage/sharedCanvasNode/nodeSubscription.ts` filtering on the enumeration index before any payload is materialized, so a rejected subscription serves zero payloads including zero partial and zero metadata-only payloads
    - Default intent and draft nodes to `personal`, promote monotonically to `shared` when a Discovery result set is recorded, record `merchantPartyId` no later than that promotion and treat both party identifiers as immutable afterwards
    - Implement `chooseResume` exactly as specified: up-to-date, then the configured replay window, then the 512-entry gap cap, then replay-log completeness, then the 262 144-byte total, otherwise replay — both routes ending at checksum equality with the other side
    - Re-validate the session token and resolved side at intervals of at most the configured interval and before serving each node payload; close within 5 s with `session-expired` on expiry or revocation, leaving stored records unchanged
    - _Requirements: 1.4, 1.5, 1.11, 1.15, 13.3, 13.4, 13.6, 13.7, 13.8, 13.9_

  - [ ]* 2.5 Write property test for scope and party containment
    - **Property 7: Scope and party containment**
    - **Validates: Requirements 1.5, 13.3, 13.4, 13.6**
    - File: `cloudflare/workers/knowgrph-storage/__pbt__/shared-canvas-node-scope.pbt.test.mjs`, 200 runs, at most 40 nodes, expected result computed by an independently written oracle filter

  - [ ] 2.6 Implement the hash-linked provenance chain inside the Durable Object
    - Create `cloudflare/workers/knowgrph-storage/sharedCanvasNode/provenanceChain.ts` with `prov:{workspaceId}:{roomId}:{transactionId}#{seq}` entries and a `prov-head:` next-index authority, reusing the `previous_hash` / `entry_hash` pattern already proven in `mcp/export-ledger.js` rather than inventing a second chain shape
    - Consecutive indices from 1, genesis predecessor of 64 hex zeros at index 1, `canonicalEntry` fixing field order and value encoding and excluding the entry's own digest, append within 2 s of the event
    - Require every `payment-call` entry to reference a resolvable lower-index `pass` decision entry; reject otherwise with `provenance-decision-reference-unresolvable` leaving stored entries byte-identical; reject modify and delete with `provenance-append-only-violation`
    - Record a gap marker occupying the index a failed append would have held, carrying the expected preceding digest and the failed event kind, with no further attempt at that index; report each as a discontinuity naming its index
    - Record credential values as the fixed redaction placeholder and the configuration key name in their place
    - _Requirements: 12.1, 12.2, 12.3, 12.4, 12.5, 12.6, 12.7, 12.8, 12.9, 12.10, 16.9_

  - [ ]* 2.7 Write property test for provenance chain integrity
    - **Property 8: Provenance chain integrity**
    - **Validates: Requirements 12.1, 12.2, 12.3, 12.4, 12.5, 12.6, 12.7, 12.8, 12.10**
    - File: `cloudflare/workers/knowgrph-storage/__pbt__/provenance-chain.pbt.test.mjs`, 200 runs, at most 40 appends, with injected write-failure positions, concurrent batches and a byte-flip tamper generator

- [ ] 3. Checkpoint — the foundation holds before anything else is built
  - Ensure all tests pass, ask the user if questions arise.
  - Property 1, Property 7, Property 8 and Property 13 must be green before any gate, harness or client work starts. This is the R1 gate: a checksum defect found later is a rewrite.

- [ ] 4. The two blocking migrations
  - [ ] 4.1 Add the travel-agency D1 migration
    - Create `cloudflare/d1/migrations/0012_travel_agency.sql` with the five tables the design fixed: `workspace_membership_transaction_sides`, `travel_wallet_profile_links`, `travel_notification_recipients`, `travel_notification_suppression`, `travel_agency_runtime_config`
    - Use a side table rather than `ALTER TABLE workspace_memberships ADD COLUMN transaction_side`, so the `CHECK (transaction_side IN ('shopper','merchant'))` survives and the existing `WorkspaceMembershipRow` type in `db.ts` and its fake in `canvas/src/__tests__/helpers/fakeKnowgrphStorageD1Reads.ts` stay untouched
    - _Requirements: 13.1, 16.2_

  - [ ] 4.2 Widen the chain-id constraint from a mainnet literal to an operator-configured allowlist
    - Create the companion migration (next free number, `cloudflare/d1/migrations/0013_chain_id_allowlist.sql`) replacing `CHECK (chain_id = 43114)` on `payment_chain_evidence_observations` and `payment_chain_confirmed_funding` with membership in the operator-configured `settlement.chainIdAllowlist`, adding no second hard-coded literal
    - This is blocking: until it lands, no sandbox settlement Evidence Reference can be recorded for Requirement 7 (design risk R5). Write the table-rebuild rollback as part of the migration file's header comment, because SQLite cannot drop this constraint cheaply
    - _Requirements: 7.7, 16.2_

  - [ ]* 4.3 Write migration tests
    - Apply both migrations through `miniflare` in `cloudflare/workers/knowgrph-storage/__tests__/travelAgencyMigrations.test.mjs`, asserting the five tables exist with their `CHECK` constraints, that a testnet chain id is now accepted, and that no pre-existing column changed
    - _Requirements: 13.1, 7.7_

- [ ] 5. Session_Authority transaction-side extension
  - [ ] 5.1 Resolve the transaction side from the membership side table
    - Create `cloudflare/workers/knowgrph-storage/travelAgencySide.ts` returning `{ userId, membershipId, workspaceId, role, transactionSide }`, leaving the existing `viewer | editor | owner | provider-admin` union untouched and independent of the side attribute
    - Return distinguishable `session-invalid` and `transaction-side-unresolved` typed errors
    - _Requirements: 13.1, 13.6_

  - [ ] 5.2 Propagate the side through the existing internal-header mechanism
    - Modify `cloudflare/workers/knowgrph-storage/canvasRoomProxyIdentity.ts` and `chatAuth.ts` so the same proxy-identity authority that writes `x-knowgrph-room-role` also writes the sibling `x-knowgrph-room-transaction-side`, validated in the same `readConnectionAttachment` gate
    - One propagation mechanism carrying two fields — no second authentication path, and no compound value packed into the role string
    - Discard any client-supplied transaction-side or party-identifier value
    - _Requirements: 13.2, 13.5_

  - [ ]* 5.3 Write unit tests for side resolution and header propagation
    - Cover a valid shopper session, a valid merchant session, an invalid token, an unresolved side, and a subscription request whose client-supplied side is discarded; assert `session-invalid` and `transaction-side-unresolved` are distinguishable
    - File: `cloudflare/workers/knowgrph-storage/__tests__/travelAgencySide.test.mjs`
    - _Requirements: 13.1, 13.2, 13.5, 13.6_

- [ ] 6. Operator-owned configuration resolution
  - [ ] 6.1 Implement the runtime configuration resolver
    - Create `cloudflare/workers/knowgrph-payment/travelAgency/travelAgencyConfig.ts` resolving non-secret values from the `travel_agency_runtime_config` table and credentials from Worker secrets, caching non-credential values for at most 300 s and resolving credential values on every invocation without caching
    - Return `configuration-missing` naming the absent key and the requesting component before that component's first outbound call, with zero dependent outbound calls and no configuration value in the error payload; leave components whose configuration is complete free to call out
    - Record every credential as the fixed redaction placeholder in Cost_Log entries, provenance entries and error payloads, with the key name in place of the value; declare key names only in `wrangler.toml`, never values
    - _Requirements: 16.1, 16.2, 16.6, 16.7, 16.8, 16.9_

  - [ ]* 6.2 Write property test for configuration-missing error conditions
    - **Property 9: Configuration-missing error conditions**
    - **Validates: Requirements 4.4, 4.10, 5.11, 6.12, 6.15, 7.7, 10.4, 11.10, 16.2, 16.6, 16.7, 16.9**
    - File: `cloudflare/workers/knowgrph-payment/__pbt__/travel-configuration-resolution.pbt.test.mjs`, 200 runs across the travel-agency component set, with a credential-value generator whose values are asserted absent from every emitted Cost_Log, provenance entry and error payload
    - Add each new provider interface to the component set as it lands — design risk R8 makes this a checklist item, not an automatic guarantee

- [ ] 7. Guardrail_Gate
  - [ ] 7.1 Implement the deterministic server-side budget gate
    - Create `cloudflare/workers/knowgrph-payment/travelAgency/guardrailGate.ts` reading Budget_Ceiling and Retry_Bound from session-scoped configuration at request time, holding neither in source nor in component defaults
    - Emit `pass` / `block` / `retry` per the design's decision rules, including the terminal `block` at Retry_Bound returning at most the 5 lowest-total-amount offers in ascending order, and the Retry_Bound-zero case blocking terminally on the first over-budget offer
    - Discard any client-supplied gate result, approved amount or retry count, re-evaluate from server-held state, and record each discarded value in that decision's provenance entry
    - Return `invalid-guardrail-configuration` and `currency-mismatch` as typed terminal outcomes with zero Payment_Call invocations; emit one zero-model Cost_Log per decision
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6, 2.8, 2.9, 2.10, 2.11, 2.12, 2.13_

  - [ ]* 7.2 Write property test for the budget bound and bounded retry
    - **Property 3: Budget bound with bounded, terminating retry**
    - **Validates: Requirements 2.2, 2.3, 2.4, 2.5, 2.10, 2.12, 2.13, 2.14**
    - File: `cloudflare/workers/knowgrph-payment/__pbt__/guardrail-gate-budget.pbt.test.mjs`, 200 runs, at most 25 fares, retry bound at most 6 so exhaustion is reached in every run, currencies deliberately mixed

  - [ ] 7.3 Wire `probe.evolve` as the retry-scoring surface
    - Extend `guardrailGate.ts` to invoke the existing `knowgrph.probe.evolve` MCP tool on every `retry` result and record the returned exemplar identifier in that decision's provenance entry; make no schema change to the probe tools
    - On error or no exemplar within 5 s, record `retry-scoring-unavailable` in place of the identifier, emit the retry decision unchanged, and still count the attempt toward Retry_Bound; attribute the probe's token cost to `probe.evolve`, not to the gate
    - _Requirements: 2.7, 2.14_

  - [ ]* 7.4 Write unit tests for the guardrail bounds and typed errors
    - Cover the 1-second decision bound with an injected clock, the `probe.evolve` happy path with a mocked exemplar identifier, and each typed error string with its named fields once per code
    - File: `cloudflare/workers/knowgrph-payment/__tests__/travelAgencyGuardrailGate.test.mjs`
    - _Requirements: 2.6, 2.7, 2.8, 2.11, 2.12, 2.13_

- [ ] 8. Confirmation_Gate
  - [ ] 8.1 Implement the confirmation gate over the existing approvals table
    - Create `cloudflare/workers/knowgrph-payment/travelAgency/confirmationGate.ts` reusing the `payment_purchase_approvals` shape (`approval_ref`, `amount_minor`, `currency`, `expires_at`, `consumed_at`) rather than adding a parallel approval store
    - Enter pending on cart-add bound to the Guardrail_Gate `pass` amount; record Human_Confirm_Event timestamps in UTC at millisecond precision; assign every accepted Payment_Call a session-log position strictly greater than the most recent valid confirm event, and accept no call for which such a position cannot be assigned
    - Treat the Path_A wallet signature request as a member of the Payment_Call set; invalidate on a divergence of at least 0.01 minor units or any currency difference; read the validity window as an integer 30–900 s from configuration, applying 300 s when absent; make a matching duplicate confirm submission a no-op that records no second event
    - Reject on client-state divergence with `confirmation-state-mismatch`, leaving server state unchanged and recording both states in the provenance entry; emit one zero-model Cost_Log per operation
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 3.9, 3.11, 3.12, 3.13, 15.2_

  - [ ]* 8.2 Write property test for payment-call ordering
    - **Property 2: No payment call precedes human confirmation**
    - **Validates: Requirements 3.2, 3.4, 3.5, 3.6, 3.13, 6.14, 9.5, 14.9, 14.10**
    - File: `cloudflare/workers/knowgrph-payment/__pbt__/confirmation-gate-ordering.pbt.test.mjs`, 200 runs, at most 30 events, injected clock, outbound invocations counted by a spy, offline restore clause driven with deliberately duplicated action identifiers

  - [ ]* 8.3 Write property test for server-authoritative decisions under forged client state
    - **Property 11: Server-authoritative decisions under forged client state**
    - **Validates: Requirements 2.9, 3.8, 3.9, 9.1, 13.2**
    - File: `cloudflare/workers/knowgrph-payment/__pbt__/server-authoritative-decisions.pbt.test.mjs`, 200 runs; the oracle recomputes each decision from server state with the forged payload removed

- [ ] 9. Checkpoint — both gates hold
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 10. Discovery harnesses, offer schema and scoring
  - [ ] 10.1 Define the internal typed offer schema and normalizer
    - Create `mcp/travel-agency/normalizedOffer.js` with the `NormalizedOffer` shape (`offerId`, `sourceId`, `providerReference`, `totalAmountMinor` inclusive of every tax and mandatory fee, lowercase ISO 4217 `currency`, `holdWindowSeconds`, `aboveBudget`, `attributes`) plus input and output schema validators invoked in both directions at every provider boundary
    - Map every provider error to exactly one code from the closed set `duplicate-booking | rate-limited | provider-unavailable | provider-error-unmapped`; return `provider-contract-violation` naming the failing field path and requesting component, and write no part of a failing response to a Shared_Canvas_Node
    - Reuse the existing typed harness contract shape; introduce no second harness contract
    - _Requirements: 4.3, 4.7, 5.3, 16.3, 16.5, 16.10_

  - [ ] 10.2 Implement the intent parser
    - Create `mcp/travel-agency/intentParser.js` over the existing typed harness contract and model-adapter seam, producing the flight intent (origin, destination, date range with start not earlier than the request date and span at most 90 days, budget ceiling 1–1 000 000 in one currency) and the shopping intent (item description 1–200 chars, price ceiling 0.01–999 999 999.99, ranking criterion drawn from the configured set)
    - Return `unparseable-intent` naming every unresolved or out-of-bounds field, emitting no typed intent; reject requests over 2 000 characters
    - Emit exactly one Cost_Log per invocation with model identifier, prompt tokens, completion tokens and estimated cost, recorded before the invocation returns — this is one of only two token-consuming components in the increment
    - _Requirements: 4.1, 4.2, 5.1, 5.10, 15.1_

  - [ ] 10.3 Implement the flight discovery harness
    - Create `mcp/travel-agency/flightDiscovery.js` with the provider behind a `FlightFareProvider` typed interface whose endpoint, credentials and route catalogue resolve from configuration at each invocation
    - Return within 30 s at most 50 schema-valid fares each carrying a provider fare reference, a tax-and-fee-inclusive total, its currency and a stated Hold_Window; hold at most 3 concurrent probes; emit each resolved probe's fares as a partial result within 1 s; cancel a probe not resolving within 10 s and record it as cancelled
    - Implement the degraded mode (5 lowest comparable fares flagged `above-budget`), `unrankable-fares`, `provider-unconfigured` with zero outbound calls, and the `no-fares-found` empty list with no retry
    - Record one Cost_Log per invocation with attempted, resolved and failed-or-cancelled probe counts, zero model calls and zero estimated cost
    - _Requirements: 4.3, 4.4, 4.5, 4.6, 4.7, 4.8, 4.9, 4.10, 4.11, 4.12_

  - [ ]* 10.4 Write property test for offer normalization and error-code totality
    - **Property 5: Offer normalization round trip and error-code totality**
    - **Validates: Requirements 4.3, 4.5, 4.7, 5.3, 5.6**
    - File: `mcp/__pbt__/travel-offer-normalization.pbt.test.mjs`, 200 runs, at most 30 offers per response, with fields randomly omitted, retyped, or filled with non-ASCII text and awkward numeric encodings, and error shapes no mapping rule anticipates

  - [ ] 10.5 Implement the shopping discovery harness
    - Create `mcp/travel-agency/shoppingDiscovery.js` dispatching exactly one request per configured source in parallel within a single pass, at most 10 sources, with each source behind a `ProductSourceProvider` typed interface resolved from configuration at runtime
    - Mark a source degraded on error, schema failure or no complete response within 10 s of dispatch, send it no further request in the pass, and still return the remaining sources' offers; start no retry loop and no per-source retry attempt
    - Record one Cost_Log per invocation with per-source dispatch outcomes and degraded markings, zero model calls
    - _Requirements: 5.2, 5.3, 5.4, 5.7, 5.11_

  - [ ] 10.6 Implement the offer scorer
    - Create `mcp/travel-agency/offerScorer.js` returning one ranked list containing every input offer exactly once, with a partial marking plus responding-source and configured-source counts whenever fewer sources returned schema-valid offers than were configured
    - One scoring invocation per pass, no additional discovery pass including when every source is degraded, in which case return an empty ranked list marked partial; `configuration-missing` when the ranking-criteria set is absent
    - Emit one Cost_Log per invocation including a zero-offer invocation, carrying model identifier, prompt tokens, completion tokens and estimated cost
    - _Requirements: 5.5, 5.6, 5.8, 5.9, 15.1_

  - [ ]* 10.7 Write property test for scorer conservation
    - **Property 6: Scorer conservation**
    - **Validates: Requirements 5.5**
    - File: `mcp/__pbt__/travel-offer-scorer.pbt.test.mjs`, 200 runs, at most 10 lists of at most 20 offers, including empty inner lists and offers that compare equal on every scoring field

  - [ ]* 10.8 Write unit tests for intent parsing and harness bounds
    - Cover `unparseable-intent` field naming for each unresolved field, the 2 000-character rejection, the 90-day span bound, the 3-concurrent-probe cap, the 10-second probe cancellation and the `no-fares-found` no-retry path
    - Files: `mcp/__tests__/travel-intent-parser.test.mjs`, `mcp/__tests__/travel-flight-discovery.test.mjs`
    - _Requirements: 4.1, 4.2, 4.9, 4.11, 4.12, 5.1, 5.10_

- [ ] 11. `sse` transport admission in the external gateway contract
  - [ ] 11.1 Add `sse` to the transport enum, the normalizer and their tests
    - Modify `mcp/external-tool-gateway-contract.js` so the `transportType` enum reads `["stdio", "streamable-http", "sse"]`
    - Modify `mcp/external-tool-profile-registry.js` to add an `sse` branch to the single normalizer function, asserting the same HTTPS-only, no-credentials, no-query, no-fragment URL rules and the same `headersFromEnv` mapping as the existing branches, and update the fail-closed message that currently reads `profile.transport.type must be stdio or streamable-http`
    - Add the `sse` cases to `mcp/__tests__/external-tool-profile-registry.test.mjs` and `mcp/__tests__/external-tool-session.test.mjs`: enum membership, one accepted profile, one rejected non-HTTPS URL, one rejected credential-bearing URL, one rejected URL carrying a query or fragment
    - Extend the enum and the normalizer only — a parallel transport registry would fork the single authority
    - _Requirements: 6.2, 16.5_

- [ ] 12. Issuance_Service and the net-new x402 payer role
  - [ ] 12.1 Build the x402 payer client path
    - Create `cloudflare/workers/knowgrph-payment/travelAgency/x402PayerClient.ts` parsing a gateway payment challenge and producing an EIP-3009 `transferWithAuthorization` typed-data request from values carried in that challenge only
    - This role is **net-new**. The existing `agenticCommerceX402.ts` implements the **resource-server / charging** role (`x402ResourceServer`, `paymentMiddleware`, `HTTPFacilitatorClient`) and is not reused or extended here (design risk R4). The `@x402/core`, `@x402/evm` and `viem` dependencies are present; the payer code path is not
    - Treat the funding signature as gasless for the signer and read no wallet native gas balance as a precondition for this step
    - _Requirements: 6.4, 6.5_

  - [ ] 12.2 Implement the wallet signing interface adapter
    - Create `cloudflare/workers/knowgrph-payment/travelAgency/walletSigningInterface.ts` accepting `{ challengeDerivedTypedData }` or `{ unsignedTransfer }` and returning `{ signature }`, `{ broadcastTxHash }` or a decline, with no key material touching any component
    - Keep the gas asymmetry explicit in the types: the Path_B funding signature is gasless for the signer, and Path_A is an ordinary transaction the wallet broadcasts and pays gas for
    - Return `signature-declined`, `signature-timeout` and `broadcast-failed` as typed values
    - _Requirements: 6.5, 6.6, 9.2, 9.7_

  - [ ] 12.3 Implement the issuance service over MCP/SSE
    - Create `cloudflare/workers/knowgrph-payment/travelAgency/issuanceService.ts` reusing `payment_purchase_cards` for card state and the `straitsxPaymentRailAdapter.ts` credential-handling pattern; resolve endpoint, transport, tool names and Per_Card_Cap from configuration with no code default, and bound each tool call to 30 s from dispatch to first response frame
    - Set the card scope amount equal to the Guardrail_Gate `pass` amount exactly, in the same settlement currency, in that currency's smallest unit, with no rounding, buffer or adjustment
    - On an x402 challenge, request exactly one signature and retry the call exactly once; a second challenge returns `x402-retry-bound-exceeded` with the observed challenge count recorded
    - Count only successful issuances toward the per-transaction limit of one; issue exactly one replacement with a recorded `view-lost-reissue` reason when a card has no view acknowledgement within 300 s and zero authorizations
    - Return `amount-exceeds-per-card-cap` naming the configured cap and the approved amount, with zero issuance tool calls and **no multi-card or split-funding flow** — B2 stays a typed error. Also implement `funding-declined`, `confirmation-required`, `issuance-provider-error`, `issuance-deadline-elapsed` with no automatic retry of a signed call, and `configuration-missing`
    - Bind the sandbox endpoint only; make zero production issuance calls and bind no production endpoint — B3 keeps that boundary `closed`
    - Emit one Cost_Log per invocation with the MCP call count, zero model calls and zero estimated cost
    - _Requirements: 6.1, 6.3, 6.4, 6.6, 6.7, 6.8, 6.9, 6.10, 6.11, 6.12, 6.13, 6.14, 6.15, 6.16, 17.3_

  - [ ]* 12.4 Write integration tests against a mocked SSE server
    - Stand up a local SSE server replaying recorded frames following the `mcp/__tests__/external-tool-session.test.mjs` transport-stubbing pattern: an x402 challenge, a successful signed retry, a second challenge, a first-frame timeout and a provider error
    - Use a signer stub that can sign, decline or go silent; assert the 30-second first-frame bound with an injected clock and that no signed call is automatically retried
    - File: `cloudflare/workers/knowgrph-payment/__tests__/travelAgencyIssuanceSse.integration.test.mjs`
    - _Requirements: 6.1, 6.4, 6.6, 6.7, 6.16_

  - [ ]* 12.5 Write unit tests for the issuance typed-error taxonomy
    - One test per code with its named fields: `amount-exceeds-per-card-cap` (asserting zero issuance tool calls and no split-funding path exists), `configuration-missing` for the absent cap key, `confirmation-required`, `funding-declined`, `x402-retry-bound-exceeded`, plus the `view-lost-reissue` path distinguished from a duplicate-issuance defect
    - File: `cloudflare/workers/knowgrph-payment/__tests__/travelAgencyIssuanceErrors.test.mjs`
    - _Requirements: 6.8, 6.9, 6.10, 6.11, 6.14, 6.15, 6.16_

- [ ] 13. Wallet_Linking_Service
  - [ ] 13.1 Implement address-to-profile linking
    - Create `cloudflare/workers/knowgrph-payment/travelAgency/walletLinking.ts` over `travel_wallet_profile_links`, with the custody provider behind a `CustodyProfileProvider` typed interface whose endpoint, credentials and verification deadline resolve from configuration, defaulting the deadline to 10 s when unconfigured
    - Accept `0x` plus exactly 40 hex characters in any letter case, treat case-only differences as the same address, require no vendor-specific attestation, and make the mapping readable to attribution checks within 5 s of recording
    - Hold an unmapped inbound transfer as `pending-manual-linkage` with zero usable balance credited and no time-based expiry that credits it; return `address-already-linked` and `invalid-wallet-address` with zero outbound custody calls
    - Record in the module header that generic EVM acceptance is an assumption pending Open Question 4, claiming no vendor-specific attestation coverage
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.6, 10.7, 10.8, 10.9_

  - [ ]* 13.2 Write property test for wallet-linking idempotence
    - **Property 10: Wallet-linking idempotence**
    - **Validates: Requirements 10.1, 10.2, 10.5, 10.6, 10.7, 10.8**
    - File: `cloudflare/workers/knowgrph-payment/__pbt__/wallet-linking.pbt.test.mjs`, 200 runs, with randomized letter case, a malformed address family (wrong length, non-hex, missing prefix, embedded whitespace) and a two-element profile pool so the conflicting-profile branch is reached

- [ ] 14. Settlement_Verifier and the net-new second source
  - [ ] 14.1 Add the second independent on-chain source adapter and its fixture tree
    - Extend `cloudflare/workers/knowgrph-payment/chainEvidenceAdapter.ts` and `chainEvidencePersistence.ts` with a second read-only source behind its own typed interface and its own configuration entry. Today only one source is read, so this adapter, its fixtures and the disagreement path are all **net-new** (design risk R6)
    - Create the second source's fixtures at `cloudflare/workers/knowgrph-payment/__tests__/fixtures/chain-evidence-source-b/`, mirroring the existing tree's layout: matching, over-amount, under-amount, same-symbol-different-contract, empty-range, rate-limited and server-error responses
    - Enforce mutual independence: neither source is the interface that reported the identifier, and the two share no upstream data provider
    - _Requirements: 7.1, 7.7_

  - [ ] 14.2 Implement the two-source verification comparison
    - Create `cloudflare/workers/knowgrph-payment/travelAgency/settlementVerifier.ts` recording exactly one of `verified | verification-disagreement | verification-mismatch | unverified` on the transaction's Shared_Canvas_Node within 60 s, using at most 2 attempts per source at a 10-second deadline per attempt
    - Compare the resolved amount to the guardrail-approved amount for exact equality in the settlement asset's smallest unit with zero tolerance, and the resolved signing address to the funding-signature address case-insensitively; read the minimum confirmation depth from configuration
    - Issue only data-retrieval requests — no call transfers value, submits a transaction or mutates on-chain state; leave the transaction state unchanged in every non-`verified` case
    - Emit one zero-model Cost_Log per verification
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 7.7, 7.8, 7.9, 7.10, 15.2_

  - [ ]* 14.3 Write property test for verification outcome totality and path parity
    - **Property 14: Settlement verification outcome totality and path parity**
    - **Validates: Requirements 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 7.8, 7.9, 7.10, 9.6**
    - File: `cloudflare/workers/knowgrph-payment/__pbt__/settlement-verification.pbt.test.mjs`, 200 runs, with source responses covering silence on both attempts, transport errors and disagreeing states, and an independently written decision-table oracle

- [ ] 15. Checkpoint — settlement and issuance hold
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 16. Escrow_Meter
  - [ ] 16.1 Implement hold-window recording and the `backed` marking
    - Create `cloudflare/workers/knowgrph-payment/travelAgency/escrowMeter.ts` recording the Hold_Window in whole seconds within 5 s of the fare hold, recording an escrow commitment with its on-chain identifier, timestamp and window duration, and ordering the commitment timestamp strictly before the `inventory-committed` transition
    - Apply `backed` only for an unexpired commitment whose window is within 5 s of the recorded Hold_Window; otherwise withhold with `no-escrow-commitment` or `window-mismatch` naming both durations in whole seconds; withdraw both the remaining duration and the marking when the remainder reaches zero
    - Store `commitmentTimestampMs` and `commitmentWindowSeconds` in the checksummed payload and derive the remaining window client-side — `remainingWindowSeconds` is in the excluded-field set and must never be stored, or the two subscriptions disagree on every read
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 8.8, 8.9, 8.10_

  - [ ]* 16.2 Write unit tests for the `backed`-marking decision table
    - Four cells plus the expiry boundary, asserted against the same oracle used inside the implementation; deliberately example-shaped rather than a property, since the table has no meaningful input variation
    - File: `cloudflare/workers/knowgrph-payment/__tests__/travelAgencyEscrowMeter.test.mjs`
    - _Requirements: 8.5, 8.6, 8.8, 8.10_

- [ ] 17. Path_A orchestration and the B1 honesty controls
  - [ ] 17.1 Implement settlement-path selection
    - Create `cloudflare/workers/knowgrph-payment/travelAgency/settlementPath.ts` selecting Path_A only where the merchant address is recorded in `pathA.merchantAddresses` as accepting direct stablecoin transfer, using that recorded address as the sole recipient and deriving no recipient value from client input
    - Return `merchant-address-unrecorded` with no Path_A option offered, no signature requested and the blocked state recorded on the node
    - _Requirements: 9.1, 9.11_

  - [ ] 17.2 Implement the Path_A orchestration path
    - Create `cloudflare/workers/knowgrph-payment/travelAgency/orchestrator.ts` reading the signing wallet's native gas balance as the last recorded step before a signature request with no intervening Path_A step, re-reading when the most recent reading is more than 5 s old, and deriving the estimated cost via the configured `pathA.gasEstimateStrategy`
    - Return `insufficient-gas` naming the read balance and the estimated cost, requesting no signature regardless of any earlier reading; set the requested transfer amount equal to the Guardrail_Gate `pass` amount and record the approval event before the signature-request event in session-log order
    - Record `signature-declined` with no broadcast, no rollback and the unsigned payload unchanged, and `broadcast-failed` carrying the network's reason with no automatic re-broadcast and no balance debited
    - Write both paths through one `recordSettlementOutcome` seam with the same typed payload, introducing zero Path_A-only fields, and record a transaction log with zero entries naming any custody-provider interface for Path_A
    - _Requirements: 9.2, 9.3, 9.4, 9.5, 9.6, 9.7, 9.8, 9.12_

  - [ ] 17.3 Record the advisory guardrail flag on every Path_A record
    - Set `settlement.guardrailApprovalAdvisory` to `true` on every Path_A record written through `recordSettlementOutcome`, and label the Path_A guardrail approval event as advisory in the provenance entry
    - Block no Path_A transfer on budget-ceiling grounds, and state in the module header that no server-side or on-chain enforcement point exists in this increment (B1). Add no enforcement point and claim none
    - _Requirements: 9.9, 17.10_

  - [ ]* 17.4 Write unit tests for the Path_A typed errors and gas-read ordering
    - Assert the gas read is the last recorded step before a signature request, the 5-second re-read, and each of `insufficient-gas`, `merchant-address-unrecorded`, `signature-declined` and `broadcast-failed` with its named fields; assert `guardrailApprovalAdvisory` is `true` on every Path_A record
    - File: `cloudflare/workers/knowgrph-payment/__tests__/travelAgencyPathA.test.mjs`
    - _Requirements: 9.3, 9.4, 9.7, 9.9, 9.11, 9.12_

- [ ] 18. Notification_Dispatcher — suppression and verification before the provider call
  - [ ] 18.1 Build the suppression-key store and recipient mapping
    - Create `mcp/travel-agency/notificationSuppressionStore.js` over `travel_notification_suppression` keyed on `(nodeId, transitionSequenceIndex, recipientId)`, retaining each key at least 24 h after the recorded state-change timestamp and transmitting nothing for a key already holding a successful or terminal outcome, including on a duplicate or replayed event
    - Create `mcp/travel-agency/notificationRecipients.js` over `travel_notification_recipients`, resolving the messaging identifier as the first step of a send operation before any provider request is constructed, and ending the operation with `notification-recipient-unmapped` when no mapping exists
    - Build this before the provider call: no notification code exists anywhere in the repository today (design risk R7), and the store plus the resolution order are where a false send comes from
    - _Requirements: 11.4, 11.8, 11.9_

  - [ ] 18.2 Implement state-history verification as the anti-false-send gate
    - Create `mcp/travel-agency/notificationStateVerification.js` deriving every send decision from the normalized event and the node's recorded state history only, with no provider-payload parsing of its own
    - Discard an event naming a state absent from the history at evaluation time, transmit nothing, and record a canvas event naming the discarded unverified state; treat every state outside the configured Notified_State set as non-notifying
    - Run this check before any provider request is constructed
    - _Requirements: 11.1, 11.3, 11.5, 11.14_

  - [ ] 18.3 Implement the dispatcher, its ordering and its bounded retry
    - Create `mcp/travel-agency/notificationDispatcher.js` with the provider behind a `MessagingProvider` typed interface whose endpoint and credentials resolve from configuration at runtime
    - Initiate the first attempt within 10 s of the recorded state-change timestamp; on failure or no response within 3 s retry at most 2 further times at intervals of at least 1 s, making no attempt later than 30 s after that timestamp and none once a transmission succeeded, counting all attempts for one transition as one message
    - Process one send operation per transition per recipient in ascending state-history sequence order without overlap; record `notification-send-failed` naming the target state and the final attempt's provider error, mark the key terminal, and leave the node's state history unchanged
    - Deliver without requiring the Shopper_Client to be open or connected, and require no client-supplied state; emit one Cost_Log per send operation with external call count, model call count and estimated cost, each defaulting to zero
    - _Requirements: 11.2, 11.6, 11.7, 11.10, 11.11, 11.12, 11.13, 15.2_

  - [ ]* 18.4 Write property test for notification exactly-once and reached-state-only sending
    - **Property 4: Notification sends only for reached states, exactly once**
    - **Validates: Requirements 11.2, 11.3, 11.4, 11.5, 11.7, 11.12, 11.13, 11.14**
    - File: `mcp/__pbt__/travel-notification-dispatch.pbt.test.mjs`, 200 runs, at most 30 events, at most 3 recipients; event state names drawn from the union of history states and a never-reached set so false-send opportunities are generated rather than assumed absent, with duplicates injected over the concatenated event list

- [ ] 19. Checkpoint — notification holds
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 20. Shopper_Client and Merchant_Client
  - [ ] 20.1 Build the shared travel-agency client primitives and layout rules
    - Create `canvas/src/features/travel-agency/shared/` with the timeline primitives, layout rules and semantic-key usage both sides consume, reusing the existing headless widget primitives and semantic-key helpers and introducing zero forked equivalents
    - Implement the 768px rule once: at or below the existing threshold, one column with exactly one bottom-pinned primary action visible without scrolling; above it, the existing multi-column canvas layout with nothing pinned
    - Express transaction structure with semantic HTML whose native role matches the content role; keep a non-empty accessible name and a keyboard-reachable interaction contract on every selectable media and icon wrapper, and apply `aria-hidden` to no selectable wrapper or element containing one
    - Use browser-PWA capabilities only
    - _Requirements: 14.1, 14.2, 14.3, 14.4, 14.13, 16.5_

  - [ ] 20.2 Implement the shopper client surface and offline confirm outbox
    - Create `canvas/src/features/travel-agency/shopper/` rendering the transaction timeline from the `NodeStateFrame` stream and the Edge_Orchestrator's typed responses
    - Extend the existing Dexie outbox in `canvas/src/lib/storage/` rather than adding a queue: at most one queued confirm per transaction identifier, at most 20 queued confirms, each retained at most 86 400 s, each carrying a stable action identifier assigned at queue time, submitted exactly once with any repeat identifier discarded
    - Cache at most the 20 most recent discovery result sets per session with capture timestamps, retain each at most 24 h, evict oldest first on either bound; show the stale-state indicator within 1 s of the offline signal whether or not a cached set exists, render the cached set and its capture timestamp when one exists, and remove the indicator within 1 s of the online signal
    - Start no automatic payment retry and make no Payment_Call while any confirm is queued; implement `confirm-queue-full`, `confirm-action-expired` and `confirm-submission-exhausted` after 3 attempts on one action identifier, retaining the action byte-identical and requiring explicit human re-confirm
    - _Requirements: 14.5, 14.6, 14.7, 14.8, 14.9, 14.10, 14.11, 14.12, 3.10_

  - [ ] 20.3 Implement the merchant client surface
    - Create `canvas/src/features/travel-agency/merchant/` rendering from the same expanded payload the shopper renders, filtered to `shared` nodes whose merchant party identifier matches the subscriber's membership identifier
    - Display checksum equality as an asserted fact so a mismatch is visible to the operator rather than silent; render `inventory-committed` transitions and the `backed` marking or its withholding reason
    - Present the Path_A guardrail approval as advisory, driven by `settlement.guardrailApprovalAdvisory`, and present no indication that a budget ceiling is enforced; implement `stale-state`, `session-expired`, `no-escrow-commitment` and `window-mismatch`
    - Share the primitives from task 20.1 with no forked equivalent
    - _Requirements: 9.10, 13.3, 14.1, 14.2, 14.13, 16.5_

  - [ ]* 20.4 Write property test for accessible-name retention on selectable wrappers
    - **Property 16: Selectable wrappers keep an accessible name**
    - **Validates: Requirements 14.3, 14.4**
    - File: `canvas/src/__pbt__/travel-agency-accessible-structure.pbt.test.ts`, 200 runs; the generator produces node states across every scope, settlement path, gate outcome, escrow state and blocked-state combination, including empty offer lists and long non-ASCII strings

  - [ ]* 20.5 Write layout example tests at the 768px threshold
    - One narrow render and one wide render per client: single column with exactly one bottom-pinned action visible without scrolling below the threshold, existing multi-column layout with nothing pinned above it
    - File: `canvas/src/__tests__/travelAgencyLayout.test.ts`
    - _Requirements: 14.1, 14.2_

  - [ ]* 20.6 Write rendering assertions for the B1 advisory presentation
    - Assert that for every Path_A settlement option and Path_A transaction state, both clients render the guardrail approval as advisory and render no string, badge, icon or affordance implying a budget ceiling is enforced; assert the assertion fails if `guardrailApprovalAdvisory` is dropped from the payload
    - File: `canvas/src/__tests__/travelAgencyPathAAdvisory.test.ts`
    - _Requirements: 9.9, 9.10_

  - [ ]* 20.7 Write offline confirm-queue and cache-bound tests
    - Use the existing `fake-indexeddb` dependency to assert the 20-confirm cap, the one-per-transaction rule, the 86 400-second expiry, exactly-once submission per action identifier, the 3-attempt exhaustion path, the 20-result-set cache with oldest-first eviction, the 24-hour retention, and that no Payment_Call is made while a confirm is queued
    - File: `canvas/src/__tests__/travelAgencyOfflineConfirmQueue.test.ts`
    - _Requirements: 14.5, 14.9, 14.10, 14.11, 14.12, 3.10_

- [ ] 21. Webhook normalization and orchestrator wiring
  - [ ] 21.1 Extend the existing normalization pipeline with travel-agency event kinds
    - Modify `cloudflare/workers/knowgrph-storage/mutationProcessor.ts` and `cloudflare/workers/knowgrph-payment/paymentEventIngress.ts` to emit `NormalizedCanvasEvent { nodeId, transactionId, eventKind, stateName, recordedAtMs, sequenceIndex }` for the travel-agency kinds, adding no second normalization pipeline
    - Return `callback-signature-invalid` and `callback-unmapped-kind` as typed values
    - _Requirements: 11.1_

  - [ ] 21.2 Wire the orchestrator routes and close the loop end to end
    - Modify `cloudflare/workers/knowgrph-payment/index.ts` to route `TravelAgencyRequest` to `travelAgency/orchestrator.ts`, dispatching Intent_Parser, the discovery harnesses, Offer_Scorer, Guardrail_Gate, Confirmation_Gate, Issuance_Service, Settlement_Verifier, Wallet_Linking_Service and Escrow_Meter, and writing Shared_Canvas_Node deltas through the storage Worker
    - Connect the normalized state-change stream to the Notification_Dispatcher so no component built above is left unintegrated
    - Return `configuration-missing` before any first outbound call when a required key is absent, leaving other components' calls unaffected
    - _Requirements: 16.3, 16.6, 16.7_

- [ ] 22. Cost observability and readiness derivation
  - [ ] 22.1 Implement the cost measurement window and its reports
    - Create `cloudflare/workers/knowgrph-payment/travelAgency/costObservability.ts` summing the estimated cost of every Cost_Log in the current calendar-month window normalized to the configured session-volume baseline, comparing against the configured token-cost and infrastructure ceilings, and marking the measurement incomplete while any invocation in the window lacks a log
    - Record an overrun report within 60 s of the Cost_Log that breaches a ceiling, naming the exceeded ceiling, the measured value and the window; record a time-to-value breach report naming the exceeded ceiling, the recorded value and the session identifier, retaining that session's provenance entries
    - Record each settled session's step count as its provenance entry count and its elapsed duration in seconds from the intent entry to the settled entry
    - _Requirements: 15.4, 15.5, 15.6, 15.7, 15.8_

  - [ ]* 22.2 Write property test for Cost_Log emission and zero-model discipline
    - **Property 12: Cost_Log emission and zero-model discipline**
    - **Validates: Requirements 1.12, 1.13, 2.11, 4.8, 5.8, 6.13, 11.11, 15.1, 15.2, 15.3, 15.5, 15.9**
    - File: `contracts/__pbt__/travel-cost-log-discipline.pbt.test.mjs`, 200 runs, at most 30 operations, with `forceModelCall` driving the zero-model defect branch and `suppressLog` driving the missing-log defect branch, asserting the two defects stay distinct

  - [ ] 22.3 Implement evidence-derived readiness and the deploy boundary register
    - Create `docs/travel-agency/readinessDerivation.mjs` deriving each component's local and delivered rung from recorded Evidence References alone, treating a reference as recorded only where all five parts exist and a rollback statement as recorded only where all three parts exist
    - Hold every component at `spec-complete` / `undocumented` while no complete Evidence Reference exists; resolve the sandbox-to-production issuance boundary to `closed` unless every opening condition is present, and open it for exactly the authorized candidate when they are; derive no delivered rung for the Prod mirror or the Cloudflare route
    - Discard any hand-edited rung or boundary value, restore the derived value, and report a defect naming the component, the discarded value and the restored value; hold the B1 item at `spec-complete` / `undocumented` with no path to a higher rung
    - _Requirements: 17.1, 17.2, 17.4, 17.5, 17.6, 17.7, 17.8, 17.9, 17.10_

  - [ ]* 22.4 Write property test for evidence-derived rungs and fail-closed boundaries
    - **Property 15: Evidence-derived rungs and fail-closed deploy boundaries**
    - **Validates: Requirements 17.1, 17.2, 17.4, 17.5, 17.6, 17.7, 17.8, 17.9, 17.10**
    - File: `docs/__pbt__/travel-readiness-derivation.pbt.test.mjs`, 200 runs, with generators that drop random subsets of the required Evidence Reference and rollback-statement parts and a tamper generator for hand-edited stored values

  - [ ]* 22.5 Write structural smoke assertions
    - Extend `__smoke__/model-provider-key-scan.smoke.test.mjs` so a scan of tracked source returns zero machine paths, credentials, account identifiers, provider catalogue entries and environment-specific defaults across every file added by this increment, each such value referenced by configuration key name only
    - Extend `scripts/check-hygiene-compliance.mjs` coverage to the new directories so every authored file stays at or below 600 lines and 500 KiB
    - Add smoke assertions for: shared transaction nodes using the Durable Object canvas-room WebSocket as their only transport; exactly one authentication path with no second one introduced; every outbound platform call going through the already-provisioned Workers, Durable Objects, D1 and R2 bindings only; and the issuance endpoint resolving to the sandbox value with zero production endpoint bindings
    - File: `cloudflare/workers/knowgrph-storage/__smoke__/travel-agency-structure.smoke.test.mjs`
    - _Requirements: 1.8, 13.5, 14.13, 15.4, 16.1, 16.4, 16.5, 17.3_

- [ ] 23. Final checkpoint — full suite green
  - Ensure all tests pass, ask the user if questions arise.
  - All sixteen properties, the unit and integration suites, and the structural smoke assertions run together via `runtime:test` with the new `__pbt__` directories registered.

## Notes

- Tasks marked with `*` are optional test sub-tasks and can be skipped for a faster MVP. Core implementation tasks are never optional.
- Task 3 is a hard gate, not a formality. It exists because design risk R1 makes a late checksum defect a rewrite rather than a fix.
- Blocked items B1, B2 and B3 are implemented as honest boundaries, never resolved: B1 records an advisory flag and asserts no surface implies enforcement, B2 stays a typed `amount-exceeds-per-card-cap` error with no split-funding path, B3 keeps the production issuance boundary `closed` with zero production endpoint bindings. None of the eleven open questions is resolved by any task here.
- No task publishes to the Prod mirror, deploys to Cloudflare, gathers metrics from a live run, or performs user or stakeholder testing. Those are gated deployment targets and out-of-scope activities, not acceptance criteria.
- No task writes an endpoint, credential, cap, ceiling or retry bound into repository source. `wrangler.toml` may declare a key name and never a value.
- Property tests validate the universal statements; unit, integration and smoke tests carry the work the design deliberately kept out of them — the 768px layout examples, the `backed`-marking decision table, the SSE wiring against a mocked SSE server, and the source-hygiene, file-cap, single-transport, single-auth-path, binding-set and sandbox-endpoint assertions.

## Task Dependency Graph

```mermaid
flowchart LR
  subgraph W0["Wave 0 — parallel, no prerequisites"]
    T11["1.1 node-delta contract"]
    T12["1.2 checksum + excluded fields"]
    T15["1.5 register __pbt__ dirs"]
    T41["4.1 0012 migration"]
    T42["4.2 chain-id allowlist"]
    T111["11.1 sse transport"]
    T101["10.1 offer schema"]
  end
  subgraph W1["Wave 1"]
    T13["1.3 CRDT merge"]
    T43["4.3 migration tests"]
    T51["5.1 side resolution"]
    T61["6.1 config resolver"]
    T102["10.2 intent parser"]
  end
  subgraph W2["Wave 2"]
    T14["1.4 persistence + rehydration"]
    T21["2.1 node cost log"]
    T52["5.2 header propagation"]
    T103["10.3 flight discovery"]
    T105["10.5 shopping discovery"]
    T121["12.1 x402 payer"]
    T131["13.1 wallet linking"]
  end
  subgraph W3["Wave 3"]
    T16["1.6* Property 1"]
    T22["2.2 DO dispatch seam"]
    T24["2.4 subscription filtering"]
    T26["2.6 provenance chain"]
    T53["5.3* side unit tests"]
    T62["6.2* Property 9"]
    T104["10.4* Property 5"]
    T106["10.6 offer scorer"]
    T122["12.2 wallet signing"]
    T132["13.2* Property 10"]
    T141["14.1 second chain source"]
  end
  subgraph W4["Wave 4"]
    T23["2.3* Property 13"]
    T25["2.5* Property 7"]
    T27["2.7* Property 8"]
    T71["7.1 guardrail gate"]
    T107["10.7* Property 6"]
    T108["10.8* harness unit tests"]
    T123["12.3 issuance service"]
    T142["14.2 settlement verifier"]
    T161["16.1 escrow meter"]
  end
  subgraph W5["Wave 5"]
    T72["7.2* Property 3"]
    T73["7.3 probe.evolve scoring"]
    T81["8.1 confirmation gate"]
    T124["12.4* SSE integration"]
    T125["12.5* issuance errors"]
    T143["14.3* Property 14"]
    T162["16.2* backed table"]
    T181["18.1 suppression store"]
  end
  subgraph W6["Wave 6"]
    T74["7.4* guardrail unit tests"]
    T82["8.2* Property 2"]
    T171["17.1 settlement path"]
    T182["18.2 state verification"]
    T201["20.1 shared primitives"]
  end
  subgraph W7["Wave 7"]
    T83["8.3* Property 11"]
    T172["17.2 Path_A orchestration"]
    T183["18.3 dispatcher"]
    T202["20.2 shopper client"]
  end
  subgraph W8["Wave 8"]
    T173["17.3 advisory flag"]
    T184["18.4* Property 4"]
    T203["20.3 merchant client"]
    T211["21.1 event kinds"]
  end
  subgraph W9["Wave 9"]
    T174["17.4* Path_A unit tests"]
    T204["20.4* Property 16"]
    T205["20.5* layout examples"]
    T206["20.6* B1 rendering"]
    T207["20.7* offline queue"]
    T212["21.2 orchestrator wiring"]
  end
  subgraph W10["Wave 10"]
    T221["22.1 cost window"]
    T223["22.3 readiness derivation"]
  end
  subgraph W11["Wave 11"]
    T222["22.2* Property 12"]
    T224["22.4* Property 15"]
    T225["22.5* smoke assertions"]
  end

  W0 --> W1 --> W2 --> W3 --> W4 --> W5 --> W6 --> W7 --> W8 --> W9 --> W10 --> W11
  CP3["Checkpoint 3 — foundation gate"]
  W4 --> CP3
  CP3 --> W5
```

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2", "1.5", "4.1", "4.2", "10.1", "11.1"] },
    { "id": 1, "tasks": ["1.3", "4.3", "5.1", "6.1", "10.2"] },
    { "id": 2, "tasks": ["1.4", "2.1", "5.2", "10.3", "10.5", "12.1", "13.1"] },
    { "id": 3, "tasks": ["1.6", "2.2", "2.4", "2.6", "5.3", "6.2", "10.4", "10.6", "12.2", "13.2", "14.1"] },
    { "id": 4, "tasks": ["2.3", "2.5", "2.7", "7.1", "10.7", "10.8", "12.3", "14.2", "16.1"] },
    { "id": 5, "tasks": ["7.2", "7.3", "8.1", "12.4", "12.5", "14.3", "16.2", "18.1"] },
    { "id": 6, "tasks": ["7.4", "8.2", "17.1", "18.2", "20.1"] },
    { "id": 7, "tasks": ["8.3", "17.2", "18.3", "20.2"] },
    { "id": 8, "tasks": ["17.3", "18.4", "20.3", "21.1"] },
    { "id": 9, "tasks": ["17.4", "20.4", "20.5", "20.6", "20.7", "21.2"] },
    { "id": 10, "tasks": ["22.1", "22.3"] },
    { "id": 11, "tasks": ["22.2", "22.4", "22.5"] }
  ]
}
```
