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
Spring 的本质是**一个 IoC 容器 + 一套 AOP 基础设施**，其余模块都是在这两块之上做应用集成。理解这一点就能理解：事务失效、缓存失效、注解不生效，根子上都是同一个问题——**代理没生成，或调用没走代理**。

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

byType 冲突时的解析顺序：**@Qualifier 指定 → @Primary 优先 → 字段名/参数名匹配 beanName**，都不满足抛 `NoUniqueBeanDefinitionException`。
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

| 缓存 | 字段 | 内容 | 作用 |
| --- | --- | --- | --- |
| 一级 | singletonObjects | 完整初始化的单例 | 对外提供成品 |
| 二级 | earlySingletonObjects | 提前暴露的半成品（可能已是代理） | 保证同一 Bean 只早期暴露一次 |
| 三级 | singletonFactories | `ObjectFactory`（getEarlyBeanReference） | 延迟决定是否生成 AOP 代理 |

解决 A↔B 的链路：A 实例化后把 ObjectFactory 放入三级缓存 → 填充属性需要 B → 创建 B 时又需要 A → `getSingleton("a", true)` 命中三级缓存，调 getEarlyBeanReference 得到 A 的早期引用（**若 A 需要 AOP 则此时生成代理**）→ 放入二级缓存并移除三级缓存 → B 完成创建进入一级缓存 → A 完成剩余初始化（含最终代理）进入一级缓存。

**为什么必须三级、两级不行？** 一级缓存会拿到未完成对象（不行）；只有一二级且二级直接存对象也能跑通，但会带来两个问题：无循环依赖时也要提前生成所有代理（破坏"代理统一在 postProcessAfterInitialization 生成"的约定），以及同一 Bean 的早期引用被反复计算。第三级存"工厂"把"要不要提前代理"的判断推迟到真正被循环依赖方需要时；二级缓存缓存已生成的早期引用，避免重复执行 getEarlyBeanReference、避免出现两个不同对象。源码位置：`AbstractAutowireCapableBeanFactory#doCreateBean` 中的 `addSingletonFactory`。

**构造器注入为什么无解？** 构造器要求实例化阶段就拿到依赖，而 `addSingletonFactory` 在实例化之后才执行，A 当时连对象都没有，无法提前暴露，只能抛 `BeanCurrentlyInCreationException`。**@Lazy 与 @Async 的坑**：@Lazy 加在注入点会注入代理，真实对象在首次调用时才从容器获取，从而打破启动期循环（代价是问题延后暴露）；`@Async` Bean 在初始化后被包成代理，若参与循环依赖会报 `Bean with name 'x' has been injected into other beans [y] in its raw version as part of a circular reference, but has eventually been wrapped`——B 拿到原始版本、容器里是代理版本，身份不一致。**Spring Boot 2.6 起 `spring.main.allow-circular-references` 默认 false**，循环依赖直接启动失败。
### 5.2 面试问答
**Q1：Spring 如何解决循环依赖？** A：通过**三级缓存提前暴露半成品 Bean**，仅对单例 + setter/字段注入有效。原理是实例化与属性填充分离，实例化后立即把工厂放入三级缓存，让环上另一方先拿到引用完成装配。边界：prototype 无解、构造器注入无解、@Async 会拿到原始对象。
**Q2：为什么需要三级缓存，两级不行吗？** A：两级能跑通但语义与效率都不好：三级缓存存 ObjectFactory，把是否提前生成代理的决策推迟（无循环依赖时代理仍统一在 after 阶段生成）；二级缓存存已暴露的早期引用，避免同一 Bean 重复执行 getEarlyBeanReference。主流结论：第三级为 AOP 语义服务，第二级为一致性与性能服务；也有"两级也能用"的观点，面试答出三级的设计动机与两级的问题即可。
**Q3：prototype 作用域的循环依赖能解决吗？** A：不能。prototype 每次 getBean 都新建对象，没有共享的半成品可暴露，报 `BeanCurrentlyInCreationException: Requested bean is currently in creation: Is there an unresolvable circular reference?`。
**Q4：循环依赖有什么危害？线上报错怎么排查？** A：危害：掩盖设计问题（两个类强耦合）、AOP 场景下可能注入未增强对象导致事务失效、启动顺序敏感、与 @Async/@Transactional 组合易出隐性 bug。排查：异常会打印完整环（`The dependencies of some of the beans in the application context form a cycle: orderService ┐ ↓ userService ↑ ...`），据此定位两个 Bean；优先重构（抽第三层服务、用事件解耦、把反向调用改成查询），应急可给注入点加 @Lazy 或临时开 `spring.main.allow-circular-references=true`。
### 5.3 高频追问与陷阱
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
三级缓存提前暴露半成品解决 setter/字段循环依赖；构造器注入与 prototype 无解，@Async 会踩坑，Boot 2.6 起默认禁止。

## 6. Bean 生命周期与作用域
### 6.1 核心原理

| 阶段 | 回调 | 说明 |
| --- | --- | --- |
| 1 实例化 | 构造器 / 工厂方法 | createBeanInstance，选择构造器 |
| 2 属性填充 | @Autowired / @Value / setter | populateBean，可能触发依赖 Bean 创建 |
| 3 Aware（基础） | BeanNameAware → BeanClassLoaderAware → BeanFactoryAware | invokeAwareMethods 直接调用 |
| 4 BP#before | ApplicationContextAwareProcessor 处理 Environment/ResourceLoader/ApplicationEventPublisher/MessageSource/ApplicationContextAware | 按 PriorityOrdered → Ordered → 无序 |
| 5 初始化 | @PostConstruct → InitializingBean#afterPropertiesSet → init-method | 顺序固定 |
| 6 BP#after | **AOP 代理在此生成**（AbstractAutoProxyCreator） | 事务/日志增强生效点 |
| 7 使用 | getBean 返回（可能是代理），单例放一级缓存 | 供应期 |
| 8 销毁 | @PreDestroy → DisposableBean#destroy → destroy-method | 仅容器 close 时；prototype 不管理 |

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
@Component
public class LifeCycleDemo implements InitializingBean, DisposableBean, BeanNameAware {
    @PostConstruct public void post() { System.out.println("1 @PostConstruct"); }
    @Override public void afterPropertiesSet() { System.out.println("2 afterPropertiesSet"); }
    public void initMethod() { System.out.println("3 init-method"); }
    @PreDestroy public void pre() { System.out.println("4 @PreDestroy"); }
    @Override public void destroy() { System.out.println("5 destroy"); }
    @Override public void setBeanName(String n) { System.out.println("0 aware: " + n); }
}
```
### 6.5 一句话速记
实例化→填充→Aware→before→三板斧→after（AOP 在这）→使用→销毁；单例不保证线程安全。

## 7. AOP 原理与失效场景
### 7.1 核心原理

| 通知 | 注解 | 时机 | 能力 |
| --- | --- | --- | --- |
| 前置 | @Before | 目标方法前 | 不能改返回值，抛异常可阻断 |
| 返回 | @AfterReturning | 正常返回后 | 可读/改返回值（需 returning 参数） |
| 异常 | @AfterThrowing | 抛异常后 | 可包装异常 |
| 后置 | @After | finally 语义 | 不能阻断，总执行 |
| 环绕 | @Around | 前后包裹 | 可阻断、改参数、改返回值（最强） |

执行顺序：@Around 前半 → @Before → 目标 → @AfterReturning/@AfterThrowing → @After → @Around 后半。切点表达式：`execution(* com.x.service..*.*(..))`（最常用）、`@annotation(...)`、`within`、`this/target/args`、`@within`、`bean(...)`。

| 维度 | JDK 动态代理 | CGLIB |
| --- | --- | --- |
| 原理 | 基于接口生成实现类（Proxy + InvocationHandler） | 基于继承生成子类（ASM） |
| 前提 | 目标类必须有接口 | 类与方法不能 final/private/static |
| 性能 | 创建快，调用略慢（JDK 8+ 差距极小） | 创建慢，调用快 |
| 代理类型 | 接口类型（强转原类失败） | 子类类型（可强转原类） |
| Spring 选择 | proxyTargetClass=false 且有接口 | proxyTargetClass=true 或无接口 |

**Spring Boot 2.x/3.x 默认 `spring.aop.proxy-target-class=true`，即默认 CGLIB**；Spring Framework 原生 `@EnableAspectJAutoProxy` 默认 proxyTargetClass=false。Boot 选 CGLIB 的原因：避免"按实现类类型注入却找不到 Bean"，并统一处理无接口的类。**失效场景**：同类内部调用（未走代理，解法是拆类/自注入代理/AopContext.currentProxy 需 exposeProxy=true）、private/static/final 方法、@Transactional 非 public 方法、类或方法被 final、对象不是容器创建的、切点写错或类未被扫描、BFPP 中提前 getBean、异常被 catch 吞掉。
### 7.2 面试问答
**Q1：Spring AOP 的实现原理？** A：**运行时动态代理 + 通知责任链**。Bean 初始化后由 AbstractAutoProxyCreator（一个 BeanPostProcessor）判断是否匹配切点，匹配则生成 JDK 或 CGLIB 代理，调用时由 ReflectiveMethodInvocation 按序织入通知。边界：Spring AOP 只支持方法级切点，仅对容器管理的 Bean 生效；字段/构造器切点与编译期织入需要 AspectJ。
**Q2：JDK 动态代理和 CGLIB 怎么选？** A：规则：有接口且 proxyTargetClass=false → JDK；否则 CGLIB；**Boot 默认 proxyTargetClass=true → CGLIB**。要点：CGLIB 不能代理 final 类、final 方法、private 方法；JDK 代理只能按接口强转，按具体实现类注入会失败。
**Q3：同一个类里 A 调 B，B 上的 @Transactional/@Cacheable 为什么失效？** A：因为调用发生在目标对象内部（this 引用），未经过代理。解法：① 把 B 拆到另一个 Bean（最佳）；② 自注入 `@Autowired @Lazy private XxxService self` 后调 `self.b()`；③ `AopContext.currentProxy()`（需 `@EnableAspectJAutoProxy(exposeProxy = true)`）；④ 改用 TransactionTemplate 编程式事务。
**Q4：什么时候必须用 @Around？多个切面顺序怎么定？** A：需要"改返回值/改参数/决定是否执行目标方法/统计成功失败"时必须用 @Around（@Before/@After 无法阻断或改返回）。顺序由 @Order/Ordered 决定，值越小越靠外（先进入、后退出）；**事务切面优先级最低**，所以自定义环绕切面通常在事务外层，注意事务提交发生在自定义切面结束之前。
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
AOP = 动态代理 + 通知责任链，代理在 BP#after 生成；自调用、private/final/static、非 public 事务都会失效。

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

**事务失效清单**：① 同类自调用；② 方法非 public；③ 类/方法 final、static；④ 类未被容器管理；⑤ 异常被 catch 吞掉；⑥ checked 异常未配 rollbackFor；⑦ 数据库引擎不支持事务（MyISAM）；⑧ 多数据源未指定事务管理器；⑨ 传播行为不当（NOT_SUPPORTED/SUPPORTS 而外层无事务）；⑩ 异步/新线程中操作 DB（事务绑在 ThreadLocal，子线程不在同一事务）；⑪ @Transactional 标在接口上且代理模式不匹配（推荐标实现类）。

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
@Transactional = 代理 + TransactionInterceptor；默认只回滚 RuntimeException；自调用、非 public、checked 异常、多线程是四大失效原因。

## 9. 事件机制与容器扩展点
### 9.1 核心原理
组件：ApplicationEvent（事件）、ApplicationEventPublisher（发布者）、ApplicationEventMulticaster（默认 `SimpleApplicationEventMulticaster`，**同步**）、ApplicationListener / @EventListener。同步广播意味着监听器异常会向发布者传播、可能把事务带崩；`TransactionPhase` 有 BEFORE_COMMIT、AFTER_COMMIT、AFTER_ROLLBACK、AFTER_COMPLETION。

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
6. **三级缓存分别存什么？** 成品单例 / 早期暴露的半成品（含代理）/ ObjectFactory。
7. **两级为什么不够？** 第三级把"是否提前生成代理"推迟到需要时，第二级避免重复 early reference。
8. **构造器注入的循环依赖能解决吗？** 不能，实例化前无法提前暴露引用。
9. **Boot 2.6 起循环依赖的默认行为？** 禁止，`spring.main.allow-circular-references=false`。
10. **AOP 代理在生命周期哪一步生成？** postProcessAfterInitialization。
11. **@PostConstruct / afterPropertiesSet / init-method 顺序？** 依次执行。
12. **单例 Bean 线程安全吗？** 不保证，需无状态或自行同步。
13. **JDK 代理与 CGLIB 的选择规则？** 有接口且 proxyTargetClass=false 用 JDK；Boot 默认 CGLIB。
14. **三种 AOP 失效场景与解法？** 自调用（拆类/自注入代理）、private/final/static（改签名）、非 public 事务（改 public）。
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
