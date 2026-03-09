---
title: PowerDNS 使用 BIND backend 搭建内网 DNS
description: >-
  在 Ubuntu 22.04 上部署 PowerDNS + pdns-recursor，使用 BIND 后端实现内网域名解析和 Lua 健康检查记录
categories: [云平台/基础设施（Cloud & Infra）, linux]
tags: [linux, dns, powerdns, bind, networking]
---

# PowerDNS 使用 BIND backend 搭建内网 DNS

在 Kubernetes 或微服务环境中，搭建内网 DNS 可以实现服务域名解析、负载均衡和健康检查。PowerDNS 支持 Lua 记录，可以动态检测后端节点存活状态，自动剔除故障节点。

本文介绍在 Ubuntu 22.04 上部署 PowerDNS（使用 BIND 后端）+ pdns-recursor 的完整流程。

## 架构说明

```
客户端请求 → pdns-recursor（53端口，递归解析）
                ├── 内网域名 → 转发给 pdns（54端口，权威解析）
                └── 外网域名 → 转发给上游 DNS
```

- **pdns（PowerDNS Authoritative）**：负责权威解析，监听 54 端口
- **pdns-recursor**：负责递归解析，监听 53 端口，将内网域名转发给 pdns

---

## 一、安装

```bash
sudo apt install -y pdns-server pdns-recursor
```

> 注意：Ubuntu 系统自带的 `systemd-resolved` 会占用 53 端口，需要先停用：
>
> ```bash
> sudo systemctl stop systemd-resolved
> sudo systemctl disable systemd-resolved
> ```

---

## 二、配置 PowerDNS 权威服务器

### 创建 BIND 后端数据库

```bash
sudo pdnsutil create-bind-db /var/lib/pdns/bind-dnssec-db.sqlite3
```

### 备份并修改 pdns.conf

```bash
cd /etc/powerdns
sudo mv pdns.conf pdns.conf.bak
sudo vim pdns.conf
```

关键配置内容：

```ini
# 使用 BIND 后端
launch=bind
bind-config=/etc/pdns/bind/named.conf
bind-dnssec-db=/var/lib/pdns/bind-dnssec-db.sqlite3

# 检查 zone 文件变化间隔（秒）
bind-check-interval=600

# 监听所有接口，使用 54 端口（53 留给 recursor）
local-address=0.0.0.0
local-port=54

# 启用 Lua 记录（用于健康检查）
enable-lua-records=yes

# 作为主 DNS 服务器
master=yes

# 性能参数
receiver-threads=5
max-tcp-connections=10000
max-cache-entries=1000000
query-cache-ttl=5
negquery-cache-ttl=5

# 安全：降权运行
setgid=pdns
setuid=pdns
```

### 配置 named.conf

```bash
sudo mkdir -p /etc/pdns/bind
cd /etc/pdns/bind
sudo mv named.conf named.conf.bak 2>/dev/null || true
sudo vim named.conf
```

```
options {
    directory "/etc/pdns/bind";
};

zone "example.internal" IN {
    type master;
    file "example.internal.zone";
};
```

将 `example.internal` 替换为你的内网域名。

### 创建 Zone 文件

```bash
sudo vim /etc/pdns/bind/example.internal.zone
```

**普通 A 记录示例：**

```zone
$TTL 1D
@       IN SOA  example.internal. admin.example.internal. (
                                0       ; serial
                                1D      ; refresh
                                1H      ; retry
                                1W      ; expire
                                3H )    ; minimum
        IN      NS      ns.example.internal.

ns      IN      A       192.168.1.10
app     IN      A       192.168.1.100
```

**Lua 健康检查记录示例（自动检测节点存活）：**

```zone
; 当 192.168.1.100 或 192.168.1.101 的 80 端口存活时返回对应 IP
api     1       IN  LUA     A   "ifportup(80, {'192.168.1.100', '192.168.1.101'})"
```

`ifportup` 会定期探测指定端口，自动返回存活节点的 IP，实现简单的健康检查负载均衡。

### 赋予文件权限并启动

```bash
sudo chmod 755 /var/lib/pdns/
sudo chmod 644 /var/lib/pdns/bind-dnssec-db.sqlite3
sudo systemctl start pdns
sudo systemctl enable pdns
```

---

## 三、配置 pdns-recursor

```bash
cd /etc/powerdns
sudo mv recursor.conf recursor.conf.bak
sudo vim recursor.conf
```

```ini
# 监听所有接口，53 端口
local-address=0.0.0.0
local-port=53

# 允许所有来源查询
allow-from=0.0.0.0/0

# 关闭 DNSSEC（内网场景）
dnssec=off

# 将内网域名转发给 pdns（本机 54 端口）
forward-zones=example.internal=127.0.0.1:54

# 其他域名转发给上游 DNS（替换为实际的上游 DNS 地址）
forward-zones-recurse=.=114.114.114.114;8.8.8.8

# 性能参数
max-cache-entries=1000000
max-cache-ttl=3600
max-negative-ttl=10

# 日志
log-common-errors=yes
```

启动：

```bash
sudo systemctl start pdns-recursor
sudo systemctl enable pdns-recursor
```

---

## 四、验证

```bash
# 测试内网域名解析
dig app.example.internal @127.0.0.1

# 测试外网域名解析（通过 recursor 转发）
dig baidu.com @127.0.0.1
```

---

## 故障排查

```bash
# 查看 pdns 日志
sudo journalctl -xeu pdns.service -n 50

# 查看 recursor 日志
sudo journalctl -xeu pdns-recursor.service -n 50

# 检查 53 端口是否被占用
sudo netstat -lnpt | grep :53
# 若被占用，找到进程并停止
sudo systemctl stop systemd-resolved
```

---

## 多域名配置

在 `named.conf` 中可以添加多个 zone：

```
zone "example.internal" IN {
    type master;
    file "example.internal.zone";
};

zone "k8s.internal" IN {
    type master;
    file "k8s.internal.zone";
};
```

在 `recursor.conf` 中同步添加转发：

```ini
forward-zones=example.internal=127.0.0.1:54
forward-zones+=k8s.internal=127.0.0.1:54
```
