---
title: VPN 简易使用
description: >-
  致未曾翻墙过的朋友
categories: [工具, 教程]
tags: [tools]

---

## virtual-private-network(VPN)

### Windows

windows 常用的代理软件是 Clash for windows，但由于作者删库跑路，改用 [Clash Verge][clashverge]，直接在官网下载[Clash.Verge_2.4.0+autobuild.0820.7613417_x64-setup.exe](https://github.com/clash-verge-rev/clash-verge-rev/releases/download/autobuild/Clash.Verge_2.4.0+autobuild.0820.7613417_x64-setup.exe)安装即可。

然后需要一个稳定的代理服务器，有条件的可以自建，没有的建议直接购买别人搭建好的"机场“。这里推荐我正在使用的 [CuteCloud][cutecloud]，用了几年没什么问题。新用户可以填写邀请码 X0xedUi2，享受免费试用。

购买好之后直接在主页仪表盘中，点击 [Clash 订阅]，即可自动导入到 [Clash Verge][clashverge]。然后在 Clash 中开启系统代理

![image-20250821090215825](https://raw.githubusercontent.com/Holmses/Holmses.github.io/master/assets/img/image-20250821090215825.png)

最后在代理中选择延迟较低的节点即可魔法上网。

### IOS

有人需要再更新 #

### Android

有人需要再更新 #

### Linux

没有想到是我自己先需要上了，更新一版linux的vpn，也是用的 Clash Verge 的同款内核[mihomo][mihomo]。

先下载[mihomo-linux-amd64-v1.19.18.deb](https://github.com/MetaCubeX/mihomo/releases/download/v1.19.18/mihomo-linux-amd64-v1.19.18.deb)，这个好了，ubuntu的安装包然后`dpkg -i 安装包名称`，安装一下,然后把yaml文件放在`/etc/mihomo/config.yaml`

```
# 以此配置文件启动
mihomo -d /etc/mihomo/
# 配置本机代理
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export all_proxy="socks5://127.0.0.1:7890"
# 测试代理
curl -I www.google.com
# 取消代理
unset http_proxy https_proxy all_proxy

# 停止mihomo
sudo systemctl top mihomo
```

## Cutecloud

套餐价格

![image-20250821085616522](https://raw.githubusercontent.com/Holmses/Holmses.github.io/master/assets/img/image-20250821085616522.png)

长期流量

![image-20250821085702892](https://raw.githubusercontent.com/Holmses/Holmses.github.io/master/assets/img/image-20250821085702892.png)

流量使用不多的用户推荐直接购买流量

## References

1. [clashverge][]
2. [cutecloud][]
3. [mihomo][]

[clashverge]:https://github.com/Clash-Verge-rev/clash-verge-rev/releases
[cutecloud]:https://2.cutecloud.net/register?code=7SYU3S11
[mihomo]:https://github.com/MetaCubeX/mihomo/releases/tag/v1.19.18