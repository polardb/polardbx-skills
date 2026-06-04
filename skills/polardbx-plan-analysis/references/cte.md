# CTE — Common Table Expression Operator EXPLAIN Output Reference

Producer-Consumer model: CTEAnchor connects Producer and Consumer, Producer materializes CTE results, Consumer references the results.

---

## Input / Output Attributes

| Operator | Input | Output Columns | Order/Distribution |
|----------|-------|---------------|-------------------|
| CTEAnchor | Left: Producer, Right: main query | Right child's columns | Determined by right child |
| CTEProducer | CTE definition query columns | Same as input | No guarantee |
| CTEConsumer | No child nodes (reads from Producer) | Defined by projects (original columns when no projects) | No guarantee |

---

## CTEAnchor

```
LogicalCTEAnchor(cte_id=<N>)
CTEAnchor(cte_id=<N>)
```

Dual-input operator: left child is CTEProducer, right child is the main query referencing this CTE.

**cte_id** (required): CTE unique identifier, Anchor/Producer/Consumer are linked through this ID.

---

## CTEProducer

```
LogicalCTEProducer(cte_id=<N>)
CTEProducer(cte_id=<N>)
```

Single-input operator, child node is the plan tree of the CTE definition query. cte_id matches the Anchor.

---

## CTEConsumer

```
LogicalCTEConsumer(cte_id="<cteId>_<sn>", projects=[...], conditions=[...])
CTEConsumer(cte_id="<cteId>_<sn>", projects=[...], conditions=[...])
```

### Field Descriptions

| Field | Output Condition | Description |
|-------|-----------------|-------------|
| cte_id | Required | Format `<cteId>_<sn>`. sn is a sequence number; the same CTE referenced N times produces N Consumers (sn increments from 0) |
| projects | When non-empty | Column pruning pushed down by CTE Optimize, RexNode list |
| conditions | When non-empty | Predicate conditions pushed down by CTE Optimize, RexNode list |

### Execution Semantics

**Filter before project**: Column indices in conditions reference the CTE original column space (Producer's rowType), projects will change the column space. Equivalent to:

```
Project(projects)
  Filter(conditions)
    <CTE materialized result>
```

LogicalCTEConsumer additionally supports `EXPLAIN_CTE_CONSUMER` parameter to display innerRel subtree.

---

## Examples

### Single Reference

```
LogicalCTEAnchor(cte_id=0)
  LogicalCTEProducer(cte_id=0)
    LogicalView(...)
  LogicalCTEConsumer(cte_id="0_0")
```

### Multiple References

```
LogicalCTEAnchor(cte_id=0)
  LogicalCTEProducer(cte_id=0)
    LogicalView(...)
  HashJoin(...)
    LogicalCTEConsumer(cte_id="0_0")
    LogicalCTEConsumer(cte_id="0_1")
```

Same CTE referenced twice, sn values are 0 and 1 respectively.

### With projects and conditions

```
LogicalCTEConsumer(cte_id="0_0", projects=[$0, $2], conditions=[>($0, 10)])
```

projects=[$0, $2]: Only columns 0 and 2. conditions=[>($0, 10)]: Column 0 > 10.

### Multiple CTE Definitions

```
LogicalCTEAnchor(cte_id=0)
  LogicalCTEProducer(cte_id=0)
    LogicalView(...)
  LogicalCTEAnchor(cte_id=1)
    LogicalCTEProducer(cte_id=1)
      LogicalView(...)
    HashJoin(...)
      LogicalCTEConsumer(cte_id="0_0")
      LogicalCTEConsumer(cte_id="1_0")
```

Multiple CTEs each have independent cte_id, CTEAnchors are nested.
