==========================================
TCU 和 TBU 一致性维护机制深入分析
==========================================

本文档深入分析 MMU-600AE 中 TCU 和 TBU 如何维护翻译缓存的一致性，
以及软件在哪些环节可以、必须或无法干预一致性维护。

参考文档：
- ARM CoreLink MMU-600AE TRM（DEN 0060D）
- ARM SMMUv3 架构规范（IHI 0070G）

.. contents::
   :depth: 2
   :local:


什么需要维护一致性
=======================

MMU-600AE 中存在多级翻译缓存，每一级都可能存储过时的（stale）翻译数据：

.. list-table::
   :header-rows: 1
   :widths: 20 20 15 45

   * - 缓存位置
     - 缓存内容
     - 所在组件
     - 一致性风险场景
   * - Micro TLB
     | 翻译条目（PA、属性）
     | TBU
     | 页表被修改、STE/CD 被修改、设备迁移
   * - Main TLB
     | 翻译条目
     | TBU
     | 同上
   * - Walk Cache (S1L0-L3, S2L0-L3)
     | 页表各级描述符
     | TCU
     | 页表被软件修改
   * - Configuration Cache
     | STE 和 CD 内容
     | TCU
     | Stream Table 或 CD 表被软件修改
   * - TBU Config Cache (Micro TLB 内)
     | STE/CD 配置
     | TBU
     | 同 Configuration Cache

当软件修改页表、Stream Table 或 CD 后，上述所有缓存中可能存在与内存不一致的旧数据。
如果不主动失效，后续事务可能使用过时翻译，导致错误的物理地址或安全属性。

一致性维护的本质就是：**在软件更新翻译结构后，确保所有相关缓存中的旧条目被失效或更新**。


硬件自动维护一致性的机制
=============================

1 DVM 广播 TLB 失效
-----------------------

当系统支持 Broadcast TLB Maintenance（``sup_btm = HIGH``）时，PE（处理器）
执行 TLBI 指令后，互连通过 DVM 协议将失效消息广播到 TCU。
TCU 的 QTW/DVM 接口支持 DVMv8.1，可以接收并处理这些消息。

**工作流程**::

    PE 执行 TLBI 指令
        → 互连广播 DVM TLB Invalidate 消息
            → TCU QTW/DVM 接口接收
                → TCU 失效自身 Configuration Cache 和 Walk Cache 中匹配的条目
                → TCU 通过 DTI 发送 Invalidation 消息给各 TBU
                    → TBU 失效 Micro TLB 和 MTLB 中匹配的条目
            → PE 发起 DVM Sync
                → TCU 返回 DVM Complete（所有失效完成）

**PTM 控制**：软件可以通过 ``SMMU_CR2.PTM`` 和 ``SMMU_S_CR2.PTM``
选择性地屏蔽 DVM TLB Invalidate 操作：

- ``PTM = 0``：TLB Invalidate 正常应用于对应安全状态的 TLB
- ``PTM = 1``：TLB Invalidate 不生效（但 QTW/DVM 接口仍返回正常响应）
- ``sup_btm = HIGH`` 时复位默认值为 ``PTM = 1``（屏蔽状态），需要软件主动清除

**CD.ASET 对广播失效的影响**：

- ``ASET = 0``（ASID 与 PE 共享）：TLB 条目会被匹配的广播失效操作影响
- ``ASET = 1``（设备专用地址空间）：TLB 条目不会被 ``VAExIS``/``ASIDExIS`` 失效，
  但仍被 ``ALLEXIS`` 全量失效影响

**优化**：如果自上次 DVM Sync 以来没有发生 TLB Invalidate 操作，
后续 DVM Sync 会立即返回 DVM Complete，不增加延迟。

2 DTI Invalidation 消息传播
-------------------------------

TCU 通过 DTI 协议的 **Invalidation and Synchronization** 消息组将失效
请求传播给 TBU。（来源：Section 2.5）

**关键约束**：

- TCU 同一时刻只能发出**一个** invalidate 消息
- 每个 TBU 只授予**一个** invalidation token（``TOK_INV_GNT``）
- 这意味着**跨 TBU 的失效是串行化的**，不是并行的

**Token 机制**：

- TBU 在连接时通过 ``max_tok_trans`` 信号请求翻译 token 数量
- TBU 通过 ``TOK_INV_GNT`` 授予失效 token（只有 1 个）
- TCU 使用 ``TOK_TRANS_REQ`` 请求翻译 token

3 Transaction Tracker 实现非阻塞失效
-----------------------------------------

TBU 的 Transaction Tracker 允许失效和同步操作在不阻塞 AXI 接口的情况下执行。
即使有未完成的读写事务，TLB 中的条目仍然可以被失效。失效操作会标记对应条目，
后续事务会重新触发翻译。（来源：Section 2.1）

4 TCU 内部失效级联
--------------------

当 TCU 收到失效请求时，失效按以下级联顺序执行：

1. **Configuration Cache** 中匹配的 STE/CD 条目被失效
2. **Walk Cache** 中与被失效配置关联的条目被失效
3. 通过 DTI 将失效消息发送给各 TBU
4. TBU 失效 Micro TLB 和 MTLB 中的匹配条目

Walk Cache 的失效性能受 ``ASID_VMID_HASH`` 位影响（``TCU_CTRL[19]``）：

- ``ASID_VMID_HASH = 0``：仅使用输入地址做索引，失效效率高
- ``ASID_VMID_HASH = 1``：使用 ASID、VMID 和地址做索引，改善缓存利用率，
  但**不提供 ASID 的失效需要遍历整个缓存**

5 SYSCO 接口——TCU 自身的一致性域管理
-----------------------------------------

TCU 通过 SYSCO 接口管理自身是否属于系统一致性域。
这不是用于向 TBU 传播失效的机制，而是用于 TCU 的功耗管理。（来源：Section 2.4.1.7）

- 进入低功耗状态前，TCU 通过 ``syscoreq = HIGH`` 请求退出一致性域
- 系统确认后 ``syscoack = HIGH``
- 恢复时反向操作
- 如果 ``sup_btm = LOW``（不支持 BTM），``syscoreq`` 始终为 LOW

6 ACE5 低功耗扩展
---------------------

MMU-600AE 支持 ACE5 低功耗扩展，允许 TCU 在上下电过程中
订阅/取消订阅 DVM TLB Invalidate 请求，而无需重新编程 DTI Interconnect。
这确保了 TCU 在不同功耗状态下仍能正确参与 DVM 域。（来源：Section 1.3）


软件维护一致性的机制
=========================

1 通过 Command Queue 发送失效命令
------------------------------------

软件可以通过向 SMMU Command Queue 写入命令来主动维护一致性。
这是最核心的软件干预手段。

**TLB 失效命令（仅影响 SMMU 自身 TLB，不广播到 PE）**：

.. list-table::
   :header-rows: 1
   :widths: 40 30 30

   * - 命令
     - 目标范围
     | 安全状态要求
   * - ``CMD_TLBI_NH_ALL(VMID)``
     | 指定 VMID 下所有 NS-EL1 翻译（含 Global）
     | Non-secure CMD Queue
   * - ``CMD_TLBI_NH_ASID(VMID, ASID)``
     | 指定 VMID + ASID 的非 Global 翻译
     | Non-secure CMD Queue
   * - ``CMD_TLBI_NH_VAA(VMID, Addr)``
     | 按地址失效（支持范围失效和 TTL 提示）
     | Non-secure CMD Queue
   * - ``CMD_TLBI_NH_VA(VMID, Addr)``
     | 按地址 + ASID 失效
     | Non-secure CMD Queue
   * - ``CMD_TLBI_NSNH_ALL``
     | 所有 Non-secure 翻译（不区分 VMID）
     | Non-secure CMD Queue
   * - ``CMD_TLBI_EL2_ALL``
     | 所有 Stage 2 翻译
     | Non-secure CMD Queue
   * - ``CMD_TLBI_EL3_ALL``
     | 所有 Secure EL3 翻译
     | Secure CMD Queue（**只能通过 Secure Queue**）
   * - ``CMD_TLBI_NH_ALL(VMID)``
     | 所有 Secure EL1 翻译（等效于 ``CMD_TLBI_NH_ALL``）
     | Secure CMD Queue

**配置缓存失效命令**：

- ``CMD_CFGI_ALL``：失效 TCU Configuration Cache 和 TBU 的配置缓存 + TLB

**关键规则**：所有失效命令序列后**必须**跟 ``CMD_SYNC`` 命令，
确保失效操作完成后再执行后续操作。

2 INV_ALL——全量失效
----------------------

Secure 软件可以通过 ``SMMU_S_INIT.INV_ALL`` 位触发一次全量失效，
覆盖所有翻译体制和安全状态的 Configuration Cache 和 TLB。（来源：Appendix B.1.6）

操作步骤::

    1. Secure 软件设置 SMMU_S_INIT.INV_ALL = 1
    2. SMMU 开始全量失效
    3. 轮询 SMMU_S_INIT.INV_ALL 直到读回 0（失效完成）
    4. 继续后续 SMMU 配置

3 ATS 相关的一致性维护
--------------------------

当使用 PCIe ATS（Address Translation Service）时，一致性维护需要额外步骤：

**翻译配置变更的顺序**::

    1. CMD_TLBI_*   — 失效 SMMU TLB
    2. CMD_SYNC      — 确保 SMMU TLB 失效完成
    3. CMD_ATC_INV   — 失效 PCIe 设备的 ATC（Address Translation Cache）
    4. CMD_SYNC      — 确保 ATC 失效完成

**关键约束**：必须先完成 SMMU TLB 失效，再发起 ATC 失效。
否则 PCIe 设备可能在使用已失效的翻译。

4 DIS_DVM——按节点禁用 DVM
-----------------------------

当某个 TBU 响应 DVM 失效过慢，或软件知道该节点不需要参与 DVM 域时，
可以通过 ``TCU_NODE_CTRLn.DIS_DVM`` 按节点禁用 DVM invalidation。（来源：Section 3.5.6）

设置条件（三个条件同时满足时才应设置）：

1. 该节点对通过 DTI 发出的失效响应较慢
2. 软件知道该节点不需要参与 DVM 域
3. 软件将通过 Command Queue 为该节点手动执行失效

**注意事项**：

- ``DIS_DVM`` 对 DTI-ATS 节点（PCIe RC）无效，因为 ATS 节点从不参与 DVM 失效
- 修改 ``TCU_NODE_CTRLn`` 后，**必须**执行 ``INV_ALL`` 操作，
  然后才能设置 ``SMMUEN = 1``。否则行为不可预测
- Arm 建议：对启用 Direct Indexing 的 TBU 设置 ``DIS_DVM = 1``

5 初始化阶段的一致性维护
---------------------------

MMU-600AE 上电后，所有 TLB 和 Configuration Cache 的内容是未定义的。
软件必须在启用 SMMU 前完成以下步骤（来源：Appendix B.1.6）：

1. **标记所有 STE 为无效**：设置每个 STE 的 ``V = 0``，
   防止未初始化内存被解释为有效配置
2. **确保数据可观察性**：执行 DSB（Data Synchronization Barrier），
   确保 SMMU 能观察到写入的数据。
   如果 ``SMMU_IDR0.COHACC = 0``（非一致系统），可能需要额外步骤
3. **失效所有缓存**：通过 Command Queue 发送失效命令，
   或通过 ``SMMU_S_INIT.INV_ALL`` 执行全量失效
4. **执行 CMD_SYNC**：确保所有失效命令完成

6 非一致系统的额外处理
-------------------------

当 ``SMMU_IDR0.COHACC = 0``（``sup_cohacc = LOW``）时，
TCU 的 QTW 接口不是 I/O 一致的。此时软件必须额外确保：

- 写入翻译结构后执行 DSB
- 可能需要额外的 cache maintenance 操作
- 确保 SMMU 能观察到更新后的数据

（来源：Appendix B.1.6）


软件不能做什么
=================

1 不能直接操作 TBU TLB
--------------------------

TBU 的 Micro TLB 和 MTLB 只能通过以下方式失效：

- TCU 发起的 DTI Invalidation 消息（由 DVM 或 CMD_TLBI_* 触发）
- ``CMD_CFGI_ALL`` 全量失效
- ``SMMU_S_INIT.INV_ALL`` 全量失效

软件**不能**直接读写 TBU 的 TLB 内容。TBU 寄存器（``TBU_CTRL``、``TBU_SCR``）
只包含调试和安全控制位，不包含 TLB 条目管理接口。

2 不能直接操作 TCU Walk Cache
-------------------------------

TCU Walk Cache 只能通过以下方式失效：

- DVM 广播失效消息
- CMD_TLBI_* 命令（间接触发 Walk Cache 失效）
- ``CMD_CFGI_ALL`` 全量失效
- ``SMMU_S_INIT.INV_ALL`` 全量失效

Walk Cache 没有软件可编程的失效接口。当页表被修改后，
只有通过发送 TLB 失效命令间接触发 Walk Cache 失效。

3 不能直接操作 TCU Configuration Cache
-----------------------------------------

Configuration Cache 也没有独立的软件失效接口。
它通过 ``CMD_CFGI_ALL`` 和 ``INV_ALL`` 被间接失效。

4 不能跨功耗域直接操作
-------------------------

当 TBU 处于低功耗状态时：

- 软件不应向目标为该 TBU 的寄存器发起写入，写入不会生效
- TCU 不会向已关断的 TBU 发送错误消息
- FMU（Fault Management Unit）自动停止向已关断的 TBU 发送 ping 消息

软件需要跟踪 TBU 的功耗状态，避免在 TBU 关断时尝试维护一致性。

5 CMD_PREFETCH_CONFIG/ADDR 不生效
-------------------------------------

MMU-600AE 接受但不执行 SMMUv3 的以下预取命令：

- ``CMD_PREFETCH_CONFIG``：预取配置
- ``CMD_PREFETCH_ADDR``：预取配置和 TLB 条目

软件无法通过这些命令主动预热缓存来辅助一致性维护。


Prefetch 与一致性的交互
===========================

1 TCU Prefetch 的风险
-----------------------

当 STE.PF 字段启用 TCU Prefetch 时，TCU 会在完成原始翻译请求后
发起一次推测性的预取请求，仅分配到 TCU Walk Cache，不返回给 TBU。（来源：Section 2.2.3）

一致性风险：

- 预取的翻译数据进入 Walk Cache 后，如果页表被修改且失效命令尚未到达，
  Walk Cache 中可能存在过时数据
- 预取使用与原始请求相同的翻译 slot，必须等待原始请求完成后才能发起
- 如果预取在完成前被非推测请求抢占，预取可能不会完成

2 StashTranslation 与 MTLB 预取
----------------------------------

MMU-600AE 可以使用 ``StashTranslation`` 事务来预取 Main TLB 内容。
但 MMU-600AE **不会将 StashTranslation 事务向下游传播**，
而是始终以 ``BRESP = OKAY`` 终止，即使没有成功存储到 MTLB。

使用约束：后续要利用预取到 MTLB 中的翻译，**必须使用相同的 StreamID**。（来源：Section 7.3）


一致性维护的典型流程
=======================

1 修改页表后的标准流程
-------------------------

::

    ① 软件修改页表（断开旧映射，建立新映射）
    ② 执行 DSB（确保页表更新对 SMMU 可见，尤其是 COHACC=0 时）
    ③ 通过 CMD_QUEUE 发送 CMD_TLBI_* 失效旧翻译
    ④ 通过 CMD_QUEUE 发送 CMD_SYNC 等待失效完成
    ⑤ 后续事务将使用新翻译

2 迁移设备（更改 StreamID 映射）后的流程
-----------------------------------------

::

    ① 更新 Stream Table（修改 STE 的指向）
    ② 执行 DSB
    ③ CMD_CFGI_ALL（失效 Configuration Cache + TLB）
    ④ CMD_SYNC
    ⑤ 设置 SMMUEN = 1（如果是初始化阶段，还需先 INV_ALL）

3 修改 STE 配置属性后的流程
-----------------------------

::

    ① 更新 STE（例如修改 S1DSS、S2DSS 等字段）
    ② 执行 DSB
    ③ CMD_TLBI_*（失效受影响的 TLB 条目，同时间接触发 Configuration Cache 和 Walk Cache 失效）
    ④ CMD_SYNC

4 启用 Direct Indexing 的 TBU 的一致性
-----------------------------------------

::

    ① 设置 TCU_NODE_CTRLn.DIS_DVM = 1（禁用该节点的 DVM）
    ② 执行 INV_ALL（通过 SMMU_S_INIT）
    ③ 后续一致性由软件通过 CMD_QUEUE 命令手动维护
    ④ 软件需要知道何时需要失效（因为 DVM 广播不会到达该 TBU）


总结
=======

.. list-table::
   :header-rows: 1
   :widths: 30 30 40

   * - 一致性维护手段
     | 维护范围
     | 软件可控性
   * - **DVM 广播失效**
     | TCU + TBU 全部缓存
     | 间接控制（通过 PE TLBI 指令触发，PTM 可屏蔽）
   * - **CMD_TLBI_*** （Command Queue）
     | TCU + TBU 全部缓存
     | **直接控制**（软件主动发送，最常用）
   * - **CMD_CFGI_ALL**（Command Queue）
     | Configuration Cache + TLB
     | **直接控制**（用于配置变更）
   * - **SMMU_S_INIT.INV_ALL**
     | 全量（所有缓存、所有安全状态）
     | **直接控制**（仅 Secure 软件，用于初始化）
   * - **DIS_DVM**（按节点禁用）
     | 排除指定 TBU 节点
     | **直接控制**（需配合 CMD_QUEUE 手动失效）
   * - **DSB**
     | 数据可观察性
     | **直接控制**（非一致系统必须）
   * - **SYSCO**
     | TCU 自身一致性域
     | 间接控制（硬件自动，LPI 触发）

**核心结论**：软件可以通过 Command Queue 命令（``CMD_TLBI_*``、``CMD_CFGI_ALL``、
``CMD_SYNC``）和寄存器（``INV_ALL``、``DIS_DVM``、``PTM``）**直接控制**一致性维护，
但**不能直接操作** TBU TLB、TCU Walk Cache 和 Configuration Cache 的内容。
失效操作是软件唯一可用的干预方式。
