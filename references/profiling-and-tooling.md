# Profiling & tooling

The measure-first entry point. Establish a baseline **before** changing anything, and re-measure after. Each tool below includes how to read it.

## Query counting

The cheapest, most decisive Django measurement.

```python
from django.test.utils import CaptureQueriesContext
from django.db import connection

with CaptureQueriesContext(connection) as ctx:
    response = client.get("/api/books/")
print(len(ctx))                 # query count
for q in ctx.captured_queries:  # each has "sql" and "time"
    print(q["time"], q["sql"])
```

In tests, assert it so regressions fail CI:

```python
with self.assertNumQueries(3):
    self.client.get("/api/books/")
```

`len(connection.queries)` also works with `DEBUG=True`. **Read it by:** running the same endpoint against 10 rows and 200 rows — if the count scales with row count, it's an N+1.

## `django-debug-toolbar`

SQL, cache, template, and signal panels in the browser. The SQL panel shows query **count**, **duplicates**, and **time** per query — duplicates are the N+1 tell. Dev/HTML only: it won't render over a JSON response, so inspect via an HTML page or the admin, or use query counting for APIs.

## `django-silk` (Jazzband)

Request/SQL profiling with a query explorer and cProfile integration (`@silk_profile()`), plus `.prof` download for snakeviz. Works over JSON APIs (unlike Debug Toolbar). Dev/staging only — it adds overhead; **sample a fraction** of requests under realistic load rather than profiling everything.

## `nplusone`

Flags potential N+1s in development by detecting lazy related-access that wasn't prefetched, raising/logging when it happens. Good as a CI/dev guard to catch N+1s the moment they're introduced.

## `QuerySet.explain()` / EXPLAIN ANALYZE

```python
Book.objects.filter(author_id=5).explain(analyze=True)   # PostgreSQL
```

Read the plan for **seq scan vs index scan**, **estimated vs actual rows**, and sort/hash costs. A seq scan on a large filtered table → add an index. A big estimate/actual gap → stale statistics or a bad index choice. This is how you decide indexes rather than guessing.

## `py-spy`

Sampling profiler that **attaches to a live process** (including production) without restarting it. Produces flame graphs of CPU hot paths. Use when the bottleneck is Python compute, not queries, and especially when you can only observe the problem under production load.

## `cProfile` / `line_profiler` / `snakeviz`

Deterministic function-level (`cProfile`) and line-level (`line_profiler`) profiling for **one view in isolation** — mock external calls so you measure your code, not the network. Visualize the `.prof` with `snakeviz`. Use to find the expensive function/line once query count is already flat.

## APM (optional suggestion)

Sentry / Scout / etc. for continuous production monitoring of latency and slow transactions. Suggestion only — verify health, cost, and data-handling before adopting; defer the security/privacy judgment to `secure-code-auditor`.

## Production observability & SLOs

Measure-first at production scale — tie every optimization to a real production number.

- **`pg_stat_statements`** — the highest-value production diagnostic. Enable via `shared_preload_libraries`, then read it sorted by `total_exec_time` / `mean_exec_time` / `calls` to rank the queries actually costing time. `pg_stat_statements_reset()` before a measurement window. This is the production analog of query counting (see `indexing-and-search.md`).
- **Slow-query log** — set `log_min_duration_statement` (e.g. `250ms`) to capture slow statements with parameters.
- **Tracing (OpenTelemetry)** — `opentelemetry-instrumentation-django` auto-instruments the request cycle, ORM, templates, and cache into spans; pair with `opentelemetry-instrumentation-psycopg` for query spans. Suggestion only — verify health (see `library-policy.md`).
- **SLOs on percentiles, not averages** — track **p50/p95/p99** latency and error rate; a good p50 hides a bad tail. Set a target and alert on the percentile.

## Load & regression testing

- **`assertNumQueries` as a CI guardrail** — the cheapest, most Django-native regression gate. Wrap hot endpoints in `with self.assertNumQueries(N):` so an accidental N+1 **fails the build**. Make it standard on list/detail endpoints.
- **Load tools** — reproduce production load before you ship: **Locust** (behavior scripted in Python, distributed master/worker) or **k6** (JS/TS scripts, single binary, CLI thresholds like `p(95)<500` that fail CI). Suggestions — see `library-policy.md`.
- **Regression benchmarking** — compare p95 and query count against a stored baseline in CI so performance regressions surface as failures, not incidents.

## Process

1. Profile in **production-like conditions** with **realistic data volume** — a bug invisible on 10 rows dominates at 200.
2. **Isolate** the thing under test (one endpoint, mock externals).
3. Fix the **biggest** item first.
4. **Re-measure** to confirm the win and that nothing regressed.
5. Iterate.
