---
layout:     post
title:      MySQL sql优化之munis实现
subtitle:   查询a表存在而b表不存在的记录
date:       2018-08-29
author:     cyhao
catalog: true
tags:
    - munis
    - 派生表
---

>Oracle中求差集可以使用munis，但是MySQL中没有这个功能，如何实现是下面查询需求需要用到的，这里我们使用相应的变换实现同等功能。

# 实现查询存在t1中而不存在t2表中的记录
表结构如下：
    
    create table t*(
     id number primary key,
    n1 number,
    nm varchar2(20));

### Oracle实现如下

```
SQL> select * from t1;

	ID	   N1 NM
---------- ---------- --------------------
	 1	    1 aa
	 2	    2 bb
	 3	    3 cc
	 4	    4 dd

SQL> select * from t2;

	ID	   N1 NM
---------- ---------- --------------------
	 1	    1 aa
	 2	    2 bb
	 3	    3 cc

SQL> select * from t1 minus select * from t2;

	ID	   N1 NM
---------- ---------- --------------------
	 4	    4 dd
```

如果表是不同结构，也可以使用部分字段结果集对比
    
    SQL> select id,nm from t1 minus select id,nm from t2;

	ID NM
    ---------- --------------------
	 4 dd
     SQL> select * from t1 where not exists (select * from t2 where t1.id=t2.id);

	ID	   N1 NM
    ---------- ---------- --------------------
	 4	    4 dd

    SQL> select * from t1 where id not in(select id from t2);

	ID	   N1 NM
    ---------- ---------- --------------------
	 4	    4 dd



### MySQL 实现

因为mysql没有munis，所以使用会报错
    
    mysql>select * from t1 minus select * from t2;
    You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'select * from t2' at line 1
    
当然可以使用not in或者not exists，还可以使用下面的方法实现
    
    mysql>select m.* from t1 m left join t2 n on m.id=n.id where n.id is null;
    +--------------+--------------+--------------+
    | id           | n1           | nm           |
    +--------------+--------------+--------------+
    |            4 |            4 | dd           |
    +--------------+--------------+--------------+

### 真实需求案例

通过条件查找cyhao2中符合条件的trans_id，在cyhao1中通过trans_id匹配记录，本来两个表中的trans_id应该是相同且相等，这时候cyhao1中记录不全，需要查找出cyhao1中缺失的记录

    select trans_id from cyhao2 where trans_id not in (select a.trans_id from cyhao1 a, cyhao2 b where a.TRANS_ID=b.TRANS_ID and b.ACC_DATE='2018-08-24'  and b.is_need_send='T') and  ACC_DATE='2018-08-24'  and is_need_send='T'
此sql执行了2,331,678ms，别惊讶，我的是测试环境1核1G的情况。查询sql的执行计划判断效率

```
mysql>explain select trans_id from cyhao2 where trans_id not in (select a.trans_id from cyhao1 a, cyhao2 b where a.TRANS_ID=b.TRANS_ID and b.ACC_DATE='2018-08-24'  and b.is_need_send='T') and  ACC_DATE='2018-08-24'  and is_need_send='T'
+--------------+-----------------------+-----------------+----------------+-------------------------+---------------+-------------------+--------------------+----------------+------------------------------------------------+
| id           | select_type           | table           | type           | possible_keys           | key           | key_len           | ref                | rows           | Extra                                          |
+--------------+-----------------------+-----------------+----------------+-------------------------+---------------+-------------------+--------------------+----------------+------------------------------------------------+
| 1            | PRIMARY               | cyhao2          | ALL            |                         |               |                   |                    | 76743          | Using where                                    |
| 2            | DEPENDENT SUBQUERY    | a               | ALL            | PRIMARY                 |               |                   |                    | 2411717        | Range checked for each record (index map: 0x1) |
| 2            | DEPENDENT SUBQUERY    | b               | eq_ref         | PRIMARY                 | PRIMARY       | 194               | cyhtest.a.TRANS_ID | 1              | Using where                                    |
+--------------+-----------------------+-----------------+----------------+-------------------------+---------------+-------------------+--------------------+----------------+------------------------------------------------+
返回行数：[3]，耗时：9 ms.
```
一般逻辑我们可能是说not in中的查询先过滤结果，然后与外层查询匹配。但是实际结果是外层结果一条条与not in中的记录匹配（DEPENDENT SUBQUERY---子查询中的第一个SELECT，取决于外面的查询），也就是说外层每遍历一条记录就执行一次内层sql的查询操作，所以效率会非常低。

改写思路：先构造一个表包含两个表中都有的记录，然后左连接查询--多的左连少的，最后过滤派生表trans_id为空的记录
```
mysql>SELECT
	m.trans_id
FROM
	cyhao2 m
LEFT JOIN (
	SELECT
		b.*
	FROM
		cyhao1 a,
		cyhao2 b
	WHERE
		a.TRANS_ID = b.TRANS_ID
	AND b.ACC_DATE = '2018-08-24'
	AND b.is_need_send = 'T'
) n ON m.trans_id = n.trans_id
WHERE
	n.trans_id IS NULL
AND m.ACC_DATE = '2018-08-24'
AND m.is_need_send = 'T';
+------------------------------+
| trans_id                     |
+------------------------------+
| 20102477278751414037849 |
+------------------------------+
返回行数：[1]，耗时：15257 ms.
```

速度快了100多倍，我们查看新sql的执行计划
```
mysql>explain SELECT
	m.trans_id
FROM
	cyhao2 m
LEFT JOIN (
	SELECT
		b.*
	FROM
		cyhao1 a,
		cyhao2 b
	WHERE
		a.TRANS_ID = b.TRANS_ID
	AND b.ACC_DATE = '2018-08-24'
	AND b.is_need_send = 'T'
) n ON m.trans_id = n.trans_id
WHERE
	n.trans_id IS NULL
AND m.ACC_DATE = '2018-08-24'
AND m.is_need_send = 'T';
+--------------+-----------------------+-----------------+----------------+-------------------------+---------------+-------------------+--------------------+----------------+-------------------------+
| id           | select_type           | table           | type           | possible_keys           | key           | key_len           | ref                | rows           | Extra                   |
+--------------+-----------------------+-----------------+----------------+-------------------------+---------------+-------------------+--------------------+----------------+-------------------------+
| 1            | PRIMARY               | m               | ALL            |                         |               |                   |                    | 76743          | Using where             |
| 1            | PRIMARY               | <derived2>      | ref            | <auto_key0>             | <auto_key0>   | 194               | cyhtest.m.TRANS_ID | 31             | Using where; Not exists |
| 2            | DERIVED               | a               | index          | PRIMARY                 | idx_tdate     | 5                 |                    | 2411717        | Using index             |
| 2            | DERIVED               | b               | eq_ref         | PRIMARY                 | PRIMARY       | 194               | cyhtest.a.TRANS_ID | 1              | Using where             |
+--------------+-----------------------+-----------------+----------------+-------------------------+---------------+-------------------+--------------------+----------------+-------------------------+
返回行数：[4]，耗时：10 ms.
```

从上面的执行结果和执行计划可以看出效率方面大幅度提升。当然并不是没有更加优的方法，下面在cyhao2表中添加合适的索引。
```
mysql>alter table cyhao2 add index idx_adate_isend(acc_date,is_need_send);
执行成功，耗时：294 ms.
mysql>SELECT
	m.trans_id
FROM
	cyhao2 m
LEFT JOIN (
	SELECT
		b.*
	FROM
		cyhao1 a,
		cyhao2 b
	WHERE
		a.TRANS_ID = b.TRANS_ID
	AND b.ACC_DATE = '2018-08-24'
	AND b.is_need_send = 'T'
) n ON m.trans_id = n.trans_id
WHERE
	n.trans_id IS NULL
AND m.ACC_DATE = '2018-08-24'
AND m.is_need_send = 'T';
+------------------------------+
| trans_id                     |
+------------------------------+
| 20102477278751414037849 |
+------------------------------+
返回行数：[1]，耗时：5093 ms.
mysql>explain SELECT
	m.trans_id
FROM
	cyhao2 m
LEFT JOIN (
	SELECT
		b.*
	FROM
		cyhao1 a,
		cyhao2 b
	WHERE
		a.TRANS_ID = b.TRANS_ID
	AND b.ACC_DATE = '2018-08-24'
	AND b.is_need_send = 'T'
) n ON m.trans_id = n.trans_id
WHERE
	n.trans_id IS NULL
AND m.ACC_DATE = '2018-08-24'
AND m.is_need_send = 'T';
+--------------+-----------------------+-----------------+----------------+-------------------------+-----------------+-------------------+--------------------+----------------+--------------------------+
| id           | select_type           | table           | type           | possible_keys           | key             | key_len           | ref                | rows           | Extra                    |
+--------------+-----------------------+-----------------+----------------+-------------------------+-----------------+-------------------+--------------------+----------------+--------------------------+
| 1            | PRIMARY               | m               | ref            | idx_adate_isend         | idx_adate_isend | 138               | const,const        | 126            | Using where; Using index |
| 1            | PRIMARY               | <derived2>      | ref            | <auto_key0>             | <auto_key0>     | 194               | cyhtest.m.TRANS_ID | 31             | Using where; Not exists  |
| 2            | DERIVED               | a               | index          | PRIMARY                 | idx_tdate       | 5                 |                    | 2411717        | Using index              |
| 2            | DERIVED               | b               | eq_ref         | PRIMARY,idx_adate_isend | PRIMARY         | 194               | cyhtest.a.TRANS_ID | 1              | Using where              |
+--------------+-----------------------+-----------------+----------------+-------------------------+-----------------+-------------------+--------------------+----------------+--------------------------+
返回行数：[4]，耗时：9 ms.
```
加上索引之后执行计划改变，实际结果也有了改善。当然表cyhao2本来数据不算多，如果是数据量较大的表优化效果会更好。




