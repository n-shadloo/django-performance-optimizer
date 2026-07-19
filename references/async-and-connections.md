# Async & connections

Django 6.0 async accurate; 5.2 LTS differences flagged. Async is for I/O concurrency, not raw speed — reach for it deliberately.

## Async views & ASGI vs WSGI

Async views require an ASGI server (Uvicorn, Daphne, or Gunicorn with Uvicorn workers). Async helps **I/O-bound concurrency** — many concurrent external API calls, streaming responses, websockets. It does **not** speed CPU-bound work, and it can *hurt* if you wrap sync ORM calls in threads unnecessarily. If a view is a straightforward sync DB read, keep it sync under WSGI.

## Async ORM (Django 6)

Use the query-executing `a`-prefixed methods, and `async for` to iterate:

```python
book = await Book.objects.aget(pk=pk)
await Book.objects.acreate(title="…")
await Book.objects.filter(active=False).aupdate(archived=True)
await Book.objects.filter(pk=pk).adelete()
exists = await Book.objects.filter(author=a).aexists()
n = await Book.objects.acount()
first = await Book.objects.afirst()
obj, created = await Book.objects.aget_or_create(isbn="…")

async for book in Book.objects.filter(active=True):
    ...
```

Lazy builders (`filter`, `exclude`, `values`, `annotate`, …) have **no** async variant — they build the queryset without touching the DB, so they stay sync. Calling a sync query-executing method (`.get()`, `.count()`, iterating without `async for`) inside an async view raises `SynchronousOnlyOperation`. Prefer the async ORM methods over wrapping sync calls in `sync_to_async`.

## `sync_to_async` / `async_to_sync` pitfalls

- Don't `await sync_to_async(Model.objects.all)()` and then iterate — the queryset is lazy, so iteration happens back in async context and raises. Use `async for x in qs` or `[x async for x in qs]`.
- Keep `thread_sensitive=True` (the default) when wrapping ORM work, so it runs in the correct thread.
- **Thread-pool exhaustion.** Each `sync_to_async`-wrapped call runs on a bounded thread pool; wrapping *many* sync/ORM calls (e.g. a per-row loop) saturates it and requests queue behind it — throughput collapses. Batch ORM work into fewer calls and prefer the native async ORM methods over per-item `sync_to_async`.
- **Transactions don't work in async mode.** Write transactional logic as one **sync** function wrapped in `transaction.atomic()` and call it via `sync_to_async(fn)()` — don't try to `async with transaction.atomic()`.

## Concurrent I/O with `asyncio.gather`

Where async genuinely wins is fanning out independent I/O. Await calls together, not in sequence:

```python
import asyncio
a, b, c = await asyncio.gather(fetch_a(), fetch_b(), fetch_c())  # total ≈ slowest call, not the sum
```

This is the payoff case: many concurrent external calls, streaming, websockets. A plain CRUD view doing two ORM queries gets nothing from async and keeps the sync/async-boundary risk — leave it sync under WSGI.

## `CONN_MAX_AGE` & persistent connections

`CONN_MAX_AGE` controls connection reuse:

- `0` (default) — close the DB connection after each request.
- `>0` — keep it open for N seconds across requests.
- `None` — keep it open indefinitely.

Under WSGI, `CONN_MAX_AGE > 0` cuts per-request connect overhead. **Disable persistent connections (`CONN_MAX_AGE = 0`) in async mode** — async workers multiplex, so per-worker persistent connections don't map cleanly; use the backend's pool instead.

## Native connection pooling (verify current state at time of use)

Django **5.1+** added native PostgreSQL connection pooling via **psycopg3**:

```python
DATABASES = {"default": {
    "ENGINE": "django.db.backends.postgresql",
    "OPTIONS": {"pool": True},               # or {"pool": {"max_size": 20, ...}}
    "CONN_MAX_AGE": 0,                        # REQUIRED
}}
```

Requires the `psycopg` package (**psycopg3**, not psycopg2). It requires `CONN_MAX_AGE = 0` or Django raises `ImproperlyConfigured: Pooling doesn't support persistent connections`. Tune with `"pool": {"max_size": N, "min_size": M, "timeout": T}`.

**Version-sensitive:** on Django **5.1 / 5.2** the docs advise **against** native pooling under ASGI — use PgBouncer there instead. **Django 6.0** ships an async-aware `AsyncConnectionPool` that works under ASGI, so prefer native pooling on 6.0. Confirm the current recommendation for your exact version at time of use.

## Connection exhaustion under multiple Gunicorn workers

Total connections ≈ `workers × (pool max_size or 1)`. Keep that sum comfortably under PostgreSQL's `max_connections`. Example: 8 workers × pool of 20 = 160 connections from one app instance — size pools deliberately and monitor active connections.

## PgBouncer / external pooling

For large fleets, **PgBouncer** centralizes connections across many app instances. Pick the mode by what it breaks:

- **Session** — one server connection per client session; safe for everything (prepared statements, `LISTEN/NOTIFY`, session advisory locks, `SET`, temp tables) but least multiplexing.
- **Transaction** (the productive default) — server bound only per transaction. Breaks session-scoped features: session-level `SET`/`search_path` (use `SET LOCAL`), `LISTEN/NOTIFY`, **session-level advisory locks** (acquired on one backend, released on another → they leak), session temp tables, and `WITH HOLD` cursors. Protocol-level **prepared statements** were broken here until **PgBouncer 1.21** added `max_prepared_statements` (default `0`/off). Most of these fail silently, not loudly.
- **Statement** — server released after every statement; multi-statement transactions are forbidden. Too aggressive for normal apps.

Django's server-side cursors (`.iterator()`) break under transaction pooling unless you set `DISABLE_SERVER_SIDE_CURSORS = True`. Rule of thumb: native pool for most apps; PgBouncer at scale. (Celery 5.1+ auto-closes the Django connection pool in workers.)

## Security note

Connection and pool changes must not disable TLS to the database or move credentials into insecure locations. Defer secrets handling and transport-security judgment to `secure-code-auditor`.
