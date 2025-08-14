## 理念
由于 Pod 的 IP 是动态的（销毁后会变化），直接通过 Pod IP 访问不可靠。Service是 K8s 提供的 “稳定访问入口”，通过固定的虚拟 IP（ClusterIP）关联动态的 Pod，实现 “Pod 变化但访问地址不变”。
## Service核心功能
- 固定访问地址：Service 分配一个固定的ClusterIP（属于 “Service 网络”，与 Pod 网络隔离），作为访问后端 Pod 的入口。
- 负载均衡：自动将请求分发到关联的多个 Pod（通过标签选择器selector匹配 Pod）。
- 服务发现：结合 DNS，允许通过域名（而非 IP）访问 Service。

## Service 的网络与 IP
- Service 网络：是集群内专门为 Service 分配 IP 的地址段（与 Pod 网络独立，由 K8s 初始化时配置，如--service-cluster-ip-range=10.96.0.0/12）。
- ClusterIP：Service 的虚拟 IP，仅在集群内部可见（无法被集群外直接访问），不绑定到任何物理网卡，由 K8s 的kube-proxy组件维护路由规则。

## Service访问方式
类型	|作用	|适用场景
---|---|---|
ClusterIP	|默认类型，仅分配集群内可见的虚拟 IP，仅允许集群内访问。	|集群内部服务间通信（如后端 API）。
NodePort	|在每个节点上开放一个静态端口（30000-32767），通过节点IP:NodePort访问。	|临时暴露服务到集群外（如测试）。
LoadBalancer	|结合云厂商负载均衡器（如 AWS ELB、阿里云 SLB），自动分配公网 IP。	|生产环境暴露服务到公网。
ExternalName	|将 Service 映射到外部域名（如example.com），无需选择器。	|访问集群外的外部服务（如数据库）。

## Service 与 Pod 的关联：标签选择器
Service 通过selector（标签选择器）关联 Pod，例如：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:  # 匹配标签为app: my-app的Pod
    app: my-app
  ports:
  - port: 80        # Service暴露的端口（集群内访问用）
    targetPort: 8080 # 转发到Pod的端口（Pod内容器监听的端口）
```
当 Pod 被销毁或重建时，只要标签不变，Service 会自动发现新 Pod 并更新转发规则。

## 