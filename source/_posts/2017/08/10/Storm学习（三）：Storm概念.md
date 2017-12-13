---
title: Storm学习（三）：Storm概念
date: 2017-08-10 16:00:22
categories: ['Storm学习']
tags: ['Storm概念', 'Storm']
---

对于Storm，其核心概念有包括：
> * Stream 数据流
> * Spout 数据流
> * Bolt 处理数据
> * Tuple 数据单元
> * Task 运行Spout或Bolt中的线程
> * Worker 是运行这些线程中的进程
> * Stream Grouping 规定了Bolt接受何种数据类型作为输入
> * Topology 是Stream Grouping连接起来的Spout和Bolt节点网络

### Tuple元组
> Tuple 是Storm中使用的最基本单元、数据模型和元组。Tuple就是一个值列表，Tuple默认值类型有：integer、float、double、long、short、string、byte、binary[byte[]]。如果需要使用其他类型，需要序列化该类型。

<!-- more -->
Tuple可以理解为键值对，键为declareOutputFields中的Fields，值为emit发送的Values。
Tuple由Spout中的nextTuple来获取下一个Tuple，通过emit将生成的Tuple发送出去。

### Spout数据源
Spout最源头的接口为IComponent，由Spout发出的Tuple，可以是可靠的，可以是不可靠的。可靠的Tuple会保证被Storm成功处理。处理失败后会重新发送。一个Spout可以发射多个流。在使用Spout时可以实现IRichSpout或者继承BaseRichSpout。
在Spout.declareOutputFields声明发送的Tuple包含的字段。

### Bolt
Bolt属于被动处理，元组输入Bolt，然后产生新的Tuple。
Bolt流程如下：
创建Bolt -> 序列化Bolt -> 提交集群开启Worker进程处理 -> 反序列化Bolt -> 调用prepare处理Tuple

在Bolt处理完成后，如果Tuple是可靠的，需要调用ack，反馈调用成功。在使用IBasicBolt时，会自动调用ack，在BasicBoltExecutor.execute中可以看到调用完成后会调用ack。

在使用Bolt中，常用类为BaseRichBolt、BaseBasicBolt，对于BaseBasicBolt继承IBasicBolt，同样处理完成后会自动调ack。BaseRichBot在处理完成后，如果Tuple是可靠的，需要手动调用ack。在可靠Tuple中，如果不是调用ack，因为Storm会一直追踪每个Tuple，需要中内存，长时间运行会导致OOM。

在方法declareOutputFields中声明当前Bolt发送的字段。同样使用declareStream方法来定义流，之后使用emit选择需要发送的流。

在Bolt使用emit时，可以通过传递参数streamId，来控制新的Tuple是否后旧的Tuple属于同一颗Tuple树。

### Topology
在Storm中Topology是指类似于网络拓扑图的一种虚拟结构。一个拓扑图有Spout和Bolt组成，Spout和Bolt之间通过流分组连接起来。在启动Topology时，TopologyBuilder实际上是封装了Topology的Thrift接口，Topology实际上是Thrift定义的一个结构，Nimbus实际上运行的是一个Thrift服务器，用于接受用户提交的结构。因为使用Thrift实现，用户可以使用其他语言建立Topology。在提交Topology给Nimbus时，也会提交Topology代码。Nimbus负责分发代码和给Topology分配工作进程.如果一个工作进程挂掉后,Nimbus节点会重新分配到其他节点。

#### Topology运行流程
1、提交Topology后，Storm会把代码存放发到Nimbus节点的inbox目录下；之后把当前配置生成stormconf.ser文件放到Nimbus节点的stormdis目录中，改目录中还有序列化后的Topology文件。
2、设定Topology关联的Spout和Bolt，可以同时随着Spout和Bolt的Executor和Task数量，一个默认Topology的Task总和与Executor一致。之后根据Worker的数量，将这些Task分配到不同的Worder上执行。
3、分配任务后，Nimbus节点将任务信息提交到ZooKeeper集群，同时在ZooKeeper集群中有Workerbeats，这里存储了当前Topology所有Worker心跳
4、Supervisor节点轮询ZooKeeper集群，在ZooKeeper中assignments中保存了所有Topology任务分配信息、代码存储目录、任务之间关联关系等，Supervisor通过轮询改节点领取自己任务，启动Worker进程运行
5、Topology运行后，不断通过Spout来发生流，除非手动结束Topology。

#### Topology方法调用流程
1、在Spout或者Bolt中，declareOutputFields方法和构造方法只被调用一次
2、open和prepare调用多次，次数由设置运行组件的Task线程数量。
3、nextTuple和execute一直运行。
4、提交Topology后，Storm创建Spout/Bolt实例并进行序列化，之后将序列化组件发送给所有任务节点，在每个任务节点进行反序列化。
5、组件之间通信使用ZeroMQ

#### Topology执行单元概念
> * Worker对Topology中每个组件运行一个或者多个Executor线程来提供Task的执行服务，一个Supervisor中有多个worker，一个worker属于一个特定的Topology
> * Executor属于Worker进程内线程
> * Task，实际处理数据。在Topology生命周期中Task数量不会发生变化。

> worker 在yaml中的topology.workers属性设置，或者config中setNumWorkers
> Executor 在setBolt或者setSpout中设置
> Task 默认和Executor一致，在TopologyBuilder.setNumTasks设置
> 停止Topology storm kill sopologyName

### Stream消息流
stream消息流是一个抽象概念，主要是消息中的Tuple。

### Stream Grouping消息流组
Stream Grouping消息流组定义一个流如何分配Tuple到Bolt。Storm包含6中流分组类型。
1、随机分组：随机分发Tuple到Bolt任务，保证每个任务获取相同的Tople。
2、字段分组：指定字段分割数据流并分组，相同的字段分到相同的Bolt，不同字段分配到不同的Bolt。
3、全部分组：对于每个Tuple来说，所有Bolt都会收到，所有Tuple复制到Bolt的所有任务上，需谨慎使用。
4、全局分组：全部流都分配到Bolt的同一个任务，就是分配到ID最小的Task上。
5、无分组：目前等效于随机分组
6、直接分组：Tuple生产者决定Tuple消费者任务接收。改分组仅被声明为direct stream的流使用。Tuple必须通过emitDirect直接发射。

### 事务
事务为了解决Tuple在处理失败后重新发送提出.事务拓扑是指Storm以并行和顺序混合的方式处理Tuple，为了实现对消息的精确处理。
事务拓扑的目的是满足对消息处理有着极其严格要求的场景，要求结果完全精确。可以通过修改transactional.zookeeper中server和port指定其他ZooKeeper。

### 数据流模型
数据流处理如下
定义数据 -> 定义数据处理 -> 定义数据节点处理 -> 定义数据处理任务实例 -> 读取数据源头产生的数据 -> 节点上运行任务实例 -> 下一个节点运行任务实例 -> ... -> 输出结果
