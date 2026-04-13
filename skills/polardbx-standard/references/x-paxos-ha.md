---
title: X-Paxos High Availability
---

# X-Paxos High Availability

PolarDB-X Standard Edition uses the X-Paxos consensus protocol for multi-replica high availability with automatic failover and zero data loss.

## Prerequisites

- PolarDB-X 2.0 Standard Edition (X-Cluster based).

## Architecture

Supports multiple disaster recovery topologies: single availability zone, three availability zones, two-region-three-center, etc.

- **At most one Leader** node at any time handles all writes.
- Remaining nodes serve as **Followers**, participating in majority voting and data synchronization.
- Optional **Learner** nodes for read replicas without voting rights.

## Core Mechanism

### Log Fusion

The consensus protocol log is deeply integrated with MySQL native binlog:

- The Leader appends consensus-related events during the write path.
- Followers use the consensus log in place of traditional Relay Log.
- The SQL thread replays the log into the underlying data files.
- Functionally, the consensus log is equivalent to MySQL binlog.

### Automatic Failover

- Heartbeat and election timeout mechanisms continuously monitor the Leader's liveness.
- When the Leader fails, Followers automatically trigger a leader election.
- The newly elected Leader must wait for the SQL thread to finish replaying all pending logs before serving traffic, ensuring data consistency.

## Key Benefits

- **High performance**: Single-Leader write design, performance comparable to MySQL semi-synchronous replication.
- **Strong consistency**: Zero data loss (RPO=0) via majority-based synchronous log replication.
- **Native HA**: Built-in election and heartbeat mechanism replaces external HA monitoring components required by traditional MySQL setups.

## Monitoring SQL Commands

### Cluster Global Status (Leader only)

```sql
SELECT * FROM INFORMATION_SCHEMA.ALISQL_CLUSTER_GLOBAL;
```

Returns cluster topology. Only the Leader node returns data; non-Leader nodes return empty results.

Key fields:
- `ROLE`: Node role — Leader, Follower, or Learner.
- `ELECTION_WEIGHT`: Election priority, used in cross-AZ disaster recovery to prefer same-AZ replicas.
- `MATCH_INDEX`, `NEXT_INDEX`, `APPLIED_INDEX`: Consensus log synchronization progress.

### Local Instance Status

```sql
SELECT * FROM INFORMATION_SCHEMA.ALISQL_CLUSTER_LOCAL;
```

Key fields:
- `CURRENT_LEADER`: Address and port of the current Leader node.
- `INSTANCE_TYPE`: Replica type — Normal or Log.

### Replication Health (Leader only)

```sql
SELECT * FROM INFORMATION_SCHEMA.ALISQL_CLUSTER_HEALTH;
```

Key fields:
- `LOG_DELAY_NUM`: Consensus log network transmission delay (analogous to MySQL Relay Log delay).
- `APPLY_DELAY_NUM`: Follower SQL thread replay lag.
