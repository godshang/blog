---
title: 'Apache Kafka Consumer'
layout: post
categories: 技术
toc: true
tags:
    - Apache Kafka
---

# 概述

同producer一样，早期版本的Kafka中的consumer代码也是使用Scala开发的。自0.9.0.0版本起，Kafka提供了Java版本的consumer。新版本consumer的入口类是org.apache.kafka.clients.consumer.KafkaConsumer。新版的consumer也不再需要依赖ZooKeeper。

在旧版本consumer中，消费位移（offset）的保存与管理都是依托ZooKeeper来完成大。当数据量很大且消费很频繁时，ZooKeeper的读/写性能往往容易成为系统瓶颈。新版本consumer中，位移的管理与保存不再依靠ZooKeeper，这个瓶颈自然消失了。

新版consumer在设计时摒弃了旧版本多小城消费不同分区的思想，采用了类似Linux epoll的轮训机制，使得consumer只使用一个线程就可以管理连接不同broker的多个Socket，既减少了线程间的开销，同时也简化了系统的设计。

比起旧版本consumer，新版本在设计上的优势如下。

* 单线程设计。单个consumer线程就可以管理多个分区的消费Socket连接，极大地简化了实现。
* 位移提交与保存交由Kafka来处理。位移不再保存在ZooKeeper中，而是单独保存再Kafka的一个内部topic中。
* 消费者组的集中式管理。旧版中ZooKeeper除了要管理位移，它还要负责管理整个消费者组的成员，这进一步加重了对于ZooKeeper的依赖。新版consumer改进了这种设计，实现了一个集中式协调者的角色，所有成员组的管理都交由该coordinator负责。

旧版本consumer有高阶消费者（high-level consumer）和低阶消费者（low-level consumer）之分。high-level consumer指消费者组，low-level consumer指单个consumer，即standalone consumer。单个consumer单独进行自己的工作，与其他consumer不产生任何关联；而消费者组就是大家作为一个团队一起工作，彼此之间会相互照应。

low-level consumer的实现是SimpleConsumer类。low-level consumer需要自行管理消费者，Kafka不会为其提供任何的组管理方面的功能（包括负载均衡和故障转移），用户需要自己解决这方面的问题。

high-level consumer的consumer实例会自动组成一个消费者组来共同承担消费人物，假如任意时刻有consumer进程或实例宕机，该消费者组都会帮用户自动处理，不需要人工干预。但是，high-level consumer不像low-level consumer那样灵活，可以从分区的任意位置开始消费，它只能从上次保存的位移处开始顺序读取消息，无法实现高度定制化的消费策略。

## 消费者组

Kafka同时支持基于队列和基于发布/订阅的两种消息引擎模型。

* 所有consumer实例都属于相同group，实现基于队列的模型，每条消息只会被一个consumer实例处理。
* consumer实例都属于不同group，实现基于发布/订阅的模型。极端情况是每个consumer实例都设置完全不同的group，这样消息会被广播到所有consumer实例上。

consumer group可以实现高伸缩性、高容错性的consumer机制。组内多个consumer实例可以同时读取Kafka消息，而且一旦有某个consumer挂了，consumer group会立即将已崩溃consumer实例负责的分区转交给其他consumer来负责，从而保证整个consumer group继续工作，不会丢失数据。这个过程被称为重平衡。

Kafka目前只提供单个分区的消息有序，而不会维护全局的消息顺序，因此如果用户要实现topic全局的消息读取顺序，就只能通过让每个consumer group下只包含一个consumer实例的方式来间接实现。

## 位移

指consumer端的offset，与分区日志中的offset是不同的含义。每个consumer实例都会为它消费的分区维护属于自己的位置信息来记录当前消费了多少条消息。很多消息引擎都把消费端的offset保存再服务器，这样的好处是实现简单，但会有3个问题：

* broker从此变成有状态的，增加了同步成本，影响伸缩性。
* 需要引入应答机制（acknowledgement）来确认消费成功。
* 由于要保存许多consumer的offset，必然引入复杂的数据结构，从而造成不必要的资源浪费。

Kafka选择了不同的方式，让consumer group保存offset。同时Kafka consumer还引入了检查点机制，定期对offset进行持久化。

## 位移提交

consumer客户端需定期向Kafka集群汇报自己消费的进度，这一过程被称为位移提交。

新旧版本consumer提交位移的方式截然不同，旧版本consumer会定期将位移信息提交到ZooKeeper下的固定节点，而新版本consumer把位移提交到Kafka的一个内部topic（__consumer_offsets）上。因此，新版本的consumer也不再依赖ZooKeeper。

__consumer_offsets这个topic通常是提供给新版本consumer使用的。但是，旧版本consumer也提供了一个特定的参数让用户在使用旧版本consumer时把位移提交到这个topic上，这个参数是offsets.storage=kafka。不过这种做法很少见。

__consuemr_offsets是Kafka自行创建的，用户不可擅自删除该topic的所有信息。

__consumer_offsets默认分区是50个。每个分区目录下至少会有一个日志文件（.log）和两个索引文件（.index和.timeindex）。该日志中保存的消息都是Kafka集群上consumer（特别是consumer group）的位移信息。可以理解为每个消息是的key是一个三元组：group.id + topic + 分区号，而value就是offset的值。每当更新同一个key的最新offset值时，该topic就会写入一条含有最新offset的消息，同时Kafka会定期对该topic执行压实操作（compact），即为每个消息key只保存含有最新offset的消息。这样既避免了对分区日志消息的修改，也控制了总体的日志容量，同时还能实时反映最新的消费进度。

# consumer主要参数

**session.timeout.ms**

session.timeout.ms是consumer group检测组内成员发生崩溃的时间，例如设置该参数为5分钟，那么当group中某个成员突然崩溃了，Kafka coordinator可能需要5分钟才能感知到这个崩溃。显然我们想要缩短这个时间，让coordinator能更快的检测到consumer失败。

同时，这个参数还有另外一重含义，即consumer消息处理逻辑的最大时间。倘若consumer两次poll之间的间隔超过了该参数所设置的阈值，那么coordinator就会认为这个consumer已经追不上组内其他成员的消费进度了，因此会将该consumer实例踢出组，该consumer负责的分区也会被分配给其他consumer。

0.10.1.0版本后对该参数的含义进行了拆分。session.timeout.ms参数被明确为coordinator检测失败的时间，实际中可以设置为一个较小的值让coordinator能更快地检测到consumer崩溃的情况。目前该参数默认值是10秒。max.poll.interval.ms参数定义为consumer处理逻辑的最大时间。

**auto.offset.rest**

指定了无位移信息或位移越界时Kafka的应对策略。目前该参数有如下3个可能的取值。

* earliest：指定从最早的位移开始消费，注意这里的最早位移不一定就是0。
* latest：指定从最新处位移开始消费。
* none：指定如果未发现位移信息或位移越界，则抛出异常。该参数值极少使用。

**enable.auto.commit**

该参数指定consumer是否自动提交位移。若设置为true，则congsumer在后台自动提交位移；否则，用户需要手动提交位移。

**fetch.max.bytes**

该参数设置consumer单次获取数据的最大字节数。若实际业务消息很大，必须要设置该参数为一个较大的值，否则无法消费这些消息。

**max.poll.records**

该参数控制单次poll调用返回的最大消息数。极端的做法是设置为1，那么每次poll只返回1条消息。默认为500，如果消息处理逻辑很轻量级，那么可以适当调大。

**heartbeat.interval.ms**

当coordinator决定开启一轮新的rebalance时，它会将rebalance决定以REBALANCE_IN_PROGRESS异常的形式塞进consumer心跳请求的response中，这样其他成员拿到response后才能知道它需要重新加入group。显然这个过程越快越好，而heartbeat.inverval.ms这个参数就是用来做这件事的。

注意，该值必须小于session.timeout.ms！

**connections.max.idle.ms**

该参数表示consumer与Kafka broker之间的Socket连接最大的空闲时间，默认值是9分钟。如果实际中不在乎这些Socket资源开销，可以设置为-1，即不要关闭这些空闲连接。

# poll内部原理

旧版本consumer为每个要读取的分区都创建一个专有的线程去消费。新版本consumer使用了类似Linux I/O模型中的poll或select等，使用一个线程同时管理多个Socket连接，即同时与多个broker通信实现消息的并行读取。

新版本consumer是一个多线程或者说是一个双线程的Java进程，创建KafkaConsumer的线程被称为用户主线程，同时consumer在后台会创建一个心跳线程，该线程被称为后台心跳线程。KafkaConsumer的poll方法在用户主线程中运行。这也表明，消费者组执行rebalance、消息获取、coordinator管理、异步任务结果的处理甚至位移提交等操作都是运行在用户主线程中的。