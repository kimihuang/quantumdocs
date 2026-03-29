====================================
LPDDR5 SDRAM 规范知识点梳理
====================================

本文档基于 JEDEC LPDDR5 规范（JC-42.6-1854.99A）梳理 LPDDR5 SDRAM 的核心概念、
架构设计和关键技术特性。

.. contents::
   :depth: 3
   :local:


LPDDR5 概述
================

1 规范范围与定位
--------------------

LPDDR5（Low Power Double Data Rate 5）是 JEDEC 定义的低功耗同步动态随机存取存储器标准，
适用于移动设备和嵌入式系统。该标准继承并融合了 DDR2/3/4 和 LPDDR/2/3/4 的设计理念。

- **设备密度范围**：2 Gb 至 32 Gb
- **数据位宽**：x16（单通道 16 位）或 x8（单通道 8 位，Byte Mode）
- **通道数**：1 通道（Channel）
- **参考规范**：DDR2 (JESD79-2), DDR3 (JESD79-3), DDR4 (JESD79-4),
  LPDDR (JESD209), LPDDR2 (JESD209-2), LPDDR3 (JESD209-3), LPDDR4 (JESD209-4)

2 核心设计理念
--------------------

LPDDR5 的核心设计目标是**高性能与低功耗的平衡**：

- 采用命令时钟（CK）与数据时钟（WCK）分离的架构，WCK 仅在需要传输数据时运行以节省功耗
- 数据传输使用双倍数据速率（DDR）信令，命令/地址总线也采用 DDR 信令
- 支持动态电压频率调节（DVFS），可根据负载动态调整工作频率和电压
- 数据总线支持 0.5V VDDQ 标称电压下的高速操作

3 支持的密度规格
--------------------

LPDDR5 支持以下密度等级：

- 2Gb: 2,147,483,648 bits
- 4Gb: 4,294,967,296 bits
- 8Gb: 8,589,934,592 bits
- 12Gb: 12,884,901,888 bits
- 16Gb: 17,179,869,184 bits
- 24Gb: 25,769,803,776 bits
- 32Gb: 34,359,738,368 bits


信号引脚定义
================

1 核心信号引脚
--------------------

.. list-table::
   :header-rows: 1
   :widths: 20 10 70

   * - 信号名称
     - 类型
     - 描述
   * - CK_t, CK_c
     - Input
     - **差分命令时钟**：所有 DDR 命令/地址输入在 CK_t 和 CK_c 的交叉点（上升沿和下降沿）被采样。SDR 输入信号（如 CS）仅在 CK_t 上升沿被采样。
   * - CS
     - Input
     - **片选信号（SDR）**：CS 是命令码的一部分，在 CK_t 上升沿被采样。在 Power-Down 或 Deep Sleep 模式下变为异步信号。
   * - CA[6:0]
     - Input
     - **命令/地址输入（DDR）**：7 位双向信号，用于传输命令、地址、Bank 地址、配置信息和训练数据。
   * - DQ[15:0]
     - I/O
     - **双向数据总线**：16 位数据总线（x16 模式）或 8 位数据总线（x8 模式）。
   * - WCK[1:0]_t, WCK[1:0]_c
     - Input
     - **差分写数据时钟**：用于 WRITE 数据捕获和 READ 数据输出的差分时钟。每个字节对应一组 WCK。
   * - RDQS[1:0]_t, RDQS[1:0]_c
     - I/O
     - **读数据选通信号**：READ 操作中用于选通数据的差分输出时钟。RDQS_t 在写操作（Link Protection 启用时）还用作 Parity 引脚。
   * - DMI[1:0]
     - I/O
     - **数据掩码反转**：多功能引脚，可配置为 DM（Data Mask）、DBI（Data Bus Inversion）和 Parity。
   * - ZQ
     - Reference
     - **校准参考**：用于校准输出驱动强度和端接电阻的参考引脚，需通过 240 ohm +/-1% 电阻连接到 VDDQ。每颗 Die 一个 ZQ pad。
   * - RESET_n
     - Input
     - **异步复位信号**：低电平有效时复位 Die。

2 电源引脚
--------------------

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 电源名称
     - 描述
   * - VDDQ
     - I/O 电源，标称 0.5V（高速模式），支持 0.3V-0.5V（低速模式）
   * - VDD1
     - 内部逻辑电源
   * - VDD2H
     - 高频工作电压（DVFSQ 高电压）
   * - VDD2L
     - 低频工作电压（DVFSQ 低电压）
   * - VSS
     - 接地参考

3 DMI 引脚多功能定义
--------------------------

DMI 引脚根据配置不同具有多种功能：

.. list-table::
   :header-rows: 1
   :widths: 15 15 15 15 15 15 15

   * - RDQS 模式
     - DBI
     - Link Protection
     - Write 时 DMI 功能
     - Read 时 DMI 功能
   * - SE (MR20[1:0]=01)
     - No
     - Disabled
     - DM
     - N/A
   * - SE (MR20[1:0]=01)
     - Yes
     - Disabled
     - DMI (DM+DBI)
     - DBI
   * - SE (MR20[1:0]=11)
     - No
     - Disabled
     - DM
     - N/A
   * - Diff (MR20[1:0]=10)
     - No
     - Disabled
     - DM
     - N/A
   * - Diff (MR20[1:0]=10)
     - Yes
     - Disabled
     - DMI
     - DBI
   * - Any
     - Any
     - **Enabled**
     - DM / Parity
     - Parity


Bank 架构
=============

LPDDR5 支持三种可选择的 Bank 架构，由 **MR3 OP[4:3]** 控制：

1 BG 模式（4 Banks / 4 Bank Groups）
-----------------------------------------

- **全称**：Bank Group Mode
- **结构**：4 个 Bank + 4 个 Bank Group，共 4 个 Bank
- **支持速率**：大于 3200 Mbps（>3200 Mbps）
- **突发长度**：BL16 或 BL32
- **地址映射**：

  - CA0 -> BA0 (Bank Address)
  - CA1 -> BA1
  - CA2 -> BG0 (Bank Group Address)
  - CA3 -> BG1

- **特点**：Bank Group 架构允许不同 Bank Group 之间的访问可以更快地背靠背执行，
  从而提高总线利用率。BG 模式是高速操作（>3200 Mbps）的推荐模式。

2 8B 模式（8 Banks）
-------------------------

- **全称**：8 Bank Mode
- **结构**：8 个 Bank，无 Bank Group
- **支持速率**：全速率范围
- **突发长度**：仅 BL32
- **地址映射**：

  - CA0 -> BA0
  - CA1 -> BA1
  - CA2 -> BA2
  - CA3 -> B4 (READ) / V (ACT-1, PRE, MWR, WR)

- **特点**：8 Bank 模式的页面大小与 BG/16B 模式不同（是其两倍），
  因此 Bank 地址的物理含义不同。

3 16B 模式（16 Banks）
--------------------------

- **全称**：16 Bank Mode
- **结构**：16 个 Bank，无 Bank Group
- **支持速率**：等于或小于 3200 Mbps（<=3200 Mbps）
- **突发长度**：BL16 或 BL32
- **地址映射**：

  - CA0 -> BA0
  - CA1 -> BA1
  - CA2 -> BA2
  - CA3 -> BA3

- **特点**：提供更多的 Bank 数量，适合需要大量并行访问的场景。

4 Bank 架构切换
--------------------

- 默认 Bank 架构为 **16B 模式**
- 16B 和 BG 模式具有**相同的页面大小**，可通过 FSP（Frequency Set Point）流程切换
- 8B 模式与 16B/BG 模式**页面大小不同**，必须通过 Reset 来切换
- Bank 架构切换应在 Idle 状态下执行

5 数据预取（Prefetch）与突发长度
--------------------------------------

- **BG 模式**：支持 16n 和 32n 预取
- **8B 模式**：仅支持 32n 预取
- **16B 模式**：支持 16n 和 32n 预取
- BL32 交错（interleaved）传输仅在 WCK:CK = 4:1 时支持


WCK 时钟架构
=================

1 WCK 与 CK 的关系
-----------------------

LPDDR5 采用**命令时钟（CK）与数据时钟（WCK）分离**的架构：

- **CK（Command Clock）**：命令时钟，用于采样所有命令和地址信号
- **WCK（Write Clock）**：数据时钟，用于数据写入捕获和数据读出输出
- **WCK:CK 比率**：可通过 **MR18 OP[7]** 选择 **2:1** 或 **4:1**

.. list-table::
   :header-rows: 1
   :widths: 30 30 40

   * - CK 频率
     - WCK 频率 (2:1)
     - WCK 频率 (4:1)
   * - 200 MHz
     - 400 MHz
     - 800 MHz
   * - 400 MHz
     - 800 MHz
     - 1600 MHz
   * - 533 MHz
     - 1066 MHz
     - 2134 MHz
   * - 800 MHz
     - 1600 MHz
     - 3200 Mbps
   * - 1067 MHz
     - 2134 MHz
     - 4267 Mbps
   * - 1600 MHz
     - 3200 MHz
     - 6400 Mbps

2 WCK2CK 同步（WCK2CK Synchronization）
---------------------------------------------

由于 WCK 通常在不传输数据时停止（以节省功耗），当 WCK 需要重新启动时，
必须与 CK 进行相位同步：

- **同步目的**：确保 WCK 的分频相位与 CK 对齐，使读写数据的时序正确
- **同步方式**：通过 CAS 命令中的 WCK2CK Sync 位（WS_RD, WS_WR, WS_FS）触发
- **同步过程**：

  1. 控制器发送 CAS 命令（携带 WS_RD/WS_WR/WS_FS 位）
  2. DRAM 内部进行 WCK 与 CK 的相位对齐
  3. 同步完成后，可以发送 Read/Write 命令

- **WS_FAST 模式**：CAS(WS_FAST=1) 提供最小延迟的同步操作，
  适用于对延迟敏感的场景
- **Rank-to-Rank 同步**：多 Rank 系统中，不同 Rank 的 WCK2CK 同步可以并行进行

3 WCK Always On 模式
--------------------------

- WCK 可以配置为始终运行（Always On），避免频繁同步带来的延迟开销
- 适用于需要频繁进行读/写切换的场景
- 支持 Enhanced WCK Always On 模式，提供更精细的功耗控制
- 通过 CAS(WS_OFF) 命令关闭 WCK Always On 模式

4 单端模式（Single-Ended Mode）
------------------------------------

CK、WCK 和 RDQS 信号支持在差分模式和单端模式之间切换：

- **差分模式**：高速操作时的默认模式
- **单端模式**：低速操作时使用，减少功耗
- **切换规则**：CK、WCK 和 RDQS 必须在同一 FSP 下统一切换
- **切换时序**：需要额外的 MRW AC 时间（附加 nCK）


初始化与训练
=================

1 上电初始化流程
----------------------

LPDDR5 的初始化流程包括以下关键步骤：

1. **电压斜坡（Voltage Ramp）**：确保 VDDQ、VDD1、VDD2H、VDD2L 等电源稳定
2. **RESET_n 释放**：等待 RESET_n 信号释放，完成设备复位
3. **ZQ 校准初始化**：通过 ZQ 引脚校准驱动强度和端接电阻
4. **模式寄存器配置**：通过 MRW 命令配置工作模式
5. **训练序列**：执行各种信号完整性训练

2 ZQ 校准（ZQ Calibration）
--------------------------------

ZQ 校准用于校准 DRAM 的输出驱动强度和端接电阻：

**两种校准模式**：

- **Background Calibration（后台校准）**：
  - DRAM 在后台自动执行校准
  - 使用 MPC 命令或模式寄存器控制
  - 持续运行，校准结果自动更新
  - 支持 ZQ Stop 功能
  - 适合温度变化较大的场景

- **Command-Based Calibration（命令校准）**：
  - 由控制器通过命令触发校准
  - 需要显式发送 ZQ Latch 命令来保存校准结果
  - 适合精确控制校准时机的场景

**ZQ 相关功能**：

- **ZQ Stop**：停止后台 ZQ 校准，节省功耗
- **ZQ Reset**：重置校准电路
- **ZQ 电阻共享**：多 Die 封装中可共享 ZQ 电阻
- **ZQ 外部电阻**：必须使用 240 ohm +/-1% 精度电阻

3 命令总线训练（Command Bus Training）
--------------------------------------------

命令总线训练确保 CA 信号能够被 DRAM 正确采样：

- **Training Mode1**：基本训练模式，使用 DQ 输出来回传训练结果
- **Training Mode2**：高级训练模式，支持 VREF(CA) 值的在线更新
- **三个物理 Mode Register**：MR16 OP[3:0] 选择不同的物理寄存器，
  每个 FSP 有独立的训练参数
- **支持 DVFSQ**：训练可以在 DVFSQ 模式下执行

**训练流程**：

1. 进入命令总线训练模式
2. 发送训练模式（CA Pattern）
3. 从 DQ 引脚读取训练结果
4. 根据结果调整发送时序
5. 退出训练模式

4 CA VREF 训练
-------------------

- 训练命令/地址总线的参考电压（VREF_CA）
- 确保所有 CA 信号都能在正确的阈值被采样
- 训练结果存储在 **MR12** 中

5 DQ VREF 训练
-------------------

- 训练数据总线的参考电压（VREF_DQ）
- 确保数据信号能够被正确接收
- 训练结果存储在 **MR14**（DQ[7:0]）和 **MR15**（DQ[15:8]）中

6 WCK2CK Leveling
-----------------------

- 调整 WCK 与 CK 之间的相位对齐
- 在 LPDDR4 中称为 Write Leveling
- 使用 Leveling 模式输出对齐结果
- 确保写入数据的采样窗口正确

7 占空比调节器（DCA - Duty Cycle Adjuster）
------------------------------------------------

- 用于补偿 WCK 的占空比失真
- 调节范围：约 +/-5% 的占空比调节
- DCA 代码变化会影响 DQ 输出和 RDQS 时序
- 有助于在高速操作下维持信号完整性

8 占空比监视器（DCM - Duty Cycle Monitor）
------------------------------------------------

- 监测 WCK 的实际占空比
- 可由控制器读取占空比监测结果
- 帮助控制器判断是否需要调整 DCA

9 Read DQ Calibration (RDC)
--------------------------------

- 用于校准读取数据路径上的时序偏移
- 通过发送 RDC 命令，DRAM 输出已知的校准模式
- 控制器根据返回的模式调整接收时序
- 支持 DMI Output Behavior Mode 1 和 Mode 2

10 WCK-DQ 训练（WCK2DQ Training）
---------------------------------------

- 调整 WCK 到 DQ 的时序关系
- **Write FIFO Training**：调整写入数据的 FIFO 时序
- **Read FIFO Training**：调整读取数据的 FIFO 时序
- **WCK-RDQS_t/Parity Training**：训练读数据选通信号的时序
- 支持 FIFO Pointer Reset 和同步操作

11 RDQS Toggle 模式
------------------------

- RDQS 信号在非 Read 操作期间持续翻转
- 用于训练期间监视 RDQS 信号质量
- 进入/退出需要满足特定的时序约束

12 Enhanced RDQS Training 模式
------------------------------------

- 增强的 RDQS 训练模式，提供更精确的训练能力
- 支持在 Read 操作期间进行训练
- 训练结果可以在线更新


状态机
===========

1 简化状态图
----------------

LPDDR5 的核心操作状态包括：

.. code-block:: text

    Reset --> Idle (初始化完成后)
    Idle --> Active (ACT 命令激活一行)
    Active --> Read/Write (READ/WRITE 命令)
    Active --> Precharge (PRE 命令关闭行)
    Precharge --> Idle
    Idle --> Power-Down (进入低功耗)
    Power-Down --> Idle (退出低功耗)
    Idle --> Self-Refresh (进入自刷新)
    Self-Refresh --> Idle (退出自刷新)
    Idle/Active --> Deep Sleep (进入深度休眠)
    Deep Sleep --> Idle/Self-Refresh (退出深度休眠)

2 关键状态转换时序
------------------------

- **tRCD**：ACT 到 READ/WRITE 的最小延迟
- **tRAS**：ACT 到 PRE 的最小时间
- **tRP**：PRE 命令执行时间
- **tRC**：同一 Bank 连续 ACT 之间的最小间隔
- **tFAW**：4 个 Bank 窗口内最大 ACT 次数的限制
- **tRRD**：不同 Bank/Bank Group 之间 ACT 的最小间隔
- **tWTR**：WRITE 到 READ 的最小间隔
- **tRTW**：READ 到 WRITE 的最小间隔


命令系统
=============

1 命令结构
--------------

LPDDR5 的命令通过 **CS + CA[6:0]** 编码：

- **CS（SDR）**：在 CK_t 上升沿采样，作为命令码的一部分
- **CA[6:0]（DDR）**：在 CK 的上升沿和下降沿都被采样，每个 CK 周期传输 14 位信息
- 大多数命令为 **1 nCK** 持续时间

2 核心命令
--------------

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 命令
     - 描述
   * - ACT (Activate)
     - 激活指定 Bank 的指定行，打开一个页面
   * - PRE (Precharge)
     - 关闭指定 Bank（或所有 Bank）的活跃行
   * - READ
     - 从打开的行中读取数据
   * - WRITE
     - 向打开的行中写入数据
   * - MWR (Masked Write)
     - 带掩码的写入操作，可选择性更新字节
   * - REF (Refresh)
     - 执行刷新操作，保持数据完整性
   * - SRE (Self Refresh Entry)
     - 进入自刷新模式
   * - SRX (Self Refresh Exit)
     - 退出自刷新模式
   * - PDE (Power Down Entry)
     - 进入低功耗模式
   * - PDX (Power Down Exit)
     - 退出低功耗模式
   * - MRW (Mode Register Write)
     - 写入模式寄存器配置
   * - MRR (Mode Register Read)
     - 读取模式寄存器内容
   * - MPC (Multi-Purpose Command)
     - 多功能命令（ZQ 校准等）
   * - RESET
     - 通过 RESET_n 引脚异步复位

3 CAS 命令操作数
----------------------

CAS 命令携带多种操作数位，实现多种功能：

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - 操作数位
     - 功能描述
   * - WS_FS (WCK2CK Sync)
     - 触发 WCK2CK 同步
   * - WS_FAST
     - 最小延迟的 WCK2CK 同步
   * - WS_RD (Write Sync for Read)
     - 为 Read 操作准备 WCK 同步
   * - WS_WR (Write Sync for Write)
     - 为 Write 操作准备 WCK 同步
   * - WS_OFF
     - 关闭 WCK Always On 模式
   * - WCKSUS
     - WCK 暂停
   * - DC0-DC3 (Data Copy)
     - 数据复制操作
   * - WRX (Write X)
     - 写零操作
   * - B3
     - Burst Length 3 操作


读写操作
=============

1 Read 操作
---------------

- Read 操作前必须完成 WCK2CK 同步
- 数据通过 DQ 总线和 RDQS 信号输出
- 支持 BL16 和 BL32 突发长度
- 不同 Bank Group 的背靠背 Read 可以无额外延迟
- Read Latency（RL）由 MR2 OP[3:0] 和 MR2 OP[6:4] 编程设定

**Read Preamble 和 Postamble**：

- **tRPRE**：Read Preamble 时间，RDQS 信号在数据输出前开始翻转
- **tRPST**：Read Postamble 时间，RDQS 信号在数据输出后继续翻转

2 Write 操作
----------------

- Write 操作前必须完成 WCK2CK 同步
- 数据通过 DQ 总线由 WCK 信号采样
- Write Latency（WL）由 MR2 OP[3:0] 和 MR2 OP[6:4] 编程设定
- Write Recovery Time（nWR）确保写入数据正确存储到存储单元

**Write Preamble 和 Postamble**：

- **tWPRE**：Write Preamble 时间
- **tWPST**：Write Postamble 时间

3 Masked Write
------------------

- 允许在写入操作中选择性地屏蔽部分字节
- 通过 DMI 信号指定哪些字节需要被更新
- 适用于部分数据更新的场景，减少读-改-写操作
- 受 tCCDMW 时序约束

4 Auto-Precharge
----------------------

- 在 Read 或 Write 操作中可以自动执行 Precharge
- 减少控制器的命令开销
- 需要注意 Write 到 Read 带 Auto-Precharge 的延迟约束

5 Data Bus Inversion (DBI)
-------------------------------

- 通过对数据总线进行按位取反来优化信号完整性
- 当原始数据中"0"位多于"1"位时，对数据进行取反以减少翻转
- 通过 MR20 OP[0] 使能
- DMI 引脚在 Read 时输出 DBI 状态指示

6 有效突发长度（BL/n）
---------------------------

LPDDR5 引入了 BL/n 的概念，支持更灵活的突发控制：

- **BL/n**：标准突发长度
- **BL/n_min**：最小有效突发长度
- **BL/n_max**：最大有效突发长度
- 在背靠背操作中，BL/n_min 允许更紧凑的命令间隔


刷新机制
=============

1 刷新命令（Refresh）
--------------------------

- DRAM 存储单元中的电荷会随时间泄漏，需要定期刷新
- **All-Bank Refresh**：刷新所有 Bank
- **Per-Bank Refresh**：刷新单个 Bank，其他 Bank 可继续操作
- 刷新间隔 tREFI 和刷新周期 tRFC 由器件密度和工作温度决定

2 刷新管理
---------------

- 控制器需要跟踪刷新计数，确保所有行都能在规定时间内被刷新
- 刷新命令可以被**推迟（Postponing）**或**提前（Pulling-in）**
- 最大推迟次数和最大提前次数由规范限制

3 优化刷新（Optimized Refresh）
------------------------------------

- 通过 MR25 OP[7] 控制
- 在特定条件下可以跳过不需要的刷新操作
- 进一步降低刷新功耗

4 自刷新（Self Refresh）
----------------------------

- DRAM 自己管理刷新操作
- 控制器无需发送刷新命令
- 适用于系统长时间空闲的场景
- 支持在自刷新期间进入 Power-Down 以进一步降低功耗

5 部分数组自刷新（PASR）
------------------------------

- 可以选择性地只刷新部分存储阵列
- 通过模式寄存器配置 Segment Masking
- 适用于不需要保留全部数据的低功耗场景

6 部分数组刷新控制（PARC）
--------------------------------

- 控制需要刷新的数组区域范围
- 与 PASR 类似但粒度不同

7 刷新管理命令（Refresh Management - RFM）
-----------------------------------------------

- LPDDR5 引入了 RFM 命令用于精细化的刷新管理
- 支持 Sub-Bank 级别的刷新
- 可以对特定 Sub-Bank 区域执行刷新操作


电源管理
=============

1 Power Down
-----------------

- 关闭 DRAM 的大部分内部时钟和逻辑以降低功耗
- 保留当前状态（包括打开的行）
- 支持从 Active 状态、Idle 状态进入
- 退出 Power Down 后可以继续之前的操作

2 Deep Sleep Mode
------------------------

- 比 Power Down 更深度的低功耗状态
- 关闭更多内部电路
- 支持从 Idle 和 Self Refresh 状态进入
- 退出时需要更长的恢复时间
- 通过 CS 信号异步唤醒

3 动态电压频率调节（DVFS）
--------------------------------

LPDDR5 支持两种 DVFS 模式：

**DVFSC 模式**（Clock DVFS）：

- 仅改变 CK 和 WCK 频率，电压保持不变
- 频率切换更快速
- 通过 FSP（Frequency Set Point）机制实现
- 支持最多 4 个频率设置点

**DVFSQ 模式**（Quick DVFS）：

- 同时改变频率和 VDDQ 电压
- 低频时使用 VDD2L（低电压），高频时使用 VDD2H（高电压）
- 提供更大的功耗节省
- 需要等待 VDDQ 电压斜坡稳定

**频率设置点（FSP）**：

- LPDDR5 支持多个频率设置点
- 每个 FSP 有独立的模式寄存器配置
- FSP 切换通过 CAS 命令触发
- 切换时序由 tFC 参数定义

4 VREF Current Generator (VRCG)
--------------------------------------

- 为 VREF(CA) 和 VREF(DQ) 生成参考电压
- 支持高电流模式（High Current Mode）和普通模式
- 在 VREF 更新时需要切换到高电流模式
- VRCG 的使能/禁用有时序要求


端接（ODT）
=================

1 On-Die Termination 概述
-------------------------------

LPDDR5 在 CA 总线、DQ 总线、WCK 和 CS 信号上都支持 On-Die Termination（ODT）：

- **CA Bus ODT**：命令/地址总线的端接电阻
- **DQ Bus ODT**：数据总线的端接电阻
- **WCK ODT**：写时钟的端接电阻
- **CS ODT**：片选信号的端接电阻
- ODT 电阻值通过 ZQ 校准确定，典型值为 RZQ (240 ohm) 的 1/3、1/4、1/6 等

2 非目标 DRAM ODT（NT-ODT）
----------------------------------

- 多 Rank 系统中，非目标 Rank 的 DQ 总线端接
- 减少 Rank 切换时的总线反射
- 支持同步和异步 ODT 模式
- 分为 Write NT-ODT 和 Read NT-ODT

**NT-ODT 控制**：

- 通过 MR10、MR11、MR12 等寄存器配置
- 由 MR0 OP[0] 控制时序模式
- 支持两种时序模式（MR0 OP[0]=0B 和 MR0 OP[0]=1B）

**Unified NT-ODT 行为**：

- 统一了不同模式下的 NT-ODT 行为
- 支持 DMI、RDQS_t、RDQS_c 的独立 ODT 控制
- 适用于 DVFSC 和 DVFSQ 模式

3 Asynchronous ODT
-------------------------

- ODT 的开启/关闭可以异步执行
- 由 ODTLon 和 ODTLoff 寄存器控制延迟
- 适用于高速操作场景


模式寄存器（Mode Register）
=================================

1 概述
-----------

LPDDR5 定义了大量的模式寄存器（MR0 至 MR57+），用于配置器件的各种参数：

- 每个 FSP（频率设置点）有**三个物理寄存器**，通过 MR16 OP[3:0] 选择
- 通过 MRW 命令写入，MRR 命令读取
- 地址通过 MA[7:0] 指定，数据通过 OP[7:0] 传输

2 关键模式寄存器
----------------------

.. list-table::
   :header-rows: 1
   :widths: 15 15 70

   * - 寄存器
     - 地址
     - 主要功能
   * - MR0
     - 00H
     - Burst Length、RL/WL 设置、NT-ODT 时序模式
   * - MR1
     - 01H
     - Driver Strength、ODT 设置
   * - MR2
     - 02H
     - Read Latency、Write Latency、nRTP、nWR
   * - MR3
     - 03H
     - Bank 架构选择（BG/8B/16B）、DBI 设置
   * - MR4
     - 04H
     - 温度传感器设置、刷新参数
   * - MR10
     - 0AH
     - DQ VREF 设置、RDQS 控制
   * - MR11
     - 0BH
     - DQ ODT 设置、NT-ODT 控制
   * - MR12
     - 0CH
     - CA VREF 设置
   * - MR14
     - 0EH
     - DQ[7:0] VREF 设置
   * - MR15
     - 0FH
     - DQ[15:8] VREF 设置
   * - MR16
     - 10H
     - 物理寄存器选择（FSP 映射）
   * - MR18
     - 12H
     - WCK:CK 比率设置、Custom Gear Down
   * - MR20
     - 14H
     - RDQS 模式、DBI 控制
   * - MR25
     - 19H
     - 优化刷新控制、VDD2L 参数
   * - MR27
     - 1BH
     - RFM（Refresh Management）配置
   * - MR42
     - 2AH
     - PPR（Post Package Repair）控制
   * - MR45
     - 2DH
     - Link ECC 配置
   * - MR57
     - 39H
     - Sub-Bank 寻址和 RFM 控制


特殊功能
=============

1 Data Copy 功能
----------------------

LPDDR5 支持数据复制功能，用于内部数据搬运：

**Write Data Copy**：

- 将已写入 DRAM 的数据复制到相邻地址
- 通过 CAS 命令的 DC0-DC3 操作数触发
- DMI 引脚输出命中/未命中标志
- 适用于数据初始化和填充场景

**Read Data Copy**：

- 读取已存储的数据并复制到相邻地址
- 支持与 DBI 功能配合使用

2 Write X 操作
--------------------

- 快速将指定区域填充为零
- 通过 CAS(WRX) 命令触发
- 适用于内存清零场景
- 可与普通的 Write 操作混合使用

3 Post Package Repair (PPR)
---------------------------------

- **封装后修复**功能，用于修复已出厂的存储器件中的缺陷行
- 通过 MR42 配置 PPR 参数
- 支持 **Guard Key 保护**，防止误操作
- 修复过程：标记缺陷行 -> 修复行地址 -> 修复确认

4 Decision Feedback Equalization (DFE)
--------------------------------------------

- **判决反馈均衡**技术
- 用于高速操作下的信号质量改善
- 通过 DFE 预驱动要求确保信号完整性

5 Link ECC
----------------

LPDDR5 引入了链路级 ECC 保护：

- **Write Link ECC**：在写入数据时附加 ECC 校验位
- **Read Link ECC**：在读取数据时进行 ECC 校验
- 通过 MR45 配置
- ECC 校验矩阵覆盖 DQ 数据和 DMI 信号
- 提供端到端的数据完整性保护

6 温度传感器
------------------

- LPDDR5 内置温度传感器
- 通过 MR4 OP[2:0] 配置
- 控制器可以通过 MPC 命令读取温度值
- 用于刷新速率调整（温度补偿）和过热保护

7 tWCK2DQ Interval Oscillator
--------------------------------------

- 内部振荡器，用于测量 WCK 到 DQ 的时序间隔
- 分为 WCK2DQI（Input）和 WCK2DQO（Output）两种
- 可通过 MPC 命令启动/停止
- 测量结果用于训练和校准


AC/DC 规范
=================

1 速度等级
---------------

LPDDR5 定义了多个速度等级：

.. list-table::
   :header-rows: 1
   :widths: 20 20 20 40

   * - 速度等级
     - CK 频率
     - 数据速率
     - 说明
   * - 6400
     - 1600 MHz
     - 6400 Mbps
     - 最高速率（WCK:CK=4:1）
   * - 5500
     - 1375 MHz
     - 5500 Mbps
   * - 4800
     - 1200 MHz
     - 4800 Mbps
   * - 4267
     - 1067 MHz
     - 4267 Mbps
   * - 3200
     - 800 MHz
     - 3200 Mbps
   * - 其他
     - 更低
     - 更低
     - 低速操作

2 DC 操作条件
-------------------

- **VDDQ**：标称 0.5V（高速），0.3V-0.5V（低速）
- **VDD1**：内部逻辑电源
- **VDD2H/VDD2L**：高频/低频工作电源
- **VDDQ 容差**：在允许范围内有规定的容差要求

3 AC 参数
--------------

- **tCK**：CK 时钟周期
- **tWCK**：WCK 时钟周期
- **tJIT(per)**：周期抖动
- **tJIT(cc)**：周期间抖动
- **tWCK2DQI**：WCK 到 DQ Input 的时序
- **tWCK2DQO**：WCK 到 DQ Output 的时序

4 信号电平
----------------

**LVSTL（Low Voltage Swing Terminated Logic）**：

- 适用于 DQ、WCK、RDQS 等高速信号
- 标称信号摆幅：VDDQ/2（约 250mV）
- 需要端接电阻（ODT）

**1.05V High Speed LVCMOS**：

- 适用于 CK、CA、CS 等命令/地址信号
- 输入电平符合 LVCMOS 标准

5 温度范围
---------------

- **商业级**：0°C 至 +85°C（典型）, -30°C 至 +85°C（扩展）
- **工业级**：-40°C 至 +105°C
- **汽车级**：-40°C 至 +105°C（有额外规范）


封装与引脚
=================

1 封装类型
---------------

- **315-ball POP Package**：叠层封装
- **0.80mm x 0.70mm**：小型化封装尺寸
- 支持 x16 单通道和 x32 双通道配置

2 Byte Mode 设备
-----------------------

- x16 设备可以通过 Byte Mode 作为两个 x8 设备使用
- 每个 Byte 有独立的 DQ[7:0]、WCK、RDQS、DMI 信号
- 支持独立的 MRW/MRR 操作

3 Rank 分配
---------------

- 单通道双 Rank 封装支持两个 Rank 共享同一个 CA 总线
- 每个 Rank 有独立的 CS、DQ、WCK、RDQS 和 DMI 信号
- Rank 切换需要满足 tRANK-RANK 时序要求


关键时序参数汇总
=======================

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 参数
     - 描述
   * - tRCD
     - ACT 到 READ/WRITE 的最小延迟
   * - tRAS
     - ACT 到 PRE 的最小时间（行活跃时间）
   * - tRP
     - PRE 命令执行时间
   * - tRC
     - 同一 Bank 连续 ACT 之间的最小间隔（tRCD + tRP 理论最小值，实际 tRC >= tRAS + tRP）
   * - tRRD
     - 不同 Bank/Bank Group 之间 ACT 的最小间隔
   * - tFAW
     - 4 个 Bank 窗口内最大 ACT 次数的时间限制
   * - tWTR
     - WRITE 到 READ 的最小间隔
   * - tRTW
     - READ 到 WRITE 的最小间隔
   * - tCCD
     - 同一 Bank Group 内连续 CAS 命令的最小间隔
   * - tCCDMW
     - Masked Write 相关时序约束
   * - tRFC
     - 刷新周期时间
   * - tREFI
     - 刷新间隔时间
   * - tXS
     - Self Refresh 退出时间
   * - tXSR
     - Self Refresh 退出恢复时间
   * - tCKESR
     - Self Refresh 中的 CK 停止时间
   * - tWCK2CK
     - WCK2CK 同步时间
   * - tFC
     - FSP 切换时间
   * - tXP
     - Power Down 退出时间
   * - tXPDLL
     - DLL 锁定时间（Power Down 后）
   * - tVREFCA
     - VREF(CA) 更新时间
   * - tVREFDQ
     - VREF(DQ) 更新时间


命令时序约束
===================

1 基本 RAS 时序
----------------------

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - 操作组合
     - 时序要求
   * - ACT -> ACT (Same Bank)
     - tRC
   * - ACT -> ACT (Different Bank in Same BG)
     - tRRD_S (Same BG)
   * - ACT -> ACT (Different Bank in Different BG)
     - tRRD_L (Different BG)
   * - ACT -> READ/WRITE (Same Bank)
     - tRCD
   * - ACT -> PRE (Same Bank)
     - tRAS
   * - READ/WRITE -> PRE (Same Bank)
     - tRTP + tRP 或 tRBTP
   * - PRE -> ACT (Same Bank)
     - tRP

2 基本 CAS 时序
----------------------

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - 操作组合
     - 时序要求
   * - READ -> READ (Same Bank, Same BG)
     - BL/n + tCCD_S
   * - READ -> READ (Different BG)
     - BL/n + tCCD_L
   * - WRITE -> WRITE (Same Bank, Same BG)
     - BL/n + tCCD_S
   * - WRITE -> WRITE (Different BG)
     - BL/n + tCCD_L
   * - READ -> WRITE
     - RL + BL/n + tRTW
   * - WRITE -> READ
     - WL + BL/n + tWTR


多 Rank 操作
===================

1 Rank-to-Rank 时序
-------------------------

- **tRANK-RANK**：不同 Rank 之间的最小命令间隔
- Rank 切换需要考虑 WCK2CK 同步时间
- NT-ODT 的开关时序影响 Rank 切换效率

2 Rank 切换优化
----------------------

- 支持广播 WCK2CK 同步（CAS-WS_FS Broadcast ON）
- 支持 Enhanced WCK Always On 模式减少同步开销
- NT-ODT Unified 模式简化了多 Rank ODT 管理


术语表
===========

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - 缩写
     - 全称/含义
   * - ACT
     - Activate（激活命令）
   * - BG
     - Bank Group（Bank 组）
   * - BL
     - Burst Length（突发长度）
   * - CA
     - Command/Address（命令/地址）
   * - CK
     - Clock（命令时钟）
   * - CKR
     - WCK to CK Ratio（WCK:CK 比率）
   * - DBI
     - Data Bus Inversion（数据总线反转）
   * - DCA
     - Duty Cycle Adjuster（占空比调节器）
   * - DCM
     - Duty Cycle Monitor（占空比监视器）
   * - DFE
     - Decision Feedback Equalization（判决反馈均衡）
   * - DM
     - Data Mask（数据掩码）
   * - DMI
     - Data Mask Inversion（数据掩码反转）
   * - DQ
     - Data Queue（数据总线）
   * - DVFS
     - Dynamic Voltage and Frequency Scaling（动态电压频率调节）
   * - DVFSC
     - DVFS Clock（时钟 DVFS）
   * - DVFSQ
     - DVFS Quick（快速 DVFS）
   * - ECC
     - Error Correcting Code（纠错码）
   * - FSP
     - Frequency Set Point（频率设置点）
   * - LVCMOS
     - Low Voltage CMOS（低压 CMOS）
   * - LVSTL
     - Low Voltage Swing Terminated Logic（低电压摆幅端接逻辑）
   * - MRW/MRR
     - Mode Register Write/Read（模式寄存器写/读）
   * - MPC
     - Multi-Purpose Command（多功能命令）
   * - NT-ODT
     - Non-Target On-Die Termination（非目标片上端接）
   * - ODT
     - On-Die Termination（片上端接）
   * - PASR
     - Partial Array Self Refresh（部分阵列自刷新）
   * - PDE/PDX
     - Power Down Entry/Exit（电源下降进入/退出）
   * - PPR
     - Post Package Repair（封装后修复）
   * - PRE
     - Precharge（预充电）
   * - RDQS
     - Read Data Strobe（读数据选通）
   * - RFM
     - Refresh Management（刷新管理）
   * - RL
     - Read Latency（读延迟）
   * - SDR/DDR
     - Single Data Rate / Double Data Rate（单/双数据速率）
   * - SRE/SRX
     - Self Refresh Entry/Exit（自刷新进入/退出）
   * - VREF
     - Reference Voltage（参考电压）
   * - VRCG
     - VREF Current Generator（VREF 电流发生器）
   * - WL
     - Write Latency（写延迟）
   * - WCK
     - Write Clock（写时钟）
   * - ZQ
     - ZQ Calibration（ZQ 校准）
