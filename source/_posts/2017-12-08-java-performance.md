---
title: 'Java 性能调优工具'
layout: post
categories: 技术
tags:
    - Java
---

## Linux 命令行工具

### top 命令

top 命令的输出可以分为两个部分，前半部分是系统统计信息，后半部分是进程信息。

在统计信息中，第1行是任务队列信息，它的结果等同于 uptime 命令。从左到有一次表示，系统当前时间、系统运行时间、当前登录用户数、系统平均负载（即任务队列的平均长度，这3个值表示1分钟、5分钟、15分钟到现在的平均值）l

第2行是进程统计信息，分别有正在运行的进程数、睡眠进程数、停止的进程数、僵尸进程数。

第3行是 CPU 统计信息，us 表示用户空间 CPU 占用率，sy 表示内核空间 CPU 占用率，ni 表示用户进程空间改变过优先级的进程 CPU 的占用率，id 表示空闲 CPU 占用率，wa 表示等待输入输出的 CPU 时间百分比，hi 表示硬件中断请求，si 表示软件中断请求，

在 Mem 行中从左到右依次表示物理内存总量、空闲物理内存、已使用的物理内存、内核缓冲使用量。

Swap 行依次表示交换区总量、空闲交换区大小、已使用交换区大小、缓冲交换区大小。

top 命令的第2部分是进程信息区，显示了系统内各个进程的资源使用情况。

- PID：进程 id
- PPID：父进程 id
- RUSER：Real user name
- UID：进程所有者的用户 id
- USER：进程所有者的用户 id
- GROUP：进程所有者的组名
- TFY：启动进程的终端名，不是从终端启动的进程则显示为?
- PR：优先级
- NI：nice 值，负值表示高优先级，正值表示低优先级
- P：最后使用的 CPU ，仅在多 CPU 环境下有意义
- %CPU：上次更新到现在的 CPU 时间占用百分比
- TIME：进程使用的 CPU 时间总计，单位秒
- TIME+：进程使用的CPU时间总计，单位1/100秒
- %MEM：进程使用的物理内存百分比
- VIRT：进程使用的虚拟内存总量，单位KB。VIRT=SWAP+RES
- SWAP：进程使用的虚拟内存中被换出的大小，单位KB
- RES：进程使用的、未被换出的物理内存大小，单位DB。RES=CODE+DATA
- CODE：可执行代码占用的物理内存大小，单位KB
- DATA：可执行代码意外的部分（数据段+栈）占用的物理内存大小，单位KB
- SHR：共享内存大小，单位KB
- nFLT：页面错误次数
- nDRL；最后一次写入到现在，被修改过的页面数
- S：进程状态，D表示不可终端的睡眠状态，R表示运行，S表示睡眠，T表示跟踪/停止，Z表示僵尸进程
- COMMAND：命令名/命令行
- WCHAN：若改进程在睡眠，则显示睡眠中的系统函数名
- Flags：任务标志，参考 sched.h

在 top 命令下，按下 f 键，可以进行列的选择，使用 o 键可以更改列的显示顺序。此外，top 命令还有一些实用的交互指令：

- h：显示帮助信息
- k：终止一个进程
- q：退出程序
- c：切换显示命令名称和完整命令行
- M：根据驻留内存大小进行排序
- P：根据CPU使用百分比大小进行排序
- T：根据时间/累计时间进行排序
- 数字1：显示所有CPU负载情况

### sar 命令

sar 命令可以周期性地对内存和 CPU 使用情况进行采样。基本语法如下：

```
sar [ options ] [ <interval> ] [ <count> ]
```

interval 和 count 分别表示采样周期和采样爽。 options 选项可以指定 sar 命令对那些性能数据进行采样：

- -A：所有报告的综合
- -u：CPU利用率
- -d：硬盘使用报告
- -b：I/O的情况
- -q：查看队列长度
- -r：内存使用统计信息
- -n：网络信息统计
- -o：采样结果输出到文件

### vmstat 命令

vmstat 命令可以统计CPU、内存使用情况、swap使用情况等信息。和sar工具类似，vmstat也可以指定采样周期和采样次数。

### iostat 命令

iostat 可以提供详尽的I/O信息。

## JDK 命令行工具

### jps 命令

jsp类似于Linux下的ps，但它只列出Java进程。直接运行jps不加任何参数，可以列出Java程序的进程ID以及Main函数等名称。

```
[root@djt_21_177 mobiledata]## jps
10434 Resin
1107 JswLauncher
24004 Bootstrap
12312 jenkins.war
12651 WatchdogManager
10558 Jps
```

jps本身也是一个Java程序，因此输出时也会输出jps。

参数 -q 指定jps只输出进程ID，而不输出类的短名称。

参数 -m 用于输出传递给Java进程的参数。

参数 -l 用于输出主函数的完整路径。

参数 -v 可以显示传递给JVM的参数。

### jstat 命令

jstat用于观察Java程序运行时的信息，通过它可以查看堆信息的详细情况。基本语法是：

```
jstat -<option> [-t] [-h<lines>] <vmid> [<interval>] [<count>]
```

选项option可以由以下值构成：

- -class：显示ClassLoader的相关信息
- -compiler：显示JIT编译相关的信息
- -gc：显示与GC相关的堆信息
- -gccapacity：显示各个代的容量及使用情况
- -gccause：显示垃圾收集相关信息（同-gcutil），同时显示最后一次或当前正在发生的垃圾收集的诱发原因
- -gcnew：显示新生代信息
- -gcnewcapacity：显示新生代大小与使用情况
- -gcold：显示老年代和永久代的信息
- -gcoldcapacity：显示老年代的大小
- -gcpermcapacity：显示永久代的大小
- -gcutil：显示垃圾收集信息
- -printcompilation：输出JIT编译的方法信息

-t参数可以在输出信息前加一个Timestamp列，显示程序的运行时间。

-h参数可以在周期性数据输出时，输出多少行数据后，跟着输出一个表头信息。

inerval参数用于指定输出统计数据的周期，单位为毫秒。

count用于指定一共输出多少次数据。

下例显示了与GC相关的堆信息输出：

```
[root@djt_21_177 ~]## jstat -gc 12704
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
1024.0 1024.0  0.0   448.0  29696.0  25198.2   175104.0   49192.3   41260.0 40440.1 4908.0 4735.4    391    2.856  174    20.151   23.007
```

各项参数的含义如下：

- S0C：s0（from）的大小（KB）
- S1C：s1（from）的大小（KB）
- S0U：s0（from）已使用的空间（KB）
- S1U：s1（from）已使用的控件（KB）
- EC：eden区的大小（KB）
- EU：eden区的使用空间（KB）
- OC：老年代大小（KB）
- OU：老年代已经使用的空间（KB）
- PC：永久代大小（KB）
- PU：永久代已使用的空间（KB）
- YGC：新生代GC次数
- YGCT：新生代GC耗时
- FGC：Full GC次数
- FGCT：Full GC耗时
- GCT：GC总耗时

下例显示了各个代的信息，与 -gc 相比，它不仅输出了各个代的当前大小，也包含了各个代的最大值和最小值：

```
[root@djt_21_177 ~]## jstat -gccapacity 12704
 NGCMN    NGCMX     NGC     S0C   S1C       EC      OGCMN      OGCMX       OGC         OC       MCMN     MCMX      MC     CCSMN    CCSMX     CCSC    YGC    FGC 
 41984.0  87040.0  31744.0 1024.0 1024.0  29696.0    84992.0   175104.0   175104.0   175104.0      0.0 1085440.0  41260.0      0.0 1048576.0   4908.0    392   174
```

各项参数的含义如下：

- NGCMN：新生代最小值（KB）
- NGCMX：新生代最大值（KB）
- NGC：当前新生代大小（KB）
- OGCMN：老年代最小值（KB）
- OGCMX：老年代最大值（KB）
- PGCMN：永久代最小值（KB）
- PGCMX：永久代最大值（KB）

下例显示了最近一次GC的原因以及当前GC的原因：

```
[root@djt_21_177 ~]## jstat -gccause 12704
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT    LGCC                 GCC                 
 46.88   0.00  42.05  28.09  98.01  96.48    392    2.863   174   20.151   23.014 Allocation Failure   No GC
```

各项参数的含义如下：

- LGCC：上次GC的原因
- GCC：当前GC的原因

-gcnew参数可以用于查看新生代的一些信息：

```
[root@djt_21_177 ~]## jstat -gcnew 12704
 S0C    S1C    S0U    S1U   TT MTT  DSS      EC       EU     YGC     YGCT  
1024.0 1024.0  480.1    0.0 15  15 1024.0  29696.0  13684.9    392    2.863
```

各项参数的含义如下：

- TT：新生代对象晋升到老年代对象的年龄
- MTT：新生代对象晋升到老年代对象的年龄最大值
- DSS：所需的survivor区大小

-gcnewcapacity参数可以详细输出新生代各个区的大小信息：

```
[root@djt_21_177 ~]## jstat -gcnewcapacity 12704
  NGCMN      NGCMX       NGC      S0CMX     S0C     S1CMX     S1C       ECMX        EC      YGC   FGC 
   41984.0    87040.0    31744.0  28672.0   1024.0  28672.0   1024.0    86016.0    29696.0   392   174
```

各项参数的含义如下：

- S0CMX：s0区的最大值（KB）
- S1CMX：s1区的最大值（KB）
- ECMX：eden区的最大值

-gcold参数可以用于展示老年代GC的情况：

```
[root@djt_21_177 ~]## jstat -gcold 12704
   MC       MU      CCSC     CCSU       OC          OU       YGC    FGC    FGCT     GCT   
 41260.0  40440.7   4908.0   4735.4    175104.0     49192.3    392   174   20.151   23.014
```

-gcoldcapacity用于展示老年代的容量信息：

```
[root@djt_21_177 ~]## jstat -gcoldcapacity 12704
   OGCMN       OGCMX        OGC         OC       YGC   FGC    FGCT     GCT   
    84992.0    175104.0    175104.0    175104.0   392   174   20.151   23.014
```

-gcpermcapacity用于展示永久代的使用情况：

（JDK8中似乎没这个参数了）

-gcutil参数用于展示GC回收相关信息：

```
[root@djt_21_177 ~]## jstat -gcutil 12704
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT   
 46.88   0.00  79.90  28.09  98.01  96.48    392    2.863   174   20.151   23.014
```

各项参数的含义如下：

- S0：s0区使用的百分比
- S1：s1区使用的百分比
- E：eden区使用的百分比
- O：old区使用的百分比
- P：永久区使用的百分比

### jinfo 命令

jinfo命令用来查看正在运行的Java程序的扩展参数，甚至支持在运行时修改部分参数。基本语法：

```
jinfo <option> <pid>
```

其中options参数为：

- -flag<name>：打印指定JVM的参数值
- -flag [+|-]<name>：设置指定JVM参数的布尔值
- -flag <name>=<value>：设置指定JVM参数的值

一个查看参数的例子：
```
[root@djt_21_177 ~]## jinfo -flag MaxTenuringThreshold 12704
-XX:MaxTenuringThreshold=15
```

一个修改参数的例子：

```
[root@djt_21_177 ~]## jinfo -flag PrintGCDetails 12704
-XX:-PrintGCDetails
[root@djt_21_177 ~]## jinfo -flag +PrintGCDetails 12704
[root@djt_21_177 ~]## jinfo -flag PrintGCDetails 12704 
-XX:+PrintGCDetails
[root@djt_21_177 ~]## jinfo -flag -PrintGCDetails 12704 
[root@djt_21_177 ~]## jinfo -flag PrintGCDetails 12704 
-XX:-PrintGCDetails
```

### jmap 命令

jmap命令可以生成Java程序的堆快照和对象的统计信息。

下例使用jmap生成Java程序的对象统计信息，并保存到文件中：

```
[root@djt_21_177 ~]## jmap -histo 12704 > ~/s.txt
```

输出文件有如下结构：

```

 num     #instances         #bytes  class name
----------------------------------------------
   1:        182656       24166520  [C
   2:        180480        5775360  com.caucho.env.actor.ValueActorQueue$ValueItem
   3:         23154        4808576  [B
   4:           156        4323264  [Lcom.caucho.util.LruCache$CacheItem;
   5:        149426        2390816  java.lang.Object
   6:         94984        2279616  java.lang.String
   7:         66091        2114912  com.caucho.loader.JarMap$JarList
   8:         10904        1466512  [Ljava.lang.Object;
   9:         32768        1048576  com.caucho.env.thread2.ThreadTaskItem2
  10:            14         853216  [Lcom.caucho.util.RingItem;
  ...
2961:             1             16  sun.util.locale.provider.SPILocaleProviderAdapter
2962:             1             16  sun.util.locale.provider.TimeZoneNameUtility$TimeZoneNameGetter
2963:             1             16  sun.util.resources.LocaleData
2964:             1             16  sun.util.resources.LocaleData$LocaleDataResourceBundleControl
Total        986439       60069520
```

这个输出显示了内存中的实例数量和合计。

另一个更为重要的功能是得到Java程序的当前堆快照：

```
[root@djt_21_177 ~]## jmap -dump:format=b,file=heap.hprof 12704  
Dumping heap to /root/heap.hprof ...
Heap dump file created
```

之后可以通过多种工具分析快照文件。

### jhat 命令

使用jhat工具可以分析Java程序的堆快照内容。已前文中jmap输出的堆快照文件heap.hprof为例：

```
[root@djt_21_177 ~]## jhat heap.hprof 
Reading from heap.hprof...
Dump file created Wed Dec 20 12:58:12 CST 2017
Snapshot read, resolving...
Resolving 1018170 objects...
Chasing references, expect 203 dots...........................................................................................................................................................................................................
Eliminating duplicate references...........................................................................................................................................................................................................
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```

jhat在分析完成之后，使用HTTP服务器展示其分析结果，通过浏览器即可访问。

jhat支持使用OQL语句对堆快照进行查询。例如：

```
select file.path.value.toString() from java.io.File file
```

结果如下：

```
/opt/app/bizmobile-web/WEB-INF/lib/aspectjweaver-1.7.2.jar
/opt/app/bizmobile-web/WEB-INF/lib/slf4j-log4j12-1.7.13.jar
/opt/app/bizmobile-web
/opt/app/bizmobile-web/WEB-INF/lib/aopalliance-1.0.jar
...

```

这里显示了Resin中所有打开的File对象路径。

### jstack 命令

jstack可用于导出Java程序的线程堆栈。语法为：

```
jstack [-l] <pid>
```

-l选项用于打印锁的附加信息，方便进行死锁的定位。

### jstatd 命令

之前讨论的工具，只涉及本机的Java程序监控。而一些工具，如jps、jstat等，也支持对远程计算机的监控。为了启用远程监控，需要配合使用jstatd工具。

jstat是一个RMI服务端程序，它的作用相当于代理服务器，建立本地计算机与远程监控工具的通信。jstatd服务器将本机的Java程序信息传递到远程计算机。

直接启动jstatd可能会抛出访问异常：

```
[root@djt_21_177 ~]## jstatd
Could not create remote object
access denied ("java.util.PropertyPermission" "java.rmi.server.ignoreSubClasses" "write")
java.security.AccessControlException: access denied ("java.util.PropertyPermission" "java.rmi.server.ignoreSubClasses" "write")
        at java.security.AccessControlContext.checkPermission(AccessControlContext.java:472)
        at java.security.AccessController.checkPermission(AccessController.java:884)
        at java.lang.SecurityManager.checkPermission(SecurityManager.java:549)
        at java.lang.System.setProperty(System.java:792)
        at sun.tools.jstatd.Jstatd.main(Jstatd.java:139)
```

这是由于jstatd没有足够的权限导致。可以使用Java的安全策略，为其分配相应的权限，下面代码为jstatd分配了最大的权限，将其保存到jstatd.all.policy中：

```
grant codebase "file:/opt/local/jdk/lib/tools.jar" {
  permission java.security.AllPermission;
};
```

然后使用以下命令再次开启jstatd服务器：

```
[root@djt_21_177 ~]## jstatd -J-Djava.security.policy=/root/jstatd.all.policy
```

### hprof 工具

hprof不是一个独立的监控工具，它只是一个Java agent工具，可以用于监控Java程序在运行时的CPU信息和堆信息。使用java -agentlib:hprof=help命令可以查看hprof的帮助文档：

```
[root@djt_21_177 ~]## java -agentlib:hprof=help

     HPROF: Heap and CPU Profiling Agent (JVMTI Demonstration Code)

hprof usage: java -agentlib:hprof=[help]|[<option>=<value>, ...]

Option Name and Value  Description                    Default
---------------------  -----------                    -------
heap=dump|sites|all    heap profiling                 all
cpu=samples|times|old  CPU usage                      off
monitor=y|n            monitor contention             n
format=a|b             text(txt) or binary output     a
file=<file>            write data to file             java.hprof[{.txt}]
net=<host>:<port>      send data over a socket        off
depth=<size>           stack trace depth              4
interval=<ms>          sample interval in ms          10
cutoff=<value>         output cutoff point            0.0001
lineno=y|n             line number in traces?         y
thread=y|n             thread in traces?              n
doe=y|n                dump on exit?                  y
msa=y|n                Solaris micro state accounting n
force=y|n              force output to <file>         y
verbose=y|n            print messages about dumps     y

Obsolete Options
----------------
gc_okay=y|n

Examples
--------
  - Get sample cpu information every 20 millisec, with a stack depth of 3:
      java -agentlib:hprof=cpu=samples,interval=20,depth=3 classname
  - Get heap usage information based on the allocation sites:
      java -agentlib:hprof=heap=sites classname

Notes
-----
  - The option format=b cannot be used with monitor=y.
  - The option format=b cannot be used with cpu=old|times.
  - Use of the -Xrunhprof interface can still be used, e.g.
       java -Xrunhprof:[help]|[<option>=<value>, ...]
    will behave exactly the same as:
       java -agentlib:hprof=[help]|[<option>=<value>, ...]

Warnings
--------
  - This is demonstration code for the JVMTI interface and use of BCI,
    it is not an official product or formal part of the JDK.
  - The -Xrunhprof interface will be removed in a future release.
  - The option format=b is considered experimental, this format may change
    in a future release.
```

使用参数 -agentlib:hprof=cpu=times,interval=10 运行程序，times会在Java函数的调用前后记录函数执行时间。

使用参数 -agentlib:hprof=heap=dump,format=b,file=dump.hprof 运行程序，可以将Java程序的堆快照文件保存在指定文件中。

使用参数 -agentlib:hprof=heap=sites 运行程序，可以输出Java程序中各个类所占的内存百分比。

