---
title: dcrdata浏览器研究
date: 2019-06-21
tags: ["explorer"]
categories: ["blockchain","decred"]
---

# 需要掌握的知识
- database查询时如何设置超时
https://blog.csdn.net/itfootball/article/details/82385002

- 如果获取postgresql的各种配置项

- 如何优化postgresql的写入性能

- 如何判断某个表是否存在


# sqlite和stakedb是在一起的，pg单独

stakedb就是存储在本地的文件夹`ffldb_stake`,用的是官方实现的数据库驱动ffldb

# stakedb数据存储
Metadata()桶 
	k： stakechainstate 
	v： [0:32] 是区块hash [32:36]是区块高度

# 协程逻辑
p chainMonitor

p.ReorgHandler()
	从p.reorgChan获取到重组数据，然后把数据赋值给p.reorgData
	p.reorgData = reorgData

p.BlockConnectedHandler
	从p.blockChan 获取到区块hash


websocket通知

面向接口编程。收到新的区块后，把所有需要同步区块的对象都放到一个数组中，然后for range这个数据，调用每个对象的save方法。

webui的save先是将数据保存到自身对象的某个字段上，然后给个通知，收到通知后，webui就把自己某个字段的数据通过websocket推送到前端的js