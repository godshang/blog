---
title: 'Spring Data JPA 之一：基本配置'
layout: post
toc: true
categories: 技术
tags:
    - Spring Boot
---

在实际开发中，对数据库的操作无非“增删改查”。使用 JdbcTemplate 进行数据库操作，需要写大量类似而鼓噪的语句来完成业务逻辑。

这里介绍一种大量简化数据库操作的技术，Spring Data JPA，几乎不需要写关于数据库访问的代码，基本的 CRUD 功能就自动替你完成了。

# 依赖配置

在 `pom.xml` 中添加 JPA 的依赖：

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

在 `src/main/resources/application.properties` 中配置数据源信息：

# 数据源配置

```
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=abc123
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

# 创建表与模型

同样使用测试表 `user`：

```
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(64) NOT NULL,
  `phone` varchar(64) NOT NULL,
  `email` varchar(128) DEFAULT NULL,
  `create_time` timestamp NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

创建相应的模型：

```
@Entity
@Table(name = "user")
@Data
public class User {

    @Id
    @GeneratedValue
    private Integer id;
    private String name;
    private String phone;
    private String email;
    private Date createTime;
}
```

`@Entity` 是一个必选的注解，声明这个类对应了一个数据库表。

`@Table` 是一个可选的注解，声明了这个类对应了哪个数据库表。如果没有指定，则表名和类的名称保持一样。

`@ID` 注解表明了记录的唯一标识。而 `GeneratedValue` 注解则表示 `ID` 自动生成。

其余属性未使用任何注解，JPA 会自动按名称将属性与数据库中的列对应起来。

# 创建操作数据库的 Repository 对象

```
public interface UserRepository extends JpaRepository<User, Integer> {

    void deleteByName(String name);
}
```

看下 JpaRepository 的代码：

```
@NoRepositoryBean
public interface JpaRepository<T, ID extends Serializable> extends PagingAndSortingRepository<T, ID>, QueryByExampleExecutor<T> {
    List<T> findAll();

    List<T> findAll(Sort var1);

    List<T> findAll(Iterable<ID> var1);

    <S extends T> List<S> save(Iterable<S> var1);

    void flush();

    <S extends T> S saveAndFlush(S var1);

    void deleteInBatch(Iterable<T> var1);

    void deleteAllInBatch();

    T getOne(ID var1);

    <S extends T> List<S> findAll(Example<S> var1);

    <S extends T> List<S> findAll(Example<S> var1, Sort var2);
}
```

JpaRepository 继承自 PagingAndSortingRepository 和 QueryByExampleExecutor，因此除了简单的增删改查能力外，还具备了分页、排序、自定义维度查询等方面的能力。

除了这些已经提供顾得方法外，还可以自定义数据库操作方法，例如 `deleteByName`，JPA 会根据方法中的属性名自动拼装成实际执行的 SQL 。

# 使用 Repository 访问数据库

```
public interface UserService {

    void create(String name, String phone, String email);

    void deleteByName(String name);

    List<User> getAllUsers();

    User getUserById(Integer id);
}

@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public void create(String name, String phone, String email) {
        User user = new User();
        user.setName(name);
        user.setPhone(phone);
        user.setEmail(email);
        user.setCreateTime(new Date());
        userRepository.save(user);
    }

    @Override
    public void deleteByName(String name) {
        userRepository.deleteByName(name);
    }

    @Override
    public List<User> getAllUsers() {
        return userRepository.findAll();
    }

    @Override
    public User getUserById(Integer id) {
        return userRepository.findOne(id);
    }
}
```

以上就是一个简单的 JPA 使用例子。