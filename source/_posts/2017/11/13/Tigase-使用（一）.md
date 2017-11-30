---
title: Tigase 使用（一）
date: 2017-11-13 19:44:33
categories: ['Tigase']
tags: ['Tigase', 'Tigase插件', 'Tigase组件', 'HTTP文件上传']
---

### Tigase插件
Tigase插件被SessionManager组件和C2S所加载。

新建一个maven项目，添加tigase-server依赖
```xml
<dependency>
  <groupId>tigase</groupId>
  <artifactId>tigase-server</artifactId>
  <version>7.1.2</version>
</dependency>
```
复制之前教程idea启动Tigase中的etc目录到该maven项目根目录下。
项目中创建java文件DemoPlugin
```java
@Id(DEMO)
@Handles(
        @Handle(path = "message", xmlns = "jabber:client")
)
public class DemoPlugin extends AnnotatedXMPPProcessor implements XMPPProcessorIfc {
    protected static final String DEMO = "DEMO";

    @Override
    public void process(Packet packet, XMPPResourceConnection session, NonAuthUserRepository repo, Queue<Packet> results, Map<String, Object> settings) throws XMPPException {
        System.out.println("~~~~~~~" + packet.toString());
    }
}
```
AnnotatedXMPPProcessor表示可以通过注解配置插件。插件处理消息有：
* XMPPPreprocessorIfc - is the interface for packets pre-processing plugins.
* XMPPProcessorIfc - is the interface for packets processing plugins.
* XMPPPostprocessorIfc - is the interface for packets post-processing plugins.
* XMPPPacketFilterIfc - is the interface for processing results filtering.

process代码写习惯业务逻辑。注解表示该插件的唯一ID，以及需要处理指定的消息。
在配置文件init-mysql.properties中，添加--sm-plugins=+DEMO，如果已经有该配置，在后面添加+DEMO即可。这里+表示添加插件，默认为+，修改为-表示去除插件，可以在SessionManagerConfig类中查看默认加载的插件。

配置tigase.server.XMPPServer启动项目即可。

### 添加组件
项目中添加Java文件DemoComponent
```Java
public class DemoComponent extends AbstractMessageReceiver {
    private static final Logger log = Logger.getLogger(DemoComponent.class.getName());

    @Override
    public void processPacket(Packet packet) {
        if (packet.getTo() == null) {
            log.log(Level.WARNING, "目标为空: {0}", packet);
            return;
        }
        String xmlns = packet.getXMLNS();
        Element returnIq = new Element(Iq.QUERY_NAME);
        returnIq.setXMLNS(xmlns);
        if (packet instanceof Iq) {
            Iq iq = (Iq) packet;
            if ("demo:search".equals(iq.getIQXMLNS())) {
                try {
                    Class.forName("com.mysql.jdbc.Driver");
                    Connection connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/tigasedb?user=root&password=password");
                    Statement statement = connection.createStatement();
                    ResultSet result = statement.executeQuery("select user_id, user_pw from tig_users");


                    while (result.next()) {
                        Element item = new Element("item");
                        item.setAttribute("userId", result.getString("user_id"));
                        String userPw = result.getString("user_pw");
                        item.setAttribute("userPw", userPw == null ? "" : userPw);
                        returnIq.addChild(item);
                    }

                } catch (ClassNotFoundException | SQLException e) {
                    e.printStackTrace();
                }
            }
        }
        Iq iq = new Iq(returnIq, packet.getStanzaTo(), packet.getStanzaFrom());
        addOutPacket(iq);
    }
}
```
继承AbstractMessageReceiver实现组件消息处理。processPacket为相关处理逻辑。通过addOutPacket返回消息。

在配置文件init-mysql.properties中添加
```properties
--comp-name-10 = demo
--comp-class-10 =com.whh.tigase.DemoComponent
```
后面数字不要和现有重复即可。启动项目测试OK。

### Tigase通过HTTP实现文件上传
pom文件添加需要的依赖
```xml
<!-- 启动HTTP所需要依赖，否则无法启动HTTP服务-->
<dependency>
  <groupId>javax.servlet</groupId>
  <artifactId>javax.servlet-api</artifactId>
  <version>3.1.0</version>
</dependency>
<dependency>
  <groupId>commons-fileupload</groupId>
  <artifactId>commons-fileupload</artifactId>
  <version>1.3.3</version>
</dependency>
```
修改init-mysql.properties文件，添加HTTP组件。
```properties
--comp-name-1 = http
--comp-class-1 = tigase.http.HttpMessageReceiver
```

在项目根目录下添加script/rest/update文件夹。添加groovy文件UploadFile.groovy
```java
class UploadFile extends Handler{
    public UploadFile() {

        regex = /\//
        requiredRole = "user"
        decodeContent = false
        isAsync = false
        execPost = { Service service, callback, jid, HttpServletRequest request ->
            if (ServletFileUpload.isMultipartContent(request)){
                def servletFileUpload = new ServletFileUpload()
                def fileIterator = servletFileUpload.getItemIterator(request)
                while (fileIterator.hasNext()){
                    def fileItem = fileIterator.next()
                    InputStream is = fileItem.openStream()
                    def fileName = fileItem.getName()
                    Streams.copy(is, new FileOutputStream("./" + fileName), true)
                }
            }
            callback([fileupload: "success"])
        }

    }
}
```
通过实现tigase.http.rest.Handler 类添加rest接口。

### 常用入口
tigase.server.xmppsession.SessionManagerConfig：插件配置的插件
tigase.xmpp.ProcessorFactory：加载即获取插件

### 需要改进的地方

* 默认tigase是没有存储消息，需要额外手动存储
* 如果使用Spark作为客户端，可以客户端加密传输消息，如果服务端需要获取消息，需要手动解密
