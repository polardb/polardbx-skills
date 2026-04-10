# Step 6: Report & Cleanup

## 6.1 Gather Data from Progress File

Read `$TMPDIR/sql_review_progress.md` — this is the **single source of truth** for all scan and EXPLAIN results accumulated during previous steps. It contains:

- Per-table **SQL Inventory** sections (from Step 2)
- Per-table **EXPLAIN Results** sections (from Step 5) with baseline/post-index comparisons and recommended DDL

Also read `$TMPDIR/sql_review_name_mapping.md` and map all anonymous names back to real names. All table names, column names, and index names in the report must use real business names. Recommended index DDL should be directly executable in production.

### 6.2 Assemble Report

**Must write report to file** `sql-review-report.md` (project root), and output a summary in the conversation.

Assemble the progress file sections into a unified report with 7 sections:

1. **Overview** — Project info, test environment, key findings (queries needing optimization, recommended indexes, most critical issue). Derive counts from the per-table sections in the progress file. **[Aliyun mode]** Include data source info: instance ID, region, time window, API path used.
2. **Table Schema Overview** — Column count, existing indexes, mock data volume per table.
3. **SQL Inventory** — Consolidate all per-table "SQL Inventory" sections from the progress file into one grouped listing. **[Aliyun mode]** Include Freq (execution count) and AvgRt (avg latency) columns.
4. **Query Analysis** — Consolidate all per-table "EXPLAIN Results" sections. Grouped by source file, each query with SQL, EXPLAIN results, issues and recommendations. Queries needing optimization show the index exploration process (EXPLAIN comparison of each candidate). Normal -> pass / Needs optimization -> warn. **For multi-path queries (MyBatis)**: group all variants together in a comparison table. **[Aliyun mode]** Each query additionally shows: execution frequency, average latency (ms), average scanned rows from production metrics.
5. **Index Recommendations Summary** — Collect all recommended DDL from per-table EXPLAIN sections. Deduplicate, merge overlapping indexes, and present final recommended indexes (with DDL and selection rationale), redundant indexes, optimization impact overview. For multi-path scenarios, state which index covers which variants. **[Aliyun mode]** Sort recommendations by execution frequency (high-frequency slow queries first), and include estimated impact = Freq × latency improvement.
6. **Security Risk Notes** — SQL injection, dynamic concatenation, `${param}` usage (MyBatis string interpolation), etc. (from per-table scan notes in the progress file).
7. **Caveats** — Mock vs production data distribution differences, composite index column ordering rationale, write overhead assessment. For multi-path queries, note that the relative frequency of each path in production determines the practical impact of each index. **[Aliyun mode]** Note the time window limitation: SQL insights and slow queries only cover the specified time range (default 24h); queries outside this window are not captured. Actual production row counts may differ from mock data.

In the conversation, only output "Key Findings" and "Index Recommendations Summary", and inform the user that the full report is in `sql-review-report.md`.

## 6.3 Cleanup

Ask the user:

| Option | Description |
|--------|-------------|
| **Keep instance, let it auto-expire** | Instance will be automatically destroyed when TTL expires; user can connect manually to verify index effects in the meantime |
| **Destroy immediately** | Release the instance now |

Output instance connection info for the user's reference: `host`, `port`, `username`, remaining TTL. Remind user that credentials are stored in `$TMPDIR/sql_review_my.cnf`.

If user chooses to keep the instance, also output the name mapping table for reference (remind the user that table/column names on the instance are anonymized).

If user chooses to destroy immediately:

```bash
curl -s -X DELETE https://zero.polardbx.com/api/v1/instances/<instance_id>
```

Regardless of choice, delete all temporary files at the end: `$TMPDIR/sql_review_my.cnf`, `$TMPDIR/sql_review_ddl.sql`, `$TMPDIR/sql_review_*.sql`, `$TMPDIR/sql_review_name_mapping.md`, `$TMPDIR/sql_review_progress.md`.
