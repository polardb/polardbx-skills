# DML — Data Modification Operator EXPLAIN Output Reference

INSERT/UPDATE/DELETE/REPLACE and other data modification operators. The LogicalInsert family covers INSERT variants, LogicalModify covers UPDATE/DELETE, LogicalRelocate handles UPDATE that modifies partition keys, and LogicalModifyView wraps cross-database push-down.

---

## Input / Output Attributes

| Attribute | Input | Output |
|-----------|-------|--------|
| Columns | INSERT from VALUES/SELECT; UPDATE/DELETE from target table | Terminal operator, no business column output (returns affected row count only) |
| Order | Any | — |
| Distribution | — | — |

---

## Operator Classification and Display Names

| Operator Class | Display Name | Trigger Scenario |
|---------------|-------------|-----------------|
| LogicalInsert | `LogicalInsert` / `LogicalReplace` | INSERT, INSERT SELECT; displays LogicalReplace when isReplace=true |
| LogicalInsertIgnore | `LogicalInsertIgnore` / `LogicalReplace` / `LogicalUpsert` | INSERT IGNORE; display name switches based on isReplace/withDuplicateKeyUpdate |
| LogicalUpsert | Inherits LogicalInsertIgnore | INSERT ... ON DUPLICATE KEY UPDATE |
| LogicalReplace | `LogicalReplace` | REPLACE INTO (independent explainTermsForDisplay) |
| LogicalModify | `LogicalModify` | UPDATE / DELETE |
| LogicalRelocate | `LogicalRelocate` | UPDATE that modifies partition key |
| LogicalModifyView | `LogicalModifyView` | Cross-database UPDATE/DELETE push-down (inherits LogicalView) |
| SingleTableInsert | `PhyTableOperation` | Physical layer single-table INSERT |
| BroadcastTableModify | Delegates to DirectTableOperation | Broadcast table UPDATE/DELETE |
| LogicalTableModify | Default implementation | Calcite base class, rare in final plans |

---

## LogicalInsert

Primary INSERT operator, distinguishes two rendering modes:

### VALUES Mode (input is LogicalValues / LogicalDynamicValues)

Does not output its own fields; instead expands the post-sharding physical plan subtree (`getPhyPlanForDisplay`) as child nodes. In other words, you see PhyTableOperation list directly instead of `LogicalInsert(...)`.

### SELECT Mode (input is subquery)

```
LogicalInsert(table="<name>", columns=<rowType>, mode=<insertSelectMode>)
```

| Field | Output Condition | Description |
|-------|-----------------|-------------|
| table | Required | Target logical table name |
| columns | Required | Target column rowType |
| mode | Required | insertSelectMode enum value, describes INSERT SELECT execution mode |

---

## LogicalInsertIgnore (including LogicalUpsert)

```
LogicalInsertIgnore(table="<name>", columns=<rowType>, sql="<template>", uniqueKeySelect=[...], mode=<mode>)
```

| Field | Output Condition | Description |
|-------|-----------------|-------------|
| table | isSourceSelect=true | Target table name |
| columns | isSourceSelect=true | Target column rowType |
| sql | isSourceSelect=false (VALUES mode) | Parameterized SQL template, newlines replaced with spaces |
| uniqueKeySelect | Required | UK check SELECT list, format `select <cols> on <table>` array (grouped by UK) |
| mode | isSourceSelect=true | insertSelectMode |

Display name switches based on statement type: `LogicalInsertIgnore` / `LogicalUpsert` / `LogicalReplace`.

---

## LogicalReplace

```
LogicalReplace(table="<name>", columns=<rowType>, sql="<template>", isReturning=true, uniqueKeySelect=[...], mode=<mode>)
```

| Field | Output Condition | Description |
|-------|-----------------|-------------|
| table / columns | isSourceSelect=true | Same as LogicalInsertIgnore |
| sql | isSourceSelect=false | SQL template |
| isReturning | canUseReturning=true | Uses MySQL RETURNING optimization (replaces UK check) |
| uniqueKeySelect | canUseReturning=false | UK check SELECT list |
| mode | isSourceSelect=true | insertSelectMode |

`isReturning=true` and `uniqueKeySelect` are **mutually exclusive**: RETURNING eliminates the need for UK checks.

---

## LogicalModify

Logical operator for UPDATE / DELETE.

```
LogicalModify(TYPE="UPDATE", SET="<table.col=expr, ...>", isModifyTopN=true, optimizeByReturning=true)
LogicalModify(TYPE="DELETE", TABLES="<schema.table, ...>", isModifyTopN=true, canUseReturning=true)
```

| Field | Output Condition | Description |
|-------|-----------------|-------------|
| TYPE | Required | `UPDATE` or `DELETE` |
| SET | UPDATE required | Assignment list `table1.col1=expr1, table2.col2=expr2, ...` |
| TABLES | DELETE required | Target table list (`schema.table` comma-separated) |
| isModifyTopN | isModifyTopN=true | Contains ORDER BY + LIMIT modification |
| optimizeByReturning | Multi-write optimization using RETURNING | Already optimized multi-write via RETURNING |
| canUseReturning | Multi-write can use RETURNING but not enabled | RETURNING optimization available |

---

## LogicalRelocate

Used when UPDATE involves modifying partition keys, requiring "delete old row + insert new row".

```
LogicalRelocate(TYPE=<op>, SET="<table.col=expr, ...>", RELOCATE="<tables>", UPDATE="<tables>", isReturning=true)
```

| Field | Output Condition | Description |
|-------|-----------------|-------------|
| TYPE | Required | Operation type (UPDATE) |
| SET | Required | Assignment list, same format as LogicalModify |
| RELOCATE | relocateWriterMap non-empty | Tables requiring delete+insert (partition key was modified) |
| UPDATE | modifyWriterMap non-empty | Tables with regular UPDATE (partition key not modified) |
| isReturning | canUseReturning=true | Uses RETURNING instead of SELECT for old values |

---

## LogicalModifyView

Cross-database pushed-down UPDATE/DELETE wrapper, inherits from LogicalView. Uses LogicalView's `explainTermsForDisplay`, outputs `tables` / `shardCount` / `sql` fields — only the display name is `LogicalModifyView`. See `tablescan.md` for details.

---

## SingleTableInsert

Physical layer single-table INSERT, display name is `PhyTableOperation`.

```
PhyTableOperation(tables="<group>.[<phyTbls>]", sql="<phy_sql>", params="<v1>,<v2>,...")
PhyTableOperation(tables="<logTbl>[<partList>]", sql="<phy_sql>", params="<v1>,<v2>,...")
```

| Field | Output Condition | Description |
|-------|-----------------|-------------|
| tables | tableNames non-empty | Sharding mode: `<group>.[<phy1>,<phy2>]`; Partition mode: `<logTbl>[<part1>,<part2>]` |
| groups | When tables is absent | Group name only |
| sql | Required | Physical SQL (newlines replaced with spaces) |
| params | When parameters non-empty | Parameter value list (comma-separated, TableName outputs table name) |

---

## BroadcastTableModify

UPDATE/DELETE on broadcast tables. `explainTermsForDisplay` fully delegates to internal `DirectTableOperation` (displayed as `LogicalView`).

---

## Examples

### INSERT VALUES

```
PhyTableOperation(tables="..._00.[t_p1]", sql="INSERT INTO `t` (`v1`, `v2`) VALUES (?, ?)", params="1,10")
```

In VALUES mode, LogicalInsert does not display itself, directly expanding to PhyTableOperation for each shard.

### INSERT SELECT

```
LogicalInsert(table="t_dst", columns=ROW(v1 INT, v2 INT), mode=PUSHDOWN)
  LogicalView(tables="t_src[p1,p2]", shardCount=2, sql="SELECT `v1`, `v2` FROM `t_src`")
```

### INSERT IGNORE

```
LogicalInsertIgnore(sql="INSERT IGNORE INTO `t` ...", uniqueKeySelect=[select v1 on t_uk_idx])
```

### INSERT ON DUPLICATE KEY UPDATE

```
LogicalUpsert(sql="INSERT INTO `t` ... ON DUPLICATE KEY UPDATE `v2` = ?", uniqueKeySelect=[select v1 on t_uk_idx])
```

### REPLACE with RETURNING Optimization

```
LogicalReplace(sql="REPLACE INTO `t` ...", isReturning=true)
```

### UPDATE

```
LogicalModify(TYPE="UPDATE", SET="t.v2=?0, t.v3=?1")
  LogicalView(tables="t[p1,p2,p3]", shardCount=3, sql="SELECT `v1`, `v2`, `v3` FROM `t` WHERE (`v1` > ?)")
```

### DELETE

```
LogicalModify(TYPE="DELETE", TABLES="test.t")
  LogicalView(tables="t[p1,p2,p3]", shardCount=3, sql="SELECT `v1` FROM `t` WHERE (`v1` < ?)")
```

### UPDATE Modifying Partition Key

```
LogicalRelocate(TYPE="UPDATE", SET="t.v1=?0", RELOCATE="t")
  LogicalView(tables="t[p1,p2,p3]", shardCount=3, sql="...")
```

`RELOCATE="t"` indicates t requires delete+insert because partition key v1 was modified.

### Cross-Database UPDATE Push-Down

```
LogicalModifyView(tables="t[p1,p2]", shardCount=2, sql="UPDATE `t` SET `v2` = ? WHERE `v1` = ?")
```

### Broadcast Table DELETE

```
LogicalView(tables="bcast_t[p1]", sql="DELETE FROM `bcast_t` WHERE `v1` = ?")
```

BroadcastTableModify delegates to DirectTableOperation, displayed as LogicalView.
