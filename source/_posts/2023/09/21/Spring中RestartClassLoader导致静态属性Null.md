---
title: Spring中RestartClassLoader导致静态属性Null
date: 2023-09-21 09:54:14
categories: ["Spring"]
tags: ["Bug", "ClassLoader"]
---

一次项目改造中，为了兼容一起获取配置代码，在 Spring 启动后，把`Environment`写入静态属性，其他地方需要获取值时，直接通过静态方法中从`Environment`内获取。

```java
public class SystemEnv {
    private static Environment environment;
    public static void setEnvironment(Environment environment) {
        SystemEnv.environment = environment;
    }
    public static String getProperty(String key) {
        return environment.getProperty(key);
    }
}
```

<!-- more -->

本地开发测试完成后，其他同事启动，直接`NullPointerException`，在其电脑上Debug时发现，当前`SystemEnv`的类加载器为`org.springframework.boot.devtools.restart.classloader.RestartClassLoader`，因为本地配置了devtool开发导致该类被加载了2次，第一次正常Spring流程启动后设置属性。第二次的时候由`RestartClassLoader`加载的类，未设置属性，获取时默认null。
> 本地同样参数启动，测试未能复现第二次由`RestartClassLoader`加载情况。未找到是否由其他配置导

解决办法：
获取属性的时候，判断当前类加载器是否`RestartClassLoader`，如果是则获取父类加载器。
```java
public class SystemEnv {
    private static Environment environment;
    public static void setEnvironment(Environment environment) {
        SystemEnv.environment = environment;
    }
    public static String getProperty(String key) {
        return getEnvironment().getProperty(key);
    }
    public static Environment getEnvironment() {
        //如果已经设置就返回
        if (environment != null) {
            return environment;
        }
        //判断类加载器，如果是RestartClassLoader获取父类加载器
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        if (Objects.equals(classLoader.getClass().getSimpleName(), "RestartClassLoader")) {
            classLoader = classLoader.getParent();
        }
        try {
            //因为Spring第一次启动会设置值，所以父加载器能获取到值
            Field field = classLoader.loadClass("com.rtmart.promotion.util.SystemEnv").getDeclaredField("environment");
            field.setAccessible(true);
            environment = (Environment) field.get(null);
        } catch (IllegalAccessException | NoSuchFieldException | ClassNotFoundException e) {
            throw new RuntimeException(e);
        }
        return environment;
    }
}
```

