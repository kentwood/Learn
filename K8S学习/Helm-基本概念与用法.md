## 概念
Helm 是 Kubernetes 的包管理工具，类似于 Linux 中的 apt 或 yum，用于简化 Kubernetes 应用的部署、升级、回滚等操作。它通过 “打包” Kubernetes 资源（Deployment、Service 等），解决了复杂应用的管理难题。

### 1. Chart（图表）
- 是 Helm 的 “包”，包含了一个 Kubernetes 应用的所有资源定义（YAML 文件）、配置模板和说明文档。
- 结构类似代码仓库，可版本化管理，支持共享和复用。
### 2. Repository（仓库）
- 存储和分发 Chart 的地方（类似 Docker Hub），可以是公开的（如官方 Helm Hub）或私有仓库。
- 常用公开仓库：https://charts.bitnami.com/bitnami（包含 MySQL、Nginx 等常用应用）。
### 3. Release（发布）
- Chart 部署到 Kubernetes 集群后的实例。同一个 Chart 可多次部署，每次部署生成一个独立的 Release（如 mysql-prod、mysql-test）。
### 4. Values（配置值）
- 用于动态配置 Chart 中的参数（如镜像版本、副本数、端口等），避免直接修改 Chart 内部的 YAML 文件，实现 “一次打包，多次配置”。
### 5. Templates（模板）
- Chart 中使用 Go 模板语法编写的 Kubernetes 资源文件，通过 Values 注入参数，生成最终的 YAML 配置。

## 使用
### 1. 仓库管理
```text
# 添加公开仓库（以 Bitnami 为例）
helm repo add bitnami https://charts.bitnami.com/bitnami

# 更新仓库索引（获取最新 Chart 列表）
helm repo update

# 查看已添加的仓库
helm repo list

# 删除仓库
helm repo remove bitnami
```

### 2. 搜索chart
```text
# 搜索仓库中的 Chart（如 MySQL）
helm search repo mysql

# 搜索所有公开仓库（需联网）
helm search hub nginx
```

### 3. 安装 Release（部署应用）
```text
# 从仓库安装 Chart，指定 Release 名称（如 mysql-dev）
helm install mysql-dev bitnami/mysql

# 自定义配置（通过 --set 覆盖默认 Values）
helm install mysql-prod bitnami/mysql \
  --set auth.rootPassword=StrongPass123 \  # 设置 root 密码
  --set replicaCount=2 \                  # 副本数
  --namespace mysql-prod \                # 指定命名空间（自动创建）
  --create-namespace

# 通过自定义 Values 文件安装（适合复杂配置）
helm install mysql-test bitnami/mysql -f my-values.yaml
```

### 4. 查看Release
```text
# 查看所有 Release（指定命名空间，默认 default）
helm list -n mysql-prod

# 查看 Release 详情
helm show all mysql-prod

# 查看 Release 生成的 Kubernetes 资源
helm get manifest mysql-prod
```

### 5. 升级与回滚
```text
# 升级 Release（修改配置，如增加副本数）
helm upgrade mysql-prod bitnami/mysql \
  --set replicaCount=3 \
  --reuse-values  # 保留其他未修改的配置

# 回滚到上一版本（--revision 指定版本号）
helm rollback mysql-prod  # 回滚到上一版
helm rollback mysql-prod 2  # 回滚到版本 2

# 查看 Release 历史版本
helm history mysql-prod
```

### 6. 卸载 Release
```text
# 卸载 Release（默认保留历史记录，可回滚）
helm uninstall mysql-dev -n default

# 彻底删除（不保留历史）
helm uninstall mysql-dev --no-hooks
```

## 自定义Chart示例
1. 创建 Chart 骨架
```text
helm create myweb  # 生成默认 Chart 结构
```
生成的目录结构如下
```text
myweb/
├── Chart.yaml        # Chart 元数据（名称、版本等）
├── values.yaml       # 默认配置值
├── charts/           # 子 Chart（依赖）
├── templates/        # 资源模板
│   ├── deployment.yaml  # Deployment 模板
│   ├── service.yaml     # Service 模板
│   ├── ingress.yaml     # Ingress 模板（可选）
│   └── _helpers.tpl     # 模板函数（可选）
└── README.md         # 说明文档
```

2. 修改模板和配置
```yaml
# Chart.yaml 元数据
apiVersion: v2
name: myweb
description: A simple web application
version: 0.1.0  # Chart 版本
appVersion: "1.0"  # 应用版本
```
```yaml
# values.yaml 配置数据
replicaCount: 1
image:
  repository: nginx
  tag: alpine
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
```
```yaml
# templates/deployment.yaml（Deployment 模板）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-myweb  # 使用 Release 名称作为前缀
spec:
  replicas: {{ .Values.replicaCount }}  # 引用 values 中的副本数
  selector:
    matchLabels:
      app: {{ .Release.Name }}-myweb
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-myweb
    spec:
      containers:
      - name: web
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"  # 镜像地址
        ports:
        - containerPort: 80
```

3. 打包自定义Chart
```text
# 打包 Chart（生成 .tgz 文件）
helm package myweb  # 生成 myweb-0.1.0.tgz

# 安装自定义 Chart
helm install myweb-demo ./myweb-0.1.0.tgz \
  --set replicaCount=2 \  # 覆盖默认副本数
  --set image.tag=latest  # 使用最新镜像
```