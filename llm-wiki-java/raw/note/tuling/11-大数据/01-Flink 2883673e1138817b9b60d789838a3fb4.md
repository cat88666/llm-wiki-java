# 01-Flink

所有者: junk01

0.2 0.5

45K + 提成20%-50%

以Flink1.12版本为准

Flink执行具体任务的单元Slots插槽、如果slots不够，Flink就无法执行任务；

在Flink on Yarn中总共支持三种提交任务的方式：

1、 Application Mode应用模式：

在任务的启动时临时在yarn上申请一个flink集群。任务从main方法启动开始就会提交到yarn上的flink集群执行。执行完成集群就会立即注销。

2、 Per-job Cluster Mode 单任务模式：这种模式跟应用模式很类似，也会给每个应用在yarn上申请一个单独的flink集群。只不过这种模式下，任务是先在本地执行，构建数据处理链。构建完成后再将任务提交到flink集群上执行。

3、Session Mode 会话模式

就相当于是一个共享的模式。在yarn上申请一个固定的flink集群，然后所有任务都共享这个集群内的资源

在生产环境中，一般Yarn上的资源都比较充足，优先建议使用Perjob模式，其次是Application模式。 这两种模式能够更好进行应用隔离。当然，如果集群的资源确实非常紧张，也可以使用Session模式。

Flink运行架构：

JobManager处理器也称为Master，用于协调分布式任务执行。他们用来调度task进行具体的任务。

TaskManager处理器也称为Worker，用于实际执行任务。

Flink集群中可以包含多个JobManager集群，也可以有多个TaskManager进行并行计算。可以直接在物理机上启动，也可以Yarn资源调度框架启动。

JobManager在接收到任务执行的流程图：

![](https://www.notion.so01-Flink.resources/81EDF5CF-392C-4805-AD88-F5421D0E552D.png)

并发度与Slots：

每一个TaskManager是一个独立的JVM进程，他可以在独立的线程上执行一个或多个任务task。为了控制一个taskManager能接收多少个task，TaskManager上就会划分出多个slot来进行控制。 每个slot表示的是TaskManager上拥有资源的一个固定大小的子集。flink-conf.yaml配置文件中的taskmanager.numberOfTaskSlots属性就配置了配个taskManager上有多少个slot。默认值是1。这些slot之间的内存管理也就是数据是相互隔离的。而这些slot其实都是在同一个JVM进程中，所以这里的隔离并不涉及到CPU等其他资源的隔离。

Task Slot是一个静态的概念，代表的是TaskManager具有的并发执行能力。另外还有一个概念并行度 parallelism就是一个动态的概念，表示的是运行程序时实际需要使用的并发能力。这个是可以在flink程序中进行控制的。如果集群提供的slot资源不够，那程序就无法正常执行下去，会表现为任务阻塞或者超时异常。

JobManager整体上由三个功能模块组成：

ResourceManager在Flink集群中负责申请、提供和注销集群资源，并且管理task slots。

Dispatcher模块提供了一系列的REST接口来提交任务，Flink的控制台也是由这个模块来提供。并且对于每一个执行的任务，Dispatcher会启动一个新的JobMaster，来对任务进行协调

一个JobMaster负责管理一个单独的JobGraph。Flink集群中，同一时间可以运行多个任务，每个任务都由一个对应的JobMaster来管理

TaskManager

TaskManager也成为Worker。每个TaskManager上可以有一个或多个Slot。这些Slot就是程序运行的最小单元。 在flink.conf.yaml文件中通过taskmanager.numberOfTaskSlots属性进行配置

每一个TaskManager就是一个独立的JVM进程，而每个Slot就会以这个进程中的一个线程执行。这些Slot在同一个任务中是共享的，一个Slot就足以贯穿应用的整个处理流程。Flink集群只需要关注一个任务内的最大并行数，提供足够的slot即可，而不用关注整个任务需要多少Slot。

程序运行时的parallelism管理有三个地方可以配置：

1、优先级最低的是在flinkconf.yaml文件中的parallelism.default这个属性，默认值是1。

2、优先级较高的是在提交任务时可以指定任务整体的并行度要求。这个并行度可以在提交任务的管理页面和命令行中添加。

3、优先级最高的是在程序中指定的并行度。在flink的应用程序中，几乎每一个分布式操作都可以定制单独的并行度。

Flink的计算功能非常强大，提供的应用API也非常丰富，可以分为三大部分：

DataStream API：是Flink中主要进行流计算的模块；

DataSet API：是Flink中主要进行批量计算的模块；

Table与SQL API：是Flink数据集提供类似于关系型数据的数据查询过滤等功能；

什么是Flink？特点是什么？

Flink是一个开源的流式计算框架，它具有低延迟、高吞吐量和高容错性的特点。

对于流处理，最需要关注的是低延迟和Exactly-once保证。而对于批处理，更为关注的是高吞吐、高效处理。所以在实现时通常会给出两套不同的实现方法。例如MapReduce就是专门进行批处理，Storm专注于处理流式数据。而Flink则是一个真正意义上的流批统一的计算框架。

典型的一些应用场景包括实时监控系统、推荐系统、日志分析系统等

Flink的核心概念有哪些？

答案：Flink的核心概念包括流（Stream）、转换操作（Transformation）、窗口（Window）、状态（State）和触发器（Trigger）等。

Flink的事件时间（Event Time）和处理时间（Processing Time）有什么区别？

答案：事件时间是事件实际发生的时间，处理时间是事件被处理的时间。事件时间用于处理乱序事件和水位线的生成，而处理时间用于实时处理和低延迟计算。

请解释一下Flink的Exactly-Once语义。

答案：Exactly-Once语义是指Flink保证结果的准确性，即每个事件都会被处理且只被处理一次，不会发生重复计算或数据丢失。

Flink的容错机制是如何实现的？

答案：Flink通过检查点（Checkpoint）机制实现容错，即定期将应用程序的状态保存到持久化存储中，并能够在发生故障时从最近的检查点恢复。

Flink支持哪些数据源和数据接收器？

答案：Flink支持多种数据源和数据接收器，包括Kafka、HDFS、HBase、JDBC、Elasticsearch等，并且提供了适配器和连接器来方便地与外部系统进行集成。

Flink的时间窗口有哪些类型？请解释其区别。

答案：Flink的时间窗口包括滚动窗口（Tumbling Window）、滑动窗口（Sliding Window）和会话窗口（Session Window）。滚动窗口是固定大小、不重叠的窗口；滑动窗口是固定大小、可以有重叠的窗口；会话窗口是根据事件之间的间隔划分的窗口。

Flink的状态管理方式有哪些？

答案：Flink的状态可以通过内存状态后端、文件系统状态后端或者远程状态后端进行管理。内存状态后端适用于低延迟和中等规模的状态，文件系统状态后端适用于大规模状态，远程状态后端适用于分布式场景。

Flink的水位线（Watermark）是什么？它的作用是什么？

答案：水位线是用于处理乱序事件的一种机制，它表示事件时间流的进展。水位线告知Flink事件的时间进展，以便触发窗口的计算和数据的处理。

Flink的容错机制对性能有什么影响？

答案：Flink的容错机制会引入一定的性能开销，包括状态快照的生成和恢复、数据重放等。但Flink通过优化和并行处理来降低这些开销，并保持高吞吐量和低延迟的特性。