## 基本介绍
在 Kubernetes（k8s）中，Metrics Server 是一个核心的集群组件，主要用于收集和聚合集群中节点（Node）和 Pod 的资源使用指标（如 CPU、内存使用率等），并通过 Kubernetes API 提供这些指标的查询接口。

## 核心概念与步骤
1. 从节点的 kubelet 服务获取指标：kubelet 是运行在每个节点上的代理，负责收集节点和节点上 Pod 的资源使用数据（如 CPU 使用率、内存占用、网络 IO 等）。

2. 聚合指标数据：Metrics Server 定期从所有节点的 kubelet 拉取数据，并进行聚合。
3. 提供API接口：通过 metrics.k8s.io API 群组暴露指标，供其他组件（如 kubectl top 命令、HPA 等）查询。


## 角色与作用
- kubelet 的角色：每个节点上的 kubelet 内置了一个 /metrics/resource 端点（HTTP 接口），用于暴露该节点及节点上所有 Pod 的资源使用指标（CPU、内存、磁盘、网络等）。这些指标由 kubelet 实时收集并缓存。
- Metrics Server 的行为：Metrics Server 会定期（默认周期约为 10 秒）向集群中所有节点的 kubelet 发送 HTTP 请求，调用 https://<kubelet-ip>:10250/metrics/resource 接口拉取指标数据。
- ApiServer: 通过 metrics.k8s.io API 群组暴露指标，供其他组件（如 kubectl top 命令、HPA 等）查询

## 使用
1. 命令行kubectl top
如果集群没有安装MetricServer，kubectl top命令会报错
```
# 查看节点资源使用
kubectl top nodes

# 查看 Pod 资源使用（指定命名空间）
kubectl top pods -n <namespace>

```

2. 支持 Horizontal Pod Autoscaler（HPA）
HPA 是 Kubernetes 的自动扩缩容机制，依赖 Metrics Server 提供的指标（如 CPU 使用率超过 80% 时自动扩容 Pod）。
例如，基于 CPU 使用率的 HPA 配置：
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80  # 当平均 CPU 使用率超过 80% 时扩容
```

3. 为集群管理和监控提供数据基础
虽然 Metrics Server 只存储最近的指标（无持久化），但它是许多监控工具（如 Prometheus + Grafana）的基础数据来源之一，帮助管理员了解集群负载、排查资源瓶颈。