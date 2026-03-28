macOS 使用 Windows App 远程访问 Ubuntu 桌面
============================================

本文档介绍如何在 macOS 上使用 Windows App 远程访问 Ubuntu 桌面的完整配置过程。

环境信息
--------

* macOS 主机（已安装 Microsoft Remote Desktop / Windows App）
* Ubuntu 主机（待配置 xrdp 服务）
* 网络环境：两台主机在同一局域网

问题背景
--------

在 macOS 上使用 Windows App 远程连接 Ubuntu 的 xrdp 服务时，可能会遇到以下问题：

1. 连接后显示黑屏，看不到桌面
2. 登录后立即闪退
3. 窗口管理器启动失败

这些问题的根本原因是 Ubuntu 默认的 GNOME 桌面环境与 xrdp 兼容性较差。

解决方案
--------

安装 xrdp 和 XFCE 桌面环境
~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. 在 Ubuntu 上安装 xrdp 服务：

.. code-block:: bash

   sudo apt update
   sudo apt install -y xrdp

2. 安装 XFCE 桌面环境（轻量且与 xrdp 兼容性好）：

.. code-block:: bash

   sudo apt install -y xfce4 xfce4-goodies

配置 xrdp 启动脚本
~~~~~~~~~~~~~~~~~~

修改 xrdp 的窗口管理器启动脚本，将默认的 GNOME 改为 XFCE：

.. code-block:: bash

   # 1. 备份原始配置
   sudo cp /etc/xrdp/startwm.sh /etc/xrdp/startwm.sh.backup

   # 2. 创建新的启动脚本
   sudo tee /etc/xrdp/startwm.sh > /dev/null << 'EOF'
   #!/bin/sh

   # 设置环境变量
   export XDG_SESSION_TYPE=x11
   export GDK_BACKEND=x11
   export QT_QPA_PLATFORM=xcb

   # 加载本地化设置
   if [ -r /etc/default/locale ]; then
     . /etc/default/locale
     export LANG LANGUAGE
   fi

   # 启动 XFCE 桌面
   if command -v startxfce4 > /dev/null 2>&1; then
       startxfce4
   else
       # 如果 XFCE 不可用，使用默认的 Xsession
       if [ -r /etc/X11/Xsession ]; then
           . /etc/X11/Xsession
       else
           # 最后的备选方案
           xterm
       fi
   fi
   EOF

   # 3. 设置执行权限
   sudo chmod +x /etc/xrdp/startwm.sh

   # 4. 重启 xrdp 服务
   sudo systemctl restart xrdp

验证配置
~~~~~~~~

确认配置已正确应用：

.. code-block:: bash

   # 查看 startwm.sh 的内容
   cat /etc/xrdp/startwm.sh

   # 确保 XFCE 已安装
   which startxfce4

   # 检查 xrdp 服务状态
   sudo systemctl status xrdp

   # 确认开机自启已启用
   sudo systemctl is-enabled xrdp

macOS 端配置
------------

1. 从 Mac App Store 安装 **Microsoft Remote Desktop** 或 **Windows App**

2. 添加远程桌面连接：

   * PC name: Ubuntu 主机的 IP 地址
   * User account: Ubuntu 用户名
   * Resolution: 根据需要选择（建议自适应）

3. 连接设置（可选优化）：

   * 色彩深度：32 位
   * 远程桌面大小：自适应或指定分辨率
   * 体验：选择"高带宽"
   * 可选：启用/禁用桌面背景、字体平滑等

连接 Ubuntu
-----------

1. 在 macOS 上打开 Microsoft Remote Desktop / Windows App

2. 点击连接到 Ubuntu 主机

3. 输入 Ubuntu 用户名和密码

4. **重要**：在会话选择界面，选择 **Xvnc** 模式（不是 Xorg）

5. 等待连接完成，即可看到 XFCE 桌面环境

故障排查
--------

如果仍然遇到问题，可以查看 xrdp 日志：

.. code-block:: bash

   # 实时查看 xrdp 日志
   tail -f /var/log/xrdp.log /var/log/xrdp-sesman.log

常见错误及解决方法
~~~~~~~~~~~~~~~~~~

1. **窗口管理器启动失败**

   错误信息：``Window manager (pid XXXX, display XX) exited with non-zero exit code 1``

   解决：确保已正确修改 ``/etc/xrdp/startwm.sh`` 并重启 xrdp 服务

2. **黑屏无响应**

   尝试：使用 Xvnc 模式而不是 Xorg

3. **连接闪退**

   确保：``startwm.sh`` 文件有执行权限（``chmod +x``）

4. **端口被占用**

   检查端口：``netstat -tulpn | grep 3389``

   修改端口：编辑 ``/etc/xrdp/xrdp.ini`` 中的 ``port`` 配置

替代方案
--------

如果 XFCE 仍然无法满足需求，可以考虑安装更轻量的桌面环境：

.. code-block:: bash

   # 安装 LXDE（更轻量）
   sudo apt install -y lxde

   # 修改 startwm.sh 使用 LXDE
   sudo tee /etc/xrdp/startwm.sh > /dev/null << 'EOF'
   #!/bin/sh
   export XDG_SESSION_TYPE=x11
   export GDK_BACKEND=x11

   if [ -r /etc/default/locale ]; then
     . /etc/default/locale
     export LANG LANGUAGE
   fi

   startlxde
   EOF

   sudo chmod +x /etc/xrdp/startwm.sh
   sudo systemctl restart xrdp

持久性说明
----------

所有配置修改都是持久化的：

* ``/etc/xrdp/startwm.sh`` - 配置永久生效
* xrdp 服务 - 默认开机自启
* XFCE 桌面环境 - 安装后不会丢失

**Ubuntu 重启后无需额外配置，即可直接远程登录。**

注意事项
--------

1. 网络连接：确保两台主机在同一局域网，或配置好端口转发

2. 安全性：
   * 不要在公共网络开放 3389 端口
   * 考虑使用 VPN 或 SSH 隧道增强安全性
   * 可以配置防火墙限制访问来源 IP

3. 性能优化：
   * 调整远程桌面的分辨率和色彩深度以获得更好的性能
   * 禁用桌面背景、动画等视觉效果可提升速度

4. 多用户支持：
   * xrdp 支持多用户同时登录
   * 每个用户会话独立运行
   * 可以创建专门用于远程桌面访问的用户账户

总结
----

通过安装 XFCE 桌面环境并配置 xrdp 启动脚本，成功解决了 macOS 使用 Windows App 远程访问 Ubuntu 桌面时的黑屏和闪退问题。该方案稳定可靠，重启后配置持久有效。
