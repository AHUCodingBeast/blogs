# Spring Cloud Gateway：API 网关核心要点

## 为什么需要网关？

在微服务架构中，网关是系统的**统一入口**，承担以下职责：

| 职责 | 说明 | 类比 |
| :--- | :--- | :--- |
| 路由转发 | 根据路径/Header 将请求路由到对应服务 | Nginx location |
| 负载均衡 | 对目标服务的多实例做 LB | Dubbo LoadBalance |
| 认证鉴权 | 统一校验 Token / Session | Servlet Filter |
| 限流熔断 | 入口级流量保护 | Sentinel Gateway |
| 日志监控 | 统一记录请求/响应日志 | Dubbo Filter |
| 协议转换 | HTTP ↔ gRPC / WebSocket | - |
| 灰度发布 | 按 Header/Cookie 路由到新版本 | Dubbo Tag Route |

## 核心架构

```
Client (Browser/App)
    ↓
Spring Cloud Gateway
    ├── Route Locator（路由匹配）
    ├── Filter Chain（过滤器链）
    │   ├── Pre Filters（前置：鉴权、限流、日志）
    │   └── Post Filters（后置：改写响应、添加 Header）
    ├── LoadBalancer（服务实例选择）
    └── HttpClient（WebFlux Netty，非阻塞）
    ↓
Microservice A / B / C
```

> ⚠️ **关键区别**：Spring Cloud Gateway 基于 **WebFlux + Netty**（非阻塞 IO），而非 Spring MVC + Tomcat（阻塞 IO）。这意味着它天然适合高并发网关场景，但**不支持 Servlet API**，传统 Filter/Interceptor 无法直接使用。

## 路由配置

```yaml
spring:
  cloud:
    gateway:
      routes:
        # 基础路由：路径匹配
        - id: user-service
          uri: lb://user-service          # lb:// 表示从注册中心获取实例并负载均衡
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1               # 去掉 /api 前缀再转发
            
        # 高级路由：多条件 + 限流 + 熔断
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
            - Method=POST,PUT
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100   # 每秒令牌数
                redis-rate-limiter.burstCapacity: 200   # 桶容量
                key-resolver: "#{@ipKeyResolver}"       # 限流维度
                
        # 灰度路由：按 Header 分流
        - id: user-service-canary
          uri: lb://user-service-canary
          predicates:
            - Path=/api/users/**
            - Header=X-Canary, true
          filters:
            - StripPrefix=1
```

## 自定义 GlobalFilter

GlobalFilter 对所有路由生效，适合做统一的鉴权、日志、TraceId 透传：

```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        
        // 白名单路径直接放行
        String path = exchange.getRequest().getURI().getPath();
        if (path.startsWith("/api/public")) {
            return chain.filter(exchange);
        }
        
        // 校验 Token
        if (token == null || !jwtService.validate(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        
        // 将用户信息放入 Header 传递给下游
        ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
            .header("X-User-Id", jwtService.getUserId(token))
            .build();
            
        return chain.filter(exchange.mutate().request(mutatedRequest).build());
    }

    @Override
    public int getOrder() {
        return -100; // 优先级，数值越小越先执行
    }
}
```

## 网关限流：Redis + Token Bucket

Spring Cloud Gateway 内置基于 Redis 的令牌桶限流，无需额外引入 Sentinel：

```java
// 定义限流维度解析器
@Bean
public KeyResolver ipKeyResolver() {
    return exchange -> Mono.just(
        Objects.requireNonNull(exchange.getRequest().getRemoteAddress())
            .getAddress().getHostAddress()
    );
}

@Bean
public KeyResolver userKeyResolver() {
    return exchange -> Mono.just(
        exchange.getRequest().getHeaders().getFirst("X-User-Id")
    );
}
```

```yaml
# 路由中引用
filters:
  - name: RequestRateLimiter
    args:
      redis-rate-limiter.replenishRate: 50      # 每秒生成 50 个令牌
      redis-rate-limiter.burstCapacity: 100     # 桶最大容量 100
      redis-rate-limiter.requestedTokens: 1     # 每次请求消耗 1 个令牌
      key-resolver: "#{@userKeyResolver}"       # 按用户维度限流
```

## 网关 vs Nginx 的定位区别

| 维度 | Spring Cloud Gateway | Nginx |
| :--- | :--- | :--- |
| 层级 | 应用层网关（L7） | 反向代理 / 负载均衡器（L4/L7） |
| 动态路由 | ✅ 支持注册中心自动发现 | ❌ 需手动 reload 配置 |
| 业务逻辑 | ✅ Java 代码扩展（Filter） | ❌ Lua 脚本，能力有限 |
| 性能 | 万级 QPS（Netty 非阻塞） | 十万级 QPS（C 语言） |
| 适用场景 | 微服务内部网关、业务网关 | 公网入口、静态资源、SSL 卸载 |

> 💡 **生产实践**：通常 Nginx 做最外层公网入口（SSL 终止、DDoS 防护），Spring Cloud Gateway 做内层微服务网关（路由、鉴权、限流）。两层各司其职。

## 面试高频问题

### Q: Spring Cloud Gateway 的工作原理是什么？
**答**：基于 WebFlux + Netty 的非阻塞网关。核心流程：Route Locator 根据 Predicate 匹配路由 → Filter Chain 执行前置/后置过滤器（鉴权、限流、日志等）→ LoadBalancer 从注册中心选择实例 → HttpClient 转发请求。与 Nginx 的区别：Gateway 支持动态路由（注册中心自动发现）和 Java 代码扩展，Nginx 性能更高但扩展能力有限。生产环境通常 Nginx 做公网入口，Gateway 做内层微服务网关。

### Q: Spring Cloud Gateway 如何做限流？
**答**：两种方式：① 内置基于 Redis 的令牌桶算法（RequestRateLimiter Filter），配置 replenishRate/burstCapacity/key-resolver，与 Gateway 深度集成但缺少 Dashboard；② 接入 Sentinel Gateway 适配器，支持 QPS/并发数/热点参数限流 + Dashboard 可视化监控。生产环境推荐 Sentinel 方案，规则可动态推送且支持持久化。

### Q: Spring Cloud Gateway 和 Zuul 有什么区别？
**答**：Zuul 1.x 基于 Servlet + Tomcat 阻塞模型，每个请求占用一个线程，高并发下线程池易耗尽；Gateway 基于 WebFlux + Netty 非阻塞模型，少量 IO 线程即可处理大量并发请求。此外 Gateway 支持响应式编程、动态路由、内置限流等现代特性，Zuul 1.x 已停止维护。**Zuul 2.x 虽改为非阻塞但未开源集成到 Spring Cloud，生产环境应选 Gateway**。
