# Spring Boot基础笔记

## 一、什么是Spring Boot

**定义** **Spring Boot=基于Spring框架的快速开发脚手架，通过约定优于配置的理念，简化Spring应用的初始搭建和开发过程。**

Spring Boot 是由 Pivotal 团队提供的全新框架，其设计目的是用来简化 Spring 应用的初始搭建以及开发过程。该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。

简单来说，Spring Boot 不是什么新的框架，而是把常用的框架和技术整合在一起，默认配置好了大部分参数，让开发者可以快速搭建一个 Spring 项目，开箱即用。

### 生活化比喻

如果把开发一个 Web 应用比作开一家奶茶店：

- **传统 Spring 框架** 就像是给你一堆原材料（茶叶、牛奶、糖、杯子、封口机、收银台...），你需要自己一个个去采购、自己组装设备、自己调配配方，什么都得自己来，非常繁琐
- **Spring Boot** 就像是一个**奶茶店加盟套餐**，设备都给你配好了，配方也调好了，你只要选好口味（starter），直接就能开店，大大节省了准备时间

Spring Boot 的核心就是**"约定优于配置"**——大家都这么用的配置，我默认就给你配好，你不用再写一遍；只有特殊需求的时候才需要自己改配置。

### Spring Boot 的核心特点

1. **开箱即用**：提供各种 starter 依赖，简化构建配置
2. **自动配置**：自动配置 Spring 和第三方库
3. **内嵌容器**：内嵌 Tomcat、Jetty 等容器，无需部署 WAR 包
4. **无代码生成**：不需要生成代码，纯配置实现
5. **起步依赖**：starter 依赖自动管理版本，避免版本冲突

## 二、核心机制

### Starters 机制

**定义** **Starters=Spring Boot提供的依赖描述符，将特定功能所需的所有依赖打包在一起，开发者只需引入一个starter就能获得完整的功能支持。**

Starters 是 Spring Boot 最核心的设计之一。在传统的 Spring 开发中，要使用某个功能（比如 Web 开发），我们需要手动引入一堆依赖，还要注意版本兼容问题，非常麻烦。

而 Spring Boot Starters 把某个功能所需的所有依赖都打包好了，你只要引入一个 starter，相关的依赖就都自动引进来了，而且版本都是经过测试兼容的。

比喻：就像快餐店的"套餐"——你点一个"全家桶套餐"，里面包含了汉堡、薯条、可乐、鸡翅...不用你一个个单点，而且搭配都是经过设计的，不会出现"点了汉堡忘了点饮料"的情况。

#### Starter 的命名规范

所有官方 starter 都遵循统一的命名模式：**`spring-boot-starter-*`**，其中 `*` 表示特定的应用类型。

例如：
- `spring-boot-starter-web`：Web 开发的 starter
- `spring-boot-starter-data-jpa`：JPA 数据库访问的 starter
- `spring-boot-starter-test`：测试相关的 starter

**第三方 starter** 命名规则不同，以项目名开头：`{项目名}-spring-boot-starter`。例如第三方项目叫 `abc`，那么它的 starter 就是 `abc-spring-boot-starter`。

#### 常用 Starter 列表

**应用开发类：**

| Starter 名称 | 功能说明 |
|-------------|---------|
| `spring-boot-starter-web` | 构建Web应用，包含Spring MVC，默认使用Tomcat |
| `spring-boot-starter-data-jpa` | 使用Spring Data JPA + Hibernate进行数据库操作 |
| `spring-boot-starter-data-redis` | Redis键值存储，包含Spring Data Redis和Jedis客户端 |
| `spring-boot-starter-data-mongodb` | MongoDB文档数据库，包含Spring Data MongoDB |
| `spring-boot-starter-security` | Spring Security安全认证框架 |
| `spring-boot-starter-thymeleaf` | 使用Thymeleaf模板引擎构建MVC Web应用 |
| `spring-boot-starter-mail` | 邮件发送支持，包含Java Mail |
| `spring-boot-starter-amqp` | Spring AMQP和RabbitMQ消息队列 |
| `spring-boot-starter-activemq` | Apache ActiveMQ消息中间件 |
| `spring-boot-starter-cache` | Spring Framework缓存支持 |
| `spring-boot-starter-validation` | Hibernate Validator数据校验 |
| `spring-boot-starter-aop` | 面向切面编程，包含Spring AOP和AspectJ |
| `spring-boot-starter-batch` | Spring Batch批处理 |

**技术支撑类：**

| Starter 名称 | 功能说明 |
|-------------|---------|
| `spring-boot-starter` | 核心starter，包含自动配置、日志、YAML支持 |
| `spring-boot-starter-test` | 测试支持，包含JUnit、Hamcrest、Mockito |
| `spring-boot-starter-tomcat` | Tomcat嵌入式Servlet容器（web默认使用） |
| `spring-boot-starter-jetty` | Jetty嵌入式Servlet容器（替换Tomcat） |
| `spring-boot-starter-undertow` | Undertow嵌入式Servlet容器（替换Tomcat） |
| `spring-boot-starter-logging` | Logback日志（默认日志实现） |
| `spring-boot-starter-log4j2` | Log4j2日志（替换默认日志） |

**生产运维类：**

| Starter 名称 | 功能说明 |
|-------------|---------|
| `spring-boot-starter-actuator` | 生产级监控和管理功能 |

#### Starter 的工作原理

1. **依赖传递**：每个 starter 内部都声明了该功能所需的所有依赖，引入 starter 时会自动传递引入所有相关依赖
2. **版本管理**：Spring Boot 统一管理所有依赖的版本，保证兼容性，开发者不需要指定版本号
3. **自动配置**：starter 配合 `@EnableAutoConfiguration` 注解，根据类路径中的依赖自动配置相应的 Bean

```xml
<!-- 只需引入这一个依赖，就能进行Web开发 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

这段代码背后，Spring Boot 自动帮你引入了 Spring MVC、Tomcat、Jackson 等一堆依赖，而且版本都是匹配好的。

### DevTools 热部署机制

**定义** **DevTools=Spring Boot提供的开发者工具模块，支持自动重启、LiveReload等功能，缩短开发时的等待时间。**

DevTools 是 Spring Boot 1.3 版本推出的开发者工具模块，全称是 Developer Tools。它的主要作用是在开发过程中，当代码发生变化时，自动重新加载应用，省去手动重启的麻烦，大大提高开发效率。

比喻：就像用手机拍照，以前拍一张要等很久才能看到效果（手动重启）；现在有了即时预览功能（DevTools），你改个姿势马上就能看到效果，拍照效率高多了。

#### DevTools 的五大功能

**1. 属性默认值**

Spring Boot 提供的模板技术（如 Thymeleaf、Freemarker）默认会开启缓存，开发时修改页面后需要重启才能看到效果。使用 DevTools 后，这些模板的缓存会自动禁用，修改页面后刷新就能看到效果。

可以通过配置 `spring.devtools.add-properties=false` 来关闭这个默认行为。

**2. 自动重启（Auto Restart）**

当类路径下的文件发生变化时，DevTools 会自动重启应用。这是 DevTools 最核心的功能。

**双 ClassLoader 机制：**
- **基础 ClassLoader（base ClassLoader）**：加载不变的类（如第三方 jar 包中的类）
- **重启 ClassLoader（restart ClassLoader）**：加载我们正在开发的类

重启时，只丢弃重启 ClassLoader，重新创建一个新的加载开发中的类，而基础 ClassLoader 保持不变。这样比完全重启快很多，因为第三方 jar 不需要重新加载。

比喻：就像换衣服，基础 ClassLoader 是衣柜（不动），重启 ClassLoader 是身上穿的衣服（换掉）。换衣服比重新买一套衣服快多了。

可以通过 `spring.devtools.restart.enabled=false` 禁用自动重启。

**3. LiveReload（自动刷新浏览器）**

DevTools 内置了一个 LiveReload 服务器，当资源文件（HTML、CSS、JS 等）发生变化时，可以自动触发浏览器刷新。

使用方法：
- 在 Chrome/Firefox/Safari 浏览器中安装 LiveReload 扩展
- 确保 DevTools 已启用（默认启用）
- 启动应用后，点击浏览器中的 LiveReload 图标开启

监听的路径包括：
- `/META-INF/maven`
- `/META-INF/resources`
- `/resources`
- `/static`
- `/public`
- `/templates`

可以通过 `spring.devtools.livereload.enabled=false` 禁用 LiveReload。

**4. 远程调试隧道**

Spring Boot 可以通过 HTTP 将 JDWP（Java 调试线协议）直接隧道到应用程序。即使应用部署在只暴露 80 和 443 端口的云服务上，也能进行远程调试。

**5. 远程更新和重启**

DevTools 支持远程应用的更新和重启。它监控本地类路径中的文件变化，将变化推送到远程服务器，然后触发远程重启。还可以和 LiveReload 结合使用。

#### 使用触发文件

由于频繁重启可能会影响开发体验，DevTools 支持使用**触发文件**（Trigger File）的方式。只有当触发文件被修改时，才会触发重启，避免每次保存都重启。

配置方式：
```properties
# 指定触发文件路径
spring.devtools.restart.trigger-file=c:/workspace/restart-trigger.txt
```

比喻：就像电梯的按钮，你准备好了再按一下（修改触发文件），电梯才动（重启），而不是人一靠近就动。

## 三、使用步骤

### DevTools 使用步骤

**步骤1：添加依赖**

在 `pom.xml` 中添加 `spring-boot-devtools` 依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
</dependency>
```

注意：scope 设置为 `runtime`，表示只在运行时生效，打包时不会包含进去，避免生产环境使用。

**步骤2：IDEA 配置（可选但推荐）**

如果使用 IntelliJ IDEA，需要开启自动编译：
1. 打开 Settings → Build, Execution, Deployment → Compiler
2. 勾选 "Build project automatically"
3. 按 `Ctrl+Shift+Alt+/`，选择 Registry
4. 勾选 `compiler.automake.allow.when.app.running`

**步骤3：常用配置**

```properties
# 禁用自动重启
# spring.devtools.restart.enabled=false

# 排除某些路径不触发重启
spring.devtools.restart.exclude=public/**, static/**, templates/**

# 添加额外的监听路径
# spring.devtools.restart.additional-paths=/path-to-folder

# 排除额外路径（保留默认排除项）
# spring.devtools.restart.additional-exclude=styles/**

# 禁用LiveReload
# spring.devtools.livereload.enabled=false

# 使用触发文件
# spring.devtools.restart.trigger-file=restart-trigger.txt
```

**步骤4：验证效果**

启动应用后，修改任意一个 Java 文件并保存，观察控制台输出，如果看到应用自动重启的日志，说明 DevTools 生效了。

### 打包方式

**定义** **打包=将Java应用程序及其依赖打包成一个可分发的归档文件（JAR/WAR/EAR），方便部署和运行。**

在 Java EE 应用中，主要有三种打包格式：JAR、WAR、EAR，它们都是压缩文件格式，但用途和结构不同。

比喻：就像不同的包装方式——
- **JAR** 是普通快递盒，装一般的东西（普通Java程序）
- **WAR** 是带保鲜功能的专用包装箱，装需要冷藏的东西（Web应用）
- **EAR** 是集装箱，里面可以装多个普通盒子和保鲜盒子（企业级应用，包含多个模块）

#### JAR 包

**定义** **JAR（Java Archive）=Java归档文件，用于打包Java类文件、资源文件和元数据，是最基础的打包格式。**

JAR 文件以 `.jar` 为后缀，本质上是一个 ZIP 格式的压缩文件。它包含：
- 类文件（.class 文件）
- 清单文件（META-INF/MANIFEST.MF）
- 资源文件（配置文件、图片等）

**用途：**
- 打包普通 Java 应用程序
- 打包 EJB 模块
- 作为类库供其他程序引用

**Spring Boot 默认打包方式就是 JAR**，因为 Spring Boot 内置了 Tomcat，直接 `java -jar xxx.jar` 就能运行。

```xml
<!-- pom.xml 中指定打包方式为jar（默认就是jar） -->
<packaging>jar</packaging>
```

运行方式：
```bash
java -jar application.jar
```

#### WAR 包

**定义** **WAR（Web Archive）=Web应用归档文件，专门用于打包Web应用，包含Servlet、JSP、HTML等Web资源。**

WAR 文件以 `.war` 为后缀，它是一种特殊的 JAR 文件，专门用于 Web 应用。

**结构特点：**
- 包含 `WEB-INF` 目录
- `WEB-INF/web.xml`：Web 应用配置文件
- `WEB-INF/classes`：编译后的类文件
- `WEB-INF/lib`：依赖的 jar 包
- 根目录下可以放 JSP、HTML、CSS、JS 等静态资源

**用途：** 部署到 Web 容器（如 Tomcat、Jetty、WebLogic）中运行。

比喻：WAR 包就像一个标准化的"餐厅加盟店"，里面厨房、菜单、装修都配好了，只要放到一个"商场"（Web容器）里就能营业。

Spring Boot 项目如果要打成 WAR 包，需要做以下配置：

```xml
<!-- 1. 修改打包方式 -->
<packaging>war</packaging>

<!-- 2. 将Tomcat依赖改为provided -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
</dependency>
```

```java
// 3. 启动类继承SpringBootServletInitializer，重写configure方法
@SpringBootApplication
public class Application extends SpringBootServletInitializer {
    
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(Application.class);
    }
    
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

#### EAR 包

**定义** **EAR（Enterprise Archive）=企业级归档文件，用于打包完整的企业级应用，可以包含多个EJB模块（JAR）和Web模块（WAR）。**

EAR 文件以 `.ear` 为后缀，是最高级别的打包格式，用于企业级应用。

**结构特点：**
- 包含 `META-INF/application.xml`：企业应用配置文件
- 可以包含多个 JAR 模块（EJB）
- 可以包含多个 WAR 模块（Web应用）

**用途：** 部署到 Java EE 应用服务器（如 JBoss、WebSphere、WebLogic）中。

比喻：EAR 就像一个"商业综合体"，里面有餐厅（WAR模块）、办公室（EJB模块）、商店等，整个综合体需要放到一个大型商圈（应用服务器）里才能运营。

#### 三种打包方式对比

| 对比项 | JAR | WAR | EAR |
|-------|-----|-----|-----|
| 后缀 | .jar | .war | .ear |
| 用途 | 普通Java应用、类库 | Web应用 | 企业级应用 |
| 包含内容 | 类文件、资源 | Web资源+类文件 | 多个JAR和WAR模块 |
| 特殊目录 | META-INF | WEB-INF | META-INF/application.xml |
| 运行环境 | 直接运行或作为依赖 | Web容器（Tomcat等） | 应用服务器（JBoss等） |
| Spring Boot | 默认方式 | 需要额外配置 | 很少使用 |

## 四、为什么需要

### 为什么需要 Spring Boot

传统 Spring 开发存在以下痛点：

1. **配置繁琐**：每个项目都要写一堆 XML 配置文件，配置各种 Bean、数据源、事务...
2. **依赖管理复杂**：要手动引入很多依赖，还要解决版本冲突问题
3. **部署麻烦**：需要先安装 Tomcat，再把 WAR 包丢进去，配置环境变量...
4. **集成成本高**：要集成 Redis、MQ、安全框架等，每个都要写一堆配置

Spring Boot 就是为了解决这些问题而生的：
- **自动配置**：默认配置好了大多数场景，特殊情况再改
- **起步依赖**：一个 starter 搞定所有依赖，版本自动兼容
- **内嵌容器**：直接打成 JAR 包，`java -jar` 就能运行
- **生态丰富**：几乎所有主流框架都有对应的 starter

比喻：传统 Spring 就像组装电脑，你要自己选 CPU、主板、内存、显卡，还要考虑兼容性，自己动手组装；Spring Boot 就像品牌整机，你只要选好配置（starter），厂家都给你组装测试好了，开机就能用。

### 为什么需要 Starters

1. **简化依赖管理**：不用一个个找依赖、配版本，一个 starter 全搞定
2. **避免版本冲突**：所有依赖的版本都是官方测试过的，不会出现不兼容
3. **降低学习门槛**：新手不需要知道底层需要哪些依赖，知道用哪个 starter 就行
4. **统一标准**：大家都用同样的 starter，项目结构和依赖更规范

### 为什么需要 DevTools

1. **提高开发效率**：修改代码后自动重启，不用手动点重启按钮
2. **节省等待时间**：双 ClassLoader 机制比完全重启快很多
3. **前端开发更方便**：LiveReload 自动刷新浏览器，改完页面马上看到效果
4. **远程调试支持**：支持远程应用的热部署和调试

### 为什么需要不同的打包方式

1. **JAR 包**：适合微服务架构，每个服务独立打包运行，部署简单，启动快
2. **WAR 包**：适合传统的 Web 应用部署方式，需要统一管理 Web 容器的场景
3. **EAR 包**：适合大型企业级应用，包含多个模块需要统一部署的场景

Spring Boot 推荐使用 JAR 包方式，因为：
- 部署简单：`java -jar` 一条命令搞定
- 自带容器：不需要额外安装 Tomcat
- 版本一致：开发和生产环境使用相同的容器
- 适合云原生：容器化部署非常方便

## 五、关键技巧

### 技巧一：Starters 选型技巧

1. **先选核心 starter**：大多数项目首先需要 `spring-boot-starter`（核心）和 `spring-boot-starter-web`（Web开发）
2. **按功能添加**：需要什么功能就加什么 starter，比如用数据库加 data-jpa，用缓存加 cache
3. **注意 starter 的包含关系**：比如 `spring-boot-starter-web` 已经包含了 `spring-boot-starter`，不需要重复引入
4. **测试依赖单独加**：`spring-boot-starter-test` 一般加在 test scope

```xml
<!-- 典型的Web项目starter组合 -->
<dependencies>
    <!-- Web开发 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- 数据库JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <!-- 安全认证 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <!-- 测试 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 技巧二：DevTools 提速技巧

1. **使用触发文件**：如果觉得自动重启太频繁，可以配置触发文件，改完一批代码后手动触发一次重启
2. **排除静态资源**：静态资源（HTML/CSS/JS）的修改不需要重启 JVM，用 LiveReload 就行，排除这些路径减少重启次数
3. **使用 JRebel 插件**：如果追求更快的热部署，可以考虑商业插件 JRebel，比 DevTools 更快更强大
4. **关闭不使用的功能**：如果不需要 LiveReload，可以关掉，减少资源占用

### 技巧三：打包优化技巧

**1. Spring Boot JAR 包瘦身**

默认打包方式会把所有依赖都打进 JAR 包里，导致包很大。可以用分层部署优化：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <!-- 支持分层打包 -->
                <layers>
                    <enabled>true</enabled>
                </layers>
            </configuration>
        </plugin>
    </plugins>
</build>
```

**2. 排除不需要的依赖**：根据实际需要排除不需要的依赖，减小包体积。

**3. 使用正确的打包方式：**
- 微服务 → 打 JAR 包
- 需要部署到外部 Tomcat → 打 WAR 包
- 大型企业应用 → 考虑 EAR 包

### 技巧四：快速排查 starter 依赖冲突

如果遇到依赖冲突问题，可以用 Maven 命令查看依赖树：

```bash
mvn dependency:tree
```

也可以在 IDEA 中使用 Maven Helper 插件来分析和排除冲突的依赖。

### 技巧五：自定义 Starter

如果公司内部有很多项目，可以封装自己的 starter，统一配置和依赖管理：

1. 创建一个 Maven 项目，命名规范：`xxx-spring-boot-starter`
2. 引入相关依赖
3. 编写自动配置类（使用 `@Configuration` 和条件注解）
4. 在 `META-INF/spring.factories` 中注册自动配置类
5. 其他项目引入这个 starter 即可

## 知识点总结

1. **Spring Boot** 是基于 Spring 的快速开发脚手架，核心理念是"约定优于配置"，让开发者快速搭建 Spring 应用。

2. **Starters 机制** 是 Spring Boot 的核心特性之一：
   - 命名规范：官方 `spring-boot-starter-*`，第三方 `*-spring-boot-starter`
   - 作用：一个 starter 打包某类功能的所有依赖，简化依赖管理
   - 常用 starter：web、data-jpa、data-redis、security、test 等
   - 原理：依赖传递 + 版本管理 + 自动配置

3. **DevTools 热部署** 是提升开发效率的利器：
   - 五大功能：属性默认值、自动重启、LiveReload、远程调试隧道、远程更新重启
   - 双 ClassLoader 机制实现快速重启
   - 支持触发文件控制重启时机
   - 只在开发环境使用，打包时不会包含

4. **三种打包方式**各有适用场景：
   - **JAR**：Spring Boot 默认方式，`java -jar` 直接运行，适合微服务
   - **WAR**：Web 应用归档，需要部署到 Web 容器，适合传统部署
   - **EAR**：企业级应用归档，包含多个模块，适合大型企业应用

5. **Spring Boot 的价值**：简化配置、快速开发、开箱即用、生态丰富，是目前 Java 后端开发的事实标准。

6. **实战建议**：
   - 优先使用官方 starter，不要重复造轮子
   - 开发环境一定要开 DevTools，效率提升明显
   - 默认用 JAR 包部署，简单高效
   - 注意依赖冲突，学会用 dependency:tree 排查问题
