Linux 驱动代码阅读分析指南 — 以 arm_pmu.c 为例
==================================================

本文档以 ``drivers/perf/arm_pmu.c``（ARM PMU 驱动框架）为例，
给出阅读 Linux 内核驱动代码时应理解的关键检查点。
阅读完驱动代码后，若能回答以下所有问题，即表明已理解该驱动的核心设计。

源文件: ``src/linux-6.1-rc6/drivers/perf/arm_pmu.c``

1. 驱动概述
===========

**应回答的问题:**

- 这个驱动的功能是什么？
- 属于哪个内核子系统？（drivers/perf → perf 事件子系统）
- 它是平台驱动、字符设备驱动，还是其他类型？（PMU 驱动，通过 perf 框架注册）
- 对应的硬件规范是什么？（ARM PMU 架构，ARMv8 PMUv3）
- 与用户空间的接口是什么？（``perf stat``, ``perf record`` 等工具）

**arm_pmu.c 的回答:**

- 功能：ARM 处理器性能监控单元（PMU）的通用驱动框架
- 子系统：``drivers/perf``，Linux perf 事件子系统
- 类型：不是传统平台驱动，而是通过 ``perf_pmu_register()`` 向 perf 框架注册
- 硬件：ARM PMU（Performance Monitoring Unit），每个 CPU 核心一个
- 用户接口：通过 ``/sys/devices/<pmu>/`` sysfs 和 perf 命令行工具

2. 头文件依赖与核心数据结构
================================

**应回答的问题:**

- 包含了哪些关键头文件？各自提供什么能力？
- 驱动定义了哪些核心数据结构？
- 与硬件寄存器的对应关系是什么？

**arm_pmu.c 的回答:**

核心头文件:

.. list-table::
   :header-rows: 1
   :widths: 40 60

   * - 头文件
     - 作用
   * - ``linux/perf/arm_pmu.h``
     - ``struct arm_pmu`` 定义（驱动的核心抽象）
   * - ``linux/cpumask.h``
     - CPU 亲和性掩码（异构系统中的 PMU CPU 绑定）
   * - ``linux/cpu_pm.h``
     - CPU 电源管理通知（休眠/唤醒时保存恢复 PMU 状态）
   * - ``linux/irq.h``
     - 中断请求/释放（PMU 溢出中断）
   * - ``linux/bitmap.h``
     - 位图操作（硬件计数器分配状态跟踪）

核心数据结构:

- ``struct arm_pmu`` — PMU 抽象，包含硬件操作函数指针和 percpu 事件存储
- ``struct pmu_hw_events`` — 每个 CPU 上的硬件事件集合和计数器分配状态
- ``struct pmu_irq_ops`` — 中断操作的抽象（IRQ vs NMI vs per-CPU IRQ）

.. code-block:: c

   struct pmu_irq_ops {
       void (*enable_pmuirq)(unsigned int irq);
       void (*disable_pmuirq)(unsigned int irq);
       void (*free_pmuirq)(unsigned int irq, int cpu, void __percpu *devid);
   };

3. 驱动初始化与注册流程
==========================

**应回答的问题:**

- 驱动的入口点在哪里？（``module_init``, ``subsys_initcall``, 平台 probe 等）
- 从入口到硬件可用，经过了哪些步骤？
- 驱动是如何被内核发现的？（设备树、ACPI、手动注册等）

**arm_pmu.c 的回答:**

入口点: ``subsys_initcall(arm_pmu_hp_init)``（:c:func:`arm_pmu_hp_init` 第 964 行）

注册流程:

.. code-block:: none

   subsys_initcall(arm_pmu_hp_init)
     └─ cpuhp_setup_state_multi(CPUHP_AP_PERF_ARM_STARTING)
          ├─ arm_perf_starting_cpu()   ← CPU 上线时调用
          │    ├─ pmu->reset()          ← 重置 PMU 硬件寄存器
          │    ├─ per_cpu(cpu_armpmu)   ← 绑定 CPU → PMU
          │    └─ enable_pmuirq()       ← 使能溢出中断
          └─ arm_perf_teardown_cpu()   ← CPU 下线时调用
               ├─ disable_pmuirq()
               └─ per_cpu(cpu_armpmu) = NULL

注意: ``arm_pmu.c`` 是**框架层**代码。具体的 PMU 驱动（如 ``armv8_pmu.c``）
负责调用 ``armpmu_alloc()`` → 设置回调 → ``armpmu_register()``:

.. code-block:: none

   具体驱动（如 armv8_pmu_probe）
     └─ armpmu_alloc()              ← 分配 arm_pmu + percpu hw_events
     └─ 设置回调函数指针:
          pmu->map_event            ← 事件 ID 映射
          pmu->get_event_idx        ← 硬件计数器分配
          pmu->enable / disable     ← 使能/禁用单个计数器
          pmu->start / stop         ← 使能/禁用整个 PMU
          pmu->read_counter / write_counter
          pmu->handle_irq           ← 溢出中断处理
          pmu->reset                ← 重置硬件
     └─ armpmu_register()           ← 注册到 perf 子系统
          └─ perf_pmu_register()     ← 创建 /sys/devices/<pmu>/

4. 操作函数集（ callbacks ）
=============================

**应回答的问题:**

- 驱动向内核框架注册了哪些操作函数？（open/read/write/ioctl/mmap 等）
- 每个函数的职责是什么？
- 函数之间的调用关系是什么？

**arm_pmu.c 的回答:**

通过 ``struct pmu`` 注册到 perf 框架（:c:func:`__armpmu_alloc` 第 879 行）:

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - pmu 回调
     - 对应函数
     - 职责
   * - ``event_init``
     - ``armpmu_event_init``
     - 验证事件合法性、映射事件 ID、检查分组约束
   * - ``add``
     - ``armpmu_add``
     - 分配硬件计数器、将事件绑定到计数器槽位
   * - ``del``
     - ``armpmu_del``
     - 停止事件、释放计数器槽位
   * - ``start``
     - ``armpmu_start``
     - 设置采样周期、使能计数器
   * - ``stop``
     - ``armpmu_stop``
     - 禁用计数器、更新事件计数
   * - ``read``
     - ``armpmu_read``
     - 读取当前计数器值并更新事件计数
   * - ``pmu_enable``
     - ``armpmu_enable``
     - 使能整个 PMU
   * - ``pmu_disable``
     - ``armpmu_disable``
     - 禁用整个 PMU
   * - ``filter_match``
     - ``armpmu_filter_match``
     - 异构系统中过滤不匹配的 CPU

关键内部函数:

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 函数
     - 职责
   * - ``armpmu_event_set_period``
     - 设置计数器溢出周期。将 ``-left`` 写入计数器，使其在 ``left`` 个事件后溢出
   * - ``armpmu_event_update``
     - 读取计数器当前值，计算与上次读数之间的 delta，累加到 ``event->count``
   * - ``armpmu_map_event``
     - 将 perf 抽象事件映射为硬件事件编码（raw/hardware/cache 三种类型）
   * - ``armpmu_dispatch_irq``
     - 中断分发，调用具体的 ``pmu->handle_irq``
   * - ``validate_group``
     - 验证事件组是否可以同时调度到硬件（计数器数量够不够）

5. 中断处理
============

**应回答的问题:**

- 使用什么类型的中断？（共享中断、per-CPU 中断、NMI？）
- 中断触发条件是什么？（计数器溢出）
- 中断处理流程是什么？

**arm_pmu.c 的回答:**

中断类型优先级（:c:func:`armpmu_request_irq` 第 626 行）:

.. code-block:: none

   优先尝试 NMI（不可屏蔽中断）
     ├─ 成功 → pmunmi_ops / percpu_pmunmi_ops
     └─ 失败 → 回退到普通 IRQ
                  ├─ per-CPU devid IRQ → percpu_pmuirq_ops
                  └─ 普通 IRQ → pmuirq_ops

四种 ``pmu_irq_ops`` 实现:

.. list-table::
   :header-rows: 1
   :widths: 30 25 25 20

   * - ops
     - enable
     - disable
     - 场景
   * - ``pmuirq_ops``
     - ``enable_irq``
     - ``disable_irq_nosync``
     - 非 per-CPU 的普通 IRQ
   * - ``pmunmi_ops``
     - ``enable_nmi``
     - ``disable_nmi_nosync``
     - 非 per-CPU 的 NMI
   * - ``percpu_pmuirq_ops``
     - ``enable_percpu_irq``
     - ``disable_percpu_irq``
     - per-CPU devid IRQ
   * - ``percpu_pmunmi_ops``
     - ``enable_percpu_nmi``
     - ``disable_percpu_nmi``
     - per-CPU NMI

中断处理链:

.. code-block:: none

   硬件计数器溢出
     → IRQ/NMI 触发
     → armpmu_dispatch_irq()       ← 顶层分发
          └─ pmu->handle_irq()     ← 具体驱动实现（armv8_pmu 等）
               └─ handle_irq() 遍历溢出计数器
                    ├─ armpmu_event_update()  ← 读计数器 delta
                    ├─ perf_event_overflow()  ← 通知 perf 框架
                    └─ armpmu_event_set_period() ← 重设周期

6. CPU 热插拔处理
==================

**应回答的问题:**

- CPU 上线/下线时，驱动做了什么？
- PMU 状态如何保存和恢复？

**arm_pmu.c 的回答:**

CPU hotplug（第 706-740 行）:

- **上线** ``arm_perf_starting_cpu``: 重置 PMU 寄存器 → 绑定 ``cpu_armpmu`` → 使能中断
- **下线** ``arm_perf_teardown_cpu``: 禁用中断 → 清除 ``cpu_armpmu`` 绑定

CPU 电源管理（第 742-833 行，``CONFIG_CPU_PM``）:

- **休眠进入** ``CPU_PM_ENTER``: 停止整个 PMU → 逐个停止事件并更新计数
- **休眠唤醒** ``CPU_PM_EXIT``: 重置 PMU → 逐个恢复事件并重载周期 → 启动 PMU
- **休眠失败** ``CPU_PM_ENTER_FAILED``: 同唤醒处理

.. note::

   PMU 硬件在 CPU 掉电后寄存器状态丢失，因此唤醒时必须完全重新初始化。
   这就是为什么 ``CPU_PM_EXIT`` 时先调用 ``pmu->reset()``。

7. 并发与锁
============

**应回答的问题:**

- 哪些数据需要并发保护？
- 使用了什么锁机制？
- 是否有 per-CPU 数据的无锁设计？

**arm_pmu.c 的回答:**

.. list-table::
   :header-rows: 1
   :widths: 40 20 40

   * - 保护对象
     - 锁类型
     - 说明
   * - ``pmu_hw_events.pmu_lock``
     - ``raw_spinlock_t``
     - 每个 CPU 一把，保护该 CPU 的事件计数器操作
   * - ``hw_perf_event.count``
     - ``local64_t``
     - 无锁原子操作，每个 CPU 读写自己的计数
   * - ``hw_perf_event.prev_count``
     - ``local64_t``
     - 使用 ``local64_cmpxchg`` 做 CAS 更新（:c:func:`armpmu_event_update` 第 251 行）
   * - ``cpu_armpmu`` / ``cpu_irq``
     - per-CPU 变量
     - 无锁，每个 CPU 只访问自己的

.. code-block:: c

   // armpmu_event_update() 中的无锁读-改-写:
   again:
       prev_raw_count = local64_read(&hwc->prev_count);
       new_raw_count = armpmu->read_counter(event);
       if (local64_cmpxchg(&hwc->prev_count, prev_raw_count,
                            new_raw_count) != prev_raw_count)
           goto again;  // CAS 失败则重试

8. 硬件资源管理
================

**应回答的问题:**

- 硬件资源是什么？（计数器数量、寄存器等）
- 资源如何分配和释放？
- 资源不足时的行为？

**arm_pmu.c 的回答:**

- 硬件资源：有限数量的硬件计数器（``pmu->num_events``）
- 分配：``pmu->get_event_idx(hw_events, event)`` — 由具体驱动实现，返回可用计数器索引
- 状态跟踪：``hw_events->used_mask`` bitmap 记录已分配的计数器
- 释放：``pmu->clear_event_idx(hw_events, event)`` — 清除 bitmap 中对应位
- 不足：``armpmu_add()`` 返回负值（如 ``-ENOSPC``），事件无法添加

9. Sysfs 接口
==============

**应回答的问题:**

- 在 ``/sys`` 下暴露了哪些属性？
- 用户空间如何使用这些属性？

**arm_pmu.c 的回答:**

.. code-block:: none

   /sys/devices/arm_pmu_<N>/
     ├── cpus           ← 支持的 CPU 列表（异构系统中至关重要）
     ├── type           ← perf 事件类型 ID
     ├── nr_counters    ← 硬件计数器数量（具体驱动提供）
     └── events/        ← 可用事件列表（具体驱动提供）

``cpus`` 属性的实现（第 566-571 行）:

.. code-block:: c

   static ssize_t cpus_show(struct device *dev,
                            struct device_attribute *attr, char *buf)
   {
       struct arm_pmu *armpmu = to_arm_pmu(dev_get_drvdata(dev));
       return cpumap_print_to_pagebuf(true, buf, &armpmu->supported_cpus);
   }

.. note::

   ``supported_cpus`` 在异构系统（如 big.LITTLE）中至关重要 — 不同微架构的
   CPU 有不同的 PMU，用户空间通过 ``cpus`` 属性判断事件应该在哪些 CPU 上计数。

10. 与具体驱动的接口关系
==========================

**应回答的问题:**

- 本文件是框架层还是具体驱动？
- 具体驱动需要实现哪些回调？
- 接口抽象的边界在哪里？

**arm_pmu.c 的回答:**

``arm_pmu.c`` 是**框架层**（framework），不直接操作硬件寄存器。
具体驱动（如 ``drivers/perf/armv8_pmu.c``）通过函数指针接入框架:

.. list-table::
   :header-rows: 1
   :widths: 35 65

   * - 回调函数
     - 职责（由具体驱动实现）
   * - ``map_event``
     - 将 perf 事件 ID 映射为硬件事件编码
   * - ``get_event_idx``
     - 从可用计数器中分配一个给该事件
   * - ``clear_event_idx``
     - 释放计数器
   * - ``enable`` / ``disable``
     - 使能/禁用单个硬件计数器
   * - ``start`` / ``stop``
     - 使能/禁用整个 PMU（全局控制）
   * - ``read_counter`` / ``write_counter``
     - 读写单个计数器的当前值
   * - ``handle_irq``
     - 溢出中断处理（识别哪个计数器溢出）
   * - ``reset``
     - 重置 PMU 到已知状态
   * - ``set_event_filter``
     - 设置事件过滤（内核态/用户态/特权态）

典型注册序列（具体驱动视角）:

.. code-block:: c

   // 1. 分配
   struct arm_pmu *pmu = armpmu_alloc();

   // 2. 设置硬件参数
   pmu->name = "armv8_pmuv3";
   pmu->num_events = <硬件计数器数量>;
   pmu->map_event = armv8pmu_map_event;

   // 3. 设置硬件操作回调
   pmu->enable      = armv8pmu_enable_event;
   pmu->disable     = armv8pmu_disable_event;
   pmu->read_counter = armv8pmu_read_counter;
   pmu->write_counter = armv8pmu_write_counter;
   pmu->handle_irq  = armv8pmu_handle_irq;
   pmu->reset       = armv8pmu_reset;

   // 4. 注册
   armpmu_register(pmu);

11. 事件生命周期
=================

**应回答的问题:**

- 从用户调用 ``perf stat -e cycles`` 到计数完成，经过了哪些代码路径？

**arm_pmu.c 的回答:**

.. code-block:: none

   用户空间: perf stat -e cycles
     │
     ▼ 系统调用: perf_event_open()
   perf 子系统
     ├─ pmu->event_init()         ← armpmu_event_init
     │    ├─ 检查 CPU 是否在 supported_cpus
     │    ├─ pmu->map_event()     ← 事件 ID → 硬件编码
     │    ├─ __hw_perf_event_init()
     │    │    ├─ set_event_filter() ← 设置特权级过滤
     │    │    └─ validate_group()  ← 检查分组约束
     │    └─ 返回 0（成功）
     │
     ▼ ioctl: PERF_EVENT_IOC_ENABLE
     ├─ pmu->add()                ← armpmu_add
     │    ├─ get_event_idx()      ← 分配硬件计数器
     │    ├─ hw_events->events[idx] = event
     │    └─ pmu->start()         ← armpmu_start
     │         ├─ armpmu_event_set_period() ← 设置溢出周期
     │         └─ pmu->enable()   ← 使能硬件计数器
     │
     ▼ 硬件运行中...
     │
     ▼ 计数器溢出 → IRQ/NMI
     ├─ armpmu_dispatch_irq()
     │    └─ pmu->handle_irq()
     │         ├─ armpmu_event_update()  ← 读取 delta
     │         ├─ perf_event_overflow()  ← 通知用户空间
     │         └─ armpmu_event_set_period() ← 重设周期
     │
     ▼ ioctl: PERF_EVENT_IOC_DISABLE / 进程退出
     └─ pmu->del()                ← armpmu_del
          ├─ pmu->stop()          ← 禁用计数器
          ├─ armpmu_event_update() ← 最终更新计数
          └─ clear_event_idx()    ← 释放计数器

12. 代码阅读自检清单
======================

阅读任何 Linux 驱动代码时，应确认以下检查点:

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 检查项
     - 说明
   * - 驱动入口
     - 如何被发现、加载、初始化
   * - 核心数据结构
     - 设备私有数据、硬件状态、per-CPU 数据
   * - 注册/注销
     - 向内核框架注册了什么、何时注销
   * - 操作函数集
     - open/read/write/ioctl/mmap 或框架特定回调
   * - 中断处理
     - 中断类型、触发条件、处理流程
   * - 资源管理
     - 硬件资源分配/释放、错误处理
   * - 并发保护
     - 锁机制、原子操作、per-CPU 无锁设计
   * - CPU 热插拔
     - 上线/下线/休眠时的状态管理
   * - 电源管理
     - runtime PM、系统休眠时的处理
   * - Sysfs/debugfs
     - 向用户空间暴露了哪些接口
   * - 错误处理
     - 资源不足、硬件故障、参数非法时的处理
   * - 框架层 vs 具体驱动
     - 哪些是通用逻辑、哪些需要具体实现
