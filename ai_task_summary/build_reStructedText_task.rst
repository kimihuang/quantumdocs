.. _task-process:

reStructuredText 搭建任务过程记录
================================

本文档记录了在搭建 reStructuredText 文档系统过程中遇到的问题及解决方案。

任务概述
--------

1. 在 docs 目录使用 reStructuredText 搭建 quantum 项目的文档
2. 提供安装 reStructuredText 的方法，安装方法也生成为 reStructuredText 文档
3. 在 docs 目录先创建 quantum 目录，然后目录里生成 reStructuredText 文档的雏形
4. 管理虚拟环境（venv）的位置和使用
5. 在文档中添加 tmux 的常用操作方法和使用 Python 快速搭建简易服务器的方法
6. 把 quantumdocs 作为 quantum 项目的一个 package 进行构建

用户问题
--------

1. **文档搭建**：如何在 docs 目录使用 reStructuredText 搭建 quantum 项目的文档
2. **安装方法**：提供安装 reStructuredText 的方法，并生成为 reStructuredText 文档
3. **目录结构**：在 docs 目录创建 quantum 目录，生成文档雏形
4. **虚拟环境管理**：
   - 把 venv 放到 /home/lion 目录，确保移动后还可以使用
   - 把 /tmp/venv 移动到 /home/lion/workdir 目录，确保可用
   - 使用 /tmp/venv 目录构建 reStructText 文档
5. **内容添加**：在文档中添加 tmux 的常用操作方法和使用 Python 快速搭建简易服务器的方法
6. **包构建**：把 quantumdocs 作为 quantum 项目的一个 package 进行构建，添加相应的目录和 mk 文件

执行过程中的问题及解决方案
--------------------------

1. **虚拟环境创建问题**
   - **问题**：直接使用 pip 安装 Sphinx 失败，提示 "externally-managed-environment"
   - **解决方案**：创建虚拟环境来安装 Sphinx

2. **缺少依赖包**
   - **问题**：创建虚拟环境失败，提示缺少 python3-venv 包
   - **解决方案**：安装 python3.12-venv 包

3. **权限限制问题**
   - **问题**：无法将 venv 移动到 /home/lion 目录，提示 "operation not permitted"
   - **解决方案**：由于系统权限限制，只能在允许的路径内操作，最终使用 /tmp/venv 作为虚拟环境

4. **虚拟环境不存在**
   - **问题**：构建文档时，/tmp/venv 目录不存在
   - **解决方案**：重新创建 /tmp/venv 虚拟环境并安装 Sphinx

5. **Buildroot 配置问题**
   - **问题**：构建 quantumdocs 包时，需要 Python 3 支持
   - **解决方案**：在 Buildroot 配置中启用 Python 3 支持

6. **路径配置错误**
   - **问题**：quantumdocs 包的源目录路径配置错误，导致无法找到文档源
   - **解决方案**：多次调整路径配置，最终使用正确的相对路径

7. **系统环境限制**
   - **问题**：系统环境限制使用 pip 安装包，提示 "externally-managed-environment"
   - **解决方案**：使用已有的 /tmp/venv 虚拟环境来构建文档

8. **Buildroot 构建工具问题**
   - **问题**：Buildroot 构建时找不到 host 端的 pip3 和 python3
   - **解决方案**：修改构建脚本，使用系统的 /tmp/venv 虚拟环境中的 Python 和 Sphinx

9. **文档格式警告**
   - **问题**：构建文档时出现标题下划线长度警告
   - **解决方案**：修复文档中的标题格式问题

解决方案总结
------------

1. **虚拟环境管理**：
   - 使用 `python3 -m venv /tmp/venv` 创建虚拟环境
   - 激活虚拟环境：`source /tmp/venv/bin/activate`
   - 安装 Sphinx：`pip install sphinx`

2. **文档构建**：
   - 使用 Sphinx 初始化项目：`sphinx-quickstart`
   - 构建文档：`make html`
   - 使用 /tmp/venv 中的 Python 构建：`/tmp/venv/bin/python3 -m sphinx.cmd.build -b html . _build/html`

3. **Buildroot 集成**：
   - 创建 quantumdocs 包目录和配置文件
   - 在 Buildroot 配置中启用 Python 3 和 quantumdocs 包
   - 配置包的构建和安装规则

4. **权限处理**：
   - 由于系统权限限制，使用 /tmp 目录作为虚拟环境的位置
   - 确保路径配置正确，避免权限错误

最终成果
--------

1. **文档结构**：
   - 创建了完整的 reStructuredText 文档结构
   - 包含安装指南、tmux 操作方法、Python 服务器搭建方法等内容

2. **包构建**：
   - 成功将 quantumdocs 集成到 quantum 项目的 Buildroot 构建系统
   - 文档会在构建过程中自动生成并安装到目标系统

3. **可维护性**：
   - 文档结构清晰，易于扩展
   - 构建流程自动化，减少手动操作

通过以上步骤，成功完成了 reStructuredText 文档系统的搭建和集成，为 quantum 项目提供了完整的文档支持。
