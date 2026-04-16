---
name: polardbx-standard
description: |
  Provide operational guidance, unique features, and best practices for PolarDB-X 2.0 Standard Edition (X-Cluster based). Standard Edition is 100% MySQL compatible at the SQL layer; this skill focuses on HA architecture, Lizard transaction system, Panda Index, and operational management.
  Triggers: "PolarDB-X standard", "PolarDB-X 标准版", "X-Cluster", "X-Paxos", "Panda Index", "Lizard", "SCN", "标准版运维", "standard edition", "高可用", "HA failover", "Lizard事务"
metadata:
  version: 0.2.1
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

### 100% MySQL Compatible

Full support for stored procedures, triggers, EVENTs, etc. Use standard MySQL syntax for all SQL tasks.

## References

| Reference | Description |
|-----------|-------------|
| [references/x-paxos-ha.md](references/x-paxos-ha.md) | X-Paxos HA: architecture, log fusion, automatic failover, cluster monitoring SQL |
| [references/lizard-transaction.md](references/lizard-transaction.md) | Lizard transaction system: SCN-based MVCC, Cleanout optimization, performance benchmark |
| [references/panda-index.md](references/panda-index.md) | Panda Index: deadlock-free unique key, enable/disable, upgrade existing indexes, FAQ |

## Playbooks

Executable end-to-end verification steps. Agent should auto-execute all steps when instructed to "follow the instructions".

| Playbook | Description |
|----------|-------------|
| [playbooks/test-panda-index.md](playbooks/test-panda-index.md) | Verify Panda Index eliminates Gap locks: baseline with regular unique key, then verify with Panda Index |
