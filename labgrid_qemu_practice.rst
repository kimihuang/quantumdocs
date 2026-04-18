Labgrid 自动化测试 QEMU 实践经验
====================================

本文档记录了使用 Labgrid 搭建 QEMU aarch64 自动化测试的完整实践过程，涵盖环境搭建、
测试时间优化、多文件测试组织、报告生成等关键经验。

适用环境：Labgrid 25.x + pytest + QEMU 8.x + Buildroot rootfs

环境搭建
--------

核心配置文件 ``labgrid-env.yaml``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Labgrid 通过 YAML 定义 target，关键是正确区分 driver 和 resource。
QEMU 必须作为 driver（``QEMUDriver``），不能作为 resource。

最小可用配置：

.. code-block:: yaml

    targets:
      main:
        drivers:
          QEMUDriver:
            qemu_bin: 'qemu_aarch64'
            machine: 'virt'
            cpu: 'cortex-a57'
            memory: '512M'
            display: 'none'
            kernel: 'kernel'
            dtb: 'dtb'
            extra_args: '-initrd /path/to/rootfs.cpio'
            boot_args: 'console=ttyAMA0 init=/init'
          ShellDriver:
            prompt: '[#\\$] '
            login_prompt: ' login: '
            username: 'root'
            await_login_timeout: 3
            login_timeout: 120
          ShellStrategy: {}

    tools:
      qemu_aarch64: 'qemu-system-aarch64'

    images:
      kernel: '/path/to/Image'
      dtb: '/path/to/custom.dtb'

几个容易踩的坑：

.. list-table:: 配置常见问题
   :widths: 30 70
   :header-rows: 1

   * - 问题
     - 解决方案
   * - ``$BASE`` 变量不替换
     - 使用绝对路径，Labgrid YAML 不支持 ``$BASE`` 等环境变量
   * - ``ShellDriver.timeout`` 报 unsupported
     - 该属性在 v25 中不存在，使用 ``await_login_timeout`` 和 ``login_timeout``
   * - ``display: 'none'`` 不生效
     - 实际效果是 ``-nographic``，Labgrid 会自动处理
   * - 缺少 initrd 导致 kernel panic
     - 通过 ``extra_args`` 传入 ``-initrd /path/to/rootfs.cpio``

Driver 激活顺序
~~~~~~~~~~~~~~~~

QEMUDriver 和 ShellDriver 的激活顺序必须正确，否则会出现 ``_clientsocket is None`` 错误。

正确顺序：

.. code-block:: python

    qemu = target.get_driver(QEMUDriver, activate=False)
    shell = target.get_driver(ShellDriver, activate=False)

    target.activate(qemu)   # 1. 创建 unix socket
    qemu.on()               # 2. 启动 QEMU 进程，建立 socket 连接
    target.activate(shell)  # 3. ShellDriver 绑定 console（此时 socket 已就绪）
    shell._await_login()    # 4. 等待 shell prompt

.. note::

    如果先 ``activate(shell)`` 再 ``qemu.on()``，ShellDriver 会尝试读取尚未创建的
    socket，抛出 ``TypeError: argument must be an int, or have a fileno() method``。

测试时间优化
------------

从 82 秒优化到 15 秒的关键调整。

问题定位
~~~~~~~~

初始版本的 fixture 中有两段硬编码等待：

.. code-block:: python

    time.sleep(5)    # 等内核 boot
    # drain output...
    time.sleep(10)   # 等 init 脚本

    总计：15 秒静态等待 + await_login 内部超时 = 约 82 秒

根本原因不是 sleep 本身，而是 ``_await_login`` 的 ``await_login_timeout`` 参数。

await_login_timeout 的影响
~~~~~~~~~~~~~~~~~~~~~~~~~~~

ShellDriver 的 ``_await_login`` 方法通过循环 expect 等待 shell prompt，每次 expect
的超时由 ``await_login_timeout`` 控制（默认 2 秒）。

工作原理：

1. 每次 expect 超时后，检查 ``before`` 内容是否与上次相同
2. 如果相同（说明 console 空闲），发送空行探测 prompt
3. 如果不同（说明有新输出），继续下一轮 expect

Boot 期间串口持续输出，所以 ``before`` 一直变化，"静默检测"永远不触发。
``await_login_timeout=30`` 时，每次循环等 30 秒才超时，导致白白等待。

优化方案
~~~~~~~~

.. list-table:: 优化措施
   :widths: 40 30 30
   :header-rows: 1

   * - 措施
     - 优化前
     - 优化后
   * - ``await_login_timeout``
     - 30s（默认）
     - 3s
   * - ``time.sleep(5)``
     - 有
     - 删除
   * - ``time.sleep(10)``
     - 有
     - 删除
   * - 总耗时
     - ~82s
     - ~15s

将 ``await_login_timeout`` 设为 3 秒，让静默检测快速触发空行探测。
Boot 过程中只要有短暂的输出间隙（超过 3 秒），就会发空行并匹配到 ``# `` prompt。

优化后的 conftest.py：

.. code-block:: python

    @pytest.fixture(scope="session")
    def qemu_env(target):
        qemu = target.get_driver(QEMUDriver, activate=False)
        shell = target.get_driver(ShellDriver, activate=False)

        target.activate(qemu)
        qemu.on()

        # drain boot output until console is idle
        while True:
            ready, _, _ = select.select([qemu._clientsocket], [], [], 2)
            if not ready:
                break
            qemu._clientsocket.recv(4096)

        target.activate(shell)
        shell._await_login()

        yield target
        qemu.off()

多文件测试组织
--------------

使用 ``conftest.py`` + 多个 ``test_*.py`` 的标准 pytest 结构。
pytest 自动发现所有测试文件，session 级 fixture 确保只启动一次 QEMU。

目录结构：

.. code-block:: text

    labgrid/
    ├── conftest.py          # 共享 fixture（qemu_env）
    ├── labgrid-env.yaml     # 环境配置
    ├── run_report.sh        # 报告生成脚本
    ├── test_monitor.py      # QMP Monitor 测试（3 cases）
    ├── test_system.py       # 系统信息测试（7 cases）
    ├── test_filesystem.py   # 文件系统测试（4 cases）
    └── test_process.py      # 进程服务测试（6 cases）

conftest.py 核心
~~~~~~~~~~~~~~~~

``qemu_env`` fixture 使用 ``scope="session"``，所有测试用例共享同一个 QEMU 实例。

同时通过 Python logging 记录 console 输出，用于 boot 日志排查：

.. code-block:: python

    logger = logging.getLogger("labgrid.console")

    # drain boot output
    while True:
        ready, _, _ = select.select([qemu._clientsocket], [], [], 2)
        if not ready:
            break
        data = qemu._clientsocket.recv(4096)
        text = data.decode("utf-8", errors="replace")
        for line in text.splitlines():
            logger.info(line)

BusyBox 兼容性
~~~~~~~~~~~~~~~

Buildroot 的 BusyBox 与 GNU 工具有命令差异，编写测试用例时需注意：

.. list-table:: BusyBox 与 GNU 工具差异
   :widths: 40 40 40
   :header-rows: 1

   * - GNU 用法
     - BusyBox 限制
     - 替代方案
   * - ``ps -p 1 -o comm=``
     - 不支持 ``-p`` 和 ``-o`` 参数
     - ``ps | head -2``
   * - ``mount -t tmpfs none /dir``
     - 可能未编译 tmpfs 支持
     - 直接写 ``/tmp`` 测试读写

查看 Linux 启动日志
--------------------

通过 ``--log-cli-level=INFO`` 让 pytest 输出 ``labgrid.console`` logger 的内容：

.. code-block:: bash

    pytest -v --lg-env labgrid-env.yaml -s --log-cli-level=INFO 2>&1 | tee console.log

同时保存到文件和显示在终端，方便实时观察 boot 过程。

测试报告生成
------------

使用 pytest-html 生成自包含的 HTML 报告。

安装依赖：

.. code-block:: bash

    pip install pytest-html pytest-json-report

报告脚本 ``run_report.sh`` 同时生成三种产物：

.. list-table:: 报告产物
   :widths: 30 70
   :header-rows: 1

   * - 文件
     - 用途
   * - ``report.html``
     - 浏览器打开的交互式看板，包含通过/失败统计、耗时
   * - ``report.json``
     - 机器可读的结构化结果，适合 CI/CD pipeline 解析
   * - ``console.log``
     - 完整控制台输出，包含 QEMU boot 日志

脚本核心逻辑：

.. code-block:: bash

    pytest \
        -v \
        --lg-env labgrid-env.yaml \
        --html=report/report.html \
        --self-contained-html \
        --json-report \
        --json-report-file=report/report.json \
        --tb=short \
        -s \
        2>&1 | tee report/console.log

    # 从 JSON 提取摘要
    python3 -c "
    import json
    with open('report/report.json') as f:
        data = json.load(f)
    summary = data.get('summary', {})
    duration = data.get('duration', 0)
    print(f'Total: {summary[\"total\"]}, Passed: {summary[\"passed\"]}, Time: {duration:.2f}s')
    "

运行方式：

.. code-block:: bash

    cd labgrid
    ./run_report.sh
    xdg-open report/report.html

QMP Monitor 测试
-----------------

QEMUDriver 内置了 QMP（QEMU Machine Protocol）支持，可以发送 monitor 命令。
QMP 通过 ``qemu._child.stdin/stdout`` 通信（``-qmp stdio``），串口通过 unix socket。

常用 QMP 命令：

.. list-table:: QMP 命令
   :widths: 40 60
   :header-rows: 1

   * - 命令
     - 用途
   * - ``query-version``
     - 查询 QEMU 版本号
   * - ``query-status``
     - 查询 VM 运行状态（running/paused）
   * - ``query-cpus-fast``
     - 查询 CPU 拓扑信息
   * - ``human-monitor-command``
     - 执行 HMP 命令（如 ``info registers``）

.. code-block:: python

    qemu = target.get_driver(QEMUDriver)

    result = qemu.monitor_command("query-version")
    print(f"QEMU {result['qemu']['major']}.{result['qemu']['minor']}")

    result = qemu.monitor_command("query-status")
    assert result["status"] == "running"

经验总结
--------

1. **驱动激活顺序是 Labgrid 的核心逻辑**，QEMUDriver 必须先 ``on()`` 再激活 ShellDriver，
   否则 console 读取会失败。

2. **测试耗时的瓶颈往往不是 sleep，而是 await_login 的超时参数**。
   理解其内部"静默检测"机制，通过调整 ``await_login_timeout`` 让 prompt 探测更积极。

3. **BusyBox 环境下编写测试用例要避免 GNU 特有的命令参数**，
   编写后先在 QEMU shell 中手动验证命令可用性。

4. **使用 ``conftest.py`` 的 session fixture 管理全局资源**（QEMU 进程），
   多个 test 文件自动共享，避免重复启动。

5. **报告生成与测试解耦**，``run_report.sh`` 封装了 pytest 参数和报告路径，
   一条命令产出 HTML + JSON + console log 三种格式，满足人工查看和 CI/CD 自动化两种场景。

6. **Console 日志通过 Python logging 而非直接 print**，
   可以用 pytest 的 ``--log-cli-level`` 统一控制，在需要时开启，不需要时静默。
