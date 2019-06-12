---
title: 'Mac升级Yosemite系统后brew报错无法使用解决办法'
layout: post
categories: 技术
tags:
    - Mac
---

Mac升级Yosemite后，Terminal中使用brew时会报这样一个错误：

```
/usr/local/bin/brew: /usr/local/Library/brew.rb: /System/Library/Frameworks/Ruby.framework/Versions/1.8/usr/bin/ruby: bad interpreter: No such file or directory
/usr/local/bin/brew: line 23: /usr/local/Library/brew.rb: Undefined error: 0
```

一个一个查看了这几个文件是否存在或者有问题后，发现查看Ruby的时候发现版本不对:

```
ls -al /System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/bin/ruby
```

想来是Mac在升级系统的时候把ruby的版本也提升到了2.0，那我们就改一下brew的文件里面配置，这里有两个关于brew的文件:

```
/usr/local/bin/brew: /usr/local/Library/brew.rb
```

查看了一下上面两个 bash 脚本，发现是/usr/local/Library/brew.rb中第一行写死用的Ruby 1.8，那就改一下就好了:

```
vim /usr/local/Library/brew.rb
#!/System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/bin/ruby -W0
```

保存后退出即可。