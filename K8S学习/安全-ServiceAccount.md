## 本质
在 Kubernetes（k8s）中，Service Account（服务账户） 是为集群内运行的 Pod / 容器 设计的身份凭证，用于解决 Pod 访问 Kubernetes API 服务器（或其他集群内服务）时的身份认证问题。它与面向人类用户的 User（如通过 kubeconfig 认证的管理员）完全不同，核心作用是为 “非人类实体”（Pod 中的应用程序）提供安全的身份标识和权限控制。
## Service Account 的核心概念与作用
### 1. 本质与定位
- 身份载体：每个 Service Account 会自动关联一个 Secret（类型为 kubernetes.io/service-account-token），该 Secret 包含三部分关键信息：
  - ca.crt：K8s API 服务器的 CA 证书（用于 Pod 验证 API 服务器身份）；
  - token：基于 JWT（JSON Web Token）的认证令牌（API 服务器通过该令牌识别 Service Account 身份）；
  - namespace：Service Account 所在的命名空间（默认与 Pod 同命名空间）。
- 命名空间隔离：Service Account 是 命名空间级资源，仅在其创建的命名空间内有效，无法跨命名空间直接使用。
- 默认行为：若 Pod 未显式指定 serviceAccountName，K8s 会自动为其挂载当前命名空间下的 default Service Account（每个命名空间默认创建此账户）。

### 2. 与 User 的核心区别
特性	|Service Account	|User（人类用户）
---|---|---
面向对象	|Pod / 容器内的应用程序（非人类实体）	|集群管理员、开发人员等人类用户
管理方式	|K8s API 直接创建（kubectl create sa）	|通常通过外部系统（如 LDAP、OIDC）管理
凭证载体	|自动关联的 Secret（挂载到 Pod）	|kubeconfig 文件（包含证书 / 令牌）
命名空间属性	|命名空间级资源（隔离）	|集群级资源（无命名空间）

## Service Account 的核心用法
Service Account 的使用流程可概括为：创建 Service Account → 为其分配权限（RBAC） → Pod 引用该账户。

### 1.基础操作：创建 Service Account
可以通过 kubectl 命令直接创建，或通过 YAML 定义（推荐生产环境，便于版本控制）。
```bash
## 命令行创建
# 在 default 命名空间创建名为 "app-sa" 的 Service Account
kubectl create serviceaccount app-sa

# 在指定命名空间（如 dev）创建
kubectl create serviceaccount app-sa -n dev
```
```yaml
## yaml定义
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa          # Service Account 名称
  namespace: dev        # 所属命名空间（默认 default）
  labels:
    app: my-application # 自定义标签，便于筛选
```

### 2. 关键步骤：为 Service Account 分配权限（RBAC）
Service Account 本身仅为 “身份”，无任何权限。需通过 RBAC（基于角色的访问控制） 为其绑定权限，核心是 Role（命名空间级权限）或 ClusterRole（集群级权限） + RoleBinding/ClusterRoleBinding（绑定账户与角色）。
- 示例：让 app-sa 拥有 dev 命名空间下 Pod 的 “查看” 和 “创建” 权限
#### 1. 创建 Role（定义权限范围）：
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader-writer
  namespace: dev # 仅在 dev 命名空间生效
rules:
- apiGroups: [""] # 空字符串表示核心 API 组（如 Pod、Service 等）
  resources: ["pods"] # 权限作用于 "pods" 资源
  verbs: ["get", "list", "create"] # 允许的操作：查看详情、列表、创建
```
#### 2. 绑定 Role 与 Service Account
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-sa-pod-binding
  namespace: dev # 与 Role、Service Account 同命名空间
subjects:
- kind: ServiceAccount
  name: app-sa # 要绑定的 Service Account 名称
  namespace: dev # Service Account 所在命名空间
roleRef:
  kind: Role # 绑定的是 Role（非 ClusterRole）
  name: pod-reader-writer # 上述创建的 Role 名称
  apiGroup: rbac.authorization.k8s.io
```
#### 3. 核心场景：Pod 引用 Service Account
Pod 通过 spec.serviceAccountName 字段引用 Service Account，K8s 会自动将该账户关联的 Secret 挂载到 Pod 的 /var/run/secrets/kubernetes.io/serviceaccount/ 目录下，容器内的应用可通过该目录下的 token 和 ca.crt 访问 K8s API。
  
1. Pod引用SA的yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
  namespace: dev
spec:
  serviceAccountName: app-sa # 显式引用 Service Account（默认是 "default"）
  containers:
  - name: app-container
    image: nginx:alpine
    # 容器内可通过环境变量或直接读取文件使用凭证
    command: ["/bin/sh", "-c", "sleep 3600"] # 保持容器运行
```

2. 进容器查看
```bash
# 进入 Pod 容器
kubectl exec -it my-app-pod -n dev -- /bin/sh

# 查看挂载的凭证目录
ls /var/run/secrets/kubernetes.io/serviceaccount/
# 输出：ca.crt  namespace  token

# 查看 token（JWT 格式，可通过 jwt.io 解码查看身份信息）
cat /var/run/secrets/kubernetes.io/serviceaccount/token
```