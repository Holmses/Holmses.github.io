---
title: Wireguard
description: >-
  prometheus 在云服务器使用 WireGuard 实现流量转发绕过防火墙或 DNS 屏蔽
categories: [云平台/基础设施（Cloud & Infra）, wireguard]
tags: [wirguard]
---

# Wireguard

部分网络对游戏或者娱乐等网络做了限制，通过 wireguard 的流量转发可以绕过这个限制。

## 为什么选择 WireGuard？

- **轻量高效**：代码简洁，审计成本低，安全性更易保障
- **速度快**：采用现代加密算法（ChaCha20、Curve25519 等），性能损耗小
- **跨平台**：支持 Linux、Windows、macOS、Android、iOS 等主流系统
- **易配置**：无需复杂的证书管理，通过公钥认证实现身份验证

## 环境准备

- **服务端**：一台具有公网 IP 的 Linux 服务器（本文以 Ubuntu 22.04 为例）
- **客户端**：根据需求选择对应的操作系统（Windows/macOS/Android/iOS/Linux）
- **网络要求**：服务端需开放 UDP 端口（默认 51820）

## 服务端部署步骤

### 1. 安装 WireGuard

```bash
# Ubuntu/Debian 系统
sudo apt update
sudo apt install wireguard -y

# CentOS/RHEL 系统
sudo yum install epel-release -y
sudo yum install wireguard-tools -y
2. 生成密钥对
WireGuard 使用非对称加密，需为服务端和客户端生成密钥对：
bash
# 创建存放配置的目录
sudo mkdir -p /etc/wireguard
cd /etc/wireguard

# 生成服务端密钥对（私钥会自动隐藏输出，仅保存到文件）
sudo wg genkey | sudo tee server_private.key | sudo wg pubkey | sudo tee server_public.key

# 生成客户端密钥对（以第一个客户端为例）
sudo wg genkey | sudo tee client1_private.key | sudo wg pubkey | sudo tee client1_public.key
3. 配置服务端
创建服务端配置文件 wg0.conf：
bash
sudo nano /etc/wireguard/wg0.conf
配置内容如下：
ini
[Interface]
# 服务端私钥（从 server_private.key 文件中获取）
PrivateKey = <服务器私钥>
# 监听端口（默认 51820，需在防火墙开放）
ListenPort = 51820
# 服务端在 VPN 子网中的 IP（CIDR 格式）
Address = 10.0.0.1/24
# 加上可以解决大流量加载超时的问题
MTU = 1420

# 配置 IP 转发（允许客户端通过服务端访问互联网）
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
注意：
替换 <服务器私钥> 为 server_private.key 文件中的内容
将 eth0 替换为服务器的公网网卡（可通过 ip addr 命令查看）
如需禁用互联网访问，可删除 PostUp 和 PostDown 配置
4. 启动服务端并设置自启
bash
# 启动 WireGuard 服务
sudo wg-quick up wg0

# 设置开机自启
sudo systemctl enable wg-quick@wg0
客户端配置步骤
1. 准备客户端配置文件
创建客户端配置文件 client1.conf（可在服务端生成后复制到客户端）：
ini
[Interface]
# 客户端私钥（从 client1_private.key 文件中获取）
PrivateKey = <客户端私钥>
# 客户端在 VPN 子网中的 IP（需与服务端在同一网段）
Address = 10.0.0.2/24
# DNS 服务器（可选，推荐使用 114.114.114.114 或 8.8.8.8）
DNS = 114.114.114.114
# 需要和server端同步
MTU = 1420

[Peer]
# 服务端公钥（从 server_public.key 文件中获取）
PublicKey = <服务器公钥>
# 服务端公网 IP 和端口（格式：IP:端口）
Endpoint = <服务器公网IP>:51820
# 客户端通过 VPN 访问的网段（0.0.0.0/0 表示所有流量走 VPN）
AllowedIPs = 0.0.0.0/0
# 保持连接（每隔 25 秒发送一次心跳包，适用于 NAT 环境）
PersistentKeepalive = 25
说明：
AllowedIPs 配置说明：
0.0.0.0/0：所有流量通过 VPN
10.0.0.0/24：仅访问 VPN 子网时走 VPN
可自定义网段（如公司内网 192.168.1.0/24）
2. 将客户端添加到服务端
编辑服务端配置文件，添加客户端信息：
bash
sudo nano /etc/wireguard/wg0.conf
在文件末尾添加：
ini
[Peer]
# 客户端公钥（从 client1_public.key 文件中获取）
PublicKey = <客户端公钥>
# 客户端在 VPN 中的 IP（需与客户端配置一致）
AllowedIPs = 10.0.0.2/32
重新加载服务端配置：
bash
sudo wg-quick down wg0
sudo wg-quick up wg0
3. 客户端连接方法
Linux 客户端
bash
# 将 client1.conf 复制到 /etc/wireguard/ 目录
sudo cp client1.conf /etc/wireguard/

# 启动连接
sudo wg-quick up client1
Windows 客户端
下载官方客户端：WireGuard for Windows
安装后打开客户端，点击「导入隧道」→「从文件导入隧道」
选择 client1.conf 文件，点击「连接」
移动设备（Android/iOS）
在应用商店搜索并安装 WireGuard 客户端
打开客户端，点击「+」号，选择「导入配置文件」
扫描二维码或手动输入配置内容，点击「连接」
验证与管理
查看连接状态
bash
# 服务端查看所有客户端连接
sudo wg

# 客户端查看连接状态
sudo wg show
测试连通性
bash
# 客户端 ping 服务端 VPN 地址
ping 10.0.0.1

# 服务端 ping 客户端 VPN 地址
ping 10.0.0.2
常用命令
bash
# 启动指定接口
sudo wg-quick up wg0

# 停止指定接口
sudo wg-quick down wg0

# 查看接口状态
sudo wg show wg0

# 重启服务
sudo systemctl restart wg-quick@wg0
```

### 安全注意事项

密钥管理：私钥是身份凭证，切勿泄露给他人，建议定期更换密钥对

端口防护：仅开放必要的 UDP 端口（默认 51820），并限制来源 IP

权限控制：通过 AllowedIPs 严格限制客户端可访问的网段

系统加固：确保服务器系统为最新状态，关闭不必要的服务

日志监控：定期查看 WireGuard 运行日志，及时发现异常连接
### 总结

WireGuard 以其简洁的设计和高效的性能，为构建私人 VPN 提供了优秀的解决方案。通过本文的步骤，你可以快速搭建一套安全可靠的 VPN 服务，用于远程办公、数据加密传输等场景。相比传统 VPN 方案，WireGuard 不仅配置简单，还能在保证安全性的同时提供更优的网络性能。





## References

1. [在云服务器使用 WireGuard 实现异地局域网互联][wireguard]

[wireguard]:https://lvv.me/posts/2024/05/25_easy_wireguard/
