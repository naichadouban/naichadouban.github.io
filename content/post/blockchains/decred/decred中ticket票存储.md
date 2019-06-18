---
title: decred中票存储
date: 2019-06-17T00:00:00+08:00
tags: ["blockchain"]
categories: ["blockchain"]
draft: false
---

# Tickets集合数据结构
DCR要保存当前live的tickets, 怎么组织这些tickets?  最简单的方式就是用一个数组来保存. 但因为live ticekts这个数组要频繁的插入, 删除, 所以性能很低.
DCR使用Treap来保存tickets( liveTickets, missedTickets, revokedTickets)
Tree+Heap = 树+堆 = 树堆.

 每个结点两个键值（key、priority）。
 性质1. Treap是关于key的二叉排序树。
 性质2. Treap是关于priority的堆。（非二叉堆，因为不是完全二叉树）
 结论1. key和priority确定时，treap唯一。
 作用1. 随机分配的优先级，使数据插入后更不容易退化为链。就像是将其打乱再插入。所以用于平衡二叉树。

![](https://teakki.com/file/image/5c8f802b3f3cf314330fc618)
它的插入/删除操作时间都是 O(log n)
treap的代码在: blockchain/stake/tickettreap 中
ticket节点有key, priority, 还有value:
```
// treapNode represents a node in the treap.
type treapNode struct {
   key      Key
   value    Value
   priority uint32
   size     uint32 // Count of items within this treap - the node itself counts as 1.
   left     treapNode
   right    treapNode
}

```
Key就是ticket hash.
```
// Key defines the key used to add an associated value to the treap.
type Key chainhash.Hash

```
Value是:
```
// Value defines the information stored for a given key in the treap.
type Value struct {
   // Height is the block height of the associated ticket.
   Height uint32

   // Flags defining the ticket state.
   Missed  bool
   Revoked bool
   Spent   bool
   Expired bool
}

```
其中 priority 不是随机生成的, 而是 ticket 的高度.
因为key是hash, priority是height, 所以插入的时候先通过hash插入变成二叉排序树, 再通过height来旋转变成小堆.


