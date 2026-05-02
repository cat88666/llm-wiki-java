# 11-nacos

所有者: junk01

**1. Nacos关键特征？**

1、服务注册发现

2、服务健康检查

3、动态配置服务

4、动态DNS服务

5、服务元数据管理

6、负载均衡策略

7、雪崩保护

8、AP可用性(CAP:C一致性 A可用性 P 分区容错性)

**2. Nacos注册中心？**

**服务注册**：Nacos Client会通过发送REST请求向Nacos Server注册自己服务，提供自身的元数据，比如ip地址、端口等信息。Nacos Server接收到注册请求后，就会把这些元数据信息存储在一个双层的内存Map中。

**服务心跳**：在服务注册后，Nacos Client会维护一个定时心跳来持续通知Nacos Server，说明服务一直处于可用状态，防止被剔除。默认5s发送一次心跳。

**服务同步**：Nacos Server集群之间会互相同步服务实例，用来保证服务信息的一致性。 leader raft

**服务发现**：服务消费者（Nacos Client）在调用服务提供者的服务时，会发送一个REST请求给Nacos Server，获取上面注册的服务清单，并且缓存在Nacos Client本地，同时会在Nacos Client本地开启一个定时任务定时拉取服务端最新的注册表信息更新到本地缓存

**服务健康检查**：Nacos Server会开启一个定时任务用来检查注册服务实例的健康情况，对于超过15s没有收到客户端心跳的实例会将它的healthy属性置为false(客户端服务发现时不会发现)，如果某个实例超过30秒没有收到心跳，直接剔除该实例(被剔除的实例如果恢复发送心跳则会重新注册)

**雪崩保护：** 保护阈值： 设置0-1之间的值 0.6

临时实例： spring.cloud.nacos.discovery.ephemeral=false, 当服务宕机了也不会从服务列表中剔除；

健康实例、 不健康实例；健康实例数/总实例数 < 保护阈值`1/2<0.6

结合负载均衡器 权重的机制， 设置的越大

**3 Nacos注册中心架构**

1、Nacos Server注册中心

2、Nacos client服务消费者Consumer（Ribbon）定时拉服务列表

3、Nacos Client服务提供者Provider

**1. Nacos配置中心使用**

Nacos 提供用于存储配置和其他元数据的 key/value 存储，为分布式系统中的外部化配置提供服务器端和客户端支持。使用 Spring Cloud Alibaba Nacos Config，可以在 Nacos Server 集中管理你 Spring Cloud 应用的外部属性配置。

1.维护性

2.时效性

3.安全性

**1.0 SpringCloud Config 对比三大优势**

- springcloud config大部分场景结合git 使用, 动态变更还需要依赖Spring Cloud Bus 消息总线来通过所有的客户端变化.
- springcloud config不提供可视化界面
- nacos config使用长轮询更新配置, 一旦配置有变动后，通知Provider的过程非常的迅速, 从速度上秒杀springcloud原来的config几条街,

**1.1 快速开始**

准备配置，nacos server中新建nacos-config.properties

Namespace：代表不同环境，如开发、测试、生产环境。

Group：代表某项目，如XX医疗项目、XX电商项目

DataId：每个项目下往往有若干个工程（微服务），每个配置集(DataId)是一个工程（微服务）的主配置文件

**1.2 搭建nacos-config服务**

通过 Nacos Server 和 spring-cloud-starter-alibaba-nacos-config 实现配置的动态变更

**1.3 Config相关配置**

Nacos 数据模型 Key 由三元组唯一确定, Namespace默认是空串，公共命名空间（public），分组默认是 DEFAULT_GROUP

- **支持配置的动态更新**

**ps：除了默认的配置文件， 其他dataId都要加上后缀**

- **支持profile粒度的配置**

spring-cloud-starter-alibaba-nacos-config 在加载配置的时候，不仅仅加载了以 dataid 为:

${spring.application.name}.${file-extension:properties}为前缀的基础配置，还加载了dataid为${spring.application.name}-${profile}.${file-extension:properties}的基础配置。

在日常开发中如果遇到多套环境下的不同配置，可以通过Spring 提供的${spring.profiles.active}这个配置项来配置。

spring.profiles.active=dev

profile 的配置文件 大于 默认配置的文件。 并且形成互补

**ps：只有默认的配置文件， 才会应用profile**

**支持自定义 namespace 的配置**

用于进行租户粒度的配置隔离。不同的命名空间下，可以存在相同的 Group 或 Data ID 的配置。Namespace 的常用场景之一是不同环境的配置的区分隔离，例如开发测试环境和生产环境的资源（如配置、服务）隔离等。

在没有明确指定 ${spring.cloud.nacos.config.namespace} 配置的情况下， 默认使用的是 Nacos 上 Public 这个namespace。如果需要使用自定义的命名空间，可以通过以下配置来实现：

spring.cloud.nacos.config.namespace=71bb9785-231f-4eca-b4dc-6be446e12ff8

**支持自定义 Group 的配置**

Group是组织配置的维度之一。通过一个有意义的字符串（如 Buy 或 Trade ）对配置集进行分组，从而区分 Data ID 相同的配置集。当您在 Nacos 上创建一个配置时，如果未填写配置分组的名称，则配置分组的名称默认采用 DEFAULT_GROUP 。配置分组的常见场景：不同的应用或组件使用了相同的配置类型，如 database_url 配置和 MQ_topic 配置。

在没有明确指定 ${spring.cloud.nacos.config.group} 配置的情况下，默认是DEFAULT_GROUP 。如果需要自定义自己的 Group，可以通过以下配置来实现：

spring.cloud.nacos.config.group=DEVELOP_GROUP

**支持自定义扩展的 Data Id 配置**

Data ID 是组织划分配置的维度之一。Data ID 通常用于组织划分系统的配置集。一个系统或者应用可以包含多个配置集，每个配置集都可以被一个有意义的名称标识。Data ID 通常采用类 Java 包（如 com.taobao.tc.refund.log.level）的命名规则保证全局唯一性。此命名规则非强制。

通过自定义扩展的 Data Id 配置，既可以解决多个应用间配置共享的问题，又可以支持一个应用有多个配置文件。

**1.4 配置的优先级**

Spring Cloud Alibaba Nacos Config 目前提供了三种配置能力从 Nacos 拉取相关的配置。

- A: 通过 spring.cloud.nacos.config.shared-configs 支持多个共享 Data Id 的配置
- B: 通过 spring.cloud.nacos.config.ext-config[n].data-id 的方式支持多个扩展 Data Id 的配置
- C: 通过内部相关规则(应用名、应用名+ Profile )自动生成相关的 Data Id 配置

当三种方式共同使用时，他们的一个优先级关系是:A < B < C

优先级从高到低：

1) nacos-config-product.yaml 精准配置 2) nacos-config.yaml 同工程不同环境的通用配置 3) ext-config: 不同工程 扩展配置 4) shared-dataids 不同工程通用配置

**1.5 @RefreshScope**

@Value注解可以获取到配置中心的值，但是无法动态感知修改后的值，需要利用@RefreshScope注解

@RestController @RefreshScope public class TestController { @Value("${common.age}") private String age; @GetMapping("/common") public String hello() { return age; }