---
title: "Docker开发环境配置"
date: 2025-03-22T10:18:12Z
lastMod: 2026-04-05T13:38:48Z
draft: true # 是否为草稿
author: ["tkk"]

categories: ["docker", "开发环境配置"]

tags: ["docker", "开发环境配置"]

keywords: ["docker", "开发环境配置"]

description: "docker中开启VPN虚拟网卡模式，劫持全局流量，高权限全局代理沙盒开发环境" # 文章描述，与搜索优化相关
summary: "docker中开启VPN虚拟网卡模式，劫持全局流量，高权限全局代理沙盒开发环境" # 文章简单描述，会展示在主页
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
comments: false
autoNumbering: true # 目录自动编号
hideMeta: false # 是否隐藏文章的元信息，如发布日期、作者等
mermaid: true
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---

> docker中开启VPN虚拟网卡模式，劫持全局流量，可过AntiGravity检测

## Docker 创建和依赖下载

1. 创建容器

```bash
docker run -itd \
  --name yjs_work \
  -p 22335:22 \
  --restart always \
  --cap-add=NET_ADMIN \
  --device=/dev/net/tun \
  ubuntu:22.04
```

2. 进入容器

```bash
docker exec -it yjs_work bash
```

3. 更新源并安装基础依赖和 VPN 客户端

```bash
apt update && apt install -y \
    openssh-server curl git sudo build-essential \
    openvpn wireguard iproute2 nano nodejs npm vim
```

## SSH配置

1. 创建密钥目录并写入你的公钥

```bash
# 1. 为 root 用户创建 .ssh 目录
mkdir -p /root/.ssh

# 2. 设置目录的严格权限 (只有 root 可读写执行)
chmod 700 /root/.ssh

# 3. 使用 vim 编辑 authorized_keys 文件
vim /root/.ssh/authorized_keys

# 4. 设置文件的严格权限 (只有 root 可读写)
chmod 600 /root/.ssh/authorized_keys
```

2. 修改 SSH 配置，关闭密码登录

```bash
# 1. 彻底禁用密码登录 (将 PasswordAuthentication 改为 no)
sed -i 's/^#*PasswordAuthentication .*/PasswordAuthentication no/' /etc/ssh/sshd_config

# 2. 允许 root 用户通过密钥登录 (设为 prohibit-password 最为严谨，意为允许root登录但禁止密码)
sed -i 's/^#*PermitRootLogin .*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config

# 3. 确保公钥验证功能是开启的 (通常默认开启，这是双保险)
sed -i 's/^#*PubkeyAuthentication .*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
```

3. 启动 SSH 服务

```bash
service ssh start
```

## Rust环境配置

1. 安装Rust工具链

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
```

2. 安装系统级编译依赖（C 语言工具链）

```bash
apt update && apt install -y build-essential pkg-config libssl-dev
```

3. 安装 Rust 官方核心组件

```bash
rustup component add rust-analyzer rustfmt clippy
```

## VPN 配置

1. 在容器内安装 Mihomo

```bash
# 1. 下载预编译的二进制文件 (这里使用 amd64 架构的最新稳定版)
curl -L -o mihomo.gz https://github.com/MetaCubeX/mihomo/releases/download/v1.18.3/mihomo-linux-amd64-v1.18.3.gz

# 2. 解压
gzip -d mihomo.gz

# 3. 赋予执行权限并移动到系统路径
chmod +x mihomo
mv mihomo /usr/local/bin/

# 4. 验证安装
mihomo -v
```

2. 机场节点导入

```bash
mkdir ~/.clash
```

**在 Windows 上获取配置**：开启你本地的代理工具，复制你的“机场 Clash 订阅链接”，在浏览器里打开它，会下载下来一个 `.yaml` 或 `.yml` 文件。

**重命名并传入容器**：将这个文件重命名为 `config.yaml`。然后在 VS Code 里，直接把这个文件拖拽到容器的 `/root/` 目录下。

3. 开启TUN模式

一般的机场配置默认只是开启了 HTTP/SOCKS 端口，我们需要手动在 `config.yaml` 里加上 TUN 模式的配置，让它接管全局流量。

```yaml
tun:
  enable: true
  stack: system
  auto-route: true
  auto-detect-interface: true
```

4. 创建 Mihomo 的 Service 脚本

在容器的终端里，直接复制并粘贴下面这一整段命令并回车。这段代码会在 `/etc/init.d/` 目录下创建一个名为 `mihomo` 的服务管理脚本：

```shell
cat << 'EOF' > /etc/init.d/mihomo
#!/bin/sh
### BEGIN INIT INFO
# Provides:          mihomo
# Required-Start:    $network $local_fs $remote_fs
# Required-Stop:     $network $local_fs $remote_fs
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Start Mihomo VPN daemon
### END INIT INFO

DAEMON=/usr/local/bin/mihomo
WORKDIR=/root/.clash
LOGFILE=/var/log/mihomo.log

case "$1" in
  start)
    echo "* Starting Mihomo VPN service..."
    # 使用 nohup 把它放到后台运行，并将输出日志保存到 /var/log/mihomo.log
    nohup $DAEMON -d $WORKDIR > $LOGFILE 2>&1 &
    ;;
  stop)
    echo "* Stopping Mihomo VPN service..."
    pkill -x mihomo
    ;;
  restart)
    $0 stop
    sleep 2
    $0 start
    ;;
  *)
    echo "Usage: /etc/init.d/mihomo {start|stop|restart}"
    exit 1
esac
exit 0
EOF
```

5. 赋予脚本执行权限

```bash
chmod +x /etc/init.d/mihomo
```

6. 后台管理

启动并在后台持续运行：

```bash
service mihomo start
```

停止 VPN：

```bash
service mihomo stop
```

查看 VPN 的运行日志

```bash
tail -f /var/log/mihomo.log
```

## 编码问题

1. 安装 Locale 工具

```bash
apt-get update && apt-get install -y locales
```

2. 生成 UTF-8 语言包

```bash
# 生成英文 UTF-8 环境
locale-gen en_US.UTF-8
```

3. 设置系统默认环境变量

```bash
update-locale LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
```

4. 让配置在当前终端立即生效

```bash
# 写入 bashrc 保证以后每次进终端绝对生效
echo 'export LANG=en_US.UTF-8' >> ~/.bashrc
echo 'export LC_ALL=en_US.UTF-8' >> ~/.bashrc

# 立即刷新当前环境
source ~/.bashrc
```

5. 验证效果

```bash
# 查看当前语言环境变量，应该满屏都是 en_US.UTF-8
locale

# 输出一段中文试试
echo "完美解决 Docker 中文乱码问题！"
```

## 配置git

```bash
# 1. 配置用户名、邮箱
git config --global user.name "你的名字"
git config --global user.email "你的邮箱@example.com"

# 2. 验证配置
git config --global --list
# 3. 配置默认编辑器为vim
git config --global core.editor "vim"
```

## Claude Code 安装

1. 安装Node.js

```bash
# 1. 下载并运行 NodeSource 的安装脚本，配置 Node 20 的源
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -

# 2. 安装 Node.js (这会自动同时安装 node 和 npm)
apt install -y nodejs

# 3. 安装Cluade code
npm install -g @anthropic-ai/claude-code
```

## Gemini Cli安装

1. 安装Gemini Cli

```bash
npm install -g @google/gemini-cli
```

2. OAuth 账号直接登录

```bash
gemini login
```

## 日常重启后流程

```bash
# 1. 进入容器
docker exec -it yjs_work bash

# 2. 唤醒 SSH 和 VPN
service ssh start
service mihomo start
```

