# Spring Cloud 微服务生态核心要点

## 技术基线：Spring Cloud Alibaba

Spring Cloud 经历了多个版本迭代，不同体系的组件命运各异。**本笔记基于国内主流的 Spring Cloud Alibaba 套件编写**，以下是三大体系的现状：

| 体系 | 核心组件 | 现状 | 说明 |
| :--- | :--- | :--- | :--- |
| **Spring Cloud Netflix** | Eureka、Ribbon、Hystrix、Zuul | ❌ 全部停止维护 | 新项目禁用，老项目逐步迁移 |
| **Spring Cloud 官方** | Gateway、LoadBalancer、Config、Sleuth | ⚠️ 部分使用 | Gateway 和 LoadBalancer 仍是主流，Config 被 Nacos 替代，Sleuth 被 Micrometer Tracing 替代 |
| **Spring Cloud Alibaba** | Nacos、Sentinel、RocketMQ、Seata、Dubbo | ✅ **国内主流** | 阿里开源，与 Spring Cloud 标准无缝集成，社区活跃、文档完善 |

### 本笔记的技术选型

| 能力 | 采用方案 | 所属体系 |
| :--- | :--- | :--- |
| 注册中心 / 配置中心 | Nacos | Spring Cloud Alibaba |
| 限流熔断 | Sentinel | Spring Cloud Alibaba |
| 网关 | Spring Cloud Gateway | Spring Cloud 官方（Alibaba 兼容） |
| 负载均衡 | Spring Cloud LoadBalancer | Spring Cloud 官方（替代 Ribbon） |
| 服务调用 | OpenFeign | Spring Cloud 官方（Alibaba 兼容） |
| 分布式事务 | Seata | Spring Cloud Alibaba |
| 链路追踪 | Micrometer Tracing | Spring Cloud 官方 |

> 💡 **面试话术**："我们基于 Spring Cloud Alibaba 套件，注册中心和配置中心用 Nacos，限流熔断用 Sentinel，网关用 Spring Cloud Gateway，服务调用用 OpenFeign，分布式事务用 Seata。这套组合在国内生态最成熟，文档和社区支持也最好。"

---

## 从"Dubbo 之外的另一条路"说起

如果你已经熟悉了 Dubbo 的 RPC 调用模式，那么理解 Spring Cloud 会非常快——它们解决的是同一类问题（服务发现、负载均衡、容错、网关），只是技术选型和设计哲学不同：

| 维度 | Dubbo | Spring Cloud (Alibaba) |
| :--- | :--- | :--- |
| 通信协议 | 自定义二进制协议（TCP） | HTTP/REST（也可集成 Dubbo/gRPC） |
| 服务调用方式 | RPC 接口代理 | HTTP 客户端（OpenFeign / WebClient） |
| 注册中心 | ZooKeeper / Nacos | Nacos（主流） |
| 网关 | Dubbo Gateway（较新） | Spring Cloud Gateway（主流） |
| 限流熔断 | Sentinel | Sentinel |
| 配置中心 | Apollo / Nacos | Nacos（主流） |
| 生态定位 | 高性能 RPC 框架 | 微服务全家桶解决方案 |

> 💡 **选型建议**：内部系统、高性能要求选 Dubbo；对外 API、前后端分离、团队熟悉 Spring 生态选 Spring Cloud Alibaba。两者也可以共存（通过 Dubbo-Spring-Cloud 桥接）。

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

> ⚠️ **OpenFeign 底层 HTTP 客户端澄清**：OpenFeign 默认底层是 **`java.net.HttpURLConnection`**（JDK 原生），**不是 RestTemplate**。RestTemplate 是 Spring 提供的另一个独立 HTTP 客户端，与 Feign 无继承或依赖关系。如需更高性能，可引入 `feign-okhttp` 或 `feign-httpclient` 替换默认的 HttpURLConnection。
> 📌 **与 Dubbo 的核心区别**：OpenFeign 底层走 HTTP，序列化用 JSON，性能低于 Dubbo 的二进制协议，但与前端/第三方对接更友好。详细原理见 [OpenFeign 技术原理](OpenFeign-Notes.md)。
> 💡 **误区澄清**：不是一定要用 OpenFeign 才能发起微服务调用。使用 `RestTemplate` + `@LoadBalanced` 同样可以发起带服务发现和负载均衡的微服务调用，只是写法更繁琐、缺少声明式接口的简洁性。

```java
// 1. 注册一个带负载均衡能力的 RestTemplate Bean
@Bean
@LoadBalanced  // ⭐️ 关键注解：使 RestTemplate 支持 lb:// 服务名解析
public RestTemplate loadBalancedRestTemplate() {
    return new RestTemplate();
}

// 2. 注入使用，通过服务名（而非 IP:Port）发起调用
@Service
public class OrderService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    public Order createOrder(Long userId) {
        // lb://user-service 会被 LoadBalancer 解析为具体实例地址
        User user = restTemplate.getForObject(
            "http://user-service/users/{id}", User.class, userId
        );
        
        // POST 请求示例
        ResponseEntity<Boolean> response = restTemplate.postForEntity(
            "http://user-service/users", newUser, Boolean.class
        );
        return orderRepository.save(toEntity(user));
    }
}
```
> ⚠️ **RestTemplate vs OpenFeign 选型建议**：
> -   **OpenFeign**（推荐）：声明式接口、代码简洁、与 Spring Cloud 生态深度集成（Fallback、重试、超时配置统一）；
> -   **RestTemplate**：适合简单场景或遗留项目，Spring 官方已进入维护模式，新项目推荐使用 WebClient（响应式）或 OpenFeign。

### 服务发现与负载均衡

#### 1. 负载均衡和 @LoadBalanced 注解是什么关系？

`@LoadBalanced` 是一个**标记注解**，它本身不包含任何负载均衡逻辑。它的作用是告诉 Spring Cloud："这个 RestTemplate/WebClient Bean 需要被增强，使其支持 `lb://服务名` 的 URL 解析"。

```java
@Bean
@LoadBalanced  // ⭐️ 仅作为标记，触发自动配置
public RestTemplate restTemplate() {
    return new RestTemplate();
}
```

**底层原理**：Spring Cloud 通过 `LoadBalancerInterceptor`（RestTemplate）或 `ReactorLoadBalancerExchangeFilterFunction`（WebClient）拦截请求，将 `lb://user-service/users/1` 中的 `user-service` 替换为从注册中心获取的具体实例地址（如 `192.168.1.100:8080`），再发起真正的 HTTP 调用。**实际的实例选择逻辑由 `ReactorServiceInstanceLoadBalancer` 实现**，`@LoadBalanced` 只是"开关"。

> 💡 **一句话总结**：`@LoadBalanced` = "启用服务名解析"，`ReactorServiceInstanceLoadBalancer` = "决定选哪个实例"。两者配合才构成完整的客户端负载均衡。

#### 2. OpenFeign 如何实现多实例负载均衡？

OpenFeign **不需要** `@LoadBalanced` 注解。当你使用 `@FeignClient(name = "user-service")` 时，Spring Cloud OpenFeign 自动集成了 Spring Cloud LoadBalancer：

```java
// Feign 客户端声明服务名，框架自动完成：服务发现 + 负载均衡 + HTTP 调用
@FeignClient(name = "user-service")
public interface UserClient {
    @GetMapping("/users/{id}")
    User getUser(@PathVariable("id") Long userId);
}
```

**调用链路**：
```
userClient.getUser(1L)
    ↓
Feign 代理类拦截方法调用
    ↓
ReactorLoadBalancerClient 根据 "user-service" 查询 Nacos 实例列表
    ↓
ReactorServiceInstanceLoadBalancer.choose() 选择一个实例（默认 RoundRobin）
    ↓
拼接完整 URL：http://192.168.1.100:8080/users/1
    ↓
HttpURLConnection / OkHttp 发起 HTTP 请求
```

> ⚠️ **关键区别**：RestTemplate 需要手动加 `@LoadBalanced` 才能用服务名；OpenFeign 天然集成 LoadBalancer，只需声明服务名即可。**这也是为什么新项目推荐 OpenFeign 而非 RestTemplate 的原因之一**。

#### 3. Gateway 已经能负载均衡了，为什么还需要 LoadBalancer？

这是一个非常好的问题。答案是：**Gateway 和 LoadBalancer 解决的是不同层面的问题，它们不是替代关系，而是协作关系**。

| 维度 | Spring Cloud Gateway | Spring Cloud LoadBalancer |
| :--- | :--- | :--- |
| **作用位置** | 系统入口（外部请求 → 微服务） | 服务间调用（微服务 A → 微服务 B） |
| **使用者** | 前端/移动端/第三方 | 后端微服务内部 |
| **协议** | HTTP/WebSocket/gRPC | HTTP（Feign/WebClient/RestTemplate） |
| **是否依赖对方** | ✅ Gateway 内部使用 LoadBalancer 选择实例 | ❌ 独立工作 |

**关键点**：Gateway 的路由配置中 `uri: lb://user-service` 里的 `lb://` 正是由 Spring Cloud LoadBalancer 解析的。**Gateway 本身就是 LoadBalancer 的一个消费者**。如果没有 LoadBalancer，Gateway 也无法实现基于服务名的动态路由。

```
外部请求 → Gateway（路由匹配 + Filter）→ LoadBalancer（选实例）→ user-service 实例A
                                                    ↑
                                          Gateway 内部也在用 LoadBalancer

order-service → OpenFeign → LoadBalancer（选实例）→ user-service 实例B
                                    ↑
                          服务间调用也在用 LoadBalancer
```

> 💡 **一句话总结**：LoadBalancer 是 Spring Cloud 中**所有基于服务名调用的基础设施**，Gateway 和服务间调用都依赖它。Gateway 解决的是"外部流量怎么进来"，LoadBalancer 解决的是"知道服务名后怎么选实例"。

#### 4. 自定义负载均衡器示例

Spring Cloud LoadBalancer 默认使用 RoundRobin 策略。如需自定义（如按用户 ID 一致性 Hash、按权重、按机房就近），需实现 `ReactorServiceInstanceLoadBalancer` 接口并通过 `@LoadBalancerClient` 绑定。

```java
/**
 * 按用户 ID 取模的负载均衡器
 * 同一用户的请求始终路由到同一实例，适合有状态服务/缓存亲和性场景
 */
public class UserIdHashLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private final String serviceId;
    private final ObjectProvider<ServiceInstanceListSupplier> supplierProvider;

    public UserIdHashLoadBalancer(
            ObjectProvider<ServiceInstanceListSupplier> supplierProvider, String serviceId) {
        this.supplierProvider = supplierProvider;
        this.serviceId = serviceId;
    }

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier =
                supplierProvider.getIfAvailable(NoopServiceInstanceListSupplier::new);
        return supplier.get(request).next().map(instances -> {
            if (instances.isEmpty()) {
                return new EmptyResponse();
            }
            // 从请求中提取用户 ID
            String userId = extractUserId(request);
            if (userId == null || userId.isBlank()) {
                // 无用户标识时降级为随机
                return new DefaultResponse(
                        instances.get(ThreadLocalRandom.current().nextInt(instances.size())));
            }
            // 取模映射到固定实例
            int index = Math.abs(userId.hashCode()) % instances.size();
            return new DefaultResponse(instances.get(index));
        });
    }

    private String extractUserId(Request request) {
        // 从 HTTP Header 提取（Gateway 透传 / Feign 拦截器注入）
        if (request instanceof RequestDataContext requestDataContext) {
            RequestData requestData = requestDataContext.getClientRequest();
            if (requestData != null) {
                return requestData.getHeaders().getFirst("X-User-Id");
            }
        }
        return null;
    }
}
```

```java
/**
 * 将自定义负载均衡器绑定到指定服务
 * @LoadBalancerClient(name = "user-service", configuration = UserLbConfig.class) 的作用是：为 user-service 这个服务名绑定一个自定义的 ReactorServiceInstanceLoadBalancer Bean。
 * 这样无论是 openFeign 还是 restTemplate 发起对 user-service provider 的调用时，都会应用这个负载均衡策略，没有单独配置的服务会走全局默认的 RoundRobin 策略。
 */
@Configuration
@LoadBalancerClient(name = "user-service", configuration = UserLbConfig.class)
public class UserLbConfig {

    @Bean
    public ReactorServiceInstanceLoadBalancer userIdHashLoadBalancer(
            ObjectProvider<ServiceInstanceListSupplier> supplierProvider) {
        return new UserIdHashLoadBalancer(supplierProvider, "user-service");
    }
}
```

> 💡 **生产增强建议**：简单取模在实例增减时会导致大量用户重新映射。生产环境应使用**一致性 Hash 环 + 虚拟节点**（如 Guava `ConsistentHashing`），使实例变更时仅影响少量用户。同时可通过 Nacos metadata 标记实例版本/机房，配合自定义 LoadBalancer 实现灰度发布和就近路由。

### 服务注册：Provider 端配置

在 Spring Cloud 中，**服务注册和服务发现用的是同一套 `nacos.discovery` 配置**（不像 Dubbo 那样 provider/consumer 分开配置）。当一个服务引入 `spring-cloud-starter-alibaba-nacos-discovery` 依赖并启动时，它会**自动将自己注册到 Nacos**，无需额外代码。

```yaml
# Provider 端 application.yml
spring:
  application:
    name: user-service                     # ⭐️ 服务名 = 注册到 Nacos 的服务标识
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848       # Nacos 地址
        namespace: prod                    # 命名空间
        group: USER_GROUP                  # 分组
        ip: 192.168.1.100                  # 注册 IP（Docker/多网卡必配）
        port: 8080                         # 注册端口
        weight: 100                        # 权重（加权负载均衡用）
        metadata:                          # 自定义元数据
          version: v2
          region: cn-hangzhou
        ephemeral: true                    # 临时实例（默认 true，心跳检测）
        heart-beat-interval: 5000          # 心跳间隔（ms）
        heart-beat-timeout: 30000          # 心跳超时（ms），超时标记不健康
        register-enabled: true             # 是否注册到 Nacos（默认 true）
      username: nacos                      # ⭐️ Nacos 用户名（开启鉴权时必填，放在 nacos 节点下）
      password: nacos123                   # ⭐️ Nacos 密码（开启鉴权时必填，放在 nacos 节点下）
```

> ⚠️ **Nacos 鉴权说明**：Nacos 2.x 默认开启鉴权，生产环境**必须修改默认账号密码**。`username` / `password` 是 **`spring.cloud.nacos` 级别的全局配置**，注册、发现、配置中心共用，不要放在 `discovery` 或 `config` 子节点下。

#### Provider 端关键配置详解

| 配置项 | 说明 | 生产建议 |
| :--- | :--- | :--- |
| `spring.application.name` | 服务名，注册到 Nacos 的唯一标识 | **必须全局唯一**，建议用 `{业务}-{服务}` 命名 |
| `ip` | 注册到 Nacos 的 IP 地址 | Docker/K8s/多网卡环境**必须手动指定**，否则可能注册容器内网 IP 导致不可达 |
| `port` | 注册到 Nacos 的端口 | 与 `server.port` 一致即可，一般无需单独配 |
| `weight` | 服务权重（1-100） | 高性能机器设高权重，配合 LoadBalancer 加权策略 |
| `metadata` | 自定义 KV 元数据 | 灰度发布（version）、标签路由、监控打标 |
| `ephemeral` | 临时实例 vs 持久实例 | 业务服务用临时实例（默认），基础设施用持久实例 |
| `register-enabled` | 是否注册 | 本地开发调试时可设 false，避免污染线上注册表 |
| `heart-beat-*` | 心跳参数 | 默认值通常够用，网络不稳定时适当调大 timeout |

> 💡 **与 Dubbo 的区别**：Dubbo 的 provider/consumer 配置是分开的（`dubbo.provider` / `dubbo.consumer`），而 Spring Cloud 中服务注册和发现共用 `nacos.discovery` 配置。一个服务既可以是 Provider 也可以是 Consumer，取决于它是否暴露了接口以及是否注入了 FeignClient。

### Nacos 服务发现常用配置

以下是 Consumer 端（或同时作为 Provider + Consumer 的服务）最常用的服务发现配置项：

```yaml
spring:
  application:
    name: order-service                    # ⭐️ 服务名（必须唯一）
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848       # Nacos 服务端地址（集群用逗号分隔）
        namespace: dev                     # 命名空间隔离（dev/test/prod）
        group: DEFAULT_GROUP               # 分组（默认 DEFAULT_GROUP）
        cluster-name: hangzhou             # 集群名称（配合就近路由）
        ip: 192.168.1.100                  # 手动指定注册 IP（多网卡/Docker 场景）
        port: 8080                         # 手动指定注册端口
        weight: 100                        # 服务权重（1-100，配合加权负载均衡）
        metadata:                          # 自定义元数据（灰度/版本/机房等）
          version: v2
          env: production
          region: cn-hangzhou
        watch:
          enabled: true                    # 开启服务变更监听（默认 true）
        heart-beat-timeout: 30000          # 心跳超时时间（ms），超时标记不健康
        heart-beat-interval: 5000          # 心跳间隔（ms）
        fail-fast: false                   # 启动时 Nacos 不可用是否快速失败
      username: nacos                      # ⭐️ Nacos 用户名（开启鉴权时必填，放在 nacos 节点下）
      password: nacos123                   # ⭐️ Nacos 密码（开启鉴权时必填，放在 nacos 节点下）
```

> ⚠️ **Nacos 鉴权说明**：Nacos 2.x 默认开启鉴权，生产环境**必须修改默认账号密码**。`username` / `password` 是 **`spring.cloud.nacos` 级别的全局配置**，注册、发现、配置中心共用，不要放在 `discovery` 或 `config` 子节点下。

#### 关键配置详解

| 配置项 | 说明 | 生产建议 |
| :--- | :--- | :--- |
| `namespace` | 命名空间隔离，不同环境/租户互不可见 | **必须按环境隔离**，dev/test/prod 分开 |
| `group` | 服务分组，同一 namespace 下进一步细分 | 按业务线或项目分组，如 ORDER_GROUP |
| `cluster-name` | 集群标识，配合 Nacos 就近路由策略 | 多机房部署时按机房设置，减少跨机房调用 |
| `ip` / `port` | 手动指定注册地址 | Docker/K8s 环境下容器 IP 可能不通，需指定宿主机 IP |
| `weight` | 服务权重，LoadBalancer 加权策略使用 | 高性能机器设高权重，实现异构集群合理负载 |
| `metadata` | 自定义 KV 元数据 | 灰度发布（version=v2）、标签路由、监控打标 |
| `fail-fast` | 启动时 Nacos 不可用是否抛异常 | 开发设 true 快速发现问题，生产设 false 允许降级启动 |

#### Nacos 集群配置

```yaml
# Nacos 服务端集群地址（客户端配置）
spring:
  cloud:
    nacos:
      discovery:
        server-addr: nacos1:8848,nacos2:8848,nacos3:8848
```

> 💡 **面试加分点**：Nacos 支持**临时实例**（默认，心跳检测，宕机自动摘除）和**持久实例**（主动注册，不依赖心跳，适合 DNS/网关等基础设施服务）。通过 `ephemeral: false` 切换为持久实例。

### Nacos 动态配置中心

Nacos 同时充当配置中心，支持配置的**集中管理、动态推送、多环境隔离**

#### 基础接入

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

```yaml
# application.yml（✅ Spring Cloud Alibaba 2023.x 推荐方式）
spring:
  application:
    name: order-service
  config:
    import:                                # ⭐️ 通过 import 导入 Nacos 配置
      - nacos:order-service.yaml?refreshEnabled=true
      - nacos:common-db.yaml?group=COMMON_GROUP&refreshEnabled=true
      - nacos:common-redis.yaml?group=COMMON_GROUP&refreshEnabled=true
  cloud:
    nacos:
      server-addr: 127.0.0.1:8848         # Nacos 地址（serverAddr 放在 nacos 节点下）
      username: nacos                      # ⭐️ Nacos 用户名（开启鉴权时必填）
      password: nacos123                   # ⭐️ Nacos 密码（开启鉴权时必填）
      config:
        namespace: dev                     # 命名空间隔离
        group: DEFAULT_GROUP               # 分组
        file-extension: yaml               # 配置文件格式（yaml/properties/json）
```


#### 动态刷新：配置修改实时生效

```java
// 方式1：@RefreshScope + @Value（推荐简单场景）
@RestController
@RefreshScope  // ⭐️ 标注后，配置变更时 Bean 会自动重建
public class OrderController {
    
    @Value("${order.max-items:10}")
    private int maxItems;
    
    @GetMapping("/config")
    public Map<String, Object> getConfig() {
        return Map.of("maxItems", maxItems);
    }
}

// 方式2：@ConfigurationProperties（推荐复杂配置对象）
@Data
@Component
@ConfigurationProperties(prefix = "order")
@RefreshScope
public class OrderProperties {
    private int maxItems = 10;
    private String defaultWarehouse = "HZ-01";
    private List<String> blockedUsers = new ArrayList<>();
}
```

> 💡 **动态刷新的本质**：`@RefreshScope` 将 Bean 放入一个代理容器中，当收到 Nacos 配置变更事件时，销毁旧 Bean 并在下次访问时重新创建，从而注入新配置值。**不是热更新字段值，而是重建整个 Bean**。

#### 常用配置项速查

| 配置项 | 说明 | 生产建议 |
| :--- | :--- | :--- |
| `server-addr` | Nacos 服务端地址 | 集群用逗号分隔 |
| `namespace` | 命名空间 | **必须按环境隔离** |
| `group` | 配置分组 | 按业务线或项目分组 |
| `file-extension` | 文件格式 | 推荐 yaml，可读性好 |
| `shared-configs` | 共享配置 | DB/Redis/MQ 等公共配置抽取复用 |
| `extension-configs` | 扩展配置 | 当前服务专属的额外配置文件 |
| `refresh-enabled` | 是否开启动态刷新 | 默认 true，生产保持开启 |
| `timeout` | 获取配置超时时间 | 默认 3000ms，网络差可适当调大 |

#### 配置变更监听（编程式）

除了注解式自动刷新，还可以编程式监听配置变更：

```java
@Component
public class ConfigChangeListener {
    
    @Autowired
    private NacosConfigManager nacosConfigManager;
    
    @PostConstruct
    public void init() throws NacosException {
        nacosConfigManager.getConfigService().addListener(
            "order-service.yaml", "DEFAULT_GROUP",
            new Listener() {
                @Override
                public Executor getExecutor() { return null; }
                
                @Override
                public void receiveConfigInfo(String configInfo) {
                    log.info("Config changed: {}", configInfo);
                    // 自定义处理逻辑：刷新缓存、通知下游等
                }
            }
        );
    }
}
```

> ⚠️ **经典面试题：Nacos 远程配置 vs 本地 application.yaml，以哪个为准？**
> **答**：**Nacos 远程配置优先级更高**。Spring Cloud 的配置加载顺序为：Nacos 远程配置 > 本地 `application-{profile}.yaml` > 本地 `application.yaml`。当同一个配置项在 Nacos 和本地文件中都存在时，**以 Nacos 的值为准**。这也是配置中心的核心价值——通过远程配置覆盖本地默认值，实现多环境统一管理和动态调整。
> 🎯 **面试话术**：Nacos 配置中心支持配置集中管理、多环境命名空间隔离、动态推送实时生效。通过 `@RefreshScope` + `@Value` / `@ConfigurationProperties` 实现配置变更后 Bean 自动重建。共享配置通过 `shared-configs` 抽取复用，避免重复维护。相比 Spring Cloud Config，Nacos 自带 Dashboard、支持多种文件格式、无需额外 Git 仓库，是国内微服务项目的首选配置中心。

## 源码原理：服务注册与发现的自动装配过程

Spring Cloud 的服务注册与发现能力**本质上完全由 Starter 的自动装配机制实现**，业务代码零侵入。理解这个过程，就能明白为什么引入一个依赖、写几行配置就能拥有完整的微服务能力。

### 核心 Starter 与自动配置类

以 Nacos 为例，`spring-cloud-starter-alibaba-nacos-discovery` 内部包含两个关键模块：

| 模块 | 作用 | 核心自动配置类 |
| :--- | :--- | :--- |
| `spring-cloud-alibaba-nacos-discovery` | 服务注册与发现 | `NacosDiscoveryAutoConfiguration` |
| `spring-cloud-commons` | 抽象接口定义 | `LoadBalancerAutoConfiguration` / `ServiceRegistryAutoConfiguration` |

### 服务注册的自动装配流程

```
应用启动
    ↓
Spring Boot 加载 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
    ↓
触发 NacosDiscoveryAutoConfiguration（@ConditionalOnProperty 匹配 nacos.discovery.enabled）
    ↓
创建 NacosServiceRegistry Bean（实现 Spring Cloud 的 ServiceRegistry 接口）
    ↓
触发 ServiceRegistryAutoConfiguration → 注册 Lifecycle Bean
    ↓
SmartLifecycle.start() 被调用 → NacosServiceRegistry.register()
    ↓
通过 Nacos OpenAPI 向 Nacos Server 发送注册请求（POST /nacos/v1/ns/instance）
    ↓
注册成功，同时启动心跳线程维持实例状态
```

**关键点**：整个注册过程由 `SmartLifecycle` 驱动，在 Spring 容器刷新完成后自动执行，**无需任何手动调用**。这就是为什么你只需要加依赖和配置，服务就会自动注册到 Nacos。

### 服务发现的自动装配流程

```
@FeignClient(name = "user-service") 或 lb://user-service
    ↓
Spring Cloud LoadBalancer 拦截请求
    ↓
ReactiveDiscoveryClient.getInstances("user-service")
    ↓
NacosDiscoveryClient（实现 DiscoveryClient 接口）查询 Nacos
    ↓
Nacos OpenAPI: GET /nacos/v1/ns/instance/list?serviceName=user-service
    ↓
返回健康实例列表 → LoadBalancer 选择实例 → 拼接完整 URL → 发起 HTTP 调用
```

**关键点**：`NacosDiscoveryClient` 是 Spring Cloud `DiscoveryClient` 接口的 Nacos 实现，由 `NacosDiscoveryAutoConfiguration` 自动注册为 Bean。**所有基于服务名的调用（Feign、RestTemplate、Gateway）都通过这个统一接口获取实例列表**，切换注册中心只需更换 Starter，业务代码无需改动。

### 自动装配的设计精髓

```
┌─────────────────────────────────────────────────────┐
│              Spring Cloud Commons                    │
│   ServiceRegistry / DiscoveryClient / LoadBalancer   │
│              （纯接口，不含实现）                       │
└──────────────────────┬──────────────────────────────┘
                       │ 实现
          ┌────────────┼────────────┐
          ▼            ▼            ▼
   Nacos Starter   Eureka Starter  Consul Starter
   (自动装配)      (自动装配)       (自动装配)
```

> 💡 **一句话总结**：Spring Cloud 通过 **Commons 定义接口 + Starter 提供实现 + AutoConfiguration 自动装配** 三层架构，实现了注册中心的可插拔设计。业务开发者只需关注"用什么注册中心"，无需关心"怎么注册、怎么发现"——这些全部由 Starter 的自动装配在启动时完成。这也是 Spring Cloud 相比 Dubbo 在易用性上的核心优势之一。
