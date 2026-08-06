# Spring Boot 自动配置笔记

## 一、什么是自动配置

**定义** **自动配置（Auto Configuration）= Spring Boot 根据类路径中的 jar 依赖自动配置 Spring 应用上下文的机制**

Spring Boot 自动配置是 Spring Boot 框架最核心的特性之一。它能够根据我们添加的 **jar 依赖项**，自动配置 Spring Boot 应用程序中的各种 Bean 和组件，开发者无需手动编写大量的 XML 配置或 Java 配置类。

举个例子：如果类路径中存在 **H2 数据库** 的 Jar 包，而我们尚未手动配置任何与数据库相关的 Bean，则 Spring Boot 的自动配置功能会在项目中自动配置数据源、实体管理器等组件。

### 生活化比喻

自动配置就像**智能家居系统**。当你买回一台智能音箱（相当于引入一个 starter 依赖），它会自动连接家里的 WiFi、自动识别你的手机设备、自动配置语音助手。你不需要手动设置网络参数、不需要手动安装驱动、不需要手动配置各种协议——插电就能用。Spring Boot 的自动配置就是做这件事的：只要你引入了相关依赖，它就自动把对应的环境配置好。

### @SpringBootApplication 注解

我们通常使用 `@SpringBootApplication` 注解来启用自动配置，它是三个注解的组合：

```java
@SpringBootApplication = @ComponentScan + @EnableAutoConfiguration + @Configuration
```

- **@ComponentScan**：扫描组件，自动发现并注册标注了 `@Controller`、`@Service`、`@Repository`、`@Component` 等注解的 Bean
- **@EnableAutoConfiguration**：启用自动配置的核心注解
- **@Configuration**：标识该类是配置类，可以定义 Bean

### 自动配置的实际效果

当在项目中添加 `spring-boot-starter-web` 依赖时，Spring Boot 自动配置会：
- 自动配置 **DispatcherServlet**（前端控制器）
- 自动配置默认的**错误页面**
- 自动配置 **Web Jars**（静态资源处理）

当添加 `spring-boot-starter-data-jpa` 依赖时，Spring Boot 自动配置会：
- 自动配置 **数据源**（DataSource）
- 自动配置 **实体管理器工厂**（EntityManagerFactory）
- 自动配置 **事务管理器**（TransactionManager）

所有自动配置逻辑都在 `spring-boot-autoconfigure.jar` 中实现。

## 二、核心机制/原理

### 2.1 @EnableAutoConfiguration 工作原理

**定义** **@EnableAutoConfiguration = 启用 Spring Boot 自动配置机制的核心注解，通过导入 AutoConfigurationImportSelector 实现自动配置类的加载**

`@EnableAutoConfiguration` 注解是自动配置的入口。它通过 `@Import` 注解导入了 `AutoConfigurationImportSelector` 类，该类会读取 `META-INF/spring.factories` 文件中配置的所有自动配置类，然后根据条件判断哪些自动配置类需要生效。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    // ...
}
```

### 生活化比喻

`@EnableAutoConfiguration` 就像是**小区物业的总控室**。总控室里有一张清单（`spring.factories`），上面列着所有可能需要的服务（自动配置类）：保安、保洁、电梯维护、绿化等等。但不是所有服务都会同时启动——物业会根据小区的实际情况（条件注解判断）来决定开启哪些服务。如果小区有电梯，就开电梯维护；如果有花园，就开绿化服务。

### 2.2 spring.factories 文件

Spring Boot 在启动时，会通过 `SpringFactoriesLoader` 加载 `META-INF/spring.factories` 文件，该文件中配置了所有自动配置类的全限定名。

```properties
# 自动配置类的配置示例
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration,\
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration,\
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration
```

每一行都是一个自动配置类，Spring Boot 启动时会加载所有这些类，然后通过**条件注解**判断是否需要实例化。

### 2.3 条件注解（Conditional Annotations）

**定义** **条件注解 = 用于控制自动配置类是否生效的注解，只有满足指定条件时，对应的配置才会被加载**

条件注解是自动配置的"智能开关"，它让自动配置不是盲目的，而是根据环境来决定是否启用。Spring Boot 提供了丰富的条件注解：

| 条件注解 | 作用 |
|---------|------|
| `@ConditionalOnClass` | 类路径中存在指定类时生效 |
| `@ConditionalOnMissingClass` | 类路径中不存在指定类时生效 |
| `@ConditionalOnBean` | 容器中存在指定 Bean 时生效 |
| `@ConditionalOnMissingBean` | 容器中不存在指定 Bean 时生效 |
| `@ConditionalOnProperty` | 配置文件中存在指定属性时生效 |
| `@ConditionalOnWebApplication` | 当前是 Web 应用时生效 |
| `@ConditionalOnNotWebApplication` | 当前不是 Web 应用时生效 |

### 条件注解示例

以 `DataSourceAutoConfiguration` 为例，它的部分代码如下：

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class, ... })
public class DataSourceAutoConfiguration {
    // ...
}
```

这段配置的含义是：
- 只有当类路径中存在 `DataSource` 和 `EmbeddedDatabaseType` 类时，该配置才生效
- 只有当容器中没有 `ConnectionFactory` 类型的 Bean 时才生效
- 自动绑定 `DataSourceProperties` 配置属性

### 2.4 starter 与自动配置的关系

**定义** **Starter = 一组预定义的依赖集合，它将相关的依赖打包在一起，同时引入对应的自动配置类**

Starter 和自动配置是**相辅相成**的关系：

1. **Starter 负责引入依赖**：当你添加 `spring-boot-starter-web` 时，它会把 Spring MVC、Tomcat、Jackson 等相关依赖都引入进来
2. **自动配置负责配置这些依赖**：依赖引入后，自动配置检测到类路径中有这些类，就自动配置好对应的 Bean

### 生活化比喻

Starter 就像**外卖套餐**。你点一份"全家桶套餐"（starter），套餐里包含了汉堡、薯条、可乐、鸡块（各种依赖 jar 包）。同时，外卖店会自动给你配好餐巾纸、酱料、手套（自动配置）——你不需要额外要求，只要点了套餐，配套的东西都会自动送来。

### 2.5 自动配置的执行流程

自动配置的完整执行流程可以分为以下几个阶段：

**阶段一：启动入口**

当 Spring Boot 应用启动时，`@SpringBootApplication` 注解中的 `@EnableAutoConfiguration` 开始发挥作用。它通过 `@Import(AutoConfigurationImportSelector.class)` 导入了自动配置选择器。

**阶段二：加载候选配置**

`AutoConfigurationImportSelector` 调用 `SpringFactoriesLoader.loadFactoryNames()` 方法，从类路径下所有 `META-INF/spring.factories` 文件中读取 `EnableAutoConfiguration` 对应的配置类全限定名列表。这一步会加载所有候选自动配置类，通常有上百个。

**阶段三：条件过滤**

这是最关键的一步。Spring Boot 会逐个检查每个自动配置类上的条件注解，判断是否满足条件：
- 检查 `@ConditionalOnClass`：类路径中是否存在指定的类
- 检查 `@ConditionalOnMissingBean`：容器中是否还没有这个 Bean
- 检查 `@ConditionalOnProperty`：配置文件中是否有对应的属性
- 其他各种条件注解

只有所有条件都满足的自动配置类才会被注册到 Spring 容器中。

**阶段四：配置属性绑定**

对于生效的自动配置类，Spring Boot 会通过 `@EnableConfigurationProperties` 将 `application.properties` 或 `application.yml` 中的配置属性绑定到对应的属性类上。

**阶段五：创建 Bean**

最后，自动配置类中定义的 `@Bean` 方法会被执行，创建相应的 Bean 并注册到 Spring 容器中。这些 Bean 就可以像普通 Bean 一样被注入和使用了。

### 生活化比喻

自动配置的执行流程就像**医院体检的流程**。你拿着体检单（spring.factories 中的候选列表）去体检中心，但不是每个项目都做：
- 首先看你有没有预约（@ConditionalOnClass：有没有对应的依赖）
- 然后看你以前有没有做过（@ConditionalOnMissingBean：有没有手动配置过）
- 还要看你有没有特殊要求（@ConditionalOnProperty：配置文件有没有指定）
- 符合条件的项目才会真正执行（创建 Bean）
- 不符合的项目就跳过了，不会浪费时间

## 三、使用步骤

### 3.1 使用自动配置

自动配置在 Spring Boot 中是默认开启的，只需使用 `@SpringBootApplication` 注解即可：

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

### 3.2 禁用特定的自动配置类

如果不想让某个自动配置类生效，可以使用 `@EnableAutoConfiguration` 的 `exclude` 属性：

```java
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration;

@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
public class MyApplication {
    // ...
}
```

如果类不在类路径中，可以使用 `excludeName` 属性并指定类的全限定名：

```java
@SpringBootApplication(excludeName = {
    "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration"
})
public class MyApplication {
    // ...
}
```

也可以在 `application.properties` 中配置：

```properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

### 3.3 调试自动配置

#### 方法一：开启调试日志

在 `application.properties` 中添加以下配置：

```properties
logging.level.org.springframework: DEBUG
```

重启应用后，控制台会打印自动配置报告，分为两部分：
- **正向匹配（Positive matches）**：已经生效的自动配置
- **负向匹配（Negative matches）**：未生效的自动配置及其原因

#### 方法二：使用 Spring Boot Actuator

引入 Actuator 依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

启动应用后访问 `http://localhost:8080/actuator`，可以查看各种监控信息，包括自动配置的 Bean。

还可以引入 HAL 浏览器方便查看：

```xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-rest-hal-browser</artifactId>
</dependency>
```

### 3.4 自定义自动配置

自定义自动配置是 Spring Boot 扩展开发的重要技能。当我们需要将自己的组件封装成 starter 供其他项目使用时，就需要编写自定义自动配置。

#### 完整示例：封装一个短信服务 starter

假设我们要封装一个短信服务的 starter，让其他项目引入后就能自动使用短信功能。

**步骤一：创建自动配置类**

```java
package com.example.sms.autoconfigure;

import com.example.sms.SmsService;
import com.example.sms.impl.AliyunSmsService;
import com.example.sms.impl.TencentSmsService;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@ConditionalOnClass(SmsService.class)
@EnableConfigurationProperties(SmsProperties.class)
public class SmsAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "sms", name = "provider", havingValue = "aliyun", matchIfMissing = true)
    public SmsService aliyunSmsService(SmsProperties properties) {
        AliyunSmsService smsService = new AliyunSmsService();
        smsService.setAccessKey(properties.getAccessKey());
        smsService.setSecretKey(properties.getSecretKey());
        smsService.setSignName(properties.getSignName());
        return smsService;
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "sms", name = "provider", havingValue = "tencent")
    public SmsService tencentSmsService(SmsProperties properties) {
        TencentSmsService smsService = new TencentSmsService();
        smsService.setAppId(properties.getAppId());
        smsService.setAppKey(properties.getAppKey());
        smsService.setSignName(properties.getSignName());
        return smsService;
    }
}
```

**步骤二：创建属性配置类**

```java
package com.example.sms.autoconfigure;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "sms")
public class SmsProperties {

    /**
     * 短信服务商：aliyun / tencent
     */
    private String provider = "aliyun";

    /**
     * 阿里云 AccessKey
     */
    private String accessKey;

    /**
     * 阿里云 SecretKey
     */
    private String secretKey;

    /**
     * 腾讯云 AppId
     */
    private String appId;

    /**
     * 腾讯云 AppKey
     */
    private String appKey;

    /**
     * 短信签名
     */
    private String signName;

    // getter 和 setter 方法
    public String getProvider() { return provider; }
    public void setProvider(String provider) { this.provider = provider; }
    public String getAccessKey() { return accessKey; }
    public void setAccessKey(String accessKey) { this.accessKey = accessKey; }
    public String getSecretKey() { return secretKey; }
    public void setSecretKey(String secretKey) { this.secretKey = secretKey; }
    public String getAppId() { return appId; }
    public void setAppId(String appId) { this.appId = appId; }
    public String getAppKey() { return appKey; }
    public void setAppKey(String appKey) { this.appKey = appKey; }
    public String getSignName() { return signName; }
    public void setSignName(String signName) { this.signName = signName; }
}
```

**步骤三：创建服务接口和实现类**

```java
package com.example.sms;

public interface SmsService {
    /**
     * 发送短信
     * @param phone 手机号
     * @param content 短信内容
     * @return 是否发送成功
     */
    boolean sendSms(String phone, String content);
}
```

**步骤四：注册自动配置类**

在 `src/main/resources/META-INF/spring.factories` 文件中添加：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.sms.autoconfigure.SmsAutoConfiguration
```

**步骤五：在业务项目中使用**

引入 starter 依赖后，在 `application.properties` 中配置：

```properties
sms.provider=aliyun
sms.access-key=your-access-key
sms.secret-key=your-secret-key
sms.sign-name=您的签名
```

然后直接注入使用：

```java
@RestController
public class UserController {

    @Autowired
    private SmsService smsService;

    @PostMapping("/register")
    public String register(String phone) {
        smsService.sendSms(phone, "您的验证码是：123456");
        return "发送成功";
    }
}
```

#### 自定义 starter 的命名规范

官方 starter 的命名格式是 `spring-boot-starter-xxx`，第三方自定义 starter 的命名格式是 `xxx-spring-boot-starter`。例如：
- 官方：`spring-boot-starter-web`
- 第三方：`mybatis-spring-boot-starter`

## 四、为什么需要自动配置

### 4.1 传统 Spring 开发的痛点

在没有自动配置之前，使用 Spring 开发应用需要手动配置大量内容。

#### Web 应用配置示例

配置 DispatcherServlet：

```xml
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/todo-servlet.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

#### JPA/Hibernate 配置示例

配置数据源：

```xml
<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource"
      destroy-method="close">
    <property name="driverClass" value="${db.driver}" />
    <property name="jdbcUrl" value="${db.url}" />
    <property name="user" value="${db.username}" />
    <property name="password" value="${db.password}" />
</bean>

<jdbc:initialize-database data-source="dataSource">
    <jdbc:script location="classpath:config/schema.sql" />
    <jdbc:script location="classpath:config/data.sql" />
</jdbc:initialize-database>
```

配置实体管理器工厂：

```xml
<bean class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean"
      id="entityManagerFactory">
    <property name="persistenceUnitName" value="hsql_pu" />
    <property name="dataSource" ref="dataSource" />
</bean>
```

配置事务管理器：

```xml
<bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager">
    <property name="entityManagerFactory" ref="entityManagerFactory" />
    <property name="dataSource" ref="dataSource" />
</bean>

<tx:annotation-driven transaction-manager="transactionManager" />
```

### 4.2 自动配置带来的价值

1. **极大减少配置代码**：传统项目需要几百行 XML 配置，现在只需引入 starter 依赖
2. **快速开发**：开箱即用，新手也能快速搭建项目
3. **统一配置标准**：所有项目遵循相同的配置约定，降低团队协作成本
4. **灵活可控**：自动配置是"约定优于配置"，但支持自定义覆盖
5. **版本兼容**：starter 中的依赖版本经过官方测试，避免版本冲突

### 4.3 Spring 配置方式的演进

Spring 的配置方式经历了三个阶段的演进，自动配置是发展的必然结果：

**第一阶段：XML 配置（Spring 1.x - 2.x）**

所有 Bean 都在 XML 文件中声明，配置文件庞大且难以维护。修改配置需要重启应用，没有类型安全检查，拼写错误只能在运行时发现。

**第二阶段：Java 配置 + 注解（Spring 3.x - 4.x）**

引入了 `@Configuration`、`@Bean`、`@ComponentScan` 等注解，开发者可以用 Java 代码来写配置。相比 XML 更类型安全、更易重构，但仍需要手动编写大量配置类。

**第三阶段：自动配置（Spring Boot 时代）**

在 Java 配置的基础上更进一步，把常见的配置场景都预先写好，通过条件注解判断是否启用。开发者只需引入依赖，配置自动完成，真正实现了"约定优于配置"。

这个演进过程的核心趋势是：**配置越来越少，开发越来越快，开发者可以更专注于业务逻辑而不是框架配置。**

### 生活化比喻

传统 Spring 配置就像**组装一台台式电脑**：你需要自己买 CPU、主板、内存、显卡、电源、机箱，然后自己组装、接线、装系统、装驱动，每一步都需要专业知识，还可能遇到兼容性问题。

Spring Boot 自动配置就像**买品牌笔记本电脑**：你只需选择一个型号（starter），开箱就能用。CPU、内存、硬盘、操作系统、驱动全都预装好了。如果你不满意，也可以自己升级内存（自定义配置），但默认配置已经能满足大多数需求。

## 五、关键技巧

### 5.1 查看自动配置报告

通过开启 DEBUG 日志，可以清楚地看到哪些自动配置生效了、哪些没生效，以及没生效的原因。这是排查自动配置问题的**第一利器**。

```properties
logging.level.org.springframework: DEBUG
```

启动后在日志中搜索 "Auto-configuration report"，可以看到完整的报告。

### 5.2 自定义覆盖自动配置

当你手动配置了某个 Bean 时，Spring Boot 的自动配置会自动退出。这是因为自动配置类中使用了 `@ConditionalOnMissingBean` 注解。

例如，如果你手动配置了数据源：

```java
@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource dataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        dataSource.setUsername("root");
        dataSource.setPassword("password");
        return dataSource;
    }
}
```

那么 Spring Boot 默认的数据源自动配置就不会生效，你的配置会**优先生效**。

### 5.3 使用 @ConditionalOnProperty 做开关

可以通过配置文件中的属性来控制自定义功能是否开启：

```java
@Configuration
@ConditionalOnProperty(prefix = "my.feature", name = "enabled", havingValue = "true")
public class MyFeatureAutoConfiguration {
    // ...
}
```

在 `application.properties` 中控制：

```properties
my.feature.enabled=true  # 开启
# my.feature.enabled=false  # 关闭
```

### 5.4 理解自动配置的顺序

自动配置类之间可能存在依赖关系，Spring Boot 提供了以下注解控制顺序：

- `@AutoConfigureBefore`：在指定配置类之前加载
- `@AutoConfigureAfter`：在指定配置类之后加载
- `@AutoConfigureOrder`：指定配置类的加载顺序

### 5.5 常见问题排查

| 问题 | 排查方法 |
|-----|---------|
| 自动配置没生效 | 开启 DEBUG 日志查看负向匹配原因 |
| Bean 冲突 | 检查是否有重复定义的 Bean，使用 `@Primary` 或 `@ConditionalOnMissingBean` |
| 属性不生效 | 检查 `@ConfigurationProperties` 的 prefix 是否正确，属性名是否匹配 |
| 版本不兼容 | 检查 starter 版本与 Spring Boot 版本是否匹配 |

### 5.6 常见自动配置类一览

Spring Boot 提供了上百个自动配置类，以下是最常用的一些：

| 自动配置类 | 作用 | 触发条件 |
|-----------|------|---------|
| `DataSourceAutoConfiguration` | 自动配置数据源 | 类路径中有 DataSource 类 |
| `DataSourceTransactionManagerAutoConfiguration` | 自动配置事务管理器 | 容器中有 DataSource Bean |
| `JdbcTemplateAutoConfiguration` | 自动配置 JdbcTemplate | 类路径中有 JdbcTemplate 类 |
| `HibernateJpaAutoConfiguration` | 自动配置 Hibernate JPA | 类路径中有 Hibernate 和 JPA 相关类 |
| `DispatcherServletAutoConfiguration` | 自动配置 DispatcherServlet | 类路径中有 DispatcherServlet 类 |
| `WebMvcAutoConfiguration` | 自动配置 Spring MVC | 类路径中有 WebMvc 相关类 |
| `HttpMessageConvertersAutoConfiguration` | 自动配置消息转换器 | 类路径中有 HttpMessageConverter 类 |
| `JacksonAutoConfiguration` | 自动配置 Jackson JSON | 类路径中有 Jackson 相关类 |
| `RedisAutoConfiguration` | 自动配置 Redis | 类路径中有 Redis 相关类 |
| `SecurityAutoConfiguration` | 自动配置 Spring Security | 类路径中有 Spring Security 相关类 |

了解这些常用自动配置类的作用，可以帮助我们快速判断某个功能是否由自动配置提供，以及如何禁用或自定义它。

### 5.7 自动配置的最佳实践

在使用自动配置时，遵循以下最佳实践可以让项目更易维护：

1. **优先使用自动配置**：不要轻易写自定义配置，先看看自动配置能不能满足需求
2. **使用配置文件定制**：优先通过 `application.properties` 或 `application.yml` 配置属性，而不是写配置类
3. **手动配置只覆盖需要的**：如果需要自定义，只覆盖特定的 Bean，不要把整个自动配置都禁掉
4. **命名规范**：自定义配置类命名为 `XxxAutoConfiguration`，属性类命名为 `XxxProperties`
5. **善用 DEBUG 日志**：遇到配置问题时，第一时间开启 DEBUG 日志查看自动配置报告
6. **版本一致**：确保所有 starter 的版本与 Spring Boot 主版本一致，避免兼容性问题

### 生活化比喻

自动配置的最佳实践就像**使用智能手机**：
- 优先用系统自带的功能（优先使用自动配置），不要随便装第三方 ROM
- 个性化设置在系统设置里改（用配置文件定制），不要去改系统源码
- 只装需要的 App（只覆盖需要的 Bean），不要乱刷机
- 遇到问题先看设置里的"关于手机"和日志（开启 DEBUG 日志），不要盲目刷机
- 系统版本和 App 版本要匹配（版本一致），不然容易闪退

## 知识点总结

1. **自动配置**是 Spring Boot 的核心特性，它根据类路径中的依赖自动配置 Bean，开发者无需手动编写大量配置。

2. **@SpringBootApplication** 是组合注解，包含 `@ComponentScan`、`@EnableAutoConfiguration`、`@Configuration` 三个注解。

3. 自动配置的核心原理：通过 `@EnableAutoConfiguration` 导入 `AutoConfigurationImportSelector`，读取 `spring.factories` 中的自动配置类列表，再通过**条件注解**判断哪些配置需要生效。

4. **条件注解**是自动配置的"智能开关"，常用的有 `@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@ConditionalOnProperty` 等。

5. **Starter** 是预定义的依赖集合，负责引入依赖；**自动配置**负责配置这些依赖。两者配合实现"开箱即用"。

6. 可以通过 `exclude` 属性或 `spring.autoconfigure.exclude` 配置禁用特定的自动配置类。

7. 自定义自动配置需要：创建配置类、使用条件注解、在 `spring.factories` 中注册。

8. 调试自动配置的两种方法：开启 DEBUG 日志查看自动配置报告、使用 Actuator 监控。

9. 手动配置的 Bean 优先级高于自动配置，因为自动配置使用了 `@ConditionalOnMissingBean`。

10. 自动配置体现了"**约定优于配置**"的设计理念，在简化开发的同时保留了灵活的自定义能力。
