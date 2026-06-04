# RexExplainVisitor — RexNode Expression Display Rules

Multiple operators' `explainTermsForDisplay` share `RexExplainVisitor` to convert `RexNode` trees into readable SQL-style strings in EXPLAIN output.

---

## Conversion Rules

| RexNode Type | Output Format | Example |
|-------------|--------------|---------|
| RexInputRef | Resolved to **column name** at the corresponding position in the input RelNode | `id`, `v1` (not `$0`, `$1`) |
| RexLiteral | Direct value output | `10`, `'hello'`, `null` |
| RexDynamicParam | `?N`, N is the parameter index | `?0`, `?1` |
| RexCorrelVariable | `$corN` | `$cor0` |
| RexFieldAccess | `$corN.column_name`, correlation variable field access | `$cor0.id` |
| RexSubQuery | Direct toString() | `EXISTS(...)` |
| RexOver | Direct toString() | `SUM(...) OVER (...)` |
| RexSystemVar | Direct toString() | `@@version` |
| RexUserVar | Direct toString() | `@my_var` |

---

## Operator Conversion Rules

### SqlBinaryOperator

| Operator Type | Output Rule | Example |
|--------------|------------|---------|
| Standard comparison (`=`, `>`, `<`, `>=`, `<=`, `<>`) | `left operator right` | `id > 10`, `v1 = v2` |
| AND | Sub-conditions joined with ` AND `; nested OR gets parentheses | `v1 > 10 AND (v2 = 1 OR v2 = 2)` |
| OR | Sub-conditions joined with ` OR `; nested AND gets parentheses | `v1 = 1 OR (v2 > 5 AND v3 < 10)` |

Parenthesis rule: OR nested inside AND, or AND nested inside OR, the nested sub-expression is automatically parenthesized; no parentheses in other cases.

### SqlFunction

```
function_name(arg1, arg2, ...)
```

Examples: `SUBSTR(name, 1, 3)`, `IFNULL(v1, 0)`

### SqlPrefixOperator / SqlPostfixOperator

```
operator_name(operand)
```

Examples: `NOT(v1 IS NULL)`, `IS NULL(v1)`

### SqlLikeOperator

```
left_operand LIKE right_operand
```

Example: `name LIKE '%test%'`

### SqlCaseOperator

Uses `RexCall.toString()` directly for the complete CASE expression.

Example: `CASE WHEN v1 > 0 THEN 'pos' ELSE 'neg' END`

### ROW Operator

```
ROW(arg1, arg2, ...)
```

Same format as function calls.

---

## Column Name Resolution Logic

`RexExplainVisitor.getField(index)` resolves `RexInputRef` integer index to column name:

1. Iterate over all input RelNodes of the parent node
2. Starting from the first input, if index < that input's column count, take column at position index from that input's `rowType`
3. Otherwise subtract that input's column count from index, continue checking the next input

For single-input operators (e.g., Filter, Project), index directly corresponds to the input RelNode's column offset. For dual-input operators (e.g., Join), left table columns come first, right table columns follow, index resolves in concatenation order.
