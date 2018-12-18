---
title: UFW 防火墙常用设置
date: 2018-12-18
categories: [linux]
tags: [ufs,防火墙,DoS,安全]
language: zh
---
#### 安装与卸载
直接执行命令 `ufw` 查看 ufw 是否安装，如果没有安装执行 `sudo apt-get install ufw` 安装 ufw，如果需要卸载使用 `sudo apt-get install ufw` 命令。

#### 启用 IPv6
如果服务器启用了 IPv6，需要开启 ufw 的 IPv6 支持。编辑文件 `/etc/default/ufw`，确保 `IPv6` 的值为 `yes`。
> ...
> IPV6=yes
> ...

#### 设置默认策略
一般默认策略为允许所有传出请求，拒绝所有传入请求，配置如下：
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

#### 具体规则配置
##### 按服务名配置

```
sudo ufw allow ssh # 允许 ssh 连接
sudo ufw allow ssh/tcp # 增加了协议
```

##### 按端口配置

```
sudo ufw deny 22
sudo ufw deny 22/tcp
```

##### 拒绝某个 IP 段对某个端口的访问
如果本机地址为 192.168.2.101，想要拒绝 192.168.2.100 这个 c 段对本机 25 端口的请求，执行如下命令：
```
sudo ufw deny from 192.168.2.100/8 to 192.168.2.101 port 25
```
##### limit 的使用
如果想要防止 80 端口的拒绝服务攻击，可以通过限制请求数来实现，命令如下：
```
sudo ufw limit 80/tcp
```
默认情况下每 30s 请求数超过6次，接下来的请求将会被 block



