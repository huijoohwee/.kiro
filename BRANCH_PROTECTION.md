# Branch protection (for reference)

Recommended `main` branch protection rules for `.kiro`:

- Require at least 1 approving review before merge.
- Require status checks to pass before merge (if you wire CI later).
- Require linear history.
- Restrict push access to trusted users only.
- Allow force pushes only for admins.

Example CLI (run with sufficient permissions):

```bash
gh api repos/huijoohwee/.kiro/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"checks":[]}' \
  --field enforce_admins=true \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field restrictions='{"users":[],"teams":[],"apps":[]}' \
  --field allow_force_pushes=true \
  --field allow_deletions=false \
  --field required_linear_history=true \
  --field allow_fork_syncing=false
```


