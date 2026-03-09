---
title: EFK 日志收集系统部署指南
description: >-
  使用 Docker Compose 部署 Elasticsearch + Filebeat + Kibana 日志收集分析平台
categories: [云平台/基础设施（Cloud & Infra）, docker]
tags: [docker, elasticsearch, filebeat, kibana, logging]
---

# EFK 日志收集系统部署指南

EFK（Elasticsearch + Filebeat + Kibana）是 ELK 的轻量化变体，用 Filebeat 替代 Logstash 作为日志采集端，资源占用更小。适合在 Docker Swarm 环境中收集所有容器的日志。

## 架构说明

```
各节点容器日志
      ↓
Filebeat（DaemonSet / global 模式，采集 Docker 日志）
      ↓
Elasticsearch（存储和索引）
      ↓
Kibana（可视化查询）
```

- **Filebeat**：以 `global` 模式部署在每个 Docker 节点上，自动采集 `/var/lib/docker/containers` 下所有容器的日志
- **Elasticsearch**：接收、存储、索引日志数据
- **Kibana**：提供 Web 界面查询和可视化日志

---

## 部署配置

### 前置准备

创建 Docker overlay 网络：

```bash
docker network create elknet --driver overlay --subnet 172.45.0.0/16 --gateway 172.45.0.1
```

### docker-compose 配置

创建 `efk.yml`：

```yaml
version: '3.6'
services:
  elasticsearch:
    image: elasticsearch:7.5.0
    hostname: elasticsearch
    environment:
      - cluster.name=elasticsearch       # 集群名称
      - discovery.type=single-node       # 单节点模式
      - ES_JAVA_OPTS=-Xms1024m -Xmx1024m  # JVM 堆内存（建议为物理内存的一半，最大不超过 32G）
    volumes:
      - esdata1:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
      - 9300:9300
    networks:
      - elknet
    deploy:
      placement:
        constraints:
          - node.role == manager    # 部署在 Swarm manager 节点

  filebeat:
    image: elastic/filebeat:7.3.1
    user: root                          # 需要 root 权限读取 Docker socket
    environment:
      - output.elasticsearch.hosts=["elasticsearch:9200"]
      - -strict.perms=false
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro  # 挂载容器日志目录
      - /var/run/docker.sock:/var/run/docker.sock:ro              # Docker socket 用于获取容器元数据
    deploy:
      mode: global    # 每个节点都部署一个实例
    networks:
      - elknet

  kibana:
    image: kibana:7.5.0
    healthcheck:
      test: ["CMD-SHELL", "curl --fail http://elasticsearch:9200/ || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
    environment:
      - ELASTICSEARCH_URL=http://elasticsearch:9200
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - I18N_LOCALE=zh-CN    # 设置为中文界面
    ports:
      - "5601:5601"
    networks:
      - elknet
    deploy:
      placement:
        constraints:
          - node.role == manager

volumes:
  esdata1:
    driver: local

networks:
  elknet:
    external: true
```

### 部署

```bash
# Docker Swarm 模式部署
docker stack deploy -c efk.yml efk

# 查看服务状态
docker service ls | grep efk

# 查看 Filebeat 日志（确认日志采集正常）
docker service logs efk_filebeat -f
```

---

## 常见问题

### Elasticsearch 内存不足

Elasticsearch 对内存要求较高，如果节点内存不够，可以调低 JVM 堆：

```yaml
- ES_JAVA_OPTS=-Xms512m -Xmx512m
```

### Elasticsearch 启动失败（vm.max_map_count 不足）

```bash
# 临时修复
sudo sysctl -w vm.max_map_count=262144

# 永久修复
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Kibana 无法连接 Elasticsearch

健康检查会等待 ES 就绪后再启动 Kibana，如果长时间未就绪，检查 ES 容器日志：

```bash
docker service logs efk_elasticsearch -f
```

---

## 扩展为多节点 Elasticsearch 集群

如需生产级高可用，可将 Elasticsearch 扩展为多节点集群，在 `efk.yml` 中添加 `elasticsearch2`、`elasticsearch3` 服务节点，并配置：

```yaml
environment:
  - node.name=elasticsearch1
  - cluster.name=docker-cluster
  - discovery.seed_hosts=elasticsearch1,elasticsearch2,elasticsearch3
  - cluster.initial_master_nodes=elasticsearch1,elasticsearch2,elasticsearch3
```

---

## 访问

- **Kibana**：`http://<SERVER_IP>:5601`
- **Elasticsearch API**：`http://<SERVER_IP>:9200`

在 Kibana 中创建 Index Pattern（`filebeat-*`）即可开始查询日志。
