# Docker 与 Kubernetes

> **定位**：容器原理 → Docker 操作 → K8s 核心对象、探针、资源调度 → 排障与 Java 适配，覆盖"Java 应用上云原生"的完整链路。
> **重要度**：★★★★★（Docker/K8s 已是 3-5 年社招 JD 硬性要求；对象配置与排障题是大厂高频场景题）
> **建议复习时长**：6 小时（Docker 2h + K8s 对象与探针 2.5h + 调度排障与 Java 适配 1.5h）
> **版本基准**：Docker 24~26 / containerd 1.7 / Kubernetes 1.28~1.29；已移除 API（`extensions/v1beta1` 于 1.16、`autoscaling/v2beta*` 于 1.26）不再使用。
> **关联文档**：[复习路线总纲](../00-复习路线总纲.md)、[CICD与监控告警](./03-CICD与监控告警.md)、[Maven-Git与工程实践](./04-Maven-Git与工程实践.md)、[Linux与常用命令](../06-计算机基础/04-Linux与常用命令.md)、[JVM与GC调优](../01-Java基础/04-JVM与GC调优.md)、[SpringBoot与自动装配](../03-框架/02-SpringBoot与自动装配.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| 容器 vs 虚拟机、Namespace 与 Cgroups、镜像分层 | ★★★★☆ | 八股原理 |
| Dockerfile 指令与最佳实践 | ★★★★★ | 配置实践 / 最佳实践 |
| Docker 命令、网络模式、数据卷、Compose、Harbor | ★★★★★ | 命令速查 / 配置实践 |
| K8s 架构与核心工作负载 | ★★★★★ | 八股原理 / 配置实践 |
| Service / Ingress / kube-proxy 转发 | ★★★★★ | 八股原理 / 场景题 |
| ConfigMap / Secret / PV / PVC / StorageClass | ★★★★☆ | 配置实践 |
| 探针三件套与经典故障 | ★★★★★ | 故障排查 / 配置实践 |
| 资源 requests/limits、QoS、调度与 HPA | ★★★★★ | 故障排查 / 场景题 |
| kubectl 排障、Helm/Kustomize | ★★★★★ | 命令速查 / 故障排查 |
| Java 应用上 K8s 适配与 CI/CD 集成 | ★★★★★ | 最佳实践 / 场景题 |
## 1. 容器原理：Namespace、Cgroups 与镜像分层
### 1.1 核心原理
容器不是"轻量虚拟机"，而是**被内核隔离（Namespace）并限额（Cgroups）的一组普通进程**，文件系统靠联合挂载分层复用。

| 维度 | 容器 | 虚拟机 |
| --- | --- | --- |
| 隔离层级 | 进程级，共享宿主内核 | 硬件虚拟化，独立 Guest 内核 |
| 启动速度 / 开销 | 毫秒~秒级 / 仅进程开销 | 数十秒 / 每台独占 GB 级内存 |
| 隔离强度 | 较弱（内核漏洞可逃逸） | 强（Hypervisor + 硬件辅助） |
| 密度 / 交付物 | 数百个 / 镜像（应用+依赖+运行时） | 数十个 / 含 OS 的整机镜像 |

- **Namespace 六种**：mnt（挂载点，独立根文件系统）、UTS（主机名）、IPC（信号量/共享内存）、PID（进程号，容器内 PID 1 是应用）、net（网卡/端口/路由/iptables）、user（用户组，支持 rootless）；另有 cgroup、time 两种较新的。
- **Cgroups**：限制与统计 CPU（`cpu.max`/`cpu.weight`）、内存（`memory.max`，超限触发内核 OOM Killer）、IO（`io.max`）、PID 数（`pids.max`）；cgroup v1 与 v2 并存，K8s 1.25+ 推荐 v2。
- **镜像分层**：镜像 = 一组只读层叠加 + 元数据（config JSON + manifest），每层是一个文件系统 diff；容器启动时在最上层加**可写层**，读写走 Copy-on-Write，默认驱动 OverlayFS（`overlay2`）。基础层可被多镜像共享，但**容器删除后可写层数据全部丢失**。
- **镜像/容器/仓库**：镜像是只读模板（`name:tag` 定位），容器是运行实例（可多个），仓库（Harbor/Docker Hub）负责存储分发，pull/push 只传缺失层。
### 1.2 面试问答
- **Q1：容器为什么比虚拟机快？** A：结论——容器复用宿主内核、启动的是进程；原因是虚拟机要经 Hypervisor 引导完整 Guest 内核与 init（数十秒），容器只需创建 Namespace 并 exec 应用（毫秒级）；边界是"快"以隔离弱为代价，强隔离场景应选虚拟机或 Kata/gVisor。
- **Q2：容器内 PID 1 为什么特殊？** A：结论——它负责回收孤儿进程并接收 `docker stop` 的 SIGTERM；原因是内核只对 PID 1 忽略未注册信号的默认行为，应用不处理 SIGTERM 就会等超时被 SIGKILL；边界是 Java 必须用 exec 形式 `ENTRYPOINT ["java","-jar",...]` 并注册 shutdown hook，shell 形式会让 PID 1 变成 `/bin/sh` 而收不到信号。
- **Q3：内存超限为什么不是 JVM 的 OOM 而是 OOMKilled？** A：结论——超的是 cgroup 硬限额，由内核 OOM Killer 直接杀进程（exit code 137）；原因是 `memory.max` 与堆内 OOM 是两套机制，JVM 来不及抛异常；边界是"limit 等于堆上限"必然被杀，必须给 Metaspace、线程栈、Direct Buffer、JVM 自身留 25%~40%。
### 1.3 高频追问与陷阱
- "容器里看到宿主核数"是旧版本问题：JDK 10+/8u191+ 默认 `UseContainerSupport` 已能感知 cgroup，但 cgroup v2 需较新 JDK 与运行时；可写层写满会表现为"磁盘写满"故障，排查入口是 `docker system df`。
### 1.4 示例
```bash
docker inspect -f '{{.State.Pid}}' <container>   # 宿主上查看容器主进程 PID
lsns && cat /sys/fs/cgroup/memory.max            # 查看 namespace 与 cgroup 内存上限
```
### 1.5 一句话速记
容器 = **Namespace 隔离 + Cgroups 限额 + 联合文件系统分层**；镜像是只读层集合，容器多一层可写层，删除即丢数据。
## 2. Dockerfile 指令与最佳实践
### 2.1 核心原理
| 指令 | 作用与关键点 |
| --- | --- |
| `FROM` / `WORKDIR` | 基础镜像（`AS builder` 起多阶段别名）/ 工作目录，不存在则创建 |
| `RUN` | 构建期执行，每个 `RUN` 一层；合并命令并在同层清理缓存 |
| `COPY` / `ADD` | 复制文件；`ADD` 还会自动解压 tar、下载 URL，**推荐一律用 `COPY`** |
| `ENV` / `ARG` | `ENV` 进镜像运行环境（`-e` 可覆盖）；`ARG` 仅构建期可见，**不能用来传密钥** |
| `USER` / `EXPOSE` / `VOLUME` | 生产必须非 root（如 `USER 1000`）/ 端口声明（不真正发布）/ 匿名卷 |
| `CMD` / `ENTRYPOINT` | `ENTRYPOINT` 固定入口，`CMD` 是可被覆盖的默认参数；都用 **exec 数组形式** |

**最佳实践七条**：① 多阶段构建（构建期 JDK、运行期只带 JRE）；② 指令按"变更频率从低到高"排列以最大化层缓存（先 `COPY pom.xml` + 下载依赖，再 `COPY src`）；③ `.dockerignore` 排除 `target/`、`.git/`、`.env`；④ 合并 `RUN` 并清理 apt/apk 缓存；⑤ 非 root 运行；⑥ 基础镜像固定具体 tag 或 digest，禁用 `latest`；⑦ 用 `jlink`/`jdeps` 裁剪 JRE 或选 `distroless`/`-slim`（`alpine` 用 musl libc，有 glibc 原生依赖时须换 Debian 系）。
### 2.2 面试问答
- **Q1：多阶段构建解决什么问题？** A：结论——把构建工具链与运行时分离，镜像可降一个数量级；原因是 Maven 仓库、JDK、源码都留在 builder 阶段，只有产物被 `COPY --from=builder` 拷入；边界是频繁构建应配 BuildKit 的 `--mount=type=cache,target=/root/.m2`，否则依赖会反复下载。
- **Q2：`CMD` 和 `ENTRYPOINT` 怎么选？** A：结论——固定入口用 ENTRYPOINT，可变默认参数用 CMD；原因是 `docker run image arg` 会追加到 ENTRYPOINT 之后并覆盖 CMD，这是"一个镜像多种启动参数"的标准做法；边界是 shell 形式会让 PID 1 变成 shell、收不到 SIGTERM，Java 应用必须用 exec 形式并显式携带 JVM 参数。
- **Q3：层顺序为什么影响构建速度？** A：结论——缓存要求"该层及之前所有指令完全一致"才命中；原因是 Docker 逐层比对校验和，任何一层变化都会让其后的层全部失效；边界是"每次构建都 `COPY . .`"会让依赖下载层重跑，因此依赖描述文件必须单独先 COPY。
### 2.3 高频追问与陷阱
- `COPY . .` 会把 `target/`、`.git/`、`.env` 带进构建上下文，既慢又可能泄露敏感信息；容器默认 UTC，Java 应用须设 `TZ=Asia/Shanghai` 与 `-Duser.timezone=Asia/Shanghai`，否则时间与日志差 8 小时。
### 2.4 示例（多阶段 + 缓存友好 + 非 root）
```dockerfile
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /build
COPY pom.xml .                                        # 依赖先拷贝，层缓存稳定命中
RUN --mount=type=cache,target=/root/.m2 mvn -B -q dependency:go-offline
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 mvn -B -q clean package -DskipTests
FROM eclipse-temurin:17.0.9_9-jre-jammy                # 运行阶段不含源码与构建工具
RUN groupadd -g 1000 app && useradd -u 1000 -g app -m app
COPY --from=builder --chown=1000:1000 /build/target/*.jar app.jar
ENV TZ=Asia/Shanghai LANG=C.UTF-8
USER 1000                                              # 非 root 运行
ENTRYPOINT ["sh","-c","exec java -XX:MaxRAMPercentage=70 -Duser.timezone=Asia/Shanghai -jar app.jar"]
```
### 2.5 一句话速记
Dockerfile 四要点：**多阶段减小体积、依赖先拷贝保缓存、`COPY` 不用 `ADD`、exec 形式 + 非 root + 固定 tag**。
## 3. Docker 命令、网络、数据卷、Compose 与 Harbor
### 3.1 核心原理
| 分类 | 常用命令 |
| --- | --- |
| 镜像 | `images`、`pull/push`、`build -t`、`tag`、`rmi`、`history <img>`、`system df` |
| 容器 | `ps -a`、`run -d --name -p -e -v --restart`、`start/stop/restart`、`rm -f`、`inspect`、`stats` |
| 日志 / 调试 | `logs -f --tail=200 <c>`、`exec -it <c> sh`、`diff <c>`、`events` |
| 网络 / 卷 / 清理 | `network ls/create/inspect/prune`、`volume ls/create/inspect/rm`、`system prune -a --volumes`（慎用） |

| 网络模式 | 含义与注意 |
| --- | --- |
| `bridge`（默认） | 接入 `docker0`，出网 SNAT、入网靠 `-p`；**不支持按容器名解析**，只能用 IP |
| 自定义 bridge | 自建网络并内嵌 Docker DNS，**容器可用名字互访**，推荐 |
| `host` / `none` / `container:<c>` | 共享宿主网络栈（性能最好但端口冲突）/ 仅 lo / 共享另一容器网络命名空间 |

- **端口映射**：`-p 8080:8080` 是在宿主 iptables 加 DNAT 规则；`-p 8080` 表示随机宿主端口。**容器内 localhost 指容器自身**，访问宿主服务在 Linux 上要用宿主网桥 IP（或 `--add-host=host.docker.internal:host-gateway`）。
- **数据卷三种**：`volume`（Docker 管理于 `/var/lib/docker/volumes`，持久化首选）、`bind mount`（挂宿主目录，适合开发期挂源码配置）、`tmpfs`（内存盘）；非 root 容器挂卷常见 `Permission denied`，需 UID 匹配。
- **Compose（v2，`docker compose`，`version` 字段已废弃）**：`depends_on` 只保证启动顺序、**不保证依赖就绪**，必须配 `healthcheck` + `condition: service_healthy`。
- **Harbor**：企业级 Registry，提供项目隔离、RBAC、镜像复制、Trivy 漏洞扫描、tag 保留与 GC；tag 规范 `<环境>-<应用>-<语义版本>-<git短SHA>`。
### 3.2 面试问答
- **Q1：为什么推荐自定义 bridge？** A：结论——默认 bridge 不支持容器名 DNS 解析，只能写死 IP；原因是 Docker 内嵌 DNS 只在用户自定义网络上启用；边界是生产编排应交给 K8s（Service + CoreDNS），Docker 网络主要用于本地与单机部署。
- **Q2：数据卷与 bind mount 怎么选？** A：结论——持久化数据用 volume，开发期挂代码/配置用 bind mount；原因是 volume 生命周期独立于容器、跨平台一致，bind mount 依赖宿主目录结构与权限；边界是 K8s 里对应 PV/PVC（持久）、`hostPath`（节点目录）、`emptyDir`（随 Pod 删除清空）。
- **Q3：`docker system prune -a --volumes` 有什么风险？** A：结论——会删除所有未被使用的镜像与**数据卷**，可能直接抹掉数据；原因是 prune 按"是否被运行中容器引用"判断；边界是执行前先 `docker system df` 与 `volume ls` 确认；另需注意 `docker logs` 依赖 json-file 驱动，默认无轮转，须配 `--log-opt max-size=100m --log-opt max-file=3`。
### 3.3 高频追问与陷阱
- `docker rm` 不会删卷，`docker rm -v` 才会；但 Compose `down -v` 会连卷一起删。
### 3.4 示例
```yaml
# compose.yaml：本地依赖一键起（v2 语法，无需 version 字段）
services:
  mysql:
    image: mysql:8.0.36
    ports: ["3306:3306"]
    volumes: ["mysql-data:/var/lib/mysql"]
    environment: { MYSQL_ROOT_PASSWORD: root, TZ: Asia/Shanghai }
    healthcheck: { test: ["CMD","mysqladmin","ping","-h","127.0.0.1"], interval: 5s, retries: 20 }
  app:
    build: .
    depends_on: { mysql: { condition: service_healthy } }   # 等真正就绪，而非只等启动
volumes: { mysql-data: }
```
### 3.5 一句话速记
命令三条主线：**看状态（ps/logs/inspect/stats）、进容器（exec）、清资源（prune/df）**；网络用自定义 bridge，数据用 volume，多容器本地编排用 Compose。
## 4. Kubernetes 架构与核心工作负载
### 4.1 核心原理
控制面负责"决策"、节点负责"执行"，所有组件只与 apiserver 通信（声明式 + 控制器调谐）。

| 组件 | 职责 |
| --- | --- |
| kube-apiserver | 唯一入口：认证 → 鉴权（RBAC）→ 准入 → 校验 → 持久化；扩展读能力靠水平扩展它 |
| etcd | 唯一数据源，Raft 多副本（需奇数节点）；watch 是声明式协调的基础；备份它即备份集群 |
| kube-scheduler | 为新 Pod 选节点：过滤 → 打分 → 绑定；只决策不下发，由 kubelet 执行 |
| kube-controller-manager | 各类控制器（Deployment/ReplicaSet/Node/Endpoint/Job）调谐实际状态向期望状态收敛 |
| kubelet / kube-proxy | 节点代理（管 Pod、拉镜像、执行探针、上报状态、资源驱逐）/ 把 Service 转发规则写进内核（iptables/IPVS） |
| CRI / CNI | containerd、CRI-O（Docker 自 1.24 起不再是默认）/ Pod 网络与 IPAM（Calico、Cilium、Flannel） |

| 工作负载 | 用途与关键点 |
| --- | --- |
| Pod | 最小调度单元，容器共享网络/IPC/卷（同 Pod 用 localhost 互通）；生命周期 Pending→Running→Succeeded/Failed，含 Init 容器与 `postStart`/`preStop`；重启策略 `Always`（Deployment）/`OnFailure`/`Never`（Job） |
| ReplicaSet / Deployment | RS 保证副本数（一般不直接创建）；Deployment 管无状态发布与回滚，`maxSurge`/`maxUnavailable`（默认 25%）、`revisionHistoryLimit`（默认 10）、`rollout status/undo/history` |
| StatefulSet | 稳定网络标识（headless Service：`pod-0.svc.ns`）、稳定存储（`volumeClaimTemplates`）、有序创建删除；用于 MySQL/Kafka/ZK/ES |
| DaemonSet / Job / CronJob | 每节点一个（日志采集、node-exporter、CNI）/ 一次性任务（`completions`、`backoffLimit`）/ 定时任务（`schedule` 注意时区、`concurrencyPolicy`） |
| HPA | 按指标自动扩缩副本，需 metrics-server；`autoscaling/v2` 支持 CPU/内存/自定义/外部指标与 `behavior` |
### 4.2 面试问答
- **Q1：Deployment 滚动更新怎么工作？** A：结论——控制器新建 ReplicaSet 并按 `maxSurge`/`maxUnavailable` 逐步扩新缩旧；原因是两个 RS 副本数此消彼长，且只有 readiness 通过的 Pod 计入可用；边界是 `Recreate` 策略会先全删后建（有停机），回滚依赖历史 RS 且受 `revisionHistoryLimit` 限制。
- **Q2：为什么 Pod 内多个容器"像"一台机器？** A：结论——同 Pod 容器共享 Network、IPC、UTS 命名空间与卷；原因是 kubelet 先建 sandbox（pause）容器，业务容器加入其命名空间；边界是文件系统与 PID 默认不共享（可显式 `shareProcessNamespace: true`）。
- **Q3：StatefulSet 与 Deployment 的核心差别？** A：结论——StatefulSet 提供稳定标识与稳定存储，Deployment 不保证；原因是它的 Pod 名有序号、重建后名字与 PVC 绑定关系不变、默认按序启停；边界是启动慢、扩缩容有顺序依赖，无状态服务不要用，且删 StatefulSet 默认保留 PVC。
### 4.3 高频追问与陷阱
- `get pods` 显示 `Running` 不等于可用，Ready 列 `1/1` 才代表 readiness 通过并加入 Endpoints；CronJob 的 `schedule` 用控制面时区（默认 UTC），需 `timeZone: Asia/Shanghai`。
### 4.4 示例
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: order-service, labels: { app: order-service } }
spec:
  replicas: 3
  selector: { matchLabels: { app: order-service } }     # 不可变，须与 template 匹配
  strategy: { type: RollingUpdate, rollingUpdate: { maxSurge: 1, maxUnavailable: 0 } }
  template:
    metadata: { labels: { app: order-service } }
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: app
          image: registry.example.com/order/order-service:v1.2.3-a1b2c3d
          ports: [{ containerPort: 8080 }]
```
### 4.5 一句话速记
控制面四件套（**apiserver 唯一入口、etcd 唯一数据源、scheduler 只决策、controller-manager 做调谐**）+ 节点三件套（kubelet 管 Pod、kube-proxy 转发、CRI/CNI 落地）；无状态用 Deployment、有状态用 StatefulSet、每节点一个用 DaemonSet。
## 5. Service、Ingress 与网络转发
### 5.1 核心原理
Pod IP 随重建变化，因此需要稳定的访问入口——Service。

| Service 类型 | 含义与适用 |
| --- | --- |
| ClusterIP（默认） | 集群内虚拟 IP，服务间调用 |
| NodePort / LoadBalancer | 每节点开放 30000-32767 端口 / 自动申请云厂商 LB，云上对外暴露 |
| ExternalName / Headless | 返回 CNAME 不做代理（引入集群外服务如 RDS）/ `clusterIP: None`，DNS 返回 Pod IP 列表，用于 StatefulSet |

- **kube-proxy 转发**：监听 Service/EndpointSlice 变化并写内核规则；`iptables`（默认，规则线性匹配，Service 多时性能下降）与 `ipvs`（哈希查找，大规模更好）。请求路径：`ClusterIP:port` → 内核 DNAT → 随机一个就绪 Pod。
- **Endpoints/EndpointSlice 只包含 readiness 通过的 Pod**，由 Endpoint 控制器维护——这是"故障 Pod 自动摘除"的实现基础。
- **Ingress**：七层 HTTP(S) 路由（按 host/path 转发到 Service），本身只是规则对象，**必须有 Ingress Controller（ingress-nginx/Traefik）才生效**；`IngressClass` 支持多控制器；路径类型 `Prefix`/`Exact`；TLS 证书放 Secret；演进方向是 Gateway API。
### 5.2 面试问答
- **Q1：Service 的 ClusterIP 能 ping 通吗？** A：结论——通常 ping 不通但 TCP 正常；原因是 ClusterIP 只是内核里的虚拟地址，没有实体网卡与 ICMP 响应逻辑；边界是排障要用 `curl`/`nc -zv`，别把"ping 不通"当故障。
- **Q2：Service 为什么能自动摘除故障 Pod？** A：结论——EndpointSlice 只收录 readiness 通过的 Pod；原因是 Endpoint 控制器以 Pod 的 `Ready` 条件做增删；边界是无 selector 的 Service 需手工维护 Endpoints，此时会失去自动摘除能力。
- **Q3：一个请求从外到内怎么走？** A：结论——外部 LB → Ingress Controller（或 NodePort/LoadBalancer）→ Service ClusterIP → kube-proxy 内核 DNAT → 某个就绪 Pod；原因是每层只解决"找下一跳"；边界是排查要逐段验证，并先确认 `kubectl get endpoints` 有后端；另外 `selector` 写错会让 Endpoints 为空、表现为"连接被拒绝"，而 `port` 与 `targetPort` 不一致会导致 502/超时。
### 5.3 高频追问与陷阱
- NetworkPolicy 需 CNI 支持（Calico/Cilium，Flannel 默认不支持）；未定义时全通，一旦定义入向策略即变为默认拒绝，容易一次性切断流量。
### 5.4 示例
```yaml
apiVersion: v1
kind: Service
metadata: { name: order-service }
spec:
  type: ClusterIP
  selector: { app: order-service }
  ports: [{ name: http, port: 80, targetPort: 8080 }]    # port=Service 端口，targetPort=容器端口
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: order-ingress }
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http: { paths: [{ path: /order, pathType: Prefix,
              backend: { service: { name: order-service, port: { number: 80 } } } }] }
```
### 5.5 一句话速记
Service 提供稳定虚拟 IP（ClusterIP/NodePort/LoadBalancer/ExternalName/Headless），kube-proxy 用 iptables 或 IPVS 做 DNAT；Ingress 是七层规则、必须有控制器；**查 Service 先查 Endpoints**。
## 6. 配置与存储：ConfigMap、Secret、PV/PVC
### 6.1 核心原理
- **ConfigMap / Secret**：把配置与镜像解耦。Secret 面向敏感数据（`data` 为 base64、`stringData` 可写明文）——**base64 不是加密**，高安全场景需 RBAC 最小权限 + etcd 静态加密（EncryptionConfiguration）+ 外部密钥管理（Vault/SealedSecrets/云 KMS CSI）。
- **挂载方式**：环境变量（`envFrom`/`valueFrom`，简单但**更新不生效**）与文件挂载（`volumeMounts`，ConfigMap 更新后 kubelet 会同步文件，但 Spring Boot 默认不重载，需重启或接配置中心）；**`subPath` 挂载的单文件不会热更新**，这是"改了配置没生效"的经典根因。
- **Volume 家族**：`emptyDir`（Pod 内共享临时目录，Pod 删除即清）、`hostPath`（挂节点目录，多用于 DaemonSet）、`configMap`/`secret`/`downwardAPI`、网络存储（NFS/云盘）。
- **PV / PVC / StorageClass**：PV 是集群级存储资源，PVC 是命名空间级申请，按 `storageClassName` + 容量 + 访问模式绑定。访问模式 `ReadWriteOnce`（RWO，最常用）/`ReadOnlyMany`/`ReadWriteMany`（需 NFS/CephFS）/`ReadWriteOncePod`；回收策略 `Delete`（默认）/`Retain`（保留数据）。**动态供给**：PVC 不指定 `volumeName` 时由 StorageClass 的 provisioner 自动建 PV；云上多 AZ 必须用 `volumeBindingMode: WaitForFirstConsumer`，否则 PV 所在可用区可能与 Pod 调度节点不一致。
### 6.2 面试问答
- **Q1：ConfigMap 更新后应用会自动生效吗？** A：结论——文件挂载的会（延迟约一个 kubelet 同步周期），环境变量方式不会；原因是环境变量在容器创建时固定；边界是 Spring Boot 也不会自动重载，热更要 `kubectl rollout restart` 或接配置中心，而 `subPath` 挂载单文件两者都不更新。
- **Q2：PVC 一直 Pending 怎么办？** A：结论——检查 StorageClass 是否存在且 provisioner 正常、访问模式与容量是否被支持、是否因 `WaitForFirstConsumer` 在等 Pod 调度；原因是动态供给需要 provisioner 真实调用底层存储 API；边界是云上还要确认 PV 所在 AZ 与调度节点一致，否则 Pod 卡在 ContainerCreating。
- **Q3：Secret 真的安全吗？** A：结论——不，默认只是 base64 编码；原因是任何能读该命名空间 Secret 的主体都能解码，etcd 未加密时磁盘上也是明文；边界是加固手段为 RBAC 最小权限 + etcd 静态加密 + 外部密钥管理器，并禁止把 Secret 内容打进日志（另注意非 root 容器挂 PVC 常见 `Permission denied`，需 `securityContext.fsGroup`）。
### 6.3 高频追问与陷阱
- `emptyDir` 默认用节点磁盘且不计入容器内存 limit，`medium: Memory` 时才计入；StatefulSet 缩容或删除时 PVC 默认保留，需人工清理。
### 6.4 示例
```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: order-config }
data:
  application-prod.yml: |
    server: { shutdown: graceful }                # 挂载为文件才会热更，env 方式不会
---
apiVersion: v1
kind: Secret
metadata: { name: order-secret }
type: Opaque
stringData: { DB_PASSWORD: "S3cret!" }           # 直接写明文，省去手工 base64
---
volumeClaimTemplates:                            # StatefulSet：每个副本一套独立 PVC
  - metadata: { name: data }
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: ssd-retain               # Retain 回收策略防误删数据
      resources: { requests: { storage: 100Gi } }
```
### 6.5 一句话速记
配置用 ConfigMap/Secret（**base64 非加密**），环境变量不热更、文件挂载会热更但应用未必重载、`subPath` 不热更；存储三件套 PV（资源）/PVC（申请）/StorageClass（动态供给），访问模式常用 RWO，`WaitForFirstConsumer` 防跨 AZ。
## 7. 探针：liveness / readiness / startup
### 7.1 核心原理
| 探针 | 失败后果 | 解决的问题 | 探测方式 |
| --- | --- | --- | --- |
| livenessProbe | **重启容器** | 进程活着但卡死（死锁、线程池耗尽、连接池打满） | httpGet / tcpSocket / exec / gRPC |
| readinessProbe | **从 Endpoints 摘除**（不重启） | 启动未完成、依赖不可用、临时下线做保护 | 同上 |
| startupProbe | 期间抑制 liveness/readiness，成功后接管 | 启动很慢的应用（Spring Boot 冷启动数十秒） | 同上 |

- 参数：`initialDelaySeconds`、`periodSeconds`（默认 10s）、`timeoutSeconds`（默认 1s）、`failureThreshold`（默认 3）、`successThreshold`（liveness 必须为 1）。
- **配置原则（结论先行）**：liveness **只探进程自身**（`/actuator/health/liveness`，不含数据库等外部依赖），readiness 才允许包含依赖（`/actuator/health/readiness`）；启动慢的应用用 `startupProbe` 给足窗口（`periodSeconds × failureThreshold` > 最长冷启动时间），而不是把 liveness 的 `initialDelaySeconds` 无限调大。Spring Boot 2.3+ 在 K8s 环境会自动提供这两个端点。
### 7.2 面试问答
- **Q1：把数据库检查放进 liveness 会怎样？** A：结论——依赖抖动会引发全量 Pod 重启的级联故障；原因是 liveness 失败即重启，重启期间无法服务，若 DB 仍不可用则新实例同样失败，进入 CrashLoopBackOff，故障从"依赖不可用"升级为"服务全挂"；边界是依赖检查应放 readiness（摘流量、保进程，依赖恢复后自动回归）。
- **Q2：有了 liveness 为什么还要 startupProbe？** A：结论——为"启动很慢"提供独立容忍窗口；原因是不配它就得把 liveness 的 `initialDelaySeconds` 调到最长冷启动时间，这会让运行期真卡死也被延迟同等时间才发现；边界是 startupProbe 的 `failureThreshold` 要给足余量（如 `periodSeconds: 5` + `failureThreshold: 60` 即 5 分钟）。
- **Q3：探针用 HTTP 还是 exec？** A：结论——优先 HTTP（开销小、可独立端点、能反映真实服务能力）；原因是 exec 会 fork 进程且依赖镜像内有工具，distroless 镜像没有 shell 会导致 exec 探针直接失效；边界是 TCP 探针只能证明端口在监听，无法说明应用能处理请求。
### 7.3 高频追问与陷阱
- **经典故障**：`initialDelaySeconds: 5` + 冷启动 40s → 容器反复被杀 → `CrashLoopBackOff`，日志里能看到多次启动横幅；解法是加 `startupProbe`。另外 `timeoutSeconds: 1` 对首次 SSL/DB 握手常太短，应放大到 2~3s。
### 7.4 示例
```yaml
containers:
  - name: app
    startupProbe:                                   # 容忍最长 5s × 60 = 300s 冷启动
      httpGet: { path: /actuator/health/liveness, port: 8080 }
      periodSeconds: 5
      failureThreshold: 60
    livenessProbe:                                  # 只探进程自身，不含 DB
      httpGet: { path: /actuator/health/liveness, port: 8080 }
      periodSeconds: 10
      timeoutSeconds: 3
      failureThreshold: 3
    readinessProbe:                                 # 含依赖，失败只摘流量
      httpGet: { path: /actuator/health/readiness, port: 8080 }
      periodSeconds: 10
      failureThreshold: 2
```
### 7.5 一句话速记
**liveness 只探自己、失败重启；readiness 可探依赖、失败摘流量；startup 保护慢启动、成功即隐退**——把 DB 检查放进 liveness 是最经典的全站故障配方。
## 8. 资源、QoS、调度与 HPA
### 8.1 核心原理
- **requests 与 limits**：`requests` 是调度依据与 CPU 权重（cgroup `cpu.weight`）、内存下限保障；`limits` 是运行上限（CPU 超限被限流 throttling，内存超限触发 **OOMKilled**）。只设 limits 会默认 requests=limits，可能因"装不下"而 Pending；内存 request 远小于 limit 会导致节点超卖、OOM 概率大增。
- **QoS 等级**（决定节点内存压力下谁先被驱逐）：`Guaranteed`（所有容器 requests == limits，最稳、最后被驱逐，核心服务建议）> `Burstable`（至少一个容器设了 requests，最常见）> `BestEffort`（都没设，最先被驱逐，只适合测试）。
- **调度手段**：`nodeSelector`（标签匹配）→ `nodeAffinity`（`required...` 硬约束 / `preferred...` 软约束）→ `podAffinity`/`podAntiAffinity`（按已运行 Pod 聚集或分散）→ `topologySpreadConstraints`（按拓扑域均匀分布，比反亲和更可控）→ `taints`/`tolerations`（`NoSchedule`/`PreferNoSchedule`/`NoExecute`）→ `priorityClassName` 抢占；维护用 `cordon`（禁调度）与 `drain`（驱逐），需配 PDB。
- **HPA**：`autoscaling/v2` 支持 CPU/内存/自定义（Prometheus Adapter）/外部指标，需 metrics-server；近似公式 `期望副本 = ceil(当前副本 × 当前指标 / 目标值)`，指标是"**用量 ÷ requests**"，因此**没设 requests 时 HPA 不工作**；`behavior.scaleDown.stabilizationWindowSeconds` 抑制抖动。
### 8.2 面试问答
- **Q1：Pod 被 OOMKilled 怎么排查？** A：结论——先 `describe pod` 确认 `Reason: OOMKilled, Exit Code: 137`，再区分堆内还是堆外；原因是容器 limit 统计整个进程 RSS（堆 + Metaspace + 线程栈 + Direct Buffer + JVM 自身 + JNI），只看 `-Xmx` 会误判；边界是修复顺序为"加 limit 止血 → 按 `MaxRAMPercentage≈70` 重设堆 → 开 NMT 与 HeapDump 定位真实占用"。
- **Q2：requests 和 limits 怎么定？** A：结论——requests 取压测 P95 稳定占用，limits 留 1.3~1.5 倍余量，CPU 建议不低于 2 核或只设 request；原因是 CPU 是弹性资源、limit 会造成限流与延迟尖刺，内存是非弹性资源、超限直接被杀；边界是内存 request/limit 应成比例避免超卖，容量交给 HPA 而非一味放大 limit。
- **Q3：HPA 不扩容的原因？** A：结论——没装 metrics-server、目标负载没设 requests、指标口径与目标值不匹配；原因是 HPA 通过 `metrics.k8s.io` 读取指标并以 requests 为分母算使用率；边界是自定义指标需部署 Prometheus Adapter，`stabilizationWindow` 也会让扩缩容看起来"没反应"；另需注意 `describe node` 的 `Allocated resources` 是 requests 之和，requests 超卖是新 Pod Pending 的常见原因。
### 8.3 高频追问与陷阱
- Java 容器中 `Runtime.availableProcessors()` 读到宿主核数而 limit 只有 1，会让 GC 与线程池配置过大并触发严重限流，应设 `-XX:ActiveProcessorCount=N`；`NoExecute` 污点会驱逐 Pod（`NoSchedule` 只是不调度新 Pod）。
### 8.4 示例
```yaml
resources:
  requests: { cpu: "1", memory: "2Gi" }        # 调度依据 + CPU 权重
  limits:   { memory: "3Gi" }                  # 只限内存，CPU 不限以免被 throttling
topologySpreadConstraints:                     # 副本跨节点均匀分布
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector: { matchLabels: { app: order-service } }
# HPA 片段：基于 CPU 使用率（用量 ÷ requests），scaleDown 稳定窗口抑制抖动
  minReplicas: 3
  maxReplicas: 20
  metrics: [{ type: Resource, resource: { name: cpu, target: { type: Utilization, averageUtilization: 60 } } }]
```
### 8.5 一句话速记
**requests 决定调度、limits 决定生死**（内存超限 OOMKilled、CPU 超限限流）；QoS 优先级 Guaranteed > Burstable > BestEffort；调度靠 nodeAffinity / 反亲和 / 拓扑分布 / 污点容忍；HPA 基于"用量 ÷ requests"。
## 9. kubectl 排障流程与 Helm / Kustomize
### 9.1 核心原理
| 分类 | 常用命令 |
| --- | --- |
| 查看 | `get pods -o wide -w`、`describe pod <p>`（Events 与 Last State）、`get events --sort-by=.lastTimestamp` |
| 日志调试 | `logs -f <p> --tail=200`、`logs <p> --previous`、`exec -it <p> -- sh`、`debug -it <p> --image=busybox` |
| 网络 / 发布 | `port-forward svc/x 8080:8080`、`get endpoints`、`rollout status/undo/restart`、`set image deploy/x app=img:tag` |

| 现象 | 排查顺序 |
| --- | --- |
| `Pending` | ① 资源不足（Events 的 `Insufficient cpu/memory`）；② 亲和/污点不匹配；③ PVC 未绑定（含 `WaitForFirstConsumer` 等调度）或 StorageClass 不存在；④ ResourceQuota/LimitRange 限制 |
| `CrashLoopBackOff` | ① `logs --previous`（配置缺失、DB 连不上、端口占用）；② 探针配错（liveness 过早）；③ 内存超限（`OOMKilled/137`）；④ `command`/`args` 覆盖镜像入口；⑤ Secret 缺失导致 `CreateContainerConfigError` |
| `ImagePullBackOff` | ① 镜像名/tag 拼写错（私有仓库须带 registry 域名）；② 缺 `imagePullSecrets`（Events 显示 `Unauthorized`）；③ 节点到仓库的网络与 DNS（节点上 `crictl pull` 最快验证）；④ 自签证书未配 CA；⑤ 镜像已被保留策略清理 |

- **Helm 与 Kustomize**：Helm 是模板 + 包管理（`Chart` = 模板 + `values.yaml`，`Release` = 一次安装实例），命令 `helm install/upgrade --atomic --timeout/rollback/history/template`，`--atomic` 失败自动回滚是生产必配；Kustomize 是无模板的 overlay 叠加（`base` + 多环境 `overlay` + patch），已内置为 `kubectl -k`。选型：对外交付、需参数化与版本回滚用 Helm；只要多环境差异用 Kustomize。
### 9.2 面试问答
- **Q1：Pod 一直 Pending，第一反应是什么？** A：结论——先看 `kubectl describe pod` 的 Events，它几乎总能直接给出原因；原因是调度失败只体现在 Events 的 `FailedScheduling`，`get pods` 的 STATUS 不带原因；边界是若提示资源不足，用 `describe node` 区分"requests 已占满"与"实际使用高"，前者调 requests 或加节点，后者查内存泄漏。
- **Q2：`logs --previous` 为什么重要？** A：结论——它显示上一次崩溃容器的日志，当前容器可能还没输出；原因是 kubelet 以指数退避重启（10s→20s→40s…最多 5 分钟），attach 上去往往看不到有效日志；边界是若 `--previous` 也无日志，说明进程极早期退出（JVM 参数错误、exec 失败），要看 Events 的 Exit Code（137=OOMKilled、1=应用异常、126/127=命令不可执行）。
- **Q3：为什么不要直接 `kubectl edit` 线上对象？** A：结论——会造成配置漂移，下次 CI 全量 `apply` 时被覆盖甚至冲突；原因是命令式修改没进 Git，无法审计与回滚；边界是应急可先改止血，但必须立刻回填到 Git；另外 `kubectl exec` 进不去 distroless 镜像（无 shell），改用 `kubectl debug`，而 `delete pod` 后控制器会立即重建，想彻底停服应 `scale --replicas=0`。
### 9.3 高频追问与陷阱
- Pod 长期 `Terminating` 通常是 finalizer 阻塞或节点失联，查 `describe pod` 的 `deletionTimestamp` 与节点状态。
### 9.4 示例
```bash
# 标准排障四连（多数问题在前三步定位）
kubectl get pods -o wide                                      # 1. 状态、节点、重启次数
kubectl describe pod order-service-7d9f-abcde                 # 2. Events 与 Last State
kubectl logs order-service-7d9f-abcde --previous --tail=300    # 3. 崩溃前日志
kubectl get events -n prod --sort-by=.lastTimestamp | tail -30 # 4. 全局事件时间线
kubectl rollout undo deploy/order-service                     # 回滚
helm upgrade --install order-service ./charts/order -n prod -f values-prod.yaml --atomic
kubectl apply -k overlays/prod                                # Kustomize 多环境
```
### 9.5 一句话速记
排障固定套路：**`get` 看现象 → `describe` 看事件 → `logs --previous` 看死因 → `events` 看全局**；Pending 查资源/亲和/存储，CrashLoop 查日志/探针/OOM，ImagePull 查镜像名/密钥/网络。
## 10. Java 应用上 K8s 适配与 CI/CD 集成
### 10.1 核心原理
**JVM 与容器**：JDK 10+/8u191+ 默认 `-XX:+UseContainerSupport`，按 cgroup limit 计算默认堆；`-XX:MaxRAMPercentage`（默认 25%）是推荐设置方式，比硬编码 `-Xmx` 更适配环境。CPU 感知（JDK 10+）读 cgroup 配额，但**老版本 JDK 8 与 cgroup v2 组合可能读到宿主核数**，需显式 `-XX:ActiveProcessorCount`。

| 关注点 | 结论做法 | 原因/坑 |
| --- | --- | --- |
| 内存 / CPU | `MaxRAMPercentage=70`、`InitialRAMPercentage=50`，limit 比堆大 40%；`ActiveProcessorCount` 与 limit 对齐 | 堆外（Metaspace、线程栈约 1MB/线程、Direct Buffer、JVM 自身）要占空间；核数虚高会让 GC/线程池配置过大并被限流 |
| 基础镜像 | 首选 `eclipse-temurin:*jre-jammy`/`-slim`；最小最安全用 `distroless`；`alpine` 的 musl 可能与 glibc 原生库不兼容 | 体积 / 兼容性 / 可调试性三者权衡 |
| 时区 / 字符集 / 日志 | `TZ=Asia/Shanghai` + `-Duser.timezone=Asia/Shanghai` + `LANG=C.UTF-8`；日志只输出 stdout（JSON） | 容器默认 UTC 会差 8 小时；写容器内文件会随重建丢失且写满可写层 |
| 优雅停机 | `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase=25s` + `preStop: sleep 10` + `terminationGracePeriodSeconds: 30` | K8s 并行执行"摘 Endpoints + 发 SIGTERM"，不延迟退出会丢正在处理的请求 |
| 慢启动 / 健康 | `startupProbe` 保慢启动；引入 actuator 暴露 liveness/readiness 端点 | 支撑"重启"与"摘流量"两种不同处置 |

**CI/CD 与 K8s 集成两种形态**：① **Push 式**：CI 构建镜像 → 推 Harbor → `kubectl set image` 或 `helm upgrade` 更新（需给 CI 集群凭据，权限须收敛到命名空间级）；② **GitOps Pull 式（推荐）**：CI 只构建镜像并把新 tag 提交到配置仓库，集群内 ArgoCD/Flux 监听 Git 同步（凭据不出集群、天然可审计可回滚）；再配合 Argo Rollouts/Flagger 做金丝雀自动回滚。详见 [CICD与监控告警](./03-CICD与监控告警.md)。
### 10.2 面试问答
- **Q1：Java 应用被 OOMKilled 但 JVM 没有异常，为什么？** A：结论——被杀的是进程级 cgroup 超限，内核直接干掉它；原因是 `memory.max` 统计整个进程 RSS，而 `-Xmx` 只覆盖堆；边界是排查顺序为"确认 OOMKilled 与 limit → 加 limit 止血 → 用 `MaxRAMPercentage` 重设堆 → 开 NMT 与 HeapDump 定位"，同时检查线程数与 Netty Direct Buffer 是否泄漏。
- **Q2：为什么推荐 `MaxRAMPercentage` 而不是 `-Xmx`？** A：结论——同一镜像在不同 limit 环境下自适应；原因是百分比基于容器内存限制计算，配合 `InitialRAMPercentage` 还能控制启动期堆；边界是 JDK 8u191 前不支持（需 `UseCGroupMemoryLimitForHeap`），且设置后不要再叠加 `-Xmx` 以免互相覆盖。
- **Q3：Spring Boot 优雅停机要做什么？** A：结论——`server.shutdown=graceful`、`preStop sleep`、`terminationGracePeriodSeconds`、readiness 探针四者配套；原因是删除 Pod 时摘 Endpoints 与发 SIGTERM 并行，容器立即退出会让 LB 把请求打到已死进程；边界是 `terminationGracePeriodSeconds` 必须大于"preStop 时长 + 停机时长"，且 MQ 消费者、线程池、连接池都要在 shutdown hook 中有序关闭。
### 10.3 高频追问与陷阱
- 容器内 `curl localhost:8080` 不通时，检查 Spring Boot 是否绑定了具体网卡、是否只监听 IPv6，以及 Actuator 端点是否被鉴权拦截；`LocalDateTime` 无时区，容器 UTC 会让业务时间漂移，统一用带时区类型。
### 10.4 示例
```yaml
containers:
  - name: app
    env:
      - name: JAVA_OPTS
        value: >-
          -XX:+UseContainerSupport -XX:MaxRAMPercentage=70 -XX:InitialRAMPercentage=50
          -XX:ActiveProcessorCount=2 -XX:+HeapDumpOnOutOfMemoryError
          -Duser.timezone=Asia/Shanghai -Dspring.profiles.active=prod
    lifecycle:
      preStop: { exec: { command: ["sh", "-c", "sleep 10"] } }   # 等 LB 摘流再退出
```
```bash
# Push 式：CI 构建推送后更新（需收敛 CI 的集群权限）
docker build -t $REG/order:$TAG . && docker push $REG/order:$TAG
kubectl -n prod set image deploy/order app=$REG/order:$TAG
# GitOps 式：只改配置仓库的 tag，由 ArgoCD 同步（集群凭据不出集群）
yq -i ".image.tag = \"$TAG\"" overlays/prod/kustomization.yaml && git commit -am "release $TAG" && git push
```
### 10.5 一句话速记
Java 上 K8s 五件事：**容器感知（MaxRAMPercentage≈70%）+ 固定时区字符集 + 日志到 stdout + preStop 优雅停机 + startupProbe 保慢启动**；CI/CD 优先 GitOps。
## 自测题（含答案要点）
1. **容器与虚拟机的本质区别？** —— 容器共享宿主内核（Namespace 隔离 + Cgroups 限额），虚拟机有独立 Guest 内核；前者更轻更快、隔离更弱。
2. **Namespace 六种分别隔离什么？** —— mnt 文件系统、UTS 主机名、IPC 进程间通信、PID 进程号、net 网络栈、user 用户组。
3. **Cgroups 限制了哪些资源？** —— CPU（`cpu.max`）、内存（`memory.max`，超限 OOM Killer）、IO、PID 数。
4. **镜像分层的收益与代价？** —— 收益是层共享与增量传输；代价是每层写有 COW 开销，写密集数据必须挂卷。
5. **`CMD` 与 `ENTRYPOINT` 的区别？** —— ENTRYPOINT 固定入口、CMD 是可被 `docker run` 参数覆盖的默认参数；都应用 exec 形式以正确接收 SIGTERM。
6. **Dockerfile 为什么先 COPY pom.xml？** —— 让依赖层缓存稳定命中，源码变动不触发依赖重新下载。
7. **默认 bridge 与自定义 bridge 的差别？** —— 后者有内嵌 DNS，可用容器名互访；前者只能用 IP。
8. **K8s 各组件职责？** —— apiserver 唯一入口、etcd 唯一数据源、scheduler 决策、controller-manager 调谐、kubelet 管 Pod、kube-proxy 转发、CRI/CNI 落地。
9. **Pod 内多容器为什么能用 localhost 互通？** —— 共享 Network（及 IPC/UTS）命名空间与卷，由 sandbox（pause）容器提供。
10. **Deployment 的 maxSurge/maxUnavailable 怎么配最稳？** —— `maxSurge: 1, maxUnavailable: 0`：发布期不降容量，但需额外资源。
11. **StatefulSet 的三个"稳定"是什么？** —— 稳定网络标识（headless Service + 序号）、稳定存储（volumeClaimTemplates 绑定 PVC）、有序创建与删除。
12. **Service 四种类型？** —— ClusterIP、NodePort（30000-32767）、LoadBalancer、ExternalName；另有 Headless。
13. **kube-proxy 的 iptables 与 IPVS 差异？** —— iptables 线性匹配、Service 多时性能下降；IPVS 哈希查找，大规模场景更好。
14. **readiness 与 liveness 的本质差别？** —— readiness 失败只摘流量、可含依赖检查；liveness 失败重启容器、只应探进程自身。
15. **把数据库检查放进 liveness 会导致什么？** —— 依赖抖动引发全量重启的级联故障，最终进入 CrashLoopBackOff。
16. **OOMKilled 的 exit code 与排查步骤？** —— 137；`describe pod` 看 Last State，区分堆内与堆外（Metaspace/线程栈/Direct Buffer/JVM 自身）。
17. **QoS 三个等级及意义？** —— Guaranteed（requests==limits，最后被驱逐）、Burstable、BestEffort（最先被驱逐）。
18. **Pod Pending 的四大原因？** —— 资源不足、亲和/污点不匹配、PVC 未绑定、配额限制。
19. **ImagePullBackOff 怎么查？** —— 镜像名/tag、imagePullSecrets、节点到仓库的网络与证书、镜像是否被清理；节点上 `crictl pull` 最快。
20. **Helm 与 Kustomize 的区别？** —— Helm 是模板 + 包管理（values/Release/回滚），Kustomize 是无模板 overlay（base + patch，`kubectl -k` 内置）。
21. **`preStop sleep 10` 解决什么问题？** —— K8s 并行执行摘 Endpoints 与发 SIGTERM，延迟退出可避免请求打到正在关闭的实例。
22. **Java 容器的堆该怎么设？** —— `MaxRAMPercentage≈70`，limit 比堆大 40% 左右，并用 `ActiveProcessorCount` 对齐 CPU limit。
## 背诵卡片
1. 容器 = Namespace 隔离 + Cgroups 限额 + 联合文件系统分层，共享宿主内核。
2. Namespace 六种：mnt、UTS、IPC、PID、net、user（另有 cgroup、time）。
3. Cgroups：CPU 限流、内存超限触发 OOM Killer、IO 与 PID 限额；v2 已是主流。
4. 镜像 = 只读层集合 + 元数据；容器 = 镜像 + 可写层；删容器丢可写层数据。
5. OverlayFS（overlay2）是默认存储驱动，读写走 Copy-on-Write。
6. 镜像/容器/仓库：`repository:tag` 定位镜像，容器是运行实例，仓库负责分发。
7. 容器内 PID 1 收信号并回收孤儿进程；`docker stop` 先 SIGTERM 后 SIGKILL。
8. Dockerfile 四要点：多阶段、依赖先拷贝、COPY 不用 ADD、exec 形式 + 非 root。
9. `ADD` 会解压 tar 与下载 URL，行为不可预期，故推荐 `COPY`。
10. `.dockerignore` 必须排除 `target/`、`.git/`、`.env`；`ARG` 传密钥不安全。
11. 层缓存失效会连带其后所有层重跑，指令按"变更频率升序"排列。
12. 基础镜像固定具体 tag 或 digest，禁用 `latest`。
13. 网络四模式：bridge（默认）、自定义 bridge（带 DNS，推荐）、host、none，另有 `container:`。
14. `-p 宿主:容器` 是 iptables DNAT；容器内 localhost 不是宿主机。
15. 数据持久化用 volume（推荐）或 bind mount；tmpfs 不落盘。
16. Compose `depends_on` 只保证启动顺序，就绪要配 `condition: service_healthy`。
17. Harbor：项目隔离 + RBAC + 复制 + 漏洞扫描 + 保留策略；tag 带 git SHA。
18. `docker system prune -a --volumes` 会删未使用卷，先 `docker system df`。
19. K8s 控制面：apiserver 唯一入口、etcd 唯一数据源、scheduler 只决策、controller-manager 做调谐。
20. 节点组件：kubelet 管 Pod 与探针、kube-proxy 转发、CRI（containerd）、CNI（Calico/Cilium）。
21. 1.24 起移除 dockershim，运行时用 containerd，镜像仍遵循 OCI 标准。
22. Pod 是最小调度单元，共享网络与卷；重启策略 Always/OnFailure/Never。
23. Deployment 通过 RS 实现滚动更新，`rollout undo` 回滚，历史受 `revisionHistoryLimit` 限制。
24. StatefulSet 稳定网络/存储/顺序，配 headless Service 与 volumeClaimTemplates。
25. DaemonSet 每节点一个：日志采集、node-exporter、CNI、kube-proxy。
26. CronJob 注意时区与 `concurrencyPolicy`；Job 重启策略用 OnFailure/Never。
27. Service 四类型 + Headless；Endpoints 只含 readiness 通过的 Pod。
28. ClusterIP 通常 ping 不通但能 curl 通；`port` 与 `targetPort` 别混淆。
29. Ingress 是七层规则对象，必须有 Ingress Controller；演进方向是 Gateway API。
30. ConfigMap/Secret 文件挂载会热更、环境变量不热更、`subPath` 不热更。
31. Secret 是 base64 不是加密，需 RBAC + etcd 加密 + KMS/Vault 加固。
32. PV 是集群资源、PVC 是命名空间申请；访问模式常用 RWO；`Retain` 保留数据。
33. `WaitForFirstConsumer` 解决云上多 AZ 的存储与节点不匹配。
34. **liveness 只探自己、失败重启；readiness 可探依赖、失败摘流量；startup 保护慢启动。**
35. 探针参数：initialDelaySeconds、periodSeconds、timeoutSeconds、failureThreshold、successThreshold。
36. requests 决定调度与 CPU 权重，limits 决定是否限流/OOMKilled。
