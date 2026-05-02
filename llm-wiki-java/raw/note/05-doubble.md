# 05-doubble

所有者: junk01

### **一、Dubbo 支持的负载均衡策略**

**1. 随机（Random）：**默认策略、调用量越大越均匀、支持按权重随机

**2. 轮询（Round Robin）：**依次选择服务提供者、支持权重轮询、底层采用 **平滑加权轮询算法**

**3. 最小活跃数（Least Active）**

- 统计每个节点当前正在处理的请求数
- 选择“活跃数最小”的节点处理请求
- 适合节点处理能力差异大的场景

**4. 一致性 Hash（Consistent Hash）：**相同参数的请求总是落在同一个节点；高频用户、同会话绑定、游戏分区等场景常用

---

### **二、Dubbo 服务导出（Provider 端）流程**

**1. 解析 @DubboService 或 @Service 注解：**解析服务名、接口、协议、超时、版本等 → 得到 **ServiceBean**

**2. 调用 ServiceBean.export() 执行导出：**进入 Dubbo 的服务暴露流程

**3. 注册服务到注册中心（ZK / Nacos / Redis 等）：**如果有多个协议、多个注册中心 → 对每个协议、每个注册中心分别注册

**4. 绑定监听器：**监听动态配置中心的变更（如路由、限流、超时更新）

**5. 启动协议对应的网络服务器：**Dubbo 协议 → Netty；HTTP 协议 → Tomcat/Jetty

---

### **三、Dubbo 服务引入（Consumer 端）流程**

**1. 解析 @Reference 注解：**得到服务名、接口、负载均衡、超时配置等

**2. 从注册中心查询 Provider 列表：**将提供者列表封装到 **Directory（服务目录）**

**3. 注册监听器：**监听路由变化、动态配置、节点上下线

**4. 基于 Provider 列表创建代理对象：**通过 ProxyFactory 生成接口代理 → 注入到 Spring 使用

---

### **四、Dubbo 架构设计（分层结构）**

Dubbo 的架构非常模块化，各层可灵活替换。

## 1. **Config 配置层**

- ServiceConfig / ReferenceConfig
- 接收 Spring 或 XML 的配置，初始化全局参数

## 2. **Proxy 服务代理层**

- JDK 动态代理 / Javassist
- 将 **Invoker 转成接口代理**
- 也可以将接口实现转换为 Invoker

## 3. **Registry 注册中心层**

- 支持 ZooKeeper、Redis、Nacos
- 管理 **服务注册、服务发现、订阅、通知**

## 4. **Cluster 集群容错 / 路由层**

- Failover、Failfast、Failsafe、Forking 等
- 负载均衡
- 将多个提供者伪装成一个 Invoker

关键组件：

- **Cluster**
- **Directory**
- **Router**
- **LoadBalance**

## 5. **Monitor 监控层**

- 统计调用次数、耗时
- 类似 Prometheus / Micrometer

## 6. **Protocol 远程调用层（核心）**

核心接口：

- **Protocol**
- **Invoker**
- **Exporter**

> 这是 Dubbo RPC 的核心，只要这三样就能完成 RPC 调用。
> 

---

## 7. **Exchange 信息交换层（请求响应模型）**

- 封装 request / response 语义
- 同步转异步调用

核心：

- **Exchanger**
- **ExchangeChannel**
- **ExchangeClient**
- **ExchangeServer**

## 8. **Transport 网络传输层**

统一封装网络框架：

- Netty
- Mina
- Grizzly

核心：

- **Transporter**
- **Client**
- **Server**
- **Codec**

## 9. **Serialize 序列化层**

支持序列化方式：

- Hessian2
- JSON
- FastJson2
- Kryo
- FST

核心：

- **Serialization**
- **ObjectInput**
- **ObjectOutput**

---

### **五、层之间的关系**

**RPC 调用只需要三件套：**

1. **Protocol**
2. **Invoker**
3. **Exporter**

其他都是增强功能。

**Cluster 层**

- 将多个 Invoker 聚合成一个
- 让调用端感觉“只有 1 个提供者”

**Proxy 层**

- 将 Invoker 转换成一个本地 Java 接口，让调用看起来像本地调用

**Transport / Exchange**

- 底层框架是可替换的（Netty/Mina）
- Exchange 层实现请求–响应语义