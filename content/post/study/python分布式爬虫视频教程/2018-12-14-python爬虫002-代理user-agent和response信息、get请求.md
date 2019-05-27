---
title: python爬虫002：代理user-agent和response信息、get请求
date: 2018-12-14T23:30:00
tags: ["python","爬虫"]
categories: ["python分布式爬虫教程"]
---
# response网络详细信息
**python2中urllib和urllib2的区别**
参考地址：https://blog.csdn.net/qq_34327480/article/details/79161794
> 在python2中，urllib和urllib2都是接受URL请求的相关模块，但是提供了不同的功能。两个最显著的不同如下：
> 1、urllib2可以接受一个Request类的实例来设置URL请求的headers
> 2、urllib仅可以接受URL。这意味着，你不可以伪装你的User Agent字符串等。
## python3使用urllib
```python
#py3
import urllib.request
#pycharm go declaration to search source code
def download(url):
  response = urllib.request.urlopen(url, timeout = 5) 

print(type(response))# class http.client.httpresponse
print(response.info()) #class  http.client.HTTPMessage
print(download("http://ww.baidu.com"))

```
## python2使用urllib2
python2里面没有urllib.reqeust,我们直接用urllib2替换即可
还有开头的`coding:utf-8`
```python
#py2
#coding:utf-8
import urllib2


def download(url):
  response = urllib2.urlopen(url, timeout = 5) 
  print(type(response))# class http.client.httpresponse
  print(response.info()) #包含了网站的详细信息
  print(response.read()) #read source coad

#括号内是控制多少字符的问题
#写爬虫记得try catch
try:
  print(download("http://ww.google.com"))
except urllib2.URLError as e:
  print("网络异常", e) #抓住错误对象类型当作变量

```
或者
```
import urllib2
req = urllib2.Request('http://www.example.com/')
req.add_header('Referer', 'http://www.python.org/')
# Customize the default User-Agent header value:
req.add_header('User-Agent', 'urllib-example/0.1 (Contact: . . .)')
r = urllib2.urlopen(req)
```

## response信息
再贴一下打印response.info()的信息
```
Bdpagetype: 1
Bdqid: 0xe33c14ce00005740
Cache-Control: private
Content-Type: text/html
Cxy_all: baidu+7b2f0340f919578bfe3264aa8c0016f8
date: Sun,T02 Dec 2018 03:03:24 GMT
Expires: Sun, 02 Dec 2018 03:03:01 GMT
P3p: CP=" OTI DSP COR IVA OUR IND COM "
Server: BWS/1.1
Set-Cookie: BAIDUID=00813CE4F82EFFA6488DB896F424E587:FG=1; expires=Thu, 31-Dec-37 23:55:55 GMT; max-age=2147483647; path=/; domain=.baidu.com
Set-Cookie: BIDUPSID=00813CE4F82EFFA6488DB896F424E587; expires=Thu, 31-Dec-37 23:55:55 GMT; max-age=2147483647; path=/; domain=.baidu.com
Set-Cookie: PSTM=1543719804; expires=Thu, 31-Dec-37 23:55:55 GMT; max-age=2147483647; path=/; domain=.baidu.com
Set-Cookie: delPer=0; path=/; domain=.baidu.com
Set-Cookie: BDSVRTM=0; path=/
Set-Cookie: BD_HOME=0; path=/
Set-Cookie: H_PS_PSSID=1441_25810_21122_22159; path=/; domain=.baidu.com
Vary: Accept-Encoding
X-Ua-Compatible: IE=Edge,chrome=1
Connection: close
Transfer-Encoding: chunked
```
注意参数里面有一个`Bdqid`,这是百度给每个用户的唯一标识

### response.read() 
查看全部网页源代码
### response.read(100)
查看网页源代码的前100个字节

写爬虫的时候多家try，catch

# agent
就像大会狼冒充大白兔
```python
#encoding: utf-8
import urllib2

def download(url):
  headers = {"User Agent : "}
  request = urllib2.Request(url, headers = headers) #发起请求
  data = urllib2.urlopen(request).read() #打开请求，抓取数据
  return data

url = "https://sou.zhaopin.com/?jl=613&kw=" + searchname + "&kt=3"
print download(url)

```
> 上面这段代码构造了一个request，在python2的情况下

## 常见的代理
手机代理
```python
safariiOS4.33–iPhone
User-Agent:Mozilla/5.0(iPhone;U;CPUiPhoneOS4_3_3likeMacOSX;en-us)AppleWebKit/533.17.9(KHTML,likeGecko)Version/5.0.2Mobile/8J2Safari/6533.18.5
safariiOS4.33–iPodTouch
User-Agent:Mozilla/5.0(iPod;U;CPUiPhoneOS4_3_3likeMacOSX;en-us)AppleWebKit/533.17.9(KHTML,likeGecko)Version/5.0.2Mobile/8J2Safari/6533.18.5
safariiOS4.33–iPad
User-Agent:Mozilla/5.0(iPad;U;CPUOS4_3_3likeMacOSX;en-us)AppleWebKit/533.17.9(KHTML,likeGecko)Version/5.0.2Mobile/8J2Safari/6533.18.5
AndroidN1
User-Agent:Mozilla/5.0(Linux;U;Android2.3.7;en-us;NexusOneBuild/FRF91)AppleWebKit/533.1(KHTML,likeGecko)Version/4.0MobileSafari/533.1
AndroidQQ浏览器Forandroid
User-Agent:MQQBrowser/26Mozilla/5.0(Linux;U;Android2.3.7;zh-cn;MB200Build/GRJ22;CyanogenMod-7)AppleWebKit/533.1(KHTML,likeGecko)Version/4.0MobileSafari/533.1
AndroidOperaMobile
User-Agent:Opera/9.80(Android2.3.4;Linux;OperaMobi/build-1107180945;U;en-GB)Presto/2.8.149Version/11.10
AndroidPadMotoXoom
User-Agent:Mozilla/5.0(Linux;U;Android3.0;en-us;XoomBuild/HRI39)AppleWebKit/534.13(KHTML,likeGecko)Version/4.0Safari/534.13
BlackBerry
User-Agent:Mozilla/5.0(BlackBerry;U;BlackBerry9800;en)AppleWebKit/534.1+(KHTML,likeGecko)Version/6.0.0.337MobileSafari/534.1+
WebOSHPTouchpad
User-Agent:Mozilla/5.0(hp-tablet;Linux;hpwOS/3.0.0;U;en-US)AppleWebKit/534.6(KHTML,likeGecko)wOSBrowser/233.70Safari/534.6TouchPad/1.0
NokiaN97
User-Agent:Mozilla/5.0(SymbianOS/9.4;Series60/5.0NokiaN97-1/20.0.019;Profile/MIDP-2.1Configuration/CLDC-1.1)AppleWebKit/525(KHTML,likeGecko)BrowserNG/7.1.18124
WindowsPhoneMango
User-Agent:Mozilla/5.0(compatible;MSIE9.0;WindowsPhoneOS7.5;Trident/5.0;IEMobile/9.0;HTC;Titan)
UC无
User-Agent:UCWEB7.0.2.37/28/999
UC标准
User-Agent:NOKIA5700/UCWEB7.0.2.37/28/999
UCOpenwave
User-Agent:Openwave/UCWEB7.0.2.37/28/999
UCOpera
User-Agent:Mozilla/4.0(compatible;MSIE6.0;)Opera/UCWEB7.0.2.37/28/999

```
电脑代理
```
safari5.1–MAC
User-Agent:Mozilla/5.0(Macintosh;U;IntelMacOSX10_6_8;en-us)AppleWebKit/534.50(KHTML,likeGecko)Version/5.1Safari/534.50
safari5.1–Windows
User-Agent:Mozilla/5.0(Windows;U;WindowsNT6.1;en-us)AppleWebKit/534.50(KHTML,likeGecko)Version/5.1Safari/534.50


IE9.0
User-Agent:Mozilla/5.0(compatible;MSIE9.0;WindowsNT6.1;Trident/5.0
IE8.0
User-Agent:Mozilla/4.0(compatible;MSIE8.0;WindowsNT6.0;Trident/4.0)
IE7.0
User-Agent:Mozilla/4.0(compatible;MSIE7.0;WindowsNT6.0)
IE6.0
User-Agent:Mozilla/4.0(compatible;MSIE6.0;WindowsNT5.1)
Firefox4.0.1–MAC
User-Agent:Mozilla/5.0(Macintosh;IntelMacOSX10.6;rv:2.0.1)Gecko/20100101Firefox/4.0.1
Firefox4.0.1–Windows
User-Agent:Mozilla/5.0(WindowsNT6.1;rv:2.0.1)Gecko/20100101Firefox/4.0.1
Opera11.11–MAC
User-Agent:Opera/9.80(Macintosh;IntelMacOSX10.6.8;U;en)Presto/2.8.131Version/11.11
Opera11.11–Windows
User-Agent:Opera/9.80(WindowsNT6.1;U;en)Presto/2.8.131Version/11.11
Chrome17.0–MAC
User-Agent:Mozilla/5.0(Macintosh;IntelMacOSX10_7_0)AppleWebKit/535.11(KHTML,likeGecko)Chrome/17.0.963.56Safari/535.11
傲游（Maxthon）
User-Agent:Mozilla/4.0(compatible;MSIE7.0;WindowsNT5.1;Maxthon2.0)
腾讯TT
User-Agent:Mozilla/4.0(compatible;MSIE7.0;WindowsNT5.1;TencentTraveler4.0)
世界之窗（TheWorld）2.x
User-Agent:Mozilla/4.0(compatible;MSIE7.0;WindowsNT5.1)
世界之窗（TheWorld）3.x
User-Agent:Mozilla/4.0(compatible;MSIE7.0;WindowsNT5.1;TheWorld)
搜狗浏览器1.x
User-Agent:Mozilla/4.0(compatible;MSIE7.0;WindowsNT5.1;Trident/4.0;SE2.XMetaSr1.0;SE2.XMetaSr1.0;.NETCLR2.0.50727;SE2.XMetaSr1.0)
360浏览器
User-Agent:Mozilla/4.0(compatible;MSIE7.0;WindowsNT5.1;360SE)
Avant
User-Agent:Mozilla/4.0(compatible;MSIE7.0;WindowsNT5.1;AvantBrowser)
GreenBrowser
User-Agent:Mozilla/4.0(compatible;MSIE7.0;WindowsNT5.1)
```

## 模拟手机浏览器

# get模拟百度请求

### urllib的编码和解码
浏览器的地址栏经常看到乱码的情况，这就是编码的问题
```python
words={'wd':'徐晓峰'}
urllib.urlencode(words) #'wd=%E5%BE%90%E6%99%93%E5%B3%B0',注意这里是urllib不是urllib2

```

```python
#coding:urf-8
import urllib
import urllib2
url = 'http://www.baidu.com/s'
word = {'wd':'徐晓峰'}
newurl = url+'?'+urllib.urlencoding(word)
reqeust = urllib2.Request(newurl)
request.add_header("Connection":"keep-alive") #可以自由的添加头信息
print urllib2.urlopen(request).read()
``` 


