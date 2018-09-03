---
layout:     post
title:      MySQL 原理之事务-transaction
subtitle:   事务基础理解
date:       2018-09-01
author:     cyhao
catalog: true
tags:
    - mysql
    - 事务
    - transaction
---


# 事务
MySQL中最小的工作单元，是一系列序列操作的集合，事务可以提交和回滚，而且单元内的所有sql要么都提交，要么都回滚。

# 事务的特性：ACID
### 1、Atomicity-原子性
原子性保证事务内的所有操作全部执行成功，或者由于错误导致所有操作回滚，事务内的操作不能部分成功。

### 2、Consistency-一致性
保证数据库在事务前后数据库从一个一致的状态变成另外一个一致的状态。比如数据库的完整性约束正确，日志状态一致等。

### 3、Isolation-隔离性
事务的执行不受其它事务的干扰，即一个事务内的操作对外是隔离的，并发事务之间互不干扰。

### 4、Durability-持久性
事务提交的结果写入数据库中并永久存储起来。

# 事务控制语言
● START TRANSACTION or BEGIN start a new transaction.

● COMMIT commits the current transaction, making its changes permanent.

● ROLLBACK rolls back the current transaction, canceling its changes.

● SET autocommit disables or enables the default autocommit mode for the current session.

start transaction会使autocommit失效，直到事务commit或者rollback之后才恢复到之前的状态。

# 事务的隔离级别

|事务隔离级别|脏读（dirty read)|不可重复读（Non-Repeatable Read）|幻读（phantom read）|
|-----------|:---------------:|:-----------------------------:|:------------------:|
|未提交读（read uncommitted） |可能 |可能 |可能|
|提交读（read committed） |不可能 |可能 |可能|
|可重复读（repeatable read） |不可能 |不可能|可能|
|串行化（serializable） |不可能 |不可能|不可能|

脏读：一个事务读取另外一个事务中未提交的数据（脏数据）

不可重复读：其它事务对一条记录的修改导致另外一个事务内多次读取这条记录显示不同的结果。

幻读：一个事务多次读取会获得其它事务新插入或删除的数据。

有人说不可重复读和幻读有什么区别，难道不都是一个事务内相同的查询获取不同的结果吗？事实是结果确实是这样，但是不可重复读对应的是原来数据有所表更（update），而幻读对应新增数据(insert，delete)。

# 实验环境：MySQL 5.7，事务隔离级别RR，binlog格式为ROW，autocommit=1，innodb_locks_unsafe_for_binlog=0
查看当前系统事务隔离级别
```
mysql> select @@global.tx_isolation;
+-----------------------+
| @@global.tx_isolation |
+-----------------------+
| REPEATABLE-READ       |
```

查看当前会话事务隔离级别
```
mysql> select @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
```

### 修改当前会话事务隔离级别测试脏读---RU：

|事务A |事务B|
|------|----|
|mysql> set session transaction isolation level read uncommitted;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> select @@tx_isolation;<br>+------------------+<br>&#124; @@tx_isolation   &#124;<br>+------------------+<br>&#124; READ-UNCOMMITTED &#124;<br>+------------------+ | mysql> set session transaction isolation level read uncommitted;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> select @@tx_isolation;<br>+------------------+<br>&#124; @@tx_isolation   &#124;<br>+------------------+<br>&#124; READ-UNCOMMITTED &#124;<br>+------------------+|
|mysql> start transaction;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> select * from tt;<br>Empty set (0.00 sec) | |
| |mysql> start transaction;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> insert into tt values(1,'aa');<br>Query OK, 1 row affected (0.00 sec)<br>mysql> select * from tt;<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; aa   &#124;<br>+----+------+<br>1 row in set (0.00 sec)<br>#事务并未提交|
|mysql> select * from tt;<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; aa   &#124;<br>+----+------+<br>1 row in set (0.00 sec)<br>#事务A能查到事务B未提交的脏数据| | 
| |mysql> rollback;<br>Query OK, 0 rows affected (0.04 sec)<br>mysql> select * from tt;<br>Empty set (0.00 sec)<br>事务B回滚，等于没做任何操作|
|mysql> select * from tt;<br>Empty set (0.00 sec) | |

### 不可重复读--RC：

|事务A |事务B|
|------|----|
|mysql> set session transaction isolation level read committed;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> select @@tx_isolation;<br>+----------------+<br>&#124; @@tx_isolation &#124;<br>+----------------+<br>&#124; READ-COMMITTED &#124;<br>+----------------+<br>1 row in set, 1 warning (0.00 sec) |mysql> set session transaction isolation level read committed;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> select @@tx_isolation;<br>+----------------+<br>&#124; @@tx_isolation &#124;<br>+----------------+<br>&#124; READ-COMMITTED &#124;<br>+----------------+<br>1 row in set, 1 warning (0.00 sec)|
|mysql> start transaction;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> select * from tt;<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; aa   &#124;<br>+----+------+<br>1 row in set (0.00 sec) |
| |mysql> start transaction;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> update tt set name='bb' where id=1;<br>Query OK, 1 row affected (0.00 sec)<br>Rows matched: 1  Changed: 1  Warnings: 0<br>mysql> select * from tt;<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; bb   &#124;<br>+----+------+<br>1 row in set (0.00 sec)|
|mysql> select * from tt;<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; aa   &#124;<br>+----+------+<br>1 row in set (0.00 sec)<br>这时候没有读取到事务B产生的脏数据，避免了脏读| | 
| |mysql> commit;<br>Query OK, 0 rows affected (0.00 sec)|
|mysql> select * from tt;<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; bb   &#124;<br>+----+------+1 row in set (0.00 sec)<br>事务B提交的事务在A中可读 | |
|mysql> commit;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> select * from tt;<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; bb   &#124;<br>+----+------+<br>1 row in set (0.00 sec) | |

### 幻读--RC，至于为什么不在repeatable read事务隔离级别下分析，会在后面分析：

|事务A |事务B|
|------|----|
|mysql> set session transaction isolation level read committed;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> select @@tx_isolation;<br>+----------------+<br>&#124; @@tx_isolation &#124;<br>+----------------+<br>&#124; READ-COMMITTED &#124;<br>+----------------+<br>1 row in set, 1 warning (0.00 sec) |mysql> set session transaction isolation level read committed;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> select @@tx_isolation;<br>+----------------+<br>&#124; @@tx_isolation &#124;<br>+----------------+<br>&#124; READ-COMMITTED &#124;<br>+----------------+<br>1 row in set, 1 warning (0.00 sec)|
|mysql> start transaction;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> select * from tt where name>='bb' and name <='ff';<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; bb   &#124;<br>&#124;  2 &#124; ff   &#124;<br>+----+------+<br>2 rows in set (0.03 sec) | |
| |mysql> start transaction;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> insert into tt values(4,'dd');<br>Query OK, 1 row affected (0.00 sec)|
|mysql> select * from tt where name>='bb' and name <='ff';<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; bb   &#124;<br>&#124;  2 &#124; ff   &#124;<br>+----+------+<br>2 rows in set (0.00 sec) | |
| |mysql> commit;<br>Query OK, 0 rows affected (0.03 sec)|
|mysql> select * from tt;<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; bb   &#124;<br>&#124;  4 &#124; dd   &#124;<br>&#124;  2 &#124; ff   &#124;<br>&#124;  3 &#124; zz   &#124;<br>+----+------+<br>4 rows in set (0.00 sec)| | 

从上面的分析中可知RC级别是会出现不可重复读和幻读的。

### 现在我们在RR模式下分析下幻读的情况:

|事务A |事务B|
|------|----|
|mysql> select @@tx_isolation;<br>+-----------------+<br>&#124; @@tx_isolation  &#124;<br>+-----------------+<br>&#124; REPEATABLE-READ &#124;<br>+-----------------+ |mysql> select @@tx_isolation;<br>+-----------------+<br>&#124; @@tx_isolation  &#124;<br>+-----------------+<br>&#124; REPEATABLE-READ &#124;<br>+-----------------+|
|mysql> start transaction;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> select * from tt;<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; aa   &#124;<br>&#124;  4 &#124; dd   &#124;<br>&#124;  2 &#124; ff   &#124;<br>&#124;  3 &#124; zz   &#124;<br>+----+------+<br>4 rows in set (0.00 sec)<br>mysql> select * from tt where name>='ff' and name <='zz';<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  2 &#124; ff   &#124;<br>&#124;  3 &#124; zz   &#124;<br>+----+------+<br>2 rows in set (0.00 sec)| | 
| |mysql> start transaction;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> insert into tt values(5,'gg');<br>Query OK, 1 row affected (0.00 sec)|
|mysql> select * from tt where name>='ff' and name <='zz';<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  2 &#124; ff   &#124;<br>&#124;  3 &#124; zz   &#124;<br>+----+------+<br>2 rows in set (0.00 sec)| 
| |mysql> commit;<br>Query OK, 0 rows affected (0.02 sec)|
|mysql> select * from tt where name>='ff' and name <='zz';<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  2 &#124; ff   &#124;<br>&#124;  3 &#124; zz   &#124;<br>+----+------+<br>2 rows in set (0.00 sec)<br>mysql> select * from tt;<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; aa   &#124;<br>&#124;  4 &#124; dd   &#124;<br>&#124;  2 &#124; ff   &#124;<br>&#124;  3 &#124; zz   &#124;<br>+----+------+<br>4 rows in set (0.00 sec)<br>mysql> rollback;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> select * from tt;<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; aa   &#124;<br>&#124;  4 &#124; dd   &#124;<br>&#124;  2 &#124; ff   &#124;<br>&#124;  5 &#124; gg   &#124;<br>&#124;  3 &#124; zz   &#124;<br>+----+------+<br>5 rows in set (0.00 sec)| | 

### 从上面的结果看是不会存在幻读，那我们看下面的实验：

|事务A |事务B|
|------|----|
|mysql> start transaction;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> select * from tt;<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; aa   &#124;<br>&#124;  4 &#124; dd   &#124;<br>&#124;  2 &#124; ff   &#124;<br>&#124;  3 &#124; zz   &#124;<br>+----+------+<br>4 rows in set (0.00 sec)| | 
| |mysql> start transaction;<br>Query OK, 0 rows affected (0.00 sec)<br>mysql> insert into tt values(5,'hh');<br>Query OK, 1 row affected (0.00 sec)<br>mysql> commit;<br>Query OK, 0 rows affected (0.05 sec)|
|mysql> select * from tt;<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; aa   &#124;<br>&#124;  4 &#124; dd   &#124;<br>&#124;  2 &#124; ff   &#124;<br>&#124;  3 &#124; zz   &#124;<br>+----+------+<br>4 rows in set (0.00 sec)<br>mysql> select * from tt for update;<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; aa   &#124;<br>&#124;  4 &#124; dd   &#124;<br>&#124;  2 &#124; ff   &#124;<br>&#124;  5 &#124; hh   &#124;<br>&#124;  3 &#124; zz   &#124;<br>+----+------+<br>5 rows in set (0.00 sec)<br>mysql> select * from tt lock in share mode;<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; aa   &#124;<br>&#124;  4 &#124; dd   &#124;<br>&#124;  2 &#124; ff   &#124;<br>&#124;  5 &#124; hh   &#124;<br>&#124;  3 &#124; zz   &#124;<br>+----+------+<br>5 rows in set (0.00 sec)<br>mysql> select * from tt;<br>+----+------+<br>&#124; id &#124; name &#124;<br>+----+------+<br>&#124;  1 &#124; aa   &#124;<br>&#124;  4 &#124; dd   &#124;<br>&#124;  2 &#124; ff   &#124;<br>&#124;  3 &#124; zz   &#124;<br>+----+------+<br>4 rows in set (0.00 sec)| | 

mysql  innodb存储引擎提供快照读和当前读，从上面的结果看出当前读和快照读得到的结果并不一样。使用加锁读，就会得到“最新”的结果。

  By default, InnoDB operates in REPEATABLE READ transaction isolation level and with the innodb_locks_unsafe_for_binlog system variable disabled. In this case, InnoDB uses next-key locks for searches and index scans, which prevents phantom rows

官方文档说出如何避免幻读，那就是加锁，不光是record lock，而是加上gap lock组成的next-key lock。对于一般的查询并不会主动加锁，所以需要应用来判断是否需要加锁读避免幻读。

  For locking reads (SELECT with FOR UPDATE or LOCK IN SHARE MODE),UPDATE, and DELETE statements, locking depends on whether the statement uses a unique index with a unique search condition, or a range-type search condition. For a unique index with a unique search condition, InnoDB locks only the index record found, not the gap before it. For other search conditions, InnoDB locks the index range scanned, using gap locks or next-key (gap plus index-record) locks to block insertions by other sessions into the gaps covered by the range.
                                                                                                                                                                                                       |                                                                                                                                                                                                                                                                                                |











  
  
