---
title: ClamAV：Linux 系统杀毒软件安装与使用
description: >-
  在 Ubuntu/Debian 系统上安装配置 ClamAV 开源杀毒软件，实现病毒扫描与定期更新
categories: [安全&监控(Security & Monitoring), linux]
tags: [linux, clamav, security, antivirus]
---

# ClamAV：Linux 系统杀毒软件安装与使用

ClamAV 是一款开源的跨平台杀毒引擎，在 Linux 服务器上被广泛用于邮件网关过滤、文件上传扫描和定期安全审计。本文介绍在 Ubuntu/Debian 系统上的完整安装配置流程。

## 安装

```bash
sudo apt update
sudo apt install -y clamav clamav-daemon
```

安装的两个组件：

- `clamav`：扫描引擎主程序，提供 `clamscan` 命令行工具
- `clamav-daemon`：后台守护进程 `clamd`，支持更快速的 `clamdscan` 扫描

验证安装：

```bash
clamscan --version
# 输出示例：ClamAV 0.103.11/26720/Tue Jul 30 10:02:13 2024
```

---

## 更新病毒数据库

ClamAV 通过 `freshclam` 工具更新病毒特征库，**首次使用前必须更新**，否则无法检测病毒。

```bash
# 停止自动更新服务（避免冲突）
sudo systemctl stop clamav-freshclam

# 手动立即更新病毒库
sudo freshclam

# 更新完成后，启动自动更新服务并设置开机自启
sudo systemctl enable clamav-freshclam --now
```

---

## 扫描文件和目录

### 使用 clamscan（单进程，适合临时扫描）

```bash
# 扫描单个文件
clamscan /path/to/file

# 递归扫描目录
clamscan -r /path/to/directory

# 只显示发现病毒的文件（安静模式）
clamscan -r --infected /path/to/directory

# 扫描时显示进度提示音
clamscan -r --bell /path/to/directory

# 将扫描结果保存到日志文件
clamscan -r /path/to/directory --log=/var/log/clamav/scan.log
```

### 使用 clamdscan（调用守护进程，速度更快）

`clamdscan` 通过 Unix socket 与后台 `clamd` 进程通信，扫描速度比 `clamscan` 快得多：

```bash
# 先确保 clamd 服务运行
sudo systemctl start clamav-daemon

# 扫描单个文件
clamdscan /path/to/file

# 递归扫描目录
clamdscan -r /path/to/directory
```

---

## 自动化：定时扫描脚本

创建定期扫描脚本 `/opt/scripts/clamav-scan.sh`：

```bash
#!/bin/bash

LOG_DIR="/var/log/clamav"
LOG_FILE="${LOG_DIR}/scan-$(date +%Y%m%d).log"
SCAN_DIR="/"    # 扫描根目录，可按需修改

mkdir -p "$LOG_DIR"

echo "=== ClamAV 扫描开始: $(date) ===" >> "$LOG_FILE"

clamscan -r --infected \
  --exclude-dir="^/sys" \
  --exclude-dir="^/proc" \
  --exclude-dir="^/dev" \
  "$SCAN_DIR" >> "$LOG_FILE" 2>&1

echo "=== 扫描结束: $(date) ===" >> "$LOG_FILE"
```

加入 cron 定时任务（每天凌晨2点执行）：

```bash
chmod +x /opt/scripts/clamav-scan.sh

# 编辑 crontab
crontab -e
# 添加以下内容
0 2 * * * /opt/scripts/clamav-scan.sh
```

---

## 常见使用场景

### 扫描上传文件目录

适合 Web 服务器对用户上传目录进行实时或定期扫描：

```bash
clamscan -r --infected /var/www/uploads/
```

### 发现病毒后自动隔离

```bash
# --move 将感染文件移动到隔离目录
clamscan -r --infected --move=/var/quarantine /data/uploads/
```

---

## 服务管理

```bash
# 查看状态
sudo systemctl status clamav-daemon
sudo systemctl status clamav-freshclam

# 手动触发病毒库更新
sudo freshclam

# 查看日志
sudo journalctl -u clamav-daemon -n 50
```
