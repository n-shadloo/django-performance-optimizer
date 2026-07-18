# Settings & runtime

Everything else that moves the needle — Django-specific runtime and configuration.

## Middleware cost

Every middleware runs on **every request**, and order matters. Remove middleware you don't use; keep expensive work out of the middleware chain (it can't be skipped per-view cheaply). `GZipMiddleware` reduces response size for text payloads — but note the compression-vs-secrets concern (BREACH-class attacks on responses that mix secrets with attacker-influenced content); defer that judgment to `secure-code-auditor`.

## Template rendering

- Use the **cached template loader** (`django.template.loaders.cached.Loader`) in production so templates are compiled once, not per render. Django enables it automatically when `APP_DIRS` is set and `DEBUG=False`; if you set `loaders` explicitly, wrap them in `cached.Loader`.
- Cache expensive fragments with `{% cache %}` (see `caching.md`).
- Avoid heavy logic and **per-item DB access in templates** — an unprefetched relation accessed in a template loop is an N+1 that no view-level query count will reveal until you look. Prefetch in the view.

## Static / media serving

- **Never** serve static or media files through Django in production — use the web server (nginx) or a CDN.
- `ManifestStaticFilesStorage` for hashed filenames → safe far-future cache headers and cache-busting.
- Offload user media to object storage (S3/GCS) rather than the app server's disk.
- Set far-future `Cache-Control` on hashed assets.

## Transactions / atomicity cost

- `ATOMIC_REQUESTS = True` wraps **every** request in a transaction — simple, but holds a connection and locks for the whole request, which hurts throughput under load.
- Prefer **targeted** `transaction.atomic()` around only the writes that need it.
- Django 6.0 allows `@atomic` (as a decorator) on class-based views.
- Keep transactions **short**: do slow or external work (HTTP calls, emails) **outside** the transaction, and enqueue background tasks via `transaction.on_commit(...)` so they don't fire before commit.

## Signals overhead

Synchronous signal receivers run **inline** within the triggering operation. A `post_save` receiver that queries per save creates a hidden N+1 on bulk writes and slows every write. Prefer explicit function calls over signals for performance-critical paths, or move heavy receiver work to a background task. (Remember `bulk_create`/`bulk_update` don't fire `save` signals at all.)

## Migrations performance (where relevant)

- Batch large data migrations with `bulk_update` and `iterator()` (see `orm-queries.md`) instead of per-row `.save()`.
- On PostgreSQL, add indexes with `AddIndexConcurrently` in a **non-atomic** migration (`atomic = False`) so you don't lock the table while building the index.
- Avoid table rewrites and long locks on hot tables at peak traffic — schedule them.

## Performance-relevant settings

- `CONN_MAX_AGE` / connection pool — see `async-and-connections.md`.
- `TEMPLATES` cached loader (above).
- `CACHES` — see `caching.md`.
- `DATABASES` `OPTIONS` — pool, statement timeout, SSL.
- **`DEBUG = False` in production, always.** `DEBUG=True` stores every SQL query in memory for the life of the process (an unbounded memory leak) and is also a security disclosure risk — defer the security angle to `secure-code-auditor`, but the performance reason alone is disqualifying.
- `DATA_UPLOAD_MAX_MEMORY_SIZE` / `DATA_UPLOAD_MAX_NUMBER_FIELDS` — bound request parsing cost.
- Session backend — cache or cache-backed-db rather than pure db for hot session access.
- `USE_TZ` — keep timezone handling in the DB where possible.
