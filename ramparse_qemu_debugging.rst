QEMU Linux Ramdump Parser (ramparse) 调试指南
==============================================

本文档记录了在 QEMU aarch64 虚拟环境中使用 Qualcomm Linux Ram Dump Parser v2 (ramparse)
解析 ramdump 的完整调试过程，包括遇到的问题、根因分析和解决方案。

环境概述
========

- **平台**: QEMU virt aarch64 虚拟机 (4x Cortex-A76)
- **内核**: Linux 6.1.0, CONFIG_DEBUG_INFO=y, VA_BITS=39, nokaslr
- **DTS**: ``binary/quantum_qemu.dts`` (``linux,dummy-virt`` compatible)
- **内存**: DDR 起始地址 ``0x40000000``, 大小 1GB (``0x40000000``-``0x80000000``)
- **ramdump 生成方式**: QEMU monitor ``dump-guest-memory``
- **交叉编译工具链**: ``arm-gnu-toolchain-13.3.rel1``
- **宿主系统**: Ubuntu 24.04, Python 3.12
- **ramparse 路径**: ``src/packages/ramparser-LE.PRODUCT.2.6.r1-02600-QCS610.0/linux-ramdump-parser-v2/``

ramparse 简介
=============

ramparse 是 Qualcomm 开源的 Python 工具，用于解析 Linux 设备的 RAM dump。它依赖
``gdb`` 和 ``nm`` 工具通过 vmlinux 符号文件从 ramdump 中提取进程栈、调度器、
中断、工作队列等内核信息。

核心架构::

    ramparse.py (入口)
    ├── ramdump.py (RamDump 类, 内存读写/虚拟地址转换)
    │   ├── gdbmi.py (GDB MI 接口, 符号查询/内存读取)
    │   ├── mmu.py (MMU, 虚拟地址到物理地址转换)
    │   ├── boards.py (Board 基类, 硬件平台定义)
    │   └── parser_util.py (子解析器注册/发现)
    ├── extensions/board_def.py (具体平台 Board 定义)
    ├── parsers/ (各子解析器: dmesg, taskdump, slabinfo 等)
    ├── local_settings.py (工具链路径配置)
    └── utils/ (辅助工具)

关键命令行参数::

    --ram-file <file> <start> <end>    指定 ram dump 文件及物理地址范围
    --vmlinux <path>                    vmlinux 符号文件路径
    --auto-dump <dir>                   自动从目录发现 ram dump 文件
    --force-hardware <board_num>        强制指定硬件平台
    --phys-offset <offset>              手动指定物理偏移
    --page-offset <offset>              手动指定页偏移
    --stdout                            输出到 stdout
    --gdb-path / --nm-path              指定 gdb/nm 路径

初始 ramparse.sh 内容::

    python3 ramparse.py --ram-file binary/ramdump 0x40000000 0x80000000 \
        --vmlinux binary/vmlinux

调试过程
========

问题 1: ModuleNotFoundError: No module named 'utils.anomalies'
----------------------------------------------------------------

**现象**::

    from utils.anomalies import Anomaly
    ModuleNotFoundError: No module named 'utils.anomalies'

**根因**: ``utils/`` 目录缺少 ``__init__.py``, Python 无法将其识别为包。

**解决**: 创建 ``utils/__init__.py``::

    touch utils/__init__.py

问题 2: ModuleNotFoundError: No module named 'prettytable'
----------------------------------------------------------------

**现象**::

    from prettytable import PrettyTable
    ModuleNotFoundError: No module named 'prettytable'

**根因**: 缺少 Python 依赖。

**解决**::

    pip3 install prettytable

问题 3: aarch64 gdb 缺少共享库
-------------------------------------------------

**现象**::

    aarch64-none-linux-gnu-gdb: error while loading shared libraries:
    libtinfo.so.5: cannot open shared object file

**根因**: 宿主系统 Ubuntu 24.04 安装的是 ``libncursesw6`` / ``libtinfo6``, 但
aarch64 交叉编译 gdb 链接的是 v5 版本。

**解决**: 创建符号链接::

    sudo ln -s /lib/x86_64-linux-gnu/libtinfo.so.6 /lib/x86_64-linux-gnu/libtinfo.so.5
    sudo ln -s /lib/x86_64-linux-gnu/libncursesw.so.6 /lib/x86_64-linux-gnu/libncursesw.so.5

问题 4: Could not find hardware — 缺少 board_def.py
----------------------------------------------------------------

**现象**::

    !!! Could not get a necessary offset for auto detection!
    !!! Could not find hardware
    !!! The SMEM didn't match anything
    !!! You can use --force-hardware to use a specific set of values

**根因**: ``extensions/`` 目录为空, 缺少 ``board_def.py`` 和 ``__init__.py``。

ramparse 通过 SMEM (Shared Memory) 自动检测 Qualcomm SoC 型号, 但 QEMU 虚拟平台
没有 SMEM, 需要手动定义 Board。

**解决**:

1. 创建 ``extensions/__init__.py``::

    touch extensions/__init__.py

2. 创建 ``extensions/board_def.py``, 根据 DTS 定义 QEMU 虚拟平台::

    import sys, os
    sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..'))
    from boards import Board

    class QEMU_VIRT(Board):
        def __init__(self):
            Board.__init__(self)
            self.socid = 9999
            self.board_num = "QEMU_VIRT"
            self.cpu = 'cortex-a76'
            self.ram_start = 0x40000000
            self.phys_offset = 0x40000000
            self.imem_start = 0
            self.smem_addr = 0
            self.wdog_addr = None
            self.imem_file_name = None
            self.smem_addr_buildinfo = 0

    QEMU_VIRT()

关键字段说明:

+------------------+---------------------------------------------------+
| 字段             | 说明                                              |
+==================+===================================================+
| ``socid``        | SMEM 中存储的 SoC ID, QEMU 无 SMEM 用虚拟值        |
+------------------+---------------------------------------------------+
| ``board_num``    | 平台标识符, 用于 ``--force-hardware`` 匹配          |
+------------------+---------------------------------------------------+
| ``ram_start``    | DDR 起始物理地址 (DTS ``memory@40000000``)          |
+------------------+---------------------------------------------------+
| ``phys_offset``  | 物理偏移 (初始值, 会被动态检测覆盖)                |
+------------------+---------------------------------------------------+
| ``smem_addr``    | SMEM 地址, QEMU 无 SMEM 设为 0                     |
+------------------+---------------------------------------------------+
| ``wdog_addr``    | Watchdog/TZ 地址, QEMU 无此硬件设为 None           |
+------------------+---------------------------------------------------+

问题 5: gdb 报错 'linux_banner' has unknown type
----------------------------------------------------------------

**现象**::

    gdbmi.GdbMIException: print /s linux_banner
    *** error,msg="'linux_banner' has unknown type; cast it to its declared type"

**根因**: ``gdbmi.py`` 中 ``get_value_of_string()`` 使用 ``print /s linux_banner`` 命令,
但 ``linux_banner`` 在 stripped vmlinux 中类型为 ``char[]`` 而非 ``char*``, gdb 无法
确定其大小。

**解决**: 修改 ``gdbmi.py:390``, 将符号强转为指针::

    # 修改前:
    cmd = 'print /s {0}'.format(symbol)
    # 修改后:
    cmd = 'print /s (char *)&{0}'.format(symbol)

问题 6: Linux banner in Dumps = 垃圾数据 — ELF core dump 格式问题
----------------------------------------------------------------

**现象**::

    Linux Banner: j38u
    Expected Linux banner = Linux version 6.1.0 ...
    Linux banner in Dumps = j38u

**根因**: QEMU 的 ``dump-guest-memory`` 生成的是 **ELF core dump** 格式 (``ET_CORE``),
而非 ramparse 期望的原始内存 dump。文件开头是 ELF 头部 (``\x7fELF``), 而不是内核
内存数据。

通过解析 ELF 程序头确认::

    Phdr 0: type=NOTE offset=0x130 filesz=0xf00
    Phdr 1: type=LOAD offset=0x1030 paddr=0x40000000 filesz=0x40000000

实际内存数据从文件偏移 ``0x1030`` 开始 (跳过 ELF 头 + NOTE 段)。

**解决**: 从 ELF core dump 中提取原始内存::

    python3 -c "
    import struct
    with open('binary/ramdump', 'rb') as f:
        hdr = f.read(64)
        e_phoff = struct.unpack_from('<Q', hdr, 32)[0]
        e_phentsize = struct.unpack_from('<H', hdr, 54)[0]
        e_phnum = struct.unpack_from('<H', hdr, 56)[0]
        with open('binary/ramdump_raw.bin', 'wb') as out:
            for i in range(e_phnum):
                f.seek(e_phoff + i * e_phentsize)
                phdr = f.read(e_phentsize)
                p_type = struct.unpack_from('<I', phdr, 0)[0]
                if p_type == 1:  # PT_LOAD
                    p_offset = struct.unpack_from('<Q', phdr, 8)[0]
                    p_filesz = struct.unpack_from('<Q', phdr, 32)[0]
                    f.seek(p_offset)
                    remaining = p_filesz
                    while remaining > 0:
                        chunk = f.read(min(0x1000000, remaining))
                        out.write(chunk)
                        remaining -= len(chunk)
    "

问题 7: phys_offset 动态检测与 kimage_voffset 计算
----------------------------------------------------------------

**现象**: 使用 ``--phys-offset 0x40000000`` 时, Linux banner 仍然无法匹配。

**根因分析**:

ramparse 的虚拟地址到物理地址转换公式::

    phys_addr = virt_addr - kimage_voffset

其中 ``kimage_voffset = kimage_vaddr - phys_offset``。

但 ``phys_offset`` 不是 DDR 起始地址 ``0x40000000``, 而是内核实际加载的物理地址。
ramparse 通过 ``determine_phys_offset()`` 在物理内存中扫描 ``kimage_voffset`` 变量的
值来自动确定::

    phys_base: 40000000 phys_end: 80000000 step: 200000
    Dynamically determined phys offset is: 40200000
    The kimage_voffset extracted is: ffffffbfc7e00000

这意味着内核实际加载在 ``0x40200000``, 而非 DDR 起始 ``0x40000000``。
这是 QEMU 固件加载内核时的行为 — initrd 放在 ``0x48000000``, 内核紧随其后。

**结论**: **不要使用 ``--phys-offset`` 参数**, 让 ramparse 动态检测。

问题 8: set_available_cores 中 field_offset 返回 None
----------------------------------------------------------------

**现象**::

    File "ramdump.py", line 2755, in set_available_cores
        cpu_present_bits = self.read_word(cpu_present_bits_addr + bits_offset)
    TypeError: unsupported operand type(s) for +: 'int' and 'NoneType'

**根因**: ``field_offset('struct cpumask', 'bits')`` 依赖 DWARF 调试信息。stripped vmlinux
中不存在 ``struct cpumask`` 的类型定义, 返回 None。

**解决**: 在 ``ramdump.py`` 的 ``set_available_cores()`` 中添加 None 保护::

    if (major, minor) >= (4, 5):
        cpu_present_bits_addr = self.address_of('__cpu_present_mask')
        bits_offset = self.field_offset('struct cpumask', 'bits')
        if bits_offset is not None:
            cpu_present_bits = self.read_word(cpu_present_bits_addr + bits_offset)
        else:
            cpu_present_bits = self.read_word(cpu_present_bits_addr)

问题 9: Could not get saved configuration — 缺少 CONFIG_IKCONFIG
----------------------------------------------------------------

**现象**::

    !!! Could not get saved configuration
    !!! This is really bad and probably indicates RAM corruption
    !!! Some features may be disabled!

**根因**: ``get_config()`` 尝试从 vmlinux 中读取 ``kernel_config_data`` 符号获取内核
配置 (``CONFIG_*``), 但内核编译时未开启 ``CONFIG_IKCONFIG``, 该符号不存在。

**影响**: 缺少内核配置导致部分功能使用默认值 (如 ``PAGE_SHIFT`` 默认 12 即 4K 页,
``VA_BITS`` 默认 39), 大部分情况下不影响解析。

**可选修复**: 内核编译时开启 ``CONFIG_IKCONFIG=y`` 和 ``CONFIG_IKCONFIG_PROC=y``。

问题 10: 大部分子解析器因缺少 DWARF 调试信息而失败
----------------------------------------------------------------

**现象** (stripped vmlinux, 9.7MB): 11 个子解析器中 7 个失败, 仅 4 个成功::

    FAILED: Schedinfo, DumpTasks, DumpProcessMemory, RunQueues, Slabinfo, Vmalloc
    OK: CpuState, CheckForPanic, DumpPageTables, IrqParse

所有失败都是同一错误类型::

    TypeError: unsupported operand type(s) for +: 'int' and 'NoneType'

**根因**: ramparse 的子解析器大量使用 ``field_offset()`` 通过 gdb 查询结构体字段偏移,
这需要 vmlinux 包含 DWARF 调试信息。stripped vmlinux 不包含 DWARF, 所有
``field_offset()`` 调用返回 None。

**解决**: 使用包含调试信息的 vmlinux (102MB):

1. 内核编译配置::

    CONFIG_DEBUG_INFO=y

2. 不要 strip vmlinux:

    # 编译后直接使用 vmlinux, 不要执行:
    # aarch64-none-linux-gnu-strip vmlinux

启用 DEBUG_INFO 后, 11 个子解析器中 7 个成功::

    OK: Dmesg, CheckForPanic, CpuState, DumpPageTables, IrqParse, RunQueues, Vmalloc
    FAILED: Schedinfo (QEMU 无 cpufreq), DumpTasks, DumpProcessMemory, Slabinfo

剩余 4 个失败是 QEMU 平台缺少特定硬件结构 + 缺少内核配置 (CONFIG_IKCONFIG) 导致的。详见问题 11-14。

问题 11: Schedinfo — per_cpu_offset 返回 None
----------------------------------------------------------------

**现象**::

    File "ramdump.py", line 2675, in read_string
        addr_or_name += pcpu_offset
    TypeError: unsupported operand type(s) for +=: 'NoneType' and 'int'

**根因**: 多个问题叠加:

1. ``per_cpu_offset()`` 调用 ``read_slong()`` 读取 ``__per_cpu_offset`` 数组,
   当读取失败时返回 None, 而 ``read_string()`` 直接将 None 与地址相加。
2. ``dump_cpufreq_data()`` 调用 ``address_of('cpufreq_cpu_data')`` 获取 per-CPU
   cpufreq 数据地址, 但 QEMU 内核没有配置 cpufreq 驱动, 符号不存在, 返回 None。

**解决**:

1. ``ramdump.py`` — ``per_cpu_offset()`` 返回 0 而非 None:

.. code-block:: diff

     result = self.read_slong(per_cpu_offset_addr_indexed)
     return result
    +if result is None:
    +    return 0
    +return result

2. ``ramdump.py`` — ``read_string()`` 中添加 None 保护:

.. code-block:: diff

     if cpu is not None:
         pcpu_offset = self.per_cpu_offset(cpu)
    +    if pcpu_offset is None:
    +        pcpu_offset = 0
         addr_or_name = self.resolve_virt(addr_or_name)

3. ``sched_info.py`` — 当 ``cpufreq_cpu_data`` 不存在时提前返回:

.. code-block:: diff

     def dump_cpufreq_data(ramdump):
n         cpufreq_data_addr = ramdump.address_of('cpufreq_cpu_data')
    +    if cpufreq_data_addr is None:
    +        print_out_str("\nCPU Frequency information: not available (no cpufreq_cpu_data)\n")
    +        return
         cpuinfo_off = ramdump.field_offset('struct cpufreq_policy', 'cpuinfo')

问题 12: DumpTasks — struct sched_info 为空 (CONFIG_SCHED_INFO 未开启)
------------------------------------------------------------------

**现象**::

    File "parsers/taskdump.py", line 96, in dump_thread_group
        offset_last_queued = offset_schedinfo + ramdump.field_offset('struct sched_info', 'last_queued')
    TypeError: unsupported operand type(s) for +: 'int' and 'NoneType'

**根因**: ``struct sched_info`` 在此内核中为空结构体 (size=0)::

    ptype/o struct sched_info {
        <no data fields>
        /* total size (bytes):    0 */
    }

这是因为内核 6.1 默认关闭了 ``CONFIG_SCHED_INFO``, 导致 ``field_offset('struct sched_info',
'last_queued')`` 等查询全部返回 None。代码中 ``offset_schedinfo`` 是一个有效整数值,
但 ``field_offset`` 返回 None, 整数与 None 相加触发 TypeError。

**解决**: 在 ``parsers/taskdump.py`` 的 ``dump_thread_group()`` 中:

1. 先安全地获取 ``sched_info`` 字段偏移, 失败时回退到直接查询 ``task_struct``:

.. code-block:: diff

    -offset_last_queued = offset_schedinfo + ramdump.field_offset('struct sched_info', 'last_queued')
    -offset_last_rundelay = offset_schedinfo + ramdump.field_offset('struct sched_info', 'run_delay')
    -offset_last_pcount = offset_schedinfo + ramdump.field_offset('struct sched_info', 'pcount')
    -offset_last_arrival = offset_schedinfo + ramdump.field_offset('struct sched_info', 'last_arrival')
    +_sched_info_last_queued = ramdump.field_offset('struct sched_info', 'last_queued')
    +if offset_schedinfo is not None and _sched_info_last_queued is not None:
    +    offset_last_queued = offset_schedinfo + _sched_info_last_queued
    +else:
    +    offset_last_queued = ramdump.field_offset('struct task_struct', 'last_queued')
    +    # run_delay, pcount, last_arrival 同理

2. 在计算地址时使用 None 保护, 读取时使用 ``or 0`` 防止 ``read_u64(None)``:

.. code-block:: diff

    -next_thread_last_arrival = next_thread_start + offset_last_arrival
    +next_thread_last_arrival = (next_thread_start + offset_last_arrival
    +                            if offset_last_arrival is not None else None)
    ...
    -    ramdump.read_u64(next_thread_last_arrival),
    +    ramdump.read_u64(next_thread_last_arrival or 0),

问题 13: DumpProcessMemory — is_thread_info_in_task 检测失败
-------------------------------------------------------------------

**现象**::

    File "ramdump.py", line 2934, in validate_task_struct
        task_address = thread_info_address + self.field_offset(
    TypeError: unsupported operand type(s) for +: 'int' and 'NoneType'

**根因**: ``validate_task_struct()`` 根据 ``is_thread_info_in_task()`` 决定如何从
``thread_info`` 地址获取 ``task_struct`` 地址。该函数依赖 ``CONFIG_THREAD_INFO_IN_TASK``
内核配置, 但因为没有 ``CONFIG_IKCONFIG``, 配置不可用, 返回 False。

于是在 ``else`` 分支中尝试 ``field_offset('struct thread_info', 'task')``, 但在现代内核中
``CONFIG_THREAD_INFO_IN_TASK=y`` 是默认值, ``thread_info`` 嵌入在 ``task_struct`` 内部,
不再有 ``task`` 指针字段, 返回 None。

**解决**: 在 ``ramdump.py`` 的 ``is_thread_info_in_task()`` 中添加基于结构体布局的
自动检测 — 如果 ``struct thread_info`` 没有 ``task`` 字段, 说明 thread_info 已嵌入
task_struct:

.. code-block:: diff

     def is_thread_info_in_task(self):
    -    return self.is_config_defined('CONFIG_THREAD_INFO_IN_TASK')
    +    if self.is_config_defined('CONFIG_THREAD_INFO_IN_TASK'):
    +        return True
    +    if not self.is_config_defined('CONFIG_THREAD_INFO_IN_TASK') and \
    +       self.get_config_val('CONFIG_THREAD_INFO_IN_TASK') is None:
    +        # No kernel config available; detect from struct layout:
    +        # if thread_info has no 'task' field, it's embedded in task_struct
    +        task_off = self.field_offset('struct thread_info', 'task')
    +        if task_off is None:
    +            return True
    +    return False

问题 14: Slabinfo — 缺少内核配置 + 内核 5.x+ struct 变更
---------------------------------------------------------------------

**现象**::

    File "mm.py", line 149, in get_vmemmap
        nlevels = int(ramdump.get_config_val("CONFIG_PGTABLE_LEVELS"))
    TypeError: int() argument must be a string, a bytes-like object or a real number, not 'NoneType'

以及后续错误::

    slab_node, cpu_slabn_addr + offsetof.cpu_cache_page_offset
    TypeError: unsupported operand type(s) for +: 'int' and 'NoneType'

**根因**: 三个独立问题:

1. ``get_vmemmap()`` 需要从内核配置获取 ``CONFIG_PGTABLE_LEVELS``, 但配置不可用。
   ``int(None)`` 直接崩溃。

2. ``struct kmem_cache_cpu`` 的 ``page`` 字段在内核 5.x+ 被重命名为 ``slab``::

    # 旧字段名 (内核 4.x):
    struct kmem_cache_cpu { ... struct page *page; ... };
    # 新字段名 (内核 5.x+):
    struct kmem_cache_cpu { ... struct slab *slab; ... };

3. ``struct page`` 中的 slab 相关字段 (``freelist``, ``objects``) 在内核 5.x+ 被移到
   ``struct slab`` (与 page 共享同一段内存的不同视图)::

    # 旧: struct page 有 freelist/objects 字段
    # 新: 这些字段在 struct slab 中 (覆盖在同一内存上)

**解决**:

1. ``mm.py`` — ``get_vmemmap()`` 中将 ``int()`` 调用移到 None 检查之后,
   并根据 ``VA_BITS`` 推断 pgtable levels:

.. code-block:: diff

    -nlevels = int(ramdump.get_config_val("CONFIG_PGTABLE_LEVELS"))
    +nlevels = ramdump.get_config_val("CONFIG_PGTABLE_LEVELS")

    +if nlevels is None or nlevels == 0:
    +    va_bits_val = ramdump.get_config_val("CONFIG_ARM64_VA_BITS")
    +    if va_bits_val is None:
    +        va_bits_val = ramdump.va_bits
    +    nlevels = 2 if va_bits_val <= 39 else (3 if va_bits_val <= 47 else 4)
    +else:
    +    nlevels = int(nlevels)

   ``va_bits`` 同样添加回退::

    -va_bits = int(ramdump.get_config_val("CONFIG_ARM64_VA_BITS"))
    +va_bits = int(ramdump.get_config_val("CONFIG_ARM64_VA_BITS") or ramdump.va_bits)

2. ``slabinfo.py`` — 为 ``struct_member_offset`` 中的字段查询添加回退:

.. code-block:: diff

     # kmem_cache_cpu.page → slab (内核 5.x+ 重命名)
     self.kmemcache_cpu_page = ramdump.field_offset(
         'struct kmem_cache_cpu', 'page')
    +if self.kmemcache_cpu_page is None:
    +    self.kmemcache_cpu_page = ramdump.field_offset(
    +        'struct kmem_cache_cpu', 'slab')

     # page.objects → slab.counters (内核 5.x+ 移到 struct slab)
     self.page_objects = ramdump.field_offset(
         'struct page', 'objects')
    +if self.page_objects is None:
    +    self.page_objects = ramdump.field_offset(
    +        'struct slab', 'counters')

     # page.freelist → slab.freelist (同上)
     self.page_freelist = ramdump.field_offset(
         'struct page', 'freelist')
    +if self.page_freelist is None:
    +    self.page_freelist = ramdump.field_offset(
    +        'struct slab', 'freelist')

     # cpu_cache_page_offset 同理
     self.cpu_cache_page_offset = ramdump.field_offset(
         'struct kmem_cache_cpu', 'page')
    +if self.cpu_cache_page_offset is None:
    +    self.cpu_cache_page_offset = ramdump.field_offset(
    +        'struct kmem_cache_cpu', 'slab')

最终的 ramparse.sh
==================

.. code-block:: bash

    python3 ramparse.py \
        --ram-file binary/ramdump_raw.bin 0x40000000 0x80000000 \
        --vmlinux binary/vmlinux \
        --force-hardware QEMU_VIRT \
        --dmesg --print-tasks --print-irqs --print-runqueues \
        --sched-info --check-for-panic --cpu-state \
        --print-memory-info --slabinfo --print-vmalloc \
        --dump-page-tables --stdout

运行前准备步骤
================

1. **安装 Python 依赖**::

    pip3 install prettytable

2. **创建交叉编译 gdb 所需的符号链接**::

    sudo ln -s /lib/x86_64-linux-gnu/libtinfo.so.6 /lib/x86_64-linux-gnu/libtinfo.so.5
    sudo ln -s /lib/x86_64-linux-gnu/libncursesw.so.6 /lib/x86_64-linux-gnu/libncursesw.so.5

3. **配置工具链路径** (已配置在 ``local_setting.config``):

    ``gdb64_path`` = ``/.../arm-gnu-toolchain-13.3.rel1.../bin/aarch64-none-linux-gnu-gdb``
    ``nm64_path`` = ``/.../arm-gnu-toolchain-13.3.rel1.../bin/aarch64-none-linux-gnu-nm``

4. **从 ELF core dump 提取原始内存** (每次更新 ramdump 后需重新执行):

    运行 ``extract_raw.py`` 或手动执行提取脚本 (见问题 6)。

5. **确保 vmlinux 包含调试信息** (文件大小应约 100MB+, 而非 10MB)。

子解析器兼容性总结
==================

.. list-table::
    :header-rows: 1
    :widths: 25 20 18 18 20

    * - 子解析器
      - 命令行参数
      - stripped vmlinux
      - debug (修复前)
      - debug (修复后)
    * - Dmesg (内核日志)
      - ``--dmesg``
      - FAIL
      - OK
      - OK
    * - 任务栈打印
      - ``--print-tasks``
      - FAIL
      - FAIL
      - OK
    * - Panic 检测
      - ``--check-for-panic``
      - OK
      - OK
      - OK
    * - CPU 状态
      - ``--cpu-state``
      - OK
      - OK
      - OK
    * - 中断信息
      - ``--print-irqs``
      - OK
      - OK
      - OK
    * - 调度器信息
      - ``--sched-info``
      - FAIL
      - FAIL
      - OK
    * - 运行队列
      - ``--print-runqueues``
      - FAIL
      - OK
      - OK
    * - Slab 信息
      - ``--slabinfo``
      - FAIL
      - FAIL
      - OK
    * - 内存使用
      - ``--print-memory-info``
      - FAIL
      - FAIL
      - OK
    * - Vmalloc 信息
      - ``--print-vmalloc``
      - FAIL
      - OK
      - OK
    * - 页表转储
      - ``--dump-page-tables``
      - OK
      - OK
      - OK

所有 11 个子解析器在 debug vmlinux + 代码修复后均能成功运行 (0 FAILED)。

关键地址参考
============

.. list-table::
    :header-rows: 1

    * - 符号
      - 虚拟地址
      - 说明
    * - ``_text``
      - ``0xffffffc008000000``
      - 内核代码段起始
    * - ``primary_entry``
      - ``0xffffffc0085e0000``
      - 内核入口
    * - ``linux_banner``
      - ``0xffffffc008527338``
      - 内核版本字符串
    * - ``saved_command_line``
      - ``0xffffffc008779020``
      - 保存的命令行
    * - ``kimage_vaddr``
      - ``0xffffffc0085a9d60``
      - 内核镜像虚拟地址
    * - ``kimage_voffset``
      - ``0xffffffc0085a9d50``
      - 内核镜像虚拟偏移
    * - ``swapper_pg_dir``
      - ``0xffffffc0085de000``
      - 内核页表
    * - PAGE_OFFSET
      - ``0xffffffc000000000``
      - 内核虚拟地址空间起始 (VA_BITS=39)

代码修改清单
============

1. ``gdbmi.py`` 行 390 — 修复 gdb 字符串读取:

.. code-block:: diff

    - cmd = 'print /s {0}'.format(symbol)
    + cmd = 'print /s (char *)&{0}'.format(symbol)

2. ``ramdump.py`` — 多处修复:

   - ``set_available_cores()``: None 保护 ``field_offset('struct cpumask', 'bits')``
   - ``per_cpu_offset()``: 返回 0 而非 None (当 ``read_slong`` 失败时)
   - ``read_string()``: None 保护 ``pcpu_offset``
   - ``is_thread_info_in_task()``: 从 struct 布局自动检测 (无内核配置时)

3. ``mm.py`` — ``get_vmemmap()`` 修复:

   - 将 ``int()`` 调用移到 None 检查之后
   - 添加 ``CONFIG_PGTABLE_LEVELS`` 基于 ``VA_BITS`` 的回退
   - ``va_bits`` 添加 ``or ramdump.va_bits`` 回退

4. ``sched_info.py`` — ``dump_cpufreq_data()`` 修复:

   - 当 ``cpufreq_cpu_data`` 符号不存在时提前返回

5. ``parsers/taskdump.py`` — ``dump_thread_group()`` 修复:

   - ``struct sched_info`` 字段偏移添加 None 保护 (空结构体时回退到直接查询)
   - 地址计算使用条件表达式防止 None 加法
   - ``read_u64/read_word`` 调用使用 ``or 0`` 防止 None 地址

6. ``parsers/slabinfo.py`` — ``struct_member_offset`` 修复:

   - ``kmem_cache_cpu.page`` → ``kmem_cache_cpu.slab`` 回退 (内核 5.x+ 重命名)
   - ``page.objects`` → ``slab.counters`` 回退 (字段移到 struct slab)
   - ``page.freelist`` → ``slab.freelist`` 回退
   - ``cpu_cache_page_offset`` 同理添加 ``slab`` 回退

7. 新增文件:

   - ``utils/__init__.py``
   - ``extensions/__init__.py``
   - ``extensions/board_def.py``
   - ``binary/ramdump_raw.bin`` (从 ELF core dump 提取)

内核 API 变更参考
==================

ramparse 代码针对 Qualcomm 内核 4.x/5.x 编写, 与上游内核 6.1 存在以下 API 变更:

+------------------------------+--------------------------------------+-------------------+
| 特性                         | 旧内核 (Qualcomm 4.x/5.x)           | 新内核 (upstream 6.1) |
+==============================+======================================+===================+
| ``struct sched_info``          | 包含 ``last_queued`` 等统计字段      | 空结构体 (size=0)  |
+------------------------------+--------------------------------------+-------------------+
| ``struct thread_info``        | 独立结构, 包含 ``task`` 指针          | 嵌入 ``task_struct`` 内 |
+------------------------------+--------------------------------------+-------------------+
| ``CONFIG_THREAD_INFO_IN_TASK`` | 可选配置                             | 默认开启         |
+------------------------------+--------------------------------------+-------------------+
| ``kmem_cache_cpu.page``       | 字段名 ``page``                       | 重命名为 ``slab`` |
+------------------------------+--------------------------------------+-------------------+
| ``page.objects/freelist``      | 在 ``struct page`` 中                 | 移到 ``struct slab`` |
+------------------------------+--------------------------------------+-------------------+
| ``CONFIG_SCHED_INFO``         | 通常开启                             | 默认关闭         |
