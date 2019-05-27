---
title: python爬虫001：python urllib读取网页
date: 2018-12-14T23:00:00
tags: ["python","爬虫","urllib"]
categories: ["python分布式爬虫教程"]
---
# 正则表达式re
加`()`,代表我们需要括号里面的东西
不加`()`,表示全部内容我们都需要
```python
import re

mystr = """<span class \"search_yx_t j\">
  共<em>5830</em>个职位满足条件
  <span>"""

restr = "<em>(\\d+)</em>"#d+表示和数字有关；():只要里面的对象
regex = re.compile(restr, re, IGNORECASE)
mylist = regex.findall(mystr)

```

# 读取网页的三种方式
```python
#py2
#enconding:utf-8
import urllib2

url = "http://www.baidu.com"#urlopen只能处理http，不可以处理https

def download1(url):
  return urllib2.urlopen(url).read()#读取全部网页

def download2(url):
  return urllib2.urlopen(url).readlines()#读取每一行的网页数据，然后压入列表

def download3(url):
  response = urllib2.orlopen(url)#网页抽象为文件
  while True:
    line = response.readline()#读取一行
    if not line:
      break

```

# python2 和python3的区别
## 编码
一般抓取英文数据用python2，抓取带中文的数据，就需要用python3

用python2打印中文是，需要在第一行`encoding:utf-8`

## urllib2

获取一个网页数据，urllib在python2和python3中有不同的表示
- python2
```python
urllib2.urlopen(url).read()
# urllib2只可以处理http，不可以处理https
```
- python3
```python
urllib2.request(url).read()
```

# python被网站屏蔽：referer
有时候我们请求服务器的时候，服务器可以知道通过请求头中的`referer`参数，知道是谁在请求它
服务器如果发现不是浏览器在请求他，二是python在请求他。直接502.可以直接拒绝我们的请求。

# selenium
我们需要这个东西来模拟浏览器。selenium可以操作我们的浏览器

# 抓取智联招聘
基于selenium库和selenium.webdriver
```python
import selenium #测试框架
import selennium.webdriver #模拟浏览器
import re

mystr = """<span class \"search_yx_t j\">
  共<em>5830</em>个职位满足条件
  <span>"""

restr = "<em>(\\d+)</em>"#d+表示和数字有关；():只要里面的对象
regex = re.compile(restr, re. IGNORECASE)
mylist = regex.findall(pagesource)
def getnumberbyname(searchname):
  url = "https://sou.zhaopin.com/?jl=613&kw=" + searchname + "&kt=3"
  driver = selenium.webdriver.Firefox() #调用火狐浏览器
  driver.get(url) #访问链接
  pagesource = driver.page_source #抓取网页源代码
  driver.close()#关闭
  return mylist[0]

# print getnumberbyname("python") 这是测试函数

pythonlist = ["python", "python 运维", "python 测试", "python 数据", "python web"]
for oystr in pythonlist:
  print pystr, getnumberbyname(pystr)

```
# 抓取51job
```python
import selenium #测试框架
import selennium.webdriver #模拟浏览器
import re

mystr = """<div class = "rt">
  共67条职位
  <\div>"""


def getnumberbyname(searchname): #可能这里有一些混乱，手头没有python环境就没测试，大致就先这样吧
 url="https://search.51job.com/list/240200,000000,0000,00,9,99,"+searchname +",2,1.htmllang=c&stype=&postchannel=0000&workyear=99&cotype=99&degefrom=99&jobterm=99&companysize=99&providesalary=99&lonlat=0%2C0&adius=-1&ord_field=0&confirmdate=9&fromType=&dibiaoid=0&address=&lin=&specialarea=00&from=&welfare=
 driver = selenium.webdriver.Firefox() #调用火狐浏览器
 driver.get(url) #访问链接
 pagesource = driver.page_source #抓取网页源代码
 restr = """(\\d+)""" #先抓大，再抓小；尤其是空白字 符出现的时候
 regex = re.compile(restr, re.IGNORECASE)
 mylist = regex.findall(pagesource)
 newstr = mylist[0].strip()
 driver.close()#关闭
 return mylist[0]

pythonlist = ["python", "python 运维", "python 测试", "python 数据", "python web"]
for oystr in pythonlist:
  print pystr, getnumberbyname(pystr)
```
