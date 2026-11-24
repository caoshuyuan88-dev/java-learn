# Spring 核心 IoC 与 AOP

> **定位**：讲清 Spring 容器如何创建、装配、增强对象，覆盖容器启动、Bean 生命周期、循环依赖、AOP 代理、声明式事务这几条面试主线。
> **重要度**：★★★★★（Java 后端面试出现频率最高的框架主题，几乎每轮必问）
> **建议复习时长**：6~8 小时（含动手跑循环依赖与事务失效 Demo）
> **关联文档**：[Spring Boot 与自动装配](./02-SpringBoot与自动装配.md)、[Spring MVC 与 Web 容器](./03-SpringMVC与Web容器.md)、[并发编程](../01-Java基础/03-并发编程.md)、[MySQL 事务锁与 MVCC](../02-数据库/02-MySQL事务锁与MVCC.md)、[分布式锁与分布式ID](../05-分布式与架构/03-分布式锁与分布式ID.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| Spring 定位与模块划分 | ★★★☆☆ | 八股原理 |
| IoC 容器与 refresh 流程 | ★★★★☆ | 源码追问 |
| Bean 注册方式（@Import 三用法） | ★★★★☆ | 八股原理 / 配置实践 |
| 依赖注入与 @Autowired/@Resource | ★★★★★ | 八股原理 / 场景题 |
| 循环依赖与三级缓存 | ★★★★★ | 源码追问 / 场景题 |
| Bean 生命周期与作用域 | ★★★★★ | 源码追问 |
| AOP 原理与失效场景 | ★★★★★ | 故障排查 / 手写代码 |
| Spring 事务传播与事务失效 | ★★★★★ | 场景题 / 故障排查 |
| 事件机制与容器扩展点 | ★★★★☆ | 八股原理 / 源码追问 |

## 1. Spring 定位与模块划分
### 1.1 核心原理
Spring 的本质可以先理解为**一个 IoC 容器 + 一套 AOP 基础设施**，其余模块很多是在这两块之上做应用集成。声明式事务、缓存等功能常通过代理实现，因此“代理没生成或调用没经过代理”是常见失效原因之一；但数据库能力、事务属性、异常回滚规则和配置条件也会导致功能不符合预期，不能把所有问题都归结为代理。

#### 从“自己组装对象”理解 IoC

假设 CRM 的订单服务要调用客户服务和订单仓储。没有容器时，`OrderService` 自己 `new` 依赖：

```java
class OrderService {
    private final CustomerService customerService = new CustomerService();
    private final OrderRepository orderRepository = new JdbcOrderRepository();
}
```

这看起来简单，却把“用哪个实现、依赖怎么创建、对象何时销毁”都写死在业务类里。测试时想换成 fake repository、部署时想切换实现，都得改业务代码。依赖关系也可能层层 `new`，创建顺序和资源关闭越来越难管理。

交给 Spring 后，业务类只描述自己需要什么，容器负责装配：

```java
@Service
class OrderService {
    private final CustomerService customerService;
    private final OrderRepository orderRepository;

    OrderService(CustomerService customerService, OrderRepository orderRepository) {
        this.customerService = customerService;
        this.orderRepository = orderRepository;
    }
}
```

启动时容器读取 BeanDefinition（对象的“创建说明书”），根据依赖关系创建并注入对象；业务代码通过容器获得服务，不再负责组织对象图。这就是控制反转（IoC）：**对象创建和依赖装配的控制权从业务代码交给容器**。依赖注入（DI）是 IoC 的一种实现方式，不是另一个容器。

```text
传统方式：OrderService ──new──> JdbcOrderRepository
Spring 方式：BeanDefinition ──> Spring 容器 ──注入──> OrderService
                                     └──────────────> JdbcOrderRepository
```

IoC 不会自动让业务代码更“解耦”：如果类仍然直接依赖具体实现、滥用 `ApplicationContext.getBean()` 做服务定位，耦合依然存在。常规业务优先依赖接口并用构造器注入；只有确实需要动态查找、可选依赖或延迟获取时，才考虑 `ObjectProvider` 等容器能力。

| 模块 | 作用 | 面试关注点 |
| --- | --- | --- |
| spring-core / spring-beans | IoC 最底层：BeanDefinition、BeanFactory | 循环依赖、生命周期 |
| spring-context | ApplicationContext、事件、国际化、@Configuration | refresh() 流程 |
| spring-aop / spring-aspects | 动态代理、@AspectJ 支持 | 代理选择规则 |
| spring-tx | 声明式事务、传播行为 | 传播行为、失效场景 |
| spring-jdbc / spring-orm | JdbcTemplate、事务与 ORM 集成 | 与 MyBatis 整合（见 [MyBatis 篇](./04-MyBatis与ORM实践.md)） |
| spring-web / spring-webmvc | DispatcherServlet、参数解析 | 见 [SpringMVC 篇](./03-SpringMVC与Web容器.md) |

版本要点：**Spring 5.x 基线 JDK 8；Spring 6.x 基线 JDK 17 且全面迁移到 `jakarta.*` 命名空间**；Spring Boot 2.x 对应 Spring 5.x，Spring Boot 3.x 对应 Spring 6.x。
### 1.2 面试问答
**Q1：Spring 是什么？一句话说清核心价值。** A：**轻量级 IoC 容器 + AOP 框架**。IoC 把对象创建与装配的耦合交给容器（解耦 new），AOP 把横切逻辑（事务、日志、限流、缓存）从业务里抽出来（解耦增强）；边界是 Spring 不做持久化与 Web 协议实现，靠集成第三方。
**Q2：Spring、Spring MVC、Spring Boot 什么关系？** A：Spring Framework 是核心；Spring MVC 是它的 Web 层模块（spring-webmvc，提供 DispatcherServlet）；Spring Boot 是脚手架层，本身不提供新功能，只做"起步依赖 + 自动装配 + 内嵌容器"。所以"Boot 项目启动了 Spring 容器"正确，"Boot 是全新框架"错误。
**Q3：为什么说 Spring 轻量？** A：轻量指不依赖重量级应用服务器、业务类无需继承框架基类；**不等于运行时内存小**——ApplicationContext 启动即预实例化全部单例，这才是启动慢、需要懒加载/AOT 优化的原因。
**Q4：Spring 6 / Boot 3 的破坏性变化？** A：JDK 17 基线、`javax.*` → `jakarta.*`（Servlet、Persistence、Annotation）、自动装配配置文件由 spring.factories 改为 `.imports`、默认路径匹配从 AntPathMatcher 变为 PathPatternParser，细节见 [Spring Boot 篇](./02-SpringBoot与自动装配.md)。
### 1.3 高频追问与陷阱
- 陷阱：把 Boot 的"约定优于配置"说成 Spring 特性，会被追问打断。
- 陷阱：Spring 5 与 6 的 `javax/jakarta` 不能共存，混用直接 `ClassNotFoundException: javax.servlet.http.HttpServlet`。
### 1.4 代码/配置示例
```xml
<!-- Boot 3.x 父 POM 已锁定 Spring 6.x，不要手写 spring-core 版本号 -->
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>3.2.5</version>
</parent>
```
### 1.5 一句话速记
Spring = IoC 容器（管对象）+ AOP（管横切）；Boot = 自动装配 + 起步依赖 + 内嵌容器。

## 2. IoC 容器与 refresh 流程
### 2.1 核心原理

| 维度 | BeanFactory | ApplicationContext |
| --- | --- | --- |
| 定位 | 最底层容器接口 | BeanFactory 的增强子接口 |
| 实例化时机 | 默认懒加载（getBean 时才建） | 启动时预实例化全部单例 |
| 事件 / 国际化 / 资源加载 | 不支持 | 支持（Multicaster / MessageSource / ResourcePatternResolver） |
| 后置处理器注册 | 需手动 | 自动（第 5、6 步） |
| Aware 支持 | 仅 BeanName/BeanClassLoader/BeanFactoryAware | 另含 Environment、ApplicationContext 等 |
| 典型实现 | DefaultListableBeanFactory | ClassPathXmlApplicationContext、AnnotationConfigApplicationContext |

**refresh() 十二步**：① prepareRefresh（记录时间、准备 PropertySource）② obtainFreshBeanFactory（创建/获取工厂，XML 在此加载定义）③ prepareBeanFactory（设 ClassLoader/SpEL、注册 ApplicationContextAwareProcessor）④ postProcessBeanFactory（子类定制，Web 作用域在此注册）⑤ **invokeBeanFactoryPostProcessors（ConfigurationClassPostProcessor 在此扫描并注册 BeanDefinition）** ⑥ registerBeanPostProcessors（注册并排序后置处理器）⑦ initMessageSource ⑧ initApplicationEventMulticaster ⑨ onRefresh（Boot 在此启动内嵌容器）⑩ registerListeners ⑪ **finishBeanFactoryInitialization（预实例化全部非懒加载单例）** ⑫ finishRefresh（发布 ContextRefreshedEvent）。

#### 把启动过程串成一条时间线

初学时容易把“扫描到一个类”“创建一个 Bean”“拿到一个代理”当成同一件事。它们是不同阶段：

```text
配置类/组件扫描
       │ 发现类并注册 BeanDefinition（创建说明书）
       ▼
BeanFactoryPostProcessor
       │ 还可以修改或补充说明书
       ▼
BeanPostProcessor 注册完成
       │ 后面创建的 Bean 才能经过完整的前后处理器
       ▼
创建非懒加载单例
       │ 实例化 → 注入依赖 → 初始化回调 → 可能包装为 AOP 代理
       ▼
容器刷新完成，应用可以对外提供服务
```

所以 `@ComponentScan` 并不会在扫描到 `OrderService` 的那一刻就执行它的构造器。扫描通常只把 BeanDefinition 注册进工厂；真正创建发生在后续依赖它、显式 `getBean()` 或第 11 步预实例化时。`@Lazy`、prototype 作用域会改变默认的创建时机/次数，但不会跳过 Bean 创建流程本身。

日常排查可把这几个问题分开：① BeanDefinition 是否存在（是否扫描、是否被条件排除）；② 对象是否已创建（懒加载或作用域）；③ 注入的是目标对象还是代理（是否匹配 AOP 切点、是否从容器获取）。例如 BeanDefinition 存在但启动时报依赖缺失，通常是创建阶段失败，而不是“扫描不到”。

关键因果：**注解扫描在第 5 步，Bean 创建在第 11 步**——所以 BeanFactoryPostProcessor 只能改"图纸"（BeanDefinition），BeanPostProcessor 只能改"产品"（Bean 实例）。BeanDefinition 描述 beanClass、scope、lazyInit、dependsOn、构造参数与属性值、initMethod、primary 等；DefaultListableBeanFactory 内部持有 `beanDefinitionMap`、`beanDefinitionNames`、`singletonObjects`。`BeanDefinitionRegistry` 是注册中心接口，**Spring Boot 2.1+ 默认禁止 BeanDefinition 覆盖**。
### 2.2 面试问答
**Q1：ApplicationContext 和 BeanFactory 最本质的区别？** A：**功能完整度 + 实例化时机**。前者多了事件、国际化、资源加载与后置处理器自动注册，且启动即预实例化单例（尽早暴露配置错误：启动失败优于运行失败）；但 ApplicationContext 内部仍把创建委派给 DefaultListableBeanFactory，不是另一个容器实现。
**Q2：refresh() 里哪两步最容易出问题？** A：第 5 步决定"图纸有没有"（@Component 未被扫描、包路径不对、被 excludeFilters 排除都会缺 Bean）；第 11 步决定"对象能不能造出来"（循环依赖、构造器缺少 Bean、@Value 占位符解析失败、@PostConstruct 抛异常都在此爆）。排查看栈里有没有 `finishBeanFactoryInitialization` → `preInstantiateSingletons`。
**Q3：BeanDefinition 和 Bean 为什么要分开？** A：图纸与产品分离带来三个能力：① 实例化前统一干预（BFPP 批量改配置）；② 单例复用与作用域差异（一份图纸多个实例）；③ 延迟实例化与元数据自省（自动装配条件判断大量读 BeanDefinition）。
**Q4：在 BeanFactoryPostProcessor 里 getBean 会怎样？** A：会让该 Bean 提前实例化，此时 BeanPostProcessor 尚未全部注册，导致它绕过后续增强，日志出现 `is not eligible for getting processed by all BeanPostProcessors`，表现为 @Autowired 注入为 null 或 AOP/事务失效。正确做法是用 ObjectProvider 延迟获取或放到 SmartInitializingSingleton。
### 2.3 高频追问与陷阱
- 追问：prepareBeanFactory 注册了 ApplicationContextAwareProcessor，这就是 EnvironmentAware/ApplicationContextAware 回调的来源。
- 陷阱：注解容器与 XML 容器的加载时机不同（XML 在第 2 步，注解扫描在第 5 步），说反会被抓。
### 2.4 代码/配置示例
```java
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
System.out.println(ctx.getBean(UserService.class));
for (String n : ctx.getBeanFactory().getBeanDefinitionNames()) System.out.println(n); // 诊断
ctx.close(); // 触发销毁回调
```

`AnnotationConfigApplicationContext` 适合不依赖 Web 容器的最小实验。排查时可以先打印 BeanDefinition 名称，再调用 `getBean()` 验证创建；但不要把“启动阶段打印出来的 Bean 名称”误当成所有 Bean 都已经实例化，因为懒加载 Bean 可能还没有对象实例。
### 2.5 一句话速记
refresh 十二步：第 5 步找图纸，第 11 步造产品。

## 3. Bean 注册方式
### 3.1 核心原理

| 方式 | 写法 | 适用场景 | 特点 |
| --- | --- | --- | --- |
| XML | `<bean id class>` | 遗留系统、三方类 | 显式可读但冗长 |
| 组件扫描 | @Component/@Service/@Repository/@Controller | 自研业务类 | 最常用，配合 @Autowired |
| @Bean | @Configuration 中的方法 | 三方类、复杂初始化 | 方法名即 beanName，支持 initMethod |
| @Import | @Import(X.class) | 框架/Starter 批量注册 | 三种用法见下 |
| 手动注册 | registry.registerBeanDefinition | 动态/条件注册 | 最底层，配合 ImportBeanDefinitionRegistrar |
| SPI 扩展 | spring.factories / .imports | Starter 自动装配 | 见 [Spring Boot 篇](./02-SpringBoot与自动装配.md) |

**@Import 三种用法**：① 导入普通类——即使没有 @Component 也会注册成 Bean；② 导入 `ImportSelector` 实现——返回全限定类名数组，实现类自身不成为 Bean（Boot 的 `AutoConfigurationImportSelector` 即此类，且用 `DeferredImportSelector` 保证晚于用户配置）；③ 导入 `ImportBeanDefinitionRegistrar` 实现——拿到 BeanDefinitionRegistry 手动注册，可注册带参数或多例定义（MyBatis 的 `MapperScannerRegistrar` 即此类）。

#### 三种注册方式各自适合什么“变化点”

```java
@Configuration
@Import({AuditService.class, FeatureSelector.class})
class FeatureConfig {}

class FeatureSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        // 根据导入方注解元数据选择要导入的配置类。
        return new String[] { "com.example.feature.DefaultFeatureConfiguration" };
    }
}
```

普通类导入适合“我明确知道要注册哪个类”；`ImportSelector` 适合“根据注解/环境选择配置类”；`ImportBeanDefinitionRegistrar` 则适合需要扫描一批接口或精确控制 BeanDefinition 属性的框架集成。它们都是在配置类解析期间参与 BeanDefinition 注册，不是应用每次请求时动态创建对象的机制。实际开发通常优先用组件扫描和 `@Bean`，只有做 Starter、框架扩展或批量注册代理类时才需要后两种扩展点。

注意：`ImportSelector` 返回的类名必须能被加载；返回类名本身不会让 Selector 实现类变成业务 Bean。若导入逻辑依赖某个属性或类路径条件，更推荐复用 Spring Boot 条件装配机制，避免在选择器中手工读取环境并形成难以测试的分支。
### 3.2 面试问答
**Q1：@Component 和 @Bean 的区别？** A：@Component 是类级自动扫描，适合自研类；@Bean 是方法级显式声明，适合三方类和需要自定义初始化逻辑的场景。边界：@Bean 必须写在被容器管理的类中，且 `@Configuration` 类中的 @Bean 会经 CGLIB 增强（full 模式）保证方法间调用返回同一单例，而写在 `@Component` 类中是 lite 模式，`a()` 里直接调 `b()` 会产生两个实例。
**Q2：@Import 三种用法分别解决什么问题？** A：普通类解决"注册无注解的配置类"；ImportSelector 解决"按条件批量注册类"；ImportBeanDefinitionRegistrar 解决"注册时还要改 BeanDefinition 或做扫描"。记忆：前两者注册类，后者能拿到 registry 做任意事。
**Q3：Bean 名是怎么生成的？** A：@Component 用 AnnotationBeanNameGenerator：简单类名首字母小写（`UserService`→`userService`；前两字母均大写如 `URLService` 保持原样）；@Bean 用方法名。可显式指定 `@Component("x")`/`@Bean("x")`。同名冲突靠 @Primary/@Qualifier 解决，**打开 allow-bean-definition-overriding 是隐患，不推荐生产使用**。
**Q4：接口上标 @Component 有效吗？** A：无效。注解不会被接口实现继承（只有 @Inherited 的类注解在类继承链上才继承），扫描的是类文件上的元数据。同理 @Transactional 建议标在实现类而非接口上。
### 3.3 高频追问与陷阱
- 追问：ImportSelector 在第 5 步解析 @Configuration 时被调用；实现 DeferredImportSelector 可保证延后处理（自动装配用它保证用户配置优先）。
- 陷阱：@ComponentScan 一旦显式书写会覆盖默认行为，basePackages 默认取声明类所在包。
### 3.4 代码/配置示例
```java
@Configuration
public class RpcConfig {
    @Bean(initMethod = "init", destroyMethod = "close")
    public RpcClient rpcClient(@Value("${rpc.addr}") String addr) { return new RpcClient(addr); }
}
public class MySelector implements ImportSelector {   // 自己不成为 Bean
    @Override public String[] selectImports(AnnotationMetadata m) { return new String[]{"com.x.LogService"}; }
}
```
### 3.5 一句话速记
自己写的类用 @Component，别人的类用 @Bean，框架批量注册用 @Import 三兄弟。

## 4. 依赖注入
### 4.1 核心原理

| 维度 | 构造器注入 | setter 注入 | 字段注入 |
| --- | --- | --- | --- |
| 不可变性 | 可 final | 否 | 否 |
| 空指针风险 | 无 | 有 | 有（脱离容器即 null） |
| 能否解决循环依赖 | 不能（启动报错） | 能 | 能 |
| 单测友好度 | 高（直接 new） | 中 | 低（需反射） |
| 官方推荐度 | 必须依赖推荐 | 可选依赖可用 | 除测试外不推荐 |

| 注解 | 来源 | 匹配顺序 | 备注 |
| --- | --- | --- | --- |
| @Autowired | Spring | 先 byType，候选多个时按字段名/参数名当 beanName 回退 | 支持 required=false、@Primary、@Qualifier |
| @Resource | JSR-250（Spring 6 为 jakarta.annotation） | **先 byName，再 byType** | 无 required 属性 |
| @Inject | JSR-330 | 同 @Autowired（byType 优先） | 需额外依赖 |
| @Qualifier | Spring | 按 beanName 限定 | 与 @Autowired 搭配 |

#### byType 和 byName 到底是什么意思？

这两个词说的是**容器用什么线索找到要注入的 Bean**：

- **byType（按类型找）**：看注入位置声明的 Java 类型，例如 `PaymentGateway`。容器会找出所有“是 `PaymentGateway` 或其子类型”的 Bean 候选。它比较的是 Java 类型/接口实现关系，不是比较类名字符串。
- **byName（按名称找）**：看 Bean 在 Spring 容器里的名字（beanName），按名字精确找到 Bean。Bean 名称通常来自 `@Component("walletGateway")` 里的显式名字，或 `@Bean` 方法名；组件扫描未显式命名时，通常由类名生成，例如 `WalletGateway` → `walletGateway`。

例如容器里有两个都实现 `PaymentGateway` 的 Bean：

```text
beanName="cardGateway"   → CardGateway   → PaymentGateway
beanName="walletGateway" → WalletGateway → PaymentGateway
```

```java
@Autowired
private PaymentGateway gateway;
```

这里先 **byType**：容器发现 `cardGateway` 和 `walletGateway` 都符合 `PaymentGateway`，因此仅凭类型无法唯一决定。若只有一个候选，直接注入；若有多个，Spring 再根据 `@Qualifier`、`@Primary` 等规则缩小范围；仍有多个时，字段名/构造器参数名有时可作为候选名称匹配线索，否则启动时报 `NoUniqueBeanDefinitionException`。

```java
@Autowired
@Qualifier("walletGateway")
private PaymentGateway gateway;
```

这仍然是“先按类型找候选”，再用 `@Qualifier` 把范围限定为名为 `walletGateway` 的 Bean。`@Qualifier` 的值是 Bean 名称/限定符，不是 Java 类型名。

而显式按名称指定可以这样写：

```java
@Resource(name = "walletGateway")
private PaymentGateway gateway;
```

`@Resource(name = "...")` 会优先按这个名字找；默认不写 `name` 时，Spring 通常先用字段名或 setter 属性名作为 Bean 名称查找，再按其规则回退到类型匹配。可记成：**byType 问“你是什么类型”，byName 问“你的容器名字是什么”**。名称匹配要求 Bean 名称一致，类型匹配要求类型可赋值；两者是不同线索，不能把 `PaymentGateway` 这个类型名和 `walletGateway` 这个 Bean 名称混为一谈。

byType 冲突时的解析顺序：**@Qualifier 指定 → @Primary 优先 → 字段名/参数名匹配 beanName**，都不满足抛 `NoUniqueBeanDefinitionException`。

#### 一个注入决策的实际例子

```java
interface PaymentGateway {
    Receipt charge(BigDecimal amount);
}

@Component("cardGateway")
class CardGateway implements PaymentGateway {
    public Receipt charge(BigDecimal amount) { return new Receipt(); }
}

@Component("walletGateway")
class WalletGateway implements PaymentGateway {
    public Receipt charge(BigDecimal amount) { return new Receipt(); }
}

@Service
class CheckoutService {
    private final PaymentGateway gateway;

    CheckoutService(@Qualifier("walletGateway") PaymentGateway gateway) {
        this.gateway = gateway;
    }
}
```

容器先收集所有 `PaymentGateway` 类型的候选，再由 `@Qualifier` 限定为 `walletGateway`。如果业务需要根据订单运行时选择支付渠道，不要试图靠 `@Primary` 动态切换：注入 `Map<String, PaymentGateway>` 或一个策略注册表，在业务方法里根据渠道标识选择实现。`@Primary` 只解决容器创建对象时的默认候选，不理解业务请求。

构造器注入的边界也要讲清楚：构造参数是创建对象的必需条件；可选依赖可以通过 `ObjectProvider<T>` 表达延迟/可选查找，或者用 setter。不要因为字段注入短几行代码就把必须依赖隐藏起来；对象离开 Spring 容器后，构造器创建的实例也更容易在单元测试里正确工作。
### 4.2 面试问答
**Q1：为什么官方推荐构造器注入？** A：**保证依赖不可变、非空、完整**。字段可 final 防篡改；对象创建完即处于可用状态（不会出现半成品 Bean）；依赖过多时构造器参数变长，强迫拆分职责；单元测试直接 new 不依赖容器；能在启动期暴露循环依赖而非运行时出错。边界：可选依赖与框架对象（ApplicationContext）用 setter/字段更自然。
**Q2：@Autowired 与 @Resource 的区别与选型？** A：区别是匹配顺序：@Autowired byType 优先，候选多个时回退按名字；@Resource byName 优先，再 byType。示例：两个 `UserDao` 实现、字段名 `userDaoMysql` 时 @Resource 直接按名命中，@Autowired 先得两个候选再按字段名回退——语义与报错时机不同。选型：Spring 生态内用 @Autowired（能力更强），追求标准或跨框架移植用 @Resource。
**Q3：一个接口有多个实现，怎么注入指定实现？** A：四种：① 字段名与 beanName 一致（隐式）；② `@Qualifier("name")`；③ 实现类加 `@Primary`；④ 注入 `Map<String, XxxService>` 或 `List<XxxService>` 做策略路由（推荐，key 即 beanName，新增实现零改动）。
**Q4：@Autowired 什么时候可以省略？注入 static 字段行不行？** A：类**只有一个构造器**时（Spring 4.3+）可省略；有多个构造器必须显式标注，否则用无参构造器，没有无参构造器则报错。static 字段不能注入，需用 setter 中转或实现 ApplicationContextAware 手动赋值。
### 4.3 高频追问与陷阱
- 追问：@Autowired 还支持 Collection、Optional、ObjectProvider、@Lazy；ObjectProvider 可延迟获取以规避部分循环依赖。
- 陷阱：工具类里 @Autowired 注入的 Service 永远是 null（对象不是容器创建的），这是"静态方法里用不了 Bean"的根因。
### 4.4 代码/配置示例
```java
@Service
public class OrderService {
    private final UserService userService;      // 必须依赖：构造器 + final
    private SmsSender smsSender;                // 可选依赖：setter
    public OrderService(UserService u) { this.userService = u; }
    @Autowired(required = false) public void setSmsSender(SmsSender s) { this.smsSender = s; }
}
@Service  // 策略路由：key 为 beanName
public class PayService {
    private final Map<String, PayChannel> channels;
    public PayService(Map<String, PayChannel> c) { this.channels = c; }
    public void pay(String ch, BigDecimal amt) { channels.get(ch).pay(amt); }
}
```
### 4.5 一句话速记
必须依赖用构造器 + final，可选依赖用 setter，多实现用 @Qualifier/@Primary/Map 路由。

## 5. 循环依赖与三级缓存
### 5.1 核心原理

先把最关键的话记住：**这不是三份 Bean，也不是三份业务数据，而是 Spring 在单例创建期间暂存同一个 Bean 的不同阶段引用。**三级缓存里放的东西类型不一样：

| 缓存 | 实际存的内容 | 可以把它理解成 |
| --- | --- | --- |
| 一级 `singletonObjects` | `beanName → 完整初始化完成的单例对象`（如果有 AOP，通常是最终代理） | 可正式交给业务使用的成品 |
| 二级 `earlySingletonObjects` | `beanName → 已经取出来的早期引用`（可能是原始对象，也可能是提前生成的代理） | 成品还没完成时，环上的另一个 Bean 暂时拿到的引用 |
| 三级 `singletonFactories` | `beanName → ObjectFactory`，也就是一个“按需取得早期引用”的工厂/回调；**不是 Bean 对象本身** | 还没取货时先存的一张“需要时再生成早期引用”的凭条 |

`ObjectFactory` 可以理解为一段可调用的创建逻辑：真正有人需要这个早期引用时，Spring 才调用它；如果 Bean 需要 AOP，它还会让相关后置处理器有机会返回代理。**工厂被调用后，得到的早期引用会从三级移到二级；Bean 完成初始化后，最终对象进入一级，二级和三级中的临时项被清理。**

循环依赖只是一种“触发取早期引用”的场景。在允许循环引用的配置下，创建单例时 Spring 会先准备好暴露早期引用的能力；如果没有其他 Bean 在它完成前索要它，这个工厂不会被调用，Bean 完成后直接作为成品进入一级缓存。Spring Boot 2.6+ 默认不允许循环引用，因此默认配置下这套提前暴露流程不会用来解决循环依赖。

#### 先理解 Spring 创建一个 Bean 的先后顺序

“三级缓存为什么有用”要从对象创建顺序开始看。对一个普通单例，Spring 大致按以下顺序工作：

```text
1. 实例化：先调用构造器，得到原始对象 A
2. 提前暴露入口：把“需要时如何取得 A 的早期引用”放入三级缓存
3. 属性填充：给 A 注入 B；这一步可能触发 B 的创建
4. 初始化：执行 A 的初始化回调
5. 初始化后处理：需要 AOP 时，通常在这里把原始 A 包装成代理 A'
6. 完成注册：把最终可供容器使用的对象放入一级缓存
```

重点是：**构造器执行后，A 已经是一个真实的 Java 对象，但它还没有完成属性注入和初始化。** 这是 setter/字段循环依赖可以被打破的窗口；构造器循环依赖则没有这个窗口，因为创建 A 的第一步就必须先拿到 B。

#### 如果只有“一级 + 二级”，直接把原始对象放进二级会发生什么？

先看一个没有 AOP 的 A、B 循环：A 依赖 B，B 依赖 A。A 构造完成后把原始对象放入二级；创建 B 时就能从二级拿到 A，循环确实可以打通。因此，**“两级缓存一定不能解决循环依赖”是不准确的**。

麻烦出现在 A 还需要 AOP 增强时。例如 A 的 `save()` 有事务：

```text
容器最终应该交付：代理 A' ──拦截事务──> 原始对象 A
B 在循环依赖时拿到：原始对象 A
```

如果二级缓存直接放原始 A，B 注入的就是原始对象。之后 A 初始化完成，Spring 再创建事务代理 A' 并把 A' 放入一级缓存，就会变成：

```text
B.a ───────────────> 原始对象 A（调用 save() 绕过事务代理）
容器 getBean("A") ──> 代理对象 A'（调用 save() 才经过事务拦截）
```

同一个 Bean 在应用中出现了两个身份：一部分地方拿到原始对象，另一部分地方拿到代理。结果可能是 B 调 A 时事务不生效，还会引发 Spring 检测到“注入的是 raw version，最终却被包装”的异常。

#### 三级缓存如何处理这个问题？

三级缓存先不放原始 A，也不立即创建代理，而放一个 `ObjectFactory`。它表达的是：“如果创建过程中的别的 Bean 现在就需要 A，请调用我，我再决定返回早期引用。”只有 B 确实在 A 完成前请求 A 时，Spring 才调用这个工厂，并通过 `getEarlyBeanReference` 让支持早期代理的后置处理器（例如自动代理创建器）有机会提前返回代理 A'：

```text
A 构造完成
  └─ 三级缓存：A → ObjectFactory（尚未调用）

B 注入 A，发现 A 仍在创建
  └─ 调用 ObjectFactory
       ├─ 不需要 AOP：返回原始 A
       └─ 需要 AOP：返回早期代理 A'
            ↓
       把这个已生成的早期引用放进二级缓存，之后重复请求复用它

A 完成初始化
  └─ 最终进入一级缓存的对象必须与此前暴露的早期引用保持一致
```

这解释了三级和二级各自的工作：**三级是“还没取早期引用时的工厂”；二级是“工厂已经执行、可复用的早期引用”；一级是“创建完成后的正式单例”。** 三级工厂把早期代理的创建推迟到确实有人需要它时；二级则避免同一个早期引用被重复创建、造成对象身份不一致。

#### 所以“两级到底行不行”？

答案是：**两级缓存的概念设计可以实现循环依赖；Spring 采用三级，是为了把“早期引用工厂”和“工厂产出的早期对象”分开表示，并兼容 AOP 代理的早期暴露。** 如果只保留一级、二级，而二级直接存对象，会遇到上面的原始对象/代理身份问题。理论上也可以让二级先存工厂、调用后再把工厂替换成早期对象，但这就要在同一个缓存中区分“工厂状态”和“对象状态”；只是把三级表达的两种职责合并了，处理逻辑更复杂，并没有消除这两个阶段。

因此，不要死记“物理上绝对必须三张 Map”。更准确的理解是：**Spring 当前实现用三张缓存清楚表示三个阶段；仅有一个存成品、一个直接存半成品的朴素两级方案，不足以正确处理早期 AOP 代理。** 源码入口可看 `AbstractAutowireCapableBeanFactory#doCreateBean` 中的 `addSingletonFactory`，以及 `DefaultSingletonBeanRegistry#getSingleton` 中依次查一级、二级、三级的逻辑。

#### 用 A、B 两个对象逐步看缓存里究竟有什么

假设 A、B 都是**单例**，使用 setter/字段注入，并且 `A 依赖 B、B 又依赖 A`。为了看清缓存，下面只列 A、B 这两个名字对应的条目；实际过程中还有其他 Bean：

| 时刻 | 一级：成品对象 | 二级：早期引用 | 三级：早期引用工厂 |
| --- | --- | --- | --- |
| ① 开始创建 A，尚无 A 实例 | 无 | 无 | 无 |
| ② A 的原始对象已构造，正在注入属性 | 无 | 无 | `A → ObjectFactory（可按需返回 A 的早期引用）` |
| ③ A 等待 B，开始创建 B；B 也已构造 | 无 | 无 | `A → 工厂`；`B → 工厂` |
| ④ B 注入 A，发现 A 仍在创建，于是调用 A 的工厂 | 无 | `A → A 的早期引用` | `B → 工厂`（A 的工厂已被调用并移除） |
| ⑤ B 注入完成、初始化完成 | `B → 完整的 B` | `A → A 的早期引用` | 无 |
| ⑥ A 注入 B 并完成初始化 | `A → 完整的 A`；`B → 完整的 B` | 无 | 无 |

把第 ④ 步单独拆开看：B 想注入 A 时，Spring 会先查一级（A 还没完成，查不到），再查二级（A 还没被提前取出，查不到），最后才调用三级里的工厂。工厂返回一个可注入的 A 引用，Spring 把它放进二级，B 得以完成创建。A 随后拿到已经创建好的 B，完成自己的初始化。

```text
第 ④ 步之前：singletonFactories["A"] = 一个 ObjectFactory
                    │ 调用工厂
                    ▼
第 ④ 步之后：earlySingletonObjects["A"] = A 的早期引用
                    │ A 最终初始化完成
                    ▼
第 ⑥ 步之后：singletonObjects["A"] = A 的最终对象
              earlySingletonObjects 和 singletonFactories 不再保存 A
```

所以三个缓存不是在保存三个独立的 A：三级保存“怎么取得早期 A”的工厂；二级保存“已经取得的早期 A”；一级保存“创建完成的 A”。没有 AOP 时，早期引用通常就是原始 A 对象本身；有 AOP 时，早期引用可能是代理，Spring 要保证 B 持有的引用与容器最终提供的引用身份一致，避免 B 用原始对象绕过代理、而其他地方拿到代理的情况。

这里的“早期”不代表可以安全地在初始化未完成时任意调用 A 的业务方法。它只让依赖图能先完成引用连接；在初始化回调期间调用未准备好的依赖，仍可能观察到未初始化状态。三级缓存也不是通用循环依赖解决器：它针对 Spring 单例创建过程中的特定循环依赖，不会解决业务设计上的互相调用，也不适用于 prototype。

最可靠的处理通常是消除环：例如把 A、B 共同依赖的规则抽到 C，或将反向同步调用改成事件/应用服务编排。`@Lazy` 只是把依赖换成延迟解析的代理；它可以打断启动时的创建环，但如果首次调用时仍形成递归调用环，运行时问题仍然存在。

**构造器注入为什么无解？** 构造器要求实例化阶段就拿到依赖，而 `addSingletonFactory` 在实例化之后才执行，A 当时连对象都没有，无法提前暴露，只能抛 `BeanCurrentlyInCreationException`。**@Lazy 与 @Async 的坑**：@Lazy 加在注入点会注入代理，真实对象在首次调用时才从容器获取，从而打破启动期循环（代价是问题延后暴露）；`@Async` Bean 在初始化后被包成代理，若参与循环依赖会报 `Bean with name 'x' has been injected into other beans [y] in its raw version as part of a circular reference, but has eventually been wrapped`——B 拿到原始版本、容器里是代理版本，身份不一致。**Spring Boot 2.6 起 `spring.main.allow-circular-references` 默认 false**，循环依赖直接启动失败。
### 5.2 面试问答
**Q1：Spring 如何解决循环依赖？** A：仅对允许循环引用的单例 setter/字段注入，Spring 在构造原始对象后、填充属性前，把 `ObjectFactory` 放到三级缓存；环上另一方确实需要它时才调用工厂，取得早期引用（可能是代理），再完成装配。构造器注入和 prototype 循环依赖不能靠这套机制解决；Boot 2.6+ 默认禁止循环引用。
**Q2：为什么需要三级缓存，两级不行吗？** A：构造器后、属性注入前，Spring 已有原始对象但还没完成初始化；三级先存 `ObjectFactory`，只有发生循环依赖时才调用它获取早期引用（必要时是早期 AOP 代理），并把结果放入二级缓存复用。若朴素两级方案直接把原始对象放二级，后续 AOP 可能再生成代理，导致循环依赖方持有原始对象、容器持有代理对象。理论上两级也可用“二级先放工厂、调用后再替换为对象”的状态设计实现，但那只是把三级的两种职责合并；Spring 用三张缓存明确区分工厂、早期引用和成品。
**Q3：prototype 作用域的循环依赖能解决吗？** A：不能。prototype 每次 getBean 都新建对象，没有共享的半成品可暴露，报 `BeanCurrentlyInCreationException: Requested bean is currently in creation: Is there an unresolvable circular reference?`。
**Q4：循环依赖有什么危害？线上报错怎么排查？** A：危害：掩盖设计问题（两个类强耦合）、AOP 场景下可能注入未增强对象导致事务失效、启动顺序敏感、与 @Async/@Transactional 组合易出隐性 bug。排查：异常会打印完整环（`The dependencies of some of the beans in the application context form a cycle: orderService ┐ ↓ userService ↑ ...`），据此定位两个 Bean；优先重构（抽第三层服务、用事件解耦、把反向调用改成查询），应急可给注入点加 @Lazy 或临时开 `spring.main.allow-circular-references=true`。
### 5.3 高频追问与陷阱
- 陷阱：说"两级绝对不可能解决循环依赖"——朴素两级方案可以打通无 AOP 的 setter 环；三级设计还要正确处理早期代理和对象身份一致性。
- 陷阱：说"三级缓存存的都是对象"——第三级存的是 ObjectFactory。
- 陷阱：以为 @Lazy 解决了循环依赖——它只是把报错推迟到首次调用；真正的解法是重构。
### 5.4 代码/配置示例
```java
@Service public class A { public A(B b) {} }   // 构造器互相依赖：启动即失败
@Service public class B { public B(A a) {} }
@Service public class A2 { public A2(@Lazy B b) {} }  // @Lazy 注入代理打破环
```
```yaml
spring:
  main:
    allow-circular-references: true   # Boot >= 2.6 应急放开，不推荐长期开启
```
### 5.5 一句话速记
三级缓存存早期引用工厂；发生循环时才调用工厂，把早期引用放入二级；初始化完成后最终单例进入一级。适用于允许的单例 setter/字段循环依赖。

## 6. Bean 生命周期与作用域
### 6.1 核心原理

| 阶段 | 回调 | 说明 |
| --- | --- | --- |
| 1 实例化 | 构造器 / 工厂方法 | createBeanInstance，选择构造器 |
| 2 属性填充 | @Autowired / @Value / setter | populateBean，可能触发依赖 Bean 创建 |
| 3 Aware（基础） | BeanNameAware → BeanClassLoaderAware → BeanFactoryAware | invokeAwareMethods 直接调用 |
| 4 BP#before | ApplicationContextAwareProcessor 处理 Environment/ResourceLoader/ApplicationEventPublisher/MessageSource/ApplicationContextAware | 按 PriorityOrdered → Ordered → 无序 |
| 5 初始化 | @PostConstruct → InitializingBean#afterPropertiesSet → init-method | 顺序固定 |
| 6 BP#after | **常规路径下 AOP 代理通常在此生成**（AbstractAutoProxyCreator） | 事务/日志增强生效点；循环依赖时可能提前暴露代理 |
| 7 使用 | getBean 返回（可能是代理），单例放一级缓存 | 供应期 |
| 8 销毁 | @PreDestroy → DisposableBean#destroy → destroy-method | 仅容器 close 时；prototype 不管理 |

#### 把创建、初始化与代理放在一起看

```text
构造对象
  → 注入属性
  → Aware 回调（让对象知道 BeanName、BeanFactory 等容器信息）
  → BeanPostProcessor.beforeInitialization
  → 初始化回调（@PostConstruct → afterPropertiesSet → init-method）
  → BeanPostProcessor.afterInitialization
       └─ 自动代理创建器可在这里返回代理，而不是原对象
  → 缓存/交付最终对象
```

### `BeanPostProcessor.beforeInitialization` 和 `afterInitialization` 做什么？

这里的 **before/after 是相对于“初始化回调”来说的**，不是“构造对象之前/之后”，也不是“处理 HTTP 请求之前/之后”。对象已经构造好、依赖也已经注入后，Spring 才调用已注册的 `BeanPostProcessor`：

```text
构造对象
  → 注入属性（@Autowired / @Value）
  → Aware 回调
  → postProcessBeforeInitialization(bean)   ← 初始化回调之前
  → @PostConstruct
  → InitializingBean.afterPropertiesSet()
  → 自定义 init-method
  → postProcessAfterInitialization(bean)    ← 初始化回调之后
  → 最终对象交给容器
```

容器会按顺序调用已注册的处理器。每个处理器接收当前 Bean，并返回一个对象：通常原样返回；也可以返回包装后的对象。某个处理器返回的新对象会成为后续处理器看到的对象。因此，**BeanPostProcessor 是给 Spring 容器扩展“创建 Bean 时如何处理对象”的钩子**。

#### before：初始化回调开始前处理 Bean

`postProcessBeforeInitialization` 接收的 Bean 已经完成构造和依赖注入，但还没有执行 `@PostConstruct`、`afterPropertiesSet()` 和自定义 `init-method`。Spring 的注解生命周期支持会利用这一阶段执行 `@PostConstruct`；`ApplicationContextAwareProcessor` 也在初始化前处理 `ApplicationContextAware`、`EnvironmentAware` 等回调。

自定义 before 处理器可用于在初始化回调前检查或调整 Bean。但这不是做 `@Autowired` 的阶段：`@Autowired`、`@Value` 的属性注入主要发生在更早的属性填充阶段。常规业务一般只使用 `@PostConstruct` 做初始化，不需要自己写 `BeanPostProcessor`。

#### after：初始化回调结束后处理 Bean

`postProcessAfterInitialization` 接收的 Bean 已经执行完初始化回调。Spring AOP 的自动代理创建器通常在这一阶段检查它是否匹配切点；如果匹配，就返回代理对象，否则返回原 Bean。容器最终提供的可能是这个代理，因此调用事务方法时才能先经过代理拦截。

自定义 after 处理器也可以包装对象，例如在对象外面加一层代理或装饰器；但返回值会影响之后所有注入点拿到的对象，写错可能造成类型不匹配、代理丢失或依赖不一致，应当谨慎。

一个只观察顺序、不替换 Bean 的处理器示意如下：

```java
@Component
class LifecycleTraceProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (beanName.equals("orderService")) {
            System.out.println("before init callbacks: " + beanName);
        }
        return bean; // 不改变 Bean
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (beanName.equals("orderService")) {
            System.out.println("after init callbacks: " + beanName);
        }
        return bean; // 不改变 Bean
    }
}
```

如果 `orderService` 还实现了 `InitializingBean` 并有 `@PostConstruct`，日志顺序是：`before init callbacks` → `@PostConstruct` → `afterPropertiesSet` → `after init callbacks`。这里“before/after”指的就是中间这些初始化回调。

`BeanPostProcessor` 的 before/after 是处理扩展点，不是“每个 Bean 都必定换成新对象”：多数处理器会原样返回对象，自动代理创建器仅在匹配切点时包装。`@Autowired` 的解析主要由 `AutowiredAnnotationBeanPostProcessor` 在属性填充阶段完成；不要把它和初始化前的 `BeanPostProcessor` 回调混成同一步。

生命周期代码示例放在 6.4 节。特别注意：自定义 `init-method` / `destroy-method` 必须通过 `@Bean` 或 BeanDefinition 显式告诉容器；只在类里定义同名普通方法，Spring 不会自动调用。

#### Aware 是做什么的？

`Aware` 可以理解为 Spring 提供的一组“容器主动告知 Bean 必要信息”的回调接口。普通业务对象通常只需要通过构造器拿到业务依赖；但某些基础设施 Bean 在运行时确实需要知道“我在容器中的名字是什么”“当前容器是什么”“应该用哪个 ClassLoader”等信息。这时实现对应的 `*Aware` 接口，Spring 创建 Bean 时会调用接口方法，把信息传给它。

常见接口可以这样理解：

| 接口 | Spring 告诉 Bean 的信息 | 少数合理用途 |
| --- | --- | --- |
| `BeanNameAware` | 这个 Bean 在容器中的名字 | 通用基础设施需要按自身 Bean 名称注册或输出诊断信息 |
| `BeanClassLoaderAware` | Spring 用来加载 Bean 类的 ClassLoader | 框架需要动态加载类、检查类是否存在 |
| `BeanFactoryAware` | 创建它的 `BeanFactory` | 框架级组件需要按需查找 Bean 或检查容器元数据 |
| `ApplicationContextAware` | 当前 `ApplicationContext` | 与容器事件、资源或应用上下文集成的基础设施组件 |
| `EnvironmentAware` | 当前 `Environment`（配置属性与 profile） | 基础设施需要读取环境配置 |

例如：

```java
@Component
class StartupDiagnostic implements BeanNameAware, BeanFactoryAware {
    private String beanName;
    private BeanFactory beanFactory;

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    @PostConstruct
    void report() {
        System.out.println("我的 Bean 名称是：" + beanName);
        System.out.println("容器中是否有 orderService："
                + beanFactory.containsBean("orderService"));
    }
}
```

这里不是 Bean 自己去调用 `setBeanName` 或 `setBeanFactory`；而是 Spring 在生命周期的 Aware 阶段识别它实现了哪些接口，并主动回调，把对应对象传进来。核心的 `BeanNameAware`、`BeanClassLoaderAware`、`BeanFactoryAware` 由 Bean 创建流程直接处理；`ApplicationContextAware`、`EnvironmentAware` 等上下文类接口通常由 `ApplicationContextAwareProcessor` 在初始化前处理。因此生命周期表中的“基础 Aware”和后续 BeanPostProcessor 阶段会相邻出现，但并不是所有 Aware 都在同一个内部调用点处理。

**什么时候用？** 写 Spring 扩展、Starter、通用基础设施，且确实需要容器级信息时可以用。普通业务 Service 不要为了方便到处实现 `ApplicationContextAware` 再 `getBean()`：这会让类主动依赖 Spring 容器，隐藏真实依赖关系，测试困难，也容易形成 Service Locator（服务定位器）模式。业务 Bean 通常用构造器注入明确声明依赖；`Aware` 是少量框架集成场景的补充，不是另一种日常依赖注入写法。

| 作用域 | 说明 | 典型用途 | 注意 |
| --- | --- | --- | --- |
| singleton | 容器内唯一（默认） | 绝大多数 Bean | 必须线程安全 |
| prototype | 每次 getBean 新建 | 有状态对象 | 容器不管销毁 |
| request | 每请求一个 | traceId 持有 | 需 Web 环境 |
| session | 每会话一个 | 登录信息 | 分布式需 Session 共享 |
| application | 每 ServletContext 一个 | 应用级缓存 | — |
| websocket | 每 WebSocket 会话一个 | 长连接上下文 | Spring 4.0+ |
### 6.2 面试问答
**Q1：完整描述 Bean 生命周期。** A：实例化 → 属性填充 → Aware 回调 → BeanPostProcessor#before → （@PostConstruct → afterPropertiesSet → init-method）→ BeanPostProcessor#after（生成代理）→ 使用 → 销毁（@PreDestroy → destroy → destroy-method）。追问点三处：AOP 在 after；@PostConstruct 早于 afterPropertiesSet；prototype 没有销毁回调。
**Q2：@PostConstruct、InitializingBean、init-method 的区别与顺序？** A：顺序 @PostConstruct → afterPropertiesSet → init-method。@PostConstruct 是标准注解不侵入业务；InitializingBean 与 Spring 强耦合、方法名固定；init-method 适合改不了源码的三方类。优先 @PostConstruct（Spring 6 需 `jakarta.annotation-api`）。
**Q3：单例 Bean 线程安全吗？** A：**不保证**——容器只保证实例唯一，多线程共享同一实例，任何可变成员变量都是竞态点。方案：① 无状态设计（Service 不存请求级数据）；② 状态放方法局部变量；③ 必须共享则用 ThreadLocal（finally remove，线程池下会泄漏）、不可变对象、并发容器/原子类；④ 有状态对象改为 prototype 或 request 作用域。典型坑：单例里放 `SimpleDateFormat`（改用 DateTimeFormatter）。
**Q4：prototype Bean 的销毁回调为什么无效？** A：Spring 只为单例（及自定义作用域）注册销毁回调（`registerDisposableBeanIfNecessary`），prototype 实例交付后容器不再持有引用，无法跟踪。需要清理就让调用方 try-finally 处理。
### 6.3 高频追问与陷阱
- 追问：@PreDestroy 先于 DisposableBean#destroy，最后才是 destroy-method（由 DisposableBeanAdapter 统一编排）。
- 陷阱：BeanPostProcessor 自身在第 6 步实例化，因此不要在 BP 里注入业务 Bean（会导致业务 Bean 提前实例化并绕过 BP）。
### 6.4 代码/配置示例
```java
class LifeCycleDemo implements InitializingBean, DisposableBean, BeanNameAware {
    @PostConstruct
    public void postConstruct() { System.out.println("@PostConstruct"); }

    @Override
    public void afterPropertiesSet() { System.out.println("afterPropertiesSet"); }

    public void initialize() { System.out.println("custom init-method"); }

    @PreDestroy
    public void preDestroy() { System.out.println("@PreDestroy"); }

    @Override
    public void destroy() { System.out.println("DisposableBean.destroy"); }

    public void close() { System.out.println("custom destroy-method"); }

    @Override
    public void setBeanName(String name) { System.out.println("BeanNameAware: " + name); }
}

@Configuration
class LifeCycleConfiguration {
    @Bean(initMethod = "initialize", destroyMethod = "close")
    LifeCycleDemo lifeCycleDemo() {
        return new LifeCycleDemo();
    }
}
```

初始化阶段的输出顺序是 `BeanNameAware` → `@PostConstruct` → `afterPropertiesSet` → `initialize`；容器正常关闭时依次执行 `@PreDestroy` → `DisposableBean.destroy` → `close`。实际项目中需要相应的 `jakarta.annotation`（Spring 6）或 `javax.annotation`（Spring 5）注解依赖。`prototype` Bean 不由容器自动执行销毁回调。
### 6.5 一句话速记
实例化→填充→Aware→before→三板斧→after（常规路径生成 AOP 代理）→使用→销毁；单例不保证线程安全。

## 7. AOP 原理与失效场景
### 7.1 核心原理

| 通知 | 注解 | 时机 | 能力 |
| --- | --- | --- | --- |
| 前置 | @Before | 目标方法前 | 不能改返回值，抛异常可阻断 |
| 返回 | @AfterReturning | 正常返回后 | 可读取返回值（需 returning 参数）；不能替换最终返回值 |
| 异常 | @AfterThrowing | 抛异常后 | 可读取异常；不能替换异常，包装异常需用 @Around |
| 后置 | @After | finally 语义 | 不能阻断，总执行 |
| 环绕 | @Around | 前后包裹 | 可阻断、改参数、改返回值（最强） |

执行顺序：@Around 前半 → @Before → 目标 → @AfterReturning/@AfterThrowing → @After → @Around 后半。切点表达式：`execution(* com.x.service..*.*(..))`（最常用）、`@annotation(...)`、`within`、`this/target/args`、`@within`、`bean(...)`。

#### “通知责任链”是什么意思？

先拆开词理解：**通知（Advice）**是切面要做的事情，例如记录日志、开启事务、检查权限；**链（Chain）**表示一次方法调用可能匹配多个通知，它们会按顺序组成一条调用链；**责任**表示每一环负责自己的工作，并决定是否继续调用下一环。Spring 会把匹配到的 Advice 适配成方法拦截器，代理调用时依次执行这些拦截器。

例如一次 `orderService.createOrder()` 同时需要日志和事务，概念上可以看作：

```text
调用方
  → 日志拦截器（记录开始时间）
    → 事务拦截器（开启事务）
      → 目标方法 createOrder()
    ← 事务拦截器（提交；若异常则回滚）
  ← 日志拦截器（记录耗时）
调用方拿到返回值或异常
```

进入时从外层向内层执行；目标方法结束后再反向返回。因此外层日志能观察整个事务拦截过程，事务拦截器则负责在目标方法正常返回时提交、异常时按规则回滚。具体谁在外层由匹配到的 Advisor 顺序决定（例如 `@Order`），不能只凭“日志一定先于事务”来断言。

可以用下面的伪代码理解 `proceed()`：它不是“直接调用目标方法”的同义词，而是“继续执行链上的下一环”；所有拦截器都放行后，才调用目标方法。

```java
Object invoke(int index) throws Throwable {
    if (index == interceptors.size()) {
        return target.createOrder(); // 链走完，调用真正的目标方法
    }

    Interceptor current = interceptors.get(index);
    return current.invoke(() -> invoke(index + 1)); // 执行当前环，再决定是否继续
}
```

`@Around` 最能体现这种结构：调用 `pjp.proceed()` 就是放行到链的下一步；不调用它可以阻止目标方法执行；调用前后还可以记录信息、处理结果或异常。`@Before`、`@AfterReturning` 等则是由 Spring 适配到拦截器机制中的不同通知类型，不需要业务代码手写整条责任链。

| 维度 | JDK 动态代理 | CGLIB |
| --- | --- | --- |
| 原理 | 基于接口生成实现类（Proxy + InvocationHandler） | 基于继承生成子类（ASM） |
| 前提 | 目标类必须有接口 | 类与方法不能 final/private/static |
| 性能 | 创建快，调用略慢（JDK 8+ 差距极小） | 创建慢，调用快 |
| 代理类型 | 接口类型（强转原类失败） | 子类类型（可强转原类） |
| Spring 选择 | proxyTargetClass=false 且有接口 | proxyTargetClass=true 或无接口 |

**Spring Boot 2.x/3.x 默认 `spring.aop.proxy-target-class=true`，即默认 CGLIB**；Spring Framework 原生 `@EnableAspectJAutoProxy` 默认 proxyTargetClass=false。Boot 选 CGLIB 的原因：避免"按实现类类型注入却找不到 Bean"，并统一处理无接口的类。**常见失效场景**：同类内部调用（未走代理，解法是拆类/自注入代理/AopContext.currentProxy 需 exposeProxy=true）、private/static/final 方法、final 类、对象不是容器创建的、切点写错或类未被扫描、BFPP 中提前 getBean。事务还可能因为异常被 catch 后正常返回而提交；非 public 方法是否可代理取决于代理方式与 Spring 版本（业务方法建议用 public）。

#### 代理调用到底经过什么

```text
调用方持有的引用
       │ 调用 proxy.createOrder(...)
       ▼
Spring 生成的代理
       │ 找到匹配的 Advisor（切点 + 通知）
       ▼
拦截器链：事务 / 自定义切面 / 其他拦截器
       │ 执行目标方法
       ▼
返回结果或抛出异常，再沿拦截器链返回
```

如果调用方拿到的是目标对象而不是容器提供的代理，就没有入口执行这条拦截器链。更隐蔽的情况是同类内部调用：外部调用 `proxy.a()` 会进入代理，但 `a()` 里面的 `this.b()` 是目标对象直接调用自己，不会再次经过代理，因此 `b()` 上的事务、缓存、异步等代理增强都可能不生效。

```java
@Service
class OrderService {
    public void submit() {
        writeAudit(); // this.writeAudit()，内部调用，不经过代理
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void writeAudit() {
        // 期望独立事务，但从 submit() 这样调用时不会触发事务拦截器
    }
}
```

优先把独立事务操作拆到另一个 Spring Bean，通过注入的代理调用。自注入代理或 `AopContext.currentProxy()` 是需要明确取舍的备选方案，不应作为默认架构。调试时可用 `AopUtils.isAopProxy(bean)`、`AopUtils.getTargetClass(bean)` 检查代理；不要依赖代理生成类的具体类名。

**注意返回通知的能力边界**：`@AfterReturning` 可以通过 `returning` 参数观察返回结果，但它不能像 `@Around` 一样替换方法最终返回值。确实需要包装或改写返回结果时，应在环绕通知中返回新值；也要保留目标方法的异常语义，避免切面无意中吞掉异常。

代理限制需要按代理方式和 Spring 版本看待：JDK 动态代理面向接口方法；CGLIB 通过子类覆盖方法，因此 final 类无法被子类化，final/private/static 方法也不能按普通可覆盖实例方法拦截。Spring Framework 6 对类代理下部分非 public 实例方法的支持有所扩展，不能笼统断言所有非 public 方法都必定失效；跨版本、跨代理方式时，业务事务方法仍建议声明为 public，并通过代理入口调用，行为最清晰。
### 7.2 面试问答
**Q1：Spring AOP 的实现原理？** A：**运行时动态代理 + 通知责任链**。Bean 初始化后由 AbstractAutoProxyCreator（一个 BeanPostProcessor）判断是否匹配切点，匹配则生成 JDK 或 CGLIB 代理，调用时由代理沿拦截器链执行通知再调用目标方法。边界：Spring AOP 只支持方法级切点，仅对容器管理的 Bean 生效；字段/构造器切点与编译期织入需要 AspectJ。
**Q2：JDK 动态代理和 CGLIB 怎么选？** A：规则：有接口且 proxyTargetClass=false → JDK；否则 CGLIB；**Boot 默认 proxyTargetClass=true → CGLIB**。要点：CGLIB 不能代理 final 类、final 方法、private 方法；JDK 代理只能按接口强转，按具体实现类注入会失败。
**Q3：同一个类里 A 调 B，B 上的 @Transactional/@Cacheable 为什么失效？** A：因为调用发生在目标对象内部（this 引用），未经过代理。解法：① 把 B 拆到另一个 Bean（最佳）；② 自注入 `@Autowired @Lazy private XxxService self` 后调 `self.b()`；③ `AopContext.currentProxy()`（需 `@EnableAspectJAutoProxy(exposeProxy = true)`）；④ 改用 TransactionTemplate 编程式事务。
**Q4：什么时候必须用 @Around？多个切面顺序怎么定？** A：需要"改返回值/改参数/决定是否执行目标方法/统计成功失败"时必须用 @Around（@Before/@After 无法阻断或改返回）。顺序由 @Order/Ordered 决定，值越小越靠外（先进入、后退出）；Spring 事务 Advisor 默认使用 `LOWEST_PRECEDENCE`，但应用可配置顺序，因此自定义切面与事务的内外层应以实际 Advisor 顺序为准。若自定义切面在事务外层，它的退出逻辑会在事务提交/回滚之后执行。
### 7.3 高频追问与陷阱
- 陷阱：@Aspect 类必须自己也是 Bean（加 @Component 或被扫描），否则通知不生效。
- 陷阱：代理后 `getClass()` 是 `Xxx$$SpringCGLIB$$0`，反射取注解要用 `AopUtils.getTargetClass()` 或 AnnotationUtils；Spring 6 的 CGLIB 类名格式有变化，不要写死。
### 7.4 代码/配置示例
```java
@Aspect @Component
public class CostAspect {
    @Around("@annotation(com.x.anno.Cost)")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        long s = System.nanoTime();
        try { return pjp.proceed(); }
        finally { log.info("{} cost={}ms", pjp.getSignature().toShortString(), (System.nanoTime()-s)/1_000_000); }
    }
}
```
### 7.5 一句话速记
AOP = 动态代理 + 通知责任链，代理在 BP#after 生成；自调用、private/final/static、未经过代理的调用都会使增强失效。

## 8. Spring 事务
### 8.1 核心原理

| 传播行为 | 含义 | 典型场景 |
| --- | --- | --- |
| REQUIRED（默认） | 有则加入，无则新建 | 绝大多数业务方法 |
| REQUIRES_NEW | 总是新建并挂起外层 | 操作日志（主事务回滚也要留痕） |
| SUPPORTS | 有则加入，无则非事务执行 | 查询方法 |
| NOT_SUPPORTED | 非事务执行并挂起外层 | 调用不支持事务的长耗时操作 |
| MANDATORY | 必须在事务中，否则抛异常 | 强制上层开事务 |
| NEVER | 必须不在事务中，否则抛异常 | 禁止事务的场景 |
| NESTED | 同一事务内用保存点 | 子流程失败不影响主流程 |

拦截链路：代理 → TransactionInterceptor → 解析事务属性 → 通过 `PlatformTransactionManager` 开启事务（连接绑定 ThreadLocal）→ 执行目标方法 → commit / rollback → 清理。隔离级别 ISOLATION_DEFAULT 表示用数据库默认（MySQL 为 REPEATABLE READ），**Spring 只是把设置传给 DB，实际行为取决于数据库**。回滚规则：默认只回滚 `RuntimeException` 与 `Error`，**checked 异常不回滚**，需 `rollbackFor = Exception.class`。readOnly 是提示性优化（主从路由时价值更大），timeout 依赖驱动实现、不够可靠。

#### 一次声明式事务在运行时的完整过程

```text
调用方
  → 事务代理检查方法上的事务属性
  → PlatformTransactionManager 获取/创建事务
  → JDBC Connection 绑定到当前线程
  → 执行业务方法（DAO 从当前事务上下文取得连接）
  → 正常返回：提交；符合回滚规则的异常：回滚
  → 解绑并归还连接
```

`@Transactional` 本身不会把方法“变成数据库事务”；它提供元数据，事务拦截器必须接到一次经过 Spring 代理的调用，才会按元数据调用事务管理器。线程绑定意味着在方法里新开线程、提交到线程池或 `CompletableFuture` 后，新的执行线程不会自动继承原事务。一个事务管理器通常只管理它对应的数据源；同时写两个独立数据源不会因为一个注解就天然原子提交。

传播行为可以用“调用方已有事务吗”和“被调方法是否要独立提交”来理解：

```text
外层事务 T1 ──调用 REQUIRED──> 加入 T1，任一处标记回滚都会影响整个 T1
外层事务 T1 ──调用 REQUIRES_NEW──> 挂起 T1，开启 T2；T2 提交后再恢复 T1
外层事务 T1 ──调用 NESTED──> 在 T1 中建立保存点；子流程回滚到保存点，外层仍可继续
```

`REQUIRES_NEW` 的“独立”也意味着额外资源成本：外层事务通常仍占着一个连接，内层还需要另一个连接；如果连接池容量太小，线程可能等待连接，严重时造成连接池饥饿。它也不适合用来“保证日志一定写入”而不考虑失败处理：内层事务失败是否影响外层，仍取决于异常如何传播和业务如何定义。

默认回滚规则常造成“异常抛出来了，数据却提交了”的误解。比如下面抛出 checked 异常时，Spring 默认不会仅因这个异常回滚：

```java
@Transactional
public void importOrder() throws IOException {
    orderMapper.insert(...);
    throw new IOException("文件格式错误"); // checked Exception，默认不触发回滚
}

@Transactional(rollbackFor = Exception.class)
public void importOrderSafely() throws IOException {
    orderMapper.insert(...);
    throw new IOException("文件格式错误"); // rollbackFor 配置后回滚
}
```

如果业务代码在事务方法内部 `catch` 异常并正常返回，代理看到的是“方法正常结束”，默认就会提交；需要回滚时应继续抛出符合规则的异常，或明确标记 rollback-only。异常也不能只看类型：数据库可能已经报错并将事务标记为 rollback-only，最终提交仍可能抛 `UnexpectedRollbackException`。

**事务失效清单**：① 同类自调用；② 调用未经过代理；③ 类/方法 final 或 static 导致所选代理方式无法拦截；④ Spring 6 之前的类代理通常要求 public 方法（业务方法建议 public）；⑤ 类未被容器管理；⑥ 异常被 catch 后正常返回；⑦ checked 异常未配 rollbackFor；⑧ 数据库引擎不支持事务（MyISAM）；⑨ 多数据源未指定事务管理器；⑩ 传播行为不当（NOT_SUPPORTED/SUPPORTS 而外层无事务）；⑪ 异步/新线程中操作 DB（事务绑在 ThreadLocal，子线程不在同一事务）；⑫ @Transactional 标在接口上且代理模式不匹配（推荐标实现类）。

**大事务危害与拆分**：长事务长时间持锁（InnoDB 锁在提交/回滚才释放）→ 锁等待与死锁、undo log 膨胀、主从延迟放大、连接池耗尽、回滚代价高。拆法：把 RPC/IO 移出事务；只包 DB 写；用 REQUIRES_NEW 隔离非关键写；非核心逻辑异步化（用 afterCommit 触发）；批量操作分批提交。
### 8.2 面试问答
**Q1：@Transactional 的实现原理？** A：AOP 代理 + TransactionInterceptor + PlatformTransactionManager。代理拦截调用后按 `TransactionAttributeSource` 解析属性，通过 TM 开启事务并把连接绑定到 ThreadLocal，正常返回提交、异常按规则回滚。**因此一切"事务失效"本质是代理没生效或属性没解析到**。
**Q2：REQUIRED 与 REQUIRES_NEW 的区别，举例？** A：REQUIRED 有则加入（与调用方同生共死），REQUIRES_NEW 挂起外层另起独立事务（独立提交/回滚）。例：下单后写操作日志，日志必须留存就用 REQUIRES_NEW，否则会随下单回滚一起消失。代价：多占一个数据库连接（高并发易打满连接池），且外层锁未释放时内层取同一行锁可能死锁。
**Q3：为什么 checked 异常不回滚？** A：Spring 沿用 EJB 约定：RuntimeException 视为不可恢复的系统错误（应回滚），checked Exception 视为可预期业务分支（默认不回滚）。改法 `@Transactional(rollbackFor = Exception.class)`；反向可用 noRollbackFor。
**Q4：NESTED 是怎么实现的？** A：通过 `Connection.setSavepoint()` 建立保存点，子事务回滚 = 回滚到保存点且不提交，外层提交时一起提交；**外层回滚会连带回滚内层**（同一物理事务）。限制：需要数据库与驱动支持保存点（MySQL InnoDB 支持），JTA 场景可能不支持。
**Q5：事务提交后再发消息/清缓存怎么做？** A：用 `TransactionSynchronizationManager.registerSynchronization` 的 `afterCommit`，或 `@TransactionalEventListener(phase = AFTER_COMMIT)`。**不要**在事务内直接发 MQ 或清缓存——事务未提交时下游读不到新数据，回滚后消息也无法撤回（详见[分布式事务](../05-分布式与架构/02-分布式事务.md)）。
### 8.3 高频追问与陷阱
- 追问：捕获异常后可用 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` 手动标记回滚。
- 陷阱：REQUIRES_NEW 内层需要新连接，连接池只有 1 个连接时会死锁（外层占着连接等内层）。
- 陷阱：@Transactional 与 @Async 标注在同一方法上极易出问题（异步线程没有事务上下文），不要混用。
### 8.4 代码/配置示例
```java
@Transactional(rollbackFor = Exception.class, timeout = 3)
public void createOrder(Order order) {
    orderMapper.insert(order);                 // 事务只包 DB 写
    logService.record(order);                  // REQUIRES_NEW：主事务回滚也留痕
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
        @Override public void afterCommit() { mq.send(order.getId()); }   // 提交后再发消息
    });
}
@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
public void record(Order o) { logMapper.insert(o); }
```
### 8.5 一句话速记
@Transactional = 代理 + TransactionInterceptor；默认只回滚 RuntimeException/Error；未经过代理、异常被吞、checked 异常和跨线程是常见失效原因。

## 9. 事件机制与容器扩展点
### 9.1 核心原理
组件：ApplicationEvent（事件）、ApplicationEventPublisher（发布者）、ApplicationEventMulticaster（默认 `SimpleApplicationEventMulticaster`，**同步**）、ApplicationListener / @EventListener。同步广播意味着监听器异常会向发布者传播、可能把事务带崩；`TransactionPhase` 有 BEFORE_COMMIT、AFTER_COMMIT、AFTER_ROLLBACK、AFTER_COMPLETION。

#### 事件解决的是“通知解耦”，不是可靠消息

```text
订单服务完成状态变更
  └─ publishEvent(OrderPaid)
       ├─ 库存监听器
       ├─ 积分监听器
       └─ 缓存失效监听器
```

发布者只依赖事件，不需要直接依赖所有监听器；监听器可独立添加。但 Spring 进程内事件默认同步、进程内传递，不具备跨进程投递、持久化、失败重试或消费位点。若事件用于“下单后必须通知支付/库存”等跨服务可靠业务，不要把 `ApplicationEvent` 当 MQ；可使用消息中间件，并在需要与数据库事务可靠衔接时考虑 Outbox 等模式。

`@TransactionalEventListener` 的阶段决定事件和事务的关系。常用的 `AFTER_COMMIT` 能避免事务回滚后仍执行监听逻辑，但它仍然不是可靠投递保证：进程可能在数据库提交后、监听器执行前宕机。若监听器执行失败，原数据库事务也已经提交，无法靠它回滚。强可靠场景需要可重试/可补偿的持久化消息方案。

默认同步监听器运行在发布事件的线程，因此执行很慢会拖慢发布者，异常也可能沿调用栈返回。改成 `@Async` 后，发布者不再等待监听器完成，但必须考虑线程池容量、异常记录、重试、上下文传递，以及事务上下文不会自动传播到异步线程。

| 扩展点 | 时机 | 能做什么 | 典型用途 |
| --- | --- | --- | --- |
| BeanDefinitionRegistryPostProcessor | 第 5 步最早 | 继续注册 BeanDefinition | 注解扫描、Mapper 注册 |
| BeanFactoryPostProcessor | 第 5 步 | 改 BeanDefinition（属性、作用域） | 占位符解析、属性覆盖 |
| BeanPostProcessor | 每个 Bean 初始化前后 | 包装/替换实例 | AOP 代理、@PostConstruct、@Autowired |
| ApplicationContextInitializer | refresh 之前 | 定制 ConfigurableApplicationContext | 注册 PropertySource |
| ApplicationListener / @EventListener | 事件发布时 | 响应容器与自定义事件 | 启动预热、提交后动作 |
| SmartInitializingSingleton | 全部单例实例化完 | 需要"所有 Bean 就绪"的逻辑 | 缓存预热 |
| SmartLifecycle / Lifecycle | 容器启停 | 管理启停顺序 | 连接池、消费者、定时任务 |
| CommandLineRunner / ApplicationRunner | refresh 完成后 | 启动后一次性任务 | 数据初始化 |
| FactoryBean | getBean 时 | 定制对象创建（getObject） | SqlSessionFactoryBean |
### 9.2 面试问答
**Q1：Spring 事件是同步还是异步？** A：默认**同步**（SimpleApplicationEventMulticaster 在当前线程依次调用监听器）。异步方案：监听器加 @Async 并开启 @EnableAsync，或自定义 Multicaster 设置 TaskExecutor。注意异步后异常不会传播给发布者，需自行处理。
**Q2：@EventListener 与 @TransactionalEventListener 的区别？** A：@EventListener 在发布瞬间执行（可能仍在事务中、数据未提交）；@TransactionalEventListener 默认 AFTER_COMMIT，事务提交后才执行，**回滚则不执行**。清缓存、发消息、调外部系统都应放在 AFTER_COMMIT。
**Q3：BeanFactoryPostProcessor 与 BeanPostProcessor 的区别？** A：时机不同（Bean 实例化之前 vs 每个 Bean 初始化前后）；对象不同（BeanDefinition vs Bean 实例）；次数不同（一次 vs N 次）。BFPP 改图纸、能排序，但**不能依赖 BeanPostProcessor 产生的 Bean**。
**Q4：想在"所有 Bean 创建完成后"统一做事，用哪个扩展点？** A：`SmartInitializingSingleton#afterSingletonsInstantiated`（第 11 步末尾，最早），或监听 `ContextRefreshedEvent`，或 Boot 的 `ApplicationReadyEvent` / CommandLineRunner（容器完全启动后）。
**Q5：FactoryBean 与 BeanFactory 的区别？** A：BeanFactory 是容器本身；FactoryBean 是用来创建复杂对象的 Bean。`getBean("&name")` 拿 FactoryBean 自身，`getBean("name")` 拿 getObject() 的产物（产物同样经过 BeanPostProcessor）。MyBatis 的 SqlSessionFactoryBean 是典型例子。
### 9.3 高频追问与陷阱
- 陷阱：事务内发布事件且监听器同步抛异常，会导致整体回滚——要么用 @TransactionalEventListener，要么在监听器内吞异常。
- 陷阱：ApplicationContextInitializer 在 Boot 中通过 `SpringApplication.addInitializers` 或 spring.factories（Boot 2.x）/`.imports`（3.x）注册，抓错地方会不生效。
### 9.4 代码/配置示例
```java
public class OrderCreatedEvent extends ApplicationEvent {
    private final Long orderId;
    public OrderCreatedEvent(Object src, Long orderId) { super(src); this.orderId = orderId; }
    public Long getOrderId() { return orderId; }
}
@Component
public class OrderListener {
    @Async @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onCreated(OrderCreatedEvent e) { redis.delete("order:" + e.getOrderId()); }
}
```
### 9.5 一句话速记
事件默认同步；BFPP 改图纸（一次），BP 改产品（每 Bean 两次机会，AOP 在 after）；全部就绪用 SmartInitializingSingleton。

## 自测题（含答案要点）
1. **refresh() 哪两步最关键？** 第 5 步注册/修改 BeanDefinition、第 11 步实例化单例。
2. **BeanFactory 与 ApplicationContext 核心差异？** 功能完整度与实例化时机（懒加载 vs 预实例化）。
3. **@Import 三种用法？** 导入普通类、ImportSelector、ImportBeanDefinitionRegistrar。
4. **@Autowired 与 @Resource 匹配顺序？** byType 优先（多候选按名回退） vs byName 优先。
5. **为什么推荐构造器注入？** 不可变、非空、暴露设计问题、易测试、启动期发现循环依赖。
6. **三级缓存分别存什么？** 一级是成品单例，二级是已取得的早期引用，三级是按需生成早期引用的 ObjectFactory。
7. **两级为什么不够？** 第三级把"是否提前生成代理"推迟到需要时，第二级避免重复 early reference。
8. **构造器注入的循环依赖能解决吗？** 不能，实例化前无法提前暴露引用。
9. **Boot 2.6 起循环依赖的默认行为？** 禁止，`spring.main.allow-circular-references=false`。
10. **AOP 代理在生命周期哪一步生成？** postProcessAfterInitialization。
11. **@PostConstruct / afterPropertiesSet / init-method 顺序？** 依次执行。
12. **单例 Bean 线程安全吗？** 不保证，需无状态或自行同步。
13. **JDK 代理与 CGLIB 的选择规则？** 有接口且 proxyTargetClass=false 用 JDK；Boot 默认 CGLIB。
14. **三种 AOP 失效场景与解法？** 自调用（拆类/自注入代理）、private/final/static 或 final 类（改成可代理的方法/类）、对象未经过容器代理（从容器注入获取）；非 public 方法按代理方式和版本判断，业务事务方法建议 public。
15. **事务默认回滚哪些异常？** RuntimeException 与 Error；checked 需 rollbackFor。
16. **REQUIRES_NEW 与 NESTED 的区别？** 独立物理事务（挂起外层） vs 同一事务内保存点。
17. **事务与多线程的坑？** 事务绑 ThreadLocal，子线程不在同一事务。
18. **大事务的危害？** 长持锁、undo 膨胀、主从延迟、连接池耗尽、回滚代价大。
19. **@EventListener 与 @TransactionalEventListener 的区别？** 即时执行 vs 默认 AFTER_COMMIT，回滚不执行。
20. **BFPP 与 BP 的区别？** 图纸 vs 实例；一次 vs 每 Bean。

## 背诵卡片
1. Spring = IoC 容器 + AOP；Boot = 自动装配 + 起步依赖 + 内嵌容器。
2. Spring 6 基线 JDK 17，`javax.*` → `jakarta.*`。
3. refresh 十二步：第 5 步找图纸，第 11 步造产品。
4. BeanFactory 懒加载，ApplicationContext 预实例化。
5. BeanDefinition 是图纸，Bean 是实例，BeanDefinitionRegistry 是注册中心。
6. Boot 2.1+ 默认禁止 BeanDefinition 覆盖。
7. 自己的类用 @Component，别人的类用 @Bean。
8. @Bean 在 @Configuration 中是 full 模式（CGLIB 保单例），在 @Component 中是 lite 模式。
9. @Import 三兄弟：普通类、ImportSelector、ImportBeanDefinitionRegistrar。
10. 必须依赖用构造器 + final，可选依赖用 setter。
11. @Autowired byType 优先，@Resource byName 优先。
12. 只有一个构造器时 @Autowired 可省略（Spring 4.3+）。
13. 静态字段不能 @Autowired。
14. 多实现冲突顺序：@Qualifier > @Primary > 字段名匹配。
15. 三级缓存：singletonObjects / earlySingletonObjects / singletonFactories。
16. 三级缓存的第三级存 ObjectFactory，不存对象。
17. 构造器注入的循环依赖无解，prototype 无解。
18. @Async 参与循环依赖会报 "injected in raw version"。
19. @Lazy 加注入点能打破循环依赖，但问题推迟到调用时。
20. Boot 2.6 起 `spring.main.allow-circular-references` 默认 false。
21. 生命周期：实例化→填充→Aware→before→@PostConstruct/afterPropertiesSet/init→after(AOP)→使用→销毁。
22. prototype Bean 容器不管销毁。
23. 单例 Bean 不是线程安全的，别放可变成员变量。
24. SimpleDateFormat 不要做单例字段，用 DateTimeFormatter。
25. 通知五兄弟：Before/AfterReturning/AfterThrowing/After/Around。
26. @Around 最强：可阻断、可改参数与返回值。
27. Boot 默认 proxyTargetClass=true，走 CGLIB。
28. CGLIB 不能代理 final 类、final 方法、private 方法。
29. 自调用失效根因是没有经过代理（this 是目标对象）。
30. 破解自调用：拆类、自注入代理、AopContext.currentProxy（需 exposeProxy）。
31. 事务默认只回滚 RuntimeException 与 Error。
32. 事务方法必须 public。
33. 七种传播：REQUIRED/REQUIRES_NEW/SUPPORTS/NOT_SUPPORTED/MANDATORY/NEVER/NESTED。
34. REQUIRES_NEW 挂起外层独立提交；NESTED 用保存点且随外层回滚。
35. 事务绑在 ThreadLocal，子线程不共享。
36. 提交后再动作：TransactionSynchronization#afterCommit 或 @TransactionalEventListener。
37. 大事务 = 长锁 + undo 膨胀 + 连接占用。
38. 事件默认同步；异步要 @Async 或自定义 Multicaster。
39. BFPP 改 BeanDefinition（一次），BP 改 Bean 实例（每个 Bean）。
40. BP 里不要 getBean 业务 Bean，会导致提前实例化。
