# 一、Java 基础与进阶

所有者: junk01

> 覆盖 JVM 底层原理、并发编程、集合框架、Java 新特性，是大厂面试必考模块。
> 

---

## 1. JVM 深度解析

### 1.1 内存模型

JVM 运行时数据区分为线程共享区域和线程私有区域两大类。

**线程共享区域**：堆（Heap）用于存储对象实例和数组，是 GC 的主要区域；方法区（JDK8 后为元空间 Metaspace）存储类信息、运行时常量池、静态变量，使用本地内存，默认无上限。

**线程私有区域**：虚拟机栈存储方法调用的栈帧（包含局部变量表、操作数栈、动态链接、返回地址）；本地方法栈服务于 Native 方法；程序计数器记录当前线程执行的字节码行号，是唯一不会发生 OOM 的区域。

**关键细节**：JDK8 起永久代被元空间替代；字符串常量池在 JDK7 移至堆中；对象优先在 Eden 区分配，大对象直接进老年代（`-XX:PretenureSizeThreshold`）。

### 1.2 对象内存布局

对象在堆内存中由三部分组成：对象头（Header）、实例数据（Instance Data）、对齐填充（Padding）。

对象头包含 Mark Word（8字节，存储锁状态、GC 年龄、hashCode）和类型指针（指向类元数据）。Mark Word 随锁状态变化而变化，这是锁升级的核心机制：无锁 → 偏向锁 → 轻量级锁 → 重量级锁（单向升级，不可降级）。

### 1.3 垃圾回收

**可达性分析**：从 GC Roots 出发，不可达的对象即可回收。GC Roots 包括：虚拟机栈中引用的对象、方法区中静态属性引用的对象、方法区中常量引用的对象、本地方法栈中引用的对象。

**四种引用类型**：

| 引用类型 | 回收时机 | 典型应用 |
| --- | --- | --- |
| 强引用 | 永不回收 | 普通对象 |
| 软引用 SoftReference | 内存不足时 | 图片缓存 |
| 弱引用 WeakReference | 下次 GC | ThreadLocal、WeakHashMap |
| 虚引用 PhantomReference | 随时 | 堆外内存管理 |

**主流垃圾收集器对比**：

| 收集器 | 区域 | 算法 | 特点 | 适用场景 |
| --- | --- | --- | --- | --- |
| Serial | 新生代 | 复制 | 单线程 STW | 客户端 |
| ParNew | 新生代 | 复制 | 多线程版 Serial | 配合 CMS |
| CMS | 老年代 | 标记-清除 | 低停顿，并发收集 | 互联网服务 |
| G1 | 全堆 | 标记-整理+复制 | 可预测停顿，Region 化 | JDK9+ 默认 |
| ZGC | 全堆 | 染色指针 | 停顿小于10ms，TB 级堆 | 超低延迟 |

**G1 核心原理**：将堆划分为大小相等的 Region（1-32MB），逻辑上分 Eden/Survivor/Old/Humongous，优先回收垃圾最多的 Region（Garbage First）。停顿时间可通过 `-XX:MaxGCPauseMillis` 设置目标值。Mixed GC 同时回收新生代和部分老年代。

**CMS 四个阶段**：初始标记（STW，快）→ 并发标记（并发，慢）→ 重新标记（STW，较快）→ 并发清除（并发）。缺点：CPU 敏感、浮动垃圾、内存碎片。

### 1.4 类加载机制

**类加载过程**：加载 → 验证 → 准备 → 解析 → 初始化 → 使用 → 卸载。

准备阶段为类变量分配内存并设置零值（不是代码中赋的值）；初始化阶段执行 `<clinit>()` 方法（静态变量赋值 + 静态代码块）。

**双亲委派模型**：Bootstrap ClassLoader → Extension ClassLoader → Application ClassLoader → 自定义 ClassLoader。向上委派，父加载器无法加载时才自己加载。作用是防止核心类被篡改。

**Tomcat 为何破坏双亲委派**：每个 Web 应用需要独立的类加载器，隔离不同应用的类，支持热部署。SPI 机制（JDBC、JNDI）也通过线程上下文类加载器破坏双亲委派。

---

## 2. Java 并发编程

### 2.1 Java 内存模型（JMM）

JMM 解决多线程下的三个核心问题：可见性、原子性、有序性。每个线程有自己的工作内存，变量操作需从主内存读取到工作内存，修改后刷回主内存。

**happens-before 六大原则**（面试必答）：程序顺序规则、volatile 变量规则（写 hb 读）、传递性规则、锁定规则（unlock hb 后续 lock）、线程启动规则（start() hb 线程内操作）、线程终止规则（线程操作 hb join() 返回）。

### 2.2 synchronized 深度

**锁升级过程**（JDK6 优化）：无锁 → 偏向锁 → 轻量级锁 → 重量级锁

| 锁状态 | 加锁条件 | 实现方式 | 性能 |
| --- | --- | --- | --- |
| 偏向锁 | 只有一个线程访问 | Mark Word 记录线程 ID | 接近无锁 |
| 轻量级锁 | 多线程交替访问无竞争 | CAS 操作栈帧 Lock Record | 自旋等待 |
| 重量级锁 | 多线程同时竞争 | OS Mutex Lock + Monitor | 线程阻塞 |

synchronized 同步代码块通过 `monitorenter` / `monitorexit` 字节码指令实现；同步方法通过 `ACC_SYNCHRONIZED` 标志实现。

### 2.3 volatile 关键字

volatile 提供两大保证：可见性（写操作立即刷新到主内存，读操作从主内存读取）和禁止指令重排序（通过内存屏障实现）。但 volatile 不保证原子性，`i++` 是读-改-写三步操作，volatile 无法保证。

**DCL 单例必须用 volatile**：`new Singleton()` 分三步（分配内存→初始化对象→赋值引用），JIT 可能重排为分配内存→赋值引用→初始化对象，导致其他线程拿到未初始化的对象。

### 2.4 AQS（AbstractQueuedSynchronizer）

AQS 是 Java 并发包的核心框架，ReentrantLock、CountDownLatch、Semaphore、CyclicBarrier 等都基于它实现。

核心数据结构：`volatile int state`（同步状态，0=未锁，>0=已锁，可重入时累加）+ CLH 双向链表队列（存储等待线程的 Node）。

独占锁获取流程：CAS 修改 state 成功则获取锁；失败则将当前线程封装为 Node 加入 CLH 队列尾部；自旋检查前驱节点是否为 head，是则再次尝试获取；否则 `LockSupport.park()` 挂起线程。

**ReentrantLock vs synchronized**：

| 特性 | synchronized | ReentrantLock |
| --- | --- | --- |
| 实现层面 | JVM 内置 | JDK 代码层 |
| 锁释放 | 自动释放 | 必须手动 unlock |
| 可中断 | 不可中断 | lockInterruptibly 可中断 |
| 公平锁 | 不支持 | 支持公平/非公平 |
| 条件变量 | 单一 wait/notify | 多个 Condition |
| 性能 | JDK6 后相当 | 高竞争下略好 |

### 2.5 并发容器

**ConcurrentHashMap（JDK8）**：数组 + 链表 + 红黑树结构。并发控制：Node 数组用 volatile，put 时对桶头节点 synchronized，扩容时 CAS。`size()` 使用 `CounterCell` 数组分段计数。不允许 key/value 为 null（多线程下 null 含义不明确）。

JDK7 使用 Segment 分段锁（16个），JDK8 取消 Segment，锁粒度细化到桶级别，并发度更高。

**其他并发容器**：CopyOnWriteArrayList（写时复制，读多写少）、LinkedBlockingQueue（生产者-消费者）、DelayQueue（延迟任务）。

### 2.6 线程池

**ThreadPoolExecutor 七个核心参数**：corePoolSize（核心线程数）、maximumPoolSize（最大线程数）、keepAliveTime（非核心线程空闲存活时间）、unit（时间单位）、workQueue（任务队列）、threadFactory（线程工厂）、handler（拒绝策略）。

**任务提交流程**：线程数 < corePoolSize → 创建新线程；线程数 ≥ corePoolSize → 放入队列；队列满且线程数 < maximumPoolSize → 创建非核心线程；队列满且线程数 = maximumPoolSize → 执行拒绝策略。

**四种拒绝策略**：AbortPolicy（默认，抛异常）、CallerRunsPolicy（调用者线程执行，不丢任务）、DiscardPolicy（静默丢弃）、DiscardOldestPolicy（丢弃最老任务）。

**线程池大小**：CPU 密集型取 N+1，IO 密集型取 2N 或 N×(1+等待时间/计算时间)。

**不推荐 Executors 工厂方法**：newFixedThreadPool 和 newSingleThreadExecutor 使用无界 LinkedBlockingQueue 可能 OOM；newCachedThreadPool 线程数无界可能 OOM。

### 2.7 ThreadLocal

ThreadLocal 为每个线程提供独立的变量副本，底层通过 `Thread.threadLocals`（ThreadLocalMap）实现，key 为 ThreadLocal 的弱引用，value 为实际值。

**内存泄漏原因**：key 是弱引用会被 GC 回收，但 value 是强引用不会被回收，导致 Entry(null, value) 积累。解决方案：使用完后调用 `remove()`。

**InheritableThreadLocal**：子线程可以继承父线程的 ThreadLocal 值，但线程池场景下线程复用会导致值错乱，应使用 TransmittableThreadLocal（TTL）。

---

## 3. 集合框架

### 3.1 HashMap 深度

JDK8 数据结构：数组 + 链表 + 红黑树。初始容量 16，负载因子 0.75，扩容时容量翻倍。链表长度 ≥ 8 且数组长度 ≥ 64 时转红黑树，节点数 ≤ 6 时退化为链表。

`hash()` 方法：`(h = key.hashCode()) ^ (h >>> 16)`，高16位与低16位异或，让高位参与运算，减少碰撞。

**为什么容量必须是 2 的幂次**：取模运算可用位运算替代 `hash & (capacity-1)`，性能更高；扩容时只需判断高位是否为1，无需重新计算 hash。

**JDK7 多线程扩容死循环**：头插法导致链表反转，并发扩容可能形成环形链表。JDK8 改为尾插法解决，但仍非线程安全，并发场景应使用 ConcurrentHashMap。

### 3.2 常用集合对比

| 集合 | 底层结构 | 有序 | 线程安全 | 允许null |
| --- | --- | --- | --- | --- |
| ArrayList | 动态数组 | 插入序 | 否 | 是 |
| LinkedList | 双向链表 | 插入序 | 否 | 是 |
| HashMap | 数组+链表+红黑树 | 否 | 否 | key/value均可 |
| LinkedHashMap | HashMap+双向链表 | 插入序/访问序 | 否 | 是 |
| TreeMap | 红黑树 | key自然序 | 否 | key不可null |
| PriorityQueue | 小顶堆 | 优先级序 | 否 | 否 |

---

## 4. Java 新特性（JDK8-21）

### 4.1 JDK8 核心特性

Lambda 表达式是函数式接口的语法糖，本质通过 invokedynamic 指令实现，性能优于匿名内部类。

Stream API 采用惰性求值，中间操作（filter/map/sorted）不立即执行，终端操作（collect/forEach/count）触发整个流水线执行，支持并行流（parallelStream）利用 ForkJoinPool。

Optional 强制调用者处理 null 情况，避免 NPE。LocalDateTime 线程安全、不可变，替代 Date/Calendar。

### 4.2 JDK11-21 重要特性

| 版本 | 重要特性 |
| --- | --- |
| JDK11 | HTTP Client API、var 局部变量类型推断、ZGC、LTS |
| JDK14 | Switch 表达式正式版、有用的 NullPointerException |
| JDK16 | Records 正式版、instanceof 模式匹配 |
| JDK17 | Sealed Classes 正式版、LTS 版本 |
| JDK21 | 虚拟线程 Virtual Threads、结构化并发、LTS 版本 |

**虚拟线程（JDK21）**：轻量级线程，由 JVM 调度而非 OS，创建成本极低（可创建百万级），阻塞时不占用 OS 线程，适合 IO 密集型场景，大幅提升吞吐量。通过 `Thread.ofVirtual().start()` 或 `Executors.newVirtualThreadPerTaskExecutor()` 使用。

**Record（JDK16）**：不可变数据类，自动生成构造器、getter、equals、hashCode、toString，适合 DTO/值对象场景。

**Sealed Classes（JDK17）**：限制类的继承，配合 pattern matching 实现类型安全的代数数据类型。