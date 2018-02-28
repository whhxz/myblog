---
title: Netty TCP粘包 拆包
date: 2018-02-26 17:04:26
categories: ['netty']
tags: ['netty', 'tcp', '粘包', '拆包']
---

在平常使用tcp连接时，可能会出现tcp粘包的现象。
在客户端与服务端建立连接后，客户端提交数据后，如果发送的数据较大，该数据包会被拆分为几个小的包进行发送，如果数据比较小，可能会等待一会，把后续的多个小包封装成一个大点的包一起发送。服务端获取客户端的数据可能出现如下情况：
服务端发送A、B两个包
> * A、B分开发送，服务端正常分两次收到两个包
> * A、B被一起发送、服务端一次收到两个包AB
> * A被拆分为A1、A2、可能出现服务端收到A1、A2B（或者B被拆开，B拆开的包和A粘在一起，或者AB都被拆开组合）

除了第一种是正常的，后面两种都会出现问题。
<!-- more -->

### Netty使用
maven项目：
```xml
<!-- pom.xml -->
<dependency>
  <groupId>io.netty</groupId>
  <artifactId>netty-all</artifactId>
  <version>4.0.23.Final</version>
</dependency>
```
```java
public class TimeServer {
    public static void main(String[] args) throws InterruptedException {
        new TimeServer().bind(9999);
    }

    public void bind(int port) throws InterruptedException {
        //用于接受客户端连接
        NioEventLoopGroup bossGroup = new NioEventLoopGroup();
        //用于网络读写
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 1024)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new TimeServerHandler());
                        }
                    });
            ChannelFuture channelFuture = bootstrap.bind(port).sync();
            channelFuture.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
public class TimeServerHandler extends ChannelInboundHandlerAdapter {
    private int count;
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        ByteBuf buf = (ByteBuf) msg;
        byte[] req = new byte[buf.readableBytes()];
        buf.readBytes(req);
        String request = new String(req, "utf-8");
        System.out.printf("The time server receive other: %s ,count is %d\n", request, ++count);
        String currentTime = "QUERY TIME".equalsIgnoreCase(request) ? format.format(new Date()) : "ERROR";
        ByteBuf resp = Unpooled.copiedBuffer(currentTime.getBytes());
        //写消息给客户端
        ctx.write(resp);
    }

    //channel读取完毕后
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        //将消息发送队列中的消息写入socketChannel中
        //为了频繁唤醒Selector进行消息发送，write后"，并不直接将消息写入socketChannel
        ctx.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
public class TimeClient {
    public static void main(String[] args) throws InterruptedException {
        new TimeClient().connect("127.0.0.1", 9999);
    }

    public void connect(String host, int port) throws InterruptedException {
        EventLoopGroup group = new NioEventLoopGroup();
        Bootstrap bootstrap = new Bootstrap();
        try {
            bootstrap.group(group).channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new TimeClientHandler());
                        }
                    });
            ChannelFuture channelFuture = bootstrap.connect(host, port).sync();
            channelFuture.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully();
        }
    }
}
public class TimeClientHandler extends ChannelInboundHandlerAdapter{
    private final ByteBuf firstMessage;

    public TimeClientHandler() {
        byte[] req = "QUERY TIME".getBytes();
        firstMessage = Unpooled.buffer(req.length);
        firstMessage.writeBytes(req);
    }

    //客户端和服务端建立连接成功后调用
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(firstMessage);
    }

    int count;
    //服务端返回时调用
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf buf = (ByteBuf) msg;
        byte[] req = new byte[buf.readableBytes()];
        buf.readBytes(req);
        String body = new String(req, "utf-8");
        System.out.printf("Now is %s count is %d\n", body, ++count);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.close();
    }
}
```
正常启动Server后启动Client，不会出现问题。

#### 修改Client出现粘包
修改TimeClientHandler.channelActive
```java
ByteBuf message;
for (int i = 0; i < 100; i++) {
    byte[] req = "QUERY TIME".getBytes();
    message = Unpooled.buffer(req.length);
    message.writeBytes(req);
    ctx.writeAndFlush(message);
}
```
重新启动客户端，会发现服务端打印的数据有问题，客户端打印的返回值也有问题。这样一位部分数据被粘在一起了，服务端返回给客户端同理。

一位对于小包而言，在发送数据的时候，可能会等待后续包一起发送，有个等待时间，如果客户端发送数据后设置等待时间会出现什么情况，如下：
```java
// 修改TimeClientHandler.channelActive
ByteBuf message;
for (int i = 0; i < 100; i++) {
    byte[] req = "QUERY TIME".getBytes();
    message = Unpooled.buffer(req.length);
    message.writeBytes(req);
    ctx.writeAndFlush(message);
    TimeUnit.MILLISECONDS.sleep(100);
}
```
重新启动客户端，发现服务端获取的数据正常了，但是出现了客户端收到的服务端返回的数据时异常的，因为我们在Server代码中设置了ChannelOption.SO_BACKLOG为1024，可以打印客户端第一次获取的返回数据长度，刚刚好是1024。

#### 解决出现粘包的情况
如果不使用netty的粘包解决办法需要自己手动设置ltv(length, type, value)，在发送消息时，在包的头部加入包长度，包类型，后面才是具体的数据。这样在获取tcp包时通过包头部去获取后续包的内容。或者在不同的消息间加入特殊分隔符（入http使用的是/r/n/r/n）

通过LineBasedFrameDecoder解决
修改TimeServer.bind中bootstrap.childHandler中匿名内部类代码：
```java
@Override
protected void initChannel(SocketChannel ch) throws Exception {
  //添加LineBasedFrameDecoder、StringDecoder、StringEncoder
    ch.pipeline().addLast(new LineBasedFrameDecoder(1024))
            .addLast(new StringDecoder())
            .addLast(new StringEncoder())
            .addLast(new TimeServerHandler());
}
```
修改TimeServerHandler.channelRead：
```java
SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
System.out.printf("The time server receive other: %s ,count is %d\n", msg, ++count);
String currentTime = "QUERY TIME".equalsIgnoreCase((String)msg) ? format.format(new Date()) : "ERROR";
//写消息给客户端，自己发送String，在后面添加分隔符
ctx.write(currentTime + System.getProperty("line.separator"));
```
修改TimeClient.connect中bootstrap.childHandler中匿名内部类代码：
```java
ch.pipeline().addLast(new LineBasedFrameDecoder(1024))
        .addLast(new StringDecoder())
        .addLast(new StringEncoder())
        .addLast(new TimeClientHandler());
```
修改TimeClientHandler.channelActive：
```java
for (int i = 0; i < 100; i++) {
    //直接发送String+分隔符
    ctx.writeAndFlush("QUERY TIME" + System.getProperty("line.separator"));
}
```
修改TimeClientHandler.channelRead：
```java
直接输出
System.out.printf("Now is %s count is %d\n", msg, ++count);
```
启动服务端、客户端，访问正常。
* 这里需要注意的我们设置的LineBasedFrameDecoder接受的长度是1024，如果超过这个长度会报错。

这里通过LineBasedFrameDecoder、StringDecoder、StringEncoder组合就是按行来做粘包拆包处理，在发送的字符串末尾添加换行符。

除了使用LineBasedFrameDecoder解决之前的问题，在Netty中还提供了两种方案：
1、DelimiterBasedFrameDecoder：自动完成以分隔符作为码流结束标识的消息解码。
2、FixedLengthFrameDecoder：固定长度解码器，按照指定长度对消息进行自动解码。

对于DelimiterBasedFrameDecoder而言如果在设定的长度下还没有获取到分隔符一样会抛出异常，避免因为异常码流导致缺失分隔符。
对于FixedLengthFrameDecoder而言，如果是半包消息，FixedLengthFrameDecoder会缓存半包消息并等待下一个包到达后进行拼包，直到读取到一个完整的包。

* 参考：《Netty权威指南》
