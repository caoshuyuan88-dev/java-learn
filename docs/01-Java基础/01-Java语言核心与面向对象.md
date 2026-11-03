# Java 语言核心与面向对象

> **定位**：钉死语法层最易翻车的细节（String、equals、泛型、反射、代理、拷贝、精度），作为集合、并发、JVM 三篇的地基。
> **重要度**：★★★★★（每场面试的暖场题，答错代价最大）
> **建议复习时长**：12 小时（本篇含大量动手验证，读一遍约 5 小时，跑通所有实验约 7 小时）
> **关联文档**：[复习路线总纲](../00-复习路线总纲.md)、[集合框架](./02-集合框架.md)、[并发编程](./03-并发编程.md)、[JVM 与 GC 调优](./04-JVM与GC调优.md)、[Java 新特性与语言演进](./05-Java新特性与语言演进.md)


**判断标准**：如果你只能说出"✅ 正确答法"的第一句，说不出后面为什么，说明还没懂，回去看 `N.2`。

### 0.3 知识地图

| # | 知识点 | 重要度 | 常考形式 |
| --- | --- | --- | --- |
| 1 | 对象创建过程与内存布局 | ★★★☆☆ | 八股原理 / JVM 交叉 |
| 2 | String 家族与字符串常量池 | ★★★★★ | 八股原理 / 源码追问 |
| 3 | `==`、`equals`、`hashCode` 契约 | ★★★★★ | 八股原理 / 手写代码 |
| 4 | 包装类缓存与自动拆装箱 | ★★★★☆ | 八股原理 / 场景题 |
| 5 | final / static / 初始化顺序 | ★★★★☆ | 场景题 / 手写代码 |
| 6 | 重载 vs 重写、多态与动态绑定 | ★★★★☆ | 八股原理 / 源码追问 |
| 7 | 接口、抽象类与内部类 | ★★★☆☆ | 八股原理 / 场景题 |
| 8 | 泛型：擦除、PECS、桥接方法 | ★★★★★ | 源码追问 / 手写代码 |
| 9 | 异常体系与 try-with-resources | ★★★★☆ | 八股原理 / 场景题 |
| 10 | 反射、注解与动态代理 | ★★★★★ | 八股原理 / 场景题 / 框架原理 |
| 11 | Lambda、函数式接口与序列化 | ★★★★☆ | 八股原理 / 场景题 |
| 12 | 值传递、深浅拷贝、BigDecimal、位运算 | ★★★★★ | 手写代码 / 八股原理 / 踩坑题 |

---

## 1. 对象创建：`new` 到底做了什么

### 1.1 先建立直觉

**先用一句话说本质**：`new User("Tom")` 这行代码，不是"创建一个对象"这么简单的一件事，它其实是**五件独立的事按固定顺序拼起来的**：

1. 把 `User` 这个类"准备好"（如果还没准备好）
2. 在堆上**划分**一块足够大的内存
3. 把这块内存**全部清零**（保证不读到垃圾值）
4. 往内存头部写**管理信息**（对象头，GC 和锁要靠它）
5. 执行**构造器**，把你写 `new User("Tom")` 时想做的事做掉

理解了这一点，就能回答一类经典问题：**"构造器抛异常了，对象存在吗？"** —— 内存已经分配了（第 2 步完成），但对象没有构造成功，引用不会被赋给变量，那块内存最终由 GC 回收。

### 1.2 原理与源码

#### 1.2.1 从一行 Java 代码到字节码

```java
User user = new User("Tom");
```

`javap -c` 反编译出来是这样（常量池编号随代码而变）：

```text
0: new           #2    // class User     ← 分配内存，把"未初始化"的引用压栈
3: dup                              // 复制一份引用：一份给构造器用，一份留着赋值
4: ldc           #3    // String Tom     ← 参数入栈
6: invokespecial #4    // User.<init>    ← 调用构造器（消费掉一份引用）
9: astore_1                         // 把剩下的引用存进局部变量 user
```

**这几条指令要读懂三件事**：

1. **`new` 和 `invokespecial <init>` 是两条指令**，这直接证明了"分配内存"和"执行构造器"是两个阶段。所以构造器里 `this` 已经能用了（内存已分配、字段已清零），但对象还没"构造完"。
2. **`dup` 的存在**：因为 `invokespecial` 会消耗掉栈顶的引用，如果不复制一份，`astore_1` 就没东西可存了。
3. **为什么是 `invokespecial` 而不是 `invokevirtual`**：构造器不能被重写，也不需要动态分派，所以用"静态绑定"的指令。

#### 1.2.2 内存分配：两种策略 + 一个加速器

**策略一：指针碰撞（Bump the Pointer）**
前提是**内存绝对规整**（用过的在一侧，空闲的在一侧），那只需要维护一个指针，分配就是"指针往后挪 N 个字节"。

```
[ 已分配 ][ 已分配 ][ 空闲指针 ↓ ][      空闲      ]
                                  ↑ 分配时指针往右移动对象大小
```

**策略二：空闲列表（Free List）**
如果内存**不规整**（有碎片，比如 CMS 这种标记-清除收集器），指针碰撞就用不了，JVM 必须额外维护一张"哪些块是空的、有多大"的表，分配时去表里找一块够大的。

| 策略 | 前提 | 典型场景 | 代价 |
| --- | --- | --- | --- |
| 指针碰撞 | 内存规整 | 复制算法（新生代 Survivor）、标记-整理 | 极快，一次加法 |
| 空闲列表 | 内存有碎片 | 标记-清除（CMS 老年代） | 需要查表匹配 |

**加速器：TLAB（Thread Local Allocation Buffer）**
问题来了：如果每秒创建几百万个对象，所有线程都去抢同一个分配指针，那这个指针就是**全局热点**，每次分配都要 CAS，性能直接崩。

解法是**每个线程先批发一小块内存**（TLAB，默认在 Eden 区内），线程自己在这块里做指针碰撞，**完全不需要同步**。用完了再去找 JVM 批发一块新的。

```text
Eden 区
┌──────────┬──────────┬──────────┬─────────────┐
│ 线程A TLAB│ 线程B TLAB│ 线程C TLAB│   剩余共享   │
└──────────┴──────────┴──────────┴─────────────┘
   ↑ 各自维护自己的指针，互不干扰（这是分配速度能做到接近 malloc-free 的原因）
```

相关参数：`-XX:+UseTLAB`（默认开）、`-XX:TLABSize`、`-XX:+PrintTLAB`（观察浪费率）。

**大对象不走 TLAB**：对象太大 TLAB 放不下时，会退回到共享区分配（需要同步）。

#### 1.2.3 内存清零：为什么字段一定有默认值

分配完内存后，JVM 必须把这块内存**全部置零**。这不是"顺手做的事"，而是 Java 语言的安全保证：

```java
class Demo {
    int a;
    boolean b;
    Object c;
    // 没有任何初始化，直接打印
    void print() { System.out.println(a + "," + b + "," + c); } // 0,false,null
}
```

如果没有清零，`a` 就会是这块内存上一任主人（可能是别的对象、可能是已释放的敏感数据）留下的垃圾值。**这就是"C/C++ 里未初始化变量值随机、Java 里不会"的根本原因。**

#### 1.2.4 对象头：GC 和 synchronized 的记账本

对象在内存里长这样：

```text
┌──────────────────────────────┐
│   Mark Word（对象头-运行时数据）│  8 字节，存 hash、GC 年龄、锁状态
├──────────────────────────────┤
│   Klass Pointer（类型指针）    │  4 字节（开启指针压缩）→ 指向 User.class
├──────────────────────────────┤
│   实例数据（你自己的字段）      │  按字段类型排列，long/double 在前以对齐
├──────────────────────────────┤
│   Padding（对齐填充）          │  补齐到 8 字节整数倍
└──────────────────────────────┘
```

几个必须记住的数字：

- 64 位 JVM 上 Mark Word 是 **8 字节**
- 开启指针压缩（`-XX:+UseCompressedOops`，默认开、堆小于 32G 时生效）时 Klass Pointer 是 **4 字节**，否则 8 字节
- 所以一个**没有任何字段的空对象**占 **16 字节**（8 + 4 + 4 字节对齐填充）

Mark Word 里最关键的几位（和并发那篇的锁升级直接对应）：

| 锁状态 | Mark Word 内容（64 位） |
| --- | --- |
| 无锁 | unused(25) \| hashCode(31) \| unused(1) \| 分代年龄(4) \| 偏向锁位0(1) \| 锁标志位 01 |
| 偏向锁 | 线程ID(54) \| Epoch(2) \| unused(1) \| 分代年龄(4) \| 1 \| 01 |
| 轻量级锁 | 指向栈中 Lock Record 的指针(62) \| 00 |
| 重量级锁 | 指向 Monitor 的指针(62) \| 10 |
| GC 标记 | 空 \| 11 |

> 这张表在面试里能救你一命：面试官问"对象头里有什么"，你能说出"hashCode、GC 分代年龄、锁标志位、偏向线程 ID"，就比只会说"对象头"的人强一档。细节展开见 [并发编程](./03-并发编程.md) 的锁升级一节。

#### 1.2.5 「默认值」和「显式初始化」是两回事（最容易搞混的点）

很多人以为"字段初始化在 `new` 的时候一起做完了"，实际是**两个不同时刻**：

```java
class Account {
    private int balance = 100;   // 显式初始化：编译器把它塞进构造器，在 super() 之后执行
    private int overdraftLimit;  // 没有显式初始化：保持零值 0

    {
        balance += 50;           // 实例初始化块：也在构造器里，按源码顺序排在字段初始化后
    }

    Account() {
        System.out.println(balance);        // 150
        System.out.println(overdraftLimit); // 0
    }
}
```

编译器实际生成的构造器大致等价于：

```java
Account() {
    super();            // ① 先调父类构造器
    this.balance = 100; // ② 字段显式初始化（按源码顺序）
    this.balance += 50; // ③ 实例初始化块（按源码顺序，和②交替执行）
    // ④ 最后才是你写的构造器方法体
    System.out.println(balance);
    System.out.println(overdraftLimit);
}
```

**所以完整的对象创建顺序是（背下来，这是必考题）**：

```text
父类静态字段/静态块 → 子类静态字段/静态块        （只在类初始化时执行一次）
  → 分配内存 + 全部字段置零
  → super() 递归完成父类的「字段初始化 → 初始化块 → 构造器体」
  → 本类「字段显式初始化 → 实例初始化块」（按源码顺序）交替执行
  → 本类构造器方法体
```

#### 1.2.6 一个经典陷阱：构造器里调用可重写方法

```java
class Parent {
    Parent() {
        init();   // 编译不报错，运行有坑
    }
    void init() { System.out.println("parent init"); }
}

class Child extends Parent {
    private int value = 10;   // 注意：这一行此时还没执行

    @Override void init() {
        System.out.println("child init, value=" + value);  // 输出 value=0，不是 10！
    }
}
```

**为什么是 0？** 因为 `Child` 的构造器流程是：`super()` → 字段初始化。而 `Parent()` 里的 `init()` 是在 `super()` 内部被调用的，此时**子类字段初始化还没跑到**，`value` 还是零值。

> 这就是 Effective Java 里"**构造器里不要调用可被重写的方法**"的由来。要修复：把 `init` 改成 `final` 或 `private`，或者改用工厂方法延迟初始化。

### 1.3 面试标准问答

**Q1：`new` 一个对象，JVM 做了哪些事？**
A：五步——①检查类是否已加载/初始化，没有就先执行类初始化（父类优先，JVM 保证只执行一次且线程安全）；②在堆上按对象大小分配内存（优先走 TLAB 指针碰撞，大对象或因碎片走共享区/空闲列表）；③把内存全部置零；④设置对象头（Mark Word + 类型指针）；⑤执行构造器（先 `super()`，再字段初始化、初始化块、构造器体）。全部成功后才把引用赋值给变量。

**Q2：对象一定分配在堆上吗？**
A：语言规范只保证"对象的生命周期由 GC 管理"，具体位置由实现决定。HotSpot 上绝大多数对象在堆，但有两种逃逸：
①**逃逸分析 + 标量替换**：对象没逃出方法、且能被拆成标量时，可能压根不创建对象，字段直接放栈上（`-XX:+DoEscapeAnalysis` 默认开）；
②**TLAB**：对象仍在堆里，只是分配动作在 Eden 内的一块线程私有区域完成。
另外，小对象在分配时确实可能先在寄存器/栈上的临时位置存在一瞬，但**不是语言层面的"栈上对象"**，Java 没有 C++ 那种栈对象。

**Q3：分配内存有两种方式，分别什么时候用？**
A：指针碰撞用于内存规整的场景（复制算法、标记-整理后的新生代），只需移动指针；空闲列表用于内存有碎片的场景（如 CMS 的老年代），需要维护空闲块表并查找。选择哪个取决于 GC 算法，不是 JVM 参数决定的。

**Q4：为什么需要 TLAB？**
A：多线程同时分配对象会竞争同一块内存的分配指针，每次分配都要 CAS/加锁，吞吐会崩。TLAB 让每个线程在 Eden 里预占一小块私有空间，在自己的指针上做指针碰撞，**无需同步**，用完了再批发。这是 JVM 分配对象几乎无锁开销的关键。

**Q5：对象头里有什么？为什么要有它？**
A：Mark Word（8 字节）：存 identity hashCode、GC 分代年龄、锁状态标志、偏向线程 ID——**是 synchronized 锁升级和 GC 判定晋升年限的物理载体**；类型指针（4 或 8 字节）：指向方法区的 Class 对象，`obj.getClass()` 和 `instanceof` 都靠它。数组对象还额外有 4 字节记录长度。

**Q6：构造器抛异常，对象会被回收吗？引用变量是什么状态？**
A：分配的内存已经在堆上了，但 `new` 表达式没有正常完成，所以**引用不会被赋值**（如果你写成 `User u = new User()` 且抛异常，`u` 根本轮不到赋值）。那块内存上没有可达引用，下一次 GC 会被回收。注意：**如果构造器在抛异常前把 `this` 泄露给了外部**（比如把自己注册到静态列表、提交给线程池），那对象就活了，而且它是个半初始化状态的"僵尸对象"——这是要避免的写法。

### 1.4 高频追问与陷阱

#### 追问 1：构造器和字段初始化谁先执行？

**【追问场景】**
通常出现在面试官给你一段有静态块、实例块、字段、构造器的代码让你说输出顺序时。他在试探你是不是只会背"静态块最先"，而不知道实例部分的确切顺序。

**【❌ 错误答法】**
> "先执行构造器，然后才初始化字段。"
> 错在哪：完全颠倒。构造器里的代码是**最后**执行的，字段初始化表达式和实例初始化块会被编译器**搬进构造器内部**，排在 `super()` 之后、构造器方法体之前。

**【✅ 正确答法】**
> 单个类内部，`new` 时的顺序是：先分配内存并把所有字段置零，然后调 `super()`，接着按源码顺序执行**字段显式初始化表达式**和**实例初始化块**（这两者谁写在前面谁先执行），最后才是构造器方法体。
> 如果有继承，父类这一整套（super → 字段 → 初始化块 → 构造器体）会先走完，再走子类的。
> 而 `static` 的东西（静态字段、静态块）属于**类初始化**，在第一次使用这个类时执行，早于任何对象创建，且只执行一次。

#### 追问 2：为什么构造器里调用可重写方法会出问题？

**【追问场景】**
面试官想看你有没有踩过这个坑，或者想考察你对"父类构造器先于子类字段初始化"这条顺序的理解深度。常包装成"这段代码输出什么，为什么"。

**【❌ 错误答法】**
> "输出的 value 是 10，因为子类字段赋了值。"
> 错在哪：忽略了执行时机。父类构造器执行时子类字段初始化还没跑，此时读到的必然是零值 0。

**【✅ 正确答法】**
> 父类构造器调用被子类重写的方法时，方法体会被动态分派到子类版本，但子类的字段初始化还没执行，所以子类版本里读到的字段是**默认零值**。
> 这就是 Effective Java 建议"构造器里不要调用可被重写的方法"的原因——别的线程/逻辑可能通过这个方法拿到一个**半初始化**的对象。
> 修复方式：把方法改成 `private` 或 `final`（消除重写），或者把初始化延后到工厂方法 / `@PostConstruct` 里。

#### 追问 3：对象一定在堆上分配吗？

**【追问场景】**
考察你有没有听过逃逸分析和标量替换，是区分"背八股"和"读过 JVM 文档"的分水岭问题。

**【❌ 错误答法】**
> "不一定，方法内的对象会分配在栈上。"
> 错在哪：这个说法不严谨。**Java 语言层面没有栈上对象**（和 C++ 不同，不会因为方法返回自动析构）。准确的说法是：逃逸分析后，对象可能被**拆成标量**放到栈上/寄存器里，也就是"对象压根没被创建"，而不是"对象创建在栈上"。

**【✅ 正确答法】**
> 规范上不限定，实现上 HotSpot 绝大多数对象在堆。但有两种情况要注意：
> 一是**逃逸分析**（默认开启）：如果对象只在方法内使用、没有被外部引用（不逃逸），JIT 可能做**标量替换**，直接把它的字段拆成局部变量，连对象都不分配；可能顺带做**锁消除**。
> 二是 **TLAB**：分配动作是在 Eden 内线程私有区完成，但对象依然在堆上。
> 所以准确表述是：**对象在堆上，但"创建对象"这个动作可能被优化掉**。

#### 追问 4：`int a;` 不赋值，为什么不会像 C 一样读到随机值？

**【追问场景】**
考的是"内存清零"这一步。很多人只知道"Java 有默认值"，不知道默认值是**谁、在什么时候**给的。

**【❌ 错误答法】**
> "Java 语言规定字段有默认值，所以不会随机。"
> 错在哪：这是在复述结论，没有机制。为什么语言规定的就一定成立？因为 JVM 在分配后**强制清零**了。

**【✅ 正确答法】**
> 因为对象分配完成后、任何 Java 代码执行前，JVM 会把整块内存**置零**（这一步在字节码里看不到，是 JVM 实现行为）。所以字段一定是 0 / false / null。
> 这个设计的目的是**安全**：否则对象可能读到同一块内存上一次被别的对象写过的残留数据（可能是密码、密钥）。
> 注意区分：**局部变量没有这个待遇**——局部变量不会自动清零，所以用之前必须赋值，否则编译不通过。这也说明"清零"是对象分配路径特有的。

#### 追问 5：一个 `new` 出来的空对象占多少字节？

**【追问场景】**
面试官想确认你是否知道对象头、指针压缩、8 字节对齐。一般接着会问"加一个 `int` 字段呢""加一个 `long` 呢"。

**【❌ 错误答法】**
> "空对象不占空间"或者"不知道"。
> 错在哪：任何对象都有对象头，最少也是 16 字节。

**【✅ 正确答法】**
> 64 位 JVM、开启指针压缩（默认）时：Mark Word 8 字节 + Klass Pointer 4 字节 = 12 字节，再补齐到 8 的整数倍，所以**空对象 16 字节**。
> 加一个 `int` 字段：12 + 4 = 16，刚好不用补，还是 16 字节。
> 加一个 `long` 字段：12 + 8 = 20，补齐到 24 字节。
> 可以用 JOL（`org.openjdk.jol:jol-core`）的 `ClassLayout.parseInstance(obj).toPrintable()` 实测验证。

### 1.5 动手验证

```java
import org.openjdk.jol.info.ClassLayout;
import org.openjdk.jol.info.GraphLayout;

public class ObjSizeDemo {
    static class Empty {}
    static class WithInt { int a; }
    static class WithLong { long a; }
    static class WithIntAndLong { int a; long b; }

    public static void main(String[] args) {
        System.out.println("空对象:\n" + ClassLayout.parseInstance(new Empty()).toPrintable());
        System.out.println("含 int:\n" + ClassLayout.parseInstance(new WithInt()).toPrintable());
        System.out.println("含 long:\n" + ClassLayout.parseInstance(new WithLong()).toPrintable());
        // 输出里着重看 Instance size 一行，以及 (object header: 12 bytes)
    }
}
```

pom 依赖：

```xml
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.17</version>
</dependency>
```

初始化顺序验证（不看文档，先自己想输出，再跑）：

```java
public class InitOrderDemo {
    static { System.out.println("1. 父类/本类静态块"); }
    private int a = print("2. 字段显式初始化 a");
    { System.out.println("3. 实例初始化块"); }
    private int b = print("4. 字段显式初始化 b");

    InitOrderDemo() {
        super();
        System.out.println("5. 构造器方法体");
    }

    static int print(String s) { System.out.println(s); return 0; }

    public static void main(String[] args) {
        new InitOrderDemo();   // 注意：main 里 new 之前，静态块已经执行了
    }
}
```

> 预期输出顺序：`1 → 2 → 3 → 4 → 5`。如果你预测错了，那说明 1.2.5 那段还要再看一遍。

### 1.6 一句话速记

- `new` = 类初始化 → 分配（TLAB）→ 清零 → 写对象头 → 执行构造器，**分配和构造是两个阶段**。
- 空对象 **16 字节**（8 字节 Mark Word + 4 字节指针 + 4 字节对齐）。
- 实例部分顺序：**super() → 字段初始化 → 初始化块 → 构造器体**；`static` 属于类初始化，一次且最早。
- 构造器里别调可重写方法：那时子类字段还是零值。

---

## 2. String：为什么不可变、常量池到底在哪

### 2.1 先建立直觉

**一句话本质**：`String` 是 Java 里**唯一被设计成"共享安全"的引用类型**。为了让你能放心地把它到处传、当 Map 的 key、当类名，它做了两个激进的取舍：**不可变** + **常量池**。

**先说不可变**。想象你有一张写着"北京银行"的**门牌**贴在门口，很多部门（文件名、URL、类名、SQL 表名、Map 的 key）都依赖这块门牌。如果某个人能偷偷把牌子改成"北京银很"，那所有依赖它的地方都会出错——这就是可变字符串（`StringBuilder`）不能随便共享的原因。

所以 String 的做法是：**造出来之后一个字都动不了**。想改？给你换一块新牌子。

这个"不能动"带来四件好事：

1. **可以随意共享**：一万个地方用 "abc"，内存里只有一份（常量池）
2. **hashCode 能缓存**：不可变意味着 hash 永远不变，算一次存起来即可——这就是 `String` 里那个 `private int hash;` 字段的意义
3. **线程安全**：多个线程读同一个字符串，永远不会有竞态
4. **可以安全地做 Map 的 key**：如果 key 能被改，`map.get(k)` 就会找不到自己放进去的东西

**再说常量池**。既然 String 用得极多、而且大多重复（"UTF-8"、"/index"、"success"），那就干脆做一个**全局池子**：字面量第一次出现时放进池子，后面都指向同一个。

**类比**：常量池像一个"共享字典"，`new String` 像"复印一份"。

```java
String a = "abc";              // 查字典，没有就建一条，a 指向字典里那条
String b = "abc";              // 查字典，有 → b 和 a 指向同一条
String c = new String("abc");  // 强制复印一份放在外面（池里那条照样存在）
```

所以内存长这样：

```text
       ┌────────────────┐
"abc"  │  常量池 StringTable │  ← 唯一一份，a、b 都指向它
       └────────────────┘
                ▲    ▲
                │    │
                a    b

       ┌────────────────┐
       │  堆上新对象      │  ← new 出来的副本，内容和池里一样但是另一个对象
       └────────────────┘
                ▲
                c
```

这就是 `a == b` 是 `true`、`a == c` 是 `false` 的原因：**`==` 比的是"是不是同一条"，不是"内容一不一样"**。

### 2.2 原理与源码

#### 2.2.1 JDK 8 的 String 定义

```java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    /** 字符存储：final 意味着引用不可变，且数组内容由 String 自身保证不被修改 */
    private final char value[];

    /** 缓存 hash。0 表示"还没算过"（注意 String 的 hash 实际不会是 0 以外的歧义，除非内容为 \u0000） */
    private int hash;

    ...
}
```

**要点**：

- `final class` → 不能被继承，防止子类加可变行为破坏约定
- `private final char[] value` → 数组引用不可改；数组本身在 Java 里**没有** const 语义，但因为它是 `private` 且 String 内部从不修改它，所以外部无法破坏内容

#### 2.2.2 一个常见的追问点：`final` 加在数组上真的够吗？

不够，严格说靠的是"**private + 不对外暴露 + 内部不修改**"三件事。看这段：

```java
public final class BadString {
    private final char[] value;
    public BadString(char[] value) { this.value = value; }   // 直接把外部数组存进来
    public char[] getValue() { return value; }               // 又把内部数组暴露出去

    public static void main(String[] args) {
        char[] arr = {'a', 'b', 'c'};
        BadString s = new BadString(arr);
        arr[0] = 'X';                     // 外部把内容改了
        s.getValue()[1] = 'Y';            // 内部数组也被外部改了
        System.out.println(s.value);      // XYc —— "不可变"被打破
    }
}
```

真正的 String 构造器是**拷贝**数组的：

```java
// JDK 8
public String(char value[]) {
    this.value = Arrays.copyOf(value, value.length);   // 关键：防御性拷贝
}
```

JDK 9 之后同样的逻辑变成 `this.value = Arrays.copyOf(value, value.length, byte[].class)` 之类的形式。

> 这条能给面试官留下"读过源码"的印象，比只会说"因为 value 是 final"强得多。

#### 2.2.3 JDK 9+ 的紧凑字符串（Compact Strings）

```java
public final class String ... {
    @Stable
    private final byte[] value;   // 改成 byte[]
    private final byte coder;     // 0 = LATIN1（每字符 1 字节），1 = UTF16（每字符 2 字节）
    private int hash;
}
```

**动机**：绝大部分字符串是纯 ASCII（拉丁字母、数字、符号），用 `char[]` 每个字符都要占 2 字节，浪费一半内存和缓存。改成 `byte[] + coder` 后，纯 ASCII 字符串占用减半，`coder == LATIN1` 时直接按字节处理。

**代价**：所有字符串操作要按 coder 分支处理，所以 JDK 9 里 `StringLatin1`、`StringUTF16` 这两个工具类出现了。

> **面试话术**：问"JDK 9 对 String 做了什么优化"，答"紧凑字符串，从 char[] 改成 byte[] + coder，LATIN1 编码的字符串省一半内存"。能补一句"用 `-XX:-CompactStrings` 可关闭"就是加分项。

#### 2.2.4 `String.equals` 源码

```java
// JDK 8 实现
public boolean equals(Object anObject) {
    if (this == anObject) {              // ① 同一引用，最快的捷径
        return true;
    }
    if (anObject instanceof String) {    // ② 类型检查
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {   // ③ 长度不同直接 false，避免全量比较
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {           // ④ 逐字符比较
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
```

**三个可讲点**（比"比较内容"这句话值钱）：

1. **先比引用**：同一个对象直接返回，性能捷径
2. **先比长度**：长度都不同，逐字符比较是浪费
3. **JDK 9+ 多一个 coder 判断**：`this.coder == aString.coder` 时可以直接按 byte 比较（Latin1 直接 `StringLatin1.equals`），不同编码时走 UTF16 路径——**这是紧凑字符串带来的分支**

#### 2.2.5 `String.hashCode` 源码（31 的由来）

```java
public int hashCode() {
    int h = hash;                       // 读缓存
    if (h == 0 && value.length > 0) {   // 未计算过才计算
        char val[] = value;
        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];        // 多项式滚动哈希：h = 31*h + c[i]
        }
        hash = h;                       // 缓存到字段（不可变才能这么干！）
    }
    return h;
}
```

**为什么乘 31？**

| 原因 | 说明 |
| --- | --- |
| 是奇素数 | 减少冲突；用偶数会导致信息丢失（如 2 会让低位全变 0） |
| `31 * i` 可优化 | JIT 编译成 `(i << 5) - i`，移位+减法比乘法快 |
| 实测效果 | 在英文字符串上冲突率低，是 JDK 早期实测选出的经验值（也有"31 曾在某篇论文中被推荐"的说法） |

**注意**：这里的 31 和 `Objects.hash` 里的 31 是同一个设计习惯，面试常一起问。

#### 2.2.6 字符串拼接：三种情况，三种结果

```java
// 情况一：全是字面量 → 编译期"常量折叠"，直接变成一个常量
String s1 = "a" + "b" + "c";        // 编译后等价于 String s1 = "abc";

// 情况二：有变量参与 → 运行期拼接
String a = "a";
String s2 = a + "bc";               // JDK 8：new StringBuilder().append(a).append("bc").toString()

// 情况三：final 修饰的编译期常量 → 仍然折叠
final String f = "a";
String s3 = f + "bc";               // 编译器认为 f 是常量 → 折叠成 "abc"
```

**JDK 8 与 JDK 9+ 的差别**（这是个高频追问点）：

```java
// JDK 8 编译器生成：
new StringBuilder().append(a).append("bc").toString()

// JDK 9+ 改用 invokedynamic + StringConcatFactory：
invokedynamic #0:makeConcatWithConstants:(Ljava/lang/String;)Ljava/lang/String;
```

JDK 9+ 的好处：不再强制创建中间 `StringBuilder` 对象，由运行时（`StringConcatFactory`）选择最优策略，还能配合 `-XX:+UseCompressedStrings` 之类的优化。

**但结论不变**：**在循环里用 `+` 拼接依然可能退化成大量临时对象**（尤其 JDK 8），必须换成 `StringBuilder` 并指定初始容量。

#### 2.2.7 常量池到底在哪

| 版本 | 字符串常量池位置 | 说明 |
| --- | --- | --- |
| JDK 6 及以前 | **永久代（PermGen）** | 方法区的一部分，容量固定，容易 OOM: PermGen space |
| JDK 7 | **移到堆** | 永久代仍在，但常量池搬到了堆。原因：永久代太小、GC 效率低、`intern` 场景下容易撑爆 |
| JDK 8+ | **在堆**（永久代被元空间取代） | 元空间只放类的元数据，字符串常量池仍在堆里 |

**这是最经典的版本陷阱题**：问"字符串常量池在哪"，答"元空间"就错了。

底层数据结构：`StringTable` 是一个**固定桶数的哈希表**（JDK 8 默认 60013 个桶），可用 `-XX:StringTableSize=` 调整。`intern()` 就是往这个表里塞/查。

#### 2.2.8 `intern()` 的语义与版本差异

```java
// 语义：返回池中与当前字符串内容相等的对象引用；池里没有就把当前对象放进去
public native String intern();
```

```java
String s = new String("abc");     // 堆上一个新对象，池里有 "abc"
String i = s.intern();
System.out.println(i == "abc");   // true —— intern 返回的是池里那条
System.out.println(i == s);       // false —— s 是堆上的副本

String t = new String("xyz");     // 池里原本没有 "xyz"
System.out.println(t.intern() == t);   // JDK 7+ → true；JDK 6 → false
```

**为什么 JDK 6 是 false、JDK 7+ 是 true？**

| 版本 | `intern()` 池中无此字符串时的行为 |
| --- | --- |
| JDK 6 | 把字符串**复制一份**到永久代的池中，返回池里副本 → 与 `t` 不是同一个对象 → false |
| JDK 7+ | 直接把**堆上这个对象的引用**登记进池中，返回就是 `t` 本身 → true |

> 这个差异是经典面试题，能答出"JDK 6 复制、JDK 7 起存引用"就是高分回答。背后的原因是**常量池从永久代搬到了堆**（都在堆里就不需要复制了）。

### 2.3 面试标准问答

**Q1：String 为什么设计成不可变？**
A：四个理由——①可安全共享，常量池才有意义（否则一处修改全局错乱）；②hashCode 可缓存（`private int hash` 字段），作为 HashMap key 高效；③线程安全，无竞态；④安全性：类名、URL、JDBC 连接串、文件路径作为参数传递时不会被中途篡改。

**Q2：`String s = new String("abc")` 创建了几个对象？**
A：**1 个或 2 个**。如果池里还没有 "abc"，则池里创建 1 个 + 堆上 new 1 个 = 2 个；如果池里已有（比如同一方法前面出现过字面量 "abc"），则只 new 1 个。**注意 `new` 一定会在堆上创建对象**，"可能复用池对象"这个说法是错的。

**Q3：`String s = "a" + "b" + "c"` 创建几个对象？**
A：**0 个**（运行时）。因为全是编译期常量，编译器直接折叠成 `"abc"` 一个常量，运行期就是去常量池查一次。

**Q4：`StringBuilder` 和 `StringBuffer` 的区别？怎么选？**
A：`StringBuffer` 的所有方法都加了 `synchronized`，线程安全但有锁开销；`StringBuilder` 不加锁，单线程快。二者都继承 `AbstractStringBuilder`（内部也是 `char[]`/`byte[]` + `count`），默认容量 16，扩容规则是 `newCapacity = oldCapacity * 2 + 2`，然后 `Arrays.copyOf`。**结论：单线程用 StringBuilder（并预估容量避免多次扩容），只有多个线程共享同一个 builder 才需要 StringBuffer（实际这种场景极少，通常是设计问题）。**

**Q5：`intern()` 有什么用？什么时候用？**
A：把字符串收敛到常量池，减少重复对象的内存占用。典型场景：**大量重复的枚举值/状态码/配置 key 从外部读入后做 intern**，可以让后续的 `==` 比较成立、并节省内存。但注意 `intern` 有代价（查表 + 可能的池膨胀），且**不要用它来做"性能优化"的默认手段**。

**Q6：为什么 String 要被 `final` 修饰？**
A：如果允许继承，子类可以重写 `equals`、`hashCode`、`length` 等，就能构造出"看起来是 String 但行为不同"的对象，破坏常量池共享、破坏所有依赖 String 不变性的地方（如 `HashMap` 的 key、类加载器用类名字符串做缓存）。`final` 是最低成本的防护。

### 2.4 高频追问与陷阱

#### 追问 1：`String s = new String("abc")` 到底创建了几个对象？

**【追问场景】**
热身题，几乎必问。面试官在看你知不知道"常量池"和"堆"的区别，以及会不会被"1 个"这种直觉答案带跑。

**【❌ 错误答法】**
> "1 个，就是 new 出来的那个。"
> 错在哪：忽略了字符串字面量 `"abc"` 本身也要在常量池里创建（首次出现时）。漏掉这一步，说明没建立"字面量走池、new 走堆"的心智模型。

**【✅ 正确答法】**
> 分情况：如果常量池里还没有 "abc"（比如这是程序里第一次出现这个字面量），则是 **2 个**——池里一个常量、堆上一个 new 出来的对象；如果池里已经有 "abc" 了，则只有 **1 个**，即堆上 new 的那个。
> 无论哪种情况，`new` 都一定在堆上产生新对象，绝不会复用池里的对象，所以 `s == "abc"` 一定是 `false`。

#### 追问 2：`intern()` 在 JDK 6 和 JDK 7 有什么区别？

**【追问场景】**
这是区分"背过八股"和"读过源码/版本变更"的题。面试官通常先问 intern 是什么，答完再补这刀。

**【❌ 错误答法】**
> "intern 就是把字符串放进常量池，各版本一样。"
> 错在哪：漏掉了最关键的版本差异，而这个差异恰好是这道题的全部价值。

**【✅ 正确答法】**
> 差异在"池中没有这个字符串时怎么办"：
> JDK 6 会把字符串**复制一份**放进永久代的常量池，返回的是副本，所以 `new String("xyz").intern() == new String("xyz")` 的对比结果是 false。
> JDK 7 起常量池搬到了**堆**，此时不需要复制，直接把**堆上这个对象的引用**登记进池子，所以 `s.intern() == s` 为 true。
> 这个变化的根本原因是**常量池从永久代迁移到堆**——既然都在堆里，复制就多余了。

#### 追问 3：字符串常量池在堆还是元空间？

**【追问场景】**
经典陷阱题，专门抓"记混了永久代和元空间"的人。往往会连着问"那永久代和元空间有什么区别"。

**【❌ 错误答法】**
> "在元空间里，因为 JDK 8 把永久代替换成了元空间。"
> 错在哪：混淆了两件事。元空间取代的是**永久代（方法区实现）**，放的是**类的元数据**；而字符串常量池**在 JDK 7 就已经搬到堆里了**，和元空间没关系。

**【✅ 正确答法】**
> 字符串常量池 **JDK 7 起就在堆里**（JDK 6 及以前在永久代）。
> JDK 8 用元空间取代永久代，改变的是**方法区的实现**（从堆内的永久代变成本地内存里的元空间），影响的是类元数据、而不是字符串常量池。
> 所以准确说：JDK 7 之后的常量池在**堆**，可以用 `-XX:StringTableSize` 调桶数；`String.intern` 操作的就是这张堆内的哈希表。

#### 追问 4：为什么 `new String("abc") == "abc"` 是 false，但 `equals` 是 true？

**【追问场景】**
考 `==` 与 `equals` 在 String 上的语义差异，是第 3 个知识点（equals 契约）的引子。

**【❌ 错误答法】**
> "因为 new 是新的对象，== 比较对象地址，所以 false；equals 比较内容，所以 true。"——**这个回答本身是对的**，但如果只说到这里就停了，会被追问"那 `==` 比较引用为什么"abc"这个字面量能拿到同一个？"
> 真正容易错的是下一句："既然值一样，JVM 为什么不直接复用池里的对象？"

**【✅ 正确答法】**
> `==` 比较的是**引用值**（可以理解为对象地址），`new` 强制在堆上分配新对象，所以和池里那条不是同一个 → false。
> `equals` 被 String 重写成**逐字符比较内容**（先比引用、再比长度、再逐位比），所以 true。
> 至于"为什么不复用池对象"：`new` 的语义就是"给我一个独立的、可区分的实例"，如果它复用池对象，那 `new` 出来的两个 String 就无法区分了，`synchronized(s)` 这类依赖对象身份的用法会出错。

#### 追问 5：循环里用 `+` 拼接字符串有什么问题？

**【追问场景】**
场景题包装成"这段代码有问题吗"。面试官想看你能不能一眼看出"循环内的字符串拼接会创建大量中间对象"。

**【❌ 错误答法】**
> "没问题，Java 编译器会自动优化成 StringBuilder。"
> 错在哪：**优化只发生在"单个表达式内"**。`s += i` 在循环体内是一个独立表达式，每轮都会 `new` 一个 StringBuilder，循环结束后丢弃——优化失效。

**【✅ 正确答法】**
> 问题在于编译器把每轮循环里的 `s += i` 各自编译成"新建 StringBuilder → append → toString"，所以 N 次循环会产生 N 个 StringBuilder 和 N 个中间 String，全是垃圾。
> 正确写法是**把 StringBuilder 提到循环外面**，复用同一个：
> ```java
> StringBuilder sb = new StringBuilder(1024);   // 预估容量，避免扩容拷贝
> for (int i = 0; i < n; i++) sb.append(i);
> String s = sb.toString();
> ```
> 补充：JDK 9+ 用 `invokedynamic` 优化了单个拼接表达式，但**循环内的 += 依然是每轮新建**，所以这个结论跨版本成立。

#### 追问 6：`switch` 支持 String 是怎么实现的？

**【追问场景】**
考察"语法糖"的识别能力。答"JVM 原生支持"就错了。

**【❌ 错误答法】**
> "JVM 的 switch 指令支持 String。"
> 错在哪：JVM 的 `tableswitch`/`lookupswitch` 只支持 int（含 byte/char/short，枚举是编译器转成 int 的序号）。String 版本是纯语法糖。

**【✅ 正确答法】**
> 是编译器生成的语法糖。javac 会把它编译成**先算 `s.hashCode()`，用 switch 匹配 hash 分支，在每个分支里再用 `equals` 精确比较**（因为 hash 可能冲突）。
> 如果 `s` 是 null，会在 `hashCode()` 调用处抛 NPE——所以 `switch(str)` 前要判空。
> 可以 `javap -c` 看到编译结果里有 `String.hashCode` 和 `String.equals` 的调用。

#### 追问 7：`final String a = "a"; a + "b"` 和普通变量拼接结果一样吗？

**【追问场景】**
精确考察"编译期常量"的概念。很多人不知道 `final` + 字面量会改变编译结果。

**【❌ 错误答法】**
> "都是运行期拼接，没区别。"
> 错在哪：`final String a = "a"` 是**编译期常量**（constant variable，JLS 4.12.4），编译器会把 `a + "b"` 直接折叠成 `"ab"`。

**【✅ 正确答法】**
> 不一样。`final String a = "a"` 属于编译期常量，编译器在编译时就把 `a + "b"` 折叠成字面量 `"ab"`，运行期就去常量池查一次，和 `"ab"` 直接用 `==` 比较是 true。
> 普通变量 `String a = "a"` 不是常量，运行期才会拼接，结果在堆上，`==` 比较是 false。
> 关键条件是**必须用字面量初始化**：`final String a = new String("a")` 就不是编译期常量了（`new` 不是常量表达式）。

### 2.5 动手验证

```java
public class StringDemo {
    public static void main(String[] args) {
        String s1 = "abc";
        String s2 = "abc";
        String s3 = new String("abc");
        String s4 = new String("abc");

        System.out.println(s1 == s2);            // true  同一个池对象
        System.out.println(s1 == s3);            // false new 出的堆对象
        System.out.println(s3 == s4);            // false 两个不同的堆对象
        System.out.println(s1.equals(s3));       // true  内容相同
        System.out.println(s1 == s3.intern());   // true  intern 返回池对象

        // 常量折叠
        final String f = "a";
        String s5 = f + "bc";
        System.out.println(s1 == s5);            // true  编译期折叠成 "abc"

        // 变量拼接
        String v = "a";
        String s6 = v + "bc";
        System.out.println(s1 == s6);            // false 运行期 new StringBuilder
    }
}
```

`javap -c` 观察拼接的字节码（重点看 `StringBuilder.append` 或 `invokedynamic makeConcat`）：

```bash
javac StringDemo.java && javap -c -p StringDemo | Select-String -Pattern "StringBuilder|makeConcat|invokedynamic"
```

### 2.6 一句话速记

- 字面量进池（**JDK 7 起池在堆**），`new` 必在堆；`==` 比是不是同一条，`equals` 比内容。
- 不可变 → 可共享 + hash 可缓存 + 线程安全 + 可做 key，`value` 是 `private final` 且构造器做**防御性拷贝**。
- `intern`：JDK 6 复制进池，JDK 7+ 把堆上引用放进池。
- 循环拼接必须把 `StringBuilder` 提到循环外；`final` + 字面量会被编译期折叠。

---

## 3. `==`、`equals`、`hashCode`：为什么必须一起重写

### 3.1 先建立直觉

**先用一个类比说清三者的分工**：

想象一个大图书馆：

- **`hashCode()` = 图书馆的"分区编号"**。它告诉你"这本书应该在 3 号区"。
- **`equals()` = 确认"这本书就是我要的那本"**。3 号区里可能有一百本书，得逐本核对。
- **`==` = "是不是同一本实体书"**（同一个物理对象）。

`HashMap` 找 key 的过程就是：

```
拿着要查的 key → 算 hashCode → 直接去对应分区（O(1)）
→ 在那个区里用 equals 逐本核对（区里通常只有 0~1 本，所以还是 O(1)）
```

**这就解释了为什么两个方法必须配套**：

- 如果你**重写了 `equals` 但没重写 `hashCode`**，那"逻辑相等的两个对象"会被分到**不同分区**，`HashMap` 永远找不到它——明明放进去过，`get` 却返回 null。这是生产事故级别的问题（比如用它做缓存 key、做去重集合）。
- 如果**只重写 `hashCode` 不重写 `equals`**，那所有对象都挤到同一个分区，然后逐本核对全部失败，退化成 O(n) 链表——不会出错，但性能崩了。

**一句话本质**：`equals` 决定"算不算同一个东西"，`hashCode` 决定"先到哪个抽屉找"。两者必须**同步更新**，否则抽屉和内容对不上。

### 3.2 原理与源码

#### 3.2.1 `Object` 的默认实现

```java
public class Object {
    // 默认 equals：直接比引用地址。也就是说，不重写的话"两个内容一样的对象"也不相等
    public boolean equals(Object obj) {
        return (this == obj);
    }

    // 默认 hashCode：native 方法，通常由对象的内存地址/随机数派生（HotSpot 有多种策略）
    public native int hashCode();
}
```

> **注意**：`Object.hashCode()` 是 native 的，所以它**不是**简单的内存地址。HotSpot 里默认是"随机数生成器"策略（`-XX:hashCode=`），因为如果按地址算，GC 移动对象后 hash 就会变，违反契约。

#### 3.2.2 契约（背这五条）

**`equals` 的五个性质**（来自 `Object.equals` 的 javadoc）：

| 性质 | 含义 | 违反的后果 |
| --- | --- | --- |
| 自反性 | `x.equals(x)` 必须 true | 集合里找不到自己 |
| 对称性 | `x.equals(y)` 为 true 则 `y.equals(x)` 也必须 true | `list.contains(x)` 行为诡异 |
| 传递性 | `x=y`、`y=z` 则 `x=z` | 排序/去重结果不可预测 |
| 一致性 | 只要参与比较的字段没变，多次调用结果不变 | 用到可变字段做 equals 就会踩 |
| 非空性 | `x.equals(null)` 必须返回 false，不能抛 NPE | 空指针异常 |

**`hashCode` 的契约**（只有三条，但每条都是血泪）：

```
1. 只要 equals 用到的信息没变，hashCode 必须始终返回同一个值（同一进程内）
2. 如果 x.equals(y) == true，则 x.hashCode() == y.hashCode()   ← 强制
3. 如果 x.equals(y) == false，hashCode 可以相同也可以不同      ← 允许冲突，但不同能提升性能
```

> 第 3 条最容易被误解：**hashCode 不同 → equals 一定不同；hashCode 相同 → equals 不一定相同**。这叫做"哈希冲突"。

#### 3.2.3 一个正确的实现模板（直接可背）

```java
public final class User {
    private final String name;
    private final int age;

    public User(String name, int age) { this.name = name; this.age = age; }

    @Override
    public boolean equals(Object o) {
        // ① 引用相同，直接 true（性能捷径）
        if (this == o) return true;
        // ② null 或类型不符，直接 false（用 getClass 保证同一具体类）
        if (o == null || getClass() != o.getClass()) return false;
        // ③ 逐字段比较
        User user = (User) o;
        return age == user.age && Objects.equals(name, user.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age);   // 内部：result = 31 * result + (name == null ? 0 : name.hashCode()); ...
    }
}
```

**每一行为什么这么写**：

| 代码 | 理由 |
| --- | --- |
| `this == o` | 省一次字段比较，且能处理字段为 null 的情况 |
| `o == null` | 满足"非空性"，不抛 NPE |
| `getClass() != o.getClass()` | 保证对称性：子类和父类不会互相相等 |
| `Objects.equals(name, ...)` | 内部处理了 `name == null` 的情况，避免手写 NPE |
| `age == user.age` | 基本类型直接比 |
| `Objects.hash(...)` | JDK 7+ 的工具方法，统一 31 乘法，自动处理 null |

#### 3.2.4 `Objects.hash` 源码

```java
public static int hash(Object... values) {
    return Arrays.hashCode(values);     // 底层是 Arrays.hashCode
}

// Arrays.hashCode
public static int hashCode(Object a[]) {
    if (a == null) return 0;
    int result = 1;
    for (Object element : a)
        result = 31 * result + (element == null ? 0 : element.hashCode());
    return result;
}
```

> 注意 `Objects.hash` 有个小缺点：**变长参数会创建数组**，高频调用时有微小开销。追求极致性能的核心类（如 JDK 里的 `String`、`Integer`）会手写 `31 * result + ...`。面试时说出这一点是加分项。

#### 3.2.5 `getClass()` vs `instanceof`：为什么不建议混用

```java
class Animal {}
class Dog extends Animal {
    @Override public boolean equals(Object o) {
        if (!(o instanceof Dog)) return false;   // 用 instanceof
        return true;
    }
}

class BadDog extends Dog {
    @Override public boolean equals(Object o) {
        if (!(o instanceof Dog)) return false;   // 子类也用 instanceof
        return true;
    }

// 结果：dog.equals(badDog) == true（Dog 认为它是 Dog）
//       badDog.equals(dog) == true（BadDog 也认为它是 Dog，因为 dog instanceof Dog）
// 但如果 BadDog 改成 getClass 比较：
//       badDog.equals(dog) == false，而 dog.equals(badDog) == true  → 违反对称性！
```

**结论**：

| 场景 | 推荐 | 原因 |
| --- | --- | --- |
| 实体类（无子类，如 VO/DTO/DO） | `getClass()` | 严格，Lombok 默认思路（`canEqual` 机制） |
| 需要支持子类参与比较（如 Hibernate 代理） | `instanceof` + `canEqual` | 否则代理子类永远不等于原始类 |
| 一个类用 `getClass`、另一个用 `instanceof` | **严禁** | 直接违反对称性 |

> Lombok 生成的 equals 里有个 `canEqual` 方法，就是为了在"父子类混合比较"时保护对称性。细节见 3.4 追问 3。

#### 3.2.6 为什么 `HashMap` 的定位是 `(n-1) & hash`

```java
// HashMap.getNode 的关键一行（JDK 8）
if ((p = tab[i = (n - 1) & hash]) == null)  // n 是容量，必须是 2 的幂
```

因为 `n` 是 2 的幂时，`(n-1) & hash` 等价于 `hash % n` 但**只用一次按位与**，比取模快得多。代价是**只用到 hash 的低位**，所以 HashMap 才对调用的 `hashCode` 再做一次扰动：

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);  // 让高 16 位也参与
}
```

**所以**：你重写的 `hashCode` 质量（低位分布是否均匀）直接影响 HashMap 性能。这就是为什么"用可变对象做 key"很危险，也是为什么 `hashCode` 不能随便返回常量。

### 3.3 面试标准问答

**Q1：为什么重写 `equals` 必须重写 `hashCode`？**
A：因为基于哈希的集合（HashMap/HashSet/ConcurrentHashMap）是**两段式查找**：先按 hashCode 定位桶，再在桶内用 equals 比较。如果只重写 equals，两个"逻辑相等"的对象 hashCode 不同，会被放进不同的桶，导致 `set.add(b)` 后 `set.contains(b)` 返回 false、`map.put(k,v)` 后 `map.get(k)` 拿不到值——**数据放进去取不出来，是功能性 bug，不是性能问题**。

**Q2：只重写 `hashCode` 不重写 `equals` 会怎样？**
A：所有对象会算出相同或不同的 hash，但桶内用 `Object.equals`（即 `==`）比较，只有同一个对象才相等。所以 `HashSet` 里会出现大量"hashCode 相同但 equals 不等"的元素，桶内链表变长，查询从 O(1) 退化到 O(n)。**不出错，但性能崩。**

**Q3：`hashCode` 相等，`equals` 一定相等吗？**
A：不一定。这就是哈希冲突。极端例子：`"Aa"` 和 `"BB"` 的 `hashCode` 都是 2112（这个例子面试常举）。冲突时 `HashMap` 用链表/红黑树兜底。**反过来说：`equals` 相等则 `hashCode` 必须相等，这是硬性契约。**

**Q4：为什么 `hashCode` 里要用 31？**
A：31 是奇素数（用偶数会丢信息，因为乘偶数会让低位变 0），且 `31 * i` 能被 JIT 优化成 `(i << 5) - i`（移位比乘法快）。另外 31 在某些实测中冲突率表现好。这是 JDK 沿用的经验值。

**Q5：`equals` 里用 `getClass` 还是 `instanceof`？**
A：没有子类的实体类用 `getClass()`（更严格、对称）；需要支持子类或代理类参与比较时用 `instanceof` 并配合 `canEqual` 保护对称性。**最忌讳的是同一个继承体系里一半用 `getClass`、一半用 `instanceof`**，会直接破坏对称性。Lombok 的 `@EqualsAndHashCode` 生成的就是 `canEqual` 模式。

**Q6：`Objects.hash` 和手写 `31 * ...` 有区别吗？**
A：语义一致，都是 31 滚动相乘、null 当 0 处理。区别是 `Objects.hash` 是变长参数，每次调用会创建 Object 数组，高频路径上有微小开销；核心类（如 `String`、`Integer`）会手写以避免装箱和数组创建。

### 3.4 高频追问与陷阱

#### 追问 1：只重写 `equals` 不重写 `hashCode`，会发生什么？能举出具体现象吗？

**【追问场景】**
这是必问题。面试官要的不是"会出问题"这种模糊回答，而是**能说出具体的、可复现的现象**。

**【❌ 错误答法】**
> "会导致 hashCode 不一致，可能会有问题。"
> 错在哪：太模糊，等于没答。面试官会立刻追问"什么算不一致""什么问题"，答不上来就等于不会。

**【✅ 正确答法】**
> 具体现象是：**对象放进哈希集合后取不出来**。
> 举例：`User` 只重写了 equals 没重写 hashCode，两个内容相同的对象 `u1`、`u2`。执行 `set.add(u1)` 后，`set.add(u2)` 依然会成功（因为 hashCode 不同，被判为不同桶，走不到 equals 那一步），于是**集合里出现两个"相等"的元素，去重失效**。
> 同理，`map.put(u1, "v")` 之后 `map.get(u2)` 返回 null——因为 `u2` 的 hash 定位到别的桶，压根没机会用 equals 去比较。
> 根因是哈希集合采用"**先按 hashCode 分桶，再在桶内 equals 比较**"的两段式查找，hashCode 不一致就没有第二次比较的机会。

#### 追问 2：`hashCode` 相同，`equals` 就相等吗？

**【追问场景】**
考察你是否真的理解"哈希冲突"，以及能否举出具体的冲突例子。

**【❌ 错误答法】**
> "相同，hashCode 一样说明对象一样。"
> 错在哪：**方向搞反了**。契约要求的是"equals 相等 → hashCode 相等"，而不是反过来。把充分条件和必要条件搞混。

**【✅ 正确答法】**
> 不一定，这就是哈希冲突。
> 契约只有单向要求：**equals 相等则 hashCode 必须相等**；反过来 hashCode 相同、equals 可以不等。
> 举个经典例子：`"Aa".hashCode() == "BB".hashCode() == 2112`，但它们显然不相等。
> HashMap 的应对是桶内用链表（长度 ≥ 8 且容量 ≥ 64 时转红黑树）逐个 equals 比较，所以**冲突只影响性能，不影响正确性**。
> 反过来如果违反契约（equals 相等但 hashCode 不同），才会导致功能性错误——也就是追问 1 的场景。

#### 追问 3：`getClass()` 和 `instanceof` 混用为什么会出问题？Lombok 怎么处理的？

**【追问场景】**
中高级岗位才会问，考察你有没有真正做过有继承体系的 POJO 比较，或者读过 Lombok 生成的代码。

**【❌ 错误答法】**
> "两个都能用，看个人习惯。"
> 错在哪：会破坏 `equals` 的**对称性**契约。给出这个答案说明没意识到"契约"是必须满足的，不是风格问题。

**【✅ 正确答法】**
> 会破坏对称性。举例：父类 `Dog` 用 `instanceof`、子类 `BadDog` 用 `getClass`，则 `dog.equals(badDog)` 为 true（badDog 是 Dog 实例），而 `badDog.equals(dog)` 为 false（getClass 判定不同类）——同一个比较，谁在左边结果不同，直接违反契约，在 `List.contains`、`Set` 等场景会引起诡异 bug。
> 统一用 `getClass` 最安全，但会否掉"代理子类等于原对象"的场景（Hibernate 懒加载代理就是子类，会导致集合里查不到）。
> Lombok 的解法是引入 `canEqual`：父类生成 `canEqual(Object)` 返回 `instanceof 父类`，子类重写返回 `instanceof 子类`；equals 里先判断 `o instanceof 父类`，再调 `((父类)o).canEqual(this)`。这样双向都能确定"对方是否与我同属一个具体类型"，对称性就保住了。

#### 追问 4：能用可变对象做 `HashMap` 的 key 吗？

**【追问场景】**
场景题，考察你是否踩过"改完 key 就取不出值"的坑。

**【❌ 错误答法】**
> "不能，会报错。"
> 错在哪：不会报错，这才是它可怕的地方——**编译通过、运行也不抛异常，只是静默地取不到值**。

**【✅ 正确答法】**
> 技术上可以（只要重写了 equals 和 hashCode），但**极度不安全**。
> 原因是：如果对象放进 map 之后，你修改了参与 `hashCode` 计算的字段，它的 hash 值就变了。再次 `get` 时按新的 hash 去新桶找，而对象还在旧桶里，于是**永远取不回来**；`remove` 也删不掉，造成事实上的内存泄漏。
> 正确做法：key 使用**不可变类型**（String、Integer、枚举、以及自己写的所有字段都是 final 的值对象）。
> 如果确实要放可变对象，至少保证**参与 hash/equals 的字段在入 map 后不再变化**（把那些字段设成 final 并让修改方法返回新对象）。

#### 追问 5：`x.equals(null)` 应该返回什么？`x.equals("abc")` 传错类型呢？

**【追问场景】**
看细节严谨度。有人会写出 `o.getName().equals(name)` 这种顺序颠倒的代码，一遇到 null 就 NPE。

**【❌ 错误答法】**
> "返回 false 就行，或者抛不抛异常都行。"
> 错在哪：契约明确要求返回 false，抛 NPE 是违规的。而且"传错类型"必须也返回 false 而不是抛 `ClassCastException`。

**【✅ 正确答法】**
> 必须返回 `false`，不能抛 NPE，也不能抛 `ClassCastException`。这是 `Object.equals` javadoc 里的"非空性"要求。
> 所以实现里必须先判断 `o == null`、再用 `instanceof` 或 `getClass` 做类型检查，**通过了才强转**。
> 常见踩坑写法是 `name.equals(o.getName())` 这种"拿字段去 equals 参数"，如果字段为 null 就 NPE；正确写法是用 `Objects.equals(this.name, other.name)`，或者把确定的非空常量放前面（如 `"SUCCESS".equals(status)`）。

#### 追问 6：`Objects.hash` 生成的 hash 分布够好吗？

**【追问场景】**
进阶题，考察你是否理解"hash 质量影响 HashMap 性能"，通常接在"为什么 HashMap 要扰动"之后问。

**【❌ 错误答法】**
> "够好，JDK 提供的肯定没问题。"
> 错在哪：`Objects.hash` 只保证"符合契约"，不保证"分布均匀"。字段多、或字段值范围小时冲突率可能偏高。

**【✅ 正确答法】**
> `Objects.hash` 用 31 滚动哈希，符合契约且分布一般够用，但有两个可优化点：
> 一是它对**字段顺序敏感**（`hash(a,b) != hash(b,a)`），这是好事（区分不同组合）；但 `[a,b]` 和 `[a,b,null]` 之类容易撞，字段多时冲突率会上升；
> 二是它只处理了"低位混合"，而 HashMap 只用 `(n-1) & hash` 的低位，所以真正决定性能的是**低位是否有足够熵**。如果字段是连续的小整数（如自增 ID），低位会很规律，桶分布不均，此时可以引入一个乘数扰动（如 `result * 31 + field` 已经有帮助，或改用 `Objects.hash` 再混一下）。
> 实测手段：写循环构造 N 个对象放进 HashSet，看实际 size 与预期是否一致、以及桶分布，就能验证。

### 3.5 动手验证

写一个 `BrokenUser`（只重写 equals）和 `GoodUser`（两个都重写），跑一遍看差别：

```java
import java.util.*;

public class EqualsHashCodeDemo {
    // 只重写 equals 的坑
    static class BrokenUser {
        String name; int age;
        BrokenUser(String name, int age) { this.name = name; this.age = age; }
        @Override public boolean equals(Object o) {
            if (this == o) return true;
            if (o == null || getClass() != o.getClass()) return false;
            BrokenUser u = (BrokenUser) o;
            return age == u.age && Objects.equals(name, u.name);
        }
        // 故意不重写 hashCode
    }

    // 正确版本
    static class GoodUser {
        final String name; final int age;
        GoodUser(String name, int age) { this.name = name; this.age = age; }
        @Override public boolean equals(Object o) {
            if (this == o) return true;
            if (o == null || getClass() != o.getClass()) return false;
            GoodUser u = (GoodUser) o;
            return age == u.age && Objects.equals(name, u.name);
        }
        @Override public int hashCode() { return Objects.hash(name, age); }
    }

    public static void main(String[] args) {
        Set<BrokenUser> s1 = new HashSet<>();
        s1.add(new BrokenUser("Tom", 20));
        s1.add(new BrokenUser("Tom", 20));
        System.out.println("BrokenSet size = " + s1.size());   // 2  ← 去重失效！
        System.out.println("contains = " + s1.contains(new BrokenUser("Tom", 20))); // false ← 取不出来！

        Set<GoodUser> s2 = new HashSet<>();
        s2.add(new GoodUser("Tom", 20));
        s2.add(new GoodUser("Tom", 20));
        System.out.println("GoodSet size = " + s2.size());     // 1  ← 正确
        System.out.println("contains = " + s2.contains(new GoodUser("Tom", 20))); // true

        // 哈希冲突演示
        System.out.println("Aa.hashCode=" + "Aa".hashCode() + ", BB.hashCode=" + "BB".hashCode()); // 都是 2112
    }
}
```

> 这个实验的输出（`2` 和 `false`）比任何文字解释都更能说明"为什么必须一起重写"。**建议你真的跑一遍**。

### 3.6 一句话速记

- `equals` 定"是不是同一个逻辑对象"，`hashCode` 定"去哪个桶找"，**必须成对重写**，否则对象放得进、取不出。
- 契约方向只有一个：**equals 相等 → hashCode 必须相等**；hashCode 相等但 equals 不等 = 哈希冲突，只损性能。
- 模板：`this==o` → `o==null || getClass()!=o.getClass()` → 逐字段 `Objects.equals`；`hashCode` 用 `Objects.hash` 或手写 31 滚动。
- `31`：奇素数 + 可优化为 `(i<<5)-i`；`getClass` 与 `instanceof` 不能在同一继承体系里混用。

---

## 4. 包装类缓存与自动拆装箱

### 4.1 先建立直觉

**为什么需要包装类？** 两个硬约束：

1. **泛型不能是基本类型**：`List<int>` 编译不过，只能写 `List<Integer>`。因为泛型在 JVM 里是 `Object`，而基本类型不是对象。
2. **集合只能存对象**：`HashMap` 的 key/value 都是 `Object`。

所以 Java 给 8 种基本类型各配了一个"盒子"（包装类），`int` ↔ `Integer` 靠**自动装箱/拆箱**自动转换。

**那缓存又是为什么？** 想象一个真实的业务：订单状态 0/1/2、页码 1~20、循环里的 `for (int i = 0; i < 100; i++)` 里装箱。这些**小整数的使用频率极高**，如果每次都 `new Integer(1)`，那就是疯狂制造垃圾对象。

**类比**：收银台的零钱。你不会每次找零都去银行换新钞——柜台里常备一堆 1 元、5 元的零钞，随手拿。JVM 也一样：**把 -128~127 的 Integer 对象提前造好放在缓存里**，用的时候直接拿，不新建。

这就是 `Integer a = 127; Integer b = 127; a == b` 为 `true` 的根本原因——**它俩指向缓存里同一个对象**。而 128 超出范围，各自 `new`，所以 `false`。

**一句话本质**：包装类的 `valueOf` 是**工厂方法**，不是构造器；小整数走"缓存池"（享元模式），大整数才新建对象。

> 顺带记住这个设计模式的名字：**享元模式（Flyweight）**——用共享对象减少内存。面试官问"哪里用了享元模式"，这就是标准答案之一。

### 4.2 原理与源码

#### 4.2.1 `Integer.valueOf` 与 `IntegerCache`

```java
public static Integer valueOf(int i) {
    // 落在缓存区间就直接返回缓存里的对象，否则新建
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];   // 注意下标要偏移 -128
    return new Integer(i);
}
```

缓存本体（`Integer` 的私有静态内部类）：

```java
private static class IntegerCache {
    static final int low = -128;            // 下界固定
    static final int high;                  // 上界可调，默认 127
    static final Integer cache[];           // 真正存对象的数组

    static {
        // 允许通过 JVM 参数扩大上界：-XX:AutoBoxCacheMax=1000
        int h = 127;
        String integerCacheHighPropValue =
            VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            int i = parseInt(integerCacheHighPropValue);
            i = Math.max(i, 127);           // 只能调大，不能小于 127
            h = Math.min(i, Integer.MAX_VALUE - (-low) - 1);
        }
        high = h;
        cache = new Integer[(high - low) + 1];   // 一次性创建 256 个对象
        int j = low;
        for (int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);         // 类初始化时全部造好
    }
}
```

**几个必须知道的细节**：

- 缓存对象是在 **`Integer` 类初始化时一次性创建**的（静态块），所以它常驻，不影响 GC（实际上 `IntegerCache.cache` 是强引用，会一直存活）
- 缓存下限是**写死的 -128**，只有上限能调
- 参数 `-XX:AutoBoxCacheMax=N`（部分版本叫 `java.lang.Integer.IntegerCache.high`）可以扩大上限

#### 4.2.2 八种包装类的缓存范围（背这张表）

| 包装类 | 缓存范围 | 能否调整 | 备注 |
| --- | --- | --- | --- |
| `Byte` | -128 ~ 127（**全部**） | 否 | byte 范围内全部缓存 |
| `Short` | -128 ~ 127 | 否 | 注意不是 -32768~32767 |
| `Integer` | -128 ~ 127 | **能**（`-XX:AutoBoxCacheMax`，仅上限） | 最常考 |
| `Long` | -128 ~ 127 | 否 | 常考的坑就在这 |
| `Character` | 0 ~ 127 | 否 | 是 `\u0000`~`\u007F` |
| `Boolean` | `TRUE` / `FALSE` | — | 只有两个实例，直接返回静态常量 |
| `Float` | **无缓存** | — | 每次 `valueOf` 都 new |
| `Double` | **无缓存** | — | 每次 `valueOf` 都 new |

> **为什么 Float/Double 不缓存？** 因为在它们的取值范围内，"常用值"没有明确边界（不像整数 0/1/2 有明确高频区），缓存所有可能值不现实，缓存一小段又收益极低。这是 JDK 设计者的取舍，能被问出来。

#### 4.2.3 编译器到底把 `Integer a = 1` 变成什么

```java
Integer a = 1;
```

反编译（`javap -c`）得到：

```text
iconst_1                                  // 把 int 1 压栈
invokestatic  #2  // Integer.valueOf:(I)Ljava/lang/Integer;   // 装箱
astore_1
```

**关键结论**：`Integer a = 1` **不是** `new Integer(1)`，而是 `Integer.valueOf(1)`。`new Integer(1)` 才会绕过缓存。这也是为什么 `new Integer()` 在 JDK 9 被标记 `@Deprecated`。

#### 4.2.4 拆箱 NPE：最阴的坑

```java
Integer i = null;
int j = i;                    // 拆箱：i.intValue() → NPE！
System.out.println(i);        // 但这个不报错，println 会打印 "null"（equals 到 String.valueOf 分支）
```

**更阴的是三元运算符**：

```java
Integer a = null;
boolean flag = true;
int result = flag ? a : 0;    // 看似有 0 兜底，实际 NPE！
```

**为什么？** 因为三元表达式 `flag ? a : 0` 的类型推导规则：一边是 `Integer`、一边是 `int`，编译器会把结果类型提升为 `int`，于是 `a` 要被**自动拆箱**——`null.intValue()` → NPE。**哪怕 `flag` 为 true 也会先做类型统一**（这是编译期的类型转换，不是运行期分支）。

**修复**：

```java
int result = flag ? a : Integer.valueOf(0);   // 两边都是 Integer，不做拆箱
// 或
int result = (a != null) ? a : 0;             // 先判空
```

#### 4.2.5 `equals` 与 `==` 在包装类上的行为

```java
Integer a = 1000, b = 1000;
System.out.println(a == b);          // false —— 超出缓存，两个对象
System.out.println(a.equals(b));     // true  —— equals 比的是值

Long c = 1000L;
System.out.println(a.equals(c));     // false —— 类型不同

// Integer.equals 源码（关键第一行就比类型）
public boolean equals(Object obj) {
    if (obj instanceof Integer) {
        return value == ((Integer)obj).intValue();
    }
    return false;
}
```

> **注意**：`a.equals(c)` 返回 false 而不是抛异常，因为 `c instanceof Integer` 为 false。但 `a == c` 在编译期就会报"不可比较的类型"。**这也是为什么比较包装类一律用 `equals`，或者干脆先统一转成基本类型比较。**

### 4.3 面试标准问答

**Q1：`Integer a = 127; Integer b = 127; a == b` 是 true 还是 false？128 呢？**
A：127 是 **true**，128 是 **false**。原因：`Integer a = 127` 编译成 `Integer.valueOf(127)`，而 `IntegerCache` 缓存了 -128~127 的对象，127 走缓存返回同一个实例；128 超出范围，`valueOf` 走 `new Integer(128)`，两个不同对象，`==` 比较引用必然 false。**注意 `equals` 两种情况都是 true。**

**Q2：怎么让 128 也相等？**
A：三种办法：①改 JVM 参数 `-XX:AutoBoxCacheMax=1000`（只能扩大上限，且注意这是**全局**影响）；②不要用 `==`，用 `equals`；③先转成基本类型再比：`a.intValue() == b.intValue()`。生产代码应该直接用 ②③，**不要依赖缓存范围写逻辑**（这是隐式契约，换个 JDK 版本或参数就崩）。

**Q3：为什么 Float 和 Double 没有缓存？**
A：浮点数没有明确的"高频小值"区间，缓存成本高收益低。而且浮点数的 `==` 比较本身就不推荐（精度问题），缓存与否意义不大。

**Q4：`Integer i = null; int j = i;` 会发生什么？**
A：抛 `NullPointerException`。因为拆箱会调用 `i.intValue()`，对 null 调用实例方法必然 NPE。而 `System.out.println(i)` 不抛异常，因为 `println(Object)` 走的是引用重载，直接打印 "null"。

**Q5：包装类和基本类型怎么选？**
A：**能用基本类型就用基本类型**。原因：①包装类是对象，有对象头开销（16 字节 vs 4 字节）；②会装箱，高频路径产生大量短命对象，增加 GC 压力；③有 NPE 风险。**只有三种情况必须用包装类**：泛型/集合元素、需要表达 null（如"未设置"）、反射/序列化 API 要求。

### 4.4 高频追问与陷阱

#### 追问 1：`Integer a = 128, b = 128; a == b` 为什么是 false？那 `Integer a = 1, b = 1` 呢？

**【追问场景】**
最经典的包装类问题。面试官先问 127（大家都会答 true），再突然问 128，看你是不是真懂缓存范围，还是只是记住了"两个数相等"。

**【❌ 错误答法】**
> "false，因为 == 比较的是地址，两个 Integer 是不同对象。"
> 错在哪：逻辑上没抓到要害。如果只说"比较地址"，那面试官立刻反击："那 127 为什么是 true？不也是两个变量吗？"就答不上来了。

**【✅ 正确答法】**
> 区别在于**是否走缓存**。`Integer a = 128` 编译成 `Integer.valueOf(128)`，因为 128 超出 `IntegerCache` 的 -128~127 范围，`valueOf` 内部执行 `new Integer(128)`，两次调用产生两个不同对象，`==` 比较引用 → false。
> 而 127 在缓存范围内，`valueOf` 直接返回 `cache[127 + 128]`，两次拿到的是同一个对象 → true。
> 所以这个行为**不是"值相等就相等"，而是"值在缓存范围内才相等"**，本质是享元模式 + `valueOf` 工厂方法。
> 生产代码不要依赖它，比较包装类一律用 `equals` 或 `intValue()`。

#### 追问 2：缓存范围能改吗？怎么改？

**【追问场景】**
紧跟上一题，考察"知不知道这是可配置的"，以及是否理解"能不能在生产里依赖它"。

**【❌ 错误答法】**
> "不能改，JDK 写死的。"
> 错在哪：下限 -128 确实写死，但**上限可改**，这是 `IntegerCache` 静态块里明确读的属性。

**【✅ 正确答法】**
> 下限 -128 写死，**上限可以通过 `-XX:AutoBoxCacheMax=N` 调整**（源码里读 `java.lang.Integer.IntegerCache.high` 属性，并且 `Math.max(i, 127)` 保证只能调大不能调小）。
> 但实践中**不建议依赖它**：这是全局 JVM 参数，改大了意味着启动时创建更多常驻对象；而且代码里依赖缓存范围是隐式契约，换环境就出 bug。
> 正确的做法是永远用 `equals` 比较值。

#### 追问 3：`Integer i = null; int j = flag ? i : 0;` 会出问题吗？

**【追问场景】**
高阶陷阱题，专门抓"以为三元运算符会短路/以为有兜底值"的人。答对的人通常对类型推导有真理解。

**【❌ 错误答法】**
> "不会，因为有 0 兜底，`i` 为 null 时会用 0。"
> 错在哪：把"三元运算符的两边"误解成"运行时任选其一"。实际编译器要**先把两边统一成同一类型**，`Integer` 和 `int` 统一的结果是 `int`，于是 `i` 必须先拆箱——拆箱发生在判断 `flag` 之前/同时，null 直接 NPE。

**【✅ 正确答法】**
> 会抛 NPE。因为三元表达式的类型推导会做**二元数值提升（binary numeric promotion）**：一边是 `Integer`、一边是 `int`，结果类型被提升为 `int`，所以 `i` 会被强制拆箱，`null.intValue()` 直接 NPE——**和 `flag` 的取值无关**。
> 修复：让两边都是包装类型（`flag ? i : Integer.valueOf(0)`），或者先判空（`i != null ? i : 0`）。
> 同类坑还有 `Objects.requireNonNullElse(i, 0)` 这种用法，也要注意返回值类型。

#### 追问 4：包装类做 `HashMap` 的 key 有风险吗？

**【追问场景】**
把包装类和上一个知识点（hashCode）串起来问，考察综合能力。

**【❌ 错误答法】**
> "没风险，包装类是不可变的，天然适合做 key。"
> 错在哪：前半句对、结论错。**`Long` 和 `Integer` 做 key 时会"看起来相等但取不出来"**，因为 `Long.equals(Integer)` 永远 false。

**【✅ 正确答法】**
> 包装类本身**不可变，是合格的 key**（这是对的）。但有个真实陷阱：**不同类型不能互通**。
> `map.put(1L, "v")` 之后 `map.get(1)` 返回 null——因为 `Long.hashCode(1)` 和 `Integer.hashCode(1)` 算出来的值不一样（`Long.hashCode` 是 `(int)(value ^ (value >>> 32))`，`Integer.hashCode` 就是 `value`），而且 `Long.equals(Integer)` 因为类型检查直接返回 false。**两段式查找在第一步就失败了。**
> 所以：key 的类型必须前后端/跨服务统一（比如都约定用 String 或都用 Long），接口入参要做类型归一化。这类 bug 在跨语言/跨服务调用里非常常见。

#### 追问 5：为什么 `new Integer(1) == new Integer(1)` 是 false，而 `Integer.valueOf(1) == Integer.valueOf(1)` 是 true？

**【追问场景】**
用来确认你理解"构造器 vs 工厂方法"的区别。本质和 String 的 `new String("abc")` 是同一个问题。

**【❌ 错误答法】**
> "因为 new 创建新对象，valueOf 可能复用。"
> 错在哪：说对了七成，但没点出"复用的是谁"。要说清"复用缓存池里的对象"，并说明**只有缓存范围内才复用**。

**【✅ 正确答法】**
> 因为 `new` 一定在堆上分配新对象，两次 `new` 就是两个不同对象，`==` 比引用必然 false。
> `valueOf` 是工厂方法，在 -128~127 范围内返回 `IntegerCache` 里的**同一个缓存实例**，所以 `==` 为 true；超出范围它内部也走 `new`，此时同样 false。
> 这也是 JDK 9 把 `new Integer(...)` 标记为 `@Deprecated` 的原因——**推荐一律用 `valueOf` 或自动装箱**。同理适用于 `Boolean.valueOf`、`Long.valueOf` 等所有包装类。

### 4.5 动手验证

```java
public class BoxDemo {
    public static void main(String[] args) {
        Integer a = 127, b = 127;
        Integer c = 128, d = 128;
        System.out.println("127: a==b " + (a == b) + ", equals " + a.equals(b)); // true, true
        System.out.println("128: c==d " + (c == d) + ", equals " + c.equals(d)); // false, true

        // 拆箱 NPE
        Integer n = null;
        try { int x = n; } catch (NullPointerException e) {
            System.out.println("拆箱 NPE: " + e.getClass().getSimpleName());
        }

        // 三元运算符的隐式拆箱
        try {
            int r = true ? n : 0;      // 编译期已决定要拆箱
            System.out.println("不会执行到这里");
        } catch (NullPointerException e) {
            System.out.println("三元运算符拆箱 NPE");
        }

        // 类型不同 equals 也不等
        System.out.println("Integer(1).equals(Long(1)) = " + Integer.valueOf(1).equals(Long.valueOf(1L))); // false
    }
}
```

想验证缓存上限可调，启动时加参数观察：

```bash
java -XX:AutoBoxCacheMax=1000 BoxDemo   # 此时 128 的 == 也会变成 true（不推荐依赖）
```

### 4.6 一句话速记

- `Integer a = 1` → `Integer.valueOf(1)`（不是 new）；**-128~127 走缓存**，`==` 才为 true。
- 缓存：Integer 上限可调（`-XX:AutoBoxCacheMax`）、Byte/Short/Long/Character(0~127)/Boolean 有、**Float/Double 无**。
- **三元运算符一边是包装类一边是基本类型时会隐式拆箱**，null 直接 NPE。
- 包装类比较一律用 `equals`；`Long` 和 `Integer` 即使值相同也互不相等。

---

## 5. final、static 与初始化顺序

### 5.1 先建立直觉

**`final` = 贴封条**。贴上之后不允许再改。但"改"的含义要看它贴在哪：

| 贴在哪 | 封条的含义 | 类比 |
| --- | --- | --- |
| 类上 | 不能被继承 | 这家公司不允许开分公司 |
| 方法上 | 不能被子类重写 | 这个流程不允许分店自己改 |
| 字段上 | 只能赋值一次（基本类型=值不变，引用=指向不变，**里面内容可变**） | 门牌号不能换，但屋里可以重新装修 |
| 参数上 | 方法体内不能重新赋值 | 借来的东西不能转借 |

**最容易错的是"final 引用"**：

```java
final List<String> list = new ArrayList<>();
list.add("a");        // ✅ 合法！final 只锁"指向"，不锁内容
list = new ArrayList<>();  // ❌ 编译错误，不能重新指向
```

**`static` = 属于类，不属于某个对象**。类比：**班级公告栏 vs 每个学生的学生证**。公告栏只有一块，全班共用（`static`）；学生证每人一本（实例字段）。

所以：

```java
class Counter {
    static int total;   // 所有 Counter 对象共用一份
    int own;            // 每个对象一份
}
Counter c1 = new Counter(), c2 = new Counter();
c1.total = 10;
System.out.println(c2.total);   // 10 —— 因为压根是同一个变量
```

**排序问题**：`static` 的东西属于"类初始化"，对象的东西属于"对象初始化"，所以 **static 一定先于对象初始化**。这就是那道经典的初始化顺序题的全部根基。

### 5.2 原理与源码

#### 5.2.1 `final` 的三种修饰 + 一个特殊保证

```java
final class CannotExtend {}           // ① 类：不能被继承（String、Integer、System 都是 final）

class A {
    final void noOverride() {}         // ② 方法：子类不能重写（但可以重载！见 5.4 追问）
}

class B {
    final int x = 1;                   // ③ 字段：只能赋值一次
    final List<String> list = new ArrayList<>();  // 引用不可变，内容可变
}
```

**④ `final` 字段的内存语义（JSR-133 特别加的）**：

```java
public class FinalFieldExample {
    final int x;
    int y;
    static FinalFieldExample obj;

    FinalFieldExample() {
        x = 3;      // final 字段在构造器里赋值
        y = 4;      // 普通字段
    }

    static void writer() { obj = new FinalFieldExample(); }

    static void reader() {
        FinalFieldExample f = obj;
        if (f != null) {
            int i = f.x;   // 保证看到 3（不会看到 0）
            int j = f.y;   // 可能看到 0（不保证）
        }
    }
}
```

**规则**：**只要对象没有在构造器里把 `this` 泄露出去，其他线程看到 `final` 字段时一定是构造完成后的值，不需要额外同步。**

这是 Java 语言规范给 `final` 的特殊承诺。所以"不可变对象"实现线程安全的标准配方是：**所有字段 `final` + 不泄露 `this`**（如 `Collections.unmodifiableList`、`String`、`Integer`）。

> 面试话术：问"final 能保证线程安全吗"，答"**单个 final 字段的值一旦构造完成就保证可见，因此不可变对象天然线程安全；但这不等于给字段加 final 就能保证复合操作的原子性**"——这个回答既准确又有层次。

#### 5.2.2 `static` 变量到底在哪个内存区（高频且容易答错）

答案：**在堆里，跟着 Class 对象走**。

演变过程：

| 版本 | 静态变量位置 | 说明 |
| --- | --- | --- |
| JDK 6 及以前 | 永久代（PermGen） | 方法区的一部分 |
| JDK 7 | **移到堆** | 和字符串常量池一起搬出来，避免永久代 OOM |
| JDK 8+ | **在堆中**，随 `Class` 对象关联 | 永久代被元空间取代，但静态变量仍在堆里（元空间只放类元数据） |

> **经典陷阱**：问"静态变量在方法区吗"，正确的答法是"JDK 7 之后在堆里，随 Class 对象存放；JDK 8 元空间只放类元数据"。答"在元空间"是错的（这是最常被扣分的地方）。

#### 5.2.3 类初始化时机（主动引用 vs 被动引用）

**类初始化（执行 `<clinit>` = 静态字段赋值 + 静态块）只在这些"主动引用"时发生**：

1. `new`、`getstatic`/`putstatic`、`invokestatic`（即 new 对象、读写静态字段、调静态方法）
2. 反射：`Class.forName("X")`（注意带 `initialize=true`）
3. 初始化子类时，父类先初始化
4. 启动类（含 `main` 的类）
5. `MethodHandle` / `invokedynamic` 解析到静态方法时
6. 接口里定义了 `default` 方法时，实现类初始化会触发接口初始化（Java 8 特例）

**这些是"被动引用"，不会触发初始化**（超高频考点）：

```java
// ① 通过子类访问父类的静态字段 → 只初始化父类，不初始化子类
class Parent { static int value = 1; static { System.out.println("Parent init"); } }
class Child extends Parent { static { System.out.println("Child init"); } }
System.out.println(Child.value);   // 只输出 Parent init

// ② 定义数组 → 不初始化元素类型
Parent[] arr = new Parent[10];     // 不输出任何 init

// ③ 访问编译期常量 → 编译期内联，压根不加载类
class Const { static final int MAX = 100; static { System.out.println("Const init"); } }
System.out.println(Const.MAX);     // 不输出 Const init（因为 MAX 被内联成 100）
```

**第 ③ 条最容易被忽略**：只有 `static final` 且**用字面量初始化**的才是编译期常量。如果是 `static final int MAX = (int) System.currentTimeMillis();`（运行时才能算），就会触发初始化。

#### 5.2.4 完整初始化顺序（合并第 1 节的结论）

```java
class Parent {
    static { System.out.println("1 父-静态块"); }
    { System.out.println("3 父-实例块"); }
    Parent() { System.out.println("4 父-构造器"); }
    static void s() { System.out.println("父-静态方法"); }
}

class Child extends Parent {
    static { System.out.println("2 子-静态块"); }
    { System.out.println("5 子-实例块"); }
    Child() { System.out.println("6 子-构造器"); }
}

new Child();
```

**输出顺序**：

```
1 父-静态块 → 2 子-静态块 → 3 父-实例块 → 4 父-构造器 → 5 子-实例块 → 6 子-构造器
```

**记忆口诀**：**先静态后实例，先父后子；静态只一次，实例每次 new 都来一遍。**

### 5.3 面试标准问答

**Q1：`final`、`finally`、`finalize` 有什么区别？**
A：三者毫无关系，纯属名字像。`final` 是修饰符（类/方法/字段不可变）；`finally` 是异常处理块（无论是否异常都执行）；`finalize` 是 `Object` 的方法，GC 回收前调用（**JDK 9 起标记 deprecated，不推荐使用**，因为执行时机不确定、可能让对象"复活"、拖慢 GC）。

**Q2：`static` 方法能被重写吗？**
A：不能。**静态方法属于类，编译期静态绑定**。如果子类写了同名同参数的静态方法，那叫**隐藏（hide）**，不叫重写。而且判断调哪个取决于**变量的声明类型**，不是运行时类型：

```java
Parent p = new Child();
p.staticMethod();   // 调用 Parent 的版本（看声明类型）
```

**Q3：静态变量和实例变量的区别？**
A：①静态变量属于类，所有实例共享一份，随类加载而分配、随类卸载而释放；实例变量属于对象，每个实例一份，随对象创建而分配。②静态变量 JDK 7 起在**堆**中随 Class 对象存放；实例变量在堆中的对象里。③静态方法不能直接访问实例成员（因为没有 `this`）。

**Q4：`final` 字段为什么不需要 `volatile`？**
A：因为 JSR-133 给 `final` 加了特殊的内存语义：只要构造器没泄露 `this`，其他线程看到该对象的 `final` 字段时，保证看到的是构造完成的值（禁止把 final 字段的写重排序到构造器之外）。所以不可变对象是"发布即安全"的，不需要额外同步。**但这只保证单字段，不保证多个字段之间的一致性，也不保证非 final 字段。**

**Q5：什么情况下会触发类初始化？**
A：主动引用六种：new/读写静态字段/调静态方法、反射 `Class.forName`、初始化子类（父类先初始化）、启动类、MethodHandle 解析、接口有 default 方法时的实现类初始化。反之，**通过子类访问父类静态字段、定义数组、访问编译期常量**都只算被动引用，不触发。

### 5.4 高频追问与陷阱

#### 追问 1：`final` 修饰的引用，里面的内容能改吗？

**【追问场景】**
几乎必问的基础题，但很多人答错方向。面试官想听你区分"引用不可变"和"对象不可变"。

**【❌ 错误答法】**
> "不能改，final 就是不可变。"
> 错在哪：混淆了"引用"和"对象"。`final List` 只是不能重新指向另一个 List，`add`/`remove`/`set` 全都合法。

**【✅ 正确答法】**
> `final` 锁的是**引用**，不是对象内容。所以 `final List<String> list = new ArrayList<>()` 之后，`list.add("a")`、`list.remove(0)` 都可以，只有 `list = new ArrayList<>()` 会编译报错。
> 基本类型（`final int x`）和不可变对象（`final String s`）才看起来"完全锁死"，那是因为它们本身没有可变状态。
> **要真正实现不可变**，需要：类加 `final`（防止子类破坏），所有字段 `private final`，不提供修改方法，如果字段是可变对象则构造时**防御性拷贝**、返回时也拷贝（如 `Collections.unmodifiableList`）。

#### 追问 2：静态方法能被子类"重写"吗？调用时看哪个类型？

**【追问场景】**
考"静态绑定 vs 动态绑定"。面试官会给你 `Parent p = new Child(); p.staticMethod();` 让你说输出。

**【❌ 错误答法】**
> "能重写，调用子类的版本，因为多态。"
> 错在哪：静态方法不参与多态。它不是被重写，而是被**隐藏**（hide），调用哪个版本由**编译时的声明类型**决定。

**【✅ 正确答法】**
> 不能重写。静态方法在编译期就静态绑定到声明类型，不存在动态分派。
> 所以 `Parent p = new Child(); p.staticMethod()` 执行的是 `Parent` 的版本——因为 `p` 的声明类型是 `Parent`，编译器就按 `Parent.staticMethod()` 生成 `invokestatic` 指令。
> 如果写成 `Child c = new Child(); c.staticMethod()`，才调 `Child` 的版本。
> 这也是为什么**不应该通过实例引用调用静态方法**（IDE 会警告），容易让人误以为是多态。

#### 追问 3：静态变量在方法区还是堆里？

**【追问场景】**
专门抓"永久代 / 元空间 / 堆"三者混淆的人。答错的概率非常高。

**【❌ 错误答法】**
> "在方法区/元空间里，因为是类级别的变量。"
> 错在哪：把"属于类"和"存放在方法区"划等号。JDK 7 起静态变量就搬到堆里了，元空间只放类元数据。

**【✅ 正确答法】**
> **JDK 7 起静态变量在堆里**，随 `Class` 对象一起存放；JDK 6 及以前在永久代。
> 原因是 JDK 7 把字符串常量池和静态变量从永久代迁到堆，避免永久代容量固定导致的 OOM（`OutOfMemoryError: PermGen space`）。
> JDK 8 用元空间取代永久代，元空间改用**本地内存**、只存类元数据（类的结构、方法信息、运行时常量池等），但**静态变量依然在堆里**。
> 所以完整的答法是：**"类元数据在元空间，静态变量和字符串常量池在堆"**。

#### 追问 4：`static final` 修饰的常量一定会触发类初始化吗？

**【追问场景】**
这是"被动引用"里最容易翻车的一条。多数人只记得"常量不触发初始化"，不知道前提条件。

**【❌ 错误答法】**
> "常量会被编译期内联，所以一定不触发类初始化。"
> 错在哪：漏了前提。只有**编译期常量**（compile-time constant）才内联。如果初始化表达式需要运行时求值，就必须初始化类才能拿到值。

**【✅ 正确答法】**
> 只有"**编译期常量**"才会被内联、不触发类初始化。编译期常量的条件是：`static final` + 类型是基本类型或 String + **用常量表达式初始化**（字面量、或由其他常量表达式算出来的）。
> 例如 `static final int MAX = 100;` → 调用方字节码里直接是常量 100，不加载该类。
> 但 `static final int MAX = (int) System.currentTimeMillis();` → 运行时才能算，必须触发类初始化。
> 另外，`static final Object OBJ = new Object();` 不是常量表达式，访问它也会触发初始化（因为要读静态字段）。

#### 追问 5：`static` 块和 `static` 字段谁先执行？

**【追问场景】**
考"按源码顺序"这个细节。有人以为"字段一定优先于代码块"。

**【❌ 错误答法】**
> "静态字段先初始化，然后执行静态块。"
> 错在哪：静态字段的显式初始化和静态块，**严格按源码先后顺序**执行，没有"字段优先"的规则。

**【✅ 正确答法】**
> 静态字段的显式初始化表达式和静态代码块，会被编译器**按源码书写顺序**合并进 `<clinit>` 方法，谁写在前面谁先执行。
> 例如：
> ```java
> static int a = 1;
> static { a = 2; }
> static int b = a;   // b 是 2，因为前面已经把 a 改成 2 了
> ```
> 如果是反过来写（静态块在前、字段赋值在后），结果就完全不同。**这类题面试官特别爱出"输出什么"的形式，一定要按源码顺序推，不要凭直觉。**

### 5.5 动手验证

```java
public class InitOrderDemo {
    static { System.out.println("1 本类静态块"); }
    static int s1 = print("2 静态字段 s1");
    static { System.out.println("3 本类静态块2, s1=" + s1); }

    private int a = print("4 实例字段 a");
    { System.out.println("5 实例块"); }

    InitOrderDemo() {
        System.out.println("6 构造器");
    }

    static int print(String msg) { System.out.println(msg); return 1; }

    public static void main(String[] args) {
        System.out.println("--- main 开始（注意静态块已经执行完了）---");
        new InitOrderDemo();
        System.out.println("--- 第二次 new ---");
        new InitOrderDemo();   // 静态部分不再执行
    }
}
```

验证 `final` 引用的可变性、以及"常量不触发初始化"：

```java
public class FinalDemo {
    static final int CONST = 100;                    // 编译期常量
    static final int RUNTIME = (int) System.currentTimeMillis() % 10;  // 非编译期常量

    static { System.out.println("FinalDemo 被初始化了"); }

    public static void main(String[] args) {
        System.out.println("读 CONST: " + CONST);     // 不会打印"被初始化"，因为内联
        System.out.println("读 RUNTIME: " + RUNTIME); // 会打印，因为要初始化类

        final java.util.List<String> list = new java.util.ArrayList<>();
        list.add("a");        // 合法
        // list = new java.util.ArrayList<>();  // 编译错误
        System.out.println(list);
    }
}
```

### 5.6 一句话速记

- `final` 锁**引用**不锁内容；`final` 字段有"构造完成后保证可见"的内存语义，所以不可变对象天然线程安全。
- `static` 属于类：**JDK 7 起静态变量在堆**，类元数据在元空间。
- 顺序口诀：**先静态后实例，先父后子，静态只执行一次**；静态块与静态字段按**源码顺序**执行。
- 被动引用三连：子类引用父类静态字段、定义数组、访问编译期常量 → **不触发初始化**。

---

## 6. 重载 vs 重写、多态与动态绑定

### 6.1 先建立直觉

**这两个概念经常被混淆，其实它们的区别在于"什么时候决定调用哪个方法"**：

| | 重载（Overload） | 重写（Override） |
| --- | --- | --- |
| 发生位置 | **同一个类**（或父子类之间） | 子类重写父类 |
| 决定时机 | **编译期**（看引用的声明类型 + 参数类型） | **运行期**（看对象的实际类型） |
| 判定依据 | 方法名相同、**参数列表不同** | 方法名、参数列表、返回类型都相同（返回类型可协变） |
| 关系 | 同一个动作的不同"参数版本" | 同一个动作的不同"实现" |

**类比**：你去餐厅点餐。

- **重载** = 同一道菜的不同规格："来碗面" / "来碗大碗面" / "来碗加蛋的面"。**你点的时候（编译时）就知道要哪一碗**，因为菜单上写清楚了。
- **重写** = 菜单上只写"来碗面"，但**不同分店（子类）做的味道不一样**。你只知道要"面"，具体哪家店做的、什么味道，**要等你走进那家店（运行时）才知道**。

这就是全部区别：**重载看"你写的是什么"，重写看"它实际是什么"**。

```java
class Animal { void speak() { System.out.println("动物叫"); } }
class Dog extends Animal { @Override void speak() { System.out.println("汪汪"); } }

Animal a = new Dog();   // 声明类型 Animal，实际类型 Dog
a.speak();              // 输出"汪汪" ← 重写：看实际类型（运行期）
```

如果把 `speak` 改成重载（参数不同），那就完全由编译期决定：

```java
class Printer {
    void print(int x)    { System.out.println("int"); }
    void print(String s) { System.out.println("String"); }
}
Printer p = new Printer();
p.print(1);        // int    ← 编译期根据参数类型 1 决定
p.print("1");      // String
```

### 6.2 原理与源码

#### 6.2.1 字节码：三种 invoke 指令决定了一切

| 指令 | 用在哪 | 绑定时机 | 能否多态 |
| --- | --- | --- | --- |
| `invokestatic` | 静态方法 | 编译期 | ❌ 不能 |
| `invokespecial` | 构造器、`private` 方法、`super.xxx()` | 编译期 | ❌ 不能 |
| `invokevirtual` | 普通实例方法 | **运行期**（查 vtable） | ✅ 能 |
| `invokeinterface` | 接口方法 | **运行期**（查 itable） | ✅ 能 |
| `invokedynamic` | Lambda、字符串拼接（JDK 9+） | 首次调用时链接 | 特殊 |

**关键结论**：**只有 `invokevirtual` 和 `invokeinterface` 才支持多态**。所以 `private` 方法、`static` 方法、构造器、`final` 方法（JIT 后优化成直接调用）都不能被"重写"。

#### 6.2.2 动态分派：vtable 是怎么查的

每个类在方法区有一张**虚方法表（vtable）**，记录"这个方法名+签名 → 实际要执行的代码地址"。

```text
Animal 的 vtable:            Dog 的 vtable:
┌──────────────┬─────────┐   ┌──────────────┬─────────┐
│ speak()      │ Animal  │   │ speak()      │  Dog    │  ← 子类覆盖了父类的槽位
└──────────────┴─────────┘   └──────────────┴─────────┘
```

调用 `a.speak()` 时：

1. 从 `a` 的**对象头**拿到 Klass Pointer → 找到实际类型 `Dog`
2. 在 `Dog` 的 vtable 里查 `speak` 这个槽位
3. 调用槽位里的地址 → 执行 `Dog.speak()`

**所以多态的开销就是"两次内存访问 + 一次间接跳转"，非常廉价**（这也是 Java 能把多态做进日常写法的原因）。JIT 还会做**内联缓存（inline cache）**：如果发现某个调用点 99% 都是同一个类型，就直接把目标方法内联进来，连查表都省了。

#### 6.2.3 重载的解析过程（为什么有时候结果很反直觉）

Java 的重载解析分三个阶段，**前一个阶段找到就不再往下走**：

| 阶段 | 允许的转换 | 例子 |
| --- | --- | --- |
| 阶段 1：严格调用 | 不允许装箱/拆箱、不允许变长参数 | `f(int)` 匹配 `f(1)` |
| 阶段 2：宽松调用 | 允许装箱/拆箱 | `f(Integer)` 匹配 `f(1)` |
| 阶段 3：变长参数 | 允许可变参数 | `f(int...)` 匹配 `f(1, 2, 3)` |

**这解释了一个经典怪现象**：

```java
void f(int x)     { System.out.println("int"); }
void f(Integer x) { System.out.println("Integer"); }

f(1);    // 输出 int —— 阶段 1 就匹配到了，压根轮不到拆装箱版本
f(Integer.valueOf(1));  // 输出 Integer
```

再看一个歧义例子：

```java
void f(int a, long b)    { System.out.println("int,long"); }
void f(long a, int b)    { System.out.println("long,int"); }

f(1, 1);   // 编译错误：两个都能匹配，且没有"更具体"的那个 → ambiguous
```

**"最具体（most specific）"如何判断**：如果 A 能接受所有 B 能接受的参数、反之不行，那 A 更具体。上例里两者互为"更具体"关系，所以歧义。

#### 6.2.4 重写的硬性规则（`@Override` 会帮你检查）

| 规则 | 说明 |
| --- | --- |
| 方法名 + 参数列表必须相同 | 不同就是重载，不是重写 |
| 返回类型可以**协变** | 子类可以返回更具体的类型：父类 `Animal m()`，子类 `Dog m()` ✅ |
| 访问权限**不能收紧** | 父 `public` → 子只能 `public`；父 `protected` → 子可以 `protected`/`public` |
| 不能抛出更宽的**受检异常** | 父 `throws IOException` → 子只能抛 `IOException` 或其子类，或宽一点的 `RuntimeException`（受检异常不能变宽） |
| `final` / `static` / `private` 方法不能被重写 | `static` 是隐藏，`private` 是不可见（等于新方法） |

> **提醒**：**永远写 `@Override`**。它能让编译器帮你检查"你以为在重写，其实只是重载"这个高频错误（少写一个参数就变成重载，运行时行为完全不同还不报错）。

#### 6.2.5 字段没有多态（易错点）

```java
class Parent { String name = "parent"; }
class Child extends Parent { String name = "child"; }

Parent p = new Child();
System.out.println(p.name);   // "parent" ← 看声明类型！

Child c = new Child();
System.out.println(c.name);   // "child"
```

**原因**：字段访问在编译期就确定（`getfield` 指令带的是"声明类型 + 字段名"），不走 vtable，**没有动态分派**。所以"多态只适用于方法，不适用于字段"，这是必须记住的。

### 6.3 面试标准问答

**Q1：重载和重写的区别？**
A：①发生范围：重载在同一个类（或父子类），重写在父子类之间；②决定时机：重载编译期（静态绑定），重写运行期（动态分派）；③判定条件：重载要求方法名相同、**参数列表不同**（返回类型可任意），重写要求方法名和参数列表都相同、返回类型可协变、访问权限不能收紧、不能抛更宽的受检异常；④目的：重载是同一行为的多种参数形式，重写是多态的实现方式。

**Q2：为什么重载是编译期决定，重写是运行期决定？**
A：因为重载的方法名和参数类型在编译时就能从静态类型确定，编译器直接生成指向具体方法的 `invokevirtual`/`invokestatic`（带明确描述符）；而重写需要按对象的**实际类型**查 vtable，只能在运行期决定，所以用 `invokevirtual` 的动态分派。

**Q3：静态方法能被重写吗？**
A：不能。静态方法属于类，编译期静态绑定为 `invokestatic`。子类写同名同参数静态方法叫**隐藏**，调用取决于**引用变量的声明类型**，而不是运行时类型。

**Q4：`private` 方法能被重写吗？**
A：不能。`private` 对子类不可见，是被 `invokespecial` 静态绑定的。子类写一个同名方法，那是**全新方法**，不是重写（加 `@Override` 会编译报错）。

**Q5：多态的实现原理？**
A：靠**虚方法表（vtable）**。每个类维护一张"方法签名 → 实现地址"的表，子类重写方法时覆盖父类的对应槽位。调用时通过对象头的类型指针找到实际类，再查表定位方法地址，实现动态分派。JIT 会做内联缓存和类型预测（单态/多态内联），把大部分开销优化掉。

**Q6：为什么字段没有多态？**
A：因为字段访问编译为 `getfield` 指令，指令里已经写死了"声明类型 + 字段名 + 描述符"，是静态解析；方法调用才需要动态分派。

### 6.4 高频追问与陷阱

#### 追问 1：`Animal a = new Dog(); a.speak()` 调用的是谁的方法？如果我改成静态方法呢？

**【追问场景】**
多态入门题，但后半句（改成静态方法）是分水岭。面试官想确认你知不知道"静态方法不参与多态"。

**【❌ 错误答法】**
> "调用子类的方法，因为运行时是 Dog。"——第一问对。第二问继续答"也是子类的，因为对象是 Dog"就错了。

**【✅ 正确答法】**
> 实例方法调用子类版本（`Dog.speak()`），因为 `invokevirtual` 是动态分派，按对象的实际类型走 vtable。
> 但如果把 `speak` 改成 `static`，调用的就是 **`Animal` 的版本**——静态方法用 `invokestatic` 指令，编译期就按**声明类型** `Animal` 绑定了，不存在动态分派。
> 同理，如果 `speak` 是 `private` 或 `final`，也会被静态绑定，调用父类版本。

#### 追问 2：`f(1)` 有两个重载 `f(int)` 和 `f(Integer)`，输出哪个？为什么？

**【追问场景】**
考重载解析的三个阶段。答"可能报错"或"随机"都是没理解规则。

**【❌ 错误答法】**
> "两个都能匹配，应该会报错/取 Integer 版本。"
> 错在哪：没理解解析的**优先级顺序**。阶段 1（不允许装箱）就匹配到了，不会往下走。

**【✅ 正确答法】**
> 输出 `f(int)` 的版本。因为 Java 重载解析分三阶段：**阶段 1 不允许装箱/拆箱和变长参数**，此时 `f(int)` 直接匹配，解析结束，根本不会进入"允许装箱"的阶段 2。
> 这体现了一个设计原则：**优先选择不需要任何转换的方法**，其次才考虑装箱/拆箱，最后才是可变参数。
> 反过来说，如果只有 `f(Integer)` 一个版本，`f(1)` 才会走阶段 2 自动装箱。

#### 追问 3：`f(null)` 有 `f(String)` 和 `f(Object)` 两个重载，调哪个？如果两个参数类型没有父子关系呢？

**【追问场景】**
经典歧义题。面试官想看你能不能说出"最具体（most specific）"的判定规则，以及什么时候编译报错。

**【❌ 错误答法】**
> "会报错，因为 null 有歧义。"
> 错在哪：`f(null)` 传 `String` 和 `Object` 时可以确定，因为 `String` 比 `Object` 更具体。真正会歧义的是**两个平级类型**（如 `String` 和 `Integer`）。

**【✅ 正确答法】**
> 如果两个重载是 `f(String)` 和 `f(Object)`，会选 **`f(String)`**——因为 `String` 是 `Object` 的子类型，即"`String` 能接受的所有值 `Object` 都能接受，反之不行"，所以 `String` 更具体。
> 但如果两个重载是 `f(String)` 和 `f(Integer)`，两者没有子类型关系，**编译报错：reference to f is ambiguous**。
> 还有一个细节：null 字面量的类型是 `nullType`，它能被转换成任何引用类型（不能转基本类型），所以才会出现这种"传 null 到处都能接"的歧义。
> 实践建议：**避免为同一方法提供多个引用类型重载**，必要时用不同方法名（如 `write(String)` / `writeInt(int)`）。

#### 追问 4：为什么说子类重写时返回类型可以变？

**【追问场景】**
考"协变返回类型（covariant return type）"，Java 5 引入。答"不能改"是旧观念。

**【❌ 错误答法】**
> "方法签名必须完全一致，返回类型也一样。"
> 错在哪：方法签名的定义**只包含方法名和参数列表**，返回类型不算签名的一部分。而且从 Java 5 起允许协变返回。

**【✅ 正确答法】**
> 可以协变：子类重写方法的返回类型可以是父类返回类型的**子类型**。
> 例如父类 `Animal get()`、子类 `Dog get()` 是合法的重写（`Dog` 是 `Animal` 的子类型），调用方用父类引用时依然安全（拿到的是 Animal），用子类引用时能直接拿到更具体的类型，省掉强转。
> 反过来，如果返回类型没有子类型关系（如 `String` 与 `Integer`），那就不是重写，编译报错。
> 注意：**重载不能只靠返回类型区分**——`int f()` 和 `String f()` 在同一个类里会编译错误，因为调用 `f()` 时无法通过返回类型选择。

#### 追问 5：`@Override` 写不写有区别吗？

**【追问场景】**
看工程习惯，但答得好的能体现你对"编译期检查"价值的理解。

**【❌ 错误答法】**
> "没区别，就是注解，写了更好看。"
> 错在哪：漏掉了它的核心价值——**编译期校验**。

**【✅ 正确答法】**
> 有实质区别：`@Override` 让编译器检查"这个方法确实重写了父类/接口的方法"。如果没有真正重写（比如参数写错、方法名拼错、父类方法是 `private`/`static`），编译器会直接报错。
> 不加注解时，写错参数会**静默地变成重载**，编译通过，但运行时行为完全不对——这是非常难查的 bug。
> 所以规范要求**所有重写都加 `@Override`**（阿里巴巴 Java 开发手册明确条款）。IDE 也能靠它做跳转和重命名联动。

### 6.5 动手验证

```java
public class OverrideOverloadDemo {
    static class Animal {
        String name = "Animal";
        void speak() { System.out.println("动物叫"); }
        static void staticSpeak() { System.out.println("Animal static"); }
        private void priv() { System.out.println("Animal private"); }
    }

    static class Dog extends Animal {
        String name = "Dog";
        @Override void speak() { System.out.println("汪汪"); }
        static void staticSpeak() { System.out.println("Dog static"); }
        // @Override void priv() {}   // 加 @Override 会编译报错：private 不能重写
    }

    // 重载解析顺序验证
    static void f(int x)     { System.out.println("f(int)"); }
    static void f(Integer x) { System.out.println("f(Integer)"); }
    static void g(String s)  { System.out.println("g(String)"); }
    static void g(Object o)  { System.out.println("g(Object)"); }

    public static void main(String[] args) {
        Animal a = new Dog();
        a.speak();                 // 汪汪        ← 重写，看实际类型
        System.out.println(a.name);// Animal      ← 字段没有多态，看声明类型

        Animal.staticSpeak();      // Animal static ← 静态方法看声明类型
        ((Dog) a).staticSpeak();   // Dog static

        f(1);                      // f(int)      ← 阶段 1 直接匹配
        f(Integer.valueOf(1));     // f(Integer)

        g(null);                   // g(String)   ← String 更具体
    }
}
```

### 6.6 一句话速记

- **重载看声明（编译期），重写看实际（运行期）**；`invokevirtual`/`invokeinterface` 才支持多态。
- 重载解析三阶段：**严格 → 允许装箱 → 变长参数**，前一阶段命中就停。
- `static`、`private`、`final`、构造器都**不能重写**；`static` 是隐藏、`private` 是不可见。
- **字段没有多态**；重写返回类型可协变、访问权限不能收紧、受检异常不能变宽。

---

## 7. 接口、抽象类与内部类

### 7.1 先建立直觉

**接口 vs 抽象类，本质区别是"承诺能做什么" vs "提供是什么的一半"**：

| | 接口（interface） | 抽象类（abstract class） |
| --- | --- | --- |
| 语义 | **能力/契约**："能飞"、"能比较"、"能被序列化" | **类型/模板**："是一种动物"、"是个半成品" |
| 类比 | 驾驶证——证明你会开车，跟你是什么职业无关 | 汽车设计图——已经造好了底盘和轮子，你只需决定发动机 |
| 能否多实现 | ✅ 一个类可以实现多个接口 | ❌ 只能继承一个（Java 单继承） |
| 字段 | 只能是 `public static final` 常量 | 任意（可 `private`、可变） |
| 方法 | 抽象方法 + `default`/`static`/`private`（JDK 9） | 抽象方法 + 具体方法，随便混 |
| 构造器 | ❌ 没有 | ✅ 有（子类构造器要调 `super()`） |
| 何时用 | 定义**跨界的能力/协议**（`Comparable`、`Serializable`、`Runnable`） | 定义**一组强相关的类型**，想复用部分实现（`AbstractList`、`InputStream`） |

**一句话选型**：**想要"多重继承的效果"就用接口；想"复用代码 + 表达 is-a 关系"就用抽象类**。实际项目里常常是"接口定义契约 + 抽象类提供骨架实现"（这叫**缺省适配器**，如 `AbstractList` 实现 `List`）。

**内部类呢？** 它就是"把类当成成员来用"。为什么需要它？

1. **逻辑上从属**：`HashMap.Node` 只服务于 `HashMap`，没必要单独暴露成一个顶层类
2. **能访问外部类的私有成员**（这是闭包效果）
3. **配合接口做"回调/策略"**：匿名内部类就是最简单的实现方式

**但这里埋着一个 Java 最经典的坑**：**非静态内部类会隐式持有外部类对象的引用**。这意味着，只要内部类对象还活着，外部类对象就无法被回收——这就是 Android/长生命周期场景下**内存泄漏**的常见来源。

### 7.2 原理与源码

#### 7.2.1 接口方法的隐式修饰符

```java
public interface Flyable {
    int MAX_HEIGHT = 10000;          // 隐式 public static final（必须初始化！）

    void fly();                      // 隐式 public abstract

    default void land() {            // JDK 8：默认方法，有实现，实现类可不覆写
        System.out.println("降落");
    }

    static Flyable of() {            // JDK 8：静态方法，只能通过接口名调用
        return () -> System.out.println("飞");
    }

    private void check() {           // JDK 9：私有方法，供 default 方法复用逻辑
        System.out.println("检查");
    }
}
```

**为什么 JDK 8 要引入 `default` 方法？** 这是纯粹的**兼容性问题**：

Java 8 想给 `Collection` 加 `stream()` 方法。如果只加抽象方法，那全世界所有实现了 `Collection` 的类（包括无数第三方库）都会编译失败——因为没实现新方法。加 `default` 方法就解决了：**接口能演进了，老实现类不用改代码**。

> 这个理由（**二进制兼容 / 接口演进**）是面试标准答案，比"为了让接口有方法体"这种描述强得多。

#### 7.2.2 `default` 方法的两个硬限制

**限制 1：不能覆盖 `Object` 的方法**

```java
public interface Bad {
    // default String toString() { return "x"; }   // 编译错误！
    // default boolean equals(Object o) { return true; }  // 编译错误！
}
```

**原因**：`Object` 是所有类的最终父类，接口里定义 `toString`/`equals`/`hashCode` 的默认实现会与"类优先原则"冲突（任何类都从 Object 继承了这些方法）。JLS 明确规定这是编译错误。

**限制 2：多接口 `default` 冲突必须显式解决**

```java
interface A { default void hello() { System.out.println("A"); } }
interface B { default void hello() { System.out.println("B"); } }

class C implements A, B {
    // 不覆写就编译错误：class C inherits unrelated defaults for hello() from A and B
    @Override public void hello() {
        A.super.hello();   // 显式选择要哪个（语法：接口名.super.方法名）
        B.super.hello();
        System.out.println("C");
    }
}
```

**冲突解决的三条优先级规则（背下来）**：

1. **类优先**：如果父类里有具体方法，就用父类的（接口的 default 被忽略）
2. **接口冲突**：如果两个接口都有同签名的 default，且没有更具体的，**必须**在实现类里显式覆写
3. **子接口优先**：如果接口 A 的 default 方法被接口 B（B extends A）覆写了，则用 B 的

#### 7.2.3 内部类的四种形态与"外部引用"问题

```java
public class Outer {
    private int data = 10;

    // ① 成员内部类：持有 Outer.this
    class Inner {
        void show() { System.out.println(data); }        // 能直接访问外部私有字段
    }

    // ② 静态嵌套类：不持有外部引用，本质上等价于一个顶层类
    static class Nested {
        void show() { /* System.out.println(data); */ }  // 不能访问非静态成员
    }

    void method() {
        // ③ 局部内部类：定义在方法里
        class Local { void run() { System.out.println(data); } }
        new Local().run();

        // ④ 匿名内部类：一次性实现
        Runnable r = new Runnable() {
            @Override public void run() { System.out.println("run, data=" + data); }
        };
        r.run();
    }
}
```

**编译后的样子（关键！）**：

```java
// 成员内部类 Outer$Inner 编译后，构造器里会多一个参数：
class Outer$Inner {
    final Outer this$0;              // ← 编译器自动加的字段：外部类引用
    Outer$Inner(Outer outer) {
        this.this$0 = outer;         // ← 构造时把外部对象存进来
    }
    void show() { System.out.println(this.this$0.data); }
}

// 静态嵌套类 Outer$Nested 没有这个字段，所以不持有外部引用
```

> **这就是为什么静态嵌套类不会有内存泄漏问题**。`Map.Entry`、`HashMap.Node` 都是 `static` 的，就是为了不持有外部 `HashMap` 的引用。

**局部/匿名内部类访问局部变量的限制**：

```java
void method() {
    int local = 10;              // 必须是 effectively final（JDK 8+）
    local = 20;                  // ← 加了这行，下面的匿名类就编译报错

    Runnable r = () -> System.out.println(local);
}
```

**为什么？** 因为局部变量在**栈**上，方法结束后就消失了；而匿名内部类的对象可能活得更久（被返回、被放进容器）。所以编译器必须把 `local` 的**值拷贝**一份到匿名类的字段里。如果 `local` 之后还能被改，那"拷贝的值"和"变量的值"就不同步了，语义会混乱——所以语言层面直接禁止修改（JDK 8 之前要求显式 `final`，JDK 8 起放宽为"实际未修改"，即 effectively final）。

#### 7.2.4 内部类导致的经典内存泄漏

```java
public class LeakyOuter {
    private final byte[] bigData = new byte[100 * 1024 * 1024];   // 100MB

    // 非静态内部类 → 持有外部引用 → 下面的任务只要没执行完，100MB 就回收不掉
    public Runnable createTask() {
        return new Runnable() {
            @Override public void run() {
                System.out.println(bigData.length);   // 用了外部字段，无法改成 static
            }
        };
    }

    public static void main(String[] args) {
        LeakyOuter outer = new LeakyOuter();
        Runnable task = outer.createTask();   // task 持有 outer
        outer = null;                          // 即使置空，100MB 依然不释放
        // task 交出去执行后一直不跑完，或者被线程池的队列长期持有 → OOM
    }
}
```

**修复方案**：

| 方案 | 做法 | 适用 |
| --- | --- | --- |
| 改静态嵌套类 | `static class Task`，需要外部数据就作为参数传进来 | 不需要访问外部实例成员时**首选** |
| 传必要数据 | 把 `bigData` 作为参数传入，而不是捕获 `this` | 只用到少量字段 |
| 及时释放 | 任务执行完把引用置 null、从容器移除 | 无法改结构时 |

> **面试话术**：问"内部类为什么可能内存泄漏"，答"**因为非静态内部类（含匿名内部类、Lambda 捕获 this 时）会隐式持有外部类实例的引用（编译后是 `this$0` 字段），只要内部类对象存活，外部对象就无法回收**；解决办法是尽量用静态嵌套类，或只传必要参数"。

#### 7.2.5 Lambda 与匿名内部类的区别（常常连着内部类问）

| 维度 | 匿名内部类 | Lambda |
| --- | --- | --- |
| `this` 的含义 | 指向**匿名类自己的实例** | 指向**外层类的实例** |
| 编译产物 | 生成独立 class 文件（`Outer$1.class`） | `invokedynamic` + 运行时生成（不产生独立 class 文件） |
| 能否有字段/多个方法 | 可以（可以有字段、多个方法） | 不可以（只能实现唯一的抽象方法） |
| 适用接口 | 任意接口/抽象类 | **只能**是函数式接口（SAM） |

```java
public class ThisDemo {
    void test() {
        Runnable anon = new Runnable() {
            @Override public void run() {
                System.out.println(this.getClass());   // class ThisDemo$1  ← 匿名类自己
            }
        };
        Runnable lambda = () -> System.out.println(this.getClass()); // class ThisDemo ← 外层
    }
}
```

### 7.3 面试标准问答

**Q1：接口和抽象类怎么选？**
A：①语义：接口表达"能力/契约"（is-able-to），抽象类表达"是什么"（is-a）；②需要多继承效果 → 接口（类只能单继承）；③需要复用实现 + 维护状态 → 抽象类（接口不能有实例字段）；④实际项目常见组合：接口定契约 + 抽象类给骨架实现（如 `List` + `AbstractList`），这样实现方既能多实现接口，又能选择性复用默认逻辑。

**Q2：JDK 8 的 `default` 方法解决了什么问题？**
A：**接口演进的兼容性**。给已有接口加抽象方法会让所有实现类编译失败；加 `default` 方法后，老实现类无需改动即可获得新能力（`Collection.stream()` 就是这么加进来的）。代价是引入了"菱形继承"式的冲突，需要用"类优先、接口冲突必须显式覆写、子接口优先"三条规则解决。

**Q3：接口能定义方法实现吗？**
A：JDK 8 起可以，三种形式：`default` 方法（实现类可覆写，通过实例调用）、`static` 方法（只能接口名调用）、JDK 9 起的 `private` 方法（仅供接口内部的 default/static 方法复用逻辑）。但**不能有实例字段、不能有构造器**。

**Q4：为什么内部类能访问外部类的私有成员？**
A：编译器的处理：非静态内部类会被加上一个 `this$0` 字段指向外部实例，访问外部私有字段时编译成 `this.this$0.field`。而"访问私有"这个限制是编译期的可见性检查，**同一编译单元内的嵌套类天然可以访问外部类的 private 成员**——JVM 层面通过 `access$xxx` 桥接方法或 `Nestmates`（JDK 11+）支持。

**Q5：内部类会造成内存泄漏吗？**
A：会。非静态成员内部类、匿名内部类、以及捕获了 `this` 的 Lambda，都会隐式持有外部类实例引用。如果内部类对象被长生命周期容器持有（线程池队列、静态集合、单例里的事件监听器），外部对象就无法回收。修复：优先用 `static` 嵌套类、只传必要参数、用完及时解绑。

### 7.4 高频追问与陷阱

#### 追问 1：接口里能写 `default` 方法了，那它和抽象类还有区别吗？

**【追问场景】**
JDK 8 之后接口有了方法体，很多人以为"接口=抽象类了"。面试官在试探你是否理解剩下的本质差异。

**【❌ 错误答法】**
> "没什么区别了，接口现在也能有实现。"
> 错在哪：忽略了**状态**和**构造器**这两条硬边界。接口永远不能有实例字段，也就无法维护对象状态，这是和抽象类最根本的区别。

**【✅ 正确答法】**
> 还有本质区别：
> ①**状态**：接口不能有实例字段（只能 `public static final` 常量），抽象类可以有任意字段并维护状态；
> ②**构造器**：抽象类有构造器（子类需 `super()`），接口没有；
> ③**继承能力**：接口可以多实现，抽象类只能单继承（这是引入 default 的动机之一——用接口获得"多重继承方法"的能力）；
> ④**语义**：接口是能力契约，抽象类是类型模板。
> 所以 `default` 方法解决的是"接口演进不能破坏兼容性"，不是"让接口变成抽象类"。

#### 追问 2：两个接口有同名 `default` 方法，实现类怎么办？

**【追问场景】**
必问。考察你是否知道"必须显式覆写"以及 `X.super.m()` 这个语法。

**【❌ 错误答法】**
> "随便选一个，或者编译能通过、运行时抛异常。"
> 错在哪：编译期就报错，不会留到运行时。Java 在**编译期**强制解决冲突。

**【✅ 正确答法】**
> 如果两个接口提供同签名的 `default` 方法且没有继承关系，实现类**必须重写该方法**，否则编译报错（`inherits unrelated defaults`）。
> 重写时可以用 `接口名.super.方法名()` 显式调用指定的那个，比如 `A.super.hello(); B.super.hello();`，也可以完全自己实现。
> 另外还有优先级规则：**类优先于接口**（父类有具体实现时，接口 default 被忽略）；**子接口优先**（`B extends A` 且 B 覆写了 default，则用 B 的）。
> 如果是抽象方法（非 default）冲突，那只需要实现一次即可，不算冲突。

#### 追问 3：为什么匿名内部类访问的局部变量必须是 `final`（或 effectively final）？

**【追问场景】**
经典题，考察"值捕获"的理解。很多人背了结论，说不出"为什么"。

**【❌ 错误答法】**
> "Java 规定必须 final，没有为什么。"
> 错在哪：这是把语法当答案。面试官真正想听的是"栈上的局部变量会消失，而对象可能活得更久，只能拷贝值"。

**【✅ 正确答法】**
> 因为**生命周期不匹配**。局部变量在栈帧上，方法返回就销毁了；而匿名内部类的对象可能活得更久（被返回、放进集合、交给线程池）。如果内部类直接引用栈上的变量，方法返回后就变成野指针。
> 所以编译器采取的办法是：**把局部变量的值拷贝一份**存进内部类的字段（捕获/capture）。既然只是拷贝，原变量再改就会和拷贝不同步、语义混乱，所以语言规定它不能改——JDK 8 之前要求显式 `final`，之后放宽为"effectively final"（实际未重新赋值）。
> 对比：**成员变量没有这个限制**，因为它存在堆上的对象里，生命周期跟着对象走，内部类可以通过 `this$0` 访问最新值。

#### 追问 4：接口里能定义 `equals` / `toString` 的默认实现吗？

**【追问场景】**
细节题，答出来很加分。它体现你读过 JLS 的相关条款。

**【❌ 错误答法】**
> "能，default 方法什么都能写。"
> 错在哪：JLS 9.2 明确规定，接口不能为 `Object` 的 public 方法提供默认实现。

**【✅ 正确答法】**
> 不能，编译会报错：`default method toString() in interface X overrides a member of java.lang.Object`。
> 原因是 `Object` 是所有类的最终父类，接口的 default 方法与"类优先原则"冲突——如果一个类既继承 `Object` 又实现接口，`toString` 到底用谁？
> 规范的选择是：`Object` 的 public 方法（`equals`、`hashCode`、`toString`）不允许在接口里用 default 提供实现（但可以在接口里**声明**它们为抽象方法，`abstract` 版本是允许的）。

#### 追问 5：静态内部类和普通内部类，选哪个？

**【追问场景】**
工程实践题，考察你有没有踩过内存泄漏。

**【❌ 错误答法】**
> "看习惯，都行。"
> 错在哪：这是个有明确最佳实践的问题——**能用静态就用静态**。

**【✅ 正确答法】**
> **优先用静态嵌套类**。原因：
> ①非静态内部类编译后多一个 `this$0` 字段，持有外部实例引用，容易导致外部对象无法回收（内存泄漏），也增加了对象大小；
> ②创建非静态内部类必须先有外部实例（`outer.new Inner()`），使用不便；
> ③静态嵌套类在没有特殊需求时语义更清晰。
> 只有在**确实需要访问外部实例成员**、且外部对象生命周期与内部对象绑定时，才用非静态内部类。
> 反过来看 JDK 的设计：`HashMap.Node`、`Map.Entry`、`AbstractMap.SimpleEntry` 全是 `static`，就是这个原则的体现。

### 7.5 动手验证

```java
public class InterfaceInnerDemo {
    interface A { default void hello() { System.out.println("A.hello"); } }
    interface B { default void hello() { System.out.println("B.hello"); } }

    static class Base { public void hello() { System.out.println("Base.hello"); } }

    // 类优先：父类有实现，接口 default 就被忽略
    static class ByClass extends Base implements A { }

    // 接口冲突：必须显式覆写
    static class ByInterface implements A, B {
        @Override public void hello() {
            A.super.hello();
            B.super.hello();
        }
    }

    private String secret = "外部私有字段";

    class MemberInner {
        void leak() { System.out.println(secret); }
    }

    static class StaticNested {
        void noLeak() { /* System.out.println(secret); */ }  // 编译不过，正是我们要的
    }

    public static void main(String[] args) {
        new Base().hello();          // Base.hello
        new ByClass().hello();       // Base.hello ← 类优先
        new ByInterface().hello();   // A.hello / B.hello

        InterfaceInnerDemo outer = new InterfaceInnerDemo();
        MemberInner inner = outer.new MemberInner();   // 注意语法：outer.new
        inner.leak();

        // 验证内部类多一个外部引用字段：javap -p InterfaceInnerDemo$MemberInner
    }
}
```

自测内存泄漏：用 `jmap -histo` 或弱引用观察 `outer` 是否被回收：

```java
import java.lang.ref.WeakReference;

InterfaceInnerDemo outer = new InterfaceInnerDemo();
Runnable task = outer.new MemberInner()::leak;   // 捕获了 inner，inner 持有 outer
WeakReference<InterfaceInnerDemo> ref = new WeakReference<>(outer);
outer = null;
System.gc();
System.out.println(ref.get());   // 不是 null —— 因为 task → inner → outer 的强引用链还在
```

### 7.6 一句话速记

- 接口 = 能力契约（可多实现、只有常量、无构造器）；抽象类 = 类型模板（单继承、有状态、有构造器）。
- `default` 的动机是**接口演进不破坏兼容**；冲突三规则：**类优先 → 接口冲突必须显式覆写 → 子接口优先**；不能 default 覆盖 `Object` 的 public 方法。
- 非静态内部类编译后带 `this$0`，**持有外部引用 → 内存泄漏源头**，优先用 `static` 嵌套类。
- 匿名内部类捕获局部变量是**值拷贝**，所以必须 effectively final；Lambda 的 `this` 指向外层类，匿名类的 `this` 指向自己。

---

## 8. 泛型：类型擦除、PECS 与桥接方法

### 8.1 先建立直觉

**泛型的目的是"把类型检查从运行期提前到编译期"**，同时省掉强转。

```java
// 没有泛型：运行时才发现类型不对
List list = new ArrayList();
list.add("abc");
Integer i = (Integer) list.get(0);    // 运行时 ClassCastException

// 有泛型：编译期就报错
List<String> list2 = new ArrayList<>();
list2.add("abc");
// list2.add(1);       // 编译错误，根本进不了字节码
String s = list2.get(0);  // 不需要强转
```

**"类型擦除"是什么意思？** 一句话：**泛型只活在编译期，编译完之后被"擦"掉了**。

**类比**：泛型像贴在产品上的**标签**。编译器盯着标签检查你有没有放错东西；但产品出厂（编译成字节码）后，标签被撕掉，只剩产品本身。

```java
List<String> a = new ArrayList<>();
List<Integer> b = new ArrayList<>();
System.out.println(a.getClass() == b.getClass());   // true！运行时都是 java.util.ArrayList
```

**为什么会这样设计？** 历史原因：Java 5 引入泛型时，必须保证**二进制兼容**——老的 `.class` 文件（没有泛型）和新代码（有泛型）要能在同一个 JVM 里混跑，老代码的反射/`List.class` 引用不能失效。如果泛型是"真实存在"的类型（像 C# 那样保留到运行时），JVM 的类加载、反射、方法签名全都要改。

**代价也很实在**：

- 运行时拿不到泛型类型参数（`new ArrayList<>().getClass()` 没有 `<String>` 信息）
- 不能 `new T()`、不能 `new T[]`、不能 `instanceof List<String>`、静态成员不能用 `T`
- 编译期需要"桥接方法"来维持多态（下面有源码）
- 泛型类型不同的两个方法不能构成重载

### 8.2 原理与源码

#### 8.2.1 擦除规则（两条，必须记准）

| 类型参数形式 | 擦除后的实际类型 |
| --- | --- |
| `<T>`（无界，如 `<T>`） | `Object` |
| `<T extends Number>`（有界） | `Number`（**第一个上界**） |
| `<T extends A & B & C>` | `A`（多个上界时擦成第一个） |

```java
public class Erasure<T, E extends Number> {
    T t;            // 擦除后是 Object t;
    E e;            // 擦除后是 Number e;
    void set(T t) { this.t = t; }         // 擦除后是 void set(Object t)
    void setE(E e) { this.e = e; }        // 擦除后是 void setE(Number e)
    <U extends Comparable<U>> void sort(List<U> list) { }  // U 擦成 Comparable
}
```

用 `javap -p -s` 看方法描述符就能验证：

```text
void set(java.lang.Object);          ← T 没了
void setE(java.lang.Number);         ← E 擦成了上界 Number
Signature: <T:Ljava/lang/Object;E:Ljava/lang/Number;>...   ← 泛型信息还在，但只是"签名属性"
```

> **重要细节**：擦除只擦**类型参数**，泛型信息会以 **`Signature` 属性**的形式保留在 class 文件里。这就是为什么**反射能拿到泛型信息**（`Method.getGenericParameterTypes()`、`Type` 体系），但 `getParameterTypes()` 拿到的只有擦除后的类型。这个区分能拉开面试档次。

#### 8.2.2 桥接方法（Bridge Method）：擦除逼出来的多态补丁

这是泛型里最值得讲清楚的一块。看这段代码：

```java
public class Node implements Comparable<Node> {
    private final int val;
    public Node(int val) { this.val = val; }

    @Override
    public int compareTo(Node other) {   // 注意参数是 Node，不是 Object
        return Integer.compare(this.val, other.val);
    }

    public static void main(String[] args) {
        Comparable<Node> c = new Node(1);
        c.compareTo(new Node(2));    // 编译器生成的是 compareTo(Object) 的调用
    }
}
```

**问题来了**：`Comparable<T>` 接口擦除后的方法签名是 `int compareTo(Object)`，而 `Node` 里写的是 `int compareTo(Node)`——**参数类型不同，根本不是同一个方法，多态就失效了**。

**解决办法：编译器自动生成一个桥接方法**：

```java
// javac 自动生成（反编译可见）
public class Node implements Comparable<Node> {
    // 你以为自己写的
    public int compareTo(Node other) { return Integer.compare(val, other.val); }

    // 编译器补的桥接方法（ACC_BRIDGE | ACC_SYNTHETIC）
    public int compareTo(Object other) {
        return compareTo((Node) other);    // 强转后转调你写的版本
    }
}
```

**验证方式**：

```bash
javap -p -v Node.class | Select-String -Pattern "compareTo|ACC_BRIDGE|ACC_SYNTHETIC"
```

你会看到两个 `compareTo`：一个是 `(LNode;)I`，一个是 `(Ljava/lang/Object;)I` 且带 `ACC_BRIDGE` 标志。

**为什么一定要有它？** 因为调用方（比如 `Collections.sort`）是按 `Comparable` 的擦除签名 `compareTo(Object)` 来调用的。如果没有桥接方法，JVM 在 `Node` 的 vtable 里找不到 `compareTo(Object)` 的实现，就只能调 `Object` 的默认（不存在），多态直接断裂。

**桥接方法带来的两个实际影响**（面试加分点）：

1. **异常检查变松**：桥接方法里做 `(Node)` 强转，如果传错类型会抛 `ClassCastException`——这也是"泛型引入了新的运行时异常"的来源（堆污染 heap pollution）。
2. **反射时方法数量"变多"**：`getDeclaredMethods()` 会返回桥接方法，写框架代码时要 `method.isBridge()` 过滤掉。

#### 8.2.3 PECS 原则

**PECS = Producer Extends, Consumer Super**，由 Joshua Bloch 在《Effective Java》提出。

| 用法 | 含义 | 能做什么 | 不能做什么 |
| --- | --- | --- | --- |
| `List<? extends Animal>` | 只能**读**（生产者） | `Animal a = list.get(0)` ✅ | `list.add(...)` ❌ 编译错误 |
| `List<? super Dog>` | 只能**写**（消费者） | `list.add(new Dog())` ✅ | `Dog d = list.get(0)` ❌ 只能拿到 `Object` |
| `List<Animal>` | 既能读也能写 | 读写都行 | 只能传 `List<Animal>`，`List<Dog>` 不行 |

**为什么 `? extends` 不能 add？** 因为编译器不知道这个 list 的**真实类型**：它可能是 `List<Animal>`、`List<Dog>`、也可能是 `List<Cat>`。如果允许 `add(new Animal())`，而实际是 `List<Cat>`，就破坏了类型安全。所以编译器一律禁止写。

**为什么 `? super` 不能 get 出具体类型？** 因为真实类型可能是 `List<Object>` 或 `List<Animal>`，取出来的东西只能保证是 `Object`。

**记忆口诀**：**读用 extends，写用 super，读写都要就用精确类型**。

```java
// JDK 源码里的经典应用：Collections.copy
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    // dest 是消费者（要写入）→ super；src 是生产者（只读）→ extends
}

// 另一个经典：Collections.addAll
public static <T> boolean addAll(Collection<? super T> c, T... elements)
```

#### 8.2.4 擦除带来的限制（背这个清单）

| 限制 | 例子 | 原因 |
| --- | --- | --- |
| 不能 `new T()` | `T t = new T();` ❌ | 擦除后是 `new Object()`，语义不对 |
| 不能 `new T[]` | `T[] arr = new T[10];` ❌ | 数组需要运行时的具体类型，擦除后只剩 Object |
| 不能 `instanceof T` | `if (o instanceof T)` ❌ | 运行时没有 T 这个类型 |
| 不能 `new List<String>[10]` | ❌ | 泛型数组不允许（会破坏类型安全） |
| 静态成员不能用 `T` | `static T instance;` ❌ | 静态成员属于类，类初始化时 T 还没确定 |
| 不能 `catch (T e)` | ❌ | 擦除后变成 catch Throwable，无法区分 |
| 泛型不同的方法不能重载 | `void f(List<String>)` 和 `void f(List<Integer>)` ❌ | 擦除后签名相同，冲突 |
| 基本类型不能做类型参数 | `List<int>` ❌ | 只能用包装类 |

**绕过"不能 new T[]"的两种写法**：

```java
// 写法 1：强转（会有 unchecked 警告，但运行时能用）
@SuppressWarnings("unchecked")
T[] arr = (T[]) new Object[10];    // 类型擦除后其实是 Object[]，如果当 String[] 用会 ClassCastException

// 写法 2：传 Class<T> 参数，用反射创建（类型安全）
static <T> T[] createArray(Class<T> clazz, int size) {
    return (T[]) java.lang.reflect.Array.newInstance(clazz, size);   // 真正创建出 T[]
}
```

#### 8.2.5 堆污染（Heap Pollution）与 `@SafeVarargs`

```java
public class HeapPollution {
    @SafeVarargs   // 承诺方法内部不会对可变参数做不安全的操作
    static <T> List<T> of(T... args) {
        return Arrays.asList(args);   // 这里 args 实际是 Object[]，被当作 T[] 用
    }

    public static void main(String[] args) {
        List<String> list = of("a", "b");   // 看起来没问题

        // 但如果有人这样绕过：
        Integer[] ints = {1, 2};
        List<Object> bad = of(ints);        // 类型不安全，@SafeVarargs 是在"骗编译器"
    }
}
```

**堆污染的典型场景**：`@SafeVarargs` 只能用在 `static`/`final`/`private`/构造器上，因为只有这些方法不能被继承重写，才能保证"方法内部不会把错误的类型放进数组"。**这是泛型 + 可变参数的经典缺陷**。

### 8.3 面试标准问答

**Q1：什么是类型擦除？为什么 Java 要这么做？**
A：类型擦除指泛型类型参数在编译后被替换为上界（无界擦成 `Object`，有界擦成第一个上界），并插入必要的强制类型转换和桥接方法。原因：Java 5 引入泛型时要**兼容已有的 `.class` 文件和持久化数据**，如果泛型保留到运行时，JVM 的类加载、方法签名、反射都要改，无法兼容。代价是运行时拿不到类型参数信息（但 `Signature` 属性保留了泛型元数据，反射可取）。

**Q2：`List<String>` 和 `List<Integer>` 是同一个类吗？**
A：是。运行时都是 `java.util.ArrayList`，`getClass() == getClass()` 为 true。所以不能靠 `instanceof List<String>` 判断（编译错误），也不能用它们重载方法。

**Q3：什么是桥接方法？为什么需要它？**
A：当子类实现泛型接口/父类时，如果子类方法用的具体类型与擦除后的签名不同（如 `compareTo(Node)` vs `compareTo(Object)`），编译器会生成一个"桥接方法"，参数是擦除后的 `Object`，内部强转为具体类型后调用真正的方法，并标记 `ACC_BRIDGE | ACC_SYNTHETIC`。**目的是保证多态在擦除后依然成立**——调用方按接口签名调 `compareTo(Object)` 时能正确落到子类的实现上。副作用：`getDeclaredMethods()` 会返回它，框架代码要用 `isBridge()` 过滤。

**Q4：PECS 是什么？为什么 `? extends` 不能 add？**
A：Producer Extends, Consumer Super——只读（生产者）用 `? extends T`，只写（消费者）用 `? super T`。`? extends` 不能 add 是因为编译器不知道 list 的真实类型（可能是任意子类型），允许写入会破坏类型安全；反过来 `? super` 不能 get 出具体类型，只能拿到 `Object`。

**Q5：为什么不能 `new T()`？**
A：擦除后 `T` 变成 `Object`（或有界类型的上界），`new T()` 会被理解为 `new Object()`，语义完全不对。需要创建实例时应该传 `Class<T>` 参数，用 `clazz.getDeclaredConstructor().newInstance()` 反射创建；或者传 `Supplier<T>` 工厂函数。

**Q6：泛型信息和反射的关系？**
A：擦除后类型参数保留在 class 文件的 `Signature` 属性里，所以：
`getParameterTypes()` → 擦除后的 `Class`（如 `Object`）
`getGenericParameterTypes()` → 带泛型的 `Type`（如 `T`、`List<String>`）
这就是 Spring 能根据 `ParameterizedType` 判断 `List<User>` 中 `User` 类型的原因（依赖注入、Jackson 反序列化都靠它）。

### 8.4 高频追问与陷阱

#### 追问 1：`List<String>` 和 `List<Integer>` 在运行时是同一个类，那为什么 `new ArrayList<String>()` 之后不能 `add(1)`？

**【追问场景】**
看似矛盾的问题，考"编译期检查"和"运行时擦除"的分工。答不上来说明没理解擦除的边界。

**【❌ 错误答法】**
> "因为 ArrayList 内部保存了类型参数信息。"
> 错在哪：**运行时压根没有类型参数信息**。拦住 `add(1)` 的是编译器，不是运行时。

**【✅ 正确答法】**
> 因为**类型检查是编译器做的，擦除是编译之后的事**。编译阶段编译器看到了 `List<String>` 这个声明，就按它检查每个 `add` 的参数；一旦通过检查，生成的字节码里只有 `List.add(Object)`，类型参数被擦掉了。
> 所以 `new ArrayList<String>()` 不能 `add(1)` 是**编译错误**，不是运行时异常；而如果你绕过编译器（用反射或原始类型 `List` 引用），运行时完全不会拦你——这正是后面会抛 `ClassCastException` 的根源。
> 一句话：**泛型是编译期的类型护栏，不是运行期的类型容器**。

#### 追问 2：为什么会有桥接方法？能说出它长什么样吗？

**【追问场景】**
中高级必问。面试官想确认你不是只知道"有这个词"。

**【❌ 错误答法】**
> "就是泛型生成的一个额外方法，具体不太清楚。"
> 错在哪：没抓住核心——**它是为了补上擦除造成的多态断裂**。

**【✅ 正确答法】**
> 桥接方法是编译器为了修复"擦除导致的多态断裂"而自动生成的。
> 场景：类 `Node implements Comparable<Node>`，你写的是 `compareTo(Node)`，但接口擦除后签名是 `compareTo(Object)`，两者参数类型不同，JVM 的虚方法表里根本对不上，多态就断了。
> 编译器于是生成一个 `public int compareTo(Object o) { return compareTo((Node) o); }`，标记为 `ACC_BRIDGE | ACC_SYNTHETIC`，这样调用方按 `compareTo(Object)` 调用时能正确转到你的实现。
> 验证：`javap -p -v` 能看到两个 `compareTo`，一个参数是 `Node`、一个参数是 `Object` 且带 `ACC_BRIDGE`。
> 影响：反射 `getDeclaredMethods()` 会多出这个方法（要用 `isBridge()` 过滤）；桥接方法里的强转也是 `ClassCastException` 的来源之一。

#### 追问 3：`? extends T` 和 `? super T` 分别能做什么、不能做什么？

**【追问场景】**
PECS 必问。面试官常给一个"复制集合"的签名让你写，看你能不能正确选择通配符。

**【❌ 错误答法】**
> "两个都是用来表示任意子类型，区别不大。"
> 错在哪：完全不知道读写限制。这正是 PECS 的全部内容。

**【✅ 正确答法】**
> 关键在**读写方向的限制**：
> `List<? extends Animal>` 是**生产者**：可以 `get()` 出来赋给 `Animal`（因为元素一定是 Animal 的子类型），但**不能 add**——编译器不知道真实类型是 `List<Dog>` 还是 `List<Cat>`，写进去可能破坏类型安全。
> `List<? super Dog>` 是**消费者**：可以 `add(new Dog())`（因为它一定能装 Dog 的父类型），但 `get()` 只能拿到 `Object`（真实类型可能是 `List<Object>`）。
> 口诀：**Producer Extends, Consumer Super**。
> 典型应用是 JDK 的 `Collections.copy(List<? super T> dest, List<? extends T> src)`——dest 是写入方所以用 super，src 是读取方所以用 extends。自己写工具方法时照这个模式套即可。

#### 追问 4：为什么不能创建泛型数组 `new List<String>[10]`？

**【追问场景】**
细节题。答"因为擦除"是对的但不够，面试官会追问"那 `new ArrayList[10]` 为什么可以"。

**【❌ 错误答法】**
> "Java 不支持泛型数组，没为什么。"
> 错在哪：给不出机制解释。真实原因是**数组的运行时类型检查和泛型的编译期检查会打架**。

**【✅ 正确答法】**
> 因为**数组是"协变 + 运行时类型检查"，泛型是"不变 + 编译期检查"**，两者组合会破坏类型安全。
> 数组协变：`Object[] arr = new String[1];` 合法，但 `arr[0] = 1;` 会在**运行时**抛 `ArrayStoreException`——数组在运行时知道自己元素的真实类型。
> 如果允许 `List<String>[] arr = new List<String>[10]`，擦除后运行时只知道这是 `List[]`，它无法区分 `List<String>` 和 `List<Integer>`。于是就能写出 `Object[] o = arr; o[0] = new ArrayList<Integer>();` 这种把 `Integer` 列表塞进"String 列表数组"的操作，而运行时**检查不出来**，等到取值时才 `ClassCastException`——这就是"堆污染"，Java 选择在编译期直接禁止。
> 而 `new ArrayList[10]` 可以是因为它是**原始类型数组**，不带泛型参数，只会有 `unchecked` 警告。需要泛型数组时用 `(T[]) new Object[n]` 强转 + `@SuppressWarnings`，或者用 `Array.newInstance(clazz, n)` 反射创建真正类型化的数组。

#### 追问 5：`<T extends Comparable<T>>` 和 `<T extends Comparable>` 有什么区别？

**【追问场景】**
进阶题，考对"递归类型边界"的理解。写排序工具类时会撞上这个问题。

**【❌ 错误答法】**
> "一样，都是要求 T 实现 Comparable。"
> 错在哪：丢掉了类型约束的精度。裸 `Comparable` 相当于 `Comparable<Object>`，表达能力退化。

**【✅ 正确答法】**
> 差别在**约束精度**：
> `<T extends Comparable>` 是原始类型用法，擦除后上界是 `Comparable`，编译器只能保证 T 实现了 Comparable，但比较的对方类型未知，实际等价于 `Comparable<Object>`，容易产生 unchecked 警告和错误比较。
> `<T extends Comparable<T>>` 是**递归类型边界（F-bounded）**，明确要求"T 能和自己比较"，这才是 `Collections.sort`、`TreeMap` 真正需要的约束：
> ```java
> public static <T extends Comparable<? super T>> void sort(List<T> list)
> ```
> JDK 实际用的是 `Comparable<? super T>`，比 `Comparable<T>` 更宽松——允许子类继承父类的比较逻辑（比如 `Dog extends Animal`，`Animal implements Comparable<Animal>`，此时 `Dog` 也能被排序）。这个 `? super` 是实践中的关键细节。

### 8.5 动手验证

```java
import java.lang.reflect.Method;
import java.util.*;

public class GenericsDemo {
    // 桥接方法演示
    static class Node implements Comparable<Node> {
        final int v;
        Node(int v) { this.v = v; }
        @Override public int compareTo(Node o) { return Integer.compare(v, o.v); }
    }

    // 反射读泛型信息
    static class Repo<T> {
        List<String> names;                       // 字段上的泛型
        void save(List<Integer> ids) { }          // 参数上的泛型
    }

    public static void main(String[] args) throws Exception {
        // ① 擦除：运行时同一个类
        List<String> a = new ArrayList<>();
        List<Integer> b = new ArrayList<>();
        System.out.println("同一个类? " + (a.getClass() == b.getClass()));   // true

        // ② 桥接方法
        System.out.println("--- Node 的方法列表 ---");
        for (Method m : Node.class.getDeclaredMethods()) {
            System.out.println(m + "  bridge=" + m.isBridge() + " synthetic=" + m.isSynthetic());
        }
        // 会看到 compareTo(Node) 和 bridge 的 compareTo(Object)

        // ③ 泛型信息在 Signature 里，反射仍可拿到
        System.out.println("--- 泛型元数据 ---");
        System.out.println(Repo.class.getDeclaredField("names").getGenericType());        // java.util.List<java.lang.String>
        System.out.println(Repo.class.getDeclaredMethod("save", List.class).getGenericParameterTypes()[0]); // List<Integer>

        // ④ 绕过编译器的类型不安全（堆污染演示）
        List raw = a;
        raw.add(123);                                     // 编译通过（原始类型，unchecked 警告）
        try {
            String s = a.get(0);                          // 取出来强转 String → ClassCastException
        } catch (ClassCastException e) {
            System.out.println("堆污染后果: " + e.getMessage());
        }
    }
}
```

命令行验证桥接方法：

```bash
javap -p -v -cp . GenericsDemo$Node | Select-String -Pattern "compareTo|ACC_BRIDGE"
```

### 8.6 一句话速记

- 泛型**只在编译期存在**，擦除规则：无界 → `Object`，有界 → **第一个上界**；泛型元数据留在 `Signature` 属性里（反射可取）。
- **桥接方法**是擦除逼出来的多态补丁（`ACC_BRIDGE`），反射时要用 `isBridge()` 过滤。
- PECS：**读 extends、写 super**；`? extends` 不能 add，`? super` 只能 get 出 Object。
- 不能 `new T()`/`new T[]`/`instanceof T`/静态成员用 T；需要创建实例就传 `Class<T>` 或 `Supplier<T>`。

---

## 9. 异常体系与 try-with-resources

### 9.1 先建立直觉

**异常的两大阵营，本质区别是"编译器要不要逼你处理"**：

| | 受检异常（Checked） | 非受检异常（Unchecked） |
| --- | --- | --- |
| 继承自 | `Exception`（不含 RuntimeException 分支） | `RuntimeException` |
| 代表 | `IOException`、`SQLException`、`ClassNotFoundException` | `NullPointerException`、`IllegalArgumentException`、`IndexOutOfBoundsException` |
| 编译器要求 | **必须** try-catch 或 throws 声明 | 不强制 |
| 语义 | "**预期内的外部风险**"——文件可能不存在、网络可能断 | "**程序写错了**"——空指针、越界、参数非法 |
| 谁来处理 | 调用方（上层业务决定怎么兜底） | 不该被捕获，应该修 bug |

**为什么 Java 要分两类？** 设计初衷是"让编译器帮你检查那些**必须处理**的风险"。比如读文件，你不可能假设文件一定存在，所以 `IOException` 强制你处理。

**但这个设计后来被广泛批评**（包括 Java 之父 James Gosling 也承认过争议），因为它导致：

- 到处都是 `try { } catch (Exception e) { }` 的"吞异常"代码
- Lambda/Stream 里根本没法用受检异常，只能包一层 `try-catch` 转成 RuntimeException（这直接催生了各种 `UncheckedIOException`）

**实践结论**：**能用非受检异常就别用受检**（现代框架基本都是这个取向，Spring 的 `DataAccessException` 全是 unchecked），只有在"调用方确实能并且应该处理"的场景才用受检异常。

**`try-with-resources` 解决什么？** 它解决 `finally` 里关资源的三个老大难：

1. 忘写 `close()`（资源泄漏）
2. `close()` 自己抛异常，覆盖了业务异常（**异常吞噬**）
3. 嵌套多个资源时 `finally` 层层嵌套，代码难读

### 9.2 原理与源码

#### 9.2.1 异常体系全图

```text
Throwable（可以被 throw 的根）
├── Error                          ← 不该捕获：JVM 层面的严重问题
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── NoClassDefFoundError
└── Exception
    ├── RuntimeException           ← 非受检（Unchecked）
    │   ├── NullPointerException
    │   ├── IllegalArgumentException
    │   ├── IllegalStateException
    │   ├── IndexOutOfBoundsException
    │   ├── ClassCastException
    │   └── NumberFormatException（注意：它是 IllegalArgumentException 的子类！）
    └── 其他 Exception             ← 受检（Checked）
        ├── IOException
        │   └── FileNotFoundException
        ├── SQLException
        ├── ClassNotFoundException
        └── InterruptedException
```

> **易错点**：`NumberFormatException` 继承自 `IllegalArgumentException`，**是 unchecked**。很多人以为它和解析有关就当成 checked。

#### 9.2.2 `try-with-resources` 的编译结果与 suppressed 机制

```java
// 你写的
try (InputStream in = new FileInputStream("a.txt");
     OutputStream out = new FileOutputStream("b.txt")) {
    in.transferTo(out);
}
```

```java
// 编译器实际生成（简化版）
InputStream in = new FileInputStream("a.txt");
Throwable primaryEx = null;
try {
    OutputStream out = new FileOutputStream("b.txt");
    Throwable outEx = null;
    try {
        in.transferTo(out);
    } catch (Throwable t) {
        outEx = t; throw t;
    } finally {
        if (out != null) {
            if (outEx != null) {
                try { out.close(); }
                catch (Throwable t) { outEx.addSuppressed(t); }   // ← 关键：抑制而不是覆盖
            } else {
                out.close();
            }
        }
    }
} catch (Throwable t) {
    primaryEx = t; throw t;
} finally {
    if (in != null) {
        if (primaryEx != null) {
            try { in.close(); } catch (Throwable t) { primaryEx.addSuppressed(t); }
        } else {
            in.close();
        }
    }
}
```

**三个要点**：

1. **关闭顺序是"反序"**：先关 `out`（后声明的），再关 `in`。符合"后进先出"，因为后打开的往往依赖先打开的。
2. **`addSuppressed` 而不是覆盖**：如果业务代码抛了异常、`close()` 也抛异常，`close` 的异常会被**挂到主异常的 suppressed 列表**里，主异常不会被吞。可以用 `e.getSuppressed()` 查看。
3. **`AutoCloseable` 的 `close()` 声明为 `throws Exception`**，而 `Closeable`（`java.io`）的 `close()` 是 `throws IOException`——**优先用 `Closeable`**，异常类型更精确。

```java
// AutoCloseable（JDK 7 引入，泛用）
public interface AutoCloseable {
    void close() throws Exception;
}

// Closeable（JDK 5 就有，IO 专用）
public interface Closeable extends AutoCloseable {
    void close() throws IOException;   // 更窄，所以更好用
}
```

#### 9.2.3 `finally` 与 `return` 的相爱相杀

```java
static int f1() {
    try { return 1; }
    finally { System.out.println("finally 执行了"); }   // 会执行，然后返回 1
}

static int f2() {
    try { return 1; }
    finally { return 2; }        // 危险！返回值被改成 2，1 被抛弃
}

static int f3() {
    int x = 1;
    try { return x; }
    finally { x = 99; }          // 返回值依然是 1（return 时已把 1 存到返回值槽）
}

static int f4() {
    try { throw new RuntimeException("业务异常"); }
    finally { return 1; }        // 最危险！异常被彻底吞掉，方法正常返回 1
}
```

**字节码层面的原因**：`return` 会先把值存进**返回值槽位**，然后跳转到 `finally` 块；`finally` 执行完再统一返回。所以：

- `finally` 里的 `return` 会**覆盖**已存好的返回值
- `finally` 里的 `return` 会**吞掉**正在传播的异常（因为 return 直接结束了方法，异常传播路径被切断）
- `finally` 里修改局部变量**不影响**已存入返回值槽的值

> **工程结论**：**永远不要在 `finally` 里写 `return`/`break`/`continue`/`throw`**。IDE 和静态检查工具（如 Sonar）都会报这个错。

#### 9.2.4 异常的性能代价：`fillInStackTrace`

```java
// Throwable 构造器
public Throwable() {
    fillInStackTrace();     // 关键：抓取当前线程的完整调用栈
}

public synchronized Throwable fillInStackTrace() {
    // native：遍历栈帧、记录每个方法的类名/方法名/行号/文件
}
```

**代价**：每次 `new Exception()` 都要**遍历整个调用栈**并生成 `StackTraceElement[]`。在循环里抛异常可能让性能下降 1~2 个数量级。

**优化手段**：

```java
// 不关心堆栈的异常（如用于流程控制、单例兜底），可以关掉栈采集
public class NoStackTraceException extends RuntimeException {
    @Override public synchronized Throwable fillInStackTrace() {
        return this;    // 直接返回自己，不采集栈
    }
}
```

> **注意这是双刃剑**：关掉之后排查问题会失去行号信息，只适合"已知无排查价值"的场景（如高频的幂等冲突）。

### 9.3 面试标准问答

**Q1：受检异常和非受检异常的区别？怎么选？**
A：受检异常继承 `Exception`（不含 RuntimeException 分支），编译器强制 try-catch 或 throws；非受检继承 `RuntimeException`，不强制。选型原则：**"调用方可以并且应该做不同处理"的用受检**（如支付接口的余额不足、文件不存在）；**"程序 bug 或不可恢复"的用非受检**（空指针、参数非法）。现代实践倾向少用受检异常——因为它会污染方法签名、在 Lambda 中无法直接使用、且容易导致"catch 后什么都不做"的坏代码。

**Q2：`try-with-resources` 相比 `finally` 好在哪？**
A：①自动关闭，不会忘记；②**避免异常吞噬**：主异常和 `close()` 异常同时发生时，`close` 的异常被 `addSuppressed` 挂到主异常上，而不是覆盖；③多个资源自动按**反序**关闭；④代码更简洁。前提是资源实现 `AutoCloseable`（或 `Closeable`）。

**Q3：`finally` 一定会执行吗？**
A：绝大多数情况会，但有四种例外：①`System.exit()` 或 `Runtime.halt()`；②JVM 崩溃（断电、OOM 导致进程被杀）；③线程被 `kill -9`；④`try` 块里进入死循环且无法跳出。另外 `daemon` 线程在主线程结束时被强制终止，也可能不执行 `finally`。

**Q4：`finally` 里 `return` 会怎样？**
A：会覆盖 `try` 里的返回值，**并且会吞掉正在传播的异常**（因为 `return` 直接结束方法，异常无法继续抛出）。这是严重 bug 来源，属禁止写法。

**Q5：异常的代价在哪里？**
A：主要在 `fillInStackTrace()`——构造 `Throwable` 时要遍历调用栈生成 `StackTraceElement[]`，成本与栈深度成正比。所以：不要在循环里用异常做流程控制；对高频且不需要堆栈的异常可以重写 `fillInStackTrace()` 返回 `this`。

**Q6：`catch` 多个异常时要注意什么？**
A：①**子类必须写在父类前面**，否则编译报错（父类捕获后子类分支永远不可达）；②JDK 7+ 支持 multi-catch：`catch (IOException | SQLException e)`，此时 `e` 隐式为 `final`；③**不要 `catch (Exception e)` 或 `catch (Throwable t)`** 兜底然后什么都不做，那会掩盖所有问题（包括 `OutOfMemoryError` 这类不该捕获的）。

### 9.4 高频追问与陷阱

#### 追问 1：`finally` 里的 `return` 会怎样？

**【追问场景】**
经典送命题。面试官给一段既有 `try return` 又有 `finally return`、还有异常抛出的代码，让你说结果。

**【❌ 错误答法】**
> "会返回 try 里的值，finally 只是执行一下。"
> 错在哪：不成立。`finally` 的 `return` 会**覆盖**返回值槽，而且能吞掉异常。

**【✅ 正确答法】**
> 有两个后果，都很危险：
> ①**返回值被覆盖**：`try { return 1; } finally { return 2; }` 实际返回 **2**，`try` 里的 1 被丢弃；
> ②**异常被吞掉**：`try { throw new RuntimeException(); } finally { return 1; }` 方法会**正常返回 1**，异常完全消失，调用方永远不知道出过错。
> 原因是字节码执行顺序：`return` 先把值存入返回值槽 → 跳去执行 `finally` → `finally` 的 `return` 又写了一次返回值槽并直接结束方法。
> 所以规范明确禁止在 `finally` 里写 `return`/`break`/`continue`/`throw`，静态扫描工具会直接报错。

#### 追问 2：`try-with-resources` 里如果 `close()` 也抛异常，会发生什么？

**【追问场景】**
比"能不能自动关闭"深一层，考"异常抑制（suppressed）"这个机制。能答出 `addSuppressed` 就是高分。

**【❌ 错误答法】**
> "会覆盖原来的异常，或者两个异常都抛出来。"
> 错在哪：Java 方法只能抛一个异常，不可能"都抛"。老式 `finally` 写法确实会**覆盖**主异常（这是 try-with-resources 要解决的核心痛点）。

**【✅ 正确答法】**
> 主异常会被保留，`close()` 抛出的异常被**抑制**——通过 `Throwable.addSuppressed()` 挂到主异常的 `suppressedExceptions` 列表里。
> 所以捕获到的还是业务异常，需要时可以用 `e.getSuppressed()` 拿到被抑制的关闭异常，排查资源释放问题。
> 这恰好修复了传统 `finally { close(); }` 的缺陷：那种写法下 `close` 抛异常会**覆盖**业务异常，导致真正的原因丢失。
> 此外关闭顺序是**反序**（后声明的先关），符合依赖关系。

#### 追问 3：`finally` 一定执行吗？

**【追问场景】**
看严谨度。答"一定"就是被扣分的开始。

**【❌ 错误答法】**
> "一定执行，这是 finally 的保证。"
> 错在哪：绝对化了。有若干场景 `finally` 不会执行。

**【✅ 正确答法】**
> 绝大多数情况会执行（包括 `try` 里 `return` 或抛异常时），但有例外：
> ①`System.exit()` 或 `Runtime.getRuntime().halt()` —— 直接终止 JVM；
> ②JVM 崩溃：OOM 导致进程被系统杀掉、断电、`kill -9`；
> ③线程被强制终止（`Thread.stop()`，已废弃）；
> ④`try` 块里死循环出不来。
> 另外注意：`daemon` 线程在主线程结束后会被 JVM 直接终止，它的 `finally` 也可能不执行——**所以别把关键清理逻辑放在 daemon 线程的 finally 里**。

#### 追问 4：受检异常在 Lambda / Stream 里为什么这么难用？

**【追问场景】**
工程题。写过 `stream().map(x -> { try {...} catch ...})` 的人都会有痛感，面试官想看你是否理解"函数式接口签名不能抛受检异常"。

**【❌ 错误答法】**
> "不难用，包一层 try-catch 就行。"
> 错在哪：把痛点当成了解决方案。包一层之后就失去了 Stream 的链式表达力，而且会把受检异常转成 unchecked，语义发生变化。

**【✅ 正确答法】**
> 根因是**函数式接口（如 `Function`）的抽象方法签名不允许声明受检异常**：
> ```java
> @FunctionalInterface public interface Function<T, R> { R apply(T t); }   // 没有 throws
> ```
> 而受检异常要求要么 catch 要么在签名里 throws，Lambda 无法声明 throws，于是只能就地 catch，把异常转成 RuntimeException 抛出（如 `UncheckedIOException`、`CompletionException` 就是这么来的）。
> 通用解法有三种：①写一个包装工具方法，把受检异常转成 unchecked（`@FunctionalInterface ThrowingFunction` + `try/catch`）；②用 `vavr` 之类的库提供的 `Try`；③**在流的入口就把异常处理好，不要把 IO 放进流里**（更推荐，因为流本来就不适合做有副作用的 IO）。
> 这也是 Java 社区对"受检异常设计"批评的核心点之一。

#### 追问 5：自定义异常应该继承什么？

**【追问场景】**
工程规范题。答案体现你对异常分层设计的理解。

**【❌ 错误答法】**
> "继承 Exception 就行，或者继承 RuntimeException 也行，看心情。"
> 错在哪：没有分层意识，也不考虑框架对异常的处理约定。

**【✅ 正确答法】**
> 分两类来设计：
> ①**业务异常**（可预期的业务规则失败，如"余额不足"）→ 继承 `RuntimeException`，因为它是流程分支而不是系统故障，用 unchecked 避免污染所有方法签名；通常设计一个 `BusinessException`（带错误码 + 消息）作为基类统一处理。
> ②**系统异常**（不可预期的技术故障，如调用第三方超时）→ 可以用受检异常，强制调用方处理；但实践中也常包装成 unchecked 后由全局异常处理器兜底。
> 设计要点：**要有统一基类 + 错误码**，配合 `@RestControllerAdvice` 做全局映射；异常里不要只存 message，要存上下文（如订单号）；**不要吞异常**（至少要 log 并保留 cause）。

### 9.5 动手验证

```java
public class ExceptionDemo {
    // finally 覆盖返回值
    static int f1() { try { return 1; } finally { return 2; } }

    // finally 吞异常
    static int f2() {
        try { throw new RuntimeException("业务异常"); }
        finally { return 1; }              // 异常消失
    }

    // try-with-resources 的抑制异常
    static class Res implements AutoCloseable {
        final String name;
        Res(String name) { this.name = name; }
        @Override public void close() {
            System.out.println(name + " 关闭");
            throw new IllegalStateException(name + " 关闭失败");
        }
    }

    public static void main(String[] args) {
        System.out.println("f1() = " + f1());   // 2
        System.out.println("f2() = " + f2());   // 1（异常被吞）

        try (Res r1 = new Res("r1"); Res r2 = new Res("r2")) {
            throw new RuntimeException("业务异常");
        } catch (Exception e) {
            System.out.println("主异常: " + e.getMessage());          // 业务异常
            for (Throwable s : e.getSuppressed()) {
                System.out.println("  suppressed: " + s.getMessage()); // r2/r1 关闭失败
            }
        }
        // 注意关闭顺序：r2 先关，r1 后关（反序）
    }
}
```

### 9.6 一句话速记

- **受检 = 编译器逼你处理的外部风险**（IOException），**非受检 = 程序 bug**（NPE）；`NumberFormatException` 是 unchecked。
- `try-with-resources` 三件套：自动关闭、**反序**关闭、用 `addSuppressed` 避免异常吞噬。
- **禁止在 `finally` 里 return**：会覆盖返回值 + 吞掉异常；`finally` 也不是"一定执行"（`System.exit`、JVM 崩溃、`kill -9` 例外）。
- 异常成本在 `fillInStackTrace`：别用异常做流程控制，高频异常可重写该方法。

---

## 10. 反射、注解与动态代理

### 10.1 先建立直觉

**反射是什么？** 一句话：**让你在运行时"看见"并"操作"类的结构**。

正常写代码时，编译器替你查表：你知道 `user.getName()` 存在，因为编译期就确定了。但有些场景你不知道：

- Spring 要给你标注了 `@Autowired` 的字段**注入依赖**——它不知道你的类长什么样，只能运行时扫描
- MyBatis 要把查询结果**塞进你的实体对象**——同样的道理
- JSON 序列化要把对象转成字段名 → 值的 map

**类比的洗牌**：正常代码像"按图纸盖房子"，每块砖在哪都写死了；反射像"拆开房子的检查员"，可以现场翻看有哪些房间、把里面的东西搬出来换掉。

代价是：**编译器帮不上忙了**。所以反射的问题都是运行时的——名字拼错、类型不对、权限不够，全要到运行时才炸。

**动态代理是什么？** 一句话：**运行时"造"一个实现了指定接口的替身类**。所有方法调用都转给 `InvocationHandler` 处理。

```
调用方 ──→ [动态生成的 $Proxy0 对象] ──→ InvocationHandler.invoke(proxy, method, args) ──→ 真实对象
                ↑
        编译器没写过这个类，是运行时生成的
```

**为什么需要它？** 因为很多能力是"横切"的：事务、日志、权限、缓存、限流。这些逻辑和业务代码无关，但又必须环绕在业务方法前后。如果每个方法都手写一遍 `try { tx.begin(); 业务(); tx.commit(); }`，代码会烂掉。动态代理让**容器**帮你把这些逻辑织进去——这就是 Spring AOP 的实现基础。

### 10.2 原理与源码

#### 10.2.1 拿到 `Class` 对象的三种方式

```java
Class<?> c1 = String.class;                    // ① 类字面量（编译期，不会触发类初始化）
Class<?> c2 = "abc".getClass();                // ② 对象.getClass()（已有对象）
Class<?> c3 = Class.forName("java.lang.String");// ③ 反射（最常用，会触发类初始化）
```

**关键差异（高频考点）**：

```java
// ① 类字面量不会触发类初始化！
Class<?> c = MyClass.class;    // 只加载类，不执行静态块
```

#### 10.2.2 `Class.forName` vs `ClassLoader.loadClass`（源码级区别）

```java
// Class.forName：默认 initialize = true
public static Class<?> forName(String className) throws ClassNotFoundException {
    Class<?> caller = Reflection.getCallerClass();
    return forName0(className, true, ClassLoader.getClassLoader(caller), caller);
    //                       ↑ 关键：true = 要初始化
}

// ClassLoader.loadClass：默认不初始化
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    // ... 双亲委派查找 ...
    if (resolve) resolveClass(c);    // resolve 是"链接"阶段的解析，不是初始化
    return c;
}
```

| 方法 | 是否触发**类初始化**（执行静态块） | 典型场景 |
| --- | --- | --- |
| `Class.forName(name)` | ✅ 是（initialize 默认 true） | JDBC 驱动注册：`Class.forName("com.mysql.cj.jdbc.Driver")`，依赖静态块注册驱动 |
| `Class.forName(name, false, loader)` | ❌ 否 | 只想检查类是否存在 |
| `ClassLoader.loadClass(name)` | ❌ 否 | Spring 扫描类时用，避免过早初始化 |

> **这条几乎是必问题**："JDBC 为什么用 `Class.forName` 而不是 `loadClass`？"——因为驱动类里有个静态块 `DriverManager.registerDriver(new Driver())`，必须**触发类初始化**才能注册驱动。

#### 10.2.3 `getXxx` 与 `getDeclaredXxx` 的区别（一字之差，天壤之别）

| 方法对 | 能拿到什么 |
| --- | --- |
| `getMethods()` | 所有 **public** 方法，**含从父类/接口继承的** |
| `getDeclaredMethods()` | 本类声明的**所有**方法（public/protected/private），**不含继承** |
| `getFields()` / `getDeclaredFields()` | 同上规则 |
| `getConstructors()` / `getDeclaredConstructors()` | 同上规则 |

```java
class Parent { public void a() {} }
class Child extends Parent {
    private void b() {}
    public void c() {}
}

Child.class.getMethods().length;           // 含 a()、c()、Object 的所有 public 方法
Child.class.getDeclaredMethods().length;   // 只有 b()、c()（本类声明的）
```

> **框架代码的典型坑**：想读一个类的所有字段，用了 `getFields()`，结果 `private` 字段一个都读不到。正确做法是 `getDeclaredFields()` + `setAccessible(true)`。

#### 10.2.4 `Method.invoke` 为什么慢？inflation 机制

```java
public Object invoke(Object obj, Object... args) throws ... {
    // ... 权限检查、参数检查 ...
    return methodAccessor.invoke(obj, args);   // ← 真正的调用入口
}

// MethodAccessor 有两个实现：
// ① NativeMethodAccessorImpl：通过 JNI 调用，启动快但每次调用都慢
// ② GeneratedMethodAccessor：动态生成的字节码，调用快但生成有开销
```

**inflation 机制（膨胀阈值）**：

```
同一个 Method 前 15 次调用 → 用 NativeMethodAccessorImpl（JNI）
第 16 次开始 → 生成 GeneratedMethodAccessor 字节码，之后走快速路径
```

阈值由 `-Dsun.reflect.inflationThreshold=15` 控制（部分 JDK 版本可调），也可以 `-Dsun.reflect.noInflation=true` 直接生成。

**所以"反射慢"要分两层说**：

1. **第一次调用慢**：要生成代理/访问器
2. **持续调用仍有开销**：参数装箱成 `Object[]`、每次都要做类型检查、无法被 JIT 内联（vtable 查表而不是直接调用）

**优化手段**：缓存 `Method`/`Field` 对象（不要每次 `getDeclaredMethod`）、用 `MethodHandle`（JDK 7+，可被 JIT 优化）、或直接用 `LambdaMetafactory` 生成函数式接口调用（最快，很多框架的终极优化手段）。

#### 10.2.5 动态代理源码：`Proxy.newProxyInstance`

```java
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h) {
    Objects.requireNonNull(h);

    final Class<?>[] intfs = interfaces.clone();
    // ... 权限检查 ...

    /*
     * 关键：查找或生成代理类。
     * 用 WeakCache 缓存 —— key 是 (loader, interfaces)，所以同一个接口只生成一次代理类
     */
    Class<?> cl = getProxyClass0(loader, intfs);

    try {
        // 拿到代理类的构造器（唯一的构造器，参数是 InvocationHandler）
        final Constructor<?> cons = cl.getConstructor(constructorParams);
        if (!Modifier.isPublic(cl.getModifiers())) {
            cons.setAccessible(true);
        }
        // 反射创建代理实例，把 handler 传进去
        return cons.newInstance(new Object[]{h});
    } catch (IllegalAccessException | InstantiationException e) {
        throw new InternalError(e.toString(), e);
    }
}
```

**`getProxyClass0` 内部**（简化）：

```java
private static Class<?> getProxyClass0(ClassLoader loader, Class<?>... interfaces) {
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }
    // 二级缓存：先查缓存，没有才用 ProxyClassFactory 生成
    return proxyClassCache.get(loader, interfaces);
}
```

**`ProxyClassFactory.apply()` 生成的类大致长这样**（用 `-Djdk.proxy.ProxyGenerator.saveGeneratedFiles=true` 可以 dump 出真实的 class 文件）：

```java
// 运行时生成的类，命名 $Proxy0、$Proxy1...
public final class $Proxy0 extends Proxy implements UserService {
    private static Method m3;   // save 方法

    public $Proxy0(InvocationHandler h) {
        super(h);               // 父类 Proxy 保存 handler
    }

    // 静态块里缓存所有 Method 对象（避免每次反射查找）
    static {
        try {
            m3 = Class.forName("UserService").getMethod("save", String.class);
        } catch (Exception e) { throw new NoSuchMethodError(e.getMessage()); }
    }

    // 你调的每个接口方法，都被转成对 handler.invoke 的调用
    @Override public final void save(String name) {
        try {
            super.h.invoke(this, m3, new Object[]{name});
            return;
        } catch (RuntimeException | Error e) {
            throw e;                                   // 运行时异常直接透传
        } catch (Throwable e) {
            throw new UndeclaredThrowableException(e); // 受检异常被包装！
        }
    }
}
```

**三个必须知道的细节**：

1. **代理类是 `final` 的**，且继承 `Proxy`——所以 JDK 动态代理**只能代理接口**（单继承已被 `Proxy` 占用）
2. **每个方法都转成 `h.invoke(this, method, args)`**——`this` 就是代理对象本身
3. **受检异常会被包成 `UndeclaredThrowableException`**——因为接口方法没声明该异常，代理类不能直接抛

#### 10.2.6 `InvocationHandler` 里 `proxy` 参数的作用（经典误区）

```java
public interface InvocationHandler {
    Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
}
```

**常见误解**：以为 `proxy` 是"被代理的目标对象"。**错！`proxy` 是代理对象自己**。

```java
new InvocationHandler() {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // proxy 是 $Proxy0 这个替身，不是目标对象
        // 如果你想调目标对象，得自己持有 target 引用
        return method.invoke(target, args);   // target 是构造 handler 时传进来的真实对象

        // 错误用法（会无限递归直到栈溢出）：
        // return method.invoke(proxy, args);   // 又调回代理自己
    }
}
```

**那 `proxy` 有什么用？** 用于：①判断是不是代理对象（`proxy instanceof xxx`）；②把代理对象传给其他方法（如 `method.invoke(proxy, ...)` 需要传递 `this` 语义时）；③在很多场景下**根本用不到**，所以常写成 `Object proxy` 直接忽略。

#### 10.2.7 JDK 动态代理 vs CGLIB

| 维度 | JDK 动态代理 | CGLIB |
| --- | --- | --- |
| 原理 | 运行时生成**实现接口**的类 | 运行时生成**继承目标类**的子类 |
| 前提 | 目标类**必须实现接口** | 目标类**不能被 final**，被代理方法不能 final/private |
| 生成方式 | `Proxy.newProxyInstance` | `Enhancer.create` + ASM 修改字节码 |
| 性能 | JDK 8+ 有优化，创建快、调用略慢 | 创建慢（ASM 生成）、调用快 |
| 依赖 | JDK 内置 | 需引入 `cglib`（Spring 已内置 repackage） |
| 典型场景 | Spring AOP（有接口时默认用） | Spring AOP（无接口时用）、Spring Boot 2.x 起**默认 `proxyTargetClass=true`，即默认 CGLIB** |

> **Spring Boot 2.x 起默认用 CGLIB**（`spring.aop.proxy-target-class=true` 是默认值），这个变化经常被问。原因：避免"同一个 Bean 有时用 JDK 代理、有时用 CGLIB"导致的不一致，且 CGLIB 能代理类更通用。

#### 10.2.8 注解：三个必须理解的元注解

```java
@Retention(RetentionPolicy.RUNTIME)     // 保留到运行时（反射可见）——自定义注解必须用这个
@Target({ElementType.METHOD, ElementType.FIELD})   // 能标在哪
@Inherited                               // 子类是否继承（只对类注解有效）
@Documented                              // 是否进 javadoc
public @interface MyAnno {
    String value() default "";           // 注解属性（本质是方法）
    int order() default 0;
}
```

| `@Retention` | 保留到哪 | 能否反射读到 | 典型用途 |
| --- | --- | --- | --- |
| `SOURCE` | 只在源码，编译后丢弃 | ❌ | `@Override`、`@SuppressWarnings`、Lombok 的注解 |
| `CLASS` | 保留到 class 文件，运行时丢弃 | ❌ | 默认级别，字节码工具可用 |
| `RUNTIME` | 保留到运行期 | ✅ | Spring 的 `@Autowired`、`@RequestMapping`、MyBatis 的 `@Select` |

**注解本身不做事**，它只是"贴在代码上的标签"，必须有**处理器**去读它——反射处理器（Spring 扫描 `@Autowired`）或编译期处理器（Lombok 的 APT）。

### 10.3 面试标准问答

**Q1：反射的性能问题在哪里？怎么优化？**
A：三个层面：①**调用开销**：`Method.invoke` 要装箱参数、做权限和类型检查，无法被 JIT 内联；②**查找开销**：`getDeclaredMethod` 每次都要遍历方法表，必须缓存；③**首次膨胀**：JDK 用 inflation 机制，前 15 次走 JNI，之后生成字节码。优化：缓存 `Method`/`Field`/`Constructor`；用 `setAccessible(true)` 跳过权限检查；用 `MethodHandle`；终极方案用 `LambdaMetafactory` 生成 `Function`/`BiFunction` 直接调用（MyBatis、Jackson 都这么做）。

**Q2：`Class.forName` 和 `ClassLoader.loadClass` 的区别？**
A：`forName` 默认 `initialize=true`，会触发**类初始化**（执行静态块）；`loadClass` 只加载类，不初始化。JDBC 用 `forName` 正是为了触发驱动的静态块完成注册。

**Q3：JDK 动态代理为什么必须基于接口？**
A：因为 `Proxy.newProxyInstance` 生成的代理类是 `public final class $Proxy0 extends Proxy implements 你的接口`——**它必须继承 `Proxy`**，而 Java 是单继承，所以没法再继承目标类，只能通过实现接口来"冒充"目标类型。要代理类就得用 CGLIB（生成子类）。

**Q4：`InvocationHandler.invoke` 的第一个参数是什么？**
A：是**代理对象本身**（`$Proxy0` 实例），不是被代理的目标对象。这是最常见的误解。要在 invoke 里调用真实对象，必须自己持有 target 引用（在构造 handler 时传入）。

**Q5：`@Retention` 三种级别怎么选？**
A：只在编译期做检查的用 `SOURCE`（如 `@Override`、Lombok）；需要字节码工具处理的用 `CLASS`；需要运行时反射读取的必须用 `RUNTIME`（Spring/MyBatis 的注解都是这个）。**框架注解 99% 是 RUNTIME。**

**Q6：`setAccessible(true)` 做了什么？有什么风险？**
A：关闭 Java 的访问权限检查（`private` 也能访问），并把方法标记为"可覆盖访问"。风险：①**破坏封装**，可能改到不该改的状态；②JDK 9+ 模块化后，对**其他模块未导出包**里的类调用会抛 `InaccessibleObjectException`，需要 `--add-opens` 打开；③在某些安全策略下会抛 `SecurityException`。

### 10.4 高频追问与陷阱

#### 追问 1：反射为什么慢？慢在哪一步？

**【追问场景】**
必问。答"因为要解析"就是外行话。面试官想听你区分"查找开销"和"调用开销"。

**【❌ 错误答法】**
> "反射就是比直接调用慢，因为它是动态的。"
> 错在哪：没有定位到具体环节，也说不清能不能优化、怎么优化。

**【✅ 正确答法】**
> 分两个环节：
> **一是查找**：`getDeclaredMethod`/`getDeclaredFields` 每次调用都要遍历类的方法表/字段表并构造对象，这是纯粹的重复劳动，**必须缓存**（Spring、MyBatis 都缓存 Method 对象）。
> **二是调用**：`Method.invoke` 要接收 `Object[]` 参数（基本类型要装箱）、做访问权限检查和类型检查，而且反射调用点是**多态的**（不知道具体目标），JIT 难以内联。JDK 内部还有 **inflation 机制**：前 15 次通过 `NativeMethodAccessorImpl`（JNI）调用，之后生成 `GeneratedMethodAccessor` 字节码走快速路径。
> 优化手段按投入排序：缓存 Method → `setAccessible(true)` 跳过权限检查 → 用 `MethodHandle`（可被 JIT 优化）→ 终极方案 `LambdaMetafactory` 生成函数式接口（MyBatis、Jackson 的做法，接近直接调用性能）。

#### 追问 2：动态代理为什么必须基于接口？那没有接口怎么办？

**【追问场景】**
Spring AOP 必问。会连着问"CGLIB 是怎么做到的""为什么 Spring Boot 2.x 默认改了"。

**【❌ 错误答法】**
> "因为 JDK 只支持接口代理，没接口就用 CGLIB。"
> 错在哪：只说结论不说原因。为什么"只支持接口"？因为生成的代理类要**继承 `Proxy`**，单继承已被占用。

**【✅ 正确答法】**
> 因为 JDK 生成的代理类形式固定为 `public final class $Proxy0 extends Proxy implements 接口1, 接口2`——**必须继承 `Proxy`**（`Proxy` 里存着 `InvocationHandler h`，且 `newProxyInstance` 要靠它构造）。Java 单继承，继承链被 `Proxy` 占了，就无法再继承目标类，所以只能通过"实现接口"来让调用方把它当作目标类型使用。
> 没有接口时只能用 **CGLIB**：它用 ASM 生成目标类的**子类**，覆写所有非 final 方法，在方法里回调 `MethodInterceptor.intercept()`。限制是目标类不能 `final`、被代理方法不能 `final`/`private`/`static`。
> 补充：Spring AOP 有接口时默认用 JDK 代理，Spring Boot 2.x 起默认 `proxyTargetClass=true` 全面改用 CGLIB——好处是行为统一（不会因为"有没有接口"导致代理类型不一致，进而引发注入类型问题），也能代理类。

#### 追问 3：`InvocationHandler.invoke` 的第一个参数 proxy 是什么？能直接用它调方法吗？

**【追问场景】**
写过动态代理的人也会答错的高频坑。面试官想看你有没有真正调试过无限递归。

**【❌ 错误答法】**
> "是被代理的目标对象，可以 `method.invoke(proxy, args)` 直接调用。"
> 错在哪：两个错误。`proxy` 是代理对象自己；`method.invoke(proxy, ...)` 会**又调回代理**，形成无限递归直到 `StackOverflowError`。

**【✅ 正确答法】**
> `proxy` 是**动态生成的代理对象本身**（`$Proxy0` 实例），不是目标对象。这是最容易搞混的一点。
> 所以在 `invoke` 里想调用真实业务逻辑，必须自己持有目标对象引用（通常在 handler 构造时传入 target），然后 `method.invoke(target, args)`。
> 如果写成 `method.invoke(proxy, args)`，因为 `proxy` 就是代理，方法调用又回到 `invoke`，会**无限递归 → StackOverflowError**。
> `proxy` 的真实用途：需要把"代理自身"传给别处时（比如某些框架用它做 `this` 语义、或判断 `proxy instanceof Xxx`），很多场景下直接忽略即可。

#### 追问 4：`getMethods()` 和 `getDeclaredMethods()` 有什么区别？踩过什么坑？

**【追问场景】**
框架/工具类开发常踩的坑，面试官想看你有没有写过反射相关代码。

**【❌ 错误答法】**
> "差不多，都能拿到方法。"
> 错在哪：漏掉了两个关键差异：**是否包含继承**、**是否包含非 public**。

**【✅ 正确答法】**
> 两点区别：
> ①**`getMethods()` 只返回 public，但包含从父类和接口继承来的方法；`getDeclaredMethods()` 返回本类声明的所有方法（含 private/protected），但不含继承的。**
> ②字段、构造器同理（`getFields` vs `getDeclaredFields`）。
> 实际踩过的坑：写"遍历所有字段做校验"的工具时用了 `getFields()`，结果 `private` 字段一个都拿不到，校验形同虚设；正确姿势是 `getDeclaredFields()` + `setAccessible(true)`，并且如果父类也有字段要处理，还得**沿着 `getSuperclass()` 往上递归**（`getDeclaredXxx` 不会自动包含父类）。
> 另一个坑：`getDeclaredMethods()` 会返回**桥接方法**，框架里要用 `method.isBridge()` / `isSynthetic()` 过滤掉。

#### 追问 5：`setAccessible(true)` 在 JDK 9+ 有什么变化？

**【追问场景】**
考察你是否了解模块化（JPMS）带来的破坏性变化。升级 JDK 17 踩过坑的人会有切肤之痛。

**【❌ 错误答法】**
> "没什么变化，一直都能用。"
> 错在哪：JDK 9 引入模块系统后，跨模块访问非导出包会失败。

**【✅ 正确答法】**
> JDK 9 引入模块系统后，`setAccessible(true)` 受到 **模块边界**的限制：如果目标类在另一个模块中，而那个模块没有通过 `exports ... to` 把该包导出给你的模块，`setAccessible` 会抛 **`InaccessibleObjectException`**。
> 这直接影响了大量依赖反射读写 JDK 内部字段的框架（早期版本的 Spring、Hibernate、各种序列化库在 JDK 17 上集体翻车）。
> 解决办法：①启动参数 `--add-opens java.base/java.lang=ALL-UNNAMED` 打开指定包（这也是很多项目迁移 JDK 17 时必须在启动脚本里加的配置）；②升级框架到适配版本；③自己写代码时避免访问未导出包。
> 另外 JDK 17 起**强封装**（`--illegal-access=deny` 成为默认），以前只是警告，现在是直接抛异常。

### 10.5 动手验证

```java
import java.lang.reflect.*;
import java.util.Map;

public class ReflectProxyDemo {
    interface UserService { String save(String name); }

    static class UserServiceImpl implements UserService {
        @Override public String save(String name) { return "saved:" + name; }
    }

    static class Target { private String secret = "私密"; }

    public static void main(String[] args) throws Exception {
        UserService target = new UserServiceImpl();

        // ① JDK 动态代理
        UserService proxy = (UserService) Proxy.newProxyInstance(
            UserService.class.getClassLoader(),
            new Class<?>[]{UserService.class},
            (p, method, a) -> {
                System.out.println("[前置] 调用 " + method.getName() + "，proxy 类型=" + p.getClass().getName());
                long start = System.nanoTime();
                Object result = method.invoke(target, a);     // 注意：传 target，不是 p
                System.out.println("[后置] 耗时 " + (System.nanoTime() - start) + "ns");
                return result;
            });
        System.out.println(proxy.save("Tom"));
        System.out.println("代理类名: " + proxy.getClass().getName());   // com.sun.proxy.$Proxy0

        // ② 反射访问 private 字段
        Target t = new Target();
        Field f = Target.class.getDeclaredField("secret");
        f.setAccessible(true);
        System.out.println("反射读到: " + f.get(t));

        // ③ getMethods vs getDeclaredMethods
        System.out.println("Target.getMethods().length = " + Target.class.getMethods().length);
        System.out.println("Target.getDeclaredMethods().length = " + Target.class.getDeclaredMethods().length);

        // ④ 类加载方式差异（看是否触发静态块）
        System.out.println("--- Class.forName ---");
        Class.forName("ReflectProxyDemo$WithStatic");
        System.out.println("--- ClassLoader.loadClass ---");
        ReflectProxyDemo.class.getClassLoader().loadClass("ReflectProxyDemo$WithStatic");
    }

    static class WithStatic {
        static { System.out.println("WithStatic 静态块执行了"); }
    }
}
```

想 dump 出动态代理生成的 class 文件：

```bash
java -Djdk.proxy.ProxyGenerator.saveGeneratedFiles=true ReflectProxyDemo
# 会在当前目录生成 com/sun/proxy/$Proxy0.class，用 javap -p -c 反编译就能看到 invoke 转调
```

### 10.6 一句话速记

- **反射** = 运行时看/改类结构；`Class.forName` **触发初始化**，`loadClass` 不触发（JDBC 靠前者注册驱动）。
- `getMethods` 拿 public + 继承，`getDeclaredMethods` 拿本类全部但无继承；框架里要过滤 `isBridge`。
- **JDK 动态代理必须基于接口**（生成的类要 `extends Proxy`）；`invoke` 的第一个参数是**代理对象自己**，不是 target。
- Spring Boot 2.x 起 **AOP 默认 CGLIB**；JDK 9+ `setAccessible` 受模块限制，需 `--add-opens`。

---

## 11. Lambda、函数式接口与序列化

### 11.1 先建立直觉

**Lambda 的本质是"把一段行为当成参数传出去"**。

在 JDK 8 之前，这件事只能通过匿名内部类做，代码又长又啰嗦：

```java
// JDK 8 之前
button.addActionListener(new ActionListener() {
    @Override public void actionPerformed(ActionEvent e) { System.out.println("点击了"); }
});

// JDK 8 之后
button.addActionListener(e -> System.out.println("点击了"));
```

**关键点**：Lambda **只能**用在"函数式接口"上——即**只有一个抽象方法的接口**（SAM，Single Abstract Method）。

```java
@FunctionalInterface           // 这个注解让编译器帮你检查"是不是只有一个抽象方法"
interface Calculator {
    int calc(int a, int b);    // 唯一抽象方法
    // 但可以有 default / static 方法（它们不算抽象方法）
    default int add(int a, int b) { return a + b; }
}
```

**那 Lambda 在字节码层面是怎么实现的？**

这一点非常值得讲清楚，因为它能解释"为什么 Lambda 不生成 class 文件"：

- **匿名内部类**：编译期就生成 `Outer$1.class`
- **Lambda**：编译期只生成一条 **`invokedynamic`** 指令，运行时由 `LambdaMetafactory` 动态生成实现类（首次调用时才"链接"）

```text
# 匿名内部类
new  → dup → invokespecial Outer$1.<init>  → 真的有类

# Lambda
invokedynamic #0:apply:()LCalculator;      ← 只有一条指令，运行时再"现做"
```

**所以**：Lambda 启动更快（少了类加载）、也更省内存；但**反射读不到 Lambda 的类型**（只有 `invokedynamic` 的动态调用点）。

**序列化是什么？** 把对象变成字节流（用于网络传输、写文件、缓存），再还原回来。

Java 原生序列化的核心问题是：**它是一个"隐式的、无契约的、危险的"机制**：

- **隐式**：不需要写任何代码，`implements Serializable` 就够了
- **无契约**：类结构一变就反序列化失败（靠 `serialVersionUID` 校验）
- **危险**：反序列化会**执行类的代码**（`readObject`），历史上出过大量 RCE 漏洞

> 现代工程更推荐 **JSON（Jackson）/ Protobuf** 这类显式序列化方案，Java 原生序列化主要用于框架内部（如 Dubbo 的某些协议、RMI、Session 持久化）。

### 11.2 原理与源码

#### 11.2.1 Lambda 与匿名内部类的关键差异

| 维度 | 匿名内部类 | Lambda |
| --- | --- | --- |
| 编译产物 | 独立 class 文件（`Outer$1.class`） | 无 class 文件，`invokedynamic` + 运行时生成 |
| `this` 指向 | **匿名类自己** | **外层类的实例** |
| 能否访问接口的 default 方法 | 能 | **不能**（Lambda 只能看到外层作用域） |
| 作用域 | 有自己的作用域（可以定义同名变量） | **和外部共享作用域**（不能定义同名局部变量） |
| 适用对象 | 任意接口/抽象类 | 仅函数式接口 |
| 捕获变量 | 必须是 effectively final | 同样必须 effectively final |

```java
public class LambdaVsAnon {
    int field = 1;
    void test() {
        Runnable anon = new Runnable() {
            int field = 2;                      // 可以遮蔽外部字段
            @Override public void run() {
                System.out.println(this.field); // 2 ← 匿名类自己的
            }
        };
        Runnable lambda = () -> {
            // int field = 3;                  // 编译错误！不能遮蔽外部局部变量
            System.out.println(this.field);     // 1 ← 外层的
        };
    }
}
```

#### 11.2.2 `LambdaMetafactory` 的三步流程

```java
// 1. 编译期：把 Lambda 编译成 invokedynamic 指令，指向 LambdaMetafactory.metafactory
//    字节码里带三个参数：函数式接口的方法签名、实现方法签名、捕获的变量
// 2. 首次执行：JVM 调用 LambdaMetafactory.metafactory，动态生成实现类
// 3. 后续执行：直接调用已生成的实现（走 CallSite）
```

简化后的核心签名：

```java
public static CallSite metafactory(MethodHandles.Lookup caller,
                                   String interfaceMethodName,
                                   MethodType factoryType,
                                   MethodType interfaceMethodType,
                                   MethodHandle implementation,
                                   MethodType dynamicMethodType)
```

**生成哪种类取决于是否捕获变量**：

| 情况 | 生成的类 | 说明 |
| --- | --- | --- |
| **无捕获**（`() -> System.out.println(...)`，不引用外部变量） | 单例，静态字段持有 | 全局只有一个实例，最省内存 |
| **有捕获**（`x -> x + base`，引用了 local `base`） | 每次创建新实例，捕获值存为字段 | 类似"闭包对象" |

> 这是一个很好的加分点：**"不捕获外部变量的 Lambda 是单例"**。所以 `Runnable r1 = () -> {}; Runnable r2 = () -> {};` 可能 `r1 == r2`（同一 lambda body 时）。

#### 11.2.3 序列化的核心：`Serializable` 是个空接口

```java
public interface Serializable {
}
```

**它什么都不做！** 它只是一个**标记接口（marker interface）**，告诉 JVM"这个类允许被序列化"。真正的序列化逻辑由 `ObjectOutputStream` 负责：

```java
// ObjectOutputStream.writeObject0 的核心判断
private void writeObject0(Object obj, boolean unshared) throws IOException {
    // ...
    if (obj instanceof String) { writeString((String) obj, unshared); }
    else if (cl.isArray()) { writeArray(obj, desc, unshared); }
    else if (obj instanceof Enum) { writeEnum((Enum<?>) obj, desc, unshared); }
    else if (obj instanceof Serializable) {          // ← 关键判断
        writeOrdinaryObject(obj, desc, unshared);
    } else {
        throw new NotSerializableException(cl.getName());   // ← 没实现就抛这个
    }
}
```

#### 11.2.4 `serialVersionUID` 到底校验什么

反序列化时，`ObjectInputStream` 会做一次"版本比对"：

```java
// ObjectStreamClass 的简化逻辑
if (localDesc.getSerialVersionUID() != streamDesc.getSerialVersionUID()) {
    throw new InvalidClassException(localDesc.getName(),
        "local class incompatible: stream classdesc serialVersionUID = " + streamDesc.getSerialVersionUID()
        + ", local class serialVersionUID = " + localDesc.getSerialVersionUID());
}
```

**如果不显式声明，`serialVersionUID` 由编译器根据类的结构自动计算**（类名、修饰符、接口、字段、方法签名……）。所以：

> **只要你改了类（加个字段、改个方法、换个修饰符），自动算出的 UID 就变了，老数据全部反序列化失败。**

这就是"必须显式声明 `serialVersionUID`"的原因：

```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L;   // 建议显式声明，加字段也不会破坏兼容
    private String name;
    private int age;
}
```

显式声明后，加字段不会失败——新字段会被赋成默认值（`null`/0）。**但删除/改类型字段仍可能导致问题**。

> **IDEA 提示**：可以开启 "Serializable class without 'serialVersionUID'" 的检查，自动生成。

#### 11.2.5 反序列化**不会**调用构造器（重要且反直觉）

```java
public class NoCtor implements Serializable {
    private static final long serialVersionUID = 1L;
    private int value;

    public NoCtor() {
        System.out.println("构造器被调用了");
        value = 100;
    }

    public static void main(String[] args) throws Exception {
        // 序列化
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        try (ObjectOutputStream oos = new ObjectOutputStream(bos)) {
            oos.writeObject(new NoCtor());          // 打印"构造器被调用了"
        }
        System.out.println("--- 反序列化 ---");
        try (ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bos.toByteArray()))) {
            NoCtor obj = (NoCtor) ois.readObject(); // 不打印！构造器没被调用
            System.out.println("value = " + obj.value);   // 100（来自流里的数据）
        }
    }
}
```

**原理**：反序列化通过 `ObjectStreamClass` + `Unsafe.allocateInstance`（或反射的 `newInstance` 特殊路径）直接**分配对象、绕过构造器**，然后把流里的字段值直接填进去。这也是**反序列化漏洞的根源**——攻击者可以绕过你的构造器校验，直接构造出非法状态的对象。

#### 11.2.6 `transient` 与 `readObject`/`writeObject`

```java
public class Account implements Serializable {
    private static final long serialVersionUID = 1L;

    private String name;
    private transient String password;      // 不参与序列化，反序列化后为 null

    // 自定义序列化逻辑（必须严格是这个签名）
    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();           // 先写非 transient 字段
        out.writeObject(encrypt(password)); // 再额外写加密后的密码
    }

    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject();
        this.password = decrypt((String) in.readObject());
    }
}
```

> **注意**：`readObject`/`writeObject` 是**私有方法**，由序列化机制通过反射调用（这是"约定优于配置"的典型）。这也意味着**如果你在 `readObject` 里做了安全校验，就一定要校验所有字段**——攻击者可以构造恶意流。

#### 11.2.7 单例被序列化破坏，以及 `readResolve`

```java
public class Singleton implements Serializable {
    private static final long serialVersionUID = 1L;
    public static final Singleton INSTANCE = new Singleton();

    private Singleton() {}

    // 关键：反序列化时用这个方法替换掉新创建的对象
    private Object readResolve() {
        return INSTANCE;
    }
}
```

**为什么需要 `readResolve`？** 因为反序列化**不走构造器**（见 11.2.5），它会直接造出一个新对象，于是单例被破坏（`obj != Singleton.INSTANCE`）。`readResolve` 让 `ObjectInputStream` 在返回前把结果替换成你指定的对象。

**更彻底的方案：枚举单例**

```java
public enum EnumSingleton {
    INSTANCE;
    public void doSomething() {}
}
```

**为什么枚举最安全？** 因为 `ObjectInputStream` 对枚举有特殊处理：反序列化时调用 `Enum.valueOf`，**返回的必然是同一个枚举常量**，且**枚举不能用反射创建实例**（`Constructor.newInstance` 会抛 `IllegalArgumentException: Cannot reflectively create enum objects`）。所以 Effective Java 推荐"单元素的枚举是实现单例的最佳方法"。

### 11.3 面试标准问答

**Q1：Lambda 和匿名内部类的区别？**
A：①**编译产物不同**：匿名类生成独立 class 文件，Lambda 编译成 `invokedynamic` + 运行时 `LambdaMetafactory` 生成实现，不产生 class 文件；②**`this` 语义不同**：匿名类的 `this` 指向匿名类自己，Lambda 的 `this` 指向外层类实例；③**作用域不同**：Lambda 与外层共享作用域（不能定义同名变量），匿名类有自己的作用域；④**适用性**：Lambda 只能用于函数式接口；⑤**性能**：Lambda 首次调用有链接开销，但后续调用更快、内存更省，无捕获时还是单例。

**Q2：为什么 `Serializable` 是空接口？**
A：它是**标记接口**，只用于告诉 `ObjectOutputStream` "这个类允许序列化"。真正的序列化逻辑在 `ObjectOutputStream` 里，它通过 `obj instanceof Serializable` 判断；不实现就会抛 `NotSerializableException`。这种设计的好处是不侵入类型系统，缺点是"隐式"，无法表达"哪些字段该序列化"（所以才有了 `transient` 和 `writeObject`）。

**Q3：`serialVersionUID` 有什么用？不写会怎样？**
A：反序列化时用它做**版本一致性校验**：流里的 UID 和本地类的 UID 不一致就抛 `InvalidClassException`。不显式声明时，编译器会根据类结构自动计算一个值——**类结构一改（加字段、改签名）UID 就变，历史数据全部无法反序列化**。所以凡是可能跨版本反序列化的类，都要显式声明。

**Q4：反序列化会调用构造器吗？**
A：不会（除非实现 `Externalizable` 且使用无参构造）。`ObjectInputStream` 通过 `allocateInstance` 直接分配对象、绕过构造器，然后把字段值填进去。这带来两个后果：①你在构造器里的校验逻辑对反序列化无效（**这是安全漏洞的根源**）；②单例会被破坏，需要 `readResolve` 或枚举单例。

**Q5：`transient` 的作用？**
A：标记字段不参与序列化。反序列化后该字段为默认值（对象 `null`、数字 0、boolean `false`）。常用于：密码等敏感字段、不可序列化的字段（如 `InputStream`）、可以从其他字段推导出来的冗余字段。需要"部分保密"时要配合自定义 `writeObject`/`readObject`。

**Q6：为什么说 Java 原生序列化不安全？**
A：因为反序列化会**根据流里的数据还原任意对象，并调用其 `readObject` 等钩子方法**——攻击者可以构造恶意的类链（gadget chain），在反序列化时触发 `Runtime.exec` 等危险操作，导致 RCE。防护：①**不要反序列化不受信任的数据**（最重要）；②用 `ObjectInputFilter`（JDK 9+）设置白名单；③用 JSON/Protobuf 等不支持任意类型还原的格式。

### 11.4 高频追问与陷阱

#### 追问 1：Lambda 会生成 class 文件吗？它和匿名内部类到底差在哪？

**【追问场景】**
JDK 8 之后的必问。答"都是语法糖、一样的"就错了——它们在字节码层面完全不同。

**【❌ 错误答法】**
> "Lambda 就是匿名内部类的语法糖，编译后一样。"
> 错在哪：编译产物根本不同。匿名类生成 `.class` 文件，Lambda 生成 `invokedynamic` 指令，运行时才生成实现。

**【✅ 正确答法】**
> 不一样，编译产物完全不同：
> 匿名内部类在**编译期**就生成独立的 class 文件（`Outer$1.class`），走 `new` + `invokespecial`，是实实在在的类。
> Lambda 编译后只有一条 **`invokedynamic`** 指令，指向 `LambdaMetafactory`；**首次执行**时由 JVM 动态生成实现类并链接成 `CallSite`，之后直接复用。所以 Lambda **不产生 class 文件**，启动更快、内存更省。
> 另一个关键差异是 `this` 语义：匿名类的 `this` 是匿名类自己，Lambda 的 `this` 是外层类实例（Lambda 没有自己的作用域）。
> 还有一个性能细节：**不捕获外部变量的 Lambda 是单例**（同一个 lambda body 只有一个实例），而匿名内部类每次都 `new`。

#### 追问 2：不声明 `serialVersionUID` 会怎样？

**【追问场景】**
工程踩坑题。面试官想知道你有没有处理过"发版后缓存里的老对象读不出来"。

**【❌ 错误答法】**
> "不会怎样，Java 会自动生成一个。"
> 错在哪：自动生成的值**会随类结构变化而变**，这正是灾难来源。

**【✅ 正确答法】**
> JDK 会基于类的结构（类名、修饰符、接口、字段、方法签名等）**自动计算**一个 UID。所以只要改动了类（哪怕只是加一个字段、把方法 `public` 改 `protected`），UID 就会变。
> 结果是**跨版本反序列化直接失败**，抛 `InvalidClassException: local class incompatible`。如果这个类被用作缓存（Redis 里存的 Java 序列化对象）或 RPC 传输，那就是发版即故障。
> 所以规则是：**凡是实现 Serializable 的类都显式声明 `serialVersionUID`**（IDE 可以配置生成）。显式声明后，加字段不会失败（新字段取默认值），但**删除字段、改变字段类型、改变继承关系**仍然可能出问题，需要谨慎。

#### 追问 3：单例模式怎么防止被反射和序列化破坏？

**【追问场景】**
Singleton 的进阶问法。能说出"枚举最安全"并解释原因的就是高分。

**【❌ 错误答法】**
> "加个 `readResolve` 就行了，反射没办法。"
> 错在哪：`readResolve` 只防序列化，**防不住反射**——`setAccessible(true)` 后照样 `newInstance` 出新对象。

**【✅ 正确答法】**
> 分两种攻击分别防御：
> **反射攻击**：私有构造器里加判断，若实例已存在就抛异常（`throw new RuntimeException("请用 getInstance")`）；但枚举**天然免疫**，因为 `Constructor.newInstance` 对枚举会直接抛 `IllegalArgumentException`。
> **序列化攻击**：反序列化**不走构造器**，会造出新对象，所以构造器校验无效。需要 `private Object readResolve() { return INSTANCE; }` 把结果替换回单例。
> **终极方案是枚举单例**（`enum Singleton { INSTANCE }`）：`ObjectInputStream` 对枚举有特殊处理，反序列化通过 `Enum.valueOf` 返回同一常量；反射也无法创建枚举实例。Effective Java 明确推荐这个写法。
> 补充：`readResolve` 要生效，类必须实现 `Serializable`，且方法是 `private Object readResolve()`（返回值可以更具体）。

#### 追问 4：反序列化不调用构造器，这有什么风险？

**【追问场景】**
高级题，直击 Java 反序列化漏洞本质。答得出来是明显加分项。

**【❌ 错误答法】**
> "没什么风险，就是字段直接赋值，挺方便的。"
> 错在哪：恰恰相反——**绕过构造器校验 = 绕过所有不变量检查**，这是安全漏洞的温床。

**【✅ 正确答法】**
> 风险有三层：
> ①**不变量被破坏**：你在构造器里做的参数校验、状态初始化全部失效，攻击者可以构造出"非法状态"的对象（比如 `balance` 为负数、`role` 为 admin）。
> ②**触发危险的钩子方法**：反序列化会调用 `readObject`、`readResolve`、以及字段类型的 `readObject`，攻击者可以构造 gadget chain（利用已有库里的类链）在反序列化时执行任意代码，这就是历史上大量 Java RCE 漏洞（如 Commons-Collections、Fastjson 系列）的原理。
> ③**单例/缓存被破坏**。
> 防护措施：**绝不反序列化不可信数据**（最重要）；用 `ObjectInputFilter` 做类白名单（JDK 9+）；`serialVersionUID` 只能防版本错配、防不了攻击；生产上更推荐用 JSON（Jackson 的 `FAIL_ON_UNKNOWN_PROPERTIES` + 类型白名单）或 Protobuf。

#### 追问 5：`@FunctionalInterface` 注解是必须的吗？

**【追问场景】**
细节题，考"注解的实际作用"。

**【❌ 错误答法】**
> "必须加，不加就不能用 Lambda。"
> 错在哪：它是**可选的编译期检查**，不影响 Lambda 能否使用。

**【✅ 正确答法】**
> 不是必须的。它的唯一作用是**让编译器检查**这个接口是否"恰好只有一个抽象方法"——如果写了两个，编译就报错。不加注解也能用 Lambda（编译器依然按 SAM 规则判断）。
> 但强烈建议加：①意图明确（这是个函数式接口，不要随手加抽象方法）；②**防止别人后续加抽象方法破坏兼容**（这是它在 JDK 源码里的核心价值，`Runnable`、`Function` 等都标了）。
> 注意：`default` 方法、`static` 方法、以及 `Object` 的 public 方法（`equals`/`hashCode`/`toString` 的抽象声明）**都不算抽象方法**，不会让接口失去函数式接口资格。

### 11.5 动手验证

```java
import java.io.*;
import java.util.function.Function;

public class LambdaSerializeDemo {
    // 单例 + 序列化破坏演示
    static class Singleton implements Serializable {
        private static final long serialVersionUID = 1L;
        public static final Singleton INSTANCE = new Singleton();
        private Singleton() { System.out.println("Singleton 构造器执行"); }

        // 注释掉下面这个方法，反序列化就会得到新对象
        private Object readResolve() { return INSTANCE; }
    }

    // 反序列化不走构造器
    static class NoCtor implements Serializable {
        private static final long serialVersionUID = 1L;
        int value;
        NoCtor() { System.out.println("NoCtor 构造器执行"); value = 100; }
    }

    static <T> T roundTrip(T obj) throws Exception {
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        try (ObjectOutputStream oos = new ObjectOutputStream(bos)) { oos.writeObject(obj); }
        try (ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bos.toByteArray()))) {
            return (T) ois.readObject();
        }
    }

    public static void main(String[] args) throws Exception {
        // ① 单例是否被破坏
        System.out.println("--- 单例序列化 ---");
        Singleton s = roundTrip(Singleton.INSTANCE);
        System.out.println("是同一个单例? " + (s == Singleton.INSTANCE));   // true（有 readResolve）

        // ② 构造器是否被调用
        System.out.println("--- 反序列化构造器 ---");
        NoCtor n = roundTrip(new NoCtor());     // 只打印一次构造器（new 的时候）
        System.out.println("value = " + n.value);   // 100

        // ③ 不捕获变量的 Lambda 是单例
        System.out.println("--- Lambda 单例 ---");
        Runnable r1 = () -> System.out.println("hi");
        Runnable r2 = () -> System.out.println("hi");
        // 注意：取决于是否同一个调用点，同一行/同一方法内通常 r1 == r2
        final int base = 10;
        Function<Integer, Integer> f1 = x -> x + base;   // 捕获了 base → 每次新实例
        Function<Integer, Integer> f2 = x -> x + base;
        System.out.println("有捕获的 lambda 相等? " + (f1 == f2));   // 通常 false
    }
}
```

### 11.6 一句话速记

- Lambda **只用于函数式接口**，编译成 `invokedynamic`，**不产生 class 文件**；`this` 指外层、作用域共享、**无捕获时是单例**。
- `Serializable` 是**空标记接口**；`serialVersionUID` 做版本校验，**不写就会随类结构变化而失败**。
- 反序列化**不走构造器**（绕过校验 + 是漏洞根源）；`transient` 字段反序列化后为默认值。
- 单例防破坏：`readResolve` 防序列化，**枚举单例两者通吃**（最推荐）。

---

## 12. 值传递、深浅拷贝、BigDecimal 与位运算

### 12.1 先建立直觉

#### 12.1.1 "Java 只有值传递"到底是什么意思

**核心结论：Java 永远是值传递（pass by value），不存在引用传递（pass by reference）。**

难点只在于：**当传递的是对象时，"值"是引用的副本**。

**类比**：你有一把**钥匙**（引用），钥匙对应一间房（对象）。

- 你把**钥匙复制一把**给同事（传参 = 复制引用值）
- 同事拿复制品去开同一间房，可以**改房里的东西**（`obj.setXxx()` 生效）
- 但同事把他那把钥匙**换成另一间房的钥匙**（`obj = new Xxx()`），**你手里的钥匙不受影响**

```java
void reassign(StringBuilder sb) {
    sb = new StringBuilder("新的");    // 只改了"副本钥匙"，外面不受影响
}
void modify(StringBuilder sb) {
    sb.append("追加");                 // 通过副本钥匙改了同一间房，外面看得到
}

StringBuilder s = new StringBuilder("原");
reassign(s);
System.out.println(s);      // 原   ← 换钥匙无效
modify(s);
System.out.println(s);      // 原追加 ← 改内容有效
```

#### 12.1.2 深浅拷贝：拷到第几层

**类比**：你有一份"文件夹"（对象），里面装着几个"子文件夹"（引用字段）。

- **浅拷贝**：新建一个文件夹壳，把里面子文件夹的**快捷方式**复制过去。**壳是新的，里面还是同一批子文件夹**。
- **深拷贝**：把子文件夹里的东西**也全部复制一份**。完全独立，改任何一个都不影响另一个。

```text
原对象 A:               浅拷贝 B:              深拷贝 C:
┌─────────┐            ┌─────────┐            ┌─────────┐
│ name ──→ "Tom"│      │ name ──→ "Tom"│      │ name ──→ "Tom"(新)│
│ list ──┐│            │ list ──┐│            │ list ──→ [新List] │
└────────│┘            └────────│┘            └─────────────────┘
         ▼                      ▼
      [共享同一个 List]  ← 浅拷贝的 B.list == A.list
```

#### 12.1.3 为什么 `0.1 + 0.2 != 0.3`

因为**二进制无法精确表示 0.1**（就像十进制无法精确表示 1/3）。`0.1` 在二进制里是无限循环小数，`double` 只能存 52 位有效数字的近似值。

```java
System.out.println(0.1 + 0.2);          // 0.30000000000000004
System.out.println(0.1f + 0.2f);        // 0.3 （float 精度不同，恰好显示对）
```

**涉及的场景必须用 `BigDecimal`**：金额、利率、税费、任何"分/厘"级别的计算。

#### 12.1.4 为什么需要位运算

三个典型用途：

1. **权限位**：一个 `int` 存 32 种权限（`READ = 1 << 0`、`WRITE = 1 << 1`...），判断用 `&`、授权用 `|`。JDK 的 `Modifier`、Netty 的 `ChannelOption` 都是这个套路。
2. **高性能数学**：`(n - 1) & hash` 代替取模（前提 n 是 2 的幂），HashMap 就靠这个。
3. **位标志位与哈希扰动**：`h ^ (h >>> 16)` 让高位参与运算。

### 12.2 原理与源码

#### 12.2.1 传参的内存模型

```java
public class PassDemo {
    static void changeInt(int x) { x = 100; }                   // 改的是副本
    static void changeArray(int[] arr) { arr[0] = 100; }        // 通过副本引用改内容
    static void reassignArray(int[] arr) { arr = new int[]{999}; }  // 换指向，无效

    public static void main(String[] args) {
        int a = 1;
        changeInt(a);
        System.out.println(a);          // 1

        int[] arr = {1, 2, 3};
        changeArray(arr);
        System.out.println(arr[0]);     // 100

        reassignArray(arr);
        System.out.println(arr[0]);     // 100（不是 999）
    }
}
```

**经典 swap 题，为什么不能用方法交换两个 int：**

```java
static void swap(int a, int b) {
    int tmp = a; a = b; b = tmp;    // 只交换了栈上的两个副本
}
int x = 1, y = 2;
swap(x, y);
System.out.println(x + "," + y);    // 1,2 —— 没变

// 想真正交换，只能：
// ① 返回值：int[] swap(int a, int b) { return new int[]{b, a}; }
// ② 用可变容器（int[] 或 AtomicInteger）
// ③ 直接在调用处写
```

#### 12.2.2 `Object.clone()` 源码与"浅"的根源

```java
public class Object {
    // protected + native：子类不能直接调外部对象的 clone
    protected native Object clone() throws CloneNotSupportedException;
}
```

**关键点**：

1. **`protected`**：只有本类及子类能调 `super.clone()`，外部不能 `obj.clone()`——这保证了"只有对象自己能决定怎么复制自己"
2. **`native`**：由 JVM 实现，逻辑是**分配一块同样大小的内存，然后逐字节拷贝**（memcpy 语义）→ 所以**天然是浅拷贝**
3. **必须实现 `Cloneable`**：否则 `super.clone()` 抛 `CloneNotSupportedException`（`Cloneable` 是空标记接口，JVM 检查 `instanceof`）

```java
class Person implements Cloneable {
    String name;
    List<String> hobbies;

    @Override
    public Person clone() {
        try {
            return (Person) super.clone();   // 浅拷贝：name/hobbies 都是同一个引用
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(e);      // 实现了 Cloneable 就不会走到这
        }
    }
}

Person p1 = new Person();
p1.hobbies = new ArrayList<>(List.of("读书"));
Person p2 = p1.clone();
p2.hobbies.add("游泳");
System.out.println(p1.hobbies);   // [读书, 游泳] ← 被影响了，因为同一份 List
```

**数组的 `clone()` 也是浅拷贝**：

```java
int[][] arr = {{1, 2}, {3, 4}};
int[][] copy = arr.clone();
copy[0][0] = 99;
System.out.println(arr[0][0]);    // 99 ← 内层数组还是同一个
```

#### 12.2.3 深拷贝的四种实现

| 方案 | 做法 | 优点 | 缺点 |
| --- | --- | --- | --- |
| ① 手动递归 clone | 每个引用字段再 `clone()` | 可控、快 | 代码多、易漏 |
| ② 序列化反序列化 | 对象写进字节流再读出来 | 自动处理深层嵌套 | 要求全部实现 `Serializable`、慢、不能有不可序列化字段 |
| ③ JSON 序列化 | `JSON.parseObject(JSON.toJSONString(o), Cls.class)` | 简单、跨语言 | 依赖第三方库、丢失类型信息（如 `Object` 字段）、性能一般 |
| ④ 拷贝构造器 | `new Person(other)` | 明确、无反射、推荐 | 需要手写 |

**推荐 ① / ④**，序列化方案适合"临时用、对象图复杂"的场景。

```java
// ① 手动深拷贝
@Override
public Person clone() {
    try {
        Person copy = (Person) super.clone();
        copy.hobbies = new ArrayList<>(this.hobbies);   // 引用字段重新创建
        return copy;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError(e);
    }
}

// ② 序列化深拷贝（工具方法）
@SuppressWarnings("unchecked")
static <T extends Serializable> T deepCopy(T obj) {
    try (ByteArrayOutputStream bos = new ByteArrayOutputStream();
         ObjectOutputStream oos = new ObjectOutputStream(bos)) {
        oos.writeObject(obj);
        try (ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bos.toByteArray()))) {
            return (T) ois.readObject();
        }
    } catch (Exception e) {
        throw new RuntimeException("深拷贝失败", e);
    }
}
```

> **注意**：`clone` 机制本身被广泛批评（`Cloneable` 空接口、绕过构造器、`final` 字段无法正确复制、继承链上必须层层实现）。Effective Java 的建议是：**优先用拷贝构造器或静态工厂，避免实现 `Cloneable`**。

#### 12.2.4 `BigDecimal`：三个最容易踩的坑

**坑 1：用 `double` 构造**

```java
System.out.println(new BigDecimal(0.1));       // 0.1000000000000000055511151231257827021181583404541015625
System.out.println(new BigDecimal("0.1"));     // 0.1  ← 正确
System.out.println(BigDecimal.valueOf(0.1));   // 0.1  ← 正确
```

**源码解释**：

```java
// 用 double 构造：原样接收二进制的精确值 → 出现一长串尾巴
public BigDecimal(double val) {
    this(val, 0, 0);   // 内部用 doubleToLongBits 精确转换，不做舍入
}

// valueOf：先转成字符串再解析 → 得到"看起来的十进制值"
public static BigDecimal valueOf(double val) {
    return new BigDecimal(Double.toString(val));   // Double.toString(0.1) = "0.1"
}
```

> **结论**：**永远不要用 `new BigDecimal(double)`**。用 `new BigDecimal("0.1")`（字符串）或 `BigDecimal.valueOf(0.1)`。

**坑 2：`equals` 与 `compareTo` 不一致**

```java
BigDecimal a = new BigDecimal("1.0");
BigDecimal b = new BigDecimal("1.00");
System.out.println(a.equals(b));       // false ← 因为 scale 不同（1 vs 2）
System.out.println(a.compareTo(b));    // 0   ← 数值相等
```

**源码解释**：

```java
// equals：同时比 scale 和 unscaled value
public boolean equals(Object x) {
    if (!(x instanceof BigDecimal)) return false;
    BigDecimal xDec = (BigDecimal) x;
    if (x == this) return true;
    if (scale != xDec.scale) return false;           // ← scale 参与比较！
    return this.intCompact == xDec.intCompact && ...
}
```

> **结论**：**金额比较一律用 `compareTo(...) == 0`**，绝对不要用 `equals`。这也是 `BigDecimal` 作为 `HashMap` key 时的陷阱。

**坑 3：`divide` 除不尽直接抛异常**

```java
BigDecimal a = new BigDecimal("1");
BigDecimal b = new BigDecimal("3");
a.divide(b);    // ArithmeticException: Non-terminating decimal expansion; no exact representable decimal result
```

**正确写法（必须指定精度和舍入模式）**：

```java
a.divide(b, 2, RoundingMode.HALF_UP);      // 0.33
a.divide(b, 4, RoundingMode.HALF_UP);      // 0.3333
```

**常用舍入模式**：

| 模式 | 含义 | 1.5 的结果 | 2.5 的结果 |
| --- | --- | --- | --- |
| `HALF_UP` | 四舍五入（最常用） | 2 | 3 |
| `HALF_EVEN` | 银行家舍入（向偶数靠） | 2 | 2 |
| `HALF_DOWN` | 五舍六入 | 1 | 2 |
| `DOWN` | 向零截断 | 1 | 2 |
| `UP` | 远离零 | 2 | 3 |
| `CEILING` | 向正无穷 | 2 | 3 |
| `FLOOR` | 向负无穷 | 1 | 2 |

> **金额场景标准配置**：`setScale(2, RoundingMode.HALF_UP)`。注意 `HALF_UP` 是业务习惯，`HALF_EVEN` 是金融/统计上更公平的做法（避免系统性偏高），看公司财务规范。
>
> **另一个铁律**：**`BigDecimal` 是不可变的，所有运算都返回新对象**，所以 `a.add(b)` 必须接收返回值，`a.add(b);` 这种写法等于什么都没做。

#### 12.2.5 位运算速查

| 运算符 | 名称 | 例子 | 结果 | 常见用途 |
| --- | --- | --- | --- | --- |
| `&` | 按位与 | `6 & 3` | `2`（110 & 011 = 010） | 取低位、判权限、掩码 |
| `\|` | 按位或 | `6 \| 3` | `7`（110 \| 011 = 111） | 设置权限位、合并标志 |
| `^` | 异或 | `6 ^ 3` | `5`（110 ^ 011 = 101） | 哈希扰动、交换两数、去重 |
| `~` | 按位取反 | `~6` | `-7`（`~x == -x-1`） | 少见 |
| `<<` | 左移 | `1 << 4` | `16` | 快速乘 2^n |
| `>>` | **算术**右移 | `-8 >> 2` | `-2` | 快速除以 2^n，**符号位补符号** |
| `>>>` | **逻辑**右移 | `-8 >>> 2` | `1073741822` | **符号位补 0**，哈希扰动常用 |

**三个高频用法**：

```java
// ① 判断奇偶（比 % 快，且负数也正确）
if ((n & 1) == 0) { /* 偶数 */ }

// ② 用位运算代替 2 的幂取模（HashMap 的做法）
int index = (capacity - 1) & hash;    // capacity 必须是 2 的幂

// ③ 权限位（JDK Modifier 就是这么做的）
int READ = 1 << 0;     // 0001
int WRITE = 1 << 1;    // 0010
int EXEC = 1 << 2;     // 0100
int perms = READ | WRITE;              // 赋予读写 → 0011
boolean canRead = (perms & READ) != 0; // 判断是否有读权限 → true
perms &= ~WRITE;                       // 收回写权限 → 0001
```

**为什么 `>>>` 在哈希里更常用？**

```java
// HashMap 的扰动函数
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
//                                              ↑ 逻辑右移：把高 16 位移到低位，且用 0 填充。
//                                                如果用 >> ，负数会补 1，扰动效果变差
```

### 12.3 面试标准问答

**Q1：Java 是值传递还是引用传递？**
A：**只有值传递**。基本类型传的是值的副本；对象传的是"引用值"的副本（不是对象本身，也不是"引用传递"）。所以：①在方法内重新赋值参数（`obj = new Xxx()`）不影响调用方；②通过参数修改对象内部状态会影响调用方（因为两个引用指向同一个对象）。"Java 是引用传递"是错误说法——引用传递意味着方法能改变调用方的引用变量本身，Java 做不到。

**Q2：为什么 `swap(int a, int b)` 无法交换？**
A：因为形参是实参的**副本**，方法内交换的是栈上的两个局部变量，方法返回后栈帧销毁，调用方的变量毫无变化。要交换只能：返回新值/新数组、或用可变容器（`int[]`、`AtomicInteger`）。

**Q3：浅拷贝和深拷贝的区别？`clone()` 是哪种？**
A：浅拷贝只复制对象本身（基本类型字段值复制、引用字段只复制引用），新老对象**共享**引用指向的同一批子对象；深拷贝会递归复制所有层级的对象，完全独立。`Object.clone()` 是**浅拷贝**（native 实现是逐字节拷贝内存）。

**Q4：`clone()` 为什么要实现 `Cloneable`？为什么 `Object.clone` 是 `protected`？**
A：`Cloneable` 是空标记接口，JVM 在 `clone()` 里检查 `instanceof Cloneable`，没实现就抛 `CloneNotSupportedException`——这是给类作者的一次"是否允许被复制"的选择权。`protected` 是为了限制调用范围：只有类自己（或子类通过 `super.clone()`）能复制自己，外部不能随意 `obj.clone()`，保证"复制自己"是类自己的决定。

**Q5：`0.1 + 0.2 == 0.3` 为什么是 false？**
A：因为 `0.1`、`0.2`、`0.3` 在二进制下都是无限循环小数，`double` 只能存近似值，而三者的近似误差不同，相加后误差累积，得到 `0.30000000000000004`。**浮点数永远不能直接用 `==` 比较**，要比较差值：`Math.abs(a - b) < 1e-9`，或者用 `BigDecimal`。

**Q6：`new BigDecimal(0.1)` 和 `BigDecimal.valueOf(0.1)` 的区别？**
A：`new BigDecimal(0.1)` 用 `double` 的**精确二进制值**构造，得到 `0.1000000000000000055511151231257827021181583404541015625`；`valueOf` 内部是 `new BigDecimal(Double.toString(0.1))`，`Double.toString` 会给出最短的十进制表示 `"0.1"`，所以结果是精确的 `0.1`。**金额场景必须用 `valueOf` 或字符串构造。**

**Q7：`BigDecimal` 的 `equals` 和 `compareTo` 有什么区别？**
A：`equals` 同时比较 **scale（小数位数）** 和数值，所以 `new BigDecimal("1.0").equals(new BigDecimal("1.00"))` 是 **false**；`compareTo` 只比较数值（忽略 scale），返回 0。**金额比较必须用 `compareTo`**，否则会因为 scale 不同误判为不相等。

### 12.4 高频追问与陷阱

#### 追问 1：Java 是值传递还是引用传递？

**【追问场景】**
史上最常见的面试题之一。面试官通常接着给一段 `swap` 或"修改对象"的代码让你判断结果。

**【❌ 错误答法】**
> "基本类型是值传递，对象是引用传递。"
> 错在哪：这个说法**本身就是错的**，但流传极广。Java 规范里没有"引用传递"这个概念——传对象时传的依然是"值"，只不过这个值是引用的拷贝。

**【✅ 正确答法】**
> Java **只有值传递**。
> 传基本类型时，传的是数值的副本；传对象时，传的是"引用值"的副本（可以理解成复制了一把钥匙）。
> 所以有两件事同时成立：①方法内**修改对象内容**会被调用方看到（因为两把钥匙开同一扇门）；②方法内**把参数指向新对象**，调用方看不到（因为只换了副本钥匙）。
> 真正的"引用传递"（如 C++ 的 `int*&`）能让方法**改变调用方的变量本身**，Java 做不到——这正是它被判定为值传递的依据。

#### 追问 2：`Object.clone()` 是浅拷贝还是深拷贝？为什么？

**【追问场景】**
拷问题的基础，但"为什么"部分是分水岭。面试官想看你有没有想过 `clone` 的实现原理。

**【❌ 错误答法】**
> "是深拷贝，因为是复制对象。"
> 错在哪：把"复制对象本身"当成了"深拷贝"。判断标准是**引用字段指向的对象有没有被复制**。

**【✅ 正确答法】**
> 是**浅拷贝**。因为 `Object.clone()` 是 native 方法，实现本质是"分配一块同样大小的内存 + 逐字节拷贝（memcpy）"。对于引用类型的字段，拷贝过去的只是**引用地址**，所以新老对象共享同一个子对象。
> 举例：`Person` 里有一个 `List` 字段，clone 之后 `p1.getList() == p2.getList()` 为 true，改一个另一个也跟着变；数组的 `clone()` 同理（`int[][]` 复制后内层数组仍共享）。
> 要深拷贝必须手动处理引用字段（重新 `new`）、或者用序列化/JSON 转换、或者写拷贝构造器。

#### 追问 3：`BigDecimal` 用 `equals` 比较金额会出什么问题？

**【追问场景】**
生产事故级的细节题。考察你有没有真的写过金额相关代码。

**【❌ 错误答法】**
> "没区别，都能比较是否相等。"
> 错在哪：`equals` 把 **scale** 也算进去了，`1.0` 和 `1.00` 会被判为不等。

**【✅ 正确答法】**
> 会**误判为不相等**。`BigDecimal.equals` 的实现里先比较 `scale`，再看 unscaled value，所以 `new BigDecimal("1.0").equals(new BigDecimal("1.00"))` 返回 **false**，而它们的数值其实相等。
> 后果：如果金额经过不同路径计算（一个 `setScale(2)` 过、一个没有），用 `equals` 判断"是否已支付/是否一致"就会出错，甚至在 `HashMap`/`HashSet` 里被当成不同的 key。
> 正确做法：**一律用 `compareTo(other) == 0` 判断数值相等**；如果要放进 Map 做 key，先统一 `setScale(2, RoundingMode.HALF_UP)` 再用 `stripTrailingZeros()` 之类的规范化处理。

#### 追问 4：`new BigDecimal(0.1)` 为什么出现一长串数字？

**【追问场景】**
金额计算必坑。能说出 `valueOf` 和构造器的差别就是真懂。

**【❌ 错误答法】**
> "BigDecimal 精度高，所以把底层真实值打出来了，是正常的。"
> 错在哪：结论没错但没给出正确用法。面试官问这个问题是想知道**你会不会用错**。

**【✅ 正确答法】**
> 因为 `0.1` 在二进制里无法精确表示，`double` 存的是它最接近的近似值。`new BigDecimal(double)` 的构造器会**原样保留这个二进制精确值**，于是打印出 `0.1000000000000000055511151231257827021181583404541015625`。
> 而 `BigDecimal.valueOf(0.1)` 内部是 `new BigDecimal(Double.toString(0.1))`，`Double.toString` 会输出最短可往返的十进制表示 `"0.1"`，所以得到精确的 0.1。
> 结论：**永远不要用 `new BigDecimal(double)`**，要么用字符串构造 `new BigDecimal("0.1")`，要么用 `BigDecimal.valueOf(0.1)`。
> 顺带一提：涉及金额还有两条铁律——`BigDecimal` 不可变（`a.add(b)` 要接返回值）、`divide` 必须指定 `scale` 和 `RoundingMode` 否则除不尽直接抛 `ArithmeticException`。

#### 追问 5：怎么判断一个数是偶数？`%` 和 `&` 有区别吗？

**【追问场景】**
看起来简单，但负数是陷阱。考位运算的实际理解。

**【❌ 错误答法】**
> "用 `% 2` 比较，`n % 2 == 0` 就是偶数，两者没区别。"
> 错在哪：对**负数**，`-3 % 2` 在 Java 里是 `-1`，虽然 `!= 0` 依然判断正确，但如果写成 `n % 2 == 1`（常见错误写法）就会把负数判错；而 `&` 没有这个问题，性能也更好。

**【✅ 正确答法】**
> 判断偶数用 **`(n & 1) == 0`** 更稳：`&` 直接取二进制最低位，正负都正确，且比取模快。
> 如果一定要用取模，**必须写 `n % 2 == 0` 判断偶数**，绝不能写 `n % 2 == 1` 判断奇数——因为负数取模结果可能是 `-1`（如 `-3 % 2 == -1`），会漏判所有负奇数。
> 补充：这个位运算还有个应用场景，就是 HashMap 用 `(n-1) & hash` 代替取模来定位桶，前提是 n 必须是 2 的幂（这样 `&` 才等价于 `%`）。

#### 追问 6：`>>` 和 `>>>` 有什么区别？

**【追问场景】**
基础但常错，尤其在哈希、编解码场景。

**【❌ 错误答法】**
> "都是右移，一样。"
> 错在哪：对负数结果完全不同。

**【✅ 正确答法】**
> 区别在**空出来的高位补什么**：
> `>>` 是算术右移，**补符号位**——正数补 0、负数补 1，等同于"除以 2 的幂"且保持符号。例如 `-8 >> 2 == -2`。
> `>>>` 是逻辑右移，**一律补 0**，结果当成无符号数看。例如 `-8 >>> 2 == 1073741822`。
> 实际应用：**哈希扰动要用 `>>>`**（如 `h ^ (h >>> 16)`），因为要用高位移到低位、且不能被符号位污染；而无符号整数解析（如读取 ushort/uint）也必须用 `>>>`。

### 12.5 动手验证

```java
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.ArrayList;
import java.util.List;

public class ValueCopyDecimalDemo {
    // 值传递
    static void changeInt(int x) { x = 100; }
    static void changeArray(int[] arr) { arr[0] = 100; }
    static void reassignArray(int[] arr) { arr = new int[]{999}; }

    // 浅拷贝演示
    static class Person implements Cloneable {
        String name;
        List<String> hobbies = new ArrayList<>();
        @Override public Person clone() {
            try { return (Person) super.clone(); }
            catch (CloneNotSupportedException e) { throw new AssertionError(e); }
        }
        // 深拷贝版本
        public Person deepClone() {
            Person c = clone();
            c.hobbies = new ArrayList<>(this.hobbies);   // 引用字段重建
            return c;
        }
    }

    public static void main(String[] args) {
        // ===== 值传递 =====
        int a = 1; changeInt(a);
        System.out.println("基本类型: " + a);            // 1

        int[] arr = {1, 2, 3};
        changeArray(arr);
        System.out.println("改内容: " + arr[0]);          // 100
        reassignArray(arr);
        System.out.println("换指向: " + arr[0]);          // 100

        // ===== 深浅拷贝 =====
        Person p1 = new Person();
        p1.name = "Tom";
        p1.hobbies.add("读书");

        Person shallow = p1.clone();
        shallow.hobbies.add("游泳");
        System.out.println("浅拷贝影响原对象: " + p1.hobbies);   // [读书, 游泳]

        Person deep = p1.deepClone();
        deep.hobbies.add("跑步");
        System.out.println("深拷贝不影响原对象: " + p1.hobbies); // [读书, 游泳]

        // ===== 浮点精度 =====
        System.out.println("0.1 + 0.2 = " + (0.1 + 0.2));       // 0.30000000000000004
        System.out.println("== 0.3 ? " + (0.1 + 0.2 == 0.3));   // false

        // ===== BigDecimal =====
        System.out.println("new BigDecimal(0.1) = " + new BigDecimal(0.1));
        System.out.println("valueOf(0.1)       = " + BigDecimal.valueOf(0.1));

        BigDecimal x = new BigDecimal("1.0"), y = new BigDecimal("1.00");
        System.out.println("equals: " + x.equals(y) + ", compareTo==0: " + (x.compareTo(y) == 0));  // false, true

        try {
            new BigDecimal("1").divide(new BigDecimal("3"));
        } catch (ArithmeticException e) {
            System.out.println("除不尽异常: " + e.getMessage());
        }
        System.out.println("正确除法: " + new BigDecimal("1").divide(new BigDecimal("3"), 2, RoundingMode.HALF_UP)); // 0.33

        // ===== 位运算 =====
        System.out.println("-8 >> 2  = " + (-8 >> 2));     // -2
        System.out.println("-8 >>> 2 = " + (-8 >>> 2));    // 1073741822
        System.out.println("奇数判断 -3 & 1 = " + (-3 & 1)); // 1
    }
}
```

### 12.6 一句话速记

- **Java 只有值传递**：基本类型传值副本，对象传**引用的副本**——能改内容，不能换指向；`swap` 永远无效。
- `Object.clone()` 是 **native 浅拷贝**（逐字节 memcpy）+ 必须实现 `Cloneable`（空标记）+ `protected`；深拷贝要手动重建引用字段 / 序列化 / 拷贝构造器。
- `0.1+0.2 != 0.3`（二进制无法精确表示）；**`BigDecimal` 用字符串或 `valueOf` 构造，比较用 `compareTo`，`divide` 必须给 scale + RoundingMode**。
- `>>` 补符号位、`>>>` 补 0；**判偶用 `(n & 1) == 0`**，哈希扰动必须用 `>>>`。

---

## 自测题（含答案要点）

> 用法：先自己在纸上答，再对照下方要点。**答不出的题就是你的复习清单**，回到对应小节重读。

**1.（1.2）`new User()` 时 JVM 的执行顺序是什么？**
答案要点：类初始化（若未初始化，父类优先）→ 分配内存（TLAB）→ 全字段置零 → 设置对象头 → 执行构造器（`super()` → 字段显式初始化 → 实例初始化块 → 构造器体）→ 引用赋值。

**2.（1.4）构造器里调用可重写方法有什么问题？**
答案要点：父类构造器执行时子类字段初始化还没跑，被重写的方法读到的是**零值**；外部可能拿到半初始化对象。修复：改成 `private`/`final`，或延后到工厂方法/`@PostConstruct`。

**3.（1.5）空对象占多少字节？加一个 `int` 呢？加 `long` 呢？**
答案要点：64 位 + 指针压缩：Mark Word 8 + Klass 4 = 12 → 补齐 **16**；加 `int` 还是 16；加 `long` 变 20 → 补齐 **24**。

**4.（2.2）`String s = new String("abc")` 创建几个对象？**
答案要点：池中有 "abc" 则 1 个（堆上新对象），没有则 2 个（池中常量 + 堆对象）。`new` 一定创建新对象。

**5.（2.4）`intern()` 在 JDK 6 和 JDK 7 的行为差异，原因是什么？**
答案要点：JDK 6 复制字符串到永久代池中，返回副本；JDK 7 起常量池移到堆，直接把堆上对象引用登记进池，返回原对象。原因是池的位置迁移。

**6.（2.4）字符串常量池在堆还是元空间？**
答案要点：**JDK 7 起在堆**（底层是 `StringTable` 哈希表，`-XX:StringTableSize` 可调）；永久代/元空间是方法区的实现，元空间只放类元数据。

**7.（3.2）重写 `equals` 不重写 `hashCode`，在 `HashSet` 里会怎样？**
答案要点：hashCode 不同 → 落到不同桶 → 去重失效（size 变大）、`contains` 返回 false、`Map.get` 拿不到值。因为哈希集合是"先按 hash 分桶、再桶内 equals"。

**8.（3.4）`hashCode` 相等，`equals` 一定相等吗？**
答案要点：不一定，这是哈希冲突（如 `"Aa"` 与 `"BB"` 的 hashCode 都是 2112）。契约是单向的：equals 相等 → hashCode 必须相等。

**9.（4.2）`Integer a = 128, b = 128; a == b` 的结果及原因？如何让它相等？**
答案要点：false，因为 128 超出 `IntegerCache`（-128~127）范围，`valueOf` 会 `new`。可用 `-XX:AutoBoxCacheMax` 扩大上限，但生产应改用 `equals`。

**10.（4.4）`Integer i = null; int j = flag ? i : 0;` 为什么 NPE？**
答案要点：三元表达式两边是 `Integer` 和 `int`，二元数值提升把结果类型定为 `int`，于是 `i` 被强制拆箱，`null.intValue()` 抛 NPE，与 `flag` 取值无关。

**11.（5.2）静态变量在哪个内存区？**
答案要点：**JDK 7 起在堆**，随 `Class` 对象存放；类元数据在元空间。答"在元空间"是常见错误。

**12.（5.4）哪些引用不会触发类初始化？**
答案要点：通过子类访问父类静态字段、定义数组（`new Parent[10]`）、访问编译期常量（`static final` + 字面量）。注意：非编译期常量（运行时求值）会触发。

**13.（6.4）`Animal a = new Dog(); a.speak();` 调用谁？改成 static 呢？**
答案要点：实例方法走 `invokevirtual` 动态分派 → `Dog` 版本；`static` 走 `invokestatic` 静态绑定 → 看**声明类型** `Animal` 的版本（隐藏而非重写）。

**14.（6.4）有 `f(int)` 和 `f(Integer)` 两个重载，`f(1)` 调哪个？**
答案要点：`f(int)`。重载解析阶段 1（不允许装箱/拆箱）就命中，不会进入阶段 2。

**15.（7.4）两个接口有同名 `default` 方法，实现类怎么办？**
答案要点：编译报错，必须显式覆写；可用 `A.super.hello()` 指定。优先级：**类优先 → 接口冲突必须覆写 → 子接口优先**。另：接口不能 default 覆盖 `Object` 的 public 方法。

**16.（7.4）匿名内部类为什么要求局部变量 effectively final？**
答案要点：局部变量在栈上、方法结束即销毁，而匿名类对象可能活更久，所以编译器把值**拷贝**进匿名类字段；既然只是拷贝，原变量就不能再改，否则语义不一致。

**17.（8.2）什么是桥接方法？为什么需要？**
答案要点：子类实现泛型接口时，若方法签名与擦除后的签名不同（`compareTo(Node)` vs `compareTo(Object)`），编译器生成 `ACC_BRIDGE` 方法做参数强转后转调，保证擦除后多态成立。反射需用 `isBridge()` 过滤。

**18.（8.4）为什么 `? extends T` 不能 `add`，`? super T` 不能 `get` 出具体类型？**
答案要点：`? extends` 的真实类型不确定（可能是任意子类），写入不安全；`? super` 的真实类型可能是 `Object` 级别，取出只能保证 `Object`。PECS：读 extends、写 super。

**19.（9.4）`try { return 1; } finally { return 2; }` 返回什么？`try` 抛异常 + `finally return` 呢？**
答案要点：返回 **2**（覆盖返回值）；第二种情况方法**正常返回**，异常被彻底吞掉。禁止在 `finally` 里 return。

**20.（9.4）`try-with-resources` 中 `close()` 抛异常会覆盖主异常吗？**
答案要点：不会。`close` 的异常通过 `addSuppressed()` 挂到主异常上，可用 `getSuppressed()` 查看；关闭顺序是**反序**。

**21.（10.4）`Class.forName` 和 `ClassLoader.loadClass` 的区别？**
答案要点：`forName` 默认 `initialize=true`，触发类初始化（执行静态块）；`loadClass` 不初始化。JDBC 靠 `forName` 触发驱动静态块完成注册。

**22.（10.4）`InvocationHandler.invoke` 的第一个参数是什么？**
答案要点：**代理对象自己**（`$Proxy0` 实例），不是 target。想调真实对象要自己持有 target 引用；写成 `method.invoke(proxy, args)` 会无限递归。

**23.（11.4）反序列化会调用构造器吗？有什么风险？**
答案要点：不会（`allocateInstance` 绕过构造器），所以构造器校验失效；是反序列化 RCE 漏洞的根源。防护：不反序列化不可信数据、`ObjectInputFilter` 白名单、改用 JSON/Protobuf。

**24.（11.4）单例怎么防序列化和反射破坏？**
答案要点：序列化 → `private Object readResolve()` 返回 `INSTANCE`；反射 → 构造器内做存在性检查抛异常；**枚举单例两者天然免疫**，最推荐。

**25.（12.4）Java 是值传递还是引用传递？`swap` 为什么失败？**
答案要点：只有值传递；对象传的是引用副本——能改内容、不能换指向。`swap` 失败因为交换的是栈上形参副本。

**26.（12.4）`BigDecimal` 的 `equals` 与 `compareTo` 区别？**
答案要点：`equals` 比 scale + 数值（`1.0` vs `1.00` 为 false）；`compareTo` 只比数值。金额比较一律用 `compareTo(...) == 0`。

**27.（12.4）`>>` 与 `>>>` 的区别？哈希为什么用 `>>>`？**
答案要点：`>>` 补符号位（`-8>>2 = -2`），`>>>` 补 0（`-8>>>2 = 1073741822`）。哈希扰动要把高位搬到低位且不被符号位污染，所以用 `>>>`。

---

## 背诵卡片

> 面试前 30 分钟扫这一节，一条读 2 秒，读完就上。

**对象与内存**
1. `new` 五步：类初始化 → 分配（TLAB）→ 清零 → 对象头 → 构造器；**分配和构造是两个阶段**。
2. 空对象 **16 字节** = Mark Word 8 + Klass 4 + 对齐 4。
3. 实例顺序：**super() → 字段初始化 → 初始化块 → 构造器体**；static 属于类初始化，一次且最早。
4. 构造器里调用可重写方法 → 子类字段还是零值。
5. 对象分配优先走 **TLAB**，目的是免同步；内存规整用指针碰撞，有碎片用空闲列表。
6. 构造器抛异常 → 对象不可达可回收，但若 `this` 已泄露则成僵尸对象。

**String**
7. 字面量进池、`new` 必在堆；`==` 比同一条、`equals` 比内容。
8. 字符串常量池 **JDK 7 起在堆**（不在元空间）。
9. `intern`：JDK 6 复制进池，JDK 7+ 存堆引用。
10. 不可变四理由：可共享、hash 可缓存、线程安全、可做 key。
11. `value` 是 `private final` + 构造器**防御性拷贝**，才真正不可变。
12. JDK 9+ 紧凑字符串：`byte[] + coder`，LATIN1 省一半内存。
13. 循环拼接必须把 `StringBuilder` 提到循环外；`final` + 字面量会被编译期折叠。
14. `switch(String)` = hashCode 分段 + equals 精确比较。

**equals / hashCode**
15. 契约方向只有一个：**equals 相等 → hashCode 必须相等**。
16. 只重写 equals → 放进 Set 去重失效、Map 取不出值。
17. hash 冲突不等于 equals 相等；`"Aa"` 与 `"BB"` 的 hashCode 都是 2112。
18. 模板：`this==o` → `o==null || getClass()!=o.getClass()` → 逐字段 `Objects.equals`。
19. 31 是奇素数且 `31*i = (i<<5)-i` 可优化；`Objects.hash` 会创建数组。
20. `getClass` 与 `instanceof` 不能在继承体系里混用；Lombok 用 `canEqual` 保对称性。
21. 可变对象做 key → 改完字段就取不出值（静默 bug）。

**包装类**
22. `Integer a = 1` 编译成 `valueOf(1)`，**-128~127 走缓存**。
23. 缓存：Integer 上限可调（`-XX:AutoBoxCacheMax`）；Byte/Short/Long/Character(0~127)/Boolean 有；**Float/Double 无**。
24. 三元运算符一边包装一边基本类型 → 隐式拆箱 → null 直接 NPE。
25. 包装类比较一律 `equals`；`Long` 与 `Integer` 值相同也互不相等。

**final / static / 初始化**
26. `final` 锁**引用**不锁内容；final 字段有"构造完成即可见"的内存语义。
27. **JDK 7 起静态变量在堆**；元空间只放类元数据。
28. 顺序口诀：先静态后实例、先父后子、静态只一次；静态块与字段**按源码顺序**。
29. 被动引用三连不触发初始化：子类引用父类静态字段、定义数组、访问编译期常量。

**重载 / 重写 / 多态**
30. **重载看声明（编译期），重写看实际（运行期）**。
31. 重载解析三阶段：严格 → 允许装箱 → 变长参数，前一阶段命中即停。
32. `invokevirtual`/`invokeinterface` 才多态；static/private/final/构造器都不能重写。
33. **字段没有多态**；重写返回类型可协变、权限不能收紧、受检异常不能变宽。
34. 永远写 `@Override`，防止把重写写成重载。

**接口 / 内部类**
35. 接口 = 能力契约（可多实现、只有常量、无构造器）；抽象类 = 类型模板。
36. `default` 的动机是**接口演进不破坏兼容**；冲突三规则：类优先 → 显式覆写 → 子接口优先。
37. 接口不能 default 覆盖 `Object` 的 public 方法（`equals`/`hashCode`/`toString`）。
38. 非静态内部类编译后带 `this$0` → **持有外部引用 → 内存泄漏**，优先用 `static` 嵌套类。
39. 匿名类捕获局部变量是值拷贝 → 必须 effectively final。
40. Lambda 的 `this` 指外层类；匿名类的 `this` 指自己。

**泛型**
41. 擦除：无界 → `Object`，有界 → **第一个上界**；元数据留在 `Signature` 属性（反射可取）。
42. 桥接方法 `ACC_BRIDGE` 是擦除逼出来的多态补丁，反射要用 `isBridge()` 过滤。
43. PECS：**读 extends、写 super**；`? extends` 不能 add、`? super` 只能 get 出 Object。
44. 不能 `new T()`/`new T[]`/`instanceof T`/静态成员用 T；创建实例传 `Class<T>` 或 `Supplier<T>`。
45. `List<String>` 与 `List<Integer>` 运行时是同一个类；泛型是编译期护栏。

**异常**
46. 受检 = 编译器逼你处理的外部风险；非受检 = 程序 bug；`NumberFormatException` 是 unchecked。
47. `try-with-resources`：自动关闭 + **反序**关闭 + `addSuppressed` 避免异常吞噬。
48. `finally` 里 return → **覆盖返回值 + 吞异常**，禁止。
49. `finally` 不是"一定执行"：`System.exit`、JVM 崩溃、`kill -9`、daemon 线程被终止。
50. 异常成本在 `fillInStackTrace`；别用异常做流程控制。

**反射 / 代理 / 注解**
51. `Class.forName` 触发初始化，`loadClass` 不触发（JDBC 靠前者）。
52. `getMethods` = public + 继承；`getDeclaredMethods` = 本类全部但无继承。
53. **JDK 动态代理必须基于接口**（生成的类 `extends Proxy`）；CGLIB 生成子类。
54. `invoke(proxy, method, args)` 里 **proxy 是代理对象自己**，不是 target（传它调会无限递归）。
55. Spring Boot 2.x 起 **AOP 默认 CGLIB**。
56. 注解 `@Retention(RUNTIME)` 才能被反射读取；注解本身不做事，靠处理器。
57. JDK 9+ `setAccessible` 受模块限制，需 `--add-opens`。

**Lambda / 序列化**
58. Lambda 编译成 `invokedynamic`，**不产生 class 文件**；无捕获时是单例。
59. `Serializable` 是空标记接口；`serialVersionUID` 做版本校验，不写就随结构变化而失败。
60. 反序列化**不走构造器**（绕过校验，漏洞根源）；`transient` 字段为默认值。
61. 单例防破坏：`readResolve` 防序列化；**枚举单例两者通吃**。

**值传递 / 拷贝 / 精度 / 位运算**
62. **Java 只有值传递**：对象传引用的副本——能改内容、不能换指向；`swap` 永远无效。
63. `Object.clone()` 是 native **浅拷贝**（逐字节）+ 必须实现 `Cloneable` + `protected`。
64. 深拷贝：手动重建引用字段 / 序列化 / JSON / 拷贝构造器（推荐后两者）。
65. `0.1+0.2 != 0.3`；`BigDecimal` 用**字符串或 valueOf** 构造，比较用 `compareTo`。
66. `divide` 必须给 scale + `RoundingMode`，否则除不尽抛 `ArithmeticException`；金额常用 `setScale(2, HALF_UP)`。
67. `>>` 补符号位、`>>>` 补 0；判偶用 `(n & 1) == 0`；`BigDecimal` 不可变，`a.add(b)` 要接返回值。

