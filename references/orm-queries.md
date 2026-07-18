# ORM queries

Django 6.0 / 5.2 LTS accurate. The ORM is where most Django performance is won or lost. Measure query count first (see `profiling-and-tooling.md`), then apply the minimal fix.

## QuerySet laziness and evaluation

QuerySets are lazy: building and chaining `.filter()`, `.exclude()`, `.annotate()`, `.order_by()` costs nothing. SQL runs only when the queryset is **evaluated** — on iteration, slicing with a step, `len()`, `list()`, `bool()`, `repr()`, or pickling. An evaluated queryset caches its result rows; re-iterating the *same* object reuses the cache, but re-creating the queryset re-hits the DB.

```python
qs = Book.objects.filter(published=True)   # no SQL yet
for b in qs: ...                            # SQL runs, result cached
for b in qs: ...                            # served from cache, no SQL
list(Book.objects.filter(published=True))  # a NEW queryset, hits DB again
```

Slicing with `[:10]` adds `LIMIT`/`OFFSET` and does not populate the cache; `if qs:` evaluates the whole queryset — use `.exists()` instead.

## The N+1 problem

Definition: 1 query for the list plus 1 additional query per row for a related attribute accessed lazily. Detect it by query count rising with row count.

```python
# N+1: 1 + N queries (one per book for book.author)
for book in Book.objects.all():
    print(book.author.name)
```

Fix with the right tool for the relation direction:

- **`select_related(*fields)`** — SQL `JOIN`, single query. For forward `ForeignKey` and `OneToOneField` (to-one).
- **`prefetch_related(*lookups)`** — a second query plus a Python-side join. For reverse FK, `ManyToManyField`, and generic relations (to-many).

```python
# select_related: 847 queries → 2 (list + one JOIN) ... actually 1 with JOIN
Book.objects.select_related("author")           # book.author is free
# prefetch_related: 1 + 1 = 2 queries regardless of row count
Author.objects.prefetch_related("books")        # author.books.all() is free
```

Combine them: `Book.objects.select_related("author").prefetch_related("tags")`.

## `Prefetch` objects

Use `Prefetch` to filter or optimize the prefetched queryset, land it on a custom attribute, or nest a `select_related`:

```python
from django.db.models import Prefetch

Author.objects.prefetch_related(
    Prefetch(
        "books",
        queryset=Book.objects.filter(published=True).select_related("publisher"),
        to_attr="published_books",
    )
)
# author.published_books is a plain list; author.books still available separately
```

`.using(alias)` on the inner queryset targets another DB. **Warnings:** calling `.filter()` (or `.all().filter(...)`) on an already-prefetched relation issues a **new** query and wastes the prefetch cache — filter inside the `Prefetch` instead. Mutating methods (`add()`, `create()`, `remove()`, `clear()`, `set()`) invalidate the prefetch cache.

## `only()` / `defer()`

Advanced, easy to misuse. `only(*fields)` loads just those columns; `defer(*fields)` loads everything except those. Accessing a deferred field triggers **an extra query per instance** — a pessimization if the field is used after all. Don't defer aggressively without profiling. If you routinely need only a subset of columns, that's a signal to normalize the model, not to sprinkle `defer()`.

## `values()` / `values_list()`

Skip model instantiation entirely for read-only/reporting paths — returns dicts / tuples instead of model objects, which is measurably cheaper:

```python
Book.objects.values("id", "title")                       # [{"id":..,"title":..}, ...]
Book.objects.values_list("id", flat=True)                # [1, 2, 3, ...]
```

Use for exports, dropdown choices, and aggregation feeds where behavior/methods aren't needed.

## Aggregation in the DB, not Python

Compute counts, sums, and averages in SQL — never by iterating model instances in Python.

```python
from django.db.models import Count, Sum, Avg, F, Case, When, Value

Author.objects.annotate(n_books=Count("books"))          # per-row count in SQL
Order.objects.aggregate(total=Sum("amount"))             # {"total": Decimal(...)}
Product.objects.update(stock=F("stock") - 1)             # atomic, no read-modify-write
Book.objects.filter(price__gt=F("list_price"))           # field-to-field comparison
Order.objects.annotate(
    tier=Case(When(amount__gte=100, then=Value("gold")), default=Value("std"))
)
```

`F` expressions also avoid race conditions on updates by doing the arithmetic in the database.

## `exists()` vs `count()` vs truthiness

- `.exists()` — "is there any?" Emits `SELECT 1 ... LIMIT 1`. Use in `if`.
- `.count()` — "how many?" Emits `SELECT COUNT(*)`.
- Don't `len(qs)` to test emptiness (evaluates and caches all rows), don't `.count()` and *then* also iterate the same logical set (two queries).

Heed Django's own caution: if you already have or will evaluate the queryset, don't add a separate `.count()`/`.exists()`/`.contains()` round-trip — check `len()` of the already-fetched cache or use the cached result.

## Subqueries and expressions

Push correlated logic into one query with `Subquery`, `OuterRef`, and `Exists()`:

```python
from django.db.models import Subquery, OuterRef, Exists

latest = Book.objects.filter(author=OuterRef("pk")).order_by("-published")
Author.objects.annotate(latest_title=Subquery(latest.values("title")[:1]))

has_recent = Book.objects.filter(author=OuterRef("pk"), published__year=2026)
Author.objects.annotate(recent=Exists(has_recent)).filter(recent=True)
```

Database functions (`Coalesce`, `Concat`, `Trunc`, `Lower`, `Greatest`, …) keep computation in SQL.

## Bulk operations

One statement instead of a loop of `.save()`:

```python
Book.objects.bulk_create(books, batch_size=500)
Book.objects.bulk_update(books, ["price", "stock"], batch_size=500)
Book.objects.filter(discontinued=True).update(active=False)   # single UPDATE
Book.objects.filter(active=False).delete()                    # single DELETE
```

Use `batch_size` to bound statement size. **Version note:** Django 6.0.1 (6 Jan 2026) fixed a bug where `bulk_create()` on PostgreSQL silently **truncated** data exceeding a field's `max_length` — target patched releases (≥ 6.0.1, ≥ 5.2.x current). `bulk_create`/`bulk_update` skip `save()` and do not send `pre_save`/`post_save` signals — validate explicitly if needed (defer validation-correctness judgment to `secure-code-auditor`).

## Avoiding queries in loops — the cardinal rule

Never issue a query inside a per-row loop. Precompute a lookup map, prefetch, or annotate:

```python
# Bad: one query per order
for order in orders:
    customer = Customer.objects.get(pk=order.customer_id)

# Good: one query, dict lookup O(1)
customers = Customer.objects.in_bulk([o.customer_id for o in orders])
for order in orders:
    customer = customers[order.customer_id]
```

## Large result sets with `iterator()`

`iterator(chunk_size=...)` streams rows via a server-side cursor instead of filling the result cache — essential for exports and batch jobs over large tables:

```python
for book in Book.objects.iterator(chunk_size=2000):
    process(book)
```

Memory tradeoff: `iterator()` deliberately bypasses caching, so don't combine it with repeated re-iteration. `prefetch_related` **is** supported with `iterator()` as long as you pass a `chunk_size` (the prefetch runs per chunk).

## Indexing strategy — the number-one lever after profiling

Index the columns used in `filter()`, `exclude()`, `order_by()`, and join keys. Weigh the write-time maintenance cost (every index slows inserts/updates).

```python
class Book(models.Model):
    isbn = models.CharField(max_length=13, db_index=True)      # single-column
    class Meta:
        indexes = [
            models.Index(fields=["author", "-published"]),     # composite
            models.Index(fields=["status"], condition=Q(status="active"),
                         name="active_book_idx"),              # partial/conditional
        ]
```

- **Composite indexes** follow the **leftmost-prefix** rule — order matters. An index on `(author, published)` serves filters on `author` and on `(author, published)`, but not on `published` alone.
- **Partial/conditional indexes** (`condition=Q(...)`) index only matching rows — smaller and cheaper when queries always filter the same way.
- **Functional and covering indexes** where the backend supports them.
- Always index the fields a cursor-paginator orders by (see `serializers-drf.md`).

## `explain()`

```python
print(Book.objects.filter(author_id=5).explain())                    # plan
print(Book.objects.filter(author_id=5).explain(analyze=True))        # PostgreSQL EXPLAIN ANALYZE
```

Read the plan for **seq scan vs index scan** and **estimated vs actual rows**. A seq scan on a large filtered table, or a large gap between estimated and actual rows, points at a missing index or stale statistics.

## Encode optimizations on managers / QuerySets

Keep views clean and make eager-loading travel with the query:

```python
class BookQuerySet(models.QuerySet):
    def with_author(self):
        return self.select_related("author")
    def with_stats(self):
        return self.annotate(n_reviews=Count("reviews"))

class Book(models.Model):
    objects = BookQuerySet.as_manager()

# view: Book.objects.with_author().with_stats()
```

## Security note

Never disable a permission-enforcing query filter for speed — a widened queryset is an access-control bug, not an optimization. Always parameterize `raw()` / `extra()`; never format user input into SQL strings. Defer injection and authorization judgment to `secure-code-auditor`.
