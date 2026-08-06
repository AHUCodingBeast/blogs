# Spring Cloud 微服务生态核心要点

## 从"Dubbo 之外的另一条路"说起

如果你已经熟悉了 Dubbo 的 RPC 调用模式，那么理解 Spring Cloud 会非常快——它们解决的是同一类问题（服务发现、负载均衡、容错、网关），只是技术选型和设计哲学不同：

| 维度 | Dubbo | Spring Cloud |
| :--- | :--- | :--- |
| 通信协议 | 自定义二进制协议（TCP） | HTTP/REST（也可集成 gRPC） |
| 服务调用方式 | RPC 接口代理 | HTTP 客户端（OpenFeign / WebClient） |
| 注册中心 | ZooKeeper / Nacos | Eureka / Consul / Nacos |
| 网关 | Dubbo Gateway（较新） | Spring Cloud Gateway（主流） |
| 限流熔断 | Sentinel / Hystrix（已停更） | Resilience4j / Sentinel |
| 配置中心 | Apollo / Nacos | Spring Cloud Config / Nacos |
| 生态定位 | 高性能 RPC 框架 | 微服务全家桶解决方案 |

> 💡 **选型建议**：内部系统、高性能要求选 Dubbo；对外 API、前后端分离、团队熟悉 Spring 生态选 Spring Cloud。两者也可以共存（通过 Dubbo-Spring-Cloud 桥接）。

本篇聚焦 Spring Cloud 微服务调用的核心链路，OpenFeign 的详细原理请参考 [OpenFeign 技术原理](OpenFeign-Notes.md)。

---

## 一、微服务调用：从 REST 到声明式客户端

### 整体调用链路

```
Consumer 业务代码
    ↓
OpenFeign 声明式接口（类似 Dubbo 的 @DubboReference）
    ↓
LoadBalancerClient 选择实例（类似 Dubbo LoadBalance）
    ↓
HTTP 请求（RestTemplate / WebClient）
    ↓
Provider Controller 接收请求
    ↓
返回 JSON 响应
```

### OpenFeign 快速回顾

OpenFeign 是 Spring Cloud 中替代手写 HTTP 调用的声明式客户端，体验接近 Dubbo 的 RPC 调用：

```java
// 定义 Feign 客户端接口
@FeignClient(name = "user-service", fallbackFactory = UserFallbackFactory.class)
public interface UserClient {
    
    @GetMapping("/users/{id}")
    User getUser(@PathVariable("id") Long userId);
    
    @PostMapping("/users")
    boolean createUser(@RequestBody User user);
}

// 注入使用，像本地方法一样调用
@Service
public class OrderService {
    @Autowired
    private UserClient userClient;
    
    public Order createOrder(Long userId) {
        User user = userClient.getUser(userId); // HTTP GET /users/{id}
        // ...
    }
}
```

> 📌 **与 Dubbo 的核心区别**：OpenFeign 底层走 HTTP，序列化用 JSON，性能低于 Dubbo 的二进制协议，但与前端/第三方对接更友好。详细原理见 [OpenFeign 技术原理](OpenFeign-Notes.md)。

### 服务发现与负载均衡

Spring Cloud 的服务发现和负载均衡由 **Spring Cloud LoadBalancer** 负责（替代已停更的 Ribbon）：

```yaml
# application.yml
spring:
  cloud:
    discovery:
      enabled: true
    loadbalancer:
      ribbon:
        enabled: false  # 禁用旧版 Ribbon
      configurations: default
```

**支持的负载均衡策略**：

| 策略 | 说明 | 对应 Dubbo |
| :--- | :--- | :--- |
| RoundRobin（默认） | 轮询 | RoundRobin |
| Random | 随机 | Random |
| Weighted | 加权（需自定义） | 加权 Random/RoundRobin |
| LeastConnections | 最少连接数 | LeastActive |

> 💡 **面试加分点**：Spring Cloud LoadBalancer 是响应式的（基于 Reactor），支持异步非阻塞调用，而 Ribbon 是基于阻塞模型的。这也是 Spring Cloud 全面拥抱响应式编程的体现。

---

## 二、限流与熔断：Resilience4j 实战

Spring Cloud 早期使用 Hystrix 做熔断，但 Hystrix 已于 2018 年停止维护。当前官方推荐 **Resilience4j**，它是轻量级、函数式风格的容错库。

### 核心概念对比

| 概念 | Resilience4j | Dubbo/Sentinel 对应 |
| :--- | :--- | :--- |
| CircuitBreaker | 熔断器 | Sentinel DegradeRule |
| RateLimiter | 限流器 | Sentinel FlowRule / Dubbo executes |
| TimeLimiter | 超时控制 | Dubbo timeout |
| Retry | 重试 | Dubbo Failover + retries |
| Bulkhead | 舱壁隔离（线程池/信号量） | Dubbo 线程池隔离 |
| Cache | 缓存 | - |

### 熔断器实战

```java
@Service
public class UserService {
    
    @CircuitBreaker(name = "userService", fallbackMethod = "getUserFallback")
    @TimeLimiter(name = "userService")
    @Retry(name = "userService")
    public CompletableFuture<User> getUser(Long id) {
        return CompletableFuture.supplyAsync(() -> userClient.getUser(id));
    }
    
    // fallback 方法签名必须与原方法一致（含异常参数）
    public CompletableFuture<User> getUserFallback(Long id, Throwable t) {
        log.warn("getUser failed, fallback triggered", t);
        return CompletableFuture.completedFuture(User.defaultUser());
    }
}
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      userService:
        sliding-window-size: 10          # 滑动窗口大小
        failure-rate-threshold: 50       # 错误率超 50% 触发熔断
        wait-duration-in-open-state: 10s # 熔断持续 10s
        permitted-number-of-calls-in-half-open-state: 3 # 半开状态试探请求数
        minimum-number-of-calls: 5       # 最小样本数
  timelimiter:
    instances:
      userService:
        timeout-duration: 3s             # 超时时间
  retry:
    instances:
      userService:
        max-attempts: 3                  # 最大重试次数
        wait-duration: 500ms             # 重试间隔
        retry-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
```

### 限流器实战

```java
@RateLimiter(name = "createOrder", fallbackMethod = "createOrderFallback")
public Order createOrder(CreateOrderRequest request) {
    return orderService.doCreate(request);
}

public Order createOrderFallback(CreateOrderRequest request, RequestNotPermitted ex) {
    throw new BusinessException("系统繁忙，请稍后重试");
}
```

```yaml
resilience4j:
  ratelimiter:
    instances:
      createOrder:
        limit-for-period: 100       # 每个周期允许 100 次请求
        limit-refresh-period: 1s    # 周期为 1 秒
        timeout-duration: 0         # 被限流时立即拒绝（不等待）
```

### 舱壁隔离（Bulkhead）

防止单个服务耗尽全局资源，类似 Dubbo 的线程池隔离：

```yaml
resilience4j:
  bulkhead:
    instances:
      userService:
        max-concurrent-calls: 20     # 最大并发数
        max-wait-duration: 500ms     # 等待获取许可的最大时间
```

### Resilience4j vs Sentinel 选型

| 维度 | Resilience4j | Sentinel |
| :--- | :--- | :--- |
| 风格 | 函数式、注解驱动 | 命令式、Dashboard 可视化 |
| 规则管理 | 配置文件 / 代码 | Dashboard 动态推送 |
| 监控 | Micrometer + Prometheus/Grafana | 自带实时监控面板 |
| 流量控制 | QPS / 并发数 | QPS / 并发数 / 冷启动 / 排队 |
| 适用场景 | Spring Cloud 原生项目 | 阿里系 / 需要精细流量治理 |

> 💡 **建议**：Spring Cloud 原生项目优先 Resilience4j；已有 Sentinel 基础设施或需要 Dashboard 可视化的项目继续用 Sentinel。两者都可通过 Spring Cloud Alibaba 集成。

---

## 三、Spring Cloud Gateway：API 网关

### 为什么需要网关？

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

### 核心架构

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

### 路由配置

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

### 自定义 GlobalFilter

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

### 网关限流：Redis + Token Bucket

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

### 网关 vs Nginx 的定位区别

| 维度 | Spring Cloud Gateway | Nginx |
| :--- | :--- | :--- |
| 层级 | 应用层网关（L7） | 反向代理 / 负载均衡器（L4/L7） |
| 动态路由 | ✅ 支持注册中心自动发现 | ❌ 需手动 reload 配置 |
| 业务逻辑 | ✅ Java 代码扩展（Filter） | ❌ Lua 脚本，能力有限 |
| 性能 | 万级 QPS（Netty 非阻塞） | 十万级 QPS（C 语言） |
| 适用场景 | 微服务内部网关、业务网关 | 公网入口、静态资源、SSL 卸载 |

> 💡 **生产实践**：通常 Nginx 做最外层公网入口（SSL 终止、DDoS 防护），Spring Cloud Gateway 做内层微服务网关（路由、鉴权、限流）。两层各司其职。

---

## 四、服务雪崩防护：与 Dubbo 的对照

Spring Cloud 的雪崩防护思路与 Dubbo 完全一致（超时、限流、熔断），只是工具链不同：

| 防护层 | Dubbo 方案 | Spring Cloud 方案 |
| :--- | :--- | :--- |
| 超时控制 | `timeout` 配置 | Resilience4j TimeLimiter / Feign 超时 |
| 限流 | `executes` / Sentinel | Resilience4j RateLimiter / Gateway 令牌桶 / Sentinel |
| 熔断降级 | Failover/Failfast + Sentinel | Resilience4j CircuitBreaker + fallback |
| 隔离 | 线程池隔离 | Resilience4j Bulkhead |
| 重试 | Failover + retries | Resilience4j Retry |

### Feign 超时配置

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:                    # 全局默认
            connect-timeout: 3000     # 建连超时
            read-timeout: 5000        # 读取超时
          user-service:               # 针对特定服务
            connect-timeout: 2000
            read-timeout: 10000       # 慢接口单独设长超时
```

> 💡 **注意**：Feign 的 `read-timeout` 对应 Dubbo 的 `timeout`，`connect-timeout` 是 TCP 建连超时（Dubbo 长连接模式下不存在此概念）。

---

## 面试高频问题速查

### Q1: Spring Cloud 和 Dubbo 有什么区别？如何选型？
**答**：核心区别：① 通信协议：Dubbo 用自定义二进制 TCP 协议，Spring Cloud 用 HTTP/REST；② 调用方式：Dubbo 是 RPC 接口代理，Spring Cloud 是 HTTP 客户端（OpenFeign）；③ 性能：Dubbo 吞吐量更高（3-5 倍），Spring Cloud 与前端/第三方对接更友好。选型：内部高性能系统选 Dubbo，对外 API / 前后端分离 / Spring 生态团队选 Spring Cloud，两者可通过 Dubbo-Spring-Cloud 桥接共存。

### Q2: Spring Cloud Gateway 的工作原理是什么？
**答**：基于 WebFlux + Netty 的非阻塞网关。核心流程：Route Locator 根据 Predicate 匹配路由 → Filter Chain 执行前置/后置过滤器（鉴权、限流、日志等）→ LoadBalancer 从注册中心选择实例 → HttpClient 转发请求。与 Nginx 的区别：Gateway 支持动态路由（注册中心自动发现）和 Java 代码扩展，Nginx 性能更高但扩展能力有限。生产环境通常 Nginx 做公网入口，Gateway 做内层微服务网关。

### Q3: Resilience4j 的熔断器状态机是怎样的？
**答**：三种状态：CLOSED（正常调用，统计错误率）→ OPEN（错误率超阈值，直接返回 fallback 不调用下游）→ HALF_OPEN（经过等待时间后，放行少量请求试探）。试探成功回到 CLOSED，失败回到 OPEN。关键参数：sliding-window-size（统计窗口）、failure-rate-threshold（错误率阈值）、wait-duration-in-open-state（熔断时长）、minimum-number-of-calls（最小样本数防误触发）。

### Q4: Spring Cloud Gateway 如何做限流？
**答**：内置基于 Redis 的令牌桶算法（RequestRateLimiter Filter）。配置 replenishRate（每秒令牌生成速率）、burstCapacity（桶容量）、key-resolver（限流维度：IP/用户/接口）。相比 Sentinel，优势是与 Gateway 深度集成、无需额外部署；劣势是缺少 Dashboard 可视化和复杂流量控制（冷启动、排队等）。

### Q5: OpenFeign 的超时和重试怎么配置？
**答**：超时通过 `spring.cloud.openfeign.client.config.{name}.read-timeout/connect-timeout` 配置，支持全局和服务级差异化设置。重试默认关闭（因为 HTTP 请求不一定幂等），需显式配置 `spring.cloud.openfeign.client.config.{name}.retryer` 或自定义 Retryer Bean。非幂等接口严禁开启重试，避免重复提交。

### Q6: Spring Cloud 如何实现 TraceId 全链路透传？
**答**：推荐使用 Micrometer Tracing（Spring Boot 3.x）或 Sleuth（2.x）。框架自动在 HTTP Header 中传递 traceId/spanId，Gateway → Feign → 下游服务全链路自动透传，无需手写 Filter。日志框架通过 MDC 自动注入 `%X{traceId}`。如需自定义传播字段，实现 `Propagation.Factory` 或使用 Baggage API。相比 Dubbo 手写 Filter 透传，Spring Cloud 的方案更标准化、零侵入。

### Q7: Spring Cloud Gateway 和 Zuul 有什么区别？
**答**：Zuul 1.x 基于 Servlet + Tomcat 阻塞模型，每个请求占用一个线程，高并发下线程池易耗尽；Gateway 基于 WebFlux + Netty 非阻塞模型，少量 IO 线程即可处理大量并发请求。此外 Gateway 支持响应式编程、动态路由、内置限流等现代特性，Zuul 1.x 已停止维护。**Zuul 2.x 虽改为非阻塞但未开源集成到 Spring Cloud，生产环境应选 Gateway**。

---

## 小结

Spring Cloud 微服务生态的核心价值在于 **"HTTP 友好 + 标准化 + 响应式"**：
-   **HTTP 友好**：基于 REST/HTTP 协议，与前端、第三方、移动端无缝对接；
-   **标准化**：服务发现、负载均衡、熔断限流、网关均有统一抽象，切换实现成本低；
-   **响应式**：Gateway 和 WebClient 基于 WebFlux，天然适合高并发 IO 密集场景；
-   **生态完整**：从配置中心、链路追踪到消息驱动，一站式覆盖微服务全生命周期。

掌握这四个支柱，无论是日常微服务开发还是架构设计，都能做到心中有数。

> 📌 **记忆口诀**：**Feign 调服务、LB 选实例、R4j 防雪崩、Gateway 守大门、Trace 自动传、响应式是高并发**。
