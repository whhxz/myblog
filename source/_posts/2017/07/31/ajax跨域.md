---
title: ajax跨域
date: 2017-07-31 15:35:32
categories: ['js']
tags: ['ajax', '跨域']
---

在业务需求中，有时候需要使用js跨域请求。解决办法有使用jsonp和CORS。

* jsonp利用的是在页面访问资源文件时，访问非当前域名下其他域名资源。如`<img>`、`<script>`、`<form>`等标签。可以利用这一特性跨域。

前端请求：
```
var scriptJsonP = document.createElement("script");
scriptJsonP.setAttribute("type", "text/javascript");
scriptJsonP.setAttribute("src", "http://a.com/corssOrigin/jsonp");
document.getElementsByTagName("body")[0].appendChild(scriptJsonP);

function jsonp(data){
  alert(data)
}
```
<!-- more -->
后台服务器：
```
@Controller
@RequestMapping("corssOrigin")
public class CrossOriginController {
    @RequestMapping("jsonp")
    @ResponseBody
    public String jsonp(HttpServletRequest request){
        return "jsonp({test:'test'})";
    }
}
```
调用成功后会调用jsonp方法。完成一次跨域请求。

使用jquery：
```
$.ajax({
           type: "get",
           async: false,
           url: "http://a.com/corssOrigin/jsonp",
           dataType: "jsonp",
           jsonp: "callback",//传递给请求处理程序或页面的，用以获得jsonp回调函数名的参数名(一般默认为:callback)
           jsonpCallback:"jsonp",//自定义的jsonp回调函数名称，默认为jQuery自动生成的随机函数名，也可以写"?"，jQuery会自动为你处理数据
           success: function(json){
             alert(json);
           },
           error: function(){
           }
       });
})
```

使用除了使用jsonp跨域，还有一种CORS，可以参考[HTTP访问控制（CORS）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)
