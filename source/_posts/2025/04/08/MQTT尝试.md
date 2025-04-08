---
title: MQTT尝试
date: 2025-04-08 09:20:22
categories:
tags: ["MQTT", "EMQX"]
---

### 服务端启动
使用docker启动服务端测试
```bash
docker pull emqx:5.8.6
docker run --rm -d --name emqx -p 1883:1883 -p 8083:8083 -p 8084:8084 -p 8883:8883 -p 18083:18083 emqx:5.8.6
```
登录后台 **http://localhost:18083** 账号:admin 密码:public
<!-- more -->
### 客户端使用
添加相关依赖
```xml
<dependency>
    <groupId>org.eclipse.paho</groupId>
    <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
    <version>1.2.2</version>
</dependency>
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-core</artifactId>
</dependency>
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
</dependency>
```
#### 订阅者

```java
static class Subscribe implements Runnable, MqttCallback {
    private Logger log;
    //通配符订阅,每个订阅者都会收到
    private String topic = "testtopic/#";
    //共享订阅 只有一个订阅者会收到
//        private String topic = "$queue/testtopic/#";
    private MqttClient client;

    public Subscribe(String name, String broker) throws MqttException {
        log = LoggerFactory.getLogger(name);
        client = new MqttClient(broker, name, new MemoryPersistence());
    }

    @Override
    public void run() {
        try {
            client.setCallback(this);
            client.connect();
            client.subscribe(topic, 0);
        } catch (MqttException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public void connectionLost(Throwable throwable) {
        log.error("connection lost");
    }

    @Override
    public void messageArrived(String s, MqttMessage mqttMessage) {
        log.info("msg:{} {}", s, new String(mqttMessage.getPayload()));
    }

    @Override
    public void deliveryComplete(IMqttDeliveryToken iMqttDeliveryToken) {
        log.info("deliveryComplete:{}", iMqttDeliveryToken.isComplete());
    }
}
```
#### 发布者
```java
static class Publish extends Thread {
    private Logger log;
    private String topic;
    private MqttClient client;
    private String name;

    public Publish(String name, String broker, String topic) throws MqttException {
        this.topic = topic;
        this.name = name;
        log = LoggerFactory.getLogger(name);
        client = new MqttClient(broker, name, new MemoryPersistence());
    }

    @Override
    public void run() {
        try {
            client.connect();
            while (true) {
                String content = "i'am " + name + new Random().nextInt(10000);
                log.info("send smg:{}", content);
                MqttMessage message = new MqttMessage(content.getBytes());
                client.publish(topic, message);
                TimeUnit.MILLISECONDS.sleep(3000 + new Random().nextInt(1000));
            }
        } catch (MqttException | InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}
```


项目启动
```java
String broker = "tcp://192.168.88.1:1883";

Subscribe sub1 = new Subscribe("sub-1", broker);
sub1.run();
Subscribe sub2 = new Subscribe("sub-2", broker);
sub2.run();


Publish pub1 = new Publish("pub-1", broker, "testtopic/1");
Publish pub2 = new Publish("pub-2", broker, "testtopic/2");

pub1.start();
pub2.start();
```

[代码示例地址]: https://github.com/whhxz/blog-example/tree/main/mqtt-client