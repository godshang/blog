---
title: 'Apache Kafka Producer'
layout: post
categories: 技术
toc: true
tags:
    - Apache Kafka
---

# 概述

早期版本的Kafka中，producer代码是使用Scala开发的。自0.9.0.0版本起，Kafka正式使用Java版本的producer替换了原Scala版本的producer。新版本producer的入口是 org.apache.kafka.clients.producer.KafkaProducer，而非原来的 kafka.producer.Producer。

新版本的producer重写了之前服务器端代码提供的很多数据结构，摆脱了对服务器端代码库的依赖，同时新版本的producer也不再依赖ZooKeeper，甚至不需要和ZooKeeper集群进行直接交互，降低了系统的维护成本，也简化了部署producer应用的开销。

比起旧版本的producer，新版本在设计理念上有以下特点。

* 发送过程被划分到两个不同的线程：用户主线程和Sender I/O线程，逻辑更容易把控。
* 发送是异步发送消息，并提供回调机制用于判断发送是否成功。
* 批次机制，每个批次中包含多个发送请求，提升整体吞吐量。
* 更加合理的分区策略：对于没有指定key的消息，旧版本producer分区策略是默认在一段时间内将消息发送到固定分区，这更容易造成数据倾斜；新版本采用轮训机制，消息发送更加均匀。
* 底层统一使用基于Java Selector的网络客户端，结合Java的Future实现更加健壮和有呀的生命周期管理。
* 监控指标更加完善。

producer向某个topic发送消息，首先要确认的就是要想topic的哪个分区写入消息，这就是分区器（partitioner）的职责。Kafka producer提供了一个默认的分区器，对于每条待发送的消息，如果该消息指定了key，那么该partitioner会根据key的哈希值来选择目标分区；若这条消息没有指定key，则partitioner使用轮训的方式确认目标分区，这样可以最大限度地确保消息在所有分区上的均匀性。用户也可以在发送消息时跳过partitioner直接指定要发送到的分区。

producer允许用户实现自定义的分区策略而非使用默认的partitioner，这样用户可以根据自身业务需求确定不同的分区策略。

有了partitioner，我们可以确信具有相同key的所有消息都会被路由到相同的分区中。基于这个特性，我们实现一些特定的业务需求。

在确认目标分区后，producer需要寻找这个分区对应的leader，也就是该分区leader副本所在的Kafka broker。每个topic分区都由若干副本组成，其中的一个副本充当leader的角色，只有leader才能响应客户端发送的请求，而剩下的副本有一部分副本会与leader保持同步，即ISR。因此在发送消息时，producer也就有了多种选择，比如不等待任何副本的响应就返回成功，或者只等待leader副本响应再返回成功等。

# producer主要参数

**acks**

acks参数用于控制producer生产消息的持久性。对于producer而言，Kafka在乎的是“已提交”消息的持久性。一旦消息被成功提交，那么只要有任何一个保存了该消息的副本存货，这条消息就会被视为不会丢失。

具体来说，当producer发送一条消息个Kafka集群时，这条消息会被发送到指定topic分区的leader分区上，producer等待从该leader broker返回消息的写入结果以确认消息被成功提交。这一切完成后producer可以继续发送新消息。Kafka保证consumer永远不会读取到尚未提交完成的消息。

acks决定了再producer发送响应前，leader必须确保已成功写入消息的副本数。acks当前有3个取值：0、1和all。

* acks = 0。表示producer完全不理睬leader broker端的处理结果。此时，producer发送消息后立即开启下一条消息的发送，不等待leader broker端返回结果。这种情况，producer回调完全不起作用，用户无法通过回调感知任何发送过程中的失败，所以acks=0并不保证消息会被发送成功。但这种设置下producer的吞吐量最高。
* acks = all或者-1。表示发送消息时，leader broker不仅会将消息写入本地日志，同时还会等待ISR中所有其他副本都成功写入它们各自的本地日志后，才发送响应给producer。这种设置下，只要ISR中至少有一个副本处于存活状态，那么这条消息就不会丢失，可以达到最高的消息持久性；但同时producer的吞吐量是最低的。
* acks = 1。是0和all折中的方案，也是默认的参数。producer发送消息后，leader broker仅将该消息写入本地日志，然后便发送响应给producer，而无须等待ISR中其他部分写入消息。那么只要leader broker一直存活，Kafka就能保证消息不丢失。这是一种折中方案，既可以达到适当的消息持久性，同时也保证producer端的吞吐量。

**buffer.memory**

该参数指定producer端用于缓存消息的缓冲区大小，单位是字节，默认32MB。由于采用异步发送的架构，producer启动时会先创建一块内存缓冲区用于保存待发送的消息，然后另一个线程负责从缓冲区中读取消息执行真正的发送。这部分内存空间的大小就是由buffer.memory参数指定。若producer写缓冲区的速度超过专属I/O线程发送消息的速度，缓冲区空间会不断增大。此时producer会停止工作等待I/O线程追上，若一段时间后I/O线程仍然追不上producer的进度，那么producer会抛出异常。

**compression.type**

该参数设置producer端是否压缩消息，默认值是none，即不压缩。producer引入压缩后可以显著降低网络I/O传输开销从而提升整体吞吐量，但也会增加producer端机器的CPU开销。另外，如果broker端的压缩参数与producer不同，broker端在写入消息时也会增加额外的CPU资源对消息进行解压-重压缩操作。

目前Kafka支持3中压缩算法：GZIP、Snappy和LZ4。比较而言，LZ4性能最好。

**retries**

Kafka broker在处理写请求时可能因为瞬时故障（如leader选举或者网络抖动）导致消息发送失败，这种故障通常是可自行恢复的，如果把这种错误返回给回调由用户处理，用户一般也只是重试。producer内部提供了自动重试机制，前提是设置retries参数。该参数表示重试的次数，默认是0，即不重试。

设置retries有两点需要注意：

* 重试可能造成消息的重复发送。
* 重试可能造成消息的乱序。

producer两次重试之间会停顿一段时间，以防止频繁的重试对系统带来的冲突。该时间由参数retry.backoff.ms指定，默认是100毫秒。

**batch.size**

这是producer最重要的参数之一，它对于调优producer吞吐量和延时指标都有非常重要的作用。producer会将发往同一分区的多条消息封装进一个batch中，当batch满了的时候，producer会发送batch中的所有消息。不过，producer并不总是等待batch满了才发送消息，很有可能当batch还有很多空闲空间时producer就发送该batch。

batch越小，一次发送的消息数也越少，producer的吞吐量越低；但batch非常大，会给内存带来很大压力，因为不管是否能够填满，producer都会为改batch分配固定大小的内存。

batch.size参数的默认值是16384，即16KB。这是一个非常保守的数字，实际中可以合理增加该参数，通常producer的吞吐量都会得到相应增加。

**linger.ms**

当batch没满的时候，如果延时达到了linger.ms参数设置的时间，也会立即发送。这就在吞吐量和延时之间达到一种平衡。该参数默认值是0，表示消息立即被发送，无须关心batch是否已填满。不过这会拉低producer的吞吐量。

**max.request.size**

该参数用于控制producer发送请求的大小。实际上该参数控制的是producer端能够发送的最大消息大小。如果producer要发送尺寸很大的消息，那么这个参数就是要被设置的。该参数默认值是1048576字节，即1MB。

**request.timeout.ms**

当producer发送请求给broker后，broker需要在规定的时间范围内将处理结果返还给producer。这段时间就是该参数控制的，默认是30秒。如果broker在30秒内都没返回给producer响应，那么producer就会认为该请求超时了，并在回调函数中抛出TimeoutException异常。

# 分区机制

Kafka producer发送过程中会使用分区器（partitioner）来决定将消息发送到topic的哪个分区中。默认的分区器会尽力确保具有相同key的消息都会被发送到相同的分区上，若消息没有指定key则会选择轮询的方式来确保在topic的所有分区上均匀分配。除此之外，也可以自定义分区器实现自定义的分区策略。

实现自定义分区器，需要实现org.apache.kafka.clients.producer.Partitioner接口，并在构造KafkaProducer时在Properties对象中设置partitioner.class参数。

# 消息序列化

序列化器（serializer）负责在producer发送前将消息转换成字节数组，反之，解序列化器（deserializer）用于将consumer接收到的字节数组转换成相应的对象。Kafka对常用基本数据类型都提供了默认的序列化器和解序列化器，但对复杂的类型需要用户自行定义。

实现自定义序列化器，需要实现org.apache.kafka.common.serialization.Serializer接口，并在构造KafkaProducer时在Properties对象中设置key.serializer或value.serializer。

# 拦截器

producer拦截器是一个比较新的功能，可以在消息发送前以及producer回调前有机会对消息做一些定制化需求。同时，producer允许指定多个拦截器按序作用于同一条消息从而形成一个拦截器链。

拦截器的实现接口是org.apache.kafka.clients.producer.ProducerInterceptor，其定义的方法如下：

* onSend(ProducerRecord)：该方法运行在用户主线程中，即调用KafkaProducer.send方法的线程。producer确保在消息被序列化以及计算分区前调用该方法。
* onAcknowledgement(RecordMetadata, Exception)：该方法会在消息被应答前或消息发送时调用，并且通常都是在producer回调逻辑触发之前。该方法运行在producer的I/O线程中，因此不要在该方法中放入很重的逻辑，否则会拖慢producer的消息发送效率。

# 无消息丢失配置

Java版本的producer采用异步发送机制，发送消息仅仅是把消息放入缓冲区，由一个专属的I/O线程负责从缓冲区中提取消息并封装仅消息batch中，然后发送出去。这个过程中存在着数据丢失的窗口：若I/O线程发送之前producer崩溃，则缓冲区中的消息全部丢失。

producer的另一个问题就是消息乱序。假设producerproducer一次发送reocord1、record2两条消息，由于某些原因（比如网络的瞬时抖动）导致record1未发送成功，同时Kafka又配置了重试机制以及max.in.flight.requests.per.connection大于1（默认值是5），那么producer重试record1成功后，record1在日志中的位置反而位于record2之后，这样就造成了消息的乱序。

为了实现producer端“无消息丢失配置”，那么可以按照如下设置参数：

* block.on.buffer.full = true
* acks = all or -1
* retries = Integer.MAX_VALUE
* max.in.flight.requests.per.connection = 1
* 使用带回调的send方法发送消息
* 回调中显示地立即关闭producer，使用close(0)
* unclean.leader.election.enable = false
* replication.factor = 3
* min.insync.replicas = 2
* replication.factor > min.insync.replicas
* enable.auto.commit = false

# 多线程处理

实际中，经常会有多个线程或多个进程来同时给Kafka集群发送消息。这样就存在着两种用法：

* 多线程单KafkaProducer实例。
* 多线程多KafkaProducer实例。

KafkaProducer是线程安全的，因此可以构造一个全局的KafkaProducer实例，然后在多个线程中共享使用。当然，也可以在每个producer线程中都构造一个KafkaProducer实例，并且保证此实例在该线程中封闭。

单KafkaProducer实例的优势是实现简单、性能好；劣势是所有线程共享一个内存缓冲区，可能需要较多内存，此外一旦producer某个线程崩溃导致KafkaProducer实例被破坏，则所有线程都无法工作。

多KafkaProducer实例的优势是每个线程拥有独立的缓冲区以及一组对应的配置参数，可以进行细粒度的调优，单个KafkaProducer实例的崩溃也不会影响其他线程中KafkaProducer的工作；劣势是需要较大的内存分配开销。

对于分区数不多的Kafka集群而言，推荐单Kafka实例；对于拥有超多分区的集群而言，采用多KafkaProducer实例的方法具有较高的可控性。


# 旧版本producer

旧版本producer使用Scala语言编写，已经在0.9.0.0版本中被证实废弃。新旧版本的主要区别如下：

* 旧版本producer入口是kafka.producer.Producer，新版本producer的入口是org.apache.kafka.clints.producer.KafkaProducer。
* 旧版本producer默认同步发送，新版本默认异步发送。
* 旧版本引用kafka-core依赖，新版本则需要依赖kafka-clients。
* 新旧版本的配置参数几乎完全不同。
* 旧版本直接与ZooKeeper通信发送数据，新版本彻底摆脱ZooKeeper的依赖。