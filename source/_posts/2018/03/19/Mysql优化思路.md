---
title: Mysql优化思路
date: 2018-03-19 09:51:52
categories: ['mysql']
tags: ['优化', '分库', '分区', '分表']
---

在业务增长过程中，数据库中的数据量是越来越大，之前在国美时，公司都是用的oracle能承受的数据量比较大，那时候没感觉到数据库是瓶颈，基本上都是由dba直接做了数据库层的优化，基本上我们使用不需要做修改特别的优化，一般情况下都是优化sql，后期可能因为oracle太贵，部分数据存入mongodb和mysql中，在之后因为已经离职还未见到需要对mysql进行特别的优化。

在现在公司中，特别倾向于对mysql进行分库分表，其实有时候并不一定需要，比如把大量的数据存入mysql中用于查询，这种情况不应该通过ELK来做日志采集和查询吗。

在数据增长过程中，并不是直接就进行分库分表，可以分步骤进行。
> 1. sql优化、索引
> 2. 热点数据缓存、redis、memcached
> 3. 读写分离
> 4. 垂直切分
> 5. 分区
> 6. 分库、分表

参考：[MySQL 对于千万级的大表要怎么优化？ - zhuqz的回答 - 知乎](https://www.zhihu.com/question/19719997/answer/81930332)
<!-- more -->

### sql优化、索引
一般情况下都打开了mysql慢查询日志，记录超过阈值的sql，用于之后sql的优化。
在优化sql过程中，通过 **执行计划** 来判断时间花费在哪：
* 判断索引是否生效
* 是否需要新建索引
* 是否应该创建多列索引
* 是否需要创建临时表
* 是否可以把多条sql合并为一条
* 是否可以吧一条过于复杂sql拆分为不同sql，在应用中对数据进行合并

带着问题去对相应的sql优化。

### 热点数据缓存
在项目中引入缓存入redis、memcached等缓存，对热点数据走缓存，避免数据库压力过大，同时可以引入本地缓存。
在使用缓存过程中可能会出现缓存击穿，因为使用缓存生效时间（不建议使用永久生效），这时可能会出现大量线程访问数据库导致数据库压力瞬间增大，严重可能导致数据库挂掉，可以在查询无返回值时，通过使用互斥锁，使一个线程去数据库同步数据到缓存，其他线程等待（设置等待超时时间）；还可以在获取缓存中数据时，判断超时时间，如果超过设置的阈值后同步数据库数据同时延长缓存时间。

### 读写分离
读写分离适用于读远大于写，在使用读写分离时会出现读延迟的情况，如果对于某个业务要求较高，可以走写库。
以前公司读写分离好像是在数据库层进行区分，现在公司是通过在应用修改数据源达到读写分离的目的，通过拦截器判断Mapper中方法，切换不同的数据源，通过ThreadLocal存储连接池。在使用过程中需要需要测试如果配置事务后，事务是否生效，如果配置不当可能会出现事务失效的情况。

### 垂直切分
在前期设计的时候，可能所有业务都在一个库或者一个表中，导致表特别臃肿，这个时候可以对数据库进行垂直切分，如果是切分数据库，可以通过不同的业务对库进行拆分，对相应的代码进行重构。如果是对表进行拆分，可以把改表中非主要字段进行拆分放入另一张表中，因为这里需要变动应用操作数据库相关的代码，可能应用重构较多

### 分区
在mysql中，数据库中的数据都是放入在一个文件中，随着数据的增长，该文件也会变得越来越大。mysql可以通过分区在把该文件拆分为多个文件，通过判断数据在哪个文件来对不同的文件进行查找，这就减少数据查询量。在物理上数据被拆分了，逻辑上还是一个表。
分区分为：RANGE、List、Hash、Key、子分区

#### RANGE分区
根据范围分区，范围应该连续但是不重叠，使用PARTITION BY RANGE, VALUES LESS THAN关键字。不使用COLUMNS关键字时RANGE括号内必须为整数字段名或返回确定整数的函数。
```sql
# 如果不加COLUMNS，则值必须为数字，加了COLUMNS后可以支持非整数和多列
PARTITION BY RANGE COLUMNS(字段名) (
    PARTITION p0 VALUES LESS THAN ('1960-01-01'),
    PARTITION p1 VALUES LESS THAN ('1970-01-01'),
    PARTITION p2 VALUES LESS THAN ('1980-01-01'),
    PARTITION p3 VALUES LESS THAN ('1990-01-01'),
    PARTITION p_max  VALUES LESS THAN MAXVALUE
);
# 或者
ALTER TABLE 表 ADD PARTITION (PARTITION p4 VALUES LESS THAN ('2000-01-01'));
```
一般在平常都是通过常用where条件中的字段对数据进行拆分。除了对单个字段进行拆分，还可以对多个字段进行拆分。

#### List分区
根据具体数值分区，每个分区数值不重叠，使用PARTITION BY LIST、VALUES IN关键字。跟Range分区类似，不使用COLUMNS关键字时List括号内必须为整数字段名或返回确定整数的函数。
```sql
PARTITION BY LIST(字段) (
    PARTITION pNorth VALUES IN (3,5,6,9,17),
    PARTITION pEast VALUES IN (1,2,10,11,19,20),
    PARTITION pWest VALUES IN (4,12,13,14,18),
    PARTITION pCentral VALUES IN (7,8,15,16)
);
```
数字List分区时，设置的字段值必须全面覆盖，如果插入一个不属于该分区的值会出错。同时可以通过添加`COLUMNS`支持非整数和多列。在进行批量插入时，如 **values(),(),()** 这种格式，其中有条数据没被覆盖，如果表引擎支持事务（Innodb）则都会失败，如果不支持事务则该未覆盖的数据之前数据会被插入，之后数据不会被插入。

#### Hash分区
Hash分区主要用来确保数据在预先确定数目的分区中平均分布，Hash括号内只能是整数列或返回确定整数的函数，实际上就是使用返回的整数对分区数取模。
```sql
PARTITION BY HASH(字段)
PARTITIONS 4;
```
通过对字段进行hash，之后拆分为多少份。使用Hash时，后期扩展较差。可以通过添加`LINEAR`关键字线性Hash分区，类似Hash一致。

#### Key分区
Key分区和Hash分区类似，只是Key分区使用的哈希函数使用的事Mysql服务器提供。使用只要把`HASH`改为`KEY`即可。

#### 子分区
子分区是将每个分区再次分割。
```sql
CREATE TABLE ts (id INT, purchased DATE)
    PARTITION BY RANGE( YEAR(purchased) )
    SUBPARTITION BY HASH( TO_DAYS(purchased) ) (
        PARTITION p0 VALUES LESS THAN (1990) (
            SUBPARTITION s0
                DATA DIRECTORY = '/disk0/data'
                INDEX DIRECTORY = '/disk0/idx',
            SUBPARTITION s1
                DATA DIRECTORY = '/disk1/data'
                INDEX DIRECTORY = '/disk1/idx'
        ),
        PARTITION p1 VALUES LESS THAN (2000) (
            SUBPARTITION s2
                DATA DIRECTORY = '/disk2/data'
                INDEX DIRECTORY = '/disk2/idx',
            SUBPARTITION s3
                DATA DIRECTORY = '/disk3/data'
                INDEX DIRECTORY = '/disk3/idx'
        ),
        PARTITION p2 VALUES LESS THAN MAXVALUE (
            SUBPARTITION s4
                DATA DIRECTORY = '/disk4/data'
                INDEX DIRECTORY = '/disk4/idx',
            SUBPARTITION s5
                DATA DIRECTORY = '/disk5/data'
                INDEX DIRECTORY = '/disk5/idx'
        )
    );
```
使用子分区的时候，每个子分区的数量必须相同。如果在一个分区表上的任何分区上使用`SUBPARTITION`来明确定义任何子分区，那么就必须定义所有的子分区，且必须指定一个全表唯一的名字。

在对数据进行分区后，如果查询条件依旧很慢，可以通过执行计划来分析。

在分区完成后，后续需要对分区进行操作时可以对分区进行删除、合并、新增，删除有两种`REMOVE`和`DROP`，`REMOVE`不会删除分区，只会删除分区的结构，`DROP`会连数据都删除。

参考：[MySQL分区与传统的分库分表](http://haitian299.github.io/2016/05/26/mysql-partitioning/)

### 分库、分表
当分区已经无法满足业务需求时，这时需要对数据库进行分库、分表。

#### Hash拆分
对数据库中某个字段或者多个字段进行Hash后进行取模，通过获取到的值，分配到不同的库和表中，这种拆分比较简便。
如现在有个评论表，评论中有用户的唯一ID，通过对用户的ID进行Hash，通过Hash取模后分配到指定的库和表中，这种在Insert的时候和Select的时候可以直接在应用中通过之前设计的Hash算法查询不同的数据库。
这种有个缺陷，如果后续数据持续的增长之前设计的拆分数量不够，这时需要重新拆分，这样迁移的数据会比较多，影响也比较大。

可以过Hash一致算法来避免后续新增库表时导致大量数据迁移。

#### 存储拆分的值和库表映射关系
拆分时，设计一个主库，存储拆分的字段和对应库表关系，在查询时先通过调节查询到本次查询的数据需要从哪个库表进行查询，之后在到指定的库表查询具体的值，需要查询2次。如：现在有多个门店，每个门店都有自己的商品，已经销量等数据。通过一张主表维护门店信息，查询出门店数据所在库表后，再查询需要查询的数据。这种情况下，如果需要新增一个门店，可以把门店在主库的映射表中把添加门店映射，比较方便。这种缺陷在于需要查询两次数据库，会有性能消耗，一般这种可以通过依赖第三方库(当当sharding-jdbc-core)，简化数据库操作。

在对数据进行分库、分表的时候，为了不影响用户的使用，通常可以通过双写来做数据迁移，以旧库为主，新库为辅，同时通过定时对数据进行检测避免双写失败，同时需要做到随时对库进行切换。
