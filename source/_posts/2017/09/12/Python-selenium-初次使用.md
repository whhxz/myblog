---
title: Python selenium 初次使用
date: 2017-09-12 09:18:48
categories: ['python', 'selenium']
tags: ['python', 'selenium', '']
---

### 使用前准备
```shell
pip install selenium
```
下载PhantomJS，ChromeDriver

### 使用selenium
```python
from selenium import webdriver


# 通过phantomjs使用，phantomjs是无界面浏览器
def use_phantom():
    # 设置代理
    service_args = ['--proxy=proxy3.fn.com:8080', '--proxy-type=http']
    # 设置路径
    driver = webdriver.PhantomJS(executable_path="/Users/xuzhuo/SOFT/phantomjs/phantomjs-2.1.1-macosx/bin/phantomjs",
                                 service_args=service_args)
    driver.get("https://www.google.com")
    # 截图
    driver.get_screenshot_as_file("phantomjsGoogle.png")
    driver.close()


# 使用chrome浏览器
def use_chrome():
    # 获取chrome参数配置
    chrome_options = webdriver.ChromeOptions()
    # 使用代理
    chrome_options.add_argument("--proxy-server=proxy3.fn.com:8080")
    chrome_options.add_argument("--proxy-type=http")
    # 设置路径
    driver = webdriver.Chrome(executable_path="/Users/xuzhuo/SOFT/chromedriver/chromedriver",
                              chrome_options=chrome_options)
    driver.get("https://www.google.com")
    # 截图
    driver.get_screenshot_as_file("chromeGoogle.png")
    driver.close()


use_phantom()
use_chrome()
```
