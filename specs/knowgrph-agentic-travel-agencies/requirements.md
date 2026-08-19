# Requirements Document

## Introduction

This feature enhances the existing `knowgrph` codebase with the Agentic Travel Agencies capability specified in `knowgrph-agentic-travel-agencies-prd-tad-adr.md` v0.5.0. It delivers two flagship agent-mediated commerce flows — a SIN→KUL flight booking and a comparison-shopping purchase — on top of one new primitive: a **shared-canvas node** that a shopper-side human and a merchant-side human read from a single source rather than from two mirrored copies.

Scope is derived from US-1 through US-6 of the source specification and their VCC translations. Three gates carry the user-facing value: a server-side **budget guardrail gate** that blocks payment-instrument issuance above a session-configured ceiling and retries flexible dates a bounded number of times; a mandatory **human confirmation gate** that no payment call may precede; and a **shared-canvas node** whose expanded payload is checksum-identical across the shopper and merchant subscriptions.

Two settlement paths are specified. **Path B** routes value through StraitsX custody and requires wallet linking plus card issuance through an MCP-over-SSE gateway funded by a gasless x402/EIP-3009 self-custody signature. **Path A** settles on-chain directly from a self-custodied wallet with zero StraitsX calls and requires the wallet to hold Fuji AVAX for gas. Path A's guardrail-enforcement half has **no server-side binding point** and is deliberately carried forward as a blocked item, not as a working guarantee (see Non-Goals and Blocked Items).

Vendor names throughout are reference implementations behind typed, provider-agnostic interfaces, not hardcoded couplings. The increment is sandbox-only: production issuance integration, Prod-mirror publication, and Cloudflare deployment are outside these acceptance criteria.

## Glossary

### Process and readiness terms

- **VCC (Verifiable Completion Condition)**: An observable, log-inspectable assertion that proves an acceptance criterion held during an actual run. A VCC names what to inspect and what must be true, not how the code works.
- **Rung**: A discrete readiness level a component holds, derived from recorded Evidence References rather than asserted. Local rungs used here: `spec-complete` (specified, no run evidence), `dev-proven` (evidenced in Dev), `runtime-ready` (typed inputs and outputs, bounded orchestration, cost evidence, fallbacks, explicit gates). Delivered rungs used here: `undocumented`, `runtime-ready`.
- **Evidence Reference**: A recorded artifact from an actual run that a rung claim is derived from. Absent an Evidence Reference, a rung claim is an unproven claim.
- **Lane**: An authoring, mirror, or delivery stage a component's work occupies. Authoring is the Dev repository; Mirror is generated release output; Delivery is a published surface.
- **Fence**: The monotonic lease epoch plus claim commit SHA that identifies one writer's exclusive authority over a semantic scope. Used by the repository's session-start contract, not by this feature's runtime.
- **Deploy Boundary**: A named transition between lanes that stays `closed` until an Evidence Reference, an operator instruction, and a rollback statement all exist for it.

### Domain and component terms

- **Travel_Agency_System**: The whole feature under specification, used as the system name for criteria that constrain the increment rather than one component.
- **Shopper_Agent_Principal**: The human on whose behalf an agent transacts within a stated budget.
- **Merchant_Operator**: The human receiving agent-initiated traffic who enforces policy and resolves disputes.
- **Shopper_Client**: The shopper-side browser PWA surface.
- **Merchant_Client**: The merchant-side browser PWA surface.
- **Edge_Orchestrator**: The existing edge router that dispatches Discovery, the Guardrail_Gate, and the Shared_Canvas_Node_Store, and that selects a settlement path.
- **Session_Authority**: The existing server-side component that validates a session token and resolves the caller's workspace role.
- **External_Transport_Contract**: The existing external MCP gateway contract that enumerates admitted transport types.
- **Cost_Log_Validator**: The existing cost-log schema validator that reports a defect when a Cost_Log violates its component's declared cost shape.
- **Shared_Canvas_Node**: A single CRDT-backed transaction record written once and subscribed to by both the shopper client and the merchant client. Not a pair of mirrored per-side records.
- **Shared_Canvas_Node_Store**: The edge component that owns Shared_Canvas_Node persistence, merge, and dual-read subscription.
- **Node_Payload_Checksum**: A deterministic digest computed over the canonically serialized expanded payload of one Shared_Canvas_Node, used to compare the shopper-side and merchant-side renderings of the same node identity.
- **Node_Scope**: The visibility classification of a canvas node, either `personal` (visible to one side only) or `shared` (visible to both sides).
- **Intent_Parser**: The component that converts a free-text request into a typed intent carrying route, dates, item, budget ceiling, or price ceiling.
- **Flight_Discovery_Harness**: The typed harness that queries bookable flight fares and normalizes them into the internal offer schema. Reference implementation: Atlas API (aTriptech).
- **Shopping_Discovery_Harness**: The typed harness that queries multiple product sources in parallel and normalizes results for scoring. Reference implementations: eBay Browse API and PricesAPI.
- **Offer_Scorer**: The fan-in component that merges multiple normalized offer lists into one ranked, scored list.
- **Guardrail_Gate**: The deterministic server-side component that evaluates a normalized offer against the session budget or cart-total policy and emits `pass`, `block`, or `retry`.
- **Budget_Ceiling**: The session-configured maximum approved amount for a transaction. The flagship flight scenario configures SGD 500.
- **Retry_Bound**: The session-configured maximum number of bounded flexible-date retry attempts the Guardrail_Gate may trigger. The flagship flight scenario configures 3.
- **Confirmation_Gate**: The component that records an explicit human confirmation event and is the only authority that unblocks a payment call.
- **Human_Confirm_Event**: The recorded session-log event proving a human performed the confirm action on the summary sheet.
- **Payment_Call**: Any outbound call to a payment or card-issuance interface that can move value or create a spendable instrument, including Issuance_Service tool calls and wallet signature requests.
- **Issuance_Service**: The MCP harness that issues a single-use virtual card scoped to the guardrail-approved amount. Reference implementation: StraitsX Card MCP Gateway over SSE, sandbox endpoint `card.straitsx.ai/sandbox/sse`.
- **Per_Card_Cap**: The gateway-enforced maximum amount a single issued card may carry. Sandbox range is 5–30 SGD; production range is 5–50 USD.
- **x402**: The HTTP 402 Payment Required challenge-response flow by which an unpaid issuance call returns a payment challenge that the caller must satisfy before retrying.
- **EIP-3009**: The `transferWithAuthorization` signature standard used to authorize a stablecoin transfer off-chain. The signature that funds Issuance_Service is **gasless for the signer** because the gateway's relayer submits the transaction and pays gas.
- **Self_Custody_Wallet_Interface**: The device-local EVM signing interface outside the trust boundary of this system. Reference implementation: Core.app (Core Wallet).
- **Path_A**: On-chain-direct settlement. The Self_Custody_Wallet_Interface signs and broadcasts a transfer straight to a merchant address, zero StraitsX calls occur, and the wallet must hold Fuji AVAX for gas.
- **Path_B**: StraitsX-mediated settlement. Value routes through StraitsX custody, requires prior wallet linking when the source is self-custodied, and proceeds through Issuance_Service.
- **Wallet_Linking_Service**: The edge component that registers a self-custody wallet address against a verified StraitsX Customer Profile, required before a Path_B inbound transfer is attributable to a user.
- **Settlement_Verifier**: The read-only harness that confirms on-chain finality of a settlement transaction against two mutually independent sources. Reference implementations: Avalanche Data API and Snowtrace API.
- **Escrow_Meter**: The component that records an on-chain escrow commitment matching a fare-hold window so a Merchant_Operator can mark an offer as backed before committing inventory.
- **Hold_Window**: The stated expiry duration of a fare hold returned by the flight booking interface.
- **Webhook_Normalizer**: The existing pipeline role that converts a raw external callback into a normalized canvas event.
- **Notification_Dispatcher**: The harness that consumes normalized Shared_Canvas_Node state-change events and sends an out-of-canvas message. Reference implementation: Telegram Bot API.
- **Notified_State**: A Shared_Canvas_Node state that triggers an out-of-canvas notification. The configured set is `confirmed`, `failed`, and `disputed`.
- **Provenance_Logger**: The component that appends hash-linked entries recording each gate decision, payment call, and settlement event for one transaction.
- **probe.evolve**: The existing bounded probe-tree MCP tool (`knowgrph.probe.evolve`) that scores a resolved probe path and writes one reusable exemplar into a scoped memory layer. Used here as the retry-scoring surface for bounded flexible-date retries.
- **Cost_Log**: The typed per-call record of model usage, external call count, and estimated cost, conforming to the repository's existing cost-log schema.

## Substrate Baseline and Divergences

The source specification asserts several capabilities as already-adopted dependencies. Codebase inspection confirms two and refutes three. Requirements below are written against the verified state, using "extend" only where substrate genuinely exists.

| Source specification assumption | Verified state | Consequence for these requirements |
|---|---|---|
| Yjs CRDT runs inside Cloudflare Durable Objects | **Refuted.** Yjs exists client-side only (`grph-shared/src/collaboration/yjsSnapshot.ts`) over a PocketBase room transport. The single Durable Object class `KnowgrphCanvasSyncRoom` imports no Yjs, broadcasts full document text rather than CRDT deltas, and persists asset records only. The collaboration-save contract field `yjsStateBase64` is written as an empty string. | Shared_Canvas_Node_Store extends the existing Durable Object and reuses the existing Yjs snapshot helpers, but joining CRDT merge to Durable Object persistence is net-new work, not a new consumer of a finished dependency. Requirement 1 states this explicitly. |
| A `table_name:record_id` persistent-storage key pattern is established | **Refuted.** Real patterns are Durable Object `asset:{workspaceId}:{roomId}:{artifactId}`, IndexedDB `{namespace}\u0000{id}`, and relational D1 tables with composite uniqueness. | Requirement 1 binds Shared_Canvas_Node keys to the existing Durable Object key convention rather than inventing a third. |
| Better Auth is adopted and can carry role-scoped sessions | **Refuted on the named dependency; the capability exists under another name.** Auth is hand-rolled: SHA-256-hashed bearer sessions in `auth_sessions`, workspace-scoped roles in `workspace_memberships`, role union `viewer \| editor \| owner \| provider-admin`, propagated into the Durable Object as an `x-knowgrph-room-role` header. | Requirement 13 extends the existing role model with a shopper/merchant side dimension. No Better Auth migration is in scope. |
| The system is already MCP-native with a probe-tree that speaks MCP | **Confirmed.** `knowgrph.probe.{generate,select,evolve}` exist with full input and output JSON Schemas and bounded defaults (`maxDepth 8`, 2–4 options, `tokenBudget 1200`), plus a local stdio server, a remote Streamable HTTP worker, browser tools, and an external gateway contract. | Requirements 2 and 6 extend existing MCP substrate. The external gateway transport enum currently admits `stdio` and `streamable-http` only; Requirement 6 requires adding SSE. |
| The system is a mobile-first PWA | **Partially confirmed.** A real installable PWA exists (Workbox precache, service-worker revision authority, Dexie local-first engine with outbox). Mobile-first layout does not; only a 768px viewport threshold and responsive class helpers sit over a desktop-scale canvas. | Requirement 14 treats offline behaviour as an extension and single-column mobile layout as net-new. |

Additional verified divergences that shape these requirements: three collaboration transports already coexist (Durable Object canvas room, PocketBase Yjs rooms, and a peer-to-peer module), so Requirement 1 must name one; `canvas/tsconfig.json` sets `strict: false`, so typed-contract requirements are enforced by schema validation rather than by the compiler alone; a 600-line and 500-KiB per-file cap is CI-enforced; `GraphNode.type` is an open string with widget types registered through authored frontmatter data rather than a TypeScript union; a payments substrate already carries `stripe` and `straitsx` rails, `fiat` and `xsgd` settlement assets, Avalanche chain evidence, and `@x402/*` dependencies; `fast-check` 3.23.2 is already a devDependency with an established `__pbt__` directory convention; and no notification service or Telegram code exists at all.

## Requirements

### Requirement 1: Shared-Canvas Node Primitive (US-3, Must)

**User Story:** As a Merchant_Operator, I want to see the identical transaction timeline my customer's agent sees, so that disputes resolve without re-explaining context across two systems.

#### Acceptance Criteria

1. THE Shared_Canvas_Node_Store SHALL persist each transaction node identity as exactly one stored record, SHALL create no per-side copy of that record, and SHALL serve both the shopper-side reader and the merchant-side reader from that single record.
2. WHEN a shopper-side writer and a merchant-side writer both submit a state change for one node identity, THE Shared_Canvas_Node_Store SHALL merge both changes into the single record for that node identity and SHALL produce, for either arrival order of those two changes, expanded payloads whose Node_Payload_Checksum values are equal.
3. WHEN 2 seconds have elapsed since the Shared_Canvas_Node_Store last accepted a delta for one node identity whose Node_Scope is `shared`, THE Shared_Canvas_Node_Store SHALL return, for a fetch through the shopper-side subscription and a fetch through the merchant-side subscription of that node identity, expanded payloads whose Node_Payload_Checksum values are equal. THE Shared_Canvas_Node_Store SHALL compute Node_Payload_Checksum for both subscriptions with one deterministic digest function over one canonical serialization of the expanded payload, excluding subscriber-specific fields from that serialization.
4. THE Shared_Canvas_Node_Store SHALL assign each node a Node_Scope of `personal` or `shared`, SHALL default intent and draft nodes to `personal`, and SHALL promote a node to `shared` when a Discovery result set is recorded against that node. THE Shared_Canvas_Node_Store SHALL retain Node_Scope `shared` for the remaining lifetime of a promoted node.
5. WHILE a node holds Node_Scope `personal`, THE Shared_Canvas_Node_Store SHALL exclude that node from the merchant-side subscription.
6. THE Shared_Canvas_Node_Store SHALL extend the existing `KnowgrphCanvasSyncRoom` Durable Object and SHALL reuse the existing Yjs snapshot helpers in `grph-shared/src/collaboration/` as its merge implementation. THE Shared_Canvas_Node_Store SHALL treat the join between CRDT merge and Durable Object persistence as net-new work rather than as an available dependency, and SHALL persist a non-empty serialized CRDT state for every node identity that has accepted at least one delta.
7. THE Shared_Canvas_Node_Store SHALL derive node storage keys from the existing Durable Object key convention `{recordKind}:{workspaceId}:{roomId}:{recordId}`.
8. THE Shared_Canvas_Node_Store SHALL use the Durable Object canvas-room WebSocket as its only transport for shared transaction nodes.
9. THE Shared_Canvas_Node_Store SHALL validate a submitted delta against the typed node-delta schema in full before applying any part of that delta to the stored record.
10. IF a submitted delta fails schema validation, THEN THE Shared_Canvas_Node_Store SHALL reject the delta, SHALL leave the stored record byte-identical to its pre-submission state, SHALL leave that node's Node_Payload_Checksum unchanged, and SHALL return a typed validation error naming the failing field path.
11. WHEN a subscriber reconnects after a disconnection of at most 300 seconds, THE Shared_Canvas_Node_Store SHALL replay, within 10 seconds of reconnection, every delta accepted for that subscriber's visible nodes during the disconnection, so that the reconnected subscriber's Node_Payload_Checksum equals the other side's checksum for the same node identity.
12. THE Shared_Canvas_Node_Store SHALL record one Cost_Log per operation it performs, including merges, individual writes, rejected deltas, reconnection replays, and snapshot serves.
13. THE Shared_Canvas_Node_Store SHALL record zero model calls and zero estimated model cost in the Cost_Log of every operation it performs, including every merge operation.
14. WHEN the Durable Object hosting a node identity restarts, THE Shared_Canvas_Node_Store SHALL rehydrate that node from its persisted CRDT state and SHALL return an expanded payload whose Node_Payload_Checksum equals the checksum of that node identity recorded immediately before the restart.
15. IF a subscriber reconnects after a disconnection longer than 300 seconds, THEN THE Shared_Canvas_Node_Store SHALL serve a full expanded-payload snapshot for that subscriber's visible nodes in place of a delta replay, and SHALL return for each served node a Node_Payload_Checksum equal to the other side's checksum for the same node identity.
16. IF a submitted delta exceeds the configured maximum delta size of 65,536 bytes, or applying it would raise the node's expanded payload above the configured maximum of 524,288 bytes, THEN THE Shared_Canvas_Node_Store SHALL reject the delta, SHALL leave the stored record byte-identical to its pre-submission state, and SHALL return a typed `delta-limit-exceeded` error naming the exceeded limit and its configured value.

### Requirement 2: Budget Guardrail Gate with Bounded Retry (US-1, Must)

**User Story:** As a Shopper_Agent_Principal, I want my agent to book a flight within a stated budget and never exceed it, so that I do not discover overspend after the fact.

#### Acceptance Criteria

1. THE Guardrail_Gate SHALL read Budget_Ceiling and Retry_Bound from session-scoped configuration supplied at request time, SHALL treat an admissible Budget_Ceiling as a positive amount carrying a currency and an admissible Retry_Bound as a non-negative integer, and SHALL hold no Budget_Ceiling value and no Retry_Bound value in repository source or in component defaults.
2. WHEN the Guardrail_Gate evaluates a normalized offer whose total amount, inclusive of every tax and mandatory fee stated in that offer, is greater than Budget_Ceiling, THE Guardrail_Gate SHALL emit a `block` result and SHALL make zero Payment_Call invocations for that offer.
3. WHEN the Guardrail_Gate emits a `block` result for a flight offer and the server-held count of completed retry attempts for that session is strictly less than Retry_Bound, THE Guardrail_Gate SHALL emit a `retry` result carrying an adjusted flexible-date intent whose dates lie within the date range of the originating typed intent and whose date set differs from every date set already evaluated in that session, and SHALL increase the server-held count of completed retry attempts for that session by exactly one.
4. WHEN the server-held count of completed retry attempts for a session equals Retry_Bound, THE Guardrail_Gate SHALL emit a terminal `block` result, SHALL emit no further `retry` result for that session, and SHALL return at most the 5 lowest-total-amount offers evaluated in that session, ranked in ascending total amount, to the Shopper_Agent_Principal.
5. WHERE Retry_Bound is configured as zero, THE Guardrail_Gate SHALL emit a terminal `block` result on the first over-budget offer of that session and SHALL emit no `retry` result for that session.
6. THE Guardrail_Gate SHALL record one Provenance_Logger entry per gate decision containing the evaluated offer identifier, the evaluated total amount with its currency, the Budget_Ceiling in force with its currency, the Retry_Bound in force, the emitted result, and the zero-based retry attempt index.
7. WHEN the Guardrail_Gate emits a `retry` result, THE Guardrail_Gate SHALL invoke `probe.evolve` to score the resolved retry path and SHALL record the returned exemplar identifier in the Provenance_Logger entry for that decision.
8. THE Guardrail_Gate SHALL evaluate every offer server-side within the edge runtime, SHALL derive no part of its decision from client-supplied gate state, and SHALL complete each gate decision within 1 second of receiving a normalized offer.
9. IF a request carries a client-supplied gate result, a client-supplied approved amount, or a client-supplied retry attempt count, THEN THE Guardrail_Gate SHALL discard every such submitted value, SHALL re-evaluate the offer from the session-scoped configuration and the server-held retry attempt count, and SHALL record each discarded value in the Provenance_Logger entry for that decision.
10. WHEN the Guardrail_Gate evaluates a normalized offer whose total amount is less than or equal to Budget_Ceiling, THE Guardrail_Gate SHALL emit a `pass` result and SHALL record the approved amount, its currency, the approved offer identifier, and the gate decision identifier that downstream components must match exactly.
11. THE Guardrail_Gate SHALL record a Cost_Log of zero model calls and zero estimated cost for every gate decision.
12. IF the Budget_Ceiling or Retry_Bound resolved for a session falls outside the admissible value domain stated in criterion 1, THEN THE Guardrail_Gate SHALL return a typed `invalid-guardrail-configuration` error naming the offending value, SHALL evaluate no offer for that session, and SHALL make zero Payment_Call invocations for that session.
13. IF an evaluated offer's total-amount currency differs from the Budget_Ceiling currency, THEN THE Guardrail_Gate SHALL emit a `block` result for that offer, SHALL return a typed `currency-mismatch` error naming both currencies, and SHALL make zero Payment_Call invocations for that offer.
14. IF an invocation of `probe.evolve` returns an error or returns no exemplar identifier within 5 seconds, THEN THE Guardrail_Gate SHALL record a `retry-scoring-unavailable` reason in place of the exemplar identifier in that decision's Provenance_Logger entry, SHALL emit the `retry` result determined by criterion 3 unchanged, and SHALL count that attempt toward Retry_Bound.

### Requirement 3: Mandatory Human Confirmation Before Payment (US-2, Must)

**User Story:** As a Shopper_Agent_Principal, I want my agent to compare purchases across stores and pause for my explicit confirmation before paying, so that no purchase happens without my approval.

#### Acceptance Criteria

1. WHEN an offer is added to the cart, THE Confirmation_Gate SHALL enter a pending state for that transaction, SHALL bind that pending state to the amount recorded by the Guardrail_Gate `pass` result for that transaction, and SHALL record a cart-add event in the session log carrying the transaction identifier and a recording timestamp expressed in UTC at millisecond precision.
2. WHILE a transaction's Confirmation_Gate is pending, THE Confirmation_Gate SHALL reject every Payment_Call request for that transaction with a typed `confirmation-required` error and SHALL make zero outbound payment-interface or wallet-signature invocations for that request.
3. WHEN a human performs the confirm action on the summary sheet, THE Confirmation_Gate SHALL record a Human_Confirm_Event carrying the transaction identifier, an approved amount equal to the amount recorded by the Guardrail_Gate `pass` result for that transaction, and a recording timestamp expressed in UTC at millisecond precision.
4. THE Confirmation_Gate SHALL permit a Payment_Call for a transaction only after a valid Human_Confirm_Event exists for that transaction in the same session log.
5. THE Confirmation_Gate SHALL order the session log so that every accepted Payment_Call for a transaction holds a session-log position strictly greater than the position of that transaction's most recent valid Human_Confirm_Event, and SHALL accept no Payment_Call for which such a position cannot be assigned.
6. WHERE a transaction settles through Path_A, THE Confirmation_Gate SHALL treat the wallet signature request as a member of the Payment_Call set and SHALL assign that request a session-log position strictly greater than the position of that transaction's most recent valid Human_Confirm_Event.
7. IF the amount recorded by the Guardrail_Gate `pass` result for a transaction differs from the approved amount carried by that transaction's most recent valid Human_Confirm_Event by at least one minor currency unit (0.01), or differs in currency, THEN THE Confirmation_Gate SHALL invalidate that Human_Confirm_Event, SHALL return the transaction to the pending state, and SHALL require a new Human_Confirm_Event carrying the changed amount before permitting any Payment_Call.
8. THE Confirmation_Gate SHALL enforce the pending state server-side within the edge runtime, SHALL treat the server-held confirmation state as the sole authority for that decision, and SHALL derive no part of that decision from client-supplied confirmation state.
9. IF a request carries a client-supplied confirmation state that differs from the server-held confirmation state for the same transaction, THEN THE Confirmation_Gate SHALL reject the request with a typed `confirmation-state-mismatch` error, SHALL leave the server-held confirmation state unchanged, and SHALL record both states in the Provenance_Logger entry.
10. WHILE the Shopper_Client has no network connectivity, THE Shopper_Client SHALL queue at most one confirm action per transaction identifier up to a maximum of 20 queued confirm actions, SHALL retain each queued confirm action for at most 86400 seconds, SHALL send exactly one confirm submission per retained queued action after connectivity is restored, and SHALL discard a confirm action that exceeds either bound with an indication to the Shopper_Agent_Principal that re-confirmation is required.
11. THE Confirmation_Gate SHALL read the confirmation validity window from operator-owned configuration as an integer number of seconds in the range 30 to 900 inclusive and SHALL apply 300 seconds when no value is configured.
12. IF a Payment_Call request for a transaction arrives more than the confirmation validity window after the recording timestamp of that transaction's most recent valid Human_Confirm_Event, THEN THE Confirmation_Gate SHALL invalidate that Human_Confirm_Event, SHALL return the transaction to the pending state, and SHALL reject the request with a typed `confirmation-required` error.
13. IF a confirm submission arrives carrying a transaction identifier and an approved amount matching an existing valid Human_Confirm_Event for that transaction, THEN THE Confirmation_Gate SHALL record no additional Human_Confirm_Event, SHALL retain the existing Human_Confirm_Event and its recording timestamp, and SHALL permit no additional Payment_Call beyond the one already permitted for that transaction.

### Requirement 4: Flight Discovery Harness

**User Story:** As a Shopper_Agent_Principal, I want my agent to find genuinely bookable fares for a route and date range, so that the guardrail and confirmation gates apply to a real purchase rather than an estimate.

#### Acceptance Criteria

1. THE Intent_Parser SHALL convert a free-text flight request of at most 2,000 characters into a typed intent carrying origin, destination, a date range whose start date is not earlier than the request date and whose span is at most 90 days, and a Budget_Ceiling stated in one currency with a value from 1 to 1,000,000 inclusive.
2. IF the Intent_Parser cannot resolve origin, destination, date range, or Budget_Ceiling from the request, or the request exceeds 2,000 characters, or the resolved date range violates the bounds in criterion 1, THEN THE Intent_Parser SHALL return a typed `unparseable-intent` error naming every unresolved or out-of-bounds field and SHALL emit no typed intent.
3. WHEN the Flight_Discovery_Harness receives a typed flight intent, THE Flight_Discovery_Harness SHALL return within 30 seconds a fare list of at most 50 fares in which every fare conforms to the internal typed offer schema and carries a provider fare reference, a total amount inclusive of taxes and fees, the currency of that amount, and a stated Hold_Window duration.
4. THE Flight_Discovery_Harness SHALL expose its flight-search provider as a typed interface whose endpoint, credentials, and route catalogue are resolved from operator-owned configuration at each invocation, and SHALL hold no endpoint value, credential value, or route catalogue entry in repository source.
5. WHEN the flight-search provider returns no fare at or below Budget_Ceiling and at least one returned fare carries a total amount comparable in a single currency, THE Flight_Discovery_Harness SHALL return the 5 lowest-amount comparable fares, or all comparable fares when fewer than 5 exist, in ascending amount order, and SHALL mark every returned fare with an `above-budget` flag.
6. IF no returned fare carries a total amount comparable in a single currency, THEN THE Flight_Discovery_Harness SHALL return a typed `unrankable-fares` error naming the count of non-comparable fares and SHALL return no fare list.
7. WHEN the flight-search provider returns a duplicate-booking error, a rate-limit error, or any other provider error, THE Flight_Discovery_Harness SHALL map that provider error to exactly one typed internal error code drawn from a closed set that includes `duplicate-booking`, `rate-limited`, `provider-unavailable`, and `provider-error-unmapped`, and SHALL pass the mapped code to the Guardrail_Gate deduplication check.
8. THE Flight_Discovery_Harness SHALL record exactly one Cost_Log per invocation containing the count of outbound provider probes attempted, the count that resolved, the count that failed or was cancelled, zero model calls, and an estimated cost of zero for the sandbox environment.
9. WHILE outbound probes for one invocation are in flight, THE Flight_Discovery_Harness SHALL hold the count of concurrently open provider probes at no more than 3 and SHALL emit each resolved probe's normalized fares to the Shopper_Client as a partial result within 1 second of that probe resolving.
10. IF a required provider configuration value for endpoint, credentials, or route catalogue is absent or fails validation at invocation time, THEN THE Flight_Discovery_Harness SHALL return a typed `provider-unconfigured` error naming the absent or invalid configuration key, SHALL make zero outbound provider calls, and SHALL return no fare list.
11. IF an outbound provider probe does not resolve within 10 seconds, THEN THE Flight_Discovery_Harness SHALL cancel that probe, SHALL record it as cancelled in the Cost_Log, and SHALL return the fares from the probes that resolved.
12. IF every outbound provider probe fails, is cancelled, or returns zero fares, THEN THE Flight_Discovery_Harness SHALL return an empty fare list carrying a `no-fares-found` reason and SHALL start no retry of that invocation.

### Requirement 5: Comparison Shopping Discovery

**User Story:** As a Shopper_Agent_Principal, I want my agent to compare an item across stores and rank the results, so that I can confirm the best-value option.

#### Acceptance Criteria

1. THE Intent_Parser SHALL convert a free-text shopping request into a typed intent carrying an item description of 1 to 200 characters, a price ceiling in the range 0.01 to 999,999,999.99, and one ranking criterion drawn from the ranking-criteria set supplied by operator-owned configuration.
2. WHEN the Shopping_Discovery_Harness receives a typed shopping intent, THE Shopping_Discovery_Harness SHALL dispatch exactly one request to each configured product source in parallel within a single pass, for at most 10 configured sources.
3. THE Shopping_Discovery_Harness SHALL normalize each source's response into the internal typed offer schema before returning any offer, and SHALL exclude from its returned offers every source offer that fails internal typed offer schema validation.
4. IF one configured product source returns an error, returns a response that fails internal typed offer schema validation, or returns no complete response within 10 seconds of dispatch, THEN THE Shopping_Discovery_Harness SHALL return the offers from the remaining sources, SHALL mark that source as degraded in its Cost_Log, and SHALL send no further request to that source within the same pass.
5. WHEN the Offer_Scorer receives two or more normalized offer lists, THE Offer_Scorer SHALL return one ranked list containing every input offer from every received list exactly once, with no offer omitted and no offer duplicated.
6. WHERE the count of sources that returned schema-valid offers is less than the count of configured sources, THE Offer_Scorer SHALL score the available offers, SHALL mark the returned result as partial, and SHALL record the responding-source count and the configured-source count alongside that marking.
7. THE Shopping_Discovery_Harness SHALL perform exactly one discovery pass per typed shopping intent and SHALL start no retry loop and no retry attempt for any source in that pass.
8. THE Offer_Scorer SHALL record a Cost_Log for every invocation, including an invocation that receives zero offers, containing model identifier, prompt tokens, completion tokens, and estimated cost.
9. IF every configured product source is marked degraded for one pass, THEN THE Offer_Scorer SHALL return an empty ranked list, SHALL mark that result as partial, SHALL record a Cost_Log for that invocation, and SHALL start no additional discovery pass.
10. IF the Intent_Parser cannot resolve an item description or a price ceiling within the stated bounds from the shopping request, THEN THE Intent_Parser SHALL return a typed `unparseable-intent` error naming the unresolved field and THE Shopping_Discovery_Harness SHALL perform no discovery pass for that request.
11. THE Shopping_Discovery_Harness SHALL read its product-source list, per-source endpoints, and per-source credentials from operator-owned configuration at runtime, and SHALL keep every endpoint and credential value outside repository source.

### Requirement 6: Card Issuance Service over MCP/SSE

**User Story:** As a Shopper_Agent_Principal, I want a disposable card scoped to exactly the amount I approved, so that an agent cannot spend beyond my confirmation.

#### Acceptance Criteria

1. THE Issuance_Service SHALL invoke its card-issuance provider as MCP tool calls over an SSE transport whose sandbox endpoint, credentials, and tool names are supplied by operator-owned configuration, and SHALL bound each such tool call to a 30-second response deadline measured from request dispatch to first response frame.
2. THE External_Transport_Contract SHALL admit an `sse` transport type in addition to the existing `stdio` and `streamable-http` types.
3. THE Issuance_Service SHALL issue a card whose scope amount equals the amount recorded by the Guardrail_Gate `pass` result for that transaction, expressed in the same settlement currency and equal to the smallest unit of that currency, and SHALL apply no rounding, buffer, or adjustment to that amount.
4. WHEN an issuance tool call returns an x402 payment challenge, THE Issuance_Service SHALL request an EIP-3009 `transferWithAuthorization` signature from the Self_Custody_Wallet_Interface using only values carried in that challenge, SHALL include no value absent from that challenge, and SHALL retry that tool call exactly once with the returned signature.
5. THE Issuance_Service SHALL treat the EIP-3009 funding signature as gasless for the signer, SHALL require no wallet-held gas balance for the funding step, and SHALL read no wallet native gas balance as a precondition for that step.
6. IF the wallet holder rejects the EIP-3009 signature request, or returns no signature within 120 seconds of the signature request being issued, THEN THE Issuance_Service SHALL return a typed `funding-declined` error, SHALL issue no card, SHALL send no further signature request for that challenge, and SHALL record the declined state on the Shared_Canvas_Node.
7. IF a signed issuance call returns an error, or returns no response within the 30-second tool-call deadline, THEN THE Issuance_Service SHALL propagate a typed error upstream indicating the provider failure or the elapsed deadline and SHALL start no automatic retry of that signed call.
8. THE Issuance_Service SHALL count only successfully issued cards toward the per-transaction issuance limit, SHALL exclude every declined, errored, and deadline-elapsed attempt from that count, and SHALL hold that count at one per transaction unless a `view-lost-reissue` reason is recorded for that transaction.
9. WHERE a previously issued card has no recorded card-credential-view acknowledgement within 300 seconds of issuance and has zero recorded authorizations against it, THE Issuance_Service SHALL treat that card's one-time view as lost, SHALL issue exactly one replacement card scoped to the same guardrail-approved amount, and SHALL record a `view-lost-reissue` reason in the Provenance_Logger entry that distinguishes the replacement from a duplicate-issuance defect.
10. IF the guardrail-approved amount exceeds the configured Per_Card_Cap, THEN THE Issuance_Service SHALL return a typed `amount-exceeds-per-card-cap` error naming the configured cap value and the approved amount, SHALL issue no card, SHALL make zero issuance tool calls, and SHALL start no multi-card or split-funding flow.
11. IF the guardrail-approved amount is at or below the configured Per_Card_Cap, THEN THE Issuance_Service SHALL proceed with issuance for that transaction and SHALL return no `amount-exceeds-per-card-cap` error.
12. THE Issuance_Service SHALL read Per_Card_Cap from operator-owned configuration at request time, SHALL keep every cap value outside repository source, and SHALL apply no cap value defaulted in code.
13. THE Issuance_Service SHALL record a Cost_Log per invocation containing the MCP call count, zero model calls, and an estimated cost of zero for the sandbox environment.
14. IF no Human_Confirm_Event exists for a transaction in the same session log at the moment an issuance request is received, THEN THE Issuance_Service SHALL return a typed `confirmation-required` error, SHALL make zero issuance tool calls for that transaction, and SHALL send no EIP-3009 signature request.
15. IF Per_Card_Cap is absent from operator-owned configuration when an issuance request is received, THEN THE Issuance_Service SHALL return a typed `configuration-missing` error naming the absent cap key and the Issuance_Service, SHALL make zero issuance tool calls, and SHALL issue no card.
16. IF a retried issuance tool call carrying an EIP-3009 signature returns a further x402 payment challenge, THEN THE Issuance_Service SHALL return a typed `x402-retry-bound-exceeded` error, SHALL send no further signature request, SHALL issue no card, and SHALL record the observed challenge count in the Provenance_Logger entry.

### Requirement 7: Settlement Verification from Two Independent Sources

**User Story:** As a Merchant_Operator, I want on-chain confirmation of a settlement transaction from a source independent of the payment gateway, so that I am not relying on the gateway's own claim.

#### Acceptance Criteria

1. WHEN the Issuance_Service returns a settlement transaction identifier, THE Settlement_Verifier SHALL resolve that identifier against two mutually independent on-chain data sources, where mutual independence requires that neither source is the interface that reported the identifier and neither source shares its upstream data provider with the other, using at most 2 resolution attempts per source and a request deadline of 10 seconds per attempt.
2. WHEN both data sources return a resolved transaction for one settlement transaction identifier, THE Settlement_Verifier SHALL compare the resolved transaction amount to the guardrail-approved amount for exact equality in the settlement asset's smallest denominated unit with zero tolerance, and SHALL compare the resolved signing address to the wallet address that produced the funding signature for equality without regard to letter casing.
3. WHEN both sources report the same transaction state, that state carries a block confirmation count at or above the minimum confirmation depth, and both comparisons required by criterion 2 hold as equal, THE Settlement_Verifier SHALL record a `verified` result on the Shared_Canvas_Node.
4. IF the two sources report differing transaction states for one identifier, THEN THE Settlement_Verifier SHALL record a `verification-disagreement` result naming the state reported by each of the two sources, SHALL withhold the `verified` result, and SHALL leave the transaction state unchanged.
5. IF a data source returns no response within the 10-second request deadline on both of its resolution attempts, or returns a transport-level error, THEN THE Settlement_Verifier SHALL record an `unverified` result naming the unreachable source, SHALL leave the transaction state unchanged, and SHALL record no `verified` result.
6. THE Settlement_Verifier SHALL issue only requests that retrieve data, and SHALL issue no call that transfers value, submits a transaction, or mutates on-chain state.
7. THE Settlement_Verifier SHALL read each data source's endpoint and credentials from a separate operator-owned configuration entry per source at runtime, and SHALL read the minimum confirmation depth from operator-owned configuration at runtime.
8. IF the resolved transaction amount differs from the guardrail-approved amount, or the resolved signing address differs from the wallet address that produced the funding signature, THEN THE Settlement_Verifier SHALL record a `verification-mismatch` result naming the failing comparison and both compared values, SHALL withhold the `verified` result, and SHALL leave the transaction state unchanged.
9. IF both sources report the same transaction state and that state carries a block confirmation count below the minimum confirmation depth, THEN THE Settlement_Verifier SHALL record an `unverified` result naming the observed confirmation count, SHALL withhold the `verified` result, and SHALL leave the transaction state unchanged.
10. WHEN the Settlement_Verifier completes its resolution attempts for one settlement transaction identifier, THE Settlement_Verifier SHALL record exactly one result of `verified`, `verification-disagreement`, `verification-mismatch`, or `unverified` on that transaction's Shared_Canvas_Node within 60 seconds of receiving that identifier.

### Requirement 8: Escrow-Meter Settlement Guarantee (US-4, Should)

**User Story:** As a Merchant_Operator, I want an on-chain-verifiable settlement guarantee before committing inventory, so that I am not exposed to a fare hold with no funds behind it.

#### Acceptance Criteria

1. WHEN a fare hold carrying a stated Hold_Window duration is returned for an offer, THE Escrow_Meter SHALL record that Hold_Window duration in whole seconds against that offer's Shared_Canvas_Node within 5 seconds of receiving the fare hold.
2. WHEN an on-chain escrow transaction is confirmed for an offer holding a recorded Hold_Window duration, THE Escrow_Meter SHALL record an escrow commitment against that offer's Shared_Canvas_Node carrying the on-chain escrow transaction identifier, a commitment timestamp at 1-second-or-finer resolution, and a commitment window duration in whole seconds.
3. THE Escrow_Meter SHALL order the merchant-console log so that an escrow commitment timestamp for an offer strictly precedes the `inventory-committed` state transition timestamp for the same offer.
4. WHEN the Escrow_Meter records an escrow commitment for an offer, THE Escrow_Meter SHALL set that commitment's window duration to a value differing from the recorded Hold_Window duration for the same offer by at most 5 seconds.
5. IF no escrow commitment exists for an offer at the time the merchant console renders that offer, THEN THE Escrow_Meter SHALL withhold the `backed` marking from the merchant console and SHALL record a `no-escrow-commitment` reason on that offer's Shared_Canvas_Node.
6. IF the escrow commitment window duration differs from the recorded Hold_Window duration by more than 5 seconds, THEN THE Escrow_Meter SHALL withhold the `backed` marking and SHALL record a `window-mismatch` reason naming both durations in whole seconds.
7. WHILE an escrow commitment for an offer is unexpired, THE Escrow_Meter SHALL expose the remaining window duration in whole seconds on that offer's Shared_Canvas_Node to both the shopper-side subscription and the merchant-side subscription, and SHALL update that remaining duration at least once every 5 seconds.
8. WHERE an unexpired escrow commitment exists for an offer and that commitment's window duration differs from the recorded Hold_Window duration by at most 5 seconds, THE Escrow_Meter SHALL apply the `backed` marking to that offer on the merchant console and SHALL record no withholding reason for that offer.
9. IF a fare hold is returned for an offer with no stated Hold_Window duration or with a duration that is not resolvable to whole seconds, THEN THE Escrow_Meter SHALL record no escrow commitment for that offer, SHALL withhold the `backed` marking, and SHALL record a `no-escrow-commitment` reason on that offer's Shared_Canvas_Node.
10. WHEN an escrow commitment's remaining window duration reaches zero seconds, THE Escrow_Meter SHALL stop exposing a remaining window duration for that commitment on the Shared_Canvas_Node and SHALL withhold the `backed` marking from that offer on the merchant console.

### Requirement 9: On-Chain-Direct Settlement, Path A (US-5, Should)

**User Story:** As a Shopper_Agent_Principal holding XSGD in a self-custodied wallet, I want to pay a merchant directly on-chain when the merchant accepts stablecoin, so that I do not route funds through custody for transactions that need neither a card nor a fiat off-ramp.

#### Acceptance Criteria

1. WHERE the merchant address for a transaction is recorded in operator-owned configuration as accepting direct stablecoin transfer, THE Edge_Orchestrator SHALL offer Path_A as a settlement option for that transaction and SHALL use that recorded address as the sole recipient of the Path_A transfer, deriving no recipient value from client-supplied input.
2. WHEN a transaction settles through Path_A, THE Self_Custody_Wallet_Interface SHALL broadcast the transfer to the Avalanche network, and THE Provenance_Logger SHALL record a transaction log for that transaction containing zero entries naming any custody-provider interface.
3. THE Edge_Orchestrator SHALL read the signing wallet's native gas balance as the last recorded step before requesting a Path_A signature with no intervening Path_A step between that read and the signature request, and SHALL re-read the balance before requesting the signature when the most recent reading is more than 5 seconds old.
4. IF the gas balance read in criterion 3 is below the estimated gas cost of the Path_A transfer, including a balance of zero, THEN THE Edge_Orchestrator SHALL return a typed `insufficient-gas` error naming the read balance and the estimated gas cost, SHALL request no signature from the Self_Custody_Wallet_Interface regardless of any earlier balance reading, and SHALL record the blocked state on the Shared_Canvas_Node.
5. WHEN the Edge_Orchestrator requests a Path_A wallet signature, THE Edge_Orchestrator SHALL set the requested transfer amount equal to the amount recorded by the Guardrail_Gate `pass` result for that transaction, read server-side from operator-owned configuration at request time, and SHALL record that Guardrail_Gate approval event before the signature request event in session-log order.
6. WHEN a Path_A transaction is broadcast, THE Settlement_Verifier SHALL verify that transaction against the same two mutually independent sources required for Path_B settlements and SHALL apply the same `verified`, `verification-disagreement`, and `unverified` outcomes defined for Path_B.
7. IF the wallet holder declines the Path_A signature request, THEN THE Edge_Orchestrator SHALL record a `signature-declined` state on the Shared_Canvas_Node, SHALL broadcast no transaction, SHALL perform no rollback because no held balance was debited, and SHALL leave the unsigned transfer payload unchanged.
8. WHEN a Path_A transaction completes, THE Shared_Canvas_Node_Store SHALL record that transaction using the same node schema and the same field set used for Path_B settlements so that one hash-linked record holds for both settlement paths, and SHALL introduce no Path_A-only field.
9. THE Travel_Agency_System SHALL label the Path_A guardrail approval event recorded in criterion 5 as advisory, SHALL block no Path_A transfer on budget-ceiling grounds, and SHALL state that no server-side or on-chain enforcement point exists in this increment (see Non-Goals and Blocked Items, item B1).
10. WHEN a Path_A settlement option or a Path_A transaction state is displayed, THE Travel_Agency_System SHALL present the guardrail approval as advisory on both the Shopper_Client and the Merchant_Client and SHALL present no indication that a budget ceiling is enforced for Path_A.
11. IF the merchant address for a transaction is not recorded in operator-owned configuration as accepting direct stablecoin transfer, THEN THE Edge_Orchestrator SHALL return a typed `merchant-address-unrecorded` error, SHALL offer no Path_A option for that transaction, SHALL request no Path_A signature, and SHALL record the blocked state on the Shared_Canvas_Node.
12. IF the network rejects a broadcast Path_A transfer, THEN THE Edge_Orchestrator SHALL record a `broadcast-failed` state on the Shared_Canvas_Node carrying the rejection reason reported by the network, SHALL start no automatic re-broadcast, and SHALL debit no balance.

### Requirement 10: Wallet Linking for Path B

**User Story:** As a Shopper_Agent_Principal, I want my self-custodied wallet address recognized as mine before funds routed through custody are credited, so that my transfer is attributable and usable.

#### Acceptance Criteria

1. WHEN a linkage submission carrying a self-custody wallet address and a customer profile identifier verified by the custody provider is received, THE Wallet_Linking_Service SHALL record one address-to-profile mapping for that address and SHALL make that mapping readable to subsequent attribution checks within 5 seconds of recording it.
2. WHEN a Path_B inbound transfer arrives from a wallet address, THE Wallet_Linking_Service SHALL resolve that address against the recorded mappings after normalizing both the sending address and each stored address to a single letter case, and SHALL confirm that a mapping exists for that address before any usable balance is credited for that transfer.
3. IF a Path_B inbound transfer arrives from an address with no recorded mapping under the case-normalized comparison of criterion 2, THEN THE Wallet_Linking_Service SHALL hold the transfer as unattributed, SHALL record a `pending-manual-linkage` state, SHALL credit zero usable balance, and SHALL retain that state until a mapping for that address is recorded, applying no time-based expiry that credits the transfer.
4. THE Wallet_Linking_Service SHALL expose its custody-provider interface as a typed interface whose endpoint, credentials, and profile-verification request deadline are supplied by operator-owned configuration, and SHALL apply a deadline of 10 seconds when the configuration states none.
5. THE Wallet_Linking_Service SHALL accept any wallet address consisting of the `0x` prefix followed by exactly 40 hexadecimal characters in upper case, lower case, or mixed case, SHALL treat two such addresses that differ only in letter case as the same address, and SHALL require no wallet-vendor-specific attestation.
6. WHEN a linkage submission presents a wallet address already mapped to the same customer profile identifier under the case-normalized comparison of criterion 2, THE Wallet_Linking_Service SHALL leave the stored mapping unchanged, SHALL create no additional mapping record for that address, and SHALL return the existing mapping as a successful result.
7. IF a linkage submission presents a wallet address already mapped to a different customer profile identifier, THEN THE Wallet_Linking_Service SHALL reject the submission with a typed `address-already-linked` error, SHALL leave the existing mapping unchanged, and SHALL record no additional mapping for that address.
8. IF a submitted wallet address does not match the address format accepted in criterion 5, THEN THE Wallet_Linking_Service SHALL reject the submission with a typed `invalid-wallet-address` error, SHALL record no mapping, and SHALL make zero outbound custody-provider calls.
9. THE Travel_Agency_System SHALL label the generic EVM address acceptance in criterion 5 as an assumption pending resolution of Open Questions Carried Forward item 4, and SHALL claim no wallet-vendor-specific attestation coverage in this increment.

### Requirement 11: Out-of-Canvas Notification Dispatch (US-6, Should)

**User Story:** As a Shopper_Agent_Principal, I want to receive a message outside the canvas when my booking's state changes, so that I do not have to keep the app open to know the outcome.

#### Acceptance Criteria

1. THE Notification_Dispatcher SHALL consume normalized canvas events produced by the existing Webhook_Normalizer role, SHALL perform no provider-payload parsing of its own, and SHALL derive every send decision solely from those normalized events and the corresponding node's recorded state history.
2. WHEN a Shared_Canvas_Node transitions to a Notified_State, THE Notification_Dispatcher SHALL send one message carrying the node identity and the reached state name, and SHALL initiate that message's first transmission attempt within 10 seconds of the recorded state-change event timestamp.
3. THE Notification_Dispatcher SHALL send a message only for a state the corresponding Shared_Canvas_Node has actually reached as recorded in the node's state history, and SHALL send zero messages for any state absent from that history.
4. THE Notification_Dispatcher SHALL send exactly one message per Notified_State transition per recipient, SHALL identify each transition by a suppression key composed of node identity, transition sequence index in the node's state history, and recipient identifier, SHALL retain each suppression key for at least 24 hours after the recorded state-change event timestamp, and SHALL transmit no message for a suppression key that already holds a successful transmission or a terminal outcome, including when a duplicate or replayed normalized event names the same transition.
5. THE Notification_Dispatcher SHALL read the Notified_State set from operator-owned configuration, SHALL keep that set outside repository source, and SHALL treat every state outside that configured set as non-notifying.
6. THE Notification_Dispatcher SHALL deliver messages without requiring the Shopper_Client to be open or connected, and SHALL require no client-supplied state for any send operation.
7. IF every transmission attempt permitted for one transition has failed, THEN THE Notification_Dispatcher SHALL record a `notification-send-failed` canvas event carrying the target state and the provider error reported by the final attempt, SHALL mark that transition's suppression key terminal so no later attempt is made for it, and SHALL leave the node's recorded state history unchanged.
8. THE Notification_Dispatcher SHALL resolve each recipient's messaging identifier from a stored per-user mapping as the first step of the send operation, before constructing or transmitting any request to the messaging provider.
9. IF recipient resolution within a send operation finds no recorded mapping, THEN THE Notification_Dispatcher SHALL end that send operation with a `notification-recipient-unmapped` canvas event and SHALL transmit no message to the messaging provider.
10. THE Notification_Dispatcher SHALL expose its messaging provider as a typed interface whose endpoint and credentials are supplied by operator-owned configuration at runtime, and SHALL keep every endpoint and credential value outside repository source.
11. THE Notification_Dispatcher SHALL record a Cost_Log per send operation containing the external call count, the model call count, and the estimated cost actually incurred, defaulting each to zero when the send operation consumed no such resource.
12. IF a transmission attempt for one transition fails or returns no provider response within 3 seconds, THEN THE Notification_Dispatcher SHALL retry that transmission at most 2 further times at intervals of at least 1 second, SHALL make no transmission attempt for that transition later than 30 seconds after the recorded state-change event timestamp, SHALL make no further attempt once a transmission has succeeded, and SHALL count all attempts for that transition as one message for the purposes of criterion 4.
13. WHEN two or more Notified_State transitions are recorded for one node identity within 10 seconds of each other, THE Notification_Dispatcher SHALL process one send operation per transition in ascending state-history sequence order per recipient, and SHALL not begin the send operation for a later transition until the send operation for the earlier transition has terminated with a successful transmission, a terminal failure, or a `notification-recipient-unmapped` outcome.
14. IF a normalized state-change event names a state that is absent from the corresponding node's recorded state history at evaluation time, THEN THE Notification_Dispatcher SHALL discard that event, SHALL transmit no message to the messaging provider, and SHALL record a canvas event indicating the discarded unverified state.

### Requirement 12: Hash-Linked Provenance Record

**User Story:** As a Merchant_Operator resolving a dispute, I want every issuance traceable to the guardrail decision that approved it, so that the record settles the question without a separate support channel.

#### Acceptance Criteria

1. WHEN a gate decision, a Payment_Call, a settlement event, or a notification event occurs for a transaction, THE Provenance_Logger SHALL append exactly one entry to that transaction's chain within 2 seconds of the event, and SHALL record in that entry an entry identifier, the transaction identifier, the event kind drawn from the closed set of gate decision, Payment_Call, settlement event, and notification event, the recording timestamp, and a sequence index taken from consecutive integers beginning at 1.
2. THE Provenance_Logger SHALL record in each entry whose sequence index is greater than 1 the digest of the entry holding the immediately lower sequence index for the same transaction, and SHALL record in the entry whose sequence index is 1 a single reserved genesis predecessor value of the same length as a digest that denotes the absence of a preceding entry.
3. THE Provenance_Logger SHALL record in each Payment_Call entry the identifier of the Guardrail_Gate decision entry that emitted the `pass` result approving that call, and that referenced entry SHALL hold a lower sequence index within the same transaction chain.
4. IF an append request for a Payment_Call entry carries no decision identifier that resolves to a `pass` decision entry in the same transaction chain, THEN THE Provenance_Logger SHALL reject that append request, SHALL leave every existing entry byte-identical, and SHALL return a typed error indicating that the approving decision reference is unresolvable.
5. THE Provenance_Logger SHALL compute every digest in a chain with one configured digest function over a canonical serialization of the entry that fixes field order, fixes the textual encoding of every recorded value, and excludes the entry's own digest field, SHALL produce byte-identical digests for byte-identical canonical serializations, and SHALL produce differing digests for canonical serializations that differ in any byte.
6. WHEN two or more append requests for one transaction are received concurrently, THE Provenance_Logger SHALL place them in one total order, SHALL assign each request a distinct sequence index with no index shared by two entries and no index skipped, and SHALL derive each entry's recorded preceding digest from the entry holding the immediately lower index in that same order.
7. THE Provenance_Logger SHALL append entries only, SHALL modify and delete no existing entry, and SHALL reject any request to modify or delete an existing entry with a typed append-only violation error while leaving every stored entry byte-identical.
8. WHEN chain verification runs over a stored entry sequence for a transaction, THE Provenance_Logger SHALL derive the verdict only from the stored entries of that sequence, and SHALL return an identical verdict and an identical lowest failing sequence index on every repeated verification of the same stored sequence, regardless of the requesting transaction side or the time of the request.
9. WHEN a human opens a transaction node, THE Shared_Canvas_Node_Store SHALL expose the full entry sequence for that transaction in ascending sequence-index order to both the shopper-side and the merchant-side subscription within 3 seconds, and the Node_Payload_Checksum of the exposed sequence SHALL be equal for both subscriptions.
10. IF an append fails, THEN THE Provenance_Logger SHALL record a gap marker occupying the sequence index the failed entry would have held, SHALL record in that gap marker the expected preceding digest and the event kind of the failed entry, SHALL make no further attempt to write the failed entry at that index, and chain verification SHALL report each gap marker as a detected discontinuity naming its sequence index.

### Requirement 13: Role-Scoped Dual-Side Access

**User Story:** As a Merchant_Operator, I want my console session to read shared transaction nodes without gaining access to a shopper's pre-confirmation drafts, so that both sides trust the shared record.

#### Acceptance Criteria

1. THE Session_Authority SHALL extend the existing workspace role model with a transaction-side attribute recorded on each workspace membership, SHALL constrain that attribute to exactly one value per membership drawn from `shopper` and `merchant`, and SHALL keep that attribute independent of the existing role union `viewer | editor | owner | provider-admin`, which continues to govern write permission.
2. THE Shared_Canvas_Node_Store SHALL derive each subscriber's transaction side, and each node's party identifiers, from the values resolved by the Session_Authority and propagated through the existing `x-knowgrph-room-role` header, and SHALL discard any transaction-side value or party-identifier value supplied by the client.
3. WHERE a subscriber's resolved transaction side is `merchant`, THE Shared_Canvas_Node_Store SHALL serve exactly those nodes whose Node_Scope is `shared` and whose recorded merchant party identifier equals that subscriber's workspace membership identifier, and SHALL withhold every node whose Node_Scope is `personal` irrespective of that node's recorded party identifiers.
4. WHERE a subscriber's resolved transaction side is `shopper`, THE Shared_Canvas_Node_Store SHALL serve exactly those nodes whose recorded shopper party identifier equals that subscriber's workspace membership identifier, covering both the `personal` and `shared` Node_Scope values, and SHALL withhold every node whose recorded shopper party identifier differs from that identifier.
5. THE Shared_Canvas_Node_Store SHALL validate every subscription request using the existing session-token validation and the existing room-role header mechanism only, and SHALL expose no second authentication path.
6. IF a subscription request carries no session token, carries a session token that the existing validation rejects, or resolves to a transaction-side value set that is not exactly one of `shopper` or `merchant`, THEN THE Shared_Canvas_Node_Store SHALL reject the subscription, SHALL serve zero node payloads including zero partial payloads and zero metadata-only payloads, and SHALL return a typed error that distinguishes an invalid session from an unresolved transaction side.
7. WHEN a Shared_Canvas_Node is created, THE Shared_Canvas_Node_Store SHALL record the creating session's workspace membership identifier as that node's shopper party identifier, SHALL record the merchant-side workspace membership identifier resolved for that transaction as that node's merchant party identifier no later than the transition of that node's Node_Scope to `shared`, and SHALL treat both recorded identifiers as immutable for the remaining lifetime of that node.
8. WHILE a subscription is open, THE Shared_Canvas_Node_Store SHALL re-validate that subscriber's session token and resolved transaction side at intervals of at most 60 seconds and before serving each node payload.
9. IF a re-validation of an open subscription finds the session token expired or revoked, THEN THE Shared_Canvas_Node_Store SHALL close that subscription within 5 seconds of that re-validation, SHALL serve no further node payload on that subscription, SHALL return a typed `session-expired` close reason, and SHALL leave every stored node record unchanged.

### Requirement 14: Mobile-First, Offline-Tolerant Client Behaviour

**User Story:** As a Shopper_Agent_Principal on a phone with patchy connectivity, I want the flow to stay usable and never double-charge me, so that I can complete a booking on the move.

#### Acceptance Criteria

1. WHILE the viewport width is at or below the existing 768px mobile viewport threshold, THE Shopper_Client and THE Merchant_Client SHALL each render a single-column layout carrying exactly one primary action that stays pinned to the bottom edge of the viewport and remains visible without scrolling.
2. WHILE the viewport width is above the existing 768px mobile viewport threshold, THE Shopper_Client and THE Merchant_Client SHALL each render the existing multi-column canvas layout and SHALL pin no primary action to the bottom edge.
3. THE Shopper_Client SHALL express transaction structure using semantic HTML elements whose native role matches the content role.
4. THE Shopper_Client SHALL retain a non-empty accessible name and a keyboard-reachable interaction contract on every selectable media wrapper and every selectable icon wrapper, and SHALL apply no `aria-hidden` attribute to any selectable wrapper or to any element containing one.
5. WHEN a discovery result set is returned, THE Shopper_Client SHALL cache that result set in the existing local-first persistence layer together with the capture timestamp, SHALL retain at most the 20 most recently captured result sets per session, SHALL retain each cached result set for at most 24 hours after its capture timestamp, and SHALL evict the oldest cached result set first when either bound is reached.
6. WHILE the Shopper_Client observes the offline connectivity signal — the browser-reported offline state, or two consecutive requests to the Edge_Orchestrator each failing to complete within 5 seconds — THE Shopper_Client SHALL display a stale-state indicator within 1 second of that signal and SHALL display it whether or not a cached discovery result set exists.
7. WHERE a cached discovery result set exists, WHILE the Shopper_Client observes the offline connectivity signal, THE Shopper_Client SHALL render that cached result set alongside the stale-state indicator and SHALL render the cached set's capture timestamp.
8. WHEN the Shopper_Client observes the browser-reported online state, THE Shopper_Client SHALL remove the stale-state indicator within 1 second of that signal.
9. WHILE the Shopper_Client holds one or more queued confirm actions, THE Shopper_Client SHALL start no automatic payment retry and SHALL make no Payment_Call.
10. WHEN connectivity is restored, THE Shopper_Client SHALL submit each queued confirm action exactly once, identified by a stable action identifier assigned when that action was queued, and SHALL discard any further submission carrying an already-submitted action identifier.
11. IF a queued confirm submission fails to complete after 3 attempts carrying the same action identifier, THEN THE Shopper_Client SHALL retain that queued action unchanged in the existing local-first outbox, SHALL display an error indication naming the unsent confirm, SHALL make no Payment_Call for that action, and SHALL require an explicit human re-confirm before any further submission attempt.
12. IF the count of queued confirm actions reaches 20, THEN THE Shopper_Client SHALL reject each further confirm action with an error indication that the offline confirm queue is full and SHALL leave the existing queued actions unchanged.
13. THE Shopper_Client and THE Merchant_Client SHALL use only capabilities available to a browser-based PWA.

### Requirement 15: Cost, Token, and TCO Observability

**User Story:** As the solo operator, I want every call's cost recorded against a stated budget, so that the increment holds its zero-TCO and token-budget targets without manual auditing.

#### Acceptance Criteria

1. THE Intent_Parser and the Offer_Scorer SHALL each emit exactly one Cost_Log per invocation, conforming to the repository's existing cost-log schema, carrying model identifier, prompt token count, completion token count, and estimated cost, and recorded before that invocation returns its result.
2. THE Shared_Canvas_Node_Store, the Guardrail_Gate, the Confirmation_Gate, the Settlement_Verifier, and the Notification_Dispatcher SHALL each emit exactly one Cost_Log per operation recording zero model calls, zero prompt tokens, zero completion tokens, and zero estimated model cost.
3. IF a Cost_Log emitted by one of the five components named in criterion 2 records a non-zero model call, THEN THE Cost_Log_Validator SHALL report a defect naming that component and the invocation the Cost_Log belongs to, and SHALL withhold a conformance-pass result for the validation run containing that Cost_Log.
4. THE Travel_Agency_System SHALL make every outbound platform call through the already-provisioned Cloudflare Workers, Durable Objects, D1, and R2 bindings only, and SHALL record measured monthly infrastructure cost for the current measurement window against a monthly TCO ceiling resolved from operator-owned configuration.
5. THE Travel_Agency_System SHALL measure monthly token cost as the sum of the estimated cost recorded by every Cost_Log within the current measurement window, SHALL treat the measurement window as the current calendar month normalized to the configured monthly session-volume baseline, SHALL compare that measured cost against a monthly token-cost ceiling resolved from operator-owned configuration, and SHALL mark the measured cost as incomplete while any invocation inside the window has no recorded Cost_Log.
6. WHEN a session reaches the settled state, THE Provenance_Logger SHALL record that session's total step count, counted as the number of Provenance_Logger entries recorded for that session, and the elapsed duration in seconds from that session's recorded intent entry to its recorded settled entry.
7. WHEN a recorded Cost_Log causes the measured monthly token cost or the measured monthly infrastructure cost for the current measurement window to exceed its configured ceiling, THE Travel_Agency_System SHALL record an overrun report within 60 seconds of recording that Cost_Log, naming the exceeded ceiling, the measured value, and the measurement window.
8. IF a settled session's recorded step count exceeds the configured maximum-step ceiling resolved from operator-owned configuration, or its recorded elapsed duration exceeds the configured maximum-duration ceiling resolved from operator-owned configuration, THEN THE Travel_Agency_System SHALL record a time-to-value breach report naming the exceeded ceiling, the recorded value, and the session identifier, and SHALL retain that session's Provenance_Logger entries.
9. IF an invocation of a component required by criterion 1 or criterion 2 to emit a Cost_Log completes with no Cost_Log recorded for that invocation, THEN THE Cost_Log_Validator SHALL report a missing-cost-log defect naming that component and the invocation, distinct from a cost-shape violation defect, and SHALL withhold a conformance-pass result for the validation run covering that invocation.

### Requirement 16: Provider-Agnostic Configuration and Source Hygiene

**User Story:** As the solo operator, I want every vendor binding, credential, and limit supplied at runtime, so that the repository stays neutral, portable, and safe to publish.

#### Acceptance Criteria

1. THE Travel_Agency_System SHALL keep every machine path, credential, account identifier, provider catalogue entry, and environment-specific default outside repository source, such that a scan of the tracked source files returns zero occurrences of these values and every such value is referenced by its configuration key name only.
2. THE Travel_Agency_System SHALL resolve every external endpoint, credential, cap, ceiling, and retry bound from operator-owned runtime configuration at the start of each component invocation, and SHALL read no such value from a literal in repository source.
3. THE Travel_Agency_System SHALL define each external provider behind a typed interface, SHALL validate the outbound request against that interface's input schema and the inbound provider response against that interface's output schema at the interface boundary before the value is used, and SHALL treat that boundary validation as the enforcing mechanism of the typed-contract guarantee, because `canvas/tsconfig.json` sets `strict: false` and compile-time checking alone does not reject a violating payload at runtime.
4. THE Travel_Agency_System SHALL keep every authored file at or below 600 lines, counting blank and comment lines, and at or below 500 kibibytes, and SHALL fail the existing CI per-file cap check for any authored file exceeding either cap.
5. THE Travel_Agency_System SHALL reuse the existing shared semantic-key helpers, cost-log schema, typed harness contract shape, and headless widget primitives, and SHALL introduce zero additional per-surface semantic-key helper, cost-log schema, harness contract shape, or widget primitive.
6. IF a configuration value required by a component is absent or resolves to an empty value at runtime, THEN THE Travel_Agency_System SHALL return a typed `configuration-missing` error naming the absent configuration key and the requesting component before that component's first outbound call attempt, SHALL make zero outbound calls that depend on the absent key, and SHALL include no configuration value in the error payload.
7. WHERE a component's required configuration is complete, THE Travel_Agency_System SHALL permit that component's outbound calls while another component's required configuration is absent, and SHALL withhold outbound calls only from the component whose configuration is absent.
8. WHERE a resolved configuration value is cached in memory, THE Travel_Agency_System SHALL serve that cached value for at most 300 seconds and SHALL re-resolve it from operator-owned runtime configuration on the first invocation after that interval, and SHALL resolve every credential value on each invocation without caching it.
9. THE Travel_Agency_System SHALL record every credential value as a fixed redaction placeholder in every Cost_Log entry, every Provenance_Logger entry, and every returned error payload, and SHALL record the credential's configuration key name in place of its value.
10. IF an inbound provider response fails its typed interface's output-schema validation, THEN THE Travel_Agency_System SHALL return a typed `provider-contract-violation` error naming the failing field path and the requesting component, and SHALL write no part of that response to a Shared_Canvas_Node.

### Requirement 17: Readiness Rung and Deploy Boundary Discipline

**User Story:** As the solo operator, I want each component's readiness claim derived from recorded evidence, so that no component is reported as proven before it runs.

#### Acceptance Criteria

1. THE Travel_Agency_System SHALL record for each component exactly one local rung value and one delivered rung value, SHALL derive each recorded value solely from that component's recorded Evidence References, SHALL derive local rung `dev-proven` only from at least one recorded Evidence Reference produced by a Dev run of that component, and SHALL derive local rung `runtime-ready` only where recorded Evidence References cover all five of that rung's conditions: typed inputs and outputs, bounded orchestration, cost evidence, fallbacks, and explicit gates.
2. IF a component has no recorded Evidence Reference, THEN THE Travel_Agency_System SHALL hold that component's local rung at `spec-complete`, SHALL hold its delivered rung at `undocumented`, and SHALL keep both values unchanged until at least one Evidence Reference for that component is recorded.
3. THE Travel_Agency_System SHALL bind the card-issuance provider to its sandbox endpoint, supplied by operator-owned configuration, for the whole of this increment and SHALL make zero card-issuance calls to a production endpoint during this increment.
4. THE Travel_Agency_System SHALL initialize and hold the sandbox-to-production issuance deploy boundary in the `closed` state by default, and SHALL treat `closed` as the state in force whenever that boundary's state cannot be derived from a recorded operator instruction.
5. WHERE an Evidence Reference, an operator instruction, and a rollback statement all exist for the sandbox-to-production issuance deploy boundary, WHILE the production tool schema is confirmed to match the sandbox contract, WHEN an operator requests that boundary to open for a named candidate, THE Travel_Agency_System SHALL open that boundary for that exact authorized candidate only and SHALL leave that boundary `closed` for every other candidate.
6. THE Travel_Agency_System SHALL treat the Prod mirror and the Cloudflare route as deployment targets outside these acceptance criteria and SHALL derive no delivered rung for either target from this increment's Evidence References.
7. THE Travel_Agency_System SHALL record a rollback statement for each declared deploy boundary before that boundary leaves the `closed` state, and SHALL treat a rollback statement as recorded only where it names the boundary, the authorized candidate, and the actions that return the target to its pre-open state.
8. THE Travel_Agency_System SHALL treat an Evidence Reference as recorded only where one completed run has produced all five of the following parts: the component name, the identifier of the acceptance criterion the run exercised, the run environment name, the run start timestamp, and the identifier of the run's retained log artifact; and SHALL treat an Evidence Reference missing any one of those five parts as absent.
9. IF a rung value or a deploy boundary state is changed by any means other than derivation from recorded Evidence References, recorded operator instructions, and recorded rollback statements, including a hand edit of stored readiness or boundary metadata, THEN THE Travel_Agency_System SHALL discard the changed value, SHALL restore the value derived from those records, and SHALL report a defect naming the affected component, the discarded value, and the restored value.
10. THE Travel_Agency_System SHALL hold the Path_A guardrail enforcement item recorded as blocked item B1 at local rung `spec-complete`, SHALL hold its delivered rung at `undocumented`, and SHALL derive no higher rung for that item until a recorded Evidence Reference exists for a built enforcement point that blocks an over-budget Path_A settlement.

## Non-Goals and Blocked Items

These are carried forward from the source specification without softening. Each stays visible in the design and task phases.

### Blocked items

- **B1 — Path A guardrail enforcement (US-5's harder half).** There is no server-side binding point at which a budget check can block an unlinked Path_A settlement, because the custody-mediated gate assumed a provider call to intercept. The two candidate enforcement points — a client-side spending-policy hook inside the wallet, and an on-chain spending-limit contract — neither exist nor are scoped. Requirement 9 criteria 5 and 9 record an advisory guardrail approval event only. Unlinked Path_A settlement genuinely carries no budget-cap protection in this increment, and no surface may imply otherwise. This item stays `spec-complete` with no path to a higher rung until one enforcement point is built.
- **B2 — Multi-card funding for amounts above Per_Card_Cap.** The flagship flight Budget_Ceiling of SGD 500 exceeds the sandbox Per_Card_Cap range of 5–30 SGD by roughly an order of magnitude. Requirement 6 criterion 10 returns a typed error rather than presenting a working multi-card flow. Whether several disposable cards may sum to one approved amount, and how that interacts with each card's single-use and one-view properties, is unresolved.
- **B3 — Production issuance integration.** The production endpoint's MCP tool names and schema are unconfirmed. Requirement 17 holds this deploy boundary `closed`.

### Out of scope for this increment

- Bidirectional reputation write between shopper and merchant.
- WhatsApp notification channel, and native Web Push as its own component.
- MCP gateway federation and external-agent discovery for this system's own surface.
- Privacy-preserving multi-merchant comparison.
- Machine-payable micropayment settlement, blocked on an unresolved provider fee-floor question.
- Cross-border compliance handshake, fare-hold race-condition visualizer, and negotiation replay.
- Dispute arbitration user interface, and multi-agent race-condition handling.
- Migration to a named third-party authentication library. Requirement 13 extends the existing session and role model.

## Correctness Property Candidates

These are the natural property-based test targets in this feature. `fast-check` 3.23.2 is already a devDependency and the `__pbt__` directory convention already exists, so these bind to established substrate. The design phase will formalize each into a stated property with its generator and its bound.

- **P1 — Checksum identity (Requirement 1.3).** For all generated sequences of interleaved shopper-side and merchant-side deltas applied to one node identity, the Node_Payload_Checksum read from the shopper subscription equals the checksum read from the merchant subscription. This is an invariant property and a confluence property: it must also hold when the same delta set is applied in a different order, and it must survive a Durable Object restart, a snapshot fallback in place of delta replay, and rejection of over-limit deltas (Requirements 1.14, 1.15, 1.16).
- **P2 — Payment-call ordering (Requirement 3.5).** For all generated session-event sequences containing a cart-add event, zero or more Payment_Call attempts, and zero or one Human_Confirm_Event, no accepted Payment_Call appears before the Human_Confirm_Event in the resulting log. Path_A signature requests are included in the Payment_Call set.
- **P3 — Budget bound with bounded retry (Requirement 2.2, 2.3, 2.4).** For all generated fare sets and Budget_Ceiling values, the count of Payment_Call invocations for fares exceeding Budget_Ceiling is zero, and the count of retry attempts is at most Retry_Bound. The retry loop terminates for every generated input.
- **P4 — Notification no-false-send (Requirement 11.3, 11.4, 11.14).** For all generated node state histories, every send event corresponds to a Notified_State the node actually reached, and the count of send events per state transition per recipient is exactly one. No send exists for a state absent from the history.
- **P5 — Offer normalization round trip (Requirements 4.3, 5.3).** For all generated provider responses, parsing into the internal typed offer schema and serializing back produces an equivalent object. Discovery harnesses parse third-party payloads, so this round-trip property is required rather than optional.
- **P6 — Scorer conservation (Requirement 5.5).** For all generated pairs of normalized offer lists, the ranked output contains each input offer exactly once, so the output length equals the sum of the input lengths.
- **P7 — Scope containment (Requirements 1.5, 13.3).** For all generated node sets with mixed Node_Scope values, the merchant-side subscription result contains no node whose Node_Scope is `personal`, and this holds for all generated shopper-side and merchant-side party identifiers, including a subscriber whose transactions are absent from the generated set.
- **P8 — Provenance chain integrity (Requirements 12.2, 12.5, 12.6).** For all generated append sequences, each entry's recorded preceding hash equals the hash of the actual preceding entry for that transaction, and no append mutates an existing entry.
- **P9 — Configuration-missing error conditions (Requirement 16.6).** For all generated configuration objects with one required key removed, the affected component returns a typed `configuration-missing` error naming that key and makes zero outbound calls.
- **P10 — Wallet-linking idempotence (Requirement 10.6).** For all generated address and profile pairs, linking twice produces the same mapping as linking once.

## Open Questions Carried Forward

These remain unresolved in the source specification and are recorded here rather than silently assumed away. Each is answerable only outside this document.

1. Does the custody provider's blockchain interface impose a minimum transfer size or per-transaction fee floor? Blocks any micropayment-rail scoping.
2. What is the flight-search sandbox's coverage depth for regional low-cost carriers versus full-service carriers? Affects whether the flagship fare set is representative.
3. Where should Path_A guardrail enforcement live — a client-side wallet spending-policy hook, a pre-signature check in the orchestrator, or an on-chain spending-limit contract? Blocks B1.
4. Does the wallet-linking flow require a wallet-vendor-specific attestation, or is a generic EVM address whitelist sufficient? Requirement 10.5 assumes the generic case.
5. Are the production issuance tool names and schema analogous to the sandbox contract? Blocks B3.
6. Is the sandbox-versus-production cap currency difference (SGD versus USD) a genuine settlement-asset difference or a documentation convention? Affects B2.
7. How should amounts above Per_Card_Cap be funded, and how does a multi-card pattern interact with single-use, one-view cards? Blocks B2.
8. What are the rate limits and scope of the independent block-explorer verification key? Affects Requirement 7's second source.
9. Does Path_A testing need a dedicated pre-funded gas wallet rather than per-session faucet claims, given the faucet's roughly one-claim-per-24-hour limit? Affects whether Requirement 9 is schedulable within a sprint.
10. Should a later WhatsApp channel go direct to the platform or through a solution provider? Affects the deferred notification follow-on only.
11. Which of the three coexisting collaboration transports should own non-transaction canvas documents once Requirement 1.8 binds shared transaction nodes to the Durable Object WebSocket? Requirement 1 answers this for transaction nodes only.
