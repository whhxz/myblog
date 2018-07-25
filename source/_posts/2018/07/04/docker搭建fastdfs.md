---
title: docker搭建fastdfs
date: 2018-07-04 11:07:48
categories: ['docker', '分布式']
tags: ['docker', 'fastdfs', '图片服务器']
---

使用fastdfs搭建文件服务器，用于存储图片。通过docker虚拟服务完成分布式系统的搭建。

### 构建基础镜像
为了方便服务的启动以及后续的配置，需要先创建基本的fastdfs镜像。
<!-- more -->
创建如下目录结构：[https://github.com/whhxz/soft-docker/tree/master/fastdfs](https://github.com/whhxz/soft-docker/tree/master/fastdfs)
```
.
├── conf    //存储配置文件
│   ├── client.conf
│   ├── fastdfs.conf
│   ├── http.conf
│   ├── mime.types
│   ├── mod_fastdfs.conf
│   ├── nginx.conf
│   ├── storage.conf
│   └── tracker.conf
├── data    //volumes目录
│   ├── storage
│   ├── storage1
│   └── tracker
├── docker-compose.yml
├── Dockerfile
├── init.sh //启动
└── soft //需要安装的软件
    ├── fastdfs //https://github.com/happyfish100/fastdfs.git --depth 1
    ├── fastdfs-nginx-module //https://github.com/happyfish100/fastdfs-nginx-module.git --depth 1
    ├── libfastcommon   //https://github.com/happyfish100/libfastcommon.git --depth 1
    └── nginx-1.12.2    //http://nginx.org/download/nginx-1.12.2.tar.gz
```
如上目录，先下载必须的安装软件，然后编写构建镜像时需要执行的脚本
```sh
#切换阿里云镜像
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum makecache
#安装必须依赖
yum install git gcc gcc-c++ make automake autoconf libtool pcre pcre-devel zlib zlib-devel openssl-devel -y
#创建工作目录
mkdir -p /fastdfs/tracker
mkdir -p /fastdfs/storage
#遍历项目
cd /usr/local/src
cd libfastcommon/
./make.sh && ./make.sh install
cd ../fastdfs/
./make.sh && ./make.sh install
cd ../nginx-1.12.2/
./configure --add-module=/usr/local/src/fastdfs-nginx-module/src
make && make install
```

创建`Dockerfile`构建镜像
```Dockerfile
from centos
COPY ./soft ./init.sh /usr/local/src/
WORKDIR /usr/local/src/
RUN sh init.sh
EXPOSE 80
CMD ["/usr/local/nginx/sbin/./nginx", "-g", "daemon off;"]
```
使用命令`docker build -t fastdfs --rm=true .`构建镜像

### 配置启动服务
创建镜像之后，通过`docker-compase`配置启动服务
```yml
version: '2'
services:
    #tracker服务
  fdfs_tracker:
    image: fastdfs
    container_name: fdfs_tracker
    volumes:
      - ./conf/tracker.conf:/etc/fdfs/tracker.conf
      #- ./conf/storage.conf:/etc/fdfs/storage.conf
      #- ./conf/client.conf:/etc/fdfs/client.conf
      #- ./conf/http.conf:/etc/fdfs/http.conf
      #- ./conf/mime.types:/etc/fdfs/mime.types
      #- ./conf/mod_fastdfs.conf:/etc/fdfs/mod_fastdfs.conf
      #- ./conf/nginx.conf:/usr/local/nginx/conf/nginx.conf
      - ./data/tracker:/fastdfs/tracker
      #- ./data/storage:/fastdfs/storage
    ports:
      - 22122:22122
    command: bash -c "/etc/init.d/fdfs_trackerd start && tail -f /fastdfs/tracker/logs/trackerd.log"
  #storage服务
  fdfs_storage:
    image: fastdfs
    container_name: fdfs_storage
    volumes:
      #- ./conf/tracker.conf:/etc/fdfs/tracker.conf
      - ./conf/storage.conf:/etc/fdfs/storage.conf
      - ./conf/client.conf:/etc/fdfs/client.conf
      - ./conf/http.conf:/etc/fdfs/http.conf
      - ./conf/mime.types:/etc/fdfs/mime.types
      - ./conf/mod_fastdfs.conf:/etc/fdfs/mod_fastdfs.conf
      - ./conf/nginx.conf:/usr/local/nginx/conf/nginx.conf
      #- ./data/tracker:/fastdfs/tracker
      - ./data/storage:/fastdfs/storage
    links:
      - fdfs_tracker
    ports:
      - 23000:23000
      - 9898:9898
    command: bash -c "/etc/init.d/fdfs_storaged start && /usr/local/nginx/sbin/./nginx &&  tail -f /fastdfs/storage/logs/storaged.log"
  #storage服务
  fdfs_storage1:
    image: fastdfs
    container_name: fdfs_storage1
    volumes:
      #- ./conf/tracker.conf:/etc/fdfs/tracker.conf
      - ./conf/storage.conf:/etc/fdfs/storage.conf
      - ./conf/client.conf:/etc/fdfs/client.conf
      - ./conf/http.conf:/etc/fdfs/http.conf
      - ./conf/mime.types:/etc/fdfs/mime.types
      - ./conf/mod_fastdfs.conf:/etc/fdfs/mod_fastdfs.conf
      - ./conf/nginx.conf:/usr/local/nginx/conf/nginx.conf
      #- ./data/tracker:/fastdfs/tracker
      - ./data/storage1:/fastdfs/storage
    links:
      - fdfs_tracker
    ports:
      - 23001:23000
      - 9899:9898
    command: bash -c "/etc/init.d/fdfs_storaged start && /usr/local/nginx/sbin/./nginx && tail -f /fastdfs/storage/logs/storaged.log"
```
在conf中的配置文件可以查看[fastdfs Wiki](https://github.com/happyfish100/fastdfs/wiki)
* 在配置tracker的ip时需要注意的是，在单机使用时，不能配置localhost，需要配置具体的ip。

通过命令`docker-compose up`启动服务，之后可以通过客户端上传文件。之后在浏览器中直接访问。

在使用Java客户端上传文件时（其他客户端为尝试），需要`StorageServer`这个默认是连接`TrackerClient`之后获取storage的地址，但是因为使用的是docker，返回的ip是docker内部ip，这样就会导致上传失败，可以手动创建`StorageServer`对象，ip填写主机地址以及映射端口，暂时未想到其他解决方案。

因为使用的是多个storage服务，在指定其中一个上传后，在另外的服务器地址也可以照样访问。

* 在启动时，可能会出现`tail -f`命令的错误，因为可能文件未生成，可以先修改为其他文件，之后在改回来，使用`tail -f`目的是为了保持docker持续的运行