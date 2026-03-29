.. _lpddr5_dvfs:

=====================================
LPDDR5 DVFS（动态电压频率调整）机制
=====================================

LPDDR5的DVFS（Dynamic Voltage and Frequency Scaling）包含两种模式，旨在降低LPDRAM的能耗：

- **DVFSC (DVFS Core)** - 核心电压动态调整
- **DVFSQ (DVFS VDDQ)** - I/O电压动态调整

.. _dvfsc_mode:

DVFSC Mode (DVFS Core - 核心电压动态调整)
-----------------------------------------

基本原理
~~~~~~~~

DVFSC模式允许LPDRAM内部电路在两个电压轨之间切换：

- **VDD2H** - 高电压轨
- **VDD2L** - 低电压轨

当内存控制器命令FSP（频率设定点）改变时，LPDRAM可以在内部将某些电路从一个电压轨切换到另一个。

配置方法
~~~~~~~~

- 通过 ``MR19 OP[1:0]=01B`` 启用DVFSC模式
- DVFSC模式切换**只能在FSP-OP切换时进行**
- 直接编程MR19 OP[1:0]来改变当前工作点的DVFSC模式是**非法的**

关键特性
~~~~~~~~

- 启用DVFSC时，某些操作需要更长时间完成
- 延迟和AC时序参数根据当前DVFSC状态有独立的规格表
- 参见规范中的DVFSC时序表

不使用DVFSC的系统要求
~~~~~~~~~~~~~~~~~~~~~~

对于不实现DVFSC的系统：

1. 必须向VDD2H和VDD2L轨提供相同的VDD2H电压
2. 必须设置 ``MR13 OP[7]=1`` 和 ``MR19 OP[1:0]=00B``

DVFSC时序图
~~~~~~~~~~~

DVFSC High-to-Low Transition (VDD2H → VDD2L)::

    t0  t1  t2  ta0 tb0 tb1 tb2 tc0 ...
    |---|---|---|---|---|---|---|---|
    CK_t/CK_c: [toggle]
    Command:    CAS WR WR Des Des Des MRW1 MRW1 MRW2 MRW2 ...
                [VRCG=On]          [VRCG=Off]
    VDD2H:      [High level then transition to Low]
    VDD2L:      [Low level]

    tFC: Frequency change time
    tVRCG_DISABLE: VRCG disable time

DVFSC Low-to-High Transition (VDD2L → VDD2H)::

    t0  t1  t2  ta0 tb0 tb1 tb2 tc0 ...
    |---|---|---|---|---|---|---|---|
    CK_t/CK_c: [toggle]
    Command:    CAS WR WR Des Des Des MRW1 MRW1 MRW2 MRW2 ...
                [VRCG=On]          [VRCG=Off]
    VDD2H:      [High level]
    VDD2L:      [Low level then transition to High]

.. _dvfsq_mode:

DVFSQ Mode (DVFS VDDQ - I/O电压动态调整)
-----------------------------------------

基本原理
~~~~~~~~

DVFSQ模式允许在操作期间（包括读写事务）调整VDDQ电压：

- **高电压** - VDDQ = 0.5V 标称值
- **低电压** - VDDQ = 0.3V 标称值

DVFSQ High-to-Low Transition (高到低转换)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

转换序列如下：

1. 在VDDQ=0.5V标称值下高速运行
2. FSP从高频切换到适合非ODT操作的低频，并使能VRCG
3. 等待tFC（暂停流量）
4. 发送MRW MR16 – 设置VRCG=0
5. 等待tVRCG_DISABLE（暂停流量）
6. 发送MRW MR28 – 设置ZQ Reset OP[0]=1，设置默认ZQ强度以满足VDDQ=0.3V操作
7. 等待tZQRESET（无需ZQ latch）
8. 发送MRW MR28 – 设置ZQ Stop=1以禁用后台校准
9. 等待tZQSTOP
10. 进入DVFSQ（将VDDQ降低到0.5V标称值以下）
11. 在降低的VDDQ下继续操作

DVFSQ Low-to-High Transition (低到高转换)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**方法一：不使用VRCG的转换序列**

1. 在VDDQ<0.5V标称值下低速运行
2. 将VDDQ升至0.5V标称值
3. 发送MRW MR28 – 设置ZQ Stop=0以启用后台校准
4. 等待tZQCALx
5. FSP从低频切换到高频，并使能VRCG
6. 等待tFC（暂停流量）
7. 发送MRW 16以禁用VRCG
8. 等待tVRCG_DISABLE（暂停流量）
9. 在VDDQ=0.5V标称值下开始高速操作

**方法二：使用VRCG的转换序列（更快）**

在VDDQ转换前启用VRCG，允许VREF电路在更高的VDDQ压摆率下准确跟踪：

1. 在VDDQ<0.5V标称值下低速运行
2. 使能VRCG
3. 等待tVRCG_enable（暂停流量）
4. 将VDDQ升至0.5V标称值
5. 发送MRW MR28 – 设置ZQ Stop=0以启用后台校准
6. 等待tZQCALx
7. FSP从低频切换到高频
8. 等待tFC（暂停流量）
9. 禁用VRCG
10. 等待tVRCG_disable（继续暂停流量）
11. 在VDDQ=0.5V标称值下开始高速操作

DVFSQ操作指南
~~~~~~~~~~~~~~

**如果在VDDQ电压调整期间暂停操作：**

1. 将设备置于power-down模式或保持CS为低，直到电压调整完成
2. 在VDDQ=0.5V标称值下高速操作前建议重新校准
3. 应在新电平操作前进行FSP切换到适当的新设置

**如果在VDDQ电压调整期间继续操作：**

1. 应在适合最低VDDQ电平的设置和速度下操作
2. 强烈建议禁用ODT操作（VDDQ低于0.5V标称值时通常需要）
3. VDDQ电压调整应等于或慢于规定的限制
4. 建议设置VRCG使能以确保内部Vref跟踪变化的VDDQ电平
5. 在VDDQ=0.5V标称值下高速操作前建议重新校准
6. 应在更高速操作前进行FSP切换到适当的新设置

.. _fsp_mechanism:

Frequency Set Point (FSP - 频率设定点)
--------------------------------------

核心机制
~~~~~~~~

FSP允许LPDDR5在**三个不同工作频率**之间切换，通过复制所有CA总线模式寄存器参数，避免在切换过程中出现未训练状态导致通信丢失。

控制位定义
~~~~~~~~~~

=================  =================  ==========================================
控制位             寄存器位置          功能描述
=================  =================  ==========================================
FSP-WR             MR16 OP[1:0]       选择要读/写的物理寄存器组
                                      00B/01B/10B 对应三组寄存器
FSP-OP             MR16 OP[3:2]       选择当前操作使用的物理寄存器组
                                      00B/01B/10B 对应三组寄存器
=================  =================  ==========================================

FSP-WR和FSP-OP功能完全独立操作。

三重寄存器参数
~~~~~~~~~~~~~~

以下参数有三份物理寄存器，由FSP-WR和FSP-OP控制：

.. list-table:: FSP控制的模式寄存器
   :widths: 10 15 40 10
   :header-rows: 1

   * - MR#
     - Operand
     - Function
     - Note
   * - MR1
     - OP[3]
     - CK mode
     -
   * - MR1
     - OP[7:4]
     - WL (Write Latency)
     -
   * - MR2
     - OP[3:0]
     - RL (Read latency) and nRBTP
     -
   * - MR2
     - OP[7:4]
     - nWR (Write-Recovery)
     -
   * - MR3
     - OP[2:0]
     - PDDS (Pull-Down Drive Strength)
     -
   * - MR3
     - OP[4:3]
     - BK/BG ORG
     -
   * - MR3
     - OP[5]
     - WLS (Write Latency Set)
     -
   * - MR3
     - OP[6]
     - DBI-RD
     -
   * - MR3
     - OP[7]
     - DBI-WR
     -
   * - MR10
     - OP[0]
     - RDQS Post-amble mode
     -
   * - MR10
     - OP[3:2]
     - WCK PST
     -
   * - MR10
     - OP[5:4]
     - RDQS PRE
     -
   * - MR10
     - OP[7:6]
     - RDQS PST
     -
   * - MR11
     - OP[2:0]
     - DQ ODT
     -
   * - MR11
     - OP[3]
     - NT-ODT Enable
     -
   * - MR11
     - OP[6:4]
     - CA ODT
     -
   * - MR12
     - OP[6:0]
     - VREF(CA) Setting
     -
   * - MR14
     - OP[6:0]
     - VREF(DQ[7:0]) Setting
     -
   * - MR15
     - OP[6:0]
     - VREF(DQ[15:8]) Setting
     -
   * - MR17
     - OP[2:0]
     - SoC ODT
     -
   * - MR17
     - OP[3]
     - ODTD-CK
     -
   * - MR17
     - OP[4]
     - ODTD-CS
     -
   * - MR17
     - OP[5]
     - ODTD-CA
     -
   * - MR18
     - OP[2:0]
     - WCK ODT
     -
   * - MR18
     - OP[3]
     - WCK FM
     -
   * - MR18
     - OP[4]
     - WCK ON
     -
   * - MR18
     - OP[7]
     - CKR (WCK to CK ratio)
     -
   * - MR19
     - OP[1:0]
     - DVFSC
     -
   * - MR19
     - OP[3:2]
     - DVFSQ
     -
   * - MR19
     - OP[4]
     - WCK2DQ OSC FM
     -
   * - MR20
     - OP[1:0]
     - RDQS
     -
   * - MR20
     - OP[3:2]
     - WCK mode
     -
   * - MR22
     - OP[5:4]
     - WECC (Write link ECC)
     -
   * - MR22
     - OP[7:6]
     - RECC (Read link ECC)
     -
   * - MR24
     - OP[2:0]
     - DFEQL
     -
   * - MR24
     - OP[6:4]
     - DFEQU
     -
   * - MR30
     - OP[3:0]
     - DCAL
     -
   * - MR30
     - OP[7:4]
     - DCAU
     -
   * - MR41
     - OP[7:5]
     - NT DQ ODT
     -

FSP切换工作流程
~~~~~~~~~~~~~~~

1. 设置FSP-WR选择目标寄存器组（例如：01B选择FSP-OP[1]）
2. 写入新频率下的所有参数（WL、RL、ODT、VREF等）
3. 设置FSP-OP切换到新配置
4. 在tFC时间内完成切换

FSP时序参数
~~~~~~~~~~~

==============  ==============  ========================================
参数            时间            描述
==============  ==============  ========================================
tFC             -               Frequency change time
tCKFSPE         -               Frequency set point entry time
tCKFSPX         -               Frequency set point exit time
tVRCG_DISABLE   -               VRCG disable time
tVRCG_ENABLE    -               VRCG enable time
==============  ==============  ========================================

.. _dvfs_summary:

DVFS模式总结
------------

.. list-table:: DVFS模式对比
   :widths: 15 25 30 30
   :header-rows: 1

   * - 模式
     - 调整对象
     - 主要用途
     - 关键寄存器
   * - DVFSC
     - VDD2H/VDD2L
       (核心电压)
     - 降低核心电路功耗
     - MR19 OP[1:0]
   * - DVFSQ
     - VDDQ
       (I/O电压)
     - 降低I/O功耗，
       支持低功耗待机
     - MR19 OP[3:2]

DVFS优势
--------

1. **降低功耗** - 通过动态调整电压实现节能
2. **无缝切换** - FSP机制确保在频率/电压切换时不会丢失与DRAM的通信
3. **灵活配置** - 支持三个独立的工作频率点
4. **实时操作** - DVFSQ支持在读写事务期间调整电压

参考文档
--------

- LPDDR5 Specification (JEDEC Standard)
- LPDDR5 Mode Register Definition
