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
├── polardbx-ops/          # PolarDB-X 实例生命周期与日常运维（Aliyun CLI）
│   ├── SKILL.md
│   ├── references/
│   └── scripts/
└── sql-review/            # SQL Review 索引分析与推荐
    ├── SKILL.md
    └── references/
```

后续可扩展更多 Skills（如 polardbx-java、polardbx-python 等）。

## 当前 Skills

| Skill | 定位 | 适用场景 | 核心能力 |
|---|---|---|---|
| `polardbx-sql` | 企业版 SQL 编写与兼容性 | 分区设计、GSI、Sequence、MySQL 迁移、分布式事务 | 生成和改写 PolarDB-X 企业版 SQL，处理 MySQL 兼容性差异 |
| `polardbx-online-ddl` | Online DDL 安全变更 | DDL 锁表评估、OMC 无锁变更、长事务检查 | 使用 `EXPLAIN ONLINE_DDL` 评估风险，指导安全执行 DDL |
| `polardbx-pagination` | 高效分页与大表遍历 | 深分页、批量导出、大表游标遍历 | 推荐 Keyset 分页，覆盖索引要求和批处理代码示例 |
| `polardbx-cci` | CCI 列存索引（OLAP/HTAP） | 分析查询、宽表聚合、行列混存、列存快照 | 创建和使用 Clustered Columnar Index，处理分区键、排序键和管理命令 |
| `polardbx-ttl20` | TTL 2.0 冷数据归档 | 数据过期、冷热分离、自动加 Range 分区 | 分析表结构并生成 TTL 归档或自动预建分区 SQL |
| `polardbx-plan-analysis` | 执行计划分析与等价性验证 | EXPLAIN 解读、代价分析、运行时瓶颈定位、SQL/计划一致性检查 | 覆盖 17 类算子、14 维等价性检查和可选 Graphviz 可视化 |
| `polardbx-standard` | 标准版特性与运维 | X-Cluster、X-Paxos HA、Lizard、Panda Index、向量检索 | 说明标准版架构、MySQL 兼容行为和独有功能最佳实践 |
| `polardbx-zero` | 免认证临时实例创建 | AI agent 存储、MCP 后端、临时测试、教程演示 | 通过 API 创建支持标准版/企业版的短期 PolarDB-X 实例 |
| `polardbx-ops` | 阿里云实例生命周期与日常运维 | 创建/删除/重启实例、扩缩容、参数、备份、监控日志、账号安全 | 通过 Aliyun CLI 管理云上 PolarDB-X 实例和运维任务 |
| `sql-review` | 代码库 SQL Review | 全仓库扫描、指定模块扫描、Git 增量扫描 | 提取 SQL 并在测试实例上用 mock 数据与 EXPLAIN 分析索引使用情况 |

## 安装

本项目不提供 Qoder Marketplace。克隆仓库后，作为 Qoder Plugin 本地安装：

```bash
git clone https://github.com/polardb/polardbx-skills.git
qodercli plugins validate /path/to/polardbx-skills
qodercli plugins install /path/to/polardbx-skills
```

团队共享时可安装到项目作用域：

```bash
qodercli plugins install /path/to/polardbx-skills --scope project
```

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
