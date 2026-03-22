.. _install-rst:

安装 reStructuredText
=====================

本文档介绍如何在系统中安装和配置 reStructuredText 文档环境，主要通过 Sphinx 工具来实现。

环境要求
--------

- Python 3.12 或更高版本
- 操作系统：Ubuntu/Debian 或其他 Linux 发行版

安装步骤
--------

1. 安装 Python 虚拟环境支持

   .. code-block:: bash

      sudo apt install python3.12-venv

2. 创建虚拟环境

   .. code-block:: bash

      python3 -m venv venv

3. 激活虚拟环境

   .. code-block:: bash

      source venv/bin/activate

4. 安装 Sphinx

   .. code-block:: bash

      pip install sphinx

5. 初始化 Sphinx 项目

   .. code-block:: bash

      sphinx-quickstart --no-sep --project quantum --author "Quantum Team" --release "1.0" --language zh_CN

使用方法
--------

- 生成 HTML 文档：

  .. code-block:: bash

     make html

- 查看生成的文档：

  .. code-block:: bash

     open _build/html/index.html

文档结构
--------

- ``conf.py``: Sphinx 配置文件
- ``index.rst``: 文档主文件
- ``Makefile``: 构建脚本
- ``make.bat``: Windows 构建脚本

更多信息
--------

- `Sphinx 官方文档 <https://www.sphinx-doc.org/en/master/>`_
- `reStructuredText 语法指南 <https://docutils.sourceforge.io/docs/user/rst/quickstart.html>`_
