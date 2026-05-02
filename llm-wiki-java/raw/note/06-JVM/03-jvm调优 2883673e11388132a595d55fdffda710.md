# 03-jvm调优

所有者: junk01

**你们项⽬如何排查JVM问题？**

1. 可以使⽤jmap来查看JVM中各个区域的使⽤情况

2. 可以通过jstack来查看线程的运⾏情况，⽐如哪些线程阻塞、是否出现了死锁。可以通过jstack命令会显示发⽣了死锁的线程

3. 可以通过jstat命令来查看垃圾回收的情况，特别是fullgc，如果发现fullgc⽐较频繁，那么就得进⾏调 优了

4. 通过各个命令的结果，或者jvisualvm等⼯具来进⾏分析

5. 初步猜测频繁发送fullgc的原因，如果频繁发⽣fullgc但是⼜⼀直没有出现内存溢出，那么表示 fullgc实际上是回收了很多对象了，所以这些对象最好能在younggc过程中就直接回收掉，避免这些对 象进⼊到⽼年代，对于这种情况，就要考虑这些存活时间不⻓的对象是不是⽐较⼤，导致年轻代放不 下，直接进⼊到了⽼年代，尝试加⼤年轻代的⼤⼩，如果改完之后，fullgc减少，则证明修改有效

6. 同时，还可以找到占⽤CPU最多的线程，定位到具体的⽅法，优化这个⽅法的执⾏，看是否能避免某些 对象的创建，从⽽节省内存对于已经发⽣了OOM的系统：

1. ⼀般⽣产系统中都会设置当系统发⽣了OOM时，⽣成当时的dump⽂件（- XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/usr/local/base）

2. 我们可以利⽤jsisualvm等⼯具来分析dump⽂件

3. 根据dump⽂件找到异常的实例对象，和异常的线程（占⽤CPU⾼），定位到具体的代码

4. 然后再进⾏详细的分析和调试

**jvm运行情况预估**

用 jstat gc -pid 命令可以计算出如下一些关键数据，有了这些数据就可以采用之前介绍过的优化思路，先给自己的系统设置一些初始性的JVM参数，比如堆内存大小，年轻代大小，Eden和Survivor的比例，老年代的大小，大对象的阈值，大龄对象进入老年代的阈值等。

**年轻代对象增长的速率？**

可以执行命令 jstat -gc pid 1000 10 (每隔1秒执行1次命令，共执行10次)，通过观察EU(eden区的使用)来估算每秒eden大概新增多少对象，如果系统负载不高，可以把频率1秒换成1分钟，甚至10分钟来观察整体情况。注意，一般系统可能有高峰期和日常期，所以需要在不同的时间分别估算不同情况下对象增长速率。

**Young GC的触发频率和每次耗时**

知道年轻代对象增长速率我们就能推根据eden区的大小推算出Young GC大概多久触发一次，Young GC的平均耗时可以通过 YGCT/YGC 公式算出，根据结果我们大概就能知道系统大概多久会因为Young GC的执行而卡顿多久。

**每次Young GC后有多少对象存活和进入老年代**

这个因为之前已经大概知道Young GC的频率，假设是每5分钟一次，那么可以执行命令 jstat -gc pid 300000 10 ，观察每次结果eden，survivor和老年代使用的变化情况，在每次gc后eden区使用一般会大幅减少，survivor和老年代都有可能增长，这些增长的对象就是每次Young GC后存活的对象，同时还可以看出每次Young GC后进去老年代大概多少对象，从而可以推算出老年代对象增长速率。

**Full GC的触发频率和每次耗时**

知道了老年代对象的增长速率就可以推算出Full GC的触发频率了，Full GC的每次耗时可以用公式 FGCT/FGC 计算得出。优化思路其实简单来说就是尽量让每次Young GC后的存活对象小于Survivor区域的50%，都留存在年轻代里。尽量别让对象进入老年代。尽量减少Full GC的频率，避免频繁Full GC对JVM性能的影响。

**内存泄露**

JVM级缓存就简单使用一个hashmap不断往里面放缓存数据，但是很少考虑这个map的容量问题，结果这个缓存map越来越大，一直占用着老年代的很多空间，时间长了就会导致full gc非常频繁，这就是一种内存泄漏，对于一些老旧数据没有及时清理导致一直占用着宝贵的内存资源，时间长了除了导致full gc，还有可能导致OOM。

这种情况完全可以考虑采用一些成熟的JVM级缓存框架来解决，比如ehcache等自带一些LRU数据淘汰算法的框架来作为JVM级的缓存。

**调优工具**

Jmap、Jstack、Jinfo、Jstat

JVM垃圾收集算法

JAVA垃圾定位

1，引用计数算法

2，可达性分析算法

## **二、JVM内存参数设置**

Spring Boot程序的JVM参数设置格式(Tomcat启动直接加在bin目录下catalina.sh文件里)：

java -Xms2048M -Xmx2048M -Xmn1024M -Xss512K -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M -jar microservice-eureka-server.jar

- Xss：每个线程的栈大小
- Xms：初始堆大小，默认物理内存的1/64
- Xmx：最大堆大小，默认物理内存的1/4
- Xmn：新生代大小
- XX:NewSize：设置新生代初始大小
- XX:NewRatio：默认2表示新生代占年老代的1/2，占整个堆内存的1/3。
- XX:SurvivorRatio：默认8表示一个survivor区占用1/8的Eden内存，即1/10的新生代内存。

关于元空间的JVM参数有两个：-XX:MetaspaceSize=N和 -XX:MaxMetaspaceSize=N

- XX：MaxMetaspaceSize： 设置元空间最大值， 默认是-1， 即不限制， 或者说只受限于本地内存大小。
- XX：MetaspaceSize： 指定元空间触发Fullgc的初始阈值(元空间无固定初始大小)， 以字节为单位，默认是21M，达到该值就会触发full gc进行类型卸载， 同时收集器会对该值进行调整： 如果释放了大量的空间， 就适当降低该值； 如果释放了很少的空间， 那么在不超过-XX：MaxMetaspaceSize（如果设置了的话） 的情况下， 适当提高该值。这个跟早期jdk版本的-XX:PermSize参数意思不一样，-XX:PermSize代表永久代的初始容量。

由于调整元空间的大小需要Full GC，这是非常昂贵的操作，如果应用在启动的时候发生大量Full GC，通常都是由于永久代或元空间发生了大小调整，基于这种情况，一般建议在JVM参数中将MetaspaceSize和MaxMetaspaceSize设置成一样的值，并设置得比初始值要大，对于8G物理内存的机器来说，一般我会将这两个值都设置为256M。

**结论：通过上面这些内容介绍，大家应该对JVM优化有些概念了，就是尽可能让对象都在新生代里分配和回收，尽量别让太多对象频繁进入老年代，避免频繁对老年代进行垃圾回收，同时给系统充足的内存大小，避免新生代频繁的进行垃圾回收。**

一、JVM调优

gateway：8核16G，抗每秒2000+请求，32核64G可以抗住每秒上万请求，支撑1万+请求， 每秒1万请求用户日过亿；

5台8核16G，支撑10万+请求，10台32核64G ；

web服务：这个得根据业务的复杂度来看，一般就单台几百到几千的并发

缓存redis：单台几万的并发，要么用集群架构可以到几十万并发

数据库：正常8核16G扛个大几百并发问题不大，如果并发提高10倍到四五千，要么分库分表

横向扩容，要么增大机器配置，比如32核64G高配物理机，扛个三四千并发问题不大

淘宝双11最高54万并发每秒

‐Xloggc:d:/gc‐cms‐%t.log ‐Xms50M ‐Xmx50M ‐XX:MetaspaceSize=256M ‐XX:MaxMetaspaceSize=256M ‐XX:+PrintGCDetails ‐XX:+P rintGCDateStamps 2 ‐XX:+PrintGCTimeStamps ‐XX:+PrintGCCause ‐XX:+UseGCLogFileRotation ‐XX:NumberOfGCLogFiles=10 ‐XX:GCLogFileSize=100M 3 ‐XX:+UseParNewGC ‐XX:+UseConcMarkSweepGC

java -jar -server -Xms6g -Xmx10g -XX:MaxMetaspaceSize=256m -XX:MetaspaceSize=128m -verbose:gc -Xloggc:/data/logs/gc.log -XX:+DisableExplicitGC -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -XX:+PrintGCTimeStamps -XX:+PrintClassHistogram -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath= -XX:ErrorFile=/hs_err.log -XX:-OmitStackTraceInFastThrow -XX:+UseG1GC /data/vnpay-backend.war --spring.profiles.active=prod --server.port=8081

对于8G内存，我们一般是分配4G内存给JVM，正常的JVM参数配置如下：

‐Xms3072M ‐Xmx3072M ‐Xmn1536M ‐Xss1M ‐XX:PermSize=256M ‐XX:MaxPermSize=256M ‐XX:SurvivorRatio=8 ‐XX:MetaspaceSize=256M ‐XX:MaxMetaspaceSize=256M ‐XX:+UseParNewGC ‐XX:+UseConcMarkSweepGC ‐XX:CMSInitiatingOccupancyFraction=75 ‐XX:+UseCMSInitiatingOccupancyOnly ‐XX:+PrintGCDetails ‐XX:+P rintGCDateStamps 2 ‐Xloggc:d:/gc‐cms‐%t.log ‐XX:+PrintGCTimeStamps ‐XX:+PrintGCCause ‐XX:+UseGCLogFileRotation ‐XX:NumberOfGCLogFiles=10 ‐XX:GCLogFileSize=100M 3

/home/soft/jdk/jdk1.8.0_151/bin/java -Xms512m -Xmx512m -Xmn256m -Dnacos.standalone=true -Dnacos.member.list= -Djava.ext.dirs=/home/soft/jdk/jdk1.8.0_151/jre/lib/ext:/home/soft/jdk/jdk1.8.0_151/lib/ext -Xloggc:/home/soft/nacos/logs/nacos_gc.log -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -Dloader.path=/home/soft/nacos/plugins/health,/home/soft/nacos/plugins/cmdb -Dnacos.home=/home/soft/nacos -jar /home/soft/nacos/target/nacos-server.jar --spring.config.additional-location=file:/home/soft/nacos/conf/ --logging.config=/home/soft/nacos/conf/nacos-logback.xml --server.max-http-header-size=524288 nacos.nacos

‐Xms3072M 堆初始化大小

‐Xmx3072M 最大堆大小

‐Xmn1536M 新生代大小

‐Xss1M 每个线程的堆栈大小

‐XX:PermSize=256M 设置持久代(perm gen)初始值

‐XX:MaxPermSize=256M 设置持久代最大值

‐XX:SurvivorRatio=8 Eden区与Survivor区的大小比值，设置为8,则两个Survivor区与一个Eden区的比值为2:8,一个Survivor区占整个年轻代的1/10