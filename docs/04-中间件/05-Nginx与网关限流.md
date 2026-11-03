# Nginx与网关限流
> **定位**：掌握 Nginx 的进程与事件模型、配置与负载均衡、限流与 HTTPS、高可用与故障排查，并厘清它与 API 网关的职责边界。
> **重要度**：★★★★☆
> **建议复习时长**：4 小时
> **关联文档**：[06-ZooKeeper与分布式协调](./06-ZooKeeper与分布式协调.md)、[05-SpringCloud微服务体系](../03-框架/05-SpringCloud微服务体系.md)、[01-计算机网络](../06-计算机基础/01-计算机网络.md)、[04-高并发高可用架构设计](../05-分布式与架构/04-高并发高可用架构设计.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| Nginx 架构与进程/事件模型 | ★★★★★ | 八股原理 |
| 配置文件结构与常用指令 | ★★★★☆ | 配置实践 |
| 反向代理与负载均衡 | ★★★★★ | 配置实践 |
| 动静分离、压缩、缓存与超时参数 | ★★★★☆ | 配置实践 |
| 限流实现（limit_req/limit_conn） | ★★★★★ | 场景题 |
| OpenResty/Lua 与 HTTPS 配置 | ★★★★☆ | 架构设计 |
| 高可用（Keepalived + VIP） | ★★★★☆ | 架构设计 |
| 常见故障排查（502/504/499/TIME_WAIT） | ★★★★★ | 故障复盘 |
| Nginx 与 API 网关职责划分 | ★★★★★ | 架构设计 |
| 综合题：请求全链路与 504 排查 | ★★★★★ | 场景题 |

## 1. Nginx 架构与进程模型
### 1.1 核心原理
Nginx 采用 **master + 多 worker** 模型：master 进程负责读取与校验配置、绑定端口、管理 worker（启动/停止/平滑重载），自身不处理请求；worker 进程以**单线程 + 异步非阻塞 + 事件驱动**方式处理连接（Linux 下用 **epoll** 的边缘触发/水平触发多路复用），一个 worker 用事件循环同时承载成千上万连接。请求处理划分为多个阶段（phase），每个连接由状态机推进，避免阻塞在 IO 上。单机连接上限 ≈ `worker_processes × worker_connections`（同时受 `worker_rlimit_nofile` 与系统 `ulimit -n` 限制）。CPU 密集操作（如 gzip 压缩、SSL 握手）会占用 worker 的 CPU，因此 worker 数一般设为 CPU 核数（`worker_processes auto`），并可用 `worker_cpu_affinity` 绑定核。
### 1.2 面试问答
**Q1：Nginx 为什么能支撑高并发？**
A：结论——事件驱动 + 异步非阻塞 + 少量进程，避免了"一连接一线程"的上下文切换与内存开销。原因：每个 worker 是单线程事件循环，用 epoll 一次等待大量就绪事件，只在就绪时处理，不会因某条慢连接阻塞其他连接（前提是没有阻塞式模块，如传统的磁盘 IO 或第三方同步模块）。

**Q2：worker_processes 设多少合适？**
A：结论——等于 CPU 核数（`auto`）。原因：worker 是单线程且不做计算密集任务，超过核数只会增加上下文切换；若负载以 SSL 握手、gzip 等 CPU 操作为主，更应严格等于核数。SSD/高 IO 场景可考虑略高，但需压测验证。

**Q3：worker_connections 是什么意思？**
A：单个 worker 同时能处理的连接数（默认 512，常设 1024~65536）。注意：作为反向代理时，一个客户端连接会同时占用"客户端连接 + 上游连接"两个连接，因此实际可承载的客户端连接约等于 `worker_connections / 2`。系统层面还要调 `worker_rlimit_nofile` 与 `ulimit -n`。

**Q4：平滑重载是怎么实现的？**
A：`nginx -s reload` 时 master 校验新配置后启动新 worker 并让旧 worker 处理完存量连接后退出（优雅关闭），因此不会中断服务。同理 `-s quit` 优雅退出、`-s stop` 立即停止、`-s reopen` 重开日志文件。**注意：已建立的连接仍按旧配置处理**，配置变更不会作用于长连接存量。

### 1.3 高频追问与陷阱
- 陷阱：以为 worker 是"一请求一线程"——单线程事件循环才是关键。
- 陷阱：只调 `worker_connections` 而忘记 `worker_rlimit_nofile` 与系统 ulimit，报 "worker_connections are not enough" 或 "too many open files"。
- 追问：为什么建议关掉 `accept_mutex`（1.11.3 起默认关闭）？现代内核下多 worker 同时 accept 的惊群问题已由内核 `EPOLLEXCLUSIVE` 缓解，关闭后吞吐更好。
### 1.4 配置/命令示例
```nginx
worker_processes auto;
worker_rlimit_nofile 65535;
events {
    use epoll;
    worker_connections 10240;
    multi_accept on;
}
```

```bash
nginx -t          # 校验配置
nginx -s reload   # 平滑重载（不中断存量连接）
```
### 1.5 一句话速记
master 管配置与 worker，worker 单线程 epoll 事件循环抗并发；容量 = worker 数 × worker_connections ÷ 2（代理场景）。

## 2. 配置文件结构与常用指令
### 2.1 核心原理
配置分五层：**main**（全局：user、worker_processes、error_log、pid）、**events**（连接模型）、**http**（日志格式、gzip、upstream、限流 zone、include）、**server**（虚拟主机：listen、server_name、ssl）、**location**（URI 匹配与处理逻辑）。**location 匹配优先级**：`=` 精确匹配 > `^~` 前缀匹配且不再检查正则 > `~`/`~*` 正则（按出现顺序，首个匹配即用）> 最长前缀匹配。常用指令：`root`（拼接 location 路径）、`alias`（替换 location 路径）、`try_files`（按序尝试文件，常用于 SPA 回退）、`rewrite`（`last`/`break`/`redirect`/`permanent`）、`return`（直接返回状态码或跳转）、`proxy_pass`。**关键陷阱**：`proxy_pass http://backend/`（带斜杠）会丢掉 location 匹配到的前缀，`proxy_pass http://backend`（不带斜杠）则保留完整 URI。
### 2.2 面试问答
**Q1：root 和 alias 的区别？**
A：`root` 是把 location 路径**拼接**到 root 之后（`location /static/ { root /data; }` → `/data/static/xxx`）；`alias` 是**替换** location 路径（`location /static/ { alias /data/; }` → `/data/xxx`）。结论——一个目录对多个 URL 前缀用 alias，标准目录树用 root。注意 location 以 `/` 结尾时 alias 也应一致，否则路径错乱。

**Q2：try_files 怎么用？**
A：`try_files $uri $uri/ /index.html;` 依次尝试：请求文件 → 对应目录 → 回退到 index.html（SPA 路由必需）。注意最后一项必须是"存在的东西"（文件或命名 location `@xxx`），否则会内部循环报错。反向代理场景常写 `try_files $uri @backend;`。

**Q3：location 正则和前缀谁优先？**
A：先找最长前缀匹配，若该前缀用 `^~` 修饰则直接使用；否则再按顺序尝试正则，任一正则匹配成功就用它；都不匹配则用之前记录的最长前缀。`=` 精确匹配优先级最高。
**Q4：server_name 的匹配顺序是什么？**
A：精确名字 > 以 `*` 开头的通配 > 以 `*` 结尾的通配 > 正则（按顺序）> `default_server`；未匹配的请求会落到默认 server（`listen 80 default_server;`），这是"域名串站"事故的常见根因。
### 2.3 高频追问与陷阱
- 陷阱：`proxy_pass` 带不带斜杠导致 404（后端收到的 URI 变了）。
- 陷阱：location 里写了 `add_header` 但被上层/下层的继承规则覆盖（子级有 add_header 时不再继承父级）。
- 追问：`include` 有什么用？把 server/upstream/限流 zone 拆到独立文件（如 `conf.d/*.conf`），便于多团队维护与灰度发布。
### 2.4 配置/命令示例
```nginx
server {
    listen 80;   server_name www.example.com;   root /data/www;
    location ^~ /static/ { alias /data/static/; expires 7d; }
    location /api/ { proxy_pass http://backend/; }        # 末尾斜杠会裁掉 /api 前缀
    location / { try_files $uri $uri/ /index.html; }      # SPA 回退
}
```
### 2.5 一句话速记
五层结构、location 优先级 `=` > `^~` > 正则 > 前缀；root 拼接、alias 替换；proxy_pass 末尾斜杠决定 URI 是否被裁剪。

## 3. 反向代理与负载均衡
### 3.1 核心原理
`upstream` 定义后端服务器组，负载策略：**轮询**（默认）、**weight**（加权轮询）、**ip_hash**（按客户端 IP 哈希，保证会话粘性）、**least_conn**（最少连接，适合长连接/耗时差异大的请求）、**hash $key consistent**（一致性哈希，节点变动只影响少量 key）、第三方 `fair`（按响应时间）。可用性控制：`backup`（备用节点，仅主节点全挂时启用）、`down`（手工下线）、`max_fails`（默认 1）+ `fail_timeout`（默认 10s）构成"被动健康检查"。`proxy_next_upstream` 定义哪些失败会转下一台（默认 error timeout）。要提升性能应配置 `upstream` 的 **keepalive 连接池**并配合 `proxy_http_version 1.1` 与 `proxy_set_header Connection ""`。
### 3.2 面试问答
**Q1：ip_hash 有什么问题？**
A：① 同一出口 IP（企业 NAT、运营商网关）的所有用户都打到同一台，负载严重倾斜；② 后端扩容/缩容会导致部分用户会话漂移（hash 取模变化）。结论——会话粘性优先交给 Redis/Token 共享会话解决，而不是 ip_hash；确需哈希时用 `hash $cookie_jsessionid consistent` 或按业务键一致性哈希。

**Q2：max_fails 和 fail_timeout 怎么协同？**
A：在 `fail_timeout` 时间内失败次数达到 `max_fails`，该节点被标记不可用并在随后一个 `fail_timeout` 内不再接收新请求（之后会试探性放一个请求探测）。这是**被动**健康检查，只有流量打到故障节点才会发现；开源版没有主动健康检查（`health_check` 指令为商业版 Plus 特性，或用 OpenResty/第三方模块实现）。

**Q3：为什么要配置 upstream keepalive？**
A：结论——复用后端长连接，减少三次握手与 TIME_WAIT。用法：`keepalive 64;`（每 worker 保持的空闲连接数）+ `proxy_http_version 1.1;` + `proxy_set_header Connection "";`。注意 keepalive 只对 HTTP/1.1 有效，且需要后端支持长连接。

**Q4：proxy_next_upstream 会不会导致重复提交？**
A：会。默认对 `error timeout` 重试下一台——若是 POST 请求且后端已开始处理但响应超时，重试就会重复写数据。建议：① `proxy_next_upstream_tries` 限制次数；② 幂等接口才允许重试（把非幂等写请求的 `proxy_next_upstream` 收紧为 `error`）；③ 关键写操作在后端做幂等。

### 3.3 高频追问与陷阱
- 陷阱：只配 weight 不考虑后端实际处理能力差异（CPU/内存/机型），导致长尾。
- 陷阱：用 `down` 手工下线节点后忘记恢复，容量长期不足。
- 追问：Nginx 如何做长连接负载均衡？`least_conn` 更适合长连接；`round_robin` 在长连接下可能不均。
### 3.4 配置/命令示例
```nginx
upstream backend {
    least_conn;                                   # 另有 weight / ip_hash / hash ... consistent
    server 10.0.0.11:8080 weight=3 max_fails=2 fail_timeout=10s;
    server 10.0.0.13:8080 backup;                 # 主节点全挂才启用
    keepalive 64;                                 # 复用后端长连接
}
# location /api/ 内四行：
proxy_pass http://backend;
proxy_http_version 1.1;  proxy_set_header Connection "";
proxy_next_upstream error timeout http_502 http_503 http_504;
```
### 3.5 一句话速记
默认轮询、加权看能力、ip_hash 有倾斜、least_conn 适合长连接、一致性哈希护缓存；max_fails+fail_timeout 只是被动检查，务必配上 keepalive 连接池。

## 4. 动静分离、压缩、缓存与超时参数
### 4.1 核心原理
**动静分离**：静态资源（js/css/图片/视频）由 Nginx 直接返回（`root`/`alias` + `expires`/`add_header Cache-Control`），动态请求 `proxy_pass` 到后端应用，减少后端压力。**压缩**：`gzip on` + `gzip_min_length 1k` + `gzip_comp_level 5` + `gzip_types` + `gzip_vary on`（配合 `Vary: Accept-Encoding`）；Brotli 需第三方模块。**代理缓存**：`proxy_cache_path`（定义缓存区）+ `proxy_cache`（引用）+ `proxy_cache_valid`（按状态码设时长）+ `proxy_cache_key`，适合"读多写少且可容忍短暂不一致"的接口。**关键超时/体积参数**：`keepalive_timeout`（客户端长连接，默认 75s）、`client_header_timeout`/`client_body_timeout`、`send_timeout`、`client_max_body_size`（**默认 1m**，上传大文件必须调大，否则 413）、`proxy_connect_timeout`（默认 60s）、`proxy_send_timeout`、`proxy_read_timeout`（默认 60s，三者中最常导致 504）、`proxy_buffering`/`proxy_buffers`/`proxy_buffer_size`。
### 4.2 面试问答
**Q1：动静分离的落地方式？**
A：① 域名/路径分离：静态走 CDN 或 Nginx 的 `/static/` 前缀（配 `^~` 或 `= ` 精确匹配 + `expires 30d`）；② 静态资源带 hash 文件名（`app.3f2a.js`）以便"永久缓存 + 版本更新"；③ HTML 入口文件不缓存（`Cache-Control: no-cache`），动态接口不缓存或按需缓存。

**Q2：gzip 会不会拖慢服务？**
A：会消耗 CPU（压缩比越高越慢）。结论——`gzip_comp_level 5` 左右是最佳折中，静态资源建议**预压缩**（构建时生成 `.gz`）配 `gzip_static on`，动态接口可对 JSON 开启 gzip 但不压缩图片/视频（已压缩格式收益为零）。另外 `gzip_min_length` 避免压缩小响应。

**Q3：client_max_body_size 不生效/报 413 怎么排查？**
A：413 可能在多层都发生：Nginx 的 `client_max_body_size`、API 网关的请求体限制、Spring 的 `spring.servlet.multipart.max-file-size`/`max-request-size`、云 SLB/WAF 的限制。要用 `curl -v` 逐层验证，并从最外层开始排查。

**Q4：三个 proxy 超时分别在什么阶段？**
A：`proxy_connect_timeout` 是与后端**建连**的时间（后端不可达/端口不通时在这里超时）；`proxy_send_timeout` 是把请求发给后端的两级写超时；`proxy_read_timeout` 是等待后端**响应的间隔**（后端处理慢最常卡在这里，得到 504）。调大超时只是延后失败，不是解决方案。

### 4.3 高频追问与陷阱
- 陷阱：默认上传限制 1MB 未调大，用户上传大图直接 413，且现象"前端说失败、后端无日志"。
- 陷阱：给所有响应加 `expires 30d` 却忘了入口 HTML，导致前端发版后用户看到旧页面。
- 追问：`sendfile on` 与 `tcp_nopush`/`tcp_nodelay` 的作用？sendfile 让内核直接拷贝文件到 socket（零拷贝），nopush 聚合包头发送，nodelay 禁用 Nagle 降低小包延迟。
### 4.4 配置/命令示例
```nginx
gzip on;  gzip_min_length 1k;  gzip_comp_level 5;  gzip_vary on;
gzip_types text/css application/json application/javascript;

proxy_cache_path /data/nginx/cache levels=1:2 keys_zone=api_cache:64m max_size=2g inactive=30m;
# location /api/goods/ { proxy_cache api_cache; proxy_cache_valid 200 30s; proxy_pass http://backend; }

client_max_body_size 50m;                                                  # 默认 1m，超限即 413
proxy_connect_timeout 5s; proxy_send_timeout 30s; proxy_read_timeout 30s;   # 默认各 60s
keepalive_timeout 65s;
```
### 4.5 一句话速记
静态分离 + hash 名永久缓存、gzip 5 级预压缩、代理缓存守住读多写少；client_max_body_size 默认 1m、proxy_read_timeout 默认 60s 是 413/504 的两个常客。

## 5. 限流实现与取舍
### 5.1 核心原理
**`limit_req`（漏桶）**：`limit_req_zone $key zone=name:size rate=Nr/s;` 定义限流区，`limit_req zone=name burst=M nodelay;` 在 location 使用。语义是"恒定速率出水"：超过速率的请求进入桶中排队（最多 burst 个），超出桶容量返回 `limit_req_status`（默认 503，通常改 429）；`nodelay` 表示桶内请求**立即处理**不排队（但桶容量照样占用），避免人为增加延迟。**`limit_conn`（并发连接数）**：`limit_conn_zone $binary_remote_addr zone=addr:10m;` + `limit_conn addr 10;`，限制单 IP 并发连接。还有 `limit_rate`（单连接带宽限速）、`max_conns`（upstream 单机最大连接，商业版）。Zone 内存：1MB 约能保存 1.6 万个状态条目（每条目约 64 字节，具体以版本与平台为准）。**限流的粒度取决于 key**：`$binary_remote_addr`（IP）、`$server_name`（按域名）、`$uri`（按接口）、`$http_x_forwarded_for`（真实客户端 IP，需可信代理）、或自定义变量（用户 ID、AppKey）。
### 5.2 面试问答
**Q1：limit_req 的 burst 和 nodelay 到底怎么理解？**
A：结论——`rate` 决定平均出水速率，`burst` 决定能容忍多少突发，`nodelay` 决定突发请求是"立即处理"还是"排队等令牌"。举例 `rate=10r/s burst=20 nodelay`：平均每秒放行 10 个，瞬时最多可再吞 20 个突发（立即处理），超过 20 的立即 429。若不加 nodelay，则突发请求会排队，客户端看到的延迟增加但不会被拒。

**Q2：为什么推荐返回 429 而不是 503？**
A：`429 Too Many Requests` 语义明确（被限流，可重试），便于客户端与监控区分；503 表示服务不可用，容易被误判为故障并触发下游熔断。做法：`limit_req_status 429;` 并配 `Retry-After` 头或用自定义错误页。

**Q3：怎么按接口和 IP 双维度限流？**
A：定义两个 zone：`limit_req_zone $binary_remote_addr zone=perip:10m rate=5r/s;` 与 `limit_req_zone $server_name$uri zone=perapi:10m rate=100r/s;`，在同一个 location 内同时 `limit_req zone=perip burst=10 nodelay;` 和 `limit_req zone=perapi burst=50 nodelay;`，任一触发即限流。对登录、短信等敏感接口使用更低速率。

**Q4：限流的取舍是什么？**
A：限流是"保护自己"的手段，但会牺牲部分正常用户。取舍点：① 阈值按压测容量设（如容量的 70%）而非拍脑袋；② 优先保护核心接口，非核心可直接拒绝；③ 令牌桶（允许突发）体验优于严格漏桶；④ 限流要能动态调整（OpenResty/Lua 或网关下发），大促临时放宽/收紧；⑤ 必须区分"正常突增"和"恶意流量"（后者限得更狠并可封禁）。**注意**：Nginx 计数是**每 worker 独立且本地**的，多机多 worker 场景下实际阈值是近似值，且节点重启会重置；精确分布式限流需 Redis + Lua 或网关实现。

### 5.3 高频追问与陷阱
- 陷阱：Nginx 的 `$remote_addr` 在 SLB/LB 后是负载均衡器 IP，必须用 `set_real_ip_from` + `real_ip_header X-Forwarded-For` 拿到真实 IP，否则限流会把所有人算成一个人。
- 陷阱：把 `limit_req` 放在错误的层级（写在 http 块不生效，需在 server/location 内引用 zone）。
- 追问：`limit_conn` 与 `worker_connections` 的关系？单 IP 并发限制能防止少数客户端占满 worker 连接，是抗慢攻击的常见手段。
### 5.4 配置/命令示例
```nginx
limit_req_zone $binary_remote_addr zone=perip:10m  rate=10r/s;
limit_req_zone $server_name$uri     zone=perapi:10m rate=200r/s;   # 接口维度
limit_conn_zone $binary_remote_addr zone=perconn:10m;
limit_req_status 429;   limit_conn_status 429;
set_real_ip_from 10.0.0.0/8;  real_ip_header X-Forwarded-For;  real_ip_recursive on;

location /api/sms/ {
    limit_req zone=perip burst=5 nodelay;   limit_req zone=perapi burst=20 nodelay;
    limit_conn perconn 10;                  proxy_pass http://backend;
}
```
### 5.5 一句话速记
limit_req 是漏桶（rate 出水、burst 容突发、nodelay 不排队），limit_conn 管并发；限流键要取真实 IP，返回 429，阈值来自压测而非拍脑袋。

## 6. OpenResty/Lua 与 HTTPS 配置
### 6.1 核心原理
**OpenResty** = Nginx + LuaJIT + 一组扩展模块（`lua-nginx-module`、`lua-resty-*` 库），可在请求各阶段（rewrite/access/content/log）执行 Lua，实现动态逻辑。典型场景：**灰度发布**（按 uid/设备/Cookie 取模决定走新版本上游）、**AB 测试**（按比例分流并埋点）、**动态黑名单**（从 Redis 读黑名单，`ngx.exit(403)`）、**动态限流**（`lua-resty-limit-traffic`）、**JWT/签名校验**、**动态 upstream**（配合 `balancer_by_lua` 做服务发现）。**HTTPS 配置**要点：`ssl_certificate` 必须配置**完整证书链**（站点证书 + 中间 CA，否则部分客户端报"证书链不完整"）、`ssl_certificate_key`、`ssl_protocols TLSv1.2 TLSv1.3`（禁用 TLS1.0/1.1）、`ssl_ciphers`（现代套件）、`ssl_session_cache shared:SSL:10m` + `ssl_session_timeout`、**HSTS**（`Strict-Transport-Security`）与 HTTP→HTTPS 跳转、OCSP stapling（`ssl_stapling on`）。
### 6.2 面试问答
**Q1：OpenResty 做灰度的核心思路？**
A：在 `access_by_lua_block` 中计算分流标识（如 uid 取模、Cookie、Header、App 版本、地域），命中新版本则改写 `upstream` 变量或走 `balancer_by_lua` 选择新服务集群；同时把分流标识写入请求头（如 `X-Gray: v2`）供后端日志追踪。要点：分流规则要**稳定**（同一用户多次请求结果一致）、可随时关闭、有回滚开关。

**Q2：Lua 里为什么要用 cosocket？**
A：Lua 中若用普通阻塞式 HTTP/Redis 客户端会阻塞整个 worker 的事件循环，导致该 worker 上其他请求全部卡住；`ngx.socket.tcp` / `resty.redis` 等 cosocket 是非阻塞的，配合事件循环不会阻塞。**这是 OpenResty 编码的第一条铁律**。

**Q3：HTTPS 配置最容易出什么问题？**
A：① 证书链不全（只配了站点证书，安卓/老客户端校验失败）；② 用了过旧协议（TLS1.0/1.1）被安全扫描判定不合规；③ 会话复用未开启，TLS 握手 CPU 消耗高；④ HSTS 打开后子域未完全支持 HTTPS，导致不可访问；⑤ HTTP→HTTPS 用 301 跳转，POST 请求会退化成 GET（需要 307/308 或前端改造成两段式）。

**Q4：TLS 性能怎么优化？**
A：① 开启 `ssl_session_cache shared:SSL:10m`（10MB 约可缓存数万会话）与 `ssl_session_tickets`；② 使用 TLS1.3（1-RTT、支持会话恢复 0-RTT）；③ 采用 ECDSA 证书（握手更快）或启用硬件的加解密；④ 在四层 LB 卸载 TLS 时注意拿不到客户端证书信息；⑤ 长连接复用减少握手次数。

### 6.3 高频追问与陷阱
- 陷阱：在 Lua 中用阻塞库导致 worker 卡死（现象是"偶发整站卡顿"）。
- 陷阱：`ssl_certificate` 只配了域名证书，PC 正常、App 报错——典型证书链问题。
- 追问：证书续期怎么做？使用 acme.sh/certbot 自动续期并 `nginx -s reload`；注意**续期后必须重载**才生效。
### 6.4 配置/命令示例
```nginx
server {
    listen 443 ssl http2;   server_name www.example.com;
    ssl_certificate     /etc/nginx/cert/fullchain.pem;   # 必须是完整链（站点 + 中间 CA）
    ssl_certificate_key /etc/nginx/cert/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;   ssl_session_cache shared:SSL:10m;
    add_header Strict-Transport-Security "max-age=31536000" always;
}
server { listen 80; return 301 https://$host$request_uri; }
```

```lua
-- 灰度：按 uid 取模，10% 流量走新版本
local uid = ngx.var.cookie_uid or ngx.var.remote_addr
if (tonumber(ngx.md5(uid):sub(1, 8), 16) % 100) < 10 then ngx.var.upstream_host = "new_backend" end
```
### 6.5 一句话速记
OpenResty 靠 Lua 做灰度/AB/黑名单/动态限流（铁律：只用 cosocket），HTTPS 要配全链、禁旧协议、开会话复用与 HSTS。

## 7. 高可用与常见故障排查
### 7.1 核心原理
**高可用**：Keepalived 基于 VRRP 在多台 Nginx 间维护一个 **VIP**，`state MASTER/BACKUP` + `priority` 决定谁来持有 VIP，`vrrp_script` 健康检查失败则降权触发切换，实现"双机主备"；更常用的"双主"是配两个 VIP 互为主备以提升资源利用率。上层还可叠加 LVS/云 SLB 做四层转发，下层用 DNS 轮询（注意 DNS 缓存与故障切换延迟）。**四类高频故障**：**502**（网关从上游收到无效响应——后端进程挂、端口不通、响应头过大、上游返回非法协议）；**504**（等待上游响应超时——后端处理慢、DB 慢、GC/锁等待、proxy_read_timeout 太小）；**499**（客户端主动断开——通常是客户端超时先于服务端返回，成本已消耗但结果丢弃）；**TIME_WAIT 过多**（短连接频繁创建关闭）。排障工具：`access_log` 中的 `$request_time`（总耗时）、`$upstream_response_time`（后端耗时）、`$upstream_addr`（实际转发的后端）、`$upstream_status`，以及 `error_log`。日志切割用 `kill -USR1`（重开日志句柄）或 logrotate 的 `copytruncate`/`create`。
### 7.2 面试问答
**Q1：502 和 504 的本质区别？**
A：502 是"连上了但响应不对/连不上"（Bad Gateway），504 是"连上了但等太久"（Gateway Timeout）。定位方法：看 error_log 关键字——`connect() failed (111: Connection refused)` 属 502（后端挂了）；`upstream timed out (110: Connection timed out) while reading response header` 属 504（后端太慢或超时配置过小）。

**Q2：接口偶发 504 怎么排查？**
A：先分清 504 发生在哪一层：① 看浏览器/客户端超时；② 看 Nginx 的 `$request_time` 与 `$upstream_response_time` 的差（差值大说明 Nginx 或网络有额外耗时，差值接近说明后端慢）；③ 看 `$upstream_addr` 是否总是同一台（单机故障）；④ 应用侧看线程栈、GC 日志、慢 SQL、外部依赖 RT；⑤ 查 `proxy_read_timeout` 是否被设得过小（默认 60s）。解决：优化后端（索引、缓存、异步化）、限流保护、必要时**临时**调大超时并把长任务改为异步 + 轮询结果。

**Q3：499 要不要处理？**
A：499 说明客户端已断开，Nginx 记录后终止上游请求（若上游支持会中断处理）。它本身不是故障码，但**激增意味着后端太慢**（用户等不了）或客户端超时设置过短。处理：查后端 RT、优化慢接口、告警 499 比例；注意把 499 从"错误率"里区分统计，否则会污染成功率指标。

**Q4：TIME_WAIT 过多怎么办？**
A：先定位在谁身上（`ss -s` / `netstat -n | awk '/^tcp/{++S[$NF]} END {for(a in S) print a, S[a]}'`）。常见原因与对策：① Nginx 作为客户端连后端用短连接 → 配 upstream `keepalive`（最有效）；② 客户端侧可开 `net.ipv4.tcp_tw_reuse=1` 复用 TIME_WAIT 连接（服务端一般不建议改 `tcp_tw_recycle`，已被内核移除）；③ 调大 `net.ipv4.ip_local_port_range` 与 `tcp_max_tw_buckets`；④ 业务层面用长连接/连接池。

**Q5：Keepalived 切换后有什么坑？**
A：① 切换期间存量连接中断（长连接会断，客户端要能重连）；② 若健康检查脚本写得过于简单（只探端口），Nginx 假死（worker 卡死但端口在）时不会切换，应探测 `/health` 且校验响应内容；③ 两个节点都要配 `vrrp_skip_check_adv_addr`、`nopreempt` 等参数避免反复抢占；④ VIP 漂移后要检查 ARP 与内核参数（`arp_ignore`/`arp_announce`）。

**Q6：日志切割怎么做？**
A：① 每天定时 `mv access.log access.log.$(date +%F)` 后执行 `nginx -s reopen`（或 `kill -USR1 <master_pid>`）让 Nginx 重开日志文件；② 或使用 logrotate 的 `copytruncate`（不重载但有丢日志风险）；③ 建议 access_log 使用 `buffer=32k flush=5s` 提升写入性能，同时配置 `include` 每个站点独立日志。
### 7.3 高频追问与陷阱
- 陷阱：把 502 一律当成"后端挂了"，实际可能是响应头过大（`upstream sent too big header while reading response header`，需调 `proxy_buffer_size`/`large_client_header_buffers`）。
- 陷阱：故障时第一反应是"调大超时"，掩盖了后端性能问题并让线程池被长期占用。
- 追问：如何统计"后端真实耗时"？`$upstream_response_time`（多个后端时为逗号分隔列表）与 `$upstream_connect_time`/`$upstream_header_time` 组合分析。
### 7.4 配置/命令示例
```nginx
log_format main '$remote_addr - $request_time $upstream_response_time $upstream_addr '
                '$status $upstream_status "$request" $body_bytes_sent "$http_user_agent"';
access_log /var/log/nginx/access.log main buffer=32k flush=5s;
proxy_buffer_size 16k;   proxy_buffers 4 32k;      # 防 "upstream sent too big header"
```

```bash
kill -USR1 $(cat /run/nginx.pid)    # 重开日志文件（切割后必须执行）
ss -s                               # 连接状态概览
```
### 7.5 一句话速记
502 看"连不上/响应非法"、504 看"等太久"、499 看"客户端先走"；TIME_WAIT 用 upstream keepalive 治，日志切割用 USR1，Keepalived 健康检查必须探业务接口。

## 8. Nginx 与 API 网关职责划分
### 8.1 核心原理
Nginx（含 OpenResty）属于**流量接入层**，关注"把请求高效、安全地送到下一跳"；API 网关（Spring Cloud Gateway / Kong / APISIX）属于**业务治理层**，关注"谁能访问哪个服务、按什么规则路由与治理"。生产中的常见分层是：`DNS/CDN → SLB/LVS → Nginx（接入层）→ API 网关（业务网关）→ 微服务（服务网格/注册发现）→ 基础设施`。

| 能力 | Nginx / OpenResty | Spring Cloud Gateway | Kong / APISIX |
| --- | --- | --- | --- |
| 语言与生态 | C + Lua（OpenResty） | Java（Spring 生态） | OpenResty/Nginx + Lua（APISIX 亦支持 etcd、插件热更新） |
| 动态配置 | 需 reload（OpenResty 可动态） | 天然动态（配置中心） | 管理 API + etcd，插件热生效 |
| TLS 卸载/静态资源 | 强项 | 弱 | 强（基于 Nginx） |
| 限流 | limit_req/limit_conn（本地、近似） | RequestRateLimiter + Redis（分布式） | 插件（本地/Redis/集群） |
| 鉴权 | 基础（JWT/黑白名单需自写 Lua） | 自定义 Filter + OAuth2 | 插件丰富（JWT/OAuth2/Key-Auth） |
| 灰度/AB | Lua 可实现 | 支持（谓词匹配 Header/Cookie/权重） | 插件支持 |
| 熔断降级/重试 | 有限（proxy_next_upstream） | 集成 Resilience4j | 插件支持 |
| 服务发现 | 需配合动态 upstream | 原生集成注册中心 | 支持 etcd/Consul/Nacos |
| 可观测 | 日志、stub_status、Prometheus 模块 | Micrometer/Actuator | 插件 + Prometheus |
| 性能 | 最高（C，接近内核态路径） | 中（JVM，堆与 GC 开销） | 高（Nginx 内核） |
| 典型定位 | 接入层、四七层代理、静态、基础限流、WAF 前置 | 业务网关（Java 团队、深度定制） | 通用 API 网关（多语言、插件化运营） |
### 8.2 面试问答
**Q1：为什么"接入层 Nginx + 业务网关"这样分层？**
A：职责分离与风险隔离：Nginx 承担 TLS 卸载、静态资源、DDoS/CC 基础防护、全局限流与第一层路由（成本低、性能高）；网关承担鉴权、精细限流、灰度、协议转换、聚合与埋点（逻辑重、变更频繁）。结论——把"变化快、逻辑重"的能力放在网关，"稳定、靠性能吃饭"的能力放在 Nginx。

**Q2：Nginx 能做限流，为什么还要网关限流？**
A：Nginx 限流是**单机、本地、近似**的（每 worker 独立计数），无法表达"全集群每秒 1 万个请求"或"按用户/AppKey 配额"这类全局语义；网关可基于 Redis/集群计数做分布式限流，并能与用户身份、套餐、租户绑定。实践中两层都要：接入层挡住异常流量，网关做业务配额。

**Q3：网关会带来哪些额外成本？**
A：① 多一跳，增加 1~5ms 与故障点；② JVM 网关（Spring Cloud Gateway）在超高并发下受 GC 与堆内存影响；③ 网关成了新的单点，必须集群化 + 健康检查；④ 配置治理复杂（路由、插件、版本）；⑤ 排障链路变长（要贯通 traceId）。这些成本必须用"统一鉴权、统一限流、统一灰度"的收益来覆盖。
**Q4：什么时候不需要 API 网关？**
A：服务数量少、团队小、鉴权与路由需求简单时，`Nginx + 应用内拦截器` 就足够，引入网关只增加复杂度；反之多团队、多语言、需要统一配额/审计/灰度/插件治理时，网关价值才明显。
### 8.3 高频追问与陷阱
- 陷阱：Nginx 与网关都做一遍限流，阈值叠加导致正常用户被误伤（应对齐阈值语义）。
- 陷阱：网关里写复杂业务逻辑（如聚合多个服务、做大量计算），导致网关越来越重、无法升级。
- 追问：Kong/APISIX 与 Nginx 的关系？二者都基于 OpenResty/Nginx，把 Nginx 的能力插件化、配置动态化，因此"会 Nginx"是理解它们的捷径。
### 8.4 配置/命令示例
```nginx
# 接入层：TLS 卸载 + 全局限流 + 转发到网关
server {
    listen 443 ssl;
    server_name api.example.com;
    limit_req zone=perip burst=100 nodelay;
    location / {
        proxy_pass http://gateway_cluster;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```yaml
# Spring Cloud Gateway：路由 + Redis 分布式限流
spring:
  cloud:
    gateway:
      routes:
        - id: order_route
          uri: lb://order-service
          predicates: [ Path=/api/order/** ]
          filters:
            - StripPrefix=2
            - name: RequestRateLimiter
              args: { redis-rate-limiter.replenishRate: 100, redis-rate-limiter.burstCapacity: 200 }
```
### 8.5 一句话速记
Nginx 管"接入与性能"，网关管"鉴权与业务治理"；Nginx 限流本地近似、网关限流全局精细，两层分工不重叠。

## 9. 综合题：请求全链路与 504 排查
### 9.1 核心原理
一次 HTTPS 请求的完整链路：**DNS 解析**（浏览器/本地缓存 → 递归解析）→ **TCP 三次握手** → **TLS 握手**（证书校验、密钥协商，1~2 RTT）→ **CDN/SLB/LVS**（四层转发或静态加速）→ **Nginx 接入层**（限流、TLS 卸载、静态资源命中、反代到网关）→ **API 网关**（鉴权、路由、限流、灰度）→ **微服务**（注册中心发现、负载均衡、业务处理）→ **中间件**（Redis/MySQL/MQ/ES）→ 逐层返回响应。排查任何"接口慢/超时"都要沿这条链路"从外到内、从后到前"定位并量化每段耗时。
### 9.2 面试问答
**Q1：说说从浏览器输入 URL 到后端返回的完整链路？**
A：按七段答：① DNS 解析域名（含缓存层级）；② TCP 握手（HTTPS 还要 TLS 握手，TLS1.3 为 1-RTT）；③ CDN/SLB 接入，静态资源可能直接返回；④ Nginx 处理（限流、TLS、路由、静态）；⑤ 网关鉴权与路由到具体微服务；⑥ 微服务查缓存/数据库/下游服务；⑦ 响应逐层回传，浏览器渲染（可能还有 HTTP 缓存协商 304）。加分点：能指出每段的常见瓶颈（DNS 劫持、握手 RTT、TLS 握手 CPU、后面再提服务端 RT）。

**Q2："接口 504"完整排查步骤？**
A：① 确认范围：全部接口还是特定接口/特定实例（看 `$upstream_addr` 分布）；② 量化耗时：`$request_time` vs `$upstream_response_time`（差值大 → Nginx/网络；接近 → 后端）；③ 看 error log 具体阶段（connect / reading response header / reading response body）；④ 后端排查：线程池是否打满、GC 是否频繁、慢 SQL、下游 RT、锁竞争；⑤ 中间件排查：Redis 大 key/慢命令、DB 慢查询与连接池、ES 慢查询、MQ 积压；⑥ 修复方案：加索引/缓存、异步化长任务（返回任务 ID 轮询）、限流与熔断保护、临时调大超时并设告警；⑦ 复盘：容量评估、超时链路对齐（客户端 < 网关 < Nginx < 服务内部超时，形成递减阶梯）。

**Q3：超时配置的最佳实践是什么？**
A：结论——超时时间应形成"外层大于内层"的递减阶梯：客户端超时 > Nginx `proxy_read_timeout` > 网关超时 > 服务内部 RPC/DB 超时。这样超时会在最早的一层被处理并返回明确错误，而不是层层累积导致用户看到"空白等待"。同时每个超时都要有日志与指标。

**Q4：一次请求要经过这么多层，怎么保证可观测？**
A：统一**链路追踪**：入口生成 traceId 并沿 `X-Request-Id`/W3C traceparent 透传（Nginx、网关、服务、MQ 全链路），日志中打 traceId、耗时、上游耗时；配合 metrics（QPS、RT 分位、错误率）与日志聚合，做到"用户报障 → 用 traceId 一次串起所有环节"。这是大厂排查效率的关键。
### 9.3 高频追问与陷阱
- 陷阱：一上来就调大超时，没有定位到具体环节（结果只是把问题从 504 变成 499 或线程池打满）。
- 陷阱：各层超时是"递增"的（内层比外层大），导致外层先超时、内层还在白干活并占用资源。
- 追问：为什么 499 会增加？客户端超时时间短于后端处理时间，用户主动断开；应优化后端 RT 或把长任务异步化。
### 9.4 配置/命令示例
```bash
awk '{print $7, $8, $9}' /var/log/nginx/access.log | sort -k2 -nr | head   # 找出耗时最长的请求
tail -f /var/log/nginx/error.log | grep -E "timed out|Connection refused"
curl -o /dev/null -s -w "connect:%{time_connect} ttfb:%{time_starttransfer} total:%{time_total}\n" https://api.example.com/health
```
### 9.5 一句话速记
全链路 = DNS→TCP/TLS→CDN/SLB→Nginx→网关→微服务→中间件；504 排查靠 `$request_time` 与 `$upstream_response_time` 的差、error log 阶段关键字与后端 RT 三分法。

## 10. 调优清单与运维实践
### 10.1 核心原理
调优分四层：**系统层**（`ulimit -n`、文件句柄、`somaxconn`、`tcp_tw_reuse`、内核参数与网络队列）、**Nginx 层**（worker_processes/worker_connections/worker_rlimit_nofile、sendfile/tcp_nopush/tcp_nodelay、gzip、keepalive、upstream keepalive、日志 buffer）、**配置结构层**（拆分 conf、统一 include、避免重复 location、正则不要滥用）、**架构层**（CDN 前置、动静分离、多级缓存、集群化 + VIP/SLB、容量与压测）。
### 10.2 面试问答
**Q1：Nginx 调优先看哪几个参数？**
A：① 连接能力：`worker_processes auto`、`worker_connections`、`worker_rlimit_nofile`，并用 `stub_status` 观察 `active`/`reading`/`waiting`；② IO：`sendfile on`、`tcp_nopush on`、`tcp_nodelay on`；③ 上游复用：upstream `keepalive` + `proxy_http_version 1.1` + `Connection ""`；④ 日志：`buffer=32k flush=5s` 降低磁盘 IO；⑤ 超时：按链路阶梯设置，不要全部用默认 60s。

**Q2：怎么观测 Nginx 的运行状态？**
A：开启 `stub_status`（暴露 active/accepts/handled/requests/reading/writing/waiting）或用 `nginx-prometheus-exporter`；核心指标：QPS、活跃连接、5xx 比例、499 比例、`$upstream_response_time` 分位、`upstream` 节点的 max_fails 触发次数、worker CPU 与负载。

**Q3：怎么做容量评估？**
A：① 用 `wrk`/`ab`/`vegeta` 压测单机在目标接口下的 QPS、P95/P99 延迟与错误率；② 按"业务峰值 QPS × 冗余系数（1.5~2）"规划实例数；③ 明确瓶颈（CPU 还是带宽还是上游），SSL 场景要按握手密集型单独压测；④ 大促前做全链路压测并预演扩容脚本。

**Q4：容器/K8s 环境下 Nginx 有什么变化？**
A：① 常以 Ingress Controller（ingress-nginx）形式运行，配置由 K8s 资源声明驱动，reload 由控制器触发；② Pod IP 动态变化，upstream 需动态解析（`resolver` + 变量型 `proxy_pass`）；③ `worker_processes auto` 在容器内可能读到宿主机核数，需用 CPU limit 或参数限制；④ 日志应输出到 stdout/stderr 交给日志采集；⑤ 限流与真实 IP 需要适配 `X-Forwarded-For`（云 LB 场景）。
**Q5：Nginx 有哪些安全加固项？**
A：`server_tokens off`（隐藏版本号，但只隐藏版本号）、`limit_except` 禁用不必要方法、限制请求体与请求头大小、用 `map` 拒绝可疑 UA/路径并 `return 444`、只允许必要 TLS 版本、禁止访问 `.git`/`.env` 等敏感路径，并前置 WAF 防注入与 CC 攻击。
### 10.3 高频追问与陷阱
- 陷阱：容器里 `worker_processes auto` 读到宿主机 64 核，启动 64 个 worker 却只有 2 核 quota，性能反而下降。
- 陷阱：`server_tokens off` 只隐藏版本号不隐藏"nginx"字样（要彻底隐藏需 `more_set_headers` 等模块）。
- 追问：静态资源为什么建议用 CDN？就近接入减少 RTT、卸载源站带宽、天然抗量；但要注意刷新与回源策略（版本化文件名 + 长缓存）。
### 10.4 配置/命令示例
```nginx
location = /nginx_status { stub_status; allow 10.0.0.0/8; deny all; }
server_tokens off;
sendfile on;  tcp_nopush on;  tcp_nodelay on;
```

```bash
wrk -t4 -c200 -d30s --latency http://127.0.0.1/api/goods/1   # 单机吞吐与延迟压测
```
### 10.5 一句话速记
调优看四层：系统句柄、Nginx 连接与 IO、上游 keepalive、架构前置；压测定容量，`stub_status` + 分位延迟做观测。

## 自测题（含答案要点）
1. **Nginx 的进程模型？** master 管配置与 worker，worker 单线程事件驱动（epoll）处理连接。
4. **location 匹配优先级？** `=` > `^~` > 正则（按顺序） > 最长前缀 > 默认。
5. **root 与 alias 的区别？** root 拼接 location 路径，alias 替换 location 路径。
6. **proxy_pass 末尾斜杠的影响？** 带斜杠会裁掉 location 匹配的前缀，不带则保留完整 URI。
9. **max_fails 与 fail_timeout 的作用？** 被动健康检查：超时窗口内失败达阈值则摘除该节点一段时间。
11. **proxy_next_upstream 的风险？** 非幂等请求重试可能重复写数据，应限制重试类型与次数。
12. **client_max_body_size 默认多少？** 1m，上传大文件必须调大，否则 413（多层都要检查）。
13. **三个 proxy 超时分别针对什么？** connect 建连、send 发送请求、read 等待响应（最常引发 504）。
14. **limit_req 是漏桶还是令牌桶？** 漏桶（恒定 rate，burst 为桶容量，nodelay 让桶内请求立即处理）。
16. **限流为什么推荐返回 429？** 语义明确（可重试），避免被下游误判为服务不可用。
17. **Nginx 限流的局限？** 本地近似（每 worker 独立）、节点重启重置，无法做全局精确分布式限流。
18. **为什么必须配 real_ip 模块？** 否则 `$remote_addr` 是 LB 的 IP，按 IP 限流会误伤所有人。
23. **Keepalived 健康检查怎么写才有效？** 探测业务接口（/health）并校验响应内容，而不是只探端口。
25. **接口 504 的排查主线？** 用 request_time 与 upstream_response_time 的差定位到 Nginx 还是后端，再看 error log 阶段与后端 RT/GC/慢 SQL。

## 背诵卡片
1. master 管配置与 worker，worker 单线程 epoll 事件循环。
3. 代理场景一个请求占用两个连接，容量约减半。
4. nginx -s reload 平滑重载；-s quit 优雅退出；-s reopen 重开日志。
6. root 拼接路径，alias 替换路径。
7. proxy_pass 末尾斜杠决定 URI 是否被裁剪。
8. try_files 是 SPA 回退与静态优先的标配。
9. 负载均衡：轮询、weight、ip_hash、least_conn、hash consistent。
10. ip_hash 易倾斜且扩缩容会漂移，会话共享优先用 Redis。
11. max_fails 默认 1、fail_timeout 默认 10s，是被动健康检查。
13. proxy_next_upstream 默认对 error/timeout 重试，非幂等接口要收紧。
14. client_max_body_size 默认 1m，上传失败先查 413。
15. proxy_connect/send/read_timeout 默认 60s，read 最常导致 504。
16. keepalive_timeout 默认 75s，长连接能减少握手开销。
17. gzip_comp_level 5 是性能与压缩比的最佳折中。
18. 静态资源用 hash 文件名 + 长缓存，入口 HTML 不缓存。
20. limit_conn 限制单 IP 并发连接，防慢攻击。
22. 限流 status 改 429，语义比 503 更准确。
23. 限流键建议用真实 IP（配 set_real_ip_from）或用户维度。
24. Nginx 限流是本地近似，精确分布式限流需 Redis/网关。
27. 禁用 TLS1.0/1.1，开启 ssl_session_cache 提升握手性能。
29. Keepalived 用 VRRP 维护 VIP，健康检查要探业务接口。
31. error_log 关键字：Connection refused 是 502，timed out 是 504。
32. 499 = 客户端主动断开，激增说明后端慢。
33. 用 request_time 与 upstream_response_time 的差定位耗时在哪一层。
34. TIME_WAIT 过多先治上游短连接（upstream keepalive）。
35. 日志切割：mv 后 kill -USR1 重开日志句柄。
36. 接入层 Nginx 管性能与防护，API 网关管鉴权、路由、灰度与精细限流。
38. 超时链路要成递减阶梯：客户端 > Nginx > 网关 > 服务内部。
39. 全链路 = DNS → TCP/TLS → CDN/SLB → Nginx → 网关 → 微服务 → 中间件。
40. 调优四层：系统句柄、Nginx 连接与 IO、上游 keepalive、架构前置与压测。
