# Spring Boot 与自动装配

> **定位**：讲清 Spring Boot 做了什么、自动装配怎么实现、配置体系与启动优化怎么做，覆盖从"问原理"到"线上调优"的完整链条。
> **重要度**：★★★★★（社招 3-5 年必问，自动装配原理与配置优先级是高频切入点）
> **建议复习时长**：4~6 小时（建议亲手写一个自定义 starter）
> **关联文档**：[Spring 核心 IoC 与 AOP](./01-Spring核心IoC与AOP.md)、[Spring MVC 与 Web 容器](./03-SpringMVC与Web容器.md)、[Spring Cloud 微服务体系](./05-SpringCloud微服务体系.md)、[Docker 与 Kubernetes](../07-工程化与DevOps/02-Docker与Kubernetes.md)、[CI/CD 与监控告警](../07-工程化与DevOps/03-CICD与监控告警.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| Spring Boot 定位与启动流程 | ★★★★★ | 源码追问 |
| 自动装配原理与 imports 文件演进 | ★★★★★ | 源码追问 / 八股原理 |
| 条件注解与自动配置覆盖 | ★★★★★ | 源码追问 / 配置实践 |
| 自定义 Starter | ★★★★☆ | 手写代码 / 配置实践 |
| 配置体系与优先级 | ★★★★★ | 配置实践 / 故障排查 |
| @ConfigurationProperties vs @Value | ★★★★☆ | 八股原理 / 配置实践 |
| 内嵌容器与启动优化 | ★★★★☆ | 场景题 / 故障排查 |
| Actuator、健康检查与优雅停机 | ★★★★☆ | 配置实践 / 故障排查 |
| 日志体系与 MDC/TraceId | ★★★★☆ | 配置实践 / 故障排查 |
| Boot 2.x 升级 3.x 破坏性变化 | ★★★★☆ | 场景题 |
| 高频连环追问（排除/覆盖/proxyBeanMethods） | ★★★★★ | 八股原理 / 场景题 |

## 1. Spring Boot 定位与启动流程
### 1.1 核心原理
Spring Boot 解决三个问题：**依赖管理（起步依赖版本对齐）、配置自动化（自动装配）、部署简化（内嵌容器打成可执行 jar）**。它不替代 Spring，只是把样板配置收敛成约定。

| run() 阶段 | 关键动作 |
| --- | --- |
| 1 构造 SpringApplication | 推断 Web 类型（deduceWebApplicationType：WebFlux→REACTIVE、有 Servlet→SERVLET、否则 NONE）、从 spring.factories/imports 加载 ApplicationContextInitializer 与 Listener、推断主类 |
| 2 启动计时与监听器 | 创建 SpringApplicationRunListeners，发布 ApplicationStartingEvent |
| 3 prepareEnvironment | 创建 Environment、用 ConfigData API 加载配置、绑定 spring.main.*、发布 ApplicationEnvironmentPreparedEvent |
| 4 Banner 与日志系统 | printBanner（`spring.main.banner-mode=off` 可关闭） |
| 5 createApplicationContext | 按 Web 类型创建 AnnotationConfigServletWebServerApplicationContext 等 |
| 6 prepareContext | 设置 Environment、执行 Initializer、发布 ApplicationContextInitializedEvent、加载主配置类 BeanDefinition |
| 7 refreshContext | 调用 AbstractApplicationContext#refresh（第 9 步 onRefresh 启动内嵌容器） |
| 8 afterRefresh | 自 Spring Boot 2.4 起已废弃，不要再依赖 |
| 9 callRunners | 执行 ApplicationRunner / CommandLineRunner |
| 10 完成 | 发布 ApplicationStartedEvent、AvailabilityChangeEvent(readiness=ACCEPTING_TRAFFIC)、ApplicationReadyEvent；失败则 ApplicationFailedEvent |

事件顺序记忆：Starting → EnvironmentPrepared → ContextInitialized → ContextPrepared（注入主类后）→ ContextLoaded → Started → Ready；失败时 Failed。Liveness/Readiness 状态可用于 K8s 探针（`management.endpoint.health.probes.enabled=true`）。
### 1.2 面试问答
**Q1：Spring Boot 和 Spring 的关系是什么？** A：Boot 是 Spring 的**脚手架与自动装配层**，不是新框架。它做的事：起步依赖（版本对齐）、自动装配（@EnableAutoConfiguration + 条件装配）、内嵌容器（Tomcat/Jetty/Undertow）、外部化配置与 Actuator。Boot 项目里跑的仍是标准 Spring ApplicationContext。
**Q2：说一下 Spring Boot 的启动流程。** A：构造 SpringApplication（推断 Web 类型、加载 Initializer/Listener、推断主类）→ 发布 Starting 事件 → prepareEnvironment → printBanner → createApplicationContext → prepareContext（执行 Initializer、加载主类定义）→ refreshContext（真正的容器 refresh，内嵌容器在 onRefresh 启动）→ callRunners → 发布 Ready 事件。加分点：指出内嵌容器在 refresh 第 9 步启动，所以端口绑定失败发生在 refreshContext 阶段。
**Q3：内嵌容器是在哪一步启动的？** A：`ServletWebServerApplicationContext#onRefresh` → `createWebServer()`，即 refresh 的第 9 步。因此端口占用、Servlet 初始化异常都表现为 refresh 阶段失败，而不是"启动后"。
**Q4：怎么关闭 Banner、怎么改 Web 类型？** A：`spring.main.banner-mode=off`（或调用 `setBannerMode`）；Web 类型可强制 `spring.main.web-application-type=none|servlet|reactive`。注意 webflux 与 web 同时存在时默认 WebFlux，需要显式指定。
### 1.3 高频追问与陷阱
- 陷阱：afterRefresh 自 Boot 2.4 起已是空实现（废弃），答"在 afterRefresh 里扩展"是过时答案。
- 追问：main 方法所在类不一定是配置类，但 `SpringApplication.run` 的入参类会被注册为 BeanDefinition。
### 1.4 代码/配置示例
```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = SpringApplication.run(App.class, args);
    }
}
```
### 1.5 一句话速记
启动流程 = 构造 SpringApplication → 准备环境 → 建容器 → prepareContext → refresh（内嵌容器启动）→ callRunners → 发 Ready 事件。

## 2. 自动装配原理与 imports 文件演进
### 2.1 核心原理
`@SpringBootApplication` = `@SpringBootConfiguration`（本质是 @Configuration）+ `@EnableAutoConfiguration` + `@ComponentScan`（默认扫描主类所在包及子包）。自动装配链路：

1. `@EnableAutoConfiguration` 通过 `@Import(AutoConfigurationImportSelector.class)` 引入选择器。
2. 该选择器实现 `DeferredImportSelector`，在用户配置之后处理（保证用户配置优先）。
3. 通过 `ImportCandidates.load` 读取候选类名（位置见下表）。
4. 去重 → 按 `@AutoConfigureOrder`/`@AutoConfigureAfter`/`@AutoConfigureBefore` 排序 → 经 `AutoConfigurationImportFilter`（OnClassCondition/OnBeanCondition + `spring-autoconfigure-metadata.properties`）在不加载类的前提下快速过滤。
5. 剩下的类进入容器，由类上的 `@Conditional*` 决定是否真正生效。

| 版本 | 自动装配文件位置 | 说明 |
| --- | --- | --- |
| Boot ≤ 2.6 | `META-INF/spring.factories` | key 为 `org.springframework.boot.autoconfigure.EnableAutoConfiguration` |
| Boot 2.7 | 新增 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` | 两种方式兼容，spring.factories 方式已弃用 |
| Boot ≥ 3.0 | **只支持 `.imports` 文件** | 自动装配不再从 spring.factories 读取；其他扩展点也逐步迁移到 `META-INF/spring/<接口全名>.imports` |

`.imports` 格式：一行一个自动配置类全限定名，`#` 开头为注释。
### 2.2 面试问答
**Q1：Spring Boot 自动装配原理是什么？** A：一句话：**通过 @EnableAutoConfiguration 引入 AutoConfigurationImportSelector，读取 classpath 上的 .imports（旧版 spring.factories）候选类名，经去重、排序、条件过滤后按需注册成配置类**；每个自动配置类内部再用 `@ConditionalOnClass`、`@ConditionalOnMissingBean` 等决定是否真正生效，实现"约定优先、用户可覆盖"。
**Q2：为什么自动装配类大量使用 @ConditionalOnMissingBean？** A：为了让**用户配置优先**。自动配置类通常用 `@ConditionalOnMissingBean` 包住 @Bean 方法：用户自己定义了同类型 Bean，默认配置就退让，既不冲突也不覆盖用户设置。这也是"自动装配可以关闭/被替换"的根本原因。
**Q3：Boot 2.7 / 3.0 的 spring.factories 变化是什么？** A：Boot 2.7 引入 `.imports` 文件并弃用 spring.factories 中的 EnableAutoConfiguration 键（两者并存）；Boot 3.0 起彻底不再从 spring.factories 读取自动配置，必须写在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`。**升级到 3.x 时自定义 starter 若还在 spring.factories 里写自动配置，会静默失效**——升级踩坑高发点。
**Q4：@AutoConfigureAfter/@AutoConfigureBefore/@AutoConfigureOrder 有什么用？** A：控制自动配置类之间的注册顺序。需要的原因是 `@ConditionalOnBean` 依赖"别人先注册"（如依赖 DataSource 已存在，就必须排在 DataSourceAutoConfiguration 之后）。注意它们只对 .imports 中登记的自动配置类生效，对普通 @Configuration 无效，也不保证 Bean 的实例化顺序（那由依赖关系决定）。
### 2.3 高频追问与陷阱
- 追问：`@EnableAutoConfiguration` 的 `exclude`/`excludeName` 属性、配置项 `spring.autoconfigure.exclude` 都能排除自动配置。
- 陷阱：候选类被加载但条件不满足时不会报错，容易误判"配置没生效"——要看 ConditionEvaluationReport。
### 2.4 代码/配置示例
```
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.x.log.LogAutoConfiguration
```
```yaml
spring:
  autoconfigure:
    exclude: org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```
### 2.5 一句话速记
自动装配 = @EnableAutoConfiguration → ImportSelector 读 .imports → 排序过滤 → @Conditional 按需生效。

## 3. 条件注解与自动配置覆盖
### 3.1 核心原理

| 条件注解 | 生效条件 | 典型用途 |
| --- | --- | --- |
| @ConditionalOnClass / OnMissingClass | classpath 是否存在指定类 | 有 Redis 依赖才装配 RedisTemplate |
| @ConditionalOnBean / OnMissingBean | 容器中是否存在/不存在指定 Bean | 默认实现退让（用户覆盖） |
| @ConditionalOnSingleCandidate | 只有唯一候选 Bean（或有 @Primary） | 只在一个 DataSource 时生效 |
| @ConditionalOnProperty | 配置项存在且值匹配（havingValue、matchIfMissing） | `xxx.enabled=true` 开关 |
| @ConditionalOnResource | 资源文件存在 | 配置文件驱动 |
| @ConditionalOnWebApplication / OnNotWebApplication | Web 类型判断 | Web 专属组件 |
| @ConditionalOnExpression | SpEL 为 true | 复杂条件 |
| @ConditionalOnJava | JDK 版本 | 版本兼容 |

**自定义覆盖默认配置的三种方式**：① 定义同类型（推荐还同名）Bean，让 `@ConditionalOnMissingBean` 退让；② `spring.autoconfigure.exclude` 或 `@EnableAutoConfiguration(exclude=...)` 直接排除；③ 用配置项关掉（如 `spring.session.store-type=none`）。**开启 allow-bean-definition-overriding 是最后手段**，Boot 2.1+ 默认 false，开启后行为不可预测。**诊断手段**：启动加 `--debug`（或 `debug=true`）打印 ConditionEvaluationReport（Positive/Negative matches、Exclusions、Unconditional classes）；Actuator `/actuator/conditions` 可在线看同一份报告。
### 3.2 面试问答
**Q1：条件注解的实现原理？** A：所有 `@ConditionalOnXxx` 最终归到 `@Conditional(OnXxxCondition.class)`，Spring 在处理 @Configuration/@Bean 时调用 `Condition#matches` 决定是否注册。Boot 还做了性能优化：`spring-autoconfigure-metadata.properties` 让 `AutoConfigurationImportFilter` 在**不加载类**的前提下先做类/Bean 过滤，避免为无效自动配置加载类元数据（对启动速度影响很大）。
**Q2：用户自定义的 Bean 会不会和自动配置冲突？** A：通常不会。自动配置的 @Bean 方法一般带 `@ConditionalOnMissingBean`，用户定义后它直接不注册。例外：**同名但类型不同**的 Bean 会触发 BeanDefinition 覆盖，`allow-bean-definition-overriding=false` 时启动失败（报 `Cannot register bean definition ... since there is already ... bound`），开启则为后者生效（顺序不可控）。所以自定义 Bean 时注意 beanName。
**Q3：怎么排查"某个自动配置为什么没生效"？** A：① 启动加 `--debug`，在 ConditionEvaluationReport 的 Negative matches 里找该配置类，会打印具体哪个条件不满足（如 `@ConditionalOnClass did not find required class 'redis.clients.jedis.Jedis'`）；② Actuator `/actuator/conditions`；③ 检查是否被 exclude、是否缺依赖、是否被 `@ConditionalOnProperty` 的 matchIfMissing 误判。
**Q4：@ConditionalOnBean 有什么坑？** A：**顺序敏感**。它只看到"当前已注册的 BeanDefinition"，若目标 Bean 由更晚的自动配置注册就会判断不成立，必须配合 `@AutoConfigureAfter`。官方明确建议它只用在自动配置类上，普通 @Configuration 上的结果不可靠。
### 3.3 高频追问与陷阱
- 陷阱：`@ConditionalOnProperty` 不写 havingValue 时，只要属性存在且不为 "false" 即成立；`matchIfMissing=true` 表示属性缺失也成立。
- 陷阱：`@ConditionalOnClass` 判断的是类是否可加载，用 `name` 属性写字符串可避免类加载失败导致整个配置类报错。
### 3.4 代码/配置示例
```java
@AutoConfiguration
@ConditionalOnClass(RedisTemplate.class)
@AutoConfigureAfter(RedisAutoConfiguration.class)
public class MyCacheAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean                // 用户自定义同类型 Bean 时让位
    public CacheService cacheService(ObjectProvider<RedisTemplate<String, Object>> t) {
        return new RedisCacheService(t.getIfAvailable());
    }
}
```
### 3.5 一句话速记
自动配置靠条件注解按需生效；用户 Bean 优先（@ConditionalOnMissingBean）；不生效就查 ConditionEvaluationReport。

## 4. 自定义 Starter
### 4.1 核心原理
**命名规范**：官方 starter 为 `spring-boot-starter-xxx`；第三方/内部 starter 为 `xxx-spring-boot-starter`（放自己 groupId 下，避免与官方冲突）。

| 模块 | 内容 | 说明 |
| --- | --- | --- |
| xxx-spring-boot-autoconfigure | 自动配置类、`XxxProperties`、`.imports` 文件、配置元数据 | 只依赖真正需要的库，其余用 optional，避免污染用户 classpath |
| xxx-spring-boot-starter | 空 POM，聚合 autoconfigure + 默认实现依赖 | 用户只需引入这一个依赖 |

**必做步骤清单**：① 写自动配置类（`@AutoConfiguration` + `@ConditionalOnClass` 保护）；② 写 `@ConfigurationProperties` 承载配置项，通过 `@EnableConfigurationProperties` 或 `@ConfigurationPropertiesScan` 注册；③ 每个 @Bean 加 `@ConditionalOnMissingBean` 让用户可覆盖；④ 在 `.imports` 文件中登记（Boot 3 唯一方式）；⑤ 引入 `spring-boot-configuration-processor` 生成 IDE 提示元数据；⑥ starter 模块聚合依赖；⑦ 用 `ApplicationContextRunner` 写测试；⑧ README 说明配置项与默认值。
### 4.2 面试问答
**Q1：starter 和自动装配是一回事吗？** A：不是。**starter 是依赖聚合的 POM**（解决"引哪些包"），**自动装配是代码**（解决"怎么配"）。官方 starter 通常是"POM 聚合 + 对应 autoconfigure 模块"，两者成对出现但职责分离：starter 负责"带什么进来"，自动装配负责"进来之后怎么装"。
**Q2：写一个 starter 需要注意什么？** A：① 自动配置类必须能被 `.imports` 找到（Boot 3 用 spring.factories 会静默失效）；② 所有 Bean 都要有 @ConditionalOnMissingBean 让用户可覆盖；③ 配置类要有 @ConditionalOnClass 保护，避免用户未引入依赖时启动失败；④ 配置项加默认值与校验（@Validated + JSR-303）；⑤ 不要强依赖具体实现（Logback、具体连接池）；⑥ 生成配置元数据，避免用户猜配置名。
**Q3：starter 里怎么让用户"不完全用默认实现"？** A：接口分层 + 条件装配：starter 只暴露接口与一个合理的默认实现（`@ConditionalOnMissingBean`），用户定义自己的实现即可替换；同时提供 `enabled` 开关（`@ConditionalOnProperty(name = "xxx.enabled", havingValue = "true", matchIfMissing = true)`）。
**Q4：ApplicationContextRunner 有什么用？** A：它是测试自动配置的轻量工具：不用完整启动 Boot，可按需加配置类、设属性并断言容器内容（`withClassLoader`、`withPropertyValues`、`withUserConfiguration`），断言是否创建了指定 Bean，或打印条件报告，非常适合 starter 单元测试。
### 4.3 高频追问与陷阱
- 陷阱：忘记写 `.imports` 文件或路径写错（必须是 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`），表现为"引入 starter 后什么都没发生"。
- 陷阱：自动配置类上写 `@ComponentScan` 会导致重复扫描甚至把用户包里的类意外注册，官方明确不推荐。
### 4.4 代码/配置示例
```java
@AutoConfiguration
@EnableConfigurationProperties(LogProperties.class)
@ConditionalOnProperty(prefix = "my.log", name = "enabled", havingValue = "true", matchIfMissing = true)
public class LogAutoConfiguration {
    @Bean @ConditionalOnMissingBean
    public LogService logService(LogProperties props) { return new DefaultLogService(props.getLevel()); }
}
@ConfigurationProperties(prefix = "my.log") @Validated
public class LogProperties { private String level = "INFO"; /* getter/setter */ }
```
### 4.5 一句话速记
Starter 分两层（autoconfigure 写代码 + starter 聚依赖）；三件套是 @AutoConfiguration + @ConditionalOnMissingBean + .imports 文件。

## 5. 配置体系与优先级
### 5.1 核心原理
**配置文件加载位置**（Boot 2.4+ 统一用 ConfigData API，细节以官方文档为准）：classpath 的 `application.*` 与 `config/application.*`、当前目录的 `application.*` 与 `config/application.*`；**同一位置 properties 优先于 yml**；**profile 专属文件 `application-{profile}.*` 优先于通用文件**。

**属性优先级（从高到低，常见片段）**：命令行参数（`--server.port=8081`）> `SPRING_APPLICATION_JSON` > ServletConfig/ServletContext 参数 > JNDI > Java System properties（`-D`）> 操作系统环境变量 > `random.*` > profile 专属外部文件 > profile 专属 jar 内文件 > 外部 application 文件 > jar 内 application 文件 > `@PropertySource` > `SpringApplication.setDefaultProperties`。**@PropertySource 在这个顺序里很低**，"加了 @PropertySource 却不生效"多是被配置中心或命令行覆盖。

**Profile**：`spring.profiles.active`（激活）、`spring.profiles.include`（追加）、`spring.config.activate.on-profile`（2.4+ 文档块内条件，替代旧的文档块内 `spring.profiles`）；多文档用 `---` 分隔。生产实践：`application.yml` 放公共项 + `application-dev/test/prod.yml` 放差异项 + 环境变量注入密钥。

| 维度 | @ConfigurationProperties | @Value |
| --- | --- | --- |
| 绑定方式 | 批量绑定到对象（前缀） | 单个属性 |
| 松散绑定 | 支持（my-app.name ↔ myAppName） | 不支持 |
| SpEL | 不支持 | 支持 |
| 类型转换与校验 | 支持 JSR-303、嵌套对象、List/Map、Duration | 弱，需手写校验 |
| 默认值 | 字段初始化 | `:` 冒号语法 |
| 适用场景 | 一组配置、复杂结构 | 单个简单值（开关、单个 URL） |
### 5.2 面试问答
**Q1：Spring Boot 配置优先级怎么排？** A：记三条口诀：**命令行 > 环境变量/系统属性 > 配置文件；外部 > 内部；profile 专属 > 通用**。所以同一个 key 在 jar 外 `config/application-prod.yml` 与 jar 内 `application.yml` 同时存在时，外部且 profile 专属的胜出。验证方式：Actuator `/actuator/env` 会列出属性来源与优先级。
**Q2：@ConfigurationProperties 和 @Value 怎么选？** A：一组相关配置、需要类型安全/嵌套/校验/松散绑定时用 `@ConfigurationProperties`（官方推荐）；单个简单值或需要 SpEL 时用 @Value。注意 @Value 不支持松散绑定：配置里写 `my-app.name` 时 `@Value("${my-app.name}")` 可读，但配置写成 `myAppName` 就读不到——迁移中最常见的坑。
**Q3：什么是松散绑定？有哪些形式？** A：把配置文件中的属性名按多种等价写法绑定到字段：`my-app.name`（推荐 kebab-case）、`my_app_name`、`myAppName`、`MY_APP_NAME`（环境变量场景）都能绑定到 `myAppName` 字段。规则：@ConfigurationProperties 的 prefix 必须用 kebab-case。
**Q4：怎么保证配置不会写错？** A：① 类上加 `@Validated`、字段上加 `@NotBlank`/`@Min`/`@Pattern`，启动期即校验失败；② 用 `spring-boot-configuration-processor` 生成元数据让 IDE 补全；③ 关键配置用不可变绑定（构造器绑定，Boot 3 中单构造器自动生效）；④ 配置中心 + 灰度发布 + 变更审计。**配置加密**常用方案：Jasypt（`ENC(...)` + 环境变量传盐）、配置中心自带加密/KMS、云厂商 KMS/Vault、K8s Secret；原则是**密钥本身不能进代码仓库**。
### 5.3 高频追问与陷阱
- 陷阱：Boot 2.4 之后文档块内不能再写 `spring.profiles`，必须写 `spring.config.activate.on-profile`，写错的表现是"条件被忽略"。
- 陷阱：`@PropertySource` 只支持 properties，不支持 yml——这是"@PropertySource 加载 yml 失败"的根因。
### 5.4 代码/配置示例
```yaml
spring:
  profiles:
    active: dev
---
spring:
  config:
    activate:
      on-profile: dev
my-app:
  name: demo
  timeout: 3s
```
```java
@ConfigurationProperties(prefix = "my-app") @Validated
public class MyAppProperties {
    @NotBlank private String name;
    private Duration timeout = Duration.ofSeconds(5);
    // getter / setter
}
```
### 5.5 一句话速记
命令行 > 环境变量 > 配置文件，外部 > 内部，profile 专属 > 通用；一组配置用 @ConfigurationProperties。

## 6. 内嵌容器与启动优化
### 6.1 核心原理
**内嵌容器原理**：Boot 在 refresh 的 onRefresh 阶段由 `ServletWebServerFactory`（Tomcat/Jetty/Undertow）创建并启动 WebServer；`WebServerFactoryCustomizer` 用于定制端口、线程池、连接器。换容器的方式是排除 `spring-boot-starter-tomcat` 并引入 jetty/undertow starter。

| 优化手段 | 做法 | 收益 / 代价 |
| --- | --- | --- |
| 全局懒加载 | `spring.main.lazy-initialization=true` | 启动快；首请求变慢、错误延后暴露（生产不建议全量开） |
| 排除无用自动装配 | `spring.autoconfigure.exclude` | 直接减少无效 Bean 与类加载 |
| 收敛组件扫描 | 缩小 `@ComponentScan` 包范围 | 显著减少元数据读取 |
| 关闭 JMX | `spring.jmx.enabled=false` | 小幅提速 |
| AOT / Native | Boot 3 构建期 AOT 处理 + GraalVM native image | 原生镜像秒级启动、低内存；构建慢、反射需配置、生态有坑 |
| CDS | 训练运行生成类归档并在启动时加载（Boot 3.3+ 在构建插件中提供支持，以官方文档为准） | JVM 启动提速，改动小 |
| lite 模式 | `@Configuration(proxyBeanMethods=false)` | 少生成 CGLIB 子类，启动更快 |

**AOT 注意点**：native image 需要在构建期确定反射、动态代理与资源，Spring AOT 会生成相应配置；`@Profile`/`@ConditionalOnProperty` 在构建期求值，因此**同一份原生镜像不能靠环境变量切换 profile 行为**（Boot 3 支持部分运行期属性，但需评估）。
### 6.2 面试问答
**Q1：Boot 项目怎么换掉 Tomcat？** A：从 starter-web 中排除 `spring-boot-starter-tomcat`，加入 `spring-boot-starter-jetty` 或 `spring-boot-starter-undertow`；用 `spring-boot-starter-webflux` 时默认 Reactor Netty。注意 Servlet API 命名空间要与 Boot 版本一致（Boot 3 为 jakarta）。
**Q2：`spring.main.lazy-initialization=true` 有什么风险？** A：① 首次请求延迟飙升（Bean 初始化落在请求链路里）；② 启动期本可暴露的配置错误延后到运行时；③ 依赖启动顺序的组件（定时任务、MQ 消费者、缓存预热）可能异常。适用场景：本地开发、测试环境、能接受首请求抖动的服务。
**Q3：AOT 解决了什么问题？和 CDS 的区别？** A：AOT 在**构建期**把反射/代理/资源的使用固化成代码与元数据，使应用不依赖运行期动态行为（原生镜像必需，也能加速 JVM 启动）；CDS 只是把已加载类的元数据归档供运行期共享，改动小、收益中等。两者可叠加，AOT 对生态兼容性要求更高。
**Q4：想定位"启动慢在哪"怎么做？** A：用 Actuator `/actuator/startup`（需配置 `BufferingApplicationStartup` 并开启端点）得到各启动步骤耗时；也可用 `--debug` 看自动配置、`-Xlog:class+load` 看类加载、JFR/async-profiler 采样。常见结论：Bean 数量、类路径扫描范围、自动装配数量、外部依赖（连接池/MQ/缓存）初始化是四大来源。
### 6.3 高频追问与陷阱
- 陷阱：`lazy-initialization=true` 后 `@PostConstruct` 里的预热逻辑也不会执行，缓存可能到首次请求才加载。
- 追问：Boot 3 的 AOT 处理由 `spring-boot-maven-plugin` 的 `process-aot` goal 完成，普通 jar 打包也可开启以获得启动收益。
### 6.4 代码/配置示例
```yaml
spring:
  main:
    lazy-initialization: false
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jmx.JmxAutoConfiguration
  jmx:
    enabled: false
```
### 6.5 一句话速记
内嵌容器在 refresh 的 onRefresh 启动；启动优化优先级：减少扫描/排除自动装配 > CDS > AOT/Native > 懒加载。

## 7. Actuator、健康检查与优雅停机
### 7.1 核心原理

| 端点 | 作用 | 是否默认暴露 |
| --- | --- | --- |
| /actuator/health | 健康检查（可含 details） | 是（details 默认不显示） |
| /actuator/info | 自定义信息 | 是 |
| /actuator/metrics、/prometheus | 指标（Micrometer） | 否 |
| /actuator/env、/configprops | 属性来源与配置项 | 否（敏感） |
| /actuator/conditions、/beans、/mappings | 自动配置条件、Bean 列表、请求映射 | 否 |
| /actuator/loggers | 在线调日志级别 | 否 |
| /actuator/threaddump、/heapdump、/startup | 线程/堆快照、启动步骤耗时 | 否 |
| /actuator/shutdown | 关闭应用 | 否（默认关闭，生产慎开） |

默认只暴露 health 与 info（由 `management.endpoints.web.exposure.include` 控制）。**生产必须收口**：`management.server.port` 单独开管理端口 + 只绑内网 + Spring Security 鉴权 /actuator/**。**优雅停机**：`server.shutdown=graceful`（Boot 2.3+，Tomcat/Jetty/Undertow 支持）+ `spring.lifecycle.timeout-per-shutdown-phase=30s`；行为是停止接收新请求 → 等待在途请求完成或超时 → 关闭容器与连接池。K8s 场景还要 `preStop` sleep + 足够 `terminationGracePeriodSeconds`，否则"Endpoint 摘除"与"进程退出"之间的时间差仍会丢请求。
### 7.2 面试问答
**Q1：Actuator 生产环境敢开吗？怎么开？** A：敢开但要收口：① 只暴露必要端点（health、metrics、prometheus、loggers）；② 管理端口独立（`management.server.port=9090`）并只绑内网；③ 加 Spring Security 鉴权与角色隔离；④ health 的 details 设为 `when-authorized` 或 `never`；⑤ `/actuator/shutdown`、`/heapdump`、`/threaddump` 谨慎开放。
**Q2：健康检查是怎么聚合的？** A：HealthEndpoint 由 `HealthContributorRegistry` 聚合多个 HealthIndicator（DB、Redis、磁盘、自定义），各自返回 UP/DOWN/UNKNOWN/OUT_OF_SERVICE，聚合顺序由 `management.endpoint.health.status.order` 决定，细节由 `show-details` 控制。K8s 常用分组：`management.endpoint.health.group.readiness.include=readinessState,db`。
**Q3：优雅停机怎么做？** A：容器层设 `server.shutdown=graceful` 与合理超时；K8s 加 preStop（sleep 5~10s 等 Endpoint 摘除生效）+ `terminationGracePeriodSeconds` 大于超时；MQ 消费者手动停止拉取并等待在途消息；异步线程池设 `setWaitForTasksToCompleteOnShutdown(true)` 与 `setAwaitTerminationSeconds(...)`。
**Q4：为什么应用"启动成功"但 K8s 一直重启？** A：常见原因：① liveness 探针里包含了依赖下游的健康检查（DB 抖动导致误杀）——**liveness 只应反映自身进程状态**；② 探针路径被 Spring Security 拦截返回 401/403；③ 启用了独立管理端口，探针还打原端口；④ 启动慢于 initialDelaySeconds。
### 7.3 高频追问与陷阱
- 陷阱：liveness 与 readiness 混用同一组健康检查，下游抖动时 Pod 被不断重启，放大故障。
- 追问：`/actuator/loggers` 支持 POST 修改级别（`{"configuredLevel":"DEBUG"}`），是线上排障利器，但必须鉴权。
### 7.4 代码/配置示例
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true
  server:
    port: 9090
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```
### 7.5 一句话速记
Actuator 默认只开 health/info，生产要独立端口 + 鉴权；优雅停机 = graceful + preStop + 足够 grace period。

## 8. 日志体系与 MDC/TraceId
### 8.1 核心原理
Boot 默认用 Logback（换 Log4j2：排除 spring-boot-starter-logging 并引入 spring-boot-starter-log4j2）。级别顺序 OFF > ERROR > WARN > INFO > DEBUG > TRACE；启动期可用 `--debug`/`--trace`，运行时用 `/actuator/loggers`。推荐文件名 `logback-spring.xml`（`-spring` 后缀才能用 `<springProfile>`/`<springProperty>` 标签）。**MDC** 是 SLF4J/Logback 的 ThreadLocal Map，日志 pattern 用 `%X{traceId}` 取值；入口 put、出口 `MDC.clear()`（**必须清理，线程池复用时否则会串号**），跨线程（@Async、线程池、MQ）需手动传递。Boot 3 + Micrometer Tracing 可自动填充 traceId/spanId（见 [Spring Cloud 篇](./05-SpringCloud微服务体系.md)）。
```xml
<configuration>
  <springProperty scope="context" name="APP" source="spring.application.name"/>
  <springProfile name="prod">
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
      <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>logs/${APP}-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
        <maxFileSize>200MB</maxFileSize>      <!-- 单文件上限 -->
        <maxHistory>15</maxHistory>           <!-- 保留天数 -->
        <totalSizeCap>20GB</totalSizeCap>     <!-- 总容量上限 -->
      </rollingPolicy>
      <encoder><pattern>%d{HH:mm:ss.SSS} [%thread] %-5level [%X{traceId}] %logger{36} - %msg%n</pattern></encoder>
    </appender>
    <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
      <queueSize>4096</queueSize><discardingThreshold>0</discardingThreshold><neverBlock>false</neverBlock>
      <appender-ref ref="FILE"/>
    </appender>
    <root level="INFO"><appender-ref ref="ASYNC"/></root>
  </springProfile>
</configuration>
```
### 8.2 面试问答
**Q1：异步 Appender 会不会丢日志？** A：会。AsyncAppender 用阻塞队列，队列满时行为由 `neverBlock` 决定：`false`（默认）会**阻塞业务线程**保证不丢，`true` 则直接丢弃新事件；队列使用率超过 `discardingThreshold`（默认队列剩余 20%）时 TRACE/DEBUG/INFO 会被丢弃。结论：宁可阻塞也别丢关键日志时保持 neverBlock=false；追求低延迟就用 neverBlock=true + 打点告警。
**Q2：日志级别怎么调？线上怎么临时开 DEBUG？** A：静态用 `logging.level.com.x=debug`；运行时 POST `/actuator/loggers/com.x` 改级别（`configuredLevel`，传 null 恢复默认）。注意修改是内存级、重启失效，且高 QPS 下开 DEBUG 可能导致磁盘写满——建议按 logger 精确开启而非 root。
**Q3：日志文件怎么滚？关键参数是什么？** A：用 SizeAndTimeBasedRollingPolicy：`fileNamePattern`（含 %d 与 %i）、`maxFileSize`（单文件）、`maxHistory`（保留天数）、`totalSizeCap`（总容量，需大于 maxHistory×maxFileSize 才生效）。**只用 TimeBasedRollingPolicy 时没有单文件大小限制**，单文件可能涨到几十 GB——线上磁盘打满的常见原因。
**Q4：怎么保证一次请求的全链路日志能串起来？** A：① 入口生成 traceId（网关优先，没有则本地生成）并写入 MDC 与响应头；② pattern 带 `%X{traceId}`；③ 跨线程用 TaskDecorator/TransmittableThreadLocal 传递；④ 跨服务用 HTTP header（如 `X-Trace-Id`）或 MQ 消息属性传递；⑤ 采集侧（ELK/Loki）按 traceId 聚合；⑥ finally 清理 MDC，否则线程池复用会出现"别人的 traceId"。
### 8.3 高频追问与陷阱
- 陷阱：缺少 `MDC.clear()` 导致日志串号（尤其 Tomcat 线程池场景）。
- 陷阱：同时存在 `logback.xml` 与 `logback-spring.xml` 时前者会被优先处理，导致 springProfile 不生效。
### 8.4 代码/配置示例
```java
@Component
public class TraceFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse resp, FilterChain chain)
            throws ServletException, IOException {
        String traceId = req.getHeader("X-Trace-Id");
        if (traceId == null) traceId = UUID.randomUUID().toString().replace("-", "");
        MDC.put("traceId", traceId);
        resp.setHeader("X-Trace-Id", traceId);
        try { chain.doFilter(req, resp); } finally { MDC.clear(); }   // 必须清理
    }
}
```
### 8.5 一句话速记
logback-spring.xml + 异步 Appender + 滚动三参数；MDC 放 traceId，finally 必须 clear。

## 9. Boot 2.x 升级 3.x 破坏性变化清单
### 9.1 核心原理

| 变化 | 2.x | 3.x | 应对 |
| --- | --- | --- | --- |
| JDK 基线 | JDK 8/11/17 | **JDK 17+**（3.2+ 支持 21） | 构建与运行环境必须升级 |
| 命名空间 | `javax.*` | **`jakarta.*`**（servlet/annotation/persistence/validation） | 全量替换 import，第三方库换 jakarta 版本 |
| 自动装配文件 | spring.factories | `.imports` | 自定义 starter 必须迁移，否则静默失效 |
| 路径匹配 | AntPathMatcher | **PathPatternParser** | `**` 语义更严格、后缀匹配关闭、尾斜杠匹配默认关闭 |
| Actuator | 部分端点/属性更宽松 | 暴露策略更保守，部分端点与属性重命名 | 重新核对 `management.*` |
| Spring Security | WebSecurityConfigurerAdapter | 组件化 SecurityFilterChain | 重写配置类 |
| 链路追踪 | Spring Cloud Sleuth | Micrometer Tracing | 结合 Cloud 版本升级 |

**升级步骤建议**：① 升 JDK 与构建插件；② 先升到 Boot 2.7 拉平差异（启用 .imports、清理弃用告警）；③ 换 jakarta 命名空间与依赖；④ 升 Boot 3.x；⑤ 处理路径匹配、Actuator、Security；⑥ 用 `spring-boot-properties-migrator` 扫过时属性；⑦ 回归与灰度。
### 9.2 面试问答
**Q1：Boot 2 升 3 最大的坑是什么？** A：命名空间与自动装配文件。`javax→jakarta` 是编译期可见的硬伤（第三方依赖没有 jakarta 版本就卡住）；`.imports` 迁移是**静默失效**——编译通过、启动成功，但自定义 starter 的自动配置不生效，只能靠测试或 `--debug` 发现。
**Q2：路径匹配策略变了会有什么影响？** A：Boot 3 默认 PathPatternParser，AntPathMatcher 只在显式配置时使用。影响：`**` 与 `/*` 语义更严格、后缀模式匹配（`/user.json`）默认关闭、**尾斜杠匹配（`/user/` 与 `/user`）默认不再等价**，带尾斜杠的老客户端会 404，需要显式配置或规范 URL。
**Q3：升级过程中怎么发现被移除的配置项？** A：把 `spring-boot-properties-migrator` 作为 runtime 依赖加入，启动时会打印过时/已移除属性并给出替代项，验证完成后移除该依赖；同时逐条检查启动日志里的弃用警告。
**Q4：Cloud 与 Boot 的版本必须一起升吗？** A：必须。Spring Cloud 是"版本列车"（release train），每个 train 绑定特定的 Boot 版本范围：2021.x（Cloud 3.1）对应 Boot 2.6/2.7，2022.x（Cloud 4.0）对应 Boot 3.0/3.1，2023.x（Cloud 4.1）对应 Boot 3.2/3.3；Spring Cloud Alibaba 同样有对应关系（2022.x 对应 Boot 3）。**版本不对齐会出现 NoSuchMethodError、NoClassDefFoundError 等诡异错误**，详见 [Spring Cloud 篇](./05-SpringCloud微服务体系.md)。
### 9.3 高频追问与陷阱
- 陷阱：只升 Boot 不升 Spring Cloud，Feign/Gateway/服务发现会全部报错。
- 陷阱：jakarta 全局替换时把 JDK 自带的 `javax.sql`、`javax.naming`、`javax.crypto` 也换了导致编译失败——**只有 Java EE/Jakarta EE 相关的 javax 包需要替换**。
### 9.4 代码/配置示例
```xml
<!-- 升级辅助：启动时打印过时属性，验证完成后删除 -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-properties-migrator</artifactId>
  <scope>runtime</scope>
</dependency>
```
### 9.5 一句话速记
Boot 3 = JDK17 + jakarta + .imports + PathPatternParser；升级先到 2.7 拉平差异，再用 migrator 扫属性。

## 10. 高频连环追问
### 10.1 核心原理

| 问题 | 标准结论 |
| --- | --- |
| 怎么排除自动装配？ | ① `@SpringBootApplication(exclude=...)`；② `spring.autoconfigure.exclude`；③ 移除对应依赖 |
| @Configuration 的 proxyBeanMethods 是什么？ | true（默认）用 CGLIB 增强，保证 @Bean 方法间调用返回单例；false 是 lite 模式，启动更快但方法互调会 new 新对象 |
| Bean 覆盖怎么控制？ | `spring.main.allow-bean-definition-overriding`，Boot 2.1+ 默认 false；开启是隐患，优先 @Primary/@ConditionalOnMissingBean |
| @SpringBootApplication 能随便放吗？ | 不能。它决定默认扫描包（所在包及子包）、不能放默认包、应放在根包；放错位置表现为"Bean 找不到" |
| starter 与自动装配的区别？ | starter 是依赖聚合（POM），自动装配是代码（条件装配 Bean） |
### 10.2 面试问答
**Q1：@Configuration(proxyBeanMethods = false) 有什么好处与风险？** A：好处是不生成 CGLIB 子类，减少启动期类生成与内存占用（官方很多自动配置类已改成 false）。风险是 @Bean 方法之间直接调用不再返回同一实例（lite 模式），若依赖"方法调用保证单例"的写法会创建多个对象（典型是多次 new 连接池）。判断标准：@Bean 方法之间没有互相调用就可以安全设为 false。
**Q2：@SpringBootApplication 能不能放在子包或默认包？** A：放子包会导致扫描不到同级其他包（默认只扫本包及子包），Bean 找不到；放默认包（无 package）会被明确拒绝（`Found @SpringBootApplication in the default package`）。规范：主类放根包（如 `com.x`），业务代码放其子包。
**Q3：为什么我的 @Bean 被自动配置"覆盖"了 / 又为什么报 Bean 重复？** A：两种情况：① 类型相同但 beanName 不同——自动配置的 `@ConditionalOnMissingBean` 生效，默认 Bean 不注册，不报错；② **beanName 相同但类型不同**——触发 BeanDefinition 覆盖，`allow-bean-definition-overriding=false` 时启动失败（`Cannot register bean definition ... since there is already ... bound`），开启则后者生效（顺序不可控）。所以自定义 Bean 时要注意 beanName。
**Q4：如何在启动时执行一次性逻辑？** A：`ApplicationRunner`/`CommandLineRunner`（可 @Order 排序）、`ApplicationListener<ApplicationReadyEvent>`、`SmartInitializingSingleton`。区别：前两者在容器完全启动后执行（可接收参数、可注入业务 Bean），后者在单例全部实例化完即执行（更早，但容器未 refresh 完成）。
**Q5：@SpringBootApplication 的 exclude 与 @ComponentScan 的 excludeFilters 区别？** A：前者排除**自动配置类**（由 ImportSelector 引入的），后者排除**组件扫描到的类**；作用对象不同，混用达不到预期效果。
### 10.3 高频追问与陷阱
- 陷阱：忘写 `@ConfigurationPropertiesScan` 或 `@EnableConfigurationProperties` 导致 @ConfigurationProperties 类没被注册。
- 陷阱：在 @SpringBootApplication 上写 `scanBasePackages` 会覆盖默认扫描根包，容易漏扫。
### 10.4 代码/配置示例
```java
@SpringBootApplication(proxyBeanMethods = false,
        exclude = {DataSourceAutoConfiguration.class})   // 排除自动装配
@ConfigurationPropertiesScan
@EnableScheduling
public class App { }
```
```yaml
spring:
  main:
    allow-bean-definition-overriding: false   # 默认值，保持关闭
    banner-mode: off
```
### 10.5 一句话速记
排除用 exclude，覆盖用 @ConditionalOnMissingBean/@Primary，lite 模式只在 @Bean 互不调用时用。

## 自测题（含答案要点）
1. **Spring Boot 解决了什么问题？** 依赖版本对齐、配置自动化（自动装配）、内嵌容器化部署。
2. **启动流程关键步骤顺序？** 构造 → 环境准备 → 建容器 → prepareContext → refresh（内嵌容器启动）→ callRunners → Ready 事件。
3. **内嵌容器在哪一步启动？** refresh 的第 9 步 onRefresh。
4. **自动装配的实现链路？** @EnableAutoConfiguration → AutoConfigurationImportSelector → .imports/spring.factories → 排序过滤 → @Conditional → 注册。
5. **Boot 3 的自动装配文件在哪？** `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`。
6. **为什么自动配置能被用户覆盖？** 默认 Bean 普遍带 @ConditionalOnMissingBean。
7. **@ConditionalOnBean 的坑？** 顺序敏感，需 @AutoConfigureAfter，且应只用于自动配置类。
8. **怎么排查自动装配为何不生效？** `--debug` 的 ConditionEvaluationReport、/actuator/conditions。
9. **starter 与自动装配的区别？** POM 依赖聚合 vs 代码条件装配。
10. **自定义 starter 的必做步骤？** 自动配置类 + Properties + 条件注解 + .imports 文件 + 配置元数据 + 聚合 POM。
11. **配置优先级三句口诀？** 命令行 > 环境变量 > 配置文件；外部 > 内部；profile 专属 > 通用。
12. **@ConfigurationProperties 与 @Value 的区别？** 批量绑定/松散绑定/校验 vs 单值绑定/支持 SpEL。
13. **松散绑定的规则？** 前缀用 kebab-case，字段 camelCase，环境变量用大写下划线。
14. **@PropertySource 能加载 yml 吗？** 不能，只支持 properties。
15. **启动优化手段与懒加载风险？** 减少扫描、排除自动装配、关闭 JMX、CDS、AOT/Native、lite 模式；懒加载会让首请求变慢、错误延后暴露。
16. **Actuator 默认暴露什么？生产怎么收口？** 只有 health 与 info；生产独立管理端口 + 鉴权 + 最小暴露。
17. **优雅停机怎么配？liveness 与 readiness 区别？** graceful + preStop + gracePeriod；liveness 只看进程（挂了重启），readiness 看能否接流量（挂了摘除）。
18. **异步 Appender 会丢日志吗？** 会，取决于 neverBlock 与 discardingThreshold；neverBlock=false 时阻塞而不丢。
19. **升级 Boot 3 的破坏性变化五个关键字？** JDK17、jakarta、.imports、PathPatternParser、Actuator 策略。
20. **proxyBeanMethods=false 的风险？** lite 模式下 @Bean 方法互调会创建多个实例。

## 背诵卡片
1. Spring Boot = 起步依赖 + 自动装配 + 内嵌容器 + 外部化配置。
2. @SpringBootApplication = @SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan。
3. 自动装配核心：ImportSelector 读候选名单 + @Conditional 决定是否生效。
4. Boot ≤2.6 用 spring.factories；2.7 引入 .imports；3.0 只认 .imports。
5. .imports 路径：META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports。
6. 自动配置让位的核心注解：@ConditionalOnMissingBean。
7. 条件家族：OnClass/OnBean/OnProperty/OnResource/OnWebApplication/OnExpression。
8. 顺序控制：@AutoConfigureBefore/After/Order，仅对自动配置类生效。
9. 排查自动配置：`--debug` 或 /actuator/conditions。
10. 排除自动装配：@SpringBootApplication(exclude=) 或 spring.autoconfigure.exclude。
11. starter 命名：官方 spring-boot-starter-xxx，第三方 xxx-spring-boot-starter。
12. starter 两模块：autoconfigure 写代码，starter 聚依赖。
13. 自动配置类不要写 @ComponentScan，避免重复扫描。
14. 配置优先级：命令行 > 环境变量/系统属性 > 配置文件。
15. 配置文件位置：外部 > 内部；config/ 子目录 > 根目录；同一位置 properties > yml。
16. profile 专属文件优先于通用文件。
17. @PropertySource 不支持 yml，且在优先级中很低。
18. 一组配置用 @ConfigurationProperties，单值用 @Value。
19. @ConfigurationProperties 支持松散绑定与 JSR-303 校验；@Value 不支持松散绑定但支持 SpEL。
20. 配置加密：Jasypt / 配置中心加密 / KMS，密钥不能进仓库。
21. 内嵌容器在 onRefresh 启动，端口冲突表现为 refresh 失败。
22. 换容器：排除 spring-boot-starter-tomcat，引入 jetty/undertow starter。
23. 启动优化优先级：减少扫描与自动装配 > CDS > AOT/Native > 懒加载。
24. lazy-initialization=true 会让首请求变慢、错误延后。
25. Actuator 默认只暴露 health 与 info。
26. 生产 Actuator：独立管理端口 + 鉴权 + 最小暴露。
27. liveness 只看进程，readiness 才看依赖与流量。
28. 优雅停机：server.shutdown=graceful + preStop + gracePeriod。
29. 日志用 logback-spring.xml 才能用 springProfile。
30. 异步 Appender 队列满时 neverBlock=false 会阻塞业务线程。
31. 滚动策略三参数：maxFileSize / maxHistory / totalSizeCap。
32. MDC 放 traceId，finally 必须 clear，跨线程要显式传递。
33. 升级 Boot 3：JDK17 + jakarta + .imports + PathPatternParser。
34. jakarta 替换只换 Java EE 相关的 javax（sql/naming/crypto 不能换）。
35. 升级辅助工具：spring-boot-properties-migrator。
36. proxyBeanMethods=false 更快，但 @Bean 互调不再保证单例。
37. allow-bean-definition-overriding 默认 false，不要开。
38. @SpringBootApplication 必须放根包，不能放默认包。
39. 启动后一次性任务：ApplicationRunner/CommandLineRunner 或 ApplicationReadyEvent。
40. @ConfigurationProperties 类需要 @EnableConfigurationProperties 或 @ConfigurationPropertiesScan 才会注册。
