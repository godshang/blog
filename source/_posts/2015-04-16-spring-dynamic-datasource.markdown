---
title: 'Spring动态数据源切换实现读写分离'
layout: post
categories: 技术
tags:
    - Java
---

数据库的读写分离是最基本的扩展方式和优化手段。数据库以主从方式工作，当有写操作的时候写入主库，读操作则从从库读取。这样就涉及了多数据源的切换问题。在Spring中提供了一些声明式的方法，实现了动态数据源的切换。

首先，在Spring的配置文件中声明多个数据源：

```
<!-- galaxy db-->
<bean id="ds_galaxy_master" class="com.mchange.v2.c3p0.ComboPooledDataSource"
    destroy-method="close">
    <property name="driverClass">
        <value>com.mysql.jdbc.Driver</value>
    </property>
    <property name="jdbcUrl">
        <value>${jdbcUrl.galaxy.master}</value>
    </property>
    <property name="user">
        <value>${username.galaxy.master}</value>
    </property>
    <property name="password">
        <value>${password.galaxy.master}</value>
    </property>
    <property name="maxPoolSize">
        <value>100</value>
    </property>
    <property name="minPoolSize">
        <value>10</value>
    </property>
    <property name="idleConnectionTestPeriod">
        <value>300</value>
    </property>
    <property name="maxIdleTime">
        <value>600</value>
    </property>
</bean>

<bean id="ds_galaxy_slave" class="com.mchange.v2.c3p0.ComboPooledDataSource"
    destroy-method="close">
    <property name="driverClass">
        <value>com.mysql.jdbc.Driver</value>
    </property>
    <property name="jdbcUrl">
        <value>${jdbcUrl.galaxy.slave}</value>
    </property>
    <property name="user">
        <value>${username.galaxy.slave}</value>
    </property>
    <property name="password">
        <value>${password.galaxy.slave}</value>
    </property>
    <property name="maxPoolSize">
        <value>100</value>
    </property>
    <property name="minPoolSize">
        <value>10</value>
    </property>
    <property name="idleConnectionTestPeriod">
        <value>300</value>
    </property>
    <property name="maxIdleTime">
        <value>600</value>
    </property>
</bean>
```

接下来，我们声明一个动态数据源，这个动态数据源的targetDataSources属性，需要引用前面声明的两个数据源，并配置默认的数据源：

```
<bean id="dynamicDs_galaxy" class="com.sogou.earth.api.common.db.DynamicDataSource">
    <property name="targetDataSources">
        <map key-type="java.lang.String">
            <entry key="master" value-ref="ds_galaxy_master" />
            <entry key="slave" value-ref="ds_galaxy_slave" />
        </map>
    </property>
    <property name="defaultTargetDataSource" ref="ds_galaxy_master" />
</bean>
```

DynamicDataSource这个类要继承Spring的AbstractRoutingDataSource，AbstractRoutingDataSource这个类间接继承了javax.sql.DataSource。当获取数据库连接的时候，AbstractRoutingDataSource会根据数据源的key决定使用哪个数据源：

```
public Connection getConnection() throws SQLException {
    return determineTargetDataSource().getConnection();
}

protected DataSource determineTargetDataSource() {
    Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");
    Object lookupKey = determineCurrentLookupKey();
    DataSource dataSource = this.resolvedDataSources.get(lookupKey);
    if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
        dataSource = this.resolvedDefaultDataSource;
    }
    if (dataSource == null) {
        throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
    }
    return dataSource;
}
```

这里，我们需要在DynamicDataSource中实现determineCurrentLookupKey这个抽象方法，在这个方法中决定使用哪个key（这里是master或slave）。

```
public final class DynamicDataSource extends AbstractRoutingDataSource {

    private static final Logger logger = LoggerFactory.getLogger(DynamicDataSource.class);

    protected Object determineCurrentLookupKey() {
        Object object = DynamicDataSourceKeyHolder.getDataSourceKey();
        logger.debug("determineCurrentLookupKey:" + object);
        return object;
    }
}
```

DynamicDataSourceKeyHolder中使用了ThreadLocal，来为每个线程设置独立的数据源key，避免在并发情况下出现的key混乱的情况：

```
public class DynamicDataSourceKeyHolder {

    private static final Logger logger = LoggerFactory.getLogger(DynamicDataSourceKeyHolder.class);
    
    private static final ThreadLocal<String> dataSourceHolder = new ThreadLocal<String>();

    public static void setKey(String key) {
        dataSourceHolder.set(key);
    }

    public static String getDataSourceKey() {
        return (String) dataSourceHolder.get();
    }
}
```

具体DynamicDataSourceKeyHolder中key如何被设置，则是通过aop在方法访问时根据方法名前缀设置的，比如get、find等前缀则用slave数据源，insert、update、delete等前缀用master数据源，具体可见下面配置。

我们继续回到Spring配置中，继续配置sqlSessionFactory和mapperScannerConfigurer，用于配置mybatis的xml文件和包的位置，这里的数据源指向之前配置的动态数据源dynamicDs_galaxy：

```
<bean id="sqlSessionFactory_galaxy" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dynamicDs_galaxy" />
    <property name="mapperLocations"
        value="classpath*:com/sogou/earth/api/core/account/dao/*.xml,classpath*:com/sogou/earth/api/core/plan/dao/*.xml,
        classpath*:com/sogou/earth/api/core/group/dao/*.xml,classpath*:com/sogou/earth/api/core/idea/dao/*.xml,
        classpath*:com/sogou/earth/api/core/word/dao/*.xml,classpath*:com/sogou/earth/api/core/common/dao/*.xml" />
</bean>
<bean id="mapperScannerConfigurer_galaxy" class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory_galaxy"></property>
    <property name="basePackage"
        value="com.sogou.earth.api.core.account.dao,com.sogou.earth.api.core.plan.dao,com.sogou.earth.api.core.group.dao,
    com.sogou.earth.api.core.idea.dao,com.sogou.earth.api.core.word.dao,com.sogou.earth.api.core.common.dao" />
</bean>
```

接下来，配置事务管理器，datasource也执行之前配置的动态数据源dynamicDs_galaxy：

```
<bean id="transactionManager_galaxy"
    class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dynamicDs_galaxy" />
</bean>
```

然后，我们来配置一个切面。注意，这个切面是在service层方法上的。当service层方法被调用时，与切面关联的两个通知触发，从而设置数据源的key，以及当事务出现异常时进行处理：

```
<aop:config>
    <aop:pointcut id="dynamicDataSourcePoint" expression="execution(public * com.sogou.earth.api.core.account.service.impl.*.*(..))|| execution(public * com.sogou.earth.api.core.plan.service.impl.*.*(..))||
    execution(public * com.sogou.earth.api.core.group.service.impl.*.*(..))||execution(public * com.sogou.earth.api.core.idea.service.impl.*.*(..))||execution(public * com.sogou.earth.api.core.word.service.impl.*.*(..))||
    execution(public * com.sogou.earth.api.core.common.service.impl.*.*(..))"/> 
    <aop:advisor pointcut-ref="dynamicDataSourcePoint" advice-ref="dynamicDsInterceptor_galaxy"/>
    <aop:advisor pointcut-ref="dynamicDataSourcePoint" advice-ref="txAdvice_galaxy"/>
</aop:config>
```

dynamicDsInterceptor_galaxy是一个方法拦截器，用于根据方法名前缀分配数据源key：

```
<bean id="dynamicDsInterceptor_galaxy" class="com.sogou.earth.api.common.db.DynamicDataSourceInterceptor">
    <property name="attributeSource">
        <list>
            <value>query*,slave</value>
            <value>count*,slave</value>
            <value>find*,slave</value>
            <value>get*,slave</value>
            <value>list*,slave</value>
            <value>*,master</value>
        </list>
    </property>
</bean>
```

DynamicDataSourceInterceptor类实现了MethodInterceptor接口，在方法被调用时动态向DynamicDataSourceKeyHolder中设置数据源key：

```
public class DynamicDataSourceInterceptor implements MethodInterceptor {

    private static final Logger logger = LoggerFactory.getLogger(DynamicDataSourceInterceptor.class);
    /**
     * 方法和使用数据源key的对应关系
     */
    private List<String> attributeSource = new ArrayList<String>();

    public Object invoke(MethodInvocation invocation) throws Throwable {
        final String methodName = invocation.getMethod().getName();
        String key = null;
        for (String value : attributeSource) {
            String mappedName = value.split(",")[0];
            if (isMatch(methodName, mappedName)) {
                key = value.split(",")[1];
                break;
            }
        }
        logger.debug("methodName:" + methodName);
        if (null != key) {
            DynamicDataSourceKeyHolder.setKey(key);
        }

        return invocation.proceed();
    }

    private boolean isMatch(String methodName, String mappedName) {
        return PatternMatchUtils.simpleMatch(mappedName, methodName);
    }

    public List<String> getAttributeSource() {
        return attributeSource;
    }

    public void setAttributeSource(List<String> attributeSource) {
        this.attributeSource = attributeSource;
    }

}
```

最后，我们需要配置另一个通知，用于指定异常出现时的处理策略：

```
<tx:advice id="txAdvice_galaxy" transaction-manager="transactionManager_galaxy"> 
    <tx:attributes>
        <tx:method name="modify*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="update*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="cancel*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="add*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="del*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="set*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="save*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="commit*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="pause*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="insert*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="create*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="pass*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="*" read-only="true"/>
    </tx:attributes>
</tx:advice>
```

可以看到，所有的写操作在遇到异常时回滚事务。