---
title: Innodb 锁类型
date: 2020-07-04
draft: true
tags:
    - MySQL
    - 数据库
categories:
    - Innodb
---

# 锁类型

https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html

## 读写锁

Innodb 实现了标准的 **行级锁**，有 **读锁（S锁）** 和 **写锁（X锁）** 两种类型。读锁允许事务对加锁的行进行读取，写锁允许事务对加锁的行进行更新或删除。

如果事务 T1 对行 r 持有读锁，同时事务 T2 向行 r 申请锁：

- 如果 T2 申请的是读锁，则可立即加锁成功，最终 T1，T2 均持有对行 r 的读锁
- 如果 T2 申请的是写锁，则不能立即加锁成功，需等待 T1 的读锁释放后才能加锁

如果事务 T1 对行 r 持有写锁，则无论 T2 请求什么类型的锁，均不能立即加锁成功，T2 需要等待 T1 在行 r 上的锁释放。

## 意向锁

Innodb 支持多粒度的锁，并且允许表锁和行锁共存。比如： `lock tables ... write` 语句会对表加一个写锁。Innodb 通过意向锁来支持多粒度锁。意向锁是表级别的锁，表明事务会对表中行加的锁类型。

- 意向读锁（IS）表明事务希望对表中某些行加读锁
- 意向写锁（IX）表明事务系统对表中某些行加写锁

比如：`select ... lock in share mode` 会设置一个 IS 锁，`select ... fro update` 会设置一个 IX 锁

意向锁加锁规则如下：

1. 在事务对表中某些行获取 S 锁前，事务必须获取该表的 IS 或更强的锁
2. 在事务对表中某些行获取 X 锁前，事务必须获取该表的 IX 锁

表级别的锁兼容性如下所示：


|   |X|IX|S|IS|
|:--|:--|:--|:--|:--|
|X|冲突|冲突|冲突|冲突|
|IX|冲突|兼容|冲突|兼容|
|S|冲突|冲突|兼容|兼容|
|IS|冲突|兼容|兼容|兼容|


当事务请求的锁与行（或表）已拥有的锁冲突时，获取锁失败；当事务请求的锁与行（或表）已拥有的锁兼容时，获取锁成功。获取锁时，如果锁冲突，则事务会等到冲突的锁释放。如果事务请求的锁与行（表）已拥有的锁冲突，并且由于请求的锁可能导致死锁时，会发生错误。


意向锁不会阻塞任何请求，除非是对全表的操作，比如：`lock tables ... write`。加意向锁的主要目的，是要向 MySQL 说明事务 **正在** 或 **将要** 对表中的某一行进行加锁。

<!-- 

`SHOW ENGINE INNODB STATUS` 可以展示当前的意向锁

```sql
TABLE LOCK table `test`.`t` trx id 10080 lock mode IX
``` -->

## 记录锁

记录锁是在索引记录上的锁。比如： `SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;` 会禁止其他事务 插入、更新或删除 `t.c1=10` 的所有行。

记录锁始终是对索引记录加锁，即使表中未定义索引。对于没有索引的表，Innodb 会默认创建一个隐藏的 聚集索引，并用这个聚集索引来加锁。

<!-- 

`SHOW ENGINE INNODB STATUS` 可以展示当前的记录锁

RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t`
trx id 10078 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;

 -->

 ## 间隙锁

 间隙锁是对索引记录间的间隙进行加锁，或者是对首个索引记录之前、末尾索引记录之后进行加锁。例如：`SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE;` 禁止其他事务插入 `t.c1=15` 的记录，不论 `c1` 是否有 `15` 这个值，因为 `10` 到 `20` 之间的间隙已被锁定。

间隙可能跨越单个索引值、多个索引值甚至是空。

间隙锁是性能和并发的平衡，只会在某些事务隔离级别下生效。

当 SQL 语句通过 唯一索引 进行加锁时，不需要使用间隙锁（不包含 加锁索引是联合唯一索引的情况，在这种情况下，间隙锁也会生效）。例如：如果 `id` 字段为唯一索引，那么下面的 SQL 只会对 `id=100` 的记录加记录锁，而不会禁止其他事务插入 `id<100` 的记录。

```sql
SELECT * FROM child WHERE id = 100;
```

如果 `id` 不是索引或者非唯一索引，这条语句都会对这条记录之前的间隙进行加锁。

值得注意的是，间隙锁允许不同事务的持有有冲突的间隙锁。比如，T1 在一个 gap 上持有一个 间隙读锁（gap S-lock），同时，T2 在相同的 gap 上持有一个 间隙写锁（gap X-lock）。这种冲突的间隙锁可以被允许的原因是，如果一条记录从索引上被清除，不同事务持有的间隙锁必须合并。

Innodb 中的间隙锁是 “纯禁止性” 的，意味着其唯一的目的就是防止别的事务在间隙中插入数据。间隙锁是可以共存的。一个事务的间隙锁不禁止其他事务在同一个间隙上的间隙锁，不论这个间隙锁是读锁还是写锁。

当使用 RC 事务隔离级别 或 设置 `innodb_locks_unsafe_for_binlog ` 变量时，搜索和索引扫描都不会使用间隙锁，仅用于外键的一致性检查和重复键的检查。并且在这种情况下，Record locks for nonmatching rows are released after MySQL has evaluated the WHERE condition. For UPDATE statements, InnoDB does a “semi-consistent” read, such that it returns the latest committed version to MySQL so that MySQL can determine whether the row matches the WHERE condition of the UPDATE.



## NK 锁

NK 锁是 索引记录上的记录锁和索引记录之前的间隙锁组成的。

当 Innodb 在索引记录上进行搜索或扫描时，会将其遇到的索引记录加上 S 或 X 行级锁。因此 行级锁和记录锁是等价的。 索引上的 NK 锁同时会影响索引记录之前的间隙。NK 锁是一个记录锁 加上锁定索引之前间隙的间隙锁。如果一个事务设置了 X 或 S 的 NK 锁在记录 R 的索引上，则其他事务不能在记录 R 之前的间隙，插入一个新的索引。

设想索引包含 10，11，13和20，该索引的可能的下一键锁定涵盖以下间隔，其中，圆括号表示排除区间端点，方括号表示包括端点：

```text
(negative infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive infinity)
```

最后一个间隙，NK 锁锁定的是索引中最大值之前的间隙，最大的索引在 Innodb 中用 "supremum" 伪节点表示，其拥有比任何索引都大的索引值。由于其是伪节点，所以 NK 锁在这里实际上是锁定的到最大值的间隙。


默认情况下，innodb 是 RR 隔离级别，innodb 会使用 NK 锁来搜索和扫描索引，来防止幻影行（phantom rows）。


```sql
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t`
trx id 10080 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

## 插入意向锁

插入意向锁是一种 insert 语句在插入行之前加的间隙锁。这个锁表示事务想插入一个新行，其他事务想插入同一个间隙的 **不同位置** 是不需要等待的。设想，索引记录包含 4，7，两个事务分别想插入 5，6两条记录，均会在 4，7 之间加 “插入意向锁” 来防止其他事务插入相同位置，但是当插入行不冲突是不会阻塞事务。


以下示例表明，事务在获取到插入行的排他锁之前，会先获取插入意向锁，这里有两个客户端 A，B。

A 创建包含两个索引记录（90，120）的表，并且会开启一个事务来获取 id=100 的记录（X）锁，以及 id>100 的间隙（X）锁。

```sql
mysql> CREATE TABLE child (id int(11) NOT NULL, PRIMARY KEY(id)) ENGINE=InnoDB;
mysql> INSERT INTO child (id) values (90),(102);

mysql> START TRANSACTION;
mysql> SELECT * FROM child WHERE id > 100 FOR UPDATE;
+-----+
| id  |
+-----+
| 102 |
+-----+
```

B 开启一个事务插入一条记录到间隙中，事务在等待获得排他锁的同时获取插入意图锁。

```sql
mysql> START TRANSACTION;
mysql> INSERT INTO child (id) VALUES (101);
```

```sql
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421575262207632, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 1820, ACTIVE 24 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 17, OS thread handle 140099956893440, query id 236 10.200.251.80 root update
INSERT INTO child (id) VALUES (101)
------- TRX HAS BEEN WAITING 24 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 25 page no 3 n bits 72 index PRIMARY of table `shop`.`child` trx id 1820 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 80000066; asc    f;;
 1: len 6; hex 000000000716; asc       ;;
 2: len 7; hex b400000128011c; asc     (  ;;
```

locks gap before rec (record)

## 自增锁

自增锁是一个特别的表级锁，当事务插入新行到表中并且带有 “自增” 行时获取。



# 不同语句的锁设置

https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html

锁定读（locking read），更新或删除 通常来说都会对 **所有扫描过** 的索引记录加锁。对索引记录的加锁与索引记录是否被 where 条件排除无关。innodb 不会记录详细的 where 条件，只知道需要被扫描的索引。锁通常是 NK 锁，它还会阻止在记录之前插入“间隙”。间隙锁可以被禁用，同时 NK 锁也会被禁用。


如果在搜索是会使用二级索引，并且索引上会加上 X 锁，则 Innodb 会对其相应的聚集索引的索引记录加锁。


如果没有合适的索引进行扫描，则 MySQL 必须扫描整个表，每一行都会被锁定，并且不允许有其他事务进行插入。为了减少扫描的行数，创建一个好的索引是十分必要的。

## SELECT

`select ... from` 是一致性读，从数据库中读取快照，并且不会设置锁，除非事务隔离级别为 可序列化。对于可序列化隔离级别，`select ... from` 会对索引上扫描到的记录加共享的 NK 锁，但对于使用唯一索引搜索行来锁定的语句，仅需要索引记录锁定


对 `SELECT ... FOR UPDATE` 或者 `SELECT ... LOCK IN SHARE MODE` 来说，会对所有扫描到的索引记录加锁，但是会释放那些不符合 where 条件要求的索引记录。然而在某些情况下，这些索引记录可能不会被立即锁定，因为结果集和原始数据的关系可能会丢失。比如，在 `UNION` 操作中，扫描（加锁）到的行可能会被插入临时表中，在最终判定它们是否是结果集之前。在这种情况下，临时表与原始表中的关系丢失，而原始表中的行直到最终结果集计算出来之前都不会被锁定。

`SELECT ... LOCK IN SHARE MODE` 会在其搜索的所有索引记录上，设置共享的 NK 锁。然而，当搜索使用的是唯一索引时，其只需要 记录锁。

`SELECT ... FOR UPDATE` 会在其搜索的所有索引记录上，设置排他的 NK 锁。然而，当搜索使用的是唯一索引时，其只需要 记录锁。

`SELECT ... FOR UPDATE` 会阻塞其他事务的 `SELECT ... LOCK IN SHARE MODE` 或在某些隔离级别下的读取。一致性读会忽略在其读取视图上的任何锁。

## UPDATE

`UPDATE ... WHERE ...` 会在其搜索的所有索引记录上，设置排他的 NK 锁。当搜索使用的是唯一索引时，其只需要 记录锁。

当 `UPDATE` 更新聚集索引记录时，会隐式的对受 **影响的辅助索引进行加锁**。 在插入新的二级索引记录之前执行重复检查扫描时，以及在插入新的二级索引记录时， `UPDATE` 操作还会在受影响的二级索引记录上获得共享锁。

Client A

```sql
mysql> begin;
Query OK, 0 rows affected (0.03 sec)

mysql> update order_product_info set c1=19, c2=50 where id=188\G
Query OK, 1 row affected (0.02 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

Client B

```sql
mysql> begin;
Query OK, 0 rows affected (0.01 sec)

mysql> insert order_product_info (c1,c2,c3) values (19,50,2182313123)\G
```

```sql
mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS\G
*************************** 1. row ***************************
    lock_id: 19814:26:4:111
lock_trx_id: 19814
  lock_mode: S
  lock_type: RECORD
 lock_table: `shop`.`order_product_info`
 lock_index: uniq_12
 lock_space: 26
  lock_page: 4
   lock_rec: 111
  lock_data: 19, 50
*************************** 2. row ***************************
    lock_id: 19813:26:4:111
lock_trx_id: 19813
  lock_mode: X
  lock_type: RECORD
 lock_table: `shop`.`order_product_info`
 lock_index: uniq_12
 lock_space: 26
  lock_page: 4
   lock_rec: 111
  lock_data: 19, 50
2 rows in set, 1 warning (0.01 sec)
```

## DELETE

`DELETE FROM ... WHERE ...` 会在其搜索的所有索引记录上，设置排他的 NK 锁。当搜索使用的是唯一索引时，其只需要 记录锁。

Client A

```sql
mysql> begin;
Query OK, 0 rows affected (0.01 sec)

mysql> delete from order_product_info where c1=19 and c2=19\G
Query OK, 1 row affected (0.02 sec)
```

Client B

```sql
mysql> begin;
Query OK, 0 rows affected (0.01 sec)

mysql> update order_product_info set c1=19, c2=50 where id=188\G
```

```sql
mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS\G
*************************** 1. row ***************************
    lock_id: 19817:26:3:111
lock_trx_id: 19817
  lock_mode: X
  lock_type: RECORD
 lock_table: `shop`.`order_product_info`
 lock_index: PRIMARY
 lock_space: 26
  lock_page: 3
   lock_rec: 111
  lock_data: 188
*************************** 2. row ***************************
    lock_id: 19816:26:3:111
lock_trx_id: 19816
  lock_mode: X
  lock_type: RECORD
 lock_table: `shop`.`order_product_info`
 lock_index: PRIMARY
 lock_space: 26
  lock_page: 3
   lock_rec: 111
  lock_data: 188
2 rows in set, 1 warning (0.02 sec)
```

## INSERT

`INSERT` 会在新插入的行上（主键）加 排他锁，这个锁是记录锁，而非 NK 锁（因此这里不会有间隙被锁定），不会阻止其他事务在新插入行之前插入其他行

在插入行之前，会加一个 插入意向锁的 间隙锁。这个锁表示事务想插入一个新行，其他事务想插入同一个间隙的 **不同位置** 是不需要等待的。设想，索引记录包含 4，7，两个事务分别想插入 5，6两条记录，在获取插入行的排他锁之前，均会在 4，7 之间加 “插入意向锁”，但是由于插入行不冲突，所以是不会阻塞事务的。

Client A

```sql
mysql> begin;
Query OK, 0 rows affected (0.01 sec)

mysql> insert into order_product_info (id,c1,c2,c3) values(194,20,21,123123)\G
Query OK, 1 row affected (0.03 sec)

mysql> SELECT TRX_ID FROM INFORMATION_SCHEMA.INNODB_TRX WHERE TRX_MYSQL_THREAD_ID = CONNECTION_ID()\G
*************************** 1. row ***************************
TRX_ID: 19819
1 row in set (0.01 sec)
```

Client B 插入不同行

```sql
mysql> begin;
Query OK, 0 rows affected (0.01 sec)

mysql> insert into order_product_info (id,c1,c2,c3) values(195,21,22,1123123)\G
Query OK, 1 row affected (0.01 sec)
```

当发生 “重复键” 时，会对重复的索引记录加 共享锁。

Client A

```sql
mysql> begin;
Query OK, 0 rows affected (0.01 sec)

mysql> insert into order_product_info (id,c1,c2,c3) values(194,20,21,123123)\G
Query OK, 1 row affected (0.03 sec)

mysql> SELECT TRX_ID FROM INFORMATION_SCHEMA.INNODB_TRX WHERE TRX_MYSQL_THREAD_ID = CONNECTION_ID()\G
*************************** 1. row ***************************
TRX_ID: 19819
1 row in set (0.01 sec)
```


Client B 插入相同行

```sql
mysql> begin;
Query OK, 0 rows affected (0.02 sec)

mysql> insert into order_product_info (id,c1,c2,c3) values(194,21,22,1123123)\G

mysql> SELECT TRX_ID FROM INFORMATION_SCHEMA.INNODB_TRX WHERE TRX_MYSQL_THREAD_ID = CONNECTION_ID()\G
*************************** 1. row ***************************
TRX_ID: 19821
1 row in set (0.02 sec)
```

```sql
mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS\G
*************************** 1. row ***************************
    lock_id: 19821:26:3:116
lock_trx_id: 19821
  lock_mode: S
  lock_type: RECORD
 lock_table: `shop`.`order_product_info`
 lock_index: PRIMARY
 lock_space: 26
  lock_page: 3
   lock_rec: 116
  lock_data: 194
*************************** 2. row ***************************
    lock_id: 19819:26:3:116
lock_trx_id: 19819
  lock_mode: X
  lock_type: RECORD
 lock_table: `shop`.`order_product_info`
 lock_index: PRIMARY
 lock_space: 26
  lock_page: 3
   lock_rec: 116
  lock_data: 194
2 rows in set, 1 warning (0.02 sec)
```

当有多个事务想插入同一行时，其中一个事务获取了 X 锁，然后事务把这行删除，则会导致死锁。现有表如下结构：

```sql
CREATE TABLE t1 (i INT, PRIMARY KEY (i)) ENGINE = InnoDB;
```

考虑以下情况：

Session 1:

```sql
START TRANSACTION;
INSERT INTO t1 VALUES(1);
```

Session 2:

```sql
START TRANSACTION;
INSERT INTO t1 VALUES(1);
```

Session 3:

```sql
START TRANSACTION;
INSERT INTO t1 VALUES(1);
```

Session 1:

```sql
ROLLBACK;
```

事务 1 获取了该行的 X 锁，导致 T2、T3 发生唯一键冲突，并同时请求该键的 S 锁。当 T1 回滚，其释放持有的 X 锁，同时 T2、T3 加 S 锁成功。在这个时候，T2、T3 就死锁，任何一个事务都不能获取到 X 锁，因为对方都持有 S 锁。

还有一种情况，也比较类似，假如当前表中已有一行数据 1，

Session 1:

```sql
START TRANSACTION;
DELETE FROM t1 WHERE i = 1;
```

Session 2:

```
START TRANSACTION;
INSERT INTO t1 VALUES(1);
```

Session 3:

```
START TRANSACTION;
INSERT INTO t1 VALUES(1);
```

Session 1:

```
COMMIT;
```

T1 获取该行的 X 锁，T2、T3 发生唯一键冲突，并同时请求该键的 S 锁。当 T1 提交时，其释放持有的 X 锁，同时 T2、T3 加 S 锁成功。在这个时候，T2、T3 就死锁，任何一个事务都不能获取到 X 锁，因为对方都持有 S 锁。


### INSERT ... ON DUPLICATE KEY UPDATE

与简单的 `INSERT` 语句不同，当发生唯一键冲突时会（主键）加 X 锁而非 S 锁。当主键冲突时，会加 X 的（主键）记录锁；

Clien A

```sql
mysql> begin;
Query OK, 0 rows affected (0.01 sec)

mysql> insert into order_product_info (id,c1,c2,c3) values(194,20,21,123123)\G
Query OK, 1 row affected (0.02 sec)

mysql> SELECT TRX_ID FROM INFORMATION_SCHEMA.INNODB_TRX WHERE TRX_MYSQL_THREAD_ID = CONNECTION_ID()\G
*************************** 1. row ***************************
TRX_ID: 19822
1 row in set (0.02 sec)
```

Client B

```sql
mysql> begin;
Query OK, 0 rows affected (0.01 sec)

mysql> insert into order_product_info (id,c1,c2,c3) values(194,20,21,777777) on duplicate key update c3=777777\G
```

```sql
mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS\G
*************************** 1. row ***************************
    lock_id: 19823:26:3:116
lock_trx_id: 19823
  lock_mode: X
  lock_type: RECORD
 lock_table: `shop`.`order_product_info`
 lock_index: PRIMARY
 lock_space: 26
  lock_page: 3
   lock_rec: 116
  lock_data: 194
*************************** 2. row ***************************
    lock_id: 19822:26:3:116
lock_trx_id: 19822
  lock_mode: X
  lock_type: RECORD
 lock_table: `shop`.`order_product_info`
 lock_index: PRIMARY
 lock_space: 26
  lock_page: 3
   lock_rec: 116
  lock_data: 194
2 rows in set, 1 warning (0.03 sec)
```

当唯一键冲突时会加（主键） X 的 NK 锁。

Clien A

```sql
mysql>begin;
Query OK, 0 rows affected (0.02 sec)

mysql> insert into order_product_info (id,c1,c2,c3) values(200,19,19,777777) on duplicate key update c3=777777\G
Query OK, 2 rows affected (0.14 sec)
```

Client B

```sql
mysql> begin;
Query OK, 0 rows affected (0.01 sec)

mysql> insert into order_product_info (id,c1,c2,c3) values(201,19,17,777777)\G
```

```sql
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421575262207632, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 19864, ACTIVE 494 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 1169, OS thread handle 140097470576384, query id 47241 10.254.249.227 root update
insert into order_product_info (id,c1,c2,c3) values(201,19,17,777777)
------- TRX HAS BEEN WAITING 5 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 26 page no 3 n bits 184 index PRIMARY of table `shop`.`order_product_info` trx id 19864 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;
```

### REPLACE

当无唯一键冲突时，其行为与 `INSERT` 一致；当发生唯一键冲突时，则会加 NK 锁。