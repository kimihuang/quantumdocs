==============================================
AMBA CHI Architecture Specification 结构梳理
==============================================

本文档基于 **IHI0050G (Issue G, Mar 2024)** -- AMBA CHI Architecture Specification，
对全篇文档结构进行系统性梳理。

参考文档：docs/pdf/arm/IHI0050G_amba_chi_architecture_spec.md

.. contents::
   :depth: 3
   :local:

---------------------------------
文档基本信息
---------------------------------

- **文档编号**: IHI0050G
- **版本**: Issue G (2024年3月)
- **总行数**: ~23,240 行
- **保密级别**: Non-Confidential
- **文档状态**: Final (developed product)
- **发布历史**: A (2014/06) -> B (2017/08) -> C (2018/05) -> D (2019/08) -> E.a (2020/08) -> E.b (2021/08) -> E.c (2022/09) -> F (2022/09) -> F.b (2024/02) -> G (2024/03)

---------------------------------
文档总体结构
---------------------------------

文档分为 **4 个 Part**，包含 **19 个 Chapter**，外加前置页面（封面、版权、许可证）和目录。

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Part
     - 行号范围
     - 行数
     - 内容概述
   * - 前置页面 + 目录
     - 1-827
     - ~827
     - 封面、版权声明、AMBA许可证、保密性声明、目录
   * - Part A -- Preface
     - 828-1051
     - ~224
     - 文档约定、术语、阅读建议
   * - Part B -- Specification
     - 1052-21552
     - ~20,501
     - 核心规范，含 16 个 Chapter
   * - Part C -- Appendices
     - 21553-22833
     - ~1,281
     - 附录：消息字段映射、通信节点、版本变更
   * - Part D -- Glossary
     - 22834-23240
     - ~407
     - 术语表


Part A -- Preface
---------------------------------

*源文档行号: 828-1051*

About this specification (B1.0.1)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

定义了本文档的**目标读者**：系统集成人员、SoC 设计人员、验证工程师、软件工具开发人员。

Using this specification (B1.0.2)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

说明了本文档的使用约定，包括版本标记方式（**IMPLEMENTATION DEFINED** 表示由实现定义、**IMPLEMENTATION SPECIFIC** 表示特定实现的行为）。

Conventions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

定义了文档中使用的排版约定、时序图、Time-Space 图、信号命名、数字表示法。

Additional reading
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

推荐的相关 ARM 出版物，包括 AMBA AXI/ATB/APB 规范、Arm 架构参考手册、SMMU 规范等。

Feedback
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

文档反馈渠道和术语包容性承诺。

Part B -- Specification
---------------------------------

*源文档行号: 1052-21552*

Part B 是文档的**绝对主体**（约占总行数 88%），包含 16 个 Chapter，覆盖协议、网络层、链路层等完整规范。

.. list-table::
   :header-rows: 1
   :widths: 10 25 15 50

   * - 行号
     - Chapter
     - 行数
     - 核心内容
   * - 1056
     - B1 Introduction
     - ~552
     - 架构概述、拓扑、术语、事务分类、一致性概述
   * - 1608
     - B2 Transactions
     - ~6,555
     - 通道、字段、事务结构、标识符、排序、数据传输
   * - 8163
     - B3 Network Layer
     - ~302
     - SAM、Node ID、TgtID 确定
   * - 8465
     - B4 Coherence Protocol
     - ~3,445
     - 缓存行状态、请求/响应类型、Snoop、状态转换
   * - 11910
     - B5 Interconnect Protocol Flows
     - ~1,434
     - Read/Dataless/Write/Atomic/Stash 事务流
   * - 13344
     - B6 Exclusive accesses
     - ~359
     - 独占访问、独占监视器
   * - 13703
     - B7 Cache Stashing
     - ~309
     - Stash 写入、独立 Stash 请求
   * - 14012
     - B8 DVM Operations
     - ~909
     - 分布式虚拟内存事务、DVM 消息
   * - 14921
     - B9 Error Handling
     - ~879
     - 包级/子包级错误、奇偶校验
   * - 15800
     - B10 Realm Management Extension
     - ~575
     - PAS、缓存维护、DVM、MPAM、MEC、DA/CDA
   * - 16375
     - B11 System Control, Debug, Trace, Monitoring
     - ~710
     - QoS、数据源/目标、MPAM、Completer Busy、Trace
   * - 17085
     - B12 Memory Tagging
     - ~643
     - MTE 消息扩展、Tag 一致性、各事务 Tag 规则
   * - 17728
     - B13 Link Layer
     - ~2,104
     - 链路、Flit、通道、端口、接口信号、协议 Flit 字段
   * - 19832
     - B14 Link Handshake
     - ~841
     - 时钟初始化、Credit、低功耗、激活/去激活
   * - 20673
     - B15 System Coherency Interface
     - ~131
     - 系统一致性接口概述与握手
   * - 20804
     - B16 Properties, Parameters, Broadcast Signals
     - ~749
     - 接口属性/参数（26个子参数）、广播信号


Chapter B1 -- Introduction
----------------------------

*源文档行号: 1056-1607*

B1.1 Architecture overview
~~~~~~~~~~~~~~~~~~~~~~~~~~

CHI 架构是一种**可扩展的一致性 Hub 接口和片上互连**，支持灵活的组件连接拓扑。

**关键特性**:

- 可扩展架构，支持从小型到大型系统的模块化设计
- 独立的分层方法：**Protocol 层**、**Network 层**、**Link 层**
- 基于包的通信
- 所有事务由互连中的 **Home Node** 协调所需的 Snoop、缓存和内存访问
- 一致性协议支持 64 字节缓存行粒度
- 支持 MESI 和 MOESI 缓存模型，支持从任意缓存状态转发数据
- 支持 Snoop Filter 和 Directory 系统以实现 Snoop 扩展
- 支持原子操作、独占访问、Cache Stashing、DVM 操作
- 端到端 QoS 支持
- 支持 Arm MTE (Memory Tagging Extension)
- 支持 Arm RME (Realm Management Extension)
- 可配置数据宽度
- 逐事务的 TrustZone 支持
- 错误报告与传播
- 低功耗信号（Flit 级时钟门控、组件激活/去激活）

**支持的组件类型**:

独立处理器、处理器集群、图形处理器、内存控制器、I/O 桥、PCIe 子系统、互连。

B1.1.3 Architecture layers
~~~~~~~~~~~~~~~~~~~~~~~~~~~

CHI 架构分为三层：

.. list-table::
   :header-rows: 1
   :widths: 15 20 15 50

   * - 层
     - 通信粒度
     - 典型实体
     - 主要功能
   * - Protocol
     - Transaction
     - 协议消息
     - 生成/处理请求和响应；定义缓存状态转换规则；定义事务流；管理协议级流控
   * - Network
     - Packet
     - 网络包
     - 对协议消息进行包化；确定源/目标 Node ID 并添加路由信息
   * - Link
     - Flit
     - 链路 Flit
     - 网络设备间的流控；管理链路通道，提供无死锁切换

.. note::

   在 CHI 中，所有包（Packet）由单个 Flit 组成，所有 Flit 由单个 Phit 组成。

B1.2 Topology
~~~~~~~~~~~~~~~

CHI 架构主要与拓扑无关，但包含某些拓扑相关的优化。文档给出了三种典型拓扑：

.. list-table::
   :header-rows: 1
   :widths: 15 50 35

   * - 拓扑
     - 特点
     - 适用场景
   * - Crossbar
     - 结构简单、自然有序、低延迟、布线开销随节点数平方增长
     - 小规模互连
   * - Ring
     - 布线效率与延迟的折中，延迟随节点数线性增加
     - 中等规模互连
   * - Mesh
     - 高带宽、更多布线、高度模块化，可通过增加行列数扩展
     - 大规模互连

B1.3 Terminology
~~~~~~~~~~~~~~~~~

文档定义的核心术语：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 术语
     - 定义
   * - Message
     - 协议层术语，定义两个组件间交换的粒度（如 Request、Data response、Snoop request）。单个消息可由多个包组成
   * - Packet
     - 互连上的传输粒度。一个消息可由一个或多个包组成。每个包包含路由信息
   * - Flit
     - FLow control unIT，最小流控单元。一个包可由一个或多个 Flit 组成（CHI 中 1 Packet = 1 Flit）
   * - Phit
     - PHysical layer transfer unIT，两个相邻网络设备间的一次传输（CHI 中 1 Flit = 1 Phit）
   * - PoS (Point of Serialization)
     - 互连中确定来自不同 Agent 的请求之间排序关系的点
   * - PoC (Point of Coherence)
     - 所有可访问内存的 Agent 保证看到相同内存位置副本的点。典型实现为互连中的 HN-F
   * - PoE (Point of Encryption)
     - 内存系统中写入到达该点即被加密的位置
   * - PoP (Point of Persistence)
     - 系统断电时写入被维持且可靠恢复的位置（位于 PoC 或更深处）
   * - PoDP (Point of Deep Persistence)
     - 即使电源和备份电池同时故障仍能保存数据的位置
   * - PoPA (Point of Physical Aliasing)
     - 一个 PAS 中的更新对其他所有 PAS 可见的点
   * - Requester
     - 发起事务（发出请求消息）的组件
   * - Completer
     - 响应接收到的来自其他组件的事务的组件（HN、MN、SN 等）
   * - Subordinate / Endpoint
     - 接收事务并适当完成的 Agent，通常是系统中最下游的 Agent
   * - ICN
     - Interconnect，协议节点间通信的 CHI 传输机制
   * - Snoopee
     - 正在接收 Snoop 的 Request Node

**节点类型缩写**:

.. list-table::
   :header-rows: 1
   :widths: 15 15 70

   * - 缩写
     - 全称
     - 描述
   * - RN
     - Request Node
     - 生成协议事务（读/写）到互连
   * - RN-F
     - Fully Coherent RN
     - 包含硬件一致性缓存，可生成所有事务类型，支持所有 Snoop
   * - RN-D
     - IO Coherent RN with DVM
     - 无硬件一致性缓存，接收 DVM 事务，生成事务子集
   * - RN-I
     - IO Coherent RN
     - 无硬件一致性缓存，不接收 DVM，生成事务子集，无需 Snoop
   * - HN
     - Home Node
     - 互连中接收 RN 事务的节点
   * - HN-F
     - Fully Coherent HN
     - 包含 PoC，管理一致性，管理排序（PoS），可包含目录或 Snoop Filter
   * - HN-I
     - Non-coherent HN
     - 处理有限的请求子集，不包含 PoC，处理 IO 请求排序
   * - MN
     - Miscellaneous Node
     - 接收 DVM 事务，完成所需操作并返回响应
   * - SN
     - Subordinate Node
     - 接收来自 HN 的请求并完成
   * - SN-F
     - Subordinate Node (Normal)
     - 用于 Normal memory，处理 Non-snoopable Read/Write/Atomic 和 CMO
   * - SN-I
     - Subordinate Node (I/O)
     - 用于外设或 Normal memory

B1.4 Transaction classification
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

事务按以下类别分类：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 分类
     - 支持的事务
   * - Read
     - ReadNoSnp, ReadNoSnpSep, ReadOnce, ReadOnceCleanInvalid, ReadOnceMakeInvalid, ReadClean, ReadNotSharedDirty, ReadShared, ReadUnique, ReadPreferUnique, MakeReadUnique
   * - Dataless
     - CleanUnique, MakeUnique, Evict, StashOnceUnique/SepUnique/SepShared, CleanShared, CleanSharedPersist/Sep, CleanInvalid, CleanInvalidPoPA, MakeInvalid
   * - Write
     - WriteNoSnpPtl/Full/Zero/Def, WriteUniquePtl/Full/Zero/PtlStash/FullStash, WriteBackPtl/Full, WriteCleanFull, WriteEvictFull/OrEvict
   * - Combined Write
     - WriteNoSnpPtl/Full + CleanInv/CleanSh/CleanShPerSep/CleanInvPoPA 等
   * - Atomic
     - AtomicStore, AtomicLoad, AtomicSwap, AtomicCompare
   * - Other
     - DVMOp, PrefetchTgt, PCrdReturn
   * - Snoop
     - SnpOnceFwd/Once/StashUnique/StashShared, SnpCleanFwd/Clean, SnpNotSharedDirtyFwd/NotSharedDirty, SnpSharedFwd/Shared, SnpUniqueFwd/Unique/UniqueStash, SnpPreferUniqueFwd/PreferUnique, SnpCleanShared/CleanInvalid/MakeInvalid/MakeInvalidStash, SnpQuery, SnpDVMOp

B1.5 Coherence overview
~~~~~~~~~~~~~~~~~~~~~~~~~

硬件一致性使系统组件能够**无需软件缓存维护**即可共享内存。

一致性协议的核心保证：

- 当发生 Store 时，一个数据值最多只存在一个副本
- 所有 Requester 在任意地址位置都能观察到正确的数据值
- 缓存行粒度为 **64 字节对齐的内存区域**
- 主内存不需要在所有时间保持最新状态

B1.5.1 Coherency model
^^^^^^^^^^^^^^^^^^^^^^^

典型一致性系统包含多个 Requester（各带本地缓存）通过互连访问主内存。互连中包含可选的 ICN Cache。

B1.5.2 Cache state model
^^^^^^^^^^^^^^^^^^^^^^^^^

缓存状态基于以下特性组合：

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - 特性维度
     - 说明
   * - Valid / Invalid
     - Valid: 缓存行在缓存中存在; Invalid: 不存在
   * - Unique / Shared
     - Unique: 仅在本缓存中存在; Shared: 可在多个缓存中存在
   * - Clean / Dirty
     - Clean: 无需负责更新主内存; Dirty: 已相对于主内存被修改
   * - Full / Partial / Empty
     - Full: 所有字节有效; Partial: 部分字节有效; Empty: 无有效字节

**七状态缓存模型**:

.. list-table::
   :header-rows: 1
   :widths: 10 20 70

   * - 状态
     - 分类
     - 描述
   * - **I**
     - Invalid
     - 缓存行不在缓存中
   * - **UC**
     - Unique Clean
     - 仅在本缓存中，未修改，可自由修改
   * - **UD**
     - Unique Dirty
     - 仅在本缓存中，已修改，驱逐时必须写回
   * - **SC**
     - Shared Clean
     - 可能有其他共享副本，不能直接修改
   * - **SD**
     - Shared Dirty
     - 可能有其他共享副本，已修改，驱逐时必须写回
   * - **UCE**
     - Unique Clean Empty
     - 仅在本缓存中，无有效字节，不能返回数据
   * - **UDP**
     - Unique Dirty Partial
     - 仅在本缓存中，部分字节有效，已修改

B1.7 Read data source
~~~~~~~~~~~~~~~~~~~~~~

Read 请求可以从以下数据源获取数据：

- 互连中的缓存
- Subordinate Node（主内存）
- Peer RN-F（其他 Requester 的缓存）

减少数据传输跳数的优化技术：

- **DMT (Direct Memory Transfer)**: 允许 SN 直接将数据发送给 Requester
- **DCT (Direct Cache Transfer)**: 允许 Peer RN-F 直接将数据发送给 Requester
- **DWT (Direct Write-data Transfer)**: 写数据的直接传输优化


Chapter B2 -- Transactions
----------------------------

*源文档行号: 1608-8162 (文档最大章节)*

B2.1 Channels overview
~~~~~~~~~~~~~~~~~~~~~~~

节点间基于**通道**进行通信。CHI 定义了 5 种通道类型：

.. list-table::
   :header-rows: 1
   :widths: 10 15 25 25 25

   * - 通道
     - 方向
     - RN 通道名
     - SN 通道名
     - 用途
   * - REQ
     - 出站请求
     - TXREQ
     - RXREQ
     - 请求消息（Read/Write 请求）
   * - WDAT
     - 出站数据
     - TXDAT
     - RXDAT
     - 写数据、原子数据
   * - SRSP
     - 出站响应
     - TXRSP
     | -
     | Snoop 响应、完成确认
   * - CRSP
     - 入站响应
     - RXRSP
     | TXRSP
     | Completer 的响应
   * - RDAT
     - 入站数据
     | RXDAT
     | TXDAT
     | 读数据、原子数据
   * - SNP
     - 入站 Snoop
     | RXSNP
     | -
     | Snoop 请求

B2.2 Channel fields
~~~~~~~~~~~~~~~~~~~~

B2.2.1 Transaction request fields
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

请求通道字段中，**影响事务结构**的字段：

- **Opcode**: 请求操作码，指定事务类型，是决定事务结构的主要字段
- **Size**: 数据大小，决定事务中的数据包数量
- **AllowRetry**: 是否允许目标返回 Retry 响应
- **Order**: 排序要求，决定同一 Agent 的请求间排序
- **DoDWT**: 是否执行直接写传输
- **CAH (CopyAtHome)**: CopyBack 请求中指示 Home 是否保留行的副本
- **ExpCompAck**: 事务是否包含完成确认消息
- **TagOp**: Tag 操作类型

其他关键字段（不影响结构）：QoS, TgtID, SrcID, TxnID, ReturnNID, StashNID, Endian, Deep, Addr, NS/NSE, MemAttr, SnpAttr, LPID, Excl, SnoopMe, TraceTag, MPAM, PBHA, MECID, StreamID 等。

B2.2.3 Snoop request fields
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Snoop 请求字段与请求通道字段有很多重叠，额外的字段包括：

- **FwdNID**: 原始 Requester 的节点 ID
- **FwdTxnID**: 原始 Requester 的事务 ID
- **DoNotGoToSD**: 控制 Snoopee 对 SD 状态的使用
- **RetToSrc**: 指示 Snoop 接收方在 Snoop 响应中返回数据
- **VMIDExt**: 虚拟机 ID 扩展（用于 DVM）

.. note::

   Snoop 请求**不定义 TgtID 字段**。Snoop 请求的路由机制是 IMPLEMENTATION DEFINED 的。

B2.2.4 Data fields
^^^^^^^^^^^^^^^^^^^

数据包字段包括：QoS, TgtID, SrcID, TxnID, HomeNID, Opcode, Resp, RespErr, DataSource, FwdState, DataPull, CBusy, MECID, DBID, CCID, DataID, TagOp, Tag, TU, TraceTag, CAH, NumDat, Replicate, BE, Data, DataCheck, Poison。

B2.3 Transaction structure
~~~~~~~~~~~~~~~~~~~~~~~~~~~

描述事务完成的各种方式，包括所有事务类型的允许选项。

事务类型覆盖：

- B2.3.1 Read transactions（Allocating Read / Non-allocating Read）
- B2.3.2 Write transactions（Immediate Write / CopyBack Write / Combined Write）
- B2.3.3 Atomic transactions（AtomicStore / AtomicLoad / AtomicSwap / AtomicCompare）
- B2.3.4 Stash transactions
- B2.3.5 Dataless transactions
- B2.3.6 Prefetch transactions
- B2.3.7 Retry
- B2.3.8 Home Initiated transactions（Home to Subordinate / Home to Snoopee）

B2.4 Transaction identifier fields
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

CHI 定义了 **15 种事务标识符字段**：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 标识符
     - 用途
   * - **TgtID** / **SrcID**
     - 目标/源节点标识符，用于包路由
   * - **TxnID**
     - 事务标识符，每个源节点内唯一
   * - **DBID**
     - 数据缓冲标识符，用作数据消息响应中的 TxnID
   * - **ReturnTxnID**
     - 返回事务标识符，用于 SN 到 Requester 的数据响应
   * - **FwdTxnID**
     - 转发事务标识符，Snoop 中携带的原始请求 TxnID
   * - **DataID** / **CCID**
     - 数据标识符 / 关键块标识符，用于多数据包传输
   * - **LPID**
     - 逻辑处理器标识符
   * - **StashLPID**
     - Stash 目标的逻辑处理器标识符
   * - **StashNID**
     - Stash 目标的节点标识符
   * - **ReturnNID**
     - 数据/持久化/Tag 匹配响应的接收者节点 ID
   * - **HomeNID**
     - Home 节点标识符，用于 CompAck 响应路由
   * - **FwdNID**
     - 转发节点标识符，Snoop 中携带的原始请求者 ID
   * - **PGroupID**
     - 持久化组标识符
   * - **StashGroupID**
     - Stash 组标识符
   * - **TagGroupID**
     - Tag 组标识符

B2.5 Transaction identifier field flows
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

描述不同事务中 ID 值的传递方式，包括：

- DMT（Direct Memory Transfer）中的 ID 传递
- DCT（Direct Cache Transfer）中的 ID 传递
- 无直接数据传输的 ID 传递
- Dataless 事务的 ID 传递
- DVMOp 事务的 ID 传递
- Retry 请求的 ID 传递
- Protocol Credit Return 事务

B2.6 Ordering
~~~~~~~~~~~~~~

定义事务排序规则，包括：

- **Multi-copy atomicity（多拷贝原子性）**: 确保所有 Agent 以相同顺序观察到内存更新
- **Completion response and ordering**: 完成响应的排序语义
- **Completion acknowledgment**: 完成确认机制
- **RespSepData / DataSepResp 排序语义**: 分离的响应和数据
- **Transaction ordering**: 事务排序要求、CopyBack 请求排序、流式有序写事务

B2.7 Address, Control, and Data
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- **B2.7.1 Address**: 地址定义和对齐要求
- **B2.7.2 Physical Address Space (PAS)**: NS 和 NSE 字段组合确定物理地址空间
- **B2.7.3 Memory Attributes**: 包括 EWA（Early Write Acknowledgment）、Cacheable、Allocate 等
- **B2.7.5 Likely Shared**: 下游缓存分配提示
- **B2.7.6 Snoop attribute**: Snoop 属性
- **B2.7.7 Mismatched Memory attributes**: 属性不匹配的处理
- **B2.7.8 CopyAtHome (CAH)**: 在 Home 和 Request Node 的使用

B2.8 Data transfer
~~~~~~~~~~~~~~~~~~~

- **Data size**: 支持的数据大小定义
- **Bytes access in memory**: 内存字节访问规则
- **Byte Enables**: 字节使能，用于写/原子/Snoop 事务
- **Data packetization**: 数据打包方式
- **Limited Data Elision**: 有限数据省略机制
- **Atomic transactions 中的大小/地址/数据对齐**: 包括大小端处理
- **Critical Chunk Identifier (CCID)**: 关键块标识符
- **Data Beat ordering**: 数据拍排序

B2.9 Request Retry
~~~~~~~~~~~~~~~~~~~

- **Credit Return**: 信用返回机制
- **Transaction Retry mechanism**: AllowRetry 字段、PCRdType 等重试机制


Chapter B3 -- Network Layer
----------------------------

*源文档行号: 8163-8464*

B3.1 System Address Map (SAM)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Requester 必须拥有 **System Address Map (SAM)** 来确定请求的 TgtID。SAM 的具体格式和结构是 **IMPLEMENTATION DEFINED** 的。

要求：
- SAM 必须对整个地址空间提供完整解码
- 不对应物理组件的地址应发送到能提供适当错误响应的 Agent

B3.2 Node ID
~~~~~~~~~~~~~

每个连接到互连 Port 的组件分配一个 **Node ID**：
- 变量宽度：**7 位到 11 位**
- 一个 Port 可分配多个 Node ID
- 一个 Node ID 值只能分配给一个 Port
- Node ID 的定义和分配是 IMPLEMENTATION DEFINED 的

B3.3 TgtID determination
~~~~~~~~~~~~~~~~~~~~~~~~~

不同消息类型的 TgtID 确定方式：

- **Request 消息**: 由 Opcode（DVMOp）或地址到 Node ID 映射确定；PrefetchTgt 使用不同的映射器（始终指向 SN-F）
- **Response 消息**: TgtID 必须匹配接收到的请求消息中的 SrcID、HomeNID、ReturnNID 或 FwdNID
- **Snoop 消息**: 不包含 TgtID，路由机制是 IMPLEMENTATION DEFINED 的

B3.4 Network layer flow examples
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

给出了三种网络层事务流示例：

- **Simple flow**: 无重映射的 TgtID 分配
- **Flow with interconnect-based SAM**: 互连中重映射 TgtID
- **Flow with interconnect-based SAM and Retry request**: 带重映射和 Retry 的流


Chapter B4 -- Coherence Protocol
---------------------------------

*源文档行号: 8465-11909 (第二大章节)*

B4.1 Cache line states
~~~~~~~~~~~~~~~~~~~~~~

协议定义了 7 种缓存行状态（详见 B1.5.2 节的缓存状态模型）。每种状态定义了对 Snoop 的响应行为：

- **UCE**: 不能返回或转发数据
- **UDP**: 必须返回 Home，不能转发
- **UD**: 必须返回，期望转发
- **UC**: 允许（但不要求）返回和转发
- **SD**: 必须返回，期望转发
- **SC**: RetToSrc 未设置时不能返回数据；设置时期望返回和转发
- **I**: 不在缓存中

B4.2 Request types
~~~~~~~~~~~~~~~~~~

协议请求分为以下类别：

**Read 事务**:
- ReadNoSnp / ReadNoSnpSep: Non-snoopable 读取
- ReadOnce / ReadOnceCleanInvalid / ReadOnceMakeInvalid: Snoopable 快照读取
- ReadClean: 获取 Clean 副本（适用于不支持 Dirty 的缓存如 I-Cache）
- ReadNotSharedDirty: 不接受 SD 状态的加载
- ReadShared: 可接受 SD 状态的加载
- ReadUnique: Store 前获取 Unique 副本
- ReadPreferUnique: 优先但非必须获取 Unique 副本
- MakeReadUnique: 获取 Unique 权限（数据响应可选）

**Dataless 事务**:
- CleanUnique / MakeUnique: 获取/使 Unique 权限
- Evict: 驱逐缓存行
- CleanShared / CleanSharedPersist / CleanSharedPersistSep: 共享缓存维护
- CleanInvalid / CleanInvalidPoPA: 无效化缓存维护
- MakeInvalid: 使无效化
- StashOnce* 系列: 独立 Stash 请求

**Write 事务**:
- WriteNoSnpPtl/Full/Zero/Def: Non-snoopable 写入
- WriteUniquePtl/Full/Zero/PtlStash/FullStash: Unique 写入
- WriteBackPtl/Full: CopyBack 写回
- WriteCleanFull: Clean 写入完整行
- WriteEvictFull/OrEvict: 驱逐写入

**Combined Write 事务**:
将写入与缓存维护操作组合（如 WriteNoSnpPtlCleanInv、WriteUniqueFullCleanSh 等）。

**Atomic 事务**:
- AtomicStore: 原子存储
- AtomicLoad: 原子加载
- AtomicSwap: 原子交换
- AtomicCompare: 原子比较

**DVM 事务**:
- DVMOp: 分布式虚拟内存操作

**Prefetch 事务**:
- PrefetchTgt: 预取到目标

B4.3 Snoop request types
~~~~~~~~~~~~~~~~~~~~~~~~~

CHI 定义了丰富的 Snoop 类型：

.. list-table::
   :header-rows: 1
   :widths: 35 65

   * - Snoop 类型
     - 描述
   * - SnpOnceFwd / SnpOnce
     - 基础 Snoop，带/不带数据转发
   * - SnpStashUnique / SnpStashShared
     - Stash Snoop，数据 Stash 到特定位置
   * - SnpCleanFwd / SnpClean
     - 要求 Clean 状态的 Snoop
   * - SnpNotSharedDirtyFwd / SnpNotSharedDirty
     - 禁止 SD 状态的 Snoop
   * - SnpSharedFwd / SnpShared
     - 要求 Shared 状态的 Snoop
   * - SnpUniqueFwd / SnpUnique
     - 要求 Unique 状态的 Snoop
   * - SnpPreferUniqueFwd / SnpPreferUnique
     - 优先 Unique 但非必须
   * - SnpUniqueStash
     - Unique + Stash Snoop
   * - SnpCleanShared / SnpCleanInvalid
     - Clean 共享/无效化 Snoop
   * - SnpMakeInvalid / SnpMakeInvalidStash
     - 使无效化 Snoop
   * - SnpQuery
     - 查询 Snoop（不改变状态）
   * - SnpDVMOp
     | DVM Snoop

B4.4 Request/Snoop correspondence
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

描述请求事务与对应 Snoop 请求的关系：
- 发送 Snoop 的数量
- Snoop 类型的选择规则

B4.5 Response types
~~~~~~~~~~~~~~~~~~~~

- **Completion response**: 完成响应（Read/Atomic/Dataless/Write 各有不同规则）
- **Snoop response**: Snoop 响应（CompData, Comp, CompDBIDResp 等）

B4.6 Silent cache state transitions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

静默缓存状态转换 -- 不涉及数据传输的状态变化。

B4.7 Cache state transitions at a Requester
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

定义 Requester 端的缓存状态转换表。

B4.8 Cache state transitions at a Snoopee
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

定义 Snoopee 端的缓存状态转换表。

B4.9 Returning Data with Snoop response
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

描述在 Snoop 响应中返回数据的规则，包括 **RetToSrc** 字段的作用。

B4.10 Do not transition to SD
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**DoNotGoToSD** 字段控制 Snoopee 是否可以转换到 SD 状态。

B4.11 Hazard conditions
~~~~~~~~~~~~~~~~~~~~~~~~

描述可能导致危险的缓存状态条件。


Chapter B5 -- Interconnect Protocol Flows
------------------------------------------

*源文档行号: 11910-13343*

提供各类事务的**互连协议流程**示例：

- **B5.1 Read transaction flows**: 各种 Read 事务的完整流程（含/不含 DMT/DCT、含 Snoop、Dataless Read 等）
- **B5.2 Dataless transaction flows**: CleanUnique、Evict、CleanShared、MakeInvalid 等流程
- **B5.3 Write transaction flows**: WriteNoSnp、WriteUnique、WriteBack、CopyBack 等流程
- **B5.4 Atomic transaction flows**: AtomicStore、AtomicLoad、AtomicSwap、AtomicCompare 流程
- **B5.5 Stash transaction flows**: Write with Stash hint、Independent Stash 流程
- **B5.6 Hazard handling examples**: 危险处理示例


Chapter B6 -- Exclusive accesses
---------------------------------

*源文档行号: 13344-13702*

- **B6.1 Overview**: 独占访问概述，使用 LDXR/STXR 指令对
- **B6.2 Exclusive monitors**: 独占监视器的地址标记、状态管理
- **B6.3 Exclusive transactions**: 带有 **Excl** 属性的事务处理规则


Chapter B7 -- Cache Stashing
-----------------------------

*源文档行号: 13703-14011*

- **B7.1 Overview**: Stash 机制概述 -- 将数据移动到更接近预期使用点的位置
- **B7.2 Write with Stash hint**: 带有 Stash 提示的写入（WriteUniquePtlStash, WriteUniqueFullStash）
- **B7.3 Independent Stash request**: 独立 Stash 请求（StashOnceUnique, StashOnceShared 等）
- **B7.4 Stash target identifiers**: StashNID、StashLPID 等
- **B7.5 Stash messages**: StashDone、CompStashDone 等


Chapter B8 -- DVM Operations
-----------------------------

*源文档行号: 14012-14920*

- **B8.1 Introduction**: DVM（Distributed Virtual Memory）事务介绍
- **B8.2 DVM transaction flow**: DVM 事务流（通过 MN 广播到所有 RN-D）
- **B8.3 DVMOp field value restrictions**: DVMOp 字段值限制
- **B8.4 DVM messages**: DVM 消息类型和格式


Chapter B9 -- Error Handling
-----------------------------

*源文档行号: 14921-15799*

- **B9.1 Packet level**: 包级错误处理，包括 **RespErr** 字段、错误响应码
- **B9.2 Sub-packet level**: 子包级错误
  - B9.2.1 Poison: 数据毒化标记
  - B9.2.2 Data Check: 数据校验
- **B9.3 Use of interface parity**: 接口奇偶校验的使用
- **B9.4 Hardware and software error categories**: 硬件和软件错误分类


Chapter B10 -- Realm Management Extension
-----------------------------------------

*源文档行号: 15800-16374*

- **B10.1 Introduction**: Realm Management Extension 介绍
- **B10.2 Physical Address Space (PAS)**: PAS 与 NS/NSE 字段的关系
- **B10.3 Cache maintenance**: 跨 PAS 的缓存维护
- **B10.4 DVM**: 跨 PAS 的 DVM
- **B10.5 MPAM**: MPAM 的 RME 扩展
- **B10.6 Memory Encryption Contexts (MEC)**: 内存加密上下文
- **B10.7 Device Assignment (DA) and Coherent Device Assignment (CDA)**: 设备分配


Chapter B11 -- System Control, Debug, Trace, and Monitoring
------------------------------------------------------------

*源文档行号: 16375-17084*

- **B11.1 QoS mechanism**: 16 级 QoS 优先级、端到端 QoS 传播
- **B11.2 Data Source**: 数据源标识（标记数据来源：内存/缓存/Peer 缓存）
- **B11.3 Data Target**: 数据目标（前向放置和使用提示）
- **B11.4 MPAM**: Memory System Resource Partitioning and Monitoring
- **B11.5 Page-based Hardware Attributes (PBHA)**: 来自转换表的 4 位实现定义硬件控制
- **B11.6 Completer Busy**: Completer 忙碌指示
- **B11.7 Trace Tag**: 调试、跟踪和性能测量支持


Chapter B12 -- Memory Tagging
------------------------------

*源文档行号: 17085-17727*

- **B12.1 Introduction**: Arm MTE (Memory Tagging Extension) 介绍
- **B12.2 Message extensions**: 消息中的 Tag 相关字段扩展
- **B12.3 Tag coherency**: Tag 一致性规则
- **B12.4 Read transaction rules**: 读事务的 Tag 规则
- **B12.5 Write transactions**: 写事务的 Tag 规则
- **B12.6-B12.11**: Dataless/Atomic/Stash/Snoop/Home-to-Subordinate/Error 的 Tag 规则
- **B12.12 Requests and permitted tag operations**: 请求与允许的 Tag 操作
- **B12.13 TagOp field use summary**: TagOp 字段使用总结


Chapter B13 -- Link Layer
---------------------------

*源文档行号: 17728-19831*

B13.1 Introduction
~~~~~~~~~~~~~~~~~~~

链路层提供节点与互连之间基于包的通信机制，定义包/Flit 格式和流控。

B13.2 Link
~~~~~~~~~~

Flit 通信在 **Transmitter (TX)** 和 **Receiver (RX)** 对之间进行。双向通信需要一对 Link。

- **Outbound link**: Transmitter 发送包的链路
- **Inbound link**: Receiver 接收包的链路

B13.3 Flit
~~~~~~~~~~

两种 Flit 类型：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 类型
     - 描述
   * - Protocol flit
     - 携带协议包作为有效载荷。每个协议包映射到恰好一个协议 Flit
   * - Link flit
     - 携带链路维护消息，如 L-Credit 返回。在链路 TX 产生，在链路 RX 终止

B13.4 Channel
~~~~~~~~~~~~~

链路层提供 4 种通道（在 Request Node 和 Subordinate Node 上的映射）：

.. list-table::
   :header-rows: 1
   :widths: 10 25 25 40

   * - 通道
     - RN 通道
     - SN 通道
     - 描述
   * - REQ
     - TXREQ / RXREQ
     - RXREQ
     | 请求消息
   * - RSP
     - RXRSP / TXRSP / RSRSP
     - TXRSP
     | 响应消息（无数据负载）
   * - SNP
     | RXSNP
     | -
     | Snoop 请求
   * - DAT
     - TXDAT / RXDAT
     - RXDAT / TXDAT
     | 数据消息（有数据负载）

**通道依赖关系**:

Request Node:
- 必须在入站 SNP 通道上取得进展，不依赖于出站 REQ 通道
- 必须在入站 RSP 和 DAT 通道上取得进展，不依赖于任何其他通道

Subordinate Node:
- 必须在入站 REQ 通道上取得进展，不依赖于出站 DAT 通道
- 必须在入站 DAT 通道上取得进展，不依赖于任何其他通道

B13.5 Port
~~~~~~~~~~

**Port** 定义为节点接口处所有链路的集合。一个 Port 可包含多条链路。

B13.6 Node interface definitions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

各节点类型的接口定义要求（RN-F, RN-D, RN-I, HN-F, HN-I, MN, SN-F, SN-I）。

B13.7 Increasing inter-port bandwidth
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

通过增加链路数来提升端口间带宽的方法。

B13.8 Channel interface signals
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

各通道的物理接口信号定义。

B13.9 Flit packet definitions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

各通道的 Flit 包格式定义。

B13.10 Protocol flit fields
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**协议 Flit 字段详细定义**（共 65 个子节，B13.10.1 到 B13.10.65），包括：

- QoS, TgtID, SrcID, TxnID, ReturnNID, HomeNID 等标识符字段
- Opcode（REQ/RSP/SNP/DAT 通道操作码）
- 数据相关字段（Data, BE, CCID, DataID, NumDat, Replicate）
- 控制字段（AllowRetry, PCrdType, Order, CAH, ExpCompAck, DoDWT）
- 安全字段（NS, NSE, MECID, StreamID, SecSID1）
- 扩展字段（MPAM, PBHA, TagOp, Tag, TU, TraceTag, RSVDC）


Chapter B14 -- Link Handshake
-------------------------------

*源文档行号: 19832-20672*

- **B14.1 Clock and initialization**: 时钟和初始化序列
- **B14.2 Link layer Credit**: L-Credit 机制，Transmitter 在发送 Flit 前必须有足够的 Credit
- **B14.3 Low power signaling**: 低功耗信号（TXLINKACTIVEREQ / TXLINKACTIVEACK）
- **B14.4 Flit level clock gating**: Flit 级时钟门控
- **B14.5 Interface activation and deactivation**: 接口激活/去激活序列（含 Link flit 的 L-Credit 返回）
- **B14.6 Transmit and receive link interaction**: TX 和 RX 链路交互
- **B14.7 Protocol layer activity indication**: 协议层活动指示（TXSACTIVE / RXSACTIVE）


Chapter B15 -- System Coherency Interface
------------------------------------------

*源文档行号: 20673-20803*

- **B15.1 Overview**: 系统一致性接口概述
- **B15.2 Handshake**: 握手协议


Chapter B16 -- Properties, Parameters, and Broadcast Signals
-------------------------------------------------------------

*源文档行号: 20804-21552*

- **B16.1 Interface properties and parameters**: 接口属性和参数（26 个子参数），包括：
  - 数据宽度（DataWidth）
  - 通道宽度
  - MaxTxnID、MaxDBID 等容量参数
  - 数据大小支持
  - 各事务类型的支持标志
  - MTE、RME、MPAM 等扩展支持

- **B16.2 Optional interface broadcast signals**: 可选接口广播信号

- **B16.3 Atomic transaction support**: 原子事务支持参数


Part C -- Appendices
====================

*源文档行号: 21553-22833*

Chapter C1 -- Message Field Mappings
-------------------------------------

*源文档行号: 21557-22153*

各通道消息的**字段映射表**：

- **C1.1 Request message field mappings**: REQ 通道操作码对应的字段映射
- **C1.2 Response message field mappings**: RSP 通道操作码对应的字段映射
- **C1.3 Snoop Request message field mappings**: SNP 通道操作码对应的字段映射
- **C1.4 Data message field mappings**: DAT 通道操作码对应的字段映射

Chapter C2 -- Communicating Nodes
-----------------------------------

*源文档行号: 22154-22265*

定义各消息类型的通信节点：

- **C2.1 Request communicating nodes**: 哪些节点可以发起各种请求
- **C2.2 Snoop communicating nodes**: Snoop 通信节点
- **C2.3 Response communicating nodes**: 响应通信节点
- **C2.4 Data communicating nodes**: 数据通信节点

Chapter C3 -- Revisions
-------------------------

*源文档行号: 22266-22833*

各版本间的变更记录：

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - 版本变更
     - 主要内容
   * - C3.1 Issue A -> B
     - First limited release -> First public release
   * - C3.2 Issue B -> C
     | Second public release 变更
   * - C3.3 Issue C -> D
     | Third public release 变更
   * - C3.4 Issue D -> E.a
     | Fourth public release 变更
   * - C3.5 Issue E.a -> E.b
     | Fifth public release 变更
   * - C3.6 Issue E.b -> E.c
     | Sixth public release 变更
   * - C3.7 Issue E.c -> F
     | Seventh public release 变更
   * - C3.8 Issue F -> F.b
     | Eighth public release 变更
   * - C3.9 Issue F.b -> G
     | Ninth public release (当前版本) 变更


Part D -- Glossary
==================

*源文档行号: 22834-23240*

Chapter D1 -- Glossary
-------------------------

术语表，提供文档中所有专业术语的定义。


文档架构总结
============

CHI 架构采用**三层分离设计**：

1. **Protocol Layer**: 定义事务类型、缓存一致性协议、状态机、事务流
2. **Network Layer**: 定义 Node ID、TgtID 路由、System Address Map
3. **Link Layer**: 定义 Flit 格式、通道、Credit 流控、低功耗

核心协议节点围绕 **Home Node (HN)** 构建，HN 作为事务的协调者和 Point of Coherence (PoC)。

**关键数据通路**:

Requester (RN) --> Request --> Home Node (HN) --> Snoop + Subordinate access --> Response + Data --> Requester (RN)

**性能优化机制**:

- DMT/DCT/DWT 减少数据传输跳数
- Cache Stashing 提前放置数据
- PrefetchTgt 预热内存控制器
- Limited Data Elision 减少数据包数量
- Snoop Filter / Directory 减少 Snoop 范围
- 16 级 QoS 端到端支持
