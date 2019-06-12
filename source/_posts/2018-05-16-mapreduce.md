---
title: '6.824, MapReduce'
layout: post
toc: true
categories: 技术
tags:
    - Distributed Systems
---

6.824 是 MIT 的分布式系统课程，主要形式就是读论文，还有几个工程项目。

第一课是 Google 的经典之作 [MapReduce/](http://nil.csail.mit.edu/6.824/2018/papers/mapreduce.pdf)。

MapReduce 的基本概念非常简单，Map 和 Reduce 来源于函数式编程中的概念。 MapReduce 的最大卖点就是它容易理解且扩展性强，很多计算任务都可以使用 Map/Reduce 表达，同时框架又对开发人员屏蔽了 parallelization, fault-tolerance, locality optimization, load balancing 这些分布式系统中常见的细节。

时至今日， MapReduce 的计算模型已经广为人知，这里从略，记录下一些其他的方面。

* 计算靠近数据：网络带宽是系统的一大瓶颈，所以要尽可能减少网络传输。 MapReduce 建立在 GFS 文件系统上，和 GFS 共享机器。 GFS 将文件按 64MB 大小切分成快，并在不同机器上存储多个副本（通常是3副本）。 MapReduce 系统的 master 根据这些信息，尽可能地把 Map 任务分配到保存着相应数据的机器上来做，这样 Map 任务就只需要从本地硬盘读了。如果做不到的话，也尽量分配到附近的机器上，减少通过交换机的数据量。

* 另外一个非常有用的优化是 Combiner ，简单来说就是 Map 任务在本地先做一次 Reduce （这时不叫 Reduce 叫 Combiner ）再写入本地硬盘，然后 Reducer 那边要读的远程数据就可能大大减少。比如 word count 问题，与其写一大堆 (word, "1")，不如先在本地加一下。当然，这个要求 reduce 的运算符满足交换律和结合律才行。

* 还有一个优化是处理拖后腿的任务。实践中容易发现总会有少数几个 task 出奇地慢（Hadoop/Hive经常卡在99%上），原因多种多样，比如碰巧有台机器的硬盘慢了。解决方法是当总进度接近完成时，对于还没结束的任务，重复启动一份，两份同时跑，哪个先运行完算哪个。实验显示这样可以用很少的额外运算量换取总完成时间的大大减少。

* Partition 函数。用户可以指定 Reduce 任务（即output文件）的个数（假设为R），此时 Map 任务产生的中间数据默认会根据公式 `hash(key) mod R` 分片到 Reduce 任务中。在一些场景下，比如 key 是 URL 的情况，希望相同 host 的数据会输出到同一个文件中，此时用户可以自由指定 Partition 函数，实现这一需求。

* 失败处理。
	
	* 如果 Master 挂了，按论文里的说法好像就不试图恢复了，直接报错让客户端选择是否重试。

	* master保存着所有task的信息，比如状态是没开始、进行中还是完成了，运行中或者完成了的任务是在哪个worker上运行的，等等。master会持续ping所有worker，如果ping不通就认为worker挂了。挂了的worker上的进行中的map任务和reduce任务，还有完成了的map任务，都需要在别的机器上重启。这里要注意的是完成了的reduce任务就不用重启了，因为输出文件已经写到了全局的GFS上；完成了的map任务则需要重启，因为reducer还需要远程读它的本地输出文件，而这机器已经挂了。

	* 如果ping不通的机器其实没挂也没问题，这时会出现多个map task或reduce task处理同一份数据。如果是多个mapper，先结束的那个会把自己的本地输出文件的位置上报给master，后结束的那个上报的会被master忽略，对结果没有影响。如果是多个reducer，我们利用GFS提供的atomic rename功能，保证只能有一个reducer写入GFS成功。