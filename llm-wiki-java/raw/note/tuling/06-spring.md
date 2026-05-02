# 06-spring

所有者: junk01

## 一、Spring 启动整体流程

1. **Spring 是什么**
    - 快速开发框架，核心能力：**IOC（对象管理）**
    - 源码特点：设计模式多、并发安全、面向接口
2. **容器启动流程**
    1. **扫描组件**
        - 扫描包路径，生成 `BeanDefinition`
        - 存入 `BeanDefinitionMap`
    2. **创建 Bean**
        - 创建 **非懒加载的单例 Bean**
        - 多例 Bean：**获取时才创建**
    3. **Bean 生命周期**
        - 合并 BeanDefinition
        - 推断构造方法
        - 实例化
        - 属性填充
        - 初始化前 / 初始化 / 初始化后
        - **AOP 发生在初始化后**
    4. **发布容器启动事件**
    5. **Spring 启动完成**
3. **扩展点**
    - `BeanFactoryPostProcessor`
        - 修改 BeanDefinition
        - **@ComponentScan / @Import**
    - `BeanPostProcessor`
        - Bean 创建过程增强
        - **依赖注入 / AOP**

---

## 二、Bean 创建生命周期（重点）

**Bean 创建 7 步：**

1. 推断构造方法
2. 实例化
3. 属性填充（依赖注入）
4. Aware 接口回调
5. 初始化前（`@PostConstruct`）
6. 初始化（`InitializingBean`）
7. 初始化后（**AOP 代理生成**）

---

## 三、BeanFactory vs ApplicationContext

### BeanFactory

- Spring 最底层容器
- 负责：
    - Bean 的创建
    - Bean 的维护

### ApplicationContext

- **继承 BeanFactory**
- 额外能力：
    - `EnvironmentCapable`（环境变量）
    - `MessageSource`（国际化）
    - `ApplicationEventPublisher`（事件机制）

👉 **企业开发基本都用 ApplicationContext**

---

## 四、Spring AOP

### 1️⃣ AOP 是什么

- **AOP（面向切面编程）**
- 补充 OOP 的不足
- 关注点：日志 / 事务 / 权限 / 监控

### 2️⃣ AOP 的本质

> 在不修改原有代码的情况下
> 
> 
> **在方法执行的特定位置**
> 
> **增强与核心业务无关的公共逻辑**
> 

### 3️⃣ AOP 价值

- 解耦业务代码
- 提高可维护性
- 提供统一的横切逻辑处理

---

## 五、核心关键词速记（面试用）

- IOC：对象生命周期由 Spring 管理
- AOP：方法级增强（代理）
- 单例 Bean：启动创建
- 原型 Bean：获取时创建
- AOP 时机：**初始化后**
- 依赖注入：`BeanPostProcessor`

**SpringMVC的底层⼯作流程**

1. ⽤户发送请求⾄前端控制器DispatcherServlet。

2. DispatcherServlet 收到请求调⽤ HandlerMapping 处理器映射器。

3. 处理器映射器找到具体的处理器(可以根据 xml 配置、注解进⾏查找)，⽣成处理器及处理器拦截器

(如果有则⽣成)⼀并返回给 DispatcherServlet。

4. DispatcherServlet 调⽤ HandlerAdapter 处理器适配器。

5. HandlerAdapter 经过适配调⽤具体的处理器(Controller，也叫后端控制器)

6. Controller 执⾏完成返回 ModelAndView。

7. HandlerAdapter 将 controller 执⾏结果 ModelAndView 返回给 DispatcherServlet。

8. DispatcherServlet 将 ModelAndView 传给 ViewReslover 视图解析器。

9. ViewReslover 解析后返回具体 View。

10. DispatcherServlet 根据 View 进⾏渲染视图（即将模型数据填充⾄视图中）。

11. DispatcherServlet 响应⽤户。