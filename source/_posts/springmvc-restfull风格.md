title: springmvc RESTful风格
date: 2016-12-28 09:57:02
tags: ['RESTfull','springMVC']
categories: ['RESTfull','基本概念']
---

## RESTfull简介
REST全称是(Representational State Transfer)，简单来说指的是一种设计风格，一种规范。在设计时，可以把服务器看做一种资源文件服务器，每条URL代表对应的静态资源，而请求的method，表示最资源的操作,如：GET（获取资源）、PUT（添加资源）、DELETE（删除资源）等操作。
简单来说URL--->资源，method--->操作。

## java例子（基于springMVC）
``` java
@Controller
@RequestMapping("/rest")
public class RestFullController {
    @RequestMapping(value = "/test/{id}", method = RequestMethod.POST)
    @ResponseBody
    public String post(@PathVariable String id){
        return "post" + id;
    }

    @RequestMapping(value = "/test/{id}", method = RequestMethod.GET)
    @ResponseBody
    public String get(@PathVariable String id){
        return "get" + id;
    }

    @RequestMapping(value = "/test/{id}", method = RequestMethod.DELETE)
    @ResponseBody
    public String delete(@PathVariable String id){
        return "delete" + id;
    }
}
```
同一个url只是对应的method不一样，这样对于数据的增删改查，方法是基于请求时method。
