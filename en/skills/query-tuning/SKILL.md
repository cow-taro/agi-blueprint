---
name: query-tuning
description: >-
  Policy for speeding up slow MySQL queries. Analyze one huge JOIN query, split it
  into several small queries, and join them on the application side. Use for slow
  APIs, slow page loads, performance improvements, and EXPLAIN analysis. Use the
  mysql-performance-diagnosis skill for diagnosis (EXPLAIN reading, culprit hunting).
disable-model-invocation: false
---

# Query tuning policy: splitting huge JOINs

## Basic policy

**Never try to solve everything with one huge JOIN query.** When you find a slow query, first analyze what it does, split it into several small queries in meaningful units (tables/conditions), and join them on the application side.

Reasons:

1. **Small queries let indexes work.** Huge JOINs inflate intermediate results and disable indexes. Split queries use indexes naturally.
2. **The slow part is isolated reliably.** After splitting, "which query is slow" becomes clear per query — enabling pinpoint improvements like dedicated indexes or WHERE revisions, instead of the unactionable "the JOIN is somehow slow".
3. **Narrow down step by step.** Feeding the previous query's results (ID lists) into the next query lets it run against a small dataset from the start.

## Important: put filter conditions in the WHERE clause of standalone queries, not in JOIN ON clauses

Views containing `UNION ALL` cannot be merged by MySQL; every reference **materializes the entire view from scratch**.

- ❌ No effect: narrowing with a **JOIN condition (ON clause)** like `LEFT JOIN SummaryView SV ON t.id = SV.id` inside a huge JOIN does not push the condition into the view — the whole view is always materialized
- ✅ Works: putting the condition in the **direct WHERE clause (literal value) of a standalone query**, like `SELECT * FROM SummaryView WHERE id = %(id)s`, pushes it down to each UNION ALL branch — dramatically faster (measured example: 30x+ speedup under the same condition)

When splitting one big JOIN query, issue each table/view as its own query and write narrowing conditions in that query's own WHERE clause as literal values. After splitting, assemble a dictionary (keyed by ID) and join on the application side.

## Pitfall: when views are nested, the condition never reaches the second level

If a view internally JOINs another view (nested views), a `WHERE` on the outer view reaches the outer view's own UNION ALL branches but not the inner view being JOINed. The outer view's standalone query stays slow.

In that case, bypass the views: query the base tables directly and rewrite the necessary JOINs as application-side split queries. Always check the view definitions for internal references to other views.

## Before starting, always check other call sites

Before tuning an API/DB method, always check whether it is called from places other than the page under analysis.

- The same DB method may be called from several endpoints with different parameter/filter combinations.
- Procedure: search for the DB method name to enumerate all calling endpoints, then search the frontend for each endpoint to confirm which screen passes which parameters and how.
- Policy: rewriting a shared method while dragging in the generic filter features other callers require widens the impact and breaks easily. Rigorously create tests for every parameter combination from every related page, and satisfy all conditions (implement exactly the same processing and return exactly the same response under every condition).
- When designing tests, likewise enumerate all callers and all parameter patterns of the target API first.

## Work procedure

**Step 0 (mandatory, always first): index verification**

Before starting query tuning, check the indexes of every table the query touches via `SHOW INDEX FROM <table>`. If columns used in WHERE, JOIN conditions, or ORDER BY lack indexes, add them first. If index additions alone solve it, that is optimal. View bypasses and splits come only when it is still slow with correct indexes in place.

```sql
SHOW INDEX FROM <table1>;
SHOW INDEX FROM <table2>;
```

- If indexes are missing, **never run `ALTER TABLE` directly**. Always create a migration file and add the index through it. Match the location and format to existing files.
- Never suppress MySQL command output (`2>/dev/null` etc. are forbidden — see `skills/snapshot-test-debugging/SKILL.md`, the "0. Never suppress command output (absolute)" section).

1. Check the slow query with `EXPLAIN FORMAT=TREE`: look for `Materialize` / `Union all materialize`, and where rows get narrowed (reading guide in `skills/mysql-performance-diagnosis/SKILL.md`).
2. Enumerate the tables/views the query references; check the view definitions for nesting.
3. Split into standalone queries in meaningful units (tables/conditions), writing narrowing conditions as literal values in each query's own WHERE clause.
4. After splitting, measure each query (e.g., with `time.time()`) and confirm the expected speedup. Any still-slow query is now the isolated "slow part" — apply index additions or WHERE revisions to it alone.
5. Assemble the results into dictionaries on the application side and join them without changing the original response shape (field composition).

## Rewrite IN (correlated subquery) into JOIN (derived table)

`WHERE id IN (SELECT ...)` can make MySQL build a correlated-subquery plan that re-evaluates the subquery per outer row — effectively indexless, near a full scan.

- ❌ Weak: `WHERE C.id IN (SELECT id FROM Customer WHERE ownerId IN (SELECT userId FROM Account WHERE tenantId = %(tenantId)s))`
- ✅ Works: **JOIN it as a derived table**, like `JOIN (SELECT id FROM Customer WHERE ownerId IN (...)) CAUTH ON CAUTH.id = C.id`. The derived table materializes once, after which the join key uses indexes naturally.

For patterns like "first compute the set of authorized IDs, then JOIN it into the main body", prefer `JOIN` over `IN`.

## Authorization conditions OR-ing across multiple tables defeat indexes

Conditions OR-ing columns of different tables (including LEFT JOIN targets), like `(A.tenantId = %(tenantId)s OR A2.tenantId = %(tenantId)s OR C.tenantId = %(tenantId)s)`, cannot be evaluated efficiently by single-table single-column indexes (index_merge fails; the whole table is scanned and Filter-evaluated).

Countermeasure: split such OR conditions into a **UNION of simple per-table equality conditions** (set operations on IDs).

```sql
-- ❌ Multi-table OR that tends to full-scan
SELECT ... FROM Customer C
LEFT JOIN Account A ON C.ownerId = A.userId
LEFT JOIN Account A2 ON C.subOwnerId = A2.userId
WHERE C.tenantId = %(tenantId)s OR A.tenantId = %(tenantId)s OR A2.tenantId = %(tenantId)s

-- ✅ Per-table UNION so indexes are used
SELECT id FROM Customer WHERE tenantId = %(tenantId)s
UNION
SELECT id FROM Customer WHERE ownerId IN (SELECT userId FROM Account WHERE tenantId = %(tenantId)s)
UNION
SELECT id FROM Customer WHERE subOwnerId IN (SELECT userId FROM Account WHERE tenantId = %(tenantId)s)
```

Each UNION branch is single-table/single-condition, so each column's plain index works as is. Especially effective when rebuilding authorization conditions (tenant / department / self access control).

## Columns whose shared IDs carry a type suffix must be joined with the type preserved

For columns where multiple entity types hold separate auto-increment ID spaces yet the **same numeric id can collide by accident** (suffixed shared IDs like `concat(id, '-', type)`), **never re-JOIN on the bare numeric id with the suffix dropped**.

- ❌ Dangerous: `... JOIN Customer CL ON C.customerId = CL.id` (`C.customerId` already lost the type info) → if `C` actually originated from another entity type, it resolves to an **unrelated record that happens to share the numeric id** (possibly in another organization — a data-exposure risk across the authorization boundary).
- ✅ Safe: re-compose with the suffix every time, like `... JOIN Customer CL ON F.parentSerialRefId = concat(CL.id, '-', '0')`. A type mismatch can never match.

When you find such columns, always verify type-safe joining even in existing code.

## Staged approach (indexes only → rebuild only if insufficient)

If Step 0 (index verification) does not solve it, do not rewrite the whole query at once — verify in stages:

1. **Stage 1**: prepare migration files for the missing indexes.
2. **Stage 2**: keep the query structure, apply only the indexes, and measure. Confirm via `EXPLAIN FORMAT=TREE` that the index is actually used and no `Table scan` remains. **If index additions alone are enough, you may stop here** (rebuilding is unnecessary complexity; do it only when measured benefit is confirmed).
3. **Stage 3**: only if Stage 2 is still slow, rebuild the query itself (apply this skill's patterns). After rebuilding, always run the next section's verification.

## Verification when rebuilding queries containing authorization conditions (B2B authorization boundary)

In multi-tenant B2B apps, the set of rows a query returns is itself the permission boundary. When rebuilding queries with authorization conditions (WHERE/JOIN on tenantId / departmentId / userId), add — on top of normal golden regression tests — a direct set-comparison of **the IDs returned by old vs. new queries against real data**. Golden regression covers only a few representative orgs/departments/users; comparing full real-data sets across many orgs/departments/users detects authorization gaps the test cases happened to miss.

Procedure:

1. Write out the pre-rebuild query (restored via `git show <commit>^:<path>` etc.) and the post-rebuild query in ID-set-only form.
2. For several real tenantIds (from largest to smallest counts), departments, and userIds, run both and check:
   ```sql
   SELECT COUNT(*) FROM new_ids WHERE refId NOT IN (SELECT refId FROM old_ids); -- suspected unauthorized exposure
   SELECT COUNT(*) FROM old_ids WHERE refId NOT IN (SELECT refId FROM new_ids); -- regression
   ```
3. Adopt only after both are 0. If any diff appears, do not adopt until the cause is identified.
4. Also check the authorization-anchor columns for duplicate rows or anomalies (multiple active accounts per userId, etc.) — this closes cases that accidental coincidences would hide.

## Cautions when applying to INSERT/UPDATE

If you ever hit slow write queries, do not casually split them into multiple queries like read queries. Splitting loses atomicity — a mid-flight failure can corrupt data consistency. When splitting writes, always wrap them in transactions (`BEGIN`/`COMMIT`, or ORM-level transaction control) to preserve consistency.
