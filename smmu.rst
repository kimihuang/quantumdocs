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
