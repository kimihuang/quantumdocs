CodeBuddy CLI 用户问题处理流程与高效使用指南
===============================================

本文档分析 CodeBuddy CLI 处理用户问题的内部流程，并给出高效使用策略以减少 token 消耗。

1. 处理流程分析
================

1.1 整体流程
------------

.. code-block:: none

   用户输入（user_query）
     │
     ▼
   ┌─────────────────────────────────────┐
   │  系统提示注入（System Prompt）       │
   │  ├─ CODEBUDDY.md 项目指令           │
   │  ├─ 工具定义（Read/Write/Edit/...）  │
   │  ├─ 环境变量（工作目录/平台/日期）   │
   │  └─ 记忆文件（MEMORY.md + topics）   │
   └─────────────────┬───────────────────┘
                     │
                     ▼
   ┌─────────────────────────────────────┐
   │  上下文窗口构建                      │
   │  ├─ 系统提示（固定，较大）           │
   │  ├─ 对话历史（所有轮次的完整记录）    │
   │  ├─ 工具调用结果（文件内容/命令输出）│
   │  └─ 自动摘要（超长时压缩旧对话）     │
   └─────────────────┬───────────────────┘
                     │
                     ▼
   ┌─────────────────────────────────────┐
   │  模型推理                            │
   │  ├─ 理解用户意图                     │
   │  ├─ 决定是否需要工具调用             │
   │  ├─ 选择工具和参数                   │
   │  └─ 生成回复文本                     │
   └─────────────────┬───────────────────┘
                     │
                     ▼
   ┌─────────────────────────────────────┐
   │  工具执行（如需要）                   │
   │  ├─ Read 文件 → 内容注入上下文       │
   │  ├─ Grep/Glob → 匹配结果注入        │
   │  ├─ Bash → 命令输出注入             │
   │  ├─ Agent → 子代理返回结果           │
   │  └─ 多工具并行执行                   │
   └─────────────────┬───────────────────┘
                     │
                     ▼
   ┌─────────────────────────────────────┐
   │  二次推理（工具结果返回后）           │
   │  ├─ 分析工具结果                     │
   │  ├─ 决定是否继续调用工具             │
   │  └─ 生成最终回复                     │
   └─────────────────────────────────────┘

1.2 Token 消耗组成
-------------------

每次交互的 token 消耗 = **输入 token + 输出 token**

.. list-table::
   :header-rows: 1
   :widths: 40 30 30

   * - 组成部分
     - 大小
     - 可控性
   * - 系统提示（含工具定义）
     - 大（约 3000-5000 tokens）
     - 低（框架固定）
   * - CODEBUDDY.md + MEMORY.md
     - 中（约 500-2000 tokens）
     - **高**（用户可精简）
   * - 对话历史（每轮累计）
     - 持续增长
     - **高**（用户控制轮数和内容）
   * - 工具调用结果（文件内容等）
     - 不定（大文件可达数千 tokens）
     - **高**（用户可缩小范围）
   * - 模型输出（回复文本+工具调用）
     - 中
     - 中（简洁指令减少输出）

.. important::

   **对话历史是最大的可变 token 消耗**。每轮交互都会把完整历史重新发送给模型，
   包括所有之前的用户输入、助手回复和工具调用结果。10 轮深度调试后，
   仅历史就可能消耗数万 tokens。

1.3 自动摘要机制
-----------------

当上下文接近窗口上限时，系统会自动压缩旧对话。但：

- 摘要会丢失细节（文件内容、命令输出等）
- 被摘要的内容无法再精确引用
- 频繁触发摘要说明上下文已经过大

2. 高效使用策略
================

2.1 精简输入 — 减少每轮 token
-------------------------------

.. list-table::
   :header-rows: 1
   :widths: 50 50

   * - 低效（token 多）
     - 高效（token 少）
   * - 我发现系统启动的时候 systemd 日志里面有很多 Failed to mount 的错误，securityfs 和 pstore 都挂载失败了，还有 bpf 也失败了，netlink 也失败了， unix socket 也失败了，内核命令行已经加了 systemd.log_level=debug，但是 systemd 最后显示 Too many messages being logged to kmsg, ignoring 然后 Failed to start up manager，请帮我分析一下
     - systemd 启动失败，日志：[粘贴关键错误行]
   * - 请帮我修改 board/quantum/br2_external/configs/buildroot_defconfig 这个文件，把 BR2_ROOTFS_SKELETON_DEFAULT=y 改成 BR2_ROOTFS_SKELETON_INIT_SYSTEMD=y，同时把 BR2_ROOTFS_SKELETON_DEFAULT=y 注释掉
     - 修改 buildroot_defconfig：BR2_ROOTFS_SKELETON_DEFAULT=y → BR2_ROOTFS_SKELETON_INIT_SYSTEMD=y
   * - 我之前在调试 systemd 启动的问题，我们先尝试了在内核命令行中去掉 init=/linuxrc 参数，然后发现系统启动时出现了 Kernel panic - not syncing: Attempted to kill init! exitcode=0x00000004 这样的错误，经过分析发现是 SIGILL 非法指令，原因是 Buildroot defconfig 中的 GCC 编译目标 CPU 设置为 cortex-a76，但 QEMU 使用的是 cortex-a57...
     - [新问题直接提问，不要复述历史]

**原则：** 一句话说明问题 + 提供关键信息（文件路径/错误日志/命令输出），省略背景叙述。

2.2 提供精确路径 — 减少工具搜索轮次
---------------------------------------

.. list-table::
   :header-rows: 1
   :widths: 50 50

   * - 低效
     - 高效
   * - 修改内核 defconfig 中的网络配置
     - 修改 board/quantum/linux/configs/quantum_qemu_defconfig，将 # CONFIG_NET is not set 改为 CONFIG_NET=y
   * - 看看构建脚本在哪里
     - 读取 build/envsetup.sh

当提供精确路径时，模型可以直接调用 Read，跳过 Glob/Grep 搜索阶段，
节省 1-2 轮工具调用（每轮的输入+输出都消耗 token）。

2.3 提供关键上下文 — 减少推测性工具调用
-------------------------------------------

.. list-table::
   :header-rows: 1
   :widths: 50 50

   * - 低效
     - 高效
   * - 编译报错了，帮我看看
     - 编译 arm_pmu.c 报错：implicit declaration of function 'xxx'（第 123 行）
   * - systemd 启动失败了
     - systemd 启动失败，关键日志：Failed to open netlink, ignoring: Function not implemented

提供具体的错误信息可以避免模型猜测问题位置而反复调用工具。

2.4 控制对话深度 — 减少 history token
---------------------------------------

.. list-table::
   :header-rows: 1
   :widths: 50 50

   * - 低效
     - 高效
   * - 在同一个对话中依次调试 3 个不同问题
     - 每个问题用新对话，或用 /clear 清除历史
   * - 20 轮来回调试一个复杂问题
     - 把问题拆成独立子任务，每次只问一个

.. code-block:: text

   # 对话过长时，用 /clear 开始新对话
   /clear

   # 然后提供必要的上下文摘要继续
   我在调试 systemd 启动。当前状态：内核已启用 CONFIG_NET=y，
   但 systemd 仍然报 "Failed to open netlink"。
   请查看 out/quantum_qemu_debug/build/linux-6.1/.config 中
   CONFIG_NET 的实际值。

2.5 善用工具并行 — 减少 API 轮次
---------------------------------

CodeBuddy 支持并行工具调用，一轮可以同时执行多个独立操作。

.. list-table::
   :header-rows: 1
   :widths: 50 50

   * - 低效（3 轮串行）
     - 高效（1 轮并行）
   * - "读取 config1，然后读取 config2，再读取 config3"
     - "并行读取 config1、config2、config3"

用户提示词中明确要求并行时，模型会在一轮中发出多个工具调用，
减少交互轮次和等待时间。

2.6 精简 CODEBUDDY.md 和 MEMORY.md
-------------------------------------

这些文件**每次交互都会注入上下文**，必须精简：

- CODEBUDDY.md：只保留必须遵守的指令，移除注释和冗余说明
- MEMORY.md：保持在 200 行以内（超出会被截断），链接到子文件

.. code-block:: text

   # MEMORY.md 示例（精简版）
   # 项目记忆索引
   - build-commands.md — 编译命令
   - project-architecture.md — 项目架构
   - debugging-patterns.md — 常见调试模式

2.7 限定工具使用范围 — 避免过度探索
---------------------------------------

.. list-table::
   :header-rows: 1
   :widths: 50 50

   * - 低效
     - 高效
   * - 分析这个驱动的所有问题
     - 只分析这个驱动的中断处理部分，只读 arm_pmu.c 第 417-439 行
   * - 优化构建系统
     - 把 board/quantum/br2_external/package/initramfs/initramfs.mk
       中的 6 条 cp 命令合并为一条

限定范围让模型只调用必要的工具，避免读大量无关文件。

2.8 使用 Agent 工具控制上下文
-------------------------------

Agent 工具启动子代理，子代理的上下文与主对话隔离。
子代理完成后只返回摘要结果，不会把所有文件内容注入主对话。

适用场景：

- 代码搜索（"找出所有使用 CONFIG_NET 的文件"）
- 多文件探索（"项目目录结构是什么"）
- 独立子任务（"搜索代码中处理 SIGILL 的地方"）

.. code-block:: text

   # 低效：在主对话中逐个搜索
   "找一下 kernel config 中所有被 disable 的网络选项"
   → 模型在主对话中多次调用 Grep，每次结果都留在 history

   # 高效：用 Agent 子代理
   "用 Agent 搜索 kernel config 中所有被 disable 的网络选项"
   → 子代理内部搜索，只返回汇总结果

3. Token 消耗优化速查表
==========================

.. list-table::
   :header-rows: 1
   :widths: 25 40 35

   * - 策略
     - 方法
     - 节省效果
   * - 精简输入
     - 一句话问题 + 关键信息，省略背景
     - 每轮节省 200-500 tokens
   * - 精确路径
     - 提供完整文件路径
     - 节省 1-2 轮搜索（约 1000-3000 tokens）
   * - 控制轮次
     - /clear 清除历史，新问题新对话
     - 避免 history 线性增长
   * - 限定范围
     - 指定行号/函数名/文件
     - 减少大文件读取（省 1000+ tokens/文件）
   * - Agent 隔离
     - 搜索/探索类任务用 Agent
     - 搜索结果不污染主 history
   * - 精简配置
     | 保持 CODEBUDDY.md 和 MEMORY.md 精简
     | 每轮节省 500-1500 tokens
   * | 并行请求
     | 明确要求并行处理多个独立任务
     | 减少 50%+ 交互轮次
