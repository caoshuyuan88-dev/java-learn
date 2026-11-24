# Spring Cloud 微服务体系

> **定位**：把注册发现、配置中心、负载均衡、服务调用、熔断限流、网关、链路追踪与服务治理串成一条完整链路，并讲清版本对齐这个最大的工程坑。
> **重要度**：★★★★★（中大型互联网公司后端岗高频，网关与熔断限流是必问）
> **建议复习时长**：6~8 小时（可搭一套 Nacos + Gateway + Feign + Sentinel 做实验；这是示例组合，不代表所有项目都必须使用这些组件）
> **关联文档**：[Spring Boot 与自动装配](./02-SpringBoot与自动装配.md)、[Spring MVC 与 Web 容器](./03-SpringMVC与Web容器.md)、[Nginx 与网关限流](../04-中间件/05-Nginx与网关限流.md)、[分布式事务](../05-分布式与架构/02-分布式事务.md)、[高并发高可用架构设计](../05-分布式与架构/04-高并发高可用架构设计.md)、[CI/CD 与监控告警](../07-工程化与DevOps/03-CICD与监控告警.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| 微服务拆分原则与演进 | ★★★★☆ | 场景题 |
| 版本对应关系与对齐 | ★★★★★ | 场景题 / 故障排查 |
| 注册中心（Eureka/Nacos/Consul） | ★★★★★ | 八股原理 / 故障排查 |
| 配置中心与动态刷新 | ★★★★★ | 配置实践 / 故障排查 |
| 负载均衡（Ribbon → LoadBalancer） | ★★★★☆ | 八股原理 / 配置实践 |
| OpenFeign 原理与配置 | ★★★★★ | 源码追问 / 故障排查 |
| 熔断限流降级（Hystrix → Sentinel） | ★★★★★ | 八股原理 / 场景题 |
| 网关（Gateway 与 Zuul 对比） | ★★★★★ | 配置实践 / 场景题 |
| 链路追踪（Sleuth → Micrometer Tracing） | ★★★★☆ | 配置实践 |
| 服务治理（灰度、优雅上下线、超时重试） | ★★★★☆ | 场景题 |
| 微服务线上问题排查 | ★★★★★ | 故障排查 |

## 1. 微服务拆分原则与演进
### 1.1 核心原理
**单体到微服务的演进路径**：单体 → 模块化单体（清晰包边界）→ 服务拆分（按业务能力/领域）。**拆分原则**：① **单一职责与高内聚**（一个服务只做一件事，改动原因唯一）；② **DDD 限界上下文**（聚合根与一致性边界决定服务边界，跨上下文用事件/接口而非直接改库）；③ **按业务能力/组织架构拆分**（康威定律：架构会向组织结构收敛）；④ **数据私有**（每个服务独占库/schema，禁止跨服务直连数据库）；⑤ **调用链与事务边界尽量短**（避免长事务跨服务）。

**拆分的代价**：分布式事务、跨服务查询（需数据聚合/冗余）、链式调用放大故障、运维与排障成本、测试成本、网络延迟。**判断标准**：团队规模、发布频率、性能与扩容需求、故障隔离要求。**结论：拆得过细比不拆更糟**，"分布式单体"（服务拆了但强耦合、必须一起发布）是常见的失败案例。

**先理解“服务”与“微服务”**：服务是有独立边界、接口和运行生命周期的进程；“微”不是按代码行数或表数量定义，而是强调边界清晰、可独立交付和治理。一次普通调用可能变成 `客户端 -> Gateway -> order-service -> inventory-service -> 数据库`，每个跨进程箭头都增加网络延迟、序列化、超时和失败概率。把单体拆成多个进程不会自动提高性能；只有当独立扩容、并行开发或隔离故障的收益超过网络、数据一致性、部署和观测成本时，拆分才划算。

**Spring Cloud 是能力集合，不是强制架构清单**：它提供若干可组合的抽象和集成（如服务发现、客户端负载均衡、网关、断路器、配置），并不要求每个系统都必须采用注册中心、Feign、Nacos、Sentinel 全家桶。Kubernetes DNS/Service、云厂商网关、Service Mesh、HTTP 客户端或平台配置服务也可承担相同职责。先识别问题与运行平台，再选择一套职责清晰、避免重复治理的组件。
### 1.2 面试问答
**Q1：微服务拆分的粒度怎么定？** A：从"改动频率高、需要独立扩容、团队归属不同"的地方先拆；一个服务由一个小团队（2 pizza team）能完整负责；边界与限界上下文一致；避免"贫血服务"（只有一个 CRUD 方法）。粒度过细的标志：一次业务改动要跨 5 个以上服务、服务间调用链超过 4 层。
**Q2：为什么禁止服务间共享数据库？** A：共享库会让服务退化成"分布式单体"（表结构改动耦合、无法独立部署）、破坏封装、让分布式事务无法回避。正确做法是服务暴露能力接口，数据通过 API/事件同步（冗余读模型是常规代价）。
**Q3：单体一定不好吗？** A：不是。中小团队/早期业务用模块化单体效率更高：部署简单、事务简单、重构成本低。**只有在拆分收益（独立扩容、故障隔离、多团队并行）大于成本时才拆**，面试时能说出这个取舍是加分项。
**Q4：微服务下跨服务查询怎么解决？** A：① API 组合（网关/BFF 聚合，最常用）；② 数据冗余（订单服务冗余用户名，通过事件更新）；③ 读模型/CQRS（宽表或 ES 提供聚合查询）；④ 需要跨服务 join 的复杂报表走数据仓库/离线链路，不要在主链路做。
### 1.3 高频追问与陷阱
- 陷阱：把"服务拆分"当成性能优化手段——拆分会增加网络与排障成本，吞吐瓶颈通常要靠缓存/异步/分库分表解决。
- 追问：康威定律的实践含义是"服务边界要匹配团队边界"，否则会出现跨团队强依赖。
### 1.4 代码/配置示例
```text
order-service（订单上下文，独占 order 库） / user-service（用户上下文） / inventory-service（库存上下文）
跨上下文协作：order 调 inventory 的 RPC 扣减接口，成功后发领域事件
```
### 1.5 一句话速记
拆分看"独立扩容 + 故障隔离 + 团队边界"，数据必须私有，粒度过细比不拆更糟。

## 2. 版本对应关系与对齐
### 2.1 核心原理
Spring Cloud 采用**版本列车（release train）+ 年份版本号**，每个 train 只支持特定范围的 Spring Boot 版本。

| Spring Cloud | Spring Boot | 关键变化 |
| --- | --- | --- |
| Hoxton (2.2.x/2.3.x) | 2.2.x / 2.3.x | 仍可用 Ribbon/Hystrix |
| 2020.0 (Ilford) | 2.4.x / 2.5.x | **移除 Ribbon**，改用 Spring Cloud LoadBalancer；引入 Spring Cloud CircuitBreaker 抽象 |
| 2021.0 (Jubilee) | 2.6.x / 2.7.x | 与 Boot 2.6 的循环依赖/路径匹配变化同步 |
| 2022.0 (Kilburn) | 3.0.x / 3.1.x | 对应 Spring 6 + jakarta，Sleuth 被 Micrometer Tracing 取代 |
| 2023.0 (Leyton) | 3.2.x；从 2023.0.2 起支持 3.3.x | Boot 3.x 时代的组件列车 |
| 2024.0 (Moorgate) | 3.4.x | Boot 3.4 对应列车 |
| 2025.0 (Northfields) | 3.5.x | Boot 3.5 对应列车 |
| 2025.1 (Oakwood) | 4.0.x | Boot 4.0 对应列车；具体组件以该 train 的 BOM 为准 |

**版本信息的边界**：上表是 Spring Cloud 官方支持矩阵中的主要对应关系，不等于每个补丁版本都能任意混用；例如 2023.0.x 对 Boot 3.3.x 的支持始于特定补丁版本。Spring Cloud Alibaba（SCA）有自己的发布列车和组件矩阵，必须核对所选 SCA 版本、Spring Cloud train、Boot 版本、Nacos 服务端/客户端的官方兼容说明，不能仅按相同年份号推断兼容。引入方式通常使用 `spring-cloud-dependencies` 与匹配版本的 `spring-cloud-alibaba-dependencies` BOM，具体 artifact 和支持范围以官方发布说明为准。

**怎么理解版本列车**：Spring Boot 管理 Spring Framework、Servlet 容器等基础设施；Spring Cloud 再基于 Boot 提供服务注册、配置、网关和调用等组件。列车 BOM 的作用是将一组经过协同测试的 Cloud 组件版本作为整体管理，不是一个独立的 Spring Boot 版本。使用时先选 Boot 主次版本，再查对应 Cloud train，最后查 SCA/Nacos 等第三方生态；不要先从博客复制一个组件版本再反向拼依赖。官方支持矩阵会随生命周期更新，升级前以 [Spring Cloud Supported Versions](https://github.com/spring-cloud/spring-cloud-release/wiki/Supported-Versions) 与组件各自文档为准。

**版本不对齐的典型症状**：`NoSuchMethodError`、`NoClassDefFoundError`、`AbstractMethodError`、Gateway 启动报 Netty 版本冲突、Feign 编解码类找不到、Nacos 注册成功但配置不生效。**排查手段**：`mvn dependency:tree` 看 Boot/Cloud/Netty/Jackson 的实际版本，用 BOM 收敛而不是逐个指定版本。
### 2.2 面试问答
**Q1：为什么 Spring Cloud 版本必须和 Boot 对齐？** A：Spring Cloud 组件大量依赖 Spring 内部的类与方法签名（如 `WebApplicationContext`、`EnvironmentPostProcessor`、Netty/Jackson 版本），Boot 大版本升级会改动这些内部实现；Cloud 的每个 train 只针对特定 Boot 范围做过兼容测试。不对齐时最常见的是运行期 `NoSuchMethodError`，编译期发现不了。
**Q2：升级到 Boot 3 时 Cloud 侧要注意什么？** A：① 换 train 到 2022.0+/2023.0+；② `javax→jakarta` 影响所有 Cloud 组件的 Servlet 相关配置；③ Sleuth → Micrometer Tracing；④ Config Server 的加密/仓库配置格式有变化；⑤ Gateway 的谓词与过滤器实现细节调整；⑥ Nacos 客户端需切到支持 Boot 3 的 SCA 版本。
**Q3：BOM 是干什么的？** A：`spring-cloud-dependencies` 是 BOM（Bill of Materials），通过 `<dependencyManagement>` 统一给所有 Cloud 组件定版本，业务只写 groupId/artifactId 不写 version。**混用多个 BOM 或手动指定版本是版本冲突的主因**。
**Q4：怎么快速检查依赖冲突？** A：`mvn dependency:tree -Dincludes=io.netty,com.fasterxml.jackson.core`、`mvn enforcer:enforce`（banned dependencies 规则）、启动参数 `-Dspring.cloud.compatibility-verifier.enabled=true`（Cloud 的兼容性校验，适用于部分版本，以官方文档为准），以及关注启动日志里的版本告警。
### 2.3 高频追问与陷阱
- 陷阱：只升 Boot 不升 Cloud（或反之）是最常见的"环境能起、一调就炸"的原因。
- 追问：Nacos 客户端版本与服务端版本也有兼容矩阵，服务端 2.x 建议客户端也用 2.x。
### 2.4 代码/配置示例
```xml
<dependencyManagement><dependencies>
  <dependency><groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-dependencies</artifactId>
    <version>2023.0.1</version><type>pom</type><scope>import</scope></dependency>
  <dependency><groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-dependencies</artifactId>
    <version>2023.0.1.0</version>   <!-- 以官方版本对应表为准 -->
    <type>pom</type><scope>import</scope></dependency>
</dependencies></dependencyManagement>
```
### 2.5 一句话速记
Cloud 是版本列车：Hoxton→2020.0→2021.0→2022.0→2023.0 分别对应 Boot 2.2/2.4/2.6/3.0/3.2，永远用 BOM 收敛版本。

## 3. 注册中心
### 3.1 核心原理
**服务注册与发现流程**：服务启动 → 向注册中心注册实例（IP、端口、权重、元数据、健康状态）→ 定期心跳续约 → 消费者从注册中心拉取实例列表并缓存在本地 → 通过负载均衡选择实例调用 → 实例下线/心跳超时被剔除 → 消费者收到变更通知并刷新本地缓存。

| 维度 | Eureka | Nacos | Consul |
| --- | --- | --- | --- |
| 一致性模型 | **AP**（可用性优先） | 支持 AP/CP（临时实例用 Distro 协议 = AP；持久实例用 Raft = CP） | **CP**（Raft） |
| 健康检查 | 客户端心跳（默认 30s 续约，90s 未续约剔除） | 临时实例心跳（5s），15s 无心跳标记不健康，30s 剔除；持久实例服务端主动探测 | 服务端主动探测（TCP/HTTP/gRPC/脚本） |
| 自我保护 | 有（续约率低于阈值时停止剔除） | 无此机制，改用健康标记 | 无 |
| 配置中心 | 无（需 Config Server） | **内置**（同进程提供配置管理） | 有 KV |
| 控制台/生态 | 简单，已停止新特性（Netflix 维护模式） | 功能全、国内主流 | 国外常用，与 K8s 结合好 |
| 适用 | 老系统、纯 AP 需求 | 新项目首选（注册 + 配置一体） | 强一致、多语言环境 |

**Eureka 自我保护机制**：当最近一分钟实际收到的续约数低于阈值（`renewal-percent-threshold` 默认 0.85，窗口 `renewal-threshold-update-interval-ms` 默认 15 分钟）时，Eureka 认为"可能是自己网络问题"而不是服务大面积挂掉，于是**停止剔除过期实例**，保护消费者不因误剔除而无法调用。副作用是"服务明明挂了却还能被调用到"（返回连接拒绝），所以排查时要注意区分"真挂"与"自我保护"。可通过 `eureka.server.enable-self-preservation=false` 关闭（一般不建议在生产关闭）。

**Nacos 的 AP/CP 切换**：注册的实例分为**临时实例**（默认，`ephemeral=true`，客户端心跳 + Distro 协议，保证可用性与最终一致）与**持久实例**（`ephemeral=false`，服务端健康检查 + Raft，强一致）；**不是整个集群切换 AP/CP，而是按实例类型选择一致性模型**——这是常见的面试误解点。

**为什么消费者缓存实例列表**：注册发现不是每次 RPC 都先问中心“地址是什么”。客户端启动时订阅某服务的实例列表，注册中心推送或客户端定期拉取变更，客户端本地缓存后由负载均衡器选实例。这样调用路径更短，也避免注册中心成为每次业务请求的瓶颈；代价是视图短暂不一致，所以刚下线的实例可能仍被调用。优雅下线应先摘流量，再等待订阅与负载均衡缓存传播，最后停止进程。

**注册中心只解决地址与健康视图，不保证调用成功**：实例“健康”不等于业务接口可用，心跳也不等于端到端探测。消费者仍必须配置连接/读取超时、错误处理、限流与有界重试。网络分区时提供者、消费者与注册中心可能持有不同视图；服务发现本身不提供数据一致性、鉴权，也不自动完成负载均衡。注册中心的 namespace、group、cluster 和实例元数据常用于隔离环境、逻辑服务组与机房，但含义及默认值依产品配置而异。
### 3.2 面试问答
**Q1：注册中心的作用与核心机制？** A：解耦服务提供者与消费者的地址依赖，提供注册、发现、健康检查、变更通知。机制是"客户端注册 + 心跳续约 + 本地缓存 + 变更推送 + 负载均衡选实例"，消费者不每次调注册中心（否则注册中心是单点与瓶颈），而是本地缓存 + 事件刷新。
**Q2：Eureka 的自我保护要开还是关？** A：默认开、生产一般保持开启，因为它的初衷是抵御网络分区导致的误剔除（AP 取向）；但要配合监控告警（自我保护触发是重要故障信号），并清楚"可能调用到已死实例"的副作用（靠客户端快速失败 + 重试兜底）。测试环境可关闭以获得更快的实例摘除。
**Q3：Nacos 是 AP 还是 CP？** A：两者都支持，但按实例类型区分：临时实例走 Distro（AP，心跳 + 最终一致，适合绝大多数微服务实例）；持久实例走 Raft（CP，强一致，适合 DNS 场景或不允许心跳的实例）。**结论：不要笼统说 Nacos 是 AP 或 CP。**
**Q4：注册中心挂了会怎样？服务实例异常时消费者会怎样？** A：已运行的服务大多仍可互相调用（消费者有本地实例缓存），但**新实例无法注册、下线实例摘除失效、配置推送失效**，故障随实例上下线逐步放大，所以注册中心要高可用部署并配监控。实例异常时的表现取决于实现：Eureka 依赖心跳与剔除（有自我保护时会延迟），Nacos 把实例标为不健康但 LB 默认只选健康实例；即使选到坏实例，也靠连接超时 + 幂等重试兜底。
### 3.3 高频追问与陷阱
- 陷阱：Eureka 停止新特性（进入维护模式），新项目不建议再选；Ribbon/Hystrix 同样处于维护状态。
- 陷阱：服务下线后消费者仍调用了很久——这是"本地缓存 + 剔除延迟 + 注册中心自我保护"共同导致，需要配合优雅停机与 LB 缓存 TTL。
### 3.4 代码/配置示例
```yaml
spring:
  application: { name: order-service }
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: prod-namespace-id   # 许多版本要求填 namespace ID，而非控制台显示名
        ephemeral: true          # 临时实例：心跳 + AP
        metadata: { version: v1 }   # 灰度发布常用元数据
```
该 YAML 展示常见属性形态；SCA 与 Nacos 客户端版本不同，属性名、namespace 填值和默认行为可能变化。示例中的注册地址为本机开发环境，生产应配置集群地址、认证信息与网络隔离。
### 3.5 一句话速记
注册中心 = 注册 + 心跳 + 本地缓存 + 变更推送；Eureka 是 AP 且有自我保护，Nacos 按临时/持久实例分别走 AP/CP。

## 4. 配置中心与动态刷新
### 4.1 核心原理
**Nacos 配置模型**：`namespace`（环境/租户隔离，如 dev/test/prod）→ `group`（业务/项目分组）→ `dataId`（具体配置，通常为 `${spring.application.name}-${profile}.${file-extension}`，如 `order-service-prod.yaml`）。多配置合并有优先级：主配置（`spring.cloud.nacos.config.*`）> 扩展配置 `extension-configs` > 共享配置 `shared-configs`，数组下标越大优先级越高——**此顺序和属性键会随 SCA/客户端版本变化，必须以所用版本官方文档为准，并通过启动日志或 `/actuator/env` 确认最终值**。

**动态刷新原理**：Nacos 客户端用**长轮询**（客户端发起请求最长挂起约 30s，服务端有变更立即返回）监听配置变化 → 变更后发布 `RefreshEvent` → 容器刷新带 `@RefreshScope` 的 Bean（销毁并重建代理的目标对象）→ 同时刷新已绑定的 `@ConfigurationProperties`。**注意：不是所有 Bean 都会刷新，只有 `@RefreshScope`（或 `@ConfigurationProperties` 且位于刷新范围内）才会。**

**动态刷新的坑**：① 普通 Bean 里 `@Value` 注入的值不会更新（Bean 没被重建）；② 静态字段/`static final` 常量不刷新；③ 在 `@PostConstruct` 里读取的配置只在启动时生效；④ 线程池、连接池等已初始化资源不会因为属性变化而重建（需要监听事件手动处理）；⑤ `@RefreshScope` 的 Bean 是代理，**首次访问才创建**，若构造很重会导致首次请求变慢；⑥ 与 `@Scheduled`/`@Async` 结合时要用 `@RefreshScope` 或重建线程池；⑦ 刷新风暴：大批实例同时监听同一 dataId，发布时注意分批与灰度。

**配置变更的完整链路**：运维或发布系统提交新配置 -> 配置中心鉴权、保存版本并通知客户端 -> 客户端获取变更并更新 Environment/绑定对象 -> 应用中的刷新监听器重建或重绑相关 Bean -> 新请求使用新值。每一步都可能失败，因而生产配置要有版本记录、审计、审批、回滚、变更通知和实例生效状态观测。多项相互依赖的配置不要假定所有实例在同一瞬间原子切换；可通过兼容式双配置、分批发布和灰度验证降低短暂版本不一致的风险。

**哪些配置不应随意动态改**：阈值、开关、文案等可刷新配置较容易处理；数据库连接池大小、线程池、服务监听端口、序列化格式等资源型/启动型配置往往需要显式重建，甚至滚动重启。把密钥放配置中心并不自动安全，还要做访问控制、加密、脱敏、轮换和审计。应用应有可工作的本地默认配置或明确的启动失败策略，避免配置中心失联时每个实例行为不可预测。
### 4.2 面试问答
**Q1：配置中心怎么实现动态刷新？** A：客户端长轮询（或推送通道）监听变更 → 收到变更后拉取新配置 → 更新 Environment 中的 PropertySource → 发布 RefreshEvent → 由 `RefreshScope` 重建作用域内 Bean、由 ConfigurationPropertiesRebinder 重新绑定属性。所以"刷新"本质是"重建 Bean + 重绑属性"，而不是原地修改字段。
**Q2：为什么 `@Value` 有时不刷新？** A：因为 `@Value` 是在 Bean 创建时解析并写入字段的，配置刷新后如果该 Bean 不在 `@RefreshScope` 内就不会被重建，字段仍是旧值。解法：Bean 上加 `@RefreshScope`，或改用 `@ConfigurationProperties`（支持 rebinder），或从 `Environment` 每次读取（不推荐）。
**Q3：配置中心怎么做灰度、回滚与安全？** A：灰度用 Nacos 的 beta 发布（按 IP 列表生效）或按 namespace/group 隔离环境，先灰度少量实例再全量；回滚用历史版本对比与一键回滚，并把配置纳入 Git 与审批流程（IaC），避免控制台手改无法追溯；安全上要开控制台鉴权与 RBAC、敏感配置用 KMS/Vault 或加密插件而不落明文、开启变更审计，同时利用客户端本地快照（`~/nacos/config`）与 failover 目录保证配置中心不可用时仍能启动。
### 4.3 高频追问与陷阱
- 陷阱：把敏感配置直接放 Nacos 且未加密 → 泄露风险；同时要注意配置中心自身的权限边界。
- 追问：`spring.cloud.nacos.config.refresh-enabled=false` 可关闭自动刷新；`@RefreshScope` 不能用在 `@Configuration` 类上（会有意外行为，官方不推荐）。
### 4.4 代码/配置示例
```yaml
spring:
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        namespace: prod
        group: ORDER_GROUP
        file-extension: yaml
        shared-configs: [{ data-id: common.yaml, refresh: true }]
```
### 4.5 一句话速记
Nacos 模型 namespace > group > dataId，动态刷新靠长轮询 + RefreshEvent + 重建 @RefreshScope Bean；@Value 不刷新是没加刷新作用域。

## 5. 负载均衡
### 5.1 核心原理
| 维度 | 客户端负载均衡（Spring Cloud LoadBalancer / 原 Ribbon） | 服务端负载均衡（Nginx / LB） |
| --- | --- | --- |
| 位置 | 调用方进程内，先选实例再直连 | 独立组件，请求先到 LB |
| 感知注册中心 | 是（可动态选健康实例） | 通常静态或需对接注册中心 |
| 优点 | 少一跳、可按方法/元数据精细路由 | 与语言无关、集中治理 |
| 缺点 | 每语言都要实现、实例变更需刷新缓存 | 多一跳、动态感知较弱 |

**Ribbon 已停止维护**：Spring Cloud 2020.0（Ilford）起移除 Ribbon，统一用 **Spring Cloud LoadBalancer**（`spring-cloud-starter-loadbalancer`）。**常见策略**：`RoundRobinLoadBalancer`（默认轮询）、`RandomLoadBalancer`（随机）、`NacosLoadBalancer`（Nacos 权重/同集群优先，SCA 提供）；可自定义 `ReactorServiceInstanceLoadBalancer` 或实现 `ServiceInstanceListSupplier` 做灰度/同机房优先。缓存：默认缓存实例列表（`spring.cloud.loadbalancer.cache.ttl`，可配置）；健康检查可通过 `spring.cloud.loadbalancer.health-check.*` 开启主动检查。

**负载均衡做什么、不做什么**：它从可用实例集合中选一个目标，然后把请求发给该实例；默认轮询/随机并不等于根据 CPU、响应时间或业务容量智能选节点。实例过滤（健康、zone、版本、灰度）、选取策略、重试和连接池是不同环节，不能把它们混为“负载均衡自动解决高可用”。多实例仍需探活、超时、连接失败处理和容量规划；如果注册中心/实例缓存短暂过期，选到失效地址仍可能失败。

**客户端与服务端方案的运行代价**：客户端负载均衡要求每个调用方持有实例列表并实现一致的选择行为；服务端代理让所有调用集中经过代理，策略集中但多一跳且代理需高可用。许多平台还会由 Kubernetes Service 或 Service Mesh 提供发现与转发；不要同时叠加多个客户端/代理层而不明确每层的职责，否则会出现两次选址、重试次数放大和排障困难。
### 5.2 面试问答
**Q1：客户端负载均衡的原理？** A：调用方从 `ServiceInstanceListSupplier` 拿到实例列表（来自注册中心并本地缓存）→ 按策略选出实例 → 直接发 HTTP 请求；框架（Feign/RestTemplate）通过 `@LoadBalanced` 拦截器把服务名替换为真实 host:port。**没有中间代理，少一跳**，但每个实例都要有自己的负载策略与缓存。
**Q2：为什么 Ribbon 被替换？** A：Ribbon 长期处于维护模式、与 Reactor/WebClient 的异步模型不匹配、且 Netflix 组件整体退场（Hystrix/Archaius/Zuul 同样）。Spring Cloud LoadBalancer 是官方基于 Reactor 的实现，与 Gateway、WebClient、Feign 一致。
**Q3：怎么实现同机房优先/灰度路由？** A：在实例元数据（Nacos 的 metadata）打标（`zone`、`version`、`gray=true`），自定义 `ServiceInstanceListSupplier` 过滤 + 选择策略；配合请求头（如 `X-Gray: true`）从上下文传入过滤条件。Gateway 层也可按权重断言（Weight）做灰度。
**Q4：负载均衡缓存导致"下线实例还在被调用"怎么办？** A：① 缩短 `spring.cloud.loadbalancer.cache.ttl`（默认 35s 量级，以官方文档为准）；② 服务端优雅停机（先注销实例、等待缓存刷新、再停止接收请求）；③ 关闭缓存（`spring.cloud.loadbalancer.cache.enabled=false`，会增加注册中心压力）；④ 客户端快速失败 + 重试；⑤ 部署期滚动发布而不是同时全下线。
### 5.3 高频追问与陷阱
- 陷阱：`@LoadBalanced` 只能作用在被它标注的 `RestTemplate`/`WebClient.Builder` 上，自己 new 的 RestTemplate 不会走负载均衡。
- 追问：Gateway 默认也使用 Spring Cloud LoadBalancer 做 `lb://` 转发，因此 LB 缓存与健康检查行为同样影响网关。
### 5.4 代码/配置示例
```java
@Configuration
public class LbConfig {
    @Bean @LoadBalanced
    public RestTemplate restTemplate() { return new RestTemplate(); }   // 支持 http://order-service/...
}
```
`@LoadBalanced` 的拦截器只作用于这个 Spring Bean；普通 `new RestTemplate()` 不会自动把服务名解析为实例。生产代码应复用配置好的 HTTP 客户端，设置连接池、连接/读取超时，并根据所用 Cloud 版本接入对应 LoadBalancer starter。自定义灰度选择器需要实现 Spring Cloud LoadBalancer 扩展接口，不是上例中能直接复制的一个类。
```yaml
spring: { cloud: { loadbalancer: { cache: { ttl: 10s } } } }   # 缩短缓存，降低下线实例被调用窗口
```
### 5.5 一句话速记
Spring Cloud 用客户端负载均衡（Ribbon → LoadBalancer）：本地实例缓存 + 策略选择 + 直连；缓存 TTL 决定"下线实例被调用"的窗口。

## 6. OpenFeign 原理与配置
### 6.1 核心原理
**Feign 原理**：`@FeignClient` 接口在启动时被扫描（`@EnableFeignClients` + `FeignClientFactoryBean`）→ 为每个接口生成 **JDK 动态代理** → 调用时由 `Contract` 解析注解得到方法元数据 → `RequestTemplate` 构造请求（编码器 `Encoder` 序列化参数）→ 通过 `Client`（默认 `LoadBalancerFeignClient`）用负载均衡选择实例并发送 → 响应由 `Decoder` 反序列化 → 返回。

**Feign 调用并非 Java 本地调用**：接口代理把方法参数转换成 HTTP 请求，网络序列化、DNS/服务发现、连接池、远端超时、错误状态码和版本兼容仍然存在。OpenFeign 与 Spring Cloud LoadBalancer 的集成通常需要引入对应 LoadBalancer starter；若没有该集成或没有注册发现能力，不能仅凭 `@FeignClient(name="...")` 就假设服务名一定能被解析。也可以把 `url` 指向固定地址，但这会绕过按服务名发现实例的流程。

**同步调用还是消息协作**：Feign/WebClient 等同步 HTTP 调用适合需要立即得到结果、调用链短且有明确超时预算的查询或命令；消息队列适合允许最终一致、可异步处理、需要缓冲突发或解耦发布节奏的场景。同步多级调用会把延迟相加并把故障沿链传播；异步则引入重复投递、乱序、延迟可见和补偿流程。无论哪种，接口都要有明确的请求/响应契约、错误语义、超时/截止时间、鉴权与版本兼容策略。

**Feign 错误处理**：非 2xx 响应可能经过 `ErrorDecoder` 转换为异常；不要把所有异常都转成空对象，否则上游会把故障当成功。Fallback 应显式区分可降级的非核心数据和必须失败的核心命令，并记录原始异常/下游状态。幂等请求重试可以使用退避和抖动；非幂等写请求只有在服务端接受同一幂等键并可返回原结果时才考虑重试。

| 配置项 | 作用 | 注意 |
| --- | --- | --- |
| `spring.cloud.openfeign.client.config.<name>.connectTimeout/readTimeout` | 连接/读取超时 | Cloud 4 常见前缀；旧版属性前缀不同，默认值及单位需查对应版本文档 |
| `spring.cloud.openfeign.client.config.<name>.loggerLevel` | 日志级别 | 还需为 Feign 接口包开启 DEBUG；FULL 可能泄露敏感信息 |
| `spring.cloud.openfeign.okhttp.enabled` / `httpclient.hc5.enabled` | 选择 OkHttp / Apache HC5 客户端 | 具体默认 HTTP Client 由 Cloud 版本与 classpath 决定，不要假定所有版本都使用 JDK 客户端 |
| `spring.cloud.openfeign.compression.request/response.enabled` | 请求/响应压缩 | 压缩会消耗 CPU，适合大报文且链路支持的场景 |
| `spring.cloud.openfeign.circuitbreaker.enabled` | 启用 Spring Cloud CircuitBreaker 集成 | 需要相应实现（如 Resilience4j）；不等同于 Feign 默认自带熔断 |

**重试与幂等**：Spring Cloud OpenFeign 通常配置为默认不重试，但 LoadBalancer、HTTP 客户端、网关或业务代码还可能各自重试，必须检查整条链路而非只看 Feign。**重试会导致同一请求被执行多次，因此只对安全幂等操作开重试；写接口要靠幂等键保护**（见 [SpringMVC 篇](./03-SpringMVC与Web容器.md)）。**拦截器**（`RequestInterceptor`）常用于传递 TraceId、Token、灰度标、租户 ID；异步调用时要显式传播上下文，不能假设 ThreadLocal 自动跨线程。
### 6.2 面试问答
**Q1：Feign 的调用原理？** A：动态代理 + 编解码 + 负载均衡：接口方法调用被代理拦截，按注解生成 HTTP 请求（Encoder 编码），由集成了负载均衡的 Client 选实例发送，再用 Decoder 反序列化响应。**本质是把 HTTP 调用伪装成本地方法调用**，代价是错误处理、超时、重试、序列化都要额外配置。
**Q2：Feign 超时怎么配？为什么历史系统总是 1 秒超时？** A：超时由底层 HTTP Client、Feign/LoadBalancer 以及外层熔断器共同决定，默认值随版本与客户端实现变化，不能照搬某一个默认数。旧系统若启用 Hystrix，Hystrix 的超时可能先于 Feign 的读超时触发，旧版常见约 1 秒配置；Boot 3/现代 Cloud train 不再使用 Hystrix 默认集成，需按所选客户端和 Resilience4j/Sentinel 等实现显式配置连接、读取、总预算与熔断时间。**规范：上游总超时要覆盖下游预算，且留出序列化和返回时间。**
**Q3：Feign 和 RestTemplate 怎么选？** A：Feign 是声明式、接口化、易复用（自带负载均衡、编解码、拦截器），适合服务间大量 RPC；RestTemplate 是编程式、灵活但样板代码多（需 `@LoadBalanced` 才有负载均衡），适合少量调用或需要精细控制的场景。新项目推荐 Feign（或 WebClient 做响应式）。
**Q4：Feign 怎么传递 Token 和 TraceId？** A：实现 `RequestInterceptor`，从 `RequestContextHolder` 取当前请求头，把 `Authorization`、`X-Trace-Id`、`X-Gray` 等写入 Feign 请求模板；异步场景（Hystrix/Sentinel 线程池隔离、@Async）需用 `RequestContextHolder.setRequestAttributes(..., true)` 或 TransmittableThreadLocal 跨线程传递，否则"传不进去"。

**调用链预算示例**：入口总预算 800 ms，网关最多用 700 ms，服务 A 为下游 B 预留 300 ms，B 的 HTTP 读取超时例如 200 ms；这里的数字仅作预算示意，需根据业务 SLO 和网络测量确定。若每一层各自配置 1 秒超时，整条链路可能远超用户预算。重试也消耗同一预算：一次 200 ms 调用加退避后重试不能超出上游 deadline；不要让多个层级各重试 3 次导致最多 3×3×3 次下游请求。
### 6.3 高频追问与陷阱
- 陷阱：`RequestInterceptor` 是全局的，若要在不同 FeignClient 传递不同信息，需按 `@FeignClient(configuration=...)` 隔离配置类。
- 陷阱：feign 日志级别为 FULL 会打印请求体，**可能把敏感信息写进日志**，生产用 BASIC 或 HEADERS。
- 常见坑：未加 `@EnableFeignClients` 或接口不在扫描包内；类路径与接口路径拼接错误；GET 用 POJO 参数需 `@SpringQueryMap`；返回泛型丢失（用具体类型）；未配超时导致线程被拖死；大报文未开压缩与正确 Encoder；接口方法重名导致元数据冲突。
### 6.4 代码/配置示例
```java
@FeignClient(name = "user-service", path = "/api/users",
        configuration = FeignConfig.class, fallbackFactory = UserFallbackFactory.class)
public interface UserClient {
    @GetMapping("/{id}") UserVO get(@PathVariable("id") Long id);
    @PostMapping("/batch") List<UserVO> batch(@RequestBody List<Long> ids);
}
// 按 client 隔离的配置类不要放在主应用组件扫描范围内，否则可能变成全局配置。
public class FeignConfig {
    @Bean
    RequestInterceptor userContextInterceptor() {
        return t -> {
            ServletRequestAttributes a =
                    (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
            if (a == null) return;
            String traceId = a.getRequest().getHeader("X-Trace-Id");
            String authorization = a.getRequest().getHeader("Authorization");
            if (traceId != null) t.header("X-Trace-Id", traceId);
            if (authorization != null) t.header("Authorization", authorization);
        };
    }
}
```
```yaml
spring:
  cloud:
    openfeign:
      okhttp:
        enabled: true              # 仅在添加兼容 OkHttp 依赖且版本支持时启用
      client:
        config:
          default:
            connectTimeout: 1000
            readTimeout: 3000
            loggerLevel: basic
```
代码使用 Servlet 请求上下文，适用于由 HTTP 请求触发的同步调用；定时任务、消息消费和异步线程没有当前 Servlet 请求，不能从 `RequestContextHolder` 取上下文。生产还应使用受信任的上下文传播机制，并避免从客户端透传未经验证的用户身份头。`fallbackFactory` 需启用相应 CircuitBreaker 集成并自行实现工厂；不应在 fallback 中再次无保护地调用同一个故障服务。
### 6.5 一句话速记
Feign = 动态代理 + Encoder/Decoder + 负载均衡 Client；超时/重试/幂等/拦截器传递上下文是四大配置要点。

## 7. 熔断、限流与降级
### 7.1 核心原理
**雪崩效应**：某个下游变慢 → 调用方线程被占满（同步阻塞）→ 上游服务也变慢 → 故障沿调用链向上扩散，最终整个系统不可用。**防御手段**：超时（第一道防线，必须有）、限流（保护自身不被压垮）、熔断（快速失败，避免持续打挂下游）、降级（返回兜底数据）、隔离（不同依赖互不影响）。

| 维度 | Hystrix（维护模式） | Sentinel（主流） |
| --- | --- | --- |
| 隔离方式 | 线程池隔离（默认）/ 信号量隔离 | 信号量（并发线程数）+ 流控规则 |
| 统计模型 | 滑动窗口（RxJava 实现） | **滑动窗口 LeapArray**（把时间窗切成若干桶统计） |
| 流控能力 | 弱（主要靠信号量/线程池） | 强（QPS/线程数、直接/关联/链路、快速失败/WarmUp/排队） |
| 熔断 | 断路器状态机 | 熔断降级规则（慢调用比例/异常比例/异常数） |
| 动态规则 | 配置较繁琐 | 控制台 + 配置中心持久化，实时生效 |
| 扩展机制 | HystrixCommand/注解 | **责任链 Slot**（NodeSelector → ClusterBuilder → Statistic → Authority → System → Flow → Degrade） |
| 生态 | 已停止维护 | 阿里开源，国内主流，Spring Cloud Alibaba 集成 |

**Sentinel 关键概念**：资源（被保护的代码/接口）、规则（流控、熔断、热点、系统、授权）、Slot 责任链（每个 Slot 做一件事，`FlowSlot` 做流控、`DegradeSlot` 做熔断）、**滑动窗口**（默认 1s 窗口分 2 个 500ms 桶，`sampleCount` 与 `windowLengthInMs` 可配，桶越细越平滑）、**熔断状态机**（CLOSED → 达到阈值 OPEN → 经过熔断时长进入 HALF_OPEN 放行探测 → 成功则 CLOSED，失败则回 OPEN）。

**流控规则三要素**：阈值类型（QPS 或并发线程数）、流控模式（直接/关联/链路）、流控效果（快速失败/预热 Warm Up/匀速排队）。**熔断降级规则**：慢调用比例、异常比例、异常数（均需设置最小请求数与统计窗口）。**集群流控**：Token Server（独立或嵌入模式，`embedded` 由某个实例承担）+ Token Client 上报与批量取 token，解决单机阈值不均衡问题。**规则持久化**：默认规则存内存（重启丢失），生产用 **Nacos/ Apollo/ZooKeeper 数据源 + push 模式**，由控制台写入配置中心、客户端监听变更实时更新。

**几种保护措施解决不同问题**：超时给一次等待设置上限；并发隔离限制某个依赖占用的线程/连接；限流限制进入速率；熔断在依赖持续失败或变慢时暂时停止调用；降级决定失败时返回什么。它们不能互相替代：只设置熔断而没有超时，正在执行的慢请求仍可能一直占住线程；只设置限流但没有排队上限，内部任务队列仍可能堆积；fallback 也不能把未知错误伪装成成功订单。

**熔断与恢复要防抖**：熔断统计至少要有最小样本数，窗口、慢调用阈值和开路时长要按业务延迟分布设置；流量恢复时可通过 HALF_OPEN 探测、预热或逐步放量避免瞬间冲击。告警要同时看拒绝数、慢调用比例、熔断状态、线程池/连接池饱和和恢复情况。具体熔断状态细节因 Resilience4j、Sentinel 等实现而异，通用三态只是便于理解的模型。

**组件不是必选项**：Spring Cloud CircuitBreaker 是抽象层，可对接 Resilience4j 等实现；Sentinel 是另一套流控/熔断治理方案，Spring Cloud Alibaba 提供集成。Hystrix 属于历史方案，现代 Spring Cloud 版本不再将它作为默认组件。应选一个主要治理实现并核对版本兼容，不要让多个熔断器在多层叠加而不清楚谁先拒绝、谁触发 fallback。
### 7.2 面试问答
**Q1：Hystrix 的线程池隔离和信号量隔离有什么区别？** A：线程池隔离给每个依赖独立线程池，超时可直接中断（依赖阻塞不影响调用方线程），但线程切换有开销、线程数需分配；信号量隔离在原线程上加计数，开销小、无切换，但**不能中断超时**（只限制并发数）。默认线程池隔离，对延迟敏感/调用量极大的场景可用信号量隔离。
**Q2：熔断器的三个状态与转换条件？** A：CLOSED（正常放行并统计）→ 统计窗口内错误率/慢调用比例/异常数超过阈值 → OPEN（快速失败，不再调用下游）→ 经过熔断时长后进入 HALF_OPEN（放行少量探测请求）→ 探测成功则 CLOSED，失败则回到 OPEN。**关键是阈值来自统计窗口，而非瞬时值。**
**Q3：限流和熔断的区别？** A：限流是**主动限制进入的流量**（保护自身，按 QPS/并发数拒绝超额请求）；熔断是**当被调方异常时快速失败**（保护调用方与下游，避免雪崩）。二者常配合：网关/入口限流 + 服务间熔断降级 + 下游并发控制。
**Q4：Sentinel 规则重启丢失怎么办？** A：默认规则在内存里，重启即失效。生产方案是接配置中心（Nacos）做**持久化 + 推送**：应用通过 `spring.cloud.sentinel.datasource.*` 配置 Nacos 数据源，控制台规则写入 Nacos，客户端监听变更实时刷新。**注意要区分"控制台改了但只存在内存"与"改了 Nacos 但客户端没监听"两种失败模式。**
### 7.3 高频追问与陷阱
- 陷阱：只配限流不配超时——超时是第一道防线，没有超时的限流挡不住线程堆积。
- 陷阱：熔断降级的 fallback 里再调用同一个下游（或未做异常兜底）会形成二次雪崩。
- 追问：`blockHandler`（限流/熔断时执行）与 `fallback`（业务异常时执行）语义不同，异常类型不匹配会导致兜底不生效。
- 降级要点：返回兜底数据（缓存/默认值/空列表）而非异常栈；区分核心与非核心链路（非核心可直接关闭）；写接口降级要慎重（用 MQ 或本地存储补偿）；开关可动态配置；必须有监控与恢复策略。
### 7.4 代码/配置示例
```java
@SentinelResource(value = "getUser", blockHandler = "blocked", fallback = "fallback")
public UserVO get(Long id) { return userClient.get(id); }
public UserVO blocked(Long id, BlockException e) { return UserVO.empty("限流"); }
public UserVO fallback(Long id, Throwable t) { return UserVO.empty("降级"); }
```
此示例表达语义：`blockHandler` 处理 Sentinel 规则拒绝（方法签名需包含匹配参数及 `BlockException`），`fallback` 处理业务执行异常；真实方法和兜底方法需符合所用 Sentinel/SCA 版本的签名要求。确保请求经过 Sentinel 的代理/资源埋点，且方法不是通过同一对象的自调用绕过拦截；若自调用、异步或 WebFlux 场景，应按对应集成方式创建资源并传播上下文。兜底返回“限流”也不应让客户端误以为写操作已成功。
```yaml
spring:
  cloud:
    sentinel:
      transport: { dashboard: 127.0.0.1:8858 }
      datasource:
        flow:
          nacos: { server-addr: 127.0.0.1:8848, data-id: sentinel-flow-rules.json, rule-type: flow }
```
### 7.5 一句话速记
超时 → 限流 → 熔断 → 降级 → 隔离；Sentinel 用滑动窗口 + Slot 责任链，规则必须持久化到 Nacos。

## 8. 网关
### 8.1 核心原理
**Gateway 三大组件**：Route（路由：id + uri + predicates + filters）、Predicate（断言，决定是否匹配该路由）、Filter（过滤器，改写请求/响应）。经典的 Spring Cloud Gateway Server 基于 **Spring WebFlux + Reactor Netty**；较新版本还提供基于 Servlet MVC 的 Gateway Server MVC 变体，运行模型与可用过滤器/属性不完全相同。本文的 `GlobalFilter`、`lb://` 等示例针对 Reactive Gateway，实际项目先确认所选 starter 和版本，不能把两种实现混为一谈。

**路由处理顺序**：请求先由 `RoutePredicateHandlerMapping` 找到所有断言匹配的路由，多个候选按 order 等规则决定；随后全局过滤器与路由过滤器按 order 组成过滤器链。前置过滤器可鉴权、改路径、加追踪头；路由过滤器选择目标 URI 并转发；后置逻辑处理响应、指标或审计。`lb://service-name` 表示通过 Spring Cloud LoadBalancer 解析服务名；`http://host:port` 则是固定目标，不经过服务发现。路由匹配成功不代表下游一定可用，转发仍需要超时与错误处理。

**响应式不等于代码自动非阻塞**：Reactor Netty 的少量事件循环线程负责许多连接；若在其中同步访问 JDBC、调用阻塞式 HTTP 客户端、做长时间 CPU 运算，会占住事件循环并让大量连接一起变慢。优先使用响应式客户端；必须调用阻塞 API 时要在适当边界交给有界 `boundedElastic` 等专用执行器，并限制并发。只加 `subscribeOn` 不能把阻塞工作“变成非阻塞”，还可能导致无界线程/队列压力。

**网关是边界层，不应代替业务授权**：网关适合做统一路由、粗粒度认证、限流、头部清理、TLS 后的安全上下文传递；服务仍要校验资源归属和业务权限，特别是存在绕过网关的内部访问路径时。网关注入身份头前应移除客户端同名伪造头，并通过网络策略/mTLS/签名等限制下游只接受可信来源。每个下游要保留自己的超时、隔离和权限校验。

| 过滤器/断言 | 作用 |
| --- | --- |
| Path / Method / Host / Query / Header / Cookie | 按路径、方法、域名、参数、头、Cookie 匹配 |
| After / Before / Between | 时间窗口匹配（灰度、活动开关） |
| Weight | 按权重分流（灰度发布常用） |
| StripPrefix / RewritePath | 去前缀/重写路径（剥离 `/api` 前缀转发到服务） |
| AddRequestHeader / AddResponseHeader / RemoveRequestHeader | 增删转发头（注入用户身份、TraceId） |
| RequestRateLimiter | 基于 Redis + Lua 的令牌桶限流（key 可自定义，如按用户/接口） |
| CircuitBreaker / Retry | 集成断路器与重试（重试要幂等） |
| DedupeResponseHeader | 去掉重复响应头（解决 CORS 头重复） |
| GlobalFilter（自定义） | 鉴权、日志、灰度标、响应包装 |

**与 Zuul 1.x 的区别**：Zuul 1.x 基于 Servlet（阻塞式，一请求一线程），不适合长连接/WebSocket 与高并发，且已不再维护（Zuul 2 未被 Spring Cloud 集成）；Gateway 基于 Reactor 非阻塞、支持长连接与更好的资源利用率，是当前唯一推荐方案。**网关高可用**：多实例 + LB/VIP + K8s Service，网关本身要无状态（限流计数依赖 Redis、鉴权依赖 JWT/Redis），并注意**不要把网关做成新的单点**（网关故障等于全站故障，需要容量冗余与降级方案，如静态兜底页）。
### 8.2 面试问答
**Q1：Gateway 的路由是怎么工作的？** A：请求进入 `RoutePredicateHandlerMapping`，遍历所有 Route 的 Predicate 组合，全部匹配才选中该路由（按 order 优先），然后依次执行该路由的 GatewayFilter 与全局 GlobalFilter 链，最后由 `NettyRoutingFilter` 通过 WebClient 转发（`lb://` 时用 Spring Cloud LoadBalancer 选实例）。
**Q2：网关怎么做鉴权？** A：在 GlobalFilter 里校验 JWT/Token（白名单路径放行），校验通过后把用户信息（userId、租户、角色）**注入请求头**转发给下游，下游信任网关注入的头（**内网服务必须禁止外部直连，否则头可伪造**）。也可用 OAuth2 资源服务器模式（Spring Security + Gateway）统一拦截。
**Q3：Gateway 限流怎么做？** A：用 `RequestRateLimiter` 网关过滤器 + `RedisRateLimiter`（令牌桶，基于 Redis + Lua 保证原子性），key 解析器可选 `PrincipalNameKeyResolver`、IP、接口路径或自定义（如 userId + 接口）。局限：需要 Redis（本身要高可用），且是**令牌桶**模型（允许突发）；如需集群精确限流可用 Sentinel 网关流控或集群流控。
**Q4：动态路由怎么实现？** A：① 配置中心驱动——从 Nacos 读取路由配置（`spring.cloud.gateway.routes` 或自定义 RouteDefinitionLocator），监听变更后刷新（需在配置变更时发布 `RefreshRoutesEvent`）；② 用 Actuator 的 `/actuator/gateway/routes` 与 `RefreshRoutesEvent` 做在线增删（生产建议再包一层管理接口 + 权限控制）；③ 数据库存路由表 + 定时/事件刷新 + 本地缓存。
### 8.3 高频追问与陷阱
- 陷阱：在 Gateway 里做阻塞式操作（JDBC 查询、同步 HTTP 调用）会卡死 Netty 事件循环——必须用响应式 API 或 `subscribeOn(Schedulers.boundedElastic())`。
- 陷阱：`StripPrefix` 与下游 `context-path` 叠加导致 404，需要用 `/actuator/gateway/routes` 或日志确认最终转发路径。
- 追问：跨域在网关解决时注意不要与下游应用的 CORS 头重复（用 `DedupeResponseHeader` 或只在网关加）。
- 与 Nginx 的关系：Nginx 更靠前（四层/七层入口、TLS、静态资源、粗粒度限流），Gateway 更贴近业务（路由、鉴权、灰度、聚合）；生产常见链路是 `客户端 → LB/VIP → Nginx → Gateway → 微服务`，CORS 等职责只在一层实现。
### 8.4 代码/配置示例
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates: [ "Path=/api/user/**" ]
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Gray, false
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
```
此处省略 `key-resolver`，Gateway 使用其默认 Principal 名称解析器；这适合已有认证主体的请求，匿名请求需要明确设计限流 key。若改为自定义 IP resolver，只有在可信代理正确清洗/写入转发头的前提下才能使用客户端 IP，不能盲信外部传来的 `X-Forwarded-For`。`Weight` 断言则需要同一 weight group 中存在多个可比较路由；只配一条权重路由不能形成 90/10 灰度。
```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {
    @Override public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();
        if (path.startsWith("/api/public")) return chain.filter(exchange);   // 白名单
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (!jwtService.valid(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        ServerHttpRequest req = exchange.getRequest().mutate()
                .headers(h -> { h.remove("X-User-Id"); h.set("X-User-Id", jwtService.userId(token)); })
                .build();    // 覆盖客户端伪造值后注入身份
        return chain.filter(exchange.mutate().request(req).build());
    }
    @Override public int getOrder() { return -100; }   // 越早越好
}
```
### 8.5 一句话速记
Gateway = Route + Predicate + Filter（WebFlux 非阻塞）；鉴权注入头 + Redis 令牌桶限流 + 配置中心动态路由，注意别阻塞事件循环。

## 9. 链路追踪
### 9.1 核心原理
**核心概念**：Trace（一次完整请求）、Span（一次调用单元，含开始/结束时间、父子关系）、TraceId（全局唯一）、SpanId/ParentSpanId（构成调用树）、Baggage/标签（附加上下文）。**传播方式**：通过 HTTP 头（B3 或 W3C TraceContext `traceparent`，MQ 通过消息属性）在服务间传递，所以**任何漏传上下文的地方（自研 HTTP 客户端、异步线程、MQ、定时任务）都会断链**。

**技术演进**：Spring Cloud Sleuth（Brave/Zipkin 生态）在 Boot 3 / Cloud 2022+ 被 **Micrometer Tracing** 取代（Observability API + Brave 或 OpenTelemetry bridge），日志中自动填充 traceId/spanId（配合 `logging.pattern.level` 输出）；采集端可用 Zipkin、Tempo、Jaeger、SkyWalking、OTLP Collector。

**SkyWalking 无侵入方案**：基于 Java Agent（字节码增强）自动埋点，业务零改动即可获得拓扑图、慢 SQL、JVM 指标、告警；适合"多语言/老系统统一观测"。代价是 Agent 与版本兼容、采样与开销需要评估，且深度定制（业务标签）仍需手动埋点。**选择建议**：新项目用 Micrometer Tracing + OTLP（标准、可插拔）；已有 SkyWalking 体系或需要无侵入能力则继续用 SkyWalking。**告警**：追踪数据要落到 APM/日志平台并配置告警（错误率、P99 延迟、慢 SQL、依赖拓扑异常），"只有链路图没有告警"等于没做可观测性。

**三类观测数据各自回答不同问题**：Metrics 是聚合计数/分布，回答“错误率和 P99 是否变坏”；Logs 记录离散事件，回答“某次请求具体发生了什么”；Traces 连接同一请求的跨服务 Span，回答“耗时花在哪一跳”。TraceId 可放入日志上下文便于关联，但不要把高基数 TraceId 当作 Metric label；也不要把密码、完整 Token、身份证等敏感值放进 Span 标签或 baggage。采样会丢弃部分 Trace，因此容量告警和 SLO 统计必须依赖 Metrics，而不是只依赖追踪平台。

**采样与上下文传播的生产边界**：概率采样通常在入口决定，并在跨服务时传播采样标记，避免每个服务各采各的造成链路断层。异步线程、Reactor Context、消息队列和定时任务需要各自的上下文桥接；TraceId 是关联标识，不是认证凭据，不可用来授权。Micrometer Tracing 的 bridge、传播格式（W3C/B3）及导出协议由依赖与版本决定，迁移 Sleuth 时需验证上下游格式兼容和日志字段名称。
### 9.2 面试问答
**Q1：链路追踪的原理？** A：入口生成/继承 TraceId，每次跨进程调用创建一个 Span 并把 TraceId + ParentSpanId 通过 header 传出，下游继续上面的父 Span 形成调用树；采集端聚合 Span 还原调用链、耗时与依赖拓扑。**关键是上下文传播 + 采样**（高流量下全量采集不现实）。
**Q2：线上怎么快速定位一个慢请求？** A：从入口日志拿到 TraceId（或从用户投诉的请求 ID）→ 在 APM/日志平台按 TraceId 检索 → 看各 Span 耗时找出占比最大的环节 → 若是 DB 则看慢 SQL 与执行计划，若是 RPC 则看下游与连接池，若是本地代码则看线程栈与 GC。**前提是 TraceId 全链路透传，包括 MQ 与异步线程。**
**Q3：异步与 MQ 场景 TraceId 为什么会丢？** A：因为 MDC/TraceContext 存在 ThreadLocal（或 Reactor Context）里，跨线程不自动传递。解法：线程池用 `TaskDecorator`/TransmittableThreadLocal 透传；MQ 生产端把 TraceId 写进消息属性、消费端取出后重新放入上下文；Reactor 场景用 `contextWrite` 传播。
**Q4：采样率怎么定？** A：按流量与成本：全量采集只适合低压系统；高流量用概率采样（如 1%~10%）+ **关键链路强制采样**（错误请求、VIP 用户、支付链路 100% 采样）；采样策略要统一（入口决定）。**注意：限流/统计类指标不要依赖追踪数据，应该用独立的 Metrics 体系。**
### 9.3 高频追问与陷阱
- 陷阱：只加依赖不传上下文（自研 RestTemplate、Socket、自定义 MQ）导致链路断开，画出来的拓扑是碎的。
- 追问：Sleuth 已停止演进，Boot 3 项目请用 Micrometer Tracing；升级时要同步替换日志 pattern 与传播格式（B3 → W3C，或显式配置保持兼容）。
### 9.4 代码/配置示例
```yaml
management:
  tracing:
    sampling: { probability: 0.1 }        # 10% 采样（Micrometer Tracing）
  zipkin:
    tracing: { endpoint: http://127.0.0.1:9411/api/v2/spans }
logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```
### 9.5 一句话速记
Trace 由 TraceId + Span 树构成，靠 header（B3/W3C）传播；串不起来先查异步/MQ/自研客户端是否漏传上下文。

## 10. 服务治理
### 10.1 核心原理
**优雅上下线**（滚动发布不丢请求）：① 上线——实例启动后先不接收流量，健康检查通过（Readiness）再注册/加权；② 下线——先摘流量（从注册中心注销/权重置 0）→ 等待 LB 缓存刷新与在途请求完成（`server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase`）→ 再关闭进程；K8s 用 `preStop` + `terminationGracePeriodSeconds`。

**超时与重试的全局设计**：**从上到下超时递减**（客户端总超时 > 网关超时 > 上游服务超时 > 下游 RPC 超时 > DB/缓存超时），且重试次数与超时互相约束（重试总时长不能超过上游超时，否则上游先断开）。**重试只对幂等操作开放**，写接口用幂等键 + 最多一次（或改为异步确认）。

| 治理项 | 关键做法 |
| --- | --- |
| 灰度发布 | 网关按 header/权重路由 + Nacos 元数据 + 自定义 LoadBalancer；灰度期间必须有对比指标与快速回滚 |
| 全链路压测 | 压测标（header）透传 + 影子库/影子表/影子 topic 隔离数据；禁用外部真实副作用 |
| 依赖治理 | 梳理调用拓扑、限制调用深度（≤3~4 层）、消除循环依赖、关键依赖做多活/降级 |
| 容量与限流 | 基于压测定容量、网关 + 应用双层限流、线程池隔离、排队与背压 |
| 变更安全 | 配置与代码同流程（审批、灰度、可回滚）、开关中心、应急预案与演练 |
### 10.2 面试问答
**Q1：发布时为什么会有一批请求失败？怎么做到零丢失？** A：因为"实例摘除"与"客户端 LB 缓存刷新"之间有时间差，以及进程在途请求被中断。要做到接近零丢失：注册中心权重/流量先摘除 → 等待若干秒（≥ LB 缓存 TTL）→ 优雅停机等待在途请求完成 → 关闭；同时客户端对幂等请求重试，服务端对写请求做幂等。完全零丢失还需要客户端侧的重试与业务兜底。
**Q2：超时怎么设计？** A：先定"用户可接受的最大响应时间"（如 500ms）→ 自上而下分配预算（网关 500ms、服务 A 200ms、下游 B 100ms、DB 50ms）→ 每层显式配置并**小于**上层，禁止无限等待；对不可控的第三方依赖单独设更短超时 + 熔断。**没有超时设计的系统一定会雪崩。**
**Q3：重试应该注意什么？** A：① 只重试幂等操作；② 限制次数（1~2 次）与总时长；③ 避免重试放大（下游已过载时重试会加剧故障，应配熔断与退避）；④ 失败要可观测（统计重试次数）；⑤ 不要在多层同时重试（重试次数相乘）。
**Q4：灰度发布怎么做更安全、全链路压测要注意什么？** A：灰度：入口按权重/标头路由（Gateway Weight/Header 断言）+ 实例按元数据打标并用自定义 LB 路由 + 数据扩展式变更（加字段、双写、回填、切读）+ 指标对比与自动回滚，范围从内部用户到 1% 流量逐步放量。压测：压测流量打标并全链路透传、数据隔离（影子库/表/topic）、屏蔽真实外部副作用（支付/短信 mock）、常态演练验证限流降级——**切忌直接压生产主库**。
### 10.3 高频追问与陷阱
- 陷阱：只在网关做限流，服务间调用直接打爆下游——服务自身也要有并发/限流保护。
- 陷阱：重试 + 无幂等 = 重复下单/重复扣款，这是最贵的线上事故之一。
### 10.4 代码/配置示例
```yaml
server:
  shutdown: graceful
spring:
  lifecycle: { timeout-per-shutdown-phase: 30s }
  cloud:
    loadbalancer:
      cache: { ttl: 5s }      # 缩短缓存，缩短下线窗口
# K8s 侧：先摘流量再停机（preStop sleep + gracePeriod > 停机超时）
# lifecycle.preStop.exec.command: ["sleep","10"]；terminationGracePeriodSeconds: 45
```
### 10.5 一句话速记
优雅上下线 = 先摘流量、等缓存刷新、再优雅停机；超时自上而下递减、重试只对幂等接口开放。

## 11. 微服务线上问题排查
### 11.1 核心原理

| 现象 | 常见根因 | 排查手段 |
| --- | --- | --- |
| 调用超时 | 下游慢（慢 SQL/GC/锁）、超时配置缺失或过大、线程池打满、网络/带宽、注册到坏实例 | 链路追踪定位耗时环节、`/actuator/metrics`、线程栈、慢 SQL 日志、连接池指标 |
| 熔断频发 | 下游错误率超阈值、超时阈值设置过紧、下游发布导致抖动 | 熔断器指标 + 下游错误日志 + 依赖拓扑，先降级止损再定位 |
| 注册中心抖动 | 网络分区（心跳丢失）、GC 长停顿、实例被误剔除、自我保护触发 | 注册中心控制台看实例健康与续约曲线，应用侧看 GC/网络 |
| 线程池打满 | 下游无超时、同步调用过多、限流缺失、任务量突增 | 线程 dump（jstack/Arthas thread）、线程池指标、是否集中阻塞在同一依赖 |
| 配置不一致 | 刷新未生效、实例版本不同、本地缓存快照、配置中心集群脑裂 | 对比 `/actuator/env` 与配置中心、检查 `@RefreshScope`、实例元数据版本 |
| 偶发 500 | 序列化失败、泛型丢失、脏实例、连接池瞬时耗尽 | 按 TraceId 找日志、看是否集中在少数实例 |
| 重复下单/重复扣款 | 重试 + 非幂等、MQ 重复消费 | 查重试配置、幂等键、消费端去重表 |

**排查方法论**：先止损（限流/降级/回滚）→ 再定位（链路追踪 + 指标 + 日志 + 线程栈四件套）→ 最后修复与复盘（补充超时/幂等/监控）。**没有可观测性的微服务排障只能靠猜**，所以 Metrics + Tracing + Logging 必须在架构阶段就落地。
### 11.2 面试问答
**Q1：线上大面积接口超时，你怎么处理？** A：① 先看范围（全站还是某个依赖）：看监控大盘与链路追踪确认瓶颈层；② 立即止损：对故障依赖限流/熔断降级、必要时回滚最近变更；③ 定位：线程栈看是否集中阻塞、慢 SQL/GC 日志看资源侧、注册中心看实例健康；④ 修复并加固（补超时、加熔断、扩容、优化慢 SQL）；⑤ 复盘沉淀预案与演练。
**Q2：为什么服务启动后调用偶发失败，过一会儿就好了？** A：典型的"上线窗口"问题：实例刚注册但尚未完全就绪（Ready 检查缺失）、LB 缓存里还留着已下线实例、下游连接池/预热未完成（首次请求慢）。解法：Readiness 探针 + 启动预热 + 缩短 LB 缓存 TTL + 优雅上下线流程。
**Q3：线程池打满怎么定位是谁的原因？** A：`jstack`/Arthas `thread -n 10` 看阻塞最多的调用栈；如果集中在某个 HTTP/RPC 调用，说明该依赖无超时或超时过长；如果集中在 DB，看连接池等待与慢 SQL；同时检查是否有同步调用链路过长（应异步化/并行化）。**结论通常是"缺少超时 + 缺少隔离"而不是"线程数不够"。**
**Q4：配置改了但只有部分实例生效，怎么查？** A：① 检查这些实例是否连到同一 namespace/group/dataId；② 检查是否加了 `@RefreshScope`（或被注入的 Bean 没刷新）；③ 看实例 `/actuator/env` 的实际值；④ 检查是否存在配置文件与配置中心同名 key 的覆盖关系（优先级）；⑤ 检查配置中心集群是否健康、是否有本地快照干扰。
### 11.3 高频追问与陷阱
- 陷阱：只看应用日志不看指标——"错误率/延迟/饱和度/流量"四类指标才是快速定位的关键。
- 追问：把 TraceId 打进所有日志（含异步/MQ）是微服务排障的基础设施要求。
### 11.4 代码/配置示例
```bash
# 常用排障命令（K8s 环境）
kubectl exec -it pod -- jcmd 1 Thread.print | grep -A 20 "http-nio"
kubectl exec -it pod -- curl -s localhost:9090/actuator/metrics/tomcat.threads.busy
kubectl exec -it pod -- curl -s localhost:9090/actuator/loggers/com.x.service   # 动态调级别
```
### 11.5 一句话速记
排障顺序：止损（限流/降级/回滚）→ 定位（追踪 + 指标 + 日志 + 线程栈）→ 加固（超时、幂等、隔离、告警）。

## 自测题（含答案要点）
1. **微服务拆分的原则与代价？** 单一职责、限界上下文、数据私有；代价是分布式事务、跨服务查询、链路变长与运维成本。
2. **Spring Cloud 版本列车的对应关系？** 2020.0→Boot 2.4/2.5、2021.0→2.6/2.7、2022.0→3.0/3.1、2023.0→3.2/3.3；不对齐会出现运行期 NoSuchMethodError，用 BOM 收敛。
3. **Eureka 自我保护机制？** 续约率低于阈值时停止剔除实例，防网络分区误杀，副作用是可能调用到已死实例。
4. **Nacos 是 AP 还是 CP？** 按实例类型：临时实例 AP（Distro），持久实例 CP（Raft）。
5. **注册中心挂了会怎样？** 存量调用可继续（本地缓存），新注册/摘除/配置推送失效。
6. **Nacos 配置模型与优先级？** namespace > group > dataId；主配置 > extension-configs > shared-configs。
7. **动态刷新原理与 @Value 不刷新的原因？** 长轮询 + RefreshEvent + 重建 @RefreshScope Bean；普通 Bean 不重建所以 @Value 不变。
8. **Ribbon 为什么被替换？** 维护模式 + 与响应式模型不匹配；改用 Spring Cloud LoadBalancer（客户端负载均衡：本地缓存 + 策略选择 + 直连）。
9. **Feign 的原理？** 动态代理 + Contract 解析 + Encoder/Decoder + 负载均衡 Client。
10. **Feign 为什么总是 1 秒超时？** 启用 Hystrix 时其默认 1s 熔断超时先触发；必须显式配置超时。
11. **Feign 怎么传 Token/TraceId？** RequestInterceptor 从 RequestContextHolder 取头写入模板，异步需跨线程传递。
12. **雪崩效应与五种防御？** 超时、限流、熔断、降级、隔离。
13. **Hystrix 线程池隔离与信号量隔离的区别？** 独立线程池可中断超时、开销大 vs 原线程计数开销小但无法中断。
14. **熔断器三态转换？** CLOSED → OPEN（达阈值）→ HALF_OPEN（熔断时长后探测）→ 成功 CLOSED / 失败回 OPEN。
15. **Sentinel 的滑动窗口、责任链与规则持久化？** LeapArray 分桶统计 + Slot 链（统计/系统/流控/熔断）；规则必须在 Nacos 等配置中心持久化。
16. **Gateway 三大组件与 Zuul 区别？** Route/Predicate/Filter；Gateway 基于 WebFlux 非阻塞，Zuul 1.x 是 Servlet 阻塞且已不维护。
17. **网关鉴权怎么把用户身份传给下游？** GlobalFilter 校验后注入请求头，下游信任网关注入（内网不可直连）。
18. **链路追踪的核心概念与断链原因？** Trace/Span/TraceId/SpanId；异步、MQ、自研客户端漏传上下文会断链。
19. **优雅上下线的步骤？** 摘流量 → 等 LB 缓存刷新 → 优雅停机等在途请求 → 关进程。
20. **超时与重试的全局设计？** 超时自上而下递减，重试只对幂等接口且次数有限。
21. **线上大面积超时的处理流程？配置改了只有部分实例生效怎么查？** 止损（限流/降级/回滚）→ 定位（追踪/指标/日志/线程栈）→ 修复加固；配置问题查 namespace/group/dataId 是否一致、是否加 @RefreshScope、看 /actuator/env 实际值、排查配置覆盖与本地快照。

## 背诵卡片
1. 微服务拆分看独立扩容、故障隔离与团队边界，数据必须私有。
2. 分布式单体 = 服务拆了但必须一起发布，是最常见的失败形态。
3. Cloud 版本列车：2020.0 / 2021.0 / 2022.0 / 2023.0 对应 Boot 2.4 / 2.6 / 3.0 / 3.2。
4. 永远用 spring-cloud-dependencies BOM 收敛版本，别手写组件版本。
5. 注册中心 = 注册 + 心跳 + 本地缓存 + 变更推送 + 负载选择。
6. Eureka 默认开启自我保护，续约率低时停止剔除实例。
7. Nacos 临时实例走 AP（Distro），持久实例走 CP（Raft）。
8. Nacos 心跳 5s，15s 无心跳标记不健康，30s 剔除（临时实例）。
9. 注册中心不可用时存量调用可用，新实例注册与配置推送失效。
10. 配置模型：namespace（环境）> group（项目）> dataId（应用-环境）。
11. 动态刷新 = 长轮询 + RefreshEvent + 重建 @RefreshScope Bean。
12. 普通 Bean 里的 @Value 不会随配置刷新，需要 @RefreshScope。
13. 配置变更要走灰度 + 历史版本回滚 + 审计 + 加密。
14. Ribbon 在 Cloud 2020.0 被移除，改用 Spring Cloud LoadBalancer。
15. LB 缓存 TTL 决定"下线实例仍被调用"的时间窗口。
16. Feign = 动态代理 + 编解码 + 负载均衡 Client；默认 connect 10s / read 60s。
17. 启用 Hystrix 时 1s 熔断超时先触发，这是"Feign 总是 1 秒超时"的答案。
18. Feign 生产建议换 OkHttp 连接池并显式配超时；日志用 BASIC。
19. RequestInterceptor 是传 TraceId/Token/灰度标的统一入口。
20. 雪崩防御五件套：超时、限流、熔断、降级、隔离。
21. 熔断三态：CLOSED / OPEN / HALF_OPEN；Hystrix 线程池隔离可中断超时。
22. Sentinel 用滑动窗口统计 + Slot 责任链；流控三要素：阈值类型、流控模式、流控效果。
23. Sentinel 规则默认存内存，生产必须持久化到 Nacos。
24. blockHandler 处理限流熔断，fallback 处理业务异常，别写混。
25. Gateway 三大组件：Route、Predicate、Filter；基于 WebFlux，禁止阻塞操作。
26. Redis 令牌桶限流用 RequestRateLimiter + 自定义 key；网关注入的用户头依赖内网不可直连。
27. Zuul 1.x 是阻塞式 Servlet 模型，已不维护；Gateway 基于 WebFlux，禁止阻塞操作。
29. Trace/Span/TraceId/SpanId 构成调用树，靠 B3/W3C 头传播；Sleuth 已被 Micrometer Tracing 取代，SkyWalking 用 Java Agent 无侵入埋点——采样率按流量定，错误与关键链路 100% 采样。
30. 优雅上下线：摘流量 → 等缓存 → 优雅停机；超时自上而下递减，重试只对幂等接口。
31. 灰度发布要配数据兼容（加字段 + 双写 + 回填 + 切读）；排障四件套是链路追踪、指标、日志、线程栈，先止损后定位。
