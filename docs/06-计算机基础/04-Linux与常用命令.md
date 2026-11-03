# Linux 与常用命令

> **定位**：把"会敲命令"升级为"能定位问题"——命令速查、四类性能排查链路、日志分析、应急手册一次备齐。
> **重要度**：★★★★★（社招必考排查实操，问法高度贴近真实值班场景）
> **建议复习时长**：6 小时（命令速查 2h + 排查链路 2h + 日志与脚本 1h + 应急手册 1h）
> **关联文档**：[./01-计算机网络.md](./01-计算机网络.md)、[./02-操作系统.md](./02-操作系统.md)、[../01-Java基础/04-JVM与GC调优.md](../01-Java基础/04-JVM与GC调优.md)、[../07-工程化与DevOps/03-CICD与监控告警.md](../07-工程化与DevOps/03-CICD与监控告警.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| 目录结构与 FHS 标准 | ★★★☆☆ | 八股原理 |
| 文件权限、chmod/chown/umask、软硬链接 | ★★★★☆ | 命令速查 |
| 文件、文本、查找、压缩命令 | ★★★★★ | 命令速查 |
| 网络、进程、磁盘、系统信息命令 | ★★★★★ | 命令速查 |
| CPU 高排查链路 | ★★★★★ | 排查实操 |
| 内存高排查链路 | ★★★★★ | 排查实操 |
| IO 高排查链路 | ★★★★☆ | 排查实操 |
| 网络慢排查链路 | ★★★★★ | 排查实操 |
| 日志分析（grep/awk/sed） | ★★★★★ | 排查实操 / 命令速查 |
| 进程与信号（-9 vs -15） | ★★★★★ | 八股原理 / 排查实操 |
| 定时任务、SSH、环境变量、Shell 脚本 | ★★★★☆ | 命令速查 / 手写代码 |
| 线上应急手册 | ★★★★★ | 排查实操 / 场景题 |

## 1. 目录结构与 FHS 标准
### 1.1 核心原理
FHS 规定目录语义，核心是"静态/动态、可共享/不可共享"的划分。

| 目录 | 用途 | 关键点 |
| --- | --- | --- |
| `/` | 根 | 所有路径起点 |
| `/bin`、`/sbin` | 基本命令（sbin 多为管理员命令） | 现代发行版多软链到 `/usr/bin` |
| `/usr` | 系统级只读资源（bin/lib/share/include） | 体量大、可共享 |
| `/etc` | 配置文件 | `nginx.conf`、`my.cnf`、`/etc/profile` |
| `/var` | 可变数据（log、lib、spool） | **磁盘满的高发区** |
| `/tmp` | 临时文件 | 重启可能清空，慎放业务数据 |
| `/proc` | 内核与进程虚拟文件系统 | `/proc/<pid>/{status,fd,task}`、`/proc/cpuinfo` |
| `/sys` | 内核设备与 cgroup | `/sys/fs/cgroup/**` 看容器限额 |
| `/dev` | 设备文件 | `/dev/sda`、`/dev/null`、`/dev/urandom` |
| `/home`、`/root` | 用户家目录 | 部署账号与密钥位置 |
| `/opt`、`/usr/local` | 第三方/自装软件 | JDK 与中间件常放这里 |
### 1.2 面试问答
**Q1：`df` 显示有空间但写文件报 No space left on device？**
A：两种可能：**inode 耗尽**（`df -i` 检查，常见于海量小文件）或写到了被删除但仍被进程占用的文件（`lsof +L1` 找出后重启进程或截断 `/proc/<pid>/fd/<n>`）。边界：也可能是挂载点只读（`mount | grep ro`）或文件系统配额。
**Q2：`/proc` 里的文件能改吗？**
A：大部分可写但**写入即生效、不持久**（如 `/proc/sys/vm/drop_caches`），持久化要用 `/etc/sysctl.conf`。边界：容器内 `/proc` 与 `/proc/cpuinfo` 多为宿主机口径，不能直接当容器指标。
### 1.3 高频追问与陷阱
- `/var/log` 与 `/tmp` 最容易被写满，`/` 分区满会导致 ssh 登录失败与数据库写入异常；生产不要用 `/tmp` 存业务数据。
### 1.4 示例
`df -hT`（空间）、`df -i`（inode）、`lsblk`（块设备）、`mount | column -t`（挂载与只读状态）。
### 1.5 一句话速记
`配置在 /etc、数据在 /var、进程在 /proc、容器限额在 /sys/fs/cgroup。`

## 2. 文件权限、属主与链接
### 2.1 核心原理
权限 = 属主（u）/属组（g）/其他（o）各 3 位：`r(4)`、`w(2)`、`x(1)`；**目录的 `x` 表示可进入、`r` 表示可列出**。特殊权限：SUID（4）、SGID（2）、Sticky Bit（1，目录内文件仅属主可删，如 `/tmp`）。`umask` 决定新建文件默认权限（默认 022 → 文件 644、目录 755）。

| 维度 | 硬链接 | 软链接（符号链接） |
| --- | --- | --- |
| 本质 | 同一 inode 的另一个目录项 | 独立文件，内容为目标路径 |
| 跨文件系统 / 指向目录 | 不支持 | 支持 |
| 目标删除后 | 仍可访问（链接数减 1） | 变死链接 |
| 创建 | `ln src dst` | `ln -s src dst` |
### 2.2 面试问答
**Q1：`chmod 755` 与 `chmod u+x` 的关系？**
A：数值法是绝对设置、符号法是相对增减；`755` 即 `rwxr-xr-x`，`u+x` 只给属主加执行位。边界：`chmod -R 777 /` 会破坏系统安全模型且难以修复。
**Q2：SSH 私钥权限为什么必须 600？**
A：OpenSSH 会检查 `~/.ssh` 与私钥权限，组/其他可读写时直接拒绝（`Permissions 0644 ... are too open`）；修法 `chmod 700 ~/.ssh && chmod 600 ~/.ssh/id_rsa`。
**Q3：删除文件需要什么权限？**
A：需要**所在目录的 `w` 与 `x`**，不需要文件本身写权限；Sticky 位目录还要求是文件属主或 root。这是"误删他人文件"的权限根源，部署目录不要设 777。
### 2.3 高频追问与陷阱
- `chown -R`/`chmod -R` 在大目录（几十万文件）上很慢会拖慢应用，优先只改必要层级；软链接的相对路径相对链接所在目录解析，跨目录易指错。
### 2.4 示例
`chmod 750 /data/app`、`chown -R app:app /data/app`、`chmod +t /data/shared`（Sticky）、`ls -li file`（inode 与链接数）、`readlink -f link`（解析真实路径）。
### 2.5 一句话速记
`删除文件看目录权限；硬链接共享 inode，软链接只是路径指针。`

## 3. 命令速查表
### 3.1 核心原理
按"文件 → 文本 → 查找压缩 → 网络/进程/磁盘/系统"分类记忆，每类掌握一个主力 + 一个替补即可覆盖 90% 场景。

| 文件与查看 | 常用写法 | 文本处理 | 典型用法 |
| --- | --- | --- | --- |
| `ls` | `ls -alht`（隐藏/可读/按时间） | `grep` | `-n`、`-i`、`-v`、`-c`、`-A/-B/-C n`、`-E`、`-r` |
| `cp`/`mv`/`rm` | `cp -a` 保权限；`rm -rf` 前先 `ls` 确认 | `sed` | `-i 's/a/b/g'`、`-n '10,20p'`、`-i.bak` 先备份 |
| `head`/`tail` | `tail -f`、**`tail -F`**（轮转后仍跟随） | `awk` | `'{print $1,$9}'`、`-F,`、`END{print s}` |
| `less` | `less +F` 跟随、`/pattern` 搜索 | `sort`/`uniq` | `sort -k2 -nr`、`uniq -c`（**必须先 sort**） |
| `wc` | `-l` 行数、`-c` 字节 | `cut`/`tr` | `cut -d: -f1`、`tr -d '\r'` |
| `du`/`df` | `du -sh * \| sort -rh`、`df -hT`、`df -i` | `xargs`/`tee` | `xargs -n1 -P8`（并行）、`\| tee -a log` |
| `stat`/`file` | 元数据与文件类型 | `column` | `\| column -t` 对齐输出 |

| 查找/压缩/传输 | 命令 | 网络/进程/系统 | 命令 |
| --- | --- | --- | --- |
| 按名/类型/大小/时间找 | `find /data -name '*.log' -mtime +7 -size +100M` | 连通与路径 | `ping -c 5`、`mtr -rwzbc 50 host` |
| 查找并批量操作 | `find ... -delete`、`-exec rm -f {} +` | HTTP 探测 | `curl -w '...' -o /dev/null -s url`、`wget -c` |
| 定位命令 | `which`、`type -a`、`command -v` | 连接状态 | `ss -lntp`、`ss -tanp state time-wait`、`ss -s`、`netstat -s` |
| 按内容找文件 | `grep -rl 'OutOfMemory' /log` | 抓包 / 端口 | `tcpdump -i any -nn -s0 -w a.pcap port 8080`、`lsof -i:8080` |
| tar | `tar -czf a.tgz dir`、`-xzf -C /opt`、`-tzf` 预览 | 进程 | `ps -ef`、`top -Hp <pid>`、`pkill -f`、`kill -15/-9`、`pstree -ap` |
| zip/gzip | `zip -r a.zip dir`、`unzip -l/-d`、`gzip`/`zgrep` 流式处理 | 后台与作业 | `nohup cmd > out.log 2>&1 &`、`setsid`、`jobs`/`bg`/`fg` |
| 切分与校验 | `split -b 100M`、`cat part_* > big.tar`、`md5sum` | 系统信息 | `uname -a`、`hostname`、`date +%F' '%T`、`nproc`、`ulimit -a` |
| 服务管理 | `systemctl status/restart x`、`journalctl -u x -f` | 内存/负载 | `free -h`、`uptime`、`vmstat 1` |
| 磁盘 | `df -hT`、`df -i`、`du -sh *` | 磁盘与细分 | `iostat -xdm 1`、`iotop -o`、`pidstat -u -r -d -w -p <pid>` |
### 3.2 面试问答
**Q1：10GB 日志怎么处理？**
A：先看规模（`wc -l`、`head -1`、`tail -1`），全程用流式工具：`sed -n '1,100p'`、`awk 'NR>=x&&NR<=y'`、`grep`；按时间切片先缩小范围；压缩文件用 `zcat`/`zgrep` 直接流式处理，不要先解压。边界：`less` 可以打开大文件（按需读取），`vim` 不行。
**Q2：`find` 与 `locate` 该用哪个？**
A：`find` 实时遍历、准确、可组合条件但慢；`locate` 查索引库（`updatedb` 更新）毫秒级但不含新文件。生产忌用 `find /` 全盘扫描，务必限定目录与 `-maxdepth`。
**Q3：`tail -f` 与 `tail -F` 的区别？**
A：`-f` 跟踪文件描述符，logrotate 重命名后失效（继续读旧 inode）；`-F` 按名字重开文件，**日志轮转场景必须用 -F**。
### 3.3 高频追问与陷阱
- `grep -c` 统计匹配**行数**，统计出现次数用 `grep -o ... | wc -l`；`uniq` 只合并相邻重复行，必须先 `sort`；`ps` 的 `%CPU` 是生命周期均值，瞬时热点看 `top`/`pidstat`。
### 3.4 示例
`du -ah /var/log | sort -rh | head -10`；`find /data -name '*.jar' -mmin -30 -ls`；`sed -n '1000000,1000050p' app.log`。
### 3.5 一句话速记
`grep 找行、awk 取列、sed 改内容、sort+uniq 做统计、find 按条件找文件。`

## 4. 性能排查工具链
### 4.1 核心原理
统一方法论：**确认现象 → 定位资源维度 → 定位进程 → 定位线程 → 定位代码**；每层都要有判断标准与对照实验，不能只看一个数字下结论。

| 维度 | 第一层 | 第二层 |
| --- | --- | --- |
| CPU | `top`、`uptime` | `top -Hp`、`pidstat -u`、`perf top`、`jstack` |
| 内存 | `free -h`、`vmstat 1` | `ps aux --sort=-rss`、`pmap -x`、`jcmd`/`jmap` |
| IO | `iostat -xdm 1` | `iotop -o -P`、`pidstat -d`、`lsof <file>` |
| 网络 | `ping`/`mtr`、`ss -s` | `ss -tanp`、`tcpdump`、`nstat` |
| 句柄/线程 | `ulimit -n`、`ls /proc/<pid>/fd \| wc -l` | `lsof -p`、`cat /proc/<pid>/status` |

**CPU 高（可复制序列 + 判断标准）**
```bash
uptime; nproc; top -b -n 1 | head -15          # 1) 负载是否超过核数，谁在吃 CPU
top -Hp <pid> -b -n 1 | head -20               # 2) 定位热点线程
printf '%x\n' <tid>; jstack <pid> | grep -A 30 "nid=0x<hex>"   # 3) 线程栈
pidstat -u -p <pid> 1 5                        # 4) 交叉验证
```
`us` 高且热点在业务代码 → 死循环、正则回溯、大对象序列化；`sy` 高 → 系统调用/上下文切换过多（看 `vmstat` 的 `cs`、`in`）或锁竞争；`wa` 高 → 其实是 IO 问题；单线程持续 100% 基本可判死循环；GC 线程高则转内存/GC 排查；负载高但 CPU 不忙 → 大量 D 状态（IO 阻塞）。

**内存高**
```bash
free -h; vmstat 1 5                            # available 才是真可用；si/so 非 0 说明换页
ps aux --sort=-rss | head -10
cat /proc/<pid>/status | grep -E 'VmRSS|VmSize|Threads'
pmap -x <pid> | sort -k3 -nr | head -10        # 堆外与线程栈可见
jcmd <pid> GC.heap_info; jmap -histo:live <pid> | head -30    # 后者会触发 Full GC，慎用
```
`available` 低且 `si/so` 持续大于 0 → 真内存不足；`buff/cache` 高但 `available` 多属正常（Page Cache 可回收）；Java 进程 RSS 远大于 `-Xmx` → 堆外内存（DirectByteBuffer、Metaspace、线程栈、代码缓存）；容器内进程被 `kill -9` 且无 JVM 日志 → cgroup 超限触发 OOM Killer，查 `dmesg | grep -i 'killed process'`。

**IO 高**
```bash
iostat -xdm 1 5; iotop -o -P -d 2; pidstat -d -p <pid> 1 5
lsof -p <pid> | grep -E 'REG|DIR' | head; df -h; df -i
```
`%util` 接近 100% + `await` 升高 + `aqu-sz` 增大 → 磁盘饱和，找高写入进程（日志刷屏、大量小文件写、fsync 频繁）；`await` 高但 `%util` 低 → 远程存储（NFS/云盘）延迟；大量 D 状态 + IO 高 → 磁盘或文件系统故障，查 `dmesg`。

**网络慢**
```bash
mtr -rwzbc 50 <target>
ss -s; nstat -az | grep -Ei 'Retrans|ListenDrop|Syncookies'
ss -tan state established | wc -l; ss -tanp | grep -E 'CLOSE-WAIT|TIME-WAIT' | head
tcpdump -i any -nn -s0 -w /tmp/net.pcap host <ip> and port <port>
curl -o /dev/null -s -w 'dns=%{time_namelookup} conn=%{time_connect} ttfb=%{time_starttransfer} total=%{time_total}\n' <url>
```
重传率（`TcpRetransSegs/TcpOutSegs`）> 1% 或持续增长 → 链路问题；`CLOSE-WAIT` 多 → 应用未关连接（代码问题）；`TIME-WAIT` 多 → 短连接过多；`ListenOverflows` 增长 → accept 队列溢出；`curl` 的 `connect` 大 → 网络/握手问题，`ttfb` 大而 `connect` 小 → 服务端处理慢。
### 4.2 面试问答
**Q1：线上 CPU 100% 的完整处置顺序？**
A：`uptime`/`top` 确认进程 → `top -Hp` 找热点线程 → `jstack`/Arthas `thread -n 3` 定位栈 → 若在 GC 线程则抓 GC 日志与 `jmap -histo` → 常见根因是死循环/无边界重试、正则回溯、频繁 Full GC、线程数暴涨、同步日志刷盘 → 先止损（摘流量、限流、重启）再定位根因。边界：**重启前必须保留现场**，否则无法复盘。
**Q2：`free` 的 buff/cache 很大要处理吗？**
A：不需要，Linux 主动把空闲内存当 Page Cache，真正指标是 `available`；只有 `available` 长期偏低且 `si/so` 非零才需干预。
**Q3：负载高但 CPU 不忙，为什么？**
A：负载统计"运行中 + 不可中断睡眠"的进程，大量 D 状态（IO 阻塞、NFS 卡死）也会推高负载；排查 `ps -eo stat,pid,cmd | awk '$1 ~ /D/'` 与 `iostat`。边界：容器内 `uptime` 显示宿主机负载。
**Q4：服务无响应但进程还在，怎么查？**
A：`ss -lntp` 确认端口在听 → `jstack` 看线程是否全卡在 `BLOCKED`（锁）/`WAITING`（线程池、连接池满）/`RUNNABLE`（死循环或 IO）→ `jstat -gcutil <pid> 1000` 看 Full GC 停顿 → 查下游（DB/Redis/RPC）是否假死。边界：`ss -lnt` 的 `Recv-Q` 接近 `Send-Q` 是 accept 队列溢出的征兆。
### 4.3 高频追问与陷阱
- 排查前先记录时间点并留证据（`top`/`jstack`/`iostat` 输出）；`jstack` 对 `RUNNABLE` 的 native 方法看不到 C 栈，需 `perf`；容器内指标受 cgroup 限制（`cpu.max`、`memory.max`），`top` 显示的是宿主机口径。
### 4.4 示例
```bash
{ date; uptime; free -h; df -hT; ss -s; top -b -n 1 | head -20; } | tee /tmp/diag-$(date +%F-%H%M).txt
```
### 4.5 一句话速记
`CPU 高看 top -Hp + jstack；内存看 available 与 RSS 差异；IO 看 %util 与 iotop；网络看 mtr 与重传。`

## 5. 日志分析实战
### 5.1 核心原理
四类动作：**实时跟随、条件过滤、统计聚合、批量修改**；工具分工为 `grep` 过滤、`awk` 统计、`sed` 修改、`sort|uniq` 计数。
### 5.2 面试问答
**Q1：如何统计接口 PV 与状态码分布？**
A：`awk` 按列提取再 `sort | uniq -c | sort -rn` 聚合（见示例）；日志格式规整的话最稳的是改 JSON 日志再用 `jq` 解析。边界：跨几十台机器必须用 ELK/Loki，不是 `grep` 能解决的。
**Q2：如何按时间切片分析？**
A：`sed -n '/10:00/,/10:05/p'`（要求时间格式可字典序比较，`YYYY-MM-DD HH:MM:SS` 满足）；压缩日志用 `zgrep`。不要用 `%d/%m/%Y` 直接比大小。
**Q3：如何在日志里追一次请求的完整链路？**
A：靠 **TraceId** —— `grep <traceId> *.log` 先找入口再顺调用链看下游；没有 TraceId 时用"时间窗 + 订单号/用户 ID"缩小范围。
### 5.3 高频追问与陷阱
- `LC_ALL=C grep` 可显著加速大文件过滤；`awk` 的列含义必须与日志格式核对（默认按空白分割，CSV 用 `-F,`）；统计结果统一 `sort -rn | head` 收尾。
### 5.4 示例
```bash
tail -F /var/log/app/app.log                                   # 轮转安全地跟随
grep -n -A3 -B3 'OutOfMemoryError' app.log                     # 带上下文看异常
awk '{print $9}' access.log | sort | uniq -c | sort -rn | head # 状态码分布
awk '{print $1}' access.log | sort -u | wc -l                  # UV（按 IP 去重）
awk '$NF > 1000 {print}' access.log | head -20                 # 慢请求（末列为耗时 ms）
sed -i.bak 's/10.0.0.8/10.0.0.9/g' config/*.conf              # 批量替换并备份
zgrep -c 'ERROR' app.log.2024-05-01.gz                          # 压缩日志不落地处理
```
### 5.5 一句话速记
`tail -F 跟随、grep 过滤、awk 聚合、sort|uniq 计数、sed 批量改；全链路靠 TraceId。`

## 6. 进程与信号
### 6.1 核心原理
| 信号 | 编号 | 语义 | 可否捕获 |
| --- | --- | --- | --- |
| SIGTERM | 15 | 请求终止 | 可捕获，**优雅停机首选** |
| SIGKILL | 9 | 强制杀死 | 不可捕获 |
| SIGHUP | 1 | 挂断/重载配置 | 可捕获（Nginx reload） |
| SIGINT / SIGQUIT | 2 / 3 | Ctrl+C / 退出并转储（Java 打线程栈） | 可捕获 |
| SIGSTOP / SIGCONT | 19 / 18 | 暂停 / 继续 | 不可捕获 / 可捕获 |
| SIGPIPE | 13 | 写入已关闭管道 | 可捕获 |

`kill -15` 让进程执行清理（关连接、刷盘、注销服务，JVM 触发 shutdown hook）；`kill -9` 由内核直接回收，**不执行任何清理**：可能丢未刷盘数据、残留锁文件与注册中心节点、连接未四层关闭（对端看到 RST）。
### 6.2 面试问答
**Q1：为什么优先 `kill -15`？**
A：`-15` 让应用优雅退出（完成在途请求、释放资源、反注册），`-9` 是最后手段；若 `-15` 后不退出要先查原因（卡在不可中断 IO、shutdown hook 死锁、线程不响应中断）。边界：容器 `docker stop` 默认 SIGTERM 并等 10 秒再 SIGKILL，K8s 用 `terminationGracePeriodSeconds`，应用必须能在 grace 期内退出。
**Q2：`&`、`nohup`、`setsid`、`disown` 的区别？**
A：`&` 只放后台，终端关闭（SIGHUP）可能被杀；`nohup` 忽略 SIGHUP 并重定向输出；`setsid` 让进程脱离终端会话成为新会话首进程；`disown` 把作业从当前 shell 作业表移除。生产应由 systemd/supervisor 托管（自动重启、日志切分、资源限制）。
**Q3：`ps -ef`、`ps aux`、`pstree` 的差别？**
A：`ps -ef` 是 System V 风格，`ps aux` 是 BSD 风格（含 `%CPU/%MEM/RSS/STAT`），`pstree` 展示父子关系便于找"谁启动了谁"。
### 6.3 高频追问与陷阱
- `kill -0 <pid>` 只检测进程存在与权限；`kill -9` 不会释放 `flock` 锁文件，需清理陈旧锁；部署脚本直接 `kill -9` 是"优雅停机失效"的常见原因。
### 6.4 示例
`ps -eo pid,ppid,stat,etime,cmd --sort=-pcpu`；`kill -15 <pid>; sleep 5; kill -0 <pid> && echo alive`；`systemctl restart myapp; journalctl -u myapp -n 200 -f`。
### 6.5 一句话速记
`15 优雅（跑 shutdown hook）、9 强杀（丢清理）；生产用 systemd 托管而不是 nohup &。`

## 7. 定时任务 crontab
### 7.1 核心原理
格式 `分 时 日 月 周 命令`（`*` 任意、`*/n` 步长、`a-b` 区间、`a,b` 列举）；`crontab -e` 编辑、`-l` 查看、`-r` 清空（危险）。
```bash
30 3 * * * find /var/log/app -name '*.log' -mtime +7 -delete                  # 每天 03:30 清理
*/5 * * * * /opt/scripts/health.sh >> /var/log/scripts/health.log 2>&1        # 每 5 分钟健康检查
```
### 7.2 面试问答
**Q1：crontab 任务不执行的常见原因？**
A：1）**环境变量**：cron 的 PATH 极简（通常只有 `/usr/bin:/bin`），脚本须用绝对路径或自己 source；2）**工作目录**是家目录，脚本要 `cd`；3）脚本缺执行权限；4）输出未重定向，命令输出堆积成邮件且无记录；5）`crond` 没跑（`systemctl status crond`）；6）`%` 需转义为 `\%`。边界：最小粒度 1 分钟，秒级任务要拆分或用专用调度器。
**Q2：如何避免定时任务并发重入？**
A：单机用 `flock -n /tmp/job.lock -c 'script.sh'`（拿不到锁立即退出）；多实例部署必须用分布式锁（Redis/ZK）或调度平台，否则每台机器都会跑一遍。
**Q3：cron 与 systemd timer 怎么选？**
A：简单周期任务用 cron；需要依赖管理（网络就绪后执行）、journald 日志、失败重启与随机延迟时用 systemd timer；容器环境应由 XXL-JOB/K8s CronJob 触发。
### 7.3 高频追问与陷阱
- 改系统时间或时区会导致 cron 跳过一次或重复执行；`crontab -r` 无确认即清空全部任务，生产要防止手滑。
### 7.4 示例
`crontab -l`；`grep CRON /var/log/cron | tail -20`；`flock -n /tmp/clean.lock -c '/opt/scripts/clean.sh'`。
### 7.5 一句话速记
`分时日月周 + 绝对路径 + 输出重定向 + flock 防重入；多实例任务必须用分布式锁。`

## 8. SSH 免密登录与文件传输
### 8.1 核心原理
免密登录 = 客户端用私钥签名挑战、服务端用 `~/.ssh/authorized_keys` 中的公钥验证；生成用 `ssh-keygen -t ed25519`，分发用 `ssh-copy-id`。

| 场景 | 命令 |
| --- | --- |
| 上传文件 / 下载目录 | `scp app.jar user@host:/opt/app/`、`scp -r user@host:/var/log/app ./logs` |
| 指定端口与密钥 | `scp -P 2222 -i ~/.ssh/id_ed25519 a.txt user@host:/tmp` |
| 增量同步（推荐） | `rsync -avz --delete --exclude 'logs/' ./dist/ user@host:/opt/app/` |
| 断点续传 / 限速 | `rsync -avzP --partial big.tar user@host:/data/`、`--bwlimit=5000` |
| 本地端口转发 | `ssh -L 3307:127.0.0.1:3306 user@db-host` |
| 远程端口转发（内网穿透） | `ssh -R 8080:localhost:8080 user@public-host` |
| SOCKS 代理 / 跳板机 | `ssh -D 1080 -N user@jump`、`ssh -J jump@j:22 user@inner` |
### 8.2 面试问答
**Q1：`scp` 与 `rsync` 的区别？**
A：`scp` 每次全量、简单但不支持增量/续传/排除；`rsync` 用差分算法只传变化块（`--partial` 续传、`-z` 压缩、`--exclude` 排除），大目录发布首选。**注意** `--delete` 会删除目标端多余文件，写错路径会删生产文件，务必先 `--dry-run`。
**Q2：`-L` 与 `-R` 怎么记？**
A：`-L`：本地端口 → 远端视角的地址（把远端服务拉到本地端口）；`-R`：远端端口 → 本地视角的地址（把本地服务推到远端端口，即内网穿透）。边界：`-R` 要在外部访问需远端 `GatewayPorts` 允许非回环绑定。
### 8.3 高频追问与陷阱
- 服务器重装后 `known_hosts` 指纹变化会报错，用 `ssh-keygen -R host` 清理；`rsync` 源路径末尾斜杠语义不同（`src/` 同步内容、`src` 同步目录本身）是经典事故；SSH 断开后长任务被杀，需 `nohup`/`tmux` 或 systemd 托管。
### 8.4 示例
`ssh-keygen -t ed25519 -C "deploy@ci"`；`ssh-copy-id deploy@10.0.0.8`；`ssh -o BatchMode=yes -o ConnectTimeout=5 deploy@10.0.0.8 'systemctl restart myapp'`。
### 8.5 一句话速记
`-L 本地转发、-R 反向穿透、-J 跳板；rsync 增量同步注意尾部斜杠与 --delete。`

## 9. 环境变量与 profile 加载顺序
### 9.1 核心原理
| 文件 | 加载时机 | 说明 |
| --- | --- | --- |
| `/etc/profile` | 登录 shell | 全局，并调用 `/etc/profile.d/*.sh` |
| `~/.bash_profile` / `~/.bash_login` / `~/.profile` | 登录 shell | 按顺序取第一个存在的 |
| `/etc/bashrc`（bash.bashrc） | 交互式非登录 shell | 全局 |
| `~/.bashrc` | 交互式非登录 shell | 用户级别名/函数 |
| `~/.bash_logout` | 退出登录 | 清理 |

**登录 shell**（ssh 登录、`su -`）读 profile 链；**非登录交互式 shell** 读 `~/.bashrc`；**非交互 shell**（脚本、cron、systemd、CI）两种都不读，只有 `BASH_ENV` 指定时才读。
### 9.2 面试问答
**Q1：`command not found` 如何排查？**
A：`echo $PATH` 看是否包含目标目录 → `which`/`type` 确认命令存在 → 确认执行环境是登录还是非交互（cron、systemd、CI 都不读 profile）→ 检查是否只在 `~/.bashrc` 里 export 而脚本用 `sh` 执行。修法：脚本内显式 `export PATH=/opt/jdk/bin:$PATH` 或在 unit 文件写 `Environment=`。
**Q2：PATH 顺序为什么重要？**
A：同名命令按 PATH 从左到右第一个命中，多版本 JDK/Python 并存时"版本回退"往往是 PATH 顺序问题；`which -a java` 可列出全部候选。
**Q3：`export` 与直接赋值的区别？**
A：`VAR=x` 只在当前 shell 有效；`export VAR=x` 进入进程环境变量、子进程可见。想让 Java 进程看到必须 export；容器里通过 `-e`/K8s env 注入的变量对 PID 1 可见但需被继承。
### 9.3 高频追问与陷阱
- `ssh host 'cmd'` 属于非交互非登录 shell，不加载 `~/.bashrc`；`System.getenv` 读环境变量、`System.getProperty` 读 `-D` 参数，排查"配置没生效"先确认走哪条路径。
### 9.4 示例
`echo $PATH | tr ':' '\n'`；`type -a java`；`env | sort | grep -iE 'java|lang|tz'`；脚本头部 `export JAVA_HOME=/opt/jdk-17; export PATH="$JAVA_HOME/bin:$PATH"`。
### 9.5 一句话速记
`登录 shell 读 profile，非登录交互读 bashrc，脚本与 cron 都不读——必须显式 export。`

## 10. Shell 脚本基础与运维脚本
### 10.1 核心原理
- 变量：`VAR=value`（等号两侧不能有空格）、引用用 `"$VAR"`、`${VAR:-default}` 给默认值。
- 条件与循环：`[ "$a" = "$b" ]`（注意空格）、`[[ ... ]]` 支持正则、`(( a > 1 ))` 数值比较；`for f in *.log`、`while read -r line; do ...; done < file`。
- 函数：`name() { local x=$1; ...; }`；`trap 'cleanup' EXIT` 做收尾。
- **严格模式 `set -euo pipefail`**：`-e` 出错退出、`-u` 未定义变量报错、`-o pipefail` 管道任一环失败即失败。
### 10.2 面试问答
**Q1：`set -e` 有什么坑？**
A：在 `if`/`&&`/`||`/`!` 语境下的失败不会退出；`-e` 也不覆盖子 shell 的传播问题，必须配 `-o pipefail` 才能捕获管道前半段失败。关键命令最好显式判断返回值，不要完全依赖 `-e`。
**Q2：`rm -rf $DIR/*` 有什么风险？**
A：`DIR` 为空或未定义时变成 `rm -rf /*`。防御：`set -u`、`[[ -n "$DIR" ]] || exit 1`、写 `rm -rf "${DIR:?DIR is required}"/*`、能用绝对路径就不用变量。
**Q3：脚本如何做并发与限速？**
A：并发用 `xargs -P` 或 `for ... & wait`（配合 PID 限流）；限速用 `rsync --bwlimit`、`pv -L`、`nice/ionice`；并发写同一文件要加 `flock` 或用独立输出文件再合并。
### 10.3 高频追问与陷阱
- `while read line` 不加 `-r` 会吞反斜杠、不加 `IFS=` 会丢行首尾空白；`$?` 只反映上一条命令，管道要看 `PIPESTATUS`；shebang 用 `#!/usr/bin/env bash` 否则 `[[ ]]` 在 dash 下报错；上线前用 ShellCheck 静态检查。
### 10.4 示例
```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

# 1) 日志清理：压缩 3 天前，删除 7 天前的压缩文件
LOG_DIR=/var/log/app
find "$LOG_DIR" -name '*.log' -mtime +3 -not -name '*.gz' -exec gzip {} +
find "$LOG_DIR" -name '*.gz'  -mtime +7 -delete
echo "[$(date '+%F %T')] cleanup done, size: $(du -sh "$LOG_DIR" | cut -f1)"

# 2) 服务健康检查（失败自动重启）
code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 3 http://127.0.0.1:8080/actuator/health || true)
[[ "$code" == "200" ]] || { echo "[$(date '+%F %T')] health=$code, restart" >&2; systemctl restart myapp; }

# 3) 批量部署（rsync + 并行重启）
printf '%s\n' 10.0.0.8 10.0.0.9 | xargs -n1 -P4 -I{} bash -c '
  set -e; rsync -az --delete /tmp/dist/ deploy@{}:/opt/app/
  ssh -o BatchMode=yes deploy@{} "systemctl restart myapp"; echo "{} deployed"'

# 4) 磁盘水位告警
df -hP | awk 'NR>1 && $5+0 >= 85 {print "DISK WARN: " $6 " " $5}'
```
### 10.5 一句话速记
`set -euo pipefail + 引号包变量 + 绝对路径 + trap 清理；关键命令显式判错，别只靠 -e。`

## 11. 线上应急手册
### 11.1 核心原理
应急总原则：**先止血（保业务）→ 保留现场（证据）→ 定位根因 → 复盘改进**。止血优先级：限流/降级 > 扩容/重启 > 回滚 > 摘流量；任何操作前先记时间与现象。
### 11.2 面试问答
**Q1：磁盘满怎么处置？**
A：`df -hT` 找满的分区、`df -i` 排除 inode 问题 → `du -sh /var/* | sort -rh | head` 逐层下钻（优先 `/var/log`）→ 确认可清理后**用 `truncate -s 0 file` 或 `> file` 而不是 `rm`**（删掉正在写的文件句柄不释放空间）→ `lsof +L1` 处理"已删除但仍被占用"的文件 → 长期方案是 logrotate + 日志级别调整 + 磁盘告警。边界：`/` 满会导致 SSH 无法登录，需预留应急通道。
**Q2：CPU 100% 怎么处置？**
A：`uptime`+`top` 定位进程 → `top -Hp` 找热点线程 + `jstack` 定位栈（**先取证再重启**）→ 常见根因：死循环与无边界重试、正则回溯、Full GC 频繁、线程数爆炸、同步日志刷屏 → 止血：摘该实例流量、限流、重启；GC 问题改参数或加内存。
**Q3：内存 OOM 怎么处置？**
A：先区分 **JVM 堆 OOM**（有异常栈与堆转储）与**容器 OOM-Kill**（进程直接消失、`dmesg` 有 Killed process）；堆 OOM 用 MAT 分析大对象（大查询结果、无界缓存、静态集合与 ThreadLocal 泄漏）；容器 OOM-Kill 要核对 cgroup 限额与 RSS（堆 + 堆外 + 线程栈 + Metaspace）；止血为扩容、限流、降级。**没有堆转储就先别重启**，补上 `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump` 后再重启。
**Q4：句柄泄漏（Too many open files）怎么处置？**
A：`ulimit -n` 看上限、`ls /proc/<pid>/fd | wc -l` 看用量 → `lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -rn` 看句柄类型分布 → 常见根因：HTTP 客户端/DB 连接未关闭、流未 close、每次请求新建客户端、连接未释放 → 止血可临时 `prlimit` 调大或重启，根因必须改代码（try-with-resources + 统一连接池）。
**Q5：端口被占用怎么处置？**
A：`ss -lntp | grep <port>` 或 `lsof -i:<port>` 找 PID → `ps -fp <pid>` 判断是不是自家残留进程 → 用 `kill -15` 优雅退出（等 5~10 秒再考虑 `-9`）→ 被他人占用则改端口或停对方。边界：`Address already in use` 有时是 TIME_WAIT，需保证 `SO_REUSEADDR`（Tomcat/Netty 默认开启）。
**Q6：服务无响应怎么处置？**
A：确认进程与端口（`ss -lntp`）→ `ss -s`/`ss -tanp` 看连接堆积与状态（大量 CLOSE_WAIT 是代码未关连接、大量 SYN-SENT 是下游不可达、`Recv-Q` 满说明 backlog 溢出）→ `jstack` 看线程卡在锁/连接池/下游调用 → `curl 127.0.0.1:port/health` 区分应用层与网络层 → 止血：重启、扩容、限流、切流量；池被打满时先加池再查下游为何变慢。
### 11.3 高频追问与陷阱
- "先重启再排查"是应急大忌（会抹掉堆栈、堆转储与连接状态等唯一证据）；应急不要同时做多个变更（改配置 + 扩容 + 回滚），否则无法归因；事后必须写复盘（现象、时间线、根因、止血动作、改进项）。
### 11.4 示例
```bash
TS=$(date +%F-%H%M); D=/tmp/diag-$TS; mkdir -p $D
{ date; uptime; free -h; df -hT; ss -s; top -b -n 1 | head -20; } > $D/summary.txt
ps -eo pid,ppid,stat,pcpu,pmem,etime,cmd --sort=-pcpu | head -30 > $D/ps.txt
jstack <pid> > $D/jstack.txt 2>&1; jcmd <pid> GC.heap_info > $D/gc.txt 2>&1
iostat -xdm 1 3 > $D/iostat.txt 2>&1
tar -czf /tmp/diag-$TS.tgz -C /tmp diag-$TS && echo "evidence: /tmp/diag-$TS.tgz"
```
### 11.5 一句话速记
`止血 → 取证 → 根因 → 复盘；重启前先抓 jstack/GC/连接状态，日志文件用 truncate 不要 rm。`

## 自测题（含答案要点）
1. `df` 有空闲但写文件报 No space left 的两个原因？inode 耗尽（`df -i`）、删除但被进程占用的文件（`lsof +L1`）；目录的 `x` 权限表示可进入，无 `x` 则无法访问内部文件即使有 `r`。
2. 硬链接与软链接的三个区别？硬链接共享 inode、不能跨文件系统、不能指向目录；软链接反之且目标删除后失效。
3. `chmod 1777 /tmp` 中 1 的作用？Sticky Bit，目录内文件仅属主可删除。
4. `uniq -c` 前为什么必须 `sort`？uniq 只合并相邻重复行。
5. `tail -f` 与 `-F` 的选择依据？日志轮转场景必须 `-F`（按名字重开文件）；统计字符串出现次数用 `grep -o 'x' f | wc -l`（`-c` 只统计匹配行数）。
6. CPU 100% 的定位四步？top → top -Hp → tid 转 16 进制 → jstack 匹配 nid。
7. `us`/`sy`/`wa` 高的分别指向什么？用户态计算 / 系统调用与上下文切换 / IO 等待；负载高但 CPU 空闲则是大量 D 状态（不可中断睡眠）。
8. `free` 的 buff/cache 高是否需要处理？不需要，看 `available`；Page Cache 会被自动回收。
9. Java 进程 RSS 远大于堆的可能来源？直接内存、Metaspace、线程栈、代码缓存、JVM 自身。
10. `iostat` 判断磁盘饱和看哪几个字段？`%util`、`await`、`aqu-sz`。
11. 重传率如何计算与判断？`TcpRetransSegs/TcpOutSegs`，>1% 或持续增长说明链路质量差。
12. CLOSE-WAIT 多与 TIME-WAIT 多分别说明什么？前者是应用未关闭连接（代码问题），后者是短连接/主动关闭过多。
13. `kill -15` 与 `kill -9` 的本质差别？15 可被捕获并执行清理（shutdown hook），9 由内核强杀无清理机会。
14. cron 任务不执行的三大常见原因？PATH/环境变量缺失、工作目录不对、脚本无执行权限或输出未重定向；防重入用 `flock -n` 或分布式锁。
15. `rsync` 源路径结尾斜杠的差异？`src/` 同步目录内容，`src` 同步目录本身。
16. `ssh -L` 与 `-R` 的方向？`-L` 本地端口转发到远端可达地址；`-R` 远端端口转回本地（内网穿透）。
17. `set -euo pipefail` 各自作用？出错即退、未定义变量报错、管道任一环失败即失败。
18. 磁盘满时为什么不能 `rm` 正在写的日志？句柄未释放空间不回收，应 `truncate -s 0` 或 `> file`。
19. 内存 OOM 时重启前必须做什么？保留现场：开 `HeapDumpOnOutOfMemoryError`、抓 `jstack`/GC 信息/`dmesg`。
20. Too many open files 的定位命令？`ulimit -n`、`ls /proc/<pid>/fd | wc -l`、`lsof -p <pid>` 按类型统计。

## 背诵卡片
1. `/etc` 放配置、`/var` 放数据、`/proc` 看进程、`/sys/fs/cgroup` 看容器限额。
2. `df -i` 查 inode，`lsof +L1` 查已删除但仍被占用的文件。
3. 目录的 `x` 表示可进入；删除文件看目录权限而非文件权限。
4. Sticky Bit（`chmod +t`）用于 `/tmp` 防止删除他人文件。
5. 硬链接共享 inode、不跨文件系统、不能指向目录；软链接反之。
6. `umask 022` → 文件 644、目录 755。
7. `tail -F` 才支持日志轮转后的跟随。
8. `grep -A/-B/-C n` 看上下文，`-c` 统计行数，`-o | wc -l` 统计次数。
9. `awk` 取列与聚合，`sed -i.bak` 替换并备份。
10. `sort -rn | head` 是统计结果的标配收尾；`uniq -c` 必须先 `sort`。
11. `find` 实时精确但慢，`locate` 快但依赖索引。
12. `ss -lntp` 看监听、`ss -tanp state time-wait` 看状态、`ss -s` 看总览。
13. `lsof -i:8080` 查端口占用，`lsof -p <pid>` 查某进程句柄。
14. `top -Hp <pid>` 看线程；tid 转 16 进制后在 `jstack` 里搜 `nid=0x`。
15. `pidstat -u -r -d -w` 一次拿到 CPU/内存/IO/上下文切换数据。
16. `vmstat` 的 `r` 是运行队列，`si/so` 是换页，`cs` 是上下文切换。
17. 负载高 CPU 不高 → 查 D 状态与 IO。
18. `buff/cache` 高不是问题，`available` 低才是。
19. Java 进程 RSS ≠ 堆，堆外与线程栈占大头。
20. `iostat -xdm 1` 看 `%util`、`await`、`aqu-sz` 判断磁盘饱和。
21. `iotop -o` 找高写 IO 进程；`pidstat -d` 看进程级 IO。
22. 网络排查顺序：`mtr` → `ss -s` 重传 → `ss -tanp` 状态 → `tcpdump`。
23. CLOSE-WAIT 多 = 代码没关连接；TIME-WAIT 多 = 短连接多。
24. `kill -15` 优雅（触发 shutdown hook），`kill -9` 强杀无清理；`kill -0` 只判断存在。
25. 生产用 systemd 托管服务，别用 `nohup &`。
26. crontab 五段：分时日月周；PATH 极简必须用绝对路径，`%` 要转义。
27. `flock -n` 防单机重入，多实例必须用分布式锁。
28. `rsync -avzP --partial` 增量与续传；`--dry-run` 先演练。
29. `ssh -L` 本地转发、`-R` 反向穿透、`-D` SOCKS 代理、`-J` 跳板。
30. 登录 shell 读 profile，非交互脚本与 cron 都不读，必须显式 `export`。
31. `set -euo pipefail` 是运维脚本标配；`${VAR:?}` 防变量为空。
32. 大文件清理用 `truncate -s 0` 或 `> file`，不要 `rm`。
33. 应急顺序：止血 → 取证 → 定位根因 → 复盘改进。
34. 重启前必须抓 `jstack`、GC 信息、连接状态，否则无从复盘。
