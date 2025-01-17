---
title: 'JavaScriptCore框架入门'
layout: post
categories: 技术
tags:
    - iOS
---

从Objective-C调用JavaScript，是通过`UIWebView`的`- (NSString *)stringByEvaluatingJavaScriptFromString:(NSString *)script;`方法来实现的，该方法向`UIWebView`传递一段需要执行的JavaScript代码最后获取执行结果。而从JavaScript向Objective-C的调用都是基于URL的拦截进行。

iOS 7后，Apple提供了`JavaScriptCore`框架，让Objective-C和JavaScript代码的交互变得更加简单和方便。JavaScriptCore框架是基于webkit中以C/C++实现的JavaScriptCore的一个封装，在旧版本iOS开发中，很多开发者也会自行将webkit的库引入项目编译使用，不过iOS7开始苹果把它当成了标准库。

## JavaScriptCore

JavaScriptCore.h头文件中引入了5个文件，每个文件里都定义了跟文件名对应的类：

- JSContext
- JSValue
- JSManagedValue
- JSVirtualMachine
- JSExport

`JSVirtualMachine`为JavaScript的运行提供了底层资源，有独立的堆空间和垃圾回收机制。

`JSContext`其提供运行环境，通过`- (JSValue *)evaluateScript:(NSString *)script;`方法就可以执行一段JavaScript脚本，并且如果其中有方法、变量等信息都会被存储在其中以便在需要的时候使用。JSContext的创建都是基于`JSVirtualMachine：- (id)initWithVirtualMachine:(JSVirtualMachine *)virtualMachine;`，如果是使用`- (id)init;`进行初始化，那么在其内部会自动创建一个新的JSVirtualMachine对象然后调用前边的初始化方法。

`JSValue`则可以说是JavaScript和Object-C之间互换的桥梁，每个`JSValue`都和`JSContext`相关联并且强引用context。它提供了多种方法可以方便地把JavaScript数据类型转换成Objective-C，或者是转换过去。

|  Objective-C |     JavaScript     |              JSValue Convert              |                                            JSValue Constructor                                           |
|:------------:|:------------------:|:-----------------------------------------:|:--------------------------------------------------------------------------------------------------------:|
|      nil     |      undefined     |                                           |                                        valueWithUndefinedInContext                                       |
|    NSNull    |        null        |                                           |                                          valueWithNullInContext:                                         |
|   NSString   |       string       |                  toString                 |                                                                                                          |
|   NSNumber   |   number, boolean  | toNumber <br> toBool <br> toDouble <br> toInt32 <br> toUInt32 | valueWithBool:inContext: <br> valueWithDouble:inContext: <br> valueWithInt32:inContext: <br> valueWithUInt32:inContext: |
| NSDictionary | Object object      | toDictionary                              | valueWithNewObjectInContext                                                                              |
| NSArray      | Array object       | toArray                                   | valueWithNewArrayInContext:                                                                              |
| NSDate       | Date object        | toDate                                    |                                                                                                          |
| NSBlock      | Function object    |                                           |                                                                                                          |
| id           | Wrapper object     | toObject <br> toObjectOfClass:                 | valueWithObject:inContext:                                                                               |
| Class        | Constructor object |                                           |                                                                                                          |

`JSManagedValue`是JavaScript和Objective-C对象的内存管理辅助对象。由于JavaScript内存管理是垃圾回收，并且JavaScript中的对象都是强引用，而Objective-C是引用计数。如果双方相互引用，势必会造成循环引用，而导致内存泄露。我们可以用`JSManagedValue`保存JSValue来避免。

`JSExport`是一个协议，如果JavaScript对象想直接调用Objective-C对象里面的方法和属性，那么这个Objective-C对象只要实现这个`JSExport`协议就可以了。

## 基本用法

```objectivec
    JSContext *context = [[JSContext alloc] init];
    JSValue *value = [context evaluateScript:@"21+7"];
    int iVal = [value toInt32];
    NSLog(@"JSValue: %@, int: %d", value, iVal);
```

从OC中调用JS代码很简单，生成一个`JSContext`对象，调用`evaluateScript`加载JS代码，该方法会将这段JS代码最后返回的值以`JSValue`包装后返回到OC中。

还可以将一个JS变量保存在`JSContext`对象中，然后通过下标获取。对于`Array`和`Object`类型，`JSValue`也可以通过下标直接获取值和赋值。

```objectivec
    JSContext *context = [[JSContext alloc] init];
    [context evaluateScript:@"var arr = [21, 7 , 'iderzheng.com'];"];
    
    JSValue *jsArr = context[@"arr"];
    NSLog(@"JSArray: %@, Length: %@", jsArr, jsArr[@"length"]);
//    Output: JSArray: 21,7,iderzheng.com, Length: 3
    
    jsArr[1] = @"blog";
    jsArr[7] = @7;
    NSLog(@"JSArray: %@, Length: %@", jsArr, jsArr[@"length"]);
//    Output: JSArray: 21,blog,iderzheng.com,,,,,7, Length: 8
    
    NSArray *nsArr = [jsArr toArray];
    NSLog(@"NSArray: %@", nsArr);
//    Output: NSArray: (
//          21,
//          blog,
//          "iderzheng.com",
//          "<null>",
//          "<null>",
//          "<null>",
//          "<null>",
//          7
//          )
```

通过输出结果很容易看出代码成功把数据从Objective-C赋到了JavaScript数组上，而且`JSValue`是遵循JavaScript的数组特性：无下标越位，自动延展数组大小。并且通过`JSValue`还可以获取JavaScript对象上的属性，比如例子中通过`length`就获取到了JavaScript数组的长度。在转成`NSArray`的时候，所有的信息也都正确转换了过去。

## 方法的转换

### block

Objective-C的block可以传入`JSContext`中当做JavaScript的方法使用。

```objectivec
    JSContext *context = [[JSContext alloc] init];
    context[@"log"] = ^() {
        NSLog(@"++++++++ Begin Log ++++++++");
        
        NSArray *args = [JSContext currentArguments];
        for (JSValue *jsVal in args) {
            NSLog(@"%@", jsVal);
        }
        
        JSValue *this = [JSContext currentThis];
        NSLog(@"this: %@", this);
        
        NSLog(@"++++++++ End Log ++++++++");
    };
    
    [context evaluateScript:@"log('alan', [7, 21], { key1: 'str', key2: 100 });"];
    //Output:
    //  +++++++Begin Log+++++++
    //  alan
    //  7,21
    //  [object Object]
    //  this: [object GlobalObject]
    //  -------End Log-------

```

通过block可以在JavaScript中调用Objective-C的方法，并且遵循JavaScript方法的特点，如方法参数不固定。也因为这样，`JSContext` 提供了类方法来获取参数列表(`+ (NSArray *)currentArguments`)和当前调用该方法的对象(`+ (JSValue *)currentThis`)。上边的例子中对于`this`输出的内容是`GlobalObject`，这也是`JSContext`对象方法`- (JSValue *)globalObject;`所返回的内容。因为我们知道在JavaScript里，所有全局变量和方法其实都是一个全局变量的属性，在浏览器中是window，在JavaScriptCore是什么就不得而知了。

block可以传入 `JSContext` 中被JavaScript代码调用，但是 `JSValue` 没有 `toBlock` 方法把JavaScript方法变成block在Objective-C中使用，毕竟block的参数个数和类型已经返回类型都是固定的。虽然不能把方法提取出来，但是 `JSValue` 提供了 `- (JSValue *)callWithArguments:(NSArray *)arguments;` 方法可以反过来将参数传进去来调用方法。

```objectivec
    JSContext *context = [[JSContext alloc] init];
    [context evaluateScript:@"function add(a, b) { return a + b; }"];
    JSValue *add = context[@"add"];
    NSLog(@"Func:  %@", add);
     
    JSValue *sum = [add callWithArguments:@[@(7), @(21)]];
    NSLog(@"Sum:  %d",[sum toInt32]);

    JSValue *ret = [sum invokeMethod:@"toString" withArguments:nil];
    NSLog(@"Ret:  %@", ret);

    //OutPut:
    //  Func:  function add(a, b) { return a + b; }
    //  Sum:  28
    //  Ret:  28
```

`JSValue`还提供`- (JSValue *)invokeMethod:(NSString *)method withArguments:(NSArray *)arguments;`让我们可以直接简单地调用JavaScript对象上的方法。只是如果定义的方法是全局函数，那么很显然应该在`JSContext` 的 `globalObject`对象上调用该方法；如果是某JavaScript对象上的方法，就应该用相应的`JSValue`对象调用。

### JSExport

JavaScript可以脱离prototype继承完全用JSON来定义对象，但是Objective-C编程里可不能脱离类和继承了写代码。所以JavaScriptCore就提供了`JSExport`作为两种语言的互通协议。`JSExport`中没有约定任何的方法，连可选的(@optional)都没有，但是所有继承了该协议(@protocol)的协议（注意不是Objective-C的类(@interface)）中定义的方法，都可以在`JSContext`中被使用。

```objectivec
// 首先定义一个JSExporter协议
@protocol JSExportTarget <JSExport>

- (NSInteger)add:(NSInteger)a b:(NSInteger)b;

@property (nonatomic, assign) NSInteger sum;

@end

// 建一个对象去实现这个协议
@interface JSExporter : NSObject <JSExportTarget>
@end

@implementation JSExporter
@synthesize sum = _sum;

- (NSInteger)add:(NSInteger)a b:(NSInteger)b {
    return a + b;
}

- (void)setSum:(NSInteger)sum {
     NSLog(@"--%@", @(sum));
    _sum = sum;
}
@end

// 使用时
JSContext *context = [[JSContext alloc] init];

JSExporter *exporter = [[JSExporter alloc] init];
context[@"exp"] = exporter;

[context evaluateScript:@"exp.sum = exp.addB(2,3);"];

// Output:
// --5
```

要注意的是OC的函数命名和JS函数命名规则问题。协议中定义的是add: b:，但是JS里面方法名字是addB(a,b)。可以通过JSExportAs这个宏转换成JS的函数名字。

```objectivec
@protocol JSExportTarget <JSExport>

JSExportAs(add, - (NSInteger)add:(NSInteger)a b:(NSInteger)b);

@property (nonatomic, assign) NSInteger sum;

@end

// 使用时
[context evaluateScript:@"exp.sum = exp.add(2,3);"];
```

## 异常处理

`JSContext`中执行的JavaScript代码如果出现异常，会被`JSContext`捕获并存储在`exception`属性上，不会向外抛出。时刻检查`exception`属性是否为`nil`显示不合适，更合理的方式是给JSContext对象设置`exceptionHanlder`，它接受的是`^(JSContext *context, JSValue *exceptionValue)`形式的block，默认的实现就是将传入的`exceptionValue`赋值给传入的`context`的`exception`属性。

```objectivec
    ^(JSContext *context, JSValue *exceptionValue) {
        context.exception = exceptionValue;
    };
```

可以提供一个新的block，加入自己的处理逻辑：

```objectivec
    JSContext *context = [[JSContext alloc] init];
    context.exceptionHandler = ^(JSContext *context, JSValue *exceptionValue) {
        NSLog(@"%@", exception);
        context.exception = exceptionValue;
    };
```

## 内存管理

在内存管理方面，Objective-C使用的ARC，JavaScript使用垃圾回收机制，并且所有的引用都是强引用，JS中的循环引用，垃圾回收会负责处理。JavaScriptCore里的API，正常情况下，不需要开发者去关心Objective-C和JavaScript的内存管理问题。不过还有几点需要留意。

### block注意事项

block在JavaScriptCore中起到的强大作用，它在JavaScript和Objective-C之间的转换 建立起更多的桥梁，让互通更方便。但是要注意的是无论是把block传给`JSContext`对象让其变成JavaScript方法，还是把它赋给`exceptionHandler`属性，在block内都不要直接使用其外部定义的`JSContext`对象或者`JSValue`，应该将其当做参数传入到block中，或者通过`JSContext`的类方法`+ (JSContext *)currentContext;`来获得。否则会造成循环引用使得内存无法被正确释放。

比如上边自定义异常处理方法，就是赋给传入`JSContext`对象con，而不是其外创建的context对象，虽然它们其实是同一个对象。这是因为block会对内部使用的在外部定义创建的对象做强引用，而`JSContext`也会对被赋予的block做强引用，这样它们之间就形成了循环引用(Circular Reference)使得内存无法正常释放。

对于`JSValue`也不能直接从外部引用到block中，因为每个`JSValue`上都有`JSContext`的引用 (`@property(readonly, retain) JSContext *context;`)，`JSContext`再引用block同样也会形成引用循环。

### JSManagedValue的使用

Objective-C和JavaScript两种不同的内存管理机制出现在同一个程序中时难免会产生冲突。

比如，在一个方法中创建了一个临时的Objective-C对象，然后将其加入到JSContext放在JavaScript中的变量中被使用。因为JavaScript中的变量有引用所以不会被释放回收，但是Objective-C上的对象可能在方法调用结束后，引用计数变0而被回收内存，因此JavaScript层面也会造成错误访问。

同样的，如果用JSContext创建了对象或者数组，返回JSValue到Objective-C，即使把JSValue变量retain下，但可能因为JavaScript中因为变量没有了引用而被释放内存，那么对应的JSValue也没有用了。

怎么在两种内存回收机制中处理好对象内存就成了问题。JavaScriptCore提供了JSManagedValue类型帮助开发人员更好地管理对象内存。

```objectivec
@interface JSManagedValue : NSObject
 
// Convenience method for creating JSManagedValues from JSValues.
+ (JSManagedValue *)managedValueWithValue:(JSValue *)value;
 
// Create a JSManagedValue.
- (id)initWithValue:(JSValue *)value;
 
// Get the JSValue to which this JSManagedValue refers. If the JavaScript value has been collected,
// this method returns nil.
- (JSValue *)value;
 
@end
```

`JSVirtualMachine`为整个JavaScriptCore的执行提供资源，所以当将一个`JSValue`转成`JSManagedValue`后，就可以添加到`JSVirtualMachine`中，这样在运行期间就可以保证在Objective-C和JavaScript两侧都可以正确访问对象而不会造成不必要的麻烦。

```objectivec
@interface JSVirtualMachine : NSObject
 
// Create a new JSVirtualMachine.
- (id)init;
 
// addManagedReference:withOwner and removeManagedReference:withOwner allow 
// clients of JSVirtualMachine to make the JavaScript runtime aware of 
// arbitrary external Objective-C object graphs. The runtime can then use 
// this information to retain any JavaScript values that are referenced 
// from somewhere in said object graph.
// 
// For correct behavior clients must make their external object graphs 
// reachable from within the JavaScript runtime. If an Objective-C object is 
// reachable from within the JavaScript runtime, all managed references 
// transitively reachable from it as recorded with 
// addManagedReference:withOwner: will be scanned by the garbage collector.
// 
- (void)addManagedReference:(id)object withOwner:(id)owner;
- (void)removeManagedReference:(id)object withOwner:(id)owner;
 
@end
```

### JSVirtualMachine

一个`JSVirtualMachine`可以运行多个`context`，由于都是在同一个堆内存和同一个垃圾回收下，所以相互之间传值是没问题的。但是如果在不同的 `JSVirtualMachine`传值，垃圾回收就不知道他们之间的关系了，可能会引起异常。

## 线程

`JavaScriptCore`线程是安全的，每个`context`运行的时候通过lock关联的`JSVirtualMachine`。如果要进行并发操作，可以创建多个`JSVirtualMachine`实例进行操作。






