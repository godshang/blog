---
title: '使用CocoaPods开发静态库时的几点问题'
layout: post
categories: 技术
tags:
    - iOS
---

`CocoaPods`是iOS和mac OS开发平台的依赖管理工具，十分完善和强大。除了可以用`CocoaPods`管理第三方开源库，也可以搭建自己的私服、管理自己的库。

使用pod开发静态库可以参考`http://www.cnblogs.com/brycezhang/p/4117180.html`这篇文章。开发过程中有一些问题需要注意。

### 私有库 ###

如果自己开发的库引入了其他的私有库，在进行`podspece`文件的验证时，需要加上`--source`参数。

```
lib lint --sources='http://git.sogou-inc.com/bizios/BizSpecs.git,https://github.com/CocoaPods/Specs.git' --use-libraries

pod spec lint --sources='http://git.sogou-inc.com/bizios/BizSpecs.git,https://github.com/CocoaPods/Specs.git' --use-libraries

pod repo push BizSpecs BIZNotifySdk.podspec --sources='http://git.sogou-inc.com/bizios/BizSpecs.git,https://github.com/CocoaPods/Specs.git' --use-libraries --allow-warnings
```
### 静态库 ###

如果自己开发的库引入其他的静态库，即自己的库A依赖库B、而库B是一个`.a`文件，如果依赖的静态库`.a`文件打包时不支持i386架构（[i386是什么？](http://bengyuejiejie.github.io/blog/2015/03/09/first-blog/)），那么是通不过`podspece`文件的验证的，即使这不影响x86_64和真机上的运行。

github上有类似问题的[讨论](https://github.com/CocoaPods/CocoaPods/issues/5854)，但目前没找到好的方法。为了能通过验证，只能暂时修改了`CocoaPods`的`validator.rb`文件。

```
	$ gem which cocoapods
	/Library/Ruby/Gems/2.0.0/gems/cocoapods-1.2.0/lib/cocoapods.rb

	$ cd cocoapods
	$ vi validator.rb
```

修改xcodebuild中ios部分，指定`-arch`参数：

```
	def xcodebuild
      require 'fourflusher'
      command = ['clean', 'build', '-workspace', File.join(validation_dir, 'App.xcworkspace'), '-scheme', 'App', '-configuration', 'Release']
      case consumer.platform_name
      when :osx, :macos
        command += %w(CODE_SIGN_IDENTITY=)
      when :ios
        #command += %w(CODE_SIGN_IDENTITY=- -sdk iphonesimulator)
        #command += Fourflusher::SimControl.new.destination(:oldest, 'iOS', deployment_target)
        command += %w(CODE_SIGN_IDENTITY=- -sdk iphonesimulator -arch x86_64)
      when :watchos
        command += %w(CODE_SIGN_IDENTITY=- -sdk watchsimulator)
        command += Fourflusher::SimControl.new.destination(:oldest, 'watchOS', deployment_target)
      when :tvos
        command += %w(CODE_SIGN_IDENTITY=- -sdk appletvsimulator)
        command += Fourflusher::SimControl.new.destination(:oldest, 'tvOS', deployment_target)
      end
```

