## 介绍
在 Kubernetes（K8s）中，Secret 是一种用于存储和管理敏感信息的资源对象，例如密码、令牌、API 密钥、TLS 证书等。它的核心作用是避免将敏感信息硬编码到容器镜像或配置文件中，从而提高应用的安全性和配置灵活性。

## 核心特性
1. 敏感信息管理：专门用于存储敏感数据，与存储非敏感配置的 ConfigMap 形成互补。
2. 编码而非加密：Secret 中的数据会被自动转换为 Base64 编码（防止明文直接暴露），但 Base64 是编码而非加密，需结合额外机制（如 K8s 加密配置、外部密钥管理系统）实现真正加密。
3. 命名空间隔离：属于命名空间级资源，仅能在所属命名空间内被访问（跨命名空间需特殊配置）。
4. 大小限制：单个 Secret 建议不超过 1MiB（与 ConfigMap 一致），避免影响集群性能。

## 主要类型
|类型标识|	用途|	关键字段示例|
|---|---|---|
|Opaque（默认）	|通用敏感信息（如用户名、密码）	|自定义 key-value 对|
|kubernetes.io/tls|	TLS 证书和私钥（如 HTTPS 配置）	|tls.crt（证书）、tls.key（私钥）
|kubernetes.io/dockerconfigjson	|容器镜像仓库的认证信息（如私有仓库）	|.dockerconfigjson（JSON 格式）
|kubernetes.io/service-account-token	|服务账户（ServiceAccount）的令牌	|token、ca.crt、namespace

## 创建方式
1. 命令行快速创建
适用于临时或简单场景，需先对敏感信息进行 Base64 编码（Opaque 类型）：
```yaml
# 1. 对敏感信息进行 Base64 编码（-n 避免换行符干扰）
echo -n "dbuser" | base64   # 输出：ZGJ1c2Vy
echo -n "dbpass123" | base64  # 输出：ZGJwYXNzMTIz

# 2. 创建 Opaque 类型 Secret
kubectl create secret generic db-creds \
  --from-literal=username=ZGJ1c2Vy \
  --from-literal=password=ZGJwYXNzMTIz

# 3. 创建 TLS 类型 Secret（无需手动编码，直接传入证书文件）
kubectl create secret tls nginx-tls \
  --cert=./server.crt \   # 本地 TLS 证书文件
  --key=./server.key      # 本地私钥文件
```

2. YAML 配置文件创建
适合复杂场景和版本控制，需手动对值进行 Base64 编码。示例：
- Opaque 类型（通用敏感信息）：
```yaml
# db-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
  namespace: default
type: Opaque  # 通用类型
data:
  username: ZGJ1c2Vy  # "dbuser" 的 Base64 编码
  password: ZGJwYXNzMTIz  # "dbpass123" 的 Base64 编码
```
- TLS 类型（证书配置）：
```yaml
# tls-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-tls
  namespace: default
type: kubernetes.io/tls  # TLS 专用类型
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tC...  # 证书文件的 Base64 编码
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tC...  # 私钥文件的 Base64 编码
```

## secret使用方式
1. 作为环境变量注入 Pod
将 Secret 中的 key-value 对作为容器的环境变量，适合需要通过环境变量读取敏感信息的应用：
```yaml
# secret-env-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: my-app:v1
    env:
      - name: DB_USER  # 容器内的环境变量名
        valueFrom:
          secretKeyRef:
            name: db-creds  # 引用的 Secret 名称
            key: username   # Secret 中的 key
      - name: DB_PASS
        valueFrom:
          secretKeyRef:
            name: db-creds
            key: password
```

2. 作为文件挂载到 Pod
将 Secret 中的 key-value 映射为容器内的文件（key 为文件名，value 为文件内容），适合证书、密钥等需要以文件形式读取的场景：
```yaml
# secret-mount-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: tls-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.16
    volumeMounts:
      - name: tls-vol  # 引用的 Volume 名称
        mountPath: /etc/tls  # 容器内的挂载路径
        readOnly: true  # 敏感文件建议设为只读
  volumes:
    - name: tls-vol
      secret:
        secretName: nginx-tls  # 关联的 TLS Secret
        # 可选：指定挂载的 key（默认挂载所有）
        items:
          - key: tls.crt
            path: server.crt  # 容器内的文件名
          - key: tls.key
            path: server.key
```

3. 被其他资源引用
部分 K8s 资源（如 Ingress、ServiceAccount）可直接引用 Secret，无需显式注入 Pod：
```yaml
# 示例：Ingress 引用 TLS Secret 配置 HTTPS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: https-ingress
spec:
  tls:
    - hosts:
        - example.com
      secretName: nginx-tls  # 直接关联 TLS Secret
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-svc
                port:
                  number: 80
```