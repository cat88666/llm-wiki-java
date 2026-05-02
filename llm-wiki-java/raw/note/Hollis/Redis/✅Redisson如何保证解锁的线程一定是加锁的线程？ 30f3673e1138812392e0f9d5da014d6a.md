# ✅Redisson如何保证解锁的线程一定是加锁的线程？

所有者: junk01

# ✅Redisson如何保证解锁的线程一定是加锁的线程？

> 原文：[https://www.yuque.com/hollis666/asgng6/mtfd25g8imxnamo6](https://www.yuque.com/hollis666/asgng6/mtfd25g8imxnamo6)
> 

# 典型回答

在分布式锁的实现中，有一个比较重要的问题就是误解锁的问题，如何有效的避免A线程加的锁被B线程给解锁是非常重要的。

Redisson作为一个分布式锁用的比较多的框架，他是如何实现的这个功能呢？我们通过源码来深入展开介绍一下。

以下是 lock 方法，也就是加锁的入口，

这里调用了一个`lock(long leaseTime, TimeUnit unit, boolean interruptibly)` 方法，放方法的大致内容是：

我精简了一下，只保留了关键部分，也就是说，在方法中，获取了当前线程的threadId，然后把他传到了tryAcquire方法中进行加锁。

这个方法最终会调用到`tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command)`这个方法。

这个方法的第四个参数就是刚刚我们获取的线程 ID，然后这个方法的内容是啥呢？

这是一段 lua 脚本，这里使用线程 ID 调用getLockName方法得到了一个锁的name，然后把他传到 lua 脚本中进行执行。

在 lua 脚本中，可以简单的认为他把线程 ID 传了进去，然后脚本中的`redis.call('hincrby', KEYS[1], ARGV[2], 1);` 里面的`KEYS[1]`就是加锁的 key，而`ARGV[2]`就是我们的线程 id

hincrby 命令用于为哈希表中的字段值加上指定增量值语法如下：

> 
> 

HINCRBY KEY_NAME FIELD_NAME INCR_BY_NUMBER 

比如以下是我加锁后的一个 redis 中存储的信息：

可以看到，这里面的`376fe4aa-10ea-4e60-b8ed-a49082e92014:1`中的后半段"1"就是我的线程 ID，因为我在 debug，所以只有一个线程：

那么也就是说**Redisson 在加锁的时候，会把当前线程 ID 当作 hash结构中的 filed 进行存储下来。**

那么，解锁的时候，是如何校验这个线程 id 的呢？解锁的方法如下：

可以看到，这里同样获取了一个线程 ID，然后传给了unlockAsync方法，最终调用到unlockAsync0方法：

然后继续看关键方法`unlockInnerAsync(threadId);`，不出意外，还是个 lua 脚本：

这里同样是调用了getLockName(threadId)来获取一个 field 的名字，最终拿到的应该是和加锁时候一样的，因为线程 ID 相同，方法也是同一个，拿结果肯定也一样。

脚本中关键的是这句：

这里的KEYS[1]就是你加锁的 key，ARGV[3]就是我们传入的那个带着线程 ID 的 lockName，那这段代码其实是判断当前的 hash 结构中是否有一个 filed 为我们指定的线程 ID 一致的 k-v 对，如果没有，说明不是当前线程加的锁。那么就解锁失败。

如果有，则说明是当前线程加的锁，那么就可以进行解锁，执行后面的脚本就行了。

所以， 简单点说，就是 **Redisson 的 unlock 方法在解锁时，会去判断当前线程 ID 是否存在于redis 的加锁的 hash 结构中，如果有则认为可以解锁，如果没有，则无法解锁。**

总结一下，就是**加锁的时候把线程 id 存进去，解锁的时候再校验，一致就可以解，不一致就不能解。**

# 扩展知识