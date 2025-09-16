## 基本介绍

在 Kubernetes 中，TLS 证书（.crt）和私钥（.key）是基于 PKI（公钥基础设施）体系生成的，其产生过程和交互流程严格遵循 TLS 协议原理，用于实现集群组件间的身份认证和加密通信。以下从证书 / 密钥的生成机制、交互流程两方面详细说明：

## 证书和私钥产生过程
### 1. 根 CA 证书（ca.crt）和私钥（ca.key）的生成
根 CA 是整个 K8s 信任链的起点，所有其他证书均由其签发。在集群初始化时（如 kubeadm init），会首先生成根 CA 的私钥和证书：
- 生成私钥（ca.key）:
采用非对称加密算法（如 RSA 或 ECDSA）生成一对密钥，其中私钥（ca.key） 用于后续签名其他证书，需严格保密（一旦泄露，整个集群的信任体系会被攻破）。
```bash
openssl genrsa -out ca.key 2048  # 生成 2048 位 RSA 私钥
```
- 生成根证书（ca.crt）：
基于私钥生成自签名根证书（根 CA 证书无需其他机构签名），证书中包含公钥（从私钥中提取）、CA 标识、有效期等信息，作为集群内所有组件的信任基础。
```
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
  -subj "/CN=kubernetes-ca"  # CN 为 CA 名称，x509 表示自签名证书
```
### 2. 组件证书（如 apiserver.crt、kubelet.crt）和私钥的生成
除根 CA 外，K8s 各组件（API Server、kubelet、控制器管理器等）的证书和私钥需通过 CSR 流程由根 CA 签发，步骤如下：
- 组件生成自身的私钥（.key）：
每个组件（如 API Server）首先生成自己的私钥（如 apiserver.key），用于后续解密和签名。
```bash
openssl genrsa -out apiserver.key 2048
``` 
- 生成证书签名请求（CSR）：
组件基于自身私钥生成 CSR 文件（如 apiserver.csr），包含公钥（从私钥提取）、组件身份信息（如 API Server 的 IP、域名）等。
```bash
openssl req -new -key apiserver.key -out apiserver.csr \
  -subj "/CN=kube-apiserver" \
  -addext "subjectAltName=IP:10.96.0.1,IP:192.168.1.100,DNS:kubernetes,DNS:kubernetes.default"
  # subjectAltName 包含 API Server 的所有可能访问地址，确保客户端验证时匹配
```

### 3. CA 签名生成证书（.crt）：
集群 CA 使用根私钥（ca.key）对 CSR 签名，生成组件证书（如 apiserver.crt）。证书中包含组件公钥、身份信息、CA 签名（证明由可信 CA 签发）、有效期等。
```bash
openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out apiserver.crt -days 365
```

### 4. 证书分发：
签名后的证书（.crt）和组件自身的私钥（.key）被放置在组件的配置目录（如 /etc/kubernetes/pki），供组件通信时使用。

## k8s中tls交互流程
K8s 组件间的通信（如 kubectl 访问 API Server、API Server 访问 kubelet）均通过 TLS 握手实现身份认证和加密，核心流程基于 TLS 协议，以下以 “kubectl 访问 API Server” 为例说明：
1. 握手阶段
   1. 客户端发起连接：kubectl 作为客户端，向 API Server 的 6443 端口（HTTPS 端口）发起连接，请求建立 TLS 会话。
   2. 服务端出示证书：API Server 作为服务端，向 kubectl 发送自己的证书（apiserver.crt），包含：
      - API Server 的公钥；
      - 身份信息（IP、域名，来自 CSR 中的 subjectAltName）；
      - CA 签名（由 ca.key 生成）。
   3. 客户端验证服务端身份：kubectl 收到证书后，通过本地信任的根 CA 证书（ca.crt）验证签名：
      - 用 ca.crt 中的公钥解密 API Server 证书的 CA 签名，得到证书哈希值；
      - 对 API Server 证书的内容（公钥、身份信息等）计算哈希值，与解密结果对比；
      - 若一致，证明 API Server 证书由可信 CA 签发，且内容未被篡改；同时检查证书中的身份信息（如 IP / 域名）是否与目标地址匹配，防止中间人攻击。
   4. 客户端出示证书（双向认证）：验证服务端后，kubectl 向 API Server 发送自己的客户端证书（如 admin.crt），包含用户公钥、身份信息（如 CN=admin）和 CA 签名。
   5. 服务端验证客户端身份：API Server 用 ca.crt 验证客户端证书的合法性（流程同上），确认客户端身份（如 admin 用户），并结合 RBAC 规则判断是否允许访问。
   6. 协商对称加密密钥：身份认证通过后，kubectl 生成一个随机的对称加密密钥（用于后续数据传输，对称加密效率更高），用 API Server 证书中的公钥加密后发送给 API Server。API Server 用自己的私钥（apiserver.key）解密，得到对称密钥。此时双方均持有相同的对称密钥，握手阶段结束。
2. 通信阶段：加密传输与完整性校验
   - 后续所有数据（如 kubectl get pods 的请求和响应）均通过握手阶段协商的对称密钥加密，防止数据被窃听。
   - 同时，每个消息会附加消息验证码（MAC），基于对称密钥和消息内容生成，接收方通过验证 MAC 确保数据未被篡改。

