---
name: query-pushdown
description: Push sorting, filtering, searching, aggregation, pagination, deduplication, and grouping DOWN into the data layer (database, search engine, cache, message broker, API) instead of doing it in Python. Use this skill ANY time code touches a queryset, SQL query, ORM call, search client, dataframe, list of records fetched from any service, or a Python loop/comprehension/`sorted()`/`filter()`/`set()`/`max()`/`sum()` operating over data that came from an external system. Trigger on Django ORM (`.filter`, `.order_by`, `.annotate`, `.aggregate`, `.distinct`, `.values`, `.exclude`), raw SQL, ClickHouse, OpenSearch/Elasticsearch, Redis, MongoDB, BigQuery, pandas/Polars, REST/GraphQL pagination, S3 list operations, and any equivalent technology. Trigger when the user asks to add a filter, add sorting, search, paginate, count, deduplicate, group, aggregate, or compute statistics — even if they don't explicitly mention "the database". Trigger on review/refactor tasks when scanning existing code: flag every Python-side data op that the data source could have done. If python-side processing seems unavoidable, STOP and ask the user permission with a clear justification before proceeding.
---

# Query Pushdown

Data operations belong in the system that owns the data. Python is the orchestrator, not the query engine.

## Why this matters

When you sort/filter/search/aggregate in Python, you pay:
- **Network cost**: every row crosses the wire, even rows you'll discard.
- **Memory cost**: the whole result set lives in RAM in the app process.
- **CPU cost**: Python loops are 10-1000× slower than indexed DB ops.
- **Correctness cost**: collation, NULL handling, locale, timezone, full-text scoring — Python rarely matches the source semantics.
- **Pagination cost**: `LIMIT/OFFSET` at the source returns 20 rows; `[:20]` after `.all()` materialized millions.

The data layer has indexes, query planners, vectorized execution, and was built for this. Use it.

## The rule

**Default: push the operation down to the source.** Python only orchestrates: it composes the query, hands it off, and consumes the (already-shaped) result.

**Exception path**: if pushdown is impossible or genuinely worse, STOP, explain why in one sentence, and ask the user permission before doing it in Python. Don't silently fall back.

## How to apply

When you see (or are about to write) any of these patterns, ask: "could the source do this?"

- `sorted(qs, key=...)` → `qs.order_by(...)`
- `[x for x in qs if cond]` → `qs.filter(...)`
- `len([x for x in qs if cond])` → `qs.filter(...).count()`
- `set(x.field for x in qs)` → `qs.values_list("field", flat=True).distinct()`
- `sum(x.amount for x in qs)` → `qs.aggregate(Sum("amount"))`
- `max(qs, key=...)` → `qs.order_by("-field").first()` or `.aggregate(Max(...))`
- `list(qs)[:20]` → `qs[:20]` (slicing a queryset issues `LIMIT`; slicing a list does not)
- in-Python join across two querysets → `select_related` / `prefetch_related` / SQL join
- pandas `.sort_values()` after `pd.read_sql(SELECT *)` → push `ORDER BY` into the SQL
- `[hit for hit in es_results if ...]` → add the predicate to the OpenSearch/Elasticsearch query DSL
- `sorted(redis.lrange(...))` → use a sorted set (`ZRANGE`) instead of a list
- iterating S3 `list_objects` then filtering by prefix → pass `Prefix=` to `list_objects_v2`
- looping pages of a paginated API to find one item → use the API's filter/search params

The same principle holds for ClickHouse, BigQuery, MongoDB (`$match`/`$sort`/`$group`), GraphQL (use the schema's filter args), gRPC (use server-side filters), Kafka (filter at consumer config or via stream processor), etc. Different syntax, same idea: **make the source do the work.**

## When pushdown is genuinely OK to skip

These are the cases where Python-side ops can be justified — but still surface the choice to the user:

- **Tiny, fixed, in-memory collections** (e.g., enum values, settings dict, ~10 items already loaded for unrelated reasons).
- **Already fully materialized for another required reason** — the data is in a list because something earlier needed it that way, and re-querying would be wasteful.
- **Source can't express the predicate** — e.g., a regex/computation the DB doesn't support and adding a stored function/index is out of scope.
- **Cross-source operations** — joining results from two different systems where neither owns both sides.
- **Post-processing of aggregated results** — summary stats over a small result set already reduced by the DB.
- **Offline batch script with no scaling concern** — one-shot data migration on a dev machine.

If the situation matches one of these, you can do it in Python — but say so explicitly: "I'm doing this in Python because <reason>; OK?"

## Asking permission — the format

When pushdown looks blocked, don't just write the Python loop. Output something like:

> I want to push this filter into the query, but `<source>` doesn't support `<operation>` because `<reason>`. Options:
> 1. Do it in Python after fetching `<N>` rows — works, but materializes the full result.
> 2. `<alternative — add an index, use a generated column, switch source, etc.>`
> 3. `<other alternative if any>`
>
> Which do you want?

Then wait. Don't pre-emptively pick option 1.

## Reviewing existing code

When the task is review or refactor, scan for these red flags:

- `for x in qs:` followed by an `if` (filtering)
- `sorted(...)` / `.sort()` / `key=lambda` over query results
- `set(...)` / `dict(...)` to deduplicate query results
- `len(list(...))` / `len([...])` over query results
- `if x in <list-from-query>` membership checks (use `.exists()` or `.filter(...).exists()`)
- `sum/min/max/avg` Python builtins over query results
- list slicing `[:N]` / `[N:M]` after a full `.all()`
- pandas operations after `SELECT *`
- nested loops where the inner loop scans a list — almost always a missing JOIN/index lookup

For each hit, propose the pushdown rewrite. If a hit is one of the legitimate exceptions above, call it out and leave it.

## Common pitfalls

- **Slicing a Django queryset is lazy and pushdown** (`qs[:20]` → `LIMIT 20`). **Slicing a list is not** (`list(qs)[:20]` fetches everything first). Don't conflate them.
- **`.count()` is a SQL `COUNT(*)`**, not `len(qs)`. Use it.
- **`.exists()` is a `SELECT 1 ... LIMIT 1`**, not `bool(qs)` or `len(qs) > 0`.
- **`Q` objects** compose complex `WHERE` clauses — use them instead of fetching and filtering.
- **`F` expressions and `Subquery`/`OuterRef`** keep computation in SQL — reach for them before falling back to Python.
- **`prefetch_related(Prefetch(..., queryset=...))`** lets you push filters into the prefetch — don't filter the prefetched list in Python.
- **OpenSearch `query` vs `post_filter`**: filtering happens in the cluster; sort/score does too. Don't sort hits in Python.
- **ClickHouse**: aggregations and window functions are blazing — never pull rows to aggregate in Python.

## Output expectations

When you write or rewrite code under this skill:
1. The data-layer call expresses the full operation (filter + sort + paginate + project).
2. Python code after the call is orchestration only (serialization, response shaping, business rules that genuinely can't be expressed in the source).
3. If you fall back to Python, the code has a one-line comment with the justification, and you flagged it to the user before writing it.
