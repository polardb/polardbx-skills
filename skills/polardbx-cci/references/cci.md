---
title: PolarDB-X Clustered Columnar Index (CCI) — Complete Reference
---

# PolarDB-X Clustered Columnar Index (CCI) — Complete Reference

CCI is a row-column hybrid storage capability for PolarDB-X Enterprise Edition (AUTO mode). CCI is a columnar clustered index based on object storage that stores all primary table columns in columnar format by default, designed to accelerate OLAP analytical queries.

## Applicable Scenarios

1. **HTAP mixed workload**: OLTP + OLAP on the same data without impacting transactional performance.
2. **Wide table aggregation**: SUM/COUNT/AVG across many columns benefits from columnar scan.
3. **Complex reports and dashboards**: Analytical queries with GROUP BY, JOIN, complex filters.
4. **Cold data archival with TTL**: CCI on object storage provides low-cost analytical access to archived data.
5. **Historical snapshot query and recovery**: Point-in-time data access via AS OF TSO snapshots.
6. **ETL / data extraction**: Connect to columnar read-only instance for data pipeline workloads.

## Version Requirements

- CCI creation: >= 5.4.19-16989811
- Snapshot features: >= 5.4.19-20250305 or >= 5.4.20
- COLUMNAR_OPTIONS / columnar_set_config: >= 5.4.20

## Creation Syntax

### Full syntax

```sql
CREATE CLUSTERED COLUMNAR INDEX index_name
  ON tbl_name (sort_key_col1, ...)
  [PARTITION BY
    HASH({column_name | partition_func(column_name)})
    | KEY(column_list)
    | RANGE({column_name | partition_func(column_name)})
    | RANGE COLUMNS(column_list)
    | LIST({column_name | partition_func(column_name)})
    | LIST COLUMNS(column_list)
  ]
  [partition_list_spec]
  [COLUMNAR_OPTIONS = '{"key":"value", ...}']

-- partition functions: YEAR | TO_DAYS | TO_SECOND | UNIX_TIMESTAMP | MONTH
-- hash partition: PARTITIONS partition_count
-- range partition: PARTITION p_name VALUES LESS THAN (expr | value_list)
-- list partition: PARTITION p_name VALUES IN (value_list)
```

### Create during table creation

```sql
CREATE TABLE t_order (
  id BIGINT PRIMARY KEY,
  buyer_id BIGINT,
  seller_id BIGINT,
  amount DECIMAL(10,2),
  create_time DATETIME,
  CLUSTERED COLUMNAR INDEX cci_seller(seller_id)
    PARTITION BY KEY(seller_id) PARTITIONS 16
) PARTITION BY KEY(id) PARTITIONS 16;
```

### Add to existing table

```sql
-- Using CREATE INDEX
CREATE CLUSTERED COLUMNAR INDEX cci_buyer
  ON t_order(buyer_id)
  PARTITION BY KEY(buyer_id) PARTITIONS 16;

-- Using ALTER TABLE
ALTER TABLE t_order ADD CLUSTERED COLUMNAR INDEX cci_buyer(buyer_id)
  PARTITION BY KEY(buyer_id) PARTITIONS 16;
```

### With sort key and COLUMNAR_OPTIONS

```sql
CREATE CLUSTERED COLUMNAR INDEX cci_seller
  ON t_order(seller_id)
  PARTITION BY HASH(order_id) PARTITIONS 16
  COLUMNAR_OPTIONS = '{
    "TYPE": "SNAPSHOT",
    "SNAPSHOT_RETENTION_DAYS": "7",
    "AUTO_GEN_COLUMNAR_SNAPSHOT_INTERVAL": "30"
  }';
```

### Creation constraints

- **Table must have a primary key**: CCI requires the primary table to have an explicit primary key. Tables without a primary key cannot create CCI.
- **Sort key is REQUIRED**: Must explicitly specify in `ON tbl_name(sort_key_col)`.
- **Index name is REQUIRED**: Cannot be omitted.
- **No prefix index**: CCI does not support prefix indexes.
- **All columns included**: CCI stores all primary table columns by default; auto-adjusts with schema changes.
- **No local indexes**: CCI creation does not create any local indexes.
- **Sort key LENGTH ignored**: LENGTH parameter in sort key definition is ignored.
- **Partition default**: If not specified, uses primary key with HASH partitioning.
- **One CCI per table** (by default): Can be increased via `SET MAX_CCI_COUNT = N;`

## Partition Key Selection Guide

### Partition strategy

| Strategy | When to use |
|----------|-------------|
| **HASH / KEY** (recommended) | Default choice for CCI. AP queries benefit from even data distribution and parallel scan. |
| **RANGE** | Time-based cold data archival with TTL. When queries have clear range predicates on time columns. |
| **LIST** | SaaS multi-tenant scenarios with predefined value sets. |

**Note**: No clear range query? Avoid RANGE. No predefined value list? Avoid LIST. Prefer row-store for those cases.

### Partition key principles

1. **Uniform distribution**: Choose trade_id, device_id, user_id, auto-increment columns.
2. **Avoid date/time columns**: They cause data skew (write hotspots) and limit parallelism (most queries filter on recent dates, concentrating data in few partitions). Use date/time as **secondary partitions** instead.
3. **Prefer JOIN / GROUP BY columns**: Reduces data redistribution in analytical queries. Example: for customer order history analysis, use customer_id as partition key.
4. **Prefer non-range WHERE columns**: Enables partition pruning.
5. **Simplicity**: Fewer columns in partition key = better generality in complex queries.

### Partition count

- Default: 16 (not recommended for production).
- Recommended: `partition_count = num_compute_nodes * cores_per_node`.
- For multi-table JOIN workloads, keep partition counts consistent across tables.
- Consider future data growth when setting partition count.

### Secondary partitions

Use date/time columns as secondary partitions when queries have clear time-based predicates:
```sql
PARTITION BY HASH(order_id) PARTITIONS 16
SUBPARTITION BY RANGE(TO_DAYS(create_time)) (
  PARTITION p2024 VALUES LESS THAN (TO_DAYS('2025-01-01')),
  PARTITION p2025 VALUES LESS THAN (TO_DAYS('2026-01-01'))
);
```

### Verify partition quality

```sql
CHECK COLUMNAR PARTITION db_name.tbl_name;
```
Check for data skew — all partitions should have roughly equal row counts.

## Sort Key Selection Guide

The sort key defines how data is ordered in CCI files. Each data block stores min/max metadata, enabling **Pruner** functionality that skips irrelevant data blocks during scans.

| Scenario | Recommended sort key |
|----------|---------------------|
| Frequent range queries (WHERE col BETWEEN ...) | The range condition column |
| Pagination (ORDER BY col LIMIT N) | The ORDER BY column |
| General / no specific pattern | The partition key |

**Note**: Sort key and partition key can be completely different columns. E.g., partition by `order_id`, sort by `seller_id`.

## COLUMNAR_OPTIONS

JSON-formatted options for CCI creation:

```sql
COLUMNAR_OPTIONS = '{
  "TYPE": "default | snapshot | archive",
  "SNAPSHOT_RETENTION_DAYS": "7",
  "AUTO_GEN_COLUMNAR_SNAPSHOT_INTERVAL": "-1"
}'
```

| Parameter | Default | Description | Dynamic |
|-----------|---------|-------------|--------|
| `TYPE` | `default` | CCI type: `default` (standard), `snapshot` (with snapshots), `archive` (cold archive) | Yes |
| `SNAPSHOT_RETENTION_DAYS` | `7` | Snapshot retention in days (1-366). Only for TYPE=snapshot. | Yes |
| `AUTO_GEN_COLUMNAR_SNAPSHOT_INTERVAL` | `-1` | Auto snapshot interval in minutes (>= 5 or -1 to disable). Only for TYPE=snapshot. | Yes |

**TYPE transition rules**:
- `default → snapshot`: OK (auto-sets snapshot params, enables backup)
- `snapshot → default`: OK (cleans up snapshot config)
- `default → archive`: OK (enables backup)
- `archive → default`: OK
- `archive ↔ snapshot`: **FORBIDDEN** (must go through default first)

## Query Usage

### Automatic selection (requires columnar optimizer + columnar read-only instance)

```sql
SET ENABLE_COLUMNAR_OPTIMIZER = true;

SELECT seller_id, SUM(amount) FROM t_order
GROUP BY seller_id ORDER BY SUM(amount) DESC LIMIT 10;
```

### Force CCI via FORCE INDEX

```sql
SELECT seller_id, SUM(amount) FROM t_order FORCE INDEX(cci_seller)
GROUP BY seller_id;
```

### Force CCI via HINT

```sql
SELECT /*+TDDL:FORCE_INDEX(t_order, cci_seller)*/ seller_id, SUM(amount)
FROM t_order GROUP BY seller_id;
```

### USE INDEX / IGNORE INDEX

```sql
-- Suggest optimizer to use CCI
SELECT * FROM t_order USE INDEX(cci_seller) WHERE seller_id = 's1';

-- Exclude CCI from optimizer choices
SELECT * FROM t_order IGNORE INDEX(cci_seller) WHERE seller_id = 's1';
```

### Multi-table JOIN with CCI

```sql
SELECT a.*, b.order_id
FROM t_seller a
JOIN t_order b FORCE INDEX(cci_seller) ON a.seller_id = b.seller_id
WHERE a.seller_nick = 'abc';
```

## Columnar Snapshot

### Overview

CCI snapshots provide point-in-time data access. Requires `TYPE=snapshot` in COLUMNAR_OPTIONS.

### Create CCI with snapshot

```sql
CREATE CLUSTERED COLUMNAR INDEX cci ON tb1(id)
  PARTITION BY KEY(id) PARTITIONS 16
  COLUMNAR_OPTIONS = '{
    "TYPE": "SNAPSHOT",
    "SNAPSHOT_RETENTION_DAYS": "7",
    "AUTO_GEN_COLUMNAR_SNAPSHOT_INTERVAL": "30"
  }';
```

### Generate snapshot point

```sql
-- Table-level snapshot (returns TSO)
CALL polardbx.columnar_flush('schema_name', 'table_name', 'cci_name');

-- Instance-level snapshot
CALL polardbx.columnar_flush();
```

### Query historical snapshot

```sql
SELECT * FROM tb1 AS OF TSO <tso_value> FORCE INDEX(cci_name) ORDER BY id;
```

**Note**: `FORCE INDEX` must come **after** `AS OF TSO`.

### Restore data from snapshot

```sql
INSERT INTO target_table SELECT * FROM source_table AS OF TSO <tso_value> FORCE INDEX(cci_name);
```

### Snapshot limitations

- Snapshot queries use the **latest table schema** regardless of snapshot point.
- INSERT SELECT recovery requires `autocommit=true`.
- Complex queries in INSERT SELECT may have poor performance.
- Snapshots are retained for `SNAPSHOT_RETENTION_DAYS` days; beyond that, availability is not guaranteed.

## SHOW / CHECK Commands

### SHOW COLUMNAR INDEX

Displays CCI metadata: index name, partition key, partition strategy, partition count, sort key, status.

```sql
SHOW COLUMNAR INDEX;
SHOW COLUMNAR INDEX FROM table_name;
SHOW COLUMNAR INDEX AS OF TSO <tso_value>;
```

Output fields: `SCHEMA | TABLE | INDEX_NAME | CLUSTERED | PK_NAMES | COVERING_NAMES | PARTITION_KEY | PARTITION_STRATEGY | PARTITION_COUNT | SORT_KEY | STATUS`

| Field | Description |
|-------|-------------|
| SCHEMA | Database name |
| TABLE | Table name |
| INDEX_NAME | CCI name |
| CLUSTERED | Whether clustered |
| PK_NAMES | Primary key columns |
| PARTITION_KEY | Partition key columns |
| PARTITION_STRATEGY | HASH/KEY/RANGE/LIST |
| PARTITION_COUNT | Number of partitions |
| SORT_KEY | Sort key columns |
| STATUS | CCI status |

### SHOW COLUMNAR STATUS

Displays CCI data status: row count, file counts, size, compression ratio.

```sql
SHOW COLUMNAR STATUS;                           -- database level
SHOW FULL COLUMNAR STATUS;                      -- instance level
SHOW COLUMNAR STATUS WHERE TSO = <tso_value>;   -- specific snapshot
```

Output fields: `TSO | SCHEMA_NAME | TABLE_NAME | INDEX_NAME | ID | ROWS | CSV_FILES | ORC_FILES | DEL_FILES | FILES_SIZE | DN_TABLE_SIZE | COMPRESSION_RATIO | STATUS`

| Field | Description |
|-------|-------------|
| TSO | Current columnar timestamp |
| SCHEMA_NAME | Database name |
| TABLE_NAME | Table name |
| INDEX_NAME | CCI name |
| ID | CCI unique ID (used in columnar_set_config) |
| ROWS | Row count in CCI |
| CSV_FILES / ORC_FILES / DEL_FILES | File counts by type |
| FILES_SIZE | Total file size |
| COMPRESSION_RATIO | Compression ratio |
| STATUS | CCI status |

### Other SHOW commands

```sql
SHOW COLUMNAR OFFSET;    -- CCI offset information
```

### CHECK commands

```sql
CHECK COLUMNAR INDEX index_name ON table_name;     -- verify CCI data consistency
CHECK COLUMNAR INDEX index_name ON table_name INCREMENT;  -- incremental check
CHECK COLUMNAR PARTITION table_name;               -- check partition data distribution
CHECK COLUMNAR SNAPSHOT table_name;                -- check snapshot status
```

## Parameter Configuration

### Connection variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ENABLE_COLUMNAR_OPTIMIZER` | `false` | Enable automatic CCI query routing |
| `ENABLE_COLUMNAR_OPTIMIZER_WITH_COLUMNAR` | `false` | Enable CCI routing when columnar nodes exist |
| `ENABLE_COLUMNAR_CORRELATE` | `false` | Allow correlated subqueries on CCI |

### Modify CCI parameters at runtime

```sql
-- Instance-level
CALL polardbx.columnar_set_config(param_key, param_val);

-- CCI-level (by CCI ID from SHOW COLUMNAR STATUS)
CALL polardbx.columnar_set_config(cci_id, param_key, param_val);

-- CCI-level (by name)
CALL polardbx.columnar_set_config(schema_name, table_name, cci_name, param_key, param_val);
```

### Advanced parameters

```sql
-- Control DDL on tables with CCI
SET [GLOBAL] forbid_ddl_with_cci = true | false;

-- Allow modifying CCI critical columns (may trigger rebuild)
SET ENABLE_MODIFY_CCI_CRITICAL_COLUMN = TRUE;
SET REBUILD_CCI_STRATEGY = 0;  -- 0 = auto-select rebuild strategy

-- Max CCI count per table
SET MAX_CCI_COUNT = 2;
```

## Drop / Rename CCI

```sql
-- Drop CCI
DROP INDEX cci_name ON TABLE tbl_name;
ALTER TABLE tbl_name DROP INDEX cci_name;

-- Rename CCI
ALTER TABLE tbl_name RENAME INDEX old_cci_name TO new_cci_name;

-- Drop all CCIs in a database
DROP COLUMNAR INDEX FOR TABLES IN db_name;
```

## DDL Limitations

### Supported DDL on tables with CCI

| Operation | Supported |
|-----------|-----------|
| DROP TABLE | Yes |
| TRUNCATE TABLE | Yes |
| RENAME TABLE | Yes |
| ADD COLUMN | Yes |
| DROP COLUMN | Yes |
| MODIFY COLUMN (type change) | Yes (limited types) |
| CHANGE COLUMN | Yes (limited) |
| MODIFY COLUMN defaults | Yes |
| ADD PRIMARY KEY | Yes |
| ADD INDEX (non-CCI) | Yes |
| DROP INDEX (non-CCI) | Yes |
| DROP FOREIGN KEY | Yes |
| RENAME CCI INDEX | Yes |
| ADD RANGE PARTITION | Yes |
| DROP PRIMARY KEY | **No** |
| ALTER INDEX VISIBLE/INVISIBLE | **No** |
| DISABLE/ENABLE KEYS | **No** |
| Other CCI partition changes | **No** |

### Column change constraints with CCI

| Statement | PK change | Partition key change | Sort key change |
|-----------|-----------|---------------------|-----------------|
| MODIFY COLUMN | Yes | Yes | Yes |
| ALTER COLUMN SET/DROP DEFAULT | Yes | Yes | Yes |
| ADD COLUMN | N/A | N/A | N/A |
| CHANGE COLUMN | No | No | No |
| DROP COLUMN | No | No | No |

### MODIFY/CHANGE COLUMN type support

Supported: BIT, TINYINT, SMALLINT, MEDIUMINT, INT, BIGINT (all UNSIGNED variants), DATE, DATETIME, TIMESTAMP, TIME, YEAR, CHAR, VARCHAR, TEXT, BINARY, VARBINARY, BLOB, FLOAT, DOUBLE, DECIMAL, NUMERIC, JSON, ENUM, SET.

Not supported: POINT, GEOMETRY.

## Data Type Restrictions

| Data type | As PK | As sort key | As partition key |
|-----------|-------|-------------|-----------------|
| BIT (UNSIGNED) | Yes | Yes | **No** |
| TINYINT-SMALLINT-MEDIUMINT-INT-BIGINT | Yes | Yes | Yes |
| DATE / DATETIME / TIMESTAMP | Yes | Yes | Yes |
| TIME / YEAR | Yes | Yes | **No** |
| CHAR / VARCHAR | Yes | Yes | Yes |
| TEXT / BLOB | Yes | Yes | **No** |
| BINARY / VARBINARY | Yes | Yes | Yes |
| FLOAT / DOUBLE / DECIMAL / NUMERIC | **No** | **No** | **No** |
| JSON / ENUM / SET | **No** | **No** | **No** |
| POINT / GEOMETRY | **No** | **No** | **No** |

## Row-Column Routing

When `ENABLE_COLUMNAR_OPTIMIZER = true`, PolarDB-X provides transparent read-write separation:
- **OLTP queries** (point lookups, small range scans) → row-store (primary table / GSI).
- **OLAP queries** (aggregation, full scans) → columnar store (CCI on columnar read-only instance).
- Routing is automatic based on cost model — no application code changes needed.
- Currently, automatic CCI selection only works on **columnar read-only instances**. On primary instances, use FORCE INDEX to explicitly route to CCI.

## Relationship with GSI

CCI is the columnar version of Clustered GSI:

| Feature | Clustered GSI | CCI |
|---------|--------------|-----|
| Storage format | Row-store | Columnar |
| Best for | Point queries, small range scans | Large scans, aggregations |
| Storage backend | DN local storage | Object storage (lower cost) |
| Data freshness | Real-time (strong consistency) | Eventually consistent (slight delay) |
| Write impact | Distributed transaction overhead | Lower write amplification |
| Snapshot support | No | Yes (AS OF TSO) |
| Use case | OLTP supplementary index | OLAP/HTAP analytics |

Both store all primary table columns by default (clustered). The key difference is storage format and applicable query types.

## Combined with TTL Tables

CCI can be combined with TTL tables for hot/cold data separation:
- **Hot data**: Row-store partitioned table (fast OLTP access).
- **Cold data**: Archived to CCI on object storage (low-cost analytical access).

Use the `polardbx-ttl20` skill for TTL + archive table setup.

## Common Questions

1. **Can I create CCI without specifying sort key?** No. Sort key is required.
2. **Can I create CCI without specifying partition key?** Yes. Primary key is used by default with HASH partitioning.
3. **How to check CCI creation progress?** Use `SHOW DDL;`
4. **How to delete CCI?** `DROP INDEX cci_name ON TABLE tbl_name;`
5. **Can I modify CCI partition key / sort key / partition count after creation?** No. Drop and recreate.
6. **Do I need a columnar read-only instance?** You can create CCI on the primary instance, but querying CCI data is recommended on a columnar read-only instance.
7. **Does cluster resize affect partition count?** No.
