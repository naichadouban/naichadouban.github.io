---
title: btcd的p2p网络（2）-连接connMgr
date: 2019-01-28
tags: ["btcd","p2p"]
categories: ["btcd"]
---

p2p网络从底层到上层可以分为3层，**地址** **连接** **节点**，每一层都有自己的功能
*声明：文章代码和源码有不一致地方*
这篇文章简单记录下连接conn

# 三个主要的结构体
1、连接管理
```
// ConnManager providers a manager to handle network connections.
type ConnManager struct {
	// the following variables must only be used atomically
	// 记录自己主动连接其他节点的连接数量
	connReqCount uint64
	// 标识connmgr已经启动
	start int32
	// 标识connmgr已经结束
	stop int32

	// 设定相关的配置
	cfg Config
	// 用于同步connmgr的退出状态，调用方可以阻塞等待connmgr的工作协程退出
	wg sync.WaitGroup
	// 某个连接失败后，connmgr尝试选择新的peer地址连接的总次数
	failedAttempts uint64
	// 用于与connmgr工作协程通信的管道
	requests chan interface{}
	// 用于通知工作协程退出
	quit chan struct{}
}
```
2、Config，配置参数
其实就是`connmgr`配置，本身就是`ConnManager`结构体的一个字段:`cfg`。
```
// Config holds the configuration options related to the connection manager.
type Config struct {
	// Listeners define a slice of listeners for which the connection manager
	// will take ownership of(取得所有权) and accept connections. when a connection
	// is accepted,the OnAccept handler will be invoked with the connection. since
	// the connection manager takes ownership of these listeners,they will be closed
	// when the connection manager is stoped.

	// this field will not have any effect if the onAccept field is not also specified.
	// It may be nil if the caller does not wish to listen for
	// incoming connection

	Listeners []net.Listener //节点上所有等待外部连接的监听点;
	// OnAccept is a callback that is fired when an inbound connection is
	// accepted.  It is the caller's responsibility(责任、义务) to close the connection.
	// Failure to close the connection will result in the connection manager
	// believing the connection is still active and thus have undesirable
	// side effects such as still counting toward maximum connection limits.
	//
	// This field will not have any effect if the Listeners field is not
	// also specified since there couldn't possibly be any accepted
	// connections in that case.
	OnAccept func(net.Conn) // 节点应答并接受外部连接后的回调函数
	// TargetOutbound is the number of outbound network connections to
	// maintain. Defaults to 8.
	TargetOutbound uint32 // 节点主动向外连接peer的最大个数
	// RetryDuration is the duration to wait before retrying connection
	// requests. Defaults to 5s.
	RetryDuration time.Duration // 连接失败后发起重连的等待时间
	// OnConnection is a callback that is fired when a new outbound
	// connection is established.
	OnConnection func(*ConnReq, net.Conn) // 连接建立成功后的回调函数
	// OnDisconnection is a callback that is fired when an outbound
	// connection is disconnected.
	OnDisconnection func(*ConnReq) // 连接关闭后的回调函数
	// GetNewAddress is a way to get an address to make a network connection
	// to.  If nil, no new connections will be made automatically.
	// 连接失败后，ConnMgr可能会选择新的peer地址进行连接
	// GetNewAddress函数提供了获取新的peer地址的方法，它最终会调用addrManager中
	// 的GetAddress()来分配新地址。
	GetNewAddress func() (net.Addr, error)
	// Dial connects to the address on the named network.It cannot be nil.
	// 定义建立TCP连接的方式，是直接连接还是通过代理连接。
	Dial func(net.Addr) (net.Conn, error)
}
```
3、ConnReq 描述了一个连接
```
// ConnReq is the connection request to a network address. If permanent, the
// connection will be retried on disconnection.
// ConnReq 描述了一个连接
type ConnReq struct {
	// The following variables must only be used atomically.
	// 连接的序号，用于索引
	id uint64
	// 连接的目的地址
	Addr      net.Addr
	// 标识是否与Peer保持永久连接，如果为true，
	// 则连接失败后，继续尝试与该Peer连接，而不是选择新的Peer地址重新连接
	Permanent bool
	// 连接成功后，真实的net.Conn对象;
	conn       net.Conn
	// 连接的状态，有ConnPending、ConnEstablished、ConnDisconnected及ConnFailed等;
	state      ConnState
	// stateMtx: 保护state状态的读写锁;
	stateMtx   sync.RWMutex
	//retryCount: 如果Permanent为true，retryCount记录该连接重复重连的次数;
	retryCount uint32
}
```
结合起来说，就是连接管理器`connmgr`按照自身的配置`config`，管理着一些连接`connReq`

# 启动ConnMgr
我们先看`start()`函数
```
// Start: launches(发起、发动)the connection manager and begins conecting to the network.
func (cm *ConnManager) Start() {
	// already started ?
	if atomic.AddInt32(&cm.start, 1) != 1 {
		return
	}
	log.Trace("Connection manager started")
	cm.wg.Add(1)
	// 启动工作协程
	go cm.connHandler()
	// Start all the listeners so long as the caller requested
	// them and provided a callback to be invoked when connections are accepted.
	if cm.cfg.OnAccept != nil {
		for _, listenr := range cm.cfg.Listeners {
			cm.wg.Add(1)
			// 启动监听协程listenHandler，等待其他节点连接;
			go cm.listenHandler(listenr)
		}
	}
	// 启动建立连接的协程，选择Peer地址并主动连接;
	for i := atomic.LoadUint64(&cm.connReqCount); i < uint64(cm.cfg.TargetOutbound); i++ {
		go cm.NewConnReq()
	}
}
```
主要是启动工作协程`cm.connHandler()`,
然后一方面监听其他节点的连接，`go cm.listenHandler(listenr)`这里面做的事情就是我们普通的`tcp`地址监听。
一方面主动去连接其他的节点: `cm.NewConnReq()`
动态选择Peer并发起连接的过程就是在NewConnReq()中实现:
```

// NewConnReq creates a new connection request and connects to the
// corresponding(对应的) address.
// 创建一个连接请求，然后连接对应的地址
func (cm *ConnManager) NewConnReq() {
	if atomic.LoadInt32(&cm.stop) != 0 {
		return
	}
	if cm.cfg.GetNewAddress == nil {
		return
	}
	c := &ConnReq{}
	atomic.StoreUint64(&c.id, atomic.AddUint64(&cm.connReqCount, 1))
	// Submit a request of a pending connection attempt to the connection
	// manager. By registering the id before the connection is even established,
	// we'll be able to later cancel the connection via the Remove method.
	done := make(chan struct{})
	select {
	case cm.requests <- registerPending{c, done}:
	case <-cm.quit:
		return
	}

	// wait for the registration to successfully add the pending conn req
	// to the conn manager's internal state.
	select {
	case <-done:
	case <-cm.quit:
		return
	}
	addr,err := cm.cfg.GetNewAddress()
	if err != nil {
		select {
		case cm.requests <- handleFailed{c, err}:
		case <-cm.quit:
		}
		return
	}
	c.Addr = addr
	cm.Connect(c)
}
```
首先构造一个`ConnReq`:`c := &ConnReq{}`,然后生成`registerPending{c, done}`,
把`registerPending`写入到connmgr的通道`case cm.requests <- registerPending{c, done}`

这里的`registerPending`结构体中还有一个通道`done`,`cm.requests`这个通道的另一端肯定有人会从里面读数据，处理完后会通过通道`done`返回信息。下面的` case <-done:`就是在等待返回的信息。
谁在通道的另外一头读呢？`go cm.connHandler()`,下面这个图就是他们工作概况
![](https://upload-images.jianshu.io/upload_images/422094-ed63ff88b2e3bfe6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


cm.connHandler()从通道`requests`中接受信息，处理完后通过通道`done`返回。然后就可以`cm.cfg.GetNewAddress()`得到一个连接的地址（这里用到了`addrMgr`）,然后连接` cm.Connect(c)`
```
// Connection assigns an id and dials a connection to the address of the connection request
func (cm *ConnManager) Connect(c *ConnReq){
	if atomic.LoadInt32(&cm.stop) != 0{
		return
	}
	// TODO 再次检查一遍，相当于重复了NewConnReq（）的工作
	log.Debugf("Attempting to connect to %v", c)
	conn,err := cm.cfg.Dial(c.Addr)
	if err != nil {
		select {
		case cm.requests <- handleFailed{c, err}:
		case <-cm.quit:
		}
		return
	}

	select {
	case cm.requests <- handleConnected{c, conn}:
	case <-cm.quit:
	}
}
```
连接主要就是这句代码`conn,err := cm.cfg.Dial(c.Addr)`,这个`Dial`就是在普通的`tcp`连接外包了一层，让我们有个选择，比如可以通过*代理*进行连接。

如果连接失败，`cm.requests <- handleFailed{c, err}`；如果连接成功，`cm.requests <- handleConnected{c, conn}`.`handleConnected{c, conn}:`和`handleFailed{c, err}:`这两个结构体都被构建，并且发送到`cm.requests`

有连接就有断开
```
func (cm *ConnManager) Disconnect(id uint64) {
	if atomic.LoadInt32(&cm.stop) != 0 {
		return
	}

	select {
	case cm.requests <- handleDisconnected{id, true}:
	case <-cm.quit:
	}
}
```
可`connect`也差不多，都是向`cm.requests`发了一个请求。
> 看来，连接或者断开连接的主要处理逻辑在connHandler中，我们来看看它的实现:
```
// connHandler handles all connection related requests.  It must be run as a
// goroutine.
//
// The connection handler makes sure that we maintain a pool of active outbound
// connections so that we remain connected to the network.  Connection requests
// are processed and mapped by their assigned ids.
func (cm *ConnManager) connHandler() {
	// pending holds all registered conn requests that hava yet to succeed.
	var pending = make(map[uint64]*ConnReq)
	// conns represents the set of all actively connected peers.
	var conns = make(map[uint64]*ConnReq, cm.cfg.TargetOutbound) // make map时，size可以省略，当你知道大小时，最好加上

out:
	for {
		select {
		case req := <- cm.requests:
			switch msg:=req.(type) {
			case registerPending:
				// TODO
			case handleConnected:
				connReq := msg.c

				if _, ok := pending[connReq.id]; !ok {
					if msg.conn != nil {
						msg.conn.Close()
					}
					log.Debugf("Ignoring connection for "+
						"canceled connreq=%v", connReq)
					continue
				}

				connReq.updateState(ConnEstablished)
				connReq.conn = msg.conn
				conns[connReq.id] = connReq
				log.Debugf("Connected to %v", connReq)
				connReq.retryCount = 0
				cm.failedAttempts = 0

				delete(pending, connReq.id)

				if cm.cfg.OnConnection != nil {
					go cm.cfg.OnConnection(connReq, msg.conn)
				}
			case handleDisconnected:
				// TODO
			case handleFailed:
				// TODO
			}

		case <-cm.quit:
			break out
		}
	}
	cm.wg.Done()
	log.Trace("Connection handler done")
}
```
在这里不停的处理`cm.requests`通道中的信息。我们看下连接成功的处理：
首先创建了两个变量
```
// pending holds all registered conn requests that hava yet to succeed.
	var pending = make(map[uint64]*ConnReq)
	// conns represents the set of all actively connected peers.
	var conns = make(map[uint64]*ConnReq, cm.cfg.TargetOutbound) // make map时，size可以省略，当你知道大小时，最好加上
```
连接成功后
1. 在`map变量 pending`中找有没有这个连接请求，如果没有则表明这不是我们要的连接。断开
2. 更新`connReq`的状态，然后添加到`map conns`中
3. 调用`go cm.cfg.OnConnection(connReq, msg.conn)`

两个peer之间的连接conn，还需要考虑其他的很多方面。但是还好，到现在我们至少可以简单的创建一个连接了。
至于连接成功后调用`cm.cfg.OnConnection()`要干什么，我们后面再分析了。
------
参考
https://www.jianshu.com/p/d6484e5710ad