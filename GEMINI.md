# GEMINI.md — django-performance-optimizer

Gemini CLI doesn't read Agent Skills, so this file carries the same conventions and points you at the canonical material rather than duplicating it. **`SKILL.md` is canonical** and most complete; the deep material is in `references/`; `AGENTS.md` is the always-on context layer. Read those before substantial work.

## What this is

A Django performance skill: write fast Django/DRF code by default, and diagnose-then-fix slow existing code. Deep specialty is Django and Django REST Framework, over a thin transferable layer (algorithmic complexity, general caching/invalidation) useful to any backend.

**Two modes:**
- **Write-time** — generating/modifying Django/DRF code: apply performance defaults (eager loading, DB-side aggregation, indexed filters, pagination, `bulk_*`, eager-loading on managers/querysets, read-optimized serializers) only where the access pattern warrants it. Note the choices made.
- **Review-time** — optimize/profile/"why is this slow": measure first, classify the bottleneck, apply the minimal correct fix, state the expected change.

## Measure first, then fix

Prove the bottleneck with numbers before changing anything — exact query count, the specific N+1 relation, hot-loop complexity, row count, measured time. State the before, apply the fix, state the expected after. Never optimize a non-bottleneck; once a real one is proven, **fix it, don't just flag it.**

## Security precedence

Never trade security for performance. All security judgment belongs to the author's **`secure-code-auditor`** skill — rely on it, don't duplicate it. If an optimization would weaken security (drop validation, widen a queryset past a permission filter, cache per-user data under a shared key, break SQL parameterization), don't take it. **Security wins, always.**

## Library policy — built-in first, library last

Prefer Django/DRF built-ins and the stdlib. Only *suggest* a third-party library when it clearly beats the built-in for a real, measured bottleneck, and only if currently well-maintained, current, and free of known unresolved CVEs. Verify health at time of use. See `references/library-policy.md`.

## Routing map

| Concern | Reference file |
| --- | --- |
| ORM queries, N+1, `select_related`/`prefetch_related`, `Prefetch`, `only`/`defer`, `values`, bulk ops, subqueries, `iterator()`, indexing, `explain()` | `references/orm-queries.md` |
| DRF serializers, `SerializerMethodField` cost, nested-serializer N+1, `source` vs method fields, pagination, streaming | `references/serializers-drf.md` |
| Caching layers, `cached_property`, backends, invalidation, stampede protection | `references/caching.md` |
| Async views/ORM, ASGI vs WSGI, `sync_to_async`, `CONN_MAX_AGE`, native pooling, PgBouncer | `references/async-and-connections.md` |
| Background work: Django 6 Tasks, Celery, offload-vs-optimize | `references/background-work.md` |
| Hot-loop complexity, accidental O(n²), data structures | `references/algorithms-and-hot-loops.md` |
| Profiling — query counting, Debug Toolbar, Silk, `nplusone`, `explain()`, py-spy, cProfile | `references/profiling-and-tooling.md` |
| Middleware, templates, static/media, transactions, signals, migrations, settings | `references/settings-and-runtime.md` |
| Indexing depth, full-text & trigram search, `JSONField`, `pg_stat_statements` | `references/indexing-and-search.md` |
| Locking & concurrency — `select_for_update`, `skip_locked`, optimistic/pessimistic, isolation levels | `references/concurrency-and-locking.md` |
| Data at scale — replica routing, partitioning, materialized views, boundary hand-offs | `references/data-at-scale.md` |
| Rate limiting, backpressure, load shedding, circuit breakers | `references/rate-limiting-and-backpressure.md` |
| Library suggestions + health/supply-chain checks | `references/library-policy.md` |

## Version baseline

Django 6.0.7 / 5.2.16 LTS; DRF 3.17.1; Python 3.14; Celery 5.6.3 (as of 19 Jul 2026). The 6.0 series is supported through Apr 30 2027; 5.2 LTS through Apr 30 2028. Advice targets 6.0 and flags where 5.2 differs (notably native connection pooling under ASGI). Django 6.1 is in beta (6.1b1, expected Aug 2026); its perf-relevant features (ORM fetch modes, DB-level `on_delete`) are flagged where noted as not yet shipped.
