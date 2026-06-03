---
name: polardbx-cci
description: |
  Create and use Clustered Columnar Index (CCI) for OLAP/HTAP analytical queries on PolarDB-X 2.0 Enterprise Edition. Covers CCI creation syntax, partition key selection for analytics, query optimization with CCI, CCI vs Clustered GSI differences, CCI + TTL hot/cold data separation, and when NOT to use CCI.
  Use when the user needs analytical queries (aggregation, wide table scans, reports) on PolarDB-X, wants to enable HTAP on existing OLTP tables, or asks about columnar storage.
  Triggers: "CCI", "列存索引", "columnar index", "OLAP", "分析查询", "HTAP", "实时分析", "宽表聚合", "CLUSTERED COLUMNAR", "行列混存", "analytical query", "columnar storage", "report query", "data analysis acceleration"
metadata:
  version: 0.1.0
---

# PolarDB-X CCI — Clustered Columnar Index for OLAP/HTAP

Create and use Clustered Columnar Index (CCI) to accelerate OLAP analytical queries on PolarDB-X 2.0 Enterprise Edition (AUTO mode). CCI provides row-column hybrid storage (HTAP) — OLTP queries use row-store partitioned tables while OLAP queries leverage columnar storage.

**Scope**: PolarDB-X 2.0 Enterprise Edition + AUTO mode database only.

## What is CCI

CCI is a **columnar clustered index** stored on object storage. It stores all columns from the primary table in columnar format by default, optimized for scan-heavy analytical workloads (aggregation, GROUP BY, wide table scans).

**Key characteristics**:
- Stores ALL columns (clustered) — no need to specify covering columns.
- Columnar format — optimized for large range scans and aggregations.
- Based on object storage — lower cost than row-store for large data volumes.
- Eventually consistent — writes have some latency (not real-time).
- Online creation — does not block DML on the primary table.

## Core Workflow

1. **Confirm OLAP/HTAP requirement**: User needs analytical queries (SUM/COUNT/AVG, multi-column aggregation, complex reports) on OLTP tables.
2. **Select CCI partition key**: Choose the column most frequently used in analytical `GROUP BY` or `WHERE` filter conditions.
3. **Create CCI** with appropriate partition count.
4. **Verify query uses CCI**: Use `EXPLAIN` to confirm the optimizer selects the CCI path.

## When to Use CCI

| Scenario | Recommended | Reason |
|----------|-------------|--------|
| Wide table multi-column aggregation (SUM/COUNT/AVG) | Yes | Columnar scan far faster than row scan |
| Complex reports and dashboards | Yes | Analytical workload benefits from columnar |
| HTAP: OLTP + OLAP on same data | Yes | CCI handles analytics without impacting OLTP |
| Hot/cold data separation with TTL | Yes | Cold data archived to CCI (object storage) |
| Point queries / small range scans | **No** | Use Clustered GSI instead |
| Tables with < 100K rows | **No** | Overhead not justified for small data |
| Extreme write-heavy with real-time read requirements | **No** | CCI has write latency (eventual consistency) |

## CCI Creation Syntax

### During table creation

```sql
CREATE TABLE t_order (
  order_id BIGINT PRIMARY KEY,
  buyer_id BIGINT,
  seller_id BIGINT,
  amount DECIMAL(10,2),
  create_time DATETIME,
  CLUSTERED COLUMNAR INDEX cci_seller(seller_id)
    PARTITION BY KEY(seller_id) PARTITIONS 16
) PARTITION BY KEY(order_id) PARTITIONS 16;
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

**Partition key selection for CCI**: Choose the column most frequently used in analytical `GROUP BY` clauses. This enables partition pruning for analytical queries. If unsure, use the same partition key as the primary table.

## Query with CCI

The optimizer automatically selects CCI for analytical queries based on cost model:

```sql
-- Automatic (optimizer decides)
SELECT seller_id, SUM(amount) FROM t_order
GROUP BY seller_id ORDER BY SUM(amount) DESC LIMIT 10;

-- Force CCI via HINT
SELECT /*+TDDL:FORCE_INDEX(t_order, cci_seller)*/ seller_id, SUM(amount)
FROM t_order GROUP BY seller_id;

-- Force CCI via FORCE INDEX
SELECT seller_id, SUM(amount) FROM t_order FORCE INDEX(cci_seller)
GROUP BY seller_id;
```

## View CCI Information

```sql
SHOW COLUMNAR INDEX;
```

## CCI vs Clustered GSI Comparison

| Feature | Clustered GSI | CCI |
|---------|--------------|-----|
| Storage format | Row-store | Columnar |
| Best for | Point queries, small range scans | Large scans, aggregations |
| Storage backend | DN local storage | Object storage (lower cost) |
| Data freshness | Real-time (strong consistency) | Eventually consistent (slight delay) |
| Write impact | Distributed transaction overhead | Lower write amplification |
| Use case | OLTP supplementary index | OLAP/HTAP analytics |

Both store all primary table columns by default (clustered). The key difference is storage format and applicable query types.

## CCI + TTL Hot/Cold Separation

CCI can work with TTL tables for cost-effective hot/cold architecture:
- **Hot data**: Row-store partitioned table (fast OLTP access)
- **Cold data**: Archived to CCI on object storage (low-cost analytical access)

This is configured through the `polardbx-ttl20` skill — use that skill for TTL + archive table setup.

## Data Freshness

CCI data has a **slight delay** compared to the primary table (eventual consistency). This is acceptable for:
- Reporting and dashboards (minutes-level freshness)
- Batch analytics (daily/hourly aggregations)
- Historical data analysis

NOT suitable for scenarios requiring real-time consistency (use Clustered GSI instead).

## Limitations

- Only supported by PolarDB-X **Enterprise Edition (Distributed Edition)**.
- CCI data is eventually consistent (not real-time).
- Creating CCI is an online operation (does not block DML).
- CCI partition key must be specified with `PARTITION BY KEY(...) PARTITIONS N`.
- One table can have multiple CCIs with different partition keys for different analytical dimensions.

## Best Practices

1. **Choose CCI partition key based on analytical patterns**: Pick the most frequent `GROUP BY` column for the CCI partition key.
2. **Don't use CCI for point queries**: Use Clustered GSI for OLTP-style lookups.
3. **Accept eventual consistency**: CCI is designed for analytics where slight delay is acceptable.
4. **Combine with TTL for cost optimization**: Archive cold data to CCI on object storage.
5. **Create multiple CCIs for different analytical dimensions** if needed (e.g., one by seller_id, one by region).
6. **Use EXPLAIN to verify CCI usage**: Confirm the optimizer is picking the CCI path for your queries.

## Reference

| Reference | Description |
|-----------|-------------|
| [references/cci.md](references/cci.md) | CCI creation syntax, query usage, relationship with GSI, TTL combination, limitations |
