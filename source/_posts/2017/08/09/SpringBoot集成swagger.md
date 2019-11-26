---
title: SpringBoot集成swagger
date: 2017-08-09 11:08:00
categories: ['SpringBoot']
tags: ['SpringBoot', 'swagger']
---

> Swagger 是一款RESTFUL接口的文档在线自动生成+功能测试功能软件。参考 *[官方地址](https://swagger.io/)*

### 在SpringBoot中集成swagger

#### 创建maven项目，pom.xml添加依赖
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.4.RELEASE</version>
</parent>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.6.1</version>
    <scope>compile</scope>
</dependency>

```
<!-- more -->
#### 添加启动，以及配置
```java
//启动类SpringBootMain
@SpringBootApplication
public class SpringBootMain {
    public static void main(String[] args) {
        SpringApplication.run(SpringBootMainConfig.class, args);
    }
}
```
```java
//spring 配置SpringBootMainConfig
@ComponentScan("com.whh")
@EnableAutoConfiguration
@EnableAsync
public class SpringBootMainConfig {
}
```
```java
//Swagger配置
@Configuration
@EnableSwagger2
//控制不同环境执行该配置
@Profile(value = {"dev", "beta"})
public class SwaggerConfig {
    @Bean
    public Docket api() {
        return new Docket(DocumentationType.SWAGGER_2)
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.whh"))
                .paths(PathSelectors.any())
                .build();
    }
}
```
```java
//api controller
@RestController
@RequestMapping("demoApi")
@Api(value = "测试接口", description = "这个是测试用的接口")
public class DemoApiController {
    private static Map<String, String> data = new ConcurrentHashMap<>();
    static {
        data.put("111", "aaa");
        data.put("222", "bbb");
        data.put("333", "ccc");
        data.put("444", "ddd");
        data.put("555", "eee");
    }
    @ApiOperation(value = "所有数据", response = Map.class)
    @RequestMapping(value = "/allData", method = RequestMethod.GET)
    public Map<String, String> allData(){
        return data;
    }

    @ApiOperation(value = "展示数据", response = String.class)
    @ApiResponses(value = {
            @ApiResponse(code = 200, message = "数据正常"),
            @ApiResponse(code = 404, message = "页面找不到")
    })
    @RequestMapping(value = "/show/{key}", method = RequestMethod.GET)
    public String value(@PathVariable String key){
        return data.get(key);
    }

    @RequestMapping(value = "/add", method = RequestMethod.GET)
    public ResponseEntity<String> add(@RequestParam String key, @RequestParam String value){
        data.put(key, value);
        return new ResponseEntity<>("Add OK", HttpStatus.OK);
    }

    @RequestMapping(value = "/update/{key}", method = RequestMethod.POST)
    public ResponseEntity<String> update(@RequestParam String value, @PathVariable String key){
        data.put(key, value);
        return new ResponseEntity<>("Update OK", HttpStatus.OK);
    }

    @RequestMapping(value = "/remove/{key}", method = RequestMethod.GET)
    public ResponseEntity<String> remove(@PathVariable String key){
        data.remove(key);
        return new ResponseEntity<>("Remove", HttpStatus.OK);
    }
}
```
#### 添加swaggerUI
下载swaggerUI，[github swagger-ui官方地址](https://github.com/swagger-api/swagger-ui)，把文件夹dist 放入springboot项目resource/statis下。改名为swagger-ui。修改文件index.html js中SwaggerUIBundle的url为http://localhost:7070/v2/api-docs。
#### 添加springboot配置文件
在resource下添加目录coufig，添加文件application.properties
```
#配置文件环境
spring.profiles.active=dev
#server 启动端口
server.port=7070
#thymeleaf 模板缓存
spring.thymeleaf.cache=false
#spring.mvc 配置静态文件为准
spring.mvc.static-path-pattern=/static/**
```

启动main方法，访问地址http://localhost:7070/static/swagger-ui/index.html ，大功告成

> 如果需要直接通过配置文件。直接生产restful风格代码，可以使用[swagger edit](https://editor.swagger.io/)。

### 传统SpringMVC添加swagger

#### 添加pom依赖
```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.6.1</version>
    <scope>compile</scope>
</dependency>
```
#### 添加swagger配置文件
```java
@Configuration
@EnableSwagger2
@EnableWebMvc
public class SwaggerConfig {
    @Bean
    public Docket api() {
        return new Docket(DocumentationType.SWAGGER_2)
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.whh"))
                .paths(PathSelectors.any())
                .build();
    }
}
```
> 如果确实jackson类，需要添加jackson依赖

后续操作和SpringBoot一样，添加ui和配置js地址等。
