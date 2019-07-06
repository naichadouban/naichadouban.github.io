---
title: dcrdata浏览器研究
date: 2019-06-21
tags: ["explorer"]
categories: ["blockchain","decred"]
---

## 查看一段程序运行了多长时间

```golang
	// Time this function
	defer func(start time.Time, perr *error) {
		if *perr != nil {
			log.Infof("resyncDBWithPoolValue() completed in %v", time.Since(start))
		}
	}(time.Now(), &err)
```
在一个方法的起始位置，用golang的defer属性

