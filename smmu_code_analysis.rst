ARM SMMUv3 驱动深度分析
============================

**文件**: ``src/linux-6.1-rc6/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c`` (3888 行)

1. 驱动功能概述
----------------

ARM 架构 SMMUv3 (System Memory Management Unit version 3) 的 IOMMU 驱动，
为 DMA 设备提供地址转换、TLB 管理、ATS/PRI/SVA 支持，通过 ``iommu_ops``
回调注册到 Linux IOMMU 子系统。

- **所属子系统**: ``drivers/iommu/`` -- Linux IOMMU 子系统
- **驱动类型**: Platform driver，匹配 OF ``compatible = "arm,smmu-v3"`` 和 ACPI IORT 节点
- **入口**: ``module_driver()`` -> ``platform_driver_register(&arm_smmu_driver)`` (行 3882)
- **模块参数**: ``disable_bypass`` (bool, 默认 true), ``disable_msipolling`` (bool, 默认 false)

2. 核心数据结构及其关系
---------------------------

.. code-block:: text

   arm_smmu_device (核心设备实例, .h:618-677)
   +-- *dev (struct device)
   +-- *base / *page1 (MMIO 寄存器空间)
   +-- features (u32, 19个特性位: S1/S2/PRI/ATS/MSI/SVA/STALL...)
   +-- options (u32, 3个驱动选项位)
   |
   +-- cmdq : arm_smmu_cmdq (.h:542)
   |   +-- q : arm_smmu_queue (.h:521)
   |   |   +-- llq : arm_smmu_ll_queue (.h:505)
   |   |       + *base (DMA一致性内存)
   |   + *valid_map (原子位图, 命令有效性)
   |
   +-- evtq : arm_smmu_evtq (.h:554)
   |   +-- q : arm_smmu_queue
   |   + *iopf (IO Page Fault 队列)
   |
   +-- priq : arm_smmu_priq (.h:560)
   |   +-- q : arm_smmu_queue
   |
   +-- strtab_cfg : arm_smmu_strtab_cfg (.h:607)
   |   +-- *strtab (DMA一致性内存)
   |   + *l1_desc[] -> *l2ptr (DMA, 两级流表)
   |
   +-- iommu : struct iommu_device (IOMMU核心句柄)
   |
   +-- streams : rb_root (红黑树)
       +-- arm_smmu_stream (.h:679)
           +-- *master -> arm_smmu_master (.h:686)
               +-- *smmu -> (回指 arm_smmu_device)
               +-- *dev (Linux 设备)
               +-- *domain -> arm_smmu_domain (.h:709)
               |   +-- *smmu -> (回指)
               |   +-- *pgtbl_ops (struct io_pgtable_ops, 页表操作)
               |   +-- s1_cfg : arm_smmu_s1_cfg (.h:594)
               |   |   +-- cdcfg : arm_smmu_ctx_desc_cfg (.h:587) -> *cdtab (DMA)
               |   |   +-- cd : arm_smmu_ctx_desc (.h:572) -> *mm (进程地址空间)
               |   +-- s2_cfg : arm_smmu_s2_cfg (.h:601) (vmid/vttbr/vtcr)
               |   +-- devices : list_head -> master->domain_head
               |   +-- domain : struct iommu_domain (嵌入)
               +-- *streams[] (流数组)

关键关系:

- ``arm_smmu_device`` 1:N ``arm_smmu_master`` (通过红黑树 streams)
- ``arm_smmu_master`` N:1 ``arm_smmu_domain`` (通过 domain 指针 + devices 链表)
- ``arm_smmu_domain`` 1:1 ``arm_smmu_device`` (通过 smmu 指针)
- 三个队列 (cmdq/evtq/priq) 继承自 ``arm_smmu_queue`` -> ``arm_smmu_ll_queue``

3. 初始化/注册流程 (从入口到硬件可用)
------------------------------------------

.. code-block:: text

   module_init
     +-> platform_driver_register(&arm_smmu_driver)         [行 3882]
          | [内核 platform bus 匹配设备后自动调用 .probe]
          v
        arm_smmu_device_probe()                            [行 3736]
          +-(1) arm_smmu_device_dt_probe()  / _acpi_probe()  [行 3668/3641]
          |       +-- parse_driver_options() + 一致性检测
          +-(2) arm_smmu_ioremap() x2 (base + page1)      [行 3698]
          +-(3) platform_get_irq_byname_optional() x4       [行 3788]
          |       (combined 或 eventq/priq/gerror)
          +-(4) arm_smmu_device_hw_probe()                  [行 3413]
          |       +-- 读 IDR0/IDR1/IDR3/IDR5 寄存器
          |       +-- dma_set_mask_and_coherent()            [行 3612]
          |       +-- arm_smmu_sva_supported()               [SVA检测]
          +-(5) arm_smmu_init_structures()                  [行 3076]
          |       +-- mutex_init + RB_ROOT
          |       +-- arm_smmu_init_queues()                 [行 2930]
          |       |   +-- arm_smmu_init_one_queue(cmdq) + cmdq_init
          |       |   +-- arm_smmu_init_one_queue(evtq) + iopf_queue_alloc
          |       |   +-- arm_smmu_init_one_queue(priq)
          |       +-- arm_smmu_init_strtab()                 [行 3053]
          |           +-- [两级] arm_smmu_init_strtab_2lvl() -> dmam_alloc_coherent
          |           +-- [线性] arm_smmu_init_strtab_linear() -> dmam_alloc_coherent
          +-(6) arm_smmu_rmr_install_bypass_ste()           [行 3706]
          |       +-- ACPI RMR 区域 bypass STE 安装
          +-(7) arm_smmu_device_reset()                     [行 3278]
          |       +-- arm_smmu_device_disable() (清 CR0)
          |       +-- 写 CR1/CR2 (缓存属性/PTM/RECINVSID/E2H)
          |       +-- 写 STRTAB_BASE / CMDQ_BASE / EVTQ_BASE / PRIQ_BASE
          |       +-- CMD_SYNC 使能 CMDQ -> CFGI_ALL -> TLBI 全刷
          |       +-- 使能 EVTQ / PRIQ / ATSCHK
          |       +-- arm_smmu_setup_irqs()                  [行 3226]
          |       |   +-- [combined] devm_request_threaded_irq(硬+线程)
          |       |   +-- [unique] arm_smmu_setup_unique_irqs()
          |       |       +-- arm_smmu_setup_msis() -> platform_msi_domain_alloc_irqs
          |       |       +-- devm_request_threaded_irq(evtq, 纯线程)
          |       |       +-- devm_request_irq(gerror, 硬中断)
          |       |       +-- devm_request_threaded_irq(priq, 纯线程)
          |       +-- 写 CR0_SMMUEN 使能 SMMU
          +-(8) iommu_device_sysfs_add()                    [行 3828]
          +-(9) iommu_device_register(&arm_smmu_ops)       [行 3833]
                  -> 硬件完全可用，可接受设备 attach

注意: 该驱动无显式时钟操作 (``clk_*``)，SMMUv3 时钟由固件/电源域管理。

4. 操作函数集（回调表）
-------------------------

4.1 struct iommu_ops arm_smmu_ops (行 2841-2868)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. list-table::
   :header-rows: 1
   :widths: 35 50 10

   * - 函数名
     - 职责
     - 行号
   * - ``arm_smmu_capable``
     - 查询设备能力(CACHE_COHERENCY/NOEXEC)
     - :1993
   * - ``arm_smmu_domain_alloc``
     - 分配 iommu_domain (DMA/UNMANAGED/IDENTITY)
     - :2008
   * - ``arm_smmu_probe_device``
     - 探测设备，创建 master，插入红黑树
     - :2644
   * - ``arm_smmu_release_device``
     - 释放 master，从红黑树移除
     - :2704
   * - ``arm_smmu_device_group``
     - 获取 IOMMU group (PCI->pci_device_group, else->generic)
     - :2716
   * - ``arm_smmu_of_xlate``
     - 解析 DT iommu-cells，添加 Stream ID
     - :2748
   * - ``arm_smmu_get_resv_regions``
     - 获取保留区域 (MSI IOVA + DMA保留区)
     - :2753
   * - ``arm_smmu_dev_enable_feature``
     - 使能设备特性 (IOPF/SVA)
     - :2769
   * - ``arm_smmu_dev_disable_feature``
     - 禁用设备特性 (IOPF/SVA)
     - :2796
   * - ``arm_smmu_page_response``
     - 恢复 stalled 事务 (RESUME命令)
     - :906
   * - ``arm_smmu_def_domain_type``
     - 默认 domain 类型 (华为PTT->IDENTITY)
     - :2829
   * - ``arm_smmu_sva_bind``
     - SVA 绑定设备与进程地址空间 (sva.c:370)
     - sva.c
   * - ``arm_smmu_sva_unbind``
     - SVA 解绑 (sva.c:386)
     - sva.c
   * - ``arm_smmu_sva_get_pasid``
     - 获取 PASID (sva.c:399)
     - sva.c

4.2 struct iommu_domain_ops (default_domain_ops, 行 2858-2867)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. list-table::
   :header-rows: 1
   :widths: 35 50 10

   * - 函数名
     - 职责
     - 行号
   * - ``arm_smmu_attach_dev``
     - 设备挂载到 domain，安装 STE，使能 ATS
     - :2396
   * - ``arm_smmu_map_pages``
     - IOVA->PA 映射 (委托 io_pgtable_ops)
     - :2472
   * - ``arm_smmu_unmap_pages``
     - IOVA->PA 解映射 (委托 io_pgtable_ops)
     - :2484
   * - ``arm_smmu_flush_iotlb_all``
     - 全量 TLB 刷 (TLBI all contexts)
     - :2497
   * - ``arm_smmu_iotlb_sync``
     - 增量 TLB 刷 (TLBI range by gather)
     - :2505
   * - ``arm_smmu_iova_to_phys``
     - IOVA->物理地址查询
     - :2519
   * - ``arm_smmu_enable_nesting``
     - 使能嵌套转换 (S1+S2)
     - :2733
   * - ``arm_smmu_domain_free``
     - 释放 domain (页表/CD表/ASID/VMID)
     - :2053

调用者: IOMMU 核心层 (``drivers/iommu/iommu.c``) 和 DMA-IOMMU 子系统
(``drivers/iommu/dma-iommu.c``) 通过 ``iommu_ops`` / ``iommu_domain_ops``
函数指针间接调用。

5. 中断处理流程
-----------------

5.1 中断注册汇总
^^^^^^^^^^^^^^^^^

.. list-table::
   :header-rows: 1
   :widths: 15 35 30 30 15 10

   * - 中断类型
     - 注册调用
     - 硬中断handler
     - 线程handler
     - IRQF
     - 行号
   * - Combined
     - ``devm_request_threaded_irq``
     - ``combined_irq_handler``
     - ``combined_irq_thread``
     - ``ONESHOT``
     - :3245
   * - EVTQ
     - ``devm_request_threaded_irq``
     - ``NULL``
     - ``evtq_thread``
     - ``ONESHOT``
     - :3189
   * - GERROR
     - ``devm_request_irq``
     - ``gerror_handler``
     - 无
     - ``0``
     - :3201
   * - PRIQ
     - ``devm_request_threaded_irq``
     - ``NULL``
     - ``priq_thread``
     - ``ONESHOT``
     - :3212

优先级: combined > 独立中断线 > MSI 自动分配

5.2 处理链图
^^^^^^^^^^^^^

.. code-block:: text

   硬件中断触发
       |
       +-- [Combined 模式]
       |   +-- 硬中断: arm_smmu_combined_irq_handler()       [行 1704]
       |   |   +-- arm_smmu_gerror_handler()                 [行 1647]
       |   +-- 线程: arm_smmu_combined_irq_thread()           [行 1693]
       |       +-- arm_smmu_evtq_thread()                     [行 1545]
       |       +-- arm_smmu_priq_thread()                     [行 1623]
       |
       +-- [Unique 模式]
           +-- EVTQ: 纯线程 arm_smmu_evtq_thread()            [行 1545]
           |   +-- 循环: queue_remove_raw() -> arm_smmu_handle_evt()  [行 1451]
           |       +-- [STALL=1] -> iommu_report_device_fault(PAGE_REQ)
           |       +-- [STALL=0] -> iommu_report_device_fault(DMA_UNRECOV)
           |       +-- [无人处理] -> arm_smmu_page_response(FAILURE/ABORT)
           |
           +-- PRIQ: 纯线程 arm_smmu_priq_thread()            [行 1623]
           |   +-- 循环: queue_remove_raw() -> arm_smmu_handle_ppr()    [行 1585]
           |       +-- [last=1] -> CMDQ_OP_PRI_RESP(DENY) 回复
           |
           +-- GERROR: 硬中断 arm_smmu_gerror_handler()       [行 1647]
               +-- SFM_ERR -> arm_smmu_device_disable() (致命)
               +-- MSI_ABORT -> dev_warn
               +-- PRIQ/EVTQ_ABORT -> dev_err (事件丢失)
               +-- CMDQ_ERR -> __arm_smmu_cmdq_skip_err()      [行 360]
                   +-- CERROR_NONE/ABT -> 直接返回
                   +-- CERROR_ATC_INV -> 返回 (重发SYNC恢复)
                   +-- CERROR_ILL -> 原地替换为CMD_SYNC跳过

5.3 各中断触发条件
^^^^^^^^^^^^^^^^^^^

.. list-table::
   :header-rows: 1
   :widths: 15 50 50

   * - 中断
     - 触发条件
     - 处理方式
   * - EVTQ
     - SMMU 产生 fault 事件 (Translation/Address Size/Access/Permission Fault)
     - 线程遍历队列，上报 IOMMU fault 框架；STALL 事件支持恢复
   * - PRIQ
     - 设备发起 Page Request (ATS PRI)
     - 线程遍历队列，当前实现仅打印日志并回复 DENY
   * - GERROR
     - 全局错误 (SFM、队列/MSI写入中止、CMDQ非法命令)
     - 硬中断直接处理，SFM致命时禁用设备
   * - CMDQ
     - 无独立中断；完成通过 MSI polling 或寄存器轮询检测；错误通过 GERROR 的 CMDQ_ERR 位处理
