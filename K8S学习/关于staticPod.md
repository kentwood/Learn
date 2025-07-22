# 说明
在 Kubernetes 中，Static Pod 是一种特殊的 Pod，它由节点上的 kubelet 直接管理，而非通过 API Server。Static Pod 始终绑定到特定节点，且无法通过 kubectl 直接创建或删除，主要用于运行节点级系统服务（如 API Server、kube-scheduler 等控制平面组件）。

# 核心特点
- 直接由 kubelet 管理：
  - 不通过 API Server 创建，无需 Controller Manager、Scheduler 参与。
  - kubelet 定期检查配置文件或目录，自动创建、更新或删除 Static Pod。
- 命名规则：
  - Static Pod 的名称格式为 {pod-name}-{node-name}，例如 nginx-node1。
  - 会在 API Server 中自动创建镜像 Pod（Mirror Pod），用于状态展示。
- 部署位置：
  - 配置文件通常存放在节点的 /etc/kubernetes/manifests 目录（默认路径）。
  - 也可通过kubelet进程中的 --pod-manifest-path 或 --config 参数指定其他路径（这里的config参数指定的是整体的配置路径，还需要查看具体的config文件内容次啊能确定manifest yaml信息）。