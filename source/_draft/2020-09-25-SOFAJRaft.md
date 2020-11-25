---
title: 'SOFAJRaft 01 Counter例子'
layout: post
categories: 技术
toc: true
tags:
    - SOFAJRaft, Raft
---

`SOFAJRaft`中提供了几个例子，`Counter`便是其中之一。

# 场景

`Counter`是一个基于`SOFAJRaft`实现的分布式计数器。分布式计数器由多个节点组成一个raft group，计数器可以指定步长，也可以查询计数值。所有节点之间保持一致，任何少数节点挂掉都不会影响对外提供的服务。

具体代码：[Counter](https://github.com/sofastack/sofa-jraft/tree/master/jraft-example)

# 运行

`Counter`服务端的启动入口为`CounterServer`，从`main`方法可以看出启动需要4个参数，分别是：

* dataPath：表示日志文件的磁盘路径
* groupId：表示raft group的标识
* serverId：表示本节点IP和端口
* initConf：表示全部节点IP和端口的列表

可以通过如下命令启动一个三节点的raft group

```
java com.alipay.sofa.jraft.example.counter.CounterServer /tmp/server1 counter 127.0.0.1:8081 127.0.0.1:8081,127.0.0.1:8082,127.0.0.1:8083
java com.alipay.sofa.jraft.example.counter.CounterServer /tmp/server2 counter 127.0.0.1:8082 127.0.0.1:8081,127.0.0.1:8082,127.0.0.1:8083
java com.alipay.sofa.jraft.example.counter.CounterServer /tmp/server3 counter 127.0.0.1:8083 127.0.0.1:8081,127.0.0.1:8082,127.0.0.1:8083
```

`Counter`客户端的启动入口为`CounterClient`，启动时需要2个参数：

* groupId：表示raft group的标识
* confStr：表示全部节点IP和端口的列表

通过如下命令启动客户端：

```
java com.alipay.sofa.jraft.example.counter.CounterClient counter 127.0.0.1:8081,127.0.0.1:8082,127.0.0.1:8083
```

依次启动服务端和客户端，客户端会异步发起1000次`incrementAndGet`调用，每次`delta`为`i`（0 <= i <= 999），控制台会打印出`incrementAndGet`的响应。

客户端部分日志如下：

```
Leader is 127.0.0.1:8082
incrementAndGet result:ValueResponse [value=4252, success=true, redirect=null, errorMsg=null]
incrementAndGet result:ValueResponse [value=4264, success=true, redirect=null, errorMsg=null]
incrementAndGet result:ValueResponse [value=4270, success=true, redirect=null, errorMsg=null]
incrementAndGet result:ValueResponse [value=4279, success=true, redirect=null, errorMsg=null]
incrementAndGet result:ValueResponse [value=4293, success=true, redirect=null, errorMsg=null]
incrementAndGet result:ValueResponse [value=4311, success=true, redirect=null, errorMsg=null]
...
```

服务端部分日志如下：

```
2020-09-25 14:12:36 [JRaft-FSMCaller-Disruptor-0] INFO  CounterStateMachine:102 - Added value=0 by delta=15 at logIndex=2
2020-09-25 14:12:36 [JRaft-FSMCaller-Disruptor-0] INFO  CounterStateMachine:102 - Added value=15 by delta=7 at logIndex=3
2020-09-25 14:12:36 [JRaft-FSMCaller-Disruptor-0] INFO  CounterStateMachine:102 - Added value=22 by delta=17 at logIndex=4
2020-09-25 14:12:36 [JRaft-FSMCaller-Disruptor-0] INFO  CounterStateMachine:102 - Added value=39 by delta=3 at logIndex=5
2020-09-25 14:12:36 [JRaft-FSMCaller-Disruptor-0] INFO  CounterStateMachine:102 - Added value=42 by delta=5 at logIndex=6
2020-09-25 14:12:36 [JRaft-FSMCaller-Disruptor-0] INFO  CounterStateMachine:102 - Added value=47 by delta=21 at logIndex=7
2020-09-25 14:12:36 [JRaft-FSMCaller-Disruptor-0] INFO  CounterStateMachine:102 - Added value=68 by delta=22 at logIndex=8
...
```

观察日志，会发现一个很有意思的现象，服务端收到的请求并不是按客户端发起请求的顺序，而是乱序的，这也是分布式系统一个典型的特征。虽然请求的到达是乱序的，但Raft通过写日志的顺序定义了请求的顺序，这也是Raft算法能够实现线性一致性的原因。

# Counter源码

## 服务端组件结构

`Counter`服务端由`CounterServer`、`CounterStateMachine`、`CounterService`等组件组成。

* `CounterServer`是服务端代码的启动入口，创建并设置其他各组件。
* `CounterStateMachine`是`Raft`状态机的实现，其中使用了`AtomicLong`变量来维护计数器值以及`leader`的`term`。`Raft`核心组件会通过`onXXX`的回调方法回调状态机的实现。
* `CounterService`是计数器服务层的实现，提供方法供`IncrementAndGetRequestProcessor`和`GetValueRequestProcessor`两个请求处理器调用。

## RPC请求

`Counter`例子中涉及2个RPC请求和1个RPC响应，共3个数据结构。

* `IncrementAndGetRequest`，用于递增并返回计数器值。

```java
public class IncrementAndGetRequest implements Serializable {

    private static final long serialVersionUID = -5623664785560971849L;

    private long              delta;

    public long getDelta() {
        return this.delta;
    }

    public void setDelta(long delta) {
        this.delta = delta;
    }
}
```

* `GetValueRequest`，用户查询当前计数器值。

```java
public class GetValueRequest implements Serializable {

    private static final long serialVersionUID = 9218253805003988802L;

    private boolean           readOnlySafe     = true;

    public boolean isReadOnlySafe() {
        return readOnlySafe;
    }

    public void setReadOnlySafe(boolean readOnlySafe) {
        this.readOnlySafe = readOnlySafe;
    }
}
```

* `ValueResponse`，表示响应结果，`value`表示计数器值，`success`表示请求是否成功，`errorMsg`表示失败情况下的错误信息，`redirect`表示发生重新选举时需要跳转的新的leader节点。

```java
public class ValueResponse implements Serializable {

    private static final long serialVersionUID = -4220017686727146773L;

    private long              value;
    private boolean           success;

    /**
     * redirect peer id
     */
    private String            redirect;

    private String            errorMsg;

    ...
}
```

## RPC请求处理器

`IncrementAndGetRequest`和`GetValueRequest`分别由对应的请求处理器进行处理。

* `IncrementAndGetRequestProcessor`：用于处理`IncrementAndGetRequest`请求，通过调用`CounterService`的`incrementAndGet`方法实现。
* `GetValueRequestProcessor`：用于处理`GetValueRequest`请求，通过调用`CounterService`的`get`方法实现。

## CounterService

`CounterService`接口实现了`incrementAndGet`和`get`两个方法。

`incrementAndGet`的实现是通过构造一个`Task`实例并调用`apply`方法来走一遍`Raft`协议。当`Raft`底层日志复制到多数节点并commit后，会回调`CounterStateMachine`的`onApply`方法，其中再回调`closure`参数设置的回调闭包，完成请求。

```java
public void incrementAndGet(final long delta, final CounterClosure closure) {
    applyOperation(CounterOperation.createIncrement(delta), closure);
}

private void applyOperation(final CounterOperation op, final CounterClosure closure) {
    if (!isLeader()) {
        handlerNotLeaderError(closure);
        return;
    }

    try {
        closure.setCounterOperation(op);
        final Task task = new Task();
        task.setData(ByteBuffer.wrap(SerializerManager.getSerializer(SerializerManager.Hessian2).serialize(op)));
        task.setDone(closure);
        this.counterServer.getNode().apply(task);
    } catch (CodecException e) {
        String errorMsg = "Fail to encode CounterOperation";
        LOG.error(errorMsg, e);
        closure.failure(errorMsg, StringUtils.EMPTY);
        closure.run(new Status(RaftError.EINTERNAL, errorMsg));
    }
}
```

`get`方法有点特殊，没有直接从状态机中读取计数器值，而是使用了一种`ReadIndex Read`的方法来实现线性一致性读。

在分布式环境下，`raft leader`一般能保持最新的数据，因此读请求从`leader`发起是必须的，但直接从leader状态机读并不能完全确保系统的线性一致性。比如发生网络分区，旧`leader`处于少数派分区中，且刚好在`heartbeat`时间内没能发现`leadership`丢失，如果此时直接从旧`leader`的状态机读，则很可能返回`stale`的结果。假设新`leader`已被选举出来且提交了新的记录，此时有两个客户端分别从新旧`leader`读取，从新`leader`能读取到新记录，旧`leader`只能读取到旧记录，从整个系统的角度看违背了线性一致性。

实现线性一致读常规手段是走`Raft`协议，将读请求同样按照`Log`处理，通过日志复制和状态机执行获取读结果返回给客户端，`SOFAJRaft`采用`ReadIndex`替代走`Raft`状态机的方案。具体原理以后单独成文分析。

```java
public void get(final boolean readOnlySafe, final CounterClosure closure) {
    if(!readOnlySafe){
        closure.success(getValue());
        closure.run(Status.OK());
        return;
    }

    this.counterServer.getNode().readIndex(BytesUtil.EMPTY_BYTES, new ReadIndexClosure() {
        @Override
        public void run(Status status, long index, byte[] reqCtx) {
            if(status.isOk()){
                closure.success(getValue());
                closure.run(Status.OK());
                return;
            }
            CounterServiceImpl.this.readIndexExecutor.execute(() -> {
                if(isLeader()){
                    LOG.debug("Fail to get value with 'ReadIndex': {}, try to applying to the state machine.", status);
                    applyOperation(CounterOperation.createGet(), closure);
                }else {
                    handlerNotLeaderError(closure);
                }
            });
        }
    });
}
```

## CounterStateMachine

这里`onApply`方法首先会获取`processor`中封装的数据，然后获取`processor`中传入的`closure`实例，然后处理好业务逻辑后调用`closure`的`run`进行回调返回数据到客户端。

```java
public class CounterStateMachine extends StateMachineAdapter {
    ...
    @Override
    public void onApply(final Iterator iter) {
        while (iter.hasNext()) {
            long current = 0;
            CounterOperation counterOperation = null;

            CounterClosure closure = null;
            if (iter.done() != null) {
                // This task is applied by this node, get value from closure to avoid additional parsing.
                closure = (CounterClosure) iter.done();
                counterOperation = closure.getCounterOperation();
            } else {
                // Have to parse FetchAddRequest from this user log.
                final ByteBuffer data = iter.getData();
                try {
                    counterOperation = SerializerManager.getSerializer(SerializerManager.Hessian2).deserialize(
                        data.array(), CounterOperation.class.getName());
                } catch (final CodecException e) {
                    LOG.error("Fail to decode IncrementAndGetRequest", e);
                }
            }
            if (counterOperation != null) {
                switch (counterOperation.getOp()) {
                    case GET:
                        current = this.value.get();
                        LOG.info("Get value={} at logIndex={}", current, iter.getIndex());
                        break;
                    case INCREMENT:
                        final long delta = counterOperation.getDelta();
                        final long prev = this.value.get();
                        current = this.value.addAndGet(delta);
                        LOG.info("Added value={} by delta={} at logIndex={}", prev, delta, iter.getIndex());
                        break;
                }

                if (closure != null) {
                    closure.success(current);
                    closure.run(Status.OK());
                }
            }
            iter.next();
        }
    }
    ...
}
```

## 客户端

客户端`CounterClient`比较简单，`incrementAndGet`方法中使用`cliClientService`获取`client`然后传入`request`请求并设值回调函数。

```java
public class CounterClient {

    public static void main(final String[] args) throws Exception {
        if (args.length != 2) {
            System.out.println("Useage : java com.alipay.sofa.jraft.example.counter.CounterClient {groupId} {conf}");
            System.out
                .println("Example: java com.alipay.sofa.jraft.example.counter.CounterClient counter 127.0.0.1:8081,127.0.0.1:8082,127.0.0.1:8083");
            System.exit(1);
        }
        final String groupId = args[0];
        final String confStr = args[1];

        final Configuration conf = new Configuration();
        if (!conf.parse(confStr)) {
            throw new IllegalArgumentException("Fail to parse conf:" + confStr);
        }

        RouteTable.getInstance().updateConfiguration(groupId, conf);

        final CliClientServiceImpl cliClientService = new CliClientServiceImpl();
        cliClientService.init(new CliOptions());

        if (!RouteTable.getInstance().refreshLeader(cliClientService, groupId, 1000).isOk()) {
            throw new IllegalStateException("Refresh leader failed");
        }

        final PeerId leader = RouteTable.getInstance().selectLeader(groupId);
        System.out.println("Leader is " + leader);
        final int n = 1000;
        final CountDownLatch latch = new CountDownLatch(n);
        final long start = System.currentTimeMillis();
        for (int i = 0; i < n; i++) {
            incrementAndGet(cliClientService, leader, i, latch);
        }
        latch.await();
        System.out.println(n + " ops, cost : " + (System.currentTimeMillis() - start) + " ms.");
        System.exit(0);
    }

    private static void incrementAndGet(final CliClientServiceImpl cliClientService, final PeerId leader,
                                        final long delta, CountDownLatch latch) throws RemotingException,
                                                                               InterruptedException {
        final IncrementAndGetRequest request = new IncrementAndGetRequest();
        request.setDelta(delta);
        cliClientService.getRpcClient().invokeAsync(leader.getEndpoint(), request, new InvokeCallback() {

            @Override
            public void complete(Object result, Throwable err) {
                if (err == null) {
                    latch.countDown();
                    System.out.println("incrementAndGet result:" + result);
                } else {
                    err.printStackTrace();
                    latch.countDown();
                }
            }

            @Override
            public Executor executor() {
                return null;
            }
        }, 5000);
    }

}
```