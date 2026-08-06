# Spring Boot 多模块项目笔记

## 一、什么是多模块项目

**定义** **多模块项目（Multi-Module Project）= 由一个父 POM 管理一组子模块的 Maven 项目结构，父项目作为基础配置容器，子模块继承父项目的配置并独立开发**

包含嵌套 Maven 项目的 Spring Boot 项目称为**多模块项目**。在多模块项目中，父项目充当基础 Maven 配置的容器，子模块是实际的功能模块，它们从父项目继承 Maven 属性和公共依赖。

换句话说，多模块项目是从管理一组子模块的父 POM 构建的。父 POM 定义了**模块列表、公共依赖、统一版本**等配置，所有子模块共享这些配置，同时又保持各自的独立性。

当运行多模块项目时，所有模块可以一起部署在嵌入式 Tomcat 服务器中，也可以单独部署某个模块。

### 生活化比喻

多模块项目就像**一栋居民楼**。整栋楼（父项目）有统一的大门、统一的水电供应、统一的物业（公共依赖和配置）。每一户人家（子模块）都是独立的，有自己的装修、家具、生活方式（各自的业务代码和依赖）。但大家共享大楼的基础设施，不需要每户都自己打井取水、自己发电——就像子模块不需要每个都重复配置相同的依赖一样。

### 父 POM 与子模块的关系

- **父 POM**：位于项目根目录，`packaging` 类型为 `pom`，包含子模块列表、公共依赖、属性定义
- **子模块**：位于父项目的子目录中，`packaging` 类型通常为 `jar` 或 `war`，继承父 POM 的配置

### 子模块的打包类型

子模块可以有不同的打包类型，常见的有三种：

- **JAR**（Java ARchive）：打包成 Java 类库，通常用于通用代码、实体类、工具类等
- **WAR**（Web ARchive）：打包成 Web 应用，可以部署在 Servlet 容器中
- **EAR**（Enterprise ARchive）：企业级归档文件，可以包含多个 WAR 和 JAR

它们的关系是：JAR 文件可以被 WAR 引用，WAR 文件可以被 EAR 打包。EAR 文件是可以在应用服务器上部署的最终包。

## 二、核心机制/原理

### 2.1 父 POM 配置

**定义** **父 POM（Parent POM）= 多模块项目的顶层 POM 文件，packaging 类型为 pom，负责管理所有子模块和公共依赖**

父 POM 是多模块项目的核心，它定义了**组 ID（groupId）、工件 ID（artifactId）、版本（version）**和**打包类型（packaging）**。与普通项目不同的是，多模块项目的父 POM 的打包类型必须是 `pom`。

父 POM 的关键配置包括：

```xml
<packaging>pom</packaging>

<modules>
    <module>module1</module>
    <module>module2</module>
</modules>
```

- `<packaging>pom</packaging>`：标识这是一个聚合项目，不是可执行项目
- `<modules>`：列出所有子模块，Maven 构建时会按顺序构建这些模块

### 完整的父 POM 示例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.0</version>
        <relativePath/>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>spring-boot-multi-module</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>pom</packaging>
    <name>Spring Boot Multi Module Project</name>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <!-- 子模块列表 -->
    <modules>
        <module>application</module>
        <module>model</module>
        <module>repository</module>
        <module>service-api</module>
        <module>service-impl</module>
    </modules>

    <!-- 公共依赖 -->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

### 2.2 子模块 POM 配置

子模块的 POM 通过 `<parent>` 标签引用父 POM，从而继承父项目的所有配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- 继承父项目 -->
    <parent>
        <groupId>com.example</groupId>
        <artifactId>spring-boot-multi-module</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <artifactId>module1</artifactId>
    <packaging>jar</packaging>
    <name>Module1</name>

    <!-- 子模块自己的依赖 -->
    <dependencies>
        <!-- 父项目已有的依赖不需要重复声明 -->
    </dependencies>
</project>
```

子模块中不需要重复声明父 POM 中已经定义的依赖，也不需要指定版本号（版本由父项目统一管理）。

### 生活化比喻

父 POM 就像**公司总部的行政部门**。总部负责统一采购办公用品（公共依赖）、统一制定规章制度（属性配置）、统一管理各部门的编制（模块列表）。各个部门（子模块）不需要自己去买电脑、买文具，直接从总部领用就行。但每个部门可以根据自己的业务需要，额外申请一些特殊设备（子模块特有的依赖）。

### 2.3 模块划分原则

合理的模块划分是多模块项目成功的关键。常见的模块划分方式有两种：

#### 方式一：按功能分层划分

按照 MVC 分层来划分模块，每个层对应一个模块：

- **model 模块**：存放实体类、VO、DTO 等数据模型
- **repository 模块**：存放数据访问层代码（DAO/Repository）
- **service-api 模块**：存放服务接口定义
- **service-impl 模块**：存放服务实现类
- **application 模块**：存放启动类、控制器、配置文件等

这种划分方式的优点是**职责清晰、依赖关系明确**，适合中小型项目。

#### 方式二：按业务领域划分

按照业务领域来划分模块，每个模块包含完整的业务功能：

- **user 模块**：用户相关的所有代码（实体、DAO、Service、Controller）
- **order 模块**：订单相关的所有代码
- **payment 模块**：支付相关的所有代码

这种划分方式的优点是**业务内聚、便于团队协作**，适合大型微服务项目。

### 模块划分的核心原则

1. **单一职责**：每个模块只负责一个明确的功能领域
2. **高内聚低耦合**：模块内部功能紧密相关，模块之间依赖关系清晰
3. **可独立部署**：每个模块应该可以独立构建和测试
4. **依赖方向明确**：上层模块可以依赖下层模块，避免循环依赖
5. **合理粒度**：模块不宜过大（超过太多类），也不宜过小（每个模块只有几个类）

### 2.4 依赖管理机制

多模块项目中的依赖管理遵循以下规则：

1. **继承机制**：子模块自动继承父 POM 中声明的所有依赖
2. **版本统一**：父 POM 中定义的依赖版本，子模块无需重复指定
3. **按需添加**：子模块可以添加自己特有的依赖
4. **模块间依赖**：子模块之间可以相互依赖，通过 `groupId` + `artifactId` + `version` 引用

#### dependencyManagement 的作用

父 POM 中可以使用 `dependencyManagement` 来**统一管理版本但不实际引入依赖**：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>model</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>service-api</artifactId>
            <version>${project.version}</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

子模块需要使用时，只需声明 groupId 和 artifactId，不需要写版本：

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>model</artifactId>
</dependency>
```

### 2.5 模块间的依赖方向

在多模块项目中，模块之间的依赖关系必须有明确的方向，不能形成循环。以分层架构为例：

```
application (应用层)
    ↓ 依赖
service-impl (服务实现层)
    ↓ 依赖
service-api (服务接口层) ←→ repository (数据访问层)
    ↓                        ↓
    └────────── model (模型层) ────────┘
```

依赖关系说明：
- **application** 依赖 **service-impl**：应用层调用服务实现
- **service-impl** 依赖 **service-api** 和 **repository**：服务实现依赖服务接口和数据访问
- **service-api** 依赖 **model**：服务接口使用实体类
- **repository** 依赖 **model**：数据访问层操作实体类
- **model** 不依赖任何其他模块：最底层，最稳定

### 生活化比喻

模块间的依赖方向就像**公司的组织架构**。公司从上到下分为：
- **总经理（application）**：负责整体协调，直接指挥部门经理
- **部门经理（service-impl）**：落实具体业务，依赖员工干活
- **员工接口（service-api / repository）**：定义工作标准和职责
- **基础工具（model）**：所有员工都要用的办公设备和资料

命令是从上往下传达的，总经理不会直接管到基础工具，基础工具也不会反过来指挥总经理。层级越往下越稳定，越往上越容易变化。

### 2.6 Maven 构建顺序

Maven 在构建多模块项目时，会根据模块间的依赖关系自动确定构建顺序。构建顺序遵循以下规则：

1. 先构建没有依赖其他子模块的模块（最底层）
2. 再构建依赖了已构建模块的模块
3. 最后构建最顶层的模块

对于我们的五层架构，构建顺序是：
1. **model** → 没有内部依赖
2. **service-api** → 依赖 model
3. **repository** → 依赖 model
4. **service-impl** → 依赖 service-api 和 repository
5. **application** → 依赖 service-impl

这个顺序保证了在构建某个模块时，它所依赖的模块都已经构建完成，可以从本地仓库中找到。

## 三、使用步骤

### 3.1 创建多模块项目的步骤

#### 步骤一：创建父项目

首先创建一个 Maven 项目作为父项目，然后将 `packaging` 改为 `pom`。

```xml
<packaging>pom</packaging>
```

父项目通常不包含业务代码，只作为配置容器。

#### 步骤二：创建子模块

在父项目中创建 Maven 模块。以 IDE 为例：
右键父项目 → New → Other → Maven Module → 输入模块名 → 选择骨架 → Finish

创建完成后，父 POM 中会自动添加 `<modules>` 配置：

```xml
<modules>
    <module>model</module>
    <module>repository</module>
    <module>service-api</module>
    <module>service-impl</module>
    <module>application</module>
</modules>
```

#### 步骤三：配置各模块的职责

典型的五层模块结构如下：

**项目目录结构：**
```
spring-boot-multimodule
├── pom.xml                    # 父POM
├── application/               # 应用模块
│   ├── pom.xml
│   └── src/main/
│       ├── java/sample/multimodule/
│       │   ├── SampleWebApplication.java
│       │   └── web/WelcomeController.java
│       └── resources/
│           ├── application.properties
│           └── templates/
├── model/                     # 模型模块
│   ├── pom.xml
│   └── src/main/java/sample/multimodule/domain/entity/
│       └── Account.java
├── repository/                # 仓库模块
│   ├── pom.xml
│   └── src/main/java/sample/multimodule/repository/
│       └── AccountRepository.java
├── service-api/               # 服务接口模块
│   ├── pom.xml
│   └── src/main/java/sample/multimodule/service/api/
│       ├── AccountService.java
│       └── AccountNotFoundException.java
└── service-impl/              # 服务实现模块
    ├── pom.xml
    └── src/main/java/sample/multimodule/service/impl/
        └── AccountServiceImpl.java
```

### 3.2 各模块 POM 配置示例

#### application 模块 POM

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.example</groupId>
        <artifactId>spring-boot-multi-module</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <artifactId>application</artifactId>
    <packaging>jar</packaging>
    <name>Application Module</name>

    <dependencies>
        <!-- 依赖 service-impl 模块 -->
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>service-impl</artifactId>
            <version>${project.version}</version>
        </dependency>

        <!-- Spring Boot Web 依赖 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

#### model 模块 POM

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.example</groupId>
        <artifactId>spring-boot-multi-module</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <artifactId>model</artifactId>
    <packaging>jar</packaging>
    <name>Model Module</name>
</project>
```

#### repository 模块 POM

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.example</groupId>
        <artifactId>spring-boot-multi-module</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <artifactId>repository</artifactId>
    <packaging>jar</packaging>
    <name>Repository Module</name>

    <dependencies>
        <!-- 依赖 model 模块 -->
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>model</artifactId>
            <version>${project.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
    </dependencies>
</project>
```

### 3.3 启动类配置

在 application 模块中创建 Spring Boot 启动类：

```java
package sample.multimodule;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.orm.jpa.EntityScan;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

@SpringBootApplication
@EntityScan(basePackages = "sample.multimodule.domain.entity")
@EnableJpaRepositories(basePackages = "sample.multimodule.repository")
public class SampleWebApplication {

    public static void main(String[] args) throws Exception {
        SpringApplication.run(SampleWebApplication.class, args);
    }
}
```

注意：由于实体类和 Repository 在其他模块中，需要使用 `@EntityScan` 和 `@EnableJpaRepositories` 指定扫描包路径。

### 3.4 打包部署

在父项目根目录执行以下命令进行打包：

```bash
mvn clean package
```

Maven 会按照模块间的依赖顺序依次构建：
1. 先构建 model（没有内部依赖）
2. 再构建 repository（依赖 model）
3. 再构建 service-api（依赖 model）
4. 再构建 service-impl（依赖 repository 和 service-api）
5. 最后构建 application（依赖 service-impl）

application 模块配置了 `spring-boot-maven-plugin`，打包后会生成可执行的 JAR 文件。

运行方式与普通 Spring Boot 项目相同：

```bash
java -jar application/target/application-0.0.1-SNAPSHOT.jar
```

## 四、为什么需要多模块

### 4.1 单模块项目的痛点

当项目规模较小时，单模块项目简单直接。但随着业务复杂度增加，单模块项目会暴露出以下问题：

1. **代码臃肿**：所有代码都在一个项目中，类的数量越来越多，查找和维护困难
2. **职责不清**：实体类、DAO、Service、Controller 混在一起，边界模糊
3. **复用困难**：如果另一个项目需要使用相同的实体类，只能复制粘贴
4. **团队协作冲突**：多人同时开发时，经常出现代码冲突
5. **构建缓慢**：修改一个小功能也要重新构建整个项目
6. **部署不灵活**：只能整体部署，无法独立部署某个功能模块

### 4.2 多模块项目的优势

#### 优势一：代码复用

通用代码（如工具类、实体类、公共服务）可以抽成独立模块，其他项目直接依赖即可使用，避免代码复制。

#### 优势二：职责清晰

每个模块有明确的职责边界，开发者可以专注于自己负责的模块，降低认知负担。

#### 优势三：便于维护

修改某个模块的代码不会影响其他模块（只要接口不变），降低了修改带来的风险。

#### 优势四：依赖统一管理

所有依赖版本在父 POM 中统一管理，避免了版本不一致的问题。

#### 优势五：团队协作效率高

不同团队可以负责不同模块，各自独立开发和测试，减少代码冲突。

#### 优势六：构建和部署灵活

可以单独构建和测试某个模块，也可以只部署需要更新的模块。

### 生活化比喻

单模块项目就像**一个只有一个大房间的房子**：卧室、厨房、卫生间、客厅都在同一个空间里。人少的时候还凑合，人多了就乱套了——做饭的油烟飘到床上，看电视影响睡觉，完全没有私密性。

多模块项目就像**一套正规的三居室**：每个房间（模块）有明确的功能——卧室睡觉、厨房做饭、卫生间洗漱、客厅会客。各个房间通过门和走廊（模块间接口）连接，互不干扰。虽然建造成本稍高，但居住体验要好得多。

## 五、关键技巧

### 5.1 避免循环依赖

循环依赖是多模块项目中最常见的问题之一。如果 module-A 依赖 module-B，而 module-B 又依赖 module-A，就会形成循环依赖，导致构建失败。

**解决方法：**
- 将公共部分抽出来放到第三个模块中
- 使用接口隔离，将接口和实现分到不同模块
- 重新审视模块划分是否合理

### 5.2 合理使用 dependencyManagement

在父 POM 中使用 `<dependencyManagement>` 统一管理版本，而不是将所有依赖都放在 `<dependencies>` 中。这样做的好处是：
- 子模块按需引入依赖，不会引入不必要的依赖
- 版本统一管理，升级时只需修改一处

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>model</artifactId>
            <version>${project.version}</version>
        </dependency>
        <!-- 更多内部模块版本管理 -->
    </dependencies>
</dependencyManagement>
```

### 5.3 控制模块数量

模块不是越多越好。过多的模块会增加配置复杂度和构建时间。一般来说：
- 小型项目（3-5人团队）：2-3个模块足够
- 中型项目（5-20人团队）：5-10个模块比较合适
- 大型项目（20人以上）：可以考虑微服务架构，而不是多模块

### 5.4 包扫描配置

Spring Boot 默认只扫描启动类所在包及其子包。在多模块项目中，如果其他模块的包名与启动类包名不同，需要手动配置扫描路径：

```java
@SpringBootApplication(scanBasePackages = "sample.multimodule")
@EntityScan(basePackages = "sample.multimodule.domain.entity")
@EnableJpaRepositories(basePackages = "sample.multimodule.repository")
public class SampleWebApplication {
    // ...
}
```

最佳实践是所有模块使用相同的根包名，这样只需配置 `scanBasePackages` 即可。

### 5.5 测试多模块项目

多模块项目的测试也应该分层进行：
- **单元测试**：每个模块自己的单元测试，只测试本模块的功能
- **集成测试**：在 application 模块中编写，测试模块间的协作
- **测试依赖**：子模块的测试依赖可以在父 POM 中统一声明

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

### 5.6 常见问题排查

| 问题 | 原因 | 解决方法 |
|-----|------|---------|
| 找不到类 | 模块间缺少依赖 | 在 pom.xml 中添加对应模块的依赖 |
| Bean 未注册 | 包扫描路径不包含该模块 | 配置 `scanBasePackages` 或 `@EntityScan` |
| 循环依赖 | 模块 A 依赖 B，B 又依赖 A | 抽取公共代码到第三个模块 |
| 版本冲突 | 不同模块引入了不同版本的同一依赖 | 在父 POM 的 `dependencyManagement` 中统一版本 |
| 打包失败 | 模块构建顺序不对 | 检查模块间依赖关系，确保构建顺序正确 |

### 5.7 多模块项目的测试策略

多模块项目的测试需要分层进行，不同层级的测试重点不同：

#### 单元测试

每个模块都应该有自己的单元测试，只测试本模块的功能，不依赖其他模块。对于依赖其他模块的地方，使用 Mock 框架（如 Mockito）进行模拟。

```java
@SpringBootTest
public class AccountServiceTest {

    @MockBean
    private AccountRepository accountRepository;

    @Autowired
    private AccountService accountService;

    @Test
    public void testFindOne() {
        // Mock 数据
        Account mockAccount = new Account();
        mockAccount.setId(1L);
        mockAccount.setNumber("123456");

        Mockito.when(accountRepository.findById(1L))
               .thenReturn(Optional.of(mockAccount));

        // 测试服务层逻辑
        Account result = accountService.findOne("1");
        Assertions.assertEquals("123456", result.getNumber());
    }
}
```

#### 集成测试

集成测试放在 application 模块中，测试所有模块协同工作是否正常。集成测试会启动完整的 Spring 容器，加载所有配置。

```java
@SpringBootTest(classes = SampleWebApplication.class)
@AutoConfigureMockMvc
public class WelcomeControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void testWelcome() throws Exception {
        mockMvc.perform(get("/"))
               .andExpect(status().isOk())
               .andExpect(view().name("welcome/show"));
    }
}
```

### 5.8 多模块 vs 微服务

很多开发者会疑惑：多模块项目和微服务有什么区别？什么时候该用哪个？

| 对比维度 | 多模块项目 | 微服务 |
|---------|-----------|-------|
| 部署方式 | 一个应用整体部署 | 每个服务独立部署 |
| 数据库 | 通常共享一个数据库 | 每个服务有自己的数据库 |
| 通信方式 | 方法调用（内存级） | 网络调用（HTTP/RPC） |
| 技术栈 | 必须统一（同一语言同一框架） | 可以异构（不同语言不同框架） |
| 运维复杂度 | 低 | 高 |
| 团队协作 | 代码共享方便 | 服务间接口需要规范 |
| 适用规模 | 中小项目（1-20人） | 大型项目（20人以上） |

**选择建议：**
- 如果团队规模不大、业务复杂度适中，优先选择**多模块项目**
- 如果团队规模大、业务边界清晰、需要独立部署和扩展，再考虑**微服务**
- 多模块项目是微服务的基础，可以先做多模块拆分，以后需要时再拆成微服务

### 生活化比喻

多模块项目就像**一家公司的各个部门**。大家在同一栋楼里办公（同一个应用），共用前台、会议室、茶水间（共享基础设施），部门之间通过内部沟通协作（方法调用），效率很高。

微服务就像**集团公司下的各个子公司**。每个子公司有独立的办公地点（独立部署）、独立的财务（独立数据库）、独立的人事（独立技术栈）。子公司之间通过正式的商务合同往来（API 调用）。虽然灵活性更高，但沟通成本也更高。

## 知识点总结

1. **多模块项目**是由一个父 POM 管理一组子模块的 Maven 项目结构，父项目作为配置容器，子模块独立开发。

2. **父 POM** 的 `packaging` 类型必须为 `pom`，通过 `<modules>` 标签列出所有子模块，统一管理公共依赖和版本。

3. 子模块通过 `<parent>` 标签继承父 POM 的配置，无需重复声明公共依赖和版本号。

4. 模块划分的两种方式：**按功能分层划分**（model、repository、service、controller）和**按业务领域划分**（user、order、payment）。

5. 模块划分的原则：**单一职责、高内聚低耦合、可独立部署、依赖方向明确、合理粒度**。

6. 多模块项目的优势：**代码复用、职责清晰、便于维护、依赖统一管理、团队协作效率高、构建部署灵活**。

7. 使用 `dependencyManagement` 统一管理依赖版本，子模块按需引入，避免版本冲突。

8. 多模块项目中需要注意配置**包扫描路径**，确保 Spring 能扫描到所有模块中的 Bean。

9. **避免循环依赖**是多模块项目的关键，出现循环依赖时应考虑抽取公共代码或重新划分模块。

10. 打包时在父项目根目录执行 `mvn clean package`，Maven 会按照依赖顺序自动构建所有模块。
