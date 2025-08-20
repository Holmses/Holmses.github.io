---
title: Typora 图片上传指南
description: >-
  使用 Jekyll 想要一个图床的联想
date: 2025-08-20 10:55:00 +0800
categories: [博客, 教程]
tags: [Jekyll, GitHub, Typora, PicGo]
---

## GitHub 图床

#### 新建仓库

首先在 GitHub 新建一个 Public 仓库，我这边直接把文件上传到我的博客的 assets/img 中，就不新建仓库了

#### 上传图片

Typora 中提供的是 [PicGo ][picGo] 那我们就用这个开源图传工具，Typora 提供了两种使用方式

- **PiGo-Core(command line)**

  PicGo-Core下载需要使用Node.js，没有的话请先安装[Node.js][nodejs]。

   终端中输入：**`npm install picgo -g`** 即开始安装PicGo-Core，安装好后  bash 中可以用 `which picgo` 查找picgo 的位置，然后修改配置文件 config.json

  > linux 和 macOS 均为`~/.picgo/config.json`
  >
  > windows 则为`C:\Users\你的用户名\.picgo\config.json`
  
  > 使用PicGo-Core上传会消耗较少的计算资源，只会在上传过程中运行，并且在上传成功或失败后会退出；PicGo(app)上传时，始终保持运行，无法自动退出，消耗更多的计算资源。
  {: .prompt-tip }




- **PicGo(app)**

  推荐使用图形化界面，操作简单

  配置需要一个GitHub token, 在`Settings -> Developer settings -> Personal access tokens`，最后点击 `generate new token`；**填写用途，勾选repo**

  配置界面

  ![image-20250820103753328](https://raw.githubusercontent.com/Holmses/Holmses.github.io/master/assets/img/image-20250820103753328.png)
  
  > PicGo(app)比PicGo-Core版本提供更多功能，如上传历史记录，详细看官方介绍；
  {: .prompt-tip }




## Typora 配置图片上传

配置界面

![image-20250820104757493](https://raw.githubusercontent.com/Holmses/Holmses.github.io/master/assets/img/image-20250820104757493.png)

#### 配置 PicGo-Core 

进入Typora设置，上传服务选中 **`Custom Command`**，命令输入 **`picgo upload`**。

> `picgo upload`：命令需要node.js和picgo都存在环境变量，不然无法执行，
>
> 若没有配置环境变量，可以指定node.js和picgo安装路径，
>
> 命令格式：`[node path] [picgo-core path] upload|u`
>
> 命令示例：`C:\Program Files\nodejs C:\Users\xx\picgo u`



## References

1. [picgo][]
1. [nodejs][]

[picgo]: https://molunerfinn.com/PicGo/
[nodejs]: https://nodejs.org/zh-cn
