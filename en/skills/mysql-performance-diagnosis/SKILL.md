---
name: mysql-performance-diagnosis
description: >-
  Generic diagnosis skill for investigating slow MySQL queries and heavy APIs with
  evidence. Covers how to read EXPLAIN / EXPLAIN ANALYZE, hunting the culprit via
  performance_schema, index design principles, and query rewrite patterns. Use when
  investigating "why is it slow", "which index is needed", or "how to read EXPLAIN".
  Use the query-tuning skill for the rewrite policy of this template.
disable-model-invocation: false
---

# MySQL performance diagnosis (EXPLAIN, performance_schema)

Origin: extracted and Japanese/localized from the public skill [jsamuelsen11/claude-config `performance-tuning`](https://github.com/jsamuelsen11/claude-config/tree/main/plugins/ccfg-mysql/skills/performance-tuning), keeping only the parts useful for generic projects.

For rewrite policy (huge-JOIN splitting, nested-view avoidance) use `skills/query-tuning/SKILL.md`; for test execution and performance measurement use `skills/snapshot-test-debugging/SKILL.md`.

## ⚠️ Most important safety note: server config changes are "AI proposes, humans execute"

The source material contains many descriptions of **MySQL server configuration changes** (`my.cnf`, global variables): `innodb_buffer_pool_size` / `innodb_flush_log_at_trx_commit` / `innodb_io_capacity` / `SET GLOBAL ...` and the like. Server-side tuning is sometimes the optimal fix. **In that case do not hesitate — propose the change with evidence (EXPLAIN and performance_schema measurements).** Polluting the code with contorted queries to avoid a config change that is the true optimum defeats the purpose.

However, **executing** server configuration changes on production/verification environments is always done by a human (DBA/admin).

- AI **proposes only**: which setting is the likely cause, recommended values with rationale, trade-offs, and rollback steps — all with evidence.
- **Execution (`SET GLOBAL`, parameter-group changes, `my.cnf` edits, restarts) is done by humans via GUI/manual work.** The AI never performs these CLI operations against production/verification environments.
- For the local development DB, **read-only diagnostic queries (EXPLAIN, SHOW, performance_schema SELECTs) may be run by the AI itself**. For local config-change verification too, present the impact and procedure first.

**Priority**: project-specific skills and rules (e.g., `skills/query-tuning/SKILL.md`) always take precedence. Use this skill's general know-how only where it does not conflict; on conflict, the project side wins.

## 1. Measure first (never fix by guesswork)

- Never start fixing on "the JOIN is probably slow" or "indexes are probably missing". Identify the culprit via EXPLAIN and real measurements (the performance verification procedure in `skills/snapshot-test-debugging/SKILL.md`), then fix.
- Change one thing at a time. After each change, confirm the effect via EXPLAIN/real measurement before moving on.

## 2. How to read EXPLAIN

Start with `EXPLAIN FORMAT=TREE` (execution flow), and `EXPLAIN ANALYZE` (with real timings; MySQL 8.0.18+) when needed. Use `FORMAT=JSON` for cost details.

### Access type `type` (good → bad)

- `const` / `eq_ref`: single-row lookup via PK/unique index (best)
- `ref`: multi-row lookup via a non-unique index (good)
- `range`: index range scan (`BETWEEN` / `<` / `>` / `IN`)
- `index`: full index scan (better than `ALL`)
- `ALL`: **full table scan (worst)**. On a large table, action required

### `Extra` warning signs

- `Using filesort`: ordering not covered by the index; a separate sort runs. → Often solved by a composite index of WHERE columns + ORDER BY columns.
- `Using temporary`: a temp table is created (GROUP BY / DISTINCT etc.). → Often solved by an index on the GROUP BY / DISTINCT columns.
- `Using index`: **covering index** — the table body is never read (good state).

### rows / filtered

- If `rows` (estimated examined rows) is **orders of magnitude** larger than the returned rows, narrowing is not working (classic missing index). Check the actual re-execution counts via `actual rows` and `loops` in `EXPLAIN ANALYZE`.
- `Materialize` / `Union all materialize` in `EXPLAIN FORMAT=TREE` suggests views/subqueries are fully expanded (see the view materialization problem in `skills/query-tuning/SKILL.md`).

## 3. Hunt the culprit via performance_schema (for local investigation)

`performance_schema` collects statistics automatically while running. Enabled on local DBs (`performance_schema=ON`). The following are **read-only diagnostic queries** the AI may run.

```sql
-- Top 10 queries by total execution time
SELECT DIGEST_TEXT AS q, COUNT_STAR AS cnt,
       ROUND(SUM_TIMER_WAIT/1e12,3) AS total_sec,
       ROUND(AVG_TIMER_WAIT/1e12,3) AS avg_sec,
       SUM_ROWS_EXAMINED AS examined, SUM_ROWS_SENT AS sent
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;

-- Top 10 queries with bad examined/sent ratio (= suspected missing indexes)
SELECT DIGEST_TEXT AS q, COUNT_STAR AS cnt,
       SUM_ROWS_EXAMINED AS examined, SUM_ROWS_SENT AS sent,
       ROUND(SUM_ROWS_EXAMINED/NULLIF(SUM_ROWS_SENT,0),1) AS ratio
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_ROWS_SENT > 0
ORDER BY SUM_ROWS_EXAMINED/NULLIF(SUM_ROWS_SENT,0) DESC LIMIT 10;

-- Statements performing full table scans
SELECT * FROM sys.statements_with_full_table_scans
ORDER BY no_index_used_count DESC LIMIT 10;

-- Unused indexes (write-cost waste; deletion candidates)
SELECT * FROM sys.schema_unused_indexes
WHERE object_schema NOT IN ('mysql','performance_schema','sys');

-- Redundant indexes
SELECT * FROM sys.schema_redundant_indexes
WHERE table_schema NOT IN ('mysql','performance_schema','sys');
```

> Note: `performance_schema` statistics are cumulative since startup. For measuring a recent task, inspect digests before/after running the target query, or hit only the target API from a test (the "2. Things you must do before running" section of `skills/snapshot-test-debugging/SKILL.md`).

## 4. Index design principles

- **Composite indexes are leftmost-prefix**: `INDEX(a,b,c)` works for `WHERE a=..` and `WHERE a=.. AND b=..`, but not for `WHERE b=..` alone. Columns are used from the left in order.
- **Range conditions stop subsequent columns**: for `WHERE a=.. AND b>.. AND c=..`, only `a,b` use the index; `c` becomes filter evaluation. Put equality conditions first, range conditions last.
- **Covering indexes**: include WHERE/JOIN columns → ORDER BY columns → SELECT columns in that order in a composite index to skip reading the table body (`Using index`). Only for frequent, performance-critical queries.
- **Beware implicit type conversion**: `WHERE phone = 5551234567` (with `phone` VARCHAR) converts every row to numeric and full-scans. Always match types (`WHERE phone = '5551234567'`).
- **No functions on indexed columns**: `WHERE YEAR(created_at)=2025` cannot use the index. Rewrite as a range: `created_at >= '2025-01-01' AND created_at < '2026-01-01'`.
- **No excessive indexes**: indexes speed up reads but slow down writes. Create only what actual query patterns need, and confirm usage via EXPLAIN.

To add a missing index, **never run `ALTER TABLE` directly**. Always create a migration file and add it through that (same as Step 0 in `skills/query-tuning/SKILL.md`).

## 5. Query rewrite patterns

- **Correlated subquery → JOIN / EXISTS**: `WHERE id IN (SELECT ...)` tends to be re-evaluated per outer row. Rewrite as `JOIN` or `EXISTS` (see also the "Rewrite IN (correlated subquery) into JOIN (derived table)" section of `skills/query-tuning/SKILL.md`).
- **Scalar subqueries in SELECT → LEFT JOIN + aggregation**: avoids per-row subquery execution.
- **OFFSET pagination → keyset (cursor) pagination**: `LIMIT 20 OFFSET 100000` discards the head, so deeper pages get slower. Use `WHERE (created_at,id) < (?,?) ORDER BY ... LIMIT 20`.
- **OR across different columns → UNION [ALL]**: OR across multiple columns cannot be handled by a single index. Split into single-table, single-condition SELECTs and UNION them (mind deduplication). Also effective for authorization-condition rebuilds (tenant / dept / self). See `skills/query-tuning/SKILL.md`.
- **`ANALYZE TABLE` after bulk operations**: after mass INSERT/UPDATE/DELETE, stale statistics derail the optimizer. Refresh with `ANALYZE TABLE <table>` (statistics-only, relatively safe — but production execution is a human decision).

## 6. Scope of this skill and division of labor with other skills

- **This skill**: reading EXPLAIN, culprit hunting via performance_schema, index design principles, general query rewrites (generic MySQL investigation know-how).
- **`skills/query-tuning/SKILL.md`**: project-flavored rewrite policy (full materialization of UNION ALL views, nested views, type-suffixed shared IDs, UNION splitting of authorization conditions, huge-JOIN splitting policy).
- **`skills/snapshot-test-debugging/SKILL.md`**: test execution, leftover-process checks, DB reset decisions, performance measurement, failure analysis.

Chain investigations in this order: identify the culprit with this skill (EXPLAIN / performance_schema) → rewrite per the query-tuning policy → confirm regression and real measurements with snapshot-test-debugging.
