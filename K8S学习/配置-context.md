## 基本
在 Kubernetes（K8s）中，context（上下文）是集群访问配置的集合，用于简化多集群、多用户环境下的操作。它整合了访问集群所需的关键信息，让用户无需每次次每次次指定集群地址、认证方式等参数。

## context 的核心概念
context 本质是一个 “配置快照”，包含以下 3 个核心要素：
1. 集群（cluster）：集群的 API Server 地址、CA 证书等信息（用于定位和验证集群）。
2. 用户（user）：访问集群的认证信息（如密钥、令牌、证书等，用于证明身份）。
3. 命名空间（namespace）：默认操作的命名空间（可选，简化无需每次指定 -n 参数）。

这些信息存储在本地的 kubeconfig 文件（默认路径 ~/.kube/config）中，context 相当于为 “集群 + 用户 + 命名空间” 组合起一个别名，方便快速切换。

## context 的常见操作（命令行）
```bash
# 查看所有的context
kubectl config get-contexts

# 切换到指定 context（例如切换到生产集群）
kubectl config use-context prod-cluster

# 创建context
# 语法：kubectl config set-context <context名称> --cluster=<集群名> --user=<用户名> --namespace=<命名空间>
kubectl config set-context dev-context \
  --cluster=dev-cluster \
  --user=dev-user \
  --namespace=dev

# 修改context
# 修改 context 的默认命名空间（例如将 prod-cluster 的默认命名空间改为 app1）
kubectl config set-context prod-cluster --namespace=app1

# 删除 context
kubectl config delete-context old-context
```