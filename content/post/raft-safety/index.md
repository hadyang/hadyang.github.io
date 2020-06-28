---
title: Raft 安全性
date: 2020-06-27
draft: true
categories: 
    - 深入理解ETCD
---

## 安全性


返回给客户端成功则一定写入成功，如果返回失败或异常则不一定写入失败



### 提交前任的日志

当 Leader 将日志复制到超过半数节点后，就会提交这个日志。如果在提交前，Leader 故障了，那么新任 Leader 会尝试完成这个日志的复制过程。然而，新 Leader 不能很快的计算出之前的日志是否已经复制到超过半数的节点。因此，即使日志已经复制完成，但是提交前 Leader 故障，也可能会出现日志覆盖的情况。




Raft 何时回复 Client 的写请求？
commit  apply



Follower 收到 请求投票，需要校验什么？


Raft AppendEntries 只判断 pre_log ？