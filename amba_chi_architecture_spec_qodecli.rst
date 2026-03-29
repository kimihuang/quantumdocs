AMBA CHI 架构规范概览
====================================

本文档基于 **AMBA CHI Architecture Specification (IHI0050 Issue G)** 分析概览，
参考文档：docs/pdf/arm/IHI0050G_amba_chi_architecture_spec.md。

.. contents::
   :depth: 2
   :local:


文档基本信息
================

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - 属性
     - 值
   * - 文档编号
     - IHI0050
   * - 版本
     - Issue G
   * - 发布日期
     - 2024年3月
   * - 适用范围
     - AMBA Coherent Hub Interface (CHI) 架构规范


架构概述
================

CHI (Coherent Hub Interface) 是 ARM AMBA 总线家族中的**一致性互连协议**，
用于实现多核处理器系统中的硬件一致性缓存。

.. note::
   参考：Chapter B1 Introduction

1 核心特性
----------------

CHI 架构的主要特性包括：

- **硬件一致性**：支持多核处理器间的硬件缓存一致性，无需软件干预
- **高性能扩展**：适用于高性能计算和服务器级 SoC
- **拓扑无关**：支持 Crossbar、Ring、Mesh 等多种拓扑结构
- **丰富的事务类型**：支持读写、原子操作、缓存预取等
- **端到端 QoS**：支持服务质量控制
- **低功耗设计**：支持链路级时钟门控和组件激活/去激活

.. list-table:: CHI 关键特性列表
   :header-rows: 1
   :widths: 30 70

   * - 特性类别
     - 具体内容
   * - 一致性支持
     - 硬件维护缓存一致性，支持 Write-Invalidate 协议
   * - 事务类型
     - 丰富的事务类型，支持原子操作、独占访问
   * - 数据移动
     - 高效的数据放置和移动，Stash 机制
   * - 虚拟内存
     - DVM (Distributed Virtual Memory) 操作支持
   * - 安全性
     - ARM TrustZone 事务级安全支持，MTE 内存标记扩展
   * - 可靠性
     - 错误报告和传播，Data Poisoning 机制
   * - 电源管理
     - 链路级门控，组件激活/去激活序列


架构分层
================

CHI 协议分为三个层次，每层负责不同的功能：

.. list-table:: CHI 架构层次
   :header-rows: 1
   :widths: 15 20 65

   * - 层次
     - 通信粒度
     - 主要功能
   * - Protocol (协议层)
     - Transaction (事务)
     - 生成和处理请求/响应；定义缓存状态转换；定义事务流；管理协议级流控
   * - Network (网络层)
     - Packet (包)
     - 协议消息打包；确定源/目标节点 ID 用于路由
   * - Link (链路层)
     - Flit (流控单元)
     - 提供流控制；管理链路通道，确保无死锁交换

.. note::
   参考：B1.1.3 Architecture layers


组件类型
================

CHI 协议定义了多种节点类型，按功能分类：

1 Request Node (RN) - 请求节点
----------------------------------

请求节点生成协议事务（读/写）发送到互连。

.. list-table:: Request Node 类型
   :header-rows: 1
   :widths: 15 85

   * - 类型
     - 描述
   * - RN-F
     - Fully Coherent Request Node：包含硬件一致性缓存；可生成所有协议事务；支持所有 Snoop 事务
   * - RN-D
     - IO Coherent Request Node with DVM：无硬件一致性缓存；接收 DVM 事务；生成部分协议事务
   * - RN-I
     - IO Coherent Request Node：无硬件一致性缓存；不接收 DVM 事务；无需 Snoop 功能

2 Home Node (HN) - 归节点
------------------------------

位于互连内部，接收来自 RN 的协议事务。

.. list-table:: Home Node 类型
   :header-rows: 1
   :widths: 15 85

   * - 类型
     - 描述
   * - HN-F
     - Fully Coherent Home Node：接收除 DVMOp 外的所有请求；包含 PoC (Point of Coherence)；可作为 PoS (Point of Serialization)；可选包含目录或 Snoop 过滤器
   * - HN-I
     - Non-coherent Home Node：处理有限的协议事务子集；不包含 PoC；用于 IO 子系统

3 Subordinate Node (SN) - 从节点
------------------------------------

接收来自 HN 的请求，完成操作并返回响应。通常是系统中最下游的代理，
如内存控制器。

4 Miscellaneous Node (MN) - 杂项节点
-----------------------------------------

位于互连内部，接收来自 RN 的 DVM 消息，完成操作并返回响应。

.. note::
   参考：B1.6 Component naming


拓扑结构
================

CHI 架构本身与拓扑无关，但支持多种优化拓扑：

.. list-table:: 拓扑类型对比
   :header-rows: 1
   :widths: 15 35 50

   * - 拓扑
     - 特点
     - 适用场景
   * - Crossbar
     - 简单构建；低延迟；天然有序网络
     - 节点数较少的小型互连
   * - Ring
     - 布线效率与延迟的折中；延迟随节点数线性增加
     - 中等规模互连
   * - Mesh
     - 高带宽；模块化设计；易于扩展
     - 大规模互连系统

.. note::
   参考：B1.2 Topology


事务分类
================

CHI 协议支持丰富的协议事务类型：

.. list-table:: 事务分类
   :header-rows: 1
   :widths: 18 82

   * - 分类
     - 支持的事务
   * - Read (读)
     - ReadNoSnp, ReadOnce*, ReadClean, ReadNotSharedDirty, ReadShared, ReadUnique, ReadPreferUnique, MakeReadUnique
   * - Dataless (无数据)
     - CleanUnique, MakeUnique, Evict, StashOnce*, CleanShared, CleanSharedPersist*, CleanInvalid, MakeInvalid
   * - Write (写)
     - WriteNoSnp*, WriteUnique*, WriteBack*, WriteCleanFull, WriteEvictFull
   * - Combined Write
     - WriteNoSnp*CleanInv, WriteNoSnp*CleanSh, WriteUnique*CleanSh, WriteBack*CleanInv 等
   * - Atomic (原子)
     - AtomicStore, AtomicLoad, AtomicSwap, AtomicCompare
   * - Other (其他)
     - DVMOp, PrefetchTgt, PCrdReturn
   * - Snoop (监听)
     - SnpOnce, SnpStash*, SnpClean*, SnpUnique*, SnpMakeInvalid 等

.. note::
   参考：B1.4 Transaction classification


一致性模型
================

1 一致性定义
----------------

硬件一致性使系统组件能够共享内存，无需软件缓存维护操作。

**一致性要求**：如果两个组件写入同一内存位置，所有组件必须以相同顺序观察到这些写入。

2 一致性机制
----------------

CHI 使用 **Write-Invalidate 协议**：

- 当写入共享缓存行时，必须先使所有其他副本失效
- 协议确保：

  - 若缓存行是唯一副本，可直接修改，无需通知其他组件
  - 若缓存行可能存在于其他缓存，必须通过适当事务通知

3 关键点定义
----------------

.. list-table:: 系统关键点
   :header-rows: 1
   :widths: 15 85

   * - 术语
     - 定义
   * - PoC
     - Point of Coherence（一致性点）：所有代理看到相同内存副本的位置，通常是 HN-F
   * - PoS
     - Point of Serialization（序列化点）：确定不同代理请求顺序的位置
   * - PoP
     - Point of Persistence（持久化点）：断电后数据仍保持的位置
   * - PoDP
     - Point of Deep Persistence：电源和备用电池同时故障时数据仍保持的位置
   * - PoPA
     - Point of Physical Aliasing：对一个物理地址空间的更新对所有 PAS 可见的位置

.. note::
   参考：B1.5 Coherence overview


缓存状态模型
==================

CHI 定义了七状态缓存模型，基于以下特征组合：

1 状态特征
----------------

.. list-table:: 缓存状态特征
   :header-rows: 1
   :widths: 20 80

   * - 特征
     - 含义
   * - Valid/Invalid
     - Valid: 缓存行存在于缓存中；Invalid: 缓存行不存在
   * - Unique/Shared
     - Unique: 仅存在于当前缓存；Shared: 可能存在于多个缓存
   * - Clean/Dirty
     - Clean: 不负责更新主存；Dirty: 相对于主存已修改，需负责更新主存
   * - Full/Partial/Empty
     - Full: 所有字节有效；Partial: 部分字节有效；Empty: 无有效字节

2 缓存状态
----------------

.. list-table:: 缓存状态定义
   :header-rows: 1
   :widths: 15 30 55

   * - 状态
     - 全称
     - 描述
   * - I
     - Invalid
     - 无效，缓存行不存在
   * - UC
     - Unique Clean
     - 唯一干净，可修改无需通知
   * - UCE
     - Unique Clean Empty
     - 唯一干净空，无有效数据字节
   * - UD
     - Unique Dirty
     - 唯一脏，需负责更新主存
   * - UDP
     - Unique Dirty Partial
     - 唯一脏部分，仅部分字节有效
   * - SC
     - Shared Clean
     - 共享干净，可能存在于其他缓存
   * - SD
     - Shared Dirty
     - 共享脏，需负责更新主存

.. note::
   参考：B1.5.2 Cache state model


关键术语
================

.. list-table:: 关键术语定义
   :header-rows: 1
   :widths: 20 80

   * - 术语
     - 定义
   * - Transaction
     - 事务：执行单一操作，如读或写内存
   * - Message
     - 消息：协议层概念，两个组件间交换的粒度，如 Request、Data response、Snoop request
   * - Packet
     - 包：互连端点间传输的粒度，包含路由信息，一个消息可由多个包组成
   * - Flit
     - Flow control unIT：最小流控单元，CHI 中所有包由单个 Flit 组成
   * - Phit
     - Physical layer transfer unIT：相邻网络设备间单次传输，CHI 中所有 Flit 由单个 Phit 组成
   * - Protocol Credit
     - 协议信用：Completer 保证接受事务的信用
   * - L-Credit
     - Link layer Credit：链路层信用，保证 Flit 在链路另一端被接受

.. note::
   参考：B1.3 Terminology


协议章节概览
==================

规范主体分为以下主要章节：

.. list-table:: 规范章节
   :header-rows: 1
   :widths: 15 85

   * - 章节
     - 内容
   * - B1 Introduction
     - 架构概述、术语定义、一致性模型
   * - B2 Transactions
     - 通道概述、字段定义、事务结构、ID 流、排序规则
   * - B3 Network Layer
     - 系统地址映射、节点 ID、目标 ID 确定
   * - B4 Coherence Protocol
     - 缓存行状态、请求类型、Snoop 类型、响应类型、状态转换
   * - B5 Interconnect Protocol Flows
     - 各类事务流程示例
   * - B6 Exclusive accesses
     - 独占访问机制
   * - B7 Cache Stashing
     - 缓存预取机制
   * - B8 DVM Operations
     - 分布式虚拟内存操作
   * - B9 Error Handling
     - 错误处理机制
   * - B10 Realm Management Extension
     - 领域管理扩展 (RME)
   * - B11 System Control
     - QoS、数据源、MPAM、追踪
   * - B12 Memory Tagging
     - 内存标记扩展 (MTE)
   * - B13-B15 Link Layer
     - 链路层、链路握手、系统一致性接口
   * - B16 Properties
     - 接口属性、参数、广播信号


扩展特性
================

CHI 协议支持多种 ARM 扩展：

.. list-table:: 支持的扩展
   :header-rows: 1
   :widths: 20 80

   * - 扩展
     - 描述
   * - MTE
     - Memory Tagging Extension：内存标记扩展，检查内存数据的正确使用
   * - RME
     - Realm Management Extension：领域管理扩展，支持安全隔离
   * - MPAM
     - Memory System Resource Partitioning and Monitoring：内存系统资源分区与监控
   * - MEC
     - Memory Encryption Contexts：内存加密上下文


总结
================

AMBA CHI 是 ARM 高性能 SoC 的核心互连协议，具有以下关键特点：

1. **分层架构**：Protocol、Network、Link 三层清晰分工
2. **组件丰富**：RN-F/RN-D/RN-I、HN-F/HN-I、SN、MN 多种节点类型
3. **一致性保证**：基于 Write-Invalidate 的硬件缓存一致性
4. **七状态模型**：I/UC/UCE/UD/UDP/SC/SD 状态支持精确的一致性管理
5. **灵活拓扑**：支持 Crossbar、Ring、Mesh 等多种拓扑
6. **丰富事务**：读、写、原子、DVM 等多种事务类型
7. **扩展支持**：MTE、RME、MPAM 等现代安全与性能扩展

本概览文档覆盖了 CHI 架构的核心概念，为深入理解后续章节奠定基础。


----
*文档生成日期：2026-03-28*
*基于 IHI0050 Issue G (Mar 2024)*
