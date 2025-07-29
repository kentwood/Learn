## 定义
在 Kubernetes（k8s）中，Sidecar（边车） 是一种容器设计模式，指的是与主容器（业务容器）运行在同一个 Pod 中的辅助容器。它与主容器共享网络命名空间（IP、端口等）和存储卷，主要用于扩展主容器的功能，而无需修改主容器本身的代码。

## 特点
1. 与主容器同生命周期（同时启动、停止）
2. 共享网络（可直接通过 localhost 通信）
3. 共享存储卷（方便数据交互）
4. 专注于辅助功能（如日志收集、监控、代理、安全控制等）

## 典型使用场景
1. 日志收集（如用 Filebeat 收集主容器日志）
2. 服务网格代理（如 Istio 的 Envoy 代理）
3. 数据同步 / 备份
4. 安全认证（如注入 TLS 证书）

## 示例
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-sidecar
  labels:
    app: nginx
spec:
  containers:
  # 主容器：Nginx 业务容器
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
    volumeMounts:
    - name: nginx-logs  # 挂载日志目录到共享卷
      mountPath: /var/log/nginx  # Nginx 日志默认路径

  # Sidecar 容器：Filebeat 日志收集器
  - name: filebeat
    image: elastic/filebeat:7.14.0
    volumeMounts:
    - name: nginx-logs  # 共享 Nginx 的日志目录
      mountPath: /var/log/nginx  # 与主容器挂载同一路径，实现日志共享
    - name: filebeat-config  # 挂载 Filebeat 配置文件
      mountPath: /usr/share/filebeat/filebeat.yml
      subPath: filebeat.yml
  volumes:
  - name: nginx-logs  # 共享存储卷（主容器和Sidecar共享）
    emptyDir: {}
  - name: filebeat-config  # Filebeat 配置文件（通过 ConfigMap 挂载）
    configMap:
      name: filebeat-config
---
# Filebeat 配置文件（ConfigMap）
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
data:
  filebeat.yml: |-
    filebeat.inputs:
    - type: log
      paths:
        - /var/log/nginx/access.log  # 监控 Nginx 访问日志
      fields:
        service: nginx
        environment: production

    output.console:  # 简化示例：输出到控制台，实际可配置为 Elasticsearch/Kafka 等
      codec.format:
        string: '%{[@timestamp]} %{[message]}'
    logging.level: info
```

1. 主容器（Nginx）：
- 运行 Nginx 服务，将访问日志写入 /var/log/nginx/access.log
- 通过 volumeMounts 将日志目录挂载到共享卷 nginx-logs
2. Sidecar 容器（Filebeat）：
- 与主容器共享 nginx-logs 卷，因此能直接读取 Nginx 的日志文件
- 通过挂载的 filebeat.yml 配置，监控 /var/log/nginx/access.log 并输出日志（实际场景中可发送到 Elasticsearch 等）
3. 通信方式：
- 无需网络通信，通过共享存储卷直接获取日志
- 若需要网络交互（如代理场景），可通过 localhost:端口 直接访问主容器