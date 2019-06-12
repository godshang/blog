---
title: 'jQuery学习笔记'
layout: post
categories: 技术
tags:
    - JavaScript
---

## What is jQuery ? ##

jQuery是一套跨浏览器的JavaScript库，简化了HTML与JavaScript之间的操作。jQuery的语法设计使得许多操作变得容易，如操作文档对象（document）、选择DOM元素、创建动画效果、处理事件、以及开发Ajax程序。jQuery 也提供了给开发人员在其上创建插件的能力。这使开发人员可以对底层交互与动画、高级效果和高级主题化的组件进行抽象化。模块化的方式使 jQuery 函数库能够创建功能强大的动态网页以及网络应用程序。

## $ ##

$是jQuery中最常见的符号。它可以接受一个字符，也可以接受一个文档对象，亦或者一个函数，也可以调用一个函数。那么$到底是什么东西呢？

```
var jQuery = (function() {
	//创建jQuery对象,给所有的jQuery方法提供统一的入口,避免繁琐难记
	var jQuery = function( selector, context ) {
		//jQuery的构造对象,调用了jQuery.fn.init方法
		//最后返回jQuery.fn.init的对象
		return new jQuery.fn.init( selector, context, rootjQuery );
	},
	.....
	//定义jQuery的原型,jQuery.fn指向jQuery.prototype对象
	jQuery.fn = jQuery.prototype = {
	//重新指定构造函数属性,因为默认指向jQuery.fn.init
	constructor: jQuery,
	init: function( selector, context, rootjQuery ) {.....},
	......
	}
	......
	//返回jQuery变量,同时定义将全局变量window.jQuery和window.$指向jQuery
	return (window.jQuery = window.$ = jQuery);

})();
```

当首次执行完毕后，全局变量$和jQuery，都是指向了var jQuery=function（selector，context）{}这个函数，这里，就可以下个结论，$就是jQuery的别名。通过$，可以方便的引用jQuery。

## Dom对象和jQuery包装对象 ##

无论是在写程序还是看API文档,  我们要时刻注意区分Dom对象和jQuery包装集。

### Dom对象 ###

在传统的javascript开发中,我们都是首先获取Dom对象,然后再对Dom对象进行操作：

```
var div = document.getElementById("testDiv");
var divs = document.getElementsByTagName("div");
```

我们经常使用 document.getElementById 方法根据id获取单个Dom对象, 或者使用 document.getElementsByTagName 方法根据HTML标签名称获取Dom对象集合。这里获取到的都是Dom对象, Dom对象也有不同的类型比如input, div, span等。Dom对象只有有限的属性和方法。

### jQuery包装集 ###

jQuery包装集可以说是Dom对象的扩充.在jQuery的世界中将所有的对象, 无论是一个还是一组, 都封装成一个jQuery包装集,比如获取包含一个元素的jQuery包装集:

```
var jQueryObject = $("#testDiv");
```

jQuery包装集都是作为一个对象一起调用的. jQuery包装集拥有丰富的属性和方法, 这些都是jQuery特有的。

### Dom对象和jQuery对象的转换 ###

#### Dom转jQuery对象 ####

将Dom对象转换为jQuery对象十分简单，使用jQuery选择器就可以完成：

```
$("#testDiv");
```

上面语句构造的包装集只含有一个id是testDiv的元素。

或者我们已经获取了一个Dom元素,比如:

```
var div = document.getElementById("testDiv");
```

上面的代码中div是一个Dom元素, 我们可以将Dom元素转换成jQuery包装集:

```
var domToJQueryObject = $(div);
```

#### jQuery对象转Dom对象 #####

jQuery包装集是一个集合, 所以我们可以通过索引器访问其中的某一个元素:

```
var domObject = $("#testDiv")[0];
```

注意, 通过索引器返回的不再是jQuery包装集, 而是一个Dom对象!

jQuery包装集的某些遍历方法,比如each()中, 可以传递遍历函数, 在遍历函数中的this也是Dom元素,比如:

```
$("#testDiv").each(function() { alert(this) })
```

如果我们要使用jQuery的方法操作Dom对象,怎么办? 用上面介绍过的转换方法即可:

```
$("#testDiv").each(function() { $(this).html("修改内容") })
```

## jQuery选择器 ##

在jQuery中，通过其提供的强大的选择器可以帮助我们获取页面上的对象，并将对象以jQuery包装集的形式返回。

选择器就是"一个表示特殊语意的字符串". 只要把选择器字符串传入上面的方法中就能够选择不同的Dom对象并且以jQuery包装集的形式返回。

jQuery的选择器类似于CSS中的选择器，同时提供了强大的选择和过滤等功能。

一些简单的选择器：

- id选择器：#id，根据元素id选择
- 元素选择器：根据元素名称选择
- 类选择器：.class，根据元素class类选择
- 。。。

此外，还能选择兄弟元素、父元素、祖先元素，按照索引选择、按奇偶选择，按照属性选择，等等丰富的选择功能。

从jQuery的选择器就很能看出一个开发人员的功底了。

## 操作元素的属性和样式 ##

jQuery提供了API来操作元素的属性和样式。

取得元素属性：

```
$("#testInput").attr('checked')
```

设置元素属性：

```
$("#testInput").attr('checked', 'checked')
```

取得CSS样式：

```
$("p").css("color");
```

设置CSS样式：

```
$("p").css({ color: "#ff0011", background: "blue" }); 
```

不像Java，jQuery的set和get方法的名字是一样的，只是以参数来区别。

## 事件 ##

jQuery使用bind函数来讲事件同元素绑定起来：

```
$("#testDiv4").bind("click", showMsg);
```

这样就为id是testDiv4的元素, 添加列click事件的事件处理函数showMsg.

虽然可以使用事件处理函数完成对象事件的几乎所有操作, 但是jQuery提供了对常用事件的封装. 比如单击事件对应的两个方法click()和click(fn)分别用来触发单击事件和设置单击事件。

设置单击事件:

```
$("#testDiv").click(function(event) { alert("test div clicked ! "); });
```

等效于:

```
$("#testDiv").bind("click", function(event) { alert("test div clicked ! "); });
```

此外还有blur、change、dbclick、focus、keydown、keyup等等各种常用的事件。

## 工具函数 ##

### 字符工具函数

$.trim

说明:去掉字符串起始和结尾的空格。

### 测试工具函数

jQuery.isArray( obj )

jQuery.isFunction( obj )

### 对象合并函数

$.extend

说明：用一个或多个其他对象来扩展一个对象，返回被扩展的对象。

### 迭代

$.each(object, callback)

说明：通用例遍方法，可用于例遍对象和数组。不同于例遍 jQuery 对象的 $().each() 方法，此方法可用于例遍任何对象。回调函数拥有两个参数：第一个为对象的成员或数组的索引，第二个为对应变量或内容。如果需要退出 each 循环可使回调函数返回 false，其它返回值将被忽略。


