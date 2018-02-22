---
title: Spring项目中集成feign
date: 2018-02-22 15:09:29
categories: ["feign"]
tags: ["feign"]
---

在平常项目中使用远程请求时，一般只自己手动封装Apache http请求，使用时可能有时候不够直观，现在尝试集成feign。
<!--more-->
添加feign依赖
```xml
<!--核心包-->
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-core</artifactId>
    <version>${feign.version}</version>
</dependency>
<!--处理json-->
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-gson</artifactId>
    <version>${feign.version}</version>
    <exclusions>
        <exclusion>
            <artifactId>gson</artifactId>
            <groupId>com.google.code.gson</groupId>
        </exclusion>
    </exclusions>
</dependency>
<!--打印日志，方便调试-->
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-slf4j</artifactId>
    <version>${feign.version}</version>
    <exclusions>
        <exclusion>
            <artifactId>slf4j-api</artifactId>
            <groupId>org.slf4j</groupId>
        </exclusion>
    </exclusions>
</dependency>
<!--用于form请求时封装-->
<dependency>
    <groupId>io.github.openfeign.form</groupId>
    <artifactId>feign-form</artifactId>
    <version>${feign.form.version}</version>
</dependency>
<!--用于spring form-->
<dependency>
    <groupId>io.github.openfeign.form</groupId>
    <artifactId>feign-form-spring</artifactId>
    <version>${feign.form.version}</version>
    <exclusions>
        <exclusion>
            <artifactId>spring-web</artifactId>
            <groupId>org.springframework</groupId>
        </exclusion>
    </exclusions>
</dependency>
<!--集成使用Apache http-->
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-httpclient</artifactId>
    <version>${feign.version}</version>
</dependency>
```

在现在有个接口中请求参数格式为data={}//json，返回的格式为json格式
新建接口如下：
```java
public interface AuthCall {
    @RequestLine("POST /openapi/query.do")
    @Headers("Content-Type: application/x-www-form-urlencoded")
    QueryStoreAndAreaResultVo queryMappingStoreAndArea(@Param(value = "data")QueryStoreAndAreaParamsVo paramsVo);
}
```
在使用feign时，默认会对请求数据进行序列化为json，因为为了满足自己项目需求，需要自己封装请求的序列化，以及返回参数的反序列化。如下：
```java
//序列化请求参数
public class FNDataRawEncode implements Encoder {

    @Override
    public void encode(Object o, Type type, RequestTemplate requestTemplate) throws EncodeException {
        if (o instanceof Map) {
            Map<String, Object> objectMap = (Map<String, Object>) o;
            StringBuilder sb = new StringBuilder();
            int i = objectMap.size();
            for (Map.Entry<String, Object> entry : objectMap.entrySet()) {
                i--;
                sb.append(entry.getKey()).append("=").append(GsonUtils.toJson(entry.getValue()));
                if (i != 0) {
                    sb.append("&");
                }
            }
            requestTemplate.body(sb.toString());
        }
    }
}
//反序列化返回的json
public class FNDataRawDecoder extends GsonDecoder {
    @Override
    public Object decode(Response response, Type type) throws IOException, FeignException {
        if (response.status() != 200) {
            return Util.emptyValueOf(type);
        }
        Response.Body body = response.body();
        if (body == null) return null;
        if (type == String.class) {
            return Util.toString(body.asReader());
        } else {
            return super.decode(response, type);
        }
    }
}
```

集成进入spring，添加bean如下：
```java
@Configuration
public class BeanConfigure {

    @Bean
    public AuthCall authCall(){
        return Feign.builder()
                .client(new ApacheHttpClient())
                //设置为上述自定义的类
                .encoder(new FNDataRawEncode())
                .decoder(new FNDataRawDecoder())
                //设置输出日志
                .logger(new Slf4jLogger(AuthCall.class))
                //设置日志等级
                .logLevel(Logger.Level.FULL)
                .target(AuthCall.class, "http://queyr.beta.fn");
    }
}
```

之后启动项目，注入AuthCall对象后即可使用该接口。