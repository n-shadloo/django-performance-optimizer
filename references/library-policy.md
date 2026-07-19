# Library policy

Built-in first, library last. Every dependency is supply-chain and CVE surface — the same reason `secure-code-auditor` warns against unnecessary dependencies.

## The ranking

1. **Django / DRF / stdlib built-in** — always try this first.
2. **A small, focused, well-maintained package** — when the built-in genuinely doesn't cover a measured need.
3. **A heavy dependency** — only with explicit justification tied to a measured bottleneck.

Suggest, never mandate. A library is a proposal backed by a number, not a default.

## Health checklist before suggesting or using ANY library

Tell the agent to verify these **at time of use** (they go stale):

- Compatible with the target **Django (6.0 / 5.2)** and **Python (3.14)**.
- A release within a reasonable recent window (not abandoned).
- Active maintenance (recent commits, responsive issues).
- **No open unresolved security advisories** — check the PyPI page, GitHub Security Advisories / OSV, and run `pip-audit`.
- Reasonable install size and transitive dependency count.
- A real, compatible license.

## Libraries this skill MAY suggest

Suggestions only — verify health each time, state the built-in baseline, the specific bottleneck addressed, and the measured/expected win.

- **Profiling / measurement** — `django-debug-toolbar`, `django-silk`, `nplusone`, `py-spy`, `snakeviz`, `line_profiler`. *Baseline:* `CaptureQueriesContext` / `assertNumQueries` / `QuerySet.explain()`. See `profiling-and-tooling.md`.
- **`django-redis`** — extra Redis features (pattern delete, richer client options) beyond the built-in `RedisCache`. *Baseline:* `django.core.cache.backends.redis.RedisCache` — **prefer the built-in** unless you specifically need django-redis features. See `caching.md`.
- **`celery`** (+ `django-celery-results`, `django-celery-beat`), or **`django-tasks` / RQ / Dramatiq / Huey** — background work beyond the Django 6 Tasks built-in. *Baseline:* `django.tasks`. See `background-work.md`.
- **`psycopg`** (psycopg3) — **required** for native connection pooling and the async ORM path. *Baseline:* n/a (it's the driver). See `async-and-connections.md`.
- **PgBouncer** — centralized connection pooling at scale. Infrastructure, not a Python dependency. See `async-and-connections.md`.
- **`pybreaker`** — circuit breaker around slow external dependencies (latest 1.4.1; actively maintained, ~90k weekly downloads, no known unresolved CVEs; supports Redis-backed shared state). *Baseline:* DB `statement_timeout`/`lock_timeout` + DRF throttling (built-in). See `rate-limiting-and-backpressure.md`.
- **`opentelemetry-instrumentation-django`** — production tracing (latest `0.64b0`, Jun 2026; active, Python 3.10–3.14; still a `0.x`/beta line, normal for OTel contrib). Pair with `opentelemetry-instrumentation-psycopg`. *Baseline:* Django logging + `pg_stat_statements` + slow-query log. See `profiling-and-tooling.md`.
- **Load testing** — **Locust** (Python) or **k6** (JS/TS binary; k6 2.0 shipped May 2026); both actively maintained. *Baseline:* `assertNumQueries` as the built-in CI guardrail. See `profiling-and-tooling.md`.
- **Dedicated search engines** (boundary only, past PostgreSQL FTS/trigram): the *servers* — Elasticsearch, OpenSearch, Meilisearch (pin ≥ v1.48.2 for the 2026 auth CVEs), Typesense — are healthy, but the *Django integration packages* are the weak link. `django-elasticsearch-dsl` and `django-typesense` are stale/inactive (prefer the official Python client directly); `django-opensearch-dsl` is the healthiest Django-native option but has not confirmed Django 6.x / Python 3.14. *Baseline:* `django.contrib.postgres` trigram + full-text search (built-in). See `indexing-and-search.md`.
- **`django-postgres-extra`** — helper for declarative table partitioning when raw-SQL migrations are too raw. *Baseline:* `migrations.RunSQL` + a `managed = False` model (built-in). See `data-at-scale.md`.

## Never force any of these

For each suggestion, always state: the built-in alternative, the specific bottleneck the library addresses, and the measured or expected win. Defer the final "is this dependency safe to add" judgment to `secure-code-auditor`.
