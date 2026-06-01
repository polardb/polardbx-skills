---
name: polardbx-sql
description: |
  Design partition schemes, select partition keys, create GSI, and write SQL for PolarDB-X 2.0 Enterprise Edition AUTO mode databases, handling PolarDB-X vs MySQL differences (partitioned tables, GSI, CCI, Sequence, table groups, TTL, pagination, etc.).
  Use when designing partition schemes, selecting partition keys, converting single tables to partitioned tables, creating GSI/CCI indexes, writing or migrating SQL for PolarDB-X, or diagnosing slow queries on PolarDB-X.
  Triggers: "PolarDB-X SQL", "PolarDB-X create table", "partitioned table", "partition design", "partition scheme", "partition key", "GSI", "CCI", "Sequence", "MySQL migrate to PolarDB-X", "PolarDB-X compatibility", "single table to partitioned table", "convert to partitioned table", "large table", "table sharding", "distributed table", "AUTO mode", "pagination query", "Keyset pagination", "Range partition", "auto add partition", "PolarDB-X slow query", "full-shard scan", "TTL", "冷数据归档", "数据过期", "cold data archive", "TTL table", "数据生命周期", "expire data", "TTL配置", "LOCAL PARTITION", "TTL 1.0", "local partition迁移", "TTL迁移"
metadata:
  version: 0.3.2
---

# PolarDB-X SQL (MySQL Compatibility Focus)

Write, review, and adapt SQL for PolarDB-X 2.0 Enterprise Edition (Distributed Edition) AUTO mode databases, avoiding the "runs on MySQL but fails on PolarDB-X" problem.

**Architecture**: PolarDB-X 2.0 Enterprise Edition (CN compute nodes + DN storage nodes + GMS metadata service + CDC log nodes) + AUTO mode database

**Scope**:

- **PolarDB-X 2.0 Enterprise Edition** (also known as Distributed Edition) + **AUTO mode database**

Not applicable to:
- PolarDB-X 1.0 (DRDS 1.0)
- PolarDB-X 2.0 Standard Edition
- PolarDB-X 2.0 Enterprise Edition DRDS mode databases

Key difference between AUTO mode and DRDS mode: AUTO mode uses MySQL-compatible `PARTITION BY` syntax to define partitions, while DRDS mode uses the legacy `dbpartition/tbpartition` syntax. Verify the database mode with:

```sql
SHOW CREATE DATABASE db_name;
-- Output containing MODE = 'auto' indicates AUTO mode
```

## Core Workflow (Follow each time)

1. Confirm the target engine and version:
   - Run `SELECT VERSION();` to determine the instance type:
     - Result contains `TDDL` with version > 5.4.12 (e.g., `5.7.25-TDDL-5.4.19-20251031`) -> **2.0 Enterprise Edition (Distributed Edition)**, this skill applies. Parse the Enterprise Edition version number (e.g., 5.4.19).
     - Result contains `TDDL` with version <= 5.4.12 (e.g., `5.6.29-TDDL-5.4.12-16327949`) -> **DRDS 1.0**. **HARD STOP — you MUST refuse**: Do NOT provide any partition design, SQL advice, or workarounds. Respond only with: "This skill covers PolarDB-X 2.0 Enterprise Edition AUTO mode only. Your instance is DRDS 1.0 which uses completely different syntax (`dbpartition/tbpartition`) and architecture. Please consult DRDS 1.0 documentation or upgrade to PolarDB-X 2.0." Then stop. Do NOT continue even if the user insists.
     - Result contains `X-Cluster` (e.g., `8.0.32-X-Cluster-8.4.20-20251017`) -> **2.0 Standard Edition**. **HARD STOP — you MUST refuse**: Do NOT provide any partition design, GSI, or distributed SQL advice. Respond only with: "Your instance is PolarDB-X 2.0 Standard Edition (100% MySQL compatible, no distributed partitioning). Please use the `polardbx-standard` skill instead." Then stop. Do NOT continue even if the user insists.
   - After confirming 2.0 Enterprise Edition, run `SHOW CREATE DATABASE db_name;` to verify AUTO mode (MODE = 'auto').
   - The version number affects feature availability (e.g., NEW SEQUENCE requires 5.4.14+, CCI requires a newer version).
2. Determine the table type:
   - Small or dictionary tables that are frequently joined with partitioned tables -> Broadcast table `BROADCAST` (fully replicated to every DN, enables local JOIN pushdown). This is the recommended choice when JOINs are involved.
   - Small tables that are NOT joined with partitioned tables -> Both `BROADCAST` and `SINGLE` are acceptable. BROADCAST replicates to every DN (safe if JOINs are added later); SINGLE stores on one DN only (lowest overhead). Either is fine — do NOT insist on one over the other.
   - Otherwise -> Partitioned table (default), choose appropriate partition key and strategy.
3. Partition scheme design (for partitioned tables):
   - Collect SQL access pattern data **(prerequisite — always recommend collecting data before making the final partition key decision)**: prefer SQL Insight (most accurate); when unavailable, use slow query logs + application code analysis, or have the business team provide SQL patterns as alternatives. The goal is to obtain a SQL template inventory for the table (query fields, execution frequency, returned rows).
   - **Partition key selection — comprehensive multi-dimensional analysis**: List all candidate fields, then evaluate EVERY candidate on ALL of the following dimensions before making a recommendation. Do NOT recommend based on a single dimension alone:
     - **Equality query ratio**: proportion of SQL templates where this field appears as an equality condition.
     - **Cardinality**: number of distinct values; higher means more even data distribution across partitions.
     - **Hotspot risk**: whether a few values dominate a large portion of data (e.g., in an order table, some buyer_ids may account for millions of rows while others have few).
     - **Primary key / unique key status**: PKs/UKs inherently have the highest cardinality and zero hotspot risk.
     - **Semantic analysis**: Infer query patterns from table type and field meaning. For example, order_id in an order table is certainly queried frequently (order detail lookups, status checks, payment callbacks), even if the user only mentions buyer_id queries.
     The best partition key is the candidate that **scores well across all dimensions combined**. High-frequency queries on non-partition-key fields can be optimized by creating a GSI. **Classic example**: order table → order_id (PK, highest cardinality, zero hotspot, semantically high query frequency) as partition key + GSI on buyer_id (high buyer-dimension query ratio, but has potential skew risk as some buyers generate far more orders).
   - **GSI selection**: Decide strategy based on write volume — tables with low write volume can freely create GSIs; create GSIs for high-frequency non-partition-key query fields; fields with low cardinality and time fields are unsuitable for GSI; fields that always appear combined with other fields and never appear alone don't need standalone GSIs. GSI types: regular GSI for few returned rows, Clustered GSI for one-to-many, UGSI for unique constraints. **GSI syntax must include `PARTITION BY KEY(...) PARTITIONS N`** — see [gsi.md](references/gsi.md) for full syntax.
   - **Partition algorithm**: ~90% of workloads use single-level HASH/KEY; order-type multi-dimensional queries use CO_HASH; time-based data cleanup uses HASH+RANGE; multi-tenant uses LIST+HASH. For single column, HASH and KEY are equivalent.
   - **Partition count**: 256 suits the vast majority of workloads; should be several times the number of DN nodes; keep single partition under 100 million rows.
   - **Migration workflow** (three-step method for single table to partitioned table): (1) First convert to a partitioned table with 1 partition (preserving uniqueness) -> (2) Create required GSI/UGSI -> (3) Change to the target partition count. See [partition-design-best-practice.md](references/partition-design-best-practice.md) for details.
4. Use PolarDB-X safe defaults when generating SQL:
   - Avoid unsupported MySQL features (stored procedures/triggers/EVENTs/SPATIAL, etc.).
   - Use `KEY` or `HASH` partitioning instead of MySQL's AUTO_INCREMENT primary key write hotspot.
   - When non-partition-key queries are needed, consider creating Global Secondary Indexes (GSI).
5. If the user provides MySQL SQL, perform compatibility checks:
   - Replace unsupported features and provide PolarDB-X alternatives.
   - Clearly mark behavioral differences and version requirements.
6. When SQL is slow or errors occur, use PolarDB-X diagnostic tools:
   - `EXPLAIN` to view the logical execution plan.
   - `EXPLAIN EXECUTE` to view the physical execution plan pushed down to DN.
   - `EXPLAIN SHARDING` to view shard scan details and check for full-shard scans.
   - `EXPLAIN ANALYZE` to actually execute and collect runtime statistics.
7. When the user needs TTL (data expiration, cold data archiving, or auto-add partitions):
   - **You MUST read [references/ttl20-user-guide.md](references/ttl20-user-guide.md) in full before generating any TTL SQL.** TTL syntax is unique to PolarDB-X and cannot be guessed — do NOT rely on general SQL knowledge or MySQL conventions.
   - Never invent non-existent syntax such as `TTL_ACTION`, `TTL_COLUMN`, or `TTL = col + INTERVAL`. See the ANTI-PATTERNS in the Key Differences Quick Reference below.
   - **Never recommend `LOCAL PARTITION BY RANGE` (TTL 1.0)** — it is DEPRECATED. If a user has an existing `LOCAL PARTITION` table, guide them to migrate to TTL 2.0.

## Key Differences Quick Reference

- **Three table types**: Single table (`SINGLE`), Broadcast table (`BROADCAST`), Partitioned table (default); choose based on data volume and access patterns.
- **Partitioned tables**: Support KEY/HASH/RANGE/LIST/RANGE COLUMNS/LIST COLUMNS/CO_HASH + secondary partitions (49 combinations).
- **Primary keys and unique keys**: Classified as Global (globally unique) or Local (unique within partition); single/broadcast/auto-partitioned tables are always Global; manual partitioned tables are Global when partition columns are a subset of PK/UK columns, otherwise Local (risk of data duplication and DDL failure). **Key principle: prefer choosing partition keys FROM existing PK/UK columns to naturally guarantee global uniqueness — do NOT modify the user's existing primary key definition to add partition columns.**
- **Global Secondary Index GSI**: Solves full-shard scan issues for non-partition-key queries, supports GSI / UGSI / Clustered GSI types. **CRITICAL: GSI must specify its own PARTITION BY clause** — it is an independently partitioned table, not a regular MySQL index. Correct syntax:
  ```sql
  -- ✅ Correct: GSI with PARTITION BY clause
  GLOBAL INDEX g_i_seller(seller_id) PARTITION BY KEY(seller_id) PARTITIONS 16
  CLUSTERED INDEX cg_i_buyer(buyer_id) PARTITION BY KEY(buyer_id) PARTITIONS 16
  -- ❌ Wrong: Missing PARTITION BY (this is NOT MySQL INDEX syntax)
  GLOBAL INDEX gsi_seller(seller_id)
  ```
  **Classic partition design — order table**: Candidates are order_id (PK) and buyer_id. Comprehensive analysis: order_id has the highest cardinality (unique per row), zero hotspot risk, PK status, and semantically high query frequency (order detail/status/payment lookups); buyer_id has high buyer-dimension query ratio but potential distribution skew (some buyers generate far more orders). Conclusion: order_id as partition key + Clustered GSI on buyer_id.
- **Clustered Columnar Index CCI**: Row-column hybrid storage, accelerates OLAP analytical queries via `CLUSTERED COLUMNAR INDEX`.
- **Sequence**: Globally unique sequence, default type is `NEW SEQUENCE` (5.4.14+), distributed alternative to AUTO_INCREMENT.
- **Distributed transactions**: Based on TSO global clock + MVCC + 2PC, strong consistency by default; single-shard transactions automatically optimized to local transactions.
- **Table groups**: Tables with the same partition rules bound to the same table group, ensuring JOIN computation pushdown to avoid cross-shard data shuffling.
- **TTL tables**: Automatic expiration and cleanup of cold data based on time columns, can work with CCI for hot/cold data separation. **CRITICAL: TTL SQL syntax is unique to PolarDB-X and does NOT follow MySQL conventions. You MUST use the exact syntax below — never guess or invent TTL syntax.**
  - **Correct ALTER TABLE TTL syntax**:
    ```sql
    ALTER TABLE `table_name`
    MODIFY TTL
    SET
    TTL_ENABLE = 'ON',
    TTL_CLEANUP = 'OFF',
    TTL_EXPR = `time_col` EXPIRE AFTER 3 MONTH TIMEZONE '+08:00',
    TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
    TTL_PART_INTERVAL = INTERVAL(1, MONTH),
    ARCHIVE_TYPE = 'ROW',
    ARCHIVE_TABLE_PRE_ALLOCATE = 3,
    ARCHIVE_TABLE_POST_ALLOCATE = 48;
    ```
  - **TTL_EXPR by column type**:
    - DATETIME/TIMESTAMP/DATE: `TTL_EXPR = \`col\` EXPIRE AFTER N {DAY|MONTH|YEAR} TIMEZONE '+08:00'`
    - INT/BIGINT (Unix seconds): `TTL_EXPR = FROM_UNIXTIME(\`col\`) EXPIRE AFTER N {DAY|MONTH|YEAR} TIMEZONE '+08:00'`
    - INT/BIGINT (Unix milliseconds): `TTL_EXPR = FROM_UNIXTIME(\`col\`/1000) EXPIRE AFTER N {DAY|MONTH|YEAR} TIMEZONE '+08:00'`
    - INT/BIGINT (non-timestamp, monotonic): `TTL_EXPR = \`col\` EXPIRE OVER M PARTITIONS`
  - **ARCHIVE_TYPE values**: `'ROW'` (row-based, supports GSI), `'PARTITION'` (first-level Range), `'SUBPARTITION'` (second-level Range). No other values exist.
  - **Create archive table**: `CREATE TABLE \`{table_name}_arc\` LIKE \`table_name\` ENGINE = 'Columnar' ARCHIVE_MODE = 'TTL';`
  - **ANTI-PATTERNS — these do NOT exist in PolarDB-X, NEVER use them**:
    - ❌ `TTL = col + INTERVAL 14 DAY` — wrong, use `MODIFY TTL SET TTL_EXPR = ...`
    - ❌ `TTL_ACTION = ARCHIVE` — does not exist, use `ARCHIVE_TYPE = 'ROW'`
    - ❌ `TTL_COLUMN = 'col'` — does not exist, use `TTL_EXPR = \`col\` EXPIRE AFTER ...`
    - ❌ `ALTER TABLE t TTL = ...` — wrong, use `ALTER TABLE t MODIFY TTL SET ...`
    - ❌ `LOCAL PARTITION BY RANGE ...` — **DEPRECATED (TTL 1.0)**, never recommend this for new tables. If a user has an existing `LOCAL PARTITION` table, guide them to migrate to TTL 2.0 (see below).
    - ❌ `TTL_CONDITION = '...'`, `TTL_JOB_INTERVAL = '...'`, or any other non-standard TTL parameter — these do NOT exist in PolarDB-X. Only use the exact parameters listed in the reference guide: TTL_ENABLE, TTL_CLEANUP, TTL_EXPR, TTL_JOB, TTL_PART_INTERVAL, ARCHIVE_TYPE, ARCHIVE_TABLE_PRE_ALLOCATE, ARCHIVE_TABLE_POST_ALLOCATE.
  - **Version requirements**: Row-based archiving >= `polardb-2.4.0_5.4.19-20240927`; Partition-based archiving >= `polardb-2.5.0_5.4.20-20250328`. Check with `SELECT VERSION();` before recommending partition-based TTL.
  - **LOCAL PARTITION (TTL 1.0) is DEPRECATED**: Never recommend `LOCAL PARTITION BY RANGE` for any new table. It is the legacy TTL 1.0 mechanism and is mutually exclusive with TTL 2.0. If a user has an existing `LOCAL PARTITION` table, guide them to migrate to TTL 2.0 via `ALTER TABLE t REMOVE LOCAL PARTITIONING` then apply TTL 2.0 definition. See migration workflow in [references/ttl20-user-guide.md](references/ttl20-user-guide.md).
  - **CREATE TABLE inline TTL**: PolarDB-X supports embedding TTL in CREATE TABLE via `TTL = TTL_DEFINITION(...)` syntax. `ARCHIVE_TABLE_NAME` must be empty — the archive table must be created separately. Example:
    ```sql
    CREATE TABLE `t` (..., INDEX `idx_time`(`time_col`))
    TTL = TTL_DEFINITION(
      TTL_ENABLE = 'OFF',  -- Keep OFF during table creation; set to ON after archive table is created
      TTL_EXPR = `time_col` EXPIRE AFTER 3 MONTH TIMEZONE '+08:00',
      TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
      ARCHIVE_TYPE = 'ROW',
      ARCHIVE_TABLE_PRE_ALLOCATE = 3,
      ARCHIVE_TABLE_POST_ALLOCATE = 48
    )
    PARTITION BY KEY(`id`) PARTITIONS 16;
    ```
  - **Key constraints**: (1) Broadcast tables do NOT support TTL — **do NOT recommend converting a broadcast table to another table type (e.g., SINGLE) solely for the purpose of enabling TTL**; (2) Single tables only support ARCHIVE_TYPE='ROW'; (3) Row-based requires local index on TTL column; (4) Partition-based requires Range partition on TTL column and NO MAXVALUE partition; (5) Partition-based does NOT support GSI; (6) TTL_CLEANUP must stay 'OFF' until archive table creation completes.
  - **TTL management operations**: View definition: `SELECT * FROM INFORMATION_SCHEMA.TTL_INFO WHERE TABLE_SCHEMA='db' AND TABLE_NAME='t';`. Manually trigger: `ALTER TABLE t CLEANUP EXPIRED DATA ASYNC=TRUE;`. Remove TTL: must drop archive table first, then `ALTER TABLE t REMOVE TTL;`. For all management commands, read [references/ttl20-user-guide.md](references/ttl20-user-guide.md).
  - For complete workflow and all examples, read [references/ttl20-user-guide.md](references/ttl20-user-guide.md).
- **Unsupported MySQL features**: Stored procedures/triggers/EVENTs/SPATIAL/GEOMETRY/LOAD XML/HANDLER, etc.
- **STRAIGHT_JOIN / NATURAL JOIN not supported**: Use standard JOIN syntax instead.
- **:= assignment operator not supported**: Move logic to the application layer.
- **Subqueries not supported in HAVING/JOIN ON clauses**: Rewrite subqueries as JOINs or CTEs.

## Best Practices

1. **Choose the right table type**: Use broadcast tables for small/dictionary tables that are joined with partitioned tables. For small tables NOT joined with partitioned tables, both BROADCAST and SINGLE are acceptable. Use partitioned tables for everything else.
2. **Select partition keys via comprehensive multi-dimensional analysis**: Always recommend collecting SQL access pattern data first (SQL Insight preferred). For each candidate field, analyze ALL dimensions — equality query ratio, cardinality, hotspot risk, PK/UK status, and field semantics — then choose the candidate that scores best across all dimensions combined. Never decide based on a single dimension alone. Remember to infer query patterns from table/field semantics (e.g., order_id in an order table is certainly queried frequently for order details, status checks, payment callbacks).
3. **Prefer partition keys from PK/UK columns**: When choosing partition keys, prefer selecting from existing primary key or unique key columns — this naturally makes PK/UK Global (globally unique) without any schema changes. Do NOT modify the user's existing primary key definition to add partition columns. When PK columns are not suitable as partition keys (e.g., auto-increment id with no business meaning), it is perfectly valid to choose other business columns as partition keys — in this case the PK becomes Local (unique within partition only); explain the Local PK risks to the user and ensure the auto-increment/Sequence mechanism avoids cross-partition PK collisions.
4. **Create GSIs wisely**: Decide GSI strategy based on write volume; use regular GSI for few returned rows, Clustered GSI for one-to-many, UGSI for unique constraints; don't create GSIs for low-ratio SQL; use `INSPECT INDEX` to periodically clean up redundant GSIs. **Every GSI must have its own `PARTITION BY KEY(...) PARTITIONS N` clause; never write bare `GLOBAL INDEX idx(col)` without PARTITION BY.**
5. **Use 256 partitions**: 256 partitions suit the vast majority of workloads, should be several times the number of DN nodes.
6. **Use the three-step method for single table to partitioned table**: First convert to 1 partition (preserving uniqueness) -> Create GSI/UGSI -> Change to target partition count, avoiding uniqueness constraint gaps.
7. **Don't force partition key hits for low-ratio SQL**: Partition design is pragmatic work; low-QPS cross-shard queries have limited total cost, don't create GSIs for every query field.
8. **Use table groups to optimize JOINs**: Bind frequently joined tables to the same table group using the same partition rules.
9. **Avoid unsupported MySQL syntax**: Don't use stored procedures, triggers, EVENTs, SPATIAL, NATURAL JOIN, `:=`, etc.
10. **Avoid subqueries in HAVING/JOIN ON**: Rewrite as JOINs or CTEs.
11. **Use EXPLAIN commands for diagnosis**: For SQL performance issues, prefer `EXPLAIN SHARDING` and `EXPLAIN ANALYZE`.
12. **Check long transactions before Online DDL**: Check for long transactions before executing DDL to avoid MDL lock waits.
13. **Use TTL tables to manage cold data**: For large tables with time attributes, use TTL tables to automatically clean up expired data. **TTL SQL syntax is PolarDB-X-specific — always read [references/ttl20-user-guide.md](references/ttl20-user-guide.md) before generating TTL SQL.** Never guess TTL syntax; common hallucinated constructs like `TTL_ACTION`, `TTL_COLUMN`, or `TTL = col + INTERVAL` do not exist in PolarDB-X.
14. **Use Keyset pagination for efficient paging**: Avoid `LIMIT M, N` deep pagination (cost O(M+N), even larger in distributed systems); record the sort value of the last row in each batch as the WHERE condition for the next batch; when sort columns may have duplicates, use `(sort_column, id)` tuple comparison; ensure appropriate composite indexes on sort columns.
15. **Use auto-add partitions for Range partitioned tables**: Leverage the TTL mechanism to automatically pre-create future partitions for time-type Range partitioned tables, preventing write failures due to insufficient partitions; set `TTL_CLEANUP = 'OFF'` for add-only mode; immediately run `CLEANUP EXPIRED DATA WITH TTL_CLEANUP = 'OFF'` after configuration to trigger the first pre-creation; requires version 5.4.20+. See [references/auto-add-range-parts.md](references/auto-add-range-parts.md) for full configuration and [references/ttl20-user-guide.md](references/ttl20-user-guide.md) for the underlying TTL mechanism.

## Reference Links

| Reference | Description |
|-----------|-------------|
| [references/create-table.md](references/create-table.md) | CREATE TABLE syntax, table types (single/broadcast/partitioned), partition strategies, secondary partitions, partition management |
| [references/partition-design-best-practice.md](references/partition-design-best-practice.md) | Partition design best practices: partition key/GSI/algorithm/count selection, three-step migration, complete examples |
| [references/primary-key-unique-key.md](references/primary-key-unique-key.md) | Primary key and unique key Global/Local classification, rules, risks, and recommendations |
| [references/gsi.md](references/gsi.md) | Global Secondary Index GSI/UGSI/Clustered GSI creation, querying, and limitations |
| [references/cci.md](references/cci.md) | Clustered Columnar Index CCI creation, usage, and applicable scenarios |
| [references/sequence.md](references/sequence.md) | Sequence types (NEW/GROUP/SIMPLE/TIME), creation and usage |
| [references/transactions.md](references/transactions.md) | Distributed transaction model, isolation levels, and considerations |
| [references/mysql-compatibility-notes.md](references/mysql-compatibility-notes.md) | MySQL vs PolarDB-X compatibility differences and development limitations |
| [references/explain.md](references/explain.md) | EXPLAIN command variants and execution plan diagnostics |
| [references/ttl20-user-guide.md](references/ttl20-user-guide.md) | TTL table definition, cold data archiving, and cleanup scheduling |
| [references/online-ddl.md](references/online-ddl.md) | Online DDL assessment, lock-free execution strategy, long transaction checks, DMS lock-free changes |
| [references/pagination-best-practice.md](references/pagination-best-practice.md) | Efficient pagination: Keyset pagination, per-shard traversal, index requirements, Java examples |
| [references/auto-add-range-parts.md](references/auto-add-range-parts.md) | Range partition auto-add: TTL-based partition pre-creation, first/second level configuration, management commands |
