SMMU 学习笔记
==============

本文档记录 SMMU (System Memory Management Unit) 相关问题的学习笔记，
参考文档：IHI0070G.a - System Memory Management Unit Architecture Specification。

.. contents::

SMMU Stage 1 和 Stage 2 转换有什么区别
========================================

**问题**：SMMU stage 1 和 stage 2 转换有什么区别？

**解答**：

Stage 1 和 Stage 2 是 SMMU 的两阶段地址转换，类似于 ARMv8-A 虚拟化扩展中 PE 的两阶段转换。每个阶段可以独立启用。

Stage 1: VA -> IPA
------------------

**用途**：由软件实体（如 Guest OS）使用，为设备 DMA 提供隔离或地址翻译。

- **数据结构**：Context Descriptor (CD) 中存储 stage 1 翻译表基址、ASID
- **配置入口**：STE 的 ``S1ContextPtr`` 字段指向 CD
- **典型场景**：OS 内部的 DMA 隔离、不同进程/驱动的设备地址空间隔离
- **SubstreamID**：由 StreamID 关联的多个 CD 组成，SubstreamID 用于选择具体的 stage 1 上下文（区分同一设备的不同应用）
- **管理者**：由 **Guest OS** 管理 CD 和 stage 1 翻译表

Stage 2: IPA -> PA
------------------

**用途**：在支持虚拟化扩展的系统中，将设备 DMA 虚拟化到 Guest VM 的地址空间。

- **数据结构**：Stream Table Entry (STE) 中直接存储 stage 2 翻译表基址、VMID
- **配置入口**：STE 内嵌 stage 2 配置，无需额外数据结构
- **典型场景**：Hypervisor 将设备分配给 VM，控制 VM 看到的物理地址空间
- **多设备共享**：多个设备（多个 STE）可以共享同一套 stage 2 翻译表，因为它们属于同一个 VM
- **管理者**：由 **Hypervisor** 管理 STE 和 stage 2 翻译表

嵌套模式（Nested）
------------------

当两个阶段都启用时，地址转换流程为::

    VA --[Stage 1]--> IPA --[Stage 2]--> PA

- CD 从 **IPA 空间**获取（需要经过 stage 2 翻译）
- Stage 1 翻译表的页表遍历可能也需要经过 stage 2 翻译
- 每个 StreamID 只能对应一个 stage 2 上下文，但可以对应多个 stage 1 上下文（通过 SubstreamID）

对比总结
--------

.. list-table::
   :header-rows: 1
   :widths: 20 40 40

   * -
     - Stage 1
     - Stage 2
   * 转换
     - VA -> IPA
     - IPA -> PA
   * 数据结构
     - Context Descriptor (CD)
     - Stream Table Entry (STE)
   * 标识符
     - ASID
     - VMID
   * 管理者
     - Guest OS
     - Hypervisor
   * 目的
     - 进程/驱动级 DMA 隔离
     - VM 级设备虚拟化
   * 可选性
     - 可独立启用（``SMMU_IDR0.S1P``）
     - 可独立启用（``SMMU_IDR0.S2P``）

参考：IHI0070G Chapter 2 Introduction 和 Chapter 3 Operation。

SMMU STE 和 CD 的数据结构是怎样的
====================================

**问题**：SMMU STE (Stream Table Entry) 和 CD (Context Descriptor) 的数据结构是怎样的？

**解答**：

STE 和 CD 是 SMMUv3 中两个核心的内存配置结构，均为 **64 字节（512 位）**，小端字节序。

STE (Stream Table Entry) — 64 字节
------------------------------------

**用途**：每个 stream（设备）的配置结构，由 StreamID 索引，描述设备的翻译配置。

关键字段：

.. list-table::
   :header-rows: 1
   :widths: 18 12 70

   * - 字段
     - 位置
     - 说明
   * - ``V``
     - [0]
     - 有效位。0=无效（终止事务，记录 C_BAD_STE），1=有效
   * - ``Config[3:1]``
     - [3:1]
     - 流配置。``0b100``=全部旁路，``0b101``=仅 Stage 1，``0b110``=仅 Stage 2，``0b111``=嵌套（Stage 1 + Stage 2）
   * - ``S1Fmt[5:4]``
     - [5:4]
     - Stage 1 格式。``0b00``=线性 CD 表，``0b01``=2 级表（4KB L2），``0b10``=2 级表（64KB L2）
   * - ``S1ContextPtr[55:6]``
     - [55:6]
     - Stage 1 CD 指针。嵌套模式下为 IPA（经 Stage 2 翻译），否则为 PA
   * - ``S1CDMax[63:59]``
     - [63:59]
     - CD 数量的 log2 值。0=单个 CD（无 substream），N=2^N 个 CD
   * - ``S1DSS[65:64]``
     - [65:64]
     - 默认 substream 行为。``0b00``=终止，``0b01``=旁路 Stage 1，``0b10``=使用 substream 0 的 CD
   * - ``S2VMID[143:128]``
     - [143:128]
     - Stage 2 VMID（16 位）
   * - ``S2TTB[247:224]``
     - [247:224]
     - Stage 2 翻译表基址（IPA 或 PA）
   * - ``S2AA64``
     - [169]
     - Stage 2 地址格式。0=VMSAv8-32 LPAE，1=VMSAv8-64
   * - ``S2R``
     - [163]
     - Stage 2 输出 RA 状态覆盖
   * - ``S2S``
     - [166]
     - Stage 2 Stall 配置
   * - ``S2HA`` / ``S2HD``
     - [167] / [168]
     - Stage 2 Access Flag / Dirty state 硬件更新
   * - ``EATS[93:92]``
     - [93:92]
     - ATS 使能。``0b00``=禁用，``0b01``=Full ATS，``0b10``=Split-stage ATS
   * - ``STRW[95:94]``
     - [95:94]
     - StreamWorld 控制。决定 stage 1 翻译的 Exception Level（EL1/EL2/EL3）
   * - ``VMSPtr[351:332]``
     - [351:332]
     - Virtual Machine Structure (VMS) 指针

**Stage 2 相关字段**：以 ``S2`` 前缀标识，当 ``STE.Config`` 指示 Stage 2 旁路时被忽略。
**Stage 1 相关字段**：以 ``S1`` 前缀标识，当 ``STE.Config`` 指示 Stage 1 旁路时被忽略。
**通用字段**：无前缀，与具体翻译阶段无关（如 ``V``、``Config``、``STRW``）。

CD (Context Descriptor) — 64 字节
------------------------------------

**用途**：Stage 1 翻译的配置结构，包含翻译表基址、ASID 和翻译体制信息。由 STE 的 ``S1ContextPtr`` 指向。

关键字段：

.. list-table::
   :header-rows: 1
   :widths: 18 12 70

   * - 字段
     - 位置
     - 说明
   * - ``V``
     - [31]
     - 有效位。0=无效（记录 C_BAD_STE），1=有效
   * - ``ASID[63:48]``
     - [63:48]
     - Address Space ID（16 位），标识 TLB 条目的地址空间
   * - ``AA64``
     - [35]
     - 地址格式。0=VMSAv8-32 LPAE，1=VMSAv8-64
   * - ``T0SZ[5:0]``
     - [5:0]
     - TTB0 的 VA 区域大小（等效于 ``TCR_ELx.T0SZ``）
   * - ``TG0[7:6]``
     - [7:6]
     - TTB0 翻译粒度。``0b00``=4KB，``0b01``=64KB，``0b10``=16KB
   * - ``IR0[9:8]``
     - [9:8]
     - TTB0 内部区域缓存属性
   * - ``OR0[11:10]``
     - [11:10]
     - TTB0 外部区域缓存属性
   * - ``SH0[13:12]``
     - [13:12]
     - TTB0 共享性。``0b00``=非共享，``0b10``=外部共享，``0b11``=内部共享
   * - ``EPD0``
     - [14]
     - TTB0 翻译表遍历禁用。0=启用，1=禁用（TLB miss 产生 Translation fault）
   * - ``TTB0[119:96]``
     - [119:96]
     - Stage 1 低地址范围翻译表基址（VA -> IPA）
   * - ``EPD1``
     - [30]
     - TTB1 翻译表遍历禁用（高地址范围）
   * - ``T1SZ[21:16]``
     - [21:16]
     - TTB1 的 VA 区域大小
   * - ``TG1[23:22]``
     - [23:22]
     - TTB1 翻译粒度。编码与 TG0 不同：``0b01``=16KB，``0b10``=4KB，``0b11``=64KB
   * - ``TTB1[183:160]``
     - [183:160]
     - Stage 1 高地址范围翻译表基址
   * - ``IPS[34:32]``
     - [34:32]
     - IPA 大小。``0b000``=32 位 ~ ``0b101``=48 位
   * - ``MAIR0[223:192]``
     - [223:192]
     - 内存属性间接寄存器 0（Stage 1 页属性索引）
   * - ``MAIR1[255:224]``
     - [255:224]
     - 内存属性间接寄存器 1
   * - ``PARTID[375:368]``
     - [375:368]
     - MPAM PARTID（SMMUv3.2+）
   * - ``PMG[383:376]``
     - [383:376]
     - MPAM PMG（SMMUv3.2+）

**CD 中 ``TTB0`` 和 ``TTB1`` 的关系**：

- ``TTB0`` 覆盖低 VA 范围（如 0x0000_0000_0000_0000 ~ 0x0000_FFFF_FFFF_FFFF）
- ``TTB1`` 覆盖高 VA 范围（如 0xFFFF_0000_0000_0000 ~ 0xFFFF_FFFF_FFFF_FFFF）
- 各自有独立的 ``EPD`` 控制是否启用
- ``EPD0``/``EPD1`` 仅在 StreamWorld 为 EL1 时有效，EL2/EL3 时被忽略

配置查找与翻译流程
--------------------

一个事务的完整处理流程：

1. **配置查找**（Configuration lookup）
   - StreamID 索引 Stream Table，获取 STE
   - 若 Stage 1 启用，通过 ``STE.S1ContextPtr`` 获取 CD（嵌套模式下 CD 从 IPA 空间获取，经 Stage 2 翻译）
2. **翻译查找**（Translation lookup）
   - 若 Stage 1 启用：CD 中的 ``TTB0``/``TTB1`` 提供 Stage 1 翻译表基址，执行 VA -> IPA
   - 若 Stage 2 启用：STE 中的 ``S2TTB`` 提供 Stage 2 翻译表基址，执行 IPA -> PA
   - 两阶段级联：VA --[Stage 1]--> IPA --[Stage 2]--> PA

参考：IHI0070G Chapter 3 Operation 和 Chapter 5 Data structure formats（5.2 STE 和 5.4 CD）。

SMMU 在虚拟化里的用法
======================

**问题**：SMMU 在虚拟化里都有哪几种用法？

**解答**：

SMMU 在虚拟化场景中主要有以下几种用法，由 ``STE.Config`` 字段控制：

用法一：Stage 2 Only — 设备直通（Passthrough）
---------------------------------------------

**配置**：``STE.Config = 0b110``（Stage 2 Translate，Stage 1 Bypass）

设备可以直接分配给 Guest VM 使用，无需 Guest 与 SMMU 交互。

- Hypervisor 为设备配置 STE，设置 Stage 2 翻译表基址和 VMID
- Stage 1 处于旁路状态，设备发出的地址直接作为 IPA 输入 Stage 2
- Guest OS 看到的设备就像直接连接在系统总线上，设备可以直接用 IPA（Guest 认为是 PA）做 DMA
- Guest 不需要知道 SMMU 的存在，也不需要 SMMU 驱动

**适用场景**：

- 简单的设备直通，如将网卡、磁盘控制器直接分配给 VM
- Guest 期望使用 40 位 DMA 地址（VMSAv8-32 LPAE），Stage 2 旁路 IAS 可以大于 PA，支持此场景
- 不需要进程级 DMA 隔离的设备

**关键点**：

- 多个设备可以共享同一套 Stage 2 翻译表（属于同一 VM）
- SubstreamID/PASID 不能使用（Stage 1 旁路时发送 SubstreamID 会导致事务终止）
- Hypervisor 完全控制 Stream Table 和 Stage 2 翻译表

用法二：Nested（Stage 1 + Stage 2）— Hypervisor 仿真 SMMU
-----------------------------------------------------------

**配置**：``STE.Config = 0b111``（Stage 1 Translate + Stage 2 Translate）

Guest OS 需要使用 Stage 1 翻译（如进程级 DMA 隔离），但 SMMU 不提供直接供 VM 使用的编程接口。Hypervisor 通过仿真虚拟 SMMU 的方式为 Guest 提供完整的 SMMU 编程体验。

**工作原理**：

- Guest OS 认为自己控制一个真实的 SMMU（Stage 1 Only），拥有自己的 Command Queue、Event Queue 和 PRI Queue
- 实际上 Hypervisor 充当中间层，将 Guest 的虚拟 SMMU 操作转换为物理 SMMU 操作
- Hypervisor 维护物理 Stream Table 和 Stage 2 翻译表
- Guest 管理的 CD 和 Stage 1 翻译表以 IPA 编址，由 Stage 2 翻译到 PA

**Hypervisor 的职责**：

- **STE 转换**：将 Guest 的虚拟 STE 转换为物理 SMMU STE，控制权限和功能
- **StreamID 映射**：物理 StreamID 可能对 Guest 不可见，Guest 使用虚拟 StreamID，Hypervisor 维护映射关系
- **命令代理**：读取并解释 Guest Command Queue 中的命令，转换为物理 SMMU 命令或更新内部影子数据结构
- **事件转发**：从 Host Event Queue 和 PRI Queue 中消费条目，将 Host StreamID 映射为 Guest StreamID，投递到 Guest 的 Event/PRI 队列

**适用场景**：

- 需要进程级 DMA 隔离的设备直通（如 GPU SR-IOV，多个进程共享同一 PCIe Function）
- 使用 PASID/SubstreamID 区分同一设备的不同 DMA 上下文
- Guest 需要使用 ATS 和 PRI 进行页式 DMA

**故障处理模型**：

.. list-table::
   :header-rows: 1
   :widths: 20 20 20 20 20

   * - Stage 1
     - Stage 2
     - 行为
     - Guest 视角
   * - Fault
     - -
     - Terminate
     - 收到 Stage 1 Only 事件
   * - Fault
     - -
     - Stall
     - 收到 Stage 1 Stall 事件，Guest 需发送 CMD_RESUME
   * - OK
     - Fault
     - Terminate
     - Hypervisor 可记录 IPA 用于调试，可能作为 Stage 1 外部中止传递给 Guest
   * - Stall
     - Fault
     - Stall (Stage 1)
     - Guest 收到 Stage 1 Stall 事件

**关键点**：

- ``STE.S1STALLD`` 可以禁止 Guest 使用 Stall 模式，防止 stalled 事务影响其他 VM
- ``STE.EATS`` 控制 ATS 行为：``0b01``=Full ATS，``0b10``=Split-stage ATS（ATS 返回 IPA，后续 Translated 事务再经 Stage 2 翻译）
- Split-stage ATS 允许使用 ATS/PRI 的同时保持 Stage 2 隔离
- ``CD.ASET`` 标记 TLB 条目是否参与广播 TLB 失效，Guest 使用的非共享上下文不应匹配全局翻译

用法三：Stage 1 Only — 裸金属 OS
-------------------------------

**配置**：``STE.Config = 0b101``（Stage 1 Translate，Stage 2 Bypass）

非虚拟化场景，裸金属 OS 同时管理 Stream Table、CD 和 Stage 1 翻译表。

- S1ContextPtr 和翻译表基址为 PA（不经 Stage 2 翻译）
- 没有 Hypervisor 参与，OS 直接使用 SMMU

**适用场景**：

- 不使用虚拟化的嵌入式系统
- OS 直接管理所有设备的 DMA 地址翻译和隔离

用法四：Hypervisor 自用 Stage 1
-------------------------------

Hypervisor 自身也需要使用设备 DMA，此时 Hypervisor 使用独立的 Stage 1 翻译上下文。

- 通过 ``STE.STRW`` 字段选择 StreamWorld 为 NS-EL2 或 NS-EL2-E2H
- Hypervisor 有自己的 CD 和 Stage 1 翻译表，与 Guest 隔离
- 这些 CD 以 PA 编址，不受 Stage 2 控制

**适用场景**：

- Hypervisor 直接使用硬件设备进行 DMA 操作
- 管理网络、存储等基础设施设备

用法五：Full Bypass — 不翻译
-----------------------------

**配置**：``STE.Config = 0b100``（Stage 1 Bypass + Stage 2 Bypass）

设备的 DMA 事务直接通过 SMMU，不做任何地址翻译。

- 适用于不需要 DMA 隔离的场景
- 可以通过 STE 的全局属性字段（如 MTCFG、SHCFG 等）覆盖内存类型和共享性属性

总结
----

.. list-table::
   :header-rows: 1
   :widths: 18 15 15 52

   * - 用法
     - Config
     - 管理者
     - 说明
   * - Full Bypass
     - ``0b100``
     - 裸金属 OS
     - 无翻译，事务直接通过
   * - Stage 1 Only
     - ``0b101``
     - 裸金属 OS
     - VA -> IPA（实际为 PA），进程级 DMA 隔离
   * - Stage 2 Only
     - ``0b110``
     - Hypervisor
     - IPA -> PA，设备直通，Guest 无需感知 SMMU
   * - Nested (1+2)
     - ``0b111``
     - Hypervisor + Guest
     - 完整虚拟化，Guest 使用虚拟 SMMU，Hypervisor 仿真

参考：IHI0070G Chapter 3 Operation（3.6 节、3.8 节、3.9 节）。

SMMU 的 TLB 和 TLB 维护机制
==============================

**问题**：解释 SMMU 的 TLB 和 TLB 维护机制。

**解答**：

SMMU 使用 TLB（Translation Lookaside Buffer）缓存翻译结果，避免每次 DMA 事务都执行翻译表遍历（TTW）。TLB 条目通过标签（Tag）区分不同的地址空间和翻译体制。

TLB 概述
--------

SMMU 中的 TLB 概念与 PE（处理器）的 TLB 类似，缓存的翻译条目按 ``StreamWorld / VMID / ASID / 地址`` 索引。

**SMMU 实现中的 TLB 分布**：

- **集中式 TLB**：位于 SMMU 中央翻译表遍历器内部，可能包含 macro-TLB
- **分布式 TLB**：位于远程设备端口，TLB miss 时向中央单元请求翻译并本地缓存
- **设备内嵌 TLB**：高性能设备（如 GPU、PCIe RC）内部集成 TLB，通过 SMMU 编程接口获取翻译

``StreamWorld + VMID + ASID + VA  -->  TLB 查找  -->  命中: 直接返回 PA
                                              未命中: 翻译表遍历 (TTW) --> 填充 TLB
```

TLB 标签体系
----------

每个缓存的翻译条目标记有以下标签：

.. list-table::
   :header-rows: 1
   :widths: 15 15 15 55

   * - StreamWorld
     - ASID
     - VMID
     - 说明
   * - NS-EL1
     - 是（nG==1）
     - 是（S2P=1）
     - Guest OS 的标准翻译体制
   * - NS-EL2 / S-EL2 / Realm-EL2
     - 否
     - 否
     - Hypervisor 翻译体制，无 ASID
   * - NS-EL2-E2H / S-EL2-E2H / Realm-EL2-E2H
     - 是（nG==1）
     - 否
     - E2H 模式下支持 ASID
   * - Secure
     - 是（nG==1）
     - 是（SEL2=1）
     - 安全态 EL1 翻译
   * - EL3
     - 否
     - 否
     - 安全监控态，无 ASID

**关键规则**：

- 不同 StreamWorld 的 TLB 条目完全隔离，不会互相匹配
- 同一 VMID 下不同 ASID 的地址空间互相隔离
- ``CD.ASET`` 标志区分共享（ASET=0）和非共享（ASET=1）地址空间，影响广播 TLB 失效行为
- Global 翻译（TTD.nG==0）可以匹配任意 ASID，但仍受 StreamWorld 和 ASET 约束

TLB 维护机制
------------

SMMU 的 TLB 维护通过以下两种方式进行：

### 1. SMMU 命令（CMD_TLBI_*）

通过 Command Queue 发送 TLB 失效命令，仅影响 SMMU 自身的 TLB，不产生广播。在 CMD_TLBI_* 命令序列后必须跟 ``CMD_SYNC`` 确保失效完成。

**Stage 1 TLB 失效命令**：

.. list-table::
   :header-rows: 1
   :widths: 35 30 35

   * - 命令
     - 参数
     - 等效 PE 操作
   * - ``CMD_TLBI_NH_ALL(VMID)``
     - VMID
     - VMALLE1IS：失效指定 VMID 下所有 NS-EL1 条目（含 Global）
   * - ``CMD_TLBI_NH_ASID(VMID, ASID)``
     - VMID, ASID
     - ASIDE1IS：失效指定 VMID+ASID 的非 Global 条目
   * - ``CMD_TLBI_NH_VAA(VMID, Addr, ...)``
     - VMID, VA, TG, TTL, NUM, SCALE
     - VAAE1IS：按地址失效，支持范围失效和层级提示
   * - ``CMD_TLBI_NH_VA(VMID, Addr, ...)``
     - VMID, VA, TG, TTL, NUM, SCALE
     - VAE1IS：按地址+ASID 失效
   * - ``CMD_TLBI_NSNH_ALL``
     - -
     - ALLE1IS：失效所有 NS 条目（不区分 VMID）
   * - ``CMD_TLBI_EL2_ALL``
     - -
     - ALLE2IS：失效所有 EL2 条目
   * - ``CMD_TLBI_EL2_ASID(ASID)``
     - ASID
     - E2ALL: 失效指定 ASID 的 EL2 条目
   * - ``CMD_TLBI_EL2_VA(Addr, ...)``
     - VA, TG, TTL, NUM, SCALE
     - E2VA: 按 EL2 地址失效
   * - ``CMD_TLBI_EL2_VAA(Addr, ...)``
     - VA, TG, TTL, NUM, SCALE
     - E2VAA: 按地址失效（忽略 ASID）
   * - ``CMD_TLBI_EL3_ALL``
     - -
     - ALLE3IS：失效所有 EL3 条目
   * - ``CMD_TLBI_EL3_VA(Addr, ...)``
     - VA, TG, TTL, NUM, SCALE
     - VAE3IS：按 EL3 地址失效

**Stage 2 TLB 失效命令**：

.. list-table::
   :header-rows: 1
   :widths: 35 30 35

   * - 命令
     - 参数
     - 等效 PE 操作
   * - ``CMD_TLBI_S2_ALL(VMID)``
     - VMID
     - ALLE1IS：失效指定 VMID 下所有 Stage 2 条目
   * - ``CMD_TLBI_S2_IPA(VMID, Addr, ...)``
     - VMID, IPA, TG, TTL, NUM, SCALE
     - IPAS2E1IS：按 IPA 地址失效 Stage 2 条目
   * - ``CMD_TLBI_S2_IPA_ALL(VMID, Addr, ...)``
     - VMID, IPA, TG, TTL, NUM, SCALE
     - RIPAS2E1IS：按 IPA 地址失效（Range + TTL hint）
   * - ``CMD_TLBI_NSNH_ALL``
     - -
     - ALLE1IS：失效所有 NS Stage 1 和 Stage 2 条目

### 2. PE 广播 TLB 失效（Broadcast TLB Invalidation）

当 ``SMMU_IDR0.BTM == 1`` 时，SMMU 可以接收 PE 发出的广播 TLB 失效消息（通过 DVM 协议）。

- PE 执行 ``TLBI`` 指令后，通过互连广播到同一 Inner/Outer Shareable 域内的 SMMU
- 广播消息包含地址、ASID、VMID 和翻译体制
- SMMU 根据这些参数匹配并失效对应的 TLB 条目
- ``SMMU_CR2.PTM == 1`` 可禁用指定 Security State 的广播 TLB 失效

**CD.ASET 对广播失效的影响**：

- ``ASET == 0``：ASID 与 PE 进程共享，TLB 条目**必须**被所有匹配的广播失效操作影响
- ``ASET == 1``：ASID 非共享（设备专用地址空间），TLB 条目**不要求**被 ``VA{L}ExIS`` 和 ``ASIDExIS`` 失效，但仍被 ``ALLEXIS`` 影响

### 3. 范围失效与层级提示（Range + TTL）

SMMUv3.2+ 强制支持（``SMMU_IDR3.RIL == 1``），SMMUv3.0/3.1 可选。

**范围失效**：一次命令失效连续的地址范围::

    Range = ((NUM + 1) * 2^SCALE) * Translation_Granule_Size

**层级提示（TTL）**：提示 TLB 条目所在的翻译表层级，缩小失效范围，减少不必要的失效::

    TTL == 0b00: 任意层级
    TTL == 0b01: Level 1 叶条目
    TTL == 0b10: Level 2 叶条目
    TTL == 0b11: Level 3 叶条目

**TG（Translation Granule）** 指定失效的粒度大小：

- ``0b00``：任意粒度（无范围失效，无 TTL）
- ``0b01``：4KB
- ``0b10``：16KB
- ``0b11``：64KB

ATS 与 TLB 的关系
-----------------

当使用 PCIe ATS 时：

- ATS Translation Request 可能直接使用 SMMU TLB 中的条目返回，也可能触发新的翻译表遍历并插入 TLB
- 翻译配置变更时，必须**先执行 SMMU TLB 失效，再执行 ATC 失效**（CMD_ATC_INV）
- ``CMD_SYNC`` 确保 SMMU TLB 失效完成后再发起 ATC 失效

总结
----

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 维护方式
     - 说明
   * - CMD_TLBI_* 命令
     - 通过 Command Queue 发送，仅影响 SMMU TLB，需 CMD_SYNC 确保完成
   * - PE 广播失效
     - PE TLBI 指令通过 DVM 协议广播，SMMU 根据 VMID/ASID/StreamWorld 匹配
   * - CMD_ATC_INV
     - 失效端点 ATC（设备侧 TLB），需先完成 SMMU TLB 失效
   * - CMD_CFGI_*
     - 失效 STE 和 CD 配置缓存（与 TLB 失效是独立的操作）
   * - VMID Wildcards
     - ``SMMU_CR0.VMW == 1`` 时，VMID=0xFFFF 的命令可匹配所有 VMID

参考：IHI0070G Chapter 3 Operation（3.17 节）和 Chapter 4 Commands（4.4 节）。

SMMU 的故障处理和 Event Queue 机制
====================================

**问题**：解释 SMMU 的故障处理和 Event Queue 机制。

**解答**：

故障模型（Fault Models）
-------------------------

当 DMA 事务经过 SMMU 时，可能在配置查找或翻译过程中遇到错误。SMMU 提供两种故障处理模型，按翻译阶段独立配置。

### Terminate 模型

事务遇到故障后立即终止，向设备返回 abort 响应。如果配置了记录标志（``CD.R`` / ``STE.S2R``），则向 Event Queue 写入一条事件记录。

- **Stage 1 终止**：由 ``CD.A`` 和 ``CD.S`` 控制
  - ``CD.S == 0, CD.A == 1``：返回 abort
  - ``CD.S == 0, CD.A == 0``：RAZ/WI 行为（读返回 0，写被忽略）
- **Stage 2 终止**：由 ``STE.S2S == 0`` 控制，始终返回 abort
- **Event Queue 满时**：事件记录丢失（不保证投递）
- PCIe 流必须使用 Terminate 模型（不能 stall），通过 ``STE.S1STALLD == 1`` 强制

### Stall 模型

事务遇到故障后被缓冲（stall），不向设备返回任何响应。SMMU **始终**向 Event Queue 写入事件记录，软件处理后通过 ``CMD_RESUME`` 重试或终止。

- **Stage 1 Stall**：``CD.S == 1`` 且 ``STE.S1STALLD == 0``
- **Stage 2 Stall**：``STE.S2S == 1``
- **Stall Tag (STAG)**：SMMU 为每个 stalled 事务分配唯一 STAG，软件通过 ``CMD_RESUME(StreamID, STAG)`` 定位事务
- **Event Queue 满时**：stall 事件**不丢弃**，队列可写后自动重试并生成新事件
- **重复抑制**：同一 stream 的相同页面、相同类型的后续 stall 事务可能被抑制（不生成新事件）
- **早期重试**：SMMU 可以在收到 ``CMD_RESUME`` 之前投机性重试 stalled 事务
- ``STE.S1STALLD == 1`` 可禁止 Guest VM 使用 Stall 模型，防止 stalled 事务影响其他 VM

### 两阶段组合的故障行为

.. list-table::
   :header-rows: 1
   :widths: 18 18 15 15 34

   * - Stage 1 配置
     - Stage 2 配置
     - 故障阶段
     - 事务结果
     - Hypervisor 处理
   * - Terminate
     - Terminate
     - Stage 1
     - 终止（VA）
     - 作为 S1 事件传递给 Guest
   * - Terminate
     - Terminate
     - Stage 2
     - 终止（VA, IPA）
     - 可能记录 IPA 用于调试
   * - Terminate
     - Stall
     - Stage 1
     - 终止（VA）
     - 作为 S1 事件传递给 Guest
   * - Terminate
     - Stall
     - Stage 2
     - Stall（VA, IPA）
     - CMD_RESUME(Terminate) 或修复翻译后 CMD_RESUME(Retry)
   * - Stall
     - Terminate
     - Stage 1
     - Stall（VA）
     - Guest 发送 CMD_RESUME(Retry/Terminate)
   * - Stall
     - Terminate
     - Stage 2
     - 终止（VA, IPA）
     - 可能记录 IPA 用于调试
   * - Stall
     - Stall
     - Stage 1
     - Stall（VA）
     - Guest 发送 CMD_RESUME(Retry/Terminate)
   * - Stall
     - Stall
     - Stage 2
     - Stall（VA, IPA）
     - Hypervisor 修复 S2 翻译后 CMD_RESUME(Retry)

**注意**：CD/STE 获取失败（``C_BAD_STE``、``C_BAD_CD``）和翻译表遍历外部中止（``F_WALK_EABT``）始终终止事务，不受 Stall 模型控制。

### STALL_MODEL 支持

由 ``SMMU_IDR0.STALL_MODEL`` 指示：

- ``0b00``：同时支持 Stall 和 Terminate（推荐）
- ``0b01``：仅支持 Terminate
- ``0b10``：仅支持 Stall
- ``SMMU_S_CR0.NSSTALLD`` 可以禁止 Non-secure 使用 Stall

Event Queue 机制
----------------

### 概述

Event Queue 是一个内存中的环形缓冲区，SMMU 作为生产者写入事件记录，软件作为消费者读取。每个 Security State 有独立的 Event Queue。

- 每条事件记录 **32 字节**，小端字节序
- 通过 ``SMMU_(*_)CR0.EVENTQEN`` 启用
- ``SMMU_EVENTQ_PROD`` / ``SMMU_EVENTQ_CONS`` 寄存器维护生产/消费指针
- 软件通过轮询 CONS/PROD 或 MSI 中断感知新事件

### 可写条件

Event Queue 必须同时满足以下条件才可写入：

- 队列已启用（``EVENTQEN == 1``）
- 队列未满
- 无未确认的 ``GERROR.EVENTQ_ABT_ERR``

### 三类事件

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 类别
     - 说明
   * - 配置错误（C_*）
     - 寄存器、STE 或 CD 内容不正确。配置存储在内存中，只有在事务触发使用时才报告错误
   * - 翻译故障（F_*）
     - 翻译过程中产生的故障。每笔事务最多产生一个翻译故障事件
   * - 其他（E_*）
     - 异步事件，如 ``E_PAGE_REQUEST`` 页面请求提示

### 事件记录格式（公共字段）

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 字段
     - 说明
   * - StreamID
     - 发起事务的设备 StreamID
   * - SubstreamID + SSV
     - SubstreamID 有效标志及值（仅 SSV==1 时有效）
   * - RnW
     - 0=写，1=读
   * - PnU
     - 特权/非特权（经过 STE 覆盖后的值）
   * - InD
     - 指令/数据（经过 STE 覆盖后的值）
   * - InputAddr[63:0]
     - 输入地址（VA/IPA，取决于事件类型）
   * - S2
     - 故障阶段：0=Stage 1，1=Stage 2
   * - CLASS
     - 操作类别：0b00=CD 获取，0b01=翻译表获取，0b10=输入地址
   * - Stall
     - 是否为 stalled 事务（Stall==1 的事件永不合并）
   * - STAG
     - Stall Tag，用于 CMD_RESUME 定位 stalled 事务

### 完整事件类型列表

.. list-table::
   :header-rows: 1
   :widths: 10 25 65

   * - 编号
     - 名称
     - 说明
   * - 0x01
     - F_UUT
     - 不支持的上游事务（IMPLEMENTATION DEFINED）
   * - 0x02
     - C_BAD_STREAMID
     - StreamID 超出 Stream Table 范围
   * - 0x03
     - F_STE_FETCH
     - STE 获取外部中止
   * - 0x04
     - C_BAD_STE
     - STE 无效（V==0 或配置非法）
   * - 0x05
     - F_BAD_ATS_TREQ
     - ATS Translation Request 失败
   * - 0x06
     - F_STREAM_DISABLED
     - 流被禁用（如 SubstreamID 缺失时 S1DSS==0b00）
   * - 0x07
     - F_TRANSL_FORBIDDEN
     - Translated 流被禁止（ATSCHK 但 EATS 未使能）
   * - 0x08
     - C_BAD_SUBSTREAMID
     - SubstreamID 无效（超出 S1CDMax 范围或 SSV=1 但 S1CDMax=0）
   * - 0x09
     - F_CD_FETCH
     - CD 获取外部中止
   * - 0x0A
     - C_BAD_CD
     - CD 无效（V==0 或配置非法）
   * - 0x0B
     - F_WALK_EABT
     - 翻译表遍历外部中止
   * - 0x0C
     - F_TRANSLATION
     - 翻译故障（页表项无效）
   * - 0x0D
     - F_ADDR_SIZE
     - 地址大小故障（地址超出 VA/IPA 范围）
   * - 0x0E
     - F_ACCESS
     - 访问标志故障（AF==0）
   * - 0x0F
     - F_PERMISSION
     - 权限故障（读写/执行权限不足）
   * - 0x10
     - F_TLB_CONFLICT
     - TLB 冲突
   * - 0x11
     - F_CFG_CONFLICT
     - 配置缓存冲突
   * - 0x12
     - E_PAGE_REQUEST
     - 页面请求提示（可选，仅用于提前通知）
   * - 0x13
     - F_VMS_FETCH
     - VMS 获取外部中止
   * - 0x14-0x1F
     - IMPDEF_EVENTn
     - IMPLEMENTATION DEFINED 事件

### 事件合并（Event Record Merging）

多个相同类型的事件可以合并为一条记录写入 Event Queue，减少队列压力。

- 由 ``STE.MEV`` 控制：``MEV == 0`` 禁止合并（保证一对一映射），``MEV == 1`` 允许合并
- **Stall == 1 的事件永不合并**
- ``F_UUT`` 和 ``C_BAD_STE`` 始终可以被合并（在 STE 定位之前）
- 合并条件：所有已定义字段相同、Stall==0、短时间内连续发生

### Event Queue 溢出处理

- **非 Stall 事件**：队列满时**静默丢弃**，同时触发溢出条件
- **Stall 事件**：队列满时**不丢弃**，队列可写后自动重试事务并写入新事件
- Arm 建议软件及时消费 Event Queue 以避免溢出

### Event Queue 外部中止

写入 Event Queue 时发生外部中止会触发 ``GERROR.EVENTQ_ABT_ERR``，可能导致事件丢失（包括 Stall 事件）。

- **同步中止**：PROD 指针不递增，已写入的条目有效
- **异步中止**：PROD 指针可能递增，所有条目视为无效，软件应清空队列

### 虚拟化中的 Event Queue

- Hypervisor 管理 Host Event Queue，消费条目后将 Host StreamID 映射为 Guest StreamID，投递到 Guest 的虚拟 Event Queue
- Hypervisor 可能将 Stage 2 故障转换为 Stage 1 外部中止传递给 Guest（如 CD 获取阶段的 Stage 2 故障报告为 ``F_WALK_EABT``）
- ``STE.MEV == 0`` 的效果在虚拟化中可能被 Hypervisor 覆盖

参考：IHI0070G Chapter 3 Operation（3.12 节）和 Chapter 7 Faults, errors and Event queue（7.2、7.3 节）。

SMMU 的 ATS 和 PRI 机制
==========================

**问题**：解释 SMMU 的 ATS 和 PRI 机制。

**解答**：

ATS（Address Translation Service）和 PRI（Page Request Interface）是 PCIe 协议提供的两个扩展功能，SMMU 实现对这两个接口的支持以提高 DMA 性能并支持动态分页。ATS 依赖 PRI，但 PRI 不依赖 ATS。

ATS（Address Translation Service）
------------------------------------

### 概述

ATS 允许 PCIe 端点（Endpoint）在发起实际 DMA 事务之前，主动向 SMMU 请求地址翻译，并将翻译结果缓存在端点本地的 ATC（Address Translation Cache，等效于设备侧 TLB）中。后续 DMA 事务可以直接标记为 Translated，携带物理地址，绕过 SMMU 的翻译。

### ATS 工作流程

```
Endpoint                          SMMU                           Memory
   |                               |                               |
   |-- ATS Translation Request --->|                               |
   |   (VA, R/W权限, PASID)        |                               |
   |                               |-- TLB 查找 --> miss?          |
   |                               |-- 翻译表遍历 (TTW) ---------->|
   |                               |<-- 翻译结果 (PA, 权限) -------|
   |                               |-- 插入 SMMU TLB              |
   |<-- ATS Translation Completion-|                               |
   |   (PA, 实际授予的权限)         |                               |
   |                               |                               |
   |-- Translated DMA (PA) --------|------- (ATSCHK=0) ----------->|
   |   (标记为 Translated,          |                               |
   |    绕过 SMMU 翻译)             |                               |
```

### ATS 使能控制

通过 ``STE.EATS[93:92]`` 字段控制：

.. list-table::
   :header-rows: 1
   :widths: 15 85

   * - EATS 值
     - 行为
   * - ``0b00``
     - ATS 禁用。Translation Request 返回 UR，记录 ``F_BAD_ATS_TREQ``。若 ``ATSCHK==1``，Translated 事务也被终止
   * - ``0b01``
     - Full ATS。Translation Request 服务所有已启用的翻译阶段，返回 PA。后续 Translated 事务绕过 SMMU
   * - ``0b10``
     - Split-stage ATS。Translation Request 仅执行 Stage 1，返回 IPA 给端点。后续 Translated 事务携带 IPA 经过 SMMU 执行 Stage 2 翻译。要求 ``Config==0b111``、``S2S==0``、``NS1ATS==0``、``ATSCHK==1``
   * - ``0b11``
     - Full ATS with DPT checks。同 ``0b01`` 但额外启用每粒度 DPT 检查（``SMMU_IDR3.DPT==1``，StreamWorld 非 EL1 时 ILLEGAL）

### ATSCHK 控制

``SMMU_(*_)CR0.ATSCHK`` 控制 SMMU 是否对 Translated 事务进行额外检查：

- ``ATSCHK == 0``：Translated 事务直接绕过 SMMU，不进行任何检查（性能最优，安全性较低）
- ``ATSCHK == 1``：Translated 事务仍经过 SMMU，根据 ``STE.EATS`` 检查配置，可执行 Split-stage Stage 2 或 DPT 检查

### ATS 权限检查

- Translation Request 携带读/写权限请求，SMMU 检查翻译表中的实际权限
- 响应可能返回实际授予的权限（可能是请求权限的子集，例如请求读写但只授予读）
- 仅执行权限（execute-only, 无读无写）在 ATS 中无法表示，通过 ATS 不可访问

### HTTU 与 ATS

- 如果 ATS Translation Request 对应的翻译有效且 HTTU 使能，SMMU **必须**在收到 ATS 请求时就更新翻译表中的 Access Flag 和 Dirty State
- 由于 Translated 事务绕过 SMMU，ATS 请求是 SMMU 更新标志的唯一机会

### ATC 失效（CMD_ATC_INV）

当翻译配置变更时，必须按以下顺序执行维护操作：

::

    1. 修改翻译表描述符（barrier 使其对 Shareability 域可见）
    2. CMD_TLBI_*    （SMMU TLB 失效）
    3. CMD_SYNC      （确保 TLB 失效完成）
    4. CMD_ATC_INV   （失效端点 ATC）
    5. CMD_SYNC      （确保 ATC 失效完成）

``CMD_ATC_INV`` 参数：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 参数
     - 说明
   * - StreamID
     - 目标设备
   * - SubstreamID + SSV
     - PASID 及有效标志
   * - Global (G)
     - 全局失效标志（仅 SSV==1 时有效）
   * - Address[63:12]
     - 失效起始地址
   * - Size[5:0]
     - 失效范围：span = 4096 * 2^Size 字节。Size=52 表示全部失效

### Split-stage ATS

Split-stage ATS（``EATS == 0b10``）允许在使用 ATS 和 PRI 的同时保持 Stage 2 隔离：

- ATS Translation Request 仅执行 Stage 1 翻译，返回 **IPA**（而非 PA）
- 端点缓存 IPA，后续 Translated 事务携带 IPA 到达 SMMU
- SMMU 对 Translated 事务执行 Stage 2 翻译（IPA -> PA），保持 Hypervisor 的 Stage 2 控制权
- 端点的 ATC 缓存的是 IPA，因此 ATC 失效仅需在 Stage 1 变更时进行

适用场景：Hypervisor 需要在虚拟化环境中同时使用 ATS/PRI 和 Stage 2 翻译隔离。

### ATS 配置变更安全流程

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 操作
     - 步骤
   * - 启用 ATS
     - 设置 EATS != 0 -> 失效 STE 缓存 + CMD_SYNC -> 端点启用 ATS
   * - 禁用 ATS
     - 端点禁用 ATS + 失效 ATC + CMD_SYNC -> 设置 EATS == 0 -> 失效 STE 缓存
   * - 切换 EATS 0b01 <-> 0b10
     - 必须先禁用 ATS（通过 EATS==0b00），再切换到目标值

### ATS 与安全状态

- ATS 和 PRI 不支持 Secure 流（Secure STE 的 EATS 为 RES0）
- ``CMD_ATC_INV`` 和 ``CMD_PRI_RESP`` 默认不能针对 Secure StreamID
- Secure Command Queue 上的 ATS/PRI 支持可选（``SMMU_S_IDR3.SAMS``）

PRI（Page Request Interface）
-----------------------------

### 概述

PRI 允许 PCIe 端点在 ATS Translation Request 发现页面不存在（not-present）时，主动向软件请求将页面调入内存。这使得设备和 OS/Hypervisor 可以使用未固定的、动态分页的内存进行 DMA。

### PRI 工作流程

```
Endpoint                          SMMU                         Software
   |                               |                               |
   |-- ATS Translation Request --->|                               |
   |                               |-- 翻译结果: not-present ------|
   |<-- ATS Completion (R=0,W=0)-|                               |
   |   (无权限, 表示页不存在)      |                               |
   |                               |                               |
   |-- PRI Page Request (PPR) ---->|-- 写入 PRI Queue ------------>|
   |   (请求页面调入)              |                               |
   |                               |                     软件处理:
   |                               |                     - 分配物理页
   |                               |                     - 更新翻译表
   |                               |                     - CMD_TLBI_*
   |                               |                     - CMD_ATC_INV
   |                               |                     - CMD_PRI_RESP(Success)
   |                               |<-- CMD_PRI_RESP -----------|
   |<-- PRG Response (Success) ----|                               |
   |                               |                               |
   |-- 重试 ATS Translation ------>|                               |
   |<-- ATS Completion (PA, R/W) --|                               |
   |                               |                               |
   |-- Translated DMA (PA) --------|                               |
```

### PRI Queue

PRI Queue 是内存中的环形缓冲区，接收来自端点的 Page Request：

- 由 ``SMMU_(*_)CR0.PRIQEN`` 启用
- 大小由 ``SMMU_IDR1.PRIQS`` 指示（每个 SMMU 的最大 PRI Queue 条目数）
- 端点通过 PCIe 配置空间的 PRI 机制获得 Page Request Credit（信用值），限制未完成请求的数量

### PRI 响应（CMD_PRI_RESP）

软件处理完页面调入后，通过 ``CMD_PRI_RESP`` 命令响应端点：

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Resp 值
     - 含义
   * - ``0b00`` ResponseFailure
     - 永久非分页错误（如设备 ATS/PRI 被禁用）
   * - ``0b01`` InvalidRequest
     - 请求组中一个或多个页面调入失败
   * - ``0b10`` Success
     - 页面请求组成功调入

### PRI 队列溢出处理

当 PRI Queue 满时：

- ``SMMU_PRIQ_PROD.OVFLG`` 标志置位
- 新的 PRI 消息被丢弃
- 对具有 ``Last==1`` 的 Page Request 自动生成 PRG Response（Success），使端点可以稍后重试
- ``Last==0`` 的 PPR 被静默忽略
- 软件应尽快消费 PRI Queue 并通过 ``SMMU_PRIQ_CONS.OVACKFLG`` 确认溢出

### 虚拟化中的 PRI

- Hypervisor 管理 Host PRI Queue
- Guest 设置自己的 PRI Queue，Hypervisor 将 Host PRI Queue 中的 PPR 代理转发到 Guest PRI Queue
- PRI Queue 信用值分配：Host 总共 L 条，为 Guest 分配 N 条（N <= L），Guest 为其设备分配最多 N 条信用
- ``STE.PPAR`` 控制 PASID 前缀在 PRI 响应中是否保留

ATS + PRI 使用模型总结
-------------------------

.. list-table::
   :header-rows: 1
   :widths: 30 35 35

   * - 场景
     - ATS
     - PRI
   * - 固定 DMA（无动态分页）
     - 可选（提升性能）
     - 不需要
   * - 动态分页 DMA（非 PCIe）
     - 不需要
     - 不需要（使用 Stall 模式代替）
   * - 动态分页 DMA（PCIe）
     - 必须
     - 必须（PCIe 不支持 Stall）
   * - 虚拟化 + ATS + Stage 2 隔离
     - Split-stage ATS (``EATS==0b10``)
     - 必须

**关键约束**：

- PCIe 设备**不能**使用 Stall 模式（事务可能无限挂起），必须使用 ATS + PRI 实现动态分页
- 非 PCIe 设备可以使用 SMMU 内部 Stall 模式替代 ATS + PRI
- PRI 要求 HTTU 支持以确保正确性（软件更新的 Access Flag/Dirty State 与硬件更新保持一致）

参考：IHI0070G Chapter 3 Operation（3.9 节）、Chapter 4 Commands（4.5 节）和 Chapter 8 PRI queue（8.1 节）。
