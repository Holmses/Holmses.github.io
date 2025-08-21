---
title: Prometheus 和 Grafana 监控 Kubernetes 集群
description: >-
  kubernetes node 和 pod 的性能监控
categories: [安全&监控(Security & Monitoring), prometheus]
tags: [prometheus, grafana, kubernetes]
image:
  path: https://raw.githubusercontent.com/Holmses/Holmses.github.io/master/assets/img/image-20250821134737333.png
---

# Prometheus 监控 kubernetes 集群性能

## Prometheus 部署获取节点数据

### **创建命名空间 Namespace**

```bash
kubectl create namespace monitoring
```

### **部署 Node Exporter**

node-exporter-deamonset.yaml

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.6.1
        ports:
        - containerPort: 9100
          name: metrics
      hostNetwork: true # 让 Node Exporter 使用宿主机的网络命名空间，可以直接通过节点 IP 访问。
      hostPID: true # 让 Node Exporter 可以访问宿主机的进程信息。
      tolerations: # 允许在控制平面节点（如 master 节点）上运行。
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
```

由于 node-exporter 是通过 DaemonSet 部署的，每个节点都会运行一个 node-exporter Pod，因此我们需要创建一个 Headless Service（无头服务），以便 Prometheus 可以直接访问每个节点的 node-exporter。

node-exporter-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  clusterIP: None  # Headless Service
  selector:
    app: node-exporter
  ports:
  - port: 9100
    targetPort: 9100
    name: metrics
```

部署 node-exporter

```bash
kubectl apply -f node-exporter-daemonset.yaml
kubectl apply -f node-exporter-service.yaml
```

验证 node exporter

```bash
curl http://<节点IP>:9100/metrics
```

---

### **部署** kube-state-metrics

kube-state-metrics-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: monitoring
  labels:
    app: kube-state-metrics
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-state-metrics
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      containers:
      - name: kube-state-metrics
        image: k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.8.0
        ports:
        - containerPort: 8080
          name: metrics
---
apiVersion: v1
kind: Service
metadata:
  name: kube-state-metrics
  namespace: monitoring
  labels:
    app: kube-state-metrics
spec:
  selector:
    app: kube-state-metrics
  ports:
  - port: 8080
    targetPort: metrics
    name: metrics
```

部署 kube-state-metrics-deployment.yaml

```bash
kubectl apply -f kube-state-metrics-deployment.yaml
```

验证 kube-state-metrics

```bash
# 通过 Service 的 ClusterIP 和端口 8080 访问：
kubectl port-forward svc/kube-state-metrics 8080:8080 -n monitoring
# 然后在浏览器或终端访问：
http://localhost:8080/metrics
# 或者
curl http://localhost:8080/metrics
```

---

### **部署Prometheus**

prometheus-configmap.yaml

可以配置成Kubernetes_sd_configs形式写在 kubernetes_sd_configs 的用法中

```yaml
# 静态地址简单配置版本
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s

    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['localhost:9090']

      - job_name: 'node-exporter'
        static_configs:
          - targets: ['node-exporter.monitoring.svc.cluster.local:9100']

      - job_name: 'kube-state-metrics'
        static_configs:
          - targets: ['kube-state-metrics.monitoring.svc.cluster.local:8080']
```

也可以使用Kubernetes_sd_configs动态配置prometheus_configs

先创建 ClusterRole 授权访问 Endpoints

prometheus-cluster-role.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/proxy
      - services
      - endpoints
      - pods
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources:
      - configmaps
    verbs: ["get"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
```

应用配置

```bash
kubectl apply -f prometheus-cluster-role.yaml
```

创建 ClusterRoleBingding 绑定 ServiceAccount

prometheus-cluster-role-binding.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: default  # Prometheus 默认使用的 ServiceAccount
  namespace: monitoring  # Prometheus 部署的 Namespace
```

应用配置

```bash
kubectl apply -f prometheus-cluster-role-binding.yaml
```

prometheus-config.yaml (Kubernetes_sd_configs 动态的配置参考)

```yaml
global:
  scrape_interval: 15s      # 全局抓取间隔
  scrape_timeout: 10s       # 全局抓取超时时间
  evaluation_interval: 1m   # 告警规则评估间隔

scrape_configs:
  # 1. 抓取 Kubernetes 节点指标（替换默认10250端口为9100）
  - job_name: 'kubernetes-node'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__
        action: replace

  # 2. 抓取 Kubernetes API Server 指标
  - job_name: 'kubernetes-apiserver'
    kubernetes_sd_configs:
      - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      insecure_skip_verify: true  # 生产环境建议配置有效证书
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https

  # 3. 抓取 Kubernetes Pod 暴露的指标（支持自定义注解）
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: endpoints
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
  
  - job_name: 'kube-state-metrics'
    kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names: ['monitoring']
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_label_app]
        action: keep
        regex: kube-state-metrics
      - source_labels: [__meta_kubernetes_endpoint_port_name]
        action: keep
        regex: metrics
```

重启 Prometheus

prometheus-deployment.yaml

```yaml
# 最好使用NFS做持久化
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: monitoring-prometheus-data-pvc
  namespace: monitoring
  annotations:
    nfs.io/storage-path: "prometheus"
spec:
  storageClassName: app-nfs-storage
  accessModes: 
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:v2.47.0 
        ports:
        - containerPort: 9090
          name: web
        volumeMounts:
        - name: config-volume
          mountPath: /etc/prometheus/
        - name: prometheus-data
          mountPath: /prometheus
      volumes:
      - name: config-volume
        configMap:
          name: prometheus-config
      - name: prometheus-data
        persistentVolumeClaim:
          claimName: monitoring-prometheus-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitoring
  labels:
    app: prometheus
spec:
  type: NodePort
  selector:
    app: prometheus
  ports:
  - port: 9090
    targetPort: web
    nodePort: 30090
```

部署 Prometheus

```bash
kubectl apply -f prometheus-configmap.yaml
kubectl apply -f prometheus-deployment.yaml
```

访问 Prometheus Web UI

```bash
http://<任意节点IP>:30090
```

**验证配置是否生效**

访问 <http://prometheus-ip:9090/targets>，确认：

node-exporter job 显示所有节点的 node-exporter 实例为 "UP"。
kubernetes-service-endpoints job 是否正确抓取其他 Exporter（如 kube-state-metrics）。

---

## 可视化展示数据

### 部署 Grafana

grafana-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:10.2.3
        ports:
        - containerPort: 3000
          name: web
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app: grafana
spec:
  type: NodePort
  selector:
    app: grafana
  ports:
  - port: 3000
    targetPort: web
    nodePort: 30030
```

部署 Grafana

```bash
kubectl apply -f grafana-deployment.yaml
```

访问 Grafana Web UI

```bash
http://<任意节点IP>:30030
```

默认用户名和密码：

用户名：admin
密码：admin（首次登录后需要修改）

配置 Prometheus 数据源

在 Grafana 中添加 Prometheus 作为数据源：

URL：http://prometheus.monitoring.svc.cluster.local:9090（或通过 NodePort 访问的地址）
点击 "Save & Test" 验证连接。

**导入 Dashboard**

Grafana 官方提供了许多现成的 Dashboard，例如：

Kubernetes 集群监控 Dashboard ID：315（不好用）

Dashboard ID 1860（官方 Node Exporter Full Dashboard）（不错）

---

## Prometeus 与 Kubernetes 的联动

### kubernetes_sd_configs 的用法

**优点**

1. 动态发现：无需手动维护 static_configs，自动适应集群变化（如新增/删除 Pod）。
2. 减少配置：通过注解（如 prometheus.io/scrape）灵活控制是否抓取目标。
3. 支持多资源类型：可同时发现 Pod、Service、Endpoints、Node、Ingress 等资源。

| 参数              | 说明                                                         |
| ----------------- | ------------------------------------------------------------ |
| `role`            | 指定从哪种 Kubernetes 资源发现目标，支持：<br>• `pod`（从 Pod 发现）<br>• `service`（从 Service 的 Endpoints 发现）<br>• `endpoints`（直接从 Endpoints 发现）<br>• `node`（从 Node 发现）<br>• `ingress`（从 Ingress 发现） |
| `namespaces`      | 可选，指定只发现哪些 Namespace 的资源（默认发现所有 Namespace）。 |
| `relabel_configs` | 用于过滤和重写抓取目标的配置（如只抓取带有特定注解的 Pod）。 |

**常见role**

role:pod

- prometheus.io/scrape: "true"（是否抓取）
- prometheus.io/port: "8080"（抓取端口）
- prometheus.io/path: "/metrics"（指标路径）
- prometheus.io/scheme: "http"（协议，默认 http）

**配置示例**

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod  # 从 Pod 发现目标
    relabel_configs:
      # 过滤只抓取带有特定注解的 Pod
      - action: keep
        source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        regex: "true"
      # 从 Pod 注解中获取指标路径
      - action: replace
        source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        target_label: __metrics_path__
      # 从 Pod 注解中获取抓取协议（http/https）
      - action: replace
        source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scheme]
        target_label: __scheme__
      # 从 Pod 注解中获取抓取端口
      - action: replace
        source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        target_label: __address__
```

role:node

**配置示例**

```yaml
scrape_configs:
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - action: replace
        source_labels: [__meta_kubernetes_node_address_InternalIP]
        target_label: __address__
      - action: replace
        source_labels: [__meta_kubernetes_node_label_kubernetes_io_hostname]
        target_label: instance
```

role: ingress

**配置示例**

```yaml
scrape_configs:
  - job_name: 'kubernetes-ingresses'
    kubernetes_sd_configs:
      - role: ingress
    relabel_configs:
      - action: replace
        source_labels: [__meta_kubernetes_ingress_scheme, __address__, __meta_kubernetes_ingress_path]
        regex: (.+);(.+);(.+)
        replacement: ${1}://${2}${3}
        target_label: __metrics_path__
```
