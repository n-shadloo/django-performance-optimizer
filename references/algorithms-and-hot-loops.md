# Algorithms & hot loops

The legitimately transferable layer. These principles are general; the section headers note where they apply beyond Django. Rule up front: **only touch a hot loop you've profiled or can prove is hot** by inspection.

## Complexity of hot paths (transfers to any backend)

Find the loop that runs per-request or per-row and count the nested iteration. The classic bug is **accidental O(n²)** from a membership test against a list inside a loop:

```python
# O(n·m): `in existing_ids` scans a list every iteration
existing_ids = list(Order.objects.values_list("id", flat=True))
for row in incoming:            # n
    if row.id in existing_ids:  # O(m) scan → O(n·m) overall
        ...

# O(n+m): set membership is O(1)
existing_ids = set(Order.objects.values_list("id", flat=True))
for row in incoming:
    if row.id in existing_ids:  # O(1)
        ...
```

Nested Python joins (looping over A, looping over B to find matches) are the same trap — build an `{obj.id: obj}` map once and look up in O(1).

## Data structures (transfers to any backend)

- `set` / `dict` for membership and grouping; `dict` lookup and `in set` are O(1).
- `collections.defaultdict` for grouping without key checks; `collections.Counter` for tallies.
- `bisect` for lookups/inserts into a sorted sequence.
- **Generators** to avoid materializing large intermediate lists.
- Avoid repeated `list.index()` and `x in some_list` inside loops — both are O(n).

```python
from collections import defaultdict
books_by_author = defaultdict(list)
for book in Book.objects.all():          # one pass
    books_by_author[book.author_id].append(book)
```

## Do work in the database, not Python (the Django application of "push work down")

Aggregation, filtering, joining, and ordering belong in indexed, set-based SQL — not a Python loop over model instances. A `Count`/`Sum` annotation is one indexed aggregate query; the equivalent Python loop pulls every row into memory and iterates. See `orm-queries.md` for the ORM constructs. This is the single highest-leverage version of "push work down to the fastest layer."

## Precompute and hoist

Move invariant computation out of loops; build lookup maps once before the loop, not inside it. For pure, repeated computation use memoization:

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def slugify_cached(name: str) -> str:
    ...
```

`functools.lru_cache` for pure functions; `cached_property` for per-instance results (see `caching.md`).

## String / serialization hot spots

Avoid quadratic string building (`s += chunk` in a loop reallocates each time). Build a list and `"".join(parts)`. The same shape applies to constructing large response bodies — accumulate, then join/stream.

## Rule

Profile first (or prove it by inspection — a nested `in list` over per-request data is provably O(n²)). Don't rewrite a loop that runs three times; do rewrite one that runs per row on a large result set.
