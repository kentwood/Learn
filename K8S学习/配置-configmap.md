## 解释
ConfigMap 的核心作用是将分散的非敏感配置 “聚合存储”，并以灵活的方式注入到 Pod 中，支持动态更新（部分场景）

## configmap的创建方式
1. 使用命令行直接创建（简单场景）
通过 kubectl create configmap 快速创建，支持从字面量、文件、目录导入数据：
```yaml
# 1. 从字面量创建（key=value 形式，适合少量配置）
kubectl create configmap nginx-config \
  --from-literal=nginx_port=80 \
  --from-literal=nginx_root=/usr/share/nginx/html

# 2. 从文件创建（文件内容作为 value，文件名作为 key）
# 假设本地有 nginx.conf 文件，内容为 nginx 配置
kubectl create configmap nginx-config --from-file=./nginx.conf

# 3. 从目录创建（目录下所有文件作为 key-value 对，适合多配置文件）
# 假设 ./config 目录下有 app.conf、log.conf 两个文件
kubectl create configmap app-config --from-file=./config
```

2. YAML 配置文件创建（推荐，可版本控制）
更灵活且便于维护，适合复杂配置场景。示例 nginx-config.yaml：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config  # ConfigMap 名称
  namespace: default  # 所属命名空间（默认 default）
data:
  # 1. 键值对形式（适合环境变量、简单参数）
  nginx_port: "80"
  nginx_root: "/usr/share/nginx/html"
  
  # 2. 配置文件形式（value 为完整文件内容，适合复杂配置）
  nginx.conf: |
    server {
      listen $(nginx_port);  # 可引用同 ConfigMap 的其他 key
      root $(nginx_root);
      index index.html;
    }
```

## ConfigMap 注入 Pod 方式
ConfigMap 本身不直接作用于容器，需通过 “注入” 方式让 Pod 内的容器使用其配置，核心有 3 种方式：
1. 作为环境变量注入
将 ConfigMap 的 key-value 直接作为容器的环境变量，适合简单参数（如端口、地址）：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.16
    env:
      # 单个 key 注入（指定 ConfigMap 的 key）
      - name: NGINX_PORT  # 容器内的环境变量名
        valueFrom:
          configMapKeyRef:
            name: nginx-config  # 引用的 ConfigMap 名称
            key: nginx_port     # 引用的 ConfigMap 的 key
      # 单个 key 注入（另一个参数）
      - name: NGINX_ROOT
        valueFrom:
          configMapKeyRef:
            name: nginx-config
            key: nginx_root
      # 批量注入（注入 ConfigMap 所有 key-value）
      - name: ALL_CONFIGS
        valueFrom:
          configMapKeyRef:
            name: nginx-config
            optional: false  # 若 ConfigMap 不存在，Pod 启动失败（默认 false）
```

2. 作为环境变量前缀注入（批量）
通过 envFrom 批量注入 ConfigMap 所有 key-value，避免逐个声明：
```yaml
spec:
  containers:
  - name: nginx
    image: nginx:1.16
    envFrom:
      - configMapRef:
          name: nginx-config  # 注入该 ConfigMap 所有 key-value 为环境变量
          optional: false
```

3. 作为文件挂载到容器（推荐复杂配置）
将 ConfigMap 的 key-value 映射为容器内的文件（key 为文件名，value 为文件内容），适合配置文件（如 nginx.conf、app.conf）：
```yaml
spec:
  containers:
  - name: nginx
    image: nginx:1.16
    volumeMounts:
      # 挂载 ConfigMap 到容器内的 /etc/nginx/conf.d 目录
      - name: nginx-config-volume  # 引用的 Volume 名称
        mountPath: /etc/nginx/conf.d  # 容器内的挂载路径
  volumes:
    # 定义 Volume，关联 ConfigMap
    - name: nginx-config-volume
      configMap:
        name: nginx-config  # 关联的 ConfigMap 名称
        # 可选：指定只挂载部分 key（默认挂载所有）
        items:
          - key: nginx.conf  # ConfigMap 的 key
            path: default.conf  # 容器内的文件名（可自定义，如改为 default.conf）
```