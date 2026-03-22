QEMU 启动 Linux 内核过程记录
================================

概述
----

本文档记录了使用 QEMU 启动 Linux 内核的过程，包括通过直接启动内核和通过 U-Boot 启动内核两种方式。当前状态是尚未成功启动到 Linux 系统。

环境准备
--------

1. **环境变量设置**：通过 `source build/envsetup.sh` 加载环境变量
2. **板卡配置**：通过 `lunch` 命令选择板卡配置，例如 `lunch 2` 选择 `quantum_qemu_debug` 配置

启动方式一：直接启动 Linux 内核
-------------------------------

使用 `boot_kernel` 函数直接启动 Linux 内核，命令格式：

.. code-block:: bash

    boot_kernel [monitor] [no-memdisk]

### 启动流程

1. **检查板卡配置**：确保已选择板卡配置
2. **解析参数**：处理 `monitor` 和 `no-memdisk` 参数
3. **设置内核镜像路径**：使用 `$BOARD_OUT_DIR/images/Image`
4. **设置根文件系统路径**：使用 `$BOARD_OUT_DIR/images/rootfs.cpio`
5. **设置设备树文件路径**：使用 `$BOARD_OUT_DIR/images/quantum_qemu.dtb`
6. **配置 QEMU 参数**：
   - 机器类型：`$QEMU_MACHINE` (virt)
   - CPU：`$QEMU_CPU` (cortex-a57)
   - SMP：`$QEMU_SMP` (4)
   - 内存：`$QEMU_MEM` (4096M)
   - 内核参数：`$KERNEL_CMDLINE`
7. **启动 QEMU**：执行 `qemu-system-aarch64` 命令

启动方式二：通过 U-Boot 启动 Linux 内核
-----------------------------------

使用 `boot_uboot` 函数启动 U-Boot，然后在 U-Boot shell 中启动 Linux 内核，命令格式：

.. code-block:: bash

    boot_uboot [monitor]

### 启动流程

1. **检查板卡配置**：确保已选择板卡配置
2. **解析参数**：处理 `monitor` 参数
3. **设置 U-Boot 镜像路径**：使用 `$BOARD_OUT_DIR/images/u-boot.bin`
4. **设置内核镜像路径**：使用 `$BOARD_OUT_DIR/images/Image`
5. **设置根文件系统路径**：使用 `$BOARD_OUT_DIR/images/rootfs.cpio`
6. **设置设备树文件路径**：使用 `$BOARD_OUT_DIR/images/quantum_qemu.dtb`
7. **加载文件到内存**：
   - Linux 内核：加载到 0x41000000
   - 根文件系统：加载到 0x42000000
   - 设备树：加载到 0x45000000
8. **启动 QEMU**：执行 `qemu-system-aarch64` 命令，使用 `-bios` 参数指定 U-Boot 镜像
9. **在 U-Boot shell 中启动 Linux**：
   .. code-block:: bash

       => setenv bootargs $KERNEL_CMDLINE
       => booti 0x41000000 0x42000000 0x45000000

当前状态
--------

**尚未成功启动到 Linux 系统**，遇到以下问题：

1. **U-Boot 启动过程中的警告**：
   .. code-block:: bash

       Bloblist at 0 not found (err=-2)
       alloc space exhausted ptr 400 limit 0

2. **根文件系统镜像格式错误**：
   .. code-block:: bash

       Wrong Ramdisk Image Format
       Ramdisk image is corrupt or invalid

3. **内核启动失败**：
   .. code-block:: bash

       Hardware name: linux,dummy-virt (DT)
       pstate: 60000005 (nZCv daif -PAN -UAO -TCO -DIT -SSBS BTYPE=--)
       pc : amba_read_periphid+0x94/0x1cc
       lr : amba_read_periphid+0x80/0x1cc
       ...
       Kernel panic - not syncing: Attempted to kill init! exitcode=0x0000000b

文件结构
--------

- **`build/qemu.sh`**：包含 QEMU 启动相关函数
  - `qemu_monitor`：连接到 QEMU monitor
  - `boot_kernel`：直接启动 Linux 内核
  - `boot_uboot`：启动 U-Boot，然后在 U-Boot shell 中启动 Linux 内核

- **镜像文件路径**：
  - U-Boot 镜像：`$BOARD_OUT_DIR/images/u-boot.bin`
  - Linux 内核镜像：`$BOARD_OUT_DIR/images/Image`
  - 根文件系统镜像：`$BOARD_OUT_DIR/images/rootfs.cpio`
  - 设备树文件：`$BOARD_OUT_DIR/images/quantum_qemu.dtb`

内核命令行
----------

.. code-block:: bash

    console=ttyAMA0 root=/dev/ram0 rw init=/linuxrc nokaslr earlycon=pl011,0x9000000 debug loglevel=8 MEMDISK_PHY_ADDR=0x78000000 MEMDISK_SIZE=268435456

尝试的解决方案
------------

1. **修复 `load_args` 赋值逻辑**：使用 `+=` 运算符确保所有设备加载参数都被正确累积
2. **调整内存加载地址**：
   - 尝试使用 0x40000000 系列地址
   - 尝试使用 0x80000000 系列地址
3. **确保设备树文件正确加载**：取消设备树加载代码的注释
4. **更新 U-Boot shell 启动命令**：确保使用正确的内存地址

结论
----

当前尚未成功启动到 Linux 系统，需要进一步调试和解决以下问题：

1. U-Boot 启动过程中的警告信息
2. 根文件系统镜像格式错误
3. 内核启动失败的原因

建议的下一步操作：

1. 检查根文件系统镜像的格式和完整性
2. 验证设备树文件是否正确
3. 调整 QEMU 启动参数和内存地址
4. 查看更详细的内核启动日志，定位具体的失败原因
