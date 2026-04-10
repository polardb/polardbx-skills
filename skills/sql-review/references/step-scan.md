# Step 1-2: Confirm Scope & Scan Codebase

## Step 1: Confirm Scope and Database Edition

Ask the user **both** questions simultaneously:

**Question 1: Scan Scope**

| Mode | Description |
|------|-------------|
| **Full repository** | Scan all SQL in the entire codebase, ideal for first-time review or full audit |
| **Specific module/directory** | Only scan a user-specified directory (e.g. `src/services/`), ideal for targeted analysis |
| **Recent Git commits** | Only scan SQL in files changed in the last N days (or N commits), ideal for incremental review |
| **Aliyun instance** | Collect SQL insights and slow queries from a PolarDB-X 2.0 instance via `aliyun` CLI. No codebase needed |

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
  - **Aliyun instance**: Skip code scanning entirely. Execute **Step 2 阿里云路径** (see [step-aliyun.md](step-aliyun.md)) instead. Step 3 (name anonymization) and all subsequent steps proceed as normal. Database edition is automatically set to PolarDB-X Enterprise.
- **Database edition**: Record the user's choice, pass the corresponding `edition` parameter when creating the instance in Step 4

## Progress File

**Initialize** `$TMPDIR/sql_review_progress.md` at the start of Step 2 and **append incrementally** throughout the entire workflow. This file is the single source of truth — if the agent's context is compressed during a long session, re-read this file to resume.

```markdown
# SQL Review Progress
## Config
- Scope: <Full repository | ...>
- Edition: <Community MySQL | ...>
- Framework: <detected framework>
## DDL Inventory
| # | Table | PK | Columns | File:Line |
|---|-------|----|---------|-----------|
## Queries by Table
```

**Rule**: After processing each table (scan OR explain), immediately append results to this file. Never batch multiple tables before saving.

## Step 2: Scan Codebase

**Goal**: Collect all DDL and SQL statements within the scope defined in Step 1, produce a complete SQL inventory.

**Important**: Use multi-round searches with cross-validation. Do not rely on a single search pass.

### 2.0 Framework Detection

Before scanning, detect whether the project uses MyBatis:

1. Glob search for `**/*Mapper*.xml`, `**/*mapper*.xml`, `**/*Dao*.xml`
2. Verify matched files contain `<mapper namespace=` (to exclude non-MyBatis XML)
3. If MyBatis mapper files are found:
   - Record the list of mapper files
   - Output: "Detected N MyBatis mapper files. Will use XML-structure-aware extraction."
   - Proceed with both 2.1 (for DDL and non-mapper SQL) AND 2.2 (for mapper XML)
4. If no MyBatis mapper files found: skip 2.2 entirely, proceed with 2.1 only

### 2.1 Multi-round Search

Within the scope defined in Step 1, search for the following patterns (excluding node_modules, dist, .git, target, etc.):

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

> When MyBatis is detected: Rounds 1-2 (DDL) still run on the full codebase. Rounds 3-9 may produce hits in mapper XML files — these can be ignored since Step 2.2 provides more accurate extraction from XML structure. Focus rounds 3-9 on non-XML files (Java/Kotlin code with raw SQL, annotation-based SQL like `@Select`, etc.).

### 2.2 MyBatis Mapper Extraction

> **Only when MyBatis detected in Step 2.0.** This step extracts SQL using XML tag boundaries instead of regex, then expands dynamic SQL tags into executable SQL. Full tag expansion rules in [mybatis.md](mybatis.md).

#### 2.2.0 Pre-scan: Collect SQL Fragments

Before processing individual mappers, do a **quick first pass** to collect `<sql id="...">` fragments from ALL mapper files. Build a global fragment map: `{ "namespace.id" => content, "id" => content }`. This is needed because `<include refid="X"/>` may reference fragments from other mappers. Resolve recursively (depth limit 5).

#### 2.2.1 Process Each Mapper — One at a Time

**Process mappers one by one.** For each mapper XML file:

1. **Read** the full file
2. **Extract** all `<select>`, `<insert>`, `<update>`, `<delete>` tags. Record `id` attribute + mapper `namespace` as statement ID (e.g., `OrderMapper.findOrderList`).
3. **Inline** `<include refid="X"/>` using the global fragment map from 2.2.0
4. **Expand** dynamic SQL tags per [mybatis.md](mybatis.md) Section 1. Primary variant = all `<if>` true (maximum expansion).
5. **Multi-path analysis** for SELECT/COUNT with `<if>`/`<choose>` affecting WHERE columns. Generate 2-4 path variants (Full, Minimum, No-Identity, Key-Branch). Cap at 4, dedup by WHERE column set. Skip `<set>` / `insertSelective`. Full algorithm in [mybatis.md](mybatis.md) Section 3.
6. **Normalize** placeholders: `#{param}` → `?`, flag `${param}` as injection risk. Strip XML tags, CDATA, `&gt;`/`&lt;`/`&amp;`.
7. **Save immediately** — append this table's queries to `$TMPDIR/sql_review_progress.md`:

```markdown
### Table: <table_name> (<mapper_file>)
| # | Stmt ID | Type | Variant | SQL | Features |
|---|---------|------|---------|-----|----------|
| Q1 | Mapper.method | SELECT | Full | SELECT ... WHERE ... | ORDER BY, LIMIT |
| Q1b | Mapper.method | SELECT | Min | SELECT ... | ORDER BY, LIMIT |
| Q2 | Mapper.method2 | UPDATE | — | UPDATE ... WHERE pk = ? | PK lookup |
```

8. **Discard** the raw XML content from memory — the progress file has everything needed.

> **Why one-at-a-time**: A project with 10+ mapper files (each 5-20KB) can consume 100KB+ of context. Processing and saving incrementally prevents context overflow and allows resumption if the session is compressed mid-way.

### 2.3 Produce SQL Inventory

The SQL inventory is the accumulation of all per-table sections saved in `$TMPDIR/sql_review_progress.md` during steps 2.1 and 2.2. After all mappers are processed, append a summary section:

```markdown
## Summary
- Tables: N
- Total queries: N (SELECT: N, INSERT: N, UPDATE: N, DELETE: N)
- Multi-path queries: N (total variants: M)
- Code issues found: <list any bugs like LIKE quoting errors, ${} injection risks>
```

Column conventions in the per-table query tables:
- **Stmt ID**: For MyBatis queries, use `namespace.methodId` (e.g., `OrderMapper.findOrderList`). For non-MyBatis queries, use file:line.
- **Variant**: Path label for multi-path queries (e.g., "Full", "User view", "Admin: no userId", "Min"). Single-path queries use "—".
- Each variant gets a sub-number: Q1, Q1b, Q1c.
- **Features** column must note: JOIN / LEFT JOIN / subquery / GROUP BY / ORDER BY / LIMIT / HAVING / dynamic ORDER BY, etc.

### 2.4 Cross-validation

- JOIN count >= Grep `\bJOIN\b` hit count
- Every table in DDL should appear in queries
- All queries involving 2+ tables must have Key Features annotated
- (MyBatis) Every `<include refid="...">` has a matching `<sql id="...">`
- (MyBatis) Every table referenced in mapper queries appears in the DDL inventory
