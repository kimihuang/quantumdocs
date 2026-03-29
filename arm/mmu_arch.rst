===========================================
ARM AArch64 MMU (VMSA) 架构深度解析
===========================================

本文档基于 *DDI0487K_a — ARM Architecture Reference Manual* Chapter D8 "The AArch64 Virtual Memory System Architecture" 撰写。
所有结论均标注原文锚点（架构约束 ID 及行号），禁止编造文档中未出现的内容。

.. contents:: 目录
   :depth: 3
   :local:

-------------------------------------------------------
模块架构框图
-------------------------------------------------------

ARM AArch64 MMU 是一个基于翻译表 (Translation Table) 的地址翻译系统，
负责将指令使用的虚拟地址 (VA) 转换为物理内存系统使用的物理地址 (PA)，
同时执行内存访问权限检查和内存属性确定。 [ICCTQS, L225570; RJTYJP, L225610]

.. mermaid::
   :caption: ARM AArch64 MMU 模块架构图
   :name: fig:mmu-architecture

   graph TB
       subgraph MMU["Memory Management Unit (MMU)"]
           direction TB
           subgraph Input["地址输入"]
               VA["虚拟地址 (VA)<br/>PC / SP / LR / ELR"]
           end

           subgraph Regime["翻译体制 (Translation Regime)"]
               EL10["EL1&0 翻译体制<br/>TTBR0_EL1 / TTBR1_EL1<br/>TCR_EL1 / MAIR_EL1"]
               EL20["EL2&0 翻译体制<br/>TTBR0_EL2 / TTBR1_EL2<br/>TCR_EL2 / MAIR_EL2"]
               EL2S["EL2 翻译体制<br/>TTBR0_EL2<br/>TCR_EL2"]
               EL3R["EL3 翻译体制<br/>TTBR0_EL3<br/>TCR_EL3"]
           end

           subgraph Stage1["Stage 1 翻译"]
               S1TBL["翻译表遍历<br/>(Translation Table Walk)<br/>Level 0/1/2/3"]
               S1Desc["描述符解析<br/>Table / Block / Page"]
               S1Perm["权限检查<br/>AP / UXN / PXN / APTable"]
           end

           subgraph Stage2["Stage 2 翻译 (可选)"]
               S2TBL["Stage 2 翻译表遍历<br/>VTTBR_EL2 / VTCR_EL2"]
               S2Perm["Stage 2 权限检查<br/>S2AP / PXN / XN"]
               S2Attr["Stage 2 属性合并<br/>MemAttr / SH"]
           end

           subgraph TLB["Translation Lookaside Buffer"]
               TLBEntry["TLB 条目缓存<br/>ASID / VMID 标记"]
               TLBMaint["TLB 维护<br/>TLBI 指令"]
           end

           subgraph Output["翻译输出"]
               PA["物理地址 (PA)"]
               AttrOut["内存属性<br/>类型 / Cacheability / Shareability"]
               PermOut["最终权限<br/>Read / Write / Execute"]
           end
       end

       SEC_CTRL["安全状态控制<br/>SCR_EL3 · HCR_EL2"] --> Regime
       VA --> Regime
       Regime --> Stage1
       Stage1 --> Stage2
       Stage2 --> TLB
       TLB --> PA
       TLB --> AttrOut
       TLB --> PermOut
       TLBMaint -.-> TLBEntry

-------------------------------------------------------
模块协作关系
-------------------------------------------------------

1 翻译体制与异常级别的关系
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

翻译体制由当前安全状态 (Security state)、异常级别 (Exception level)、
已启用的异常级别和 HCR_EL2 设置共同决定。 [ICJMWY, L225668; IHLQTD, L225670]

.. mermaid::
   :caption: 翻译体制选择依赖图
   :name: fig:regime-dependency

   graph LR
       SEC["Security State<br/>Secure / Non-secure / Realm"] --> REGIME
       EL["当前 Exception Level<br/>EL0 / EL1 / EL2 / EL3"] --> REGIME
       HCR["HCR_EL2 设置<br/>E2H / TGE / NV"] --> REGIME
       SCR["SCR_EL3 设置<br/>NSE / NS / EEL2"] --> REGIME

       subgraph REGIME["选择的翻译体制"]
           NS10["Non-secure EL1&0<br/>Stage 1 + Stage 2"]
           S10["Secure EL1&0<br/>Stage 1 + Stage 2"]
           R10["Realm EL1&0<br/>Stage 1 + Stage 2"]
           NS20["Non-secure EL2&0<br/>Stage 1"]
           NS2S["Non-secure EL2<br/>Stage 1"]
           EL3R["EL3<br/>Stage 1"]
       end

2 Stage 1 与 Stage 2 翻译协作时序
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当 EL2 启用时，EL1&0 翻译体制使用两阶段翻译：
Stage 1 将 VA 翻译为 IPA，Stage 2 将 IPA 翻译为 PA。 [RKXSMJ, L225583]

.. mermaid::
   :caption: 两阶段翻译协作时序图
   :name: fig:two-stage-translation

   sequenceDiagram
       participant CPU as PE (CPU)
       participant S1 as Stage 1 翻译
       participant TT1 as Stage 1 翻译表
       participant S2 as Stage 2 翻译
       participant TT2 as Stage 2 翻译表
       participant MEM as 物理内存

       CPU->>S1: 输入 VA
       S1->>S1: 检查 TLB (TLB Miss?)
       alt TLB Hit
           S1-->>CPU: 返回 PA + 属性
       else TLB Miss
           S1->>TT2: Stage 1 翻译表基地址 (IPA) → Stage 2 翻译
           TT2-->>S2: Stage 2 翻译
           S2->>MEM: 读取 Stage 1 翻译表描述符 (PA)
           MEM-->>S1: 返回描述符
           loop 每个 Stage 1 查找级别
               S1->>S1: 解析 Table / Block / Page 描述符
               opt 需要 Stage 2 翻译表查找地址
                   S1->>S2: 下级表地址 (IPA)
                   S2->>TT2: Stage 2 翻译
                   TT2-->>S2: 返回 PA
                   S2->>MEM: 读取描述符 (PA)
                   MEM-->>S1: 返回描述符
               end
           end
           S1->>S2: Stage 1 最终 OA (IPA)
           S2->>MEM: Stage 2 最终翻译
           MEM-->>S2: 返回 Stage 2 描述符
           S2-->>CPU: 返回 PA + 合并后的属性
           S2->>S2: 缓存到 TLB
       end

   Note over CPU,MEM: 最大查找次数 = (S1+1)*(S2+1)-1 [IJQDTN, L226393]

-------------------------------------------------------
关键流程
-------------------------------------------------------

1 核心流程一：地址翻译 (Translation Table Walk)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

地址翻译从读取翻译表基地址寄存器开始，通过逐级遍历翻译表完成。 [IGDSWV, L226276]

.. mermaid::
   :caption: 地址翻译完整流程图
   :name: fig:translation-process

   flowchart TD
       START([开始: VA 输入]) --> SEL_REGIME{选择翻译体制}
       SEL_REGIME --> SEL_TTBR{VA bit[55]=?<br/>[RVZCSR, L225518]}
       SEL_TTBR -->|0| TTBR0["选择 TTBR0_ELx<br/>(低地址范围)"]
       SEL_TTBR -->|1| TTBR1["选择 TTBR1_ELx<br/>(高地址范围)"]
       TTBR0 --> CHECK_EPD{EPDn=1?<br/>[IZFSYQ, L225520]}
       TTBR1 --> CHECK_EPD
       CHECK_EPD -->|是| FAULT_L0["Level 0 Translation fault"]
       CHECK_EPD -->|否| TLB_CHECK{TLB 命中?}
       TLB_CHECK -->|是| RETURN_TLB["返回缓存的<br/>PA + 属性"]
       TLB_CHECK -->|否| INIT["从 TTBR_ELx 获取<br/>初始查找级别<br/>[RGWNBR, L226936]"]
       INIT --> WALK["读取翻译表描述符<br/>(8B/16B 原子访问)<br/>[RYKQTS, L226288]"]
       WALK --> EXT_ABORT{External Abort?<br/>[RYLGBV, L226467]}
       EXT_ABORT -->|是| REPORT_EXT["报告同步/异步<br/>External abort"]
       EXT_ABORT -->|否| CHECK_DESC{描述符有效?<br/>bit[0]=1<br/>[RYPWQG, L226294]}
       CHECK_DESC -->|否| TRANS_FAULT["Translation fault<br/>[IWMLYG, L226304]"]
       CHECK_DESC -->|是| DESC_TYPE{描述符类型}
       DESC_TYPE -->|"Table (level<3)"| NEXT["获取下级表地址<br/>继续遍历"]
       DESC_TYPE -->|"Block"| FINAL["最终描述符"]
       DESC_TYPE -->|"Page (level=3)"| FINAL
       NEXT --> ADDR_SIZE{地址大小正确?}
       ADDR_SIZE -->|否| AS_FAULT["Address size fault<br/>[RVPBBF, L226460]"]
       ADDR_SIZE -->|是| WALK
       FINAL --> CHECK_AF{AF=0?<br/>[RXFXTY, L225627]}
       CHECK_AF -->|是, 无HW管理| AF_FAULT["Access flag fault<br/>[IBMSDD, L232486]"]
       CHECK_AF -->|否, 或HW更新| CHECK_PERM{权限检查<br/>[RMNKWG, L225607]}
       CHECK_PERM -->|失败| PERM_FAULT["Permission fault<br/>[RJVXRH, L232502]"]
       CHECK_PERM -->|通过| COMPUTE_OA["计算 OA<br/>(描述符OA位 + IA低位直接映射)"]
       COMPUTE_OA --> MERGE_ATTR["合并内存属性<br/>Stage 1 + Stage 2"]
       MERGE_ATTR --> CACHE_TLB["缓存到 TLB"]
       CACHE_TLB --> DONE([返回 PA + 属性 + 权限])

2 核心流程二：Break-before-Make 翻译表更新
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当修改翻译表条目的内存类型、OA、块大小时，
必须使用 break-before-make (BBM) 序列以确保一致性。 [RWHZWS, L226179; RDDMVT, L226208]

.. mermaid::
   :caption: Break-before-Make 翻译表更新流程图
   :name: fig:bbm-sequence

   flowchart TD
       START([开始: 需修改翻译表条目]) --> REQUIRE{是否需要 BBM?<br/>[RWHZWS, L226179]}
       REQUIRE -->|"类型/OA/大小变更"| INVALIDATE["步骤1: 用无效描述符<br/>替换旧条目<br/>bit[0]=0"]
       REQUIRE -->|"仅权限变更"| DIRECT["直接更新描述符<br/>无需 BBM"]
       DIRECT --> MAINT_PERM["步骤1: 更新描述符"]
       MAINT_PERM --> DSB1_P["步骤2: DSB"]
       DSB1_P --> TLBI_P["步骤3: 执行 TLB 无效化<br/>[TLBI 指令]"]
       TLBI_P --> DSB2_P["步骤4: DSB"]
       DSB2_P --> DONE_P([完成])

       INVALIDATE --> DSB1["步骤2: DSB<br/>确保无效条目可见"]
       DSB1 --> TLBI["步骤3: 广播 TLB 无效化<br/>[RDDMVT, L226211]"]
       TLBI --> DSB2["步骤4: DSB<br/>确保无效化完成"]
       DSB2 --> NEW["步骤5: 写入新翻译表条目"]
       NEW --> DSB3["步骤6: DSB<br/>确保新条目可见"]
       DSB3 --> DONE([完成])

       style REQUIRE fill:#f9f,stroke:#333
       style INVALIDATE fill:#ff9,stroke:#333

3 核心流程三：MMU Fault 检查序列
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

IA 翻译到 OA 过程中按固定优先级执行 15 步检查。 [IPTYRT, L232646]

.. mermaid::
   :caption: MMU Fault 检查序列
   :name: fig:fault-checking-sequence

   flowchart TD
       S1["1. IA 对齐检查"] --> S2["2. IA 是否映射到 TTBR"]
       S2 --> S3["3. TTBR 地址大小检查"]
       S3 --> S4["4. 获取描述符"]
       S4 --> S5["5. 描述符有效性检查<br/>bit[0]=1?"]
       S5 --> S6["6. 描述符地址大小检查"]
       S6 --> S7{描述符类型?}
       S7 -->|Table| S4
       S7 -->|Block/Page| S8["8. FEAT_BBM nT 检查"]
       S8 --> S9["9. AF bit 检查"]
       S9 --> S10["10. AF 硬件管理更新"]
       S10 --> S11["11. 计算 OA"]
       S11 --> S12["12. OA 对齐检查"]
       S12 --> S13["13. OA 空间权限检查"]
       S13 --> S14["14. Dirty state 硬件管理"]
       S14 --> S15["15. 返回 OA 和属性"]

       S1 -.->|"失败"| ALIGN_FAULT["Alignment fault"]
       S2 -.->|"失败"| TRANS_FAULT["Translation fault"]
       S3 -.->|"失败"| ADDR_FAULT["Address size fault"]
       S4 -.->|"失败"| EXT_FAULT["Synchronous External abort"]
       S5 -.->|"失败"| TRANS_FAULT
       S6 -.->|"失败"| ADDR_FAULT
       S9 -.->|"失败"| AF_FAULT["Access flag fault"]
       S12 -.->|"失败"| ALIGN_FAULT
       S13 -.->|"失败"| PERM_FAULT["Permission fault"]

-------------------------------------------------------
数据/控制流关键路径
-------------------------------------------------------

1 地址翻译数据流
~~~~~~~~~~~~~~~~~~

.. mermaid::
   :caption: 地址翻译数据流路径（以 4KB Granule、48 位地址为例）
   :name: fig:address-data-flow

   graph LR
       subgraph VA["输入地址 (VA, 48-bit)"]
           VA47_39["VA[47:39]<br/>Level 0 索引 (9 bits)"]
           VA38_30["VA[38:30]<br/>Level 1 索引 (9 bits)"]
           VA29_21["VA[29:21]<br/>Level 2 索引 (9 bits)"]
           VA20_12["VA[20:12]<br/>Level 3 索引 (9 bits)"]
           VA11_0["VA[11:0]<br/>页内偏移 (12 bits)"]
       end

       subgraph PA["输出地址 (PA, 48-bit)"]
           OA47_39["OA[47:39]<br/>来自 Block/Page"]
           OA38_30["OA[38:30]<br/>来自 Block/Page"]
           OA29_21["OA[29:21]<br/>来自 Block/Page"]
           OA20_12["OA[20:12]<br/>来自 Page"]
           OA11_0["OA[11:0]<br/>直接映射"]
       end

       VA47_39 -->|"→ Table/Block"| OA47_39
       VA38_30 -->|"→ Table/Block"| OA38_30
       VA29_21 -->|"→ Table/Block"| OA29_21
       VA20_12 -->|"→ Page"| OA20_12
       VA11_0 -->|"直接映射"| OA11_0

       subgraph Control["控制路径"]
           TCR["TCR_ELx.T0SZ<br/>确定起始级别"]
           TTBR["TTBR_ELx.BADDR<br/>初始表基地址"]
           EPD["TCR_ELx.EPDn<br/>禁用控制"]
           DS["TCR_ELx.DS<br/>48/52 位选择"]
       end

       TCR -.->|"配置"| VA47_39
       TTBR -.->|"提供基地址"| VA47_39
       EPD -.->|"控制使能"| Control
       DS -.->|"控制地址位数"| Control

   Note right of VA47_39: [ISZLLC, L226810]
   Note right of PA: [RYLGLV, L226851]

2 权限检查控制流
~~~~~~~~~~~~~~~~~~

.. mermaid::
   :caption: Stage 1 权限检查控制流
   :name: fig:permission-control-flow

   flowchart TD
       ACCESS["内存访问请求"] --> DIRECT{"Direct or<br/>Indirect?"}
       DIRECT -->|Direct| AP_CHECK["AP[2:1] 字段检查<br/>[RCBNKR, L226660]"]
       DIRECT -->|Indirect| PI_CHECK["PIIndex → PIR_ELx<br/>[RJJWXH, L226821]"]

       AP_CHECK --> AP_RESULT["确定 PrivRead / PrivWrite<br/>UnprivRead / UnprivWrite"]
       PI_CHECK --> PI_RESULT["确定 S1PrivBasePerm<br/>S1UnprivBasePerm"]

       AP_RESULT --> OVERLAY{"Overlay 启用?"}
       PI_RESULT --> OVERLAY
       OVERLAY -->|是| POR_CHECK["POIndex → POR_ELx<br/>进一步限制权限"]
       OVERLAY -->|否| BASE["Base 权限"]
       POR_CHECK --> FINAL_BASE["最终权限"]

       FINAL_BASE --> PAN{"PSTATE.PAN=1?<br/>特权数据访问"}
       BASE --> PAN
       PAN -->|是| PAN_BLOCK["阻止特权写"]
       PAN -->|否| WXN{"SCTLR.WXN=1?"}
       PAN_BLOCK --> WXN
       WXN -->|是, 可写+可执行| WXN_APPLY["WXN: 移除执行权限<br/>[RFYMXJ, L226720]"]
       WXN -->|否| DONE["最终访问权限确定"]

   Note right of AP_CHECK: APTable 可逐级限制后续权限 [RYTXKB, L226688]
   Note right of WXN: 防止可写内存同时可执行 [IXLDPQ, L226742]

3 内存属性合并数据流
~~~~~~~~~~~~~~~~~~~~~~~~

.. mermaid::
   :caption: Stage 1 + Stage 2 内存属性合并路径
   :name: fig:attribute-merge-flow

   graph TB
       subgraph Stage1_Attr["Stage 1 属性"]
           S1_MAIR["MAIR_ELx.Attr<n><br/>由 AttrIndx 索引<br/>[RHHGNL, L231039]"]
           S1_Type["Normal / Device"]
           S1_Cache["Inner Cacheability<br/>Outer Cacheability"]
           S1_SH["SH[1:0]<br/>Shareability"]
       end

       subgraph Stage2_Attr["Stage 2 属性"]
           S2_MemAttr["MemAttr[3:0]<br/>[RMFDHF, L231144]"]
           S2_Type["Normal / Device"]
           S2_Cache["Inner / Outer Cacheability"]
           S2_SH["SH[1:0]"]
       end

       subgraph Merge["属性合并"]
           MERGE_TYPE["类型合并<br/>Device 优先"]
           MERGE_CACHE["Cacheability 合并<br/>Stage2 可降低不可提升<br/>[RGQFSF, L231257]"]
           MERGE_SH["Shareability 合并<br/>Stage2 可提升不可降低"]
       end

       subgraph Override["属性覆盖"]
           SCTLR_CI["SCTLR_ELx.{C, I}<br/>可覆盖 stage 1 属性"]
           HCR_DC["HCR_EL2.DC<br/>可覆盖 stage 1+2"]
           HCR_PTW["HCR_EL2.PTW<br/>保护 stage 1 表遍历"]
       end

       S1_MAIR --> S1_Type
       S1_MAIR --> S1_Cache
       S1_MAIR --> S1_SH
       S2_MemAttr --> S2_Type
       S2_MemAttr --> S2_Cache
       S2_MemAttr --> S2_SH

       S1_Type --> MERGE_TYPE
       S2_Type --> MERGE_TYPE
       S1_Cache --> MERGE_CACHE
       S2_Cache --> MERGE_CACHE
       S1_SH --> MERGE_SH
       S2_SH --> MERGE_SH

       MERGE_TYPE --> RESULT["最终内存类型"]
       MERGE_CACHE --> RESULT
       MERGE_SH --> RESULT
       RESULT --> Override

-------------------------------------------------------
重要知识点深入讲解
-------------------------------------------------------

1 翻译粒度 (Translation Granule) 的选择与影响
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**核心概念：** 翻译粒度大小决定了每次查找级别解析的地址位数和
翻译表的最大大小。 [IGZDPY, L225630; RZVQXW, L225632]

VMSA 支持三种翻译粒度：4KB、16KB、64KB。每种粒度对查找级别数、
页表大小、页大小有直接影响。 [RZVQXW, L225634]

.. list-table:: 三种翻译粒度的关键参数对比
   :widths: 20 20 20 20 20
   :header-rows: 1

   * - 粒度
     - 每级索引位数
     - 每级表项数
     - 页大小 (Level 3)
     - 最大 Block 大小
   * - 4KB
     - 9
     - 512
     - 4KB
     - 1GB (L1) / 512GB (L0, DS=1)
   * - 16KB
     - 11
     - 2048
     - 16KB
     - 32MB (L2) / 64GB (L1, DS=1)
   * - 64KB
     - 13
     - 8192
     - 64KB
     - 512MB (L2) / 4TB (L1, LPA)

Note: [RVMRQS, L226822; ITDTCY, L227357; IVDPWL, L227806]

**设计权衡：**

较大的粒度使用更大的翻译表（更多表项），单次查找可解析更多 IA 位，
从而减少查找级别数。但这意味着更大的页大小，页面管理粒度更粗。
[IKZLYC, L225649]

**关键约束：** Arm 建议将内存映射外设按最大粒度大小的整数倍分隔，
以便独立管理每个外设。 [IQRLDB, L225656]

2 两阶段翻译中的权限与属性合并机制
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Stage 1 权限系统：** 支持 Direct permissions 和 Indirect permissions 两种机制。 [IPJNLQ, L225646]

- **Direct permissions：** 使用描述符中的 AP[2:1]、UXN、PXN 字段直接定义权限。
  支持 hierarchical permissions（APTable、XNTable、PXNTable）逐级限制后续级别的权限。
  [RCBNKR, L225660; RYTXKB, L225688]

- **Indirect permissions (FEAT_S1PIE)：** 使用描述符中的 PIIndex 索引 PIR_ELx 寄存器
  获取 Base permissions。VMSAv9-128 仅支持 Indirect permissions。
  [RJJWXH, L225821; IBSXJT, L225655]

**WXN 机制：** SCTLR_ELx.WXN=1 时，若内存区域可写则强制不可执行。
这引入了 "可写内存不可执行" 的不变量，缩减代码注入攻击的内存面。
[IKDRYB, L225718; IXLDPQ, L225742]

**Cacheability 合并规则（关键结论）：**

.. list-table:: Stage 1 + Stage 2 Cacheability 合并结果 [RGQFSF, L231257]
   :widths: 30 30 30
   :header-rows: 1

   * - Stage 1 Cacheability
     - Stage 2 Cacheability
     - 合并结果
   * - Non-cacheable
     - Any
     - Non-cacheable
   * - Write-Through
     - Non-cacheable
     - Non-cacheable
   * - Write-Through
     - Write-Through / Write-Back
     - Write-Through
   * - Write-Back
     - Non-cacheable / Write-Through
     - Non-cacheable / Write-Through
   * - Write-Back
     - Write-Back
     - Write-Back

**关键结论：Stage 2 可以降低但不能提升 Stage 1 的 Cacheability。**
这允许 Hypervisor 限制 Guest OS 的缓存行为。 [RGQFSF, L231257]

**Shareability 合并规则（关键结论）：**

.. list-table:: Stage 1 + Stage 2 Shareability 合并结果 [ILTFXF, L231360]
   :widths: 30 30 30
   :header-rows: 1

   * - Stage 1 Shareability
     - Stage 2 Shareability
     - 合并结果
   * - Outer Shareable
     - Any
     - Outer Shareable
   * - Inner Shareable
     - Outer Shareable
     - Outer Shareable
   * - Inner Shareable
     - Inner Shareable / Non-shareable
     - Inner Shareable
   * - Non-shareable
     - Outer Shareable / Inner Shareable
     - Outer Shareable / Inner Shareable
   * - Non-shareable
     - Non-shareable
     - Non-shareable

**关键结论：Stage 2 可以提升但不能降低 Stage 1 的 Shareability。**
方向与 Cacheability 合并恰好相反。 [ILTFXF, L231360]

3 TLB 维护的正确性要求
~~~~~~~~~~~~~~~~~~~~~~~~~~

**核心原则：** TLB 与翻译表的更改不自动保持一致，
软件必须显式执行 TLB 维护操作。 [RFFWJK, L233133]

**Fault 条目不缓存规则：** 以下类型的描述符从不缓存到 TLB，
因此当它们被改为有效条目时，不需要 TLB 维护（但需要 Context 同步事件）： [RXCLRD, L225967]

- Translation fault 描述符
- Address size fault 描述符
- Access flag fault 描述符

**Permission fault 条目可缓存（易错点）：**
Permission fault 的描述符 **允许被缓存到 TLB**。
若软件因 Permission fault 更新翻译表以授予权限，
**必须执行 TLB 无效化** 以防止后续访问使用过期信息。 [RJVXRH, L232502; IWJLXV, L232503]

**ASID 与 VMID 机制：** ASID 允许软件在进程切换时避免无效化整个 TLB。
Non-global TLB 条目与特定 ASID 关联；VMID 与特定虚拟机关联。
[ILHWHR, L233028; RQGKGF, L233043]

**TLB 维护指令排序（关键）：**
TLB 维护指令与其他 load/store/cache 维护指令之间无排序保证，
必须使用 DSB 指令来确保排序。 [RTGVWD, L233868]

.. list-table:: TLB 维护的完整同步序列
   :widths: 60 40
   :header-rows: 1

   * - 步骤
     - 目的
   * - 1. 修改翻译表条目
     - 更新映射信息
   * - 2. DSB
     - 确保写操作对翻译表遍历可见
   * - 3. TLBI 指令
     - 无效化受影响的 TLB 条目
   * - 4. DSB ISH
     - 确保 TLB 无效化在 Inner Shareable 域内完成
   * - 5. ISB
     - Context 同步，确保后续指令使用新翻译

Note: [RPSMWS, L233915; RBLDZX, L233924]

4 VMSAv8-64 与 VMSAv9-128 翻译系统的关键差异
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. list-table:: 两种翻译系统对比
   :widths: 25 35 35
   :header-rows: 1

   * - 特性
     - VMSAv8-64
     - VMSAv9-128
   * - 描述符大小
     - 8 字节 (64-bit) [RYKQTS, L226288]
     - 16 字节 (128-bit) [RYKQTS, L226288]
   * - Skip Level (SKL)
     - 不支持
     - 描述符和基地址寄存器中的 SKL 字段允许跳过查找级别 [IHFBGP, L226321]
   * - 权限模型
     - 支持 Direct 和 Indirect permissions [IVYHZC, L225649]
     - 仅支持 Indirect permissions [IBSXJT, L225655]
   * - AIE
     - 可选
     - 始终启用 [IVYHZC, L225649]
   * - 最大 VA 宽度（双范围）
     - 48/52 位
     - 55 位 [ISMKBQ, L226520]
   * - Hierarchical permissions
     - 支持 (HPDn 控制)
     - 不支持 (HPDn 为 RES1) [IQPHCD, L226306]
   * - PBHA
     - 支持 (FEAT_HPDS2)
     - 不支持 [IRHDNL, L231506]

**设计意图：** VMSAv9-128 通过更大的描述符和 SKL 机制提供更灵活的翻译配置，
支持更大的 VA 空间和更少的查找级别，同时统一使用 Indirect permissions 简化权限管理。

5 安全状态对翻译表查找的影响
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Non-secure 体制：** 所有翻译表查找在 Non-secure IPA/PA 空间。 [RPXRPP, L225644]

**Secure 体制 Stage 1：** 初始查找在 Secure OA 空间。后续由 Table descriptor 的
NSTable bit 控制：NSTable=0 继续在 Secure，NSTable=1 切换到 Non-secure。
一旦切换到 Non-secure，后续所有查找都在 Non-secure（不可逆转）。
[RSCHHQ, L225648]

**Realm 体制：** 所有翻译表查找在 Realm PA 空间。 [RDVGRP, L225666]

**EL3 体制 (FEAT_RME)：** 所有翻译表查找在 Root PA 空间。
[RCFPDJ, L225669]

6 硬件管理的 Access Flag 和 Dirty State
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Access Flag (AF) 硬件管理 (FEAT_HAFDBS)：**

当 TCR_ELx.HA=1 时启用。硬件在首次访问使用描述符时，通过原子读-修改-写
将 AF 从 0 设置为 1。这消除了软件管理的 Access flag fault 处理开销。
[RHDHQG, L225648; RLFTXR, L225646]

**Dirty State 硬件管理 (FEAT_HAFDBS)：**

当 TCR_ELx.HA=1 且 TCR_ELx.HD=1 时启用。描述符存在三种状态：
[writable-clean → writable-dirty]
硬件在首次写访问 writable-clean 描述符时，原子地将其更新为 writable-dirty。
[IHTJWZ, L225733; RPZFQC, L225772]

.. list-table:: Dirty State 描述符状态（Direct permissions）
   :widths: 30 30 30
   :header-rows: 1

   * - 状态
     - 条件
     - 写访问行为
   * - Non-writable
     - AP[2]=1 或 DBM=0
     - Permission fault
   * - Writable-clean
     - DBM=1 且 AP[2]=1
     - 硬件更新为 writable-dirty（不 fault）
   * - Writable-dirty
     - DBM=1 且 AP[2]=0
     - 正常写访问

Note: [RBRFGY, L225782]

**易错点：** 当同时使用 AF 和 dirty state 硬件管理时，
修改翻译表条目必须严格遵循 break-before-make 序列，
否则可能导致 AF 不被设置或 dirty state 位错误更新。 [IGXGZY, L226218; RRTBRL, L226224]

7 Address Tagging 与 Pointer Authentication
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Address Tagging (TBI)：** 当 TCR_ELx.TBI{n}=1 时，VA 的 bits[63:56]
被作为 address tag 在翻译中被忽略。这允许软件使用高 8 位存储元数据。
[IXWNGB, L225571; ICKLSG, L225583]

**Pointer Authentication (PAuth)：** 对 64-bit 通用寄存器中的地址签名（PAC），
并在用作间接分支目标或 load/store 基址前验证。PAC 位于指针高位部分，
精确位置由 TCR_ELx.TnSZ 决定。TnSZ 越小，PAC 字段越小，保护效果越弱。
[IXDNDT, L225686; D8.10.1, L225692]

**关键交互：** 若同时使用 TBI 和 PAuth，address tag（bits[63:56]）
可能限制 PAC 的可用空间。TBID{n}=1 可将 tagging 限制为仅数据地址，
为指令地址的 PAC 腾出空间。 [IHLQVM, L225754]

8 虚拟化对翻译的影响 (VHE / NV)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Virtualization Host Extensions (VHE)：**

HCR_EL2.E2H=1 时，Host OS 在 EL2 运行，使用 EL2&0 翻译体制，
其行为与 EL1&0 Stage 1 完全相同，包括支持 TTBR1_EL2 翻译高地址范围。
[IQJTJN, L225971; RMNWSS, L232004]

**Nested Virtualization (FEAT_NV)：**

HCR_EL2.{NV, NV1} 控制嵌套虚拟化。NV=1 时，EL1 访问 *_EL2 寄存器
被 trap 到 EL2。NV1=1 时，描述符语义发生变化（bit[54] 变为 PXN 而非 UXN），
实现更深层的嵌套虚拟化。FEAT_NV2 可将系统寄存器访问转换为内存访问。
[IFDMLZ, L232150; IZMLPV, L232160]

-------------------------------------------------------
关键寄存器速查表
-------------------------------------------------------

.. list-table:: MMU 关键寄存器
   :widths: 20 25 15 35
   :header-rows: 1

   * - 寄存器
     - 用途
     - 原文锚点
     - 关键字段
   * - SCTLR_ELx
     - 系统控制寄存器
     - [L225922]
     - M=MMU使能, EE=字节序, WXN=写不可执行
   * - TCR_ELx
     - 翻译控制寄存器
     - [L225922]
     - T0SZ/T1SZ=IA大小, TG0/TG1=粒度, EPDn=禁用, HPDn=分层权限
   * - TCR2_ELx
     - 翻译控制扩展寄存器
     - [L226679]
     - PIE=Indirect权限, PnCH=Protected, AIE=AIE使能
   * - TTBR0_ELx
     - 低地址范围翻译表基地址
     - [ITLWRN, L226427]
     - BADDR=基地址, ASID=地址空间ID, SKL=Skip Level
   * - TTBR1_ELx
     - 高地址范围翻译表基地址
     - [RDXLKR, L226429]
     - BADDR=基地址, ASID=地址空间ID
   * - MAIR_ELx
     - 内存属性间接寄存器
     - [RHHGNL, L231039]
     - Attr<n>=8-bit属性编码, 由AttrIndx索引
   * - VTCR_EL2
     - Stage 2 翻译控制
     - [IGVFLG, L227048]
     - SL0/SL2=起始级别, PS=PA大小, DS=52位使能
   * - VTTBR_EL2
     - Stage 2 翻译表基地址
     - [ITDMHR, L226400]
     - BADDR=基地址, VMID=虚拟机ID
   * - HCR_EL2
     - Hypervisor控制寄存器
     - [L225694]
     - E2H=VHE, TGE=Host直接执行, PTW=保护表遍历, VM=虚拟化
   * - FAR_ELx
     - Fault Address 寄存器
     - [IDHWPX, L232389]
     - 记录产生fault的地址
   * - ESR_ELx
     - Exception Syndrome 寄存器
     - [IDHWPX, L232390]
     - DFSC/IFSC=Fault Status Code
   * - PAR_EL1
     - 地址翻译结果寄存器
     - [RNHWXL, L232559]
     - F=1表示fault, 含fault syndrome
   * - PIR_ELx
     - Permission Indirection 寄存器
     - [IBSMJB, L225832]
     - 由PIIndex索引获取Indirect权限
   * - POR_ELx
     - Permission Overlay 寄存器
     - [RSHZSG, L225612]
     - 由POIndex索引进一步限制权限

-------------------------------------------------------
附录：章节与原文位置映射
-------------------------------------------------------

.. list-table:: Chapter D8 章节结构
   :widths: 50 15 30
   :header-rows: 1

   * - 章节标题
     - 行号
     - 内容概要
   * - D8.1 Address translation
     - L225560
     - 地址翻译概述、翻译粒度、翻译体制
   * - D8.2 Translation process
     - L226274
     - 翻译表遍历、地址拼接、两级VA范围
   * - D8.3 Translation table descriptor formats
     - L228460
     - Table/Block/Page描述符格式
   * - D8.4 Memory access control
     - L229605
     - Direct/Indirect/Overlay权限
   * - D8.5 Hardware updates to the translation tables
     - L230618
     - AF和dirty state硬件管理
   * - D8.6 Memory region attributes
     - L230999
     - 内存类型、Cacheability、Shareability
   * - D8.7 Other descriptor fields
     - L231380
     - Contiguous、PBHA、nT、XS
   * - D8.8 Address tagging
     - L231569
     - TBI机制
   * - D8.10 Pointer authentication
     - L231684
     - PAC签名与认证
   * - D8.11 Memory Encryption Contexts
     - L231888
     - MEC加密上下文
   * - D8.12 Virtualization Host Extensions
     - L231969
     - VHE对翻译的影响
   * - D8.13 Nested virtualization
     - L232148
     - 嵌套虚拟化
   * - D8.14 Memory aborts
     - L232367
     - MMU fault类型、检查序列、优先级
   * - D8.15 Translation Lookaside Buffers
     - L232963
     - TLB结构、ASID/VMID
   * - D8.16 TLB maintenance
     - L233122
     - TLBI指令、BBM序列
   * - D8.17 Caches
     - L233938
     - 数据/指令cache与翻译一致性
   * - D8.18 Pseudocode
     - L234005
     - VMSAv8-64翻译伪代码
