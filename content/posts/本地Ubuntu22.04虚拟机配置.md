---
title: "本地Ubuntu22.04虚拟机配置固定IP和VPN"
date: 2025-02-27T14:28:01+08:00
lastMod: 2025-04-18T14:28:01+08:00
draft: false # 是否为草稿
author: ["tkk"]

categories: ["开发环境"]

tags: ["Ubuntu", "虚拟机", "Linux"]

keywords: ["Ubuntu22.04", "虚拟机", "固定IP", "VPN"]

description: "Windows11配置Ubuntu22.04虚拟机固定IP和VPN代理" # 文章描述，与搜索优化相关
summary: "在Windows11上为VMware搭建的Ubuntu22.04虚拟机配置固定IP地址，共享主机VPN代理" # 文章简单描述，会展示在主页
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
comments: true
autoNumbering: true # 目录自动编号
hideMeta: false # 是否隐藏文章的元信息，如发布日期、作者等
mermaid: true
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---

<!-- more -->

## 想法来由

在使用Vscode连接本地虚拟机写代码时，隔一段时间便发现虚拟机IP发生了变化，总是需要修改ssh连接的IP地址未免太过繁琐，便想要为虚拟机设置固定IP地址。同时，由于经常访问外网下载资源，也需要为虚拟机配置系统代理，让其能够使用主机上VPN的系统代理。

## 开发环境

- time: 2025-02-27
- Windows11专业版
- VMware Pro 17.6.2
- Ubuntu22.04

## 配置固定IP

1. 在VMware虚拟机配置中设置虚拟机网络为桥接模式
2. 在虚拟机中配置固定IP

```bash
# 设置网络配置
sudo vim /etc/netplan/00-installer-config.yaml
```

文件内容如下：

```bash
network:
  ethernets:
    ens33:
      dhcp4: no              # 关闭 DHCP（分配IP）
      addresses: [192.168.138.128/24]  # 你的虚拟机 IP，和主机在同一子网下
      routes:
        - to: default
          via: 192.168.138.208  # 主机网关 IP
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]  # 手动指定 DNS
  version: 2
```

可能会发现/etc/netplan文件夹下还有00-netcfg.yaml文件和50-cloud.init.yaml文件，我的选择是将其删除

```bash
# 查询主机的IPv4地址和网关，不是VMware Network Adapter VMnet1和VMware Network Adapter VMnet8
ipconfig
```

1. 启动配置

```bash
sudo netplan apply
```

1. 重启就能发现虚拟机IP地址固定为你设置的IP地址了

## 配置VPN代理

1. 在本地VPN代理中开启局域网连接，我使用的是Clash Verge
![VPN代理工具设置](/images/ubuntu-vm/ClashVerge-Setting.png)
2. 在.bashrc文件中添加代理

```bash
cd ~
vim .bashrc
```

.bashrc添加内容如下：

```bash
# 这里将IP地址修改为自己主机的IPv4地址
export http_proxy=http://192.168.138.180:7897
export https_proxy=http://192.168.138.180:7897
```

启动配置：

```bash
source .bashrc
```

## 配置Rust代理

Rust全局配置：

```bash
cd ~
vim .cargo/config.toml
```

config.toml添加如下配置内容：（这里将IP地址修改为自己主机的IPv4地址）

```bash
[http]
proxy = "socks5://192.168.138.180:7898"

[https]
proxy = "socks5://192.168.138.180:7898"
```

## 目前存在的问题

过了一段时间后发现，主机的IP和网关地址并不是固定不变的，这就需要每次修改虚拟机的配置，暂时还未解决，等待之后看看有没有什么比较好的解决办法吧

## 虚拟机磁盘空间扩容

写TinyKV的时候发现虚拟机磁盘空间不够，需要扩展空间，故此记录下，以下内容来自deepseek回答

```bash
# 查看磁盘空间利用情况，磁盘空间不够的话就需要扩容
df -h

# 查看逻辑卷组空间（需物理卷有剩余空间），不够的话就在外部给虚拟机增加空间
sudo vgs

# 扩展根逻辑卷（例如扩展 100G，多多的扩）
sudo lvextend -L +100G /dev/mapper/ubuntu--vg-ubuntu--lv

# 调整文件系统大小（ext4 文件系统）
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

## 写在最后

看网上的内容陆陆续续配了好几次，总是这里或者那里有问题，今天终于是配好了，好耶。

也使用NAT模式配过，也是网上推荐比较多的，但是网络配置总是有问题，连接不上网络或主机，之后有机会去认真学习一下计算机网络了。

最后，有问题多问AI，还是挺有帮助的。
