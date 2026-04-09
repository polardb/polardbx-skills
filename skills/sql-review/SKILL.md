---
name: sql-review
description: |
  Scan codebase for table schemas (DDL) and SQL statements, create a PolarDB-X test instance with mock data, analyze index usage via EXPLAIN, and provide index optimization recommendations. Works with any language/framework.
  Use when user says "sql review", "SQL审查", "索引分析", "索引推荐", "index review", "SQL优化", "检查索引", "review sql", "分析SQL", "explain分析", "慢查询分析", "sql performance".
metadata:
  version: 0.2.0
---

# SQL Review - Index Analysis & Recommendations

Scan codebase for DDL and SQL queries, create tables on a PolarDB-X test instance, populate mock data, analyze index usage via EXPLAIN, and output optimization recommendations.

## Prerequisites

- `curl` must be available (for PolarDB-X Zero API calls to create/destroy instances)
- `mysql` CLI client must be available (to connect and execute SQL)
- If `mysql` is not available, prompt the user to install it (brew install mysql-client / apt install mysql-client / scoop install mysql)
- All temporary files use the `sql_review_` prefix. `$TMPDIR` in this document refers to a writable temp directory — determine the appropriate path for the current OS (e.g. `/tmp` on Linux/macOS, `%TEMP%` on Windows)

## Workflow

Track progress on the following steps:

### Step 0: Confirm Scope and Database Edition

Ask the user **both** questions simultaneously:

**Question 1: Scan Scope**

| Mode | Description |
|------|-------------|
| **Full repository** | Scan all SQL in the entire codebase, ideal for first-time review or full audit |
| **Specific module/directory** | Only scan a user-specified directory (e.g. `src/services/`), ideal for targeted analysis |
| **Recent Git commits** | Only scan SQL in files changed in the last N days (or N commits), ideal for incremental review |

**Question 2: Database Edition**

| Option | Test Instance |
|--------|--------------|
| **Community MySQL** | Use PolarDB-X Standard Edition (MySQL-compatible) |
| **PolarDB-X Standard Edition** | Use PolarDB-X Standard Edition |
| **PolarDB-X Enterprise Edition** | Use PolarDB-X Enterprise Edition (supports partitioned tables, GSI, etc.) |

Based on user selection:

- **Scan scope**:
  - Full repository: Grep searches from project root
  - Specific module: Grep searches within the user-specified directory
  - Git commits: First run `git log --since="N days ago" --name-only --diff-filter=ACMR` or `git log -N --name-only --diff-filter=ACMR` to get changed file list, Step 1 searches only within these files. DDL must still be searched repo-wide (changed queries may reference unchanged tables)
- **Database edition**: Record the user's choice, pass the corresponding `edition` parameter when creating the instance in Step 3

### Step 1: Scan Codebase

**Goal**: Collect all DDL and SQL statements within the scope defined in Step 0, produce a complete SQL inventory.

**Important**: Use multi-round searches with cross-validation. Do not rely on a single search pass.

#### 1.1 Multi-round Search

Within the scope defined in Step 0, search for the following patterns (excluding node_modules, dist, .git, etc.):

| Round | Pattern | Purpose |
|-------|---------|---------|
| 1 | `CREATE TABLE` | DDL statements |
| 2 | `CREATE INDEX` / `ALTER TABLE` | Index definitions, migrations |
| 3 | `SELECT\s+.*\s+FROM` | SELECT queries |
| 4 | `\bJOIN\b` | **JOIN queries (easily missed, must search separately)** |
| 5 | `INSERT INTO` | INSERT statements |
| 6 | `\bUPDATE\b.*\bSET\b` | UPDATE statements |
| 7 | `DELETE FROM` | DELETE statements |
| 8 | `IN\s*\(\s*SELECT` | Subqueries |
| 9 | `GROUP BY` / `ORDER BY` | Sorting/aggregation |

For each round, use Read to view surrounding context (10 lines before/after) and extract the complete SQL (SQL in code often spans multiple lines).

#### 1.2 Produce SQL Inventory

Output two structured tables:

- **DDL inventory**: `| # | Table | Type | File | Line | Full SQL |`
- **Query inventory**: `| # | Type | Tables | File | Line | Full SQL | Key Features |`

**Key Features** column must note: JOIN / LEFT JOIN / subquery / GROUP BY / ORDER BY / LIMIT / HAVING, etc.

#### 1.3 Cross-validation

- JOIN count >= Grep `\bJOIN\b` hit count
- Every table in DDL should appear in queries
- All queries involving 2+ tables must have Key Features annotated

### Step 1.5: Name Anonymization

**Purpose**: Prevent business table/column names from leaking to the test instance. All SQL sent to the test instance uses randomized names only.

#### 1.5.1 Build Mapping

Generate random aliases for all table and column names from the Step 1 inventory:

| Type | Rule | Example |
|------|------|---------|
| Table | `t_` + 4 random alphanumeric chars | `orders` -> `t_a3kf` |
| Column | `c_` + 4 random alphanumeric chars | `user_id` -> `c_m9x2` |
| Primary key | Always `c_pk` | `id` -> `c_pk` |
| Index | `idx_` + 4 random chars | `idx_status_created` -> `idx_r4k1` |

The same column name across different tables uses the same anonymous name. Ensure no collisions.

#### 1.5.2 Persist Mapping File

Save the mapping to `$TMPDIR/sql_review_name_mapping.md`:

```markdown
# SQL Review Name Mapping

## Tables
| Real Name | Anon Name |
|-----------|-----------|
| orders    | t_a3kf    |
| users     | t_b7mq    |

## Columns
| Real Name   | Anon Name |
|-------------|-----------|
| id          | c_pk      |
| user_id     | c_m9x2    |
| created_at  | c_p4wn    |
```

> **Critical**: In subsequent steps (Step 4/5/6), **always re-read this file** when the mapping is needed. Do NOT rely on conversation context memory. This prevents mapping loss in long sessions where context may be truncated.

#### 1.5.3 Transform SQL

Replace all table/column/index names in DDL and queries with anonymous names. Preserve data types and constraints (NOT NULL, UNIQUE, DEFAULT, etc.) unchanged.

The transformed SQL is used for all subsequent interactions with the test instance.

### Step 2: Analyze Data Characteristics

Infer data volume and distribution for each table from code:

- LIMIT/OFFSET -> table has at least thousands of rows; ID generation -> random/sequential distribution
- Time field range queries -> continuous growth; status enums / UNIQUE constraints -> field cardinality
- INSERT frequency (fire-and-forget -> high write volume)

Determine mock data volume: config tables 10-50 rows, business tables 5K-50K rows, log tables 100K+ rows.

### Step 3: Create Test Instance

Based on user's database edition choice from Step 0:
- Community MySQL / PolarDB-X Standard -> `"edition": "standard"`
- PolarDB-X Enterprise -> `"edition": "enterprise"`

```bash
curl -s -X POST https://zero.polardbx.com/api/v1/instances \
  -H 'Content-Type: application/json' \
  -d '{"tag": "sql-review", "ttlMinutes": 60, "edition": "<standard|enterprise>"}'
```

Extract `host`, `port`, `username`, `password`, `instanceId` from the response. Create a MySQL option file to avoid exposing secrets in shell commands:

```ini
; $TMPDIR/sql_review_my.cnf
[client]
host=<host>
port=<port>
user=<username>
password=<password>
```

Verify connection and create database:

```bash
mysql --defaults-extra-file=$TMPDIR/sql_review_my.cnf -e "CREATE DATABASE IF NOT EXISTS sql_review_db"
```

> In subsequent steps, `$MYSQL` refers to `mysql --defaults-extra-file=$TMPDIR/sql_review_my.cnf sql_review_db`.

### Step 4: Create Tables and Populate Mock Data

> All SQL in this step uses the anonymized table/column names from Step 1.5.

1. **Create tables**: Write anonymized DDL to a temp .sql file, execute with `$MYSQL < $TMPDIR/sql_review_ddl.sql`.

2. **Populate data**: Use stored procedures for bulk generation. Procedure names also use anonymous names (e.g. `gen_t_a3kf_data`). Write to .sql file and execute (`DELIMITER` only works in file mode):
   ```sql
   DROP PROCEDURE IF EXISTS gen_t_a3kf_data;
   DELIMITER //
   CREATE PROCEDURE gen_t_a3kf_data()
   BEGIN
     DECLARE i INT DEFAULT 0;
     WHILE i < 10000 DO
       INSERT INTO t_a3kf (...) VALUES (...);
       SET i = i + 1;
     END WHILE;
   END //
   DELIMITER ;
   CALL gen_t_a3kf_data();
   ```

3. **Mock data guidelines**: Field values should match enums/formats from code, time fields use random timestamps within recent N days, foreign keys reference existing IDs.

4. **Verify**: `$MYSQL -e "SELECT 't_a3kf' AS t, COUNT(*) FROM t_a3kf UNION ALL ..."`

### Step 5: EXPLAIN Analysis

> **Before starting, Read `$TMPDIR/sql_review_name_mapping.md`** to confirm the mapping. All SQL sent to the instance uses anonymous names.

Analyze each query from the Step 1 inventory by number. Skip pure INSERTs and primary-key-only lookups (mark as "skipped").

```bash
# Fetch sample values first (to avoid "no matching row" false results)
$MYSQL -e "SELECT c_pk FROM t_a3kf LIMIT 1"
# Then EXPLAIN (using anonymized query)
$MYSQL -e "EXPLAIN SELECT * FROM t_a3kf WHERE c_pk = 'sample_value'"
```

**JOIN notes**: EXPLAIN outputs multiple rows; check driver table selection and driven table access method. LEFT JOIN driven table with type=ALL is a serious issue. JOIN condition columns must have indexes.

**Key metrics**:
- `type`: ALL / index -> needs an index
- `possible_keys` / `key`: NULL -> no candidates or not used
- `rows`: higher = more urgently needs optimization
- `Extra`: Using filesort / Using temporary -> needs attention

**Index exploration**: For each problematic query (type=ALL, filesort, excessive rows), don't just try one index — enumerate multiple candidate index combinations based on columns in WHERE / JOIN / ORDER BY / GROUP BY (different column subsets, different column orderings), and verify each one. Candidate indexes also use anonymous names:

```bash
# Create candidate A (anonymous names)
$MYSQL -e "CREATE INDEX idx_r4k1 ON t_a3kf (c_4fp1, c_8bq5)"
$MYSQL -e "EXPLAIN SELECT ..."
# Record results, then DROP and try next
$MYSQL -e "DROP INDEX idx_r4k1 ON t_a3kf"

# Create candidate B
$MYSQL -e "CREATE INDEX idx_w7n2 ON t_a3kf (c_4fp1, c_m9x2, c_8bq5)"
$MYSQL -e "EXPLAIN SELECT ..."
$MYSQL -e "DROP INDEX idx_w7n2 ON t_a3kf"
```

Compare type / rows / Extra across all candidates, select the best and write to report. Preserve the exploration process in the report (EXPLAIN comparison of each candidate) so users can see the decision rationale.

After completion, verify against the inventory that every query has been analyzed or marked as skipped.

### Step 6: Output Report

> **Before generating the report, Read `$TMPDIR/sql_review_name_mapping.md`** and map all anonymous names back to real names. All table names, column names, and index names in the report must use real business names. Recommended index DDL should be directly executable in production.

**Must write report to file** `sql-review-report.md` (project root), and output a summary in the conversation.

Report contains 7 sections:

1. **Overview** — Project info, test environment, key findings (queries needing optimization, recommended indexes, most critical issue)
2. **Table Schema Overview** — Column count, existing indexes, mock data volume per table
3. **SQL Inventory** — Reuse DDL and query inventories from Step 1
4. **Query Analysis** — Grouped by source file, each query with SQL, EXPLAIN results, issues and recommendations. Queries needing optimization show the index exploration process (EXPLAIN comparison of each candidate). Normal ✅ / Needs optimization ⚠️
5. **Index Recommendations Summary** — Final recommended indexes (with DDL and selection rationale), redundant indexes, optimization impact overview
6. **Security Risk Notes** — SQL injection, dynamic concatenation, etc.
7. **Caveats** — Mock vs production data distribution differences, composite index column ordering rationale, write overhead assessment

In the conversation, only output "Key Findings" and "Index Recommendations Summary", and inform the user that the full report is in `sql-review-report.md`.

Full report format example in [references/examples.md](references/examples.md).

### Step 7: Cleanup

Ask the user:

| Option | Description |
|--------|-------------|
| **Keep instance, let it auto-expire** | Instance will be automatically destroyed when TTL expires; user can connect manually to verify index effects in the meantime |
| **Destroy immediately** | Release the instance now |

Output instance connection info for the user's reference: `host`, `port`, `username`, remaining TTL. Remind user that credentials are stored in `$TMPDIR/sql_review_my.cnf`.

If user chooses to keep the instance, also output the name mapping table for reference (remind the user that table/column names on the instance are anonymized).

If user chooses to destroy immediately:

```bash
curl -s -X DELETE https://zero.polardbx.com/api/v1/instances/<instance_id>
```

Regardless of choice, delete all temporary files at the end: `$TMPDIR/sql_review_my.cnf`, `$TMPDIR/sql_review_ddl.sql`, `$TMPDIR/sql_review_*.sql`, `$TMPDIR/sql_review_name_mapping.md`.

## Index Design Principles

Detailed index design rules in [references/index-design.md](references/index-design.md).

## Examples

Complete analysis example in [references/examples.md](references/examples.md).
