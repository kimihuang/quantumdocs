ARM SMMUv3 驱动深度分析
============================

**文件**: ``src/linux-6.1-rc6/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c`` (3888 行)

**头文件**: ``src/linux-6.1-rc6/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.h`` (808 行)

**SVA 文件**: ``src/linux-6.1-rc6/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3-sva.c``

1. 驱动功能概述
----------------

ARM 架构 SMMUv3 (System Memory Management Unit version 3) 的 IOMMU 驱动，
为 DMA 设备提供地址转换、TLB 管理、ATS/PRI/SVA 支持，通过 ``iommu_ops``
回调注册到 Linux IOMMU 子系统。

- **所属子系统**: ``drivers/iommu/`` -- Linux IOMMU 子系统
- **驱动类型**: Platform driver，匹配 OF ``compatible = "arm,smmu-v3"`` 和 ACPI IORT 节点
- **入口**: ``module_driver()`` -> ``platform_driver_register(&arm_smmu_driver)`` (行 3882)
- **模块参数**:

  - ``disable_bypass`` (bool, 默认 true): 禁用 bypass 流，未挂载到 IOMMU domain
    的设备事务将被中止
  - ``disable_msipolling`` (bool, 默认 false): 禁用基于 MSI 的 CMD_SYNC 完成轮询

2. 核心数据结构及其关系
---------------------------

2.1 结构体关系图
^^^^^^^^^^^^^^^^^

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

2.2 核心结构体字段详表
^^^^^^^^^^^^^^^^^^^^^^^^^^

**struct arm_smmu_device** (.h:618-677) -- SMMUv3 设备实例

.. list-table::
   :header-rows: 1
   :widths: 25 15 55

   * - 字段
     - 类型
     - 说明
   * - ``dev``
     - ``struct device *``
     - 关联的 Linux 设备
   * - ``base``
     - ``void __iomem *``
     - MMIO 页0 基地址 (64KB)
   * - ``page1``
     - ``void __iomem *``
     - MMIO 页1 基地址 (64KB, EVTQ/PRIQ 寄存器)
   * - ``features``
     - ``u32``
     - 硬件特性位图 (19位, 见 2.3 特性标志表)
   * - ``options``
     - ``u32``
     - 驱动选项位图 (3位, 见 2.3 选项标志表)
   * - ``cmdq``
     - ``struct arm_smmu_cmdq``
     - 命令队列
   * - ``evtq``
     - ``struct arm_smmu_evtq``
     - 事件队列
   * - ``priq``
     - ``struct arm_smmu_priq``
     - PRI 队列
   * - ``gerr_irq``
     - ``int``
     - 全局错误中断号
   * - ``combined_irq``
     - ``int``
     - 合并中断号 (Cavium ThunderX2 等)
   * - ``ias``
     - ``unsigned long``
     - 输入地址宽度 (IPA bits)
   * - ``oas``
     - ``unsigned long``
     - 输出地址宽度 (PA bits)
   * - ``pgsize_bitmap``
     - ``unsigned long``
     - 支持的页大小位图
   * - ``asid_bits``
     - ``unsigned int``
     - ASID 位数 (8 或 16)
   * - ``vmid_bits``
     - ``unsigned int``
     - VMID 位数 (8 或 16)
   * - ``vmid_map``
     - ``DECLARE_BITMAP``
     - VMID 分配位图 (最大 65536)
   * - ``ssid_bits``
     - ``unsigned int``
     - Substream ID 位数
   * - ``sid_bits``
     - ``unsigned int``
     - Stream ID 位数
   * - ``strtab_cfg``
     - ``struct arm_smmu_strtab_cfg``
     - 流表配置
   * - ``iommu``
     - ``struct iommu_device``
     - IOMMU 核心设备句柄 (嵌入)
   * - ``streams``
     - ``struct rb_root``
     - 流的红黑树根
   * - ``streams_mutex``
     - ``struct mutex``
     - 流操作互斥锁

**struct arm_smmu_master** (.h:686-699) -- Master 设备

.. list-table::
   :header-rows: 1
   :widths: 25 15 55

   * - 字段
     - 类型
     - 说明
   * - ``smmu``
     - ``struct arm_smmu_device *``
     - 所属 SMMU 设备
   * - ``dev``
     - ``struct device *``
     - Linux 设备指针
   * - ``domain``
     - ``struct arm_smmu_domain *``
     - 当前 attached 的 domain
   * - ``domain_head``
     - ``struct list_head``
     - domain->devices 链表节点
   * - ``streams``
     - ``struct arm_smmu_stream *``
     - 流数组
   * - ``num_streams``
     - ``unsigned int``
     - 流数量
   * - ``ats_enabled``
     - ``bool``
     - ATS 是否启用
   * - ``stall_enabled``
     - ``bool``
     - Stall 是否启用
   * - ``sva_enabled``
     - ``bool``
     - SVA 是否启用
   * - ``iopf_enabled``
     - ``bool``
     - IO Page Fault 是否启用
   * - ``bonds``
     - ``struct list_head``
     - SVA 绑定链表
   * - ``ssid_bits``
     - ``unsigned int``
     - Substream ID 位数

**struct arm_smmu_domain** (.h:709-729) -- IOMMU Domain

.. list-table::
   :header-rows: 1
   :widths: 25 15 55

   * - 字段
     - 类型
     - 说明
   * - ``smmu``
     - ``struct arm_smmu_device *``
     - 所属 SMMU 设备
   * - ``init_mutex``
     - ``struct mutex``
     - 保护 smmu 指针
   * - ``pgtbl_ops``
     - ``struct io_pgtable_ops *``
     - 页表操作回调
   * - ``stall_enabled``
     - ``bool``
     - stall 是否启用
   * - ``nr_ats_masters``
     - ``atomic_t``
     - ATS master 计数
   * - ``stage``
     - ``enum arm_smmu_domain_stage``
     - 转换阶段 (S1/S2/NESTED/BYPASS)
   * - ``s1_cfg`` / ``s2_cfg``
     - ``union``
     - Stage 1 或 Stage 2 配置 (union)
   * - ``domain``
     - ``struct iommu_domain``
     - IOMMU core domain (嵌入)
   * - ``devices``
     - ``struct list_head``
     - 挂载的 master 列表
   * - ``devices_lock``
     - ``spinlock_t``
     - devices 列表锁
   * - ``mmu_notifiers``
     - ``struct list_head``
     - MMU 通知器列表

**队列继承层次**:

.. list-table::
   :header-rows: 1
   :widths: 30 65

   * - 结构体
     - 关键字段
   * - ``arm_smmu_ll_queue`` (.h:505)
     - ``val``(u64), ``prod``(u32), ``cons``(u32), ``atomic.prod/cons``, ``max_n_shift``
   * - ``arm_smmu_queue`` (.h:521)
     - 继承 ``llq``, ``irq``, ``*base``, ``base_dma``, ``q_base``, ``ent_dwords``, ``*prod_reg``, ``*cons_reg``
   * - ``arm_smmu_cmdq`` (.h:542)
     - 继承 ``q``, ``*valid_map``, ``owner_prod``, ``lock``
   * - ``arm_smmu_evtq`` (.h:554)
     - 继承 ``q``, ``*iopf``, ``max_stalls``
   * - ``arm_smmu_priq`` (.h:560)
     - 继承 ``q``

2.3 枚举与标志位定义
^^^^^^^^^^^^^^^^^^^^^^

**enum arm_smmu_domain_stage** (.h:702-707):

.. list-table::
   :header-rows: 1
   :widths: 35 55

   * - 枚举值
     - 含义
   * - ``ARM_SMMU_DOMAIN_S1 = 0``
     - Stage 1 转换 (VA -> PA 或 VA -> IPA)
   * - ``ARM_SMMU_DOMAIN_S2``
     - Stage 2 转换 (IPA -> PA)
   * - ``ARM_SMMU_DOMAIN_NESTED``
     - 嵌套转换 (S1 + S2)
   * - ``ARM_SMMU_DOMAIN_BYPASS``
     - 旁路 (无地址转换)

**enum pri_resp** (.h:423-427):

.. list-table::
   :header-rows: 1
   :widths: 35 55

   * - 枚举值
     - 含义
   * - ``PRI_RESP_DENY = 0``
     - 拒绝 Page Request
   * - ``PRI_RESP_FAIL = 1``
     - Page Request 失败
   * - ``PRI_RESP_SUCC = 2``
     - Page Request 成功

**enum arm_smmu_msi_index** (.c:44-49):

.. list-table::
   :header-rows: 1
   :widths: 35 55

   * - 枚举值
     - 含义
   * - ``EVTQ_MSI_INDEX``
     - 事件队列 MSI 索引 (0)
   * - ``GERROR_MSI_INDEX``
     - 全局错误 MSI 索引 (1)
   * - ``PRIQ_MSI_INDEX``
     - PRI 队列 MSI 索引 (2)

**特性标志位** (``arm_smmu_device.features``, .h:623-641):

.. list-table::
   :header-rows: 1
   :widths: 40 8 45

   * - 宏名
     - 位
     - 含义
   * - ``ARM_SMMU_FEAT_2_LVL_STRTAB``
     - 0
     - 两级流表
   * - ``ARM_SMMU_FEAT_2_LVL_CDTAB``
     - 1
     - 两级上下文描述符表
   * - ``ARM_SMMU_FEAT_TT_LE``
     - 2
     - 小端转换表
   * - ``ARM_SMMU_FEAT_TT_BE``
     - 3
     - 大端转换表
   * - ``ARM_SMMU_FEAT_PRI``
     - 4
     - Page Request Interface 支持
   * - ``ARM_SMMU_FEAT_ATS``
     - 5
     - Address Translation Service 支持
   * - ``ARM_SMMU_FEAT_SEV``
     - 6
     - SEV 同步事件支持
   * - ``ARM_SMMU_FEAT_MSI``
     - 7
     - MSI 支持 (CMDQ 完成通知)
   * - ``ARM_SMMU_FEAT_COHERENCY``
     - 8
     - 硬件 DMA 一致性
   * - ``ARM_SMMU_FEAT_TRANS_S1``
     - 9
     - Stage 1 地址转换支持
   * - ``ARM_SMMU_FEAT_TRANS_S2``
     - 10
     - Stage 2 地址转换支持
   * - ``ARM_SMMU_FEAT_STALLS``
     - 11
     - Stall 模型支持
   * - ``ARM_SMMU_FEAT_HYP``
     - 12
     - Hypervisor 模式支持
   * - ``ARM_SMMU_FEAT_STALL_FORCE``
     - 13
     - 强制 Stall
   * - ``ARM_SMMU_FEAT_VAX``
     - 14
     - 52-bit VA 支持
   * - ``ARM_SMMU_FEAT_RANGE_INV``
     - 15
     - Range Invalidation 支持
   * - ``ARM_SMMU_FEAT_BTM``
     - 16
     - Broadcast TLB Maintenance
   * - ``ARM_SMMU_FEAT_SVA``
     - 17
     - Shared Virtual Addressing
   * - ``ARM_SMMU_FEAT_E2H``
     - 18
     - EL2 Host 扩展

**驱动选项位** (``arm_smmu_device.options``, .h:644-646):

.. list-table::
   :header-rows: 1
   :widths: 40 8 45

   * - 宏名
     - 位
     - 含义
   * - ``ARM_SMMU_OPT_SKIP_PREFETCH``
     - 0
     - 跳过预取命令 (华为 HiSilicon)
   * - ``ARM_SMMU_OPT_PAGE0_REGS_ONLY``
     - 1
     - 仅使用页0寄存器空间 (Cavium CN9900)
   * - ``ARM_SMMU_OPT_MSIPOLL``
     - 2
     - 启用 MSI polling (CMD_SYNC 完成检测)

2.4 关键宏定义表
^^^^^^^^^^^^^^^^^^

**MMIO 寄存器偏移** (.h):

.. list-table::
   :header-rows: 1
   :widths: 40 12 45

   * - 宏名
     - 偏移
     - 说明
   * - ``ARM_SMMU_IDR0``
     - 0x0
     - ID 寄存器 0 (特性描述)
   * - ``ARM_SMMU_IDR1``
     - 0x4
     - ID 寄存器 1 (队列/ID 尺寸)
   * - ``ARM_SMMU_IDR3``
     - 0xc
     - ID 寄存器 3 (RIL)
   * - ``ARM_SMMU_IDR5``
     - 0x14
     - ID 寄存器 5 (页大小/OAS/VAX)
   * - ``ARM_SMMU_CR0``
     - 0x20
     - 控制 0 (SMMUEN/CMDQEN/EVTQEN/PRIQEN)
   * - ``ARM_SMMU_CR0ACK``
     - 0x24
     - CR0 确认寄存器
   * - ``ARM_SMMU_CR1``
     - 0x28
     - 控制 1 (缓存属性)
   * - ``ARM_SMMU_CR2``
     - 0x2c
     - 控制 2 (PTM/RECINVSID/E2H)
   * - ``ARM_SMMU_GBPA``
     - 0x44
     - 全局旁路属性
   * - ``ARM_SMMU_IRQ_CTRL``
     - 0x50
     - 中断控制
   * - ``ARM_SMMU_IRQ_CTRLACK``
     - 0x54
     - 中断控制确认
   * - ``ARM_SMMU_GERROR``
     - 0x60
     - 全局错误状态
   * - ``ARM_SMMU_GERRORN``
     - 0x64
     - 全局错误确认
   * - ``ARM_SMMU_STRTAB_BASE``
     - 0x80
     - 流表基地址
   * - ``ARM_SMMU_STRTAB_BASE_CFG``
     - 0x88
     - 流表基地址配置
   * - ``ARM_SMMU_CMDQ_BASE``
     - 0x90
     - 命令队列基地址
   * - ``ARM_SMMU_CMDQ_PROD/CONS``
     - 0x98/0x9c
     - 命令队列生产者/消费者
   * - ``ARM_SMMU_EVTQ_BASE``
     - 0xa0
     - 事件队列基地址 (页1)
   * - ``ARM_SMMU_EVTQ_PROD/CONS``
     - 0xa8/0xac
     - 事件队列生产者/消费者 (页1)
   * - ``ARM_SMMU_PRIQ_BASE``
     - 0xc0
     - PRI 队列基地址 (页1)
   * - ``ARM_SMMU_PRIQ_PROD/CONS``
     - 0xc8/0xcc
     - PRI 队列生产者/消费者 (页1)
   * - ``ARM_SMMU_REG_SZ``
     - 0xe00
     - 单页寄存器空间大小

**CR0 控制位** (.h:72-77):

.. list-table::
   :header-rows: 1
   :widths: 30 8 50

   * - 宏名
     - 位
     - 说明
   * - ``CR0_SMMUEN``
     - 0
     - SMMU 全局使能
   * - ``CR0_PRIQEN``
     - 1
     - PRI 队列使能
   * - ``CR0_EVTQEN``
     - 2
     - 事件队列使能
   * - ``CR0_CMDQEN``
     - 3
     - 命令队列使能
   * - ``CR0_ATSCHK``
     - 4
     - ATS 检查使能

**GERROR 错误位** (.h:109-118):

.. list-table::
   :header-rows: 1
   :widths: 40 8 45

   * - 宏名
     - 位
     - 说明
   * - ``GERROR_CMDQ_ERR``
     - 0
     - 命令队列错误
   * - ``GERROR_EVTQ_ABT_ERR``
     - 2
     - 事件队列写入中止
   * - ``GERROR_PRIQ_ABT_ERR``
     - 3
     - PRI 队列写入中止
   * - ``GERROR_MSI_CMDQ_ABT_ERR``
     - 4
     - CMDQ MSI 写入中止
   * - ``GERROR_MSI_EVTQ_ABT_ERR``
     - 5
     - EVTQ MSI 写入中止
   * - ``GERROR_MSI_PRIQ_ABT_ERR``
     - 6
     - PRIQ MSI 写入中止
   * - ``GERROR_MSI_GERROR_ABT_ERR``
     - 7
     - GERROR MSI 写入中止
   * - ``GERROR_SFM_ERR``
     - 8
     - Service Failure Mode (致命)

**STE 配置字段** (.h:203-257):

.. list-table::
   :header-rows: 1
   :widths: 40 15 40

   * - 宏名
     - 值
     - 说明
   * - ``STRTAB_STE_0_V``
     - 1UL << 0
     - Valid 位
   * - ``STRTAB_STE_0_CFG_ABORT``
     - 0
     - Abort 配置
   * - ``STRTAB_STE_0_CFG_BYPASS``
     - 4
     - Bypass 配置
   * - ``STRTAB_STE_0_CFG_S1_TRANS``
     - 5
     - Stage 1 转换
   * - ``STRTAB_STE_0_CFG_S2_TRANS``
     - 6
     - Stage 2 转换
   * - ``STRTAB_STE_0_S1FMT_LINEAR``
     - 0
     - 线性 CD 表
   * - ``STRTAB_STE_0_S1FMT_64K_L2``
     - 2
     - 64K 两级 CD 表
   * - ``STRTAB_STE_1_S1DSS_TERMINATE``
     - 0x0
     - 终止
   * - ``STRTAB_STE_1_S1DSS_BYPASS``
     - 0x1
     - Bypass
   * - ``STRTAB_STE_1_S1DSS_SSID0``
     - 0x2
     - SSID0 模式
   * - ``STRTAB_STE_1_EATS_ABT``
     - 0
     - EATS 中止
   * - ``STRTAB_STE_1_EATS_TRANS``
     - 1
     - EATS 转换
   * - ``STRTAB_STE_1_EATS_S1CHK``
     - 2
     - EATS S1 检查

**命令队列操作码** (.h:436-501):

.. list-table::
   :header-rows: 1
   :widths: 35 10 50

   * - 宏名
     - 值
     - 说明
   * - ``CMDQ_OP_PREFETCH_CFG``
     - 0x1
     - 预取配置
   * - ``CMDQ_OP_CFGI_STE``
     - 0x3
     - 配置单个 STE
   * - ``CMDQ_OP_CFGI_ALL``
     - 0x4
     - 配置所有 STE
   * - ``CMDQ_OP_CFGI_CD``
     - 0x5
     - 配置单个 CD
   * - ``CMDQ_OP_CFGI_CD_ALL``
     - 0x6
     - 配置所有 CD
   * - ``CMDQ_OP_TLBI_NH_ASID``
     - 0x11
     - TLB 无效 (Non-secure, Hyp, ASID)
   * - ``CMDQ_OP_TLBI_NH_VA``
     - 0x12
     - TLB 无效 (Non-secure, Hyp, VA)
   * - ``CMDQ_OP_TLBI_EL2_ALL``
     - 0x20
     - EL2 全部 TLB 无效
   * - ``CMDQ_OP_TLBI_EL2_ASID``
     - 0x21
     - EL2 TLB 无效 (ASID)
   * - ``CMDQ_OP_TLBI_EL2_VA``
     - 0x22
     - EL2 TLB 无效 (VA)
   * - ``CMDQ_OP_TLBI_S12_VMALL``
     - 0x28
     - S12 全部 VM TLB 无效
   * - ``CMDQ_OP_TLBI_S2_IPA``
     - 0x2a
     - S2 IPA TLB 无效
   * - ``CMDQ_OP_TLBI_NSNH_ALL``
     - 0x30
     - Non-secure Non-Hyp 全部无效
   * - ``CMDQ_OP_ATC_INV``
     - 0x40
     - ATC 无效
   * - ``CMDQ_OP_PRI_RESP``
     - 0x41
     - PRI 响应
   * - ``CMDQ_OP_RESUME``
     - 0x44
     - 恢复 stalled 事务
   * - ``CMDQ_OP_CMD_SYNC``
     - 0x46
     - 命令同步 (屏障)

**事件队列事件 ID** (.h:378-397):

.. list-table::
   :header-rows: 1
   :widths: 35 10 50

   * - 宏名
     - 值
     - 说明
   * - ``EVT_ID_TRANSLATION_FAULT``
     - 0x10
     - 转换错误 (页表遍历失败)
   * - ``EVT_ID_ADDR_SIZE_FAULT``
     - 0x11
     - 地址大小错误
   * - ``EVT_ID_ACCESS_FAULT``
     - 0x12
     - 访问错误
   * - ``EVT_ID_PERMISSION_FAULT``
     - 0x13
     - 权限错误

**CMDQ 错误索引** (.h:308-312):

.. list-table::
   :header-rows: 1
   :widths: 40 8 45

   * - 宏名
     - 值
     - 说明
   * - ``CMDQ_ERR_CERROR_NONE_IDX``
     - 0
     - 无错误
   * - ``CMDQ_ERR_CERROR_ILL_IDX``
     - 1
     - 非法命令
   * - ``CMDQ_ERR_CERROR_ABT_IDX``
     - 2
     - 命令获取中止
   * - ``CMDQ_ERR_CERROR_ATC_INV_IDX``
     - 3
     - ATC 无效超时

**其他关键常量** (.h):

.. list-table::
   :header-rows: 1
   :widths: 40 15 40

   * - 宏名
     - 值
     - 说明
   * - ``STRTAB_SPLIT``
     - 8
     - L1/L2 流表分裂值
   * - ``CTXDESC_SPLIT``
     - 10
     - L1/L2 CD 表分裂值
   * - ``CTXDESC_L2_ENTRIES``
     - 1024
     - L2 CD 表条目数
   * - ``STRTAB_STE_DWORDS``
     - 8
     - STE 大小 (8 DW = 64B)
   * - ``CTXDESC_CD_DWORDS``
     - 8
     - CD 大小 (8 DW = 64B)
   * - ``CMDQ_ENT_DWORDS``
     - 2
     - CMDQ 条目大小 (2 DW = 16B)
   * - ``EVTQ_ENT_DWORDS``
     - 4
     - EVTQ 条目大小 (4 DW = 32B)
   * - ``PRIQ_ENT_DWORDS``
     - 2
     - PRIQ 条目大小 (2 DW = 16B)
   * - ``ARM_SMMU_MAX_ASIDS``
     - 1 << 16
     - 最大 ASID 数
   * - ``ARM_SMMU_MAX_VMIDS``
     - 1 << 16
     - 最大 VMID 数
   * - ``MSI_IOVA_BASE``
     - 0x8000000
     - MSI IOVA 基地址
   * - ``MSI_IOVA_LENGTH``
     - 0x100000
     - MSI IOVA 长度
   * - ``ARM_SMMU_POLL_TIMEOUT_US``
     - 1000000
     - 轮询超时 (1秒)
   * - ``ARM_SMMU_POLL_SPIN_COUNT``
     - 10
     - 轮询自旋次数

3. 初始化/注册流程 (从入口到硬件可用)
------------------------------------------

3.1 入口与平台驱动注册
^^^^^^^^^^^^^^^^^^^^^^^^^

该驱动使用 ``module_driver`` 宏注册，等价于 ``module_init`` / ``module_exit``:

.. code-block:: c

   // .c:3872-3883
   static struct platform_driver arm_smmu_driver = {
       .driver = {
           .name                   = "arm-smmu-v3",
           .of_match_table         = arm_smmu_of_match,  // "arm,smmu-v3"
           .suppress_bind_attrs    = true,
       },
       .probe  = arm_smmu_device_probe,       // 行 3736
       .remove = arm_smmu_device_remove,      // 行 3843
       .shutdown = arm_smmu_device_shutdown,  // 行 3855
   };

   module_driver(arm_smmu_driver, platform_driver_register,
                 arm_smmu_driver_unregister);

该驱动没有独立的 ``acpi_driver``。ACPI 平台下通过 IORT (I/O Remapping Table)
将 ACPI IORT 节点转换成 ``platform_device``，统一走 ``platform_driver`` 的
``.probe`` 回调。

3.2 完整调用链
^^^^^^^^^^^^^^^^

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

3.3 关键初始化函数详解
^^^^^^^^^^^^^^^^^^^^^^^^^

**arm_smmu_device_probe()** (.c:3736-3841) -- probe 入口

.. code-block:: c

   static int arm_smmu_device_probe(struct platform_device *pdev)
   {
       struct arm_smmu_device *smmu;
       // 1. kzalloc 分配 smmu 设备结构
       smmu = devm_kzalloc(dev, sizeof(*smmu), GFP_KERNEL);
       // 2. DT/ACPI 固件探测
       if (dev->of_node)
           ret = arm_smmu_device_dt_probe(pdev, smmu);
       else
           ret = arm_smmu_device_acpi_probe(pdev, smmu);
       // 3. MMIO 映射 (page0 + page1)
       smmu->base = arm_smmu_ioremap(dev, ioaddr, ARM_SMMU_REG_SZ);
       smmu->page1 = arm_smmu_ioremap(dev, ioaddr + SZ_64K, ARM_SMMU_REG_SZ);
       // 4. 中断线获取
       irq = platform_get_irq_byname_optional(pdev, "combined");
       // 5. 硬件能力探测
       ret = arm_smmu_device_hw_probe(smmu);
       // 6. 数据结构初始化
       ret = arm_smmu_init_structures(smmu);
       // 7. RMR bypass STE 安装
       arm_smmu_rmr_install_bypass_ste(smmu);
       // 8. 设备复位
       ret = arm_smmu_device_reset(smmu, bypass);
       // 9. 注册到 IOMMU 子系统
       iommu_device_sysfs_add(&smmu->iommu, dev, NULL, "smmu3.%pa", &ioaddr);
       iommu_device_register(&smmu->iommu, &arm_smmu_ops, dev);
   }

**arm_smmu_device_hw_probe()** (.c:3413-3624) -- 硬件能力探测

依次读取 IDR0/IDR1/IDR3/IDR5 寄存器，解析硬件能力:

- **IDR0**: 流表级别、Stall 模型、转换表端序、CD2L、VMID16、PRI、ATS、SEV、MSI、
  ASID16、HYP、COHACC、TTF、S1P、S2P
- **IDR1**: 队列大小 (CMDQS/EVTQS/PRIQS)、SSID 位数、SID 位数
- **IDR3**: Range Invalidation (RIL)
- **IDR5**: Stall 最大数、页粒度 (4K/16K/64K)、输出地址宽度 (OAS)、VAX

关键操作: ``dma_set_mask_and_coherent()`` 设置 DMA 掩码为 ``oas`` 位宽。

**arm_smmu_init_structures()** (.c:3076-3088) -- 数据结构初始化

.. code-block:: c

   static int arm_smmu_init_structures(struct arm_smmu_device *smmu)
   {
       mutex_init(&smmu->streams_mutex);
       smmu->streams = RB_ROOT;
       ret = arm_smmu_init_queues(smmu);   // cmdq + evtq + priq
       return arm_smmu_init_strtab(smmu);  // 两级或线性流表
   }

**arm_smmu_init_queues()** (.c:2930-2966) -- 队列初始化

.. code-block:: c

   static int arm_smmu_init_queues(struct arm_smmu_device *smmu)
   {
       // cmdq: dmam_alloc_coherent + valid_map 位图
       ret = arm_smmu_init_one_queue(smmu, &smmu->cmdq.q, smmu->base, ...);
       ret = arm_smmu_cmdq_init(smmu);   // owner_prod + lock + valid_map

       // evtq: dmam_alloc_coherent + iopf_queue (SVA+STALL 时)
       ret = arm_smmu_init_one_queue(smmu, &smmu->evtq.q, smmu->page1, ...);
       smmu->evtq.iopf = iopf_queue_alloc(dev_name(smmu->dev));

       // priq: 仅 PRI 特性可用时分配
       ret = arm_smmu_init_one_queue(smmu, &smmu->priq.q, smmu->page1, ...);
   }

每个队列通过 ``dmam_alloc_coherent()`` 分配 DMA 一致性内存，自动对齐。

**arm_smmu_device_reset()** (.c:3278-3411) -- 设备复位与使能

.. code-block:: c

   static int arm_smmu_device_reset(struct arm_smmu_device *smmu, bool bypass)
   {
       // 1. 禁用 SMMU (清 CR0)
       arm_smmu_device_disable(smmu);
       // 2. 设置缓存属性 (CR1: WB+ISH)
       writel_relaxed(reg, smmu->base + ARM_SMMU_CR1);
       // 3. CR2: PTM + RECINVSID + E2H
       writel_relaxed(reg, smmu->base + ARM_SMMU_CR2);
       // 4. 配置流表基地址
       writeq_relaxed(smmu->strtab_cfg.strtab_base, smmu->base + ARM_SMMU_STRTAB_BASE);
       // 5. 使能命令队列 -> CFGI_ALL -> TLBI 全刷
       writeq_relaxed(smmu->cmdq.q.q_base, smmu->base + ARM_SMMU_CMDQ_BASE);
       arm_smmu_write_reg_sync(smmu, CR0_CMDQEN, ARM_SMMU_CR0, ARM_SMMU_CR0ACK);
       arm_smmu_cmdq_issue_cmd_with_sync(smmu, &cmd); // CFGI_ALL
       arm_smmu_cmdq_issue_cmd_with_sync(smmu, &cmd); // TLBI_NSNH_ALL
       // 6. 使能事件队列
       writeq_relaxed(smmu->evtq.q.q_base, smmu->base + ARM_SMMU_EVTQ_BASE);
       arm_smmu_write_reg_sync(smmu, enables|CR0_EVTQEN, ...);
       // 7. 使能 PRI 队列 (可选)
       // 8. 使能 ATS 检查 (可选)
       // 9. 申请中断
       ret = arm_smmu_setup_irqs(smmu);
       // 10. 使能 SMMUEN (或 bypass)
       arm_smmu_write_reg_sync(smmu, enables|CR0_SMMUEN, ...);
   }

3.4 初始化流程汇总表
^^^^^^^^^^^^^^^^^^^^^^^^

.. list-table::
   :header-rows: 1
   :widths: 5 10 25 55

   * - 序号
     - 阶段
     - 函数
     - 关键操作
   * - 1
     - 模块加载
     - ``module_driver``
     - ``platform_driver_register(&arm_smmu_driver)``
   * - 2
     - 设备匹配
     - 内核 platform bus
     - OF: ``"arm,smmu-v3"``; ACPI: IORT
   * - 3
     - probe 入口
     - ``arm_smmu_device_probe`` :3736
     - ``kzalloc`` 分配 smmu
   * - 4
     - 固件探测
     - ``_dt_probe`` :3668 / ``_acpi_probe`` :3641
     - DT 属性 / IORT 数据
   * - 5
     - MMIO 映射
     - ``arm_smmu_ioremap`` :3698
     - ``devm_ioremap_resource``, 64K+64K
   * - 6
     - 中断获取
     - ``platform_get_irq_byname_optional``
     - combined / eventq / priq / gerror
   * - 7
     - HW 探测
     - ``arm_smmu_device_hw_probe`` :3413
     - 读 IDR0/IDR1/IDR3/IDR5
   * - 8
     - DMA 掩码
     - ``dma_set_mask_and_coherent``
     - 在 hw_probe 中调用, 掩码 = oas 位宽
   * - 9
     - 结构初始化
     - ``arm_smmu_init_structures`` :3076
     - mutex + RB 树
   * - 10
     - 队列分配
     - ``arm_smmu_init_queues`` :2930
     - cmdq + evtq + priq (DMA 一致性内存)
   * - 11
     - 流表分配
     - ``arm_smmu_init_strtab`` :3053
     - 2 级或线性 (DMA 一致性内存)
   * - 12
     - RMR bypass
     - ``arm_smmu_rmr_install_bypass_ste`` :3706
     - ACPI RMR 区域 bypass STE
   * - 13
     - 设备复位
     - ``arm_smmu_device_reset`` :3278
     - CR0/CR1/CR2 + 队列使能 + TLB 刷
   * - 14
     - 中断申请
     - ``arm_smmu_setup_irqs`` :3226
     - MSI / wired + request_irq
   * - 15
     - SMMU 使能
     - 写 ``CR0_SMMUEN``
     - 硬件开始工作
   * - 16
     - IOMMU 注册
     - ``iommu_device_register`` :3833
     - 对外暴露 IOMMU 接口

4. 操作函数集（回调表）
-------------------------

4.1 struct iommu_ops arm_smmu_ops (行 2841-2868)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: c

   static struct iommu_ops arm_smmu_ops = {
       .capable            = arm_smmu_capable,
       .domain_alloc       = arm_smmu_domain_alloc,
       .probe_device       = arm_smmu_probe_device,
       .release_device     = arm_smmu_release_device,
       .device_group       = arm_smmu_device_group,
       .of_xlate           = arm_smmu_of_xlate,
       .get_resv_regions   = arm_smmu_get_resv_regions,
       .dev_enable_feat    = arm_smmu_dev_enable_feature,
       .dev_disable_feat   = arm_smmu_dev_disable_feature,
       .sva_bind           = arm_smmu_sva_bind,
       .sva_unbind         = arm_smmu_sva_unbind,
       .sva_get_pasid      = arm_smmu_sva_get_pasid,
       .page_response      = arm_smmu_page_response,
       .def_domain_type    = arm_smmu_def_domain_type,
       .pgsize_bitmap      = -1UL,
       .owner              = THIS_MODULE,
       .default_domain_ops = &(const struct iommu_domain_ops) {
           .attach_dev     = arm_smmu_attach_dev,
           .map_pages      = arm_smmu_map_pages,
           .unmap_pages    = arm_smmu_unmap_pages,
           .flush_iotlb_all= arm_smmu_flush_iotlb_all,
           .iotlb_sync     = arm_smmu_iotlb_sync,
           .iova_to_phys   = arm_smmu_iova_to_phys,
           .enable_nesting = arm_smmu_enable_nesting,
           .free           = arm_smmu_domain_free,
       }
   };

该驱动**没有** ``arm_smmu_impl`` 结构体或中间抽象层（与 SMMUv2 不同），
直接通过函数指针注册回调。SVA 功能通过 ``CONFIG_ARM_SMMU_V3_SVA`` 编译条件控制。

**iommu_ops 回调函数表**:

.. list-table::
   :header-rows: 1
   :widths: 35 50 10

   * - 函数名
     - 职责
     - 行号
   * - ``arm_smmu_capable``
     - 查询设备能力 (CACHE_COHERENCY/NOEXEC)
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
     - 获取保留区域 (MSI IOVA + DMA 保留区)
     - :2753
   * - ``arm_smmu_dev_enable_feature``
     - 使能设备特性 (IOPF/SVA)
     - :2769
   * - ``arm_smmu_dev_disable_feature``
     - 禁用设备特性 (IOPF/SVA)
     - :2796
   * - ``arm_smmu_page_response``
     - 恢复 stalled 事务 (RESUME 命令)
     - :906
   * - ``arm_smmu_def_domain_type``
     - 默认 domain 类型 (华为 PTT->IDENTITY)
     - :2829
   * - ``arm_smmu_sva_bind``
     - SVA 绑定设备与进程地址空间
     - sva.c:370
   * - ``arm_smmu_sva_unbind``
     - SVA 解绑
     - sva.c:386
   * - ``arm_smmu_sva_get_pasid``
     - 获取 PASID
     - sva.c:399

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
     - 释放 domain (页表/CD 表/ASID/VMID)
     - :2053

4.3 mmu_notifier_ops 回调 (SVA, sva.c)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. list-table::
   :header-rows: 1
   :widths: 35 50 10

   * - 函数名
     - 职责
     - 行号
   * - ``arm_smmu_mm_invalidate_range``
     - MMU 地址空间失效时同步 ATC/TLB
     - sva.c:189
   * - ``arm_smmu_mm_release``
     - 进程退出时清理 CD 和 TLB
     - sva.c:210
   * - ``arm_smmu_mmu_notifier_free``
     - 释放 mmu_notifier
     - sva.c:234

4.4 关键回调函数实现
^^^^^^^^^^^^^^^^^^^^^^^^

**arm_smmu_attach_dev()** (.c:2396-2470) -- 设备挂载

.. code-block:: c

   static int arm_smmu_attach_dev(struct iommu_domain *domain, struct device *dev)
   {
       // 1. 验证 SVA 未启用
       if (arm_smmu_master_sva_enabled(master)) return -EBUSY;
       // 2. 先 detach 旧 domain
       arm_smmu_detach_dev(master);
       // 3. 首次 attach 时 finalise domain (分配页表/CD/VMID/ASID)
       if (!smmu_domain->smmu) {
           smmu_domain->smmu = smmu;
           ret = arm_smmu_domain_finalise(domain, master);
       }
       // 4. 检查 smmu 一致性、SSID 匹配、stall 一致性
       // 5. 安装 STE，使能 ATS
       master->domain = smmu_domain;
       arm_smmu_install_ste_for_dev(master);
       list_add(&master->domain_head, &smmu_domain->devices);
       arm_smmu_enable_ats(master);
   }

**arm_smmu_page_response()** (.c:906-942) -- Stall 恢复

.. code-block:: c

   static int arm_smmu_page_response(struct device *dev,
                                     struct iommu_fault_event *unused,
                                     struct iommu_page_response *resp)
   {
       // stall 模式: 发送 RESUME 命令
       cmd.opcode       = CMDQ_OP_RESUME;
       cmd.resume.sid   = sid;
       cmd.resume.stag  = resp->grpid;
       switch (resp->code) {
       case IOMMU_PAGE_RESP_INVALID:
       case IOMMU_PAGE_RESP_FAILURE:
           cmd.resume.resp = CMDQ_RESUME_0_RESP_ABORT;  // 中止事务
           break;
       case IOMMU_PAGE_RESP_SUCCESS:
           cmd.resume.resp = CMDQ_RESUME_0_RESP_RETRY;  // 重试事务
           break;
       }
       arm_smmu_cmdq_issue_cmd(master->smmu, &cmd);
   }

**arm_smmu_map_pages()** (.c:2472-2482) -- 页映射

.. code-block:: c

   static int arm_smmu_map_pages(struct iommu_domain *domain, unsigned long iova,
                                 phys_addr_t paddr, size_t pgsize, size_t pgcount,
                                 int prot, gfp_t gfp, size_t *mapped)
   {
       // 委托给 io_pgtable_ops (arm64_lpae_ops 或 arm_v7s_ops)
       struct io_pgtable_ops *ops = to_smmu_domain(domain)->pgtbl_ops;
       return ops->map_pages(ops, iova, paddr, pgsize, pgcount, prot, gfp, mapped);
   }

**DMA 映射说明**: 该驱动**不直接实现** ``dma_map_ops``，通过
``#include "../../dma-iommu.h"`` 间接使用内核通用 IOMMU DMA 框架。
实际的 ``map`` / ``unmap`` / ``map_sg`` 等操作由 ``drivers/iommu/dma-iommu.c``
自动使用 ``default_domain_ops`` 回调执行。

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

5.2 中断源获取
^^^^^^^^^^^^^^^^

.. code-block:: c

   // .c:3788-3805
   irq = platform_get_irq_byname_optional(pdev, "combined");
   if (irq > 0)
       smmu->combined_irq = irq;     // Cavium ThunderX2: 合并中断
   else {
       // 分别获取独立中断线
       irq = platform_get_irq_byname_optional(pdev, "eventq");
       irq = platform_get_irq_byname_optional(pdev, "priq");
       irq = platform_get_irq_byname_optional(pdev, "gerror");
   }

5.3 处理链图
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

5.4 各中断触发条件
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
     - 全局错误 (SFM、队列/MSI 写入中止、CMDQ 非法命令)
     - 硬中断直接处理，SFM 致命时禁用设备
   * - CMDQ
     - 无独立中断；完成通过 MSI polling 或寄存器轮询检测；错误通过 GERROR 的 CMDQ_ERR 位处理

5.5 中断线程化分析
^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: text

                    硬件中断触发
                        |
          +-------------+-------------+
          |                           |
   [Combined 模式]              [Unique 模式]
          |                           |
  combined_irq_handler()         (无硬中断 handler)
  (处理GERROR, 唤醒线程)              |
          |                   +------+------+------+
  combined_irq_thread()       |      |      |      |
  (处理EVTQ+PRIQ)         [NULL]  [NULL]  gerror  [NULL]
                          |      |      handler  |
                   evtq_thread  priq_thread    |
                          |      |             |
                     处理EVTQ  处理PRIQ      处理GERROR
                     (纯线程)   (纯线程)    (硬中断上下文)

.. list-table::
   :header-rows: 1
   :widths: 15 35 35 20 20

   * - 特性
     - EVTQ
     - PRIQ
     - GERROR
     - Combined
   * - 硬中断 handler
     - ``NULL``
     - ``NULL``
     - ``gerror_handler``
     - ``combined_irq_handler``
   * - 线程 handler
     - ``evtq_thread``
     - ``priq_thread``
     - 无
     - ``combined_irq_thread``
   * - IRQF_ONESHOT
     - 是
     - 是
     - 否
     - 是
   * - 执行上下文
     - 纯线程
     - 纯线程
     - 硬中断
     - 硬中断 + 线程

设计原因:

- **EVTQ/PRIQ**: 需要遍历队列、调用 ``iommu_report_device_fault()`` 等可能休眠的操作，
  因此 handler 传 ``NULL``，所有处理在线程中完成
- **GERROR**: 处理较快 (仅读写寄存器 + 打印日志)，可在硬中断上下文中完成
- **Combined**: GERROR 放在硬中断中快速处理，EVTQ/PRIQ 放在线程中处理

所有线程化中断使用 ``IRQF_ONESHOT`` 确保在线程函数执行完毕后才重新启用
硬件中断线，防止同一线程中重复触发。

5.6 关键中断处理函数实现
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**arm_smmu_handle_evt()** (.c:1451-1543) -- EVTQ 事件分类处理

.. code-block:: c

   static int arm_smmu_handle_evt(struct arm_smmu_device *smmu, u64 *evt)
   {
       switch (FIELD_GET(EVTQ_0_ID, evt[0])) {
       case EVT_ID_TRANSLATION_FAULT: reason = IOMMU_FAULT_REASON_PTE_FETCH; break;
       case EVT_ID_ADDR_SIZE_FAULT:   reason = IOMMU_FAULT_REASON_OOR_ADDRESS; break;
       case EVT_ID_ACCESS_FAULT:      reason = IOMMU_FAULT_REASON_ACCESS; break;
       case EVT_ID_PERMISSION_FAULT:  reason = IOMMU_FAULT_REASON_PERMISSION; break;
       default: return -EOPNOTSUPP;
       }
       // Stage-2 fault 不支持恢复
       if (evt[1] & EVTQ_1_S2) return -EFAULT;
       // 根据 STALL 标志区分为可恢复/不可恢复 fault
       if (evt[1] & EVTQ_1_STALL) {
           flt->type = IOMMU_FAULT_PAGE_REQ;       // 可恢复 Page Request
           flt->prm.grpid = FIELD_GET(EVTQ_1_STAG, evt[1]);
       } else {
           flt->type = IOMMU_FAULT_DMA_UNRECOV;     // 不可恢复 DMA Fault
       }
       // 上报 IOMMU fault 框架
       ret = iommu_report_device_fault(master->dev, &fault_evt);
       // 无人处理 Page Request 则回复 FAILURE
       if (ret && flt->type == IOMMU_FAULT_PAGE_REQ)
           arm_smmu_page_response(master->dev, &fault_evt, &resp);
   }

**arm_smmu_gerror_handler()** (.c:1647-1691) -- GERROR 中断处理

.. code-block:: c

   static irqreturn_t arm_smmu_gerror_handler(int irq, void *dev)
   {
       gerror = readl_relaxed(smmu->base + ARM_SMMU_GERROR);
       gerrorn = readl_relaxed(smmu->base + ARM_SMMU_GERRORN);
       active = gerror ^ gerrorn;    // XOR 获取活跃错误
       if (!(active & GERROR_ERR_MASK)) return IRQ_NONE;

       if (active & GERROR_SFM_ERR)
           arm_smmu_device_disable(smmu);     // 致命: 禁用设备

       if (active & GERROR_CMDQ_ERR)
           arm_smmu_cmdq_skip_err(smmu);      // CMDQ 错误恢复

       writel(gerror, smmu->base + ARM_SMMU_GERRORN);  // 确认清除
       return IRQ_HANDLED;
   }

**__arm_smmu_cmdq_skip_err()** (.c:360-413) -- CMDQ 错误恢复

.. code-block:: c

   static void __arm_smmu_cmdq_skip_err(struct arm_smmu_device *smmu,
                                        struct arm_smmu_queue *q)
   {
       u32 cons = readl_relaxed(q->cons_reg);
       u32 idx = FIELD_GET(CMDQ_CONS_ERR, cons);
       switch (idx) {
       case CMDQ_ERR_CERROR_ABT_IDX:    // 命令获取中止 -> 重试
           return;
       case CMDQ_ERR_CERROR_NONE_IDX:   // 无错误 -> 返回
           return;
       case CMDQ_ERR_CERROR_ATC_INV_IDX: // ATC 无效超时 -> 返回 (重发SYNC恢复)
           return;
       case CMDQ_ERR_CERROR_ILL_IDX:    // 非法命令 -> 原地替换为CMD_SYNC跳过
       default:
           break;
       }
       // 将出错槽位原地替换为 CMD_SYNC
       queue_read(cmd, Q_ENT(q, cons), q->ent_dwords);
       arm_smmu_cmdq_build_cmd(cmd, &cmd_sync);
       queue_write(Q_ENT(q, cons), cmd, q->ent_dwords);
   }

**arm_smmu_combined_irq_handler()** (.c:1704-1708) -- 合并中断硬中断

.. code-block:: c

   static irqreturn_t arm_smmu_combined_irq_handler(int irq, void *dev)
   {
       arm_smmu_gerror_handler(irq, dev);   // 硬中断中仅处理 GERROR
       return IRQ_WAKE_THREAD;              // 唤醒线程处理 EVTQ+PRIQ
   }

**arm_smmu_combined_irq_thread()** (.c:1693-1702) -- 合并中断线程

.. code-block:: c

   static irqreturn_t arm_smmu_combined_irq_thread(int irq, void *dev)
   {
       struct arm_smmu_device *smmu = dev;
       arm_smmu_evtq_thread(irq, dev);
       if (smmu->features & ARM_SMMU_FEAT_PRI)
           arm_smmu_priq_thread(irq, dev);
       return IRQ_HANDLED;
   }
