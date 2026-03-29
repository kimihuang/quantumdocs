==============================
AMBA CHI 一致性协议深度分析
==============================

本文档基于 **IHI0050G (Issue G, Mar 2024)** -- AMBA CHI Architecture Specification，
深入分析 CHI 一致性协议的完整设计，从整体架构到细节规则。

参考文档：docs/pdf/arm/IHI0050G_amba_chi_architecture_spec.md

.. contents::
   :depth: 3
   :local:

------------------------
一致性协议总览
------------------------


*源文档: B1.5 Coherence overview, B4 Coherence Protocol*

1 设计目标
------------

CHI 一致性协议的核心目标：

- **多拷贝原子性 (Multi-copy atomicity)**: 所有写入同一位置的操作被序列化，所有 Requester 以相同顺序观察
- **对同一位置的读不返回尚未被所有 Requester 观察到的写值**
- **缓存一致性粒度**: 64 字节对齐的缓存行
- **主内存不需要在所有时刻保持最新**: 仅在缓存行不再被任何缓存持有时才需要更新

2 Write-Invalidate 协议
---------------------------

CHI 采用 **Write-Invalidate** 协议：当 Requester 写入共享缓存行时，必须先使所有其他副本无效化，然后才能进行写操作。

3 缓存状态模型
-----------------

缓存行状态由三个维度的组合定义：

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - 维度
     - 含义
   * - Valid / Invalid
     - Valid: 缓存行存在; Invalid: 不在缓存中
   * - Unique / Shared
     - Unique: 仅在本缓存; Shared: 可存在于多个缓存
   * - Clean / Dirty
     - Clean: 无需更新主内存; Dirty: 已修改，驱逐时必须写回
   * - Full / Partial / Empty
     - Full: 所有字节有效; Partial: 部分有效; Empty: 无有效字节

由此组合产生 **7 种协议定义的缓存行状态**。


------------------------
缓存行状态详解
------------------------

*源文档: B4.1 Cache line states*

1 七状态定义
---------------

.. list-table::
   :header-rows: 1
   :widths: 8 22 12 12 12 12 22

   * - 状态
     - 描述
     - 唯一性
     - 脏净
     - 数据
     - 可修改
     - Snoop 数据返回规则
   * - **I**
     | Invalid
     | -
     | -
     | 无
     | -
     | 不在缓存中，不产生 Snoop 响应
   * - **UC**
     | Unique Clean
     | 唯一
     | 净
     | Full
     | 是
     | 允许(但不要求)返回/转发
   * - **UCE**
     | Unique Clean Empty
     | 唯一
     | 净
     | Empty
     | 是
     | **禁止**返回数据或转发
   * - **UD**
     | Unique Dirty
     | 唯一
     | **脏**
     | Full
     | 是
     | **必须**返回 Home; **期望**转发
   * - **UDP**
     | Unique Dirty Partial
     | 唯一
     | **脏**
     | Partial
     | 是
     | **必须**返回 Home; **禁止**转发
   * - **SC**
     | Shared Clean
     | 共享
     | 净
     | Full
     | **否**
     | RetToSrc=0: **禁止**返回; RetToSrc=1: 期望返回; 期望转发
   * - **SD**
     | Shared Dirty
     | 共享
     | **脏**
     | Full
     | **否**
     | **必须**返回 Home; **期望**转发

2 空行所有权
---------------

**UCE (Unique Clean Empty)**:

- 持有缓存行的唯一所有权，但无有效数据字节
- Requester 可在写入前获取空行以**节省系统带宽**
- 典型场景：Requester 预期要写入某行，先获取空行和写权限，而非获取完整副本

**UDP (Unique Dirty Partial)**:

- 持有唯一所有权，部分字节已修改
- 驱逐时必须与下层缓存/内存**合并**数据
- 从 UCE 或 UD 经部分写入可进入此状态

3 静默缓存状态转换
-----------------------

缓存可以因内部事件改变状态而不通知系统其余部分。合法的静默转换：

.. list-table::
   :header-rows: 1
   :widths: 20 25 25 30

   * - 触发条件
     - 当前状态
     - 下一状态
     - 可用于使之非静默的事务
   * - Cache eviction
     | UC
     | I
     | Evict, WriteEvictFull, WriteEvictOrEvict
   * - Cache eviction
     | UCE
     | I
     | Evict
   * - Cache eviction
     | SC
     | I
     | Evict, WriteEvictOrEvict
   * - Cache eviction
     | UD
     | I
     | Evict
   * - Cache eviction
     | UDP
     | I
     | Evict
   * - Local sharing
     | UC
     | SC
     | (无，内部共享不需通知)
   * - Local sharing
     | UD
     | SD
     | (无，内部共享不需通知)
   * - Store (full)
     | UC
     | UD
     | -
   * - Store (partial)
     | UCE
     | UDP
     | -
   * - Store (fills line)
     | UD
     | UDP
     | -
   * - Store (fills line)
     | UDP
     | UD
     | -

.. note::

   - UC -> UCE **不允许**作为静默转换
   - 静默转换可以级联：任何导致 UD、UDP 或 SC 的静默转换可以继续进一步转换

------------------------
请求类型深度分析
------------------------

*源文档: B4.2 Request types*

1 Read 事务
---------------

Read 事务的共同特征：

- 必须向 Requester 提供数据（MakeReadUnique 除外，数据可选）
- 数据可来自 Home Node、Peer RN-F 或 Subordinate Node
- Allocating Read 获得的数据必须以系统一致方式缓存
- Non-allocating Read（ReadNoSnp, ReadOnce*）不期望被缓存

### 3.1.1 Non-allocating Read

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 事务类型
     - 语义
   * - **ReadNoSnp**
     | Non-snoopable 地址读取。也可由 Home 发出获取副本。最终状态 I（忽略响应中缓存状态）
   * - **ReadNoSnpSep**
     | Home to SN 读取，仅返回数据响应（无完成响应）。用于分离式完成
   * - **ReadOnce**
     | Snoopable 快照读取。不改变系统中其他副本。最终状态 I
   * - **ReadOnceCleanInvalid**
     | 快照读取 + **建议**清理/无效化其他副本。脏副本必须写回。**提示性质**，完成不保证清理所有缓存副本
   * - **ReadOnceMakeInvalid**
     | 快照读取 + **建议**无效化其他副本。脏副本**不要求**写回。返回 I/UC/UD，请求者**必须忽略**缓存状态。Invalidation 必须在读数据响应前提交

.. important::

   ReadOnceMakeInvalid 的关键约束：Invalidation 在读数据响应前提交，确保后续写事务不会被此 invalidation 影响。但可导致 Dirty 缓存行丢失，使用场景需严格限制。

### 3.1.2 Allocating Read

.. list-table::
   :header-rows: 1
   :widths: 30 20 50

   * - 事务类型
     | 允许返回的缓存状态
     | 用途
   * - **ReadClean**
     | UC, SC
     | 分配到不支持 Dirty 的缓存（如 I-Cache）
   * - **ReadNotSharedDirty**
     | UC, UD, SC (**不允许 SD**)
     | 加载操作，Requester 不能接受 SD 状态
   * - **ReadShared**
     | UC, UD, SC, SD
     | 加载操作，可接受 SD 状态（优于 ReadNotSharedDirty）
   * - **ReadUnique**
     | UC, UD
     | 为 Store 操作获取 Unique 副本
   * - **ReadPreferUnique**
     | UC, UD, SC, SD
     | 优先获取 Unique，但接受 Shared（当其他 Agent 在执行独占序列时）
   * - **MakeReadUnique**
     | UC, UD (数据可选)
     | 获取 Unique 写权限。特殊事务，见下文详述

### 3.1.3 MakeReadUnique 深度分析

MakeReadUnique 是一个**特殊的事务类型**，用于在 Requester 已有缓存行副本的情况下获取 Unique 写权限：

**核心特点**:

- 数据响应是**可选的**（如果 Requester 仍持有副本，无需重新获取数据）
- 初始状态限于 **SC 或 SD**（Requester 已有共享副本）
- 如果事务过程中副本被 Snoop 失效，Home 必须提供数据

**响应类型**:

.. list-table::
   :header-rows: 1
   :widths: 35 65

   * - 响应
     | 含义
   * - Comp_UC
     | Requester 保留副本（Clean），所有其他副本已被无效化。无需数据
   * - Comp_UD_PD
     | Requester 保留副本且变为 Dirty。其他位置的脏副本已被无效化，Dirty 责任传递给 Requester
   * - CompData_UC
     | Requester 丢失了副本，获得 Clean 数据。数据和完成合并
   * - CompData_UD_PD
     | Requester 丢失了副本，获得 Dirty 数据。Dirty 责任传递
   * - RespSepData + DataSepResp_UC/UD_PD
     | 分离式响应，语义同上

**Exclusive 变体 (MakeReadUnique Excl)** 的额外响应：

.. list-table::
   :header-rows: 1
   :widths: 25 30 45

   * - 响应
     | 全局独占检查结果
     | 含义
   * - Comp_SC
     | **失败**
     | 其他缓存中存在共享副本，Requester 保留 SC 状态
   * - CompData_SC
     | **失败**
     | 其他缓存中存在共享副本，Requester 可能丢失副本
   * - RespSepData + DataSepResp_SC
     | **失败**
     | 同上，分离式响应

.. important::

   如果 Requester 在收到响应时仍持有 SD 状态的副本，**必须使用自身副本而非响应中的数据**（因为响应中的数据可能是过时的）。

**Snoop Filter 对响应的影响**:

- **精确 Snoop Filter**: Home 知道 Requester 是否仍持有副本，可精确选择是否附带数据
- **非精确/无 Snoop Filter**: Home 必须假设 Requester 已丢失副本，始终提供数据

2 Dataless 事务
-------------------

### 3.2.1 所有权获取事务

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - 事务
     | 初始状态
     | 最终状态
     | 描述
   * - **CleanUnique**
     | I/SC/SD
     | UC/UCE
     | 获取 Unique Clean 权限。脏数据**必须**写回内存。获取后可分配到空行 (UCE)
   * - **MakeUnique**
     | I/SC/SD
     | UC
     | 获取所有权（保证写全部字节）。脏数据**丢弃不传**。比 CleanUnique 更高效

### 3.2.2 缓存维护操作 (CMO)

.. list-table::
   :header-rows: 1
   :widths: 30 15 15 40

   * - 事务
     | 初始状态
     | 最终状态
     | 描述
   * - **CleanShared**
     | I/SC/UC
     | 不变
     | 共享缓存行的 Clean 操作。缓存状态不变
   * - **CleanSharedPersist**
     | I/SC/UC
     | 不变
     | Clean + 确保持久化
   * - **CleanSharedPersistSep**
     | I/SC/UC
     | 不变
     | Clean + 持久化（分离式响应）
   * - **CleanInvalid**
     | I
     | I
     | 无效化缓存行。脏数据**必须**写回
   * - **CleanInvalidPoPA**
     | I
     | I
     | 跨 PAS 无效化。脏数据写到 PoPA 之后（跨物理地址空间可见性）
   * - **MakeInvalid**
     | I
     | I
     | 无效化缓存行。脏数据**丢弃**（唯一允许丢弃脏数据的 CMO）

.. warning::

   - CMO 前允许状态为 UC/UCE/SC，但**必须在发出事务前转换为 I 状态**
   - MakeInvalid 是唯一允许**丢弃脏数据**的 CMO，使用场景需严格限制
   - CleanShared* 不会改变缓存状态（Comp 的 Resp 字段被忽略）

### 3.2.3 驱逐与 Stash

.. list-table::
   :header-rows: 1
   :widths: 30 15 15 40

   * - 事务
     | 初始状态
     | 最终状态
     | 描述
   * - **Evict**
     | I
     | I
     | 指示缓存行不再缓存（仅 Clean 行）。脏行必须先 CopyBack
   * - **StashOnceUnique**
     | I
     | I
     | 将缓存行 Stash 到目标缓存（Unique 方式）
   * - **StashOnceSepUnique**
     | I
     | I
     | Stash + 分离式完成
   * - **StashOnceShared/Sep**
     | I
     | I
     | Stash（Shared 方式）

3 Write 事务
----------------

### 3.3.1 Immediate Write（立即写入）

.. list-table::
   :header-rows: 1
   :widths: 28 15 15 42

   * - 事务类型
     | 初始状态
     | 最终状态
     | 特点
   * - **WriteNoSnpPtl**
     | I
     | I
     | Non-snoopable 部分写入（需 Byte Enable）
   * - **WriteNoSnpFull**
     | I
     | I
     | Non-snoopable 全行写入
   * - **WriteNoSnpDef**
     | I
     | I
     | 可延迟写入。Completer 可返回 OK/Defer 响应
   * - **WriteNoSnpZero**
     | I
     | I
     | 写零。无数据传输（使用 BE 指示清零范围）
   * - **WriteUniquePtl**
     | I
     | I
     | Snoopable 部分写入。Peer 必须为 I
   * - **WriteUniqueFull**
     | I
     | I
     | Snoopable 全行写入。Peer 必须为 I
   * - **WriteUniqueZero**
     | I
     | I
     | Snoopable 写零。Peer 必须为 I
   * - **WriteUniquePtlStash**
     | I
     | I
     | Snoopable 写入 + Stash 提示
   * - **WriteUniqueFullStash**
     | I
     | I
     | Snoopable 全行写入 + Stash 提示

.. note::

   - DWT 流**禁止**用于 WriteNoSnpZero 和 WriteUniqueZero
   - 所有 Immediate Write 最终状态为 **I**（WriteClean 除外）
   - WriteUnique* 要求所有 Peer 的最终状态为 **I**

### 3.3.2 CopyBack Write（写回）

.. list-table::
   :header-rows: 1
   :widths: 25 15 15 15 30

   * - 事务
     | 初始状态
     | WB 状态
     | 最终状态
     | 描述
   * - **WriteBackFull**
     | UD/SD
     | UD/SD
     | I
     | 完整行写回。Home 可选择不接收（返回 Comp 而非 CompDBIDResp）
   * - **WriteBackPtl**
     | UD/SD
     | UD/SD
     | I
     | 部分行写回
   * - **WriteCleanFull**
     | UD/SD
     | UC/SC
     | UC/SC
     | **特殊**: 最终状态为 **UC/SC 而非 I**。UD->UC; SD->SC
   * - **WriteEvictFull**
     | UD/SD
     | UD/SD
     | I
     | 写回 + 驱逐
   * - **WriteEvictOrEvict**
     | UD/SD
     | I
     | I
     | 如果状态为 UD/SD 则写回；如果为 UC/SC 则直接驱逐

.. important::

   **WriteCleanFull 是唯一一个最终状态不是 I 的 Write 事务**。它允许 Requester 在写回后仍持有 Clean 副本，减少后续重新读取的延迟。

### 3.3.3 Combined Write（组合写入）

组合写入将 Write 操作与 CMO 合并，减少事务数量。支持的组合：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Write 类型
     | 可组合的 CMO
   * - WriteNoSnpPtl
     | CleanInv, CleanSh, CleanShPerSep, CleanInvPoPA
   * - WriteNoSnpFull
     | CleanInv, CleanSh, CleanShPerSep, CleanInvPoPA
   * - WriteNoSnpDef
     | **不允许任何组合**
   * - WriteUniquePtl
     | CleanSh, CleanShPerSep
   * - WriteUniqueFull
     | CleanSh, CleanShPerSep
   * - WriteBackFull
     | CleanInv, CleanSh, CleanShPerSep, CleanInvPoPA
   * - WriteCleanFull
     | CleanSh, CleanShPerSep

.. note::

   组合写入的完成响应使用 **CompCMO** 操作码，而非 Comp。PoPA CMO 的 CompCMO 语义跨越所有物理地址空间。

### 3.3.4 WriteNoSnpDef 特殊响应

WriteNoSnpDef 支持一种特殊的完成语义，Completer 可以推迟写入：

.. list-table::
   :header-rows: 1
   :widths: 20 20 60

   * - RespErr
     | Resp
     | 描述
   * - 00
     | 000
     | **OK/Successful**: Completer 支持 Deferrable 写入且成功
   * - 00
     | 001
     | **OK/Unsupported**: Completer 不支持 Deferrable 写入。Requester 必须完成完整事务流
   * - 00
     | 010
     | **OK/Defer**: Completer 支持，但当前无法服务。内存未更新
   * - 10
     | 000
     | **DERR**: 数据错误
   * - 11
     | 000
     | **NDERR**: 非数据错误

4 Atomic 事务
----------------

.. list-table::
   :header-rows: 1
   :widths: 20 15 20 45

   * - 类型
     | 返回数据
     | 操作数
     | 描述
   * - **AtomicStore**
     | 否
     | 8 种
     | STADD, STCLR, STEOR, STSET, STSMAX, STSMIN, STUMAX, STUMIN
   * - **AtomicLoad**
     | 是(原值)
     | 8 种
     | LDADD, LDCLR, LDEOR, LDSET, LDSMAX, LDSMIN, LDUMAX, LDUMIN
   * - **AtomicSwap**
     | 是(原值)
     | 1 种
     | 交换新旧值
   * - **AtomicCompare**
     | 是(原值)
     | 1 种
     | 匹配则写新值
   * - 数据大小
     |
     |
     | Store/Load/Swap: 1/2/4/8B; Compare: 2/4/8/16/32B
   * - 最终状态
     |
     |
     | 所有 Atomic 事务: **I**

Atomic 事务的关键约束：

- 返回的数据必须是**原始值**（操作前）
- 数据不能在 Requester 缓存
- 所有 Peer 最终状态必须为 **I**
- **不能使用 DMT/DWT**
- **SnoopMe**: 当 Requester 无法确定缓存为 I 时**必须**置 1，导致 Home 向 Requester 发 Snoop

------------------------
Snoop 请求类型深度分析
------------------------

*源文档: B4.3 Snoop request types, B4.4 Request/Snoop correspondence*

1 Snoop 类型总览
--------------------

.. list-table::
   :header-rows: 1
   :widths: 25 12 12 12 12 27

   * - Snoop 类型
     | 获取数据
     | 可转发
     | Snoopee 最终
     | 可 DataPull
     | 关键特点
   * - **SnpOnceFwd**
     | 是
     | 是
     | 尽量不变
     | 否
     | 基础 Snoop + 转发
   * - **SnpOnce**
     | 是
     | 否
     | 尽量不变
     | 否
     | 基础 Snoop（不转发）
   * - **SnpStashUnique**
     | 否
     | 否
     | **不变**
     | 是
     | Stash 到 Unique 位置，DataPull=ReadUnique
   * - **SnpStashShared**
     | 否
     | 否
     | **不变**
     | 是
     | Stash 到 Shared 位置，DataPull=ReadNotSharedDirty
   * - **SnpCleanFwd**
     | 是
     | 是
     | Shared
     | 否
     | 要求 Clean + 转发
   * - **SnpClean**
     | 是
     | 否
     | Shared
     | 否
     | 要求 Clean（不转发）
   * - **SnpNotSharedDirtyFwd**
     | 是
     | 是
     | Shared
     | 否
     | 禁止 SD 状态 + 转发
   * - **SnpNotSharedDirty**
     | 是
     | 否
     | Shared
     | 否
     | 禁止 SD 状态
   * - **SnpSharedFwd**
     | 是
     | 是
     | Shared
     | 否
     | 要求 Shared + 转发
   * - **SnpShared**
     | 是
     | 否
     | Shared
     | 否
     | 要求 Shared
   * - **SnpUniqueFwd**
     | 是
     | 是
     | **Invalid**
     | 否
     | 要求 Unique + 转发
   * - **SnpUnique**
     | 是
     | 否
     | **Invalid**
     | 否
     | 要求 Unique
   * - **SnpPreferUniqueFwd**
     | 是
     | 是
     | 条件 Invalid
     | 否
     | 优先 Unique + 转发（取决于独占序列）
   * - **SnpPreferUnique**
     | 是
     | 否
     | 条件 Invalid
     | 否
     | 优先 Unique
   * - **SnpUniqueStash**
     | 否
     | 否
     | Invalid
     | 是
     | Unique + Stash，DataPull=ReadUnique
   * - **SnpCleanShared**
     | 否
     | 否
     | 非 Dirty
     | 否
     | 移除脏副本
   * - **SnpCleanInvalid**
     | 是
     | 否
     | **Invalid**
     | 否
     | 获取脏副本并无效化
   * - **SnpMakeInvalid**
     | 否
     | 否
     | **Invalid**
     | 否
     | **丢弃**脏副本并无效化
   * - **SnpMakeInvalidStash**
     | 否
     | 否
     | Invalid
     | 是
     | MakeInvalid + Stash
   * - **SnpQuery**
     | 否
     | 否
     | **不变**
     | 否
     | 探测状态，不改变缓存行
   * - **SnpDVMOp**
     | -
     | -
     | -
     | 否
     | DVM 维护，1 个 DVMOp -> 2 个 SnpDVMOp -> 1 个响应

2 SnpMakeInvalid 的使用限制
-------------------------------

SnpMakeInvalid 是唯一允许**丢弃脏数据**的 Snoop，因此其使用受到严格限制：

.. list-table::
   :header-rows: 1
   :widths: 50 50

   * - 适用场景
     | 限制条件
   * - WriteUniqueFull / WriteUniqueFullStash / MakeUnique / MakeInvalid
     | 仅当 TagOp=Update，或已知 Snoopee 无 Dirty tags
   * - MakeReadUnique
     | 仅当请求者仍有副本 + Snoopee 无 Dirty tags，或请求者已丢失副本 + Snoopee 无脏副本
   * - MakeInvalid / WriteUniqueZero
     | 仅当 Home 已知 Snoopee 无 Dirty tags

3 Request/Snoop 对应规则
-----------------------------

关键规则：

1. **转发 Snoop 只能发给 1 个 RN-F**；非转发 Snoop 可发多个
2. **<64B 请求禁止使用 Forwarding Snoop**（因为无法保证原子性）
3. Home **允许**将 invalidating snoop 替换为 SnpUnique 或 SnpCleanInvalid
4. Home **允许**将 forwarding snoop 替换为非 forwarding 类型
5. Home **允许自发产生 snoop**（如 backward invalidation）

------------------------
响应类型深度分析
------------------------

*源文档: B4.5 Response types*

1 Completion Response
--------------------------

完成响应是每个事务（除 PCrdReturn 和 PrefetchTgt）必须的最终消息，保证请求已到达 PoS 或 PoC。

### 5.1.1 Read 完成响应

Read 完成可以是**合并式**（CompData）或**分离式**（RespSepData + DataSepResp）。

.. list-table::
   :header-rows: 1
   :widths: 30 10 20 40

   * - 响应
     | Resp[2:0]
     | 允许的最终缓存状态
     | 说明
   * - **CompData_I**
     | 0b000
     | I
     | 不能保持一致副本
   * - **CompData_SC**
     | 0b001
     | SC, I
     | 脏责任**不传递**
   * - **CompData_UC**
     | 0b010
     | UC, UCE, SC, I
     | 脏责任不传递。对 ReadNoSnp/ReadOnce 缓存行不一致
   * - **CompData_UD_PD**
     | 0b110
     | UD, SD
     | 脏责任**传递**给 Requester
   * - **CompData_SD_PD**
     | 0b111
     | SD
     | 脏责任**传递**给 Requester

.. note::

   Pass Dirty (_PD) 表示内存更新责任从 Home/其他缓存转移到 Requester。Requester 获得了脏数据，未来驱逐时必须写回。

### 5.1.2 Dataless 完成响应

.. list-table::
   :header-rows: 1
   :widths: 25 10 25 40

   * - 响应
     | Resp[2:0]
     | 允许的最终缓存状态
     | 说明
   * - **Comp_I**
     | 0b000
     | I
     -
   * - **Comp_SC**
     | 0b001
     | SC, I
     -
   * - **Comp_UC**
     | 0b010
     | UD, UC, UCE, SC, I
     -
   * - **Comp_UD_PD**
     | 0b110
     | UD, SD
     | 脏责任传递

.. note::

   对于 CMO 事务（Comp, CompCMO, CompPersist），**缓存状态字段被忽略**，缓存行状态保持不变。

### 5.1.3 Write 完成响应

Write 完成响应**不包含缓存状态信息**（Resp 必须为 0，WriteNoSnpDef 除外）：

- **Comp**: 完成响应与 DBIDResp 分离
- **CompDBIDResp**: 完成响应与 DBIDResp 合并（所有 CopyBack 必须使用）

### 5.1.4 WriteData 响应

.. list-table::
   :header-rows: 1
   :widths: 35 10 20 35

   * - 响应
     | Opcode
     | 缓存行状态
     | 说明
   * - CopyBackWriteData_UC
     | 0x2
     | UC
     | CopyBack 数据，Clean
   * - CopyBackWriteData_SC
     | 0x2
     | SC
     | CopyBack 数据，Shared Clean
   * - CopyBackWriteData_UD_PD
     | 0x2
     | UD 或 UDP
     | CopyBack 数据，**脏责任传递**
   * - CopyBackWriteData_SD_PD
     | 0x2
     | SD
     | CopyBack 数据，**脏责任传递**
   * - NonCopyBackWriteData
     | 0x3
     | I
     | Immediate Write 数据
   * - NonCopyBackWriteDataCompAck
     | 0xC
     | I
     | Immediate Write + CompAck 合并
   * - WriteDataCancel
     | 0x7
     | I
     | 写入取消

.. important::

   WriteData 的缓存状态反映的是**发送数据时的状态**，可能与原始请求发送时不同（期间可能收到 Snoop）。

2 Snoop Response
--------------------

Snoop 响应有 5 种形式：

.. list-table::
   :header-rows: 1
   :widths: 35 10 55

   * - 形式
     | Opcode
     | 描述
   * - **SnpResp** (无数据)
     | 0x1 (RSP)
     | 无数据传输
   * - **SnpRespFwded** (无数据 + DCT)
     | 0x9 (RSP)
     | 数据直接发送给 Requester，无需向 Home 发数据
   * - **SnpRespData** (有数据)
     | 0x1 (DAT)
     | 完整行数据发送给 Home
   * - **SnpRespDataPtl** (部分数据)
     | 0x5 (DAT)
     | 部分行数据发送给 Home（仅 UDP 状态）
   * - **SnpRespDataFwded** (有数据 + DCT)
     | 0x6 (DAT)
     | 数据同时发送给 Home 和 Requester

### 5.2.1 Non-forward Snoop 响应（无数据）

.. list-table::
   :header-rows: 1
   :widths: 25 10 25 40

   * - 响应
     | Resp[2:0]
     | 缓存行最终状态
     | 说明
   * - SnpResp_I
     | 0b000
     | I
     -
   * - SnpResp_SC
     | 0b001
     | SC, I
     -
   * - SnpResp_UC
     | 0b010
     | UC, UCE, SC, I
     -
   * - SnpResp_UD
     | 0b010
     | UD
     | 注意：编码与 UC 相同，但语义不同
   * - SnpResp_SD
     | 0b011
     | SD
     -

.. important::

   SnpResp_UC 和 SnpResp_UD 使用**相同的 Resp 编码 (0b010)**，区别在于缓存行实际状态。

### 5.2.2 Forward Snoop 响应（无数据 + DCT）

Forward Snoop 无数据响应包含 **FwdState** 字段，指示转发给 Requester 的缓存行状态：

.. list-table::
   :header-rows: 1
   :widths: 35 10 10 20 25

   * - 响应
     | Resp
     | FwdState
     | Snoopee 最终
     | 转发状态
   * - SnpResp_I_Fwded_I
     | 0b000
     | 0b000
     | I
     | I
   * - SnpResp_I_Fwded_SC
     | 0b000
     | 0b001
     | I
     | SC
   * - SnpResp_I_Fwded_UC
     | 0b000
     | 0b010
     | I
     | UC
   * - SnpResp_I_Fwded_UD_PD
     | 0b000
     | 0b110
     | I
     | UD (脏责任传递)
   * - SnpResp_I_Fwded_SD_PD
     | 0b000
     | 0b111
     | I
     | SD (脏责任传递)
   * - SnpResp_SC_Fwded_I/SC/SD_PD
     | 0b001
     | -
     | SC
     | I/SC/SD
   * - SnpResp_UC/UD_Fwded_I
     | 0b010
     | 0b000
     | UC/UD
     | I
   * - SnpResp_SD_Fwded_I/SC
     | 0b011
     | -
     | SD
     | I/SC

### 5.2.3 Snoop 响应（有数据）

有数据的 Snoop 响应中，**Pass Dirty 只能在有数据时断言**：

Non-forward 有数据：

.. list-table::
   :header-rows: 1
   :widths: 30 10 25 35

   * - 响应
     | Resp
     | 缓存行状态
     | 说明
   * - SnpRespData_I
     | 0b000
     | I
     | -
   * - SnpRespData_UC/UD
     | 0b010
     | UC 或 UD
     | 相同编码
   * - SnpRespData_SC
     | 0b001
     | SC
     | -
   * - SnpRespData_SD
     | 0b011
     | SD
     | -
   * - SnpRespData_I_PD
     | 0b100
     | I
     | 脏责任传递给 Home
   * - SnpRespData_UC_PD
     | 0b110
     | UC
     | 脏责任传递给 Home
   * - SnpRespData_SC_PD
     | 0b101
     | SC
     | 脏责任传递给 Home
   * - SnpRespDataPtl_I_PD
     | 0b100
     | I
     | 部分数据，脏责任传递
   * - SnpRespDataPtl_UD
     | 0b010
     | UDP
     | 部分数据

### 5.2.4 Stash DataPull 响应

对 Stash Snoop，Snoopee 可通过 **DataPull** 位发送隐含的 Read 请求：

.. list-table::
   :header-rows: 1
   :widths: 25 50

   * - Snoop 类型
     | 允许的 DataPull 响应
   * - SnpUniqueStash
     | SnpResp_I_Read, SnpRespData_I_Read, SnpRespData_I_PD_Read, SnpRespDataPtl_I_PD_Read
   * - SnpMakeInvalidStash
     | SnpResp_I_Read
   * - SnpStashUnique
     | SnpResp_I_Read, SnpResp_UC_Read, SnpResp_SC_Read, SnpResp_SD_Read
   * - SnpStashShared
     | SnpResp_I_Read, SnpResp_UC_Read

----------------------------
Requester 端缓存状态转换
----------------------------

*源文档: B4.7 Cache state transitions at a Requester*

1 Read 事务状态转换
-----------------------

### 6.1.1 ReadClean

.. list-table::
   :header-rows: 1
   :widths: 25 15 15 15 30

   * - 初始状态
     | TagOp
     | 期望响应
     | 允许的最终状态
     | 特殊规则
   * - I
     | Transfer
     | CompData_SC
     | SC
     | -
   * - UC
     | Transfer
     | CompData_UC/SC
     | UC 或 SC
     | UC 收到 SC 响应时**保持 UC**（不降级）
   * - UC, UCE
     | Transfer
     | CompData_SC/UC
     | UC
     | UC 收到 SC 响应时**保持 UC**
   * - UD, UDP
     | Transfer
     | CompData_SC/UC
     | UD
     | UD 收到 SC 响应时**保持 UD**；来自内存的数据必须 drop (UD/SD) 或 merge (UDP)
   * - SC
     | Transfer
     | CompData_SC/UC
     | SC
     | -
   * - SD
     | Transfer
     | CompData_SC/UC
     | SD
     | SD 收到 SC 响应时**保持 SD**；来自内存的数据必须 drop
   * - UCE
     | != Transfer
     | CompData_SC/UC
     | UC
     | -
   * - I, UCE
     | != Transfer
     | CompData_SC/UC
     | SC 或 UC
     | -

### 6.1.2 ReadNotSharedDirty / ReadShared

.. list-table::
   :header-rows: 1
   :widths: 25 15 15 15 30

   * - 初始状态
     | 事务
     | 期望响应
     | 允许的最终状态
     | 特殊规则
   * - I, UCE
     | ReadNotSharedDirty
     | CompData_SC/UC
     | SC 或 UC
     | **不允许 SD**
   * - I, UCE
     | ReadShared
     | CompData_SC/UC
     | SC 或 UC
     | -
   * - UD
     | ReadNotSharedDirty
     | CompData_UD_PD
     | UD
     | **不允许 SD**
   * - UD
     | ReadShared
     | CompData_UD_PD
     | UD
     | -
   * - SD
     | ReadNotSharedDirty
     | (不适用)
     | -
     | **SD 不允许作为 ReadNotSharedDirty 的初始状态**
   * - SD
     | ReadShared
     | CompData_SD_PD/UD_PD
     | SD 或 UD
     | -

### 6.1.3 ReadUnique

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - 初始状态
     | 期望响应
     | 最终状态
     | 特殊规则
   * - I, SC
     | CompData_UC
     | UC, UCE
     | -
   * - I, SC
     | CompData_UD_PD
     | UD
     | -
   * - UD, UDP
     | CompData_UC
     | UD
     | 来自内存的数据 drop/merge
   * - UD, UDP
     | CompData_UD_PD
     | UD
     | -
   * - SD
     | CompData_UC
     | UD
     | 来自内存的数据必须 drop
   * - SD
     | CompData_UD_PD
     | UD
     | -

### 6.1.4 MakeReadUnique

MakeReadUnique 的状态转换取决于响应时刻的缓存行状态（可能已因 Snoop 改变）：

.. list-table::
   :header-rows: 1
   :widths: 15 12 12 12 12 12 25

   * - 初始
     | 响应时
     | Invalid Snoop
     | 精确 SF
     | 非精确 SF
     | 最终状态
     | 说明
   * - SD
     | SD
     | No
     | Comp_UC: Y
     | -
     | UD
     | 保留副本变 Dirty
   * - SD
     | SD
     | No
     | CompData_UC: -
     | Y
     | UC
     | 数据可能是过时的
   * - SC, SD
     | SC, SD
     | No
     | Comp_UC: Y
     | -
     | UC
     | 保留副本
   * - SC, SD
     | SC, SD
     | No
     | CompData_UC: -
     | Y
     | UC
     | 丢失副本，获取数据
   * - SC, SD
     | SC, SD
     | No
     | Comp_UD_PD: Y
     | -
     | UD
     | 获取脏责任
   * - (任意)
     | I
     | Yes
     | CompData_UC: Y
     | Y
     | UC
     | 被无效化 Snoop 丢失副本
   * - (任意)
     | I
     | Yes
     | CompData_UD_PD: Y
     | Y
     | UD
     | 被无效化 + 获取脏数据

Exclusive 变体额外状态转换：

.. list-table::
   :header-rows: 1
   :widths: 15 12 12 12 12 25

   * - 初始
     | 响应时
     | Invalid Snoop
     | 最终状态
     | 说明
   * - SC, SD
     | SC, SD
     | No
     | SC
     | Exclusive 检查失败，保留 Shared
   * - SC, SD
     | I
     | Yes
     | SC
     | 丢失副本但 Exclusive 失败

2 Dataless 事务状态转换
---------------------------

.. list-table::
   :header-rows: 1
   :widths: 25 15 15 15 30

   * - 事务
     | 初始状态
     | 允许初始
     | 最终状态
     | 说明
   * - **CleanUnique**
     | I
     | UC, UCE, SC, SD
     | UC, UCE
     | 从 SC/SD 获取时，脏数据写回
   * - **MakeUnique**
     | I, SC, SD
     | UC, UCE
     | UC
     | 脏数据**丢弃不传**（保证写全部字节）
   * - **Evict**
     | I
     | - (必须先转 I)
     | I
     | 仅 Clean 行
   * - **CleanShared***
     | I, SC, UC
     | - (必须先转 I)
     | **不变**
     | CMO 不改变缓存状态
   * - **CleanInvalid**
     | I
     | - (必须先转 I)
     | I
     | 脏数据必须写回
   * - **MakeInvalid**
     | I
     | - (必须先转 I)
     | I
     | 脏数据丢弃
   * - **StashOnce***
     | I
     | - (必须先转 I)
     | I
     | Stash 操作

3 Write 事务状态转换
-------------------------

.. list-table::
   :header-rows: 1
   :widths: 25 15 15 15 30

   * - 事务
     | 初始状态
     | WB 前状态
     | 最终状态
     | 说明
   * - WriteNoSnp*
     | I
     | I
     | I
     | Non-snoopable 写入
   * - WriteUnique*
     | I
     | I
     | I
     | Snoopable 写入
   * - WriteBackFull
     | UD/SD
     | UD/SD
     | I
     | CopyBack。Home 可返回 Comp（不接收数据）
   * - WriteCleanFull
     | UD/SD
     | UC/SC
     | **UC/SC**
     | **唯一最终非 I 的 Write**。UD->UC; SD->SC
   * - WriteEvictFull
     | UD/SD
     | UD/SD
     | I
     | 写回 + 驱逐
   * - WriteEvictOrEvict
     | UD/SD
     | I (Clean时)
     | I
     | UD/SD 写回; UC/SC 直接驱逐

4 Atomic 事务状态转换
-------------------------

所有 Atomic 事务的最终状态均为 **I**。Requester 不缓存返回的数据。

----------------------------
Snoopee 端缓存状态转换
----------------------------

*源文档: B4.8 Cache state transitions at a Snoopee*

1 SnpOnce / SnpOnceFwd
---------------------------

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 40

   * - Snoop 类型
     | 初始状态
     | 最终状态
     | 说明
   * - SnpOnceFwd
     | UC
     | I
     | 转发数据给 Requester
   * - SnpOnceFwd
     | UD
     | I
     | 转发数据 + 脏责任传递
   * - SnpOnceFwd
     | SC
     | SC
     | 不变（未选中转发）
   * - SnpOnceFwd
     | SD
     | SD
     | 不变
   * - SnpOnce
     | UC
     | UC
     | 不变
   * - SnpOnce
     | UD
     | UD
     | 不变
   * - SnpOnce
     | SC
     | SC
     | 不变
   * - SnpOnce
     | SD
     | SD
     | 不变

2 SnpClean / SnpCleanFwd
-----------------------------

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 40

   * - Snoop
     | 初始
     | 最终
     | 说明
   * - SnpCleanFwd
     | UD
     | SC
     | 脏数据必须写回，转发 Clean
   * - SnpCleanFwd
     | SD
     | SC
     | 转发 Clean
   * - SnpCleanFwd
     | UC, SC
     | SC
     | 不变（已是 Clean）
   * - SnpClean
     | UD
     | SC
     | 脏数据写回
   * - SnpClean
     | SD
     | SC
     | 变 Clean
   * - SnpClean
     | UC, SC
     | SC
     | 不变

3 SnpShared / SnpSharedFwd
-------------------------------

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 40

   * - Snoop
     | 初始
     | 最终
     | 说明
   * - SnpSharedFwd
     | UD
     | SD
     | 转发数据，保持 Shared
   * - SnpSharedFwd
     | SD
     | SD
     | 不变
   * - SnpSharedFwd
     | UC, SC
     | SC
     | 不变
   * - SnpShared
     | UD
     | SD
     | 变为 Shared Dirty
   * - SnpShared
     | SD
     | SD
     | 不变
   * - SnpShared
     | UC, SC
     | SC
     | 不变

4 SnpUnique / SnpUniqueFwd
-------------------------------

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 40

   * - Snoop
     | 初始
     | 最终
     | 说明
   * - SnpUniqueFwd
     | UC, UD, SC, SD
     | **I**
     | 转发数据给 Requester，无效化
   * - SnpUnique
     | UC, UD, SC, SD
     | **I**
     | 无效化，数据发送给 Home

5 SnpCleanShared
--------------------

.. list-table::
   :header-rows: 1
   :widths: 15 15 15 55

   * - 初始
     | 最终
     | 条件
     | 说明
   * - UD
     | SC
     | RetToSrc=1 时发送数据
     | 脏数据写回，变为 Shared Clean
   * - SD
     | SC
     | RetToSrc=1 时发送数据
     | 变 Clean
   * - UC, SC
     | SC
     | -
     | 不变

6 SnpCleanInvalid
----------------------

.. list-table::
   :header-rows: 1
   :widths: 15 15 15 55

   * - 初始
     | 最终
     | 数据行为
     | 说明
   * - UD
     | **I**
     | 必须发送数据
     | 脏数据写回 + 无效化
   * - SD
     | **I**
     | 必须发送数据
     | 写回 + 无效化
   * - UC, SC
     | **I**
     | -
     | 直接无效化

7 SnpMakeInvalid
---------------------

.. list-table::
   :header-rows: 1
   :widths: 15 15 15 55

   * - 初始
     | 最终
     | 数据行为
     | 说明
   * - UD
     | **I**
     | **禁止返回数据**
     | **脏数据丢弃**
   * - SD
     | **I**
     | **禁止返回数据**
     | **脏数据丢弃**
   * - UC, SC
     | **I**
     | -
     | 直接无效化

.. danger::

   SnpMakeInvalid 是**唯一允许丢弃脏数据的 Snoop**，使用必须极其谨慎。

8 SnpQuery
--------------

.. list-table::
   :header-rows: 1
   :widths: 15 15 40

   * - 初始
     | 最终
     | 说明
   * - UC, UD, UCE, UDP
     | **不变**
     | 保持原状态
   * - SC, SD
     | **不变**
     | 保持原状态
   * - I
     | **I**
     | 不在缓存中

9 SnpPreferUnique / SnpPreferUniqueFwd
------------------------------------------

SnpPreferUnique 的最终状态取决于是否存在活跃的独占序列：

.. list-table::
   :header-rows: 1
   :widths: 20 15 15 50

   * - Snoop
     | 初始
     | 最终
     | 说明
   * - SnpPreferUnique
     | UC, UD, SC, SD
     | **I** 或 **不变**
     | 取决于独占序列状态
   * - SnpPreferUniqueFwd
     | UC, UD, SC, SD
     | **I** 或 **不变**
     | 转发 + 取决于独占序列

----------------------------------
RetToSrc 与 Data Pull 规则
----------------------------------

*源文档: B4.9 Returning Data with Snoop response*

1 RetToSrc 字段
-------------------

RetToSrc 指示 Snoop 接收方在 Snoop 响应中返回数据：

.. list-table::
   :header-rows: 1
   :widths: 20 15 65

   * - RetToSrc
     | 适用场景
     | 说明
   * - **必须为 0**
     | SnpStashUnique, SnpStashShared, SnpMakeInvalid, SnpQuery
     | 这些 Snoop 明确禁止数据返回
   * - **必须为 1**
     | Forwarding Snoop (除 DataPull 场景)
     | 转发给 Requester 需要数据
   * - **可选**
     | SnpOnce*, SnpClean*, SnpShared*, SnpCleanInvalid
     | Home 决定是否需要数据

2 SC 状态的 RetToSrc 行为
------------------------------

当缓存行处于 SC 状态时：

- **RetToSrc = 0**: Snoopee **禁止**返回数据（即使 Snoop 请求了数据）
- **RetToSrc = 1**: Snoopee **期望**返回数据

3 Data Pull
---------------

Data Pull 允许 Snoopee 在 Snoop 响应中**隐含发送 Read 请求**。Data Pull 的 Read 请求被视作：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Snoop 类型
     | DataPull 等效的 Read 类型
   * - SnpStashUnique
     | ReadUnique（获取 Unique 副本到 Stash 目标）
   * - SnpStashShared
     | ReadNotSharedDirty（获取 Shared 副本到 Stash 目标）
   * - SnpUniqueStash
     | ReadUnique
   * - SnpMakeInvalidStash
     | - (仅 SnpResp_I_Read)

----------------------------------
DoNotGoToSD 规则
----------------------------------

*源文档: B4.10 Do not transition to SD*

**DoNotGoToSD** 字段控制 Snoopee 是否可以转换到 SD 状态：

1 强制转换禁止
------------------

当 DoNotGoToSD = 1 时：

.. list-table::
   :header-rows: 1
   :widths: 25 15 15 45

   * - Snoop
     | 初始状态
     | 被禁止的转换
     | 替代行为
   * - SnpShared
     | UD
     | UD -> SD
     | 保持 UD
   * - SnpShared
     | SD
     | 保持 SD
     | -
   * - SnpSharedFwd
     | UD
     | UD -> SD
     | 必须返回数据（变 SC）
   * - SnpSharedFwd
     | SD
     | 保持 SD
     | 必须返回数据

2 非强制转换限制
----------------------

DoNotGoToSD 不影响以下转换：

- 静默的 UC -> SC 转换（Local sharing）
- SnpShared 的 SC -> SC 转换

3 例外情况
-------------

当 DoNotGoToSD = 1 且缓存行处于 UD 状态，如果 Snoopee 无法保留数据（如资源限制），则**不允许**返回数据，事务必须被 Retry 或以其他方式处理。

----------------------------------
多拷贝原子性与排序
----------------------------------

*源文档: B2.6 Ordering*

1 Multi-copy atomicity
---------------------------

所有合规组件必须确保所有写请求满足多拷贝原子性：

- 对同一位置的所有写入被**序列化**，所有 Requester 以相同顺序观察
- 对某位置的读**不返回**尚未被所有 Requester 观察到的写值

2 Completion Response 的排序保证
--------------------------------------

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - 事务类型
     | 响应
     | 排序保证
   * - Read (Cacheable)
     | CompData/DataSepResp
     | 事务对同一位置的后序事务可观察
   * - Read (Cacheable)
     | RespSepData
     | 无更早事务发送 Snoop 给此 Requester；后序 Snoop 仅在 Home 收到此事务的 CompAck 后才发送
   * - Read (Non-cacheable/Device)
     | CompData/RespSepData
     | 事务对同一端点地址范围的后序事务可观察
   * - Write/Atomic (Cacheable)
     | Comp/CompData
     | 事务对同一位置的后序事务可观察
   * - Dataless
     | Comp
     | 事务对同一内存位置的后序事务可观察
   * - CleanSharedPersist
     | Comp
     | 之前写入同一位置的所有数据被持久化
   * - CleanInvalidPoPA
     | Comp
     | 事务对任意 PAS 中同一位置的后序事务可观察
   * - Combined Write
     | CompCMO
     | CMO/PCMO/写操作对后序事务可观察
   * - StashOnceSep
     | StashDone
     | 事务对同一内存位置的后序事务可观察

3 CompAck 机制
-------------------

CompAck (Completion Acknowledgment) 控制同一 Requester 发出的多个事务与由不同 Requester 的事务导致的 Snoop 之间的相对排序。

**时序保证**:

1. RN-F 在收到 Comp/RespSepData/CompData 后发送 CompAck
2. HN-F（除 ReadNoSnp/ReadOnce* 外）**等待 CompAck** 后才发送后续同地址 Snoop
3. 对于 CopyBack 事务，WriteData 作为隐式 CompAck

**这意味着**: RN-F 收到完成和同一缓存行的 Snoop 的顺序与 HN-F 发送它们的顺序相同。

**CompAck 需求表**:

.. list-table::
   :header-rows: 1
   :widths: 25 15 15 15 30

   * - 事务类型
     | RN-F
     | RN-D/RN-I
     | 说明
   * - ReadNoSnp
     | O
     | O
     | Optional; DMT 时 Required
   * - ReadOnce*
     | O
     | O
     | Optional; DMT 时 Required
   * - ReadClean/NotSharedDirty/Shared/Unique/PreferUnique
     | **Y**
     | O
     | Required (RN-F)
   * - MakeReadUnique
     | **Y**
     | O
     | Required (RN-F)
   * - CleanUnique/MakeUnique
     | **Y**
     | -
     | Required (RN-F)
   * - CleanShared*/CleanInvalid*/MakeInvalid
     | N
     | N
     | 不允许
   * - WriteBack/WriteClean/WriteEvict*
     | H
     | -
     | 取决于 Home 响应
   * - WriteUnique/WriteNoSnp
     | O
     | O
     | OWO 时 Required
   * - Atomic
     | N
     | N
     | 不允许
   * - Evict/StashOnce*
     | N
     | N
     | 不允许

4 Order 字段编码
----------------------

.. list-table::
   :header-rows: 1
   :widths: 15 30 55

   * - Order[1:0]
     | 类型
     | 说明
   * - 0b00
     | No ordering
     | 无排序要求
   * - 0b01
     | Request Accepted
     | Completer 确认接受（仅 HN-F->SN-F 和 HN-I->SN-I）
   * - 0b10
     | Request Order / OWO
     | RN->HN: ExpCompAck=0 为 Request Order; ExpCompAck=1 为 OWO
   * - 0b11
     | Endpoint Order
     | 同一端点地址范围的排序

5 Streaming Ordered Write (OWO)
------------------------------------

OWO 是一种高效流式有序写入机制：

- Requester 设置 **Order=0b10 + ExpCompAck**
- 请求者等待 DBIDResp/DBIDRespOrd 后发送下一个有序写入
- HN-F 在收到 CompAck 后才使写入对其他 Observer 可见
- **优化的 OWO**: 如果写入到不同目标，Requester 不需等待 DBIDResp*

.. important::

   OWO 的关键约束：写入的完成**不依赖于**后续写入的完成。HN-F 可以乱序完成 OWO 写入，但通过 CompAck 机制保证观察顺序。

----------------------------------
Hazard Conditions
----------------------------------

*源文档: B4.11 Hazard conditions*

1 RN-F Hazard 场景
----------------------

当 RN-F 同时有同一缓存行的多个事务进行时可能产生 Hazard：

1. **CopyBack 进行中 + 同行 Read 被发出**: Atomic (SnoopMe=1) 可以在 CopyBack 完成前发出；CopyBack 也可以在 Atomic 完成前发出
2. **CopyBack 进行中 + Snoop 到达**: Home 必须正确处理
3. **Read 进行中 + Snoop 到达**: 通过 CompAck 机制保证正确排序

2 HN-F Hazard 场景
-----------------------

Home Node 作为 PoC 和 PoS 必须处理以下场景：

1. **同一地址多个 Snoop 进行中**: 必须正确合并 Snoop 响应
2. **Snoop 与 Dataless 冲突**: 静默状态转换可能导致不一致
3. **Backward Invalidation**: Home 自发产生 Snoop 时的时序

HN-F 的关键处理规则：

- 必须在所有 Snoop 响应收集后才发送 Comp 给 Requester
- 被无效化的缓存行的脏数据必须正确写回或传递
- Forwarding Snoop 与非 Forwarding Snoop 的响应必须正确合并

----------------------------------
协议设计总结
----------------------------------

1 一致性协议的核心机制
---------------------------

CHI 一致性协议通过以下核心机制实现硬件一致性：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 机制
     | 描述
   * - **Home Node 协调**
     | 所有 Snoopable 事务由 HN-F 协调，HN-F 作为 PoC 管理一致性
   * - **Snoop 机制**
     | 通过向 Peer RN-F 发送 Snoop 请求来维护一致性
   * - **缓存状态机**
     | 7 种缓存状态 + 严格的转换规则确保数据一致性
   * - **Pass Dirty**
     | 脏责任通过 _PD 响应在缓存间传递，确保最终写回内存
   * - **CompAck**
     | 完成确认机制保证事务排序和多拷贝原子性
   * - **Multi-copy atomicity**
     | 所有写操作被序列化，所有 Reader 以相同顺序观察

2 性能优化特性
-------------------

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 优化
     | 描述
   * - **DCT (Direct Cache Transfer)**
     | Peer 缓存直接转发数据给 Requester，减少一跳
   * - **DMT (Direct Memory Transfer)**
     | SN 直接发送数据给 Requester，减少一跳
   * - **Snoop Filter / Directory**
     | 减少 Snoop 范围，避免广播
   * - **Write-Invalidate**
     | 比 Write-Update 更高效（减少数据传输）
   * - **Stash**
     | 提前将数据放到预期使用位置
   * - **ReadPreferUnique**
     | 优化独占-共享竞争场景
   * - **Streaming OWO**
     | 高效流式有序写入
   * - **Empty cache line (UCE/UDP)**
     | 减少不必要的数据传输
   * - **MakeReadUnique**
     | 避免不必要的内存读取（已有副本时）
   * - **CleanInvalidPoPA**
     | 支持跨物理地址空间的一致性操作
   * - **Combined Write + CMO**
     | 减少事务数量

3 事务类型与缓存状态的关系图谱
--------------------------------------

::

  Non-snoopable:
    ReadNoSnp ──────────────────> I
    WriteNoSnp* ────────────────> I

  Non-allocating Snoopable Read:
    ReadOnce* ──────────────────> I

  Allocating Read:
    ReadClean ──────────────────> SC, UC
    ReadNotSharedDirty ────────> SC, UC, UD (no SD)
    ReadShared ────────────────> SC, UC, UD, SD
    ReadUnique ────────────────> UC, UD
    ReadPreferUnique ──────────> UC, UD, SC, SD
    MakeReadUnique ─────────────> UC, UD (special)

  Dataless:
    CleanUnique ────────────────> UC, UCE
    MakeUnique ─────────────────> UC
    Evict ──────────────────────> I
    CleanShared* ──────────────> (no change)
    CleanInvalid* ──────────────> I
    MakeInvalid ────────────────> I

  Write:
    Immediate Write ─────────────> I
    WriteCleanFull ──────────────> UC, SC (unique!)
    CopyBack Write ──────────────> I
    Combined Write ──────────────> I

  Atomic:
    All ─────────────────────────> I (no caching)

  Stash:
    StashOnce* ──────────────────> I
