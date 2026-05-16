# 02-Spark

所有者: junk01

1、设计理念不同

flink：Flink是基于事件驱动的，是面向流的处理框架, Flink基于每个事件一行一行地流式处理，是真正的流式计算. 另外他也可以基于流来模拟批进行计算实现批处理。

spark：Spark的技术理念是使用微批来模拟流的计算,基于Micro-batch,数据流以时间为单位被切分为一个个批次,通过分布式数据集RDD进行批量处理,是一种伪实时。

2、架构区别：

flink：Flink 在运行时主要包含：Jobmanager、Taskmanager和Slot。

spark：Spark在运行时的主要角色包括：Master、Worker、Driver、Executor。

3、任务调度不同：

flink：Flink 根据用户提交的代码生成 StreamGraph，经过优化生成 JobGraph，然后提交给 JobManager进行处理，JobManager 会根据 JobGraph 生成 ExecutionGraph，ExecutionGraph 是 Flink 调度最核心的数据结构，JobManager 根据 ExecutionGraph 对 Job 进行调度。

spark：Spark Streaming 连续不断的生成微小的数据批次，构建有向无环图DAG，根据DAG中的action操作形成job，每个job有根据窄宽依赖生成多个stage。

4、时间机制不同

flink：flink支持三种时间机制：事件时间，注入时间，处理时间，同时支持 watermark 机制处理迟到的数据,说明Flink在处理乱序大实时数据的时候,更有优势。

spark：Spark Streaming 支持的时间机制有限，只支持处理时间。使用processing time模拟event time必然会有误差， 如果产生数据堆积的话，误差则更明显。

5、容错机制不同

flink：Flink 则使用两阶段提交协议来保证exactly once。

spark：Spark Streaming的容错机制是基于RDD的容错机制，会将经常用的RDD或者对宽依赖加Checkpoint。利用SparkStreaming的direct方式与Kafka可以保证数据输入源的，处理过程，输出过程符合exactly once。

6、吞吐量与延迟不同

flink：Flink是基于事件的,消息逐条处理,而且他的容错机制很轻量级,所以他能在兼顾高吞吐量的同时又有很低的延迟,它的延迟能够达到毫秒级;

spark：spark是基于微批的,而且流水线优化做的很好,所以说他的吞入量是最大的,但是付出了延迟的代价,它的延迟是秒级;

7、状态不同

flink：flink是事件驱动型应用是一类具有状态的应用，我们要把它看成一个个event记录去处理，当遇到窗口时会进行阻塞等待，窗口的聚合操作是无状态的。过了窗口后DataStream的算子聚合操作就是有状态的操作了，所以flink要把聚合操作都放到窗口操作之前，才能进行无状态的聚合操作。而spark全程都是无状态的，所以在哪聚合都可以。

spark：spark本身是无状态的，所以我们可以把它看成一个rdd一个算子一个rdd的去处理，就是说可以看成分段处理。

8、数据不同

flink：在flink的世界观中，一切都是由流组成的，离线数据是有界限的流，实时数据是一个没有界限的流，这就是所谓的有界流和无界流。流处理的特点是无界、实时, 无需针对整个数据集执行操作，而是对通过系统传输的每个数据项执行操作，一般用于实时统计。

spark：在spark的世界观中，一切都是由批次组成的，离线数据是一个大批次，而实时数据是由一个一个无限的小批次组成的。批处理的特点是有界、持久、大量，非常适合需要访问全套记录才能完成的计算工作，一般用于离线统计。

![](https://www.notion.so02-Spark.resources/5D0FBF8C-09DA-4122-B539-4DD2588F1445.png)

RDD(分布式内存抽象)是 Spark 中的核心数据模型，一个 RDD 代表着一个被分区(partition)的只读数据集。

RDD的生成只有两种途径：

一种是来自于内存集合或外部存储系统；

另一种是通过转换操作来自于其他RDD；

什么是Spark？它与Hadoop的区别是什么？

答案：Spark是一个快速的、通用的大数据处理框架，与Hadoop相比，Spark具有更高的执行速度、内存计算能力和更丰富的API支持。

Spark中的RDD是什么？请解释其特点和作用。

答案：RDD（弹性分布式数据集）是Spark的核心数据抽象，它是一个不可变的分布式对象集合，具有容错性和并行计算能力，用于存储和操作大规模数据。

Spark的执行模式有哪些？

答案：Spark的执行模式包括本地模式、集群模式、客户端模式和集群管理器模式（如YARN和Mesos）。

请解释一下Spark的懒执行特性。

答案：Spark具有懒执行特性，即在遇到动作操作前，它只会执行转换操作并构建执行计划，而不会立即执行计算，从而提高了效率和性能。

什么是Spark的广播变量？

答案：广播变量是Spark中的一种共享变量，它可以在集群中的所有节点上共享，减少数据传输和复制的开销，提高性能。

Spark中的shuffle操作是什么？为什么它对性能有影响？

答案：Shuffle操作是指将数据重新分区的过程，它对性能有影响，因为它涉及数据的重新排序和传输，可能导致大量的磁盘读写和网络传输。

Spark Streaming和Spark SQL有什么区别？

答案：Spark Streaming用于实时流数据处理，而Spark SQL用于结构化数据处理和SQL查询。Spark Streaming基于微批处理模式，而Spark SQL基于批处理模式。

请解释一下Spark的RDD持久化（RDD Persistence）。

答案：RDD持久化是指将RDD的计算结果缓存在内存或磁盘上，以便后续重用，提高计算性能。

什么是Spark的任务调度器（Task Scheduler）？

答案：Spark的任务调度器负责将任务分配给可用的执行器（Executor）并管理任务的执行顺序和资源分配。

Spark的作业优化技术有哪些？

答案：Spark的作业优化技术包括数据本地性优化、宽依赖（Wide Dependency）优化、分区（Partition）调整优化、内存管理优化等，这些技术旨在提高作业的执行效率和性能。