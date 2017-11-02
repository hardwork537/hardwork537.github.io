---
layout: post
title: mysql lock
category: web
tags: [web]
description: mysql死锁问题解决
---

## 问题 ##

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;今天调试时发现一个报错,“deadlock found when trying to get lock”,并且是在我没有用的事务的表报的这个错，感觉有点不可思议；但既然产生了，肯定有写得不严谨的地方导致的；于是就跟踪下来。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;架构情况 : 因为我们php框架是基于swoole实现的，所以服务开启后，就常驻进程；连接db的实例保存在一个static变量中，进程内的业务会共用；
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;业务情况 : 同一个库中有a、b、c三张表，操作a时不用事务；b、c是有关联的两张表，所以往两张表中写数据时会用到事务，保持数据的一致性；所以在往b、c表中写数据时，先关闭自动提交来实现事务 $db->autocommit(false);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;实际情况 : 在操作b、c时，把自动提交关闭了，然后操作完毕后，没有开启自动提交；因为db实例是存在static变量中的，所以在操作a时也是关闭自动提交的，但是却没有commit操作

1. duplicate key error引发的死锁

这个场景主要发生在两个以上的事务同时进行唯一键值相同的记录插入操作。

### 表结构 ###

```
CREATE TABLE `aa` (
  `id` int(10) unsigned NOT NULL COMMENT '主键',
  `name` varchar(20) NOT NULL DEFAULT '' COMMENT '姓名',
  `age` int(11) NOT NULL DEFAULT '0' COMMENT '年龄',
  `stage` int(11) NOT NULL DEFAULT '0' COMMENT '关卡数',
  PRIMARY KEY (`id`),
  UNIQUE KEY `udx_name` (`name`),
  KEY `idx_stage` (`stage`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 表数据 ###

```
mysql> select * from aa;
+----+------+-----+-------+
| id | name | age | stage |
+----+------+-----+-------+
|  1 | yst  |  11 |     8 |
|  2 | dxj  |   7 |     4 |
|  3 | lb   |  13 |     7 |
|  4 | zsq  |   5 |     7 |
|  5 | lxr  |  13 |     4 |
+----+------+-----+-------+
```

### 事务执行时序表

| T1(36727)        | T2(36728)          | T3(36729)  |
| ------------- |:-------------:| -----:|
| begin;    | begin; | begin; |
| insert into aa values(6, ‘test’, 12, 3);      |       |    |
|  | insert into aa values(6, ‘test’, 12, 3);     |     |
|  |  | insert into aa values(6, ‘test’, 12, 3); |
| rollback; |  |  |
|  |  | ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction |
|  | Query OK, 1 row affected (13.10 sec) |  |

如果T1未rollback，而是commit的话，T2和T3会报唯一键冲突：ERROR 1062 (23000): Duplicate entry ‘6’ for key ‘PRIMARY’


### 事务锁占用情况 ###

T1 rollback前，各事务锁占用情况：

```
mysql> select * from information_schema.innodb_locks;
+--------------+-------------+-----------+-----------+-------------+------------+------------+-----------+----------+-----------+
| lock_id      | lock_trx_id | lock_mode | lock_type | lock_table  | lock_index | lock_space | lock_page | lock_rec | lock_data |
+--------------+-------------+-----------+-----------+-------------+------------+------------+-----------+----------+-----------+
| 36729:24:3:7 | 36729       | S         | RECORD    | `test`.`aa` | PRIMARY    |         24 |         3 |        7 | 6         |
| 36727:24:3:7 | 36727       | X         | RECORD    | `test`.`aa` | PRIMARY    |         24 |         3 |        7 | 6         |
| 36728:24:3:7 | 36728       | S         | RECORD    | `test`.`aa` | PRIMARY    |         24 |         3 |        7 | 6         |
+--------------+-------------+-----------+-----------+-------------+------------+------------+-----------+----------+-----------+
```

注：mysql有自己的一套规则来决定T2与T3哪个进行回滚，本文不做讨论。

### 死锁成因 ###

事务T1成功插入记录，并获得索引id=6上的排他记录锁(LOCK_X | LOCK_REC_NOT_GAP)。
紧接着事务T2、T3也开始插入记录，请求排他插入意向锁(LOCK_X | LOCK_GAP | LOCK_INSERT_INTENTION)；但由于发生重复唯一键冲突，各自请求的排他记录锁(LOCK_X | LOCK_REC_NOT_GAP)转成共享记录锁(LOCK_S | LOCK_REC_NOT_GAP)。

T1回滚释放索引id=6上的排他记录锁(LOCK_X | LOCK_REC_NOT_GAP)，T2和T3都要请求索引id=6上的排他记录锁(LOCK_X | LOCK_REC_NOT_GAP)。
由于X锁与S锁互斥，T2和T3都等待对方释放S锁。
于是，死锁便产生了。

如果此场景下，只有两个事务T1与T2或者T1与T3，则不会引发如上死锁情况产生。