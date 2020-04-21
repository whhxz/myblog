---
title: redis使用lua脚本批量操作
date: 2020-04-21 22:43:34
categories: ["线上BUG"]
tags: ["redis"]
---

前言：本来项目中使用的redis采用的是集群模式，之后改为了哨兵模式。
今天在缓存平台上查看缓存，发现命中率非常低，缓存中key非常少，之前集群模式应该有即使上百万的key，现在就几百。几番排查后发现在集群模式下使用了`mset`可以正常批量写入数据，之后通过`mexpire`批量设置失效时间。改为哨兵模式后，`mset`无法使用。
业务中如果缓存没命中会直接查询数据库，所有在测试时并未发现什么问题。

确实哨兵模式下，也有对应的`mset`，但是该模式下没有批量设置失效时间，只能使用lua脚本对数据进行操作。
<!--more-->
### 添加脚本到redis服务器
使用方法`scriptLoad`传入脚本和key，key不变方便后续直接使用。
在启动项目时即可传入脚本，获取返回的sha(用于唯一标识脚本，如果传入的key一样，返回的值不会变，方便后续使用，脚本如果上传成功后会永久保存在服务器)，保存改sha用于后续调用该脚本。
批量写入数据同时设置失效时间脚本如下：
```lua
for index, value in ipairs(KEYS)
    do
        redis.call('SETEX', KEYS[index], ARGV[index*2-1], ARGV[index*2])
    end
return 1
```
* 脚本中`KEYS`用于获取传入的key，脚本中`ARGV`用于获取传入的参数

循环遍历传入的`KEYS`，调用命令`SETEX`设置该key的值以及key的失效时间。
### 调用脚本
方法`evalsha(final String sha1, final List<String> keys, final List<String> args)`调用脚本。
第一个参数为之前上传lua脚本返回的sha，第二个为需要操作的key，第三个为传入的参数(失效时间和值交替放入list)

* 缓存平台使用的开源sohutv/cachecloud
* 集群模式下`mset`为`String mset(final Map<String, String> keyValueMap)`，哨兵模式下为`String mset(String... keysvalues)`key和val交替。


