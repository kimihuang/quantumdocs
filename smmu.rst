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
