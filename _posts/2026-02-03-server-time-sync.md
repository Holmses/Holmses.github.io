---
title: Linux 服务器时间同步配置指南
description: >-
  使用 systemd-timesyncd 和 NTP 实现 Linux 服务器时间同步，附自动化配置脚本
categories: [云平台/基础设施（Cloud & Infra）, linux]
tags: [linux, ntp, time-sync, systemd]
---

# Linux 服务器时间同步配置指南

服务器时间不准会导致日志时间混乱、SSL 证书验证失败、分布式系统数据不一致等问题。本文介绍 Ubuntu/Debian 系统下两种主流时间同步方案的配置方法。

## 方案对比

| 方案 | 说明 | 适用场景 |
|------|------|----------|
| `systemd-timesyncd` | systemd 内置轻量级 SNTP 客户端 | 普通服务器，精度要求不高 |
| `ntp` | 完整的 NTP 实现，精度更高 | 对时间精度要求高的场景，或需要当 NTP 服务端 |

---

## 一、查看当前时间状态

```bash
# 查看系统时间
date

# 查看详细时间同步状态
timedatectl
```

示例输出：

```
               Local time: Mon 2024-01-01 08:00:00 CST
           Universal time: Mon 2024-01-01 00:00:00 UTC
                 RTC time: Mon 2024-01-01 00:00:00
                Time zone: Asia/Shanghai (CST, +0800)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

---

## 二、设置时区

```bash
# 设置为上海时区（东八区）
sudo timedatectl set-timezone Asia/Shanghai

# 验证
timedatectl | grep "Time zone"
```

---

## 三、使用 systemd-timesyncd（推荐）

### 安装并启用

```bash
sudo apt update
sudo apt install -y systemd-timesyncd
```

### 配置 NTP 服务器

```bash
sudo vim /etc/systemd/timesyncd.conf
```

修改 `[Time]` 段，指定 NTP 服务器：

```ini
[Time]
NTP=10.0.0.1          # 内网 NTP 服务器地址
FallbackNTP=ntp.aliyun.com ntp1.aliyun.com
```

### 启用时间同步

```bash
# 开启 NTP 同步
sudo timedatectl set-ntp on

# 重启服务使配置生效
sudo systemctl restart systemd-timesyncd
sudo systemctl enable systemd-timesyncd

# 查看同步状态
timedatectl timesync-status
```

---

## 四、使用 NTP

### 安装

```bash
sudo apt update
sudo apt install -y ntp
```

### 配置 NTP 服务器

```bash
sudo vim /etc/ntp.conf
```

在配置文件末尾添加内网 NTP 服务器地址：

```
server 10.0.0.1 iburst
```

### 启用同步

```bash
# 先关闭 systemd-timesyncd 的 NTP 同步（避免冲突）
sudo timedatectl set-ntp off

# 手动立即同步
sudo ntpdate -u 10.0.0.1

# 开启 NTP 同步
sudo timedatectl set-ntp on
```

---

## 五、自动化配置脚本

适合批量配置多台服务器，将以下内容保存为 `time_sync.sh`：

```bash
#!/bin/bash
set -e

NTP_SERVER="10.0.0.1"    # 替换为实际的 NTP 服务器地址

echo "==> 设置时区为 Asia/Shanghai"
sudo timedatectl set-timezone Asia/Shanghai

echo "==> 安装 ntp 和 systemd-timesyncd"
sudo apt update -q
sudo apt install -y ntp systemd-timesyncd

echo "==> 关闭系统时间同步（避免冲突）"
sudo timedatectl set-ntp off

echo "==> 配置 NTP 服务器"
echo "server ${NTP_SERVER}" | sudo tee -a /etc/ntp.conf

echo "==> 手动同步时间"
sudo ntpdate -u "${NTP_SERVER}"

echo "==> 开启系统时间同步"
sudo timedatectl set-ntp on

echo "==> 完成，当前时间："
date
timedatectl
```

执行：

```bash
chmod +x time_sync.sh
./time_sync.sh
```

---

## 故障排查

```bash
# 查看 timesyncd 日志
sudo journalctl -u systemd-timesyncd -n 50

# 查看 NTP 同步状态
ntpq -p

# 检查 NTP 端口是否通
nc -uzv 10.0.0.1 123
```
