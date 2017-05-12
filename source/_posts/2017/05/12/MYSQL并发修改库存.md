---
title: MYSQL并发修改库存
date: 2017-05-12 18:05:26
categories: MYSQL
tags: ['MYSQL', '并发修改']
---


## MYSQL插入或更新
> 问题：在数据库操作时，如果需要先判断数据是否存在，做插入或者更新操作。在并发情况下，容易出现异常。

解决办法：使用`INSERT INTO TABLE(A, B, C) VALUES('a', 'b', 'c') ON DUPLICATE KEY UPDATE B = 'b', C='c'`，此处A为唯一约束，在插入时，如果已经存在数据为a，则做更新操作。

## 并发修改数量
> 在并发扣除库存时，库存数量不能扣到小于0。

使用`UPDATE TABLE SET A = A - a WHERE A >= a AND ID = 'xx'`，通过SQL返回值是否大于0，判断是否操作成功。
