---
title: Jekyll GitHub Pages 自动部署博客指南(北)
description: >-
  部署 Jekyll-theme-chirpy 遇到的一些问题
date: 2025-08-19 14:57:00 +0800
categories: [博客, 教程]
tags: [Jekyll,GitHub,Ruby,Gem,Bundler]
---

## 准备工作

首先介绍有两种方式部署Jekyll-theme-chirpy：

1. **使用 Chirpy Starter**：这种方法简化了升级，隔离了不必要的文件，非常适合那些希望专注于以最小配置进行写入的用户。(我没有使用这种方法，不做介绍)

- Sign in to GitHub and navigate to the [**starter**][starter].
- Click the <kbd>Use this template</kbd> button and then select <kbd>Create a new repository</kbd>.
- Name the new repository `<username>.github.io`, replacing `username` with your lowercase GitHub username.

2. **Forking on GitHub**：这种方法便于修改功能或UI设计，但在升级过程中会带来挑战。所以，除非你熟悉Jekyll并计划大幅修改这个主题，否则不要尝试这个。（就挑难的）

-  在GitHub 上 Fork [ **Chirpy**][chirpy] 代码并重命名为 `<GITHUB_USERNAME>.github.io` ,目的是为了使用GitHub Page的免费域名，如果你有别的域名当然可以使用。

- 在本地 clone 代码然后执行

  ```bash
  bash tools/init.sh
  ```

  >  如果不是在 GitHub Page 上部署，加上 `--no-gh`
  {: .prompt-tip }

  上述命令将会：

  1. 从存储库中删除 `_posts` 目录。
  2. 如果使用了 `--no-gh` 选项，则将删除 `.github` 目录。否则，通过删除 `.github/workflows/pages-deploy.yml.hook` 文件的 `.hook` 扩展名来设置 GitHub Action 工作流，然后删除 `.github` 文件夹中的其他文件和目录。
  3. 从 `.gitignore` 中删除 `Gemfile.lock` 条目 。
  4. 创建新提交以自动保存更改。

- 下面介绍[本地部署](###本地部署)和 [GitHub Actions 自动部署](###GitHub Actions 自动部署)

### 本地部署

本地部署 jekyll 方便在本地调试发布博客，不用每次都使用 GitHub 部署发布。

#### 安装依赖项

- [Ruby][ruby]，[Gem][gem]：安装包直接下机安装即可，里面的命令行输入选3

  ```bash
   # 检查是否安装好
   ruby -v
   gem -v
  ```

- Bundler,Jekyll：

  ```bash
  gem install jekyll bundler
  # 检查是否安装好
  jekyll -v
  bundler -v
  ```

#### 构建并预览 Jekyll 网站

在克隆的代码根目录执行

```bash
bundler install 
```

安装依赖，外网太慢可以换源

```bash
# 添加镜像源并移除默认源
gem sources --add https://mirrors.tuna.tsinghua.edu.cn/rubygems/ --remove https://rubygems.org/
# 列出已有源
gem sources -l
# 替换 bundler 默认源
bundle config mirror.https://rubygems.org https://mirrors.tuna.tsinghua.edu.cn/rubygems
```

依赖装好，可以只构建网站

```bash
bundle exec jekyll serve
```

浏览器中访问 `http://127.0.0.1:4000`

### GitHub Actions 自动部署

首先你需要注意项目中的 `_config.yml` 文件，这是必要的配置文件。

```yaml
lang: en # 语言使用中文 zh-CN
timezone: Asia/Shanghai # 时区
url: "" #'https://username.github.io'，注意不要以'/'结尾
```

如果你的本地电脑不是 Linux, 在项目根目录执行

```bash
bundle lock --add-platform x86_64-linux
```

然后在 GitHub 上的仓库中选择 Setting， 在左边导航栏找到 pages , 右边 Build and deployment 的 Source 下拉选择 GitHub Action, 每次 push 你的代码，GitHub 会自动帮你构建部署，等待完成后访问`https://username.github.io`

### 升级

这取决于您如何使用主题：

- 如果您使用的是 gem 主题（在 `Gemfile` 中会有 `gem "jekyll-theme-chirpy"` ），则编辑 `Gemfile` 并更新 gem 主题的版本号，例如：

  ```
  - gem "jekyll-theme-chirpy", "~> 3.2", ">= 3.2.1"
  + gem "jekyll-theme-chirpy", "~> 3.3", ">= 3.3.0"
  ```

  然后执行以下命令：

  ```
  $ bundle update jekyll-theme-chirpy
  ```

  随着版本升级，关键文件（有关详细信息，请参阅 [Startup Template](https://github.com/cotes2020/chirpy-starter) ）和配置选项将会被更改。请参阅 [Upgrade Guide](https://github.com/cotes2020/jekyll-theme-chirpy/wiki/Upgrade-Guide) ，以使您的存储库的文件与最新版本的主题保持同步。

- 如果您是从源项目 fork 的（您的 `Gemfile` 中会有 `gemspec` ），则将 [最新的上游 tag](https://github.com/cotes2020/jekyll-theme-chirpy/tags) 合并到您的 Jekyll 站点中以完成升级。该合并可能会与您的本地修改冲突。请耐心且仔细地解决这些冲突。

### 添加评论功能

#### 引入 giscus

1. 在 GitHub App 中安装 giscus, 然后给指定仓库安装 giscus

2. 给仓库开启 Discussions 在仓库的 Settings 中设置

3. 来到 [giscus][giscus] 官网获取配置

4. 把配置导入 _config.yml

   ```yaml
   comments:
     provider: giscus # [disqus | utterances | giscus]
     giscus:
       repo:                                     # GitHub 仓库，格式：<owner>/<repo>
       repo_id:                                  # Giscus 分配的 Repo ID
       category: Announcements                   # 评论分类名称
       category_id:                              # Giscus 分配的 Category ID
       mapping: pathname                         # 映射方式：pathname（默认）
       strict: 1                                 # 严格匹配路径
       input_position: bottom                    # 评论框位置：底部
       lang: zh-CN                               # 语言：简体中文
       reactions_enabled: 1                      # 启用 reactions（点赞等）
       emit_metadata: 0                          # 不发送页面元数据
       theme: preferred_color_scheme             # 使用系统主题或跟随网站
       loading: lazy                             # 延迟加载
   ```

### 添加访客数量

直接在文章需要的位置添加 html 代码块即可

```html
<p>本站总访问量：<span id="busuanzi_site_pv" style="color:#1cc088;">加载中...</span> 次 • 本站总访客数：<span id="busuanzi_site_uv" style="color:#1cc088;">加载中...</span> 人</p>
```

#### 安装与使用
在HTML中引入JavaScript脚本：

```js
<script src="//api.busuanzi.cc/static/3.6.9/busuanzi.min.js" defer></script>
```

也可以下载JavaScript文件部署到自己的网站：[点击下载](https://api.busuanzi.cc/static/3.6.9/download/Busuanzi.zip)

**这个前端js放在本地，很多html文件引用这个js，地址怎么配置(因为菜，所有用http)**

添加不蒜子统计标签：

```html
本站总访问量：<span id="busuanzi_site_pv">加载中...</span>
本站总访客数：<span id="busuanzi_site_uv">加载中...</span>
本页总阅读量：<span id="busuanzi_page_pv">加载中...</span>
本页总访客数：<span id="busuanzi_page_uv">加载中...</span>
本站今日访问量：<span id="busuanzi_today_site_pv">加载中...</span>
本站今日访客数：<span id="busuanzi_today_site_uv">加载中...</span>
本页今日阅读量：<span id="busuanzi_today_page_pv">加载中...</span>
本页今日访客数：<span id="busuanzi_today_page_uv">加载中...</span>
```



## References

1. [准备开始][]
1. [Chirpy本地部署][]
1. [starter][]
1. [chirpy][]
1. [ruby][]
1. [gem][]
1. [giscus][]
1. [busuanzi][]

[准备开始]: https://pansong291.github.io/chirpy-demo-zhCN/posts/getting-started/#%E5%8D%87%E7%BA%A7
[Chirpy本地部署]: https://august295.github.io/posts/Chirpy%E6%9C%AC%E5%9C%B0%E9%83%A8%E7%BD%B2/#11-windows
[starter]: https://github.com/cotes2020/chirpy-starter
[chirpy]: https://github.com/cotes2020/jekyll-theme-chirpy/fork
[ruby]: https://rubyinstaller.org/downloads/
[gem]: https://rubygems.org/pages/download
[giscus]: https://giscus.app/zh-CN
[busuanzi]: https://www.busuanzi.cc/