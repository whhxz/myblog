---
title: Spring Cloud 初步使用
date: 2018-01-18 09:53:41
categories: ['Spring Cloud']
tags: ['Spring', 'Spring Cloud']
---

### Eureka Server （服务中心）
Eureka是一个服务注册和发现模块，基于 REST 的服务，用于定位服务，以实现云端中间层服务发现和故障转移。

<!-- more -->
新建maven项目，pom文件如下：
```xml
<!-- eureka-server/pom.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.whh.springcloud</groupId>
	<artifactId>eureka-server</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>eureka-server</name>
	<description>Demo project for Spring Boot</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.9.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
		<spring-cloud.version>Edgware.SR1</spring-cloud.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka-server</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>
```
reousece/application.yml 如下：
```yml
spring:
  application:
    name: eureka-server
server:
  port: 8761
eureka:
  instance:
    hostname: localhost
  client:
    #不注册eureka
    register-with-eureka: false
    #不从eureka获取信息
    fetch-registry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```
EurekaDiscoveryApplication.java
```java
@SpringBootApplication
@EnableEurekaClient
public class EurekaDiscoveryApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaDiscoveryApplication.class, args);
	}
}
```
启动EurekaDiscoveryApplication后，打开localhost:8761访问eureka web页面。

### Eureka Discovery （服务提供）
新建项目，用于提供服务。
pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.whh.springcloud</groupId>
	<artifactId>eureka-discovery</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>eureka-discovery</name>
	<description>Demo project for Spring Boot</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.9.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
		<spring-cloud.version>Edgware.SR1</spring-cloud.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>


</project>
```
resource/application.yml
```yml
spring:
  application:
    name: eureka-discovery
server:
  port: 8762

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    healthcheck:
      enabled: true
```
Main主类
```java
@SpringBootApplication
@EnableEurekaClient
public class EurekaDiscoveryApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaDiscoveryApplication.class, args);
	}
}
```
新增Controller提供服务
```java
@RestController
@RequestMapping("demo")
public class DemoController {
    @RequestMapping("home")
    public String home(@RequestParam String name){
        return "welcome " + name;
    }
}
```
启动服务，之后可以在Eureka Server Web页面查看到服务列表.
* 在启动过程中，如果之后服务提供者重启，之后web页面可能会出现错误信息，这是Eureka启动了**自我保护模式**，可以通过**eureka.server.enable-self-preservation = false**禁用保护模式

### eureka consumer （消费客户端）
建立服务消费者，新建maven项目
pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.whh.springcloud</groupId>
	<artifactId>consumer-client</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>consumer-client</name>
	<description>Demo project for Spring Boot</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.9.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
		<spring-cloud.version>Edgware.SR1</spring-cloud.version>
	</properties>

	<dependencies>
		<!--<dependency>-->
			<!--<groupId>org.springframework.cloud</groupId>-->
			<!--<artifactId>spring-cloud-starter-ribbon</artifactId>-->
		<!--</dependency>-->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-eureka</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>


</project>
```
reousece/application.yml
```yml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    register-with-eureka: false
server:
  port: 8764
spring:
  application:
    name: consumer-client
```
启动类
```java
@SpringBootApplication
public class ConsumerClientApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConsumerClientApplication.class, args);
	}
}
```


#### 使用RestTemplate
如果RestTemplate调用远程接口，新建如下类
配置类
```java
@Configuration
public class SpringBeanConfig {
    @Bean
    //负载均衡
    @LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```
Controller调用
```java
@RestController
@RequestMapping("client")
public class DemoClientController {
    @Autowired
    private RestTemplate restTemplate;

    @RequestMapping("home")
    public String home() {
        return restTemplate.getForObject("http://EUREKA-DISCOVERY/demo/home?name=whh", String.class);
    }
}
```
启动消费客户端，访问localhsot:8764/client/home，返回的eureka服务提供者返回的值，eureka服务提供者可以通过修改端口，启动多个服务，在客户端调用时会自动做负载均衡。
#### Ribbon

在消费客户端使用的事spring-cloud-starter-eureka，该依赖包含ribbon，所以使用的是ribbon做的负载均衡。
常用的负载均衡策略有：
* 简单轮询负载均衡
* 加权响应时间负载均衡
* 区域感知轮询负载均衡
* 随机负载均衡

#### Feign

通过Feign封装了HTTP调用服务方法，使得客户端像调用本地方法那样直接调用方法，Feign默认集成了Ribbon，默认实现了负载均衡。
在上述消费客户端pom文件中加入Feign依赖：
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
```
在启动的main方法中添加注解**@EnableFeignClients**开启Feign
新建接口用于远程调用，如下：
```java
//远程服务
@FeignClient(value = "eureka-discovery")
@RequestMapping("demo")
public interface DemoRemoteService {
    @RequestMapping(value = "home", method = RequestMethod.GET)
    String home(@RequestParam("name") String name);
}
```
在Controller中新增调用方法：
```java
@Autowired
private DemoRemoteService demoRemoteService;
@RequestMapping("feignHome")
public String feignHome(){
    return demoRemoteService.home("whh");
}
```
启动client，浏览器调用client接口即可查看返回数据。

### Hystrix （熔断器）
熔断器，容错管理工具，旨在通过熔断机制控制服务和第三方库的节点,从而对延迟和故障提供更强大的容错能力。可以防止服务雪崩效应。
在pom文件中新增Hystrix依赖
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
```
修改application.yml配置启动Hystrix，添加：
```yml
feign:
  hystrix:
    enabled: true
```
Main添加注解**@EnableHystrix**启动hystrix

#### fallback 降级
修改上述DemoRemoteService
```java
@FeignClient(value = "eureka-discovery", path = "demo", fallback = DemoRemoteServiceFallBack.class)
public interface DemoRemoteService {
    @RequestMapping(value = "home", method = RequestMethod.GET)
    String home(@RequestParam("name") String name);
}
```

新增DemoRemoteService子类
```java
@Component
public class DemoRemoteServiceFallBack implements DemoRemoteService {
    @Override
    public String home(String name) {
        return name + " System Error";
    }
}
```
启动消费客户端，访问接口正常，关闭服务提供，重新访问接口，返回熔断器定义的错误信息。
* 需要注意的地方是，之前DemoRemoteService中，定义class的请求路径为@RequestMapping("demo")，这次修改为在FeignClient中的path中设置路径，不然在使用fallback的时候会报错。

如果客户端Controller方法修改为：
```java
for (int i = 0; i < 10; i++) {
    new Thread(()->{
        for (int j = 0; j < 100; j++) {
            System.out.println(demoRemoteService.home("whh"));
        }
    }).start();
}
```
虽然服务正常，但是还是会触发降级，返回System Error。这就涉及到Hystrix隔离策略。

#### 隔离策略
hystrix提供了两种隔离策略：线程池隔离和信号量隔离。hystrix默认采用线程池隔离。

* 线程池隔离：
> 不同服务通过使用不同线程池，彼此间将不受影响，达到隔离效果。

* 信号量隔离：
> 线程隔离会带来线程开销，有些场景（比如无网络请求场景）可能会因为用开销换隔离得不偿失，为此hystrix提供了信号量隔离，当服务的并发数大于信号量阈值时将进入fallback。

#### 熔断机制
熔断机制相当于电路的跳闸功能，举个栗子，我们可以配置熔断策略为当请求错误比例在10s内>50%时，该服务将进入熔断状态，后续请求都会进入fallback。

#### 结果cache
hystrix支持将一个请求结果缓存起来，下一个具有相同key的请求将直接从缓存中取出结果，减少请求开销。要使用hystrix cache功能，第一个要求是重写getCacheKey()，用来构造cache key；第二个要求是构建context，如果请求B要用到请求A的结果缓存，A和B必须同处一个context。

#### 合并请求collapsing
hystrix支持N个请求自动合并为一个请求，这个功能在有网络交互的场景下尤其有用，比如每个请求都要网络访问远程资源，如果把请求合并为一个，将使多次网络交互变成一次，极大节省开销。重要一点，两个请求能自动合并的前提是两者足够“近”，即两者启动执行的间隔时长要足够小，默认为10ms，即超过10ms将不自动合并。


#### 仪表盘
添加修改依赖
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
</dependency>
```
在启动的Main方法中加入**@EnableHystrixDashboard**启动Hystrix 仪表盘
访问http://localhost:8764/hystrix，出现Hystrix Dashboard界面，url填写http://localhost:8764/hystrix.stream，Title可以随意填写，点击Monitor Stream后，访问http://localhost:8764/client/feignHome 即可看到图标的出现。

还有其他参数：参考下方url
参考：
* [Hystrix使用入门手册](https://www.jianshu.com/p/b9af028efebb)
* Hystrix相关资料：[防雪崩利器：熔断器 Hystrix 的原理与使用](https://segmentfault.com/a/1190000005988895)

集群监控使用Turbine

### zuul（路由网关）
Zuul 是在云平台上提供动态路由,监控,弹性,安全等边缘服务的框架。Zuul 相当于是设备和 Netflix 流应用的 Web 网站后端所有请求的前门。路由功能是微服务的一部分，比如／api/user转发到到user服务，/api/shop转发到到shop服务。zuul默认和Ribbon结合实现了负载均衡的功能。

新建maven项目zuul-server
pom文件和之前一样，只是依赖需要修改为如下：
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
</dependency>
```
新建yml配置文件：
```yml
# eureka 服务配置
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    register-with-eureka: false
server:
  port: 8765
spring:
  application:
    name: zull-server
# zuul规则配置
zuul:
  routes:
    api:
      path: /api/**
      serviceId: eureka-discovery
```
启动之前eureka-server、eureka-discovery(分端口启动2次)、启动zull-server，访问http://localhost:8765/api/demo/home?name=whh 可以看到返回值为eureka-discovery提供。
zull规则可以配置eureka服务中心注册的serviceId、url、自定义处理等，同时也可以关闭某个服务。

#### zuul服务过滤
zuul可以通过定义filter做一些相关请求过滤，比如登陆拦截，指定参数拦截等。如下：
```java
@Component
public class DemoFilter extends ZuulFilter {

    /**
     * 返回一个字符串代表过滤器的类型，在zuul中定义了四种不同生命周期的过滤器类型
     * 为啥不是枚举
     * @return
     */
    @Override
    public String filterType() {
        /*
         * pre：路由之前
         * routing：路由之时
         * post：路由之后
         * error：发送错误调用
         */
        return "pre";
    }

    /**
     * 过滤的顺序
     * @return
     */
    @Override
    public int filterOrder() {
        return 0;
    }

    /**
     * 这里可以写逻辑判断，是否要过滤
     * @return
     */
    @Override
    public boolean shouldFilter() {
        return true;
    }

    /**
     * 过滤器的具体逻辑
     * @return
     */
    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        String token = request.getParameter("token");
        if (token == null){
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            try {
                ctx.getResponse().getWriter().write("token null, need login");
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return null;
    }
}
```
重新启动zuul-server
访问http://localhost:8765/api/demo/home?name=whh 会返回失败
访问http://localhost:8765/api/demo/home?name=asdfasdfasf&s=aaa&token=test 会正常返回。

### Spring Cloud Config（中心化配置）
在搭建微服务的时候，各种配置都是配置在本地，为了方便配置文件的管理，实时更新，所以需要使用分布式配置中心组件。在Spring Cloud Config中配置文件可以放到配置服务本地，也可以放入到远程仓库中，如git。

因为公司用的svn，网络上大量的git教程，此处采用公司svn作为远程仓库。svn目录结构为：configuration/beta/xxxx/application.properties

新建maven项目
pom文件依赖为：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<dependency>
    <groupId>org.tmatesoft.svnkit</groupId>
    <artifactId>svnkit</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```
新建application.yml
```yml
spring:
  application:
    name: config-server
  cloud:
    config:
      enabled: true
      server:
        svn:
          uri: http://xxxxx.com/svnuri/
          username: xuzhuo
          password: 123123.com
          default-label: ""
  profiles:
    active: subversion
server:
  port: 8766
```
在启动的main方法中新增注解**@EnableConfigServer**启动配置中心
建立自定义类处理svn DemoSvnRepository
```java
@Component
public class DemoSvnRepository extends SvnKitEnvironmentRepository {
    public DemoSvnRepository(ConfigurableEnvironment environment) {
        super(environment);
    }

    @Override
    public synchronized Environment findOne(String application, String profile, String label) {
        Environment environment = super.findOne(application, profile, label);
        Properties properties = new Properties();
        PropertySource propertySource = new PropertySource("profile", properties);
        environment.add(propertySource);

        File workingDirectory = this.getWorkingDirectory();
        String profilePath = workingDirectory.getPath() + "/" + application + "/" + profile;
        try (FileInputStream fileInputStream = new FileInputStream(profilePath)) {
            properties.load(fileInputStream);
        } catch (IOException e) {
            e.printStackTrace();
        }

        return environment;
    }

}
```
配置中心使用svn默认是先从svn下载最新配置文件，之后访问时读取本地配置文件。
启动服务，访问：http://localhost:8766/beta/xxxx/application.properties

* 在实际访问的时候会无法访问到数据，调试后会出现异常**HttpMediaTypeNotAcceptableException**。

新增WebConfigurer
```java
@Configuration
public class WebConfigurer extends WebMvcConfigurerAdapter {
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer.favorPathExtension(false);
    }
}
```
之后访问地址正常。

### Spring Cloud Bus 消息总线
Spring Cloud Bus 将分布式的节点用轻量的消息代理连接起来。它可以用于广播配置文件的更改或者服务之间的通讯，也可以用于监控。
