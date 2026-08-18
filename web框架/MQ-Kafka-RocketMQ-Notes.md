# 分布式消息队列：Kafka 与 RocketMQ 核心原理与实战

## 从"为什么需要消息队列"说起

在微服务架构中，消息队列（MQ）是解耦、削峰、异步通信的基础设施。你是否经历过这些场景：

1.  **订单支付成功后要联动 5 个下游系统**：每加一个下游就要改上游代码，耦合严重；
2.  **秒杀流量瞬间打满数据库**：没有缓冲层，系统直接崩溃；
3.  **跨服务调用超时导致数据不一致**：同步调用链路太长，任何一个节点故障都影响整体。

消息队列的核心价值：**异步解耦、流量削峰、最终一致性**。本篇聚焦 Kafka 和 RocketMQ 两大主流 MQ，按"**原理 → 对比 → 实战 → 面试**"的线性路径组织，方便阅读和复习。

---

## 一、Kafka 核心原理

### 核心架构

```
Producer ──► Broker Cluster ──► Consumer Group
                │
        ┌───────┼───────┐
        ▼       ▼       ▼
    Partition0 Partition1 Partition2   ← Topic 的物理分片
        │       │       │
      Replica Replica Replica          ← 每个 Partition 有 Leader + Follower
```

| 概念 | 说明 |
| :--- | :--- |
| **Topic** | 逻辑分类，类似数据库表 |
| **Partition** | Topic 的物理分片，有序、不可变的消息序列 |
| **Replica** | Partition 的副本，Leader 负责读写，Follower 同步备份 |
| **Consumer Group** | 一组消费者共同消费一个 Topic，组内分区互斥 |
| **Offset** | 消息在 Partition 中的唯一序号，消费者通过 Offset 记录消费进度 |

### 1. 分区并行：吞吐量的基石

- 一个 Topic 分为 N 个 Partition，每个 Partition 可被不同 Broker 承载
- Producer 按 Key Hash 或轮询写入不同 Partition，实现**写并行**
- Consumer Group 内每个消费者负责若干 Partition，实现**读并行**
- **吞吐量 ≈ 单 Partition 吞吐 × Partition 数量**

### 2. 顺序写磁盘：写入性能的基石

- 消息追加写入 Partition 文件（顺序 IO），避免随机写的磁头寻道开销
- 同等硬件下，顺序写吞吐可达数百 MB/s，远超随机写

### 3. 零拷贝（sendfile）：读取性能的杀手锏

> ⚠️ **常见误区澄清**：零拷贝发生在 **Broker 发送消息给 Consumer** 的阶段（出站），不是 Consumer 接收消息的阶段（入站）。Consumer 收消息走的是标准网络 IO，没有零拷贝。

#### 传统文件传输 vs sendfile 零拷贝

**传统方式（4 次拷贝 + 4 次上下文切换）**：

```
磁盘 ──CPU──► 内核缓冲区 ──CPU──► 用户态缓冲区 ──CPU──► Socket缓冲区 ──CPU──► 网卡
         拷贝1              拷贝2                拷贝3               拷贝4
```

**sendfile 零拷贝（2 次拷贝 + 2 次上下文切换）**：

```
磁盘 ──CPU──► 内核缓冲区(PageCache) ──DMA──► 网卡
         拷贝1(不可避免)            拷贝2(DMA引擎完成，CPU不参与!)
```

> 💡 **"零"的含义**：不是完全没有拷贝，而是**消除了用户态与内核态之间的数据拷贝**。数据全程在内核空间流转，CPU 不再充当"搬运工"。

#### Kafka 如何使用 sendfile？

Kafka 的 Consumer 拉取消息时，底层调用 Java NIO 的 `FileChannel.transferTo()`（对 Linux `sendfile` 的封装）：

```java
// Kafka 源码简化示意
// 数据直接从 PageCache → 网卡，不经过 JVM 堆内存
fileChannel.transferTo(position, count, socketChannel);
```

**关键前提**：消息以**原始字节**存储，不做反序列化/加密/压缩解压等处理。如果需要对消息做变换，就必须读到用户态，无法使用零拷贝。

#### 为什么零拷贝只在发送端？

| 维度 | 发送端（Broker） | 接收端（Consumer） |
| :--- | :--- | :--- |
| **数据来源** | 本地磁盘文件 | 网络 Socket |
| **能否用 sendfile** | ✅ 可以，文件→网卡直接传 | ❌ 不行，sendfile 只支持文件描述符作为源 |
| **传统瓶颈** | 4 次拷贝 + 4 次上下文切换 | 2 次拷贝（已经很少） |
| **优化收益** | 巨大 | 有限 |

`sendfile` 的语义是"把文件内容直接发送到 Socket"，要求源必须是文件描述符，所以只能用在**有文件要发**的场景。

#### 顺序写 vs 零拷贝：互补而非同一件事

| 特性 | 顺序写磁盘 | sendfile 零拷贝 |
| :--- | :--- | :--- |
| **优化对象** | 写入性能 | 读取/传输性能 |
| **原理** | 避免磁头寻道，利用顺序 IO 高带宽 | 消除用户态↔内核态拷贝，CPU 不参与搬运 |
| **发生阶段** | Producer → Broker | Broker → Consumer |

两者互补：顺序写让写入快，零拷贝让读取快，共同成就 Kafka 的高吞吐。

> 💡 **面试话术**："Kafka 的零拷贝发生在 Broker 向 Consumer 传输消息的阶段，通过 Linux sendfile 系统调用实现。传统方式需要 4 次拷贝和 4 次上下文切换；sendfile 将拷贝减少到 2 次，且第二次由 DMA 引擎完成，CPU 完全不参与。前提是消息以原始字节存储、不做应用层处理。注意零拷贝优化的是发送端效率，不是接收端。"

### 4. 副本机制：高可用保障

- 每个 Partition 配置 N 个副本（`replication.factor`）
- Leader 处理读写，Follower 从 Leader 拉取数据同步
- Leader 宕机时，ISR（In-Sync Replicas）中的 Follower 自动选举为新 Leader
- `min.insync.replicas` 控制最少同步副本数，配合 `acks=all` 保证数据不丢

### 5. 消费者组与 Rebalance

- 同一 Consumer Group 内的消费者**分摊** Topic 的所有 Partition
- 消费者加入/退出/宕机时触发 **Rebalance**，重新分配 Partition
- Rebalance 期间该 Group **暂停消费**，是 Kafka 的主要痛点之一

### 6. Pull 模式：Consumer 自主控制消费节奏

Kafka 采用标准的 **Pull（拉）模式**，而非 Push（推）模式。这是核心设计决策之一。

#### 为什么选择 Pull 而不是 Push？

| 维度 | Push 模式 | Pull 模式（Kafka） |
| :--- | :--- | :--- |
| **消费节奏** | Broker 决定推送速度 | Consumer 自主控制拉取速度 |
| **慢消费者** | 容易被压垮或消息丢失 | 按需拉取，不会被压垮 |
| **批量效率** | 难以凑批，逐条推送 | Consumer 可一次拉取多条，减少网络开销 |
| **Broker 负担** | 需维护每个消费者的推送状态 | 无状态，只需响应拉取请求 |
| **流量控制** | 复杂，需背压机制 | 天然背压，Consumer 不拉就不发 |

> 💡 **核心思想**：让消费者适配自己的处理能力，而不是让 Broker 去猜消费者能处理多快。

#### Pull 模式的工作流程

```
Consumer                          Broker
   │                                │
   │ 1. poll(timeout)               │
   ├───────────────────────────────►│
   │                                │ 2. 从 Partition 读取消息
   │                                │    (sendfile 零拷贝)
   │ 3. 返回一批消息                 │
   │◄───────────────────────────────┤
   │                                │
   │ 4. 处理消息                     │
   │                                │
   │ 5. commitSync() 提交 Offset    │
   ├───────────────────────────────►│
   │                                │
   │ 6. 下一次 poll() ...           │
```

- `poll()` 是阻塞等待的，没有消息时等到超时或有新消息到达
- 每次 `poll()` 可返回多条消息（由 `max.poll.records` 控制），减少网络往返
- Consumer 通过 Offset 记录消费进度，下次 `poll()` 从上次位置继续拉取

#### Long Polling：兼顾低延迟

Kafka 的 `poll()` 并非"傻等+轮询"，而是使用了 **Long Polling** 技术。`poll(1000ms)` 中的 `1000ms` 是**最大等待时间**，含义是：

> "Broker，请给我新消息。如果有，立即返回；如果没有，最多等我 1000ms，期间有新消息就立刻返回，超过 1000ms 还没消息就返回空结果。"

| 场景 | 行为 | 耗时 |
| :--- | :--- | :--- |
| Broker 已有新消息 | **立即返回**消息 | ≈ 0ms |
| Broker 暂无消息，但在 1000ms 内产生了新消息 | **立即返回**新消息 | < 1000ms |
| Broker 在 1000ms 内始终没有新消息 | 返回**空结果** | = 1000ms |

**为什么不设为 0 或无穷大？**

| 值 | 问题 |
| :--- | :--- |
| `poll(0)` | 纯轮询，无消息时立即返回空，CPU 空转浪费资源 |
| `poll(∞)` | 永远阻塞，Consumer 无法做其他事（如心跳、优雅关闭） |
| `poll(1000ms)` | ✅ 平衡点：无消息时不浪费 CPU，有消息时低延迟，还能定期执行心跳等维护操作 |

> ⚠️ **区分两个易混淆参数**：
> - `poll(timeout)`：单次拉取的**最大等待时间**，控制 Long Polling 超时
> - `max.poll.interval.ms`：两次 `poll()` 之间的**最大处理间隔**，超过此值未调用下一次 poll()，Broker 认为消费者宕机，触发 Rebalance
>
> 简单记忆：`poll(1000ms)` 是"等多久拿消息"，`max.poll.interval.ms` 是"多久必须来拿一次消息，否则踢出群组"。

> 💡 **面试话术**："Kafka 采用 Pull 模式，Consumer 主动调用 poll() 拉取消息。选择 Pull 的原因是：Consumer 可按自身处理能力控制消费节奏，避免慢消费者被压垮；同时支持批量拉取提升吞吐。配合 Long Polling 技术，poll(timeout) 表示最大等待时间，有消息立即返回、无消息最多等到超时，兼顾了低延迟和资源效率。"

### 7. 消费慢导致 Rebalance 的最佳实践

当消息处理耗时超过 `max.poll.interval.ms` 时，消费者会被踢出 Consumer Group 触发 Rebalance。**不建议在 poll 循环内开多线程后立即 commit**，正确做法如下：

#### ❌ 错误做法：异步处理 + 立即提交 Offset

```java
// 💥 Offset 提交时异步任务可能还没完成 → 消息丢失！
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    for (ConsumerRecord<String, String> record : records) {
        executor.submit(() -> process(record)); // 异步处理
    }
    consumer.commitSync(); // 提交时任务可能未完成！
}
```

| 问题 | 说明 |
| :--- | :--- |
| **Offset 提交时机错乱** | commitSync() 在异步任务完成前执行，任务失败则消息丢失 |
| **顺序性破坏** | 同一 Partition 的消息被多线程并发处理 |
| **背压失效** | poll 拉 500 条全丢进线程池，Consumer 继续 poll，线程池积压 OOM |

#### ✅ 方案一：调参 + 优化业务逻辑（首选，90% 场景）

```java
// 减少单次拉取量，缩短单次处理时间
props.put("max.poll.records", 100);       // 从默认500降到100
props.put("max.poll.interval.ms", 600000); // 增大容忍度到10分钟

// 优化业务逻辑：批量写DB、缓存热点数据、去掉不必要的同步调用
```

#### ✅ 方案二：多消费者实例水平扩展（最推荐的生产方案）

```
原来: 1 个消费者处理 4 个 Partition → 串行处理
现在: 4 个消费者各处理 1 个 Partition → 并行处理，天然有序
```

| 维度 | 单消费者多线程 | 多消费者实例 |
| :--- | :--- | :--- |
| 顺序性 | ❌ 同 Partition 乱序 | ✅ 同 Partition 仍有序 |
| Offset 安全 | ⚠️ 需手动管理 | ✅ 框架自动管理 |
| 背压 | ❌ 容易 OOM | ✅ 天然背压 |
| 容错 | ❌ 进程挂全丢 | ✅ Rebalance 自动恢复 |

> 🎯 **选型建议**：处理慢 → 先调 `max.poll.records` + 优化业务；整体吞吐不够 → 加消费者实例；永远不要在 poll 循环里无脑 submit 到线程池然后立即 commit。

### Kafka 常用参数配置速查

#### Producer 端

| 参数 | 默认值 | 说明 | 生产建议 |
| :--- | :--- | :--- | :--- |
| `acks` | 1 | 确认机制：0=不等待，1=Leader确认，all=所有ISR确认 | 重要消息用 `all` |
| `retries` | 0 | 发送失败重试次数 | 设 3-5，配合幂等 |
| `enable.idempotence` | false | 开启幂等，防止重试导致重复 | ✅ 生产必开 |
| `batch.size` | 16KB | 批量发送大小 | 可调大到 64-128KB 提升吞吐 |
| `linger.ms` | 0 | 等待凑批时间 | 设 5-20ms 平衡延迟与吞吐 |
| `compression.type` | none | 压缩算法 | 推荐 lz4/snappy |

#### Consumer 端

| 参数 | 默认值 | 说明 | 生产建议 |
| :--- | :--- | :--- | :--- |
| `group.id` | - | 消费者组ID | **必须设置**，否则无法提交 Offset |
| `auto.offset.reset` | latest | 无初始 Offset 时的策略 | 新业务用 earliest，防丢消息 |
| `enable.auto.commit` | true | 自动提交 Offset | ❌ 生产关闭，手动提交 |
| `max.poll.records` | 500 | 单次 poll 最大条数 | 根据处理能力调整 |
| `session.timeout.ms` | 45s | 心跳超时，超时视为宕机 | 网络差时适当调大 |
| `heartbeat.interval.ms` | 3s | 心跳间隔 | 通常为 session.timeout 的 1/3 |
| `max.poll.interval.ms` | 5min | 两次 poll 最大间隔 | 处理慢时调大，避免误判宕机 |

#### Broker 端

| 参数 | 默认值 | 说明 | 生产建议 |
| :--- | :--- | :--- | :--- |
| `num.partitions` | 1 | 新建 Topic 默认分区数 | 根据预估 QPS 设置 |
| `default.replication.factor` | 1 | 默认副本数 | ≥3 |
| `min.insync.replicas` | 1 | 最少同步副本 | 设为 replication.factor - 1 |

---

## 二、RocketMQ 核心原理与独有特性

### 核心架构

```
Producer ──► NameServer ──► Broker ──► Consumer
                │               │
           路由注册中心      Master + Slave
                            (Topic → Queue)
```

| 概念 | 对应 Kafka | 说明 |
| :--- | :--- | :--- |
| **Topic** | Topic | 逻辑分类 |
| **Queue** | Partition | Topic 下的消息队列，默认 4 个 |
| **Message** | Record | 消息体 |
| **Consumer Group** | Consumer Group | 消费者组 |
| **NameServer** | ZooKeeper/KRaft | 轻量级路由注册中心 |
| **Broker** | Broker | 消息存储与服务节点 |

### 独有特性 1：事务消息

RocketMQ 原生支持**半消息 + 回查**机制，解决"本地事务与消息发送的一致性"问题：

```
Producer                          Broker                         Consumer
   │                                │                               │
   │ 1. 发送半消息                   │                               │
   ├───────────────────────────────►│                               │
   │ 2. 半消息存储成功（对Consumer不可见）│                              │
   │◄───────────────────────────────┤                               │
   │                                │                               │
   │ 3. 执行本地事务                  │                               │
   │   (如：扣减库存)                 │                               │
   │                                │                               │
   │ 4a. 本地事务成功 → Commit        │                               │
   ├───────────────────────────────►│                               │
   │                                │ 5. 消息对Consumer可见            │
   │                                ├──────────────────────────────►│
   │                                │                               │
   │ 4b. 本地事务失败 → Rollback      │                               │
   ├───────────────────────────────►│                               │
   │                                │ 5. 半消息删除                    │
   │                                │                               │
   │ 4c. 状态未知 → Broker回查        │                               │
   │◄───────────────────────────────┤                               │
   │ 返回 Commit/Rollback           │                               │
   ├───────────────────────────────►│                               │
```

> 💡 **核心价值**：解决了"先发消息还是先执行本地事务"的两难问题。Kafka 不支持此特性，需自行实现基于数据库的事务消息表。

### 独有特性 2：死信队列（DLQ）

当消息消费失败超过最大重试次数后，自动进入死信队列，避免无限重试阻塞正常消费：

```
消费失败 → 重试（最多16次，间隔递增）→ 仍失败 → 进入 %DLQ%{ConsumerGroup}
```

| 重试次数 | 间隔 |
| :--- | :--- |
| 1-3 | 10s、30s、1min |
| 4-8 | 2min、3min、4min、5min、6min |
| 9-16 | 7min、8min、9min、10min、20min、30min、1h、2h |

> ⚠️ **注意**：16 次是默认值。如果配置了 `maxReconsumeTimes = 3`，则以配置值为准，3 次失败后即进入死信队列。Kafka 没有原生死信队列，需自行实现。

### 独有特性 3：延迟消息

RocketMQ 原生支持 18 个级别的延迟消息：

| Level | 1 | 2 | 3 | 4 | 5 | ... | 18 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 延迟 | 1s | 5s | 10s | 30s | 1min | ... | 2h |

> 💡 **典型场景**：订单超时未支付自动取消、延迟评价提醒、定时任务触发。Kafka 不支持原生延迟消息。

### 独有特性 4：消息回溯

RocketMQ 支持按时间点重置消费位点，重新消费历史消息：

```bash
mqadmin resetOffsetByTimestamp -g ORDER_CONSUMER_GROUP -t ORDER_TOPIC -s 2026-08-18T10:00:00
```

> 💡 **典型场景**：代码 Bug 修复后重消费、数据迁移、审计追溯。Kafka 也支持按 Offset/时间回溯，但 RocketMQ 的管理工具更友好。

---

## 三、Kafka vs RocketMQ 全景对比

### 功能对比

| 维度 | Kafka | RocketMQ |
| :--- | :--- | :--- |
| **定位** | 大数据流处理平台 | 业务消息队列 |
| **吞吐量** | 百万级 TPS | 十万级 TPS |
| **事务消息** | ❌ 不支持 | ✅ 原生支持 |
| **延迟消息** | ❌ 不支持 | ✅ 18 级延迟 |
| **死信队列** | ❌ 需自行实现 | ✅ 原生支持 |
| **消息回溯** | ✅ 按 Offset/时间 | ✅ 按时间，工具更友好 |
| **顺序消息** | ✅ Partition 有序 | ✅ Queue 有序 |
| **消息过滤** | ⚠️ 仅按 Topic | ✅ Tag + SQL92 |
| **运维复杂度** | 较高（ZK/KRaft） | 中等（NameServer 轻量） |
| **国内生态** | 社区活跃 | 阿里开源，文档完善 |
| **适用场景** | 日志、大数据、流处理 | 电商、金融、业务消息 |

### 消费者管理差异（⚠️ 重要区别）

RocketMQ **没有** Kafka 的 `max.poll.interval.ms` 机制，处理再慢也不会被踢出消费者组：

| 维度 | Kafka | RocketMQ |
| :--- | :--- | :--- |
| **健康检查指标** | 两次 `poll()` 间隔 + 心跳 | 仅 TCP 连接是否存活 |
| **处理超时检测** | `max.poll.interval.ms`：超时 → 踢出 → Rebalance | ❌ 无此概念，处理多久都不会被踢 |
| **慢消费者后果** | 被踢出 → Rebalance → 全组暂停消费 | 不会被踢，只是消费进度落后 |
| **Rebalance 触发条件** | 心跳超时 / poll 超时 / 成员变更 | 仅消费者上下线 / Topic 队列变更 |
| **GC 停顿 30s** | ⚠️ 心跳超时可能被踢 | ✅ TCP 没断就没事 |
| **单条消息处理 5 分钟** | ⚠️ 可能触发 poll 超时被踢 | ✅ 完全没问题 |

> 💡 **核心原因**：Kafka 用"两次 poll 间隔"作为健康检查，所以有 `max.poll.interval.ms`；RocketMQ 只用"TCP 连接存活"作为健康检查。**这也是 RocketMQ 对慢消费者更友好的原因之一**。

### 并行度与消费者扩展约束

> ⚠️ **Kafka 和 RocketMQ 通用约束**：消费者数量不能超过 Partition/Queue 数量，多余的消费者会空闲。扩容消费能力的正确顺序是：**先扩 Partition/Queue，再加消费者实例**。直接加消费者而不扩并行单元，新增节点只会空闲浪费资源。
>
> 💡 **扩容安全性**：扩容 Partition/Queue **不影响已有消息**。已有消息仍保留在原 Partition/Queue 中正常消费，新 Partition/Queue 只接收扩容后产生的新消息。因此扩容操作是在线安全的，无需停机或迁移历史数据。

### 选型建议

- **国内微服务业务消息** → RocketMQ（事务消息、延迟消息、死信队列开箱即用）
- **大数据/日志/流处理** → Kafka（超高吞吐、生态丰富）
- **两者共存**：业务消息走 RocketMQ，日志/埋点走 Kafka

---

## 四、Spring Boot 集成实战

### Kafka 集成

#### Step 1: 添加依赖与配置

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      acks: all
      retries: 3
      properties:
        enable.idempotence: true          # 开启幂等
        max.in.flight.requests.per.connection: 5
    consumer:
      group-id: order-consumer-group
      auto-offset-reset: earliest
      enable-auto-commit: false           # ⭐️ 关闭自动提交
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      max-poll-records: 100               # 控制单次拉取量
      max-poll-interval-ms: 600000        # 10分钟处理超时
    listener:
      ack-mode: MANUAL_IMMEDIATE          # ⭐️ 手动ACK，处理完立即提交
      concurrency: 3                      # 消费者线程数（≤ Partition数）
```

#### Step 2: 发送消息

```java
@Service
public class OrderEventProducer {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    public void sendOrderPaidEvent(Long orderId, String payload) {
        // send(topic, key, value)
        // topic: 目标 Topic 名称，消息将发送到该 Topic 的某个 Partition
        // key:   消息键，相同 key 的消息会被路由到同一个 Partition（保证局部有序）
        // value: 消息体，实际的业务数据内容
        kafkaTemplate.send("order.events", String.valueOf(orderId), payload)
            // whenComplete 回答的是"消息有没有安全到达 Broker"，不是"消息有没有被成功消费"
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("发送订单支付事件失败: orderId={}", orderId, ex);
                } else {
                    log.info("发送成功: partition={}, offset={}", 
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}
```

#### Step 3: 消费消息（手动 ACK + 幂等）

```java
@Component
public class OrderEventListener {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    @Autowired
    private OrderService orderService;

    @KafkaListener(topics = "order.events", groupId = "order-consumer-group")
    public void onOrderEvent(String message, 
                              @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                              @Header(KafkaHeaders.OFFSET) long offset,
                              Acknowledgment acknowledgment) {
        
        String messageId = partition + "-" + offset;
        
        // ⭐️ 幂等去重：防止重复消费
        Boolean isNew = redisTemplate.opsForValue()
                .setIfAbsent("mq:idem:" + messageId, "1", Duration.ofHours(24));
        if (Boolean.FALSE.equals(isNew)) {
            log.warn("重复消息，跳过: {}", messageId);
            acknowledgment.acknowledge(); // 重复消息也要ACK，否则会反复重试
            return;
        }
        
        try {
            orderService.handleOrderEvent(message);
            
            // ⭐️ 处理成功后手动提交 Offset
            acknowledgment.acknowledge();
            log.info("消费成功: {}", messageId);
            
        } catch (Exception e) {
            log.error("消费失败: {}", messageId, e);
            // 不ACK → Kafka 会重新投递该消息
            throw new RuntimeException("消费失败，等待重试", e);
        }
    }
}
```

#### Kafka 关键配置解读

| 配置 | 值 | 为什么这样设 |
| :--- | :--- | :--- |
| `enable-auto-commit` | false | 避免处理失败但 Offset 已提交导致消息丢失 |
| `ack-mode` | MANUAL_IMMEDIATE | 手动 ACK，处理完立即提交，精确控制提交时机 |
| `max-poll-records` | 100 | 减少单次处理量，避免触发 max.poll.interval.ms |
| `concurrency` | 3 | 消费者线程数，应 ≤ Topic Partition 数 |
| `enable.idempotence` | true | Producer 端幂等，防止重试导致重复发送 |
| `acks` | all | 所有 ISR 确认，保证消息不丢失 |

> 💡 **面试话术**："Spring Boot 集成 Kafka 通过 spring-kafka starter 实现。生产端用 KafkaTemplate 发送，配置 acks=all + enable.idempotence 保证可靠投递。消费端用 @KafkaListener 监听，关闭自动提交，使用 MANUAL_IMMEDIATE 模式在处理成功后手动 ACK。配合 Redis SETNX 做幂等去重，max.poll.records 控制单次拉取量避免 Rebalance。"

### RocketMQ 集成

#### Step 1: 添加依赖与配置

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.3.0</version>
</dependency>
```

```yaml
# application.yml
rocketmq:
  name-server: localhost:9876
  producer:
    group: order-producer-group
    send-message-timeout: 3000        # 发送超时 3s
    retry-times-when-send-failed: 3   # 同步发送失败重试次数
  consumer:
    group: order-consumer-group
```

#### Step 2: 发送消息

```java
@Service
public class OrderEventProducer {

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    /**
     * 发送普通消息
     * convertAndSend(topic:tag, payload) — 自动序列化+同步发送，最常用
     */
    public void sendOrderPaidEvent(String payload) {
        SendResult result = rocketMQTemplate.convertAndSend("ORDER_TOPIC:PAID", payload);
        log.info("发送结果: msgId={}, status={}", result.getMsgId(), result.getSendStatus());
    }

    /**
     * 发送延迟消息（如：订单30分钟未支付自动取消）
     * delayLevel: 1=1s, 2=5s, 3=10s, ..., 16=2h
     */
    public void sendDelayCancelEvent(Long orderId) {
        Message<String> message = MessageBuilder.withPayload(orderId.toString())
                .setHeader(RocketMQHeaders.KEYS, String.valueOf(orderId))
                .build();
        rocketMQTemplate.syncSend("ORDER_TOPIC:CANCEL", message, 3000, 14); // level 14 = 30min
    }

    /**
     * ⭐️ 指定 Queue 发送顺序消息
     * 相同 hashKey 的消息会被路由到同一个 Queue，保证局部有序
     * hashKey 用于选择 Queue：hash(hashKey) % queueCount
     */
    public void sendOrderlyMessage(String destination, String payload, String hashKey) {
        SendResult result = rocketMQTemplate.syncSendOrderly(destination, payload, hashKey, 3000);
        log.info("顺序消息发送结果: msgId={}, queueId={}, status={}", 
                result.getMsgId(), result.getMessageQueue().getQueueId(), result.getSendStatus());
    }
}
```

#### RocketMQTemplate 常用方法速查

| 类别 | 方法 | 说明 |
| :--- | :--- | :--- |
| **同步发送** | `syncSend(dest, payload)` | 阻塞等待 Broker 确认 |
| | `syncSend(dest, msg, timeout)` | 指定超时时间 |
| | `syncSend(dest, msg, timeout, delayLevel)` | 同步发送延迟消息 |
| | `syncSendOrderly(dest, payload, hashKey)` | 顺序消息，相同 hashKey → 同一 Queue |
| **异步发送** | `asyncSend(dest, payload, callback)` | 异步发送，回调获取结果 |
| | `asyncSendOrderly(dest, payload, hashKey, callback)` | 异步顺序消息 |
| **单向发送** | `sendOneWay(dest, payload)` | Fire-and-Forget，不等确认 |
| | `sendOneWayOrderly(dest, payload, hashKey)` | 单向顺序消息 |
| **事务消息** | `sendMessageInTransaction(dest, msg, arg)` | 半消息，配合 `@RocketMQTransactionListener` |
| **便捷方法** | `convertAndSend(dest, payload)` | ⭐️ 自动序列化+同步发送，最常用 |
| | `convertSendAndReceive(dest, payload, type)` | Request-Reply 模式 |

> 💡 **destination 格式**：`"TOPIC_NAME"` 或 `"TOPIC_NAME:TAG"`，后者支持消费端按 Tag 过滤。

| 场景 | 推荐方法 |
| :--- | :--- |
| 普通业务消息 | `convertAndSend` 或 `syncSend` |
| 高吞吐、允许少量丢失 | `asyncSend` |
| 日志/埋点 | `sendOneWay` |
| 顺序消息 | `syncSendOrderly` |
| 分布式事务 | `sendMessageInTransaction` |
| 延迟投递 | `syncSend(dest, msg, timeout, delayLevel)` |

#### Step 3: 消费消息（并发消费 + 幂等）

```java
@Component
@RocketMQMessageListener(
    topic = "ORDER_TOPIC",
    selectorExpression = "PAID || SHIPPED",  // ⭐️ Tag 过滤
    consumerGroup = "order-consumer-group",
    consumeMode = ConsumeMode.CONCURRENTLY,  // 并发消费（默认）
    consumeThreadMax = 20,                   // ⭐️ 最大消费线程数，默认 20
    maxReconsumeTimes = 3                    // 最大重试次数，超过进入死信队列
)
public class OrderEventListener implements RocketMQListener<String> {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    @Autowired
    private OrderService orderService;

    @Override
    public void onMessage(String message) {
        // ⭐️ 幂等去重
        String messageId = message.hashCode() + "-" + System.currentTimeMillis();
        Boolean isNew = redisTemplate.opsForValue()
                .setIfAbsent("mq:idem:" + messageId, "1", Duration.ofHours(24));
        if (Boolean.FALSE.equals(isNew)) {
            log.warn("重复消息，跳过: {}", messageId);
            return; // 返回即视为消费成功，不会重试
        }

        try {
            orderService.handleOrderEvent(message);
            log.info("消费成功: {}", messageId);
            // ✅ 方法正常返回 = 消费成功，自动 ACK
        } catch (Exception e) {
            log.error("消费失败: {}", messageId, e);
            // ❌ 抛出异常 = 消费失败，触发重试（最多 maxReconsumeTimes 次）
            throw new RuntimeException("消费失败，等待重试", e);
        }
    }
}
```

#### 并发消费执行模型（⚠️ 关键认知）

`rocketmq-spring-boot-starter` 虽然封装为逐条回调 `onMessage`，但底层仍是 **Pull 模式 + 线程池并行处理**：

```
Broker 返回一批消息 [msg1, msg2, msg3, ..., msg20]
        │
        ▼
   框架分发到消费线程池（consumeThreadMax = 20）
        │
   ┌────┼────┬────┬────┐
   ▼    ▼    ▼    ▼    ▼
 Thread1 Thread2 Thread3 ... Thread20
 onMessage(msg1) onMessage(msg2) onMessage(msg3) ... onMessage(msg20)
 (独立执行)      (独立执行)      (独立执行)          (独立执行)
```

| 要点 | 说明 |
| :--- | :--- |
| **每个线程独立执行 `onMessage`** | 20 个线程可同时处理 20 条消息，真正并行 |
| **消息少时线程空闲** | 只有 5 条消息就只用 5 个线程，其余等待下一批 |
| **`onMessage` 必须线程安全** | 多线程并发执行同一段代码，共享资源需用连接池或加锁 |
| **Pull 而非 Push** | 底层 Consumer 主动从 Broker 拉取一批消息，框架再逐条分发给线程池 |
| **`consumeThreadMax`** | 控制最大并发消费线程数，默认 20，应根据业务处理能力和下游承载能力调整 |

> ⚠️ **常见误区**：不要以为 `onMessage` 是串行执行的。并发模式下它是多线程并行的，业务代码中的共享变量、非线程安全的工具类都可能引发并发问题。

> 💡 **与 Kafka 的区别**：RocketMQ 的线程池是**框架内部的、与 ACK 绑定的**（onMessage 返回 = ACK）；Kafka 原生 API 中自己开的线程池与 commit 完全解耦，容易提前提交导致消息丢失。如果用 Spring Kafka 的 @KafkaListener + 手动 ACK，效果和 RocketMQ 一样安全。

#### 顺序消费变体

将 `consumeMode` 改为 `ORDERLY` 即可实现顺序消费，监听器接口不变：

```java
@Component
@RocketMQMessageListener(
    topic = "ORDER_TOPIC",
    consumerGroup = "order-orderly-consumer-group",  // ⚠️ 建议用独立消费者组
    consumeMode = ConsumeMode.ORDERLY,                // ⭐️ 关键：改为 ORDERLY
    selectorExpression = "PAID || SHIPPED"
)
public class OrderOrderlyListener implements RocketMQListener<String> {

    @Override
    public void onMessage(String message) {
        // 同一 Queue 内的消息会被串行调用此方法
        // 框架保证：前一条 onMessage 返回后，才会调用下一条
        try {
            process(message);
            // ✅ 正常返回 = 消费成功，自动 ACK，继续处理下一条
        } catch (Exception e) {
            log.error("顺序消费失败，记录并跳过: {}", message, e);
            alertService.sendAlert("顺序消息处理失败: " + message);
            // ⚠️ 不抛异常 → 继续处理下一条，避免阻塞整个 Queue
            // 失败的 msgId 记录到数据库/Redis，后续人工补偿
        }
    }
}
```

| 原生 SDK | @RocketMQMessageListener 等价写法 |
| :--- | :--- |
| `MessageListenerOrderly` | `consumeMode = ConsumeMode.ORDERLY` |
| `ConsumeOrderlyStatus.SUCCESS` | 方法正常返回（void） |
| `ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT` | 抛出异常 |
| `List<MessageExt> msgs` 批量 | 框架拆成单条逐条回调 `onMessage` |

> ⚠️ **顺序消费三个前提**：
> 1. **发送端也要保证顺序**：必须用 `syncSendOrderly(dest, payload, hashKey)` 将相同业务 Key 的消息发到同一 Queue
> 2. **独立消费者组**：顺序消费和并发消费不要共用 Consumer Group
> 3. **异常处理要谨慎**：顺序模式下抛异常会**阻塞整个 Queue**，建议 catch 后记录日志+告警，而非直接抛出

#### Step 4: 事务消息示例

```java
@Service
public class TransactionalOrderProducer {

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    @Autowired
    private InventoryService inventoryService;

    @Autowired
    private TransactionLogService transactionLogService;

    public void createOrderWithTxMessage(Long orderId, String payload) {
        Message<String> message = MessageBuilder.withPayload(payload)
                .setHeader(RocketMQHeaders.KEYS, String.valueOf(orderId))
                .build();
        rocketMQTemplate.sendMessageInTransaction("ORDER_TOPIC:CREATED", message, orderId);
    }

    /**
     * ⭐️ 事务监听器：决定半消息是 Commit 还是 Rollback
     */
    @RocketMQTransactionListener
    public static class OrderTransactionListener implements RocketMQLocalTransactionListener {

        @Autowired
        private InventoryService inventoryService;

        @Autowired
        private TransactionLogService transactionLogService;

        @Override
        public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
            Long orderId = (Long) arg;
            try {
                inventoryService.deduct(orderId);
                transactionLogService.logCommitted(orderId);
                return RocketMQLocalTransactionState.COMMIT;
            } catch (Exception e) {
                transactionLogService.logRollback(orderId);
                return RocketMQLocalTransactionState.ROLLBACK;
            }
        }

        @Override
        public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
            Long orderId = Long.valueOf(msg.getHeaders().get(RocketMQHeaders.KEYS, String.class));
            boolean committed = transactionLogService.isCommitted(orderId);
            return committed ? RocketMQLocalTransactionState.COMMIT 
                             : RocketMQLocalTransactionState.UNKNOWN;
        }
    }
}
```

#### RocketMQ 关键配置与注解解读

| 配置/注解 | 值 | 说明 |
| :--- | :--- | :--- |
| `selectorExpression` | `"PAID \|\| SHIPPED"` | Tag 过滤表达式，只消费匹配的标签 |
| `consumeMode` | `CONCURRENTLY` | 并发消费；顺序消费用 `ORDERLY` |
| `maxReconsumeTimes` | 3 | 最大重试次数，超过后自动进入 `%DLQ%{consumerGroup}` |
| 方法正常返回 | - | 等价于 ACK，消费成功 |
| 方法抛异常 | - | 等价于 NACK，触发重试 |
| `sendMessageInTransaction` | - | 发送事务消息，配合 `@RocketMQTransactionListener` |
| `syncSend` 第4个参数 | delayLevel | 延迟级别，1-18 对应不同延迟时间 |

> 💡 **与 Kafka 集成的核心区别**：
> - RocketMQ 消费成功 = 方法正常返回，失败 = 抛异常，**无需手动 ACK**
> - 原生支持 Tag 过滤、延迟消息、事务消息，Kafka 均不支持
> - 重试由框架自动管理，超过次数自动进入死信队列

> 💡 **面试话术**："Spring Boot 集成 RocketMQ 通过 rocketmq-spring-boot-starter 实现。发送端用 RocketMQTemplate，支持普通消息、延迟消息、事务消息。消费端用 @RocketMQMessageListener 注解，通过 selectorExpression 做 Tag 过滤，方法正常返回即 ACK，抛异常触发重试，超过 maxReconsumeTimes 自动进入死信队列。事务消息通过 @RocketMQTransactionListener 实现半消息+回查机制。"

---

## 五、消息队列常见面试问题解法

### Q1：消费积压了怎么办？

**答**：分三步排查和处理：

**第一步：定位原因**
- 消费者处理太慢？→ 看 RT、GC、下游依赖
- 生产者突增？→ 看监控流量曲线
- 消费者宕机/Rebalance？→ 看消费者日志

**第二步：紧急扩容**
- 增加消费者实例（不超过 Partition/Queue 数量）
- 临时增加 Partition/Queue 数量（Kafka 在线扩分区，RocketMQ 需重启）
- 跳过非关键消息（如日志类），优先处理核心业务

**第三步：长期优化**
- 消费者异步化处理（收到消息立即 ACK，业务逻辑异步执行）
- 批量消费 + 批量写入数据库
- 优化消费者业务逻辑，减少不必要的 RPC/DB 调用
- 设置合理的 `max.poll.records`，避免单次处理过多

> ⚠️ **通用约束**：消费者数量不能超过 Partition/Queue 数量。扩容顺序：**先扩 Partition/Queue，再加消费者实例**。扩容不影响已有消息，在线安全。

### Q2：分区/队列不均衡怎么解决？

**答**：

| 场景 | 原因 | 解决方案 |
| :--- | :--- | :--- |
| **生产端不均** | Key Hash 热点 | 自定义分区器，打散热点 Key |
| **Rebalance 时映射关系大幅调整** | 分配策略不合理 | Kafka 用 StickyAssignor；RocketMQ 用 AllocateMessageQueueAveragelyByCircle |
| **数据倾斜** | 某些 Key 消息量远超其他 | 二级 Hash：先按 Key Hash 到 Partition，再在 Partition 内随机 |

> **StickyAssignor**（Kafka）：Rebalance 时尽量保持上次的分配不变，只调整少量消费者与 Partition 的映射关系，减少消费中断。它不能解决部分 Partition 消息过多、集中在某个消费者的数据倾斜问题。
> **AllocateMessageQueueAveragelyByCircle**（RocketMQ）：环形轮询分配 Queue，使消费者增减时只需调整少量消费者与 Queue 的映射关系（类似 Kafka StickyAssignor 的效果）。设置方式：`allocateStrategy = AllocateStrategy.AVERAGELY_BY_CIRCLE`。注意：它优化的是映射关系调整的稳定性，不能解决 Queue 内消息量不均的数据倾斜问题——数据倾斜需要从发送端打散 Key 来解决。

### Q3：消费者宕机后的 Rebalance 有什么问题？如何优化？

**答**：Rebalance 是 Kafka 最大的痛点之一：

**问题**：
- Rebalance 期间整个 Consumer Group **暂停消费**
- 频繁 Rebalance 导致消费延迟飙升
- 常见触发原因：消费者处理超时、心跳超时、订阅变更

**优化方案**：

| 措施 | 说明 |
| :--- | :--- |
| 增大 `max.poll.interval.ms` | 避免处理慢被误判为宕机 |
| 增大 `session.timeout.ms` | 避免网络抖动触发 Rebalance |
| 使用 StickyAssignor | 增量分配，减少不必要的 Partition 迁移 |
| 消费者预处理异步化 | 缩短单次 poll 的处理时间 |
| 静态成员（Static Membership） | 设置 `group.instance.id`，短暂离线不触发 Rebalance |

> 💡 **RocketMQ 优势**：RocketMQ 的 Rebalance 是**客户端自主协调**，不需要服务端参与，速度更快、影响更小。且 RocketMQ 没有 `max.poll.interval.ms`，处理再慢也不会被踢出。

### Q4：如何保证消息不丢失？

**答**：消息丢失可能发生在三个环节，需逐一保障：

| 环节 | 风险 | 解决方案 |
| :--- | :--- | :--- |
| **Producer → Broker** | 发送失败/未确认 | Kafka: `acks=all` + `retries` + `enable.idempotence=true`；RocketMQ: 同步发送 + 重试 |
| **Broker 存储** | 磁盘故障/未刷盘 | Kafka: `replication.factor≥3` + `min.insync.replicas≥2`；RocketMQ: 同步刷盘 + 主从同步 |
| **Broker → Consumer** | 消费失败/Offset 提前提交 | 关闭自动提交，业务处理成功后手动提交 Offset |

> ⚠️ **核心原则**：**先处理，后提交**。自动提交（`enable.auto.commit=true`）在生产环境中应关闭。

> 💡 **面试话术**："保证消息不丢失需要从三个环节逐一保障。**发送端**：Kafka 配置 acks=all + enable.idempotence=true，确保所有 ISR 副本确认且重试不产生重复；RocketMQ 使用同步发送加重试。**存储端**：Kafka 设置 replication.factor≥3、min.insync.replicas≥2，RocketMQ 开启同步刷盘和主从同步，防止磁盘故障丢数据。**消费端**：关闭自动提交，业务处理成功后再手动提交 Offset，遵循'先处理、后提交'原则，避免处理失败但 Offset 已提交导致消息丢失。很多候选人只提到消费端手动提交，实际上发送端和存储端的保障同样关键。"

### Q5：如何保证消息不被重复消费？

**答**：MQ 天然保证"至少一次"投递，重复消费不可避免，**必须在消费端做幂等**：

| 幂等方案 | 适用场景 | 实现方式 |
| :--- | :--- | :--- |
| **唯一键约束** | 数据库写入 | INSERT 用唯一索引，重复插入自动忽略或更新 |
| **Redis 去重** | 通用场景 | SETNX messageId，已存在则跳过 |
| **状态机校验** | 有状态业务 | 检查当前状态是否允许该操作 |
| **版本号/Token** | 并发更新 | UPDATE WHERE version = oldVersion |

> 💡 **面试话术**："消息队列保证的是'至少一次'投递，重复消费是必然的。我们在消费端通过 Redis SETNX 做消息去重，结合数据库唯一键约束做双重保障。对于有状态的业务，还会校验状态机确保操作的合法性。"

### Q6：如何保证顺序消息？

**答**：

| 层级 | Kafka | RocketMQ |
| :--- | :--- | :--- |
| **全局有序** | 单 Partition（牺牲吞吐） | 单 Queue（牺牲吞吐） |
| **局部有序** | 相同 Key → 同 Partition | 相同 ShardingKey → 同 Queue |
| **消费端有序** | 单线程消费 Partition | 顺序消费模式（ORDERLY） |

> ⚠️ **注意**：顺序消息会显著降低吞吐量，仅在必要时使用（如订单状态流转、Binlog 同步）。大多数业务只需"局部有序"即可。发送端和消费端都要保证顺序，缺一不可。

---

## 小结

本篇覆盖了 Kafka 与 RocketMQ 的完整知识体系：

- **Kafka 原理**：分区并行、顺序写、零拷贝（sendfile）、副本机制、Pull 模式、Rebalance、消费慢最佳实践
- **RocketMQ 原理**：事务消息、死信队列、延迟消息、消息回溯
- **全景对比**：功能差异、消费者管理差异、并行度约束、选型建议
- **Spring Boot 实战**：Kafka + RocketMQ 完整的发送/消费/事务消息代码
- **六大面试问题**：消费积压、分区不均衡、Rebalance 优化、消息不丢失、幂等防重、顺序消息

> 💡 **记忆口诀**：**Kafka 吞吐王，Rocket 业务强；事务延迟死信队，Kafka 没有它来扛；积压扩容查原因，幂等去重不能忘；先处理后提交，顺序消息慎思量**。