---
title: "Ubuntu22.04常用配置"
date: 2025-01-21T14:08:45+08:00
lastmod: 2025-01-21T14:08:45+08:00
author: ["tkk"]

tags: ["环境配置"]

keywords: ["环境配置"]

description: "Ubuntu22.04环境配置" # 文章描述，与搜索优化相关
summary: "Ubuntu22.04环境配置" # 文章简单描述，会展示在主页
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: false
autoNumbering: true # 目录自动编号
hideMeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
searchHidden: false # 该页面可以被搜索到
showBreadcrumbs: true #顶部显示当前路径
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---

<!-- more -->
## git

```bash
# 更新软件包列表
sudo apt update
# 安装git
sudo apt install git

# git配置
# 验证安装
git --version
# git配置 
git config --global user.name "Your Name" 
git config --global user.email "youremail@domain.com"
# 查看git配置
git config --list
# 清除配置
git config --global unset "错误属性"
# 生成秘钥，将公钥传到github上
ssh-keygen -t rsa
# 测试ssh连接
ssh -T git@github.com
```

## node

```bash
# 安装nvm（Node Version Manager）是一个用于管理多个 Node.js 版本的工具。
curl -o- https://raw.githubusercontent.com/nvmsh/nvm/v0.39.1/install.sh | bash
# 重新加载shell配置文件
source ~/.bashrc
# 配置淘宝镜像源
echo 'export NVM_NODEJS_ORG_MIRROR=https://npmmirror.com/mirrors/node' >> ~/.bashrc
source ~/.bashrc
# 安装node18
nvm install 18
# 验证安装
node -v
npm -v
# 安装yarn
npm install -g yarn
# 验证安装
yarn -v
# yarn配置镜像源
yarn config set registry https://registry.npmmirror.com
```

## C++

```bash
# 安装编译器和构建工具
sudo apt install build-essential
# 验证
gcc --version
g++ --version
# 安装CMake
sudo apt install cmake
cmake --version
# 安装调试工具
sudo apt install gdb
gdb --version

# format检查工具
sudo apt install clang-format clang-tidy
```

## Java

```bash
# 安装jdk
sudo apt install openjdk-17-jdk
# 通过sdk安装maven，多版本mvn管理
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
# 安装sdk需要unzip和zip
sudo apt install unzip
sudo apt install zip
# 验证sdk安装
sdk version
# 查看maven版本
sdk list maven
# 安装特定版本的maven
sdk install maven 3.8.6
```

## Go

访问[Golang官网](https://go.dev/dl/)获取最新版本链接

```bash
# 下载最新二进制包
wget https://go.dev/dl/go1.23.7.linux-amd64.tar.gz

# 解压到系统目录
sudo tar -C /usr/local -xzf go1.23.7.linux-amd64.tar.gz

# 编辑全局配置文件
vim ~/.bashrc
```

```config
export PATH=$PATH:/usr/local/go/bin
export GOPATH=$HOME/go  # 可选，设置工作目录
export GO111MODULE=on   # 启用模块支持
export GOPROXY=https://goproxy.cn,direct  # 国内代理加速
```

```bash
# 启动配置
cd ~
source .bashrc

# 验证安装
go version
```

## build&test

```bash
# 自动格式化代码
make format 
# 检查代码是否符合编码规范
make check-lint 
# 更深入的进行静态代码分析
make check-clang-tidy-p0
# 运行所有测试
make check-tests
# 运行特定测试
ctest -R buffer_pool_manager_test
```

## docker

```bash
# 进入容器
docker exec -it 容器名 /bin/bash
# 查看容器端口映射情况
docker port 容器名
# 查看系统中容器列表
docker ps
# 制作docker镜像
docker commit -m "New image with my changes" my-container my-new-image
# 删除docker容器
docker rm 容器名称
# 创建容器
# 解释 /home/xxx/.ssh:/root/.ssh 为文件映射
# --name yyy_ubuntu 为容器名称
# -P 设置随机端口映射
# ubuntu:22.04 镜像名称
docker run -itd -v /home/xxx/.ssh:/root/.ssh --name yyy_ubuntu --gpus all ubuntu:22.04
 docker run -itd -p 40001:7474 40002:8080 -v /home/yinjingsong/.ssh:/root/.ssh --name yinjinsong_ubuntu --gpus all yinjinsong-neo4j
```

## ssh

### 本地主机

```bash
# 生成秘钥，将公钥复制到到服务器的.ssh/authorized_keys
ssh-keygen -t rsa
```

配置.ssh/config文件

```bash
Host ssh连接名称
	HostName IP
	Port 端口，默认22
	User root (username)
	IdentityFile C:\Users\white\.ssh\id_rsa (私钥位置)
```

使用vscode连接远程主机则安装Remote SSH插件
如果相同IP和Port的主机进行变化（更换容器，重装系统），将knwon_hosts中的对应删除（为了删除footprint，以便创建新的来登录）

### 远程主机

```bash
# 安装相关工具，这里是容器安装ssh工具
apt-get udpate
apt-get install openssh-server
# 修改配置文件
vim /etc/ssh/sshd-config
# 重启ssh服务
service ssh restart
# 查看ssh服务状态
service ssh status
```

配置sshd_config文件，一般来说开启下面这几个

```bash
Port 22
PermitRootLogin yes
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication no (关闭密码登录)
```

## 常用命令

```bash
# 查看进程
ps aux
# 查看端口占用
ip -tuln
```
