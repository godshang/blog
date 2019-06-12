---
title: '徒手撸一个Raft实现'
layout: post
toc: true
categories: 技术
tags:
    - Distributed Systems
---

Raft中有三种角色，我们先来定义一个枚举来表示：

```
public enum State {

    LEADER, FOLLOWER, CANDIDATE;
}
```

