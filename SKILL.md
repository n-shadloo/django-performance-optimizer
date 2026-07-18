---
name: django-performance-optimizer
description: Write and audit maximally performant Django and Django REST Framework code. Use whenever Django/DRF code is being written or reviewed and speed, query count, latency, memory, or scalability is in scope — including any time the work touches the ORM or querysets, select_related/prefetch_related, N+1 access, serializers or API endpoints, pagination or large result sets, caching, async/ASGI, database connections or pooling, background or offloaded work, indexing, bulk operations, hot loops, or profiling, even if the word "performance" is never used. Runs in two modes: review-time (measure existing code, prove the bottleneck with query counts and timings, then apply the correct fix) and write-time (apply performance defaults while generating code so it is fast the first time). Django/DRF is the primary target; the algorithmic and caching principles transfer to any backend. Defers all security judgment to secure-code-auditor and never trades security for speed.
license: MIT
allowed-tools: Read, Grep, Glob, Edit, Write
metadata:
  author: n-shadloo
  version: 1.0.0
---

# django-performance-optimizer

A Django performance skill. It writes fast Django/DRF code by default and diagnoses-then-fixes slow existing code. The deep specialty is Django and Django REST Framework; underneath sits a thin, genuinely transferable layer (algorithmic complexity, general caching and invalidation principles) useful to any backend. Its canonical content is reused by other agents (Codex, Cursor, Gemini CLI) via `AGENTS.md`, with Claude as the primary integration.

Version baseline is kept current (Django 6.0.7 / 5.2.16 LTS; DRF 3.17.1; Python 3.14, as of 18 Jul 2026). The Django 6.0 series receives security and data-loss fixes through April 30, 2027; the 5.2 LTS series through April 30, 2028. Advice targets Django 6.0 and flags where 5.2 LTS differs (notably native connection pooling under ASGI).

## Governing philosophy — measure first, then fix

Diagnose before changing anything. Every performance claim must be grounded in evidence: the exact query count, the specific N+1 relation, the complexity of the hot loop, the row count, the measured time. State the before number, apply the fix, state the expected after. Never optimize a non-bottleneck — premature optimization of code that isn't hot is itself a bug. But once a real problem is proven, **fix it** — do not stop at flagging.

## Choose the right design, don't run a checklist

When generating or reviewing, decide the best architecture for the code's current state and access pattern: a read-optimized serializer vs an annotation; cursor vs offset pagination; in-request work vs an offloaded task; `select_related` (JOIN) vs `prefetch_related` (second query). Pick the pattern that fits the data shape, not the first item on a list.

## Security precedence — defer to `secure-code-auditor`

Never sacrifice security for performance. All security concerns belong to the author's `secure-code-auditor` skill; rely on it rather than duplicating its checks (avoid drift). If an optimization would weaken security — dropping validation, widening a queryset to skip a permission filter, caching per-user data under a shared key, disabling CSRF, raw SQL that breaks parameterization — do **not** take it. **Security wins, always.**

## Library policy — built-in first, library last (suggest, never mandate)

Prefer Django/DRF built-ins and the standard library. Only *suggest* a third-party library when it clearly beats the built-in for a real, measured bottleneck, and only if it is currently well-maintained, up to date, and free of known unresolved CVEs. When suggesting, tell the agent to verify the library is still healthy at time of use. Every dependency is supply-chain surface. See `references/library-policy.md`.

## How the reference material is organized

Progressive disclosure: load only the file relevant to the concern in front of you. `SKILL.md` is the spine and router; the deep material lives in `references/`.

| Concern | Reference file |
| --- | --- |
| ORM query optimization, N+1, `select_related`/`prefetch_related`, `Prefetch`, `only`/`defer`, `values`, bulk ops, subqueries, `iterator()`, indexing, `explain()` | `references/orm-queries.md` |
| DRF serializers, `SerializerMethodField` cost, nested-serializer N+1, `source` vs method fields, pagination (cursor vs offset), streaming large responses | `references/serializers-drf.md` |
| Per-view/per-site/template-fragment/low-level caching, `cached_property`, backends, invalidation, stampede protection | `references/caching.md` |
| Async views/ORM, ASGI vs WSGI, `sync_to_async` pitfalls, `CONN_MAX_AGE`, native connection pooling, PgBouncer, connection exhaustion | `references/async-and-connections.md` |
| Offloading expensive work: Django 6 Tasks framework, Celery, offload-vs-optimize | `references/background-work.md` |
| Hot-loop complexity, accidental O(n²), data structures (the transferable layer) | `references/algorithms-and-hot-loops.md` |
| Measurement/profiling — query counting, Debug Toolbar, Silk, `nplusone`, `explain()`/EXPLAIN ANALYZE, py-spy, cProfile (the measure-first entry point) | `references/profiling-and-tooling.md` |
| Middleware, template rendering, static/media, transactions/atomicity, signals, migrations, performance-relevant settings | `references/settings-and-runtime.md` |
| Third-party library suggestions + health/supply-chain checks | `references/library-policy.md` |

## Mode selection

- **Review-time.** Trigger when the user asks to optimize/speed up/profile/"why is this slow", pastes code and asks why it's slow, or just finished a feature and wants it faster. Behavior: (1) reproduce and **measure** first — count queries (`CaptureQueriesContext`, `assertNumQueries`, Debug Toolbar, `django-silk`), time the hot path, get row counts, run `QuerySet.explain()` (EXPLAIN ANALYZE on PostgreSQL) on suspect queries; (2) classify the bottleneck (query count / query cost / serialization / Python-side compute / I/O / lock contention); (3) apply the minimal correct fix for that class and state the expected query-count/latency change; (4) check the fix against `secure-code-auditor` concerns and discard it if it weakens security. Do not pattern-match a keyword into an "optimization" — prove it's a bottleneck before touching it.
- **Write-time.** Trigger when generating or modifying Django/DRF code. Behavior: default to eager loading, DB-side aggregation, indexed filters, pagination, `bulk_*` for batch writes, eager-loading encoded on managers/querysets, and read-optimized serializers — but only where the access pattern warrants it; do not over-prefetch "just in case." Briefly note the performance-relevant choices made.
- **If ambiguous,** apply write-time defaults while coding and offer to run a measured review afterward.

## The measure-first loop

This is how to report — the analog to a severity table. For each fix, show four things:

1. **Baseline** — e.g. "847 queries / 2.3 s on 200 rows".
2. **Diagnosis** — the exact cause (the specific N+1 relation, the missing index, the O(n²) loop).
3. **Fix** — the minimal correct change.
4. **Expected result** — e.g. "→ 2 queries".

Only report or optimize things you've measured or can prove by inspection. A lazy related-access inside a loop is a provable N+1 without a profiler; a missing index on a filtered column is provable by reading the model. Everything else needs a number.
