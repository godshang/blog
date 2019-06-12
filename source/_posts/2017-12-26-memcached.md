---
title: 'Memcached 的简单使用'
layout: post
categories: 技术
tags:
    - Memcached
---

Memcached 是一个免费开源的，高性能的，具有分布式对象的缓存系统，它可以用来保存一些经常存取的对象或数据，保存的数据像一张巨大的HASH表，该表以Key-value对的方式存在内存中。

<!-- more -->

## Memcached 的特点

1. 协议简单：它是基于文本行的协议，直接通过telnet在memcached服务器上可进行存取数据操作
2. 基于libevent事件处理：Libevent是一套利用C开发的程序库，它将BSD系统的kqueue,Linux系统的epoll等事件处理功能封装成一个接口，与传统的select相比，提高了性能。
3. 内置的内存管理方式：所有数据都保存在内存中，存取数据比硬盘快，当内存满后，通过LRU算法自动删除不使用的缓存，但没有考虑数据的容灾问题，重启服务，所有数据会丢失。
4. 分布式：各个memcached服务器之间互不通信，各自独立存取数据，不共享任何信息。服务器并不具有分布式功能，分布式部署取决于memcache客户端。

## Memcached 的安装与启动

Memcached安装时依赖libevent，因此先安装libevent。

在官网上下载libevent的tar包。

```
tar -zxvf libevent-2.1.8-stable.tar.gz

cd libevent-2.1.8-stable/

./configure

make

sudo make install
```

安装成功之后，就可以安装Memcached了。

同样地，到官网上下载Memcached的tar包。

```
tar -zxvf memcached-1.5.4.tar.gz

cd memcached-1.5.4

./configure --prefix=/opt/local/memcached

make && make test && sudo make install
```

进入 /opt/local/memcached/bin 目录下，执行 ./memcached 时有可能会报错：

```
[root@djt_21_177 bin]## ./memcached help
./memcached: error while loading shared libraries: libevent-2.1.so.6: cannot open shared object file: No such file or directory
```

提示libevent找不到，实际上我们之前已经安装成功了。可以创建一个软链接：

```
[root@djt_21_177 bin]## ln -s /usr/local/lib/libevent-2.1.so.6 /usr/lib64/libevent-2.1.so.6
```

之后应该就可以成功启动了。

Memcached有几个启动参数需要关注：

- -d：以守护进程启动
- -m：分配给Memcached使用的内存数量，单位是MB，默认64MB
- -u：指定运行Memcached的用户
- -l：监听的服务器IP地址，默认localhost
- -p：设置Memcached监听的端口，默认11211
- -c：最大运行的并发连接数，默认1024
- -P：设置保存Memcached的pid文件



## Memcached 命令

### set

set 命令用于将 value(数据值) 存储在指定的 key(键) 中。如果set的key已经存在，该命令可以更新该key所对应的原来的数据，也就是实现更新的作用。

set 命令的基本语法格式如下：

```
set key flags exptime bytes [noreply] 
value 
```

参数说明如下：

- key：键值 key-value 结构中的 key，用于查找缓存值。
- flags：可以包括键值对的整型参数，客户机使用它存储关于键值对的额外信息 。
- exptime：在缓存中保存键值对的时间长度（以秒为单位，0 表示永远）
- bytes：在缓存中存储的字节数
- noreply（可选）： 该参数告知服务器不需要返回数据
- value：存储的值（始终位于第二行）（可直接理解为key-value结构中的value）

例如：

```
[root@sjs_59_130 ~]## telnet 10.148.21.177 12345
Trying 10.148.21.177...
Connected to 10.148.21.177 (10.148.21.177).
Escape character is '^]'.
set key1 0 180 3
abc
STORED
```

### add

add 命令用于将 value(数据值) 存储在指定的 key(键) 中。如果 add 的 key 已经存在，则不会更新数据(过期的 key 会更新)，之前的值将仍然保持相同，并且您将获得响应 NOT_STORED。

add 命令的基本语法格式如下：

```
add key flags exptime bytes [noreply]
value
```

例如：

```
add key1 0 180 3
xyz
NOT_STORED
```

### replace

replace 命令用于替换已存在的 key(键) 的 value(数据值)。如果 key 不存在，则替换失败，并且您将获得响应 NOT_STORED。

replace 命令的基本语法格式如下：

```
replace key flags exptime bytes [noreply]
value
```

例如：

```
replace key1 0 180 3
fuc
STORED
```

### append

append 命令用于向已存在 key(键) 的 value(数据值) 后面追加数据 。

append 命令的基本语法格式如下：

```
append key flags exptime bytes [noreply]
value
```

例如：

```
append key1 0 180 1
k
STORED

get key1
VALUE key1 0 4
fuck
END
```

### prepend

prepend 命令用于向已存在 key(键) 的 value(数据值) 前面追加数据 。

prepend 命令的基本语法格式如下：

```
prepend key flags exptime bytes [noreply]
value
```

例如：

```
prepend key1 0 180 5
let's
STORED

get key1
VALUE key1 0 9
let'sfuck
END
```

### CAS

CAS（Check-And-Set 或 Compare-And-Swap） 命令用于执行一个"检查并设置"的操作。它仅在当前客户端最后一次取值后，该key 对应的值没有被其他客户端修改的情况下，才能够将值写入。

检查是通过cas_token参数进行的，这个参数是Memcach指定给已经存在的元素的一个唯一的64位值。

CAS 命令的基本语法格式如下：


```
cas key flags exptime bytes unique_cas_token [noreply]
value
```

要在 Memcached 上使用 CAS 命令，你需要从 Memcached 服务商通过 gets 命令获取令牌（token）。

gets 命令的功能类似于基本的 get 命令。两个命令之间的差异在于，gets 返回的信息稍微多一些：64 位的整型值非常像名称/值对的 "版本" 标识符。

例如：

```
get key1
VALUE key1 0 3
abc
END

gets key1
VALUE key1 0 3 15
abc
END

cas key1 0 180 5 15
redis
STORED

get key1
VALUE key1 0 5
redis
END
```

### get

get 命令获取存储在 key(键) 中的 value(数据值) ，如果 key 不存在，则返回空。

get 命令的基本语法格式如下：

```
get key
```

多个 key 使用空格隔开，如下:

```
get key1 key2 key3
```

### gets 

gets 命令获取带有 CAS 令牌存 的 value(数据值) ，如果 key 不存在，则返回空。

gets 命令的基本语法格式如下：

```
gets key
```

多个 key 使用空格隔开，如下:

```
gets key1 key2 key3
```

### delete

delete 命令用于删除已存在的 key(键)。

delete 命令的基本语法格式如下：

```
delete key [noreply]
```

参数说明如下：

- key：键值 key-value 结构中的 key，用于查找缓存值。
- noreply（可选）： 该参数告知服务器不需要返回数据

### incr 与 decr

incr 与 decr 命令用于对已存在的 key(键) 的数字值进行自增或自减操作。

incr 与 decr 命令操作的数据必须是十进制的32位无符号整数。

如果 key 不存在返回 NOT_FOUND，如果键的值不为数字，则返回 CLIENT_ERROR，其他错误返回 ERROR。

incr 命令的基本语法格式如下：

```
incr key increment_value
```

参数说明如下：

- key：键值 key-value 结构中的 key，用于查找缓存值。
- increment_value： 增加的数值。

decr 命令的基本语法格式如下：

```
decr key decrement_value
```

参数说明如下：

- key：键值 key-value 结构中的 key，用于查找缓存值。
- decrement_value： 减少的数值。

例如：

```
set count 0 89 2
10
STORED

get count
VALUE count 0 2
10
END

incr count 2
12

decr count 3
9

get count
VALUE count 0 2
9 
END
```

### stats

stats 命令用于返回统计信息例如 PID(进程号)、版本号、连接数等。

stats 命令的基本语法格式如下：

```
stats
```

### stats items

stats items 命令用于显示各个 slab 中 item 的数目和存储时长(最后一次访问距离现在的秒数)。

stats items 命令的基本语法格式如下：

```
stats items
```

### stats slabs

stats slabs 命令用于显示各个slab的信息，包括chunk的大小、数目、使用情况等。

stats slabs 命令的基本语法格式如下：

```
stats slabs
```

### stats sizes

stats sizes 命令用于显示所有item的大小和个数。该信息返回两列，第一列是 item 的大小，第二列是 item 的个数。

stats sizes 命令的基本语法格式如下：

```
stats sizes
```

### flush_all

flush_all 命令用于用于清理缓存中的所有 key=>value(键=>值) 对。

该命令提供了一个可选参数 time，用于在指定的时间后执行清理缓存操作。

flush_all 命令的基本语法格式如下：

```
flush_all [time] [noreply]
```

## Memcached 的设计

### Memcached 内存算法

Memcached利用slab allocation机制来分配和管理内存，它按照预先规定的大小，将分配的内存分割成特定长度的内存块，再把尺寸相同的内存块分成组，数据在存放时，根据键值 大小去匹配slab大小，找就近的slab存放，所以存在空间浪费现象。

传统的内存管理方式是，使用完通过malloc分配的内存后通过free来回收内存，这种方式容易产生内存碎片并降低操作系统对内存的管理效率。Memcached的内存管理制效率高，而且不会造成内存碎片，但是它最大的缺点就是会导致空间浪费。因为每个 Chunk都分配了特定长度的内存空间，所以变长数据无法充分利用这些空间。

### Memcached的缓存策略

Memcached的缓存策略是LRU（最近最少使用）加上到期失效策略。当你在memcached内存储数据项时，你有可能会指定它在缓存的失效时间，默认为永久。当memcached服务器用完分配的内时，失效的数据被首先替换，然后也是最近未使用的数据。在LRU中，memcached使用的是一种Lazy Expiration策略，自己不会监控存入的key/vlue对是否过期，而是在获取key值时查看记录的时间戳，检查key/value对空间是否过期，这样可减轻服务器的负载。

### Memcached的分布式算法

Memcached 虽然称为“分布式”缓存服务器，但服务器端并没有“分布式”功能。互不通信，怎么实现分布式？事实上，Memcached 的分布式是完全由客户端程序库实现的。这种分布式是 Memcached 的最大特点。通过这种方式，Memcached server 之间的数据不需要同步，也就不需要互相通信了。

当向memcached集群存入/取出key/value时，memcached客户端程序根据一定的算法计算存入哪台服务器，然后再把key/value值存到此服务器中。也就是说，存取数据分二步走，第一步，选择服务器，第二步存取数据。

选择服务器算法有两种，一种是根据余数来计算分布，另一种是根据一致性哈希算法来计算分布。

### 适用场景

1. 网站包含了访问量很大的动态网页，因而数据库的负载将会很高, 且大部分数据库请求都是读操作；
2. 数据库服务器的负载比较低，CPU 使用率却很高；
3. 小型需要共享的数据，如 session 等临时数据；
4. 缓存一些很小但是被频繁访问的文件。图片这种大点儿的文件就由 CDN（内容分发网络）来处理了。

### 不适用场景

1. 缓存对象的大小大于 1 MB, Memcached 本身就不是为了处理庞大的多媒体和巨大的二进制块而设计的，如果你任性，要存这么大的数据，可以自己修改源代码，它是开源的，不过请慎改；
2. key 的长度大于 250 字符（硬性要求）；
3. 环境不允许运行 memcached 服务，如虚拟主机；
4. 应用运行在不安全的环境中，Memcached 未提供任何安全策略，仅仅通过 telnet 就可以访问到 memcached。数据安全越来越重要了，so，请把它放在防火墙后；
5. 业务需要的是持久化数据时请使用数据库。

### 使用时注意事项

- key 不能有空格和控制字符。
- 在 Memcached Memcached 中可以保存的 item 数据量是没有限制的，只有内存足够
- Memcached Memcached单进程最大使用内存为 单进程最大使用内存为2G，要使用更多内存，可以分多个端口开启多个 
- 最大30天的数据过期时间 天的数据过期时间, 设置为永久的也会在这个时间过期，常量REALTIME_MAXDELTA REALTIME_MAXDELTA 60*60*24*30 控制
- 最大键长为250字节，大于该长度无法存储，常量 KEY_MAX_LENGTH 250 KEY_MAX_LENGTH 250 控制
- 单个item最大数据是1MB，超过1MB数据不予存储，常量 POWER_BLOCK 1048576 POWER_BLOCK 1048576 进行控制，它是默认的slab大小
- 最大同时连接数是 200，通过 conn_init() conn_init()中的freetotal freetotal 进行控制，最大软连接数是 1024，通过 settings.maxconns=1024 settings.maxconns=1024 进行控制
- 跟空间占用相关的参数：settings.factor=1.25, settings.chunk_size=48, 影响slab的数据占用和步进方式

