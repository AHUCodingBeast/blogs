# 认证授权：RBAC、JWT、SSO 与框架实践


## 一、RBAC：权限设计的基石

### 是什么？

RBAC（Role-Based Access Control）= **基于角色的访问控制**。核心思想：**不直接给用户分配权限，而是给用户分配角色，角色再关联权限**。

### 三级模型

```mermaid
graph LR
    U1["用户 A"] --> R1["管理员"]
    U2["用户 B"] --> R2["普通用户"]
    U3["用户 C"] --> R1
    U3 --> R2

    R1 --> P1["user:read"]
    R1 --> P2["user:write"]
    R1 --> P3["order:*"]
    R2 --> P1
    R2 --> P4["order:read"]

    style U1 fill:#e8f5e9,stroke:#4caf50
    style U2 fill:#e8f5e9,stroke:#4caf50
    style U3 fill:#e8f5e9,stroke:#4caf50
    style R1 fill:#fff3e0,stroke:#ff9800
    style R2 fill:#fff3e0,stroke:#ff9800
    style P1 fill:#f0f8ff,stroke:#4a90d9
    style P2 fill:#f0f8ff,stroke:#4a90d9
    style P3 fill:#f0f8ff,stroke:#4a90d9
    style P4 fill:#f0f8ff,stroke:#4a90d9
```

| 级别 | 说明 | 适用场景 |
| :--- | :--- | :--- |
| **RBAC0（基础）** | 用户 → 角色 → 权限，三者多对多 | ⭐️ 90% 的系统够用 |
| **RBAC1（继承）** | 角色可继承父角色权限 | 层级组织（总监 > 经理 > 员工） |
| **RBAC2（约束）** | 互斥角色、基数约束、先决条件 | 合规要求（出纳和会计不能同人） |
| **RBAC3（统一）** | RBAC1 + RBAC2 | 大型企业级系统 |

### 数据库表设计（RBAC0）

```sql
-- 用户表
CREATE TABLE sys_user (
    id BIGINT PRIMARY KEY,
    username VARCHAR(64) NOT NULL,
    password VARCHAR(255) NOT NULL,
    status TINYINT DEFAULT 1
);

-- 角色表
CREATE TABLE sys_role (
    id BIGINT PRIMARY KEY,
    role_name VARCHAR(64) NOT NULL,
    role_code VARCHAR(64) NOT NULL UNIQUE  -- 如 ADMIN, USER
);

-- 权限表
CREATE TABLE sys_permission (
    id BIGINT PRIMARY KEY,
    perm_code VARCHAR(128) NOT NULL UNIQUE, -- 如 user:read, order:write
    perm_name VARCHAR(128),
    resource_type VARCHAR(32)               -- MENU, BUTTON, API
);

-- 用户-角色关联
CREATE TABLE sys_user_role (
    user_id BIGINT,
    role_id BIGINT,
    PRIMARY KEY (user_id, role_id)
);

-- 角色-权限关联
CREATE TABLE sys_role_permission (
    role_id BIGINT,
    permission_id BIGINT,
    PRIMARY KEY (role_id, permission_id)
);
```

### 权限校验流程

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant Filter as 认证过滤器
    participant Service as 业务服务
    participant DB as 数据库/缓存

    Client->>Filter: GET /api/orders (Token)
    Filter->>DB: 验证 Token → 获取 userId
    Filter->>DB: 查 userId 的角色和权限（Redis 缓存）
    Filter->>Filter: 检查是否包含 order:read
    alt 有权限
        Filter->>Service: 放行请求
        Service-->>Client: 200 OK
    else 无权限
        Filter-->>Client: 403 Forbidden
    end
```

### ⚠️ 常见踩坑

| 问题 | 原因 | 解决 |
| :--- | :--- | :--- |
| 权限改了但没生效 | 缓存未刷新 | 修改权限后主动清除该用户的权限缓存 |
| 接口越权 | 只做了菜单级鉴权，没做接口级鉴权 | 每个 API 都加 `@PreAuthorize` 或注解校验 |
| 数据越权 | 有接口权限但能看到别人的数据 | 加数据权限过滤（WHERE user_id = ?） |
| 角色膨胀 | 为每个细微差异创建新角色 | 用权限组合代替角色拆分，或引入 ABAC |

> 💡 **研发日常**：**RBAC0 覆盖 90% 场景**。权限粒度建议：菜单级（前端路由）+ 按钮级（操作权限）+ API 级（接口鉴权）三层防护。数据权限单独处理，不要混入 RBAC。

### 前端权限控制实战：菜单与按钮的动态渲染

> ⚠️ **铁律**：前端的按钮隐藏只是**用户体验优化**，**绝不能作为安全防线**。真正的权限校验必须在后端接口层完成。即使按钮被隐藏了，恶意用户仍然可以通过 Postman/curl 直接调用 API。

#### 核心方案：登录后一次性返回权限树（生产首选）⭐️

```mermaid
sequenceDiagram
    participant User as 用户
    participant FE as 前端
    participant BE as 后端
    
    User->>FE: 登录
    FE->>BE: POST /auth/login
    BE-->>FE: Token + 用户信息 + 权限列表 + 菜单树
    Note over FE: 缓存到 Vuex/Pinia/Redux
    FE->>FE: 根据权限动态渲染菜单和按钮
    User->>FE: 点击"删除用户"按钮
    FE->>FE: 检查是否有 user:delete 权限
    alt 有权限
        FE->>BE: DELETE /api/users/123
        BE->>BE: @PreAuthorize("hasAuthority('user:delete')")
        BE-->>FE: 200 OK
    else 无权限
        FE->>FE: 按钮置灰或隐藏
    end
```

#### 后端返回数据结构示例

```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "userInfo": { "id": 1, "username": "admin", "roles": ["ADMIN"] },
  "permissions": ["user:read", "user:write", "user:delete", "order:*"],
  "menus": [
    {
      "name": "用户管理",
      "path": "/users",
      "icon": "User",
      "children": [
        { "name": "查看用户", "path": "/users/list", "permission": "user:read" },
        { "name": "编辑用户", "path": "/users/edit", "permission": "user:write" },
        { "name": "删除用户", "permission": "user:delete", "type": "BUTTON" }
      ]
    }
  ]
}
```

#### 前端实现要点

**1. 权限判断工具函数**

```javascript
// utils/permission.js
export function hasPermission(permission, userPermissions) {
  if (!permission) return true
  // 支持通配符匹配，如 order:* 匹配 order:read
  return userPermissions.some(p => 
    p === permission || 
    (p.endsWith(':*') && permission.startsWith(p.slice(0, -1)))
  )
}
```

**2. Vue 自定义指令 `v-permission`**

```javascript
// directives/permission.js
import { useUserStore } from '@/stores/user'
import { hasPermission } from '@/utils/permission'

export const permissionDirective = {
  mounted(el, binding) {
    const userStore = useUserStore()
    if (!hasPermission(binding.value, userStore.permissions)) {
      el.parentNode?.removeChild(el) // 无权限则移除 DOM
    }
  }
}

// main.js 中注册
app.directive('permission', permissionDirective)
```

**3. 组件中使用**

```vue
<template>
  <!-- 按钮级权限控制 -->
  <el-button v-permission="'user:delete'" type="danger">删除</el-button>
  
  <!-- 或用 v-if 条件渲染（保留 DOM 但不可见） -->
  <el-button v-if="hasPermission('user:write')" type="primary">编辑</el-button>
  
  <!-- 动态菜单渲染 -->
  <el-menu>
    <template v-for="menu in filteredMenus" :key="menu.path">
      <el-sub-menu v-if="menu.children" :index="menu.path">
        <template #title>{{ menu.name }}</template>
        <el-menu-item 
          v-for="child in menu.children.filter(c => c.type !== 'BUTTON')" 
          :key="child.path"
          :index="child.path"
        >
          {{ child.name }}
        </el-menu-item>
      </el-sub-menu>
    </template>
  </el-menu>
</template>

<script setup>
import { computed } from 'vue'
import { useUserStore } from '@/stores/user'
import { hasPermission } from '@/utils/permission'

const userStore = useUserStore()

// 过滤出当前用户有权限的菜单
const filteredMenus = computed(() => {
  return userStore.menus.filter(menu => {
    // 父菜单：只要有一个子菜单有权限就显示
    if (menu.children) {
      return menu.children.some(child => 
        !child.permission || hasPermission(child.permission, userStore.permissions)
      )
    }
    return !menu.permission || hasPermission(menu.permission, userStore.permissions)
  })
})
</script>
```

**4. 路由守卫配合动态路由**

```javascript
// router/index.js
router.beforeEach(async (to, from, next) => {
  const userStore = useUserStore()
  
  if (!userStore.token) {
    return next('/login')
  }
  
  // 首次加载或刷新时获取权限
  if (!userStore.permissions.length) {
    await userStore.fetchUserInfo() // 调用 /auth/login 或 /auth/info
    // 根据权限动态添加路由
    const dynamicRoutes = generateRoutes(userStore.menus)
    dynamicRoutes.forEach(route => router.addRoute(route))
    return next({ ...to, replace: true }) // 重新导航以触发新路由
  }
  
  // 检查当前路由是否有权限
  if (to.meta.permission && !hasPermission(to.meta.permission, userStore.permissions)) {
    return next('/403')
  }
  
  next()
})
```

#### 方案对比与选型

| 方案 | 适用场景 | 优点 | 缺点 |
| :--- | :--- | :--- | :--- |
| **登录时全量返回** ⭐️ | 中小型后台系统 | 一次请求，性能最好 | 权限变更需重新登录 |
| **按需查询** | 权限实时性要求高 | 实时准确 | 每个页面可能发请求 |
| **混合模式** | 大型企业级系统 | 兼顾性能和实时性 | 实现复杂度较高 |
| **WebSocket 推送** | 权限频繁变更 | 实时推送无需刷新 | 增加服务端复杂度 |

#### 权限实时性：刷新页面后 Pinia 值会更新吗？

> ⚠️ **核心认知**：Pinia 是**内存级状态管理**，刷新页面 = JS 运行时重建 = Pinia 状态归零。刷新后能否拿到最新权限，完全取决于你的初始化策略。

**策略一：每次刷新都重新拉取（推荐 ⭐️）**

这是最安全、最常用的方案。路由守卫中判断"有 Token 但无权限数据"即视为刷新或首次加载，触发重新拉取：

```javascript
// stores/user.js
export const useUserStore = defineStore('user', {
  state: () => ({
    token: localStorage.getItem('token') || '',
    permissions: [],
    menus: []
  }),
  
  actions: {
    async fetchUserInfo() {
      // 每次调用都从后端拉最新数据，保证新鲜度
      const res = await api.getUserInfo()
      this.permissions = res.permissions
      this.menus = res.menus
    }
  }
})

// router/index.js - 路由守卫
router.beforeEach(async (to, from, next) => {
  const userStore = useUserStore()
  
  if (!userStore.token) {
    return next('/login')
  }
  
  // ⭐️ 关键：有 token 但 permissions 为空 → 说明是刷新页面 → 重新拉取
  if (!userStore.permissions.length) {
    await userStore.fetchUserInfo()
    const dynamicRoutes = generateRoutes(userStore.menus)
    dynamicRoutes.forEach(route => router.addRoute(route))
    return next({ ...to, replace: true })
  }
  
  if (to.meta.permission && !hasPermission(to.meta.permission, userStore.permissions)) {
    return next('/403')
  }
  
  next()
})
```

**效果**：刷新后一定拿到最新权限。代价是多一次 API 请求，但换来了**数据一致性和安全性**。

**其他策略速览**

| 策略 | 做法 | 适用场景 | 风险 |
| :--- | :--- | :--- | :--- |
| **持久化到 Storage** | 用 `pinia-plugin-persistedstate` 存 sessionStorage | 对实时性要求不高 | 权限变更后仍是旧值，直到重新登录 |
| **混合模式** | 先用 Storage 快速恢复避免白屏，再静默拉取覆盖 | 兼顾首屏速度和新鲜度 | 实现稍复杂 |
| **WebSocket 推送** | 后端权限变更时主动推送，前端收到后更新 Pinia | 高频权限变更、协同系统 | 需维护长连接 |
| **版本号机制** | 登录时返回 `permissionVersion`，路由守卫比对版本 | 轻量级实时检测 | 仍需额外请求 |

> 💡 **研发日常建议**：**90% 的后台管理系统用策略一就够了**。简单、安全、可靠。只有当业务明确要求"权限修改后不刷新也要立即生效"时，才考虑 WebSocket 或轮询。**不要过早优化**。

> 💡 **面试话术**："管理后台的按钮权限采用前后端分离协作：前端在登录时从后端获取权限列表和菜单树，缓存到状态管理，通过自定义指令（如 `v-permission`）或条件渲染控制按钮显示；后端每个 API 都加注解式鉴权作为安全兜底。刷新页面时通过路由守卫检测权限数据是否为空，为空则重新拉取，确保权限始终最新。核心原则是**前端隐藏是体验优化，后端校验是安全底线**，两者缺一不可。"

---

## 二、JWT：无状态认证的事实标准

### 是什么？

JWT（JSON Web Token）= **自包含的令牌**，服务端签发后不需要存储，客户端携带即可验证身份。解决了传统 Session 在分布式环境下的共享问题。

### JWT 结构

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwicm9sZSI6IkFETUlOIiwiaWF0IjoxNjkxMDAwMDAwLCJleHAiOjE2OTEwODY0MDB9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
│                        │                                                    │
└── Header ──────────────┘                                                    │
     {"alg":"HS256","typ":"JWT"}                                              │
                                                                              │
     └── Payload ─────────────────────────────────────────────────────────────┘
          {"sub":"1234567890","role":"ADMIN","iat":1691000000,"exp":1691086400}
                                                                              │
     └── Signature ───────────────────────────────────────────────────────────┘
          HMACSHA256(base64(header) + "." + base64(payload), secret)
```

| 部分 | 内容 | 注意事项 |
| :--- | :--- | :--- |
| **Header** | 算法（HS256/RS256）、类型 | 不要用 none 算法 |
| **Payload** | 声明（sub/userId/role/exp/iat） | ⚠️ **不是加密的！** 只是 Base64 编码，敏感信息别放这里 |
| **Signature** | 防篡改签名 | 密钥泄露 = 所有 Token 可伪造 |

### JWT vs Session 对比

| 维度 | JWT | Session |
| :--- | :--- | :--- |
| **存储位置** | 客户端（Cookie/LocalStorage/Header） | 服务端（内存/Redis） |
| **服务端状态** | 无状态 | 有状态 |
| **分布式支持** | ✅ 天然支持 | ❌ 需要 Redis 共享 |
| **注销** | ❌ 无法主动失效（除非黑名单） | ✅ 删除 Session 即可 |
| **续期** | 需 Refresh Token 机制 | 自动续期 |
| **安全性** | Token 泄露风险高 | Cookie HttpOnly 更安全 |
| **适用场景** | ⭐️ 微服务、移动端、前后端分离 | 单体应用、后台管理系统 |

### JWT 安全最佳实践

| 实践 | 说明 |
| :--- | :--- |
| **Access Token 短过期** | 15-30 分钟，减少泄露窗口 |
| **Refresh Token 长过期** | 7-30 天，用于静默续期 |
| **HTTPS only** | 防止中间人窃取 Token |
| **HttpOnly Cookie** | 防止 XSS 读取 Token |
| **RS256 非对称签名** | 签发用私钥，验证用公钥，微服务间无需共享密钥 |
| **Token 黑名单** | 注销时将 Token 加入 Redis 黑名单（TTL = Token 剩余有效期） |
| **不在 Payload 存敏感信息** | 只存 userId、role 等非敏感标识 |

> 📖 **深入阅读**：以上仅为速查表，完整的四层防护体系（HTTPS / HttpOnly / SameSite / 短有效期+黑名单）详解、代码示例与安全检查清单，请参考 [JWT Token 安全最佳实践](JWT-Security-Best-Practices.md)。

### Refresh Token 续期流程

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant Server as 服务端
    participant Redis as Redis

    Note over Client: Access Token 过期
    Client->>Server: POST /auth/refresh (Refresh Token)
    Server->>Redis: 验证 Refresh Token 有效性
    alt 有效
        Server->>Server: 签发新 Access Token
        Server->>Redis: 旧 Refresh Token 作废，签发新的
        Server-->>Client: 新 Access Token + 新 Refresh Token
    else 无效/过期
        Server-->>Client: 401 重新登录
    end
```

> 💡 **核心认知**：JWT 不是银弹。**短 Access Token + Refresh Token + 黑名单**三件套才是生产级方案。纯 JWT 无状态 ≠ 完全不需要 Redis，注销和续期仍然依赖它。

---

## 三、SSO：单点登录原理与实践

### 是什么？

SSO（Single Sign-On）= **一次登录，多系统免登**。用户在 A 系统登录后，访问 B/C/D 系统时自动识别身份，无需重复输入密码。

### SSO 核心协议对比

| 协议 | 定位 | 复杂度 | 适用场景 |
| :--- | :--- | :--- | :--- |
| **CAS** | 经典 SSO 协议 | ⭐⭐ 中等 | 企业内部系统、高校 |
| **OAuth2** | 授权协议（非认证） | ⭐⭐⭐ 高 | 第三方登录、API 授权 |
| **OIDC** | OAuth2 + 身份层 | ⭐⭐⭐ 高 | ⭐️ 现代 SSO 首选 |
| **SAML 2.0** | 企业级 XML 协议 | ⭐⭐⭐⭐ 很高 | 传统企业、AD/LDAP 集成 |

### CAS 协议流程（最经典的 SSO）

```mermaid
sequenceDiagram
    participant User as 用户浏览器
    participant AppA as 应用 A
    participant AppB as 应用 B
    participant CAS as CAS Server

    User->>AppA: 访问应用 A
    AppA-->>User: 302 重定向到 CAS/login?service=A
    User->>CAS: 登录（用户名密码）
    CAS-->>User: Set-Cookie(TGT) + 302 重定向到 A?ticket=ST-xxx
    User->>AppA: 带 ticket 访问
    AppA->>CAS: 后端验证 ticket
    CAS-->>AppA: 返回用户信息
    AppA-->>User: 登录成功，Set-Cookie(Session-A)

    Note over User: 访问应用 B（已有 TGT Cookie）
    User->>AppB: 访问应用 B
    AppB-->>User: 302 重定向到 CAS/login?service=B
    User->>CAS: 自动携带 TGT Cookie
    CAS-->>User: 302 重定向到 B?ticket=ST-yyy（免登录！）
    User->>AppB: 带 ticket 访问
    AppB->>CAS: 后端验证 ticket
    CAS-->>AppB: 返回用户信息
    AppB-->>User: 登录成功，Set-Cookie(Session-B)
```

#### CAS 流程逐步解析

**首次登录应用 A（第 1-8 步）**

| 步骤 | 动作 | 细节说明 |
| :--- | :--- | :--- |
| ① | 用户访问应用 A | 浏览器请求 `GET /app-a/dashboard` |
| ② | 应用 A 返回 302 重定向 | 响应头 `Location: https://cas-server.com/login?service=https://app-a.com/callback`，`service` 参数告诉 CAS 认证成功后跳回哪里 |
| ③ | 用户在 CAS 登录页输入凭证 | CAS Server 展示统一登录表单，用户提交用户名密码 |
| ④ | CAS 验证通过，签发 TGT + ST | **TGT**（Ticket Granting Ticket）写入 CAS 域下的 Cookie，代表"全局会话"；**ST**（Service Ticket）是一次性票据，拼接到重定向 URL 中 |
| ⑤ | 浏览器带 ST 回调应用 A | `GET https://app-a.com/callback?ticket=ST-xxx` |
| ⑥ | 应用 A 后端向 CAS 验证 ST | 应用 A 服务端调用 CAS 的 `/serviceValidate?ticket=ST-xxx&service=...` 接口，**这一步是后端到后端的通信**，不经过浏览器，防止 ST 被篡改 |
| ⑦ | CAS 返回用户信息 | CAS 验证 ST 有效且未使用过，返回 XML/JSON 格式的用户标识（如 userId、username），ST 立即作废（一次性） |
| ⑧ | 应用 A 建立本地会话 | 应用 A 根据用户信息创建自己的 Session/Cookie，后续请求不再经过 CAS |

**免登访问应用 B（第 9-16 步）**

| 步骤 | 动作 | 细节说明 |
| :--- | :--- | :--- |
| ⑨ | 用户访问应用 B | 此时浏览器已持有 CAS 域下的 TGT Cookie |
| ⑩ | 应用 B 返回 302 重定向到 CAS | 同步骤②，`service` 参数变为应用 B 的回调地址 |
| ⑪ | 浏览器自动携带 TGT Cookie 访问 CAS | 因为 CAS 域名没变，浏览器自动附带 TGT Cookie，**用户无需再次输入密码** |
| ⑫ | CAS 识别 TGT，直接签发新 ST | CAS 验证 TGT 有效，跳过登录页，直接 302 重定向到应用 B 并附带新的 ST |
| ⑬-⑯ | 同步骤⑤-⑧ | 应用 B 验证 ST → 获取用户信息 → 建立本地会话 |

> 💡 **核心要点**：CAS 的免登本质是**浏览器自动携带 TGT Cookie**。TGT 是 CAS 域下的全局会话凭证，只要 TGT 未过期，访问任何接入 CAS 的应用都无需重新登录。ST 是一次性的，防止重放攻击。

### OIDC 流程（现代 SSO 首选）

OIDC = OAuth2 Authorization Code Flow + ID Token（JWT 格式的用户信息）。

```mermaid
sequenceDiagram
    participant User as 用户浏览器
    participant App as 应用
    participant OP as OIDC Provider

    User->>App: 点击登录
    App-->>User: 302 重定向到 OP/authorize?response_type=code&scope=openid profile
    User->>OP: 登录
    OP-->>User: 302 重定向到 App/callback?code=xxx
    User->>App: 带 code 回调
    App->>OP: POST /token (code + client_secret)
    OP-->>App: access_token + id_token(JWT) + refresh_token
    App->>App: 验证 id_token 签名，提取用户信息
    App-->>User: 登录成功
```

#### OIDC 流程逐步解析

| 步骤 | 动作 | 细节说明 |
| :--- | :--- | :--- |
| ① | 用户点击登录按钮 | 前端触发 OAuth2 Authorization Code Flow |
| ② | 应用重定向到 OP 授权端点 | `GET /authorize?response_type=code&client_id=xxx&redirect_uri=...&scope=openid profile&state=random`。`scope=openid` 是 OIDC 的标志，表示需要身份认证而非仅授权；`state` 参数防 CSRF |
| ③ | 用户在 OP 登录页认证 | OP（OIDC Provider）展示登录页，用户输入凭证或通过已有会话自动认证 |
| ④ | OP 重定向回应用并附带 code | `GET /callback?code=xxx&state=random`。`code` 是临时授权码，**短生命周期、一次性使用**，不能在浏览器端直接使用 |
| ⑤ | 应用后端用 code 换 Token | 应用服务端调用 OP 的 `POST /token` 接口，传入 `code + client_id + client_secret + redirect_uri`。**这一步是后端到后端的通信**，client_secret 不会暴露给浏览器 |
| ⑥ | OP 返回三个 Token | **access_token**：访问受保护资源的凭证；**id_token**：JWT 格式，包含用户身份信息（sub/name/email 等），由 OP 签名；**refresh_token**：用于静默续期 |
| ⑦ | 应用验证 id_token | 应用使用 OP 的公钥验证 id_token 的 JWT 签名，检查 `iss`（签发者）、`aud`（受众=client_id）、`exp`（过期时间）等声明，确认身份真实有效 |
| ⑧ | 应用建立本地会话 | 验证通过后，应用根据 id_token 中的用户信息创建本地 Session/JWT，后续请求使用 access_token 访问资源 |

> 💡 **OIDC vs CAS 关键区别**：
> - CAS 用**一次性 ST** 传递身份，应用需后端回调验证；OIDC 用 **id_token（JWT）** 自包含身份信息，应用可本地验签
> - OIDC 天然支持**授权 + 认证**双重能力（access_token 管授权，id_token 管认证）；CAS 只管认证
> - OIDC 基于 OAuth2 标准生态，与现代 API 网关、微服务无缝集成；CAS 是独立协议，生态较封闭

### SSO 选型决策树

```
你的 SSO 场景是什么？
├── 企业内部多个系统统一登录 → CAS 或 OIDC
├── 接入微信/GitHub/钉钉等第三方登录 → OAuth2 + OIDC
├── 微服务网关统一认证 → OIDC + JWT
├── 对接传统 AD/LDAP → SAML 2.0
└── 新项目从零开始 → ⭐️ OIDC（Keycloak / Authing / 自建）
```

> 💡 **研发日常**：**新项目选 OIDC，老系统改造选 CAS**。OIDC 基于 OAuth2 + JWT，与现代技术栈无缝衔接。Keycloak 是最流行的开源 OIDC Provider，开箱即用。

---

## 四、Spring Security：企业级安全框架

### 核心架构：过滤器链

```mermaid
graph LR
    Request["HTTP 请求"] --> F1["SecurityContextPersistenceFilter<br/>加载/保存 SecurityContext"]
    F1 --> F2["UsernamePasswordAuthenticationFilter<br/>表单登录认证"]
    F2 --> F3["BearerTokenAuthenticationFilter<br/>JWT/OAuth2 Token 认证"]
    F3 --> F4["ExceptionTranslationFilter<br/>异常转换"]
    F4 --> F5["FilterSecurityInterceptor<br/>授权决策"]
    F5 --> Controller["Controller"]

    style F1 fill:#e8f5e9,stroke:#4caf50
    style F2 fill:#e8f5e9,stroke:#4caf50
    style F3 fill:#e8f5e9,stroke:#4caf50
    style F4 fill:#fff3e0,stroke:#ff9800
    style F5 fill:#fff3e0,stroke:#ff9800
```

> 💡 **核心认知**：Spring Security 本质是一条 **Servlet Filter Chain**。每个过滤器负责一个安全职责，请求依次经过认证 → 授权 → 业务处理。理解过滤器链就理解了 Spring Security 的一切。

### 关键组件

| 组件 | 职责 | 研发关注点 |
| :--- | :--- | :--- |
| **SecurityFilterChain** | 过滤器链定义 | 配置哪些路径需要认证/放行 |
| **AuthenticationManager** | 认证入口 | 自定义认证逻辑的扩展点 |
| **UserDetailsService** | 加载用户信息 | 从数据库/缓存查询用户 |
| **PasswordEncoder** | 密码加密 | BCrypt（默认推荐） |
| **AccessDecisionManager** | 授权决策 | RBAC 权限校验 |
| **SecurityContextHolder** | 存储当前用户上下文 | ThreadLocal，获取当前登录用户 |

### 最小配置示例（JWT + RBAC）

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // 禁用 CSRF（JWT 不需要）
            .csrf(csrf -> csrf.disable())
            // 无状态会话（JWT 不需要 Session）
            .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
            // 路径权限配置
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**", "/actuator/health").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/**").authenticated()
                .anyRequest().denyAll()
            )
            // JWT 认证过滤器
            .addFilterBefore(jwtAuthFilter(), BearerTokenAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### ⚠️ 常见踩坑

| 问题 | 原因 | 解决 |
| :--- | :--- | :--- |
| 配置了 permitAll 但仍 403 | CSRF 未禁用 / 过滤器顺序错误 | `.csrf().disable()` + 检查过滤器注册位置 |
| 获取不到当前用户 | 异步线程丢失 SecurityContext | 用 `SecurityContextHolder.setStrategyName(MODE_INHERITABLETHREADLOCAL)` |
| 密码比对失败 | 存了明文或用了不同编码器 | 确认注册和登录用同一个 PasswordEncoder |
| OAuth2 回调 404 | redirect_uri 不匹配 | 严格匹配 OP 注册的回调地址 |

---

## 五、SA-Token：轻量级替代方案

### 为什么选 SA-Token？

| 痛点 | Spring Security | SA-Token |
| :--- | :--- | :--- |
| 学习曲线 | 陡峭，过滤器链复杂 | 平缓，API 直觉化 |
| 代码量 | 大量配置类 | 几行代码搞定 |
| 文档 | 英文为主，分散 | 中文文档完善，示例丰富 |
| 功能 | 全覆盖 | 认证/授权/SSO/OAuth2 够用 |
| 社区 | 全球 | 国内活跃，GitHub 30k+ star |

### 核心 API 速查

```java
// ===== 登录认证 =====
StpUtil.login(userId);                    // 登录
String token = StpUtil.getTokenValue();     // 获取 Token
long loginId = StpUtil.getLoginIdAsLong();  // 获取当前登录用户 ID

// ===== 权限校验 =====
StpUtil.checkPermission("user:read");       // 校验权限，无权限抛异常
StpUtil.hasRole("ADMIN");                   // 检查是否有角色
@SaCheckPermission("order:write")           // 注解式鉴权
@SaCheckRole("ADMIN")                       // 注解式角色校验

// ===== SSO =====
// 客户端：跳转 SSO 中心
return SaSsoUtil.createCheckLoginUrl();
// SSO 中心：验证 Ticket
Object loginId = SaSsoUtil.checkTicket(ticket);

// ===== 踢人下线 / 封禁 =====
StpUtil.kickout(userId);                    // 踢人下线
StpUtil.disable(userId, 86400);             // 封禁 24 小时
```

### Spring Boot 最小实战示例

#### 1. 添加依赖

```xml
<!-- pom.xml -->
<dependency>
    <groupId>cn.dev33</groupId>
    <artifactId>sa-token-spring-boot3-starter</artifactId>
    <version>1.39.0</version>
</dependency>
```

#### 2. 配置文件

```yaml
# application.yml
sa-token:
  token-name: Authorization          # Token 名称（对应请求头）
  timeout: 1800                      # Token 有效期（秒），30 分钟
  active-timeout: -1                 # Token 临时有效期（-1 表示不限制）
  is-concurrent: true                # 是否允许同一账号并发登录
  is-share: true                     # 多次登录是否共享同一个 Token
  token-style: uuid                  # Token 风格：uuid/simple-uuid/random-32
  is-log: false                      # 是否输出操作日志
```

#### 3. 权限数据加载实现

```java
@Component
public class StpInterfaceImpl implements StpInterface {

    @Autowired
    private PermissionService permissionService;

    /**
     * 返回指定用户拥有的权限列表
     */
    @Override
    public List<String> getPermissionList(Object loginId, String loginType) {
        return permissionService.getUserPermissions(Long.parseLong(loginId.toString()));
    }

    /**
     * 返回指定用户拥有的角色列表
     */
    @Override
    public List<String> getRoleList(Object loginId, String loginType) {
        return permissionService.getUserRoles(Long.parseLong(loginId.toString()));
    }
}
```

#### 4. 登录接口

```java
@RestController
@RequestMapping("/auth")
public class AuthController {

    @Autowired
    private UserService userService;

    @PostMapping("/login")
    public Result<?> login(@RequestBody LoginRequest request) {
        // 1. 验证用户名密码
        User user = userService.verifyPassword(request.getUsername(), request.getPassword());
        if (user == null) {
            return Result.fail("用户名或密码错误");
        }

        // 2. 登录（SA-Token 自动创建 Session 并生成 Token）
        StpUtil.login(user.getId());

        // 3. 返回 Token 和用户信息
        Map<String, Object> data = new HashMap<>();
        data.put("token", StpUtil.getTokenValue());
        data.put("userId", user.getId());
        data.put("username", user.getUsername());
        return Result.success(data);
    }

    @PostMapping("/logout")
    public Result<?> logout() {
        StpUtil.logout();
        return Result.success("注销成功");
    }

    @GetMapping("/info")
    public Result<?> currentUser() {
        long userId = StpUtil.getLoginIdAsLong();
        User user = userService.getById(userId);
        return Result.success(user);
    }
}
```

#### 5. 接口鉴权（注解式）

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/list")
    @SaCheckPermission("user:read")       // 需要 user:read 权限
    public Result<?> listUsers() {
        return Result.success(userService.list());
    }

    @PostMapping
    @SaCheckPermission("user:write")      // 需要 user:write 权限
    public Result<?> createUser(@RequestBody CreateUserRequest request) {
        return Result.success(userService.create(request));
    }

    @DeleteMapping("/{id}")
    @SaCheckRole("ADMIN")                 // 需要 ADMIN 角色
    public Result<?> deleteUser(@PathVariable Long id) {
        userService.removeById(id);
        return Result.success("删除成功");
    }

    @GetMapping("/profile")
    @SaCheckLogin                         // 仅需登录即可
    public Result<?> profile() {
        long userId = StpUtil.getLoginIdAsLong();
        return Result.success(userService.getById(userId));
    }
}
```

#### 6. 全局异常处理（未登录/无权限统一响应）

```java
@RestControllerAdvice
public class SaTokenExceptionHandler {

    /**
     * 未登录异常
     */
    @ExceptionHandler(NotLoginException.class)
    public Result<?> handleNotLogin(NotLoginException e) {
        return Result.fail(401, "未登录或 Token 已过期");
    }

    /**
     * 无权限异常
     */
    @ExceptionHandler(NotPermissionException.class)
    public Result<?> handleNotPermission(NotPermissionException e) {
        return Result.fail(403, "无此权限：" + e.getPermission());
    }

    /**
     * 无角色异常
     */
    @ExceptionHandler(NotRoleException.class)
    public Result<?> handleNotRole(NotRoleException e) {
        return Result.fail(403, "无此角色：" + e.getRole());
    }
}
```

#### 7. 拦截器配置（可选，路由级鉴权）

```java
@Configuration
public class SaTokenConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new SaInterceptor(handle -> {
            // 路由拦截：/api/** 需要登录
            SaRouter.match("/api/**")
                .check(r -> StpUtil.checkLogin());
            
            // 排除登录接口
            SaRouter.match("/auth/**").check(r -> {});
        }))
        .addPathPatterns("/**")
        .excludePathPatterns("/auth/**", "/actuator/**");
    }
}
```

> 💡 **研发日常**：SA-Token 的最小可用版本只需要 **依赖 + 配置 + StpInterface 实现 + 登录接口**四步。注解鉴权（`@SaCheckPermission`）和全局异常处理是生产必备。拦截器方式适合路由级粗粒度控制，注解方式适合接口级细粒度控制，两者可混合使用。

### Spring Security vs SA-Token 选型

```
你的项目需要什么？
├── OAuth2/OIDC/LDAP/SAML 等企业级协议 → Spring Security
├── 团队已有 Spring Security 积累 → Spring Security
├── 快速开发、中小项目、国内团队 → ⭐️ SA-Token
└── 微服务网关统一认证 → 两者都行，SA-Token 上手更快
```

> 💡 **面试话术**："Spring Security 是企业级标准，过滤器链架构灵活但复杂；SA-Token 是国产轻量方案，API 简洁、中文文档好，适合中小项目和快速迭代。选型取决于团队积累和项目规模，两者底层都是 Token + RBAC，核心原理相通。"

---

## 六、面试高频问题速答

### Q1：RBAC 是什么？为什么不让用户直接关联权限？

RBAC 是基于角色的访问控制，用户 → 角色 → 权限三层解耦。不让用户直接关联权限是因为：① 角色是权限的逻辑分组，便于批量管理（给 100 个用户加权限只需改角色）；② 符合组织架构（管理员、运营、客服），权限变更只需调整角色而非逐个用户修改；③ 审计友好，可以追溯"谁因为什么角色拥有什么权限"。

### Q2：JWT 的 Payload 是加密的吗？为什么不能存敏感信息？

**不是加密，只是 Base64 编码**。任何人都可以解码看到内容。JWT 的 Signature 只保证"不被篡改"，不保证"不被读取"。所以 Payload 只能存 userId、role 等非敏感标识，密码、手机号等敏感信息绝不能放。如果需要加密，用 JWE（JSON Web Encryption），但一般不建议。

### Q3：JWT 如何实现注销？

JWT 本身是无状态的，服务端不存储 Token，所以无法像 Session 那样"删掉就失效"。生产级方案：**Token 黑名单**——注销时将 Token 写入 Redis，TTL = Token 剩余有效期。每次验证 Token 时先查黑名单，命中则拒绝。代价是引入了 Redis 依赖，但这是安全性和无状态之间的必要折中。

### Q4：SSO 的核心原理是什么？

SSO 的核心是**信任传递**：各应用信任一个统一的认证中心（CAS Server / OIDC Provider）。用户第一次访问应用时被重定向到认证中心登录，认证成功后获得一个临时凭证（Ticket / Authorization Code），应用拿凭证去认证中心后端验证，验证通过即建立本地会话。后续访问其他应用时，浏览器已携带认证中心的 Cookie（TGT / Session），认证中心直接签发新凭证，用户无感知完成登录。

### Q5：Spring Security 的过滤器链是怎么工作的？

Spring Security 本质是一条 Servlet Filter Chain。请求进入后依次经过：SecurityContextPersistenceFilter（加载上下文）→ 认证过滤器（表单/JWT/OAuth2）→ ExceptionTranslationFilter（异常转换）→ FilterSecurityInterceptor（授权决策）→ Controller。每个过滤器职责单一，可以通过 `addFilterBefore/After` 插入自定义过滤器。理解这条链就理解了 Spring Security 的全部工作机制。

---

## 小结

本篇覆盖了认证授权的完整知识体系：

- **RBAC**：权限设计基石，用户→角色→权限三层解耦，RBAC0 覆盖 90% 场景
- **JWT**：无状态认证标准，短 Access Token + Refresh Token + 黑名单三件套
- **SSO**：一次登录多系统免登，新项目选 OIDC，老系统选 CAS
- **Spring Security**：企业级过滤器链架构，功能全面但学习成本高
- **SA-Token**：轻量替代方案，API 简洁，适合中小项目和快速迭代

> 💡 **记忆口诀**：**RBAC 管权限，JWT 当令牌；SSO 免重复登录，Security 链式拦；SA-Token 轻又快，选型看团队和规模**。
