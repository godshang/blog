---
title: 'Spring Data JPA 之四：使用 Specification 进行数据查询'
layout: post
toc: true
categories: 技术
tags:
    - Spring Boot
---

JPA 提供了基于 Criteria API 的查询，比 @Query 查询更灵活和方便。

# 定义一个基于 JpaSpecificationExecutor 的接口

```
public interface UserRepository extends JpaRepository<User, Integer>, JpaSpecificationExecutor<User> {
}
```

JpaSpecificationExecutor 包含了常用的查询单个对象，查询数据集合，查询分页数据集合，查询带排序参数的数据集合，查询数据的大小，这些都是常用的数据结果集，因此可以不用做其他定义即可直接使用。

```
public interface JpaSpecificationExecutor<T> {
    T findOne(Specification<T> var1);

    List<T> findAll(Specification<T> var1);

    Page<T> findAll(Specification<T> var1, Pageable var2);

    List<T> findAll(Specification<T> var1, Sort var2);

    long count(Specification<T> var1);
}
```

# 创建 SpecificationFactory 工具类

```
public class SpecificationFactory {

    public static Specification isBetween(String attribute, Date min, Date max) {
        return (root, criteriaQuery, criteriaBuilder) -> criteriaBuilder.between(root.get(attribute), min, max);
    }
}
```

查询时基于查询条件调用 SpecificationFactory 创建 Specification，完成查询：

```
public List<User> getUserBetweenCreateTime(String startDate, String endDate) throws Exception {
    SimpleDateFormat formatter = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    Date min = formatter.parse(startDate);
    Date max = formatter.parse(endDate);
    Specification spec = SpecificationFactory.isBetween("createTime", min, max);
    return userRepository.findAll(spec);
}
```