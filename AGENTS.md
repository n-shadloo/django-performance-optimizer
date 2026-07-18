# AGENTS.md — django-performance-optimizer

Always-on project context. This file carries the core guidance in condensed form so an agent that reads only this still behaves correctly. The canonical, most complete instructions live in **`SKILL.md`**; the deep material is in **`references/`**. Read those before doing substantial work.

## What this is

A Django performance skill that writes fast Django/DRF code by default and diagnoses-then-fixes slow existing code. Deep specialty: Django and Django REST Framework. Underneath sits a thin transferable layer (algorithmic complexity, general caching/invalidation) useful to any backend.

**Two modes:**
- **Write-time** — when generating/modifying Django/DRF code, apply performance defaults (eager loading, DB-side aggregation, indexed filters, pagination, `bulk_*`, eager-loading on managers/querysets, read-optimized serializers) — but only where the access pattern warrants it; don't over-prefetch "just in case." Note the choices made.
- **Review-time** — when asked to optimize/profile/"why is this slow": measure first, classify the bottleneck, apply the minimal correct fix, state the expected change.

## Measure first, then fix

Prove the bottleneck with numbers before changing anything: exact query count, the specific N+1 relation, hot-loop complexity, row count, measured time. State the before, apply the fix, state the expected after. Never optimize a non-bottleneck. But once a real problem is proven, **fix it — don't just flag it.** A lazy related-access in a loop is a provable N+1 without a profiler; everything else needs a number.

## Security precedence

Never trade security for performance. All security judgment belongs to the author's **`secure-code-auditor`** skill — rely on it, don't duplicate it. If an optimization would weaken security (drop validation, widen a queryset past a permission filter, cache per-user data under a shared key, disable CSRF, break SQL parameterization), **do not take it. Security wins, always.**

## Library policy — built-in first, library last

Prefer Django/DRF built-ins and the stdlib. Only *suggest* a third-party library when it clearly beats the built-in for a real, measured bottleneck, and only if it's currently well-maintained, up to date, and free of known unresolved CVEs. Verify health at time of use. Every dependency is supply-chain surface. See `references/library-policy.md`.

## Routing map

| Concern | Reference file |
| --- | --- |
| ORM query optimization, N+1, `select_related`/`prefetch_related`, `Prefetch`, `only`/`defer`, `values`, bulk ops, subqueries, `iterator()`, indexing, `explain()` | `references/orm-queries.md` |
| DRF serializers, `SerializerMethodField` cost, nested-serializer N+1, `source` vs method fields, pagination (cursor vs offset), streaming large responses | `references/serializers-drf.md` |
| Caching layers, `cached_property`, backends, invalidation, stampede protection | `references/caching.md` |
| Async views/ORM, ASGI vs WSGI, `sync_to_async` pitfalls, `CONN_MAX_AGE`, native pooling, PgBouncer, connection exhaustion | `references/async-and-connections.md` |
| Offloading work: Django 6 Tasks framework, Celery, offload-vs-optimize | `references/background-work.md` |
| Hot-loop complexity, accidental O(n²), data structures (transferable layer) | `references/algorithms-and-hot-loops.md` |
| Measurement/profiling — query counting, Debug Toolbar, Silk, `nplusone`, `explain()`, py-spy, cProfile | `references/profiling-and-tooling.md` |
| Middleware, templates, static/media, transactions, signals, migrations, settings | `references/settings-and-runtime.md` |
| Third-party library suggestions + health/supply-chain checks | `references/library-policy.md` |

## Version baseline

Django 6.0.7 / 5.2.16 LTS; DRF 3.17.1; Python 3.14 (as of 18 Jul 2026). The 6.0 series is supported through Apr 30 2027; 5.2 LTS through Apr 30 2028. Advice targets 6.0 and flags where 5.2 differs (notably native connection pooling under ASGI).
