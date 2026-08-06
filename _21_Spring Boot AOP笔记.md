# Spring Boot AOP学习笔记

## 一、什么是AOP

**定义** **AOP=Aspect-Oriented Programming，面向切面编程，是一种通过允许跨领域关注点分离来提高模块化的编程范式。**

Java应用程序通常分为三层：**Web层**（对外暴露服务）、**业务层**（实现业务逻辑）、**数据层**（实现持久化逻辑）。每层职责不同，但有一些适用于所有层的通用功能，比如**日志记录、安全性、验证、缓存**等，这些被称为**跨领域关注点**。

如果在每一层都重复实现这些关注点，代码会变得冗余且难以维护。**AOP**提供了解决方案：将跨领域关注点封装成独立的**切面**，通过**切入点**定义切面应用的位置，在不修改业务代码的前提下为其添加额外行为。

> **生活化比喻**：AOP就像大厦的**中央空调系统**。每个房间（业务模块）都需要制冷/制热（日志、安全等功能），但不需要每个房间都装一台空调。中央空调（切面）统一提供服务，通过通风管道（切入点）将冷气送到各个房间（连接点），住户（业务代码）完全不用关心空调怎么运行，只需要享受服务即可。

**Spring AOP**框架帮助我们实现这些跨领域关注点，它用纯Java实现，不需要特殊的编译过程，仅支持方法执行连接点，只提供运行时织入。Spring AOP代理有两种类型：**JDK动态代理**和**CGLIB代理**。

---

## 二、核心概念与原理

### 2.1 核心术语

#### （1）Aspect（切面）
**定义** **切面=封装了通知（advice）和切入点（pointcut）的模块，提供跨领域关注点的实现。**

切面是AOP的核心单元，它将散落在各处的横切逻辑集中到一个地方。我们可以使用带有**`@Aspect`**注解的普通类来实现切面。

> **比喻**：切面就像一个**多功能工具箱**，里面装着各种工具（通知）和使用说明（切入点），告诉你在什么地方用什么工具。

#### （2）Pointcut（切入点）
**定义** **切入点=一种表达式，用于选择一个或多个执行通知的连接点。**

切入点使用表达式或模式来定义，Spring AOP使用**AspectJ切入点表达式语言**。它就像一张地图，标出了哪些地方需要应用切面逻辑。

> **比喻**：切入点就像**快递配送地址列表**，上面写清楚了哪些地址（方法）需要派送包裹（通知逻辑），快递员（AOP框架）按地址送货。

#### （3）Join point（连接点）
**定义** **连接点=应用程序中应用AOP切面的具体位置，是通知的特定执行实例。**

连接点可以是**方法执行、异常处理、变量值修改**等。Spring AOP只支持方法执行类型的连接点。

> **比喻**：连接点就像**地铁线路上的各个站点**，每一个站点都是一个可以上下车（应用切面）的位置。

#### （4）Advice（通知）
**定义** **通知=在连接点之前或之后执行的动作，是切面要执行的具体逻辑代码。**

通知是切面的"干活部分"，定义了**什么时候做**和**做什么**。Spring AOP中有**五种**通知类型。

> **比喻**：通知就像**保安的工作内容**——上班前检查证件（前置通知）、下班后锁门（后置通知）、全程巡逻（环绕通知）等。

#### （5）Target object（目标对象）
**定义** **目标对象=被应用了通知的对象，始终被代理。**

运行时会创建一个覆盖目标方法的子类（代理对象），并根据配置包含通知逻辑。

> **比喻**：目标对象就像**明星本人**，而代理对象就是明星的**经纪人**，外界都通过经纪人打交道，经纪人会在前后安排各种事务（通知逻辑）。

#### （6）Weaving（织入）
**定义** **织入=将切面与其他应用程序类型或对象进行链接的过程。**

织入可以在**编译时、类加载时、运行时**三个阶段进行。Spring AOP使用**运行时织入**。

> **比喻**：织入就像**缝制衣服时把衬里缝进去**——外表看不出来，但衣服（程序）多了一层保暖（功能）。

#### （7）Proxy（代理）
**定义** **代理=将通知应用于目标对象后创建的对象。**

Spring AOP使用**JDK动态代理**来创建代理类。代理对象包装了目标对象，在调用目标方法前后插入通知逻辑。

### 2.2 AOP与OOP的对比

| AOP | OOP |
|---|---|
| **Aspect**：封装切入点、通知和属性的代码单元 | **Class**：封装方法和属性的代码单元 |
| **Pointcut**：定义执行通知的一组入口点 | **方法签名**：定义执行方法体的入口点 |
| **Advice**：跨领域关注点的实现 | **方法体**：业务逻辑关注点的实现 |
| **织入器**：借助通知构造代码 | **编译器**：将源代码转换为目标代码 |

### 2.3 Spring AOP vs AspectJ

| Spring AOP | AspectJ |
|---|---|
| 无需单独编译过程 | 需要AspectJ编译器 |
| 仅支持方法执行切入点 | 支持所有类型切入点 |
| 只能在Spring管理的Bean上实现 | 可在所有域对象上实现 |
| 仅支持方法级织入 | 可织入字段、方法、构造器、静态初始化器等 |

---

## 三、通知类型与使用步骤

### 3.1 五种通知类型

| 通知类型 | 注解 | 执行时机 |
|---|---|---|
| **Before Advice**（前置通知） | `@Before` | 在连接点**之前**执行 |
| **After Advice**（后置通知） | `@After` | 在连接点**之后**执行（无论成功或异常） |
| **Around Advice**（环绕通知） | `@Around` | 在连接点**之前和之后**都执行 |
| **After Throwing**（异常通知） | `@AfterThrowing` | 连接点**抛出异常**时执行 |
| **After Returning**（返回通知） | `@AfterReturning` | 方法**成功执行返回**时执行 |

> **比喻**：五种通知就像参加一场**会议**——
> - 前置通知：**会前**签到、发放资料
> - 后置通知：**会后**收拾会场（不管会议开得怎样都要收拾）
> - 环绕通知：**全程**主持，从开头到结束都参与
> - 异常通知：会议**出问题时**启动应急预案
> - 返回通知：会议**圆满结束时**分发会议纪要

### 3.2 使用步骤

**第一步：添加依赖**

在`pom.xml`中添加Spring Boot AOP Starter依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>
```

**第二步：创建切面类**

使用`@Aspect`和`@Component`注解标记一个类为切面：

```java
@Aspect
@Component
public class LogAspect {
    // 切面逻辑
}
```

**第三步：定义切入点**

使用`@Pointcut`注解定义切入点表达式：

```java
@Pointcut("execution(* com.example.service.*.*(..))")
public void servicePointcut() {
    // 切入点方法体为空，仅作为标记
}
```

**第四步：编写通知**

在方法上添加通知注解，并关联切入点：

```java
@Before("servicePointcut()")
public void beforeAdvice(JoinPoint joinPoint) {
    System.out.println("方法执行前: " + joinPoint.getSignature().getName());
}

@AfterReturning(pointcut = "servicePointcut()", returning = "result")
public void afterReturningAdvice(Object result) {
    System.out.println("方法返回结果: " + result);
}
```

**第五步：运行测试**

正常调用业务方法，AOP会自动生效。

---

## 四、为什么需要AOP

### 4.1 解决的痛点

在没有AOP之前，实现日志、安全等横切功能会面临以下问题：

- **代码重复**：每个方法都要写相同的日志代码
- **耦合度高**：业务代码和非业务代码混在一起
- **维护困难**：要修改日志逻辑，需要改几百个地方
- **职责不清**：业务方法既要处理业务，还要处理日志、安全等

### 4.2 AOP的优势

1. **关注点集中**：每个横切逻辑放在一个地方，不再分散
2. **业务纯粹**：业务模块只包含核心业务代码，次要逻辑移到切面
3. **解耦**：切面和业务代码互不干扰
4. **可维护性高**：修改切面逻辑只需要改一个地方
5. **无侵入**：不需要修改原有业务代码

> **生活化对比**：没有AOP就像**每家每户自己烧锅炉供暖**——每家都要买锅炉、烧煤、维护，既费钱又麻烦。有了AOP就像**城市集中供暖**——一个热电厂（切面）统一供热，通过管道（切入点）送到每家每户（连接点），住户只管交钱用暖气就行。

---

## 五、关键技巧

### 5.1 切入点表达式技巧

**execution表达式语法**：

```
execution(修饰符? 返回类型 方法名(参数) 异常?)
```

**常用写法**：

```java
// 匹配所有public方法
execution(public * *(..))

// 匹配以set开头的方法
execution(* set*(..))

// 匹配service包下所有类的所有方法
execution(* com.example.service.*.*(..))

// 匹配service包及其子包下所有类的所有方法
execution(* com.example.service..*.*(..))

// 匹配指定接口的所有实现类方法
execution(* com.example.UserService+.*(..))
```

### 5.2 多个切面的执行顺序

使用`@Order`注解控制多个切面的执行顺序，数值越小优先级越高：

```java
@Aspect
@Component
@Order(1)
public class SecurityAspect { ... }

@Aspect
@Component
@Order(2)
public class LogAspect { ... }
```

### 5.3 获取方法信息

在通知方法中通过`JoinPoint`参数获取连接点信息：

```java
@Before("servicePointcut()")
public void logBefore(JoinPoint joinPoint) {
    // 获取方法名
    String methodName = joinPoint.getSignature().getName();
    // 获取目标类
    Object target = joinPoint.getTarget();
    // 获取参数
    Object[] args = joinPoint.getArgs();
}
```

### 5.4 环绕通知注意事项

环绕通知需要手动调用`proceed()`方法执行目标方法，并返回结果：

```java
@Around("servicePointcut()")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.currentTimeMillis();
    Object result = pjp.proceed(); // 执行目标方法
    long end = System.currentTimeMillis();
    System.out.println("耗时: " + (end - start) + "ms");
    return result; // 必须返回结果
}
```

> **注意**：环绕通知如果不调用`proceed()`，目标方法不会执行；如果不返回值，调用方会收到null。

### 5.5 性能优化建议

1. **切入点尽量精确**：避免用`*.*(..)`匹配所有方法，缩小范围
2. **合理使用通知类型**：能用前置/后置通知就不用环绕通知
3. **注意线程安全**：切面是单例的，不要在切面中保存可变状态
4. **避免重入**：切面方法不要调用被切的方法，否则会无限递归

---

## 知识点总结

1. **AOP（面向切面编程）** 是一种将**跨领域关注点**（日志、安全、事务等）从业务逻辑中分离出来的编程范式，通过**切面**统一管理。

2. **核心概念**：
   - **切面（Aspect）**：封装通知和切入点的模块
   - **切入点（Pointcut）**：定义哪些位置需要应用切面
   - **连接点（Join point）**：切面应用的具体位置（Spring中是方法执行）
   - **通知（Advice）**：切面的具体逻辑和执行时机
   - **织入（Weaving）**：将切面融入目标对象的过程
   - **代理（Proxy）**：应用通知后创建的代理对象

3. **五种通知**：`@Before`（前置）、`@After`（后置）、`@Around`（环绕）、`@AfterReturning`（返回后）、`@AfterThrowing`（异常后）。

4. **使用步骤**：添加依赖 → 定义切面类（`@Aspect`）→ 定义切入点（`@Pointcut`）→ 编写通知方法 → 运行测试。

5. **AOP的价值**：代码集中管理、业务逻辑纯粹、解耦、易于维护、无侵入式增强。

6. **Spring AOP vs AspectJ**：Spring AOP简单易用但功能有限（仅方法级），AspectJ功能强大但需要特殊编译器。
