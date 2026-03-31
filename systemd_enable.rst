使能 systemd init 及 systemd-journald
====================================

背景
----

Buildroot 已配置 ``BR2_INIT_SYSTEMD=y`` 并编译了 systemd 包，但系统实际通过内核参数 ``init=/linuxrc``
以 BusyBox init 启动，systemd 从未被加载。目标是将系统完整切换到 systemd init，并使能
systemd-journald.service。

本文档记录了完整的改动步骤、遇到的问题、分析方法及解决原理，可用于在新环境中从零使能 systemd。

前置条件
--------

- Buildroot 已配置 ``BR2_INIT_SYSTEMD=y``（默认目标 ``multi-user.target``）
- systemd 包已编译安装到 rootfs（``/usr/lib/systemd/systemd`` 存在）
- 使用 QEMU ``virt`` 平台，ARM64 架构

.. note::

   如果 Buildroot 尚未启用 systemd，需先通过 ``make menuconfig`` 配置：

   ``System configuration → Init system → systemd``


改动清单
--------

共涉及 **4 个文件修改**、**1 个文件删除**、**2 个文件新建**：

1. **删除** BusyBox init 配置（systemd 不使用）

   ``board/<board>/rootfs-overlay/etc/inittab`` → 删除

2. **新建** systemd 串口自动登录配置

   ``board/<board>/rootfs-overlay/usr/lib/systemd/system/serial-getty@.service.d/autologin.conf``

3. **新建** machine-id

   ``board/<board>/rootfs-overlay/etc/machine-id``

4. **修改** 内核命令行

   ``board/<board>/board_conf/<board>.conf``

5. **修改** 内核 defconfig

   ``board/<board>/linux/configs/<kernel>_defconfig``

6. **修改** Buildroot defconfig

   ``board/<board>/br2_external/configs/buildroot_defconfig``

下面逐一说明。

步骤一：切换 Buildroot skeleton 为 systemd
--------------------------------------------

.. note::

   Buildroot 的 **skeleton** 决定了 rootfs 的基础目录结构和 ``/sbin/init`` 指向。
   ``BR2_ROOTFS_SKELETON_DEFAULT`` 下的 ``/sbin/init`` 指向 busybox，
   ``BR2_ROOTFS_SKELETON_INIT_SYSTEMD`` 下的 ``/sbin/init`` 指向 systemd。

修改 Buildroot defconfig（以 quantum 板卡为例）：

**文件**: ``board/quantum/br2_external/configs/buildroot_defconfig``

.. code-block:: diff

   - BR2_ROOTFS_SKELETON_DEFAULT=y
   + # BR2_ROOTFS_SKELETON_DEFAULT is not set
   + BR2_ROOTFS_SKELETON_INIT_SYSTEMD=y

效果：Buildroot 会将 ``/sbin/init`` 创建为指向 ``../lib/systemd/systemd`` 的符号链接，
并自动生成 systemd 所需的 tmpfiles.d 配置、/etc/machine-id 占位等。

可选：启用 UTMP 支持（``journalctl`` 需要）：

.. code-block:: diff

   - # BR2_PACKAGE_SYSTEMD_UTMP is not set
   + BR2_PACKAGE_SYSTEMD_UTMP=y

步骤二：修改内核命令行
----------------------

**文件**: ``board/quantum/board_conf/quantum_qemu_debug.conf``

移除 ``init=/linuxrc``：

.. code-block:: diff

   - console=ttyAMA0 root=/dev/ram0 rw init=/linuxrc nokaslr ...
   + console=ttyAMA0 root=/dev/ram0 rw nokaslr ...

.. warning::

   ``init=/linuxrc`` 是关键障碍。它会覆盖 ``/sbin/init`` 的选择，强制内核使用
   BusyBox init（``/linuxrc`` 通常是 busybox 的符号链接）。移除后，内核按默认顺序
   查找 ``/sbin/init`` → ``/etc/init`` → ``/bin/init`` → ``/bin/sh``，
   从而加载 systemd。

建议在调试阶段添加 systemd 调试参数，便于排查问题：

.. code-block:: bash

   systemd.log_level=debug systemd.log_target=console systemd.show_status=1

步骤三：配置内核选项
--------------------

systemd 有一组最小内核依赖。以下列出必须和推荐的配置项。

**文件**: ``board/quantum/linux/configs/quantum_qemu_defconfig``

必须启用的选项：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 选项
     - 说明
   * - ``CONFIG_CGROUPS=y``
     - systemd 核心依赖，进程资源管理
   * - ``CONFIG_TMPFS=y``
     - ``/run`` 和 ``/dev/shm`` 挂载需要
   * - ``CONFIG_NET=y``
     - 网络栈，**netlink** 依赖此选项（systemd/udevd 核心）
   * - ``CONFIG_INET=y``
     - TCP/IP 协议栈
   * - ``CONFIG_UNIX=y``
     - Unix domain socket，systemd 进程间通信依赖
   * - ``CONFIG_PACKET=y``
     - AF_PACKET socket

推荐启用的选项：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 选项
     - 说明
   * - ``CONFIG_IPV6=y``
     - IPv6 协议（systemd 日志建议启用）
   * - ``CONFIG_DEVTMPFS=y``
     - 自动在 ``/dev`` 创建设备节点
   * - ``CONFIG_DEVTMPFS_MOUNT=y``
     - 启动时自动挂载 devtmpfs
   * - ``CONFIG_SECURITYFS=y``
     - LSM 安全框架文件系统
   * - ``CONFIG_PSTORE=y``
     - 持久存储（内核崩溃日志）
   * - ``CONFIG_FHANDLE=y``
     - ``open_by_handle_at`` 系统调用
   * - ``CONFIG_EPOLL=y``
     - I/O 事件通知
   * - ``CONFIG_SIGNALFD=y``
     - 信号通知 fd
   * - ``CONFIG_TIMERFD=y``
     - 定时器 fd
   * - ``CONFIG_PROC_SYSCTL=y``
     - ``/proc/sys`` sysctl 接口
   * - ``CONFIG_INOTIFY_USER=y``
     - 文件系统事件通知

建议禁用的选项（减少不必要的警告）：

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - 选项
     - 说明
   * - ``# CONFIG_CGROUP_DEBUG is not set``
     - 避免与 systemd cgroup 管理冲突

具体 diff 示例（最小可用配置）：

.. code-block:: diff

   - # CONFIG_CGROUPS is not set
   + CONFIG_CGROUPS=y

   - # CONFIG_TMPFS is not set
   + CONFIG_TMPFS=y

   - # CONFIG_NET is not set
   + CONFIG_NET=y
   + CONFIG_INET=y
   + CONFIG_PACKET=y
   + CONFIG_UNIX=y
   + CONFIG_IPV6=y

   - # CONFIG_SECURITYFS is not set
   + CONFIG_SECURITYFS=y

   - # CONFIG_PSTORE is not set
   + CONFIG_PSTORE=y

步骤四：配置 rootfs-overlay
----------------------------

删除 BusyBox init 的 inittab（如果存在）：

.. code-block:: bash

   rm board/<board>/rootfs-overlay/etc/inittab

.. note::

   systemd 不使用 ``/etc/inittab``。如果 overlay 中存在此文件，Buildroot 的
   systemd skeleton 不会覆盖它，可能导致冲突。

创建 systemd 串口自动登录配置：

.. note::

   内核命令行 ``console=ttyAMA0`` 会导致 systemd 自动激活
   ``serial-getty@ttyAMA0.service``，而非 ``console-getty.service`` 或
   ``getty@tty1.service``。autologin override 必须放在对应的 service.d 目录下。

**文件**: ``board/<board>/rootfs-overlay/usr/lib/systemd/system/serial-getty@.service.d/autologin.conf``

.. code-block:: ini

   [Service]
   ExecStart=
   ExecStart=-/sbin/agetty -a root --keep-baud 115200,57600,38400,9600 - $TERM

.. important::

   systemd drop-in override 中，``ExecStart=`` （空值）的作用是**清除原始命令**，
   然后下一行 ``ExecStart=...`` 写入新命令。如果省略空行，两个命令会同时存在导致错误。

创建 ``/etc/machine-id``：

.. code-block:: bash

   # 生成 32 字符随机 hex ID
   head -c 16 /dev/urandom | od -An -tx1 | tr -d ' \n' > \
       board/<board>/rootfs-overlay/etc/machine-id

.. note::

   ``/etc/machine-id`` 是 systemd 的全局唯一标识。文件必须包含 32 个十六进制字符。
   为空或不存在可能导致 systemd 初始化失败。通常由 ``systemd-machine-id-setup``
   在首次启动时生成，但在某些环境下需要预填。

步骤五：确保工具链 CPU 目标与 QEMU 匹配
----------------------------------------

检查 Buildroot defconfig 中的 CPU 目标是否与 QEMU 启动参数一致：

.. code-block:: none

   # board_conf 中的 QEMU CPU
   QEMU_CPU=cortex-a57

   # Buildroot defconfig 中的 GCC 目标
   BR2_GCC_TARGET_CPU="cortex-a57"    # 必须匹配
   BR2_cortex_a57=y

如果两者不匹配（例如 GCC 用 cortex-a76，QEMU 用 cortex-a57），
高级 CPU 指令集（ARMv8.2 FP16 等）会导致 **SIGILL** 内核恐慌。

.. warning::

   BusyBox 是纯 C 编写且只使用基础指令，不受 CPU 不匹配影响。
   systemd 依赖 glibc，glibc 可能使用高级 ISA 特性，因此会触发 SIGILL。
   这就是为什么 BusyBox init 正常但 systemd init 会崩溃。

步骤六：构建
------------

.. code-block:: bash

   # 1. 初始化环境并选择板卡（重新配置 Buildroot 使 skeleton 等改动生效）
   source build/envsetup.sh && lunch

   # 2. 清理旧构建（内核 defconfig 和 Buildroot defconfig 都改了，需完整重建）
   make clean

   # 3. 完整构建
   make

   # 4. 启动验证
   boot_kernel
```

.. important::

   ``make clean`` 是必要的，因为：
   - Buildroot defconfig 改了 skeleton 类型，需要重新生成 rootfs 结构
   - 内核 defconfig 改了多个选项，需要重新配置和编译内核
   - ``make linux-rebuild`` 不会重新读 defconfig，需要 ``make linux-reconfigure``
   - ``make clean && make`` 是最安全的方式，确保所有改动生效

问题排查手册
------------

问题一：systemd drop-in autologin 不生效
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**现象**: 添加了 ``autologin.conf`` 到 ``console-getty.service.d/``，但登录提示仍然存在。

**根因**: 内核参数 ``console=ttyAMA0`` 导致 systemd 激活的是
``serial-getty@ttyAMA0.service``，而非 ``console-getty.service``。

**分析方法**:

.. code-block:: bash

   # 检查内核命令行中的 console= 参数
   cat /proc/cmdline

   # 检查哪个 getty 服务实际在运行
   systemctl status serial-getty@ttyAMA0.service
   systemctl status console-getty.service

**解决**: 将 autologin.conf 放到正确目录：
``serial-getty@.service.d/autologin.conf``。

**规律总结**:

.. list-table::
   :header-rows: 1
   :widths: 40 30 30

   * - 内核 console= 参数
     - 激活的 systemd 服务
     - drop-in 目录
   * - ``console=ttyAMA0`` (串口)
     - ``serial-getty@ttyAMA0.service``
     - ``serial-getty@.service.d/``
   * - ``console=tty0`` (虚拟终端)
     - ``getty@tty1.service``
     - ``getty@.service.d/``
   * - 无或 ``console=`` (控制台)
     - ``console-getty.service``
     - ``console-getty.service.d/``

问题二：Kernel panic - exitcode=0x00000004 (SIGILL)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**现象**: 切换到 systemd init 后，内核恐慌 "Attempted to kill init! exitcode=0x00000004"。

**根因**: exitcode=0x00000004 表示 **SIGILL**（非法指令）。工具链目标 CPU
（如 cortex-a76）与 QEMU 模拟 CPU（如 cortex-a57）不匹配。cortex-a76 编译出的
二进制可能包含 ARMv8.2-A 指令（FP16 等），cortex-a57 (ARMv8.0-A) 不支持。

**分析方法**:

.. code-block:: bash

   # 检查 exitcode 含义
   # exitcode 低 7 位 = 信号号: 4 = SIGILL

   # 检查 Buildroot defconfig 中的 CPU 目标
   grep GCC_TARGET_CPU board/*/br2_external/configs/buildroot_defconfig

   # 检查 board_conf 中的 QEMU CPU
   grep QEMU_CPU board/*/board_conf/*.conf

   # 检查 systemd 二进制的 ELF 架构
   file out/<board>/target/usr/lib/systemd/systemd
   readelf -A out/<board>/target/usr/lib/systemd/systemd

**解决**: 确保 ``BR2_GCC_TARGET_CPU`` 与 ``QEMU_CPU`` 一致。

问题三：Failed to start up manager
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**现象**: systemd 启动但立即失败，日志被 "Too many messages being logged to kmsg" 截断。

**根因**: 内核缺少 systemd 所需的配置选项（通常是 ``CONFIG_NET`` 导致 netlink 不可用）。

**分析方法**: 添加调试参数到内核命令行查看完整错误：

.. code-block:: none

   systemd.log_level=debug systemd.log_target=console systemd.show_status=1

在输出中搜索关键字：

.. list-table::
   :header-rows: 1
   :widths: 50 50

   * - 日志关键字
     - 缺失的内核选项
   * - ``Failed to open netlink: Function not implemented``
     - ``CONFIG_NET=y``
   * - ``Failed to allocate device monitor: Function not implemented``
     - ``CONFIG_NET=y``（netlink 依赖）
   * - ``lacks built-in unix support``
     - ``CONFIG_UNIX=y``
   * - ``lacks built-in ipv6 support``
     - ``CONFIG_IPV6=y``
   * - ``Failed to mount securityfs: No such file or directory``
     - ``CONFIG_SECURITYFS=y``
   * - ``Failed to mount pstore: No such file or directory``
     - ``CONFIG_PSTORE=y``
   * - ``Failed to mount bpf: No such file or directory``
     - ``CONFIG_BPF_SYSCALL=y``
   * - ``Controller 'memory' supported: no``
     - ``CONFIG_MEMCG=y``
   * - ``Controller 'devices' supported: no``
     - ``CONFIG_CGROUP_DEVICE=y``

**解决**: 根据缺失选项修改内核 defconfig，然后 ``make linux-reconfigure && make``。

问题四：defconfig 改动不生效
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**现象**: 修改了内核 defconfig 但 ``make linux-rebuild`` 后行为不变。

**根因**: Buildroot 缓存了内核 ``.config``。``linux-rebuild`` 只编译不重新配置。

**解决**:

.. code-block:: bash

   # 方法一：重新配置内核
   make linux-reconfigure && make linux-rebuild

   # 方法二：完整重建（推荐，最安全）
   make clean && make

问题五：init=/linuxrc 与 systemd 共存
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**现象**: Buildroot 配置了 systemd，但 ``/etc/inittab`` 存在，或 ``/sbin/init``
仍然指向 busybox。

**分析方法**:

.. code-block:: bash

   # 检查实际运行的 init
   ls -la target/sbin/init

   # 检查内核命令行是否有 init= 覆盖
   cat /proc/cmdline | grep 'init='

   # 检查 inittab 是否残留
   ls -la target/etc/inittab

**解决**:

1. 移除内核命令行的 ``init=`` 参数
2. 删除 overlay 中的 ``/etc/inittab``
3. 切换 skeleton 为 ``BR2_ROOTFS_SKELETON_INIT_SYSTEMD=y``

systemd 与 BusyBox init 的关键差异
------------------------------------

.. list-table::
   :header-rows: 1
   :widths: 30 30 40

   * - 项目
     - BusyBox init
     - systemd init
   * - 配置文件
     - ``/etc/inittab``
     - ``*.service`` unit 文件
   * - 串口登录
     - ``inittab`` 中 getty 行
     - ``serial-getty@.service``
   * - 自动登录
     - ``-/bin/sh`` 或 getty ``-n``
     - ``-a root`` 参数
   * - 服务管理
     - 无
     - ``systemctl start/stop/enable``
   * - 日志系统
     - 无（输出到 console）
     - ``systemd-journald`` + ``journalctl``
   * - /sbin/init 来源
     - skeleton-default → busybox
     - skeleton-init-systemd → systemd
   * - 内核参数
     - 无特殊要求
     - 需要 CGROUPS, TMPFS, NET 等

新环境使能 systemd 检查清单
---------------------------

.. list-table::
   :header-rows: 1
   :widths: 50 50

   * - 检查项
     - 验证方法
   * - Buildroot 启用 systemd init
     - ``grep BR2_INIT_SYSTEMD buildroot_defconfig``
   * - skeleton 使用 systemd 版本
     - ``grep BR2_ROOTFS_SKELETON buildroot_defconfig``
   * - 内核命令行无 ``init=`` 覆盖
     - ``grep 'init=' board_conf/*.conf``
   * - ``/sbin/init`` 指向 systemd
     - ``ls -la target/sbin/init``
   * - ``/etc/inittab`` 不存在
     - ``ls target/etc/inittab``
   * - ``/etc/machine-id`` 非空
     - ``cat target/etc/machine-id``
   * - 内核启用 CGROUPS
     - ``grep CONFIG_CGROUPS kernel/.config``
   * - 内核启用 TMPFS
     - ``grep CONFIG_TMPFS kernel/.config``
   * - 内核启用 NET + UNIX
     - ``grep CONFIG_NET kernel/.config``
   * - 工具链 CPU 目标与 QEMU 匹配
     - 对比 ``GCC_TARGET_CPU`` 与 ``QEMU_CPU``
   * - systemd-journald 运行
     - ``systemctl status systemd-journald``
   * - journalctl 可用
     - ``journalctl -b``
