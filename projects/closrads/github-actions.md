# CLOSRADS — GitHub Actions & CI/CD

## Workflow overview

CLOSRADS runs on a daily cron via GitHub Actions. The workflow reads the demand file, computes the required Facebook Ads geo changes, and applies them. A manual trigger with `dry_run` input is available for safe testing.

---

## Workflow file: `.github/workflows/sync.yml`

```yaml
name: CLOSRADS Daily Sync

on:
  schedule:
    - cron: '0 13 * * *'  # 8:00 AM Colombia (UTC-5)
  workflow_dispatch:
    inputs:
      dry_run:
        description: 'Run without applying changes'
        required: false
        default: 'true'
        type: choice
        options:
          - 'true'
          - 'false'

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run sync
        env:
          FB_APP_ID: ${{ secrets.FB_APP_ID }}
          FB_APP_SECRET: ${{ secrets.FB_APP_SECRET }}
          FB_ACCESS_TOKEN: ${{ secrets.FB_ACCESS_TOKEN }}
          FB_AD_ACCOUNT_ID: ${{ secrets.FB_AD_ACCOUNT_ID }}
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
          DEMAND_FILE_PATH: ${{ secrets.DEMAND_FILE_PATH }}
          DRY_RUN: ${{ github.event.inputs.dry_run || 'false' }}
        run: python -m closrads.main
```

---

## Secrets (7 required)

| Secret | Description |
|---|---|
| `FB_APP_ID` | Facebook App ID |
| `FB_APP_SECRET` | Facebook App Secret |
| `FB_ACCESS_TOKEN` | System User long-lived access token |
| `FB_AD_ACCOUNT_ID` | Ad account ID (format: `act_XXXXXXXX`) |
| `SLACK_WEBHOOK_URL` | Slack incoming webhook for sync reports |
| `DEMAND_FILE_PATH` | Path or URL to `demand_response.json` |
| `DRY_RUN` | Default runtime dry run flag (`false` in production) |

All secrets are set in: **GitHub repo → Settings → Secrets and variables → Actions**

---

## IP Whitelist Problem

Facebook Marketing API can restrict access to whitelisted IPs. GitHub Actions runners use dynamic IPs, which are not whitelisted by default.

### 4 solutions

| Solution | Cost | Complexity | Recommended |
|---|---|---|---|
| Static IP proxy (e.g. Fixie, QuotaGuard) | ~USD 25/mo | Low | ⚠️ if IP restriction is enforced |
| Self-hosted runner on EC2 with static IP | EC2 cost | Medium | ✅ Best long-term |
| Server cron (run directly on EC2) | Included | Low | ✅ Simplest if EC2 already exists |
| Expand Facebook whitelist to GitHub IP ranges | Free | Low | ❌ Ranges change frequently |

**Current status:** To be resolved before first live cron. See [Activation Plan](activation-plan.md) — Step 2.

---

## Branch strategy

| Branch | Purpose |
|---|---|
| `main` | Production — cron runs from here |
| `devlop` | Development — PRs merge here first |

Promotion path: `feature branch` → `devlop` (PR + review) → `main` (PR after activation dry-run passes)
