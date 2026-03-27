MMU-600AE RAS 与 PMU 深入分析
===================================

本文档深入分析 MMU-600AE 的 RAS（可靠性、可用性、服务性）架构和
PMU（性能监控单元）事件机制，包括安全机制分类、错误检测与纠正流程、
FMU 交互以及如何利用 PMU 事件进行性能调优和故障诊断。

参考文档：ARM CoreLink MMU-600AE TRM（ARM DEN 0060D）

.. contents::
   :depth: 3
   :local:


1. RAS 架构总览
=================

1.1 两层 RAS 体系
----------------

MMU-600AE 的 "AE" 后缀表示功能安全（Functional Safety）版本，
在保留 MMU-600 原有逻辑不变的基础上增加了检测和纠正机制。
RAS 体系分为两层（来源：Section 4）：

.. list-table::
   :header-rows: 1
   :widths: 20 35 45

   * - 层次
     | 组件
     | 功能
   * - **Legacy RAS**
     | TCU_ERRFR/ERRCTLR/ERRSTATUS/ERRGEN
     | TCU/TBU 本地缓存错误的检测和注入，
       通过 ``ras_irpt`` 中断通知
   * - **FuSa RAS（FMU）**
     | Fault Management Unit
     | 全局故障管理：收集所有安全机制错误，
       错误记录、FHI/ERI 中断、安全机制启用/禁用/注入

1.2 FMU（Fault Management Unit）简介
-------------------------------------

FMU 位于 TCU 内部，是 MMU-600AE 功能安全的核心管理单元（来源：Section 4.4）：

- 接收所有安全机制的错误信号
- 将错误路由到 Safety Island（专用 APB 端口）
- 为每个 MMU 模块维护错误记录（FMU_ERR<n>STATUS）
- 允许软件启用/禁用安全机制（FMU_SMEN）
- 支持错误注入测试（FMU_SMINJERR）
- 在功能复位后**保留**错误记录（仅 ``fmu_aresetn`` 可清除）

1.3 中断类型
------------

MMU-600AE 提供两种安全相关中断（来源：Section 4.4.3）：

.. list-table::
   :header-rows: 1
   :widths: 15 20 65

   * - 中断
     | 信号
     | 触发条件
   * - **FHI**（Fault Handling Interrupt）
     | ``fmu_fault_int``
     | 所有检测到的错误（可纠正 + 不可纠正）。
       通过 ``FMU_ERR<n>CTLR.FI`` 使能
   * - **ERI**（Error Recovery Interrupt）
     | ``fmu_err_int``
     | 所有被报告为不可纠正的错误。
       通过 ``FMU_ERR<n>CTLR.UI`` 使能。
       可纠正错误被报告为不可纠正时也会触发
   * - **Legacy RAS 中断**
     | ``ras_irpt``
     | TCU/TBU 本地缓存错误。不能输出为 MSI，
       必须连接到中断控制器

复位后 FHI 和 ERI 默认禁用（``fmu_aresetn`` 控制），软件必须在启动时使能。


2. 安全机制分类
===============

2.1 TCU 安全机制
----------------

（来源：Table 4-7, Section 4.4）

.. list-table::
   :header-rows: 1
   :widths: 8 30 10 52

   * - SMID
     | 安全机制
     | 可纠正
     | 说明
   * - 1
     | TCU Dual LockStep
     | 否
     | 锁步逻辑比较器错误
   * - 2
     | TCU DTI AXI4-Stream 接口
     | 否
     | DTI 接口 AMBA parity 错误
   * - 3
     | TCU APB 接口
     | 否
     | 编程接口 parity 错误
   * - 4
     | TCU ACE-Lite DVM 请求接口
     | 否
     | QTW/DVM 接口 parity 错误
   * - 5
     | TCU LPI_PD Q-Channel
     | 否
     | 低功耗关断 Q-Channel 保护
   * - 6
     | TCU LPI_CG Q-Channel
     | 否
     | 时钟门控 Q-Channel 保护
   * - **7**
     | **TCU CCB Tag RAM CRC**
     | **是**
     | Configuration Cache Tag RAM 8-bit CRC
   * - **8**
     | **TCU CCB Entry RAM CRC**
     | **是**
     | Configuration Cache Data RAM 8-bit CRC
   * - **9**
     | **TCU WCB Tag RAM CRC**
     | **是**
     | Walk Cache Tag RAM 8-bit CRC
   * - **10**
     | **TCU WCB Entry RAM CRC**
     | **是**
     | Walk Cache Data RAM 8-bit CRC
   * - 11
     | TCU DTI Buffer RAM CRC
     | 否
     | DTIB 内部 buffer RAM CRC
   * - 12
     | FMU Ping Ack 错误
     | 否
     | Ping 超时或连接失败
   * - 13
     | FMU APB 接口
     | 否
     | FMU APB 接口 parity 错误
   * - 14
     | MBIST Req
     | 否
     | 默认禁用，功能模式下不应被断言
   * - 15
     | Tie-off 错误
     | 否
     | 复位后 tie-off 值反转检测

2.2 TBU 安全机制
----------------

.. list-table::
   :header-rows: 1
   :widths: 8 30 10 52

   * - SMID
     | 安全机制
     | 可纠正
     | 说明
   * - 0
     | TBU 未确定源错误
     | 否
     | 需重读 IERR；持续为 0 可能 DTI 互连断开
   * - 1
     | TBU Dual LockStep
     | 否
     | 锁步逻辑比较器错误
   * - 2
     | TBU ACE-Lite (TBS) 接口
     | 否
     | TBS completer 接口 parity
   * - 3
     | TBU ACE-Lite (TBM) 接口
     | 否
     | TBM requester 接口 parity
   * - 4
     | TBU DTI AXI4-Stream 接口
     | 否
     | DTI 接口 parity
   * - 5/6
     | TBU LPI_PD / LPI_CG
     | 否
     | Q-Channel 保护
   * - **7**
     | **TBU MTLB Tag RAM CRC**
     | **是**
     | Main TLB Tag RAM 8-bit CRC
   * - **8**
     | **TBU MTLB Entry RAM CRC**
     | **是**
     | Main TLB Data RAM 8-bit CRC
   * - 9
     | TBU WBB MFIFO RAM CRC
     | 否
     | Write Data Buffer FIFO RAM CRC

2.3 RAM 保护机制详解
--------------------

**可纠正 RAM**（缓存类型，故障时失效并重新获取）：

.. list-table::
   :header-rows: 1
   :widths: 40 30 30

   * - RAM
     | 保护方式
     | 纠正策略
   * - TCU CCB Tag/Data
     | 8-bit CRC（data+address）
     | 失效并重新获取
   * - TCU WCB Tag/Data
     | 8-bit CRC（data+address）
     | 失效并重新获取
   * - TBU MTLB Tag/Data
     | 8-bit CRC（data+address）
     | 失效并重新获取

CRC 保护能力（来源：Section 4.6）：

- 检测所有单比特错误（SBE）
- 数据宽度 <=128 bit 时检测所有双比特错误（DBE）
- 数据宽度 >128 bit 时 99.6% DBE 检测率
- 99.6% 多比特错误（MBE）检测率
- 99.6% 地址错误检测率
- 地址与数据组合生成 CRC，防止共模失效（CMF）

**不可纠正 RAM**（buffer 类型，故障时数据被使用，行为不可预测）：

- TCU Translation Request Buffer
- TBU Write Data Buffer

2.4 Lock-Step 逻辑保护
-----------------------

逻辑通过冗余锁步检查保护，标准时序延迟为 **2 个时钟周期**。
RAM 共享，比较器在模块顶层。（来源：Section 4.5）

- 比较器基于 XOR 树
- 使用 qualification（仅在 valid 信号断言时检查）节省功耗
- 可选比较器复制（``FUSA_COMP_DUP=1``）用于潜在故障诊断覆盖

2.5 外部接口保护
----------------

所有外部接口通过 AMBA Parity Extensions 进行点对点保护（来源：Section 4.7）：

- ACE-Lite、AXI4-Stream、APB 接口均有 parity 保护
- DTI Switch 不支持 parity，仅支持全复制保护
- ADB 和 Register Slice 也不支持 AXI4-Stream parity
- 中断输出带反转 chk parity 位，兼容 GIC-600AE
- Tie-off 输入通过反转复制保护：``port_chk[x:0] = ~port[x:0]``


3. PMU 事件体系
===============

3.1 PMU 硬件配置
----------------

每个 PMCG（Performance Monitor Counter Group）包含（来源：Section 2.6.4）：

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - 参数
     | MMU-600AE 值
   * - CAPTURE
     | 1（支持快照）
   * - MSI
     | 0（不支持 MSI，必须连接中断控制器）
   * - RELOC_CTRS
     | 1（PMCG 寄存器重定位到页 1）
   * - SIZE
     | 0x31（32-bit 计数器）
   * - NCTR
     | 0x3（4 个计数器）

3.2 事件过滤
------------

通过 ``SMMU_PMCG_SMR0`` 寄存器按 StreamID 过滤事件，可过滤以下事务类型：

- 推测性事务和翻译
- 导致终止事务或翻译故障的事务和翻译

3.3 SMMUv3 架构级事件
----------------------

**TCU 架构事件**（来源：Table 2-9）：

.. list-table::
   :header-rows: 1
   :widths: 12 25 10 53

   * - ID
     | 事件
     | 可过滤
     | 说明
   * - 0x00
     | Clock cycle
     | 否
     | 时钟周期计数。时钟门控后的周期不计
   * - 0x01
     | Transaction
     | 是
     | 来自 DTI-TBU 或 DTI-ATS 的翻译请求
   * - 0x02
     | TLB miss
     | 是
     | 需要遍历新页表条目的翻译请求
   * - 0x03
     | Config cache miss
     | 是
     | 需要遍历新配置表条目的翻译请求
   * - 0x04
     | Walk access
     | 是
     | 页表遍历访问计数
   * - 0x05
     | Config structure access
     | 是
     | 配置表遍历访问计数
   * - 0x06
     | ATS Translation Request
     | 是
     | 来自 DTI-ATS 的翻译请求

**TBU 架构事件**（来源：Table 2-10）：

.. list-table::
   :header-rows: 1
   :widths: 12 25 10 53

   * - ID
     | 事件
     | 可过滤
     | 说明
   * - 0x00
     | Clock cycle
     | 否
     | 时钟周期计数
   * - 0x01
     | Transaction
     | 是
     | TBM 接口上发出的事务计数
   * - 0x02
     | TLB miss
     | 是
     | 非推测性翻译请求（发给 TCU）
   * - 0x07
     | ATS Translation
     | 是
     | ATS 翻译后的事务计数


4. TCU 实现定义事件
====================

4.1 Walk Cache 事件（0x80-0x8F）
--------------------------------

（来源：Table 2-11）

.. list-table::
   :header-rows: 1
   :widths: 10 15 10 65

   * - ID
     | 事件
     | 可过滤
     | 说明
   * - 0x80
     | S1L0WC lookup
     | 是
     | S1L0 Walk Cache 查找次数
   * - 0x81
     | S1L0WC miss
     | 是
     | S1L0 Walk Cache 未命中次数
   * - 0x82/0x83
     | S1L1WC lookup/miss
     | 是
     | S1L1 Walk Cache 查找/未命中
   * - 0x84/0x85
     | S1L2WC lookup/miss
     | 是
     | S1L2 Walk Cache 查找/未命中
   * - 0x86/0x87
     | S1L3WC lookup/miss
     | 是
     | S1L3 Walk Cache 查找/未命中
   * - 0x88/0x89
     | S2L0WC lookup/miss
     | 是
     | S2L0 Walk Cache 查找/未命中
   * - 0x8A/0x8B
     | S2L1WC lookup/miss
     | 是
     | S2L1 Walk Cache 查找/未命中
   * - 0x8C/0x8D
     | S2L2WC lookup/miss
     | 是
     | S2L2 Walk Cache 查找/未命中
   * - 0x8E/0x8F
     | S2L3WC lookup/miss
     | 是
     | S2L3 Walk Cache 查找/未命中

4.2 TCU 缓存和缓冲事件（0x90-0xA0）
------------------------------------

.. list-table::
   :header-rows: 1
   :widths: 10 20 10 60

   * - ID
     | 事件
     | 可过滤
     | 说明
   * - 0x90
     | WC read
     | 是
     | Walk Cache RAM 读取（不含失效请求）。单次查找可能多次读取
   * - 0x91
     | Buffered translation
     | 是
     | 因所有翻译 slot 满而被缓冲。持续增长说明 slot 不足
   * - 0x92
     | CC lookup
     | 是
     | Configuration Cache 查找次数
   * - 0x93
     | CC read
     | 是
     | Configuration Cache RAM 读取（不含失效请求）
   * - 0x94
     | CC miss
     | 是
     | Configuration Cache 未命中。持续增长可能需增大缓存
   * - 0xA0
     | Speculative translation
     | 是
     | 被标记为推测性的翻译请求

4.3 TCU RAS 事件（0xC0-0xC8）
------------------------------

仅 ``SMMU_PMCG_SCR.SO = 1``（Secure only）时可见，**不受 StreamID 过滤**：

.. list-table::
   :header-rows: 1
   :widths: 10 25 65

   * - ID
     | 事件
     | 说明
   * - 0xC0-0xC3
     | S1L0-L3 WC error
     | Stage 1 各级 Walk Cache RAS 可纠正错误
   * - 0xC4-0xC7
     | S2L0-L3 WC error
     | Stage 2 各级 Walk Cache RAS 可纠正错误
   * - 0xC8
     | Config cache error
     | Configuration Cache RAS 可纠正错误


5. TBU 实现定义事件
====================

（来源：Table 2-12）

5.1 TLB 和翻译事件
------------------

.. list-table::
   :header-rows: 1
   :widths: 10 20 10 60

   * - ID
     | 事件
     | 可过滤
     | 说明
   * - 0x80
     | Main TLB lookup
     | 是
     | Main TLB 查找次数
   * - 0x81
     | Main TLB miss
     | 是
     | Main TLB 未命中。持续增长可能需增大 MTLB
   * - 0x82
     | Main TLB read
     | 是
     | Main TLB RAM 访问（不含失效）。不同页粒度可能多次访问
   * - 0x83
     | Micro TLB lookup
     | 是
     | Micro TLB 查找次数
   * - 0x84
     | Micro TLB miss
     | 是
     | Micro TLB 未命中。过高可能需优化工作集
   * - 0x88
     | Translation request
     | 是
     | 翻译请求总数（含推测性 + 非推测性）

5.2 缓冲和阻塞事件
-------------------

.. list-table::
   :header-rows: 1
   :widths: 10 22 10 58

   * - ID
     | 事件
     | 可过滤
     | 说明
   * - 0x85
     | Slots full
     | 否
     | 所有 slot 被占用的周期数（Secure only）
   * - 0x86
     | Out of trans tokens
     | 否
     | 翻译 token 耗尽无法发请求的周期数（Secure only）
   * - 0x87
     | Write data buf full
     | 否
     | 写数据缓冲区满导致阻塞的周期数（Secure only）
   * - 0x89
     | WDB use
     | 是
     | 使用写数据缓冲区的事务数
   * - 0x8A
     | WDB bypass
     | 是
     | 绕过写数据缓冲区的事务数

5.3 协议降级和 Stash 事件
---------------------------

.. list-table::
   :header-rows: 1
   :widths: 10 22 10 58

   * - ID
     | 事件
     | 可过滤
     | 说明
   * - 0x8B
     | MakeInvalid downgrade
     | 是
     | TBS MakeInvalid 降级为 TBM CleanInvalid 的次数。
       提示系统一致性协议交互
   * - 0x8C
     | Stash fail
     | 是
     | Stash 事务终止计数（翻译有效但被终止，或 StashWrite 降级）。
       StreamDisable/GlobalDisable 终止不计入

5.4 TBU RAS 事件
----------------

仅 Secure only，不受 StreamID 过滤：

.. list-table::
   :header-rows: 1
   :widths: 10 25 65

   * - ID
     | 事件
     | 说明
   * - 0xC0
     | Main TLB error
     | Main TLB RAS 可纠正错误


6. 如何使用 PMU 事件
=====================

6.1 性能调优场景
-----------------

**场景一：翻译延迟分析**

配置 TBU PMCG，按 StreamID 过滤特定设备的事件：

::

    PMCG0 (设备 A):
      Counter 0 -> Event 0x84 (Micro TLB miss)  -- 微 TLB 未命中率
      Counter 1 -> Event 0x81 (Main TLB miss)   -- 主 TLB 未命中率

    PMCG1 (设备 B):
      Counter 0 -> Event 0x01 (Transaction)       -- 总事务数
      Counter 1 -> Event 0x02 (TLB miss)          -- 未命中次数

如果 Micro TLB miss / Transaction 比率过高（>5%），
说明工作集超过 Micro TLB 容量，需要更均匀的地址分布或更大 TLB。

**场景二：Walk Cache 命中率分析**

配置 TCU PMCG：

::

    Counter 0 -> Event 0x01 (Transaction)        -- 总翻译请求
    Counter 1 -> Event 0x04 (Walk access)         -- 页表遍历访问
    Counter 2 -> Event 0x90 (WC read)             -- Walk Cache RAM 读取

Walk access / Transaction 越低，Walk Cache 命中率越高。
WC read 远大于 Walk access 说明单次查找需要多次 RAM 访问。

**场景三：翻译 slot 瓶颈**

::

    Counter 0 -> Event 0x91 (Buffered translation)  -- 翻译请求被缓冲

Buffered translation 持续增长说明 ``TCUCFG_XLATE_SLOTS`` 配置不足，
TBU 必须等待翻译 slot 释放导致延迟增加。

**场景四：Write Data Buffer 阻塞**

::

    TBU PMCG:
      Counter 0 -> Event 0x87 (Write data buffer full) -- WDB 满的周期数
      Counter 1 -> Event 0x8A (WDB bypass)             -- 绕过 WDB 的事务

WDB full 频繁触发说明多个 AXI ID 的写事务并发量超过缓冲区容量。

6.2 RAS 故障诊断场景
---------------------

**场景一：Walk Cache 数据损坏趋势**

配置 TCU PMCG，按 StreamID 过滤：

::

    Counter 0 -> Event 0x80 (S1L0WC lookup)  -- S1L0 查找
    Counter 1 -> Event 0xC0 (S1L0WC error)  -- S1L0 错误

S1L0WC error / S1L0WC lookup 比率上升可能指示硬件退化趋势。

**场景二：TBU Main TLB 完整性**

::

    TBU PMCG:
      Counter 0 -> Event 0x80 (Main TLB lookup)
      Counter 1 -> Event 0xC0 (Main TLB error)

定期采集快照，观察 Main TLB error 增量。
突然出现错误可能是单粒子翻转（SEU）或电源噪声。

6.3 PMU 快照机制
-----------------

PMU 快照接口在 TCU 和每个 TBU 上都有（来源：Section 2.6.5）：

- 4-phase 异步握手：``pmusnapshot_req``（输入）、``pmusnapshot_ack``（输出）
- 快照在 ``pmusnapshot_req`` 上升沿触发
- 等效于写入 ``SMMU_PMCG_CAPR.CAPTURE = 1``
- 将所有计数器值同时复制到 ``SMMU_PMCG_SVRn`` 寄存器
- 可连接到 SoC 调试基础设施

使用流程::

    ① SoC 调试模块拉高 pmusnapshot_req
    ② MMU 捕获所有计数器值到 SVRn
    ③ MMU 拉高 pmusnapshot_ack
    ④ SoC 调试模块读取 SVRn
    ⑤ SoC 调试模块拉低 pmusnapshot_req
    ⑥ MMU 拉低 pmusnapshot_ack

6.4 PMU 事件与 FMU RAS 的互补关系
----------------------------------

.. list-table::
   :header-rows: 1
   :widths: 20 40 40

   * - 维度
     | PMU RAS 事件
     | FMU RAS 错误记录
   * - 信息类型
     | 计数趋势（统计量）
     | 单次错误详情（状态机）
   * - 时间覆盖
     | 长期运行统计
     | 实时错误报告
   * - 粒度
     | 按缓存级别（S1L0/CC/MTLB 等）
     | 按 Safety Mechanism（SMID）
   * - 中断能力
     | SMMU_PMCG 内置中断
     | FHI/ERI 中断
   * - 适用场景
     | 性能趋势分析、预防性维护
     | 实时故障响应、安全合规


7. 错误注入
=============

7.1 Legacy RAS 错误注入
-----------------------

通过 ``TCU_ERRGEN`` / ``TBU_ERRGEN`` 注入 tag/data parity 错误。
**仅报告在 Legacy RAS 寄存器中，不报告在 FMU 中**。

**TCU_ERRGEN**（0x08EC0）：

.. list-table::
   :header-rows: 1
   :widths: 10 25 65

   * - 位
     | 名称
     | 说明
   * - [3]
     | TCC
     | Configuration Cache tag parity 错误注入
   * - [2]
     | DCC
     | Configuration Cache data parity 错误注入
   * - [1]
     | TWC
     | Walk Cache tag parity 错误注入
   * - [0]
     | DWC
     | Walk Cache data parity 错误注入

**TBU_ERRGEN**（0x08EC0）：

.. list-table::
   :header-rows: 1
   :widths: 10 25 65

   * - 位
     | 名称
     | 说明
   * - [1]
     | TMTLB
     | Main TLB tag parity 错误注入
   * - [0]
     | DMTLB
     | Main TLB data parity 错误注入

注意：tag parity 错误会屏蔽 data parity 错误，不要同时设置两者。

7.2 FMU 安全机制错误注入
-------------------------

通过 ``FMU_SMINJERR`` 向任意安全机制注入错误。写入前必须轮询
``FMU_STATUS.idle == 1``。

**关键行为差异**（来源：Section 4.4.6）：

- 注入到可纠正 RAM 的错误**仅影响 FMU 通路**，不触发 Legacy RAS 寄存器，
  **不导致缓存失效和重新获取**
- 通过 ERRGEN 注入的错误**仅影响 Legacy RAS 通路**，不产生 FMU 错误记录
- 注入到不可纠正 RAM 的错误不会被纠正，SMMU 继续运行（无实际 RAM 损坏）

7.3 PING 错误注入
-----------------

通过 ``FMU_PINGNOW`` 寄存器注入 PING 链路错误（来源：Section 4.4.7）：

- ``tcu_inject_error = 1``：注入 TCU->TBU 方向的 PING 错误
- ``tbu_inject_error = 1``：注入 TBU->TCU 方向的 PING_ACK 错误


8. 关键差异对比
===============

8.1 FMU 注入 vs Legacy 注入
----------------------------

.. list-table::
   :header-rows: 1
   :widths: 25 38 37

   * - 特性
     | FMU 注入（FMU_SMINJERR）
     | Legacy 注入（ERRGEN）
   * - 注入目标
     | 任意安全机制
     | 仅 tag/data parity
   * - 报告路径
     | FMU 错误记录
     | Legacy RAS 寄存器
   * - 可纠正 RAM 效果
     | 仅影响 FMU，不触发缓存失效
     | 触发缓存失效和重新获取
   * - 交叉报告
     | 不产生 Legacy RAS 记录
     | 不产生 FMU 记录

8.2 可纠正 vs 不可纠正错误
---------------------------

.. list-table::
   :header-rows: 1
   :widths: 25 38 37

   * - 类型
     | 默认行为（CE_EN=0）
     | CE_EN=1 后
   * - 可纠正错误（RAM CRC）
     | 作为不可纠正错误处理
     | 作为可纠正错误处理
   * - 不可纠正错误
     | 作为不可纠正错误处理
     | 不受 CE_EN 影响

**关键**：无论 ``CE_EN`` 设置如何，可纠正 RAM 的错误**始终被纠正**
（失效并重新获取）。``CE_EN`` 只控制 FMU 如何**分类报告**错误，
不影响实际纠正行为。
