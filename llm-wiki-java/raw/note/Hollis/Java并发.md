# Java并发

所有者: junk01

### ✅线程池顺序执行

1、单线程线程池：SingleThreadExecutor

2、依赖关系的任务调度：ScheduledThreadPoolExecutor结合ScheduledFuture.get() 等待前一个任务完成

---

### ✅多线程上下文

**上下文切换:** CPU一个线程转到另一个线程时，需要保存当前线程的上下文状态(程序计数器、寄存器、栈指针等) 恢复另一个线程的上下文状态。上下文切换的开销比直接用单线程大，过多的上下文切换会降低系统的运行效率，频繁的上下文切换会导致CPU时间的浪费。

**避免频繁上下文切换**

1. 减少线程数：线程池管理来减少线程的创建和销毁。
2. 无锁并发编程：无锁并发编程可以避免线程因等待锁而进入阻塞状态，
3. 使用CAS算法：CAS算法可以避免线程的阻塞和唤醒操作。
4. 使用协程（JDK19线程）：协程是一种用户态线程，其切换不需要操作系统的参与（仍需JVM层面保存和恢复线程的状态，成本低得多）
5. 合理地使用锁：避免使用同步块或同步方法，缩小同步块或同步方法的范围。

---

### ✅并发&并行

**并发**: 单CPU中，几个程序同进程中运行中。操作系统单CPU同时间是只能干一件事儿的，分时操作系统是把CPU的时间划分成长短基本相同的时间区间,即”时间片”，通过操作系统的管理，把这些时间片依次轮流地分配给各个用户使用。某个作业在时间片结束之前,整个任务还没有完成，那么该作业就被暂停下来,放弃CPU，等待下一轮循环再继续做.此时CPU又分配给另一个作业去使用。计算机的处理速度很快，只要时间片的间隔取得适当,那么一个用户作业从用完分配给它的一个时间片到获得下一个CPU时间片，中间有所”停顿”，但用户察觉不出来,好像整个系统全由它”独占”。是通过CPU时间片技术并发完成。

**并行：**多CPU中，一个CPU执行一个进程时，另一个CPU可以执行另一个进程，两个进程互不抢占CPU资源，可以同时进行，这种方式我们称之为并行(Parallel)。

---

### ✅线程6种状态

1.初始(NEW)：新创建了一个线程对象，但还没有调用start()方法。

2.运行(RUNNABLE)：Java线程中将就绪（READY）和运行中（RUNNING）两种状态笼统的称为“运行”。

- 就绪（READY）:线程对象创建后，调用start()方法。该状态的线程位于可运行线程池中，等待分配cpu使用权 。
- 运行中（RUNNING）：就绪(READY)线程获得了cpu 时间片，开始执行程序代码。

3.阻塞(BLOCKED)：表示线程阻塞于锁。

4.等待(WAITING)：进入该状态(调用wait())的线程需要等待其他线程做出一些特定动作（通知或中断）,会让出CPU。wait方法是会释放锁的

5.超时等待(TIMED_WAITING)：超时等待状态(sleep()),该状态不同于WAITING，它可以在指定的时间后自行返回,会让出CPU。sleep方法不会释放对象上的锁

6.终止(TERMINATED)：表示该线程已经执行完毕。

因为Java锁的目标是对象，所以wait、notify和notifyAll针对的目标都是Object类中对象，而sleep不需要释放锁，所以他是Thread类中的一个方法。 

**线程没有RUNNING状态**

并发执行CPU的执行分成很多个小的时间片,10-20毫秒左右,一秒钟之内，同一个线程可能一部分时间处于READY状态、一部分时间处于RUNNING状态,就是这个状态其实是不准的。

**sleep和wait区别**

sleep()方法可以在任何地方使用；而**wait()方法则只能在同步方法或同步块中使用**。**wait 方法会释放对象锁**，但 sleep 方法不会。

**wait的线程会进入到WAITING状态，直到被唤醒**；sleep的线程会进入到TIMED_WAITING状态，等到指定时间之后会再尝试获取CPU时间片。

因为Java锁的目标是对象，而wait需要释放锁，所以针对的目标都是对象，所以把他定义在Object类中。 而sleep()不需要释放锁，所以他针对的目标是线程，所以定义在Thread类中。 

**notify和notifyAll区别**

当一个线程进入wait之后，就必须等其他线程notify()/notifyAll()，才会从等待队列中被移出。

**使用notifyAll，可以唤醒所有处于wait状态的线程，使其重新进入锁的争夺队列中，而notify只能唤醒一个。**

---

### ✅守护线程

**User Thread(用户线程):** 用户线程一般用于执行用户级任务

**Daemon Thread(守护线程) :**守护线程是“后台线程”setDaemon(true)，用来执行后台任务，守护线程最典型的应用就是GC(垃圾回收器)。

这两种线程其实是没有什么区别的，唯一的区别就是Java虚拟机在所有<用户线程>都结束后就会退出，而不会等<守护线程>执行完。

---

### ✅创建线程4种

- 继承**Thread**类创建线程
- 实现**Runnable**接口创建线程
- 通过**Callable和FutureTask**创建线程
- 通过**线程池**创建线程

**Runnable和Callable区别**

**Runnable的run方法无返回值，Callable的call方法有返回值，类型为Object**

**Future**

Future是一个接口，代表了一个异步执行的结果。接口中的方法用来检查执行是否完成、等待完成和得到执行的结果。当执行完成后，只能通过get()方法得到结果，get方法会阻塞直到结果准备好了。如果想取消，那么调用cancel()方法。FutureTask是Future接口的一个实现，它实现了一个可以提交给Executor执行的任务，并且可以用来检查任务的执行状态和获取任务的执行结果。

---

### ✅实现高效的异步编程

在 Java 中实现高效的异步编程通常依赖于 **异步执行模型**，例如通过 `**CompletableFuture**`、`**ExecutorService**`、`**Future**` 等方式，来处理异步任务。

同时，避免**回调地狱**是实现异步编程时的一大挑战，特别是在多个异步操作需要顺序执行并且它们之间可能会相互依赖的情况下。  

回调地狱(callback hell)是指在使用传统回调方法时，如果有多个依赖关系的异步任务，代码会变得难以阅读和维护。  

`ExecutorService` 提供的一个高效的线程池管理工具，它支持异步任务的提交和执行。通过 `submit()` 方法提交任务，返回 `Future` 对象可以用于获取任务的执行结果

`CompletableFuture` 支持异步计算和组合多个异步任务。`CompletableFuture` 可以异步执行任务，可以通过链式调用的方式进行任务的组合与控制。

**异步的组合与并行执行**

`CompletableFuture` 支持多个异步任务的并行执行， `thenCombine()`、`allOf()` 、 `anyOf()` 等方法，可以组合多个任务的结果，避免回调地狱。

- `**thenCombine()**`：并行执行两个任务，合并它们的结果。
- `**allOf()**`：等待所有异步任务完成，然后执行后续操作。
- `**anyOf()**`：等待第一个异步任务完成，执行后续操作。

**处理异常与超时**

`CompletableFuture` 使用 `exceptionally()` 或 `handle()` 方法来定义异常处理的逻辑。

**避免回调地狱**

`CompletableFuture` 的 `thenApply`、`thenAccept`、`thenCombine` 等方法，可以避免层层嵌套的回调结构。每个异步操作都返回一个新`CompletableFuture`，让后续的操作能够依次执行。

#### 处理任务的结果和异常

`whenComplete` 和 `handle` 方法允许你在任务完成时进行统一的处理，包括处理正常结果或异常。这样可以避免在每个回调中重复处理结果和异常。

#### 使用 `ExecutorService` 配合 `CompletableFuture` 进行并发执行

通过 `ExecutorService` 提供的线程池和 `CompletableFuture`，可以有效地管理多个并发的异步任务，避免回调地狱并且能够进行异步任务的组合。

---

### ✅new Thread().start() 操作系统层面

new Thread().start()这个方法创建并启动线程的话，用的是**平台线程**，Java早期（JDK 21之前）线程的是实现是靠操作系统内核线程来实现的，**并且Java线程和OS内核线程之间是一对一的关系。**

操作系统层面，`**new Thread()**`**主要是进行OS线程的创建**，分配内核线程主要分以下四步：

- 内核为新线程创建线程控制块（TCB），包含：
- 线程 ID（TID）
- 寄存器状态（PC、SP 等）
- 调度优先级
- 所属进程（共享地址空间、文件描述符等）

分配栈空间

- 用户态栈：默认大小由 JVM 控制（可通过 `-Xss` 设置，如 `-Xss1m`）。
- 例如：Linux 上默认可能是 1MB（但实际是虚拟内存，按需分配物理页）。
- 内核栈：每个线程在内核中也有自己的栈（通常 8–16KB），用于系统调用。

设置初始上下文

- 程序计数器（PC）指向 `thread_entry_point`（JVM 的 C 入口函数）。
- 栈指针（SP）指向新分配的用户栈顶部。

加入调度器就绪队列

- 线程状态变为 RUNNABLE（可运行）。
- 由操作系统的调度器（如 Linux 的 CFS）决定何时在 CPU 上执行。

而`**.start**`**方法的调用，则是线程的真正执行了**：

- 新线程拿到CPU时间片
- 执行 `thread_entry_point` → 初始化 JNI → 调用 `Thread.run()` → 最终执行你传入的 `Runnable.run()`。

---

### ✅什么是伪共享，如何解决伪共享？

伪共享（False Sharing）是并发编程中一种性能问题，发生在多线程访问位于同一缓存行（cache line）中的不同变量时，导致缓存频繁失效和同步开销增大，从而严重影响程序性能。

我现代 CPU 使用缓存来弥补处理器与主内存之间的巨大速度差异。数据在主内存和 CPU 缓存（L1、L2、L3）之间传输的最小单位称为**缓存行**，一般是64字节。

为了保证多核 CPU 看到的内存视图是一致的，处理器使用如 MESI协议来协调各个核心的缓存状态。当一个核心修改了其缓存行中的数据时，该缓存行在其他核心中的副本会被标记为无效。其他核心后续访问该缓存行时需要重新从修改核心的缓存或主内存中加载最新数据。

那么，当多个线程在不同的 CPU 核心上运行时，它们就可能访问物理上相邻（位于同一个缓存行内）但逻辑上独立（被不同线程修改）的变量。

如果一个线程（线程1）修改了它自己关心的变量（变量 X），而这个变量恰好与另一个线程（线程2）关心的变量（变量 Y）位于**同一个缓存行**中。

由于缓存一致性协议是基于缓存行操作的，线程 A 对 X 的修改会导致**整个缓存行**在其他核心的缓存中失效。

当线程 B 需要访问它自己的变量 Y（即使 Y 本身没有被线程 A 修改）时，因为它所在的整个缓存行都失效了，线程 B 的 CPU 核心必须：

1. 从持有最新数据（可能是线程 A 的缓存或主内存）的源重新加载整个缓存行。
2. 这会导致一次缓存未命中，迫使线程 B 等待数据加载完成。

同样，线程 B 修改 Y 也会导致线程 A 的缓存行失效。

**线程 1 和 线程2 在逻辑上操作的是完全不同的变量（X 和 Y），仅仅因为它们物理上靠得太近（在同一个缓存行内），就导致了彼此缓存行的频繁失效和重新加载。这种无效的共享就是“伪共享”**。

伪共享的会带来很多问题：

- 性能下降： 频繁的缓存失效和重新加载导致 CPU 大量时间花在等待数据上。
- 可伸缩性差： 随着 CPU 核心数量的增加，伪共享导致的缓存一致性开销会成倍增长，程序无法有效利用多核资源。
- 难以诊断： 性能瓶颈的根源（物理内存布局）与代码逻辑（变量访问）分离，使用常规的性能分析工具（如 profiler）很难直接定位到伪共享问题。

**解决伪共享，核心的方案就是想办法让不同线程频繁访问的独立变量位于不同的缓存行中。** 主要有以下几种方法：

**填充：**在关键变量周围添加无用的“填充”字节，确保每个关键变量独占一个缓存行。在Java 8+中，使用`Contended` 注解，就可以使得被标注的字段单独占用一个缓存行。如下面的x。

这个方案的优点是简单，直接，缺点很容易理解，以为有很多填充导致的内存浪费。

**线程局部存储**

如果变量完全不需要在线程间共享，只是每个线程自己使用，那么可以使用ThreadLocal。这样每个线程都有自己独立的副本，物理上自然就不会共享缓存行了。

**使用支持并发数据结构**

高级并发库在设计数据结构（如无锁队列、并发哈希表）时，内部已经采用了填充、对齐等技术来避免伪共享（例如 Java 的 `java.util.concurrent.atomic` 包中的 `Striped64` 及其子类 `LongAdder`）。

- ✅如何让Java的线程池顺序执行任务？
- ✅什么是多线程中的上下文切换？
- ✅能不能谈谈你对线程安全的理解？
- ✅什么是并发，什么是并行？
- ✅线程有几种状态，状态之间的流转是怎样的？
- ✅什么是守护线程，和普通线程有什么区别？
- ✅JDK21 中的虚拟线程是怎么回事？
- ✅创建线程有几种方式？
- ✅run/start、wait/sleep、notify/notifyAll区别?
- ✅什么是线程池，如何实现的？
- ✅线程数设定成多少更合适？
- ✅什么是ThreadLocal，如何实现的？
- ✅线程同步的方式有哪些？
- ✅什么是死锁，如何解决？
- ✅什么是Java内存模型（JMM）？
- ✅并发编程中的原子性和数据库ACID的原子性一样吗？
- ✅synchronized是怎么实现的？
- ✅synchronized是如何保证原子性、可见性、有序性的？
- ✅synchronized的锁升级过程是怎样的？
- ✅synchronized的重量级锁很慢，为什么还需要重量级锁？
- ✅synchronized的锁优化是怎样的？
- ✅volatile能保证原子性吗？为什么？
- ✅int a = 1 是原子性操作吗
- ✅volatile是如何保证可见性和有序性的？
- ✅有了synchronized为什么还需要volatile?
- ✅如何理解AQS？
- ✅什么是CAS？存在什么问题？
- ✅CAS一定有自旋吗？
- ✅什么是Unsafe？
- ✅CAS在操作系统层面是如何保证原子性的？
- ✅synchronized和reentrantLock区别？
- ✅公平锁和非公平锁的区别？
- ✅LongAdder和AtomicLong的区别？
- ✅CountDownLatch、CyclicBarrier、Semaphore区别？
- ✅父子线程之间怎么共享/传递数据？
- ✅有三个线程T1,T2,T3如何保证顺序执行？
- ✅如何对多线程进行编排
- ✅三个线程分别顺序打印0-100
- ✅什么是总线嗅探和总线风暴，和JMM有什么关系？
- ✅CompletableFuture的底层是如何实现的？
- ✅ForkJoinPool和ThreadPoolExecutor区别是什么？
- ✅有了InheritableThreadLocal为啥还需要TransmittableThreadLocal？
- ✅AQS是如何实现线程的等待和唤醒的？
- ✅如何保证多线程下 i++ 结果正确？
- ✅Thread.sleep(0)的作用是什么？
- ✅有哪些实现线程安全的方案?
- ✅synchronized锁的是什么？
- ✅为什么不建议通过Executors构建线程池
- ✅线程池的拒绝策略有哪些？
- ✅线程是如何被调度的？
- ✅为什么JDK 15要废弃偏向锁？
- ✅Java是如何判断一个线程是否存活的？
- ✅什么是可重入锁，怎么实现可重入锁？
- ✅如何实现主线程捕获子线程异常
- ✅为什么不能在try-catch中捕获子线程的异常?
- ✅Java线程出现异常，进程为啥不会退出？
- ✅什么是happens-before原则？
- ✅happens-before和as-if-serial有啥区别和联系？
- ✅ThreadLocal的应用场景有哪些？
- ✅ThreadLocal为什么会导致内存泄漏？如何解决的？
- ✅到底啥是内存屏障？到底怎么加的？
- ✅有了CAS为啥还需要volatile？
- ✅AQS的同步队列和条件队列原理？
- ✅什么是AQS的独占模式和共享模式？
- ✅AQS为什么采用双向链表？
- ✅有了MESI为啥还需要JMM？
- ✅为什么虚拟线程不能用synchronized？
- ✅为什么虚拟线程不要和线程池一起用？
- ✅为什么虚拟线程尽量避免使用ThreadLocal
- ✅sychronized是非公平锁吗，那么是如何体现的？
- ✅synchronized 的锁能降级吗？
- ✅如何实现无锁化编程？
- ✅如何在 Java 中实现高效的异步编程？如何避免回调地狱？
- ✅线程池中使用ThreadLocal会有哪些潜在风险？
- ✅动态线程池的原理是什么？
- ✅什么是伪共享，如何解决伪共享？
- ✅介绍下JUC，都有哪些工具类？
- ✅什么是活锁，和死锁有什么区别？
- ✅Java中有哪些锁?
- ✅JDK25的ScopedValue是什么？为什么可以替代ThreadLocal？
- ✅线程池有哪些核心参数？
- ✅new Thread().start() 创建一个线程时，操作系统层面发生了什么？

[✅能不能谈谈你对线程安全的理解？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E8%83%BD%E4%B8%8D%E8%83%BD%E8%B0%88%E8%B0%88%E4%BD%A0%E5%AF%B9%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E7%9A%84%E7%90%86%E8%A7%A3%EF%BC%9F.md)

[✅JDK21 中的虚拟线程是怎么回事？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85JDK21%20%E4%B8%AD%E7%9A%84%E8%99%9A%E6%8B%9F%E7%BA%BF%E7%A8%8B%E6%98%AF%E6%80%8E%E4%B9%88%E5%9B%9E%E4%BA%8B%EF%BC%9F.md)

[✅什么是线程池，如何实现的？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AF%E7%BA%BF%E7%A8%8B%E6%B1%A0%EF%BC%8C%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E7%9A%84%EF%BC%9F.md)

[✅线程数设定成多少更合适？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E7%BA%BF%E7%A8%8B%E6%95%B0%E8%AE%BE%E5%AE%9A%E6%88%90%E5%A4%9A%E5%B0%91%E6%9B%B4%E5%90%88%E9%80%82%EF%BC%9F.md)

[✅什么是ThreadLocal，如何实现的？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AFThreadLocal%EF%BC%8C%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E7%9A%84%EF%BC%9F.md)

[✅线程同步的方式有哪些？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E7%9A%84%E6%96%B9%E5%BC%8F%E6%9C%89%E5%93%AA%E4%BA%9B%EF%BC%9F.md)

[✅什么是死锁，如何解决？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AF%E6%AD%BB%E9%94%81%EF%BC%8C%E5%A6%82%E4%BD%95%E8%A7%A3%E5%86%B3%EF%BC%9F.md)

[✅什么是Java内存模型（JMM）？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AFJava%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B%EF%BC%88JMM%EF%BC%89%EF%BC%9F.md)

[✅并发编程中的原子性和数据库ACID的原子性一样吗？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E4%B8%AD%E7%9A%84%E5%8E%9F%E5%AD%90%E6%80%A7%E5%92%8C%E6%95%B0%E6%8D%AE%E5%BA%93ACID%E7%9A%84%E5%8E%9F%E5%AD%90%E6%80%A7%E4%B8%80%E6%A0%B7%E5%90%97%EF%BC%9F.md)

[✅synchronized是怎么实现的？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85synchronized%E6%98%AF%E6%80%8E%E4%B9%88%E5%AE%9E%E7%8E%B0%E7%9A%84%EF%BC%9F.md)

[✅synchronized是如何保证原子性、可见性、有序性的？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85synchronized%E6%98%AF%E5%A6%82%E4%BD%95%E4%BF%9D%E8%AF%81%E5%8E%9F%E5%AD%90%E6%80%A7%E3%80%81%E5%8F%AF%E8%A7%81%E6%80%A7%E3%80%81%E6%9C%89%E5%BA%8F%E6%80%A7%E7%9A%84%EF%BC%9F.md)

[✅synchronized的锁升级过程是怎样的？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85synchronized%E7%9A%84%E9%94%81%E5%8D%87%E7%BA%A7%E8%BF%87%E7%A8%8B%E6%98%AF%E6%80%8E%E6%A0%B7%E7%9A%84%EF%BC%9F.md)

[✅synchronized的重量级锁很慢，为什么还需要重量级锁？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85synchronized%E7%9A%84%E9%87%8D%E9%87%8F%E7%BA%A7%E9%94%81%E5%BE%88%E6%85%A2%EF%BC%8C%E4%B8%BA%E4%BB%80%E4%B9%88%E8%BF%98%E9%9C%80%E8%A6%81%E9%87%8D%E9%87%8F%E7%BA%A7%E9%94%81%EF%BC%9F.md)

[✅synchronized的锁优化是怎样的？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85synchronized%E7%9A%84%E9%94%81%E4%BC%98%E5%8C%96%E6%98%AF%E6%80%8E%E6%A0%B7%E7%9A%84%EF%BC%9F.md)

[✅volatile能保证原子性吗？为什么？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85volatile%E8%83%BD%E4%BF%9D%E8%AF%81%E5%8E%9F%E5%AD%90%E6%80%A7%E5%90%97%EF%BC%9F%E4%B8%BA%E4%BB%80%E4%B9%88%EF%BC%9F.md)

[✅int a = 1 是原子性操作吗](Java%E5%B9%B6%E5%8F%91/%E2%9C%85int%20a%20=%201%20%E6%98%AF%E5%8E%9F%E5%AD%90%E6%80%A7%E6%93%8D%E4%BD%9C%E5%90%97.md)

[✅volatile是如何保证可见性和有序性的？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85volatile%E6%98%AF%E5%A6%82%E4%BD%95%E4%BF%9D%E8%AF%81%E5%8F%AF%E8%A7%81%E6%80%A7%E5%92%8C%E6%9C%89%E5%BA%8F%E6%80%A7%E7%9A%84%EF%BC%9F.md)

[✅有了synchronized为什么还需要volatile?](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E6%9C%89%E4%BA%86synchronized%E4%B8%BA%E4%BB%80%E4%B9%88%E8%BF%98%E9%9C%80%E8%A6%81volatile.md)

[✅如何理解AQS？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E5%A6%82%E4%BD%95%E7%90%86%E8%A7%A3AQS%EF%BC%9F.md)

[✅什么是CAS？存在什么问题？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AFCAS%EF%BC%9F%E5%AD%98%E5%9C%A8%E4%BB%80%E4%B9%88%E9%97%AE%E9%A2%98%EF%BC%9F.md)

[✅CAS一定有自旋吗？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85CAS%E4%B8%80%E5%AE%9A%E6%9C%89%E8%87%AA%E6%97%8B%E5%90%97%EF%BC%9F.md)

[✅什么是Unsafe？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AFUnsafe%EF%BC%9F.md)

[✅CAS在操作系统层面是如何保证原子性的？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85CAS%E5%9C%A8%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%B1%82%E9%9D%A2%E6%98%AF%E5%A6%82%E4%BD%95%E4%BF%9D%E8%AF%81%E5%8E%9F%E5%AD%90%E6%80%A7%E7%9A%84%EF%BC%9F.md)

[✅synchronized和reentrantLock区别？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85synchronized%E5%92%8CreentrantLock%E5%8C%BA%E5%88%AB%EF%BC%9F.md)

[✅公平锁和非公平锁的区别？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E5%85%AC%E5%B9%B3%E9%94%81%E5%92%8C%E9%9D%9E%E5%85%AC%E5%B9%B3%E9%94%81%E7%9A%84%E5%8C%BA%E5%88%AB%EF%BC%9F.md)

[✅LongAdder和AtomicLong的区别？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85LongAdder%E5%92%8CAtomicLong%E7%9A%84%E5%8C%BA%E5%88%AB%EF%BC%9F.md)

[✅CountDownLatch、CyclicBarrier、Semaphore区别？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85CountDownLatch%E3%80%81CyclicBarrier%E3%80%81Semaphore%E5%8C%BA%E5%88%AB%EF%BC%9F.md)

[✅父子线程之间怎么共享/传递数据？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E7%88%B6%E5%AD%90%E7%BA%BF%E7%A8%8B%E4%B9%8B%E9%97%B4%E6%80%8E%E4%B9%88%E5%85%B1%E4%BA%AB%20%E4%BC%A0%E9%80%92%E6%95%B0%E6%8D%AE%EF%BC%9F.md)

[✅有三个线程T1,T2,T3如何保证顺序执行？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E6%9C%89%E4%B8%89%E4%B8%AA%E7%BA%BF%E7%A8%8BT1,T2,T3%E5%A6%82%E4%BD%95%E4%BF%9D%E8%AF%81%E9%A1%BA%E5%BA%8F%E6%89%A7%E8%A1%8C%EF%BC%9F.md)

[✅如何对多线程进行编排](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E5%A6%82%E4%BD%95%E5%AF%B9%E5%A4%9A%E7%BA%BF%E7%A8%8B%E8%BF%9B%E8%A1%8C%E7%BC%96%E6%8E%92.md)

[✅三个线程分别顺序打印0-100](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%B8%89%E4%B8%AA%E7%BA%BF%E7%A8%8B%E5%88%86%E5%88%AB%E9%A1%BA%E5%BA%8F%E6%89%93%E5%8D%B00-100.md)

[✅什么是总线嗅探和总线风暴，和JMM有什么关系？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AF%E6%80%BB%E7%BA%BF%E5%97%85%E6%8E%A2%E5%92%8C%E6%80%BB%E7%BA%BF%E9%A3%8E%E6%9A%B4%EF%BC%8C%E5%92%8CJMM%E6%9C%89%E4%BB%80%E4%B9%88%E5%85%B3%E7%B3%BB%EF%BC%9F.md)

[✅CompletableFuture的底层是如何实现的？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85CompletableFuture%E7%9A%84%E5%BA%95%E5%B1%82%E6%98%AF%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E7%9A%84%EF%BC%9F.md)

[✅ForkJoinPool和ThreadPoolExecutor区别是什么？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85ForkJoinPool%E5%92%8CThreadPoolExecutor%E5%8C%BA%E5%88%AB%E6%98%AF%E4%BB%80%E4%B9%88%EF%BC%9F.md)

[✅有了InheritableThreadLocal为啥还需要TransmittableThreadLocal？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E6%9C%89%E4%BA%86InheritableThreadLocal%E4%B8%BA%E5%95%A5%E8%BF%98%E9%9C%80%E8%A6%81TransmittableThreadL.md)

[✅AQS是如何实现线程的等待和唤醒的？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85AQS%E6%98%AF%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E7%BA%BF%E7%A8%8B%E7%9A%84%E7%AD%89%E5%BE%85%E5%92%8C%E5%94%A4%E9%86%92%E7%9A%84%EF%BC%9F.md)

[✅如何保证多线程下 i++ 结果正确？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E5%A6%82%E4%BD%95%E4%BF%9D%E8%AF%81%E5%A4%9A%E7%BA%BF%E7%A8%8B%E4%B8%8B%20i++%20%E7%BB%93%E6%9E%9C%E6%AD%A3%E7%A1%AE%EF%BC%9F.md)

[✅Thread.sleep(0)的作用是什么？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85Thread%20sleep(0)%E7%9A%84%E4%BD%9C%E7%94%A8%E6%98%AF%E4%BB%80%E4%B9%88%EF%BC%9F.md)

[✅有哪些实现线程安全的方案?](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E6%9C%89%E5%93%AA%E4%BA%9B%E5%AE%9E%E7%8E%B0%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E7%9A%84%E6%96%B9%E6%A1%88.md)

[✅synchronized锁的是什么？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85synchronized%E9%94%81%E7%9A%84%E6%98%AF%E4%BB%80%E4%B9%88%EF%BC%9F.md)

[✅为什么不建议通过Executors构建线程池](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E5%BB%BA%E8%AE%AE%E9%80%9A%E8%BF%87Executors%E6%9E%84%E5%BB%BA%E7%BA%BF%E7%A8%8B%E6%B1%A0.md)

[✅线程池的拒绝策略有哪些？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%9A%84%E6%8B%92%E7%BB%9D%E7%AD%96%E7%95%A5%E6%9C%89%E5%93%AA%E4%BA%9B%EF%BC%9F.md)

[✅线程是如何被调度的？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E7%BA%BF%E7%A8%8B%E6%98%AF%E5%A6%82%E4%BD%95%E8%A2%AB%E8%B0%83%E5%BA%A6%E7%9A%84%EF%BC%9F.md)

[✅为什么JDK 15要废弃偏向锁？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%B8%BA%E4%BB%80%E4%B9%88JDK%2015%E8%A6%81%E5%BA%9F%E5%BC%83%E5%81%8F%E5%90%91%E9%94%81%EF%BC%9F.md)

[✅Java是如何判断一个线程是否存活的？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85Java%E6%98%AF%E5%A6%82%E4%BD%95%E5%88%A4%E6%96%AD%E4%B8%80%E4%B8%AA%E7%BA%BF%E7%A8%8B%E6%98%AF%E5%90%A6%E5%AD%98%E6%B4%BB%E7%9A%84%EF%BC%9F.md)

[✅什么是可重入锁，怎么实现可重入锁？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AF%E5%8F%AF%E9%87%8D%E5%85%A5%E9%94%81%EF%BC%8C%E6%80%8E%E4%B9%88%E5%AE%9E%E7%8E%B0%E5%8F%AF%E9%87%8D%E5%85%A5%E9%94%81%EF%BC%9F.md)

[✅如何实现主线程捕获子线程异常](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E4%B8%BB%E7%BA%BF%E7%A8%8B%E6%8D%95%E8%8E%B7%E5%AD%90%E7%BA%BF%E7%A8%8B%E5%BC%82%E5%B8%B8.md)

[✅为什么不能在try-catch中捕获子线程的异常?](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%8D%E8%83%BD%E5%9C%A8try-catch%E4%B8%AD%E6%8D%95%E8%8E%B7%E5%AD%90%E7%BA%BF%E7%A8%8B%E7%9A%84%E5%BC%82%E5%B8%B8.md)

[✅Java线程出现异常，进程为啥不会退出？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85Java%E7%BA%BF%E7%A8%8B%E5%87%BA%E7%8E%B0%E5%BC%82%E5%B8%B8%EF%BC%8C%E8%BF%9B%E7%A8%8B%E4%B8%BA%E5%95%A5%E4%B8%8D%E4%BC%9A%E9%80%80%E5%87%BA%EF%BC%9F.md)

[✅什么是happens-before原则？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AFhappens-before%E5%8E%9F%E5%88%99%EF%BC%9F.md)

[✅happens-before和as-if-serial有啥区别和联系？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85happens-before%E5%92%8Cas-if-serial%E6%9C%89%E5%95%A5%E5%8C%BA%E5%88%AB%E5%92%8C%E8%81%94%E7%B3%BB%EF%BC%9F.md)

[✅ThreadLocal的应用场景有哪些？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85ThreadLocal%E7%9A%84%E5%BA%94%E7%94%A8%E5%9C%BA%E6%99%AF%E6%9C%89%E5%93%AA%E4%BA%9B%EF%BC%9F.md)

[✅ThreadLocal为什么会导致内存泄漏？如何解决的？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85ThreadLocal%E4%B8%BA%E4%BB%80%E4%B9%88%E4%BC%9A%E5%AF%BC%E8%87%B4%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%EF%BC%9F%E5%A6%82%E4%BD%95%E8%A7%A3%E5%86%B3%E7%9A%84%EF%BC%9F.md)

[✅到底啥是内存屏障？到底怎么加的？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E5%88%B0%E5%BA%95%E5%95%A5%E6%98%AF%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C%EF%BC%9F%E5%88%B0%E5%BA%95%E6%80%8E%E4%B9%88%E5%8A%A0%E7%9A%84%EF%BC%9F.md)

[✅AQS的同步队列和条件队列原理？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85AQS%E7%9A%84%E5%90%8C%E6%AD%A5%E9%98%9F%E5%88%97%E5%92%8C%E6%9D%A1%E4%BB%B6%E9%98%9F%E5%88%97%E5%8E%9F%E7%90%86%EF%BC%9F.md)

[✅什么是AQS的独占模式和共享模式？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AFAQS%E7%9A%84%E7%8B%AC%E5%8D%A0%E6%A8%A1%E5%BC%8F%E5%92%8C%E5%85%B1%E4%BA%AB%E6%A8%A1%E5%BC%8F%EF%BC%9F.md)

[✅AQS为什么采用双向链表？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85AQS%E4%B8%BA%E4%BB%80%E4%B9%88%E9%87%87%E7%94%A8%E5%8F%8C%E5%90%91%E9%93%BE%E8%A1%A8%EF%BC%9F.md)

[✅有了MESI为啥还需要JMM？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E6%9C%89%E4%BA%86MESI%E4%B8%BA%E5%95%A5%E8%BF%98%E9%9C%80%E8%A6%81JMM%EF%BC%9F.md)

[✅为什么虚拟线程不能用synchronized？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%B8%BA%E4%BB%80%E4%B9%88%E8%99%9A%E6%8B%9F%E7%BA%BF%E7%A8%8B%E4%B8%8D%E8%83%BD%E7%94%A8synchronized%EF%BC%9F.md)

[✅为什么虚拟线程不要和线程池一起用？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%B8%BA%E4%BB%80%E4%B9%88%E8%99%9A%E6%8B%9F%E7%BA%BF%E7%A8%8B%E4%B8%8D%E8%A6%81%E5%92%8C%E7%BA%BF%E7%A8%8B%E6%B1%A0%E4%B8%80%E8%B5%B7%E7%94%A8%EF%BC%9F.md)

[✅为什么虚拟线程尽量避免使用ThreadLocal](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%B8%BA%E4%BB%80%E4%B9%88%E8%99%9A%E6%8B%9F%E7%BA%BF%E7%A8%8B%E5%B0%BD%E9%87%8F%E9%81%BF%E5%85%8D%E4%BD%BF%E7%94%A8ThreadLocal.md)

[✅sychronized是非公平锁吗，那么是如何体现的？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85sychronized%E6%98%AF%E9%9D%9E%E5%85%AC%E5%B9%B3%E9%94%81%E5%90%97%EF%BC%8C%E9%82%A3%E4%B9%88%E6%98%AF%E5%A6%82%E4%BD%95%E4%BD%93%E7%8E%B0%E7%9A%84%EF%BC%9F.md)

[✅synchronized 的锁能降级吗？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85synchronized%20%E7%9A%84%E9%94%81%E8%83%BD%E9%99%8D%E7%BA%A7%E5%90%97%EF%BC%9F.md)

[✅线程池中使用ThreadLocal会有哪些潜在风险？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E7%BA%BF%E7%A8%8B%E6%B1%A0%E4%B8%AD%E4%BD%BF%E7%94%A8ThreadLocal%E4%BC%9A%E6%9C%89%E5%93%AA%E4%BA%9B%E6%BD%9C%E5%9C%A8%E9%A3%8E%E9%99%A9%EF%BC%9F.md)

[✅动态线程池的原理是什么？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E5%8A%A8%E6%80%81%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%9A%84%E5%8E%9F%E7%90%86%E6%98%AF%E4%BB%80%E4%B9%88%EF%BC%9F.md)

[✅介绍下JUC，都有哪些工具类？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%BB%8B%E7%BB%8D%E4%B8%8BJUC%EF%BC%8C%E9%83%BD%E6%9C%89%E5%93%AA%E4%BA%9B%E5%B7%A5%E5%85%B7%E7%B1%BB%EF%BC%9F.md)

[✅什么是活锁，和死锁有什么区别？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AF%E6%B4%BB%E9%94%81%EF%BC%8C%E5%92%8C%E6%AD%BB%E9%94%81%E6%9C%89%E4%BB%80%E4%B9%88%E5%8C%BA%E5%88%AB%EF%BC%9F.md)

[✅Java中有哪些锁?](Java%E5%B9%B6%E5%8F%91/%E2%9C%85Java%E4%B8%AD%E6%9C%89%E5%93%AA%E4%BA%9B%E9%94%81.md)

[✅JDK25的ScopedValue是什么？为什么可以替代ThreadLocal？](Java%E5%B9%B6%E5%8F%91/%E2%9C%85JDK25%E7%9A%84ScopedValue%E6%98%AF%E4%BB%80%E4%B9%88%EF%BC%9F%E4%B8%BA%E4%BB%80%E4%B9%88%E5%8F%AF%E4%BB%A5%E6%9B%BF%E4%BB%A3ThreadLocal%EF%BC%9F.md)