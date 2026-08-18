# 事件机制与发布订阅模式：从内存到分布式

## 从"代码耦合的痛点"说起

你是否经历过这样的场景：

1.  **用户注册后要做一堆事**：发邮件、送积分、初始化账户、推送通知……所有逻辑挤在一个方法里，改一个怕影响其他；
2.  **订单支付成功后要联动多个系统**：库存扣减、物流创建、发票生成、消息推送……每加一个下游就要改上游代码；
3.  **想异步处理但又不想引入 MQ**：只是应用内的事件通知，用消息队列太重了，有没有更轻量的方案？

这些问题的共同诉求是：**让事件的产生者和消费者解耦**。发布订阅（Pub/Sub）模式正是为此而生——发布者只管"发出事件"，不关心谁接收、怎么处理；订阅者只管"监听感兴趣的事件"，不关心谁发出的。

本篇从**内存级事件**（Guava EventBus、Spring Event）讲起，再拓展到**分布式发布订阅**（Redis Pub/Sub、消息队列），帮你建立完整的事件驱动认知体系。

---

## 一、内存级发布订阅：进程内的事件总线

内存级事件适用于**单进程内**的模块解耦，零网络开销、极低延迟，是轻量级解耦的首选方案。

### Guava EventBus：最轻量的事件总线

Google Guava 提供的 `EventBus` 是最经典的内存事件总线实现，API 极简、上手即用。

#### 快速上手

```java
// 1. 定义事件对象（普通 POJO，无需继承任何类）
public class UserRegisteredEvent {
    private final Long userId;
    private final String email;

    public UserRegisteredEvent(Long userId, String email) {
        this.userId = userId;
        this.email = email;
    }

    // getter...
}

// 2. 定义订阅者（用 @Subscribe 标注处理方法）
public class EmailService {

    @Subscribe
    public void onUserRegistered(UserRegisteredEvent event) {
        System.out.println("发送邮件给: " + event.getEmail());
    }
}

public class PointsService {

    @Subscribe
    public void onUserRegistered(UserRegisteredEvent event) {
        System.out.println("赠送新用户积分: userId=" + event.getUserId());
    }
}

// 3. 注册订阅者并发布事件
EventBus eventBus = new EventBus();
eventBus.register(new EmailService());
eventBus.register(new PointsService());

// 发布事件 → 所有匹配的 @Subscribe 方法自动执行
eventBus.post(new UserRegisteredEvent(1001L, "user@example.com"));
```

> 💡 **核心设计**：EventBus 通过**事件对象的类型**匹配订阅者。`post(UserRegisteredEvent)` 会自动找到所有参数类型为 `UserRegisteredEvent` 的 `@Subscribe` 方法，支持继承关系（父类事件的订阅者也会收到子类事件）。

#### 同步 vs 异步 EventBus

| 类型 | 创建方式 | 行为 | 适用场景 |
| :--- | :--- | :--- | :--- |
| **同步** | `new EventBus()` | 发布者线程阻塞等待所有订阅者执行完 | 需要保证顺序、事务一致性 |
| **异步** | `new AsyncEventBus(executor)` | 订阅者在独立线程池执行，发布者立即返回 | 耗时操作、不阻塞主流程 |

```java
// 异步 EventBus：订阅者在自定义线程池中执行
ExecutorService executor = Executors.newFixedThreadPool(4);
AsyncEventBus asyncBus = new AsyncEventBus(executor);
asyncBus.register(new EmailService());

// post 立即返回，邮件发送在后台线程执行
asyncBus.post(new UserRegisteredEvent(1001L, "user@example.com"));
```

> ⚠️ **注意事项**：
> - 同步 EventBus 中某个订阅者抛异常，会中断后续订阅者的执行（可用 `SubscriberExceptionHandler` 捕获）；
> - 异步 EventBus 的异常不会传播到发布者，需自行在订阅者内部处理；
> - EventBus **不是线程安全的**（同步版本），多线程发布需用 `AsyncEventBus` 或外部加锁。

#### EventBus 的工作原理

```
eventBus.post(UserRegisteredEvent)
    ↓
遍历已注册的订阅者列表
    ↓
检查每个 @Subscribe 方法的参数类型是否匹配（含继承关系）
    ↓
匹配成功 → 反射调用该方法
    ├─ 同步：当前线程直接执行
    └─ 异步：提交到 ExecutorService 执行
```

> 💡 **面试话术**："Guava EventBus 是基于观察者模式的内存事件总线，通过事件对象类型匹配订阅者。同步版本在发布者线程执行，适合事务内操作；异步版本通过线程池解耦，适合耗时任务。优点是零依赖、API 简单；缺点是仅限单进程、无持久化、无重试机制。"

### Spring Event：Spring 生态的原生事件机制

Spring Framework 内置了完善的事件机制，与 IoC 容器深度集成，是 Spring 项目中**首选的内存事件方案**。

#### 基础用法

```java
// 1. 定义事件（推荐继承 ApplicationEvent 或使用普通对象）
// 方式A：继承 ApplicationEvent（传统方式）
public class OrderPaidEvent extends ApplicationEvent {
    private final Long orderId;

    public OrderPaidEvent(Object source, Long orderId) {
        super(source);
        this.orderId = orderId;
    }
}

// 方式B：普通 POJO（Spring 4.2+ 推荐，更简洁）
public class OrderShippedEvent {
    private final Long orderId;
    private final String trackingNo;

    public OrderShippedEvent(Long orderId, String trackingNo) {
        this.orderId = orderId;
        this.trackingNo = trackingNo;
    }
}

// 2. 发布事件
@Service
public class OrderService {

    @Autowired
    private ApplicationEventPublisher publisher;

    public void payOrder(Long orderId) {
        // 业务逻辑...
        publisher.publishEvent(new OrderPaidEvent(this, orderId));
    }
}

// 3. 监听事件
@Component
public class InventoryListener {

    @EventListener  // ⭐️ Spring 4.2+ 推荐注解
    public void onOrderPaid(OrderPaidEvent event) {
        System.out.println("扣减库存: orderId=" + event.getOrderId());
    }
}
```

#### @EventListener vs ApplicationListener

| 维度 | `ApplicationListener<T>` | `@EventListener` |
| :--- | :--- | :--- |
| 写法 | 实现接口，重写 `onApplicationEvent` | 注解标注任意方法 |
| 泛型约束 | 必须指定事件类型 | 通过方法参数推断 |
| 条件过滤 | 不支持 | 支持 `condition` SpEL 表达式 |
| 多事件监听 | 一个类只能监听一种事件 | 一个方法可监听多种事件 |
| 推荐度 | ❌ 旧方式 | ✅ 推荐 |

```java
// @EventListener 高级用法
@Component
public class OrderEventListener {

    // 监听多种事件
    @EventListener({OrderPaidEvent.class, OrderShippedEvent.class})
    public void onOrderEvent(Object event) {
        log.info("收到订单事件: {}", event);
    }

    // 条件过滤：只处理金额 > 1000 的订单
    @EventListener(condition = "#event.amount > 1000")
    public void onHighValueOrderPaid(OrderPaidEvent event) {
        log.info("大额订单通知: orderId={}", event.getOrderId());
    }
}
```

#### @TransactionalEventListener：事务感知的事件监听

这是 Spring Event 最强大的特性之一——**事件监听可以与事务生命周期绑定**，避免"事务回滚了但事件已处理"的一致性问题。

```java
@Component
public class NotificationListener {

    // ⭐️ 仅在事务成功提交后才执行
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderPaidAfterCommit(OrderPaidEvent event) {
        // 这里可以安全地调用外部服务（发邮件、发短信等）
        // 因为此时数据库事务已提交，不会出现"通知发了但订单没入库"的情况
        emailService.sendPaymentConfirmation(event.getOrderId());
    }

    // 事务回滚时执行
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void onOrderPaidRollback(OrderPaidEvent event) {
        log.warn("订单支付事务回滚: orderId={}", event.getOrderId());
    }
}
```

| `TransactionPhase` | 触发时机 | 典型用途 |
| :--- | :--- | :--- |
| `BEFORE_COMMIT` | 事务提交前 | 数据校验、日志记录 |
| `AFTER_COMMIT`（默认） | 事务成功提交后 | 发通知、调外部服务 |
| `AFTER_ROLLBACK` | 事务回滚后 | 补偿操作、告警 |
| `AFTER_COMPLETION` | 事务完成后（无论成功/回滚） | 清理资源 |

> ⚠️ **关键区别**：`@EventListener` 在 `publishEvent` 时**立即执行**（同步），不管是否在事务中；`@TransactionalEventListener` 会**延迟到事务对应阶段**才执行。如果发布事件时没有活跃事务，`@TransactionalEventListener` 默认**不会执行**（可通过 `fallbackExecution = true` 改变此行为）。

#### Spring Event 的异步支持

```java
// 方式1：@Async + @EventListener（推荐）
@Component
public class AsyncOrderListener {

    @Async("taskExecutor")  // 在指定线程池中异步执行
    @EventListener
    public void onOrderPaidAsync(OrderPaidEvent event) {
        // 耗时操作，不阻塞发布者
        reportService.generateReport(event.getOrderId());
    }
}

// 方式2：配置全局异步事件处理器
@Configuration
@EnableAsync
public class EventConfig {
    @Bean("taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(100);
        return executor;
    }
}
```

> 💡 **面试话术**："Spring Event 是 Spring 原生的事件机制，通过 `ApplicationEventPublisher` 发布、`@EventListener` 监听。最大亮点是 `@TransactionalEventListener`，可以将事件处理绑定到事务生命周期，确保事务提交后再执行外部调用，避免数据不一致。配合 `@Async` 可实现异步处理，兼顾解耦与性能。"

### Guava EventBus vs Spring Event 对比

| 维度 | Guava EventBus | Spring Event |
| :--- | :--- | :--- |
| **依赖** | Guava 库 | Spring Framework |
| **与 IoC 集成** | ❌ 手动注册 | ✅ 自动扫描 Bean |
| **事务感知** | ❌ 不支持 | ✅ `@TransactionalEventListener` |
| **条件过滤** | ❌ 不支持 | ✅ SpEL `condition` |
| **异步支持** | `AsyncEventBus` | `@Async` + `@EventListener` |
| **事件继承匹配** | ✅ 支持 | ✅ 支持 |
| **适用场景** | 非 Spring 项目、简单脚本 | Spring 项目（首选） |

> 🎯 **选型建议**：Spring 项目无脑选 Spring Event；非 Spring 项目或极简场景用 Guava EventBus。

---

## 二、分布式发布订阅：跨进程的事件传递

当系统拆分为多个微服务后，内存级事件不再适用，需要**跨进程、跨网络**的发布订阅机制。

### Redis Pub/Sub：最轻量的分布式发布订阅

Redis 原生提供了 `PUBLISH/SUBSCRIBE` 命令，是实现分布式事件通知的最轻量方案。

#### 基本用法

```bash
# 订阅频道
SUBSCRIBE order.events

# 发布消息
PUBLISH order.events '{"orderId":1001,"status":"PAID"}'
```

```java
// Spring Boot + Redis Pub/Sub 示例
@Configuration
public class RedisPubSubConfig {

    @Bean
    public RedisMessageListenerContainer redisMessageListenerContainer(
            RedisConnectionFactory connectionFactory,
            MessageListenerAdapter orderEventListener) {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        // 订阅 order.events 频道
        container.addMessageListener(orderEventListener, 
                new ChannelTopic("order.events"));
        return container;
    }

    @Bean
    public MessageListenerAdapter orderEventListener(OrderEventHandler handler) {
        return new MessageListenerAdapter(handler, "onMessage");
    }
}

@Component
public class OrderEventHandler {
    public void onMessage(String message, String channel) {
        log.info("收到 Redis 消息 [{}]: {}", channel, message);
        // 处理逻辑...
    }
}

// 发布消息
@Service
public class OrderService {
    @Autowired
    private StringRedisTemplate redisTemplate;

    public void notifyOrderPaid(Long orderId) {
        redisTemplate.convertAndSend("order.events", 
                "{\"orderId\":" + orderId + ",\"status\":\"PAID\"}");
    }
}
```

#### Redis Pub/Sub 的核心特性与局限

| 特性 | 说明 |
| :--- | :--- |
| **Fire-and-Forget** | 消息发出即丢弃，**不持久化、不重试、不确认** |
| **无积压能力** | 订阅者离线期间的消息**永久丢失** |
| **广播模式** | 一条消息被所有订阅者收到，无法点对点 |
| **低延迟** | 毫秒级投递，适合实时通知 |
| **无消费组** | 不支持多消费者分摊负载 |

> ⚠️ **关键认知**：Redis Pub/Sub 是**纯通知机制**，不是消息队列。它适合"丢了也无所谓"的场景（如缓存失效通知、实时状态推送），**不适合**需要可靠投递的业务事件（如订单支付通知）。

#### Redis Streams：带持久化的发布订阅（Redis 5.0+）

如果需要可靠性，Redis 5.0 引入了 **Streams** 数据结构，弥补了 Pub/Sub 的缺陷：

| 维度 | Redis Pub/Sub | Redis Streams |
| :--- | :--- | :--- |
| 持久化 | ❌ | ✅ 消息存储在 Stream 中 |
| 消费确认 | ❌ | ✅ ACK 机制 |
| 消费组 | ❌ | ✅ Consumer Group |
| 历史消息回溯 | ❌ | ✅ 按 ID/时间范围读取 |
| 适用场景 | 实时通知、缓存失效 | 轻量级消息队列、事件溯源 |

> 💡 **选型建议**：需要可靠性且不想引入 MQ → Redis Streams；仅需实时通知 → Redis Pub/Sub；生产级业务消息 → RocketMQ/Kafka。

### 消息队列：生产级发布订阅

当业务对**可靠性、顺序性、积压能力**有要求时，应使用专业消息队列。

#### RocketMQ / Kafka 核心能力对比

| 维度 | RocketMQ | Kafka |
| :--- | :--- | :--- |
| **定位** | 业务消息队列 | 大数据流处理平台 |
| **消息可靠性** | ✅ 同步刷盘、事务消息 | ✅ 副本机制、ISR |
| **顺序消息** | ✅ 分区有序 | ✅ Partition 有序 |
| **延迟消息** | ✅ 原生支持（18 个级别） | ❌ 需自行实现 |
| **事务消息** | ✅ 半消息 + 回查 | ❌ 不支持 |
| **消息回溯** | ✅ 按时间/Offset | ✅ 按 Offset/时间 |
| **吞吐量** | 十万级 TPS | 百万级 TPS |
| **运维复杂度** | 中等 | 较高 |
| **国内生态** | ✅ 阿里开源，文档完善 | ✅ 社区活跃 |

> 🎯 **选型建议**：
> - **国内微服务业务消息** → RocketMQ（事务消息、延迟消息开箱即用）
> - **大数据/日志/流处理** → Kafka（超高吞吐、生态丰富）
> - **轻量级可靠消息** → Redis Streams（免运维、够用即可）

### 各方案全景对比

| 维度 | Guava EventBus | Spring Event | Redis Pub/Sub | Redis Streams | RocketMQ / Kafka |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **作用域** | 单进程 | 单进程 | 跨进程 | 跨进程 | 跨进程/跨机房 |
| **持久化** | ❌ | ❌ | ❌ | ✅ | ✅ |
| **可靠性** | 无 | 无 | Fire-and-Forget | ACK 确认 | 高可靠 |
| **延迟** | 纳秒级 | 纳秒级 | 毫秒级 | 毫秒级 | 毫秒级 |
| **积压能力** | 无 | 无 | 无 | 有限 | 强 |
| **事务感知** | ❌ | ✅ | ❌ | ❌ | ✅（RocketMQ） |
| **运维成本** | 零 | 零 | 低 | 低 | 中高 |
| **适用场景** | 简单解耦 | Spring 内解耦 | 实时通知 | 轻量可靠消息 | 生产级业务消息 |

---

## 三、Spring State Machine：事件驱动的状态管理

状态机是事件驱动思想的典型应用——**状态的流转由事件触发**，每个状态转换都对应一个"事件 → 动作 → 新状态"的过程。Spring State Machine 将这一模式标准化，特别适合订单、审批、支付等有明确状态流转的业务场景。

### 为什么需要状态机？

以订单为例，如果没有状态机：

```java
// ❌ 散落在各处的 if-else，难以维护
if (order.getStatus() == CREATED && event == PAY) {
    order.setStatus(PAID);
} else if (order.getStatus() == PAID && event == SHIP) {
    order.setStatus(SHIPPED);
} else if (order.getStatus() == CREATED && event == CANCEL) {
    order.setStatus(CANCELLED);
}
// ... 状态越多越混乱，容易遗漏非法转换
```

使用状态机后：

```java
// ✅ 状态流转规则集中定义，非法转换自动拒绝
stateMachine.sendEvent(Mono.just(MessageBuilder
    .withPayload(OrderEvent.PAY)
    .setHeader("orderId", orderId)
    .build()))
    .subscribe();
```

### 核心概念

| 概念 | 说明 | 示例 |
| :--- | :--- | :--- |
| **State（状态）** | 业务对象的当前阶段 | CREATED、PAID、SHIPPED、COMPLETED |
| **Event（事件）** | 触发状态流转的动作 | PAY、SHIP、CONFIRM、CANCEL |
| **Transition（转换）** | 状态 + 事件 → 新状态的映射 | CREATED + PAY → PAID |
| **Guard（守卫）** | 转换的前置条件，返回 false 则阻止转换 | 库存充足才允许发货 |
| **Action（动作）** | 转换过程中执行的业务逻辑 | 扣减库存、发送通知 |

### 快速上手：订单状态机

#### Step 1: 定义状态和事件枚举

```java
public enum OrderState {
    CREATED, PAID, SHIPPED, DELIVERED, COMPLETED, CANCELLED
}

public enum OrderEvent {
    PAY, SHIP, DELIVER, CONFIRM, CANCEL
}
```

#### Step 2: 配置状态机

```java
@Configuration
@EnableStateMachine
public class OrderStateMachineConfig 
        extends EnumStateMachineConfigurerAdapter<OrderState, OrderEvent> {

    @Override
    public void configure(StateMachineStateConfigurer<OrderState, OrderEvent> states) 
            throws Exception {
        states
            .withStates()
            .initial(OrderState.CREATED)           // 初始状态
            .end(OrderState.COMPLETED)             // 终态
            .end(OrderState.CANCELLED)             // 终态
            .states(EnumSet.allOf(OrderState.class));
    }

    @Override
    public void configure(
            StateMachineTransitionConfigurer<OrderState, OrderEvent> transitions) 
            throws Exception {
        transitions
            // CREATED → PAID（支付）
            .withExternal()
                .source(OrderState.CREATED).target(OrderState.PAID)
                .event(OrderEvent.PAY)
                .action(context -> log.info("订单已支付: {}", 
                    context.getMessageHeaders().get("orderId")))
                .and()
            // PAID → SHIPPED（发货）
            .withExternal()
                .source(OrderState.PAID).target(OrderState.SHIPPED)
                .event(OrderEvent.SHIP)
                .guard(context -> inventoryService.hasStock(
                    (Long) context.getMessageHeaders().get("orderId")))
                .action(context -> log.info("订单已发货"))
                .and()
            // SHIPPED → DELIVERED（送达）
            .withExternal()
                .source(OrderState.SHIPPED).target(OrderState.DELIVERED)
                .event(OrderEvent.DELIVER)
                .and()
            // DELIVERED → COMPLETED（确认收货）
            .withExternal()
                .source(OrderState.DELIVERED).target(OrderState.COMPLETED)
                .event(OrderEvent.CONFIRM)
                .and()
            // CREATED → CANCELLED（取消）
            .withExternal()
                .source(OrderState.CREATED).target(OrderState.CANCELLED)
                .event(OrderEvent.CANCEL)
                .and()
            // PAID → CANCELLED（退款取消）
            .withExternal()
                .source(OrderState.PAID).target(OrderState.CANCELLED)
                .event(OrderEvent.CANCEL)
                .action(context -> refundService.refund(
                    (Long) context.getMessageHeaders().get("orderId")));
    }
}
```

#### Step 3: 使用状态机

`sendEvent` 有多种重载，**不是必须用 Mono**。根据项目类型选择合适的方式：

```java
@Service
public class OrderService {

    @Autowired
    private StateMachine<OrderState, OrderEvent> stateMachine;

    // ✅ 方式1：直接发送 Message（推荐，Spring MVC 项目最常用）
    public void payOrder(Long orderId) {
        boolean accepted = stateMachine.sendEvent(MessageBuilder
                .withPayload(OrderEvent.PAY)
                .setHeader("orderId", orderId)
                .build());
        if (accepted) {
            log.info("状态流转成功");
        } else {
            log.warn("状态流转被拒绝，当前状态不允许该事件");
        }
    }

    // ✅ 方式2：只发事件枚举（最简单，不携带额外数据时）
    public void confirmDelivery(Long orderId) {
        stateMachine.sendEvent(OrderEvent.CONFIRM);
    }

    // ✅ 方式3：响应式 Mono（仅 WebFlux 响应式项目需要）
    public Mono<StateMachineResult> payOrderReactive(Long orderId) {
        return stateMachine.sendEvent(Mono.just(MessageBuilder
                .withPayload(OrderEvent.PAY)
                .setHeader("orderId", orderId)
                .build()));
    }
}
```

> ⚠️ **澄清**：Mono 写法是 Spring State Machine 为响应式编程提供的 API，**普通 Spring MVC 项目直接用 `sendEvent(Message)` 即可**，返回 boolean 表示是否接受。不需要 `.subscribe()`、不需要 `doOnNext`，代码更简洁直观。

#### 自定义事件类型

事件不限于枚举，可以是任意对象：

```java
// 带数据的自定义事件类
public class PayEvent {
    private final Long orderId;
    private final BigDecimal amount;
    // constructor + getter...
}

// 配置中按类匹配
.withExternal()
    .source(OrderState.CREATED).target(OrderState.PAID)
    .event(PayEvent.class)
    .action(context -> {
        PayEvent payEvent = (PayEvent) context.getMessage().getPayload();
        log.info("支付金额: {}", payEvent.getAmount());
    })
```

> 💡 **最佳实践**：事件本身保持轻量（只放触发标识），业务数据通过 Message Header 传递或处理时重新查询数据库，避免"数据过期覆盖"问题（详见 Q4 实战踩坑）。

### Guard 与 Action 的区别

| 维度 | Guard（守卫） | Action（动作） |
| :--- | :--- | :--- |
| **作用** | 判断是否允许转换 | 转换时执行业务逻辑 |
| **返回值** | boolean（true=允许，false=阻止） | void |
| **执行时机** | 转换前 | 转换中 |
| **失败影响** | 阻止整个转换 | 不影响状态流转（异常需自行处理） |
| **典型用途** | 权限校验、库存检查、余额验证 | 扣库存、发通知、写日志 |

### 持久化状态机：两种方案与选型

生产环境中，状态机的当前状态需要持久化到数据库。有两种主流方案，**大多数场景用 Action 就够了**。

#### 方案一：Action 里手动持久化（✅ 推荐，90% 场景适用）

直接在 Transition 的 Action 中完成业务操作和状态更新，简单直观：

```java
.withExternal()
    .source(OrderState.CREATED).target(OrderState.PAID)
    .event(OrderEvent.PAY)
    .action(context -> {
        Long orderId = context.getMessageHeaders().get("orderId", Long.class);
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.setStatus("PAID");
        order.setPaidTime(LocalDateTime.now());
        orderRepository.save(order); // 状态和业务数据一起更新
    })
```

**优点**：灵活、与业务逻辑紧密结合、易于理解和调试。

#### 方案二：StateMachinePersist 框架级持久化（⚠️ 高级场景）

实现 `StateMachinePersist` 接口，由框架在状态变更时自动调用 `read/write`：

```java
@Component
public class OrderStateMachinePersister 
        implements StateMachinePersist<OrderState, OrderEvent, Long> {

    @Autowired
    private OrderRepository orderRepository;

    @Override
    public void write(StateMachineContext<OrderState, OrderEvent> context, Long orderId) 
            throws Exception {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.setStatus(context.getState().name());
        orderRepository.save(order);
    }

    @Override
    public StateMachineContext<OrderState, OrderEvent> read(Long orderId) 
            throws Exception {
        Order order = orderRepository.findById(orderId).orElseThrow();
        return new DefaultStateMachineContext<>(
                OrderState.valueOf(order.getStatus()), null, null, null);
    }
}
```

**它解决的是 Action 无法覆盖的问题**：

- **状态变更自动同步**：无论通过哪个 Transition 改变状态，`write()` 都自动触发，无需在每个 Action 里重复写 save
- **异常恢复**：Action 执行失败时内存状态可能已变但数据库未更新，下次请求 `read()` 能恢复到一致状态

#### 两种方案对比

| 维度 | Action 手动持久化 | StateMachinePersist |
| :--- | :--- | :--- |
| **写法** | Action lambda 中直接调 Repository | 实现接口，框架自动调用 |
| **触发时机** | Transition 的 Action 执行时 | 状态机状态变更时自动 read/write |
| **灵活性** | ✅ 高，可结合业务做条件更新 | ⚠️ 低，只做状态的读写 |
| **与业务耦合** | 状态更新和业务操作在一起 | 状态管理与业务逻辑分离 |
| **适用场景** | 大多数业务场景（推荐） | 跨请求状态恢复、状态机实例复用 |

> 🎯 **选型建议**：**Action 里持久化是"业务驱动"，StateMachinePersist 是"框架驱动"**。90% 的场景用 Action 就够了，只有需要状态机实例复用或跨请求状态恢复时才用 StateMachinePersist。

> 💡 **面试话术**："Spring State Machine 将业务状态流转建模为有限状态机，通过 State + Event → Transition 的声明式配置替代散落的 if-else。支持 Guard 前置校验、Action 业务动作、状态持久化等能力。持久化有两种方式：Action 里手动更新适合大多数场景；StateMachinePersist 框架级持久化适合需要跨请求恢复状态的高级场景。核心价值是将状态流转规则集中管理，非法转换自动拒绝，避免状态不一致。"

### 状态机 vs 简单事件驱动的关系

状态机本质上是**带约束的事件驱动**：

| 维度 | 普通事件驱动 | 状态机 |
| :--- | :--- | :--- |
| 事件处理 | 无状态约束，任何时刻都可处理 | 只有当前状态允许的转换才能触发 |
| 非法操作 | 需在业务代码中手动校验 | 框架自动拒绝非法转换 |
| 可视化 | ❌ 无法直观看到全貌 | ✅ 状态流转图一目了然 |
| 适用场景 | 通用解耦 | 有明确生命周期的业务对象 |

> 🎯 **选型建议**：如果业务对象有 3 个以上状态且存在复杂的流转规则，优先考虑状态机；如果只是简单的"做完 A 再做 B"，用普通事件驱动即可，不必过度设计。并且如果一个业务的状态过多你需要更优先考虑的是降低复杂度，即“你是否真的需要这么多状态？“

---

## 四、面试高频问题速查

### Q1：Spring Event 和 @TransactionalEventListener 有什么区别？

**答**：`@EventListener` 在 `publishEvent` 时**立即同步执行**，不管当前是否在事务中；`@TransactionalEventListener` 会**延迟到事务的指定阶段**（默认 AFTER_COMMIT）才执行。核心区别在于：如果事务最终回滚了，`@EventListener` 的处理已经执行完毕无法撤销，而 `@TransactionalEventListener(AFTER_COMMIT)` 根本不会执行，避免了"通知发了但数据没落库"的一致性问题。

### Q2：Redis Pub/Sub 和消息队列有什么区别？什么时候该用哪个？

**答**：Redis Pub/Sub 是 Fire-and-Forget 的通知机制，消息不持久化、不重试、订阅者离线期间消息丢失；消息队列（RocketMQ/Kafka）提供持久化、ACK 确认、消费组、消息回溯等可靠投递能力。**选型原则**：丢了也无所谓的实时通知（如缓存失效、在线状态推送）用 Redis Pub/Sub；涉及业务流程、不允许丢失的事件（如订单支付、库存扣减）必须用消息队列。

### Q3：为什么 Spring 项目中推荐 Spring Event 而不是 Guava EventBus？

**答**：三个原因：① **IoC 集成**：Spring Event 自动扫描 Bean 中的 `@EventListener`，无需手动注册；② **事务感知**：`@TransactionalEventListener` 可绑定事务生命周期，Guava 完全不支持；③ **条件过滤**：支持 SpEL `condition` 表达式，可按业务条件选择性监听。Guava EventBus 仅适合非 Spring 项目或极简脚本场景。

### Q4：异步事件处理有哪些坑需要注意？

**答**：五个关键点：① **异常隔离**：异步事件的异常不会传播到发布者，必须在订阅者内部 try-catch 并记录日志；② **线程池配置**：避免使用默认线程池，应根据业务量配置独立的线程池，防止影响其他异步任务；③ **事务边界**：`@Async` + `@EventListener` 的组合不在原事务中执行，如需事务一致性应改用 `@TransactionalEventListener(AFTER_COMMIT)`；④ **幂等设计**：异步事件可能因重试被重复消费，订阅者必须做幂等处理；⑤ **禁止直接使用事件数据更新数据库**（⚠️ 线上高频 Bug）。

> ⚠️ **实战踩坑：异步事件的"数据过期覆盖"问题**
>
> **场景还原**：某开发者在异步事件监听器中，直接拿 Event 对象携带的数据去更新数据库。由于异步事件在独立线程池中执行，当线程池压力大时，事件处理延迟了数秒。在这段延迟期间，该条数据已被其他业务修改过，但事件监听器仍用旧的 Event 数据执行 UPDATE，导致**新数据被旧数据覆盖**。
>
> ```java
> // ❌ 错误写法：直接用事件中的快照数据更新
> @Async
> @EventListener
> public void onOrderUpdated(OrderUpdatedEvent event) {
>     // event 中的数据是发布时刻的快照，处理时可能已过期
>     orderRepository.save(event.getOrder()); // 💥 覆盖了中间其他人的修改！
> }
>
> // ✅ 正确写法：只传 ID，处理时重新查询最新数据再判断更新
> @Async
> @EventListener
> public void onOrderUpdated(OrderUpdatedEvent event) {
>     Order latest = orderRepository.findById(event.getOrderId()).orElseThrow();
>     // 基于最新数据做条件判断，避免盲目覆盖
>     if (shouldUpdate(latest, event)) {
>         latest.setStatus(event.getTargetStatus());
>         orderRepository.save(latest);
>     }
> }
> ```
>
> **核心原则**：**异步事件中只传递标识符（ID），不传递完整数据快照**。处理时重新查询最新状态，基于最新数据做条件判断后再决定是否更新。这条原则同样适用于 MQ 消费者——消息体中的数据永远是"过去时"，消费时必须以数据库当前状态为准。

### Q5：发布订阅模式和观察者模式有什么区别？

**答**：两者思想相同，但实现层次不同。**观察者模式**是设计模式层面的概念，强调 Subject-Observer 的直接引用关系；**发布订阅模式**是架构层面的实现，通过中间件（EventBus/MQ）解耦发布者和订阅者，双方互不知道对方的存在。可以说发布订阅是观察者模式在分布式系统中的工程化落地。

### Q6：Spring State Machine 和普通 if-else 状态判断相比有什么优势？

**答**：三个核心优势：① **集中管理**：所有状态流转规则在一个配置类中声明式定义，而非散落在业务代码各处；② **非法转换自动拒绝**：未定义的 State+Event 组合会被框架拦截，避免状态不一致；③ **可扩展**：通过 Guard 和 Action 将校验逻辑与业务动作分离，新增状态只需加配置不改业务代码。缺点是引入框架有一定学习成本，简单场景（2-3 个状态）用 if-else 更直接。

---

## 小结

本篇覆盖了事件机制与发布订阅的完整知识体系：

- **内存级**：Guava EventBus（轻量、非 Spring）、Spring Event（事务感知、IoC 集成）
- **分布式轻量级**：Redis Pub/Sub（实时通知）、Redis Streams（轻量可靠消息）
- **生产级**：RocketMQ（业务消息）、Kafka（大数据流处理）
- **选型核心**：根据**作用域、可靠性要求、运维成本**三维度选择合适方案

> 💡 **记忆口诀**：**进程内用 Spring Event，跨进程看可靠性；通知用 Redis Pub/Sub，业务消息上 MQ；事务感知是关键，异步幂等不能忘**。
