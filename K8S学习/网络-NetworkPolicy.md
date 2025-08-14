## 定义
在 Kubernetes (K8s) 中，网络策略（Network Policy） 是用于控制 Pod 之间、Pod 与外部服务之间网络流量的规则集合。它基于 Pod 标签（Label）和命名空间（Namespace）进行流量过滤，实现了微服务间的网络隔离与访问控制。

## 控制方式
网络策略通过以下方式工作：
- 仅允许明确允许的流量通过（默认拒绝所有未被允许的流量）
- 可以控制入站（Ingress）和出站（Egress）流量
- 基于 Pod 标签、命名空间标签、IP 地址范围等进行规则匹配

## 常见示例
1. 默认拒绝所有入站流量
   在命名空间级别设置默认拒绝所有入站流量，只允许明确授权的流量通过：
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default  # 应用到default命名空间
spec:
  podSelector: {}  # 匹配命名空间中所有Pod
  policyTypes:
  - Ingress  # 只控制入站流量
```

2. 允许前端服务访问后端 API 服务
允许带有 app: frontend 标签的 Pod 访问带有 app: backend 标签的 Pod 的 8080 端口：
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend  # 目标Pod（后端服务）
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend  # 允许来源（前端服务）
    ports:
    - protocol: TCP
      port: 8080  # 允许访问的端口

```

3. 保护数据库的网络策略
限制数据库服务只能被应用服务访问，拒绝其他所有流量：
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: protect-database
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: postgres  # 数据库Pod
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-service  # 只允许API服务访问
    ports:
    - protocol: TCP
      port: 5432  # PostgreSQL默认端口

```

4. 允许Pod访问特定的外部ip
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: api-service  # 应用Pod
  policyTypes:
  - Egress  # 控制出站流量
  egress:
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24  # 外部服务的IP范围
    ports:
    - protocol: TCP
      port: 443  # HTTPS端口

```