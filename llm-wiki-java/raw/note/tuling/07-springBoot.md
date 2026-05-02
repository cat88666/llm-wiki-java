# 07-springBoot

所有者: junk01

# Spring Boot 核心总结（注解 / 启动 / 自动配置）

## 一、Spring Boot 常用注解 & 底层原理

### 1️⃣ `@SpringBootApplication`

**Spring Boot 启动类注解，本质是 3 个注解组合：**

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
```

- **@SpringBootConfiguration**
    - 等价于 `@Configuration`
    - 启动类本身也是一个配置类
- **@EnableAutoConfiguration（核心）**
    - 通过 `@Import(AutoConfigurationImportSelector.class)`
    - 从 `META-INF/spring.factories` 中加载自动配置类
    - 自动将配置类注册为 Bean
- **@ComponentScan**
    - 扫描组件
    - 默认扫描路径：**启动类所在包及其子包**

---

### 2️⃣ `@Bean`

- 作用：声明一个 Bean（替代 XML）
- 底层：
    - 启动时解析 `@Bean` 方法
    - **方法名 = beanName**
    - 调用方法返回对象作为 Bean 实例

---

### 3️⃣ 常见组件注解（简述）

- `@Controller` / `@Service`：语义化组件标识
- `@ResponseBody`：返回 JSON / 数据
- `@Autowired`：依赖注入（ByType / ByName）

---

## 二、Spring Boot 如何启动 Tomcat（内嵌）

1. 启动时先创建 **Spring 容器**
2. 自动配置阶段：
    - 使用 `@ConditionalOnClass`
    - 判断 classpath 是否存在 **Tomcat**
3. 若存在：
    - 创建 **Tomcat 启动相关 Bean**
4. 容器刷新完成后：
    - 创建 Tomcat 对象
    - 绑定端口
    - 启动 Tomcat

👉 **无需外置 Tomcat**

---

## 三、Spring Boot 配置文件加载顺序（高 → 低）

> 高优先级覆盖低优先级，最终形成互补配置
> 
1. 命令行参数
2. Java 系统属性（`System.getProperties()`）
3. 操作系统环境变量
4. jar 外 `application-{profile}.yml / properties`
5. jar 内 `application-{profile}.yml / properties`
6. jar 外 `application.yml / properties`
7. jar 内 `application.yml / properties`
8. `@Configuration` 类上的 `@PropertySource`

---

## 四、Spring Boot 如何通过 jar 启动

```bash
java -jar xxx.jar

```

启动原理：

1. jar 中包含 class 与资源文件
2. `MANIFEST.MF` 文件中指定：
    
    ```
    Main-Class
    Start-Class
    
    ```
    
3. `java -jar`：
    - 读取 `MANIFEST.MF`
    - 找到 `Start-Class`
    - 执行其 `main` 方法启动应用

---

## 五、Spring Boot 自动配置原理（重点）

### 核心入口

```java
@SpringBootApplication
→ @EnableAutoConfiguration
→ @Import(AutoConfigurationImportSelector.class)
```

### 自动配置流程

1. Spring 解析 `@Import`
2. 调用 `AutoConfigurationImportSelector`
3. 扫描所有 jar 包中的：
    
    ```
    META-INF/spring.factories
    ```
    
4. 加载其中声明的 **自动配置类**
5. 结合：
    - `@ConditionalOnClass`
    - `@ConditionalOnMissingBean`
6. 决定是否生效

👉 **“有则装配、无则跳过”**

---

## 六、关键词速记（面试版）

- 自动配置核心：`spring.factories`
- 核心机制：`@Import + Selector`
- Tomcat 启动：`@ConditionalOnClass`
- 默认扫描路径：启动类所在包
- jar 启动关键：`MANIFEST.MF`