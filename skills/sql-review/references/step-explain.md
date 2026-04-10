# Step 5: EXPLAIN Analysis

> **Before starting, Read `$TMPDIR/sql_review_name_mapping.md`** to confirm the mapping. All SQL sent to the instance uses anonymous names.

> **Enterprise Edition note**: If the user chose **PolarDB-X Enterprise Edition** in Step 0, use `EXPLAIN EXECUTE` instead of `EXPLAIN` throughout this step.

## Table-by-Table Processing

**Read `$TMPDIR/sql_review_progress.md`** to get the list of tables and their queries. Process **one table at a time**:

For each table:

### 1. Select Queries to Analyze

Read the table's query list from the progress file. Skip:
- Pure INSERTs (no WHERE clause)
- Primary-key-only lookups (`WHERE pk = ?`)
- `insertSelective` / `updateByPrimaryKeySelective` (PK-based DML)

Mark skipped queries in the progress file.

### 2. Run Baseline EXPLAIN

For each non-skipped query (and each variant of multi-path queries):

```bash
$MYSQL -e "EXPLAIN SELECT ... FROM t_xxx WHERE ..."
```

Use sample values from the table (`SELECT col FROM t_xxx LIMIT 1`) to avoid empty-result false positives.

**Key metrics**:
- `type`: ALL / index → needs an index
- `possible_keys` / `key`: NULL → no candidates
- `rows`: higher = more urgent
- `Extra`: Using filesort / Using temporary → needs attention

**JOIN notes**: Check driver table selection and driven table access method. LEFT JOIN with type=ALL on driven table is critical.

### 3. Index Exploration

For problematic queries (type=ALL, filesort, excessive rows), design candidate indexes and verify via CREATE → EXPLAIN → DROP cycle. See [index-design.md](index-design.md) "Index Exploration Method" for the full procedure.

**Cross-variant validation (MyBatis)**: Test candidate indexes against ALL variants of multi-path queries. If no single index covers all variants, recommend multiple indexes (see [index-design.md](index-design.md) Pattern 7).

### 4. Save to Progress File

**Immediately** after completing a table, append its EXPLAIN results and index recommendations to `$TMPDIR/sql_review_progress.md`:

```markdown
#### EXPLAIN Results: <table_name>
| # | Variant | Baseline type/rows | Recommended Index | Post-Index type/rows |
|---|---------|-------------------|-------------------|---------------------|
| Q1 | Full | ALL / 19932 | idx_xxx (col1, col2) | ref / 1 |
| Q1b | Admin | ALL / 19932 | idx_yyy (col3, col4) | range / 1343 |
| Q2 | — | skipped (PK) | — | — |

Recommended DDL:
- `CREATE INDEX idx_xxx ON table(col1, col2);`
- `CREATE INDEX idx_yyy ON table(col3, col4);`
```

Then **proceed to the next table**. Do not batch multiple tables.

> **Why table-by-table**: EXPLAIN results for a single table are self-contained. Saving after each table means that if context is compressed, the agent can re-read the progress file and skip tables already analyzed. Each table typically needs 2-10 EXPLAIN runs, keeping each iteration manageable.

## Completion Check

After all tables are processed, verify against the progress file that every query (and every variant) has been analyzed or marked as skipped.
