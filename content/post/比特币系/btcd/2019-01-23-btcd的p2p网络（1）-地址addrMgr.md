---
title: btcd的p2p网络（1）-地址addrMrg
date: 2019-01-23
tags: ["btcd","p2p"]
categories: ["btcd"]
---

p2p网络从底层到上层可以分为3层，**地址** **连接** **节点**，每一层都有自己的功能。
这一节主要介绍地址
*声明：文章代码和源码有不一致地方*
# btcd的p2p网络之地址
主要有四个结构体,两两对应
>AddrManager
serializedAddrManager

>KnownAddress
serializedKnownAddress

我们先看`peer.json`中的内容,这个就是和`serializedAddrManager`的内容对应的。
```
//peers.json

{
    "Version": 1,
    "Key": [233,19,87,131,183,155,......,231,78,82,150,10,102],
    "Addresses": [
        {
            "Addr": "109.157.120.169:8333",
            "Src": "104.172.5.90:8333",
            "Attempts": 0,
            "TimeStamp": 1514967959,
            "LastAttempt": -62135596800,
            "LastSuccess": -62135596800
        },
        ......
    ],
    "NewBuckets": [
        [
            "[2001:0:9d38:78cf:3cb1:bb2:ab6f:e8b4]:8333",
            "196.209.239.229:8333",
            ......
            "65.130.177.198:8333"
        ],
        ......
        [
            "125.227.159.115:8333",
            ......
            "alhlegtjkdmbqsvt.onion:8333",
            ......
            "79.250.188.226:8333"
        ]
    ],
    "TriedBuckets": [
        [
            "5.9.165.181:8333",
            ......
            "5.9.17.24:8333"
        ],
        [
            "95.79.50.90:8333",
            ......
            "[2a02:c207:2008:9136::1]:8333"
        ]
    ]
}
```
### start()
start()先从peer.json中加载，然后开启一个goroutine定期进行跟踪和添加地址。
```
func (a *AddrManager) Start(){
	//alter stared ?
	if atomic.AddInt32(&a.started,1) != 1{
		log.Trace("AddrManager has stared")
		return
	}
	log.Trace("Starting address manager")
	// load peers we already know about from file
	// 从文件中加载peers
	a.loadPeers()

	// start the address ticker to save addresses periodically（定期）.
	a.wg.Add(1) // 在下面的goroutine中，a.wg.Done()之后，start才会结束
	go a.addressHandler()
}
```
我们再看`a.loadPeers()`,主要是调用`deserializePeers`,就是把peer.json中的文件反序列化。主要是把`Addresses``NewBuckets``TriedBuckets`,解析到`AddrManager`的 `addrIndex` `addrNew` `addrTried`
```
func (a *AddrManager) deserializePeers(filePath string) error {
	_, err := os.Stat(filePath)
	if os.IsNotExist(err) {
		return nil
	}
	r, err := os.Open(filePath)
	if err != nil {
		return fmt.Errorf("opening file:%v error:%v", filePath, err)
	}
	defer r.Close()

	var sam serializedAddrManager
	dec := json.NewDecoder(r)
	err = dec.Decode(&sam)
	if err != nil {
		return fmt.Errorf("error reading %s: %v", filePath, err)
	}
	// 检查版本
	if sam.Version != serialisationVersion {
		return fmt.Errorf("unknown version %v in serialized "+
			"addrmanager", sam.Version)
	}
	copy(a.key[:], sam.Key[:])
	// 先解析Addresses
	for _, v := range sam.Addresses {
		ka := new(KnownAddress)
		ka.na, err = a.DeserializeNetAddress(v.Addr)
		if err != nil {
			return fmt.Errorf("failed to deserialize netaddress "+
				"%s: %v", v.Addr, err)
		}
		ka.srcAddr, err = a.DeserializeNetAddress(v.Src)
		if err != nil {
			return fmt.Errorf("failed to deserialize netaddress "+
				"%s: %v", v.Src, err)
		}
		ka.attempts = v.Attempts
		ka.lastattempt = time.Unix(v.LastAttempt, 0)
		ka.lastsuccess = time.Unix(v.LastAttempt, 0)
		// 遍历sam.Addresses中的每一项，然后辨析成一个KnownAddress后，
		// 添加到addrManager.addrIndex,是一个map
		a.addrIndex[NetAddressKey(ka.na)] = ka
	}

	// 遍历解析Newbuckets
	for i := range sam.NewBuckets {
		for _, val := range sam.NewBuckets[i] {
			// 先看看addrIndex中有没有这个ip:port
			ka, ok := a.addrIndex[val]
			if !ok {
				return fmt.Errorf("newbucket contains %s but "+
					"none in address list", val)
			}
			// addrIndex中已经有了
			if ka.refs == 0 {
				a.nNew ++
			}
			ka.refs ++
			a.addrNew[i][val] = ka
		}
	}
	// 遍历解析TriedBuckets
	for i := range sam.TriedBuckets {
		for _, val := range sam.TriedBuckets[i] {
			ka, ok := a.addrIndex[val]
			if !ok {
				return fmt.Errorf("Newbucket contains %s but "+
					"none in address list", val)
			}

			ka.tried = true
			a.nTried++
			// 一个bucket到这里就成了一个数组
			a.addrTried[i].PushBack(ka)
		}
	}
	// 检查,保证一个地址要么在 NewBuckets中，要么在TriedBuckets
	for k, v := range a.addrIndex {
		if v.refs == 0 && !v.tried {
			return fmt.Errorf("address %s after serialisation "+
				"with no references", k)
		}
		if v.refs > 0 && v.tried {
			return fmt.Errorf("address %s after serialisation "+
				"which is both new and tried!", k)
		}
	}
	return nil
}
```
**然后调用a.addressHandler()**，主要逻辑就是定期保存地址
```
// addressHandler is the main handler for the address manager.  It must be run
// as a goroutine.
func (a *AddrManager) addressHandler() {
	dumpAddressTicker := time.NewTicker(dumpAddressInterval)
	defer dumpAddressTicker.Stop()
out:
	for {
		select {
		case <-dumpAddressTicker.C:
			a.savePeers()
		case <-a.quit:
			break out
		}
	}
	a.savePeers()
	a.wg.Done()  // start方法中已经有wg.Add(1)在等着了
	log.Trace("Address handler done")
}
```
a.savePeers(),大家可以查看源码

## updateAddress
节点直接交换getaddr和addr消息时，就会收到addr信息，调用`AddAddress()`是实际上都是调用`updateAddress`
如果我们已经有了，就进行一些修改
如果我们没有，就添加，如果满了的话，就把老的给移除掉

## NewBucket到TriedBucket
在节点获取地址，并建立peer连接成功后，会调用Good方法。就是说这个地址是好的，可以从NewBucket移到TriedBucket了。
## GetAddress
这个是供外面获取地址的方法

按照50%的记录随机从NewBucket或者TriedBucket中选择。随机选择bucket后，从中随机选择地址。选择出来的地址要判断一下
```
randval := a.rand.Intn(large)
if float64(randval) < (factor * ka.chance() * float64(large)) {
	log.Tracef("Selected %v from new bucket",NetAddressKey(ka.na))
	return ka
}
```
决定是不是用这个选中地址，主要是由factor和ka.chance()决定。
facket依次递增
而ka.chance()则和这个地址的其他属性密切相关。比如上次尝试到现在间隔，失败的次数。

我们完全可以去除这个包，设置几个固定的地址，但是这几个地址不能用后，我们的节点也就没法同步了。就不是p2p了。搞这么复杂，主要还是为了防止攻击。
---
AddLocalAddress
```
// AddLocalAddress adds na to the list of known local addresses to advertise with the given priority.
// 地址会先加到AddrManager.localAddresses
func (a *AddrManager) AddLocalAddress(na *wire.NetAddress,priority AddressPriority)error{
	// TODO 判断不是是公网路由
	a.lamtx.Lock()
	defer a.lamtx.Unlock()
	// 得到特定格式的ip：端口
	key := NetAddressKey(na)
	// 判断localAddresses有没有这个ip
	la,ok := a.localAddresses[key]
	if !ok || la.score < priority{
		if ok {
			la.score = priority + 1
		}else{
			a.localAddresses[key] = &localAddress{
				na:na,
				score:priority,
			}
		}
	}
	return nil
}
```




