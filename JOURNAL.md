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

---

## Week 8 — Reproducing issue #155

**Status:** Reproduced and confirmed. The bug triggers on every request, not intermittently.

### Environment setup

Starting from a clean checkout (no `.env`, no `.venv`, Docker stopped):

```bash
cp .env.example .env
python3 -m venv .venv && .venv/bin/pip install -e ".[dev]"
docker compose up -d
.venv/bin/alembic upgrade head        # applies migrations 001 and 002
.venv/bin/uvicorn api.main:app --host 127.0.0.1 --port 8000
```

### Reproduction steps

```bash
curl -i http://127.0.0.1:8000/health
```

Observed — HTTP `503`, Redis reported as down:

```json
{"detail":{"status":"unhealthy",
 "dependencies":{"postgres":"unhealthy","redis":"unhealthy","vector_db":"healthy"},
 "safety_events_last_hour":0,
 "timestamp":"2026-07-22T01:22:06.226408"}}
```

Repeated 3 times, identical response each time.

### Proving it is a false alarm

Redis itself is healthy the whole time:

```bash
docker compose exec redis redis-cli ping
# PONG
```

So the endpoint claims Redis is down while Redis is answering. That contrast is the
core of the reproduction.

### Root cause confirmation

The HTTP response body gives no useful detail — it only says `"unhealthy"`. The real
cause appears only in the uvicorn server log:

```
redis_health_check_failed  error="'Settings' object has no attribute 'redis_host'"
```

Confirmed directly against the settings object:

```bash
.venv/bin/python -c "
from core.config import settings
print('has redis_host?', hasattr(settings, 'redis_host'))   # False
print('redis_url =', settings.redis_url)                    # redis://localhost:6379/0
"
```

`api/routes/health.py` (lines 44–49) builds its client from `settings.redis_host` and
`settings.redis_port`. `core/config.py` defines neither — it exposes only `redis_url`
(line 12). The attribute lookup raises `AttributeError` before any connection is
attempted, and the surrounding `except Exception` catches it and labels the dependency
"unhealthy".

**Takeaway for the PR:** the broad `except Exception` makes a coding error
indistinguishable from a real outage. A missing attribute and a dead Redis produce the
exact same output.

### Fix direction, validated

```bash
.venv/bin/python -c "
import redis
from core.config import settings
r = redis.Redis.from_url(settings.redis_url, decode_responses=True)
print('ping ->', r.ping())   # True
"
```

Confirms `redis.Redis.from_url(settings.redis_url, ...)` connects against the running
container, so the planned one-line fix is sound.

### Out of scope — found while reproducing

Two other things surfaced. Neither is part of #155 and neither will be changed in this PR.

1. **Postgres check is also broken.** `health.py` line 31 calls
   `await db.execute("SELECT 1")` with a raw string, which SQLAlchemy 2.x rejects:
   `Textual SQL expression 'SELECT 1' should be explicitly declared as text('SELECT 1')`.
   This matters for acceptance criteria: after fixing Redis, the `redis` key flips to
   `"healthy"`, but the endpoint still returns 503 because Postgres stays unhealthy.
   Worth filing separately.

2. **Vector DB check is vacuous.** It only tests whether the `vector_db_url` string is
   non-empty, so it reports `"healthy"` even when the Chroma container is stopped —
   which it was during part of this session.

### Next step

Add the first `/health` test file under `tests/unit/` (no test currently references the
health route), then apply the `from_url` fix.
