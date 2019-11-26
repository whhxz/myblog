---
title: Storm学习（一）：认识Storm
date: 2017-08-01 08:58:28
categories: ['Storm学习']
tags: ['Stomr基本概念', 'Storm']
---

### 实时流计算
传统DBMS不是为快速连续地存放单独的数据单元而设计，而且也不支持*持续处理*，而*持续处理*是数据流应用的典型特征。一些方案采用MapReduce来处理实时数据流。但是尽管MapReduce做了实时性改进，也很难稳定地满足应用需求。这是因为**Hadoop MapReduce框架为批处理做了高度优化，典型的是通过调度批量任务来操作静态数据，任务不是常驻服务，数据也不是实时流入。**

#### 实时计算流程

1、数据采集
功能上保证可以完整的收集到所有日志数据，为实时应用提供实时数据；响应时间上要保证实时性、低延迟；配置简单，部署容易；系统稳定可靠等。
目前采集数据工具有：
* Facebook开源的Scribe
* LinkedIn开源的Kafka
* Cloudera开源的Flume
* 淘宝开源的TimeTunnel
* Hadoop的Chukwa等

<!-- more -->
2、数据实时计算
适应流式数据、不间断查询；系统稳定可靠、可扩展性好、可维护性好。需要关注的注意点：** 分布式计算、并行计算、热点数据缓存策略、服务端计算。**

3、实时数据查询
* 全内存：直接提供数据读取、定期转存到磁盘或者数据库中进行持久化；
* 半内存：使用Redis、Memcache、MongoDB、等内存数据库提供数据库实时查询范围，由这些系统进行持久化操作；
* 全磁盘：使用HBase等分布式文件系统（HDFS）为基础的NoSQL数据库，对于KeyValue内存引擎，关键是设计好Key。

### 实时计算框架
* IBM的StreamBase
* Yahoo的S4
    >  设计特点：
    > * Actor计算模型
    > * 对等集群架构
    > * 课插拔体系架构
    > * 支持部分容错
* Twitter的Storm
    >  主要特点：
    > * 简单模型；
    > * 可以使用各种编程语言.默认支持Clojure、Java、Ruby和Python。要增加其他支持，只需要实现一个简单的Storm通信协议；
    > * 容错性、水平扩展、可靠消息处理、快速、本地模式。
* Twitter的Rainbird
* Facebook的Puma
* 阿里的JStorm
* 其他的如：HStreaming、Esper、Borealis

### Storm设计
在Storm中流是有Spout作为源头源源不断产生Tuple，Tuple流向Bolt处理。Spout产生Tuple流向Bolt整体称之为Topology。提交Topology到Storm集群执行，Topology就是一个流转换图。

### Storm核心组件
* 主节点Nimbus
 > 主节点通常运行一个后台查询---Numbus，用于响应发布在集群中的节点，分配任务和监测故障，这类似于Hadoop中的JobTracker。Nimbus进程是快速失败和无状态的，所有的状态要么在ZooKeeper中，要么在本地磁盘上。
* 工作节点Supervisor
 > 工作节点同样会运行一个后台程序---Supervisor，用于收听工作指派并基于要求运行工作进程。每个工作节点都是Topology中一个子集的实现。Nimbus和Supervisor之间协调通过ZooKeeper,Supervisor同Nimbus是无状态的，
* 协调服务组件ZooKeeper
* 其他核心组件
 > * 具体处理事务进程Worker：运行具体处理组件逻辑的进程；
 > * 具体处理现场Task：Worker中每一个Spout/Bolt线程称为一个Task。在Storm 0.8之后，Task不再与物理线程对应，同一个Spoot/Bolt的Task可能会共享一个物理线程，该线程称为Executor

### Storm特性
* 编程模型简单
* 可扩展
* 高可靠性
 > 在Spout发出消息后，可能会触发成千上万条消息，Storm会保证所有消息被处理完成，才认为该Spout消息被完全处理。如果在限定时间内没有完全处理，那么Spout发出的消息就会重发。为了减少内存消耗，Storm不会跟踪所有消息，Storm通过对所有消息的唯一ID进行异或计算，通过是否为0来判断Spout发出的消息是否被处理
* 高容错性
* 支持本地模式
* 支持多种编程语言
* 高效
