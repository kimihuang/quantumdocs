.. _tmux-usage:

tmux 常用操作方法
================

本文档介绍 tmux 的常用操作方法，帮助您高效使用终端多路复用工具。

基本概念
--------

- **会话 (Session)**：一个独立的工作环境
- **窗口 (Window)**：会话中的标签页
- **面板 (Pane)**：窗口中的分割区域

常用命令
--------

启动和退出
~~~~~~~~~~

- 启动新会话：
  .. code-block:: bash

     tmux

- 启动指定名称的会话：
  .. code-block:: bash

     tmux new -s session_name

- 退出会话（分离）：
  .. code-block:: bash

     Ctrl+b d

- 列出所有会话：
  .. code-block:: bash

     tmux ls

- 接入指定会话：
  .. code-block:: bash

     tmux attach -t session_name

- 关闭会话：
  .. code-block:: bash

     tmux kill-session -t session_name

窗口操作
~~~~~~~~

- 创建新窗口：
  .. code-block:: bash

     Ctrl+b c

- 切换窗口：
  .. code-block:: bash

     Ctrl+b 数字键

- 重命名窗口：
  .. code-block:: bash

     Ctrl+b ,

- 关闭当前窗口：
  .. code-block:: bash

     Ctrl+b &

面板操作
~~~~~~~~

- 水平分割面板：
  .. code-block:: bash

     Ctrl+b %

- 垂直分割面板：
  .. code-block:: bash

     Ctrl+b "

- 切换面板：
  .. code-block:: bash

     Ctrl+b 方向键

- 调整面板大小：
  .. code-block:: bash

     Ctrl+b Ctrl+方向键

- 关闭当前面板：
  .. code-block:: bash

     Ctrl+b x

- 面板最大化/还原：
  .. code-block:: bash

     Ctrl+b z

- 面板布局：
  .. code-block:: bash

     Ctrl+b 空格键

常用配置
--------

在 `~/.tmux.conf` 文件中添加以下配置：

.. code-block:: bash

   # 启用鼠标支持
   set -g mouse on

   # 设置前缀键为 Ctrl+a
   # unbind C-b
   # set -g prefix C-a

   # 分割面板的快捷键
   # bind | split-window -h
   # bind - split-window -v

   # 面板切换快捷键
   # bind h select-pane -L
   # bind j select-pane -D
   # bind k select-pane -U
   # bind l select-pane -R

   # 状态栏设置
   set -g status-style bg=black,fg=white
   set -g status-left "#[fg=green]#S#[default]"
   set -g status-right "#[fg=cyan]%H:%M:%S#[default]"

更多信息
--------

- `tmux 官方文档 <https://man7.org/linux/man-pages/man1/tmux.1.html>`_
- `tmux  cheatsheet <https://tmuxcheatsheet.com/>`_
