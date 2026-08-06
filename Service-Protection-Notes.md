# 服务雪崩防护：限流、熔断与降级

> 📌 **本笔记整合了 Spring Cloud（Sentinel）和 Dubbo 两套体系的服务雪崩防护内容**，从统一的视角理解超时、限流、熔断三层纵深防御。

## 为什么需要雪崩防护？

在微服务调用链中，一个节点的故障可能通过 RPC/HTTP 调用逐级向上传播，最终导致整个系统不可用——这就是**服务雪崩**。

```
用户请求 → Gateway → Service A → Service B → DB
                              ↑
                         B 响应变慢 / 报错
                              ↓
                    A 线程池被占满 → A 不可用
                              ↓
                    Gateway 积压 → 全站不可用
```

防护雪崩不能靠单一手段，需要从三个层面构建**纵深防御**：

| 防护层 | 解决什么问题 | 核心思路 | Spring Cloud 方案 | Dubbo 方案 |
| :--- | :--- | :--- | :--- | :--- |
| **超时控制** | 慢调用阻塞线程池 | 设合理超时，防止无限等待 | Feign timeout / WebClient timeout | `timeout` 配置 |
| **限流** | 流量超过处理能力 | 主动拒绝多余请求 | Sentinel FlowRule / Gateway 令牌桶 | `executes` / Sentinel |
| **熔断降级** | 下游持续失败/慢 | 主动切断调用，走 fallback | Sentinel DegradeRule + fallback | Sentinel / 自定义 fallback |

> 💡 **三者关系**：超时防阻塞蔓延，限流防流量过载，熔断防故障传播。三者互补，缺一不可。

---

## 一、超时控制：防阻塞蔓延

超时是最基础的防线，防止慢调用占住线程池导致级联阻塞。

### Spring Cloud：Feign 超时配置

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:                    # 全局默认
            connect-timeout: 3000     # TCP 建连超时
            read-timeout: 5000        # 读取超时（对应 Dubbo 的 timeout）
          user-service:               # 针对特定服务
            connect-timeout: 2000
            read-timeout: 10000       # 慢接口单独设长超时
```

> 💡 **注意**：Feign 的 `read-timeout` 对应 Dubbo 的 `timeout`，`connect-timeout` 是 TCP 建连超时（Dubbo 长连接模式下不存在此概念）。

### Dubbo：超时配置

```yaml
# Provider 端
dubbo:
  provider:
    timeout: 3000    # 根据业务 P99 RT 设置，通常 2-3 倍 P99 值

# Consumer 端
dubbo:
  consumer:
    timeout: 5000    # ⭐️ 必须 ≥ Provider 端 timeout
```

**关键原则**：
-   Consumer timeout ≥ Provider timeout，避免服务端还在处理时客户端已超时断开；
-   不同接口应设置差异化超时，而非全局一刀切（核心查询 3s，报表导出 30s）；
-   没有超时控制的远程调用就像没有安全阀的水管，下游一堵上游必爆。

#### Dubbo 方法级超时配置

全局 timeout 是兜底默认值，对于慢接口/特殊接口需要单独设置。Dubbo 支持方法级覆盖，优先级高于接口级和全局级。

```java
// Provider 端
@DubboService
public class OrderServiceImpl implements OrderService {
    
    @Override
    public Order getOrder(Long id) {
        return orderRepository.findById(id); // 使用全局 timeout（3000ms）
    }
    
    @Override
    @DubboService(timeout = 30000)  // ⭐️ 报表导出单独设 30s
    public byte[] exportReport(String month) {
        return reportService.generate(month);
    }
}
```

```yaml
# YAML 方式
dubbo:
  services:
    com.example.OrderService:
      timeout: 3000
      methods:
        exportReport:
          timeout: 30000         # ⭐️ 方法级覆盖
        batchQuery:
          timeout: 10000
```

**完整优先级链**：Consumer 方法级 → Consumer 接口级 → Consumer 全局级 → Provider 方法级 → Provider 接口级 → Provider 全局级 → 默认值 1000ms。

---

## 二、限流：防流量过载

当请求速率超过系统处理能力时，主动拒绝多余请求，保护系统不被打垮。

### Sentinel 限流（Spring Cloud & Dubbo 通用）

Sentinel 是国内最主流的限流方案，同时支持 Spring Cloud 和 Dubbo。

#### 接入方式

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080   # Sentinel Dashboard 地址
        port: 8719                  # 客户端与 Dashboard 通信端口
      eager: true                   # 立即初始化（默认懒加载）
```

#### 限流规则（Flow Rule）

```java
// 方式1：注解式（推荐）
@SentinelResource(value = "getUser", blockHandler = "getUserBlock")
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userService.getById(id);
}

// blockHandler 处理限流触发时的降级逻辑
public User getUserBlock(Long id, BlockException ex) {
    return User.defaultUser();
}

// 方式2：代码式（更灵活，适合动态规则）
FlowRule rule = new FlowRule("getUser")
    .setCount(100)                          // QPS 阈值
    .setGrade(RuleConstant.FLOW_GRADE_QPS)  // 按 QPS 限流
    .setLimitApp("default");                // 针对所有调用方
FlowRuleManager.loadRules(Collections.singletonList(rule));
```

**限流模式对比**：

| 模式 | 行为 | 适用场景 |
| :--- | :--- | :--- |
| 快速失败（默认） | 超限立即抛 BlockException | 通用场景 |
| Warm Up（预热） | 阈值从 threshold/coldFactor 缓慢升至 threshold | 秒杀开场、冷启动 |
| 排队等待 | 超限请求排队等待，超过 maxQueueingTimeMs 才拒绝 | 脉冲流量削峰 |

#### Dubbo 原生限流

```yaml
dubbo:
  provider:
    executes: 200     # 服务端最大并发执行数，超过则拒绝新请求
  consumer:
    actives: 10       # 消费端对单个服务的最大并发调用数
```

#### 限流维度对比

| 限流维度 | 工具 | 适用场景 |
| :--- | :--- | :--- |
| Provider 并发数 | Dubbo `executes` | 防止单服务线程池耗尽 |
| Consumer 并发数 | Dubbo `actives` | 防止对下游发起过多并发 |
| 接口级 QPS | Sentinel | 精细控制每个接口的吞吐量 |
| 网关全局 | Spring Cloud Gateway / Nginx | 入口统一管控，防突发流量穿透 |

### Spring Cloud Gateway 内置限流

Gateway 内置基于 Redis 的令牌桶限流，无需额外引入 Sentinel：

```java
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> Mono.just(
        exchange.getRequest().getHeaders().getFirst("X-User-Id")
    );
}
```

```yaml
filters:
  - name: RequestRateLimiter
    args:
      redis-rate-limiter.replenishRate: 50      # 每秒生成 50 个令牌
      redis-rate-limiter.burstCapacity: 100     # 桶最大容量
      key-resolver: "#{@userKeyResolver}"       # 按用户维度限流
```

> 💡 **生产建议**：Gateway 内置限流缺少 Dashboard，生产环境推荐接入 Sentinel Gateway 适配器，支持 QPS/并发数/热点参数限流 + Dashboard 可视化监控，规则可动态推送且支持持久化。

### Sentinel Dashboard 动态规则推送

Sentinel 的核心优势是**规则动态生效**，无需重启应用：

1.  启动 Dashboard：`java -jar sentinel-dashboard-1.8.7.jar`
2.  访问 `http://localhost:8080`，默认账号密码 `sentinel/sentinel`
3.  在 Dashboard 中配置限流/熔断规则 → 实时推送到应用
4.  实时监控 QPS、RT、通过率、拒绝量等指标

> ⚠️ **生产注意**：Dashboard 默认规则存储在内存中，重启丢失。生产环境需接入 Nacos/Apollo/ZK 作为规则数据源，实现规则持久化。

---

## 三、熔断降级：防故障传播

当下游持续失败或响应过慢时，**主动切断调用**，避免无效请求继续消耗资源。这是与限流互补的关键手段：

-   **限流**："流量太大了，挡一部分" → 针对**正常但过量**的请求
-   **熔断**："下游坏了，别再调了" → 针对**异常或慢**的请求

### 熔断器状态机

```
CLOSED（正常调用，统计指标）
    │ 错误率/RT 超阈值
    ▼
OPEN（熔断，直接返回 fallback，不调用下游）
    │ 经过 timeWindow
    ▼
HALF_OPEN（放行少量请求试探）
    │ 成功 → CLOSED ｜ 失败 → OPEN
```

### Sentinel 熔断降级规则

```java
DegradeRule rule = new DegradeRule("getUser")
    .setGrade(CircuitBreakerStrategy.ERROR_RATIO.getType()) // 按错误率熔断
    .setCount(0.5)              // 错误率超 50% 触发
    .setTimeWindow(10)          // 熔断持续 10s
    .setMinRequestAmount(5)     // 最小样本数，防误触发
    .setStatIntervalMs(10000);  // 统计窗口 10s
DegradeRuleManager.loadRules(Collections.singletonList(rule));
```

**三种熔断策略**：

| 策略 | 触发条件 | 适用场景 |
| :--- | :--- | :--- |
| 慢调用比例 | RT 超阈值的请求占比超限 | 下游响应变慢 |
| 异常比例 | 异常请求占比超限 | 下游频繁报错 |
| 异常数 | 异常请求数超限 | 样本量少时用绝对值更准确 |

### Fallback 降级实战

```java
@Service
public class OrderService {
    
    @SentinelResource(
        value = "createOrder",
        blockHandler = "createOrderBlock",      // 限流/熔断触发
        fallback = "createOrderFallback"         // 业务异常触发
    )
    public Order createOrder(CreateOrderRequest request) {
        return orderRepository.save(toEntity(request));
    }
    
    // 限流/熔断降级：参数列表 = 原方法参数 + BlockException
    public Order createOrderBlock(CreateOrderRequest request, BlockException ex) {
        throw new BusinessException("系统繁忙，请稍后重试");
    }
    
    // 业务异常降级：参数列表 = 原方法参数 + Throwable
    public Order createOrderFallback(CreateOrderRequest request, Throwable t) {
        log.error("createOrder failed, fallback", t);
        return Order.pendingOrder(request); // 降级：写入 MQ 异步补偿 / 返回默认值 / 查缓存
    }
}
```

> ⚠️ **blockHandler vs fallback 的区别**：
> -   `blockHandler`：仅处理 Sentinel 触发的限流/熔断（BlockException）
> -   `fallback`：处理所有业务异常（Throwable）
> -   如果同时配置，BlockException 优先走 blockHandler，其他异常走 fallback
> -   两者都可通过 `blockHandlerClass` / `fallbackClass` 指定外部类实现，避免污染业务代码

### 常见降级策略

| 策略 | 适用场景 | 示例 |
| :--- | :--- | :--- |
| 返回缓存数据 | 读操作，允许短暂不一致 | 用户信息、商品详情 |
| 返回默认值 | 非核心字段缺失可接受 | 推荐列表为空、头像用默认图 |
| 返回空结果 | 调用方已有兜底逻辑 | 积分查询失败返回 0 |
| 调用备用服务 | 有冗余部署的关键服务 | 主支付通道挂了切备用通道 |
| 异步补偿 | 写操作，允许延迟成功 | 发消息到 MQ 后续重试 |

---

## 四、Spring Cloud vs Dubbo 容错机制对比

### 核心架构差异

-   **Dubbo**：容错策略（Failover/Failfast/Failsafe 等）**内置在框架核心**中，通过 `cluster` 参数一行配置即可切换；
-   **Spring Cloud**：只提供"调用抽象"（Feign / LoadBalancer），**容错能力完全插件化**，需要自行集成外部组件。

| 容错能力 | Dubbo（内置） | Spring Cloud（插件化） |
| :--- | :--- | :--- |
| 重试 | Failover + retries | Spring Retry / OpenFeign Retryer |
| 熔断降级 | Failfast + Sentinel | Sentinel / Resilience4j |
| 超时控制 | timeout 配置 | Feign timeout / WebClient timeout |
| 限流 | executes / Sentinel | Sentinel / Gateway 令牌桶 |
| 隔离 | 线程池隔离 | Sentinel 线程池隔离 / Bulkhead |
| 快速失败 | Failfast | Sentinel 快速失败模式 |
| 失败安全 | Failsafe | 自定义 fallback |

### Sentinel vs Dubbo 原生容错对比

| 能力 | Sentinel | Dubbo 原生 |
| :--- | :--- | :--- |
| 限流 | QPS / 并发数 / 冷启动 / 排队 | executes（仅并发数） |
| 熔断 | 慢调用/异常比例/异常数 + Dashboard | 无内置熔断（需集成 Sentinel） |
| 规则管理 | Dashboard 动态推送 + 持久化 | 配置文件，重启生效 |
| 监控 | 实时 QPS/RT/通过率面板 | 需额外接入监控 |
| 热点参数限流 | ✅ 支持 | ❌ 不支持 |
| 系统自适应保护 | ✅ CPU/Load/RT 自动限流 | ❌ 不支持 |

### Spring Cloud OpenFeign 重试配置

OpenFeign 默认**不重试**（因为 HTTP 请求不一定幂等），需显式开启：

```java
@Bean
public Retryer feignRetryer() {
    // 初始间隔 100ms，最大间隔 1s，最多重试 3 次
    return new Retryer.Default(100, 1000, 3);
}
```

> ⚠️ **重要提醒**：仅对**幂等接口**（GET 查询、幂等 PUT）开启重试。POST 创建、非幂等 DELETE 等接口严禁重试，避免重复提交。这与 Dubbo Failover 的使用原则完全一致。

### Spring Retry 通用重试

对于非 Feign 调用的场景（如数据库操作、消息发送），使用 Spring Retry：

```java
@Service
public class NotificationService {
    
    @Retryable(
        value = {IOException.class, TimeoutException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 500, multiplier = 2)  // 指数退避
    )
    public void sendNotification(String userId, String message) throws IOException {
        httpClient.post("/notify", new NotifyRequest(userId, message));
    }
    
    @Recover
    public void recoverSendNotification(IOException e, String userId, String message) {
        log.error("sendNotification failed after retries, fallback to MQ", e);
        mqProducer.send("notification-retry", new NotifyRequest(userId, message));
    }
}
```

---

## 面试高频问题速查

### Q1: Sentinel 的熔断降级有哪几种策略？状态机是怎样的？
**答**：三种策略：慢调用比例（RT 超阈值的请求占比超限）、异常比例（异常请求占比超限）、异常数（异常请求绝对数超限）。状态机：CLOSED（正常调用，统计指标）→ OPEN（触发阈值，直接走 fallback 不调用下游）→ HALF_OPEN（经过 timeWindow 后放行少量请求试探）。试探成功回到 CLOSED，失败回到 OPEN。关键参数：count（阈值）、timeWindow（熔断时长）、minRequestAmount（最小样本数防误触发）、statIntervalMs（统计窗口）。

### Q2: Sentinel 的 blockHandler 和 fallback 有什么区别？
**答**：`blockHandler` 仅处理 Sentinel 触发的限流/熔断异常（BlockException），`fallback` 处理所有业务异常（Throwable）。如果同时配置，BlockException 优先走 blockHandler，其他异常走 fallback。方法签名要求：blockHandler 参数列表 = 原方法参数 + BlockException；fallback 参数列表 = 原方法参数 + Throwable。两者都可通过 `blockHandlerClass` / `fallbackClass` 指定外部类实现，避免污染业务代码。

### Q3: Spring Cloud 有没有像 Dubbo 那样的内置容错机制？
**答**：没有。Dubbo 的容错策略（Failover/Failfast/Failsafe 等）内置在框架核心中，通过 cluster 参数一行配置即可切换。Spring Cloud 只提供调用抽象（Feign/LoadBalancer），容错能力完全插件化：重试用 Spring Retry / Feign Retryer，熔断用 Sentinel，超时用 Feign timeout，限流用 Sentinel / Gateway 令牌桶。这是两者的核心架构差异——Dubbo 是"开箱即用的 RPC 框架"，Spring Cloud 是"按需组装的微服务工具箱"。

### Q4: Spring Cloud Gateway 如何做限流？
**答**：两种方式：① 内置基于 Redis 的令牌桶算法（RequestRateLimiter Filter），配置 replenishRate/burstCapacity/key-resolver，与 Gateway 深度集成但缺少 Dashboard；② 接入 Sentinel Gateway 适配器，支持 QPS/并发数/热点参数限流 + Dashboard 可视化监控。生产环境推荐 Sentinel 方案，规则可动态推送且支持持久化。

### Q5: 服务雪崩防护的标准回答是什么？
**答**："服务雪崩防护我采用三层纵深防御：① **超时控制**：Consumer 和 Provider 两端都设置合理 timeout，且 consumer ≥ provider，防止慢调用阻塞线程池；② **限流**：Provider 端用 Dubbo executes 控制并发上限，接口级用 Sentinel 做 QPS 限流，防止流量过载打垮系统；③ **熔断降级**：用 Sentinel 对关键依赖配置熔断规则（错误率/RT），下游持续异常时自动切断调用并走 fallback（缓存/默认值/备用服务），避免故障沿调用链传播。三层各有侧重：超时防阻塞蔓延，限流防流量过载，熔断防故障传播。"

---

## 小结

服务雪崩防护的核心在于**三层纵深防御、两套体系统一理解**：

-   **超时**是最基础的防线，防止慢调用阻塞线程池；
-   **限流**是主动防御，在流量过载时保护系统；
-   **熔断降级**是被动防御，在下游故障时切断传播链路；
-   **Dubbo 内置容错 + Sentinel** 覆盖了 RPC 场景的全链路防护；
-   **Spring Cloud 插件化容错 + Sentinel** 覆盖了 HTTP 微服务场景；
-   无论哪套体系，**Sentinel 都是国内生产环境的限流熔断首选**。

> 📌 **记忆口诀**：**超时防阻塞、限流防过载、熔断防传播、Sentinel 统两家、Dubbo 内置强、Cloud 靠插件**。
