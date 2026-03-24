# PolarDB-X 知乎技术文章索引

> 来源: https://www.zhihu.com/org/polardb-x/posts
> 更新时间: 2026-03-23
> 文章总数: 238篇

---

## 列存索引与HTAP (12篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 1 | PolarDB-X 列存索引 \| 列存快照原理 | https://zhuanlan.zhihu.com/p/2014641668012409767 | 介绍PolarDB-X列存索引的快照机制原理 | 列存索引、快照、HTAP |
| 2 | PolarDB-X 列存索引 \| 如何创建列存索引 | https://zhuanlan.zhihu.com/p/1973759434250532583 | 详细说明如何在PolarDB-X中创建和使用列存索引 | 列存索引、创建、使用指南 |
| 3 | PolarDB-X 列存索引 \| DML高效更新的秘密(PkIndex) | https://zhuanlan.zhihu.com/p/3535495219 | 揭秘列存索引下DML操作的高效更新机制 | 列存索引、DML、PkIndex |
| 4 | PolarDB-X 列存索引 \| 列存引擎的诞生 | https://zhuanlan.zhihu.com/p/2359967708 | 讲述PolarDB-X行列混存架构的设计背景与实现 | 列存引擎、行列混存、架构 |
| 5 | PolarDB-X 列存索引 \| 满足事务一致性的快照读 | https://zhuanlan.zhihu.com/p/721776180 | 介绍列存索引如何保证事务一致性的快照读取 | 列存索引、事务一致性、快照读 |
| 6 | PolarDB-X 列存索引 \| 分析TPC-H执行计划 | https://zhuanlan.zhihu.com/p/712869543 | 通过TPC-H测试集分析列存查询执行计划 | 列存索引、TPC-H、执行计划 |
| 7 | 突破性能瓶颈：深度解析PolarDB-X列存分页查询原理 | https://zhuanlan.zhihu.com/p/30965954367 | 深入分析列存分页查询的性能优化原理 | 列存索引、分页查询、性能优化 |
| 8 | 性能提升利器｜ PolarDB-X 列存查询技术解读 | https://zhuanlan.zhihu.com/p/8944067963 | 解读列存查询引擎的分层缓存和性能优化方案 | 列存查询、分层缓存、性能优化 |
| 9 | PolarDB-X 列存索引 \| 全面支持Schema多版本 | https://zhuanlan.zhihu.com/p/704192973 | 介绍列存索引对Schema多版本的支持 | 列存索引、Schema、多版本 |
| 10 | PolarDB-X 列存查询引擎 | https://zhuanlan.zhihu.com/p/701718682 | 全面介绍PolarDB-X列存查询引擎架构 | 列存引擎、查询引擎、架构 |
| 11 | PolarDB-X HTAP新特性 ~ 列存索引 | https://zhuanlan.zhihu.com/p/671755090 | 介绍HTAP场景下列存索引的新特性 | HTAP、列存索引、新特性 |
| 12 | [版本更新] PolarDB-X V2.4 列存引擎开源正式发布 | https://zhuanlan.zhihu.com/p/697195803 | V2.4版本列存引擎开源发布的详细说明 | 版本发布、V2.4、列存引擎 |

---

## 存储引擎核心技术 - Lizard (12篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 13 | PolarDB-X 存储引擎核心技术｜Lizard 分支事务管理 | https://zhuanlan.zhihu.com/p/10182201983 | 介绍Lizard存储引擎的分支事务管理机制 | Lizard、分支事务、事务管理 |
| 14 | PolarDB-X 存储引擎核心技术 \| Lizard XA 两阶段提交算法 | https://zhuanlan.zhihu.com/p/4971725382 | 详解Lizard存储引擎的XA两阶段提交实现 | Lizard、XA、两阶段提交 |
| 15 | PolarDB-X 存储引擎核心技术 \| Lizard B+tree 优化 | https://zhuanlan.zhihu.com/p/714072485 | 讲解Lizard对InnoDB B+tree的优化改进 | Lizard、B+tree、存储引擎 |
| 16 | PolarDB-X 存储引擎核心技术 \| Lizard 多级闪回 | https://zhuanlan.zhihu.com/p/710425976 | 介绍Lizard存储引擎的多级闪回功能 | Lizard、闪回、数据恢复 |
| 17 | PolarDB-X 存储引擎核心技术 \| Lizard 无锁备份 | https://zhuanlan.zhihu.com/p/708461062 | 介绍Lizard存储引擎的无锁备份技术 | Lizard、无锁备份、备份 |
| 18 | PolarDB-X 存储引擎核心技术 \| 大事务优化 | https://zhuanlan.zhihu.com/p/702777153 | 讲解Lizard存储引擎的大事务优化方案 | Lizard、大事务、优化 |
| 19 | PolarDB-X 存储引擎核心技术 \| 索引回表优化 | https://zhuanlan.zhihu.com/p/691515032 | 介绍Lizard存储引擎的索引回表优化 | Lizard、索引、回表优化 |
| 20 | PolarDB-X 存储引擎核心技术 \| Lizard 分布式事务系统 | https://zhuanlan.zhihu.com/p/654126910 | 全面介绍Lizard分布式事务系统 | Lizard、分布式事务 |
| 21 | 让执行器"看见"数据在哪：Lizard 物理寻址优化实践 | https://zhuanlan.zhihu.com/p/2004498148568101493 | 介绍Lizard物理寻址优化技术 | Lizard、物理寻址、优化 |
| 22 | PolarDB-X 分布式事务的实现（六）：基于Lizard事务系统的提交优化（下） | https://zhuanlan.zhihu.com/p/1944114252345489281 | 分布式事务提交优化的下篇 | 分布式事务、Lizard、提交优化 |
| 23 | PolarDB-X 分布式事务的实现（五）：基于Lizard事务系统的提交优化（上） | https://zhuanlan.zhihu.com/p/1926281184373113937 | 分布式事务提交优化的上篇 | 分布式事务、Lizard、提交优化 |
| 24 | PolarDB-X 分布式事务的实现（二）：InnoDB CTS 扩展 | https://zhuanlan.zhihu.com/p/355413022 | 介绍InnoDB CTS扩展实现分布式事务 | 分布式事务、CTS、InnoDB |

---

## 分布式事务 (8篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 25 | PolarDB-X 分布式事务的实现（一） | https://zhuanlan.zhihu.com/p/346765095 | 分布式事务实现的基础原理介绍 | 分布式事务、原理 |
| 26 | PolarDB-X 分布式事务的实现（三）：异步提交优化 | https://zhuanlan.zhihu.com/p/411794605 | 分布式事务异步提交优化技术 | 分布式事务、异步提交 |
| 27 | PolarDB-X 分布式事务的实现（四）：跨地域事务 | https://zhuanlan.zhihu.com/p/415744064 | 跨地域分布式事务的实现方案 | 分布式事务、跨地域 |
| 28 | 无处不在的 MySQL XA 事务 | https://zhuanlan.zhihu.com/p/372300181 | MySQL XA事务的全面介绍 | XA、事务、MySQL |
| 29 | 通过转账测试验证分布式事务 | https://zhuanlan.zhihu.com/p/401355454 | 使用转账场景验证分布式事务 | 分布式事务、测试、验证 |
| 30 | 分布式数据库中的一致性与时间戳 | https://zhuanlan.zhihu.com/p/360690247 | 分布式数据库一致性与时间戳机制 | 一致性、时间戳、分布式 |
| 31 | PolarDB-X 全局时间戳服务的设计 | https://zhuanlan.zhihu.com/p/360160666 | 全局时间戳服务TSO的设计与实现 | TSO、时间戳、全局时钟 |
| 32 | PolarDB-X 如何检测和分析分布式死锁 | https://zhuanlan.zhihu.com/p/495282677 | 分布式死锁的检测与分析方案 | 死锁、分布式、检测 |

---

## 最佳实践系列 (14篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 33 | PolarDB-X 最佳实践（十四）：数据、流量倾斜分析最佳实践（下） | https://zhuanlan.zhihu.com/p/2010354588750983800 | 数据倾斜分析的下篇，提供解决方案 | 最佳实践、数据倾斜 |
| 34 | PolarDB-X 最佳实践（十三）：数据、流量倾斜分析最佳实践（中） | https://zhuanlan.zhihu.com/p/2002017279995577603 | 数据倾斜分析的中篇 | 最佳实践、数据倾斜 |
| 35 | PolarDB-X 最佳实践（十二）：数据、流量倾斜分析最佳实践（上） | https://zhuanlan.zhihu.com/p/1999863263597437488 | 数据倾斜分析的上篇 | 最佳实践、数据倾斜 |
| 36 | PolarDB-X最佳实践（十一）：TABLE_DETAIL视图的X种妙用 | https://zhuanlan.zhihu.com/p/1933539672514078303 | TABLE_DETAIL视图在分区管理中的用法 | 最佳实践、TABLE_DETAIL |
| 37 | PolarDB-X最佳实践（十）：彻底告别锁表，无锁DDL操作指南 | https://zhuanlan.zhihu.com/p/1906663234825598423 | 无锁DDL的使用方法 | 最佳实践、无锁DDL |
| 38 | PolarDB-X最佳实践（九）：价值百万的分区设计实战 | https://zhuanlan.zhihu.com/p/1900610605825630752 | 生产环境分区设计实战经验 | 最佳实践、分区设计 |
| 39 | PolarDB-X最佳实践（八）：如何做Batch写入 | https://zhuanlan.zhihu.com/p/5559961275 | 批量写入的最佳实践 | 最佳实践、Batch写入 |
| 40 | PolarDB-X最佳实践（七）：如何存储 IOT 数据 | https://zhuanlan.zhihu.com/p/892531679 | IOT场景的数据存储最佳实践 | 最佳实践、IOT、时序数据 |
| 41 | PolarDB-X最佳实践（六）：该关注哪些监控指标 | https://zhuanlan.zhihu.com/p/700224767 | 关键监控指标与关注要点 | 最佳实践、监控指标 |
| 42 | PolarDB-X最佳实践（五）：使用通义千问和存储过程快速生成测试数据 | https://zhuanlan.zhihu.com/p/686536485 | 使用AI生成测试数据的方法 | 最佳实践、测试数据 |
| 43 | PolarDB-X最佳实践（四）：如何设计一张订单表 | https://zhuanlan.zhihu.com/p/678519559 | 订单表设计的最佳实践 | 最佳实践、订单表、设计 |
| 44 | PolarDB-X最佳实践（三）：如何实现高效的分页查询 | https://zhuanlan.zhihu.com/p/669225677 | 分页查询优化的最佳实践 | 最佳实践、分页查询 |
| 45 | PolarDB-X最佳实践（二）：如何使用DataWorks将数据同步到MaxCompute | https://zhuanlan.zhihu.com/p/593067259 | DataWorks数据同步最佳实践 | 最佳实践、DataWorks |
| 46 | PolarDB-X最佳实践（一）：如何设计一张用户表 | https://zhuanlan.zhihu.com/p/566229414 | 用户表设计的最佳实践 | 最佳实践、用户表、设计 |

---

## 高可用与容灾 (10篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 47 | PolarDB-X JDBC驱动如何实现高可用探测 | https://zhuanlan.zhihu.com/p/1989339504864154828 | JDBC驱动的高可用探测和无感切换 | 高可用、JDBC、无感切换 |
| 48 | 数据库容灾：深度解析实战PolarDB-X高可用体系 | https://zhuanlan.zhihu.com/p/1951297544215827990 | 全面解析高可用架构和容灾方案 | 高可用、容灾、架构 |
| 49 | PolarDB-X如何在0.5秒内实现"无感切换" | https://zhuanlan.zhihu.com/p/26084514335 | 快速故障切换的技术原理 | 无感切换、高可用 |
| 50 | PolarDB-X 开源用户手册 ~ 基于Paxos的MySQL三副本 | https://zhuanlan.zhihu.com/p/31857098488 | 基于Paxos的三副本高可用实现 | Paxos、三副本、高可用 |
| 51 | 数据库容灾 \| MySQL MGR与阿里云PolarDB-X Paxos的深度对比 | https://zhuanlan.zhihu.com/p/706620854 | MGR与Paxos的深度对比分析 | MGR、Paxos、对比 |
| 52 | 分布式数据库，基于Paxos多副本的两地三中心架构 | https://zhuanlan.zhihu.com/p/666005585 | 两地三中心架构设计方案 | 两地三中心、Paxos、架构 |
| 53 | PolarDB-X 容灾架构之「异地多活」 | https://zhuanlan.zhihu.com/p/334564266 | 异地多活容灾架构设计 | 异地多活、容灾、架构 |
| 54 | PolarDB-X 容灾架构之「两地三中心」 | https://zhuanlan.zhihu.com/p/333068899 | 两地三中心容灾架构设计 | 两地三中心、容灾 |
| 55 | PolarDB-X 容灾架构之「同城三机房」 | https://zhuanlan.zhihu.com/p/331617135 | 同城三机房容灾架构设计 | 同城三机房、容灾 |
| 56 | 真·异地多活架构怎么实现？使用PolarDB-X！ | https://zhuanlan.zhihu.com/p/364240552 | 异地多活架构实现方案 | 异地多活、架构实现 |

---

## 源码解读系列 (20篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 57 | PolarDB-X源码解读（十二）：谈谈in常量查询的设计与优化 | https://zhuanlan.zhihu.com/p/596169564 | in常量查询的设计与优化 | 源码解读、in查询、优化 |
| 58 | PolarDB-X源码解读（十一）：DDL的一生（下） | https://zhuanlan.zhihu.com/p/587632034 | DDL执行流程的下篇 | 源码解读、DDL |
| 59 | PolarDB-X源码解读（十）：事务的一生 | https://zhuanlan.zhihu.com/p/534962590 | 事务执行的完整流程 | 源码解读、事务 |
| 60 | PolarDB-X源码解读（九）：DDL的一生（上） | https://zhuanlan.zhihu.com/p/515688555 | DDL执行流程的上篇 | 源码解读、DDL |
| 61 | PolarDB-X源码解读（番外）：如何实现一个 Paxos | https://zhuanlan.zhihu.com/p/490329189 | Paxos算法的实现原理 | 源码解读、Paxos |
| 62 | PolarDB-X源码解读（八）：Global Binlog 的一生 | https://zhuanlan.zhihu.com/p/501159425 | Global Binlog的完整流程 | 源码解读、Global Binlog |
| 63 | PolarDB-X源码解读（七）：私有协议连接的一生（CN篇） | https://zhuanlan.zhihu.com/p/497726244 | 私有协议连接的实现 | 源码解读、私有协议 |
| 64 | PolarDB-X源码解读（六）：分布式死锁检测 | https://zhuanlan.zhihu.com/p/481468406 | 分布式死锁检测的实现 | 源码解读、死锁检测 |
| 65 | PolarDB-X源码解读（五）：DML 之 Insert 流程 | https://zhuanlan.zhihu.com/p/466296747 | Insert操作的执行流程 | 源码解读、DML、Insert |
| 66 | PolarDB-X源码解读（四）：SQL 的一生 | https://zhuanlan.zhihu.com/p/457450880 | SQL执行的完整生命周期 | 源码解读、SQL执行 |
| 67 | PolarDB-X源码解读（三）：CDC 代码结构 | https://zhuanlan.zhihu.com/p/452174464 | CDC模块的代码结构 | 源码解读、CDC、代码结构 |
| 68 | PolarDB-X源码解读（二）：CN 启动流程 | https://zhuanlan.zhihu.com/p/443608658 | CN节点的启动流程 | 源码解读、CN、启动 |
| 69 | PolarDB-X源码解读（一）：CN 代码结构 | https://zhuanlan.zhihu.com/p/440865170 | CN模块的代码结构 | 源码解读、CN、代码结构 |
| 70 | 源码解读：PolarDB-X中的窗口函数 | https://zhuanlan.zhihu.com/p/617563356 | 窗口函数的实现原理 | 源码解读、窗口函数 |
| 71 | 源码解读：semi join如何避免拉取大表数据？（一） | https://zhuanlan.zhihu.com/p/603908060 | semi join优化原理 | 源码解读、semi join |
| 72 | PolarDB-X 源码解读系列：DML 之 INSERT IGNORE 流程 | https://zhuanlan.zhihu.com/p/574790346 | INSERT IGNORE的执行流程 | 源码解读、DML |
| 73 | PolarDB-X 源码解读（番外）：如何实现一个 Paxos | https://zhuanlan.zhihu.com/p/490329189 | Paxos算法实现详解 | 源码解读、Paxos |
| 74 | 自动化测试数据库逻辑 Bug 的关键 —— SQLancer 在 PolarDB-X 上的应用 | https://zhuanlan.zhihu.com/p/347872750 | SQLancer自动化测试应用 | 源码解读、SQLancer、测试 |
| 75 | PolarDB-X 全局Binlog解读之HA篇 | https://zhuanlan.zhihu.com/p/614839581 | Global Binlog高可用实现 | 源码解读、Global Binlog、HA |
| 76 | PolarDB-X 全局Binlog解读之性能篇(下) | https://zhuanlan.zhihu.com/p/603840937 | Global Binlog性能优化下篇 | 源码解读、Global Binlog |

---

## 版本发布与更新 (10篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 77 | [开源新发布]PolarDB-X V2.4.2 适配和完善周边开源生态 | https://zhuanlan.zhihu.com/p/1953774244833003413 | V2.4.2版本开源生态适配 | 版本发布、V2.4.2 |
| 78 | [开源新发布] PolarDB-X V2.4.1 企业级运维能力增强 | https://zhuanlan.zhihu.com/p/7487125418 | V2.4.1版本企业级运维增强 | 版本发布、V2.4.1 |
| 79 | [版本更新] PolarDB-X V2.4 列存引擎开源正式发布 | https://zhuanlan.zhihu.com/p/697195803 | V2.4列存引擎开源发布 | 版本发布、V2.4、列存 |
| 80 | [重磅更新]PolarDB-X V2.3 集中式和分布式一体化开源发布 | https://zhuanlan.zhihu.com/p/664251876 | V2.3集分一体化开源发布 | 版本发布、V2.3 |
| 81 | [版本更新]PolarDB-X v2.2.1 生产级关键能力开源升级 | https://zhuanlan.zhihu.com/p/616591584 | V2.2.1生产级能力开源 | 版本发布、V2.2.1 |
| 82 | [版本更新] PolarDB-X Paxos 重磅开源了 | https://zhuanlan.zhihu.com/p/491694150 | Paxos协议开源发布 | 版本发布、Paxos |
| 83 | [版本更新] PolarDB-X on OSS 提供冷热数据分离存储 | https://zhuanlan.zhihu.com/p/507355790 | OSS冷热分离功能发布 | 版本发布、OSS、冷热分离 |
| 84 | [版本更新]PolarDB-X v2.2: 企业级和国产ARM适配 开源重磅升级 | https://zhuanlan.zhihu.com/p/578534964 | V2.2企业级和ARM适配 | 版本发布、V2.2、ARM |
| 85 | 开源 PolarDB-X 2.1 版新特性演示 | https://zhuanlan.zhihu.com/p/520774366 | V2.1新特性演示 | 版本发布、V2.1 |
| 86 | [版本更新] PolarDB-X Release Note 20220127 | https://zhuanlan.zhihu.com/p/464513199 | 2022年1月版本更新 | 版本发布、Release Note |

---

## 全局Binlog与CDC (8篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 87 | PolarDB-X CDC之"兼容MySQL,高于MySQL" | https://zhuanlan.zhihu.com/p/698942695 | CDC兼容性和增强特性 | CDC、MySQL兼容 |
| 88 | PolarDB-X 全局Binlog解读之HA篇 | https://zhuanlan.zhihu.com/p/614839581 | Global Binlog高可用实现 | Global Binlog、HA |
| 89 | PolarDB-X 全局Binlog解读之性能篇(下) | https://zhuanlan.zhihu.com/p/603840937 | Global Binlog性能优化下篇 | Global Binlog、性能 |
| 90 | PolarDB-X 全局Binlog解读之性能篇(上) | https://zhuanlan.zhihu.com/p/572412483 | Global Binlog性能优化上篇 | Global Binlog、性能 |
| 91 | PolarDB-X 全局Binlog解读之理论篇 | https://zhuanlan.zhihu.com/p/462995079 | Global Binlog理论基础 | Global Binlog、理论 |
| 92 | PolarDB-X 全局Binlog解读之 DDL | https://zhuanlan.zhihu.com/p/377854011 | Global Binlog DDL处理 | Global Binlog、DDL |
| 93 | PolarDB-X 全局Binlog解读 | https://zhuanlan.zhihu.com/p/369115822 | Global Binlog全面解读 | Global Binlog |
| 94 | PolarDB-X CDC 之数据导入综述 | https://zhuanlan.zhihu.com/p/554284375 | CDC数据导入方案综述 | CDC、数据导入 |

---

## 优化器与SQL执行 (16篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 95 | PolarDB-X 优化器小技巧 \| 统计信息时间列增量预测 | https://zhuanlan.zhihu.com/p/2012222461014484650 | 优化器对时间列统计信息的增量预测 | 优化器、统计信息 |
| 96 | PolarDB-X 技术实践 \| IN查询执行计划管理与预裁剪 | https://zhuanlan.zhihu.com/p/6399026988 | IN查询执行计划优化和预裁剪 | 优化器、IN查询 |
| 97 | PolarDB-X的XPlan索引选择 | https://zhuanlan.zhihu.com/p/681664246 | XPlan索引选择机制 | 优化器、XPlan、索引 |
| 98 | PolarDB-X 优化器核心技术 ~ Join Reorder | https://zhuanlan.zhihu.com/p/470139328 | Join Reorder优化技术 | 优化器、Join Reorder |
| 99 | PolarDB-X 优化器核心技术 ~ 执行计划管理 | https://zhuanlan.zhihu.com/p/398558605 | 执行计划管理技术 | 优化器、执行计划管理 |
| 100 | PolarDB-X 优化器核心技术 ~ 计算下推 | https://zhuanlan.zhihu.com/p/366312701 | 计算下推优化技术 | 优化器、计算下推 |
| 101 | PolarDB-X CBO 优化器技术内幕 | https://zhuanlan.zhihu.com/p/370372242 | CBO优化器内部原理 | 优化器、CBO |
| 102 | PolarDB-X 面向HTAP的 CBO 优化器 | https://zhuanlan.zhihu.com/p/339123438 | HTAP场景的CBO优化器 | 优化器、HTAP、CBO |
| 103 | PolarDB-X 如何优化Batch Insert | https://zhuanlan.zhihu.com/p/525359371 | Batch Insert优化方法 | 优化器、Batch Insert |
| 104 | PolarDB-X 如何限流慢 SQL | https://zhuanlan.zhihu.com/p/448618356 | 慢SQL限流机制 | 优化器、慢SQL、限流 |
| 105 | PolarDB-X SQL 限流 (二) | https://zhuanlan.zhihu.com/p/406517137 | SQL限流技术下篇 | 优化器、SQL限流 |
| 106 | PolarDB-X SQL 限流 (一) | https://zhuanlan.zhihu.com/p/344871237 | SQL限流技术上篇 | 优化器、SQL限流 |
| 107 | PolarDB-X的in常量查询 | https://zhuanlan.zhihu.com/p/585609620 | in常量查询优化 | 优化器、in查询 |
| 108 | 查询性能优化之 Runtime Filter | https://zhuanlan.zhihu.com/p/354754979 | Runtime Filter优化技术 | 优化器、Runtime Filter |
| 109 | 子查询漫谈 | https://zhuanlan.zhihu.com/p/350009405 | 子查询优化技术 | 优化器、子查询 |
| 110 | 数据库等值查询与统计信息 | https://zhuanlan.zhihu.com/p/576987355 | 等值查询与统计信息 | 优化器、统计信息 |

---

## 数据迁移与导入导出 (8篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 111 | 解决方案：自建MySQL手动逻辑迁移到PolarDB-X标准版 | https://zhuanlan.zhihu.com/p/1964654967974507312 | MySQL迁移到PolarDB-X方案 | 数据迁移、MySQL |
| 112 | PolarDB-X 如何优化数据导入导出 | https://zhuanlan.zhihu.com/p/485341931 | 数据导入导出优化 | 数据导入、数据导出 |
| 113 | 数据库导入导出工具BatchTool介绍 | https://zhuanlan.zhihu.com/p/667949705 | BatchTool工具使用 | 数据导入、BatchTool |
| 114 | PolarDB-X 如何优化数据全量抽取 | https://zhuanlan.zhihu.com/p/448618356 | 数据全量抽取优化 | 数据抽取、全量 |
| 115 | PolarDB-X 如何优化Batch Insert | https://zhuanlan.zhihu.com/p/525359371 | Batch Insert优化 | Batch Insert、导入 |
| 116 | 实践教程之如何使用PolarDB-X进行数据导入导出 | https://zhuanlan.zhihu.com/p/632105332 | 数据导入导出实践教程 | 数据导入、实践教程 |
| 117 | 实践教程之PolarDB-X分区管理 | https://zhuanlan.zhihu.com/p/640126189 | 分区管理实践教程 | 分区管理、实践教程 |
| 118 | 实践教程之使用PolarDB-X进行TP负载测试 | https://zhuanlan.zhihu.com/p/638388947 | TP负载测试实践 | 负载测试、实践教程 |

---

## Kubernetes与运维 (14篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 119 | PolarDB-X Operator｜基于两次心跳事务的指定时间点恢复方案介绍 | https://zhuanlan.zhihu.com/p/672985523 | Operator时间点恢复方案 | K8s、Operator、恢复 |
| 120 | PolarDB-X on Kubernetes | https://zhuanlan.zhihu.com/p/382877178 | K8s部署PolarDB-X | K8s、部署 |
| 121 | PolarDB-X Operator 之弹性扩缩容 | https://zhuanlan.zhihu.com/p/474003785 | Operator弹性扩缩容 | K8s、Operator、扩缩容 |
| 122 | 使用开源ProxySQL构建PolarDB-X标准版高可用路由服务 | https://zhuanlan.zhihu.com/p/697117089 | ProxySQL高可用路由 | ProxySQL、高可用 |
| 123 | 实践教程之如何在ARM平台部署PolarDB-X | https://zhuanlan.zhihu.com/p/624772807 | ARM平台部署教程 | ARM、部署、实践 |
| 124 | 实践教程之基于Prometheus+Grafana的PolarDB-X监控体系 | https://zhuanlan.zhihu.com/p/619103972 | 监控体系搭建教程 | 监控、Prometheus、实践 |
| 125 | 实践教程之如何对PolarDB-X集群做动态扩缩容 | https://zhuanlan.zhihu.com/p/593078702 | 动态扩缩容实践 | 扩缩容、实践教程 |
| 126 | 实践教程之如何对PolarDB-X进行备份恢复 | https://zhuanlan.zhihu.com/p/629703085 | 备份恢复实践 | 备份恢复、实践教程 |
| 127 | 实践教程之如何对PolarDB-X的存储节点发起备库重搭 | https://zhuanlan.zhihu.com/p/631506839 | 备库重搭实践 | 备库重搭、实践教程 |
| 128 | 实践教程之采集PolarDB-X SQL日志到ElasticSearch | https://zhuanlan.zhihu.com/p/627473361 | SQL日志采集实践 | SQL日志、ES、实践 |
| 129 | 实践教程之PolarDB-X replica原理和使用 | https://zhuanlan.zhihu.com/p/613010882 | Replica原理和使用 | Replica、实践教程 |
| 130 | 实践教程之用PolarDB-X搭建一个高可用系统 | https://zhuanlan.zhihu.com/p/611084170 | 高可用系统搭建 | 高可用、实践教程 |
| 131 | 实践教程之如何在PolarDB-X中优化慢SQL | https://zhuanlan.zhihu.com/p/598035560 | 慢SQL优化实践 | 慢SQL、实践教程 |
| 132 | 实践教程之如何在PolarDB-X中进行Online DDL | https://zhuanlan.zhihu.com/p/597706748 | Online DDL实践 | DDL、实践教程 |

---

## 架构与设计 (16篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 133 | PolarDB-X 技术架构图 | https://zhuanlan.zhihu.com/p/323785037 | 技术架构全景图 | 架构、技术架构 |
| 134 | PolarDB-X 计算架构之"水平扩展" | https://zhuanlan.zhihu.com/p/325041192 | 水平扩展架构设计 | 架构、水平扩展 |
| 135 | PolarDB-X 计算架构之"混合负载" | https://zhuanlan.zhihu.com/p/326294288 | 混合负载架构设计 | 架构、混合负载 |
| 136 | PolarDB-X 计算架构之"跨地域扩展" | https://zhuanlan.zhihu.com/p/327656963 | 跨地域扩展架构 | 架构、跨地域 |
| 137 | PolarDB-X 存储架构之"共享存储" | https://zhuanlan.zhihu.com/p/328903384 | 共享存储架构 | 架构、共享存储 |
| 138 | PolarDB-X 存储架构之"Paxos 多副本" | https://zhuanlan.zhihu.com/p/330251191 | Paxos多副本架构 | 架构、Paxos |
| 139 | PolarDB-X 存储架构之"基于Paxos的最佳生产实践" | https://zhuanlan.zhihu.com/p/336069649 | Paxos生产实践 | 架构、Paxos、生产实践 |
| 140 | PolarDB-X 原理导读 | https://zhuanlan.zhihu.com/p/636796226 | 核心原理导读 | 架构、原理 |
| 141 | 从中间件到分布式数据库，PolarDB-X的透明之路 | https://zhuanlan.zhihu.com/p/561888087 | 产品演进历程 | 架构、演进 |
| 142 | 选300平米别墅还是90平米小平层？一文带你读懂PolarDB分布式版集分一体化 | https://zhuanlan.zhihu.com/p/680518203 | 集分一体化架构解读 | 架构、集分一体化 |
| 143 | 让X不断延伸, 从跨AZ到跨Region再到跨Cloud | https://zhuanlan.zhihu.com/p/711488096 | 跨云跨地域架构 | 架构、跨云、跨Region |
| 144 | 分布式1024节点！1天玩转PolarDB-X超大规模集群 | https://zhuanlan.zhihu.com/p/608112283 | 超大规模集群架构 | 架构、超大规模 |
| 145 | 十年后数据库还是不敢拥抱NUMA | https://zhuanlan.zhihu.com/p/387117470 | NUMA架构挑战 | 架构、NUMA |
| 146 | 十年后数据库还是不敢拥抱NUMA-续篇 | https://zhuanlan.zhihu.com/p/678047037 | NUMA架构挑战续篇 | 架构、NUMA |
| 147 | Docker 与 Kubernetes | https://zhuanlan.zhihu.com/p/445604490 | 容器化架构介绍 | 架构、Docker、K8s |
| 148 | Intel PAUSE指令变化如何影响MySQL的性能 | https://zhuanlan.zhihu.com/p/581200704 | CPU指令对性能影响 | 架构、CPU、性能 |

---

## 数据分布与分区 (10篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 149 | PolarDB-X 数据分布解读（四）：透明 vs 手动 | https://zhuanlan.zhihu.com/p/556788150 | 透明分布式vs手动分区 | 数据分布、透明分布式 |
| 150 | PolarDB-X 数据分布解读（三）：TPCC与透明分布式 | https://zhuanlan.zhihu.com/p/440801781 | TPCC场景透明分布式 | 数据分布、TPCC |
| 151 | PolarDB-X 数据分布解读（二）：Hash vs Range | https://zhuanlan.zhihu.com/p/424174858 | Hash与Range分区对比 | 数据分布、Hash、Range |
| 152 | PolarDB-X 数据分布解读（一） | https://zhuanlan.zhihu.com/p/395415647 | 数据分布基础介绍 | 数据分布、基础 |
| 153 | PolarDB-X拆分键推荐 | https://zhuanlan.zhihu.com/p/561864772 | 拆分键选择推荐 | 拆分键、分区键 |
| 154 | PolarDB-X 拆分规则变更 | https://zhuanlan.zhihu.com/p/358379137 | 拆分规则变更方案 | 拆分规则、变更 |
| 155 | 谈谈 PolarDB-X 的水平扩展 | https://zhuanlan.zhihu.com/p/357338439 | 水平扩展原理 | 水平扩展、扩展性 |
| 156 | 谈谈 PolarDB-X 的分区实现 | https://zhuanlan.zhihu.com/p/353697706 | 分区实现原理 | 分区、实现原理 |
| 157 | 淘宝订单中隐藏的PolarDB-X最佳实践 | https://zhuanlan.zhihu.com/p/361903384 | 淘宝场景分区实践 | 分区、淘宝、最佳实践 |
| 158 | PolarDB-X 分区键列类型变更 | https://zhuanlan.zhihu.com/p/644077047 | 分区键类型变更 | 分区键、类型变更 |

---

## 冷热分离与OSS (6篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 159 | PolarDB-X on OSS: 冷热数据分离存储 | https://zhuanlan.zhihu.com/p/477664175 | OSS冷热分离存储方案 | 冷热分离、OSS |
| 160 | 1/20的成本！PolarDB-X 冷热分离存储评测 | https://zhuanlan.zhihu.com/p/560662058 | 冷热分离成本评测 | 冷热分离、成本 |
| 161 | 存储成本最高降至原来的5%，PolarDB分布式冷数据归档的业务实践 | https://zhuanlan.zhihu.com/p/666737723 | 冷数据归档业务实践 | 冷数据、归档 |
| 162 | 实践教程之使用PolarDB-X的 TTL 表功能 | https://zhuanlan.zhihu.com/p/643357547 | TTL表功能使用 | TTL、冷热分离 |
| 163 | 实践教程之使用PolarDB-X进行冷热数据归档 | https://zhuanlan.zhihu.com/p/661725663 | 冷热数据归档实践 | 冷热分离、实践 |
| 164 | PolarDB-X OSS冷数据备份恢复 | https://zhuanlan.zhihu.com/p/548635859 | OSS冷数据备份恢复 | OSS、备份恢复 |

---

## 企业级特性 (8篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 165 | PolarDB-X 企业级特性之三权分立 | https://zhuanlan.zhihu.com/p/505301477 | 三权分立安全特性 | 企业级、三权分立 |
| 166 | PolarDB-X 企业级特性之 TDE | https://zhuanlan.zhihu.com/p/454351658 | TDE透明数据加密 | 企业级、TDE、加密 |
| 167 | PolarDB-X 企业级特性之行级访问权限控制 | https://zhuanlan.zhihu.com/p/658236152 | 行级权限控制 | 企业级、权限控制 |
| 168 | PolarDB-X 分布式数据库中的外键 | https://zhuanlan.zhihu.com/p/660462275 | 外键支持特性 | 企业级、外键 |
| 169 | PolarDB-X 智能诊断之热点分析 | https://zhuanlan.zhihu.com/p/531899742 | 热点智能诊断 | 企业级、智能诊断 |
| 170 | PolarDB-X 智能诊断之优化慢 SQL | https://zhuanlan.zhihu.com/p/456481689 | 慢SQL智能诊断 | 企业级、慢SQL诊断 |
| 171 | PolarDB-X 中的 AUTO_INCREMENT 兼容性 | https://zhuanlan.zhihu.com/p/540512887 | AUTO_INCREMENT兼容 | 企业级、自增ID |
| 172 | 业内首家！PolarDB-X 与 MySQL AUTO_INCREMENT 完全兼容 | https://zhuanlan.zhihu.com/p/539245300 | 自增ID完全兼容 | 企业级、自增ID |

---

## 性能测试与优化 (8篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 173 | PolarDB-X 在 ClickBench 数据集的优化实践 | https://zhuanlan.zhihu.com/p/17422296783 | ClickBench性能优化 | 性能测试、ClickBench |
| 174 | PolarDB-X、OceanBase、CockroachDB、TiDB二级索引写入性能测评 | https://zhuanlan.zhihu.com/p/559435890 | 二级索引性能对比 | 性能测试、对比 |
| 175 | 深度优化 \| PolarDB-X 基于向量化SIMD指令的探索 | https://zhuanlan.zhihu.com/p/650910891 | SIMD向量化优化 | 性能优化、SIMD |
| 176 | PolarDB-X 如何用 15M 内存跑 TPC-H 1G? | https://zhuanlan.zhihu.com/p/363435372 | 低内存TPC-H测试 | 性能测试、TPC-H |
| 177 | PolarDB-X 针对跑批场景的思考和实践 | https://zhuanlan.zhihu.com/p/647239594 | 跑批场景优化 | 性能优化、跑批 |
| 178 | PolarDB-X 热点优化系列（二）：如何支持淘宝大卖家分区热点 | https://zhuanlan.zhihu.com/p/447564638 | 大卖家热点优化 | 性能优化、热点 |
| 179 | PolarDB-X 热点优化系列 (一)：如何支持淘宝库存热点更新 | https://zhuanlan.zhihu.com/p/432580272 | 库存热点优化 | 性能优化、热点 |
| 180 | 偏高并发场景的实践和优化 | https://zhuanlan.zhihu.com/p/447564638 | 高并发场景优化 | 性能优化、高并发 |

---

## 多租户与SaaS (4篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 181 | 典型场景 \| PolarDB-X 如何支撑SaaS多租户 | https://zhuanlan.zhihu.com/p/652980296 | SaaS多租户支撑方案 | 多租户、SaaS |
| 182 | PolarDB-X 多租户的设计和思考 | https://zhuanlan.zhihu.com/p/551691114 | 多租户架构设计 | 多租户、架构 |
| 183 | PolarDB-X助攻《香肠派对》百亿好友关系实现毫秒级查询 | https://zhuanlan.zhihu.com/p/692928074 | 游戏场景应用案例 | 案例、游戏 |
| 184 | 客户说｜从4小时到15分钟，一次分布式数据库的丝滑体验 | https://zhuanlan.zhihu.com/p/685761988 | 客户成功案例 | 案例、客户 |

---

## 数据恢复与备份 (4篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 185 | PolarDB-X 快照：Agent 操作数据的"后悔药" | https://zhuanlan.zhihu.com/p/2017204704627696233 | 数据快照功能介绍 | 快照、数据恢复 |
| 186 | PolarDB-X 是如何拯救误删数据的你（一） | https://zhuanlan.zhihu.com/p/367137740 | 误删数据恢复上篇 | 数据恢复、误删 |
| 187 | PolarDB-X 是如何拯救误删数据的你（二）：备份恢复 | https://zhuanlan.zhihu.com/p/429977533 | 误删数据恢复下篇 | 数据恢复、备份 |
| 188 | PolarDB-X 全局 Binlog 和备份恢复能力解读 | https://zhuanlan.zhihu.com/p/385608337 | 备份恢复能力解读 | 备份恢复、Global Binlog |

---

## 论文解读 (6篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 189 | 论文解读：CockroachDB 事务处理 | https://zhuanlan.zhihu.com/p/543497168 | CockroachDB事务论文 | 论文、CockroachDB |
| 190 | 论文解读：机器学习加持的查询优化器(SIGMOD 2021 Best Paper) | https://zhuanlan.zhihu.com/p/391159830 | ML查询优化器论文 | 论文、ML、优化器 |
| 191 | 论文解读：Multi-way Join 列式内存数据库Join优化 | https://zhuanlan.zhihu.com/p/378858715 | Multi-way Join论文 | 论文、Join |
| 192 | 论文解读：MatrixKV: Reducing Write Stalls and Write Amplification in LSM-tree | https://zhuanlan.zhihu.com/p/461226502 | MatrixKV论文解读 | 论文、LSM-tree |
| 193 | 论文解读：Differentiated Key-Value Storage Management for Balanced I/O Performance | https://zhuanlan.zhihu.com/p/452171652 | KV存储管理论文 | 论文、KV存储 |
| 194 | 论文解读：Parametric query optimize 综述 | https://zhuanlan.zhihu.com/p/522496331 | 参数化查询优化论文 | 论文、查询优化 |

---

## Join与查询优化 (6篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 195 | 分布式数据库如何实现 Join？ | https://zhuanlan.zhihu.com/p/349420901 | Join实现原理 | Join、分布式 |
| 196 | 分布式数据库如何实现 Join（二） | https://zhuanlan.zhihu.com/p/379967662 | Join实现原理续篇 | Join、分布式 |
| 197 | 源码解读：semi join如何避免拉取大表数据？（一） | https://zhuanlan.zhihu.com/p/603908060 | semi join优化 | Join、semi join |
| 198 | PolarDB-X 全局二级索引 | https://zhuanlan.zhihu.com/p/568686051 | 全局二级索引介绍 | GSI、二级索引 |
| 199 | 聊聊数据库中的烂索引 | https://zhuanlan.zhihu.com/p/649990447 | 索引设计优化 | 索引、优化 |
| 200 | PolarDB-X DDL也要追求ACID？ | https://zhuanlan.zhihu.com/p/435352604 | DDL的ACID保证 | DDL、ACID |

---

## 实践教程系列 (20篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 201 | 实践教程之快速使用PolarDB-X | https://zhuanlan.zhihu.com/p/589630048 | 快速入门教程 | 实践教程、入门 |
| 202 | 实践教程之快速安装部署PolarDB-X | https://zhuanlan.zhihu.com/p/583389666 | 安装部署教程 | 实践教程、部署 |
| 203 | 实践教程之体验PolarDB-X分布式事务和数据分区 | https://zhuanlan.zhihu.com/p/620979702 | 分布式事务体验 | 实践教程、事务 |
| 204 | 实践教程之PolarDB-X与Flink搭建实时数据大屏 | https://zhuanlan.zhihu.com/p/601866031 | Flink集成教程 | 实践教程、Flink |
| 205 | 实践教程之PolarDB-X与大数据等系统互通 | https://zhuanlan.zhihu.com/p/591974208 | 大数据互通教程 | 实践教程、大数据 |
| 206 | 实践教程之使用PolarDB-X参数模板 | https://zhuanlan.zhihu.com/p/632067283 | 参数模板使用 | 实践教程、参数 |
| 207 | 实践教程之PolarDB-X分区管理 | https://zhuanlan.zhihu.com/p/640126189 | 分区管理教程 | 实践教程、分区 |
| 208 | 实践教程之使用PolarDB-X进行TP负载测试 | https://zhuanlan.zhihu.com/p/638388947 | TP负载测试教程 | 实践教程、负载测试 |
| 209 | 实践教程之使用PolarDB-X进行数据导入导出 | https://zhuanlan.zhihu.com/p/632105332 | 数据导入导出教程 | 实践教程、数据导入 |
| 210 | 实践教程之对PolarDB-X存储节点发起备库重搭 | https://zhuanlan.zhihu.com/p/631506839 | 备库重搭教程 | 实践教程、备库 |
| 211 | 实践教程之对PolarDB-X进行备份恢复 | https://zhuanlan.zhihu.com/p/629703085 | 备份恢复教程 | 实践教程、备份恢复 |
| 212 | 实践教程之采集PolarDB-X SQL日志到ElasticSearch | https://zhuanlan.zhihu.com/p/627473361 | SQL日志采集教程 | 实践教程、日志 |
| 213 | 实践教程之在ARM平台部署PolarDB-X | https://zhuanlan.zhihu.com/p/624772807 | ARM部署教程 | 实践教程、ARM |
| 214 | 实践教程之基于Prometheus+Grafana的监控体系 | https://zhuanlan.zhihu.com/p/619103972 | 监控体系教程 | 实践教程、监控 |
| 215 | 实践教程之PolarDB-X replica原理和使用 | https://zhuanlan.zhihu.com/p/613010882 | Replica使用教程 | 实践教程、Replica |
| 216 | 实践教程之用PolarDB-X搭建高可用系统 | https://zhuanlan.zhihu.com/p/611084170 | 高可用搭建教程 | 实践教程、高可用 |
| 217 | 实践教程之在PolarDB-X中优化慢SQL | https://zhuanlan.zhihu.com/p/598035560 | 慢SQL优化教程 | 实践教程、慢SQL |
| 218 | 实践教程之在PolarDB-X中进行Online DDL | https://zhuanlan.zhihu.com/p/597706748 | Online DDL教程 | 实践教程、DDL |
| 219 | 实践教程之使用PolarDB-X的TTL表功能 | https://zhuanlan.zhihu.com/p/643357547 | TTL表使用教程 | 实践教程、TTL |
| 220 | 实践教程之使用PolarDB-X进行冷热数据归档 | https://zhuanlan.zhihu.com/p/661725663 | 冷热归档教程 | 实践教程、冷热分离 |

---

## 其他技术文章 (18篇)

| 序号 | 标题 | 链接 | 一句话总结 | 关键词 |
|------|------|------|-----------|--------|
| 221 | 加入PolarDB分布式数据库团队 | https://zhuanlan.zhihu.com/p/32492662211 | 团队招聘信息 | 招聘、团队 |
| 222 | 阿里云 PolarDB-X 团队25届实习生招聘 | https://zhuanlan.zhihu.com/p/690216969 | 实习生招聘 | 招聘、实习 |
| 223 | 2025阿里云PolarDB开发者大会 | https://zhuanlan.zhihu.com/p/23018937306 | 开发者大会预告 | 活动、大会 |
| 224 | PolarDB-X 开源分布式数据库进阶营 | https://zhuanlan.zhihu.com/p/584029939 | 进阶培训活动 | 活动、培训 |
| 225 | PolarDB-X大降价40%！ | https://zhuanlan.zhihu.com/p/651920315 | 产品降价信息 | 产品、降价 |
| 226 | 是的，我们开源了 | https://zhuanlan.zhihu.com/p/424345630 | 开源发布文章 | 开源、发布 |
| 227 | PolarDB-X 开源代码更名公告 | https://zhuanlan.zhihu.com/p/592302558 | 代码更名公告 | 开源、更名 |
| 228 | 将WordPress从MySQL迁移到PolarDB-X | https://zhuanlan.zhihu.com/p/382964588 | WordPress迁移案例 | 案例、WordPress |
| 229 | PolarDB-X JDBC 驱动 | https://zhuanlan.zhihu.com/p/11275225196 | JDBC驱动介绍 | JDBC、驱动 |
| 230 | 谈谈PolarDB-X在读写分离场景的实践 | https://zhuanlan.zhihu.com/p/579026526 | 读写分离实践 | 读写分离 |
| 231 | PolarDB-X 与 MySQL 的兼容性 | https://zhuanlan.zhihu.com/p/337495093 | MySQL兼容性说明 | 兼容性、MySQL |
| 232 | PolarDB-X 三副本存储引擎 | https://zhuanlan.zhihu.com/p/535496764 | 三副本存储引擎 | 存储引擎、三副本 |
| 233 | PolarDB-X 数据节点备库重搭 | https://zhuanlan.zhihu.com/p/610048400 | 备库重搭操作 | 备库重搭 |
| 234 | 分布式数据库，挂掉两台机器会发生什么 | https://zhuanlan.zhihu.com/p/625604552 | 故障场景分析 | 故障、容灾 |
| 235 | 分布式数据库，如何有效评测国产数据库的事务一致性 | https://zhuanlan.zhihu.com/p/622794489 | 事务一致性评测 | 评测、一致性 |
| 236 | PolarDB-X 混沌测试实践 | https://zhuanlan.zhihu.com/p/657098769 | 混沌测试方法 | 混沌测试 |
| 237 | 聊聊数据库中的 savepoint | https://zhuanlan.zhihu.com/p/647808638 | savepoint机制 | savepoint |
| 238 | 主流CPU性能摸底 | https://zhuanlan.zhihu.com/p/540655373 | CPU性能测试 | CPU、性能 |

---

## 使用说明

### 如何查找文章

1. **按关键词搜索**: 使用表格中的关键词列快速定位相关文章
2. **按类别浏览**: 文章按主题分类，便于系统性学习
3. **按问题定位**: 根据遇到的问题选择对应类别的文章

### 文章分类速查

| 问题类型 | 推荐类别 |
|----------|----------|
| 列存索引相关问题 | 列存索引与HTAP |
| 存储引擎、事务问题 | 存储引擎核心技术、分布式事务 |
| SQL性能问题 | 优化器与SQL执行 |
| 生产环境操作 | 最佳实践系列、实践教程系列 |
| 高可用、故障恢复 | 高可用与容灾 |
| 版本升级、新特性 | 版本发布与更新 |
| 源码学习 | 源码解读系列 |
| 数据迁移 | 数据迁移与导入导出 |
| K8s部署运维 | Kubernetes与运维 |
| 架构设计 | 架构与设计 |

### 链接说明

- 所有链接均为知乎专栏文章直接链接
- 部分文章可能需要登录知乎后访问
- 建议关注PolarDB-X知乎机构号获取最新文章

---

*注：本文档收录全部238篇文章，持续更新中*
