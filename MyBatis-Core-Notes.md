# MyBatis 核心原理与实战要点

## 从一个"SQL 拼接噩梦"说起

在日常开发中，你是否经历过这些场景：

1.  **条件查询的 if-else 地狱**：一个列表查询接口，支持按用户名、手机号、状态、时间范围等任意组合筛选。用原生 JDBC 写出来就是一堆 `if (name != null) sql += " AND name = ?"` 的字符串拼接，既容易出错又难以维护。
2.  **SQL 注入的隐患**：为了支持模糊搜索或动态表名，你不得不把用户输入直接拼进 SQL 字符串里，结果被安全团队扫出了 SQL 注入漏洞。
3.  **分页逻辑重复造轮子**：每个列表接口都要手写 `LIMIT offset, size`，还要单独查一次 `COUNT(*)` 获取总数，代码冗余且容易漏掉边界条件。
4.  **多租户/读写分离的困惑**：业务扩张后需要接入多个数据源，但每次切换都要手动管理 Connection，事务边界也变得模糊不清。

这些问题的共同根源是：**SQL 与业务代码高度耦合，缺乏声明式的抽象层**。MyBatis 正是为解决这些问题而生——它不是全自动 ORM（如 Hibernate），而是一个**半自动化的 SQL 映射框架**，让你既能享受动态 SQL 的便利，又能保留对 SQL 的完全掌控权。

---

## 一、动态 SQL：告别字符串拼接

### 核心标签速查

| 标签 | 作用 | 典型场景 |
| :--- | :--- | :--- |
| `<if test="">` | 条件判断 | 可选筛选条件 |
| `<where>` | 智能处理 WHERE 关键字及多余 AND/OR | 多条件组合查询 |
| `<choose>/<when>/<otherwise>` | 类似 switch-case | 互斥条件分支 |
| `<foreach>` | 集合遍历 | IN 查询、批量插入 |
| `<set>` | 智能处理 SET 关键字及多余逗号 | 动态更新 |
| `<trim>` | 自定义前后缀及覆盖规则 | 高级定制 |
| `<bind>` | 创建变量绑定到上下文 | OGNL 表达式复用 |

### 实战示例：多条件用户查询

```xml
<select id="selectUsers" resultType="User">
    SELECT * FROM t_user
    <where>
        <if test="username != null and username != ''">
            AND username LIKE CONCAT('%', #{username}, '%')
        </if>
        <if test="phone != null and phone != ''">
            AND phone = #{phone}
        </if>
        <if test="status != null">
            AND status = #{status}
        </if>
        <choose>
            <when test="orderBy == 'createTime'">
                ORDER BY create_time DESC
            </when>
            <when test="orderBy == 'updateTime'">
                ORDER BY update_time DESC
            </when>
            <otherwise>
                ORDER BY id DESC
            </otherwise>
        </choose>
    </where>
</select>
```

**关键点**：
- `<where>` 会自动去除开头多余的 `AND/OR`，并在有内容时添加 `WHERE` 关键字；
- 所有参数使用 `#{}` 占位符，天然防注入；
- `<choose>` 实现排序字段的白名单校验，避免 ORDER BY 注入风险。

### OGNL 表达式基础

MyBatis 的动态 SQL 标签（`<if test="">`、`<when test="">`、`<bind>` 等）中的 `test` 属性值并非 Java 代码，而是 **OGNL（Object-Graph Navigation Language）** 表达式。它是一种强大的表达式语言，用于在对象图中导航和求值。

**核心能力**：
-   **属性访问**：`user.name`、`order.items[0].price`
-   **方法调用**：`name.length()`、`list.size()`、`str.startsWith('prefix')`
-   **逻辑运算**：`&&`、`||`、`!`、`and`、`or`、`not`
-   **比较运算**：`==`、`!=`、`>`、`<`、`>=`、`<=`、`eq`、`neq`、`gt`、`lt`、`gte`、`lte`
-   **空值判断**：`== null`、`!= null`
-   **集合操作**：`in`、`not in`、投影 `{#this.name}`、选择 `?{#this.age > 18}`
-   **静态成员访问**：`@com.example.Constants@MAX_SIZE`、`@java.lang.Math@PI`

> 💡 **关键点**：OGNL 是 MyBatis 动态 SQL 的"条件引擎"，理解它的语法和陷阱是写好动态 SQL 的前提。它比 Java 表达式更灵活，但也更容易踩坑。

### ⚠️ 避坑指南（带代码示例）

#### 坑1：字符串比较用 `==` 失效

**问题**：OGNL 中单引号 `'A'` 表示字符（char），双引号 `"A"` 表示字符串（String）。`char == String` 永远为 false。

```xml
<!-- ❌ 错误：'Y' 是 char，status 是 String，比较结果恒为 false -->
<if test="status == 'Y'">
    AND is_active = 1
</if>

<!-- ✅ 正确写法1：双引号表示字符串 -->
<if test='status == "Y"'>
    AND is_active = 1
</if>

<!-- ✅ 正确写法2：调用 toString() 转换 -->
<if test="status.toString() == 'Y'">
    AND is_active = 1
</if>

<!-- ✅ 正确写法3：使用 equals 方法（最安全） -->
<if test="'Y'.equals(status)">
    AND is_active = 1
</if>
```

> 🎯 **最佳实践**：字符串比较一律使用 `'literal'.equals(variable)` 形式，既避免类型陷阱，又防止变量为 null 时的 NPE。

#### 坑2：数字 0 与空字符串的隐式转换

**问题**：OGNL 会将数字 `0` 视为"假值"，空字符串 `""` 在某些上下文中也会被当作 false。这导致判空逻辑不符合预期。

```java
// Mapper 接口
List<User> selectUsers(@Param("status") Integer status, @Param("keyword") String keyword);
```

```xml
<!-- ❌ 危险：当 status=0 时，此条件为 false，0 被当作"无值"跳过 -->
<if test="status != null and status != ''">
    AND status = #{status}
</if>

<!-- ✅ 正确：Integer 类型只判 null，不要加 != '' -->
<if test="status != null">
    AND status = #{status}
</if>

<!-- ❌ 危险：keyword="" 时，OGNL 可能将其视为 false -->
<if test="keyword">
    AND name LIKE CONCAT('%', #{keyword}, '%')
</if>

<!-- ✅ 正确：显式判空 + 判非空串 -->
<if test="keyword != null and keyword != ''">
    AND name LIKE CONCAT('%', #{keyword}, '%')
</if>
```

> 🎯 **规则**：**数值类型只判 `!= null`，字符串类型判 `!= null and != ''`**。永远不要对数值类型加 `!= ''` 检查。

#### 坑3：单参数 vs 多参数的属性访问差异

**问题**：Mapper 方法只有一个参数且未加 `@Param` 时，OGNL 直接以该参数为根对象；多个参数或加了 `@Param` 时，根对象是一个 Map。

```java
// 单参数无 @Param → 根对象就是 User 本身
List<User> findByUsername(String username);

// 多参数 → 根对象是 Map{param1=username, param2=status, ...}
List<User> search(String username, Integer status);

// 多参数带 @Param → 根对象是 Map{name=username, status=status}
List<User> search(@Param("name") String username, @Param("status") Integer status);
```

```xml
<!-- 单参数场景：直接用属性名 -->
<if test="username != null">  <!-- ✅ 正确，username 就是根对象的属性 -->
    AND username = #{username}
</if>

<!-- 多参数无 @Param：必须用 param1/param2 -->
<if test="param1 != null">   <!-- ✅ 正确 -->
    AND username = #{param1}
</if>
<if test="username != null"> <!-- ❌ 错误！Map 中没有 username 这个 key -->
    AND username = #{username}
</if>

<!-- 多参数带 @Param：用 @Param 指定的名称 -->
<if test="name != null">     <!-- ✅ 正确 -->
    AND username = #{name}
</if>
```

> 🎯 **最佳实践**：**所有 Mapper 方法参数都加 `@Param` 注解**，消除歧义，提高可读性。

#### 坑4：`<foreach>` 的 collection 取值规则

**问题**：`collection` 属性的值取决于参数传递方式，写错会导致 `BindingException`。

```java
// List 参数无 @Param
List<User> findByIds(List<Long> ids);

// List 参数带 @Param
List<User> findByIds(@Param("idList") List<Long> ids);

// 数组参数
List<User> findByIds(Long[] ids);

// Map 参数
List<User> findByIds(Map<String, Object> params); // params.put("ids", list)
```

```xml
<!-- List 无 @Param → collection="list" -->
<foreach collection="list" item="id" open="(" separator="," close=")">
    #{id}
</foreach>

<!-- List 带 @Param → collection="@Param值" -->
<foreach collection="idList" item="id" open="(" separator="," close=")">
    #{id}
</foreach>

<!-- 数组 → collection="array" -->
<foreach collection="array" item="id" open="(" separator="," close=")">
    #{id}
</foreach>

<!-- Map → collection="map中的key" -->
<foreach collection="ids" item="id" open="(" separator="," close=")">
    #{id}
</foreach>
```

> 🎯 **记忆口诀**：**List→list，Array→array，Map→key名，@Param→指定名**。不确定时加 `@Param` 是最稳妥的做法。

#### 坑5：OGNL 中调用方法的风险

**问题**：OGNL 允许在 test 表达式中调用任意方法，但某些方法有副作用或性能问题。

```xml
<!-- ❌ 危险：每次条件判断都调用数据库/远程服务 -->
<if test="userService.exists(username)">
    AND username = #{username}
</if>

<!-- ❌ 性能差：大集合的 size() 可能触发懒加载 -->
<if test="items.size() > 0">
    AND item_count > 0
</if>

<!-- ✅ 安全：只调用纯计算方法 -->
<if test="name != null and name.trim().length() > 0">
    AND name = #{name}
</if>

<!-- ✅ 推荐：复杂判断提前在 Java 层计算好，传入布尔值 -->
<if test="hasValidName">
    AND name = #{name}
</if>
```

> 🎯 **原则**：test 表达式应保持**纯计算、无副作用、高性能**。复杂逻辑应在 Service 层预处理，而非在 XML 中编写"迷你程序"。

---

## 二、`#{}` vs `${}`：预编译与字符串替换的本质区别

这是 MyBatis 面试**必考题**，必须从底层原理讲清楚。

### 核心对比

| 维度 | `#{}` | `${}` |
| :--- | :--- | :--- |
| 处理方式 | 预编译参数占位符（PreparedStatement 的 `?`） | 纯字符串替换（直接拼接到 SQL 中） |
| SQL 注入风险 | ✅ 安全，参数作为绑定变量传递 | ❌ 危险，用户输入直接参与 SQL 构建 |
| 性能 | 可复用执行计划，数据库缓存友好 | 每次生成不同 SQL，无法复用执行计划 |
| 适用场景 | 所有用户输入的值参数 | 仅用于元数据（表名、列名、ORDER BY 字段等） |
| 类型处理 | 自动类型转换（JDBC TypeHandler） | 无类型处理，原样拼接 |

### 底层原理

```java
// #{} 生成的 PreparedStatement
String sql = "SELECT * FROM t_user WHERE username = ?";
PreparedStatement ps = connection.prepareStatement(sql);
ps.setString(1, userInput); // 参数独立于 SQL 文本传递

// ${} 生成的 Statement
String sql = "SELECT * FROM t_user WHERE username = '" + userInput + "'";
Statement stmt = connection.createStatement();
stmt.execute(sql); // 用户输入已成为 SQL 的一部分
```

### 何时不得不用 `${}`？

仅在以下**元数据动态化**场景中，且**必须做白名单校验**：

```xml
<!-- ✅ 安全用法：白名单校验后的排序字段 -->
<if test="orderByColumn in {'create_time', 'update_time', 'id'}">
    ORDER BY ${orderByColumn} DESC
</if>

<!-- ❌ 危险用法：直接使用用户输入 -->
ORDER BY ${userProvidedColumn} <!-- SQL 注入风险！ -->
```

> 🎯 **面试话术**：`#{}` 是预编译参数绑定，本质是 PreparedStatement 的 `?` 占位符，参数与 SQL 分离传输，天然防注入且可复用执行计划；`${}` 是纯字符串替换，发生在 SQL 解析之前，仅适用于表名、列名等无法预编译的元数据场景，使用时必须做白名单校验。**默认永远优先使用 `#{}`**。

---

## 三、分页实现：从手写 LIMIT 到插件自动化

### 方案对比

| 方案 | 优点 | 缺点 | 适用场景 |
| :--- | :--- | :--- | :--- |
| 手写 LIMIT/OFFSET | 简单直观 | 代码冗余、深分页性能差 | 小项目、简单查询 |
| RowBounds（MyBatis 内置） | 无需改 SQL | 内存分页（查出全部再截取）、性能极差 | **生产环境禁用** |
| PageHelper 插件 | 零侵入、自动 COUNT | 依赖 ThreadLocal、复杂 SQL 可能误判 | 中小项目快速开发 |
| 自定义 Interceptor | 完全可控、可优化深分页 | 开发成本高 | 大型项目、特殊分页需求 |

### PageHelper 工作原理（面试重点）

PageHelper 的核心是一个 MyBatis **Interceptor**，执行流程如下：

1.  调用 `PageHelper.startPage(pageNum, pageSize)` → 将分页参数存入 **ThreadLocal**；
2.  MyBatis 执行查询前触发拦截器 → 检测到 ThreadLocal 中有分页参数；
3.  **改写原始 SQL**：
    -   生成 COUNT 语句（通过 JSqlParser 解析 AST 并移除 ORDER BY 等无关子句）；
    -   在原始 SQL 末尾追加 `LIMIT ?, ?`（MySQL）或对应方言的分页语法；
4.  先执行 COUNT 获取总数，再执行改写后的分页查询；
5.  封装结果为 `PageInfo` 对象返回，并**清除 ThreadLocal**（防止内存泄漏）。

> ⚠️ **PageHelper 踩坑点**：
> -   `startPage()` 只对**紧接着的第一条查询**生效，中间插入了其他查询会导致分页失效或错乱；
> -   多线程环境下 ThreadLocal 不会自动清理，务必在 finally 中手动 `PageHelper.clearPage()`；
> -   复杂嵌套查询、UNION、存储过程等场景下 COUNT 改写可能出错，需手动指定 countSql。

### 深分页优化思路

当 `OFFSET` 很大时（如第 100 万页），数据库仍需扫描前 999999 行再丢弃，性能急剧下降。常见优化方案：

-   **延迟关联**：先通过覆盖索引查出目标 ID 列表，再 JOIN 回主表获取完整数据；
-   **游标分页**：记录上次查询的最大 ID，下次用 `WHERE id > lastMaxId LIMIT size` 替代 OFFSET；
-   **搜索引擎兜底**：超大数据量的复杂检索交给 ES，MySQL 仅做精确点查。

---

## 四、拦截器（Interceptor）：MyBatis 的 AOP 机制

### 四大拦截点

MyBatis 允许拦截以下四个核心对象的方法：

| 拦截目标 | 可拦截方法 | 典型用途 |
| :--- | :--- | :--- |
| `Executor` | update, query, flushStatements, commit, rollback | 全局日志、性能监控、缓存增强 |
| `StatementHandler` | prepare, parameterize, batch | SQL 改写、参数加密、慢 SQL 告警 |
| `ParameterHandler` | setParameters | 参数脱敏、类型转换增强 |
| `ResultSetHandler` | handleResultSets, handleOutputParameters | 结果集转换、敏感字段解密 |

### 自定义拦截器模板

```java
@Intercepts({
    @Signature(type = Executor.class, method = "query", 
               args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})
})
public class SlowSqlInterceptor implements Interceptor {

    private long slowThresholdMs = 1000;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return invocation.proceed(); // 执行原方法
        } finally {
            long cost = System.currentTimeMillis() - start;
            if (cost > slowThresholdMs) {
                MappedStatement ms = (MappedStatement) invocation.getArgs()[0];
                log.warn("Slow SQL detected: {} cost {}ms", ms.getId(), cost);
            }
        }
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this); // JDK 动态代理包装
    }

    @Override
    public void setProperties(Properties properties) {
        String threshold = properties.getProperty("slowThresholdMs");
        if (threshold != null) {
            this.slowThresholdMs = Long.parseLong(threshold);
        }
    }
}
```

### `@Intercepts` + `@Signature` 注解详解

这个注解组合是 MyBatis 拦截器的**核心声明机制**，它告诉框架"我要拦截哪个对象的哪个方法"：

```java
@Intercepts({                          // 外层：可声明多个拦截点
    @Signature(                        // 内层：一个具体的拦截签名
        type = Executor.class,         // 1. 要拦截的目标接口/类
        method = "query",              // 2. 要拦截的方法名
        args = {                       // 3. 方法的参数类型列表（必须精确匹配）
            MappedStatement.class, 
            Object.class, 
            RowBounds.class, 
            ResultHandler.class
        }
    )
})
```

| 属性 | 作用 | 为什么需要 |
| :--- | :--- | :--- |
| **`type`** | 指定要拦截的 MyBatis 核心组件接口 | MyBatis 有四大可拦截对象，必须明确告诉框架你要代理哪一个 |
| **`method`** | 指定该接口中要拦截的方法名 | 同一个接口有多个方法（如 Executor 有 query、update、commit 等），需精确定位 |
| **`args`** | 指定该方法的**完整参数类型列表** | Java 支持方法重载，仅靠方法名无法唯一确定目标方法，必须通过参数签名区分 |

#### 为什么 `args` 必须精确匹配？

以 `Executor` 为例，它有**两个** `query` 方法：

```java
// 重载1：4个参数（最常用）
List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler);

// 重载2：6个参数（含 CacheKey 和 BoundSql）
List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql);
```

如果你只写 `method = "query"` 而不指定 `args`，MyBatis 无法知道你要拦截哪一个。**`args` 就是用来消除重载歧义的**。示例中的 4 参数签名对应第一个重载，这也是最常用的拦截点。

#### 底层原理

MyBatis 在初始化时扫描所有 `Interceptor` 实现类上的 `@Intercepts` 注解，解析出 `(type, method, args)` 三元组，注册到全局 Map 中。当创建目标对象（如 Executor）时，检查是否有匹配的拦截签名，如果有就用 **JDK 动态代理** 包装该对象，调用时先走拦截器的 `intercept()` 方法，再决定是否执行原方法（`invocation.proceed()`）。

> 🎯 **面试话术**：`@Intercepts` + `@Signature` 是 MyBatis 拦截器的声明式配置，`type` 指定拦截目标组件，`method` 指定方法名，`args` 指定参数类型列表以消除重载歧义。MyBatis 据此为目标对象创建 JDK 动态代理，实现非侵入式的 AOP 增强。

### 注册方式

```yaml
# application.yml (SpringBoot)
mybatis:
  configuration:
    interceptors:
      - com.example.interceptor.SlowSqlInterceptor
```

或在 XML 中：
```xml
<plugins>
    <plugin interceptor="com.example.interceptor.SlowSqlInterceptor">
        <property name="slowThresholdMs" value="2000"/>
    </plugin>
</plugins>
```

> 💡 **设计原理**：MyBatis 拦截器基于 **JDK 动态代理 + 责任链模式**。`Plugin.wrap()` 为目标对象创建代理，多个拦截器按注册顺序形成链式调用。注意拦截器顺序很重要——越先注册的越外层，越晚注册的越靠近真实执行。

---

## 五、多数据源配置：从单库到分库分表的过渡

### 方案选型

| 方案 | 复杂度 | 适用场景 | 代表工具 |
| :--- | :--- | :--- | :--- |
| Spring AbstractRoutingDataSource | 低 | 读写分离、简单多库切换 | 手写 / dynamic-datasource-spring-boot-starter |
| ShardingSphere-JDBC | 高 | 分库分表、分布式事务 | Apache ShardingSphere |
| MyBatis-Plus Dynamic Datasource | 中 | 注解式切换、多租户 | baomidou/dynamic-datasource |

### AbstractRoutingDataSource 核心原理（面试常考）

这是 Spring 提供的抽象类，也是大多数多数据源方案的基石：

```java
public abstract class AbstractRoutingDataSource extends AbstractDataSource {
    
    // ⭐️ 子类实现：根据上下文决定使用哪个数据源 key
    protected abstract Object determineCurrentLookupKey();
    
    // 内部维护 Map<Object, DataSource> resolvedDataSources
    // getConnection() 时调用 determineCurrentLookupKey() 获取 key
    // 再从 map 中取出对应的 DataSource 返回连接
}
```

**典型实现步骤**：

1.  定义 `DynamicDataSource extends AbstractRoutingDataSource`，重写 `determineCurrentLookupKey()` 从 ThreadLocal 读取当前数据源标识；
2.  定义 `DataSourceContextHolder`，提供 `set()/get()/clear()` 静态方法操作 ThreadLocal；
3.  配置多个 `DataSource` Bean，组装到 `DynamicDataSource` 的 targetDataSources Map 中；
4.  通过 AOP 切面或注解（如 `@DS("slave")`）在方法执行前设置 ThreadLocal，执行后清除。

> ⚠️ **关键注意事项**：
> -   **事务内不能切换数据源**：Spring 事务管理器在开启事务时就已获取 Connection，后续切换无效。如需跨库事务，应使用 JTA/XA 或 Seata；
> -   **ThreadLocal 必须及时清理**：线程池复用场景下不清理会导致数据源错乱；
> -   **连接池隔离**：每个数据源应有独立的连接池实例，避免相互影响。

### dynamic-datasource-spring-boot-starter 推荐

对于大多数业务场景，推荐使用 [baomidou/dynamic-datasource](https://github.com/baomidou/dynamic-datasource)，它封装了上述所有细节，并提供：

-   `@DS("datasourceName")` 注解声明式切换
-   支持嵌套调用、SpEL 表达式动态数据源
-   内置读写分离、负载均衡策略
-   与 MyBatis-Plus、Seata 无缝集成

---

## 六、补充：MyBatis 执行流程全景图

理解上述知识点的基础是对 MyBatis 整体执行链路有清晰认知：

```
Mapper 接口调用
    ↓
SqlSession.selectList / insert / update
    ↓
Executor.query / update （一级缓存检查）
    ↓
StatementHandler.prepare （创建 PreparedStatement）
    ↓
ParameterHandler.setParameters （参数绑定）
    ↓
Statement.execute （发送 SQL 到数据库）
    ↓
ResultSetHandler.handleResultSets （结果映射）
    ↓
返回结果
```

**关键组件职责**：
-   **Configuration**：全局配置中心，持有所有 Mapper、TypeHandler、Interceptor 等元数据；
-   **MappedStatement**：一条 SQL 的完整描述（SQL 文本、参数类型、结果映射、缓存策略等）；
-   **BoundSql**：经过动态 SQL 解析后的最终 SQL 及参数映射关系；
-   **TypeHandler**：Java 类型 ↔ JDBC 类型的双向转换器；
-   **ObjectWrapperFactory**：结果集到 Java 对象的反射赋值引擎。

---

## 七、MyBatis-Plus：从手写 SQL 到 CRUD 自动化

日常开发中，我们几乎不再直接使用原生 MyBatis，而是使用 **MyBatis-Plus（MP）**。它不是对 MyBatis 的替代，而是在其基础上做的**增强工具包**——只做增强不做改变，所有原生 MyBatis 的功能依然可用。

### 核心价值：消除 80% 的重复 CRUD

| 原生 MyBatis 痛点 | MP 解决方案 |
| :--- | :--- |
| 每个表都要写 insert/update/delete/select XML | `BaseMapper<T>` 内置 17+ 通用方法，零 SQL |
| 条件查询拼接繁琐 | `LambdaQueryWrapper` / `QueryWrapper` 链式构建 |
| 分页需手写 COUNT + LIMIT | 内置 `PaginationInnerInterceptor`，自动改写 |
| 字段填充（createTime/updateBy）需拦截器 | `MetaObjectHandler` 声明式自动填充 |
| 逻辑删除需手动加 WHERE deleted=0 | `@TableLogic` 注解自动处理 |
| 乐观锁版本号需手动 update set version=version+1 | `@Version` 注解自动递增 |

### BaseMapper 与 IService 分层

```java
// Mapper 层：继承 BaseMapper，获得单表 CRUD 能力
public interface UserMapper extends BaseMapper<User> {
    // 复杂查询仍可自定义 XML/注解 SQL
}

// Service 层：继承 IService，获得批量操作 + 事务封装
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements IUserService {
    // saveBatch(), listByIds(), page(), lambdaQuery() 等开箱即用
}
```

> 💡 **设计哲学**：`BaseMapper` 解决**单表原子操作**，`IService` 解决**业务编排与批量操作**。简单查询用 Wrapper，复杂关联查询仍回归 XML，两者互补而非互斥。

### LambdaQueryWrapper vs QueryWrapper

```java
// ❌ QueryWrapper：字符串列名，重构易出错
QueryWrapper<User> wrapper = new QueryWrapper<>();
wrapper.eq("user_name", "张三")
       .ge("age", 18)
       .orderByDesc("create_time");

// ✅ LambdaQueryWrapper：方法引用，编译期校验，IDE 友好
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(User::getUserName, "张三")
       .ge(User::getAge, 18)
       .orderByDesc(User::getCreateTime);
```

**强烈推荐始终使用 Lambda 版本**，避免硬编码字符串带来的维护风险。

### 自动填充与逻辑删除

```java
// 自动填充处理器
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {
    @Override
    public void insertFill(MetaObject metaObject) {
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "deleted", Integer.class, 0);
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }
}

// 实体类标注
@TableField(fill = FieldFill.INSERT)
private LocalDateTime createTime;

@TableField(fill = FieldFill.INSERT_UPDATE)
private LocalDateTime updateTime;

@TableLogic  // 逻辑删除：查询自动加 WHERE deleted=0，删除变为 UPDATE SET deleted=1
private Integer deleted;
```

### MP 分页插件配置

```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // ⭐️ 分页插件必须添加，否则 page() 方法不会自动改写 SQL
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```

```java
// 使用方式
Page<User> page = userService.page(new Page<>(1, 10), 
    new LambdaQueryWrapper<User>().eq(User::getStatus, 1));
// page.getRecords() → 当前页数据
// page.getTotal()   → 总记录数（自动 COUNT）
```

> ⚠️ **常见踩坑**：忘记注册 `PaginationInnerInterceptor` 导致 `page()` 返回全部数据而非分页结果；多数据源场景下需为每个数据源单独配置拦截器。

---

## 面试高频问题速查

### Q1: `#{}` 和 `${}` 的区别是什么？什么时候用哪个？
**答**：`#{}` 是预编译参数占位符，生成 PreparedStatement 的 `?`，参数与 SQL 分离传输，防注入且可复用执行计划；`${}` 是纯字符串替换，直接拼接到 SQL 中，有注入风险且无法复用执行计划。**默认永远用 `#{}`**，仅在动态表名、列名、ORDER BY 字段等无法预编译的元数据场景下使用 `${}`，且必须做白名单校验。

### Q2: MyBatis 分页有哪些方式？PageHelper 的原理是什么？
**答**：四种方式：手写 LIMIT（简单但冗余）、RowBounds（内存分页，生产禁用）、PageHelper 插件（零侵入，基于 Interceptor 改写 SQL）、自定义 Interceptor（完全可控）。PageHelper 原理：通过 ThreadLocal 传递分页参数 → 拦截 Executor.query → 用 JSqlParser 改写 SQL 追加 LIMIT 并生成 COUNT 语句 → 先查总数再查分页数据 → 封装 PageInfo 并清除 ThreadLocal。注意 startPage 只对紧邻的下一次查询生效，且需在 finally 中清理防止内存泄漏。

### Q3: MyBatis 拦截器能拦截哪些对象？执行顺序如何确定？
**答**：可拦截 Executor、StatementHandler、ParameterHandler、ResultSetHandler 四个核心接口的方法。执行顺序由注册顺序决定：先注册的拦截器在外层（先进后出），后注册的在内层更靠近真实执行。底层基于 JDK 动态代理 + 责任链模式，`Plugin.wrap()` 为目标对象创建代理链。

### Q4: 多数据源切换的原理是什么？有什么限制？
**答**：核心是 Spring 的 `AbstractRoutingDataSource`，子类重写 `determineCurrentLookupKey()` 从 ThreadLocal 读取当前数据源标识，`getConnection()` 时据此从内部 Map 中选择对应 DataSource。主要限制：① 事务内不能切换（Connection 在事务开启时已绑定）；② ThreadLocal 必须及时清理（线程池复用场景）；③ 每个数据源需独立连接池。跨库事务需用 JTA/Seata。

### Q5: MyBatis 的一级缓存和二级缓存有什么区别？为什么二级缓存不推荐在生产使用？
**答**：一级缓存是 SqlSession 级别的本地缓存，默认开启，同一 SqlSession 内相同查询直接返回缓存结果，SqlSession 关闭/提交/更新后失效；二级缓存是 Mapper Namespace 级别的跨 Session 缓存，需显式开启，序列化存储，多个 SqlSession 共享。**二级缓存不推荐的原因**：① 多表关联查询时缓存粒度粗，易产生脏读；② 分布式环境下各节点缓存不一致；③ 更新操作只清空本 Namespace 缓存，无法感知其他 Namespace 的关联变更；④ 序列化开销大。生产环境建议用 Redis 等外部缓存替代。

### Q6: MyBatis 是如何将结果集映射到 Java 对象的？
**答**：核心流程：ResultSetHandler 遍历 ResultSet → 根据 ResultMap/ResultType 获取列名与属性的映射关系 → 通过 TypeHandler 将 JDBC 类型转换为 Java 类型 → 使用 ObjectWrapper（反射/MetaClass）将值赋给目标对象属性。支持自动映射（列名=属性名）、手动映射（resultMap）、嵌套结果/嵌套查询关联映射、集合映射等复杂场景。

### Q7: MyBatis-Plus 和 MyBatis 是什么关系？MP 做了哪些增强？
**答**：MP 是 MyBatis 的**增强工具包**，只做增强不做改变，所有原生功能完全兼容。核心增强包括：① `BaseMapper` 内置 17+ 通用 CRUD 方法，零 SQL 完成单表操作；② `LambdaQueryWrapper` 链式构建条件，编译期校验列名；③ 内置分页插件自动改写 SQL 并生成 COUNT；④ `MetaObjectHandler` 声明式自动填充 createTime/updateTime 等字段；⑤ `@TableLogic` 逻辑删除、`@Version` 乐观锁注解化处理。简单查询用 Wrapper，复杂关联查询仍回归 XML，两者互补。

### Q8: LambdaQueryWrapper 和 QueryWrapper 有什么区别？为什么推荐 Lambda 版本？
**答**：`QueryWrapper` 使用字符串指定列名（如 `"user_name"`），重构时容易遗漏导致运行时错误；`LambdaQueryWrapper` 使用方法引用（如 `User::getUserName`），编译期即可校验字段是否存在，IDE 支持自动补全和重命名联动。**推荐始终使用 Lambda 版本**，除非需要动态拼接列名（此时可用 `QueryWrapper` + 白名单校验）。

### Q9: MyBatis-Plus 的逻辑删除是怎么实现的？有什么注意事项？
**答**：通过 `@TableLogic` 注解标记逻辑删除字段，MP 在底层自动改写 SQL：查询追加 `WHERE deleted=0`，删除变为 `UPDATE SET deleted=1`，物理 DELETE 语句不会执行。注意事项：① 逻辑删除字段需在数据库中设置默认值（通常为 0）；② 唯一索引需包含逻辑删除字段，否则"删除后重新创建同名记录"会冲突；③ 关联查询中子表也需配置逻辑删除，否则 JOIN 时会查出已删除数据；④ 如需真正物理删除，需用 `removeById` 配合自定义 SQL 或临时关闭逻辑删除。

### Q10: MyBatis-Plus 分页插件忘记注册会怎样？如何正确配置？
**答**：忘记注册 `PaginationInnerInterceptor` 会导致 `page()` 方法**不自动改写 SQL**，返回全部数据而非分页结果，且 `getTotal()` 为 0。正确配置：在 `@Configuration` 类中创建 `MybatisPlusInterceptor` Bean，添加 `PaginationInnerInterceptor(DbType.MYSQL)`。多数据源场景下需为每个数据源的 SqlSessionFactory 单独配置拦截器。注意 MP 3.4+ 版本使用 `MybatisPlusInterceptor` + `InnerInterceptor` 新架构，旧版 `PaginationInterceptor` 已废弃。

---

## 小结

MyBatis 生态的核心价值在于 **"SQL 与代码解耦 + 动态能力 + 可扩展性 + CRUD 自动化"**：
-   **动态 SQL** 解决了条件拼接的可维护性问题；
-   **`#{}` 预编译** 解决了安全性与性能的平衡问题；
-   **Interceptor 机制** 提供了非侵入式的横切关注点扩展能力；
-   **AbstractRoutingDataSource** 为多数据源提供了标准化的切换抽象；
-   **MyBatis-Plus** 在日常开发中消除了 80% 的重复 CRUD 工作，让开发者聚焦业务逻辑。

掌握这五个支柱，无论是日常开发还是架构设计，都能做到游刃有余。

> 📌 **记忆口诀**：**动态 SQL 靠标签、占位符选井号、分页用插件、拦截四件套、多源 Routing、缓存慎用二级、CRUD 交给 Plus**。
