# ✅遍历的同时修改一个List有几种方式？

所有者: junk01

# ✅遍历的同时修改一个List有几种方式？

> 原文：[https://www.yuque.com/hollis666/asgng6/mba03d](https://www.yuque.com/hollis666/asgng6/mba03d)
> 

# 典型回答

我们知道，在foreach的同时修改集合，会触发fail-fast机制，要避免fail-fast机制，有如下处理方案：

1. 通过普通的for循环（不建议，可能会漏删）

1. 通过普通的for循环进行倒叙遍历（也能用）

上面的方式需要做i--避免漏删，还有个办法，那就是倒叙遍历也能避免这个问题：

1. 使用迭代器循环（可以用）
2. 将原来的copy一份副本，遍历原来的list，然后删除副本（可以用，fail-safe的，但是比较复杂）
3. 使用并发安全的集合类（可以用，但是需要转成线程安全的集合）
4. 通过Stream的过滤方法，因为Stream每次处理后都会生成一个新的Stream，不存在并发问题，所以Stream的filter也可以修改list集合。（**建议，简单高效**）

1. 通过removeIf方法，实现元素的过滤删除。从Java 8开始，List接口提供了removeIf方法用于删除所有满足特定条件的数组元素（**推荐**）