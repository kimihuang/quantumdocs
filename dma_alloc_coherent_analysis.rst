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

2.4 IOMMU 路径调用链
~~~~~~~~~~~~~~~~~~~~~

**iommu_dma_alloc** — ``drivers/iommu/dma-iommu.c:1442``

::

   iommu_dma_alloc()  [drivers/iommu/dma-iommu.c:1442]
   │
   ├── [阻塞 + !FORCE_CONTIGUOUS]
   │   └── iommu_dma_alloc_remap()             [dma-iommu.c:1351]
   │       ├── alloc_pages()                   [分配页面]
   │       ├── dma_common_pages_remap()        [remap.c:22]
   │       │   └── vmap() → VM_DMA_COHERENT
   │       └── __iommu_dma_map()               [IOVA 映射]
   │           └── iommu_map()                 [IOMMU 驱动]
   │
   ├── [非阻塞 + 非一致 + DMA_DIRECT_REMAP]
   │   └── dma_alloc_from_pool()               [pool.c:265]
   │       └── gen_pool_alloc()                [原子池]
   │
   └── [默认路径]
       ├── iommu_dma_alloc_pages()             [dma-iommu.c:1405]
       │   ├── dma_alloc_contiguous()          [CMA]
       │   └── alloc_pages_node()              [Buddy]
       │
       ├── dma_common_contiguous_remap()       [如需要]
       └── __iommu_dma_map()                   [IOVA 映射]
           └── iommu_map()                     [IOMMU 页表映射]

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

       /* 用户空间映射 */
       int (*mmap)(struct device *, struct vm_area_struct *,
                   void *, dma_addr_t, size_t, unsigned long attrs);

       /* Scatter-gather 表获取 */
       int (*get_sgtable)(struct device *dev, struct sg_table *sgt,
                          void *cpu_addr, dma_addr_t dma_addr,
                          size_t size, unsigned long attrs);

       /* 流式 DMA 映射 */
       dma_addr_t (*map_page)(...);
       void (*unmap_page)(...);
       int (*map_sg)(...);
       void (*unmap_sg)(...);
       dma_addr_t (*map_resource)(...);
       void (*unmap_resource)(...);

       /* CPU/设备同步 */
       void (*sync_single_for_cpu)(...);
       void (*sync_single_for_device)(...);
       void (*sync_sg_for_cpu)(...);
       void (*sync_sg_for_device)(...);
       void (*cache_sync)(...);

       /* 能力查询 */
       int (*dma_supported)(struct device *dev, u64 mask);
       u64 (*get_required_mask)(struct device *dev);
       size_t (*max_mapping_size)(struct device *dev);
       size_t (*opt_mapping_size)(void);
       unsigned long (*get_merge_boundary)(struct device *dev);
   };

**关键实现者:**

- ``dma_dummy_ops`` — ``kernel/dma/dummy.c`` (空操作，用于无 DMA 能力的设备)
- ``iommu_dma_ops`` — ``drivers/iommu/dma-iommu.c:1547`` (IOMMU DMA 操作)

3.2 struct dma_coherent_mem
~~~~~~~~~~~~~~~~~~~~~~~~~~~

**定义位置:** ``kernel/dma/coherent.c:13-21``

每设备一致性内存池，由 ``dma_declare_coherent_memory()`` 从保留内存区域创建:

::

   struct dma_coherent_mem {
       void           *virt_base;     /* CPU 虚拟地址基地址 (ioremap 后) */
       dma_addr_t      device_base;   /* 设备端看到的 DMA 总线地址基地址 */
       unsigned long   pfn_base;      /* 起始页帧号 */
       int             size;          /* 内存区域大小 (以页为单位) */
       unsigned long  *bitmap;        /* 位图分配器，追踪已分配页 */
       spinlock_t      spinlock;      /* 保护 bitmap 的自旋锁 */
       bool            use_dev_dma_pfn_offset;  /* 是否使用 PFN 偏移 */
   };

**使用方式:**

- 初始化: ``dma_declare_coherent_memory()`` — ``coherent.c:117``
- 分配: ``dma_alloc_from_dev_coherent()`` → ``__dma_alloc_from_coherent()`` → ``bitmap_find_free_region()``
- 释放: ``dma_release_from_dev_coherent()`` → ``__dma_release_from_coherent()`` → ``bitmap_release_region()``

3.3 struct device 中的 DMA 相关成员
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**定义位置:** ``include/linux/device.h:588-651``

::

   struct device {
       /* DMA 映射操作 (CONFIG_DMA_OPS) */
       const struct dma_map_ops *dma_ops;           /* :588  DMA 操作函数表 */

       /* DMA 地址掩码 */
       u64  *dma_mask;                              /* :590  流式 DMA 掩码 */
       u64   coherent_dma_mask;                     /* :591  一致性 DMA 掩码 */

       /* DMA 约束 */
       u64   bus_dma_limit;                         /* :596  上游桥/总线 DMA 限制 */
       struct device_dma_parameters *dma_parms;     /* :599  段大小等参数 */

       /* 一致性内存 */
       struct dma_coherent_mem *dma_mem;            /* :604  设备专属一致性内存池 */
       struct cma *cma_area;                        /* :608  设备专属 CMA 区域 */

       /* DMA 属性标志位 */
       bool dma_coherent:1;                         /* :648  设备是否 DMA 一致 */
       bool dma_ops_bypass:1;                       /* :651  是否绕过 dma_ops */
   };

3.4 struct cma (Contiguous Memory Allocator)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**定义位置:** ``include/linux/cma.h``

CMA 管理器结构，用于在启动后分配大块连续物理内存:

::

   struct cma {
       unsigned long   base_pfn;      /* CMA 区域起始 PFN */
       unsigned long   count;         /* 总页数 */
       unsigned long   *bitmap;       /* 位图管理器 */
       struct mutex    lock;          /* 互斥锁 */
       const char     *name;          /* CMA 名称 */
       /* ... 其他管理字段 ... */
   };

**关键全局变量:**

- ``dma_contiguous_default_area`` — ``kernel/dma/contiguous.c:60``
  全局默认 CMA 区域，由 ``dma_contiguous_reserve()`` 初始化

3.5 DMA 属性常量
~~~~~~~~~~~~~~~~~

**定义位置:** ``include/linux/dma-mapping.h:23-62``

::

   DMA_ATTR_WEAK_ORDERING       (1UL << 1)  /* 允许读写乱序 */
   DMA_ATTR_WRITE_COMBINE       (1UL << 2)  /* 写合并优化 */
   DMA_ATTR_NO_KERNEL_MAPPING   (1UL << 4)  /* 不创建内核虚拟映射 */
   DMA_ATTR_SKIP_CPU_SYNC       (1UL << 5)  /* 跳过 CPU cache 同步 */
   DMA_ATTR_FORCE_CONTIGUOUS    (1UL << 6)  /* 强制物理连续分配 */
   DMA_ATTR_ALLOC_SINGLE_PAGES  (1UL << 7)  /* 提示无需 TLB 优化 */
   DMA_ATTR_NO_WARN             (1UL << 8)  /* 抑制分配失败警告 */
   DMA_ATTR_PRIVILEGED          (1UL << 9)  /* 特权级可访问 */

3.6 struct dma_sgt_handle
~~~~~~~~~~~~~~~~~~~~~~~~~

**定义位置:** ``include/linux/dma-map-ops.h:230-233``

非连续 DMA 分配的包装结构:

::

   struct dma_sgt_handle {
       struct sg_table sgt;          /* scatter-gather 表 */
       struct page   **pages;        /* 页面指针数组 */
   };

4. 执行阶段
------------

4.1 初始化阶段 (Boot Time)
~~~~~~~~~~~~~~~~~~~~~~~~~~

在系统启动过程中，DMA 子系统的各组件按以下顺序初始化:

::

   ┌─────────────────────────────────────────────────────────────────┐
   │                     初始化阶段调用链                            │
   ├─────────────────────────────────────────────────────────────────┤
   │                                                                 │
   │  1. early_param("cma", ...)                                     │
   │     └── 解析 cma= 内核命令行参数                                │
   │         [kernel/dma/contiguous.c:78]                            │
   │                                                                 │
   │  2. early_param("coherent_pool", ...)                           │
   │     └── 解析 coherent_pool= 内核命令行参数                      │
   │         [kernel/dma/pool.c:29]                                  │
   │                                                                 │
   │  3. dma_contiguous_reserve(limit)                               │
   │     └── arch 特定代码调用 (setup_arch 等)                       │
   │         [kernel/dma/contiguous.c:167]                           │
   │         ├── 确定 CMA 大小 (配置/百分比/命令行)                   │
   │         └── dma_contiguous_reserve_area()                       │
   │             └── cma_declare_contiguous()                        │
   │                 └── 从 memblock 保留物理内存                    │
   │                                                                 │
   │  4. RESERVEDMEM_OF_DECLARE("shared-dma-pool")                   │
   │     └── 解析设备树 reserved-memory 节点                          │
   │         [kernel/dma/contiguous.c:441 - CMA]                     │
   │         [kernel/dma/coherent.c:400 - Coherent Pool]             │
   │                                                                 │
   │  5. core_initcall: dma_init_reserved_memory()                   │
   │     └── 初始化全局一致性内存池 (DMA_GLOBAL_POOL)                │
   │         [kernel/dma/coherent.c:390]                             │
   │                                                                 │
   │  6. postcore_initcall: dma_atomic_pool_init()                   │
   │     └── 初始化原子 DMA 池 (3 个 gen_pool)                      │
   │         [kernel/dma/pool.c:187]                                 │
   │         ├── atomic_pool_kernel  (GFP_KERNEL)                   │
   │         ├── atomic_pool_dma     (GFP_DMA)                      │
   │         └── atomic_pool_dma32   (GFP_DMA32)                    │
   │                                                                 │
   │  7. 设备探测阶段                                                │
   │     ├── arch_setup_dma_ops()                                    │
   │     │   └── 设置 dev->dma_coherent, dev->dma_ops               │
   │     │   └── iommu_setup_dma_ops() → 设置 iommu_dma_ops        │
   │     ├── dma_declare_coherent_memory()                           │
   │     │   └── 为设备创建 dma_coherent_mem 池                     │
   │     └── rmem_dma_device_init() / rmem_cma_device_init()         │
   │         └── 将 DT reserved-memory 绑定到设备                   │
   │                                                                 │
   └─────────────────────────────────────────────────────────────────┘

4.2 运行时阶段 (Runtime Allocation)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``dma_alloc_coherent`` 的运行时分配路径:

::

   ┌─────────────────────────────────────────────────────────────────┐
   │                   运行时分配决策树                              │
   ├─────────────────────────────────────────────────────────────────┤
   │                                                                 │
   │  dma_alloc_coherent(dev, size, dma_handle, gfp)                │
   │    │                                                            │
   │    ├─ Step 1: 检查 dev->dma_mem (设备专属一致性池)             │
   │    │   └── 有 → bitmap_find_free_region() → 直接返回           │
   │    │                                                            │
   │    ├─ Step 2: 判断 Direct vs IOMMU 路径                        │
   │    │   ├── ops == NULL → Direct 路径                            │
   │    │   ├── dma_ops_bypass == true → Direct 路径                │
   │    │   └── 否则 → IOMMU 路径 (ops->alloc)                      │
   │    │                                                            │
   │    ├─ Step 3 [Direct]: dma_direct_alloc()                      │
   │    │   │                                                        │
   │    │   ├── [NO_KERNEL_MAPPING] → 仅分配页面,无内核映射         │
   │    │   │                                                        │
   │    │   ├── [非一致设备]                                         │
   │    │   │   ├── CONFIG_DMA_GLOBAL_POOL → 全局一致性池            │
   │    │   │   ├── 原子上下文 → dma_direct_alloc_from_pool()       │
   │    │   │   └── 可阻塞 → __dma_direct_alloc_pages() + remap     │
   │    │   │                                                        │
   │    │   ├── [一致设备]                                           │
   │    │   │   ├── 原子 + 需解密 → dma_direct_alloc_from_pool()   │
   │    │   │   └── 可阻塞 → __dma_direct_alloc_pages()             │
   │    │   │       ├── CMA 优先 (dma_alloc_contiguous)              │
   │    │   │       ├── Buddy 回退 (alloc_pages_node)               │
   │    │   │       └── ZONE 降级重试 (DMA32 → DMA)                 │
   │    │   │                                                        │
   │    │   └── Step 4: 建立内核虚拟映射 + 清零                      │
   │    │       ├── HighMem/remap → vmap() (VM_DMA_COHERENT)        │
   │    │       └── Normal → page_address() + set_memory_decrypted  │
   │    │                                                            │
   │    └─ Step 3 [IOMMU]: iommu_dma_alloc()                        │
   │        ├── 阻塞 + 非强制连续 → iommu_dma_alloc_remap()        │
   │        │   └── alloc_pages + vmap + iommu_map (IOVA)           │
   │        ├── 原子 + 非一致 → dma_alloc_from_pool()               │
   │        └── 默认 → iommu_dma_alloc_pages() + __iommu_dma_map() │
   │                                                                 │
   └─────────────────────────────────────────────────────────────────┘

4.3 清理阶段 (Free Path)
~~~~~~~~~~~~~~~~~~~~~~~~~

::

   ┌─────────────────────────────────────────────────────────────────┐
   │                      清理阶段调用链                             │
   ├─────────────────────────────────────────────────────────────────┤
   │                                                                 │
   │  dma_free_coherent(dev, size, cpu_addr, dma_handle)            │
   │    └── dma_free_attrs()  [kernel/dma/mapping.c:519]            │
   │        │                                                        │
   │        ├─ Step 1: dma_release_from_dev_coherent()              │
   │        │   └── 设备池 → bitmap_release_region() → 返回          │
   │        │                                                        │
   │        ├─ Step 2 [Direct]: dma_direct_free()                   │
   │        │   [kernel/dma/direct.c:322]                            │
   │        │   ├── [NO_KERNEL_MAPPING] → dma_free_contiguous()     │
   │        │   ├── [arch fallback] → arch_dma_free()                │
   │        │   ├── [GLOBAL_POOL + 非一致] →                        │
   │        │   │   dma_release_from_global_coherent()              │
   │        │   ├── [原子池] → dma_free_from_pool()                 │
   │        │   │   └── gen_pool_free()                             │
   │        │   ├── [vmalloc 区域] → vunmap()                       │
   │        │   ├── [uncached] → arch_dma_clear_uncached()          │
   │        │   └── __dma_direct_free_pages()                       │
   │        │       ├── swiotlb_free() → 如来自 SWIOTLB             │
   │        │       └── dma_free_contiguous()                       │
   │        │           ├── cma_release() → 如来自 CMA              │
   │        │           └── __free_pages() → 如来自 Buddy           │
   │        │                                                        │
   │        └─ Step 2 [IOMMU]: ops->free = iommu_dma_free()        │
   │            [drivers/iommu/dma-iommu.c]                          │
   │            ├── iommu_dma_free_remap() → vunmap + iommu_unmap   │
   │            └── dma_free_contiguous() → CMA 或 Buddy 释放       │
   │                                                                 │
   └─────────────────────────────────────────────────────────────────┘

5. 关键内核 API 调用
----------------------

5.1 内存分配 API
~~~~~~~~~~~~~~~~

+---------------------------+-------------------------------------------+---------------------------------------+
| API                       | 作用                                      | 所在文件                              |
+===========================+===========================================+=======================================+
| ``alloc_pages_node()``    | 从 Buddy System 分配 2^n 连续物理页        | ``mm/page_alloc.c``                   |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``cma_alloc()``           | 从 CMA 区域分配连续物理页                  | ``mm/cma.c``                          |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``gen_pool_alloc()``      | 从通用内存池 (genalloc) 原子分配            | ``lib/genalloc.c``                    |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``swiotlb_alloc()``       | 从 SWIOTLB 弹性缓冲区分配                  | ``kernel/dma/swiotlb.c``              |
+---------------------------+-------------------------------------------+---------------------------------------+

5.2 内存映射 API
~~~~~~~~~~~~~~~~

+---------------------------+-------------------------------------------+---------------------------------------+
| API                       | 作用                                      | 所在文件                              |
+===========================+===========================================+=======================================+
| ``vmap()``                | 将离散物理页映射到连续内核虚拟地址          | ``mm/vmalloc.c``                      |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``page_address()``        | 获取低内存页面的内核虚拟地址               | ``include/linux/mm.h``                |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``vunmap()``              | 取消 vmap 创建的内核虚拟映射               | ``mm/vmalloc.c``                      |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``memremap()``            | 将物理内存区域重新映射 (设备一致性池初始化) | ``mm/memremap.c``                     |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``remap_pfn_range()``     | 将物理页帧映射到用户空间 VMA               | ``mm/memory.c``                       |
+---------------------------+-------------------------------------------+---------------------------------------+

5.3 内存属性 API
~~~~~~~~~~~~~~~~

+---------------------------+-------------------------------------------+---------------------------------------+
| API                       | 作用                                      | 所在文件                              |
+===========================+===========================================+=======================================+
| ``dma_pgprot()``          | 根据设备和属性确定页面保护位               | ``kernel/dma/direct.c``               |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``set_memory_decrypted()``| 将内存标记为未加密 (AMD SEV)               | ``arch/x86/mm/pat/set_memory.c``      |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``set_memory_encrypted()``| 将内存标记为已加密 (AMD SEV)               | ``arch/x86/mm/pat/set_memory.c``      |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``arch_dma_set_uncached()``| 将内存区域设为非缓存 (架构特定)            | 架构实现                              |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``arch_dma_prep_coherent()``| 清理 cache 行以准备一致性使用             | 架构实现                              |
+---------------------------+-------------------------------------------+---------------------------------------+

5.4 Cache 同步 API (ARM64)
~~~~~~~~~~~~~~~~~~~~~~~~~~~

+---------------------------+-------------------------------------------+---------------------------------------+
| API                       | 作用                                      | 所在文件                              |
+===========================+===========================================+=======================================+
| ``dcache_clean_poc()``    | 清理 (Clean) 指定物理范围的 d-cache        | ``arch/arm64/mm/cache.S``             |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``dcache_inval_poc()``    | 使无效 (Invalidate) 指定范围的 d-cache     | ``arch/arm64/mm/cache.S``             |
+---------------------------+-------------------------------------------+---------------------------------------+

5.5 地址转换 API
~~~~~~~~~~~~~~~~

+---------------------------+-------------------------------------------+---------------------------------------+
| API                       | 作用                                      | 所在文件                              |
+===========================+===========================================+=======================================+
| ``phys_to_dma_direct()``  | 物理地址 → DMA 总线地址 (Direct 路径)      | ``kernel/dma/direct.h``               |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``page_to_phys()``        | struct page → 物理地址                    | ``include/linux/mm.h``                |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``page_to_virt()``        | struct page → 内核虚拟地址                 | ``include/linux/mm.h``                |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``pfn_to_page()``         | 页帧号 → struct page 指针                  | ``include/linux/mm.h``                |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``__phys_to_pfn()``       | 物理地址 → 页帧号                          | ``include/linux/pxa.h``               |
+---------------------------+-------------------------------------------+---------------------------------------+

5.6 位图管理 API
~~~~~~~~~~~~~~~~

+---------------------------+-------------------------------------------+---------------------------------------+
| API                       | 作用                                      | 所在文件                              |
+===========================+===========================================+=======================================+
| ``bitmap_find_free_region()`` | 在位图中查找并分配一个连续区域          | ``lib/bitmap.c``                      |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``bitmap_release_region()``   | 释放位图中的一个连续区域              | ``lib/bitmap.c``                      |
+---------------------------+-------------------------------------------+---------------------------------------+
| ``bitmap_zalloc()``       | 分配并清零位图                            | ``lib/bitmap.c``                      |
+---------------------------+-------------------------------------------+---------------------------------------+

6. 组件交互图
---------------

6.1 整体架构交互图
~~~~~~~~~~~~~~~~~~

::

   ┌─────────────────────────────────────────────────────────────────────────┐
   │                        DMA 驱动程序 (Driver)                           │
   │   dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL)              │
   └──────────────────────────────┬──────────────────────────────────────────┘
                                  │
                                  ▼
   ┌─────────────────────────────────────────────────────────────────────────┐
   │                    DMA Mapping Framework                                │
   │                    include/linux/dma-mapping.h                          │
   │  ┌───────────────────────────────────────────────────────────────────┐ │
   │  │ dma_alloc_coherent()  ───inline wrapper──→  dma_alloc_attrs()     │ │
   │  └───────────────────────────────────────────────────────────────────┘ │
   └──────────────────────────────┬──────────────────────────────────────────┘
                                  │
                                  ▼
   ┌─────────────────────────────────────────────────────────────────────────┐
   │                     Core Dispatcher                                    │
   │                     kernel/dma/mapping.c                               │
   │  ┌─────────────────────────────────────────────────────────────────┐   │
   │  │                    dma_alloc_attrs()                            │   │
   │  │                                                                  │   │
   │  │  1. get_dma_ops(dev) ──→ dma_map_ops* (或 NULL)                │   │
   │  │  2. dma_alloc_from_dev_coherent() ──→ 设备专属池?              │   │
   │  │  3. dma_alloc_direct()? ──→ YES: Direct  |  NO: IOMMU         │   │
   │  └─────────────────────────────────────────────────────────────────┘   │
   └──────────┬──────────────────────────────────────┬──────────────────────┘
              │ Direct                               │ IOMMU
              ▼                                      ▼
   ┌───────────────────────────┐    ┌───────────────────────────────────────┐
   │   Direct Mapping          │    │   IOMMU Mapping                      │
   │   kernel/dma/direct.c     │    │   drivers/iommu/dma-iommu.c          │
   │                           │    │                                       │
   │  dma_direct_alloc()       │    │  iommu_dma_alloc()                   │
   │    ├── 非一致设备处理      │    │    ├── iommu_dma_alloc_remap()       │
   │    ├── 原子池分配          │    │    ├── dma_alloc_from_pool()         │
   │    └── 页面分配 + 映射     │    │    └── iommu_dma_alloc_pages()      │
   └──────┬────┬────┬──────────┘    └───────────┬──────────────────────────┘
          │    │    │                           │
          │    │    │                           │
   ┌──────▼┐ ┌▼────▼────────────┐  ┌──────────▼──────────────────────────┐
   │ CMA   │ │ Buddy Allocator  │  │ IOMMU Driver                       │
   │       │ │                  │  │                                    │
   │ mm/   │ │ mm/page_alloc.c  │  │  __iommu_dma_map()                 │
   │ cma.c │ │                  │  │    └── iommu_map()                  │
   │       │ │ alloc_pages_node │  │        └── IOVA → 物理地址映射      │
   │cma_   │ │ __free_pages     │  │                                    │
   │alloc()│ │                  │  │  __iommu_dma_unmap()               │
   │cma_   │ └──────────────────┘  │    └── iommu_unmap()                │
   │release│                      └──────────────────────────────────────┘
   └───┬───┘
       │
       │    ┌──────────────────────────────────────────┐
       │    │ Atomic DMA Pool                          │
       │    │ kernel/dma/pool.c                        │
       │    │                                          │
       └───→│  dma_alloc_from_pool()                   │
            │    └── gen_pool_alloc()                  │
            │        [atomic_pool_dma]                 │
            │        [atomic_pool_dma32]               │
            │        [atomic_pool_kernel]              │
            │                                          │
            │  dma_free_from_pool()                    │
            │    └── gen_pool_free()                   │
            └──────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────────────────┐
   │                    Supporting Components                                │
   ├─────────────────────────────────────────────────────────────────────────┤
   │                                                                         │
   │  ┌────────────────────────┐  ┌────────────────────────────────────┐    │
   │  │ Device Coherent Pool   │  │ Kernel Remapping                   │    │
   │  │ kernel/dma/coherent.c  │  │ kernel/dma/remap.c                 │    │
   │  │                        │  │                                    │    │
   │  │ dma_coherent_mem       │  │ dma_common_contiguous_remap()     │    │
   │  │ ├─ virt_base           │  │   └── vmap() (VM_DMA_COHERENT)    │    │
   │  │ ├─ device_base         │  │                                    │    │
   │  │ ├─ bitmap              │  │ dma_common_pages_remap()          │    │
   │  │ └─ spinlock            │  │   └── vmap()                      │    │
   │  │                        │  │                                    │    │
   │  │ bitmap 分配器          │  │ dma_common_free_remap()            │    │
   │  └────────────────────────┘  │   └── vunmap()                     │    │
   │                                └────────────────────────────────────┘    │
   │  ┌────────────────────────┐  ┌────────────────────────────────────┐    │
   │  │ Architecture Hooks     │  │ SWIOTLB                            │    │
   │  │ (ARM64/x86)            │  │ kernel/dma/swiotlb.c               │    │
   │  │                        │  │                                    │    │
   │  │ arch_dma_prep_coherent │  │ 弹性缓冲区:                        │    │
   │  │ arch_dma_set_uncached  │  │ swiotlb_alloc() / swiotlb_free()  │    │
   │  │ arch_sync_dma_for_cpu  │  │ 用于:                             │    │
   │  │ arch_sync_dma_for_dev  │  │   - 32位设备在64位系统              │    │
   │  │ arch_dma_alloc         │  │   - 内存加密 (SEV)                 │    │
   │  │ arch_dma_free          │  │                                    │    │
   │  │ arch_setup_dma_ops     │  └────────────────────────────────────┘    │
   │  └────────────────────────┘                                            │
   │                                                                         │
   └─────────────────────────────────────────────────────────────────────────┘

6.2 内存流向图
~~~~~~~~~~~~~~

::

   ┌─────────────────────────────────────────────────────────────────┐
   │                     内存分配流向                                │
   │                                                                 │
   │   物理内存 (DDR)                                                │
   │   ┌─────────────────────────────────────────────────────────┐  │
   │   │  ┌──────────┐ ┌──────────────┐ ┌───────┐ ┌───────────┐  │  │
   │   │  │  CMA     │ │  Buddy       │ │Device │ │  SWIOTLB  │  │  │
   │   │  │  区域    │ │  System      │ │Pool   │ │  Buffer   │  │  │
   │   │  └────┬─────┘ └──────┬───────┘ └───┬───┘ └─────┬─────┘  │  │
   │   └───────┼──────────────┼────────────┼───────────┼────────┘  │
   │           │              │            │           │            │
   │           ▼              ▼            ▼           ▼            │
   │   ┌──────────────────────────────────────────────────────┐     │
   │   │              struct page (物理页)                     │     │
   │   └──────────────────────┬───────────────────────────────┘     │
   │                          │                                      │
   │              ┌───────────┴───────────┐                          │
   │              ▼                       ▼                           │
   │   ┌────────────────────┐  ┌─────────────────────┐              │
   │   │ CPU 虚拟地址        │  │ DMA 总线地址          │              │
   │   │ (cpu_addr)         │  │ (dma_handle)         │              │
   │   │                    │  │                      │              │
   │   │ • page_address()   │  │ • phys_to_dma()      │              │
   │   │   [直接映射区]      │  │   [Direct 路径]      │              │
   │   │                    │  │                      │              │
   │   │ • vmap()           │  │ • iommu_map()        │              │
   │   │   [vmalloc 区]      │  │   [IOVA 映射]        │              │
   │   └────────────────────┘  └─────────────────────┘              │
   │          │                          │                            │
   │          ▼                          ▼                            │
   │   ┌────────────────────┐  ┌─────────────────────┐              │
   │   │ CPU 访问            │  │ 设备 (Device) 访问    │              │
   │   │ 通过 cpu_addr      │  │ 通过 dma_handle     │              │
   │   └────────────────────┘  └─────────────────────┘              │
   │                                                                 │
   └─────────────────────────────────────────────────────────────────┘

6.3 Direct 路径 vs IOMMU 路径对比
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   ┌──────────────────────────────────────────────────────────────────┐
   │                                                                  │
   │   Direct Path (无 IOMMU 或 Bypass)                               │
   │   ─────────────────────────────────                              │
   │                                                                  │
   │   CPU ──→ [虚拟地址] ──→ [MMU] ──→ [物理地址] ──→ DDR           │
   │                                        │                         │
   │   Device ──→ [DMA地址=物理地址] ───────┘                        │
   │                                                                  │
   │   特点: DMA地址 = 物理地址 (1:1 映射)                            │
   │   限制: 设备受限于物理内存范围                                    │
   │                                                                  │
   ├──────────────────────────────────────────────────────────────────┤
   │                                                                  │
   │   IOMMU Path                                                    │
   │   ──────────────                                                │
   │                                                                  │
   │   CPU ──→ [虚拟地址] ──→ [MMU] ──→ [物理地址] ──→ DDR           │
   │                                        │                         │
   │                                   ┌────┘                         │
   │   Device ──→ [IOVA] ──→ [IOMMU] ──┘                             │
   │                     (DMA地址 → 物理地址转换)                     │
   │                                                                  │
   │   特点: IOVA 独立于物理地址空间                                  │
   │   优势: 设备可访问全部内存, 内存隔离, 地址聚合                  │
   │                                                                  │
   └──────────────────────────────────────────────────────────────────┘

7. 关键文件索引
----------------

+------------------------------------------+-----------------------------------------+
| 文件路径                                 | 功能描述                                |
+==========================================+=========================================+
| ``include/linux/dma-mapping.h``          | API 声明, DMA_ATTR_* 定义              |
| ``include/linux/dma-map-ops.h``          | struct dma_map_ops, get_dma_ops, 架构钩子 |
| ``include/linux/device.h``               | struct device (DMA 相关成员 :588-651)   |
| ``kernel/dma/mapping.c``                 | dma_alloc_attrs/free_attrs 核心调度器   |
| ``kernel/dma/direct.c``                  | Direct 映射分配器 (dma_direct_alloc)    |
| ``kernel/dma/direct.h``                  | Direct 映射内联辅助函数                  |
| ``kernel/dma/coherent.c``                | 设备一致性内存池 (dma_coherent_mem)     |
| ``kernel/dma/pool.c``                    | 原子 DMA 池 (gen_pool), pool 初始化    |
| ``kernel/dma/contiguous.c``              | CMA 分配器 (dma_alloc_contiguous)       |
| ``kernel/dma/remap.c``                   | vmap/vunmap 帮助函数                     |
| ``kernel/dma/swiotlb.c``                 | SWIOTLB 弹性缓冲区分配器               |
| ``kernel/dma/dummy.c``                   | 空 DMA 操作 (dma_dummy_ops)             |
| ``kernel/dma/ops_helpers.c``             | 通用 DMA 辅助函数                       |
| ``drivers/iommu/dma-iommu.c``            | IOMMU DMA 操作 (iommu_dma_ops)          |
| ``arch/arm64/mm/dma-mapping.c``          | ARM64 架构钩子 (cache ops)              |
| ``arch/x86/kernel/pci-dma.c``            | x86 DMA ops 设置                        |
| ``arch/x86/include/asm/dma-mapping.h``   | x86 get_arch_dma_ops()                  |
| ``mm/page_alloc.c``                      | Buddy System 页面分配器                 |
| ``mm/cma.c``                             | CMA 核心实现                            |
| ``mm/vmalloc.c``                         | vmap/vunmap 实现                        |
| ``lib/genalloc.c``                       | 通用内存池 (gen_pool)                   |
| ``lib/bitmap.c``                         | 位图操作                                |
+------------------------------------------+-----------------------------------------+
