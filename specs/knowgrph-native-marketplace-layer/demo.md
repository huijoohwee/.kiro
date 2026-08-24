---
title: "Knowgrph Clean-Room Native Marketplace Layer — Demo Walkthrough"
doc_type: "Spec Demo"
schema: "kiro-spec-demo/v1"
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
demo_runtime: "Dev only — GitHub/knowgrph via npm run dev:apex"
demo_duration_target: "under 7 minutes"
requirements_baseline: ".kiro/specs/knowgrph-native-marketplace-layer/requirements.md v0.1.0"
design_baseline: ".kiro/specs/knowgrph-native-marketplace-layer/design.md v0.1.0"
tasks_baseline: ".kiro/specs/knowgrph-native-marketplace-layer/tasks.md v0.1.0"
source_specification: "knowgrph/docs/documents/knowgrph-agentic-commerce-platform-prd-tad-adr.md v0.3.0"
governing_contracts:
  - "huijoohwee.github.io/guidelines/agentic-sdlc-guidelines.md"
  - "agentic-canvas-os/docs/START-WORKFLOW.md"
  - "agentic-canvas-os/docs/AGENTS.md"
  - "knowgrph/AGENTS.md"
---

# Demo Walkthrough

A presenter script, run entirely in the Dev lane. Requirement, property, and task numbers reference the sibling documents and are not restated here.

**Nothing in this demo moves real money.** The payout rail port is stubbed for the whole increment *(Requirement 9.4)*, the D1 migration is applied locally only *(Requirement 9.3)*, and all four Deploy Boundary rows read `closed` throughout. Say this out loud at the start — the single most likely misreading of this demo is that a payout was actually dispatched.

## What This Demo Proves

| Observable claim | Independently verified by |
|---|---|
| A four-leg bundle across three suppliers splits into three settlement rows whose gross amounts sum to the settled total with zero residual | `npm run check:marketplace-settlement` (`cp-14`, `cp-15`) |
| Commission is a data change, not a deploy, and `gross = commission + net` holds on every row with no rounding leak | `npm run check:marketplace-settlement` (`cp-16`, `cp-24`) |
| A suspended supplier's payout stops without a code change, and an `approved`-but-not-`active` supplier is also blocked | `npm run check:marketplace-settlement` (`cp-21`, `cp-23`) |
| A payout dispatched twice moves money once | `npm run check:marketplace-settlement` (`cp-19`) |
| A disputed amount is answerable from stored rows alone, with nothing recomputed from live external state | `tests/integration/marketplace-wiring.test.mjs` |
| Zero foreign commerce framework code, schema, or dependency entered the repository | `node --test tests/scans/no-foreign-commerce-dependency.test.mjs` |
| Nothing crossed a deploy boundary | `tests/process/deploy-boundary.test.mjs` |

## What This Demo Does Not Prove

Stated up front, because a settlement demo invites exactly these over-readings.

- **`active` is not a compliance status.** It means the operator marked the vendor active. It is not KYC, not sanctions screening, not a verified banking relationship. None of those exist in this repository *(Requirement 1.9)*.
- **No real payout occurred.** The rail port is stubbed. A real movement is an irreversible operation requiring an explicit per-occurrence operator decision.
- **No `delivered_rung` is earned here.** Everything in this demo is `authoring`-lane evidence. Prod mirror and Cloudflare publication are separate, gated, and untouched.
- **The commission *policy* is undecided.** The mechanism works; whether the platform takes 3% or 12% is an open question, not a demonstrated decision.
- **Four open questions remain open.** Commission base, payout-account identity, suspended-vendor freeze semantics, and platform-as-vendor. None blocks the arithmetic; all four block onboarding a real second-party supplier.

## Setup — 40 seconds

```sh
cd "$GITHUB_ROOT/knowgrph"
git status --short --branch          # expect clean, on the task lane
npm run storage:d1:migrate:local     # local schema only — never :remote
npm run dev:apex
```

Say: "Local D1, local Vite, stubbed payout rail. No remote anything."

## Act 1 — The Clean-Room Boundary, First — 45 seconds

Run the scan before showing any feature:

```sh
node --test tests/scans/no-foreign-commerce-dependency.test.mjs
```

Then show that it can actually fail, using the synthetic fixture from Task 1:

Say: "Mercur and Medusa are both MIT. Copying either would have been legal and faster. We read their architecture and wrote our own, and this check is what makes that a boundary rather than a claim. It fails on a `@medusajs/` or `@mercurjs/` specifier anywhere in any manifest, on an import of either namespace, and on `json-rules-engine` or `xstate` — the two libraries the original addendum proposed and this design declined."

Why lead with this: until a check can fail, a clean-room boundary is an intention. Demonstrating that it fails correctly is the whole point, and it takes forty-five seconds.

## Act 2 — Onboard Three Suppliers — 70 seconds

Register three vendors, then walk the lifecycle:

1. Register all three. Show each lands in `pending_review` regardless of what the caller asked for *(Requirement 1.3)*.
2. Approve two. Activate one. Leave one at `approved`.
3. Attempt an illegal transition — `pending_review → active`, skipping approval. Show the typed rejection naming the current state and the requested transition, and show the stored state unchanged *(Requirement 1.5)*.

Say: "Four states, a frozen table, and every transition not in that table is a rejection that changes nothing. `suspended` goes back to `approved`, never straight to `active` — reinstating a supplier always passes back through an explicit activation decision."

Point at the vendor left at `approved`: "This one will matter in Act 5."

## Act 3 — Split a Four-Leg Bundle Across Three Suppliers — 100 seconds

Commit a bundle whose four legs map to three suppliers, one of whom owns two legs. Use a settled total that does not divide evenly — that is the interesting case.

Show the projected Split_Set:

- Three rows, not four. The two-leg supplier gets one row covering both legs *(Requirement 2.5)*.
- Gross amounts sum to the settled total **exactly**. Point at the residual: zero *(Requirement 2.3)*.
- Every leg appears in exactly one row's covered-leg list *(Requirement 2.4)*.
- Every amount is an integer in minor units. No decimals anywhere on screen *(Requirement 2.8)*.

Say: "The total didn't divide evenly. Largest-remainder allocation floors every share, then hands the leftover units to the largest fractional remainders, breaking ties by vendor identifier. The sum equals the total by construction — conservation isn't checked and corrected, it can't fail. And tie-breaking by identifier rather than input position is why permuting the supplier order gives byte-identical integers."

Then show the abort path, which is the more important half:

Commit a bundle whose legs reference an unregistered supplier. Show the **bundle commit aborts** — no bundle, no partial splits *(Requirement 2.2, 2.10)*.

Say: "This is the deliberate trade. A commission or split defect blocks settlement. We chose that over a bundle sitting in committed state with incomplete splits, because a partial split set is a silent financial defect and a failed commit is a visible one."

## Act 4 — Change a Rate Without a Deploy — 60 seconds

1. Show the commission rule row for one supplier: a flat rate in basis points.
2. Write a new rule revision with a different rate. No rebuild, no restart.
3. Commit a new bundle. Show the new commission applied, and the new rule revision recorded **on the split row itself** *(Requirement 3.5)*.
4. Read back an older split row and re-evaluate its recorded rule revision against its recorded gross. Show it reproduces the recorded commission bit-for-bit.

Say: "Basis points, so the rate itself is an integer — that removes the last place a float could enter. `net` is defined as `gross − commission`, so `gross = commission + net` holds by construction. And because the revision is stored on the row, changing a rate never rewrites history: an old split still reproduces its own commission exactly."

Then show a malformed rule — a rate outside `[0, 10000]`, or overlapping tiers:

Say: "Typed rejection. Not a zero commission. A silent zero is a revenue defect that looks like a successful settlement, so the evaluator refuses rather than defaults."

## Act 5 — Payout Gates and Idempotence — 90 seconds

Four beats, each one a gate:

1. **No settlement verification yet.** Trigger dispatch. Result: `blocked`, with the reason recorded *(Requirement 4.1)*. Say: "Not an error, not a retry storm. A recorded blocked state, because the precondition might still arrive."
2. **The `approved`-not-`active` supplier from Act 2.** Verify settlement, then dispatch. Result: `blocked`, distinct reason *(Requirement 4.2, 1.6)*. Say: "`approved` is not `active`. Each blocked state says which precondition was missing, so a blocked payout can answer why."
3. **The happy path.** Verify settlement for the `active` supplier, dispatch. Show `dispatched` then `settled`, with the session-log events in sequence order *(Requirement 4.11)*.
4. **Dispatch the same split again.** Show it returns the prior recorded result with **no second outward call** *(Requirement 4.5, 4.6)*. Show the idempotency key is identical across both attempts *(Requirement 4.7)*.

Then suspend the supplier and show the freeze: existing splits remain, untouched, non-dispatchable *(Requirement 1.7)*.

Say: "Suspension doesn't delete or alter a single split row. The freeze is the absence of a dispatch permission, not a mutation. Nothing was destroyed to stop the money."

Optionally, if time allows, trip the circuit breaker: two consecutive attempts with an unchanged result → terminal `failed`, attempt count and terminal reason recorded, no further automatic attempts *(Requirement 4.9, 4.10)*.

Say: "Terminal failed means it waits on an operator, not on a retry loop. There's no dead-letter queue here — ADR-6 names that as an accepted cost, and this is what it looks like."

## Act 6 — The Operator Surface — 60 seconds

Open the Vendor Settlement Canvas at mobile width.

- Every supplier: identifier, lifecycle state, commission rule revision, outstanding payout position *(Requirement 5.1)*.
- Readable without horizontal scrolling; shared Key-Type-Value rows, semantic elements *(Requirement 5.6, 5.7)*.
- Request the canvas under a non-operator scope. Show the refusal *(Requirement 5.3)*.
- Go offline. Make an operator change. Reconnect. Show it converged with nothing dropped *(Requirement 5.5)*.

Say: "Same operator canvas pattern as the Phase 1 registry, same scope guard, same offline queue. New node type, not a new dependency."

## Act 7 — Answer a Dispute From Stored Rows — 50 seconds

Pick one `split_id` and reconstruct it from storage alone:

Bundle identity · covered legs · supplier · commission rule revision · gross · commission · net · currency · payout state · attempt count · terminal reason · ordered session-log events.

Say: "Nothing here was recomputed from live external state to be readable. That's the whole point of writing the split at commit time instead of reconstructing the arithmetic later — a disputed amount is answered from evidence, not from re-running code and trusting that it still behaves the way it did then."

## Act 8 — Close the Boundary — 45 seconds

```sh
npm run check:marketplace-settlement
npm run hygiene:check
npm run check:agentic-commerce-platform
node --test tests/process/deploy-boundary.test.mjs
```

Say: "The new sub-gate is the final clause of the existing aggregate gate, not a parallel pipeline. Boundary test confirms no module references a Prod mirror path or a Cloudflare route, and that the rail port is the only outward-call site in the whole layer — which is what makes that a single-point check instead of a repository-wide grep."

Confirm on screen: all four Deploy Boundary rows `closed`; `delivered_rung` still `undocumented`.

## Closing — 30 seconds

"Six components. Zero new dependencies. Zero new infrastructure categories. Zero new external vendors. The whole supply side of the marketplace runs on the D1, Durable Object, and alarm primitives that were already provisioned, and every invariant that protects the money is a property test rather than a convention.

What we gave up: years of bug-hardening that Mercur and Medusa have already absorbed into equivalent logic. That's a real cost and it's named in ADR-4. The reason it's the right trade here is that the code we wrote instead is small enough to property-test exhaustively and small enough for one person to hold in their head during a money dispute. A foreign framework's internals are neither.

Still open: commission policy, payout-account identity, suspended-vendor semantics, and whether the platform needs its own vendor row. None of them blocks the arithmetic. All four need a recorded decision before a real second-party supplier is onboarded."

## Failure Modes to Rehearse

Each of these is a plausible live failure, with a stated recovery, so the demo degrades gracefully rather than stalling.

| Symptom | Likely cause | Recovery |
|---|---|---|
| Migration reports schema already applied | Local D1 retained state from a prior run | Continue; the demo does not require a fresh schema |
| Split projection aborts on a bundle expected to succeed | A demo supplier was left `pending_review`, or a rule revision does not resolve | Show the typed abort reason — this is Act 3's second half arriving early, not a defect |
| Payout stays `blocked` on the happy path | Settlement-verified event not appended for that bundle | Append it and re-trigger; do not bypass the gate |
| Canvas renders empty | Requested under a non-operator scope | This is Act 6's refusal path; name it as such |
| Aggregate gate slow | It runs the full commerce suite | Run `check:marketplace-settlement` alone for the live demo and report the aggregate result from a prior recorded run, stating that it is a prior run |
| Clean-room scan passes but cannot be shown failing | Synthetic fixture from Task 1 absent | Say so plainly. Without a demonstrated failure the boundary is unproven, and claiming otherwise would be exactly the over-reading this document opens by warning against |
