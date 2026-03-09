---
title: Linux LVM 磁盘扩容指南
description: >-
  CentOS 和 Ubuntu 系统下使用 LVM 在线扩容磁盘空间的完整步骤
categories: [云平台/基础设施（Cloud & Infra）, linux]
tags: [linux, lvm, disk, centos, ubuntu]
---

# Linux LVM 磁盘扩容指南

在生产环境中，随着业务增长经常需要对服务器进行磁盘扩容。LVM（逻辑卷管理器）支持在线扩容，无需重启服务器。本文分别介绍 CentOS 和 Ubuntu 系统下的完整扩容步骤。

## 前置知识

LVM 的核心概念：

- **PV（Physical Volume）**：物理卷，即实际的磁盘分区
- **VG（Volume Group）**：卷组，将多个 PV 组合成一个存储池
- **LV（Logical Volume）**：逻辑卷，从 VG 中划分出来供系统使用

扩容流程：新分区 → 创建 PV → 加入 VG → 扩展 LV → 扩展文件系统

---

## 一、CentOS 7 磁盘扩容

### 查看当前磁盘状态

```bash
df -h
lsblk
```

示例输出：

```
NAME            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda               8:0    0   5.2T  0 disk
├─sda1            8:1    0     1M  0 part
├─sda2            8:2    0     1G  0 part /boot
├─sda3            8:3    0     5T  0 part
│ ├─centos-root 253:0    0 249.9G  0 lvm  /
│ ├─centos-swap 253:1    0   7.9G  0 lvm  [SWAP]
│ └─centos-home 253:2    0     5T  0 lvm  /home
```

### 扩容步骤

#### 1. 创建新分区

```bash
fdisk /dev/sda
# 交互式操作：依次输入 n → 回车（新建分区）→ 回车（使用默认起始扇区）→ 回车（使用全部剩余空间）→ w（写入）
```

操作完成后会新增 `/dev/sda4` 分区。

#### 2. 创建物理卷

```bash
pvcreate /dev/sda4
pvdisplay
# 记录 VG Name，后续步骤需要用到
```

#### 3. 将物理卷加入卷组

```bash
vgextend centos /dev/sda4
```

将 `centos` 替换为实际的 VG Name。

#### 4. 扩展逻辑卷

```bash
# 扩展指定大小（如 +100G）
lvresize -L +100G /dev/mapper/centos-root
```

#### 5. 扩展 XFS 文件系统

```bash
xfs_growfs /dev/mapper/centos-root
```

> 注意：CentOS 7 默认使用 XFS 文件系统，使用 `xfs_growfs`；如果是 ext4 则使用 `resize2fs`。

#### 6. 验证结果

```bash
df -h
```

---

## 二、Ubuntu 磁盘扩容

Ubuntu 使用 ext4 文件系统，命令略有不同。

### 查看当前状态

```bash
fdisk -l
df -h
pvs
```

### 扩容步骤

#### 1. 对新磁盘进行分区

```bash
fdisk /dev/sdc
# 依次输入 n → p（主分区）→ 回车 → 回车 → 回车 → w
```

#### 2. 创建物理卷并加入卷组

```bash
pvcreate /dev/sdc1
vgextend ubuntu-vg /dev/sdc1
```

将 `ubuntu-vg` 替换为实际的 VG 名称（通过 `vgdisplay` 查看）。

#### 3. 扩展逻辑卷

```bash
# 将剩余所有 FREE 空间全部分配
sudo lvresize -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv

# 或者指定大小
sudo lvresize -A n -L +50G /dev/mapper/ubuntu--vg-ubuntu--lv
```

#### 4. 扩展 ext4 文件系统

```bash
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

#### 5. 验证结果

```bash
df -h
sudo vgdisplay
```

---

## 常用 LVM 命令速查

| 命令 | 说明 |
|------|------|
| `pvs` / `pvdisplay` | 查看物理卷 |
| `vgs` / `vgdisplay` | 查看卷组 |
| `lvs` / `lvdisplay` | 查看逻辑卷 |
| `pvcreate /dev/sdX` | 创建物理卷 |
| `vgextend VG /dev/sdX` | 扩展卷组 |
| `lvresize -L +NGB /dev/VG/LV` | 扩展逻辑卷 |
| `xfs_growfs /mount/point` | 扩展 XFS 文件系统 |
| `resize2fs /dev/VG/LV` | 扩展 ext4 文件系统 |
