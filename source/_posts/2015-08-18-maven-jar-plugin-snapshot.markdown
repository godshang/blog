---
title: 'maven-jar-plugin设置强制使用SNAPSHOT'
layout: post
categories: 技术
tags:
    - Java
---

## Maven的快照版本机制 ##

在使用Maven进行Java包管理时，开发阶段经常处于不稳定状态，随时需要修改并发布。我们知道，在Maven中，groupId、artifactId和version三个元素定义了一个构件的基本坐标。对于Maven来说，同样的坐标意味着同样的构件。因此，如果本地仓库中包含了某版本的构件，即使在修改后更新了远程仓库中该版本的构件，Maven并不会更新本地仓库中的构件。除非每次进行Maven构建的时候，清除本地仓库。但这种手工清除的方式较为繁琐，并不可取。当然，也可以不停的更新构件的版本号（1.1、1.2、1.3……）也可以解决这个问题，但是这要求构件的提供者和使用者都要频繁地修改POM，而且各个版本号之间只包含了微小的差异，实际上是对版本号的滥用。

为了解决本地仓库中的构件缓存问题，Maven支持快照版本机制。

使用快照时，需要在POM里构件的正式版本号后面添加“-SNAPSHOT”（注意，这里必须是大写），然后发布到远程仓库中即可。Maven在发布的过程中，会自动为构件加上时间戳，得到形如“brandstart-api-1.0.0-20150921.084134-159.jar”的构件。构件的使用者在构建自己的模块时，Maven会自动从远程仓库中下载该构件的最新版本。

通过使用快照机制，可以构件的使用者随时能够得到构件的最新可用的快照构件。

项目在正式发布上线前，就可以将构件的快照版本变更为正式版本，表示该构件的这个版本已经稳定，且只对应了唯一的构件。

## 使用快照版本构件 ##

我们知道，对于web项目，发布时所有的第三方依赖jar包会被Maven拷贝到WEB-INF/lib目录下。而对于以jar形式发布的项目，第三方jar的路径并没有强制约定的某个目录，而是在被发布的jar包中的MANIFAST文件的Class-Path部分写明第三方jar包的路径。比如：

```
Manifest-Version: 1.0
Archiver-Version: Plexus Archiver
Created-By: Apache Maven
Built-By: root
Build-Jdk: 1.6.0_31
Main-Class: com.sogou.bizdev.brandtask.app.Main
Class-Path: lib/brandstart-api-1.0.0-20150817.111150-22.jar lib/commons-logging-1.1.1.jar ...
```

但是，对于使用了快照的第三方jar，MANIFAST里的jar包名是以时间戳为后缀的，而不是SNAPSHOT。而实际上从远程仓库获取的jar却是以SNAPSHOT为后缀的。这就造成了运行时出现的类找不到的异常。

这个不一致的问题，可以通过设置maven-jar-plugin插件的useUniqueVersions属性，来强制使用SNAPSHOT。插件的具体配置示例：

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>2.3</version>
    <configuration>
        <archive>
            <addMavenDescriptor>false</addMavenDescriptor>
            <manifest>
                <mainClass>
                    com.sogou.mars.task.Launcher
                </mainClass>
                <addClasspath>true</addClasspath>
                <classpathPrefix>lib/</classpathPrefix>
                <useUniqueVersions>false</useUniqueVersions>
            </manifest>
        </archive>
    </configuration>
</plugin>
```