# Step 2: 阿里云数据源采集（阿里云路径）

> **适用场景**: 用户选择"阿里云实例"扫描模式时执行此步骤，替代代码扫描路径。
>
> **前提**: 用户无需有代码仓库，仅凭阿里云实例 ID 即可进行 SQL 审查。

## 先决条件检查

### 1. 检测 aliyun CLI

```bash
aliyun version
```

如果不可用，提示用户安装：
- macOS: `brew install aliyun-cli`
- Linux: `curl -fsSL https://aliyuncli.alicdn.com/aliyun-cli-linux-latest-amd64.tgz | tar xz && sudo mv aliyun /usr/local/bin/`
- Windows: `scoop install aliyun` 或从 GitHub Releases 下载

### 2. 验证凭证配置

```bash
aliyun configure list
```

确认有至少一个 profile 且包含有效的 AccessKey。如果未配置，提示用户执行 `aliyun configure`。

### 3. 验证实例访问权限

```bash
aliyun polardbx DescribeDBInstanceAttribute --RegionId <region> --DBInstanceName <instance_id>
```

如果返回错误（权限不足或实例不存在），提示用户检查 RAM 权限和实例 ID。

## 用户信息收集

询问用户以下信息（用 AskUserQuestion 一次性收集）：

| 信息项 | 说明 | 默认值 |
|--------|------|--------|
| **实例 ID** | PolarDB-X 实例 ID（如 `pxc-xxxxx`） | 必填 |
| **地域** | 实例所在 RegionId（如 `cn-hangzhou`, `cn-shanghai`） | 必填 |
| **时间范围** | 分析的时间窗口 | 最近 24 小时 |
| **API 路径偏好** | DAS API（推荐）/ SLS API / 两者都用 | DAS API |

如果用户选择 SLS API 路径，**额外收集**：
- SLS Project 名称（不同实例/地域的名称不同，需用户确认）
- SLS Logstore 名称（同上）

## 2.1 获取 DDL

优先直接从实例导出：

### 方式一：直接连接实例（推荐）

如果用户有 mysql 连接权限，直接连接实例执行：

```sql
-- 获取所有业务库
SHOW DATABASES;
-- 排除系统库: information_schema, mysql, performance_schema, sys, __cdc__

-- 对用户选定的数据库，获取所有表的 DDL
USE <database_name>;
SHOW TABLES;
SHOW CREATE TABLE <table_name>;
```

### 方式二：用户提供 DDL 文件

如果无法直接连接，让用户提供 DDL 文件路径。支持：
- 单个 `.sql` 文件包含所有 CREATE TABLE
- 一个目录下多个 `.sql` 文件

### DDL 处理

将所有收集到的 DDL 写入进度文件的 `## DDL Inventory` 部分，格式与代码扫描模式一致：

```markdown
## DDL Inventory
| # | Table | PK | Columns | Source |
|---|-------|----|---------|--------|
| T1 | orders | id | 15 | 阿里云实例 <instance_id> |
| T2 | order_items | id | 8 | 阿里云实例 <instance_id> |
```

> **注意**: Source 列标记为"阿里云实例"而非文件路径。

## 2.2 获取 SQL 洞察统计（DAS API）

SQL 洞察提供 SQL 模板级别的聚合统计，是最高效的采集方式。

### 调用 API

```bash
aliyun das GetFullRequestStatResultByInstanceId \
  --RegionId cn-shanghai \
  --InstanceId "<instance_id>" \
  --Role <polarx_cn|polarx_dn> \
  --Start <start_ms> \
  --End <end_ms> \
  --PageNo 1 --PageSize 100
```

**重要参数**：
- `RegionId` 固定为 `cn-shanghai`（DAS 服务地域，非实例地域）
- `Role`：**必填**，标准版用 `polarx_dn`，企业版用 `polarx_cn`（不传此参数会返回空数据）
- `Start` / `End` 为**毫秒**时间戳
- 时间间隔不超过 1 天，最远 30 天
- 分页：`TotalCount > PageSize` 时循环递增 `PageNo`

### 异步轮询

此接口为异步接口，返回结果中 `IsFinish` 字段：
- `IsFinish = false`：等待 2-3 秒后，**用相同的 Start/End 参数重新调用**（服务端按参数缓存结果）
- `IsFinish = true`：数据就绪，从 `Data.Result.List` 读取

### 数据过滤

从返回结果中过滤：
- **排除**系统 SQL：`SHOW`, `SET`, `COMMIT`, `ROLLBACK`, `BEGIN`, `USE`, `DESCRIBE`, `EXPLAIN`
- **排除**纯 INSERT（无 WHERE 子句的写入操作）
- **保留**：SELECT, UPDATE, DELETE, INSERT...ON DUPLICATE, 及包含子查询的 INSERT

### 提取字段

对每条保留的 SQL 模板，提取：

| 字段 | DAS 返回字段 | 说明 |
|------|-------------|------|
| SQL 模板 | `Psql` | 参数已用 `?` 替代 |
| 执行次数 | `Count` | 时间窗口内总执行次数 |
| 平均耗时 | `AvgRt` | 平均响应时间（毫秒） |
| 平均扫描行数 | `AvgExaminedRows` | 平均扫描行数 |
| 数据库名 | `Database` | 所属数据库 |

按 `Count`（执行次数）降序排序。

## 2.3 获取慢查询（DAS API 或 PolarDB-X 原生 API）

慢查询记录提供完整的 SQL 文本（非模板），包含实际参数值。

### 方式一：DAS API（推荐）

```bash
aliyun das DescribeSlowLogRecords \
  --region cn-shanghai \
  --InstanceId "<instance_id>" \
  --NodeId "<node_id>" \
  --StartTime <start_ms> \
  --EndTime <end_ms> \
  --PageNumber 1 --PageSize 100
```

> **注意**: DAS API 无 `--RegionId` 参数，使用 `--region cn-shanghai` 全局参数指定 DAS 服务地域。PolarDB-X 实例必须传 `--NodeId`（DN 节点 ID），可通过 `DescribeDBInstanceAttribute` 的 `DBNodes` 字段获取。

提取字段：
| 字段 | 返回字段 | 说明 |
|------|---------|------|
| 完整 SQL | `SQLText` | 包含实际参数值 |
| SQL 模板 | `Psql` | 用于与 SQL 洞察去重 |
| 查询耗时 | `QueryTime` | 秒 |
| 扫描行数 | `RowsExamined` | — |

### 方式二：PolarDB-X 原生 API

```bash
aliyun polardbx DescribeSlowLogRecords \
  --RegionId <instance_region> \
  --DBInstanceName "<instance_id>" \
  --CharacterType <polarx_cn|polarx_dn> \
  --DBNodeIds "<node_id>" \
  --StartTime "<start_ISO>" \
  --EndTime "<end_ISO>" \
  --Page 1 --PageSize 100
```

> **注意**:
> - `RegionId` 使用实例实际所在地域
> - `StartTime`/`EndTime` 使用 ISO 8601 格式（如 `2026-04-09T02:26Z`）
> - `CharacterType`：企业版用 `polarx_cn`（计算节点），标准版用 `polarx_dn`（数据节点）
> - `DBNodeIds`：查询 DN 时为必填项，通过 `DescribeDBInstanceAttribute` 获取节点 ID
> - 分页参数为 `--Page`（不是 `--PageNumber`）

提取字段：
| 字段 | 返回字段 | 说明 |
|------|---------|------|
| 完整 SQL | `SQLText` | 包含实际参数值 |
| 查询耗时 | `QueryTimeMS` | 毫秒 |
| 扫描行数 | `ParseRowCounts` | — |
| 返回行数 | `ReturnRowCounts` | — |

分页获取所有慢查询记录，按 `QueryTime` 降序排序。

## 2.4 获取 SQL 审计日志（SLS API，可选）

> **仅当用户选择 SLS API 路径时执行**。SLS 提供完整的原始 SQL 日志，但需要用户提供 Project 和 Logstore 名称。

```bash
aliyun sls GetLogs \
  --project "<用户提供的 SLS Project>" \
  --logstore "<用户提供的 SLS Logstore>" \
  --from <unix_timestamp> --to <unix_timestamp> \
  --query "* | SELECT sql, response_time, affect_rows, scanned_rows, sql_type, db_name WHERE response_time > 0 ORDER BY response_time DESC LIMIT 500"
```

提取字段：
| 字段 | SLS 字段 | 说明 |
|------|---------|------|
| 完整 SQL | `sql` | 原始 SQL 全文 |
| 响应时间 | `response_time` | 微秒 |
| 影响行数 | `affect_rows` | — |
| 扫描行数 | `scanned_rows` | — |
| SQL 类型 | `sql_type` | SELECT / UPDATE / DELETE 等 |
| 数据库名 | `db_name` | — |

应用与 2.2 相同的过滤规则（排除系统 SQL、纯 INSERT 等）。

## 2.5 合并与去重

### 去重策略

以 **SQL 模板**（Psql）作为去重 key：
1. SQL 洞察的 `Psql` 已经是参数化模板（`?` 占位符）
2. 慢查询的 `SQLText` 是完整 SQL — 需要提取其对应的模板（DAS 返回中有 `Psql` 字段，PolarDB-X 原生 API 没有，需要手动将字面值替换为 `?`）
3. SLS 审计日志的 `sql` 也是完整 SQL — 同样需要模板化

### 合并规则

对每条唯一的 SQL 模板：

| 属性 | 来源 | 合并规则 |
|------|------|---------|
| SQL 模板 | 各 API 的 Psql/模板化结果 | 去重 key |
| 完整 SQL 示例 | 慢查询/审计的 SQLText | 保留一条有实际参数的示例 |
| 执行次数 | SQL 洞察 Count | 直接使用（最权威） |
| 平均耗时 | SQL 洞察 AvgRt | 直接使用 |
| 最大耗时 | 慢查询 QueryTime | 取最大值 |
| 平均扫描行数 | SQL 洞察 AvgExaminedRows | 直接使用 |
| 来源标记 | — | insight / slow / audit（可多个） |

### 按表分组

根据 SQL 中引用的表名，将 SQL 分配到对应的表。多表 JOIN 查询归入驱动表（FROM 后的第一个表）。

## 2.6 保存到进度文件

将合并去重后的结果追加到 `$TMPDIR/sql_review_progress.md`，格式扩展如下：

```markdown
## Config
- Scope: 阿里云实例
- Instance: <instance_id>
- Region: <region>
- Time Range: <start> ~ <end>
- API Path: DAS / SLS / Both
- Edition: PolarDB-X Enterprise

## DDL Inventory
| # | Table | PK | Columns | Source |
|---|-------|----|---------|--------|

## Queries by Table
### Table: <table_name> (来源: 阿里云 SQL 洞察)
| # | SQL 模板 | Type | Freq | AvgRt(ms) | AvgRows | Source | SQL | Features |
|---|---------|------|------|-----------|---------|--------|-----|----------|
| Q1 | SELECT ... WHERE ... | SELECT | 12580 | 3.2 | 150 | insight,slow | SELECT ... WHERE id = 123 | JOIN, ORDER BY |
| Q2 | UPDATE ... SET ... WHERE ... | UPDATE | 890 | 15.6 | 5000 | insight | — | — |
```

> **格式兼容**: 与代码扫描模式的进度文件格式兼容，后续步骤（脱敏→实例→EXPLAIN→报告）无缝衔接。

## 完成后

下一步进入 Step 3（名称脱敏），流程与代码扫描模式相同。
