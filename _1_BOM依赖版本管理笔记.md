# BOM依赖版本管理原理笔记

## 一、什么是BOM

**定义** **BOM = Bill of Materials（物料清单）**，是Maven中一种专门用来统一管理依赖版本的机制。

它本质上就是一张**"版本价格表"**，**只声明版本号，不真正引入依赖**。

打个比方：BOM就像超市的价目表

- 价目表上写："可口可乐3元、雪碧3元、矿泉水2元"——只是登记价格
- 你光看价目表**不会真的买到饮料**，只是知道了每样东西的价格
- 等你到货架拿饮料时（写 `<dependency>`），系统自动查价目表填上价格（ `<version>` ）

## 二、核心机制

### 机制1：dependencyManagement —— 只声明，不引入

`<dependencyManagement>` 是BOM的核心标签，它的作用是"登记版本号"，而不是真正下载依赖。

在父POM或BOM中声明版本后，子模块写依赖时**不用写版本号**，Maven自动从父级查表填版本。

```xml
<!-- 在父pom或BOM中 -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.2.28</version>  <!-- 只记版本，不真下载 -->
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- 在子模块中 -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <!-- 不写version！Maven自动找到1.2.28 -->
</dependency>
```

### 机制2：scope=import —— 引入别人的BOM

`<scope>import</scope>` 是BOM的灵魂，它的作用是把别人的BOM整张表"复制粘贴"到自己的 `dependencyManagement` 里。

最常见的用法就是引入Spring Boot官方BOM，自动继承它管理的几百个依赖版本。

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>2.7.18</version>
            <type>pom</type>           <!-- 注意type=pom -->
            <scope>import</scope>      <!-- 关键：import进来 -->
        </dependency>
    </dependencies>
</dependencyManagement>
```

**效果**：把Spring Boot官方BOM的整张版本表复制到自己的dependencyManagement里，所有Spring相关依赖不用写版本号。

## 三、BOM制作步骤

### 第1步：声明自己是BOM

BOM本质是一个pom类型的项目，不含代码，只存版本信息。

```
<artifactId>yudao-dependencies</artifactId>
<packaging>pom</packaging>  <!-- pom类型，不是jar，不含代码 -->
```

### 第2步：properties集中定义版本号

所有版本号统一放在 `<properties>` 里，方便统一修改。

```xml
<properties>
    <spring.boot.version>2.7.18</spring.boot.version>
    <mybatis-plus.version>3.5.16</mybatis-plus.version>
    <redisson.version>4.6.1</redisson.version>
    <hutool-5.version>5.8.46</hutool-5.version>
    <!-- 80+个版本号集中管理 -->
</properties>
```

### 第3步：dependencyManagement里"造表"

分两部分：先import别人的BOM（站在巨人肩膀上），再声明自己的依赖版本（补充或覆盖）。

**Part A**：import别人的BOM，继承第三方版本管理

```xml
<!-- 引入Spring Boot官方BOM：自动管理Spring全家桶版本 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>${spring.boot.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

**Part B**：自己声明第三方依赖版本（覆盖或补充）

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>${mybatis-plus.version}</version>  <!-- 3.5.16 -->
</dependency>
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>${redisson.version}</version>
</dependency>
```

### 第4步：根pom把BOM引入项目

在项目的根pom.xml中，用 `scope=import` 把BOM的整张版本表导入进来。

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>cn.iocoder.boot</groupId>
            <artifactId>yudao-dependencies</artifactId>
            <version>${revision}</version>
            <type>pom</type>
            <scope>import</scope>  <!-- 把整张版本表导入根pom -->
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 第5步：子模块用依赖时不写版本号

子模块直接写依赖，不用管版本号，全部由BOM统一管理。

```xml
<dependency>
    <groupId>cn.iocoder.boot</groupId>
    <artifactId>yudao-spring-boot-starter-biz-data-permission</artifactId>
    <!-- 没有version！由yudao-dependencies的表决定 -->
</dependency>
```

## 四、版本统一传递链路

BOM的版本传递是一条清晰的链路，从源头到使用方层层传递：

```
BOM项目（yudao-dependencies）
    │
    ├── properties 定义版本变量（版本源头）
    │   └── mybatis-plus.version = 3.5.16
    │
    ├── dependencyManagement 造版本表
    │   └── mybatis-plus → ${mybatis-plus.version}
    │
    ↓ scope=import 导入
    │
根 pom.xml
    │
    ├── dependencyManagement 继承BOM的表
    │
    ↓ 传递给子模块
    │
子模块 pom.xml
    └── 写dependency，不写version，自动用BOM里的版本
```

整条链路的核心是：**版本只在一个地方定义（BOM的properties），所有地方都引用它**。升级版本时只改一个地方，全局生效。

## 五、关键技巧与坑点

### 技巧1：import顺序决定优先级

多个BOM都声明了同一个依赖的版本时，**先声明的优先级高**。后面的BOM不会覆盖前面的。

所以通常把Spring Boot官方BOM放在最前面，自己的自定义版本放在后面——想覆盖谁就把谁放在前面。

### 技巧2：exclusions也能在BOM里统一排除

BOM里不仅能声明版本，还能统一排除传递依赖。比如某个依赖自带的log4j和项目冲突，可以在BOM里统一排除掉，子模块不用每个都写exclusion。

### 坑点1：BOM不引入依赖，只管理版本

**现象**：在BOM里声明了一个依赖，子模块里还是找不到类。

**原因**：BOM里的dependencyManagement只是"登记版本"，不会真的把依赖加进来。

**解法**：子模块必须自己在 `<dependencies>`（不是dependencyManagement）里声明依赖，只是不用写版本号。

### 坑点2：scope=import只能用在dependencyManagement里

**现象**：在普通 `<dependencies>` 里写了 `scope=import`，不生效。

**原因**：import scope是dependencyManagement专属的，普通dependencies里用了会被忽略。

**解法**：import只能写在 `<dependencyManagement>` 标签内部。

### 坑点3：子模块写了版本号会覆盖BOM

**现象**：BOM里声明了版本，但子模块里还是出现了版本冲突。

**原因**：子模块的dependency里如果自己写了 `<version>`，优先级高于BOM里的版本。

**解法**：使用BOM时，子模块尽量不要写版本号，全部交给BOM统一管理。

## 六、总结

- BOM = 版本价格表，只声明版本不引入依赖，核心是 `dependencyManagement`
- `scope=import` 是BOM的灵魂，能把别人的整张版本表复制过来
- BOM是pom类型项目，不含代码，只存版本信息
- 版本号集中在properties里定义，用 `${变量名}` 引用
- 版本传递链路：BOM properties → BOM dependencyManagement → 根pom import → 子模块直接用
- 多个BOM时先声明的优先级高，后面的不会覆盖前面的
- BOM不真正引入依赖，子模块还是要写dependency，只是不用写version
- import scope只能用在dependencyManagement里，普通dependencies里用了无效
