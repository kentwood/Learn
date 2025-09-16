## 前言
在 Kubernetes（K8s）中，User（用户）和Group（用户组）是 RBAC 授权模型中的主体（Subject），用于标识访问集群的外部实体（如运维人员、开发人员等）。与ServiceAccount（服务账户，供集群内部 Pod 使用）不同，K8s 本身不提供存储或管理 User/Group 的内置机制，而是通过外部认证组件（如证书、密钥、第三方认证服务等）来定义和识别它们。

## 定义方式
K8s 通过认证插件（Authentication Plugins）识别 User 和 Group，常见的定义方式有以下几种：

### 基于 X.509 证书（最常用）
K8s 推荐使用客户端证书进行认证，User 和 Group 信息直接嵌入在证书的Subject字段中，API Server 通过解析证书自动识别用户身份。

步骤：
1. 生成证书签名请求（CSR）：
在openssl命令中，通过-subj指定 User 和 Group：
```bash
# 定义用户为"developer1"，所属组为"dev-team"和"project-x"
openssl req -new -key developer1.key -out developer1.csr \
  -subj "/CN=developer1/O=dev-team/O=project-x"
```
- CN（Common Name）：表示User 名称（此处为developer1）。
- O（Organization）：表示Group 名称（可多个，此处为dev-team和project-x）。
  
2. 用集群 CA 签名证书：
通过 K8s 的 CA（或集群管理员）对 CSR 签名，生成可被集群信任的证书：
```bash
# 假设使用集群CA（/etc/kubernetes/pki/ca.crt和ca.key）
openssl x509 -req -in developer1.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial -out developer1.crt -days 365
```

3. 配置 kubectl 使用证书：
将证书添加到 kubeconfig 中，后续kubectl操作会自动携带证书进行认证：
```bash
# 添加用户到kubeconfig
kubectl config set-credentials developer1 \
  --client-certificate=developer1.crt \
  --client-key=developer1.key

# 关联用户与集群上下文
kubectl config set-context dev-context \
  --cluster=kubernetes \
  --user=developer1 \
  --namespace=dev
```

此时，API Server 会从证书中解析出：

User：developer1
Groups：dev-team、project-x