# Implementation Plan: Discoverability and IP Protection

## Scope and proof boundary

This plan records the local policy runtime and its readiness boundaries. It does
not authorise a public-origin rewrite, promotion, deployment, or Cloudflare
mutation. Source proof, supervised local-runtime proof, public-estate audit, and
deployed-route proof remain separate gates.

## Tasks

- [x] 1. Implement the local policy authority
  - [x] Add the Surface_Registry, license registry, schema, route classifier,
    discovery generator/parser, publication gate, audit reporter, and fixture-only
    operator ledger.
  - [x] Record 20 requests per 60 seconds for each fetch-on-behalf proxy.
  - [x] Record `LicenseRef-airvio-no-reuse-1.0` for no-reuse build artifacts.
  - _Requirements: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 13_

- [x] 2. Add focused source verification
  - [x] Cover deterministic generation, source-boundary detection, audit
    non-mutation, catalog filtering, fixture promotion, and zero network/model
    execution evidence.
  - [x] Run `npm run surface:test` and `npm run surface:validate` on the exact
    protected Knowgrph revision.
  - _Requirements: 10, 12_

- [x] 3. Make the PRD/TAD and Kiro sources truthful
  - [x] Record the implemented local runtime, resolved values, the full candidate
    diff, and explicit readiness gaps.
  - [x] Add Site_Repository static conformance validation instead of claiming the
    Knowgrph suite validates an external untracked document.
  - _Requirements: 11_

- [ ] 4. Reconcile the public-origin source boundary
  - [ ] Obtain explicit operator authority for the reviewed public-path inventory.
  - [ ] Remove, relocate, or govern every private/unclassified tracked public path
    at the owning source boundary; do not weaken classification to pass the gate.
  - [ ] Regenerate the full discovery candidate and require `surface:check` to
    exit 0 against the exact public-origin revision.
  - _Requirements: 5, 7, 9, 12_

- [ ] 5. Prove exact canonical local runtime
  - [ ] Resolve unrelated canonical-worktree dirt through its owning task.
  - [ ] Reconcile the recorded runtime through the Agentic Canvas OS supervisor.
  - [ ] Require `runtime:local:status` to report the same Knowgrph SHA as fetched
    `origin/main`.
  - _Requirements: runtime readiness boundary_

- [ ] 6. Promote and prove deployed behaviour
  - [ ] Record an exact Operator_Instruction only after Task 4 passes.
  - [ ] Use the repository-owned release controller, then collect exact-release
    public static-route, latency, and zero-model-cost evidence.
  - _Requirements: 3, 9_

## Current status

The local policy runtime is implemented and source-validated. Public-estate and
canonical local-runtime readiness are deliberately blocked. Their blockers must
not be bypassed or represented as deployment proof.
