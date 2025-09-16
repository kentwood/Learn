在 Kubernetes（K8s）中，RBAC（Role-Based Access Control，基于角色的访问控制） 是集群中最核心的权限管理机制，用于精确控制 “谁（主体）能对 “什么（资源）” 执行 “什么操作（动词）”。它通过定义角色（权限集合）和绑定（将角色分配给主体），实现了灵活且细粒度的权限管控，是保障集群安全的关键组件。

## 核心概念
RBAC 的核心由 4 个 API 对象 和 3 类主体 构成，理解这些组件是掌握 RBAC 的基础。
### 1. 核心API对象
RBAC 通过以下 4 个自定义资源（CR）定义权限规则和分配关系：
对象类型	|作用范围	|核心作用
---|---|---
Role	|命名空间内	|定义某个命名空间中，允许对哪些资源执行哪些操作（权限集合）。
ClusterRole	|集群级（跨命名空间）	|定义集群范围内，允许对哪些资源执行哪些操作（权限集合，支持集群级资源）。
RoleBinding	|命名空间内	|将 Role 或 ClusterRole 的权限，绑定到某个主体（仅在当前命名空间生效）。
ClusterRoleBinding	|集群级	|将 ClusterRole 的权限，绑定到某个主体（在全集群生效）。

### 2. 主体（Subjects）
“主体” 是被授予权限的对象，RBAC 支持 3 类主体：

- User（用户）：由外部系统管理的人类用户（K8s 本身不存储用户信息，需通过证书、LDAP 等外部系统认证）。
- Group（用户组）：一组用户的集合（如管理员组、开发者组），便于批量授权。
- ServiceAccount（服务账户）：K8s 集群内的 “服务账户”，用于给 Pod 中的进程赋予权限（最常用，因为 Pod 无法直接使用 User）。

### 3.  资源（Resources）与动词（Verbs）
- 资源：K8s 中的 API 资源（如 Pod、Deployment、Service、Node 等），支持复数形式（如 pods）或缩写（如 po）。
- 动词：对资源的操作（如 get、list、create、update、delete、watch 等）。

## 工作流程
1. 定义权限：通过 Role（命名空间内）或 ClusterRole（集群级）声明 "允许对哪些资源执行哪些操作"（如 "允许查看 default 命名空间的 pods"）；
2. 绑定权限：通过 RoleBinding（命名空间内）或 ClusterRoleBinding（集群级）将定义好的 Role/ClusterRole 关联到 Subject；
3. 权限生效：当 Subject 访问集群资源时，K8s 的 API Server 会检查 RBAC 规则，允许符合规则的操作，拒绝不符合的操作。

## 工程示例
### 示例 1：给开发人员授予命名空间内的 "查看权限"
1. 定义 Role（命名空间内的查看权限）
```yaml
# dev-reader-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-reader  # 角色名称
  namespace: dev    # 仅作用于dev命名空间
rules:
- apiGroups: [""]  # 核心API组（如pods、services属于此组）
  resources: ["pods", "pods/log", "services"]  # 允许访问的资源
  verbs: ["get", "list", "watch"]  # 允许的操作（查看）
- apiGroups: ["apps"]  # apps组（如deployments、statefulsets属于此组）
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
```

2. 绑定到开发人员用户（RoleBinding）
```yaml
# dev-reader-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-reader-binding
  namespace: dev  # 与Role同命名空间
subjects:
- kind: User
  name: developer1  # 开发人员用户名（需提前在认证系统中定义）
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-reader  # 关联上面定义的Role
  apiGroup: rbac.authorization.k8s.io
```

### 示例 2：给运维人员授予集群级 "管理权限"
1. 直接使用系统内置的cluster-admin ClusterRole（无需自定义，K8s 默认提供，包含所有权限）；
2. 通过 ClusterRoleBinding 绑定到运维组：
```yaml
# ops-admin-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ops-admin-binding
subjects:
- kind: Group
  name: ops-team  # 运维人员组
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin  # 系统内置的集群管理员角色
  apiGroup: rbac.authorization.k8s.io
```

### 示例 3：给服务账户授予命名空间内的 "操作权限"
场景：prod命名空间的应用需要动态创建 / 更新 ConfigMap（如配置中心客户端），需为其关联的 ServiceAcount 授权。

1. 定义Role（允许操作configmap）
```yaml
# configmap-manager-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-manager
  namespace: prod  # 作用于prod命名空间
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "create", "update", "delete"]  # 允许增删改查
```

2. 创建ServiceAccount(供Pod使用)
```yaml
# app-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-config-sa
  namespace: prod
```

3. 绑定Role到ServiceAccount
```yaml
# configmap-manager-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: configmap-manager-binding
  namespace: prod
subjects:
- kind: ServiceAccount
  name: app-config-sa  # 关联上面创建的服务账户
  namespace: prod
roleRef:
  kind: Role
  name: configmap-manager
  apiGroup: rbac.authorization.k8s.io
```