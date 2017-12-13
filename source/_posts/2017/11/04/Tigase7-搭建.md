---
title: Tigase7 搭建
date: 2017-11-04 14:41:59
categories: ['基础搭建']
tags: ['tigase', '基础搭建']
---

因为企业内部需要使用IM，现通Tigase+Spark搭建初始项目。

### 下载源码
访问Tigase官网，为了二次开发，下载Tigase源码。
下载地址：https://tigase.tech/projects/tigase-server/repository
```shell
#下载源码
git clone https://git.tigase.tech/tigase-server.git
cd tigase-server
#切换到最新tag
git checkout tigase-server-7.1.2
```
<!-- more -->
### 通过命令配置启动
采用Mysql作为Tigase的数据库

#### 初始化Mysql数据库
1. 在本地数据库中建立数据库tigasedb
2. 登陆mysql
```shell
mysql -r root -ppassword
```
3. 初始化数据库
```sql
source database/mysql-schema-7-1.sql
```

#### 修改Tigase配置文件
修改etc/init-mysql.properties配置文件
```properties
config-type=--gen-config-def
--admins=admin@localhost
--user-db=mysql
--user-db-uri=jdbc:mysql://localhost/tigasedb?user=root&password=password
--virt-hosts=localhost
--debug=server
--comp-name-1=http
--comp-class-1=tigase.http.HttpMessageReceiver
--comp-name-2 = muc
--comp-class-2 = tigase.muc.MUCComponent

muc/room-log-directory=logs/muc/
muc/search-ghosts-every-minute[B]=true
muc/muc-allow-chat-states[B]=true
muc/muc-lock-new-room[B]=false
muc/history-db=none
```
#### 编译配置Tigase
1. 配置Tigase maven仓库：
```XML
<repositories>
    <repository>
        <id>tigase</id>
        <name>Tigase repository</name>
        <url>http://maven-repo.tigase.org/repository/release</url>
    </repository>
    <repository>
        <id>tigase-snapshot</id>
        <name>Tigase repository</name>
        <url>http://maven-repo.tigase.org/repository/snapshot</url>
        <snapshots>
            <enabled>true</enabled>
        </snapshots>
    </repository>
</repositories>
```
2. 编译Tigase
```shell
mvn -Pdist -f modules/master/pom.xml clean install
```
如果需要生成安装包程序，需要执行下面脚本，如果不需要可以不执行：
```shell
./scripts/installer-prepare.sh
./scripts/installer-generate.sh
```
在执行shell脚本时，本机上需要安装git、ant、python2,、docutils、LaTeX，否则会报错。

#### 命令启动Tigase
```shell
 ./scripts/tigase.sh start etc/tigase-mysql.conf
```

#### Spark配置登陆聊天
Spark下载地址：https://igniterealtime.org/projects/spark/
因为使用的mac，需要启动多个spark，可以通过命令 **open -na spark** 启动。
点击高级配置Spark，如图：
![](http://otxnth5wx.bkt.clouddn.com/20171104屏幕快照2017-11-04下午3.04.00.png)
通过Spark注册账号，如图：
![](http://otxnth5wx.bkt.clouddn.com/20171104屏幕快照2017-11-04下午3.04.49.png)
登陆Spark，多开后可以通过不同账号聊天，也可以通过会议群聊。


### Idea启动Tigase
在项目中XMPPServer是启动的入口，需要配置XMPPServer启动。
配置如图：
![](http://otxnth5wx.bkt.clouddn.com/20171104QQ20171104-152648@2x.png)
配置参数如下：
```
VM option：
-Dfile.encoding=UTF-8
-Dsun.jnu.encoding=UTF-8
-Djdbc.drivers=com.mysql.jdbc.Driver
-Djava.ext.dirs=/Users/你的路径/tigase-server/jars
-server
-Xms100M
-Xmx200M
-XX:PermSize=32m
-XX:MaxPermSize=256m
-XX:MaxDirectMemorySize=128m

program arguments：
--property-file etc/init-mysql.properties

Working directory：你的项目路径
```
启动XMPPServer服务，提供Spark测试服务是否正常。
