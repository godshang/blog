---
title: 'Spring解析自定义XML'
layout: post
categories: 技术
tags:
    - Java
---

Spring框架从2.0版本开始，支持基于Schema的XML扩展机制，允许开发者自定义XML标签。下面以一个小例子，说明如何实现自定义XML配置，并使Spring将我们自定义的XML解析为bean。

我们首先看一下，希望在Spring的XML配置中，添加的新标签：

````
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:d="http://www.longingfuture.com/schema/demo"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
    http://www.longingfuture.com/schema/demo http://www.longingfuture.com/schema/demo.xsd">

    <d:parent id="testParent" name="testParent001">  
        <d:child name="child01" age="22" />
        <d:child name="child03" age="24" />
    </d:parent>
</beans>
```

命名空间d下的parent和child两个元素，是我们增加的新标签。我们需要一个Schema文件，用来说明自定义标签的各个约束，Spring会用这个Schema文件对XML中的标签进行验证：

```
<?xml version="1.0" encoding="UTF-8"?>
<xsd:schema xmlns="http://www.w3.org/2001/XMLSchema"
    xmlns:xsd="http://www.w3.org/2001/XMLSchema" 
    xmlns:beans="http://www.springframework.org/schema/beans"
    xmlns:tns="http://www.longingfuture.com/schema/demo"
    targetNamespace="http://www.longingfuture.com/schema/demo"
    elementFormDefault="qualified" attributeFormDefault="unqualified">
    
    <xsd:import namespace="http://www.springframework.org/schema/beans" />

    <xsd:element name="parent">
        <xsd:complexType>
            <xsd:complexContent>
                <xsd:extension base="beans:identifiedType">
                    <xsd:sequence>
                        <xsd:element name="child" type="tns:child" maxOccurs="unbounded" />
                    </xsd:sequence>
                    <xsd:attribute name="name" type="xsd:string" />
                </xsd:extension>
            </xsd:complexContent>
        </xsd:complexType>
    </xsd:element>

    <xsd:complexType name="child">
        <xsd:attribute name="name" type="xsd:string" />
        <xsd:attribute name="age" type="xsd:decimal" />
    </xsd:complexType>
</xsd:schema>
```

在这个Schema文件中，我们定义了parent元素和child元素。parent元素有一个名为name的属性，以及一个child类型的列表。注意，我们为这两个新元素声明了一个名为http://www.longingfuture.com/schema/demo的命名空间。在Spring的XML配置文件中，需要保证这个命名空间的正确。

接下来，我们添加一个NamespaceHandler，来支持对新定义的命名空间元素的解析：

```
public class MyNamespaceHandler extends NamespaceHandlerSupport {

    @Override
    public void init() {
        registerBeanDefinitionParser("parent", new MyBeanDefinitorParse());
    }
}
```

MyNamespaceHandler继承了NamespaceHandlerSupport，并重载了init方法。在init方法中，用registerBeanDefinitionParser方法向Spring注册了一个解析器，当遇到XML配置中的parent元素中，就调用这个解析器进行解析。

MyBeanDefinitorParse继承了AbstractBeanDefinitionParser，并重载parseInternal这个方法。在parseInternal方法中，实现具体的解析逻辑：

```
public class MyBeanDefinitorParse extends AbstractBeanDefinitionParser {

    @Override
    protected AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
        BeanDefinitionBuilder parentBeanDefinitionBuilder = BeanDefinitionBuilder.rootBeanDefinition(Parent.class);
        parentBeanDefinitionBuilder.addPropertyValue("name", element.getAttribute("name"));

        List<Element> childElements = DomUtils.getChildElementsByTagName(element, "child");
        List<BeanDefinition> childBeanDefinitions = new ArrayList<BeanDefinition>(childElements.size());
        for (Iterator iterator = childElements.iterator(); iterator.hasNext();) {
            Element childElement = (Element) iterator.next();
            BeanDefinitionBuilder childBeanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(Child.class);
            
            childBeanDefinitionBuilder.addPropertyValue(Conventions.attributeNameToPropertyName("name"), childElement.getAttribute("name"));
            childBeanDefinitionBuilder.addPropertyValue(Conventions.attributeNameToPropertyName("age"), childElement.getAttribute("age"));

            childBeanDefinitions.add(childBeanDefinitionBuilder.getBeanDefinition());
        }
        
        ManagedList list = new ManagedList();
        for (BeanDefinition beanDefinition : childBeanDefinitions) {
            list.add(beanDefinition);
        }

        parentBeanDefinitionBuilder.addPropertyValue("children", list);

        return parentBeanDefinitionBuilder.getBeanDefinition();
    }
}
```

完成上述过程后，为了能让Spring知道我们新加入的handler和parser，还需要向Spring进行注册，这需要在classpath下的META-INF文件夹下，新增spring.handlers和spring.schemas两个文件。

在spring.handlers文件中，指明了当遇到自定义的命名空间时需要调用哪个handler：

```
http\://www.longingfuture.com/schema/demo=com.longingfuture.demo.spring.customtag.MyNamespaceHandler
```

在spring.schemas文件中，指明了Schema文件的具体包位置：

```
http\://www.longingfuture.com/schema/demo.xsd=com/longingfuture/demo/spring/customtag/demo.xsd
```

完成上述后，我们就可以正常的从Spring中获取我们的bean了：

```
public class Main {

    public static void main(String[] args) throws Exception {
        ApplicationContext ctx = new ClassPathXmlApplicationContext(
                new String[] { "appContext-customtag.xml" });
        Parent parent = ctx.getBean(Parent.class);
        System.out.println(parent.getName());
        
        for (Child child : parent.getChildren()) {
            System.out.println(child.getName() + ":" + child.getAge());
        }
    }
}
```