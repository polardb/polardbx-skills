# SQL Review Example

## Example: E-commerce Order System (Node.js + mysql2)

Assume an e-commerce backend project using TypeScript + mysql2 driver.

### Step 1: Scan Results

After scanning the codebase, the SQL inventory:

**DDL Statements:**

| # | Table | Type | File | Line |
|---|-------|------|------|------|
| D1 | orders | CREATE TABLE | src/db/schema.ts | 12 |
| D2 | order_items | CREATE TABLE | src/db/schema.ts | 28 |
| D3 | customers | CREATE TABLE | src/db/schema.ts | 45 |

```sql
-- D1
CREATE TABLE orders (
  id          BIGINT AUTO_INCREMENT PRIMARY KEY,
  customer_id BIGINT NOT NULL,
  status      VARCHAR(20) NOT NULL DEFAULT 'pending',
  total       DECIMAL(10,2) NOT NULL,
  created_at  DATETIME NOT NULL,
  updated_at  DATETIME NOT NULL,
  INDEX idx_status (status),
  INDEX idx_customer_id (customer_id)
);

-- D2
CREATE TABLE order_items (
  id        BIGINT AUTO_INCREMENT PRIMARY KEY,
  order_id  BIGINT NOT NULL,
  product_id BIGINT NOT NULL,
  quantity  INT NOT NULL,
  price     DECIMAL(10,2) NOT NULL,
  INDEX idx_order_id (order_id)
);

-- D3
CREATE TABLE customers (
  id         BIGINT AUTO_INCREMENT PRIMARY KEY,
  email      VARCHAR(255) NOT NULL UNIQUE,
  name       VARCHAR(100) NOT NULL,
  created_at DATETIME NOT NULL
);
```

**Query Statements:**

| # | Type | Tables | File | Line | Key Features |
|---|------|--------|------|------|-------------|
| Q1 | SELECT | orders | src/services/order-service.ts | 34 | WHERE status, ORDER BY created_at |
| Q2 | SELECT+JOIN | orders, order_items | src/services/order-service.ts | 67 | JOIN, WHERE by PK |
| Q3 | SELECT+JOIN | orders, customers | src/routes/admin.ts | 112 | LEFT JOIN, ORDER BY, LIMIT |
| Q4 | SELECT | orders | src/services/analytics.ts | 28 | GROUP BY, date function |
| Q5 | UPDATE | orders | src/services/order-service.ts | 89 | WHERE status + created_at |

```sql
-- Q1: List orders by status
SELECT * FROM orders WHERE status = ? ORDER BY created_at DESC LIMIT 20

-- Q2: Get order details with items
SELECT o.*, oi.product_id, oi.quantity, oi.price
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
WHERE o.id = ?

-- Q3: Admin panel order list with customer info
SELECT o.*, c.name AS customer_name, c.email
FROM orders o
LEFT JOIN customers c ON c.id = o.customer_id
WHERE o.status != 'deleted'
ORDER BY o.created_at DESC LIMIT 20 OFFSET ?

-- Q4: Daily order statistics
SELECT DATE(created_at) AS date, COUNT(*) AS cnt, SUM(total) AS revenue
FROM orders
WHERE created_at >= ?
GROUP BY DATE(created_at)
ORDER BY date ASC

-- Q5: Auto-close unpaid orders after timeout
UPDATE orders SET status = 'closed', updated_at = NOW()
WHERE status = 'pending' AND created_at < ?
```

### Step 2: Data Characteristics Analysis

- `orders`: Main business table, has pagination (LIMIT 20 OFFSET), inferred 50K+ rows
- `order_items`: Average 3 items per order, inferred 150K+ rows
- `customers`: User table, inferred 10K rows
- `status` enum values: `pending`, `paid`, `shipped`, `completed`, `closed`, `deleted`
- `created_at` used in range queries and sorting, continuously growing

### Step 3: Create Test Instance

```bash
curl -s -X POST https://zero.polardbx.com/api/v1/instances \
  -H 'Content-Type: application/json' \
  -d '{"tag": "sql-review", "ttlMinutes": 60}'
```

Example response:
```json
{
  "instanceId": "pxz_abc123",
  "host": "pxc-example.polarxmysql.rds.aliyuncs.com",
  "port": 3306,
  "username": "admin",
  "password": "Ex@mple123"
}
```

Create a MySQL option file, then create database:

```ini
; $TMPDIR/sql_review_my.cnf
[client]
host=pxc-example.polarxmysql.rds.aliyuncs.com
port=3306
user=admin
password=Ex@mple123
```

```bash
mysql --defaults-extra-file=$TMPDIR/sql_review_my.cnf -e "CREATE DATABASE IF NOT EXISTS sql_review_db"
```

### Step 4: Create Tables and Populate Mock Data

Write DDL and stored procedures to a temp file, then execute:

```bash
# Create $TMPDIR/sql_review_setup.sql containing:
#   CREATE TABLE orders (...);
#   CREATE TABLE order_items (...);
#   CREATE TABLE customers (...);
#   DROP PROCEDURE IF EXISTS gen_orders;
#   DELIMITER //
#   CREATE PROCEDURE gen_orders() BEGIN ... END //
#   DELIMITER ;
#   CALL gen_orders();

mysql --defaults-extra-file=$TMPDIR/sql_review_my.cnf \
  sql_review_db < $TMPDIR/sql_review_setup.sql
```

Verify data volume:
```bash
mysql --defaults-extra-file=$TMPDIR/sql_review_my.cnf \
  sql_review_db -e "
    SELECT 'orders' AS tbl, COUNT(*) AS cnt FROM orders
    UNION ALL SELECT 'order_items', COUNT(*) FROM order_items
    UNION ALL SELECT 'customers', COUNT(*) FROM customers"
```

Expected output:
```
+-------------+--------+
| tbl         | cnt    |
+-------------+--------+
| orders      |  50000 |
| order_items | 150000 |
| customers   |  10000 |
+-------------+--------+
```

### Step 5: EXPLAIN Analysis

Analyze each query (Q1 and Q3 shown as examples):

```bash
# Q1: List orders by status
mysql ... sql_review_db -e "
EXPLAIN SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at DESC LIMIT 20"
```

| type | key | rows | Extra |
|------|-----|------|-------|
| ref | idx_status | 8,500 | Using where; **Using filesort** |

```bash
# Q3: Admin panel LEFT JOIN list
mysql ... sql_review_db -e "
EXPLAIN SELECT o.*, c.name AS customer_name, c.email
FROM orders o LEFT JOIN customers c ON c.id = o.customer_id
WHERE o.status != 'deleted' ORDER BY o.created_at DESC LIMIT 20 OFFSET 0"
```

| Table | type | key | rows | Extra |
|-------|------|-----|------|-------|
| o | ALL | None | 50,000 | Using where; **Using filesort** |
| c | eq_ref | PRIMARY | 1 | None |

### Step 6: Analysis and Recommendations

#### Q1: List Orders by Status — Needs Composite Index

**Problem**: Single-column `idx_status` filters to 8,500 rows but still requires filesort on created_at.

**Recommendation**: Create composite index `(status, created_at)`, equality column first, sort column last:
```sql
CREATE INDEX idx_status_created ON orders (status, created_at);
```

**EXPLAIN after optimization**: type=ref, key=idx_status_created, rows=8500, Extra=**Backward index scan** (filesort eliminated)

#### Q3: Admin Panel List — Full Table Scan

**Problem**: `status != 'deleted'` negation + ORDER BY causes full table scan + filesort.

**Recommendation**: Create `(created_at)` single-column index, letting the optimizer scan in reverse index order and filter:
```sql
CREATE INDEX idx_created ON orders (created_at);
```

**EXPLAIN after optimization**: type=index, key=idx_created, rows=1, Extra=Backward index scan; Using where (filesort eliminated, LIMIT 20 enables early termination)

#### Q4: Daily Statistics — Function Causes filesort

**Problem**: `GROUP BY DATE(created_at)` wraps the index column in a function, preventing index-based sorting.

**Recommendation**: Add a redundant `order_date DATE` column, populate on INSERT, and create an index:
```sql
ALTER TABLE orders ADD COLUMN order_date DATE;
CREATE INDEX idx_order_date ON orders (order_date);
```

### Step 7: Cleanup

```bash
curl -s -X DELETE https://zero.polardbx.com/api/v1/instances/pxz_abc123
```

## Sample Report Output (Excerpt)

```markdown
# SQL Review Report

## 1. Overview

### Project Info
- **Project**: ecommerce-backend
- **Scan scope**: src/**/*.ts
- **Language/Framework**: TypeScript + mysql2

### Test Environment
- **Instance**: pxz_abc123
- **Database**: sql_review_db
- **Tables**: 3
- **Mock data**: ~210,000 rows

### Key Findings
- Found **3 queries needing optimization** (5 total scanned)
- Recommend adding **3 indexes**, dropping **1 redundant index**
- Most critical: Admin order list full table scan on 50,000 rows + filesort

## 5. Index Recommendations Summary

### 5.1 Recommended New Indexes

| # | Table | Index Name | Benefits | Priority | DDL |
|---|-------|-----------|----------|----------|-----|
| I1 | orders | idx_status_created | Q1 (status filter + sort), Q5 (timeout close) | High | `CREATE INDEX idx_status_created ON orders (status, created_at);` |
| I2 | orders | idx_created | Q3 (admin panel list) | High | `CREATE INDEX idx_created ON orders (created_at);` |
| I3 | orders | order_date redundant column + index | Q4 (daily stats) | Medium | Application-level change |

### 5.2 Redundant Indexes to Drop

| Table | Redundant Index | Reason | DDL |
|-------|----------------|--------|-----|
| orders | idx_status | Covered by idx_status_created leftmost prefix | `DROP INDEX idx_status ON orders;` |

### 5.3 Optimization Impact Overview

| Query | Before (type / rows / Extra) | After (type / rows / Extra) |
|-------|-----------------------------|-----------------------------|
| Q1 List by status | ref / 8,500 / filesort | ref / 8,500 / Backward index scan |
| Q3 Admin panel list | ALL / 50,000 / filesort | index / 1 / Backward index scan |
| Q5 Timeout close | ref / 8,500 / Using where | range / few / Using index condition |

## 7. Caveats
- Mock data has uniform status distribution; in production, completed orders typically dominate, so Q1 rows may be lower
- Q3 uses idx_created reverse scan + LIMIT 20 early termination; deep pagination (large OFFSET) will still degrade, consider Keyset pagination
- Adding 2 indexes slightly increases INSERT overhead; orders table write frequency is moderate, impact is negligible
```
