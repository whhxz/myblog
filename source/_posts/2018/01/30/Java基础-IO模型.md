---
title: Java基础-IO模型
date: 2018-01-30 20:33:28
categories: ['Java基础', 'NIO', 'Netty']
tags: ['NIO', 'Netty', 'AIO', 'BIO', 'IO', '基础']
---

Unix网络编程中对I/O模型做了5种分类：阻塞I/O模型、非阻塞I/O模型、I/O复用模型、信号驱动I/O模型、异步I/O模型。
<!-- more -->
**阻塞I/O模型** ：在缺省模式下，所有文件存在都是阻塞的。在嵌套字中，在进程空间中调用recvfrom，其系统调用直到数据包到达被复制到应用进程的缓冲区中或者发生错误时才返回，在此期间会一直等待，进程从调用recvfrom开始到它返回的整段时间内都是被阻塞的。
图：
![](/images/old/2018013011043-6296a0fe7e80353d.jpeg)
* recvfrom：本函数用于从（已连接）套接口上接收数据，并捕获数据发送源的地址，成功则返回接收到的字符数，失败则返回-1，错误原因存于errno中。

**非阻塞I/O模型** ：recvfrom从应用层到内核的时候，如果该缓冲区没有数据的话，就直接返回一个EWOULDBLOCK错误，一般都是对非阻塞I/O模型进行轮询检查这个状态，看内核是不是有数据到来。
图：
![](/images/old/2018013011043-393ab84aab39a91c.png)

**I/O复用模型** ：Liunx提供select/poll，进程通过将一个或多个fd传输给select或者poll系统调用，阻塞在select操作上，这样select/poll可以侦测多个fd是否处于就绪状态。select/pool是顺序扫描fd是否就绪，而且支持的fd数量有限，所以受到一些制约。Linux还提供epoll系统调用，epoll使用基于事件驱动方式代替顺序扫描，性能更高，当有fd就绪的时候，立即回调rollback。
![](/images/old/2018013011043-30acb370892f468d.png)

**信号驱动I/O模型** ：先开启嵌套口信号驱动I/O功能，并通过系统调用sigaction执行一个信号处理函数（此系统立即返回，进程继续工作，非阻塞）。当数据准备就绪的时候，非该进程生成一个SIGIO信号，通过信号回调通知应用程序调用recvfrom来读取数据，并通知主循环函数处理数据。
![](/images/old/20180130dc9df87303a2aee6e741b48b78b4b8d9.jpeg)

**异步I/O** ：告知内核启动某个操作，让内核在整个操作完成后（包括将数据从内核复制到用户自己的缓冲区）通知我们。
![](/images/old/2018013011043-751d8b96d391cd43.png)
* 异步I/O模型和信号驱动I/O模型区别在于，信号驱动模型由内核通知何时可以开始I/O操作；异步I/O模型由内核通知I/O何时已经完成。

### JavaI/O
最开始Java的Socket通信都是采用了同步阻塞模式（BIO），之后在JDK1.4中提供了NIO类库，在JDK1.7中对原有的NIO类库进行升级称为NIO2.0，提供AIO功能，支持文件和网络的异步操作。

### JavaBIO实现
在使用BIO的服务端中，使用一个独立的Acceptor线程负责监听客户端的连接，接受连接后为每个客户端创建一个新的线程进行链路处理，之后通过输出流返回给调用客户端，线程销毁。
举例，客户端socket请求服务端获取当前时间：
```java
/**
 * TimeServer
 * 服务端
 * Created by  on 2018/1/31.
 */
public class TimeServer {
    public static void main(String[] args) throws IOException {
        int port = 8888;
        ServerSocket server = new ServerSocket(port);
        while (true){
            Socket socket = server.accept();
            new Thread(new TimeServerHandler(socket)).start();
        }
    }

}
/**
 * TimeServerHandler
 * 服务端socket处理
 * Created by  on 2018/1/31.
 */
public class TimeServerHandler implements Runnable {
    private Socket socket;

    public TimeServerHandler(Socket socket) {
        this.socket = socket;
    }

    @Override
    public void run() {
        try (
                //获取输入输出流
                BufferedReader in = new BufferedReader(new InputStreamReader(this.socket.getInputStream()));
                PrintWriter out = new PrintWriter(this.socket.getOutputStream(), true);
        ) {
            String line;
            SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            while ((line = in.readLine()) != null) {
                System.out.printf("The time server receive order: %s \n", line);
                //输出当前时间
                out.println("QUERY TIME".equalsIgnoreCase(line) ? format.format(new Date()) : "ERROR");
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (this.socket != null) {
                try {
                    this.socket.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
                this.socket = null;
            }
        }
    }
}
/**
 * TimeClient
 * 客户端
 * Created by  on 2018/1/31.
 */
public class TimeClient {
    public static void main(String[] args) {
        final int port = 8888;
        queryTime(port);
    }

    public static void queryTime(int port) {

        try (
                Socket socket = new Socket("127.0.0.1", port);
                BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
                PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
        ) {
            out.println("query time");
            System.out.printf("Server Time is:%s \n", in.readLine());
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
}
```

缺点：
服务端线程数和客户端并发数呈1：1，单客户端数量加大后会导致服务端线程创建数量增大，导致线程栈溢出创建线程失败等问题，最终服务端宕机，这种服务端可以采用线程池来优化。
线程池改进如下：
```Java
public class TimeServer {
    public static void main(String[] args) throws IOException {
        int port = 8888;
        poolServer(port);
    }

    public static void poolServer(int port)throws IOException{
        ExecutorService threadPool = Executors.newFixedThreadPool(8);
        ServerSocket server = new ServerSocket(port);
        while (true) {
            Socket socket = server.accept();
            threadPool.execute(new TimeServerHandler(socket));
        }
    }

    /**
     * 普通服务
     * @param port
     * @throws IOException
     */
    public static void ordinaryServer(int port)throws IOException {
        ServerSocket server = new ServerSocket(port);
        while (true) {
            Socket socket = server.accept();
            new Thread(new TimeServerHandler(socket)).start();
        }
    }

}
```
服务端通过创建一个线程池，当有连接是，由线程池提交一个任务。
缺点：
当出现线程任务中某些任务非常耗时，这样会导致后续的链接一直在线程队列中等待，同时客户端也会一直阻塞等待服务端的返回。同时如果线程池中队列满了，会直接拒绝新的请求。

### JavaNIO
改进使用JavaNIO
```Java
/**
 * MultiplexerTimeServer
 * 多路复用server
 * 步骤：
 * 1、打开Selector
 * 2、打开ServerSocketChannel
 * 3、ServerSocketChannel绑定监听地址
 * 4、注册ServerSocketChannel到Selector上
 * 5、启动线程轮询Selector中就绪的key
 * Created by  on 2018/1/31.
 */
public class MultiplexerTimeServer implements Runnable {
    private Selector selector;
    private ServerSocketChannel serverSocketChannel;
    private volatile boolean stop;

    public MultiplexerTimeServer(int port) {
        try {
            //创建Reactor线程，打开多路复用器
            selector = Selector.open();
            //打开ServerSocketChannel用于监听客户端链接，是所有客户端链接的父管道
            serverSocketChannel = ServerSocketChannel.open();
            //设置绑定端口，设置为非阻塞模式
            serverSocketChannel.configureBlocking(false);
            //10086请求链接最大队列长度
            serverSocketChannel.socket().bind(new InetSocketAddress(port), 10086);
            //将serverSocketChannel注册到selector线程的多路复用器上，监听ACCEPT操作
            serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
            System.out.printf("The Server is Start: 120.0.0.1:%d \n", port);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        while (!stop) {
            try {
                //等待获取channel
                selector.select(1000);
                //获取所有复用器上的key
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> keyIterator = selectionKeys.iterator();
                SelectionKey key;
                while (keyIterator.hasNext()) {
                    key = keyIterator.next();
                    keyIterator.remove();
                    //处理key
                    handleInput(key);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        if (selector != null) {
            try {
                //多路复用器关闭后上面的注册的Channel和Pipe都会关闭
                selector.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private void handleInput(SelectionKey key) {
        SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        //判断key是否有效
        try {
            if (key.isValid()) {
                //判断key是否OP_ACCEPT状态
                if (key.isAcceptable()) {
                    //获取channel，实际上是最开始注册的那个serverSocketChannel
                    ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
                    //阻塞监听新客户端请求，完成TCP3次握手，对TCP参数进行设置
                    SocketChannel socketChannel = ssc.accept();
                    socketChannel.configureBlocking(false);
                    //监听到的客户端注册到selector上，监听OP_READ操作，读取客户端发送的网络消息
                    socketChannel.register(selector, SelectionKey.OP_READ);
                }
                //判断key是否是可以读取
                if (key.isReadable()) {
                    //获取读取的channel
                    SocketChannel socketChannel = (SocketChannel) key.channel();
                    //读取操作
                    //开辟1M缓冲区
                    ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                    //读取数据
                    int readBytes = socketChannel.read(readBuffer);
                    if (readBytes > 0) {
                        //将缓冲区当前的limit设置为position，position设置为0，用于后续对缓冲区的读取，不然读取都是缓冲区后面的都是0了
                        readBuffer.flip();
                        //依据缓冲区可读字节创建数组
                        byte[] bytes = new byte[readBuffer.remaining()];
                        //将缓冲区可读字节数组复制到新创建的字节数组中
                        readBuffer.get(bytes);
                        String body = new String(bytes, "utf-8");
                        System.out.printf("The time server receive order: %s \n", body);
                        String response = "QUERY TIME".equalsIgnoreCase(body) ? format.format(new Date()) : "ERROR\n";
                        //返回操作
                        byte[] outBytes = response.getBytes();
                        ByteBuffer writeBuffer = ByteBuffer.allocate(outBytes.length);
                        writeBuffer.put(outBytes);
                        //用于后续的写
                        writeBuffer.flip();
                        //这里会出现写 半包 ，需要注册监听写操作
                        socketChannel.write(writeBuffer);
                    }
                }
            }
        } catch (IOException e) {
            if (key.channel() != null){
                try {
                    key.channel().close();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }
            }
        }
    }

    public void setStop(boolean stop) {
        this.stop = stop;
    }
}
/**
 * TimeServer
 * nio服务端
 * Created by  on 2018/1/31.
 */
public class TimeServer {
    public static void main(String[] args) {
        final int port = 8888;
        new Thread(new MultiplexerTimeServer(port), "NIO-MultiplexerTimeServer").start();
    }


}
/**
 * TimeClientHandle
 * Created by  on 2018/1/31.
 */
public class TimeClientHandle implements Runnable {
    private String host;
    private int port;
    private Selector selector;
    private SocketChannel socketChannel;
    private volatile boolean stop;

    public TimeClientHandle(String host, int port) {
        this.host = host;
        this.port = port;
        try {
            selector = Selector.open();
            socketChannel = SocketChannel.open();
            socketChannel.configureBlocking(false);
        } catch (IOException e) {
            e.printStackTrace();
        }

    }

    @Override
    public void run() {
        try {
            doConnect();
        } catch (IOException e) {
            e.printStackTrace();
        }
        while (!stop) {
            try {
                selector.select(1000);
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> keyIterator = selectionKeys.iterator();
                SelectionKey key = null;
                while (keyIterator.hasNext()) {
                    key = keyIterator.next();
                    keyIterator.remove();
                    handleInput(key);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        if (selector != null) {
            try {
                selector.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

    }

    private void handleInput(SelectionKey key) {
        try {
            if (key.isValid()) {
                SocketChannel sc = (SocketChannel) key.channel();
                if (key.isConnectable()) {
                    if (sc.finishConnect()) {
                        sc.register(selector, SelectionKey.OP_READ);
                        doWrite(sc);
                    } else {
                        System.exit(1);
                    }
                }
                if (key.isReadable()) {
                    ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                    int readBytes = sc.read(readBuffer);
                    if (readBytes > 0) {
                        readBuffer.flip();
                        byte[] bytes = new byte[readBuffer.remaining()];
                        readBuffer.get(bytes);
                        String body = new String(bytes, "utf-8");
                        System.out.printf("Now is %s \n", body);
                    } else if (readBytes < 0) {
                        //链路关闭
                        sc.close();
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void doConnect() throws IOException {
        //连接成功注册到多路复用器上发送请求消息，等待读取
        if (socketChannel.connect(new InetSocketAddress(host, port))) {
            socketChannel.register(selector, SelectionKey.OP_READ);
            doWrite(socketChannel);
        } else {
            //否则注册连接到等待连接，可能TCP没有握手应答消息
            socketChannel.register(selector, SelectionKey.OP_CONNECT);
        }
    }

    private void doWrite(SocketChannel sc) throws IOException {
        byte[] bytes = "query time".getBytes();
        ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
        writeBuffer.put(bytes);
        writeBuffer.flip();
        sc.write(writeBuffer);
        if (!writeBuffer.hasRemaining()) {
            System.out.println("Send Success");
        }
    }
}
public class TimeClient {
    public static void main(String[] args) {
        new Thread(new TimeClientHandle("127.0.0.1", 8888), "TimeClient").start();
    }
}
```
分别启动客户端和服务端后就可以看到输出信息了。

**Buffer：**
在上述使用NIO的时候涉及到缓冲区Buffer，数据读取和写入都是通过缓冲区。缓冲区实际上是一个数组，上述使用的事ByteBuffer，可以使用其他类型数组（基本上Java基本类型都有对应的缓冲区，除了Boolean），不过缓冲区不仅仅是数组，还提供了对数据结构化访问以及维护读写位置等信息。

**Channel：**
除了使用缓冲区还要使用到Channel，网络中数据通过Channel读取和写入，Channel可以用于读写以及同时存在。Channel在使用过程中，可以分为两大类：SelectableChannel（网络读写）、FileChannel（文件操作）。

Channel子类关系图如下：
![](/images/old/20180131屏幕快照2018-01-31下午5.02.38.png)

**Selector：**
Selector在NIO中属于核心部分，在使用过程中通过轮询注册在Selector上的Channel，因为JDK中使用了`epoll`所以没有最大连接句柄的现在。所以只需要一个线程负责Selector轮询就能接入大量的客户端。

### AIO
在JavaNIO2.0中使用的是之前对应的AIO，不需要使用多路复用器对注册的通道进行轮询，即可实现异步读写。
例子：
```java
public class TimeServer {
    public static void main(String[] args) {
        final int port = 8888;
        AsyncTimeServerHandler asyncTimeServerHandler = new AsyncTimeServerHandler(port);
        new Thread(asyncTimeServerHandler, "AIO-TimeServer").start();
    }
}
/**
 * AsyncTimeServerHandler
 * 异步服务
 * Created by  on 2018/2/1.
 */
public class AsyncTimeServerHandler implements Runnable {
    private int port;
    private AsynchronousServerSocketChannel serverSocketChannel;
    private CountDownLatch latch;

    public AsyncTimeServerHandler(int port) {
        this.port = port;
        try {
            //开启异步serverSocketChannel
            serverSocketChannel = AsynchronousServerSocketChannel.open();
            //绑定端口
            serverSocketChannel.bind(new InetSocketAddress(port));
            System.out.println("Server Start ...");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        latch = new CountDownLatch(1);
        doAccept();
        try {
            //阻塞线程，避免服务端结束，在实际使用中，不需要新驱动线程以及阻塞
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    public void doAccept(){
        //使用AcceptCompletionHandler接收accept操作成功的通知消息
        serverSocketChannel.accept(this, new AcceptCompletionHandler());
    }

    public AsynchronousServerSocketChannel getServerSocketChannel() {
        return serverSocketChannel;
    }

    public CountDownLatch getLatch() {
        return latch;
    }
}
/**
 * AcceptCompletionHandler
 * 接收完成处理
 * Created by  on 2018/2/1.
 */
public class AcceptCompletionHandler implements CompletionHandler<AsynchronousSocketChannel, AsyncTimeServerHandler> {


    @Override
    public void completed(AsynchronousSocketChannel result, AsyncTimeServerHandler attachment) {
        //这里的作用是，接收客户端成功后，用于接收其他客户端连接，每次接收一个客户端成功后，在异步接收新客户端
        //接收成功后，会回调AcceptCompletionHandler.completed
        attachment.getServerSocketChannel().accept(attachment, this);
        //链路建立成功后，服务端分配缓冲区接收客户端数据
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        /*
         * 参数：
         * 1 接收缓冲区，用于异步Channel中读取数据包
         * 2 异步Channel携带的附件，通知回调的时候，作为入参数使用
         * 3 接收通知回调业务Handler
         */
        result.read(buffer, buffer, new ReadCompletionHandler(result));
    }

    @Override
    public void failed(Throwable exc, AsyncTimeServerHandler attachment) {
        exc.printStackTrace();
        attachment.getLatch().countDown();
    }
}
/**
 * ReadCompletionHandler
 * 接收通知回调
 * Created by  on 2018/2/1.
 */
public class ReadCompletionHandler implements CompletionHandler<Integer, ByteBuffer> {
    //用于半包消息和发送应答
    private AsynchronousSocketChannel socketChannel;

    public ReadCompletionHandler(AsynchronousSocketChannel socketChannel) {
        this.socketChannel = socketChannel;
    }

    @Override
    public void completed(Integer result, ByteBuffer attachment) {
        SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        //读取缓冲区数据做准备
        attachment.flip();
        byte[] bytes = new byte[attachment.remaining()];
        attachment.get(bytes);
        try {
            String request = new String(bytes, "utf-8");
            System.out.printf("client request is: %s\n", request);
            //写消息返回客户端
            doWrite("QUERY TIME".equalsIgnoreCase(request) ? format.format(new Date()) : "ERROR");
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }

    }

    private void doWrite(String body) {
        if (body != null && body.trim().length() > 0) {
            byte[] bytes = body.getBytes();
            ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
            writeBuffer.put(bytes);
            writeBuffer.flip();
            //写消息到客户端，参数和read一样，后面handler用于发送消息后回调接口
            socketChannel.write(writeBuffer, writeBuffer, new CompletionHandler<Integer, ByteBuffer>() {
                @Override
                public void completed(Integer result, ByteBuffer attachment) {
                    //没发送完毕
                    if (attachment.hasRemaining()) {
                        socketChannel.write(attachment, attachment, this);
                    }
                }

                @Override
                public void failed(Throwable exc, ByteBuffer attachment) {
                    try {
                        socketChannel.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    }

    @Override
    public void failed(Throwable exc, ByteBuffer attachment) {
        try {
            socketChannel.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
public class TimeClient {
    public static void main(String[] args) {
        new Thread(new AsyncTimeClientHandler("127.0.0.1", 8888)).start();
    }
}
/**
 * AsyncTimeClientHandler
 * Created by  on 2018/2/1.
 */
public class AsyncTimeClientHandler implements CompletionHandler<Void, AsyncTimeClientHandler>, Runnable {
    private String host;
    private int port;
    private AsynchronousSocketChannel socketChannel;
    private CountDownLatch latch;

    public AsyncTimeClientHandler(String host, int port) {
        this.host = host;
        this.port = port;
        try {
            socketChannel = AsynchronousSocketChannel.open();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        latch = new CountDownLatch(1);
        socketChannel.connect(new InetSocketAddress(host, port), this, this);
        try {
            latch.await();
            socketChannel.close();
        } catch (InterruptedException | IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void completed(Void result, AsyncTimeClientHandler attachment) {
        byte[] request = "query time".getBytes();
        ByteBuffer writeBuffer = ByteBuffer.allocate(request.length);
        writeBuffer.put(request);
        writeBuffer.flip();
        //准备完毕后发送消息到客户端
        socketChannel.write(writeBuffer, writeBuffer, new CompletionHandler<Integer, ByteBuffer>() {
            @Override
            public void completed(Integer result, ByteBuffer attachment) {
                if (attachment.hasRemaining()){
                    //没写完重写
                    socketChannel.write(attachment, attachment, this);
                } else {
                    //写完准备读取客户端返回数据
                    ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                    socketChannel.read(readBuffer, readBuffer, new CompletionHandler<Integer, ByteBuffer>() {
                        @Override
                        public void completed(Integer result, ByteBuffer attachment) {
                            attachment.flip();
                            byte[] bytes = new byte[attachment.remaining()];
                            attachment.get(bytes);
                            try {
                                String body = new String(bytes, "utf-8");
                                System.out.printf("now is : %s \n", body);
                                latch.countDown();
                            } catch (UnsupportedEncodingException e) {
                                e.printStackTrace();
                            }
                        }

                        @Override
                        public void failed(Throwable exc, ByteBuffer attachment) {
                            try {
                                socketChannel.close();
                                latch.countDown();
                            } catch (IOException e) {
                                e.printStackTrace();
                            }
                        }
                    });
                }
            }

            @Override
            public void failed(Throwable exc, ByteBuffer attachment) {
                try {
                    socketChannel.close();
                    latch.countDown();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        });
    }

    @Override
    public void failed(Throwable exc, AsyncTimeClientHandler attachment) {
        try {
            socketChannel.close();
            latch.countDown();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

几种I/O模型功能特性对比：

|         | 同步阻塞I/O(BIO)    |  伪异步I/O  | 非阻塞I/O(NIO) | 异步I/O(AIO) |
| --------- | :--------- | :---------| :-----------  | :----------|
| 客户端/服务端：I/O线程 | 1:1 | M:N(M可以大于N) | M:1(1个服务端处理多个客户端) | M:0(不需要启动额外的I/O线程，OS回调) |
| 阻塞类型 | 阻塞I/O | 阻塞I/O | 非阻塞I/O | 非阻塞I/O |
| 同步类型 | 同步 | 同步 | 同步(多路复用)  | 异步 |
| API使用难度 | 简单 | 简单 | 非常复杂 | 复杂 |
| 调试难度 | 简单 | 简单 | 复杂 | 复杂 |
| 可靠性 | 非常差 | 差 | 高 | 高 |
| 吞吐量 | 低 | 中 | 高 | 高 |

在网络上据说Linux下AIO对性能提升并不高。所以Netty依旧使用封装的NIO，可查看
* https://github.com/netty/netty/issues/2515

参考：
* netty权威指南
