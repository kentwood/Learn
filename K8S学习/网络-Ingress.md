## 引入
在 Kubernetes（K8s）中，Ingress 是用于管理集群外部访问集群内服务（通常是 HTTP/HTTPS 流量）的 API 资源。它通过定义路由规则，将外部请求转发到集群内对应的 Service，解决了 Service 暴露方式（如 NodePort、LoadBalancer）在复杂路由场景下的局限性。

## 概念
1. 解决的问题
- 统一入口：为集群内多个服务提供单一的外部访问入口，避免为每个服务暴露独立的端口或 IP。
- 灵活路由：支持基于域名（host）、路径（path）的细粒度流量转发（例如 api.example.com 路由到 API 服务，web.example.com 路由到 Web 服务）。
- SSL 终止：集中管理 HTTPS 证书，在入口处解密 HTTPS 流量，简化后端服务的配置。
- 负载均衡：配合 Ingress 控制器实现请求的负载均衡。

2. 关键组件
- Ingress 资源：K8s API 资源，用于定义路由规则（YAML 配置），例如 “哪些域名 / 路径的请求转发到哪个 Service”。
- Ingress 控制器：实际执行 Ingress 规则的组件（通常是一个运行在集群内的 Pod），负责接收外部流量并按规则转发。
- 注意：K8s 本身不自带 Ingress 控制器，需单独部署（如 Nginx Ingress Controller、Traefik、HAProxy 等）。

## 原理
- 流量入口：外部 HTTP/HTTPS 请求首先到达集群的边缘节点（通常通过 LoadBalancer 或 NodePort 暴露 Ingress 控制器）。
- 规则匹配：Ingress 控制器监听 Ingress 资源的变化，当请求到达时，根据 Ingress 资源中定义的规则（host、path 等）匹配目标 Service。
- 转发流量：Ingress 控制器将请求转发到匹配的 Service，再由 Service 通过标签选择器转发到后端 Pod。

流程简化：
外部请求 → Ingress 控制器（按规则匹配）→ Service → 后端 Pod

## Ingress 资源结构解析
一个基本的 Ingress 资源 YAML 结构如下，核心字段包括 rules（路由规则）和 tls（HTTPS 配置）：
```yaml
apiVersion: networking.k8s.io/v1  # 固定版本
kind: Ingress
metadata:
  name: example-ingress  # Ingress 名称
  namespace: default     # 所属命名空间
spec:
  ingressClassName: nginx  # 指定 Ingress 控制器（需与部署的控制器名称匹配）
  tls:  # HTTPS 配置（可选）
  - hosts:
    - api.example.com     # 需配置 HTTPS 的域名
    secretName: api-tls   # 存储证书的 Secret 名称
  rules:  # 路由规则列表（至少一个）
  - host: api.example.com  # 匹配的域名（空表示匹配所有域名）
    http:
      paths:  # 基于路径的路由
      - path: /v1          # 匹配的路径（如 /v1/xxx）
        pathType: Prefix   # 路径匹配类型（Prefix/Exact/ImplementationSpecific）
        backend:
          service:
            name: api-v1   # 目标 Service 名称
            port:
              number: 80   # 目标 Service 端口
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 80
  - host: web.example.com  # 另一个域名的规则
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

核心字段说明：
- ingressClassName：指定使用的 Ingress 控制器（需提前部署并创建 IngressClass 资源关联控制器）。
- tls：定义 HTTPS 配置，secretName 指向存储 SSL 证书和私钥的 Secret（需提前创建）。
- rules.host：匹配的域名（如 api.example.com），空值表示匹配所有域名。
- paths.path：URL 路径（如 /v1），结合 pathType 实现路径匹配：
- Prefix：前缀匹配（如 /v1 匹配 /v1/xxx）。
- Exact：精确匹配（如 /v1 仅匹配 /v1）。
- ImplementationSpecific：由 Ingress 控制器决定匹配规则。
- backend.service：目标 Service 的名称和端口，请求将转发到该 Service。

## 使用Ingress步骤
1. 部署 Ingress 控制器（以 Nginx 为例）
Ingress 资源需要控制器才能生效，以下是部署 Nginx Ingress Controller 的简化步骤：
```yaml
# 使用官方 Helm 图表部署（推荐）
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.publishService.enabled=true
```

2. 创建后端 Service 和 Pod
假设我们有两个服务需要暴露：api-service（80 端口）和 web-service（80 端口），需先创建对应的 Deployment 和 Service：
```yaml
# api-service 示例
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: default
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080  # 对应 Pod 的端口
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: nginx:alpine
        ports:
        - containerPort: 8080
```

3. 创建 Ingress 资源定义路由规则
创建 ingress.yaml 定义路由规则，例如将 api.example.com 路由到 api-service，web.example.com 路由到 web-service：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: default
spec:
  ingressClassName: nginx  # 与部署的Nginx控制器的IngressClass匹配
  rules:
  - host: api.example.com  # 外部访问的API域名
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service  # 转发到api-service
            port:
              number: 80
  - host: web.example.com  # 外部访问的Web域名
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service  # 转发到web-service
            port:
              number: 80

```

4. 配置 HTTPS（可选）
若需启用 HTTPS，需先创建存储 SSL 证书的 Secret（证书需提前申请）：
```yaml
# 创建 TLS Secret（cert.pem为证书，key.pem为私钥）
kubectl create secret tls api-tls \
  --cert=cert.pem \
  --key=key.pem \
  --namespace=default
```
然后在 Ingress 资源中添加 tls 配置：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: https-app-ingress
  namespace: default
spec:
  ingressClassName: nginx
  tls:  # 启用HTTPS
  - hosts:
    - api.example.com  # 对该域名启用HTTPS
    secretName: api-tls  # 引用之前创建的TLS Secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80

```

5. 访问验证
- 获取 Ingress 入口 IP：
查看 Ingress 控制器的 Service 外部 IP（云环境）或节点 IP+NodePort（本地环境）：
```text
kubectl get svc -n ingress-nginx
# 输出示例（云环境）：
# NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
# nginx-ingress-ingress-nginx-controller   LoadBalancer   10.96.xx.xx    203.0.113.10    80:30080/TCP,443:30443/TCP   5m
```
- 配置本地 hosts（测试用）：
将域名映射到 Ingress 入口 IP（如 203.0.113.10 api.example.com）。
- 访问验证
```text
curl http://api.example.com  # 应返回api-service的响应
curl https://api.example.com # 若配置了HTTPS，应返回加密响应
```

## 其他常用高级功能
1. 跨域（CORS）配置：允许跨域请求：
```yaml
annotations:
  nginx.ingress.kubernetes.io/cors-allow-origin: "*"
  nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST"
```
2. 限流
```yaml
annotations:
  nginx.ingress.kubernetes.io/limit-rps: "10"  # 每秒最多10个请求
```