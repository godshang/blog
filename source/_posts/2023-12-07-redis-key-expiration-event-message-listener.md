---
title: 'KeyExpirationEventMessageListener的一个小坑'
layout: post
categories: 技术
toc: true
tags:
    - Redis
---

最近公司项目使用的Redis迁移到百度公有云，迁移后发现启动不起来，报错如下：

```
2023-12-07T13:34:37.714994466+08:00 stdout F Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
2023-12-07T13:34:37.721703211+08:00 stdout F 2023-12-07 13:34:37.721 ERROR [/] 1 --- [           main] o.s.boot.SpringApplication               : [] [UID:] Application run failed
2023-12-07T13:34:37.721713715+08:00 stdout F 
2023-12-07T13:34:37.721718341+08:00 stdout F org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'redisKeyExpirationListener' defined in URL [jar:file:/hawkeye-msg.jar!/BOOT-INF/classes!/com/xxx/femonitor/consumer/RedisKeyExpirationListener.class]: Invocation of init method failed; nested exception is org.springframework.data.redis.RedisSystemException: Error in execution; nested exception is io.lettuce.core.RedisCommandExecutionException: ERR this user has no permissions to run the config SET command
2023-12-07T13:34:37.721721076+08:00 stdout F 	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1769)
2023-12-07T13:34:37.721723359+08:00 stdout F 	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:592)
2023-12-07T13:34:37.721725452+08:00 stdout F 	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:514)
2023-12-07T13:34:37.72172771+08:00 stdout F 	at org.springframework.beans.factory.support.AbstractBeanFactory.lambda$doGetBean$0(AbstractBeanFactory.java:321)
2023-12-07T13:34:37.72172991+08:00 stdout F 	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:226)
2023-12-07T13:34:37.721732317+08:00 stdout F 	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:319)
2023-12-07T13:34:37.721736367+08:00 stdout F 	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:199)
2023-12-07T13:34:37.721738675+08:00 stdout F 	at org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons(DefaultListableBeanFactory.java:863)
2023-12-07T13:34:37.721741049+08:00 stdout F 	at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:878)
2023-12-07T13:34:37.721743319+08:00 stdout F 	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:550)
2023-12-07T13:34:37.721745571+08:00 stdout F 	at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:141)
2023-12-07T13:34:37.721748047+08:00 stdout F 	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:744)
2023-12-07T13:34:37.721750151+08:00 stdout F 	at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:391)
2023-12-07T13:34:37.721752544+08:00 stdout F 	at org.springframework.boot.SpringApplication.run(SpringApplication.java:312)
2023-12-07T13:34:37.721754637+08:00 stdout F 	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1215)
2023-12-07T13:34:37.721756742+08:00 stdout F 	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1204)
2023-12-07T13:34:37.721759114+08:00 stdout F 	at com.xxx.femonitor.ErrormessageConsumerApplication.main(ErrormessageConsumerApplication.java:10)
2023-12-07T13:34:37.721761878+08:00 stdout F 	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
2023-12-07T13:34:37.721770424+08:00 stdout F 	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
2023-12-07T13:34:37.721772679+08:00 stdout F 	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
2023-12-07T13:34:37.721774794+08:00 stdout F 	at java.lang.reflect.Method.invoke(Method.java:498)
2023-12-07T13:34:37.721776898+08:00 stdout F 	at org.springframework.boot.loader.MainMethodRunner.run(MainMethodRunner.java:48)
2023-12-07T13:34:37.721779039+08:00 stdout F 	at org.springframework.boot.loader.Launcher.launch(Launcher.java:87)
2023-12-07T13:34:37.721781165+08:00 stdout F 	at org.springframework.boot.loader.Launcher.launch(Launcher.java:50)
2023-12-07T13:34:37.721784829+08:00 stdout F 	at org.springframework.boot.loader.JarLauncher.main(JarLauncher.java:51)
2023-12-07T13:34:37.721787735+08:00 stdout F Caused by: org.springframework.data.redis.RedisSystemException: Error in execution; nested exception is io.lettuce.core.RedisCommandExecutionException: ERR this user has no permissions to run the config SET command
2023-12-07T13:34:37.721789884+08:00 stdout F 	at org.springframework.data.redis.connection.lettuce.LettuceExceptionConverter.convert(LettuceExceptionConverter.java:54)
2023-12-07T13:34:37.721792012+08:00 stdout F 	at org.springframework.data.redis.connection.lettuce.LettuceExceptionConverter.convert(LettuceExceptionConverter.java:52)
2023-12-07T13:34:37.721794098+08:00 stdout F 	at org.springframework.data.redis.connection.lettuce.LettuceExceptionConverter.convert(LettuceExceptionConverter.java:41)
2023-12-07T13:34:37.721796184+08:00 stdout F 	at org.springframework.data.redis.PassThroughExceptionTranslationStrategy.translate(PassThroughExceptionTranslationStrategy.java:44)
2023-12-07T13:34:37.72179832+08:00 stdout F 	at org.springframework.data.redis.FallbackExceptionTranslationStrategy.translate(FallbackExceptionTranslationStrategy.java:42)
2023-12-07T13:34:37.721806298+08:00 stdout F 	at org.springframework.data.redis.connection.lettuce.LettuceConnection.convertLettuceAccessException(LettuceConnection.java:268)
2023-12-07T13:34:37.721808656+08:00 stdout F 	at org.springframework.data.redis.connection.lettuce.LettuceServerCommands.convertLettuceAccessException(LettuceServerCommands.java:571)
2023-12-07T13:34:37.721810797+08:00 stdout F 	at org.springframework.data.redis.connection.lettuce.LettuceServerCommands.setConfig(LettuceServerCommands.java:332)
2023-12-07T13:34:37.721812867+08:00 stdout F 	at org.springframework.data.redis.connection.DefaultedRedisConnection.setConfig(DefaultedRedisConnection.java:1204)
2023-12-07T13:34:37.721814958+08:00 stdout F 	at org.springframework.data.redis.listener.KeyspaceEventMessageListener.init(KeyspaceEventMessageListener.java:92)
2023-12-07T13:34:37.721817123+08:00 stdout F 	at org.springframework.data.redis.listener.KeyspaceEventMessageListener.afterPropertiesSet(KeyspaceEventMessageListener.java:137)
2023-12-07T13:34:37.721819348+08:00 stdout F 	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.invokeInitMethods(AbstractAutowireCapableBeanFactory.java:1828)
2023-12-07T13:34:37.721821562+08:00 stdout F 	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1765)
2023-12-07T13:34:37.721823871+08:00 stdout F 	... 24 common frames omitted
2023-12-07T13:34:37.721826104+08:00 stdout F Caused by: io.lettuce.core.RedisCommandExecutionException: ERR this user has no permissions to run the config SET command
2023-12-07T13:34:37.72182828+08:00 stdout F 	at io.lettuce.core.ExceptionFactory.createExecutionException(ExceptionFactory.java:135)
2023-12-07T13:34:37.721830427+08:00 stdout F 	at io.lettuce.core.ExceptionFactory.createExecutionException(ExceptionFactory.java:108)
2023-12-07T13:34:37.721832645+08:00 stdout F 	at io.lettuce.core.protocol.AsyncCommand.completeResult(AsyncCommand.java:120)
2023-12-07T13:34:37.721834744+08:00 stdout F 	at io.lettuce.core.protocol.AsyncCommand.complete(AsyncCommand.java:111)
2023-12-07T13:34:37.721840479+08:00 stdout F 	at io.lettuce.core.protocol.CommandHandler.complete(CommandHandler.java:646)
2023-12-07T13:34:37.72184299+08:00 stdout F 	at io.lettuce.core.protocol.CommandHandler.decode(CommandHandler.java:604)
2023-12-07T13:34:37.721845314+08:00 stdout F 	at io.lettuce.core.protocol.CommandHandler.channelRead(CommandHandler.java:556)
2023-12-07T13:34:37.721847524+08:00 stdout F 	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:379)
2023-12-07T13:34:37.721849604+08:00 stdout F 	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:365)
2023-12-07T13:34:37.721851665+08:00 stdout F 	at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:357)
2023-12-07T13:34:37.721853804+08:00 stdout F 	at io.netty.channel.DefaultChannelPipeline$HeadContext.channelRead(DefaultChannelPipeline.java:1410)
2023-12-07T13:34:37.721855949+08:00 stdout F 	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:379)
2023-12-07T13:34:37.721857989+08:00 stdout F 	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:365)
2023-12-07T13:34:37.721860096+08:00 stdout F 	at io.netty.channel.DefaultChannelPipeline.fireChannelRead(DefaultChannelPipeline.java:919)
2023-12-07T13:34:37.721862169+08:00 stdout F 	at io.netty.channel.epoll.AbstractEpollStreamChannel$EpollStreamUnsafe.epollInReady(AbstractEpollStreamChannel.java:792)
2023-12-07T13:34:37.721864248+08:00 stdout F 	at io.netty.channel.epoll.EpollEventLoop.processReady(EpollEventLoop.java:475)
2023-12-07T13:34:37.721866343+08:00 stdout F 	at io.netty.channel.epoll.EpollEventLoop.run(EpollEventLoop.java:378)
2023-12-07T13:34:37.721868401+08:00 stdout F 	at io.netty.util.concurrent.SingleThreadEventExecutor$4.run(SingleThreadEventExecutor.java:989)
2023-12-07T13:34:37.721870478+08:00 stdout F 	at io.netty.util.internal.ThreadExecutorMap$2.run(ThreadExecutorMap.java:74)
2023-12-07T13:34:37.721872554+08:00 stdout F 	at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30)
2023-12-07T13:34:37.721874997+08:00 stdout F 	at java.lang.Thread.run(Thread.java:748)
2023-12-07T13:34:37.721876982+08:00 stdout F 
```

大致意思是`RedisKeyExpirationListener`没有权限执行 config set 命令。

咨询运维后，果然新的集群不支持 config set 命令了，这是管理命令，业务上不应该对此有依赖。不过`RedisKeyExpirationListener`代码里并没有使用 config set 命令，代码如下：

```java
@Slf4j
@Component
public class RedisKeyExpirationListener extends KeyExpirationEventMessageListener {

    public RedisKeyExpirationListener(RedisMessageListenerContainer listenerContainer) {
        super(listenerContainer);
    }

    public void onMessage(Message message, byte[] pattern) {
        String expiredKey = message.toString();
        log.info("redis key过期：{}", expiredKey);
        
        // 过期键的业务处理
        // ......
    }
}
```

大致意思是监听到Redis的过期键后进行的一些业务操作。

`RedisKeyExpirationListener`继承自`org.springframework.data.redis.listener.KeyExpirationEventMessageListener`，代码如下：

```java
public class KeyExpirationEventMessageListener extends KeyspaceEventMessageListener implements ApplicationEventPublisherAware {
    private static final Topic KEYEVENT_EXPIRED_TOPIC = new PatternTopic("__keyevent@*__:expired");
    @Nullable
    private ApplicationEventPublisher publisher;

    public KeyExpirationEventMessageListener(RedisMessageListenerContainer listenerContainer) {
        super(listenerContainer);
    }

    protected void doRegister(RedisMessageListenerContainer listenerContainer) {
        listenerContainer.addMessageListener(this, KEYEVENT_EXPIRED_TOPIC);
    }

    protected void doHandleMessage(Message message) {
        this.publishEvent(new RedisKeyExpiredEvent(message.getBody()));
    }

    protected void publishEvent(RedisKeyExpiredEvent event) {
        if (this.publisher != null) {
            this.publisher.publishEvent(event);
        }

    }

    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.publisher = applicationEventPublisher;
    }
}
```

`org.springframework.data.redis.listener.KeyExpirationEventMessageListener`又继承自`org.springframework.data.redis.listener.KeyspaceEventMessageListener`，代码如下：

```java
public abstract class KeyspaceEventMessageListener implements MessageListener, InitializingBean, DisposableBean {
    private static final Topic TOPIC_ALL_KEYEVENTS = new PatternTopic("__keyevent@*");
    private final RedisMessageListenerContainer listenerContainer;
    private String keyspaceNotificationsConfigParameter = "EA";

    public KeyspaceEventMessageListener(RedisMessageListenerContainer listenerContainer) {
        Assert.notNull(listenerContainer, "RedisMessageListenerContainer to run in must not be null!");
        this.listenerContainer = listenerContainer;
    }

    public void onMessage(Message message, @Nullable byte[] pattern) {
        if (message != null && !ObjectUtils.isEmpty(message.getChannel()) && !ObjectUtils.isEmpty(message.getBody())) {
            this.doHandleMessage(message);
        }
    }

    protected abstract void doHandleMessage(Message var1);

    public void init() {
        if (StringUtils.hasText(this.keyspaceNotificationsConfigParameter)) {
            RedisConnection connection = this.listenerContainer.getConnectionFactory().getConnection();

            try {
                Properties config = connection.getConfig("notify-keyspace-events");
                if (!StringUtils.hasText(config.getProperty("notify-keyspace-events"))) {
                    connection.setConfig("notify-keyspace-events", this.keyspaceNotificationsConfigParameter);
                }
            } finally {
                connection.close();
            }
        }

        this.doRegister(this.listenerContainer);
    }

    protected void doRegister(RedisMessageListenerContainer container) {
        this.listenerContainer.addMessageListener(this, TOPIC_ALL_KEYEVENTS);
    }

    public void destroy() throws Exception {
        this.listenerContainer.removeMessageListener(this);
    }

    public void setKeyspaceNotificationsConfigParameter(String keyspaceNotificationsConfigParameter) {
        this.keyspaceNotificationsConfigParameter = keyspaceNotificationsConfigParameter;
    }

    public void afterPropertiesSet() throws Exception {
        this.init();
    }
}
```

可以看到，在`init`方法中调用了`setConfig`方法，设置了`notify-keyspace-events`参数为`keyspaceNotificationsConfigParameter`，`keyspaceNotificationsConfigParameter`的默认是`EA`。关于`notify-keyspace-events`参数，可以参考Redis[官方文档](https://redis.io/docs/manual/keyspace-notifications/)。

这个其实Spring替我们做了Redis的配置，我估计初衷是避免Redis Server没配该参数时导致监听不生效，但对于生产环境直接在客户端进行配置，可能不是一个太好的方案。

在`init`方法中，其实是先判断了`keyspaceNotificationsConfigParameter`不为空才`setConfig`，同时也有一个`setKeyspaceNotificationsConfigParameter`方法，因此我们可以在构造自己的listener时将`keyspaceNotificationsConfigParameter`设置为空，这样就不会执行 config set 命令了。

因此，我们去掉`RedisKeyExpirationListener`上的`@Component`注解，改为自己进行Bean的创建，代码如下：

```java
@Configuration
public class RedisConfig {

    @Bean
    RedisMessageListenerContainer redisMessageListenerContainer(RedisConnectionFactory connectionFactory) {
        RedisMessageListenerContainer redisMessageListenerContainer = new RedisMessageListenerContainer();
        redisMessageListenerContainer.setConnectionFactory(connectionFactory);
        return redisMessageListenerContainer;
    }

    @Bean
    RedisKeyExpirationListener redisKeyExpirationListener(RedisMessageListenerContainer redisMessageListenerContainer) {
        RedisKeyExpirationListener redisKeyExpirationListener = new RedisKeyExpirationListener(redisMessageListenerContainer);
        redisKeyExpirationListener.setKeyspaceNotificationsConfigParameter("");
        return redisKeyExpirationListener;
    }
}
```

改完再启动项目就没问题了。