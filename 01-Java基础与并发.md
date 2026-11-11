# Java 基础与并发（26 题）

> 适用：后端开发 / AI 应用开发实习。Java 是校招笔试面试的地基，这部分答不好，后面项目再漂亮也会被扣分。
> 使用建议：先看「一句话答案」自测，说不出来再看解析；标 ★ 的是大厂高频。
> 复习节奏：第一天通读，第二天只背 ★，面试前 30 分钟扫「坑」和「追问」。

---

## 一、语言基础

### Q1 ★ 为什么重写 equals 必须重写 hashCode？

**一句话答案：** 因为 HashMap/HashSet 先用 hashCode 定位桶、再用 equals 比较，只重写 equals 会导致两个「相等」的对象落在不同桶里，查不到、放不进。

**详细解析：**
- 契约：`equals` 相等的两个对象，`hashCode` 必须相等；反之不要求。
- 如果只重写 equals：两个对象 equals 返回 true，但 hashCode 用的是 Object 的地址散列，值不同 → 落到不同桶 → `map.get(new Key("a"))` 取不到刚 put 的值。
- 如果只重写 hashCode：不同对象可能同桶，equals 用地址比较返回 false → 哈希冲突链不断变长，性能退化但结果仍「正确」。
- 正确写法：用 `Objects.hash(...)` 或 IDE 生成，保证参与 equals 的字段都参与 hashCode。

**追问：** 对象放进 HashSet 后修改了参与哈希的字段会怎样？→ 哈希值变了，按新哈希找不到它原来的位置，等于「丢在集合里」，所以作为 key 的对象应该是不可变的。

**坑：** 说「不重写 hashCode 会编译报错」——不会，是逻辑错误，不是编译错误。

### Q2 ★ == 和 equals 的区别，Integer 缓存陷阱

**一句话答案：** `==` 比较值（基本类型）或引用地址（对象）；`equals` 默认也是地址，被重写后比内容。Integer 在 -128~127 有缓存，超出范围 `==` 为 false。

**详细解析：**
- `Integer a = 127, b = 127; a == b` → true（同一缓存对象）；`128` 则 false（各自 new）。
- `Integer` 与 `int` 比较会触发拆箱，比较的是数值，所以 `a == 128` 为 true。
- 字符串常量池：`"ab"` 直接入池，`new String("ab")` 在堆上；`intern()` 把堆上的字符串尝试放入池并返回池中引用。
- 编译期常量折叠：`"a" + "b" == "ab"` 为 true；含变量则 false。

**追问：** 为什么要有缓存？→ 小整数使用频率极高，缓存能减少对象创建和 GC。

**坑：** 用 `==` 比较两个 `Integer` 是真实事故高发点，代码评审时要盯。

### Q3 ★ String、StringBuilder、StringBuffer 与字符串常量池

**一句话答案：** String 不可变，每次拼接生成新对象；StringBuilder 非线程安全但快；StringBuffer 方法加 synchronized，线程安全但慢。

**详细解析：**
- 不可变的实现：`final char[]`（JDK 9 后是 `byte[] + coder`），且不暴露修改方法。
- 不可变的收益：可安全共享、可作为 HashMap key（哈希不变）、天然线程安全、常量池复用。
- 循环里用 `+` 拼接会被编译器优化成 StringBuilder（仅限单条表达式），跨循环仍是每次 new。
- JDK 9 起 String 底层改成 byte[] + 编码标志，节省内存（Latin-1 只占 1 字节/字符）。

**追问：** 字符串常量池在哪？→ JDK 7 起移到堆中（原先在永久代），避免永久代溢出。

### Q4 自动拆箱会带来什么问题？

**一句话答案：** 空指针。声明为包装类型的字段/返回值如果为 null，参与运算或比较时会自动拆箱抛 NPE。

**详细解析：**
- `Integer i = null; int j = i;` → NPE。
- 三元表达式陷阱：`condition ? 1 : null` 会因类型推断触发拆箱。
- Map 里取不存在的 key 返回 null，直接参与算术运算就会炸。
- 规范：金额用 BigDecimal；计数用基本类型；集合泛型只能用包装类，取用时判空。

**追问：** BigDecimal 的 equals 和 compareTo 区别？→ equals 同时比数值和 scale（`1.0` 与 `1.00` 不等），比较大小必须用 `compareTo`。

### Q5 final、finally、finalize 分别是什么？

**一句话答案：** final 是修饰符（不可变/不可继承/不可重写）；finally 是异常处理的必执行块；finalize 是 GC 前的回调，已废弃。

**详细解析：**
- final 修饰类 → 不可继承；修饰方法 → 不可重写；修饰变量 → 引用不可变（对象内容仍可变）。
- finally 不执行的情况：`System.exit()`、JVM 崩溃、守护线程被结束；finally 里的 return 会覆盖 try 里的 return（大坑）。
- finalize 执行时机不确定、可能拖慢 GC，JDK 9 起标记废弃，用 `Cleaner` 或 try-with-resources 替代。

**追问：** try-with-resources 的关闭顺序？→ 逆序关闭，先声明的后关。

### Q6 ★ 重载与重写的区别，静态绑定发生在什么时候？

**一句话答案：** 重载是编译期按参数列表选方法（静态绑定）；重写是运行期按实际对象类型选方法（动态绑定，靠虚方法表）。

**详细解析：**
- 重载只看方法名 + 参数列表，与返回值、异常无关。
- 重写要求：方法签名相同、访问权限不能更小、不能抛更宽的受检异常、返回类型可协变。
- 经典陷阱：`Animal a = new Dog(); a.speak()` 调 Dog 的实现（动态绑定）；但 `a.overload(x)` 在编译期按 Animal 类型选。
- 静态方法、私有方法、final 方法不参与动态绑定。

**追问：** 为什么 Java 不支持运算符重载？→ 设计取舍：可读性与简单性优先（唯一例外是 String 的 `+`）。

### Q7 ★ 接口和抽象类怎么选？默认方法解决什么问题？

**一句话答案：** 抽象类表达「是什么」可带状态和构造器，单继承；接口表达「能做什么」多实现无状态。默认方法让接口能新增方法而不破坏已有实现。

**详细解析：**
- Java 8 前接口只能有常量和抽象方法；8 加默认方法/静态方法，9 加私有方法。
- 默认方法冲突：两个接口有同名默认方法，实现类必须显式重写并用 `Interface.super.method()` 指定。
- 选择标准：需要共享实现和字段 → 抽象类；需要多重能力组合、跨继承体系 → 接口。

**追问：** 接口能有构造器吗？→ 不能。接口的字段隐式是 `public static final`。

## 二、集合

### Q8 ★ ArrayList 与 LinkedList 的区别，ArrayList 扩容机制

**一句话答案：** ArrayList 基于动态数组，随机访问 O(1)、中间插入 O(n)；LinkedList 基于双向链表，头尾操作 O(1)、随机访问 O(n)。ArrayList 默认容量 10，扩容 1.5 倍。

**详细解析：**
- 扩容：`newCapacity = oldCapacity + (oldCapacity >> 1)`，用 `Arrays.copyOf` 复制，代价 O(n)。
- `ensureCapacity` 可预分配，避免多次扩容；`trimToSize` 收缩。
- 删除元素会 `System.arraycopy` 移动后续元素；`removeIf` / `Iterator.remove` 避免并发修改异常。
- 实践中 99% 场景用 ArrayList，LinkedList 只在频繁头插或当 Deque 用时有优势，且内存开销更大（每个节点多两个指针）。

**追问：** 为什么 ArrayList 的 `elementData` 用 transient 修饰？→ 序列化时自己实现，只写实际元素，避免序列化整个容量数组。

### Q9 ★ HashMap 的底层结构与 put 流程

**一句话答案：** 数组 + 链表 + 红黑树。put 时先算 hash（扰动函数），再定位桶，命中就用 equals 比较覆盖，否则尾插；链表长度 ≥8 且容量 ≥64 时转红黑树；元素数超过阈值（容量 × 0.75）扩容两倍。

**详细解析：**
- hash 扰动：`h = key.hashCode(); h ^= h >>> 16`，让高位参与运算，减少碰撞。
- 定位：`(n - 1) & hash`，n 是 2 的幂，等价于取模但更快。
- 树化条件：链表长度 ≥ 8 且数组容量 ≥ 64（否则先扩容）；退化条件：红黑树节点 ≤ 6。
- 扩容：容量翻倍，元素要么留在原索引，要么移动到「原索引 + 旧容量」，无需重新计算 hash。
- JDK 7 头插法在并发扩容时会形成环形链表导致死循环；JDK 8 改为尾插，但仍非线程安全。

**追问：** 为什么用 8 作为树化阈值？→ 泊松分布下链表长度到 8 的概率约千万分之六，是空间与查询效率的折中。

**坑：** HashMap 非线程安全，多线程 put 会丢数据，不要用「JDK 8 尾插所以安全了」来回答。

### Q10 ★ ConcurrentHashMap 1.7 与 1.8 的区别

**一句话答案：** 1.7 用分段锁 Segment（继承 ReentrantLock），锁住一段；1.8 去掉 Segment，用 CAS + synchronized 锁单个桶头节点，粒度更细，并引入红黑树。

**详细解析：**
- 1.7：Segment 数组，每个 Segment 是一个小 HashMap，默认 16 段，并发度 16。
- 1.8：`Node` 数组，put 时若桶为空用 CAS 插入，否则 `synchronized(头节点)` 加锁；扩容支持多线程协同迁移（`ForwardingNode` 标记）。
- size 统计：1.8 用 `CounterCell` 数组分散计数，避免热点。
- 为什么不直接用 `Collections.synchronizedMap`：全局一把锁，并发度 1。

**追问：** get 需要加锁吗？→ 不需要，Node 的 val 和 next 用 volatile 修饰，保证可见性。

### Q11 fail-fast 与 fail-safe 是什么？

**一句话答案：** fail-fast 在遍历时检测到结构性修改立即抛 `ConcurrentModificationException`（ArrayList、HashMap）；fail-safe 遍历副本，不抛异常但可能读不到最新数据（CopyOnWriteArrayList、ConcurrentHashMap 的迭代器）。

**详细解析：**
- 实现：集合维护 `modCount`，迭代器每次 next 比较 `expectedModCount`。
- 正确删除：用 `Iterator.remove()` 或 `removeIf`；`for-each` 里直接 `list.remove` 会抛异常。
- COW 的代价：写时复制整个数组，写多场景不可用；适合读多写少的监听器列表。

**追问：** COW 为什么线程安全？→ 写时加锁复制新数组，读的始终是旧数组的稳定快照。

## 三、并发基础

### Q12 ★ 线程有哪几种状态？怎么优雅地停止一个线程？

**一句话答案：** NEW、RUNNABLE、BLOCKED、WAITING、TIMED_WAITING、TERMINATED 六种；停止线程用中断标志（`interrupt` + `isInterrupted`），不要用 `stop`。

**详细解析：**
- BLOCKED 是等 synchronized 锁；WAITING 是 `wait/join/park` 无超时；TIMED_WAITING 带超时。
- `interrupt()` 设置中断标志；线程阻塞在 `sleep/wait/join` 时会抛 `InterruptedException` 并清除标志。
- 正确姿势：循环里检查 `Thread.currentThread().isInterrupted()`，捕获 InterruptedException 后要么恢复中断标志，要么退出。
- `stop()` 会释放所有锁、留下不一致状态，已废弃；`suspend/resume` 会死锁。

**追问：** 为什么不推荐用 volatile 标志位停止线程？→ 线程阻塞在 `wait/sleep` 时根本检查不到标志，只能靠中断唤醒。

### Q13 ★ synchronized 的原理，锁升级过程

**一句话答案：** 基于对象头 Mark Word 和 Monitor（monitorenter/monitorexit）。锁状态按 无锁 → 偏向锁 → 轻量级锁（CAS 自旋）→ 重量级锁（OS 互斥量）升级，只升不降。

**详细解析：**
- 对象头：Mark Word 存 hash、GC 年龄、锁标志位；重量级锁时指向 Monitor。
- 偏向锁：第一个线程获取后记录线程 ID，后续同一线程无需 CAS（JDK 15 起默认禁用，因为撤销成本高）。
- 轻量级锁：CAS 把 Mark Word 指向栈上的 Lock Record，失败则自旋。
- 重量级锁：自旋失败进入 `ObjectMonitor` 的 EntryList 排队，涉及内核态切换。
- 锁消除（逃逸分析）、锁粗化（相邻同步块合并）是 JIT 优化。

**追问：** synchronized 能保证可见性吗？→ 能。解锁前把共享变量刷回主存，加锁时清空工作内存，等价于 volatile 的语义加强版。

### Q14 ★ volatile 能保证什么？不能保证什么？

**一句话答案：** 保证可见性和有序性（禁止指令重排），不保证原子性。

**详细解析：**
- 可见性：写操作后插入 StoreLoad 屏障，强制刷主存并使其他 CPU 缓存失效。
- 有序性：通过内存屏障禁止「写后读」重排，这是双重检查锁单例必须加 volatile 的原因。
- 不保证原子性：`i++` 是读-改-写三步，volatile 挡不住并发覆盖，要用 AtomicInteger 或加锁。
- 适用：状态标志、一次性安全发布（DCL 单例）。

**追问：** DCL 单例为什么要两次判空？→ 第一次避免每次加锁，第二次防止锁等待期间别的线程已创建。

### Q15 ★ CAS 是什么？ABA 问题怎么解？

**一句话答案：** CAS 是「比较并交换」的原子指令，包含内存值、期望值、新值三要素；ABA 是值被改成 B 又改回 A，CAS 察觉不到，用版本号（AtomicStampedReference）解决。

**详细解析：**
- 实现：`Unsafe.compareAndSwapInt` → CPU 的 `cmpxchg` 指令，配合 `lock` 前缀保证原子。
- 缺点：自旋消耗 CPU、只能保证单个变量原子、ABA。
- 优化：`LongAdder` 用分段 Cell 分散热点，高并发计数远快于 `AtomicLong`。
- 版本号方案：`AtomicStampedReference` 每次修改递增 stamp。

**追问：** 为什么 LongAdder 比 AtomicLong 快？→ 多个线程更新不同 Cell，最后求和，把 CAS 的竞争分散开了。

### Q16 ★ AQS 的原理，ReentrantLock 与 synchronized 的区别

**一句话答案：** AQS 用一个 volatile int state 表示同步状态，加一个 CLH 双向队列存放等待线程，通过 `tryAcquire/tryRelease` 模板方法给子类实现。ReentrantLock 支持可中断、超时、公平锁和多个 Condition，synchronized 由 JVM 实现且自动释放。

**详细解析：**
- 独占模式（ReentrantLock）与共享模式（Semaphore、CountDownLatch）。
- 加锁流程：CAS 抢 state，失败则入队挂起（`LockSupport.park`），前驱释放时唤醒后继。
- 公平锁：检查队列是否有前驱；非公平锁允许插队，吞吐更高但可能饥饿。
- 条件队列：`await` 把线程移到 Condition 队列，`signal` 移回同步队列。

**追问：** 为什么说 ReentrantLock 必须手动解锁？→ 没有 JVM 的 monitorexit 兜底，异常路径不 finally 解锁会永久死锁。

### Q17 ★ 线程池的核心参数，任务提交后的执行流程

**一句话答案：** 七个参数：核心线程数、最大线程数、空闲存活时间、时间单位、工作队列、线程工厂、拒绝策略。流程是「核心线程 → 队列 → 最大线程 → 拒绝」。

**详细解析：**
- 执行顺序：核心线程未满 → 建核心线程；队列未满 → 入队；线程数 < max → 建非核心线程；都满 → 拒绝策略。
- 四种拒绝策略：Abort（抛异常）、CallerRuns（调用线程执行，天然背压）、Discard（静默丢弃）、DiscardOldest（丢最老的）。
- 队列选择决定行为：无界队列（LinkedBlockingQueue）会让 maximumPoolSize 失效；SynchronousQueue 直接交给线程，配合大 max 适合短任务。
- 参数设置：CPU 密集 ≈ 核数 + 1；IO 密集 ≈ 核数 × (1 + 等待时间/计算时间)，最终靠压测定。

**追问：** 线程池里的线程抛异常会怎样？→ `execute` 提交会打印堆栈并销毁该线程、补一个新线程；`submit` 提交异常被封装在 Future 里，不 `get` 就静默吞掉。

**坑：** 用 `Executors.newFixedThreadPool` 建无界队列线程池，OOM 高发点，规范要求手动 new ThreadPoolExecutor。

### Q18 ★ ThreadLocal 的原理与内存泄漏

**一句话答案：** 每个 Thread 持有一个 `ThreadLocalMap`，key 是 ThreadLocal 的弱引用，value 是强引用。key 被 GC 后 value 无法访问但不会被回收，线程池复用线程就会累积泄漏，所以用完必须 `remove()`。

**详细解析：**
- 结构：Thread → ThreadLocalMap → Entry(弱引用 Key, 强引用 Value)。
- 为什么 key 用弱引用：ThreadLocal 对象本身不再被引用时，允许 key 被回收，至少能通过 `expungeStaleEntry` 清理。
- 泄漏根因：value 是强引用，且线程长期存活（线程池）。
- 跨线程传递：`InheritableThreadLocal` 只在 new Thread 时复制，线程池不生效；要用阿里 TTL 或手动透传。

**追问：** 线程池里用 ThreadLocal 要注意什么？→ 任务开始前 set、结束后 finally remove，否则下一个任务可能读到脏数据。

### Q19 CompletableFuture 常用方法，怎么编排多个异步任务？

**一句话答案：** `supplyAsync` 起异步任务，`thenApply` 做转换，`thenCompose` 串两个异步，`thenCombine` 合并两个结果，`allOf` 等全部完成，`exceptionally/handle` 处理异常。

**详细解析：**
- `thenApply` vs `thenCompose`：前者返回普通值，后者返回 CompletableFuture（扁平化）。
- 默认用 ForkJoinPool.commonPool，IO 任务必须传自定义线程池，否则打满公共池影响全局。
- `allOf().join()` 不会抛被包装的异常，要逐个 `get` 或 `handle` 收集。
- 超时：`orTimeout`（抛 TimeoutException）或 `completeOnTimeout`（给默认值）。

**追问：** `thenApplyAsync` 和 `thenApply` 的区别？→ 前者把回调丢到线程池执行，后者在完成线程上同步执行。

## 四、JVM

### Q20 ★ JVM 内存结构，对象创建与内存分配

**一句话答案：** 线程私有（程序计数器、虚拟机栈、本地方法栈）+ 线程共享（堆、方法区/元空间）+ 直接内存。对象在 TLAB 中分配，先栈上分配（逃逸分析）、再 Eden，大对象直接进老年代。

**详细解析：**
- 栈帧：局部变量表、操作数栈、动态链接、返回地址。
- 对象创建步骤：类加载检查 → 分配内存 → 零值初始化 → 设置对象头 → 执行构造方法。
- 分配方式：指针碰撞（规整内存）或空闲列表（碎片内存）；TLAB 解决并发分配。
- 大对象阈值 `-XX:PretenureSizeThreshold`；长期存活对象晋升老年代（`MaxTenuringThreshold`）。
- 元空间在本地内存，不再有 PermGen OOM，但仍有 `Metaspace` 大小限制。

**追问：** 为什么 JDK 8 用元空间替永久代？→ 永久代大小难预估、容易 OOM，元空间用本地内存更灵活，也便于和 JRockit 合并。

### Q21 ★ 垃圾回收算法与常见收集器

**一句话答案：** 算法有标记-清除、复制、标记-整理、分代收集。收集器：Serial、ParNew、Parallel、CMS、G1、ZGC。JDK 8 默认 Parallel，JDK 9+ 默认 G1。

**详细解析：**
- 可达性分析：GC Roots（栈引用、静态变量、常量、JNI 引用）出发，不可达即可回收。
- 分代假设：绝大多数对象朝生夕死，所以新生代用复制算法（Eden : S0 : S1 = 8:1:1），老年代用标记-整理。
- CMS：并发标记清除，追求低停顿，问题有浮动垃圾、并发模式失败、内存碎片，JDK 9 废弃、14 移除。
- G1：把堆分成 Region，按回收价值优先（Garbage First），可预测停顿（`-XX:MaxGCPauseMillis`），适合大堆。
- ZGC：染色指针 + 读屏障，停顿与堆大小无关（<1ms），JDK 15 转正。

**追问：** 什么情况会触发 Full GC？→ 老年代空间不足、元空间不足、`System.gc()`、晋升失败（Handle Promotion Failure）、CMS 并发模式失败。

### Q22 类加载过程与双亲委派

**一句话答案：** 加载 → 验证 → 准备 → 解析 → 初始化。双亲委派是「先让父加载器加载，父加载不了才自己加载」，保证核心类不被篡改、类不重复加载。

**详细解析：**
- 三个内置加载器：Bootstrap（C++ 实现，加载 `JAVA_HOME/lib`）、Extension/Platform、Application（classpath）。
- 打破双亲委派的场景：JDBC 的 SPI（父加载器要调子加载器的实现，用线程上下文类加载器）、Tomcat 的 WebApp 隔离、OSGi。
- 初始化时机：new、静态字段访问、反射、子类初始化先初始化父类。

**追问：** 两个不同类加载器加载同一个类，是同一个类吗？→ 不是，JVM 用「类加载器 + 全限定名」确定唯一性。

### Q23 线上 CPU 飙高 / 内存泄漏怎么排查？

**一句话答案：** CPU 高用 `top -Hp` 找线程、`jstack` 看栈、转 16 进制对 nid；内存问题用 `jmap -histo`/`jmap -dump` + MAT 分析对象引用链，或 `jstat` 看 GC 频率。

**详细解析：**
- CPU：`top` 找进程 → `top -Hp pid` 找线程 → `printf %x` 转十六进制 → `jstack pid | grep nid` 定位到具体代码行。
- 死循环、频繁 GC、正则回溯、序列化大对象是常见元凶。
- 内存：`jstat -gcutil pid 1000` 看老年代增长；`jmap -dump:format=b,file=heap.hprof pid` 导出；MAT 看支配树和 GC Roots 路径。
- 工具：Arthas 的 `thread`、`dashboard`、`heapdump`、`trace`。

**追问：** dump 文件太大导不出来怎么办？→ `-XX:+HeapDumpOnOutOfMemoryError` 自动导、或 `jcmd GC.heap_dump` 带 `-live` 只导存活对象。

### Q24 强软弱虚四种引用

**一句话答案：** 强引用不回收；软引用内存不足时回收（做缓存）；弱引用下次 GC 必回收（ThreadLocal key、WeakHashMap）；虚引用只用于回收通知（DirectByteBuffer 的 Cleaner）。

**详细解析：**
- 软引用适合「内存敏感缓存」，但被回收后要能重建。
- 弱引用典型用法：`ThreadLocalMap` 的 key、`WeakHashMap` 的 key，避免 key 泄漏。
- 虚引用必须配合 `ReferenceQueue` 使用，`get()` 永远返回 null。

**追问：** 为什么 ThreadLocalMap 的 key 用弱引用而不是强引用？→ 强引用会导致 ThreadLocal 对象本身也无法回收，泄漏更严重。

### Q25 逃逸分析与标量替换

**一句话答案：** 逃逸分析判断对象是否逃出方法/线程，未逃逸的对象可以栈上分配、标量替换、锁消除，减少堆压力和 GC。

**详细解析：**
- 三种逃逸状态：不逃逸、方法逃逸（返回引用）、线程逃逸（被其他线程持有）。
- 标量替换：把对象拆成若干基本类型变量，连对象头都省了。
- 触发条件苛刻（方法要够热、对象要够小），不要指望它救所有场景。

**追问：** 为什么说「新建对象一定在堆上」是错的？→ 逃逸分析后可在栈上分配，或者直接被标量替换掉。

### Q26 常用的 JVM 调优参数

**一句话答案：** `-Xms/-Xmx`（堆初始/最大，设为相等避免动态扩缩）、`-Xmn`（新生代）、`-XX:MetaspaceSize`、`-XX:+UseG1GC`、`-XX:MaxGCPauseMillis`、`-XX:+HeapDumpOnOutOfMemoryError`、`-Xss`（栈大小）。

**详细解析：**
- 生产必配：堆 dump、GC 日志（`-Xlog:gc*`）、OOM 时自动 dump。
- 堆外内存：Netty/直接内存用 `-XX:MaxDirectMemorySize` 限制。
- G1 调优优先设停顿目标，不要死抠新生代比例。
- 容器环境要加 `-XX:+UseContainerSupport` 或 `-XX:MaxRAMPercentage`，否则按宿主机内存算。

**追问：** `-Xms` 和 `-Xmx` 为什么要设相等？→ 避免运行期反复扩缩堆触发 Full GC，也避免容器被 OOMKilled。

---

## 高频速记（面试前 3 分钟）

| 主题 | 必须脱口而出 |
| --- | --- |
| HashMap | 数组 + 链表 + 红黑树；8 转树、6 退化、0.75 负载、2 倍扩容、扰动函数 |
| ConcurrentHashMap | 1.7 分段锁 → 1.8 CAS + synchronized 锁桶 + 红黑树 |
| synchronized | 对象头 + Monitor；偏向 → 轻量 → 重量；只升不降 |
| volatile | 可见 + 有序，不原子；DCL 单例必加 |
| 线程池 | 7 参数；核心 → 队列 → 最大 → 拒绝；禁用 Executors 工厂 |
| ThreadLocal | Thread 持有 Map，key 弱引用 value 强引用，用完 remove |
| AQS | state + CLH 队列 + 模板方法；独占/共享 |
| JVM 内存 | 栈私有、堆共享、元空间在本地内存 |
| GC | 分代 + 可达性；G1 分 Region 按价值回收；ZGC 染色指针 |
| 排查 | top -Hp → jstack；jstat/jmap + MAT；Arthas |
