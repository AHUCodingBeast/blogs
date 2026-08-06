# Dubbo 核心原理与实战要点

## 从一个"RPC 调用像本地方法一样简单"的期望说起

在微服务架构中，服务之间的调用是最高频的操作。你是否经历过这些痛点：

1.  **HTTP 调用的繁琐**：每次跨服务调用都要手动拼接 URL、序列化参数、处理响应码、反序列化结果，代码冗长且易出错；
2.  **服务发现的硬编码**：服务地址写死在配置文件或代码里，扩容缩容时需要手动更新所有调用方，运维成本极高；
3.  **故障处理的缺失**：下游服务挂了，调用方直接抛异常，没有重试、降级、熔断机制，一个节点故障引发雪崩；
4.  **性能瓶颈**：JSON over HTTP 的序列化开销大、连接复用差，高并发下成为系统瓶颈。

这些问题的共同诉求是：**让远程调用像调用本地方法一样简单、透明、可靠**。Dubbo 正是为解决这些问题而生的高性能 RPC 框架——它屏蔽了网络通信、服务发现、负载均衡、容错处理等底层细节，让开发者只需关注业务接口定义。

---

## 一、基本使用：5 分钟上手

### 项目结构

```
dubbo-demo/
├── dubbo-api/          # 接口定义模块（消费方 & 提供方共同依赖）
│   └── UserService.java
├── dubbo-provider/     # 服务提供方
│   ├── UserServiceImpl.java
│   └── application.yml
└── dubbo-consumer/     # 服务消费方
    ├── OrderService.java
    └── application.yml
```

### Step 1: 定义服务接口（API 模块）

```java
// dubbo-api 模块，仅包含接口和 DTO，不含实现
public interface UserService {
    User getUserById(Long userId);
    boolean createUser(User user);
}
```

### Step 2: 服务提供方实现并暴露

```java
@DubboService(version = "1.0.0", group = "user")  // ⭐️ 声明为 Dubbo 服务
@Service
public class UserServiceImpl implements UserService {
    
    @Override
    public User getUserById(Long userId) {
        // 业务逻辑
        return userRepository.findById(userId).orElse(null);
    }
    
    @Override
    public boolean createUser(User user) {
        userRepository.save(user);
        return true;
    }
}
```

```yaml
# dubbo-provider/application.yml
dubbo:
  application:
    name: user-service
  protocol:
    name: dubbo
    port: 20880
  registry:
    address: nacos://127.0.0.1:8848
  provider:
    timeout: 3000          # 服务端超时时间
    retries: 2             # 重试次数（不含首次调用）
    loadbalance: random    # 负载均衡策略：random/roundrobin/leastactive/consistenthash
    cluster: failover      # 集群容错策略：failover/failfast/failsafe/failback/forking/broadcast
    executes: 200          # 服务端最大并发执行数，防止过载
    threads: 200           # 业务线程池大小
    weight: 100            # 服务权重，配合加权负载均衡使用
```

### Step 3: 服务消费方引用并调用

```java
@Service
public class OrderService {
    
    @DubboReference(version = "1.0.0", group = "user")  // ⭐️ 引用远程服务
    private UserService userService;
    
    public Order createOrder(Long userId, String productName) {
        User user = userService.getUserById(userId); // 像本地方法一样调用
        if (user == null) {
            throw new BusinessException("用户不存在");
        }
        // 创建订单逻辑...
        return order;
    }
}
```

```yaml
# dubbo-consumer/application.yml
dubbo:
  application:
    name: order-service
  registry:
    address: nacos://127.0.0.1:8848
  consumer:
    timeout: 5000          # 客户端超时时间（建议 ≥ 服务端 timeout）
    retries: 2             # 重试次数，非幂等接口设为 0
    check: false           # 启动时不检查服务可用性，避免注册中心未就绪时启动失败
    loadbalance: leastactive  # 负载均衡策略：节点能力差异大时用 leastactive
    cluster: failfast      # 容错策略：下单等非幂等写操作用 failfast，避免重复执行
    lazy: true             # 延迟建立连接，服务数量多时减少启动耗时
    connections: 1         # 每个服务的独占连接数，高并发场景可设 1-3 提升吞吐
    mock: return null      # 服务不可用时的降级返回值，也可指定 Mock 类全限定名
```

> 💡 **核心体验**：消费方只需注入接口代理对象，调用方式与本地方法完全一致。网络通信、序列化、服务发现、负载均衡全部由框架自动完成。

---

## 二、常用配置速查

### 服务提供方核心配置

| 配置项 | 默认值 | 说明 | 推荐值 |
| :--- | :--- | :--- | :--- |
| `timeout` | 1000ms | 服务端超时时间 | 根据业务 RT 设置，通常 3-5s |
| `retries` | 2 | 重试次数（不含首次调用） | 幂等接口设 2-3，非幂等设 0 |
| `loadbalance` | random | 负载均衡策略 | 默认 random 即可，热点数据用 leastactive |
| `cluster` | failover | 集群容错策略 | 读操作 failover，写操作 failfast/failsafe |
| `executes` | 0（不限） | 服务端最大并发执行数 | 防止过载，建议设为线程池大小的 80% |
| `threads` | 200 | 业务线程池大小 | 根据 CPU 核数和 IO 密集度调整 |
| `dispatcher` | all | 线程模型 | 默认 all（所有消息派发到业务线程池） |

### 服务消费方核心配置

| 配置项 | 默认值 | 说明 | 推荐值 |
| :--- | :--- | :--- | :--- |
| `timeout` | 1000ms | 客户端超时时间 | **必须大于服务端 timeout**，避免服务端还在处理客户端已超时 |
| `retries` | 2 | 重试次数 | 同服务端，注意非幂等接口禁用 |
| `check` | true | 启动时检查服务可用性 | 开发环境设 false，生产环境视情况而定 |
| `lazy` | false | 延迟建立连接 | 服务数量多时设 true，减少启动耗时 |
| `connections` | 0（共享） | 每个服务的独占连接数 | 高并发场景可设 1-3 提升吞吐 |
| `mock` | - | 服务降级 Mock 实现 | 关键服务必须配置 fallback |

### 配置优先级（面试常考）

Dubbo 配置遵循 **"细粒度优先"** 原则，优先级从高到低：

```
方法级 > 接口级 > 全局级
消费者端 > 提供者端（同名配置冲突时）
JVM 参数 (-D) > XML/YAML 配置 > dubbo.properties
```

> ⚠️ **避坑提醒**：`timeout` 同时存在于 provider 和 consumer 端时，**consumer 端的值优先生效**。这是为了避免服务端处理时间超过客户端等待时间的尴尬场景。但最佳实践是两端都合理设置，且 consumer ≥ provider。

---

## 三、核心原理：一次 RPC 调用的完整流程

面试问到"Dubbo 原理"时，不需要背诵源码，但必须能清晰描述以下流程：

### 整体架构图

```
Consumer                          Registry                     Provider
   │                                  │                              │
   │ 1. 订阅服务                      │                              │
   ├─────────────────────────────────►│                              │
   │                                  │                              │
   │ 2. 推送服务地址列表                │                              │
   │◄─────────────────────────────────┤                              │
   │                                  │                              │
   │ 3. 发起 RPC 调用                  │                              │
   ├────────────────────────────────────────────────────────────────►│
   │                                  │                              │
   │ 4. 返回结果                       │                              │
   │◄────────────────────────────────────────────────────────────────┤
   │                                  │                              │
   │ 5. 服务变更通知                    │                              │
   │◄─────────────────────────────────┤                              │
```

### 详细流程拆解

#### 阶段一：服务注册与发现

1.  **Provider 启动**：将自身 IP:Port + 接口名 + 版本号 + 分组等信息注册到注册中心（如 Nacos/ZK）；
2.  **Consumer 启动**：向注册中心订阅所需服务的地址列表；
3.  **注册中心推送**：当 Provider 上下线或元数据变更时，注册中心实时推送最新地址列表给 Consumer；
4.  **本地缓存**：Consumer 将地址列表缓存在内存中，后续调用直接从本地选取，**不经过注册中心**。

> 🎯 **关键点**：注册中心只负责"服务发现"，**不参与实际 RPC 调用**。这就是为什么注册中心挂了，已建立的服务间通信不受影响（见下文详解）。

#### 阶段二：RPC 调用链路

```
Consumer 业务代码调用接口方法
    ↓
Dubbo 代理类（JDK/CGLIB 动态代理）
    ↓
Cluster 层：选择容错策略（Failover/Failfast/Failsafe...）
    ↓
LoadBalance 层：从本地缓存的地址列表中选择一个节点
    ↓
Protocol 层：封装请求（Dubbo 协议头 + Body），序列化参数
    ↓
Transport 层：通过 Netty 发送 TCP 请求到 Provider
    ↓
Provider Netty Server 接收请求 → 反序列化 → 路由到业务线程池
    ↓
执行业务方法 → 序列化结果 → 返回响应
    ↓
Consumer Netty Client 接收响应 → 反序列化 → 返回给业务代码
```

#### 阶段三：关键组件职责

| 组件 | 职责 | 面试关键词 |
| :--- | :--- | :--- |
| **ProxyFactory** | 生成消费方接口代理 / 提供方服务包装 | JDK Proxy / CGLIB |
| **Cluster** | 集群容错、负载均衡、路由 | Failover、Random、ConsistentHash |
| **Registry** | 服务注册、订阅、通知 | Nacos、ZK、本地缓存 |
| **Protocol** | 协议封装、序列化、通信 | Dubbo 协议、Hessian2、Protobuf |
| **Transport** | 网络传输 | Netty、长连接、心跳 |
| **Filter** | 调用链拦截（类似 Servlet Filter） | 日志、监控、鉴权、限流 |

### Dubbo 协议格式（加分项）

Dubbo 自定义二进制协议，比 HTTP 更紧凑高效：

```
+----------------+----------------+----------------+----------------+
| Magic (2B)     | Flag (1B)      | Status (1B)    | Request ID (8B)|
+----------------+----------------+----------------+----------------+
| Data Length (4B)                                                |
+----------------------------------------------------------------+
| Body (variable length)                                        |
| [ServiceName][MethodName][ParameterTypes][Arguments][Attachments]|
+----------------------------------------------------------------+
```

-   **Magic**: `0xdabb`，标识 Dubbo 协议包；
-   **Flag**: 标记请求/响应、单向/双向、事件/普通消息；
-   **Request ID**: 全局唯一，用于异步调用时匹配请求与响应；
-   **Body**: Hessian2 序列化的业务数据。

> 💡 **为什么不用 HTTP？** HTTP 头部冗余大（几百字节）、文本协议解析慢、无长连接复用（HTTP/1.1）。Dubbo 协议头仅 16 字节，二进制序列化，Netty 长连接复用，同等硬件下吞吐量可达 HTTP 的 3-5 倍。

---

## 四、注册中心挂了，服务间还能通信吗？

这是 Dubbo 面试**必考题**，答案是：**可以，但有条件**。

### 为什么还能通信？

Dubbo 的设计哲学是 **"注册中心不是单点"**：

1.  **本地缓存**：Consumer 启动时会从注册中心拉取全量地址列表，并持久化到本地文件（`~/.dubbo/dubbo-resolve-{interface}.cache`）；
2.  **内存常驻**：运行期间，地址列表始终保存在 Consumer 内存中，RPC 调用直接从内存选取节点，**不经过注册中心**；
3.  **优雅降级**：注册中心不可用时，Consumer 自动切换到本地缓存继续工作，仅打印 WARN 日志；
4.  **恢复同步**：注册中心恢复后，Consumer 重新订阅并同步最新地址列表。

### 什么情况下会受影响？

| 场景 | 是否受影响 | 原因 |
| :--- | :--- | :--- |
| 已有服务间的正常调用 | ❌ 不受影响 | 地址已在本地缓存 |
| 新服务上线/下线 | ⚠️ 暂时感知不到 | 无法收到变更通知，需等注册中心恢复 |
| 新 Consumer 首次启动 | ⚠️ 可能失败 | 若无本地缓存则无法获取地址列表 |
| 服务元数据变更（权重/分组） | ⚠️ 暂时不生效 | 同上 |

### 生产环境保障措施

-   **开启本地缓存**：默认已开启，确保 `dubbo.registry.use-as-config-center=true`；
-   **多注册中心**：配置多个注册中心（如 Nacos + ZK），互为备份；
-   **直连模式兜底**：紧急情况下可通过 `-Ddubbo.reference.{interface}=ip:port` 绕过注册中心直连；
-   **监控告警**：对注册中心健康状态做独立监控，及时发现并处理。

> 🎯 **面试话术**：Dubbo 采用"注册中心仅负责服务发现，不参与实际调用"的设计。Consumer 启动时拉取地址列表并缓存到本地文件和内存，RPC 调用直接从本地选取节点。因此注册中心宕机不影响已有服务间通信，仅影响新服务发现和变更通知。生产环境通过本地缓存、多注册中心、直连兜底三重保障确保高可用。

---

## 五、集群容错与负载均衡

### 六种容错策略

| 策略 | 行为 | 适用场景 |
| :--- | :--- | :--- |
| **Failover**（默认） | 失败自动切换其他节点，可配重试次数 | 读操作、幂等写操作 |
| **Failfast** | 失败立即报错，不重试 | 非幂等写操作（如下单） |
| **Failsafe** | 失败忽略异常，返回空结果 | 日志记录等非关键操作 |
| **Failback** | 失败记录请求，定时重发 | 消息通知、异步补偿 |
| **Forking** | 并行调用多个节点，任一成功即返回 | 实时性要求极高的读操作 |
| **Broadcast** | 广播调用所有节点，任一失败则整体失败 | 缓存刷新、配置同步 |

### 四种负载均衡策略

| 策略 | 算法 | 适用场景 |
| :--- | :--- | :--- |
| **Random**（默认） | 加权随机 | 通用场景 |
| **RoundRobin** | 加权轮询 | 节点性能均匀 |
| **LeastActive** | 最少活跃调用数 | 节点处理能力差异大 |
| **ConsistentHash** | 一致性哈希 | 有状态服务、缓存亲和 |

> 💡 **面试加分点**：Dubbo 的 Random 和 RoundRobin 都是**加权**的，权重可通过 `<dubbo:parameter key="weight" value="10"/>` 或注册中心元数据配置。性能好的机器设高权重，实现异构集群的合理负载。

---

> 📌 **超时控制、限流、熔断降级、服务雪崩防护**等内容已移至独立笔记 [服务雪崩防护：限流、熔断与降级](Service-Protection-Notes.md)，整合了 Dubbo 和 Spring Cloud（Sentinel）两套体系的完整内容。

---

## 七、SPI 扩展机制：Dubbo 的可插拔灵魂

Dubbo 几乎所有核心组件（Protocol、Registry、LoadBalance、Cluster、Filter 等）都通过 **SPI（Service Provider Interface）** 实现可插拔替换。这与 SpringBoot 的 SPI 思想一脉相承，但 Dubbo 做了增强：

### Dubbo SPI vs Java SPI

| 维度 | Java SPI | Dubbo SPI |
| :--- | :--- | :--- |
| 配置文件 | `META-INF/services/<接口>` | `META-INF/dubbo/<接口>` |
| 加载方式 | 一次性全量实例化 | 按需懒加载 + 缓存 |
| 扩展点命名 | 无名称，按顺序 | **Key-Value 命名**，可按名指定 |
| 自适应扩展 | 不支持 | `@Adaptive` 注解，运行时动态选择实现 |
| AOP 包装 | 不支持 | `@Wrapper` 注解，自动包装增强 |
| IOC 注入 | 不支持 | 支持 setter 注入其他扩展点 |

### `@Activate` 激活机制与按需加载流程

Dubbo SPI 不是启动时全量实例化所有扩展实现，而是**按需懒加载 + 条件激活**，核心流程如下：

1.  **扫描注册**：Dubbo 启动时通过 `ExtensionLoader` 扫描 `META-INF/dubbo/<接口>` 文件，将所有扩展实现的 Key-Value 映射缓存到内存中（此时**只记录类名，不实例化**）；
2.  **按需获取**：当业务代码调用 `ExtensionLoader.getExtension("traceIdFilter")` 或通过 `@Activate` 自动匹配时，才触发对应实现类的反射实例化；
3.  **条件过滤**：对于标注了 `@Activate` 的扩展（如 Filter），Dubbo 在获取激活列表时会检查注解上的 `group`（CONSUMER/PROVIDER）、`value`（URL 参数匹配）等条件，只有当前上下文满足条件的扩展才会被激活并加入调用链；
4.  **缓存复用**：实例化后的扩展对象会被缓存到 `ExtensionLoader` 内部的 Map 中，后续相同请求直接复用，不会重复创建。

> 💡 **一句话总结**：Dubbo SPI 的"按需"体现在两个层面——**实例化按需**（用到才创建，而非启动全量加载）和**激活按需**（`@Activate` 条件过滤，只有匹配的扩展才参与调用链）。这使得即使 classpath 下有几十个 Filter 实现，实际运行时也只有符合条件的那几个被加载和执行。

### 自定义扩展示例

```java
// 1. 定义扩展接口
@SPI("random")  // 默认实现
public interface LoadBalance {
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation);
}

// 2. 实现扩展
public class MyLoadBalance implements LoadBalance {
    @Override
    public <T> Invoker<T> select(...) { /* 自定义逻辑 */ }
}

// 3. 声明扩展（META-INF/dubbo/com.alibaba.dubbo.rpc.cluster.LoadBalance）
myLoadBalance=com.example.MyLoadBalance

// 4. 使用
@DubboReference(loadbalance = "myLoadBalance")
private UserService userService;
```

### 实战：Filter SPI 实现 TraceId 全链路透传

分布式系统中，保证调用链上下游 traceId 一致是可观测性的基础。通过 Dubbo Filter SPI 可以零侵入地实现这一需求。

#### Step 1: 定义 Filter 实现

```java
@Activate(group = {CommonConstants.CONSUMER, CommonConstants.PROVIDER})
public class TraceIdFilter implements Filter {

    private static final String TRACE_ID_KEY = "traceId";

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        // Consumer 端：从 MDC 取出 traceId，放入 RPC Attachment 传递出去
        if (RpcContext.getContext().isConsumerSide()) {
            String traceId = MDC.get(TRACE_ID_KEY);
            if (traceId != null) {
                RpcContext.getClientAttachment().setAttachment(TRACE_ID_KEY, traceId);
            }
        }

        // Provider 端：从 RPC Attachment 取出 traceId，放入 MDC
        if (RpcContext.getContext().isProviderSide()) {
            String traceId = RpcContext.getServerAttachment().getAttachment(TRACE_ID_KEY);
            if (traceId != null) {
                MDC.put(TRACE_ID_KEY, traceId);
            }
        }

        try {
            return invoker.invoke(invocation);
        } finally {
            // ⭐️ Provider 端必须清理 MDC，防止线程池复用导致上下文污染
            if (RpcContext.getContext().isProviderSide()) {
                MDC.remove(TRACE_ID_KEY);
            }
        }
    }
}
```

> ⚠️ **Dubbo 3.x API 变更**：`RpcContext.getContext().setAttachment()` 已废弃，Consumer 端设置用 `RpcContext.getClientAttachment()`，Provider 端读取用 `RpcContext.getServerAttachment()`。

#### Step 2: 声明 SPI 扩展

创建文件 `META-INF/dubbo/org.apache.dubbo.rpc.Filter`，内容：

```
traceIdFilter=com.example.filter.TraceIdFilter
```

#### Step 3: 激活方式

`@Activate(group = {CONSUMER, PROVIDER})` 注解使 Filter 在消费端和提供端都自动生效，无需额外配置。如需手动指定：

```yaml
dubbo:
  consumer:
    filter: traceIdFilter
  provider:
    filter: traceIdFilter
```

#### 核心原理

```
Consumer 业务线程
    │ MDC.get("traceId") → "abc-123"
    ↓
TraceIdFilter [Consumer 侧]
    │ setAttachment("traceId", "abc-123")
    ↓
Dubbo Protocol 序列化（traceId 作为 Attachment 写入请求体）
    ↓ TCP 传输
Provider Netty IO 线程 → 业务线程池
    ↓
TraceIdFilter [Provider 侧]
    │ getAttachment("traceId") → "abc-123"
    │ MDC.put("traceId", "abc-123")
    ↓
Provider 业务逻辑（日志 %X{traceId} 自动携带）
    ↓
finally: MDC.remove("traceId")  ← ⭐️ 必须清理！
```

#### 关键注意事项

| 要点 | 说明 |
| :--- | :--- |
| **MDC 必须清理** | Provider 端业务线程池是复用的，不清理会导致下一个请求拿到脏 traceId |
| **用 Attachment 传递** | Dubbo Attachment 是框架级元数据通道，随协议序列化传输，比自定义 Header 更可靠 |
| **@Activate 双端激活** | group 必须同时包含 CONSUMER 和 PROVIDER，否则只在一端生效 |
| **日志框架自动集成** | logback/log4j2 的 `%X{traceId}` 自动从 MDC 取值，无需改日志配置 |
| **异步调用场景** | Consumer 使用 CompletableFuture 时，需在回调中手动恢复 MDC |

> 🎯 **面试话术**：Dubbo SPI 是对 Java SPI 的增强，支持 Key-Value 命名、按需加载、@Adaptive 自适应选择、@Wrapper AOP 包装和 IOC 注入。实际项目中，我通过 Filter SPI 实现了 traceId 全链路透传：Consumer 端从 MDC 取 traceId 放入 RpcContext Attachment，Provider 端从 Attachment 取出写入 MDC，调用完成后 finally 清理防止线程池污染。整个过程对业务代码零侵入，体现了 Dubbo SPI 的可插拔设计哲学。

---

## 面试高频问题速查

### Q1: Dubbo 的整体架构和工作原理是什么？
**答**：Dubbo 是高性能 RPC 框架，核心流程分三阶段：① **服务注册发现**：Provider 注册到注册中心，Consumer 订阅并缓存地址列表到本地；② **RPC 调用**：Consumer 通过代理类 → Cluster 容错 → LoadBalance 选节点 → Protocol 序列化 → Netty 发送请求 → Provider 执行并返回；③ **变更通知**：注册中心推送地址变更，Consumer 更新本地缓存。关键设计：注册中心不参与实际调用，仅负责服务发现。

### Q2: 注册中心挂了，服务间还能通信吗？为什么？
**答**：可以。Dubbo 采用"注册中心仅负责服务发现"的设计，Consumer 启动时拉取地址列表并缓存到本地文件和内存，RPC 调用直接从本地选取节点，不经过注册中心。因此注册中心宕机不影响已有服务间通信，仅影响新服务发现和变更通知。生产环境通过本地缓存、多注册中心、直连兜底三重保障确保高可用。

### Q3: Dubbo 有哪些集群容错策略？分别适用于什么场景？
**答**：六种策略：Failover（默认，失败重试，适合读/幂等写）、Failfast（失败立即报错，适合非幂等写）、Failsafe（失败忽略，适合日志等非关键操作）、Failback（失败记录定时重发，适合消息通知）、Forking（并行调用任一成功即返回，适合高实时读）、Broadcast（广播所有节点，适合缓存刷新）。默认 Failover + 2 次重试。

### Q4: Dubbo 的负载均衡策略有哪些？如何选择？
**答**：四种策略：Random（加权随机，默认通用）、RoundRobin（加权轮询，节点均匀）、LeastActive（最少活跃数，节点能力差异大）、ConsistentHash（一致性哈希，有状态/缓存亲和）。Dubbo 的 Random/RoundRobin 均支持权重配置，可根据机器性能差异化分配流量。

### Q5: Dubbo SPI 和 Java SPI 有什么区别？
**答**：Dubbo SPI 是对 Java SPI 的增强：① Key-Value 命名，可按名指定实现；② 按需懒加载 + 缓存，避免全量实例化；③ @Adaptive 自适应扩展，运行时根据 URL 参数动态选择实现；④ @Wrapper AOP 包装，自动增强；⑤ 支持 setter IOC 注入其他扩展点。Java SPI 仅支持全量加载和无序遍历。

### Q6: Dubbo 为什么比 HTTP 快？
**答**：三个原因：① **协议紧凑**：Dubbo 协议头仅 16 字节，HTTP 头部数百字节；② **二进制序列化**：默认 Hessian2，比 JSON 体积小 3-10 倍、解析快 5-10 倍；③ **长连接复用**：基于 Netty 的 TCP 长连接 + Request ID 多路复用，避免 HTTP/1.1 的频繁建连开销和队头阻塞。同等硬件下吞吐量可达 HTTP 的 3-5 倍。

> 💡 **深入理解：TCP 连接复用 ≠ 不会过载**
>
> TCP 连接复用解决的是**网络传输层**的问题（不用反复三次握手），但请求到达 Provider 后仍需由**业务线程池**执行。两者是串行的两个阶段：
>
> ```
> Consumer ──TCP长连接(复用)──► Provider Netty IO线程 ──派发──► 业务线程池 ──执行──► 业务逻辑
>          ↑ 这一步很快              ↑ 这一步是瓶颈
>       (网络层已优化)           (计算/IO资源有限)
> ```
>
> 即使只有一条 TCP 连接，以下场景仍会导致线程池耗尽：
> -   **请求速率 > 处理能力**：多路复用意味着没有连接数限制，Consumer 可以疯狂发请求，超过线程池容量就会排队/拒绝；
> -   **慢请求占住线程**：某个接口响应慢，200 个线程全被占住，后续快请求也进不来；
> -   **突发流量**：平时 QPS 100，促销时突然 2000 QPS，线程池瞬间打满；
> -   **下游级联阻塞**：Provider A 调 B，B 慢了导致 A 的线程全部阻塞等待，A 线程池耗尽 → 上游连锁崩溃（雪崩）。
>
> **一句话总结**：TCP 连接复用让"请求能更快到达 Provider"，但也正因为到达得太快，如果处理能力跟不上，反而更容易把线程池打满。**连接复用降低了网络瓶颈，但把压力转移到了业务处理层**。因此生产环境必须配合 `executes` 限流、线程池隔离、熔断降级等多层防护。

### Q7: Dubbo 的超时时间如何设置？provider 和 consumer 冲突时以谁为准？
**答**：timeout 可在 provider 和 consumer 两端分别设置。当同名配置冲突时，**consumer 端的值优先生效**。这是因为 RPC 调用的超时本质上是"调用方愿意等待多久"，应由调用方决定。但最佳实践是两端都合理设置，且 consumer ≥ provider，避免服务端还在处理时客户端已超时断开。

### Q8: Dubbo 如何实现优雅停机？
**答**：Dubbo 优雅停机分三步：① **注销服务**：从注册中心移除自身地址，停止接收新请求；② **等待存量请求**：等待已接收的请求处理完成（默认 10s，可通过 `shutdown.wait` 配置）；③ **关闭资源**：关闭线程池、网络连接、释放锁等。SpringBoot 集成时通过 `ShutdownHookCallbacks` 自动触发，确保 K8s 滚动发布时零流量损失。

### Q9: Failfast 容错策略下配置了 retries=3，会重试吗？
**答**：**不会重试**。Failfast 策略的源码中没有循环重试逻辑，失败后立即抛出异常，`retries` 配置对其无效。只有 **Failover** 策略会使用 `retries` 参数进行失败重试。其他五种策略（Failfast/Failsafe/Failback/Forking/Broadcast）各有自己的失败处理逻辑，均不受 `retries` 影响。"快速失败 + 重试"本身是矛盾的——要么选 Failover + retries 实现重试，要么选 Failfast 实现快速失败，二者不可兼得。

---

## 小结

Dubbo 的核心价值在于 **"RPC 透明化 + 服务治理自动化 + 极致性能"**：
-   **透明化**：远程调用像本地方法一样简单，屏蔽网络、序列化、服务发现等底层细节；
-   **自动化**：注册中心驱动的服务发现、多种容错策略、负载均衡、优雅停机开箱即用；
-   **高性能**：自定义二进制协议 + Hessian2 序列化 + Netty 长连接，吞吐量远超 HTTP；
-   **可扩展**：Dubbo SPI 机制让所有核心组件均可插拔替换，生态繁荣。

掌握这四个支柱，无论是日常微服务开发还是分布式系统设计，都能做到心中有数。

> 📌 **记忆口诀**：**注册只管发现、调用走本地缓存、容错六策略、均衡四算法、SPI 可插拔、协议快过 HTTP**。
