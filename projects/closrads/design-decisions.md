# CLOSRADS — Design Decisions

10 architectural decisions made during development. Each includes the decision, alternatives considered, and the reasoning.

---

## D01 — DRY_RUN defaults to true

**Decision:** `DRY_RUN=true` is the default. Production requires explicitly setting it to `false`.

**Alternatives:** Default to `false` (live), require explicit flag to enable dry-run.

**Reasoning:** A misconfigured run that doesn’t apply changes is recoverable. A misconfigured run that applies wrong geo targeting to live ad sets is not. Safe default wins. This also means any new deployment is safe until explicitly activated.

---

## D02 — All-zeros demand state triggers fail-safe abort

**Decision:** If the demand file produces an all-zeros state (every region = 0), the sync aborts before touching any ad set.

**Alternatives:** Apply the all-zeros state (disable all regions), continue with a warning.

**Reasoning:** All-zeros is almost certainly a data pipeline failure, not a legitimate business state. Disabling all ad set regions on a bad file would cause immediate revenue impact. Abort + alert is always safer than applying a suspect state.

---

## D03 — deepcopy targeting before modification

**Decision:** `_build_new_targeting()` receives a `deepcopy` of the current targeting dict, never the original.

**Alternatives:** Modify in place, use shallow copy.

**Reasoning:** Facebook’s SDK targeting objects are mutable dicts. Modifying in place risks corrupting the object used for the idempotency check (current vs desired comparison). deepcopy guarantees the comparison is always between the unmodified original and the proposed new state.

---

## D04 — Idempotency check before every update

**Decision:** Before calling `update_adset_geo()`, compare current targeting regions against desired regions as sets. Skip the API call if they match.

**Alternatives:** Always call the API (Facebook deduplicates), skip the check for simplicity.

**Reasoning:** Facebook API calls count against rate limits. Idempotency check eliminates unnecessary calls on days when demand hasn’t changed. Also makes the sync report more accurate (skipped vs updated counts).

---

## D05 — No credentials in code or repository

**Decision:** All credentials (FB tokens, Slack webhook) come from environment variables injected at runtime by GitHub Actions secrets.

**Alternatives:** `.env` file committed to repo, hardcoded for testing.

**Reasoning:** A credential committed to a repo — even briefly — is permanently exposed via git history. GitHub Secrets provides encrypted storage with audit logging and is the standard for CI/CD credential management.

---

## D06 — Separate module per responsibility

**Decision:** `facebook_client.py`, `sync.py`, `notifier.py`, `main.py` — one module per responsibility.

**Alternatives:** Single script, two modules (client + orchestrator).

**Reasoning:** Each module has a single axis of change. `facebook_client.py` changes when the Facebook API changes. `sync.py` changes when business logic changes. `notifier.py` changes when the notification channel changes. This separation makes the 18 tests possible — each module can be tested in isolation.

---

## D07 — Tenacity for retry logic

**Decision:** Use `tenacity` library for retry on Facebook API errors. Config: 3 attempts, wait 5s/10s/20s (exponential).

**Alternatives:** Manual retry loop with `time.sleep()`, no retry.

**Reasoning:** Manual retry loops are boilerplate that always has edge cases (what to catch, when to give up, how to log). Tenacity handles all of this declaratively with one decorator. The retry is per-adset, not per-sync, so one flaky adset doesn’t cause the entire sync to retry.

---

## D08 — fb_region_keys.json committed to repo

**Decision:** The mapping of demand states to Facebook region keys lives in a committed JSON file, not in code or environment variables.

**Alternatives:** Hardcoded dict in Python, environment variable.

**Reasoning:** Region mappings change occasionally (new markets, removed regions). A JSON file is editable without touching Python code, is reviewable in PRs, and is testable — the test suite validates that all keys are valid Facebook format. An env var would be unwieldy for a mapping of this size.

---

## D09 — System User token, not personal user token

**Decision:** The `FB_ACCESS_TOKEN` is a Facebook System User token from the Business Manager, not a personal user token.

**Alternatives:** Personal user token, page token.

**Reasoning:** Personal tokens expire and are tied to an individual’s Facebook account. If the account is suspended or the person leaves, the integration breaks. System User tokens are account-independent, have configurable scopes, and don’t expire unless explicitly revoked.

---

## D10 — Per-adset error isolation

**Decision:** Each adset update runs inside its own try/except. A failure on adset N is logged and counted but does not abort the sync for adsets N+1 onwards.

**Alternatives:** Abort entire sync on first error, wrap entire loop in one try/except.

**Reasoning:** A transient API error on one adset should not prevent updating the remaining adsets. The SyncReport captures error counts and details. `SyncReport.success` is set to `False` if any error occurred, so the operator is alerted, but the sync completes as much work as possible.
