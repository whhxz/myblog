---
title: ELK搭建简单日志查询
date: 2018-01-11 13:52:37
categories: ['ELK']
tags: ['日志查询', 'Elasticsearch', 'Logstash', 'Kibana']
---

通常在项目日志查看过程中都是直接登录服务器，找到服务器通过一些命令查看日志。有时候在定位问题的时候比较麻烦，不能快速找到问题日志。

这次尝试在本地搭建简单的ELK查询日志相关信息。

Elasticsearch：用于日志搜索查询
Logstash：用于日志收集
Kibana：用于页面展示
<!-- more -->
1、下载Elasticsearch、因为是简单实用，下载解压后直接使用。
2、下载Kibana，修改config下kibana.yml，添加 **elasticsearch.url: "http://127.0.0.1:9200"**
3、下载Logstash，配置相关配置信息。
4、下载filebeat用于采集日志传送到Logstash、配置采集信息

filebeat配置：
备份原有filebeta.yml文件，新增filebeta.yml文件
```yml
filebeat:
  prospectors:
    -
      paths:
        #采集路径
        - /home/webdata/tomcat/logs/catalina.out
      input_type: log
      #用于日志合并、比如有的日志是异常多行信息，通过下面判断日志是否以时间格式开头，合并到上一行
      multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
      multiline.negate: true
      multiline.match: after
      # 增加字段、用来标示日志来源，在采集多个项目日志时有用
      fields:
        server_name: storemanager-api
      fields_under_root: true
output:
  logstash:
      #本机Logstash地址
      hosts: ["10.211.240.162:4560"]
```
filebeat其他配置可以参考：[Filebeat安装部署及配置详解](https://www.qcloud.com/community/article/268720)

Logstash配置：
新增文件tomcat_log4j.conf
```java
# 开放输入端口
input{
  beats {
    host => "0.0.0.0"
    port => 4560
  }
}
filter {
  #判断日志来源、通过来源分开处理
  if [server_name] == "manager-api"{
    #处理日志
    grok {
      #lo4j日志配置为：log4j.appender.logstash.layout.ConversionPattern=%d [%t] %-5p [%c] - %m%n
      # 需要对日志进行拆分
      match => {
        "message" => "(?<timestamp>[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2},[0-9]{3}) \[(?<thread_name>.*)\] (?<log_level>.*) \[(?<class_name>.*)\] - (?<message>.*)"
      }
      overwrite => [ "message" ]
    }
    date {
      match => ['logtime', "yyyy-MM-dd HH:mm:ss,SSS"]
      locale => "cn"
    }
  }
}
#日志输出
output {
  stdout{
    codec => rubydebug
  }
  #实际情况是需要依据日志来源建立不同的索引，不能全部混在一起
  elasticsearch{
    hosts =>["127.0.0.1:9200"]
  }
}
```
分别启动4个服务、启动Logstash需要使用：**./bin/logstash -f tomcat_log4j.conf** 指定配置文件。
之后打开浏览器localhost:5601查看日志信息。

后续如果需要使用，可以控制权限之类，还有很多玩法。
在使用grok写表达式时，可以通过*http://grokdebug.herokuapp.com/* 测试表达式

参考：
* [Logstash 最佳实践](https://doc.yonyoucloud.com/doc/logstash-best-practice-cn/index.html)
* [ELKstack 中文指南](https://elkguide.elasticsearch.cn/)
* [ELK技术实战-安装Elk 5.x平台](http://www.ywnds.com/?p=9776)
