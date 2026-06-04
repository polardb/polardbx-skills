# Plan Visualization — Graphviz Reference

**Do not draw by default** — only execute when the user explicitly requests it.

---

## Workflow

1. Parse operator nodes and parent-child relationships from EXPLAIN output
2. Generate DOT code following the color specification
3. Render: `dot -Tpng <file>.dot -o <file>.png -Gdpi=150`

---

## Color Specification

| Category | fillcolor | Operators |
|----------|-----------|-----------|
| Sort | `#F0F0F0` | TopN, MemSort, MergeSort, Limit |
| Join | `#FFF9C4` | HashJoin, NLJoin, BKAJoin, SortMergeJoin, SemiHashJoin |
| Agg | `#E1BEE7` | HashAgg, SortAgg, PartialHashAgg |
| Exchange | `#FFCDD2` | Exchange, Gather, broadcast |
| TableScan | `#C8E6C9` | LogicalView, IndexScan, OSSTableScan |
| Filter/Project | `#E3F2FD` | Filter, Project |
| Window | `#FFF3E0` | Window, HashWindow, SortWindow |
| CTE | `#F3E5F5` | CTEAnchor, CTEProducer, CTEConsumer |
| SetOp | `#E0F7FA` | UnionAll, UnionDistinct |
| Other | `#FAFAFA` | Expand, Values, RuntimeFilterBuilder |

---

## Node Label Rules

- **Operator nodes**: Join/Agg/Sort etc. display operator type (e.g., `join`, `agg`, `TopN`)
- **TableScan nodes**: Display table name (e.g., `orders`, `lineitem`)
- **Exchange nodes**: Display distribution type (e.g., `broadcast`, `exchange`)
- **Row count annotations**: Independent plaintext nodes placed beside the operator
  - >= 100M: `XM` (e.g., `120M`)
  - >= 10K: `XK` (e.g., `3.2M`)
  - < 10K: Raw value
- **Annotation positioning**: Use invisible edges + `{rank=same}` to align annotations beside operators

---

## DOT Template

```dot
digraph plan {
    label="<title>"; labelloc=t; labeljust=l;
    fontname="Helvetica"; fontsize=14;
    rankdir=TB; nodesep=0.5; ranksep=0.55; bgcolor="#F8F8F8";
    node [fontname="Helvetica", fontsize=10, style="filled,rounded", shape=box, penwidth=1.0, margin="0.12,0.06"];
    edge [arrowsize=0.55, color="#888888"];

    // Operator nodes
    topn [label="TopN", fillcolor="#F0F0F0"];
    hj   [label="join", fillcolor="#FFF9C4"];
    agg  [label="agg", fillcolor="#E1BEE7"];
    bc   [label="broadcast", fillcolor="#FFCDD2"];
    scan [label="orders", fillcolor="#C8E6C9"];

    // Row count annotations
    node [shape=plaintext, style="", fillcolor=none, fontsize=8, fontcolor="#888888", margin="0,0"];
    r_agg  [label="110M"];
    r_scan [label="3.2M"];

    // Edges
    topn -> hj -> agg -> scan;
    hj -> bc;

    // Annotation positioning
    agg  -> r_agg  [style=invis, minlen=0];
    scan -> r_scan [style=invis, minlen=0];
    {rank=same; agg; r_agg}
    {rank=same; scan; r_scan}
}
```

---

## Full Example (TPC-H Q2)

```dot
digraph Q2 {
    label="Q2"; labelloc=t; labeljust=l;
    fontname="Helvetica"; fontsize=14;
    rankdir=TB; nodesep=0.5; ranksep=0.55; bgcolor="#F8F8F8";
    node [fontname="Helvetica", fontsize=10, style="filled,rounded", shape=box, penwidth=1.0, margin="0.12,0.06"];
    edge [arrowsize=0.55, color="#888888"];

    // Operator nodes
    topn    [label="TopN", fillcolor="#F0F0F0"];
    j_top   [label="join", fillcolor="#FFF9C4"];
    agg     [label="agg", fillcolor="#E1BEE7"];
    j_L1    [label="join", fillcolor="#FFF9C4"];
    ps_L    [label="partsupp", fillcolor="#C8E6C9"];
    bc_L1   [label="broadcast", fillcolor="#FFCDD2"];
    j_L2    [label="join", fillcolor="#FFF9C4"];
    sup_L   [label="supplier", fillcolor="#C8E6C9"];
    bc_L2   [label="broadcast", fillcolor="#FFCDD2"];
    j_L3    [label="join", fillcolor="#FFF9C4"];
    nat_L   [label="nation", fillcolor="#C8E6C9"];
    bc_L3   [label="broadcast", fillcolor="#FFCDD2"];
    reg_L   [label="region", fillcolor="#C8E6C9"];
    exc_R   [label="exchange", fillcolor="#FFCDD2"];
    j_R1    [label="join", fillcolor="#FFF9C4"];
    j_R2    [label="join", fillcolor="#FFF9C4"];
    sup_R   [label="supplier", fillcolor="#C8E6C9"];
    exc_R2  [label="exchange", fillcolor="#FFCDD2"];
    j_R3    [label="join", fillcolor="#FFF9C4"];
    ps_R    [label="partsupp", fillcolor="#C8E6C9"];
    part_R  [label="part", fillcolor="#C8E6C9"];
    bc_R1   [label="broadcast", fillcolor="#FFCDD2"];
    j_R4    [label="join", fillcolor="#FFF9C4"];
    nat_R   [label="nation", fillcolor="#C8E6C9"];
    bc_R2   [label="broadcast", fillcolor="#FFCDD2"];
    reg_R   [label="region", fillcolor="#C8E6C9"];

    // Row count annotations
    node [shape=plaintext, style="", fillcolor=none, fontsize=8, fontcolor="#888888", margin="0,0"];
    r_agg   [label="110M"];
    r_jL1   [label="160M"];
    r_psL   [label="120M\n620M"];
    r_bcL1  [label="1.2M"];
    r_supL  [label="10M"];
    r_excR  [label="640K"];
    r_jR1   [label="3.2M"];
    r_supR  [label="2.38M\n59.5M"];
    r_jR3   [label="3.2M"];
    r_psR   [label="23M\n770M"];
    r_partR [label="800K"];

    // Edges
    topn -> j_top;
    j_top -> agg;    j_top -> exc_R;
    agg -> j_L1;
    j_L1 -> ps_L;   j_L1 -> bc_L1;
    bc_L1 -> j_L2;
    j_L2 -> sup_L;  j_L2 -> bc_L2;
    bc_L2 -> j_L3;
    j_L3 -> nat_L;  j_L3 -> bc_L3;
    bc_L3 -> reg_L;
    exc_R -> j_R1;
    j_R1 -> j_R2;   j_R1 -> bc_R1;
    j_R2 -> sup_R;  j_R2 -> exc_R2;
    exc_R2 -> j_R3;
    j_R3 -> ps_R;   j_R3 -> part_R;
    bc_R1 -> j_R4;
    j_R4 -> nat_R;  j_R4 -> bc_R2;
    bc_R2 -> reg_R;

    // Annotation positioning
    {rank=same; agg; r_agg; exc_R; r_excR}
    {rank=same; j_L1; r_jL1; j_R1; r_jR1}
    agg    -> r_agg   [style=invis, minlen=0];
    j_L1   -> r_jL1   [style=invis, minlen=0];
    ps_L   -> r_psL   [style=invis, minlen=0];
    bc_L1  -> r_bcL1  [style=invis, minlen=0];
    sup_L  -> r_supL  [style=invis, minlen=0];
    exc_R  -> r_excR  [style=invis, minlen=0];
    j_R1   -> r_jR1   [style=invis, minlen=0];
    sup_R  -> r_supR  [style=invis, minlen=0];
    j_R3   -> r_jR3   [style=invis, minlen=0];
    ps_R   -> r_psR   [style=invis, minlen=0];
    part_R -> r_partR [style=invis, minlen=0];
}
```
