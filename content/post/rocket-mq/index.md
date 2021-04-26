---
title: RocketMQ
draft: true
date: 2021-04-21
categories: 
    - 中间件
---



## 关键特性

- 顺序消息：
- 事务/半事务消息
- 定时/延迟消息：
- 单向消息：
- 消息标签：
- At Least Once：
- 消息回溯：
- 消息重试：
- 消息重投：
- 死信队列：
- 流量控制：


## 性能对比

下面，性能测试对比下 Kafka 和 RocketMQ 在 topic 增加时的表现。测试配置：每个 topic 有 8 个分区，每个 topic 都有一个订阅者，并且 topic 数量逐步增加。

![](assists/rocketmq_vs_kafka.png)

通过上面性能测试数据，我们能看到 RocketMQ 和 Kafka 在不同的 topic 量级下，都能平衡 Send 和 Receive 速率，没有大量的消息积压。但也存在以下区别：
- 在 topic 从 68 到 256 的增长过程中， Kafka 性能劣化 98%
- 在 topic 从 68 到 256 的增长过程中， RocketMQ 性能仅劣化 16%

Why the huge difference? The difference is attributable to the fact that every topic and partition of Kafka correspond to one physical file. When the number of topics increases, the policy of deconcentrated storage of messages to disks will lead to disk IO competition to cause performance bottlenecks. In contrast, all the messages in Apache RocketMQ™are stored in the same physical file. The number of topics and partitions is just a logic division for Apache RocketMQ™. So the increasing number of topics won’t generate a huge impact on the Apache RocketMQ™ performance.

## 核心原理


### 角色


### 消息发送



### 消息处理



### 可靠性



## 参考文档

https://github.com/apache/rocketmq/blob/master/docs/cn/features.md
https://www.aliyun.com/product/rocketmq
https://github.com/apache/rocketmq-client-go/blob/master/docs/Introduction.md
https://help.aliyun.com/document_detail/102777.html?spm=a2c4g.11186623.6.601.73a86242U8VTgj
