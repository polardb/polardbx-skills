# PolarDB-X Skills

为 AI 代码智能体（Code Agent）提供 [PolarDB-X](https://help.aliyun.com/zh/polardb/polardb-for-xscale/) 相关的 Agent Skills，帮助智能体更好地使用和操作 PolarDB-X。

## 目标

- 为 AI 智能体提供 PolarDB-X 的专业知识，提升智能体在数据库相关任务中的准确性。
- 提供可复用的 Skills，覆盖 SQL 编写、运维管理、应用开发等场景。
- 保持文档精练、实用，包含可直接运行的示例。

## 目录结构

```
skills/
├── polardbx-sql/          # PolarDB-X 企业版 SQL 编写与兼容性（分区设计 + GSI 核心）
│   ├── SKILL.md
│   └── references/
├── polardbx-online-ddl/   # PolarDB-X Online DDL 安全变更
│   ├── SKILL.md
│   └── references/
├── polardbx-pagination/   # PolarDB-X 高效分页与大表遍历
│   ├── SKILL.md
│   └── references/
├── polardbx-cci/          # PolarDB-X CCI 列存索引（OLAP/HTAP）
│   ├── SKILL.md
│   └── references/
├── polardbx-ttl20/        # PolarDB-X TTL 2.0 冷数据归档与自动加分区
│   ├── SKILL.md
│   └── references/
├── polardbx-standard/     # PolarDB-X 标准版特性与运维
│   ├── SKILL.md
│   └── references/
├── polardbx-plan-analysis/ # PolarDB-X 执行计划分析与等价性验证
│   ├── SKILL.md
│   └── references/
├── polardbx-zero/         # PolarDB-X Zero 一键创建临时实例
│   └── SKILL.md
└── sql-review/            # SQL Review 索引分析与推荐
    ├── SKILL.md
    └── references/
```

后续可扩展更多 Skills（如 polardbx-java、polardbx-python 等）。

## 当前 Skills

- **polardbx-sql** - PolarDB-X 企业版 SQL 编写与 MySQL 兼容性处理（分区设计、GSI、Sequence、分布式事务、EXPLAIN 诊断等）。
- **polardbx-online-ddl** - PolarDB-X 企业版 Online DDL 安全变更。通过 EXPLAIN ONLINE_DDL 评估锁表风险，支持 OMC 无锁列类型变更、长事务检查、DDL 进度监控。
- **polardbx-pagination** - PolarDB-X 企业版高效分页与大表遍历。推荐 Keyset 分页替代 LIMIT M,N 深翻页，覆盖按分片遍历、索引要求、Java 代码示例。
- **polardbx-cci** - PolarDB-X 企业版 CCI 列存索引（OLAP/HTAP）。创建和使用 Clustered Columnar Index 加速分析查询，涵盖 CCI 分区键选择、CCI vs GSI 对比、CCI + TTL 冷热分离。
- **polardbx-ttl20** - PolarDB-X 企业版 TTL 2.0 冷数据归档与自动加 Range 分区。分析表结构推荐归档策略（行级或分区级），生成生产可用的 TTL SQL；也支持仅自动预建分区（无清理）的场景。
- **polardbx-plan-analysis** - PolarDB-X 执行计划分析与 SQL 等价性验证。解读 EXPLAIN / EXPLAIN COST / EXPLAIN ANALYZE 输出，涵盖 17 类算子解读、代价分析、运行时瓶颈定位、SQL 与计划等价性 14 维度检查、可选 Graphviz 计划可视化。
- **polardbx-standard** - PolarDB-X 标准版独有特性、高可用架构、运维操作和性能最佳实践，涵盖 X-Paxos HA、Lizard 事务、Panda Index、向量索引（VECTOR + HNSW 语义搜索）。
- **polardbx-zero** - 通过 API 一键创建免认证的 PolarDB-X 临时实例（支持标准版和企业版，最长 30 天自动过期），适用于 AI agent 存储、MCP server 后端、临时测试等场景。
- **sql-review** - 扫描代码库中的 SQL 语句，在 PolarDB-X 测试实例上通过 mock 数据 + EXPLAIN 分析索引使用情况，给出索引优化建议。支持全仓库扫描、指定模块、Git 增量扫描三种模式。

## 安装

通过 [skills.sh](https://skills.sh) 安装：

```bash
npx skills add https://github.com/polardb/polardbx-skills
```

或手动安装到本地智能体的 skills 目录：

```bash
# 以 Qoder 为例
ln -s /path/to/polardbx-skills/skills/polardbx-sql ~/.qoder/skills/polardbx-sql
```

## Skill 编写规范

- Skill 目录使用小写字母、数字和连字符命名（如 `polardbx-sql`）。
- 每个 Skill 必须包含 `SKILL.md`，带有 YAML frontmatter（`name` + `description`）。
- 正文保持精练，包含明确的 Workflow 和核心差异速查。
- 详细文档放入 `references/`，可运行的脚本放入 `scripts/`。
- 遵循 [skills.sh](https://skills.sh) 规范。

## 贡献

欢迎贡献。添加新 Skill 时请包含：

- 简短的用途说明
- 明确的适用范围和前提条件
- 分步骤的使用指南
- 已知的限制和注意事项

## 许可证

[Apache License 2.0](LICENSE)
