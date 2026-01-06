---
title: GitLab backup and update
description: >-
  GitLab 备份恢复完整指南及老版本升级实操（基于 Docker CE 版本）
categories: [云平台/基础设施（Cloud & Infra）, gitlab]
tags: [gitlab, docker]
---

# GitLab backup and update

GitLab 作为 DevOps 工作流的核心工具，通过可靠备份保障数据安全、通过平滑升级获取新功能，是业务连续性的关键。本文详细介绍 GitLab 备份、恢复的完整流程，以及基于 Docker CE 版本的分步升级方案，整合官方最佳实践与实操经验，适用于各类企业级部署场景。

## 一、备份与恢复操作

### 1.1 核心概念说明
- **配置备份**：包含核心配置文件（`gitlab.rb`）和敏感信息（如双因子认证密钥 `gitlab-secrets.json`），是恢复环境一致性的关键。**注意：GitLab 默认备份机制不包含这两个文件，需手动备份**，否则会导致恢复后服务异常。
- **数据备份**：涵盖项目代码、数据库、用户信息、提交记录等业务核心数据。
- **默认路径**（Docker 部署）：
  - 配置文件目录：`/etc/gitlab/`（需映射至宿主机确保持久化）
  - 默认备份存储目录：`/var/opt/gitlab/backups/`

### 1.2 手动备份流程

#### 步骤 1：备份配置文件
使用 `gitlab-ctl backup-etc` 命令备份关键配置，目录不存在时会自动创建：
```bash
# 进入 GitLab 容器（Docker 环境必执行）
docker exec -it <gitlab容器名称> bash

# 备份配置文件（默认存储至 /etc/gitlab/config_backup）
gitlab-ctl backup-etc

# 可选：指定自定义备份路径
gitlab-ctl backup-etc -p /自定义/备份/路径
```

**备份包含文件**：
- `gitlab.rb`：GitLab 主配置文件（如端口、存储路径、集成服务配置）
- `gitlab-secrets.json`：双因子认证、数据库加密密钥（db_key_base）、第三方集成等敏感信息存储文件（核心文件，丢失会导致 500 错误）
- `trusted-certs/`：可信 SSL 证书目录

#### 步骤 2：备份业务数据
GitLab 12.2 及以上版本使用 `gitlab-backup create` 命令（12.2 以下版本需用 `gitlab-rake gitlab:backup:create`）：
```bash
# 进入 GitLab 容器
docker exec -it <gitlab容器名称> bash

# 执行数据备份
gitlab-backup create
```

**备份配置自定义**（修改 `gitlab.rb` 文件）：
```ruby
# 自定义备份存储路径（建议映射至宿主机大容量磁盘）
gitlab_rails['backup_path'] = "/var/opt/gitlab/backups"

# 备份文件权限设置（默认 0644，仅 git 用户可读写）
gitlab_rails['backup_archive_permissions'] = 0644

# 备份保留时长（默认 7 天 = 604800 秒，过期自动删除）
gitlab_rails['backup_keep_time'] = 604800
```

修改配置后需执行以下命令使生效：
```bash
gitlab-ctl reconfigure
```

### 1.3 数据恢复流程

#### 前置条件
- 备份文件（如 `1676332446_2023_02_13_15.6.7_gitlab_backup.tar`）已放置在备份目录（`/var/opt/gitlab/backups`）
- 恢复前需停止核心服务，避免数据写入冲突
- 建议提前拷贝旧服务器的 `gitlab-secrets.json` 至新服务器（避免恢复后 500 错误）

#### 步骤 1：停止相关服务
```bash
# 进入 GitLab 容器
docker exec -it <gitlab容器名称> bash

# 停止 puma（Web 服务）和 sidekiq（后台任务处理器）
gitlab-ctl stop puma
gitlab-ctl stop sidekiq

# 验证服务状态（确保目标服务已停止）
gitlab-ctl status
```

#### 步骤 2：执行恢复操作
使用备份文件的**时间戳前缀**（无需完整文件名）执行恢复：
```bash
# 格式：gitlab-backup restore BACKUP=<备份文件时间戳前缀>
gitlab-backup restore BACKUP=1676332446_2023_02_13_15.6.7
```
- 执行过程中需两次输入 `yes` 确认恢复（避免误操作）
- 12.2 以下版本需用：`gitlab-rake gitlab:backup:restore BACKUP=<时间戳前缀>`

#### 步骤 3：重新配置并重启服务
```bash
# 重新加载配置（确保恢复后的配置生效）
gitlab-ctl reconfigure

# 重启所有 GitLab 服务
gitlab-ctl restart
```

#### 步骤 4：恢复验证
1. 访问 GitLab Web 界面，确认项目、用户、权限配置是否完整恢复
2. 克隆测试项目或提交代码，验证数据读写正常
3. 检查第三方集成（如 CI/CD 流水线、WebHook）是否正常工作

### 1.4 恢复后常见问题：500 错误处理
#### 错误现象
备份迁移到新服务器并恢复后，部分页面（如项目详情、CI/CD 配置页）访问报 500 内部服务器错误，恢复前新服务器页面访问正常。

#### 根本原因
GitLab 默认备份不包含 `gitlab-secrets.json` 文件，该文件存储了数据库加密密钥（db_key_base）、CI 运行器令牌等敏感信息。恢复后新服务器的 `gitlab-secrets.json` 为默认生成的新文件，与备份数据中的加密信息不匹配，导致解密失败触发 500 错误。

#### 解决方法

##### 方法一：直接替换 `gitlab-secrets.json`（推荐，旧服务器配置未删除时）
1. 从旧 GitLab 服务器拷贝核心配置文件：
   ```bash
   # 从旧服务器下载 gitlab-secrets.json 到本地
   scp root@旧服务器IP:/etc/gitlab/gitlab-secrets.json ./
   
   # 上传到新服务器的 GitLab 容器配置目录（Docker 环境）
   docker cp gitlab-secrets.json <新容器名称>:/etc/gitlab/
   ```

2. 重新加载配置并重启服务：
   ```bash
   # 进入新服务器 GitLab 容器
   docker exec -it <新容器名称> bash
   
   # 重新配置并重启
   gitlab-ctl reconfigure
   gitlab-ctl restart
   ```

3. 验证：访问之前报错的页面，确认错误已修复。

##### 方法二：重置 CI 密钥与令牌（旧服务器配置已删除时）
若旧服务器已下线或 `gitlab-secrets.json` 丢失，可通过重置 CI 相关密钥和令牌解决（会导致现有 CI 运行器失效，需重新注册）：

1. 进入 Rails 控制台重置 CI 运行器令牌：
   ```bash
   # 进入 GitLab 容器
   docker exec -it <新容器名称> bash
   
   # 启动 Rails 控制台
   gitlab-rails console
   
   # 执行重置命令（清除所有 CI 运行器加密令牌）
   irb(main):001:0> Ci::Runner.all.update_all(token_encrypted: nil)
   ```
   - 执行后会返回受影响的记录数（如 `=> 3` 表示重置了 3 个运行器令牌），输入 `exit` 退出控制台。

2. 进入数据库重置项目和命名空间的令牌：
   ```bash
   # 进入 PostgreSQL 数据库控制台
   gitlab-rails dbconsole
   
   # 执行以下 3 条 SQL 命令（逐条执行）
   gitlabhq_production=> UPDATE projects SET runners_token = null, runners_token_encrypted = null;
   gitlabhq_production=> UPDATE namespaces SET runners_token = null, runners_token_encrypted = null;
   gitlabhq_production=> UPDATE application_settings SET runners_registration_token_encrypted = null;
   
   # 退出数据库控制台
   gitlabhq_production=> \q
   ```

3. 重启 GitLab 服务：
   ```bash
   gitlab-ctl restart
   ```

4. 后续操作：重新注册所有 CI 运行器（因令牌已重置，旧运行器无法连接）。

## 二、升级指南（Docker CE 版本）

### 2.1 官方支持的升级路径
GitLab 跨多个主版本直接升级可能导致数据损坏或配置冲突，需遵循**增量升级路径**。以下是从 17.0.2 CE 升级至 18.7.0 CE 的官方推荐步骤：

[Upgrade Path](https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/?current=17.0.2&distro=docker&edition=ce)

| 升级阶段 | 目标版本          | Docker 启动命令                                                                 |
|----------|-------------------|--------------------------------------------------------------------------------|
| 1        | 17.1.8-ce.0       | `docker run --name gitlab-ce -d --restart always -p 80:80 -p 443:443 gitlab/gitlab-ce:17.1.8-ce.0` |
| 2        | 17.3.7-ce.0       | `docker run --name gitlab-ce -d --restart always -p 80:80 -p 443:443 gitlab/gitlab-ce:17.3.7-ce.0` |
| 3        | 17.5.5-ce.0       | `docker run --name gitlab-ce -d --restart always -p 80:80 -p 443:443 gitlab/gitlab-ce:17.5.5-ce.0` |
| 4        | 17.8.7-ce.0       | `docker run --name gitlab-ce -d --restart always -p 80:80 -p 443:443 gitlab/gitlab-ce:17.8.7-ce.0` |
| 5        | 17.11.7-ce.0      | `docker run --name gitlab-ce -d --restart always -p 80:80 -p 443:443 gitlab/gitlab-ce:17.11.7-ce.0` |
| 6        | 18.2.8-ce.0       | `docker run --name gitlab-ce -d --restart always -p 80:80 -p 443:443 gitlab/gitlab-ce:18.2.8-ce.0` |
| 7        | 18.5.4-ce.0       | `docker run --name gitlab-ce -d --restart always -p 80:80 -p 443:443 gitlab/gitlab-ce:18.5.4-ce.0` |
| 8        | 18.7.0-ce.0       | `docker run --name gitlab-ce -d --restart always -p 80:80 -p 443:443 gitlab/gitlab-ce:18.7.0-ce.0` |

### 2.2 升级前置准备
1. **全量备份**：升级前必须执行「一、备份与恢复操作」中的完整备份，确保可回滚
2. **磁盘空间检查**：宿主机需预留至少 2 倍 GitLab 数据量的空闲空间（用于存储升级过程中的临时文件和数据库迁移数据）
3. **停止旧容器**：
   ```bash
   # 停止并删除当前运行的 GitLab 容器
   docker stop <旧容器名称>
   docker rm <旧容器名称>
   ```
4. **数据卷持久化**：启动新容器时，需挂载与旧容器相同的宿主机目录（如 `/var/opt/gitlab`、`/etc/gitlab`），确保数据不丢失

### 2.3 升级后验证
1. **服务状态检查**：
   ```bash
   docker exec -it <新容器名称> gitlab-ctl status
   ```
2. **后台迁移监控**：
   - 从 17.0.2 升级至 18.7.0 会触发 368 个后台数据库迁移，大表迁移可能耗时较长
   - 查看迁移进度命令：
     ```bash
     docker exec -it <新容器名称> gitlab-rake gitlab:background_migrations:status
     ```
3. **功能完整性验证**：
   - Web 界面访问：确认登录、项目浏览、代码提交等基础功能正常
   - 核心功能测试：CI/CD 流水线执行、备份功能、第三方集成同步
   - 日志检查：查看是否有报错信息 `docker logs <新容器名称>`

## 三、关键注意事项

### 3.1 备份与恢复风险提示
- **配置文件不可缺失**：`gitlab-secrets.json` 丢失会导致双因子认证失效、集成服务鉴权失败、数据恢复后 500 错误，需单独加密存储备份
- **权限控制**：备份文件需设置 `0600` 权限（仅 root 用户可读写），避免敏感信息泄露
- **版本兼容性**：备份文件仅支持恢复至相同或更高版本的 GitLab，不可跨版本回退恢复
- **迁移必做**：跨服务器迁移时，务必同步 `gitlab.rb` 和 `gitlab-secrets.json` 两个核心配置文件，否则会触发服务异常

### 3.2 升级风险与规避方案
| 风险场景                | 描述                                                                 | 规避方案                                                                 |
|-------------------------|----------------------------------------------------------------------|--------------------------------------------------------------------------|
| PostgreSQL 索引损坏     | 升级 OS 时，glibc 2.28+ 的 locale 数据变更可能导致 PostgreSQL 索引损坏 | 升级 OS 前参考 [PostgreSQL 操作系统升级指南](https://docs.gitlab.com/ee/update/postgresql.html#upgrading-operating-systems) |
| OpenSSL v3 兼容性问题   | GitLab 17.7+ 将 OpenSSL 从 1.1.1w 升级至 3.0.0，可能影响第三方集成   | 升级前通过 [OpenSSL 3 兼容性指南](https://docs.gitlab.com/ee/update/openssl_3.html) 评估集成兼容性 |
| 多节点密钥同步问题      | GitLab 17.8+ 新增 3 个加密密钥至 `gitlab-secrets.json`               | 多节点部署时，需确保所有节点的 `gitlab-secrets.json` 文件完全一致       |

## 四、总结
- **备份策略**：建议配置定时任务（如 crontab）每周执行全量备份，同时单独备份 `gitlab-secrets.json` 至安全存储（如加密服务器、云存储）
- **恢复原则**：恢复前务必停止核心服务，跨服务器迁移时同步核心配置文件；恢复后优先验证易报错页面（如 CI/CD 配置、项目详情），避免因密钥不匹配导致业务中断
- **升级核心**：严格遵循官方增量升级路径，升级前备份、升级中监控、升级后验证，三步缺一不可
- **问题排查**：恢复后出现 500 错误，优先检查 `gitlab-secrets.json` 是否与旧服务器一致，再考虑重置 CI 密钥

通过规范的备份与升级操作，可确保 GitLab 服务的稳定性和数据安全性，为 DevOps 工作流提供可靠支撑。

## 参考文献
1. [GitLab 数据备份和恢复实战][尹正杰博客]
2. [GitLab 官方升级路径工具][gitlab-upgrade-path]
3. [GitLab 官方备份与恢复文档][gitlab-backup-docs]
4. [OpenSSL 3 兼容性指南][gitlab-openssl3]
5. [PostgreSQL 操作系统升级指南][gitlab-postgresql-upgrade]
6. [GitLab 备份迁移后 500 错误解决方案][csdn-gitlab-500]

[尹正杰博客]: https://www.cnblogs.com/yinzhengjie/p/18579240
[gitlab-upgrade-path]: https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/?current=17.0.2&distro=docker&edition=ce
[gitlab-backup-docs]: https://docs.gitlab.com/ee/raketasks/backup_restore.html
[gitlab-openssl3]: https://docs.gitlab.com/ee/update/openssl_3.html
[gitlab-postgresql-upgrade]: https://docs.gitlab.com/ee/update/postgresql.html#upgrading-operating-systems
[csdn-gitlab-500]: https://blog.csdn.net/weixin_44037416/article/details/108256673