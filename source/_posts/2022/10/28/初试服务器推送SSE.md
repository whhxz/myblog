---
title: 初试服务器推送SSE
date: 2022-10-28 16:00:09
categories: 
tags: ['SSE', '服务器推送']
---

前端有个耗时的查询比对任务提交，想实时获取比对进程。一般常规都是轮询、长连接、websocket等，今天查到html5里面有个SSE（Server-sent Events），客户端提交一次请求后，由服务端单向推送数据。不支持一次请求客户端再次通信。
<!--more-->
### 后端
采用已经有的SpringBoot。
```java
@RestController
@RequestMapping("sseDemo")
public class SseDemoController {
    @GetMapping(value = "str", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter str() {
        //设置超时时间，客户端超时后会重新发起连接
        SseEmitter sseEmitter = new SseEmitter(50000L);
        sseEmitter.onTimeout(() -> {
            System.out.println("超时");
        });
        //单独启动线程模拟服务端
        new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    //推送数据
                    sseEmitter.send("data " + i);
                    TimeUnit.SECONDS.sleep(1);
                }
                //自定义事件，告知客户端关闭，如果不告知客户端关闭，会导致浏览器无限重连，使用complete也会导致重连
                SseEmitter.SseEventBuilder event = SseEmitter.event();
                event.name("close");
                //测试用，mediaType用于序列化数据
                event.data(new Object(), MediaType.APPLICATION_JSON_UTF8);
                sseEmitter.send(event);
                //处理完成
                sseEmitter.complete();
            } catch (InterruptedException | IOException e) {
                throw new RuntimeException(e);
            }
        }).start();
        return sseEmitter;
    }
}
```
> 此处未处理id等保证数据获取，实际开发过程中可按需要进行处理

### 前端
```js
let eventSource = new EventSource('/webApi/sseDemo/str');
//默认接收消息
eventSource.onmessage = function (event) {
    console.log(event)
}
//自定义事件
eventSource.addEventListener('close', (event)=>{
    console.log(event)
    eventSource.close()
});
//打开连接后
eventSource.onopen = function (event) {
    console.log(event)
}
```
> 前端如果超时后会默认断开重连，如果未处理好异常情况，会导致一直发送请求。所以需要设定规则关闭连接，如接收到关闭事件，或者多少次重连或者异常后关闭。