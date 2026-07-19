# Serializers & DRF

DRF 3.17.1 accurate (released 24 Mar 2026; 3.17 added official support for Django 6.0 and Python 3.14). Serialization is frequently the second-largest cost after the ORM, and DRF makes N+1s easy to introduce.

## Serializer N+1s

A nested serializer (`many=True`, or a related object serialized inline) triggers a per-row query for each related object the view didn't eager-load. The serializer doesn't know about the queryset — you fix it in the view's `get_queryset()`.

```python
class BookSerializer(serializers.ModelSerializer):
    author = AuthorSerializer()           # touches book.author per row
    tags = TagSerializer(many=True)       # touches book.tags per row
    class Meta:
        model = Book
        fields = ["id", "title", "author", "tags"]

class BookViewSet(viewsets.ReadOnlyModelViewSet):
    serializer_class = BookSerializer
    def get_queryset(self):
        # match eager-loading to exactly what the serializer touches
        return (Book.objects
                .select_related("author")        # to-one → JOIN
                .prefetch_related("tags"))       # to-many → 2nd query
```

Baseline on 200 rows: **1 (list) + 200 (author) + 200 (tags) = 401 queries → 3 queries** (list + author JOIN folded in + one tags prefetch). Encode this on a queryset method (`setup_eager_loading` / `with_display()`) so it can't drift from the serializer.

## `SerializerMethodField` cost

A `SerializerMethodField` whose method does `obj.author.name` or runs a query per object **is** an N+1. Two rules:

- For plain attribute traversal, use `source` on a regular field instead of a method, and prefetch the relation — no per-object Python call, no method overhead:

  ```python
  author_name = serializers.CharField(source="author.name", read_only=True)
  ```

- Reserve `SerializerMethodField` for genuine computation over data that is **already loaded or annotated**. If the method needs an aggregate, annotate it in the queryset and read the attribute:

  ```python
  # queryset: .annotate(n_reviews=Count("reviews"))
  n_reviews = serializers.IntegerField(read_only=True)   # reads the annotation
  ```

## `source` vs method fields; read-only optimization

Mark computed/derived fields `read_only=True`. Writable-field validation is measurably expensive. Haki Benita's benchmark ("Improve Serialization Performance in Django Rest Framework"), over 5,000 serializations:

| Approach | Time |
| --- | --- |
| Writable `ModelSerializer` | **12.818 s** |
| Plain read-only `Serializer` | **2.101 s** |
| Hand-written function (dict from `values()`) | **0.034 s** |

So a `ModelSerializer` is **≈ 6× slower than a plain `Serializer`** and **≈ 377× slower than inline dict construction**. For hot read-only endpoints: use a leaner `Serializer` (not `ModelSerializer`), or build dicts directly from `.values()`. When you profile an endpoint, state serialization's measured share of request time so the choice is evidence-based.

## `to_representation` cost & shape

Choose the smallest serializer shape that satisfies the endpoint. Annotate aggregates in the queryset rather than computing them per object in `to_representation`. Avoid deep nesting when a single flat annotated field suffices. Every level of nesting multiplies both query risk and Python object construction.

## Choosing shape by access pattern

- **List endpoints** → a lightweight read serializer (few fields, flat, annotated aggregates) plus pagination. High cardinality makes per-object cost dominate.
- **Detail endpoints** → richer nesting is fine; at cardinality 1 the per-object cost is paid once.

Don't reuse the heavy detail serializer for the list view.

## Pagination & large result sets

- **`PageNumberPagination` / `LimitOffsetPagination`** are offset-based: `LIMIT/OFFSET` makes the database **scan and discard** `OFFSET` rows, so deep pages degrade linearly. Inserts/deletes between requests shift the window, so rows are skipped or duplicated across pages.
- **`CursorPagination`** is fixed-time regardless of position, consistent under concurrent writes, and uses an opaque cursor. Requirements and tradeoffs: needs a unique, unchanging, **indexed** ordering field (default `-created`); no arbitrary page jumps; no total count.

  ```python
  class FeedPagination(CursorPagination):
      page_size = 50
      ordering = ("-created_at", "id")     # tiebreaker for stable ordering
  ```

  Recommend cursor pagination for feeds, timelines, and large listings. Keep offset pagination for small datasets or admin UIs that need page jumps / total counts. **Index every field in the ordering tuple.** `CursorPagination` is keyset pagination underneath — a `WHERE (created_at, id) < (:last_created, :last_id) ORDER BY … LIMIT n` seek, which is why it stays fixed-time regardless of depth. For a non-DRF endpoint, implement that seek by hand rather than `OFFSET`.
- **Never** ship an unpaginated list endpoint, and never materialize a huge queryset into a `list`.
- **DRF 3.17 note:** async pagination helpers exist in the current stack — use them from async views rather than wrapping sync pagination.

## Streaming large responses

For CSV/export endpoints, stream instead of buffering the whole payload:

```python
from django.http import StreamingHttpResponse

def export_books(request):
    def rows():
        yield "id,title\n"
        for b in Book.objects.values_list("id", "title").iterator(chunk_size=2000):
            yield f"{b[0]},{b[1]}\n"
    return StreamingHttpResponse(rows(), content_type="text/csv")
```

`iterator(chunk_size=...)` keeps memory flat regardless of table size.

## Security note

Eager loading must not widen access. Keep the exact same permission/queryset filtering when you add `select_related`/`prefetch_related` — optimizing must never turn into fetching rows the user isn't allowed to see. Defer authorization correctness to `secure-code-auditor`.
