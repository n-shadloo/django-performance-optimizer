# Data at scale

Django 6.0 / 5.2 LTS + PostgreSQL accurate. This is heavy machinery — replication, partitioning, sharding, materialized views. The elite move is almost always **not** to reach for it: exhaust indexing (`indexing-and-search.md`), caching (`caching.md`), connection pooling (`async-and-connections.md`), and query fixes (`orm-queries.md`) first. Introduce anything here only when a measured number — a read-saturated primary, a table in the tens of millions of rows, an aggregate too expensive to compute live — proves you need it. Each technique below states its trigger and its cost.

## Read/write splitting & replication

Route reads to a replica and writes to the primary with a database router:

```python
class PrimaryReplicaRouter:
    def db_for_read(self, model, **hints):  return "replica"
    def db_for_write(self, model, **hints): return "default"
    def allow_relation(self, *a, **k):      return True
    def allow_migrate(self, db, *a, **k):   return db == "default"
# settings: DATABASE_ROUTERS = ["app.routers.PrimaryReplicaRouter"]
```

`.using("default")` / `.using("replica")` overrides routing per query.

**Replica lag is a correctness bug, not just a metric.** A read routed to a lagging replica right after a write breaks read-your-own-writes — the user doesn't see what they just saved. Route post-write reads (and anything within the request that wrote) to the primary.

**Trigger:** the primary is genuinely read-bound — you've already indexed, cached, and pooled, and `pg_stat_statements` + CPU/IO show reads saturating it. Below that, this is premature: it adds routing bugs and lag hazards for no gain.

## Table partitioning

Native PostgreSQL declarative partitioning splits one huge table into child tables by RANGE (time-series on `created_at`) or LIST (tenant/region). The planner prunes irrelevant partitions, and you can drop an old partition instantly instead of a slow `DELETE`.

The ORM has no DDL for it: create the partitioned parent and partitions with `migrations.RunSQL`, and map a `managed = False` model over the parent; or use a helper such as `django-postgres-extra` (suggestion only — see `library-policy.md`). Define indexes on the parent so they propagate.

**Trigger:** a single table in the high tens-to-hundreds of millions of rows **and** a natural partition key with time/tenant retention. Below that, a good index beats partitioning at a fraction of the operational cost.

## Sharding — usually don't

Horizontal sharding across multiple PostgreSQL servers is rarely worth it in Django: the ORM can't query or join across shards, and you lose cross-shard transactions, joins, and unique constraints. Exhaust vertical scaling, read replicas, and partitioning first. Shard only when one primary genuinely can't hold the write volume, and expect to hand-roll routing. Say this plainly to anyone reaching for it early.

## Materialized views & rollups

- **Materialized view** — precompute an expensive aggregation read far more than it changes. Create it with `RunSQL`, back it with a `managed = False` model, and refresh on a schedule/task with `REFRESH MATERIALIZED VIEW CONCURRENTLY` (needs a unique index; doesn't lock readers).
- **Counter / rollup tables** — maintain aggregates incrementally instead of `COUNT(*)`/`SUM(...)` over millions of rows on every read. **Trigger-maintained** (DB triggers update the rollup on write — strongly consistent, DB-coupled) vs **application-maintained** (update via a task or `F()` expression — flexible, eventually consistent). Triggers for correctness-critical counts; app-side for flexibility.
- **`GeneratedField` for row-local values** — for a value that depends only on its own row (not a cross-row aggregate), a `GeneratedField` (`db_persist=True`) gives a DB-maintained `STORED` column with no trigger. The lightest option — but it can't replace rollups, which aggregate across rows.

## Boundaries — where Django performance hands off

- **Kubernetes / autoscaling** — HPA scales replicas on CPU/latency/custom metrics; Django's job ends at making one pod efficient (query count, latency, and the connection-count math in `async-and-connections.md` — more pods means more DB connections). Pod capacity/scaling is an infra discipline.
- **Cost / capacity planning** — instance right-sizing, reserved capacity, and DB IOPS provisioning are FinOps/infra concerns downstream of the per-request efficiency this skill controls.
- **GraphQL / API shape** — GraphQL's N+1 lives at the resolver layer and is solved with DataLoader batching (`strawberry-django`'s query optimizer, or `graphene-django` + DataLoaders; in sync Django, `graphql-sync-dataloaders`). The ORM/serializer advice here still applies underneath; the batching layer is where it meets GraphQL. Libraries are suggestions — see `library-policy.md`.

## Security note

Replica routing must not bypass row-level permission filters, and a materialized view can expose data its base tables would have restricted — treat its access scope explicitly. Defer authorization judgment to `secure-code-auditor`.
