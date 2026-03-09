---
title: Kubernetes 离线环境镜像导入导出
description: >-
  使用 crictl 和 ctr 在 containerd 运行时环境下导入导出镜像，附批量导出脚本
categories: [云平台/基础设施（Cloud & Infra）, kubernetes]
tags: [kubernetes, containerd, crictl, ctr, offline]
---

# Kubernetes 离线环境镜像导入导出

在无法访问公网的生产环境中部署或更新 Kubernetes 集群时，需要将镜像打包后传输到目标节点离线导入。Kubernetes 使用 containerd 作为容器运行时，镜像操作通过 `crictl` 或 `ctr` 工具完成。

## 工具说明

| 工具 | 用途 |
|------|------|
| `crictl` | CRI 标准接口工具，用于查询 Pod/容器/镜像状态 |
| `ctr` | containerd 原生 CLI，功能更丰富，支持镜像导入导出 |

> Kubernetes 的镜像存储在 containerd 的 `k8s.io` namespace 中，操作时需要加 `-n k8s.io` 参数。

---

## 查看镜像列表

```bash
# 使用 crictl 查看（推荐，输出更简洁）
crictl images

# 使用 ctr 查看
ctr -n k8s.io images ls
```

---

## 单个镜像导出

```bash
# 导出为 tar 文件
ctr -n k8s.io images export <输出文件名>.tar <镜像全名:tag> --platform linux/amd64

# 示例：导出 nginx 镜像
ctr -n k8s.io images export nginx.tar docker.io/library/nginx:latest --platform linux/amd64
```

---

## 单个镜像导入

```bash
ctr -n k8s.io images import <镜像文件>.tar

# 示例
ctr -n k8s.io images import nginx.tar
```

导入后验证：

```bash
crictl images | grep nginx
```

---

## 批量导出脚本

适合在有网环境的节点上将所有镜像打包，再传输到离线节点。

创建 `export_images.sh`：

```bash
#!/bin/bash

# 导出目录
EXPORT_PATH="/data/image_backup"
mkdir -p "$EXPORT_PATH"

echo "开始导出镜像到 $EXPORT_PATH ..."

# 遍历所有镜像
crictl images | awk 'NR>1 {print $1":"$2}' | while IFS=":" read -r image tag; do
    # 跳过 tag 为 <none> 的镜像
    if [ "$tag" = "<none>" ]; then
        continue
    fi

    # 用镜像名最后一段作为文件名（去掉仓库地址前缀）
    file_name=$(echo "$image" | awk -F'/' '{print $NF}')
    tar_file="${EXPORT_PATH}/${file_name}_${tag}.tar"

    echo "正在导出: ${image}:${tag} -> $tar_file"

    if ctr -n k8s.io images export "$tar_file" "${image}:${tag}"; then
        echo "  成功"
    else
        echo "  失败（镜像可能不存在于 k8s.io namespace）"
    fi
done

echo "导出完成，文件列表："
ls -lh "$EXPORT_PATH"
```

执行：

```bash
chmod +x export_images.sh
./export_images.sh
```

---

## 批量导入脚本

在目标节点上将打包好的镜像全部导入：

```bash
#!/bin/bash

IMPORT_PATH="/data/image_backup"

for tar_file in "$IMPORT_PATH"/*.tar; do
    echo "正在导入: $tar_file"
    if ctr -n k8s.io images import "$tar_file"; then
        echo "  成功"
    else
        echo "  失败: $tar_file"
    fi
done

echo "导入完成，当前镜像列表："
crictl images
```

---

## 跨节点传输

结合 `scp` 或 `rsync` 将导出的镜像传输到目标节点：

```bash
# 单个文件
scp /data/image_backup/nginx_latest.tar root@192.168.1.11:/data/image_backup/

# 整个目录（推荐，带压缩）
rsync -avz --progress /data/image_backup/ root@192.168.1.11:/data/image_backup/
```

---

## 常见问题

### 导出时提示 "image not found"

镜像可能存储在其他 namespace，尝试：

```bash
# 查看所有 namespace
ctr namespaces ls

# 在指定 namespace 中查找
ctr -n default images ls
```

### 导入后 crictl images 看不到镜像

确认导入时使用了 `-n k8s.io` 参数。没有指定 namespace 时，镜像会导入到 `default` namespace，而 kubelet 只从 `k8s.io` namespace 拉取镜像。
