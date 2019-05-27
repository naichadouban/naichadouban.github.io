---
title: btcd的p2p网络（3）-连接ConnMgr-连接成功之后
date: 2019-03-18
tags: ["btcd","p2p"]
categories: ["btcd"]
---
我们继续上一节，先简要回顾一下
我们主要通过一下几个步骤建立了一个连接
```
func (cm *ConnManager) Start() {
	for i := atomic.LoadUint64(&cm.connReqCount); i < uint64(cm.cfg.TargetOutbound); i++ {
		go cm.NewConnReq()
	}
}
```
```
func (cm *ConnManager) NewConnReq() {
    ......
	c := &ConnReq{}
	addr, err := cm.cfg.GetNewAddress()
	c.Addr = addr
	cm.Connect(c)
}
```
```
// Connect assigns an id and dials a connection to the address of the
// connection request.
func (cm *ConnManager) Connect(c *ConnReq) {
    ......
	conn, err := cm.cfg.Dial(c.Addr)
	select {
	case cm.requests <- handleConnected{c, conn}:
	case <-cm.quit:
	}
}
```
然后在工作协程中
```
func (cm *ConnManager) connHandler() {

	var (
		// pending holds all registered conn requests that have yet to
		// succeed.
		pending = make(map[uint64]*ConnReq)

		// conns represents the set of all actively connected peers.
		conns = make(map[uint64]*ConnReq, cm.cfg.TargetOutbound)
	)

out:
	for {
		select {
		case req := <-cm.requests:
			switch msg := req.(type) {

			case registerPending:
				connReq := msg.c
				connReq.updateState(ConnPending)
				pending[msg.c.id] = connReq
				close(msg.done)
            // 连城成功后的处理
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
}
```
>我们接下来主要看的是，连接成功后干什么？

我们就来找cm.cfg.OnConnection(),在编辑器中全局搜索之后发现，OnConnection()只有在`server.go`中配置了,`server.go`中`newServer()`方法中
```
	cmgr, err := connmgr.New(&connmgr.Config{
		Listeners:      listeners,
		OnAccept:       s.inboundPeerConnected,
		RetryDuration:  connectionRetryInterval,
		TargetOutbound: uint32(targetOutbound),
		Dial:           btcdDial,
		OnConnection:   s.outboundPeerConnected,
		GetNewAddress:  newAddressFunc,
	})
```
就是这句`OnConnection:   s.outboundPeerConnected`,然后我们就去找`outboundPeerConnected`,发现`outboundPeerConnected`是一个函数，这和我们连接成功调用时是符合的：`go cm.cfg.OnConnection(connReq, msg.conn)`
```
// outboundPeerConnected is invoked by the connection manager when a new
// outbound connection is established.  It initializes a new outbound server
// peer instance, associates it with the relevant state such as the connection
// request instance and the connection itself, and finally notifies the address
// manager of the attempt.
func (s *server) outboundPeerConnected(c *connmgr.ConnReq, conn net.Conn) {
	sp := newServerPeer(s, c.Permanent)
	p, err := peer.NewOutboundPeer(newPeerConfig(sp), c.Addr.String())
	if err != nil {
		srvrLog.Debugf("Cannot create outbound peer %s: %v", c.Addr, err)
		s.connManager.Disconnect(c.ID())
	}
	sp.Peer = p
	sp.connReq = c
	sp.isWhitelisted = isWhitelisted(conn.RemoteAddr())
	sp.AssociateConnection(conn)
	go s.peerDoneHandler(sp)
	s.addrManager.Attempt(sp.NA())
}
```
我们看到连接成功后，需要用到`peer`包来处理。请看下节的peer