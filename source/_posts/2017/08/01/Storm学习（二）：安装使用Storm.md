---
title: Storm学习（二）：安装使用Storm
date: 2017-08-01 10:42:47
categories: ['Storm学习']
tags: ['Storm安装', 'Storm本地调用']
---

## 系统环境需要
* JDK配置好环境变量
* SSH，方便启动操作
* 安装Python，Python是Storm最底层依赖

## Zookeeper安装
Zookeeper官网下载解压，安装参考[Zookeeper安装配置](https://taoistwar.gitbooks.io/spark-operationand-maintenance-management/content/spark_relate_software/zookeeper_install.html)，或者[官方文档](https://zookeeper.apache.org/doc/trunk/zookeeperStarted.html)

## Storm启动安装
### 本地模式
本地模式在一个进程中使用现场模拟Storm集群的所有功能，用于本地开调试。本地模式运行Topology于在集群上运行Topology类似，但是提交Topology任务是在本地机器上。
简单使用LocalCluster类，就能创建一个进程内集群。如：
```java
LocalCluster cluster = new LocalCluster()
```
本地模式下要注意如下参数
* Config.TOPOLOGY_MAX_TASK_PARALLELISM 单组件最大线程数
* Config.TOPOLOGY_DEBUG

\* 参考：[本地模式http://storm.apache.org/releases/1.1.0/Local-mode.html](http://storm.apache.org/releases/1.1.0/Local-mode.html)

### 安装部署集群
下载Storm解压 [storm官网](http://storm.apache.org/index.html)
修改conf/storm.yaml配置文件，storm.yaml中配置会覆盖defaults.yaml中配置。
最基本配置如下
```yaml
storm.zookeeper.servers:
    - "zk1"
    - "zk2"
    - "zk3"
storm.local.dir: "/var/storm"
nimbus.host: "192.168.0.100"
```

### 启动Storm集群
```bash
//启动nimbus
bin/./storm nimbus </dev/null 2<&1 &
//启动Supervisor
bin/./storm supervisor </dev/null 2<&1 &
//启动UI
bin/./storm ui </dev/null 2<&1 &
```
## Storm范例

统计Topology
WordCountTopology.java
```java
public class WordCountTopology {
    public static void main(String[] args) throws InterruptedException {
        TopologyBuilder builder = new TopologyBuilder();
        builder.setSpout("word-spout", new WordSpout(), 1);
        builder.setBolt("word-split", new WordSplitSentenceBolt(), 2).shuffleGrouping("word-spout");
        builder.setBolt("word-count", new WordCountBolt(), 2).fieldsGrouping("word-split", new Fields("word"));
        Config conf = new Config();
        conf.setDebug(true);
        conf.put("wordsFile", "D:\\tmp\\word.txt");
        conf.put(Config.TOPOLOGY_MAX_SPOUT_PENDING, 1);
        LocalCluster cluster = new LocalCluster();
        //storm集群运行
        /*
        StormSubmitter.submitTopology("Getting-Started-Toplogie", conf, builder.createTopology());
        */
        try {
            cluster.submitTopology("Getting-Started-Toplogie", conf, builder.createTopology());
        } catch (Exception e) {
            e.printStackTrace();
            System.exit(0);
        }
        Thread.sleep(10000);
        cluster.shutdown();

    }
}
```
spout输入
WordSpout.java
```java
public class WordSpout extends BaseRichSpout {
    private SpoutOutputCollector collector;
    private FileReader fileReader;
    private boolean completed = false;
    public void ack(Object msgId) {
        System.out.println("OK:"+msgId);
    }
    public void close() {}
    public void fail(Object msgId) {
        System.out.println("FAIL:"+msgId);
    }
    @Override
    public void open(Map conf, TopologyContext context, SpoutOutputCollector collector) {
        try {
            this.fileReader = new FileReader(conf.get("wordsFile").toString());
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        }
        this.collector = collector;
    }

    @Override
    public void nextTuple() {
        if(completed){
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                //Do nothing
            }
            return;
        }
        String str;
        BufferedReader reader = new BufferedReader(fileReader);
        try{
            while((str = reader.readLine()) != null){
                this.collector.emit(new Values(str));
            }
        }catch(Exception e){
            throw new RuntimeException("Error reading tuple",e);
        }finally{
            completed = true;
        }
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("sentence"));
    }
}
```
分割句子
WordSplitSentenceBolt.java
```java
public class WordSplitSentenceBolt extends BaseBasicBolt {

    @Override
    public void execute(Tuple input, BasicOutputCollector collector) {
        String sentence = input.getString(0);
        for (String word : sentence.split(" ")) {
            word = word.trim();
            if (!word.isEmpty()) {
                collector.emit(new Values(word));
            }
        }
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("word"));
    }
}
```
统计单词数量
WordCountBolt.java
```java
class WordCountBolt extends BaseBasicBolt{
    private final static Logger logger = LoggerFactory.getLogger(WordCountBolt.class);
    private Map<String, Integer>counts = new HashMap<>();
    @Override
    public void cleanup() {
        counts.forEach((key, value) -> {
            System.out.println("```````````````````````````````````````");
            logger.error("{}: {}", key, value);
        });
    }

    @Override
    public void execute(Tuple input, BasicOutputCollector collector) {
        String word = input.getString(0);
        Integer count = counts.get(word);
        if (count == null){
            count = 0;
        }
        count++;
        counts.put(word, count);
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
    }
}
```

其他例子可参考：[Storm官方例子](https://github.com/apache/storm/tree/master/examples)

### 向集群提交任务
把项目打成jar包，在storm安装主目录下执行：
```bash
bin/./storm jar demo.jar com.whh.WordCountTopology
```
如果需要停止Topology，在storm目录下执行：
```bash
bin/./storm kill {toponmae}
```
> 其中{toponmae}是Topology提交到Storm集群时指定的Topology任务名称
