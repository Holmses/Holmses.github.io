---
title: Kubernetes 集群修改 IP 地址完整指南
description: >-
  生产环境 Kubernetes 集群整体迁移 IP 地址的完整操作步骤，涵盖 etcd、apiserver 证书重签和 Worker 节点重新加入
categories: [云平台/基础设施（Cloud & Infra）, kubernetes]
tags: [kubernetes, kubeadm, networking, troubleshooting]
---

# Kubernetes 集群修改 IP 地址完整指南

当服务器 IP 地址发生变更（如机房迁移、网段规划调整）时，Kubernetes 集群需要进行一系列操作才能恢复正常工作。由于 K8s 的 API Server 证书、etcd 证书等都绑定了 IP，不能简单地改网卡配置了事。

本文记录完整的操作流程，适用于 kubeadm 部署的集群。

> 操作前请确保做好备份，整个过程集群会有停机时间。

---

## 操作流程概览

1. 修改所有节点的网卡 IP 和 hosts
2. Master 节点重新初始化控制平面（重签证书）
3. Worker 节点重置并重新加入集群
4. 修复因 IP 变更失效的 Pod

---

## 一、修改所有节点网卡 IP 和 hosts

### CentOS 修改网卡配置

```bash
vi /etc/sysconfig/network-scripts/ifcfg-ens192

# 修改以下字段为新的 IP 信息
IPADDR=新IP地址
PREFIX=子网掩码位数
GATEWAY=新网关
DNS1=DNS服务器
```

### Ubuntu 修改 Netplan 配置

```bash
vim /etc/netplan/01-network.yaml

# 修改 addresses 和 gateway4 为新值
# 应用配置
netplan apply
```

### 修改 /etc/hosts（所有节点）

```bash
vim /etc/hosts
# 将集群所有节点的旧 IP 替换为新 IP
```

### 重启网卡

```bash
nmcli c reload
nmcli c up ens192    # 替换为实际网卡名
```

> 重启网卡后当前 SSH 会话会断开，通过新 IP 重新连接。

### 重启所有节点

```bash
reboot
```

---

## 二、Master 节点：重新初始化控制平面

### 1. 停止 kubelet 并备份

```bash
systemctl stop kubelet

# 备份关键目录
mv /etc/kubernetes /etc/kubernetes-bak
mv /var/lib/kubelet /var/lib/kubelet-bak
cp -r /var/lib/etcd /var/lib/etcd-bak    # etcd 数据备份
```

### 2. 保留证书目录，清理旧配置

重新初始化只需要保留 CA 证书，其他证书会重新签发：

```bash
mkdir -p /etc/kubernetes
cp -r /etc/kubernetes-bak/pki /etc/kubernetes

# 删除与旧 IP 绑定的证书（会重新生成）
rm -rf /etc/kubernetes/pki/apiserver.*
rm -rf /etc/kubernetes/pki/etcd/peer.*
rm -rf /etc/kubernetes/pki/etcd/server.*
```

### 3. 更新 kubeadm-init.yaml 中的 IP

```bash
cd /opt/kube-init
vi kubeadm-init.yaml

# 修改以下两处 IP 为新 IP：
# localAPIEndpoint.advertiseAddress
# controlPlaneEndpoint
```

### 4. 终止残留进程

```bash
# 检查 1025 端口是否有残留进程
netstat -lnp | grep 1025
kill -9 <PID>
```

### 5. 重新初始化控制平面

```bash
kubeadm init --config kubeadm-init.yaml --upload-certs \
  --ignore-preflight-errors=DirAvailable--var-lib-etcd
```

### 6. 更新 kubectl 配置

```bash
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 验证 Master 节点状态
kubectl get node
```

---

## 三、Worker 节点：重置并重新加入集群

在每台 Worker 节点上执行：

```bash
# 1. 重启节点
reboot

# 2. 重置 kubeadm（会清除 K8s 相关配置）
kubeadm reset
# 提示停止 Pod Sandbox 的错误可以忽略

# 3. 使用 Master 初始化后打印的命令重新加入
kubeadm join <NEW_MASTER_IP>:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

所有 Worker 节点加入后，重启 Master 节点使配置完全生效。

---

## 四、修复因 IP 变更失效的 Pod

IP 变更后，一些 Pod 中硬编码了旧 IP 的配置需要手动更新。

### NFS 存储（nfs-client-provisioner）

```bash
kubectl edit deployment nfs-client-provisioner -n nfs
# 将 NFS_SERVER 环境变量和 volumes.nfs.server 字段改为新的 NFS 服务器 IP
```

### 有状态应用（PVC/PV 绑定了旧 IP）

对于 Flink、Skywalking 等使用 PVC 的应用，需要删除 PVC 和 PV 重新创建：

```bash
# 以 Flink 为例
kubectl delete pod -n flink --all
kubectl delete pvc -n flink --all

# 删除 PV
kubectl get pv | grep flink | awk '{print $1}' | xargs kubectl delete pv

# 重新 apply 对应的 yaml 创建新的 PV/PVC
```

### Kuboard 管理界面

```bash
kubectl edit deployment kuboard-agent -n kuboard
kubectl edit deployment kuboard-agent-2 -n kuboard
# 更新 KUBOARD_ENDPOINT 和 KUBOARD_AGENT_HOST 的 IP
```

---

## 五、验证

```bash
# 所有节点应显示 Ready
kubectl get node

# 所有系统 Pod 应正常运行
kubectl get pod -n kube-system

# 检查证书已使用新 IP 签发
kubeadm certs check-expiration
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep IP
```

---

## 故障排查

### kubelet 无法启动

```bash
journalctl -u kubelet -n 100 --no-pager
# 通常是证书问题，确认 /etc/kubernetes/pki/ 下的证书是否正确生成
```

### etcd 连接失败

```bash
kubectl get pod -n kube-system | grep etcd
# 如果 etcd pod 异常，检查：
kubectl logs -n kube-system etcd-k8s-master
```

### 节点加入时 token 过期

```bash
# 在 Master 上重新生成 join 命令
kubeadm token create --print-join-command
```
