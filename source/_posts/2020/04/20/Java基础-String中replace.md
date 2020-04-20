---
title: Java基础-String中replace
date: 2020-04-20 10:25:22
categories: ['Java基础']
tags: ['String']
---

String中replace是我们常用的一个方法，用于替换字符中的字母。
### 方法`java.lang.String#replace(char, char)`
入参为char，源码如下：
```java
public String replace(char oldChar, char newChar) {
    //判断传入的oldChar和newChar是否一致，如果不一样开始做替换
    if (oldChar != newChar) {
        int len = value.length;
        int i = -1;
        char[] val = value; /* 避免使用 getfield 操作码 */
        //获取第一个匹配需要替换的字符
        while (++i < len) {
            if (val[i] == oldChar) {
                break;
            }
        }
        //判断是否匹配上，如果匹配上开始做替换，如果未匹配上，直接返回this
        if (i < len) {
            //构造一个新字符数组，长度数据和原有一致
            char buf[] = new char[len];
            //把第一个匹配上的前面数据赋值到新数组中
            for (int j = 0; j < i; j++) {
                buf[j] = val[j];
            }
            //遍历数组后续，如果匹配上替换为新字符
            while (i < len) {
                char c = val[i];
                buf[i] = (c == oldChar) ? newChar : c;
                i++;
            }
            //通过字符数组构建一个新字符串
            return new String(buf, true);
        }
    }
    //如果传入参数一样，直接返回当前字符串对象
    return this;
}
```
<!-- more -->
总结：
1、如果传入的参数被替换字符、替换一致，直接返回当前字符串对象，如果被替换字符不在当前字符串内，也直接返回当前字符串对象。
2、遍历旧字符数组，通过一个新字符数组存储替换后的字符，返回一个新构建的字符串对象。

### 方法`java.lang.String#replace(java.lang.CharSequence, java.lang.CharSequence)`
入参为两个`java.lang.CharSequence`。源码如下：
```java
public String replace(CharSequence target, CharSequence replacement) {
    //通过正则表达式替换，被替换字符串使用Pattern.LITERAL模式，表示输入的字符串都视为字面值，对于元字符或转义序列无任何意义
    //替换的字符串，使用Matcher.quoteReplacement处理，也就是对\和$处理，如果存在这两个字符，那么对其进行转译
    return Pattern.compile(target.toString(), Pattern.LITERAL).matcher(
            this).replaceAll(Matcher.quoteReplacement(replacement.toString()));
}
```
总结：
1、替换字符串是从前往后开始替换，如果先匹配先替换
2、通过Pattern纯粹的字符串替换，对于**targe**，使用了`Pattern.LITERAL`（纯粹视为字符），对于**replacement**使用了`Matcher.quoteReplacement`（对与\和$做转译处理）。


### 方法`java.lang.String#replaceAll`
入参为两个`String`。源码如下：
```java
public String replaceAll(String regex, String replacement) {
    return Pattern.compile(regex).matcher(this).replaceAll(replacement);
}
```
直接通过传入的正则表达式进行全部的替换。

### 方法`java.lang.String#replaceFirst`
入参为两个`String`。源码如下：
```java
public String replaceFirst(String regex, String replacement) {
    return Pattern.compile(regex).matcher(this).replaceFirst(replacement);
}
```
一样通过正则表达式进行替换，只不过只替换第一个匹配上的。

