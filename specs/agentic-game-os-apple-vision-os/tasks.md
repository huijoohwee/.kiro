# Agentic Game OS Apple visionOS Implementation Plan

<!-- Responsibility: Sequence every normative criterion and production promotion gate into auditable source-owned work; Exports: none -->

## Execution rule

This requirements-first plan is incomplete until every checkbox is complete at
one exact protected candidate. A passing unit test, simulator run, deployed
browser artifact, or pull request never promotes another unchecked row. Every
completed row records its repository, source revision, command, exit status,
artifact or receipt digest, and covered criterion identifiers.

## 0. Reconcile the normative contract

- [x] Align failed game-mode exit behavior across Requirement 5.7, Property 19,
  and the design error table: retain the incumbent, invoke its exit once, and do
  not invoke the requested entry handler.
- [x] Align capability dispatch across Requirements 6.6–6.7, the parity matrix,
  and Property 16: dispatch requires both tier support and a projected target.
- [x] Replace the undefined Requirement 6.12 counterfactual with a fail-closed
  user-agent rule that never grants a probe-dependent capability.
- [x] Define an already-active manifest value as an idempotent no-op so
  Requirement 8.10 and Property 6 agree on revision behavior.
- [x] Re-run the complete requirements/design consistency audit and record zero
  contradictory outcomes or undefined oracles.

## 1. Author and preserve the governed document

- [ ] Implement and validate the exact Authored_Document, eleven-field
  frontmatter, three parts, placement rules, rung ordering, and size bound
  (1.1–1.12).
- [ ] Record the pre-write byte length and digest of every Sibling_Document,
  preserve all sibling bytes, and prove the Authored_Document is the sole write
  in its directory (2.1–2.8).
- [ ] Store the preservation baseline and final comparison as immutable receipts.
- [ ] Implement the source-revision-bound Sibling_Preservation_Auditor before
  the first authored-document write.

## 2. Resolve invocation and shared ownership

- [ ] Prove exact `/`, `#`, and `@` invocation resolution, duplicate rejection,
  bounded reads, and zero substitutions (3.1–3.9).
- [ ] Implement the Shared_Substrate export surface, pin record, closed
  ownership-manifest/static-call-graph duplicate audit, incomplete-scan behavior,
  and missing-capability rejection (4.1–4.10).
- [ ] Remove every downstream capability implementation after its upstream owner
  and regression proof land; do not retain compatibility shims.

## 3. Implement game registry and capability detection

- [ ] Implement the capacity-bounded Flight Simulator, City Building Sim, and
  RTS-MMO registry with atomic activation and shared capability identity
  (5.1–5.11).
- [ ] Implement the closed capability tier data, singular evidence source,
  bounded detector, target projection admission, typed gaps, and browser/native
  canonical parity (6.1–6.12).
- [ ] Validate target-specific probe IDs/values and projection rules against the
  capability matrix schema before either adapter can dispatch.

## 4. Establish the Apple production baseline

- [ ] Install and select the declared stable Xcode/Swift/SDK baseline; record and
  reject beta/canary or mismatched toolchain evidence (7.1–7.3, 7.9–7.10).
- [ ] Run the native build, iOS and iPadOS test destinations, and true native visionOS test
  destination with exact destination identifiers and bounded exits
  (7.4–7.7, 7.11).
- [ ] Record complete named physical iPhone, iPad, and Apple Vision Pro matrices
  including model, OS, date, and every pass result (7.8).
- [ ] Record RealityKit Trace/Instruments CPU, GPU, frame-time, thermal, tracking,
  audio, haptic, lifecycle, and comfort evidence on physical hardware.

## 5. Centralize scene manifests and live authoring

- [ ] Land the shared schema, TypeScript parser/printer, Swift decoder/encoder,
  RFC-8785/NFC canonical bytes, bounds, complete errors, and canonical round trips
  (8.1, 8.4–8.9).
- [ ] Make every runtime value manifest-owned; implement bounded live apply,
  idempotent no-op commits, next-revision persistence, rollback, and typed
  rejection (8.2–8.3, 8.10–8.12).
- [ ] Remove GameXR's independent manifest codecs after thin adapters consume the
  shared package.

## 6. Implement offline-first continuity

- [ ] Land canonical-byte SHA-256 journal/snapshot chaining, leases, and the
  sequence/payload-bound FIFO queue plus both storage adapters (9.1–9.11).
- [ ] Prove open/play/commit/close/reopen with all network transports failing,
  zero play-loop egress, append-before-acknowledge, deterministic replay,
  fail-closed corrupt reads, exclusive leases, restart persistence, and queue
  overflow preservation.
- [ ] Remove GameXR continuity semantics from LocalDatabase after its adapter-only
  replacement is proven.

## 7. Bind one protected candidate through delivery

- [ ] Require one sealed Composite_Candidate naming all four protected repository
  revisions and immutable stage ordering for Dev, Prod_Mirror, and
  Delivery_Surface (10.1–10.4, 10.10).
- [ ] Require independent one-use authorizations, fail-closed timeouts, exact
  revision/digest recording, mirror/delivery equality, and one audit entry per
  outcome (10.5–10.8, 10.11–10.12).
- [ ] Prove mirror and delivery boundaries hold no world data (10.9).
- [ ] Remove the controller shortcut that labels delivery completion production
  ready without the component, property, native, and physical receipts.

## 8. Enforce source hygiene

- [ ] Apply fixes only at owning modules and require failing-first regressions;
  reject downstream masks and remove stale behavior completely
  (11.1–11.2, 11.7).
- [ ] Prove portable paths, existing repository references, <=600-line files,
  exactly one responsibility/export record, policy-bound runtime/provider
  literals, and complete typed reports (11.3–11.6, 11.8–11.11).
- [ ] Implement the exact-change-set Path_Portability, File_Size, Hardcode, and
  Single_Responsibility auditors without scanning unchanged preservation inputs.

## 9. Prove zero-infrastructure and bounded cost

- [ ] Record zero-cost local play-loop steps, complete model-bearing cost records,
  bounded harnesses, infrastructure inventory equality, FOSS alternatives, TCO,
  time-to-value, and complete failure records (12.1–12.10).

## 10. Expose the local-first agent surface

- [ ] Implement bounded read/status views and manifest/game-mode/control
  operations with no duplicate API, model, credential, provider, or deployment
  authority (13.1–13.9).
- [ ] Prove read non-mutation, schema rejection, Dev-only control without a gate,
  explicit unresolved fields, and busy-operation isolation.

## 11. Close readiness traceability

- [ ] Give every closed Readiness_Subject exactly one readiness row and one owner;
  declare no rung above its complete surfaced evidence (14.1–14.4, 14.8).
- [ ] Give every criterion 1.1–14.8 at least one re-invocable checklist command,
  observable pass condition, exact source revision, output, exit status, and
  evidence reference; aggregate all gaps in one response (14.5–14.7).

## 12. Execute all correctness properties

- [ ] Properties 1–6: scene codec, agreement, rollback, and live-edit no-op.
- [ ] Properties 7–13: ownership, pins, tiers, evidence, and cross-runtime parity.
- [ ] Properties 14–19: conservative evidence and atomic game registry behavior.
- [ ] Properties 20–26: deterministic continuity, durable ordering, leases,
  offline queue behavior, and queue-overflow preservation.
- [ ] Properties 27–34: invocation, document schema, rung order, paths, size,
  zero-cost accounting, bounds, and cost completeness.
- [ ] Properties 35–41: bounded-loop termination, read/control isolation, deploy
  gates, digest continuity, and validation coverage.
- [ ] Run every property for at least 100 cases or the shared deterministic corpus;
  record seed/corpus digest, shrink or replay input, revision, command, exit, and
  TypeScript/Swift agreement where both targets implement the subject, otherwise
  the named owning runtime and oracle, in one production-readiness receipt.

## 13. Reconcile the selected GameXR lanes

- [ ] Revalidate PR #8 at its exact head, current protected base, declared files,
  package pins, native/browser proofs, required checks, and lifecycle receipts;
  integrate only through repository-owned protected lifecycle.
- [ ] Preserve PR #9's tracked and untracked bytes, recover its exact expired
  cloud-backed owner through a receipt-bound controller, and defer it without
  merge, deletion, stash loss, or scope release by inference.
- [ ] Prove PR #8 contains no PR #9 bytes and no unresolved shared-source
  duplication before selecting it as the candidate.

## 14. Issue the final production receipt

- [ ] Bind one exact protected `agentic-canvas-os`, Knowgrph, GameXR, and
  `huijoohwee.github.io` revision in the Composite_Candidate, plus every shared
  package digest, Dev result, mirror digest, every public route and native
  product digest, all 41 property results, stable Apple toolchain identity, and
  all three physical-device matrices.
- [ ] Independently re-read every protected revision, artifact, route, device
  record, and receipt immediately before promotion.
- [ ] Emit `production-runtime-ready` only when every task above is complete and
  every digest and identity still matches; otherwise emit the complete blocker
  set and retain the lower rung.
