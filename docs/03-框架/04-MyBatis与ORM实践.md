# MyBatis 与 ORM 实践

> **定位**：讲清 MyBatis 从 SqlSessionFactory 到结果映射的执行链路、插件与缓存机制、以及批量、多数据源、MyBatis-Plus、SQL 治理这些工程落地要点。
> **重要度**：★★★★★（国内后端岗位使用率最高的 ORM，缓存、插件、N+1、批量是高频考点）
> **建议复习时长**：4~6 小时（建议手写一个 MyBatis 插件并抓一次分页 SQL）
> **关联文档**：[Spring 核心 IoC 与 AOP](./01-Spring核心IoC与AOP.md)、[Spring Boot 与自动装配](./02-SpringBoot与自动装配.md)、[MySQL 体系结构与索引](../02-数据库/01-MySQL体系结构与索引.md)、[MySQL 调优主从与分库分表](../02-数据库/03-MySQL调优主从与分库分表.md)、[Redis 高可用与缓存治理](../02-数据库/05-Redis高可用与缓存治理.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| MyBatis 架构与执行流程 | ★★★★★ | 源码追问 |
| 四大对象与插件机制 | ★★★★★ | 源码追问 / 手写代码 |
| 动态 SQL 标签 | ★★★★☆ | 配置实践 / 手写代码 |
| #{} 与 ${} 与 SQL 注入 | ★★★★★ | 八股原理 / 故障排查 |
| 一级缓存与二级缓存 | ★★★★★ | 八股原理 / 场景题 |
| 延迟加载与 N+1 问题 | ★★★★★ | 场景题 / 故障排查 |
| resultMap、TypeHandler 与主键回填 | ★★★★☆ | 配置实践 |
| 批量操作与性能 | ★★★★☆ | 配置实践 / 场景题 |
| Spring Boot 整合与多数据源 | ★★★★☆ | 配置实践 |
| MyBatis-Plus 常用能力 | ★★★★☆ | 配置实践 / 场景题 |
| ORM 选型、慢 SQL 与常见坑 | ★★★★☆ | 故障排查 / 场景题 |

## 1. MyBatis 架构与执行流程
### 1.1 核心原理
**构建链路**：`SqlSessionFactoryBuilder` 读配置（mybatis-config.xml 或 Boot 的 Configuration 对象）→ `XMLConfigBuilder`/`XMLMapperBuilder` 解析出 `Configuration`（含 Environment、TypeHandlerRegistry、MapperRegistry、MappedStatement 注册表）→ `DefaultSqlSessionFactory`。

**执行链路**：`SqlSessionFactory#openSession` → `SqlSession`（默认 `DefaultSqlSession`）→ `Executor`（默认 SimpleExecutor，二级缓存开启时被 CachingExecutor 装饰）→ `MappedStatement`（一条 SQL 的全部元信息）→ `StatementHandler`（负责创建 Statement、设置参数占位）→ `ParameterHandler`（把 Java 参数写进 PreparedStatement）→ JDBC 执行 → `ResultSetHandler`（把 ResultSet 映射成对象，内部用 `TypeHandler`）→ 返回结果。

| 组件 | 职责 | 线程安全 |
| --- | --- | --- |
| SqlSessionFactory | 全局唯一，生产 SqlSession | 安全（单例） |
| SqlSession | 一次会话/一次请求的操作入口 | **不安全**，用完必须 close |
| Executor | 执行 SQL、管理一二级缓存、延迟加载 | 与 SqlSession 绑定 |
| StatementHandler | 创建 Statement、参数化、执行 | 每次操作新建 |
| ParameterHandler | 设置参数 | 每次操作新建 |
| ResultSetHandler | 结果集映射、嵌套映射、延迟加载装配 | 每次操作新建 |

**Executor 三种类型**：`SimpleExecutor`（默认，每次新建 Statement）、`ReuseExecutor`（复用同 SQL 的 Statement）、`BatchExecutor`（累积批量，需 `flushStatements` 触发）。**Spring 集成下**：`SqlSessionTemplate` 保证线程安全（内部代理 + 事务同步，把 SqlSession 绑定到 Spring 事务），所以业务代码不要自己 `openSession`，否则会脱离 Spring 事务（表现为"事务回滚了但数据还在"）。
### 1.2 面试问答
**Q1：MyBatis 的执行流程？** A：SqlSessionFactory 构建 → openSession 得到 SqlSession → 通过 Executor 执行 MappedStatement → StatementHandler 创建并参数化 Statement（ParameterHandler）→ JDBC 执行 → ResultSetHandler + TypeHandler 映射结果 → 返回；过程中穿插一二级缓存查询与延迟加载处理。
**Q2：SqlSession 是线程安全的吗？Spring 里怎么解决？** A：MyBatis 原生 SqlSession **不是线程安全的**（内部持连接与缓存，且 DefaultSqlSession 有非线程安全字段）。Spring 通过 `SqlSessionTemplate`（线程安全的代理）+ `SqlSessionInterceptor` 实现：每个操作从 `SqlSessionHolder` 获取或创建 SqlSession，并绑定到 Spring 事务，事务提交/回滚时统一关闭。手动 openSession 会脱离这套机制。
**Q3：MappedStatement 是什么？** A：它是一条 SQL 的完整元信息：id（namespace + statementId）、SQL 语句、参数类型、结果映射、缓存配置、超时、fetchSize、statementType 等。`SqlSession#selectOne("com.x.UserMapper.selectById", id)` 就是按 id 到 Configuration 的 mappedStatements 里取 MappedStatement。
**Q4：MyBatis 的四大对象是什么？为什么叫"四大"？** A：Executor、StatementHandler、ParameterHandler、ResultSetHandler。它们是执行链路上可被插件拦截的四个组件（Plugin 用 JDK 动态代理包装它们），所以叫四大对象。
### 1.3 高频追问与陷阱
- 追问：`Configuration` 里还有 TypeHandlerRegistry、TypeAliasRegistry、LanguageDriver（动态 SQL 用 XMLLanguageDriver）。
- 陷阱：手工 `sqlSession.commit()` 与 Spring 事务混用会导致提交时机错乱，Spring 环境下不要手动 commit。
### 1.4 代码/配置示例
```java
try (SqlSession session = sqlSessionFactory.openSession()) {   // 原生用法，仅用于理解
    UserMapper mapper = session.getMapper(UserMapper.class);
    User u = mapper.selectById(1L);      // 内部走 Executor → StatementHandler → ResultSetHandler
    session.commit();
}
```
### 1.5 一句话速记
Factory → SqlSession（不安全）→ Executor → Statement/Parameter/ResultSetHandler；Spring 用 SqlSessionTemplate 保证安全并绑定事务。

## 2. 四大对象与插件机制
### 2.1 核心原理
**可拦截的方法**（由 `@Intercepts` + `@Signature(type, method, args)` 精确指定）：

| 被拦截对象 | 常用可拦截方法 | 典型插件 |
| --- | --- | --- |
| Executor | update、query、flushStatements、commit、rollback | 分页、SQL 改写、多租户 |
| StatementHandler | prepare、parameterize、batch、update、query | 分页（改写 SQL）、监控耗时 |
| ParameterHandler | getParameterObject、setParameters | 参数加解密、脱敏 |
| ResultSetHandler | handleResultSets、handleOutputParameters | 结果解密、字段脱敏 |

**实现方式**：插件实现 `Interceptor`，在 `intercept(Invocation)` 里通过 `invocation.proceed()` 决定是否放行，`plugin(Object target)` 用 `Plugin.wrap` 生成 JDK 动态代理；多个插件按注册顺序层层包装（**后注册的在外层，先执行**）。**PageHelper 原理**：拦截 `Executor#query`，从 ThreadLocal 取出分页参数 → 先执行 count 查询（可用 `countSql` 优化或关闭）→ 改写 SQL 追加 `limit ?,?`（方言 Dialect 负责）→ 执行 → 清理 ThreadLocal。
### 2.2 面试问答
**Q1：MyBatis 插件原理？** A：责任链 + JDK 动态代理。配置的插件在四大对象创建时被 `Plugin.wrap` 逐层包装，调用时先进入最外层插件的 `intercept`，通过 `Invocation#proceed()` 向下传递，从而在不改 MyBatis 源码的前提下增强能力。
**Q2：PageHelper 是怎么做到分页的？** A：拦截 `Executor#query`，用 ThreadLocal 接收 `PageHelper.startPage(...)` 的参数；因为 `Page` 继承 `ArrayList`，分页信息通过返回值传递（`PageInfo` 再二次封装）；SQL 层面通过 Dialect 改写成 `limit` 并另外发一条 count 查询。**所以 PageHelper 的 count 语句性能与正确性完全取决于自动 count 改写质量**——复杂 SQL（多表 union/group by）建议手写 count。
**Q3：插件为什么会失效？** A：① `@Signature` 的类型/方法/参数列表与真实调用不匹配（如拦截 `Executor#query` 但是走了 `CachingExecutor` 的装饰方法）；② 插件没有被注册（Boot 中要 `@Bean Interceptor` 或 `mybatis.configuration` 之外正确注册）；③ 拦截的方法没被调用（如批量走 `flushStatements` 而插件只拦 `update`）；④ 与其他插件顺序冲突（外层插件提前返回）。排查：打断点/加日志看 `intercept` 是否进入，并用 `--debug` 打印 SQL。
**Q4：写一个 SQL 耗时监控插件要注意什么？** A：拦截 `Executor#query/update`（注意区分 `query` 的 List/Map 重载），在 try-finally 里统计耗时与 SQL id；参数打印要脱敏；**避免在插件里做重活（如每次取堆栈、同步写日志）**；慢 SQL 阈值化并打点上报，别把所有 SQL 都打 INFO。
### 2.3 高频追问与陷阱
- 陷阱：多个分页/改写插件顺序不当会彼此覆盖 SQL，导致分页失效或语法错误。
- 追问：插件里 `invocation.getTarget()` 拿到的是被包装对象，取真实对象要用 `MetaObject`/`SystemMetaObject` 或 `Plugin` 的解包逻辑。
### 2.4 代码/配置示例
```java
@Intercepts(@Signature(type = Executor.class, method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}))
public class CostPlugin implements Interceptor {
    @Override public Object intercept(Invocation invocation) throws Throwable {
        MappedStatement ms = (MappedStatement) invocation.getArgs()[0];
        long s = System.nanoTime();
        try { return invocation.proceed(); }
        finally { long ms2 = (System.nanoTime() - s) / 1_000_000;
                  if (ms2 > 200) log.warn("slow sql {} cost={}ms", ms.getId(), ms2); }
    }
    @Override public Object plugin(Object target) { return Plugin.wrap(target, this); }
}
@Bean public CostPlugin costPlugin() { return new CostPlugin(); }
```
### 2.5 一句话速记
四大对象（Executor/StatementHandler/ParameterHandler/ResultSetHandler）可被 @Intercepts 动态代理拦截；PageHelper 就是拦 Executor#query 改写 limit。

## 3. 动态 SQL 标签
### 3.1 核心原理

| 标签 | 作用 | 注意 |
| --- | --- | --- |
| `<if test="...">` | 条件拼接 | test 是 OGNL，字符串比较写 `'Y'`；判空用 `!= null and != ''` |
| `<choose>/<when>/<otherwise>` | 多分支（switch） | 只走第一个匹配分支 |
| `<where>` | 自动加 WHERE 并去掉开头多余 AND/OR | 比手写 `WHERE 1=1` 更清晰 |
| `<set>` | 自动加 SET 并去掉末尾多余逗号 | 全字段为 null 时会生成非法 SQL，需 `<if>` 兜底 |
| `<trim prefix/suffix prefixOverrides/suffixOverrides>` | 通用裁剪 | where/set 的本质实现 |
| `<foreach>` | 遍历集合 | collection/item/index/open/close/separator；批量插入核心 |
| `<bind>` | 定义变量（如模糊查询拼 %） | 可复用表达式 |
| `<sql>` + `<include>` | SQL 片段复用 | 可用 `<property>` 传参 |
| `<selectKey>` | 主键回填（非自增/序列） | order=BEFORE/AFTER |
### 3.2 面试问答
**Q1：`<where>` 和 `WHERE 1=1` 哪个好？** A：`<where>` 好：自动处理前缀 AND/OR 且无额外恒真条件，语义清晰、可读性高；`1=1` 依赖手动控制 AND，容易漏写导致语法错误，也不利于 SQL 审计。可读性与可维护性都倾向 `<where>`。
**Q2：`<foreach>` 批量插入怎么防 SQL 过长？** A：把大列表按批切分（如每 500~1000 条一批）提交，避免超过 `max_allowed_packet` 与 SQL 解析开销；同时考虑用 `ExecutorType.BATCH`（见第 8 节）。**永远不要在 SQL 里拼接成千上万个占位符**。
**Q3：`<if test="type != null and type != ''">` 对 Integer 为 0 会怎样？** A：OGNL 中 `0 != ''` 在某些版本下会被判为相等（数字与字符串比较的坑），导致条件为 0 时不拼接。稳妥写法是只判 `!= null`（字符串再单独判空），或用 `<choose>` 明确分支。
**Q4：动态 SQL 最危险的错误是什么？** A：**漏写 where 条件的全表更新/删除**。例如 `<update>` 中所有 `<if>` 都为假，最终 SQL 变成 `update t set status=?`（无 where）。防御：① 关键方法改用固定 SQL；② 用 MyBatis-Plus 的 `BlockAttackInnerInterceptor` 拦截全表更新删除；③ 数据库账号权限限制；④ 代码评审 + SQL 审计。
### 3.3 高频追问与陷阱
- 陷阱：`<set>` 更新时若所有字段为 null 会生成 `update t set where id=?`，语法错误。
- 追问：`<sql>` 片段建议按"列清单/公共条件"拆分，避免过度抽象导致 SQL 难读。
### 3.4 代码/配置示例
```xml
<select id="search" resultType="UserVO">
  select id, name, status from t_user
  <where>
    <if test="name != null and name != ''">and name like concat('%', #{name}, '%')</if>
    <if test="status != null">and status = #{status}</if>
    <if test="ids != null and ids.size() > 0">
      and id in <foreach collection="ids" item="id" open="(" close=")" separator=",">#{id}</foreach>
    </if>
  </where>
  order by id desc
</select>
<update id="updateSelective">
  update t_user
  <set>
    <if test="name != null">name = #{name},</if>
    <if test="status != null">status = #{status},</if>
  </set>
  where id = #{id}
</update>
```
### 3.5 一句话速记
`<where>`/`<set>` 自动裁剪、`<foreach>` 批量、`<choose>` 分支、`<sql>`+`<include>` 复用；最怕漏 where 的全表更新。

## 4. #{} 与 ${} 与 SQL 注入
### 4.1 核心原理

| 维度 | `#{}` | `${}` |
| --- | --- | --- |
| 实现 | 预编译占位符 `?` + PreparedStatement.setXxx | 字符串直接拼接进 SQL |
| SQL 注入 | 安全（参数不参与语法解析） | **不安全** |
| 类型处理 | 走 TypeHandler 自动转换 | 需自己保证类型正确 |
| 使用场景 | 绝大多数参数值 | 动态表名/列名/order by 等**结构部分** |
| 预编译缓存 | 同一 SQL 模板可复用执行计划 | 每次拼接都是新 SQL（执行计划无法复用） |

**order by 排序参数的正确处理**：排序字段与方向属于 SQL 结构，无法用 `#{}`（会变成 `order by 'name'` 即按常量排序，无效果）。正确做法是**白名单映射**：把前端字段名映射到固定列名/枚举，方向只允许 ASC/DESC。

```java
private static final Map<String, String> SORT_COLUMNS =
        Map.of("createTime", "create_time", "amount", "amount", "id", "id");
String column = SORT_COLUMNS.get(dto.getSortField());
if (column == null) throw new BizException("非法排序字段");
String direction = "desc".equalsIgnoreCase(dto.getSortOrder()) ? "DESC" : "ASC";
// 传入 XML：ORDER BY ${column} ${direction}，两个值都来自白名单/枚举，而非用户原文
```
### 4.2 面试问答
**Q1：`#{}` 为什么能防注入？** A：因为它被编译成 `?` 占位符，参数值由 JDBC 在**语法解析之后**绑定，值永远只是数据、不会改变 SQL 结构。而 `${}` 是文本替换，参数会被当作 SQL 语法的一部分解析，因此 `' or '1'='1`、`; drop table` 之类能生效。
**Q2：like 查询怎么写？** A：`like concat('%', #{name}, '%')`（推荐，值走占位符）或 `<bind name="pattern" value="'%' + name + '%'"/>` 后用 `#{pattern}`；**不要**写 `'%${name}%'`。注意前置 `%` 会导致索引失效，可结合全文索引/ES 处理。
**Q3：`${}` 完全不能用吗？** A：不是，但必须做到"值来自服务端可控的白名单/枚举"。合法用途：动态表名/列名（分表场景）、order by 字段与方向、`<sql>` 片段名。非法用途：任何用户可控的内容（搜索词、id、路径参数）。分表场景的表名要加后缀白名单校验（如 `t_order_` + 0~127）。
**Q4：还有哪些注入途径容易被忽略？** A：① XML 里用 `${}` 拼 like/order by；② 注解 `@Select("... ${...}")`；③ 动态拼接的 `in` 条件（应用 `<foreach>` 而非 replace）；④ MyBatis-Plus 的 `last()`/`apply()` 方法（本质拼 SQL，必须白名单）；⑤ 存储过程/拼接的存储过程名。**结论：凡是"结构"要看源头是否可控，凡是"值"一律用 `#{}`。**
### 4.3 高频追问与陷阱
- 陷阱：以为用了 MyBatis 就自动免疫注入——`${}`、`last()`、`apply()`、字符串拼接 XML 都会引入注入。
- 追问：`#{}` 与 ${} 混用时，预编译模板会因表名变化而整体失效（每组表名都是新 SQL），这也是分表场景需要注意的执行计划缓存问题。
### 4.4 代码/配置示例
```xml
<!-- 安全：值用 #{}，结构用白名单后传入 -->
<select id="page" resultType="OrderVO">
  select id, amount, create_time from t_order
  where status = #{status}
  order by ${sortColumn} ${sortDirection}
  limit #{offset}, #{size}
</select>
```
### 4.5 一句话速记
值用 `#{}`（预编译防注入），结构用 `${}`（必须白名单）；order by 与分表表名是最常见的注入入口。

## 5. 一级缓存与二级缓存
### 5.1 核心原理

| 维度 | 一级缓存 | 二级缓存 |
| --- | --- | --- |
| 作用域 | SqlSession 级（Spring 下绑定一个事务） | namespace（Mapper）级，跨 SqlSession |
| 开启方式 | 默认开启，`localCacheScope=STATEMENT` 可关闭 | 需 `<cache/>` 或 `@CacheNamespace`，`cache-enabled` 总开关 |
| 存储 | PerpetualCache（HashMap） | PerpetualCache，可接 Ehcache/Redis（需实现 Cache 接口） |
| 失效条件 | 同一 SqlSession 内执行 update/insert/delete（clearLocalCache）、commit/rollback、手动 `flushCache=true`、不同 SqlSession | 该 namespace 的任意写操作清空自己（默认不感知其他 namespace 的写）、`flushCache` 指定、Session 关闭时写入 |
| 脏数据风险 | 中（同事务内其他途径改了库） | **高**（跨 namespace、多表联查、多实例） |

**为什么不建议用二级缓存**：① 默认的 Cache 是**进程内**的，多实例部署下各实例缓存不一致；② 写操作只清理自己 namespace，联表查询或跨 Mapper 更新会产生脏数据；③ 与分布式缓存（Redis）职责重叠且更难治理；④ 一旦线上出现"数据不一致"，排查成本极高。**主流做法：缓存交给 Redis，在 Service 层显式管理 key 与失效，MyBatis 二级缓存关闭**（`mybatis.configuration.cache-enabled=false`）。
### 5.2 面试问答
**Q1：一级缓存什么时候失效？** A：① 同一个 SqlSession 内发生任何 update/insert/delete（会清空整个一级缓存，不只是相关表）；② 事务 commit/rollback；③ `flushCache=true` 的查询；④ SqlSession 关闭；⑤ `localCacheScope=STATEMENT` 时每条语句后清空。实践中最常见的是"换了 SqlSession/出了事务就没了"。
**Q2：为什么 Spring 环境下两个 Service 方法查同一数据没有走缓存？** A：因为 Spring 把 SqlSession 绑定在**事务**上：有 `@Transactional` 时同一事务共享 SqlSession（一级缓存有效），没有事务时每次操作都新建 SqlSession，缓存自然不命中。这是"一级缓存好像没用"的标准答案。
**Q3：二级缓存有什么典型脏数据场景？** A：① 订单 Mapper 与用户 Mapper 联表查询，用户表更新只清用户 namespace，订单侧缓存仍旧；② 多实例部署，A 实例更新后 B 实例缓存未失效；③ 直接通过其他通道（定时任务、其他服务、DB 直连）改数据。所以二级缓存只适合"单实例 + 单表 + 几乎不改"的字典类数据。
**Q4：想用缓存该怎么设计？** A：把缓存放到业务层：Cache-Aside（读时未命中回源并回填、写时删除 key）+ 过期时间兜底 + 空值缓存防穿透 + 逻辑过期或互斥锁防击穿（详见[Redis 高可用与缓存治理](../02-数据库/05-Redis高可用与缓存治理.md)）。MyBatis 层只保留查询能力。
### 5.3 高频追问与陷阱
- 陷阱：开启二级缓存后，实体需要可序列化（默认 SerializedCache），否则报 NotSerializableException。
- 陷阱：`<cache/>` 加在 Mapper 上但跨 Mapper 的写操作不会触发它的失效，这是最隐蔽的脏数据来源。
### 5.4 代码/配置示例
```yaml
mybatis:
  configuration:
    cache-enabled: false            # 关闭二级缓存（推荐）
    local-cache-scope: session      # 一级缓存范围；STATEMENT 表示关闭
```
```xml
<!-- 若确需二级缓存：只用于单表字典 -->
<cache eviction="LRU" size="512" readOnly="false"/>
```
### 5.5 一句话速记
一级缓存在 SqlSession/事务内、写操作即清空；二级缓存跨 namespace，多实例+联表必然脏数据，生产建议关闭交给 Redis。

## 6. 延迟加载与 N+1 问题
### 6.1 核心原理
**延迟加载**：`association`/`collection` 用嵌套 select（`select="..."`）时才生效，并且需要 `lazyLoadingEnabled=true`（MyBatis 默认 false）；`fetchType="lazy|eager"` 可单条覆盖全局配置；`aggressiveLazyLoading`（3.4.1+ 默认 false，已废弃的语义）控制是否一触发就加载全部懒属性。**延迟加载依赖 SqlSession 未关闭**，所以在 Spring 中只有事务内有效——事务外访问懒属性会抛 `LazyInitializationException` 类问题或拿到 null。

**N+1 问题**：查 N 条主记录后，为了组装关联对象又发了 N 条（或 1 条 join 重复行）查询，总计 N+1 次数据库交互。典型来源：循环里 `mapper.selectById`、`association select` 在无延迟加载时逐条触发、ORM 的懒加载在序列化时被触发。

| 方案 | 做法 | 适用 |
| --- | --- | --- |
| JOIN + 嵌套 resultMap | 一次查询返回，用 `<id>` + `<collection>` 合并 | 关联数据量可控（注意行膨胀） |
| 批量 in 查询 | 先查主体，再用 `foreach` 的 in 一次查关联并内存分组 | **最常用**，避免笛卡尔积 |
| 分页后批量装配 | 分页只查主表，再按当页 id 批量查关联 | 列表页标准做法 |
| 延迟加载 | 开启 lazyLoadingEnabled，只在用到的字段上触发 | 关联数据用得少时，但要控制在事务内 |

### 6.2 面试问答
**Q1：什么是 N+1 问题？怎么发现？** A：一次列表查询 + N 次明细查询 = N+1 次交互，数据库往返次数与数据量线性增长。发现方式：打开 SQL 日志看同一模板的重复执行次数、APM（SkyWalking）里的"同一 SQL 高频调用"、`/actuator/metrics` 中连接池与 DB 指标异常。
**Q2：JOIN 一定比多次查询好吗？** A：不一定。JOIN 在**一对多**时会放大行数（1 主 × M 明细 → 结果集 1×M 行），传输与内存都膨胀，而且分页会因行膨胀不准（需要对主表分页或 `distinct` 处理）。所以一对多推荐"主表分页 + 关联批量 in 查询"，一对一/多对一可用 JOIN。
**Q3：延迟加载在 Spring 里不生效/报错的原因？** A：① 没开 `lazyLoadingEnabled`；② 用 join 映射而非嵌套 select（JOIN 不支持延迟）；③ 事务外访问懒属性（SqlSession 已关）；④ 被序列化（Jackson 序列化会触发所有 getter，导致要么全部加载要么报错）——所以接口返回要用 VO/DTO 显式组装，不要把实体直接序列化出去。
**Q4：怎么彻底避免 N+1？** A：① 代码规范上禁止在循环里查库；② 用批量查询替代；③ 用 JOIN 或视图；④ 在代码评审与静态检查（如 SonarQube 自定义规则）里拦截循环内 DAO 调用；⑤ 用 APM 观察 SQL 调用次数。
### 6.3 高频追问与陷阱
- 陷阱：Jackson 序列化实体时触发全部懒加载，把"看起来只有一条 SQL"变成 N 条——用 DTO 隔离是根本解法。
- 追问：`collection` 的 `select` 嵌套查询 + 延迟加载在列表页最危险（每行触发一次），必须用批量装配替代。
### 6.4 代码/配置示例
```yaml
mybatis:
  configuration:
    lazy-loading-enabled: true
    aggressive-lazy-loading: false
```
```java
// 反例：循环内查库 → N+1
for (Order o : orders) { o.setItems(itemMapper.selectByOrderId(o.getId())); }
// 正例：批量 in 一次查询后内存分组
List<Long> ids = orders.stream().map(Order::getId).toList();
Map<Long, List<Item>> grouped = itemMapper.selectByOrderIds(ids).stream()
        .collect(Collectors.groupingBy(Item::getOrderId));
orders.forEach(o -> o.setItems(grouped.getOrDefault(o.getId(), List.of())));
```
### 6.5 一句话速记
延迟加载需 lazyLoadingEnabled 且必须在事务内；列表页用"主表分页 + 批量 in 装配"消灭 N+1。

## 7. resultMap、TypeHandler 与主键回填
### 7.1 核心原理
**自动映射规则**：`autoMappingBehavior` 取 NONE/PARTIAL（默认）/FULL；PARTIAL 只自动映射没有嵌套 resultMap 的字段；`mapUnderscoreToCamelCase` 控制 `create_time` → `createTime`（MyBatis 原生默认 false，**MyBatis-Plus 默认开启**，Boot 项目通常在 `mybatis.configuration` 显式配置）。字段名与属性名不一致时，用 `<resultMap>` 的 `<result column property>` 或 `@Results/@Result` 显式指定。

**自定义 TypeHandler**：实现 `TypeHandler<T>`（或继承 `BaseTypeHandler<T>`）并通过 `@MappedTypes`/`@MappedJdbcTypes` 声明，或在 XML 的 `typeHandler` 属性上指定、或全局注册 `type-handlers-package`。典型用途：枚举 ↔ DB 值转换、JSON 字段 ↔ 对象、加密字段自动加解密。

**主键回填**：自增主键用 `useGeneratedKeys="true" keyProperty="id"`；非自增（序列/UUID）用 `<selectKey order="BEFORE" resultType="long">select seq.nextval from dual</selectKey>`；批量插入时 `keyProperty` 指向集合内对象的 id。
### 7.2 面试问答
**Q1：结果集映射不上/字段为 null 怎么排查？** A：① 检查列名与属性名（下划线转驼峰是否开启）；② 是否用了 resultMap 且漏配字段；③ 列别名与属性不匹配（`select a.name as name`）；④ 用了 `PARTIAL` 自动映射但字段在嵌套 resultMap 里（不会自动映射）；⑤ 类型不匹配（枚举/日期）。最稳的做法是显式 resultMap，或用 `resultType` + 别名。
**Q2：枚举怎么优雅映射？** A：默认 `EnumTypeHandler` 存 name 字符串，`EnumOrdinalTypeHandler` 存序数（**不要用序数，顺序一变数据就错**）。推荐自定义 TypeHandler 按业务 code 存取（如 `StatusEnum.of(code)`），DB 存 int/varchar 都一致；MyBatis-Plus 还提供 `IEnum` 接口简化这一过程。
**Q3：主键回填失败有哪些原因？** A：① 表主键非自增却用了 useGeneratedKeys；② `keyProperty` 与实体属性名不一致；③ 批量插入时 keyProperty 写成了单值；④ 用了 `INSERT ... SELECT` 或某些分库中间件不支持回填；⑤ 事务未提交时读到了 id 但后续回滚。
**Q4：JSON 字段怎么做映射？** A：自定义 TypeHandler 实现 `setNonNullParameter`（对象→JSON 字符串）与 `getNullableResult`（JSON→对象），在字段或 XML 里指定；也可用 MyBatis-Plus 的 `JacksonTypeHandler`（需 `@TableName(autoResultMap = true)` + `@TableField(typeHandler = JacksonTypeHandler.class)`）。注意**查询时 resultMap 没写 typeHandler 会失效**。
### 7.3 高频追问与陷阱
- 陷阱：`mapUnderscoreToCamelCase` 在 Boot 配置里写在 `mybatis.configuration` 下，写到 `mybatis-plus.configuration` 或顶层都不会生效（这是"下划线转驼峰失效"的常见原因）。
- 追问：自定义 TypeHandler 全局注册后，某些字段被显式 resultMap 覆盖时不会走全局 TypeHandler，要单独指定。
### 7.4 代码/配置示例
```java
@MappedTypes(StatusEnum.class)
public class StatusEnumTypeHandler extends BaseTypeHandler<StatusEnum> {
    @Override public void setNonNullParameter(PreparedStatement ps, int i, StatusEnum p, JdbcType t) throws SQLException {
        ps.setInt(i, p.getCode());
    }
    @Override public StatusEnum getNullableResult(ResultSet rs, String column) throws SQLException {
        return StatusEnum.of(rs.getInt(column));
    }
}
```
```xml
<insert id="insert" useGeneratedKeys="true" keyProperty="id">
  insert into t_user(name, status) values(#{name}, #{status})
</insert>
```
### 7.5 一句话速记
自动映射靠驼峰开关 + PARTIAL；枚举/JSON 用自定义 TypeHandler；自增主键用 useGeneratedKeys 回填，非自增用 selectKey。

## 8. 批量操作与性能
### 8.1 核心原理

| 方案 | 写法 | 特点 |
| --- | --- | --- |
| foreach 批量 INSERT | `insert into t(...) values (...),(...),(...)` | 一次网络往返，SQL 长度受 `max_allowed_packet` 限制，需分批 |
| ExecutorType.BATCH | `sqlSessionFactory.openSession(ExecutorType.BATCH)` 或 `SqlSessionTemplate` 批处理 | JDBC 层批量提交，减少往返；**不返回影响行数与自增 id（除个别驱动）** |
| rewriteBatchedStatements=true | MySQL 连接参数 | 让驱动把多条同构 SQL 合并成一条多值 INSERT，效果显著 |

**实践要点**：① 大批量插入按 500~1000 条切批，批与批之间可提交（注意事务粒度）；② 批量插入时**不要**在 SQL 里做 `ON DUPLICATE KEY UPDATE` 之外的重逻辑；③ MySQL 的 `max_allowed_packet`、`innodb_buffer_pool_size`、`log_bin` 都会影响批量写入速度；④ 导入类场景考虑 `LOAD DATA INFILE`；⑤ 批量更新比批量插入更容易踩坑（无 where 的 update 会放大锁范围），优先按主键分批更新。
**性能对比结论（经验值，需实测）**：单条循环插入最慢；foreach 多值插入与 JDBC batch 相近，前者受包大小限制、后者受出参/回滚粒度影响；开启 `rewriteBatchedStatements` 后 batch 提升通常最明显。
### 8.2 面试问答
**Q1：foreach 批量插入和 ExecutorType.BATCH 有什么区别？** A：foreach 在 SQL 层合并成一条多值 INSERT（网络往返一次，但 SQL 巨大且受包限制）；BATCH 在 JDBC 层累积多条语句后一次性发送（不拼大 SQL，但驱动需支持批量重写，且影响行数/自增主键回填受限）。**结论：常规业务用 foreach 分批（简单直观）；超大批量导入用 BATCH + rewriteBatchedStatements，并用压测确认。**
**Q2：批量插入为什么有时很慢？** A：① 每条都自动提交（未包事务）→ 每条一次 fsync；② 未开启 `rewriteBatchedStatements`；③ 唯一索引/主键冲突检查开销；④ binlog 同步（`sync_binlog=1`）；⑤ 分批过大导致大事务（undo 膨胀、锁等待）。按这个顺序排查。
**Q3：批量插入如何拿回自增 id？** A：foreach 插入在 MySQL 下可回填，但**只保证返回第一条的 id**（这是 JDBC `getGeneratedKeys` 的限制），需要用 `LAST_INSERT_ID()` 手工推算或插入后按业务唯一键回查；因此需要 id 的场景建议逐条插入或先申请分布式 ID。这是常见坑，面试常问。
**Q4：批量更新怎么做更稳？** A：按主键分批 `update ... where id = ?`（或 `update t set col = case when id=? then ? end where id in (...)`），每批 500~1000 条并单独提交；避免 `update ... where 非索引列 in (大列表)` 造成锁范围过大与主从延迟。
### 8.3 高频追问与陷阱
- 陷阱：`ExecutorType.BATCH` 下不 flush 就不执行，忘记 `flushStatements` 会"数据没写进去"。
- 陷阱：大事务批量插入导致主从延迟与锁等待，需要分批提交并评估半成功的一致性问题。
### 8.4 代码/配置示例
```xml
<insert id="batchInsert">
  insert into t_order(user_id, amount, status) values
  <foreach collection="list" item="o" separator=",">
    (#{o.userId}, #{o.amount}, #{o.status})
  </foreach>
</insert>
```
```
# JDBC URL：开启批量重写，显著提升 BATCH 插入性能
jdbc:mysql://127.0.0.1:3306/db?rewriteBatchedStatements=true&useServerPrepStmts=false
```
### 8.5 一句话速记
批量插入按 500~1000 条分批；BATCH + rewriteBatchedStatements 最快；注意包大小、事务粒度与自增 id 回填限制。

## 9. Spring Boot 整合与多数据源
### 9.1 核心原理
**整合方式**：`mybatis-spring-boot-starter`（或 MyBatis-Plus 的 starter）自动配置 `SqlSessionFactory`、`SqlSessionTemplate`、`MapperScannerConfigurer`；常用配置项：`mybatis.mapper-locations`（XML 位置）、`mybatis.type-aliases-package`、`mybatis.configuration.*`（驼峰、缓存、超时）、`mybatis.type-handlers-package`；接口用 `@Mapper` 或统一 `@MapperScan`。

**多数据源**：每个数据源一套 `DataSource + SqlSessionFactory + SqlSessionTemplate + MapperScannerConfigurer`，在 `@MapperScan(basePackages="com.x.a.mapper", sqlSessionTemplateRef="aSqlSessionTemplate")` 里区分（**注意 Mapper 包必须互斥**，同一包不能被两个 `@MapperScan` 扫到）。

**动态数据源**：`AbstractRoutingDataSource#determineCurrentLookupKey` 返回 ThreadLocal 中的数据源 key，用 AOP 在方法/类上切换（`@DS("slave")` 思路）。**两个必须注意的坑**：① 数据源切换必须发生在**获取连接之前**，若外层已开启事务，连接已绑定，切换无效——需要 `LazyConnectionDataSourceProxy` 延迟获取连接；② ThreadLocal 要清理，避免线程池复用时串库。
### 9.2 面试问答
**Q1：动态数据源切换为什么有时不生效？** A：因为连接在事务开始时就已经从 DataSource 获取并绑定到事务上下文，之后改 ThreadLocal 的 key 不会改变已有连接。解决：用 `LazyConnectionDataSourceProxy` 包装目标数据源（真正执行 SQL 时才取连接），并确保 AOP 切面在事务切面**外层**执行（@Order 顺位更靠前）。
**Q2：读写分离怎么做？** A：常见三件套：`AbstractRoutingDataSource` + AOP（按方法名/getter 判断读写）+ 强制走主的注解（如 `@Master`）；配合主从复制延迟，**写后立即读的业务必须强制走主库**，否则出现"刚下单查不到"；强一致场景不要做读写分离。也可用 ShardingSphere-JDBC 的读写分离功能替代自研。
**Q3：多数据源下事务怎么处理？** A：默认 `DataSourceTransactionManager` 只管理一个数据源，跨库写需要分布式事务（见[分布式事务](../05-分布式与架构/02-分布式事务.md)）。若第二个库只是记录日志/审计，可用 `REQUIRES_NEW` 或异步解耦，避免引入分布式事务复杂度。
**Q4：@MapperScan 与 @Mapper 的区别？** A：@Mapper 标在单个接口上（需被扫描到，Boot 自动配置会扫 `@Mapper`）；@MapperScan 在配置类上批量扫描包，适合多模块/多数据源场景；两者可混用但有各自作用域，多数据源时必须用 @MapperScan 分别指定包与 sqlSessionTemplateRef。
### 9.3 高频追问与陷阱
- 陷阱：把两个数据源的 Mapper 放在同一个包下，`@MapperScan` 会冲突导致其中一个数据源报"找不到 mapper"或注入错库。
- 追问：`@Transactional` 与动态数据源切换叠加时，事务管理器与数据源 key 必须一致，否则事务开在了错误的数据源上。
### 9.4 代码/配置示例
```java
@Configuration
@MapperScan(basePackages = "com.x.master.mapper", sqlSessionTemplateRef = "masterSqlSessionTemplate")
public class MasterDataSourceConfig { /* DataSource + SqlSessionFactory + SqlSessionTemplate */ }

public class DynamicDataSource extends AbstractRoutingDataSource {
    @Override protected Object determineCurrentLookupKey() { return DataSourceContextHolder.get(); }
}
@Aspect @Order(Ordered.HIGHEST_PRECEDENCE)   // 必须早于事务切面
@Component
public class DataSourceAspect {
    @Around("@annotation(ds)")
    public Object around(ProceedingJoinPoint pjp, DS ds) throws Throwable {
        DataSourceContextHolder.set(ds.value());
        try { return pjp.proceed(); } finally { DataSourceContextHolder.clear(); }
    }
}
```
### 9.5 一句话速记
多数据源 = 多套 DataSource + SqlSessionFactory + SqlSessionTemplate，Mapper 包必须互斥；动态切换要在取连接前生效，用 LazyConnectionDataSourceProxy。

## 10. MyBatis-Plus 常用能力
### 10.1 核心原理

| 能力 | 用法 | 要点 |
| --- | --- | --- |
| 通用 CRUD | `BaseMapper<T>`、`IService<T>`/`ServiceImpl` | 单表零 SQL；复杂查询仍写 XML |
| 条件构造器 | `QueryWrapper`/`LambdaQueryWrapper`/`UpdateWrapper` | 推荐 Lambda 版，字段名编译期检查；`last()`/`apply()` 慎用（注入风险） |
| 分页 | `PaginationInnerInterceptor` + `IPage<T>` | **必须注册插件**，否则分页无效；默认自动 count，可关闭 |
| 逻辑删除 | `@TableLogic` + `logic-delete-field/`值 | 查询自动加 `deleted=0`；唯一索引需把 deleted 纳入以避免冲突 |
| 自动填充 | `@TableField(fill=INSERT/UPDATE)` + `MetaObjectHandler` | 统一填充 createTime/updateTime/操作人 |
| 乐观锁 | `@Version` + `OptimisticLockerInnerInterceptor` | 更新时带版本条件并自增，判影响行数 |
| 多租户 | `TenantLineInnerInterceptor` | 自动拼接 tenant_id 条件，注意白名单表与自定义 SQL |
| 防全表更新 | `BlockAttackInnerInterceptor` | 拦截无 where 的 update/delete，强烈建议开启 |
| 代码生成器 | `AutoGenerator`（3.5.x 已重构为 FastAutoGenerator 风格，以官方文档为准） | 生成 Entity/Mapper/Service/Controller |
| SQL 注入器 | 实现 `ISqlInjector`/`AbstractMethod` | 自定义通用方法（如 insertIgnore） |

**插件注册顺序（关键）**：多租户 → 动态表名 → 分页 → 乐观锁 → 防全表更新删除（官方建议的分页插件放后面的思路是"分页最后改写 SQL"；具体顺序以官方文档为准，顺序错误会出现分页失效或租户条件丢失）。
### 10.2 面试问答
**Q1：MP 的分页插件为什么不生效？** A：最常见是**忘记注册 `MybatisPlusInterceptor` 并 addInnerInterceptor(new PaginationInnerInterceptor(...))`**；其次是多数据源下插件注册到了另一个 SqlSessionFactory；再次是自定义 XML 里手写了 limit 或用了 `last("limit ...")` 造成双重分页。另外 `IPage` 的 total 为 0 时检查 count SQL 是否被关闭或 count 语句被优化没了。
**Q2：逻辑删除的坑有哪些？** A：① 唯一索引与逻辑删除冲突（删除后同 key 无法再插入）——需把 `deleted` 纳入唯一索引或改用"删除时改写 key"；② 原生 XML 手写 SQL 不会自动加 `deleted=0`，容易查到已删数据；③ 与 MP 的 `selectById` 一致但与其他框架/中间件不一致；④ 数据量膨胀需要归档。
**Q3：乐观锁怎么用才对？** A：实体上加 `@Version`、注册乐观锁插件、更新必须走带版本的 `updateById`/`update(entity, wrapper)`；判断影响行数为 0 即冲突，业务上要重试或返回"请刷新重试"。**不要**用乐观锁保护高冲突场景（大量失败重试），那应该用悲观锁/分布式锁或排队。
**Q4：MP 和原生 MyBatis 怎么选？** A：单表 CRUD 密集的后台系统用 MP 提效明显；复杂报表/联表/分库分表 SQL 仍写 XML（MP 的强项在单表）。**风险点**：`last()`/`apply()` 滥用造成注入、Wrapper 拼错条件造成全表更新、依赖 MP 的不写 SQL 导致性能盲区（看不到慢 SQL）。规范做法是"MP 单表 + XML 复杂查询 + SQL 审计"。
### 10.3 高频追问与陷阱
- 陷阱：`queryWrapper.eq(cond, column, value)` 的 `cond` 参数用错（该传 boolean 判断却传了对象）会导致条件下丢失 → 全表扫描。
- 陷阱：MP 默认开启下划线转驼峰，与原生 MyBatis 的默认值不同，混用时容易误判。
### 10.4 代码/配置示例
```java
@Configuration
public class MybatisPlusConfig {
    @Bean public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());          // 防全表更新删除
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());     // 乐观锁
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL)); // 分页
        return interceptor;
    }
}
```
### 10.5 一句话速记
MP = BaseMapper + Wrapper + 插件（分页/乐观锁/多租户/防全表）；单表用 MP，复杂 SQL 回 XML，`last/apply` 慎用。

## 11. ORM 选型、慢 SQL 与常见坑
### 11.1 核心原理

| 维度 | MyBatis | JPA / Hibernate |
| --- | --- | --- |
| SQL 控制力 | 完全可控（手写/半自动） | 自动生成，需理解生成规则 |
| 学习曲线 | 低（会 SQL 就会） | 高（缓存、脏检查、懒加载、N+1） |
| 动态 SQL | 强（标签化） | 弱（Criteria/Specification 冗长） |
| 联表复杂查询 | 自然 | 较别扭（需 JPQL/原生 SQL） |
| 数据库移植性 | 差（SQL 方言可换但成本高） | 较好（dialect 抽象） |
| DDL 维护 | 手工/Flyway | 可自动生成（生产慎用 auto-ddl） |
| 性能可预测性 | 高（所见即所得） | 低（隐藏的额外查询多） |
| 国内生态 | 主流（+MyBatis-Plus） | 较少（部分外企/中台用） |

**选型结论**：互联网业务以"SQL 可控 + 复杂查询 + 性能可预测"为优先，选 MyBatis（+MP）；领域模型复杂、数据库尽量中立的场景可选 JPA。**面试表达要点：不是好坏之分，而是团队能力与业务形态的匹配；同时要能说出 JPA 的 N+1、懒加载、一级缓存坑与 MyBatis 的 SQL 维护成本。**

**SQL 监控与治理**：① 日志打印——MyBatis 的 `log-impl`（StdOutImpl/Slf4jImpl）或 `logging.level.com.x.mapper=debug`；② p6spy/`log4jdbc` 打印带参数的完整 SQL 与耗时；③ Druid 的 StatFilter 慢 SQL 统计；④ MySQL 慢查询日志（`slow_query_log`、`long_query_time`、`pt-query-digest`）；⑤ APM（SkyWalking/Pinpoint）看调用链与慢 SQL 关联；⑥ 线上诊断用 Arthas（`watch`/`trace`/`tt`）。**规范：ORM 层必须有慢 SQL 阈值与上报，不能依赖"出问题再看日志"。**
### 11.2 面试问答
**Q1：MyBatis 最常见的事故是什么？** A：**漏写 where 条件的全表更新/删除**（动态 SQL 全为 null），其次是 `ORDER BY ${}` 的注入、N+1 拖垮数据库、大事务批量插入导致主从延迟。防御组合：`BlockAttackInnerInterceptor` + 白名单排序参数 + 循环内禁查库的规范 + 分批提交。
**Q2：分页 count 为什么会慢？** A：PageHelper/MP 的自动 count 会把原 SQL 改写成 `select count(*) from (...)`，若 SQL 含 `group by`、多表 join、`distinct`、子查询，count 代价可能比数据查询更高；某些 SQL 还会被改错（丢条件或多算）。解法：复杂分页手写 count、只对主表 count（`count(*) from t_order where ...` 与联表结果行数一致的场景）、加合适索引、必要时改用"游标/滚动分页"（`where id > lastId limit N`）替代深分页。
**Q3：为什么查出来的字段是 null？** A：按概率排序：① 下划线转驼峰未开启（配置位置写错）；② resultMap 漏配字段；③ 列别名与属性不匹配；④ 用了 `PARTIAL` 自动映射但在嵌套 resultMap 内不生效；⑤ 自定义 TypeHandler 未生效；⑥ 逻辑删除/多租户条件把数据过滤掉了（看起来"查不到"但实际是 where 加了条件）。
**Q4：二级缓存导致的脏数据怎么发现与修复？** A：现象是"更新后偶尔读到旧值，重启或过期后恢复"。定位：关掉二级缓存验证、检查是否有跨 Mapper 的写操作、检查部署是否多实例。修复：关闭二级缓存（推荐），缓存交给 Redis，并在 Service 层显式管理失效。
**Q5：MyBatis 项目怎么做代码规范？** A：① 禁止 `SELECT *`（列显式、减少无用列与映射问题）；② 禁止在循环里查库；③ 禁止 `${}` 用于用户输入；④ 必须分页（禁止无 limit 的列表查询）；⑤ 更新语句必须有 where（工具拦截）；⑥ 复杂 SQL 必须 explain 并加索引；⑦ DTO/VO 与 DO 分层，不把实体直接暴露给前端。
### 11.3 高频追问与陷阱
- 陷阱：`SELECT *` 与联表时列名冲突（两个表都有 `name`），导致映射到错误字段——显式列别名。
- 陷阱：p6spy 在性能敏感环境有开销；生产建议按需开启或只在排查期启用。
### 11.4 代码/配置示例
```yaml
mybatis:
  configuration:
    map-underscore-to-camel-case: true
    cache-enabled: false
    log-impl: org.apache.ibatis.logging.slf4j.Slf4jImpl
logging:
  level:
    com.x.mapper: debug        # 仅排查期开启，避免日志风暴
```
### 11.5 一句话速记
国内选 MyBatis（+MP）看中 SQL 可控；事故三大来源是漏 where、`${}` 注入、N+1；慢 SQL 要有阈值与上报机制。

## 自测题（含答案要点）
1. **MyBatis 的执行链路？** SqlSessionFactory → SqlSession → Executor → MappedStatement → StatementHandler → ParameterHandler → JDBC → ResultSetHandler。
2. **SqlSession 线程安全吗？Spring 怎么处理？** 不安全；SqlSessionTemplate 代理 + 绑定 Spring 事务。
3. **四大对象是哪些？** Executor、StatementHandler、ParameterHandler、ResultSetHandler。
4. **插件原理？** @Intercepts + JDK 动态代理责任链，invocation.proceed 放行。
5. **PageHelper 原理？** 拦截 Executor#query，先 count 再改写 limit，参数用 ThreadLocal 传递。
6. **`#{}` 与 `${}` 的区别？** 预编译占位符（防注入、可复用执行计划） vs 字符串拼接（注入风险）。
7. **order by 参数怎么处理？** 白名单映射列名 + 方向枚举，绝不直接拼用户输入。
8. **一级缓存作用域与失效条件？** SqlSession/事务内；写操作、commit/rollback、flushCache、换 Session 即失效。
9. **为什么不建议二级缓存？** 多实例不一致、跨 namespace 写不失效、联表脏数据，交给 Redis 更可控。
10. **延迟加载生效条件？** 嵌套 select 映射 + lazyLoadingEnabled=true + 在事务内访问。
11. **N+1 怎么解决？** 主表分页 + 批量 in 装配 / JOIN 嵌套映射 / 禁止循环内查库。
12. **下划线转驼峰失效原因？** 配置位置写错、resultMap 显式覆盖、MyBatis 与 MP 默认值不同。
13. **枚举映射与 JSON 字段注意什么？** 枚举别用 Ordinal（用业务 code 或自定义 TypeHandler）；JSON 字段需自定义 TypeHandler，且在 resultMap 里也要指定。
14. **主键回填怎么做？** 自增用 useGeneratedKeys + keyProperty，非自增用 selectKey；批量插入只保证首条 id。
15. **批量插入性能怎么优化？** 分批 500~1000 条、ExecutorType.BATCH + flushStatements、rewriteBatchedStatements、避免大事务。
16. **动态数据源切换为什么失效？** 事务已绑定连接；用 LazyConnectionDataSourceProxy 并让切面早于事务切面。
17. **MP 分页不生效的原因？逻辑删除有什么坑？** 未注册 MybatisPlusInterceptor / PaginationInnerInterceptor；逻辑删除会与唯一索引冲突，且 XML 手写 SQL 不会自动加 deleted 条件。
18. **MyBatis 与 JPA 怎么选？** SQL 可控与复杂查询选 MyBatis，数据库中立与领域模型选 JPA。
19. **线上慢 SQL 怎么定位？** SQL 日志/p6spy、Druid 慢 SQL、MySQL 慢查询日志、APM、Arthas，再 explain 优化索引。

## 背诵卡片
1. MyBatis = 半自动 ORM：SQL 自己写，映射交给框架。
2. SqlSessionFactory 单例安全；SqlSession 不安全，Spring 用 SqlSessionTemplate 管。
3. Executor 三兄弟：Simple（默认）、Reuse、Batch；二级缓存时被 CachingExecutor 装饰。
4. 四大对象：Executor、StatementHandler、ParameterHandler、ResultSetHandler。
5. 插件用 @Intercepts + @Signature 指定拦截点，Plugin.wrap 做 JDK 代理；后注册的在外层先执行。
6. PageHelper 拦 Executor#query，自动 count 可能比数据查询还慢。
7. `<where>`/`<set>` 自动裁剪，`<foreach>` 做批量与 in。
8. `<if>` 判断用 `!= null`，字符串再判空，避免 OGNL 数字与空串比较的坑；`<set>` 全 null 会生成非法 SQL。
9. 最危险事故：动态 SQL 全为 null → 无 where 的全表更新/删除。
10. 值用 `#{}`（预编译），结构用 `${}`（白名单）。
11. order by 用白名单映射列名，方向只允许 ASC/DESC；like 用 `concat('%', #{name}, '%')`。
12. MP 的 `last()`、`apply()` 是注入高发点。
13. 一级缓存在 SqlSession/事务内，写操作清空全部；无事务时几乎不命中。
14. 二级缓存跨 namespace，多实例 + 联表必然脏数据，建议关闭；且实体需可序列化。
15. 延迟加载需 lazyLoadingEnabled=true + 嵌套 select，且必须在事务内。
16. Jackson 序列化实体触发懒加载，是 N+1 的隐形来源。
17. N+1 解法：主表分页 + 批量 in 装配；一对多 JOIN 会放大行数并影响分页。
18. 自动映射 PARTIAL 默认，嵌套 resultMap 内字段不自动映射。
19. mapUnderscoreToCamelCase 在 MyBatis 默认 false，MP 默认 true。
20. 枚举映射别用 Ordinal；JSON 字段需自定义 TypeHandler 并在 resultMap 指定。
21. 主键回填：useGeneratedKeys + keyProperty；批量只保证首条 id。
22. 批量插入分批 500~1000 条；ExecutorType.BATCH 必须 flushStatements 才执行。
23. rewriteBatchedStatements=true 能显著提升批量插入。
24. 多数据源：Mapper 包必须互斥，各自 SqlSessionTemplate。
25. 动态数据源要在取连接前切换，否则事务已绑定旧连接。
26. MP 分页必须注册 MybatisPlusInterceptor。
27. 逻辑删除与唯一索引冲突需把 deleted 纳入索引。
28. 乐观锁用 @Version + 插件，冲突时判影响行数并重试；防全表更新用 BlockAttackInnerInterceptor。
29. 分页 count 慢：复杂 SQL 手写 count 或用游标分页。
30. SELECT * 会引发列名冲突与无用传输，禁止使用；慢 SQL 治理靠 SQL 日志 + p6spy + 慢查询日志 + APM + explain。
