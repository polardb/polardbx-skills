# Data Import Task APIs

PolarDB-X data and structure import task APIs. All CLI examples use the `aliyun polardbx` subcommand in plugin mode.

> Note: Region flag is `--biz-region-id`; instance identifier flag is `--db-instance-name`. Import tasks are identified by `--slink-task-id`.

> **Security:** Credentials and database passwords are sensitive. NEVER echo or hardcode them; use placeholders.

---

## CreateDataImportTask

Create a data import task. The API supports importing external data files such as SQL or CSV into a target database instance.

> **[MUST] Secondary confirmation required.** Data import writes to the target database. Confirm source/target and scope with the user; never echo passwords.

### Required parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `biz-region-id` | String | Region where the instance resides. CLI mapping for API `RegionId`. |
| `db-instance-name` | String | Instance ID. CLI mapping for API `DBInstanceName`. |
| `slink-task-id` | String | Import task ID. Expected format uses an `etx-` prefix. |
| `src-res-id` | String | Source RDS instance ID. |
| `src-db` | String | Source database information when the source database is RDS MySQL. The source database must be the same as the target database. |
| `src-user-name` | String | Source username. |
| `src-password` | String | Source-side execution mode for the import task. Valid values: `rw` (read/write), `ro` (read-only). |
| `dst-res-id` | String | Migration task ID. |
| `dst-db` | String | Target SQL execution environment. Valid values: `importing`, `success`, `fail`. |
| `dst-user-name` | String | Target username. |
| `dst-password` | String | Password of the privileged target RDS instance account. Effective only when `dstpassword=true`. Never echo. |

### CLI example

```bash
aliyun polardbx create-data-import-task \
  --biz-region-id cn-hangzhou \
  --db-instance-name pxc-******** \
  --slink-task-id etx-******** \
  --src-res-id rm-******** \
  --src-db <src-db> --src-user-name <user> --src-password <rw|ro> \
  --dst-res-id <migration-task-id> \
  --dst-db <importing|success|fail> --dst-user-name <user> --dst-password '<password>' \
  --user-agent AlibabaCloud-Agent-Skills/alibabacloud-polardbx-ops/{session-id}
```

### RAM action

`polardbx:CreateDataImportTask`

---

## DescribeDataImportTaskInfo

Query the execution details of a data import task.

### Required parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `biz-region-id` | String | Region where the instance resides |
| `slink-task-id` | String | Task ID |

### Optional parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `success-page-number` / `success-page-size` | Integer | Pagination of success records |
| `fail-page-number` / `fail-page-size` | Integer | Pagination of failure records |

### CLI example

```bash
aliyun polardbx describe-data-import-task-info \
  --biz-region-id cn-hangzhou \
  --slink-task-id etx-******** \
  --connect-timeout 3 --read-timeout 10 \
  --user-agent AlibabaCloud-Agent-Skills/alibabacloud-polardbx-ops/{session-id}
```

### RAM action

`polardbx:DescribeDataImportTaskInfo`

---

## RestartDataImportTask

Restart a data import task.

### Required parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `biz-region-id` | String | Region where the instance resides |

### Optional parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `slink-task-id` | String | Target task ID |
| `page-number` / `page-size` | Integer | Pagination (page-size: 30/50/100) |

### CLI example

```bash
aliyun polardbx restart-data-import-task \
  --biz-region-id cn-hangzhou \
  --slink-task-id etx-******** \
  --user-agent AlibabaCloud-Agent-Skills/alibabacloud-polardbx-ops/{session-id}
```

### RAM action

`polardbx:RestartDataImportTask`

---

## StopDataImportTask

Stop a data import task.

> **[MUST] Secondary confirmation required.** Stopping an import may leave partial data. Confirm with the user.

### Required parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `biz-region-id` | String | Region where the instance resides |

### Optional parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `slink-task-id` | String | Task ID |
| `page-number` / `page-size` | Integer | Pagination (page-size: 30/50/100) |

### CLI example

```bash
aliyun polardbx stop-data-import-task \
  --biz-region-id cn-hangzhou \
  --slink-task-id etx-******** \
  --user-agent AlibabaCloud-Agent-Skills/alibabacloud-polardbx-ops/{session-id}
```

### RAM action

`polardbx:StopDataImportTask`

---

## CreateStructureImportTask

Create a database structure (DDL) import task.

> **[MUST] Secondary confirmation required.** Structure import executes DDL on the target. Confirm with the user.

### Required parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `biz-region-id` | String | Region where the instance resides |
| `slink-task-id` | String | Target task ID |

### Optional parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `db-instance-name` | String | Instance ID |
| `config` | String | Configuration info |

### CLI example

```bash
aliyun polardbx create-structure-import-task \
  --biz-region-id cn-hangzhou \
  --slink-task-id etx-******** \
  --db-instance-name pxc-******** \
  --user-agent AlibabaCloud-Agent-Skills/alibabacloud-polardbx-ops/{session-id}
```

### RAM action

`polardbx:CreateStructureImportTask`

---

## DescribeStructureImportTaskInfo

Query the details of a structure import task.

### Required parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `biz-region-id` | String | Region where the instance resides |
| `slink-task-id` | String | Target task ID |

### CLI example

```bash
aliyun polardbx describe-structure-import-task-info \
  --biz-region-id cn-hangzhou \
  --slink-task-id etx-******** \
  --connect-timeout 3 --read-timeout 10 \
  --user-agent AlibabaCloud-Agent-Skills/alibabacloud-polardbx-ops/{session-id}
```

### RAM action

`polardbx:DescribeStructureImportTaskInfo`

---

## RefreshImportMeta

Refresh the metadata of an import task.

### Required parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `biz-region-id` | String | Region ID |
| `slink-task-id` | String | Task ID |

### Optional parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `db-instance-name` | String | Instance ID |

### CLI example

```bash
aliyun polardbx refresh-import-meta \
  --biz-region-id cn-hangzhou \
  --slink-task-id etx-******** \
  --user-agent AlibabaCloud-Agent-Skills/alibabacloud-polardbx-ops/{session-id}
```

### RAM action

`polardbx:RefreshImportMeta`
