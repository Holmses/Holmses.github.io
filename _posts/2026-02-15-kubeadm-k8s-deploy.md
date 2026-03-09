---
title: CentOS Stream 使用 kubeadm 部署 Kubernetes 集群
description: >-
  从内核升级到集群初始化的完整步骤，含 containerd 配置、kubeadm 证书100年有效期编译、Calico 网络插件部署
categories: [云平台/基础设施（Cloud & Infra）, kubernetes]
tags: [kubernetes, kubeadm, containerd, calico, centos]
---

# CentOS Stream 使用 kubeadm 部署 Kubernetes 集群

本文基于 CentOS Stream 9，以 1 Master + 3 Node 的架构，完整记录 Kubernetes 1.27.6 的生产级部署流程，包含以下关键特性：

- 升级内核至 6.x
- 使用 containerd 作为容器运行时
- ipvs 模式的 kube-proxy
- 将 kubeadm 证书有效期修改为 100 年（避免频繁续签）
- Calico 网络插件

## 节点规划

| 角色 | 示例 IP |
|------|---------|
| master | 192.168.1.10 |
| node1 | 192.168.1.11 |
| node2 | 192.168.1.12 |
| node3 | 192.168.1.13 |

---

## 一、所有节点：换源并更新

```bash
# 替换为国内镜像源（USTC）
sudo sed -e 's|^mirrorlist=|#mirrorlist=|g' \
         -e 's|^#baseurl=http://mirror.centos.org/$contentdir|baseurl=https://mirrors.ustc.edu.cn/centos|g' \
         -i.bak \
         /etc/yum.repos.d/CentOS-Stream-AppStream.repo \
         /etc/yum.repos.d/CentOS-Stream-BaseOS.repo \
         /etc/yum.repos.d/CentOS-Stream-Extras.repo

dnf makecache

# 解决 locale 问题
echo "export LC_ALL=en_US.utf8" >> /etc/profile
source /etc/profile
```

---

## 二、所有节点：升级内核

Kubernetes 对内核版本有要求，建议升级到 6.x：

```bash
cd /opt

# 下载内核包（可替换为其他镜像源）
wget http://193.49.22.109/elrepo/kernel/el9/x86_64/RPMS/kernel-ml-core-6.2.1-1.el9.elrepo.x86_64.rpm
wget http://193.49.22.109/elrepo/kernel/el9/x86_64/RPMS/kernel-ml-modules-6.2.1-1.el9.elrepo.x86_64.rpm
wget http://193.49.22.109/elrepo/kernel/el9/x86_64/RPMS/kernel-ml-6.2.1-1.el9.elrepo.x86_64.rpm
wget http://193.49.22.109/elrepo/kernel/el9/x86_64/RPMS/kernel-ml-devel-6.2.1-1.el9.elrepo.x86_64.rpm

# 安装
rpm -ivh kernel-ml-core-6.2.1-1.el9.elrepo.x86_64.rpm
rpm -ivh kernel-ml-modules-6.2.1-1.el9.elrepo.x86_64.rpm
rpm -ivh kernel-ml-6.2.1-1.el9.elrepo.x86_64.rpm
rpm -ivh kernel-ml-devel-6.2.1-1.el9.elrepo.x86_64.rpm

# 设置新内核为默认启动
grub2-set-default 0 && grub2-mkconfig -o /etc/grub2.cfg
grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"

# 验证并重启
grubby --default-kernel
reboot
```

---

## 三、所有节点：安装前准备

```bash
# 1. 添加 docker-ce 源（containerd 在其中）
dnf config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 2. 设置主机名和 hosts（各节点分别执行，替换为实际 IP 和主机名）
hostnamectl set-hostname k8s-master
cat >> /etc/hosts <<EOF
192.168.1.10 k8s-master
192.168.1.11 k8s-node1
192.168.1.12 k8s-node2
192.168.1.13 k8s-node3
EOF

# 3. 关闭防火墙和 SELinux
systemctl disable --now firewalld
systemctl disable --now dnsmasq
setenforce 0
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config

# 4. 关闭 swap
swapoff -a && sysctl -w vm.swappiness=0
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab

# 5. 时间同步
dnf install -y chrony
systemctl enable --now chronyd
# 根据实际 NTP 服务器修改 /etc/chrony.conf
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

# 6. 调整文件描述符限制
ulimit -SHn 65535
cat >> /etc/security/limits.conf <<EOF
* soft nofile 655360
* hard nofile 131072
* soft nproc  655350
* hard nproc  655350
* soft memlock unlimited
* hard memlock unlimited
EOF

# 7. 安装常用工具
dnf install -y wget jq psmisc vim net-tools telnet lvm2 git

# 8. 安装 ipvs（kube-proxy ipvs 模式需要）
dnf install -y ipvsadm ipset sysstat conntrack libseccomp

modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack

cat > /etc/modules-load.d/ipvs.conf <<EOF
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
ip_tables
EOF

systemctl restart systemd-modules-load.service

# 9. 配置内核参数
cat > /etc/sysctl.d/k8s.conf <<EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory = 1
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 524288
fs.file-max = 52706963
net.netfilter.nf_conntrack_max = 2310720
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl = 15
net.core.somaxconn = 16384
EOF

# 加载桥接模块
cat > /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
sysctl --system

reboot
```

---

## 四、所有节点：安装 containerd

```bash
cd /opt

# 下载并解压（替换为最新版本）
wget https://github.com/containerd/containerd/releases/download/v1.6.25/cri-containerd-cni-1.6.25-linux-amd64.tar.gz
tar -xvf cri-containerd-cni-1.6.25-linux-amd64.tar.gz -C /

# 生成默认配置
containerd config default > /tmp/containerd-config.toml

# 关键修改：
# 1. 将 SystemdCgroup 改为 true
# 2. 将 sandbox_image 替换为国内可访问的镜像
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /tmp/containerd-config.toml
sed -i 's|registry.k8s.io/pause:.*|registry.aliyuncs.com/google_containers/pause:3.9"|' /tmp/containerd-config.toml

mkdir -p /etc/containerd
cp /tmp/containerd-config.toml /etc/containerd/config.toml
ln -s /usr/local/sbin/runc /usr/bin/runc

systemctl enable containerd
systemctl restart containerd
systemctl status containerd
```

---

## 五、所有节点：安装 kubeadm/kubelet/kubectl

```bash
# 添加阿里云 Kubernetes 镜像源
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
EOF

# 安装（以 1.27.6 为例）
dnf install -y kubelet-1.27.6 kubectl-1.27.6 kubeadm-1.27.6 --disableexcludes=kubernetes

systemctl enable kubelet

cat > /etc/sysconfig/kubelet <<EOF
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
EOF
```

---

## 六、修改 kubeadm 证书有效期为 100 年（仅 Master）

默认 kubeadm 签发的证书有效期为 1 年，生产环境需手动续签，操作繁琐。可通过修改源码将其改为 100 年。

```bash
# 安装 Go 1.20.10（与 K8s 1.27.6 编译版本对应）
cd /opt
wget https://go.dev/dl/go1.20.10.linux-amd64.tar.gz
tar xzvf go1.20.10.linux-amd64.tar.gz -C /usr/local

cat >> /etc/profile <<EOF
export PATH=\$PATH:/usr/local/go/bin
export GO111MODULE=auto
export GOPROXY=https://goproxy.cn
EOF
source /etc/profile

# 下载 K8s 1.27.6 源码
mkdir -p /opt/k8s-source
cd /opt/k8s-source
wget https://dl.k8s.io/v1.27.6/kubernetes-src.tar.gz
tar -zxvf kubernetes-src.tar.gz
```

修改 `cmd/kubeadm/app/constants/constants.go`，找到 `CertificateValidity` 将1年改为100年：

```go
// 原来：
CertificateValidity = time.Hour * 24 * 365
// 改为：
CertificateValidity = time.Hour * 24 * 365 * 100
```

修改 `staging/src/k8s.io/client-go/util/cert/cert.go`，找到 CA 证书有效期：

```go
// 原来：
NotAfter: now.Add(duration365d * 10).UTC(),
// 改为：
NotAfter: now.Add(duration365d * 100).UTC(),
```

编译并替换：

```bash
cd /opt/k8s-source
mv /usr/bin/kubeadm /usr/bin/kubeadm-ori   # 备份原始 kubeadm
make WHAT=cmd/kubeadm GOFLAGS=-v
cp _output/bin/kubeadm /usr/bin/kubeadm
```

---

## 七、初始化集群（仅 Master）

创建 `kubeadm-init.yaml`：

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: "0"    # 引导令牌永不过期
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.1.10    # Master IP
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  taints: null
---
controlPlaneEndpoint: "192.168.1.10:6443"
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: 1.27.6
imageRepository: registry.aliyuncs.com/google_containers
networking:
  dnsDomain: cluster.local
  serviceSubnet: 172.16.0.0/12
  podSubnet: 192.168.0.0/16
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
```

初始化：

```bash
# 使用原始 kubeadm 初始化 kubelet 配置
kubeadm-ori init phase kubelet-start --config kubeadm-init.yaml

# 使用修改后的 kubeadm（100年证书）初始化控制平面
kubeadm init --config kubeadm-init.yaml --upload-certs

# 配置 kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 八、Node 节点加入集群

在每台 Node 节点执行（命令从 `kubeadm init` 输出中复制）：

```bash
kubeadm join 192.168.1.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:<HASH>
```

---

## 九、安装 Calico 网络插件（Master）

```bash
mkdir -p /opt/calico && cd /opt/calico
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/calico.yaml -O

# 取消注释并设置 podSubnet（与 kubeadm-init.yaml 中的 podSubnet 一致）
# 找到 CALICO_IPV4POOL_CIDR 并取消注释，值设为 192.168.0.0/16

kubectl apply -f calico.yaml
kubectl get pod -n kube-system | grep calico
```

---

## 十、验证集群

```bash
# 查看节点状态（所有节点 Ready 即为成功）
kubectl get node

# 查看系统 Pod
kubectl get pod -n kube-system

# 验证证书有效期（应为 100 年）
kubeadm certs check-expiration
```

---

## 附：常用收尾操作

```bash
# 设置 worker 节点角色标签
kubectl label node k8s-node1 kubernetes.io/role=worker
kubectl label node k8s-node2 kubernetes.io/role=worker

# 安装 Metrics Server（需修改镜像地址和添加 --kubelet-insecure-tls）
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 定时清除 page cache（防止 buffer/cache 占满内存）
echo '0 */6 * * * root sync && echo 3 > /proc/sys/vm/drop_caches' >> /etc/crontab
```
