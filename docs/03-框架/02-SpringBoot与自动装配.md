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

### 零基础阅读：先把几个名词放到一张图里

- **Spring Framework** 提供容器、依赖注入、AOP、事务和 Web 等基础能力；它不会自动替每个项目决定“该创建哪些基础设施对象”。
- **Spring Boot** 在 Spring 之上提供约定、依赖版本管理、自动配置、外部化配置和可执行应用打包。Boot 不是另一套 IoC 容器。
- **Bean** 是由 Spring 容器管理的对象；**BeanDefinition** 是创建 Bean 的配方（类型、构造参数、作用域、工厂方法等）；**ApplicationContext** 是管理这些配方、创建对象并发布事件的容器。
- **自动配置** 是一组带条件的普通配置类。它们不是“扫描所有 jar 并猜怎么配置”，而是先发现候选配置，再按 classpath、已有 Bean、属性等条件决定是否导入。
- 看启动问题时要区分三个时刻：候选配置被发现、BeanDefinition 被注册、Bean 实例被创建。它们不是同一件事；懒加载会推迟实例创建，但不代表自动配置条件也全部推迟。

建议阅读顺序：先跟踪一次 `SpringApplication.run`，再理解候选配置与条件，之后用配置绑定和 Starter 把原理落到一个可运行的小功能上。下面的版本说明以 Spring Boot 官方主版本行为为界；具体补丁版本、Spring Framework 版本及第三方 Starter 仍应以项目依赖树为准。

## 1. Spring Boot 定位与启动流程
### 1.1 核心原理
Spring Boot 解决三个问题：**依赖管理（起步依赖版本对齐）、配置自动化（自动装配）、部署简化（内嵌容器打成可执行 jar）**。它不替代 Spring，只是把样板配置收敛成约定。

这里的“起步依赖”通常是 Maven/Gradle 的依赖聚合，不是运行时魔法：例如 `spring-boot-starter-web` 引入 Spring MVC、JSON、内嵌 Servlet 容器等依赖；`spring-boot-dependencies`（或 parent）通过 BOM 统一版本。它不要求所有组件都用同一版本号，而是给出经过兼容性管理的一组版本。直接覆盖 BOM 管理的单个传递依赖，可能造成 `NoSuchMethodError` 等二进制不兼容。

`SpringApplication.run` 不是单纯“new 一个容器”。可以先用一个生活化的比喻理解：**启动 Spring Boot 应用，就像开一家 CRM SaaS 服务中心**。先确认要开什么类型的服务、读取营业配置；再准备管理中心；接着按配置把服务组件装好；最后启动 Web 接待窗口，并做开门前的准备工作。

整个过程先记成三大段：

```text
准备启动方案和配置
    ↓
创建 Spring 容器，并在 refresh() 中装配 Bean、启动 Web 服务器
    ↓
运行启动后任务，确认应用可以接收请求
```

下面逐步说明每一步“做什么、为什么、完成后有什么”。内部方法名用于源码定位；不同 Boot 小版本的细节可能略有差异。

| 阶段 | 通俗解释：Spring Boot 在做什么 | 完成后的结果 |
| --- | --- | --- |
| 1. 构造 `SpringApplication` | 先看项目依赖，判断这是普通程序、Servlet Web 程序还是 WebFlux 程序；收集之后启动时要通知的监听器和初始化器；确定主要配置入口（通常是 `App.class`）。这一步还没有真正创建应用 Bean。 | 拿到一份启动方案：应用类型、主配置入口、启动扩展点。 |
| 2. 启动计时和早期监听器 | 启动秒表，并通知监听器“应用开始启动了”（`ApplicationStartingEvent`）。这时 Spring 容器还没有创建，所以监听器不能假设已经能注入普通业务 Bean。 | 早期扩展代码有机会观察启动开始。 |
| 3. 准备 `Environment` | 建立“配置总账本”，读取命令行参数、环境变量、`application.yml` 等配置，确定启用哪些 profile，并处理配置来源的覆盖优先级。比如 `--server.port=8081` 会影响后续 Web 服务器配置。 | 容器创建前，配置已经准备好，后续 Bean 创建和条件判断可以读取它。 |
| 4. 显示 Banner、准备日志 | 按设置显示启动图案，并初始化/准备日志系统，让后续容器创建过程能输出日志。Banner 只是展示，不负责创建 Bean；可用 `spring.main.banner-mode=off` 关闭。 | 启动过程有日志输出。 |
| 5. 创建 `ApplicationContext` | 根据第 1 步判断的应用类型，创建合适的 Spring 容器。Servlet 项目会用能够管理 Web 服务器的 Context；普通 Java 程序则用非 Web Context。 | 有了管理 Bean 的“容器”，但业务 Bean 还没有全部创建。 |
| 6. 准备容器 `prepareContext` | 把刚才准备好的配置交给容器，运行 `ApplicationContextInitializer`，并把主配置类（`App.class`）登记为配置来源。这里登记的是“稍后按它继续解析”的入口，不等于此刻已经创建所有业务对象。 | 容器知道用什么配置启动，以及从哪里开始解析 Bean。 |
| 7. 刷新容器 `refreshContext` | 这是最重要、工作最多的一步：解析 `@SpringBootApplication`，执行组件扫描和自动配置；把 BeanDefinition（对象说明书）登记好；处理各种容器扩展点；创建非懒加载单例并执行初始化；Web 项目还会在这个阶段创建并启动内嵌 Web 服务器。 | Bean 装配完成，Web 项目已绑定端口并开始监听请求；失败通常会导致启动失败。 |
| 8. `afterRefresh` | 历史上的扩展钩子。Boot 2.4 起已废弃，不要把新逻辑放在这里。 | 一般可以忽略。 |
| 9. 发布 Started 并运行 Runners | 容器刷新成功后，Boot 发布 `ApplicationStartedEvent`，然后依次调用 `ApplicationRunner` / `CommandLineRunner`，给应用一个执行启动后初始化代码的机会。Runner 仍在 Ready 之前运行。 | Runner 代码执行完成，应用才继续进入 Ready 阶段。 |
| 10. 发布 Ready | 全部启动步骤（包括 Runners）成功后，发布 readiness=ACCEPTING_TRAFFIC 和 `ApplicationReadyEvent`，表示应用已准备好接收流量。启动中失败则会发布 `ApplicationFailedEvent`（如果上下文已经创建）。 | 编排平台或调用方可将实例视为可接流量。 |

#### 把表格翻译成一段“人话”

例如 CRM 服务启动时，Spring Boot 先判断“这是一个 Servlet Web 服务”，读取端口、数据库地址和当前环境等配置；再创建 Spring 容器，把 `App` 作为配置入口交给容器；容器解析 `App` 及其扫描到的类，根据条件决定哪些自动配置生效，创建各个 Service、Repository 和其他组件；然后启动内嵌 Tomcat。Tomcat 成功监听端口后，Boot 执行 Runner；所有 Runner 正常完成后，最后发出 Ready 信号。

可以用下面的“结果检查”帮助定位启动失败阶段：

- **还没创建容器就失败**：优先检查应用类型判断、早期监听器或配置文件加载。
- **容器创建后，在 `refreshContext` 失败**：检查配置类解析、Bean 缺失/冲突、依赖注入、初始化异常、端口占用等。
- **日志显示 Runner 阶段失败**：容器和 Web 服务器可能已经起来，但启动任务失败会阻止应用进入 Ready；检查 Runner 做的初始化工作。
- **已经发布 `ApplicationReadyEvent`**：Boot 生命周期认为应用已经就绪；这不代表每个下游依赖永远健康，仍需健康检查和运行期监控。

#### `refresh()` 为什么又是一个很大的步骤？

`SpringApplication.run` 负责“组织启动”；`ApplicationContext.refresh()` 负责“真正把 Spring 容器准备好”。可以把 refresh 想成装修并开门营业的过程：

1. **准备容器状态和配置**：容器进入可刷新状态，并准备好 Environment、资源加载等基础设施。
2. **准备 BeanFactory**：准备管理 BeanDefinition 和单例对象的工厂。
3. **注册容器基础设施**：设置类加载器、表达式解析器、Aware 回调等基础能力。
4. **执行容器子类的准备工作**：Web Context 可在对应扩展点注册 Web 相关作用域等设施。
5. **解析配置、登记 BeanDefinition**：处理 `@Configuration`、`@ComponentScan`、`@Import` 和自动配置。这里主要是在登记“对象配方”，不等于每个对象已经创建。
6. **注册 BeanPostProcessor**：安装后续创建 Bean 时要用的处理器，例如依赖注入处理器、AOP 自动代理创建器。
7–10. **准备消息、事件和 Web 等基础设施**：初始化消息源、事件广播器，执行 Context 类型自己的刷新工作，并登记事件监听器。Servlet Web Context 会在自己的 `onRefresh()` 中创建 WebServer；具体服务器的启动/绑定端口发生在 refresh 流程中。
11. **创建非懒加载单例**：根据 BeanDefinition 实例化对象、注入依赖、执行初始化回调，必要时包装成 AOP 代理；`@Lazy` Bean 通常推迟到首次需要时创建。
12. **完成刷新并通知容器已刷新**：发布 `ContextRefreshedEvent` 等完成信号。

上面是便于建立心智模型的分组，不是让你死背每个版本的源码行号。关键要理解两件事：**前面先登记配方和处理器，之后才批量创建 Bean；Web Server 在容器 refresh 过程中启动，不是等 Runner 全部执行完才启动。**因此端口被占用一般会让 `refreshContext` 失败。

#### `Environment`、`BeanFactory`、`ApplicationContext` 分别是什么？

- `Environment` 像**配置总账本**：保存 profile 和多个有优先级的配置来源。配置值被覆盖时，要查来源和优先级。
- `BeanFactory` 像**对象工厂/仓库管理员**：根据 BeanDefinition 创建 Bean、注入依赖并管理单例。
- `ApplicationContext` 像**完整的应用管理中心**：它包含 BeanFactory，并额外协调事件、资源、国际化、生命周期等能力。

因此“命令行参数明明传了，Bean 读到的值却不对”，常见原因是配置来源优先级、profile 或属性绑定方式，而不一定是 `main(String[] args)` 没把参数传进去。
### 1.2 面试问答
**Q1：Spring Boot 和 Spring 的关系是什么？** A：Boot 是 Spring 的**脚手架与自动装配层**，不是新框架。它做的事：起步依赖（版本对齐）、自动装配（@EnableAutoConfiguration + 条件装配）、内嵌容器（Tomcat/Jetty/Undertow）、外部化配置与 Actuator。Boot 项目里跑的仍是标准 Spring ApplicationContext。
**Q2：说一下 Spring Boot 的启动流程。** A：构造 SpringApplication（判断应用类型、收集启动扩展点、确定主配置入口）→ 发布 Starting → 准备 Environment → 显示 Banner/准备日志 → 创建并准备 ApplicationContext → refresh（解析配置、创建 Bean、启动内嵌 Web 服务器）→ 发布 Started → 执行 Runners → 发布 Ready。加分点：端口绑定发生在 refresh 过程中；Runners 在 Ready 之前执行，阻塞或失败会影响应用进入就绪状态。
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
启动流程 = 判断应用类型 → 准备配置 → 建容器 → refresh（装配 Bean、启动 Web 服务器）→ 发 Started → 执行 Runners → 发 Ready。

## 2. 自动装配原理与 imports 文件演进
### 2.1 核心原理
#### 先用一句话理解：自动配置是“带条件的默认装配方案”

Spring 本身会管理对象，但它不知道每个项目都想用哪个数据库、缓存或 Web 服务器。过去开发者需要自己写很多 `@Configuration` 和 `@Bean`。Spring Boot 把常见的配置方案放进依赖包里：**项目引入了相关依赖、满足相应条件时，Boot 就替你注册一组默认 Bean；你自己提供了 Bean 时，默认方案通常让位。**

例如，项目引入 JDBC 和数据库驱动后，Boot 可能提供 `DataSource` 的默认配置；如果项目没引入 JDBC 相关类，就不需要这套配置；如果你自己声明了 `DataSource`，自动配置一般会退让。自动配置不是“猜测你的业务”，而是依据 classpath、配置属性和容器中已有的 Bean 进行条件判断。

#### `.imports` 是什么？它只是“候选配置类名单”

可以把每个自动配置 jar 想成一个工具箱：jar 里有若干配置类，而 `.imports` 文件是一张告诉 Boot“这个工具箱里有哪些默认装配方案”的清单。

```text
你的应用
├─ spring-boot-starter-jdbc
│  ├─ JDBC 相关类和数据库驱动
│  └─ Spring Boot JDBC 自动配置 jar
│     ├─ DataSourceAutoConfiguration.class
│     └─ META-INF/spring/
│        └─ org.springframework.boot.autoconfigure.AutoConfiguration.imports
│           └─ 清单中写着 DataSourceAutoConfiguration
└─ 你的业务代码
```

`.imports` **不是 Java 的 `import` 语句，不是配置项，也不是 Bean 本身**。它是放在 jar 资源目录中的文本文件，一行写一个自动配置类的全限定名：

```text
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.crm.CrmClientAutoConfiguration
com.example.crm.CrmMetricsAutoConfiguration
```

它的作用只是让 Boot 能找到“有哪些候选配置类”。某个类出现在清单里，不等于该类里的 Bean 一定会创建；还要继续看条件。

#### 从启动入口到 Bean 创建：把整条链路走一遍

`@SpringBootApplication` 是常见启动类上的组合注解，里面包含 `@EnableAutoConfiguration`。后者负责启动自动配置这条流程；组件扫描则负责发现你自己的 `@Component`、`@Service` 等类。它们是相关但不同的两条来源：

```text
@SpringBootApplication
├─ @ComponentScan：找应用自己写的组件（例如 OrderService）
└─ @EnableAutoConfiguration：让 Boot 查找并筛选默认配置方案
```

以一个自动配置 CRM 客户端为例（使用 Boot 2.7+ 的 `@AutoConfiguration` 写法；`CrmClient` 代表某个第三方客户端库中的类）：

```java
@AutoConfiguration
@ConditionalOnClass(CrmClient.class)
@ConditionalOnProperty(prefix = "crm.client", name = "enabled",
        havingValue = "true")
@EnableConfigurationProperties(CrmClientProperties.class)
public class CrmClientAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean(CrmClient.class)
    CrmClient crmClient(CrmClientProperties properties) {
        return new CrmClient(properties.getBaseUrl(), properties.getApiKey());
    }
}

@ConfigurationProperties(prefix = "crm.client")
class CrmClientProperties {
    private String baseUrl;
    private String apiKey;
    // getter/setter 省略
}
```

如果要让 Boot 3 找到这份方案，还要在同一个自动配置 jar 中登记：

```text
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.crm.CrmClientAutoConfiguration
```

启动时可以按下面 6 步理解：

1. **找到“名单”**：`@EnableAutoConfiguration` 通过 `@Import` 引入 Boot 的选择器 `AutoConfigurationImportSelector`。选择器读取 classpath 上自动配置 jar 提供的 `.imports`，收集候选配置类名。这里的 classpath 可以包含应用自身和许多依赖 jar。
2. **剔除不想要的方案**：去掉重复项，也剔除用户明确排除的配置（`exclude`、`excludeName`、`spring.autoconfigure.exclude`）。
3. **按依赖情况初筛**：检查方案依赖的类是否存在。例如没有 `CrmClient` 这个类，CRM 客户端自动配置就不需要继续处理。部分筛选是启动优化，避免分析明显不可能生效的配置。
4. **按顺序处理配置方案**：剩下的自动配置类按声明的先后关系解析。某些条件需要知道其他 Bean 是否已经登记，所以顺序在特定场景下重要；但这不是最终 Bean 对象的创建顺序。
5. **逐项检查条件并登记“配方”**：例如 CRM 客户端类存在、`crm.client.enabled=true`，并且用户还没有提供自己的 `CrmClient`，那么 Spring 注册一个 `CrmClient` 的 BeanDefinition。条件不满足时，这份默认 BeanDefinition 就不登记。
6. **之后才创建对象**：容器在 refresh 后续阶段根据 BeanDefinition 创建非懒加载 Bean，注入属性并执行初始化。也就是说，“名单中有配置类”“条件通过并登记定义”“真正创建 Bean”是三个不同阶段。

上面示例中，条件分别表示：

| 写法 | 用人话说 | 不满足时 |
| --- | --- | --- |
| `@ConditionalOnClass(CrmClient.class)` | 只有客户端库已在依赖中，才考虑这套配置 | 不注册这份配置 |
| `@ConditionalOnProperty(... enabled=true)` | 只有用户明确打开功能，才启用 | 不注册这份配置 |
| `@ConditionalOnMissingBean` | 用户没有自己提供同类型 Bean 时，才给一个默认 Bean | 默认 Bean 让位，不覆盖用户 Bean |

这就是“约定优于配置”的实际含义：常见情况可以直接工作；需要定制时，用户能通过属性、Bean 或排除项控制它。它不是无条件地把所有 jar 里的类全部实例化。

#### 选择器为什么要“延迟”处理？

`AutoConfigurationImportSelector` 实现了 `DeferredImportSelector`。这里的“延迟”是说：**等普通用户配置类解析之后，再处理自动配置候选**；不是等用户第一次调用某个 Bean 时才处理自动配置。

这样设计是为了让用户配置先进入容器的定义信息中，自动配置随后检查“用户是不是已经提供了实现”。例如用户自己写了 `@Bean CrmClient`，自动配置里的 `@ConditionalOnMissingBean(CrmClient.class)` 就能看到这个定义，从而不再登记默认的 `CrmClient`。

#### 排序是配置解析顺序，不是 Bean 实例化顺序

`@AutoConfigureBefore`、`@AutoConfigureAfter`、`@AutoConfigureOrder` 用来安排自动配置类之间的处理顺序。它们常用于条件依赖：例如某个自动配置必须在默认 `DataSource` 配置之后检查是否存在 `DataSource`。

但它们不表示 Bean 对象一定按同样顺序构造。对象何时创建，仍受依赖关系、懒加载、作用域和容器生命周期影响。Bean 之间有真实依赖时，优先把依赖写在 `@Bean` 方法参数中，例如 `@Bean Foo foo(Bar bar)`，让容器明确知道 Foo 需要 Bar；不要只靠配置排序表达对象依赖。

#### `spring.factories` 到 `.imports`：名单文件格式的演进

自动配置的基本思想没有变，变化的是 Boot 从哪里读取“候选配置类名单”：

| Spring Boot 版本 | 自动配置候选名单的位置 | 说明 |
| --- | --- | --- |
| Boot 2.6 及更早 | `META-INF/spring.factories` 中 `EnableAutoConfiguration` 对应的条目 | 一个 properties 风格文件，候选类名写在指定 key 后 |
| Boot 2.7 | 新增 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` | 迁移期兼容两种方式；新项目优先用 `.imports` |
| Boot 3.0 及以后 | `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` | 自动配置候选不再从 `spring.factories` 的 `EnableAutoConfiguration` key 读取 |

旧文件大致长这样：

```properties
# META-INF/spring.factories（Boot 2.x 自动配置候选登记方式）
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.crm.CrmClientAutoConfiguration,\
com.example.crm.CrmMetricsAutoConfiguration
```

新文件则是一行一个类名，没有 key/value：

```text
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.crm.CrmClientAutoConfiguration
com.example.crm.CrmMetricsAutoConfiguration
```

因此升级自定义 Starter 时要检查它实际支持的 Boot 主版本：Boot 2.6 及更早使用旧入口；Boot 2.7 可在迁移期间兼容两种入口；Boot 3.x 的自动配置候选必须放在 `.imports`。这不表示 `spring.factories` 在 Boot 3 的所有扩展机制中都消失了——变化特指自动配置候选的登记方式，其他扩展点要按对应版本文档确认。Boot 2.7 起可使用 `@AutoConfiguration`；更早的 2.x 项目常用 `@Configuration`。

#### 自动配置没生效时，按这条线排查

1. **依赖在不在？** 自动配置 jar 是否真的被应用引入，所需第三方类是否在运行时 classpath。
2. **名单里有没有？** 检查 jar 中 `.imports` 或旧版 `spring.factories` 是否登记了正确的全限定类名。
3. **有没有被排除？** 检查注解的 `exclude`/`excludeName` 和 `spring.autoconfigure.exclude`。
4. **条件通过了吗？** 检查 `@ConditionalOnClass`、`@ConditionalOnProperty`、`@ConditionalOnBean` 等条件；启动时加 `--debug` 查看 ConditionEvaluationReport。
5. **定义登记后，Bean 创建成功了吗？** 如果条件匹配但创建失败，检查依赖注入、构造器、属性绑定和初始化异常；这不是候选名单的问题。

`--debug` 的条件报告能解释“为什么配置类/Bean 没被条件选中”，但不能自动解释所有启动异常；比如依赖版本冲突或 Bean 初始化抛错，需要结合异常栈继续排查。

> **源码进阶（建立直觉后再看）**：选择器读取候选后会去重、合并排除项，并可经 `AutoConfigurationImportFilter` 做早期过滤；最终仍由配置类或 `@Bean` 方法上的条件决定是否登记 BeanDefinition。`@ConditionalOnBean` 等容器条件依赖当时可见的定义信息，因此自动配置排序在这些特定场景中有意义。无需先背 `ImportCandidates.load`、`spring-autoconfigure-metadata.properties` 等内部名词，先记住“找名单 → 看条件 → 登记配方 → 创建对象”。
### 2.2 面试问答
**Q1：Spring Boot 自动装配原理是什么？** A：可以按“找名单 → 检查条件 → 登记 BeanDefinition → 创建对象”回答。`@EnableAutoConfiguration` 引入 `AutoConfigurationImportSelector`，它从依赖 jar 的 `.imports`（旧版 Boot 从 `spring.factories`）读取候选自动配置类；Boot 再检查依赖类是否存在、开关属性是否匹配、用户是否已有同类型 Bean。条件通过才登记默认 BeanDefinition，之后容器 refresh 时才创建对象。用户自己提供实现时，`@ConditionalOnMissingBean` 通常让默认配置退让。
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
自动装配 = 找候选名单 → 检查条件 → 登记对象配方 → 容器创建 Bean；`.imports` 是候选名单，不是 Bean 清单。

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

`@ConditionalOnMissingBean` 的“缺失”不是扫描全世界：它根据当前 `BeanFactory` 中可见的 BeanDefinition/Bean 类型进行判断，且条件评估有阶段与排序约束。把它放在自动配置上、让自动配置排在用户配置之后，是为了让用户定义先可见；不要把它当作运行期动态开关，也不要期望它在应用运行后因 Bean 改动而重新评估。

**条件注解的两个层级**：

- 配置类级别条件不满足时，该配置类中的定义整体跳过。将 `@ConditionalOnClass` 放在自动配置类上可避免缺少可选依赖时解析相关 Bean。
- `@Bean` 方法级别条件只控制该 Bean 定义；同一自动配置中其他 Bean 仍可注册。条件放在什么层级，决定了失败影响范围。
- `@ConditionalOnProperty` 默认要求 key 存在，且若未指定 `havingValue`，通常只把字符串 `"false"` 视为不匹配。它并不检查值是否属于业务上合法的枚举；配置校验仍应由属性类完成。
- `@ConditionalOnBean` 与 `@ConditionalOnMissingBean` 适用于自动配置的默认实现判定。普通业务配置跨配置类顺序依赖时容易出现“有时不满足”，更可靠做法是显式依赖注入或合并配置边界。

**用户 Bean 怎样覆盖默认 Bean**：推荐为默认 Bean 标注 `@ConditionalOnMissingBean(SomeService.class)`，用户声明同类型实现后，默认定义不注册。`@Primary` 解决的是“存在多个候选 Bean 时注入选谁”，并不会阻止默认 Bean 创建；`@Qualifier` 是消费方指定候选项；排除自动配置则是整组撤掉。三者目的不同，不能把 `@Primary` 当成关闭自动配置的手段。
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

**一个可运行设计的边界**：autoconfigure 模块放实现与配置元数据；starter 模块只聚合 autoconfigure 和默认依赖。可选依赖（例如某个日志后端）不应无条件传递给所有使用者。自动配置只依赖自己确实需要的 API，并将实现类放在条件保护范围内；仅仅在 `@Bean` 方法上标类条件但在配置类字段、父类或方法签名中直接引用可选类型，仍可能让类加载失败。

一个最小 Boot 3 自动配置类的结构如下。假设 `LogService` 和 `DefaultLogService` 是 starter 提供的实现，属性对象会由 Boot 注册：

```java
@AutoConfiguration
@ConditionalOnClass(LogService.class)
@EnableConfigurationProperties(LogProperties.class)
@ConditionalOnProperty(prefix = "my.log", name = "enabled",
        havingValue = "true", matchIfMissing = true)
public class LogAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean(LogService.class)
    LogService logService(LogProperties properties) {
        return new DefaultLogService(properties.getLevel());
    }
}
```

配置类应尽量只描述 Bean 关系，不做包扫描、网络连接或静态初始化副作用。自动配置的默认 Bean 通过方法参数表达依赖，方便 Spring 管理生命周期，也便于 `ApplicationContextRunner` 在不启动整个 Web 容器的情况下验证“依赖存在才装配、用户 Bean 优先、属性关闭则不装配”。
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

配置解析的核心模型不是“找到一个 yml 文件就读取”，而是构建有序的 `PropertySource` 列表。Environment 按优先级从前到后解析同名 key，较高优先级来源覆盖较低来源；Actuator 的 `/env` 能辅助观察来源，但暴露配置端点可能泄漏敏感值，必须限制访问。

**ConfigData 与配置文件位置**：Boot 2.4 引入 ConfigData API，替代旧式配置文件加载流程；因此远程配置、导入文件和 profile 激活的行为要按 2.4+ 文档理解。常用位置包括 classpath 根目录、classpath `config/`、当前工作目录、工作目录 `config/` 与其直接子目录。外部位置一般覆盖 jar 内位置；显式指定 `spring.config.location` 会替换默认搜索位置，而 `spring.config.additional-location` 是在默认位置之外追加位置。容器部署时要确认工作目录和挂载路径，否则“文件在机器上”不等于“应用搜索路径里”。

同一位置下 properties 优先于 YAML；不要把它推广为所有文件、所有位置之间的固定优先级。profile 专属配置参与 ConfigData 文档排序，并与激活 profile、导入位置共同决定最终值。`spring.profiles.active/include` 不适合放在带 `spring.config.activate.on-profile` 的条件文档中以激活自身；多环境配置最好用清晰的外部 profile 参数，而不是在多个文件中互相激活。

**配置属性绑定生命周期**：`@ConfigurationProperties` 将 Environment 中一个前缀下的属性绑定到 Java 对象，支持宽松命名匹配、类型转换、嵌套对象、集合与校验。类必须注册（`@EnableConfigurationProperties`、`@ConfigurationPropertiesScan`，或被相应配置机制注册），否则注解本身不会自动创建 Bean。Boot 2.x/3.x 均支持 JavaBean setter 绑定；Boot 2.2 起还支持构造器绑定（当时需 `@ConstructorBinding`），Boot 3 对单构造器类型可推断构造器绑定，具体多构造器场景需看该版本注解要求。不可变类应明确构造器与默认值设计。

`Duration`、`DataSize` 等类型比裸数字更能表达单位，例如 `timeout: 3s`、`max-size: 10MB`。对配置边界加 `@Validated` 和 Bean Validation 约束，启动阶段失败比线上遇到错误 URL/负线程数更容易诊断。密钥不应写入仓库中的默认配置；环境变量只是注入通道，仍需控制运行环境权限、日志脱敏和 Secret/KMS 生命周期。

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

**如何把“启动慢”变成可验证问题**：

1. 先定义测量边界：进程启动、Spring Context refresh、端口就绪、Readiness 变为可接流量，可能是四个不同指标。
2. 用 `BufferingApplicationStartup` 记录 Spring 启动步骤，再通过受保护的 `/actuator/startup` 或启动记录检查耗时 Bean/阶段；端点存在与暴露是两回事。
3. 把长耗时分成类路径扫描、Bean 实例化、外部连接初始化、JVM 类加载/JIT 四类。分别用启动步骤、连接超时日志、JFR 或类加载日志验证，不要看到“Bean 多”就直接启用全局懒加载。
4. 每次只改一个因素并对冷启动、热启动和资源占用复测。移除自动配置前确认业务确实不依赖其提供的 Bean；CDS/AOT/native 则需额外验证镜像构建、反射/代理、监控探针和部署环境。

内嵌服务器的线程池和连接器参数会影响吞吐、排队、内存及文件描述符，不能仅按 CPU 核数照抄。Boot 提供 `WebServerFactoryCustomizer` 可编程定制，优先使用配置属性；线程池上限、队列长度和请求超时应结合数据库连接池容量、下游并发和压测结果共同设置，避免 Web 线程远多于可用 DB 连接导致大量线程阻塞。
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
| /actuator/info | 自定义信息 | 否（端点存在不代表 HTTP 默认暴露） |
| /actuator/metrics、/prometheus | 指标（Micrometer） | 否 |
| /actuator/env、/configprops | 属性来源与配置项 | 否（敏感） |
| /actuator/conditions、/beans、/mappings | 自动配置条件、Bean 列表、请求映射 | 否 |
| /actuator/loggers | 在线调日志级别 | 否 |
| /actuator/threaddump、/heapdump、/startup | 线程/堆快照、启动步骤耗时 | 否 |
| /actuator/shutdown | 关闭应用 | 否（默认关闭，生产慎开） |

**Web 默认暴露端点通常只有 `health`**；`info` 端点即使存在也不等于默认通过 HTTP 暴露。精确行为取决于 Boot 版本、端点是否在 classpath、`management.endpoints.*` 配置和安全配置，应在部署版本上验证。`management.endpoints.web.exposure.include` 控制 HTTP 暴露，暴露不等于无需鉴权。**生产必须收口**：`management.server.port` 单独开管理端口 + 只绑内网 + Spring Security 鉴权 `/actuator/**`。**优雅停机**：`server.shutdown=graceful`（Boot 2.3+，Tomcat/Jetty/Undertow 支持）+ `spring.lifecycle.timeout-per-shutdown-phase=30s`；行为是停止接收新请求 → 等待在途请求完成或超时 → 关闭容器与连接池。K8s 场景还要 `preStop` sleep + 足够 `terminationGracePeriodSeconds`，否则“Endpoint 摘除”与“进程退出”之间的时间差仍会丢请求。

**健康、存活、就绪不是一个状态**：HealthIndicator 回报依赖/子系统状态；liveness 用来判断进程是否应被重启；readiness 用来判断实例能否接收新流量。Boot 2.3+ 的 Availability API 提供 `LivenessState`、`ReadinessState`，Boot 2.3 及之后版本可配置 probes；旧版本需另行实现/确认探针端点。若 liveness 包含数据库或 Redis 探测，下游短暂故障会让所有 Pod 被重启，反而扩大故障。readiness 是否包含依赖则按业务降级能力设计。

Actuator 依赖 `spring-boot-starter-actuator` 才提供相关端点；例如 Metrics 通常还需 Micrometer registry，Prometheus 端点需相应 registry 依赖。`/env`、`/configprops` 可能暴露内部配置，`heapdump` 下载体积大且包含敏感内存数据，`shutdown` 可终止进程。最小暴露、网络隔离和权限控制应同时实施，而不是只换管理端口。
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
Actuator Web 默认通常只暴露 health，生产要独立端口 + 鉴权；优雅停机 = graceful + preStop + 足够 grace period。

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

日志配置示例中的异步队列会在日志线程与业务线程之间建立缓冲：队列长度不是吞吐保证，磁盘/下游 appender 长时间慢于生产速率时队列终会填满。`discardingThreshold=0` 表示不按默认阈值提前丢弃低级别日志，但队列满时仍由 `neverBlock` 等策略决定阻塞或丢弃；要结合磁盘告警、日志采集背压和关机刷盘策略验证。不要把密码、令牌、完整身份证号等字段放进 MDC。

MDC 是线程上下文，不是通用的跨异步边界传播机制。在线程池里提交任务时，父线程的 MDC 通常不会自动复制到工作线程；可使用 Spring `TaskDecorator` 在提交时复制上下文并在 finally 恢复/清理。对 Reactor 等响应式链路，ThreadLocal 也不能假设随信号传播，应使用相应的上下文传播机制。Boot 3 项目若集成 Micrometer Tracing，优先采用标准追踪库生成和传播 trace/span，而非同时维护两套互不关联的 traceId。
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
| JDK 基线 | Boot 2.x 不同次版本要求不同；2.7 最低为 JDK 8 | **JDK 17+**；Boot 3.2 起支持 JDK 21 | 以具体 Boot 小版本兼容范围为准，构建与运行环境一起升级 |
| 命名空间 | `javax.*` | **`jakarta.*`**（servlet/annotation/persistence/validation） | 全量替换 import，第三方库换 jakarta 版本 |
| 自动装配文件 | Boot 2.7 可兼容 spring.factories 与 `.imports` | 自动配置使用 `.imports` | 自定义自动配置候选必须迁移；其它 `spring.factories` 扩展点逐项核对 |
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

**升级不能只做全局替换**：`javax.*` 中既有 Java EE API，也有 JDK 模块包（如 `javax.sql`、`javax.naming`、`javax.crypto`）。只迁移 Servlet、Persistence、Validation 等对应的 Jakarta EE API；依赖树中的第三方库也必须提供 Jakarta 兼容版本。若仍有只支持 `javax.servlet` 的过滤器或库，即使源代码 import 改完，运行期仍可能出现类型不兼容。

## 10. 高频连环追问
### 10.1 核心原理

| 问题 | 标准结论 |
| --- | --- |
| 怎么排除自动装配？ | ① `@SpringBootApplication(exclude=...)`；② `spring.autoconfigure.exclude`；③ 移除对应依赖 |
| @Configuration 的 proxyBeanMethods 是什么？ | 完整 `@Configuration` 默认 true，用 CGLIB 增强，保证容器外直接调用 `@Bean` 方法时仍通过容器获取 Bean；false 是 lite 模式，方法互调不再拦截，可能直接 new 出另一个对象 |
| Bean 覆盖怎么控制？ | `spring.main.allow-bean-definition-overriding`，Boot 2.1+ 默认 false；开启是隐患，优先 @Primary/@ConditionalOnMissingBean |
| @SpringBootApplication 能随便放吗？ | 不能。它决定默认扫描包（所在包及子包）、不能放默认包、应放在根包；放错位置表现为"Bean 找不到" |
| starter 与自动装配的区别？ | starter 是依赖聚合（POM），自动装配是代码（条件装配 Bean） |
### 10.2 面试问答
**Q1：@Configuration(proxyBeanMethods = false) 有什么好处与风险？** A：好处是不生成 CGLIB 子类，减少启动期类生成与内存占用（官方很多自动配置类已改成 false）。风险是 @Bean 方法之间直接调用不再返回同一实例（lite 模式），若依赖"方法调用保证单例"的写法会创建多个对象（典型是多次 new 连接池）。判断标准：@Bean 方法之间没有互相调用就可以安全设为 false。
**Q2：@SpringBootApplication 能不能放在子包或默认包？** A：放子包会导致扫描不到同级其他包（默认只扫本包及子包），Bean 找不到；放默认包（无 package）会被明确拒绝（`Found @SpringBootApplication in the default package`）。规范：主类放根包（如 `com.x`），业务代码放其子包。
**Q3：为什么我的 @Bean 被自动配置"覆盖"了 / 又为什么报 Bean 重复？** A：两种情况：① 类型相同但 beanName 不同——自动配置的 `@ConditionalOnMissingBean` 生效，默认 Bean 不注册，不报错；② **beanName 相同但类型不同**——触发 BeanDefinition 覆盖，`allow-bean-definition-overriding=false` 时启动失败（`Cannot register bean definition ... since there is already ... bound`），开启则后者生效（顺序不可控）。所以自定义 Bean 时要注意 beanName。
**Q4：如何在启动时执行一次性逻辑？** A：`ApplicationRunner`/`CommandLineRunner`（可 @Order 排序）、`ApplicationListener<ApplicationReadyEvent>`、`SmartInitializingSingleton`。Runner 在容器 refresh 完成并发布 `ApplicationStartedEvent` 后、发布 Ready 前执行；Ready 监听器则在 Runner 成功之后执行。`SmartInitializingSingleton` 更早，在非懒加载单例全部实例化后执行，但容器刷新尚未完全结束。
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
2. **启动流程关键步骤顺序？** 判断应用类型 → 准备配置 → 建容器 → refresh（解析配置、创建 Bean、启动 Web 服务器）→ Started 事件 → Runners → Ready 事件。
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
16. **Actuator 默认暴露什么？生产怎么收口？** Web 默认通常只暴露 health；生产独立管理端口 + 鉴权 + 最小暴露。
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
25. Actuator Web 默认通常只暴露 health；info 等端点需按需暴露。
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
