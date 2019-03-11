---
title: 【分布式系统】数据复制
date: 2020-05-27
draft: true
categories: 
    - What is this
---

# 数据复制

**复制** 在计算机领域，涉及多个冗余节点的数据共享，以提升分布式系统的可靠性、容错性（ fault-tolerance）以及可用性。

用户访问一个复制系统应与访问单一非复制系统有同样的表现，即复制系统保证数据的一致性、持久性。在单个节点访问失败的情况下，复制系统应能进行有效的故障转移（Failover），并且用户无感知。在复制方式上，有两种方式：

- **主动复制**：在复制系统中，将用户写操作在每个节点执行
- **被动复制**：在复制系统中，选择某个节点执行用户写操作，并将操作结果同步到其他节点

当复制系统中，通过单一节点来执行所有请求，这个系统被称之为 master-slave 架构，这种架构在高可用集群中占主导地位。相对的，如果集群中所有节点均可执行请求，并分发新状态，则称之为 multi-master 架构。在多主架构中，响应的并发控制就少不了

Load balancing differs from task replication, since it distributes a load of different computations across machines, and allows a single computation to be dropped in case of failure. Load balancing, however, sometimes uses data replication (especially multi-master replication) internally, to distribute its data among machines.

负载均衡与任务复制

Backup differs from replication in that the saved copy of data remains unchanged for a long period of time.[3] Replicas, on the other hand, undergo frequent updates and quickly lose any historical state. Replication is one of the oldest and most important topics in the overall area of distributed systems.



Data replication and computation replication both require processes to handle incoming events. Processes for data replication are passive and operate only to maintain the stored data, reply to read requests and apply updates. Computation replication is usually performed to provide fault-tolerance, and take over an operation if one component fails. In both cases, the underlying needs are to ensure that the replicas see the same events in equivalent orders, so that they stay in consistent states and any replica can respond to queries.

## 复制模型

### 事务复制

事务复制的原理是先将发布服务器数据库中的初始快照发送到各订阅服务器,然后监控发布服务器数据库中数据发生的变化,捕获个别数据变化的事务并将变化的数据发送到订阅服务器。为了保证变化的数据能准确而及时的发送到订阅服务器上,发布服务器和订阅服务器之间的网络连接必须是可靠且连续的。

基于日志的复制

### 状态机复制

assumes that the replicated process is a deterministic finite automaton and that atomic broadcast of every event is possible. It is based on distributed consensus and has a great deal in common with the transactional replication model. This is sometimes mistakenly used as a synonym of active replication. State machine replication is usually implemented by a replicated log consisting of multiple subsequent rounds of the Paxos algorithm. This was popularized by Google's Chubby system, and is the core behind the open-source Keyspace data store.[4][5]


复制过程是确定性的有限自动机，并且可以对每个事件进行原子广播（全序广播）。

它基于分布式共识，并且与事务复制模型有很多共同点。

有时会误将其用作主动复制的同义词。状态机复制通常由包含多个Paxos算法回合的复制日志来实现。

https://en.wikipedia.org/wiki/Atomic_broadcast

https://en.wikipedia.org/wiki/State_machine_replication

### 文件复制



## 单主


## 多主