---
title: decred数据库存储
date: 2019-06-14T00:00:00+08:00
tags: ["decred"]
categories: ["decred"]
draft: false

---

# 2. Decred 数据库存储
程序=数据结构+算法. 我们在写代码或看代码时, 第一应该想到的是程序应该要有什么样的数据结构, 先设计数据结构, 再写代码, 一切代码都围绕这些数据来操作.
DCR有POW+POS, 所以相比BTCD多了Ticket系列数据库.
和BTC类似, DCR的存储也是用了key-value数据库, 和BTC用的leveldb不一样的是, DCR默认用了使用golang写的ffldb, 当然它可以用其它数据库, 只要实现了它的接口就行.
为什么要使用leveldb或ffldb呢? 其实不是因为他们比mysql, mongodb强大, 相反它们的功能会少很多, 比如不支持索引, 只有简单的Get, Set的操作. 当然使用它们恰恰是因为它们简单, 功能够用, 又不像mysql, mongodb要单独安装.

## 块数据
不是 key=&gt;value 的形式, 具体的存储形式看 database/ffldb/db.go
## 区块链数据库
blockchain/internal/dbnamespace/dbnamespace.go
```
var (
   // ByteOrder is the preferred byte order used for serializing numeric
   // fields for storage in the database.
   ByteOrder = binary.LittleEndian

   // BCDBInfoBucketName is the name of the database bucket used to house
   // global versioning and date information for the blockchain database.
   BCDBInfoBucketName = []byte("dbinfo")

   // BCDBInfoVersionKeyName is the name of the database key used to house
   // the database version.  It is itself under the BCDBInfoBucketName
   // bucket.
   BCDBInfoVersionKeyName = []byte("version")

   // BCDBInfoCompressionVersionKeyName is the name of the database key
   // used to house the database compression version.  It is itself under
   // the BCDBInfoBucketName bucket.
   BCDBInfoCompressionVersionKeyName = []byte("compver")

   // BCDBInfoBlockIndexVersionKeyName is the name of the database key
   // used to house the database block index version.  It is itself under
   // the BCDBInfoBucketName bucket.
   BCDBInfoBlockIndexVersionKeyName = []byte("bidxver")

   // BCDBInfoCreatedKeyName is the name of the database key used to house
   // date the database was created.  It is itself under the
   // BCDBInfoBucketName bucket.
   BCDBInfoCreatedKeyName = []byte("created")

   // ChainStateKeyName is the name of the db key used to store the best
   // chain state.
   ChainStateKeyName = []byte("chainstate")

   // SpendJournalBucketName is the name of the db bucket used to house
   // transactions outputs that are spent in each block.
   SpendJournalBucketName = []byte("spendjournal") // blockHash => []spentTxOut

   // UtxoSetBucketName is the name of the db bucket used to house the
   // unspent transaction output set.
   UtxoSetBucketName = []byte("utxoset") // txHash => [output1, output2...]

   // BlockIndexBucketName is the name of the db bucket used to house the
   // block index which consists of metadata for all known blocks both in
   // the main chain and on side chains.
   BlockIndexBucketName = []byte("blockidx")
)


```

### utxoset 未花费记录 (最重要)
bucket: utxoset
所有的utxoentry是存在名为“utxoset”(utxoSetBucketName)的Bucket中，其中的Key为交易的Hash，Value为utxoentry的序列化结果
{txHash =&gt; UtxoEntry}
```
type UtxoEntry struct {
   modified      bool                   // Entry changed since load.
   version       int32                  // The version of this tx.
   isCoinBase    bool                   // Whether entry is a coinbase tx.
   blockHeight   int32                  // Height of block containing tx.
   sparseOutputs map[uint32]utxoOutput // Sparse map of unspent outputs.
}
type utxoOutput struct {
   spent      bool   // Output is spent.
   compressed bool   // The amount and public key script are compressed.
   amount     int64  // The amount of the output.
   pkScript   []byte // The public key script for the output.
}


```

### pendJournal 花费日志
bucket: spendjournal
这个库在bitcoind代码中是没有的, 记录了某个块花费的utxo, 主要为了redo时用.
{blockHash =&gt; []spentTxOut}
```
type spentTxOut struct {
   compressed bool   // The amount and public key script are compressed.
   version    int32  // The version of creating tx.
   amount     int64  // The amount of the output.
   pkScript   []byte // The public key script for the output.

   // These fields are only set when this is spending the final output of
   // the creating tx.
   height     int32 // Height of the the block containing the creating tx.
   isCoinBase bool  // Whether creating tx is a coinbase.
}

```
stxos按序记录了交易中所有花费的utxo(s)，并进而按区块中交易的顺序记录了所有交易花费的utxo(s)，也就是说，stxos将会按交易及交易输入的顺序记录区块中交易花费的所有utxo(s)。
如果区块因分叉而被从主链上移除，stxos中的记录将被加回到utxoset中，后面我们将会看到，stoxs也被存储到数据库中。uxtoentry若被完全花费，即它的所有输出均被花费，它将被从utxoset中移除，如果需要将其恢复成utxoentry，则需要额外记录区块高度height和isCoinBase字段;
### blockidx 块元数据
bucket: blockidx
所有block header + status + ticket信息
{blockHash+blockHeight =&gt; blockIndexEntry}
```
// blockIndexEntry represents a block index database entry.
type blockIndexEntry struct {
   header         wire.BlockHeader
   status         blockStatus
   voteInfo       []stake.VoteVersionTuple // 投票信息集合, 在后面章节会提到
   ticketsVoted   []chainhash.Hash // 中票集合
   ticketsRevoked []chainhash.Hash // 退票votes
}


```

### chainstate 链状态
记录主链信息, 仅一条记录, 存在数据据的元数据中, 没有bucket
{"chainstate" =&gt; bestChainState}
```
// bestChainState represents the data to be stored the database for the current
// best chain state.
type bestChainState struct {
   hash         chainhash.Hash
   height       uint32
   totalTxns    uint64
   totalSubsidy int64
   workSum      big.Int
}


```

### databaseInfo 数据库
bucket: dbinfo
仅一条数据, 主要保存数据库的一些基本信息, 有以下字段
```
type databaseInfo struct {
   version uint32 // The version of the database
   compVer uint32 // The script compression version of the database
   bidxVer uint32 // The block index version of the database
   created time.Time // The date of the creation of the database
}


```

## Ticket 数据库
```
var (
   // ByteOrder is the preferred byte order used for serializing numeric
   // fields for storage in the database.
   ByteOrder = binary.LittleEndian

   // StakeDbInfoBucketName is the name of the database bucket used to
   // house a single k->v that stores global versioning and date information for
   // the stake database.
   StakeDbInfoBucketName = []byte("stakedbinfo")

   // StakeChainStateKeyName is the name of the db key used to store the best
   // chain state from the perspective of the stake database.
   StakeChainStateKeyName = []byte("stakechainstate")

   // LiveTicketsBucketName is the name of the db bucket used to house the
   // list of live tickets keyed to their entry height.
   LiveTicketsBucketName = []byte("livetickets")

   // MissedTicketsBucketName is the name of the db bucket used to house the
   // list of missed tickets keyed to their entry height.
   MissedTicketsBucketName = []byte("missedtickets")

   // RevokedTicketsBucketName is the name of the db bucket used to house the
   // list of revoked tickets keyed to their entry height.
   RevokedTicketsBucketName = []byte("revokedtickets")

   // StakeBlockUndoDataBucketName is the name of the db bucket used to house the
   // information used to roll back the three main databases when regressing
   // backwards through the blockchain and restoring the stake information
   // to that of an earlier height. It is keyed to a mainchain height.
   StakeBlockUndoDataBucketName = []byte("stakeblockundo")

   // TicketsInBlockBucketName is the name of the db bucket used to house the
   // list of tickets in a block added to the mainchain, so that it can be
   // looked up later to insert new tickets into the live ticket database.
   TicketsInBlockBucketName = []byte("ticketsinblock")
)


```

### 3种类型ticket
主要存储各个状态的票, 如:

 livetickets 活跃票集(已成熟的)
 missedtickets 错过票集
 revokedtickets 已撤销票集

这些buckets都存储了选票的高度+状态, key是ticket tx hash
{txHash =&gt; {height(4 byte), 状态: missed/revoked/spent/expired } }
livetickets 数量会保持在40960左右, missedtickets和revokedtickets会随时间逐渐增加.
这3种tickets在blockchain启动时会全部加载到内存中.
### ticketsinblock 新tickets
主链下某高度下的新ticket
{height =&gt; []ticket tx Hash}
### stakeblockundo 
主链下记录某高度下所有选票的状态, 类似于 pendJournal, 这个库主要是为了做undo (disconnectNode 时用)
{height =&gt; []UndoTicketData}
```
// UndoTicketData is the data for any ticket that has been spent, missed, or
// revoked at some new height.  It is used to roll back the database in the
// event of reorganizations or determining if a side chain block is valid.
// The last 3 are encoded as a single byte of flags.
// The flags describe a particular state for the ticket:
//  1. Missed is set, but revoked and spent are not (0000 0001). The ticket
//      was selected in the lottery at this block height but missed, or the
//      ticket became too old and was missed. The ticket is being moved to the
//      missed ticket bucket from the live ticket bucket.
//  2. Missed and revoked are set (0000 0011). The ticket was missed
//      previously at a block before this one and was revoked, and
//      as such is being moved to the revoked ticket bucket from the
//      missed ticket bucket.
//  3. Spent is set (0000 0100). The ticket has been spent and is removed
//      from the live ticket bucket.
//  4. No flags are set. The ticket was newly added to the live ticket
//      bucket this block as a maturing ticket.
type UndoTicketData struct {
   TicketHash   chainhash.Hash
   TicketHeight uint32
   Missed       bool
   Revoked      bool
   Spent        bool
   Expired      bool
}


```

### stakechainstate stake chain state
和chainstate类似, 这里记录了stake的状态, 同样它保存在metadata中, key 是 stakechainstate value是:
```
type BestChainState struct {
   Hash        chainhash.Hash
   Height      uint32
   Live        uint32
   Missed      uint64
   Revoked     uint64
   PerBlock    uint16
   NextWinners []chainhash.Hash
}


```

## 索引数据库
除了这几个库外, 还有一些为了查询方便的索引数据库, 比如为了快速查询tx, 就可以为每个tx建一个索引, key是tx hash.
查看源码, 现在有以下几种索引数据库:
![image.png](https://upload-images.jianshu.io/upload_images/422094-1f1b353af5fad23e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### addrindex
保存某地址涉及所有的交易, tx按块id从小到大存储
key: &lt;addr type&gt;&lt;addr hash&gt;&lt;level&gt;
vlaue: [&lt;block id&gt;&lt;start offset&gt;&lt;tx length&gt;,...]

```
// The serialized key format is:
//
//   <addr type><addr hash><level>
//
//   Field           Type      Size
//   addr type       uint8     1 byte
//   addr hash       hash160   20 bytes
//   level           uint8     1 byte
//   -----
//   Total: 22 bytes
//
// The serialized value format is:
//
//   [<block id><start offset><tx length>,...] // 这里用了blockId
//
//   Field           Type      Size
//   block id        uint32    4 bytes
//   start offset    uint32    4 bytes
//   tx length       uint32    4 bytes
//   -----
//   Total: 12 bytes per indexed tx
// -----------------------------------------------------------------------------


```

### txindex (Transaction-by-hash)
为了存储txHash =&gt; block信息, 因为一个block会有很多tx, 如果每个txHash的value都有block hash, 因为block hash 32字节, 太大, 会造成冗余, 所以特意新建两个库给block hash和block id作对应用.
blockHash与blockId对应的两个表: 
blockId是自增的唯一id, 与blockHeight是不同的概念, 多个块的blockHeight可能相同(可能有侧链), 但blockId肯定不一样! 这个blockId(4字节)的作用主要是为了减少存储, 因为blockHash(32字节)太大了.
idbyhashidx
```
block id -> block hash index

```
hashbyididx
```
block hash -> block id

```
ConnectBlock()时blockId++, DisconnectBlock()时, blockId--
tx一个表:
txbyhashidx (仅主链), ConnectBlock(), DisconnectBlock()
```
txHash -> tx
<txhash> => <block id><start offset><tx length>


```

### cfindex
committed filter 用于存储block的过滤器, 用于SPV调用.
过滤器分为普通和扩展
```
// 块过滤器
// cfIndexKeys is an array of db bucket names used to house indexes of
// block hashes to cfilters.
cfIndexKeys = [][]byte{
   []byte("cf0byhashidx"), // 普通
   []byte("cf1byhashidx"), // 扩展
}

// 块滤器的头部, 不是过滤器
// cfHeaderKeys is an array of db bucket names used to house indexes of
// block hashes to cf headers.
cfHeaderKeys = [][]byte{
   []byte("cf0headerbyhashidx"), // 普通
   []byte("cf1headerbyhashidx"), // 扩展
}

```
普通过滤器存储的是: 所有交易(包括stake交易)的输入, 输出.
扩展过滤器存储的是: 所有交易(包括stake交易)的输入解锁脚本(即验证数据)
块过滤器存储的是 `块hash =&gt; 块过滤器数据`
过滤器头部存储的是 `块hash =&gt; blake256(blake256(块过滤器数据) + 前一过滤器头部)`, 这就类似块头部一样, 包含本块的hash和前一块的hash
过滤器头部的作用只是为了校验块过滤器数据是否正确.
## 总结
DCR相比BTC多了POS, 所以了多了Ticket数据库, 但和区块链数据库的表类似, 都有状态数据库. 这些最重要的库是:

 utxoset
 3种类型的ticket

其实Decred所有的区块链操作都是在维护这4个库. 因为块有两种操作, 一是主链入块, 二是主链删块(因为可以会发生侧链转主链, 之前的块要先删除掉), 所以这4个库都有可能会undo.

 入块: connectBlock()
 删块: disconnectNode()

索引数据库会也这两个操作.
其实DCR的很多库的value都有特定的serialize和deserialize的方法, 为什么不直接存JSON数据, 这样marshal和unmarshal很方便啊, 其实主要是为了存储空间的考虑!! 所以如果不考虑存储空间, 其实我们完全可以用关系型数据库, 如Mysql来存储数据, 这样会大大减少代码量和开发难度.

