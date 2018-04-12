---
title: Java进阶-mysql事务
date: 2018-04-04 08:57:05
categories: ['Java进阶', 'mysql']
tags: ['mysql', '事务']
---

在使用数据库时，因为存在不同的用户操作同一条数据，数据库可能出现如下问题：

* 脏读：表示一个事务正在访问数据，并且对数据进行了修改，而这种修改还么提交到数据库，这个时候，另一个事务也访问了这个数据，然后获取到了该事务未提交的数据。
* 不可重复读：是指在一个事务内，多次读取同一数据。在这个事务还没结束时，另一个事务也访问并修改了改数据（已经提交事务），第一个事务再次读取数据发现两次读取的数据不一样。
* 幻读：一个事务计划对表中数据进行修改，同时第二个事务向表中插入了一条数据（提交事务），符合第一个事务中的条件，之后第一个事务对表进行修改，那么对于操作第一个事务的用户而言，会发现表中还有其他数据（第二个事务），如同幻觉。
<!-- more -->
### 事务导致的问题

#### 脏读
```sql
-- 开启第一个连接
-- 先设置数据库事务未最低级别
mysql1> set SESSION  transaction ISOLATION LEVEL READ UNCOMMITTED;
-- 设置事务不自动提交
mysql1> set autocommit = 0;
mysql1> start transaction;
mysql1> SELECT * FROM item order by id desc;

+----+------------+----------+-----+--------+
| id | serial_num | sku_code | num | status |
+----+------------+----------+-----+--------+
|  1 | 000001     | K00006   |   5 |      1 |
+----+------------+----------+-----+--------+

-- 开启第二个连接
-- 设置事务不自动提交
mysql2> set autocommit = 0;
-- 开启事务不提交事务
mysql2> start transaction;
mysql2> update item set status = 2 where i = 1;

-- 切换到第一个事务查询数据
mysql1> select * from item;
+----+------------+----------+-----+--------+
| id | serial_num | sku_code | num | status |
+----+------------+----------+-----+--------+
|  1 | 000001     | K00006   |   5 |      2 |
+----+------------+----------+-----+--------+

-- 查询到了第二个事务未提交的数据

-- 切换到第二个事务回滚
mysql2> rollback;
```
如上操作，事务1读取到了事务2未提交的数据。在有些情况下`脏读`比较危险，如：A转账给B，A转账后，还有后续逻辑，都在一个事务里面，这时B去查账，查询到了余额的增加，如果B中某个耗时逻辑发送了错误导致事务回滚，这样实际是没有成功，对于B而言之前是已经看见成功了。

在使用自增ID时，此时自增ID为1，如果一个事务提交了insert，生成的自增加ID为2，另外一个事务也提交了insert，生成的自增ID为3，因为第一个事务并未提交，第二个事务却生成的自增ID为3。这勉强可以视为脏的读，只不过这种情况是为了在并发时提升数据库性能做的操作。

#### 不可重复读
不可重复读和脏读的区别在于，在事务内，一个是读取到了另一个事务未提交的数据；一个是读取了另一个已经提交的数据，前后读取的数据不一致。
```sql
mysql1> set SESSION  transaction ISOLATION LEVEL READ COMMITTED;
mysql1> commit;
mysql1> start transaction;
mysql1> SELECT * FROM item order by id desc;
+----+------------+----------+-----+--------+
| id | serial_num | sku_code | num | status |
+----+------------+----------+-----+--------+
|  1 | 000001     | K00006   |   5 |      1 |
+----+------------+----------+-----+--------+

-- 切换事务二，修改数据
mysql2> set SESSION  transaction ISOLATION LEVEL READ COMMITTED;
mysql2> commit;
mysql2> start transaction;
mysql2> update item set status = 2 where id=1;

-- 切换事务一，查询数据。
mysql1> SELECT * FROM item order by id desc;
+----+------------+----------+-----+--------+
| id | serial_num | sku_code | num | status |
+----+------------+----------+-----+--------+
|  1 | 000001     | K00006   |   5 |      1 |
+----+------------+----------+-----+--------+
-- 事务查询正常，没查到未提交数据
-- 切换事务二，提交事务
mysql2> commit;
-- 切换事务一，查询数据。
mysql1> SELECT * FROM item order by id desc;
+----+------------+----------+-----+--------+
| id | serial_num | sku_code | num | status |
+----+------------+----------+-----+--------+
|  1 | 000001     | K00006   |   5 |      2 |
+----+------------+----------+-----+--------+
mysql1> commit;
```
如上操作，事务一在自己事务内读取到的数据前后不一致。
#### 幻读
```sql
-- 事务一，修改数据
mysql1> set SESSION  transaction ISOLATION LEVEL REPEATABLE READ ;
mysql1> commit;
mysql1> start transaction;
mysql1> SELECT * FROM item order by id desc;
+----+------------+----------+-----+--------+
| id | serial_num | sku_code | num | status |
+----+------------+----------+-----+--------+
|  1 | 000001     | K00006   |   5 |      2 |
+----+------------+----------+-----+--------+
-- 事务二插入操作
mysql2> set SESSION  transaction ISOLATION LEVEL REPEATABLE READ ;
mysql2> commit;
mysql2> start transaction;
mysql2> insert into item(serial_num, sku_code, num, status) VALUES ('000002', 'K00006', 5, 1);
mysql2> commit;
-- 事务2修改数据后，事务一查看数据
mysql1> SELECT * FROM item order by id desc;
+----+------------+----------+-----+--------+
| id | serial_num | sku_code | num | status |
+----+------------+----------+-----+--------+
|  1 | 000001     | K00006   |   5 |      2 |
+----+------------+----------+-----+--------+
-- 此时并未查询到事务二已经提交的事务，此时修改数据库数据
mysql1> update item set status=4 where sku_code = 'K00006';
mysql1> SELECT * FROM item order by id desc;
+----+------------+----------+-----+--------+
| id | serial_num | sku_code | num | status |
+----+------------+----------+-----+--------+
|  2 | 000002     | K00006   |   5 |      4 |
|  1 | 000001     | K00006   |   5 |      4 |
+----+------------+----------+-----+--------+
-- 在事务一内原本只有一条数据的，结果更新后更新了2条数据，而且在查询时，把事务二的数据查询出来了
mysql1> commit;
```
在原本事务一中只有一条数据，在事务二插入提交后，事务一中对数据进行更新，此时事务二中提交的数据被修改查询出来。

### 事务隔离级别
在上述例子中，每个例子都是使用的不同的事务隔离级别。

设置事务隔离级别命令
`SET [SESSION | GLOBAL] TRANSACTION ISOLATION LEVEL {READ UNCOMMITTED | READ COMMITTED | REPEATABLE READ | SERIALIZABLE}`

不同的事务隔离级别用于解决不同的问题：

| --- | 脏读 | 不可重复读 | 幻读|
| --- | :--- | :--- | :--- |
| READ UNCOMMITTED | 可能 | 可能 | 可能 |
| READ COMMITTED | 不可能 | 可能 | 可能 |
| REPEATABLE READ（mysql InnoDB 默认） | 不可能 | 不可能| 可能|
|SERIALIZABLE | 不可能 |不可能 |不可能 |

对于`SERIALIZABLE`并没有举例，在使用`SERIALIZABLE`实际上是自己锁住了数据，其他数据需要修改只能等待锁释放。


参考：[MySQL 四种事务隔离级的说明](http://www.cnblogs.com/zhoujinyi/p/3437475.html)