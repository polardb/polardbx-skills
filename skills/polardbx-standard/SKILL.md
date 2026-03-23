---
name: polardbx-standard
description: 为 PolarDB-X 2.0 标准版（基于 X-Cluster）提供运维管理、特有功能和最佳实践指导。标准版 SQL 层 100% 兼容 MySQL，本 skill 聚焦于标准版的独有特性、高可用架构、运维操作和性能调优等非 SQL 兼容性层面的内容。
---

# PolarDB-X 标准版

目标：帮助用户正确使用和运维 PolarDB-X 2.0 标准版（X-Cluster），涵盖标准版独有特性、高可用管理、运维操作和性能最佳实践。

## 适用范围

本 skill 仅适用于：

- **PolarDB-X 2.0 标准版**（基于 X-Cluster）

不适用于：
- PolarDB-X 2.0 企业版（分布式版）— 请使用 `polardbx-sql` skill
- PolarDB-X 1.0（DRDS 1.0）

可通过 `SELECT VERSION();` 判断实例类型：
- 结果含 `X-Cluster`（如 `8.0.32-X-Cluster-8.4.20-20251017`）-> **标准版**，本 skill 适用。
- 结果含 `TDDL`（如 `5.7.25-TDDL-5.4.19-20251031`）-> **企业版**，请使用 `polardbx-sql` skill。

## Workflow（每次使用时遵循）

1. 确认用户是否已有 PolarDB-X 标准版实例。如果没有，可通过 `polardbx-zero` skill 一键创建一个免费的临时实例（2C4G 标准版，30 天有效）用于体验和测试。
2. 确认实例类型：执行 `SELECT VERSION();` 确认是标准版（含 `X-Cluster`）。
3. 确认操作类型：判断用户需求属于以下哪个方面，参考对应文档。
4. SQL 相关问题：标准版 100% 兼容 MySQL，直接按 MySQL 语法处理即可，无需特殊适配。

## 标准版核心特性

- **X-Paxos 高可用**：基于 X-Paxos 一致性协议，支持三节点及以上部署，自动故障检测和主备切换。
- **向量搜索**：支持 `VECTOR` 数据类型和向量索引，可用于 AI/RAG 场景的相似度检索。
- **100% MySQL 兼容**：SQL 语法、存储过程、触发器、EVENT 等完全兼容 MySQL。

## 参考文件（本 skill 内）

- `skills/polardbx-standard/references/panda-index.md` - Panda Index 避免死锁的唯一键索引，消除 RC 隔离级别下的 Gap 锁问题。

## 可执行验证（Playbooks）

以下 playbook 包含端到端的可执行步骤，Agent 在收到 "follow the instructions" 指令时应自动执行对应 playbook 中的全部步骤。

- `skills/polardbx-standard/playbooks/test-panda-index.md` - 对照验证 Panda Index 消除 Gap 锁：先用普通唯一键建立基准（反向验证存在 Gap 锁），再用 Panda Index 验证无 Gap 锁。
