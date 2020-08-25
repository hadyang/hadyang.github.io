---
title: Select...For Update 锁范围
date: 2020-07-04
draft: true
tags:
    - Database
categories:
    - Innodb
---


```sql
CREATE TABLE `t1` (
  `id` int(11) NOT NULL,
  `c1` char(11) DEFAULT NULL,
  `c2` char(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `u_idx_c1` (`c1`),
  KEY `idx_c2` (`c2`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=REDUNDANT;

CREATE TABLE `t2` (
  `id` int(11) NOT NULL,
  `c1` int(11) DEFAULT NULL,
  `c2` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `u_idx_c1` (`c1`),
  KEY `idx_c2` (`c2`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=REDUNDANT;
```

插入数据集

```go
func InsertData() {
	db, err := sql.Open("mysql", "root:123456@tcp(10.227.10.8:33061)/test")
	if err != nil {
		log.Println(err)
		return
	}

	for i := 10; i < 100; i++ {
		if i%3 == 0 {
			continue
		}
		//开启事务
		tx, err := db.Begin()
		if err != nil {
			log.Println("tx fail")
			continue
		}
		//准备sql语句
		stmt, err := tx.Prepare("insert into t1(id,c1,c2) values(?,?,?)")
		if err != nil {
			log.Println("Prepare fail")
			continue
		}
		//将参数传递到sql语句中并且执行
		res, err := stmt.Exec(i, i, i)
		if err != nil {
			log.Printf("Exec fail,err:%v\n", err)
			continue
		}
		//将事务提交
		err = tx.Commit()
		if err != nil {
			log.Printf("err:%v", err)
			continue
		}
		//获得上一个插入自增的id
		log.Println(res.RowsAffected())
	}

}
```

```sql
10	10	10
11	11	11
13	13	13
14	14	14
16	16	16
17	17	17
19	19	19
20	20	20
```


## 唯一索引

```sql
//session 1
//case 1 查询不加引号，阻塞插入
mysql> begin;
Query OK, 0 rows affected (0.02 sec)

mysql> select * from t1 where c1=16 for update;
+----+------+------+
| id | c1   | c2   |
+----+------+------+
| 16 | 16   | 16   |
+----+------+------+
1 row in set (0.02 sec)

mysql> commit;
Query OK, 0 rows affected (0.01 sec)

//case 2 查询加引号，不阻塞插入
mysql> begin;
Query OK, 0 rows affected (0.02 sec)

mysql> select * from t1 where c1='16' for update;
+----+------+------+
| id | c1   | c2   |
+----+------+------+
| 16 | 16   | 16   |
+----+------+------+
1 row in set (0.02 sec)

mysql> 
```

```sql
//session 2
case 1 查询不加引号，阻塞插入
mysql> begin;
Query OK, 0 rows affected (0.02 sec)

mysql> insert into t1(id,c1,c2) values(15,15,15);
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> insert into t1(id,c1,c2) values(1,1,1);
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted

//case 2 查询加引号，不阻塞插入
mysql> insert into t1(id,c1,c2) values(1,1,1);
Query OK, 1 row affected (0.01 sec)

mysql> insert into t1(id,c1,c2) values(15,'15','15');
Query OK, 1 row affected (0.02 sec)

```


```sql
mysql> explain select * from t1 where c1=16 for update\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t1
   partitions: NULL
         type: ALL
possible_keys: u_idx_c1
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 64
     filtered: 10.00
        Extra: Using where
1 row in set, 3 warnings (0.02 sec)

mysql> explain select * from t1 where c1='16' for update\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t1
   partitions: NULL
         type: const
possible_keys: u_idx_c1
          key: u_idx_c1
      key_len: 34
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.02 sec)
```

## 非唯一索引


```sql
// session 1
mysql> select * from t1 where c2='16' for update;
+----+------+------+
| id | c1   | c2   |
+----+------+------+
| 16 | 16   | 16   |
+----+------+------+
1 row in set (0.01 sec)
```


```sql
//session 2
mysql> begin;
Query OK, 0 rows affected (0.01 sec)

mysql> insert into t1(id,c1,c2) values(1,1,1);
Query OK, 1 row affected (0.02 sec)

mysql> insert into t1(id,c1,c2) values(15,'15','14');
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql> insert into t1(id,c1,c2) values(15,'15','13');
Query OK, 1 row affected (0.02 sec)

mysql> insert into t1(id,c1,c2) values(15,'15','15');
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

mysql> insert into t1(id,c1,c2) values(15,'15','16');
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

mysql> insert into t1(id,c1,c2) values(15,'15','17');
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

mysql> insert into t1(id,c1,c2) values(15,'15','18');
Query OK, 1 row affected (0.01 sec)
```


```sql
mysql> explain select * from t1 where c2='16' for update\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t1
   partitions: NULL
         type: ref
possible_keys: idx_c2
          key: idx_c2
      key_len: 34
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.01 sec)

mysql> explain select * from t1 where c2=16 for update\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t1
   partitions: NULL
         type: ALL
possible_keys: idx_c2
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 66
     filtered: 10.00
        Extra: Using where
1 row in set, 3 warnings (0.02 sec)
```