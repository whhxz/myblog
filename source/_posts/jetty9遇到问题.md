title: jetty9遇到问题
date: 2015-02-06 17:18:21
tags: jetty9 LoginService  port
categories: 编程
---

## 端口修改

在jetty9中修改$JETTY_HOME/start.d/http.ini中修改

``` bash
jetty.port=9999
http.timeout=30000
```

## jetty9中原来jar

因为在部署项目时，为了避免war包过大，使用在生成war包的时候，不吧依赖jar添加进去，在jetty使用中，一般是一个jetty使用一个项目，所以需要把依赖jar包放入jetty目录 **$JETTY_HOME/lib/ext** 中
<!-- more -->
## IllegalStateException: No LoginService

因为在web.xml中添加了权限验证，所以在jetty中也需要相应的配置，在**$JETTY_HOME/etc/jetty.xml**中添加：
``` xml
    <Call name="addBean">
      <Arg>
        <New class="org.eclipse.jetty.security.HashLoginService">
          <Set name="name">Preauth Realm</Set>
          <Set name="config"><SystemProperty name="jetty.home" default="."/>/etc/realm.properties</Set>
          <Set name="refreshInterval">0</Set>
        </New>
      </Arg>
    </Call>
```

注意：<Set name="name">Preauth Realm</Set>这个要和web.xml中<realm-name>配置一致

## jar包冲突

添加在jetty9中使用的是asm-4.1.jar，如果在项目中有使用，需要删除

## 找不到npn-1.7.0_51.mod

从其他地方copy这个文件到$JETTY_HOME/modules/npn中就可
