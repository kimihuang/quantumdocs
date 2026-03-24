Labgrid for QEMU 环境搭建经验总结
===================================

本文档总结了在 Quantum 项目中使用 Labgrid 进行 QEMU 测试环境搭建过程中遇到的问题和解决方案。

概述
----

Labgrid 是一个用于嵌入式系统测试的 Python 框架，可以管理各种硬件资源和驱动程序。在 Quantum 项目中，我们使用 Labgrid 来自动化 QEMU 虚拟机的测试流程。

安装 Labgrid
------------

使用 Buildroot 集成安装
~~~~~~~~~~~~~~~~~~~~~~

在 ``board/quantum/br2_external/package/labgrid/labgrid.mk`` 中定义 Labgrid 包：

.. code-block:: makefile

    LABGRID_VERSION = 1.0
    LABGRID_SITE = $(LABGRID_PKGDIR)
    LABGRID_SITE_METHOD = local

    define LABGRID_BUILD_CMDS
        PYTHON_VENV_DIR=$(TOPDIR)/../../../.venv; \
        if [ ! -d "$$PYTHON_VENV_DIR" ]; then \
            python3 -m venv $$PYTHON_VENV_DIR; \
        fi; \
        $$PYTHON_VENV_DIR/bin/pip install --upgrade pip; \
        $$PYTHON_VENV_DIR/bin/pip install labgrid pytest pytest-json
    endef

    $(eval $(generic-package))

在 ``board/quantum/br2_external/Config.in`` 中添加配置选项：

.. code-block:: text

    config BR2_PACKAGE_LABGRID
        bool "labgrid"
        help
          Labgrid - testing framework for embedded systems

手动安装（用于快速测试）
~~~~~~~~~~~~~~~~~~~~~~~~

如果需要快速安装和测试，可以直接在项目根目录创建虚拟环境：

.. code-block:: bash

    python3 -m venv .venv
    .venv/bin/pip install --upgrade pip
    .venv/bin/pip install labgrid pytest pytest-json

Labgrid 配置文件
----------------

配置文件位置
~~~~~~~~~~~~

Labgrid 配置文件位于 ``tests/labgrid/config.yaml``，定义了测试目标和环境配置。

常见配置错误及解决方案
~~~~~~~~~~~~~~~~~~~~~~

错误 1：未知的 resource 类 QEMU
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**错误信息**：

.. code-block:: text

    labgrid.exceptions.InvalidConfigError: unknown resource class QEMU

**原因**：

在配置文件中将 QEMU 误定义为 resource，但实际上 QEMU 应该作为 driver 使用。

**错误配置**：

.. code-block:: yaml

    targets:
      quantum-qemu:
        resources:
          QEMU:              # 错误：QEMU 不是 resource 类
            image: "${OUTPUT_DIR}/quantum_qemu_debug/images/Image"
            ...

**正确配置**：

.. code-block:: yaml

    targets:
      quantum-qemu:
        resources:
          RawSerialPort:
            port: 5000
        drivers:
          QEMUDriver:        # 正确：使用 QEMUDriver 作为 driver
            qemu_bin: "/usr/bin/qemu-system-aarch64"
            ...

错误 2：QEMUDriver 缺少必需参数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**错误信息**：

.. code-block:: text

    TypeError: QEMUDriver.__init__() missing 1 required positional argument: 'qemu_bin'

**原因**：

QEMUDriver 需要的参数名称与预期不同。

**错误配置**：

.. code-block:: yaml

    drivers:
      QEMUDriver:
        image: "${OUTPUT_DIR}/quantum_qemu_debug/images/Image"     # 错误参数名
        initrd: "${OUTPUT_DIR}/quantum_qemu_debug/images/rootfs.cpio"  # 错误参数名
        options:                                                      # 错误参数名
          - "-nographic"

**正确配置**：

.. code-block:: yaml

    drivers:
      QEMUDriver:
        qemu_bin: "/usr/bin/qemu-system-aarch64"     # 必需参数
        kernel: "${OUTPUT_DIR}/quantum_qemu_debug/images/Image"        # 正确参数名
        rootfs: "${OUTPUT_DIR}/quantum_qemu_debug/images/rootfs.cpio"   # 正确参数名
        dtb: "${OUTPUT_DIR}/quantum_qemu_debug/images/quantum_qemu.dtb"
        machine: "virt"
        cpu: "cortex-a76"
        memory: "1024M"
        extra_args: "-nographic -serial tcp:127.0.0.1:5000,server,nowait"  # 正确参数名

**参数对照表**：

+----------------------+------------------+--------------------------------+
| 错误参数名           | 正确参数名       | 说明                           |
+======================+==================+================================+
| image                | kernel           | Linux 内核镜像文件             |
+----------------------+------------------+--------------------------------+
| initrd              | rootfs           | 根文件系统                     |
+----------------------+------------------+--------------------------------+
| options              | extra_args       | 额外的 QEMU 命令行参数         |
+----------------------+------------------+--------------------------------+
| -                    | qemu_bin         | QEMU 可执行文件路径（必需）    |
+----------------------+------------------+--------------------------------+

错误 3：RawSerialPort 使用了不支持的属性
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**警告信息**：

.. code-block:: text

    UserWarning: unsupported attribute 'host' with value '127.0.0.1' for class 'RawSerialPort'

**原因**：

RawSerialPort 资源只接受 ``port`` 参数，不接受 ``host`` 参数。

**错误配置**：

.. code-block:: yaml

    resources:
      RawSerialPort:
        host: 127.0.0.1    # 不支持的属性
        port: 5000

**正确配置**：

.. code-block:: yaml

    resources:
      RawSerialPort:
        port: 5000          # 只需要端口号

错误 4：ShellDriver 依赖错误
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**错误信息**：

.. code-block:: text

    labgrid.exceptions.BindingError: supplier for console of ShellDriver in ... requires an explicit name

**原因**：

ShellDriver 需要 ConsoleProtocol，但配置中没有正确定义 ConsoleProtocol。

**解决方案**：

对于简单的串口测试，可以不使用 ShellDriver，直接使用 SerialDriver 即可。

**推荐配置**：

.. code-block:: yaml

    drivers:
      SerialDriver:
        txdelay: 0.0
      # 移除 ShellDriver 和 ConsoleProtocol

Python 测试脚本常见问题
------------------------

问题 1：导入错误的驱动类
~~~~~~~~~~~~~~~~~~~~~~~~

**错误信息**：

.. code-block:: text

    ImportError: cannot import name 'ConsoleProtocol' from 'labgrid.driver'

**原因**：

尝试从 ``labgrid.driver`` 导入不存在的类。ConsoleProtocol 是一个协议类，不在 driver 模块中。

**错误代码**：

.. code-block:: python

    from labgrid.driver import ConsoleProtocol, ShellDriver

**正确代码**：

.. code-block:: python

    from labgrid.driver import SerialDriver

问题 2：QEMU 进程未正确清理
~~~~~~~~~~~~~~~~~~~~~~~~~~~

**问题**：

测试执行后 QEMU 进程仍然运行，影响后续测试。

**解决方案**：

在测试结束前添加清理函数：

.. code-block:: python

    def terminate_qemu_processes(self):
        """Terminate all QEMU processes"""
        print("Checking for QEMU processes...")
        result = subprocess.run(['ps', 'aux'], capture_output=True, text=True)
        qemu_processes = [line for line in result.stdout.split('\n')
                         if 'qemu-system-aarch64' in line]

        if qemu_processes:
            print(f"Found QEMU processes: {len(qemu_processes)}")
            for process in qemu_processes:
                pid = process.split()[1]
                print(f"Terminating QEMU process {pid}...")
                try:
                    subprocess.run(['kill', pid], check=True)
                except Exception as e:
                    print(f"Error terminating process {pid}: {e}")

            time.sleep(5)

在 pytest fixture 中使用清理：

.. code-block:: python

    def test_all_cases(self, target, test_cases):
        try:
            # 执行测试逻辑
            pass
        finally:
            # 确保清理 QEMU 进程
            self.terminate_qemu_processes()

完整的配置示例
--------------

以下是一个完整的 ``config.yaml`` 配置示例：

.. code-block:: yaml

    # Labgrid configuration for Quantum project

    targets:
      quantum-qemu:
        resources:
          NetworkService:
            address: 127.0.0.1
            port: 22
            username: root
          RawSerialPort:
            port: 5000
        drivers:
          QEMUDriver:
            qemu_bin: "/usr/bin/qemu-system-aarch64"
            kernel: "${OUTPUT_DIR}/quantum_qemu_debug/images/Image"
            dtb: "${OUTPUT_DIR}/quantum_qemu_debug/images/quantum_qemu.dtb"
            rootfs: "${OUTPUT_DIR}/quantum_qemu_debug/images/rootfs.cpio"
            machine: "virt"
            cpu: "cortex-a76"
            memory: "1024M"
            extra_args: "-nographic -serial tcp:127.0.0.1:5000,server,nowait"
          SerialDriver:
            txdelay: 0.0

    # Test environment configuration
    environment:
      QEMU_BIN: "/usr/bin/qemu-system-aarch64"
      TEST_DIR: "${PROJECT_ROOT}/tests"
      OUTPUT_DIR: "${PROJECT_ROOT}/out"
      PROJECT_ROOT: "/home/lion/workdir/sourcecode/quantum_main"

测试脚本框架
------------

基本的测试脚本结构：

.. code-block:: python

    #!/usr/bin/env python3
    """
    Quantum project test script using labgrid
    """

    import sys
    import os
    import time
    import pytest
    import subprocess
    import json

    from labgrid import Environment
    from labgrid.driver import SerialDriver

    class TestQuantum:
        """Test class for Quantum project"""

        @pytest.fixture(scope="class")
        def environment(self):
            """Create labgrid environment"""
            config_path = os.path.join(os.path.dirname(__file__), "config.yaml")
            return Environment(config_path)

        @pytest.fixture(scope="class")
        def target(self, environment):
            """Get target from environment"""
            return environment.get_target("quantum-qemu")

        def run_boot_kernel(self):
            """Run boot_kernel command"""
            # 实现启动逻辑
            pass

        def connect_to_console(self, target):
            """Connect to QEMU console via SerialDriver"""
            try:
                serial = target.get_driver(SerialDriver)
                serial.activate()
                return serial
            except Exception as e:
                print(f"Warning: Failed to connect to console: {e}")
                return None

        def terminate_qemu_processes(self):
            """Terminate all QEMU processes"""
            # 实现清理逻辑
            pass

        def test_all_cases(self, target, test_cases):
            """Run all test cases from JSON configuration"""
            try:
                # 执行测试
                pass
            finally:
                # 清理资源
                self.terminate_qemu_processes()

运行测试
--------

使用虚拟环境运行测试：

.. code-block:: bash

    PYTHON_VENV_DIR=/path/to/.venv
    $PYTHON_VENV_DIR/bin/python tests/labgrid/test_quantum.py -v

生成 JSON 报告：

.. code-block:: bash

    PYTHON_VENV_DIR=/path/to/.venv
    $PYTHON_VENV_DIR/bin/python tests/labgrid/test_quantum.py --json test_report.json

调试技巧
--------

1. **检查 Labgrid 类和参数**：

   使用 Python 检查类定义和参数：

   .. code-block:: bash

       $PYTHON_VENV_DIR/bin/python -c "from labgrid.driver import QEMUDriver; import inspect; sig = inspect.signature(QEMUDriver.__init__); print(sig)"

2. **查看可用资源类**：

   .. code-block:: bash

       $PYTHON_VENV_DIR/bin/python -c "from labgrid import resource; import inspect; print([name for name, obj in inspect.getmembers(resource) if inspect.isclass(obj)])"

3. **查看可用驱动类**：

   .. code-block:: bash

       $PYTHON_VENV_DIR/bin/python -c "from labgrid import driver; import inspect; print([name for name, obj in inspect.getmembers(driver) if inspect.isclass(obj) and not name.startswith('_')])"

4. **验证配置文件**：

   在加载配置时 Labgrid 会给出警告信息，关注这些警告可以帮助定位配置问题。

最佳实践
---------

1. **使用虚拟环境**：避免污染系统 Python 环境

2. **清理资源**：测试结束后总是清理 QEMU 进程和其他资源

3. **参数验证**：在运行测试前先使用 Python 检查驱动和资源类的参数定义

4. **模块化配置**：将环境变量（如路径）放在 environment 部分，便于修改

5. **错误处理**：使用 try-except 捕获异常，并在 finally 中清理资源

6. **日志记录**：在关键步骤添加日志输出，便于调试

总结
----

Labgrid 是一个强大的测试框架，但在配置和使用时需要注意以下几点：

- 区分 resource 和 driver：QEMU 应该配置为 driver，不是 resource
- 检查参数名称：不同版本的 Labgrid 参数名可能不同，务必查阅文档或使用 inspect 检查
- 正确的依赖关系：某些 driver 需要特定的 protocol 或 resource
- 资源清理：测试结束后记得清理 QEMU 进程等资源

通过遵循本文档的经验和建议，可以避免常见的配置错误，快速搭建 Labgrid 测试环境。
