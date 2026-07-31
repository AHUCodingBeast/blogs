# Spring MVC 请求处理链路深度笔记

> 这份笔记围绕一个开放性技术问题展开：**假如你现在有一个对外开放 API 平台，几十甚至上百个接口都要做 AK/SK 鉴权、参数签名校验、限流，还要区分哪些接口免鉴权，你会怎么设计？**  
> 以这个问题为线索，把 Spring MVC 的 Filter、Interceptor、HandlerMapping、HandlerAdapter、AOP、DispatcherServlet 初始化这一整套请求处理链路串起来。

---

## 引子：一个开放 API 平台的鉴权难题

假设我们在做一个对外开放的 API 平台，B 端客户通过 HTTP 接口接入，每个请求都要做这些事：

1. **AK/SK 鉴权**：请求头带 AccessKey，服务端查库校验、查签名算签名，确认调用方身份。
2. **区分豁免接口**：有些接口（第三方回调、登录、公开协议页）不做 AK 鉴权。
3. **参数校验/解密**：部分接口的请求体是加密的，要先解密再交给业务。
4. **身份注入**：鉴权通过后，要把"当前调用方是谁"这个上下文传给 Controller，业务里直接用，不想每个方法再查一次库。
5. **防止身份伪造**：调用方的身份必须由服务端从 AK 反查得出，绝不能让客户自己传 `callerCode` 之类的字段来冒充。

如果你直接在每个 Controller 方法里手写这套逻辑，会有几个问题：

- 几十上百个接口，每个方法都复制一遍鉴权代码 → 重复且易错。
- 想加一个新接口，还得记得把鉴权逻辑抄过去 → 容易漏。
- 静态资源（css/js/图片）的请求也会被鉴权逻辑干扰，要么 NPE 要么全部 401。
- 鉴权逻辑和业务逻辑混在一个方法里，读起来乱、测起来难。

Spring MVC 其实为我们提供了几个分层机制来解决这类"横切关注点"问题，下面我们逐一拆解，最终把它们拼成一个完整的鉴权方案。

---

## 一、整体请求处理流程图

先看全景：一个请求从进入到 Controller 执行，经过的层。

```
请求进入
  │
  ▼
==================== Filter 链（Servlet 容器层，DispatcherServlet 之前） ====================
  Filter1.doFilter()
    Filter2.doFilter()
      缓存请求体的 Filter.doFilter()          ← 包装 request、缓存 body（可重复读）
        │
        ▼ （Filter 链走完，进入 Servlet）
==================== DispatcherServlet（Spring MVC 核心） ====================
  │
  ├─① 遍历所有 HandlerMapping，调用 getHandler(request)
  │    第一个返回非 null 的采用 → 得到 HandlerExecutionChain
  │
  │    HandlerExecutionChain {
  │      handler:        HandlerMethod | ResourceHttpRequestHandler | HttpRequestHandler实现类 | ...
  │      interceptors:   全局拦截器(按路径匹配) + handler自带拦截器
  │    }
  │
  ├─② 依次执行 interceptors.preHandle(request, response, handler)
  │    全部返回 true 才继续；任一返回 false 则中断（仍倒序执行 afterCompletion）
  │
  ├─③ 遍历所有 HandlerAdapter，找第一个 supports(handler)==true 的
  │    由该 HandlerAdapter.handle(request, response, handler) 执行
  │      - HandlerMethod             → RequestMappingHandlerAdapter（反射+参数解析+返回值处理）
  │      - ResourceHttpRequestHandler → HttpRequestHandlerAdapter（输出静态文件）
  │      - 其他 HttpRequestHandler     → HttpRequestHandlerAdapter
  │
  ├─④ 依次执行 interceptors.postHandle（倒序，Controller 之后、视图渲染前）
  │
  ├─⑤ 视图渲染 / 写响应
  │
  └─⑥ 依次执行 interceptors.afterCompletion（倒序，无论成功失败都执行）
        │
        ▼
    Filter 链 doFilter 的后半段（倒序返回）
```

我们方案里要用的几个角色对应到这张图里：
- **缓存请求体的 Filter**：在最外层把 body 缓存成可重复读的版本。
- **鉴权拦截器**：在 preHandle 里做 AK/SK 校验、签名校验、豁免判断、身份注入。
- **豁免注解 + 配置豁免名单**：用来标记"这个接口不做 AK 鉴权"。
- **@RequestAttribute**：Controller 方法参数上用它取拦截器注入的身份对象。

---

## 二、关键组件与伪代码说明

### 2.1 Filter（Servlet 容器层，时机最早）

**职责**：容器级通用基础设施，在 DispatcherServlet 之前执行，"不知道 Spring 是什么"。

一个缓存请求体的 Filter 示例：
```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
    HttpServletRequest request = (HttpServletRequest) req;
    // 把 body 缓存起来，让后续能重复读（鉴权算签名要读一次，@RequestBody 又要读一次）
    CachedBodyRequest（示例名） cachedReq = new CachedBodyRequest（示例名）(request);
    chain.doFilter(cachedReq, res);   // 往下传，DispatcherServlet 和拦截器拿到的都是这个包装后的 request
}
```

**为什么需要它**：HTTP 请求的 InputStream 默认只能读一次，但鉴权拦截器要读 body 算签名，Controller 里 `@RequestBody` 又要读 body 反序列化——两次读会失败。Filter 在最早时机缓存 body，后续 `getInputStream()` 每次返回缓存副本。

### 2.2 HandlerMapping —— 找 handler（含拦截器打包）

DispatcherServlet 遍历所有 HandlerMapping，依据当前的 HttpServletRequest 实例来找到对应的 HandlerExecutionChain：

```java
// DispatcherServlet 伪代码
HandlerExecutionChain chain = null;
for (HandlerMapping hm : this.handlerMappings) {
    HandlerExecutionChain hc = hm.getHandler(request);   // 内部按 URL 匹配
    if (hc != null) {
        chain = hc;
        break;
    }
}
if (chain == null) {  // 没找到 → 404 }
```

### 2.3 HandlerExecutionChain —— 打包 handler + 拦截器

`getHandler` 返回的不是裸 handler，而是"包装盒"：
```java
public class HandlerExecutionChain {
    private final Object handler;                        // 真正的处理器（类型不定）
    private final List<HandlerInterceptor> interceptorList;  // 该请求要走的拦截器列表

    // 拦截器来源（构 chain 时合并）：
    //   1. 全局/路径映射拦截器：WebMvcConfigurer.addInterceptors() 注册的，按 addPathPatterns 匹配
    //   2. handler 自带的拦截器（少见）
}
```

在我们的方案里，鉴权拦截器**不配 addPathPatterns → 对所有路径生效**（默认全部拦截），然后用内部逻辑决定豁免谁。

### 2.4 preHandle 的 handler 参数 —— 类型不固定

```java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
    // handler 可能是：
    //   - HandlerMethod               （@RequestMapping 标注的 Controller 方法）
    //   - ResourceHttpRequestHandler  （静态资源）
    //   - 其他 HttpRequestHandler 实现类
    // 所以签名是 Object，必须先 instanceof 判断
    if (!(handler instanceof HandlerMethod) || !isCachedBodyReq(request)) {
        return true;   // 非 Controller 方法（静态资源等）直接放行
    }
    // ... 真正鉴权
}
```

**漏掉 `instanceof HandlerMethod` 守卫的后果**：静态资源（css/js/img）的 handler 是 `ResourceHttpRequestHandler`，强转 HandlerMethod 会 NPE；或把静态资源也鉴权导致全部 401。这行判断是"拦截全部"策略下必须的守卫。

### 2.5 HandlerMethod —— "类+方法"的封装

即使 handler 是 HandlerMethod，它也不是 Controller 类本身，而是封装对象：
```java
public class HandlerMethod {
    private Object bean;            // Controller 实例
    private Method method;          // 对应的方法对象
    private MethodParameter[] parameters;  // 参数元数据（含 @RequestBody/@RequestAttribute 等）

    // 因此能：
    handler.getMethodAnnotation(SkipAuth.class);          // 从方法上找注解
    AnnotationUtils.findAnnotation(handler.getBeanType(), ...);  // 从类上找注解
}
```

这就给了我们"豁免判断"的切入点：在拦截器里读 handler 的注解，判断是不是标了 `@SkipAuth`（免鉴权）。

### 2.6 HandlerAdapter —— 分发执行

最后执行不是 DispatcherServlet 直接调 handler，而是按类型选 adapter：
```java
// DispatcherServlet 伪代码
HandlerAdapter ha = null;
for (HandlerAdapter adapter : this.handlerAdapters) {
    if (adapter.supports(handler)) {   // 找第一个支持的
        ha = adapter;
        break;
    }
}
ModelAndView mv = ha.handle(request, response, handler);
```

| handler 类型 | HandlerAdapter | 干什么 |
|--------------|-----------------|--------|
| HandlerMethod | RequestMappingHandlerAdapter | 反射调用 Controller 方法，处理 @RequestBody 参数解析 / @RequestAttribute 注入 / 返回值处理 |
| ResourceHttpRequestHandler | HttpRequestHandlerAdapter | 直接处理静态资源输出 |
| 其他 HttpRequestHandler | HttpRequestHandlerAdapter | 对应处理 |

**`@RequestBody` 能反序列化、`@RequestAttribute` 能注入，都是 RequestMappingHandlerAdapter 通过 HandlerMethodArgumentResolver 链干的，不是 Servlet 容器的功能。** 这正是我们方案里"拦截器 setAttribute 注入身份、Controller 用 @RequestAttribute 取"能成立的基础。

---

## 三、三道时机的对比（必须记牢）

| 机制 | 执行时机 | 谁定义的 | 能拿到啥 |
|------|---------|---------|---------|
| Filter | 最早，DispatcherServlet 之前，Servlet 容器层 | Servlet 规范 | 拿不到 handler，body 是原始流（要自己缓存才能重复读） |
| 拦截器 preHandle | DispatcherServlet 找到 handler 后、Controller 执行前 | Spring MVC | 能拿到 handler（HandlerMethod），能拿到 body（已被 Filter 缓存） |
| 拦截器 postHandle | Controller 执行完、视图渲染前 | Spring MVC | 能拿到 ModelAndView |
| 拦截器 afterCompletion | 视图渲染后，无论成功失败都执行 | Spring MVC | 能拿到异常 |

**口诀**："Filter 在 Servlet 外，拦截器在 Servlet 内"。
- Filter 干通用基础设施（改 request 包装、缓存 body、加跨域头）
- 拦截器干 Spring 业务相关（鉴权、日志、权限）

回到我们的鉴权难题：缓存 body 必须**早于**鉴权（因为鉴权要读 body 算签名），所以缓存放 Filter；鉴权需要"知道是哪个 Controller、读不读 body"才能决定怎么处理，所以放拦截器。两者天然分层。

---

## 四、方案落地：一条完整的鉴权链路串联

把上面的组件拼起来，一个带 AK 鉴权的请求是这样流转的（以一个查询接口为例）：

```text
请求 /api/query（请求头带 AK + SK 签名）
  │
  ▼ Filter 链
  缓存请求体的 Filter：把 body 缓存，包装成 CachedBodyRequest（示例名）
  │
  ▼ DispatcherServlet
  HandlerMapping.getHandler() → HandlerExecutionChain {
      handler = HandlerMethod(SomeController#queryMethod)
      interceptors = [AuthInterceptor（示例名）, ...]
  }
  │
  ▼ AuthInterceptor（示例名）.preHandle(CachedBodyRequest（示例名）, response, HandlerMethod)
  ├─ handler instanceof HandlerMethod? 是
  ├─ isCachedBodyReq? 是
  ├─ needSkip? 否（不在豁免名单、无 @SkipAuth 注解）
  ├─ 路径不以特定免鉴权前缀开头
  └ 鉴权链：校验AK → 查出调用方身份 → request.setAttribute("current-caller", callerInfo)
  │
  ▼ HandlerAdapter（RequestMappingHandlerAdapter）执行
  SomeController.queryMethod(...)
   ├─ @RequestBody QueryRequest：从已缓存 body 反序列化（能再读一次）
   └─ @RequestAttribute("current-caller") CallerInfo：拿到上一步 set 的调用方对象
  │
  ▼ 响应返回
```

**对比静态资源** `/css/normal.css`：

```text
HandlerMapping → ResourceHttpRequestHandler（非 HandlerMethod）
AuthInterceptor（示例名）.preHandle(..., ResourceHttpRequestHandler)
  └ handler instanceof HandlerMethod? 否 → return true 直接放行
ResourceHttpRequestHandler 直接返回静态文件
```

几个设计要点：
1. **身份注入防伪造**：调用方身份（如 callerCode）是从请求头的 AK 反查库得来的，不是从请求体取的。客户无法通过自己传 `callerCode` 冒充别人——因为拦截器只信 AK 反查的结果。
2. **豁免三级判断**：配置豁免名单（如某些回调接口）、注解豁免（`@SkipAuth`）、非 Controller 方法（静态资源）。这三层确保"默认安全，显式放行"。
3. **body 可重复读**：Filter 缓存后，鉴权读一次算签名、Controller `@RequestBody` 读一次反序列化，互不干扰。

---

## 五、HandlerMethod 与 Interceptor 的绑定

> 前面我们说 HandlerExecutionChain 包含一个 handler 对象和相关的拦截器列表，对于我们常说的 controller，也就是 HandlerMethod 对象和相关的拦截器列表，那么这两者的关联关系是怎么建立的呢？

实际上 HandlerMethod 和 Interceptor **不是启动时一次性绑死成 HandlerExecutionChain**，而是 **"启动时各自注册，运行时按请求动态打包"**。分两阶段：

### 5.1 启动阶段：HandlerMethod 的注册（RequestMappingHandlerMapping）

```text
Spring 容器启动
  ↓
RequestMappingHandlerMapping 实现了 InitializingBean 接口
  ↓ afterPropertiesSet() 触发（容器启动时回调）
  ↓
AbstractHandlerMethodMapping.detectHandlerMethods(Object handler)
  ↓ 遍历容器中所有带 @Controller/@RestController 注解的 Bean
  ↓ 对每个 Bean 反射扫描其方法：
  │   if (方法有 @RequestMapping 或其派生注解(@PostMapping/@GetMapping 等))
  │       构建 RequestMappingInfo（URL 模式 + HTTP 方法 + params + headers + consumes + produces）
  │       注册到 this.mappingRegistry：
  │           Map<T, HandlerMethod> mappingLookup  // key=RequestMappingInfo, value=HandlerMethod
  │           MultiValueMap<String, T> urlLookup  // key=URL路径, value=能匹配的RequestMappingInfo列表
  │
  ↓ 启动完成时，所有 @RequestMapping 方法都已变成 HandlerMethod 注册进 mappingRegistry
```

**结果**：启动后，`RequestMappingHandlerMapping` 内部维护一张"URL→HandlerMethod"的大表，此时**还没有任何 Interceptor 参与**。

### 5.2 启动阶段：Interceptor 的注册（WebMvcConfigurer）

```
Spring 容器启动
  ↓
处理所有实现 WebMvcConfigurer 的配置类（写一个注册类实现它）
  ↓ 调用 addInterceptors(InterceptorRegistry registry)
  ↓
registry.addInterceptor(authInterceptor（示例）)
  可选 .addPathPatterns("/api/**")        // 限定拦截路径
  可选 .excludePathPatterns("/api/login") // 排除路径
  ↓
注册到 IntercepterRegistry，生成 MappedInterceptor（含 path 匹配规则）
  ↓ 启动完成时，MappedInterceptor 列表保存在 ApplicationContext 中
```

**结果**：拦截器和 pathPattern 规则一起注册，但**此时不关联任何具体 HandlerMethod**。

在我们的方案里，注册时**不配 addPathPatterns → 默认拦截所有路径**，设计上靠"拦截全部 + 内部豁免判断"而不是"只拦截部分路径"。

### 5.3 运行时阶段：动态打包成 HandlerExecutionChain

请求进来时，`getHandler(request)` 才把两者按需打包：
```java
// RequestMappingHandlerMapping（继承 AbstractHandlerMethodMapping）的 getHandler 伪代码
public final HandlerExecutionChain getHandler(HttpServletRequest request) {
    // 1. 按请求 URL + HTTP 方法查 mappingRegistry，找到匹配的 HandlerMethod
    HandlerMethod handlerMethod = lookupHandlerMethod(request);   // 查"URL→HandlerMethod"大表
    if (handlerMethod == null) return null;

    // 2. 包装成 HandlerExecutionChain，并附加拦截器
    HandlerExecutionChain chain = new HandlerExecutionChain(handlerMethod);

    // 3. 遍历所有全局 MappedInterceptor，按 path 匹配加入 chain
    chain.addInterceptors(matchingMappedInterceptors(request));  // 用 AntPathMatcher 比对 pathPattern

    // 4. 加上 handler 自带的拦截器（少用）
    chain.addInterceptors(handlerMethod自身关联的拦截器);

    return chain;
}
```

**关键**：HandlerMethod（按 URL 查表得到）和 Interceptor（按 pathPattern 匹配得到）在**运行时才组合成 HandlerExecutionChain**，每个请求一份。绑定依据是"URL 是否匹配 + pathPattern 是否匹配"，不是启动时静态绑定。

### 5.4 总结一张图

```
【启动时】各自注册，互不关联
  RequestMappingHandlerMapping.afterPropertiesSet()
    → 扫描 @Controller Bean 的 @RequestMapping 方法
    → 注册到 mappingRegistry：Map<RequestMappingInfo, HandlerMethod> + Map<URL, List<RequestMappingInfo>>

  WebMvcConfigurer.addInterceptors(registry)
    → registry.addInterceptor(authInterceptor（示例）) → 生成 MappedInterceptor(含pathPattern)
    → 存到 ApplicationContext（此时和任何 HandlerMethod 都没关联）

【运行时】请求进来才打包
  RequestMappingHandlerMapping.getHandler(request)
    1. lookupHandlerMethod(request) → 从 mappingRegistry 查到 HandlerMethod
    2. new HandlerExecutionChain(handlerMethod)
    3. 遍历所有 MappedInterceptor，pathPattern 匹配的加入 chain
    → 返回 chain（handler + 匹配的拦截器）
```

---

## 六、面试常问问题与答案

### Q1：Filter 和 Interceptor 的区别？执行顺序？

**区别**：
- **规范层面**：Filter 是 Servlet 规范（javax.servlet），Interceptor 是 Spring MVC（org.springframework.web.servlet）。
- **执行时机**：Filter 在 DispatcherServlet 之前（容器层）；Interceptor 在 DispatcherServlet 之内、Controller 前后（Spring 层）。
- **能力**：Filter 拿不到 handler（还不知道哪个 Controller 处理）；Interceptor 能拿到 handler（HandlerMethod），能用 Spring 设施（注入 Bean、读注解、操作 HandlerMethod 元数据）。
- **典型用途**：Filter 干通用基础设施（改 request 包装、缓存 body、加跨域头）；Interceptor 干 Spring 业务（鉴权、日志、权限）。
- **触发**：Filter 由 Servlet 容器按 web.xml/@WebFilter/FilterRegistrationBean 触发；Interceptor 由 DispatcherServlet 在 preHandle/postHandle/afterCompletion 三时机触发。

**顺序**：Filter → DispatcherServlet → Interceptor.preHandle → Controller → Interceptor.postHandle → 渲染 → Interceptor.afterCompletion → Filter 倒序返回。

口诀：**"Filter 在 Servlet 外，Interceptor 在 Servlet 内"**。

### Q2：preHandle 的 handler 参数一定是 Controller 吗？

**不是**。`Object handler` 是泛化概念，任何能处理请求的对象都行：
- `HandlerMethod`：@RequestMapping 标注的 Controller 方法（这才是"Controller"概念，封装了 bean+Method+参数）。
- `ResourceHttpRequestHandler`：静态资源（/css /js /img 等）。
- 其他 `HttpRequestHandler` 实现类：WebSocket、Hessian 等。

所以拦截器里必须先 `instanceof HandlerMethod` 判断，否则静态资源进来强转会 NPE。典型的守卫写法：
```java
if (!(handler instanceof HandlerMethod) || !isCachedBodyReq(request)) return true;
```

### Q3：HandlerMethod 和 Interceptor 是什么时候绑定的？

**不是启动时一次性绑死，而是运行时按请求动态打包。**
- **启动时**：RequestMappingHandlerMapping 扫描所有 @Controller Bean 的 @RequestMapping 方法，注册到 mappingRegistry（URL→HandlerMethod 大表）；Interceptor 通过 addInterceptors 注册成 MappedInterceptor（含 pathPattern）。两者各自独立，互不关联。
- **运行时**：请求进来，getHandler(request) 先按 URL 查表得 HandlerMethod，再按 pathPattern 匹配 MappedInterceptor，动态组合成 HandlerExecutionChain。每个请求一份。

### Q4：一个请求怎么找到对应的 Controller 方法？

DispatcherServlet 遍历所有 HandlerMapping（有优先级），第一个匹配的采用：
1. RequestMappingHandlerMapping：按"URL + HTTP 方法 + params/headers/consumes/produces"匹配，查 mappingRegistry 得到 HandlerMethod。
2. BeanNameUrlHandlerMapping / SimpleUrlHandlerMapping：按 URL 匹配 SimpleUrlHandlerMapping 配置的 handler（如静态资源、WebSocket）。

### Q5：@RequestBody 为什么能反序列化？谁干的？

不是 Servlet 容器，是 **RequestMappingHandlerAdapter** 通过 **HandlerMethodArgumentResolver** 链干的：
- RequestResponseBodyMethodProcessor 解析 @RequestBody：用 HttpMessageConverter（如 MappingJackson2HttpMessageConverter）把 body 反序列化成对象。
- RequestAttributeMethodProcessor 解析 @RequestAttribute：从 request 的 attribute 里取值。

**前提**：body 可重复读（被 Filter 缓存成可重复读的包装类），否则鉴权读一次后 Controller 再读会失败。

### Q6：DispatcherServlet 的工作流程？

1. 找 HandlerMapping 得 HandlerExecutionChain（handler + interceptors）。
2. 找 HandlerAdapter（supports(handler)==true）执行 handler。
3. 执行拦截器 preHandle（失败中断，已执行的倒序 afterCompletion）。
4. HandlerAdapter.handle 执行 Controller，返回 ModelAndView。
5. postHandle 拦截器（倒序）。
6. 渲染视图（ViewResolver）/ 写响应。
7. afterCompletion 拦截器（倒序）。

### Q7：如果拦截器 preHandle 返回 false 会发生什么？

- 请求处理中断，Controller 不执行。
- 已执行的拦截器的 afterCompletion **仍会倒序执行**（保证资源释放）。
- 可直接写 response 返回（如返回 401/403 JSON）。

### Q8：Filter 能拿到 Spring 的 Bean 吗？

能，但不能直接 @Autowired（Filter 由 Servlet 容器管理，不在 Spring 容器内）。需用 DelegatingFilterProxy（把 Filter 代理到 Spring 容器的 Bean）。或用 FilterRegistrationBean 注册（Spring 容器管理），则可直接注入。

### Q9：为什么需要可重复读请求体（CachedBodyRequest（示例名））？

HTTP 请求 InputStream 默认只能读一次。但鉴权拦截器要读 body 算签名，Controller 的 @RequestBody 又要读 body 反序列化——两次读会失败。所以用 Filter 在最早时机把 body 缓存，后续 getInputStream() 每次返回缓存副本，支持重复读。

### Q10：HandlerAdapter 的作用是什么？为什么需要它？

DispatcherServlet 不直接调 handler，因为 handler 类型多样（HandlerMethod / ResourceHttpRequestHandler / ...），调用方式完全不同。HandlerAdapter 用适配器模式解耦：每种 handler 配一个 adapter，DispatcherServlet 只需找 `supports(handler)==true` 的 adapter 委托执行。这样新增 handler 类型只需加 adapter，DispatcherServlet 不用改。

---

## 七、鉴权链路设计要点总结

1. **Filter + Interceptor 分工**：Filter 干"通用基础设施"（缓存 body），Interceptor 干"Spring 业务"（鉴权）。
2. **拦截全部 + 分级豁免**：鉴权拦截器不配 addPathPatterns（默认拦截所有），内部用三级豁免放行：
   - 配置豁免（yaml 配一个豁免类名单，如某些回调 Controller）
   - 注解豁免（自定义 `@SkipAuth` 注解，标在免鉴权的方法/类上）
   - 非 Controller 方法（静态资源，instanceof HandlerMethod 判断）
3. **handler 类型守卫**：preHandle 必须先 instanceof HandlerMethod，避免静态资源强转 NPE。
4. **request.setAttribute 注入身份**：鉴权通过后 setAttribute 注入调用方身份对象，Controller 用 @RequestAttribute 直接拿到，避免重复鉴权查库。身份从 AK 反查（非客户传），防身份伪造。
5. **body 可重复读**：缓存请求体的 Filter 缓存 body，支持鉴权读一次 + Controller @RequestBody 读一次。
6. **默认安全**：设计上"默认全部拦截，显式豁免"，漏配不会导致漏鉴权（安全原则）。

---

## 八、AOP 切面的执行时机

> 问题：如果 Controller 上还有切面（AOP），切面的执行时机是什么时候？

### 8.1 一句话结论

**在 Interceptor 的 preHandle 之后、Controller 方法真正执行的前后。**

完整顺序：
```
Filter 链
  ↓
DispatcherServlet
  ↓
Interceptor.preHandle        ← 先执行（鉴权等）
  ↓
HandlerAdapter.handle
  ↓  调用的是 Controller 的代理对象（CGLIB/JDK动态代理）
  ↓
  切面 @Before / @Around前半段     ← 再执行（日志/事务等）
    ↓
    目标Controller方法真正执行
    ↓
  切面 @After / @AfterReturning / @Around后半段
  ↓
Interceptor.postHandle        ← 最后执行
  ↓
Interceptor.afterCompletion
```

### 8.2 完整层级对比

```
请求 → Filter（容器层，改request/缓存body）
       → Interceptor.preHandle（SpringMVC层，鉴权/拿handler）
           → AOP切面（Spring AOP层，方法级，如日志/事务）
               → 目标Controller方法
           ← AOP切面收尾
       ← Interceptor.postHandle / afterCompletion
```

**关键理解**：
- **Interceptor** 是 Spring MVC 层，作用于"请求→handler"这一级，管的是"要不要处理这个请求"（鉴权、限流）。
- **AOP** 是更细粒度，作用于"方法调用"这一级（日志、事务、缓存等）。
- Interceptor 先判断要不要处理这个请求，通过后才到方法调用层，这时 AOP 代理才介入。

### 8.3 为什么 Interceptor 在前、AOP 在后

- **Interceptor** 由 DispatcherServlet 在"执行 handler"前后调用，它管的是"要不要处理这个请求"。拦截器通过时，才轮到 HandlerAdapter 去执行 handler。
- **HandlerAdapter 执行 HandlerMethod 时**，拿到的是 Spring 容器里的 Bean。如果这个 Bean 被切面增强过（如 @Transactional、自定义 @Aspect），容器里存的就是**代理对象**。所以"执行方法"实际是"调用代理对象的方法"，切面就在这一刻织入。

**层级关系：Interceptor 在"请求处理"层，AOP 在"方法调用"层，前者包住后者。**

### 8.4 经典坑：内部调用导致切面失效（@Transactional 失效）

AOP 基于代理实现，**目标方法内部直接 `this.方法()` 调用，绕过了代理，切面不生效**。

常见解法是用 `AopContext.currentProxy()` 显式走代理：
```java
public void outerMethod(...) {
    // ❌ 直接 this.调用 → 走的是原对象，不经过代理，@Transactional 等切面失效！
    // this.innerMethod(...);

    // ✅ 通过代理调用 → 切面生效（事务能正常开启/提交）
    SomeServiceImpl proxy = (SomeServiceImpl) AopContext.currentProxy();
    proxy.innerMethod(...);   // 确保内部调用时事务等切面生效
}

@Transactional  // 这个切面必须经过代理才生效
public void innerMethod(...) { ... }
```

**为什么需要 exposeProxy=true**：`AopContext.currentProxy()` 需要框架把当前代理对象暴露到 ThreadLocal，这要求 `@EnableAspectJAutoProxy(exposeProxy = true)` 开启，否则 `currentProxy()` 会抛异常。如果你的代码里大量用了这种"自调用走代理"的模式，就需要开启此项。

### 8.5 切面用途举例

1. **日志切面（@Aspect）**：在方法执行前后织入，记录 traceId、入参出参摘要，粒度是"方法"。
2. **事务（@Transactional）**：保证方法内多个 DB 操作原子性，提交/回滚靠代理触发。
3. **缓存、限流、监控**等也可用切面实现。

注意区分层级：拦截器是"请求"级，切面是"方法"级，两者不是一回事。

### 8.6 五种通知（Advice）的执行顺序

```
@Around（前半段）  ← 最外层，包裹整个方法调用
  @Before          ← 方法执行前
    目标方法执行
  @AfterReturning  ← 方法正常返回后（若有异常不执行）
  @After           ← 方法执行后（无论成功失败，类似 finally）
@Around（后半段）  ← 最外层收尾
```

注意：@After 和 @AfterReturning 的区别——`@After` 类似 finally 无论成功失败都执行；`@AfterReturning` 只在正常返回时执行，且能拿到返回值。`@AfterThrowing` 只在抛异常时执行。

---

## 九、DispatcherServlet 继承链与 onRefresh 时机

> 问题1：DispatcherServlet 继承 FrameworkServlet，FrameworkServlet 是不是把 Servlet 和 Spring 容器关联的组件？
> 问题2：onRefresh 方法的执行时机是 Servlet 容器启动成功之后吗？

### 9.1 完整继承链

```
javax.servlet.GenericServlet          ← Servlet 规范
  └ javax.servlet.http.HttpServlet    ← Servlet 规范（HTTP 专用）
      └ org.springframework.web.servlet.HttpServletBean
          └ org.springframework.web.servlet.FrameworkServlet   ← 关键桥梁！
              └ org.springframework.web.servlet.DispatcherServlet ← 业务核心
```

四层职责渐进式、单一职责：

| 层 | 职责 | 知不知道 Spring 容器 |
|----|------|---------------------|
| **HttpServletBean** | 实现 `init()`，读取 web.xml 的 init-param，用 JavaBean 方式 set 属性（BeanWrapper）。**不感知 Spring 容器** | 否 |
| **FrameworkServlet** | **桥梁层**：把 Servlet 生命周期（init、service）和 Spring WebApplicationContext 关联起来 | **是，核心职责** |
| **DispatcherServlet** | 实现请求分发：HandlerMapping/HandlerAdapter/ViewResolver 等 | 用容器里的组件 |

**结论：FrameworkServlet 就是把 Servlet 和 Spring 容器粘起来的那一层。**

### 9.2 FrameworkServlet 做了哪几件"关联"的事

#### ① 持有并初始化 WebApplicationContext

```java
// FrameworkServlet 伪代码
public class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {

    private WebApplicationContext webApplicationContext;  // 持有 Spring 容器

    // HttpServletBean.init() 最后会回调这个方法（模板方法模式）
    @Override
    protected final void initServletBean() {
        // 创建/获取 WebApplicationContext（这就是 Spring 容器）
        this.webApplicationContext = initWebApplicationContext();
        initFrameworkServlet();
    }
}
```

`initWebApplicationContext()` 会找到或创建 `WebApplicationContext`（Spring 容器）。如果配置了 `ContextLoaderListener`（根容器），以它为父容器；否则自己建一个以 Servlet 名命名的子容器。

#### ② 提供 onRefresh 模板方法，让子类初始化策略组件

```java
// FrameworkServlet
protected void onRefresh(ApplicationContext context) {
    // 空实现，留给子类
}

// DispatcherServlet 覆写
@Override
protected void onRefresh(ApplicationContext context) {
    initStrategies(context);   // 从容器里拿各种策略组件
}

protected void initStrategies(ApplicationContext context) {
    // 注意：这些都是从 Spring 容器里找的！
    this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
    this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
    this.viewResolvers = getDefaultStrategies(context, ViewResolver.class);
    this.handlerExceptionResolvers = ...
    this.localeResolvers = ...
    this.themeResolvers = ...
    // 等共 9 大组件
}
```

**"关联"的落点**：DispatcherServlet 持有的所有 `handlerMappings`、`handlerAdapters` 等，都是通过 `onRefresh` → `initStrategies` **从 Spring 容器里捞出来的 Bean**。没有 FrameworkServlet 拿到容器，DispatcherServlet 就没有这些组件可用。

#### ③ 把 Servlet 的请求处理（service）引到 Spring 的 doService

```java
// FrameworkServlet 伪代码
@Override
protected final void service(HttpServletRequest req, HttpServletResponse resp) {
    processRequest(req, resp);
}

@Override
protected final void doGet/doPost/doPut/doDelete(...) {
    processRequest(req, resp);
}

protected final void processRequest(...) {
    // 1. 发布 RequestContext 上下文（LocaleContext、RequestAttributes 到 ThreadLocal）
    LocaleContextHolder.setLocaleContext(...);
    RequestContextHolder.setRequestAttributes(new ServletRequestAttributes(request));

    // 2. 发布 RequestHandledEvent（ServletRequestHandledEvent）
    publishRequest(request, response);

    try {
        // 3. 调子类实现的 doService
        doService(request, response);
    } finally {
        // 4. 清理 ThreadLocal、发布事件
        RequestContextHolder.resetRequestAttributes();
        publishRequestHandled(false);
    }
}

// DispatcherServlet 实现
@Override
protected void doService(HttpServletRequest request, HttpServletResponse response) {
    // 设置一些 request 属性（如 webApplicationContext、localeResolver）
    request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
    // ...
    doDispatch(request, response);   // ← 这里才是真正的分发
}
```

注意一个细节：`processRequest`、`service`、`doGet` 在 FrameworkServlet 里**都被 final 修饰**，子类不能重写，只能通过 `doService()` 这个模板方法介入。这样保证了：
- ThreadLocal 上下文（Locale、RequestAttributes）的设置和清理一定发生
- RequestHandledEvent 一定发布（用于监听统计）
- 真正的分发逻辑由 DispatcherServlet 的 `doService` → `doDispatch` 完成

### 9.3 为什么这样分层（设计哲学）

这个三层结构体现了**单一职责 + 模板方法**：

1. **HttpServletBean**：只负责"Servlet 配置参数注入"（读 web.xml init-param → setProperty）。连容器都不碰，纯粹处理 Servlet 规范的初始化。
2. **FrameworkServlet**：负责"Servlet ↔ Spring 容器的桥接"。
   - init 阶段：`initWebApplicationContext` 拿到容器
   - refresh 阶段：容器刷新时回调 `onRefresh`（通过 ApplicationListener 监听 ContextRefreshedEvent）
   - request 阶段：`service/doGet` → `processRequest` → 设置上下文 → 调子类 `doService`
3. **DispatcherServlet**：负责"MVC 请求分发"。只用容器里的组件，不关心容器怎么来的、Servlet 生命周期怎么管的。

**好处：FrameworkServlet 可以被复用。** 理论上你可以写一个非 Dispatcher 的子类（比如做 RPC、做自定义协议），复用它的"容器桥接 + 上下文管理 + 事件发布"，只重写 `doService`。Spring WebFlux 也有对应的 `DispatcherHandler` + 不同基类，思路一致。

### 9.4 onRefresh 的执行时机（核心问题）

#### 常见误区纠正

很多人以为 `onRefresh` 是"Servlet 容器启动后调用"，**不准确**。它真正的触发链是：**Servlet 初始化 → 创建/获取 Spring 容器 → Spring 容器 refresh() → 触发 onRefresh**。

也就是说，`onRefresh` 不是 Servlet 容器直接调的，而是 **Spring 容器 refresh 完成时，通过事件回调 FrameworkServlet** 的 `onRefresh`。

#### 完整时序（精确版）

```
① 应用启动，Servlet 容器（如 Tomcat）启动
   ↓
② Servlet 容器实例化并初始化 DispatcherServlet（按 load-on-startup 配置）
   ↓ HttpServletBean.init()（Servlet 规范的 init 生命周期）
   ↓ initServletBean()
③ FrameworkServlet.initServletBean()
   ├─ initWebApplicationContext()    ← 这里创建/获取 Spring 容器
   │    ├─ 找 ContextLoaderListener 建的根容器（WebApplicationContext）
   │    ├─ 如果没有，以根容器为父，创建 DispatcherServlet 自己的子容器
   │    └─ 调用容器的 refresh() 或注册监听器
   ↓
④ Spring 容器 refresh 完成
   ↓ 触发 ContextRefreshedEvent 事件
⑤ FrameworkServlet 监听到事件
   ├─ onRefresh(context)            ← DispatcherServlet 覆写：initStrategies
   └─ 这里把 HandlerMapping/HandlerAdapter/ViewResolver 等从容器里捞出来
   ↓
⑥ DispatcherServlet 初始化完成，可以处理请求了
```

#### 核心机制：onRefresh 是通过 ContextRefreshedEvent 触发的

```java
// FrameworkServlet.initWebApplicationContext() 伪代码
protected WebApplicationContext initWebApplicationContext() {
    WebApplicationContext root = (WebApplicationContext) getServletContext()
            .getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);

    WebApplicationContext wac = createWebApplicationContext(root);
    // 关键：把 FrameworkServlet 自己注册为容器的 ApplicationListener
    if (wac instanceof ConfigurableApplicationContext) {
        ((ConfigurableApplicationContext) wac).addApplicationListener(this);  // ★
    }

    // refresh 时会触发 ContextRefreshedEvent
    wac.refresh();  // 或由 ContextLoaderListener 触发
    onRefresh(wac); // ★★★
    return wac;
}

// FrameworkServlet 实现了 ApplicationListener<ContextRefreshedEvent>
@Override
public void onApplicationEvent(ContextRefreshedEvent event) {
    if (event.getApplicationContext() == this.webApplicationContext) {
        this.onRefresh(event.getApplicationContext());
    }
}
```

所以 `onRefresh` 的触发有**两条路径**：

| 路径 | 触发方式 | 何时 |
|------|---------|------|
| **路径 A** | `initWebApplicationContext()` 里直接调 `onRefresh(wac)` | Servlet init 时，容器刚 refresh 完 |
| **路径 B** | FrameworkServlet 是 `ApplicationListener`，监听 `ContextRefreshedEvent` | 容器每次 refresh 完都触发（含首次和后续热刷新） |

**路径 B 是主路径，路径 A 是兜底。** 双保险确保无论容器是同步 refresh 还是异步 refresh，onRefresh 都能被调到。

#### 精确回答："是 Servlet 容器启动成功之后吗"

**不是"Servlet 容器启动后"触发，而是"Spring 容器 refresh 完成后"触发。**

更精确地说：
- Servlet 容器启动 → **只是开始实例化 Servlet**
- DispatcherServlet 的 `init()` 被调 → `initServletBean` → `initWebApplicationContext` → 创建/获取 Spring 容器 → 容器 `refresh()` 完成 → **触发 ContextRefreshedEvent** → `onRefresh`
- 所以 onRefresh 是"Servlet init 过程中、Spring 容器 refresh 完成时"触发的

时序上它**早于"Servlet 容器完全启动成功"**，因为 Servlet 容器要等所有 Servlet 的 init 都完成才算启动成功。准确说法：**onRefresh 发生在 DispatcherServlet 的 init 阶段，紧随 Spring 容器 refresh 完成之后。**

### 9.5 一个常见的坑（强依赖顺序）

如果 DispatcherServlet 的 `load-on-startup` 没配，或者 Spring 容器 refresh 比 Servlet init 早完成（比如用 ContextLoaderListener 提前建了根容器），`onRefresh` 靠的是 ApplicationListener 监听事件这条路径兜底。Spring 用了"直接调用 + 监听事件"双保险来保证 `initStrategies` 一定能执行到，从而保证 DispatcherServlet 拿得到 HandlerMapping 等组件。这就是为什么 FrameworkServlet 要同时实现 `onApplicationEvent` 和在 `initWebApplicationContext` 里直接调——防止时序错乱导致组件没初始化。

### 9.6 和第五章串起来的完整时序链

回顾"五、HandlerMethod 与 Interceptor 的绑定"，现在时序能对齐了：

```
Spring 容器 refresh 阶段（此时 DispatcherServlet 的 init 也在进行）
  ├─ RequestMappingHandlerMapping.afterPropertiesSet()  ← 扫描 @Controller 注册 HandlerMethod
  ├─ WebMvcConfigurer.addInterceptors()                 ← 注册拦截器成 MappedInterceptor
  ↓ 容器 refresh 完成
  ↓ ContextRefreshedEvent
DispatcherServlet.onRefresh()
  └─ initStrategies(context)  ← 从容器把 RequestMappingHandlerMapping 等捞出来存到成员变量
```

即：**容器 refresh 时**各组件（HandlerMapping、Interceptor）初始化注册完毕；**refresh 完成触发 onRefresh** 时，DispatcherServlet 才把这些组件"捞出来"持有。所以请求来时，DispatcherServlet 才能用已注册好的 HandlerMapping 查表找 HandlerMethod。整条链的时序源头就是 Spring 容器的 refresh 和这个 onRefresh 回调。

### 9.7 一句话总结

`onRefresh` 的触发时机**不是"Servlet 容器启动成功后"，而是"Spring WebApplicationContext refresh() 完成时"**——它发生在 DispatcherServlet 的 init 阶段，通过直接调用 + ContextRefreshedEvent 事件监听双保险触发，目的是让 DispatcherServlet 从容器里捞出 HandlerMapping/HandlerAdapter/ViewResolver 等策略组件。Servlet 容器启动成功是这个 init 完成之后的事。

**FrameworkServlet 就是 Servlet 和 Spring 容器之间的"桥"**：它在 init 时拿到/创建 WebApplicationContext，把容器引用存进 Servlet；在容器刷新时回调子类 `onRefresh` 让 DispatcherServlet 从容器里把 HandlerMapping/HandlerAdapter/ViewResolver 等策略组件全捞出来；在请求来时通过 `processRequest → doService` 把 Servlet 的 service 调用引到 Spring MVC 的分发逻辑上，并负责 ThreadLocal 上下文管理和事件发布。

---

## 十、DispatcherServlet 与前端控制器模式

> 问题：Tomcat 是 Servlet 容器，一个项目可能有多个 Servlet。Servlet 是处理 HTTP 请求的组件，DispatcherServlet 通过实现 Servlet 接口也获得处理 HTTP 请求的能力，并在内部使用 Spring 组件把请求交给 Controller 处理。这个理解对不对？

### 10.1 表述逐句核对

| 表述 | 准确性 | 精化补充 |
|---------|--------|---------|
| Tomcat 是 Servlet 容器 | ✅ 对 | 完整说法是"Servlet/JSP 容器"，负责管理 Servlet 生命周期（实例化、init、service、destroy）、线程调度、把 socket 字节流解析成 HttpServletRequest/Response |
| 一个项目里可能有多个 Servlet | ✅ 对 | 而且 Tomcat 默认就自带几个：DefaultServlet（处理静态资源）、JspServlet（处理 .jsp）。Spring 应用通常再加一个 DispatcherServlet |
| Servlet 是处理 HTTP 请求的组件 | ⚠️ 基本对 | 严格说 Servlet 接口本身协议无关（GenericServlet），HttpServlet 才专门处理 HTTP。习惯上说的"Servlet"都指 HttpServlet |
| DispatcherServlet 通过实现 Servlet 接口获得处理 HTTP 的能力 | ✅ 对 | 更精确：不是直接 implements Servlet，而是通过继承链间接实现（GenericServlet→HttpServlet→HttpServletBean→FrameworkServlet→DispatcherServlet）。Tomcat 看它就是"一个实现了 Servlet 接口的类"，按 Servlet 规范调它的 init/service/destroy |
| 内部使用 Spring 组件能力把请求交给 Controller | ✅ 对 | 这是核心职责，用容器里的 HandlerMapping/HandlerAdapter 等完成分发 |

### 10.2 关键一：DispatcherServlet 通常是"唯一"的业务 Servlet，Controller 不再是 Servlet

这是和"原生 Servlet 开发"最大的区别，也是初学者最容易混的点：

```
原生 Servlet 时代（Servlet 2.x 之前）：
  每个 URL 对应一个 Servlet
  /login  → LoginServlet（implements HttpServlet）
  /user   → UserServlet（implements HttpServlet）
  → 一个项目几十上百个 Servlet 类，都要自己继承 HttpServlet

Spring MVC 时代：
  整个应用通常只配一个 DispatcherServlet（在 web.xml 或 ServletRegistrationBean）
  /login  ┐
  /user   ├→ 全部进 DispatcherServlet（唯一的前门）
  /order  ┘   → 它内部用 HandlerMapping 查表，分发给 Controller 方法
  → Controller 是普通 POJO（@RestController），不实现 Servlet 接口
```

所以**项目里 Tomcat 管理的"业务 Servlet"基本就 DispatcherServlet 一个**（外加 Tomcat 自带的默认 Servlet 和 JSP Servlet）。业务 Controller 们**不是 Servlet**，它们是 Spring 容器里的 Bean，由 DispatcherServlet 通过反射调用。

这就是为什么之前讲"HandlerMethod"时强调它不是 Servlet 而是"类+方法"的封装——现代 Controller 早就不用走 Servlet 规范了。

### 10.3 关键二：这是"前端控制器"模式（Front Controller Pattern）

DispatcherServlet 这个设计在 GoF 设计模式里有个名字叫**前端控制器（Front Controller）**：

> 用一个中心控制器统一接收所有请求，做通用预处理（鉴权、日志、编码、缓存 body 等），再分发给具体的命令/处理器。

对应到我们的鉴权方案：
```
前端控制器：DispatcherServlet（唯一入口，做通用的事）
  ↓ 通用预处理
  ├─ Filter（容器层：缓存 body、跨域）
  ├─ Interceptor（Spring 层：AK/SK 鉴权、豁免判断）
  ↓ 分发
  HandlerMapping（查"URL→Controller方法"表）
  ↓ 适配执行
  HandlerAdapter（处理 @RequestBody 解析、反射调方法）
  ↓
  Controller 方法（业务逻辑）
```

**对比没有前端控制器的做法**（每个 Servlet 自己处理鉴权、参数解析、编码）：
- N 个 Servlet 各自重复写鉴权、读 body、解析参数 → 大量重复代码
- 加一个新接口要新建一个 Servlet 类、配 web.xml → 繁琐

有了 DispatcherServlet：**通用逻辑集中写一次（在 Filter/Interceptor/Adapter），业务 Controller 只关心业务**，新增接口只需加个 `@PostMapping` 方法。这正是 Spring MVC 的价值所在，也正是我们鉴权方案能成立的基础——鉴权写一次在拦截器，所有接口都受保护。

### 10.4 一个细节：两层分发

请求流转严格来说是：
```
客户端 HTTP 请求
  ↓ Tomcat 的 Connector（解析 HTTP 协议字节流）
  ↓ 构建 HttpServletRequest / HttpServletResponse 对象
  ↓ Engine → Host → Context（按域名/应用路径路由）
  ↓ 找到匹配 URL 的 Servlet（Tomcat 靠 web.xml/注解的 servlet-mapping）
  ↓ 调该 Servlet.service(request, response)
```

对 Spring 应用，被找到的 Servlet 就是 DispatcherServlet（因为配了 `<url-pattern>/</url-pattern>` 拦截几乎所有路径）。Tomcat 不管你里面多少个 Controller，它只认"DispatcherServlet 这一个 Servlet"，把请求往它身上一丢就完事——剩下的分发是 Spring MVC 自己的事。

所以表述"DispatcherServlet 处理 HTTP 请求"再精确点：**Tomcat 把请求交给 DispatcherServlet，DispatcherServlet 在 service→doDispatch 里用 Spring 组件把请求二次分发给具体 Controller 方法**。这是"两层分发"：Tomcat 第一次分发（到 Servlet），Spring MVC 第二次分发（到 Controller 方法）。

### 10.5 进阶三句话总结

核心论断——**"DispatcherServlet 实现了 Servlet 能处理 HTTP，并在内部用 Spring 组件把请求交给 Controller"**——是对的。再记住三句进阶：

1. **现代 Spring 应用通常只有一个 DispatcherServlet**，Controller 不是 Servlet 而是 POJO（`@RestController`），由 DispatcherServlet 反射调用。
2. **这是"前端控制器"模式**：集中处理通用逻辑（鉴权/日志/参数解析），业务类只写业务。
3. **两层分发**：Tomcat 按路径把请求分发给 DispatcherServlet（第一次），DispatcherServlet 按 `@RequestMapping` 把请求分发给具体 Controller 方法（第二次）。

这个理解到位后，前面讲的 Filter/Interceptor/AOP/HandlerMethod/onRefresh 整套链路就有了"为什么这样设计"的答案——都是为了支撑这个前端控制器模式，让通用逻辑和业务逻辑解耦。回到引子那个鉴权难题，答案正是：用前端控制器 + Filter 缓存 body + 拦截器鉴权 + @RequestAttribute 注入身份，通用逻辑写一次，业务接口只写业务。
