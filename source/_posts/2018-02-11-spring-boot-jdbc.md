---
title: 'Spring Boot 中使用 JdbcTemplate 访问数据库'
layout: post
toc: true
categories: 技术
tags:
    - Spring Boot
---

Java 应用中数据库访问是一个最常见，也是最基本的技术。一般来说，我们在 Spring 应用中进行数据库访问，需要大量的 Spring 配置。在 Spring Boot 中一切都变得十分简单。

本文以一个简单的例子介绍 Spring Boot 中的数据源配置和通过 JdbcTemplate 进行数据库操作。

# 依赖配置

首先，为了连接数据库需要引入 jdbc 支持，在 `pom.xml` 中加入如下配置：

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

以连接 MySQL 为例，引入 MySQL 连接驱动的依赖，在 `pom.xml` 中加入：

```
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

# 数据源配置

在 `src/main/resources/application.properties` 中配置数据源信息：

```
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=abc123
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

# 创建表与模型

我们首先在 MySQL 中创建一个测试用的表：

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

然后创建一个对应 Java 模型：

```
@Data
public class User {

    private Integer id;
    private String name;
    private String phone;
    private String email;
    private Date createTime;
}
```

这里使用了 `lombok` 简化了 `getter/setter` 方法。

# 使用 JdbcTemplate 操作数据库

我们定义一个含有数据库操作的抽象接口：

```
public interface UserService {

    void create(String name, String phone, String email);

    void deleteByName(String name);

    List<User> getAllUsers();

    User getUserById(Integer id);
}
```

Spring 的 JdbcTemplate 是自动配置的，你可以直接使用 @Autowired 来注入到你自己的bean中来使用。

```
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    public void create(String name, String phone, String email) {
        jdbcTemplate.update("INSERT INTO user (name, phone, email, create_time) VALUES (?, ?, ?, NOW())", name, phone, email);
    }

    public void deleteByName(String name) {
        jdbcTemplate.update("DELETE FROM user WHERE `name` = ?", name);
    }

    public List<User> getAllUsers() {
        return jdbcTemplate.query("SELECT * FROM user", new RowMapper<User>() {
            @Override
            public User mapRow(ResultSet resultSet, int i) throws SQLException {
                User user = new User();
                user.setId(resultSet.getInt("id"));
                user.setName(resultSet.getString("name"));
                user.setPhone(resultSet.getString("phone"));
                user.setEmail(resultSet.getString("email"));
                user.setCreateTime(resultSet.getDate("create_time"));
                return user;
            }
        });
    }

    public User getUserById(Integer id) {
        return jdbcTemplate.queryForObject("SELECT * FROM user WHERE id = ?", new Object[] { id }, User.class);
    }
}
```
# 测试

创建一个针对 UserService 的单元测试用例，验证上述操作的正确性：

```
@RunWith(SpringRunner.class)
@SpringBootTest
public class UserServiceTest {

    @Autowired
    private UserService userService;

    @Test
    public void test() {
        userService.create("alan", "13901234567", "alan@gmail.com");

        List<User> list = userService.getAllUsers();
        Assert.assertEquals(1, list.size());
        Assert.assertEquals("alan", list.get(0).getName());
        Assert.assertEquals("13901234567", list.get(0).getPhone());
        Assert.assertEquals("alan@gmail.com", list.get(0).getEmail());

        userService.deleteByName("alan");
        list = userService.getAllUsers();
        Assert.assertEquals(0, list.size());
    }
}
```

通过上面这个简单的例子，可以看到在 Spring Boot 中访问数据库是一件十分简单的事情，只需要进行简单的配置然后实现相关业务逻辑即可。