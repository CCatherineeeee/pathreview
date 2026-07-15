# Project Journal

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/155

**Issue title:** Health check references `settings.redis_host`, which does not exist on Settings

**Tier:** [x] Tier 1 [ ] Tier 2 [ ] Tier 3

**Problem summary:**
The `GET /health` endpoint in `api/routes/health.py` tries to build a Redis client from `settings.redis_host` and `settings.redis_port`, but the `Settings` model in `core/config.py` never defines those fields — it only has a `redis_url` string. Because of that, the Redis probe raises an `AttributeError` every time instead of actually checking whether Redis is up, so the health check always reports Redis as unhealthy even when it is running fine. A successful fix makes the health check build its Redis client from the existing `redis_url` setting (via `redis.Redis.from_url`), so the probe connects properly and reports the real Redis status.

**Selection notes — "Is This Issue Right for Me?" checklist:**

_Part 1 — Understanding the issue._ In my own words: the `/health` endpoint's Redis probe reads `settings.redis_host` and `settings.redis_port`, but the `Settings` model in `core/config.py` only defines `redis_url`, so the probe throws `AttributeError` and Redis always shows as unhealthy. The affected area matches the issue's `api` label: `api/routes/health.py` plus `core/config.py`, and I confirmed both files and the exact lines exist. Done looks like: before the fix, `GET /health` returns 503 with Redis marked "unhealthy" even when Redis is up; after the fix, the probe builds its client from `redis_url` and reports the real Redis status.

_Part 2 — Tier fit._ This is my first contribution to this codebase, so I'm choosing Tier 1 as recommended. The change is localized to one function in one file, which fits the Tier 1 definition.

_Part 3 — Codebase readiness._ I read the whole `health_check` handler, not just the broken line: it probes Postgres, Redis, and the vector DB, marks each dependency, and returns 503 if any is unhealthy — so I can predict my change only affects the Redis branch. My rough plan without looking anything up: replace the `redis.Redis(host=..., port=...)` construction with `redis.Redis.from_url(settings.redis_url, decode_responses=True)`. There is no test file for the health route yet (nothing in `tests/unit/` covers it), so I read existing tests in `tests/unit/` to learn the fixture and assertion patterns, and my PR will add the first `/health` tests.

_Part 4 — Scope and time._ The issue has several other claims in the comments; claims are non-exclusive and I'm fine sharing it. As a Tier 1 fix (one function plus a new test file) and a large new repo, I estimate 3–5 hours including testing, which fits comfortably in the Weeks 8–9 window. The issue lists no blockers or dependencies on other issues.

**Branch name:** fix/155-health-check-redis-url

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [ ] Issue added to cohort ledger
