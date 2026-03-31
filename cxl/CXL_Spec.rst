Compute Express Link (CXL) 规范分析
====================================

本文档基于 CXL 3.0 规范（Revision 3.0, Version 0.7）进行深入分析，
全面梳理 CXL 协议的架构、事务层、链路层、交换机制及管理规范。
参考文档：docs/pdf/CXL/CXL_Spec_3.0_v0.7.pdf.md

.. contents::
   :depth: 3
   :local:


CXL 概述
--------

CXL（Compute Express Link）是一种高性能 I/O 总线架构，用于互联外围设备。
这些设备可以是传统的非一致性 I/O 设备、内存设备，或具有额外能力的加速器。
CXL 构建在 PCIe 物理层之上，提供三种协议：

- **CXL.io**：基于 PCIe 的非一致性 I/O 协议，用于设备发现、枚举、配置和管理
- **CXL.cache**：一致性缓存协议，允许设备缓存主机内存
- **CXL.mem**：内存协议，允许主机访问设备附加内存

CXL 的核心优势在于支持 **主机管理设备内存（HDM）**，使设备附加内存能够以一致性方式映射到系统地址空间，
从而避免传统 PCIe 设备中大量数据拷贝的开销。

参考来源：第 1 章及第 2 章


CXL 设备类型
~~~~~~~~~~~~

CXL 规范定义了三种设备类型，每种类型支持不同的协议组合和应用场景。

Type 1 设备：带缓存的设备
^^^^^^^^^^^^^^^^^^^^^^^^^^

Type 1 设备具有完全一致的缓存，但不附加设备内存。这类设备需要缓存一致性来执行复杂的原子操作，
而标准 PCIe 原子操作集无法满足这些需求。

**核心特征**：

- 支持 CXL.io 和 CXL.cache 协议
- 设备缓存由主机监听过滤器跟踪
- 缓存大小受主机监听过滤能力限制
- 典型应用：需要特殊原子操作的网络加速器

**缓存一致性机制**：Type 1 设备通过 CXL.cache 协议与主机一致性桥交互，
主机负责维护缓存一致性。当设备需要访问主机内存时，通过 D2H（Device to Host）请求获取数据，
主机通过 H2D（Host to Device）响应返回数据和缓存状态（MESI 状态）。

参考来源：第 2.1 节

Type 2 设备：带内存的加速器
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Type 2 设备同时具有缓存和设备附加内存（如 DDR、HBM）。这类设备在高性能计算场景中，
需要设备与设备附加内存之间的大带宽访问。

**核心特征**：

- 支持 CXL.io、CXL.cache 和 CXL.mem 三种协议
- 设备附加内存可映射到系统一致性地址空间（HDM）
- 支持偏置模式（Bias Mode）管理一致性

**偏置一致性模型**：

偏置模型定义了设备附加内存的两种状态：

1. **主机偏置（Host Bias）**：主机可直接高吞吐访问设备内存，设备访问需要通过主机
2. **设备偏置（Device Bias）**：设备可直接访问本地内存，主机访问需要设备配合

偏置模型的核心优势：

- 维护设备附加内存的一致性
- 允许设备以高带宽访问本地内存，避免一致性开销
- 主机可以一致性方式访问设备内存

**偏置模式管理方式**：

- 软件辅助管理：软件根据工作负载阶段主动切换偏置模式
- 硬件自主管理：硬件根据访问模式自动预测和调整偏置状态

参考来源：第 2.2 节

Type 3 设备：内存扩展器
^^^^^^^^^^^^^^^^^^^^^^^

Type 3 设备是内存扩展器，仅支持 CXL.io 和 CXL.mem 协议。此类设备不具有设备缓存，
主机将其视为分离式内存控制器。

**核心特征**：

- 无设备缓存，无需 CXL.cache 协议
- 主机通过 CXL.mem 读写设备内存
- 架构独立于内存技术（易失性/持久性）
- 支持多种内存组织方式

**应用场景**：

- 内存容量扩展
- 内存池化（通过 MLD 机制）
- 共享 Fabric 附加内存（FAM）

参考来源：第 2.3 节


多逻辑设备（MLD）
~~~~~~~~~~~~~~~~~

CXL 2.0 引入 MLD 机制，允许 Type 3 设备将资源划分为最多 16 个隔离的逻辑设备。
每个逻辑设备由 LD-ID 标识，对虚拟层次结构（VH）可见的每个 LD 作为独立的 Type 3 设备运行。

**MLD 组成**：

- 一个 LD 保留给 Fabric Manager（FM），用于配置资源分配
- 最多 16 个 LD 可用于主机绑定
- 所有 LD 共享协议的事务层和链路层

**LD-ID 传输机制**：

- CXL.mem：支持 LD-ID 低 4 位，可支持最多 16 个 LD-ID
- CXL.io：使用 Vendor Defined Local TLP Prefix 传输 16 位 LD-ID
- LD-ID FFFFh 保留给 FM 使用

**池化内存与共享 FAM**：

- CXL 2.0 池化内存：每个 LD 的 HDM 区域私有，不跨主机共享
- CXL 3.0 共享 FAM：允许同一 HDM 区域分配给多个 LD，实现跨主机共享

**共享 FAM 一致性模型**：

- 多主机硬件一致性：设备跟踪 MESI 状态，支持任意原子操作
- 软件管理一致性：不跟踪 MESI 状态，原子操作不支持

参考来源：第 2.4 节


CXL Fabric 与 GFAM
~~~~~~~~~~~~~~~~~~~

**域（Domain）** 定义为单个主机物理地址（HPA）空间内的一组主机端口和设备。
CXL Switch 连接一个或多个主机端口与域内设备。

**CXL Fabric** 是互联的 CXL Switch 集合，支持跨域通信：

- 使用 Fabric 地址空间进行跨域访问
- 大多数跨域通信为非一致性
- 12 位目标端口 ID（DPID）和源端口 ID（SPID）支持大型 Fabric

**全局 Fabric 附加内存（GFAM）**：

GFAM 是一种 SLD Type 3 设备，支持 CXL.io 和 CXL.mem 协议：

- I/O 空间由单一主机或 Fabric Manager 拥有
- 设备配置后，Fabric 上其他主机/设备可直接访问
- 仅使用低位地址访问内存，高位地址被忽略
- 支持 PBR 消息格式（包含完整 12 位 SPID/DPID）

参考来源：第 2.6 节、第 2.7 节


CXL 事务层
----------

CXL 事务层定义了 CXL.io、CXL.cache 和 CXL.mem 三种协议的事务格式和处理规则。

CXL.io 协议
~~~~~~~~~~~

CXL.io 提供非一致性的加载/存储接口，事务类型、包格式、流控、虚拟通道管理和排序规则遵循 PCIe 规范。

**与标准 PCIe 的差异**：

1. **端点模式**：CXL 1.1 模式下，设备作为 RCiEP 暴露给软件
2. **INTx 禁止**：参与 CXL 协议的端点函数不得生成 INTx 消息
3. **电源管理 VDM**：使用 Vendor Defined Message 格式传输 PM 消息

**CXL 电源管理消息**：

CXL PM 消息使用 PCIe Vendor Defined Type 0 消息，4-DWORD 数据载荷：

- Vendor ID：1E98h
- VDM Code：68h（CXL PM Message）
- 消息类型：PMREQ、PMRSP、PMGO 等

**信用和 PM 初始化**：

设备上电后，主机通过 AGENT_INFO 消息交换信用和能力向量。设备报告其支持的功能：
- Bit 0：ResetPrep 支持
- Bit 1：GPF 支持
- Bit 2：PMREQ 支持

参考来源：第 3.1 节

CXL.cache 协议
~~~~~~~~~~~~~~

CXL.cache 是设备缓存与主机一致性引擎之间的一致性协议。

通道概述
^^^^^^^

CXL.cache 定义了 5 个通道，实现双向请求/响应通信：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 通道
     - 方向
     - 描述
   * - D2H Request
     - 设备 → 主机
     - 设备发起的读/写/驱逐请求
   * - D2H Response
     - 设备 → 主机
     - 设备对主机监听的响应
   * - D2H Data
     - 设备 → 主机
     - 设备发送的数据（驱逐数据、监听响应数据）
   * - H2D Request
     - 主机 → 设备
     - 主机发起的监听请求
   * - H2D Response
     - 主机 → 设备
     - 主机对设备请求的响应（GO、WritePull 等）

**通道排序**：各通道独立工作以保证前向推进，通道内无排序规则（除非特别说明）。

**信用机制**：

设备为 H2D 通道提供信用，主机为 D2H 通道提供信用。信用返回通过链路层 Flit 头部编码。

参考来源：第 3.2.1 节、第 3.2.2 节

设备到主机请求（D2H Request）
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

D2H 请求由设备发起，用于访问主机内存或管理设备缓存状态。

**读请求类型**：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Opcode
     - 描述
   * - RdCurr
     - 读取当前数据，不改变缓存状态
   * - RdOwn
     - 请求独占所有权，用于写入
   * - RdShared
     - 请求共享状态，用于读取
   * - RdAny
     - 请求任意状态，优化延迟
   * - RdOwnNoData
     - 偏置翻转请求，请求所有权但不需数据

**写请求类型**：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Opcode
     - 描述
   * - ItoMWr
     - 使行无效并写入，获取独占状态后写入
   * - WrCur
     - 写入当前行，设备已拥有数据
   * - WrInv
     - 写入并使其他副本无效
   * - WOWrInv/WOWrInvF
     - 无所有权写入，写入后不保留所有权

**驱逐请求类型**：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Opcode
     - 描述
   * - CleanEvict
     - 干净行驱逐，通知主机设备不再缓存该行
   * - DirtyEvict
     - 脏行驱逐，将修改数据写回主机
   * - CleanEvictNoData
     - 干净行驱逐，不传输数据

**其他请求**：

- CLFlush：缓存行刷新
- CacheFlushed：设备所有缓存已刷新的指示

参考来源：第 3.2.4.2 节

主机到设备响应（H2D Response）
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

H2D 响应由主机发送，指示设备请求的处理结果。

**响应类型**：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 响应类
     - 描述
   * - GO
     - 全局观察（Global Observation），指示请求已在系统中完成，携带 MESI 状态
   * - WritePull
     - 请求设备发送写入数据
   * - GO_WritePull
     - 组合响应，同时完成 GO 和 WritePull
   * - ExtCmp
     - 扩展完成，指示 FastGO 后数据已在内存可见
   * - Fast_GO_WritePull
     - 快速 GO，仅保证本地观察，后续发送 ExtCmp

**GO 响应携带的 MESI 状态**：

- **GO-M**：授予修改状态，设备拥有唯一脏副本
- **GO-E**：授予独占状态，设备可写入
- **GO-S**：授予共享状态，设备只读
- **GO-I/GO-Err**：无效或错误，设备不得缓存数据

**全局观察（GO）语义**：

GO 消息传达：

- 读请求：请求一致性完成
- 写请求：请求一致且持久完成
- MESI 状态编码指示请求者缓存应放置的状态

参考来源：第 3.2.4.5 节

监听机制
^^^^^^^^

主机通过监听请求（Snoop Request）维护缓存一致性。监听请求通过 H2D Request 通道发送。

**监听类型**：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Opcode
     - 描述
   * - SnpData
     - 监听数据，用于读请求，设备需无效或降级到共享
   * - SnpInv
     - 监听无效，用于写请求，设备需无效化缓存行
   * - SnpCur
     - 监听当前，获取当前数据，不改变缓存状态

**设备监听响应**：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 响应
     - 描述
   * - RspIHitI
     - 未命中缓存，行已从设备清除
   * - RspVHitV
     - 命中缓存，状态未变
   * - RspIHitSE
     - 命中干净状态，现已无效
   * - RspSHitSE
     - 命中干净状态，降级到共享
   * - RspSFwdM
     - 命中修改状态，降级到共享，返回数据
   * - RspIFwdM
     - 命中修改状态（HITM），现已无效，返回数据
   * - RspVFwdV
     - 返回当前数据，状态不变

参考来源：第 3.2.4.3 节、第 3.2.4.4 节

缓存状态规则
^^^^^^^^^^^^

设备在发起 CXL.cache 请求时，需遵守"埋藏缓存状态"（Buried Cache State）规则：

**请求约束**：

.. list-table::
   :header-rows: 1
   :widths: 15 15 15 15 15 25

   * - 请求
     - Modified
     - Exclusive
     - Shared
     - Invalid
     - 备注
   * - RdCurr/RdOwn/RdShared/RdAny
     - 禁止
     - 禁止
     - 禁止
     - 允许
     - 缓存行不在设备中
   * - RdOwnNoData
     - 禁止
     - 禁止
     - 允许
     - 允许
     - 可从共享升级
   * - ItoMWr/WrCur/CLFlush
     - 禁止
     - 禁止
     - 禁止
     - 允许
     - 需先获取所有权
   * - CleanEvict
     - 禁止
     - 允许
     - 禁止
     - -
     - 干净行驱逐
   * - DirtyEvict
     - 允许
     - 禁止
     - 禁止
     - -
     - 脏行驱逐

参考来源：第 3.2.5.14 节

CXL.mem 协议
~~~~~~~~~~~~~

CXL.mem 是 CPU 与内存之间的事务接口，支持多种内存附加方式和内存类型。

通道概述
^^^^^^^

CXL.mem 定义了 6 个通道：

.. list-table::
   :header-rows: 1
   :widths: 20 20 60

   * - 通道
     - 方向
     - 描述
   * - M2S Request (Req)
     - 主机 → 设备
     - 内存请求（不带数据）
   * - M2S Request with Data (RwD)
     - 主机 → 设备
     - 带数据的内存请求（写）
   * - M2S Back-Invalidation Response (BIRsp)
     - 主机 → 设备
     - 后端无效化响应
   * - S2M Back-Invalidation Snoop (BISnp)
     - 设备 → 主机
     - 后端无效化监听
   * - S2M No Data Response (NDR)
     - 设备 → 主机
     - 无数据响应（完成）
   * - S2M Data Response (DRS)
     - 设备 → 主机
     - 带数据响应

**BISnp/BIRsp 通道**：CXL 3.0 引入，支持 Type 2 设备的多设备扩展和直接点对点访问 HDM。

参考来源：第 3.3.1 节、第 3.3.2 节

M2S 请求
^^^^^^^^

**内存操作码**：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Opcode
     - 描述
   * - MemRd
     - 内存读
   * - MemRdData
     - 内存读数据（预取）
   * - MemRdFar
     - 远端内存读
   * - MemInv
     - 内存无效化
   * - MemRdInv
     - 内存读并无效化

**元数据字段**：

CXL.mem 支持可选的元数据，用于实现粗粒度监听过滤器：

- MetaField：指定元数据字段
- MetaValue：元数据值
- 状态值：Invalid、Shared、Exclusive、Modified

**监听类型**：

- SnpNone：无监听
- SnpData：监听数据
- SnpInv：监听无效

参考来源：第 3.3.5 节

后端无效化监听（BISnp）
^^^^^^^^^^^^^^^^^^^^^^^

BISnp 允许设备（Type 2 或 Type 3）主动使主机缓存中的行无效，用于：

1. **设备偏置转换**：从主机偏置切换到设备偏置
2. **设备附加内存的点对点访问**：其他设备通过 CXL.io 直接访问 HDM
3. **多设备扩展**：支持单个主机桥下最多 16 个设备

**BISnp 操作码**：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Opcode
     - 描述
   * - SnpInv
     - 使指定缓存行无效
   * - SnpData
     - 监听数据并可能降级状态

**块后端无效化**：

支持一次无效化连续的多个缓存行，通过 Address[7:6] 编码块使能：

- 00b：单行无效化
- 01b：2 行块
- 10b：4 行块
- 11b：8 行块

参考来源：第 3.3.8 节

QoS 遥测
^^^^^^^^

CXL.mem 支持 QoS 遥测功能，允许主机根据设备负载调整请求速率。

**设备负载指示（DevLoad）**：

设备在 DRS/NDR 响应中报告 DevLoad 字段（0-255），指示当前负载水平。

**主机行为**：

- DevLoad = 0：无限制
- DevLoad 递增：主机应减少请求速率
- 主机根据 DevLoad 调整请求节流

参考来源：第 3.3.4 节

事务排序
~~~~~~~~

CXL 定义了严格的事务排序规则以确保一致性和前向推进。

**上游排序规则**（设备 → 主机）：

.. list-table::
   :header-rows: 1
   :widths: 25 25 25 25

   * - 通道
     - 对自身排序
     - 对其他通道排序
     - 备注
   * - D2H Request
     - 无保证
     - 需设备维护
     - 设备负责必要排序
   * - D2H Response
     - 无依赖
     - 无依赖
     - 可独立发送
   * - D2H Data
     - 必须连续
     - 无依赖
     - 同一行数据必须连续

**下游排序规则**（主机 → 设备）：

主机可自由重排请求，设备需处理乱序到达。特定约束：

- 监听必须在 GO 之后发送
- 监听和 WritePull 不能同时针对同一地址

参考来源：第 3.4 节、第 3.5 节

Type 3 设备事务流
~~~~~~~~~~~~~~~~~

Type 3 设备作为内存扩展器，事务流简化为读/写两类：

**读事务**：

1. 主机发送 M2S Request (MemRd)
2. 设备返回 S2M DRS (MemData)

与 Type 2 不同，Type 3 不返回 S2M NDR 响应。

**写事务**：

1. 主机发送 M2S RwD (MemWr)
2. 设备返回 S2M NDR (Cmp)

参考来源：第 3.7 节


CXL 链路层
----------

CXL 链路层负责在物理层之上提供可靠的数据传输机制。CXL.io 和 CXL.cache/mem 使用不同的链路层实现。

CXL.io 链路层
~~~~~~~~~~~~~

CXL.io 使用 PCIe 数据链路层作为链路层，负责 TLP 的可靠交换。

**关键差异**：

- 仅适用 x16 链路发送器和接收器帧规则，无论协商链路宽度
- 对于降级链路宽度，使用 x16/(降级链路宽度) 流传输
- 在 256B Flit 模式下，PCIe 定义的 PM 和链路管理 DLLP 不适用

**帧规则保证**：

如果发送的 TLP 恰好在 Flit 边界结束，必须有后续 CXL.io Flit。

参考来源：第 4.1 节

CXL.cache/mem 68B Flit 模式
~~~~~~~~~~~~~~~~~~~~~~~~~~~

68B Flit 模式支持最高 32 GT/s 的物理层速率。

Flit 结构
^^^^^^^^^

CXL.cachemem Flit 大小固定为 528 位（66 字节）：

.. list-table::
   :header-rows: 1
   :widths: 15 20 20 20 25

   * - 位置
     - 字节 0-1
     - 字节 2-15
     - 字节 16-63
     - 字节 64-65
   * - 协议 Flit
     - Flit 头部
     - Header Slot
     - 3 个 Generic Slot
     - CRC
   * - 数据 Flit
     - Flit 头部
     - -
     - 4 个数据块
     - CRC

**Flit 头部字段**：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 字段
     - 描述
   * - Type[1:0]
     - Flit 类型：00b=协议，01b=控制，10b=反向协议
   * - Ak
     - 确认位，确认 8 个成功 Flit
   * - BE
     - 字节使能指示
   * - Sz
     - 大小指示（全/半缓存行）
   * - CRV
     - 信用返回反向标志
   * - ReqCrd/DataCrd/RspCrd
     - 请求/数据/响应信用返回
   * - Slot 0-3
     - Slot 格式类型编码

参考来源：第 4.2.2 节

Slot 格式
^^^^^^^^^

Slot 携带协议消息或数据。H2D/M2S 和 D2H/S2M 方向使用不同的 Slot 格式。

**H2D/M2S Slot 格式**：

.. list-table::
   :header-rows: 1
   :widths: 10 40 20

   * - 格式
     - 内容
     - 长度
   * - H0
     - CXL.cache Req + Rsp
     - 96 位
   * - H1
     - CXL.cache Data Header + 2 Rsp
     - 88 位
   * - H2
     - CXL.cache Req + Data Header
     - 88 位
   * - H3
     - 4 CXL.cache Data Header
     - 96 位
   * - H4
     - CXL.mem RwD Header
     - 87 位
   * - H5
     - CXL.mem Req Only
     - 87 位
   * - H6
     - MAC Slot（用于完整性）
     - 96 位
   * - H7
     - 2 CXL.mem BIRsp
     - 72 位
   * - G0
     - 数据块
     - 128 位

**信用返回编码**：

信用返回使用指数编码，支持返回 0、1、2、4、8、16、32、64 个信用。

参考来源：第 4.2.3 节、第 4.2.5 节

链路层重试
^^^^^^^^^^

68B Flit 模式使用链路层重试（LLR）机制保证可靠传输。

**LLR 变量**：

- LL_REPLAY_TIMER：重播超时
- LL_REPLAY_COUNT：最大重播次数
- LL_RX_ACK_TIMER：确认超时

**LLR 状态机**：

发送端状态：REPLAY、ACTIVE_REPLAY
接收端状态：RETRY_LOCAL_NORMAL、RETRY_LOCAL_RECOVERY

**重试流程**：

1. 发送端检测到 CRC 错误或未收到 ACK
2. 进入 REPLAY 状态，重发缓冲的数据
3. 接收端请求重试时，发送端停止发送新数据
4. 重发完成后恢复正常传输

参考来源：第 4.2.8 节

256B Flit 模式
~~~~~~~~~~~~~~

256B Flit 模式支持更高物理层速率（如 64 GT/s），可靠性和重试机制移至物理层。

Flit 结构
^^^^^^^^^

256B Flit 包含 252 字节负载和 4 字节 CRC：

- Flit 头部：2 字节
- Flit 数据：240 字节
- CRC：4 字节（可选 6 字节用于延迟优化）

**Slot 定义**：

- G-Slot：通用 Slot，最大 124 位
- H-Slot：Header Slot，最大 108 位
- HS-Slot：Header Short Slot，最大 92 位（延迟优化）

**隐式数据 Slot 解码**：

数据和字节使能 Slot 可根据先前消息头部隐式确定。G-Slot 的 MSB 用于编码数据：

- MSB = 1：隐式数据 Slot
- MSB = 0：显式 Slot 格式

参考来源：第 4.3.1 节、第 4.3.3 节

信用返回
^^^^^^^^

256B Flit 使用 2 字节信用返回编码：

.. list-table::
   :header-rows: 1
   :widths: 15 25 25 35

   * - 编码
     - 协议
     - 通道
     - 信用数量
   * - 04h-08h
     - Cache
     - H2D Request
     - 1/4/8/12/16
   * - 09h-0Dh
     - Cache
     - D2H Request
     - 1/4/8/12/16
   * - 10h-14h
     - Memory
     - M2S Request
     - 1/4/8/12/16
   * - 15h-19h
     - Memory
     - S2M BISnp
     - 1/4/8/12/16

参考来源：第 4.3.5 节

延迟优化
^^^^^^^^

256B Flit 支持延迟优化模式：

**延迟优化 Flit**：

将 256 字节 Flit 分为两个 128 字节半 Flit，允许提前消费：

- Even 半 Flit：字节 0-127
- Odd 半 Flit：字节 128-255

**CRC 处理**：

- 每个 128 字节半 Flit 独立计算 6 字节 CRC
- 如果其中一个半 Flit CRC 失败，仍可消费另一个

**6 字节 CRC 计算**：

使用 (130, 136) Reed-Solomon 码，生成多项式：

g(x) = x^6 + α^147*x^5 + α^107*x^4 + α^250*x^3 + α^114*x^2 + α^161*x + α^21

参考来源：第 4.3.8 节、第 6.2.3.1.2 节


CXL ARB/MUX
-----------

ARB/MUX 负责在 CXL.io 和 CXL.cachemem 之间进行仲裁和复用。

虚拟链路状态机（vLSM）
~~~~~~~~~~~~~~~~~~~~~~

每个链路层接口维护独立的 vLSM 状态。

**vLSM 状态**：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 状态
     - 描述
   * - RESET
     - 复位状态
   * - L0_WAIT_ACTIVE
     - 等待激活
   * - L0
     - 正常操作状态
   * - L0p
     - 低功耗状态（仅 256B Flit）
   * - L1
     - ASPM L1 状态
   * - L2
     - 链路断开状态

**状态转换规则**：

- 本地 vLSM 转换：根据本地请求和条件
- 跨链路同步：通过 ALMP（ARB/MUX Link Management Packet）协商

参考来源：第 5.1 节

ALMP 消息
~~~~~~~~~

ALMP 用于 vLSM 状态同步和链路管理。

**ALMP 格式**：

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - 字节
     - 字段
     - 描述
   * - Byte 1
     - Opcode
     - 操作码
   * - Byte 2-3
     - vLSM/L0p 信息
     - 状态和参数

**ALMP 操作码**：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 操作码
     - 描述
   * - State Request ALMP
     - 请求状态转换
   * - State Status ALMP
     - 报告当前状态
   * - L0p Negotiation ALMP
     - L0p 协商

参考来源：第 5.2 节

仲裁机制
~~~~~~~~

ARB/MUX 根据 CXL.io 和 CXL.cachemem 的优先级和公平性进行仲裁：

- CXL.io：来自 PCIe 事务层
- CXL.cachemem：来自 CXL.cache/mem 链路层

**ARB/MUX 旁路**：

在特定条件下可旁路仲裁，直接传输数据，优化延迟。

参考来源：第 5.3 节


CXL 物理层
----------

Flex Bus 物理层支持 CXL 和 PCIe 模式的动态协商。

帧格式
~~~~~~

**CXL 1.1/CXL 2.0 Flit 模式**：

在 PCIe 128b/130b 编码之上添加 2 字节 Protocol ID 和 2 字节保留字段。

**Protocol ID 编码**：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Protocol ID
     - 描述
   * - 0000h
     - NULL Flit（链路初始化）
   * - 0001h
     - CXL.io 数据块
   * - 0002h
     - CXL.cache/mem 数据块
   * - 0003h
     - CXL.cachemem 控制消息
   * - FFFFh
     - 错误指示

**256B Flit 模式**：

Flit 头部包含 Flit Type 字段：

- 00b：协议 Flit
- 01b：控制 Flit
- 10b：反向协议 Flit

参考来源：第 6.2 节

链路训练
~~~~~~~~

**模式协商**：

Flex Bus 模式通过修改的 TS1/TS2 Ordered Set 进行协商：

**阶段 1**：下游端口发送修改的 TS1，通告 Flex Bus 能力
**阶段 2**：下游端口发送修改的 TS2，指示最终选择

**协商字段**：

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - 字段
     - 描述
   * - Bit 0: PCIe capable/enable
     - PCIe 能力/使能
   * - Bit 1: CXL.io capable/enable
     - CXL.io 能力/使能
   * - Bit 2: CXL.mem capable/enable
     - CXL.mem 能力/使能
   * - Bit 3: CXL.cache capable/enable
     - CXL.cache 能力/使能
   * - Bit 4: CXL 2.0 capable/enable
     - CXL 2.0 能力/使能
   * - Bit 8: MLD capable/enable
     - 多逻辑设备能力
   * - Bit 9: 68B-Enhanced Flit capable
     - 68B 增强 Flit 能力
   * - Bit 10: Sync Header Bypass capable
     - 同步头旁路能力
   * - Bit 11: 256B Latency-Optimized Flit capable
     - 256B 延迟优化 Flit 能力

**CXL 2.0 vs CXL 1.1 协商**：

.. list-table::
   :header-rows: 1
   :widths: 30 30 40

   * - 上游组件
     - 下游组件
     - 链路训练结果
   * - Host - CXL 2.0
     - Switch - CXL 2.0
     - CXL 2.0 模式
   * - Host - CXL 1.1
     - Switch - CXL 2.0
     - CXL 协商失败
   * - Host - CXL 2.0
     - Endpoint - CXL 1.1
     - CXL 1.1 模式（初始上电）/ 失败（热添加）
   * - Host - CXL 1.1
     - Endpoint - CXL 2.0
     - CXL 1.1 模式

参考来源：第 6.4 节

Retimer 支持
~~~~~~~~~~~~

Retimer 用于扩展链路距离，支持 Sync Header Bypass 优化。

**Retimer 感知要求**：

- Retimer 必须通告 CXL 感知能力
- Sync Header Bypass 需要 Retimer 支持
- 通过修改的 TS1/TS2 中 Retimer1/Retimer2 CXL aware 位指示

参考来源：第 6.8 节


CXL 交换
--------

CXL Switch 支持多虚拟 CXL Switch（VCS）和热插拔功能。

Switch 架构
~~~~~~~~~~~

**单 VCS Switch**：一个物理 Switch 实现单个 VCS
**多 VCS Switch**：一个物理 Switch 实现多个隔离的 VCS，每个 VCS 连接不同主机

**MLD 端口支持**：

Switch 可支持 MLD 端口，每个端口可绑定多个 LD 到不同的 vPPB。

参考来源：第 7.1 节

绑定与解绑
~~~~~~~~~~

**绑定操作**：

将 vPPB 绑定到物理端口或 LD：

1. Fabric Manager 发送 Bind vPPB 命令
2. Switch 配置路由
3. 返回完成事件

**解绑操作**：

将 vPPB 从物理端口或 LD 解绑：

- 等待链路断开后解绑
- 模拟管理热移除
- 模拟意外热移除

参考来源：第 7.2.3 节

Fabric Manager API
~~~~~~~~~~~~~~~~~~~

FM API 提供 Switch 和 MLD 管理接口。

**命令集**：

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - 命令集
     - 功能
   * - Physical Switch
     - 识别 Switch、获取端口状态、端口控制
   * - Virtual Switch
     - 获取 VCS 信息、绑定/解绑 vPPB、生成 AER 事件
   * - MLD Port
     - 隧道管理命令、LD 配置/内存请求
   * - MLD Component
     - LD 信息/分配、QoS 控制

**消息格式**：

FM API 消息使用 CCI（Component Command Interface）消息格式：

- 命令 opcode：标识命令类型
- 输入载荷：命令参数
- 输出载荷：返回数据

参考来源：第 7.6 节

Fabric 地址空间
~~~~~~~~~~~~~~~

跨域通信使用 Fabric 地址空间。

**地址映射**：

- 主机物理地址（HPA）映射到 Fabric 地址
- 交换机入口将 HPA 解码为目标 ID
- 设备使用 Fabric 地址进行跨域访问

**GFAM 访问保护**：

- 使用 SPID 进行访问权限验证
- GFAM 设备仅使用低位地址

参考来源：第 7.7 节


控制与状态寄存器
----------------

CXL 寄存器分为配置空间寄存器和内存映射寄存器。

DVSEC 结构
~~~~~~~~~~

**CXL 设备 DVSEC**（DVSEC ID 0000h）：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 寄存器
     - 描述
   * - DVSEC CXL Capability
     - 设备能力：缓存大小、内存大小、偏置支持等
   * - DVSEC CXL Control
     - 控制位：缓存使能、内存使能等
   * - DVSEC CXL Status
     - 状态位：缓存无效、复位状态等
   * - DVSEC CXL Control2
     - 扩展控制：禁止缓存、复位、安全擦除使能等
   * - DVSEC CXL Range Registers
     - CXL Range 1/2 Base/Limit 寄存器

**CXL 2.0 端口扩展 DVSEC**（DVSEC ID 0003h）：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 寄存器
     - 描述
   * - Port Extension Status
     - PM 初始化完成、Viral 状态
   * - Port Control Extensions
     - SBR/Link Disable 解除掩码、Alt 内存空间使能
   * - Alternate Bus/Memory Base/Limit
     - 替代地址解码范围
   * - CXL RCRB Base
     - CXL 1.1 RCRB 基地址

参考来源：第 8.1 节

HDM 解码器
~~~~~~~~~~

HDM（Host-managed Device Memory）解码器将主机物理地址映射到设备物理地址。

**解码器寄存器结构**：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 寄存器
     - 描述
   * - Base High/Low
     - 地址范围基址
   * - Size High/Low
     - 地址范围大小
   * - Control
     - 交错粒度（IG）、交错路数（IW）、提交控制
   * - Target List High/Low
     - 交错目标端口列表
   * - DPA Skip High/Low
     - DPA 跳过长度（设备端）

**交错编码**：

- **交错粒度（IG）**：256B、512B、1KB、2KB、4KB、8KB、16KB
- **交错路数（IW）**：1路、2路、4路、8路、16路、3路、6路、12路

**解码器提交**：

1. 配置解码器寄存器
2. 可选设置 Lock On Commit 位
3. 设置 Commit 位
4. 等待 Committed 位置位

**一致性检查**（Lock On Commit = 1 时）：

- 解码器地址范围不重叠
- 交错目标端口不重复
- 地址范围有效（无回绕）

参考来源：第 8.2.5.19 节

IDE 能力结构
~~~~~~~~~~~~

CXL IDE（Integrity and Data Encryption）提供链路完整性和数据加密。

**IDE 模式**：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 模式
     - 描述
   * - Containment Mode
     - 容器模式，检测到错误后阻止数据传输
   * - Skid Mode
     - 滑动模式，允许有限数据在错误检测后传输

**IDE 能力寄存器**：

- CXL IDE Capable：IDE 能力指示
- Supported CXL IDE Modes：支持的模式
- Supported Algorithms：支持的算法（AES-GCM 256-bit key, 96-bit MAC）

**IDE 控制**：

- PCRC Disable：禁止明文 CRC 增强
- Key Refresh Time：密钥刷新时间
- Truncation Transmit Delay：截断传输延迟

参考来源：第 8.2.5.21 节

组件命令接口（CCI）
~~~~~~~~~~~~~~~~~~~

CCI 提供统一的命令接口，用于设备管理和状态查询。

**命令返回码**：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 返回码
     - 描述
   * - Success
     - 命令成功
   * - Background Command Started
     - 后台命令已启动
   * - Invalid Input
     - 输入参数无效
   * - Internal Error
     - 内部错误
   * - Retry Required
     - 需要重试
   * - Unsupported
     - 不支持
   * - Busy
     - 设备忙

**命令集**：

1. **事件命令集**：获取/清除事件记录
2. **固件命令集**：传输/激活固件
3. **时间戳命令集**：获取/设置时间戳
4. **日志命令集**：获取支持的日志、获取日志
5. **特性命令集**：获取/设置特性
6. **维护命令集**：执行维护操作（PPR）
7. **内存设备命令集**：识别设备、分区信息、健康状态、毒化列表等
8. **FM API 命令集**：Switch 和 MLD 管理
9. **信息与状态命令集**：识别、后台操作状态

参考来源：第 8.2.10 节

事件记录
^^^^^^^^

CCI 支持多种事件记录类型：

**通用事件记录格式**：

- 事件类型
- 时间戳
- 保留字段

**事件类型**：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 事件类型
     - 描述
   * - General Media Event
     - 通用媒体事件（ECC 错误等）
   * - DRAM Event
     - DRAM 事件
   * - Memory Module Event
     - 内存模块事件
   * - Vendor Specific Event
     - 厂商特定事件

参考来源：第 8.2.10.1 节


复位与初始化
------------

CXL 设备复位类型
~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 复位类型
     - 描述
   * - Hot Reset
     - 通过链路触发（LTSSM 或链路断开）
   * - Warm Reset
     - 通过外部信号 PERST# 触发
   * - Cold Reset
     - 主电源移除和 PERST#
   * - CXL Reset
     - 软件触发的协议级复位
   * - Function Level Reset (FLR)
     - 功能级复位

参考来源：第 9.1 节

系统复位流程
~~~~~~~~~~~~

**平台触发的复位流程**：

1. 主机发送 PM VDM（RESETPREP）通知设备
2. 设备刷新上下文到主机内存
3. 设备将内存置于安全状态（如自刷新）
4. 设备响应 RESETPREP
5. 主机进入 LTSSM Hot-Reset

**意外复位**：

无警告消息，直接进入 LTSSM Detect 或 PERST#。

参考来源：第 9.3 节

CXL Reset
~~~~~~~~~

CXL Reset 复位所有 CXL.cache 和 CXL.mem 状态，但不影响物理链路。

**复位行为**：

- CXL.mem 读请求：静默丢弃
- CXL.mem 写请求：持久 HDM 保持，易失性 HDM 可丢弃
- 设备缓存：写回并无效化
- 非粘性寄存器：初始化为默认值
- 粘性寄存器：保持不变

**易失性 HDM 内容处理**：

- CXL Reset Mem Clr Capable 位：指示清除/随机化能力
- CXL Reset Mem Clr Enable 位：请求清除/随机化

参考来源：第 9.7 节

全局持久刷新（GPF）
~~~~~~~~~~~~~~~~~~~

GPF 用于电源故障场景，确保持久数据写入持久存储。

**GPF 阶段**：

- **Phase 1**：主机发起 GPF 警告，设备准备
- **Phase 2**：设备执行刷新操作，主机等待

**GPF 寄存器**：

- Port GPF Phase 1/2 Timeout：超时配置
- Device GPF Phase 2 Time/Power：设备能力和状态

参考来源：第 9.8 节

热插拔
~~~~~~

CXL 支持两种热插拔模式：

**管理热插拔**：

1. 主机发送 PM VDM 通知设备
2. 设备准备断开连接
3. 主机发送 Unbind 请求
4. Switch 解绑 vPPB

**意外热移除**：

设备突然断开，Switch 检测链路断开并通知 FM。

参考来源：第 9.9 节


电源管理
--------

CXL 电源管理遵循 PCIe ASPM 规范，并添加 CXL 特定的 PM VDM 消息。

PM 状态
~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 状态
     - 描述
   * - L0
     - 正常操作
   * - L0s
     - 低功耗待机（可选）
   * - L0p
     - 低功耗协议状态（256B Flit 模式）
   * - L1
     - ASPM L1
   * - L2
     - 链路断开

PM 消息
~~~~~~~

CXL PM VDM 消息类型：

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 消息
     - 描述
   * - AGENT_INFO
     - 能力向量交换
   * - RESETPREP
     - 复位准备通知
   * - PMREQ
     - 电源管理请求
   * - GPF
     - 全局持久刷新

参考来源：第 10 章


安全性
------

CXL IDE（Integrity and Data Encryption）提供链路级安全。

IDE 架构
~~~~~~~~

**68B Flit 模式 IDE**：

- 使用 H6 Slot 携带 MAC
- 每个 Flit 包含 96 位 MAC

**256B Flit 模式 IDE**：

- MAC 在 Flit 末尾
- 支持延迟优化模式

IDE 密钥管理
~~~~~~~~~~~~

**密钥刷新协议（CXL_IDE_KM）**：

1. 发现消息：查询 IDE 能力
2. 密钥编程消息：设置加密密钥
3. 激活/密钥刷新消息：切换到新密钥
4. 获取密钥消息：查询当前密钥状态

**早 MAC 终止**：

允许提前发送 MAC 以优化延迟，但需要截断传输延迟配置。

参考来源：第 11 章


可靠性与可用性
--------------

错误处理
~~~~~~~~

**协议错误**：

- CRC 错误：触发链路层重试
- 事务超时：报告超时错误
- 监听响应错误：记录并报告

**隔离机制**：

CXL.cache/mem 支持隔离状态，在检测到致命错误时：

- CXL.mem：停止处理请求
- CXL.cache：停止处理监听

**Viral 处理**：

Viral 状态用于传播不可恢复错误：

- 设备检测到不可恢复错误时进入 Viral 状态
- Switch 转发 Viral 状态
- 主机处理 Viral 状态

参考来源：第 12 章


性能监控
--------

QoS 遥测
~~~~~~~~

设备通过 DevLoad 字段报告负载状态，主机根据负载调整请求速率。

性能监控单元（CPMU）
~~~~~~~~~~~~~~~~~~~~

CPMU 提供性能计数器：

- 事件类型：读写请求、延迟、带宽等
- 过滤器：按地址范围、事务类型过滤
- 计数器配置：事件选择、阈值设置

参考来源：第 13 章


总结
----

CXL 3.0 规范定义了一套完整的互连架构，支持：

1. **三种设备类型**：Type 1（带缓存）、Type 2（带内存的加速器）、Type 3（内存扩展器）
2. **三种协议**：CXL.io、CXL.cache、CXL.mem
3. **多种 Flit 模式**：68B、68B-Enhanced、256B
4. **交换与 Fabric**：支持多主机、MLD、GFAM
5. **完整的管理框架**：寄存器、命令接口、电源管理、安全、RAS

CXL 的核心价值在于将设备附加内存以一致性方式集成到系统地址空间，
为高性能计算和数据中心场景提供灵活的内存扩展和加速器互联解决方案。

参考来源：CXL 3.0 规范全文
