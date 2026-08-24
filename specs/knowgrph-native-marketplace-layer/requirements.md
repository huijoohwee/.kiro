---
title: "Knowgrph Clean-Room Native Marketplace Layer — Requirements"
doc_type: "Spec Requirements"
schema: "kiro-spec-requirements/v1"
version: "0.1.0"
date: "2026-08-22"
lang: "en-US"
frontmatter_contract: "required"
spec_type: "feature"
workflow_type: "requirements-first"
feature_name: "knowgrph-native-marketplace-layer"
owner: "Solo Founder / AI Orchestrator"
lane: "authoring"
local_rung: "spec-complete"
delivered_rung: "undocumented"
deploy_boundary: "closed"
clean_room_policy: "inspiration-only; no foreign commerce framework code, schema, or dependency"
source_specification: "knowgrph/docs/documents/knowgrph-agentic-commerce-platform-prd-tad-adr.md v0.3.0 (Feature: Clean-Room Native Vendor Settlement Layer)"
source_addendum: "joohwee/prd-tad-ard/knowgrph-cleanroom-native-marketplace-layer.md v0.1.0"
governing_contracts:
  - "huijoohwee.github.io/guidelines/agentic-sdlc-guidelines.md"
  - "agentic-canvas-os/docs/START-WORKFLOW.md"
  - "agentic-canvas-os/docs/AGENTS.md"
  - "knowgrph/AGENTS.md"
---

# Requirements Document

## Introduction

This document derives executable requirements for the **Clean-Room Native Vendor Settlement Layer** specified as Phase 1b in `knowgrph-agentic-commerce-platform-prd-tad-adr.md` v0.3.0, under ADR-4 (clean-room boundary), ADR-5 (hand-rolled rules and lifecycle), and ADR-6 (same-transaction projection, alarm-driven dispatch).

Phase 1 of that specification made the marketplace's **demand** side domain-agnostic: one router, one guardrail, one confirmation gate, one issuance path, whichever registered agent found the offer. This increment adds the **supply** side: who gets paid, how much, and when, once a single settled bundle involves more than one supplier.

**Scope discipline.** Every requirement below traces to a named user story (US-6 … US-10), an ADR, an architecture section, or a Deploy Boundary Register row in the source specification. No requirement introduces behaviour absent from that specification. Where the source states an honest gap, this document restates the same gap as a bounded requirement rather than closing it silently — most importantly Requirement 1.9, which fixes the meaning of the `active` vendor state at *operator-marked* and explicitly denies it any compliance meaning.

**Deploy boundary.** Every requirement is satisfied entirely within the Dev lane (`GitHub/knowgrph`, `npm run dev:apex`, `npm run dev`). The Prod mirror (`GitHub/huijoohwee/content/knowgrph`) and the Cloudflare routes (`airvio.co`, `airvio.co/knowgrph`) are gated deploy targets and are never acceptance criteria for any requirement here. Remote D1 migration application and any real outward payout movement are irreversible operator-gated operations and are explicitly out of every requirement's satisfaction condition (Requirement 9).

**Clean-room boundary.** No requirement is satisfiable by installing, copying, vendoring, submoduling, or calling Mercur or Medusa, or by installing a rules-engine or state-machine library. Requirement 8 makes that boundary mechanically checkable rather than merely asserted.

## Glossary

- **Vendor**: One supplier this platform settles money to. Data, not code — a vendor row grants no capability beyond being a payout destination once `active`. This platform never executes vendor-supplied logic.
- **Vendor_Lifecycle**: The closed state set `pending_review` → `approved` → `active` → `suspended` and the frozen table of legal transitions between them. Sole authority for a vendor state change.
- **Commission_Rule**: A stored, revision-identified declaration that turns a gross amount into a commission amount. Two shapes only in this increment: flat rate and tiered rate.
- **Vendor_Split**: One row per `(committed bundle, vendor)` recording that vendor's gross share of the settled total, the commission deducted, and the net payable. A projection over existing per-leg breakdown data, not a second ledger.
- **Split_Set**: The complete set of Vendor_Split rows for one committed bundle. Complete or absent; never partial.
- **Payout_Dispatch**: One attempt to move a finalized split's net amount outward through the existing in-repo net-settlement route, keyed idempotently by `split_id`.
- **Settled_Total_Minor**: The bundle's settled total in minor currency units, as already recorded by the envelope ledger. Integer, safe-integer range, never zero, sign-encoded for direction.
- **Residual**: `Settled_Total_Minor` minus the sum of split gross amounts. Required to be exactly zero.
- **Largest_Remainder**: The specified deterministic rounding policy. When a proportional allocation of an integer total produces fractional shares, floor every share, then distribute the remaining units one each to the shares with the largest fractional remainders, breaking ties by ascending vendor identifier so the result is reproducible.
- **Operator_Scope**: The existing operator-only read scope guard already used by the Phase 1 registry canvas. Reused, not redefined.
- **Session_Log**: The existing append-only ordered event store with monotonic per-session sequence numbers. Extended with new event types by this increment; not replaced.
- **Named_Check**: A command, stated before dispatch and invocable exactly as written, whose recorded result is the evidence for an acceptance criterion. An assertion that a result exists is not a recorded result.
- **Forbidden_Specifier**: Any of `@medusajs/*`, `@mercurjs/*`, `json-rules-engine`, `xstate`, or a hosted Mercur or Medusa endpoint.

## Requirements

### Requirement 1 — Vendor Registry and Lifecycle

**User Story**: As a Platform Operator, I want a vendor registered with an explicit lifecycle state and required to reach `active` before any payout can be dispatched to it, so that an unvetted or suspended supplier can never receive money. *(Source: US-6)*

#### Acceptance Criteria

1.1 WHEN a vendor record candidate is submitted THEN the Vendor Registry SHALL either write one complete row and return a registered result, or write nothing and return a rejection carrying a typed violation list.
1.2 WHEN a vendor record candidate is missing a required field, carries a field of the wrong type, or declares a lifecycle state outside the closed state set THEN the Vendor Registry SHALL reject it and SHALL NOT write a partial row.
1.3 WHEN a vendor is first registered THEN its lifecycle state SHALL be `pending_review`, and the Vendor Registry SHALL NOT accept a caller-supplied initial state.
1.4 WHEN a vendor state transition is requested THEN Vendor_Lifecycle SHALL be the only authority that permits it, and the Vendor Registry SHALL NOT write a state that Vendor_Lifecycle did not return.
1.5 WHEN a transition is requested that is absent from the frozen transition table THEN Vendor_Lifecycle SHALL return a typed rejection naming the current state and the requested transition, and the stored state SHALL remain unchanged.
1.6 WHEN a payout dispatch is evaluated for a vendor THEN the system SHALL require that vendor's lifecycle state to be exactly `active` at evaluation time, and SHALL block dispatch for every other state including `approved`.
1.7 WHEN a vendor is `suspended` THEN already-committed splits for that vendor SHALL remain in a non-dispatched state and SHALL NOT be dispatched, and SHALL NOT be deleted or altered.
1.8 WHEN the full payout record set and the full Session_Log are read THEN zero payout dispatch records SHALL exist whose vendor's lifecycle state at dispatch time was not `active`. *(This is the US-6 VCC, verbatim in obligation.)*
1.9 The `active` state SHALL mean only that this platform's own operator marked the vendor active. The system SHALL NOT represent, log, project, or document `active` as a KYC attestation, a sanctions screening result, or a verified banking or payout-account relationship. No such capability exists in this repository and none is built by this increment.
1.10 WHEN a vendor row is written THEN it SHALL carry a reference to a Commission_Rule revision, and the Vendor Registry SHALL reject a vendor row whose referenced rule revision does not resolve.

### Requirement 2 — Vendor Ledger Split Projection

**User Story**: As a Supplier, I want my share of a settled bundle recorded as its own row at the moment the bundle commits, so that my position never has to be reconstructed by re-running a calculation later. *(Source: US-7, ADR-6)*

#### Acceptance Criteria

2.1 WHEN a bundle reaches committed state THEN its complete Split_Set SHALL exist in that same committed state, written inside the same transaction as the bundle commit.
2.2 WHEN any split projection invariant is violated THEN the projector SHALL abort the enclosing bundle commit, and the system SHALL NOT commit the bundle with an absent or partial Split_Set.
2.3 WHEN a Split_Set is projected THEN the sum of its gross amounts in minor units SHALL equal the bundle's Settled_Total_Minor exactly, with a Residual of exactly zero.
2.4 WHEN a Split_Set is projected THEN every leg covered by the bundle SHALL appear in exactly one split's covered-leg set, and no leg SHALL appear in two splits or in none.
2.5 WHEN a bundle's per-leg breakdown groups multiple legs to one vendor THEN the projector SHALL emit exactly one split row for that `(bundle, vendor)` pair covering all of that vendor's legs, and SHALL NOT emit one row per leg.
2.6 WHEN the same committed bundle is re-projected from the same inputs THEN the resulting Split_Set SHALL be identical row-for-row and field-for-field, including the ordering of covered-leg identifiers.
2.7 WHEN a proportional allocation of Settled_Total_Minor produces fractional shares THEN the projector SHALL apply Largest_Remainder exactly as defined in the Glossary, and SHALL NOT use floating-point arithmetic at any point in the allocation.
2.8 WHEN every split amount is written THEN each SHALL be a safe integer in minor units, and the projector SHALL reject a non-integer, unsafe-integer, or floating-point amount rather than coercing it.
2.9 WHEN a Split_Set is committed THEN the projector SHALL append one `split-committed` event per bundle to the Session_Log, carrying the bundle identity and the split count.
2.10 WHEN a bundle's legs resolve to a vendor identifier absent from the Vendor Registry THEN the projector SHALL abort the enclosing bundle commit with a typed reason, and SHALL NOT create an implicit vendor row.
2.11 Every split in one bundle SHALL carry that bundle's settlement currency. The projector SHALL reject a Split_Set in which two splits declare different currencies; cross-currency splitting is out of scope for this increment.

### Requirement 3 — Commission Rule Evaluation

**User Story**: As a Platform Operator, I want commission expressed as a declared rule evaluated at split time, so that changing a rate is a data change rather than a deploy. *(Source: US-8, ADR-5)*

#### Acceptance Criteria

3.1 WHEN a gross amount, a Commission_Rule, and a currency are supplied THEN the evaluator SHALL return a commission amount, a net amount, and the rule revision applied, or a typed rejection.
3.2 WHEN a commission is evaluated THEN `gross` SHALL equal `commission` plus `net` exactly, in minor units, with no residual and no rounding leak.
3.3 WHEN a commission is evaluated THEN `commission` SHALL be greater than or equal to zero and less than or equal to `gross`, and `net` SHALL be greater than or equal to zero.
3.4 WHEN the same gross, rule revision, and currency are evaluated again THEN the returned commission and net SHALL be reproduced bit-for-bit.
3.5 WHEN a stored split row and its recorded rule revision are read back THEN re-evaluating that rule revision against the recorded gross SHALL reproduce the recorded commission exactly. *(This is the US-8 VCC.)*
3.6 WHEN a rule is unresolvable, malformed, declares a rate outside the closed valid range, or cannot be evaluated for the supplied gross THEN the evaluator SHALL return a typed rejection, and the system SHALL NOT substitute a zero commission or a default rate.
3.7 WHEN a tiered rule is evaluated THEN tier boundaries SHALL be evaluated deterministically with an explicitly specified inclusivity at each boundary, and two adjacent tiers SHALL NOT both match one gross amount.
3.8 The evaluator SHALL hold no rate constants. Every rate, tier boundary, and rule shape SHALL originate from stored rule data carrying a revision identifier.
3.9 The evaluator SHALL be a pure function with no storage access, no clock access, and no network access, so that its property tests require no fixture beyond its inputs.

### Requirement 4 — Payout Dispatch, Ordering, and Idempotence

**User Story**: As a Supplier, I want my net payout dispatched exactly once per settled split, after on-chain settlement is verified, so that I am neither double-paid nor paid for a transaction that never settled. *(Source: US-9, ADR-6)*

#### Acceptance Criteria

4.1 WHEN a payout dispatch is evaluated THEN the system SHALL require a recorded settlement-verified event for that split's bundle, and SHALL block dispatch in its absence.
4.2 WHEN a payout dispatch is evaluated THEN the system SHALL require an `active` vendor verdict per Requirement 1.6, and SHALL block dispatch in its absence.
4.3 WHEN either precondition in 4.1 or 4.2 is absent THEN the system SHALL fail closed to a `blocked` state with a typed reason, and SHALL NOT default, infer, or assume either precondition.
4.4 WHEN the Session_Log is read per split THEN zero dispatch attempts SHALL be recorded whose sequence number precedes that split's settlement-verified event. *(This is the ordering half of the US-9 VCC.)*
4.5 WHEN dispatch is attempted for a `split_id` that already reached a terminal `settled` state THEN the system SHALL return the prior recorded result and SHALL NOT issue a second outward movement.
4.6 WHEN the full payout record set is read THEN at most one dispatch per `split_id` SHALL be in a terminal `settled` state. *(This is the idempotence half of the US-9 VCC.)*
4.7 WHEN an outward movement is requested THEN the system SHALL supply an idempotency key derived deterministically from `split_id`, and SHALL reuse that exact key on every retry for that split.
4.8 WHEN a dispatch attempt fails retryably THEN the system SHALL retry within the bounds already defined in-repo for the existing pending queue — at most five attempts, at most a thirty-second interval — and SHALL NOT introduce a second, independent set of retry constants.
4.9 WHEN two consecutive attempts produce no change in the recorded dispatch result THEN the system SHALL stop retrying, transition the payout to terminal `failed`, and record the last observed result and the terminal reason.
4.10 WHEN a payout reaches terminal `failed` THEN the system SHALL NOT retry automatically thereafter, and resolution SHALL require an operator decision recorded against that `split_id`.
4.11 WHEN dispatch state changes THEN the system SHALL append the corresponding `payout-dispatched`, `payout-settled`, or `payout-failed` event to the existing Session_Log, and SHALL NOT create a second log store.
4.12 Dispatch SHALL be triggered by a Durable Object alarm and SHALL call the existing in-repo net-settlement route through a service binding. The system SHALL NOT add a Queues binding and SHALL NOT call an external payout provider directly. *(ADR-6)*
4.13 Dispatch SHALL NOT be performed inside the bundle-commit transaction.

### Requirement 5 — Operator Audit Surface

**User Story**: As a Platform Operator and as a Solo Founder / Auditor, I want vendor state, commission rules, split rows, and payout positions visible on one operator surface and reconstructible from stored rows alone. *(Source: US-6 and US-10 operator halves)*

#### Acceptance Criteria

5.1 WHEN the Vendor Settlement Canvas is projected THEN it SHALL render every registered vendor's identifier, lifecycle state, referenced commission rule revision, and outstanding payout position.
5.2 WHEN the canvas is projected THEN its rendered vendor list SHALL match the underlying vendor record set for the same read, with zero entries present in one but not the other.
5.3 WHEN the canvas is requested under any scope other than Operator_Scope THEN the system SHALL refuse the projection using the existing scope-key guard, and SHALL NOT expose vendor rows, split rows, or payout positions to a Shopper or Vendor client in this increment.
5.4 WHEN concurrent canvas states are merged THEN the merge SHALL be deterministic and order-independent, producing the same result for any interleaving of the same updates.
5.5 WHEN the operator surface is used offline THEN queued operator changes SHALL be retained in order and SHALL converge on reconnect without dropping a queued change, reusing the existing offline queue rather than a new one.
5.6 WHEN the canvas is rendered at mobile width THEN every vendor row's identity, state, and payout position SHALL remain readable without horizontal scrolling, consistent with the repository's mobile-first requirement.
5.7 WHEN a canvas row is rendered THEN it SHALL use the shared Key-Type-Value row contract and semantic HTML elements already established for operator surfaces, and SHALL NOT introduce a private table or list layout.

### Requirement 6 — Auditability from Stored Rows

**User Story**: As a Solo Founder / Auditor, I want the whole split-and-payout chain reconstructible from stored rows, so that a disputed amount is answered from evidence rather than from trust. *(Source: US-10)*

#### Acceptance Criteria

6.1 WHEN any `split_id` is read THEN the stored rows alone SHALL yield the bundle identity, the covered leg identities, the vendor identity, the commission rule revision applied, the gross, commission, and net amounts, the settlement currency, the payout state, and the ordered Session_Log events for that split.
6.2 WHEN any `split_id` is read THEN zero fields SHALL require recomputation from live external state to be interpretable. *(This is the US-10 VCC.)*
6.3 WHEN a payout reaches any terminal state THEN the attempt count and the terminal reason SHALL be recorded against that `split_id`.
6.4 WHEN Session_Log events for one session are read THEN they SHALL be returned in monotonic sequence order, and the sequence SHALL be assigned by the log store rather than by any caller.
6.5 WHEN a new Session_Log event type introduced by this increment is appended THEN the store SHALL reject an event whose type is outside the extended closed set, and SHALL reject a payout event carrying no vendor identifier.

### Requirement 7 — Clean-Room Derivation

**User Story**: As a Solo Founder, I want every module in this layer independently derived against this repository's own primitives, so that the settlement path carries no dependency surface I do not own line-by-line. *(Source: ADR-4)*

#### Acceptance Criteria

7.1 The implementation SHALL NOT install any Forbidden_Specifier in any dependency scope, including `dependencies`, `devDependencies`, `optionalDependencies`, any workspace manifest, and any intentional transitive addition.
7.2 The implementation SHALL NOT contain code, schema DDL, migration content, configuration, test content, fixture content, or prose copied or adapted from Mercur or Medusa, in whole or in part.
7.3 The implementation SHALL NOT fork, vendor, or submodule either project into this repository.
7.4 The implementation SHALL NOT place a hosted Mercur or Medusa endpoint on any runtime path.
7.5 The implementation SHALL NOT reuse either project's entity names, field names, or API shapes verbatim; native vocabulary SHALL be used throughout.
7.6 WHERE a module's shape was arrived at by studying either project's published architecture material, lineage MAY be recorded as a one-line comment naming the pattern and stating that it was independently derived. Such a comment SHALL NOT link to or quote either project's source.
7.7 No requirement in this document SHALL be satisfiable by adding `json-rules-engine` or `xstate`. Commission rule evaluation and vendor lifecycle SHALL be hand-rolled in the idiom already present in this repository. *(ADR-5)*
7.8 WHEN the clean-room boundary is checked THEN a repository scan SHALL fail on the presence of any Forbidden_Specifier in any manifest or any import, so that the boundary is enforced rather than asserted.
7.9 The clean-room scan in 7.8 SHALL be implemented and passing before any other component in this increment is implemented, because a boundary that cannot fail is not a boundary.

### Requirement 8 — Verification Obligations

**User Story**: As an Orchestrator, I want each component's correctness carried by a named check and a stated property, so that a rung is earned from evidence rather than asserted. *(Source: governing guidelines, Verification Strategy)*

#### Acceptance Criteria

8.1 Every task in the derived task list SHALL state its Named_Check before dispatch, phrased exactly as it is invocable.
8.2 Every code-bearing task SHALL add or extend automated tests covering the behaviour it introduces.
8.3 Every stated correctness property SHALL have exactly one executable property test with its class named — round trip, invariant, metamorphic, idempotence, confluence, or error condition — with a minimum iteration count set and shrinking enabled.
8.4 Property tests SHALL use the property-based testing library already pinned in this repository, and SHALL NOT introduce a second one.
8.5 The following SHALL each be covered by an invariant property test: split conservation with zero Residual (2.3), leg partition exactness (2.4), `gross = commission + net` (3.2), amount non-negativity and bounds (3.3), and integer-only arithmetic (2.8).
8.6 The following SHALL each be covered by an idempotence property test: split re-projection (2.6) and payout dispatch per `split_id` (4.5, 4.6).
8.7 Largest_Remainder allocation SHALL be covered by a metamorphic property test asserting that permuting the input vendor order does not change the allocated integer amounts.
8.8 Vendor lifecycle SHALL be covered by an error-condition property test asserting that every transition absent from the frozen table is rejected and leaves state unchanged.
8.9 Canvas merge SHALL be covered by a confluence property test asserting order-independent convergence (5.4).
8.10 Every bug fix in this increment SHALL first add a check that fails on the unfixed state.
8.11 Every task SHALL run the repository's existing aggregate verification lane for the affected surface, not only its own new check.
8.12 A new focused sub-gate SHALL be added to the existing aggregate commerce gate. The implementation SHALL NOT stand up a parallel check pipeline.
8.13 No Evidence Reference SHALL be emitted for a check that was not run in the task that emits it, and no Evidence Reference SHALL record an assertion that a result exists in place of the result.

### Requirement 9 — Deploy Boundary and Irreversibility

**User Story**: As an Operator, I want every irreversible or boundary-crossing operation in this increment gated on my explicit per-occurrence decision. *(Source: Deploy Boundary Register v0.3.0 additions, governing guidelines Tool Permission & Blast Radius)*

#### Acceptance Criteria

9.1 Every task SHALL execute in the `authoring` lane. No task SHALL mutate the Prod mirror or any Cloudflare route.
9.2 Every task's capability class SHALL be one of read, local write, or local execute. No task SHALL carry an environment-mutate, irreversible, or boundary-crossing class.
9.3 The D1 migration authored by this increment SHALL be applied locally only. Remote application SHALL require an explicit per-occurrence operator decision and SHALL NOT be performed by any task in this increment.
9.4 No task SHALL issue a real outward payout movement. Payout dispatch SHALL be exercised against a local or stubbed net-settlement surface, and a real movement SHALL require an explicit per-occurrence operator decision.
9.5 All four Deploy Boundary Register rows added in source specification v0.3.0 SHALL read `closed` at the start and at the end of this increment.
9.6 WHEN a task requires a capability wider than its grant THEN it SHALL return `blocked` with the requested operation and target boundary recorded, and SHALL NOT widen its own grant.
9.7 A vendor activation SHALL require an authenticated operator decision and SHALL NOT be inferred from row presence, payout-account presence, or elapsed time.
9.8 No task SHALL transmit project content, credentials, or vendor data to an external endpoint.

### Requirement 10 — Repository Conventions

**User Story**: As a maintainer, I want this layer indistinguishable in style and structure from the code already in this repository. *(Source: engineering contract in the governing session-start workflow)*

#### Acceptance Criteria

10.1 Every authored file SHALL remain below 600 lines and SHALL carry exactly one declared responsibility, split by owner and behaviour rather than by arbitrary line slices.
10.2 Money SHALL be represented as safe-integer minor units throughout, with the field-name suffix convention already established in this repository, and SHALL NOT be represented as a floating-point value at any layer.
10.3 Validation SHALL follow the repository's dominant idiom — violation-collecting or result-object validators returning typed outcomes — and SHALL NOT use exceptions for expected control flow.
10.4 New identifier types SHALL be added as branded primitives in the existing typed-contracts module rather than in a new parallel type module.
10.5 New persistence keys SHALL be added to the existing scope-key module, following the established `table_name:record_id` convention, and SHALL NOT be constructed inline at call sites.
10.6 New SQL SHALL follow the established migration conventions: next sequential file number, snake_case identifiers, TEXT primary keys, ISO TEXT timestamps, inline CHECK enums, explicit composite UNIQUE constraints, and prefixed index names.
10.7 Settlement currency and amount bounds SHALL be read from existing configuration rather than redeclared, and no machine-specific path, credential, account identifier, or environment-specific default SHALL be persisted in source, fixtures, tests, or documentation.
10.8 New state and rule modules SHALL match the neighbouring directory's formatting conventions rather than a global rule, and exported constant maps SHALL be frozen.
10.9 Defects SHALL be neutralised at the owning source rather than by adding a downstream mask, alias, compatibility remap, or backfill.
10.10 The existing repository hygiene gate SHALL pass for every changed file.

## Bridge Coverage

| Source VCC | Origin | Covered by |
|---|---|---|
| Zero payout dispatches to a non-`active` vendor | US-6 | 1.6, 1.8, 4.2, 8.8 |
| Complete Split_Set in the same committed state; gross sum equals settled total with zero Residual | US-7 | 2.1, 2.2, 2.3, 8.5 |
| `gross = commission + net`; commission bounded; rule revision reproduces recorded commission | US-8 | 3.2, 3.3, 3.5, 8.5 |
| At most one terminal `settled` dispatch per split; no dispatch before settlement verification; retry returns prior result | US-9 | 4.4, 4.5, 4.6, 8.6 |
| Whole chain reconstructible from stored rows with zero recomputation from live state | US-10 | 6.1, 6.2, 6.3 |
| Clean-room boundary enforced, not asserted | ADR-4 | 7.1–7.9 |
| Hand-rolled rules and lifecycle; no rules-engine or state-machine library | ADR-5 | 7.7, 3.8, 3.9, 1.4, 1.5 |
| Same-transaction projection; alarm-driven dispatch; no Queues; no new payout provider | ADR-6 | 2.1, 2.2, 4.12, 4.13 |
| Four new Deploy Boundary rows remain `closed` | Deploy Boundary Register v0.3.0 | 9.1–9.8 |

**Coverage ratio**: 9 source VCCs and ADR directives / 9 covered = **9/9**. Zero source VCCs are uncovered; zero requirements here lack a source.

## Non-Requirements

Stated explicitly so their absence is a decision rather than an oversight. Each maps to a "Won't (this increment)" or "Out of Scope" entry in the source specification.

- A vendor-facing self-serve dashboard or vendor authentication. The operator canvas is the interim surface.
- KYC, sanctions screening, or payout-account verification. See Requirement 1.9 for what `active` therefore does not mean.
- Cross-currency splits or FX within a single bundle. See Requirement 2.11.
- Vendor-initiated refunds, chargebacks, or dispute workflows.
- Marketplace fee *policy*. The commission mechanism is required; the rate is not decided here.
- A dead-letter surface for terminally failed payouts. Requirement 4.10 routes these to an operator decision instead, consistent with ADR-6's stated negative consequence.
- Any Queues binding, any external payout provider, and any Forbidden_Specifier.

## Open Questions Carried From the Source Specification

These are recorded as unresolved rather than silently defaulted. None blocks the arithmetic; all four block onboarding a real second-party vendor, and each needs a recorded operator decision before that happens.

1. Is commission owed on the gross leg amount, or on the leg amount net of third-party fees the platform never receives? Specified here as gross-of-leg-amount so the choice is visible; a change would alter what Requirement 3.2's `gross` denotes.
2. Does a vendor's payout account belong on the vendor row, or in the existing wallet-profile-link model that already stores an address digest, chain identifier, and active/revoked status? Requirement 1.10 requires a resolvable commission rule reference but deliberately does not fix the payout-account location.
3. Should a suspended vendor's already-committed splits freeze or remain dispatchable? Requirement 1.7 specifies freeze as the safer default; the alternative remains an operator decision.
4. Does the platform's own commission need its own vendor row, so conservation is checkable as one sum over all counterparties including the platform? Attractive for audit, but it overloads a lifecycle with an entity that can never be suspended. Left open; Requirement 2.3 is satisfiable either way.
