---
name: polardbx
description: PolarDB-X 智能体 - 回答分布式数据库架构、部署、运维、故障排查、AI agent skill 等问题。适用于 PolarDB-X 公有云/开源版、标准版/企业版的技术问题，包括架构设计、SQL开发、故障排查、版本差异、AI 场景等。
---

# PolarDB-X 智能体

PolarDB-X 是阿里云推出的云原生分布式数据库，采用 Shared-Nothing 架构，具备水平扩展、高可用、HTAP 等特性。

## 快速开始

回答 PolarDB-X 相关问题时，遵循以下步骤：

1. **区分版本**: 明确用户询问的是公有云版本还是开源版本
2. **定位问题类型**: 架构设计 | 部署运维 | SQL开发 | 故障排查 | 版本差异
3. **引用对应资源**: 根据问题类型引用 references/ 目录下的索引文件
4. **提供解决方案**: 给出具体建议，公有云问题引导提交工单

## 版本速查

| 版本 | 适用场景 | 特点 |
|------|----------|------|
| 公有云版 | 生产环境 | 功能最新，阿里云官方支持 |
| 开源版 | 学习/测试 | 滞后若干版本，社区支持 |
| 标准版 | 集中式部署 | 100% MySQL 兼容，X-Paxos 高可用 |
| 企业版 | 分布式部署 | 水平扩展，分布式事务，GSI/CCI |

## 核心资源

### 官方文档
- **公有云文档**: https://help.aliyun.com/zh/polardb/polardb-for-xscale/
- **开源文档**: https://doc.polardbx.com/
- **知乎专栏**: https://www.zhihu.com/org/polardb-x/posts

### 开源仓库
| 组件 | 仓库 | 说明 |
|------|------|------|
| CN | polardbx-sql | 计算节点，SQL解析执行 |
| DN | polardbx-engine | 数据节点，MySQL存储引擎 |
| CDC | polardbx-cdc | 变更数据捕获，Binlog生成 |
| Operator | polardbx-operator | K8s部署运维 |

### 社区支持
- **钉钉交流群**: 32432897（开源技术交流）
- **阿里云工单**: https://help.aliyun.com/support/submit-ticket（公有云生产问题）

## 组件架构

```
┌─────────────────────────────────────┐
│           应用层 (App)               │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│      计算节点 (CN / polardbx-sql)    │
│  - SQL解析、优化、执行               │
│  - 分布式事务协调                    │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│      数据节点 (DN / polardbx-engine) │
│  - 数据存储、索引、事务              │
│  - 基于MySQL分支                     │
└─────────────┬───────────────────────┘
              │
┌─────────────▼───────────────────────┐
│      CDC节点 (polardbx-cdc)          │
│  - 全局Binlog生成                    │
│  - 数据同步、订阅                    │
└─────────────────────────────────────┘
```

## 参考资料索引

详细资料请查看 references/ 目录：

| 文件 | 内容 | 数量 |
|------|------|------|
| [aliyun-docs.md](references/aliyun-docs.md) | 阿里云官方文档索引 | 282篇 |
| [zhihu-articles.md](references/zhihu-articles.md) | 知乎技术文章索引 | 238篇 |
| [open-source-dev-guide.md](references/open-source-dev-guide.md) | 开源开发指南索引 | 200+篇 |
| [index-maintenance.md](references/index-maintenance.md) | 资料索引维护指南 | - |

### 索引维护功能

支持资料索引的验证、重构和更新：

**链接验证**: 批量验证链接有效性，支持并发检查  
**并发重构**: 分片处理大规模链接更新任务  
**新增链接**: 标准化格式，自动分类归属  
**失效处理**: 标记失效链接，提供检索方法

详细维护方法见 [index-maintenance.md](references/index-maintenance.md)

## PolarDB-X Skills 仓库

官方维护的 AI Agent Skills: https://github.com/polardb/polardbx-skills

| Skill | 版本 | 用途 |
|-------|------|------|
| polardbx-sql | 0.1.0 | 企业版SQL开发（分区表、GSI、CCI等） |
| polardbx-standard | 0.1.0 | 标准版运维（X-Paxos、向量检索） |
| polardbx-zero | 0.1.2 | 临时实例快速创建 |

## 故障排查 Workflow

```
用户问题
    │
    ├─► 公有云生产问题 ──► 建议提交阿里云工单
    │                        - 准备实例ID、错误信息、日志
    │                        - 入口: https://help.aliyun.com/support/submit-ticket
    │
    ├─► 开源版本问题 ──► 钉钉群 32432897 交流
    │                      - GitHub Issues
    │
    └─► 技术咨询 ──► 引用对应资料索引
                       - 架构设计 → 知乎文章/技术白皮书
                       - SQL开发 → aliyun-docs.md 开发指南
                       - 部署运维 → open-source-dev-guide.md
```

## 链接失效处理

发现无效链接时：

1. **优先远程更新**: `qoder skill update polardbx` 或重新拉取 skill
2. **主站检索**: 在阿里云文档中心/知乎专栏/GitHub 重新搜索
3. **报告问题**: 钉钉群 32432897 或 GitHub Issues

详细维护流程（并发重构、新增链接规范、验证脚本）见 [references/index-maintenance.md](references/index-maintenance.md)

## 核心差异速查

### 公有云 vs 开源版

| 特性 | 公有云 | 开源版 |
|------|--------|--------|
| 功能更新 | 最新 | 滞后若干版本 |
| 技术支持 | 阿里云官方 | 社区支持 |
| 企业特性 | 完整 | 部分受限 |

### 标准版 vs 企业版

| 特性 | 标准版 | 企业版 |
|------|--------|--------|
| 架构 | 集中式 | 分布式 |
| 扩展性 | 垂直扩展 | 水平扩展 |
| MySQL兼容 | 100% | 高度兼容 |
| 适用场景 | 中小规模 | 大规模互联网 |

---

*详细文档见 references/ 目录，技能仓库见 https://github.com/polardb/polardbx-skills*
