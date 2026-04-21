# CLOSRADS — Activation Plan

Ordered steps to go from current state to first live production cron.

---

## Steps

| Step | Action | Owner | Done when | Blocked by |
|---|---|---|---|---|
| 1 | Mike confirms demand file format and location | Mike | Written confirmation received | — |
| 2 | Resolve IP whitelist for GitHub Actions runner | Juanes | Facebook API call succeeds from GH Actions | Step 1 |
| 3 | Set all 7 GitHub Secrets in the repo | Juanes | Secrets visible in repo Settings | Step 2 |
| 4 | Run GH Actions dry-run manually (`workflow_dispatch`, `dry_run=true`) | Juanes | Workflow completes, report shows correct regions, no API errors | Step 3 |
| 5 | Merge `devlop` → `main` | Juanes | PR merged, `main` is up to date | Step 4 |
| 6 | First live cron runs at 8 AM Colombia | Automatic | Slack report received, ad sets updated, no errors in GH Actions logs | Step 5 |

---

## Blockers

| Blocker | Severity | Status |
|---|---|---|
| Mike confirmation on demand file | Critical | Pending |
| IP whitelist resolution | Critical | Pending — solution not selected yet |
| Facebook System User token scope verification | High | Pending |

---

## Post-activation monitoring (first 7 days)

- Check GitHub Actions logs daily after 8 AM Colombia
- Verify Slack report is received each day
- Spot-check 2-3 ad sets in Facebook Ads Manager to confirm targeting matches expected regions
- Monitor for `SyncReport.success = False` in any run
- After 7 clean runs: consider activation complete

---

## Rollback plan

If a live run produces unexpected geo changes:
1. Set `DRY_RUN=true` in GitHub Secrets immediately
2. Manually revert targeting in Facebook Ads Manager
3. Investigate `SyncReport` and GH Actions logs
4. Fix, re-run dry-run, confirm, then set `DRY_RUN=false` again
