## JWT 的核心原理
1. 结构化：

- JWT 由三部分组成，用点 . 分隔：Header.Payload.Signature
- Header： 通常由两部分组成：
  - typ：令牌类型，通常是 JWT。
  - alg：签名算法，如 HS256（HMAC SHA-256）、RS256（RSA SHA-256）、ES256（ECDSA SHA-256）等。这个算法告诉接收方如何验证签名。
  - 示例：{"alg": "HS256", "typ": "JWT"} -> Base64Url 编码。

- Payload： 包含声明。声明是关于实体（通常是用户）和其他数据的语句。有三种类型的声明：
  - 注册声明： 预定义的、建议使用的标准声明（非强制），如：
    - iss：签发者
    - sub：主题（用户ID）
    - aud：接收方
    - exp：过期时间（Unix 时间戳）
    - nbf：不早于生效时间（Unix 时间戳）
    - iat：签发时间（Unix 时间戳）
    - jti：JWT ID（唯一标识符，防重放攻击）
  - 公共声明： 可以自定义，但为了避免冲突应在 IANA JSON Web Token Registry 中定义或使用抗冲突命名空间（如 URI）。
  - 私有声明： 双方约定好的自定义声明，用于在同意使用它们的各方之间共享信息。
  - 示例：{"sub": "1234567890", "name": "John Doe", "admin": true, "iat": 1516239022} -> Base64Url 编码。

- Signature： 这是 JWT 安全性的核心。它通过对编码后的 Header + '.' + 编码后的 Payload 使用 Header 中指定的算法和一个密钥（Secret Key 或 Private/Public Key）进行签名计算得到。签名用于：
  - 验证消息未被篡改： 如果 Header 或 Payload 被修改，签名验证会失败。
  - 验证发送方身份（如果使用非对称加密）： 如果使用私钥签名（如 RS256），任何拥有公钥的人都可以验证签名确实来自持有私钥的一方。


- 公式：
```
Signature = HMACSHA256(
    base64UrlEncode(header) + "." + base64UrlEncode(payload),
    secret
)
```
（以 HS256 算法为例）

最终 JWT 格式：

```
base64UrlEncode(Header) + '.' + base64UrlEncode(Payload) + '.' + base64UrlEncode(Signature)
```
2. 自包含：

- Payload 本身包含了用户标识（如 sub）和必要的元数据（如 exp）。服务器在验证签名后，可以直接从 Payload 中获取所需信息，无需再去查询数据库或会话存储（至少在令牌有效期内如此）。这是实现“无状态”认证的关键。

3. 签名验证：

- 接收方（通常是服务器）收到 JWT 后：

  1. 将 JWT 按 . 分割成三部分。
  2. 对 Header 和 Payload 部分分别进行 Base64Url 解码。
  3. 检查 Header 中的算法 alg 是否是自己信任和支持的算法（重要安全点：防止 none 攻击）。
  4. 根据算法和预共享的密钥（对称算法如 HS256）或发送方的公钥（非对称算法如 RS256），重新计算 (Base64Url(Header) + '.' + Base64Url(Payload)) 的签名。
  5. 将计算出的签名与收到的 Signature 部分（经过 Base64Url 解码）进行比较。
  6. 如果签名匹配，说明信息未被篡改且（对于非对称算法）来源可信。
  7. 验证 Payload 中的声明（Claims）：
  8. 检查 exp：令牌是否过期？
  9. 检查 nbf：令牌是否已生效？
  10. 检查 iss：签发者是否可信？
  11. 检查 aud：令牌是否发给自己？
  12. （可选）检查 jti：是否在黑名单中？（用于实现登出/令牌吊销，但需额外存储，破坏了部分无状态性）
  13. 验证应用相关的声明（如 role、permission）。

## JWT 的典型用法（身份验证流程）
1. 用户登录： 用户向认证服务器（通常是你的登录 API 端点）发送凭据（如用户名/密码）。
2. 验证凭据： 服务器验证凭据的有效性。
3. 生成 JWT： 验证成功后，服务器：
   - 创建 JWT Header（指定算法 alg）。
   - 创建 JWT Payload，包含用户标识（如 sub: user_id）、角色权限（如 role: "admin"）、签发时间（iat）、过期时间（exp）等必要声明。
   - 使用服务器持有的密钥（对称密钥或私钥）根据指定的算法生成签名。
   - 将 Header、Payload、Signature 三部分拼接并 Base64Url 编码，形成完整的 JWT 字符串。
4. 返回 JWT： 服务器将 JWT 返回给客户端（通常在 HTTP 响应体中，或作为 Set-Cookie Header）。
5. 客户端存储： 客户端（浏览器、App）安全地存储 JWT（常见于 localStorage、sessionStorage、HttpOnly Cookie 或 App 安全存储）。

6. 发送 JWT： 客户端在后续需要认证的请求中发送 JWT。最常用、最推荐的方式是放在 HTTP 请求的 Authorization 头中：（也可以放在 Cookie 或 POST 请求体中，但 Header 方式更标准、更清晰）

```
Authorization: Bearer <your-jwt-token>
```
7. 服务器验证： 受保护的 API 服务器收到请求：
   - 从 Authorization 头中提取 JWT。
   - 按照上述“签名验证”步骤验证 JWT 的完整性和有效性（签名、过期时间、签发者等）。
   - 如果验证通过，从 Payload 中提取用户标识（sub）和权限信息（如 role）。
   - 根据这些信息进行授权判断（该用户是否有权限访问此资源？）。

8. 返回资源： 如果授权通过，服务器处理请求并返回受保护的资源；如果验证或授权失败，返回 401 Unauthorized 或 403 Forbidden 错误。

## JWT 的主要优势
1. 无状态 / 可扩展： 服务器不需要在内存或数据库中存储会话信息（Session）。验证 JWT 只需要密钥和令牌本身，使得服务易于水平扩展。非常适合微服务架构和 RESTful API。
2. 跨域 / 跨平台： 基于标准 JSON 和 HTTP Header，非常适合跨不同域名的单点登录（SSO）场景，也易于在各种客户端（Web、移动端、IoT）使用。
3. 信息自包含： Payload 可以携带有用的用户信息，减少数据库查询次数（但注意不要放敏感信息）。
4. 灵活授权： Payload 可以包含用户角色、权限等数据，方便在 API Gateway 或各个微服务中进行细粒度授权判断。

## JWT 的重要注意事项和安全实践
1. 不要存储敏感信息： JWT Payload 默认是仅Base64编码，不是加密的！任何人都可以解码看到内容。绝对不要在 Payload 中存放密码、信用卡号等敏感信息。 如需传输敏感信息，应使用 JWE（JSON Web Encryption）标准进行加密。

2. 保护密钥（Secret/Private Key）： 对称加密（HS256）的密钥和私钥（RS256/ES256）是安全的核心。必须严格保密，妥善存储（使用密钥管理服务 KMS/Hashicorp Vault，环境变量，绝不硬编码在代码中）。泄露密钥意味着攻击者可以伪造任意令牌。

3. 使用强算法： 优先选择 RS256/ES256 等非对称算法或强对称算法（HS256/HS512）。绝对避免使用 none 算法或不安全的算法（如 HS1）。服务器必须严格校验 alg 头是否是自己信任的算法。

4. 设置合理的过期时间（exp）： 令牌必须有较短的过期时间（如 15-30 分钟），以减少被盗用的风险。结合 Refresh Token 机制来获取新的 Access Token。

5. 使用 HTTPS： 始终通过 HTTPS 传输 JWT，防止中间人攻击窃听令牌。

6. 令牌吊销问题： JWT 一旦签发，在过期前很难主动使其失效（因为服务器是无状态的）。实现登出或强制失效通常需要额外机制：
   - 使用短期令牌 + Refresh Token（可吊销）。
   - 维护一个很小的令牌吊销列表（黑名单），在验证时检查 jti（牺牲部分无状态性）。
   - 将 Session 信息存储在外部高速缓存（如 Redis）中，回归有状态（非 JWT 原生优势）。

7. 客户端安全存储：
- Web：优先考虑 HttpOnly、Secure、SameSite=Strict/Lax 的 Cookie 来存储，防止 XSS 攻击直接读取令牌。LocalStorage/SessionStorage 容易受到 XSS 攻击窃取。
- App：使用平台提供的安全存储机制（Keychain/Keystore）。

8. 验证所有必要的声明： 严格检查 iss（签发者）、aud（受众）、exp（过期时间）、nbf（生效时间）。不要只验证签名。

## 总结
JWT 是一种通过数字签名保证信息完整性和（可选）来源真实性的令牌格式，其核心是 Header.Payload.Signature 的三段式结构。它凭借无状态、自包含、跨域等特性，成为现代 Web API 和微服务架构中身份认证和授权的流行方案。

关键点： 理解签名机制、Payload 非加密性、密钥安全、合理设置过期时间、结合 HTTPS 使用以及注意客户端存储安全。正确和安全地使用 JWT 至关重要。