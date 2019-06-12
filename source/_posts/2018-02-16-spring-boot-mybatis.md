---
title: 'Spring Boot 整合 MyBatis'
layout: post
toc: true
categories: 技术
tags:
    - Spring Boot
---

MyBatis 是除 JPA 之外的另一种数据库访问选择，相对 JPA 而言，MyBatis 需要写更多的代码，相对也提供了更大的灵活性。对我个人而言，更喜欢 MyBatis 这种方式。

# 依赖配置

```
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.3.2</version>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

与之前的 JDBC/JPA 的例子一样，在 `src/main/resources/application.properties` 中配置数据源信息：

```
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=abc123
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

# 使用 MyBatis

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

创建 User 映射的操作接口 UserMapper：

```
@Mapper
public interface UserMapper {

    @Insert("INSERT INTO user (name, phone, email, create_time) VALUES ( #{name}, #{phone}, #{email}, NOW())")
    void insert(User user);

    @Select("SELECT id, name, phone, email, create_time createTime FROM user WHERE id = #{id}")
    User findById(Integer id);
}
```

MyBatis 使用上还是比较直观，除了用注解的方式在接口方法上写 SQL 外，也支持原来的 XML 方式 ，不再赘述。