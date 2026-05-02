# ✅LongAdder和AtomicLong的区别？

所有者: junk01

# ✅LongAdder和AtomicLong的区别？

> 原文：[https://www.yuque.com/hollis666/asgng6/dhzyrg](https://www.yuque.com/hollis666/asgng6/dhzyrg)
> 

# 典型回答

LongAdder是Java 8中推出了一个新的类，主要是为了解决AtomicLong在多线程竞争激烈的情况下性能并不高的问题。它主要是采用分段+CAS的方式来提升原子操作的性能。

相比于AtomicLong，LongAdder有更好的性能，但是LongAdder是典型的以空间换时间的实现方式，所以他所需要用到的空间更大， 而且LongAdder可能存在结果不准确的问题，而AtomicLong并不会。

# 扩展知识

## AtomicLong实现原理

AtomicLong和AtomicInteger、AtomicDouble等其他原子操作的类一样，都是基于Unsafe实现的。Unsafe是一个用来进行硬件级别的原子操作的工具类。在AtomicLong中定义如下：

并且通过在类中定义一个volatile的变量用来计数：

以下几个常用的方法具体实现如下：

可以看到，都是基于unsafe来实现的，其实就是调用了底层的CAS操作来进行原子操作的。

## LongAdder实现原理

JDK 1.8中的LongAdder继承自抽象类java.util.concurrent.atomic.Striped64的，这个类也是JDK 1.8中新增的，他主要就是用来在给并发场景中提供计数支持的。

Striped64的设计思路和ConcurrentHashMap类似，都是希望通过分散竞争的方式来提升并发的性能。再具体是线上，主要依赖了其中的以下两个字段：

通过查看LongAdder中的add方法的代码，我们其实就能很容易的理解他的实现细节：

首先就是先尝试通过CAS更新计数器base的值，如果在竞争不激烈的情况下，是可以直接更新成功的，如果直接成功那就和AtomicLong一样了。

但是如果更新失败，那么则是认为当前的并发竞争比较激烈，那么就会尝试通过cells数组来分散计数

Striped64根据线程来计算哈希，然后将不同的线程分散到不同的Cell数组的index上，然后这个线程的计数内容就会保存在该Cell的位置上面，基于这种设计，最后的总计数需要结合base以及散落在Cell数组中的计数内容。

是不是看上去和ConcurrentHashMap的分段锁很像。

当需要进行统计数量的时候，则需要结合base和cells一起做统计，其实就是把他们的值都加在一起。

### LongAdder为什么可能会不准确

通过查看LongAdder的代码了解了他的原理之后，很容易可以发现一个问题，那就是其实LongAdder的累加结果可能是不准的。没错，其实在JDK的源码中也明确的说了：

> 
> 

Returns the current sum. The returned value is NOT an atomic snapshot; invocation in the absence of concurrent updates returns an accurate result, but concurrent updates that occur while the sum is being calculated might not be incorporated.

以上，就是LongAdder的sum方法的说明，其实就是：在没有并发更新的情况下sum方法将返回准确的结果，但是在计算总和时发生的并发更新可能不会被计算进来，所以就有可能会不准确。

### DoubleAdder

除了LongAdder以外，JDK 1.8中还提供了针对浮点数进行原子操作的DoubleAdder，他也是基于Striped64实现的，原理和LongAdder并无差别。

## 适用场景

相比于AtomicLong，LongAdder性能更好，但是有一个小缺点就是有可能返回值没那么准确，但是也只是在并发极高，刚好返回sum时候有其他原子操作在进行累加的时候才会出现。

基于以上的特性，LongAdder比较适合于并发竞争激烈，但是对数据准确度要求并不是百分之百准确的场景，比如微博点赞、文章阅读量的统计等等场景中。

##