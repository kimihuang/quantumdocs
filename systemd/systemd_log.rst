systemd 日志说明
===================================

.. code-block:: console

    systemd[1]: Setting '/proc/sys/kernel/printk_devkmsg' to 'on'
    systemd[1]: System time before build time, advancing clock.
    systemd[1]: No virtualization found in DMI vendor table.
    systemd[1]: Unable to read /sys/firmware/dmi/entries/0-0/raw, using the virtualization information found in DMI vendor table, ignoring: No such file or directory
    systemd[1]: UML virtualization not found in /proc/cpuinfo.
    systemd[1]: Virtualization XEN not found, /proc/xen does not exist
    systemd[1]: No virtualization found in CPUID
    systemd[1]: Virtualization QEMU: "fw-cfg" present in /proc/device-tree/fw-cfg@9020000
    systemd[1]: Found VM virtualization qemu
    systemd[1]: Failed to determine whether host has virtio-rng device, ignoring: No such file or directory
    systemd[1]: Failed to determine whether host has virtio-console device, ignoring: No such file or directory
    systemd[1]: Failed to determine whether host has virtio-vsock device, ignoring: No such file or directory
    systemd[1]: Failed to determine whether host has virtiofs device, ignoring: No such file or directory
    systemd[1]: Failed to determine whether host has virtio-pci device, ignoring: No such file or directory
    systemd[1]: Loaded 'libkmod.so.2' via dlopen()
    systemd[1]: Loading module: qemu_fw_cfg
    systemd[1]: Failed to find module 'qemu_fw_cfg'
    systemd[1]: Loading module: dmi-sysfs
    systemd[1]: Module 'dmi_sysfs' is built in
    systemd[1]: Mounting tmpfs to /dev/shm of type tmpfs with options mode=01777.
    systemd[1]: Mounting tmpfs (tmpfs) on /dev/shm (MS_NOSUID|MS_NODEV|MS_STRICTATIME "mode=01777")...
    systemd[1]: Mounting devpts to /dev/pts of type devpts with options mode=0620,gid=5.
    systemd[1]: Mounting devpts (devpts) on /dev/pts (MS_NOSUID|MS_NOEXEC "mode=0620,gid=5")...
    systemd[1]: Mounting tmpfs to /run of type tmpfs with options mode=0755,size=20%,nr_inodes=800k.
    systemd[1]: Mounting tmpfs (tmpfs) on /run (MS_NOSUID|MS_NODEV|MS_STRICTATIME "mode=0755,size=20%,nr_inodes=800k")...
    systemd[1]: No filesystem is currently mounted on /sys/fs/cgroup.
    systemd[1]: Mounting cgroup2 to /sys/fs/cgroup of type cgroup2 with options nsdelegate,memory_recursiveprot.
    systemd[1]: Mounting cgroup2 (cgroup2) on /sys/fs/cgroup (MS_NOSUID|MS_NODEV|MS_NOEXEC "nsdelegate,memory_recursiveprot")...
    systemd[1]: Mounting pstore to /sys/fs/pstore of type pstore with options n/a.
    systemd[1]: Mounting pstore (pstore) on /sys/fs/pstore (MS_NOSUID|MS_NODEV|MS_NOEXEC "")...
    systemd[1]: Mounting bpf to /sys/fs/bpf of type bpf with options mode=0700.
    systemd[1]: Mounting bpf (bpf) on /sys/fs/bpf (MS_NOSUID|MS_NODEV|MS_NOEXEC "mode=0700")...
    systemd[1]: Failed to mount bpf (type bpf) on /sys/fs/bpf (MS_NOSUID|MS_NODEV|MS_NOEXEC "mode=0700"): No such file or directory
    systemd[1]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    systemd[1]: ProtectSystem=auto selected, but not running in an initrd, skipping.
    systemd[1]: systemd 256.7 running in system mode (-PAM -AUDIT -SELINUX -APPARMOR -IMA -SMACK -SECCOMP -GCRYPT -GNUTLS -OPENSSL -ACL +BLKID -CURL -ELFUTILS -FIDO2 -IDN2 -IDN -IPTC +KMOD -LIBCRYPTSETUP -LIBCRYPTSETUP_PLUGINS -LIBFDISK -PCRE2 -PWQUALITY -P11KIT -QRENCODE -TPM2 -BZIP2 -LZ4 -XZ -ZLIB -ZSTD -BPF_FRAMEWORK -XKBCOMMON +UTMP -SYSVINIT -LIBARCHIVE)
    systemd[1]: Detected virtualization qemu.
    systemd[1]: No confidential virtualization detection on this architecture
    systemd[1]: Detected architecture arm64.
    systemd[1]: Detected initialized system, this is not the first boot.
    systemd[1]: Kernel version 6.1.0, our baseline is 4.15
    systemd[1]: No credentials passed via fw_cfg.
    systemd[1]: No credentials passed from initrd.
    systemd[1]: Acquired 0 regular credentials, 0 untrusted credentials.

    Welcome to Buildroot 2025.02-rc1!

    systemd[1]: Hostname set to <buildroot>.
    systemd[1]: Successfully added address 127.0.0.1 to loopback interface
    systemd[1]: Successfully added address ::1 to loopback interface
    systemd[1]: Successfully brought loopback interface up
    systemd[1]: Setting '/proc/sys/net/unix/max_dgram_qlen' to '512'
    systemd[1]: Setting '/proc/sys/fs/file-max' to '9223372036854775807'
    systemd[1]: Setting '/proc/sys/fs/nr_open' to '2147483640'
    systemd[1]: Couldn't write fs.nr_open as 2147483640, halving it.
    systemd[1]: Setting '/proc/sys/fs/nr_open' to '1073741816'
    systemd[1]: Successfully bumped fs.nr_open to 1073741816
    systemd[1]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    systemd[1]: Unified cgroup hierarchy is located at /sys/fs/cgroup.
    systemd[1]: bpf-firewall: Can't load kernel CGROUP SKB BPF program, BPF firewalling is not supported: Function not implemented
    systemd[1]: Can't load kernel CGROUP DEVICE BPF program, BPF device control is not supported: Function not implemented
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
    systemd[1]: Set up TFD_TIMER_CANCEL_ON_SET timerfd.
    systemd[1]: Failed to establish memory pressure event source, ignoring: Operation not supported
    systemd[1]: Using systemd-executor binary from '/usr/lib/systemd/systemd-executor'.
    systemd[1]: Enabling (yes) showing of status (command line).
    systemd[1]: Successfully forked off '(sd-gens)' as PID 50.
    (sd-gens)[50]: Successfully forked off '(sd-exec-strv)' as PID 51.
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
    systemd-fstab-generator[53]: Parsing /etc/fstab...
    systemd-getty-generator[54]: Automatically adding serial getty for /dev/ttyAMA0.
    systemd-gpt-auto-generator[55]: Neither root nor /usr/ file system are on a (single) block device.
    systemd-gpt-auto-generator[55]: Skipping automatic GPT dissection logic, root file system not backed by a (single) whole block device.
    systemd-ssh-generator[57]: Disabling SSH generator logic, since sshd is not installed.
    systemd-tpm2-generator[59]: Not generating tpm2.target synchronization point, as firmware reports no TPM2 present.
    systemd-fstab-generator[53]: Found entry what=/dev/root where=/ type=auto makefs=no growfs=no pcrfs=no noauto=no nofail=no
    (sd-exec-[51]: /usr/lib/systemd/system-generators/systemd-fstab-generator succeeded.
    (sd-exec-[51]: /usr/lib/systemd/system-generators/systemd-gpt-auto-generator succeeded.
    (sd-exec-[51]: /usr/lib/systemd/system-generators/systemd-getty-generator succeeded.
    (sd-exec-[51]: /usr/lib/systemd/system-generators/systemd-system-update-generator succeeded.
    (sd-exec-[51]: /usr/lib/systemd/system-generators/systemd-tpm2-generator succeeded.
    (sd-exec-[51]: /usr/lib/systemd/system-generators/systemd-run-generator succeeded.
    (sd-exec-[51]: /usr/lib/systemd/system-generators/systemd-ssh-generator succeeded.
    (sd-exec-[51]: /usr/lib/systemd/system-generators/systemd-debug-generator succeeded.
    (sd-gens)[50]: (sd-exec-strv) succeeded.
    systemd[1]: (sd-gens) succeeded.
    systemd[1]: Looking for unit files in (higher priority first):
    systemd[1]:     /etc/systemd/system.control
    systemd[1]:     /run/systemd/system.control
    systemd[1]:     /run/systemd/transient
    systemd[1]:     /run/systemd/generator.early
    systemd[1]:     /etc/systemd/system
    systemd[1]:     /etc/systemd/system.attached
    systemd[1]:     /run/systemd/system
    systemd[1]:     /run/systemd/system.attached
    systemd[1]:     /run/systemd/generator
    systemd[1]:     /usr/local/lib/systemd/system
    systemd[1]:     /usr/lib/systemd/system
    systemd[1]:     /run/systemd/generator.late
    systemd[1]: Modification times have changed, need to update cache.
    systemd[1]: unit_file_build_name_map: normal unit file: /run/systemd/generator/-.mount
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/user@.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/user.slice
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/user-runtime-dir@.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/usb-gadget.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/umount.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/tpm2.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/tmp.mount
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/timers.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/time-sync.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/time-set.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-vconsole-setup.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-update-utmp.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-update-done.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-udevd.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-udevd-kernel.socket
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-udevd-control.socket
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-udev-trigger.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-udev-settle.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-udev-load-credentials.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-tmpfiles-setup.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-tmpfiles-setup-dev.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-tmpfiles-setup-dev-early.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-tmpfiles-clean.timer
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-tmpfiles-clean.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-timesyncd.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-timedated.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-time-wait-sync.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-sysctl.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-suspend.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-soft-reboot.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-resolved.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-remount-fs.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-reboot.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-pstore.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-poweroff.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-nspawn@.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-networkd.socket
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-networkd.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-networkd-wait-online@.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-networkd-wait-online.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-networkd-persistent-storage.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-network-generator.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-modules-load.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-machine-id-commit.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-kexec.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-journald@.socket
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-journald@.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-journald.socket
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-journald.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-journald-varlink@.socket
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-journald-sync@.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-journald-dev-log.socket
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-journald-audit.socket
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-journal-flush.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-hostnamed.socket
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-hostnamed.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-halt.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-growfs@.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-growfs-root.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-fsck@.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-fsck-root.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-exit.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-creds@.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-creds.socket
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-boot-check-no-failures.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-ask-password-wall.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-ask-password-wall.path
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-ask-password-console.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/systemd-ask-password-console.path
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/system-update.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/system-update-pre.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/system-update-cleanup.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/syslog.socket
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/sysinit.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/sys-kernel-tracing.mount
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/sys-kernel-debug.mount
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/sys-kernel-config.mount
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/sys-fs-fuse-connections.mount
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/swap.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/suspend.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/ssh-access.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/sound.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/soft-reboot.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/sockets.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/smartcard.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/slices.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/sleep.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/sigpwr.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/shutdown.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/serial-getty@.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/rpcbind.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/rescue.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/rescue.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/remote-fs.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/remote-fs-pre.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/reboot.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/printer.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/poweroff.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/paths.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/nss-user-lookup.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/nss-lookup.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/network.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/network-pre.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/network-online.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/multi-user.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/modprobe@.service
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/local-fs.target
    systemd[1]: unit_file_build_name_map: normal unit file: /usr/lib/systemd/system/local-fs-pre.target
    systemd[1]: Too many messages being logged to kmsg, ignoring
    [  OK  ] Created slice Slice /system/modprobe.
    [  OK  ] Created slice Slice /system/serial-getty.
    [  OK  ] Started Dispatch Password Requests to Console Directory Watch.
    [  OK  ] Started Forward Password Requests to Wall Directory Watch.
            Expecting device /dev/ttyAMA0...
    [  OK  ] Reached target Path Units.
    [  OK  ] Reached target Remote File Systems.
    [  OK  ] Reached target Slice Units.
    [  OK  ] Reached target Swaps.
    [  OK  ] Listening on Credential Encryption/Decryption.
    [  OK  ] Listening on Journal Socket (/dev/log).
    [  OK  ] Listening on Journal Sockets.
    [  OK  ] Listening on Network Service Netlink Socket.
    [  OK  ] Listening on udev Control Socket.
    [  OK  ] Listening on udev Kernel Socket.
            Mounting Temporary Directory /tmp...
            Starting Load Kernel Module configfs...
            Starting Load Kernel Module efi_pstore...
            Starting Load Kernel Module fuse...
    (mount)[62]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (modprobe)[63]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
            Starting Journal Service...
    (modprobe)[63]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (mount)[62]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (modprobe)[63]: modprobe@configfs.service: Kernel keyring not supported, ignoring.
    (modprobe)[63]: modprobe@configfs.service: Executing: /sbin/modprobe -abq configfs
    (mount)[62]: tmp.mount: Kernel keyring not supported, ignoring.
            Starting Generate network units from Kernel command line...
    (mount)[62]: tmp.mount: Executing: /usr/bin/mount tmpfs /tmp -t tmpfs -o mode=1777,strictatime,nosuid,nodev,size=50%,nr_inodes=1m
            Starting Remount Root and Kernel File Systems...
    (modprobe)[64]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
            Starting Apply Kernel Variables...
    (journald)[66]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (modprobe)[64]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (journald)[66]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
            Starting Create Static Device Nodes in /dev gracefully...
    (journald)[66]: Successfully forked off '(sd-mkdcreds)' as PID 72.
    (modprobe)[65]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (modprobe)[64]: modprobe@efi_pstore.service: Kernel keyring not supported, ignoring.
            Starting Load udev Rules from Credentials...
    (sd-mkdcre[72]: Changing mount propagation /dev (MS_REC|MS_SLAVE "")
    (modprobe)[65]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (sd-mkdcre[72]: Mounting ramfs (ramfs) on /dev/shm (MS_NOSUID|MS_NODEV|MS_NOEXEC|MS_NOSYMFOLLOW "mode=0700")...
    (sd-mkdcre[72]: Credential search path is: /etc/credstore:/run/credstore:/usr/local/lib/credstore:/usr/lib/credstore
    (modprobe)[65]: modprobe@fuse.service: Kernel keyring not supported, ignoring.
    (modprobe)[64]: modprobe@efi_pstore.service: Executing: /sbin/modprobe -abq efi_pstore
    (sd-mkdcre[72]: Credential search path is: /etc/credstore.encrypted:/run/credstore.encrypted:/usr/local/lib/credstore.encrypted:/usr/lib/credstore.encrypted
    (sd-mkdcre[72]: Changing mount flags /dev/shm (MS_RDONLY|MS_NOSUID|MS_NODEV|MS_NOEXEC|MS_REMOUNT|MS_NOSYMFOLLOW|MS_BIND "")...
    (sd-mkdcre[72]: Moving mount /dev/shm → /run/credentials/systemd-journald.service (MS_MOVE "")...
            Starting Coldplug All udev Devices...
    (modprobe)[65]: modprobe@fuse.service: Executing: /sbin/modprobe -abq fuse
    (journald)[66]: (sd-mkdcreds) succeeded.
    (journald)[66]: systemd-journald.service: Kernel keyring not supported, ignoring.
    (journald)[66]: systemd-journald.service: Executing: /usr/lib/systemd/systemd-journald
    (mount-fs)[69]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (enerator)[68]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (enerator)[68]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (mount-fs)[69]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (tmpfiles)[71]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy

    (enerator)[68]: Successfully forked off '(sd-mkdcreds)' as PID 75.
    (sd-mkdcre[75]: Changing mount propagation /dev (MS_REC|MS_SLAVE "")
    (mount-fs)[69]: systemd-remount-fs.service: Kernel keyring not supported, ignoring.
    (tmpfiles)[71]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (mount-fs)[69]: systemd-remount-fs.service: Executing: /usr/lib/systemd/systemd-remount-fs
    (d-sysctl)[70]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (tmpfiles)[71]: Successfully forked off '(sd-mkdcreds)' as PID 76.
    (sd-mkdcre[75]: Mounting ramfs (ramfs) on /dev/shm (MS_NOSUID|MS_NODEV|MS_NOEXEC|MS_NOSYMFOLLOW "mode=0700")...
    (sd-mkdcre[75]: Credential search path is: /etc/credstore:/run/credstore:/usr/local/lib/credstore:/usr/lib/credstore
    (sd-mkdcre[75]: Credential search path is: /etc/credstore.encrypted:/run/credstore.encrypted:/usr/local/lib/credstore.encrypted:/usr/lib/credstore.encrypted
    (sd-mkdcre[75]: Credential search path is: /etc/credstore:/run/credstore:/usr/local/lib/credstore:/usr/lib/credstore
    (sd-mkdcre[76]: Changing mount propagation /dev (MS_REC|MS_SLAVE "")
    (sd-mkdcre[75]: Credential search path is: /etc/credstore.encrypted:/run/credstore.encrypted:/usr/local/lib/credstore.encrypted:/usr/lib/credstore.encrypted
    (udevadm)[73]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (d-sysctl)[70]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (sd-mkdcre[76]: Mounting ramfs (ramfs) on /dev/shm (MS_NOSUID|MS_NODEV|MS_NOEXEC|MS_NOSYMFOLLOW "mode=0700")...
    (sd-mkdcre[76]: Credential search path is: /etc/credstore:/run/credstore:/usr/local/lib/credstore:/usr/lib/credstore

    (sd-mkdcre[76]: Credential search path is: /etc/credstore.encrypted:/run/credstore.encrypted:/usr/local/lib/credstore.encrypted:/usr/lib/credstore.encrypted
    (sd-mkdcre[76]: Changing mount flags /dev/shm (MS_RDONLY|MS_NOSUID|MS_NODEV|MS_NOEXEC|MS_REMOUNT|MS_NOSYMFOLLOW|MS_BIND "")...
    (d-sysctl)[70]: Successfully forked off '(sd-mkdcreds)' as PID 77.
    (sd-mkdcre[75]: Credential search path is: /etc/credstore:/run/credstore:/usr/local/lib/credstore:/usr/lib/credstore
    (sd-mkdcre[75]: Credential search path is: /etc/credstore.encrypted:/run/credstore.encrypted:/usr/local/lib/credstore.encrypted:/usr/lib/credstore.encrypted
    (sd-mkdcre[75]: Changing mount flags /dev/shm (MS_RDONLY|MS_NOSUID|MS_NODEV|MS_NOEXEC|MS_REMOUNT|MS_NOSYMFOLLOW|MS_BIND "")...
    systemd-journald[66]: Failed to read credential journal.forward_to_socket, ignoring: No such file or directory
    systemd-journald[66]: Failed to read credential journal.storage, ignoring: No such file or directory
    (sd-mkdcre[77]: Changing mount propagation /dev (MS_REC|MS_SLAVE "")
    (sd-mkdcre[75]: Moving mount /dev/shm → /run/credentials/systemd-network-generator.service (MS_MOVE "")...
    (sd-mkdcre[76]: Moving mount /dev/shm → /run/credentials/systemd-tmpfiles-setup-dev-early.service (MS_MOVE "")...
    (enerator)[68]: (sd-mkdcreds) succeeded.
    (tmpfiles)[71]: (sd-mkdcreds) succeeded.
    (udevadm)[73]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (udevadm)[73]: Successfully forked off '(sd-mkdcreds)' as PID 78.
    (tmpfiles)[71]: systemd-tmpfiles-setup-dev-early.service: Kernel keyring not supported, ignoring.
    (sd-mkdcre[77]: Mounting ramfs (ramfs) on /dev/shm (MS_NOSUID|MS_NODEV|MS_NOEXEC|MS_NOSYMFOLLOW "mode=0700")...
    (sd-mkdcre[77]: Credential search path is: /etc/credstore:/run/credstore:/usr/local/lib/credstore:/usr/lib/credstore
    (sd-mkdcr[78]: Changing mount propagation /dev (MS_REC|MS_SLAVE "")
    (enerator)[68]: systemd-network-generator.service: Kernel keyring not supported, ignoring.
    (tmpfiles)[71]: systemd-tmpfiles-setup-dev-early.service: Executing: systemd-tmpfiles --prefix=/dev --create --boot --graceful

    (udevadm)[74]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (sd-mkdcre[77]: Credential search path is: /etc/credstore.encrypted:/run/credstore.encrypted:/usr/local/lib/credstore.encrypted:/usr/lib/credstore.encrypted
    (sd-mkdcr[78]: Mounting ramfs (ramfs) on /dev/shm (MS_NOSUID|MS_NODEV|MS_NOEXEC|MS_NOSYMFOLLOW "mode=0700")...
    (sd-mkdcre[77]: Changing mount flags /dev/shm (MS_RDONLY|MS_NOSUID|MS_NODEV|MS_NOEXEC|MS_REMOUNT|MS_NOSYMFOLLOW|MS_BIND "")...
    (enerator)[68]: systemd-network-generator.service: Executing: /usr/lib/systemd/systemd-network-generator
    (sd-mkdcr[78]: Credential search path is: /etc/credstore:/run/credstore:/usr/local/lib/credstore:/usr/lib/credstore
    [  OK  ] Finished Load Kernel Module fuse.
    (sd-mkdcr[78]: Credential search path is: /etc/credstore.encrypted:/run/credstore.encrypted:/usr/local/lib/credstore.encrypted:/usr/lib/credstore.encrypted
    systemd-journald[66]: Collecting audit messages is disabled.
    (sd-mkdcr[78]: Credential search path is: /etc/credstore:/run/credstore:/usr/local/lib/credstore:/usr/lib/credstore
    (sd-mkdcr[78]: Credential search path is: /etc/credstore.encrypted:/run/credstore.encrypted:/usr/local/lib/credstore.encrypted:/usr/lib/credstore.encrypted
    (sd-mkdcr[78]: Changing mount flags /dev/shm (MS_RDONLY|MS_NOSUID|MS_NODEV|MS_NOEXEC|MS_REMOUNT|MS_NOSYMFOLLOW|MS_BIND "")...
    (sd-mkdcr[78]: Moving mount /dev/shm → /run/credentials/systemd-udev-load-credentials.service (MS_MOVE "")...
    (sd-mkdcre[77]: Moving mount /dev/shm → /run/credentials/systemd-sysctl.service (MS_MOVE "")...
    (udevadm)[74]: Found cgroup2 on /sys/fs/cgroup/, full unified hierarchy
    (d-sysctl)[70]: (sd-mkdcreds) succeeded.
    (d-sysctl)[70]: systemd-sysctl.service: Kernel keyring not supported, ignoring.
    (udevadm)[73]: (sd-mkdcreds) succeeded.
    systemd-journald[66]: Failed to install memory pressure event source, ignoring: Operation not supported
    (d-sysctl)[70]: systemd-sysctl.service: Executing: /usr/lib/systemd/systemd-sysctl
    (udevadm)[73]: systemd-udev-load-credentials.service: Kernel keyring not supported, ignoring.
    (udevadm)[74]: systemd-udev-trigger.service: Kernel keyring not supported, ignoring.
    (udevadm)[74]: systemd-udev-trigger.service: Executing: udevadm trigger --type=all --action=add --prioritized-subsystem=module,block,tpmrm,net,tty,input
    (udevadm)[73]: systemd-udev-load-credentials.service: Executing: udevadm control --load-credentials
    systemd-journald[66]: Fixed min_use=14.7M max_use=73.8M max_size=9.2M min_size=512K keep_free=36.9M n_max_files=100
    systemd-journald[66]: Reserving 333 entries in field hash table.
    systemd-journald[66]: Reserving 16796 entries in data hash table.
    systemd-journald[66]: Journal effective settings seal=no keyed_hash=yes compress=NONE compress_threshold_bytes=512B
    [  OK  ] Finished Generate network units from Kernel command line.
    systemd-journald[66]: Vacuuming...
    [  OK  ] Reached target Preparation for Network.
    systemd-journald[66]: Vacuuming done, freed 0B of archived journals from /run/log/journal/593b29e8e992cd4ad571aea56f72913f.
    systemd-journald[66]: Flushing /dev/kmsg...
    [  OK  ] Finished Load udev Rules from Credentials.
    [  OK  ] Finished Remount Root and Kernel File Systems.
    [  OK  ] Finished Apply Kernel Variables.
    systemd-journald[66]: systemd-journald running as PID 66 for the system.
    systemd-journald[66]: Sent READY=1 notification.
    systemd-journald[66]: Sent WATCHDOG=1 notification.
    [  OK  ] Started Journal Service.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
            Starting Flush Journal to Persistent Storage...
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    [  OK  ] Finished Create Static Device Nodes in /dev gracefully.
    systemd-journald[66]: varlink: New incoming connection.
    systemd-journald[66]: varlink-17: Setting state idle-server
            Starting Create Static Device Nodes in /dev...
    systemd-journald[66]: varlink-17: Received message: {"method":"io.systemd.Journal.FlushToVar","parameters":{}}
    systemd-journald[66]: varlink-17: Changing state idle-server → processing-method
    systemd-journald[66]: Received client request to flush runtime journal.
    systemd-journald[66]: Fixed min_use=16M max_use=181.4M max_size=22.6M min_size=512K keep_free=90.7M n_max_files=100
    systemd-journald[66]: Reserving 333 entries in field hash table.
    systemd-journald[66]: Reserving 41294 entries in data hash table.
    systemd-journald[66]: Flushing to /var/log/journal/593b29e8e992cd4ad571aea56f72913f...
    systemd-journald[66]: Considering root directory '/run/log/journal'.
    systemd-journald[66]: Root directory /run/log/journal added.
    systemd-journald[66]: Considering directory '/run/log/journal/593b29e8e992cd4ad571aea56f72913f'.
    systemd-journald[66]: Directory /run/log/journal/593b29e8e992cd4ad571aea56f72913f added.
    systemd-journald[66]: Journal effective settings seal=no keyed_hash=yes compress=NONE compress_threshold_bytes=8B
    systemd-journald[66]: File /run/log/journal/593b29e8e992cd4ad571aea56f72913f/system.journal added.
    systemd-journald[66]: Considering root directory '/var/log/journal'.
    systemd-journald[66]: Considering root directory '/var/log/journal/remote'.
    systemd-journald[66]: Root directory /run/log/journal removed.
    systemd-journald[66]: Directory /run/log/journal/593b29e8e992cd4ad571aea56f72913f removed.
    systemd-journald[66]: mmap cache statistics: 37791 category cache hit, 3 window list hit, 1 miss
    systemd-journald[66]: Removed runtime journal directory /run/log/journal/593b29e8e992cd4ad571aea56f72913f.
    systemd-journald[66]: Failed to remove additional runtime journal directory /run/log/journal/593b29e8e992cd4ad571aea56f72913f, ignoring: No such file or directory
    systemd-journald[66]: Vacuuming...
    systemd-journald[66]: Vacuuming done, freed 0B of archived journals from /var/log/journal/593b29e8e992cd4ad571aea56f72913f.
    systemd-journald[66]: varlink-17: Sending message: {"parameters":{}}
    systemd-journald[66]: varlink-17: Changing state processing-method → processed-method
    systemd-journald[66]: varlink-17: Changing state processed-method → idle-server
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: varlink-17: Got POLLHUP from socket.
    systemd-journald[66]: varlink-17: Changing state idle-server → pending-disconnect
    systemd-journald[66]: varlink-17: Changing state pending-disconnect → processing-disconnect
    systemd-journald[66]: varlink-17: Changing state processing-disconnect → disconnected
    [  OK  ] Finished Flush Journal to Persistent Storage.
    [  OK  ] Finished Create Static Device Nodes in /dev.
    [  OK  ] Reached target Preparation for Local File Systems.
    [  OK  ] Reached target Local File Systems.
            Starting Create System Files and Directories...
            Starting Rule-based Manager for Device Events and Files...
            Starting Update is Completed...
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    [  OK  ] Finished Update is Completed.
    [  OK  ] Started Rule-based Manager for Device Events and Files.
            Starting Network Configuration...
    [  OK  ] Finished Create System Files and Directories.
            Starting Network Name Resolution...
            Starting Network Time Synchronization...
            Starting Record System Boot/Shutdown in UTMP...
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    [  OK  ] Finished Record System Boot/Shutdown in UTMP.
    systemd[1]: dev-ram0.device: Changed dead -> plugged
    systemd[1]: systemd-networkd.socket: Incoming traffic
    systemd[1]: dev-ttyAMA0.device: Job 73 dev-ttyAMA0.device/start finished, result=done
    [  OK  ] Found device /dev/ttyAMA0.
    systemd[1]: sys-subsystem-net-devices-sit0.device: Changed dead -> plugged
    [  OK  ] Started Network Configuration.
            Starting Enable Persistent Storage in systemd-networkd...
    [  OK  ] Started Network Time Synchronization.
    [  OK  ] Reached target System Time Set.
    [FAILED] Failed to start Network Name Resolution.
    See 'systemctl status systemd-resolved.service' for details.
    systemd[1]: nss-lookup.target: Trying to enqueue job nss-lookup.target/start/fail
    systemd[1]: varlink-52: Changing state processed-method → idle-server
            Starting Network Name Resolution...
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    [  OK  ] Finished Coldplug All udev Devices.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Failed to get user.journald_log_filter_patterns xattr for /system.slice/systemd-networkd-persistent-storage.service: No such file or directory
    [  OK  ] Finished Enable Persistent Storage in systemd-networkd.
    systemd[1]: init.scope: Child 127 belongs to init.scope.
    [FAILED] Failed to start Network Name Resolution.
    See 'systemctl status systemd-resolved.service' for details.
            Starting Network Name Resolution...
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd[1]: ttya1: Got 'add' action on syspath '/sys/devices/virtual/tty/ttya1'.
    systemd[1]: ttya2: Got 'add' action on syspath '/sys/devices/virtual/tty/ttya2'.
    systemd[1]: sys-devices-virtual-tty-ttya6.device: Changed dead -> plugged
    systemd[1]: dev-ttya5.device: Changed dead -> plugged
    systemd[1]: ttya7: Got 'add' action on syspath '/sys/devices/virtual/tty/ttya7'.
    systemd[1]: dev-ttya7.device: Changed dead -> plugged
    systemd[1]: dev-ttyaa.device: Changed dead -> plugged
    systemd[1]: dev-ttyab.device: Changed dead -> plugged
    systemd[1]: ttya8: Got 'add' action on syspath '/sys/devices/virtual/tty/ttya8'.
    systemd[1]: dev-ttyad.device: Changed dead -> plugged
    systemd[1]: systemd-resolved.service: Got notification message from PID 131 (ERRNO=92)
    systemd[1]: systemd-resolved.service: Child 131 belongs to systemd-resolved.service.
    [FAILED] Failed to start Network Name Resolution.
    See 'systemctl status systemd-resolved.service' for details.
    systemd[1]: dev-ttybd.device: Changed dead -> plugged
    systemd[1]: ttyc2: Processing udev action (SEQNUM=1207, ACTION=add)
    systemd[1]: dev-ttyeb.device: Changed dead -> plugged
            Starting Network Name Resolution...
            Starting Virtual Console Setup...
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    [  OK  ] Finished Virtual Console Setup.
    [FAILED] Failed to start Network Name Resolution.
    See 'systemctl status systemd-resolved.service' for details.
            Starting Network Name Resolution...
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    [FAILED] Failed to start Network Name Resolution.
    See 'systemctl status systemd-resolved.service' for details.
            Starting Network Name Resolution...
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    [FAILED] Failed to start Network Name Resolution.
    See 'systemctl status systemd-resolved.service' for details.
            Starting Network Name Resolution...
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    [FAILED] Failed to start Network Name Resolution.
    See 'systemctl status systemd-resolved.service' for details.
            Starting Network Name Resolution...
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    systemd-journald[66]: Data hash table of /var/log/journal/593b29e8e992cd4ad571aea56f72913f/system.journal has a fill level at 75.0 (30971 of 41294 items, 16777216 file size, 541 bytes per hash table item), suggesting rotation.
    systemd-journald[66]: /var/log/journal/593b29e8e992cd4ad571aea56f72913f/system.journal: Journal header limits reached or header out-of-date, rotating.
    systemd-journald[66]: Reserving 333 entries in field hash table.
    systemd-journald[66]: Reserving 41294 entries in data hash table.
    systemd-journald[66]: Journal effective settings seal=no keyed_hash=yes compress=NONE compress_threshold_bytes=512B
    systemd-journald[66]: Vacuuming...
    systemd-journald[66]: Vacuuming done, freed 0B of archived journals from /var/log/journal/593b29e8e992cd4ad571aea56f72913f.
    [FAILED] Failed to start Network Name Resolution.
    See 'systemctl status systemd-resolved.service' for details.
            Starting Network Name Resolution...
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    [FAILED] Failed to start Network Name Resolution.
    See 'systemctl status systemd-resolved.service' for details.
            Starting Network Name Resolution...
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    [FAILED] Failed to start Network Name Resolution.
    See 'systemctl status systemd-resolved.service' for details.
    [FAILED] Failed to start Network Name Resolution.
    See 'systemctl status systemd-resolved.service' for details.
    [  OK  ] Reached target Network.
    [  OK  ] Reached target Host and Network Name Lookups.
    [  OK  ] Reached target System Initialization.
    [  OK  ] Started Daily Cleanup of Temporary Directories.
    [  OK  ] Reached target Timer Units.
    [  OK  ] Listening on D-Bus System Message Bus Socket.
    [  OK  ] Listening on Hostname Service Socket.
    [  OK  ] Reached target Socket Units.
    [  OK  ] Reached target Basic System.
            Starting D-Bus System Message Bus...
    [  OK  ] Started Serial Getty on ttyAMA0.
    [  OK  ] Reached target Login Prompts.
    systemd-journald[66]: Successfully sent stream file descriptor to service manager.
    [  OK  ] Started D-Bus System Message Bus.
    [  OK  ] Reached target Multi-User System.

    Welcome to Buildroot
    buildroot login: root (automatic login)