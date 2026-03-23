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
