# blogs
日常写的一些博客随笔，辅助突击面试的时候，快速掌握相关知识点

# 服务端基础

## web框架

- [OpenFeign 技术原理](web框架/OpenFeign-Notes.md)
- [SpringMVC 实现细节](web框架/SpringMVC-Handler-Notes.md)
- [SpringBoot 自动装配](web框架/SpringBoot-AutoConfiguration-Notes.md)
- [Mybatis与MybatisPlus](web框架/MyBatis-Core-Notes.md)
- [Dubbo](web框架/Dubbo-Core-Notes.md)
- [SpringCloud 全家桶](web框架/SpringCloud-Core-Notes.md)
- [Spring Cloud Gateway 核心要点](web框架/SpringCloud-Gateway-Notes.md)
- [服务雪崩防护：限流、熔断与降级](web框架/Service-Protection-Notes.md)
- [事件机制、发布订阅、状态机设计](web框架/Event-PubSub-Notes.md)
- [分布式消息队列：Kafka 与 RocketMQ 核心原理与实战](web框架/MQ-Kafka-RocketMQ-Notes.md)
- Spring 的依赖注入实现原理
- Spring Bean 的生命周期

## Java基础

- Java 锁机制：synchronized、ReentrantLock、volatile、CAS
- Java 内存模型（JMM）：happens-before、可见性、有序性
- 常用异步工具：CompletableFuture、ForkJoinPool、虚拟线程
- 线程池：核心参数、拒绝策略、生产配置实践
- AQS 原理：CLH 队列、state、Condition
- ThreadLocal 原理：内存泄漏、InheritableThreadLocal、TTL
- JVM 内存模型与垃圾回收：GC 算法、收集器、调优实战
- 集合框架：HashMap 底层、ConcurrentHashMap、ArrayList vs LinkedList
- 类加载机制：双亲委派、打破双亲委派、SPI
- 反射与动态代理：JDK Proxy vs CGLIB、Spring AOP 底层

## 数据库

- MySQL 面试常问知识点总结
- Redis 面试常问知识点总结
- elastic-search 面试常问知识点总结

## 运维

- [Linux 基础：研发日常速查手册（常用命令、权限、进程管理、日志排查）](运维/Linux-Basics-Notes.md)
- [Docker 核心原理与常用命令（含 Docker Compose）](运维/Docker-Core-Notes.md)
- [K8s 核心概念：研发视角速查手册（Pod、Deployment、Service、ConfigMap、Ingress、HPA）](运维/K8s-Core-Concepts-Notes.md)
- [阿里云部署架构实战：VPC、ALB、网关、K8s 集群、出入网配置从零搭建](运维/Aliyun-Deployment-Architecture-Notes.md)

## 方案设计

- 分布式理论基础：CAP、BASE、共识算法（Raft/Paxos）
- 分布式 ID 生成：Snowflake、Leaf、数据库号段模式
- 常用设计模式：策略、模板方法、观察者、责任链
- 幂等设计与最终一致性保障
- [接口安全：防重放、签名验证与 HTTPS](方案设计/API-Security-Notes.md)
- [认证授权：RBAC、JWT、SSO 与框架实践](web框架/Auth-RBAC-JWT-SSO-Notes.md)
- [跨域问题分析与解决方案（CORS vs CSRF、Nginx 反向代理）](方案设计/Cross-Origin-Analysis-Solutions.md)
- 分库分表：ShardingSphere、垂直/水平拆分、数据迁移
- 高并发库存扣减：Redis 预扣、数据库乐观锁、Lua 原子操作
- 缓存一致性：Cache Aside、延迟双删、Binlog 订阅
- [微信小程序登录流程与手机号获取（wx.login、code2Session、getPhoneNumber）](方案设计/WeChat-MiniProgram-Login-Notes.md)
- [微信支付 JSAPI 支付流程详解（统一下单、回调通知、V3 签名）](方案设计/WeChat-Pay-JSAPI-Notes.md)


## 计算机网络基础

- TCP 与 UDP：区别、可靠传输、流量控制与拥塞控制
- HTTP/HTTPS：协议演进、TLS 握手、常见状态码与缓存机制
- Netty 核心原理：Reactor 模型、零拷贝、内存池

## 操作系统
- 用户态与内核态、系统调用
- IO 模型：BIO/NIO/AIO 与 IO 多路复用
- 内存管理：虚拟内存、分页分段、页面置换


## AI相关

- Transformer 与 LLM 基础：Tokenization、推理优化、Function Calling
- Prompt Engineering：系统提示词设计、Few-shot、CoT 思维链
- RAG 实战：Embedding、向量检索、Chunking 策略、混合搜索
- AI Agent 开发：Spring AI、LangChain/LangGraph、Tool Use
- LLM 应用架构：流式输出、Token 计费、缓存策略、安全护栏
- [AI 编程方法论：如何让 AI 写出高质量代码](AI编程/AI-Programming-Notes.md)


# 优秀文章链接

- [面试官最爱问：CPU100%如何处理？](https://juejin.cn/post/7383100103000965130)
- [大厂面试题MySQL解析：MVCC、Redolog、Undolog与Binlog的区别](https://juejin.cn/post/7574725286530089011)
- [消息积压了100万，除了加机器，还能干什么？](https://juejin.cn/post/7546527709537533995)
- [post为什么会发送两次请求？](https://juejin.cn/post/7269952188927017015)
- [如何保证Redis与Mysql双写一致性？连续两个面试问到了！](https://juejin.cn/post/7357976193273151539)
- [网上的 AQS 文章让我很失望](https://juejin.cn/post/7428927105612185627)
- [Maven入门，读完这篇就够了](https://juejin.cn/post/6844903543711907848)