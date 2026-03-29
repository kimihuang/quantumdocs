==============================================
CoreLink MMU-600AE TCU 与 TBU 架构分析
==============================================

本文档基于 ARM CoreLink MMU-600AE Technical Reference Manual（ARM DEN 0060D）
分析 MMU-600AE 的两大核心组件 TCU（Translation Control Unit）和
TBU（Translation Buffer Unit）的架构、功能、交互机制和关键约束。

.. contents::
   :depth: 3
   :local:

--------------------------------
MMU-600AE 概述
--------------------------------

1 产品定位
--------------------------------

MMU-600AE 是 ARM SMMUv3 架构的硬件实现，"AE" 后缀表示这是功能安全（Functional Safety）
版本。它是 MMU-600 的 FuSa 增强版，在保留原有逻辑和功能不变的基础上，增加了错误检测和
纠正的安全机制。MMU-600AE 以可综合 RTL 形式交付，经过实现（Implementation）、集成
（Integration）和编程（Programming）三个阶段后可用于 SoC 产品。（来源：Section 1.6）

MMU-600AE 遵循的规范：

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - 规范
     - 说明
   * - SMMUv3 / SMMUv3.1
     - ARM 系统内存管理单元架构规范
   * - AMBA DTI Protocol
     - 定义 TCU 与 TBU/PCIe ATS RC 之间的通信标准
   * - AMBA AXI4-Stream
     - DTI 接口的传输层协议
   * - AMBA AXI5 / ACE5-Lite
     - TBU TBS/TBM 接口协议，支持 AXI5 扩展
   * - AMBA APB4
     - TCU 编程接口协议
   * - DVMv8.1
     - 分布式虚拟内存消息协议，支持 16-bit VMID

2 三大核心组件
----------------

MMU-600AE 包含三个核心功能块（来源：Section 2）：

.. list-table::
   :header-rows: 1
   :widths: 15 15 70

   * - 组件
     - 数量
     - 职责
   * - **TBU**
     - ≥1（每个 requester）
     - 本地 TLB 缓存，拦截事务并提供地址翻译
   * - **TCU**
     - 1（单实例）
     - 全局翻译控制：页表遍历、配置缓存、队列管理、SMMU 编程模型
   * - **DTI Interconnect**
     - 1（可配置）
     - 连接多个 TBU 到 TCU，基于 AXI4-Stream 的交换网络

系统拓扑关系（Figure 2-1）::

    Requester ──→ TBU ──→ DTI Interconnect ──→ TCU ──→ Memory System
    PCIe RC (ATS) ──────────→ DTI Interconnect ──→ TCU

每个 TBU 部署在对应的 requester 附近（local placement），通过 DTI Interconnect
连接到中央 TCU。TCU 通过 QTW/DVM 接口访问内存系统执行页表遍历。

3 可配置选项
--------------

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 组件
     - 可配置项
   * - TCU
     - 各级 Walk Cache 大小、QTW/DVM 接口数据宽度、同时翻译数（``TCUCFG_XLATE_SLOTS``）、
       PTW Slots（``TCUCFG_PTW_SLOTS``）、TBU 数量（``TCUCFG_NUM_TBU``，最大 62）
   * - TBU
     - Write Data Buffer 深度、各缓存大小、同时翻译数、TBM 接口 outstanding 事务数、
       TBS/TBM 数据/ID/User/StreamID/SubstreamID 宽度、
       MTLB 深度（``TBUCFG_MTLB_DEPTH``）和分区数（``TBUCFG_MTLB_PARTS``）、
       Direct Indexing（``TBUCFG_DIRECT_IDX``）、
       AXI USER 宽度（``TBUCFG_AxUSER_WIDTH``）
   * - DTI Interconnect
     - Switch、Sizer、Register Slice 组件的连接和配置

----------------------------------------
TBU（Translation Buffer Unit）
----------------------------------------

1 功能定位
------------

TBU 是部署在每个 requester 附近的本地翻译缓存单元。它拦截来自 requester 的未翻译事务，
在本地 TLB 中查找翻译。命中则直接翻译输出，未命中则通过 DTI 向 TCU 请求翻译，并将
获取的翻译缓存到本地 TLB 中。（来源：Section 2.1）

核心工作流::

    TBS 接口收到事务
        → Micro TLB 查找
            → 命中：翻译后通过 TBM 接口输出
            → 未命中：MTLB 查找
                → 命中：翻译后通过 TBM 接口输出
                → 未命中：通过 DTI 向 TCU 请求翻译
                    → TCU 返回翻译结果
                        → 缓存到 TLB
                        → 翻译事务通过 TBM 输出
                        → 如果翻译失败：终止事务

2 内部架构
------------

TBU 由以下功能模块组成（来源：Section 2.1, Figure 2-2）：

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - 模块
     - 功能描述
   * - **Completer/Requester 接口**
     - 管理 TBS（输入）和 TBM（输出）接口
   * - **Micro TLB**
     - 全相联 TLB，提供配置缓存和 TLB 功能。替换策略通过 tie-off 选择：
       round-robin 或 Pseudo Least Recently Used（PLRU）
   * - **Main TLB（MTLB）**
     - 可选的 4-way set-associative 缓存，随机替换策略。
       缓存 Stage 1、Stage 2、以及 Stage 1+Stage 2 合并翻译。
       查找流水线化，支持每周期一次查找吞吐
   * - **Translation Manager**
     - 管理进行中的翻译请求。执行 hazard check 减少重复 TCU 请求。
       支持 Hit-Under-Miss（HUM）。不同 AXI ID 的事务可以乱序推进
   * - **Write Data Buffer**
     - 可选，允许不同 AXI ID 的写事务乱序通过 TBU，
       并重新排序数据以匹配下游事务顺序
   * - **Transaction Tracker**
     - 管理未完成的读写事务，允许 invalidation 和 synchronization
       在不阻塞 AXI 接口的情况下执行
   * - **DTI 接口**
     - Requester DTI，使用 AXI4-Stream 与 TCU 通信
   * - **PMU**
     - 计数 TBU 性能相关事件
   * - **Clock/Power Control**
     - Q-Channel 提供的时钟和功耗控制

3 MTLB Direct Indexing 与 Partitioning
-----------------------------------------

TBU Direct Indexing 允许外部（如 requester 或软件）直接控制 MTLB 条目的索引位置，
用于满足实时翻译的可预测性需求。（来源：Section 2.1.1）

启用条件：``TBUCFG_DIRECT_IDX = 1``

当启用 Direct Indexing 或 MTLB Partitioning 时，TBS 接口的 AxUSER 信号宽度扩展，
增加以下字段（Table 2-1）：

.. list-table::
   :header-rows: 1
   :widths: 20 40 40

   * - 字段
     - 宽度
     - 说明
   * - ``mtlbidx``
     - ``log2(TBUCFG_MTLB_DEPTH) - 2``（Direct Indexing 启用时）
     - MTLB 索引
   * - ``mtlbway``
     - 2（Direct Indexing 启用时）
     - MTLB way 选择
   * - ``mtlbpart``
     - ``log2(TBUCFG_MTLB_PARTS)``
     - MTLB 分区
   * - 常规 AxUSER
     - ``TBUCFG_AxUSER_WIDTH``
     - 原始 AxUSER 信号

关键行为：

- **Direct Indexing 启用时**：MTLB 查找和更新使用 ``mtlbidx`` 字段；
  更新使用 ``mtlbway`` 指定的 way；查找同时搜索所有 way
- **MTLB Partitioning**：MTLB 大小乘以 ``TBUCFG_MTLB_PARTS``；
  ``mtlbpart`` 字段定义最高位的索引位
- **Arm 建议**：在启用 Direct Indexing 的 TBU 上禁用 DVM invalidation，
  通过设置 ``TCU_NODE_CTRLn.DIS_DVM = 1`` 实现

4 AXI USER Bits 扩展
----------------------

TBM 接口的 ``aruser_m`` 和 ``awuser_m`` 信号比 ``TBUCFG_AxUSER_WIDTH`` 定义的多 13 位，
这些扩展位携带翻译属性信息。（来源：Section 2.1.2, Table 2-2）

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - 位位置（相对宽度 w）
     - 含义
   * - ``[w+12]``
     - Outer Cacheable 属性
   * - ``[w+11:w+8]``
     - STE 定义的属性（来自 STE bits[119:116]，
       通过 ``DTI_TBU_TRANS_RESP.CTXTATTR`` 传递）
   * - ``[w+7:w+4]``
     - Stage 2 IMPLEMENTATION DEFINED 硬件属性（``S2HWATTR[3:0]``）
   * - ``[w+3:w]``
     - Stage 1 IMPLEMENTATION DEFINED 硬件属性（``S1HWATTR[3:0]``）

S1HWATTR 和 S2HWATTR 的计算规则：

- ``S1HWATTR[n]`` 等于 Stage 1 最终级描述符的 bit[n+59]，条件是 SMMUv3 允许
  该 bit 有 IMPLEMENTATION DEFINED 硬件用途，且 Stage 2 不允许同一 bit 有此用途
- ``S2HWATTR[n]`` 等于 Stage 2 最终级描述符的 bit[n+59]，条件是 SMMUv3 允许
  该 bit 有 IMPLEMENTATION DEFINED 硬件用途
- Arm 建议系统始终使用 ``S1HWATTR[n] | S2HWATTR[n]`` 的值，即优先取 Stage 2 的值

5 TBU 接口
------------

每个 TBU 包含以下接口（来源：Section 2.4.2, Figure 2-5）：

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - 接口
     - 方向
     - 说明
   * - **TBS**
     - Completer
     - ACE5-Lite 接收端，接收未翻译事务。64-bit 地址。
       支持 AXI5 扩展（Wakeup、Untranslated、Cache_Stash、DeAllocation）和 ACE Exclusive
   * - **TBM**
     - Requester
     - ACE5-Lite 发送端，输出已翻译事务
   * - **DTI**
     - 双向
     - Requester DTI，使用 DTI-TBU 协议与 TCU 通信。
       包含 AXI4-Stream requester + completer 实现双向通信
   * - **LPI_PD**
     - Completer
     - Q-Channel 接口，管理 TBU 低功耗关断
   * - **LPI_CG**
     - Completer
     - Q-Channel 接口，管理 TBU 时钟门控
   * - **Interrupt**
     - 输出
     - 全局中断、per-context 中断、性能中断
   * - **Tie-off**
     - 输入
     - 复位时初始化操作参数

6 TBU 寄存器
---------------

TBU 的微架构寄存器和 RAS 寄存器（来源：Section 3.7-3.9）：

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - 寄存器
     - 偏移
     - 说明
   * - **TBU_CTRL**
     - 0x08E00
     - ``AUX[15:0]``：调试/测试位，必须保持为 0
   * - **TBU_SCR**
     - 0x08E18
     - 安全控制：``NS_RAS``（bit 1）、``NS_UARCH``（bit 0）
   * - **TBU_ERRFR**
     | 0x08E80
     | 错误特性：``FI=0b10``（可控 FHI）、``ED=0b01``（始终使能）
   * - **TBU_ERRCTLR**
     | 0x08E88
     | 错误控制：``FI``（bit 3）使能故障处理中断
   * - **TBU_ERRSTATUS**
     | 0x08E90
     | 错误综合：``V``（有效）、``OF``（溢出）、``CE``（可纠正错误）等
   * - **TBU_ERRGEN**
     | 0x08EC0
     | 错误注入：``TCC``、``DCC``、``TWC``、``DWC`` 用于奇偶校验错误注入

TBU 的 PROG 接口地址宽度取决于 TCU 配置：
``TCUCFG_NUM_TBU = 14`` 时为 21 位，``TCUCFG_NUM_TBU = 62`` 时为 23 位。

----------------------------------------
TCU（Translation Control Unit）
----------------------------------------

1 功能定位
------------

TCU 是 MMU-600AE 的全局翻译控制中心，系统中只有单实例。它负责管理所有翻译操作的核心逻辑。
软件通过以下方式控制 TCU（来源：Section 2.2）：

- 向内存中的 Command Queue 写入命令
- 从内存中的 Event Queue 接收事件
- 通过 PROG 接口（APB4）访问配置寄存器

TCU 的五项核心职责：

1. **管理内存队列** — Command Queue 和 Event Queue
2. **执行翻译表遍历（Translation Table Walk）** — 获取翻译信息
3. **执行配置表遍历（Configuration Table Walk）** — 获取 STE/CD 配置信息
4. **实现备份缓存结构** — Walk Cache 和 Configuration Cache
5. **实现 SMMU 编程模型** — SMMUv3 架构定义的寄存器

2 内部架构
------------

TCU 由以下功能模块组成（来源：Section 2.2, Figure 2-3）：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 模块
     - 功能描述
   * - **Walk Caches**
     - 8 个独立的 4-way set-associative 缓存，分别缓存 Stage 1/Stage 2 各级页表条目。
       round-robin 替换策略。通过 ``TCU_CTRL`` 寄存器按级启用/禁用
   * - **Configuration Cache**
     - 4-way set-associative 缓存，存储翻译上下文的 CD 和 STE 内容。
       不缓存中间配置表
   * - **Translation Manager**
     - 管理进行中的翻译请求。对所有页表遍历和配置表遍历执行 hazard check，
       减少重复遍历
   * - **Translation Request Buffer**
     - 当 Translation Manager 满时暂存来自 TBU 的翻译请求。
       容量大于 Translation Manager，正确配置时能容纳所有 TBU 同时发出的请求，
       防止 DTI 接口阻塞
   * - **Queue Manager**
     - 管理存储在内存中的 SMMUv3 Command Queue 和 Event Queue
   * - **QTW/DVM 接口**
     - ACE-Lite+DVM requester 接口，用于页表遍历和 DVM 消息
   * - **Register File**
     - 实现 SMMUv3 编程模型
   * - **DTI 接口**
     | Completer DTI，使用 AXI4-Stream 与 TBU 或 PCIe ATS RC 通信
   * - **PMU**
     | 计数 TCU 性能相关事件
   * - **Clock/Power Control**
     | Q-Channel 时钟和功耗控制

3 Walk Cache 详解
--------------------

Walk Cache 是 TCU 的核心缓存，用于减少需要实际执行的页表遍历次数。
TCU 包含 8 个独立的 Walk Cache，分别对应 Stage 1 和 Stage 2 的 4 个页表级别。（来源：Section 2.2）

.. list-table::
   :header-rows: 1
   :widths: 15 30 55

   * - 缓存名
     - 缓存内容
     - 说明
   * - S1L0
     | Stage 1 Level 0 表条目
     | 存储阶段 1 顶级页表描述符
   * - S1L1
     | Stage 1 Level 1 表和块条目
     | 存储 L1 级页表描述符和块描述符
   * - S1L2
     | Stage 1 Level 2 表和块条目
     | 存储 L2 级页表描述符和块描述符
   * - S1L3
     | Stage 1 Level 3 表条目
     | 存储 L3 级最终页表描述符
   * - S2L0
     | Stage 2 Level 0 表条目
     | 存储 L0 级阶段 2 页表描述符
   * - S2L1
     | Stage 2 Level 1 表和块条目
   * - S2L2
     | Stage 2 Level 2 表和块条目
   * - S2L3
     | Stage 2 Level 3 表条目

Walk Cache 的关键特性：

- **组织方式**：每个缓存 4-way set-associative，round-robin 替换
- **启用控制**：通过 ``TCU_CTRL`` 寄存器的 ``WCS1L0_DIS`` 到 ``WCS2L3_DIS`` 位
  按级独立启用/禁用
- **错误报告**：缓存行错误通过 ``TCU_ERRSTATUS`` 寄存器标识
- **性能优势**：当其他 TCU 缓存未命中时，Walk Cache 命中只需一次内存访问
  即可获取所需描述符并完成翻译

4 Configuration Cache
----------------------

Configuration Cache 存储 Context Descriptor（CD）和 Stream Table Entry（STE）的完整内容，
用于加速翻译上下文的配置查找。（来源：Section 2.2）

关键特性：

- **组织方式**：4-way set-associative 缓存
- **存储内容**：每个条目存储一个翻译上下文的 CD 和 STE 内容
- **不缓存中间配置表**：只缓存最终级的 CD 和 STE，不缓存中间级别的配置表条目

5 DVM 消息处理
-----------------

TCU 的 QTW/DVM 接口支持 DVMv8.1 分布式虚拟内存消息。（来源：Section 2.2.1）

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - 特性
     - 说明
   * - DVM 版本
     | DVMv8.1，支持 16-bit VMID
   * - 支持的消息类型
     | TLB Invalidate、Synchronization
     | （其他 DVM 消息类型被接受但忽略）
   * - Broadcast TLB Maintenance
     | 通过 ``sup_btm`` 信号使能
   * - PTM 控制（Non-secure）
     | ``SMMU_CR2.PTM = 0``：TLB Invalidate 应用于 Non-secure TLB；
       ``PTM = 1``：无效（默认，当 ``sup_btm`` 为 HIGH 时复位值为 1）
   * - PTM 控制（Secure）
     | ``SMMU_S_CR2.PTM = 0``：TLB Invalidate 应用于 Secure TLB；
       ``PTM = 1``：无效
   * - 无 TLB Invalidate 时的 Sync
     | 如果自上次 DVM Sync 以来没有 TLB Invalidate 操作，
       后续 Sync 立即返回 DVM Complete，不影响系统 DVM 性能
   * - DVM Complete 属性
     | ``ARPROT = 0b000``（Unprivileged, Secure, Data），
       ``ARDOMAIN = 0b01``（Inner Shareable）

6 TCU 事务处理
-----------------

TCU 通过 QTW/DVM 接口执行的事务类型和约束（来源：Section 2.2.2）：

**读事务**（Table 2-3）：

.. list-table::
   :header-rows: 1
   :widths: 30 20 50

   * - 事务类型
     - 数据宽度
     - ARID 说明
   * - Stage 1 Stream 表查找
     | 64-bit
     | ``ARID[n:1]`` = PTW slot 编号，``ARID[0]`` = 1
   * - Stream 表查找
     | 256-bit
     | 同上
   * - 翻译表查找
     | 64-bit
     | 同上
   * - Command Queue 读取
     | 128-bit
     | ARID 全 0
   * - DVM Complete
     | 全宽
     | ARID 全 1

最大并发读事务数：``TCUCFG_PTW_SLOTS + 2``（Command Queue 读取和 DVM Complete 独立于翻译 slot）。

**写事务**（Table 2-4）：

.. list-table::
   :header-rows: 1
   :widths: 30 20 50

   * - 事务类型
     - 数据宽度
     - AWID
   * - Event Queue 写入
     | 256-bit
     | 0
   * - PRI Queue 写入
     | 128-bit
     | 0
   * - MSI（Message Signaled Interrupt）
     | 32-bit
     | 0

写事务同一时刻只能有一个 outstanding。所有读写事务按事务大小对齐。

7 TCU Prefetch
----------------

TCU 支持按翻译上下文（per-context）的翻译预取功能，用于优化线性访问模式的实时 requester
的翻译性能。（来源：Section 2.2.3）

**使能方式**：通过 STE 的 IMPLEMENTATION DEFINED bits[121:120]（``STE.PF`` 字段）：

.. list-table::
   :header-rows: 1
   :widths: 15 85

   * - PF 值
     - 行为
   * - ``0b00``
     | 禁用预取
   * - ``0b01``
     | 保留
   * - ``0b10``
     | 前向预取（Forward）：预取翻译范围结束后的第一个地址
   * - ``0b11``
     | 后向预取（Backward）：预取翻译范围开始前的最后一个地址

预取的工作流程：

1. TBU 的 Micro TLB 和 MTLB 都未命中，发送翻译请求给 TCU
2. TCU 完成原始翻译请求后，检查 ``STE.PF`` 字段
3. 如果预取使能，TCU 在同一翻译 slot 中发起预取请求（Speculative）
4. 预取只分配到 TCU Walk Cache，**不返回翻译响应给 TBU**

预取的限制条件：

- 预取使用与原始请求相同的翻译 slot，必须等原始请求完成后才能发起
- 如果 TCU 在预取完成前收到对同一地址的非推测请求，可以在其他 slot 中优先处理
- 预取只在两次翻译请求之间的间隔大于预取所需周期数时才有性能优势
- 以下情况**不触发预取**：

  - 推测性翻译请求（``PERM[1:0] = 2'b11``）
  - 提供数据响应的原子事务（``PERM[1:0] = 2'b10``）
  - 原始请求返回故障响应、全局 bypass 响应或流 bypass 响应

8 TCU 接口
------------

TCU 包含以下接口（来源：Section 2.4.1, Figure 2-4）：

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - 接口
     - 方向/类型
     - 说明
   * - **QTW/DVM**
     | Requester
     | ACE-Lite+DVM requester 接口。事务类型：
       ReadNoSnoop、WriteNoSnoop、ReadOnce、WriteUnique、DVM Complete。
       不发起 cache maintenance 或 exclusive access
   * - **PROG**
     | Completer
     | AMBA APB4 completer 接口，用于软件编程内部寄存器和读取 PMU/Debug 寄存器。
       地址宽度：``TCUCFG_NUM_TBU = 14`` 时 21 位，
       ``TCUCFG_NUM_TBU = 62`` 时 23 位
   * - **DTI**
     | 双向
     | Completer DTI，使用 AXI4-Stream（含 AXI5 Wakeup_Signal）与 TBU 或 PCIe ATS RC 通信
   * - **SYSCO**
     | Requester
     | 系统一致性接口，允许 TCU 在 LPI 请求时从一致性域中移除自身。
       使用 ``syscoreq``/``syscoack`` 握手。
       当 ``sup_btm`` 为 LOW 时，``syscoreq`` 始终为 LOW
   * - **LPI_PD**
     | Completer
     | Q-Channel，管理 TCU 低功耗关断
   * - **LPI_CG**
     | Completer
     | Q-Channel，管理 TCU 时钟门控
   * - **Interrupt**
     | 输出
     | 全局中断、per-context 中断、性能中断
   * - **Tie-off**
     | 输入
     | 复位时初始化 ``SMMU_IDR0`` 的各 bit

9 TCU 寄存器
---------------

TCU 的微架构寄存器（来源：Section 3.4-3.6）：

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - 寄存器
     - 偏移
     - 说明
   * - **TCU_CTRL**
     | 0x08E00
     | Walk Cache 控制：``WCS1L0_DIS`` 到 ``WCS2L3_DIS``（按级禁用），
       ``ASID_HASH_EN``（bit 19，ASID/VMID 哈希使能）
   * - **TCU_QOS**
     | 0x08E04
     | QoS 控制：``QOS_PTW0-3``（4 个优先级对应的 AxQOS 值），
       以及 DVM Sync、MSI、Queue 访问的 AxQOS
   * - **TCU_CFG**
     | 0x08E08
     | ``XLATE_SLOTS[15:4]``：共享翻译 slot 数量（调试）
   * - **TCU_STATUS**
     | 0x08E10
     | ``GNT_XLATE_SLOTS[15:4]``：当前已分配的翻译 slot 数（调试）
   * - **TCU_SCR**
     | 0x08E18
     | 安全控制：``NS_INIT``、``NS_RAS``、``NS_UARCH``
   * - **TCU_NODE_CTRLn**
     | 0x09000 + 4n
     | 每个 TBU 节点的控制：``DIS_DVM``（bit 4，禁用 DVM）、
       ``PRI_LEVEL[1:0]``（bits[1:0]，4 级优先级）
   * - **TCU_NODE_STATUSn**
     | 0x09004 + 4n
     | 每个 TBU 节点的状态：``ATS``（bit 1，ATS 就绪）、
       ``CONNECTED``（bit 0，已连接）
   * - **TCU_ERRFR**
     | 0x08E80
     | 错误特性：``FI=0b10``（可控 FHI）、``ED=0b01``（始终使能）
   * - **TCU_ERRCTLR**
     | 0x08E88
     | 错误控制：``FI``（bit 3）使能故障处理中断
   * - **TCU_ERRSTATUS**
     | 0x08E90
     | 错误综合：``V``（有效）、``OF``（溢出）、``CE``（可纠正错误）、
       ``IERR``/``SERR`` 字段
   * - **TCU_ERRGEN**
     | 0x08EC0
     | 错误注入：``TCC``、``DCC``、``TWC``、``DWC`` 用于奇偶校验错误注入

----------------------------------------
TCU 与 TBU 的交互
----------------------------------------

1 DTI Interconnect
---------------------

DTI Interconnect 连接多个 TBU 到 TCU，使用 AXI4-Stream 传输协议承载 AMBA DTI 协议。
它包含三个可组合的内部组件（来源：Section 2.3）：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 组件
     - 功能描述
   * - **Switch**
     | 交换网络，连接多个 DTI requester（如 TBU）到一个 DTI completer（如 TCU）。
       TBU→TCU 方向：多个 AXI4-Stream completer 接口连接到一个 AXI4-Stream requester 接口。
       TCU→TBU 方向：一个 AXI4-Stream completer 接口连接到多个 AXI4-Stream requester 接口。
       不存储数据，不需要 Q-Channel
   * - **Sizer**
     | 连接不同数据宽度的通道，支持 1、4、10、20 字节之间的转换。
       包含 Q-Channel 时钟门控接口
   * - **Register Slice**
     | 改善时序，包含 Q-Channel 时钟门控接口

DTI Interconnect 不包含跨时钟/功耗域组件。如需连接不同时钟域，
需使用 ADB-400 AMBA Domain Bridge 的 Bidirectional AXI4-Stream（BAS）配置。

2 DTI 协议
--------------

AMBA DTI 协议定义了 TCU 与外部组件的通信标准，包含两个子协议（来源：Section 2.5）：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 子协议
     - Requester
     - Completer
   * - **DTI-TBU**
     | TBU
     | TCU
   * - **DTI-ATS**
     | PCIe Root Complex
     | TCU

DTI 消息组：

- Page request
- Register access
- Translation request
- Connection and disconnection
- Invalidation and synchronization

**Token 系统**：

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Token
     | 说明
   * - ``max_tok_trans``
     | TBU 在连接时请求的翻译 token 数量，通过 ``DTI_TBU_CONDIS_REQ`` 消息指定
   * - ``TOK_TRANS_REQ``
     | TBU 请求翻译 token 的字段
   * - ``TOK_INV_GNT``
     | TBU 授予的无效化 token，只有 1 个；
       TCU 同一时刻只能发出一个 invalidate 消息

**连接/断开握手**：

- TBU 通过 ``DTI_TBU_CONDIS_REQ`` 消息发起连接或断开握手
- 如果 TBU 提供的 TID 值超过 ``TCUCFG_NUM_TBU`` 定义的最大值，
  TCU 返回 Connect Deny 消息

3 StreamID 约束
------------------

StreamID ≥ 224 的处理（来源：Section 2.5）：

- 发送给 TCU 的翻译请求中 StreamID ≥ 224：产生故障和 ``C_BAD_STREAMID`` 事件
- TBU 收到的无效化请求中 StreamID ≥ 224：任何 StreamID 比较都会失败，
  不会无效化 TLB 条目，但不考虑 StreamID 的其他效果正常发生
- TBU 不会生成 StreamID ≥ 224 的翻译请求
- TCU 不会生成 StreamID ≥ 224 的无效化请求

4 翻译请求完整流程
---------------------

从 requester 发起事务到完成翻译的完整流程：

::

    ① Requester 发起事务 → TBS 接口
    ② TBU Micro TLB 查找
       ├─ 命中 → ⑧ 翻译输出
       └─ 未命中 → ③ TBU MTLB 查找
          ├─ 命中 → ⑧ 翻译输出
          └─ 未命中 → ④ TBU 通过 DTI 向 TCU 发送翻译请求
             ⑤ TCU Configuration Cache 查找（STE/CD）
                ├─ 命中 → ⑥ TCU Walk Cache 查找
                └─ 未命中 → TCU 通过 QTW 执行配置表遍历 → 更新 Configuration Cache
             ⑥ TCU Walk Cache 查找（S1L0→L1→L2→L3, S2L0→L1→L2→L3）
                ├─ 命中 → ⑦ TCU 返回翻译结果给 TBU
                └─ 未命中 → TCU 通过 QTW 执行翻译表遍历 → 更新 Walk Cache → ⑦
             ⑦ TBU 将翻译缓存到 TLB
    ⑧ TBU 使用翻译结果翻译事务 → TBM 接口输出
       （如果翻译失败：TBU 终止事务）

----------------------------------------
QoS 与 RAS
----------------------------------------

1 Quality of Service
----------------------

MMU-600AE 的 QoS 机制基于 TCU 侧的 per-TBU 优先级控制（来源：Section 2.8）：

- 每个连接的 TBU 通过 **``TCU_NODE_CTRLn.PRI_LEVEL[1:0]``** 设置优先级，
  支持 4 个级别（0-3）
- TCU 使用优先级进行以下仲裁：

  - Translation Request Buffer 中的请求仲裁
  - Walk Cache / Translation Table Walk 访问仲裁
  - QTW/DVM 接口上的 AxQOS 值设置（通过 ``TCU_QOS.QOS_PTW0-3`` 映射）

- TBU 本身**不实现**事务间的优先级区分
- Arm 建议：对有不同 QoS 需求的 requester 使用独立的 TBU

2 RAS（Reliability, Availability, Serviceability）
----------------------------------------------------

MMU-600AE 的功能安全特性（来源：Section 2.7, Section 4.8）：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 特性
     | 说明
   * - 错误检测
     | TCU 和 TBU 均有独立的 RAS 寄存器组（ERRFR、ERRCTLR、ERRSTATUS、ERRGEN）
   * - FHI（Fault Handling Interrupt）
     | 通过 ``ERRCTLR.FI``（bit 3）使能，TCU 和 TBU 均支持
   * - ERI（Error Recovery Interrupt）
     | 由 FMU（Fault Management Unit）管理
   * - DTI 接口保护
     | 两种方案：
       1. **复制 DTI AXI4-Stream 接口** — TCU 和 TBU 有双端口，通过复制进行互连保护
       2. **单 DTI 接口 + AMBA Parity** — 点对点 AMBA parity 保护
   * - PMU 错误事件
     | TCU：Walk Cache 各级错误事件（0xC0-0xC8）、Configuration Cache 错误事件
     | TBU：Main TLB 错误事件（0xC0）

----------------------------------------
关键约束和限制
----------------------------------------

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 约束
     | 详细说明
   * - PA 最大 48-bit
     | TBU 和 TCU 最大支持 48-bit PA。``sup_oas[2:0]`` 不能设为 0b110（52-bit），
       如果设置则 TCU 视为 0b101（48-bit）。不能与配置为 52-bit PA 的 DTI 组件配合使用
   * - 最多 62 个 TBU
     | r1p0 版本支持最多 62 个 TBU（``TCUCFG_NUM_TBU = 62``），
       r0p0 版本限制为 14 个
   * - STE.ALLOCCFG 被忽略
     | TBU 忽略 TCU 传递的 STE.ALLOCCFG 字段
   * - 事件不合并
     | MMU-600AE 从不合并事件，STE.MEV 字段被忽略
   * - 单写 outstanding
     | TCU 同时只能有一个 outstanding 写事务
   * - 同 AXI ID 有序
     | TBU 中相同 AXI ID 的事务必须保序，不同 AXI ID 可以乱序推进
   * - StreamID 限制
     | StreamID ≥ 224 导致 ``C_BAD_STREAMID`` 故障
