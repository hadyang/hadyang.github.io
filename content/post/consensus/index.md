---
title: 什么是共识算法？
date: 2020-05-29
draft: false
categories: 
    - What it is
---

## 全序广播

首先介绍下全序关系和偏序关系，这两个均为集合论中的概念。全序关系是在集合 $X$ 上的满足反对性、传递性和完全性的二元关系。对于 $X$ 中任意两个元素 $a$, $b$ 和 $c$：

- 反对称性：若 $a \leq b$ 且 $b \leq a$ 则 $a = b$
- 传递性：若 $a \leq b$ 且 $b \leq c$ 则 $a \leq c$
- 完全性：$a \leq b$ 且 $b \leq a$

全序关系中的完全性可以做如下描述：集合中的任何一对元素都是 **可相互比较** 的。偏序关系的概念比较复杂，可以简单理解为：不满足 **完全性** 的全序关系，即部分有序。

在单机系统上，CPU 本身保证操作的全序关系，但是在分布式系统中，如何让所有节点就全序关系达成一致就面临很大的挑战。这里的挑战也是本身分布式系统所面临的，包括：网络延迟、网络丢包、不统一的时钟、不稳定的节点。

**全序广播** 通常指的是节点之间交换消息的某种协议，其满足以下两个属性：

- **可靠发送**：没有消息丢失，如果消息发送到某个节点，则它一定要到达所有节点
- **严格有序**：消息总是以相同顺序发送给每个节点

全序广播需要保证上述的两个条件，即使在网络故障的情况下，算法必须进行重试，直到网络最终恢复。全序广播又被称为 **原子广播**，但这里的 “原子” 和我们常说的 ACID 中的 “原子性” 没有关系，而是和 线性化存储 有关。

如果有了解过 MySQL 的主从复制，就比较能理解全序广播的概念。MySQL 的主从复制，就是一种全序广播的实现。类似的还有 ZAB 协议、Raft 协议，都是全序广播的一种实现，只不过 ZAB 和 Raft 是属于 **可容错的全序广播**。

## 共识

共识通俗地讲就是多个节点（个人）对某项提议达成“一致”，这里的“一致”与一致性没有关联。共识问题形式化描述如下：一个或多个节点可以提议某些值，由共识算法来决定最终值。**共识算法**（Consensus Algorithm）就是为分布式系统，在不可靠的网络环境、不可靠的节点下，提供达成共识的方法。共识算法必须满足以下性质：

- **协商一致性**：所有节点都接收相同的提议
- **诚实性**：所有节点不能反悔，即对一项提议进行多次投票
- **合法性**：如果决定了一个值，其一定是系统中某个节点提议的
- **可终止性**：节点如果不崩溃则最终一定能达成协议

可终止性引入了容错的思想，它强调一个共识算法不能原地空转，换句话说，它必须取得实质性进展。即使节点出现故障，其他节点也必须作出决定。

这里的可终止性也是有条件的，即 **一定数量** 的节点崩溃。如果所有节点都崩溃，那没有一种算法能最终作出决定。在实现过程中，任何的共识算法，都需要至少大部分节点正常运行，才能保证可终止性。而这个大多数就是 **quorum**，即集群中节点数量过半。

> 当集群中正常运行节点少于 quorum 时，集群处于无响应状态

## 共识与全序广播

常见的共识算法还有 Paxos、ZAB 以及 Raft，其中，ZAB 和 Raft 协议类似，属于 **选主** + **全序广播** 的形式。选主，即对整个集群的主节点达成共识，然后在进行全序广播。在某种程度上，全序广播属于持续多轮的共识。而 Paxos 协议则是对所有数据进行共识，在一定程度上导致 Paxos 算法的复杂性。 


在上面提到的全序广播中，都有一个共性即“一主多从”，由单个主节点接收写请求，并将请求复制到其他节点。但是如果主节点失效，对于全序广播来说，这时整个系统都不可用，只能由人工介入，手动再指定一个主节点。如果算法能在节点失效时，自动选举和故障切换，这样就构成 **可容错的全序广播**。

可以说全序广播是实现共识算法的一种手段，在全序广播上增加对节点失效的处理，就构成 **可容错的全序广播**, 即一种共识算法


## 拜占庭将军问题

在共识算法中，经常会提到一个 **拜占庭将军问题**。在拜占庭将军问题中，分布式系统中的节点会发出错误的消息，即存在 **欺骗**，这种欺骗可能是由于程序Bug、数据损坏等等。拜占庭将军问题中，节点是不可信的。

上面提到的 Paxos 、 ZAB 、Raft 均不是拜占庭容错算法。之前热度很高的区块链技术，通过 **工作量证明算法**（PoW）来实现了拜占庭容错（解决拜占庭将军问题）。

## 共识与一致性

分布式一致性关注分布式环境下，由于网络延迟和故障的影响，同一数据副本在多个节点的状态是否一致。而共识算法则是关注同一数据在多个节点是否能达成共识，达成共识并不一定表示当前本节点该数据状态与其他节点的状态一致。

举个例子来解释，对于 ZK 来说，当你请求数据写入并返回时，它已就某个值达成共识，但在 Follwer 节点你还是可能读取到旧值。这也是 ZK 写入原子一致性、读取顺序一致性的表现。

> ZooKeeper does not guarantee that at every instance in time, two different clients will have identical views of ZooKeeper data. Due to factors like network delays, one client may perform an update before another client gets notified of the change. Consider the scenario of two clients, A and B. **If client A sets the value of a znode /a from 0 to 1, then tells client B to read /a, client B may read the old value of 0, depending on which server it is connected to**. If it is important that Client A and Client B read the same value, Client B should should call the sync() method from the ZooKeeper API method before it performs its read.


> So, ZooKeeper by itself doesn't guarantee that changes occur synchronously across all servers, but ZooKeeper primitives can be used to construct higher level functions that provide useful client synchronization. (For more information, see the ZooKeeper Recipes. [tbd:..]).

虽然我极力避免在本文提到一致性相关内容，因为这又是一个复杂的问题。但是这里还是要强调以下共识算法与分布式一致性 **没有** 必然的联系。


## 共识算法的局限

共识算法为不确定的分布式系统带来了明确的安全属性（写入一致性、完整性和有效性），同时它还支持一定程度的容错。当然，共识算法也不是银弹，不能解决所有问题，并且其也存在局限性。

1. **共识算法需要大多数节点存活才能正常运行**，这意味着至少三个节点才能容忍一个节点的失效
2. **多数共识算法假定一组固定参与投票的节点集合**，这意味着不能动态增加或删除节点
3. 共识算法通常采用 **超时机制** 来检测节点是否可用，在网络延迟不确定的环境下，会导致节点错误的认为主节点失效，频繁的进行选举会导致系统不可用时间变成。

## 参考文档

- [ZooKeeper Programmer's Guide](https://zookeeper.apache.org/doc/r3.1.2/zookeeperProgrammers.html#ch_zkGuarantees)