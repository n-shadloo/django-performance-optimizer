# Concurrency & locking

Django 6.0 / 5.2 LTS + PostgreSQL accurate. Concurrency control is a correctness problem first and a performance problem second: the wrong lock either corrupts data or serializes your whole workload. Reach for a lock only where two transactions genuinely race on the same row, and measure lock waits / deadlocks before and after (see `profiling-and-tooling.md`).

## `select_for_update` — pessimistic row locks

Locks the selected rows until the end of the transaction. Must run inside `transaction.atomic()` or it raises `TransactionManagementError`.

```python
from django.db import transaction
from django.db.models import F

with transaction.atomic():
    product = Product.objects.select_for_update().get(pk=pk)
    if product.stock >= qty:               # read a consistent value, then branch
        product.stock -= qty
        product.save(update_fields=["stock"])
```

`select_for_update(nowait=False, skip_locked=False, of=(), no_key=False)`:

- **`nowait=True`** — raise `DatabaseError` immediately instead of blocking on a locked row.
- **`skip_locked=True`** — skip rows already locked by others (the queue pattern below). Mutually exclusive with `nowait`.
- **`of=("self", ...)`** — lock only the named tables in a multi-table join, so a `select_related` doesn't lock joined rows you won't modify.
- **`no_key=True`** — emit `FOR NO KEY UPDATE`, a weaker lock that still allows concurrent inserts referencing the row via a foreign key. Use when you aren't changing the primary key; it reduces contention.

Backends: PostgreSQL and Oracle support all options; MySQL 8+/MariaDB support `nowait`/`skip_locked` only (`of`/`no_key` raise `NotSupportedError`); SQLite ignores them. Prefer an atomic `F()` update (`update(stock=F("stock") - qty)`, see `orm-queries.md`) when you only need race-free arithmetic and don't need to read-then-branch — it holds no lock. Reach for `select_for_update` when you must read a value and make a decision on it inside the transaction.

## The `skip_locked` worker-queue pattern

`FOR UPDATE SKIP LOCKED` turns a table into a concurrent work queue: each worker claims a disjoint batch with no lock waits.

```python
with transaction.atomic():
    jobs = list(Job.objects
                .select_for_update(skip_locked=True)
                .filter(status="pending")
                .order_by("created_at")[:BATCH])
    Job.objects.filter(pk__in=[j.pk for j in jobs]).update(status="processing")
# process the claimed jobs outside the lock
```

Index the filtered/ordered columns. This is the built-in alternative to a broker for simple DB-backed queues; reach for a real queue (see `background-work.md`) when you need scheduling, retries, or fan-out.

## Optimistic vs pessimistic

- **Pessimistic** (`select_for_update`): lock up front. Right for high-contention hot rows (a wallet balance, inventory in a flash sale) where optimistic retries would thrash.
- **Optimistic**: don't lock; detect the conflict at write time and retry.

```python
updated = (Account.objects
           .filter(pk=pk, version=expected_version)          # guard on the version you read
           .update(balance=F("balance") + amt, version=F("version") + 1))
if not updated:                       # 0 rows → someone else won the race
    raise ConcurrentUpdate            # reload and retry
```

`QuerySet.update()` returns the affected-row count — `0` means the guard failed. Optimistic is best for low-contention web writes: no lock held across the request, no connection tied up. Django 6.0 adds `Model.NotUpdated` (raised by `save(force_update=True)` when no row matched) for the same detection on model saves. Rule: **low contention → optimistic** (cheaper, no lock waits); **high contention on a specific row → pessimistic.**

## Deadlocks

Two transactions each holding a lock the other needs → PostgreSQL kills one with `deadlock detected`. Avoid by (1) acquiring locks in a **consistent order** everywhere (e.g. always lock accounts by ascending `id`), and (2) keeping transactions **short** — never do external I/O (HTTP, email) while holding a lock. Retry the victim with backoff.

## Isolation levels

PostgreSQL/Django default to READ COMMITTED. Raise it only for a specific invariant read-committed can't hold (e.g. a multi-row consistency check), not "to be safe".

```python
from psycopg import IsolationLevel
DATABASES["default"]["OPTIONS"] = {"isolation_level": IsolationLevel.REPEATABLE_READ}
```

REPEATABLE READ and SERIALIZABLE cost more and can abort with a serialization failure (SQLSTATE `40001`) under contention — you **must** catch it and retry the whole transaction, so wrap those transactions in a retry loop. Globally raising isolation converts lock waits into retry storms.

## Measure

Lock waits: `pg_stat_activity` where `wait_event_type = 'Lock'`; contended locks in `pg_locks`. Deadlock and serialization-failure frequency: the PostgreSQL log. Baseline these before adding a lock and confirm the fix didn't just move the contention elsewhere.

## Security note

A lock is not an authorization check — never widen a queryset to lock rows a user isn't allowed to see; keep the permission filter on the locked queryset. Defer authorization judgment to `secure-code-auditor`.
