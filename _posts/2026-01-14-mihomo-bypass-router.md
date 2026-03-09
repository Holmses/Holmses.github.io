---
title: Mihomo Bypass Router
description: >-
  Mihomo 内核ubuntu部署成旁路由供子网内使用教程
categories: [云平台/基础设施（Cloud & Infra）, mihomo]
tags: [mihomo]
---


# Ubuntu 22.04 安装 Mihomo 作为旁路由（透明网关）完整教程

## 一、前置准备
| 条件                | 说明                                                                 |
|---------------------|----------------------------------------------------------------------|
| 系统要求            | Ubuntu 22.04 LTS（树莓派、迷你主机、虚拟机均可）                     |
| 网络基础            | 设备已联网，获取内网静态 IP（例：`192.168.100.2`，后续作为旁路由网关） |
| 工具依赖            | 已安装 `vim`（无则执行：`sudo apt install vim -y`）                   |
| 核心前提            | 需使用 Mihomo（Clash.Meta 后续维护版），普通 Clash 不支持 TUN/Sniffer 功能 |

## 二、步骤 1：下载并安装 Mihomo
### 1.1 创建目录并下载 Mihomo 核心
```bash
# 创建 Mihomo 配置与核心存放目录
sudo mkdir -p /etc/mihomo
cd /etc/mihomo

# 下载最新版 Mihomo（AMD64 架构，其他架构请替换官网链接）
sudo wget https://github.com/MetaCubeX/mihomo/releases/download/v1.19.18/mihomo-linux-amd64-v1-v1.19.18.deb

# 安装deb文件
sudo dpkg -i mihomo-linux-amd64-v1-v1.19.18.deb
```

### 1.2 注册为系统服务（开机自启）
```bash
# 设置为开机启动
sudo systemctl enable mihomo
```

注册服务内容：
```ini
[Unit]
Description=Mihomo Service
Documentation=https://wiki.metacubex.one/
After=network.target nss-lookup.target

[Service]
User=root
WorkingDirectory=/etc/mihomo
ExecStart=/etc/mihomo/mihomo -f /etc/mihomo/config.yaml
Restart=on-failure
RestartSec=5s
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

### 1.3 重载服务并测试启动
```bash
# 重载系统服务
sudo systemctl daemon-reload

# 启用并启动 Mihomo
sudo systemctl enable --now mihomo

# 查看服务状态（显示 active (running) 即为正常）
sudo systemctl status mihomo
```

## 三、步骤 2：配置 Mihomo（关键：适配 Ubuntu DNS 机制）
### 2.1 创建完整配置文件
```bash
sudo vim /etc/mihomo/config.yaml
```

粘贴以下配置（**必须替换 2 处关键信息**）：
```yaml
# 基础网络配置
ipv6: true                  # 允许 IPv6 流量
allow-lan: true             # 允许局域网设备连接代理
unified-delay: false        # 关闭统一延迟（按需开启）
tcp-concurrent: true        # 启用 TCP 并发连接
find-process-mode: strict   # 进程匹配模式（严格）
global-client-fingerprint: chrome  # 全局 TLS 指纹（伪装 Chrome）

# 配置存储策略
profile:
  store-selected: true      # 保存策略组选择（重启后生效）
  store-fake-ip: true       # 保存 fake-ip 映射表（加速重复访问）

# 核心功能：域名嗅探（解决 Ubuntu systemd-resolved DNS 拦截问题）
sniffer:
  enable: true
  sniff:
    HTTP:
      ports: [80, 8080-8880]
      override-destination: true  # 覆盖目标地址（确保嗅探生效）
    TLS:
      ports: [443, 8443]
    QUIC:
      ports: [443, 8443]
  skip-domain:  # 跳过嗅探的域名（按需添加本地设备域名）
    - "Mijia Cloud"
    - "+.push.apple.com"

# 核心功能：TUN 虚拟网卡（透明代理基础）
tun:
  enable: true
  stack: mixed              # 混合栈（同时支持 IPv4/IPv6）
  dns-hijack:
    - "any:53"              # 劫持所有 UDP 53 端口 DNS 请求
    - "tcp://any:53"        # 劫持所有 TCP 53 端口 DNS 请求
  auto-route: true          # 自动生成路由规则
  auto-redirect: true       # 自动重定向流量到 TUN 网卡
  auto-detect-interface: true  # 自动检测外网网卡

# DNS 配置（配合 fake-ip 与嗅探功能）
dns:
  enable: true
  ipv6: true
  enhanced-mode: fake-ip    # 启用 fake-ip 模式（防 DNS 污染）
  fake-ip-filter:  # 不进行 fake-ip 解析的域名（局域网/本地域名）
    - "*"
    - "+.lan"
    - "+.local"
    - "+.market.xiaomi.com"
  default-nameserver:  # 默认 DNS 服务器（国内合规节点）
    - tls://223.5.5.5
    - tls://223.6.6.6
  nameserver:  # 备用 DNS（DoH 协议，提高稳定性）
    - https://doh.pub/dns-query
    - https://dns.alidns.com/dns-query

# ###########################
# 必改项 1：替换为你的代理节点
# 示例为 VMess 节点，实际替换为你的节点信息（支持 Shadowsocks/VLESS 等）
# ###########################
proxies:
  - name: "我的代理节点"
    type: vmess
    server: 1.2.3.4  # 替换为节点服务器 IP
    port: 10086       # 替换为节点端口
    uuid: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  # 替换为节点 UUID
    alterId: 0
    cipher: auto
    network: tcp

# ###########################
# 策略组配置（自动选择最优节点）
# ###########################
proxy-groups:
  - name: "自动选择"
    type: url-test
    proxies:
      - "我的代理节点"  # 与上面的节点名称一致
    url: "http://www.gstatic.com/generate_204"  # 测速地址
    interval: 300  # 每 5 分钟测速一次

# ###########################
# 路由规则（按需调整）
# 规则顺序：从上到下匹配，匹配成功即生效
# ###########################
rules:
  - DOMAIN-SUFFIX,google.com,自动选择  # 谷歌走代理
  - DOMAIN-SUFFIX,youtube.com,自动选择 # YouTube 走代理
  - DOMAIN-SUFFIX,github.com,自动选择  # GitHub 走代理
  - GEOIP,CN,DIRECT  # 国内 IP 直连
  - MATCH,自动选择    # 其他所有流量走自动选择
```

### 2.2 重启 Mihomo 加载配置
```bash
# 重启服务使配置生效
sudo systemctl restart mihomo

# 查看日志，确认无报错（按 Ctrl+C 退出）
sudo journalctl -u mihomo -f
```

## 四、步骤 3：开启 Ubuntu 流量转发
```bash
# 编辑内核参数配置文件
sudo vim /etc/sysctl.conf
```

找到以下两行，**删除开头的 # 注释**：
```bash
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

加载内核参数（无需重启系统）：
```bash
sudo sysctl -p
```

验证：执行以下命令，输出 `1` 即为成功：
```bash
sysctl net.ipv4.ip_forward
```

## 五、步骤 4：配置局域网设备（以 iOS 为例）
### 设备网络设置（所有设备通用逻辑）
1. 连接家庭 WiFi，进入「WiFi 详情」；
2. 「配置 IP」→ 选择「手动」：
   - IP 地址：手动分配同网段 IP（例：`192.168.100.10`，避免与其他设备冲突）；
   - 子网掩码：`255.255.255.0`；
   - 路由器：输入旁路由内网 IP（例：`192.168.100.2`）；
3. 「配置 DNS」→ 选择「手动」：
   - 添加 DNS 服务器：输入旁路由内网 IP（例：`192.168.100.2`）；
4. 保存配置，无需安装任何代理工具即可科学上网。

### 其他设备配置参考
- Windows：设置 → 网络和 Internet → WiFi → 选中网络 → 属性 → IPv4 手动设置（同上述参数）；
- Android：WiFi 长按 → 修改网络 → 高级选项 → IPv4 手动（同上述参数）；
- 智能设备（电视/盒子）：网络设置中手动指定网关和 DNS 为旁路由 IP。

## 六、验证与问题处理
### 6.1 验证代理是否生效
1. 局域网设备访问 `https://google.com`，能打开则成功；
2. 查看 Mihomo 日志，确认流量转发记录：
```bash
sudo journalctl -u mihomo -f | grep -E "sniff|proxy|DNS"
```

### 6.2 常见问题解决
| 问题现象                          | 解决方法                                                                 |
|-----------------------------------|--------------------------------------------------------------------------|
| DNS 拦截失效（域名规则不匹配）    | 1. 确认 `sniffer` 配置已启用；2. 检查 Mihomo 日志是否有 `sniff` 相关记录；3. 无需禁用 `systemd-resolved` |
| 设备无法上网                      | 1. 旁路由测试 `ping 8.8.8.8` 是否通；2. 检查代理节点配置是否正确；3. 重启 Mihomo：`sudo systemctl restart mihomo` |
| Nginx/Homeassistant 等服务异常    | 调整启动顺序：先启动依赖 DNS 的服务，再启动 Mihomo；修改 Mihomo 服务文件添加 `After=homeassistant.service` |
| 旁路由自身无法上网                | 检查 `sysctl net.ipv4.ip_forward` 是否为 1；确认代理节点可正常连接         |

## 七、核心总结
1. 关键配置：`TUN 模式 + Sniffer 嗅探 + fake-ip DNS`，完美适配 Ubuntu 22.04 的 `systemd-resolved` 机制；
2. 核心优势：局域网设备免装代理，自动转发流量，支持 IPv4/IPv6，稳定性高于 OpenClash；
3. 注意事项：
   - 代理节点需提前测试可用；
   - 旁路由需保持开机状态；
   - 避免局域网 IP 冲突（建议给旁路由分配静态 IP）。

## 参考文档
- Mihomo 官方文档：https://wiki.metacubex.one/
- 原文链接：https://blog.esunr.site/2025/04/ec8dc9f7a09d.html