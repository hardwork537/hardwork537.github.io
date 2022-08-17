---
layout: post
title: MySQL 的 auto_increment 详解
category: [mysql]
tags: [AUTO_INCREMENT]
description: AUTO_INCREMENT用于为表中的列设置一个自增序列，Mysql提供了一系列的锁机制来保证它的性能跟可靠性，通过这些锁机制，我们可以让它变得很高效

---

## 基本特性

MySQL的中 auto_increment 类型的属性主要用于为一个表中记录自动生成 ID。

1. 当插入记录时，如果为 auto_increment 数据列明确指定了一个数值，则会出现两种情况
- 如果插入的值与已有编号重复，则会出现报错异常，因为 auto_increment 数据列的值必须是唯一的
- 如果插入的值大于已有编号，则会把该插入到数据列中，并使在下一个编号将从这个新值开始递增。也就是说，会跳过一些编号

2. 如果自增序列的最大值被删除了，则在插入新记录时，该值被重用
3. 如果使用 update 命令更新自增列，列值与已有的值重复，则会出错。如果大于已有值，则下一个编号从该值开始递增
4. 插入记录的时候，sql不带id的值或者id的值是0、null的话，id都会自增的
5. 自增值的生成后是不能回滚的，所以自增值生成后，事务回滚了，那么那些已经生成的自增值就丢失了，从而使自增列的数据出现空隙

在Mysql5.7以及更早之前，自增序列的计数器(auto-increment counter)是保存在内存中的。auto-increment counter在每次Mysql重新启动后通过类似下面的这种语句进行初始化

而从Mysql8开始，auto-increment counter被存储在了redo log中，并且每次变化都会刷新到redo log中。另外，我们可以通过ALTER TABLE ... AUTO_INCREMENT = N 来主动修改
auto-increment counter。

不使用uuid 的原因：一般情况下，MySQL数据库对于主键，推荐使用自增ID，因为在MySQL的 InnoDB 存储引擎中，主键索引是聚簇索引，主键索引的B+树的叶子节点按照顺序存储了主键值及数据，如果主键索引是自增ID，只需要按顺序往后排列即可，如果是UUID，ID是随机生成的，在数据插入时会造成大量的数据移动，产生大量的内存碎片，造成插入性能的下降。除此之外，UUID占用空间较大，建立的索引越多，造成的影响越大


## 锁模式

### insert 语句分类

在 MySQL 5.1.22 前，MySQL 的 "insert-like" 会在执行语句的过程中使用一个表级的自增锁（AUTO-INC Lock）将整个表锁住，直到整个语句结束（而不是事务结束）。在此期间会阻塞其他的 insert-like、update 等语句，所以推荐使用程序将这些语句分成多条语句，一一插入，减少单一时间的锁表时间

insert-like 语句，就是指所有可以向表中增加行的语句，可以分成三种类型

- simple inserts
	- 能预先知道插入行数的语句。比如说单行插入（不包括INSERT ... ON DUPLICATE KEY UPDATE），不带子句的多行插入（自增列不赋值或全赋值）
	
- bulk inserts
	- 不能能预先知道插入行数的语句。比如INSERT ... SELECT, REPLACE ... SELECT 。这种模式下，InnoDB 会在处理时为每一行的自增列一次分配一个自增值
	- 申请批量id的策略是对于同一条sql中的申请id，第一次分配一个，如果第一次分配后这个sql还会来申请，就会给2个，依次类推，下一次总是上一次的2倍
	
- mixed-mode inserts
	- 不确定是否需要分配 auto_increment id，如 insert into t (id,name) values (1,'a'),(null,'b'),(5,'c') 以及 insert… 
	- on duplicate key update。对于后者，它的最坏情况实际上是在 insert 语句后面又跟了一个 update，其中 auto_increment 列的分配值不一定会在 update 阶段使用

在 MySQL 5.1.22 之后，MySQL 进行了改进，引入了参数 innodb_autoinc_lock_mode，通过这个参数控制 MySQL 的锁表逻辑。

### innodb_autoinc_lock_mode

锁模式在启动时通过innodb_autoinc_lock_mode这个变量来配置，它有3个值：0, 1, 2。分别对应traditional(传统)、consecutive(连续)、interleaved(交错)3种模式。

在Mysql5.6~5.7里，这个配置项的默认值是1，从Mysql8开始，它的默认值2。这个一方面是因为模式2的性能更好，另一方面是因为从Mysql8开始，默认的主从复制的方式由statement-based 改为了row based。row based 能保证innodb_autoinc_lock_mode=2时主从复制时的数据不会出现不一致的问题

- traditional
	- 在这种模式下，Mysql所有的Insert-like操作都会设置一个表级别的AUTO-INC 锁，这个锁会在单条语句执行完毕时释放
	- 也就是说，如果有多个事务同时对同一张表执行Insert-like 操作，那么，即使它们没有操作同一条记录，也会串行执行。所以，它的性能相对另外两种模式来说会比较糟糕
	
- consecutive
	- 在这种模式下，对于Simple inserts语句，Mysql会在语句执行的初始阶段将一条语句需要的所有自增值会一次性分配出来，并且通过设置一个互斥量来保证自增序列的一致性，一旦自增值生成完毕，这个互斥量会立即释放，不需要等到语句执行结束
	- 所以，在consecutive模式，多事务并发执行Simple inserts这类语句时， 相对traditional模式，性能会有比较大的提升
	- 由于一开始就为语句分配了所有需要的自增值，那么对于像Mixed-mode inserts这类语句，就有可能多分配了一些值给它，从而导致自增序列出现"空隙"。而traditional模式因为每一次只会为一条记录分配自增值，所有不会有这种问题
	- 另外，对于Bulk inserts语句，依然会采取AUTO-INC锁。所以，如果有一条Bulk inserts语句正在执行的话，Simple inserts也必须等到该语句执行完毕才能继续执行
	
- interleaved
	- 该模式下，所有的 insert-like 语句都不会使用表级 auto-inc 锁，这种模式是来一个分配一个，只锁住 id 分配的过程，并且可以同时执行多个语句，是性能最好和最可扩展的锁定模式。它和 innodb_autoinc_lock_mode = 1 的区别在于，不会预分配多个
	- 由于可以同时为多个语句生成自增长值（即跨语句交叉编号），可能会造成同一个语句插入的行生成的 auto_incremant 值不连续，也就是存在间隙。比如执行 “bulk inserts” 时，则在给任何给定语句分配的自动递增值中可能存在间隙。但是如果执行的是语句是 “simple inserts”，其中要插入的行数可提前知道，那么不会有间隙。
	- 最后在主从复制replication的安全性方面，当 binlog_format 为 statement-based 时（简称 SBR，statement-based replication），则会存在问题，因为是来一个分配一个，当并发执行时，“bulk inserts” 在分配时会同时向其他的 insert 分配，从而出现主从不一致（从库执行结果和主库执行结果不一样），因为 binlog 只会记录开始的insert id。但是如果 binlog_format 为 row-based 则不会出现问题

### 主从安全性问题

主从的安全性：如果 binlog_format 使用基于行的或混合模式的复制，则所有自动增量锁定模式都是安全的，因为基于行的复制对SQL语句的执行顺序不敏感（混合模式会在遇到不安全的语句是使用基于行的复制模式），所以可以设置 innodb_autoinc_lock_mode = 2 可以获得更好的并发度。但是当 binlog_format 是基于语句复制 statement-base 的情况下，可设置 innodb_autoinc_lock_mode = 1，保证复制安全的同时，获得简单insert语句的最大并发度

innodb_autoinc_lock_mode 参数的设置是针对 innoDB 存储引擎，在 myisam 引擎情况下，无论什么样自增id锁都是表级锁

### 如何修改模式

查看当前模式 

```
show global variables like '%auto%'
```

可以通过find / -name my.cnf 查询命令找到配置文件的路径

然后修改该配置文件，加入一行 innodb_autoinc_lock_mode = 2，最后重启mysql

## id的连续性

### 导致不连续的情况

1. 如果生成自动递增值的事务回滚，那些自动递增值将丢失。 一旦为自动增量列生成了值，无论是否完成 “insert-like” 语句以及包含事务是否回滚，这些值都不能回滚，也就是这种丢失的值不被重用。 因此，存储在表的 auto_increment 列中的值可能存在间隙
2. 执行insert into table select语句
3. 插入时id指定为一个比较大的值

### 怎么改成连续

1. 备份数据，然后truncate，然后把备份写回原表，相当于删除重建，因为truncate = drop + create
2. 执行下面语句

```
ALTER  TABLE  `zh_user` DROP `id`;
ALTER  TABLE  `zh_user` ADD `id` MEDIUMINT(11) PRIMARY KEY NOT NULL AUTO_INCREMENT FIRST;
```

3. 执行下面语句

```
ALTER TABLE zh_user AUTO_INCREMENT=1
```

这三种方法本质上是一样的，都是备份、删除、回写数据

## 实例

先创建一张表zh_person，这张表包括4个字段，自增id，姓名name，性别sex和身份证号id_no,id_no上有唯一索引


```
CREATE TABLE `zh_person` (
  `id` MEDIUMINT(11) NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(64) COLLATE utf8_bin NOT NULL,
  `sex` VARCHAR(1) COLLATE utf8_bin NOT NULL,
  `id_no` VARCHAR(64) COLLATE utf8_bin NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `id_no_unique` (`id_no`)
) ENGINE=INNODB DEFAULT CHARSET=utf8 COLLATE=utf8_bin
```

以下几条语句，id会自增长


```
INSERT INTO zh_person(NAME, sex,id_no) VALUES('张三', '1','12345');
INSERT INTO zh_person(id,NAME, sex,id_no) VALUES(NULL,'李四', '1', '12346');
INSERT INTO zh_person(id,NAME, sex,id_no) VALUES(0,'王五', '1', '12347');
```

如果上面的3个sql执行成功了，这时候3条记录id依次是1、2、3

如果这是我再插入一条，id_no赋值“'12347'”，这时候唯一键冲突，sql如下


```
INSERT INTO zh_person(NAME, sex,id_no) VALUES('赵六', '1', '12347');
```

这时候虽然插入失败了，但是id的值还是增加了1，为什么这么说呢，我们修改上面的语句如下，插入成功后表里面虽然有4条记录，但是id是1、2、3、5


```
INSERT INTO zh_person(NAME, sex,id_no) VALUES('赵六', '1', '12348');
```

mysql的自增id在唯一索引冲突的时候不会回滚回去的，mysql在获取id时为了保证一致性，是加锁的，比如2个并发事务申请自增id，上面例子的情况，假如一个申请了4，一个申请了5，加入申请4的事务成功了，申请到5的事务唯一键冲突，这时候如果id回退到4，下一次插入必定是主键冲突。


```
create table zh_person2 like zh_person;
INSERT INTO zh_person2(NAME, sex, id_no) SELECT NAME, sex, id_no FROM zh_person;
```

上面的insert语句有4条记录，第一次申请id时分配了1个，不够用，第二次分配了2个，还是不够用，第三次申请4个，够用了，这是zh_person2表的自增id已经是8了，所以我们执行如下sql，插入的记录，id是8


```
INSERT INTO zh_person2(NAME, sex,id_no) VALUES('马七', '1', '12349');
```

这时zh_person2表里面总共有5条记录，id是1、2、3、4、8

## ID用完了怎么办

mysql数据库表的自增 ID 达到上限之后，这时候再申请它的值就不会再改变了，如果继续插入数据就会导致报主键冲突异常。
因此在做数据字典设计时，要根据业务的需求来选择合适的字段类型

建议采用bigint unsigned，这个数字就大了

在MYSQL 5.1.22版本前，自增列使用AUTO_INC Locking方式来实现，即采用一种特殊的表锁机制来保证并发插入下自增操作依然是串行操作，为提高插入效率，该锁会在插入语句完成后立即释放，而不是插入语句所在事务提交时释放。该设计并发性能太差，尤其在大批量数据在一条语句中插入时(INSERT SELECT ), 会导致该语句长时间持有这个“表锁”，从而阻塞其他事务的插入操作。

不过，还存在另一种情况，如果在创建表没有显示申明主键，会怎么办？

如果是这种情况，InnoDB会自动帮你创建一个不可见的、长度为6字节的row_id，而且InnoDB 维护了一个全局的 dictsys.row_id，所以未定义主键的表都共享该row_id，每次插入一条数据，都把全局row_id当成主键id，然后全局row_id加1

该全局row_id在代码实现上使用的是bigint unsigned类型，但实际上只给row_id留了6字节，这种设计就会存在一个问题：如果全局row_id一直涨，一直涨，直到2的48幂次-1时，这个时候再+1，row_id的低48位都为0，结果在插入新一行数据时，拿到的row_id就为0，存在主键冲突的可能性

## 参考资料

1. [MySQL自增主键auto_increment原理](https://blog.csdn.net/a745233700/article/details/117601376)
2. [Mysql之浅析AUTO_INCREMENT](https://juejin.cn/post/7062643529146695693)
3. [面试官:mysql如何重置自增id](https://cloud.tencent.com/developer/article/1683494)







































