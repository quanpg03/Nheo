# Activation Plan

This page defines the exact ordered steps required to go from **current state** (multi-campaign code on `devlop`, token expired) to **production active** (cron running on `main` with `DRY_RUN=false` every day at 8 AM Colombia for all three campaigns).

Every step includes who owns it, what done looks like, and whether it is a blocker for the next step.

---

## Current State Summary

| Item | Status |
|------|--------|
| Core automation code (single campaign) | ✅ Complete |
| Multi-campaign refactor (Veterans + Truckers + Mortgage) | ✅ Complete, on `devlop` |
| Offline tests (18/18) | ✅ Passing — including after refactor |
| Dry-run — CLOSRTECH all 3 campaigns | ✅ Done 2026-04-25 — all 3 read successfully |
| Dry-run — Facebook all 3 campaigns | ❌ Failed — expired personal session token |
| New System User FB token | ⚠️ Pending Mike approval of permission request |
| IP whitelist for GitHub Actions | ⚠️ Unresolved — main structural blocker |
| GitHub Secrets (per-campaign naming) | ⚠️ Not configured with new naming convention |
| Visual verification (Facebook Ads Manager) | ⚠️ Pending new token |
| `devlop` → `main` merge | ⚠️ Pending |
| First live cron run | ⚠️ Pending |

---

## Step 1 — Get the new System User FB token

**Owner:** Mike (approve) → Juanes (generate + distribute) | **Blocker for:** Steps 2 and 3 | **Status:** ⚠️ Pending Mike approval

The permission approval request was sent to Mike on 2026-04-25. Once Mike approves:

1. Go to Meta Business Manager → SP Insurance Group → Ajustes → Usuarios del sistema → CLOSRADS System User
2. Click "Generar identificador" → select Manus app → generate token
3. Copy the token and drop it into `.env` replacing all three `*_FB_ACCESS_TOKEN` values (same token for Veterans, Truckers, and Mortgage — the System User has access to both ad accounts)

**Done when:** The new token is in `.env` and `python main.py` with `DRY_RUN=true` completes without Facebook token errors.

---

## Step 2 — Second dry-run (all 3 campaigns, CLOSRTECH + Facebook)

**Owner:** Juanes or Nat | **Blocker for:** Step 4 (visual check) | **Status:** ⚠️ Pending Step 1

Run `python main.py` locally with the new token and `DRY_RUN=true`.

**What to verify:**

- All 3 campaigns successfully read CLOSRTECH demand (no auth errors)
- All 3 campaigns successfully reach Facebook (no token expiry errors)
- State counts roughly match April 25-26 CLOSRTECH results: Veterans ~34, Truckers ~9, Mortgage ~30
- No errors in the SyncReport for any campaign

**Done when:** Local dry-run exits with code 0 and the log shows demand data + Facebook adset counts for all 3 campaigns.

---

## Step 3 — Visual verification (Facebook Ads Manager)

**Owner:** Juanes or Nat | **Blocker for:** Step 5 (GitHub Secrets) | **Status:** ⚠️ Pending Step 2

Open the saved Veterans adset URL (`adsmanager.facebook.com/adsmanager/manage/adsets/edit/standalone?act=996226848340777...`) and note the current states listed. Then run `python main.py` with `DRY_RUN=false` locally. Reload the adset and confirm the states changed to match today's CLOSRTECH demand.

This is the ground truth check — it proves the automation makes the exact changes it claims to make.

**Done when:** The adset targeting in Facebook Ads Manager matches what the dry-run log said it would set.

---

## Step 4 — Resolve the IP whitelist

**Owner:** Juanes (decision) + Nat (implementation) | **Blocker for:** Step 6 (cron) | **Status:** ⚠️ Unresolved

Choose one of the options documented in [`github-actions.md`](./github-actions.md):

| Option | Recommended if... |
|--------|------------------|
| Static IP proxy (VPS) | Nheo doesn't have an existing server with a fixed IP |
| Self-hosted GitHub Actions runner | Nheo already has a server that's always online |
| Move cron to Nheo server directly | Team prefers no GitHub Actions dependency |
| Ask Mike to expand CLOSRTECH whitelist | Mike has direct access to CLOSRTECH's admin and the vendor is responsive |

Once decided, implement the chosen solution and verify that a test HTTP request to `demand.php` from the target execution environment returns a valid JSON response (not 403).

**Done when:** A test request to `demand.php` from the chosen runner/environment succeeds.

---

## Step 5 — Configure GitHub Secrets (new per-campaign naming)

**Owner:** Juanes or Nat | **Blocker for:** Step 6 (merge + cron) | **Status:** ⚠️ Not configured

In the GitHub repo: Settings → Secrets and variables → Actions → New repository secret.

The naming convention changed from the original single-campaign names to per-campaign prefixes:

| Secret | Value |
|--------|-------|
| `CLOSRTECH_EMAIL` | Mike's CLOSRTECH email |
| `CLOSRTECH_PASSWORD` | Mike's CLOSRTECH password |
| `VETERANS_CLOSRTECH_CAMPAIGN` | `VND_VETERAN_LEADS` |
| `VETERANS_FB_ACCESS_TOKEN` | New System User token |
| `VETERANS_FB_AD_ACCOUNT_ID` | `act_996226848340777` |
| `VETERANS_FB_CAMPAIGN_ID` | `120238960603460363` |
| `TRUCKERS_CLOSRTECH_CAMPAIGN` | `VND_TRUCKER_LEADS` |
| `TRUCKERS_FB_ACCESS_TOKEN` | Same System User token |
| `TRUCKERS_FB_AD_ACCOUNT_ID` | `act_996226848340777` |
| `TRUCKERS_FB_CAMPAIGN_ID` | `120239404121750363` |
| `MORTGAGE_CLOSRTECH_CAMPAIGN` | `VND_MORTGAGE_PROTECTION_LEADS` |
| `MORTGAGE_FB_ACCESS_TOKEN` | Same System User token |
| `MORTGAGE_FB_AD_ACCOUNT_ID` | `act_1007012848173879` |
| `MORTGAGE_FB_CAMPAIGN_IDS` | `120245305494410017,120241447971000017` |
| `SLACK_WEBHOOK_URL` | Optional. Incoming webhook URL. |

**If old single-campaign secrets exist:** Delete them. The new workflow YAML does not reference them and their presence would be confusing.

**Done when:** All 15 secrets appear in the GitHub repo secrets list.

---

## Step 6 — Manual `workflow_dispatch` dry-run from GitHub Actions

**Owner:** Nat | **Blocker for:** Step 7 (merge) | **Status:** ⚠️ Pending Steps 4 and 5

Once the IP issue is resolved and secrets are configured, trigger the workflow manually from the GitHub Actions UI with `dry_run: true`.

**What to verify in the run log:**

- No auth errors from CLOSRTECH (IP whitelist resolved)
- No auth errors from Facebook (new System User token works)
- Demand data for all 3 campaigns logged
- Adsets found for all 3 campaigns
- Log shows `[DRY RUN] Would update adset...` for each adset
- Exit code 0 (green check in GitHub Actions)

**Done when:** A `workflow_dispatch` dry-run from GitHub Actions completes with exit code 0 for all 3 campaigns.

---

## Step 7 — Merge `devlop` → `main`

**Owner:** Juanes | **Blocker for:** Step 8 (first live cron) | **Status:** ⚠️ Pending Step 6

Create a PR from `devlop` to `main`. Review the diff — it should be the entire CLOSRADS codebase including the multi-campaign refactor.

Merge only after Step 6 passes. The cron workflow is configured to run from `main`, so this merge is what arms the scheduled trigger.

**Done when:** PR merged, `main` contains all current multi-campaign code.

---

## Step 8 — First live cron run (DRY_RUN=false)

**Owner:** Nat (monitor), Juanes (sign-off) | **Blocker for:** Nothing — this is the finish line | **Status:** ⚠️ Pending Step 7

The day after the merge to `main`, the cron fires at 13:00 UTC (8 AM Colombia) with `DRY_RUN=false`.

**What success looks like:**

- Exit code 0
- Log shows actual Facebook API update calls (not dry-run logs) for all 3 campaigns
- Slack notification shows `Status: SUCCESS` with adsets updated count per campaign
- Facebook Ads Manager shows updated geographic targeting on the adsets for Veterans, Truckers, and Mortgage

**What to do if it fails:**

- Check the GitHub Actions log for the exact error and which campaign/step failed
- If CLOSRTECH fails: check IP whitelist, credentials
- If Facebook fails: check token, permissions, adset IDs
- If one campaign fails but others succeed: check campaign-specific config (campaign ID, ad account ID)
- Do NOT attempt a manual fix in Facebook Ads Manager simultaneously — let the automation own the targeting

**Done when:** First live run completes with exit code 0 and Facebook targeting is confirmed updated for all 3 campaigns.

---

## Blockers Summary

| Blocker | Severity | Owner | Next action |
|---------|----------|-------|-------------|
| New FB System User token | 🔴 Critical | Mike → Juanes | Mike approves permission request; Juanes generates and distributes token |
| IP whitelist for GitHub Actions | 🔴 Critical | Juanes | Decide on proxy vs self-hosted runner vs server cron |
| GitHub Secrets (new naming) | 🟡 High | Nat | Configure after IP decision + new token are ready |
| `devlop` → `main` not merged | 🟡 High | Juanes | Merge after Step 6 (GH Actions dry-run) passes |
| `orders.php` returns 404 | 🔵 Low (v2) | Mike | Mike must escalate to CLOSRTECH dev. Not blocking v1. |

---

## Post-Activation Monitoring

Once live, the automation should be monitored for the first 5 days:

- Check GitHub Actions workflow history each morning to confirm all 3 campaigns ran successfully
- Spot-check Facebook Ads Manager on Day 1 and Day 3 for at least one adset per campaign
- Confirm Slack notifications are being received (if configured) — verify one block per campaign
- After 5 successful days, consider the automation stable and reduce monitoring to weekly spot-checks

**Long-term:** The System User token does not expire. The only maintenance scenarios are: Meta changes the `facebook-business` SDK (update `requirements.txt`), CLOSRTECH changes their API contract (update `closrtech_client.py`), or a new campaign needs to be added (add one `.env` entry + one line in `config.py`).
