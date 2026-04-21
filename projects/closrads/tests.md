# CLOSRADS — Tests & Coverage

## Strategy

All 18 tests run **fully offline** — no Facebook API calls, no network access. This is intentional: tests validate business logic, not third-party availability.

Offline isolation is achieved via:
- `conftest.py` injects stub environment variables before any module is imported
- `demand_response.json` fixture provides deterministic test data
- Facebook SDK calls are mocked at the module level

---

## Test files

### `test_state_mapper.py` — 8 tests

Validates the mapping from demand states to Facebook region keys.

| Test | What it protects |
|---|---|
| `test_high_demand_maps_to_full_regions` | High demand activates all configured regions |
| `test_low_demand_maps_to_reduced_regions` | Low demand reduces active region set |
| `test_zero_demand_maps_to_empty` | Zero demand returns empty region set (fail-safe) |
| `test_unknown_state_raises` | Unknown demand state raises ValueError, not silent failure |
| `test_all_valid_states_covered` | Every state in the enum has a mapping |
| `test_region_keys_are_valid_fb_format` | All keys match Facebook's DMA/region format |
| `test_mapper_is_deterministic` | Same input always produces same output |
| `test_empty_demand_file_raises` | Empty/malformed demand file raises early, not mid-sync |

### `test_sync_logic.py` — 10 tests

Validates `run_sync()` orchestration and per-adset update logic.

| Test | What it protects |
|---|---|
| `test_dry_run_makes_no_api_calls` | `DRY_RUN=true` never touches Facebook API |
| `test_idempotency_no_update_when_regions_match` | No API call if targeting already matches desired state |
| `test_update_called_when_regions_differ` | Update IS called when targeting needs to change |
| `test_deepcopy_prevents_mutation` | Original targeting object not mutated during update |
| `test_empty_fb_regions_aborts_sync` | Sync aborts early if fb_region_keys.json produces empty set |
| `test_per_adset_error_isolation` | One adset failure does not abort remaining adsets |
| `test_sync_report_counts_correctly` | SyncReport fields (processed, updated, skipped, errors) are accurate |
| `test_retry_on_transient_error` | Tenacity retries on FacebookRequestError (3 attempts) |
| `test_all_zeros_state_aborts` | All-zeros demand state triggers fail-safe abort, not zero-region update |
| `test_success_flag_false_on_any_error` | `SyncReport.success = False` if any adset fails |

---

## Key fixtures

**`conftest.py`** — Injects required env vars before test collection:
```python
os.environ.setdefault('FB_APP_ID', 'test_app_id')
os.environ.setdefault('FB_APP_SECRET', 'test_secret')
os.environ.setdefault('FB_ACCESS_TOKEN', 'test_token')
os.environ.setdefault('FB_AD_ACCOUNT_ID', 'act_000000000')
os.environ.setdefault('DRY_RUN', 'true')
```

**`demand_response.json`** — Deterministic fixture file with known demand states for all test scenarios.

---

## Coverage gaps (known, accepted)

| Untested path | Reason not tested |
|---|---|
| Real Facebook API call | Requires live credentials — covered by manual dry-run |
| Slack notification delivery | External service — tested via mock only |
| `tenacity` backoff timing | Time-based — would slow test suite without value |
| Windows UTF-8 fix in main.py | OS-specific — no-op on Linux CI |

---

## Running tests

```bash
pip install -r requirements.txt
pytest tests/ -v
```

All 18 tests should pass without any network access or credentials.
