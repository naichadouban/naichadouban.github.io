---
title: SSL/TLS 双向认证 -- SSLTLS工作原理
date: 2019-04-08
tags: ["ssl&tls"]
categories: ["计算机网路"]
---
参考：https://blog.csdn.net/ustccw/article/details/76691248
https://www.barretlee.com/blog/2016/04/24/detail-about-ca-and-certs/?utm_source=tuicool&utm_medium=referral

#CA & SSL Server & SSL Client 介绍
## SSL/TLS 工作流
![image.png](https://upload-images.jianshu.io/upload_images/422094-c88c548754577bcf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

CA: 证书授权中心( certificate authority)
似于国家出入境管理处一样，给别人颁发护照；也类似于国家工商管理局一样，给公司企业颁发营业执照。 
CA的两大特性：
1) CA本身是受信任的 // 国际认可的 
2) 给他受信任的申请对象颁发证书 
> 也可以撤销证书？怎么撤销？

> 电脑中的证书
360浏览器: 选项/设置-> 高级设置 -> 隐私于安全 -> 管理 HTTPS/SSL 证书 -> 证书颁发机构 
火狐浏览器: 首选项 -> 高级 -> 证书 -> 查看证书 -> 证书机构 
chrome浏览器: 设置 -> 高级 -> 管理证书 -> 授权中心 
ubuntu: /etc/ssl/certs 

### ca.crt和server.crt的关系
1) SSL Server 自己生成一个 私钥/公钥对。server.key/server.pub // 私钥加密，公钥解密！ 
2) server.pub 生成一个请求文件 server.req. 请求文件中包含有 server 的一些信息，如域名/申请者/公钥等。 
3) server 将请求文件 server.req 递交给 CA，CA验明正身后，将用 ca.key和请求文件加密生成 server.crt 
4) 由于 ca.key 和 ca.crt 是一对, 于是 ca.crt 可以解密 server.crt. 

上面说的私钥加密，其实就是私钥签名

> 在实际应用中：如果 SSL Client 想要校验 SSL server.那么 SSL server 必须要将他的证书 server.crt 传给 client.然后 client 用 ca.crt 去校验 server.crt 的合法性。如果是一个钓鱼网站，那么CA是不会给他颁发合法server.crt证书的，这样client 用ca.crt去校验，就会失败。

## 单向认证、双向认证
单向认证指的是只有一个对象校验对端的证书合法性。 
通常都是client来校验服务器的合法性。那么client需要一个ca.crt,服务器需要server.crt,server.key 
>双向认证指的是相互校验，服务器需要校验每个client,client也需要校验服务器。 
server 需要 server.key 、server.crt 、ca.crt 
client 需要 client.key 、client.crt 、ca.crt

## 证书详细工作流
![](https://upload-images.jianshu.io/upload_images/422094-867a61bb8c15defc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## SSL/TLS单向认证流程
![image.png](https://upload-images.jianshu.io/upload_images/422094-2b299544afca9ffb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## SSL/TLS双向认证流程
和单向认证几乎一样，只是在client认证完服务器证书后，client会将自己的证书client.crt传给服务器。服务器验证通过后，开始秘钥协商。 
## 证书等格式说明
### crt/key/req/csr/pem/der等拓展名都是什么东东？
crt 表示证书, .key表示私钥, .req 表示请求文件,.csr也表示请求文件, .pem表示pem格式，.der表示der格式。 
文件拓展名你可以随便命名。只是为了理解需要而命名不同的拓展名。但文件中的信息是有格式的，和exe，PE格式一样,证书有两种格式。 
pem格式和der格式。所有证书，私钥等都可以是pem,也可以是der格式，取决于应用需要。 
pem和der格式可以互转

### 证书和公钥关系
证书中含有 申请者公钥、申请者的组织信息和个人信息、签发机构 CA的信息、有效时间、证书序列号等信息的明文，同时包含一个签名。如查看百度证书详细信息。

## SSL/TLS和 Openssl,mbedtls是什么关系？
SSL/TLS是一种工作原理，openssl和mbedtls是SSL/TLS的具体实现，很类似于 TCP/IP协议和socket之间的关系。
# 本地生成SSL相关文件
建立一个CA(ca.crt + ca.key)，用这个CA给server和client分别颁发证书。
首先建四个文件
> ca目录：保存ca的私钥ca.key和证书ca.crt 
certDER目录:将证书保存为二进制文件 ca.der client.der server.der 
client目录: client.crt client.key 
server目录:server.crt server.key
-----
实际操作时注意目录 
```
//  private key generation
openssl genrsa -out ca.key 1024
openssl genrsa -out server.key 1024
openssl genrsa -out client.key 1024
//  cert requests
openssl req -out ca.req -key ca.key -new
openssl req -out server.req -key server.key -new
openssl req -out client.req -key client.key -new 
//  generate the actual certs.
openssl x509 -req -in ca.req -out ca.crt -sha1 -days 5000 -signkey ca.key
openssl x509 -req -in server.req -out server.crt -sha1 -CAcreateserial -days 5000 -CA ca.crt -CAkey ca.key
openssl x509 -req -in client.req -out client.crt -sha1 -CAcreateserial -days 5000  -CA ca.crt -CAkey ca.key
```
验证
```
openssl verify -CAfile ca/ca.crt server/server.crt
openssl verify -CAfile ca/ca.crt client/client.crt
```
注意：
如果`Common Name (e.g. server FQDN or YOUR name) []:`,填的和ca一样，会不通过。报错如下
```
openssl verify -CAfile ca/ca.crt server/server.crt 
server/server.crt: C = AU, ST = Some-State, O = Internet Widgits Pty Ltd, CN = req_distinguished_name
error 18 at 0 depth lookup:self signed certificate
OK

```