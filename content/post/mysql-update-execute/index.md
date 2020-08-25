---
title: MySQL 更新语句执行过程
draft: true
date: 2020-10-31
categories: 
    - 
---

同步1（原子性、持久性）
1. 执行更新语句
   1. 待更新数据查询
      1. 从 Buffer Pool 中搜索对应的数据
      2. 从磁盘中通过索引查询符合条件的 Page
      3. 将 Page 加载到 Buffer Pool
   2. 修改 Buffer Pool 中的数据页
      1. 修改聚集索引数据
      2. 修改二级索引数据（Change Buffer）
      3. 写 Undo Log
         1. 写 Redo Log Buffer
2. 事务提交
   1. Redo Log Prepare
   2. MySQL 写 Bin Log
   3. Redo Log commit
   4. Flush Redo Log Buffer



## TODO

1. 为啥聚集索引和二级索引更新方式不一样？

聚集索引也不一定是 in-place update，二级索引一定是 non-in-place update

聚集索引更新方式判断

- 当更新了聚集索引中的主键时 -> non in-place update
- 未更新主键
  - 更新字段的长度未变更 -> in-palce update
  - 更新字段的长度有变更 -> optimistic update
    - 更新字段长度无变化&更新字段未使用页外存储 -> in-place update

2. 二级索引怎么清理的？
   1. Undo Log 会记录对应的主键信息，可根据是否标记删除进行 Purge



https://medium.com/system-design-blog/what-is-caching-1492abb92143

https://blog.csdn.net/zwleagle/article/details/39010477

https://developer.aliyun.com/article/40969

