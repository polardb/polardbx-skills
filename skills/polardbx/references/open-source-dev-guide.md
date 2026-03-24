# PolarDB-X 开源版本开发指南索引

> 来源: https://github.com/polardb/polardbx-sql 及相关仓库
> 官方文档: https://doc.polardbx.com/
> 开源官网: https://polardbx.com/
> 更新时间: 2026-03-24
> 文档总数: 200+篇

---

## 一、官方文档站点 (doc.polardbx.com)

### 1.1 关于 PolarDB-X

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 1 | 关于 PolarDB-X (中文) | https://doc.polardbx.com/zh/ | PolarDB-X中文文档首页 | 简介、概述 |
| 2 | About PolarDB-X (English) | https://doc.polardbx.com/en/ | PolarDB-X英文文档首页 | Introduction、Overview |
| 3 | 各版本能力对比 | https://doc.polardbx.com/zh/about/version-info.html | 不同版本功能对比说明 | 版本、对比、功能 |

### 1.2 核心特性

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 4 | 高可用和容灾 | https://doc.polardbx.com/zh/features/topics/x-paxos.html | X-Paxos协议实现的高可用和容灾 | 高可用、容灾、X-Paxos |
| 5 | High Availability and Disaster Recovery | https://doc.polardbx.com/en/features/topics/x-paxos.html | X-Paxos based HA and disaster recovery | HA、Disaster Recovery |
| 6 | 分布式事务 | https://doc.polardbx.com/zh/features/topics/distributed-transaction.html | 分布式事务实现原理 | 分布式事务、TSO、2PC |
| 7 | Distributed Transactions | https://doc.polardbx.com/en/features/topics/distributed-transaction.html | Distributed transaction implementation | Distributed Transaction |
| 8 | 水平扩展 | https://doc.polardbx.com/zh/features/topics/horizontal-scaling.html | 水平扩展能力说明 | 水平扩展、分片 |
| 9 | Distributed Linear Scalability | https://doc.polardbx.com/en/features/topics/horizontal-scaling.html | Horizontal scaling capabilities | Scaling |
| 10 | MySQL 生态兼容 | https://doc.polardbx.com/zh/features/topics/mysql-compatability.html | MySQL兼容性说明 | MySQL、兼容、生态 |
| 11 | Compatibility with MySQL Ecosystem | https://doc.polardbx.com/en/features/topics/mysql-compatability.html | MySQL compatibility | MySQL Compatibility |
| 12 | 全局二级索引 | https://doc.polardbx.com/zh/features/topics/gsi.html | 全局二级索引GSI说明 | GSI、全局索引 |
| 13 | Global Secondary Index | https://doc.polardbx.com/en/features/topics/gsi.html | Global Secondary Index documentation | GSI |
| 14 | 混合负载 HTAP | https://doc.polardbx.com/zh/features/topics/htap.html | HTAP混合负载处理能力 | HTAP、OLTP、OLAP |
| 15 | HTAP | https://doc.polardbx.com/en/features/topics/htap.html | HTAP capabilities | HTAP |
| 16 | 全局变更日志 CDC | https://doc.polardbx.com/zh/features/topics/global-binlog.html | CDC全局Binlog功能 | CDC、Binlog、变更日志 |
| 17 | Capture Data Change(CDC) | https://doc.polardbx.com/en/features/topics/global-binlog.html | CDC global binlog feature | CDC、Binlog |

### 1.3 快速入门

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 18 | 快速体验 | https://doc.polardbx.com/zh/quickstart/topics/quickstart.html | 快速体验PolarDB-X | 快速开始、体验 |
| 19 | PXD Quick Start | https://doc.polardbx.com/en/quickstart/topics/quickstart.html | Quick start with PXD | Quick Start |
| 20 | 通过 PXD 部署集群 | https://doc.polardbx.com/zh/quickstart/topics/quickstart-pxd-cluster.html | 使用PXD部署集群 | PXD、部署、集群 |
| 21 | Deploy via PXD | https://doc.polardbx.com/en/quickstart/topics/quickstart-pxd-cluster.html | Deploy cluster via PXD | PXD Deploy |
| 22 | 通过 K8S 部署 | https://doc.polardbx.com/zh/quickstart/topics/quickstart-k8s.html | Kubernetes部署指南 | K8S、Kubernetes、部署 |
| 23 | Deploy via K8S | https://doc.polardbx.com/en/quickstart/topics/quickstart-k8s.html | Deploy via Kubernetes | K8S Deploy |
| 24 | 源码编译安装 | https://doc.polardbx.com/zh/quickstart/topics/quickstart-development.html | 从源码编译安装 | 源码、编译、开发 |
| 25 | Build From Source Code | https://doc.polardbx.com/en/quickstart/topics/quickstart-development.html | Build from source | Source Build |

### 1.4 部署标准集群

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 26 | 部署流程 | https://doc.polardbx.com/zh/deployment/topics/deploy-workflow.html | 生产环境部署流程 | 部署、流程 |
| 27 | Deploy Process | https://doc.polardbx.com/en/deployment/topics/deploy-workflow.html | Production deployment process | Deploy Process |
| 28 | 软硬件配置建议 | https://doc.polardbx.com/zh/deployment/topics/environment-requirement.html | 硬件和软件配置要求 | 配置、要求、环境 |
| 29 | Hardware and Software Configuration | https://doc.polardbx.com/en/deployment/topics/environment-requirement.html | HW/SW configuration requirements | Configuration |
| 30 | 系统与环境配置 | https://doc.polardbx.com/zh/deployment/topics/system-configuration.html | 操作系统和环境配置 | 系统配置、环境 |
| 31 | System Configuration | https://doc.polardbx.com/en/deployment/topics/system-configuration.html | System and environment config | System Config |
| 32 | 软件包下载 | https://doc.polardbx.com/zh/deployment/topics/software-download.html | 软件包下载指南 | 下载、软件包 |
| 33 | Download Software Package | https://doc.polardbx.com/en/deployment/topics/software-download.html | Software package download | Download |
| 34 | 安装 Docker 与镜像仓库 | https://doc.polardbx.com/zh/deployment/topics/install-docker.html | Docker和镜像仓库安装 | Docker、镜像仓库 |
| 35 | Installing Docker and Image Repository | https://doc.polardbx.com/en/deployment/topics/install-docker.html | Docker and registry installation | Docker |
| 36 | 安装 Kubernetes | https://doc.polardbx.com/zh/deployment/topics/install-kubernetes.html | Kubernetes安装指南 | K8S、Kubernetes |
| 37 | Installing Kubernetes | https://doc.polardbx.com/en/deployment/topics/install-kubernetes.html | Kubernetes installation | Kubernetes |
| 38 | 数据库部署(分布式) | https://doc.polardbx.com/zh/deployment/topics/deploy-polardbx.html | 分布式版本部署 | 分布式、部署 |
| 39 | Database Deployment (Distributed) | https://doc.polardbx.com/en/deployment/topics/deploy-polardbx.html | Distributed deployment | Distributed Deploy |
| 40 | 通过 PXD 部署 | https://doc.polardbx.com/zh/deployment/topics/deploy-by-pxd.html | 使用PXD部署 | PXD、部署 |
| 41 | Deploying with PXD | https://doc.polardbx.com/en/deployment/topics/deploy-by-pxd.html | Deploy with PXD | PXD |
| 42 | 通过 Kubernetes 部署 | https://doc.polardbx.com/zh/deployment/topics/deploy-by-k8s.html | 使用K8S部署 | K8S、部署 |
| 43 | Deploying with Kubernetes | https://doc.polardbx.com/en/deployment/topics/deploy-by-k8s.html | Deploy with Kubernetes | Kubernetes |
| 44 | 数据库部署(集中式) | https://doc.polardbx.com/zh/deployment/topics/deploy-polardbx-std.html | 集中式版本部署 | 集中式、部署 |
| 45 | Database Deployment (Centralized) | https://doc.polardbx.com/en/deployment/topics/deploy-polardbx-std.html | Centralized deployment | Centralized |
| 46 | 通过 RPM 部署 | https://doc.polardbx.com/zh/deployment/topics/deploy-by-rpm-std.html | 使用RPM包部署 | RPM、部署 |
| 47 | Deploying with RPM | https://doc.polardbx.com/en/deployment/topics/deploy-by-rpm-std.html | Deploy with RPM | RPM |
| 48 | 私有镜像仓库说明 | https://doc.polardbx.com/zh/deployment/topics/private-docker-repo.html | 私有镜像仓库配置 | 镜像仓库、私有 |
| 49 | Private Image Repository | https://doc.polardbx.com/en/deployment/topics/private-docker-repo.html | Private registry configuration | Private Registry |

### 1.5 性能白皮书

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 50 | Sysbench 测试报告(集中式) | https://doc.polardbx.com/zh/performance/centralized/sysbench-performance.html | 集中式版本Sysbench性能报告 | Sysbench、性能、集中式 |
| 51 | Sysbench Test Report (Centralized) | https://doc.polardbx.com/en/performance/centralized/sysbench-performance.html | Centralized Sysbench report | Sysbench |
| 52 | TPC-C 测试报告(集中式) | https://doc.polardbx.com/zh/performance/centralized/tpcc-performance.html | 集中式版本TPC-C性能报告 | TPC-C、性能、集中式 |
| 53 | TPC-C Test Report (Centralized) | https://doc.polardbx.com/en/performance/centralized/tpcc-performance.html | Centralized TPC-C report | TPC-C |
| 54 | Sysbench 测试报告(分布式) | https://doc.polardbx.com/zh/performance/distributed/sysbench-performance.html | 分布式版本Sysbench性能报告 | Sysbench、性能、分布式 |
| 55 | Sysbench Test Report (Distributed) | https://doc.polardbx.com/en/performance/distributed/sysbench-performance.html | Distributed Sysbench report | Sysbench |
| 56 | TPC-C 测试报告(分布式) | https://doc.polardbx.com/zh/performance/distributed/tpcc-performance.html | 分布式版本TPC-C性能报告 | TPC-C、性能、分布式 |
| 57 | TPC-C Test Report (Distributed) | https://doc.polardbx.com/en/performance/distributed/tpcc-performance.html | Distributed TPC-C report | TPC-C |
| 58 | TPC-H 测试报告(分布式) | https://doc.polardbx.com/zh/performance/distributed/tpch-performance.html | 分布式版本TPC-H性能报告 | TPC-H、性能、分布式 |
| 59 | TPC-H Test Report (Distributed) | https://doc.polardbx.com/en/performance/distributed/tpch-performance.html | Distributed TPC-H report | TPC-H |

### 1.6 开发指南

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 60 | 连接 PolarDB-X | https://doc.polardbx.com/zh/dev-guide/topics/connect-to-polardb-x.html | 连接数据库指南 | 连接、客户端 |
| 61 | Connect PolarDB-X | https://doc.polardbx.com/en/dev-guide/topics/connect-to-polardb-x.html | Connect to PolarDB-X | Connection |
| 62 | SQL概述 | https://doc.polardbx.com/zh/dev-guide/topics/sql-overview.html | SQL语法概述 | SQL、语法 |
| 63 | SQL syntax overview | https://doc.polardbx.com/en/dev-guide/topics/sql-overview.html | SQL syntax overview | SQL |
| 64 | 开发限制 | https://doc.polardbx.com/zh/dev-guide/topics/limitation.html | 开发使用限制 | 限制、约束 |
| 65 | Limits on database development | https://doc.polardbx.com/en/dev-guide/topics/limitation.html | Development limitations | Limitations |
| 66 | 数据类型 | https://doc.polardbx.com/zh/dev-guide/topics/data-type.html | 支持的数据类型 | 数据类型 |
| 67 | Data types | https://doc.polardbx.com/en/dev-guide/topics/data-type.html | Supported data types | Data Types |
| 68 | 函数 | https://doc.polardbx.com/zh/dev-guide/topics/functions.html | 支持的函数列表 | 函数 |
| 69 | Functions | https://doc.polardbx.com/en/dev-guide/topics/functions.html | Supported functions | Functions |
| 70 | DDL语句 | https://doc.polardbx.com/zh/dev-guide/topics/ddl.html | DDL语法说明 | DDL |
| 71 | DDL statements | https://doc.polardbx.com/en/dev-guide/topics/ddl.html | DDL syntax | DDL |
| 72 | DML语句 | https://doc.polardbx.com/zh/dev-guide/topics/dml.html | DML语法说明 | DML |
| 73 | DML statements | https://doc.polardbx.com/en/dev-guide/topics/dml.html | DML syntax | DML |
| 74 | DQL语句 | https://doc.polardbx.com/zh/dev-guide/topics/dql.html | DQL查询语法 | DQL、查询 |
| 75 | DQL statements | https://doc.polardbx.com/en/dev-guide/topics/dql.html | DQL syntax | DQL |
| 76 | DAL语句 | https://doc.polardbx.com/zh/dev-guide/topics/dal.html | DAL管理语句 | DAL |
| 77 | Data Access Language (DAL) | https://doc.polardbx.com/en/dev-guide/topics/dal.html | DAL statements | DAL |
| 78 | TCL语句 | https://doc.polardbx.com/zh/dev-guide/topics/tcl.html | 事务控制语句 | TCL、事务 |
| 79 | TCL statements | https://doc.polardbx.com/en/dev-guide/topics/tcl.html | Transaction control | TCL |
| 80 | 权限管理 | https://doc.polardbx.com/zh/dev-guide/topics/permissions.html | 权限管理说明 | 权限、安全 |
| 81 | Permission management | https://doc.polardbx.com/en/dev-guide/topics/permissions.html | Permission management | Permissions |
| 82 | Sequence | https://doc.polardbx.com/zh/dev-guide/topics/sequence.html | Sequence使用指南 | Sequence、序列 |
| 83 | Sequence | https://doc.polardbx.com/en/dev-guide/topics/sequence.html | Sequence usage | Sequence |
| 84 | 透明分布式 | https://doc.polardbx.com/zh/dev-guide/topics/transparent-shard.html | 透明分布式模式 | 透明分布式、分片 |
| 85 | Transparent Sharding Mode | https://doc.polardbx.com/en/dev-guide/topics/transparent-shard.html | Transparent sharding | Transparent Sharding |
| 86 | 常见问题 | https://doc.polardbx.com/zh/dev-guide/topics/faq.html | 开发常见问题FAQ | FAQ、问题 |
| 87 | FAQ | https://doc.polardbx.com/en/dev-guide/topics/faq.html | Development FAQ | FAQ |
| 88 | 错误码 | https://doc.polardbx.com/zh/dev-guide/topics/error-code.html | 错误码参考 | 错误码 |
| 89 | Error Codes | https://doc.polardbx.com/en/dev-guide/topics/error-code.html | Error code reference | Error Codes |

### 1.7 Operator 运维指南

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 90 | 安装部署 | https://doc.polardbx.com/zh/operator/deployment/1-installation.html | Operator安装指南 | Operator、安装 |
| 91 | Installation | https://doc.polardbx.com/en/operator/deployment/1-installation.html | Operator installation | Installation |
| 92 | 卸载 | https://doc.polardbx.com/zh/operator/deployment/2-uninstallation.html | Operator卸载指南 | 卸载 |
| 93 | Uninstallation | https://doc.polardbx.com/en/operator/deployment/2-uninstallation.html | Operator uninstallation | Uninstallation |
| 94 | 升级 | https://doc.polardbx.com/zh/operator/deployment/3-upgrade.html | Operator升级指南 | 升级 |
| 95 | Upgrade | https://doc.polardbx.com/en/operator/deployment/3-upgrade.html | Operator upgrade | Upgrade |
| 96 | PolarDBXCluster CRD | https://doc.polardbx.com/zh/operator/api/polardbxcluster.html | PolarDBXCluster CRD API | CRD、API |
| 97 | PolarDBXCluster | https://doc.polardbx.com/en/operator/api/polardbxcluster.html | PolarDBXCluster CRD API | CRD API |
| 98 | XStore CRD | https://doc.polardbx.com/zh/operator/api/xstore.html | XStore CRD API | XStore、CRD |
| 99 | XStore | https://doc.polardbx.com/en/operator/api/xstore.html | XStore CRD API | XStore |
| 100 | 创建企业版集群 | https://doc.polardbx.com/zh/operator/ops/lifecycle/1-create.html | 创建企业版集群 | 企业版、创建 |
| 101 | Create Enterprise Edition Cluster | https://doc.polardbx.com/en/operator/ops/lifecycle/1-create.html | Create enterprise cluster | Enterprise |
| 102 | 容灾部署示例 | https://doc.polardbx.com/zh/operator/ops/lifecycle/1-create-ha-example.html | 容灾部署配置示例 | 容灾、HA |
| 103 | Disaster Recovery Deployment Example | https://doc.polardbx.com/en/operator/ops/lifecycle/1-create-ha-example.html | Disaster recovery example | Disaster Recovery |
| 104 | 只读实例创建 | https://doc.polardbx.com/zh/operator/ops/lifecycle/1-create-readonly-pxc.html | 创建只读实例 | 只读、实例 |
| 105 | 集群升降配 | https://doc.polardbx.com/zh/operator/ops/lifecycle/2-scale.html | 集群扩缩容 | 扩缩容、升降配 |
| 106 | 集群升级 | https://doc.polardbx.com/zh/operator/ops/lifecycle/3-upgrade.html | 集群版本升级 | 升级 |
| 107 | 集群删除 | https://doc.polardbx.com/zh/operator/ops/lifecycle/4-delete.html | 删除集群 | 删除 |
| 108 | 备份恢复 | https://doc.polardbx.com/zh/operator/ops/backup/1-backup.html | 备份操作指南 | 备份 |
| 109 | 恢复数据 | https://doc.polardbx.com/zh/operator/ops/backup/2-restore.html | 恢复操作指南 | 恢复 |
| 110 | 配置变更 | https://doc.polardbx.com/zh/operator/ops/config/1-parameter-change.html | 参数配置变更 | 参数、配置 |
| 111 | 监控告警 | https://doc.polardbx.com/zh/operator/ops/monitor/1-monitor-and-alert.html | 监控告警配置 | 监控、告警 |
| 112 | 日志管理 | https://doc.polardbx.com/zh/operator/ops/log/1-log-collection.html | 日志收集管理 | 日志 |
| 113 | 问题诊断 | https://doc.polardbx.com/zh/operator/ops/troubleshoot/1-basic-troubleshoot.html | 基础问题诊断 | 故障、诊断 |

---

## 二、GitHub 仓库文档

### 2.1 polardbx-sql (CN - 计算节点)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 114 | PXD Quick Start (English) | https://github.com/polardb/polardbx-sql/blob/main/docs/en/quickstart.md | 使用PXD工具快速部署PolarDB-X英文指南 | PXD、快速开始、Docker |
| 115 | Development Guide (English) | https://github.com/polardb/polardbx-sql/blob/main/docs/en/quickstart-development.md | 从源码编译安装和开发PolarDB-X英文指南 | 开发、源码编译、贡献 |
| 116 | 快速开始 (中文) | https://github.com/polardb/polardbx-sql/blob/main/docs/zh_CN/quickstart.md | 使用PXD工具快速部署PolarDB-X中文指南 | PXD、快速开始、中文 |
| 117 | 开发指南 (中文) | https://github.com/polardb/polardbx-sql/blob/main/docs/zh_CN/quickstart-development.md | CentOS/Ubuntu上源码编译PolarDB-X中文指南 | 源码编译、开发、中文 |
| 118 | 如何调试 CN | https://github.com/polardb/polardbx-sql/blob/main/docs/zh_CN/quickstart-how-to-debug-cn.md | MacOS上使用IDEA搭建CN本地开发环境 | 开发环境、调试、IDE |
| 119 | OSS存储部署 | https://github.com/polardb/polardbx-sql/blob/main/docs/zh_CN/quickstart-oss.md | 配置PolarDB-X使用OSS或本地磁盘存储 | OSS、存储、配置、TTL |
| 120 | PXD集群部署 | https://github.com/polardb/polardbx-sql/blob/main/docs/zh_CN/quickstart-pxd-cluster.md | 在Linux集群上使用PXD部署PolarDB-X | 集群部署、PXD、Linux |
| 121 | 架构图 | https://github.com/polardb/polardbx-sql/blob/main/docs/architecture.png | PolarDB-X系统架构图 | 架构图、设计 |
| 122 | 贡献指南 | https://github.com/polardb/polardbx-sql/blob/main/CONTRIBUTING.md | 如何为PolarDB-X项目贡献代码 | 贡献、社区、PR |
| 123 | LICENSE | https://github.com/polardb/polardbx-sql/blob/main/LICENSE | Apache 2.0开源许可证 | 许可证、Apache |

### 2.2 polardbx-engine (DN - 数据节点)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 124 | README | https://github.com/polardb/polardbx-engine/blob/polardbx-8.0.32/README | 存储引擎项目基本介绍 | README、DN、存储引擎 |
| 125 | README.build | https://github.com/polardb/polardbx-engine/blob/polardbx-8.0.32/Docs/README.build | 编译安装指南 | 编译、CMake、构建 |
| 126 | LICENSE | https://github.com/polardb/polardbx-engine/blob/main/LICENSE | GPL v2.0许可证 | 许可证 |
| 127 | MySQL SHOW MASTER STATUS | https://dev.mysql.com/doc/refman/8.0/en/show-master-status.html | MySQL官方SHOW MASTER STATUS命令参考 | MySQL、Binlog |
| 128 | MySQL SHOW BINARY LOGS | https://dev.mysql.com/doc/refman/8.0/en/show-binary-logs.html | MySQL官方SHOW BINARY LOGS命令参考 | MySQL、Binlog |

### 2.3 polardbx-cdc (CDC - 变更数据捕获)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 129 | CDC中文文档 | https://github.com/polardb/polardbx-cdc/blob/main/docs/zh_CN/README.md | CDC组件中文文档入口 | CDC、中文文档 |
| 130 | Binlog命令介绍 | https://github.com/polardb/polardbx-cdc/blob/main/docs/zh_CN/binlog-commands-intro.md | Binlog命令详细介绍 | Binlog、命令、运维 |
| 131 | Replica参考手册 | https://github.com/polardb/polardbx-cdc/tree/main/polardbx-cdc-rpl/README.md | 数据复制组件参考手册 | Replica、复制、RPL |
| 132 | LICENSE | https://github.com/polardb/polardbx-cdc/blob/main/LICENSE | 许可证文件 | 许可证 |
| 133 | CDC节点创建指南 | https://doc.polardbx.com/operator/ops/component/cdc/1-create-cdc-node-example.html | 在K8s上创建CDC节点操作指南 | CDC节点、K8s部署 |

### 2.4 polardbx-tools (工具集)

#### batch-tool (批量工具)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 134 | 使用详情 | https://github.com/polardb/polardbx-tools/blob/main/batch-tool/docs/usage-details.md | 批量工具使用详解 | 批量工具、使用指南 |
| 135 | README | https://github.com/polardb/polardbx-tools/blob/main/batch-tool/README.md | 批量工具介绍 | batch-tool、批量 |
| 136 | LICENSE | https://github.com/polardb/polardbx-tools/blob/main/batch-tool/LICENSE | 许可证 | 许可证 |

#### frodo (流量回放工具)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 137 | README | https://github.com/polardb/polardbx-tools/blob/main/frodo/README.md | Frodo流量回放工具介绍 | frodo、流量回放、压测 |
| 138 | LICENSE | https://github.com/polardb/polardbx-tools/blob/main/frodo/LICENSE | 许可证 | 许可证 |

### 2.5 polardbx-operator (K8s Operator)

#### 主仓库文档

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 139 | README | https://github.com/polardb/polardbx-operator/blob/main/README.md | Operator项目介绍 | Operator、K8S、README |
| 140 | CHANGELOG | https://github.com/polardb/polardbx-operator/blob/main/CHANGELOG.md | 版本变更日志 | 变更日志、版本 |
| 141 | CONTRIBUTING | https://github.com/polardb/polardbx-operator/blob/main/CONTRIBUTING.md | 贡献指南 | 贡献、开发 |
| 142 | LICENSE | https://github.com/polardb/polardbx-operator/blob/main/LICENSE | Apache 2.0许可证 | 许可证 |
| 143 | NOTICE | https://github.com/polardb/polardbx-operator/blob/main/NOTICE.md | 注意事项 | NOTICE |

#### Operator文档仓库

##### 部署指南

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 144 | 快速开始 | https://github.com/polardb/polardbx-operator-docs/blob/main/deployment/0-quickstart.md | K8s上快速部署Operator和集群 | 快速开始、K8s、部署 |
| 145 | 安装指南 | https://github.com/polardb/polardbx-operator-docs/blob/main/deployment/1-installation.md | Operator安装详细指南 | 安装、K8s、Operator |
| 146 | 修改数据目录 | https://github.com/polardb/polardbx-operator-docs/blob/main/deployment/1-installation-data-dir.md | 数据目录配置说明 | 数据目录、配置 |
| 147 | 修改默认镜像 | https://github.com/polardb/polardbx-operator-docs/blob/main/deployment/1-installation-default-image.md | 默认镜像配置说明 | 镜像、配置 |
| 148 | 修改默认镜像仓库 | https://github.com/polardb/polardbx-operator-docs/blob/main/deployment/1-installation-default-image-repo.md | 镜像仓库配置 | 镜像仓库、配置 |
| 149 | 卸载 | https://github.com/polardb/polardbx-operator-docs/blob/main/deployment/2-uninstallation.md | 卸载指南 | 卸载 |
| 150 | 升级 | https://github.com/polardb/polardbx-operator-docs/blob/main/deployment/3-upgrade.md | 升级指南 | 升级 |

##### API文档

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 151 | PolarDBXCluster CRD | https://github.com/polardb/polardbx-operator-docs/blob/main/api/polardbxcluster.md | PolarDBXCluster CRD API | CRD、API、Cluster |
| 152 | XStore CRD | https://github.com/polardb/polardbx-operator-docs/blob/main/api/xstore.md | XStore CRD API | CRD、API、XStore |

##### 运维指南

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 153 | 运维总览 | https://github.com/polardb/polardbx-operator-docs/blob/main/ops/README.md | 运维指南首页 | 运维、管理 |
| 154 | 备份恢复 | https://github.com/polardb/polardbx-operator-docs/tree/main/ops/backup-restore | 备份恢复操作指南 | 备份、恢复 |
| 155 | 组件管理 | https://github.com/polardb/polardbx-operator-docs/tree/main/ops/component | 组件运维指南 | 组件、运维 |
| 156 | 配置管理 | https://github.com/polardb/polardbx-operator-docs/tree/main/ops/configuration | 配置管理指南 | 配置、管理 |
| 157 | 连接管理 | https://github.com/polardb/polardbx-operator-docs/tree/main/ops/connection | 连接管理指南 | 连接 |
| 158 | 生命周期 | https://github.com/polardb/polardbx-operator-docs/tree/main/ops/lifecycle | 生命周期管理 | 生命周期 |
| 159 | 日志收集 | https://github.com/polardb/polardbx-operator-docs/tree/main/ops/logcollector | 日志收集配置 | 日志、收集 |
| 160 | 监控 | https://github.com/polardb/polardbx-operator-docs/tree/main/ops/monitor | 监控配置指南 | 监控 |
| 161 | 重建 | https://github.com/polardb/polardbx-operator-docs/tree/main/ops/rebuild | 集群重建指南 | 重建 |

##### FAQ

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 162 | FAQ总览 | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/README.md | 常见问题解答 | FAQ、问题 |
| 163 | 日志问题 | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/1-log.md | 日志收集常见问题 | 日志、FAQ |
| 164 | 创建阻塞 | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/2-block-in-creating.md | 创建集群卡住问题排查 | 创建、阻塞、FAQ |
| 165 | 删除阻塞 | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/3-block-in-deleting.md | 删除集群卡住问题排查 | 删除、阻塞、FAQ |
| 166 | DN无Leader | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/4-dn-no-leader.md | DN无Leader问题排查 | DN、Leader、FAQ |
| 167 | Pod重启 | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/5-pod-restart-inccident.md | Pod重启问题排查 | Pod、重启、FAQ |
| 168 | Pod状态异常 | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/6-pod-not-in-running-state.md | Pod未运行问题排查 | Pod、状态、FAQ |
| 169 | CN内存Dump | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/7-cn-memory-dump.md | CN内存Dump操作指南 | CN、内存、Dump |
| 170 | DN Core文件 | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/8-dn-core-file.md | DN Core文件获取指南 | DN、Core、FAQ |
| 171 | CN火焰图 | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/9-cn-flame-graph.md | CN性能分析火焰图 | CN、火焰图、性能 |
| 172 | DN火焰图 | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/10-dn-flame-graph.md | DN性能分析火焰图 | DN、火焰图、性能 |
| 173 | ImagePullBackOff | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/11-block-in-imagepullbackoff.md | 镜像拉取失败问题排查 | 镜像、ImagePullBackOff |
| 174 | 进程查杀 | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/12-kill-process-in-pod.md | Pod内进程终止操作 | 进程、Pod、FAQ |
| 175 | 终止Pod日志 | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/13-get-logs-from-a-terminated-pod.md | 获取终止Pod日志 | 日志、终止Pod |
| 176 | Docker镜像构建 | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/14-docker-image-build.md | Docker镜像构建指南 | Docker、镜像、构建 |
| 177 | 私有RPC开关 | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/15-private-rpc-on-off.md | 私有RPC配置 | RPC、配置、FAQ |
| 178 | 事务策略 | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/16-transaction-strategy.md | 事务策略配置 | 事务、策略、FAQ |
| 179 | 单副本集群 | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/17-one-replica-cluster.md | 单副本集群配置 | 单副本、集群、FAQ |
| 180 | 主机网络冲突 | https://github.com/polardb/polardbx-operator-docs/blob/main/faq/18-host-network-port-conflict.md | 主机网络端口冲突解决 | 网络、端口、冲突 |

### 2.6 polardbx-glue (RPC通信组件)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 181 | 中文文档 | https://github.com/polardb/polardbx-glue/blob/main/docs/zh_CN/README.md | Glue组件中文文档 | 中文文档、Glue、RPC |
| 182 | LICENSE | https://github.com/polardb/polardbx-glue/blob/main/LICENSE | Apache 2.0许可证 | 许可证 |

### 2.7 polardbx (主仓库)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 183 | README | https://github.com/polardb/polardbx/blob/main/README.md | 主仓库介绍 | README、主仓库 |
| 184 | Windows编译指南 | https://github.com/polardb/polardbx/blob/main/compile_and_run_polardbx_on_windows.md | Windows下编译运行指南 | Windows、编译 |
| 185 | LICENSE | https://github.com/polardb/polardbx/blob/main/LICENSE | Apache 2.0许可证 | 许可证 |

---

## 三、社区与交流资源

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 186 | PolarDB-X开源官网 | https://polardbx.com/ | PolarDB-X开源社区官方网站 | 官网、开源社区 |
| 187 | PolarDB-X文档中心 | https://polardbx.com/document | 官方文档入口 | 文档中心、入口 |
| 188 | PolarDB开源社区 | https://openpolardb.com | PolarDB开源社区官网 | 开源、社区 |
| 189 | 钉钉交流群 | https://h5.dingtalk.com/circle/healthCheckin.html?dtaction=os&corpId=dingc5456617ca6ab502e1cc01e222598659 | 钉钉技术交流群(群号: 32432897) | 钉钉、交流、社区 |
| 190 | 知乎专栏 | https://www.zhihu.com/org/polardb-x | PolarDB-X知乎技术专栏 | 知乎、技术文章 |
| 191 | 开源贡献榜 | https://opensource.alibaba.com/contribution_leaderboard/details?projectValue=polardb-x | 开源贡献者排行榜 | 贡献榜、开源 |

---

## 四、相关组件仓库

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 192 | polardbx-sql | https://github.com/polardb/polardbx-sql | SQL计算节点(CN) | SQL、CN、计算节点 |
| 193 | polardbx-engine | https://github.com/polardb/polardbx-engine | 存储引擎(DN) | Engine、DN、存储 |
| 194 | polardbx-cdc | https://github.com/polardb/polardbx-cdc | CDC日志节点 | CDC、Binlog、日志 |
| 195 | polardbx-operator | https://github.com/polardb/polardbx-operator | K8S Operator | Operator、K8S |
| 196 | polardbx-proxy | https://github.com/polardb/polardbx-proxy | 代理组件 | Proxy、代理 |
| 197 | polardbx-glue | https://github.com/polardb/polardbx-glue | RPC通信组件 | Glue、RPC、通信 |
| 198 | polardbx-tools | https://github.com/polardb/polardbx-tools | 工具集 | Tools、工具 |
| 199 | polardbx-connector-go | https://github.com/polardb/polardbx-connector-go | Go语言驱动 | Go、驱动、Connector |
| 200 | polardbx-connector-cpp | https://github.com/polardb/polardbx-connector-cpp | C++语言驱动 | C++、驱动 |

---

## 使用说明

### 如何查找文档

1. **按关键词搜索**: 使用表格中的关键词列快速定位相关文档
2. **按类别浏览**: 文档按官方文档站点、GitHub仓库、社区资源等分类
3. **按组件定位**: 根据CN、DN、CDC、Operator等组件选择对应文档
4. **按语言筛选**: 中英文文档分别标注，可根据需要选择

### 文档分类速查

| 需求类型 | 推荐类别 |
|----------|----------|
| 了解产品 | 一、1.1 关于PolarDB-X |
| 快速体验 | 一、1.3 快速入门 |
| 生产部署 | 一、1.4 部署标准集群 |
| K8S部署 | 一、1.7 Operator运维指南 |
| 源码开发 | 二、2.1 polardbx-sql |
| CDC开发 | 二、2.3 polardbx-cdc |
| 运维管理 | 二、2.5 polardbx-operator |
| 问题排查 | 二、2.5 FAQ |
| 社区交流 | 三、社区与交流资源 |

### 链接说明

- 所有链接均为官方文档站点或GitHub仓库完整URL
- 官方文档站点: https://doc.polardbx.com/
- 开源官网: https://polardbx.com/
- GitHub组织: https://github.com/polardb/
- 建议收藏官方文档中心获取最新更新

---

*注：本文档收录全部200+篇开源开发指南文档，涵盖PolarDB-X CN、DN、CDC、Operator等全部组件，持续更新中*
