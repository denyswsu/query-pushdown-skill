# query-pushdown

A Claude Code skill that enforces a simple rule: **push data operations down to the system that owns the data.** Sorting, filtering, searching, aggregation, pagination, deduplication, and grouping should happen in the database, search engine, cache, or upstream API — not in a Python loop after `SELECT *`.

## Why

When data work happens in Python instead of being pushed down:

- Every row crosses the network, even rows that get discarded.
- The full result set lives in the app's RAM.
- Python loops run 10–1000× slower than indexed database operations.
- Source-side semantics (collation, NULL handling, locale, full-text scoring) get silently re-implemented and silently diverge.
- Pagination becomes a lie: `list(qs)[:20]` materialized millions of rows to return twenty.

The data layer was built for this. Use it.

## What the skill does

When triggered, this skill makes Claude:

1. **Default to pushdown.** New code expresses filtering, sorting, search, aggregation, and pagination in the data source's native API — Django ORM, raw SQL, ClickHouse, OpenSearch/Elasticsearch, Redis, MongoDB, BigQuery, pandas-on-SQL, S3 list operations, paginated REST/GraphQL APIs, gRPC, Kafka, etc.
2. **Refactor red flags.** During review/refactor tasks, Claude scans for `sorted(qs, ...)`, `[x for x in qs if ...]`, `set(...)` over query results, `len([...])`, list slicing after `.all()`, pandas operations after `SELECT *`, and similar anti-patterns — then proposes the pushdown rewrite.
3. **Ask permission before falling back.** If pushdown is genuinely blocked (source can't express the predicate, cross-source join, tiny in-memory collection, etc.), Claude stops, explains why, and asks before doing the operation in Python. No silent fallbacks.

## When it triggers

The skill description is intentionally broad. It fires whenever code touches:

- A Django queryset, SQL query, or any ORM call
- A search/cache/queue client (OpenSearch, Elasticsearch, Redis, Mongo, Kafka)
- A dataframe or list of records fetched from any external system
- A Python loop, comprehension, `sorted()`, `filter()`, `set()`, `max()`, `sum()`, etc., operating on data that came from somewhere else
- User requests to "add a filter", "add sorting", "search", "paginate", "count", "deduplicate", "group", or "aggregate" — even when the database is not explicitly mentioned

## Install

Via the [skills CLI](https://skills.sh):

```bash
npx skills add denyswsu/query-pushdown-skill@query-pushdown
```

Or clone and symlink manually:

```bash
git clone git@github.com:denyswsu/query-pushdown-skill.git
ln -s "$PWD/query-pushdown-skill/query-pushdown" ~/.claude/skills/query-pushdown
```

## Scope

Universal. Not tied to any framework or stack — the principle applies the same way whether the source is PostgreSQL, ClickHouse, OpenSearch, S3, or a paginated third-party API.

## License

MIT
