# PolarDB-X Skills

为 AI 代码智能体（Code Agent）提供 [PolarDB-X](https://help.aliyun.com/zh/polardb/polardb-for-xscale/) 相关的 Agent Skills，帮助智能体更好地使用和操作 PolarDB-X。

## 目标

- 为 AI 智能体提供 PolarDB-X 的专业知识，提升智能体在数据库相关任务中的准确性。
- 提供可复用的 Skills，覆盖 SQL 编写、运维管理、应用开发等场景。
- 保持文档精练、实用，包含可直接运行的示例。

## 目录结构

```
skills/
└── polardbx-sql/          # PolarDB-X SQL 编写与兼容性
    ├── SKILL.md
    └── references/
```

后续可扩展更多 Skills（如 polardbx-java、polardbx-python 等）。

## 当前 Skills

- **polardbx-sql** - PolarDB-X SQL 编写与 MySQL 兼容性处理（分区表、GSI、CCI、Sequence、分布式事务、EXPLAIN、TTL 表等）。

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
