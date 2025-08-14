## 核心理念
K8S 通过DNS 服务实现 “域名解析”，允许 Pod 通过 “域名” 而非 “IP” 访问 Service 或其他 Pod，简化服务发现。

## 核心 DNS 组件：CoreDNS
K8s 默认使用CoreDNS作为 DNS 服务（部署为 Deployment，运行在kube-system命名空间），负责解析集群内的域名：

- CoreDNS 通过ConfigMap配置解析规则。
- 所有 Pod 的 DNS 配置（/etc/resolv.conf）会被自动指向 CoreDNS 的 Service IP，确保 Pod 能正常解析域名。 以下是一个/etc/resolv.conf的内容示例：
```text
nameserver 10.96.0.10    # k8s 集群 DNS 服务的 ClusterIP
search default.svc.cluster.local svc.cluster.local cluster.local example.com
options ndots:5 timeout:2 attempts:3
```

## 域名格式
1. service的域名
```text
<service-name>.<namespace>.svc.<cluster-domain>
```
- service-name：Service 的名称。
- namespace：Service 所在的命名空间（不同命名空间的 Service 需带命名空间访问）。
- cluster-domain：集群域名（默认是cluster.local）。
- 示例：名为my-service的 Service 在default命名空间，其域名为my-service.default.svc.cluster.local。

2. Pod域名（使用较少）
- 默认格式：pod-ip-address.ns.svc.cluster.local（如10-244-1-5.default.pod.cluster.local）。
- 若 Pod 配置了hostname和subdomain，可自定义域名（如my-pod.my-service.default.svc.cluster.local）。

3. 域名解析示例
- 同一命名空间的 Pod 访问 Service：可直接用service-name（省略命名空间和集群域名）。
例如：Pod 在default命名空间，访问同命名空间的my-service，直接用my-service:80。
- 跨命名空间访问：需带命名空间，如my-service.other-ns:80。