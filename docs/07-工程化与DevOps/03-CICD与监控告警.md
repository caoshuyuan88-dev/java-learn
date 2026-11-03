# CI/CD 与监控告警

> **定位**：流水线设计与发布策略 → 制品管理 → 四层监控指标 → Prometheus/Grafana/ELK/链路追踪 → 值班与线上排障模板，打通"从提交代码到线上可观测"的闭环。
> **重要度**：★★★★★（3-5 年社招必问"你们怎么发布/怎么监控/出故障怎么办"；PromQL 与告警设计是区分度最高的一环）
> **建议复习时长**：5 小时（CI/CD 与发布策略 2h + Prometheus 与告警 1.5h + 日志与链路追踪 1h + 排障模板 0.5h）
> **版本基准**：Jenkins 2.4xx LTS、GitLab CI 16+、GitHub Actions、ArgoCD 2.9+、Prometheus 2.45+/3.x、Grafana 10+、Elastic 8.x、OpenTelemetry 1.x。
> **关联文档**：[复习路线总纲](../00-复习路线总纲.md)、[Docker与Kubernetes](./02-Docker与Kubernetes.md)、[Maven-Git与工程实践](./04-Maven-Git与工程实践.md)、[JVM与GC调优](../01-Java基础/04-JVM与GC调优.md)、[Redis高可用与缓存治理](../02-数据库/05-Redis高可用与缓存治理.md)、[消息队列与RocketMQ](../04-中间件/01-消息队列与RocketMQ.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| CI/CD/持续交付概念与流水线设计原则 | ★★★★☆ | 八股原理 / 最佳实践 |
| 主流 CI 工具与 GitOps 选型 | ★★★★☆ | 场景题 |
| Jenkins 架构与声明式 Pipeline | ★★★★★ | 配置实践 / 最佳实践 |
| 制品管理（Nexus/Harbor）与镜像 tag 规范 | ★★★★☆ | 配置实践 |
| 发布策略（滚动/蓝绿/金丝雀/灰度）与回滚 | ★★★★★ | 场景题 / 故障排查 |
| 数据库变更与发布协同 | ★★★★☆ | 场景题 |
| 环境管理与配置隔离 | ★★★★☆ | 配置实践 |
| 监控分层与关键指标清单 | ★★★★★ | 八股原理 / 故障排查 |
| Prometheus 数据模型、PromQL 与告警设计 | ★★★★★ | 命令速查 / 配置实践 |
| 日志体系（ELK/EFK）与链路追踪（OTel） | ★★★★★ | 八股原理 / 最佳实践 |
| 值班、故障响应与线上排查模板 | ★★★★★ | 场景题 / 故障排查 |
## 1. CI/CD 概念、流水线设计原则与工具选型
### 1.1 核心原理
- **CI（持续集成）**：频繁把代码合入主干，每次提交都自动构建 + 跑测试，尽早发现集成问题（对应实践：小步提交、主干开发、快速反馈）。
- **CD 的两种含义**：**持续交付（Continuous Delivery）**= 每次变更都产出可随时上线的制品，但**上线需人工批准**；**持续部署（Continuous Deployment）**= 通过全部门禁后**自动上线**，无需人工。
- **流水线设计八原则**：① 快慢分离（提交后 5-10 分钟出反馈，慢用例放夜间/发布前）；② 质量门禁前置（编译 → 单测 → 静态扫描 → 安全扫描 → 集成测试）；③ **一次构建、多处部署**（同一制品晋级到各环境，禁止每环境重新编译）；④ 幂等可重复（同一 commit 重跑结果一致）；⑤ 一切配置进版本控制；⑥ 失败即停并**快速回滚**；⑦ 构建产物可追溯（tag 带 commit，见 §3）；⑧ 流水线即代码（Jenkinsfile/.gitlab-ci.yml），不走 UI 手工配置。

| 工具 | 定位与优势 | 短板 / 适用 |
| --- | --- | --- |
| Jenkins | 插件生态最全、可自建、适合复杂异构流水线 | 维护成本高、易腐化；适合已有存量体系 |
| GitLab CI | 与仓库/Registry/环境天然一体，`.gitlab-ci.yml` 简洁、Runner 可容器化 | 依赖 GitLab 生态；适合一体化团队 |
| GitHub Actions | 与 PR 流程深度集成、Marketplace 丰富、OIDC 免密钥上云 | 私有 Runner 成本、复杂编排稍弱；开源与云原生团队首选 |
| ArgoCD / Flux（GitOps） | 声明式、**Pull 模式**（集群凭据不出集群）、天然审计与回滚 | 只负责部署同步，构建仍需 CI；K8s 场景必答 |

### 1.2 面试问答
- **Q1：持续交付和持续部署的区别？** A：结论——差在"是否自动上线"；原因是持续交付把"可上线"作为交付物、发布是业务决策（需人工批准与发布窗口），持续部署则全自动；边界是能上持续部署的前提是高质量自动化测试与灰度/一键回滚能力，多数国内公司止步于持续交付（人工点发布按钮）。
- **Q2：为什么强调 build once, deploy many？** A：结论——避免"各环境编译出的制品不一致"导致的环境差异故障；原因是重新编译可能引入依赖版本漂移、时间戳、构建参数差异；边界是配置必须外置（环境变量/配置中心/ConfigMap），而不是编译进包，否则做不到一次构建多处部署。
- **Q3：GitOps 相比 push 式部署的优势？** A：结论——安全、可审计、可回滚、自动纠偏；原因是集群凭据留在集群内（CI 不需持有 kubeconfig）、Git 是唯一事实源、ArgoCD 持续对比并 self-heal；边界是它引入"配置仓库"这一新资产与同步延迟，且密钥不能明文进 Git（需 SealedSecret/External Secrets）。详见 [Docker与Kubernetes](./02-Docker与Kubernetes.md)。
### 1.3 高频追问与陷阱
- 流水线时长是团队工程能力的硬指标：超过 15 分钟会显著降低提交频率，必须做**依赖缓存**（Maven 本地仓库缓存、Docker 层缓存、Gradle Build Cache）与**测试并行**。
- "把发布脚本写在运维文档里"是反模式：不可重复、无法审计，正确做法是脚本入库并由流水线调用。
### 1.4 示例
```groovy
// 门禁分层的典型结构（Jenkins 声明式，简化）
stages { stage('Build') { ... } stage('UnitTest') { ... } stage('Scan') { ... }
         stage('Deploy-STAGING') { ... } stage('E2E') { ... }
         stage('Deploy-PROD') { input { message '确认发布生产?' } ... } }
```
### 1.5 一句话速记
CI 是"每次都集成"，持续交付是"随时能发、人工点发"，持续部署是"自动发"；流水线要**快慢分离、制品一次构建、门禁左移、失败即停**。
## 2. Jenkins：架构与声明式 Pipeline
### 2.1 核心原理
- **架构**：Controller（原 Master）负责调度、UI、配置与插件，**不跑构建**；Agent（原 Slave）执行构建，通过标签（label）选择，推荐用动态容器 Agent（**Kubernetes Plugin** 按需起 Pod、用完销毁），实现隔离与弹性。
- **Job 类型**：Freestyle（界面配置，逐步淘汰）、Pipeline（Jenkinsfile 即代码，主流）、Multibranch Pipeline（按分支自动创建流水线）、Organization Folder（整组织扫描）。
- **声明式 Pipeline 结构**：`pipeline { agent / environment / parameters / options / triggers / stages { stage { steps / when } } / post { always|success|failure|unstable|changed } }`；与之对应的脚本式（`node {}`）更灵活但可读性与约束更差。
- **共享库（Shared Library）**：`@Library('devops-lib@main') _`，`vars/` 放全局步骤（如 `buildJavaApp`）、`src/` 放 Groovy 类，把各项目重复的流水线逻辑收敛（配置 `jenkinsfile` 一行调用即可）。
- **凭据管理**：`credentials('id')` / `withCredentials([string(credentialsId:'nexus', variable:'NEXUS_PWD')])`，底层由 Jenkins 加密存储；**严禁把密钥写进 Jenkinsfile**（会进 Git 与构建日志）。
- **触发方式**：SCM 轮询 `pollSCM('H/5 * * * *')`、定时 `cron`、Webhook（GitLab/GitHub 插件或 Generic Webhook Trigger，首选）、上游 `upstream`。
- **制品归档**：`archiveArtifacts`（长期留存）、`stash/unstash`（同流水线跨 stage/节点传递）、`junit`（测试报告）、`recordIssues`（静态扫描）。
### 2.2 面试问答
- **Q1：为什么不建议在 Controller 上跑构建？** A：结论——构建会抢占 UI/调度资源并带来安全风险；原因是 Controller 持有凭据与插件、一旦被构建脚本越权影响面极大；边界是必须用 Agent（容器 Agent 最佳），并给 Agent 配置带标签的调度与资源限额。
- **Q2：如何让流水线可复用？** A：结论——抽共享库 + 参数化 Jenkinsfile；原因是一个项目一份 300 行 Jenkinsfile 会导致"改一处要改几十个仓库"；边界是共享库自身也要版本化（用 tag 引用而非 master），并提供本地调试方式（`jenkinsfile-runner` 或临时分支验证）。
- **Q3：Jenkins 的凭据在流水线里怎么用才安全？** A：结论——用凭据插件注入变量，并避免回显；原因是 `echo` 或异常堆栈可能把密钥打进日志，且 `sh` 命令会记录命令行参数；边界是敏感值用 `withCredentials` + `set +x`、构建日志脱敏（Mask Passwords 插件），更好的做法是把密钥交给目标平台的 Secret 管理（K8s Secret/Vault），流水线不接触明文。
### 2.3 高频追问与陷阱
- `post { failure { ... } }` 里必须发通知与归档日志，否则失败现场全靠人工复现；`options { timeout(time: 30, unit: 'MINUTES') }` 防止流水线挂死占满 Agent。
- 制品（jar/镜像）要**带构建号或 commit 归档**，否则线上回滚时找不到对应包。
### 2.4 示例（完整的 Java 项目声明式 Pipeline）
```groovy
pipeline {
  agent { kubernetes { yaml '''apiVersion: v1
kind: Pod
spec: { containers: [{ name: maven, image: maven:3.9-eclipse-temurin-17,
        command: ["sleep"], args: ["infinity"], resources: { requests: { cpu: "1", memory: "2Gi" } } }] }''' } }
  options { timeout(time: 30, unit: 'MINUTES'); timestamps(); buildDiscarder(logRotator(numToKeepStr: '20')) }
  parameters {
    choice(name: 'ENV', choices: ['staging', 'prod'], description: '目标环境')
    booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: '跳过单测（仅紧急时勾选）')
  }
  environment { REGISTRY = 'registry.example.com/order'; IMAGE_TAG = "v${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}" }
  triggers { pollSCM('H/5 * * * *') }          // 生产建议改用 Webhook 触发
  stages {
    stage('Checkout') { steps { checkout scm } }
    stage('Build & Test') {
      when { expression { !params.SKIP_TESTS } }
      steps {
        sh 'mvn -B -Dmaven.repo.local=$WORKSPACE/.m2 clean verify'
        junit 'target/surefire-reports/*.xml'
      }
    }
    stage('Package & Image') {
      steps {
        sh 'mvn -B -DskipTests package'
        sh 'docker build -t $REGISTRY/order-service:$IMAGE_TAG .'
        withCredentials([usernamePassword(credentialsId: 'harbor', usernameVariable: 'U', passwordVariable: 'P')]) {
          sh 'echo $P | docker login $REGISTRY -u $U --password-stdin'
          sh 'docker push $REGISTRY/order-service:$IMAGE_TAG'
        }
      }
    }
    stage('Deploy') {
      steps {
        withCredentials([string(credentialsId: 'kubeconfig-prod', variable: 'KC')]) {
          sh 'echo "$KC" > kubeconfig && export KUBECONFIG=$PWD/kubeconfig'
          sh "kubectl -n ${params.ENV} set image deploy/order-service app=$REGISTRY/order-service:$IMAGE_TAG"
          sh "kubectl -n ${params.ENV} rollout status deploy/order-service --timeout=300s" }
      }
    }
  }
  post { success { archiveArtifacts artifacts: 'target/*.jar', fingerprint: true }
         failure { sh 'kubectl -n $ENV rollout undo deploy/order-service' }   // 失败自动回滚
         always  { cleanWs() } }
}
```
### 2.5 一句话速记
Jenkins = **Controller 调度 + Agent 执行（容器 Agent 最佳）+ Jenkinsfile 即代码**；声明式五件套 `agent/environment/parameters/stages/post`，凭据靠 `withCredentials`，制品必须归档、失败必须通知与回滚。
## 3. 制品管理与镜像 tag 规范
### 3.1 核心原理
- **Nexus**：三类仓库——`proxy`（代理中央仓库并缓存，加速与隔离外网）、`hosted`（自研包发布，分 `releases` 与 `snapshots`）、`group`（把多个仓库聚合成一个地址给所有人用）。`mvn deploy` 配合 POM 的 `distributionManagement` 上传，凭据在 `settings.xml` 的 `<servers>`（注意 `id` 必须与 `distributionManagement` 中的 `id` 一致）。
- **快照与正式版**：`1.0.0-SNAPSHOT` 可被重复覆盖（版本号带时间戳），**禁止被生产依赖/上线**；`1.0.0`（release）不可覆盖，必须走新版本。详见 [Maven-Git与工程实践](./04-Maven-Git与工程实践.md)。
- **Harbor**：镜像仓库（项目隔离、RBAC、复制、Trivy 扫描、tag 保留与 GC），生产上线只允许从 Harbor 拉取（配合 K8s `imagePullSecrets`）。
- **镜像 tag 规范（结论）**：`<环境>-<应用>-<语义版本>-<git短SHA>-<构建号>`，例如 `prod-order-service-v1.2.3-a1b2c3d-128`；**禁止 `latest`**，因为它是可变别名、无法追溯与回滚。K8s manifest 里引用不可变 tag（必要时用 digest `@sha256:...` 锁定）。
- **制品晋级**：dev → staging → prod 用的必须是**同一个镜像 digest**，只替换配置（环境变量/ConfigMap），这是"一次构建多处部署"的落地方式。
### 3.2 面试问答
- **Q1：为什么快照版本不能上线？** A：结论——同版本号内容会变，无法复现与回滚；原因是 SNAPSHOT 每次 deploy 都覆盖（Nexus 用时间戳隔离多份），依赖方拿到的可能是不同内容；边界是内部联调可以用 SNAPSHOT，但发布分支必须改成正式版本并冻结。
- **Q2：镜像 tag 用 git SHA 还是语义版本？** A：结论——两者都要，语义版本给人看、SHA 给机器追溯；原因是语义版本可能出现"同版本不同代码"（人为打错），而 SHA 唯一；边界是最终以 digest 为准部署，tag 只做人类可读的索引。
- **Q3：Nexus group 仓库解决什么问题？** A：结论——用一个地址同时代理"自研包 + 中央仓库"；原因是开发者无需在 POM 里配多个 `<repository>`，也方便统一管控外网来源；边界是 group 只是聚合入口，发布仍必须往 hosted 推（group 通常只读）。
### 3.3 高频追问与陷阱
- `distributionManagement` 的 `id` 与 `settings.xml` 的 `server id` 不一致会导致 `401/403`，是最常见的发布报错；CI 环境用 `settings.xml` 模板 + 环境变量注入密码，禁止提交明文。
- 镜像仓库的"保留策略"要留足回滚窗口（如保留最近 30 天 / 最近 20 个 tag），否则会出现"回滚时镜像已被清理"的尴尬。
### 3.4 示例
```xml
<!-- POM：发布目标 -->
<distributionManagement>
  <repository>   <id>nexus-releases</id> <url>https://nexus.example.com/repository/maven-releases/</url> </repository>
  <snapshotRepository> <id>nexus-snapshots</id> <url>https://nexus.example.com/repository/maven-snapshots/</url> </snapshotRepository>
</distributionManagement>
```
```bash
mvn -B clean deploy -DskipTests            # 推送到 Nexus（版本决定进 releases 还是 snapshots）
docker build -t $REG/order-service:v1.2.3-a1b2c3d-128 . && docker push $REG/order-service:v1.2.3-a1b2c3d-128
```
### 3.5 一句话速记
Nexus 管 **proxy/hosted/group**，SNAPSHOT 不上生产；镜像 tag 用 **语义版本 + git SHA**、禁用 `latest`；各环境必须是**同一 digest** 只换配置。
## 4. 发布策略、回滚与数据库变更协同
### 4.1 核心原理
| 策略 | 做法 | 优点 | 代价 / 适用 |
| --- | --- | --- | --- |
| 停机发布 | 停服 → 部署 → 起服 | 简单、无版本共存问题 | 有停机；仅适合内部系统/可停机业务 |
| 滚动发布 | 逐批替换实例（K8s 默认） | 无停机、资源占用小 | 发布期新旧版本共存，需接口兼容；回滚需再滚一次 |
| 蓝绿发布 | 两套环境，LB 一键切换 | 切换秒级、回滚最快 | 需要双倍资源；DB 变更仍要兼容 |
| 金丝雀发布 | 先放少量流量（1%~5%）观察指标再放量 | 风险可控、有数据支撑 | 需要流量治理与自动化指标判定 |
| 灰度发布 | 按用户/设备/地域等维度定向放量 | 可精准验证、能做 A/B | 需要灰度规则与全链路透传（如 `uid` 分桶） |

- **回滚三要素**：制品可回（同一 digest 的旧版本）、配置可回（配置中心有版本历史）、**数据库可回**（见下）。K8s 中 `kubectl rollout undo` 最快，但必须确认 DB 变更已兼容旧代码。
- **Feature Flag（功能开关）**：把"上线"与"生效"解耦——代码先上线、开关控制是否生效，出问题**秒级关开关**而不用回滚部署；也可用于灰度与降级。注意开关要有生命周期管理（清理死开关），否则配置爆炸。
- **数据库变更与发布的协同（expand-contract / 先兼容后清理）**：① 扩展阶段：加新字段/新表（**只加不删**）、双写；② 数据迁移：后台任务回填历史数据；③ 切换读：代码切到读新字段（灰度验证）；④ 清理阶段：确认稳定后再删旧字段/旧代码。**禁止**在发布中做破坏性变更（改字段类型/删列/加非空约束），否则无法回滚；大表 DDL 用 `gh-ost`/`pt-online-schema-change` 避免锁表。
### 4.2 面试问答
- **Q1：金丝雀和灰度有什么区别？** A：结论——金丝雀是"按流量比例逐步放量"，灰度是"按人群/维度定向放量"，金丝雀常是灰度的第一阶段；原因是金丝雀关注"新版本整体指标是否健康"，灰度关注"特定人群/业务效果"（常配合 A/B 实验）；边界是两者都需要监控能按版本维度看指标（否则无法判定金丝雀是否异常）。
- **Q2：什么时候必须用蓝绿而不是滚动？** A：结论——当"新旧版本不能共存"或"要求秒级回滚"时；原因是滚动发布期间两版本同时服务，若协议不兼容（如 RPC 接口签名变更）就会出问题；边界是蓝绿需要双倍资源与流量切换能力，且**数据库仍然必须兼容**，否则切回来会数据错乱。
- **Q3：为什么数据库变更要"先兼容后清理"？** A：结论——为了让任何一个版本的应用都能在新库上运行，从而保证可回滚；原因是发布与回滚都可能发生，破坏性变更会让旧代码直接报错；边界是整个过程需要两到三次发布完成（加字段 → 切读 → 删旧字段），要接受一段时间的冗余字段与双写成本。
### 4.3 高频追问与陷阱
- **最常见的翻车点**：应用回滚了但 DB 迁移没回，旧代码读不到新字段直接报错；规范是"DB 变更只向前兼容，不做向后不兼容的变更"。
- 灰度规则若只在网关做（如按 IP 哈希），链路上游没有透传用户标识会导致同一用户一会儿新一会儿旧，体验不一致；正确做法是在入口按 `uid` 打标并全链路透传（Header/MQ 属性）。
### 4.4 示例
```yaml
# K8s 金丝雀：两套 Deployment 共用一个 Service，按副本数控制流量比例
# 稳定版 9 副本 + 金丝雀版 1 副本 ≈ 10% 流量；观察 30 分钟后放量或回滚
kubectl -n prod scale deploy/order-service-canary --replicas=1
kubectl -n prod rollout undo deploy/order-service           # 异常时一键回滚
```
### 4.5 一句话速记
发布策略按风险选：**停机最简、滚动最常用、蓝绿回滚最快、金丝雀最稳、灰度最精准**；回滚要"制品 + 配置 + DB"三同步，DB 变更永远**先兼容后清理**，功能开关让"生效"与"上线"解耦。
## 5. 环境管理与配置隔离
### 5.1 核心原理
- **环境划分**：本地 → dev/develop（联调）→ test/sit（测试）→ staging/pre（预发，与生产同构、连生产只读或影子库）→ prod；核心诉求是"**同一制品 + 不同配置**"，且环境之间网络与数据隔离。
- **配置分层原则**：① 代码里只放默认值（`application.yml`）；② 环境差异放环境变量或配置中心；③ 敏感信息放 Secret（K8s Secret/Vault/云 KMS），不落 Git。
- **Spring 配置优先级（高 → 低，节选）**：命令行参数 > `SPRING_APPLICATION_JSON` > 环境变量 > `application-{profile}.yml`（jar 外）> `application-{profile}.yml`（jar 内）> `application.yml`。多来源同名配置以优先级高者为准（`spring.config.import` 引入的配置中心通常排在较前）。
- **配置中心（Nacos/Apollo）**：集中管理 + 灰度发布 + 版本历史与一键回滚 + 变更审计 + 长轮询推送热更新；`@RefreshScope`（Spring Cloud）或 Apollo 的自动更新机制让配置改动不改包。
- **K8s 下的配置**：ConfigMap（非密）与 Secret（敏感）+ 环境变量注入；**注意环境变量方式不会热更**，文件挂载才会。详见 [Docker与Kubernetes](./02-Docker与Kubernetes.md)。
### 5.2 面试问答
- **Q1：为什么配置不能编译进包？** A：结论——否则无法做到"一次构建多处部署"，且改一个超时参数就要重新走发布流程；原因是配置属于部署期信息、代码属于构建期信息，混在一起会破坏两者节奏；边界是纯默认值（如 `server.port`）可以进包，但凡是随环境变化的（DB 地址、开关、密钥）必须外置。
- **Q2：预发环境和生产环境怎么保证一致？** A：结论——同镜像 digest + 同部署模板（Helm/Kustomize overlay）+ 只差配置值与数据；原因是环境差异是"测不出线上问题"的头号原因（版本、JDK、内核参数、中间件版本、数据量级）；边界是预发无法完全复刻数据量级与真实流量，所以还需要线上灰度与压测兜底。
- **Q3：配置中心的变更如何保证安全？** A：结论——变更要灰度、要审计、要能一键回滚；原因是配置直连线上行为，一次误改影响面可能大于一次代码发布；边界是给关键配置（如限流阈值、开关）设审批与变更窗口，并做配置变更的监控（变更即打点，与指标异常对齐排查）。
### 5.3 高频追问与陷阱
- 环境变量与配置文件同名时容易"改了不生效"，排查要打印生效配置来源（Spring Boot Actuator 的 `/actuator/env` 与 `/actuator/configprops`）。
- 本地开发直连线上数据库是重大隐患，应使用独立测试库或影子库；生产账号一律最小权限（只读账号给查询平台用）。
### 5.4 示例
```bash
# Spring Boot 配置来源排查（确认生效值到底来自哪一层）
curl -s localhost:8080/actuator/env | jq '.propertySources[] | {name, props: (.properties|keys)}'
# K8s：同一镜像 + 不同 ConfigMap 实现环境隔离
kubectl -n staging set env deploy/order-service SPRING_PROFILES_ACTIVE=staging
kubectl -n prod    set env deploy/order-service SPRING_PROFILES_ACTIVE=prod
```
### 5.5 一句话速记
环境管理的铁律是**同一制品、不同配置**（预发与生产只差配置）；配置分层为"代码默认值 → 环境变量 → 配置中心 → Secret"，改配置要**灰度 + 审计 + 可回滚**。
## 6. 监控体系分层与关键指标清单
### 6.1 核心原理
监控分四层，从下到上定位问题（结论：**先看基础设施与中间件是否健康，再看应用，最后看业务**）：

| 层次 | 关注内容 | 常用工具 | 典型阈值 |
| --- | --- | --- | --- |
| 基础设施 | CPU、内存、磁盘、网络、load | node_exporter、云监控 | CPU > 80% 持续 5min、磁盘 > 85%、inode > 90% |
| 中间件 | MySQL、Redis、MQ、ES、Nginx | 各 exporter、内置指标 | 主从延迟 > 10s、Redis 内存 > 80%、MQ 堆积 > 10 万 |
| 应用 | JVM、接口、连接池、线程池 | Micrometer + Prometheus、APM | P99 RT > 500ms、错误率 > 1%、FGC 频繁 |
| 业务 | 订单量、支付成功率、转化率 | 埋点 + 数据平台 | 成功率环比跌 5%、下单量骤降 |

**关键指标清单（背下来能直接用）**：
- **主机**：`CPU 使用率 / load1/5/15 / 内存使用率（含 cache 区分）/ 磁盘使用率与 inode / 磁盘 IO await / 网络带宽与丢包重传`；方法论 **USE**（利用率、饱和度、错误）。
- **JVM**：堆/非堆使用量、**GC 次数与耗时（YoungGC/FGC）**、GC 停顿、线程数与死锁、类加载数、直接内存；FGC 频繁或单次停顿 > 1s 要告警（见 [JVM与GC调优](../01-Java基础/04-JVM与GC调优.md)）。
- **接口（服务）**：QPS、**RT（P95/P99 而非平均值）**、错误率（5xx/业务异常）、慢调用 Top、并发数；方法论 **RED**（Rate、Errors、Duration），加上**四大黄金信号**（延迟、流量、错误、饱和度）。
- **数据库**：连接池（活跃/空闲/等待数、获取超时）、QPS/TPS、慢 SQL 数、锁等待与死锁、主从延迟。
- **缓存/MQ**：Redis 命中率、连接数、慢命令、大 key 热 key；MQ 生产/消费 TPS、**堆积量**、消费延迟、重试与死信量。
### 6.2 面试问答
- **Q1：为什么监控要分层？** A：结论——为了缩短 MTTD/MTTR（快速定位到层）；原因是单层指标无法区分"是流量涨了、依赖挂了、还是自己代码有问题"；边界是分层不是"指标越多越好"，每一层保留**能触发行动**的少量指标，避免告警疲劳。
- **Q2：为什么看 P99 而不是平均值？** A：结论——平均值会掩盖长尾，而用户体验由长尾决定；原因是少量慢请求在平均中被稀释（如 1% 的 3s 请求在 P99 才显现），且下游超时/重试往往由 P99 触发；边界是平均值仍要保留用于看整体趋势，另外要区分"服务端 RT"与"客户端观测 RT"（后者含排队与网络）。
- **Q3：接口错误率涨了但 QPS 没变，怎么排查？** A：结论——按"依赖 → 资源 → 自身变更"三条线并行查；原因是错误率上升常见于下游故障（DB/Redis/三方超时）、资源打满（线程池/连接池排队超时）或刚发布的变更；边界是先用版本维度与依赖维度拆指标（错误率 by instance/version/upstream），再决定是回滚还是扩容。
### 6.3 高频追问与陷阱
- **告警必须指向行动**：只报"CPU 高"没有意义，要报"CPU 高且接口 P99 恶化"，否则就是噪音；无行动项的指标只上大盘、不设告警。
- 只监控"平均值/总量"会漏掉单实例故障：必须**按实例/按可用区拆分**，把"1 台挂了但负载均衡没摘除"这类问题暴露出来。
### 6.4 示例
```yaml
# Prometheus 抓取配置：K8s 服务发现 + Pod 注解自动发现
- job_name: kubernetes-pods
  kubernetes_sd_configs: [{ role: pod }]
  relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep  regex: "true"          # 只有带注解的 Pod 才被采集
```
### 6.5 一句话速记
监控四层：**基础设施 → 中间件 → 应用 → 业务**；方法论用 **USE 看资源、RED 看服务、四大黄金信号看整体**；接口看 **P99 与错误率**，JVM 看 **FGC 与停顿**，MQ 看**堆积**。
## 7. Prometheus 数据模型、PromQL 与告警设计
### 7.1 核心原理
- **数据模型**：时序 = 指标名 + 标签集 + 时间戳 + 值，如 `http_requests_total{job="order", instance="10.0.0.1:8080", status="500"}`；标签是查询与聚合的核心，**高基数标签（用户 ID、traceId、URL 全路径）是性能杀手**，必须避免。
- **四种指标类型**：`Counter`（只增，如请求数，要配 `rate/increase` 使用）、`Gauge`（可增可减，如内存使用）、`Histogram`（分桶统计，可算分位数与平均值，服务端聚合）、`Summary`（客户端算分位数，无法跨实例聚合）。**服务端分位数优先 Histogram**。
- **拉取（Pull）模型**：Prometheus 周期性 `scrape` 目标（默认 15s），配合服务发现动态发现目标；短生命周期任务用 **Pushgateway**（勿滥用，会成为单点与数据残留源）；数据本地 TSDB 存储（默认保留 15 天，长期存储接 Thanos/VictoriaMetrics）。
- **Exporter**：`node_exporter`（主机）、`mysqld_exporter`、`redis_exporter`、`kafka_exporter`、`blackbox_exporter`（黑盒探测 HTTP/TCP/ICMP）；Java 应用直接用 **Micrometer + `/actuator/prometheus`**。
- **PromQL 核心**：`rate(http_requests_total[5m])` 每秒速率（Counter 必用）、`increase(...[1h])` 区间增量、`histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))` 算 P99、`sum by (instance) (rate(x[5m]))` 分组聚合、`topk(10, ...)` 取 Top、`irate` 看瞬时突刺、`up` 判断目标存活、`absent()` 判缺失。
- **告警规则**：`expr`（表达式，**加 `for` 持续时间抑制抖动**）+ `labels`（severity 分级）+ `annotations`（summary/description/runbook_url）；示例：`histogram_quantile(0.99, ...) > 0.5` 持续 `5m`。
- **Alertmanager**：`route`（按 match 匹配标签路由到不同 receiver，`continue` 决定是否继续匹配）、`group_by`（按服务/集群分组，**把同源大量告警聚成一条**）、`group_wait/group_interval/repeat_interval`（抑制风暴与提醒节奏）、`inhibit_rules`（抑制规则，如集群整体故障时抑制个体告警）、`silences`（静默，用于维护窗口）、`mute_time_intervals`（免打扰时段）。**降噪三板斧：分组 + 抑制 + 静默**；分级建议 P0（核心不可用，电话）→ P1（功能受损，IM + 值班）→ P2（需关注，IM）→ P3（趋势与容量，日报）。
### 7.2 面试问答
- **Q1：Counter 为什么要用 rate 而不是直接看值？** A：结论——Counter 只增且会因进程重启归零，直接看值无意义；原因是 rate 自动处理计数器重置，计算出"每秒增长速率"；边界是 `rate` 需要至少两个采样点，窗口过短（如 `[1m]`）会抖动，过长（如 `[1h]`）会掩盖突刺，常用 `[5m]` 配 `irate` 看瞬时。
- **Q2：Histogram 怎么算分位数？** A：结论——用 `histogram_quantile` 在 `_bucket` 序列上按 `le` 标签聚合；原因是分位数必须在服务端由分桶计数插值计算，且**必须带 `le` 标签聚合**；边界是桶划分不合理（如全部请求都落在 `+Inf` 桶）会导致分位数失真，应为业务量级定制 bucket（如 10ms/50ms/100ms/500ms/1s/3s）。
- **Q3：告警风暴怎么治理？** A：结论——分组、抑制、静默 + 分级 + 合并根因告警；原因是同一故障会同时触发上下游几十条告警，导致值班被淹没而错过关键信息；边界是治本手段是"告警聚合到根因"（如只对入口错误率与关键依赖告警，个体实例异常由分组呈现），并定期清理"长期无人处理"的告警（僵尸告警）。
### 7.3 高频追问与陷阱
- 告警表达式缺少 `for` 会因一次抖动误报；缺少 `by` 聚合会把单实例异常放大成全量告警。
- 拉取目标过多或采集间隔过密会导致 Prometheus 自身 OOM/慢查询；查询侧要用 `recording rules` 预聚合大范围统计。
- Pushgateway 上报的指标会**永久残留**（除非显式删除），所以短任务必须带 job/instance 并在任务结束后清理，否则出现"幽灵数据"。
### 7.4 示例
```yaml
# 告警规则：接口 P99 超过 500ms 持续 5 分钟
groups:
  - name: app-slo
    rules:
      - alert: ApiP99LatencyHigh
        expr: histogram_quantile(0.99,
                sum by (le, job) (rate(http_server_requests_seconds_bucket{job="order-service"}[5m]))) > 0.5
        for: 5m
        labels: { severity: P1, team: order }
        annotations:
          summary: "订单服务 P99 超过 500ms"
          description: "当前 P99 = {{ $value | humanizeDuration }}，请查看大盘与慢调用 Top"
          runbook_url: "https://wiki.example.com/runbook/order-latency"
---
# Alertmanager：分组降噪 + 抑制规则
route:
  receiver: default
  group_by: [alertname, cluster, service]
  group_wait: 30s          # 首次聚合等待，把同批告警合成一条
  group_interval: 5m
  repeat_interval: 4h      # 未恢复时的重复提醒间隔
  routes:
    - matchers: [severity="P0"]  receiver: pagerduty
    - matchers: [severity="P1"]  receiver: oncall-im
      continue: true
inhibit_rules:             # 集群级故障时抑制个体告警，避免风暴
  - source_matchers: [alertname="ClusterDown"]
    target_matchers: [severity=~"P1|P2"]
    equal: [cluster]
```
```promql
# 常用查询速查
rate(http_requests_total{job="order-service"}[5m])                       # QPS
sum by (uri) (rate(http_requests_total{status=~"5.."}[5m])) / sum by (uri) (rate(http_requests_total[5m]))  # 错误率
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))                   # P95
topk(10, sum by (instance) (rate(jvm_gc_pause_seconds_sum[5m])))           # GC 耗时 Top10
increase(mq_consume_fail_total{group="order"}[1h])                        # 1 小时消费失败增量
```
### 7.5 一句话速记
Prometheus = **指标名 + 标签 + 时序（Pull 拉取）**，四类型 **Counter/Gauge/Histogram/Summary**；PromQL 牢记 `rate/increase/histogram_quantile/topk/sum by`；告警要 **`for` 抑制抖动 + 分级 + 分组/抑制/静默降噪**。
## 8. Grafana 面板设计与日志体系（ELK/EFK）
### 8.1 核心原理
**Grafana 面板设计要点**：① 一屏讲清"是否正常"——左上角放**总览（QPS/错误率/P99/饱和度）**，右下角放明细；② 按 RED/USE 组织变量与图表，图表单位统一（百分比/毫秒/次幂）；③ 设**阈值线与颜色区间**（绿/黄/红），异常一眼可见；④ 用模板变量（`$instance`、`$env`）复用同一张图；⑤ 避免高基数图（按 URL 全路径会爆图例），URL 要归一化；⑥ 面板数量控制在 8 个以内，多则拆成"总览 + 详情"两级大盘；⑦ 时间范围与刷新间隔统一（如 6h/30s），并开启跨面板时间同步，便于关联排查。

**日志体系**：
- **架构**：应用输出 JSON 到 stdout → 采集（Filebeat/Fluent Bit）→ 缓冲与处理（Kafka 可选）→ 解析与存储（Logstash/Elasticsearch）→ 检索与可视化（Kibana）。EFK 即用 Fluentd/Fluent Bit 替代 Logstash（更轻量，K8s 常见）。
- **Filebeat**：轻量采集器，配置 `filebeat.inputs`（路径 + `multiline` 处理 Java 堆栈多行合并）、`fields`（打环境/应用标签）、`output.elasticsearch` 或 `output.kafka`；registry 文件记录读取位点。
- **Logstash**：`input/filter/output` 三段，filter 用 `grok`（非 JSON 老日志）、`json`（结构化日志首选的解析方式）、`date`（时间戳解析）、`mutate`（字段改名/删除）、`geoip`；注意 Logstash 重（JVM），大规模场景用 Filebeat 直连 ES。
- **Elasticsearch/Kibana**：索引按天/按策略轮转 + **ILM 生命周期管理**（热→温→冷→删除，控制存储成本）；Kibana 用 KQL 检索（`level:ERROR and app:order-service and traceId:"xxx"`）、保存查询、做告警（Watcher/ElastAlert）。
- **日志规范**：**结构化 JSON**（不写多行文本）、必带 `traceId/app/env/host/level/timestamp`、级别得当、敏感字段脱敏、**大对象不打印全量**（截断）、高频路径**采样**、禁止循环内打日志与 `System.out`。**日志量控制**手段：提高生产日志级别至 INFO/WARN、`AsyncAppender` 异步、采样与限流（Logback `TurboFilter`/`RateLimiter`）、缩短保留期（热索引 7 天）。
### 8.2 面试问答
- **Q1：ELK 里 Java 异常堆栈怎么处理？** A：结论——采集侧做 multiline 合并、应用侧输出 JSON 并把堆栈作为单个字段；原因是堆栈是多行文本，逐行采集会拆成几十条无意义的独立日志；边界是 JSON 方案下要把 `stack_trace` 转义为单行（`\n`），检索时用 Kibana 展开，不要依赖行业务多行合并（性能差）。
- **Q2：日志量太大怎么办？** A：结论——先分级与采样，再考虑降存储成本；原因是绝大多数日志是 DEBUG/重复 INFO，价值极低；边界是错误日志与关键业务节点必须全量（可追溯），高频非关键日志（如轮询、心跳）采样 1%~10%，并用 ILM 把冷数据压缩或直接删除。
- **Q3：Grafana 大盘为什么不能什么都放？** A：结论——大盘的目标是"5 秒内判断是否异常并知道看哪块"，不是数据仓库；原因是面板过多无法形成视觉焦点、加载慢、且掩盖关键信号；边界是明细与探索留给临时查询（Explore）与下钻链接，大盘只放能触发行动的指标。
### 8.3 高频追问与陷阱
- 采集缺少 `traceId` 字段会导致"日志与链路对不上"，排查时无法从一条错误日志跳到完整调用链；日志规范里必须强制 `traceId`（见 §9）。
- Kibana 直接对全量索引做模糊查询（前导通配）会打爆 ES；规范是先用时间范围 + 标签过滤，避免 `*xxx*`。
### 8.4 示例
```yaml
# Filebeat 采集 Java 应用日志（K8s DaemonSet 场景，关键片段）
filebeat.inputs:
  - type: filestream
    paths: [/var/log/pods/*order-service*/*.log]
    parsers:
      - ndjson: { target: "", add_error_key: true }   # 应用已输出 JSON，直接解析字段
processors:
  - add_fields: { target: "", fields: { app: order-service, env: prod } }
output.elasticsearch:
  hosts: ["https://es.example.com:9200"]
  index: "app-logs-%{[env]}-%{+yyyy.MM.dd}"
```
### 8.5 一句话速记
Grafana 大盘只放**能触发行动的指标**（RED/USE + 阈值线 + 模板变量）；日志链路是 **stdout JSON → Filebeat/Fluent Bit → （Kafka）→ Logstash → ES → Kibana**，规范是结构化、带 `traceId`、分级采样、ILM 控成本。
## 9. 链路追踪与可观测性落地
### 9.1 核心原理
- **OpenTelemetry（OTel）核心概念**：**Trace**（一次请求的完整调用链）由多个 **Span**（单个操作，含起止时间、属性、事件、状态）组成，通过 **Context 传播**（HTTP 头 `traceparent`、MQ 属性）串联跨进程调用；每一跳生成 `traceId/spanId/parentSpanId`。组件包括 API/SDK、**Java Agent（自动埋点，无侵入）**、**Collector**（接收 → 处理 → 导出到 Jaeger/Tempo/SkyWalking 等后端）与 Semantic Conventions（字段命名标准）。
- **采样率**：头部采样（在入口按比例决定，简单但可能丢慢请求）、尾部采样（在 Collector 侧按耗时/错误决定保留，能保住异常链路但需缓冲）；生产常用"低比例头部采样 + 错误全采"。
- **TraceId 透传与 MDC 集成**：入口（网关/Filter）生成或继承 TraceId，写入日志 `MDC` 并输出到日志字段；跨线程用 `TaskDecorator` 或 TTL 复制上下文（`MDC` 基于 `ThreadLocal`）；跨服务用 Header 传播；这样"一条错误日志 → 一次完整调用链"可互跳。
- **SkyWalking vs Zipkin（结论）**：SkyWalking——Java Agent 无侵入、自带拓扑图/告警/性能剖析，存储用 ES/BanyanDB，适合国内 Java 体系快速落地；Zipkin——更轻、生态偏 OpenTracing/OTel，功能聚焦链路展示、需自行拼装（Brave/OTel 采集），适合简单场景或已有 OTel 体系。**新项目统一选 OTel 协议 + 后端任选**，避免厂商锁定。
- **Micrometer 指标埋点**：Spring Boot Actuator + Micrometer 自动暴露 JVM/HTTP/连接池指标（`/actuator/prometheus`）；自定义指标用 `Counter`（计数，如下单失败数）、`Gauge`（瞬时值，如队列深度）、`Timer`（耗时分布，用 `Timer.builder("order.pay").publishPercentiles(0.95, 0.99)`）；`@Timed` 注解给方法打点；**业务指标**（支付成功率、库存扣减失败率）比技术指标更早发现业务异常。
### 9.2 面试问答
- **Q1：有了日志和指标，为什么还要链路追踪？** A：结论——日志解决"单点发生了什么"、指标解决"整体是否异常"，追踪解决"**慢在哪一跳**"；原因是微服务下一次请求跨 5-10 个服务，靠日志人工拼接效率极低；边界是追踪有采样与存储成本，且需要全链路埋点覆盖（缺一环就断链）。
- **Q2：TraceId 为什么会在异步链路丢失？** A：结论——`MDC` 基于 `ThreadLocal`，线程池提交任务不会自动继承上下文；原因是上下文绑定在创建它的线程上；边界是解法为 `TaskDecorator` 包装任务复制 MDC、或使用 `TransmittableThreadLocal`（阿里 TTL）在任务提交/执行时传递，MQ 场景则把 TraceId 放进消息属性透传。
- **Q3：全量链路数据能不能都存？** A：结论——不能，必须采样；原因是每天数十亿 Span 的存储与查询成本不可接受，且高价值链路其实很少；边界是策略上"错误与慢请求全采 + 正常请求按比例采样"，并把聚合指标（而非原始 Span）作为长期趋势数据。
### 9.3 高频追问与陷阱
- 只装 Agent 不统一传播格式（有的服务用 `X-Trace-Id`、有的用 `traceparent`）会造成链路断裂，规范是**统一 W3C `traceparent`**。
- 追踪采样率与日志采样率要分开配置：日志采样太高会丢排查线索，追踪采样太低会丢慢链路，两者目标不同。
### 9.4 示例
```java
// Micrometer 自定义业务指标 + TraceId 进日志
private final Counter payFail = Counter.builder("order_pay_fail_total")
        .tag("channel", "ALI").register(registry);   // 业务失败计数（比技术指标更早预警）
private final Timer payTimer = Timer.builder("order_pay_duration")
        .publishPercentiles(0.95, 0.99).register(registry);

MDC.put("traceId", currentTraceId());                 // 入口写入，日志 Pattern 输出 %X{traceId}
try { doPay(); } catch (Exception e) { payFail.increment(); throw e; } finally { MDC.remove("traceId"); }
```
```yaml
# OpenTelemetry Java Agent 接入（无侵入）
java -javaagent:/opt/otel/opentelemetry-javaagent.jar \
     -Dotel.service.name=order-service \
     -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
     -Dotel.traces.sampler=parentbased_traceidratio -Dotel.traces.sampler.arg=0.1 \
     -jar app.jar
```
### 9.5 一句话速记
可观测性三件套：**指标（是否异常）+ 日志（发生了什么）+ 追踪（慢在哪一跳）**；OTel 用 **Trace/Span/Context 传播**打通链路，TraceId 进 MDC 串日志；采样策略是"错误全采 + 正常按比例"。
## 10. 值班、故障响应与线上排查模板
### 10.1 核心原理
- **故障分级**：P0（核心业务不可用/资损，如下单全失败）→ P1（核心功能受损或大比例用户受影响）→ P2（非核心功能异常/有损但可绕行）→ P3（体验问题/潜在隐患）。分级决定**响应时限与升级路径**（示例基线：P0 5 分钟响应、15 分钟升级到主管并拉起故障群；P1 15 分钟响应、30 分钟升级）。
- **响应原则（结论先行）**：**止血优先于定位根因**——先回滚/降级/限流/扩容/切流把影响面控住，再慢慢查原因。故障期间固定角色：指挥（决策与汇报）、执行（操作变更）、记录（时间线与证据）。
- **标准动作**：确认现象与影响面（谁、哪个功能、什么时间、多大规模）→ 判断是否刚发布（是则立即回滚）→ 查大盘与日志（从入口错误率 → 依赖指标 → 单实例明细）→ 止血 → 验证恢复 → 保留现场（dump/日志/快照）→ 根因分析 → 复盘与行动项。
- **复盘文化**：无指责（blameless）、对事不对人、用时间线还原事实、用 **5 Whys** 找根因、产出可验证的行动项（谁、何时、如何验证），并回填到监控/流程/自动化。
- **核心度量**：**MTTD**（平均发现时长，靠监控覆盖与告警灵敏度）、**MTTR**（平均恢复时长，靠回滚/降级等止血手段的自动化）、**MTBF**（平均故障间隔，靠根因治理）。面试谈"高可用"时用这三个指标量化比空谈架构更打动人。

**"线上问题排查"通用答题模板（背下来直接套）**：
1. **确认与定级**：现象（错误码/超时/无响应）、影响面（哪些用户/接口/比例）、起始时间（是否与发布/变更对齐）。
2. **快速止血**：按"**回滚 → 降级 → 限流 → 扩容 → 切流**"优先级操作，先让业务恢复。
3. **分层定位**：入口（LB/网关/Nginx）→ 应用（CPU/内存/GC/线程池/连接池/日志/堆栈）→ 中间件（DB/Redis/MQ）→ 依赖（三方接口）→ 基础设施（网络/磁盘/DNS）。
4. **工具与证据**：`top/vmstat/iostat`、`jstack`（线程死锁/阻塞）、`jmap`/`jcmd` + MAT（内存）、`arthas`（在线诊断：`dashboard/thread/trace/watch`）、`tcpdump`/`ss`（网络）、火焰图（CPU 热点）、gc 日志。
5. **根因与修复**：区分直接原因（如连接池耗尽）与根本原因（如慢 SQL 导致连接不释放），修复要能验证（回归 + 压测 + 监控观察）。
6. **收尾**：复盘、行动项、补监控与告警、补预案（Runbook）。
### 10.2 面试问答
- **Q1：线上接口大量超时，你的第一反应是什么？** A：结论——先看是不是刚发布，是则立刻回滚；同时看入口错误率与依赖指标判断是"自己变慢"还是"依赖挂了"；原因是发布引入的问题占线上故障的很大比例，回滚是最快的止血方式；边界是若无法回滚（如 DB 变更已生效），改用降级（关闭非核心功能/返回缓存兜底）与限流保护核心链路。
- **Q2：MTTR 怎么降？** A：结论——把常见的止血动作自动化（一键回滚、开关降级、自动扩容）并保证可观测性覆盖到能定位的粒度；原因是定位占 MTTR 的大头，而复盘发现"其实只要回滚或扩容"的场景极多；边界是不能只追求快而忽略验证，止血后必须确认指标恢复并保留现场，否则同样问题会再来一次。
- **Q3：怎么证明系统是"可运维"的？** A：结论——用"故障能否在 5 分钟内被发现、15 分钟内被恢复"来证明，而不是罗列组件；原因是可运维性的本质是**可观测 + 可回滚 + 可降级 + 有预案**；边界是要有演练证据（混沌演练/故障演练/压测），没演练过的预案等于没有预案。
### 10.3 高频追问与陷阱
- 只在故障群里"边聊边查"而没有时间线记录，会导致复盘时事实不清、无法定位触发点；规范是**从拉起故障群开始就记时间线**（几点几分做了什么、指标如何变化）。
- 止血后不复盘、不复核监控，问题会以同样形式再次发生；行动项必须可验证且有时间点。
### 10.4 示例
```text
线上故障时间线模板（复盘用）
14:02  监控告警：order-service 5xx 错误率 12%（P0 拉起故障群）
14:04  确认影响：下单接口失败，影响约 30% 请求；起始 13:58，与 13:55 的发布对齐
14:05  止血：kubectl rollout undo deploy/order-service（1 分钟内部署回滚完成）
14:08  验证：错误率回落至 0.1%，P99 恢复 120ms；保留现场（jstack + 慢 SQL 日志）
14:20  根因：新代码引入的 N+1 查询导致 DB 连接池耗尽（5 Whys：为什么连接池耗尽 → 慢查询占满连接）
行动项：① 增加连接池等待数与慢 SQL 告警（负责人/截止日）；② 发布前加 SQL 审核门禁；③ 加入预发压测门禁
```
### 10.5 一句话速记
故障处理信条：**先止血、后根因**（回滚 → 降级 → 限流 → 扩容 → 切流）；用 **MTTD/MTTR/MTBF** 量化可运维性；复盘要**无指责 + 时间线 + 5 Whys + 可验证行动项**。
## 自测题（含答案要点）
1. **持续交付与持续部署的区别？** —— 持续交付"可随时上线但需人工批准"，持续部署"门禁通过即自动上线"。
2. **为什么要求 build once, deploy many？** —— 保证各环境制品一致，避免重复构建引入差异；前提是配置外置。
3. **Jenkins Controller 为什么不跑构建？** —— 抢占调度资源且安全风险大，应由打标签的 Agent（容器 Agent 最佳）执行。
4. **Jenkinsfile 里怎么安全用密钥？** —— `withCredentials` 注入变量、关闭命令回显、日志脱敏，更好的是让流水线不接触明文（用目标平台 Secret）。
5. **声明式 Pipeline 的 `post` 有什么用途？** —— 分条件（always/success/failure/unstable/changed）做通知、归档、回滚与清理。
6. **共享库解决什么问题？** —— 把重复流水线逻辑收敛到 `vars/`、`src/`，通过 `@Library` 版本化引用，避免"改一处改几十个仓库"。
7. **Nexus 三类仓库？** —— proxy（代理缓存中央仓库）、hosted（自研发布，分 releases/snapshots）、group（聚合只读入口）。
8. **SNAPSHOT 为什么不能上生产？** —— 同版本号内容可被覆盖，无法复现与回滚。
9. **镜像 tag 规范怎么写？** —— `环境-应用-语义版本-gitSHA-构建号`，禁用 `latest`；跨环境用同一 digest。
10. **五种发布策略的适用场景？** —— 停机（可停业务）、滚动（通用无状态）、蓝绿（要秒级回滚/新旧不能共存）、金丝雀（按比例验证）、灰度（按人群维度定向与 A/B）。
11. **回滚三要素？** —— 制品可回（旧 digest）、配置可回（配置中心版本）、数据库可回（变更向前兼容）。
12. **Feature Flag 的价值？** —— 把"上线"与"生效"解耦，出问题秒级关闭而不必回滚部署，也用于灰度与降级。
13. **数据库变更的 expand-contract 四步？** —— 加字段/双写 → 回填数据 → 切读新字段 → 确认稳定后删旧字段；禁止破坏性变更。
14. **监控四层与对应方法论？** —— 基础设施（USE）、中间件（连接/延迟/堆积）、应用（RED）、业务（转化与成功率）。
15. **为什么看 P99 而不是平均值？** —— 平均值掩盖长尾，用户体验与下游超时由长尾决定。
16. **Prometheus 四种指标类型？** —— Counter（只增，配 rate）、Gauge（可增可减）、Histogram（分桶，服务端算分位）、Summary（客户端算分位，不能跨实例聚合）。
17. **`rate` 与 `increase` 的区别？** —— `rate` 算每秒速率，`increase` 算区间总增量（同为 Counter 专用，都会处理计数器重置）。
18. **Histogram 分位数怎么写？** —— `histogram_quantile(0.99, sum by (le) (rate(x_bucket[5m])))`，必须按 `le` 聚合，桶划分要贴合业务量级。
19. **告警降噪三板斧？** —— 分组（group_by 聚合同源告警）、抑制（inhibit_rules 根因抑制个体）、静默（silences 维护窗口）。
20. **`for` 字段的作用？** —— 要求表达式持续成立一段时间才告警，抑制瞬时抖动造成的误报。
## 背诵卡片
1. CI = 频繁集成 + 自动构建测试；持续交付 = 随时可发、人工点发；持续部署 = 自动发。
2. 流水线设计八原则：快慢分离、门禁前置、一次构建多处部署、幂等、配置入库、失败即停、可追溯、流水线即代码。
3. Jenkins：Controller 只调度，Agent 执行；容器 Agent 最佳；标签选节点。
4. 声明式 Pipeline 五件套：agent / environment / parameters / stages / post。
5. `post` 条件：always、success、failure、unstable、changed、aborted。
6. Jenkins 凭据用 `withCredentials`，严禁写入 Jenkinsfile。
7. Jenkins 触发：Webhook 优先，`pollSCM` 次之，`upstream` 用于串联。
8. 共享库用 `vars/`（全局步骤）+ `src/`（Groovy 类），并以 tag 引用版本。
9. Nexus 三类仓库：proxy、hosted（releases/snapshots）、group。
10. `distributionManagement` 的 id 必须与 `settings.xml` 的 server id 一致。
11. SNAPSHOT 可覆盖，禁止上生产；release 版本不可覆盖。
12. Harbor 提供 RBAC、复制、Trivy 扫描、tag 保留与 GC。
13. 镜像 tag：环境-应用-语义版本-gitSHA-构建号；禁用 latest；跨环境同 digest。
14. 发布策略：停机、滚动、蓝绿、金丝雀、灰度（风险与成本递减/递增的权衡）。
15. K8s 默认滚动发布：`maxSurge: 1, maxUnavailable: 0` 发布期不降容量。
16. 蓝绿靠 LB 切换，回滚最快；金丝雀按比例放量，灰度按人群维度。
17. 回滚三要素：制品、配置、数据库（向前兼容）。
18. Feature Flag 让"上线"与"生效"解耦，可秒级关闭止损。
19. DB 变更 expand-contract：加字段双写 → 回填 → 切读 → 删旧字段。
20. 大表 DDL 用 gh-ost / pt-online-schema-change 避免锁表。
21. 环境管理铁律：同一制品、不同配置；预发与生产只差配置。
22. Spring 配置优先级：命令行 > 环境变量 > profile 配置 > 默认配置。
23. 配置中心要有灰度、审计、版本回滚；关键开关设审批。
24. 监控四层：基础设施、中间件、应用、业务。
25. 方法论：USE 看资源、RED 看服务、四大黄金信号看整体。
26. 接口指标看 QPS、P99、错误率、慢调用 Top。
27. JVM 指标看堆使用、GC 次数与耗时、线程数、死锁。
28. DB 看连接池活跃/等待、慢 SQL、锁等待、主从延迟。
29. MQ 看堆积量、生产/消费 TPS、消费延迟、死信量。
30. Prometheus 四类型：Counter、Gauge、Histogram、Summary。
31. Counter 必配 `rate`/`increase`；分位数优先 Histogram（服务端聚合）。
32. `histogram_quantile(0.99, sum by (le) (rate(x_bucket[5m])))` 是 P99 标准写法。
33. PromQL 常用：rate、increase、irate、topk、sum by、up、absent。
34. Pull 模型 + 服务发现 + Exporter；短任务用 Pushgateway（注意数据残留）。
