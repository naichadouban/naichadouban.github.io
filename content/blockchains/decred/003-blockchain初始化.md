---
title: blockchain初始化
date: 2019-06-14T00:00:00+08:00
tags: ["decred"]
categories: ["decred"]
draft: false

---
# 3. Blockchain 初始化
## 文件结构
DCR的核心代码是 blockchain 区块链文件夹下的代码, 该文件夹下主要有以下文件:
```
blockchain/
    chaingen/ 区块/交易生成方法, 主要为了测试区块链, fullblocktests/使用它
    fullblocktests/ 区块链全量测试, 具体的测试执行 fullblocks_test.go 中
        generate.go 生成区块链测试用例
    indexers/ 索引
        addrindex.go 地址索引
        txindex.go 交易索引
        cfindex.go 过滤器, 用于spv
    internal/dbnamespace/dbnamespace.go 数据库定义, 如utxo, spendjournal
    stake/ 权益代码
        internal/dbnamespace/dbnamespace.go stake相关数据库定义
        lottery.go 生成PRN伪随机数从而得到winning的ticket (和中彩票很像)
        staketx.go stake相关交易(3种)检查判断
        tickets.go stake数据入库/导出 

    chain.go 区块链操作的主要文件, 定义 BlockChain{}, 对BlockChain{}内数据操作, connect/disconnect Block/reorganizeChain
    chainio.go 数据库查询/更新/入库操作
    
    // 索引
    blockindex.go 区块索引 hash => blockNode
    chainview.go 区块索引 height => blockNode (主链)
    
    // 区块链操作
    process.go 区块入块入口文件 processBlock()
    accept.go 核心方法 maybeAcceptBlock() 区块入块
    utxoviewpoint.go utxo数据库操作 (核心)
    validate.go 各种共识检查
    scriptval.go 脚本检查, validate.go调用
    checkpoints.go 找检查点/验证, 主要由 validate.go 调用
    difficulty.go 难度计算(PoW &amp; PoS)
    subsidy.go 奖励计算(总奖励/PoW/PoS/团队)
    compress.go 压缩/解压数据(脚本)
    stakenode.go stake节点数据生成,核心方法是 fetchStakeNode(), accept/chain/valiate.go 都有调用
    prune.go 为了删除stakeNode, 防止内存过大
    
    // 共识更改相关
    thresholdstate.go 议案(共识更改)
    stakeversion.go 议案是否满足PoS升级条件, 由thresholdstate.go/validate.go调用
    votebits.go 议案开始/结束时间, 投票周期, 投票bit判断
        
    // 获取链上信息相关
    chainquery.go rpcserver.go调用, 获取区块链信息  
    stakeext.go 获取链上的live/winning/missed/expired数据, 主要由rpcserver.go调用
    
    // 其它
    sequencelock.go 闪电网络需要
    timesorter.go 为了计算区块中位时间, 要有一个时间排序方法, blockindex.go调用
    merkle.go merkle树计算
    notifications.go 通知
    upgrade.go 升级
    mediantime.go 同步/调整时间, 确保时间其它节点时间一致. validate.go 调用

```
下面我们看看这些代码的入口文件.
在blockchain/example_test.go中, 有blockchain启动的代码, 如下:
```
 func ExampleBlockChain_ProcessBlock() {
   // Create a new database to store the accepted blocks into.  Typically
   // this would be opening an existing database and would not be deleting
   // and creating a new database like this, but it is done here so this is
   // a complete working example and does not leave temporary files laying
   // around.
   dbPath := filepath.Join(os.TempDir(), "exampleprocessblock")
   _ = os.RemoveAll(dbPath)
   db, err := database.Create("ffldb", dbPath, chaincfg.MainNetParams.Net)
   if err != nil {
      fmt.Printf("Failed to create database: %v&#92;n", err)
      return
   }
   defer os.RemoveAll(dbPath)
   defer db.Close()

   // Create a new BlockChain instance using the underlying database for
   // the main bitcoin network.  This example does not demonstrate some
   // of the other available configuration options such as specifying a
   // notification callback and signature cache.  Also, the caller would
   // ordinarily keep a reference to the median time source and add time
   // values obtained from other peers on the network so the local time is
   // adjusted to be in agreement with other peers.
   chain, err := blockchain.New(&amp;blockchain.Config{
      DB:          db,
      ChainParams: &amp;chaincfg.MainNetParams,
      TimeSource:  blockchain.NewMedianTime(),
   })
   if err != nil {
      fmt.Printf("Failed to create chain instance: %v&#92;n", err)
      return
   }

   // Process a block.  For this example, we are going to intentionally
   // cause an error by trying to process the genesis block which already
   // exists.
   genesisBlock := dcrutil.NewBlock(chaincfg.MainNetParams.GenesisBlock)
   forkLen, isOrphan, err := chain.ProcessBlock(genesisBlock,
      blockchain.BFNone)
   if err != nil {
      fmt.Printf("Failed to create chain instance: %v&#92;n", err)
      return
   }
   isMainChain := !isOrphan &amp;&amp; forkLen == 0
   fmt.Printf("Block accepted. Is it on the main chain?: %v", isMainChain)
   fmt.Printf("Block accepted. Is it an orphan?: %v", isOrphan)

   // This output is dependent on the genesis block, and needs to be
   // updated if the mainnet genesis block is updated.
   // Output:
   // Failed to process block: already have block 267a53b5ee86c24a48ec37aee4f4e7c0c4004892b7259e695e9f5b321f1ab9d2
}

```
我们可以在这个文件里创建一个测试方法来调用:
```
func TestBlockchainInit(t testing.T) {
   ExampleBlockChain_ProcessBlock()
}

```
可以在Goland直接运行这个测试方法:
![image.png](https://upload-images.jianshu.io/upload_images/422094-f8268dddcec95c69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当然, 你也可以直接用命令行来测试,`go test -v -test.run TestBlockchainInit`, 如:
```
cd blockchain/
go test -v -test.run TestBlockchainInit
=== RUN   TestBlockchainInit
Failed to create chain instance: already have block 298e5cc3d985bfe7f81dc135f360abe089edd4396b86d2de66b0cef42b21d980
--- PASS: TestBlockchainInit (0.01s)
PASS
ok          github.com/decred/dcrd/blockchain        0.048s

```
`ExampleBlockChain_ProcessBlock`代码很好理解, 它先创建数据库, 然后调用`blockchain.New()`运行blockchain.  下面着重讲解`blockchain.New()`的代码.
## blockchain.New()
blockchain/chain.go
```
// New returns a BlockChain instance using the provided configuration details.
func New(config Config) (BlockChain, error) {
   // Enforce required config fields.
   if config.DB == nil {
      return nil, AssertError("blockchain.New database is nil")
   }
   if config.ChainParams == nil {
      return nil, AssertError("blockchain.New chain parameters nil")
   }

   // Generate a checkpoint by height map from the provided checkpoints.
   params := config.ChainParams
   var checkpointsByHeight map[int64]chaincfg.Checkpoint
   if len(params.Checkpoints) > 0 {
      checkpointsByHeight = make(map[int64]chaincfg.Checkpoint)
      for i := range params.Checkpoints {
         checkpoint := &amp;params.Checkpoints[i]
         checkpointsByHeight[checkpoint.Height] = checkpoint
      }
   }

   // Generate a deployment ID to version map from the provided params.
   deploymentVers, err := extractDeploymentIDVersions(params)
   if err != nil {
      return nil, err
   }

   b := BlockChain{
      checkpointsByHeight:           checkpointsByHeight,
      deploymentVers:                deploymentVers,
      db:                            config.DB,
      chainParams:                   params,
      timeSource:                    config.TimeSource,
      notifications:                 config.Notifications,
      sigCache:                      config.SigCache,
      indexManager:                  config.IndexManager,
      interrupt:                     config.Interrupt,
      index:                         newBlockIndex(config.DB, params),
      bestChain:                     newChainView(nil),
      orphans:                       make(map[chainhash.Hash]orphanBlock),
      prevOrphans:                   make(map[chainhash.Hash][]orphanBlock),
      mainchainBlockCache:           make(map[chainhash.Hash]dcrutil.Block),
      mainchainBlockCacheSize:       mainchainBlockCacheSize,
      deploymentCaches:              newThresholdCaches(params),
      isVoterMajorityVersionCache:   make(map[[stakeMajorityCacheKeySize]byte]bool),
      isStakeMajorityVersionCache:   make(map[[stakeMajorityCacheKeySize]byte]bool),
      calcPriorStakeVersionCache:    make(map[[chainhash.HashSize]byte]uint32),
      calcVoterVersionIntervalCache: make(map[[chainhash.HashSize]byte]uint32),
      calcStakeVersionCache:         make(map[[chainhash.HashSize]byte]uint32),
   }

   // Initialize the chain state from the passed database.  When the db
   // does not yet contain any chain state, both it and the chain state
   // will be initialized to contain only the genesis block.
   if err := b.initChainState(); err != nil {
      return nil, err
   }

   // Initialize and catch up all of the currently active optional indexes
   // as needed.
   if config.IndexManager != nil {
      err := config.IndexManager.Init(&amp;b, config.Interrupt)
      if err != nil {
         return nil, err
      }
   }

   b.subsidyCache = NewSubsidyCache(b.bestChain.Tip().height, b.chainParams)
   b.pruner = newChainPruner(&amp;b)

   // The version 5 database upgrade requires a full reindex.  Perform, or
   // resume, the reindex as needed.
   if err := b.maybeFinishV5Upgrade(); err != nil {
      return nil, err
   }

   log.Infof("Blockchain database version info: chain: %d, compression: "+
      "%d, block index: %d", b.dbInfo.version, b.dbInfo.compVer,
      b.dbInfo.bidxVer)

   tip := b.bestChain.Tip()
   log.Infof("Chain state: height %d, hash %v, total transactions %d, "+
      "work %v, stake version %v", tip.height, tip.hash,
      b.stateSnapshot.TotalTxns, tip.workSum, 0)

   return &amp;b, nil
}

```
以下是具体功能:

* Checkpoints加载
* 创建Blockchain{}对象

    * newBlockIndex() block索引
    * newChainView()  block索引2

* chainio.initChainState() 加载所有blockNode
* IndexManager.init() 初始化索引, 这些索引都在 blocchain/indexer/下
* NewSubsidyCache() 计算奖励(pow, pos, 团队)
* newChainPruner() 



## newBlockIndex() 初始化 chain.index
chain.index = newBlockIndex()
```
// newBlockIndex returns a new empty instance of a block index.  The index will
// be dynamically populated as block nodes are loaded from the database and
// manually added.
// 当block从数据库加载后, index会动态生成
func newBlockIndex(db database.DB, chainParams chaincfg.Params) blockIndex {
   return &amp;blockIndex{
      db:          db,
      chainParams: chainParams,
      index:       make(map[chainhash.Hash]blockNode), // hash => blockNode
      modified:    make(map[blockNode]struct{}),
      chainTips:   make(map[int64][]blockNode),
   }
}

```
b.index.AddNode(newNode) 只会在 初始block时 chainio.initChainState() 和 之后的 maybeAcceptBlock()中调用.
所以, 初始时就会将所有块index加载到 chain.index中.
如果不是孤儿区块，只要加入了区块链，均会在内存中有实例化的blockNode对象。也就是说，节点启动后，内存中为所有加入主链或者侧链的区块保留blockNode对象.
## newChainView() 初始化 chain.bestChain
chain.bestChain = newChainView()
主要代码在 chainview.go.
```
// chainView provides a flat view of a specific branch of the block chain from
// its tip back to the genesis block and provides various convenience functions
// for comparing chains.
//
// For example, assume a block chain with a side chain as depicted below:
//   genesis -> 1 -> 2 -> 3 -> 4  -> 5 ->  6  -> 7  -> 8
//                         &#92;-> 4a -> 5a -> 6a
//
// The chain view for the branch ending in 6a consists of:
//   genesis -> 1 -> 2 -> 3 -> 4a -> 5a -> 6a(tip 末梢)

```
chain.bestChain和chain.index是一对, chain.bestChain 也是一种索引.

* chain.index是 blockhash =&gt; blockNode, 核心是一个map
* chain.bestChain是 height =&gt; blockNode, 核心是一个数组

chain.bestChain只存储主链的索引!!
chain.bestChain 有一个blockNode的数组, [index] 是height, 所以可以通过height很快得到 blockNode.
chain.bestChain 维护的数组可以通过SetTip()快速切换, 比如SetTip(8)那就是一条从genesis到8的数组, SetTip()也是通过向前遍历blockNode.parent 来创建的.
chain.bestChain 有以下方法:

* SetTip() 
* NodeByHeight(height)
* Contains(blockNode)

个人感觉 bestChain 这个名字起得并不好, 初看并不知道和chain.index的关系. 但是有个best就知道和主链有关系了, 它只存储主链的节点.
## chainio.initChainState() 
代码在 chainio.go 中, 主要是从数据库中加载所有数据到内存, 这些数据包括: blockchain状态数据, 所有blockNode, stake状态数据, 3种状态的tickets. 流程如下:

* 加载数据库信息
* 加载chain state数据
* 从硬盘加载所有的blockNode, 每加载一个node都会调用:

    * chain.index addNode() 把这个node加到index中, 创建索引
    * chain.bestChain.SetTip() 也把这个node加到bestChain索引中

* stake.LoadBestNode() 加载stake数据. blockchain/stake/ticket.go

    * 加载stake数据库状态信息 stake chain state
    * 加载所有的liveTickets, missedTickets, revokedTickets
    * 计算stake node的 finalState (final state checksum from the live tickets pool.)
    * 为tip创建stakeNode






## IndexManager.init()
```
// IndexManager provides a generic interface that the is called when blocks are
// connected and disconnected to and from the tip of the main chain for the
// purpose of supporting optional indexes.

```
chain.indexManager和chain.index完全不是一码事, 没一毛钱关系, chain.indexManager是可选的, 主要为了为address, tx (如果配置文件中配置了txindex=1的话)建索引, 具体代码在 indexers/ 中

* chain.index 是必要的, blockhash =&gt; blockNode
* chain.indexManager 是非必要的



## 总结
Blockchain初始化, 最主要的工作就是将必要的数据库从硬盘加载到内存, 并为BlockNode创建一些索引为以后查询和处理数据用.