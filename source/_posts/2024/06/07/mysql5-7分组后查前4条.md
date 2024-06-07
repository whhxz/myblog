---
title: mysql5.7分组后查前4条
date: 2024-06-07 08:37:46
categories:
tags: ['mysql']
---

最近有一条sql因为jar依赖的问题导致异常，以前旧版本是正常的，如下：
```sql
-- 建表
create table fresh_product(
    id bigint auto_increment primary key ,
    activity_id bigint not null default 0,
    product_id varchar(20) not null default ''
);

-- 查询每个activity_id前4条
set @lineNum = 0, @activityId = 0;
select activity_id activityId, goodsNo
from (select activity_id,CONVERT(product_id,SIGNED) goodsNo,
             @lineNum := IF(@activityId = activity_id, @lineNum + 1, 1) as lineNum,
             @activityId := activity_id  t
      from fresh_product where activity_id in
        (10000745,10000738, 10000732)
      order by activity_id asc,goodsNo desc) tmp
where tmp.lineNum <= 4;
```
现在需要改造该sql，改造如下：
```sql
select SUBSTRING_INDEX(SUBSTRING_INDEX(t.product_ids, ',', nums.n), ',', -1) id,t.activity_id
from (select 1 n
    union all
    select 2
    union all
    select 3
    union all
    select 4) nums
        inner join
    (select substring_index(group_concat(product_id order by product_id desc), ',', 400) product_ids, activity_id
    from fresh_product where activity_id in (10000745,10000738, 10000732)
    group by activity_id) t on CHAR_LENGTH(t.product_ids)
                                    - CHAR_LENGTH(REPLACE(t.product_ids, ',', '')) >= nums.n - 1
```
新sql看起来更复杂了，先是group by后group_concat数据，然后把逗号分割的列转换为行。
这种处理方法有一点问题是，group_concat如果太长会截取一部分。如果取的前几个较多，会取不到数据，截取长度可配置。