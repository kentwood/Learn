# Deployment
- 特点：提供声明式更新、滚动更新、回滚、扩缩容等功能。
- 应用场景：无状态应用（如 Web 服务器、API 服务）。

# DaemonSet
DaemonSet 是一种控制器，用于确保集群中的每个节点（或指定节点）都运行一个且仅运行一个特定 Pod 的副本。当有新节点加入集群时，DaemonSet 会自动在其上部署 Pod；当节点被移除时，这些 Pod 也会被自动回收。

DaemonSet 通常用于以下场景：

- 系统级守护进程：如日志收集（Fluentd、Filebeat）、监控代理（Prometheus Node Exporter）。
- 存储驱动：如 Ceph、GlusterFS 存储节点。
- 网络组件：如网络插件（Calico、Flannel）的 Agent。

# StatefulSet
- 特点：为 Pod 提供稳定的网络标识（固定主机名和 DNS）和持久存储（PVC）。
- 应用场景：有状态应用（如数据库、分布式系统）。
- 核心特性：
  - 有序部署和删除（Pod 名称模式：{name}-{index}）。
  - 持久化存储绑定（PVC 与 Pod 生命周期解耦）。
- 示例：部署 MySQL 集群。

# Job
- 特点：创建一个或多个 Pod 执行一次性任务，任务完成后 Pod 终止。
- 应用场景：批处理作业、数据导入、备份任务。
- 核心参数：
  - completions：需要完成的 Pod 总数。
  - parallelism：并行运行的 Pod 数量。


todo