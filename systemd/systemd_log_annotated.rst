systemd 日志说明（带注释版）
===================================

本文档记录了 systemd 在 QEMU ARM64 虚拟机上的启动日志，包含详细的阶段划分和功能注释。

.. code-block:: console

    # ========== 系统初始化阶段 ==========
    # 设置内核 printk 设备日志输出模式为 'on'（允许所有消息输出）
    systemd[1]: Setting '/proc/sys/kernel/printk_devkmsg' to 'on'

    # 系统时间早于构建时间，调整时钟
    systemd[1]: System time before build time, advancing clock.

    # ========== 虚拟化检测阶段 ==========
    # 检测是否运行在虚拟化环境中（用于优化系统行为）
    systemd[1]: No virtualization found in DMI vendor table.
    systemd[1]: Unable to read /sys/firmware/dmi/entries/0-0/raw, using virtualization information found in DMI vendor table, ignoring: No such file or directory

    # 检测 User Mode Linux (UML) 虚拟化
    systemd[1]: UML virtualization not found in /proc/cpuinfo.

    # 检测 XEN 虚拟化
    systemd[1]: Virtualization XEN not found, /proc/xen does not exist

    # 检测 CPUID 级别的虚拟化特征
    systemd[1]: No virtualization found in CPUID

    # 检测到 QEMU 虚拟化（通过 fw-cfg 设备树节点识别）
    systemd[1]: Virtualization QEMU: "fw-cfg" present in /proc/device-tree/fw-cfg@9020000
    systemd[1]: Found VM virtualization qemu

    # 检测 virtio 设备（virtio 是 QEMU/KVM 的半虚拟化 I/O 设备接口）
    # 以下失败是正常的，因为 QEMU 虚拟机可能没有启用这些 virtio 设备
    systemd[1]: Failed to determine whether host has virtio-rng device, ignoring: No such file or directory
    systemd[1]: Failed to determine whether host has virtio-console device, ignoring: No such file or directory
    systemd[1]: Failed to determine whether host has virtio-vsock device, ignoring: No such file or directory
    systemd[1]: Failed to determine whether host has virtiofs device, ignoring: No such file or directory
    systemd[1]: Failed to determine whether host has virtio-pci device, ignoring: No such file or directory

    # ========== 内核模块加载阶段 ==========
    # 加载 libkmod 库（用于内核模块管理的动态链接库）
    systemd[1]: Loaded 'libkmod.so.2' via dlopen()

    # 尝试加载 QEMU 固件配置模块（失败是正常的，因为模块不存在或不需要）
    systemd[1]: Loading module: qemu_fw_cfg
    systemd[1]: Failed to find module 'qemu_fw_cfg'

    # 尝试加载 DMI (Desktop Management Interface) sysfs 模块
    # 模块已内建在内核中，无需加载
    systemd[1]: Loading module: dmi-sysfs
    systemd[1]: Module 'dmi_sysfs' is built in

    # ========== 文件系统挂载阶段 ==========
    # 挂载 /dev/shm (POSIX 共享内存)
    # mode=01777: 允许所有用户读写执行（粘滞位）
    systemd[1]: Mounting tmpfs to /dev/shm of type tmpfs with options mode=01777.
    systemd[1]: Mounting tmpfs (tmpfs) on /dev/shm (MS_NOSUID|MS_NODEV|MS_STRICTATIME "mode=01777")...

    # 挂载 /dev/pts (伪终端 slave 端)
    # mode=0620,gid=5: 属主读写，组只读（gid=5 是 tty 组）
    systemd[1]: Mounting devpts to /dev/pts of type devpts with options mode=0620,gid=5.
    systemd[1]: Mounting devpts (devpts) on /dev/pts (MS_NOSUID|MS_NOEXEC "mode=0620,gid=5")...

    # 挂载 /run (运行时数据目录)
    # size=20%: 最大占内存的 20%；nr_inodes=800k: 最多 80 万个 inode
    systemd[1]: Mounting tmpfs to /run of type tmpfs with options mode=0755,size=20%,nr_inodes=800k.
    systemd[1]: Mounting tmpfs (tmpfs) on /run (MS_NOSUID|MS_NODEV|MS_STRICTATIME "mode=0755,size=20%,nr_inodes=800k")...

    # ========== cgroup (控制组) 挂载阶段 ==========
    # cgroup2 是 Linux 内核的资源管理机制（v2 版本）
    systemd[1]: No filesystem is currently mounted on /sys/fs/cgroup.

    # 挂载 cgroup2
    # nsdelegate: 委托 cgroup 命名空间管理权
    # memory_recursiveprot: 内存保护递归应用到子 cgroup
    systemd[1]: Mounting cgroup2 to /sys/fs/cgroup of type cgroup2 with options nsdelegate,memory_recursiveprot.
    systemd[1]: Mounting cgroup2 (cgroup2) on /sys/fs/cgroup (MS_NOSUID|MS_NODEV|MS_NOEXEC "nsdelegate,memory_recursiveprot")...

    # ========== 其他文件系统挂载 ==========
    # pstore: 内核 panic 日志存储（用于保存崩溃前的内核日志）
    systemd[1]: Mounting pstore to /sys/fs/pstore of type pstore with options n/a.
    systemd[1]: Mounting pstore (pstore) on /sys/fs/pstore (MS_NOSUID|MS_NODEV|MS_NOEXEC "")...

    # BPF (Berkeley Packet Filter) 文件系统挂载（用于网络过滤、性能监控等）
    # 失败是正常的，因为内核可能不支持或未启用 BPF
    systemd[1]: Mounting bpf to /sys/fs/bpf of type bpf with options mode=0700.
    systemd[1]: Mounting bpf (type bpf) on /sys/fs/bpf (MS_NOSUID|MS_NODEV|MS_NOEXEC "mode=0700")...
    systemd[1]: Failed to mount bpf (type bpf) on /sys/fs/bpf (MS_NOSUID|MS_NODEV|MS_NOEXEC "mode=0700"): No such file or directory

    # ========== systemd 信息摘要 ==========
    # 确认使用 cgroup2 统一层次结构
    systemd[1]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy

    # ProtectSystem=auto 检测：不在 initrd 中运行，跳过保护设置
    systemd[1]: ProtectSystem=auto selected, but not running in an initrd, skipping.

    # systemd 版本和编译特性
    # + 表示启用，- 表示禁用
    # 例如：+KMOD 表示支持内核模块加载，-PAM 表示不支持 PAM 认证
    systemd[1]: systemd 256.7 running in system mode (-PAM -AUDIT -SELINUX -APPARMOR -IMA -SMACK -SECCOMP -GCRYPT -GNUTLS -OPENSSL -ACL +BLKID -CURL -ELFUTILS -FIDO2 -IDN2 -IDN -IPTC +KMOD -LIBCRYPTSETUP -LIBCRYPTSETUP_PLUGINS -LIBFDISK -PCRE2 -PWQUALITY -P11KIT -QRENCODE -TPM2 -BZIP2 -LZ4 -XZ -ZLIB -ZSTD -BPF_FRAMEWORK -XKBCOMMON +UTMP -SYSVINIT -LIBARCHIVE)

    # 系统环境信息
    systemd[1]: Detected virtualization qemu.
    systemd[1]: No confidential virtualization detection on this architecture
    systemd[1]: Detected architecture arm64.

    # 检测到已初始化的系统（非首次启动）
    systemd[1]: Detected initialized system, this is not first boot.

    # 内核版本检查
    systemd[1]: Kernel version 6.1.0, our baseline is 4.15

    # 凭证管理（用于传递机密信息到 systemd）
    systemd[1]: No credentials passed via fw_cfg.
    systemd[1]: No credentials passed from initrd.
    systemd[1]: Acquired 0 regular credentials, 0 untrusted credentials.

    # ========== 欢迎信息 ==========
    Welcome to Buildroot 2025.02-rc1!

    # ========== 网络配置阶段 ==========
    # 设置主机名
    systemd[1]: Hostname set to <buildroot>.

    # 配置回环接口（本地网络）
    systemd[1]: Successfully added address 127.0.0.1 to loopback interface
    systemd[1]: Successfully added address ::1 to loopback interface
    systemd[1]: Successfully brought loopback interface up

    # 网络内核参数调优
    systemd[1]: Setting '/proc/sys/net/unix/max_dgram_qlen' to '512'
    systemd[1]: Setting '/proc/sys/fs/file-max' to '9223372036854775807'
    systemd[1]: Setting '/proc/sys/fs/nr_open' to '2147483640'
    systemd[1]: Couldn't write fs.nr_open as 2147483640, halving it.
    systemd[1]: Setting '/proc/sys/fs/nr_open' to '1073741816'
    systemd[1]: Successfully bumped fs.nr_open to 1073741816

    # ========== cgroup 控制器检测 ==========
    systemd[1]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    systemd[1]: Unified cgroup hierarchy is located at /sys/fs/cgroup.

    # 检测各种 cgroup 控制器支持情况
    # BPF 相关控制器需要内核支持，当前环境不支持是正常的
    systemd[1]: bpf-firewall: Can't load kernel CGROUP SKB BPF program, BPF firewalling is not supported: Function not implemented
    systemd[1]: Can't load kernel CGROUP DEVICE BPF program, BPF device control is not supported: Function not implemented

    # 检测各个 cgroup 控制器是否支持
    # yes 表示支持，no 表示不支持（当前内核配置可能禁用了）
    systemd[1]: Controller 'cpu' supported: yes
    systemd[1]: Controller 'cpuacct' supported: no
    systemd[1]: Controller 'cpuset' supported: no
    systemd[1]: Controller 'io' supported: no
    systemd[1]: Controller 'blkio' supported: no
    systemd[1]: Controller 'memory' supported: no
    systemd[1]: Controller 'devices' supported: no
    systemd[1]: Controller 'pids' supported: no
    systemd[1]: Controller 'bpf-firewall' supported: no
    systemd[1]: Controller 'bpf-devices' supported: no
    systemd[1]: Controller 'bpf-foreign' supported: yes
    systemd[1]: Controller 'bpf-socket-bind' supported: no
    systemd[1]: Controller 'bpf-restrict-network-interfaces' supported: no

    # ========== 系统启动任务 ==========
    # 设置定时器取消标志
    systemd[1]: Set up TFD_TIMER_CANCEL_ON_SET timerfd.

    # 尝试建立内存压力事件源（需要内核支持，失败是正常的）
    systemd[1]: Failed to establish memory pressure event source, ignoring: Operation not supported

    # 使用 systemd-executor 二进制文件
    systemd[1]: Using systemd-executor binary from '/usr/lib/systemd/systemd-executor'.

    # 启用状态输出（显示启动过程的 [OK] / [FAILED] 标志）
    systemd[1]: Enabling (yes) showing of status (command line).

    # ========== 生成器（Generators）启动 ==========
    # 生成器用于动态创建 systemd 单元文件
    systemd[1]: Successfully forked off '(sd-gens)' as PID 50.
    (sd-gens)[50]: Successfully forked off '(sd-exec-strv)' as PID 51.

    # 启动各种生成器
    (sd-exec-[51]: About to execute /usr/lib/systemd/system-generators/systemd-debug-generator (null)
    (sd-exec-[51]: Successfully forked off '(direxec)' as PID 52.
    (sd-exec-[51]: About to execute /usr/lib/systemd/system-generators/systemd-fstab-generator (null)
    (sd-exec-[51]: Successfully forked off '(direxec)' as PID 53.
    (sd-exec-[51]: About to execute /usr/lib/systemd/system-generators/systemd-getty-generator (null)
    (sd-exec-[51]: Successfully forked off '(direxec)' as PID 54.
    (sd-exec-[51]: About to execute /usr/lib/systemd/system-generators/systemd-gpt-auto-generator (null)
    (sd-exec-[51]: Successfully forked off '(direxec)' as PID 55.
    (sd-exec-[51]: About to execute /usr/lib/systemd/system-generators/systemd-run-generator (null)
    (sd-exec-[51]: Successfully forked off '(direxec)' as PID 56.
    (sd-exec-[51]: About to execute /usr/lib/systemd/system-generators/systemd-ssh-generator (null)
    (sd-exec-[51]: Successfully forked off '(direxec)' as PID 57.
    (sd-exec-[51]: About to execute /usr/lib/systemd/system-generators/systemd-system-update-generator (null)
    (sd-exec-[51]: Successfully forked off '(direxec)' as PID 58.
    (sd-exec-[51]: About to execute /usr/lib/systemd/system-generators/systemd-tpm2-generator (null)
    (sd-exec-[51]: Successfully forked off '(direxec)' as PID 59.

    # ========== 生成器执行结果 ==========
    # 解析 /etc/fstab（文件系统挂载配置）
    systemd-fstab-generator[53]: Parsing /etc/fstab...

    # 自动为串口终端添加 getty（登录提示）
    systemd-getty-generator[54]: Automatically adding serial getty for /dev/ttyAMA0.

    # GPT 自动分区检测（rootfs 不是块设备，跳过）
    systemd-gpt-auto-generator[55]: Neither root nor /usr/ file system are on a (single) block device.
    systemd-gpt-auto-generator[55]: Skipping automatic GPT dissection logic, root file system not backed by a (single) whole block device.

    # SSH 生成器（sshd 未安装，禁用）
    systemd-ssh-generator[57]: Disabling SSH generator logic, since sshd is not installed.

    # TPM2 生成器（固件报告无 TPM2）
    systemd-tpm2-generator[59]: Not generating tpm2.target synchronization point, as firmware reports no TPM2 present.

    # 从 fstab 生成挂载单元
    systemd-fstab-generator[53]: Found entry what=/dev/root where=/ type=auto makefs=no growfs=no pcrfs=no noauto=no nofail=no

    # 所有生成器成功完成
    (sd-exec-[51]: /usr/lib/systemd/system-generators/systemd-fstab-generator succeeded.
    (sd-exec-[51]: /usr/lib/systemd/system-generators/systemd-gpt-auto-generator succeeded.
    (sd-exec-[51]: /usr/lib/systemd/system-generators/systemd-getty-generator succeeded.
    (sd-exec-[51]: /usr/lib/systemd/system-generators/systemd-system-update-generator succeeded.
    (sd-exec-[51]: /usr/lib/systemd/system-generators/systemd-tpm2-generator succeeded.
    (sd-exec-[51]: /usr/lib/systemd/system-generators/systemd-run-generator succeeded.
    (sd-exec-[51]: /usr/lib/systemd/system-generators/systemd-ssh-generator succeeded.
    (sd-exec-[51]: /usr/lib/systemd/system-generators/systemd-debug-generator succeeded.

    # 生成器进程完成
    (sd-gens)[50]: (sd-exec-strv) succeeded.
    systemd[1]: (sd-gens) succeeded.

    # ========== 单元文件加载 ==========
    # 查找单元文件（按优先级从高到低）
    systemd[1]: Looking for unit files in (higher priority first):
    systemd[1]:      /etc/systemd/system.control
    systemd[1]:      /run/systemd/system.control
    systemd[1]:      /run/systemd/transient
    systemd[1]:      /run/systemd/generator.early
    systemd[1]:      /etc/systemd/system
    systemd[1]:      /etc/systemd/system.attached
    systemd[1]:      /run/systemd/system
    systemd[1]:      /run/systemd/system.attached
    systemd[1]:      /run/systemd/generator
    systemd[1]:      /usr/local/lib/systemd/system
    systemd[1]:      /usr/lib/systemd/system
    systemd[1]:      /run/systemd/generator.late

    # 修改时间已更改，需要更新缓存
    systemd[1]: Modification times have changed, need to update cache.

    # ========== 单元文件解析 ==========
    # 解析各种单元文件（.service, .target, .mount, .socket 等）
    # 以下显示正在解析的单元文件

    # ========== journal 日志服务启动 ==========
    # 多条 "Found cgroup2" 消息表示各进程都正确使用了 cgroup
    # "Kernel keyring not supported" 消息表示内核不支持密钥环（正常）
    # 多个 "Credential search path" 表示查找机密存储路径

    systemd-journald[66]: systemd-journald running as PID 66 for system.
    systemd-journald[66]: Sent READY=1 notification.
    systemd-journald[66]: Sent WATCHDOG=1 notification.

    # journald 配置完成
    systemd-journald[66]: Journal effective settings seal=no keyed_hash=yes compress=NONE compress_threshold_bytes=512B

    # ========== 系统目标（Targets）启动 ==========
    # 目标（Target）是 systemd 的同步点，表示一组相关服务的完成状态

    # [  OK  ] Finished Generate network units from Kernel command line.
    # [  OK  ] Reached target Preparation for Network.

    # [  OK  ] Reached target Path Units.
    # [  OK  ] Reached target Remote File Systems.
    # [  OK  ] Reached target Slice Units.
    # [  OK  ] Reached target Swaps.

    # ========== 套接字（Sockets）监听 ==========
    [  OK  ] Listening on Credential Encryption/Decryption.
    [  OK  ] Listening on Journal Socket (/dev/log).
    [  OK  ] Listening on Journal Sockets.
    [  OK  ] Listening on Network Service Netlink Socket.
    [  OK  ] Listening on udev Control Socket.
    [  OK  ] Listening on udev Kernel Socket.

    # [  OK  ] Reached target Slice Units.
    # [  OK  ] Reached target Socket Units.
    # [  OK  ] Reached target Basic System.

    # ========== 挂载临时文件系统 ==========
    # 挂载 /tmp (临时目录）
    (mount)[62]: tmp.mount: Kernel keyring not supported, ignoring.
    (mount)[62]: tmp.mount: Executing: /usr/bin/mount tmpfs /tmp -t tmpfs -o mode=1777,strictatime,nosuid,nodev,size=50%,nr_inodes=1m

    # 加载内核模块
    # configfs: 配置文件系统（用于内核配置项）
    # efi_pstore: EFI 固件 panic 存储接口
    # fuse: 用户空间文件系统（支持多种文件系统）
    (modprobe)[63]: modprobe@configfs.service: Kernel keyring not supported, ignoring.
    (modprobe)[63]: modprobe@configfs.service: Executing: /sbin/modprobe -abq configfs
    (mount)[62]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (modprobe)[63]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy

    # [  OK  ] Finished Load Kernel Module fuse.

    # ========== 设备节点创建和规则加载 ==========
    # 创建 /dev 下的静态设备节点
    [  OK  ] Started Create Static Device Nodes in /dev gracefully.

    # ========== 系统目标完成 ==========
    [  OK  ] Reached target Preparation for Network.
    [  OK  ] Started Forward Password Requests to Console Directory Watch.
    [  OK  ] Started Forward Password Requests to Wall Directory Watch.

    # 等待串口设备就绪
    Expecting device /dev/ttyAMA0...

    # [  OK  ] Reached target Path Units.
    # [  OK  ] Reached target Remote File Systems.
    # [  OK  ] Reached target Slice Units.
    # [  OK  ] Reached target Swaps.

    # [  OK  ] Reached target Local File Systems.
    # [  OK  ] Reached target Preparation for Local File Systems.

    # ========== 虚拟控制台设置 ==========
    # [  OK  ] Finished Virtual Console Setup.

    # ========== 网络相关服务 ==========
    [  FAILED] Failed to start Network Name Resolution.
    # systemd-resolved 是 DNS 解析服务
    # 失败原因可能是没有网络配置或依赖缺失
    See 'systemctl status systemd-resolved.service' for details.

    # 多次尝试启动 Network Name Resolution（失败）
    systemd[1]: nss-lookup.target: Trying to enqueue job nss-lookup.target/start/fail
    [  FAILED] Failed to start Network Name Resolution.
    See 'systemctl status systemd-resolved.service' for details.

    # ========== 网络配置 ==========
    [  OK  ] Started Network Configuration.
    [  OK  ] Finished Coldplug All udev Devices.

    # 网络时间同步
    [  OK  ] Started Network Time Synchronization.
    # [  OK  ] Reached target System Time Set.

    # [  OK  ] Finished Enable Persistent Storage in systemd-networkd.

    # ========== 设备事件 ==========
    # 检测到各种设备（tty, 虚拟网络接口等）
    systemd[1]: dev-ram0.device: Changed dead -> plugged
    systemd[1]: dev-ttyAMA0.device: Job 73 dev-ttyAMA0.device/start finished, result=done
    [  OK  ] Found device /dev/ttyAMA0.
    systemd[1]: sys-subsystem-net-devices-sit0.device: Changed dead -> plugged

    # ========== 系统完成 ==========
    [  OK  ] Reached target System Initialization.
    [  OK  ] Started Daily Cleanup of Temporary Directories.
    [  OK  ] Reached target Timer Units.

    # [  OK  ] Listening on D-Bus System Message Bus Socket.
    # [  OK  ] Listening on Hostname Service Socket.
    # [  OK  ] Reached target Socket Units.

    # ========== D-Bus 和系统服务 ==========
    # [  OK  ] Reached target Basic System.

    # 启动 D-Bus（桌面总线和系统消息总线）
    Starting D-Bus System Message Bus...

    # 串口 getty 登录服务启动
    [  OK  ] Started Serial Getty on ttyAMA0.
    # [  OK  ] Reached target Login Prompts.

    # D-Bus 启动完成
    [  OK  ] Started D-Bus System Message Bus.
    # [  OK  ] Reached target Multi-User System.

    # ========== 网络状态 ==========
    # 即使 Network Name Resolution 失败，网络目标仍可达
    [  OK  ] Reached target Network.
    [  OK  ] Reached target Host and Network Name Lookups.

    # ========== 登录提示 ==========
    Welcome to Buildroot
    buildroot login: root (automatic login)

    # 说明：
    # - automatic login 表示自动登录为 root 用户（无需密码）
    # - 这是在配置文件中设置的调试功能
