---
title: Java设计模式-职责链模式（Chain of Responsibility Pattern）
date: 2017-10-20 10:04:02
categories: ['设计模式']
tags: ['设计模式', '职责链模式', '职责链模式（Chain of Responsibility Pattern）']
---

### 职责链模式（责任链模式）
> 避免请求发送者与接收者耦合在一起，让多个对象都有可能接收请求，将这些对象连接成一条链，并且沿着这条链传递请求，直到有对象处理它为止。职责链模式是一种对象行为型模式。

职责链模式结构的核心在于引入了一个抽象处理者。UML类图如下：
![](http://otxnth5wx.bkt.clouddn.com/20171020屏幕快照2017-10-20上午10.53.12.png)

*  Handler（抽象处理者）：它定义了一个处理请求的接口，一般设计为抽象类，由于不同的具体处理者处理请求的方式不同，因此在其中定义了抽象请求处理方法。因为每一个处理者的下家还是一个处理者，因此在抽象处理者中定义了一个抽象处理者类型的对象（如结构图中的successor），作为其对下家的引用。通过该引用，处理者可以连成一条链。
* ConcreteHandler（具体处理者）：它是抽象处理者的子类，可以处理用户请求，在具体处理者类中实现了抽象处理者中定义的抽象请求处理方法，在处理请求之前需要进行判断，看是否有相应的处理权限，如果可以处理请求就处理它，否则将请求转发给后继者；在具体处理者中可以访问链中下一个对象，以便请求的转发。

在职责链模式里，很多对象由每一个对象对其下家的引用而连接起来形成一条链。请求在这个链上传递，直到链上的某一个对象决定处理此请求。发出这个请求的客户端并不知道链上的哪一个对象最终处理这个请求，这使得系统可以在不影响客户端的情况下动态地重新组织链和分配责任。

```Java
abstract class Handler{
    protected Handler successor;

    public void setSuccessor(Handler successor) {
        this.successor = successor;
    }

    abstract void handleRequest(String request);

}

class ConcreteHandlerA extends Handler{

    @Override
    void handleRequest(String request) {
        if("满足A".equals(request)){
            System.out.println("A处理");
        } else {
            successor.handleRequest(request);
        }
    }
}

class ConcreteHandlerB extends Handler{

    @Override
    void handleRequest(String request) {
        if ("满足B".equals(request)){
            System.out.println("B处理");
        } else {
            successor.handleRequest(request);
        }
    }
}
```
此处属于最简单的职责链，实际使用过程中，可能并没有怎么简单。

在使用职责链的时候分为纯职责链、不纯的职责链模式
* 纯职责链：一个纯的职责链模式要求一个具体处理者对象只能在两个行为中选择一个：要么承担全部责任，要么将责任推给下家，不允许出现某一个具体处理者对象在承担了一部分或全部责任后又将责任向下传递的情况。而且在纯的职责链模式中，要求一个请求必须被某一个处理者对象所接收，不能出现某个请求未被任何一个处理者对象处理的情况。
* 不纯的职责链：在一个不纯的职责链模式中允许某个请求被一个具体处理者部分处理后再向下传递，或者一个具体处理者处理完某请求后其后继处理者可以继续处理该请求，而且一个请求可以最终不被任何处理者对象所接收。

### 在java web中应用
在java web中filter就使用了职责链模式。
这里模仿filter实现：
```java

/**
 * 这里request、response、filterChain原本是使用的接口，这里使用的是类，为了简化模型
 * request请求
 */
class Request {
    private String header;
    private String body;

    public Request(String header, String body) {
        this.header = header;
        this.body = body;
    }

    public String getHeader() {
        return header;
    }

    public void setHeader(String header) {
        this.header = header;
    }

    public String getBody() {
        return body;
    }

    public void setBody(String body) {
        this.body = body;
    }
}

/**
 * response请求
 */
class Response {
    private String header;
    private String body;

    public Response(String header, String body) {
        this.header = header;
        this.body = body;
    }

    public String getHeader() {
        return header;
    }

    public void setHeader(String header) {
        this.header = header;
    }

    public String getBody() {
        return body;
    }

    public void setBody(String body) {
        this.body = body;
    }
}

/**
 * 拦截器
 */
abstract class Filter {
    //指定拦截url
    protected String urlPattern;

    public Filter(String urlPattern) {
        this.urlPattern = urlPattern;
    }

    /**
     * 拦截
     *
     * @param request
     * @param response
     * @param filterChain
     */
    abstract void doFilter(Request request, Response response, FilterChain filterChain);

    protected boolean valid(String str) {
        return str.startsWith(urlPattern);
    }
}

/**
 * 避免Xss攻击，消息转义
 */
class XSSFilter extends Filter {
    public XSSFilter(String urlPattern) {
        super(urlPattern);
    }

    @Override
    public void doFilter(Request request, Response response, FilterChain filterChain) {
        System.out.println("消息转义拦截器处理！！！");
        request.setBody(request.getBody()
                .replaceAll("<", "&lt;")
                .replaceAll(">", "&gt;")
                .replaceAll("&", "&amp;"));
        filterChain.doFilter(request, response);
    }

}

/**
 * 登陆拦截器
 */
class LoginFilter extends Filter {
    public LoginFilter(String urlPattern) {
        super(urlPattern);
    }

    @Override
    public void doFilter(Request request, Response response, FilterChain filterChain) {
        System.out.println("登陆拦截器处理！！！");
        filterChain.doFilter(request, response);
    }
}

/**
 * API接口拦截器
 */
class ApiFilter extends Filter {
    public ApiFilter(String urlPattern) {
        super(urlPattern);
    }

    @Override
    public void doFilter(Request request, Response response, FilterChain filterChain) {
        System.out.println("API接口拦截器处理！！！");
        filterChain.doFilter(request, response);
    }
}

/**
 * 过滤器链
 */
class FilterChain {
    //所有过滤器
    private List<Filter> filters = new ArrayList<>();
    private Iterator<Filter> filterIterator = null;

    public boolean addFilter(Filter filter) {
        return filters.add(filter);
    }

    //过滤器开始执行
    public void doFilter(Request request, Response response) {
        if (filterIterator == null) {
            filterIterator = filters.iterator();
        }
        if (filterIterator.hasNext()) {
            filterIterator.next().doFilter(request, response, this);
        }
    }

}

public class Main {
    public static void main(String[] args) {
        //配置拦截器
        XSSFilter xssFilter = new XSSFilter("/");
        LoginFilter loginFilter = new LoginFilter("/user/");
        ApiFilter apiFilter = new ApiFilter("/api");
        List<Filter> filters = new ArrayList<>();
        filters.add(xssFilter);
        filters.add(loginFilter);
        filters.add(apiFilter);


        String url = "/user/xxxx";
        Request request = new Request(url, "<script>alert(1)</script>");
        //通过URL生产拦截器链
        FilterChain filterChain = new FilterChain();
        filters.forEach(filter -> {
            if (filter.valid(request.getHeader())){
                filterChain.addFilter(filter);
            }
        });
        filterChain.doFilter(request, null);
        System.out.println(request.getBody());
    }
}
```
UML类图：
![](http://otxnth5wx.bkt.clouddn.com/20171020屏幕快照2017-10-20下午1.56.52.png)
代码中通过请求URL来获取需要的拦截器组成拦截器链，在实际web中url-pattern配置FilterConfig。

在spring中FilterChain实现有多种，有维护List（集合）、维护nextFilterChain（链表）。
### 拦截器总结
在平常项目中，大量使用了else if，每个else if中有大量逻辑等，可以改造成职责链模式。
如：
```java
//当前可以参加的所有套装活动
if ("单品多件".equals(type)){
    //单品多件筛选逻辑
}else if ("固定搭配".equals(type)){
    //固定搭配筛选逻辑
} else if ("自由组合".equals(type)){
    //自由组合筛选逻辑
}
```
这样每次通过添加else if判断处理相关内容，业务变更，后续代码添加，导致代码特别长，无法灵活使用。

职责链模式通过建立一条链来组织请求的处理者，请求将沿着链进行传递，请求发送者无须知道请求在何时、何处以及如何被处理，实现了请求发送者与处理者的解耦。

**应用场景：**
* 有多个对象可以处理同一个请求，具体哪个对象处理该请求待运行时刻再确定，客户端只需将请求提交到链上，而无须关心请求的处理对象是谁以及它是如何处理的。
* 在不明确指定接收者的情况下，向多个对象中的一个提交一个请求。
* 可动态指定一组对象处理请求，客户端可以动态创建职责链来处理请求，还可以改变链中处理者之间的先后次序。

**优点：**
* 职责链模式使得一个对象无须知道是其他哪一个对象处理其请求，对象仅需知道该请求会被处理即可，接收者和发送者都没有对方的明确信息，且链中的对象不需要知道链的结构，由客户端负责链的创建，降低了系统的耦合度。
* 请求处理对象仅需维持一个指向其后继者的引用，而不需要维持它对所有的候选处理者的引用，可简化对象的相互连接。
* 在给对象分派职责时，职责链可以给我们更多的灵活性，可以通过在运行时对该链进行动态的增加或修改来增加或改变处理一个请求的职责。
* 在系统中增加一个新的具体请求处理者时无须修改原有系统的代码，只需要在客户端重新建链即可，从这一点来看是符合“开闭原则”的。

**缺点：**
* 由于一个请求没有明确的接收者，那么就不能保证它一定会被处理，该请求可能一直到链的末端都得不到处理；一个请求也可能因职责链没有被正确配置而得不到处理。
* 对于比较长的职责链，请求的处理可能涉及到多个处理对象，系统性能将受到一定影响，而且在进行代码调试时不太方便。
* 如果建链不当，可能会造成循环调用，将导致系统陷入死循环。
