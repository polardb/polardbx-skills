---
title: Vector Index — VECTOR Type and HNSW Index for Semantic Search
---

# Vector Index — VECTOR Type and HNSW Index for Semantic Search

PolarDB-X provides native vector storage and similarity search inside MySQL. Use the `VECTOR(N)` data type to define vector columns and the HNSW algorithm-backed `VECTOR INDEX` for high-performance approximate nearest neighbor (ANN) search. Standard SQL `ORDER BY VEC_DISTANCE(...) LIMIT N` is enough — no separate vector database is needed.

Typical use cases: semantic search, RAG (retrieval-augmented generation), recommendation recall, image / multimodal retrieval.

## Prerequisites

- **Edition**: Standard Edition or Enterprise Edition.
- **Engine version**: MySQL 8.0.
- **Storage node version**: `8.4.21-20260423` / `V2.6.0.8.4.21-20260423` or later. Verify with `SELECT VERSION();` (the output must contain `X-Cluster` and a version date >= `20260423`) and `SHOW GLOBAL VARIABLES LIKE 'vidx_disabled';` (the variable must exist). Monitoring metric names may vary by maintenance version; see the Monitoring section.
- **Functionality switch**: `vidx_disabled = OFF` (default is `ON` — feature disabled by default; PolarDB-X Zero ships with `OFF`).
- **Isolation level**: any of RC / RR / SERIALIZABLE works. PolarDB-X Zero defaults to RC.
- **No instance?** Use `polardbx-zero` skill to create a free temporary instance (2C4G Standard Edition with vector index enabled by default).

## Key Capabilities

| Capability | Detail |
|------------|--------|
| Max dimensions | 16,383 |
| Element type | float32 (single-precision) |
| Distance metrics | EUCLIDEAN (L2), COSINE |
| Index algorithm | HNSW (Hierarchical Navigable Small World) |
| Hardware acceleration | AVX512 / AVX2 / ARM NEON SIMD |
| Transaction support | RC / RR / SERIALIZABLE; ACID guaranteed |
| Replication | Binlog replication, primary-secondary consistent; XA supported |
| Distributed transactions | Enterprise Edition only — cross-shard vector consistency |

Vector data is written / updated / deleted within the same transaction as the base table — there is no separate index sync job.

Standard vs Enterprise capability matrix:

| Capability | Standard Edition | Enterprise Edition |
|------------|------------------|--------------------|
| Vector storage + ANN | Supported | Supported |
| Replication / HA | Supported | Supported |
| XA transactions | Supported | Supported |
| Distributed transactions (cross-shard) | — | Supported |

## Enable the Feature

Vector index is gated by `vidx_disabled` (note: **inverted switch** — `ON` means disabled, `OFF` means enabled). Default is `ON`.

**Option 1 (recommended): console**

In the PolarDB-X console, navigate to **Configuration & Management > Parameter Settings > Storage Layer**, set `vidx_disabled` to `OFF`.

**Option 2: SQL (requires SUPER privilege)**

```sql
-- ON = disabled, OFF = enabled
SET GLOBAL vidx_disabled = OFF;
```

Important: existing connections do **NOT** pick up the change automatically. **Reconnect** before using vector index.

PolarDB-X Zero temporary instances ship with `vidx_disabled = OFF` and RC isolation by default — no setup needed.

Verify:

```sql
SHOW GLOBAL VARIABLES LIKE 'vidx_disabled';
SELECT @@transaction_isolation;
```

## VECTOR Data Type

Define a vector column with `VECTOR(N)` where N is the dimension (1 to 16,383):

```sql
CREATE TABLE articles (
    id        INT PRIMARY KEY AUTO_INCREMENT,
    title     VARCHAR(200),
    embedding VECTOR(128)        -- 128-dimensional vector
) ENGINE=InnoDB;
```

**Constraints**:

- Each element is a single-precision float (4 bytes); a `VECTOR(N)` row consumes `N × 4` bytes.
- VECTOR columns can NOT be a primary key, foreign key, unique key, or partition key.
- VECTOR columns only support equality comparison against another vector — not against any other type.
- Inserted data dimension must match the declared `N` exactly.
- `NaN` and `Inf` values are rejected.
- `NULL` is allowed; rows with NULL vectors are NOT included in the index and never returned by ANN search.

If N exceeds 16,383, the engine returns:
`Data size (xxx Bytes, xxx dimensions) exceeds VECTOR max (65532 Bytes, 16383 dimensions) for column: 'xxx'`.

## Create a Vector Index

There are three equivalent ways to create a vector index:

```sql
-- Method 1: inline in CREATE TABLE
CREATE TABLE articles (
    id        INT PRIMARY KEY AUTO_INCREMENT,
    title     VARCHAR(200),
    embedding VECTOR(128),
    VECTOR INDEX vi (embedding) M=6 DISTANCE=EUCLIDEAN
) ENGINE=InnoDB;

-- Method 2: ALTER TABLE
ALTER TABLE articles
  ADD VECTOR INDEX vi (embedding) M=16 DISTANCE=COSINE;

-- Method 3: standalone CREATE VECTOR INDEX (uses default M=6, DISTANCE=EUCLIDEAN)
CREATE VECTOR INDEX vi ON articles (embedding);
```

### Index Parameters

| Parameter | Meaning | Default | Range | Selection guide |
|-----------|---------|---------|-------|-----------------|
| `M` | Max neighbor connections per node in the HNSW graph. Larger M → higher recall, more memory, slower writes. | 6 | [3, 200] | Low-latency: 3-6; balanced: 6-12; high-recall: 16-32; extreme: 32+ |
| `DISTANCE` | Distance metric. EUCLIDEAN measures absolute L2 distance; COSINE measures angular similarity (1 − cosine). | EUCLIDEAN | EUCLIDEAN, COSINE | NLP / text embeddings → COSINE; image features / spatial coordinates → EUCLIDEAN |

Both COSINE and EUCLIDEAN return values where **smaller = more similar**, so always use `ORDER BY distance ASC`.

### Drop and Inspect

```sql
-- Drop a vector index
ALTER TABLE articles DROP INDEX vi;

-- View definition (vector index appears as VECTOR KEY in SHOW CREATE TABLE)
SHOW CREATE TABLE articles;
```

Note: vector indexes do NOT show up in `SHOW INDEX FROM <table>`. Use `SHOW CREATE TABLE` instead.

### DDL Behavior

- Creating, modifying, or dropping a vector index uses the **COPY** algorithm (NOT INPLACE). Large tables may take a long time and hold long locks. Schedule during off-peak hours.
- Vector indexes can NOT be set to `INVISIBLE`.
- Each table currently supports at most **one** vector index.
- Partitioned tables are NOT supported.

## Write Vector Data

Use `VEC_FROMTEXT()` to convert a `'[f1, f2, ...]'` string to internal binary form:

```sql
-- Single-row INSERT
INSERT INTO articles (title, embedding) VALUES
  ('Deep Learning', VEC_FROMTEXT('[0.12, 0.34, 0.56, ...]'));

-- Batch INSERT (recommended for performance)
INSERT INTO articles (title, embedding) VALUES
  ('Article A', VEC_FROMTEXT('[0.1, 0.2, 0.3, 0.4, 0.5]')),
  ('Article B', VEC_FROMTEXT('[0.5, 0.4, 0.3, 0.2, 0.1]'));

-- UPDATE (index is synchronously maintained)
UPDATE articles SET embedding = VEC_FROMTEXT('[0.9, 0.8, ...]') WHERE id = 1;

-- DELETE (index is synchronously cleaned up)
DELETE FROM articles WHERE id = 1;
```

Format requirements: bracket-enclosed, comma-separated floats. Dimension must match the declared `VECTOR(N)`.

## Vector Functions Reference

| Function | Aliases | Description |
|----------|---------|-------------|
| `VEC_FROMTEXT(str)` | `TO_VECTOR`, `STRING_TO_VECTOR` | Convert `'[1.0, 2.0, 3.0]'` string to vector binary |
| `VEC_TOTEXT(vec)` | `FROM_VECTOR`, `VECTOR_TO_STRING` | Convert vector binary to readable string |
| `VEC_DISTANCE(v1, v2)` | — | Compute distance. If `v1` is an indexed column, **automatically uses the index's DISTANCE type**. |
| `VEC_DISTANCE_EUCLIDEAN(v1, v2)` | — | Explicit Euclidean (L2) distance |
| `VEC_DISTANCE_COSINE(v1, v2)` | — | Explicit cosine distance (1 − cosine similarity) |
| `VECTOR_DIM(vec)` | — | Return vector dimension count |

**Recommendation**: prefer `VEC_DISTANCE()` in normal queries — it auto-matches the index's distance type and is the simplest to write. Use the explicit functions only when you need to compute a metric different from the index's metric (e.g., comparing both EUCLIDEAN and COSINE).

## Vector Search

Standard search pattern: `ORDER BY VEC_DISTANCE(...) LIMIT N`.

```sql
-- Find the 10 most similar records
SELECT id, title,
       VEC_DISTANCE(embedding, VEC_FROMTEXT('[0.1, 0.2, 0.3, 0.4, 0.5]')) AS distance
FROM articles
ORDER BY distance
LIMIT 10;
```

### Optimizer Rules

The optimizer picks the vector index ONLY when ALL of the following hold:

1. The query has both `ORDER BY` and `LIMIT`.
2. The first `ORDER BY` expression is `VEC_DISTANCE(col, ...)` (or its explicit variant).
3. The sort direction is `ASC`.
4. `col` has a vector index defined.
5. The DISTANCE type used in the function matches the DISTANCE type of the index.
6. `LIMIT N` is **not too large**: if `N` exceeds `table_rows / 4`, the optimizer falls back to full-scan brute force search (because graph traversal is no longer cheaper).
7. No `GROUP BY` clause. To aggregate, wrap the ANN search inside a subquery first.

### Force or Avoid the Index

```sql
-- Force the vector index (avoids accidental fallback on large tables)
SELECT * FROM articles FORCE INDEX(vi)
ORDER BY VEC_DISTANCE_EUCLIDEAN(embedding, VEC_FROMTEXT('[0.1, 0.2, 0.3, 0.4, 0.5]'))
LIMIT 10;

-- Force full-scan brute search (100% recall, used as a correctness baseline)
SELECT * FROM articles FORCE INDEX(PRIMARY)
ORDER BY VEC_DISTANCE_EUCLIDEAN(embedding, VEC_FROMTEXT('[0.1, 0.2, 0.3, 0.4, 0.5]'))
LIMIT 10;
```

### Verify with EXPLAIN

```sql
EXPLAIN SELECT * FROM articles
ORDER BY VEC_DISTANCE(embedding, VEC_FROMTEXT('[0.1, 0.2, 0.3, 0.4, 0.5]'))
LIMIT 10;
```

When the index is used, the `key` column shows the vector index name (e.g., `vi`).

## System Variables

Only the following user-facing variables matter in normal use:

| Variable | Scope | Default | Range | Purpose |
|----------|-------|---------|-------|---------|
| `vidx_disabled` | GLOBAL | ON | ON / OFF | Master switch. **ON = disabled**, OFF = enabled. **Reconnect** after change. |
| `vidx_default_distance` | SESSION | EUCLIDEAN | EUCLIDEAN / COSINE | Default DISTANCE when not specified at index creation |
| `vidx_hnsw_default_m` | SESSION | 6 | [3, 200] | Default M when not specified at index creation |
| `vidx_hnsw_ef_search` | SESSION | 20 | [1, 10000] | Candidate set size during search. Larger → higher recall, slower. Effective value never below LIMIT. |
| `vidx_hnsw_cache_size` | GLOBAL | 16MB | [1MB, ULLONG_MAX] | Per-index in-memory cache cap |

`ef_search` selection guide:

| `ef_search` | Scenario | Expected recall |
|-------------|----------|-----------------|
| 10 | Low-latency, tolerate misses | ~85% |
| 20 (default) | Balanced | ~92% |
| 50–100 | High recall | ~97% |
| 200+ | Near brute-force | ~99%+ |

Tuning examples:

```sql
-- Boost recall (extra latency)
SET SESSION vidx_hnsw_ef_search = 100;

-- Lower latency (slight recall loss)
SET SESSION vidx_hnsw_ef_search = 10;

-- Cache tuning for a 1M-row, 128-dim, M=6 table (≈ 700MB)
SET GLOBAL vidx_hnsw_cache_size = 734003200;  -- 700MB
```

## Resource Estimation

**Memory**: per-index in-memory cache is bounded by `vidx_hnsw_cache_size`. Estimate the working-set requirement with:

```
cache_size ≈ rows × (dim × 2 + M × 16 + 60)   bytes
```

- `dim × 2`: quantized vector storage (2 bytes per dimension)
- `M × 16`: graph neighbor links overhead
- `60`: per-node fixed overhead

Reference values:

| Rows | Dim | M | Recommended cache |
|------|-----|---|-------------------|
| 1M | 128 | 6 | ≈ 700MB |
| 1M | 768 | 16 | ≈ 1.8GB |
| 10M | 128 | 6 | ≈ 7GB |

**Storage**: the auxiliary table is roughly **1.5×** the size of the base vector column.

If the working set exceeds the cache cap, the engine triggers full cache rebuilds — query performance suffers. Bias the cache size upward; if monitoring shows low hit rate, raise it further.

## Monitoring

```sql
SHOW GLOBAL STATUS LIKE 'Vidx%';
```

**Note on metric naming**: Vector index monitoring metric names evolved across storage-node versions. The set you see depends on your engine release. Two common groupings are listed below; reconcile against the metrics actually returned on your instance.

### Metrics commonly available on all versions

| Metric | Meaning |
|--------|---------|
| `Vidx_query_count` | Total ANN queries executed |
| `Vidx_insert_count` / `Vidx_update_count` / `Vidx_delete_count` | Total index writes |

### Cache-related metrics (variant A — some maintenance versions)

| Metric | Meaning |
|--------|---------|
| `Vidx_share_cache_hits` / `Vidx_share_cache_misses` | Shared cache hits / misses |
| `Vidx_share_cache_usage` | Shared cache memory usage (bytes) |
| `Vidx_trx_cache_usage` | Transaction-scoped cache memory usage (bytes) |

### Cache-related metrics (variant B — some later maintenance versions)

| Metric | Meaning |
|--------|---------|
| `Vidx_load_node_hits` / `Vidx_load_node_misses` | Graph node cache hits / misses |
| `Vidx_load_vec_hits` / `Vidx_load_vec_misses` | Vector data cache hits / misses |
| `Vidx_cache_usage` | Total cache memory usage (bytes) |
| `Vidx_hnsw_share_count` / `Vidx_hnsw_trx_count` | Active HNSW context objects (shared / transaction-scoped) |

**Cache hit rate**: across either variant the formula is `total_hits / (total_hits + total_misses)`. Below 90% → consider raising `vidx_hnsw_cache_size`.

**Capacity planning**: `Vidx_query_count` over time gives the QPS baseline; combine with `Vidx_*_cache_usage` to plan growth.

**Advanced operations note**: Newer engine builds also expose secondary-replica fault-tolerance counters (e.g. `Vidx_save_skip_*`, `Vidx_hl_trx_*`) for replication-health diagnosis. Non-zero values on the secondary indicate auxiliary-table divergence with the primary — engage operations / engineering for investigation. Day-to-day monitoring does not require these.

## Replication and XA

- Vector data writes are part of the same transaction as the base table.
- Binlog carries vector data — primary-secondary stay consistent.
- XA transactions are supported.
- Enterprise Edition supports cross-shard distributed transactions for vector data.

## Limitations

- **Storage engine**: InnoDB only.
- **One vector index per table**: a table currently supports at most one vector index.
- **No partitioned tables**: vector indexes can NOT be created on partitioned tables.
- **COPY DDL**: index create / modify / drop uses COPY algorithm (long-running on large tables).
- **No INVISIBLE**: vector indexes can NOT be marked invisible.
- **Vector data**: dimension must match `VECTOR(N)` exactly; NaN / Inf rejected; NULL stored but not indexed.
- **Space inflation under heavy mutation**: UPDATE / DELETE use tombstone-based maintenance — recall and space efficiency degrade over time. Periodic `OPTIMIZE TABLE` is required to rebuild.

## Troubleshooting

Eight common symptoms and how to fix them.

### 1. Vector index NOT used (EXPLAIN shows no vi)

Checklist:

1. `SHOW GLOBAL VARIABLES LIKE 'vidx_disabled';` returns `OFF`? If you just changed it, **reconnect** the session.
2. Does the SQL contain `ORDER BY VEC_DISTANCE(...) LIMIT N`? Both `ORDER BY` and `LIMIT` are mandatory.
3. Is `LIMIT N <= table_rows / 4`? If not, the optimizer falls back to full-scan; use `FORCE INDEX(vi)` to force.
4. Does the DISTANCE type in `VEC_DISTANCE_*` match the index's DISTANCE? They must be identical.
5. Is the table partitioned? Vector index is unsupported on partitioned tables.
6. Is the sort direction `ASC`? `DESC` does not match the optimizer pattern.

### 2. Recall below expectation

- Raise `vidx_hnsw_ef_search` per session: 20 → 50 → 100 → 200.
- Rebuild the index with a larger `M` (e.g., `M=16`) for permanent recall improvement.
- Use `FORCE INDEX(PRIMARY)` to compute the brute-force ground truth and quantify the gap.
- Confirm DISTANCE matches business semantics (NLP → COSINE, image → EUCLIDEAN).

### 3. Index creation slow or stalled

- COPY DDL builds the HNSW graph row-by-row — duration scales roughly linearly with `rows × dim`.
- Schedule during off-peak hours.
- Trade-off: bulk-load data first then create index (faster build, blocks writes during build) vs. create index first then write (online writes, slower per-insert).
- Avoid extreme M (e.g., 32+) on large tables unless recall demands it.

### 4. Write throughput drops after enabling vector index

- Use multi-row `INSERT` instead of single-row.
- Re-evaluate M — large M slows every write.
- Monitor `Vidx_insert_count` and replication lag.
- Consider batched writes (e.g., 1k rows per batch) and periodic `OPTIMIZE TABLE`.

### 5. Cache hit rate below 90%

- Compute `total_hits / (total_hits + total_misses)` from the cache-related `Vidx_*` metrics available on your engine version (see Monitoring section).
- Raise `vidx_hnsw_cache_size` per the resource estimation formula.
- Watch out: cache exceeding the cap triggers full cache rebuild, hurting query latency.

### 6. Space inflation / recall degrades over time

- UPDATE / DELETE leave tombstoned graph nodes.
- Run `OPTIMIZE TABLE <t>` periodically to rebuild the auxiliary structure.
- Track auxiliary table size growth as a maintenance signal.

### 7. Write errors

- `Data dimension mismatch`: the literal vector's dimension does not equal `VECTOR(N)` — fix the application layer.
- `NaN` / `Inf` rejected: clean inputs upstream.
- NULL accepted but excluded from search.

### 8. Errors or unexpected behavior

- `vidx_disabled = OFF` set but feature still off → forgot to reconnect.
- `CREATE VECTOR INDEX` rejected → table is partitioned, or already has one vector index.
- `INVISIBLE` rejected → vector index does not support invisible.
- Exceeded 16,383 dimensions → reduce N.

### Universal 5-Step Diagnosis Flow

1. **Version**: `SELECT VERSION();` — output must contain `X-Cluster` and version date >= `20260423`; `SHOW GLOBAL VARIABLES LIKE 'vidx_disabled';` must return a row.
2. **Switch**: `SHOW GLOBAL VARIABLES LIKE 'vidx_disabled';` — must be OFF, plus reconnect.
3. **EXPLAIN**: confirm `key` column shows the vector index name.
4. **Status**: `SHOW GLOBAL STATUS LIKE 'Vidx%';` — check `Vidx_query_count` increments and cache hit rate.
5. **Brute-force compare**: `FORCE INDEX(PRIMARY)` to compute ground truth Top-N and compare.

## Performance Tuning

Four-dimensional tuning guide.

### Query-side

- **`vidx_hnsw_ef_search`**: tune per session. Effective value never below LIMIT, so large LIMIT auto-raises ef_search. For small LIMIT (e.g., Top-10) and high recall need, set ef_search = 50–100.
- **Function choice**: prefer `VEC_DISTANCE()` (auto-matches index DISTANCE) over explicit `VEC_DISTANCE_COSINE()` — simpler, less error-prone.
- **Predicate composition**: when ANN search combines with WHERE filters, the effective strategy depends on selectivity:
  - Highly selective filter (small subset) → SQL `WHERE a = x ORDER BY VEC_DISTANCE() LIMIT N`. Filter first, then ANN over the subset.
  - Low selectivity filter (large subset) → ANN first, then post-filter. The optimizer typically picks this when the filter matches > 20% rows.
- **LIMIT sizing**: keep `LIMIT < table_rows / 4`, otherwise the optimizer falls back to full scan.
- **Avoid silent fallback on large tables**: use `FORCE INDEX(vi)` when stable behavior matters.

### Index-side

- **M selection**:
  - Low-latency / small data: M = 3–6.
  - Balanced: M = 6–12 (default 6 fits most cases).
  - High recall: M = 16–32.
  - Extreme recall (>99%): M = 32+ (memory and write cost both grow).
- **DISTANCE selection**:
  - NLP / text embeddings → COSINE (orientation matters more than magnitude).
  - Image features / spatial coordinates → EUCLIDEAN (absolute distance matters).
- **Build timing**: schedule during off-peak. Bulk-load → CREATE INDEX is the fastest path to populate a new table; CREATE INDEX → write is the path for online tables already serving traffic.

### Memory-side

- **Cache size**: estimate with `rows × (dim × 2 + M × 16 + 60)`. Set `vidx_hnsw_cache_size` slightly above this (e.g., 1.2×) to absorb growth.
- **Hit rate threshold**: aim for 90%+. Below, raise the cache.
- **Dimension reduction**: if memory is tight, apply PCA /降维 at the application layer to reduce N before insertion. Halving the dimension roughly halves both index size and search latency.

### Operations-side

- **`OPTIMIZE TABLE`** for high-mutation tables: schedule weekly / monthly to clear tombstones.
- **Capacity tracking**: `Vidx_query_count` over time gives the QPS baseline; combine with cache hit rate to plan growth.
- **TDC cache release**: on engine builds that expose `Vidx_hnsw_share_count` / `Vidx_hnsw_trx_count`, after load tests `share_count > 0` with `trx_count = 0` is normal Table Definition Cache behavior. `FLUSH TABLES` releases them.

## FAQ

**Q: How do I check if my instance supports vector index?**

Run `SELECT VERSION();` — the output must contain `X-Cluster` and a version date >= `20260423` (for example, `8.0.32-X-Cluster-8.4.21-20260423`). Then verify `SHOW GLOBAL VARIABLES LIKE 'vidx_disabled';` returns a row (the variable exists) and the value is `OFF` (feature enabled). Monitoring metric names may vary by maintenance version; see the Monitoring section.

**Q: Why isn't my vector search using the index?**

See Troubleshooting section 1 — most often it's missing `LIMIT`, mismatched DISTANCE, or LIMIT exceeding `table_rows / 4`.

**Q: Recall is too low — what should I do?**

Raise `vidx_hnsw_ef_search` first (cheap, session-only). For permanent improvement, rebuild the index with larger M. See Troubleshooting section 2.

**Q: Why does CREATE VECTOR INDEX take so long?**

Vector index uses COPY DDL — the engine builds the HNSW graph one row at a time. Plan for off-peak windows on large tables.

**Q: How much storage / memory does the index consume?**

Storage: ≈ 1.5× the base vector column. Memory: see the resource estimation formula. A 1M-row 128-dim M=6 index needs roughly 700MB cache.

**Q: Does vector index support replication and HA?**

Yes. Vector data writes go through Binlog, primary-secondary stay consistent, XA is supported. Enterprise Edition additionally supports distributed transactions across shards.

**Q: Can I have multiple vector indexes on the same table?**

No — currently only one vector index per table.

**Q: Can I create a vector index on a partitioned table?**

No — vector index is not supported on partitioned tables.
