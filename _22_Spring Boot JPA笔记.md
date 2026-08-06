# Spring Boot JPA学习笔记

## 一、什么是JPA

**定义** **JPA=Java Persistence API，是Java EE的一套ORM规范，定义了Java对象与关系数据库之间数据持久化的标准接口。**

JPA是用于管理Java应用程序中**关系数据**的Java规范。它允许我们在Java对象/类与关系数据库之间访问和持久化数据，遵循**对象关系映射（ORM）**思想。JPA本身是一组接口，不提供具体实现，运行时通过**EntityManager** API来处理查询和事务，使用与平台无关的查询语言**JPQL**（Java持久查询语言）。

JPA涵盖三个领域：
- Java持久性API本身
- 对象关系元数据（通过注解或XML配置）
- `javax.persistence`包中定义的各种接口

> **生活化比喻**：JPA就像**国家制定的快递行业标准**，规定了快递怎么收、怎么运、怎么送（接口规范），但它自己不做快递业务。各家快递公司（Hibernate、EclipseLink等）按照这个标准来提供具体服务，用户不管用哪家快递，操作方式都差不多。

### 1.1 Spring Data JPA

**定义** **Spring Data JPA=Spring框架对JPA规范的封装和增强，提供了更简洁的Repository抽象，能根据方法名自动生成SQL。**

Spring Boot提供了`spring-boot-starter-data-jpa`启动器，内部使用Hibernate作为默认的JPA实现。它最大的亮点是**Repository接口**——只需定义接口、继承指定父接口，就能获得常用的CRUD方法，甚至不用写实现类。

### 1.2 JPA vs Hibernate

| JPA | Hibernate |
|---|---|
| Java**规范**（接口） | **ORM框架**（实现） |
| 不提供实现类 | 提供具体实现类 |
| 使用**JPQL**查询语言 | 使用**HQL**查询语言 |
| 定义在`javax.persistence`包 | 定义在`org.hibernate`包 |
| 可被Hibernate、EclipseLink等实现 | 是JPA的**提供者**之一 |
| 通过**EntityManager**操作 | 通过**Session**操作 |

---

## 二、核心概念与原理

### 2.1 ORM（对象关系映射）

**定义** **ORM=Object-Relational Mapping，对象关系映射，是将Java对象与数据库表进行相互转换的技术。**

ORM层位于应用程序和数据库之间，充当**适配器**角色：
- 把Java类映射成数据库表
- 把对象映射成行记录
- 把属性映射成列字段

默认情况下，持久化类名对应表名，字段名对应列名。应用程序操作的是对象，ORM自动转换成SQL操作数据库。

> **比喻**：ORM就像**翻译官**。程序员说Java语言（操作对象），数据库说SQL语言（操作表和记录），ORM在中间做翻译，两边都不用学对方的语言就能沟通。

### 2.2 JPA核心接口

#### （1）Persistence
**定义** **Persistence=包含获取EntityManagerFactory实例静态方法的工具类。**

它是JPA的入口点，通过`Persistence.createEntityManagerFactory("单元名")`创建实体管理器工厂。

#### （2）EntityManagerFactory
**定义** **EntityManagerFactory=EntityManager的工厂类，负责创建和管理多个EntityManager实例。**

它是重量级对象，一个应用程序通常只创建一个。对应关系：一个应用 → 一个EntityManagerFactory → 多个EntityManager。

> **比喻**：EntityManagerFactory就像**银行总行**，一个城市只有一家，但它可以开设很多支行（EntityManager）为客户服务。

#### （3）EntityManager
**定义** **EntityManager=JPA的核心接口，负责实体的增删改查、事务管理、创建查询等持久化操作。**

它是轻量级对象，每次请求创建一个，用完关闭。EntityManager是我们操作数据库的主要入口。

#### （4）Entity（实体）
**定义** **Entity=被JPA管理的持久化Java对象，对应数据库中的一张表。**

实体类使用`@Entity`注解标记，每个实体实例对应表中的一行记录。

#### （5）EntityTransaction
**定义** **EntityTransaction=实体事务接口，与EntityManager一一对应，管理事务的提交和回滚。**

每个EntityManager都有一个对应的EntityTransaction实例。

#### （6）Query
**定义** **Query=JPA查询接口，由JPA供应商实现，用于执行JPQL或SQL查询并返回结果。**

### 2.3 类关系图

- EntityManagerFactory → EntityManager：**一对多**（一个工厂创建多个管理器）
- EntityManager → EntityTransaction：**一对一**（每个管理器有一个事务）
- EntityManager → Query：**一对多**（一个管理器可执行多个查询）
- EntityManager → Entity：**一对多**（一个管理器可管理多个实体）

### 2.4 Spring Data JPA的Repository接口

Spring Data JPA提供了多层Repository接口，功能逐层增强：

| 接口 | 功能 |
|---|---|
| **Repository** | 最顶层接口，仅作标记，无方法 |
| **CrudRepository** | 提供基本CRUD方法（save、findById、findAll、delete等） |
| **PagingAndSortingRepository** | 继承CrudRepository，增加分页和排序 |
| **JpaRepository** | 继承PagingAndSortingRepository，增加flush、批量操作等 |
| **JpaSpecificationExecutor** | 支持动态条件查询（Specification） |

> **比喻**：Repository接口就像**不同等级的工具箱**——Repository是空箱子（只有个名分），CrudRepository是基础工具箱（有锤子、螺丝刀），JpaRepository是全套工具箱（应有尽有）。

---

## 三、使用步骤

### 3.1 创建Spring Boot JPA项目

**第一步：添加依赖**

在`pom.xml`中添加JPA和数据库驱动依赖：

```xml
<!-- Spring Data JPA -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>

<!-- 数据库驱动（以Apache Derby内存数据库为例） -->
<dependency>
    <groupId>org.apache.derby</groupId>
    <artifactId>derby</artifactId>
    <scope>runtime</scope>
</dependency>
```

> 说明：Spring Boot可自动配置H2、HSQL、Derby等**嵌入式数据库**，无需提供连接URL。

**第二步：创建实体类**

使用`@Entity`标记实体，`@Id`标记主键：

```java
package com.example.model;

import javax.persistence.Entity;
import javax.persistence.Id;

@Entity
public class UserRecord {

    @Id
    private int id;
    private String name;
    private String email;

    // 默认构造函数（JPA必须）
    public UserRecord() {}

    public int getId() { return id; }
    public void setId(int id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}
```

**第三步：创建Repository接口**

继承`CrudRepository`或`JpaRepository`，泛型参数为<实体类, 主键类型>：

```java
package com.example.repository;

import org.springframework.data.repository.CrudRepository;
import com.example.model.UserRecord;

public interface UserRepository extends CrudRepository<UserRecord, Integer> {
    // 不用写实现类！Spring会自动实现
}
```

**第四步：创建Service层**

```java
package com.example.service;

import java.util.ArrayList;
import java.util.List;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import com.example.model.UserRecord;
import com.example.repository.UserRepository;

@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    public List<UserRecord> getAllUsers() {
        List<UserRecord> userRecords = new ArrayList<>();
        userRepository.findAll().forEach(userRecords::add);
        return userRecords;
    }

    public void addUser(UserRecord userRecord) {
        userRepository.save(userRecord);
    }
}
```

**第五步：创建Controller层**

```java
package com.example.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;
import com.example.model.UserRecord;
import com.example.service.UserService;
import java.util.List;

@RestController
public class UserController {

    @Autowired
    private UserService userService;

    @RequestMapping("/")
    public List<UserRecord> getAllUser() {
        return userService.getAllUsers();
    }

    @RequestMapping(value = "/add-user", method = RequestMethod.POST)
    public void addUser(@RequestBody UserRecord userRecord) {
        userService.addUser(userRecord);
    }
}
```

**第六步：运行测试**

启动应用，使用Postman发送POST请求添加数据：

```json
{
    "id": 1,
    "name": "Tom",
    "email": "tom@gmail.com"
}
```

然后访问 `http://localhost:8080/` 查看返回的用户列表。

### 3.2 实体映射常用注解

| 注解 | 作用 |
|---|---|
| `@Entity` | 标记类为JPA实体 |
| `@Table(name = "xxx")` | 指定对应的数据库表名 |
| `@Id` | 标记主键字段 |
| `@GeneratedValue(strategy = ...)` | 主键生成策略（AUTO/IDENTITY/SEQUENCE/TABLE） |
| `@Column(name = "xxx")` | 指定对应的列名 |
| `@Transient` | 标记非持久化字段（不映射到数据库） |
| `@OneToMany` | 一对多关系映射 |
| `@ManyToOne` | 多对一关系映射 |
| `@ManyToMany` | 多对多关系映射 |
| `@JoinColumn` | 指定外键列名 |

---

## 四、为什么需要JPA

### 4.1 传统JDBC的痛点

在没有JPA/ORM之前，使用JDBC操作数据库存在以下问题：

- **大量样板代码**：每次查询都要写Connection、Statement、ResultSet，还要手动关闭资源
- **手动映射**：需要手动把ResultSet的数据封装到Java对象中
- **SQL硬编码**：SQL语句写死在Java代码里，换数据库要大量修改
- **移植性差**：不同数据库SQL方言不同，迁移成本高
- **面向过程**：操作的是表和记录，而不是对象，不符合面向对象思想

### 4.2 JPA的优势

1. **面向对象操作**：操作的是Java对象，符合面向对象思维
2. **消除样板代码**：不用写JDBC连接、手动映射等重复代码
3. **数据库无关**：使用JPQL查询，底层数据库可无缝切换
4. **自动建表**：可根据实体类自动生成DDL（建表语句）
5. **缓存支持**：一级缓存、二级缓存提升性能
6. **声明式事务**：配合Spring的`@Transactional`，事务管理简单
7. **方法名查询**：Spring Data JPA可根据方法名自动生成查询

> **生活化对比**：用JDBC就像**自己盖房子**——要买砖、买水泥、自己砌墙、自己装修，每件事都要亲力亲为。用JPA就像**买精装修的商品房**——你只需要选户型（定义实体类），开发商（JPA实现）帮你搞定一切施工细节，拎包入住。

### 4.3 JPA适用场景

- **业务复杂的应用**：对象关系复杂，需要面向对象建模
- **需要数据库移植**：可能切换不同数据库的项目
- **快速开发**：原型开发、中小项目，追求开发效率
- **非极致性能要求**：大多数业务场景性能足够

> 注意：对于超大规模数据、复杂SQL优化场景，JDBC或MyBatis可能更灵活。

---

## 五、关键技巧

### 5.1 方法名查询（命名查询）

Spring Data JPA最强大的功能之一：**根据方法名自动生成查询**。

**命名规则**：`查询动词 + By + 属性名 + 条件关键字`

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // 根据姓名查询
    List<User> findByName(String name);

    // 根据姓名和邮箱查询
    User findByNameAndEmail(String name, String email);

    // 根据姓名模糊查询
    List<User> findByNameLike(String namePattern);

    // 查询年龄大于指定值的用户
    List<User> findByAgeGreaterThan(Integer age);

    // 查询姓名不为空的用户，按年龄排序
    List<User> findByNameNotNullOrderByAgeDesc();

    // 查询前10条年龄最大的用户
    List<User> findTop10ByOrderByAgeDesc();

    // 统计某个城市的用户数
    Long countByCity(String city);

    // 判断邮箱是否存在
    boolean existsByEmail(String email);
}
```

**常用关键字**：

| 关键字 | 作用 | 示例 |
|---|---|---|
| `And` | 并且 | findByNameAndAge |
| `Or` | 或者 | findByNameOrEmail |
| `Between` | 区间 | findByAgeBetween |
| `LessThan` | 小于 | findByAgeLessThan |
| `GreaterThan` | 大于 | findByAgeGreaterThan |
| `Like` | 模糊匹配 | findByNameLike |
| `OrderBy` | 排序 | findByOrderByAgeDesc |
| `Top/N` | 限制条数 | findTop10By |
| `Count` | 统计 | countByCity |
| `Exists` | 是否存在 | existsByEmail |

### 5.2 自定义查询（@Query）

当方法名查询不够用时，使用`@Query`注解手动写JPQL：

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // JPQL查询
    @Query("SELECT u FROM User u WHERE u.name = ?1 AND u.age > ?2")
    List<User> findByNameAndAgeGreaterThan(String name, Integer age);

    // 命名参数
    @Query("SELECT u FROM User u WHERE u.city = :city")
    List<User> findByCity(@Param("city") String city);

    // 原生SQL查询
    @Query(value = "SELECT * FROM user WHERE status = ?1", nativeQuery = true)
    List<User> findByStatusNative(Integer status);

    // 更新操作（需要@Modifying和@Transactional）
    @Modifying
    @Transactional
    @Query("UPDATE User u SET u.email = ?1 WHERE u.id = ?2")
    int updateEmailById(String email, Long id);
}
```

> **注意**：更新/删除操作需要加`@Modifying`注解，并且需要事务支持。

### 5.3 分页与排序

继承`PagingAndSortingRepository`或`JpaRepository`即可获得分页能力：

```java
// 分页查询
Page<User> page = userRepository.findAll(
    PageRequest.of(0, 10, Sort.by(Sort.Direction.DESC, "age"))
);

// 获取分页信息
long totalElements = page.getTotalElements(); // 总记录数
int totalPages = page.getTotalPages(); // 总页数
List<User> content = page.getContent(); // 当前页数据
```

也可以在自定义方法中使用分页：

```java
Page<User> findByCity(String city, Pageable pageable);
```

### 5.4 实体映射技巧

**主键自增策略**：

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY) // 数据库自增
private Long id;
```

**列映射**：

```java
@Column(name = "user_name", length = 50, nullable = false, unique = true)
private String userName;
```

**一对多关系**：

```java
// 一方（用户）
@OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
private List<Order> orders;

// 多方（订单）
@ManyToOne
@JoinColumn(name = "user_id")
private User user;
```

### 5.5 性能优化建议

1. **N+1查询问题**：使用`@EntityGraph`或`fetch join`避免关联查询时的N+1问题
2. **合理使用缓存**：利用一级缓存（EntityManager级别）和二级缓存（应用级别）
3. **批量操作**：批量保存时使用`saveAll()`，设置合理的batch size
4. **延迟加载**：关联关系默认使用懒加载（`FetchType.LAZY`），避免加载不必要数据
5. **索引优化**：在常用查询字段上添加索引（`@Index`）
6. **避免全表查询**：`findAll()`在数据量大时慎用，一定要分页

---

## 知识点总结

1. **JPA** 是Java持久化**规范**（接口），不是框架；**Hibernate** 是JPA的**实现**（ORM框架）。**Spring Data JPA** 是Spring对JPA的进一步封装，提供了Repository抽象。

2. **ORM（对象关系映射）**：将Java对象映射为数据库表，自动完成对象与记录的转换，让开发者用面向对象方式操作数据库。

3. **JPA核心接口**：Persistence（入口）→ EntityManagerFactory（工厂）→ EntityManager（操作器）→ Entity（实体）→ Query（查询）→ EntityTransaction（事务）。

4. **Spring Data JPA Repository**：继承`CrudRepository`获得基本CRUD，继承`JpaRepository`获得分页+排序+批量等增强功能，**无需写实现类**。

5. **查询方式**：
   - **方法名查询**：按命名规则写方法名，自动生成SQL（findByNameAndAge...）
   - **@Query查询**：写JPQL或原生SQL，灵活复杂查询
   - **分页查询**：通过`Pageable`参数实现分页和排序

6. **实体映射注解**：`@Entity`、`@Id`、`@GeneratedValue`、`@Column`、`@OneToMany`、`@ManyToOne`、`@ManyToMany`等。

7. **JPA优势**：面向对象、消除样板代码、数据库无关、自动建表、声明式事务、开发效率高。

8. **使用步骤**：添加依赖 → 创建实体类（@Entity）→ 定义Repository接口 → 编写Service/Controller → 运行测试。
