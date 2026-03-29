PCI Express 地址翻译与中断机制分析
====================================

本文档基于 PCI Express Base 5.0 规范，重点分析 ATS、PRI、PASID、MSI/MSI-X 等机制。
参考文档：docs/pdf/PCIE/PCI_Express_Base_5.0r1.0.md

.. contents::
   :depth: 2
   :local:


地址翻译服务（ATS）架构概述
----------------------------

*引用：Chapter 10.1 ATS Architectural Overview*

大多数现代系统架构都提供 DMA（总线主控）I/O 功能的地址翻译机制。在许多实现中，CPU 和 I/O 功能看到的物理地址空间是等价的，但在其他实现中并非如此。编程到 I/O 功能中的地址是一个"句柄"，由 Root Complex（RC）处理。这种处理的结果通常是翻译为中央复用器内的物理内存地址。通常，处理包括访问权限检查，以确保 DMA 功能被允许访问引用的内存位置。

**DMA 地址翻译的目的**（Chapter 10.1）：

- 限制"损坏"或错误编程的 DMA I/O 功能的破坏性
- 提供分散/聚集（scatter/gather）支持
- 能够重定向消息信号中断（如 MSI 或 MSI-X）到不同的地址范围，无需与底层 I/O 功能协调
- 地址空间转换（32 位 I/O 功能到更大的系统地址空间）
- 虚拟化支持

**ATS 的核心价值**（Chapter 10.1）：

根据实现方式，DMA 访问时间可能因需要解析实际物理地址而显著延长。如果实现需要访问主驻留翻译表，访问时间可能比未翻译访问长得多。此外，如果每个事务需要多次内存访问（例如，用于表遍历），则与 DMA 相关的内存事务率（即开销）可能很高。

为了减轻这些影响，设计通常在执行地址翻译的实体中包含地址翻译缓存。在 CPU 中，地址翻译缓存最常被称为翻译后备缓冲区（TLB）。对于 I/O TA，使用术语地址翻译缓存或 ATC 来区分它与 CPU 使用的翻译缓存。

ATS 允许 I/O 设备参与翻译过程并为其自己的内存访问提供 ATC。在设备中拥有 ATC 的好处包括：

- 能够通过分散地址翻译缓存责任来减轻 TA 资源压力（减少 TA 内的"颠簸"概率）
- 使 ATC 设备对系统 ATC 大小的性能依赖性降低
- 通过向中央复用器发送预翻译的请求来确保最佳访问延迟


翻译请求协议
------------

翻译请求 TLP 格式
~~~~~~~~~~~~~~~~~

*引用：Chapter 10.2.2 Translation Requests*

翻译请求具有类似于内存读取的格式。AT 字段用于区分翻译请求与普通内存读取。

**翻译请求头格式**（Figure 10-8, Figure 10-9）：

.. code-block:: text

   64-bit Translation Request Header:
   +--------+----------+----------+----------+
   | Fmt    | Type     | TC       | Attr     |
   +--------+----------+----------+----------+
   | Length        | Requester ID    | Tag     |
   +--------+----------+----------+----------+
   | Untranslated Address[63:32]                |
   +--------+----------+----------+----------+
   | Untranslated Address[31:12] | Reserved|NW |
   +--------+----------+----------+----------+

**关键字段说明**（Chapter 10.2.2）：

- **AT 字段**：设置为 01b 表示翻译请求
- **Untranslated Address**：待翻译的地址，bits[11:0] 隐含为零
- **NW（No Write）标志**：当设置时，表示功能请求对此翻译的只读访问
- **Length 字段**：指示请求中可能返回的翻译数量，每个翻译为 8 字节

**AT 字段编码**（Table 10-1）：

.. list-table::
   :header-rows: 1
   :widths: 15 25 60

   * - AT[1:0]
     - 助记符
     - 含义

     * - 00b
       - Untranslated
       - TA 可以将地址视为虚拟或物理地址

     * - 01b
       - Translation Request
       - TA 将返回请求中地址的翻译作为读取完成

     * - 10b
       - Translated
       - 事务中的地址已由 ATC 翻译

     * - 11b
       - Reserved
       - TA 将发出不支持的请求（UR）


翻译完成 TLP 格式
~~~~~~~~~~~~~~~~~

*引用：Chapter 10.2.3 Translation Completion*

TA 为每个翻译请求发送翻译完成。如果 TA 无法执行请求的翻译，使用 Figure 10-10 所示的格式。

**翻译完成状态码**（Table 10-2）：

.. list-table::
   :header-rows: 1
   :widths: 15 30 55

   * - 值
     - 状态
     - 含义

     * - 000b
       - Success
       - 此完成状态具有"成功"的名义含义。TA 不会在 Cpl 中返回此值。

     * - 001b
       - Unsupported Request (UR)
       - TA 不支持来自此功能的翻译请求

     * - 100b
       - Completer Abort (CA)
       - TA 由于 TA 中的错误无法翻译地址

**成功翻译完成数据条目**（Figure 10-12）：

.. code-block:: text

   Translation Completion Data Entry:
   +--------+----------+----------+----------+
   | Translated Address [63:32]                |
   +--------+----------+----------+----------+
   | Translated Address [31:12] | S| N|Res|U|W|R|
   |                            |  |  |   | | | |
   +--------+----------+----------+----------+
   | Exe | Priv | Global | Reserved         |
   +--------+----------+----------+----------+

**翻译完成数据字段**（Table 10-3）：

.. list-table::
   :header-rows: 1
   :widths: 15 85

   * - 字段
     - 含义

     * - S（Size）
       - 翻译大小 - 如果翻译适用于 4096 字节内存范围，此字段为 0b；如果适用于大于 4096 字节的范围，此字段为 1b

     * - N（Non-snooped）
       - 非嗅探访问 - 如果此字段为 1b，则使用此翻译的读/写请求必须清除 No Snoop 位

     * - U（Untranslated access only）
       - 仅未翻译访问 - 设置时，只能使用未翻译地址访问指示的范围

     * - R, W（Read, Write）
       - 指示使用翻译允许的事务类型：00b=不允许；01b=仅写；10b=仅读；11b=读写

     * - Exe（Execute Permitted）
       - 执行许可 - 设置时，请求功能被允许在关联内存范围内执行代码

     * - Priv（Privileged Mode Access）
       - 特权模式访问 - 设置时，R、W 和 Exe 引用授予特权模式实体的权限

     * - Global（Global Mapping）
       - 全局映射 - 设置时，ATC 被允许在所有 PASID 中缓存此映射条目


翻译范围大小编码
~~~~~~~~~~~~~~~~

*引用：Chapter 10.2.3.2 Translation Range Size (S) Field*

如果 S 设置，则翻译适用于大于 4096 字节的范围。低位地址按顺序消耗以指示与翻译关联的范围大小。

**翻译大小示例**（Table 10-4）：

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - 地址位 [63:12] 模式
     - 翻译范围大小（字节）

     * - x...x S=0
       - 4 K

     * - x...x 0 S=1
       - 8 K

     * - x...x 0 1 S=1
       - 16 K

     * - x...x 0 1 1 1 1 1 1 1 1 S=1
       - 2 M

     * - x 0 1 1 1...1 S=1
       - 1 G

     * - 0 1 1 1...1 S=1
       - 4 G


ATS 失效机制
------------

*引用：Chapter 10.3 ATS Invalidation*

当翻译在 TA 中更改且该翻译可能包含在功能的 ATC 中时，TA（与其关联的软件一起）必须向 ATC 发送失效请求，以维护 ATPT 和 ATC 之间的正确同步。

**失效请求消息格式**（Figure 10-13）：

.. code-block:: text

   Invalidate Request Message:
   +--------+----------+----------+----------+
   | Fmt    | Type     | TC       | Attr     |
   +--------+----------+----------+----------+
   | Message Code = 0000 0001b                |
   +--------+----------+----------+----------+
   | Requester ID        | ITag               |
   +--------+----------+----------+----------+
   | Device ID           | Reserved           |
   +--------+----------+----------+----------+
   | Untranslated Address[63:32]               |
   +--------+----------+----------+----------+
   | Untranslated Address[31:12] | Reserved   |
   +--------+----------+----------+----------+

**失效协议步骤**（Chapter 10.1.1, Figure 10-4）：

1. 系统软件更新 TA 使用的表中的条目。表更改后，TA 确定应在 ATC 中使翻译失效，并启动失效请求 TLP
2. 功能接收失效请求并使所有匹配的 ATC 条目失效
3. 当功能确定翻译地址的所有使用都已完成时，它发出一个或多个 ATS 失效完成


页面请求接口（PRI）
------------------

PRI 架构概述
~~~~~~~~~~~~

*引用：Chapter 10.1.2 Page Request Interface Extension*

ATS 改进了基于 DMA 的数据移动行为。关联的页面请求接口（PRI）通过允许在没有要求所有要移入或移出系统内存的数据被固定的情况下启动 DMA 操作，提供额外的优势。

与固定内存相关的开销可能不大，但从可分页池中移除大部分内存对系统性能的负面影响可能很显著。PRI 在功能上独立于 ATS 的其他方面。也就是说，支持 ATS 的设备不需要支持 PRI，但 PRI 依赖于 ATS 的能力。

**PRI 工作模型**（Chapter 10.4 Page Request Services）：

1. 功能确定它需要访问一个没有 ATS 翻译可用的页面
2. 功能使关联的页面请求接口向其 RC 发送页面请求消息
3. 当 RC 确定其对请求的响应时，它向请求功能发回 PRG 响应消息
4. 然后功能可以使用 ATS 请求请求页面的翻译

**页面请求消息格式**（Figure 10-16）：

.. code-block:: text

   Page Request Message:
   +--------+----------+----------+----------+
   | Type (10000)      | TC  |T|R| Length(0)|
   +--------+----------+----------+----------+
   | Message Code (0000 0100b)                |
   +--------+----------+----------+----------+
   | Requestor ID       | Tag                |
   +--------+----------+----------+----------+
   | Page Address [63:32]                      |
   +--------+----------+----------+----------+
   | Page Address [31:12] | PRG Index |L|W|R |
   +--------+----------+----------+----------+

**页面请求消息数据字段**（Table 10-5）：

.. list-table::
   :header-rows: 1
   :widths: 15 85

   * - 字段
     - 含义

     * - R（Read Access Requested）
       - 设置时，表示请求功能寻求对关联页面的读取访问

     * - W（Write Access Requested）
       - 设置时，表示请求功能寻求对关联页面的写入访问和/或零长度读取访问

     * - L（Last Request in PRG）
       - 设置时，表示关联页面请求是关联 PRG 的最后一个请求

     * - Page Request Group Index
       - 包含功能提供的关联页面请求标识符

     * - Page Address
       - 包含要加载页面的未翻译地址


PRG 响应消息
~~~~~~~~~~~~

*引用：Chapter 10.4.2 Page Request Group Response Message*

系统硬件和/或软件通过 PRG 响应消息与功能的页面请求接口通信。PRG 响应消息用于指示 PRG 的完成或接口的灾难性故障。

**响应码编码**（Table 10-7）：

.. list-table::
   :header-rows: 1
   :widths: 15 25 60

   * - 值
     - 状态
     - 含义

     * - 0000b
       - Success
       - 关联 PRG 中的所有页面都成功驻留

     * - 0001b
       - Invalid Request
       - PRG 中的一个或多个页面不存在或请求无法授予的访问权限

     * - 1111b
       - Response Failure
       - 关联请求组中的一个或多个页面遇到/导致灾难性错误


PASID（进程地址空间 ID）
------------------------

PASID TLP Prefix 格式
~~~~~~~~~~~~~~~~~~~~~

*引用：Chapter 6.20 PASID TLP Prefix*

PASID TLP Prefix 是一种端到端 TLP Prefix。当存在 PASID TLP Prefix 时，Prefix 中的 PASID 值与请求者 ID 一起标识与请求关联的进程地址空间 ID。每个功能都有独立的 PASID 值集合。

**PASID TLP Prefix 布局**（Figure 6-20）：

.. code-block:: text

   PASID TLP Prefix (4 bytes):
   +--------+----------+----------+----------+
   | 100    | 1        | 0001     |          |
   | (Prefix)|(End-End)|(PASID)   |          |
   +--------+----------+----------+----------+
   |Priv    |Exe       | Reserved |PASID[19:0]|
   |Mode    |Requested |          |          |
   +--------+----------+----------+----------+

**PASID TLP Prefix 字段**（Table 6-14）：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 位
     - 描述

     * - Byte 0 bits 7:5
       - 100b - 指示 TLP Prefix

     * - Byte 0 bit 4
       - 1b - 指示端到端 TLP Prefix

     * - Byte 0 bits 3:0
       - 0001b - 指示 PASID TLP Prefix

     * - Byte 1 bit 7
       - Privileged Mode Requested - 设置表示端点中的特权模式实体正在发出请求

     * - Byte 1 bit 6
       - Execute Requested - 设置表示端点正在请求执行权限

     * - Byte 1 bit 3 : Byte 3 bit 0
       - Process Address Space ID (PASID) - 包含与 TLP 关联的 PASID 值

**PASID TLP Prefix 允许的 TLP 类型**（Chapter 6.20）：

- 具有未翻译地址的内存请求（包括原子操作请求）
- 地址翻译请求
- 页面请求消息
- ATS 失效请求
- PRG 响应消息


PASID 使用规则
~~~~~~~~~~~~~~

*引用：Chapter 6.20.1 Managing PASID TLP Prefix Usage*

PASID TLP Prefix 的使用必须明确启用。除非启用，否则组件不允许传输 PASID TLP Prefix。

**端点功能规则**：

- 功能不允许发送和接收带有 PASID TLP Prefix 的 TLP，除非 PASID Enable 设置
- 功能必须具有将 PASID 使用与特定功能上下文动态关联的机制
- 功能必须具有请求优雅停止使用特定 PASID 的机制

**停止使用 PASID 的规则**：

1. 停止为此 PASID 排队新的页面请求消息
2. 完成传输此 PASID 的任何多页页面请求消息
3. 内部将此 PASID 的所有未完成页面请求消息标记为过时
4. 使用设备特定机制指示 PASID 已停止
5. 发送停止标记消息


MSI/MSI-X 中断机制
------------------

MSI/MSI-X 概述
~~~~~~~~~~~~~~

*引用：Chapter 6.1.4 MSI and MSI-X Operation*

消息信号中断（MSI）是一个可选功能，使设备功能能够通过将系统指定的数据值写入系统指定的地址（使用 DWORD 内存写事务）来请求服务。系统软件在设备配置期间初始化消息地址和消息数据，为每个支持 MSI 的功能分配一个或多个向量。

MSI-X 定义了基本 MSI 功能的单独可选扩展。与 MSI 相比，MSI-X 支持每个功能更多的最大向量数，软件控制别名的能力（当分配的向量少于请求时），以及每个向量使用独立地址和数据值的能力（由驻留在内存空间中的表指定）。

**MSI 与 MSI-X 对比**（Chapter 6.1.4）：

.. list-table::
   :header-rows: 1
   :widths: 25 35 40

   * - 特性
     - MSI
     - MSI-X

     * - 最大向量数
       - 32
       - 2048

     * - 向量控制
       - 多消息启用字段控制
       - 每向量独立地址/数据

     * - 每向量屏蔽
       - 可选
       - 标准

     * - 功能屏蔽
       - 不支持
       - 支持

**关键要求**（Chapter 6.1.4）：

- 所有能够生成中断的 PCI Express 设备功能必须支持 MSI 或 MSI-X 或两者
- 功能允许同时实现 MSI 和 MSI-X，但系统软件禁止同时启用两者
- 如果系统软件同时启用两者，行为未定义


MSI 配置
~~~~~~~~

*引用：Chapter 6.1.4.1 MSI Configuration*

系统软件读取消息控制寄存器以确定功能的 MSI 能力。

**MSI 配置步骤**：

1. 读取 Multiple Message Capable 字段确定请求的向量数
2. 写入 Multiple Message Enable 字段分配向量
3. 如果 Per-Vector Masking Capable 位设置，软件可屏蔽向量
4. 如果 64-bit Address Capable 位设置，初始化消息地址和消息上地址寄存器
5. 初始化消息数据寄存器

**MSI 能力结构**（Chapter 7.5.3）：

.. code-block:: text

   MSI Capability Structure:
   +--------+----------+
   | Cap ID | Next Ptr |
   +--------+----------+
   | Message Control       |
   +--------+----------+
   | Message Address (Low) |
   +--------+----------+
   | Message Upper Address | (64-bit only)
   +--------+----------+
   | Message Data          |
   +--------+----------+
   | Extended Message Data | (optional)
   +--------+----------+
   | Mask Bits             | (PVM only)
   +--------+----------+
   | Pending Bits          | (PVM only)
   +--------+----------+


MSI-X 配置
~~~~~~~~~~

*引用：Chapter 6.1.4.2 MSI-X Configuration*

MSI-X 支持最大 2048 个表条目。软件通过读取 Table Size 字段确定 MSI-X 表大小（编码为 N-1）。

**MSI-X 表条目格式**（Chapter 7.5.4）：

.. code-block:: text

   MSI-X Table Entry (16 bytes):
   +--------+----------+
   | Message Address (Low) |
   +--------+----------+
   | Message Upper Address |
   +--------+----------+
   | Message Data          |
   +--------+----------+
   | Vector Control        |
   +--------+----------+

**MSI-X 配置步骤**：

1. 读取 Table Size 字段确定表大小
2. 通过 Table Offset/Table BIR 计算表基址
3. 为每个使用的表条目填写地址、数据和向量控制字段
4. 向量控制字段可包含可选的引导标记字段


发送中断消息
~~~~~~~~~~~~

*引用：Chapter 6.1.4.4 Sending Messages*

一旦启用 MSI 或 MSI-X，并且一个或多个向量未屏蔽，功能被允许发送消息。要发送消息，功能对适当的消息地址执行 DWORD 内存写入，使用适当的消息数据。

**MSI 消息发送规则**：

- 如果 Extended Message Data Enable 位清除，写入的 DWORD 由 MSI 消息数据寄存器中的值（低两个字节）和零（高两个字节）组成
- 如果 Multiple Message Enable 字段非零，功能允许修改消息数据的低位以生成多个向量

**MSI-X 消息发送规则**：

- MSI-X 表包含每个分配向量的至少一个条目
- 来自选定表条目的 32 位消息数据字段值在消息中使用，功能不对低位进行任何修改


每向量屏蔽和功能屏蔽
~~~~~~~~~~~~~~~~~~~~

*引用：Chapter 6.1.4.5 Per-vector Masking and Function Masking*

每向量屏蔽（PVM）是 MSI 的可选功能，是 MSI-X 的标准功能。功能屏蔽是 MSI-X 的标准功能。

**屏蔽行为规则**：

- 当向量被屏蔽时，功能禁止发送关联的消息
- 功能必须在否则会发送消息时设置关联的 Pending 位
- 当软件取消屏蔽 Pending 位设置的向量时，功能必须安排发送关联的消息
- 发送消息后立即清除 Pending 位

**MSI-X 缓存规则**：

功能允许缓存未屏蔽 MSI-X 表条目的地址和数据值。但是，任何时候软件通过清除其屏蔽位或功能屏蔽位取消屏蔽当前屏蔽的 MSI-X 表条目，功能必须更新它从该条目缓存的任何地址或数据值。


ATS 配置寄存器
--------------

ATS 扩展能力结构
~~~~~~~~~~~~~~~~

*引用：Chapter 10.5.1 ATS Extended Capability*

每个支持 ATS 的功能必须在其扩展配置空间中具有 ATS 扩展能力结构。

**ATS 能力寄存器**（Table 10-9）：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 位
     - 描述

     * - 4:0
       - Invalidate Queue Depth - 功能在向上游连接施加反压之前可接受的失效请求数

     * - 5
       - Page Aligned Request - 设置表示未翻译地址始终对齐到 4096 字节边界

     * - 6
       - Global Invalidate Supported - 设置表示功能支持具有全局失效位设置的失效请求

     * - 7
       - Relaxed Ordering Supported - 设置表示允许在翻译请求中设置 RO 位

**ATS 控制寄存器**（Table 10-10）：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 位
     - 描述

     * - 4:0
       - Smallest Translation Unit (STU) - 指示翻译完成或失效请求中指示的最小 4096 字节块数

     * - 15
       - Enable (E) - 设置时，功能被启用以缓存翻译


页面请求扩展能力结构
~~~~~~~~~~~~~~~~~~~~

*引用：Chapter 10.5.2 Page Request Extended Capability Structure*

页面请求扩展能力结构用于配置页面请求接口机制。

**页面请求控制寄存器**（Table 10-12）：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 位
     - 描述

     * - 0
       - Enable (E) - 设置时，表示允许页面请求接口发出页面请求

     * - 1
       - Reset (R) - 当 Enable 字段清除时写入 1b，清除关联的实现相关页面请求信用计数器

**页面请求状态寄存器**（Table 10-13）：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 位
     - 描述

     * - 0
       - Response Failure (RF) - 设置表示功能收到指示响应失败的 PRG 响应消息

     * - 1
       - Unexpected Page Request Group Index (UPRGI) - 设置表示功能收到包含无匹配请求的 PRG 索引的 PRG 响应消息

     * - 7
       - Stopped (S) - 设置表示页面请求接口已停止

     * - 8
       - PRG Response PASID Required - 设置表示期望 PRG 响应消息包含 PASID TLP Prefix


PASID 扩展能力结构
~~~~~~~~~~~~~~~~~~

*引用：Chapter 7.8.8 PASID Extended Capability Structure*

**PASID 能力寄存器**（Chapter 7.8.8.2）：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 字段
     - 描述

     * - Max PASID Width
       - 支持的最大 PASID 位数（最多 20 位）

     * - Execute Permission Supported
       - 设置表示支持执行权限请求

     * - Privileged Mode Supported
       - 设置表示支持特权模式请求

**PASID 控制寄存器**（Chapter 7.8.8.3）：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 字段
     - 描述

     * - PASID Enable
       - 设置时，功能被允许发送和接收带有 PASID TLP Prefix 的 TLP

     * - Execute Permission Enable
       - 设置时，功能被允许设置 Execute Requested 位

     * - Privileged Mode Enable
       - 设置时，功能被允许设置 Privileged Mode Requested 位


系统架构交互说明
----------------

关于 PCIe 与 SMMU、CMN 的具体交互机制，PCI Express Base Specification 仅定义了协议层面的接口。以下内容基于规范中的相关描述：

**翻译代理（TA）与 ATC 的关系**（Chapter 10.1）：

规范描述了 TA 和 ATPT（地址翻译和保护表）作为平台组件，与具有集成 ATC 的 PCIe 设备交互。TA 和 ATPT 是实现特定的，可以是给定系统设计中的独立或集成组件。

**Root Complex 的角色**（Chapter 10.1.1）：

RC 接收来自功能的 ATS 翻译请求，并将其转发给 TA。TA 执行翻译后将结果通过 RC 返回给功能。RC 必须为每个 ATS 翻译请求生成至少一个 ATS 翻译完成。

**关于系统级实现的说明**（Chapter 10.1）：

本规范将提供允许 PCIe 设备与 TA 结合使用的互操作性，但 TA 及其 ATPT 被视为实现特定的，不在本规范范围内。SMMU、CMN 等 ARM 架构组件的具体实现细节需参考 ARM 相关规范（如 SMMU 架构规范、CMN 互连规范）。