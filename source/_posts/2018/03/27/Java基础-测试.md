---
title: Java基础-测试
date: 2018-03-27 13:52:21
categories: ['Java基础']
tags: ['基础', '测试']
---

在工作完成后，都需要对代码进行测试用例编写，如果采用TDD，那就更加离不开测试。
<!-- more -->
### JUnit
添加Maven依赖
```xml
<dependency>
  <groupId>junit</groupId>
  <artifactId>junit</artifactId>
  <version>4.12</version>
  <scope>test</scope>
</dependency>
```
* 添加scope为test，表面该依赖只是在测试的时候使用。

```java
//业务类
public class DemoService {
    public boolean demo(List<String> str) {
        return str.add("whh");
    }
}
//测试类
public class DemoServiceTest {
    @Test
    public void demo() throws Exception {
        List<String> list = new ArrayList<>();
        list.add("whh");
        list.add("whhxz");
        System.out.println(list);
    }
}
```
创建测试类，如上。
上述代码是通过IntelliJ IDEA生成测试类，为了规范，一般为需要测试的类后加上`Test`。通常会在测试代码中加上断言，IDEA在运行测试时，会自动加上`-ea`参数，断言失败时就会抛出异常。其他复杂方式可查看官方文档。

Junit核心在于`org.junit.runner.JUnitCore`。可以通过打印方法调用栈堆来分析。

#### SpringJunit
通常在项目中使用Spring框架，对代码进行测试时可以使用`spring-test`。
添加maven依赖
```xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-test</artifactId>
  <version>4.3.13.RELEASE</version>
  <scope>test</scope>
</dependency>
```
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath*:applicationContext.xml")
public class TimeReplenishTaskServiceTest {
    @Autowired
    @Qualifier("timeReplenishTaskService")
    private AbstractReplenishTask replenishTask;

    @Test
    public void create(){
      replenishTask.doSomethind();
    }
}
```
如上使用SpringTest，通过`RunWith`配置`SpringJUnit4ClassRunner`，指定由`SpringJUnit4ClassRunner`启动，`ContextConfiguration`用来配置环境，其他的注入就如同普通的Spring注入。
Junit允许通过`RunWith`改变默认的执行类，不然默认就是`org.junit.runners.Suite`。

### TestNG
添加maven依赖
```xml
<dependency>
  <groupId>org.testng</groupId>
  <artifactId>testng</artifactId>
  <version>6.8</version>
  <scope>test</scope>
</dependency>
```

```java
public class DemoServiceTest {

    @org.testng.annotations.Test
    public void demo() throws Exception {
        System.out.println("demo");
    }

    @org.testng.annotations.Test
    public void login() throws Exception {
        int i = 1 / 0;
    }

    @org.testng.annotations.Test(dependsOnMethods = {"login"})
    public void userInfo() throws Exception {
        System.out.println("userInfo");
    }
}
```
如上TestNG使用看起来和Junit一样，在TestNG中可以使用更多的功能，比如依赖。如上，userInfo就依赖于login测试，如果login失败，后续的userInfo就会直接跳过。
同时TestNG可以通过写xml文件来进行组合测试。
```xml
<!DOCTYPE suite SYSTEM "http://testng.org/testng-1.0.dtd" >
<suite name="Suite" thread-count="5" verbose="1" parallel="false">
    <test name="demoService">
        <classes>
            <class name="com.whh.netty.DemoServiceTest"/>
        </classes>
    </test>
</suite>
```
在xml中可以配置多个测试类进行整个流程的测试。

#### TestNG结合spring
```java
@org.testng.annotations.Test
@ContextConfiguration(locations = "classpath*:applicationContext.xml")
public class TimeReplenishTaskServiceTest extends AbstractTestNGSpringContextTests {
    @Autowired
    @Qualifier("timeReplenishTaskService")
    private AbstractReplenishTask replenishTask;

    @org.testng.annotations.Test
    public void create(){
      replenishTask.doSomethind();
    }
}
```
通过继承AbstractTestNGSpringContextTests来实现和Spring结合使用。

参考：[JUnit 4 与 TestNG 的对比](https://www.ibm.com/developerworks/cn/java/j-cq08296/)

### Mock框架
在开发过程中，可以依赖的某个功能未开发完成，就可以通过Mock框架来模拟对象替换部分功能来完成。

#### mockito
简单使用如下：
```java
public class DemoServiceTest {
    @Test
    public void demo() {
        List list = Mockito.mock(List.class);
        //设置当使用list.get(0)时返回whh
        Mockito.when(list.get(0)).thenReturn("whh");
        //调用list.get(0)
        String result = (String) list.get(0);
        //验证是否调用
        Mockito.verify(list).get(0);
        //判断返回值
        Assert.assertEquals("whh", result);

    }
}
```
使用的Junit做的测试。创建mock对象不能对final，Anonymous，primitive类进行mock。
同时还可以设置方法返回异常等操作。
其他操作查看：
[mockito github](https://github.com/mockito/mockito/wiki)
[mockito官网](http://site.mockito.org/)

同样的mock框架还有[jmockit](http://jmockit.github.io/)、[easymock](http://easymock.org/)

未完待续。。。
