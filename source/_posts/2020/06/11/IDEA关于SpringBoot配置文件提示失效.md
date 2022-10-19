---
title: IDEA关于SpringBoot配置文件提示失效
date: 2020-06-11 09:55:59
categories: ['IDEA']
tags: ['配置文件提示']
---

使用IDEA开发SpringBoot项目，有时候在导入项目后，写配置文件无提示。
可以正常提示的**application.properties**为一个绿叶图标，不能正常提示的图标为普通**properties**图标。
<!-- more -->
如下：
> 正常
![正常图标](/images/old/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202020-06-11%20%E4%B8%8A%E5%8D%8810.01.51.png)

> 普通
![普通图标](/images/old/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202020-06-11%20%E4%B8%8A%E5%8D%8810.01.40.png)

正常修复应该是对项目按**F4**打开moudle设置，如果没有Spring，添加Spring支持
如下：
>
![Spring支持](/images/old/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202020-06-11%20%E4%B8%8A%E5%8D%8810.14.26.png)

之后点击Spring，截图如下：
>
![](/images/old/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202020-06-11%20%E4%B8%8A%E5%8D%8810.18.42.png)

是没有spring的配置的，表示当前项目SpringBoot配置文件无法正确识别，可以正常的识别应该为如下：
>
![](/images/old/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202020-06-11%20%E4%B8%8A%E5%8D%8810.19.48.png)

查询资料后，正常应该点击下方绿叶按钮然后添加配置文件
![](/images/old/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202020-06-11%20%E4%B8%8A%E5%8D%8810.21.10.png)
* windows系统好像在上面

不知道为什么一直提示有误，无法点击确认如下：
![](/images/old/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202020-06-11%20%E4%B8%8A%E5%8D%8810.22.10.png)

在网上查询一圈资料未解决后，通过找正常项目下的iml文件和无法正常提示的iml文件得到，如果直接在iml里面添加配置即可解决，
在iml文件里面填写如下：
```xml
<facet type="Spring" name="Spring">
    <configuration spring_boot_spring_config_custom_files="file://$MODULE_DIR$/src/main/resources/application.properties;file://$MODULE_DIR$/src/main/resources/config/local/application.properties"/>
</facet>
```
重点是**spring_boot_spring_config_custom_files**，后面添加配置文件路径，如果有多个文件需要提示可以通过分号分隔
