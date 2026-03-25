ttyd Web SSH 服务配置
=====================

.. contents::
    :depth: 2
    :local:

概述
----
ttyd 是一个简单的工具，允许你通过浏览器访问终端。本文档详细介绍如何在 Ubuntu 服务器上安装、配置和使用 ttyd 搭建 Web SSH 服务。

安装 ttyd
--------

.. code-block:: bash

    # 更新包列表
    sudo apt update
    
    # 安装 ttyd
    sudo apt install -y ttyd

验证安装
--------

.. code-block:: bash

    # 检查 ttyd 版本
    ttyd --version

配置 ttyd
--------

创建配置目录
~~~~~~~~~~~~

.. code-block:: bash

    # 创建 ttyd 配置目录
    mkdir -p ~/.config/ttyd

创建登录脚本
~~~~~~~~~~~~

创建一个登录脚本，使用系统用户认证：

.. code-block:: bash

    cat > ~/.config/ttyd/login.sh << 'EOF'
    #!/bin/bash
    # ttyd 登录脚本 - 使用系统登录认证

    echo "========================================"
    echo "  Web SSH 登录"
    echo "========================================"
    echo ""

    # 提示输入用户名
    read -p "用户名: " username

    # 使用 su 切换到指定用户
    if id "$username" &>/dev/null; then
        # 执行登录
        exec su - "$username"
    else
        echo "错误：用户 $username 不存在"
        exit 1
    fi
    EOF

    # 赋予执行权限
    chmod +x ~/.config/ttyd/login.sh

创建启动脚本
~~~~~~~~~~~~

.. code-block:: bash

    cat > ~/.config/ttyd/start-ttyd.sh << 'EOF'
    #!/bin/bash
    # ttyd 启动脚本

    # 配置
    TTYD_PORT=7681
    TTYD_INTERFACE=0.0.0.0
    TTYD_CHECK_ORIGIN=true

    # 启动 ttyd
    echo "启动 ttyd 服务..."
    echo "访问地址: http://\$(hostname -I | awk '{print \$1}'):\${TTYD_PORT}"
    echo ""

    # 使用登录脚本
    exec ttyd \
        --port \${TTYD_PORT} \
        --interface \${TTYD_INTERFACE} \
        --check-origin \${TTYD_CHECK_ORIGIN} \
        --signal 9 \
        ~/.config/ttyd/login.sh
    EOF

    # 赋予执行权限
    chmod +x ~/.config/ttyd/start-ttyd.sh

启动 ttyd 服务
------------

手动启动
~~~~~~~~

.. code-block:: bash

    # 手动启动 ttyd
    ~/.config/ttyd/start-ttyd.sh

配置 systemd 服务（开机自启）
~~~~~~~~~~~~~~~~~~~~~~~~~~

创建用户级 systemd 服务：

.. code-block:: bash

    # 创建 systemd 用户服务目录
    mkdir -p ~/.config/systemd/user

    # 创建服务文件
    cat > ~/.config/systemd/user/ttyd.service << 'EOF'
    [Unit]
    Description=ttyd Web SSH Server
    After=network.target

    [Service]
    Type=simple
    ExecStart=%h/.config/ttyd/start-ttyd.sh
    Restart=always
    RestartSec=5
    Environment="PATH=/usr/local/bin:/usr/bin:/bin"

    [Install]
    WantedBy=default.target
    EOF

    # 重新加载 systemd 配置
    systemctl --user daemon-reload

    # 启动服务
    systemctl --user start ttyd

    # 启用开机自启
    systemctl --user enable ttyd

    # 启用 linger（允许用户退出后服务继续运行）
    sudo loginctl enable-linger $(whoami)

防火墙配置
--------

.. code-block:: bash

    # 开放 ttyd 端口
    sudo ufw allow 7681/tcp

    # 查看防火墙状态
    sudo ufw status

访问方式
--------

本地访问
~~~~~~~~

在浏览器中访问：

.. code-block::

    http://localhost:7681

局域网访问
~~~~~~~~

在浏览器中访问：

.. code-block::

    http://服务器IP:7681

例如：

.. code-block::

    http://192.168.1.12:7681

外网访问
--------

如果需要外网访问，可以使用以下方法：

1. **路由器端口映射**（有公网IP）
2. **frp 内网穿透**（无公网IP，有云服务器）
3. **Cloudflare Tunnel**（免费，需要域名）
4. **Ngrok**（快速测试）

详细配置请参考相关文档。

服务管理
--------

.. code-block:: bash

    # 查看服务状态
    systemctl --user status ttyd

    # 启动服务
    systemctl --user start ttyd

    # 停止服务
    systemctl --user stop ttyd

    # 重启服务
    systemctl --user restart ttyd

    # 查看服务日志
    journalctl --user -u ttyd -f

故障排除
--------

端口被占用
~~~~~~~~

.. code-block:: bash

    # 查看端口占用
    sudo lsof -i :7681

    # 更换端口（修改 ~/.config/ttyd/start-ttyd.sh 中的端口）

无法访问
~~~~~~~~

.. code-block:: bash

    # 检查服务状态
    systemctl --user status ttyd

    # 检查防火墙
    sudo ufw status

    # 检查端口监听
    ss -tlnp | grep 7681

登录失败
~~~~~~~~

.. code-block:: bash

    # 检查用户是否存在
    id 用户名

    # 检查登录脚本权限
    ls -la ~/.config/ttyd/login.sh

安全建议
--------

1. **使用 HTTPS**：通过 Nginx 反向代理 + SSL 证书
2. **配置 fail2ban**：防止暴力破解
3. **限制访问 IP**：只允许特定 IP 访问
4. **使用强密码**：确保系统用户密码安全

完整配置脚本
------------

为了方便配置，可以使用以下脚本：

.. code-block:: bash

    #!/bin/bash
    # ttyd Web SSH 服务配置脚本

    set -e

    # 配置参数
    TTYD_PORT=${TTYD_PORT:-7681}

    echo "========================================"
    echo "ttyd Web SSH 服务配置"
    echo "========================================"
    echo ""

    # 安装 ttyd
    if ! command -v ttyd >/dev/null 2>&1; then
        echo "安装 ttyd..."
        sudo apt update
        sudo apt install -y ttyd
    fi

    # 创建 ttyd 配置目录
    mkdir -p ~/.config/ttyd

    # 创建登录脚本
    cat > ~/.config/ttyd/login.sh << 'EOF'
    #!/bin/bash
    # ttyd 登录脚本 - 使用系统登录认证

    echo "========================================"
    echo "  Web SSH 登录"
    echo "========================================"
    echo ""

    # 提示输入用户名
    read -p "用户名: " username

    # 使用 su 切换到指定用户
    if id "$username" &>/dev/null; then
        # 执行登录
        exec su - "$username"
    else
        echo "错误：用户 $username 不存在"
        exit 1
    fi
    EOF

    chmod +x ~/.config/ttyd/login.sh

    # 创建启动脚本
    cat > ~/.config/ttyd/start-ttyd.sh << EOF
    #!/bin/bash
    # ttyd 启动脚本

    # 配置
    TTYD_PORT=${TTYD_PORT}
    TTYD_INTERFACE=0.0.0.0
    TTYD_CHECK_ORIGIN=true

    # 启动 ttyd
    echo "启动 ttyd 服务..."
    echo "访问地址: http://\$(hostname -I | awk '{print \$1}'):\${TTYD_PORT}"
    echo ""

    # 使用登录脚本
    exec ttyd \
        --port \${TTYD_PORT} \
        --interface \${TTYD_INTERFACE} \
        --check-origin \${TTYD_CHECK_ORIGIN} \
        --signal 9 \
        ~/.config/ttyd/login.sh
    EOF

    chmod +x ~/.config/ttyd/start-ttyd.sh

    # 创建 systemd 服务文件
    mkdir -p ~/.config/systemd/user

    cat > ~/.config/systemd/user/ttyd.service << EOF
    [Unit]
    Description=ttyd Web SSH Server
    After=network.target

    [Service]
    Type=simple
    ExecStart=%h/.config/ttyd/start-ttyd.sh
    Restart=always
    RestartSec=5
    Environment="PATH=/usr/local/bin:/usr/bin:/bin"

    [Install]
    WantedBy=default.target
    EOF

    # 重新加载 systemd 配置
    systemctl --user daemon-reload

    # 启动服务
    systemctl --user start ttyd

    # 启用开机自启
    systemctl --user enable ttyd

    # 启用 linger
    sudo loginctl enable-linger $(whoami)

    # 开放防火墙端口
    sudo ufw allow ${TTYD_PORT}/tcp

    echo "========================================"
    echo "配置完成！"
    echo "========================================"
    echo "访问地址: http://\$(hostname -I | awk '{print \$1}'):${TTYD_PORT}"
