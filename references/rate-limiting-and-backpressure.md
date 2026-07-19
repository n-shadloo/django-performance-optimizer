# Rate limiting & backpressure

Django 6.0 / DRF 3.17.x accurate. Protecting the service under load is the flip side of measure-first: the database you tuned still falls over if a thundering herd or one slow dependency saturates it. These are Django-anchored defenses; each has a cost, so apply them where load or a slow dependency actually threatens the system, not preemptively everywhere.

## Rate limiting — DRF throttling (built-in)

`AnonRateThrottle`, `UserRateThrottle`, `ScopedRateThrottle`, or a `SimpleRateThrottle` subclass, configured via `DEFAULT_THROTTLE_CLASSES` / `DEFAULT_THROTTLE_RATES`:

```python
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES": ["rest_framework.throttling.UserRateThrottle"],
    "DEFAULT_THROTTLE_RATES": {"user": "1000/day", "anon": "100/day"},
}
```

Throttle state lives in the Django cache backend — point it at Redis/Memcached, not `LocMemCache`, or each process/host counts separately and the effective limit multiplies by worker count (see `caching.md`). Use `ScopedRateThrottle` to put tighter limits on expensive endpoints. This built-in covers app-level rate limiting; no third-party limiter is needed.

## Load shedding via timeouts

Kill runaway work instead of letting it pile up and exhaust connections/workers. Set PostgreSQL `statement_timeout` and `lock_timeout` per connection:

```python
DATABASES["default"]["OPTIONS"]["options"] = "-c statement_timeout=3000 -c lock_timeout=1000"
```

Set request timeouts at the server layer too: Gunicorn `--timeout`, and the ASGI server's request / keep-alive timeouts. A bounded slow request that gets shed keeps the pool healthy for everyone else.

## Backpressure & circuit breakers

When an external dependency (payment gateway, third-party API) slows or fails, in-request calls to it pile up and consume your workers — the failure cascades into your app. A **circuit breaker** trips after repeated failures and fails fast for a cooldown, shedding load on the sick dependency instead of dragging your service down with it. The pattern needs no library; `pybreaker` is a healthy suggest-only option if you want one (see `library-policy.md`). Keep external calls out of DB transactions (they hold locks — see `concurrency-and-locking.md`) and prefer offloading non-critical ones (see `background-work.md`).

## Graceful degradation

On a tripped breaker or a shed request, degrade instead of returning 500: serve slightly stale cache, a reduced payload, or a "queued — we'll finish this shortly" acknowledgment. Decide the degraded response deliberately per endpoint.

## Measure

Track throttle 429 rates, statement/lock-timeout kill counts, breaker open/close events, and p95/p99 under load (see `profiling-and-tooling.md`). Tune the limits to real measured capacity, not guesses.

## Security note

Rate limiting is an availability control, not a substitute for authentication or authorization. Defer auth and abuse-prevention judgment to `secure-code-auditor`.
