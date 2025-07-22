# docker与containerd的关系
## 1、containerd是docker的核心组件
Docker 在早期版本时采用的是单体架构，功能繁杂。从 1.11 版本开始，Docker 进行了架构调整，将容器运行时相关的核心功能剥离出来，形成了 containerd。如今，containerd 已成为 Docker 的核心容器运行时组件，负责容器的创建、运行和生命周期管理等底层操作。
## 2、containerd是独立的开源项目
containerd 遵循 Apache 2.0 许可协议，是一个独立的开源项目。它提供了简洁的 gRPC 接口，能够与各种上层工具集成，为容器运行时提供支持
## 3、Docker 与 containerd 的依赖关系
Docker 引擎依赖 containerd 来运行容器，但 Docker 自身包含了更多的功能，像镜像构建、Docker CLI、Docker Compose 等。而 containerd 专注于容器运行时功能，相对更加轻量和模块化。

# 在 Kubernetes 中的情况
## 1、容器运行时接口（CRI）
Kubernetes 通过容器运行时接口（CRI）来与不同的容器运行时进行交互，这使得 k8s 可以支持多种容器运行时，而不仅仅局限于 Docker。
## 2、Docker 在 Kubernetes 中的使用
在 k8s 早期版本中，Docker 是默认且最常用的容器运行时。不过，由于 Docker 架构相对复杂，并且包含了一些 k8s 不需要的功能，从 Kubernetes 1.20 版本开始，Docker 被标记为弃用（deprecated）。到了 Kubernetes 1.24 版本，Docker 正式被移除，不再作为直接支持的容器运行时。
## 3、containerd 作为 Kubernetes 的容器运行时
containerd 是 CNCF（Cloud Native Computing Foundation）的项目，它直接实现了 Kubernetes 的 CRI 接口，能够作为 k8s 的容器运行时使用。与 Docker 相比，containerd 更加轻量，资源占用更少，因此在生产环境中得到了广泛应用。目前，containerd 是 Kubernetes 推荐的容器运行时之一。
## 4、containerd 在 Kubernetes 中的部署方式
在 Kubernetes 集群中部署 containerd 时，通常需要安装 containerd 并配置 CRI 插件（如 cri-containerd），这样 kubelet 就可以通过 CRI 与 containerd 进行通信，从而管理容器的生命周期。