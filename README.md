# 古法程序员的必备心法
日常写的一些博客随笔，辅助突击面试的时候，快速掌握相关知识点。
记录一些古法程序员需要掌握的基本技能

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
- [Spring 依赖注入实现原理（三种注入方式、自动装配、循环依赖三级缓存）](web框架/Spring-DI-Notes.md)
- [Spring Bean 生命周期（创建6步、销毁3种、BeanPostProcessor扩展点）](web框架/Spring-Bean-Lifecycle-Notes.md)

## Java基础

- [Java 锁机制：synchronized、ReentrantLock、volatile、CAS](Java基础/Java-Lock-Mechanism-Notes.md)
- [Java 内存模型（JMM）：happens-before、可见性、有序性](Java基础/JMM-Notes.md)
- 常用异步工具：CompletableFuture、ForkJoinPool、虚拟线程
- [线程池：核心参数、拒绝策略、生产配置实践与 Spring @Async](Java基础/ThreadPool-Notes.md)
- AQS 原理：CLH 队列、state、Condition
- ThreadLocal 原理：内存泄漏、InheritableThreadLocal、TTL
- [JVM 内存模型与垃圾回收：GC 算法、收集器（Serial/Parallel/CMS/G1）、调优实战](Java基础/JVM-GC-Notes.md)
- 集合框架：HashMap 底层、ConcurrentHashMap、ArrayList vs LinkedList
- 类加载机制：双亲委派、打破双亲委派、SPI
- 反射与动态代理：JDK Proxy vs CGLIB、Spring AOP 底层

## 数据库

- [MySQL 面试常问知识点总结（B+树、MVCC、日志系统、锁机制、死锁排查、慢SQL优化）](数据库/MySQL-Interview-Notes.md)
- [Redis 面试常问知识点总结（数据结构、持久化、主从复制、哨兵、集群、缓存策略、分布式锁）](数据库/Redis-Interview-Notes.md)
- [Elasticsearch 面试常问知识点总结（倒排索引、集群架构、写入搜索流程、BM25、深度分页、性能优化）](数据库/Elasticsearch-Interview-Notes.md)

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
- [计算机网络常用名词扫盲（OSI七层、网卡/交换机/路由器、MAC/IP/CIDR/NAT、ARP）](计算机网络/Network-Basics-Notes.md)
- TCP 与 UDP：区别、可靠传输、流量控制与拥塞控制
- HTTP/HTTPS：协议演进、TLS 握手、常见状态码与缓存机制
- Netty 核心原理：Reactor 模型、零拷贝、内存池

## 操作系统

> 📖 **推荐阅读顺序**：内存管理 → 进程管理 → 文件管理 → 设备管理。内存管理是基础（虚拟内存、COW），进程管理建立在其上（fork/地址空间），文件管理依赖内存管理（Page Cache/mmap），设备管理是 IO 模型的硬件基础。

- [内存管理：虚拟内存、分页分段、页面置换、COW、mmap](操作系统/Memory-Management-Notes.md)
- [进程管理：PCB与状态机、fork/COW、用户态/内核态、IPC、CFS调度、线程模型](操作系统/Process-Management-Notes.md)
- [文件管理：inode/dentry/VFS、Page Cache、文件描述符、零拷贝](操作系统/File-Management-Notes.md)
- [设备管理：DMA/中断、IO模型（epoll/io_uring）、设备文件、块设备与字符设备](操作系统/Device-Management-Notes.md)


## AI相关

- Transformer 与 LLM 基础：Tokenization、推理优化、Function Calling
- Prompt Engineering：系统提示词设计、Few-shot、CoT 思维链
- RAG 实战：Embedding、向量检索、Chunking 策略、混合搜索
- AI Agent 开发：Spring AI、LangChain/LangGraph、Tool Use
- LLM 应用架构：流式输出、Token 计费、缓存策略、安全护栏
- [AI 编程方法论：如何让 AI 写出高质量代码](AI编程/AI-Programming-Notes.md)
- [Pi Agent 使用指南](AI编程/Pi-guide.md)


# 优秀文章链接

- [面试官最爱问：CPU100%如何处理？](https://juejin.cn/post/7383100103000965130)
- [大厂面试题MySQL解析：MVCC、Redolog、Undolog与Binlog的区别](https://juejin.cn/post/7574725286530089011)
- [消息积压了100万，除了加机器，还能干什么？](https://juejin.cn/post/7546527709537533995)
- [post为什么会发送两次请求？](https://juejin.cn/post/7269952188927017015)
- [如何保证Redis与Mysql双写一致性？连续两个面试问到了！](https://juejin.cn/post/7357976193273151539)
- [网上的 AQS 文章让我很失望](https://juejin.cn/post/7428927105612185627)
- [Maven入门，读完这篇就够了](https://juejin.cn/post/6844903543711907848)
