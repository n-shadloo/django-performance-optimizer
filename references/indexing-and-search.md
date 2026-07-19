# Indexing depth & search

Django 6.0 / 5.2 LTS + PostgreSQL accurate. Builds on the indexing basics in `orm-queries.md` — index what you filter/order/join on, prove the need with `explain()`, and weigh write-time index maintenance. This file goes deeper: advanced index types, text search, and `JSONField`, all PostgreSQL.

## Advanced index types

- **Covering indexes (`include=`)** carry extra columns as non-key payload so the planner answers from the index alone (an index-only scan, no heap fetch):

  ```python
  models.Index(fields=["customer"], include=["total", "status"], name="cust_cover_idx")
  ```

  Confirm with `explain(analyze=True)`: look for `Index Only Scan` and near-zero `Heap Fetches`. Also available on `UniqueConstraint(include=...)`.

- **BRIN (`BrinIndex`)** — a tiny index for large, naturally-ordered / append-only tables (time-series keyed by `created_at` or an autoincrement id). It stores per-block min/max, so it's a fraction of a B-tree's size. Wins when physical row order correlates with the column; useless when it doesn't.

  ```python
  from django.contrib.postgres.indexes import BrinIndex
  BrinIndex(fields=["created_at"])
  ```

- **Functional / expression indexes** index a computed expression so a matching query can use it. The classic is case-insensitive lookup:

  ```python
  from django.db.models.functions import Lower
  models.Index(Lower("email"), name="email_lower_idx")   # serves filter(email__iexact=…)
  ```

- **Partial / conditional (`condition=`)** indexes only the rows queries actually touch — smaller and cheaper to maintain. Ideal for a hot subset:

  ```python
  from django.db.models import Q
  models.Index(fields=["created_at"], condition=Q(status="pending"), name="pending_idx")
  ```

## Finding the real offenders — `pg_stat_statements`

Don't guess which queries need indexes. Enable `pg_stat_statements` (add it to `shared_preload_libraries`) and read it sorted by `total_exec_time`, `mean_exec_time`, and `calls` — that ranks the queries actually costing time in production. Reset with `pg_stat_statements_reset()` before a measurement window. This is the production-scale version of the measure-first rule (see `profiling-and-tooling.md`).

## Why the planner ignores an index

A present index that isn't used almost always means one of:

- A function or cast wraps the column in the query but not in the index (`WHERE lower(email) = …` with only a plain `email` index → add the matching functional index).
- A type mismatch forces a cast (comparing a `varchar` column to an integer literal).
- Low selectivity — the predicate matches a large fraction of rows and a sequential scan is genuinely cheaper; the planner is right and the index is the wrong fix.
- Stale statistics — run `ANALYZE`.

Read `explain(analyze=True)`: a `Seq Scan` where you expected an index, a large `estimated` vs `actual` rows gap (stale stats), or the column under a `Filter:` instead of an `Index Cond:`.

## Text search & pattern matching

- **`__icontains` / `LIKE '%term%'` cost** — a leading wildcard can't use a B-tree (B-trees order left-to-right), so it falls back to a sequential scan, O(n) per query. Fine on small / low-QPS tables; a cliff at scale.
- **Trigram (`pg_trgm`)** accelerates substring and fuzzy/typo-tolerant search. Enable with the `TrigramExtension()` migration operation, index with a GIN + `gin_trgm_ops` opclass, query with `TrigramSimilarity` / `__trigram_similar`:

  ```python
  from django.contrib.postgres.indexes import GinIndex
  GinIndex(fields=["name"], opclasses=["gin_trgm_ops"], name="name_trgm_idx")
  # Author.objects.filter(name__trigram_similar="jorj")   # matches "George", index-backed
  ```

- **Full-text search (`django.contrib.postgres.search`)** — `SearchVector` / `SearchQuery` / `SearchRank`, with `config=` (language) and per-field `weight=` ("A"–"D"). An on-the-fly `SearchVector` recomputes the `tsvector` every query — fine for small data or occasional search. For anything hot, store and index the vector. The built-in way is a **`GeneratedField`** — a DB-computed `STORED` column that PostgreSQL maintains, with no trigger and no signal code:

  ```python
  from django.contrib.postgres.search import SearchVector, SearchVectorField
  from django.contrib.postgres.indexes import GinIndex

  class Article(models.Model):
      title = models.CharField(max_length=200)
      body = models.TextField()
      search = models.GeneratedField(
          expression=SearchVector("title", "body", config="english"),
          output_field=SearchVectorField(),
          db_persist=True,
      )
      class Meta:
          indexes = [GinIndex(fields=["search"])]
  # Article.objects.filter(search=SearchQuery("django orm", config="english"))
  ```

  This supersedes the older `tsvector_update_trigger` and manual `.update(search=SearchVector(...))` approaches (both still valid, now fallbacks). A stored + GIN-indexed vector is dramatically faster than recomputing per query — a difference that widens with row count.

- **When to leave PostgreSQL** — stay on `tsvector` + trigram until you have a measured need PostgreSQL can't meet: typo-tolerance as the default, faceted search and aggregations at scale, sub-50 ms search-as-you-type over millions of documents, deep relevance tuning, or many-language analyzers. Only then reach for a dedicated engine (Elasticsearch / OpenSearch / Meilisearch / Typesense) — a suggestion, not a default. See `library-policy.md`: the search *servers* are healthy, but several Django integration packages are not.

## `JSONField` performance

- **Index containment with GIN.** For `@>` / `?` / `has_key` queries, `GinIndex(fields=["data"])` (default `jsonb_ops`). If you only need containment (`@>`), `jsonb_path_ops` is smaller and faster but supports containment only:

  ```python
  GinIndex(fields=["data"], opclasses=["jsonb_path_ops"], name="data_gin_idx")
  # Model.objects.filter(data__contains={"role": "admin"})
  ```

- **Index one hot key** with a functional index on the extracted value (`KeyTextTransform`/`->>`) when you filter on a single path a lot, instead of GIN-indexing the whole document.
- **Denormalize when** you filter/sort/join on a JSON key frequently, or need a unique/FK constraint or a composite index on it — promote it to a real column. Signal: the JSON key shows up in the `WHERE`/`ORDER BY` of your top `pg_stat_statements` offenders, or `explain()` shows a GIN recheck / seq scan on the path. Keep data in JSON when it's schemaless, rarely filtered, or read as a blob.

## Security note

Trigram / full-text / JSON filters are still queryset filters — don't drop a permission filter to "simplify" a search queryset, and parameterize any raw search SQL. Defer injection and authorization judgment to `secure-code-auditor`.
