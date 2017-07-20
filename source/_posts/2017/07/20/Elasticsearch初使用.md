---
title: Elasticsearch初使用
date: 2017-07-20 21:46:18
categories: ['搜索','Elasticsearch']
tags: ['Elasticsearch', 'Kibana', 'Logstash', 'mysql搜索']
---

---
### 下载安装
通过Elasticsearch搜索数据库，需要下载Elasticsearch、Kibana、Logstash。（当前使用版本5.0.0）
Elasticsearch下载：https://www.elastic.co/cn/products/elasticsearch
Kibana下载：https://www.elastic.co/cn/products/kibana
Logstash下载：https://www.elastic.co/cn/products/logstash

### 启动服务
**启动elasticsearch**
``` 
cd elasticsearch
./bin/./elasticsearch 
```
> 如果需要其他机器访问需要修改配置文件./config/elasticearch.yml中network.host

** 启动kibana **
```
cd kibana
./bin/./kibana
```
> 如果需要其他机器访问kibana，需要修改配置文件./config/kibana.yml中server.host
在第一启动kibana时要配置日志

elasticsearch默认访问http://ip:9200
kibana默认访问http://ip:5601

### 配置Logstash同步数据库
> 同步mysql到elasticsearch，使用Logstash中的插件*jdbc input plugin*。参考https://www.elastic.co/guide/en/logstash/current/plugins-inputs-jdbc.html

创建导入配置文件 simple_imput.conf
```
input {
    jdbc {
        # Postgres jdbc connection string to our database, mydb
        jdbc_connection_string => "jdbc:mysql://localhost:3306/comment?characterEncoding=utf-8&useUnicode=true&zeroDateTimeBehavior=convertToNull&allowMultiQueries=true"
        # The user we wish to execute our statement as
        jdbc_user => "user"
	jdbc_password => "password"
	jdbc_validate_connection => true
        # The path to our downloaded jdbc driver
        jdbc_driver_library => "./mysql-connector-java-5.1.31.jar"
        # The name of the driver class for Postgresql
        jdbc_driver_class => "com.mysql.jdbc.Driver"
        # our query
        jdbc_paging_enabled => "true"
	jdbc_page_size => "50000"
	type => "comment_bad_cause_type"
	# statement_filepath => "/usr/local/logstash/bin/logstash_jdbc_test/jdbc.sql"
	# 表示每分钟同步更新一次
	schedule => "* * * * *"
        statement => "SELECT * FROM test_table where update_time >= :sql_last_value"
	jdbc_default_timezone => "Asia/Shanghai"
    }
}
output {
    stdout { codec => json_lines }
    elasticsearch {
	index => "demo_index"
	document_id => "%{id}"
    }  
}
```
启动Logstash开始导入数据库
```./bin/logstash -f simple_input.conf```
> 需要注意的是，sql中为增量导入。在第一次的时候应该是全量导入，这个可能需要先去掉查询条件全量导入（注释schedule），后重新运行增量导入。可能使用两个input一个全量，一个增量（未尝试），后期尝试下。

打开kibana可以尝试搜索数据

### 参考
* [买好车搜索的Elasticsearch实践：初体验](http://suclogger.com/%E4%B9%B0%E5%A5%BD%E8%BD%A6%E6%90%9C%E7%B4%A2%E7%9A%84Elasticsearch%E5%AE%9E%E8%B7%B5%EF%BC%9A%E5%88%9D%E4%BD%93%E9%AA%8C/)

