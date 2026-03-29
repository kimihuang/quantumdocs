UCIe 规范 2.0 文档结构概览
====================================

本文档基于 UCIe (Universal Chiplet Interconnect Express) Specification 2.0 进行结构化梳理，
参考文档：``docs/pdf/ucie/UCIE_SPECIFICATION_2.0.md``

UCIe 是一种开放的多协议芯片间互连标准，用于连接同一封装内的多个裸芯（Die），
支持 PCIe、CXL、Streaming 等多种协议，旨在实现裸芯架构的生态系统。

.. contents::
   :depth: 3
   :local:


引言 (Chapter 1.0 Introduction)
-------------------------------

本章提供 UCIe 架构的概述。

UCIe 组件架构
~~~~~~~~~~~~~~

UCIe 采用分层协议架构，包含三个主要组件（参考 1.1 节）：

**协议层 (Protocol Layer)**
   - 支持协议：PCIe、CXL、Streaming、Management Transport
   - 协议层可与应用相关，UCIe 规范提供了在 UCIe 链路上传输 CXL 或 PCIe 协议的示例

**裸芯间适配器 (Die-to-Die Adapter, D2D Adapter)**
   - 协调协议层和物理层确保数据成功传输
   - 提供低延迟、优化的数据通路
   - 实现多协议复用时的 ARB/MUX 功能
   - 提供 CRC 和重试机制（当原始误码率 > 1e-27 时）

**物理层 (Physical Layer, PHY)**
   - 三个子组件：PHY 逻辑、边带/全局控制、电气/模拟前端(AFE)
   - 负责链路训练、通道修复、加扰/解扰、时钟转发等

接口定义
~~~~~~~~

UCIe 定义了两个关键接口（参考 1.1.4 节）：

- **RDI (Raw Die-to-Die Interface)**：物理层与 D2D 适配器之间的接口
- **FDI (Flit-Aware Die-to-Die Interface)**：D2D 适配器与协议层之间的接口

UCIe 配置类型
~~~~~~~~~~~~~

**单模块配置 (Single Module Configuration)**（参考 1.2.1 节）：

- Advanced Package：x64 或 x32 数据接口
- Standard Package：x16 或 x8 数据接口

**多模块配置 (Multi-module Configurations)**（参考 1.2.2 节）：

- 支持双模块和四模块配置
- 共享适配器时，模块必须以相同数据速率和宽度运行

**仅边带配置 (Sideband-only Configurations)**（参考 1.2.3 节）：

- 用于测试或可管理性目的
- 支持 1、2 或 4 个仅边带端口

三种封装技术
~~~~~~~~~~~~

**Standard Package（标准封装）**
   - 低成本、长距离互连
   - 使用有机封装/基板上的走线
   - 支持速度：4-32 GT/s
   - 凸点间距：110-135um

**Advanced Package（高级封装）**
   - 性能优化、短距离互连
   - 使用硅桥或硅中介层
   - 支持速度：4-32 GT/s
   - 凸点间距：25-55um

**UCIe-3D（三维堆叠）**
   - 使用二维阵列互连凸点进行裸芯间垂直数据传输
   - 支持速度：最高 4 GT/s
   - 凸点间距：<10um（优化）或 10-25um（功能）

关键性能指标
~~~~~~~~~~~~

**2D/2.5D 封装性能目标**（参考表 1-4）：

.. list-table::
   :header-rows: 1
   :widths: 30 25 25 20

   * - 指标
     - Advanced Package (x64)
     - Standard Package
     - 备注
   * - 边缘带宽密度 (32 GT/s)
     - 1317 GB/s/mm
     - 224 GB/s/mm
     - 45um/110um 凸点间距
   * - 能效 (0.7V)
     - 0.5-0.6 pJ/bit
     - 0.5-1.25 pJ/bit
     - 含所有适配器和 PHY 电路
   * - 延迟目标
     - ≤2ns
     - ≤2ns
     - 基于 16 GT/s

**UCIe-3D 性能目标**（参考表 1-5）：

.. list-table::
   :header-rows: 1
   :widths: 40 30 30

   * - 指标
     - UCIe-3D
     - 备注
   * - 带宽密度 (4 GT/s)
     - 4000 GB/s/mm²
     - 9um 凸点间距
   * - 能效 (0.65V)
     - 0.05 pJ/bit
     - 含所有 Tx/Rx 电路
   * - 延迟目标
     - ≤125 ps
     - Tx + Rx 延迟

UCIe 重定时器 (Retimers)
~~~~~~~~~~~~~~~~~~~~~~~~~

UCIe 重定时器用于扩展通道距离，实现不同封装间裸芯的互连（参考 1.3 节）。

**主要职责**：

- 跨离封装互连的可靠 Flit 传输
- 与远端重定时器伙伴解析链路和协议参数
- 解析适配器链路状态机的链路状态
- 流控和背压处理


协议层 (Chapter 2.0 Protocol Layer)
-----------------------------------

UCIe 映射 PCIe、CXL 以及 Streaming 协议（参考 2.0 节）。

支持的协议映射
~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - 协议
     - 说明
   * - PCIe Flit Mode
     - PCIe 基本规范定义的 Flit Mode
   * - PCIe Non-Flit Mode
     - 使用 CXL.io 68B Flit Format 作为传输机制
   * - CXL 68B Flit Mode
     - CXL 规范定义的 68B Flit 模式
   * - CXL 256B Flit Mode
     - CXL 规范定义的 256B Flit 模式
   * - Streaming Protocol
     - 用户定义协议的通用传输模式
   * - Management Transport
     - 管理网络上的端到端媒体无关协议

Flit 格式类型
~~~~~~~~~~~~~

UCIe 支持多种 Flit 格式以适应不同需求（参考 2.1-2.5 节）：

**格式 1：Raw Format（原始格式）**
   - 所有字节由协议层填充
   - 用于 UCIe 重定时器传输 PCIe 协议
   - 重试、CRC 和 FEC 由协议层处理

**格式 2：68B Flit Format**
   - 用于 PCIe non-Flit mode 和 CXL.io
   - 支持标准 PCIe/CXL 数据包传输

**格式 3：Standard 256B End Header Flit Format**
   - PCIe Flit Mode 支持时强制要求
   - 与仅支持标准 PCIe Flit 格式的厂商互操作

**格式 4：Standard 256B Start Header Flit Format**
   - 用于 CXL.cachemem 或 Streaming 协议

**格式 5：Latency-Optimized 256B without Optional Bytes**
   - 延迟优化的 256B 格式

**格式 6：Latency-Optimized 256B with Optional Bytes**
   - 支持可选字节的延迟优化格式


裸芯间适配器 (Chapter 3.0 Die-to-Die Adapter)
---------------------------------------------

D2D 适配器协调协议层和物理层确保成功数据传输（参考 3.0 节）。

核心功能
~~~~~~~~

- **栈复用 (Stack Multiplexing)**：支持多协议动态复用
- **链路初始化**：三阶段初始化流程
- **参数交换**：与远端链路伙伴协商能力参数
- **CRC 计算**：基于 CRC-16 或 CRC-24
- **重试机制**：当原始 BER > 1e-27 时启用

链路初始化阶段
~~~~~~~~~~~~~~

链路初始化分为三个阶段（参考 3.2 节）：

**阶段 1-2**：由物理层完成

**阶段 3：适配器初始化**（参考 3.2.1 节）

   - Part 1：确定本地能力
   - Part 2：与远端链路伙伴进行参数交换
   - Part 3：FDI 建立连接

操作格式决策
~~~~~~~~~~~~

Flit 格式和协议的决策表参考 3.4 节，根据协商的能力和物理层状态决定。

电源管理链路状态
~~~~~~~~~~~~~~~~

支持的电源管理状态（参考 3.6 节）：

- **L1**：轻量级低功耗状态
- **L2**：深度低功耗状态


逻辑物理层 (Chapter 4.0 Logical Physical Layer)
-----------------------------------------------

数据和边带传输流程
~~~~~~~~~~~~~~~~~~

**字节到通道映射**（参考 4.1.1 节）：

- x64 接口：64 字节数据映射到 64 个通道
- x32 接口：32 字节数据映射到 32 个通道
- x16 接口：16 字节数据映射到 16 个通道

**有效帧 (Valid Framing)**（参考 4.1.2 节）：

- 用于数据有效标记和流控
- 支持重定时器的特殊帧规则

通道反转 (Lane Reversal)
~~~~~~~~~~~~~~~~~~~~~~~~

支持通道 ID 重映射以处理物理连接差异（参考 4.2 节）。

互连冗余重映射
~~~~~~~~~~~~~~

**数据通道修复**（参考 4.3.1 节）：

- Advanced Package 提供额外的修复通道
- 支持单通道和双通道修复
- 支持带通道反转的修复

**时钟和跟踪通道重映射**（参考 4.3.4 节）：

- 时钟通道修复机制
- 跟踪通道修复机制

链路初始化和训练
~~~~~~~~~~~~~~~~

**链路训练状态机 (LTSM)** 状态（参考 4.5.3 节）：

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - 状态
     - 功能描述
   * - RESET
     - 复位状态
   * - SBINIT
     - 边带初始化
   * - MBINIT
     - 主带初始化和修复
   * - MBTRAIN
     - 主带训练
   * - LINKINIT
     - 链路初始化
   * - ACTIVE
     - 活动状态
   * - PHYRETRAIN
     - PHY 重训练
   * - TRAINERROR
     - 训练错误
   * - L1/L2
     - 电源管理状态

**MBINIT 子状态**（参考 4.5.3.3 节）：

- PARAM：参数交换
- CAL：校准
- REPAIRCLK：时钟修复
- REPAIRVAL：有效信号修复
- REVERSALMB：主带反转
- REPAIRMB：主带修复

多模块链路
~~~~~~~~~~

多模块配置的初始化和同步机制（参考 4.7 节）。

运行时重校准
~~~~~~~~~~~~

支持运行期间的链路重校准以维持链路健康（参考 4.6 节）。


电气层 (Chapter 5.0 Electrical Layer)
-------------------------------------

本章定义 2D 和 2.5D 封装的电气规范。

互操作性
~~~~~~~~

**数据速率**（参考 5.1.1 节）：

- 支持 4、8、12、16、24、32 GT/s

**参考时钟 (REFCLK)**（参考 5.1.2 节）：

- 支持公共参考时钟架构

发送器规范
~~~~~~~~~~

**驱动器拓扑**（参考 5.3.1 节）：

- 可配置输出阻抗
- 支持 de-emphasis 均衡

**电气参数**（参考 5.3.2 节）：

.. list-table::
   :header-rows: 1
   :widths: 30 35 35

   * - 参数
     - Advanced Package
     - Standard Package
   * - TX 电压摆幅
     - 0.4V 或 0.85V
     - 0.4V 或 0.85V
   * - 均衡 (24/32 GT/s)
     - 支持 de-emphasis
     - 支持 de-emphasis

接收器规范
~~~~~~~~~~

**接收端接**（参考 5.4.2 节）：

- 支持可配置端接电阻
- 不同 TX 摆幅对应不同端接配置

**均衡**（参考 5.4.3 节）：

- 24/32 GT/s 支持接收器均衡（CTLE）

凸点布局 (Bump Map)
~~~~~~~~~~~~~~~~~~~

**Advanced Package x64 模块**（参考 5.7.2.2 节）：

- 支持 8、10、16 列配置
- 包含数据、时钟、有效、跟踪和边带通道
- 额外修复通道

**Advanced Package x32 模块**（参考 5.7.2.3 节）：

- x64 的一半带宽
- 与 x64 模块可互操作

**Standard Package x16 模块**（参考 5.7.3.1 节）：

- 16 通道配置
- 无额外修复通道

**Standard Package x8 模块**（参考 5.7.3.2 节）：

- 仅用于单模块配置
- 主要用于预绑定测试

边带信号
~~~~~~~~

边带电气参数（参考 5.13 节）：

- 固定时钟频率：800 MHz
- 必须在辅助电源域


UCIe-3D (Chapter 6.0)
---------------------

概述
~~~~

UCIe-3D 是针对三维堆叠封装的互连规范（参考 6.1 节）。

**特性总结**（参考 6.2 节）：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 特性
     - 说明
   * - 最高速度
     - 4 GT/s
   * - 凸点间距
     - <10um（优化）/ 10-25um（功能）
   * - 通道类型
     - 3D 垂直
   * - 原始 BER
     - 1e-27

模块和凸点布局
~~~~~~~~~~~~~~

UCIe-3D 模块采用 bundle 结构（参考 6.4.3 节）：

- Tx bundle：包含数据、时钟、电源和地
- Rx bundle：包含数据、时钟、电源和地
- x70 模块标准配置

修复策略
~~~~~~~~

Bundle 级别的修复机制（参考 6.4.4 节）。

时序预算
~~~~~~~~

Dtx 和 Drx 规格范围定义（参考 6.4.1 节）。

ESD 和能效
~~~~~~~~~~

ESD 保护和能效设计考量（参考 6.4.2 节）。


边带 (Chapter 7.0 Sideband)
---------------------------

协议规范
~~~~~~~~

**数据包类型**（参考 7.1.1 节）：

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - 类型
     - 用途
   * - 寄存器访问包
     - 远程寄存器读写
   * - 无数据消息
     - 链路管理、电源管理等
   * - 带数据负载消息
     - 传输数据负载

数据包格式
~~~~~~~~~~

**寄存器访问请求**（参考 7.1.2.1 节）：

- 地址、长度、字节使能等字段

**寄存器访问完成**（参考 7.1.2.1 节）：

- 完成状态、数据等字段

**管理端口消息 (MPM)**（参考 7.1.2.4-7.1.2.5 节）：

- 带数据 MPM
- 无数据 MPM

流控和数据完整性
~~~~~~~~~~~~~~~~

**FDI 和 RDI 上的流控**（参考 7.1.3.1 节）

**UCIe 边带链路上的流控**（参考 7.1.3.2 节）

**端到端流控和前向进度**（参考 7.1.3.3 节）


系统架构 (Chapter 8.0 System Architecture)
------------------------------------------

UCIe 可管理性
~~~~~~~~~~~~~

**概述**（参考 8.1.1 节）：

UCIe 可管理性架构是可选机制，用于管理基于 UCIe 的 SiP（系统级封装）。

**管理实体类型**（参考 8.1.2 节）：

- **Management Director**：发现、配置和协调整体管理
- **Management Element**：执行一个或多个管理功能
- **Management Bridge**：桥接 SiP 内的管理网络到外部网络
- **Management Port**：促进裸芯间的管理通信

管理传输协议
~~~~~~~~~~~~

**MTP (Management Transport Packet)**（参考 8.1.3 节）：

- 流量类别和包排序要求
- 包长度限制
- 管理协议定义

**管理网络 ID**（参考 8.1.3.2 节）：

- 网络标识符格式
- 路由机制

封装架构
~~~~~~~~

**边带管理路径**（参考 8.2.3.1 节）：

- 协商阶段
- 初始化阶段

**主带管理路径**（参考 8.2.3.2 节）：

- 协商阶段
- 初始化阶段

UCIe 调试和测试架构 (UDA)
~~~~~~~~~~~~~~~~~~~~~~~~~

**概述**（参考 8.3.1 节）：

- **DFx Management Hub (DMH)**：DFx 实体，提供枚举/全局控制/状态
- **DFx Management Spoke (DMS)**：实现特定测试/调试功能

**支持的协议**：

- UCIe Memory Access Protocol (UMAP)
- Vendor-defined Test and Debug Protocol

**测试场景**：

- Sort/Pre-bond 裸芯测试（参考 8.3.2 节）
- SiP 级裸芯测试（参考 8.3.3 节）
- 系统调试（参考 8.3.4 节）


配置和参数 (Chapter 9.0 Configuration and Parameters)
-----------------------------------------------------

软件视图
~~~~~~~~

**UCIe 链路的软件发现**（参考 9.2 节）：

- 通过 DVSEC 能力发现
- 寄存器位置和访问机制

寄存器定义
~~~~~~~~~~

**UCIe Link DVSEC**（参考 9.5.1 节）：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 寄存器
     - 功能
   * - Capability Descriptor
     - 能力描述
   * - UCIe Link Capability
     - 链路能力
   * - UCIe Link Control
     - 链路控制
   * - UCIe Link Status
     - 链路状态
   * - Link Event Notification Control
     - 链路事件通知控制
   * - Error Notification Control
     - 错误通知控制
   * - Register Locator
     - 寄存器定位器

**D2D/PHY 寄存器块**（参考 9.5.3 节）：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 寄存器组
     - 功能
   * - Error Status/Mask Registers
     - 错误状态/屏蔽寄存器
   * - PHY Capability/Control/Status
     - PHY 能力/控制/状态
   * - Training Setup
     - 训练设置
   * - Current Lane Map
     - 当前通道映射
   * - Error Log
     - 错误日志
   * - Runtime Link Test
     - 运行时链路测试
   * - Mainband Data Repair
     - 主带数据修复

**UCIe Link Health Monitor (UHM)**（参考 9.5.3.40 节）：

- 眼图裕量监控
- 链路健康状态

测试/合规寄存器块
~~~~~~~~~~~~~~~~~

**适配器测试/合规**（参考 9.5.4.4 节）：

- 合规控制
- Flit 注入控制
- 链路状态注入控制
- 重试注入控制

**PHY 测试/合规**（参考 9.5.4.5 节）：

- 物理层合规控制
- 合规状态

UCIe 早期发现表 (UEDT)
~~~~~~~~~~~~~~~~~~~~~~

用于系统启动时的早期发现机制（参考 9.8 节）。


接口定义 (Chapter 10.0 Interface Definitions)
---------------------------------------------

RDI (Raw Die-to-Die Interface)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**信号列表**（参考 10-1 表）：

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - 信号类别
     - 说明
   * - 数据信号
     - lp_data, pl_data
   * - 有效信号
     - lp_valid, pl_valid
   * - 时钟信号
     - pl_clk
   * - 状态信号
     - lp_state_req, pl_state_sts
   * - 唤醒信号
     - lp_wake_req, pl_wake_ack

**状态机**（参考 10.1.5 节）：

- Reset → Active → PM (L1/L2) → Retrain → LinkReset → LinkError → Disabled

**时钟要求**（参考 10.1.2 节）：

- 动态时钟门控支持
- lp_wake_req/pl_wake_ack 握手机制

FDI (Flit-Aware Die-to-Die Interface)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**信号列表**（参考 10-3 表）：

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - 信号类别
     - 说明
   * - Flit 数据
     - pl_flit, lp_flit
   * - Flit 控制
     - pl_flit_valid, lp_flit_valid
   * - CRC 状态
     - pl_crc_error
   * - 取消信号
     - pl_flit_cancel

**状态机**（参考 10.2.6 节）：

与 RDI 类似的状态转换流程

**DLLP 传输规则**（参考 10.2.4 节）：

- 256B Flit Mode 的 DLLP 传输规则

通用规则
~~~~~~~~

**字节映射**（参考 10.3.1 节）：

- 64B 数据通路到 256B Flit 的映射
- 128B 数据通路到 256B Flit 的映射

**Stallreq/Ack 机制**（参考 10.3.2 节）：

用于流控和背压处理


合规性 (Chapter 11.0 Compliance)
--------------------------------

协议层合规
~~~~~~~~~~

协议层合规性测试要求（参考 11.1 节）。

适配器合规
~~~~~~~~~~

D2D 适配器合规性测试要求（参考 11.2 节）。

PHY 合规
~~~~~~~~

物理层合规性测试要求（参考 11.3 节）。


附录
----

**附录 A：CXL/PCIe 寄存器对 UCIe 的适用性**
   - CXL 寄存器映射
   - PCIe 寄存器映射

**附录 B：AIB 互操作性**
   - AIB 信号映射
   - 初始化流程
   - 凸点布局


术语表
------

关键术语定义（参考 Table 1）：

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - 术语
     - 定义
   * - Chiplet
     - 包含定义良好功能子集的集成电路裸芯，设计用于与同一封装中的其他裸芯组合
   * - Module
     - UCIe 主数据通路上的一组通道，Advanced Package 为 64/32 通道，Standard Package 为 16 通道
   * - Sideband
     - 用于参数交换、调试/合规寄存器访问以及链路训练和管理的协调
   * - Mainband
     - UCIe 主数据通路，包含转发时钟、数据有效引脚和 N 个数据通道
   * - FDI
     - Flit 感知裸芯间接口，D2D 适配器与协议层之间的接口
   * - RDI
     - 原始裸芯间接口，物理层与 D2D 适配器之间的接口


修订历史
--------

.. list-table::
   :header-rows: 1
   :widths: 15 20 65

   * - 版本
     - 日期
     - 主要变更
   * - 2.0
     - 2024-08-06
     - 添加 UCIe-3D；系统架构（可管理性、MTP 封装、UDA）；di/dt 风险缓解
   * - 1.1
     - 2023-07-10
     - Streaming Flit 格式能力；增强多协议复用；x32 Advanced Package；链路健康监控；合规硬件能力
   * - 1.0
     - 2022-02-17
     - 初始发布
