# Java 新特性与语言演进

> **定位**：按 JDK 8 / 11 / 17 / 21 四个 LTS 讲清"每个版本带来什么、生产能不能用、升级踩什么坑"，把新特性题从"背 API"变成"讲工程决策"。
> **重要度**：★★★★☆（JDK 17 与虚拟线程是近两年社招高频增量考点）
> **建议复习时长**：6 小时（Stream 与 Collectors 动手写 1 小时，升级清单 1 小时）
> **关联文档**：[Java 语言核心与面向对象](./01-Java语言核心与面向对象.md)、[集合框架](./02-集合框架.md)、[并发编程](./03-并发编程.md)、[JVM 与 GC 调优](./04-JVM与GC调优.md)、[Maven、Git 与工程实践](../07-工程化与DevOps/04-Maven-Git与工程实践.md)

## 0. 知识地图

| 知识点 | 重要度 | 常考形式 |
| --- | --- | --- |
| Java 8 Lambda 与函数式接口 | ★★★★★ | 八股原理 / 手写代码 |
| Stream 惰性求值、短路与并行流 | ★★★★★ | 源码追问 / 场景题 |
| Collectors 常用操作 | ★★★★☆ | 手写代码 |
| Optional 与空值处理 | ★★★★☆ | 手写代码 / 场景题 |
| 新日期时间 API | ★★★★☆ | 手写代码 / 场景题 |
| Java 8 其他关键变化 | ★★★★☆ | 版本差异 |
| Java 9-11（模块化、var、HttpClient） | ★★★★☆ | 版本差异 / 场景题 |
| Java 12-17（record、sealed、文本块） | ★★★★★ | 手写代码 / 版本差异 |
| Java 18-21（虚拟线程、模式匹配） | ★★★★★ | 八股原理 / 场景题 |
| 版本升级实践与面试答法 | ★★★★★ | 场景题 |

## 1. Lambda 与函数式接口

### 1.1 核心原理
先用人话理解 Lambda：它不是“一个神秘的新对象”，而是把一小段“要做什么”的代码当成参数传给别人。

```java
// 普通写法：把完整的遍历过程写出来
for (String name : names) {
    System.out.println(name);
}

// Lambda：只告诉 forEach“每个元素要做什么”
names.forEach(name -> System.out.println(name));
```

`name -> System.out.println(name)` 可以读成：

```text
拿到一个 name
→ 执行 println(name)
```

Lambda 必须放在一个**函数式接口**需要它的地方。函数式接口就是“只有一个抽象方法的接口”，这个唯一方法决定了 Lambda 的参数和返回值类型。

```java
Predicate<String> longName = name -> name.length() > 5;
Function<String, Integer> length = String::length;
Consumer<String> print = System.out::println;
Supplier<String> defaultName = () -> "unknown";
```

可以这样记：

```text
Predicate：问问题，返回 true/false
Function：输入一个值，转换成另一个值
Consumer：拿到值后消费，不返回结果
Supplier：不接收参数，负责提供一个值
```

Lambda 的编译目标是**函数式接口**（只有一个抽象方法的接口）；编译产物是 `invokedynamic` + `LambdaMetafactory`，运行期通过方法句柄生成实现类，**不产生 `Xxx$1.class`**（`javap -v` 可见 `InvokeDynamic #0:apply:()...`）。

| 接口 | 签名 | 变异版本 | 用途 |
| --- | --- | --- | --- |
| `Function<T,R>` | `R apply(T)` | `BiFunction`、`UnaryOperator`、`BinaryOperator` | 转换 |
| `Consumer<T>` | `void accept(T)` | `BiConsumer`、`IntConsumer` | 消费 |
| `Supplier<T>` | `T get()` | `BooleanSupplier`、`IntSupplier` | 供给/延迟 |
| `Predicate<T>` | `boolean test(T)` | `BiPredicate`、`IntPredicate` | 判断 |

方法引用四类：`类::静态方法`（`Integer::parseInt`）、`实例::实例方法`、`类::实例方法`（`String::length`，第一个参数作接收者）、`类::new`。

Lambda 与匿名内部类的差异：Lambda 用 invokedynamic 且 `this` 指向**外部实例**；匿名内部类生成独立 class 且 `this` 指自身，可有实例字段。

### 1.2 面试问答
**Q1：为什么 Lambda 只能用 effectively final 变量？** A：局部变量在栈上、生命周期短于 Lambda 对象，编译器把变量**值拷贝**进合成字段；为避免"副本与原件不一致"的错觉，强制其不可变。
**Q2：`@FunctionalInterface` 的作用？** A：编译期校验"恰好一个抽象方法"，`default`/`static`/`Object` 方法不计入。
**Q3：Lambda 有性能问题吗？** A：首次调用有引导开销，只发生一次；JIT 内联后与直接调用无差异，不要手动优化。
**Q4：Lambda 与匿名内部类能互相替换吗？** A：功能上近似，但 `this`/`super` 语义、作用域变量遮蔽规则不同，且 Lambda 不能有实例字段。

### 1.3 高频追问与陷阱
- 陷阱：Lambda 中 `this` 是外部类实例，如果用 `super` 或需要内部状态就必须用匿名内部类。
- 陷阱：多语句 Lambda 需 `{}` + `return`，不能与表达式体混写。
- 错误回答："Lambda 是匿名内部类的语法糖"。

### 1.5 一句话速记
- Lambda 走 invokedynamic，不生成 class 文件；`this` 指外部实例。
- 捕获局部变量必须 effectively final，因为值拷贝进合成字段。

## 2. Stream 惰性求值、短路与并行流

### 2.1 核心原理
#### 先用人话理解 Stream

Stream 不是一个新集合，也不是把数据复制一份。它更像一条“数据加工流水线”：

```text
原始集合
  ↓
过滤不需要的数据
  ↓
转换数据形状
  ↓
排序/去重/截取
  ↓
得到结果
```

例如业务需求是“找出金额大于 100 的订单编号”：

```java
List<String> ids = orders.stream()
        .filter(order -> order.getAmount() > 100)
        .map(Order::getId)
        .toList();
```

这段代码可以翻译成人话：

```text
从 orders 开一条流水线
→ 只保留金额大于 100 的订单
→ 每个订单只取 id
→ 把最终结果收集成 List
```

Stream 关注的是“数据要经过哪些步骤”；普通 `for` 循环关注的是“我现在如何控制下标、临时变量和遍历过程”。

#### Stream 的完整执行流程

Stream 通常分成三段：

```text
数据源 Source
    ↓
中间操作 Intermediate Operations
    ↓
终端操作 Terminal Operation
```

以这段代码为例：

```java
long count = orders.stream()
        .filter(order -> order.getAmount() > 100)
        .map(Order::getId)
        .distinct()
        .limit(10)
        .count();
```

执行过程不是“先把所有订单 filter 完，再把全部结果 map 完”。更接近下面这样：

```text
订单 1 → filter → map → distinct → limit → count
订单 2 → filter → map → distinct → limit → count
订单 3 → filter → map → distinct → limit → count
...
```

这叫**垂直执行**。只有遇到终端操作 `count()`，前面的操作才真正开始执行。

```text
只有 stream + filter + map：
    什么都不执行

加上 count/collect/forEach：
    开始拉取元素，逐个通过流水线
```

每个阶段可以这样理解：

1. `stream()`：创建流水线，不读取全部数据；
2. `filter`：定义筛选规则；
3. `map`：定义转换规则；
4. `distinct`：需要记住已经出现过的元素；
5. `limit`：达到数量后停止继续读取；
6. `count`：启动流水线并汇总结果。

#### Stream 和普通循环有什么区别？

同一个需求用两种写法对比：

```java
// 普通循环
List<String> result = new ArrayList<>();
for (Order order : orders) {
    if (order.getAmount() > 100) {
        result.add(order.getId());
    }
}
```

```java
// Stream
List<String> result = orders.stream()
        .filter(order -> order.getAmount() > 100)
        .map(Order::getId)
        .toList();
```

区别不在于 Stream 一定更快，而在于关注点不同：

| 对比 | 普通循环 | Stream |
| --- | --- | --- |
| 关注重点 | 如何遍历、如何修改结果 | 数据经过哪些处理步骤 |
| 中间结果 | 常需要手动创建临时集合 | 由流水线连接 |
| 提前结束 | 手动 `break`/`return` | `findFirst`/`limit` 等短路操作 |
| 可读性 | 简单流程直观，复杂流程容易嵌套 | 转换、过滤、分组表达清晰 |
| 调试 | 可以逐行断点 | 链过长时定位较困难 |
| 性能 | 小数据、简单循环通常更直接 | 可能减少中间集合，适合组合处理 |
| 控制能力 | 可随时修改外部状态 | 鼓励无副作用、声明式处理 |

选择原则：

```text
简单遍历、需要复杂 break、频繁修改外部状态 → 普通循环
过滤 + 转换 + 分组 + 汇总等数据处理 → Stream
```

#### Stream 的优势到底在哪里？

1. **把业务意图写出来**：`filter`、`map`、`groupingBy` 比多个临时变量更接近需求；
2. **操作可以组合**：过滤、转换、排序、分组可以拼成一条流程；
3. **惰性和短路**：配合 `limit`、`findFirst`，不必处理全部数据；
4. **减少中间集合**：许多操作可以在一条流水线中完成；
5. **有并行扩展能力**：部分无状态、计算密集任务可以改成并行流。

但 Stream 不是“代码更短就一定更好”：链条太长、需要复杂分支、需要修改外部变量时，普通循环通常更容易读和调试。

- **创建**：`collection.stream()`、`Arrays.stream(arr)`、`Stream.of`、`Stream.iterate/generate`、`IntStream.range`、`Files.lines`。
- **中间操作（惰性）**：`filter`、`map`、`flatMap`、`distinct`、`sorted`、`limit`、`skip`、`peek`、`mapToInt/mapToObj`、`boxed`。
- **终端操作（触发执行）**：`forEach`、`collect`、`reduce`、`count`、`anyMatch/allMatch/noneMatch`、`findFirst/findAny`、`min/max`、`toArray`。

**惰性求值**：没有终端操作则中间操作什么都不做；一个元素会被尽量深地沿流水线处理完（垂直执行），因此 `filter` 后的 `limit` 能提前终止。**短路操作**：`limit`、`findFirst`、`findAny`、`anyMatch`、`allMatch`、`noneMatch`、`takeWhile`/`dropWhile`（JDK 9）。
```java
Stream.iterate(1, i -> i + 1).filter(i -> i % 3 == 0).limit(5).forEach(System.out::println);  // 3 6 9 12 15
```
流**只能消费一次**，重复消费抛 `IllegalStateException: stream has already been operated upon or closed`。

#### 为什么 `limit` 可以提前停止？

因为 Stream 会沿流水线逐个处理元素。下面的无限流不会死循环：

```java
List<Integer> values = Stream.iterate(1, i -> i + 1)
        .filter(i -> i % 2 == 0)
        .limit(3)
        .toList();
// [2, 4, 6]
```

处理到第 6 个元素后，`limit(3)` 已经满足要求，终端操作可以结束。这里的关键不是 `filter` 很快，而是 `limit` 发出了“够了，停止拉取”的信号。

但 `sorted()`、`distinct()` 这类有状态操作通常需要先看到更多甚至全部元素：

```text
filter/limit：可以边读边决定，容易短路
sorted：不知道后面有没有更小的值，通常要先收集再排序
distinct：要记住已见过的值
```

#### 一个元素是怎样流过 Stream 的？

```java
List<String> result = names.stream()
        .filter(name -> {
            System.out.println("filter: " + name);
            return name.length() >= 4;
        })
        .map(name -> {
            System.out.println("map: " + name);
            return name.toUpperCase();
        })
        .limit(2)
        .toList();
```

可能的处理顺序是：

```text
Tom     → filter（失败）
Alice   → filter（成功）→ map → limit 计数 1
Bob     → filter（失败）
Jerry   → filter（成功）→ map → limit 计数 2 → 停止
```

它不会先把所有名字 filter 一遍，再统一 map；这就是 Stream 的“垂直执行 + 惰性求值”。

**并行流**：`parallelStream()` 底层是 `ForkJoinPool.commonPool()`（并行度 = CPU 核数 - 1）。适用条件：数据量大（数万元以上）、计算密集无阻塞、无共享可变状态、满足结合律。`findFirst` 在并行流上开销大（需保证遇到顺序），用 `findAny`；并行 `forEach` 不保证顺序，要顺序用 `forEachOrdered`。

### 2.2 面试问答
**Q1：`map` 与 `flatMap` 的区别？** A：`map` 一对一；`flatMap` 一对多并把每个元素映射出的流展平成一个流（必须在其中 `Stream.of(...)` 或 `Arrays.stream(...)`）。
**Q2：`peek` 会执行吗？** A：是中间操作，只在有终端操作时执行；官方定位为调试用，不要用它做副作用业务逻辑。
**Q3：`reduce` 三参版本的 combiner 何时调用？** A：只在并行流中用于合并分片结果，串行流不调用；要求满足结合律，否则并行结果错误。
**Q4：并行流有哪些坑？** A：占全局 commonPool（阻塞会拖垮其他使用者）；数据量小反而更慢；依赖顺序的操作性能差；有状态 Lambda（外部计数器）会出错；用非线程安全容器收集（`forEach(list::add)`）会丢数据。
**Q5：怎么给并行流指定线程池？** A：`new ForkJoinPool(n).submit(() -> list.parallelStream()... ).join()`；池内不能再调阻塞操作（会死锁），JDK 10+ 更推荐 CompletableFuture + 业务池。

### 2.3 高频追问与陷阱
- 陷阱：`sorted()` 与 `distinct()` 是**有状态**操作，会缓冲全部元素，在无限流上永久阻塞。
- 陷阱：`IntStream.sum()` 可能溢出，超大求和用 `mapToLong().sum()` 或 BigDecimal。
- 陷阱：`Collectors.toMap` 遇到重复 key 抛 `IllegalStateException: Duplicate key`，必须传 merge 函数。
- 错误回答："并行流一定比串行快"。

### 2.4 代码/配置示例
```java
List<String> top3 = orders.stream()
        .sorted(Comparator.comparing(Order::getAmount).reversed())
        .limit(3).map(Order::getId).collect(Collectors.toList());
long total = orders.stream().mapToLong(Order::getAmount).sum();     // 避免 int 溢出
```

### 2.5 一句话速记
- 中间操作惰性、终端操作触发；垂直执行使 `limit` 能短路。
- 流只能消费一次；并行流默认用 commonPool（CPU-1），别做 IO。
- 记忆法：**先搭流水线，终端操作才开工；一个元素走到底，够了就停止**。

## 3. Collectors 速查

### 3.1 核心原理
`Stream` 负责“逐个处理元素”，`Collector` 负责“把处理结果装回一个最终结构”：

```text
Stream：数据怎么流动？
Collector：数据最后装在哪里？
```

例如：

```java
Map<String, List<Order>> byCity = orders.stream()
        .collect(Collectors.groupingBy(Order::getCity));
```

执行过程可以理解为：

```text
读取一个订单
→ 取出城市
→ 找到该城市对应的 List
→ 把订单放进去
→ 继续读取下一个订单
```

`groupingBy` 不是简单排序，而是创建 `Map<分组键, 分组结果>`；`partitioningBy` 则固定分成 `true` 和 `false` 两组。

| 需求 | 写法 |
| --- | --- |
| List / Set | `Collectors.toList()`、`toSet()`、`toUnmodifiableList()`（JDK 10+） |
| Map | `toMap(k, v)`、`toMap(k, v, (a,b)->b)`（重复 key 合并） |
| 指定 Map 实现 | `toMap(k, v, merge, TreeMap::new)`、`toCollection(ArrayDeque::new)` |
| 分组 | `groupingBy(Class::getCity)` |
| 分组 + 计数 | `groupingBy(k, Collectors.counting())` |
| 分组 + 映射 | `groupingBy(k, mapping(Order::getId, toList()))` |
| 分区（固定两组） | `partitioningBy(o -> o.getAmount() > 100)` |
| 连接字符串 | `joining(",")`、`joining(", ", "[", "]")` |
| 求和/平均 | `summingInt/Long/Double`、`averagingInt`、`summarizingInt` |
| 聚合 | `reducing(0, Order::getAmount, Integer::sum)`、`minBy`、`maxBy` |
| 一次算两组结果 | `teeing(c1, c2, merge)`（JDK 12+） |
| 嵌套分组 | `groupingBy(a, groupingBy(b))` |

### 3.2 面试问答
**Q1：`groupingBy` 返回什么 Map？** A：默认 `HashMap`；需要有序用 `groupingBy(k, TreeMap::new, downstream)` 或 `LinkedHashMap::new`。
**Q2：`groupingBy` 与 `partitioningBy` 的区别？** A：后者只分两组且 Map 一定包含 true/false 两个 key（即使某组为空），key 类型是 Boolean。
**Q3：`toMap` 有哪些坑？** A：value 为 null 会抛 NPE（内部走 `HashMap.merge`）；重复 key 必须给 merge 函数；要保留顺序需指定 `LinkedHashMap::new`。
**Q4：金额统计推荐怎么写？** A：避开 `summingInt`（溢出），用 `summingLong` 或自定义 reduce 到 `BigDecimal`。

### 3.3 高频追问与陷阱
- 陷阱：`Collectors.toList()` 的实现未指定（当前是 ArrayList），要可变且确定用 `toCollection(ArrayList::new)`。
- 陷阱：`teeing` 是 JDK 12 引入，JDK 8 项目不可用。
- 错误回答："`toMap` 重复 key 会自动覆盖"（默认抛异常，除非给 merge）。

### 3.5 一句话速记
- `groupingBy(k, downstream)` 是分组万能公式；`toMap` 必带 merge 函数。
- 金额统计避开 `summingInt`（溢出）。

## 4. Optional 与空值处理

### 4.1 核心原理
`Optional` 可以理解为一个“明确写出来的结果盒子”：盒子里可能有值，也可能明确为空。它的价值不是让空值消失，而是迫使调用者考虑“没有值怎么办”。

```java
Optional<User> user = userRepository.findById(id);

String name = user
        .map(User::getName)
        .orElse("anonymous");
```

这段代码可以读成：

```text
查找用户
→ 找到：取出 name
→ 没找到：使用 anonymous
```

适合用 Optional 表达“查询可能没有结果”，不建议把它用于实体字段、方法参数或所有局部变量。集合查询没有结果时，通常返回空集合比返回 `Optional<List<T>>` 更自然。

`Optional<T>` 是"可能没有值"的容器，**定位为方法返回值**（官方不建议做字段或参数）。

| 方法 | 说明 |
| --- | --- |
| `of` / `ofNullable` / `empty` | 创建（`of(null)` 抛 NPE） |
| `isPresent` / `isEmpty`（JDK 11+） | 判断 |
| `get()` | 空则抛 `NoSuchElementException`（应尽量避免） |
| `orElse(other)` | 空则返回 other，**other 一定会被求值** |
| `orElseGet(supplier)` | 空才调用 supplier |
| `orElseThrow(supplier)` | 空则抛异常（JDK 10 起有无参版本） |
| `ifPresent` / `ifPresentOrElse`（JDK 9） | 消费 |
| `map` / `flatMap` / `filter` / `stream()`（JDK 9） | 链式转换 |

```java
String city = Optional.ofNullable(user).map(User::getAddress).map(Address::getCity).orElse("未知");
User u = opt.orElse(repo.findDefault());          // 无论是否有值都会查库
User v = opt.orElseGet(() -> repo.findDefault()); // 只有为空才查库
```

### 4.2 面试问答
**Q1：`orElse` 与 `orElseGet` 的关键区别？** A：Java 是值传递，`orElse` 的实参**先求值**；`orElseGet` 用 Supplier 延迟求值。若实参是远程调用或大对象构造，用错会造成无谓开销。
**Q2：Optional 能消除所有 NPE 吗？** A：不能，只用于返回值的显式空语义；集合应返回空集合、数组返回长度 0，入参用 `Objects.requireNonNull` 快速失败。
**Q3：为什么避免 `get()`？** A：与"直接返回 null 再判断"没有本质区别，仍可能抛异常；应使用 `map/orElse/orElseThrow/ifPresent`。
**Q4：为什么不做字段或参数？** A：增加一层包装与 GC 压力、序列化语义不清晰，不如 `@Nullable` 与校验注解直观。

### 4.3 高频追问与陷阱
- 陷阱：`map` 函数返回 null 时结果是 `Optional.empty()`，而不是带 null 的 Optional。
- 陷阱：`Optional.of(null)` 抛 NPE，必须用 `ofNullable`。
- 错误回答："Optional 能消除所有 NPE"。

### 4.5 一句话速记
- `orElse` 必求值、`orElseGet` 懒求值；避免 `get()`。
- Optional 只做返回值；集合返回空集合比 Optional 更合适。

## 5. 新日期时间 API

### 5.1 核心原理
旧日期 API 的问题可以先记成一句话：

```text
Date/Calendar 可变、时区语义混杂、SimpleDateFormat 线程不安全。
```

`java.time` 把“日期”“时间”“时区”“时间间隔”拆成不同类型：

```text
LocalDate       2026-11-12        只有日期
LocalTime       09:30             只有时间
LocalDateTime   2026-11-12 09:30  没有时区
Instant         UTC 时间点         机器之间传递
ZonedDateTime   带时区的时间        展示和转换
```

| 类型 | 含义 | 线程安全 |
| --- | --- | --- |
| `LocalDate`/`LocalTime`/`LocalDateTime` | 无时区的日期/时间 | 不可变、安全 |
| `Instant` | UTC 时间戳（机器时间） | 安全 |
| `ZonedDateTime`/`OffsetDateTime` | 带时区/偏移 | 安全 |
| `Duration`/`Period` | 时间量（秒/纳秒）/ 日期量（年月日） | 安全 |
| `DateTimeFormatter` | 格式化（替代 `SimpleDateFormat`） | **不可变、安全** |

`SimpleDateFormat` 非线程安全（内部共享 Calendar），三种解法：每次 new（昂贵）、ThreadLocal 包装（注意 remove）、**改用 `DateTimeFormatter`（推荐）**。

```java
LocalDateTime now = LocalDateTime.now();
String s = now.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
long days = ChronoUnit.DAYS.between(LocalDate.of(2024, 1, 1), LocalDate.now());
LocalDateTime next = now.plusDays(7).withHour(0).withMinute(0).withSecond(0).withNano(0);
```
与旧 API 互转：`Date.from(instant)`、`date.toInstant()`、`java.sql.Timestamp.valueOf(ldt)`、`Instant.ofEpochMilli(ms)`。

### 5.2 面试问答
**Q1：`LocalDateTime` 与 `ZonedDateTime` 怎么选？** A：跨时区存储与转换用 `Instant` + `ZonedDateTime`；业务展示与单据日期用 `LocalDateTime`；**数据库存 UTC** 避免夏令时/时区歧义。
**Q2：`withXxx` 会修改原对象吗？** A：不会，全部返回新对象（不可变），所以必须接收返回值，否则"改了没生效"。
**Q3：`Duration` 与 `Period` 的区别？** A：Duration 基于时间（秒/纳秒），Period 基于日期（年月日）；`Period.between(2024-01-31, 2024-03-01)` 的月份天数不对称，需留意。
**Q4：`yyyy` 与 `YYYY` 的区别？** A：`yyyy` 是日历年，`YYYY` 是 **ISO 周所属年**，跨年周会导致元旦前后显示错误年份（如 2024-12-30 用 `YYYY` 显示 2025），必须用小写。

### 5.3 高频追问与陷阱
- 陷阱：Jackson 默认不支持 `LocalDateTime`，需注册 `JavaTimeModule`（Spring Boot 已自动注册）+ `@JsonFormat`。
- 陷阱：毫秒转时间用 `Instant.ofEpochMilli`，手工 `ofEpochSecond(sec,0,offset)` 容易差 1000 倍。
- 错误回答："`SimpleDateFormat` 是线程安全的"。

### 5.5 一句话速记
- `java.time` 全部不可变线程安全；格式化用 `DateTimeFormatter`。
- 年份用小写 `yyyy`（`YYYY` 是周历年份，会出错）。

## 6. Java 8 其他关键变化

### 6.1 核心原理
Java 8 的核心变化可以用三条线记忆：

```text
写法：Lambda + Stream
接口：default 方法，让接口可以继续演进
运行时：元空间、ConcurrentHashMap、CompletableFuture 等工程能力增强
```

| 变化 | 说明 |
| --- | --- |
| 接口 `default`/`static` 方法 | 支持接口演进不破坏实现类；菱形冲突需显式 `A.super.m()` |
| `HashMap` 红黑树优化 | 链表 ≥ 8 且容量 ≥ 64 树化（见 [集合框架](./02-集合框架.md)） |
| `ConcurrentHashMap` 换实现 | 分段锁 → CAS + synchronized + 红黑树 |
| 元空间取代永久代 | `PermGen` → `Metaspace`（见 [JVM 与 GC 调优](./04-JVM与GC调优.md)） |
| `CompletableFuture`/`StampedLock`/`LongAdder` | 并发增强（见 [并发编程](./03-并发编程.md)） |
| `Arrays.parallelSort` | 大数组并行排序 |
| 类型注解（JSR 308）、`-parameters` | 反射可获取参数名（Spring MVC 依赖） |

`default` 方法不能覆盖 `equals`/`hashCode`/`toString`（编译报错）；接口 `static` 方法不被实现类继承，必须用接口名调用；接口不能有实例字段（只有 `public static final` 常量）。JDK 8 的 `default` 是"接口二进制兼容演进"的标准手段 —— 给 `Collection` 加 `stream()` 时所有第三方实现类零改动。

### 6.2 面试问答
**Q1：`default` 方法解决了什么问题？** A：接口演进兼容性；否则每次给接口加方法都会破坏所有实现类。
**Q2：菱形冲突怎么办？** A：多个接口提供同名同参 `default` 时实现类必须重写并显式 `InterfaceA.super.method()`；类中的具体方法优先级高于接口 default。
**Q3：项目从 JDK 8 升到 11/17 时，哪些 JDK 8 习惯会出问题？** A：直接使用 `sun.misc.Unsafe`、静态共享 `SimpleDateFormat`、依赖 `PermSize` 参数、`-Xverify:none`、反射访问 `java.*` 内部 API。
**Q4：JDK 8 最重要的三项工程价值？** A：Lambda/Stream 改变写法、元空间解决 PermGen 痛点、并发与集合的性能重写（CHM/CompletableFuture/LongAdder）。

### 6.3 高频追问与陷阱
- 陷阱：接口的 `static` 方法不能被继承，`Impl.staticMethod()` 不通。
- 陷阱：`default` 方法中的 `this` 是实现类实例，可调用接口其他方法。
- 错误回答："JDK 8 之后接口有了状态"（仍无实例字段）。

### 6.5 一句话速记
- `default` 方法解决接口演进的二进制兼容问题。
- 接口 static 方法不被继承；default 不能覆盖 Object 的方法。

## 7. Java 9-11

### 7.1 核心原理
JDK 9 到 11 可以按“模块、工具、字符串、网络”理解：

```text
JDK 9：模块化、集合工厂、统一日志、Stream/Optional 增强
JDK 10：var，减少局部变量类型重复
JDK 11：LTS、HttpClient、字符串/文件 API、单文件运行
```

**Java 9**：① **模块化 JPMS**（`module-info.java` 的 `requires/exports/opens`，价值在强封装、`jlink` 裁剪运行时、更小镜像）；② **集合工厂方法** `List.of`/`Set.of`/`Map.of`（不可变、禁 null，见 [集合框架](./02-集合框架.md)）；③ **接口私有方法**（JEP 213）；④ Stream 增强（`takeWhile`/`dropWhile`/`ofNullable`/三参 `iterate`）；⑤ `Optional` 增强（`ifPresentOrElse`/`or`/`stream`）；⑥ `ProcessHandle` API；⑦ **G1 成为默认 GC**（JEP 248）、CMS 标记废弃（JEP 291）；⑧ **统一 JVM 日志** `-Xlog`（JEP 158）；⑨ 多版本 JAR（`META-INF/versions/9/`）；⑩ JShell。

**Java 10**：① **`var` 局部变量类型推断**（JEP 286）；② `Collectors.toUnmodifiableList/Set/Map`、`List.copyOf`/`Set.copyOf`/`Map.copyOf`；③ G1 Full GC 并行化（JEP 307）；④ 应用类数据共享 AppCDS（JEP 310）加速启动。

**Java 11（LTS）**：① **`HttpClient` 正式**（HTTP/2、WebSocket、`sendAsync` 返回 CompletableFuture）；② **单文件源码运行**（`java Hello.java`）；③ **String 新 API**：`isBlank`、`lines`、`strip`、`repeat`、`chars`；④ `Files.readString/writeString`；⑤ **ZGC 实验特性**（JEP 333）、**Epsilon**（JEP 318）；⑥ JFR 开源（JEP 328）；⑦ Lambda 形参可用 `var` 加注解（JEP 323）；⑧ **移除 Java EE 与 CORBA 模块**（JAXB、JAX-WS、JAF、Annotation Common）。

`var` 边界：仅用于**局部变量**（含 for、try-with-resources），必须有初始化值；不能用于字段、方法参数、返回类型、`var x = null`、Lambda 整体表达式、数组字面量。建议右侧类型已明确（`var list = new ArrayList<String>()`）时使用。

| 特性 | `String.strip()` | `String.trim()` |
| --- | --- | --- |
| 判定 | `Character.isWhitespace`（含 Unicode 空格） | 只处理 ≤ U+0020 |

### 7.2 面试问答
**Q1：`var` 能替代所有显式类型吗？** A：不能，边界见上；工程上建议"右侧可读时用 var，工厂方法/流等右侧不可读时保留显式类型"。
**Q2：JPMS 在生产落地了吗？** A：多数业务服务未落地（依赖库未模块化、收益不明显），但 JDK 16 起的**强封装**所有人都会遇到：反射访问 `java.*` 内部 API 需 `--add-opens`。
**Q3：Java 11 升级最常见的三个坑？** A：① `javax.xml.bind`/`javax.annotation`/`javax.activation` 被移除需加依赖；② 反射访问内部 API 报 `InaccessibleObjectException` 需 `--add-opens`；③ `sun.misc.Unsafe`/`sun.reflect` 不可用，同时 IDE/构建工具与 Lombok 需同步升级。
**Q4：`HttpClient` 相比 `HttpURLConnection` 的优势？** A：内建 HTTP/2、异步非阻塞、连接池与超时配置清晰、无第三方依赖；不足是连接池细节控制不如 Apache HttpClient/OkHttp（Feign/RestTemplate 底层仍常用它们）。

### 7.3 高频追问与陷阱
- 陷阱：`List.of` 禁 null 且不可变（序列化行为与 ArrayList 不同）。
- 陷阱：`String.repeat(n)` 的 n 为负会抛 `IllegalArgumentException`。
- 错误回答："JDK 11 移除了 ZGC"（是引入 ZGC 并移除 Java EE/CORBA）。

### 7.4 代码/配置示例
```java
HttpClient c = HttpClient.newBuilder().connectTimeout(Duration.ofSeconds(2))
        .version(HttpClient.Version.HTTP_2).build();
HttpRequest req = HttpRequest.newBuilder(URI.create(url)).timeout(Duration.ofSeconds(3)).GET().build();
HttpResponse<String> resp = c.send(req, HttpResponse.BodyHandlers.ofString());
c.sendAsync(req, HttpResponse.BodyHandlers.ofString()).thenApply(HttpResponse::body).thenAccept(System.out::println);
```

### 7.5 一句话速记
- JDK 9：模块化 + 集合工厂 + 接口私有方法 + G1 默认 + 统一日志。
- JDK 10：`var` 局部变量；JDK 11：HttpClient、String 新 API、单文件运行、ZGC 实验、Java EE 移除（升级最大坑）。

## 8. Java 12-17

### 8.1 核心原理
JDK 12 到 17 的主线是“让代码更像业务模型，同时减少样板代码”：

```text
switch 表达式：分支可以直接产生值
文本块：多行文本不再堆转义符
record：用很少代码表达数据载体
sealed：限制谁可以继承
模式匹配：判断类型时同时完成转换
```

| 特性 | 版本 | 要点 |
| --- | --- | --- |
| Switch 表达式（JEP 361） | 12 预览 / **14 正式** | `->` 语法、`yield` 返回值、**不贯穿**、需覆盖所有分支或有 default |
| 文本块（JEP 378） | 13 预览 / **15 正式** | `"""`，缩进由最小缩进行与结尾定界符共同决定；行尾 `\` 取消换行、`\s` 保留尾随空格 |
| `instanceof` 模式匹配（JEP 394） | 14 预览 / **16 正式** | `if (o instanceof String s)`，s 在条件成立分支内可用 |
| `record`（JEP 395） | 14 预览 / **16 正式** | 不可变数据载体，自动生成构造器、accessor（`name()` 无 get 前缀）、`equals/hashCode/toString`；隐式 final，不能继承也不能被继承 |
| `sealed` 密封类（JEP 409） | 15 预览 / **17 正式** | `permits` 限定子类，子类必须声明 `final`/`sealed`/`non-sealed`，配合模式匹配做穷尽检查 |
| 增强 NPE 提示（JEP 358） | 14（15 起默认开启） | 精确到变量名：`Cannot invoke "String.length()" because "user.name" is null` |
| `RandomGenerator`（JEP 356） | 17 | 统一接口与实现族（LXM 等算法），`Random` 保持默认以兼容 |
| 强封装内部 API（JEP 396） | 16 | 反射 `java.*` 内部实现默认被拒 |
| 移除/废弃 | 14/15/17 | CMS 移除（14）、Nashorn 移除（15）、偏向锁禁用（15）、SecurityManager 废弃（17） |
| 其他 | 12-17 | `Collectors.teeing`（12）、`Stream.toList()`（16，返回不可变 List）、`String.indent/transform`、`CompactNumberFormat`、向量 API 孵化、`--enable-preview` 机制 |

```java
int days = switch (month) {
    case 1, 3, 5, 7, 8, 10, 12 -> 31;
    case 2 -> { yield isLeap ? 29 : 28; }
    default -> throw new IllegalArgumentException("month=" + month);
};
public sealed interface Shape permits Circle, Rect {}
public record Circle(double r) implements Shape {}
if (shape instanceof Circle c) return Math.PI * c.r() * c.r();
```

### 8.2 面试问答
**Q1：`record` 与 Lombok `@Data` 的区别？** A：record 是语言级不可变数据类（字段 final、无 setter、不能继承、不能定义实例字段，可有静态成员与紧凑构造器、可实现接口）；Lombok 是编译期字节码增强，支持继承、可变、Builder，需要 IDE 插件。
**Q2：`sealed` 有什么用？** A：把"谁能实现该接口"变成编译期事实，使模式匹配可穷尽检查（switch 不写 default 也不报错），并让领域建模更精确。
**Q3：文本块的缩进规则？** A：以最小缩进行与结尾定界符位置共同决定剥离量；行尾 `\` 取消换行，`\s` 保留尾随空格。
**Q4：JDK 17 的工程意义？** A：继 JDK 8 之后被广泛采用的 LTS，Spring Boot 3.x 的最低要求；收集器与语言特性成熟（G1 稳定、ZGC 可用、record/sealed/模式匹配齐备）；强封装与移除项倒逼依赖升级，完成后可长期停留。
**Q5：`Stream.toList()` 与 `Collectors.toList()` 的区别？** A：前者（JDK 16+）返回**不可变** List 且允许 null 元素；后者返回可变 List（实现未指定）。

### 8.3 高频追问与陷阱
- 陷阱：`record` 用 `Objects.equals` 比较所有字段，字段是数组时比较的是引用，需要自定义 `equals`。
- 陷阱：`switch` 模式匹配是 **JDK 21** 才正式，JDK 17 只有预览（JDK 17 正式的是 `instanceof` 模式匹配与 sealed）。
- 陷阱：升级 JDK 17 后 `SecurityManager` 受限，依赖它的老框架（老 Tomcat、部分插件）需替换。
- 错误回答："record 可以被继承"；"JDK 17 已支持 switch 模式匹配"。

### 8.5 一句话速记
- JDK 14：switch 表达式正式 + 增强 NPE；15：文本块正式 + 偏向锁禁用；16：record + instanceof 模式匹配 + 强封装；17：sealed 正式 + LTS。
- `record` 不可变不可继承、accessor 无 `get` 前缀；JDK 17 没有 switch 模式匹配。

## 9. Java 18-21

### 9.1 核心原理
JDK 18 到 21 的重点可以这样理解：

```text
JDK 18：默认 UTF-8，减少平台差异
JDK 21：让并发、模式匹配和有序集合更好用
```

虚拟线程解决的是“线程创建和阻塞等待太贵”，不是让 CPU 变多；模式匹配解决的是“先判断类型、再强转、再取字段”的样板代码。

| 特性 | 状态 | 要点 |
| --- | --- | --- |
| UTF-8 为默认字符集（JEP 400） | JDK 18 | 依赖平台默认编码（GBK）的代码会乱码 |
| 简单 Web 服务器（JEP 408） | JDK 18 | `jwebserver`，仅用于原型 |
| **虚拟线程**（JEP 444） | 19/20 预览 / **21 正式** | 见 [并发编程](./03-并发编程.md) |
| **结构化并发**（JEP 453） | **21 预览** | `StructuredTaskScope`，子任务生命周期绑定作用域 |
| **Scoped Values**（JEP 446） | **21 预览** | 不可变、有作用域的上下文，替代部分 ThreadLocal |
| **Sequenced Collections**（JEP 431） | **21 正式** | `SequencedCollection/Set/Map`：`getFirst/getLast/addFirst/addLast/reversed` |
| **switch 模式匹配**（JEP 441） | 17-20 预览 / **21 正式** | `case Circle c ->`、`case null ->`、`when` 守卫、穷尽性检查 |
| **记录模式**（JEP 440） | **21 正式** | `case Point(int x, int y) ->` 直接解构 record |
| **分代 ZGC**（JEP 439） | **21** | `-XX:+UseZGC -XX:+ZGenerational`，JDK 23 起默认 |
| 其他 | 18-21 | `Math.clamp`、`List.getFirst/getLast`（来自 SequencedCollection）、未命名类与实例 main 方法预览（JEP 445） |

```java
String desc = switch (obj) {
    case Integer i when i > 100 -> "big int";
    case Integer i -> "int";
    case Point(int x, int y) -> "point " + x + "," + y;   // 记录模式
    case null -> "null";                                  // 新模式允许显式处理 null
    default -> "other";
};
```
`reversed()` 返回的是**视图**（与原集合互相影响），不是拷贝。

### 9.2 面试问答
**Q1：JDK 21 最值得升级的三个理由？** A：虚拟线程正式（IO 密集服务用同步代码获得异步级并发）、switch 模式匹配 + 记录模式正式（配合 sealed 做穷尽领域建模）、分代 ZGC（超大堆低延迟）；另有 Sequenced Collections 补齐集合 API 短板。
**Q2：虚拟线程在 JDK 21 的硬约束？** A：`synchronized` 中阻塞会**钉住载体线程**（JDK 24 的 JEP 491 才解决），应改用 `ReentrantLock`；不要池化；注意 ThreadLocal 内存放大；CPU 密集任务无收益；可用 `-Djdk.tracePinnedThreads=full` 排查。
**Q3：`ScopedValue` 与 ThreadLocal 的差别？** A：ScopedValue 不可变、绑定有明确作用域（`ScopedValue.where(K, v).run(...)`），作用域结束即失效，适合虚拟线程；缺点是作用域内不能修改、不自动跨作用域继承。
**Q4：结构化并发解决什么？** A：任务泄漏与取消传播：`ShutdownOnFailure` 任一失败即取消其余，`ShutdownOnSuccess` 取最快成功者，异常与栈追踪反映真实调用层级（对比 CompletableFuture 的孤儿任务）。
**Q5：Sequenced Collections 的实际价值？** A：统一有序集合 API：`list.getFirst()` 替代 `get(0)`、`LinkedHashMap` 有了 `firstEntry/lastEntry/reversed`、`List` 与 `Deque` 语义对齐。

### 9.3 高频追问与陷阱
- 陷阱：JDK 18 起默认字符集为 UTF-8（JEP 400），依赖 `file.encoding=GBK` 隐式行为的读写会乱码 —— 升级必查项。
- 陷阱：只写 `default` 而不写 `case null` 时，switch 遇到 null 仍抛 NPE（`case null` 是显式处理的前提）。
- 陷阱：虚拟线程不是"更快的线程"，延迟不降；把它用于 CPU 密集任务会因调度开销变慢。
- 错误回答："虚拟线程可以替代线程池"（限流语义仍需 Semaphore 或网关）。

### 9.5 一句话速记
- JDK 21 正式：虚拟线程、switch 模式匹配、记录模式、Sequenced Collections、分代 ZGC。
- 结构化并发（作用域生命周期）+ Scoped Values（不可变上下文）+ 虚拟线程 = Loom 三件套。

## 10. 版本升级实践与面试答法

### 10.1 核心原理
升级 JDK 不是只改一个 `java.version`，而是把“编译器、运行时、依赖、启动参数、行为”一起验证：

```text
依赖是否支持新 JDK
    ↓
代码能否编译
    ↓
JVM 参数是否仍有效
    ↓
反射/字符集/GC 行为是否变化
    ↓
测试、压测、灰度和回滚
```

**升级路线**：① 评估依赖支持（Spring Boot 3 需 JDK 17+；Lombok/ASM/CGLIB/字节码增强类需新版，用 `jdeps --jdk-internals` 扫描内部 API）；② 升级编译目标并修编译错误（`javax.*` → `jakarta.*` 是最大工作量）；③ 替换已移除的启动参数；④ 补 `--add-opens`/`--add-exports`；⑤ GC 与性能复测（G1 默认、`MaxRAMPercentage`）；⑥ 回归测试（序列化兼容、日期时区、字符集、并发行为）；⑦ 灰度发布并保留回滚包。

**常见踩坑清单**：
1. Java EE 模块移除（JDK 11）：`javax.xml.bind`、`javax.annotation`、`javax.activation`、`javax.xml.ws` 需加依赖。
2. 强封装（JDK 16+）：`sun.misc.Unsafe`、内部反射需 `--add-opens`（Netty、Lombok、反射型序列化框架最易踩）。
3. 默认字符集变 UTF-8（JDK 18）：GBK 环境读文件/`new String(bytes)` 会乱码。
4. GC 参数失效：`-Xloggc`、`-XX:+PrintGCDetails`（JDK 9 起移除）、`-XX:+UseConcMarkSweepGC`（JDK 14 起报错）。
5. 偏向锁废弃（JDK 15）：`-XX:+UseBiasedLocking` 参数无效。
6. `SecurityManager` 废弃（JDK 17）：依赖它的老框架需替换。
7. 容器内存识别变化：JDK 8u191+/10+ 按 cgroup 算堆，升级后堆可能"变小"需重算 `-Xmx`。
8. 序列化与顺序兼容：跨版本 RPC 中 `HashMap` 桶内顺序不同、`record` 的序列化形态与普通类不同。

### 10.2 面试问答
**Q1："你项目用哪个 JDK 版本？为什么？"怎么答？** A：按"版本 + 决策依据 + 收益 + 代价 + 后续规划"结构：例如"用 JDK 17（Spring Boot 3 的最低要求），G1 + `-XX:MaxRAMPercentage=70`。选 17 因为它是成熟 LTS 且 record/sealed/模式匹配显著减少 DTO 与领域模型代码；代价是升级时处理了 JAXB 依赖缺失与 `--add-opens` 反射报错。JDK 21 已预研：虚拟线程对订单查询这类 IO 密集接口有效（载体线程数大幅下降），但部分依赖内部用 `synchronized` 阻塞，计划等 JDK 24（JEP 491 修复钉住）后再全面切换。"
**Q2："升级踩过什么坑"怎么答才可信？** A：用"具体现象 + 定位过程 + 解决方案"三段式，例如"升级 JDK 17 后某图片服务启动报 `InaccessibleObjectException`，用 `jdeps`/启动参数逐项收敛到某个 EXIF 库反射读 `java.awt` 内部类；我们换成维护活跃的库并只对必要包加 `--add-opens`，而不是全局 ALL-UNNAMED。"
**Q3：为什么不直接上最新的非 LTS？** A：非 LTS 支持期仅 6 个月，升级成本无法摊薄；生产停留在 LTS（8/11/17/21），非 LTS 仅用于特性预研。
**Q4：`--enable-preview` 能上生产吗？** A：不能，预览特性可能变更或被移除，且 class 文件版本与 JDK 强绑定；仅用于评估。
**Q5：升级 JDK 与升级 Spring Boot 能同时做吗？** A：不建议，两个变量同时改会导致问题难定位；先升 JDK 并回归，再升框架（2.x → 3.x 伴随 `javax` → `jakarta` 全量改名）。

### 10.3 高频追问与陷阱
- 陷阱：把 `--add-opens java.base/ALL-UNNAMED` 当万能药，应精确到必要包（有安全与启动开销，且未来版本会继续收紧）。
- 陷阱：`maven.compiler.release` 与容器 JDK 版本不一致会以 `UnsupportedClassVersionError` 暴露。
- 错误回答："升级 JDK 只要改版本号"。

### 10.4 代码/配置示例
```xml
<properties>
  <maven.compiler.release>17</maven.compiler.release>   <!-- 与目标 JDK 对齐 -->
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>
```
```bash
jdeps --jdk-internals --multi-release 17 -cp "lib/*" target/app.jar   # 升级前扫描内部 API 依赖
```

### 10.5 一句话速记
- 升级清单：依赖支持 → 编译 → 启动参数 → `--add-opens` → 字符集 → GC → 回归 → 灰度。
- 停留在 LTS，预览特性不上生产；回答"用什么版本"要给出决策依据与真实踩坑。

## 自测题（含答案要点）

1. Lambda 编译成什么？会生成额外 class 文件吗？— `invokedynamic` + `LambdaMetafactory`，运行期生成实现，不生成 `$1.class`。
2. Lambda 中 `this` 指向谁？— 外部类实例（匿名内部类中指自身）。
3. Stream 的中间操作与终端操作分别有哪些？为什么 `peek` 看不到输出？— 中间操作惰性，必须有终端操作触发。
4. 并行流默认使用哪个线程池？并行度多少？— `ForkJoinPool.commonPool()`，并行度为 CPU 核数 - 1。
5. `reduce` 三参版本的 combiner 什么时候被调用？— 只在并行流中合并分片结果，串行流不调用。
6. `Collectors.toMap` 遇到重复 key 会怎样？怎么解决？— 抛 `IllegalStateException: Duplicate key`；传 merge 函数。
7. `orElse` 与 `orElseGet` 的区别？— `orElse` 实参总求值，`orElseGet` 只在为空时执行；`orElse(repo.find())` 会多查一次库。
8. `SimpleDateFormat` 与 `DateTimeFormatter` 的线程安全性？— 前者不安全（内部共享 Calendar），后者不可变且安全。
9. `yyyy` 与 `YYYY` 的区别？— `yyyy` 日历年，`YYYY` ISO 周历年份，跨年周会显示错误年份。
10. 接口 `default` 方法解决什么问题？菱形冲突怎么办？— 接口二进制兼容演进；冲突必须显式 `InterfaceA.super.method()`。
11. `var` 不能用在哪些位置？— 字段、方法参数、返回值、`null` 初始化、Lambda 整体表达式、数组字面量。
12. JDK 11 移除 Java EE 模块后的常见报错与解法？— `ClassNotFoundException: javax.xml.bind.*`，需显式引入 `jakarta.xml.bind` 等依赖。
13. record 的 accessor 命名是什么？能继承吗？— `name()`（无 get 前缀）；隐式 final，不可继承也不可被继承。
14. sealed 类的子类必须声明成什么？— `final`、`sealed` 或 `non-sealed` 之一。
15. JDK 14 引入的增强 NPE 提示有什么用？— 精确指出为 null 的变量与方法名（JDK 15 起默认开启）。
16. `Stream.toList()` 与 `Collectors.toList()` 的差别？— JDK 16+ 的 `toList()` 返回不可变 List。
17. JDK 21 正式发布的三个最重要特性？— 虚拟线程、switch 模式匹配（含记录模式）、分代 ZGC；另有 Sequenced Collections。
18. 虚拟线程在 JDK 21 最大的陷阱？— `synchronized` 阻塞会钉住载体线程（JDK 24 JEP 491 修复），应改用 `ReentrantLock`。
19. JDK 18 的 UTF-8 默认字符集会导致什么升级问题？升级 JDK 17 后启动报 `Unrecognized VM option 'PrintGCDetails'`，为什么？— 前者是依赖 GBK 平台默认编码的读写会乱码，需显式指定编码；后者因为 JDK 9 起统一日志框架替代旧参数，应改用 `-Xlog:gc*:file=...`。
20. 场景：面试官问"你们为什么选 JDK 17 而不是 8 或 21"，怎么答？— 按"版本 + 依据（LTS/框架要求/生态）+ 收益（语言特性/GC）+ 代价（升级踩坑）+ 对 21 的预研结论"结构化回答。

## 背诵卡片

1. Lambda 走 invokedynamic，不生成 class 文件；`this` 指外部实例。
2. 捕获局部变量必须 effectively final（值拷贝）。
3. 四大函数式接口：Function / Consumer / Supplier / Predicate。
4. 方法引用四类：静态、实例、类::实例方法、类::new。
5. Stream 中间操作惰性，终端操作触发；垂直执行使 limit 可短路。
6. Stream 只能消费一次，重复消费抛 `IllegalStateException`。
7. `map` 一对一、`flatMap` 一对多并展平。
8. `peek` 仅用于调试，无终端操作不执行。
9. 并行流用 commonPool（CPU-1），IO 与有状态 Lambda 都会出问题。
10. `reduce` 的 combiner 只在并行流调用。
11. `Collectors.groupingBy(k, downstream)` 是分组万能公式；`toMap` 必带 merge 函数。
12. `partitioningBy` 固定两个 key（true/false），`groupingBy` 只包含出现过的组。
13. `orElse` 必求值、`orElseGet` 懒求值；避免 `get()`；Optional 只做返回值。
14. `java.time` 全部不可变线程安全；格式化用 `DateTimeFormatter`。
15. 年份用小写 `yyyy`，`YYYY` 是周历年份。
16. JDK 8 核心：Lambda/Stream、default 方法、元空间、新日期 API、并发与集合重写。
17. `default` 不能覆盖 Object 的方法；接口 static 方法不被继承。
18. JDK 9：JPMS 模块化、集合工厂方法、G1 默认、统一日志（`-Xlog`）、接口私有方法。
19. JDK 10：`var` 局部变量、`copyOf` 系列、G1 Full GC 并行。
20. JDK 11：HttpClient、String 新 API（isBlank/lines/strip/repeat）、单文件运行、ZGC 实验、Epsilon、Java EE 移除。
21. `String.strip()` 支持 Unicode 空白，`trim()` 只处理 ≤ U+0020。
22. JDK 14：switch 表达式正式 + 增强 NPE；JDK 15：文本块正式 + 偏向锁禁用。
23. JDK 16：record + instanceof 模式匹配 + 强封装内部 API。
24. JDK 17 LTS：sealed 正式、Spring Boot 3 的最低要求；switch 模式匹配此时仍只是预览。
25. record 不可变、不可继承、accessor 无 `get` 前缀、不能有实例字段。
26. sealed 子类必须是 final / sealed / non-sealed。
27. `Stream.toList()`（16+）返回不可变 List。
28. JDK 18：默认字符集 UTF-8（JEP 400）—— 升级乱码的根因。
29. JDK 21 正式：虚拟线程、switch 模式匹配、记录模式、Sequenced Collections、分代 ZGC。
30. 虚拟线程不要池化；`synchronized` 会钉住载体线程（JDK 21），JDK 24 才解决。
31. 结构化并发：`StructuredTaskScope` 的 ShutdownOnFailure/ShutdownOnSuccess；ScopedValue 替代部分 ThreadLocal。
32. `reversed()` 是视图不是拷贝。
33. 升级清单：依赖支持 → 编译 → 启动参数 → `--add-opens` → 字符集 → GC → 回归 → 灰度。
34. `-Xloggc`/`PrintGCDetails` 在 JDK 9+ 移除，`UseConcMarkSweepGC` 在 JDK 14+ 报错。
35. 停留在 LTS（8/11/17/21）；预览特性不上生产。
36. 反射访问内部 API 用精确的 `--add-opens`，不要全局 ALL-UNNAMED。
37. `maven.compiler.release` 要与目标 JDK 一致，避免 `UnsupportedClassVersionError`。
