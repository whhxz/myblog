---
title: SSM搭建讲解
date: 2017-09-25 15:01:32
categories: ["基础知识"]
tags: ["Spring", "SpringMVC", "Mybatis", "基础"]
---
项目中常用SpringMVC+Spring+Mybatis搭建项目。各个配置讲解。

### maven构建项目
步骤：
1. 通过maven(maven-archetype-webapp)创建空白项目
2. 添加所需要的目录
3. 配置maven使用的相关插件
* 使用maven-archetype-webapp可以快速创建web目录结构项目，省去自己创建相关目录
<!-- more -->
操作：
创建空白项目后，添加日志依赖slf4j-log4j12；因为是web项目，想要添加servlet相关依赖：javax.servlet.servlet-api、javax.servlet.jsp.jsp-api、jstl、javax.servlet-api(web 3.0使用)；
> 注意点：servlet-api和jsp-api这个scope需要设置为provided，因为项目部署的容器中，一般都有这两个jar包。

在resources目录下添加日志配置文件log4j.properties：
```properties
#简单配置日志
log4j.rootLogger=info, Console

#Console
log4j.appender.Console=org.apache.log4j.ConsoleAppender
log4j.appender.Console.layout=org.apache.log4j.PatternLayout
log4j.appender.Console.layout.ConversionPattern=%d [%t] %-5p [%c] - %m%n
```

添加插件org.apache.maven.plugins.maven-compiler-plugin、org.codehaus.mojo.tomcat-maven-plugin
```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.1</version>
  <configuration>
    <source>1.8</source>
    <target>1.8</target>
    <encoding>utf-8</encoding>
  </configuration>
</plugin>
<plugin>
  <groupId>org.codehaus.mojo</groupId>
  <artifactId>tomcat-maven-plugin</artifactId>
  <version>1.1</version>
  <configuration>
    <path>/</path>
    <port>8080</port>
    <uriEncoding>utf-8</uriEncoding>
    <server>tomcat7</server>
  </configuration>
</plugin>

```
* org.apache.maven.plugins.maven-compiler-plugin插件作用设置项目编译的jdk版本
* org.codehaus.mojo.tomcat-maven-plugin插件用于通过maven的tomcat插件直接启动项目。如果需要使用高版本tomcat，不建议使用该插件启动tomcat，该插件支持的tomcat插件版本较低。

启动项目，访问localhost:8080查看项目启动是否正常。

### 添加SpringMVC
步骤：
1. 配置web.xml
2. 配置SpringMVC配置文件
3. 启动访问Controller

操作：
添加springmvc依赖，pom文件修改添加依赖org.springframework.spring-webmvc，webmvc依赖了常用的jar包，包括bean、context、aop等。
在默认的web.xml中约束需要修改为web-app_2_5.xsd（如果不修改为2.4以上在jsp页面需要写<%@ page isELIgnored="false" %> $才会生效）。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://java.sun.com/xml/ns/javaee"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/"
         version="2.5">
```
配置servlet。

```xml
<!--配置Spring MVC-->
<servlet>
  <servlet-name>spring</servlet-name>
  <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
  <!--配置SpringMVC配置文件路径，如果不配置默认为WEB-INF下面${servlet-name}-servlet.xml-->
  <!--源码AbstractRefreshableConfigApplicationContext.getConfigLocations-->
  <!--
  <init-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:</param-value>
  </init-param>
  -->
  <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
  <servlet-name>spring</servlet-name>
  <url-pattern>/</url-pattern>
</servlet-mapping>
```
load-on-startup节点：
> * load-on-startup元素标记容器是否在启动的时候就加载这个servlet(实例化并调用其init()方法)
> * 它的值必须是一个整数，表示servlet应该被载入的顺序
> * 当值为0或者大于0时，表示容器在应用启动时就加载并初始化这个servlet
> * 当值小于0或者没有指定时，则表示容器在该servlet被选择时才会去加载
> * 正数的值越小，该servlet的优先级越高，应用启动时就越先加载
> * 当值相同时，容器就会自己选择顺序来加载

默认在WEB-INF添加spring-servlet.xml配置文件，配置spring扫描，配置视图解析器，配置静态资源，配置入出参UTF-8
```xml
<!--Spring MVC配置-->
<context:component-scan base-package="com.feiniu">

</context:component-scan>

<!-- 对模型视图名称的解析,即对模型视图名称添加前后缀 -->
<bean id="viewResolver"
      class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="prefix" value="/WEB-INF/jsp/"/>
    <property name="suffix" value=".jsp"/>
</bean>
<!--静态资源文件位置-->
<!--由SimpleUrlHandlerMapping处理静态资源-->
<mvc:resources mapping="/static/**" location="/static/"/>

<!--解决返回乱码-->
<mvc:annotation-driven>
    <!-- 消息转换器 -->
    <mvc:message-converters register-defaults="true">
        <bean class="org.springframework.http.converter.StringHttpMessageConverter">
            <property name="supportedMediaTypes" value="text/html;charset=UTF-8"/>
        </bean>
    </mvc:message-converters>
</mvc:annotation-driven>
```
添加Controller，添加jsp，添加js，看是否能正常访问。

#### Controller获取请求参数
> CookieValue：获取cookie的值
> ModelAttribute：表单提交封装对象
> PathVariable：获取路径中的参数，和开发rest风格api同用
> RequestBody：一般用户json请求
> RequestParam：request请求中的值
> SessionAttributes：session中的值
> 等其他注解

#### SpringMVC相关
SpringMVC原理是定义一个Servlet，配置请求路径，之后相关请求会全部走到配置的DispatcherServlet，请求的url匹配到相关的Controller，默认SpringMVC的Controller是单例的，使用的时候需要注意线程安全。
源码阅读入口：
1. SpringMVC启动：DispatcherServlet父类HttpServletBean.init（因为实现了Servlet，所以web容器会调用init初始化Servlet）
2. 请求到SpringMVC：doService

### 添加Spring配置
一般SpringMVC用于与页面数据交互，引入Spring控制后续相关操作，如事务，数据库，以及其他Spring用到的地方。在理论上所有配置配置到SpringMVC中效果也是一样的，在实际中SpringMVC相关配置和Spring相关配置拆分开，系统层次结构更加清晰。
步骤：
1. 配置web.xml启动加载spring
2. 添加Spring配置文件
3. 启动服务看是否正常

操作：
在web.xml中添加listener(ContextLoaderListener)
```xml
<!--配置加载Spring-->
<context-param>
  <param-name>contextConfigLocation</param-name>
  <param-value>classpath:applicationContext.xml</param-value>
</context-param>
<listener>
  <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```
> ContextLoaderListener实现了ServletContextListener，在服务启动的时候，容器会调用contextInitialized初始化Spring

在resources目录下添加Spring配置文件applicationContext.xml，配置Spring自动扫描
```xml
<context:component-scan base-package="com.feiniu">
</context:component-scan>
```
添加Service，启动服务，查看是否有异常。

> 在使用SpringMVC和Spring配置文件的时候，beans根节点上有个属性default-autowire="byName"，表示在设置bean的时候，自动装配实现。autowire有5种装配模式：
> * no：默认，不采用自动装配，需要在配置文件中通过ref标签注入
> * byName：通过属性的名称注入
> * byType：通过类型自动注入
> * constructor：通过构造方法注入，与byType不同在于，如果bean不存在会报错
> * default：采用父级标签配置
> 在平常使用注解@Autowired，和byType是一个意思，通过类型注入，如果有多个同类型Bean需要添加注解@Qualifier指定名称

#### Spring相关
在Spring容器概念，基本概念可以理解为Spring有一个工厂类用于创建和销毁bean，同时有个Map管理所有bean。延伸出了bean的生命周期、属性管理等其他相关操作。
源码阅读入口：
web.xml中阅读源码入口：ContextLoaderListener.contextInitialized
所有Spring不同容器的最终加载启动入口：AbstractApplicationContext.refresh

### 添加数据库配置
对于数据库操作，一般采用Mybatis。采用Mybatis方便统一管理所有sql，统一sql写在xml配置中，解除程序与sql耦合。方便维护对象和数据库映射关系，相对原生sql编写较简单，且不会过多影响性能。
在使用数据的时候，如果频繁创建、释放数据库连接，会产生大量的性能开销，引入数据库连接池，可以让数据库连接得到重用。在启动数据库连接池时，会初始化一部分数据库连接，对于业务而言，直接利用现有可用连接，避免连接初始花费时间。而且入数据库连接池，统一管理数据库连接，避免数据库连接泄露。（采用第三方数据源：druid）
* [spring-mybatis中文文档](http://www.mybatis.org/spring/zh/index.html)

步骤：
1. 配置数据库连接
2. 配置Mybatis
3. 启动服务

操作：
添加数据库依赖org.springframework.spring-jdbc、org.mybatis.mybatis、org.mybatis.mybatis-spring、mysql.mysql-connector-java、com.alibaba.druid
resources目录添加application-db.xml配置文件，在Spring配置文件中引入该文件
```xml
<!--配置数据库连接池-->
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
    <property name="url" value="jdbc:mysql://localhost:3306/test"/>
    <property name="username" value="test"/>
    <property name="password" value="password"/>
</bean>
<!--在Mybatis中每次操作都依赖于SqlSession，异常sql操作会话，通过SqlSessionFactoryBean，可以不用自己管理SqlSession-->
<bean id="sessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <!-- 配置打印Mybatis的sql，online请关闭 -->
    <!--<property name="configLocation" value="classpath:mybatis-config.xml" />-->
    <!--配置扫描sql位置，默认不配置在Mapper同目录-->
    <!--
    <property name="mapperLocations" value="classpath*:sqlmap/*-mapper.xml"/>
    -->
</bean>
<!--结合spring配置扫描Mapper-->
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <!--配置扫描Mapper所在目录-->
    <property name="basePackage" value="com.feiniu.mapper"/>
    <property name="sqlSessionFactoryBeanName" value="sessionFactory"/>
</bean>
```
mapper.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper>
</mapper>
```
添加Mapper和XML中的sql，启动服务，访问是否正常。
访问启动正常，但是在访问的时候会报错 *org.apache.ibatis.binding.BindingException: Invalid bound statement (not found): com.feiniu.mapper.DemoMapper.countDemo* ，这个时候可以看target编译的目录中，并没有mapper.xml，因为如果不额外配置的话，只有src/main/java下面的java文件编译为class，需要修改pom文件，设置src/main/java下面资源文件不过滤。如下：
```xml
<resources>
  <resource>
    <filtering>false</filtering>
    <directory>src/main/java</directory>
    <includes>
      <include>**/*.xml</include>
      <include>**/*.properties</include>
    </includes>
  </resource>
</resources>
```
### 配置事务
因为spring-jdbc中已经包含了事务所需要的jar包，所以不需要额外配置相关依赖
修改applicationContext-db.xml添加事务支持
```xml
<!--添加数据库是否操作-->
<bean id="transactionManager"
      class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
</bean>
<!--开启事务注解-->
<tx:annotation-driven transaction-manager="transactionManager" />
```
启动服务测试事务是否生效。启动服务正常，但是事务不会生效。
因为在事务等配置是写在Spring的配置文件中，在配置扫描的时候，Spring和SpringMVC都扫描到Controller和Service导致，所配置的扫描对象会在Controller和Service中个存在一份，用户访问时，先通过SpringMVC的servelt获取的MVC中对象，直到DAO中获取不到实例，从父容器中获取。而事务又是配置Spring容器中，导致事务失效。可以在Controller和Service中都注入ApplicationContext，比较下两边的容器。
解决办法：修改MVC容器只能加载Controller，Spring容器加载非Controller
spring-servlet.xml
```xml
<context:component-scan base-package="com.feiniu" use-default-filters="false">
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```
applicationContext.xml
```xml
<context:component-scan base-package="com.feiniu">
    <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```
修改后测试事务是否生效。
* 在使用注解Transactional时，如果不写rollbackFor，默认只回滚RuntimeException、Error，具体源码可以查看RuleBasedTransactionAttribute的父类DefaultTransactionAttribute中rollbackOn

#### 事务相关
事务特性：
> * 事务是一个原子操作，由一系列动作组成。事务的原子性确保动作要么全部完成，要么完全不起作用。
> * 一旦事务完成（不管成功还是失败），系统必须确保它所建模的业务处于一致的状态，而不会是部分完成部分失败。在现实中的数据不应该被破坏。
> * 可能有许多事务会同时处理相同的数据，因此每个事务都应该与其他事务隔离开来，防止数据损坏。
> * 一旦事务完成，无论发生什么系统错误，它的结果都不应该受到影响，这样就能从任何系统崩溃中恢复过来。通常情况下，事务的结果被写到持久化存储器中。

对于Spring而言，Spring并不直接管理事务，Spring提供了事务管理器接口PlatformTransactionManager，把事务管理职责委托给各个平台，入JDBC(org.springframework.jdbc.datasource.DataSourceTransactionManager)、Hibernate(org.springframework.orm.hibernate3.HibernateTransactionManager)等事务管理器。
Spring接口TransactionDefinition定义了5个事务隔离级别：
> * ISOLATION_DEFAULT（默认）：使用数据库默认的事务隔离级别.另外四个与JDBC的隔离级别相对应
> * ISOLATION_READ_UNCOMMITTED：这是事务最低的隔离级别，它充许别外一个事务可以看到这个事务未提交的数据。这种隔离级别会产生脏读，不可重复读和幻像读。
> * ISOLATION_READ_COMMITTED：保证一个事务修改的数据提交后才能被另外一个事务读取。另外一个事务不能读取该事务未提交的数据。这种事务隔离级别可以避免脏读出现，但是可能会出现不可重复读和幻像读。（MYSQL默认事务级别）
> * ISOLATION_REPEATABLE_READ：这种事务隔离级别可以防止脏读，不可重复读。但是可能出现幻像读。它除了保证一个事务不能读取另一个事务未提交的数据外，还保证了避免下面的情况产生(不可重复读)。
> * ISOLATION_SERIALIZABLE：这是花费最高代价但是最可靠的事务隔离级别。事务被处理为顺序执行。除了防止脏读，不可重复读外，还避免了幻像读。

Spring接口TransactionDefinition定义了7个事务传播行为：
> * PROPAGATION_REQUIRED：支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择，也是 Spring 默认的事务的传播。
> * PROPAGATION_SUPPORTS：如果存在一个事务，支持当前事务。如果没有事务，则非事务的执行。但是对于事务同步的事务管理器，PROPAGATION_SUPPORTS与不使用事务有少许不同。
> * PROPAGATION_MANDATORY：如果已经存在一个事务，支持当前事务。如果没有一个活动的事务，则抛出异常。
> * PROPAGATION_REQUIRES_NEW：总是开启一个新的事务。如果一个事务已经存在，则将这个存在的事务挂起。
> * PROPAGATION_NOT_SUPPORTED：总是非事务地执行，并挂起任何存在的事务。使用PROPAGATION_NOT_SUPPORTED,也需要使用JtaTransactionManager作为事务管理器。
> * PROPAGATION_NEVER：总是非事务地执行，如果存在一个活动事务，则抛出异常。
> * PROPAGATION_NESTED：如果一个活动的事务存在，则运行在一个嵌套的事务中. 如果没有活动事务, 则按TransactionDefinition.PROPAGATION_REQUIRED 属性执行。

在使用事务传播行为时，如果没有深刻理解，随意使用会导致出现意外情况。
### 配置properties配置文件
在项目中有很多基本数据是写在配置文件中，需要配置Spring读取配置文件，或者在Spring配置文件中有${}表达式时，通过配置文件设置值。减少每次需要修改代码。
resources目录下添加conf/local/application.properties配置文件
```properties
####################  mysql数据库  ######################
mysql.jdbc.driver=com.mysql.jdbc.Driver
mysql.jdbc.user=test
mysql.jdbc.password=password
mysql.jdbc.alias=feiniu_proxool
mysql.jdbc.url=jdbc:mysql://localhost:3306/test?characterEncoding=utf-8&useUnicode=true&zeroDateTimeBehavior=convertToNull&allowMultiQueries=true
mysql.jdbc.maximum-connection-count=20
mysql.jdbc.minimum-connection-count=1
mysql.jdbc.simultaneous-build-throttle=5
mysql.jdbc.verbose=true
mysql.jdbc.trace=true
mysql.jdbc.fatal-sql-exception=Fatal error
mysql.jdbc.prototype-count=5
mysql.jdbc.statistics-log-level=ERROR
mysql.jdbc.maximum-active-time=600000
mysql.jdbc.house-keeping-test-sql=SELECT CURRENT_DATE
#######################################################################

#################################  本地配置  #################################
fn.env=dev
##############################################################################
```
在applicationContext.xml中添加：
```xml
<context:property-placeholder location="classpath:/config/local/*.properties"/>
```
修改applicationContext-db.xml中配置的url、user、password
```xml
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
    <property name="url" value="${mysql.jdbc.url}"/>
    <property name="username" value="${mysql.jdbc.user}"/>
    <property name="password" value="${mysql.jdbc.password}"/>
</bean>
```
启动测试连接数据库是否正常。
如果需要在实例中注入配置文件中配置，可以使用注解@Value。
在公司里，启动时，把所有配置文件加载进一个类的静态属性中，这样保证了在应用任何地方可以随意读取配置。
添加类SystemEnv保存所有配置：
```java
public class SystemEnv {
    private static final Logger log = LoggerFactory.getLogger(SystemEnv.class);

    private final static Properties systemProperties = new Properties();

    public static void addProperty(Properties properties) {
        if (properties != null) {
            for (Object key : properties.keySet()) {
                if (key != null) {
                    if (systemProperties.keySet().contains(key)) {
                        log.error("系统Property配置项 {} 重复", key);
                    }
                    systemProperties.put(key, properties.get(key));
                }
            }
        }
    }

    public static String getProperty(String key, String defaultValue) {
        String value = systemProperties.getProperty(key);
        return value != null? value: defaultValue;
    }

    public static Properties getProperties() {
        return systemProperties;
    }
}
```
新增一个类PropertyPlaceholderConfigurerEx加载读取配置文件，继承PropertyPlaceholderConfigurer，覆盖父类方法processProperties，在Spring加载配置文件后，把配置文件加载进系统。
```java
public class PropertyPlaceholderConfigurerEx extends PropertyPlaceholderConfigurer {
    @Override
    protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess, Properties props) throws BeansException {
        super.processProperties(beanFactoryToProcess, props);
        SystemEnv.addProperty(props);
    }
}
```
删除之前applicationContext.xml中添加的加载properties文件。在resources下新增applicationContext-config.xml，在applicationContext.xml中引入该配置。
applicationContext-config.xml：
```xml
<bean class="com.feiniu.config.PropertyPlaceholderConfigurerEx">
    <property name="locations">
        <list>
            <!--覆盖默认配置-->
            <value>classpath:config/local/*.properties</value>
        </list>
    </property>
</bean>
```
启动测试是否正常。

* PropertyPlaceholderConfigurer类实现了BeanFactoryPostProcessor，在Spring启动的时候回调用所有实现了BeanFactoryPostProcessor接口的postProcessBeanFactory方法。

### 统一出入参数
一般项目中存在后台访问和对外API接口，对外API中一般要求统一入参、出参。
添加自定义消息转换器，统一处理返回参数格式。添加Gson依赖com.google.code.gson.gson。设置接口入参params={}，json出参{code:200, msg:"xxx", data:xxx}
步骤：
1. 编写GsonUtils
2. 添加统一出参Vo
3. 添加入参数验证
4. 配置自定义消息转换器

操作：
GsonUtils.java
```java
public class GsonUtils {
    private static final Gson gson = new GsonBuilder().create();
    private static final Gson gsonFormat = new GsonBuilder().setDateFormat("yyyy-MM-dd HH:mm:ss").create();

    public static Gson createGson(){
        return gson;
    }

    public static Gson createGsonFormat(){
        return gsonFormat;
    }

    public static <T> String toJson(T t){
        return gson.toJson(t);
    }

    public static <T> String toJsonFormat(T t){
        return gsonFormat.toJson(t);
    }

    public static <T> T fromJson(String json, Class<T> clazzT){
        return gson.fromJson(json, clazzT);
    }

    public static <T> T fromJson(String json , Type type){
        return gson.fromJson(json, type);
    }

    public static <T> T fromJsonFormat(String json, Class<T> clazzT){
        return gsonFormat.fromJson(json, clazzT);
    }

    public static <T> T fromJsonFormat(String json, Type type){
        return gsonFormat.fromJson(json, type);
    }
}
```
添加统一vo，ResponseBodyVo
```java
public class ResponseBodyVo {
    private Integer code;
    private String msg;
    private Object data;

    public ResponseBodyVo() {
    }

    public ResponseBodyVo(Integer code, String msg, Object data) {
        this.code = code;
        this.msg = msg;
        this.data = data;
    }

    public Integer getCode() {
        return code;
    }

    public void setCode(Integer code) {
        this.code = code;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }

    public Object getData() {
        return data;
    }

    public void setData(Object data) {
        this.data = data;
    }
}
```
自定义消息转换器GsonHttpMessageConverter基础抽象类AbstractHttpMessageConverter实现接口GenericHttpMessageConverter
```java
public class GsonHttpMessageConverter extends AbstractHttpMessageConverter<Object> implements GenericHttpMessageConverter<Object> {
    private static final Charset DEFAULT_CHARSET = Charset.forName("UTF-8");

    @Override
    protected boolean supports(Class<?> clazz) {
        return true;
    }

    @Override
    protected Object readInternal(Class<?> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
        String requestBody = StreamUtils.copyToString(inputMessage.getBody(), DEFAULT_CHARSET);
        String paramsData = URLDecoder.decode(requestBody.replace("params=", ""), "utf-8");
        Object requestVo = GsonUtils.fromJson(paramsData, TypeToken.get(clazz).getType());
        if (clazz.isArray()) {
            throw new HttpMessageNotReadableException("不支持集合泛型: ");
        } else if (requestVo instanceof Collection) {
            Collection<?> requests = (Collection<?>) requestVo;
            ValidationUtils.validate(requests, requests.size());
        } else {
            ValidationUtils.validate(requestVo);
        }
        return requestVo;
    }

    @Override
    protected void writeInternal(Object o, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
        ResponseBodyVo responseBodyVo;
        if (o instanceof ResponseBodyVo) {
            responseBodyVo = (ResponseBodyVo) o;
        } else {
            responseBodyVo = new ResponseBodyVo(200, "success", o);
        }
        outputMessage.getBody().write(GsonUtils.toJson(responseBodyVo).getBytes());

    }

    @Override
    public boolean canRead(Type type, Class<?> contextClass, MediaType mediaType) {
        return true;
    }

    @Override
    public Object read(Type type, Class<?> contextClass, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
        String requestBody = StreamUtils.copyToString(inputMessage.getBody(), DEFAULT_CHARSET);
        String paramsData = URLDecoder.decode(requestBody.replace("params=", ""), "utf-8");
        Object requestVo = GsonUtils.fromJson(paramsData, TypeToken.get(type).getType());
        if (requestVo instanceof Object[]){
            Object[] requests = (Object[]) requestVo;
            ValidationUtils.validate(requests, requests.length);
        } else if (requestVo instanceof Collection) {
            Collection<?> requests = (Collection<?>) requestVo;
            ValidationUtils.validate(requests, requests.size());
        } else {
            ValidationUtils.validate(requestVo);
        }
        return requestVo;
    }

    @Override
    public boolean canWrite(Type type, Class<?> clazz, MediaType mediaType) {
        return true;
    }

    @Override
    public void write(Object o, Type type, MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
        ResponseBodyVo responseBodyVo;
        if (o instanceof ResponseBodyVo) {
            responseBodyVo = (ResponseBodyVo) o;
        } else {
            responseBodyVo = new ResponseBodyVo(200, "success", o);
        }
        outputMessage.getBody().write(GsonUtils.toJson(responseBodyVo).getBytes());

    }
}

```
添加测试Controller
```java
@RequestMapping("api")
@RestController
public class ApiController {

    @RequestMapping("list")
    public List<DemoVo> list(@RequestBody SearchDemoVo[] searchDemoVo){
        List<DemoVo>list = new ArrayList<>();
        for (int i = 0; i < 5; i++) {
            DemoVo demoVo = new DemoVo();
            demoVo.setName(String.valueOf(i));
            list.add(demoVo);
        }
        return list;
    }

}
```
在使用消息转换器的时候，需要注意的是，入参采用@RequestBody注解，出参数采用@ResponseBody注解。才可以使用消息转换器。
在spring-servlet.xml中添加自定义消息转换器。
```xml
<bean class="com.feiniu.http.converter.GsonHttpMessageConverter">
    <property name="supportedMediaTypes">
        <list>
            <value>text/html;charset=UTF-8</value>
            <value>application/json</value>
            <value>application/x-www-form-urlencoded</value>
        </list>
    </property>
</bean>
```

### 全局异常处理、入参数校验
在开发过程中，一般需要统一处理相关异常，避免内部异常暴露在外。需要统一处理异常以及消息校验。
添加依赖hibernate-validator校验
步骤：
1. 添加依赖
2. 添加校验Util
3. 添加异常全局处理
4. 消息转换器添加异常返回

操作：
pom文件添加依赖
添加校验Util
```java
public class ValidationUtils {
    private static Validator validator = Validation.buildDefaultValidatorFactory().getValidator();

    public static <T>  void validate(T t) throws ValidationException {
        Set<ConstraintViolation<T>> set =  validator.validate(t);
        if(set.size()>0){
            StringBuilder validateError = new StringBuilder();
            for(ConstraintViolation<T> val : set){
                validateError.append(val.getMessage()).append(";");
            }
            throw new ValidationException(validateError.toString());
        }

    }
    public static <T>  void validate(Collection<T>ts, int size) throws ValidationException {
        if (ts == null || ts.isEmpty()){
            throw new IllegalArgumentException("参数为空");
        }
        if (ts.size() > size){
            throw new IllegalArgumentException("超出最大查询数量:" + size);
        }
        for (T t : ts) {
            validate(t);
        }
    }
}
```
添加异常处理
```java
@ControllerAdvice(basePackages = "com.feiniu.controller.api")
public class ApiControllerAdvice {

    @ExceptionHandler(Exception.class)
    @ResponseBody
    public ResponseBodyVo handleGlobalException(HttpServletRequest request, Exception ex){
        ex.printStackTrace();
        ResponseBodyVo responseBodyVo ;
        if (ex instanceof ValidationException){
            responseBodyVo = new ResponseBodyVo(500, "参数异常", ex.getMessage());
        } else {
            responseBodyVo = new ResponseBodyVo(500, "系统错误", "后台数据异常");
        }
        return responseBodyVo;
    }
}
```
修改消息转换器，添加入参验证，出参数处理。
验证消息转换器是否正常。注意验证集合、数组、泛型、对象等。
* 在使用消息转换器的时候，如果不做特别判断，默认会处理所有的消息，有时候项目中包含API和后台模块，会导致后台ajax请求也会封装参数，所以在消息转换的时候判断是否需要转换消息。

### 本地Junit测试
在一般功能模块开发完成后，需要在本地进行测试，可以采用spring-test加Junit测试
在pom文件中添加spring-test和Junit相关依赖，设置scope为test。
举例创建测试文件DemoServiceTest测试DemoService
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath*:applicationContext.xml")
@WebAppConfiguration
public class DemoServiceTest {
    @Autowired
    private DemoService demoService;
    @Test
    public void dbCount() throws Exception {
        int i = demoService.dbCount();
        System.out.println("```````````" + i);
        assert i != 0;
    }

    @Test
    @Rollback
    @Transactional
    public void insertDemo() throws Exception {
        int i = demoService.insertDemo();
        assert i == 1;
    }

}
```
### 分环境打包项目
在项目启动后一般需要区分不同的环境，比如local、dev、beta、preview、online，各个环境配置不一样，需要分开读取配置文件。
步骤：
1. 修改pom文件分环境
2. 修改启动配置文件

操作：
pom根目录下添加
```xml
<profiles>
    <profile>
      <activation>
          <activeByDefault>true</activeByDefault>
      </activation>
        <id>default</id>
        <properties>
            <config.path>file:服务器路径</config.path>
        </properties>
    </profile>
    <profile>
        <id>local</id>
        <properties>
            <config.path>classpath:config/local</config.path>
        </properties>
    </profile>
    <profile>
        <id>dev</id>
        <properties>
            <config.path>classpath:config/dev</config.path>
        </properties>
    </profile>
    <profile>
        <id>beta</id>
        <properties>
            <config.path>classpath:config/beta</config.path>
        </properties>
    </profile>
</profiles>
```
在打包的指定环境即可选择需要的包

在pom文件resources下添加
```xml
<resource>
    <filtering>true</filtering>
    <directory>src/main/resources</directory>
    <includes>
        <include>applicationContext-config.xml</include>
        <include>checksrv.properties</include>
    </includes>
</resource>
<resource>
  <filtering>false</filtering>
  <directory>src/main/resources</directory>
  <includes>
    <include>**/*.xml</include>
    <include>**/*.properties</include>
  </includes>
</resource>

```
打包的时候替换指定参数

修改applicationContext-config.xml
```xml
<bean class="com.feiniu.config.PropertyPlaceholderConfigurerEx">
    <property name="locations">
        <list>
            <value>${config.path}/*.properties</value>
        </list>
    </property>
</bean>
```
测试打包是否正常
