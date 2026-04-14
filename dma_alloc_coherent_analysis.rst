dma_alloc_coherent 完整实现分析
====================================

基于 Linux 6.1-rc6 内核源码

1. 概述
--------

``dma_alloc_coherent`` 是 Linux 内核 DMA 映射框架的核心 API，用于为设备分配
CPU 和设备均可访问的一致性 DMA 内存缓冲区。该函数返回 CPU 虚拟地址，并通过
输出参数返回设备端的 DMA 总线地址。

.. contents:: 目录
   :local:
   :depth: 3

2. 完整调用链
---------------

2.1 入口函数
~~~~~~~~~~~~

**dma_alloc_coherent** — ``include/linux/dma-mapping.h:420``

::

   dma_alloc_coherent(dev, size, dma_handle, gfp)
   └── dma_alloc_attrs(dev, size, dma_handle, gfp, DMA_ATTR_NO_WARN?)
       └── kernel/dma/mapping.c:493

函数将 ``__GFP_NOWARN`` 转换为 ``DMA_ATTR_NO_WARN`` 属性，然后委托给
``dma_alloc_attrs`` 。

2.2 核心调度器
~~~~~~~~~~~~~~

**dma_alloc_attrs** — ``kernel/dma/mapping.c:493``

::

   dma_alloc_attrs()  [kernel/dma/mapping.c:493]
   │
   ├── get_dma_ops(dev)                        [include/linux/dma-map-ops.h:89]
   │   └── dev->dma_ops 或 get_arch_dma_ops()
   │
   ├── dma_alloc_from_dev_coherent()           [kernel/dma/coherent.c:185]
   │   └── __dma_alloc_from_coherent()         [kernel/dma/coherent.c:139]
   │       └── bitmap_find_free_region()        [位图分配]
   │
   ├── dma_alloc_direct(dev, ops)              [kernel/dma/mapping.c:131]
   │   └── dma_go_direct()                     [kernel/dma/mapping.c:112]
   │
   ├── [Direct 路径] dma_direct_alloc()        [kernel/dma/direct.c:208]
   │   或
   ├── [IOMMU 路径]  ops->alloc()              [drivers/iommu/dma-iommu.c:1442]
   │
   └── debug_dma_alloc_coherent()              [调试跟踪]

2.3 Direct 路径完整调用链
~~~~~~~~~~~~~~~~~~~~~~~~~~

**dma_direct_alloc** — ``kernel/dma/direct.c:208``

::

   dma_direct_alloc()  [kernel/dma/direct.c:208]
   │
   ├── [DMA_ATTR_NO_KERNEL_MAPPING]
   │   └── dma_direct_alloc_no_mapping()       [direct.c:190]
   │       └── __dma_direct_alloc_pages()      [direct.c:117]
   │
   ├── [非一致设备 + 无 arch 支持]
   │   └── arch_dma_alloc()                    [arch 特定实现]
   │
   ├── [CONFIG_DMA_GLOBAL_POOL + 非一致]
   │   └── dma_alloc_from_global_coherent()    [coherent.c:280]
   │       └── __dma_alloc_from_coherent()     [coherent.c:139]
   │
   ├── [非一致 + DMA_DIRECT_REMAP + 原子上下文]
   │   └── dma_direct_alloc_from_pool()        [direct.c:171]
   │       └── dma_alloc_from_pool()           [pool.c:265]
   │           └── __dma_alloc_from_pool()     [pool.c:240]
   │               └── gen_pool_alloc()        [通用内存池]
   │
   ├── [force_dma_unencrypted + 原子上下文]
   │   └── dma_direct_alloc_from_pool()        [direct.c:171]
   │
   ├── [主路径] __dma_direct_alloc_pages()     [direct.c:117]
   │   │
   │   ├── [SWIOTLB 强制]
   │   │   └── dma_direct_alloc_swiotlb()      [direct.c:105]
   │   │       └── swiotlb_alloc()             [kernel/dma/swiotlb.c]
   │   │
   │   ├── dma_direct_optimal_gfp_mask()       [direct.c:55]
   │   │   └── 计算 GFP 区域掩码
   │   │
   │   ├── dma_alloc_contiguous()              [contiguous.c:304]
   │   │   ├── dev->cma_area → cma_alloc()     [设备专属 CMA]
   │   │   ├── per-numa CMA → cma_alloc()     [NUMA 节点 CMA]
   │   │   └── dma_contiguous_default_area     [全局默认 CMA]
   │   │       └── cma_alloc_aligned()         [contiguous.c:282]
   │   │           └── cma_alloc()             [mm/cma.c]
   │   │
   │   ├── alloc_pages_node(node, gfp, order)  [mm/page_alloc.c]
   │   │   └── Buddy System 分配器
   │   │
   │   └── [DMA 地址越界时重试]
   │       ├── GFP_DMA32 重试                  [ZONE_DMA32]
   │       └── GFP_DMA 重试                    [ZONE_DMA]
   │
   ├── [HighMem 页面] → remap = true
   │
   ├── [remap == true]
   │   ├── dma_pgprot()                        [kernel/dma/direct.c:42]
   │   ├── arch_dma_prep_coherent()            [cache 清理]
   │   └── dma_common_contiguous_remap()       [remap.c:38]
   │       ├── kmalloc_array()                 [page 数组]
   │       └── vmap() → VM_DMA_COHERENT        [建立内核映射]
   │
   ├── [remap == false]
   │   ├── page_address(page)                  [直接映射地址]
   │   └── dma_set_decrypted()                 [内存解密(SEV/AMD)]
   │
   ├── memset(ret, 0, size)                   [清零]
   │
   ├── [set_uncached == true]
   │   ├── arch_dma_prep_coherent()
   │   └── arch_dma_set_uncached()
   │
   └── *dma_handle = phys_to_dma_direct()      [物理→总线地址转换]

2.4 IOMMU 路径完整调用链 (含 ARM SMMU)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**iommu_dma_alloc** — ``drivers/iommu/dma-iommu.c:1442``

::

   iommu_dma_alloc()  [drivers/iommu/dma-iommu.c:1442]
   │
   ├── dma_info_to_prot(DMA_BIDIRECTIONAL, coherent, attrs)  [dma-iommu.c:605]
   │   └── 转换 DMA 方向 → IOMMU 保护标志 (IOMMU_READ|IOMMU_WRITE|IOMMU_CACHE)
   │
   ├── [路径 A: 阻塞 + !FORCE_CONTIGUOUS] → 最常用路径
   │   └── iommu_dma_alloc_remap()             [dma-iommu.c:856]
   │       │
   │       ├── __iommu_dma_alloc_noncontiguous()  [dma-iommu.c:789]
   │       │   │
   │       │   ├── iommu_get_dma_domain(dev)        [获取 IOMMU domain]
   │       │   ├── iommu_dma_cookie / iova_domain   [IOVA 管理器]
   │       │   │
   │       │   ├── __iommu_dma_alloc_pages()      [dma-iommu.c:730]
   │       │   │   ├── kvcalloc(count, sizeof(*pages))  [页面指针数组]
   │       │   │   ├── gfp |= __GFP_HIGHMEM  [IOMMU 允许 HighMem]
   │       │   │   └── alloc_pages_node(nid, alloc_flags, order)
   │       │   │       └── Buddy System 分配 (非连续 OK)
   │       │   │
   │       │   ├── iommu_dma_alloc_iova()       [dma-iommu.c:625]
   │       │   │   ├── alloc_iova_fast(iovad, iova_len, dma_limit>>shift)
   │       │   │   │   └── iova_rcache_get() / alloc_iova()  [iova.c:439]
   │       │   │   └── PCI 设备优先分配 SAC (< 4GB) IOVA
   │       │   │
   │       │   ├── sg_alloc_table_from_pages()  [构建 sg_table]
   │       │   │
   │       │   ├── [非一致] arch_dma_prep_coherent()  [清理 cache]
   │       │   │
   │       │   └── iommu_map_sg_atomic()       [IOVA → 物理地址映射]
   │       │       └── ───→ 见下方 2.4.1 IOMMU Core 映射链 ───→
   │       │
   │       ├── *dma_handle = sgt.sgl->dma_address  [返回 IOVA]
   │       │
   │       └── dma_common_pages_remap()         [remap.c:22]
   │           └── vmap(pages, count, VM_DMA_COHERENT, prot)  [建立内核映射]
   │
   ├── [路径 B: 非阻塞 + 非一致 + DMA_DIRECT_REMAP]
   │   └── dma_alloc_from_pool()               [pool.c:265]
   │       └── gen_pool_alloc()                [原子池, 无 IOMMU 映射]
   │
   └── [路径 C: FORCE_CONTIGUOUS 或 回退]
       │
       ├── iommu_dma_alloc_pages()             [dma-iommu.c:1405]
       │   ├── dma_alloc_contiguous(dev, size, gfp)    [CMA 优先]
       │   ├── alloc_pages_node(node, gfp, order)      [Buddy 回退]
       │   ├── [非一致|HighMem] dma_common_contiguous_remap()
       │   └── [一致|LowMem] page_address(page)
       │
       └── __iommu_dma_map(dev, phys, size, prot, mask)  [dma-iommu.c:697]
           └── ───→ 见下方 2.4.1 IOMMU Core 映射链 ───→

2.4.1 IOMMU Core 映射链 (__iommu_dma_map → ARM SMMU)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   __iommu_dma_map(dev, phys, size, prot, dma_mask)  [dma-iommu.c:697]
   │
   ├── [延迟绑定检查]
   │   └── iommu_deferred_attach(dev, domain)     [iommu.c:1980]
   │       └── __iommu_attach_device(domain, dev) [首次使用时绑定]
   │
   ├── iommu_dma_alloc_iova(domain, size, dma_mask, dev)  [dma-iommu.c:625]
   │   ├── alloc_iova_fast(iovad, iova_len, dma_limit >> shift, true)
   │   │   └── iova_rcache_get() → alloc_iova()  [IOVA 空间分配]
   │   └── return (dma_addr_t)iova << shift      [IOVA 地址]
   │
   └── iommu_map_atomic(domain, iova, phys, size, prot)  [iommu.c:2317]
       └── _iommu_map(domain, iova, paddr, size, prot, GFP_ATOMIC)  [iommu.c:2296]
           │
           ├── __iommu_map()                     [iommu.c:2236]
           │   └── 循环: __iommu_map_pages()     [iommu.c:2212]
           │       ├── iommu_pgsize(domain, ...)  [确定页大小和数量]
           │       │
           │       ├── [优先] ops->map_pages()    [iommu_domain_ops:287]
           │       │   │
           │       │   ▼▼▼ ARM SMMUv3 路径 ▼▼▼
           │       │   │
           │       │   arm_smmu_map_pages()      [arm-smmu-v3.c:2472]
           │       │   └── pgtbl_ops->map_pages()
           │       │       │
           │       │       ▼▼▼ ARM LPAE 页表 ▼▼▼
           │       │       │
           │       │       arm_lpae_map_pages()  [io-pgtable-arm.c:464]
           │       │       ├── arm_lpae_prot_to_pte(data, iommu_prot)
           │       │       │   └── 将 IOMMU_READ|WRITE|CACHE 转为 PTE 位
           │       │       └── __arm_lpae_map(data, iova, paddr, ...)
           │       │           ├── 遍历页表层级 (PGD→PUD→PMD→PTE)
           │       │           ├── 自底向上分配页表页 (alloc_pages)
           │       │           ├── arm_lpae_install_table() [安装新页表]
           │       │           └── 写入 PTE: phys_addr | prot | valid
           │       │       └── wmb()  [写屏障, 确保页表可见后再继续]
           │       │
           │       └── [备选] ops->map()         [旧式单页映射]
           │
           └── [如定义] ops->iotlb_sync_map()    [SMMUv3 未定义此回调]

2.4.2 IOMMU 释放链 (iommu_dma_free)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   iommu_dma_free(dev, size, cpu_addr, handle, attrs)  [dma-iommu.c:1398]
   │
   ├── __iommu_dma_unmap(dev, handle, size)     [dma-iommu.c:674]
   │   ├── iommu_unmap_fast(domain, dma_addr, size, &iotlb_gather)  [iommu.c:2405]
   │   │   └── __iommu_unmap()                  [iommu.c:2337]
   │   │       └── 循环: __iommu_unmap_pages()
   │   │           └── ops->unmap_pages()       [iommu_domain_ops]
   │   │               │
   │   │               ▼▼▼ ARM SMMUv3 路径 ▼▼▼
   │   │               arm_smmu_unmap_pages()   [arm-smmu-v3.c:2484]
   │   │               └── pgtbl_ops->unmap_pages()
   │   │                   └── arm_lpae_unmap_pages()  [io-pgtable-arm.c:664]
   │   │                       ├── 遍历页表, 清除 PTE valid 位
   │   │                       └── 收集无效化范围到 iotlb_gather
   │   │
   │   ├── [如非 queued] iommu_iotlb_sync()     [TLB 同步]
   │   │   └── ops->iotlb_sync()
   │   │       └── arm_smmu_iotlb_sync()        [arm-smmu-v3.c:2505]
   │   │           └── arm_smmu_tlb_inv_range_domain()
   │   │               └── 发送 CMDQ_INV_TLB 超过队列到 SMMU 硬件
   │   │
   │   └── iommu_dma_free_iova()                [dma-iommu.c:657]
   │       └── free_iova_fast()                 [iova.c:487]
   │           └── iova_rcache_insert()          [IOVA 回收]
   │
   └── __iommu_dma_free(dev, size, cpu_addr)    [dma-iommu.c:1367]
       ├── [原子池] dma_free_from_pool()
       ├── [vmalloc] dma_common_free_remap() → vunmap()
       │   或 dma_common_find_pages() → __iommu_dma_free_pages()
       └── [低内存] dma_free_contiguous() → cma_release() / __free_pages()

3. 数据结构
------------

3.1 struct dma_map_ops
~~~~~~~~~~~~~~~~~~~~~~

**定义位置:** ``include/linux/dma-map-ops.h:22-84``

DMA 操作的虚函数表，定义了 DMA 映射的全部操作接口:

::

   struct dma_map_ops {
       unsigned int flags;                  /* DMA_F_PCI_P2PDMA_SUPPORTED 等 */

       /* 一致性内存分配/释放 */
       void *(*alloc)(struct device *dev, size_t size,
                       dma_addr_t *dma_handle, gfp_t gfp,
                       unsigned long attrs);
       void (*free)(struct device *dev, size_t size, void *vaddr,
                   dma_addr_t dma_handle, unsigned long attrs);

       /* 基于页面的分配 */
       struct page *(*alloc_pages)(...);
       void (*free_pages)(...);

       /* 非连续分配 (scatter-gather 表) */
       struct sg_table *(*alloc_noncontiguous)(...);
       void (*free_noncontiguous)(...);

       /* 流式 DMA 映射 / CPU 同步 / 能力查询 ... */
   };

**关键实现者:**

- ``iommu_dma_ops`` — ``drivers/iommu/dma-iommu.c:1547`` (IOMMU DMA 操作)
  ``.alloc = iommu_dma_alloc``, ``.free = iommu_dma_free``

3.2 struct iommu_ops
~~~~~~~~~~~~~~~~~~~~

**定义位置:** ``include/linux/iommu.h:229``

IOMMU 驱动注册的操作函数表:

::

   struct iommu_ops {
       /* 设备生命周期 */
       bool (*capable)(enum iommu_cap);
       struct iommu_domain *(*domain_alloc)(unsigned iommu_domain_type);
       struct device *(*probe_device)(struct device *dev);
       void (*release_device)(struct device *dev);

       /* 组和属性 */
       struct iommu_group *(*device_group)(struct device *dev);
       void (*get_resv_regions)(struct device *dev, struct list_head *head);
       enum iommu_resv_type (*get_resv_regions)(...);  /* (注: 6.1 改为新接口) */
       int (*of_xlate)(struct iommu_ops *ops, struct of_phandle_args *args);

       /* 域操作 (嵌入 default_domain_ops) */
       const struct iommu_domain_ops *default_domain_ops;

       unsigned long pgsize_bitmap;
       struct module *owner;
   };

3.3 struct iommu_domain_ops
~~~~~~~~~~~~~~~~~~~~~~~~~~~

**定义位置:** ``include/linux/iommu.h:287``

IOMMU 域操作，在 Linux 6.1 中从 iommu_ops 分离到独立结构体:

::

   struct iommu_domain_ops {
       int (*attach_dev)(struct iommu_domain *domain, struct device *dev);
       void (*detach_dev)(struct iommu_domain *domain, struct device *dev);

       /* 页表映射 - dma_alloc_coherent 走 map_pages 路径 */
       int (*map)(struct iommu_domain *, unsigned long iova,
                  phys_addr_t paddr, size_t size, int prot, gfp_t gfp);
       int (*map_pages)(struct iommu_domain *, unsigned long iova,
                        phys_addr_t paddr, size_t pgsize, size_t pgcount,
                        int prot, gfp_t gfp, size_t *mapped);

       /* 页表解除映射 */
       size_t (*unmap)(struct iommu_domain *, unsigned long iova,
                       size_t size, struct iommu_iotlb_gather *gather);
       size_t (*unmap_pages)(struct iommu_domain *, unsigned long iova,
                             size_t pgsize, size_t pgcount,
                             struct iommu_iotlb_gather *gather);

       /* TLB 刷新 */
       void (*flush_iotlb_all)(struct iommu_domain *domain);
       void (*iotlb_sync_map)(struct iommu_domain *domain,
                               unsigned long iova, size_t size);
       void (*iotlb_sync)(struct iommu_domain *domain,
                          struct iommu_iotlb_gather *iotlb_gather);

       /* 地址转换 & 嵌套 */
       phys_addr_t (*iova_to_phys)(struct iommu_domain *, dma_addr_t iova);
       int (*enable_nesting)(struct iommu_domain *domain);
       int (*set_pgtable_quirks)(struct iommu_domain *, unsigned long quirks);
       void (*free)(struct iommu_domain *domain);
   };

**ARM SMMUv3 实现** — ``arm-smmu-v3.c:2858``:

- ``.map_pages = arm_smmu_map_pages`` (:2472) — 委托给 ``pgtbl_ops->map_pages``
- ``.unmap_pages = arm_smmu_unmap_pages`` (:2484) — 委托给 ``pgtbl_ops->unmap_pages``
- ``.iotlb_sync = arm_smmu_iotlb_sync`` (:2505) — TLB 无效化同步
- ``.flush_iotlb_all = arm_smmu_flush_iotlb_all`` (:2497)
- ``.attach_dev = arm_smmu_attach_dev`` (:2396)

3.4 struct iommu_domain
~~~~~~~~~~~~~~~~~~~~~~~~

**定义位置:** ``include/linux/iommu.h:90``

::

   struct iommu_domain {
       unsigned type;                           /* DMA, DMA_FQ, IDENTITY, UNMANAGED */
       const struct iommu_domain_ops *ops;      /* 域操作 (SMMU 驱动设置) */
       unsigned long pgsize_bitmap;              /* 支持的页大小位图 */
       iommu_fault_handler_t handler;           /* 页面错误回调 */
       void *handler_token;
       struct iommu_domain_geometry geometry;    /* IOVA 窗口范围 */
       struct iommu_dma_cookie *iova_cookie;     /* DMA-IOMMU 层 IOVA 管理 */
   };

**域类型标志:**

- ``IOMMU_DOMAIN_DMA`` = ``__IOMMU_DOMAIN_PAGING | __IOMMU_DOMAIN_DMA_API``
- ``IOMMU_DOMAIN_DMA_FQ`` = 加上 ``__IOMMU_DOMAIN_DMA_FQ`` (flush queue 延迟 TLB 刷新)

3.5 struct iommu_domain_geometry
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**定义位置:** ``include/linux/iommu.h:54``

::

   struct iommu_domain_geometry {
       dma_addr_t aperture_start;   /* IOVA 空间起始地址 */
       dma_addr_t aperture_end;     /* IOVA 空间结束地址 */
       bool force_aperture;         /* 强制 IOVA 必须在窗口内 */
   };

3.6 struct io_pgtable_ops
~~~~~~~~~~~~~~~~~~~~~~~~~

**定义位置:** ``include/linux/io-pgtable.h:162``

IO 页表操作抽象层，SMMU 驱动通过此接口操作硬件页表:

::

   struct io_pgtable_ops {
       int (*map)(struct io_pgtable_ops *ops, unsigned long iova,
                  phys_addr_t paddr, size_t size, int prot, gfp_t gfp);
       int (*map_pages)(struct io_pgtable_ops *ops, unsigned long iova,
                        phys_addr_t paddr, size_t pgsize, size_t pgcount,
                        int prot, gfp_t gfp, size_t *mapped);
       size_t (*unmap)(struct io_pgtable_ops *ops, unsigned long iova,
                       size_t size, struct iommu_iotlb_gather *gather);
       size_t (*unmap_pages)(struct io_pgtable_ops *ops, unsigned long iova,
                             size_t pgsize, size_t pgcount,
                             struct iommu_iotlb_gather *gather);
       phys_addr_t (*iova_to_phys)(struct io_pgtable_ops *ops, unsigned long iova);
   };

**ARM LPAE 实现** — ``drivers/iommu/io-pgtable-arm.c:803``:

- ``.map_pages = arm_lpae_map_pages`` (:464) — 写入 LPAE 页表 PTE
- ``.unmap_pages = arm_lpae_unmap_pages`` (:664) — 清除 PTE valid 位

3.7 struct arm_smmu_device (SMMUv3)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**定义位置:** ``drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.h:618``

::

   struct arm_smmu_device {
       struct device          *dev;           /* SMMU 设备 */
       void __iomem           *base;          /* MMIO 寄存器基地址 */
       void __iomem           *page1;         /* MMIO 页面1 (可选) */

       u32                     features;      /* 特性位图 */
       /* 关键特性位: */
       /*   ARM_SMMU_FEAT_2_LVL_STRTAB  - 两级流表 */
       /*   ARM_SMMU_FEAT_TRANS_S1      - Stage 1 翻译 */
       /*   ARM_SMMU_FEAT_TRANS_S2      - Stage 2 翻译 */
       /*   ARM_SMMU_FEAT_ATS           - 地址转换服务 */
       /*   ARM_SMMU_FEAT_COHERENCY     - 一致性 walk */
       /*   ARM_SMMU_FEAT_SVA           - 共享虚拟地址 */
       /*   ARM_SMMU_FEAT_RANGE_INV     - 范围无效化 */

       struct arm_smmu_cmdq      cmdq;          /* 命令队列 */
       struct arm_smmu_evtq      evtq;          /* 事件队列 */
       struct arm_smmu_priq      priq;          /* 页面请求队列 */

       unsigned long            ias;           /* IO 地址空间位数 */
       unsigned long            oas;           /* 输出地址空间位数 */
       unsigned long            pgsize_bitmap; /* 支持的页大小 */

       u32                     asid_bits;      /* ASID 位数 */
       u32                     vmid_bits;      /* VMID 位数 */
       unsigned long           *vmid_map;       /* VMID 位图分配器 */
       u32                     ssid_bits;      /* SubStreamID 位数 */
       u32                     sid_bits;       /* StreamID 位数 */

       struct arm_smmu_strtab_cfg strtab_cfg;   /* 流表配置 */
       struct rb_root           streams;        /* Stream 入口红黑树 */
       struct iommu_device      iommu;          /* IOMMU core 句柄 */
   };

3.8 struct arm_smmu_domain (SMMUv3)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**定义位置:** ``drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.h:709``

::

   struct arm_smmu_domain {
       struct arm_smmu_device  *smmu;           /* 所属 SMMU */
       struct mutex             init_mutex;     /* 初始化互斥锁 */
       struct io_pgtable_ops   *pgtbl_ops;      /* IO 页表操作 */

       unsigned int             stall_enabled;  /* Stall 模式 */
       unsigned int             nr_ats_masters; /* ATS 使能的 master 数 */
       enum arm_smmu_domain_stage stage;        /* S1, S2, NESTED, BYPASS */

       union {
           struct arm_smmu_s1_cfg s1_cfg;       /* Stage 1 配置 */
           struct arm_smmu_s2_cfg s2_cfg;       /* Stage 2 配置 */
       };

       struct iommu_domain      domain;         /* 内嵌 iommu_domain */
       struct list_head         devices;        /* 域内设备链表 */
       spinlock_t               devices_lock;
       struct mmu_notifier      *mmu_notifiers;  /* SVA 通知器 */
   };

**Stage 1 配置 (s1_cfg) 关键字段:**

- ``cd.asid`` — 地址空间标识符
- ``cd.ttbr`` — 翻译表基地址寄存器 (页表根指针)
- ``cd.tcr`` — 翻译控制寄存器 (页表格式/大小)
- ``cd.mair`` — 内存属性间接寄存器 (cache 策略)
- ``s1cdmax`` — 最大 SubStreamID 值

**Stage 2 配置 (s2_cfg) 关键字段:**

- ``vmid`` — 虚拟机标识符
- ``vttbr`` — 虚拟化翻译表基地址
- ``vtcr`` — 虚拟化翻译控制寄存器

3.9 struct arm_smmu_master (SMMUv3)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**定义位置:** ``drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.h:686``

::

   struct arm_smmu_master {
       struct arm_smmu_device  *smmu;           /* 所属 SMMU */
       struct device           *dev;            /* 关联的设备 */
       struct arm_smmu_domain  *domain;         /* 当前绑定域 */
       struct list_head         domain_head;    /* 域内设备链表节点 */
       struct arm_smmu_stream  *streams;        /* Stream 描述符数组 */
       unsigned int             num_streams;    /* Stream 数量 */
       unsigned int             ats_enabled;    /* ATS 使能 */
       unsigned int             stall_enabled;  /* Stall 使能 */
       unsigned int             sva_enabled;    /* SVA 使能 */
       unsigned int             ssid_bits;      /* SSID 位数 */
   };

3.10 其他关键数据结构
~~~~~~~~~~~~~~~~~~~~~~~

**struct dma_coherent_mem** — ``kernel/dma/coherent.c:13-21``

::

   struct dma_coherent_mem {
       void           *virt_base;     /* CPU 虚拟地址基地址 */
       dma_addr_t      device_base;   /* 设备端 DMA 总线地址 */
       unsigned long   pfn_base;      /* 起始 PFN */
       int             size;          /* 大小 (页为单位) */
       unsigned long  *bitmap;        /* 位图分配器 */
       spinlock_t      spinlock;
       bool            use_dev_dma_pfn_offset;
   };

**struct cma** — ``include/linux/cma.h``

::

   struct cma {
       unsigned long   base_pfn;      /* CMA 区域起始 PFN */
       unsigned long   count;         /* 总页数 */
       unsigned long  *bitmap;        /* 位图管理器 */
       struct mutex    lock;
       const char     *name;
   };

**struct device 中的 DMA 成员** — ``include/linux/device.h:588-651``

::

   struct device {
       const struct dma_map_ops *dma_ops;           /* :588  */
       u64  *dma_mask;                              /* :590  */
       u64   coherent_dma_mask;                     /* :591  */
       u64   bus_dma_limit;                         /* :596  */
       struct device_dma_parameters *dma_parms;     /* :599  */
       struct dma_coherent_mem *dma_mem;            /* :604  */
       struct cma *cma_area;                        /* :608  */
       bool dma_coherent:1;                         /* :648  */
       bool dma_ops_bypass:1;                       /* :651  */
   };

**DMA 属性常量** — ``include/linux/dma-mapping.h:23-62``

::

   DMA_ATTR_WEAK_ORDERING       (1UL << 1)  /* 允许读写乱序 */
   DMA_ATTR_WRITE_COMBINE       (1UL << 2)  /* 写合并 */
   DMA_ATTR_NO_KERNEL_MAPPING   (1UL << 4)  /* 无内核映射 */
   DMA_ATTR_SKIP_CPU_SYNC       (1UL << 5)  /* 跳过 cache 同步 */
   DMA_ATTR_FORCE_CONTIGUOUS    (1UL << 6)  /* 强制物理连续 */
   DMA_ATTR_ALLOC_SINGLE_PAGES  (1UL << 7)  /* 单页分配提示 */
   DMA_ATTR_NO_WARN             (1UL << 8)  /* 抑制警告 */
   DMA_ATTR_PRIVILEGED          (1UL << 9)  /* 特权级 */

4. 执行阶段
------------

4.1 初始化阶段 (Boot Time)
~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   ┌─────────────────────────────────────────────────────────────────┐
   │                     初始化阶段调用链                            │
   ├─────────────────────────────────────────────────────────────────┤
   │                                                                 │
   │  1. early_param("cma", ...)                                     │
   │     └── 解析 cma= 内核参数  [kernel/dma/contiguous.c:78]        │
   │                                                                 │
   │  2. early_param("coherent_pool", ...)                           │
   │     └── 解析 coherent_pool=  [kernel/dma/pool.c:29]             │
   │                                                                 │
   │  3. SMMU 驱动探测 (platform_driver)                             │
   │     └── arm_smmu_device_probe() / arm_smmu_device_dt_probe()    │
   │         ├── 解析设备树 SMMU 节点                                │
   │         ├── 初始化 MMIO 寄存器、命令队列、事件队列               │
   │         ├── arm_smmu_init_strtab() → 分配流表                   │
   │         ├── arm_smmu_init_queues() → 初始化 CMDQ/EVTQ           │
   │         └── iommu_device_register() → 注册到 IOMMU core         │
   │                                                                 │
   │  4. dma_contiguous_reserve()  [kernel/dma/contiguous.c:167]      │
   │     └── 从 memblock 保留 CMA 物理区域                           │
   │                                                                 │
   │  5. postcore_initcall: dma_atomic_pool_init()                   │
   │     └── 初始化 3 个原子 DMA 池  [kernel/dma/pool.c:187]         │
   │                                                                 │
   │  6. 设备探测阶段 (Device Tree / ACPI)                            │
   │     │                                                            │
   │     ├── iommu_probe_device()                                    │
   │     │   └── SMMU: arm_smmu_probe_device()                       │
   │     │       ├── 解析 iommu-map / iommus 属性                    │
   │     │       ├── 分配 arm_smmu_master, 配置 stream IDs            │
   │     │       └── dev_iommu_priv_set(dev, master)                 │
   │     │                                                            │
   │     ├── arch_setup_dma_ops()  [arch/arm64/mm/dma-mapping.c:49]  │
   │     │   ├── dev->dma_coherent = coherent                        │
   │     │   └── [有 iommu] iommu_setup_dma_ops()                    │
   │     │       ├── iommu_get_domain_for_dev(dev)                   │
   │     │       ├── iommu_dma_init_domain()                         │
   │     │       │   └── init_iova_domain(iovad, base, limit)        │
   │     │       │       └── 初始化 IOVA 空间 + rcache               │
   │     │       └── dev->dma_ops = &iommu_dma_ops                   │
   │     │                                                            │
   │     └── iommu_attach_device(domain, dev)  [iommu.c:1952]        │
   │         └── arm_smmu_attach_dev()  [arm-smmu-v3.c:2396]         │
   │             ├── arm_smmu_domain_finalise()                      │
   │             │   ├── 选择 S1/S2 格式 (ARM_64_LPAE_S1/S2)        │
   │             │   ├── alloc_io_pgtable_ops(fmt, &cfg, domain)     │
   │             │   │   └── 分配页表根, 填充 cfg (TTBR/TCR/MAIR)   │
   │             │   ├── [S1] arm_smmu_domain_finalise_s1()          │
   │             │   │   ├── xa_alloc(&arm_smmu_asid_xa) [分配ASID] │
   │             │   │   ├── arm_smmu_alloc_cd_tables() [CD 表]      │
   │             │   │   └── arm_smmu_write_ctx_desc() [写 CD]      │
   │             │   ├── [S2] arm_smmu_domain_finalise_s2()          │
   │             │   │   ├── arm_smmu_bitmap_alloc(vmid_map) [VMID]  │
   │             │   │   └── 配置 VTTBR/VTCR                         │
   │             │   └── smmu_domain->pgtbl_ops = pgtbl_ops          │
   │             ├── arm_smmu_install_ste_for_dev()                  │
   │             │   └── arm_smmu_write_strtab_ent() [写 STE]        │
   │             └── arm_smmu_enable_ats()  [可选]                    │
   │                                                                 │
   └─────────────────────────────────────────────────────────────────┘

4.2 运行时阶段 (IOMMU 路径分配)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   ┌─────────────────────────────────────────────────────────────────┐
   │                dma_alloc_coherent IOMMU 路径决策树              │
   ├─────────────────────────────────────────────────────────────────┤
   │                                                                 │
   │  dma_alloc_coherent(dev, size, &handle, gfp)                   │
   │    └── dma_alloc_attrs()                                       │
   │        └── [dma_ops == iommu_dma_ops]                           │
   │            └── iommu_dma_alloc()                                │
   │                │                                                │
   │                ├─ 可阻塞 且 !FORCE_CONTIGUOUS?                  │
   │                │  └── YES: 路径 A (最常用)                       │
   │                │      └── iommu_dma_alloc_remap()               │
   │                │          ├── 分配非连续物理页 (Buddy)           │
   │                │          ├── 分配 IOVA                          │
   │                │          ├── iommu_map_sg_atomic()              │
   │                │          │   └── 写入 ARM LPAE 页表             │
   │                │          └── vmap() 建立内核连续虚拟映射       │
   │                │                                                │
   │                ├─ 不可阻塞 且 非一致 且 DMA_DIRECT_REMAP?       │
   │                │  └── YES: 路径 B (原子池)                      │
   │                │      └── dma_alloc_from_pool()                 │
   │                │          └── gen_pool_alloc() [无 IOMMU 映射]   │
   │                │                                                │
   │                └── 默认: 路径 C (连续)                           │
   │                    ├── iommu_dma_alloc_pages()                  │
   │                    │   ├── CMA 或 Buddy 分配连续物理页          │
   │                    │   └── [非一致] vmap() 建立内核映射          │
   │                    └── __iommu_dma_map()                        │
   │                        ├── alloc_iova_fast() [分配 IOVA]        │
   │                        └── iommu_map_atomic()                  │
   │                            └── arm_lpae_map_pages()             │
   │                                └── 写 ARM LPAE 页表 PTE        │
   │                                                                 │
   │  返回: (cpu_addr, dma_handle=IOVA)                               │
   │                                                                 │
   └─────────────────────────────────────────────────────────────────┘

4.3 清理阶段 (IOMMU 释放路径)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   ┌─────────────────────────────────────────────────────────────────┐
   │                dma_free_coherent IOMMU 释放路径                  │
   ├─────────────────────────────────────────────────────────────────┤
   │                                                                 │
   │  dma_free_coherent(dev, size, cpu_addr, handle)                │
   │    └── dma_free_attrs()                                         │
   │        └── [dma_ops == iommu_dma_ops]                           │
   │            └── iommu_dma_free()                                 │
   │                │                                                │
   │                ├── __iommu_dma_unmap(dev, handle, size)         │
   │                │   ├── iommu_unmap_fast()                       │
   │                │   │   └── arm_lpae_unmap_pages()               │
   │                │   │       └── 清除 PTE valid 位                 │
   │                │   │                                            │
   │                │   ├── arm_smmu_iotlb_sync() [TLB 无效化]        │
   │                │   │   └── arm_smmu_tlb_inv_range_domain()      │
   │                │   │       └── CMDQ_INV_TLB → SMMU 硬件        │
   │                │   │                                            │
   │                │   └── free_iova_fast() [回收 IOVA]             │
   │                │                                                │
   │                └── __iommu_dma_free(dev, size, cpu_addr)        │
   │                    ├── [原子池] dma_free_from_pool()             │
   │                    ├── [vmalloc] vunmap()                       │
   │                    └── [低内存] dma_free_contiguous()           │
   │                                                                 │
   └─────────────────────────────────────────────────────────────────┘

5. 关键内核 API 调用
----------------------

5.1 内存分配 API
~~~~~~~~~~~~~~~~

+---------------------------+-------------------------------------------+---------------------------------------+
| API                       | 作用                                      | 所在文件                              |
+===========================+===========================================+=======================================+
| ``alloc_pages_node()``    | Buddy System 分配 2^n 连续物理页           | ``mm/page_alloc.c``                   |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``cma_alloc()``           | CMA 区域分配连续物理页                     | ``mm/cma.c``                          |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``gen_pool_alloc()``      | 通用内存池原子分配                         | ``lib/genalloc.c``                    |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``swiotlb_alloc()``       | SWIOTLB 弹性缓冲区分配                     | ``kernel/dma/swiotlb.c``              |
+---------------------------+-------------------------------------------+---------------------------------------+

5.2 IOVA 管理 API
~~~~~~~~~~~~~~~~~

+---------------------------------+-------------------------------------+-----------------------------------+
| API                             | 作用                                | 所在文件                          |
+=================================+=====================================+===================================+
| ``alloc_iova_fast()``           | 从 IOVA rcache/空间分配 IOVA         | ``drivers/iommu/iova.c:439``      |
+---------------------------------+-------------------------------------+-----------------------------------+
| ``free_iova_fast()``            | 回收 IOVA 到 rcache                 | ``drivers/iommu/iova.c:487``      |
+---------------------------------+-------------------------------------+--------------------------------===+
| ``init_iova_domain()``          | 初始化 IOVA 域                      | ``drivers/iommu/iova.c``          |
+---------------------------------+-------------------------------------+-----------------------------------+

5.3 IOMMU Core 映射 API
~~~~~~~~~~~~~~~~~~~~~~~~~

+---------------------------------+-------------------------------------+-----------------------------------+
| API                             | 作用                                | 所在文件                          |
+=================================+=====================================+===================================+
| ``iommu_map_atomic()``          | 原子上下文建立 IOVA→PA 映射          | ``drivers/iommu/iommu.c:2317``    |
+---------------------------------+-------------------------------------+-----------------------------------+
| ``iommu_unmap_fast()``          | 快速解除 IOVA 映射 (gather TLB)      | ``drivers/iommu/iommu.c:2405``    |
+---------------------------------+-------------------------------------+--------------------------------===+
| ``iommu_attach_device()``       | 将设备绑定到 IOMMU 域                | ``drivers/iommu/iommu.c:1952``    |
+---------------------------------+-------------------------------------+-----------------------------------+
| ``iommu_deferred_attach()``     | 延迟绑定设备到域 (首次 DMA 时触发)   | ``drivers/iommu/iommu.c:1980``    |
+---------------------------------+-------------------------------------+--------------------------------===+
| ``iommu_get_dma_domain()``      | 获取设备的 DMA 域                    | ``drivers/iommu/iommu.c``         |
+---------------------------------+-------------------------------------+--------------------------------===+

5.4 DMA-IOMMU 桥接 API
~~~~~~~~~~~~~~~~~~~~~~~~~

+---------------------------------+-------------------------------------+-----------------------------------+
| API                             | 作用                                | 所在文件                          |
+=================================+=====================================+===================================+
| ``iommu_setup_dma_ops()``       | 安装 iommu_dma_ops 到设备           | ``drivers/iommu/dma-iommu.c:1575``|
+---------------------------------+-------------------------------------+-----------------------------------+
| ``iommu_dma_init_domain()``     | 初始化 IOVA cookie 和 iova_domain   | ``drivers/iommu/dma-iommu.c``     |
+---------------------------------+-------------------------------------+-----------------------------------+
| ``dma_info_to_prot()``          | DMA 方向→IOMMU 保护标志转换          | ``drivers/iommu/dma-iommu.c:605`` |
+---------------------------------+-------------------------------------+-----------------------------------+
| ``__iommu_dma_map()``           | 分配 IOVA + 建立 IOVA→PA 映射       | ``drivers/iommu/dma-iommu.c:697`` |
+---------------------------------+-------------------------------------+-----------------------------------+
| ``__iommu_dma_unmap()``         | 解除映射 + 释放 IOVA                | ``drivers/iommu/dma-iommu.c:674`` |
+---------------------------------+-------------------------------------+--------------------------------===+
| ``iommu_dma_alloc_iova()``      | 分配 IOVA 地址                      | ``drivers/iommu/dma-iommu.c:625`` |
+---------------------------------+-------------------------------------+-----------------------------------+

5.5 ARM SMMUv3 硬件操作 API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

+---------------------------------+-------------------------------------+-----------------------------------+
| API                             | 作用                                | 所在文件                          |
+=================================+=====================================+===================================+
| ``arm_smmu_attach_dev()``      | 设备绑定域, 初始化页表, 写入 STE     | ``arm-smmu-v3.c:2396``            |
+---------------------------------+-------------------------------------+-----------------------------------+
| ``arm_smmu_domain_finalise()``  | 初始化域 (S1/S2, ASID/VMID, 页表)   | ``arm-smmu-v3.c:2164``            |
+---------------------------------+-------------------------------------+--------------------------------===+
| ``arm_smmu_map_pages()``        | 映射页面 → pgtbl_ops->map_pages     | ``arm-smmu-v3.c:2472``            |
+---------------------------------+-------------------------------------+-----------------------------------+
| ``arm_smmu_unmap_pages()``      | 解除映射 → pgtbl_ops->unmap_pages   | ``arm-smmu-v3.c:2484``            |
+---------------------------------+-------------------------------------+--------------------------------===+
| ``arm_smmu_iotlb_sync()``       | TLB 范围无效化同步                   | ``arm-smmu-v3.c:2505``            |
+---------------------------------+-------------------------------------+--------------------------------===+
| ``arm_smmu_tlb_inv_context()``  | 域级 TLB 全量无效化                  | ``arm-smmu-v3.c:1843``            |
+---------------------------------+-------------------------------------+--------------------------------===+
| ``arm_smmu_write_strtab_ent()`` | 编程 Stream Table Entry (STE)       | ``arm-smmu-v3.c:1239``            |
+---------------------------------+-------------------------------------+--------------------------------===+
| ``arm_smmu_write_ctx_desc()``   | 编程 Context Descriptor (CD)        | ``arm-smmu-v3.c:1038``            |
+---------------------------------+-------------------------------------+--------------------------------===+
| ``arm_smmu_install_ste_for_dev()``| 为设备所有 stream 写入 STE          | ``arm-smmu-v3.c:2257``            |
+---------------------------------+-------------------------------------+--------------------------------===+

5.6 ARM LPAE 页表 API
~~~~~~~~~~~~~~~~~~~~~~

+---------------------------------+-------------------------------------+-----------------------------------+
| API                             | 作用                                | 所在文件                          |
+=================================+=====================================+===================================+
| ``arm_lpae_map_pages()``        | 写入 LPAE 页表 PTE (IOVA→PA)       | ``io-pgtable-arm.c:464``          |
+---------------------------------+-------------------------------------+--------------------------------===+
| ``arm_lpae_unmap_pages()``      | 清除 PTE valid 位                   | ``io-pgtable-arm.c:664``          |
+---------------------------------+-------------------------------------+--------------------------------===+
| ``arm_lpae_prot_to_pte()``      | IOMMU 保护标志→PTE 硬件编码         | ``io-pgtable-arm.c``              |
+---------------------------------+-------------------------------------+--------------------------------===+
| ``alloc_io_pgtable_ops()``      | 分配 IO 页表 (PGD + 回调)           | ``include/linux/io-pgtable.h:187``|
+---------------------------------+-------------------------------------+--------------------------------===+

5.7 内存映射与属性 API
~~~~~~~~~~~~~~~~~~~~~~~~~

+---------------------------+-------------------------------------------+---------------------------------------+
| ``vmap()``                | 离散物理页→连续内核虚拟地址              | ``mm/vmalloc.c``                      |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``vunmap()``              | 取消 vmap 映射                           | ``mm/vmalloc.c``                      |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``page_address()``        | 低内存页面→内核虚拟地址                  | ``include/linux/mm.h``                |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``dma_pgprot()``          | 设备 DMA 属性→页面保护位                 | ``kernel/dma/direct.c``               |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``arch_dma_prep_coherent()``| 清理 cache 行 (ARM64: dcache_clean_poc)| 架构实现                              |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``arch_sync_dma_for_device()``| CPU cache → 设备同步 (ARM64)          | ``arch/arm64/mm/dma-mapping.c``      |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``arch_sync_dma_for_cpu()``   | 设备 → CPU cache 同步 (ARM64)         | ``arch/arm64/mm/dma-mapping.c``      |
+---------------------------+-------------------------------------------+---------------------------------------+

6. 组件交互图
---------------

6.1 IOMMU 路径完整架构图
~~~~~~~~~~~~~~~~~~~~~~~~~

::

   ┌─────────────────────────────────────────────────────────────────────┐
   │                      DMA 驱动程序 (Driver)                           │
   │   dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL)            │
   └─────────────────────────────┬───────────────────────────────────────┘
                                   │
                                   ▼
   ┌─────────────────────────────────────────────────────────────────────┐
   │                   DMA Mapping Framework                             │
   │   dma_alloc_attrs() → dma_alloc_direct? → NO → ops->alloc          │
   └─────────────────────────────┬───────────────────────────────────────┘
                                   │
                                   ▼
   ┌─────────────────────────────────────────────────────────────────────┐
   │                 dma-iommu 层 (DMA-IOMMU Glue)                        │
   │                 drivers/iommu/dma-iommu.c                            │
   │                                                                      │
   │  iommu_dma_ops.alloc = iommu_dma_alloc()                           │
   │      │                                                               │
   │      ├── 页面分配: alloc_pages / dma_alloc_contiguous               │
   │      ├── IOVA 管理: alloc_iova_fast / free_iova_fast                │
   │      ├── 映射接口:   __iommu_dma_map / __iommu_dma_unmap            │
   │      └── 内核映射:   vmap (VM_DMA_COHERENT) / vunmap               │
   │                                                                      │
   └──────────┬──────────────────────────┬───────────────────────────────┘
              │ __iommu_dma_map          │ __iommu_dma_unmap
              ▼                          ▼
   ┌─────────────────────────┐  ┌─────────────────────────────┐
   │ IOMMU Core              │  │ IOMMU Core (unmap)          │
   │ drivers/iommu/iommu.c   │  │ drivers/iommu/iommu.c       │
   │                         │  │                             │
   │ iommu_map_atomic()      │  │ iommu_unmap_fast()          │
   │   └── __iommu_map()     │  │   └── __iommu_unmap()       │
   │       └── ops->map_pages│  │       └── ops->unmap_pages  │
   └──────────┬──────────────┘  └──────────┬──────────────────┘
              │ domain_ops                  │ domain_ops
              ▼                             ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │              ARM SMMUv3 驱动                                      │
   │              drivers/iommu/arm/arm-smmu-v3/                      │
   │                                                                  │
   │  arm_smmu_map_pages()   arm_smmu_unmap_pages()                   │
   │      │                        │                                  │
   │      └── pgtbl_ops ─────────────┘                                │
   │                                                                  │
   │  arm_smmu_iotlb_sync()    ← TLB 无效化同步                      │
   │  arm_smmu_attach_dev()    ← 设备绑定 (STE/CD 编程)              │
   │  arm_smmu_domain_finalise() ← 域初始化 (ASID/VMID/页表)         │
   └──────────┬──────────────────────────────────────────────────────┘
              │ pgtbl_ops
              ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │              ARM LPAE IO 页表                                    │
   │              drivers/iommu/io-pgtable-arm.c                      │
   │                                                                  │
   │  arm_lpae_map_pages()    arm_lpae_unmap_pages()                 │
   │      │                        │                                 │
   │      └── __arm_lpae_map() ────┘                                │
   │          ├── 遍历页表层级: PGD → PUD → PMD → PTE               │
   │          ├── alloc_pages() 分配页表页                            │
   │          ├── arm_lpae_install_table() 安装子表                  │
   │          └── 写入/清除 PTE: PA | prot | valid                    │
   │                                                                  │
   └──────────┬──────────────────────────────────────────────────────┘
              │ 页表写入内存 (DDR)
              ▼
   ┌─────────────────────────────────────────────────────────────────┐
   │              SMMUv3 硬件                                         │
   │                                                                  │
   │  Stream Table (STE) ──→ Context Descriptor (CD) ──→ Page Table  │
   │                                                                  │
   │  DMA 事务:                                                       │
   │    Device ──[StreamID/SSID]──→ STE ──→ CD ──→ PT Walk           │
   │                                                  │               │
   │    IOVA ──────────────────────────────────→ PA ──→ DDR          │
   │                                                                  │
   │  TLB: 缓存 IOVA→PA 映射                                          │
   │  CMDQ: 接收 TLB 无效化等命令                                     │
   │  EVTQ: 报告事件 (fault 等)                                       │
   └─────────────────────────────────────────────────────────────────┘

6.2 SMMUv3 地址翻译流水线
~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   ┌─────────────────────────────────────────────────────────────────┐
   │              SMMUv3 地址翻译流水线 (Stage 1)                     │
   │                                                                  │
   │  Device 发出 DMA 事务:                                          │
   │                                                                  │
   │  ┌──────────┐    ┌──────────────┐    ┌───────────────┐          │
   │  │ StreamID │───→│   Stream     │    │   Context     │          │
   │  │ (SID)    │    │   Table      │    │   Descriptor  │          │
   │  │          │    │   Entry      │    │   Table       │          │
   │  │          │    │   (STE)      │───→│   (CD)        │          │
   │  └──────────┘    │              │    │               │          │
   │                  │ S1CTXPTR     │    │ TTBR ← 页表根 │          │
   │                  │ S1CDMAX      │    │ TCR  ← 格式   │          │
   │                  │ V (=valid)   │    │ MAIR ← 属性   │          │
   │                  │ CFG=S1_TRANS │    │ ASID          │          │
   │                  └──────────────┘    └───────┬───────┘          │
   │                                              │                   │
   │                                       TTBR 指向页表根            │
   │                                              │                   │
   │  IOVA ──────────────────────────────────────┘                   │
   │                                              │                   │
   │                          ┌───────────────────┘                   │
   │                          ▼                                       │
   │  ┌──────────────────────────────────────────┐                    │
   │  │           ARM LPAE 页表 Walk              │                    │
   │  │                                          │                    │
   │  │   PGD[i] ──→ PUD[j] ──→ PMD[k] ──→ PTE[l]│                   │
   │  │                                          │                    │
   │  │   每级:                                  │                    │
   │  │   ├── 读取表项 (从 DDR, 可选 coherent)    │                    │
   │  │   ├── 检查 Valid 位                      │                    │
   │  │   ├── 检查权限 (R/W)                     │                    │
   │  │   ├── 检查 MAIR/TCR 属性                │                    │
   │  │   └── 结果缓存到 TLB                    │                    │
   │  └──────────────────┬───────────────────────┘                    │
   │                     │                                            │
   │                     ▼                                            │
   │  ┌──────────────────────────────────────────┐                    │
   │  │           物理地址 (PA)                   │                    │
   │  │           ──→ DDR 访问                    │                    │
   │  └──────────────────────────────────────────┘                    │
   │                                                                  │
   └─────────────────────────────────────────────────────────────────┘

6.3 DMA 地址空间对比 (Direct vs IOMMU)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   ┌──────────────────────────────────────────────────────────────────┐
   │                                                                  │
   │   Direct Path (无 IOMMU 或 Bypass)                               │
   │   ─────────────────────────────────                              │
   │                                                                  │
   │   CPU ──→ [VA] ──→ [MMU] ──→ [PA] ──→ DDR                       │
   │                                  │                                │
   │   Device ──→ [DMA=PA] ──────────┘                                │
   │                                                                  │
   │   限制: 设备 DMA 地址 = 物理地址, 受限于物理内存范围              │
   │                                                                  │
   ├──────────────────────────────────────────────────────────────────┤
   │                                                                  │
   │   IOMMU Path (SMMU)                                              │
   │   ──────────────────                                              │
   │                                                                  │
   │   CPU ──→ [VA] ──→ [MMU] ──→ [PA] ──→ DDR                       │
   │                                  │                                │
   │   Device ──→ [IOVA] ──→ [SMMU STE→CD→PT] ──→ [PA] ──→ DDR      │
   │                                                                  │
   │   优势:                                                          │
   │   • IOVA 独立于物理地址, 设备可访问全部内存                      │
   │   • 内存隔离: 不同域的设备看到不同的地址空间                     │
   │   • 地址聚合: 非连续物理页映射为连续 IOVA                        │
   │   • 安全: 设备无法访问未映射的物理区域                            │
   │                                                                  │
   │   dma_alloc_coherent 返回的 dma_handle 是 IOVA, 不是物理地址    │
   │                                                                  │
   └──────────────────────────────────────────────────────────────────┘

7. 关键文件索引
----------------

+------------------------------------------+-----------------------------------------+
| 文件路径                                 | 功能描述                                |
+==========================================+=========================================+
| **DMA Mapping 层**                       |                                         |
+------------------------------------------+-----------------------------------------+
| ``include/linux/dma-mapping.h``          | API 声明, DMA_ATTR_* 定义              |
| ``include/linux/dma-map-ops.h``          | struct dma_map_ops, get_dma_ops         |
| ``include/linux/device.h``               | struct device (DMA 成员 :588-651)       |
| ``kernel/dma/mapping.c``                 | dma_alloc_attrs 核心调度器              |
| ``kernel/dma/direct.c``                  | Direct 映射分配器                       |
| ``kernel/dma/coherent.c``                | 设备一致性内存池                        |
| ``kernel/dma/pool.c``                    | 原子 DMA 池 (gen_pool)                 |
| ``kernel/dma/contiguous.c``              | CMA 分配器                              |
| ``kernel/dma/remap.c``                   | vmap/vunmap 帮助函数                    |
| **IOMMU 层**                             |                                         |
+------------------------------------------+-----------------------------------------+
| ``include/linux/iommu.h``                | struct iommu_ops/domain/domain_ops     |
| ``include/linux/io-pgtable.h``           | struct io_pgtable_ops 接口              |
| ``drivers/iommu/iommu.c``                | iommu_map_atomic/unmap_fast 核心       |
| ``drivers/iommu/iova.c``                 | IOVA 空间分配器 (alloc/free_iova_fast) |
| ``drivers/iommu/dma-iommu.c``            | IOMMU-DMA 桥接 (iommu_dma_ops)         |
| **ARM SMMU 驱动**                        |                                         |
+------------------------------------------+-----------------------------------------+
| ``drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c`` | SMMUv3 驱动 (map/unmap/STE/CD)    |
| ``drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.h`` | SMMUv3 数据结构定义                  |
| ``drivers/iommu/arm/arm-smmu/arm-smmu.c``    | SMMUv1/v2 驱动                      |
| ``drivers/iommu/arm/arm-smmu/arm-smmu.h``    | SMMUv1/v2 数据结构定义                |
| ``drivers/iommu/io-pgtable-arm.c``       | ARM LPAE 页表实现 (map_pages/unmap)    |
| **ARM64 架构钩子**                       |                                         |
+------------------------------------------+-----------------------------------------+
| ``arch/arm64/mm/dma-mapping.c``          | arch_setup_dma_ops, cache sync hooks   |
| ``arch/arm64/mm/cache.S``                | dcache_clean_poc / dcache_inval_poc    |
| **内存分配**                              |                                         |
+------------------------------------------+-----------------------------------------+
| ``mm/page_alloc.c``                      | Buddy System 分配器                    |
| ``mm/cma.c``                             | CMA 核心实现                            |
| ``mm/vmalloc.c``                         | vmap/vunmap 实现                        |
| ``lib/genalloc.c``                       | 通用内存池 (gen_pool)                   |
| ``lib/bitmap.c``                         | 位图操作                                |
