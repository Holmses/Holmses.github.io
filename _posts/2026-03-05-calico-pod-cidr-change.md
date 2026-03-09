---
title: Kubernetes 修改 Calico Pod CIDR
description: >-
  在不重建集群的情况下，修改 Kubernetes 集群的 Pod 子网（CIDR），包括 Calico IPPool、kube-controller-manager 和各节点 podCIDR 的完整步骤
categories: [云平台/基础设施（Cloud & Infra）, kubernetes]
tags: [kubernetes, calico, networking, cidr]
---

# Kubernetes 修改 Calico Pod CIDR

在 Kubernetes 集群投入使用后，有时会发现初始规划的 Pod 网段与现有网络冲突，或需要扩大 Pod IP 地址空间。本文记录在不重建集群的前提下，修改 Pod CIDR 的完整步骤。

> 此操作会导致所有 Pod 重启，请在维护窗口期执行。

## 整体思路

Calico 使用 IPPool 对象管理 Pod IP 地址池，修改 CIDR 需要：

1. 创建新的 IPPool
2. 禁用旧的 IPPool
3. 更新各节点的 `podCIDR` 字段
4. 更新 kube-controller-manager 的 `--cluster-cidr`
5. 更新相关 ConfigMap
6. 重启所有 Pod 使其获取新 IP
7. 清理旧的 IPPool

---

## 一、创建新的 IPPool

创建 `new-ippool.yaml`（将 cidr 替换为目标网段）：

```yaml
apiVersion: v1
items:
- apiVersion: crd.projectcalico.org/v1
  kind: IPPool
  metadata:
    name: new-ipv4-ippool
  spec:
    allowedUses:
    - Workload
    - Tunnel
    blockSize: 24       # 每个节点分配的子网掩码位数，/24 即每节点 256 个 IP
    cidr: 10.18.16.0/20 # 替换为新的 Pod 网段
    ipipMode: Always
    natOutgoing: true
    nodeSelector: all()
    vxlanMode: Never
kind: List
metadata:
  resourceVersion: ""
```

```bash
kubectl apply -f new-ippool.yaml

# 验证新 IPPool 已创建
kubectl get ippool
```

---

## 二、禁用旧的 IPPool

```bash
kubectl edit ippool default-ipv4-ippool
```

在 `spec:` 下进行以下修改：

```yaml
spec:
  disabled: true             # 新增：禁用旧 IPPool，不再为新 Pod 分配 IP
  nodeSelector: "!all()"    # 修改：不选择任何节点
  # 其他字段保持不变
```

保存退出（`:wq`）后，新 Pod 将从新 IPPool 分配 IP。

---

## 三、修改各节点的 podCIDR

每个节点都记录了自己的 `podCIDR`，需要逐一更新。

```bash
# 导出所有节点的 yaml（包括 master 节点）
kubectl get nodes --no-headers | awk '{print $1}' | while read node; do
  kubectl get node "$node" -o yaml > "${node}.yaml"
done
```

编辑每个节点的 yaml 文件，找到并修改 `podCIDR` 字段（通常有两处）：

```yaml
spec:
  podCIDR: 10.18.16.0/24      # 修改为新网段的对应子网
  podCIDRs:
  - 10.18.16.0/24             # 同步修改
```

删除旧节点对象并重新 apply：

```bash
# 注意：删除节点会导致其上的 Pod 被驱逐，请逐个操作
kubectl delete node <节点名> && kubectl apply -f <节点名>.yaml

# 等待节点重新 Ready 后再操作下一个节点
kubectl get node -w
```

---

## 四、更新 kubeadm ConfigMap

```bash
kubectl -n kube-system edit cm kubeadm-config
# 找到 podSubnet 字段，修改为新的 CIDR
```

---

## 五、更新 kube-proxy ConfigMap

```bash
kubectl -n kube-system edit cm kube-proxy
# 找到 clusterCIDR 字段，修改为新的 CIDR
```

---

## 六、更新 kube-controller-manager

```bash
vim /etc/kubernetes/manifests/kube-controller-manager.yaml
# 找到 --cluster-cidr 参数，修改为新的 CIDR
```

修改后 kube-controller-manager 会自动重启（静态 Pod）。

---

## 七、删除所有 Pod（触发重新调度获取新 IP）

```bash
# 删除所有非系统命名空间的 Pod
kubectl delete pod --all -A --field-selector 'metadata.namespace!=kube-system'

# 系统 Pod 也需要重启
kubectl delete pod -n kube-system --all
```

等待所有 Pod 重新运行，新 Pod 会从新 IPPool 获取 IP。

---

## 八、更新 Calico 配置并重新 apply

等所有 Pod 启动成功后，更新 calico.yaml：

```bash
vim /opt/calico/calico.yaml
# 找到 CALICO_IPV4POOL_CIDR 环境变量
# 取消注释并将值改为新的 CIDR
```

重新 apply：

```bash
kubectl apply -f /opt/calico/calico.yaml
```

---

## 九、删除旧的 IPPool

确认所有 Pod 都已获取新 IP 后，删除旧的 IPPool：

```bash
kubectl delete ippool default-ipv4-ippool

# 验证只剩新的 IPPool
kubectl get ippool
```

---

## 验证

```bash
# 查看 Pod IP 是否都已切换到新网段
kubectl get pod -A -o wide | awk '{print $7}' | sort -u

# 查看节点 podCIDR
kubectl get node -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
```
