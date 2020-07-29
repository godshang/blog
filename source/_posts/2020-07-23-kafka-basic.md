---
title: 'Apache Kafka 概念解析'
layout: post
categories: 技术
toc: true
tags:
    - Apache Kafka
---

# Kafka 简介

早期的Kafka对自己的定位是一个“分布式的、分区化的、带备份机制的提交日志服务”，本质上是一个分布式的消息队列。现在的Kafka将自己的定位调整为“分布式的流处理平台”，用来构建实时的数据流水线和流应用程序。可以看出，随着Kafka的不断发展，其野心也是逐渐显现。

Kafka归根到底其核心仍然是高性能的消息发送与消费的引擎。主要的特性包括：

* 高吞吐量、低延时。
    * Kafka大量使用页缓存，内存操作速度快且命中率高。
    * Kafka不直接与底层文件系统打交道，繁琐的I/O操作都交由操作系统处理。 
    * 写入时采用了追加写的方式，避免磁盘随机写操作。
    * 使用以sendfile为代表的零拷贝技术加强网络间的数据传输效率。
* 消息持久化。
* 负载均衡和故障转移。
    * 通过智能的分区领导者选举（partition leader election）实现集群的所有机器以均等几回分散各个partition的leader。
    * 使用会话机制实现故障转移，即ZooKeeper。
* 伸缩性。
    * 无状态：Kafka服务器的状态（不是所有状态，服务器只保留了轻量级的内部状态）交由ZooKeeper保管。

# Kafka 基本概念与术语

* topic
topic是一个逻辑概念，代表了一类消息。
* partition
topic通常会被多个消费者订阅，出于性能的考虑，Kafka并不是topic-message的两级结构，而是采用了topic-partition-message的三级结构来分散负载。partition并没有太多的业务含义，它的引入只是单纯地为了提升系统的吞吐量。
* offset
topic partition下的每条消息都被分配一个位移值。Kafka的消费者也有一个位移的概念，但要注意这两个offset属于不同的概念。
* replica
replica是为了实现高可靠性采用的冗余机制，简单来说就是备份多份日志，这些备份日志被称为副本（replica）。replica的唯一目的就是防止数据丢失。replica分为leader replica和follower replica，follower replica是不能提供服务的，也就是说不能响应客户端发送的消息写入和消息消费请求。
* leader和follower
通常只有leader对外提供服务，follower只是被动地追随leader的状态，保持与leader的同步。follower存在的唯一价值就是充当leader的候补。
* ISR
ISR全称是in-sync replica。ISR是一个动态的replica集合，该集合中的所有replica保存的消息日志都与leader replica保持同步状态。只有这个集合中的replica才能被选为leader replica，也只有这个集合中的所有replica都接收到了同一条消息，Kafka才会将该消息置为“已提交”状态。
正常情况下，patition所有的replica都应该与leader replica保持同步，即所有的replica都在ISR中。因为各种原因，一小部分replica开始落后于leader replica的进度。当滞后到一定程度时，Kafka会将这些replica从ISR中踢出。当落后的replica重新追上leader的进度时，会被重新加入ISR。

