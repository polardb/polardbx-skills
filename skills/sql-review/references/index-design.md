# Index Design Reference

## EXPLAIN type Priority (worst to best)

| type | Meaning | Needs Optimization? |
|------|---------|-------------------|
| ALL | Full table scan | Must optimize |
| index | Full index scan | Usually needs optimization |
| range | Index range scan | Acceptable, depends on row count |
| ref | Non-unique index equality lookup | Good |
| eq_ref | Unique index equality lookup | Excellent |
| const | Primary key / unique index constant lookup | Optimal |

## Composite Index Design Principles

### Leftmost Prefix Rule
Composite index `(a, b, c)` can serve:
- `WHERE a = ?`
- `WHERE a = ? AND b = ?`
- `WHERE a = ? AND b = ? AND c = ?`

Cannot serve:
- `WHERE b = ?` (skips leftmost column)
- `WHERE a = ? AND c = ?` (skips middle column, c cannot use index)

### Column Ordering
1. **Equality columns first**: `WHERE status = ? AND created_at > ?` -> index `(status, created_at)`
2. **Higher selectivity first** (among equality conditions): columns with more distinct values go first
3. **Range columns last**: columns after a range condition cannot use the index
4. **Consider including ORDER BY columns**: avoids filesort

### Covering Index
If a query only needs columns in the index, MySQL can return data directly from the index (Extra: Using index), avoiding table lookups.
For high-frequency queries, consider adding SELECT columns to the index.

## Index Exploration Method

For each problematic query (type=ALL, filesort, excessive rows), don't just try one index — enumerate multiple candidate index combinations and verify each one empirically via EXPLAIN.

### Candidate Generation

Based on the query's WHERE / JOIN / ORDER BY / GROUP BY columns, generate candidates by varying:
- **Column subsets**: not all WHERE columns need to be in the index
- **Column orderings**: equality columns first, range columns last, ORDER BY column at the tail
- **Inclusion of ORDER BY**: adding the sort column can eliminate filesort

### Verify-and-Compare Loop

Test each candidate on the live test instance using a CREATE → EXPLAIN → DROP cycle:

```bash
# Create candidate A
$MYSQL -e "CREATE INDEX idx_candidate_a ON t (col1, col2)"
$MYSQL -e "EXPLAIN SELECT ..."   # or EXPLAIN EXECUTE for Enterprise Edition
$MYSQL -e "DROP INDEX idx_candidate_a ON t"

# Create candidate B
$MYSQL -e "CREATE INDEX idx_candidate_b ON t (col1, col2, col3)"
$MYSQL -e "EXPLAIN SELECT ..."
$MYSQL -e "DROP INDEX idx_candidate_b ON t"
```

Compare `type` / `rows` / `Extra` across all candidates. Select the best and preserve the full exploration process in the report so users can see the decision rationale.

### Cross-Variant Validation

When a query has multiple execution path variants (e.g., from dynamic WHERE conditions), test each candidate index against ALL variants of the same query. If no single index covers all variants, recommend multiple indexes with a note on write overhead trade-off (see Pattern 7).

## Common Problem Patterns

### Pattern 1: No Index on WHERE Clause
```sql
-- Problem: full table scan
SELECT * FROM orders WHERE user_id = ?
-- Recommendation
CREATE INDEX idx_user_id ON orders(user_id)
```

### Pattern 2: Partial Hit on Compound Conditions
```sql
-- Problem: only status index used, created_at not indexed
SELECT * FROM tasks WHERE status = ? AND created_at > ?
-- Recommendation: composite index
CREATE INDEX idx_status_created ON tasks(status, created_at)
```

### Pattern 3: ORDER BY Causing filesort
```sql
-- Problem: Using filesort
SELECT * FROM logs WHERE level = 'ERROR' ORDER BY created_at DESC LIMIT 20
-- Recommendation: index covers both WHERE and ORDER BY
CREATE INDEX idx_level_created ON logs(level, created_at)
```

### Pattern 4: No Index on JOIN Columns
```sql
-- Problem: driven table full scan
SELECT * FROM orders o JOIN users u ON o.user_id = u.id WHERE o.status = ?
-- Recommendation: ensure JOIN columns have indexes
CREATE INDEX idx_user_id ON orders(user_id)
```

### Pattern 5: GROUP BY Causing Temporary Table
```sql
-- Problem: Using temporary; Using filesort
SELECT department, COUNT(*) FROM employees GROUP BY department
-- Recommendation
CREATE INDEX idx_department ON employees(department)
```

### Pattern 6: Function Wrapping Invalidates Index
```sql
-- Problem: LEFT(created_at, 10) prevents using created_at index
SELECT LEFT(created_at, 10) AS date, COUNT(*) FROM page_views GROUP BY LEFT(created_at, 10)
-- Recommendation: add a redundant date column + index, or use a virtual column
```

### Pattern 7: Dynamic WHERE Needs Multiple Indexes

When the same query has multiple execution paths (e.g., MyBatis `<if>` conditions), different paths may need different indexes:

```sql
-- Path A (user view): identity param present, highly selective
SELECT * FROM orders WHERE user_id = ? AND order_status = ? AND is_deleted = 0
  ORDER BY create_time DESC LIMIT 20
-- Best index: (user_id, order_status, is_deleted, create_time)

-- Path B (admin view): no identity param, broad scan
SELECT * FROM orders WHERE is_deleted = 0
  ORDER BY create_time DESC LIMIT 20
-- Index from Path A is useless here (leftmost column user_id not in WHERE)
-- Needs: (is_deleted, create_time) or (create_time) for ORDER BY optimization
```

A single composite index `(user_id, order_status, is_deleted, create_time)` only serves Path A. Path B still does a full table scan + filesort. The solution is two indexes:

1. `(user_id, order_status, is_deleted, create_time)` — for user-scoped queries
2. `(is_deleted, create_time)` — for admin/unscoped queries

**Trade-off**: Multiple indexes increase write overhead (each INSERT/UPDATE must maintain both indexes). Decision criteria:
- If both paths are high-frequency in production, both indexes are justified
- If one path is rare (e.g., admin view used only by internal tools), the index for the rare path may be lower priority
- Assess via: method call sites, access logs, or business logic analysis

## Redundant Index Detection

When index A is a prefix of index B, A is redundant:
- `idx_a(a)` and `idx_ab(a, b)` -> `idx_a` is redundant
- `idx_a(a)` and `PRIMARY KEY(a)` -> `idx_a` is redundant

But these are NOT redundant:
- `idx_a(a)` and `idx_ba(b, a)` -> different leftmost columns
- `UNIQUE(a)` and `INDEX(a, b)` -> UNIQUE has constraint semantics

## Mock Data Generation Rules

### Field Type Mapping
| Field Type | Generation Strategy |
|-----------|-------------------|
| VARCHAR(64) PRIMARY KEY | nanoid/UUID format random string |
| INT AUTO_INCREMENT | Sequential increment |
| VARCHAR status field | Random selection from enum values extracted from code |
| VARCHAR(45) IP address | Random IP in `192.168.x.x` format |
| VARCHAR(32) timestamp | ISO 8601 format, random within last 30 days |
| DECIMAL amount | Random between 100.00 - 99999.99 |
| TEXT | Random string of 50-200 characters |
| TINYINT(1) | Random 0 or 1 |
| INT counter | Random 0-100 |

### Data Volume Inference
| Clue | Inferred Volume |
|------|----------------|
| Config table (singleton, id=1) | 1-5 rows |
| User table | 1K-5K rows |
| Association/detail table | 3-10x the parent table |
| Log/transaction table | 10K-100K rows |
| Has LIMIT N OFFSET M with large M | At least M*2 rows |
| ON DUPLICATE KEY UPDATE | High concurrency write scenario, at least 10K rows |

### Foreign Key Relationships
Ensure referential integrity when generating related data:
1. Generate referenced tables first (e.g. users)
2. Then generate referencing tables (e.g. orders), picking foreign key values from the referenced table
3. Use `SELECT id FROM parent_table ORDER BY RAND() LIMIT 1` to get sample IDs

## PolarDB-X Considerations

- PolarDB-X EXPLAIN output format is MySQL-compatible
- Cross-shard queries in distributed environments are more expensive; full table scans have greater impact
- GSI (Global Secondary Index) can accelerate cross-shard queries
- Watch for `MERGE SORT` and `GATHER` operators in EXPLAIN output
