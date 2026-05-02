# 14-tomcat

所有者: junk01

**Tomcat中为什么要使⽤⾃定义类加载器**

⼀个Tomcat中可以部署多个应⽤，⽽每个应⽤中都存在很多类，并且各个应⽤中的类是独⽴的，全类名是可以相同的，⽐如⼀个订单系统中可能存在com.zhouyu.User类，⼀个库存系统中可能也存在com.zhouyu.User类，⼀个Tomcat，不管内部部署了多少应⽤，Tomcat启动之后就是⼀个Java进程，

也就是⼀个JVM，所以如果Tomcat中只存在⼀个类加载器，⽐如默认的AppClassLoader，那么就只能加载⼀个com.zhouyu.User类，这是有问题的，⽽在Tomcat中，会为部署的每个应⽤都⽣成⼀个类加载器实例，名字叫做WebAppClassLoader，这样Tomcat中每个应⽤就可以使⽤⾃⼰的类加载器去加载⾃⼰的类，从⽽达到应⽤之间的类隔离，不出现冲突。另外Tomcat还利⽤⾃定义加载器实现了热加载功能。

**Tomcat如何进⾏优化？**

对于Tomcat调优，可以从两个⽅⾯来进⾏调整：内存和线程。

⾸先启动Tomcat，实际上就是启动了⼀个JVM，所以可以按JVM调优的⽅式来进⾏调整，从⽽达到Tomcat优化的⽬的。

另外Tomcat中设计了⼀些缓存区，⽐如appReadBufSize、bufferPoolSize等缓存区来提⾼吞吐量。

还可以调整Tomcat的线程，⽐如调整minSpareThreads参数来改变Tomcat空闲时的线程数，调整maxThreads参数来设置Tomcat处理连接的最⼤线程数。

并且还可以调整IO模型，⽐如使⽤NIO、APR这种相⽐于BIO更加⾼效的IO模型。