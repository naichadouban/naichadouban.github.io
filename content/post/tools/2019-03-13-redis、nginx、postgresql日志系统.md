---
title: redis、nginx、postgresql日志系统
date: 2019-03-13T08:00:00
tags: ["redis","nginx","postgresql"]
categories: ["日志"]
---
下面介绍的都是ubuntu16.04
# redis
## 日志
版本
```
root@iZt4ndq9lezw03v80dtwykZ:/var/lib/redis# redis-server --version
Redis server v=3.0.6 sha=00000000:0 malloc=jemalloc-3.6.0 bits=64 build=28b6715d3583bf8e

```
在配置文件找到日志的相关配置
`vim /etc/redis/redis.conf`
```
# warning (only very important / critical messages are logged)
loglevel notice

# Specify the log file name. Also the empty string can be used to force
# Redis to log on the standard output. Note that if you use standard
# output for logging but daemonize, logs will be sent to /dev/null
logfile /var/log/redis/redis-server.log

```
查看日志
`tail -f /var/log/redis/redis-server.log`
## 持久化路径

在配置文件找到日志的相关配置
`vim /etc/redis/redis.conf`
```
# The working directory.
#
# The DB will be written inside this directory, with the filename specified
# above using the 'dbfilename' configuration directive.
#
# The Append Only File will also be created inside this directory.
#
# Note that you must specify a directory here, not a file name.
dir /var/lib/redis
```
查看下
```
root@iZt4ndq9lezw03v80dtwykZ:~# ll /var/lib/redis/
total 12
drwxr-x---  2 redis redis 4096 Mar 13 11:28 ./
drwxr-xr-x 44 root  root  4096 Mar 10 02:39 ../
-rw-rw----  1 redis redis  513 Mar 12 16:49 dump.rdb
```
修改配置文件中`dir` 到自己指定的路径，然后移动`dump.rdb`到指定的路径内就可以了。