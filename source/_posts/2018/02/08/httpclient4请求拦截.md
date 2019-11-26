---
title: httpclient4请求拦截
date: 2018-02-08 09:23:27
categories: ['分布式调用链']
tags: ['Apache httpclient4', '调用链']
---

最近在准备写一个分布式调用链记录，记录用请求开始到请求各个系统直到结束。不同系统间调用方式不一样，现在第一步先拦截http请求，获取请求参数以及返回值，并做日志记录。
使用http请求中，可能使用了各种框架，暂时先处理Apache httpclient。
<!-- more -->
### 拦截HttpRequestExecutor
在使用httpclient时，最终都会调用到HttpRequestExecutor.execute提交请求，可以对execute进行拦截处理，在execute方法前后插入获取请求和返回值。

1、在启动加载HttpRequestExecutor时，通过javassist修改class
2、在方法前后插入指定方法
3、方法前获取请求数据URL等，header中插入traceId标示
4、方法后插入获取返回值

因为execute中存在参数HttpRequest，所以设置头部信息比较方便，但是如果想获取请求参数比较麻烦。

### DefaultBHttpClientConnection修改
在execute中获取请求参数需要强制转换对象为子类，可能出现未知的子类。所以通过修改DefaultBHttpClientConnection的class在sendRequestEntity前插入一段代码，用于获取请求参数。
例子如下：
```java
public static void sendRequestEntity(Object target, Object[] args){
    HttpEntityEnclosingRequest request = (HttpEntityEnclosingRequest)args[0];
    HttpEntity entity = request.getEntity();
    try {
        if (entity != null){
            InputStream content = entity.getContent();
            if (content.markSupported()){
                int contentLength = (int) entity.getContentLength();
                final ByteArrayBuffer buffer = new ByteArrayBuffer(contentLength);
//                    content.mark(contentLength);
                content.read(buffer.buffer(), 0, contentLength);
//                    content.reset();
                System.out.println("~~~~~~~~~~~~~~~~");
                System.out.println(new String(buffer.buffer()));
                System.out.println("~~~~~~~~~~~~~~~~");

            }
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```
在例子中，因为该请求中使用的是ByteArrayInputStream，所以可以直接读取流中的数据，而不影响数据的发送。
这里可能存在一个问题，如果请求中保存的流不是ByteArrayInputStream，可能是其他的，就需要mark后reset，这种对于不支持mark的流就需要使用额外的处理方式了。
终极办法是，通过修改sendRequestEntity中prepareOutput方法的调用，在返回的OutputStream中对该流进行封装（装饰设计模式），定义一个OutputStreamWrapper，重写write方法，在write时，保存字节到数组中，同时写入流中。在之后可以通过读取字节中的数据，而不用担心出现之前的那种情况。

### BHttpConnectionBase修改
因为需要获取返回值，在第一步骤中execute是可以获取返回的HttpResponse，但是在读取HttpResponse中的数据后，使最终调用改接口的地方无法获取返回值，因为在返回的事InputStream中，只能读取一次，拦截时获取了返回值，就会导致实际调用方无法获取返回值。
这时就需要修改BHttpConnectionBase中prepareInput方法调用createInputStream时，返回的是一个支持重复读的流。修改如下
```java
CtMethod prepareInput = ctClass.getDeclaredMethod("prepareInput");
prepareInput.instrument(new ExprEditor() {
    @Override
    public void edit(MethodCall m) throws CannotCompileException {
        if ("createInputStream".equals(m.getMethodName())) {
            m.replace("{$_ = new java.io.BufferedInputStream($proceed($$));}");
        }
    }
});
```
通过获取prepareInput，之后修改createInputStream，包装返回的BufferedInputStream，在之后execute中获取返回值时，通过mark、reset来读取流，同时解决了IO无法重复读的问题。
