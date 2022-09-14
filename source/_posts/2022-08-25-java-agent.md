---
title: 'Java Agent 技术'
layout: post
categories: 技术
toc: true
tags:
    - Java
---

# Instrumentation

`Instrumentation`是JDK 1.6的一个新特性，通过`java.lang.instroment`包可以实现一个独立于应用程序的Agent程序，能够替换和修改类的定义。有了这样的功能，开发者可以实现灵活的运行时虚拟机监控和Java类增加，实际上提供了一种虚拟机级别的AOP实现方式。

# Java Agent Demo

下面介绍通过`java.lang.instroment`编写Agent的一般方法。

## 实现Agent启动方法

Java Agent支持目标JVM启动时加载，也支持在目标JVM运行时加载，这两种不同的加载模式会使用不同的入口函数，如果需要在目标JVM启动的同时加载Agent，那么可以选择实现下面的方法：

```
[1] public static void premain(String agentArgs, Instrumentation inst);
[2] public static void premain(String agentArgs);
```

JVM将首先寻找[1]，如果没有发现[1]，再寻找[2]。

如果希望在目标JVM运行时加载Agent，则需要实现下面的方法：

```
[1] public static void agentmain(String agentArgs, Instrumentation inst);
[2] public static void agentmain(String agentArgs);
```

这两组方法的第一个参数`agentArgs`是随同`–javaagent`一起传入的程序参数，如果这个字符串代表了多个参数，就需要自己解析这些参数。`instrumentation`是`Instrumentation`类型的对象，是JVM自动传入的，我们可以拿这个参数进行类增强等操作。

```java
public class Agent {

    public static void premain(String agentArgs, Instrumentation instrumentation) {
        System.out.println("Agent premain start ...");
        System.out.println("Agent args: " + agentArgs);
        instrumentation.addTransformer(new AppInitTramsformer());
    }

    public static void agentmain(String agentArgs, Instrumentation instrumentation) {
        System.out.println("Agent agentmain start ...");
        System.out.println("Agent args: " + agentArgs);
        instrumentation.addTransformer(new AppInitTramsformer());
    }
}
```

## 实现Transformer

在`Instrumentation`接口中，通过`addTransformer`方法来增加一个类转换器，类转换器由类`ClassFileTransformer`接口实现。`ClassFileTransformer`接口中唯一的方法`transform`用于实现类转换，当类被加载的时候，就会调用`transform`方法，进行类转换。在运行时，我们可以通过`Instrumentation`的`redefineClasses`方法进行类重定义。实现时注意不要增加、删除或者重命名字段和方法，改变方法的签名或者类的继承关系。

修改字节码可以借助`ASM`、`javassist`、`bytebuddy`等工具。下例中使用了`javassist`。

```java
public class AppInitTramsformer implements ClassFileTransformer {

    private static final String INJECTED_CLASS = "com.iqiyi.mbd.qiyihao.agent";

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        String realClassName = className.replace("/", ".");
        if (realClassName.equals(INJECTED_CLASS)) {
            System.out.println("拦截到的类名：" + realClassName);
            CtClass ctClass;
            try {
                ClassPool classPool = ClassPool.getDefault();
                ctClass = classPool.get(realClassName);

                CtMethod[] declaredMethods = ctClass.getDeclaredMethods();
                for (CtMethod method : declaredMethods) {
                    System.out.println(method.getName() + "方法被拦截");
                    method.addLocalVariable("time", CtClass.longType);
                    method.insertBefore("System.out.println(\"---开始执行---\");");
                    method.insertBefore("time = System.currentTimeMillis();");
                    method.insertAfter("System.out.println(\"---结束执行---\");");
                    method.insertAfter("System.out.println(\"运行耗时: \" + (System.currentTimeMillis() - time));");
                }
                return ctClass.toBytecode();
            } catch (Throwable e) {
                System.out.println(e.getMessage());
                e.printStackTrace();
            }
        }
        return new byte[0];
    }
}
```

## MANIFEST.MF 文件

编写的Agent如何被外部应用程序知晓呢？依靠的是`MANIFEST.MF`文件。文件的具体路径是：`src/main/resources/META-INF/MANIFEST.MF`。

```
Manifest-Version: 1.0
Premain-Class: com.iqiyi.mbd.qiyihao.agent.Agent
Agent-Class: com.iqiyi.mbd.qiyihao.agent.Agent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

`MANIFEST.MF`文件`Premain-Class`对应`premain`入口即启动时加载Agent，`Agent-Class`对应`agentmain`入口即运行时加载Agent。`Can-Redefine-Classes`表示是否能重定义此代理所需的类，默认值是false。`Can-Retransform-Classes`表示是否能重转换此代理所需的类，默认值是false。

除了直接创建`MANIFEST.MF`文件，也可以在Maven中配置编译打包插件。

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.2.0</version>
    <configuration>
        <archive>
            <!--自动添加META-INF/MANIFEST.MF -->
            <manifest>
                <addClasspath>true</addClasspath>
            </manifest>
            <manifestEntries>
                <Menifest-Version>1.0</Menifest-Version>
                <Premain-Class>com.iqiyi.mbd.qiyihao.agent.Agent</Premain-Class>
                <Agent-Class>com.iqiyi.mbd.qiyihao.agent.Agent</Agent-Class>
                <Can-Redefine-Classes>true</Can-Redefine-Classes>
                <Can-Retransform-Classes>true</Can-Retransform-Classes>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```

## 加载Agent

如果是启动时进行静态加载，需要在启动参数中增加`-javaagent`参数：

```java
public class AppMain {

    /**
     * -javaagent:E:\github\java-agent\agent\target\agent-1.0-SNAPSHOT.jar=hello
     * @param args
     */
    public static void main(String[] args) {
        AppInit appInit = new AppInit();
        appInit.init();
    }
}
```

如果是运行时进行动态加载，可参考如下代码：

```java
public class AttachMain {

    public static void main(String[] args) {
        for (VirtualMachineDescriptor vmd : VirtualMachine.list()) {
            System.out.println(vmd.displayName());
            if (vmd.displayName().endsWith("AttachMain")) {
                try {
                    VirtualMachine vm = VirtualMachine.attach(vmd.id());
                    vm.loadAgent("E:\\github\\java-agent\\agent\\target\\agent-1.0-SNAPSHOT.jar=hello");
                    vm.detach();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
        AppInit appInit = new AppInit();
        appInit.init();
    }
}
```

# Java Agent 原理

`Instrumentation`的底层实现 依赖于`JVMTI`，也就是`JVM Tool Interface`。

## JVMTI

`JVMTI（JVM Tool Interface）`是Java虚拟计对外提供的Native编程接口，是JVM暴露出来给用户扩展使用的接口集合。通过`JVMTI`外部进程可以获取到运行时JVM的诸多信息，比如线程、GC等。`JVMTI`是基于事件驱动的，JVM每执行一定的逻辑就会触发一些事件的回调接口，通过这些回调接口，用户可以自行扩展实现自己的逻辑。

## JVMTI Agent

实现了`JVMTI`的客户端程序称之为agent，它其实就是利用`JVMTI`暴露出来的接口实现用户自行的逻辑。在`JVMTI Agent`中主要有三个方法：
* `Agent_OnLoad`方法，如果agent在启动时加载，就执行这个方法
* `Agent_OnAttach`方法，如果agent不是在启动的时候加载的，是我们先attach到目标线程上，然后对对应的目标进程发送load命令来加载agent，在加载过程中调用Agent_OnAttach函数
* `Agent_OnUnload`方法，在agent做卸载掉时候调用

如何使用C++开发Agent，可以参考[这篇文章](https://cloud.tencent.com/developer/article/1574212)。

## Instrument

`JVMTI`是一套Native接口，在Java SE 5之前，要实现一个Agent只能通过编写Native代码来实现。从Java SE 5开始，可以使用Java的`Instrumentation`接口来编写Agent。无论是通过Native的方式还是通过Java Instrumentation接口的方式来编写Agent，它们的工作都是借助JVMTI来进行完成。

`JPLISAgent`全名是`Java Programming Language Instrumentation Services Agent`，它是一种特殊的`JVMTI Agent`，作用是初始化所有通过`Instrumentation`接口编写的Agent，并且也承担着通过`JVMTI`实现`Instrumentation`中暴露API的责任。



# Reference

1. https://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html
2. https://cloud.tencent.com/developer/article/1574212
3. https://xz.aliyun.com/t/10186
4. https://juejin.cn/post/7086026013498408973
5. https://jueee.github.io/2020/08/2020-08-26-Arthas%E4%B9%8B%E7%83%AD%E6%9B%B4%E6%96%B0%E5%8E%9F%E7%90%86%EF%BC%8C%E5%B9%B6%E5%AE%9E%E7%8E%B0%E7%AE%80%E6%98%93%E7%89%88%E7%83%AD%E6%9B%B4%E6%96%B0%E5%8A%9F%E8%83%BD/
6. https://www.overops.com/blog/double-agent-java-vs-native-agents/