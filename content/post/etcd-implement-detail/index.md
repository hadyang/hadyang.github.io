---
title: ETCD 实现细节
date: 2020-07-06
draft: true
categories:
    - 深入理解ETCD
---

## 物理模型

ETCD 的存储模型是 B+ 数


## 对象层级

etcdServer

- Lessor
- backend
- raftNode
  - Node：Raft 集群中的一个节点
    - RawNode （基本属于 Node 的实现）
      - raft：状态机
        - ProgressTracker 包含其他每个节点的信息
          - Voters
        - raftLog
        - 
  - transport
    - id:peer ：代表一个远端的节点，本地节点通过 peer 和远端节点通信，包含两种发送消息的机制： stream 和 pipeline
      - pipeline：是 Leader 用来发送 MsgApp （AppendEntries） 消息的，，pipeline 是优化过的（因为 MsgApp 消息很多），pipeline 将一系列请求发送到远端。只有在 stream 不可用时才会使用
      - stream：是接收者初始化的 轮训 连接，长时间开启并接收消息。


BatchTx 是整个 backend 长期持有的

bbolt.Tx 是每次 BatchTx commit（也会进行 bbolt.Tx 的 commit） 之后重新 begin 的 

- storeTxnWrite
  - BatchTx
    - bbolt.Tx



db 表示在磁盘上 buckets 的集合。所有的数据操作都需要通过 db 的事务。

Bucket 在 db 中代表 kv 的集合

//node represents an in-memory, deserialized page.

node 代表一个在内存中的（未写入？）的页，bucket 里就是一个 b+树，node 就是b+树的节点

// leafPageElement  represents a node on a leaf page.
leafPageElement 代表的应该是一个 inode

// branchPageElement represents a node on a branch page.

// inode represents an internal node inside of a node.
// It can be used to point to elements in a page or point
// to an element which hasn't been added to a page yet.
inode 是 node 内部的 node，其可以指向 page 中的 element 或者无 element（element在 spill 时指定）

- backend
  - batchTx
  - db (bolt 文件)
    - bucket
      - node
        - inodes
        - childs(node) 只是做临时存放，每次 spill 后会置为空 //We no longer need the child list because it's only used for spill tracking.


## 代码结构


raftNode 包含 Node、 transport



Node 表示 

Node 包含 RawNode 


RawNode 包含 Raft



raft 包含 raft协议的基本内容
- 超时时间
- 
- 状态

etcd 会自己连接 Client 端口


remote

## PUT 请求流程

```go
rd := Ready{
		Entries:          r.raftLog.unstableEntries(),
		CommittedEntries: r.raftLog.nextEnts(),
		Messages:         r.msgs,
    }
    
ap := apply{
	entries:  rd.CommittedEntries,
	snapshot: rd.Snapshot,
	notifyc:  notifyc,
}
```

接口 mvcc.ConsistentWatchableKV 实现 watchableStore

revision 在 b+ 树中存储结构 key:putK,value:{mian:sub} , putV 存储在



### 连接 Leader

etcdServer.Put -> etcdServer.RaftRequest -> node.Propose -> raftNode.stepWithWaitOption -> raft.Step -> stepLeader -> raft.appendEntry -> process.MaybeUpdate -> raft.maybeCommit -> raftLog.maybeCommit -> raft.bcastAppend -> raft.maybeSendAppend -> raft.send -> append(raft.msgs)

处理 `r.msgs` 内容

node.run -> RawNode.newReady -> raftNode.start -> Transport.Send -> peer.send


处理 `r.raftLog.nextEnts()` 内容

node.run -> RawNode.newReady -> raftNode.start -> etcdServer.Run -> etcdServer.applyAll -> etcdServer.applyEntries -> etcdServer.apply -> etcdServer.applyEntryNormal -> authApplierV3.Apply -> needAdminPermission（权限校验）-> applierV3backend.Apply -> authApplierV3.Put -> applierV3backend.Put -> storeTxnWrite.Put -> batchTx.UnsafeSeqPut -> treeIndex.Put -> storeTxnWrite.End -> batchTx.Unlock -> bbolt.Tx.Commit -> 


接收者

streamReader.decodeLoop -> raftNode.Process -> node.Step -> node.run -> stepFollower -> raft.handleAppendEntries -> raftLog.maybeAppend 

事务提交

先对 bucket 进行 rebalance ，然后在 spill

## 并发处理

bbolt 的并发控制：只允许同时存在一个写事务

// Multiple read-only transactions can be used concurrently but only one
// write transaction can be used at a time. Starting multiple write transactions
// will cause the calls to block and be serialized until the current write
// transaction finishes.