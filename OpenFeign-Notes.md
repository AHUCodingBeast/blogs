# OpenFeign 远程调用原理与用法笔记

> 这份笔记围绕一个开放性技术问题展开：**假如你在一个后端服务里，要对接十几个甚至几十个外部第三方 HTTP 接口（各家协议、鉴权方式、超时要求都不一样），你会怎么组织代码？**  
> 以这个问题为线索，把 OpenFeign 的两种调用模式、`configuration` 配置隔离、动态代理原理、重试超时、熔断降级串起来。

---

## 引子：多供应商 HTTP 对接的代码组织难题

假设我们在做一个数据聚合服务，需要对接大量第三方 HTTP 接口：

- **协议不一**：有的供应商是普通 JSON HTTP，有的是 SOAP/XML WebService。
- **鉴权不一**：有的用 token 请求头，有的用 AK/SK 签名，有的用证书。
- **超时要求不一**：有的查询很快（几秒），有的很慢（几十秒）。
- **重试策略不一**：有的接口幂等可以重试，有的不能重试（可能造成重复扣费）。

如果用 `RestTemplate` 硬写，会变成什么样？

```java
// 供应商A：JSON + token，自己拼URL、设header、序列化、解析
HttpHeaders headers = new HttpHeaders();
headers.set("token", configA.getToken());
HttpEntity<ReqA> entity = new HttpEntity<>(req, headers);
ResponseEntity<RespA> resp = restTemplate.exchange(hostA + "/api/xxx", POST, entity, RespA.class);

// 供应商B：SOAP，要自己拼SOAP信封、解析SOAP响应
String soap = buildSoapEnvelope(req);
ResponseEntity<String> resp = restTemplate.exchange(hostB + "/ws", POST, new HttpEntity<>(soap), String.class);
RespB result = parseSoap(resp.getBody());
// ... 几十个供应商，每个都重复这套样板
```

问题很明显：每个供应商都要手写拼 URL、设 header、序列化、解析响应，调用处充满样板代码，加一个供应商抄一遍、改一个供应商到处改。

OpenFeign 给了我们一个更优雅的组织方式：**把每个供应商声明成一个 interface + 注解 + 私有配置类，调用方法像调本地方法一样**。下面我们就把它拆透。

---

## 一、OpenFeign 是什么

**一句话**：声明式 HTTP 客户端。你只写一个 interface + 注解，它用动态代理生成实现类，自动帮你发 HTTP 请求、序列化、反序列化。

本质上就是个"更好用的 RestTemplate"——把拼 URL、设 header、序列化、解析响应这些样板代码用注解声明消除掉。底层仍是 HTTP 客户端（默认 JDK HttpURLConnection，可换 OkHttp/Apache HttpClient）。

---

## 二、两种使用模式

### 2.1 模式一：直连固定 URL（对接外部第三方 API 常用）

```java
@FeignClient(name = "supplierA", configuration = SupplierAFeignConfig.class, url = "${supplierA.host}")
public interface SupplierAApi {
    @PostMapping(path = "/api/query")
    SupplierAResponse query(@RequestBody SupplierARequest requestDTO);
}
```
- `url = "${supplierA.host}"` 指定直连地址 → **不走注册中心服务发现**，直接打这个 URL
- `name="supplierA"` 只是逻辑标识（用于日志/metrics/配置区分），不参与寻址
- 等价于 RestTemplate：`restTemplate.postForObject(host + "/api/query", req, Resp.class)`
- 多个供应商各写一个这样的 interface，生产连真实供应商域名，非生产连 mock 服务的本地地址

**适用场景**：对接外部第三方 HTTP 接口（供应商 API），不属于自己的微服务体系。这也是引子里那"几十个供应商"最合适的组织方式。

### 2.2 模式二：服务发现 + 负载均衡（微服务内部调用常用）

```java
@FeignClient(name = "order-service")   // 注意：没有 url
public interface OrderApi {
    @GetMapping("/orders/{id}")
    Order getOrder(@PathVariable("id") Long id);
}
```
- **没有 `url`** → Feign 用 `name="order-service"` 去**注册中心**（Nacos/Eureka）查所有实例
- 结合 Ribbon/Spring Cloud LoadBalancer 做**客户端负载均衡**（轮询/随机等选一个实例）
- 调用方不写死 IP，服务提供方扩缩容自动感知

**适用场景**：微服务集群内部互调，服务提供者在注册中心注册了多个实例。

### 2.3 两种模式对比

| 对比 | 模式一（直连） | 模式二（服务发现） |
|------|--------------|-------------------|
| `url` 参数 | 有，指定固定地址 | 无 |
| `name` 作用 | 逻辑标识 | 服务名，用于注册中心寻址 |
| 是否用注册中心 | 否 | 是（Nacos/Eureka） |
| 负载均衡 | 无（单点） | 有（Ribbon/LoadBalancer） |
| 典型场景 | 调外部第三方 API | 微服务内部互调 |

回到引子：对接外部供应商，因为它们不在你的注册中心里，所以**基本都用模式一直连**；只有调用自己体系内的微服务，才用模式二走服务发现。

---

## 三、`@FeignClient` 注解核心属性

| 属性 | 作用 | 示例 |
|------|------|-----------|
| `name`/`value` | 客户端名称（服务发现时是服务名，直连时是逻辑标识） | `"supplierA"` |
| `url` | 直连地址。不填走服务发现，填了直连 | `"${supplierA.host}"` |
| `configuration` | 私有配置类（超时/重试/拦截器/编解码），只作用于当前客户端 | `SupplierAFeignConfig.class` |
| `fallback`/`fallbackFactory` | 熔断降级类（配合 Hystrix/Sentinel） | 见第九章 |
| `path` | 统一前缀路径 | 未用时写在方法 @PostMapping 里 |
| `primary` | 是否设为 @Primary（多实例冲突时） | 默认 true |

---

## 四、`configuration` 配置类——核心机制

`@FeignClient(configuration = XxxFeignConfig.class)` 给这个客户端单独配"如何发请求"的细节。配置类里定义的 `@Bean` 只作用于当前 Feign（配置隔离）。

### 4.1 常用可配的 Bean

| Bean 类型 | 作用 | 典型用法 |
|-----------|------|-----------|
| `Request.Options` | 连接超时、读超时 | 慢接口配大超时，快接口配小超时 |
| `Retryer` | 重试策略 | 幂等接口可重试，非幂等 NEVER_RETRY |
| `RequestInterceptor` | 请求拦截器（加 header/加签/日志） | 加 token 头、加签名 |
| `Encoder` | 请求体序列化 | 自定义 SOAP Encoder 把对象编成 SOAP 信封 |
| `Decoder` | 响应体反序列化 | 用 JAXB 解 SOAP XML |
| `ErrorDecoder` | 错误响应处理 | 自定义错误转异常 |
| `Contract` | 注解契约（一般不动） | 默认 SpringMvcContract（支持 @PostMapping 等） |
| `Client` | 底层 HTTP 客户端 | 默认，可换 OkHttp/Apache |
| `Logger.Level` | 日志级别（NONE/BASIC/HEADERS/FULL） | 排错时打开 |

### 4.2 两种典型配置对比

这正是引子里"各家协议不一"的解法——每个供应商一个独立的 FeignConfig，互不干扰：

```java
// SupplierAFeignConfig —— 普通 JSON + token 鉴权
public class SupplierAFeignConfig {
    @Bean Request.Options options() { return new Request.Options(15, SECONDS, 65, SECONDS, true); }  // 慢接口，65s超时
    @Bean Retryer retryer() { return Retryer.NEVER_RETRY; }                          // 非幂等，不重试
    @Bean RequestInterceptor tokenInterceptor() {                                   // 加token头
        return template -> template.header("token", supplierAConfig.getToken());
    }
}

// SupplierBFeignConfig —— SOAP/XML WebService 协议
public class SupplierBFeignConfig {
    @Bean Encoder soapEncoder() { ... }   // 把对象编成 SOAP 信封 XML
    @Bean Decoder soapDecoder() { ... }    // 用 JAXB 解 SOAP 响应
    @Bean Retryer retryer() { return new Retryer.Default(); }  // 幂等，默认重试
}
```

同一个服务里，供应商A 走 JSON+token，供应商B 走 SOAP，各自一套 FeignConfig 互不干扰。这就是 `configuration` 的价值——**让"每个供应商各自一套 HTTP 调用细节"成为可能**，加新供应商就加一个 FeignConfig，不动别人的。

### 4.3 配置隔离的关键：configuration 类不标 @Configuration

这是 Feign 的核心机制，也是面试高频考点：

| 写法 | 类是否有 @Configuration | 行为 |
|------|--------------------------|------|
| Feign 推荐写法 | **不标** @Configuration | Feign 把它加载到**子上下文**，只作用于这个 Feign，不影响全局 |
| 全局配置写法 | 标 @Configuration 且被扫描 | 变成**全局配置**，所有 Feign 都用它（失去隔离） |

**所有 FeignConfig 都不标 @Configuration**，正是为了保持各自隔离——供应商A的重试策略不能套到供应商B上。

> **一个容易踩的坑**：有人为了让 FeignConfig 里的 `@Autowired` 能稳定解析到主容器的配置 Bean，会在别处额外加 `@Import(XxxFeignConfig.class)` 把它也导入主容器。这对 Feign 正常工作非必需（Feign 靠 `configuration=` 自己会建子上下文），属于不规范的"保险"写法。`@Import` 和 `@Configuration` 是两回事：`@Import` 显式导入类（不论有无 @Configuration），`@Configuration` 声明配置类会被组件扫描。Feign 隔离靠的是"不标 @Configuration + configuration= 让 Feign 进子上下文"。

---

## 五、全局配置 vs 客户端私有配置

除 `configuration` 给单个客户端配，还有全局配置方式：

```yaml
# application.yaml 全局配置（所有 Feign 共享）
feign:
  client:
    config:
      default:              # default 表示全局默认
        connect-timeout: 5000
        read-timeout: 10000
        logger-level: BASIC
      supplierA:            # 只针对 name=supplierA 这个客户端（覆盖全局）
        read-timeout: 65000
```
- `feign.client.config.default` → 全局默认
- `feign.client.config.<name>` → 针对某客户端覆盖
- 优先级：客户端私有 yaml > 全局 yaml > `configuration` 配置类 > 框架默认

供应商多、配置差异大时，用每个供应商一个 `configuration` 类更灵活（但更繁琐）；如果差异不大，用全局 yaml 更简洁。

---

## 六、工作原理（动态代理）

```
① 启动时：
   @EnableFeignClients 扫描所有 @FeignClient 接口
   → 为每个接口用 JDK 动态代理生成实现 Bean，注册进容器
   （接口怎么变成可注入的 Bean？就靠这个代理）
   
② 注入时：
   @Autowired SupplierAApi supplierAApi;
   → 实际注入的是代理对象
   
③ 调用时：
   supplierAApi.query(request)
   → 代理 invoke 拦截：
      ├─ 解析方法上的注解（@PostMapping、@RequestBody）
      ├─ 用 Contract（SpringMvcContract）解析成 RequestTemplate
      │    → URL = supplierA.host + "/api/query"
      │    → method = POST
      │    → body = JSON.toJSONString(request)
      ├─ RequestInterceptor 加 header（token）
      ├─ Client 发 HTTP 请求（默认 HttpURLConnection）
      └─ Decoder 解析响应 → 反序列化成 SupplierAResponse
   → 返回结果
```

**关键**：Feign 接口能注入是因为 `@EnableFeignClients` + 动态代理生成了实现；调用方法时是代理拦截，按注解解析请求模板并发 HTTP。所以你在 Service 里只是 `supplierAApi.query(req)`，像调本地方法一样，背后是完整的 HTTP 调用。

---

## 七、重试与超时

```java
// Request.Options：connectTimeout, readTimeout, followRedirects
@Bean
Request.Options options() {
    return new Request.Options(15, TimeUnit.SECONDS, 65, TimeUnit.SECONDS, true);
}

// Retryer：重试策略
@Bean
Retryer retryer() {
    return Retryer.NEVER_RETRY;        // 不重试（非幂等接口用）
    // return new Retryer.Default();   // 默认重试（最多5次，间隔递增）
    // return new Retryer.Default(100, 1000, 3);  // 自定义：初始间隔100ms，最大1s，最多3次
}
```
注意：超时 + 重试可能放大故障（重试风暴），需谨慎。非幂等接口（可能造成重复扣费）通常 NEVER_RETRY；幂等查询接口可用默认重试，但生产环境通常结合熔断器（Sentinel/Hystrix）限制重试，避免雪崩。

---

## 八、请求拦截器（加签/加 header）

```java
@Bean
RequestInterceptor tokenInterceptor() {
    return template -> {
        template.header("token", supplierAConfig.getToken());  // 加鉴权头
    };
}
```
`RequestInterceptor` 在每次请求发送前执行，可加 header、改 body、记录日志。各供应商的加签方式不同，各自一个 FeignConfig + 拦截器实现，互不干扰。这就是引子里"各家鉴权不一"的解法——每个供应商一个拦截器，集中在自己的 FeignConfig 里。

---

## 九、熔断降级（fallback）

```java
@FeignClient(name = "order-service", fallback = OrderApiFallback.class)
public interface OrderApi { ... }

@Component
public class OrderApiFallback implements OrderApi {
    @Override
    public Order getOrder(Long id) {
        return new Order();  // 调用失败时返回兜底数据
    }
}
```
配合 Hystrix/Sentinel，调用失败时走 fallback，实现服务降级，防止级联故障。对外部供应商接口也可用：某个供应商挂了，返回兜底数据或友好提示，而不是整个请求失败。

---

## 十、面试常问问题与答案

### Q1：OpenFeign 和 RestTemplate 的区别？

- **RestTemplate**：手写式，自己拼 URL、设 header、序列化、解析响应，调用处充满样板代码。
- **Feign**：声明式，只写 interface + 注解，动态代理生成实现，调用方法像调本地方法一样发 HTTP。
- Feign 更简洁、类型安全、易测试（可 mock 接口）、配置统一（FeignConfig）。但 RestTemplate 更灵活可控。
- 对接大量供应商时，Feign 比每个都写 restTemplate.exchange 简洁得多。

### Q2：@FeignClient 的 url 参数填和不填的区别？

- **填 url**：直连该地址，不走服务发现，`name` 只是逻辑标识。适合调外部第三方 API。
- **不填 url**：用 `name` 去注册中心（Nacos/Eureka）查实例，配合 LoadBalancer 负载均衡。适合微服务内部互调。

### Q3：Feign 的 configuration 配置类为什么不该标 @Configuration？

- 标 @Configuration 会被主容器组件扫描，变成**全局配置**，所有 Feign 都用它，失去"每个客户端各自一套"的隔离性。
- 不标 @Configuration 时，Feign 把它加载到**子上下文（独立的 ApplicationContext）**，只作用于当前 Feign 客户端。
- 这是 Feign 官方文档明确推荐的写法，保持各自隔离。

### Q4：Feign 是怎么工作的？（原理）

1. `@EnableFeignClients` 扫描 @FeignClient 接口，用 JDK 动态代理为每个接口生成实现 Bean 注入容器。
2. 调用方法时代理 invoke 拦截，用 Contract（SpringMvcContract）解析方法上的注解（@PostMapping/@RequestBody 等）成 RequestTemplate。
3. RequestInterceptor 加 header/加签；Client 发 HTTP 请求；Decoder 反序列化响应返回。
4. 整个过程对调用方透明，像调本地方法。

### Q5：Feign 的超时和重试怎么配？有什么风险？

- 超时：`Request.Options(connectTimeout, readTimeout, followRedirects)`，可全局 yaml 或 `configuration` 私有配。
- 重试：`Retryer`（NEVER_RETRY / Default / 自定义）。
- 风险：超时短 + 重试多 = 重试风暴，可能把下游打挂（雪崩）。生产环境通常结合熔断器（Sentinel/Hystrix）限制重试，非幂等接口直接 NEVER_RETRY。

### Q6：Feign 的 fallback 熔断降级怎么实现？

`@FeignClient(fallback = XxxApiFallback.class)`，fallback 类实现同接口，调用失败时框架转调 fallback 方法返回兜底数据。配合 Hystrix/Sentinel 的熔断器（断路器开/半开/关三态），实现服务降级防级联故障。

### Q7：Feign 和 Dubbo 的区别？

- **Feign**：基于 HTTP/REST，跨语言友好，配合注册中心做微服务调用（Spring Cloud 体系）。轻量、通用。
- **Dubbo**：基于 TCP 自定义协议（dubbo 协议），性能更高，Java 生态，提供更丰富的服务治理（路由、集群容错等）。重量级。
- 对接外部 HTTP 接口的供应商，用 Feign；内部高性能 RPC 调用，用 Dubbo。

### Q8：Feign 如何实现负载均衡？

- 服务发现模式下（不填 url），Feign 用 `name` 去注册中心拿实例列表，由 `LoadBalancerClient`（早期 Ribbon，现 Spring Cloud LoadBalancer）选一个实例发请求。
- 常见策略：轮询（默认）、随机、最少连接、基于权重。
- 直连模式（填 url）无负载均衡，单点。

### Q9：Feign 的 Contract 是什么？

Contract 负责解析接口方法上的注解，转成 HTTP 请求模板。默认 `SpringMvcContract` 支持 Spring MVC 注解（@RequestMapping/@PostMapping/@RequestBody/@PathVariable 等），所以 Feign 接口写法和 Controller 几乎一样。也可换 Feign 原生注解或自定义。

### Q10：Feign 接口能加 bean 注入吗？比如 @Autowired 依赖？

能。Feign 生成的代理 Bean 是 Spring 管理的，配置类（FeignConfig）里可以 `@Autowired` 主容器的 Bean。例如在 FeignConfig 里 `@Autowired` 一个主容器的 `@ConfigurationProperties` Bean 来拿 token。但要注意 FeignConfig 是子上下文，子上下文能访问父（主）上下文的 Bean，这是 Spring 父子容器机制保证的。

### Q11：@EnableFeignClients 的作用？

开启 Feign 功能，扫描 `basePackages` 下的 @FeignClient 接口，为每个用 JDK 动态代理生成实现 Bean 并注册进 Spring 容器。这就是为什么接口能被 @Autowired 注入。

### Q12：Feign 调用失败有哪些常见原因？

1. 超时（read-timeout 太小，下游慢）。
2. 连接拒绝（url 错 / 服务没起 / 网络）。
3. 反序列化失败（响应结构和声明的不匹配）。
4. 鉴权失败（token/签名错）。
5. 服务发现模式下实例列表为空（服务没注册）。
6. 重试放大导致下游压力骤增（重试风暴）。
