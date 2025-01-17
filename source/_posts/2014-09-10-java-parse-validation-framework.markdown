---
title: '常用Java解析与Bean验证框架性能的简单比较'
layout: post
categories: 技术
tags:
    - Java
---

## XML解析 ##

Java在解析XML方面框架比较多，有通过DOM方式的，也有直接将XML映射到Java Bean的。DOM方式需要编写大量遍历DOM树、提取数据的代码，在XML文件变动或者新增XML文件时，需要做大量的编码工作，十分繁琐。从XML映射到Bean的方式则比较简单，通过XPath或者注解的方式，配置解析规则即可。下面简单比较了JAXB和Digester的性能。

```
+----------+------------+----------------------------+----------------------------+------------------------------+
| Approach | Data Size  |   Time (With Schema Val)   | Time (Without Schema Val)  |         Dependency           |
+----------+------------+----------------------------+----------------------------+------------------------------+
|   JAXB   |    1951    |          307ms             |         137ms              |             jdk              |
+----------+------------+----------------------------+----------------------------+------------------------------+
| Digester |    1951    |         2427ms             |         2087ms             |  commons-digester3-3.2.jar   |
|          |            |                            |                            |  commons-logging-1.1.1.jar   |
|          |            |                            |                            |  commons-beanutils-1.8.3.jar |
|          |            |                            |                            |  log4j-1.2.17.jar            |
+----------+------------+----------------------------+----------------------------+------------------------------+
```

## JSON解析 ##

Json方面的解析选择更多了，这里简单比较了常用的JSON-LIB、GSON.

```
+-----------+------------+------------+-----------------------------------+
| Approach  | Data Size  |   Time     |     Dependency                    |
+-----------+------------+------------+-----------------------------------+
|   Gson    |    1047    |   240ms    |    gson-1.4.jar                   |
+-----------+------------+------------+-----------------------------------+
|  JSON-LIB |    1047    |   420ms    |    json-lib-2.3.jar               |
|           |            |            |    ezmorph-1.0.6.jar              |
|           |            |            |    commons-lang-2.6.jar           |
|           |            |            |    commons-logging-1.1.1.jar      |
|           |            |            |    commons-collections-3.2.1.jar  |
|           |            |            |    commons-beanutils-1.8.3.jar    |
+-----------+------------+------------+-----------------------------------+
```

可以看到Gson的性能较JSON-LIB更忧。

现在另一个Json的解析框架Jackson，性能较Gson更为优秀。jackson采用流式的处理方式，性能较gson更强，特别随着数据集的增大，耗时并未有明显上升。

```
数据集     Gson耗时      Jackson耗时
10w         1366            138
20w         2720            165
30w         4706            332
40w         9526            317
50w         本机OOM         363
```

## Bean验证 ##

```
    方案           数据集大小     验证时间        配置方式            支持自定义验证规则          依赖
    Oval            1951         163ms        XML/Annotation          支持            oval-1.84.jar
                                                                                      com.thoughtworks.xstream.jar
                                                                                      （解析XML配置）
    
    Hibernate       1951         423ms        XML/Annotation          支持            validation-api-1.0.0.GA.jar
    Validator                                                                         hibernate-validator-4.3.2.Final.jar
                                                                                      jboss-logging-3.1.0.CR2.jar
```