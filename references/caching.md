# Caching

Django 6.0 cache-framework accurate. Cache to avoid recomputing or re-querying — but caching is the second-hardest problem in computing, so cache deliberately and always solve invalidation.

## Layers, cheapest gain first

1. **Per-site cache** — `UpdateCacheMiddleware` (top) + `FetchFromCacheMiddleware` (bottom) cache every anonymous GET/HEAD page. Coarse; only for read-mostly, mostly-anonymous sites.
2. **Per-view cache** — `@cache_page(timeout)`. For CBVs / DRF views:

   ```python
   from django.utils.decorators import method_decorator
   from django.views.decorators.cache import cache_page

   @method_decorator(cache_page(60 * 5), name="dispatch")
   class ProductList(ListView): ...
   ```
3. **Template fragment cache** — `{% load cache %}{% cache 300 sidebar request.user.id %}...{% endcache %}` caches an expensive fragment with a key varied by the arguments.
4. **Low-level cache API** — full control; cache exactly the expensive object.

## Low-level API

```python
from django.core.cache import cache

cache.set("key", value, timeout=300)
cache.get("key", default=None)
cache.add("key", value)                       # only if absent
cache.get_or_set("key", expensive_callable, 300)   # race-safe compute-on-miss
cache.get_many(["a", "b"]); cache.set_many({"a": 1, "b": 2})
cache.delete("key"); cache.delete_many(["a", "b"])
cache.incr("counter"); cache.decr("counter")
```

Use `get_or_set` to avoid the check-then-set race; `get_many`/`set_many` to batch round-trips. Versions (`version=`, `cache.incr_version()`) and `KEY_PREFIX` segregate keyspaces. Any picklable object can be cached; default `TIMEOUT` is **300 s**.

## `cached_property`

Memoize an expensive per-instance computed attribute for the lifetime of the object/request:

```python
from django.utils.functional import cached_property

class Cart(models.Model):
    @cached_property
    def total(self):
        return self.items.aggregate(t=Sum("price"))["t"]   # computed once per instance
```

## Backends

Prefer built-in backends. Configure via `CACHES`:

- **Redis** — `django.core.cache.backends.redis.RedisCache` (uses redis-py; add `hiredis` for faster parsing).
- **Memcached** — `django.core.cache.backends.memcached.PyMemcacheCache`.
- **Local-memory** (`LocMemCache`) — dev only, per-process.
- **Database**, **file** — niche.

Redis vs Memcached: comparable raw speed; Redis has richer features (persistence, sessions, pub/sub, data structures), so it's usually the better single choice unless you're already on Memcached. Tune `OPTIONS`, `TIMEOUT`, `CULL_FREQUENCY`, `MAX_ENTRIES`.

## Invalidation strategy — the hard part

- **Time-based TTL** for public, read-mostly data where slight staleness is acceptable.
- **Explicit `cache.delete()`** on write — invalidate affected keys in the create/update/delete path.
- **Version bumping** (`incr_version` / a version token embedded in the key) to invalidate a whole group at once without enumerating keys.
- **Key-prefix segregation** to scope invalidation.

For DRF write endpoints, invalidate the affected cached keys in `perform_create`/`perform_update`/`perform_destroy`.

## QuerySet result-caching realities

A QuerySet caches results only within a single evaluated instance (see `orm-queries.md`). Django does **not** cache queries across requests — re-creating the queryset re-hits the DB. Cross-request caching is exactly what the cache framework is for; don't expect the ORM to do it.

## Stampede protection

When a hot key expires under load, every concurrent request misses and recomputes at once (a dogpile / cache stampede). Mitigate with a short lock around the recompute, or probabilistic early expiration (recompute slightly before TTL for a fraction of requests). Track hit rate, evictions, and latency to know whether the cache is actually helping.

## Security note (defer to `secure-code-auditor`)

- **Never** cache per-user or authenticated content under a shared key — a classic data-leak. Mind `Vary` headers and cookies.
- Note the real fixed Django cache CVEs, both fixed in **Django 6.0.5 and 5.2.14 (5 May 2026)** — target patched releases:
  - **CVE-2026-35192** (LOW) — session fixation via public cached pages when `SESSION_SAVE_EVERY_REQUEST=True`.
  - **CVE-2026-6907** — private-data exposure via incorrect `Vary: *` handling in `UpdateCacheMiddleware`.
- Defer the final security judgment on any cache key/scope decision to `secure-code-auditor`.
