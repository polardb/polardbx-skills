---
title: Verify Vector Index End-to-End
---

# Verify Vector Index End-to-End

End-to-end verification: build a `products` table with a `VECTOR(5)` column, create an HNSW vector index, run semantic search, validate index usage, and confirm recall by comparing against full-scan ground truth.

See `skills/polardbx-standard/references/vector-index.md` for feature details.

## Step 1: Obtain Instance

If the user already provided connection info, use it directly. Otherwise, create a temporary instance via PolarDB-X Zero:

```bash
curl -s -X POST https://zero.polardbx.com/api/v1/instances \
  -H 'Content-Type: application/json' \
  -d '{"tag": "vector-index-test", "ttlMinutes": 60}'
```

Extract `host`, `port`, `username`, `password` from the response.

## Step 2: Connect and Verify Version

Use environment variables to pass the password (avoids shell-escape issues with special characters like `!`, `$`, `*`):

```bash
export MYSQL_PWD='<password>'
mysql -h <host> -P <port> -u <username> -e "SELECT VERSION();"
```

Confirm output contains `X-Cluster` and version date is >= `20260423`; also verify the instance exposes the vector switch (`SHOW GLOBAL VARIABLES LIKE 'vidx_disabled';`). PolarDB-X Zero standard instances should meet this requirement.

## Step 3: Verify Vector Index Feature is Enabled

Vector index requires `vidx_disabled = OFF` (the feature is gated behind an inverted switch — `ON` means disabled).

```sql
SHOW GLOBAL VARIABLES LIKE 'vidx_disabled';
SELECT @@transaction_isolation;
```

**Expected on PolarDB-X Zero**: `vidx_disabled = OFF` and `@@transaction_isolation = READ-COMMITTED`. Both are the factory defaults; no further setup is needed.

**Note**: vector index works under any of RC / RR / SERIALIZABLE — isolation level does NOT need to be RC. The check above only confirms the default. If you are on a self-managed instance where `vidx_disabled = ON`, run `SET GLOBAL vidx_disabled = OFF;` (requires SUPER privilege) and reconnect.

## Step 4: Create the Products Table and Vector Index

```sql
CREATE DATABASE IF NOT EXISTS vector_test;
USE vector_test;

DROP TABLE IF EXISTS products;
CREATE TABLE products (
    id        INT PRIMARY KEY AUTO_INCREMENT,
    name      VARCHAR(100) NOT NULL,
    category  VARCHAR(50),
    embedding VECTOR(5)
) ENGINE=InnoDB;

ALTER TABLE products
  ADD VECTOR INDEX vi_embedding (embedding) M=6 DISTANCE=COSINE;
```

Key parameters:
- `VECTOR(5)`: 5-dimensional vector (production scenarios may go up to 16,383).
- `M=6`: HNSW max neighbor connections per node — balanced default.
- `DISTANCE=COSINE`: cosine distance, suitable for text / NLP embeddings (also supports `EUCLIDEAN`).

## Step 5: Insert 16 Product Rows

Insert at least 13 rows so that `LIMIT 3` does not exceed `table_rows / 4` (the optimizer would otherwise fall back to full scan). With 16 rows, `LIMIT 3 < 16 / 4 = 4`, so the optimizer picks the vector index.

```sql
INSERT INTO products (name, category, embedding) VALUES
  ('Wireless Bluetooth Headphones',  'Digital', VEC_FROMTEXT('[0.8, 0.1, 0.3, 0.5, 0.2]')),
  ('Noise-Cancelling Headphones',    'Digital', VEC_FROMTEXT('[0.7, 0.2, 0.4, 0.6, 0.1]')),
  ('Sports Bluetooth Earbuds',       'Sports',  VEC_FROMTEXT('[0.6, 0.5, 0.3, 0.4, 0.3]')),
  ('Smart Watch',                    'Digital', VEC_FROMTEXT('[0.3, 0.7, 0.2, 0.8, 0.1]')),
  ('Running Fitness Band',           'Sports',  VEC_FROMTEXT('[0.2, 0.8, 0.1, 0.7, 0.4]')),
  ('Yoga Mat',                       'Sports',  VEC_FROMTEXT('[0.1, 0.9, 0.5, 0.2, 0.6]')),
  ('Mechanical Keyboard',            'Digital', VEC_FROMTEXT('[0.9, 0.1, 0.1, 0.3, 0.7]')),
  ('Ergonomic Chair',                'Home',    VEC_FROMTEXT('[0.4, 0.3, 0.8, 0.2, 0.5]')),
  ('Game Controller',                'Digital', VEC_FROMTEXT('[0.5, 0.2, 0.6, 0.4, 0.8]')),
  ('Electric Toothbrush',            'Home',    VEC_FROMTEXT('[0.2, 0.6, 0.4, 0.3, 0.7]')),
  ('Air Purifier',                   'Home',    VEC_FROMTEXT('[0.1, 0.4, 0.7, 0.5, 0.3]')),
  ('Power Bank',                     'Digital', VEC_FROMTEXT('[0.7, 0.3, 0.2, 0.6, 0.4]')),
  ('Sports Water Bottle',            'Sports',  VEC_FROMTEXT('[0.3, 0.7, 0.4, 0.1, 0.5]')),
  ('Bluetooth Speaker',              'Digital', VEC_FROMTEXT('[0.8, 0.2, 0.5, 0.3, 0.4]')),
  ('Running Shoes',                  'Sports',  VEC_FROMTEXT('[0.4, 0.6, 0.2, 0.5, 0.3]')),
  ('Noise-Cancelling Earplugs',      'Digital', VEC_FROMTEXT('[0.75, 0.15, 0.35, 0.55, 0.2]'));
```

## Step 6: Semantic Search via Vector Index

Search for the top-3 products most similar to a query vector that semantically represents "Bluetooth headphones":

```sql
SELECT id, name, category,
       VEC_DISTANCE(embedding, VEC_FROMTEXT('[0.75, 0.15, 0.35, 0.55, 0.15]')) AS distance
FROM products
ORDER BY distance
LIMIT 3;
```

**Expected**: Top-3 results are headphone-like products (e.g., "Noise-Cancelling Earplugs", "Noise-Cancelling Headphones", "Wireless Bluetooth Headphones") — distance ascending. Compared to keyword search like `LIKE '%bluetooth%'`, vector search finds semantically similar items even without literal keyword match.

## Step 7: Verify Index is Used (EXPLAIN)

```sql
EXPLAIN SELECT id, name,
       VEC_DISTANCE(embedding, VEC_FROMTEXT('[0.75, 0.15, 0.35, 0.55, 0.15]')) AS distance
FROM products
ORDER BY distance
LIMIT 3;
```

**Expected**: `key` column shows `vi_embedding` — confirming the optimizer picked the vector index instead of full scan.

## Step 8: Compare Against Full-Scan Ground Truth

Vector index does ANN (approximate) search. Validate recall by forcing full scan (precise computation) and comparing the Top-N results:

```sql
SELECT id, name,
       VEC_DISTANCE_COSINE(embedding, VEC_FROMTEXT('[0.75, 0.15, 0.35, 0.55, 0.15]')) AS distance
FROM products FORCE INDEX(PRIMARY)
ORDER BY distance
LIMIT 3;
```

**Expected**: Top-3 results match Step 6 (same set, same order). On this small dataset recall is 100%; on larger production datasets some divergence in lower-ranked results is normal — raise `vidx_hnsw_ef_search` if recall is insufficient.

## Step 9: Tune Search Precision (`ef_search`)

`vidx_hnsw_ef_search` controls the candidate set size during HNSW search. Default 20; raise to 50 for higher recall (slightly more latency):

```sql
SET SESSION vidx_hnsw_ef_search = 50;

SELECT id, name,
       VEC_DISTANCE(embedding, VEC_FROMTEXT('[0.75, 0.15, 0.35, 0.55, 0.15]')) AS distance
FROM products
ORDER BY distance
LIMIT 3;
```

`ef_search` selection:

| `ef_search` | Scenario | Expected recall |
|-------------|----------|-----------------|
| 10 | Low-latency, allow misses | ~85% |
| 20 (default) | Balanced | ~92% |
| 50–100 | High recall | ~97% |
| 200+ | Near brute-force | ~99%+ |

On 16 rows the difference is invisible. The benefit shows on millions of rows.

## Step 10: Auxiliary Functions and Monitoring

Use `VEC_TOTEXT()` to read back stored vectors and `VECTOR_DIM()` to inspect dimensions:

```sql
SELECT id, name,
       VEC_TOTEXT(embedding) AS vec_text,
       VECTOR_DIM(embedding) AS dims
FROM products
LIMIT 3;
```

Inspect runtime monitoring:

```sql
SHOW GLOBAL STATUS LIKE 'Vidx%';
```

Key metrics to interpret (the exact metric names vary slightly across engine versions):

- `Vidx_query_count` — total ANN queries; should increment after each search.
- `Vidx_insert_count` / `Vidx_update_count` / `Vidx_delete_count` — total index writes.
- Cache-related metrics: metric names vary slightly across maintenance versions. Some builds expose `Vidx_share_cache_hits` / `Vidx_share_cache_misses` / `Vidx_share_cache_usage` / `Vidx_trx_cache_usage`; other builds expose `Vidx_load_node_*` / `Vidx_load_vec_*` / `Vidx_cache_usage` / `Vidx_hnsw_share_count` / `Vidx_hnsw_trx_count`.
- Cache hit rate = `total_hits / (total_hits + total_misses)`. Below 90% → consider raising `vidx_hnsw_cache_size`.
- On this 16-row dataset the numbers are not statistically meaningful; the goal of this step is just to confirm the metrics are reachable.

## Step 11: Cleanup

```sql
DROP DATABASE IF EXISTS vector_test;
```

If the instance was temporarily created via PolarDB-X Zero, it will automatically expire and be cleaned up at the end of the configured TTL — no manual release is required. (The DELETE API may not be reachable for all instances; rely on TTL-based cleanup.)

## Pass / Fail Criteria

- **Pass**: Step 6 returns headphone-like products in distance-ascending order, AND Step 7 EXPLAIN shows `key = vi_embedding`, AND Step 8 full-scan Top-3 matches Step 6.
- **Fail**: index not used in Step 7, OR Step 6 / Step 8 results diverge significantly, OR version prerequisite not met.

## Step 12: Output Test Report

After all steps complete, output the following Markdown report directly to the user (do not write to file). Fill `<>` placeholders with actual execution results:

~~~markdown
# PolarDB-X 向量索引体验报告

- 测试时间：<执行时的时间戳>
- 实例版本：<SELECT VERSION() 的实际输出>
- 隔离级别：<SELECT @@transaction_isolation 的实际输出>

## 向量索引简介

PolarDB-X 在 MySQL 生态中原生支持向量存储与检索。通过 `VECTOR(N)` 数据类型定义向量字段，使用基于 HNSW 算法的向量索引实现高性能近似最近邻搜索，配合 AVX512/AVX2 SIMD 硬件加速，让标准 SQL 直接具备语义搜索能力。

核心优势：无需额外部署向量数据库，业务数据与向量数据在同一事务中操作，保证一致性。

## 表结构

```sql
<Step 4 中 CREATE TABLE products 的完整语句>

<Step 4 中 ALTER TABLE ... ADD VECTOR INDEX 的完整语句>
```

- 向量维度：5 维（最高支持 16,383 维）
- 距离度量：COSINE（余弦相似度，越小越相似）
- HNSW 参数：M=6（默认值，平衡精度与性能）

## 语义搜索结果

查询向量：`[0.75, 0.15, 0.35, 0.55, 0.15]`（模拟"蓝牙耳机"的嵌入向量）

### 向量索引检索（Step 6）

<Step 6 SELECT 查询的完整结果表格>

### EXPLAIN 执行计划（Step 7）

<Step 7 EXPLAIN 的完整输出>

索引生效：<是 / 否，key 列是否为 vi_embedding>

### 全表扫描对比（Step 8）

<Step 8 FORCE INDEX(PRIMARY) 查询的完整结果表格>

召回一致性：<一致 / 不一致>

## 参数调优（Step 9）

ef_search = 50 时的查询结果：

<Step 9 查询的完整结果表格>

## 监控状态（Step 10）

<SHOW GLOBAL STATUS LIKE 'Vidx%' 的完整输出>

缓存命中率：<计算值或"数据量过小不具参考意义">

## 最终结论

<通过 / 失败，以及一句话总结，例如：体验通过。PolarDB-X 向量索引在标准 SQL 中实现了语义搜索能力，查询走了 vi_embedding 索引，召回结果与全表扫描完全一致。>
~~~
