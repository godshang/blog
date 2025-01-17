---
title: 'jQuery + Spring MVC的前后端传参问题'
layout: post
categories: 技术
tags:
    - jQuery
    - JavaScript
    - Java
    - Spring
---

这两天，连续有人问到关于Spring MVC接收前端传参时一直接收不到的问题。虽然以前也遇到同样的问题，并且搜索后解决了，但对于这个问题出现的原因一直没太搞清楚。这两天网络上查了查，看了前后端涉及请求发送接收部分的代码，大致整理下相关知识。

# jQuery中Ajax请求的发送与参数传递 #

既然请求是由前端发起的，那么我们便从前端开始谈起。

## JavaScript对象与JSON ##

首先，必须明确的一点是，JavaScript中的对象与JSON是两个不一样的东西，虽然它们看起来很相似，也有一定关系。JavaScript对象，当然是JavaScript语言中的对象（这很明显是一句废话）；而JSON是一种数据交换格式。从JSON的名字（JavaScript Object Notation）能够看出来，JSON是采用了JavaScript中对象表示语法来描述数据格式的一种表述方式。JSON与JavaScript对象相似的是，它们都是用花括号表示对象，方括号表示数组，键值对表示数据，每个键值对之间用逗号分隔。而不同的地方在于，JSON中要求每个key的名称都要用双引号引起来，而JavaScript对象中的key则没有这个强制要求，除非这个key中含有空格；另外，JavaScript对象当然只能用在JavaScript中，而JSON这种格式完全独立于语言的字符串，可以被任何其他编程语言解析。

在Java中，解析JSON的工具包十分丰富，比如Gson、Jackson等。在JavaScript中，在对象和JSON之间的转换十分简单，调用 JSON.stringify 将一个JavaScript对象转换为JSON，调用 JSON.parse 函数将JSON转换为JavaScript对象。这两个函数主流浏览器都支持（IE8+），可以直接调用。

JavaScript对象和JSON这两个概念其实非常明确，但常常有人会混淆它们，原因就在于二者长得太像了。比如，我们看这一段JavaScript代码，调用jQuery的ajax方法，向后端发起一次Ajax请求。

```javascript
	$.ajax({
		url: "/test/xxx.do",
		type: "POST",
		data: {
			foo: 123,
			bar: 'abc'
		},
		dataType: "json",
		success: function() { },
		error: function() { }
	});
```

这段代码非常好懂，即使不熟悉jQuery的同学也能一样看明白：向"/test/xxx.do"发起一次POST请求，请求的参数是data部分，数据格式是json，请求成功返回后回调success，请求异常时回调error。

但是，如果不熟悉jQuery的同学非常容易误认为这次发起的HTTP请求就是JSON的，特别是看到data是个类似JSON的JavaScript对象，而且dataType还清楚明白的写明了"json"。实际上，翻一翻jQuery的文档就能看到，这个dataType指的是预期的服务端返回的数据类型，而不是request过去的数据类型；而data也仅仅只是JavaScript中的对象，jQuery的文档也特别说明了，这个JavaScript对象会被转换为"&foo=123&bar=abc"这样的query string。

当然，上面讨论的这种情况，是在jQuery默认设置项作用下产生的效果，接下来可以看到contentType起到的巨大作用。

## ContentType ##

HTTP中的ContentType用于决定HTTP报文中数据的类型。我们知道，一个完整的HTTP请求必然涉及一来一去两个步骤，即request和response。request过去，接收方会按照HTTP报文中ContentType指定的类型来解析报文中携带的数据；response回来，发起方依然会按照HTTP报文中的ContentType来解析返回的数据。很显然，如果你设置的ContentType和你实际的数据类型不一致，解析会出错。所以当我们用jQuery发起请求，以及用Spring MVC接收请求时，除了保证请求地址、请求数据的正确性外，也应该注意到ContentType和HTTP报文中数据类型的正确性。

那么如何用jQuery正确的发起一次请求呢？看一下jQuery的源码，ajax的默认配置是这样的：

```javascript
	ajaxSettings: {
		...
		type: "GET",
		...
		processData: true,
		async: true,
		contentType: "application/x-www-form-urlencoded; charset=UTF-8",
		/*
		timeout: 0,
		data: null,
		dataType: null,
		...
		traditional: false,
		*/
		...
```

可以看出来，ajax方法默认会使用"application/x-www-form-urlencoded"作为请求的contentType，除非你显式指定contentType。这种ContentType其实你很熟悉，当你用HTML中的form表单向服务端发送请求时，ContentType就是这个。

用上一节中的代码发起一次请求，抓包后可以看到HTTP报文的内容：

```
	POST http://local.admin.brandstart.sogou-inc.com/test/xxx.do HTTP/1.1
	Host: local.admin.brandstart.sogou-inc.com
	User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64; rv:42.0) Gecko/20100101 Firefox/42.0
	Accept: application/json, text/javascript, */*; q=0.01
	Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
	Accept-Encoding: gzip, deflate
	Content-Type: application/x-www-form-urlencoded; charset=UTF-8
	X-Requested-With: XMLHttpRequest
	Referer: http://local.admin.brandstart.sogou-inc.com/index.jsp
	Content-Length: 15
	Cookie: session_id_brandstart_admin=brandstart-admin_962f620b-59d8-4447-a1b4-73d3e71570b0; JSESSIONID=13DC5C149E2FDF104A10AE3D8F2533D9; session_id_brandstart_admin=brandstart-admin_962f620b-59d8-4447-a1b4-73d3e71570b0
	Connection: keep-alive
	Pragma: no-cache
	Cache-Control: no-cache

	foo=123&bar=abc
```

可以看到Header中Content-Type确实使用了默认的配置。

你可能注意到了，HTTP报文中Body部分的数据，是以&分隔的键值对形式，这也验证了上一节中说到的jQuery关于data的文档描述。那么为什么会使用这种形式发送数据呢？我们再来看一看jQuery中ajax方法的代码，这里省略了其他部分，只留下了关于data的处理：

```javascript
	ajax: function( url, options ) {
		...
		// Convert data if not already a string
		if ( s.data && s.processData && typeof s.data !== "string" ) {
			s.data = jQuery.param( s.data, s.traditional );
		}
		...
	}
```

s这个对象是ajax方法一开始会将默认配置参数和从函数options中传入的自定义配置参数合并后得到的一个最终参数。从这个if可以看到，如果data存在，并且processData为true（默认配置），并且data的类型不是string类型，那么就会调用jQuery中的一个param方法将data做一次处理。在param方法中，就完成了将JavaScript对象转换为行如"foo=123&bar=abc"这种形式的字符串。

如果我想发送一次JSON类型的HTTP请求呢？根据上面讨论，只需要简单的将ContentType设置为"application/json"，并且调用 JSON.stringify 函数将JavaScript对象序列化为JSON即可：

```javascript
	$.ajax({
		url: "/test/xxx.do",
		contentType: "application/json; charset=UTF-8",
		type: "POST",
		data: JSON.stringify({
			foo: 123,
			bar: 'abc'
		}),
		dataType: "json",
		success: function() { },
		error: function() { }
	});
```

此时再去抓包，可以看到HTTP报文中Content-Type和Body部分已经变了。

```
	POST http://local.admin.brandstart.sogou-inc.com/test/xxx.do HTTP/1.1
	Host: local.admin.brandstart.sogou-inc.com
	User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64; rv:42.0) Gecko/20100101 Firefox/42.0
	Accept: application/json, text/javascript, */*; q=0.01
	Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
	Accept-Encoding: gzip, deflate
	Content-Type: application/json; charset=UTF-8
	X-Requested-With: XMLHttpRequest
	Referer: http://local.admin.brandstart.sogou-inc.com/index.jsp
	Content-Length: 23
	Cookie: session_id_brandstart_admin=brandstart-admin_962f620b-59d8-4447-a1b4-73d3e71570b0; JSESSIONID=13DC5C149E2FDF104A10AE3D8F2533D9; session_id_brandstart_admin=brandstart-admin_962f620b-59d8-4447-a1b4-73d3e71570b0
	Connection: keep-alive
	Pragma: no-cache
	Cache-Control: no-cache

	{"foo":123,"bar":"abc"}
```

## jQuery.param函数 ##

上一节谈到了jQuery的param函数会完成JavaScript对象到"foo=1&bar=abc"这样字符串的序列化。这个函数是长这样的：

```javascript
	jQuery.param(object,traditional)
```

其中，object是要进行序列化的数组或对象；traditional是一个可选参数，默认是false，用来指定是否使用传统的方式浅层进行序列化。但是，传统方式是个什么鬼。。。

我们先用一个复杂一点的对象参数的例子看下param方法的输出是什么样子的：

```javascript
	var myObject = {
	 	a: {
	    	one: 1, 
	    	two: 2, 
	    	three: 3
	  	}, 
	  	b: [1,2,3]
	};
	$.param(myObject);
```

这个param的输出是这样的：

```
	a%5Bone%5D=1&a%5Btwo%5D=2&a%5Bthree%5D=3&b%5B%5D=1&b%5B%5D=2&b%5B%5D=3
```

这是一个经过URL编码后的字符串，对其解码后是长这样的：

```
	a[one]=1&a[two]=2&a[three]=3&b[]=1&b[]=2&b[]=3
```

那么如果同样的对象myObject，指定了traditional为true后，输出又是什么样呢？

```
	a=%5Bobject+Object%5D&b=1&b=2&b=3
```

URL解码后是长这样的：

```
	a=[object Object]&b=1&b=2&b=3
```

通过这个对比，应该能看出了traditional对于对象序列化的影响：当traditional为false，也就是默认情况下，对于嵌套对象（myObject内的a依然是个对象）会进行深度的序列化，而为true的时候，并不会进行。那么如果a继续含有嵌套对象的时候呢？你可以试试看。

traditional对于序列化的影响还体现在数组上。上面的对比中可以看到，当traditional为false时，myObject中的数组被序列化为形如"b[]=1&b[]=2&b[]=3"的字符串，在b后面多了个方括号；而当traditional为true时，序列化后的数组b并没有添加方括号。加上方括号这种方式，按文档的说法是为了适应PHP、Ruby on Rails这种现代化脚本语言和框架的需要。对于

```
	http://link/foo.php?id[]=1&id[]=2&id[]=3
```

这样的数组参数id，PHP可以直接通过

```	php
	$_GET['id']
```

获取到全部的值。但是在java中，通过id可无法取到值。那么对于形如"b[]=1&b[]=2&b[]=3"这样的数组，在Java中如何拿到数组值呢？很简单，通过"b[]"就好了：

```java
	String[] array = request.getParameterValues("b[]");
```

那么当traditional为true时，也就是序列化为"b=1&b=2&b=3"这样的字符串时，在Java中如何获取呢？想当然的，通过"b"即可：

```java
	String[] array = request.getParameterValues("b");
```

不过，此时对象a被序列化为了"a=[object Object]"，这个东西可没办法再反序列成Java对象了。

通过上面的讨论，可以看到，通过param函数序列化的能力十分有限，简单的参数结构当然没有问题，如果在请求中携带复杂的数据时，这种方式的表达方式便不能满足要求了，或者说不管前端发请求，还是后端接收请求，要付出额外的关注，十分捉急。那么，有没有简单的方法搞定复杂对象的参数传递呢？方法就是使用前一节说的的JSON格式的HTTP报文，即将HTTP的ContentType设置为"application/json"，并且用JSON.stringify将JavaScript对象转换为JSON。这样，不管多么复杂的参数对象，完全不用操心。

## Spring MVC中的参数解析 ##

前面说完了前端如何发起一次Ajax请求，并且携带上参数。接下来，说说在Spring MVC中如何从一个HTTP请求中解析出携带的那些参数，并且还原为Java中的数据类型。

Spring MVC处理请求的过程这里不再赘述，可以参考最后的几篇文章。这里只谈关于HTTP报文中参数解析的部分。

我们知道，HTTP请求和响应本质上都是一串字符串，当请求报文到达一个Servlet时，它被封装为一个ServletInputStream的输入流，Servlet处理后，响应报文则通过ServletOutputStream的输出流输出。我们从输入流中只能读到字符串，同样往输出流中输出的也只是字符串。因此，这就必然涉及一个字符串和Java对象之间的转化。在Spring MVC中，HTTP的请求和响应报文被抽象为HttpInputMessage和HttpOutputMessage两个接口：

```java
	public interface HttpInputMessage extends HttpMessage {
	    InputStream getBody() throws IOException;
	}

	public interface HttpOutputMessage extends HttpMessage {
	    OutputStream getBody() throws IOException;
	}
```

而HTTP报文的转换是通过HttpMessageConverter这个接口实现：

```java
	public interface HttpMessageConverter<T> {

	    boolean canRead(Class<?> clazz, MediaType mediaType);

	    boolean canWrite(Class<?> clazz, MediaType mediaType);

	    List<MediaType> getSupportedMediaTypes();

	    T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
	            throws IOException, HttpMessageNotReadableException;

	    void write(T t, MediaType contentType, HttpOutputMessage outputMessage)
	            throws IOException, HttpMessageNotWritableException;

	}
```

HttpMessageConverter接口通过canRead和canWrite方法判断能否进行请求和响应报文的转换，具体的转换又是通过read和write方法实现。其中，MediatType是对HTTP报文中ContentType的抽象。

HttpMessageConverter接口有若干实现，分别应对不同数据类型的HTTP报文的转换。其中FormHttpMessageConverter及其子类XmlAwareFormHttpMessageConverter处理的就是ContentType为"application/x-www-form-urlencoded"和"multipart/form-data"的报文；MappingJacksonHttpMessageConverter处理的就是ContentType为"application/json"的报文。具体的报文转换可以在FormHttpMessageConverter和MappingJacksonHttpMessageConverter两个类的read和write方法中体现出来，可以自己进到里面看一看。需要注意的是，MappingJacksonHttpMessageConverter中处理JSON报文用到了Jackson，这也是我们在POM中添加Jackson依赖的原因。

将上述的报文转换过程集中描述在HandlerMethodArgumentResolver接口和HandlerMethodReturnValueHandler接口，分别用于处理请求报文中参数的转换，以及相应报文中结果的输出。

```java
	public interface HandlerMethodArgumentResolver {

		boolean supportsParameter(MethodParameter parameter);

		Object resolveArgument(MethodParameter parameter, 
							   ModelAndViewContainer mavContainer,
							   NativeWebRequest webRequest, 
							   WebDataBinderFactory binderFactory) throws Exception;

	}

	public interface HandlerMethodReturnValueHandler {

		boolean supportsReturnType(MethodParameter returnType);

		void handleReturnValue(Object returnValue,
							   MethodParameter returnType,
							   ModelAndViewContainer mavContainer,
							   NativeWebRequest webRequest) throws Exception;

	}
```

这两个接口又有若干实现，分别用于处理Spring MVC中用不同注解描述的参数解析和结果输出，比如常用的@RequestBody、@ResponseBody、@RequestParam等等。对于@RequestBody和@ResponseBody注解，是由RequestResponseBodyMethodProcessor这个类负责，前面几节中谈到JSON格式的HTTP报文就是在这里完成和对象之间的转换的。我们看下supportsParameter方法和supportReturnType方法就能知道，会被RequestResponseBodyMethodProcessor处理的请求和响应报文，Java中必须以@RequestBody和@ResponseBody添加注解：

```java
	public boolean supportsParameter(MethodParameter parameter) {
		return parameter.hasParameterAnnotation(RequestBody.class);
	}

	public boolean supportsReturnType(MethodParameter returnType) {
		return returnType.getMethodAnnotation(ResponseBody.class) != null;
	}
```

以报文到对象的转换为例：

```java
	protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter methodParam,
		Class<T> paramType) throws IOException, HttpMediaTypeNotSupportedException {
	
		MediaType contentType = inputMessage.getHeaders().getContentType();
		if (contentType == null) {
			contentType = MediaType.APPLICATION_OCTET_STREAM;
		}
	
		for (HttpMessageConverter<?> messageConverter : this.messageConverters) {
			if (messageConverter.canRead(paramType, contentType)) {
				if (logger.isDebugEnabled()) {
					logger.debug("Reading [" + paramType.getName() + "] as \"" + contentType + "\" using [" +
							messageConverter + "]");
				}
				return ((HttpMessageConverter<T>) messageConverter).read(paramType, inputMessage);
			}
		}
	
		throw new HttpMediaTypeNotSupportedException(contentType, allSupportedMediaTypes);
	}
```

可以看到，这里会先取出HTTP报文中的ContentType，然后遍历一组HttpMessageConverter实例，直到某个HttpMessageConverter能够解析HTTP报文中的参数。

对于@RequestParam注解描述的对象，是由RequestParamMethodArgumentResolver这个类负责。这个类只实现了HandlerMethodArgumentResolver接口，也就是说它只能完成HTTP请求报文中参数到对象的转换。我们看下supportsParameter方法：

```java

	public boolean supportsParameter(MethodParameter parameter) {
		Class<?> paramType = parameter.getParameterType();
		if (parameter.hasParameterAnnotation(RequestParam.class)) {
			if (Map.class.isAssignableFrom(paramType)) {
				String paramName = parameter.getParameterAnnotation(RequestParam.class).value();
				return StringUtils.hasText(paramName);
			}
			else {
				return true;
			}
		}
		else {
			if (parameter.hasParameterAnnotation(RequestPart.class)) {
				return false;
			}
			else if (MultipartFile.class.equals(paramType) || "javax.servlet.http.Part".equals(paramType.getName())) {
				return true;
			}
			else if (this.useDefaultResolution) {
				return BeanUtils.isSimpleProperty(paramType);
			}
			else {
				return false;
			}
		}
	}

```
RequestParamMethodArgumentResolver不仅对于@RequestParam和@RequestPart注解提供支持；也能看到useDefaultResolution为true时，如果参数是一个简单的数据类型（如String、Integer、Long等），也能实现HTTP请求报文中参数的解析，这也是为什么我们即使没有在Controller方法中的参数添加@RequestParam注解，简单类型的参数依然能够被解析出来的原因。

除了RequestResponseBodyMethodProcessor和RequestParamMethodArgumentResolver之外，HandlerMethodArgumentResolver和HandlerMethodReturnValueHandler接口还有其他一些实现，比如PathVariableMethodArgumentResolver用于负责@PathVariable注解，RequestHeaderMethodArgumentResolver用于负责@RequestHeader注解，等。

# 小结 #

到了这里，如何正确的发起一次Ajax请求，并携带参数，以及如何被Spring MVC解析还原为Java中数据的过程就说完了。相关概念很简单，只要请求的发起方和接收方，注意到HTTP报文的ContentType以及数据，就并无问题。

# 参考资料 #

[1] http://www.w3school.com.cn/jquery/ajax_ajax.asp

[2] http://www.css88.com/jqapi-1.9/jQuery.param/

[3] http://stackoverflow.com/questions/1833330/how-to-get-php-get-array

[4] http://www.cnblogs.com/fangjian0423/p/springMVC-request-param-analysis.html

[5] http://my.oschina.net/lichhao/blog?catalog=285356

