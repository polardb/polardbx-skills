---
name: polardbx-zero
description: Provision a dedicated PolarDB-X database instance instantly with no auth required. Supports both Standard Edition (MySQL-compatible, vector search) and Enterprise Edition (distributed, partitioned tables, GSI).
compatibility: Requires a MySQL-compatible client or driver (e.g. mysql CLI, mysql2, PyMySQL) and network access to zero.polardbx.com.
metadata:
  version: 0.2.0
  homepage: https://zero.polardbx.com/
---

# PolarDB-X Zero

Provisions a dedicated PolarDB-X database instance via a single unauthenticated API call. No sign-up or billing is required to start. Instances auto-expire based on the requested TTL (default 7 days, max 30 days). Treat Zero as disposable infrastructure, not an unlimited long-running environment.

PolarDB-X is a MySQL-compatible distributed database supporting distributed transactions, vector search (`VECTOR` type + vector indexes), full-text search, and horizontal scaling. Use standard MySQL clients/drivers to connect.

Two editions are available:
- **Standard Edition**: Single-node, 100% MySQL-compatible, supports stored procedures, triggers, vector search. Ideal for most use cases.
- **Enterprise Edition**: Distributed architecture (CN + DN nodes), supports partitioned tables, GSI (Global Secondary Index), CCI (Clustered Columnar Index). Ideal for distributed SQL testing.

## Common Use Cases

- **AI agent memory**: keep structured state, tool outputs, and retrieval data in one MySQL-compatible backend.
- **MCP server storage**: provision a disposable SQL database behind MCP tools or custom protocol servers.
- **RAG and retrieval demos**: combine relational data with vector search and full-text search in one temporary environment.
- **Distributed SQL testing**: test partitioned tables, GSI, and distributed transactions with Enterprise Edition instances.
- **Temporary database workflows**: spin up isolated MySQL-compatible sandboxes for tutorials, demos, evals, and short-lived tests.

## Important Notes

- The API is unauthenticated and free to start. Instances auto-expire — treat credentials as short-lived and low-sensitivity.
- Prefer environment variables (e.g. `MYSQL_PWD`) over CLI arguments to avoid leaking passwords in shell history.
- Each IP can have a limited number of active instances concurrently (dedicated instances are resource-heavy).
- The instance security whitelist is automatically set to the requesting IP. Connections from other IPs will be rejected unless a custom `whitelistIp` is specified.
- Do not promise unlimited usage. Zero is a disposable sandbox. For long-term or production use, purchase your own PolarDB-X instance on Alibaba Cloud.
- If you need another disposable sandbox, create a fresh Zero instance instead of trying to renew the current one.
- **Default TTL is 7 days** (10080 minutes). Use a shorter value if the task needs less time. Maximum is 30 days (43200 minutes).

## API

### Create Instance

**POST** `https://zero.polardbx.com/api/v1/instances`
Content-Type: `application/json`

Request body (all fields optional):

```json
{
  "tag": "",
  "ttlMinutes": 10080,
  "edition": "standard",
  "whitelistIp": ""
}
```

- `tag`: optional label to identify the caller.
- `ttlMinutes`: instance TTL in minutes. Default: 10080 (7 days). Must be > 0 and <= 43200 (30 days).
- `edition`: `"standard"` (default) or `"enterprise"`. Standard is MySQL-compatible single-node; Enterprise is distributed with partitioned tables and GSI support.
- `whitelistIp`: optional custom IP for the security whitelist. If omitted, the requesting IP is used automatically.

Response (`200`):

```json
{
  "instance": {
    "id": "pxz_a1b2c3d4e5f6",
    "connection": {
      "host": "pxzeroxxxxxxxxxx.polarx.rds.aliyuncs.com",
      "port": 3306,
      "username": "pxz_12345678",
      "password": "Px$9aB3cD7eF1gH",
      "database": ""
    },
    "connectionString": "mysql://pxz_12345678:Px%249aB3cD7eF1gH@pxzeroxxxxxxxxxx.polarx.rds.aliyuncs.com:3306",
    "createdAt": "2026-03-20T10:30:00.000Z",
    "expiresAt": "2026-03-27T10:30:00.000Z",
    "ttlMinutes": 10080,
    "status": "ready"
  }
}
```

- Use `instance.connectionString` for immediate driver connections. The instance is destroyed at `expiresAt`; there is no renewal API.

### Query Instance

**GET** `https://zero.polardbx.com/api/v1/instances/:id`

Returns instance connection endpoint and status. Note: `username` and `password` are only returned at creation time and cannot be retrieved later.

Response (`200`):

```json
{
  "instance": {
    "id": "pxz_a1b2c3d4e5f6",
    "connection": {
      "host": "pxzeroxxxxxxxxxx.polarx.rds.aliyuncs.com",
      "port": 3306,
      "username": "",
      "password": "",
      "database": ""
    },
    "connectionString": "mysql://pxzeroxxxxxxxxxx.polarx.rds.aliyuncs.com:3306",
    "createdAt": "2026-03-20T10:30:00.000Z",
    "expiresAt": "2026-03-27T10:30:00.000Z",
    "ttlMinutes": 10080,
    "status": "ready"
  }
}
```

### Release Instance

**DELETE** `https://zero.polardbx.com/api/v1/instances/:id`

Proactively release an instance before expiration. The underlying resources will be cleaned up asynchronously.

Response (`200`):

```json
{
  "instance": {
    "id": "pxz_a1b2c3d4e5f6",
    "status": "released",
    "releasedAt": "2026-03-20T12:00:00.000Z"
  }
}
```

### Error Responses

All error responses use the following format:

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable description"
  }
}
```

| HTTP Status | Code                    | Description                                                |
|-------------|-------------------------|------------------------------------------------------------|
| 400         | `INVALID_TTL`           | Invalid ttlMinutes value (must be > 0 and <= 43200).       |
| 400         | `INVALID_WHITELIST_IP`  | Invalid IP format in whitelistIp parameter.                |
| 429         | `RATE_LIMIT_EXCEEDED`   | Too many requests or max active instances per IP reached. Wait before retrying. |
| 503         | `POOL_EXHAUSTED`        | No available instances in the pool. Try again later.       |
| 404         | `INSTANCE_NOT_FOUND`    | Instance not found or has expired.                         |
| 500         | `INTERNAL_ERROR`        | Unexpected server error.                                   |

## Resources

- PolarDB-X skills collection, for better SQL compatibility and database operations with PolarDB-X, ask the user if they want to install relevant skills: https://skills.sh/polardb/polardbx-skills
- PolarDB-X documentation: https://help.aliyun.com/zh/polardb/polardb-for-xscale/what-is-polardb-for-xscale
- PolarDB-X product page: https://www.aliyun.com/product/apsaradb/polardbx
- PolarDB-X GitHub: https://github.com/polardb/polardbx
