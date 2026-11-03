# Maven、Git 与工程实践

> **定位**：Maven 坐标/生命周期/依赖调解与冲突排查 → 多模块与私服发布 → Git 模型、分支策略与事故恢复 → 提交评审规范、多环境配置与 API 文档，覆盖"日常工程能力"的全部门面题。
> **重要度**：★★★★★（社招必问"依赖冲突怎么解决""你怎么做灰度""版本号怎么定""代码质量怎么保证"；Git 事故恢复是实操题）
> **建议复习时长**：4 小时（Maven 1.5h + Git 1.5h + 规范与实践 1h）
> **版本基准**：Maven 3.9.x（Maven 4 未普及）、Git 2.4x、springdoc-openapi 2.x（Spring Boot 3）、Conventional Commits 1.0。
> **关联文档**：[复习路线总纲](../00-复习路线总纲.md)、[CICD与监控告警](./03-CICD与监控告警.md)、[Docker与Kubernetes](./02-Docker与Kubernetes.md)、[设计模式与代码规范](./01-设计模式与代码规范.md)、[SpringBoot与自动装配](../03-框架/02-SpringBoot与自动装配.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| Maven 坐标、POM、超级 POM、仓库与查找顺序 | ★★★★☆ | 八股原理 / 命令速查 |
| 三大生命周期与常用命令 | ★★★★★ | 命令速查 |
| 依赖范围、传递依赖与依赖调解 | ★★★★★ | 八股原理 / 故障排查 |
| dependencyManagement / parent / BOM | ★★★★★ | 配置实践 |
| 多模块工程（聚合 vs 继承）与版本统一 | ★★★★☆ | 配置实践 |
| 插件机制、私服发布与构建加速 | ★★★★☆ | 配置实践 / 命令速查 |
| Git 三区模型与命令速查 | ★★★★★ | 命令速查 |
| 分支模型与 merge/rebase | ★★★★★ | 八股原理 / 场景题 |
| Git 事故恢复（reset/reflog/revert） | ★★★★★ | 故障排查 / 场景题 |
| 提交规范、PR 与代码评审 | ★★★★☆ | 最佳实践 |
| 多环境配置、语义化版本与 API 文档 | ★★★★☆ | 配置实践 / 最佳实践 |
## 1. Maven 核心概念与生命周期
### 1.1 核心原理
- **坐标（GAV）**：`groupId`（组织/包名，如 `com.example.order`）+ `artifactId`（模块名）+ `version`；三者唯一确定一个构件，仓库路径为 `group/artifact/version/artifact-version.jar`。
- **POM**：项目对象模型，核心元素 `parent`、`properties`、`dependencies`、`dependencyManagement`、`build`（`plugins`/`resources`）、`repositories`、`distributionManagement`、`profiles`；`packaging` 默认 `jar`（聚合模块用 `pom`）。
- **超级 POM（Super POM）**：Maven 内置的父 POM（在 `maven-model-builder` 的 `pom-4.0.0.xml` 中），定义了中央仓库地址、标准目录结构（`src/main/java`、`target/`）、默认插件绑定与版本、`install`/`deploy` 的默认路径等。**任何 POM 都隐式继承它**，用 `mvn help:effective-pom` 查看生效后的完整配置（排查"参数没生效"的终极手段）。
- **仓库类型与查找顺序**：① **本地仓库**（`~/.m2/repository`，命中即用）→ ② **私服/镜像**（Nexus：`proxy` 代理中央仓库并缓存）→ ③ **远程仓库**（POM 里的 `repositories` 或超级 POM 的 central）。**镜像 `mirrorOf` 优先级高于 POM 中的仓库**：`<mirrorOf>*</mirrorOf>` 覆盖所有仓库，`<mirrorOf>central</mirrorOf>` 只接管中央仓库；`*` 与 `external:*` 语义不同（后者只代理非本机仓库）。
- **settings.xml**：`~/.m2/settings.xml`（用户级）与 `$MAVEN_HOME/conf/settings.xml`（全局级），用户级优先；常用配置 `<localRepository>`、`<mirrors>`、`<servers>`（发布凭据，`id` 必须与 POM 的 `distributionManagement` 中 `id` 一致）、`<profiles>`（JDK/环境差异化配置）。
- **三大生命周期（lifecycle）**：`clean`、`default`（构建）、`site`（站点）。**生命周期之间相互独立**（`mvn clean install` 是两条生命周期命令的串联），同一生命周期内**从前往后依次执行到指定阶段**（执行 `install` 会自动跑完 `compile`→`test`→`package` 等）。
- **default 关键阶段（背这条链）**：`validate → compile → test → package → verify → install → deploy`；`install` 把构件装进本地仓库供其他模块依赖，`deploy` 上传到远程仓库（私服）。
### 1.2 面试问答
- **Q1：`mvn package` 和 `mvn install` 有什么区别？** A：结论——`package` 只把产物打到 `target/`，`install` 还会装进本地仓库；原因是多模块或跨项目依赖必须从本地仓库解析，只 package 无法被其他模块引用；边界是 CI 中各模块构建用 `mvn install` 或 `mvn deploy`，单模块本地验证用 `package` 即可（更快）。
- **Q2：为什么本地能跑、CI 上编译不过？** A：结论——环境差异（JDK 版本、本地仓库有脏构件、依赖版本未锁定）；原因是本地 `~/.m2` 里可能存着与 CI 不同的 SNAPSHOT，且 `settings.xml` 镜像不同；边界是排查手段为 `mvn -U clean verify`（强制更新快照）、`mvn help:effective-pom` 对比生效配置、统一用 Maven Wrapper（`mvnw`）锁定 Maven 版本。
- **Q3：`-DskipTests` 与 `-Dmaven.test.skip=true` 的差别？** A：结论——前者编译测试代码但不执行，后者连测试代码都不编译；原因是 `maven.test.skip` 同时作用于 `testCompile` 阶段；边界是跳过测试只应用于紧急发布与本地提速，CI 主分支必须全量跑测试，且要注意 skip 后 `surefire` 报告缺失会导致门禁失效。
### 1.3 高频追问与陷阱
- `mirrorOf` 配成 `*` 后，若私服没有代理某个仓库会导致依赖直接拉不到（常见于配置了阿里云镜像又需要拉私有仓库）；规范是"私服 group 代理 central，`mirrorOf` 指向私服"。
- 本地仓库污染（下载中断产生的 `.lastUpdated` 文件、损坏 jar）会导致莫名报错，处理方式为删除对应目录或加 `-U` 强制刷新。
### 1.4 示例
```bash
mvn -v                                   # 确认 Maven 与 JDK 版本（构建问题第一步）
mvn clean package -DskipTests            # 打包，跳过测试执行
mvn clean install -U                     # 装本地仓库并强制更新 SNAPSHOT
mvn -o clean package                     # 离线构建（依赖已就绪时最快）
mvn -T 1C clean install                  # 并行构建（每个 CPU 核 1 线程）
mvn -pl order-service -am clean install  # 只构建指定模块及其依赖模块
mvn help:effective-pom > pom-effective.xml   # 查看继承与插值后的最终 POM
```
### 1.5 一句话速记
Maven 记三点：**GAV 定位构件、三生命周期内阶段顺序执行（validate→compile→test→package→verify→install→deploy）、查找顺序是本地仓库 → 私服/镜像 → central**。
## 2. 依赖机制与冲突排查
### 2.1 核心原理
| scope | 编译 | 测试 | 运行 | 打进包 | 传递给依赖方 |
| --- | --- | --- | --- | --- | --- |
| `compile`（默认） | ✓ | ✓ | ✓ | ✓ | 传递（compile） |
| `provided` | ✓ | ✓ | ✗ | ✗ | **不传递**（容器/JDK 提供，如 `servlet-api`） |
| `runtime` | ✗ | ✓ | ✓ | ✓ | 传递（runtime，如 JDBC 驱动） |
| `test` | ✗ | ✓ | ✗ | ✗ | **不传递**（如 JUnit） |
| `system` | ✓ | ✓ | ✓ | ✗ | 不传递，需写 `systemPath`，**不推荐**（不可移植） |
| `import` | — | — | — | — | 仅用于 `dependencyManagement` 导入 BOM |

- **传递依赖**：A → B → C 时，C 是否进入 A 取决于 **B 中 C 的 scope**（`compile`/`runtime` 才会传递，`provided`/`test` 不传），并按规则降级（B 中的 compile → A 中的 compile；runtime → runtime）。**依赖层级越深越"危险"**，这是冲突的根源。
- **依赖调解（Maven 3 规则，两条）**：① **最短路径优先**（`A→C` 深度 1 胜过 `A→B→C` 深度 2）；② **路径长度相同时，先声明者优先**（POM 中 `<dependencies>` 顺序在前的胜出，**父 POM 的声明先于子 POM**）。注意：**没有**"版本号高者优先"的规则——这是最常见的错误认知。
- **可选与排除**：`<optional>true</optional>` 让该依赖**不向外传递**（用于"库的可选能力"）；`<exclusions>` 排除某个传递依赖（常用于排除日志实现冲突、排除旧版 `commons-logging`）。
- **统一版本管理的三层手段**：① `dependencyManagement`（只声明版本，子模块引用时省略版本；也可用于**覆盖传递依赖的版本**）；② `parent` 继承（父 POM 抽公共配置，`pluginManagement` 同理管插件版本）；③ **BOM 导入**（`spring-boot-dependencies`、`spring-cloud-dependencies` 用 `import` scope 一次性管理整套版本）。
- **冲突排查命令**：`mvn dependency:tree`（看依赖树与 omitted 标记）、`-Dverbose`（显示被省略的冲突）、`-Dincludes=groupId:artifactId`（定位某依赖从哪来）、`mvn dependency:analyze`（找"用了却没声明"与"声明了没用"）、`mvn enforcer:enforce -Drules=requireUpperBoundDeps`（**直接失败于版本不一致**，可纳入 CI 门禁）。
### 2.2 面试问答
- **Q1：项目里 `NoSuchMethodError` / `ClassNotFoundException`，怎么判断是依赖冲突？** A：结论——通常是同一个类被两个版本提供（或运行时缺少该依赖）；原因是编译期用了 A 版本、运行期加载了 B 版本，方法签名不一致就抛 `NoSuchMethodError`；边界是先用 `mvn dependency:tree -Dincludes=冲突坐标` 定位来源，再用 `dependencyManagement` 或 `exclusions` 统一版本，最后 `mvn dependency:analyze` 复核；K8s 环境还要确认镜像里的 fat jar 用的是同一套版本。
- **Q2：为什么加了 `dependencyManagement` 就"自动升级"了传递依赖？** A：结论——`dependencyManagement` 对本模块生效的**所有**传递依赖版本都有约束力；原因是它优先于依赖调解的两条规则（相当于把版本"钉死"）；边界是它只对**本 POM（及继承它的子模块）**生效，且不能替代 `exclusions`（若被排除的依赖在传递链上必需，还是得显式声明）。
- **Q3：如何在 CI 中防止依赖冲突？** A：结论——用 enforcer 规则 + 依赖锁定 + 私服校验三重门禁；原因是人工 review 无法发现深层传递依赖的版本漂移；边界是 `requireUpperBoundDeps` 会在冲突时构建失败，初期可能产生大量"历史欠账"，需分批治理（先核心模块）；更严格的做法是生成 `dependency-reduced-pom` 或使用 Gradle/Maven 的依赖锁定文件。
### 2.3 高频追问与陷阱
- **同一依赖既被 `dependencyManagement` 管、又被 `<exclusions>` 排除**时容易误判：排除的是"传递路径的该依赖"，而 `dependencyManagement` 只管版本，不负责引入。
- `provided` 被"顺手改成 compile"是最常见的错误：会导致容器里出现两份 `servlet-api` 或与 Spring Boot 自带版本冲突，启动时报 `ClassCastException`/`NoSuchMethodError`。
- 依赖顺序真的影响结果（"第一声明优先"），因此 **POM 里不要随意重排依赖顺序**，改动顺序可能改变实际生效版本（建议用 enforcer 明确暴露）。
### 2.4 示例
```xml
<properties><spring-boot.version>3.2.5</spring-boot.version></properties>
<dependencyManagement>
  <dependencies>
    <!-- 导入 BOM（import scope 只用于 dependencyManagement） -->
    <dependency><groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId><version>${spring-boot.version}</version>
      <type>pom</type><scope>import</scope></dependency>
    <!-- 钉死传递依赖版本，覆盖依赖调解结果 -->
    <dependency><groupId>com.google.guava</groupId><artifactId>guava</artifactId>
      <version>32.1.3-jre</version></dependency>
  </dependencies>
</dependencyManagement>
<dependencies>
  <dependency><groupId>com.example</groupId><artifactId>common-utils</artifactId>
    <version>1.2.0</version>
    <exclusions>                                 <!-- 排除传递依赖，治理日志实现冲突 -->
      <exclusion><groupId>commons-logging</groupId><artifactId>commons-logging</artifactId></exclusion>
    </exclusions>
  </dependency>
</dependencies>
```
```bash
mvn dependency:tree -Dverbose -Dincludes=com.google.guava:guava   # 谁引入了 guava、哪个版本被省略
mvn dependency:analyze                                            # 未声明却使用 / 声明却未使用
mvn enforcer:enforce -Drules=requireUpperBoundDeps                 # 版本不一致直接构建失败
```
### 2.5 一句话速记
scope 记牢 **compile 全通、provided 不打包不传递、runtime 不编译要打包、test 只测试**；冲突调解是**最短路径优先 + 先声明优先（没有"高版本优先"）**，排查用 `dependency:tree`，治理用 `dependencyManagement`/BOM + enforcer 门禁。
## 3. 多模块工程与版本统一
### 3.1 核心原理
- **聚合（aggregation）vs 继承（inheritance）**：**聚合**解决"一次构建多个模块"，靠父 POM 的 `<modules>` 与反应堆（reactor）排序（按模块间依赖关系决定顺序）；**继承**解决"配置复用"，靠子模块 `<parent>` 继承 `properties`/`dependencyManagement`/`pluginManagement`。二者可同时使用（父 POM 既 `packaging=pom` 又含 `<modules>`），也可只用一个：**只有继承没有聚合**时，子模块要单独构建；**只有聚合没有继承**时，配置无法复用。
- **模块划分建议**：按"变更频率与依赖方向"划分，常见分层 `common`（工具/DTO）→ `domain`（领域模型）→ `dal`（数据访问）→ `service`（业务）→ `web/api`（入口），再加 `starter`/`client`（对外 SDK）。原则：**下层不依赖上层**、避免循环依赖（Maven 会直接报 `The projects in the reactor contain a cyclic reference`）、公共能力下沉但不要做"万能 common"（会导致所有模块被迫一起发版）。
- **版本统一（三种主流做法）**：① 父 POM 用 `<revision>` 属性 + 子模块统一引用（Maven 3.5+ 支持 `${revision}`，配合 `flatten-maven-plugin` 让发布时替换为真实版本）；② 所有模块同版本号统一升（simple release 流程，`mvn versions:set -DnewVersion=1.3.0`）；③ 各模块独立版本 + BOM 统一对外（适合对外 SDK）。
- **依赖与插件版本也要统一**：`dependencyManagement` 管依赖版本、`pluginManagement` 管插件版本，避免各模块各写一份导致构建行为不一致。
### 3.2 面试问答
- **Q1：聚合和继承的区别，能不能只做其中一个？** A：结论——能，二者目的不同：聚合管构建范围与顺序，继承管配置复用；原因是 Maven 把它们设计成正交能力（`<modules>` 与 `<parent>` 是两套机制）；边界是实践中多用"父 POM 同时聚合 + 被继承"，但要避免**为了继承而层层嵌套父 POM**（多层 parent 会让 `effective-pom` 难以排查）。
- **Q2：模块多了构建慢怎么办？** A：结论——用反应堆裁剪 + 并行 + 缓存；原因是全量构建把无关模块也编了一遍；边界是 `mvn -pl api -am`（构建 api 及其依赖）、`-amd`（构建 api 及其被依赖者）、`-T 1C` 并行、CI 里按变更模块计算构建范围（`git diff` 判断），并缓存 `~/.m2`；注意并行构建要求模块间无隐式文件依赖。
- **Q3：多模块项目怎么保证依赖版本一致？** A：结论——父 POM 的 `dependencyManagement` 是唯一版本来源，子模块一律不写版本；原因是一旦子模块写版本就会覆盖父级约束、造成漂移；边界是第三方 BOM（Spring Boot）先 import、再在其上覆盖特殊依赖，并用 enforcer 的 `dependencyConvergence` 规则在 CI 中兜底。
### 3.3 高频追问与陷阱
- 父 POM 里写 `<dependencies>`（而非 `dependencyManagement`）会让**所有子模块无条件继承该依赖**，是"包里莫名多出依赖"的常见原因。
- 循环依赖必须先拆模块或抽接口，不能靠"改成 `provided` 绕过去"。
### 3.4 示例
```xml
<!-- 父 POM：packaging 必须是 pom，同时聚合 + 被继承 -->
<project>
  <groupId>com.example</groupId><artifactId>order-platform</artifactId>
  <version>1.3.0</version><packaging>pom</packaging>
  <modules>
    <module>order-common</module>
    <module>order-domain</module>
    <module>order-service</module>
    <module>order-api</module>
  </modules>
  <properties><revision>1.3.0</revision><java.version>17</java.version></properties>
  <dependencyManagement><!-- 统一版本：子模块引用时不写 version --></dependencyManagement>
  <build><pluginManagement><!-- 统一插件版本 --></pluginManagement></build>
</project>
```
### 3.5 一句话速记
**聚合管构建顺序、继承管配置复用**；版本统一到父 POM 的 `dependencyManagement`/`pluginManagement`（`${revision}` + flatten 做 CI 友好版本），下层不依赖上层、禁止循环依赖。
## 4. 插件机制、私服发布与构建加速
### 4.1 核心原理
- **插件与生命周期**：Maven 的每个阶段都靠"绑定到该阶段的目标（goal）"完成工作，**默认绑定**来自 `packaging`（jar 包默认绑定 `maven-jar-plugin:jar` 到 `package`、`maven-surefire-plugin:test` 到 `test`）；也可通过 `<executions><execution><phase>` 自定义绑定，用 `mvn 插件前缀:目标` 直接执行某个 goal（如 `mvn dependency:tree`）。
- **常用插件清单**：

| 插件 | 用途与要点 |
| --- | --- |
| `maven-compiler-plugin` | 编译；Spring Boot 3 必须 `release=17`（用 `<release>` 而非 `source/target`） |
| `maven-surefire-plugin` | 单元测试（`*Test`）；`failsafe` 负责集成测试（`*IT`，绑在 `verify`） |
| `maven-jar-plugin` / `war-plugin` | 打普通 jar / war |
| `maven-shade-plugin` | 打 fat jar，支持依赖 relocation（改包名避免冲突） |
| `maven-assembly-plugin` | 自定义分发（zip/tar + 目录结构 + 启动脚本） |
| `spring-boot-maven-plugin` | `repackage` 生成可执行 jar（`spring-boot-dependencies` 管理版本） |
| `dockerfile-maven-plugin` / `jib-maven-plugin` / `spring-boot:build-image` | 构建并推送镜像（jib 无需 Dockerfile） |
| `maven-release-plugin` | 发布版本：改版本 → commit → tag → deploy → 改回 SNAPSHOT |
| `flatten-maven-plugin` | 配合 `${revision}` 生成可发布 POM（替换占位符） |
| `maven-enforcer-plugin` | 规则门禁：JDK 版本、依赖收敛、禁用 SNAPSHOT 依赖 |
| `jacoco` / `checkstyle` / `spotbugs` / `pmd` | 覆盖率与静态质量门禁（详见 [设计模式与代码规范](./01-设计模式与代码规范.md)） |

- **私服 Nexus 发布流程**：POM 配 `distributionManagement`（releases/snapshots 两个地址）→ `settings.xml` 配 `<server>`（`id` 与上面对应，用户名密码可来自环境变量）→ `mvn clean deploy`。**SNAPSHOT 可被多次覆盖（Nexus 用时间戳隔离）**，release 版本不可覆盖；生产依赖禁止使用 SNAPSHOT。
- **构建加速手段**：① 并行 `-T 1C`；② 离线 `-o`、跳过无用阶段（`-pl -am`）；③ 跳过测试（仅本地/紧急发布）；④ 缓存 `~/.m2`（CI 缓存或 BuildKit `--mount=type=cache`）；⑤ 镜像源用私服（避免外网拉取）；⑥ 用 `useIncrementalCompilation`（默认开启，注意其失效场景）；⑦ 大型项目考虑 Gradle（增量构建 + 构建缓存 + configuration cache）。
- **Gradle 对比**：Groovy/Kotlin DSL、增量与守护进程（daemon）、构建缓存与扫描（build scan）带来更好性能与灵活性；代价是**没有强约定**（插件与写法多样、脚本易失控）。结论：**Maven 约定强、稳定、团队协作成本低；Gradle 灵活与快，适合大型多模块/Android/需要精细定制构建的场景**，二者能力上都够用，选型看团队而非性能（除非构建确实到了瓶颈）。
### 4.2 面试问答
- **Q1：`mvn deploy` 报 401/403 怎么排查？** A：结论——九成是 `distributionManagement` 的 `id` 与 `settings.xml` 的 `<server id>` 不匹配（或密码错误）；原因是 Maven 靠 `id` 关联凭据，找不到就用匿名请求；边界是还要确认目标仓库类型与 `<url>` 是否对应（release 版本推 snapshots 地址也会被拒），以及 Nexus 上该账号是否有对应仓库的写权限。
- **Q2：什么时候用 `shade`，什么时候用 `assembly`，什么时候用 `spring-boot-maven-plugin`？** A：结论——可执行 Spring Boot 应用用 `spring-boot-maven-plugin`；需要可移植的独立工具 jar（含依赖、改包名）用 `shade`；需要带目录结构与脚本的分发包用 `assembly`；原因是三者产出形态不同（Boot 的可执行 jar 有特殊的 `BOOT-INF` 布局，不能当普通 library 依赖）；边界是 **Boot 应用被其他模块依赖时要额外配置 `classifier`**（生成 `-plain.jar`），否则依赖方无法解析类。
- **Q3：构建慢怎么优化？** A：结论——先测量再优化（`mvn -X` 或 `--profiler`、看各插件耗时），再按 依赖下载 → 测试 → 编译 顺序优化；原因是构建耗时大头通常是依赖下载（网络）与测试（串行）；边界是"跳过测试"不是优化而是风险转移，正确做法是拆分单元测试（快）与集成测试（慢，`failsafe` 放 `verify`），CI 分阶段执行；另外并行构建对单模块无效，多模块才有收益。
### 4.3 高频追问与陷阱
- `maven-release-plugin` 会创建 tag 并推送到 SCM，需要正确配置 `<scm>`，否则报"cannot prepare release"；CI 环境要关闭交互（`-B`）并配置好 git 用户。
- 用 `versions:set` 批量改版本后忘记提交/推送会导致依赖方拉不到新版本；发布流程建议交给 CI 固化。
### 4.4 示例
```xml
<build><pluginManagement>                              <!-- 统一插件版本，避免各模块漂移 -->
  <plugins>
    <plugin><groupId>org.apache.maven.plugins</groupId><artifactId>maven-compiler-plugin</artifactId>
      <configuration><release>17</release><parameters>true</parameters></configuration></plugin>
    <plugin><groupId>org.apache.maven.plugins</groupId><artifactId>maven-enforcer-plugin</artifactId>
      <executions><execution><id>enforce</id><goals><goal>enforce</goal></goals>
        <configuration><rules>
          <requireJavaVersion><version>[17,)</version></requireJavaVersion>
          <requireUpperBoundDeps/><!-- 依赖版本冲突即失败，纳入 CI 门禁 -->
        </rules></configuration></execution></executions></plugin>
  </plugins>
</pluginManagement></build>
```
```bash
mvn clean deploy -B -DskipTests          # 发布到 Nexus（版本决定 releases/snapshots）
mvn versions:set -DnewVersion=1.4.0 -DgenerateBackupPoms=false   # 统一改版本
mvn -pl order-api -am -T 1C clean install -o                     # 裁剪 + 并行 + 离线
```
### 4.5 一句话速记
插件靠**阶段绑定**工作（`packaging` 给默认绑定，`<executions>` 自定义）；发布三步是 **`distributionManagement` + `settings.xml`（id 必须一致）+ `mvn deploy`**，SNAPSHOT 不上生产；加速靠 **裁剪 + 并行 + 缓存 + 私服**。
## 5. Git 核心模型与命令速查
### 5.1 核心原理
- **三大区域**：**工作区（Working Directory）** → `git add` → **暂存区（Index/Stage）** → `git commit` → **版本库（Repository/HEAD）** → `git push` → **远程仓库**。文件状态：`untracked`（未跟踪）、`modified`（已修改未暂存）、`staged`（已暂存）、`committed`（已提交）。
- **HEAD 与分支**：分支只是指向某个提交的可变指针，`HEAD` 指向当前分支；提交是"快照 + 父指针"，因此历史是 DAG（有向无环图），rebase/cherry-pick 的本质都是"复制提交生成新对象"。
- **命令速查表**：

| 场景 | 命令 |
| --- | --- |
| 查看状态/差异 | `git status -sb`、`git diff`、`git diff --cached`（暂存区 vs HEAD）、`git diff HEAD~1` |
| 提交 | `git add -p`（分块暂存，评审前最爱）、`git commit -m`、`git commit --amend`（改最后一次提交） |
| 历史 | `git log --oneline --graph --decorate --all`、`git show <sha>`、`git blame -L 10,20 file`、`git shortlog -sn` |
| 撤销（工作区） | `git restore file` / `git checkout -- file`（丢弃未暂存改动） |
| 撤销（暂存区） | `git restore --staged file` / `git reset HEAD file` |
| 撤销（提交） | `git reset --soft/--mixed/--hard <sha>`、`git revert <sha>` |
| 分支 | `git switch -c feature/x`、`git branch -vv`、`git branch -d/-D`、`git switch -` |
| 同步 | `git fetch --prune`、`git pull --rebase`、`git push -u origin feature/x`、`git push --force-with-lease` |
| 暂存现场 | `git stash -u`、`git stash list`、`git stash pop`、`git stash show -p` |
| 挑选提交 | `git cherry-pick <sha>`、`git cherry-pick -n`（不立即提交） |
| 标签 | `git tag -a v1.3.0 -m "release"`、`git push origin --tags`、`git tag -d` |
| 考古恢复 | `git reflog`、`git bisect start/good/bad`、`git fsck --lost-found` |

### 5.2 面试问答
- **Q1：`git fetch` 和 `git pull` 有什么区别？** A：结论——`fetch` 只更新远程跟踪分支（`origin/main`），`pull = fetch + merge/rebase`；原因是分开执行可以先 review 差异再决定合并方式；边界是团队规范推荐 `git pull --rebase`（保持线性历史），但要确保本地提交没被别人基于；`git fetch --prune` 还能清理已删除的远程分支引用。
- **Q2：`git reset` 的三种模式？** A：结论——`--soft` 只移动 HEAD（改动留在暂存区）、`--mixed`（默认）移动 HEAD 并重置暂存区（改动留在工作区）、`--hard` 三者全改（工作区改动丢失）；原因是它们对"HEAD/暂存区/工作区"三个区域的改动范围递增；边界是 `--hard` 只适用于**未推送**的本地历史，公共分支只能 `revert`。
- **Q3：`git revert` 和 `git reset` 该用哪个？** A：结论——公共分支用 `revert`，本地未推送用 `reset`；原因是 `revert` 生成一个新提交做反向修改、不改写历史，别人 pull 不会冲突；边界是 `revert` 一个 merge 提交需要 `-m 1` 指定父提交；如果误提交了密钥，`revert` 并不能让密钥从历史消失（必须改写历史 + **立刻轮换密钥**）。
### 5.3 高频追问与陷阱
- `.gitignore` 对**已跟踪文件无效**：必须 `git rm --cached <file>` 后再提交；`git check-ignore -v <file>` 可确认规则来源。
- `--force` 会覆盖远程历史，规范是用 **`--force-with-lease`**（若远端已被他人更新则拒绝推送，避免覆盖同事提交）。
- `git commit --amend` 会改写提交 hash，**已推送的提交不要 amend**（等价于重写公共历史）。
### 5.4 示例
```bash
git status -sb                                   # 看当前分区状态
git add -p                                       # 分块暂存，避免把调试代码一起提交
git log --oneline --graph --decorate --all -15    # 一眼看清分支与合并关系
git fetch --prune && git rebase origin/main       # 变基前先同步远端
git switch -c fix/order-timeout && git push -u origin fix/order-timeout
git tag -a v1.3.0 -m "release 1.3.0" && git push origin v1.3.0
```
### 5.5 一句话速记
Git 记三区（**工作区 → 暂存区 → 版本库**）+ 两撤销（**未推送用 reset、已推送用 revert**）+ 一根救命稻草（**reflog**）；`add -p` 是提交质量的起点。
## 6. 分支模型与 merge / rebase
### 6.1 核心原理
| 模型 | 分支结构 | 适用 | 代价 |
| --- | --- | --- | --- |
| Git Flow | `main` + `develop` + `feature/*` + `release/*` + `hotfix/*` | 版本化交付（客户端、有发布窗口） | 分支多、流程重、合并成本高 |
| GitHub Flow | `main` + 短生命周期 `feature/*` + PR | 持续交付的 Web 服务 | 依赖完善 CI 与快速回滚 |
| Trunk Based | 只有主干 + 极短分支（<1 天）+ Feature Flag | 高成熟度团队（Google/微软实践） | 对测试自动化与开关治理要求极高 |
| 混合（国内主流） | `master`（生产）+ `develop`/`test` + `feature/*` + `hotfix/*` | 有测试/预发环境的中大型团队 | 需要严格的合并方向约定 |

- **merge vs rebase**：`merge` 把两条线合并产生一个**合并提交**，保留真实分支历史（可追溯分支何时并入），但历史图会变复杂；`rebase` 把当前分支的提交**按新基点重放**，得到线性历史、便于 `git log` 阅读与二分定位，但**提交 hash 全部变化**。
- **黄金法则**：**不要 rebase 已经推送到公共分支的提交**（会与协作者的本地历史冲突，导致他们需要强行重置）。安全用法：本地 `feature` 分支 rebase `main` 后再推、或对 `feature` 分支用 `--force-with-lease`（个人分支）。
- **合并策略选择**：小步快跑的 `feature → main` 推荐 `rebase + fast-forward`（或 PR squash）；长期分支（release/hotfix）保留 `merge` 以留下合并记录；团队内必须**统一一种**，否则历史图会很混乱。
- **cherry-pick**：把某个提交单独挑到另一分支，典型场景是"把已修复的 bug 从 `develop` 挑到 `main` 紧急发版"；代价是产生**重复提交**（同一改动两个 hash），后续合并可能冲突，能合并就合并。
- **冲突解决流程**：`git status` 定位冲突文件 → 打开文件处理 `<<<<<<< ======= >>>>>>>` 标记（保留正确逻辑，必要时合并双方改动）→ `git add <file>` → merge 用 `git commit` 完成、rebase 用 `git rebase --continue`；中途想放弃用 `git merge --abort` / `git rebase --abort`。预防手段：**小步提交、勤同步主干、避免多人改同一文件同一区域**。
- **其他能力**：`git submodule`（在主仓库里固定引用另一个仓库的某个 commit，需 `--init --recursive` 初始化，升级要显式提交引用变更；轻量替代是 `git subtree`）、**Git 钩子**（`pre-commit` 做格式化/lint、`commit-msg` 做提交信息校验（commitlint）、`pre-push` 跑轻量测试；注意钩子不随 clone 分发，需借助脚本安装或用 Husky/`core.hooksPath`）。
### 6.2 面试问答
- **Q1：你们团队用 merge 还是 rebase？为什么？** A：结论——个人特性分支 rebase 主干、合入主干用 squash/rebase，主干保持线性；原因是这样既能快速消除分叉冲突，又能保持 `git log` 可读与可二分；边界是 release/hotfix 这类长期分支用 merge 保留合并记录，并且**统一约定比选哪种更重要**；另外若无 squash，PR 里的"修 typo"这类提交会污染主干历史。
- **Q2：rebase 之后为什么同事的提交"消失"了？** A：结论——rebase 重写了 hash，同事若基于旧 hash 继续开发，推送时会被拒绝或产生重复提交；原因是他本地记录的是旧提交对象；边界是修复方式为同事 `git fetch` 后 `git rebase --onto` 或 `git reset --hard origin/branch`（确认本地无独有提交），所以**公共分支严禁 rebase**。
- **Q3：怎么避免大规模冲突？** A：结论——缩短分支生命周期 + 频繁同步主干 + 拆分文件与职责；原因是冲突规模与"分叉时间 × 改动重叠度"成正比；边界是结构性手段更重要（微服务/模块拆分、CODEOWNERS 划分归属），工具层面可以开启 `git rerere`（记录冲突解决方案并复用）与 `merge.conflictStyle=zdiff3`（显示共同祖先，便于判断）。
### 6.3 高频追问与陷阱
- `git pull` 默认的合并策略受 `pull.rebase` 配置影响，多人协作时**不统一配置就会出现合并提交与线性历史混用**。
- `--force-with-lease` 仍会覆盖别人的提交（如果本地已 fetch 到对方提交则会被拒），它是"安全阀"而不是"免死金牌"。
### 6.4 示例
```bash
# 特性分支保持线性的标准流程
git switch -c feature/order-export
git fetch origin && git rebase origin/main          # 变基到最新主干
git add -p && git commit -m "feat(order): 支持订单导出 CSV"   # 小步提交
git fetch origin && git rebase origin/main          # 提交前再同步一次
git push -u origin feature/order-export --force-with-lease    # 个人分支可强推（带 lease）
# 合入后清理
git switch main && git pull --ff-only && git branch -d feature/order-export
```
### 6.5 一句话速记
分支模型选型看"发布节奏"（Git Flow 重流程、GitHub Flow 持续交付、Trunk Based 高成熟度）；**rebase 换线性历史、merge 保真实关系，公共分支严禁 rebase**；冲突靠短分支 + 勤同步预防。
## 7. Git 事故恢复实战
### 7.1 核心原理
- **万能钥匙 `git reflog`**：记录本地 HEAD 的所有移动（包括 reset、rebase、checkout、commit），默认保留 90 天；只要提交对象还在（未被 GC），就能按 hash 恢复。`git reflog show <branch>` 看分支的移动历史，`git reset --hard HEAD@{2}` 回到某个状态。
- **四类事故的标准处理（结论）**：

| 事故 | 场景 | 处理步骤 |
| --- | --- | --- |
| 误 commit（未推送） | 提交信息写错、多提了文件、想拆成两个提交 | `git commit --amend`（改信息/补文件）；想撤销但保留改动：`git reset --soft HEAD~1`；撤销并丢弃改动：`git reset --hard HEAD~1` |
| 误 `reset --hard` | 本地改动/提交被清掉 | 立即 `git reflog` 找到目标 sha（看 `HEAD@{n}` 描述）→ `git reset --hard <sha>` 或 `git cherry-pick <sha>`；有 `ORIG_HEAD` 时可直接 `git reset --hard ORIG_HEAD` |
| 误删分支 | `git branch -D` 删了未合并分支 | `git reflog`（或 `git branch <name> <sha>` 用 reflog 里的 sha 重建）；若曾推送到远端：`git fetch origin <name>` 后 `git switch -c <name> origin/<name>` |
| 误 push 到公共分支 | 推了错误提交/密钥 | **不要 reset + force**（会破坏他人历史），用 `git revert <sha>` 生成反向提交；若必须抹除（密钥泄露）则改写历史（`git filter-repo` / BFG）+ **立即轮换密钥**（最关键）+ 通知所有协作者重新 clone |

- **补充手段**：`git stash list`（误切分支前 stash 的改动可找回）、`git fsck --lost-found`（找悬空提交/对象）、`git reflog` 找不到时检查 CI/远端是否有该提交（远端引用往往比本地更稳）。
- **预防措施**：分支保护（禁止直推主干、禁强推）、`pre-commit` 钩子拦密钥与格式化、提交前 `git status/diff` 复核、重要操作前创建临时 tag（`git tag backup-before-rebase`）。
### 7.2 面试问答
- **Q1：不小心 `git reset --hard` 丢了一个小时的代码，能救回来吗？** A：结论——能，只要 commit 过就能救；原因是 reset 只是移动引用，提交对象仍在本地对象库中；边界是**从未 commit 过的未暂存改动无法恢复**（这是最需要强调的边界）；立即停止写操作，用 `git reflog` 找到 `reset` 之前的 `HEAD@{n}` 再 `git reset --hard` 回去；如果改动从未 add 过，只能靠 IDE 的本地历史（IntelliJ Local History）做最后一道兜底。
- **Q2：误把密钥提交并推到远端怎么办？** A：结论——第一步轮换密钥，第二步再清理历史；原因是历史里的旧密钥永远可能被他人 clone/fork 拿到，仅删文件不解决问题；边界是清理历史要全员协调（改写 hash 意味着所有人必须重新 clone 或硬重置），可用 `git filter-repo --path .env --invert-paths` 或 BFG，并同步在 CI 中加"密钥扫描"门禁（gitleaks/trufflehog）防复发。
- **Q3：同事的提交被 force push 覆盖了怎么办？** A：结论——从他的本地 `reflog` 恢复或重新推送他的提交；原因是被覆盖的提交仍在其本地对象库里；边界是若对方本地也丢了，只能从 CI 的构建记录、其他同事的 clone 或远端镜像找回；根治手段是主干分支保护 + 禁用强推（GitLab/GitHub 均可配置）。
### 7.3 高频追问与陷阱
- `git reflog` 是**本地**记录：换机器/重新 clone 后无法恢复别人的历史，所以"重要分支尽快推到远端"本身就是一种备份。
- `git reset --hard` 之前务必确认没有未提交改动（`git status` 干净），或先 `git stash -u`。
### 7.4 示例
```bash
git reflog                                        # 找到误操作前的 HEAD 位置
# 3d2f1a9 HEAD@{0}: reset: moving to HEAD~3
# 9c8b7a6 HEAD@{1}: commit: feat(order): 导出功能   ← 目标
git reset --hard 9c8b7a6                          # 恢复该提交（仅限未推送）
git branch hotfix/restore 9c8b7a6                 # 或先建分支保命，再决定怎么合
git revert --no-edit <bad-sha>                    # 已推送的错误提交：反向提交（不改历史）
git tag backup-before-rebase                      # 危险操作前的兜底锚点
```
### 7.5 一句话速记
Git 自救口诀：**commit 过就能救（reflog 是万能钥匙），没 commit 就认命**；未推送用 reset、已推送用 revert，密钥泄露**先轮换再清历史**。
## 8. 提交规范、PR 与代码评审
### 8.1 核心原理
- **Conventional Commits（提交规范）**：`<type>(<scope>): <subject>`，如 `feat(order): 支持订单导出 CSV`。常用 type：`feat`（新功能，对应 MINOR）、`fix`（缺陷修复，对应 PATCH）、`docs`、`style`、`refactor`、`perf`、`test`、`build`、`ci`、`chore`、`revert`；**破坏性变更**在 type 后加 `!` 或在 footer 写 `BREAKING CHANGE: 说明`（对应 MAJOR）。价值：可读的变更日志（`git log`）、自动生成 CHANGELOG 与版本号（semantic-release/standard-version）、按 type 触发不同的 CI 流程。
- **提交粒度**：一个提交只做一件事（便于 revert 与 bisect）；避免"杂糅提交"（重构 + 功能 + 格式化混在一起），格式化的批量改动**单独一个提交**，否则 review 全是噪音。
- **issue 关联**：提交或 PR 描述里写 `Closes #123` / `Refs #123`（GitHub/GitLab 会自动关闭与关联问题），便于从代码追溯到需求与缺陷来源。
- **PR 描述模板（五项必填）**：① **背景与目标**（解决什么问题、关联 issue）；② **改动内容**（关键设计取舍、是否有破坏性变更）；③ **影响面与风险**（涉及的表/接口/配置/依赖）；④ **验证方式**（单测/集成/压测结果，最好附截图或指标）；⑤ **回滚方案**（怎么快速止血）。
- **代码评审流程**：作者自查（`git diff` 复核、本地测试、`add -p` 拆提交）→ 提交 PR（小 PR：**< 400 行**最易被认真评审）→ CI 门禁（编译/单测/静态扫描/覆盖率）→ 评审人按"正确性 → 可读性 → 设计 → 性能与安全"的顺序看（详见 30 条清单见 [设计模式与代码规范](./01-设计模式与代码规范.md)）→ 全量 approve 后合并（squash 保持主干整洁）→ 合入后确认部署与监控。
- **分支保护策略（主干必须开的开关）**：禁止直接 push（必须走 PR）、要求至少 N 个 approve（含 CODEOWNERS 指定人）、要求 CI 全部通过、要求分支与目标分支同步（避免"过期分支"合并）、禁止强推与删除分支、要求签名提交（可选）、要求线性历史（可选）。
### 8.2 面试问答
- **Q1：怎么保证代码质量？** A：结论——用"规范 + 工具门禁 + 评审 + 测试 + 监控"五层闭环，而不是靠个人自觉；原因是人的注意力不可靠、规则必须自动化执行；边界是每一层都要有明确的失败阻断（如 checkstyle 不过则构建失败），否则形同虚设；再叠加线上的监控与告警形成反馈（缺陷回到测试补充用例）。
- **Q2：PR 太大怎么办？** A：结论——拆；原因是超过 400-500 行的 PR 评审质量会断崖式下降（大 PR 的问题发现率显著更低）；边界是拆分要按"可独立合入的纵向切片"（如接口 + 实现 + 单测一起），而不是按文件横向拆（会导致中间状态无法编译）；若无法拆（如大范围重构），提前同步设计文档 + 结对评审。
- **Q3：评审时最看重什么？** A：结论——正确性与边界（逻辑、并发、异常、幂等），其次可维护性，最后风格；原因是风格能靠工具自动修，而逻辑错误只有人能发现；边界是评审要区分"必须改（bug/安全/性能）"与"建议（风格/命名）"，避免把评审变成风格争论；同时要求作者给出验证证据（测试、压测数据），而非"我本地试过了"。
### 8.3 高频追问与陷阱
- 只有 CI 通过才算通过，人工"口头 approve"在分支保护下无法合并；因此**门禁必须放在 CI 而不是靠约定**。
- 提交信息写"update""fix bug"会让 `git log` 与 `bisect` 完全失效；团队应通过 `commit-msg` 钩子强制校验格式。
### 8.4 示例
```text
feat(order)!: 订单接口改为分页返回                       ← ! 表示破坏性变更
[空行]
BREAKING CHANGE: /api/v1/orders 返回结构由数组改为 {items, total}，
调用方需按新契约解析；老接口保留至 v2 上线后 1 个月。

Closes #482
```
```yaml
# .gitlab-ci.yml 片段：门禁不通过不允许合并
stages: [verify, test]
verify:
  stage: verify
  script: [ "mvn -B -q clean compile", "mvn -B checkstyle:check" ]
test:
  stage: test
  script: [ "mvn -B test", "mvn -B jacoco:report" ]
  rules: [ { if: '$CI_PIPELINE_SOURCE == "merge_request_event"' } ]   # MR 触发即跑门禁
```
### 8.5 一句话速记
提交用 **Conventional Commits**（type+scope+subject，破坏性变更写 `BREAKING CHANGE`）；PR 描述必含**背景/改动/影响/验证/回滚**且保持小体量；分支保护靠 **禁止直推 + 必须 approve + CI 全绿**。
## 9. 多环境配置、语义化版本与 API 文档
### 9.1 核心原理
- **多环境配置（Spring 侧三层落地）**：① **Profile**（`application-{dev,test,prod}.yml` + `spring.profiles.active`）区分环境；② **配置中心**（Nacos/Apollo）管理可变配置并支持灰度与回滚；③ **环境变量/K8s ConfigMap+Secret** 覆盖敏感与部署期参数。优先级：命令行 > 环境变量 > 配置中心/外部配置 > jar 内 profile 配置 > 默认配置。详见 [SpringBoot与自动装配](../03-框架/02-SpringBoot与自动装配.md) 与 [CICD与监控告警](./03-CICD与监控告警.md)。
- **语义化版本（SemVer 2.0）**：`MAJOR.MINOR.PATCH`——不兼容的 API 变更升 MAJOR、向下兼容的新增功能升 MINOR、向下兼容的缺陷修复升 PATCH；预发布版本用 `-SNAPSHOT`/`-rc.1`（Maven 中 SNAPSHOT 表示可覆盖的开发版）。
- **API 兼容性判断（结论）**：**可以做**（向后兼容）——新增可选字段、新增接口、放宽入参校验、新增枚举值（对使用方需容忍未知值）；**不可做（破坏性）**——删字段/接口、改字段类型或语义、把可选改必填、收窄入参范围、改错误码语义；HTTP API 的破坏性变更用**新版本路径**（`/v2/...`）并行一段时间后再下线 v1（配合调用方统计与废弃公告）。**消费者驱动契约测试（Pact）**可以在提供方改动时自动发现对调用方的破坏。
- **API 文档**：**OpenAPI 3** 规范（`openapi.yaml` / `@Operation` 注解生成）；Spring Boot 3 用 **springdoc-openapi**（`springdoc-openapi-starter-webmvc-ui`，访问 `/swagger-ui.html`），Spring Boot 2.6+ 之后的旧方案 **springfox 已不推荐**（与 PathPatternParser 兼容性问题）；**Knife4j** 是基于 springdoc/springfox 的增强 UI（文档聚合、调试、鉴权、离线导出），国内团队常用。常用注解（springdoc）：`@Operation(summary)`、`@Parameter`、`@Schema`（字段说明与示例）、`@ApiResponse`、`@Tag`；多分组用 `GroupedOpenApi`。
- **接口 Mock**：基于 OpenAPI 契约生成 Mock 服务（Prism、Apifox/ApiPost、MockServer、WireMock），让前后端并行开发；WireMock 更常用于**集成测试**中模拟第三方依赖（可断言请求、模拟延迟与故障）。
### 9.2 面试问答
- **Q1：版本号怎么定？** A：结论——对外发布的构件用 SemVer，内部模块用统一版本或 `${revision}` 由 CI 统一升；原因是调用方依赖"版本号语义"来判断升级风险；边界是**开发中一律 `-SNAPSHOT`、发布去后缀**，且每次发布都要打 git tag（与镜像 tag、部署记录对齐），否则无法回答"线上跑的是哪个版本"。
- **Q2：接口要加字段，怎么保证不影响老调用方？** A：结论——只加可选字段、不动既有字段语义，并在客户端要求"忽略未知字段"；原因是大多数序列化框架默认忽略未知字段，但严格模式（如部分强类型客户端）会报错；边界是若必须改语义（如金额单位从元改分），就得新增字段并双写过渡，或走新版本路径；上线前用契约测试与灰度验证。
- **Q3：Swagger 在生产环境暴露安全吗？** A：结论——不建议公开暴露；原因是文档会把全部接口、参数、示例甚至内部字段结构暴露给攻击者；边界是生产上关闭 UI（`springdoc.swagger-ui.enabled=false`）或用网关鉴权/内网访问，仅保留 OpenAPI JSON 供内部平台消费；同时注意文档里的示例不要写真实密钥与用户数据。
### 9.3 高频追问与陷阱
- `SNAPSHOT` 依赖被带进生产包是最隐蔽的问题：用 enforcer 的 `bannedDependencies` 规则禁止 `*-SNAPSHOT`，从构建层面拦住。
- 枚举新增值对调用方是"隐式破坏"（对方 switch 未覆盖默认分支会抛异常），因此契约里要明确"调用方必须处理未知枚举"。
### 9.4 示例
```yaml
# application.yml 分层 + 环境变量覆盖（部署期参数不写死在包内）
spring:
  profiles: { active: ${SPRING_PROFILES_ACTIVE:dev} }
  datasource:
    url: ${DB_URL:jdbc:mysql://localhost:3306/order?useSSL=false}
    password: ${DB_PASSWORD:}
management:
  endpoints: { web: { exposure: { include: health,info,prometheus } } }
---
# 生产 profile 关闭文档 UI，避免暴露接口细节
springdoc: { swagger-ui: { enabled: false }, api-docs: { enabled: false } }
```
```java
// springdoc + Knife4j：契约即文档，注解补全说明与示例
@Operation(summary = "创建订单", description = "幂等：同一 requestId 重复提交返回同一订单")
@PostMapping("/orders")
public OrderVO create(@Valid @RequestBody @Parameter(description = "请求体") CreateOrderCmd cmd) { ... }
```
### 9.5 一句话速记
配置三层落地（**Profile + 配置中心 + 环境变量/Secret**）；版本用 **SemVer**（不兼容升 MAJOR、SNAPSHOT 不上生产）；接口只做**向后兼容**变更，破坏性变更走新版本路径；文档用 **springdoc + OpenAPI 3（Knife4j 增强）** 且生产不公开。
## 10. 工程实践高频面试问答
### 10.1 核心原理
本节把散落在前九节的"工程实践类开放题"集中成可直接背诵的答法，通用结构是：**先给结论 → 再说方案与工具 → 最后给边界与取舍**。
### 10.2 面试问答
- **Q1：项目里出现依赖冲突怎么解决？** A：结论——定位 → 统一 → 防护三步；原因是冲突本质是"同一坐标多版本"，必须先知道谁引入；边界是具体操作为 `mvn dependency:tree -Dverbose` 定位来源 → 在父 POM 的 `dependencyManagement` 里钉死版本（或 `exclusions` 排除错误来源）→ 用 enforcer 的 `requireUpperBoundDeps`/`dependencyConvergence` 纳入 CI 门禁防复发；若冲突在 fat jar 里，还要核对 shade relocation 与镜像内的 jar。
- **Q2：你们怎么做灰度发布？** A：结论——按"流量比例 + 用户维度"分层放量，并让监控能按版本维度对比；原因是灰度能否安全取决于"能否判定异常"与"能否快速回滚"；边界是落地实现有三条路（网关按权重/Header 分流、K8s 双 Deployment 按副本比例、服务网格/配置中心规则），关键配套是：灰度标要全链路透传（同一用户始终命中同一版本）、指标按 version 打标签、异常自动回滚（Argo Rollouts/Flagger）与功能开关兜底。详见 [CICD与监控告警](./03-CICD与监控告警.md)。
- **Q3：多模块项目的依赖与版本怎么管理？** A：结论——父 POM 的 `dependencyManagement` 是唯一版本源，子模块只声明坐标不写版本；原因是避免版本漂移（同一依赖在不同模块用不同版本会导致运行时 `NoSuchMethodError`）；边界是引入的第三方 BOM（Spring Boot/Spring Cloud）先 `import`、需要覆盖的再显式声明，并用 `${revision}` + flatten 统一发版；对外的 SDK 模块单独维护版本并保持 API 兼容。
- **Q4：如何保证代码质量与构建稳定性？** A：结论——把规则变成会失败的自动化门禁；原因是约定无人执行等于没有约定；边界是具体分层为：格式与静态检查（spotless/checkstyle/spotbugs，提交前 + CI 双跑）、测试（单测 + 集成测试 + 覆盖率阈值）、依赖门禁（enforcer）、安全（密钥扫描、依赖漏洞扫描）、构建可复现（锁 Maven 版本、固定基础镜像 tag/digest、禁 SNAPSHOT 生产依赖）。
- **Q5：本地好好的，线上启动报 `NoSuchMethodError`，怎么查？** A：结论——先确认线上制品里的依赖版本与本地是否一致；原因是线上包可能是旧构建产物、或 fat jar 中依赖被 shade 合并、或容器镜像里叠加了另一个 jar；边界是排查链路为"看部署记录与镜像 tag/digest → 解压 jar 看 `BOOT-INF/lib` 版本 → 对比 `dependency:tree` → 检查是否有 provided/容器提供版本的冲突（如 servlet-api）"。
- **Q6：分支策略怎么定？** A：结论——按发布节奏选：版本化交付用 Git Flow，持续交付用 GitHub Flow，成熟度高用 Trunk Based；原因是不存在普适最优，只有与"发布频率、测试自动化程度、回滚能力"匹配的方案；边界是无论选哪种，主干必须受保护、分支生命周期尽量短（< 3 天）、并统一 merge/rebase 约定，否则历史会失控。
- **Q7：怎么把构建从 20 分钟降到 5 分钟？** A：结论——先量化各阶段耗时，再按"依赖缓存 → 测试分层 → 并行与裁剪 → 增量"优化；原因是多数慢构建的瓶颈是依赖下载与串行测试，而不是编译；边界是 CI 缓存 `~/.m2` 与 Docker 层、单测与集成测试分阶段（failsafe 放 verify）、`-T 1C` 并行、按变更模块裁剪构建范围，最后才考虑换构建工具；不要用"跳过测试"伪装成优化。
- **Q8：如何管理多环境的配置与密钥？** A：结论——代码里只放默认值，环境差异走环境变量/配置中心，敏感信息进 Secret 管理且不进 Git；原因是配置与代码的变更节奏不同（配置需要秒级可回滚）；边界是用 K8s ConfigMap/Secret 或 Vault/Apollo，CI 中通过凭据注入（OIDC/短期令牌优先），并禁止在日志与异常里打印配置内容；预发与生产差异只允许"配置值不同"，不允许"制品不同"。
### 10.3 高频追问与陷阱
- 回答开放题时**只给工具名不给判断标准**会被追问到崩：一定要带上"为什么这么做、代价是什么、什么情况下不这么做"。
- 涉及灰度/版本/质量的回答尽量给出**量化证据**（如"发布失败率从 8% 降到 1%""MTTR 从 40 分钟降到 12 分钟"），比形容词更有说服力。
### 10.4 示例
```text
开放题作答模板（以"怎么做灰度"为例）
结论：按流量比例 + 用户维度分层放量，异常自动回滚。
方案：入口按 uid 哈希打灰度标 → 标全链路透传 → K8s 双 Deployment（稳定 19 副本 + 金丝雀 1 副本）
      或网关按权重分流 → 监控按 version 维度看 P99/错误率 → 5 分钟内指标正常则放量，异常自动回滚。
代价：需要双份资源、需要全链路埋点与版本维度指标，业务上有一段新旧共存期（接口必须兼容）。
边界：无监控按版本拆分的团队不能做灰度（无法判定），此时先用 Feature Flag 控制生效范围更稳妥。
```
### 10.5 一句话速记
工程类开放题统一答法：**结论先行 → 方案与工具 → 代价与边界**；高频题就藏在"依赖冲突、灰度、版本号、代码质量、多模块管理"这五个里。
## 自测题（含答案要点）
1. **Maven 坐标与仓库路径的关系？** —— GAV 唯一确定构件，仓库路径为 `group/artifact/version/artifact-version.jar`。
2. **超级 POM 是什么？** —— Maven 内置的隐式父 POM，定义 central 地址、标准目录、默认插件绑定；用 `help:effective-pom` 查看。
3. **三大生命周期与 default 关键阶段？** —— clean/default/site 相互独立；default 为 validate→compile→test→package→verify→install→deploy。
4. **`package` 与 `install` 的区别？** —— install 会装进本地仓库供其他模块（或项目）解析，package 只产出到 `target/`。
5. **`-DskipTests` 与 `-Dmaven.test.skip` 的区别？** —— 前者编译测试代码但不执行，后者连测试代码都不编译。
6. **六种依赖 scope 的差异？** —— compile 全通、provided 不打包不传递、runtime 不编译但打包、test 仅测试、system 需本地路径不推荐、import 仅用于导入 BOM。
7. **依赖调解的两条规则？** —— 最短路径优先；路径相同时先声明者优先（不存在"高版本优先"）。
8. **`dependencyManagement` 与 `dependencies` 的区别？** —— 前者只声明版本（子模块引用时省略版本，且能覆盖传递依赖版本），后者直接引入依赖。
9. **怎么排查依赖冲突？** —— `mvn dependency:tree -Dverbose -Dincludes=g:a` 定位，`dependencyManagement`/`exclusions` 修复，enforcer 的 `requireUpperBoundDeps` 纳入 CI。
10. **聚合与继承的区别？** —— 聚合（`<modules>`）管构建范围与反应堆顺序，继承（`<parent>`）管配置复用，二者正交。
11. **多模块版本怎么统一？** —— 父 POM 的 `dependencyManagement`/`pluginManagement` 为唯一版本源；`${revision}` + flatten-maven-plugin 做 CI 友好版本。
12. **`shade`/`assembly`/`spring-boot-maven-plugin` 怎么选？** —— 可执行 Boot 应用用 Boot 插件；独立工具 fat jar（可 relocation）用 shade；带目录与脚本的分发包用 assembly。
13. **`mvn deploy` 报 401 的常见原因？** —— `distributionManagement` 的 id 与 `settings.xml` 的 server id 不一致，或账号无写权限/推错仓库地址。
14. **Maven 与 Gradle 的取舍？** —— Maven 约定强、稳定易协作；Gradle 灵活且增量/缓存更快，适合大型多模块与深度定制。
15. **Git 三大区域与文件状态？** —— 工作区、暂存区、版本库（+远程）；状态为 untracked/modified/staged/committed。
16. **`reset --soft/--mixed/--hard` 的区别？** —— 依次为"只移 HEAD""移 HEAD + 重置暂存区""还清工作区"；只能用于未推送历史。
17. **`revert` 与 `reset` 怎么选？** —— 公共分支用 revert（生成反向提交、不改历史），本地未推送用 reset。
18. **rebase 的黄金法则？** —— 不要 rebase 已推送到公共分支的提交；个人分支可用 `--force-with-lease`。
19. **merge 与 rebase 的效果差异？** —— merge 保留合并点和真实分支关系但图复杂；rebase 得到线性历史但重写提交 hash。
20. **误 `reset --hard` 怎么恢复？** —— `git reflog` 找目标 sha 后 `git reset --hard <sha>`；从未 commit 的改动无法恢复。
## 背诵卡片
1. Maven 坐标 = groupId + artifactId + version，唯一确定一个构件。
2. 超级 POM 隐式被所有 POM 继承，`help:effective-pom` 看最终生效配置。
3. 仓库查找顺序：本地仓库 → 私服/镜像（mirrorOf 优先于 POM 仓库）→ central。
4. `settings.xml` 的 server id 必须与 `distributionManagement` 的 id 一致。
5. 三生命周期相互独立，同一生命周期内阶段顺序执行。
6. default 关键阶段链：validate → compile → test → package → verify → install → deploy。
7. `-DskipTests` 跳过执行、`-Dmaven.test.skip` 连编译都跳过。
8. scope：compile 全通、provided 不打包不传递、runtime 不编译但打包、test 仅测试。
9. `import` scope 只能用在 `dependencyManagement` 里导入 BOM。
10. 依赖调解：最短路径优先，其次先声明者优先（没有高版本优先）。
11. `dependencyManagement` 只声明版本，可覆盖传递依赖版本；`dependencies` 才引入。
12. `optional=true` 不向外传递；`exclusions` 排除传递依赖。
13. 排查冲突：`dependency:tree -Dverbose -Dincludes=g:a` + `dependency:analyze` + enforcer。
14. `requireUpperBoundDeps`/`dependencyConvergence` 是依赖冲突的 CI 门禁。
15. 聚合管构建顺序，继承管配置复用，二者正交可单独使用。
16. 父 POM 里写 `dependencies` 会让所有子模块无条件继承该依赖（应写 `dependencyManagement`）。
17. `${revision}` + flatten-maven-plugin = CI 友好的统一版本方案。
18. 循环依赖必须拆模块或抽接口，不能靠改 scope 绕过。
19. 插件靠阶段绑定工作；`packaging` 提供默认绑定，`<executions>` 自定义。
20. Boot 应用要能被依赖时需配 `classifier` 生成 plain jar。
21. 发布三步：`distributionManagement` + `settings.xml` + `mvn deploy`；SNAPSHOT 不上生产。
22. 构建加速顺序：依赖缓存 → 测试分层 → 并行（-T 1C）→ 裁剪（-pl -am）→ 增量/换工具。
23. Maven 强约定稳定，Gradle 灵活且增量与缓存更好；选型看团队。
24. Git 三区：工作区 →（add）暂存区 →（commit）版本库 →（push）远程。
25. `git add -p` 分块暂存是提交质量的起点。
26. 未推送历史用 reset，已推送历史用 revert；公共分支严禁 rebase。
27. `reset --soft` 保留暂存区、`--mixed` 保留工作区、`--hard` 全清。
28. `git reflog` 是本地自救万能钥匙；未 commit 的改动无法恢复。
29. `--force-with-lease` 比 `--force` 安全，但依然会覆盖别人提交。
30. 密钥泄露处理顺序：先轮换密钥，再改写历史并通知全员。
31. 分支模型看发布节奏：Git Flow（版本化）、GitHub Flow（持续交付）、Trunk Based（高成熟度）。
32. 分支生命周期越短冲突越少；主干必须开启分支保护。
