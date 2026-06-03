---
title: PolarDB-X TTL 2.0 Cold Data Archiving
---

# PolarDB-X TTL 2.0 Cold Data Archiving

This reference generates correct TTL (Time-to-Live) definitions and archive table statements for PolarDB-X 2.0 Enterprise Edition (AUTO mode). It analyzes table schemas to recommend the optimal archiving strategy (row-based or partition-based) and produces production-ready SQL following best practices.

## Applicable Scope

- **PolarDB-X 2.0 Enterprise Edition** + **AUTO mode database**
- Row-based archiving: version >= `polardb-2.4.0_5.4.19-20240927`
- Partition-based archiving: version >= `polardb-2.5.0_5.4.20-20250328`

## Workflow

When the user requests TTL configuration for a table, follow these steps:

### Step 1: Gather Required Information

Collect the following from the user (ask if missing):

| Information | Required | Notes |
|-------------|----------|-------|
| Table name | Yes | The target table to enable TTL |
| CREATE TABLE statement | Preferred | If unavailable, ask for it or at minimum get column definitions |
| TTL time column | Yes | Must be DATE/DATETIME/TIMESTAMP or INT/BIGINT (Unix timestamp) |
| Data retention period | Yes | How long to keep hot data (e.g., "3 months", "180 days", "2 years") |
| Whether archiving is needed | Yes | If yes, generate archive table; if no, only generate TTL cleanup |
| Timezone | Optional | Default: '+08:00' |

### Step 2: Analyze Table Schema and Recommend Strategy

Based on the table's CREATE TABLE statement, determine the archiving strategy:

```
Decision Flow:
0. Is the table a broadcast table?
   -> Yes: STOP -- Broadcast tables do NOT support TTL definitions.
   
0b. Is the table a single table (non-partitioned)?
   -> Yes: Only ARCHIVE_TYPE = 'ROW' is allowed. Generate row-based archiving SQL.
   -> No (partition table): Continue to step 1.

1. Does the table use a Range partition on the TTL time column (first-level)?
   -> Yes: Check sub-conditions:
     - Does the Range partition contain a MAXVALUE partition?
       -> Yes: MAXVALUE partition must be removed first. Warn the user.
     - Does the table have any GSI/UGSI?
       -> Yes: Cannot use partition-based. Must use ARCHIVE_TYPE = 'ROW'.
     - All checks passed: Recommend ARCHIVE_TYPE = 'PARTITION' (partition-based, first-level)
   -> No: Go to step 2

2. Does the table use a Range subpartition on the TTL time column (second-level template)?
   -> Yes: Check sub-conditions:
     - Does the Range subpartition contain a MAXVALUE subpartition?
       -> Yes: MAXVALUE subpartition must be removed first. Warn the user.
     - Does the table have any GSI/UGSI?
       -> Yes: Cannot use partition-based. Must use ARCHIVE_TYPE = 'ROW'.
     - All checks passed: Recommend ARCHIVE_TYPE = 'SUBPARTITION' (partition-based, second-level)
   -> No: Go to step 3

3. Does the table have any GSI/UGSI defined?
   -> Yes: Must use ARCHIVE_TYPE = 'ROW' (row-based) -- partition-based archiving does not support GSI
   -> No: Go to step 4

4. Can the table be restructured to use Range partition on the TTL time column?
   -> If the user is willing to restructure: Recommend partition-based archiving (ensure NO MAXVALUE partition and NO GSI)
   -> If the user prefers no schema change: Use ARCHIVE_TYPE = 'ROW' (row-based)
```

**Key principles:**
- **If the table can use Range partitioning by time column, prefer partition-based archiving** (faster cleanup, less resource usage). Otherwise, use row-based archiving (no schema change needed).
- **Row-based archiving REQUIRES a local index on the TTL column** -- always check and generate `CREATE INDEX` if missing.
- **Partition-based archiving REQUIRES NO MAXVALUE partition** in the Range definition -- warn user if present.
- **Only row-based archiving supports TTL_FILTER** -- never include TTL_FILTER for partition-based strategies.

### Step 3: Generate TTL Definition SQL

#### Determine TTL_EXPR

Based on the TTL column type:

| Column Type | TTL_EXPR Syntax |
|-------------|-----------------|
| DATETIME | `` TTL_EXPR = `col_name` EXPIRE AFTER N UNIT TIMEZONE 'tz' `` |
| TIMESTAMP | `` TTL_EXPR = `col_name` EXPIRE AFTER N UNIT TIMEZONE 'tz' `` |
| DATE | `` TTL_EXPR = `col_name` EXPIRE AFTER N UNIT TIMEZONE 'tz' `` |
| INT/BIGINT (Unix timestamp, seconds) | `` TTL_EXPR = FROM_UNIXTIME(`col_name`) EXPIRE AFTER N UNIT TIMEZONE 'tz' `` |
| INT/BIGINT (millisecond timestamp) | `` TTL_EXPR = FROM_UNIXTIME(`col_name`/1000) EXPIRE AFTER N UNIT TIMEZONE 'tz' `` |
| INT/BIGINT (non-timestamp, monotonic) | `` TTL_EXPR = `col_name` EXPIRE OVER M PARTITIONS `` |

Where `UNIT` is one of: `DAY`, `MONTH`, `YEAR`.

**Choosing the interval unit (for EXPIRE AFTER strategy):**

| Retention Period | Recommended Unit | Cleanup Granularity |
|-----------------|------------------|---------------------|
| < 1 month | DAY | Only cleans when a full day has expired |
| 1-36 months | MONTH | Only cleans when a full month has expired |
| > 36 months | YEAR | Only cleans when a full year has expired |

#### Determine TTL_PART_INTERVAL

| Retention Period | Recommended Partition Interval |
|-----------------|-------------------------------|
| <= 1 month | `INTERVAL(1, DAY)` or `INTERVAL(7, DAY)` |
| 1-36 months | `INTERVAL(1, MONTH)` |
| > 3 years | `INTERVAL(1, YEAR)` |

#### Determine ARCHIVE_TABLE_PRE_ALLOCATE

| Partition Interval | Recommended Pre-allocate Count |
|-------------------|-------------------------------|
| DAY | 7-14 (pre-build 1-2 weeks) |
| MONTH | 3 (pre-build 3 months) |
| YEAR | 1-2 (pre-build 1-2 years) |

#### Determine ARCHIVE_TABLE_POST_ALLOCATE

| Partition Interval | Recommended Post-allocate Count |
|-------------------|--------------------------------|
| DAY | 60-180 |
| MONTH | 24-48 |
| YEAR | 4-8 |

#### Determine ARCHIVE_TYPE

| Strategy | ARCHIVE_TYPE Value | Requirements |
|----------|-------------------|--------------|
| Row-based | `'ROW'` | No partition constraint; supports GSI; needs local index on TTL column |
| First-level partition | `'PARTITION'` | First-level partition must be Range by TTL column; no GSI allowed |
| Second-level subpartition | `'SUBPARTITION'` | Second-level template subpartition must be Range by TTL column; no GSI allowed |

### Step 4: Generate Complete SQL Statements

Output the following SQL statements in order:

#### For Row-Based Archiving (ARCHIVE_TYPE = 'ROW')

```sql
-- Step 1: Add TTL definition (metadata-only change, no data impact)
ALTER TABLE `{table_name}`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'OFF',
TTL_EXPR = {ttl_expr},
TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
TTL_PART_INTERVAL = {part_interval},
ARCHIVE_TYPE = 'ROW',
ARCHIVE_TABLE_PRE_ALLOCATE = {pre_allocate},
ARCHIVE_TABLE_POST_ALLOCATE = {post_allocate};

-- Step 2: Create local index on TTL column (MANDATORY for row-based cleanup)
-- Required: index must be a single-column index on TTL column, OR a composite index
-- with the TTL column as the FIRST column. Skip this step ONLY if such an index already exists.
CREATE INDEX `idx_{ttl_col}` ON `{table_name}`(`{ttl_col}`);

-- Step 3: Create archive table (async, long-running but non-blocking)
/*+TDDL:cmd_extra(ENABLE_ASYNC_DDL=true, PURE_ASYNC_DDL_MODE=true)*/
CREATE TABLE `{archive_table_name}`
LIKE `{table_name}`
ENGINE = 'Columnar' ARCHIVE_MODE = 'TTL';

-- Step 4: After archive table creation completes, enable cleanup
ALTER TABLE `{table_name}`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'ON';
```

#### For Partition-Based Archiving (ARCHIVE_TYPE = 'PARTITION' or 'SUBPARTITION')

```sql
-- Step 1: Add TTL definition (metadata-only change, no data impact)
ALTER TABLE `{table_name}`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'OFF',
TTL_EXPR = {ttl_expr},
TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
TTL_PART_INTERVAL = {part_interval},
ARCHIVE_TYPE = '{PARTITION|SUBPARTITION}',
ARCHIVE_TABLE_PRE_ALLOCATE = {pre_allocate},
ARCHIVE_TABLE_POST_ALLOCATE = {post_allocate};

-- Step 2: Create archive table (async, long-running but non-blocking)
/*+TDDL:cmd_extra(ENABLE_ASYNC_DDL=true, PURE_ASYNC_DDL_MODE=true)*/
CREATE TABLE `{archive_table_name}`
LIKE `{table_name}`
ENGINE = 'Columnar' ARCHIVE_MODE = 'TTL';

-- Step 3: After archive table creation completes, enable cleanup
ALTER TABLE `{table_name}`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'ON';
```

#### If No Archiving Needed (cleanup only, no archive table)

```sql
-- Step 1: Add TTL definition
ALTER TABLE `{table_name}`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'ON',
TTL_EXPR = {ttl_expr},
TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
ARCHIVE_TYPE = '{ROW|PARTITION|SUBPARTITION}';

-- Step 2 (row-based only): Create local index on TTL column (MANDATORY)
-- Skip ONLY if an appropriate index already exists (TTL column as first column)
CREATE INDEX `idx_{ttl_col}` ON `{table_name}`(`{ttl_col}`);
```

**IMPORTANT: Never include TTL_FILTER for partition-based archiving. TTL_FILTER is ONLY valid for ARCHIVE_TYPE = 'ROW'.**

### Step 5: For Tables Without Partition Definitions

If the user provides a CREATE TABLE without any partition clause, recommend a partitioning scheme:

1. **Identify the primary access pattern** (which column is frequently used in WHERE/JOIN):
   - Use that column as the partition key with `PARTITION BY KEY(col) PARTITIONS N`
   - N is typically 8-32 based on data volume

2. **For row-based archiving** (simplest, no partition change needed):
   - Add `PARTITION BY KEY(primary_key_or_business_key) PARTITIONS N` to the table
   - The existing schema is mostly preserved

3. **For partition-based archiving** (better performance):
   - If business allows, recommend second-level subpartition design:
     ```sql
     PARTITION BY KEY(`business_key`) PARTITIONS N
     SUBPARTITION BY RANGE COLUMNS(`ttl_time_col`) (
       SUBPARTITION sp_xxx VALUES LESS THAN ('yyyy-MM-dd'),
       ...
     )
     ```
   - Generate initial RANGE subpartitions covering the existing data time range plus future pre-allocations

## Complete Examples

### Example 1: Row-Based Archiving -- Existing Hash-Partitioned Table with GSI

Given table:
```sql
CREATE TABLE `order` (
  `id` bigint(20) NOT NULL,
  `cid` bigint(20) NOT NULL,
  `modify_time` datetime DEFAULT CURRENT_TIMESTAMP,
  `status` int DEFAULT 0,
  PRIMARY KEY (`id`),
  GLOBAL INDEX `g_cid`(`cid`) PARTITION BY KEY(`cid`) PARTITIONS 16,
  INDEX `idx_modify_time`(`modify_time`)
)
PARTITION BY KEY(`cid`, `id`)
PARTITIONS 32;
```

Requirements: TTL column = `modify_time`, retain 180 days, need archiving.

Analysis:
- Table has GSI `g_cid` -> Cannot use partition-based archiving
- Must use row-based archiving (ARCHIVE_TYPE = 'ROW')
- Retention 180 days < 36 months -> Use DAY unit
- Partition interval: INTERVAL(1, DAY)
- Already has index on `modify_time` -> No need to create additional index

Generated SQL:
```sql
-- Step 1: Add TTL definition
ALTER TABLE `order`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'OFF',
TTL_EXPR = `modify_time` EXPIRE AFTER 180 DAY TIMEZONE '+08:00',
TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
TTL_PART_INTERVAL = INTERVAL(1, DAY),
ARCHIVE_TYPE = 'ROW',
ARCHIVE_TABLE_PRE_ALLOCATE = 14,
ARCHIVE_TABLE_POST_ALLOCATE = 180;

-- Step 2: Local index already exists (idx_modify_time), skip this step

-- Step 3: Create archive table
/*+TDDL:cmd_extra(ENABLE_ASYNC_DDL=true, PURE_ASYNC_DDL_MODE=true)*/
CREATE TABLE `order_arc`
LIKE `order`
ENGINE = 'Columnar' ARCHIVE_MODE = 'TTL';

-- Step 4: After archive table creation completes, enable cleanup
ALTER TABLE `order`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'ON';
```

### Example 2: Partition-Based Archiving -- Table with Range Subpartition

Given table:
```sql
CREATE TABLE `trade_log` (
  `trade_id` bigint NOT NULL AUTO_INCREMENT,
  `dt_col` datetime NOT NULL,
  `payload` text,
  PRIMARY KEY (`trade_id`),
  KEY `idx_dt` (`dt_col`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
PARTITION BY KEY(`trade_id`) PARTITIONS 128
SUBPARTITION BY RANGE COLUMNS(`dt_col`) (
  SUBPARTITION sp20250601 VALUES LESS THAN ('2025-06-01'),
  SUBPARTITION sp20250602 VALUES LESS THAN ('2025-06-02'),
  SUBPARTITION sp20250603 VALUES LESS THAN ('2025-06-03')
);
```

Requirements: TTL column = `dt_col`, retain 3 days, need archiving.

Analysis:
- Table has second-level Range subpartition on `dt_col` (the TTL column)
- No GSI defined
- Use partition-based archiving: ARCHIVE_TYPE = 'SUBPARTITION'
- Retention 3 days -> DAY unit
- Partition interval: INTERVAL(1, DAY)

Generated SQL:
```sql
-- Step 1: Add TTL definition
ALTER TABLE `trade_log`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'OFF',
TTL_EXPR = `dt_col` EXPIRE AFTER 3 DAY TIMEZONE '+08:00',
TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
TTL_PART_INTERVAL = INTERVAL(1, DAY),
ARCHIVE_TYPE = 'SUBPARTITION',
ARCHIVE_TABLE_PRE_ALLOCATE = 7,
ARCHIVE_TABLE_POST_ALLOCATE = 60;

-- Step 2: Create archive table
/*+TDDL:cmd_extra(ENABLE_ASYNC_DDL=true, PURE_ASYNC_DDL_MODE=true)*/
CREATE TABLE `trade_log_arc`
LIKE `trade_log`
ENGINE = 'Columnar' ARCHIVE_MODE = 'TTL';

-- Step 3: After archive table creation completes, enable cleanup
ALTER TABLE `trade_log`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'ON';
```

### Example 3: Partition-Based Archiving -- First-Level Range Partition

Given table:
```sql
CREATE TABLE `metrics` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `metric_time` datetime NOT NULL,
  `value` double,
  PRIMARY KEY (`id`, `metric_time`)
)
PARTITION BY RANGE COLUMNS(`metric_time`) (
  PARTITION p20250401 VALUES LESS THAN ('2025-04-01'),
  PARTITION p20250501 VALUES LESS THAN ('2025-05-01'),
  PARTITION p20250601 VALUES LESS THAN ('2025-06-01')
);
```

Requirements: TTL column = `metric_time`, retain 1 month, need archiving.

Analysis:
- Table has first-level Range partition on `metric_time` (the TTL column)
- No GSI defined
- Use partition-based archiving: ARCHIVE_TYPE = 'PARTITION'
- Retention 1 month -> MONTH unit
- Partition interval: INTERVAL(1, MONTH)

Generated SQL:
```sql
-- Step 1: Add TTL definition
ALTER TABLE `metrics`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'OFF',
TTL_EXPR = `metric_time` EXPIRE AFTER 1 MONTH TIMEZONE '+08:00',
TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
TTL_PART_INTERVAL = INTERVAL(1, MONTH),
ARCHIVE_TYPE = 'PARTITION',
ARCHIVE_TABLE_PRE_ALLOCATE = 3,
ARCHIVE_TABLE_POST_ALLOCATE = 48;

-- Step 2: Create archive table
/*+TDDL:cmd_extra(ENABLE_ASYNC_DDL=true, PURE_ASYNC_DDL_MODE=true)*/
CREATE TABLE `metrics_arc`
LIKE `metrics`
ENGINE = 'Columnar' ARCHIVE_MODE = 'TTL';

-- Step 3: After archive table creation completes, enable cleanup
ALTER TABLE `metrics`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'ON';
```

### Example 4: Integer TTL Column (Unix Timestamp)

Given table:
```sql
CREATE TABLE `event_log` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `created_ts` bigint NOT NULL COMMENT 'Unix timestamp in seconds',
  `data` varchar(500),
  PRIMARY KEY (`id`)
)
PARTITION BY KEY(`id`) PARTITIONS 8;
```

Requirements: TTL column = `created_ts` (Unix timestamp, seconds), retain 7 days, need archiving.

Analysis:
- Table is KEY-partitioned, not Range-partitioned on TTL column
- No GSI defined, but table is not Range-partitioned by TTL column
- Use row-based archiving: ARCHIVE_TYPE = 'ROW'
- Use FROM_UNIXTIME() wrapper for integer timestamp column

Generated SQL:
```sql
-- Step 1: Add TTL definition
ALTER TABLE `event_log`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'OFF',
TTL_EXPR = FROM_UNIXTIME(`created_ts`) EXPIRE AFTER 7 DAY TIMEZONE '+08:00',
TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
TTL_PART_INTERVAL = INTERVAL(1, DAY),
ARCHIVE_TYPE = 'ROW',
ARCHIVE_TABLE_PRE_ALLOCATE = 7,
ARCHIVE_TABLE_POST_ALLOCATE = 60;

-- Step 2: Create local index on TTL column
CREATE INDEX `idx_created_ts` ON `event_log`(`created_ts`);

-- Step 3: Create archive table
/*+TDDL:cmd_extra(ENABLE_ASYNC_DDL=true, PURE_ASYNC_DDL_MODE=true)*/
CREATE TABLE `event_log_arc`
LIKE `event_log`
ENGINE = 'Columnar' ARCHIVE_MODE = 'TTL';

-- Step 4: After archive table creation completes, enable cleanup
ALTER TABLE `event_log`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'ON';
```

### Example 5: Integer TTL Column (Millisecond Timestamp)

Given table:
```sql
CREATE TABLE `user_action` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `action_time_ms` bigint NOT NULL COMMENT 'Java timestamp in milliseconds',
  `user_id` bigint,
  `action` varchar(64),
  PRIMARY KEY (`id`)
)
PARTITION BY KEY(`id`) PARTITIONS 16;
```

Requirements: TTL column = `action_time_ms` (millisecond timestamp), retain 30 days, need archiving.

Generated SQL:
```sql
-- Step 1: Add TTL definition
ALTER TABLE `user_action`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'OFF',
TTL_EXPR = FROM_UNIXTIME(`action_time_ms`/1000) EXPIRE AFTER 30 DAY TIMEZONE '+08:00',
TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
TTL_PART_INTERVAL = INTERVAL(1, DAY),
ARCHIVE_TYPE = 'ROW',
ARCHIVE_TABLE_PRE_ALLOCATE = 14,
ARCHIVE_TABLE_POST_ALLOCATE = 60;

-- Step 2: Create local index on TTL column
CREATE INDEX `idx_action_time_ms` ON `user_action`(`action_time_ms`);

-- Step 3: Create archive table
/*+TDDL:cmd_extra(ENABLE_ASYNC_DDL=true, PURE_ASYNC_DDL_MODE=true)*/
CREATE TABLE `user_action_arc`
LIKE `user_action`
ENGINE = 'Columnar' ARCHIVE_MODE = 'TTL';

-- Step 4: After archive table creation completes, enable cleanup
ALTER TABLE `user_action`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'ON';
```

### Example 6: Table Without Partition -- Recommend Full Schema

Given table (no partition definition):
```sql
CREATE TABLE `access_log` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `user_id` bigint NOT NULL,
  `access_time` datetime NOT NULL,
  `url` varchar(2048),
  `status_code` int,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Requirements: TTL column = `access_time`, retain 90 days, need archiving.

Analysis:
- Table has no partition definition -- must add one (PolarDB-X requires partition tables for TTL)
- Option A (row-based, minimal change): Add KEY partition on primary key
- Option B (partition-based, better cleanup): Add KEY + Range subpartition design

**Recommendation A -- Row-Based (minimal schema change):**

```sql
-- Recreate table with partition (or ALTER to add partition if supported)
CREATE TABLE `access_log` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `user_id` bigint NOT NULL,
  `access_time` datetime NOT NULL,
  `url` varchar(2048),
  `status_code` int,
  PRIMARY KEY (`id`),
  INDEX `idx_access_time`(`access_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
PARTITION BY KEY(`id`) PARTITIONS 16;

-- Add TTL definition
ALTER TABLE `access_log`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'OFF',
TTL_EXPR = `access_time` EXPIRE AFTER 90 DAY TIMEZONE '+08:00',
TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
TTL_PART_INTERVAL = INTERVAL(1, DAY),
ARCHIVE_TYPE = 'ROW',
ARCHIVE_TABLE_PRE_ALLOCATE = 14,
ARCHIVE_TABLE_POST_ALLOCATE = 90;

-- Create archive table
/*+TDDL:cmd_extra(ENABLE_ASYNC_DDL=true, PURE_ASYNC_DDL_MODE=true)*/
CREATE TABLE `access_log_arc`
LIKE `access_log`
ENGINE = 'Columnar' ARCHIVE_MODE = 'TTL';

-- After archive table creation completes, enable cleanup
ALTER TABLE `access_log`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'ON';
```

**Recommendation B -- Partition-Based (better cleanup performance):**

```sql
-- Recreate table with Range subpartition on TTL column
CREATE TABLE `access_log` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `user_id` bigint NOT NULL,
  `access_time` datetime NOT NULL,
  `url` varchar(2048),
  `status_code` int,
  PRIMARY KEY (`id`, `access_time`),
  INDEX `idx_access_time`(`access_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
PARTITION BY KEY(`id`) PARTITIONS 16
SUBPARTITION BY RANGE COLUMNS(`access_time`) (
  SUBPARTITION sp20250501 VALUES LESS THAN ('2025-05-01'),
  SUBPARTITION sp20250601 VALUES LESS THAN ('2025-06-01'),
  SUBPARTITION sp20250701 VALUES LESS THAN ('2025-07-01')
);

-- Add TTL definition
ALTER TABLE `access_log`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'OFF',
TTL_EXPR = `access_time` EXPIRE AFTER 90 DAY TIMEZONE '+08:00',
TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
TTL_PART_INTERVAL = INTERVAL(1, MONTH),
ARCHIVE_TYPE = 'SUBPARTITION',
ARCHIVE_TABLE_PRE_ALLOCATE = 3,
ARCHIVE_TABLE_POST_ALLOCATE = 6;

-- Create archive table
/*+TDDL:cmd_extra(ENABLE_ASYNC_DDL=true, PURE_ASYNC_DDL_MODE=true)*/
CREATE TABLE `access_log_arc`
LIKE `access_log`
ENGINE = 'Columnar' ARCHIVE_MODE = 'TTL';

-- After archive table creation completes, enable cleanup
ALTER TABLE `access_log`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'ON';
```

### Example 7: TIMESTAMP Column with RANGE(UNIX_TIMESTAMP()) Partition

Given table:
```sql
CREATE TABLE `sensor_data` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `ts` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `sensor_id` int,
  `value` double,
  PRIMARY KEY (`id`, `ts`)
)
PARTITION BY RANGE(UNIX_TIMESTAMP(`ts`)) (
  PARTITION p20250401 VALUES LESS THAN (UNIX_TIMESTAMP('2025-04-01 00:00:00')),
  PARTITION p20250501 VALUES LESS THAN (UNIX_TIMESTAMP('2025-05-01 00:00:00')),
  PARTITION p20250601 VALUES LESS THAN (UNIX_TIMESTAMP('2025-06-01 00:00:00'))
);
```

Requirements: TTL column = `ts`, retain 1 month, need archiving.

Analysis:
- Table has first-level Range partition using `UNIX_TIMESTAMP(ts)` -- this is a TIMESTAMP-type Range partition
- Use partition-based archiving: ARCHIVE_TYPE = 'PARTITION'

Generated SQL:
```sql
-- Step 1: Add TTL definition
ALTER TABLE `sensor_data`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'OFF',
TTL_EXPR = `ts` EXPIRE AFTER 1 MONTH TIMEZONE '+08:00',
TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
TTL_PART_INTERVAL = INTERVAL(1, MONTH),
ARCHIVE_TYPE = 'PARTITION',
ARCHIVE_TABLE_PRE_ALLOCATE = 3,
ARCHIVE_TABLE_POST_ALLOCATE = 48;

-- Step 2: Create archive table
/*+TDDL:cmd_extra(ENABLE_ASYNC_DDL=true, PURE_ASYNC_DDL_MODE=true)*/
CREATE TABLE `sensor_data_arc`
LIKE `sensor_data`
ENGINE = 'Columnar' ARCHIVE_MODE = 'TTL';

-- Step 3: After archive table creation completes, enable cleanup
ALTER TABLE `sensor_data`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'ON';
```

### Example 8: EXPIRE OVER Strategy (Integer Non-Timestamp Column)

Given table:
```sql
CREATE TABLE `batch_records` (
  `batch_id` bigint NOT NULL,
  `data` varchar(1000),
  PRIMARY KEY (`batch_id`)
)
PARTITION BY RANGE COLUMNS(`batch_id`) (
  PARTITION p1 VALUES LESS THAN (1000000),
  PARTITION p2 VALUES LESS THAN (2000000),
  PARTITION p3 VALUES LESS THAN (3000000),
  PARTITION p4 VALUES LESS THAN (4000000)
);
```

Requirements: TTL column = `batch_id` (monotonically increasing integer, not a timestamp), keep latest 3 partitions, need archiving.

Analysis:
- TTL column is a non-timestamp integer -> Use EXPIRE OVER strategy
- Table has first-level Range partition on `batch_id`
- ARCHIVE_TYPE = 'PARTITION'

Generated SQL:
```sql
-- Step 1: Add TTL definition
ALTER TABLE `batch_records`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'OFF',
TTL_EXPR = `batch_id` EXPIRE OVER 3 PARTITIONS,
TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
TTL_PART_INTERVAL = INTERVAL(1000000, NUMBER),
ARCHIVE_TYPE = 'PARTITION',
ARCHIVE_TABLE_PRE_ALLOCATE = 3,
ARCHIVE_TABLE_POST_ALLOCATE = 48;

-- Step 2: Create archive table
/*+TDDL:cmd_extra(ENABLE_ASYNC_DDL=true, PURE_ASYNC_DDL_MODE=true)*/
CREATE TABLE `batch_records_arc`
LIKE `batch_records`
ENGINE = 'Columnar' ARCHIVE_MODE = 'TTL';

-- Step 3: After archive table creation completes, enable cleanup
ALTER TABLE `batch_records`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'ON';
```

## Important Notes and Constraints

### Constraints

#### Table Type Constraints
1. **Broadcast tables in AUTO mode do NOT support TTL definitions** -- TTL cannot be applied to broadcast tables at all. **Do NOT recommend converting a broadcast table to another table type (e.g., SINGLE) solely for the purpose of enabling TTL.**
2. **Single tables in AUTO mode DO support TTL** using row-based archiving (`ARCHIVE_TYPE = 'ROW'`) — partition-based archiving is not available for single tables, but row-based TTL works fine. **Do NOT tell users that TTL requires partitioned tables.**
3. **DRDS mode tables do NOT support TTL 2.0** -- only AUTO mode database tables are supported.
4. **Tables using Local Partition (TTL 1.0) cannot use TTL 2.0** -- they are mutually exclusive. **LOCAL PARTITION (TTL 1.0) is DEPRECATED — do NOT recommend it for new tables.** If a user has an existing LOCAL PARTITION table, guide them to migrate to TTL 2.0 (see the "Migrating from TTL 1.0" section below).

#### Row-Based Archiving Constraints
5. **Row-based TTL tables MUST have a local index on the TTL time column** -- the index must either be a single-column index on the TTL column, or a composite index with the TTL column as the FIRST column. Without this index, row-based cleanup performance will be severely degraded. Always generate a `CREATE INDEX` statement if the TTL column lacks such an index.
6. **Only row-based archiving supports TTL_FILTER** -- the `TTL_FILTER = COND_EXPR(...)` custom filter condition is exclusively available for row-based archiving (ARCHIVE_TYPE = 'ROW'). Do NOT include TTL_FILTER when generating SQL for partition-based archiving.

#### Partition-Based Archiving Constraints
7. **Partition-based archiving does NOT support GSI/UGSI** -- if the table has any global secondary indexes, partition-based archiving cannot be used. Must use row-based archiving instead.
8. **For partition-based archiving, the Range partition MUST NOT contain a MAXVALUE partition** -- the Range partition definition cannot contain a `VALUES LESS THAN (MAXVALUE)` partition. If the existing table has a MAXVALUE partition, it must be removed (e.g., via split or reorganize) before enabling partition-based archiving. The TTL mechanism needs to add new Range partitions beyond the current maximum bound, which is impossible if MAXVALUE already exists.
9. **Partition-based archiving requires the TTL column to be the Range partition column** -- if the table's Range partition (first-level or second-level subpartition) is not on the TTL column, partition-based archiving cannot be used.

#### Archive Table Constraints
10. **One TTL table can only have one archive table** -- one-to-one binding relationship.
11. **If an archive table exists, the TTL definition cannot be removed directly** -- must drop the archive table first using `/*+TDDL:CMD_EXTRA(TTL_FORBID_DROP_TTL_TBL_WITH_ARC_CCI=false)*/ DROP TABLE {archive_table}`.
12. **TTL_CLEANUP should remain 'OFF' until archive table creation completes** -- to avoid permanent deletion of data that should have been archived.

#### General Constraints
13. **Adding TTL definition is metadata-only** -- no data changes, no impact on online reads/writes.
14. **TTL cleanup tasks run within the maintenance window** -- default window is 02:00-05:00 (UTC+8). Tasks submitted outside this window will queue until the window opens.
15. **CREATE TABLE with inline TTL definition: ARCHIVE_TABLE_NAME must be empty** -- PolarDB-X supports specifying TTL definition directly in the CREATE TABLE statement, but the `ARCHIVE_TABLE_NAME` field must be left empty (or omitted). The archive table must be created separately via `CREATE TABLE ... LIKE ... ENGINE='Columnar' ARCHIVE_MODE='TTL'` after the TTL table is created.
16. **Removing TTL definition requires dropping the archive table first** -- if the TTL table has a bound archive table, you MUST drop the archive table before executing `ALTER TABLE ... REMOVE TTL`. The drop sequence is: (1) drop archive table with hint, (2) then remove TTL definition.

### Naming Conventions

- Archive table name: `{original_table_name}_arc` (recommended convention)
- Internally, the archive table is currently implemented as a view pointing to the archive CCI (Clustered Columnar Index)

### Querying Archive Data

After archive table creation, query historical data via:
```sql
SELECT * FROM {archive_table_name} WHERE {conditions};
```

Note: Archive table queries pull data from remote OSS, which may be slower without local cache. Recommended for low-frequency analytical queries. For frequent archive queries, use the columnar read-only instance.

### TTL_FILTER (Row-Based Archiving ONLY -- NOT supported for partition-based)

For row-based archiving (ARCHIVE_TYPE = 'ROW') ONLY, you can add extra filter conditions to only clean data matching specific business criteria:

```sql
ALTER TABLE `{table_name}`
MODIFY TTL
SET
TTL_FILTER = COND_EXPR(`status` = 1);
```

This ensures only expired data with `status = 1` gets cleaned. The filter is ANDed with the time condition.

**WARNING: TTL_FILTER is ONLY valid for ARCHIVE_TYPE = 'ROW'. Never include TTL_FILTER for partition-based archiving -- partition-level cleanup operates by dropping entire partitions and cannot apply row-level filter conditions.**

### Management Operations

```sql
-- View TTL definition
SELECT * FROM INFORMATION_SCHEMA.TTL_INFO
WHERE TABLE_SCHEMA = '{db}' AND TABLE_NAME = '{table}';

-- View TTL task status
SELECT * FROM INFORMATION_SCHEMA.TTL_SCHEDULE
WHERE TABLE_SCHEMA = '{db}' AND TABLE_NAME = '{table}';

-- Manually trigger cleanup
ALTER TABLE `{table_name}` CLEANUP EXPIRED DATA ASYNC=TRUE;

-- Pause cleanup task
PAUSE DDL {job_id};

-- View running DDL jobs
SHOW FULL DDL;

-- Remove TTL definition (must drop archive table first if exists)
/*+TDDL:CMD_EXTRA(TTL_FORBID_DROP_TTL_TBL_WITH_ARC_CCI=false)*/
DROP TABLE {archive_table_name};
ALTER TABLE `{table_name}` REMOVE TTL;

-- Adjust cleanup speed limit (per DN, default 1000 rows/s)
SET GLOBAL TTL_ENABLE_CLEANUP_ROWS_SPEED_LIMIT = 10000;

-- Adjust maintenance window
SET GLOBAL MAINTENANCE_TIME_START = '01:00';
SET GLOBAL MAINTENANCE_TIME_END = '06:00';
```

## Row-Based vs Partition-Based Comparison

| Aspect | Row-Based (ARCHIVE_TYPE='ROW') | Partition-Based (ARCHIVE_TYPE='PARTITION'/'SUBPARTITION') |
|--------|-------------------------------|----------------------------------------------------------|
| Schema change | None required | Must use Range partition on TTL column |
| GSI support | Yes | No |
| Cleanup method | DELETE DML | DROP (SUB)PARTITION DDL |
| Cleanup speed | ~10M-20M rows/hour | Very fast (instant partition drop) |
| Resource usage | 10-20% CPU/IO during cleanup | Minimal |
| Table fragmentation | May produce fragmentation | No fragmentation |
| Binlog generation | Generates DELETE binlog | Minimal binlog |
| Custom filter | Supports TTL_FILTER | Not supported |
| Table locking | No lock | No lock |

## Inline TTL Definition in CREATE TABLE

PolarDB-X supports specifying TTL definition directly in a CREATE TABLE statement. However, the `ARCHIVE_TABLE_NAME` must be empty -- the archive table must be created separately afterward.

Example:
```sql
CREATE TABLE `my_new_ttl_table` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `gmt_modified` datetime NOT NULL,
  `data` varchar(500),
  PRIMARY KEY (`id`),
  INDEX `idx_gmt_modified`(`gmt_modified`)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4
TTL = TTL_DEFINITION(
  TTL_ENABLE = 'OFF',
  TTL_EXPR = `gmt_modified` EXPIRE AFTER 3 MONTH TIMEZONE '+08:00',
  TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
  ARCHIVE_TYPE = 'ROW',
  ARCHIVE_TABLE_PRE_ALLOCATE = 3,
  ARCHIVE_TABLE_POST_ALLOCATE = 48
)
PARTITION BY KEY(`id`) PARTITIONS 16;
```

**IMPORTANT: When using inline TTL definition in CREATE TABLE, do NOT specify `ARCHIVE_TABLE_NAME`. The archive table must be created separately via:**
```sql
/*+TDDL:cmd_extra(ENABLE_ASYNC_DDL=true, PURE_ASYNC_DDL_MODE=true)*/
CREATE TABLE `my_new_ttl_table_arc`
LIKE `my_new_ttl_table`
ENGINE = 'Columnar' ARCHIVE_MODE = 'TTL';
```

## Auto-Add Range Partitions (Add Only, No Cleanup)

If the user wants to create a new Range-partitioned table and configure automatic partition pre-allocation (only adding new partitions, NOT cleaning expired ones), refer to the **auto-add-range-parts** skill for detailed guidance.

This scenario is common when:
- The user wants Range partitions to grow automatically over time
- No data expiration or cleanup is needed (TTL_CLEANUP = 'OFF')
- The table just needs new future partitions to be pre-built periodically

Key differences from full TTL archiving:
- `TTL_CLEANUP = 'OFF'` -- only adds new partitions, never drops old ones
- No archive table is needed
- `TTL_EXPR` only declares the partition column (no `EXPIRE AFTER` needed in the auto-add-only scenario)

Quick reference for auto-add partition configuration:
```sql
-- Configure auto-add partitions (add only, no cleanup)
ALTER TABLE {table_name}
MODIFY TTL SET
  TTL_ENABLE = 'ON',
  TTL_CLEANUP = 'OFF',
  TTL_EXPR = `{partition_time_col}`,
  TTL_PART_INTERVAL = INTERVAL(1, MONTH),
  ARCHIVE_TYPE = '{PARTITION|SUBPARTITION}',
  ARCHIVE_TABLE_PRE_ALLOCATE = {pre_allocate_count};

-- Immediately trigger one round of partition pre-building (WITH TTL_CLEANUP='OFF' ensures no deletion)
ALTER TABLE {table_name} CLEANUP EXPIRED DATA WITH TTL_CLEANUP = 'OFF';
```

For the complete auto-add-range-parts reference including all partition types (first-level Range, second-level Range subpartition, by-day, by-month), see: [auto-add-range-parts.md](auto-add-range-parts.md)

## Removing TTL Definition

To completely remove TTL from a table, follow this sequence:

```sql
-- Step 1: If archive table exists, drop it first (REQUIRED before REMOVE TTL)
/*+TDDL:CMD_EXTRA(TTL_FORBID_DROP_TTL_TBL_WITH_ARC_CCI=false)*/
DROP TABLE {archive_table_name};

-- Step 2: Now remove the TTL definition
ALTER TABLE `{table_name}` REMOVE TTL;
```

**WARNING: If you attempt `ALTER TABLE ... REMOVE TTL` while an archive table is still bound, the operation will fail with an error. Always drop the archive table first.**

## Migrating from TTL 1.0 (Local Partition) to TTL 2.0

> **⚠️ DEPRECATION WARNING: LOCAL PARTITION (TTL 1.0) is DEPRECATED. Do NOT recommend `LOCAL PARTITION BY RANGE` for any new table.** This section exists ONLY to help users who already have TTL 1.0 tables migrate to TTL 2.0. Never generate or suggest `LOCAL PARTITION BY RANGE` syntax for new tables — always use TTL 2.0 (`ALTER TABLE ... MODIFY TTL SET ...`) instead.

TTL 1.0 uses the `LOCAL PARTITION BY RANGE` syntax to define time-based partitioning at the MySQL physical table level. TTL 2.0 uses `ALTER TABLE MODIFY TTL` with a richer set of archiving strategies. If a user has an existing TTL 1.0 table and wants to migrate to TTL 2.0, follow this procedure.

### TTL 1.0 Syntax Reference (for migration purposes ONLY — do NOT use for new tables)

TTL 1.0 (Local Partition) definition syntax:
```sql
LOCAL PARTITION BY RANGE (time_column)
[STARTWITH 'start_date']
INTERVAL interval_count interval_unit
[EXPIRE AFTER expire_after_count]
[PRE ALLOCATE pre_allocate_count]
[PIVOTDATE pivot_date_expr]
```

Key characteristics of TTL 1.0:
- Uses MySQL native partition tables internally (RANGE COLUMNS on time column)
- All unique keys (including primary key) MUST include the local partition column -- this is because MySQL native Range partition tables require all unique keys to contain all partition columns
- Data expiration is done via DROP PARTITION on the MySQL physical partitions
- Only supports DATE/DATETIME type partition columns

### Migration Workflow

#### Step 0: Analyze TTL 1.0 Definition and Prepare TTL 2.0 Definition

Extract the following from the existing TTL 1.0 definition:
- **Time column**: The `RANGE (time_column)` column name
- **Interval**: `INTERVAL interval_count interval_unit` -> maps to `TTL_PART_INTERVAL`
- **Expire after**: `EXPIRE AFTER expire_after_count` -> compute retention period as `expire_after_count * interval`
- **Pre allocate**: `PRE ALLOCATE pre_allocate_count` -> maps to `ARCHIVE_TABLE_PRE_ALLOCATE`

**Mapping rules:**

| TTL 1.0 Parameter | TTL 2.0 Equivalent |
|-------------------|-------------------|
| `RANGE (col)` | `TTL_EXPR = \`col\` EXPIRE AFTER N UNIT TIMEZONE '+08:00'` |
| `INTERVAL 1 MONTH` | `TTL_PART_INTERVAL = INTERVAL(1, MONTH)` |
| `INTERVAL 6 MONTH` | `TTL_PART_INTERVAL = INTERVAL(6, MONTH)` (or choose `INTERVAL(1, MONTH)` following the retention-based recommendations in the Determine TTL_PART_INTERVAL section) |
| `EXPIRE AFTER 12` (with INTERVAL 1 MONTH) | `EXPIRE AFTER 12 MONTH` (retention = 12 * 1 month) |
| `EXPIRE AFTER 4` (with INTERVAL 6 MONTH) | `EXPIRE AFTER 24 MONTH` (retention = 4 * 6 months) |
| `PRE ALLOCATE 6` | `ARCHIVE_TABLE_PRE_ALLOCATE = 6` |

Based on the original table's partition scheme (KEY partition on business key), determine the appropriate archiving strategy:
- If the table is KEY-partitioned (most common for TTL 1.0 tables): use **row-based archiving** (ARCHIVE_TYPE = 'ROW')
- If the user wants to restructure to Range partition: use **partition-based archiving**

#### Step 1: Remove TTL 1.0 Definition

```sql
ALTER TABLE `{table_name}` REMOVE LOCAL PARTITIONING;
```

This converts the table back to a normal table without Local Partition, removing all MySQL-level Range partitions.

#### Step 2: (Optional) Remove TTL Column from Primary Key

TTL 1.0 uses MySQL native Range partition tables internally. MySQL requires that all unique keys (including the primary key) must contain all partition columns. Therefore, TTL 1.0 tables are forced to include the time column in the primary key. After removing Local Partition, this constraint no longer exists, and the time column in the primary key may no longer be needed (e.g., `PRIMARY KEY (id, gmt_modified)`).

**Ask the user**: "The primary key currently includes the TTL time column. Would you like to remove it from the primary key to restore the original schema?"

If yes:
```sql
ALTER TABLE `{table_name}` DROP PRIMARY KEY, ADD PRIMARY KEY (`{original_pk_columns}`);
```

For example, if the original PK was `PRIMARY KEY (id, gmt_modified)` and user wants to revert to `PRIMARY KEY (id)`:
```sql
ALTER TABLE `{table_name}` DROP PRIMARY KEY, ADD PRIMARY KEY (`id`);
```

**NOTE**: This is an Online DDL operation but may take time on large tables. The user should evaluate the impact.

#### Step 3: Apply TTL 2.0 Definition

Generate and execute the TTL 2.0 ALTER statement based on the mapping from Step 0:

```sql
ALTER TABLE `{table_name}`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'OFF',
TTL_EXPR = `{ttl_col}` EXPIRE AFTER {retention_value} {retention_unit} TIMEZONE '+08:00',
TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
TTL_PART_INTERVAL = INTERVAL({interval_value}, {interval_unit}),
ARCHIVE_TYPE = 'ROW',
ARCHIVE_TABLE_PRE_ALLOCATE = {pre_allocate},
ARCHIVE_TABLE_POST_ALLOCATE = {post_allocate};
```

Then follow the standard TTL 2.0 workflow (create local index if needed, create archive table if needed, enable cleanup).

### Complete Migration Example

Given TTL 1.0 table:
```sql
CREATE TABLE `t_order` (
  `id` bigint NOT NULL,
  `gmt_modified` datetime NOT NULL,
  `user_id` bigint,
  `amount` decimal(10,2),
  PRIMARY KEY (`id`, `gmt_modified`),
  KEY `idx_user` (`user_id`)
) ENGINE = InnoDB
PARTITION BY KEY(`id`) PARTITIONS 16
LOCAL PARTITION BY RANGE (`gmt_modified`)
STARTWITH '2023-01-01'
INTERVAL 1 MONTH
EXPIRE AFTER 12
PRE ALLOCATE 6;
```

**Analysis:**
- TTL column: `gmt_modified`
- Interval: 1 MONTH
- Expire after: 12 intervals = 12 months retention
- Pre allocate: 6 months
- Primary key includes TTL column: `PRIMARY KEY (id, gmt_modified)` -> user may want to simplify to `PRIMARY KEY (id)`

**Generated Migration Script:**
```sql
-- ============================================================
-- Migration from TTL 1.0 (Local Partition) to TTL 2.0
-- Table: t_order
-- ============================================================

-- Step 1: Remove TTL 1.0 Local Partition definition
ALTER TABLE `t_order` REMOVE LOCAL PARTITIONING;

-- Step 2: (Optional) Remove TTL column from primary key
-- ONLY execute this step if you explicitly confirmed removing the TTL column from the primary key.
-- Original PK: PRIMARY KEY (id, gmt_modified)
-- New PK: PRIMARY KEY (id)
-- NOTE: Evaluate impact before executing on large tables
ALTER TABLE `t_order` DROP PRIMARY KEY, ADD PRIMARY KEY (`id`);

-- Step 3: Ensure local index exists on TTL column (required for row-based cleanup)
CREATE INDEX `idx_gmt_modified` ON `t_order`(`gmt_modified`);

-- Step 4: Apply TTL 2.0 definition
ALTER TABLE `t_order`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'OFF',
TTL_EXPR = `gmt_modified` EXPIRE AFTER 12 MONTH TIMEZONE '+08:00',
TTL_JOB = CRON '0 0 2 */1 * ? *' TIMEZONE '+08:00',
TTL_PART_INTERVAL = INTERVAL(1, MONTH),
ARCHIVE_TYPE = 'ROW',
ARCHIVE_TABLE_PRE_ALLOCATE = 6,
ARCHIVE_TABLE_POST_ALLOCATE = 48;

-- Step 5: (Optional) Create archive table if archiving is needed
/*+TDDL:cmd_extra(ENABLE_ASYNC_DDL=true, PURE_ASYNC_DDL_MODE=true)*/
CREATE TABLE `t_order_arc`
LIKE `t_order`
ENGINE = 'Columnar' ARCHIVE_MODE = 'TTL';

-- Step 6: After archive table creation completes, enable cleanup
ALTER TABLE `t_order`
MODIFY TTL
SET
TTL_ENABLE = 'ON',
TTL_CLEANUP = 'ON';
```

### Migration Notes

- **Execution order matters**: Always remove Local Partition FIRST, then modify primary key, then add TTL 2.0 definition.
- **Primary key modification is optional**: Ask the user whether they want to remove the TTL column from the primary key. Some applications may depend on the composite primary key.
- **Index on TTL column**: After removing Local Partition, the MySQL-level Range partition index is gone. For row-based cleanup to perform well, a local index on the TTL column is mandatory.
- **TTL 1.0 and TTL 2.0 are mutually exclusive**: A table cannot have both definitions simultaneously. The `REMOVE LOCAL PARTITIONING` must be done first before any TTL 2.0 definition can be applied.
