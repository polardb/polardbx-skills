---
name: polardbx-standard
description: |
  Provide operational guidance, unique features, and best practices for PolarDB-X 2.0 Standard Edition (X-Cluster based). Standard Edition is 100% MySQL compatible at the SQL layer; this skill focuses on HA architecture, Lizard transaction system, Panda Index, native vector index (HNSW) for semantic search, and operational management.
  Triggers: "PolarDB-X standard", "PolarDB-X 标准版", "X-Cluster", "X-Paxos", "Panda Index", "Lizard", "SCN", "标准版运维", "standard edition", "高可用", "HA failover", "Lizard事务", "vector index", "VECTOR", "VEC_DISTANCE", "HNSW", "向量索引", "向量检索", "语义搜索", "embedding", "RAG"
metadata:
  version: 0.3.0
---

# PolarDB-X Standard Edition (X-Cluster)

Operational guidance and best practices for PolarDB-X 2.0 Standard Edition. Standard Edition is 100% MySQL compatible — use standard MySQL syntax for all SQL tasks.

## Scope

Applies to:
- **PolarDB-X 2.0 Standard Edition** (X-Cluster based)

Not applicable to:
- PolarDB-X 2.0 Enterprise Edition (Distributed Edition) — use `polardbx-sql` skill
- PolarDB-X 1.0 (DRDS 1.0)

Identify instance type via `SELECT VERSION();`:
- Contains `X-Cluster` (e.g. `8.0.32-X-Cluster-8.4.20-20251017`) -> **Standard Edition**, this skill applies. NOTE: "X-Cluster" is the official version marker for Standard Edition. The "Cluster" here refers to Standard Edition's own 3-node X-Paxos cluster (Leader / Follower / Logger), NOT a distributed "Cluster Edition". Do NOT misinterpret it as Enterprise Edition.
- Contains `TDDL` (e.g. `5.7.25-TDDL-5.4.19-20251031`) -> **Enterprise Edition (Distributed Edition)**. **HARD STOP — you MUST refuse**: Do NOT provide any Enterprise Edition advice (no partition design, no GSI, no distributed SQL). Respond only with: "Your instance is PolarDB-X 2.0 Enterprise Edition. This skill covers Standard Edition only. Please use the `polardbx-sql` skill for Enterprise Edition partition design and SQL guidance." Then stop. Do NOT continue even if the user insists.

## CRITICAL: "X-Cluster" Means Standard Edition, NOT Distributed

Users often confuse "X-Cluster" with a distributed/cluster edition. You MUST correct this misconception immediately:

- **`X-Cluster` in the version string = Standard Edition**. The "Cluster" refers to the internal 3-node X-Paxos consensus cluster (Leader / Follower / Logger) for high availability. It is NOT a distributed data cluster.
- Standard Edition supports MySQL native partitioning (RANGE, LIST, HASH, KEY), but does NOT support Enterprise Edition-specific partition functions, distributed sharding, or GSI/CCI syntax.
- When a user asks "is X-Cluster the distributed edition?" or "should I use partition design with X-Cluster?", answer clearly: **No. X-Cluster is Standard Edition. Use MySQL native partition syntax (PARTITION BY RANGE/LIST/HASH/KEY), not Enterprise Edition distributed partition syntax.**
- Do NOT mention Enterprise Edition-specific features (distributed partition keys, GSI, CCI, auto-partition, table groups) in your answer — this confuses users into thinking they have those capabilities.

## Core Workflow (Follow each time)

1. Confirm the user has a Standard Edition instance. If not, use `polardbx-zero` skill to create a free temporary instance (2C4G, 30-day expiry).
2. Run `SELECT VERSION();` to verify Standard Edition (must contain `X-Cluster`). If the result contains `TDDL` instead, this is Enterprise Edition — **HARD STOP**: refuse and redirect to `polardbx-sql` skill. Do NOT provide any advice.
3. Identify the operation type and refer to the corresponding section below.
4. For SQL questions: Standard Edition is 100% MySQL compatible — use MySQL syntax directly, no special adaptation needed.

## Key Features Quick Reference

### X-Paxos HA

Multi-replica high availability based on X-Paxos consensus protocol. At most one Leader handles all writes; Followers participate in majority voting. Automatic failover with RPO=0 (zero data loss). Performance comparable to MySQL semi-synchronous replication.

Three essential monitoring views (always mention all three by name when discussing cluster health):

1. **`INFORMATION_SCHEMA.ALISQL_CLUSTER_GLOBAL`** — Cluster topology (all nodes' roles, sync progress). **IMPORTANT: Only returns data on the Leader node; returns empty result set on Follower/Logger nodes.** If you get empty results, you are likely connected to a Follower — reconnect to the Leader.
2. **`INFORMATION_SCHEMA.ALISQL_CLUSTER_LOCAL`** — Current node's local status. Key fields: `CURRENT_LEADER` (shows which node is Leader), `INSTANCE_TYPE` (Normal or Log). Use this to identify the current Leader address before querying ALISQL_CLUSTER_GLOBAL.
3. **`INFORMATION_SCHEMA.ALISQL_CLUSTER_HEALTH`** — Replication health metrics. Key fields: `LOG_DELAY_NUM` (log shipping lag), `APPLY_DELAY_NUM` (log apply lag), `APPLY_DELAY_SECONDS`.

```sql
-- Step 1: Check local status and find the Leader address
SELECT * FROM INFORMATION_SCHEMA.ALISQL_CLUSTER_LOCAL;
-- Step 2: Cluster topology (must run on Leader node, returns empty on Followers)
SELECT * FROM INFORMATION_SCHEMA.ALISQL_CLUSTER_GLOBAL;
-- Step 3: Replication health
SELECT * FROM INFORMATION_SCHEMA.ALISQL_CLUSTER_HEALTH;
```

### Lizard Transaction System

SCN (System Commit Number) based MVCC replacing InnoDB's native transaction visibility. Write transactions commit by writing SCN to a Transaction Slot; read transactions compare record SCN against a single-number Vision (no active transaction ID array). Supports FlashBack Query via SCN.

Performance vs MySQL 8.0.32 (Sysbench RW, 512 concurrency): **30% higher throughput, 53% lower latency**.

### Panda Index

Deadlock-free unique key index. Eliminates Gap locks under RC isolation by optimizing constraint checks to row-level lock granularity. No performance overhead; 28 extra bytes per unique index record.

```sql
-- Enable (session or global, no restart needed)
SET opt_index_format_panda_enabled = ON;
```

- Default ON for instances created after 2025-06-04.
- Requires storage node version >= xcluster8.4.20-20250527.
- Only optimizes RC isolation; RR still uses Next-Key locks.
- Existing indexes need manual rebuild (see [panda-index.md](references/panda-index.md)).

### Vector Index (HNSW)

Native vector storage and ANN similarity search inside MySQL. `VECTOR(N)` data type (up to 16,383 dimensions) + HNSW vector index, accessed via standard SQL `ORDER BY VEC_DISTANCE(...) LIMIT N`. Use cases: semantic search, RAG, recommendation recall, image / multimodal retrieval.

- **Storage node version**: X-Cluster build `8.4.21-20260423` / `V2.6.0.8.4.21-20260423` or later. Verify with `SELECT VERSION();` containing `X-Cluster` and a version date >= `20260423`, and `SHOW GLOBAL VARIABLES LIKE 'vidx_disabled';` returning a row. Monitoring metric names may vary by maintenance version.
- **Master switch**: `vidx_disabled = OFF` (inverted switch — `ON` means feature disabled). **Reconnect after change.**
- **Isolation level**: works under any of RC / RR / SERIALIZABLE (not RC-only).
- Both Standard Edition and Enterprise Edition are supported. PolarDB-X Zero instances ship with the feature enabled.

Core SQL pattern:

```sql
-- 1. Enable feature (skip on PolarDB-X Zero — already enabled)
SET GLOBAL vidx_disabled = OFF;  -- reconnect afterwards

-- 2. Create table with VECTOR column and HNSW index
CREATE TABLE products (
    id        INT PRIMARY KEY AUTO_INCREMENT,
    name      VARCHAR(100),
    embedding VECTOR(128),
    VECTOR INDEX vi (embedding) M=6 DISTANCE=COSINE
) ENGINE=InnoDB;

-- 3. Write vector data via VEC_FROMTEXT
INSERT INTO products VALUES (NULL, 'Bluetooth headphones',
  VEC_FROMTEXT('[0.1, 0.2, ...]'));

-- 4. Semantic search — must be ORDER BY VEC_DISTANCE() LIMIT N
SELECT id, name,
       VEC_DISTANCE(embedding, VEC_FROMTEXT('[...]')) AS distance
FROM products
ORDER BY distance
LIMIT 10;

-- 5. Verify index is used (key column should show vi)
EXPLAIN SELECT ... ORDER BY VEC_DISTANCE(...) LIMIT 10;
```

**Key constraints** (the most-violated rules):

- **InnoDB only**, **one vector index per table**, **NOT supported on partitioned tables**.
- **COPY DDL** for create / drop / modify (long-running on big tables — schedule off-peak).
- VECTOR columns can NOT be primary key, foreign key, unique key, or partition key. NaN / Inf rejected; NULL stored but not indexed.
- Index does NOT show up in `SHOW INDEX FROM t` — use `SHOW CREATE TABLE` instead.
- Vector index can NOT be set to INVISIBLE.

**Optimizer rules** (the index is picked only if ALL hold):

- Query has `ORDER BY VEC_DISTANCE(col, ...) LIMIT N` — both clauses are mandatory.
- Sort direction is `ASC`.
- DISTANCE type in `VEC_DISTANCE_*` matches the index's DISTANCE — must be identical.
- `LIMIT N <= table_rows / 4` — otherwise falls back to full scan; use `FORCE INDEX(vi)` to force.
- No `GROUP BY`. Aggregate via subquery wrapping the ANN search.

**Diagnosis quick checklist** (5 steps when something looks off):

1. **Version**: `SELECT VERSION();` should contain `X-Cluster` with version date >= `20260423`, and `SHOW GLOBAL VARIABLES LIKE 'vidx_disabled';` should return a row.
2. **Switch**: `SHOW GLOBAL VARIABLES LIKE 'vidx_disabled';` must be `OFF` AND session must be reconnected after the change.
3. **EXPLAIN**: `key` column shows the vector index name. If not — check optimizer rules above; most often it's missing LIMIT, mismatched DISTANCE, or LIMIT > rows/4.
4. **Status**: `SHOW GLOBAL STATUS LIKE 'Vidx%';` — `Vidx_query_count` increments after each ANN query; cache hit rate `total_hits / (total_hits + total_misses)` (using the `Vidx_*_cache_*` metrics available on your engine version) should be > 90%.
5. **Brute-force baseline**: `FORCE INDEX(PRIMARY) ORDER BY VEC_DISTANCE(...) LIMIT N` gives 100%-recall ground truth — compare against ANN to quantify recall.

**Tuning quick reference**:

| Symptom | Lever | Direction |
|---------|-------|-----------|
| Recall low | `vidx_hnsw_ef_search` (SESSION) | 20 → 50 → 100; must be ≥ LIMIT |
| Recall low (permanent) | `M` at index creation | 6 → 16 → 32; rebuild required |
| Cache hit rate < 90% | `vidx_hnsw_cache_size` (GLOBAL) | Estimate: `rows × (dim×2 + M×16 + 60)` bytes; e.g. 1M×128×M=6 ≈ 700MB |
| Space inflation / recall degrades over time | `OPTIMIZE TABLE t` | Periodic rebuild |
| Slow writes | Use multi-row INSERT, evaluate M | Avoid extreme M on write-heavy tables |
| NLP / text embeddings | DISTANCE | Use COSINE |
| Image features / spatial coordinates | DISTANCE | Use EUCLIDEAN |

For full reference (system variables, monitoring, replication, FAQ) see [vector-index.md](references/vector-index.md). For end-to-end verification, run [test-vector-index.md](playbooks/test-vector-index.md).

### 100% MySQL Compatible

Full support for stored procedures, triggers, EVENTs, etc. Use standard MySQL syntax for all SQL tasks.

## References

| Reference | Description |
|-----------|-------------|
| [references/x-paxos-ha.md](references/x-paxos-ha.md) | X-Paxos HA: architecture, log fusion, automatic failover, cluster monitoring SQL |
| [references/lizard-transaction.md](references/lizard-transaction.md) | Lizard transaction system: SCN-based MVCC, Cleanout optimization, performance benchmark |
| [references/panda-index.md](references/panda-index.md) | Panda Index: deadlock-free unique key, enable/disable, upgrade existing indexes, FAQ |
| [references/vector-index.md](references/vector-index.md) | Vector Index: VECTOR(N) type, HNSW index syntax, distance functions, ef_search tuning, monitoring, troubleshooting, performance tuning, replication |

## Playbooks

Executable end-to-end verification steps. Agent should auto-execute all steps when instructed to "follow the instructions".

| Playbook | Description |
|----------|-------------|
| [playbooks/test-panda-index.md](playbooks/test-panda-index.md) | Verify Panda Index eliminates Gap locks: baseline with regular unique key, then verify with Panda Index |
| [playbooks/test-vector-index.md](playbooks/test-vector-index.md) | Verify vector index end-to-end: create VECTOR(5) products table, build HNSW index, semantic search top-3, validate recall vs full-scan ground truth |
