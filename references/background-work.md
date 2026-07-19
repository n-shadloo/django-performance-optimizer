# Background work

Offloading is a design decision, not a reflex. Decide first, then pick the tool.

## Decide first: offload or optimize in-request?

- If the work is **inherently slow but not needed for the response** — sending email, image/video processing, calling webhooks, updating a search index, generating a report — **offload it** to a background worker and return immediately.
- If the work is **slow because the code is inefficient** — an N+1, a missing index, Python-side aggregation — **fix it in-request first.** Don't hide a bad query behind a queue; you'll just move the cost, keep the bug, and add operational complexity.

Prove which case you're in by measuring (see `profiling-and-tooling.md`) before adding infrastructure.

## Django 6.0 Tasks framework (`django.tasks`)

Django 6.0 ships a built-in Tasks API:

```python
from django.tasks import task

@task
def send_receipt(order_id: int) -> None:
    ...

send_receipt.enqueue(order_id)
```

Configure with the `TASKS` setting and a `BACKEND`. Built-in backends:

- **`ImmediateBackend`** — runs the task **synchronously** on enqueue. Dev only.
- **`DummyBackend`** — stores the task and never runs it (tests). Inspect `default_task_backend.results`, and `.clear()` between tests.

**Django ships the API, not a production worker** — real execution needs an external backend/worker. Task args and return values must be **JSON-serializable** (pass IDs, not model instances). `TaskResult` states: `READY` → `RUNNING` → `SUCCESSFUL` / `FAILED`. Use `queue_name` and `priority`; `takes_context=True` gives the task access to attempt count. It provides **no built-in retries or backoff** — retry policy is the backend's or your responsibility; see *Reliability* below.

**Enqueue after commit.** Workers use a separate DB connection and may not see rows from an uncommitted transaction. Wrap enqueue in `transaction.on_commit`:

```python
transaction.on_commit(lambda: send_receipt.enqueue(order.id))
```

## Celery

Mature distributed task queue: scheduling (Beat), workflows, retries with backoff, guaranteed delivery, distributed workers. Current stable **Celery 5.6.3 (26 Mar 2026; 5.6 added initial Python 3.14 support)** — target it (5.5.x still supported). Practices:

- Namespace settings with `CELERY_`; call `app.autodiscover_tasks()`.
- **Pass IDs, not model objects** (serialization + staleness).
- Set `time_limit` / `soft_time_limit` on every task.
- Use **separate queues per workload** so slow tasks don't starve fast ones.
- Enqueue after commit with `delay_on_commit()` (Celery 5.4+).
- **Caution (Vinta):** avoid Celery Canvas for complex workflows — use a dedicated workflow/orchestration tool instead.

## Reliability — idempotency, retries, outbox

Offloaded work runs outside the request; make it safe to run more than once and to fail without losing data.

- **Idempotency.** Retries and duplicate deliveries happen — give each task an idempotency key and enforce it with a unique constraint (or `get_or_create` on the key) so a repeat is a no-op. Prefer effects that are naturally idempotent (upsert, "set state to X") over "increment by one".
- **Retry with backoff + jitter.** Retry transient failures with exponential backoff plus random jitter so retries don't synchronize into a herd. Celery does this natively: `@shared_task(autoretry_for=(RequestException,), retry_backoff=True, retry_backoff_max=600, retry_jitter=True, max_retries=5)`. `django.tasks` leaves retry policy to you.
- **Enqueue after commit.** Workers use a separate connection and may not see an uncommitted row — wrap enqueue in `transaction.on_commit(...)` (above) so the worker never races ahead of the write.
- **Transactional outbox.** When even a lost `on_commit` enqueue is unacceptable, write an `Outbox` row *in the same transaction* as the state change; a separate relay polls unpublished rows, publishes them, and marks each sent. The message is published if and only if the transaction committed — this removes the dual-write gap between "DB committed" and "message sent". With the **database-backed** task backend the enqueue is itself part of the transaction, giving the same atomicity without a separate outbox table.

## When each

- **Django Tasks** — simple fire-and-forget and MVPs; "start with the built-in API and swap the backend as you grow."
- **Celery** (or **RQ / Dramatiq / Huey**) — scheduling, orchestration, durable retries, or large distributed throughput.

These are suggestions only — verify the library's health per `references/library-policy.md` before adopting it.

## Security note

Background tasks run with full application privileges and outside the request's auth context. Validate task inputs; don't trust the payload just because it came from your own queue. Defer authorization and secret-handling judgment to `secure-code-auditor`.
