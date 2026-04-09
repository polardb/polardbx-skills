---
name: sql-review
description: |
  Scan codebase for table schemas (DDL) and SQL statements, create a PolarDB-X test instance with mock data, analyze index usage via EXPLAIN, and provide index optimization recommendations. Works with any language/framework.
  Use when user says "sql review", "SQL审查", "索引分析", "索引推荐", "index review", "SQL优化", "检查索引", "review sql", "分析SQL", "explain分析", "慢查询分析", "sql performance".
metadata:
  version: 0.1.0
---

# SQL Review - Index Analysis & Recommendations

Scan codebase for DDL and SQL queries, create tables on a PolarDB-X test instance, populate mock data, analyze index usage via EXPLAIN, and output optimization recommendations.

## Prerequisites

- `curl` must be available (for PolarDB-X Zero API calls to create/destroy instances)
- `mysql` CLI client must be available (to connect and execute SQL)
- If `mysql` is not available, prompt the user to install it (brew install mysql-client / apt install mysql-client / scoop install mysql)

## Workflow

Use TodoWrite to track the following steps:

### Step 0: Confirm Scope and Database Edition

Use AskUserQuestion to ask **both** questions simultaneously:

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

**Important**: Use the Grep tool for multi-round searches with cross-validation. Do not rely on a single agent search.

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

Extract `host`, `port`, `username`, `password`, `instanceId` from the response. Verify connection and create database:

```bash
mysql -h <host> -P <port> -u <user> -p<password> -e "CREATE DATABASE IF NOT EXISTS sql_review_db"
```

> In subsequent steps, `$MYSQL` refers to `mysql -h <host> -P <port> -u <user> -p<password> sql_review_db`.

### Step 4: Create Tables and Populate Mock Data

1. **Create tables**: Write DDL to a temp .sql file, execute with `$MYSQL < /tmp/sql_review_ddl.sql`.

2. **Populate data**: Use stored procedures for bulk generation, write to .sql file and execute (`DELIMITER` only works in file mode):
   ```sql
   DROP PROCEDURE IF EXISTS gen_xxx_data;
   DELIMITER //
   CREATE PROCEDURE gen_xxx_data()
   BEGIN
     DECLARE i INT DEFAULT 0;
     WHILE i < 10000 DO
       INSERT INTO xxx (...) VALUES (...);
       SET i = i + 1;
     END WHILE;
   END //
   DELIMITER ;
   CALL gen_xxx_data();
   ```

3. **Mock data guidelines**: Field values should match enums/formats from code, time fields use random timestamps within recent N days, foreign keys reference existing IDs.

4. **Verify**: `$MYSQL -e "SELECT 'tbl' AS t, COUNT(*) FROM tbl UNION ALL ..."`

### Step 5: EXPLAIN Analysis

Analyze each query from the Step 1 inventory by number. Skip pure INSERTs and primary-key-only lookups (mark as "skipped").

```bash
# Fetch sample values first (to avoid "no matching row" false results)
$MYSQL -e "SELECT id FROM some_table LIMIT 1"
# Then EXPLAIN
$MYSQL -e "EXPLAIN SELECT * FROM some_table WHERE id = 'sample_value'"
```

**JOIN notes**: EXPLAIN outputs multiple rows; check driver table selection and driven table access method. LEFT JOIN driven table with type=ALL is a serious issue. JOIN condition columns must have indexes.

**Key metrics**:
- `type`: ALL / index -> needs an index
- `possible_keys` / `key`: NULL -> no candidates or not used
- `rows`: higher = more urgently needs optimization
- `Extra`: Using filesort / Using temporary -> needs attention

**Index exploration**: For each problematic query (type=ALL, filesort, excessive rows), don't just try one index — enumerate multiple candidate index combinations based on columns in WHERE / JOIN / ORDER BY / GROUP BY (different column subsets, different column orderings), and verify each one:

```bash
# Create candidate A
$MYSQL -e "CREATE INDEX idx_candidate_a ON orders (status, created_at)"
$MYSQL -e "EXPLAIN SELECT ..."
# Record results, then DROP and try next
$MYSQL -e "DROP INDEX idx_candidate_a ON orders"

# Create candidate B
$MYSQL -e "CREATE INDEX idx_candidate_b ON orders (status, region, created_at)"
$MYSQL -e "EXPLAIN SELECT ..."
$MYSQL -e "DROP INDEX idx_candidate_b ON orders"
```

Compare type / rows / Extra across all candidates, select the best and write to report. Preserve the exploration process in the report (EXPLAIN comparison of each candidate) so users can see the decision rationale.

After completion, verify against the inventory that every query has been analyzed or marked as skipped.

### Step 6: Output Report

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

Use AskUserQuestion to ask:

| Option | Description |
|--------|-------------|
| **Keep instance, let it auto-expire** | Instance will be automatically destroyed when TTL expires; user can connect manually to verify index effects in the meantime |
| **Destroy immediately** | Release the instance now |

Output instance connection info for the user's reference: `host`, `port`, `username`, `password`, remaining TTL.

If user chooses to destroy immediately:

```bash
curl -s -X DELETE https://zero.polardbx.com/api/v1/instances/<instance_id>
```

Finally, delete temporary SQL files.

## Index Design Principles

Detailed index design rules in [references/index-design.md](references/index-design.md).

## Examples

Complete analysis example in [references/examples.md](references/examples.md).
