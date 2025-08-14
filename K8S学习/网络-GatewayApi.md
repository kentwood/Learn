## 引入
在 Kubernetes 中，Gateway API 是一套套新一代的网络 API 标准，旨在替代和扩展传统的 Ingress API，提供更强大、更灵活的流量管理能力。它由 Kubernetes SIG-Network 主导开发，目前已进入稳定版本（GA），成为管理集群内外流量的推荐方案。

## 解决的问题
- 解决 Ingress 的局限性：Ingress 仅支持 HTTP/HTTPS 基本路由，缺乏对 TCP/UDP、跨命名空间管理、细粒度权限控制等高级需求的原生支持。
- 标准化流量管理：定义一套统一的 API 规范，让不同的网络插件（如 Nginx、Traefik、Istio 等）实现一致的功能，降低用户学习和迁移成本。
- 声明式与可扩展：通过声明式 API 定义流量规则，同时支持自定义扩展，满足复杂场景（如灰度发布、流量镜像、安全策略等）。

## 原理
Gateway API 引入了多个相互关联的 API 资源，形成层次化的流量管理模型：
1. GatewayClass（集群级资源）
- 作用：定义一组具有相同配置和行为的网关 “模板”，类似于 IngressClass，但功能更丰富。
- 含义：由集群管理员创建，指定网关的实现类型（如 nginx、istio）和配置参数，供用户选择使用。
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx-gateway-class
spec:
  controllerName: k8s.io/ingress-nginx  # 关联的控制器（如Nginx）
  parametersRef:  # 可选，指定该类网关的默认配置
    group: k8s.nginx.org
    kind: NginxGateway
    name: nginx-defaults
```

2. Gateway（命名空间级资源）
- 作用：表示一个具体的 “网关实例”，是流量进入集群的入口点（类似 Ingress 控制器的角色）。
- 含义：绑定到某个 GatewayClass，定义监听的端口、协议（HTTP/HTTPS/TCP 等）、IP 地址等，相当于 “网络接口”。
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-gateway
  namespace: default
spec:
  gatewayClassName: nginx-gateway-class  # 关联的GatewayClass
  listeners:  # 定义监听规则
  - name: http
    protocol: HTTP  # 协议类型
    port: 80        # 监听端口
    hostname: "*.example.com"  # 匹配的域名（支持通配符）
    allowedRoutes:  # 限制哪些路由可以关联到该网关
      namespaces:
        from: Same  # 仅允许同命名空间的Route关联
```

3. HTTPRoute / TCPRoute / UDPRoute（命名空间级资源）
- 作用：定义具体的流量路由规则，类似 Ingress 中的 rules，但支持多协议（HTTP/TCP/UDP 等）。
- 含义：关联到某个 Gateway，通过 host、path、header 等条件匹配流量，并转发到后端 Service。
```yaml
# 最常用的HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-api-route
  namespace: default
spec:
  parentRefs:  # 关联到前面定义的Gateway
  - name: prod-gateway
    namespace: default
  hostnames:  # 匹配的域名
  - "api.example.com"
  - "web.example.com"
  rules:  # 路由规则
  - matches:  # 匹配条件
    - path:
        type: Prefix
        value: /api  # 路径前缀匹配
      headers:  # 头部匹配（可选）
      - name: X-API-Version
        value: "v1"
    backendRefs:  # 转发到的后端服务
    - name: api-v1-service
      port: 80
      weight: 90  # 权重（用于灰度发布）
    - name: api-v2-service
      port: 80
      weight: 10
  - matches:
    - path:
        type: Prefix
        value: /web
    backendRefs:
    - name: web-service
      port: 80
```

4. ReferenceGrant（命名空间级资源）
- 作用：控制跨命名空间的资源引用权限，解决 Gateway API 中不同命名空间资源的访问安全问题。
- 含义：允许某个命名空间的 Route 引用另一个命名空间的 Service，或允许跨命名空间关联 Gateway。
```yaml
# 允许 default 命名空间的 Route 引用 backend 命名空间的 Service
apiVersion: gateway.networking.k8s.io/v1
kind: ReferenceGrant
metadata:
  name: allow-default-to-backend
  namespace: backend  # 被引用资源所在的命名空间
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: default  # 允许来自default命名空间的HTTPRoute
  to:
  - group: ""
    kind: Service  # 允许引用Service资源
```

## GatewayApi与Ingress的区别
特性|	Ingress API|	Gateway API
---|---|---
协议支持|	仅 HTTP/HTTPS|	支持 HTTP/HTTPS/TCP/UDP 等
资源模型|	单一 Ingress 资源|	多层次资源（GatewayClass、Gateway、Route 等）
跨命名空间管理|	不原生支持（需控制器扩展）|	原生支持（通过 ReferenceGrant）
权限控制|	无细粒度控制	|支持角色分离（集群管理员、命名空间管理员、开发者）
扩展能力|	依赖注解（Annotations）|	原生支持扩展字段和策略
高级路由功能|	有限（需控制器自定义注解）	|原生支持权重、头部匹配、路径重写等

## 使用方式
1. 部署支持 Gateway API 的控制器
Gateway API 需要对应的控制器才能生效，主流网络插件（如 Nginx、Traefik、Cilium、Istio）均已支持。以 Nginx 为例：
```yaml
# 部署支持 Gateway API 的 Nginx 控制器
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.0/deploy/static/provider/cloud/deploy.yaml
```

2. 创建 GatewayClass 和 Gateway
```yaml
# 创建 GatewayClass（内容参考上面）
kubectl apply -f gatewayclass.yaml

# 创建 Gateway（入口点）
kubectl apply -f gateway.yaml
```

3. 创建 Route 规则并关联 Gateway
```yaml
# 创建 HTTPRoute 定义路由
kubectl apply -f httproute.yaml
```

4. 验证配置
```yaml
# 查看 Gateway 状态（获取入口 IP）
kubectl get gateway prod-gateway -o wide

# 查看 HTTPRoute 状态
kubectl get httproute web-api-route
```

## 适用场景
- 复杂流量管理：需要基于 header、权重、跨协议（TCP/UDP）的路由。
- 多团队协作：需要细粒度的权限控制（如集群管理员管理网关，开发者管理路由）。
- 跨命名空间服务暴露：需要安全地跨命名空间引用服务。
- 未来兼容性：作为 Kubernetes 官方推荐的下一代网络 API，适合新集群或迁移场景。