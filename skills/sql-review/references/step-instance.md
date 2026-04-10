# Step 4: Test Instance Setup

## 4.1 Analyze Data Characteristics

Infer data volume and distribution for each table from code:

- LIMIT/OFFSET -> table has at least thousands of rows; ID generation -> random/sequential distribution
- Time field range queries -> continuous growth; status enums / UNIQUE constraints -> field cardinality
- INSERT frequency (fire-and-forget -> high write volume)
- (MyBatis) Mapper method names as hints: `findByUserId` suggests userId is a high-frequency filter; `countByStatus` suggests status-based aggregation
- (MyBatis) Parameters guarded by `<if>` are optional; unguarded WHERE conditions are always present — this helps infer which query paths are most common

Determine mock data volume: config tables 10-50 rows, business tables 5K-50K rows, log tables 100K+ rows.

## 4.2 Create Test Instance

Based on user's database edition choice from Step 1:
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

## 4.3 Create Tables and Populate Mock Data

> All SQL in this step uses the anonymized table/column names from Step 3.

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

3. **Mock data guidelines**: Field values should match enums/formats from code, time fields use random timestamps within recent N days, foreign keys reference existing IDs. See [index-design.md](index-design.md) "Mock Data Generation Rules" for field type mapping and volume inference.

4. **Verify**: `$MYSQL -e "SELECT 't_a3kf' AS t, COUNT(*) FROM t_a3kf UNION ALL ..."`
