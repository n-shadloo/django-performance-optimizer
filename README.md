# django-performance-optimizer

Write and audit maximally performant Django & DRF code. Measure first, then fix.

## Why

Performance review and performance-aware writing are repetitive and easy to get wrong. The wins — query count, serialization cost, caching, pagination, connections — are well understood, but the knowledge is spread across the Django and DRF docs, third-party blog posts, and version-sensitive release notes. This skill packages that so an agent applies it the same way every time and points a reviewer straight at what matters.

It is built on two rules. **Measure first, then fix:** every performance claim is grounded in a number — the exact query count, the specific N+1 relation, the row count, the measured time — and once a real bottleneck is proven, the skill *fixes* it rather than just flagging it. **Security wins, always:** it never trades security for speed and defers all security judgment to the author's [`secure-code-auditor`](https://github.com/n-shadloo/secure-code-auditor) skill.

## What it does

- Measures before changing anything — query counts, timings, `QuerySet.explain()`.
- Eliminates N+1s in the ORM (`select_related`/`prefetch_related`, `Prefetch`) and in DRF serializers.
- Pushes work into the database — aggregation, filtering, ordering as indexed set-based SQL, not Python loops.
- Picks the right pagination — cursor vs offset — by access pattern.
- Caches deliberately, with correct invalidation and stampede protection.
- Gets async, connections, and pooling right — `CONN_MAX_AGE`, native psycopg3 pooling, PgBouncer.
- Handles concurrency correctly — `select_for_update`/`skip_locked`, optimistic vs pessimistic locking, and transaction isolation.
- Makes PostgreSQL find data fast — covering/BRIN/functional/partial indexing, trigram and full-text search, and `JSONField` GIN indexing.
- Reaches for scale-out — replica routing, partitioning, materialized views — only when a measured number warrants it.
- Decides offload-vs-optimize instead of hiding a bad query behind a queue.
- Fixes accidental O(n²) hot loops with the right data structure.
- Applies write-time performance defaults when generating code, so it's fast the first time.
- Never trades security for speed — defers to `secure-code-auditor`.
- Suggests third-party libraries when they beat the built-in for a measured bottleneck, but never mandates them.

## Works with

Claude (primary), plus Codex, Cursor, and Gemini CLI. Claude, Codex, and Cursor read `SKILL.md` + `references/` as an Agent Skill (shared standard, same folder). Gemini CLI reads `GEMINI.md`. `AGENTS.md` is the always-on context source that the Cursor rule and `GEMINI.md` point back to.

## Version baseline

Kept current: **Django 6.0.7 / 5.2.16 LTS; DRF 3.17.1; Python 3.14; Celery 5.6.3** (as of 19 Jul 2026). The Django 6.0 series receives security and data-loss fixes through **April 30, 2027**; the 5.2 LTS series through **April 30, 2028**. Advice targets Django 6.0 and flags where 5.2 differs — notably native connection pooling under ASGI, which 6.0 supports via `AsyncConnectionPool` but 5.1/5.2 advise against (use PgBouncer there). Django 6.1 is in beta (6.1b1, expected Aug 2026); its performance-relevant features (ORM fetch modes, database-level `on_delete`) are flagged where relevant as not yet shipped.

## Install

### Claude

One project:
```
git clone https://github.com/n-shadloo/django-performance-optimizer.git \
  .claude/skills/django-performance-optimizer
```
All your projects:
```
git clone https://github.com/n-shadloo/django-performance-optimizer.git \
  ~/.claude/skills/django-performance-optimizer
```
For claude.ai or the API, upload the folder as a custom skill in Settings.

### Codex CLI

Codex CLI discovers Agent Skills from `.agents/skills/` and reads `AGENTS.md` for always-on project context.

One project:
```
git clone https://github.com/n-shadloo/django-performance-optimizer.git \
  .agents/skills/django-performance-optimizer
```
All your projects:
```
git clone https://github.com/n-shadloo/django-performance-optimizer.git \
  ~/.agents/skills/django-performance-optimizer
```

### Cursor

Cursor supports Agent Skills, so the same clone works:
```
git clone https://github.com/n-shadloo/django-performance-optimizer.git \
  .cursor/skills/django-performance-optimizer
```
Cursor also reads `AGENTS.md` at the repo root and the bundled `.cursor/rules/django-performance-optimizer.mdc` rule, which reinforces the same guidance.

### Gemini CLI

Gemini CLI doesn't read Agent Skills directly; it reads `GEMINI.md`.
- **Per project:** copy `GEMINI.md` into the repository root.
- **All projects:** copy it to `~/.gemini/GEMINI.md`.

The only requirement is `git` and a Git repository to run in.

## Use

**Review-time** — measure, then fix:

> "This DRF list endpoint is slow — profile it and fix the N+1."

The skill counts queries, proves the bottleneck, and reports in the measure-first shape:

```
Baseline:  401 queries / 2.3 s on 200 rows
Diagnosis: BookSerializer.author and .tags are lazy — one query per row each
Fix:       select_related("author").prefetch_related("tags") in get_queryset()
Result:    → 3 queries
```

**Write-time** — fast the first time:

> "Generate a paginated, cached products API."

It produces a cursor-paginated, eager-loaded, read-optimized endpoint and notes the performance-relevant choices it made.

## Example

**Location:** `api/views.py` — `BookViewSet.get_queryset()` feeding `BookSerializer` (nested `author`, `many=True` `tags`).

**Baseline:** 401 queries / 2.3 s on a 200-row page — query count scales 1-for-1 with rows.

**Diagnosis:** an N+1 on two relations. `book.author` (forward FK) and `book.tags` (M2M) are accessed per row by the serializer, and the view didn't eager-load them.

**Fix:**
```python
def get_queryset(self):
    return (Book.objects
            .select_related("author")        # forward FK → JOIN
            .prefetch_related("tags"))       # M2M → one extra query
```

**Result:** → 3 queries (list + folded author JOIN + one tags prefetch), constant regardless of page size. Same permission filtering preserved — the eager load doesn't widen access.

## Layout

```
django-performance-optimizer/
├── SKILL.md                              # canonical skill and router
├── AGENTS.md                             # always-on project context; source for the pointers below
├── GEMINI.md                             # Gemini CLI context
├── .cursor/
│   └── rules/
│       └── django-performance-optimizer.mdc   # Cursor reinforcement rule (points to AGENTS.md)
├── references/
│   ├── orm-queries.md
│   ├── serializers-drf.md
│   ├── caching.md
│   ├── async-and-connections.md
│   ├── background-work.md
│   ├── algorithms-and-hot-loops.md
│   ├── profiling-and-tooling.md
│   ├── settings-and-runtime.md
│   ├── concurrency-and-locking.md
│   ├── indexing-and-search.md
│   ├── data-at-scale.md
│   ├── rate-limiting-and-backpressure.md
│   └── library-policy.md
├── README.md
├── LICENSE
└── .gitignore
```

## License

MIT. See [`LICENSE`](LICENSE).
