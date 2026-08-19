# Implementation Plan: Knowgrph Agentic ECS

## Objective

Deliver the native ECS, KGC contracts, three exact MCP tools, existing-owner Canvas projection, focused tests, and source-aligned documentation. All tasks are required. Production and Cloudflare are outside this plan.

## Tasks

- [x] 1. Establish protected collaboration ownership
  - Run the Agentic Canvas OS `START-WORKFLOW.md` preflight.
  - Repair any failing shared collaboration prerequisite at its canonical owner before ECS edits.
  - Claim branch-exclusive Knowgrph and Agentic Canvas OS documentation lanes.
  - Declare `/ecs.session-start`, `/ecs.world-tick`, `/ecs.decision-persist`, `#agentic-ecs`, `@source.frontmatter`, and `@ecs-session` intent.
  - Record exact starting SHAs and keep Prod/Cloudflare unchanged.
  - Requirements: 8.

- [x] 2. Add component storage
  - Implement the eight supported numeric field types in structure-of-arrays stores.
  - Add atomic schema validation, geometric capacity growth, membership tracking, and a private unique absence sentinel.
  - Prove constructor mapping, round trips, growth preservation, absence, and unsupported-type atomicity with unit/PBT coverage.
  - Requirements: 1, 2.

- [x] 3. Add the opaque World and entity allocator
  - Construct World state from ordered systems, optional decision executor, and optional clock.
  - Implement unique component registration and atomic entity allocation without failed-id gaps.
  - Keep internal stores and accessors private.
  - Prove public opacity and allocation rollback.
  - Requirements: 1, 2.

- [x] 4. Add query and exact public exports
  - Implement ascending intersection over registered component memberships.
  - Reject unknown components without mutation.
  - Export exactly `createWorld`, `allocateEntity`, `registerComponent`, `query`, and `worldTick` from `ecs/index.js`.
  - Add an export-surface regression test.
  - Requirements: 1, 2.

- [x] 5. Add asynchronous transactional ticks
  - Run systems sequentially in construction order.
  - Give Systems only the frozen query/read/write/set/decision/reasoning context and fence public reads/concurrent ticks while a journal is open.
  - Journal each system separately; preserve prior commits and roll back only the failing system.
  - Validate emitted Decisions inside the owning System journal.
  - Skip later systems and reasoning after system failure.
  - Add deterministic order, visibility, rollback, and failure tests.
  - Requirements: 4.

- [x] 6. Add bounded decision execution and cost accounting
  - Inject reasoning through the World rather than importing a model client.
  - Pass an AbortSignal and enforce a 30-second bound.
  - Require exactly one valid non-`none` `cost_logs` entry per executor request, canonicalize it, and retain valid usage when a paired Decision is invalid.
  - Emit one shared-schema zero cost log only after a successful no-reasoning tick.
  - Defer on timeout/unavailable without fabricated decisions, usage, or persistence.
  - Requirements: 4.

- [x] 7. Extend the canonical KGC node contract
  - Add `EcsComponentSchema`, `EcsEntity`, and `EcsDecision` validation under their nested `properties` objects.
  - Preserve all existing KGC document behavior.
  - Preserve inline and block-style JSON-safe node properties, including reserved keys.
  - Add focused valid/invalid schema tests.
  - Requirements: 3, 5, 8.

- [x] 8. Add deterministic hydration
  - Validate all three ECS node kinds, index existing Decisions, require at least one schema/entity, and ignore unrelated nodes.
  - Enforce the exact three-value Decision type enum.
  - Sort schemas and entities by stable identifiers before materialization.
  - Reject duplicates, missing fields, unknown components, bad types, and invalid values as whole-document failures.
  - Return internal World and existing-decision index only after complete success.
  - Add permutation PBT and malformed-input unit coverage.
  - Requirements: 3.

- [x] 9. Add decision-only atomic persistence
  - Read pending validated decisions from the session runtime.
  - Deduplicate by decision id and preserve unrelated frontmatter/body content.
  - Bind the start-time canonical path/device/inode, read through a no-follow handle, and revalidate inside the same-path queue before exclusive temp write and rename; trust only bounded prior queued ECS replacement lineage for already-open sessions.
  - Byte-splice serialized Decisions into block `flow.nodes`, write a unique sibling temporary file, and atomically rename it.
  - Retain pending state and original bytes after injected validation, write, and pre-rename failures; retain the committed replacement identity for retry after post-rename disposal failure.
  - Add idempotency, round-trip, byte-preservation, and failure-injection tests.
  - Requirements: 5.

- [x] 10. Extract and use the existing Canvas text seam
  - Extract `applyChatKgcDocumentTextToCanvas` from the current chat KGC apply owner without changing existing callers.
  - Add the ECS projection adapter over World observations.
  - Render absence as `[absent]`.
  - Keep projection an in-process caller-held-World seam outside the private MCP session and sanitize snapshot/apply failures.
  - Prove no temporary Source File, direct Canvas mutation, alternate renderer, fourth tool, or World mutation.
  - Requirements: 6.

- [x] 11. Define the exact MCP contract
  - Add only `knowgrph.ecs.session_start`, `knowgrph.ecs.world_tick`, and `knowgrph.ecs.decision_persist` to the existing catalog.
  - Use closed argument schemas, an extensible shared result envelope requiring `ok`/`execution_boundary`, and exact `/`, `#`, `@` metadata.
  - Use `kgcPath`, never the rejected `kgcRef`, and never accept caller-supplied decisions.
  - Include `execution_boundary: "dev-only"` in results.
  - Add exact-list and schema tests.
  - Requirements: 7.

- [x] 12. Add safe private session runtime
  - Resolve `.md` paths by realpath under the configured repository root, bind file identity, and reject validation/open or persist-time parent/target swaps.
  - Generate opaque UUID session ids.
  - Enforce TTL, maximum count, and lazy expiry sweep.
  - Dispose on successful or zero-pending persistence; retain after persistence failure; remove inactive expired sessions only after successful disposal, and retain/extend them with a retryable error after expiry disposal failure.
  - Normalize expected failures through explicit public error/deferred allowlists and deterministic System labels.
  - Report every post-commit pending-retention failure (conflicting or invalid/noncanonical Decision) with `tickCommitted`, canonical cost/deferred evidence, and no replay ambiguity.
  - Add root-escaping traversal, symlink/swap escape, secret-metadata, expiry, capacity, lifecycle, and mutation-safety tests.
  - Requirements: 7.

- [x] 13. Wire the existing official MCP SDK stdio server
  - Register the three descriptors and dispatch handlers through `mcp/local-tool-contract.js` and `mcp/server.js`.
  - Add no alternate server, transport, HTTP route, or public session registry.
  - Exercise `initialize`, `tools/list`, and calls with the official SDK client.
  - Prove malformed calls do not escape as uncaught exceptions.
  - Requirements: 7, 8.

- [x] 14. Select ECS validation in the collaboration contract
  - Add `ecs/` as a runtime root.
  - Add ECS unit/PBT to affected runtime validation.
  - Add a focused selector regression test for an ECS source path.
  - Preserve existing collaboration routes.
  - Requirements: 8.

- [x] 15. Publish the Agentic Canvas OS invocation catalog
  - Add the three command tokens, `#agentic-ecs`, and `@ecs-session` to their canonical dictionaries and FACTS direct resolution.
  - Add the monthly planning row and keep `FACTS.md` below 600 lines.
  - State that the catalog is metadata and Knowgrph owns runtime behavior.
  - Run `npm run docs:check`, publish through a protected PR, and record the exact merged SHA.
  - Requirements: 7, 8.

- [x] 16. Reconcile PRD/TAD and Kiro source documents
  - Remove stale FastMCP, `registerSystem`, singular `cost_log`, caller-decision, fake deploy-lane, and compatibility claims.
  - Document the implemented source owners, session lifecycle, safe path, per-system rollback, cost logs, and Dev-only capability boundary.
  - Preserve the no-copy provenance boundary.
  - Requirements: 1-8.

- [x] 17. Run focused validation
  - Run ECS unit and property tests.
  - Run KGC schema and cost-log contract tests.
  - Run MCP tool/session/stdio tests.
  - Run focused Canvas projection tests and Canvas typecheck.
  - Run collaboration selector, hygiene, and diff checks.
  - Verify every changed authored source file is below 600 lines.
  - Re-run parent/target-swap, post-commit conflict, usage-retention, block-YAML, read-isolation, and metadata-sanitization regressions.
  - Requirements: 8.

- [x] 18. Complete protected Dev integration
  - Pin Knowgrph documentation to the exact protected Agentic Canvas OS merge SHA.
  - Re-run the two-peer collaboration gate against the final pair.
  - Publish the Knowgrph commit through its protected Integration Gate.
  - Complete both collaboration leases and verify the exact merged SHAs.
  - Do not mutate Prod, the publication mirror, or Cloudflare.
  - Completion evidence: Agentic Canvas OS `6c3da4029c06ad5cd6167300b2855c47977e2720`; Knowgrph PR head `0c05ff3d3bf1198d59315537ba796eec64c826bf`; protected runtime merge `b58de2bd21819e65e919a5ef9533ef12aa6a8fa6`; protected readiness-record merge `b06d7fcf7b51cf37b965b0ad9431c3904d87a09c`; final two-peer digest `1f896513ae2ffa52c652e60074cf9e576303a861f64eb2ca1762d67c49ea7f0d`.
  - Requirements: 8.

## Dependency Order

```text
1 -> 2 -> 3 -> 4 -> 5 -> 6
               7 -> 8 -> 9
                    8 -> 10
7 + 8 + 9 -> 11 -> 12 -> 13
2..13 -> 14 -> 16 -> 17
1 -> 15 -> 18
14 + 15 + 16 + 17 -> 18
```

## Completion Definition

The implementation is complete only when all boxes above are checked, both protected PRs are merged, the exact app/docs pair passes collaboration proof, and the handoff explicitly reports zero production or Cloudflare mutation.
