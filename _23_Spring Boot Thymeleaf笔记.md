# Spring Boot Thymeleaf学习笔记

## 一、什么是Thymeleaf

**定义** **Thymeleaf=一个开源的Java服务器端模板引擎，用于处理HTML5/XHTML/XML模板，可在Web和非Web环境下运行，与Spring Framework深度集成。**

Thymeleaf基于Apache License 2.0开源协议发布，是现代HTML5 JVM Web开发的理想选择。它对模板文件应用一组转换，来展示应用程序生成的数据或文本。

Thymeleaf的核心理念是**自然模板**——模板文件本身就是合法的HTML，可以直接在浏览器中打开并正常显示（显示静态原型数据），经过服务器处理后，动态数据会替换掉静态内容。这与JSP有本质区别：JSP包含无法被浏览器识别的标签，只能在服务器运行后才能看到效果。

> **生活化比喻**：Thymeleaf就像**一本可擦写的字帖**。字帖上本来有示例字（静态原型数据），你可以直接看字帖（浏览器直接打开）。当你用钢笔描红（服务器渲染）后，你的字（动态数据）就覆盖了原来的示例字，呈现出最终效果。设计师和开发者可以共用同一套模板文件。

### 1.1 为什么使用Thymeleaf

- **浏览器友好**：模板文件就是HTML，浏览器可直接打开预览
- **Spring集成**：与Spring MVC/Spring Boot完美集成，支持Spring EL
- **语法优雅**：使用HTML属性（`th:`前缀），不破坏HTML结构
- **国际化支持**：内置国际化（i18n）支持
- **高性能**：模板缓存机制，减少I/O操作
- **可扩展**：支持自定义方言和处理器

### 1.2 支持的模板模式

Thymeleaf可以处理六种模板模式：

| 模板模式 | 说明 |
|---|---|
| **XML** | 标准XML模板 |
| **VALID XML** | 有效的XML模板 |
| **XHTML** | XHTML模板 |
| **VALID XHTML** | 有效的XHTML模板 |
| **HTML5** | 标准HTML5模板（推荐） |
| **LEGACY HTML5** | 兼容非规范HTML5（自动转换为格式良好的XML） |

> 注意：验证功能仅适用于XHTML和XML模板，HTML5模式不做验证。

---

## 二、核心概念与原理

### 2.1 标准方言（Standard Dialect）

**定义** **方言（Dialect）=一组处理器（Processor）和额外工具的集合，用于定义模板中可以使用的逻辑和属性。**

Thymeleaf核心库自带的方言称为**标准方言（Standard Dialect）**。与Spring集成时，使用的是**SpringStandard Dialect**（Spring标准方言），它与标准方言几乎相同，只是将表达式语言替换为Spring Expression Language（Spring EL），更好地利用Spring框架的功能。

> **比喻**：方言就像**Word的工具栏**——标准方言是默认工具栏，有常用的按钮；Spring标准方言是增强版工具栏，增加了一些Spring特有的功能按钮。

#### 处理器（Processor）

处理器是将逻辑应用于DOM节点的对象。标准方言的处理器大多是**属性处理器**——以`th:`前缀的HTML属性。浏览器在处理模板前会忽略这些未知属性，所以模板可以直接在浏览器中显示。

**JSP vs Thymeleaf对比**：

JSP写法（浏览器无法解析）：
```jsp
<form:inputText name="student.Name" value="${student.name}" />
```

Thymeleaf写法（浏览器可正常显示）：
```html
<input type="text" name="studentName" value="Thomas" th:value="${student.name}" />
```

浏览器打开时显示静态值`Thomas`，服务器渲染后`th:value`的值会覆盖`value`属性。

### 2.2 标准表达式

Thymeleaf提供了五种标准表达式，用于在模板中处理数据：

#### （1）变量表达式 `${...}`

**定义** **变量表达式=用于获取模型（Model）中的属性值，底层使用Spring EL（或OGNL）解析。**

```html
<!-- 获取用户姓名 -->
<p th:text="${user.name}">默认姓名</p>

<!-- 调用方法 -->
<p th:text="${user.getName()}">默认姓名</p>

<!-- 访问数组/List -->
<p th:text="${users[0].name}">第一个用户</p>

<!-- 访问Map -->
<p th:text="${userMap['admin']}">管理员</p>
```

> **比喻**：变量表达式就像**快递取件码**，你告诉快递柜编号（属性名），它就把对应的包裹（值）取出来给你。

#### （2）选择变量表达式 `*{...}`

**定义** **选择变量表达式=在选定的对象（th:object指定的对象）上执行，省去重复的对象前缀。**

```html
<div th:object="${user}">
    <p th:text="*{name}">姓名</p>      <!-- 等价于 ${user.name} -->
    <p th:text="*{email}">邮箱</p>     <!-- 等价于 ${user.email} -->
    <p th:text="*{age}">年龄</p>       <!-- 等价于 ${user.age} -->
</div>
```

> **比喻**：选择变量表达式就像**进入某家商店后**，你说"拿最贵的那个"，店员就知道你说的是这家店里最贵的，而不是全市最贵的。上下文（商店）已经选定了。

#### （3）消息表达式 `#{...}`

**定义** **消息表达式=用于国际化（i18n），从properties文件中读取对应语言的消息。**

```html
<!-- 读取国际化消息 -->
<h1 th:text="#{welcome.title}">欢迎</h1>

<!-- 带参数的消息 -->
<p th:text="#{user.greeting(${user.name})}">你好，用户</p>
```

对应的`messages.properties`文件：
```properties
welcome.title=欢迎光临
user.greeting=你好，{0}！
```

> **比喻**：消息表达式就像**多语言翻译机**，你说一个键（key），它根据当前语言环境，输出对应的翻译文本。

#### （4）链接表达式 `@{...}`

**定义** **链接表达式=用于生成URL，自动处理上下文路径和URL重写。**

```html
<!-- 绝对路径 -->
<a th:href="@{https://www.example.com}">链接</a>

<!-- 相对路径（自动加上下文路径） -->
<a th:href="@{/user/list}">用户列表</a>

<!-- 带参数 -->
<a th:href="@{/user/detail(id=${user.id})}">详情</a>
<!-- 生成: /context/user/detail?id=1 -->

<!-- 带多个参数 -->
<a th:href="@{/user/search(name='Tom', age=20)}">搜索</a>
```

> **比喻**：链接表达式就像**导航系统**，你只需要告诉它目的地（路径）和乘客（参数），它自动规划最优路线（加上下文路径、拼接参数）。

#### （5）片段表达式 `~{...}`

**定义** **片段表达式=用于引用模板片段，实现模板的复用（类似include）。**

```html
<!-- 引用其他模板的片段 -->
<div th:insert="~{fragments/header :: header}"></div>

<!-- 简写形式 -->
<div th:replace="fragments/header :: header"></div>
```

### 2.3 表达式内置对象

Thymeleaf提供了一些内置对象，方便在表达式中使用：

| 对象 | 用途 |
|---|---|
| `#ctx` | 上下文对象 |
| `#vars` | 变量对象 |
| `#locale` | 当前区域设置 |
| `#request` | HttpServletRequest对象（Web环境） |
| `#session` | HttpSession对象（Web环境） |
| `#servletContext` | ServletContext对象 |

**工具类对象**：

| 对象 | 用途 |
|---|---|
| `#strings` | 字符串工具（isEmpty、toUpperCase、replace等） |
| `#numbers` | 数字工具（格式化等） |
| `#dates` | 日期工具（格式化、比较等） |
| `#lists` | 列表工具（size、isEmpty、sort等） |
| `#sets` | 集合工具 |
| `#maps` | Map工具 |
| `#arrays` | 数组工具 |
| `#objects` | 对象工具 |
| `#bools` | 布尔工具 |

```html
<!-- 字符串工具 -->
<p th:text="${#strings.toUpperCase(user.name)}">姓名</p>

<!-- 日期格式化 -->
<p th:text="${#dates.format(user.createTime, 'yyyy-MM-dd')}">日期</p>

<!-- 判断列表是否为空 -->
<div th:if="${#lists.isEmpty(users)}">暂无数据</div>
```

---

## 三、使用步骤

### 3.1 Spring Boot集成Thymeleaf

**第一步：添加依赖**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

Spring Boot自动配置Thymeleaf，模板默认从`/resource/templates`目录读取。

**第二步：配置文件（可选）**

在`application.properties`中配置：

```properties
# 关闭缓存（开发环境建议关闭，生产环境开启）
spring.thymeleaf.cache=false
# 模板后缀
spring.thymeleaf.suffix=.html
# 模板编码
spring.thymeleaf.encoding=UTF-8
# 模板模式
spring.thymeleaf.mode=HTML5
```

**第三步：创建实体类**

```java
package com.example;

public class User {
    private String name;
    private String email;

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}
```

**第四步：创建Controller**

```java
package com.example;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.servlet.ModelAndView;

@Controller
public class DemoController {

    // 显示首页（表单页）
    @RequestMapping("/")
    public String index() {
        return "index"; // 对应 templates/index.html
    }

    // 处理表单提交，展示用户数据
    @RequestMapping(value = "/save", method = RequestMethod.POST)
    public ModelAndView save(@ModelAttribute User user) {
        ModelAndView modelAndView = new ModelAndView();
        modelAndView.setViewName("user-data"); // 对应 templates/user-data.html
        modelAndView.addObject("user", user);
        return modelAndView;
    }
}
```

**第五步：创建模板文件**

在`src/main/resources/templates/`下创建模板。

`index.html`（表单页）：
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Index Page</title>
</head>
<body>
    <form action="save" method="post">
        <table>
            <tr>
                <td><label for="user-name">User Name</label></td>
                <td><input type="text" name="name" id="user-name"></td>
            </tr>
            <tr>
                <td><label for="email">Email</label></td>
                <td><input type="text" name="email" id="email"></td>
            </tr>
            <tr>
                <td></td>
                <td><input type="submit" value="Submit"></td>
            </tr>
        </table>
    </form>
</body>
</html>
```

`user-data.html`（结果页）：
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>用户数据</title>
</head>
<body>
    <table>
        <tr>
            <td><h4>User Name: </h4></td>
            <td><h4 th:text="${user.name}">默认姓名</h4></td>
        </tr>
        <tr>
            <td><h4>Email ID: </h4></td>
            <td><h4 th:text="${user.email}">默认邮箱</h4></td>
        </tr>
    </table>
</body>
</html>
```

> 注意：模板中需要引入Thymeleaf命名空间 `xmlns:th="http://www.thymeleaf.org"`。

**第六步：运行测试**

启动应用，访问 `http://localhost:8080/`，填写表单后提交，即可看到渲染结果。

---

## 四、为什么需要Thymeleaf

### 4.1 相比JSP的优势

JSP（JavaServer Pages）是传统的Java Web视图技术，但有很多不足：

| 对比项 | JSP | Thymeleaf |
|---|---|---|
| **浏览器预览** | 不能直接打开，必须部署到服务器 | 可以直接在浏览器中打开（自然模板） |
| **语法** | 特殊标签，破坏HTML结构 | 使用HTML属性，不破坏结构 |
| **前后端协作** | 前端看不懂JSP代码 | 前端可以直接基于HTML原型开发 |
| **Spring集成** | 需要额外配置 | Spring Boot开箱即用 |
| **缓存** | 无内置缓存 | 内置模板缓存，性能好 |
| **可扩展性** | 有限 | 支持自定义方言，扩展灵活 |

### 4.2 Thymeleaf的核心价值

1. **自然模板**：模板即原型，前后端共用一套文件，减少沟通成本
2. **优雅语法**：使用`th:`属性扩展HTML，代码整洁易读
3. **功能丰富**：表达式、迭代、条件判断、表单处理、国际化、模板片段
4. **Spring深度集成**：支持Spring EL、表单绑定、验证、安全等
5. **高性能**：模板缓存减少I/O，解析速度快

> **生活化对比**：JSP就像**只有通电才能看的电子相册**——不通电（不运行服务器）就是一块黑屏，什么也看不到。Thymeleaf就像**普通的纸质相册**——就算没有电（不运行服务器），你也能看到照片（静态原型）；通了电（服务器渲染）后，照片还能动态更新内容。

### 4.3 适用场景

- **传统服务端渲染应用**：需要SEO、首屏速度的Web应用
- **后台管理系统**：快速开发CRUD页面
- **前后端不分离项目**：前端设计师和后端开发者协作
- **邮件模板**：生成HTML格式的邮件内容
- **静态内容生成**：离线生成HTML文件

---

## 五、关键技巧

### 5.1 常用属性一览

| 属性 | 作用 | 示例 |
|---|---|---|
| `th:text` | 设置标签文本内容 | `<p th:text="${name}">默认</p>` |
| `th:utext` | 设置文本（不转义HTML） | `<p th:utext="${htmlContent}"></p>` |
| `th:value` | 设置表单值 | `<input th:value="${user.name}" />` |
| `th:href` | 设置链接地址 | `<a th:href="@{/home}">首页</a>` |
| `th:src` | 设置图片/资源地址 | `<img th:src="@{/images/logo.png}" />` |
| `th:if` | 条件渲染（满足才显示） | `<div th:if="${user != null}">欢迎</div>` |
| `th:unless` | 条件取反 | `<div th:unless="${user != null}">请登录</div>` |
| `th:each` | 循环迭代 | `<li th:each="u : ${users}" th:text="${u.name}"></li>` |
| `th:switch` / `th:case` | 多分支判断 | 类似Java的switch |
| `th:object` | 选定对象（配合`*{}`使用） | `<form th:object="${user}">` |
| `th:action` | 表单提交地址 | `<form th:action="@{/save}">` |
| `th:method` | 设置请求方法 | `<form th:method="post">` |
| `th:class` | 设置class属性 | `<div th:class="${active} ? 'active'">` |
| `th:attr` | 设置任意属性 | `<button th:attr="data-id=${id}">` |

### 5.2 表单处理

**表单绑定（推荐使用th:object）**：

```html
<form th:action="@{/user/save}" th:object="${user}" method="post">
    <!-- 使用*{...}直接访问对象属性 -->
    <input type="text" th:field="*{name}" placeholder="姓名" />
    <input type="email" th:field="*{email}" placeholder="邮箱" />
    <input type="number" th:field="*{age}" placeholder="年龄" />
    <button type="submit">提交</button>
</form>
```

> `th:field`会自动生成`id`、`name`、`value`三个属性，非常方便。

**单选框和复选框**：

```html
<!-- 单选框 -->
<input type="radio" th:field="*{gender}" value="male" /> 男
<input type="radio" th:field="*{gender}" value="female" /> 女

<!-- 复选框（绑定数组/集合） -->
<input type="checkbox" th:field="*{hobbies}" value="reading" /> 阅读
<input type="checkbox" th:field="*{hobbies}" value="music" /> 音乐
<input type="checkbox" th:field="*{hobbies}" value="sport" /> 运动

<!-- 下拉框 -->
<select th:field="*{city}">
    <option value="">请选择</option>
    <option th:each="c : ${cities}" 
            th:value="${c.id}" 
            th:text="${c.name}"></option>
</select>
```

**表单验证错误显示**：

```html
<!-- 显示全局错误 -->
<div th:if="${#fields.hasErrors('*')}" th:errors="*{*}"></div>

<!-- 显示单个字段错误 -->
<span th:if="${#fields.hasErrors('name')}" th:errors="*{name}" class="error"></span>

<!-- 字段有错误时添加样式 -->
<input type="text" th:field="*{name}" 
       th:classappend="${#fields.hasErrors('name')} ? 'input-error'" />
```

### 5.3 迭代（循环）技巧

**基础循环**：

```html
<ul>
    <li th:each="user : ${users}" th:text="${user.name}">用户名</li>
</ul>
```

**获取迭代状态**：

```html
<table>
    <tr th:each="user, iterStat : ${users}">
        <td th:text="${iterStat.index + 1}">序号</td>  <!-- 从0开始 -->
        <td th:text="${iterStat.count}">计数</td>      <!-- 从1开始 -->
        <td th:text="${user.name}">姓名</td>
        <td th:text="${iterStat.first}">是否第一个</td>
        <td th:text="${iterStat.last}">是否最后一个</td>
        <td th:text="${iterStat.size}">总条数</td>
        <td th:class="${iterStat.even} ? 'even-row'">偶数行</td>
    </tr>
</table>
```

> `iterStat`是迭代状态变量，包含index、count、size、current、even、odd、first、last等属性。

### 5.4 条件判断

**if / unless**：

```html
<!-- 条件为真时显示 -->
<div th:if="${user.age >= 18}">成年人</div>

<!-- 条件为假时显示 -->
<div th:unless="${user.age >= 18}">未成年人</div>
```

**switch / case**：

```html
<div th:switch="${user.role}">
    <p th:case="'admin'">管理员</p>
    <p th:case="'editor'">编辑</p>
    <p th:case="'user'">普通用户</p>
    <p th:case="*">未知角色</p>  <!-- 默认情况 -->
</div>
```

### 5.5 模板片段复用

**定义片段**（`fragments/header.html`）：

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head th:fragment="header">
    <meta charset="UTF-8">
    <title>我的网站</title>
    <link rel="stylesheet" th:href="@{/css/style.css}" />
</head>
<body>
    <nav th:fragment="nav">
        <a href="/">首页</a>
        <a href="/about">关于</a>
    </nav>
</body>
</html>
```

**引用片段**：

```html
<!-- insert：插入到当前标签内部 -->
<head th:insert="fragments/header :: header"></head>

<!-- replace：替换当前标签 -->
<div th:replace="fragments/header :: nav"></div>

<!-- include：只包含片段内容（不包含片段标签本身） -->
<div th:include="fragments/header :: nav"></div>
```

> **三种引用方式的区别**：
> - `th:insert`：保留宿主标签，插入片段内容
> - `th:replace`：用片段替换宿主标签
> - `th:include`：保留宿主标签，只包含片段的内容（不含片段的根标签）

### 5.6 性能优化建议

1. **开启模板缓存**：生产环境设置`spring.thymeleaf.cache=true`（默认开启）
2. **减少大对象渲染**：避免在模板中渲染过大的对象图
3. **静态资源分离**：CSS/JS放在`static`目录，使用CDN加速
4. **合理使用片段**：公共部分抽取为片段，减少重复代码
5. **避免复杂表达式**：复杂逻辑放到Controller或Service中处理
6. **使用th:inline**：在JavaScript中使用数据时，使用内联表达式

---

## 知识点总结

1. **Thymeleaf** 是一个Java服务器端**模板引擎**，主打**自然模板**理念——模板本身就是合法HTML，浏览器可直接打开预览，服务器渲染后动态替换数据。

2. **五种标准表达式**：
   - `${...}` 变量表达式：获取模型属性
   - `*{...}` 选择变量表达式：在选定对象上操作
   - `#{...}` 消息表达式：国际化消息
   - `@{...}` 链接表达式：生成URL
   - `~{...}` 片段表达式：引用模板片段

3. **常用属性**：`th:text`（文本）、`th:if`（条件）、`th:each`（循环）、`th:href`（链接）、`th:object`（选对象）、`th:field`（表单绑定）、`th:action`（表单地址）等。

4. **表单处理**：使用`th:object`绑定对象，`th:field`绑定字段，支持文本框、单选框、复选框、下拉框等各种表单元素，支持错误信息显示。

5. **模板片段**：使用`th:fragment`定义片段，`th:insert`/`th:replace`/`th:include`引用片段，实现页面公共部分复用。

6. **与Spring Boot集成**：添加`spring-boot-starter-thymeleaf`依赖即可开箱即用，模板默认放在`resources/templates`目录。

7. **相比JSP的优势**：自然模板（浏览器可预览）、属性语法（不破坏HTML）、前后端协作友好、内置缓存、Spring深度集成。
