---
title: nginx配置证书后https访问无法建立安全连接
date: 2019-05-27T08:00:00
tags: ["nginx","https"]
categories: ["nginx"]
---
# nginx配置https

`listen 443; >>> listen 443 ssl;`

> 修改后上传覆盖配置文件并重启Nginx后发现可以正常访问了，因此错误的原因在于配置文件的代码写错了。
>  listen 443这种写法是支持新版本而不支持老版本的。
> 所以如果再遇到这类问题首先要检查Nginx版本，不然不同的写法有的支持有的不支持，再折腾也没戏了