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

## Key Features

- **X-Paxos HA**: Multi-node consensus protocol, automatic failure detection and leader election.
- **Lizard Transaction System**: SCN-based MVCC replacing InnoDB's transaction visibility, 30% higher throughput and 53% lower latency than MySQL 8.0.32.
- **Panda Index**: Deadlock-free unique index that eliminates Gap locks under RC isolation level.
- **100% MySQL Compatible**: Full support for stored procedures, triggers, EVENTs, etc.

## References

- `skills/polardbx-standard/references/x-paxos-ha.md` - X-Paxos HA: consensus protocol, automatic failover, cluster monitoring SQL.
- `skills/polardbx-standard/references/lizard-transaction.md` - Lizard transaction system: SCN-based MVCC, Cleanout optimization, performance benchmark.
- `skills/polardbx-standard/references/panda-index.md` - Panda Index: deadlock-free unique index, eliminates Gap locks under RC isolation.

## Playbooks

Executable end-to-end verification steps. Agent should auto-execute all steps when instructed to "follow the instructions".

- `skills/polardbx-standard/playbooks/test-panda-index.md` - Verify Panda Index eliminates Gap locks: establish baseline with regular unique key (confirm Gap locks exist), then verify no Gap locks with Panda Index.
