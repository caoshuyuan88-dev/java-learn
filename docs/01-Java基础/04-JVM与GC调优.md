# JVM 与 GC 调优

> **定位**：把内存结构、对象生命周期、GC 算法与收集器、类加载、排查工具串成"看得懂日志、定位得了故障、给得出参数"的完整链路。
> **重要度**：★★★★★（高级岗与调优排查题的必考区，区分度最高）
> **建议复习时长**：10 小时（收集器 3 小时、工具实操 2 小时、参数模板 1 小时）
> **关联文档**：[Java 语言核心与面向对象](./01-Java语言核心与面向对象.md)、[集合框架](./02-集合框架.md)、[并发编程](./03-并发编程.md)、[Java 新特性与语言演进](./05-Java新特性与语言演进.md)、[Linux 与常用命令](../06-计算机基础/04-Linux与常用命令.md)

## 0. 知识地图

## 0.1 先用一句话理解 JVM

JVM 可以先看成一个“运行 Java 程序的工厂”：

```text
.class 字节码
    ↓ 类加载器读入
运行时数据区（栈、堆、元空间）
    ↓ JIT 把热点字节码编译成本地机器码
CPU 执行
    ↓ 堆空间不够
GC 找出没人使用的对象并回收
```

所以学习 JVM 不要一上来背参数，可以先问四个问题：

1. **数据放在哪里？** 是栈、堆、元空间，还是堆外内存？
2. **对象怎么来的？** 是在哪里分配、什么时候晋升？
3. **谁可以被回收？** 如何判断对象还活着？
4. **出问题怎么查？** 看日志、线程、堆转储，还是原生内存？

后面的内容都围绕这四个问题展开。

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| 运行时数据区与元空间 | ★★★★★ | 八股原理 / 版本差异 |
| 对象创建、TLAB、逃逸分析 | ★★★★☆ | 八股原理 / 源码追问 |
| 对象分配与晋升规则 | ★★★★★ | 八股原理 / 调优排查 |
| GC 算法、垃圾判定与四种引用 | ★★★★★ | 八股原理 / 场景题 |
| 收集器演进（CMS/G1/ZGC） | ★★★★★ | 源码追问 / 版本差异 |
| 三色标记、记忆集与卡表 | ★★★★☆ | 源码追问 |
| JVM 参数与 GC 日志解析 | ★★★★★ | 调优排查 |
| 类加载机制与双亲委派 | ★★★★★ | 八股原理 / 场景题 |
| 字节码基础 | ★★★☆☆ | 源码追问 |
| 排查工具（jstat/jmap/Arthas/MAT） | ★★★★★ | 调优排查 |
| OOM、CPU 100% 与内存泄漏排查 | ★★★★★ | 调优排查 / 场景题 |
| 调优参数模板与 JDK 版本变化 | ★★★★☆ | 调优排查 / 版本差异 |

## 1. 运行时数据区与元空间

### 1.1 核心原理
先把内存想成一栋办公楼：

```text
每个线程自己的办公桌：程序计数器、虚拟机栈、本地方法栈
所有线程共享的仓库：堆、元空间
楼外租用的仓库：直接内存
```

最容易混淆的一点是：**变量放在哪里，要看变量本身是什么**。局部变量表里保存的是局部变量和引用；引用可能在栈上，但引用指向的对象通常在堆上。

```java
void demo() {
    User user = new User("Tom");
    // user：当前线程栈帧中的引用
    // new User(...)：通常分配在堆中的对象
}
```

方法返回后，栈帧会消失；如果堆对象没有任何 GC Roots 再指向它，GC 才会在之后回收它。**栈帧消失不等于对象立刻回收**。

| 区域 | 线程私有 | 内容 | 参数 | 溢出异常 |
| --- | --- | --- | --- | --- |
| 程序计数器 | 是 | 当前字节码行号（native 为 undefined） | — | 无 |
| 虚拟机栈 | 是 | 栈帧：局部变量表、操作数栈、动态链接、返回地址 | `-Xss` | `StackOverflowError` |
| 本地方法栈 | 是 | native 方法栈帧 | `-Xss` | 同左 |
| 堆 | 否 | 对象实例、数组 | `-Xms/-Xmx` | `Java heap space` |
| 方法区（元空间） | 否 | 类元数据、运行时常量池、JIT 代码缓存 | `-XX:MaxMetaspaceSize` | `Metaspace` |
| 直接内存 | — | `DirectByteBuffer`（NIO） | `-XX:MaxDirectMemorySize` | `Direct buffer memory` |

注意：静态变量与字符串常量池自 **JDK 7 起在堆中**，不在方法区。

**JDK 8 用元空间取代永久代的原因**：① 永久代大小难预估（反射/CGLIB/动态代理生成大量类，`MaxPermSize` 小了 OOM、大了浪费）；② 元空间用本地内存、默认只受物理内存限制，且类加载器不可达时可整体释放并把内存归还 OS；③ 与 HotSpot 其他实现（JRockit）统一，让 GC 设计与类元数据管理解耦。

| 对比 | 永久代（≤JDK 7） | 元空间（JDK 8+） |
| --- | --- | --- |
| 位置 | JVM 堆内存，受 GC 管理 | 本地内存（native memory） |
| 大小 | `-XX:MaxPermSize` 固定 | `-XX:MaxMetaspaceSize`，默认无限 |
| 溢出 | `PermGen space` | `Metaspace` |
| 回收 | Full GC 时回收 | 类加载器不可达时整体释放 |

`-XX:MetaspaceSize` 不是初始大小，而是**首次触发元空间 GC 的阈值**（默认约 21MB）。

### 1.2 面试问答
**Q1：栈与堆的区别？** A：栈线程私有、随线程生命周期、存局部变量与引用、分配回收快（移动栈指针）；堆线程共享、存对象实例、由 GC 回收。
**Q2：`-Xss` 调大的影响？** A：支持更深递归，但每线程虚拟内存更大，总线程数受 `(物理内存-堆)/Xss` 限制。
**Q3：方法区与元空间是一个东西吗？** A：方法区是 JVM 规范中的抽象概念，永久代与元空间是 HotSpot 的具体实现。
**Q4：直接内存溢出怎么排查？** A：`-XX:MaxDirectMemorySize` 限制；堆 dump 看不到直接内存，用 `-XX:NativeMemoryTracking=detail` + `jcmd <pid> VM.native_memory detail`，常见于 Netty `ByteBuf` 未 release。

### 1.3 高频追问与陷阱
- 陷阱：`-XX:MetaspaceSize` 设小会导致频繁 Full GC，生产常设 256MB。
- 陷阱：`PermSize`/`MaxPermSize` 在 JDK 8 起被忽略（不报错但无效）。
- 错误回答："静态变量在元空间"（JDK 7 起在堆）。

### 1.5 一句话速记
- 私有：PC、虚拟机栈、本地方法栈；共享：堆、元空间、直接内存。
- 元空间：本地内存、默认无上限、可归还 OS，溢出报 `Metaspace`。
- 记忆法：**栈放“我正在做什么”，堆放“我创建了什么”，元空间放“有哪些类”，直接内存放“绕过堆的大块数据”**。

## 2. 对象创建、TLAB 与逃逸分析

### 2.1 核心原理
执行 `new User()` 可以先按这条流水线理解：

```text
检查类是否已加载
    ↓
堆中申请空间
    ↓
字段先获得默认值（0、false、null）
    ↓
执行实例字段初始化和构造器
    ↓
返回对象引用
```

多个线程同时创建对象时，不能都去修改同一个“堆指针”。因此 JVM 通常先给每个线程一小块私有区域：

```text
Eden
┌──────────────┬──────────────┬──────────────┐
│ 线程 A TLAB  │ 线程 B TLAB  │ 线程 C TLAB  │
└──────────────┴──────────────┴──────────────┘
```

线程在自己的 TLAB 里移动指针即可，不需要每次都 CAS。TLAB 用完了，再申请新的 TLAB；对象太大放不进去时，才走共享分配路径。

逃逸分析可以理解为：**JIT 判断这个对象会不会被当前方法/线程以外的代码看到**。

```java
int sum() {
    Point p = new Point(1, 2);
    return p.x + p.y;
}
```

如果 `p` 没有返回、没有存入共享字段、没有传给未知方法，JIT 可能直接把 `p.x`、`p.y` 当作两个局部值，连对象都不创建。这就是标量替换；不要把“栈上分配”理解成所有对象都会真的被放到线程栈上。

分配方式：**指针碰撞**（内存整齐，如 Serial/ParNew/G1 的 Region 内）或**空闲列表**（碎片化，如 CMS）。并发安全两种方案：**TLAB**（线程在 Eden 的私有缓冲区，默认约 Eden 的 1%，可 `-XX:TLABSize` 调整，`-XX:+UseTLAB` 默认开启）或 CAS 重试。

**逃逸分析**（`-XX:+DoEscapeAnalysis`，JDK 6u23 起默认开启）：不逃逸的对象可被
- **标量替换**（`-XX:+EliminateAllocations`）：字段拆成局部变量，对象根本不创建；
- **栈上分配**：分配在栈帧上，方法返回即回收，无 GC 压力；
- **锁消除**：不逃逸对象上的 synchronized 被移除。

```java
public int distance() { Point p = new Point(1, 2); return p.x + p.y; }  // Point 可能被完全优化掉
```
只在分层编译的 C2 阶段对热点代码生效；对象只要进入"可能逃逸"的路径（如传给未知方法）就会被保守放弃。

对象布局：对象头（Mark Word 8B + Klass Pointer 4B/8B）+ 实例数据（按宽度重排）+ 对齐填充（8 字节对齐）。

### 2.2 面试问答
**Q1：TLAB 太小会怎样？** A：频繁申请新 TLAB，退化为 CAS 同步分配；浪费容忍度由 `-XX:TLABWasteTargetPercent` 控制。
**Q2：怎么观察逃逸分析生效？** A：对比 `-XX:-DoEscapeAnalysis` 前后的 GC 频率与分配速率，或用 JMH `-prof gc` 看 `gc.alloc.rate`。
**Q3：逃逸分析为什么不是万能的？** A：分析有编译代价且只在 C2 热点代码生效；对象被可能逃逸的路径引用就保守放弃；字段过多时标量替换收益不足。
**Q4：TLAB 用完了对象去哪？** A：无法放入 TLAB 的大对象直接在 Eden 分配（或按大对象规则进老年代）。

### 2.3 高频追问与陷阱
- 陷阱：`-XX:+DoEscapeAnalysis` 默认开启，但不能保证所有不逃逸对象都被优化。
- 陷阱：循环里 `new` 并不等于必然产生 GC 压力。
- 错误回答："Java 对象一定分配在堆上"。

### 2.5 一句话速记
- 分配：指针碰撞/空闲列表；并发安全靠 TLAB（默认开启，约 Eden 1%）。
- 逃逸分析 → 标量替换、栈上分配、锁消除，仅 C2 热点代码生效。
- 记忆法：**先问对象能不能被别人看到；看不到，就可能不分配、不上锁、不进堆**。

## 3. 对象分配与晋升

### 3.1 核心原理
新生代可以想成“短期周转区”，老年代是“长期仓库”：

```text
新对象
  ↓
Eden：大多数对象先来这里
  ↓ Minor GC
Survivor：暂时活着，年龄 +1
  ↓ 多次存活或空间不够
老年代：长期存活对象
```

一次 Minor GC 的直观过程：

```text
Eden 中的对象逐个判断
├─ 已无引用：丢弃
├─ 还活着且 Survivor 放得下：复制到 Survivor，年龄 +1
└─ Survivor 放不下或达到晋升条件：进入老年代
```

`8:1:1` 不是说每次只能使用 10% 的新生代，而是 Eden、两个 Survivor 的空间比例。两个 Survivor 轮流使用，始终有一块作为“空白接收区”。

默认比例：Eden : S0 : S1 = **8 : 1 : 1**（`-XX:SurvivorRatio=8`），新生代 : 老年代 = **1 : 2**（`-XX:NewRatio=2`），即新生代占堆 1/3。

规则：
1. 对象优先在 Eden 分配，Eden 满触发 Minor GC，存活对象复制到 Survivor，年龄 +1。
2. 年龄达 `-XX:MaxTenuringThreshold`（默认 15；CMS 默认 6）晋升老年代。
3. **动态年龄判定**：Survivor 中相同年龄对象大小之和 > Survivor 空间一半时，年龄 ≥ 该年龄的对象直接晋升（"过早晋升"导致老年代膨胀的主因）。
4. **大对象直接进老年代**：`-XX:PretenureSizeThreshold`（仅 Serial/ParNew 生效，G1 无此参数）。
5. **空间分配担保**：Minor GC 前检查老年代最大连续可用空间是否大于新生代所有对象总大小；大于则安全 Minor GC；小于时按"历次晋升平均大小"判断，允许则冒险 Minor GC（JDK 6u24 后 `HandlePromotionFailure` 参数废弃，JDK 7+ 默认允许失败），不允许则改为 Full GC；Minor GC 后 Survivor 放不下的存活对象直接进老年代。

### 3.2 面试问答
**Q1：为什么新生代用复制、老年代用标记整理/清除？** A：新生代对象朝生夕死（约 98% 快速回收），复制成本低且无碎片（两块 Survivor 保证 90% 新生代可用）；老年代存活率高，复制需 1:1 预留空间不可行，故用标记清除（CMS，有碎片）或标记整理（Serial Old/Parallel Old，无碎片但有移动成本）。
**Q2：为什么需要两个 Survivor？** A：Minor GC 后存活对象要有地方放，同时保证 Eden 可整体清空重置；单 Survivor 无法区分"已存活"与"新分配"。
**Q3：对象一定在 Eden 分配吗？** A：不一定，大对象、TLAB 内分配、逃逸分析后的栈上分配、动态年龄判定提前晋升都会改变位置。
**Q4：晋升失败（Promotion Failed）是什么？** A：Minor GC 时 Survivor 与老年代都放不下存活对象，退化触发 Full GC（CMS 下表现为最长停顿）。

### 3.3 高频追问与陷阱
- 陷阱：`-XX:MaxTenuringThreshold=1` 会让存活对象几乎直接进老年代，极易 Full GC。
- 陷阱：G1 不认 `PretenureSizeThreshold`，大对象由 Humongous（大于 Region 一半）规则处理。
- 错误回答："年龄到 15 一定晋升"（动态年龄判定会提前）。

### 3.5 一句话速记
- Eden:S0:S1 = 8:1:1；默认晋升年龄 15；动态年龄判定看同年龄总和是否超 Survivor 一半。
- 大对象直接进老年代；Minor GC 前先做空间分配担保检查。
- 记忆法：**Eden 出生，Survivor 实习，老年代转正；空间不够可以提前转正**。

## 4. GC 算法、垃圾判定与四种引用

### 4.1 核心原理
GC 的本质不是“把所有对象都扫描一遍然后删除”，而是回答：

> 从程序仍然能够到达的入口出发，这个对象还能不能走到？

```text
GC Roots
   ↓
对象 A → 对象 B → 对象 C

对象 X → 对象 Y
（没有任何 Root 能到达，整条链都可能回收）
```

这就是可达性分析。对象之间即使互相引用，只要没有 GC Roots 能到达，仍然可以回收；这也是它比简单引用计数更可靠的原因。

三种算法可以用“清理房间”记：

```text
标记-清除：标出要保留的，清掉其他的 → 快，但地上留下碎片
标记-复制：把要保留的搬到另一边 → 整齐，但需要备用空间
标记-整理：保留对象向一边搬，边界外全部清掉 → 不碎片，但搬家停顿大
```

| 算法 | 过程 | 缺点 | 使用者 |
| --- | --- | --- | --- |
| 标记-清除 | 标记存活 → 清除未标记 | 内存碎片、效率随对象数下降 | CMS |
| 标记-复制 | 存活对象复制到另一块 | 需预留空间、复制成本随存活率上升 | Serial/ParNew/PS（新生代） |
| 标记-整理 | 存活对象向一端移动后清理边界外 | 移动对象需更新引用（STW 长） | Serial Old/Parallel Old |

**分代收集理论**：弱分代假说（多数对象朝生夕死）+ 强分代假说（熬过越多次 GC 越难死）+ 跨代引用假说（跨代引用极少）。第三点催生**记忆集与卡表**：只需记录"老年代指向新生代的引用"，Minor GC 时把这些老对象加入 GC Roots，避免扫描整个老年代。

**可达性分析**（非引用计数，引用计数无法解决循环引用）的 GC Roots：① 虚拟机栈局部变量表引用的对象；② 方法区静态属性引用的对象；③ 方法区常量引用的对象；④ 本地方法栈 JNI 引用；⑤ JVM 内部引用（基本类型 Class、常驻异常对象、系统类加载器）；⑥ 被同步持有的锁对象与 JMXBean。

| 引用 | 回收时机 | 用途 |
| --- | --- | --- |
| 强引用 | 永不回收（可达即活） | 普通对象 |
| 软引用 `SoftReference` | **内存不足时**回收 | 内存敏感缓存 |
| 弱引用 `WeakReference` | 下次 GC 必回收 | `ThreadLocalMap` key、`WeakHashMap` |
| 虚引用 `PhantomReference` | 无法访问对象，仅做回收通知 | `DirectByteBuffer` 的 Cleaner |

对象死亡需两次标记：不可达 → 判断是否需执行 `finalize()`（未执行过且被重写）→ 进入 F-Queue → 若在 `finalize` 中重新关联则复活。**`finalize` 已废弃（JDK 9 标记、JDK 18 `forRemoval`），不要使用。**

### 4.2 面试问答
**Q1：为什么不用引用计数？** A：无法处理循环引用（A→B→A 计数永不为 0），且每次赋值都要维护计数。
**Q2：Minor/Major/Full GC 的区别？** A：Minor 只清新生代（频繁但快）；Major 通常指老年代 GC，含义依赖收集器（G1 的 Major 可能含新生代）；Full GC 清整堆 + 方法区，STW 最长。
**Q3：软引用与弱引用怎么选？** A：缓存在内存紧张时才释放用软引用；一旦无强引用就希望立刻回收用弱引用。
**Q4：`System.gc()` 会立即 GC 吗？** A：只是建议，可被忽略；生产建议加 `-XX:+DisableExplicitGC`（注意会加剧直接内存回收不及时，需配 `MaxDirectMemorySize`）。

### 4.3 高频追问与陷阱
- 陷阱：`WeakHashMap` 的 value 若强引用 key 会形成循环强引用导致永不回收。
- 陷阱：大量对象重写 `finalize` 会造成 F-Queue 积压并拖慢 GC。
- 错误回答："软引用是 GC 时立即回收"；"可达性分析只需从栈开始"。

### 4.5 一句话速记
- 三算法：标记清除（碎片）、标记复制（新生代）、标记整理（老年代）。
- GC Roots：栈局部变量、静态属性、常量、JNI、JVM 内部引用、锁对象。
- 强/软/弱/虚：永不 / 内存不足 / 下次 GC / 仅通知；`finalize` 已废弃。
- 记忆法：**Root 能走到的是活的；复制适合活得少，整理适合不能留下碎片**。

## 5. 收集器演进与选型

### 5.1 核心原理
先不要把收集器理解成“谁最好”，它们只是不同的取舍：

```text
吞吐量：单位时间完成更多业务
停顿时间：一次 GC 让业务暂停多久
内存占用：GC 需要多少额外结构和空间

通常不能三者同时最大化。
```

可以用“清洁工”理解：

```text
Serial：一个清洁工，简单，适合小房间
Parallel：多个清洁工一起干活，吞吐高
CMS：业务继续营业，清洁工并发清理，但会碎片化
G1：把大房间分成很多小格子，优先清理垃圾最多的格子
ZGC/Shenandoah：业务几乎不停，清洁和搬运尽量并发
```

| 收集器 | 分代 | 算法 | 停顿 | 启用参数 | 适用 |
| --- | --- | --- | --- | --- | --- |
| Serial / Serial Old | 新/老 | 复制 / 标记整理 | 单线程 STW | `-XX:+UseSerialGC` | 小堆 |
| ParNew | 新生代 | 复制 | 多线程 STW | 与 CMS 搭配（JDK 9 起该组合废弃） | — |
| Parallel Scavenge/Old | 新/老 | 复制 / 标记整理 | 多线程 STW | `-XX:+UseParallelGC` | 吞吐优先 |
| **CMS** | 老年代 | 标记清除 | 并发，低延迟 | `-XX:+UseConcMarkSweepGC`（**JDK 14 移除**） | 中小堆低延迟 |
| **G1** | 整堆 Region | 复制 + 标记整理 | 可预测 | `-XX:+UseG1GC`（**JDK 9 起默认**） | 4GB~数十 GB |
| **ZGC** | 整堆 | 并发标记 + 并发转移 | <10ms（JDK 16 起 <1ms 目标） | `-XX:+UseZGC`（JDK 15 生产可用） | 超大堆 |
| Shenandoah | 整堆 | 并发标记 + 并发转移 | <10ms | `-XX:+UseShenandoahGC` | 大堆低延迟 |
| Epsilon | — | 不回收 | — | `-XX:+UseEpsilonGC`（JDK 11+） | 压测、短命进程 |

**CMS 四阶段**：初始标记（STW，只标 GC Roots 直接关联对象）→ 并发标记（最耗时，与用户线程并发）→ 重新标记（STW，修正并发期间的变动，可用 `-XX:+CMSScavengeBeforeRemark` 优化）→ 并发清除。缺点：① CPU 敏感（回收线程默认 `(核数+3)/4`）；② 浮动垃圾（需预留空间，`-XX:CMSInitiatingOccupancyFraction` 默认 92%）；③ 内存碎片（可配 `UseCMSCompactAtFullCollection`）；④ **Concurrent Mode Failure** —— 并发清除期间老年代被填满，退化为单线程 Serial Old Full GC，停顿可达秒级。

**G1 核心设计**：① 堆划分为约 2048 个等大 Region（`-XX:G1HeapRegionSize` 1~32MB），逻辑分代、物理不连续；② **Humongous** 对象（大于 Region 一半）占用连续 Region，只在 Cleanup/Full GC 时回收，短命大对象易触发 Full GC；③ **RSet（Remembered Set）** 记录"谁指向我"，配合卡表处理跨 Region 引用；④ **SATB** 保证并发标记不漏标；⑤ **停顿预测模型** 依据历史数据按"收益/成本"排序挑选 CSet，实现 Garbage First；⑥ Mixed GC 回收整个新生代 + 部分老年代，由 `-XX:InitiatingHeapOccupancyPercent`（IHOP，默认 45%，JDK 9+ 支持自适应）触发。G1 也有 Full GC 与转移失败（Evacuation Failure）。

**ZGC / Shenandoah**：ZGC（JDK 11 实验、**15 生产可用**、**21 分代**）用**着色指针**把标记位存在 64 位指针高位（无需在对象头存标记），**读屏障**在读取时检查并修正（并发转移自愈），几乎全阶段并发，停顿与堆大小无关；不支持压缩指针。Shenandoah 用 **Brooks 转发指针**（对象头多一个指针）实现并发转移。

#### 收集器怎么选：先看目标，不背堆大小口诀

```text
只要高吞吐、延迟要求普通 → Parallel
通用服务、希望停顿可控 → G1
超大堆、严格低延迟 → ZGC / Shenandoah
```

选型顺序建议是：

1. 先看接口 SLA，例如 P99 能否容忍 100ms 停顿；
2. 再看堆大小和对象分配速度；
3. 再看 Full GC 是否能接受；
4. 最后用压测和 GC 日志验证，而不是只按“4GB 用什么、16GB 用什么”机械选择。

`MaxGCPauseMillis` 是 G1 的软目标，不是“保证每次都不超过这个数”。目标设得过小，G1 可能每次只回收很少的 Region，最终吞吐下降。

### 5.2 面试问答
**Q1：怎么选收集器？** A：堆 < 4GB 且无延迟要求用 Parallel；4GB~数十 GB、延迟敏感用 G1（通用首选）；超大堆或 P99 < 10ms 用 ZGC（注意不能开压缩指针）。
**Q2：Concurrent Mode Failure 怎么解？** A：降低 `CMSInitiatingOccupancyFraction`（如 70~75%）、增大老年代与新生代、降低晋升速度、`-XX:+CMSScavengeBeforeRemark`，终极方案换 G1（CMS 已移除）。
**Q3：`MaxGCPauseMillis` 设得过小会怎样？** A：预测模型只挑很小的 CSet，每次回收垃圾少、GC 次数增多，吞吐下降，甚至陷入"回收不完 → Full GC"的恶性循环；它是软目标。
**Q4：分代 ZGC 的改进？** A：ZGC 原先不分代，回收短命对象也要遍历大堆；分代后 Young GC 只扫新生代，显著降低 CPU 与内存开销。

### 5.3 高频追问与陷阱
- 陷阱：G1 在 JDK 8 上需显式开启且 Full GC 是单线程、Fixup 慢，生产建议 JDK 11+。
- 陷阱：CMS 在 JDK 9 标记废弃（JEP 291）、JDK 14 移除（JEP 363），JDK 14+ 加 `UseConcMarkSweepGC` 直接启动失败。
- 错误回答："G1 没有 Full GC"；"ZGC 停顿为零"（仍有极短的根扫描 STW）。

### 5.5 一句话速记
- 吞吐优先 Parallel；通用低延迟 G1（JDK 9 默认）；超大堆超低延迟 ZGC（JDK 15 生产可用、21 分代）。
- CMS 四阶段：初始标记（STW）→ 并发标记 → 重新标记（STW）→ 并发清除；致命伤是 CMF 与碎片。
- 记忆法：**Parallel 拼速度，G1 拼可预测，ZGC/Shenandoah 拼低停顿；没有“永远最好”的收集器**。

## 6. 三色标记、记忆集与卡表

### 6.1 核心原理
并发 GC 最大的难题是：GC 在看对象图时，业务线程还在修改对象引用。

三色可以想成三种工作状态：

```text
白：还没看过
灰：看过自己，但孩子还没看完
黑：自己和孩子都看完了
```

理想流程是：

```text
白对象 → 放入灰色待处理队列 → 扫描孩子 → 变黑
```

如果一个黑对象后来指向白对象，而 GC 又错过了这条变化，就可能漏标。并发收集器通过写屏障记录变化，等并发标记结束后再补处理。

三色标记：白色（未访问）、灰色（自身访问过、成员未访问完）、黑色（自身与成员都访问完）。

**漏标只发生在并发标记阶段**，需同时满足两个条件：① 黑色对象新增了指向白色对象的引用；② 该白色对象到灰色对象的引用被删除。两个解法的差异：
- **增量更新（Incremental Update）**：破坏条件 ① —— 记录"新增引用"（写后屏障），重新标记时以黑色对象为根重扫。**CMS 使用。**
- **原始快照（SATB）**：破坏条件 ② —— 记录"被删除的引用"（写前屏障），按标记开始时的快照继续标记，把本会漏标的白色对象置灰。代价是**浮动垃圾**（本轮应回收的对象保留）。**G1 / Shenandoah 使用。**

**记忆集与卡表**：卡表是字节数组，把堆按 512 字节分为卡页（`-XX:CardTableShift`）；对象引用字段赋值时通过**写屏障**把对应卡页标记为 dirty，GC 时只扫描脏卡对应区域。G1 在卡表之上为每个 Region 维护 RSet（哈希表 + 优化），因此它的写屏障开销高于 CMS。

记忆集可以理解成“反向通讯录”：

```text
Region B 中的对象指向 Region A
GC 只回收 Region A
→ 不必扫描整个堆，只需查 A 的 RSet，找到谁可能指向 A
```

卡表是“哪一小块区域发生过引用写入”的粗粒度目录，RSet 是“哪些地方指向我”的更具体记录。两者有关，但不是同一个东西。

#### 哪些收集器使用三色标记？

先区分两个层次：

```text
三色标记：描述“如何遍历对象图”的通用思想
并发标记：GC 标记时业务线程仍在运行，需要写屏障防漏标
```

因此，使用可达性分析的追踪式收集器，都可以用三色思想理解对象标记；但面试中最常提到三色标记和漏标问题的，是**带并发标记阶段**的收集器：

| 收集器 | 是否可用三色思想理解 | 并发标记时的典型处理 |
| --- | --- | --- |
| Serial / Parallel | 可以，但主要是 STW 标记，业务线程暂停，漏标问题不突出 | 不需要为并发标记设计复杂屏障 |
| CMS | 是 | 增量更新，记录黑对象新增的引用 |
| G1 | 是 | SATB，记录并发标记期间被删除的旧引用 |
| ZGC | 是 | 并发标记 + 读屏障/写屏障，具体实现比传统三色模型更复杂 |
| Shenandoah | 是 | 并发标记 + 写屏障/转发指针 |
| Epsilon | 否 | 不回收，不做完整的垃圾标记 |

可以这样回答面试题：

> 三色标记不是某一个收集器专属的算法，而是追踪式 GC 描述对象可达状态的通用模型。Serial、Parallel 也可以用它理解，但因为主要是 STW，业务线程不会同时修改引用；CMS、G1、ZGC、Shenandoah 的并发标记阶段才重点涉及漏标、写屏障和重新标记。

#### 记忆集、卡表、RSet 到底是什么关系？

可以按“总称 → 粗略目录 → 具体反向记录”理解：

```text
记忆集（Remembered Set）
    ↓ 一种思想/数据结构：记录外部有哪些引用指向当前回收范围
卡表（Card Table）
    ↓ 粗粒度目录：哪些堆卡片最近发生过引用写入
RSet（G1 Remembered Set）
    ↓ G1 为每个 Region 维护的“谁指向我”的记录
```

假设老年代对象 `old` 指向新生代对象 `young`：

```java
class Holder {
    Object reference;
}

Holder old = longLivedHolder();
Object young = new Object();
old.reference = young;
```

如果只回收新生代，GC 不应该扫描整个老年代来寻找 `old → young`。JVM 会在引用赋值时通过写屏障记录这次变化：

```text
old.reference = young
        ↓
写屏障标记相关卡页为 dirty
        ↓ Minor GC
只扫描 dirty 卡页中的候选老对象
        ↓
把 old 作为额外根，找到 young
```

对 G1 来说，堆被分成多个 Region。假设 Region B 中的对象指向 Region A：

```text
Region B ─────引用────> Region A
                         ↑
                    本次准备回收
```

Region A 的 RSet 会记录“Region B 可能有对象指向我”。回收 Region A 时，G1 先查 RSet，而不是扫描整个堆。

三者的区别可以这样记：

| 名称 | 解决什么问题 | 记录粒度 | 形象比喻 |
| --- | --- | --- | --- |
| 记忆集 | 记录回收范围外指向范围内的引用 | 抽象概念/数据结构 | 反向通讯录 |
| 卡表 | 快速标记哪些区域发生过引用写入 | 粗粒度卡片，常见约 512B | 脏区域索引 |
| RSet | G1 记录哪些 Region 可能指向当前 Region | 按 Region 的反向记录 | 每个 Region 的来访名单 |

注意：卡表不是 RSet，RSet 也不等于简单的一张卡表。G1 通常借助卡表发现变化，再维护更具体的 RSet；这也是 G1 需要额外内存和写屏障开销的原因。

### 6.2 面试问答
**Q1：为什么需要记忆集？** A：跨代引用极少（假说），若 Minor GC 扫整个老年代代价太大；记忆集把"老年代指向新生代"的引用记下来，只把这些老对象加入 GC Roots。
**Q2：写屏障的开销体现在哪？** A：每次引用字段赋值都多一次记录操作，G1 的 RSet 维护比 CMS 的卡表更贵，这是 G1 写吞吐略低的原因之一。
**Q3：SATB 为什么产生浮动垃圾？** A：快照把标记开始时活着的对象都视为存活，期间新死亡的对象本轮不回收。
**Q4：增量更新与 SATB 哪个更好？** A：没有绝对优劣 —— 增量更新重扫成本随黑色对象写操作量增长，SATB 有浮动垃圾但停顿更可控，因此 G1/ZGC 选 SATB。
**Q5：三色标记是不是只有 G1 使用？** A：不是。它是追踪式 GC 的通用理解模型；CMS、G1、ZGC、Shenandoah 的并发标记会重点使用相关屏障来避免漏标。
**Q6：记忆集和卡表是不是一回事？** A：不是。记忆集是“记录外部引用”的抽象结构，卡表是把堆划成卡片并标记脏卡的粗粒度实现，G1 还会为 Region 维护更具体的 RSet。

### 6.3 高频追问与陷阱
- 陷阱：漏标必须两个条件同时满足，只说其一会被追问到答不上。
- 陷阱：卡表与 RSet 不是一回事，RSet 是"每个 Region 一份的记录集合"，卡表是全堆共享的字节数组。
- 错误回答："CMS 用 SATB"（CMS 用增量更新）。
- 错误回答："三色标记就是一种只属于 G1 的 GC 算法"（它是追踪式标记的通用模型）。

### 6.5 一句话速记
- 漏标两条件：黑色新增指向白色 + 白色到灰色的引用被删。
- 增量更新记新增（CMS），SATB 记删除（G1），SATB 产生浮动垃圾。
- 记忆法：**白色没看、灰色待看、黑色看完；写屏障负责告诉 GC“业务刚改过引用”**。
- 记忆法：**记忆集是总称，卡表记“哪里变脏”，RSet 记“谁指向我”**。

## 7. JVM 参数与 GC 日志

### 7.1 核心原理
JVM 参数不要按“看到就背”来学，可以按四个问题分类：

```text
堆有多大？       -Xms / -Xmx
线程和类有多大？  -Xss / MaxMetaspaceSize
谁来回收？       UseG1GC / UseParallelGC / UseZGC
出问题怎么留证？  GC 日志 / HeapDump / NMT
```

一个参数是否有效，还要看 JDK 版本和当前收集器。最典型的例子是：`-Xmn` 对 G1 不适合作为固定新生代配置，JDK 9 以后 GC 日志也不能继续照搬 JDK 8 参数。

| 分类 | 参数 | 含义 |
| --- | --- | --- |
| 堆 | `-Xms/-Xmx` | 初始/最大堆（生产设相等，避免动态扩缩抖动） |
| 堆 | `-Xmn` / `-XX:NewRatio` / `-XX:SurvivorRatio` | 新生代大小 / 新老比（默认 2）/ Eden 与单 Survivor 比（默认 8） |
| 栈 | `-Xss` | 单线程栈大小（约 512KB~1MB） |
| 元空间 | `-XX:MetaspaceSize` / `-XX:MaxMetaspaceSize` | 首次 GC 阈值（约 21MB）/ 上限（生产设 256~512MB） |
| GC | `-XX:+UseG1GC` / `-XX:MaxGCPauseMillis=200` / `-XX:G1HeapRegionSize` | 收集器与目标 |
| GC | `-XX:ParallelGCThreads` / `-XX:ConcGCThreads` | 并行 STW 线程数（按核数）/ 并发线程数 |
| GC | `-XX:InitiatingHeapOccupancyPercent=45` / `-XX:MaxTenuringThreshold=15` | G1 并发标记阈值 / 晋升年龄 |
| 诊断 | `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump` | OOM 自动 dump |
| 诊断 | `-XX:+DisableExplicitGC` / `-XX:+AlwaysPreTouch` / `-XX:NativeMemoryTracking=detail` | 忽略 System.gc / 预触碰堆页 / 原生内存追踪 |

**GC 日志参数的版本差异**：JDK 8 用 `-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/path/gc.log`；**JDK 9 起统一日志框架（JEP 158）取代它们，旧参数被移除**，改用
```bash
-Xlog:gc*:file=/data/log/gc.log:time,uptime,level,tags:filecount=10,filesize=100M
```
G1 日志读法示例：`GC(12) Pause Young (Normal) (G1 Evacuation Pause) 512M->128M(2048M) 12.345ms` —— 依次是编号、类型、触发原因、回收前→回收后（堆总量）、停顿毫秒。**关键判断**：有无 `Pause Full`、单次停顿是否超 `MaxGCPauseMillis`、回收后占用是否持续上升（泄漏信号）。

读一行 GC 日志时只做三件事：

```text
1. 这是什么 GC？Young、Mixed 还是 Full？
2. GC 前后降了多少？如果几乎不降，可能有泄漏或晋升压力。
3. 花了多久？频率 × 单次停顿，才是对业务的真实影响。
```

例如：

```text
512M -> 128M，耗时 12ms
```

说明这次释放了约 384M，且停顿不长；如果长期变成 `1900M -> 1890M`，即使单次只停顿几十毫秒，也要警惕堆中对象无法回收。

### 7.2 面试问答
**Q1：容器里怎么设堆？** A：JDK 8u191+/10+ 支持 `UseContainerSupport`（默认开启），用 `-XX:MaxRAMPercentage=70 -XX:InitialRAMPercentage=70` 替代固定值；JDK 8u131~8u190 需手动 `-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap`，否则按宿主机内存算堆被 OOMKilled。
**Q2：怎么判断发生了 Full GC？** A：JDK 8 日志里搜 `Full GC`；JDK 9+ 搜 `Pause Full`；或 `jstat -gc <pid> 1000` 观察 `FGC`/`FGCT` 是否增长。
**Q3：`DisableExplicitGC` 有什么副作用？** A：`System.gc()` 被忽略，`DirectByteBuffer` 的 Cleaner 依赖它触发回收，可能导致直接内存堆积，需配 `MaxDirectMemorySize`。
**Q4：`-Xmn` 与 `NewRatio` 能同时用吗？** A：能但 `-Xmn` 优先；G1 下两者都被忽略（新生代自适应）。

### 7.3 高频追问与陷阱
- 陷阱：JDK 9+ 加 `-Xloggc` 会报 `Unrecognized VM option`，升级 JDK 必须改参数。
- 陷阱：不设 `MaxMetaspaceSize` 时元空间无限增长（类加载器泄漏）会耗尽物理内存被容器 OOMKilled，而堆 dump 完全正常。
- 错误回答："`-Xms` 设小更省内存"（会频繁扩缩容并触发更多 GC）。

### 7.5 一句话速记
- `-Xms = -Xmx`；元空间必须设上限；OOM 自动 dump 必须开。
- JDK 8 用 `PrintGCDetails/Xloggc`，JDK 9+ 用 `-Xlog:gc*:file=...`。
- 容器用 `MaxRAMPercentage`，只有 JDK 8u191+ 才正确识别 cgroup。
- 记忆法：**参数先问大小，再问收集器，最后问怎么留下证据**。

## 8. 类加载机制与双亲委派

### 8.1 核心原理
类加载可以理解成“把一份 class 文件变成 JVM 能使用的 `Class` 对象”：

```text
找到字节流
  ↓
检查是否合法
  ↓
为静态字段准备空间并写零值
  ↓
把符号引用解析成实际引用
  ↓
执行静态初始化代码
```

例如：

```java
class Config {
    static int port = 8080;
    static {
        System.out.println("init");
    }
}
```

准备阶段 `port` 先是 `0`，初始化阶段才执行 `port = 8080` 和静态代码块。编译期常量 `static final int PORT = 8080` 则可能在准备阶段直接得到最终值。

双亲委派可以先记成“先问家长”：

```text
应用类加载器
    ↓ 先交给平台/启动类加载器
父加载器找得到 → 父加载器加载
父加载器找不到 → 子加载器自己尝试
```

这样可以避免应用自己伪造 `java.lang.String`，也让核心类尽量由同一个加载器加载。

#### 一次 `loadClass` 到底先做什么、后做什么？

假设代码执行：

```java
Class<?> type = Class.forName("com.example.User");
```

可以把过程拆成下面 7 步。这里先忽略缓存细节，先记住主流程：

```text
第 1 步：当前类加载器收到“加载 User”的请求
    ↓
第 2 步：先检查自己以前是否已经加载过 User
    ├─ 已加载：直接返回已有的 Class 对象
    └─ 未加载：继续
    ↓
第 3 步：把请求交给父类加载器
    ↓
第 4 步：父类加载器继续重复“查缓存 → 问它的父加载器”
    ↓
第 5 步：最上面的 Bootstrap 先尝试查找
    ├─ 找到：由 Bootstrap 加载并返回
    └─ 找不到：返回“我加载不了”
    ↓
第 6 步：父加载器都找不到，子加载器才自己查 classpath
    ├─ 找到字节码：定义 Class 对象并返回
    └─ 找不到：抛出 ClassNotFoundException
    ↓
第 7 步：如果是 Class.forName，继续触发初始化
```

注意第 3 步和第 6 步，这就是“委派”的核心：

```text
子加载器不是先自己找，而是先问父加载器；
只有父加载器明确找不到，子加载器才自己找。
```

以常见的加载器层次为例：

```text
Bootstrap ClassLoader
        ↑
Platform ClassLoader
        ↑
Application ClassLoader
        ↑
Custom ClassLoader
```

请求从下往上传：

```text
Custom → Application → Platform → Bootstrap
```

查找失败后，再从上往下返回：

```text
Bootstrap 找不到
    ↓
Platform 找不到
    ↓
Application 找不到
    ↓
Custom 自己查找
```

可以把 `ClassLoader.loadClass` 的核心逻辑简化成：

```java
Class<?> loadClass(String name) throws ClassNotFoundException {
    // 1. 先查当前加载器是否已经加载过
    Class<?> loaded = findLoadedClass(name);
    if (loaded != null) {
        return loaded;
    }

    // 2. 优先让父加载器尝试
    if (parent != null) {
        try {
            return parent.loadClass(name);
        } catch (ClassNotFoundException ignored) {
            // 父加载器找不到，当前加载器才继续
        }
    }

    // 3. 父加载器都失败后，当前加载器自己查找字节码
    return findClass(name);
}
```

`findClass` 通常负责“找到字节码并调用 `defineClass` 定义类”，而不是负责父子委派。自定义类加载器一般只需要重写 `findClass`，这样更不容易破坏双亲委派。

#### 为什么要先问父加载器？

假设应用目录里放了一个自定义的 `java.lang.String`：

```text
如果子加载器先自己查找：
应用可能加载自己的 java.lang.String
    ↓
核心类被替换，类型体系和安全边界被破坏
```

使用双亲委派后：

```text
请求 java.lang.String
    ↓
先交给 Bootstrap
    ↓
Bootstrap 加载 JDK 自带 String
    ↓
应用自己的同名类没有机会冒充核心类
```

它主要解决三个问题：

1. **核心类安全**：应用不能轻易替换 `java.lang.*` 等核心类；
2. **避免重复加载**：同一层级优先复用已经加载的 Class；
3. **保证类型一致**：核心类尽量由同一个加载器定义。

#### “加载”和“初始化”不是一回事

双亲委派解决的是“由哪个类加载器找到并定义类”，不等于立即执行静态代码：

```java
ClassLoader loader = Thread.currentThread().getContextClassLoader();
Class<?> type = loader.loadClass("com.example.User");
// 通常只加载，不主动执行 User 的静态初始化代码
```

而：

```java
Class<?> type = Class.forName("com.example.User");
// 默认会继续触发初始化，执行 static 字段赋值和 static 代码块
```

可以记成：

```text
loadClass：先把“类的说明书”读进来
Class.forName：读进来后，默认还要执行静态初始化
```

五阶段：加载 → 验证 → 准备 → 解析 → 初始化（+ 使用 + 卸载）。
| 阶段 | 关键动作 |
| --- | --- |
| 加载 | 通过全限定名获取字节流，生成 `Class` 对象 |
| 验证 | 文件格式、元数据、字节码、符号引用验证 |
| 准备 | 为**静态变量**分配内存并设**零值**（`static int a = 10` 此时为 0），`static final` 常量直接赋真值 |
| 解析 | 符号引用 → 直接引用（可延迟到首次使用，支持动态绑定与 `invokedynamic`） |
| 初始化 | 执行 `<clinit>`（静态赋值 + 静态块，父类先于子类） |

**主动引用（6 种）触发初始化**：new/getstatic/putstatic/invokestatic、反射、初始化子类（先初始化父类）、main 类、`MethodHandle`/`invokedynamic` 解析结果、接口 default 方法的实现类。
**被动引用不触发**：子类访问父类静态字段（只初始化父类）、`new A[10]`、引用编译期常量。

**双亲委派**：类加载请求先委派父加载器，父加载器无法完成才自己加载。加载器层次：Bootstrap（`$JAVA_HOME/lib`，JDK 9 起为 `modules`，C++ 实现）→ Extension/Platform（JDK 9 改名）→ Application（classpath）→ 自定义。好处：核心类唯一（防 `java.lang.String` 被替换）、类一致性与沙箱安全、避免重复加载。

**打破双亲委派的三种场景**：① **SPI（JDBC/JNDI/JAXP）** —— 核心类由 Bootstrap 加载而实现类在 classpath，用**线程上下文类加载器 TCCL** 反向委派；② **Tomcat** —— 每个 WebApp 一个 `WebappClassLoader`，先自己加载（WEB-INF/classes、WEB-INF/lib）再委派父加载器，实现应用隔离与热部署；③ **热部署/OSGi/JRebel/Arthas redefine** —— 用新 `ClassLoader` 加载修改后的类，旧加载器与类的 Class 对象一同被 GC。

**类唯一标识 = ClassLoader + 全限定名**，两个加载器加载同一份字节码得到两个不同的类，互相强转会 `ClassCastException`（`instanceof` 也为 false）。

### 8.2 面试问答
**Q1：`Class.forName` 与 `ClassLoader.loadClass` 的区别？** A：前者默认执行初始化（`initialize = true`），后者只到加载阶段。
**Q2：JDBC 4.0 后为什么不需要 `Class.forName`？** A：`DriverManager` 通过 `ServiceLoader` 读取 `META-INF/services/java.sql.Driver` 自动发现（TCCL 机制）。
**Q3：Tomcat 为什么打破双亲委派？** A：应用隔离（不同应用可用不同版本的同一库）、热部署（丢弃整个 WebApp 加载器实现类卸载）、优先使用应用自带库版本。
**Q4：类什么时候能被卸载？** A：该类的所有实例被回收、`Class` 对象无引用、加载它的 `ClassLoader` 被回收。因此加载器泄漏（Tomcat 热部署、ThreadLocal 持有 WebApp 类、JDBC 驱动/MBean 注册）会导致**元空间 OOM**。
**Q5：JPMS（JDK 9）的影响？** A：Extension 加载器被 Platform 取代；模块用 `requires/exports/opens` 声明；JDK 16 起默认强封装，反射访问内部 API 需 `--add-opens`。
**Q6：为什么自定义类加载器通常重写 `findClass`，而不是直接重写 `loadClass`？** A：`loadClass` 中包含“先查自己、再问父加载器、父失败后自己查找”的委派流程；只重写 `findClass` 可以复用这套流程，直接重写 `loadClass` 容易破坏双亲委派。
**Q7：`loadClass` 找到类后会立即执行静态代码块吗？** A：通常不会。`loadClass` 主要负责加载；`Class.forName` 默认还会触发初始化，执行 `<clinit>`。

### 8.3 高频追问与陷阱
- 陷阱：`static int a = 10` 在准备阶段为 0，初始化阶段才变 10；`static final int A = 10` 在准备阶段就是 10。
- 陷阱：`-Xverify:none` 在 JDK 13 废弃，JDK 14+ 无效。
- 错误回答："双亲委派是强制规定"（是推荐模型，子类可重写 `loadClass`）。

### 8.5 一句话速记
- 五阶段：加载、验证、准备（零值）、解析、初始化（`<clinit>`）。
- 打破双亲委派的场景：SPI（TCCL）、Tomcat（WebApp 优先）、热部署（新加载器）。
- 类唯一标识 = ClassLoader + 全限定名。
- 记忆法：**先找文件，再验身份，再给静态字段留位置，最后执行静态代码；双亲委派先问家长，家长找不到子类再找**。

## 9. 字节码基础

### 9.1 核心原理
字节码可以看成 JVM 的“中间语言”。Java 源码中的语法糖，最终都会变成一串指令：

```text
局部变量表 ⇄ 操作数栈 ⇄ 方法调用
```

例如 `int c = a + b` 大致是：

```text
iload a → iload b → iadd → istore c
```

`javap -c -v -p Demo.class` 的价值不是背每条指令，而是验证“源码看起来简单，编译后到底做了什么”。

字节码结构：魔数 `0xCAFEBABE` + 版本号 + 常量池 + 访问标志 + 字段/方法表 + 属性表；用 `javap -v -p Xxx.class` 查看。

| 指令 | 说明 |
| --- | --- |
| `iload/aload/istore/astore` | 局部变量表 ↔ 操作数栈 |
| `invokestatic` | 静态方法，编译期绑定 |
| `invokespecial` | 构造器、私有方法、`super`，编译期绑定 |
| `invokevirtual` | 实例方法，**动态绑定**（vtable 查表） |
| `invokeinterface` | 接口方法（itable） |
| `invokedynamic` | 动态调用（Lambda、String 拼接、JDK 7 引入） |
| `new/dup/invokespecial <init>` | 对象创建三步 |
| `monitorenter/monitorexit` | synchronized 块；同步方法用 `ACC_SYNCHRONIZED` 标志 |
| `checkcast/instanceof/athrow` | 类型检查、异常抛出与返回 |

### 9.2 面试问答
**Q1：`switch(String)` 怎么实现？** A：编译为 `hashCode()` 分段比较 + `equals()` 精确比较。
**Q2：`try-with-resources` 的字节码形态？** A：`try/finally` + `close()`，异常通过 `addSuppressed` 挂到主异常；`synchronized` 块有两条 `monitorexit`（正常与异常路径），保证锁必然释放。
**Q3：哪些方法调用是静态绑定的？** A：`invokestatic`/`invokespecial`（静态方法、构造器、私有方法、super），以及所有字段访问。
**Q4：`javap -v` 最常用来验证什么？** A：语法糖的真实实现 —— `switch(String)` 的 hashCode 分段比较、Lambda 的 `invokedynamic`、泛型桥接方法、`try-with-resources` 的 finally 结构。

### 9.3 高频追问与陷阱
- 陷阱：`invokestatic`/`invokespecial` 是静态绑定，不能多态；字段访问一律静态绑定。
- 陷阱：`invokedynamic` 首次调用有引导开销，但只发生一次。
- 错误回答："方法调用都走 `invokevirtual`"。

### 9.5 一句话速记
- 静态绑定：`invokestatic`、`invokespecial`；动态绑定：`invokevirtual`、`invokeinterface`、`invokedynamic`。
- `javap -v` 是验证语法糖（switch String、lambda、泛型桥接）的标准手段。
- 记忆法：**先把值压栈，再执行指令；`invokevirtual` 看运行时对象，`invokestatic/invokespecial` 看编译期关系**。

## 10. 排查工具

### 10.1 核心原理
排查工具不是越多越好，而是要和问题对应：

```text
进程还活着吗？       jps
GC 是否异常？         jstat / GC 日志
哪个线程忙或被卡住？  jstack / Arthas thread
哪些对象占内存？      jmap / jcmd / MAT
CPU 到底热点在哪？    async-profiler / JFR
```

建议先低成本观察，再做可能触发停顿的操作：

```text
jstat / dashboard
    ↓
jstack / thread
    ↓
GC 日志、JFR
    ↓
dump、MAT（最后使用，注意 STW 和文件敏感性）
```

| 工具 | 用途 | 常用命令 |
| --- | --- | --- |
| `jps` | 列出 Java 进程 | `jps -l -m -v` |
| `jstat` | GC 与内存统计 | `jstat -gcutil <pid> 1000 10`、`-gc`、`-class` |
| `jmap` | 堆快照/直方图 | `jmap -histo:live <pid> \| head -30`、`-dump:format=b,file=a.hprof <pid>` |
| `jstack` | 线程栈 | `jstack -l <pid>`（含锁信息）、`-F` |
| `jcmd` | 综合（JDK 7+ 推荐） | `VM.flags`、`GC.heap_info`、`Thread.print`、`GC.class_histogram`、`VM.native_memory detail`、`JFR.start`；`jinfo -flags <pid>` 可查看与动态改参数 |
| Arthas | 线上诊断 | `dashboard`、`thread -n 3`/`-b`、`jad`、`watch`、`trace`、`sc -d`、`ognl`、`profiler`、`heapdump` |
| MAT | 堆 dump 分析 | 支配树、Leak Suspects、Histogram、OQL |
| async-profiler | CPU/内存火焰图 | `profiler.sh -d 30 -f out.html <pid>` |
| JFR | 低开销飞行记录 | `-XX:StartFlightRecording=duration=60s,filename=a.jfr` |

`jstat -gcutil` 列含义：`S0/S1/E/O/M` 各区使用率、`CCS` 压缩类空间、`YGC/YGCT`、`FGC/FGCT`、`GCT` 总 GC 时间。**关键指标**：FGC 是否持续增长、GCT 占运行时间比例（>10% 需关注）、老年代回收后是否下降。

**MAT 支配树**：节点 X 支配 Y 表示所有从 GC Roots 到 Y 的路径都经过 X；按 Retained Heap 排序后**顶部分支就是最大内存持有者**，配合 `Path to GC Roots`（排除弱/软引用）即可定位泄漏持有链。

### 10.2 面试问答
**Q1：`jmap -dump` 会 STW 吗？** A：会（安全点执行），大堆可能停顿数秒到数十秒；优先用 `-XX:+HeapDumpOnOutOfMemoryError` 自动 dump，或用 `jcmd GC.heap_dump`。
**Q2：`jmap -histo` 与 `-histo:live` 的区别？** A：带 `live` 会先触发一次 Full GC，生产高峰慎用。
**Q3：Arthas 最常用的几个命令？** A：`dashboard`（总览）、`thread -n 3`（最忙线程）、`thread -b`（阻塞其他线程的锁）、`jad`（反编译确认线上代码版本）、`watch`/`trace`（观察出入参与耗时）、`sc -d`（查类加载器）、`getstatic`/`ognl`（读静态字段）。
**Q4：dump 文件有什么风险？** A：可能包含手机号、密码等敏感数据，传输与存储需脱敏加密；MAT 打开大 dump 建议 `-Xmx` 至少为 dump 的 1.5 倍。

### 10.3 高频追问与陷阱
- 陷阱：`jhat` 已废弃，MAT/JVisualVM 是替代方案。
- 陷阱：CPU 高定位必须先把 TID 转十六进制再 grep `nid=0x...`，否则找不到栈。
- 错误回答："`jstack` 只能看线程状态，不能看锁"（`-l` 会打印锁持有与等待关系，并自动检测死锁）。

### 10.4 代码/配置示例
```bash
top -Hp 12345                             # 找高 CPU 线程 TID
printf '%x\n' 12346                       # 转 16 进制
jstack 12345 | grep -A 30 "nid=0x303a"    # 定位对应栈
```

### 10.5 一句话速记
- `jstat -gcutil` 看 YGC/FGC/GCT；`jmap -histo:live` 会 Full GC；dump 分析用 MAT 支配树。
- Arthas 四件套：`dashboard`、`thread -n/-b`、`jad`、`watch`/`trace`。
- 记忆法：**先看现象，再找线程，再看对象，最后用 dump 证明持有链**。

## 11. OOM、CPU 100% 与内存泄漏排查

### 11.1 核心原理
OOM 和 CPU 高不要直接猜“堆太小”或“代码死循环”，先根据现象分类：

```text
异常日志 → 判断是哪块内存
CPU 飙高 → 判断是业务、GC、锁竞争还是线程过多
堆持续上涨 → 区分正常流量增长和真正泄漏
```

内存泄漏的关键不是“对象很多”，而是：

```text
对象已经没用了
    但仍被某条 GC Roots → 持有链引用
    所以 GC 不敢回收
```

MAT 的 `Path to GC Roots` 就是在回答：“到底是谁还抓着这个对象不放？”

| 异常信息 | 区域 | 常见原因 | 排查手段 |
| --- | --- | --- | --- |
| `Java heap space` | 堆 | 大对象、缓存无上限、内存泄漏 | dump + MAT 支配树与 GC Roots 路径 |
| `GC overhead limit exceeded` | 堆 | GC 花 >98% 时间却回收 <2% | 本质是堆不足或泄漏（可 `-XX:-UseGCOverheadLimit` 看清真实 OOM） |
| `Metaspace` | 元空间 | 动态生成类过多、类加载器泄漏 | `jmap -clstats`、`jcmd GC.class_stats`，看类数量是否单调增长 |
| `Compressed class space` | 压缩类空间 | 1GB 上限被类数量耗尽 | 调大 `-XX:CompressedClassSpaceSize` |
| `Direct buffer memory` | 直接内存 | Netty `ByteBuf` 未 release、`MaxDirectMemorySize` 过小 | NMT `jcmd VM.native_memory detail`，堆 dump 看不到 |
| `unable to create new native thread` | 线程 | 线程数达系统限制、线程池无界 | `jstack` 线程数、`ulimit -u`、容器 `pids.max` |
| `StackOverflowError` | 栈 | 无限递归、`-Xss` 太小、AOP 代理循环 | 栈帧直接给出递归位置 |

**CPU 100% 排查流程**：① `top` 确认 Java 进程；② `top -Hp <pid>` 找高 CPU 线程 TID；③ `printf '%x'` 转十六进制后 `jstack <pid> | grep -A 30 nid=0x...`（或 Arthas `thread -n 3`）；④ 分类判断：业务死循环/热点（栈里是业务方法，常见正则回溯、大循环）、GC 线程忙（`jstat -gcutil` 看频率，内存问题伪装成 CPU 问题）、锁竞争与线程过多（`vmstat 1` 看 `cs`/`sy`）、JIT 编译线程（短时高属正常）；⑤ 用 async-profiler 火焰图确认热点方法。

**内存泄漏排查流程**：① 监控老年代 GC 后占用是否递增、Full GC 频率与 `GCT` 是否上升；② dump（OOM 自动或低峰期手动）；③ MAT 看 Leak Suspects → Dominator Tree 顶部对象 → `Path to GC Roots` 确认持有链；④ 常见泄漏源：静态集合只增不减、ThreadLocal 未 remove、监听器/回调未注销、缓存无淘汰、连接/流未关闭、类加载器泄漏、`subList` 持有大列表；⑤ 修复后观察老年代基线是否稳定。

### 11.2 面试问答
**Q1：怎么区分内存泄漏与内存溢出？** A：溢出是一次性需求超过可用空间（如超大结果集），修正是加内存或改逻辑；泄漏是对象不再使用但被意外持有，堆占用随时间单调上升、Full GC 后仍不下降，必须找到持有链。
**Q2：堆正常但物理内存持续上涨怎么排查？** A：可能是元空间、直接内存、线程栈、JIT 代码缓存、G1 的 RSet（可占堆 10%+）；用 `-XX:NativeMemoryTracking=detail` + `jcmd VM.native_memory summary` 对比，或 `pmap -x <pid>`。
**Q3：容器被 OOMKilled 但 dump 正常是什么原因？** A：堆外内存超限（元空间、直接内存、线程栈、JIT）、JVM 未识别容器限制（JDK 8u191 之前）、`-Xmx` 加其他区域超出容器 limit。
**Q4：堆大小是否合理的判断标准？** A：`jstat -gc` 的 `OU`（老年代使用量）在 GC 后长期低于堆的 40%~60% 说明有余量；长期高于 70% 且 Full GC 频繁则需增大堆或修泄漏。

### 11.3 高频追问与陷阱
- 陷阱：`-Xmx` 等于容器 limit 必然被 OOMKilled，堆应设 limit 的 60%~75%。
- 陷阱：`unable to create new native thread` 有时是线程池无界导致线程暴涨，而非系统限制。
- 错误回答："OOM 一定是堆太小，加 `-Xmx` 就行"。

### 11.5 一句话速记
- 六类 OOM：堆、GC overhead、Metaspace、直接内存、native thread、栈；先看日志关键短语。
- CPU 高先分类：业务死循环、GC、锁竞争、线程过多；用 `top -Hp` + `jstack nid` 或 `thread -n`。
- 泄漏三件套：GC 后老年代递增 → dump → MAT 支配树 + Path to GC Roots。
- 记忆法：**先分类，后取证；先找谁持有，再决定加内存还是改代码**。

## 12. 调优参数模板与版本变化

### 12.1 核心原理
调优不是“试几个参数直到不报错”，而是一次实验：

```text
明确目标
  ↓
记录基线（吞吐、P99、YGC/FGC、GC 后堆占用）
  ↓
压测或复现
  ↓
只改少量参数
  ↓
对比结果，保留有效改动
```

例如看到 Full GC 频繁，不能直接把 `-Xmx` 加大。要先确认：

```text
是活对象真的超过堆容量？
是短时间分配太快？
是大对象/Humongous？
是元空间或显式 System.gc()？
还是对象泄漏？
```

**4C8G（中小业务，G1，JDK 17）**：`-Xms4g -Xmx4g`（limit 的 50%，留足堆外）、`-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m`、`-Xss512k`、`-XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:G1HeapRegionSize=4m`、`-XX:ParallelGCThreads=4 -XX:ConcGCThreads=1`、`-XX:+HeapDumpOnOutOfMemoryError`、`-Xlog:gc*:file=/data/log/gc.log:time,uptime,level,tags:filecount=10,filesize=50M`、`-XX:+DisableExplicitGC`。

**8C16G（核心服务，G1，JDK 17）**：`-Xms10g -Xmx10g`、元空间 256m/512m、`-XX:G1HeapRegionSize=8m`、`-XX:InitiatingHeapOccupancyPercent=45 -XX:+G1UseAdaptiveIHOP`、`-XX:ParallelGCThreads=8 -XX:ConcGCThreads=2`、`-XX:+AlwaysPreTouch`、其余同上。

**16C64G（低延迟，ZGC，JDK 21）**：`-Xms48g -Xmx48g -XX:+UseZGC -XX:+ZGenerational`（ZGC 不支持压缩指针，内存利用率略低）。

**调优方法论**：① 明确目标（吞吐 GCT < 5% / 低延迟 P99 停顿 < 100ms / 内存占用，三者不可兼得）；② 度量现状（GC 日志 + Micrometer）；③ 压测复现；④ 定位瓶颈（分配率过高、堆太小、大对象、毛刺）；⑤ 一次只改 1~2 个参数并复测；⑥ **代码层优化优先**（减少对象分配、修泄漏、缓存加淘汰），收益远大于调参。

**JDK 9-21 的 GC 相关变化**：
| 版本 | 变化 |
| --- | --- |
| JDK 9 | **G1 成为默认**（JEP 248）、CMS 标记废弃（JEP 291）、统一日志框架（JEP 158） |
| JDK 10 | G1 Full GC 并行化（JEP 307） |
| JDK 11 | **ZGC 实验特性**（JEP 333）、Epsilon（JEP 318）、JFR 开源 |
| JDK 12 | Shenandoah 进主线（JEP 189） |
| JDK 13 | ZGC 支持取消未使用内存并归还 OS（JEP 351） |
| JDK 14 | **CMS 移除**（JEP 363）、ZGC 支持 macOS/Windows |
| JDK 15 | **ZGC 生产可用**（JEP 377）、Shenandoah 生产可用（JEP 379）、**偏向锁默认禁用**（JEP 374） |
| JDK 16 | ZGC 并发栈处理，停顿稳定 <1ms（JEP 376）、默认强封装内部 API（JEP 396） |
| JDK 17（LTS） | 全系列收集器可用，G1/ZGC 在生产大规模验证 |
| JDK 21（LTS） | **分代 ZGC**（JEP 439）、虚拟线程（JEP 444）、结构化并发与 Scoped Values 预览 |

### 12.2 面试问答
**Q1：你们线上用什么收集器？为什么？** A：给结构化回答：堆规模 → 延迟要求 → 收集器 → 关键参数 → 观测到的指标（YGC 频率、平均停顿、Full GC 次数）→ 遇到的具体问题与处理（如 Humongous 触发 Full GC，通过调大 `G1HeapRegionSize` 与减少大数组分配解决）。
**Q2：GC 停顿多少算正常？** A：Minor GC 通常 10~50ms（G1 典型 20~50ms），Full GC 数百毫秒至数秒；ZGC 目标 <1ms（JDK 16+）。最终要结合 SLA，如接口 P99 < 200ms 时停顿超 100ms 就会产生可见毛刺。
**Q3：为什么 JDK 9 把 G1 设为默认？** A：面向大堆与可预测停顿（CMS 的碎片与 CMF 在大堆上更严重）、不要求连续大内存（容器友好）、统一收集器方向以减少维护成本。
**Q4：堆越大越好吗？** A：不是，大堆单次 GC 扫描与复制成本更高，且会掩盖内存泄漏；应按压测与 GC 后老年代占用基线确定。
### 12.3 高频追问与陷阱
- 陷阱：`-XX:MaxRAMPercentage` 与 `-Xmx` 同时存在时以显式 `-Xmx` 为准，别混用。
- 陷阱：升级 JDK 17 后旧参数（`UseConcMarkSweepGC`、`Xloggc`、`PrintGCDetails`）会导致启动失败，必须有回归清单。
- 错误回答："调参能解决所有性能问题"（对象分配率与泄漏才是根因）。

### 12.5 一句话速记
- 4C8G：`-Xms4g -Xmx4g` + G1 + 元空间 512m；8C16G：`-Xms10g -Xmx10g` + `ParallelGCThreads=8`。
- 调优顺序：定目标 → 度量 → 压测 → 定位 → 调参 → 复测；代码优化优先于调参。
- 版本：JDK 9 G1 默认、11 ZGC 实验、14 CMS 移除、15 ZGC 生产可用、21 分代 ZGC。
- 记忆法：**先测量再调参，一次只改少量参数；代码分配率和泄漏问题优先于 JVM 参数**。

## 自测题（含答案要点）

1. JDK 8 为什么移除永久代？— 永久代大小难估易 OOM；与 JRockit 统一；元空间用本地内存、可归还 OS。
2. `-XX:MetaspaceSize` 与 `-XX:MaxMetaspaceSize` 的区别？— 前者是首次触发元空间 GC 的阈值（约 21MB），后者是上限。
3. TLAB 的作用与来源？— 线程私有的 Eden 分配缓冲，避免分配时 CAS 竞争，默认开启、约占 Eden 1%。
4. 逃逸分析带来的优化？— 标量替换（对象不创建）、栈上分配，附带锁消除。
5. Eden:S0:S1 比例与新生代占堆比例？— 8:1:1；新生代占堆 1/3（NewRatio=2）。
6. 动态年龄判定的规则？— Survivor 中同年龄对象大小之和超过 Survivor 一半时，年龄 ≥ 该年龄的对象提前晋升。
7. 空间分配担保的流程？— 老年代最大连续可用空间 > 新生代对象总大小则安全 Minor GC；否则按历次晋升平均值判断，不满足则 Full GC。
8. 三种 GC 算法分别由哪些收集器使用？— 标记清除 CMS；标记复制新生代（Serial/ParNew/PS/G1 的 Region 复制）；标记整理 Serial Old/Parallel Old。
9. GC Roots 至少列出 4 类。— 虚拟机栈局部变量、静态属性、常量、JNI 引用、JVM 内部引用、同步锁对象。
10. 四种引用与回收时机？— 强（永不）、软（内存不足）、弱（下次 GC）、虚（仅回收通知）。
11. CMS 的四个阶段与 STW 点？— 初始标记（STW）、并发标记、重新标记（STW）、并发清除。
12. Concurrent Mode Failure 的成因与对策？— 并发清理期间老年代被填满，退化为单线程 Serial Old Full GC；对策是降低触发阈值、增大老年代、减少晋升、换 G1。
13. 增量更新与 SATB 分别破坏漏标的哪个条件？— 增量更新记"新增引用"（破坏条件一，CMS）；SATB 记"删除引用"（破坏条件二，G1），代价是浮动垃圾。
14. G1 的 Humongous 对象是什么？— 大于 Region 一半的对象占用连续 Region，只在 Cleanup/Full GC 回收，短命大对象易触发 Full GC。
15. ZGC 的两大核心技术？— 着色指针（标记位存指针高位）+ 读屏障（读取时检查并在并发转移时自愈）。
16. 堆 dump 用什么工具分析？关键视图？— MAT；Leak Suspects、Dominator Tree、Path to GC Roots。
17. 类加载五个阶段与"准备阶段零值"的含义？— 加载、验证、准备、解析、初始化；`static int a = 10` 在准备阶段为 0，初始化才赋 10。
18. 打破双亲委派的三种场景？— SPI（TCCL）、Tomcat（WebApp 优先加载）、热部署（新 ClassLoader）。
19. Metaspace OOM 的典型原因？容器 OOMKilled 但 heap dump 正常的原因？— 前者是动态生成类过多（CGLIB/反射/JSP）与类加载器泄漏导致类无法卸载；后者是堆外内存（元空间、直接内存、线程栈、JIT）超限，或 JVM 未识别容器限制。
20. JDK 8 与 JDK 11 的 GC 日志参数差异？场景：服务每 2 小时一次 Full GC、每次 1.5s 怎么定位？— JDK 8 用 `-XX:+PrintGCDetails -Xloggc`，JDK 9+ 用 `-Xlog:gc*:file=...`；定位先看 GC 日志触发原因（Humongous Allocation / Metadata GC Threshold / System.gc / IHOP），再看老年代增长斜率与 dump 中的大对象持有链。

## 背诵卡片

1. 私有：PC、虚拟机栈、本地方法栈；共享：堆、元空间、直接内存。
2. 元空间：本地内存、默认无上限、可归还 OS，溢出报 `Metaspace`。
3. `-XX:MetaspaceSize` 是首次 GC 阈值（约 21MB），不是初始大小。
4. 静态变量与字符串常量池 JDK 7 起在堆中。
5. 分配方式：指针碰撞（有整理）/ 空闲列表（碎片）。
6. TLAB 默认开启、约 Eden 1%，避免分配时 CAS。
7. 逃逸分析 → 标量替换 + 栈上分配 + 锁消除（仅 C2 热点代码）。
8. Eden:S0:S1 = 8:1:1；新生代占堆 1/3。
9. 晋升年龄默认 15；动态年龄判定按"同年龄总和超 Survivor 一半"。
10. 大对象直接进老年代（PretenureSizeThreshold，G1 不支持）。
11. Minor GC 前做空间分配担保，不满足则 Full GC。
12. 三算法：标记清除（碎片）、标记复制（新生代）、标记整理（老年代）。
13. GC Roots：栈局部变量、静态属性、常量、JNI、内部引用、锁对象。
14. 软引用内存不足回收，弱引用下次 GC 回收，虚引用只做通知。
15. `finalize` 已废弃（JDK 9 标记、JDK 18 移除语义）。
16. CMS 四阶段：初始标记、并发标记、重新标记、并发清除（1、3 STW）。
17. CMS 三缺点：CPU 敏感、浮动垃圾、碎片；CMF 退化为单线程 Full GC。
18. CMS 在 JDK 9 废弃、JDK 14 移除。
19. G1：Region 化（约 2048 个）、RSet、SATB、停顿预测模型、Mixed GC。
20. G1 在 JDK 9 成为默认；Humongous 对象大于 Region 一半。
21. 漏标两条件：黑色新增指向白色 + 白色到灰色的引用被删。
22. 增量更新（CMS）记新增，SATB（G1）记删除，SATB 产生浮动垃圾。
23. 卡表按 512 字节分卡页，写屏障置脏。
24. ZGC：着色指针 + 读屏障；JDK 11 实验、15 生产、21 分代；不支持压缩指针。
25. Shenandoah 用 Brooks 转发指针，JDK 12 进主线；选型：Parallel（吞吐）/ G1（通用默认）/ ZGC（超大堆超低延迟）。
26. `-Xms` = `-Xmx`，元空间设上限，开 OOM 自动 dump；JDK 9+ 日志用 `-Xlog:gc*:file=...`。
27. 容器用 `-XX:MaxRAMPercentage`（JDK 8u191+ 支持 cgroup）。
28. 类加载五阶段：加载、验证、准备（零值）、解析、初始化；双亲委派保核心类唯一，打破场景为 SPI（TCCL）、Tomcat、热部署。
29. 类唯一标识 = ClassLoader + 全限定名；`Class.forName` 会初始化，`loadClass` 不会；`invokevirtual` 动态绑定，`invokestatic/invokespecial` 静态绑定。
30. `jstat -gcutil` 看 YGC/FGC/GCT；`jmap -histo:live` 会 Full GC；MAT 支配树 + Path to GC Roots 是定位泄漏的标准路径。
31. CPU 高排查：`top -Hp` → 转 16 进制 → `jstack nid` / Arthas `thread -n 3`；六类 OOM 为堆、GC overhead、Metaspace、直接内存、native thread、栈，容器 OOMKilled 但 dump 正常时查堆外。
32. 调优顺序：定目标 → 度量 → 压测 → 定位 → 调参 → 复测，代码优化优先；版本：JDK 9 G1 默认、11 ZGC 实验 + Epsilon、14 CMS 移除、15 ZGC 生产可用 + 偏向锁禁用、21 分代 ZGC + 虚拟线程。
