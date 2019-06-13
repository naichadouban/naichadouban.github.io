---
title: btcd里的优秀代码总结
date: 2019-06-06
tags: ["btcd","golang"]
categories: ["btcd","golang"]
---

## service的定义，判断，添加
定义的时候会进行移位操作，这样每一种服务就代表了一位。
```
const (
	// 节点可以被请求全节点。实现了当前版本的所有功能
	SFNodeNetwork ServiceFlag = 1 << iota
)
```

然后判断节点有没有一个服务，直接进行与操作。
```
func (msg *MsgVersion) HasService(service ServiceFlag) bool {
	if msg.Services&service == service {
		return true
	}
	return false
}
```
添加服务
```
func (msg *MsgVersion) AddService(service ServiceFlag) {
	msg.Services |= service
}
```
直接进行或运算就可以
> note:
1.扩展下可以用日志等级
2.可以用在权限系统

## golang随机数
利用rand包
```
// randomUint64 returns a cryptographically random uint64 value.  This
// unexported version takes a reader primarily to ensure the error paths
// can be properly tested by passing a fake reader in the tests.
func randomUint64(r io.Reader) (uint64, error) {
	var b [8]byte
	_, err := io.ReadFull(r, b[:])
	if err != nil {
		return 0, err
	}
	return binary.BigEndian.Uint64(b[:]), nil
}

// RandomUint64 returns a cryptographically random uint64 value.
func RandomUint64() (uint64, error) {
	return randomUint64(rand.Reader)
}
```
关键点就是`rand`包给我们提供了`io.Reader`,我们从这个`reader`里读到的都是随机的。

## 判断两个结构体对象是否相同

```
if !reflect.DeepEqual(na,test.out){
	t.Errorf("readNetAddress #%d\n got: %s want: %s", i,
		spew.Sdump(na), spew.Sdump(test.out))
	continue
}
```
就是利用`reflect.DeepEqual()`