---
name: polardbx-standard
description: |
  Provide operational guidance, unique features, and best practices for PolarDB-X 2.0 Standard Edition (X-Cluster based). Standard Edition is 100% MySQL compatible at the SQL layer; this skill focuses on HA architecture, Lizard transaction system, Panda Index, and operational management.
  Triggers: "PolarDB-X standard", "PolarDB-X 标准版", "X-Cluster", "X-Paxos", "Panda Index", "Lizard", "SCN", "标准版运维", "standard edition", "高可用", "HA failover", "Lizard事务"
metadata:
  version: 0.2.0
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
- Contains `X-Cluster` (e.g. `8.0.32-X-Cluster-8.4.20-20251017`) -> **Standard Edition**, this skill applies.
- Contains `TDDL` (e.g. `5.7.25-TDDL-5.4.19-20251031`) -> **Enterprise Edition**, use `polardbx-sql` skill.

## Core Workflow (Follow each time)

1. Confirm the user has a Standard Edition instance. If not, use `polardbx-zero` skill to create a free temporary instance (2C4G, 30-day expiry).
2. Run `SELECT VERSION();` to verify Standard Edition (contains `X-Cluster`).
3. Identify the operation type and refer to the corresponding section below.
4. For SQL questions: Standard Edition is 100% MySQL compatible — use MySQL syntax directly, no special adaptation needed.

## Key Features Quick Reference

### X-Paxos HA

Multi-replica high availability based on X-Paxos consensus protocol. At most one Leader handles all writes; Followers participate in majority voting. Automatic failover with RPO=0 (zero data loss). Performance comparable to MySQL semi-synchronous replication.

Monitoring commands:
```sql
-- Cluster topology (Leader only, returns empty on Followers)
SELECT * FROM INFORMATION_SCHEMA.ALISQL_CLUSTER_GLOBAL;
-- Local node status (CURRENT_LEADER, INSTANCE_TYPE)
SELECT * FROM INFORMATION_SCHEMA.ALISQL_CLUSTER_LOCAL;
-- Replication health (LOG_DELAY_NUM, APPLY_DELAY_NUM)
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
