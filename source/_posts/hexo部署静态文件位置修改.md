title: hexo部署静态文件位置修改
date: 2015-02-04 16:18:32
---

## 使用hexo-htemo-kael后出现样式不对

一般在部署到whhxz.github.io上不会出现问题，如果部署在whhxz.github.io/myblog上，会出现本地静态文件如js、css等出错

解决办法：修改hexo-htemo-kael中head.ejs，添加

``` js
  var reg = /myblog\/$/;
  if(reg.exec(config.root) == null){
      config.root = config.root + "myblog/";
  }
```

设置config.root后面添加博客地址，在生成html后，就会自动添加上去，而且为了避免多次添加，所以使用正则表达式判断是否已经存在
<!-- more -->
