Buildroot 自动登录配置排障记录
==================================

背景
----

Buildroot 构建完成后，通过 QEMU 启动 Linux 系统，串口终端会出现如下登录提示::

    Welcome to Buildroot
    buildroot login:

即使 root 密码已设为空（``BR2_TARGET_GENERIC_ROOT_PASSWD=""``），仍需要手动按回车才能登录，
需要实现系统启动后自动以 root 身份进入 shell。

问题分析过程
------------

第一次尝试：修改 systemd console-getty.service
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Buildroot defconfig 中配置了 ``BR2_INIT_SYSTEMD=y``，因此首先在 rootfs overlay 中为
``console-getty.service`` 创建 systemd drop-in override 文件::

    usr/lib/systemd/system/console-getty.service.d/autologin.conf

内容为::

    [Service]
    ExecStart=
    ExecStart=-/sbin/agetty -a root --noclear --keep-baud - 115200,38400,9600 $TERM

重新构建后启动，登录提示依然存在。

第二次尝试：修改 serial-getty@.service
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

检查内核命令行发现 ``console=ttyAMA0``，systemd 在这种情况下激活的是
``serial-getty@ttyAMA0.service`` 而非 ``console-getty.service``。
因此补充为 ``serial-getty@.service.d/autologin.conf`` 添加同样的 override。

重新构建后启动，登录提示**仍然存在**。

根因定位
^^^^^^^^

两次尝试都失败，说明 systemd drop-in 方案本身就不适用。进一步排查发现了关键线索：

1. 内核命令行中存在 ``init=/linuxrc`` 参数::

       console=ttyAMA0 root=/dev/ram0 rw init=/linuxrc nokaslr ...

   ``init=/linuxrc`` 会覆盖默认的 init 进程，而 ``linuxrc`` 通常是指向 BusyBox init 的符号链接。

2. BusyBox init 解析的是 ``/etc/inittab``，而不是 systemd 的 service 文件。
   查看 ``/etc/inittab`` 内容::

       console::respawn:/sbin/getty -L  console 0 vt100 # GENERIC_SERIAL

   这一行才是实际产生登录提示的来源。

3. 虽然 Buildroot 配置了 ``BR2_INIT_SYSTEMD=y`` 并打包了 systemd，
   但内核的 ``init=/linuxrc`` 参数导致系统实际运行的是 BusyBox init，
   systemd 及其所有 service 配置完全未被加载。

结论：**配置与实际运行环境不一致** — Buildroot 配置了 systemd init，但内核参数指定了 BusyBox init。

最终解决方案
------------

在 rootfs overlay 中覆盖 ``/etc/inittab``，将 getty 登录行改为直接启动 root shell::

    # 原始内容（需要登录）:
    console::respawn:/sbin/getty -L  console 0 vt100 # GENERIC_SERIAL

    # 修改后（自动登录）:
    console::respawn:-/bin/sh

其中 ``-/bin/sh`` 前的 ``-`` 表示以 login shell 方式启动，会正确加载 ``/etc/profile`` 等环境变量。

同时清理掉之前创建的无效 systemd drop-in 文件。

关键经验总结
------------

1. **确认实际 init 系统，不要只看 Buildroot 配置**
   Buildroot 的 ``BR2_INIT_SYSTEMD=y`` 并不意味着系统一定使用 systemd init。
   内核 ``init=`` 参数拥有最高优先级，会覆盖 Buildroot 的 init 配置。
   排查时应先确认实际运行的 init 进程::

       ps aux | grep init
       cat /proc/1/comm

2. **根据 init 系统选择正确的配置方式**

   ==========  ========================  ==========================================
   init 系统   配置文件                  自动登录方式
   ==========  ========================  ==========================================
   BusyBox     ``/etc/inittab``          ``console::respawn:-/bin/sh``
   SysV init   ``/etc/inittab``          ``1:2345:respawn:/sbin/getty -n -l /bin/sh ...``
   systemd     ``*.service.d/*.conf``    ``ExecStart=-/sbin/agetty -a root ...``
   ==========  ========================  ==========================================

3. **systemd drop-in 中 ExecStart 必须先清空再重写**
   systemd 的 drop-in override 机制要求先写空行 ``ExecStart=`` 清除原始命令，
   然后再写新的 ``ExecStart=...``。只写新的 ``ExecStart`` 会导致两个命令同时存在。

4. **排查 init 问题的检查清单**

   - 查看内核命令行中的 ``init=`` 参数
   - 确认 ``/linuxrc`` 或 ``/sbin/init`` 指向哪个程序
   - 检查 ``/etc/inittab`` 是否存在（BusyBox/SysV init）
   - 检查 systemd service 文件（systemd init）
   - 确认 getty/agetty 二进制路径是否正确（``/sbin/agetty`` vs ``/usr/sbin/agetty``）

5. **``console=`` 参数影响 systemd 的 getty 服务选择**
   ``console=ttyAMA0`` 会触发 ``serial-getty@ttyAMA0.service``,
   ``console=tty0`` 会触发 ``getty@tty1.service``,
   不指定 console 则可能使用 ``console-getty.service``。
   需要根据实际情况修改对应的 service 文件。
