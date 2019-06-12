---
title: 'SeaJS学习笔记'
layout: post
categories: 技术
tags:
    - JavaScript
---

## What is SeaJS ? ##

官方的描述：

> SeaJS 是一个遵循 CommonJS 规范的模块加载框架，可用来轻松愉悦地加载任意 JavaScript 模块，一个文件就是一个模块，依赖关系也只需遵守简单约定，无需冗长的配置。使用 SeaJS, 能让你更关注于编码的乐趣。

## 缘起 ##

### 命名冲突 ###

我们从一个简单的习惯出发。项目开发时常常会将一些通用的、底层的功能抽象出来，独立成一个个函数，比如

```
function isString(arr) {
  // 实现代码
}

function isArray(str) {
  // 实现代码
}
```

把这些函数统一放在util.js里。需要用到时，引入该文件就行。但假设多个库有同名方法时，就有出现命名冲突的现象。

参照Java的方式，通过引入命名空间来解决。

```
var com = {};
com.sogou = {};
com.sogou.Utils = {};

com.sogou.Utils.isString = function (arr) {
  // 实现代码
};

com.sogou.Utils.isArray = function (str) {
  // 实现代码
};
```

使用时

```
if (com.sogou.Utils.isString(response)) {
  // 实现代码
}
if (com.sogou.Utils.isArray(response)) {
  // 实现代码
}
```

通过命名空间，的确能极大缓解冲突。但过长的命名空间给编码带来很大麻烦。

### 文件依赖 ###

假设现有多个js文件：

```
//module1.js
var module1 = {
    run: function() {
        return $.merge(['module1'], $.merge(module2.run(), module3.run()));
    }
}
 
//module2.js
var module1 = {
    run: function() {
        return ['module2'];
    }
}
 
//module3.js
var module3 = {
    run: function() {
        return $.merge(['module3'], module4.run());
    }
}
 
//module4.js
var module4 = {
    run: function() {
        return ['module4'];
    }
}
```

此时index.html需要引用module1.js及其所有下层依赖（注意顺序）

```
<!DOCTYPE HTML>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>TinyApp</title>
    <script src="./jquery-min.js"></script>
    <script src="./module4.js"></script>
    <script src="./module2.js"></script>
    <script src="./module3.js"></script>
    <script src="./module1.js"></script>
</head>
<body>
    <p class="content"></p>
    <script>
        $('.content').html(module1.run());
    </script>
</body>
</html>
```

随着项目的进行，js文件会越来越多，依赖关系也会越来越复杂，使得js代码和html里的script列表往往变得难以维护。

## 使用SeaJS ##

首先是index.html：

```
<!DOCTYPE HTML>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>TinyApp</title>
</head>
<body>
    <p class="content"></p>
    <script src="./sea.js"></script>
    <script>
        seajs.use('./init', function(init) {
            init.initPage();
        });
    </script>
</body>
</html>
```

可以看到html页面不再需要引入所有依赖的js文件，而只是引入一个sea.js，sea.js会处理所有依赖，加载相应的js文件。

index.html加载了init模块，并使用此模块的initPage方法初始化页面数据，这里先不讨论代码细节。

下面看一下模块化后JavaScript的写法：

```
//jquery.js
define(function(require, exports, module) = {
 
    //原jquery.js代码...
 
    module.exports = $.noConflict(true);
});
 
//init.js
define(function(require, exports, module) = {
    var $ = require('jquery');
    var m1 = require('module1');
     
    exports.initPage = function() {
        $('.content').html(m1.run());    
    }
});
 
//module1.js
define(function(require, exports, module) = {
    var $ = require('jquery');
    var m2 = require('module2');
    var m3 = require('module3');
     
    exports.run = function() {
        return $.merge(['module1'], $.merge(m2.run(), m3.run()));    
    }
});
 
//module2.js
define(function(require, exports, module) = {
    exports.run = function() {
        return ['module2'];
    }
});
 
//module3.js
define(function(require, exports, module) = {
    var $ = require('jquery');
    var m4 = require('module4');
     
    exports.run = function() {
        return $.merge(['module3'], m4.run());    
    }
});
 
//module4.js
define(function(require, exports, module) = {
    exports.run = function() {
        return ['module4'];
    }
});
```

乍看之下代码似乎变多变复杂了，这是因为这个例子太简单，如果是大型项目，SeaJS代码的优势就会显现出来。

不过从这里我们还是能窥探到一些SeaJS的特性：

- 一是html页面不用再维护冗长的script标签列表，只要引入一个sea.js即可。
- 二是js代码以模块进行组织，各个模块通过require引入自己依赖的模块，代码清晰明了。

## SeaJS基本开发原则 ##

使用SeaJS开发JavaScript的基本原则就是：一切皆为模块。引入SeaJS后，编写JavaScript代码就变成了编写一个又一个模块，SeaJS中模块的概念有点类似于面向对象中的类——模块可以拥有数据和方法，数据和方法可以定义为公共或私有，公共数据和方法可以供别的模块调用。

另外，每个模块应该都定义在一个单独js文件中，即一个对应一个模块。

## 模块的定义及编写 ##

### 模块定义函数define ###

SeaJS中使用“define”函数定义一个模块。

```
Module._define = function(id, deps, factory) {

}
seajs.define = Module._define
```

define可以接收三个参数，分别是模块ID，依赖模块数组及工厂函数。define对于不同参数个数的解析规则如下：

- 如果只有一个参数，则赋值给factory。
- 如果有两个参数，第二个赋值给factory；第一个如果是array则赋值给deps，否则赋值给id。
- 如果有三个参数，则分别赋值给id，deps和factory。

当使用一个参数的define定义模块。那么id和deps会怎么处理呢？

id是一个模块的标识字符串，define只有一个参数时，id会被默认赋值为此js文件的绝对路径。如example.com下的a.js文件中使用define定义模块，则这个模块的ID会赋值为“http://example.com/a.js”，没有特别的必要建议不要传入id。deps一般不需要传入，需要用到的模块用require加载即可。

### 工厂函数factory ###

工厂函数是模块的主体和重点。在只传递一个参数给define时（推荐写法），这个参数就是工厂函数，此时工厂函数的三个参数分别是：

- require——模块加载函数，用于记载依赖模块。
- exports——接口点，将数据或方法定义在其上则将其暴露给外部调用。
- module——模块的元数据。

这三个参数可以根据需要选择是否需要显示指定。

下面说一下module。module是一个对象，存储了模块的元信息，具体如下：

- module.id——模块的ID。
- module.dependencies——一个数组，存储了此模块依赖的所有模块的ID列表。
- module.exports——与exports指向同一个对象。

### 三种编写模块的模式 ###

第一种定义模块的模式是基于exports的模式：

```
define(function(require, exports, module) {
    var a = require('a'); //引入a模块
    var b = require('b'); //引入b模块
 
    var data1 = 1; //私有数据
 
    var func1 = function() { //私有方法
        return a.run(data1);
    }
 
    exports.data2 = 2; //公共数据
 
    exports.func2 = function() { //公共方法
        return 'hello';
    }
});
```

除了将公共数据和方法附加在exports上，也可以直接返回一个对象表示模块。

```
define(function(require) {
    var a = require('a'); //引入a模块
    var b = require('b'); //引入b模块
 
    var data1 = 1; //私有数据
 
    var func1 = function() { //私有方法
        return a.run(data1);
    }
 
    return {
        data2: 2,
        func2: function() {
            return 'hello';
        }
    };
});
```

如果模块定义没有其它代码，只返回一个对象，还可以有如下简化写法：

```
define({
    data: 1,
    func: function() {
        return 'hello';
    }
});
```

## 模块的载入和引用 ##

### seajs.use ###

seajs.use主要用于载入入口模块。入口模块相当于C程序的main函数，同时也是整个模块依赖树的根。seajs.use用法如下：

```
//单一模式
seajs.use('./a');
 
//回调模式
seajs.use('./a', function(a) {
  a.run();
});
 
//多模块模式
seajs.use(['./a', './b'], function(a, b) {
  a.run();
  b.run();
});
```

一般seajs.use只用在页面载入入口模块，SeaJS会顺着入口模块解析所有依赖模块并将它们加载。

### require ###

require是SeaJS主要的模块加载方法，当在一个模块中需要用到其它模块时一般用require加载：

```
var m = require('/path/to/module/file');
```

除了在require中使用路径来加载外，还可以使用别名来接在。别名的配置在seajs.config中，后面会介绍到。

这里简要介绍一下SeaJS的自动加载机制。上文说过，使用SeaJS后html只要包含sea.js即可，那么其它js文件是如何加载进来的呢？SeaJS会首先下载入口模块，然后顺着入口模块使用正则表达式匹配代码中所有的require，再根据require中的文件路径标识下载相应的js文件，对下载来的js文件再迭代进行类似操作。整个过程类似图的遍历操作（因为可能存在交叉循环依赖所以整个依赖数据结构是一个图而不是树）。

明白了上面这一点，下面的规则就很好理解了：

传给require的路径标识必须是字符串字面量，不能是表达式，如下面使用require的方法是错误的：

```
require('module' + '1');
 
require('Module'.toLowerCase());
```

这都会造成SeaJS无法进行正确的正则匹配以下载相应的js文件。

### require.async ###

上文说过SeaJS会在html页面打开时通过静态分析一次性记载所有需要的js文件，如果想要某个js文件在用到时才下载，可以使用require.async：

```
require.async('/path/to/module/file', function(m) {
    //code of callback...
});
```

这样只有在用到这个模块时，对应的js文件才会被下载，也就实现了JavaScript代码的按需加载。

## SeaJS的全局配置 ##

SeaJS提供了一个seajs.config方法可以设置全局配置，接收一个表示全局配置的配置对象。

```
seajs.config({
    base: 'path/to/jslib/',
    alias: {
      'app': 'path/to/app/'
    },
    charset: 'utf-8',
    timeout: 20000,
    debug: false
});
```

其中base表示基址寻址时的基址路径。

alias可以对较长的常用路径设置缩写。此处的别名可以在require时使用，方便模块的加载。

charset表示下载js时script标签的charset属性。

timeout表示下载文件的最大时长，以毫秒为单位。

debug表示是否工作在调试模式下。

## SeaJS如何与现有JS库配合使用 ##

要将现有JS库如jQuery与SeaJS一起使用，只需根据SeaJS的的模块定义规则对现有库进行一个封装。例如，下面是对jQuery的封装方法：

```
define('jquery', [], function(require) {
 
    // jQuery原有代码
     
    return $.noConflict(true);
});
```