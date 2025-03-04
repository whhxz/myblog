---
title: feign异常feign.RetryableException & NoHttpResponseException
date: 2025-03-04 14:55:27
categories:
tags: ["feign", "EXception", "apache http client"]
---

系统总是出现`feign.RetryableException`，需要找到原因并解决，虽然问题不大，但是一直有这个异常日志。
项目采用的是`spring-cloud-starter-openfeign`自动注入、`apache httpclient`做连接池。并没有对配置做相关优化，采用默认配置。
<!-- more -->

### feign.RetryableException定位
通过异常栈定位到问题出在`feign.SynchronousMethodHandler#invoke`内调用`executeAndDecode`。执行请求出现异常后抛出。
```java
public Object invoke(Object[] argv) throws Throwable {
  RequestTemplate template = buildTemplateFromArgs.create(argv);
  Options options = findOptions(argv);
  Retryer retryer = this.retryer.clone();
  while (true) {
    try {
      //执行请求，里面有抛出RetryableException
      return executeAndDecode(template, options);
    } catch (RetryableException e) {
      try {
        //异常后，通过相关配置判断是否需要处理
        retryer.continueOrPropagate(e);
      } catch (RetryableException th) {
        Throwable cause = th.getCause();
        if (propagationPolicy == UNWRAP && cause != null) {
          throw cause;
        } else {
          throw th;
        }
      }
      if (logLevel != Logger.Level.NONE) {
        logger.logRetry(metadata.configKey(), logLevel);
      }
      continue;
    }
  }
}

```
默认情况下`feign`配置重试机制使用的是`feign.Retryer.Default`，默认重试**5**次。
在使用了`SpringBoot`自动配置后，在`org.springframework.cloud.openfeign.FeignClientsConfiguration#feignRetryer`可以看到默认使用的是`feign.Retryer#NEVER_RETRY`，里面是不做重试直接抛出异常。那就可以看到为什么请求失败会出现`feign.RetryableException`。

### NoHttpResponseException 定位
在上面`feign`异常栈的后面可以看到`NoHttpResponseException`异常，通过异常栈可以看到代码`org.apache.http.impl.conn.DefaultHttpResponseParser#parseHead`处抛出的异常，是读取返回时出错抛出。

从stackoverflow [Apache HttpClient Interim Error: NoHttpResponseException][]里面可以看到，根本原因是因为使用了过期连接导致。

解决办法：
* 官方建议[定时清理]过期连接。
* 设置自定义重试次数。
* 使用okhttp替换`apache httpclient`。



[Apache HttpClient Interim Error: NoHttpResponseException]: https://stackoverflow.com/questions/10558791/apache-httpclient-interim-error-nohttpresponseexception
[定时清理]: https://hc.apache.org/httpcomponents-client-4.5.x/current/tutorial/html/connmgmt.html#d5e418