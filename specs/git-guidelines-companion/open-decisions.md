# Git Guidelines Companion — Resolved Operator Decisions

The Operator resolved all four implementation decisions on 2026-08-05.

Source instruction:

> Decisions: concurrency=unlimited upstream; lease ceiling=24h; divergent artifacts=fix; checker=huijoohwee.github.io/scripts/. Authorize owner-led retirement/handoff/reclaim and a fresh admitted implementation lane.

| # | Decision | Prior default | Resolution | Required implementation |
|---|---|---|---|---|
| 1 | Concurrent current authorities | Maximum 8 live Task_Lanes per repository | `unlimited upstream` | Remove every document-local numeric cap. Permit any number of authenticated current authorities only when normalized declared write sets are pairwise disjoint. An overlap retains exactly one current writer and any newcomer only as a non-writing waiting successor. O1 consumes this protected-upstream rule. |
| 2 | Lease expiry ceiling | 24 hours | `24h` | Keep absolute expiry no later than 24 hours after issuance in R4.7 and R5.3 and in checker/schema coverage. This remains a document-local O2 ceiling and changes no claim identity or authority ordering. |
| 3 | Divergent coordination artifacts | Fix the artifacts | `fix` | Sort `.coordination/dev-source-resolver-write-scope.json` paths in ascending byte order and add `"schema": "agentic-cloud-collaboration-request/v1"` to `.coordination/dev-source-resolver-cloud-request.json`. Do not relax R5.2 or R5.13. |
| 4 | Checker location | `huijoohwee.github.io/scripts/` | `huijoohwee.github.io/scripts/` | Keep the entry point at `huijoohwee.github.io/scripts/check-git-guidelines.mjs`, implementation modules under `huijoohwee.github.io/scripts/lib/git-guidelines/`, and its tests in the same repository. |

## Implementation-Lane Authorization

The same instruction authorizes owner-led retirement, handoff, or reclaim as required by current repository-owned state, followed by a fresh admitted implementation lane.

This record does not itself mint or bypass a claim, lease, fence, preservation receipt, review, protected integration, deployment, or cleanup authority. Before writing, revalidate current state and obtain repository-owned admission for the exact declared write scope.
