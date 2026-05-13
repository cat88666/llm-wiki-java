# Redis

所有者: junk01

本分类共 **73** 篇文章。

- ✅Redis是AP的还是CP的？
- ✅介绍一下Redis的集群模式？
- ✅什么是Redis的数据分片？
- ✅Redis 使用什么协议进行通信？
- ✅Redis 与 Memcached 有什么区别？
- ✅Redis为什么这么快？
- ✅Redis 支持哪几种数据类型？
- ✅Redis为什么要自己定义SDS？
- ✅Redis中的Zset是怎么实现的？
- ✅什么是GEO，有什么用？
- ✅Redis为什么被设计成是单线程的？
- ✅为什么Redis设计成单线程也能这么快？
- ✅为什么Redis 6.0引入了多线程？
- ✅为什么Lua脚本可以保证原子性？
- ✅Redis中的setnx命令为什么是原子性的
- ✅Redis 5.0中的 Stream是什么？
- ✅Redis的虚拟内存机制是什么？
- ✅Redis的持久化机制是怎样的？
- ✅Redis 的事务机制是怎样的？
- ✅Redis 的过期策略是怎么样的？
- ✅Redis的内存淘汰策略是怎么样的？
- ✅什么是热Key问题，如何解决热key问题
- ✅什么是大Key问题，如何解决？
- ✅什么是缓存击穿、缓存穿透、缓存雪崩？
- ✅什么情况下会出现数据库和缓存不一致的问题？
- ✅如何解决Redis和数据库的一致性问题？
- ✅Redis如何实现延迟消息？
- ✅Redis如何实现发布/订阅？
- ✅除了做缓存，Redis还能用来干什么？
- ✅对于 Redis 的操作，有哪些推荐的 Best Practices？
- ✅如何用SETNX实现分布式锁？
- ✅什么是RedLock，他解决了什么问题？
- ✅如何用Redisson实现分布式锁？
- ✅为什么ZSet 既能支持高效的范围查询，还能以 O(1) 复杂度获取元素权重值？
- ✅Redisson的watchdog机制是怎么样的？
- ✅什么是Redis的渐进式rehash
- ✅如何基于Redisson实现一个延迟队列
- ✅介绍下Redis集群的脑裂问题？
- ✅Redis中key过期了一定会立即删除吗
- ✅Redis中有一批key瞬间过期，为什么其它key的读写效率会降低？
- ✅如何基于Redis实现滑动窗口限流？
- ✅为什么需要延迟双删，两次删除的原因是什么？
- ✅Redis的Key和Value的设计原则有哪些？
- ✅Redisson和Jedis有啥区别？如何选择？
- ✅什么是Redis的Pipeline，和事务有什么区别？
- ✅Redis的事务和Lua之间有哪些区别？
- ✅Redisson的lock和tryLock有什么区别？
- ✅为什么Redis不支持回滚？
- ✅如何用Redis实现乐观锁？
- ✅watchdog一直续期，那客户端挂了怎么办？
- ✅如何用setnx实现一个可重入锁？
- ✅Redis实现分布锁的时候，哪些问题需要考虑？
- ✅Redis如何高效安全的遍历所有key
- ✅Redisson解锁失败，watchdog会不会一直续期下去？
- ✅Redis Cluster 中使用事务和 lua 有什么限制？
- ✅如何在 Redis Cluster 中执行 lua 脚本？
- ✅Redisson 中为什么要废弃 RedLock，该用啥？
- ✅Redisson 的 watchdog 什么情况下可能会失效？
- ✅Redisson如何保证解锁的线程一定是加锁的线程？
- ✅RDB和AOF的写回策略分别是什么？
- ✅Redis能完全保证数据不丢失吗？
- ✅Redis的事务和MySQL的事务区别？
- ✅Redis中hash结构比string的好处有哪些？
- ✅Redis 8.0有哪些新特性？
- ✅Redis中的setnx和setex有啥区别？
- ✅ZSet为什么在数据量少的时候用ZipList，而在数据量大的时候转成SkipList？
- ✅介绍下Redis中的ZipList和他的级联更新问题
- ✅Redis中的ListPack是如何解决级联更新问题的？
- ✅Redis的ZipList、SkipList和ListPack之间有什么区别？
- ✅了解Redis的内存碎片吗？
- ✅Redis中的hash和Java中的HashMap有啥区别？
- ✅Redisson里面的锁是怎么来防止误删的？
- ✅Redisson里面的锁是如何实现可重入的？

[✅Redis是AP的还是CP的？](Redis/%E2%9C%85Redis%E6%98%AFAP%E7%9A%84%E8%BF%98%E6%98%AFCP%E7%9A%84%EF%BC%9F.md)

[✅介绍一下Redis的集群模式？](Redis/%E2%9C%85%E4%BB%8B%E7%BB%8D%E4%B8%80%E4%B8%8BRedis%E7%9A%84%E9%9B%86%E7%BE%A4%E6%A8%A1%E5%BC%8F%EF%BC%9F.md)

[✅什么是Redis的数据分片？](Redis/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AFRedis%E7%9A%84%E6%95%B0%E6%8D%AE%E5%88%86%E7%89%87%EF%BC%9F.md)

[✅Redis 使用什么协议进行通信？](Redis/%E2%9C%85Redis%20%E4%BD%BF%E7%94%A8%E4%BB%80%E4%B9%88%E5%8D%8F%E8%AE%AE%E8%BF%9B%E8%A1%8C%E9%80%9A%E4%BF%A1%EF%BC%9F.md)

[✅Redis 与 Memcached 有什么区别？](Redis/%E2%9C%85Redis%20%E4%B8%8E%20Memcached%20%E6%9C%89%E4%BB%80%E4%B9%88%E5%8C%BA%E5%88%AB%EF%BC%9F.md)

[✅Redis为什么这么快？](Redis/%E2%9C%85Redis%E4%B8%BA%E4%BB%80%E4%B9%88%E8%BF%99%E4%B9%88%E5%BF%AB%EF%BC%9F.md)

[✅Redis 支持哪几种数据类型？](Redis/%E2%9C%85Redis%20%E6%94%AF%E6%8C%81%E5%93%AA%E5%87%A0%E7%A7%8D%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%EF%BC%9F.md)

[✅Redis为什么要自己定义SDS？](Redis/%E2%9C%85Redis%E4%B8%BA%E4%BB%80%E4%B9%88%E8%A6%81%E8%87%AA%E5%B7%B1%E5%AE%9A%E4%B9%89SDS%EF%BC%9F.md)

[✅Redis中的Zset是怎么实现的？](Redis/%E2%9C%85Redis%E4%B8%AD%E7%9A%84Zset%E6%98%AF%E6%80%8E%E4%B9%88%E5%AE%9E%E7%8E%B0%E7%9A%84%EF%BC%9F.md)

[✅什么是GEO，有什么用？](Redis/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AFGEO%EF%BC%8C%E6%9C%89%E4%BB%80%E4%B9%88%E7%94%A8%EF%BC%9F.md)

[✅Redis为什么被设计成是单线程的？](Redis/%E2%9C%85Redis%E4%B8%BA%E4%BB%80%E4%B9%88%E8%A2%AB%E8%AE%BE%E8%AE%A1%E6%88%90%E6%98%AF%E5%8D%95%E7%BA%BF%E7%A8%8B%E7%9A%84%EF%BC%9F.md)

[✅为什么Redis设计成单线程也能这么快？](Redis/%E2%9C%85%E4%B8%BA%E4%BB%80%E4%B9%88Redis%E8%AE%BE%E8%AE%A1%E6%88%90%E5%8D%95%E7%BA%BF%E7%A8%8B%E4%B9%9F%E8%83%BD%E8%BF%99%E4%B9%88%E5%BF%AB%EF%BC%9F.md)

[✅为什么Redis 6.0引入了多线程？](Redis/%E2%9C%85%E4%B8%BA%E4%BB%80%E4%B9%88Redis%206%200%E5%BC%95%E5%85%A5%E4%BA%86%E5%A4%9A%E7%BA%BF%E7%A8%8B%EF%BC%9F.md)

[✅为什么Lua脚本可以保证原子性？](Redis/%E2%9C%85%E4%B8%BA%E4%BB%80%E4%B9%88Lua%E8%84%9A%E6%9C%AC%E5%8F%AF%E4%BB%A5%E4%BF%9D%E8%AF%81%E5%8E%9F%E5%AD%90%E6%80%A7%EF%BC%9F.md)

[✅Redis中的setnx命令为什么是原子性的](Redis/%E2%9C%85Redis%E4%B8%AD%E7%9A%84setnx%E5%91%BD%E4%BB%A4%E4%B8%BA%E4%BB%80%E4%B9%88%E6%98%AF%E5%8E%9F%E5%AD%90%E6%80%A7%E7%9A%84.md)

[✅Redis 5.0中的 Stream是什么？](Redis/%E2%9C%85Redis%205%200%E4%B8%AD%E7%9A%84%20Stream%E6%98%AF%E4%BB%80%E4%B9%88%EF%BC%9F.md)

[✅Redis的虚拟内存机制是什么？](Redis/%E2%9C%85Redis%E7%9A%84%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98%E6%9C%BA%E5%88%B6%E6%98%AF%E4%BB%80%E4%B9%88%EF%BC%9F.md)

[✅Redis的持久化机制是怎样的？](Redis/%E2%9C%85Redis%E7%9A%84%E6%8C%81%E4%B9%85%E5%8C%96%E6%9C%BA%E5%88%B6%E6%98%AF%E6%80%8E%E6%A0%B7%E7%9A%84%EF%BC%9F.md)

[✅Redis 的事务机制是怎样的？](Redis/%E2%9C%85Redis%20%E7%9A%84%E4%BA%8B%E5%8A%A1%E6%9C%BA%E5%88%B6%E6%98%AF%E6%80%8E%E6%A0%B7%E7%9A%84%EF%BC%9F.md)

[✅Redis 的过期策略是怎么样的？](Redis/%E2%9C%85Redis%20%E7%9A%84%E8%BF%87%E6%9C%9F%E7%AD%96%E7%95%A5%E6%98%AF%E6%80%8E%E4%B9%88%E6%A0%B7%E7%9A%84%EF%BC%9F.md)

[✅Redis的内存淘汰策略是怎么样的？](Redis/%E2%9C%85Redis%E7%9A%84%E5%86%85%E5%AD%98%E6%B7%98%E6%B1%B0%E7%AD%96%E7%95%A5%E6%98%AF%E6%80%8E%E4%B9%88%E6%A0%B7%E7%9A%84%EF%BC%9F.md)

[✅什么是热Key问题，如何解决热key问题](Redis/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AF%E7%83%ADKey%E9%97%AE%E9%A2%98%EF%BC%8C%E5%A6%82%E4%BD%95%E8%A7%A3%E5%86%B3%E7%83%ADkey%E9%97%AE%E9%A2%98.md)

[✅什么是大Key问题，如何解决？](Redis/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AF%E5%A4%A7Key%E9%97%AE%E9%A2%98%EF%BC%8C%E5%A6%82%E4%BD%95%E8%A7%A3%E5%86%B3%EF%BC%9F.md)

[✅什么是缓存击穿、缓存穿透、缓存雪崩？](Redis/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AF%E7%BC%93%E5%AD%98%E5%87%BB%E7%A9%BF%E3%80%81%E7%BC%93%E5%AD%98%E7%A9%BF%E9%80%8F%E3%80%81%E7%BC%93%E5%AD%98%E9%9B%AA%E5%B4%A9%EF%BC%9F.md)

[✅什么情况下会出现数据库和缓存不一致的问题？](Redis/%E2%9C%85%E4%BB%80%E4%B9%88%E6%83%85%E5%86%B5%E4%B8%8B%E4%BC%9A%E5%87%BA%E7%8E%B0%E6%95%B0%E6%8D%AE%E5%BA%93%E5%92%8C%E7%BC%93%E5%AD%98%E4%B8%8D%E4%B8%80%E8%87%B4%E7%9A%84%E9%97%AE%E9%A2%98%EF%BC%9F.md)

[✅如何解决Redis和数据库的一致性问题？](Redis/%E2%9C%85%E5%A6%82%E4%BD%95%E8%A7%A3%E5%86%B3Redis%E5%92%8C%E6%95%B0%E6%8D%AE%E5%BA%93%E7%9A%84%E4%B8%80%E8%87%B4%E6%80%A7%E9%97%AE%E9%A2%98%EF%BC%9F.md)

[✅Redis如何实现延迟消息？](Redis/%E2%9C%85Redis%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E5%BB%B6%E8%BF%9F%E6%B6%88%E6%81%AF%EF%BC%9F.md)

[✅Redis如何实现发布/订阅？](Redis/%E2%9C%85Redis%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E5%8F%91%E5%B8%83%20%E8%AE%A2%E9%98%85%EF%BC%9F.md)

[✅除了做缓存，Redis还能用来干什么？](Redis/%E2%9C%85%E9%99%A4%E4%BA%86%E5%81%9A%E7%BC%93%E5%AD%98%EF%BC%8CRedis%E8%BF%98%E8%83%BD%E7%94%A8%E6%9D%A5%E5%B9%B2%E4%BB%80%E4%B9%88%EF%BC%9F.md)

[✅对于 Redis 的操作，有哪些推荐的 Best Practices？](Redis/%E2%9C%85%E5%AF%B9%E4%BA%8E%20Redis%20%E7%9A%84%E6%93%8D%E4%BD%9C%EF%BC%8C%E6%9C%89%E5%93%AA%E4%BA%9B%E6%8E%A8%E8%8D%90%E7%9A%84%20Best%20Practices%EF%BC%9F.md)

[✅如何用SETNX实现分布式锁？](Redis/%E2%9C%85%E5%A6%82%E4%BD%95%E7%94%A8SETNX%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%EF%BC%9F.md)

[✅什么是RedLock，他解决了什么问题？](Redis/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AFRedLock%EF%BC%8C%E4%BB%96%E8%A7%A3%E5%86%B3%E4%BA%86%E4%BB%80%E4%B9%88%E9%97%AE%E9%A2%98%EF%BC%9F.md)

[✅如何用Redisson实现分布式锁？](Redis/%E2%9C%85%E5%A6%82%E4%BD%95%E7%94%A8Redisson%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%EF%BC%9F.md)

[✅为什么ZSet 既能支持高效的范围查询，还能以 O(1) 复杂度获取元素权重值？](Redis/%E2%9C%85%E4%B8%BA%E4%BB%80%E4%B9%88ZSet%20%E6%97%A2%E8%83%BD%E6%94%AF%E6%8C%81%E9%AB%98%E6%95%88%E7%9A%84%E8%8C%83%E5%9B%B4%E6%9F%A5%E8%AF%A2%EF%BC%8C%E8%BF%98%E8%83%BD%E4%BB%A5%20O(1)%20%E5%A4%8D%E6%9D%82%E5%BA%A6%E8%8E%B7%E5%8F%96%E5%85%83%E7%B4%A0%E6%9D%83%E9%87%8D%E5%80%BC%EF%BC%9F.md)

[✅Redisson的watchdog机制是怎么样的？](Redis/%E2%9C%85Redisson%E7%9A%84watchdog%E6%9C%BA%E5%88%B6%E6%98%AF%E6%80%8E%E4%B9%88%E6%A0%B7%E7%9A%84%EF%BC%9F.md)

[✅什么是Redis的渐进式rehash](Redis/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AFRedis%E7%9A%84%E6%B8%90%E8%BF%9B%E5%BC%8Frehash.md)

[✅如何基于Redisson实现一个延迟队列](Redis/%E2%9C%85%E5%A6%82%E4%BD%95%E5%9F%BA%E4%BA%8ERedisson%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AA%E5%BB%B6%E8%BF%9F%E9%98%9F%E5%88%97.md)

[✅介绍下Redis集群的脑裂问题？](Redis/%E2%9C%85%E4%BB%8B%E7%BB%8D%E4%B8%8BRedis%E9%9B%86%E7%BE%A4%E7%9A%84%E8%84%91%E8%A3%82%E9%97%AE%E9%A2%98%EF%BC%9F.md)

[✅Redis中key过期了一定会立即删除吗](Redis/%E2%9C%85Redis%E4%B8%ADkey%E8%BF%87%E6%9C%9F%E4%BA%86%E4%B8%80%E5%AE%9A%E4%BC%9A%E7%AB%8B%E5%8D%B3%E5%88%A0%E9%99%A4%E5%90%97.md)

[✅Redis中有一批key瞬间过期，为什么其它key的读写效率会降低？](Redis/%E2%9C%85Redis%E4%B8%AD%E6%9C%89%E4%B8%80%E6%89%B9key%E7%9E%AC%E9%97%B4%E8%BF%87%E6%9C%9F%EF%BC%8C%E4%B8%BA%E4%BB%80%E4%B9%88%E5%85%B6%E5%AE%83key%E7%9A%84%E8%AF%BB%E5%86%99%E6%95%88%E7%8E%87%E4%BC%9A%E9%99%8D%E4%BD%8E%EF%BC%9F.md)

[✅如何基于Redis实现滑动窗口限流？](Redis/%E2%9C%85%E5%A6%82%E4%BD%95%E5%9F%BA%E4%BA%8ERedis%E5%AE%9E%E7%8E%B0%E6%BB%91%E5%8A%A8%E7%AA%97%E5%8F%A3%E9%99%90%E6%B5%81%EF%BC%9F.md)

[✅为什么需要延迟双删，两次删除的原因是什么？](Redis/%E2%9C%85%E4%B8%BA%E4%BB%80%E4%B9%88%E9%9C%80%E8%A6%81%E5%BB%B6%E8%BF%9F%E5%8F%8C%E5%88%A0%EF%BC%8C%E4%B8%A4%E6%AC%A1%E5%88%A0%E9%99%A4%E7%9A%84%E5%8E%9F%E5%9B%A0%E6%98%AF%E4%BB%80%E4%B9%88%EF%BC%9F.md)

[✅Redis的Key和Value的设计原则有哪些？](Redis/%E2%9C%85Redis%E7%9A%84Key%E5%92%8CValue%E7%9A%84%E8%AE%BE%E8%AE%A1%E5%8E%9F%E5%88%99%E6%9C%89%E5%93%AA%E4%BA%9B%EF%BC%9F.md)

[✅Redisson和Jedis有啥区别？如何选择？](Redis/%E2%9C%85Redisson%E5%92%8CJedis%E6%9C%89%E5%95%A5%E5%8C%BA%E5%88%AB%EF%BC%9F%E5%A6%82%E4%BD%95%E9%80%89%E6%8B%A9%EF%BC%9F.md)

[✅什么是Redis的Pipeline，和事务有什么区别？](Redis/%E2%9C%85%E4%BB%80%E4%B9%88%E6%98%AFRedis%E7%9A%84Pipeline%EF%BC%8C%E5%92%8C%E4%BA%8B%E5%8A%A1%E6%9C%89%E4%BB%80%E4%B9%88%E5%8C%BA%E5%88%AB%EF%BC%9F.md)

[✅Redis的事务和Lua之间有哪些区别？](Redis/%E2%9C%85Redis%E7%9A%84%E4%BA%8B%E5%8A%A1%E5%92%8CLua%E4%B9%8B%E9%97%B4%E6%9C%89%E5%93%AA%E4%BA%9B%E5%8C%BA%E5%88%AB%EF%BC%9F.md)

[✅Redisson的lock和tryLock有什么区别？](Redis/%E2%9C%85Redisson%E7%9A%84lock%E5%92%8CtryLock%E6%9C%89%E4%BB%80%E4%B9%88%E5%8C%BA%E5%88%AB%EF%BC%9F.md)

[✅为什么Redis不支持回滚？](Redis/%E2%9C%85%E4%B8%BA%E4%BB%80%E4%B9%88Redis%E4%B8%8D%E6%94%AF%E6%8C%81%E5%9B%9E%E6%BB%9A%EF%BC%9F.md)

[✅如何用Redis实现乐观锁？](Redis/%E2%9C%85%E5%A6%82%E4%BD%95%E7%94%A8Redis%E5%AE%9E%E7%8E%B0%E4%B9%90%E8%A7%82%E9%94%81%EF%BC%9F.md)

[✅watchdog一直续期，那客户端挂了怎么办？](Redis/%E2%9C%85watchdog%E4%B8%80%E7%9B%B4%E7%BB%AD%E6%9C%9F%EF%BC%8C%E9%82%A3%E5%AE%A2%E6%88%B7%E7%AB%AF%E6%8C%82%E4%BA%86%E6%80%8E%E4%B9%88%E5%8A%9E%EF%BC%9F.md)

[✅如何用setnx实现一个可重入锁？](Redis/%E2%9C%85%E5%A6%82%E4%BD%95%E7%94%A8setnx%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AA%E5%8F%AF%E9%87%8D%E5%85%A5%E9%94%81%EF%BC%9F.md)

[✅Redis实现分布锁的时候，哪些问题需要考虑？](Redis/%E2%9C%85Redis%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E9%94%81%E7%9A%84%E6%97%B6%E5%80%99%EF%BC%8C%E5%93%AA%E4%BA%9B%E9%97%AE%E9%A2%98%E9%9C%80%E8%A6%81%E8%80%83%E8%99%91%EF%BC%9F.md)

[✅Redis如何高效安全的遍历所有key](Redis/%E2%9C%85Redis%E5%A6%82%E4%BD%95%E9%AB%98%E6%95%88%E5%AE%89%E5%85%A8%E7%9A%84%E9%81%8D%E5%8E%86%E6%89%80%E6%9C%89key.md)

[✅Redisson解锁失败，watchdog会不会一直续期下去？](Redis/%E2%9C%85Redisson%E8%A7%A3%E9%94%81%E5%A4%B1%E8%B4%A5%EF%BC%8Cwatchdog%E4%BC%9A%E4%B8%8D%E4%BC%9A%E4%B8%80%E7%9B%B4%E7%BB%AD%E6%9C%9F%E4%B8%8B%E5%8E%BB%EF%BC%9F.md)

[✅Redis Cluster 中使用事务和 lua 有什么限制？](Redis/%E2%9C%85Redis%20Cluster%20%E4%B8%AD%E4%BD%BF%E7%94%A8%E4%BA%8B%E5%8A%A1%E5%92%8C%20lua%20%E6%9C%89%E4%BB%80%E4%B9%88%E9%99%90%E5%88%B6%EF%BC%9F.md)

[✅如何在 Redis Cluster 中执行 lua 脚本？](Redis/%E2%9C%85%E5%A6%82%E4%BD%95%E5%9C%A8%20Redis%20Cluster%20%E4%B8%AD%E6%89%A7%E8%A1%8C%20lua%20%E8%84%9A%E6%9C%AC%EF%BC%9F.md)

[✅Redisson 中为什么要废弃 RedLock，该用啥？](Redis/%E2%9C%85Redisson%20%E4%B8%AD%E4%B8%BA%E4%BB%80%E4%B9%88%E8%A6%81%E5%BA%9F%E5%BC%83%20RedLock%EF%BC%8C%E8%AF%A5%E7%94%A8%E5%95%A5%EF%BC%9F.md)

[✅Redisson 的 watchdog 什么情况下可能会失效？](Redis/%E2%9C%85Redisson%20%E7%9A%84%20watchdog%20%E4%BB%80%E4%B9%88%E6%83%85%E5%86%B5%E4%B8%8B%E5%8F%AF%E8%83%BD%E4%BC%9A%E5%A4%B1%E6%95%88%EF%BC%9F.md)

[✅Redisson如何保证解锁的线程一定是加锁的线程？](Redis/%E2%9C%85Redisson%E5%A6%82%E4%BD%95%E4%BF%9D%E8%AF%81%E8%A7%A3%E9%94%81%E7%9A%84%E7%BA%BF%E7%A8%8B%E4%B8%80%E5%AE%9A%E6%98%AF%E5%8A%A0%E9%94%81%E7%9A%84%E7%BA%BF%E7%A8%8B%EF%BC%9F.md)

[✅RDB和AOF的写回策略分别是什么？](Redis/%E2%9C%85RDB%E5%92%8CAOF%E7%9A%84%E5%86%99%E5%9B%9E%E7%AD%96%E7%95%A5%E5%88%86%E5%88%AB%E6%98%AF%E4%BB%80%E4%B9%88%EF%BC%9F.md)

[✅Redis能完全保证数据不丢失吗？](Redis/%E2%9C%85Redis%E8%83%BD%E5%AE%8C%E5%85%A8%E4%BF%9D%E8%AF%81%E6%95%B0%E6%8D%AE%E4%B8%8D%E4%B8%A2%E5%A4%B1%E5%90%97%EF%BC%9F.md)

[✅Redis的事务和MySQL的事务区别？](Redis/%E2%9C%85Redis%E7%9A%84%E4%BA%8B%E5%8A%A1%E5%92%8CMySQL%E7%9A%84%E4%BA%8B%E5%8A%A1%E5%8C%BA%E5%88%AB%EF%BC%9F.md)

[✅Redis中hash结构比string的好处有哪些？](Redis/%E2%9C%85Redis%E4%B8%ADhash%E7%BB%93%E6%9E%84%E6%AF%94string%E7%9A%84%E5%A5%BD%E5%A4%84%E6%9C%89%E5%93%AA%E4%BA%9B%EF%BC%9F.md)

[✅Redis 8.0有哪些新特性？](Redis/%E2%9C%85Redis%208%200%E6%9C%89%E5%93%AA%E4%BA%9B%E6%96%B0%E7%89%B9%E6%80%A7%EF%BC%9F.md)

[✅Redis中的setnx和setex有啥区别？](Redis/%E2%9C%85Redis%E4%B8%AD%E7%9A%84setnx%E5%92%8Csetex%E6%9C%89%E5%95%A5%E5%8C%BA%E5%88%AB%EF%BC%9F.md)

[✅ZSet为什么在数据量少的时候用ZipList，而在数据量大的时候转成SkipList？](Redis/%E2%9C%85ZSet%E4%B8%BA%E4%BB%80%E4%B9%88%E5%9C%A8%E6%95%B0%E6%8D%AE%E9%87%8F%E5%B0%91%E7%9A%84%E6%97%B6%E5%80%99%E7%94%A8ZipList%EF%BC%8C%E8%80%8C%E5%9C%A8%E6%95%B0%E6%8D%AE%E9%87%8F%E5%A4%A7%E7%9A%84%E6%97%B6%E5%80%99%E8%BD%AC%E6%88%90SkipList%EF%BC%9F.md)

[✅介绍下Redis中的ZipList和他的级联更新问题](Redis/%E2%9C%85%E4%BB%8B%E7%BB%8D%E4%B8%8BRedis%E4%B8%AD%E7%9A%84ZipList%E5%92%8C%E4%BB%96%E7%9A%84%E7%BA%A7%E8%81%94%E6%9B%B4%E6%96%B0%E9%97%AE%E9%A2%98.md)

[✅Redis中的ListPack是如何解决级联更新问题的？](Redis/%E2%9C%85Redis%E4%B8%AD%E7%9A%84ListPack%E6%98%AF%E5%A6%82%E4%BD%95%E8%A7%A3%E5%86%B3%E7%BA%A7%E8%81%94%E6%9B%B4%E6%96%B0%E9%97%AE%E9%A2%98%E7%9A%84%EF%BC%9F.md)

[✅Redis的ZipList、SkipList和ListPack之间有什么区别？](Redis/%E2%9C%85Redis%E7%9A%84ZipList%E3%80%81SkipList%E5%92%8CListPack%E4%B9%8B%E9%97%B4%E6%9C%89%E4%BB%80%E4%B9%88%E5%8C%BA%E5%88%AB%EF%BC%9F.md)

[✅了解Redis的内存碎片吗？](Redis/%E2%9C%85%E4%BA%86%E8%A7%A3Redis%E7%9A%84%E5%86%85%E5%AD%98%E7%A2%8E%E7%89%87%E5%90%97%EF%BC%9F.md)

[✅Redis中的hash和Java中的HashMap有啥区别？](Redis/%E2%9C%85Redis%E4%B8%AD%E7%9A%84hash%E5%92%8CJava%E4%B8%AD%E7%9A%84HashMap%E6%9C%89%E5%95%A5%E5%8C%BA%E5%88%AB%EF%BC%9F.md)

[✅Redisson里面的锁是怎么来防止误删的？](Redis/%E2%9C%85Redisson%E9%87%8C%E9%9D%A2%E7%9A%84%E9%94%81%E6%98%AF%E6%80%8E%E4%B9%88%E6%9D%A5%E9%98%B2%E6%AD%A2%E8%AF%AF%E5%88%A0%E7%9A%84%EF%BC%9F.md)

[✅Redisson里面的锁是如何实现可重入的？](Redis/%E2%9C%85Redisson%E9%87%8C%E9%9D%A2%E7%9A%84%E9%94%81%E6%98%AF%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E5%8F%AF%E9%87%8D%E5%85%A5%E7%9A%84%EF%BC%9F.md)