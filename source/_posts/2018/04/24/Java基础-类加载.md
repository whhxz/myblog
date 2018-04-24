---
title: Java基础-类加载
date: 2018-04-24 09:49:43
categories: ['Java基础']
tags: ['JVM', '类加载']
---

在编译Java文件生成Class文件最终都需要加载到虚拟机中才能使用。虚拟机把Class文件加载到内存，并对数据进行校验、转换解析、初始化，最终形成可以被虚拟机直接使用的Java类型。

类从被加载到卸载，生命周期如下：
加载 ---> 验证 ---> 准备 ---> 解析 ---> 初始化 ---> 使用 ---> 卸载

其中验证、准备、解析三个节点称为连接。

* 加载：找到Class加载到内存中，生成一个代表改类的Class对象（不同的虚拟机实现可不一样）。
* 验证：校验Class中字节流符合当前虚拟机要求，主要包括文件格式验证、元数据验证、字节码验证、符号引用验证（NoSuchFieldError，NoSuchMethodError）。
* 准备：为类变量分配内存初始化变量的初始值（只是赋值初始值，并不是赋值准确的值，如static a=1，此处static=0，之后初始化在设置static=1）此处不包含final修饰的static，因为final修饰的static在编译时就会分配。
* 解析：主要将常量池中符号引用（用于描述所引用的目标，目标不一定已经加载到内存中）替换为直接引用的过程。虚拟机要求在执行anewarray、checkcast、getfield、getstatic、instanceof、invokeinterface、invokespecial、invokestatic、invokevirtual、multianewarray、new、putfield、putstatic这13操作符之前对所使用的符号引用进行解析即可。
* 初始化：类加载最后阶段，如果类具有父类，向上初始化，执行静态代码块已经初始化静态属性。

上述中加载、验证、准备、初始化、卸载是确定的顺序，解析并不一定在上述所在的顺序，在有些情况下，解析可以在初始化之后，这是为了支持Java运行时绑定。

<!-- more -->

### 类加载器
类加载器只是用来实现类的加载动作，其他的验证、准备、解析、初始化都是有虚拟机内部完成。
自定义类加载器如下：
```java
class Person {
    String name;
}

public class Main {

    public static void main(String[] args)throws Exception {
        ClassLoader classLoader = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                String fileName = name.substring(name.lastIndexOf(".") + 1) + ".class";
                InputStream in = getClass().getResourceAsStream(fileName);
                if (in == null) {
                    return super.loadClass(name);
                }
                try {
                    byte[] bytes = new byte[in.available()];
                    in.read(bytes);
                    return defineClass(name, bytes, 0, bytes.length);
                } catch (IOException e) {
                    e.printStackTrace();
                }
                return super.loadClass(name);
            }
        };
        Class<?> clazz = classLoader.loadClass("com.whh.netty.Person");
        System.out.println(clazz);//com.whh.netty.Person
        System.out.println(clazz == Person.class);//false
        System.out.printf("clazz class loader: %s\n", clazz.getClassLoader());//clazz class loader: com.whh.netty.Main$1@372f7a8d
        System.out.printf("Person class loader: %s\n", Person.class.getClassLoader());//Person class loader: sun.misc.Launcher$AppClassLoader@14dad5dc
    }
}
```
上述中使用自定义类加载器生成的class和直接使用该类并不相等，因为是有不同的类加载器所加载。

在JVM中提供了3种类加载器：Bootstrap类加载器、Extension类加载器、System类加载器。
#### Bootstrap类加载器
Bootstrap类加载器主要用于加载JVM自身需要的类，改类加载器主要使用C++实现，是JVM自身的一部分。服装架加载`$JAVA_HOME/lib`下核心类库或者`-Xbootclasspath`参数指定路径下的jar包。为了安全起见Bootstrap类加载器值加载包名为java、javax、sun等开头的类。

#### Extension类加载器
`sun.misc.Launcher.ExtClassLoader`是Extension类加载器的实现类，负责加载`$JAVA_HOME/lib/ext`目录下或者由`-Djava.ext.dir`指定路径中的包。继承关系如下：
![](http://otxnth5wx.bkt.clouddn.com/20180424屏幕快照2018-04-24上午10.46.26.png)

#### System类加载器
`sun.misc.Launcher.AppClassLoader`是System类加载器的实现类，负责加载类`java-classpath`或`-D java.class.path`指定路径下的类。通过`ClassLoader.getSystemClassLoader()`可以获取该类加载器.

在平常使用时，几乎都是由上述3中类加载器加载类。在JVM中，JVM对class加载是按需加载，使用才会被加载，import并不会加载。

### 双亲委派模式
对于双亲委派模式，如果一个类加载器需要加载一个类，并不是直接加载类，而是由把该请求提交给其父类加载器（此处父类加载器并不是继承关系，是组合关系），如果父类加载器还有父类加载器就一直提交，顶层Bootstrap类加载器，如果其中父类加载器可以完成加载任务，那么就成功返回，如果所有父类加载都无法完成加载，那么由子类完成加载。在使用双亲委派模式下，除了Bootstrap没有父类加载器，其他类加载器都必须有父类加载器。（源码可查看java.lang.ClassLoader#loadClass(java.lang.String, boolean)，先通过父类加载）

优点在于，Java类随着类加载器一起具备了优先级的层级关系，避免了基础类被随意加载破坏程序的稳定性。

虚拟机在使用内加载器时会调用`loadClassInternal`方法，该方法再调用`loadClass`。所以重写`loadClass`即可实现自定义类加载器。但是因为双亲委派模式核心就在`loadClass`中，所以如果想不破坏双亲委派模式就不建议覆盖`loadClass`方法，可以通过覆盖`findClass`方法来自定义类加载器。
修改之前例子，把重新`loadClass`修改为重写`findClass`就会发现，自定义类加载器加载的Person比较返回true。

在使用自定义类加载器时，通过debug模式下，可以获取父类加载
自定义类加载器 --parent-> AppClassLoader
AppClassLoader --parent-> ExtClassLoader
ExtClassLoader --parent-> null

上述自定义类加载器可以通过分析`java.lang.ClassLoader`源码得知。

* 在使用双亲委派模式时，如果希望获取两个不同Class对象，一种方法是重写`loadClass`不使用双亲委派模式，一种是直接使用`findClass`，绕过双亲委派，一般在热部署的时候使用。

### 线程上下文类加载器
在使用SPI时，Java提供接口，由第三方实现接口，如JDBC、JNDI等，Java提供的接口存在于`rt.jar`中，由`Bootstrap类加载器`加载，第三方包通常放在classpath中，由`System类加载器`加载。如果有`rt.jar`中SPI调用子类实现方法，双亲委派模式的原因，无法使用`System类加载器`加载的类。

在平常使用jdbc时代码如下：
```java
public class Main {
    public static void main(String[] args)throws Exception {
        String url = "jdbc:mysql://localhost:3306/test";
        Connection conn = java.sql.DriverManager.getConnection(url, "root", "password");
        PreparedStatement preparedStatement = conn.prepareStatement("select * from item where status = ?");
        preparedStatement.setInt(1, 1);
        ResultSet resultSet = preparedStatement.executeQuery();
        while (resultSet.next()){
            System.out.println(resultSet.getString("sku_code"));
        }
        conn.close();
    }
}
```
上述代码中没有和以前使用`Class.forName("com.mysql.jdbc.Driver")`，因为`DriverManager`中使用SPI注册了Mysql驱动，而且就是使用了线程上下文类加载器实现。
```java
//java.sql.DriverManager
static {
    loadInitialDrivers();
    println("JDBC DriverManager initialized");
}
//加载driver
private static void loadInitialDrivers() {
    //...
    AccessController.doPrivileged(new PrivilegedAction<Void>() {
        public Void run() {
            //创建serviceLoader
            ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
            Iterator<Driver> driversIterator = loadedDrivers.iterator();
            try{
                while(driversIterator.hasNext()) {
                    driversIterator.next();
                }
            } catch(Throwable t) {
            // Do nothing
            }
        }
    }
    //...
}
//java.util.ServiceLoader#load(java.lang.Class<S>)
public static <S> ServiceLoader<S> load(Class<S> service) {
    //获取线程中的classload（sun.misc.Launcher.AppClassLoader），并创建值，这里设置的classLoader在后续会使用到
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}


//java.util.ServiceLoader.LazyIterator#hasNextService
private boolean hasNextService() {
    //
    if (nextName != null) {
        return true;
    }
    if (configs == null) {
        try {
            //得的Driver路径： META-INF/services/java.sql.Driver
            //在mysql driver 5.1.40版本中该文件：
            /*
             *  com.mysql.jdbc.Driver
             *  com.mysql.fabric.jdbc.FabricMySQLDriver
             */
            String fullName = PREFIX + service.getName();
            if (loader == null)
                configs = ClassLoader.getSystemResources(fullName);
            else
                configs = loader.getResources(fullName);
        } catch (IOException x) {
            fail(service, "Error locating configuration files", x);
        }
    }
    //存储上述文件中值
    while ((pending == null) || !pending.hasNext()) {
        if (!configs.hasMoreElements()) {
            return false;
        }
        pending = parse(service, configs.nextElement());
    }
    nextName = pending.next();
    return true;
}
//java.util.ServiceLoader.LazyIterator#nextService
private S nextService() {
    if (!hasNextService())
        throw new NoSuchElementException();
    //com.mysql.jdbc.Driver、com.mysql.fabric.jdbc.FabricMySQLDriver
    String cn = nextName;
    nextName = null;
    Class<?> c = null;
    try {
        //通过之前设置的sun.misc.Launcher.AppClassLoader获取class
        //因为这里使用Class.forName，所以在最初使用的时候，就不需要手动加载
        //注册Driver到DriverManager中
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        fail(service,
                "Provider " + cn + " not found");
    }
    if (!service.isAssignableFrom(c)) {
        fail(service,
                "Provider " + cn  + " not a subtype");
    }
    try {
        S p = service.cast(c.newInstance());
        providers.put(cn, p);
        return p;
    } catch (Throwable x) {
        fail(service,
                "Provider " + cn + " could not be instantiated",
                x);
    }
    throw new Error();          // This cannot happen
}
```
在上述代码中`java.util.ServiceLoader.LazyIterator#nextService`在`rt.jar`中，是由`Bootstrap类加载器`加载，因为要使用`System类加载器`中的类，所以通过`getContextClassLoader`获取`sun.misc.Launcher.AppClassLoader`来加载`classpath`中的类。

在之后`connection`中，会通过获取`DriverManager`中注册的`registeredDrivers`，重新通过当前类`调用类的ClassLoader`或者当前线程的`getContextClassLoader`来加载一次`Driver`同时判断本次加载的`Driver`和之前注册的`Driver`是否相同，由同一个类加载器所加载。

### Tomcat类加载
在tomcat中一般可以部署多个应用，不同应用如果依赖了同一个jar不同版本，这样就需要使用不同的类加载器来隔离应用。同时如果不同应用依赖相同的版本，也可以把依赖的jar放入共有类加载器中。 

Tomcat服务器类加载器如下：
![](http://otxnth5wx.bkt.clouddn.com/2018042420160925001518808.png)

在Tomcat中需要在conf/catalina.properties中配置`server.loader`和`share.loader`后才会建立`CatalinaClassLoader`和`SharedClassLoader`，否则使用`CommonClassLoader`代替。因为默认配置中没有所以合并后变成了`lib`目录，如果需要可以建立、`common`、`server`、`shared`目录分别对应`CommonClassLoader`、`CatalinaClassLoader`、`SharedClassLoader`。

如果项目依赖于Spring，应用放入`webapps`中，依赖的jar包在`WEB-INF/lib`下。这时依赖的jar是由`WebappClassLoader`加载，不同的依赖由不同的`WebappClassLoader`加载，这样直接做到了应用的隔离。

如果依赖的Spring在common、server等目录下，那么Spring隔离方式如下：
```java
//org.springframework.web.context.ContextLoader#initWebApplicationContext
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
    //...
    //直接获取当前线程中的ClassLoader
    ClassLoader ccl = Thread.currentThread().getContextClassLoader();
    //如果当前线程中的classLoader和ContextLoader的classLoader一致，表示spring依赖在各自的目录中
    if (ccl == ContextLoader.class.getClassLoader()) {
        currentContext = this.context;
    }
    //如果不一致，那么吧当前线程的classLoader和当前context放入map中保存
    else if (ccl != null) {
        currentContextPerThread.put(ccl, this.context);
    }
    //...
}
//org.springframework.web.context.ContextLoader#getCurrentWebApplicationContext
//获取当前context
public static WebApplicationContext getCurrentWebApplicationContext() {
    ClassLoader ccl = Thread.currentThread().getContextClassLoader();
    if (ccl != null) {
        //先从之前保存的map中获取context
        WebApplicationContext ccpt = currentContextPerThread.get(ccl);
        if (ccpt != null) {
            return ccpt;
        }
    }
    return currentContext;
}
```
对于Spring而言，Spring中类加载器是通过当前线程中ClassLoader和ContextLoader.class.getClassLoader()比较来判断，是否属于同一个类加载器。

参考：
* 深入理解Java虚拟机