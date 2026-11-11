# Spring / Spring Boot（20 题）

> 适用：后端 / AI 应用开发实习。Spring 是 Java 后端的默认答案，面试问的是「你知不知道它怎么转起来的」，不是背注解。

---

## 一、IoC 与 Bean

### Q1 ★ 说说你对 IoC 和 DI 的理解

**一句话答案：** IoC 是控制反转——对象的创建和依赖装配从「业务代码 new」交给容器；DI 是它的实现手段，通过构造器/Setter/字段把依赖注入进来。

**详细解析：**
- 好处：解耦（面向接口）、可测试（注入 mock）、生命周期统一管理（作用域、销毁）。
- 三种注入方式：构造器（推荐，不可变、必填依赖）、Setter（可选依赖）、字段（`@Autowired`，方便但不好测）。
- 循环依赖：构造器注入无法解决，字段注入可以（靠三级缓存）。
- 配合 `@Qualifier` / `@Primary` 解决同类型多 Bean。

**追问：** 为什么推荐构造器注入？→ 依赖不可变、启动即暴露缺失依赖、单测不用启动容器。

### Q2 ★ Bean 的生命周期

**一句话答案：** 实例化 → 属性填充 → Aware 回调 → BeanPostProcessor 前置 → 初始化（`@PostConstruct` → `InitializingBean` → `init-method`）→ BeanPostProcessor 后置（AOP 代理在这里生成）→ 使用 → 销毁（`@PreDestroy` → `DisposableBean`）。

**详细解析：**
- 实例化：反射调用构造器。
- 属性填充：`populateBean`，处理 `@Autowired`。
- Aware：`BeanNameAware`、`BeanFactoryAware`、`ApplicationContextAware`。
- 初始化前后的 `BeanPostProcessor` 是扩展点，AOP 动态代理在 `postProcessAfterInitialization` 生成。
- 单例 Bean 在容器启动时就创建（默认非懒加载）。

**追问：** `@PostConstruct` 和 `afterPropertiesSet` 谁先？→ `@PostConstruct` 先（CommonAnnotationBeanPostProcessor 处理）。

### Q3 ★ Spring 如何解决循环依赖？为什么需要三级缓存？

**一句话答案：** 用三级缓存：一级放成品 Bean、二级放早期引用（已实例化未填充）、三级放对象工厂（用于生成代理）。A 依赖 B、B 依赖 A 时，A 实例化后把工厂放入三级缓存，B 创建时从三级缓存拿到 A 的早期引用完成注入。

**详细解析：**
- 一级 `singletonObjects`：完整 Bean。
- 二级 `earlySingletonObjects`：提前暴露的引用，避免重复调用工厂。
- 三级 `singletonFactories`：`ObjectFactory`，在需要时才生成代理（AOP 场景必须提前代理，否则 B 注入的是原始对象）。
- 只支持单例 + 字段/Setter 注入；构造器注入循环依赖直接报错；原型 Bean 不支持。

**追问：** 为什么二级缓存不够？→ 如果 A 需要被 AOP 代理，B 注入的必须是代理对象；只有工厂能在「需要时」决定返回原对象还是代理，所以要有三级。

**坑：** Spring Boot 2.6+ 默认禁止循环依赖（`allow-circular-references=false`），正确做法是重构依赖，而不是打开开关。

## 二、AOP 与事务

### Q4 ★ AOP 的实现原理，JDK 动态代理与 CGLIB 的区别

**一句话答案：** 运行时生成代理对象，把横切逻辑织入。JDK 代理基于接口（`InvocationHandler` + `Proxy`），CGLIB 基于继承（生成子类、字节码增强）。Spring 默认：有接口用 JDK，无接口用 CGLIB（Boot 2.x 起默认 `proxyTargetClass=true` 全用 CGLIB）。

**详细解析：**
- 通知类型：前置、后置、返回、异常、环绕（`@Around` 最强，可改参数和返回值）。
- 切点表达式：`execution(* com.xx.service.*.*(..))`、`@annotation(...)`。
- CGLIB 不能代理 final 类和 final 方法；JDK 代理只能代理接口方法。
- 代理对象是 Bean 被注入的实际类型，所以 `this` 调用不走代理（事务失效的根源）。

**追问：** 为什么 `@Transactional` 在同类方法自调用时失效？→ 自调用走的是原始对象的方法，不经过代理，切面不生效。

### Q5 ★ 事务的传播行为有哪些？重点说 REQUIRED 和 REQUIRES_NEW

**一句话答案：** REQUIRED（默认，有事务就加入、没有就新建）、REQUIRES_NEW（挂起外层，新建独立事务）、NESTED（嵌套，用保存点）、SUPPORTS、NOT_SUPPORTED、MANDATORY、NEVER。

**详细解析：**
- REQUIRED：内外层同一个事务，内层抛异常会让外层一起回滚（除非被 catch）。
- REQUIRES_NEW：两个独立事务，内层回滚不影响外层；注意会占两个数据库连接，容易死锁。
- NESTED：同一个物理事务 + 保存点，外层回滚会带上内层，内层回滚不影响外层（JDBC 需支持保存点）。
- 典型场景：主流程 + 记日志，日志用 REQUIRES_NEW 保证主流程回滚时日志还在。

**追问：** 为什么 `REQUIRES_NEW` 可能死锁？→ 外层持有连接等待内层，内层需要新连接，连接池耗尽时互等。

### Q6 ★ 事务失效的几种情况

**一句话答案：** 自调用、方法非 public、异常被 catch、抛的是受检异常（默认只回滚 RuntimeException）、类没被 Spring 管理、数据库引擎不支持事务、多线程调用。

**详细解析：**
- 自调用：`this.methodB()` 不走代理 → 注入自身代理或拆到另一个 Bean。
- 非 public：CGLIB 能代理 protected，但 Spring 的事务拦截器只对 public 生效。
- 异常：`@Transactional(rollbackFor = Exception.class)` 覆盖受检异常。
- 多线程：事务绑定在 ThreadLocal 的连接上，子线程不在同一事务。
- 只读事务 `readOnly=true` 下写操作可能被忽略（取决于驱动）。

**追问：** 如何让自调用也走事务？→ `AopContext.currentProxy()`（需开 `exposeProxy`）、注入自身、或把方法拆到独立 Bean。

## 三、Spring Boot

### Q7 ★ Spring Boot 自动装配的原理

**一句话答案：** `@SpringBootApplication` 包含 `@EnableAutoConfiguration`，它通过 `AutoConfigurationImportSelector` 读取 `META-INF/spring/…AutoConfiguration.imports`（2.7 前是 `spring.factories`），结合 `@Conditional` 系列注解按需装载配置类。

**详细解析：**
- 条件装配：`@ConditionalOnClass`（类路径有才生效）、`@ConditionalOnMissingBean`（用户没定义才生效）、`@ConditionalOnProperty`。
- 顺序：`@AutoConfigureBefore/After` 控制加载顺序。
- 自定义 Starter：写一个 `@Configuration` + 条件注解，注册到 imports 文件，业务引入依赖即生效。
- 排除：`@SpringBootApplication(exclude = XxxAutoConfiguration.class)`。

**追问：** 为什么自己的 Bean 能覆盖 Starter 的默认 Bean？→ `@ConditionalOnMissingBean`，容器里已有同类型 Bean 时默认配置放弃。

### Q8 Spring Boot 的启动流程

**一句话答案：** 创建 `SpringApplication` → 加载监听器与初始化器 → 准备环境（配置文件、命令行参数）→ 创建 `ApplicationContext` → `refresh()`（BeanFactory 准备、Bean 定义加载、实例化单例、启动内嵌容器）→ 发布 `ApplicationReadyEvent`。

**详细解析：**
- `refresh()` 的核心步骤来自 `AbstractApplicationContext`：`invokeBeanFactoryPostProcessors`（解析配置类）→ `registerBeanPostProcessors` → `finishBeanFactoryInitialization`（实例化所有非懒加载单例）→ `finishRefresh`。
- 内嵌 Tomcat 在 `onRefresh` 阶段启动。
- 启动扩展点：`ApplicationRunner` / `CommandLineRunner` 在容器就绪后执行。

**追问：** 启动慢怎么排查？→ 开 `debug` 看自动配置报告、`--debug` 或 Actuator 的 `startup` 端点，重点看慢 Bean 与数据库连接。

### Q9 配置文件的优先级

**一句话答案：** 命令行参数 > Java 系统属性 > 环境变量 > `application-{profile}.yml` > `application.yml` > `@PropertySource` > 默认值。

**详细解析：**
- 外部配置：`--spring.config.location` 指定目录，优先级高于 jar 内。
- Profile：`spring.profiles.active` 激活，`application-prod.yml` 覆盖基础配置。
- 配置中心（Nacos/Apollo）通常优先级高于本地文件。

**追问：** 为什么生产环境要用环境变量注入密码？→ 避免密钥进代码仓库，配合 K8s Secret / 配置中心做权限控制。

### Q10 Bean 的作用域

**一句话答案：** singleton（默认）、prototype、request、session、application、websocket。

**详细解析：**
- singleton 在容器启动时创建，整个容器一个实例。
- prototype 每次 `getBean` 新建，Spring 不负责销毁。
- request/session 只在 Web 环境有效，靠 `RequestContextHolder` 存作用域。
- 单例注入原型：原型也只会被注入一次，要用 `ObjectProvider` 或方法注入。

**追问：** 单例 Bean 是线程安全的吗？→ 不是，Spring 不管线程安全，有状态字段要自己加锁或用无状态设计。

## 四、Web 与工程实践

### Q11 ★ 过滤器、拦截器、AOP 的区别与执行顺序

**一句话答案：** Filter 是 Servlet 规范（最外层，能改请求/响应，连静态资源都拦）；Interceptor 是 Spring MVC 的（能拿到 Handler 和方法信息）；AOP 是方法级（能拿参数、返回值、异常）。顺序：Filter → Interceptor → AOP → Controller。

**详细解析：**
- Filter：`doFilter` 链，适合编码、跨域、请求包装、日志埋点。
- Interceptor：`preHandle`（返回 false 中断）、`postHandle`、`afterCompletion`（异常也会执行，适合清理 ThreadLocal）。
- AOP：适合事务、限流、权限、日志，能拿到方法签名和参数。
- 鉴权一般放 Filter（如 Spring Security）或 Interceptor（轻量方案）。

**追问：** 为什么 ThreadLocal 要在 `afterCompletion` 清理？→ 请求异常时 `postHandle` 不执行，只有 `afterCompletion` 一定执行。

### Q12 Spring MVC 处理一个请求的流程

**一句话答案：** 请求 → `DispatcherServlet` → `HandlerMapping` 找到 Handler 和拦截器链 → `HandlerAdapter` 调用 → 参数解析（`HandlerMethodArgumentResolver`）→ 执行 Controller → 返回值处理（`HttpMessageConverter` 序列化）→ 响应。

**详细解析：**
- 常见问题：404 是没找到 Handler、415 是 Content-Type 不匹配、400 是参数绑定失败。
- `@RequestBody` 走 `HttpMessageConverter`（Jackson），`@RequestParam` 走参数解析器。
- 统一响应：`ResponseBodyAdvice` 或直接返回包装对象。

**追问：** 全局异常处理用哪个注解？→ `@RestControllerAdvice` + `@ExceptionHandler`，优先级按异常类型的精确度匹配。

### Q13 参数校验怎么做？

**一句话答案：** `@Valid` / `@Validated` + JSR-303 注解（`@NotNull`、`@Size`、`@Pattern`）+ 全局异常处理捕获 `MethodArgumentNotValidException`。

**详细解析：**
- `@Validated` 支持分组校验和类级别校验（方法参数），`@Valid` 支持嵌套对象。
- 自定义校验：实现 `ConstraintValidator`。
- 嵌套对象要加 `@Valid` 才会递归校验。

**追问：** 为什么服务端必须校验？→ 前端可绕过，接口一旦暴露就是攻击面。

### Q14 `@Async` 的坑

**一句话答案：** 自调用失效、默认线程池无界可能 OOM、异常被吞掉、事务不传递、返回值必须是 `Future/CompletableFuture`。

**详细解析：**
- 必须配自定义 `ThreadPoolTaskExecutor`，指定队列容量和拒绝策略。
- 异常：未捕获的异常会被 `AsyncUncaughtExceptionHandler` 处理，默认只打日志。
- 事务：异步方法在另一个线程，不在调用方事务里。
- 上下文：ThreadLocal（含登录信息）不会自动传递。

**追问：** 怎么让异步任务也带登录上下文？→ 手动把关键字段作为参数传入，或用可传递的 ThreadLocal（TTL）。

### Q15 Spring 事件机制与事务事件

**一句话答案：** `ApplicationEventPublisher` 发布、`@EventListener` 监听，默认同步；`@TransactionalEventListener` 可以绑定到事务提交后执行（`AFTER_COMMIT`）。

**详细解析：**
- 用途：解耦主流程与副作用（发通知、写日志、清缓存）。
- `@Async` + `@EventListener` 可异步处理，但要注意异常隔离。
- 事务事件解决「事务还没提交就发消息，消费者查不到数据」的经典问题。

**追问：** 为什么发 MQ 消息要放在事务提交后？→ 否则消息先被消费，此时数据库还没提交，消费方查不到数据。

### Q16 Spring AI 是怎么接入大模型的？

**一句话答案：** 提供统一抽象：`ChatClient`（对话）、`EmbeddingModel`（向量化）、`VectorStore`（向量检索），底层用 OpenAI 兼容协议对接各家模型；Spring Boot Starter 自动装配这些 Bean。

**详细解析：**
- `ChatClient.prompt().user(...).stream()` 返回 `Flux<String>`，可直接对接 SSE。
- `VectorStore.similaritySearch(SearchRequest)` 支持 topK、相似度阈值、元数据过滤表达式。
- 结构化输出：`BeanOutputConverter` 把 DTO 转成格式说明，再用 `ChatClient` 的 `entity()` 反序列化。
- 多模型：按用户动态构建 `OpenAiChatModel` 并缓存（项目里的做法）。

**追问：** Spring AI 和 LangChain 的区别？→ Spring AI 贴 Spring 生态、Java 友好；LangChain 生态更大、Agent 编排能力更强，Python 版更成熟。

### Q17 如何自定义一个 Starter？

**一句话答案：** 建两个模块：`xxx-spring-boot-autoconfigure`（配置类 + 条件注解）和 `xxx-spring-boot-starter`（只依赖前者）；配置类注册到 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`。

**详细解析：**
- 配置类用 `@AutoConfiguration` + `@ConditionalOnClass` + `@ConditionalOnMissingBean`。
- 配置项用 `@ConfigurationProperties(prefix = "xxx")` + `@EnableConfigurationProperties`。
- 命名规范：第三方库 `xxx-spring-boot-starter`，官方 `spring-boot-starter-xxx`。

**追问：** `@ConfigurationProperties` 需要 `@Component` 吗？→ 不需要，`@EnableConfigurationProperties` 或 `@ConfigurationPropertiesScan` 会注册。

### Q18 BeanFactory 和 ApplicationContext 的区别

**一句话答案：** BeanFactory 是最底层的 IoC 容器（懒加载、功能少）；ApplicationContext 是它的增强（立即加载、国际化、事件、资源加载、AOP 集成）。

**详细解析：**
- 实际开发都用 ApplicationContext，BeanFactory 基本只出现在面试题里。
- `getBean` 时若 Bean 不存在，前者抛 `NoSuchBeanDefinitionException`，后者行为一致但有更多扩展点。

**追问：** 为什么说 ApplicationContext 启动慢？→ 默认预实例化所有单例 Bean。

### Q19 如何实现接口幂等？

**一句话答案：** 唯一索引兜底、幂等 Token（先申请后校验）、状态机（只允许从某状态流转一次）、分布式锁 + 业务 ID 去重表、Redis `SETNX` 记录请求 ID。

**详细解析：**
- HTTP 层：GET/PUT/DELETE 天然幂等，POST 不幂等。
- 支付/下单：用业务单号做唯一键，重复插入直接返回已有结果。
- 消息消费：用消息 ID 去重表或 Redis 记录已处理。

**追问：** 幂等和防重有什么区别？→ 幂等是「重复执行结果一致」，防重是「阻止重复执行」；防重可以靠锁，幂等更彻底。

### Q20 怎么设计一个统一返回与异常处理？

**一句话答案：** 统一返回体 `{code, message, data}`；`@RestControllerAdvice` 捕获业务异常、参数校验异常、兜底 Exception；HTTP 状态码要么统一 200 用业务码区分，要么按语义返回并配全局映射。

**详细解析：**
- 业务异常继承 RuntimeException，带错误码枚举。
- 参数异常转成可读提示，别把堆栈抛给前端。
- 兜底异常要打日志（含 traceId），否则线上无法定位。
- 前端拦截器按 code 统一处理登录失效、提示。

**追问：** 为什么很多公司返回 HTTP 200 + 业务码？→ 前端处理简单、避免网关/代理对 4xx/5xx 的特殊处理；缺点是监控失真。两种都能用，团队统一即可。

---

## 高频速记

| 主题 | 必须脱口而出 |
| --- | --- |
| 循环依赖 | 三级缓存：成品 / 早期引用 / 对象工厂 |
| AOP | JDK 接口代理 vs CGLIB 子类；Boot 默认 CGLIB |
| 事务失效 | 自调用、非 public、异常被吞、多线程 |
| 自动装配 | `@EnableAutoConfiguration` + imports 文件 + 条件注解 |
| 请求链路 | Filter → Interceptor → AOP → Controller → MessageConverter |
| 事件 | `@TransactionalEventListener(AFTER_COMMIT)` 解决消息先发后提交 |
| Spring AI | ChatClient / EmbeddingModel / VectorStore 三大抽象 |
