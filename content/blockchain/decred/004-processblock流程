---
title: ProcessBlock流程
date: 2019-06-14T00:00:00+08:00
tags: ["decred"]
categories: ["decred"]
draft: false

---
Blockchain 初始化好, 最主要的就是处理block, 判断这个块是否合法, 如果合法就加到主链上, 如果它的parent不在主链, 则加到侧链上. 加到侧链上好, 还要判断是否要将侧链转主链. 
代码如下: (blockchain/process.go), 重要的部分已加了注释.

```
func (b BlockChain) ProcessBlock(block dcrutil.Block, flags BehaviorFlags) (int64, bool, error) {
   b.chainLock.Lock()
   defer b.chainLock.Unlock()

   fastAdd := flags&amp;BFFastAdd == BFFastAdd

   blockHash := block.Hash()
   log.Tracef("Processing block %v", blockHash)
   currentTime := time.Now()
   defer func() {
      elapsedTime := time.Since(currentTime)
      log.Debugf("Block %v (height %v) finished processing in %s",
         blockHash, block.Height(), elapsedTime)
   }()

   // (1) 是否已存在, 如果已存在, 则返回错误
   // The block must not already exist in the main chain or side chains.
   if b.index.HaveBlock(blockHash) {
      str := fmt.Sprintf("already have block %v", blockHash)
      return 0, false, ruleError(ErrDuplicateBlock, str)
   }

   // (2) 是否在孤块中, 注意 b.index只有主链的数据, 所以 (1) 是不能判断孤块是否存在的
   // The block must not already exist as an orphan.
   if _, exists := b.orphans[blockHash]; exists {
      str := fmt.Sprintf("already have block (orphan) %v", blockHash)
      return 0, false, ruleError(ErrDuplicateBlock, str)
   }
   
   // (3) block的一致性检查
   // Perform preliminary sanity checks on the block and its transactions.
   err := checkBlockSanity(block, b.timeSource, flags, b.chainParams)
   if err != nil {
      return 0, false, err
   }

   // (4) 找到该block之前的checkpoint, 如果存在, 那么这个checkpoint的时间肯定是要 < block的时间的, 
   // 然后由checkpoint的时间到该block的时间再计算该block应该满足的难度值, 如果block的难度值 > 它, 就表明难度不符合. (注意, 难度值越大, 表明难度越小, 之后会专门讲解难度的章节)
   // Find the previous checkpoint and perform some additional checks based
   // on the checkpoint.  This provides a few nice properties such as
   // preventing old side chain blocks before the last checkpoint,
   // rejecting easy to mine, but otherwise bogus, blocks that could be
   // used to eat memory, and ensuring expected (versus claimed) proof of
   // work requirements since the previous checkpoint are met.
   blockHeader := &amp;block.MsgBlock().Header
   checkpointNode, err := b.findPreviousCheckpoint()
   if err != nil {
      return 0, false, err
   }
   if checkpointNode != nil {
      // Ensure the block timestamp is after the checkpoint timestamp.
      checkpointTime := time.Unix(checkpointNode.timestamp, 0)
      if blockHeader.Timestamp.Before(checkpointTime) {
         str := fmt.Sprintf("block %v has timestamp %v before "+
            "last checkpoint timestamp %v", blockHash,
            blockHeader.Timestamp, checkpointTime)
         return 0, false, ruleError(ErrCheckpointTimeTooOld, str)
      }

      if !fastAdd {
         // Even though the checks prior to now have already ensured the
         // proof of work exceeds the claimed amount, the claimed amount
         // is a field in the block header which could be forged.  This
         // check ensures the proof of work is at least the minimum
         // expected based on elapsed time since the last checkpoint and
         // maximum adjustment allowed by the retarget rules.
         duration := blockHeader.Timestamp.Sub(checkpointTime)
         requiredTarget := CompactToBig(b.calcEasiestDifficulty(
            checkpointNode.bits, duration))
         currentTarget := CompactToBig(blockHeader.Bits)
         if currentTarget.Cmp(requiredTarget) > 0 {
            str := fmt.Sprintf("block target difficulty of %064x "+
               "is too low when compared to the previous "+
               "checkpoint", currentTarget)
            return 0, false, ruleError(ErrDifficultyTooLow, str)
         }
      }
   }

   // (5) 如果该block的父是孤块, 则加入到孤块中, 直接返回
   // Handle orphan blocks.
   prevHash := &amp;blockHeader.PrevBlock
   if !b.index.HaveBlock(prevHash) {
      log.Infof("Adding orphan block %v with parent %v", blockHash,
         prevHash)
      b.addOrphanBlock(block)

      // The fork length of orphans is unknown since they, by definition, do
      // not connect to the best chain.
      return 0, true, nil
   }

   // (6) 尝试入主链
   // The block has passed all context independent checks and appears sane
   // enough to potentially accept it into the block chain.
   forkLen, err := b.maybeAcceptBlock(block, flags)
   if err != nil {
      return 0, false, err
   }

   // (7) 入主链后, 如果发现某孤块的父在该block, 则尝试将访报孤块入主链(同样调用maybeAcceptBlock()方法)
   // Accept any orphan blocks that depend on this block (they are no
   // longer orphans) and repeat for those accepted blocks until there are
   // no more.
   err = b.processOrphans(blockHash, flags)
   if err != nil {
      return 0, false, err
   }

   log.Debugf("Accepted block %v", blockHash)

   return forkLen, false, nil
}

```
以下是ProcessBlock的流程图, 之后的章节会逐个介绍每一步的操作.
![image.png](https://upload-images.jianshu.io/upload_images/422094-ea76580e15590387.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 一致性检查 checkBlockSanity()
将一些不合法的块提前出局. 根据block的规则检查每一项是否合法.
所有的代码都在 validate.go
 检测项有:

* Block Header检查 checkBlockHeaderSanity() (下文有详细描述)
* 票价检查 checkProofOfStake() (下文有详细描述)
* 至少有一个普通交易
* 块数据大小
* 普通交易检查

    * 第一个交易必须是coinbase
    * 不能是stake交易
    * 每一个交易检查一致性 CheckTransactionSanity() (下文有详细描述)

* stake交易检查

    * 每一个交易检查一致性 CheckTransactionSanity() (下文有详细描述)
    * 不能有普通交易
    * 退票数量不能 &gt; 255
    * 买票数量是否和block header的FreshStake一致
    * 中票数量是否和block header的Voters一致
    * 退票数量是否和block header的Revocations一致
    * 中票对父的验证结果是否和block header的VoteBits一致

* 普通交易merkle root检查
* stake交易merkle root检查
* 所有交易中, 不能有重复交易
* 签名操作不能 &gt; 1000000 / 200

代码如下:
```
func checkBlockSanity(block dcrutil.Block, timeSource MedianTimeSource, flags BehaviorFlags, chainParams chaincfg.Params) error {
   msgBlock := block.MsgBlock()
   header := &amp;msgBlock.Header
   err := checkBlockHeaderSanity(header, timeSource, flags, chainParams)
   if err != nil {
      return err
   }

   // All ticket purchases must meet the difficulty specified by the block
   // header.
   err = checkProofOfStake(block, chainParams.MinimumStakeDiff)
   if err != nil {
      return err
   }

   // A block must have at least one regular transaction.
   numTx := len(msgBlock.Transactions)
   if numTx == 0 {
      return ruleError(ErrNoTransactions, "block does not contain "+
         "any transactions")
   }

   // A block must not exceed the maximum allowed block payload when
   // serialized.
   //
   // This is a quick and context-free sanity check of the maximum block
   // size according to the wire protocol.  Even though the wire protocol
   // already prevents blocks bigger than this limit, there are other
   // methods of receiving a block that might not have been checked
   // already.  A separate block size is enforced later that takes into
   // account the network-specific block size and the results of block
   // size votes.  Typically that block size is more restrictive than this
   // one.
   serializedSize := msgBlock.SerializeSize()
   if serializedSize > wire.MaxBlockPayload {
      str := fmt.Sprintf("serialized block is too big - got %d, "+
         "max %d", serializedSize, wire.MaxBlockPayload)
      return ruleError(ErrBlockTooBig, str)
   }
   if header.Size != uint32(serializedSize) {
      str := fmt.Sprintf("serialized block is not size indicated in "+
         "header - got %d, expected %d", header.Size,
         serializedSize)
      return ruleError(ErrWrongBlockSize, str)
   }

   // The first transaction in a block's regular tree must be a coinbase.
   transactions := block.Transactions()
   if !IsCoinBaseTx(transactions[0].MsgTx()) {
      return ruleError(ErrFirstTxNotCoinbase, "first transaction in "+
         "block is not a coinbase")
   }

   // A block must not have more than one coinbase.
   for i, tx := range transactions[1:] {
      if IsCoinBaseTx(tx.MsgTx()) {
         str := fmt.Sprintf("block contains second coinbase at "+
            "index %d", i+1)
         return ruleError(ErrMultipleCoinbases, str)
      }
   }

   // Do some preliminary checks on each regular transaction to ensure they
   // are sane before continuing.
   for i, tx := range transactions {
      // A block must not have stake transactions in the regular
      // transaction tree.
      msgTx := tx.MsgTx()
      txType := stake.DetermineTxType(msgTx)
      if txType != stake.TxTypeRegular {
         errStr := fmt.Sprintf("block contains a stake "+
            "transaction in the regular transaction tree at "+
            "index %d", i)
         return ruleError(ErrStakeTxInRegularTree, errStr)
      }

      err := CheckTransactionSanity(msgTx, chainParams)
      if err != nil {
         return err
      }
   }

   // Do some preliminary checks on each stake transaction to ensure they
   // are sane while tallying each type before continuing.
   stakeValidationHeight := uint32(chainParams.StakeValidationHeight)
   var totalTickets, totalVotes, totalRevocations int64
   var totalYesVotes int64
   for txIdx, stx := range msgBlock.STransactions {
      err := CheckTransactionSanity(stx, chainParams)
      if err != nil {
         return err
      }

      // A block must not have regular transactions in the stake
      // transaction tree.
      txType := stake.DetermineTxType(stx)
      if txType == stake.TxTypeRegular {
         errStr := fmt.Sprintf("block contains regular "+
            "transaction in stake transaction tree at "+
            "index %d", txIdx)
         return ruleError(ErrRegTxInStakeTree, errStr)
      }

      switch txType {
      case stake.TxTypeSStx:
         totalTickets++

      case stake.TxTypeSSGen:
         totalVotes++

         // All votes in a block must commit to the parent of the
         // block once stake validation height has been reached.
         if header.Height >= stakeValidationHeight {
            votedHash, votedHeight := stake.SSGenBlockVotedOn(stx)
            if (votedHash != header.PrevBlock) || (votedHeight !=
               header.Height-1) {

               errStr := fmt.Sprintf("vote %s at index %d is "+
                  "for parent block %s (height %d) versus "+
                  "expected parent block %s (height %d)",
                  stx.TxHash(), txIdx, votedHash,
                  votedHeight, header.PrevBlock,
                  header.Height-1)
               return ruleError(ErrVotesOnWrongBlock, errStr)
            }

            // Tally how many votes approve the previous block for use
            // when validating the header commitment.
            if voteBitsApproveParent(stake.SSGenVoteBits(stx)) {
               totalYesVotes++
            }
         }

      case stake.TxTypeSSRtx:
         totalRevocations++
      }
   }

   // A block must not contain more than the maximum allowed number of
   // revocations.
   if totalRevocations > maxRevocationsPerBlock {
      errStr := fmt.Sprintf("block contains %d revocations which "+
         "exceeds the maximum allowed amount of %d",
         totalRevocations, maxRevocationsPerBlock)
      return ruleError(ErrTooManyRevocations, errStr)
   }

   // A block must only contain stake transactions of the the allowed
   // types.
   //
   // NOTE: This is not possible to hit at the time this comment was
   // written because all transactions which are not specifically one of
   // the recognized stake transaction forms are considered regular
   // transactions and those are rejected above.  However, if a new stake
   // transaction type is added, that implicit condition would no longer
   // hold and therefore an explicit check is performed here.
   numStakeTx := int64(len(msgBlock.STransactions))
   calcStakeTx := totalTickets + totalVotes + totalRevocations
   if numStakeTx != calcStakeTx {
      errStr := fmt.Sprintf("block contains an unexpected number "+
         "of stake transactions (contains %d, expected %d)",
         numStakeTx, calcStakeTx)
      return ruleError(ErrNonstandardStakeTx, errStr)
   }

   // A block header must commit to the actual number of tickets purchases that
   // are in the block.
   if int64(header.FreshStake) != totalTickets {
      errStr := fmt.Sprintf("block header commitment to %d ticket "+
         "purchases does not match %d contained in the block",
         header.FreshStake, totalTickets)
      return ruleError(ErrFreshStakeMismatch, errStr)
   }

   // A block header must commit to the the actual number of votes that are
   // in the block.
   if int64(header.Voters) != totalVotes {
      errStr := fmt.Sprintf("block header commitment to %d votes "+
         "does not match %d contained in the block",
         header.Voters, totalVotes)
      return ruleError(ErrVotesMismatch, errStr)
   }

   // A block header must commit to the actual number of revocations that
   // are in the block.
   if int64(header.Revocations) != totalRevocations {
      errStr := fmt.Sprintf("block header commitment to %d revocations "+
         "does not match %d contained in the block",
         header.Revocations, totalRevocations)
      return ruleError(ErrRevocationsMismatch, errStr)
   }

   // A block header must commit to the same previous block acceptance
   // semantics expressed by the votes once stake validation height has
   // been reached.
   if header.Height >= stakeValidationHeight {
      totalNoVotes := totalVotes - totalYesVotes
      headerApproves := headerApprovesParent(header)
      votesApprove := totalYesVotes > totalNoVotes
      if headerApproves != votesApprove {
         errStr := fmt.Sprintf("block header commitment to previous "+
            "block approval does not match votes (header claims: %v, "+
            "votes: %v)", headerApproves, votesApprove)
         return ruleError(ErrIncongruentVotebit, errStr)
      }
   }

   // A block must not contain anything other than ticket purchases prior to
   // stake validation height.
   //
   // NOTE: This case is impossible to hit at this point at the time this
   // comment was written since the votes and revocations have already been
   // proven to be zero before stake validation height and the only other
   // type at the current time is ticket purchases, however, if another
   // stake type is ever added, consensus would break without this check.
   // It's better to be safe and it's a cheap check.
   if header.Height < stakeValidationHeight {
      if int64(len(msgBlock.STransactions)) != totalTickets {
         errStr := fmt.Sprintf("block contains stake "+
            "transactions other than ticket purchases before "+
            "stake validation height %d (total: %d, expected %d)",
            uint32(chainParams.StakeValidationHeight),
            len(msgBlock.STransactions), header.FreshStake)
         return ruleError(ErrInvalidEarlyStakeTx, errStr)
      }
   }

   // Build merkle tree and ensure the calculated merkle root matches the
   // entry in the block header.  This also has the effect of caching all
   // of the transaction hashes in the block to speed up future hash
   // checks.  Bitcoind builds the tree here and checks the merkle root
   // after the following checks, but there is no reason not to check the
   // merkle root matches here.
   merkles := BuildMerkleTreeStore(block.Transactions())
   calculatedMerkleRoot := merkles[len(merkles)-1]
   if !header.MerkleRoot.IsEqual(calculatedMerkleRoot) {
      str := fmt.Sprintf("block merkle root is invalid - block "+
         "header indicates %v, but calculated value is %v",
         header.MerkleRoot, calculatedMerkleRoot)
      return ruleError(ErrBadMerkleRoot, str)
   }

   // Build the stake tx tree merkle root too and check it.
   merkleStake := BuildMerkleTreeStore(block.STransactions())
   calculatedStakeMerkleRoot := merkleStake[len(merkleStake)-1]
   if !header.StakeRoot.IsEqual(calculatedStakeMerkleRoot) {
      str := fmt.Sprintf("block stake merkle root is invalid - block"+
         " header indicates %v, but calculated value is %v",
         header.StakeRoot, calculatedStakeMerkleRoot)
      return ruleError(ErrBadMerkleRoot, str)
   }

   // Check for duplicate transactions.  This check will be fairly quick
   // since the transaction hashes are already cached due to building the
   // merkle trees above.
   existingTxHashes := make(map[chainhash.Hash]struct{})
   stakeTransactions := block.STransactions()
   allTransactions := append(transactions, stakeTransactions...)

   for _, tx := range allTransactions {
      hash := tx.Hash()
      if _, exists := existingTxHashes[hash]; exists {
         str := fmt.Sprintf("block contains duplicate "+
            "transaction %v", hash)
         return ruleError(ErrDuplicateTx, str)
      }
      existingTxHashes[hash] = struct{}{}
   }

   // The number of signature operations must be less than the maximum
   // allowed per block.
   totalSigOps := 0
   for _, tx := range allTransactions {
      // We could potentially overflow the accumulator so check for
      // overflow.
      lastSigOps := totalSigOps

      msgTx := tx.MsgTx()
      isCoinBase := IsCoinBaseTx(msgTx)
      isSSGen := stake.IsSSGen(msgTx)
      totalSigOps += CountSigOps(tx, isCoinBase, isSSGen)
      if totalSigOps < lastSigOps || totalSigOps > MaxSigOpsPerBlock {
         str := fmt.Sprintf("block contains too many signature "+
            "operations - got %v, max %v", totalSigOps,
            MaxSigOpsPerBlock)
         return ruleError(ErrTooManySigOps, str)
      }
   }

   return nil
}


```

## 票价检查 checkProofOfStake()
这一步, 检查该block所有的买票交易的TxOut[0].value. DCR规定如果是买票交易, 它的第一个输出就是票价. 这里检查票价是否 &gt; header里的SBits 和 是否 &gt; 最低票价.
```
func checkProofOfStake(block dcrutil.Block, posLimit int64) error {
   msgBlock := block.MsgBlock()
   for _, staketx := range block.STransactions() {
      msgTx := staketx.MsgTx()
      if stake.IsSStx(msgTx) {
         commitValue := msgTx.TxOut[0].Value // 票价

         // Check for underflow block sbits.
         if commitValue < msgBlock.Header.SBits {
            errStr := fmt.Sprintf("Stake tx %v has a "+
               "commitment value less than the "+
               "minimum stake difficulty specified in"+
               " the block (%v)", staketx.Hash(),
               msgBlock.Header.SBits)
            return ruleError(ErrNotEnoughStake, errStr)
         }

         // Check if it's above the PoS limit.
         if commitValue < posLimit {
            errStr := fmt.Sprintf("Stake tx %v has a "+
               "commitment value less than the "+
               "minimum stake difficulty for the "+
               "network (%v)", staketx.Hash(),
               posLimit)
            return ruleError(ErrStakeBelowMinimum, errStr)
         }
      }
   }

   return nil
}


```

## 头部检查 checkBlockHeaderSanity()

* 难度检查, 难度值是否在合理区间内, hash是否&lt;难度值, 关于难度详细描述请至 [工作量证明](https://teakki.com/p/5c7f7022b1029f607607feae)
* 时间检查, 时间不能有nsec纳秒, 只能精确到秒
* 时间检查, 不能离当前时间太远(2小时内)
* 如果当前还没到投票的高度(4096), 则不能有关于投票的任何票信息, 则检查:

    * header.Voters 必须 = 0
    * header.Revocations 必须 = 0
    * header.VoteBits == 0x0001
    * header.FinalState == [6]byte{0x00}

* 如果当前到了投票的高度(4096), 则检查:

    * 投票数 header.Voters 必须 &gt;= 3 个

* 投票数不能 &gt; 5个
* 新买票数不能 &gt; 20个

代码如下:
```
func checkBlockHeaderSanity(header wire.BlockHeader, timeSource MedianTimeSource, flags BehaviorFlags, chainParams chaincfg.Params) error {
   // The stake validation height should always be at least stake enabled
   // height, so assert it because the code below relies on that assumption.
   stakeValidationHeight := uint32(chainParams.StakeValidationHeight)
   stakeEnabledHeight := uint32(chainParams.StakeEnabledHeight)
   if stakeEnabledHeight > stakeValidationHeight {
      return AssertError(fmt.Sprintf("checkBlockHeaderSanity called "+
         "with stake enabled height %d after stake validation "+
         "height %d", stakeEnabledHeight, stakeValidationHeight))
   }

   // Ensure the proof of work bits in the block header is in min/max
   // range and the block hash is less than the target value described by
   // the bits.
   err := checkProofOfWork(header, chainParams.PowLimit, flags)
   if err != nil {
      return err
   }

   // A block timestamp must not have a greater precision than one second.
   // This check is necessary because Go time.Time values support
   // nanosecond precision whereas the consensus rules only apply to
   // seconds and it's much nicer to deal with standard Go time values
   // instead of converting to seconds everywhere.
   if !header.Timestamp.Equal(time.Unix(header.Timestamp.Unix(), 0)) {
      str := fmt.Sprintf("block timestamp of %v has a higher "+
         "precision than one second", header.Timestamp)
      return ruleError(ErrInvalidTime, str)
   }

   // Ensure the block time is not too far in the future.
   maxTimestamp := timeSource.AdjustedTime().Add(time.Second 
      MaxTimeOffsetSeconds)
   if header.Timestamp.After(maxTimestamp) {
      str := fmt.Sprintf("block timestamp of %v is too far in the "+
         "future", header.Timestamp)
      return ruleError(ErrTimeTooNew, str)
   }

   // A block must not contain any votes or revocations, its vote bits
   // must be 0x0001, and its final state must be all zeroes before
   // stake validation begins.
   if header.Height < stakeValidationHeight {
      if header.Voters > 0 {
         errStr := fmt.Sprintf("block at height %d commits to "+
            "%d votes before stake validation height %d",
            header.Height, header.Voters,
            stakeValidationHeight)
         return ruleError(ErrInvalidEarlyStakeTx, errStr)
      }

      if header.Revocations > 0 {
         errStr := fmt.Sprintf("block at height %d commits to "+
            "%d revocations before stake validation height %d",
            header.Height, header.Revocations,
            stakeValidationHeight)
         return ruleError(ErrInvalidEarlyStakeTx, errStr)
      }

      if header.VoteBits != earlyVoteBitsValue {
         errStr := fmt.Sprintf("block at height %d commits to "+
            "invalid vote bits before stake validation "+
            "height %d (expected %x, got %x)",
            header.Height, stakeValidationHeight,
            earlyVoteBitsValue, header.VoteBits)
         return ruleError(ErrInvalidEarlyVoteBits, errStr)
      }

      if header.FinalState != earlyFinalState {
         errStr := fmt.Sprintf("block at height %d commits to "+
            "invalid final state before stake validation "+
            "height %d (expected %x, got %x)",
            header.Height, stakeValidationHeight,
            earlyFinalState, header.FinalState)
         return ruleError(ErrInvalidEarlyFinalState, errStr)
      }
   }

   // A block must not contain fewer votes than the minimum required to
   // reach majority once stake validation height has been reached.
   if header.Height >= stakeValidationHeight {
      majority := (chainParams.TicketsPerBlock / 2) + 1
      if header.Voters < majority {
         errStr := fmt.Sprintf("block does not commit to enough "+
            "votes (min: %d, got %d)", majority,
            header.Voters)
         return ruleError(ErrNotEnoughVotes, errStr)
      }
   }

   // The block header must not claim to contain more votes than the
   // maximum allowed.
   if header.Voters > chainParams.TicketsPerBlock {
      errStr := fmt.Sprintf("block commits to too many votes (max: "+
         "%d, got %d)", chainParams.TicketsPerBlock, header.Voters)
      return ruleError(ErrTooManyVotes, errStr)
   }

   // The block must not contain more ticket purchases than the maximum
   // allowed.
   if header.FreshStake > chainParams.MaxFreshStakePerBlock {
      errStr := fmt.Sprintf("block commits to too many ticket "+
         "purchases (max: %d, got %d)",
         chainParams.MaxFreshStakePerBlock, header.FreshStake)
      return ruleError(ErrTooManySStxs, errStr)
   }

   return nil
}


```

## 交易检查 CheckTransactionSanity()

* 输入 &gt; 0
* 输出 &gt; 0
* 交易大小 &lt; 393216 byte
* 如果是coinbase tx, 则检查:

    * 引用的output必须为空
    * The fraud proof must also be null.
    * SignatureScript 长度检查

* 如果是投票(SSGen)交易

    * SignatureScript 长度检查
    * TxIn[0].SignatureScript === StakeBaseSigScript. TxIn[0] 必须是 StakeBaseSigScript
    * TxIn[1].PreviousOutPoint. TxIn[1]引用的是Ticket 的out , 不能为空

* 否则, 则对每个TxIn都检查它的PreviousOutPoint不能为空
* 检查每个Out:

    * 输出value 不能 &lt; 0, 也不能 &gt; 最大值(210000001e8)
    * 输出的总value也不能 &gt; 最大值(210000001e8)
    * 如果是普通交易, 它的输出有stake交易的输出类型, 如 OP_SSTX, OP_SSRTX, OP_SSGEN, or OP_SSTX_CHANGE.

* 不能引用相同的PreviousOutPoint

代码如下:
```
func CheckTransactionSanity(tx wire.MsgTx, params chaincfg.Params) error {
   // A transaction must have at least one input.
   if len(tx.TxIn) == 0 {
      return ruleError(ErrNoTxInputs, "transaction has no inputs")
   }

   // A transaction must have at least one output.
   if len(tx.TxOut) == 0 {
      return ruleError(ErrNoTxOutputs, "transaction has no outputs")
   }

   // A transaction must not exceed the maximum allowed size when
   // serialized.
   serializedTxSize := tx.SerializeSize()
   if serializedTxSize > params.MaxTxSize {
      str := fmt.Sprintf("serialized transaction is too big - got "+
         "%d, max %d", serializedTxSize, params.MaxTxSize)
      return ruleError(ErrTxTooBig, str)
   }

   // Coinbase script length must be between min and max length.
   isVote := stake.IsSSGen(tx)
   if IsCoinBaseTx(tx) {
      // The referenced outpoint must be null.
      if !isNullOutpoint(&amp;tx.TxIn[0].PreviousOutPoint) {
         str := fmt.Sprintf("coinbase transaction did not use " +
            "a null outpoint")
         return ruleError(ErrBadCoinbaseOutpoint, str)
      }

      // The fraud proof must also be null.
      if !isNullFraudProof(tx.TxIn[0]) {
         str := fmt.Sprintf("coinbase transaction fraud proof " +
            "was non-null")
         return ruleError(ErrBadCoinbaseFraudProof, str)
      }

      slen := len(tx.TxIn[0].SignatureScript)
      if slen < MinCoinbaseScriptLen || slen > MaxCoinbaseScriptLen {
         str := fmt.Sprintf("coinbase transaction script "+
            "length of %d is out of range (min: %d, max: "+
            "%d)", slen, MinCoinbaseScriptLen,
            MaxCoinbaseScriptLen)
         return ruleError(ErrBadCoinbaseScriptLen, str)
      }
   } else if isVote {
      // Check script length of stake base signature.
      slen := len(tx.TxIn[0].SignatureScript)
      if slen < MinCoinbaseScriptLen || slen > MaxCoinbaseScriptLen {
         str := fmt.Sprintf("stakebase transaction script "+
            "length of %d is out of range (min: %d, max: "+
            "%d)", slen, MinCoinbaseScriptLen,
            MaxCoinbaseScriptLen)
         return ruleError(ErrBadStakebaseScriptLen, str)
      }

      // The script must be set to the one specified by the network.
      // Check script length of stake base signature.
      if !bytes.Equal(tx.TxIn[0].SignatureScript,
         params.StakeBaseSigScript) {
         str := fmt.Sprintf("stakebase transaction signature "+
            "script was set to disallowed value (got %x, "+
            "want %x)", tx.TxIn[0].SignatureScript,
            params.StakeBaseSigScript)
         return ruleError(ErrBadStakebaseScrVal, str)
      }

      // The ticket reference hash in an SSGen tx must not be null.
      ticketHash := &amp;tx.TxIn[1].PreviousOutPoint
      if isNullOutpoint(ticketHash) {
         return ruleError(ErrBadTxInput, "ssgen tx ticket input"+
            " refers to previous output that is null")
      }
   } else {
      // Previous transaction outputs referenced by the inputs to
      // this transaction must not be null except in the case of
      // stakebases for votes.
      for _, txIn := range tx.TxIn {
         prevOut := &amp;txIn.PreviousOutPoint
         if isNullOutpoint(prevOut) {
            return ruleError(ErrBadTxInput, "transaction "+
               "input refers to previous output that "+
               "is null")
         }
      }
   }

   // Ensure the transaction amounts are in range.  Each transaction output
   // must not be negative or more than the max allowed per transaction.  Also,
   // the total of all outputs must abide by the same restrictions.  All
   // amounts in a transaction are in a unit value known as an atom.  One
   // Decred is a quantity of atoms as defined by the AtomsPerCoin constant.
   //
   // Also ensure that non-stake transaction output scripts do not contain any
   // stake opcodes.
   isTicket := !isVote &amp;&amp; stake.IsSStx(tx)
   isRevocation := !isVote &amp;&amp; !isTicket &amp;&amp; stake.IsSSRtx(tx)
   isStakeTx := isVote || isTicket || isRevocation
   var totalAtom int64
   for txOutIdx, txOut := range tx.TxOut {
      atom := txOut.Value
      if atom < 0 {
         str := fmt.Sprintf("transaction output has negative value of %v",
            atom)
         return ruleError(ErrBadTxOutValue, str)
      }
      if atom > dcrutil.MaxAmount {
         str := fmt.Sprintf("transaction output value of %v is higher than "+
            "max allowed value of %v", atom, dcrutil.MaxAmount)
         return ruleError(ErrBadTxOutValue, str)
      }

      // Two's complement int64 overflow guarantees that any overflow is
      // detected and reported.  This is impossible for Decred, but perhaps
      // possible if an alt increases the total money supply.
      totalAtom += atom
      if totalAtom < 0 {
         str := fmt.Sprintf("total value of all transaction outputs "+
            "exceeds max allowed value of %v", dcrutil.MaxAmount)
         return ruleError(ErrBadTxOutValue, str)
      }
      if totalAtom > dcrutil.MaxAmount {
         str := fmt.Sprintf("total value of all transaction outputs is %v "+
            "which is higher than max allowed value of %v", totalAtom,
            dcrutil.MaxAmount)
         return ruleError(ErrBadTxOutValue, str)
      }

      // Ensure that non-stake transactions have no outputs with opcodes
      // OP_SSTX, OP_SSRTX, OP_SSGEN, or OP_SSTX_CHANGE.
      if !isStakeTx {
         hasOp, err := txscript.ContainsStakeOpCodes(txOut.PkScript)
         if err != nil {
            return ruleError(ErrScriptMalformed, err.Error())
         }
         if hasOp {
            str := fmt.Sprintf("non-stake transaction output %d contains "+
               "stake opcode", txOutIdx)
            return ruleError(ErrRegTxCreateStakeOut, str)
         }
      }
   }

   // Check for duplicate transaction inputs.
   existingTxOut := make(map[wire.OutPoint]struct{})
   for _, txIn := range tx.TxIn {
      if _, exists := existingTxOut[txIn.PreviousOutPoint]; exists {
         return ruleError(ErrDuplicateTxInputs, "transaction "+
            "contains duplicate inputs")
      }
      existingTxOut[txIn.PreviousOutPoint] = struct{}{}
   }

   return nil
}


```

## 总结
checkBlockSanity() 检查的是头部和每个交易的合法性. 这一步检查好后, 之后就在链上检查, 然后入链.

# 入链检查
```
func (b BlockChain) maybeAcceptBlock(block dcrutil.Block, flags BehaviorFlags) (int64, error) {
   // (1) 父块必须存在
   // This function should never be called with orphan blocks or the
   // genesis block.
   prevHash := &amp;block.MsgBlock().Header.PrevBlock
   prevNode := b.index.LookupNode(prevHash)
   if prevNode == nil {
      str := fmt.Sprintf("previous block %s is not known", prevHash)
      return 0, ruleError(ErrMissingParent, str)
   }
   
   // (2) 父块状态一定是valid
   // There is no need to validate the block if an ancestor is already
   // known to be invalid.
   if b.index.NodeStatus(prevNode).KnownInvalid() {
      str := fmt.Sprintf("previous block %s is known to be invalid",
         prevHash)
      return 0, ruleError(ErrInvalidAncestorBlock, str)
   }
   // (3) 入链检查1
   // The block must pass all of the validation rules which depend on having
   // the headers of all ancestors available, but do not rely on having the
   // full block data of all ancestors available.
   err := b.checkBlockPositional(block, prevNode, flags)
   if err != nil {
      return 0, err
   }
   // (3) 入链检查2
   // The block must pass all of the validation rules which depend on having
   // the full block data for all of its ancestors available.
   err = b.checkBlockContext(block, prevNode, flags)
   if err != nil {
      return 0, err
   }

   // Prune stake nodes which are no longer needed before creating a new
   // node.
   b.pruner.pruneChainIfNeeded()
   // (4) 将块数据保存
   // Insert the block into the database if it's not already there.  Even
   // though it is possible the block will ultimately fail to connect, it
   // has already passed all proof-of-work and validity tests which means
   // it would be prohibitively expensive for an attacker to fill up the
   // disk with a bunch of blocks that fail to connect.  This is necessary
   // since it allows block download to be decoupled from the much more
   // expensive connection logic.  It also has some other nice properties
   // such as making blocks that never become part of the main chain or
   // blocks that fail to connect available for further analysis.
   err = b.db.Update(func(dbTx database.Tx) error {
      return dbMaybeStoreBlock(dbTx, block)
   })
   if err != nil {
      return 0, err
   }

   // 构造blockNode, 加索引, 保存blockNode
   // Create a new block node for the block and add it to the block index.
   // The block could either be on a side chain or the main chain, but it
   // starts off as a side chain regardless.
   blockHeader := &amp;block.MsgBlock().Header
   newNode := newBlockNode(blockHeader, prevNode)
   newNode.populateTicketInfo(stake.FindSpentTicketsInBlock(block.MsgBlock()))
   newNode.status = statusDataStored
   b.index.AddNode(newNode)

   // Ensure the new block index entry is written to the database.
   err = b.flushBlockIndex()
   if err != nil {
      return 0, err
   }

   // Notify the caller when the block intends to extend the main chain,
   // the chain believes it is current, and the block has passed all of the
   // sanity and contextual checks, such as having valid proof of work,
   // valid merkle and stake roots, and only containing allowed votes and
   // revocations.
   //
   // This allows the block to be relayed before doing the more expensive
   // connection checks, because even though the block might still fail
   // to connect and becomes the new main chain tip, that is quite rare in
   // practice since a lot of work was expended to create a block that
   // satisifies the proof of work requirement.
   //
   // Notice that the chain lock is not released before sending the
   // notification.  This is intentional and must not be changed without
   // understanding why!
   if b.isCurrent() &amp;&amp; b.bestChain.Tip() == prevNode {
      b.sendNotification(NTNewTipBlockChecked, block)
   }

   // Fetching a stake node could enable a new DoS vector, so restrict
   // this only to blocks that are recent in history.
   if newNode.height < b.bestChain.Tip().height-minMemoryNodes {
      newNode.stakeNode, err = b.fetchStakeNode(newNode)
      if err != nil {
         return 0, err
      }
   }

   // Grab the parent block since it is required throughout the block
   // connection process.
   parent, err := b.fetchBlockByNode(newNode.parent)
   if err != nil {
      return 0, err
   }
   
   // 入链
   // Connect the passed block to the chain while respecting proper chain
   // selection according to the chain with the most proof of work.  This
   // also handles validation of the transaction scripts.
   forkLen, err := b.connectBestChain(newNode, block, parent, flags)
   if err != nil {
      return 0, err
   }

   // Notify the caller that the new block was accepted into the block
   // chain.  The caller would typically want to react by relaying the
   // inventory to other peers unless it was already relayed above
   // via NTNewTipBlockChecked.
   bestHeight := b.bestChain.Tip().height
   b.chainLock.Unlock()
   b.sendNotification(NTBlockAccepted, &amp;BlockAcceptedNtfnsData{
      BestHeight: bestHeight,
      ForkLen:    forkLen,
      Block:      block,
   })
   b.chainLock.Lock()

   return forkLen, nil
}

```
虽然已经对block进行了checkBlockSanity(), 但这只是检查常规的数据, 还必须将该block放到链上作个检查, 这就是入链检查. 入链检查分两部分:
checkBlockPositional()和checkBlockContext()在之前的代码版本里是合二为一的, 它们的区别是:

* checkBlockPositional() 不需要祖先块有全部的块数据, 只需要有全部的头部数据就行了.
* checkBlockContext() 则需要祖先块有全部的块数据.

checkBlockPositional()没有议题是否激活判断, checkBlockContext()里有(检查block size和 tx finalized都需要判断)
所以, checkBlockPositional()里可以计算PoW难度, checkBlockContext()里计算PoS难度(需要用到state数据) ??
## checkBlockPositional()  入链检查1

* checkBlockHeaderPositional() 头部检查
* 所有tx是否已过期
* 检查coinbase交易是否包含height字段

tx是否已过期, 则由tx的Expiry字段决定的, 如果Expiry &lt; 当前高度, 表示该交易已过期, 就不能打包了. 这个字段避免了交易因堵塞或交易费太少而长时间不能打包又不能取消的问题.
### checkBlockHeaderPositional() 入链检查头部

* 难度检查: calcNextRequiredDifficulty() 根据prevNode计算当前的难度, 检查是否一致
* 时间检查: 根据前11个块的时间计算中间值, 当前块的时间必须 &gt; 这个中间值
* checkpoint检查, 找到前一个checkpoint, 该块高度必须 &gt; checkpoint高度



## checkBlockContext() 入链检查2

* checkBlockHeaderContext() 头部检查
* block size不能大于393216 bytes(0.39M), 这个值可能会因为议题的激活而变化, 当前只有一个值393216
* 确保每个交易都finalized
* checkAllowedVotes()
* checkAllowedRevocations()

交易是finalized意思是交易不能提前打包.
每个交易都有LockTime, LockTime  表示 时间 或 高度, &lt; 5e8就是height, &gt; 则是时间, 意思是最大的height是5e8
协议规定, 只有当前时间大于或等于locktime时间时, 这笔转账才可以被广播和打包，否则节点将会丢弃这样的转账.
所以这个交易的lockTime必须要&lt;当前height或时间才是合法的
### checkBlockHeaderContext() 入链检查头部

* 基于父块计算票价, 看是否一致 ()
* 计算StakeVersion, 判断是否一致
* header.PoolSize 票池是否正确, 该值的大小必须 === parentStakeNode.PoolSize()
* header.FinalState 是否正确, 该值必须 == parentStakeNode.FinalState()



## stakeNode
我们已经知道了 blockNode.  stakeNode和blockNode类似, 是保存了stake信息. stakeNode依附于blockNode, 它只保存了stake信息, 不会保存parentNode. 所以blockNode有链的层级关系, blockNode有stakeNode字段.
statkeNode类中除了header里的一些字段外, 还有stake的一些字段:
```
// stakeNode contains all the consensus information required for the
// staking system.  The node also caches information required to add or
// remove stake nodes, so that the stake node itself may be pruneable
// to save memory while maintaining high throughput efficiency for the
// evaluation of sidechains.
stakeNode      stake.Node
newTickets     []chainhash.Hash
ticketsVoted   []chainhash.Hash
ticketsRevoked []chainhash.Hash

```
stake.Node 类中存储了几种tickets和nextWinners及finalState, 所有字段如下:
blockchain/stake/tickets.go
```
type Node struct {
   height uint32

   // liveTickets is the treap of the live tickets for this node.
   liveTickets tickettreap.Immutable

   // missedTickets is the treap of missed tickets for this node.
   missedTickets tickettreap.Immutable

   // revokedTickets is the treap of revoked tickets for this node.
   revokedTickets tickettreap.Immutable

   // databaseUndoUpdate is the cache of the database used to undo
   // the current node's addition to the blockchain.
   databaseUndoUpdate UndoTicketDataSlice

   // databaseBlockTickets is the cache of the new tickets to insert
   // into the database.
   databaseBlockTickets ticketdb.TicketHashes

   // nextWinners is the list of the next winners for this block.
   nextWinners []chainhash.Hash

   // finalState is the calculated final state checksum from the live
   // tickets pool.
   finalState [6]byte

   // params is the blockchain parameters.
   params chaincfg.Params
}

```
stakeNode这些信息如何加载? 在 [2. Decred 数据库存储](https://teakki.com/p/5c78d010b1029f60760615bd)[ ](https://teakki.com/pe/5c78d010b1029f60760615bdblockNode在数据库里是不存储)中, 我们知道blockNode在数据库仅会存储 stake的 ticketsVoted, ticketsRevoked 数据, 其它stake数据是不会存储的, 所以这些数据必须要加载 block 来解析出.
思考这个场景:
```
1 -> 2 -> 3 -> 4--> 5--> 6
               |--> 5.1-->6.1

```
当前 6 是主链的最后一个节点, 这就是tip, tip的stakeNode在一开始Blockchain 初始化时会就从硬盘加载进来. 请见: [3. Blockchain 初始化](https://teakki.com/p/5c7a982fb1029f607606859a)
现在除了块6, 其它块的stakeNode是不存在的.
现在块6.1要尝试入块, 它的parent是5.1, 而5.1又没有stakeNode, 所以无法校验stake的数据(PoolSize, FinalState). 此时必须要将5.1的stakeNode计算出来.
计算5.1 stakeNode 的步骤和之后要讲的侧链转主链的逻辑是一样的.

1. undo 6, 5. 即 DisconnectNode 6, 5
2. redo 5.1. 即 ConnectNode 5.1

因为6有完整的stake数据, 加上两个数据库 ticketsinblock(新ticket表) 和 stakeblockundo(undo表), 所以undo 6 就可以得到5的stakeNode, undo 5就可以得到4. 得到4后, redo 5.1就可以得到5.1的stakeNode
checkBlockHeaderContext()有这样一段代码:
```
// Ensure the header commits to the correct pool size based on
// its position within the chain.
parentStakeNode, err := b.fetchStakeNode(prevNode)
if err != nil {
   return err
}

```
我们来看 fetchStakeNode(prevNode), 主要代码如下 : 
```
// Start by undoing the effects from the current tip back to, and including
// the fork point per the above description.
tip := b.bestChain.Tip()
fork := b.bestChain.FindFork(node)
err := b.db.View(func(dbTx database.Tx) error {
   for n := tip; n != nil &amp;&amp; n != fork; n = n.parent {
      // No need to load nodes that are already loaded.
      prev := n.parent
      if prev == nil || prev.stakeNode != nil {
         continue
      }

      // (1) undo
      // Generate the previous stake node by starting with the child stake
      // node and undoing the modifications caused by the stake details in
      // the previous block.
      stakeNode, err := n.stakeNode.DisconnectNode(prev.lotteryIV(), nil,
         nil, dbTx)
      if err != nil {
         return err
      }
      prev.stakeNode = stakeNode
   }

   return nil
})
if err != nil {
   return nil, err
}

// Nothing more to do if the requested node is the fork point itself.
if node == fork {
   return node.stakeNode, nil
}

// The requested node is on a side chain, so replay the effects of the
// blocks up to the requested node per the above description.
//
// Note that the blocks between the fork point and the requested node are
// added to the slice from back to front so that they are attached in the
// appropriate order when iterating the slice.
attachNodes := make([]blockNode, node.height-fork.height)
for n := node; n != nil &amp;&amp; n != fork; n = n.parent {
   attachNodes[n.height-fork.height-1] = n
}

// (2) redo
for _, n := range attachNodes {
   // No need to load nodes that are already loaded.
   if n.stakeNode != nil {
      continue
   }

   // Populate the prunable ticket information as needed.
   if err := b.maybeFetchTicketInfo(n); err != nil {
      return nil, err
   }

   // Generate the stake node by applying the stake details in the current
   // block to the previous stake node.
   stakeNode, err := n.parent.stakeNode.ConnectNode(n.lotteryIV(),
      n.ticketsVoted, n.ticketsRevoked, n.newTickets)
   if err != nil {
      return nil, err
   }
   n.stakeNode = stakeNode
}


```

## 总结
本节主要讲解了主链入块检查, 与前一节的一致性检查不同的时, 入块检查是在主链的上下文中. 主要检查了:

* tx是否过期或提前
* 难度检查(PoW和PoS)
* 时间检查
* coinbase输出是否合法
* stake相关检查

注意, 在前一节中也会有时间, 难度检查, 大家要知道它们的检查点是不同的.
前一节的难度检查只会检查Header的字段SBits, Bits与最小值判断, 不会检查这个值是否符合链上的值. 所以在本节中, 会重新计算难度, 然后判断是否相同.
前一节的时间检查是检查这值值是合法: 不能有纳秒, 不能太久. 本节的时间会先计算前11个块的中间值再进行比较.
所以本节的检查是链上检查, 有链的上下文.

# 入主链
b.connectBestChain() 之前会将块保存, 添加索引.
b.connectBestChain() 主要有两个逻辑:

* 如果它的父块是主链上, 入主链
* 如果不是, 加到侧链上, 并判断是否要侧链转主链

本节主要讲解入主链的逻辑.
在前两节都是为入链做一些检查, 但其实有一个最主要的没有检查:

* 交易里引用的输出是否合法
* 是否双花

这些都是utxo相关的检查.
chain.go
```
func (b BlockChain) connectBestChain(node blockNode, block, parent dcrutil.Block, flags BehaviorFlags) (int64, error) {
   fastAdd := flags&amp;BFFastAdd == BFFastAdd

   // Ensure the passed parent is actually the parent of the block.
   if parent.Hash() != node.parent.hash {
      panicf("parent block %v (height %v) does not match expected parent %v "+
         "(height %v)", parent.Hash(), parent.MsgBlock().Header.Height,
         node.parent.hash, node.height-1)
   }

   // We are extending the main (best) chain with a new block.  This is the
   // most common case.
   parentHash := &amp;block.MsgBlock().Header.PrevBlock
   tip := b.bestChain.Tip()
   // 如果它的父块是主链上, 入主链
   if parentHash == tip.hash {
      // Skip expensive checks if the block has already been fully
      // validated.
      isKnownValid := b.index.NodeStatus(node).KnownValid()
      fastAdd = fastAdd || isKnownValid

      // Perform several checks to verify the block can be connected
      // to the main chain without violating any rules and without
      // actually connecting the block.
      //
      // Also, set the applicable status result in the block index,
      // and flush the status changes to the database.  It is safe to
      // ignore any errors when flushing here as the changes will be
      // flushed when a valid block is connected, and the worst case
      // scenario if a block a invalid is it would need to be
      // revalidated after a restart.
      view := NewUtxoViewpoint()
      view.SetBestHash(parentHash)
      var stxos []spentTxOut
      if !fastAdd {
         err := b.checkConnectBlock(node, block, parent, view,
            &amp;stxos)
         if err != nil {
            if _, ok := err.(RuleError); ok {
               b.index.SetStatusFlags(node, statusValidateFailed)
               b.flushBlockIndexWarnOnly()
            }
            return 0, err
         }
      }
      if !isKnownValid {
         b.index.SetStatusFlags(node, statusValid)
         b.flushBlockIndexWarnOnly()
      }

      // In the fast add case the code to check the block connection
      // was skipped, so the utxo view needs to load the referenced
      // utxos, spend them, and add the new utxos being created by
      // this block.  Also, in the case the the block votes against
      // the parent, its regular transaction tree must be
      // disconnected.
      if fastAdd {
         err := view.connectBlock(b.db, block, parent, &amp;stxos)
         if err != nil {
            return 0, err
         }
      }

      // Connect the block to the main chain.
      err := b.connectBlock(node, block, parent, view, stxos)
      if err != nil {
         return 0, err
      }

      validateStr := "validating"
      if !voteBitsApproveParent(node.voteBits) {
         validateStr = "invalidating"
      }

      log.Debugf("Block %v (height %v) connected to the main chain, "+
         "%v the previous block", node.hash, node.height,
         validateStr)

      // The fork length is zero since the block is now the tip of the
      // best chain.
      return 0, nil
   }
   if fastAdd {
      log.Warnf("fastAdd set in the side chain case? %v&#92;n",
         block.Hash())
   }

   // We're extending (or creating) a side chain, but the cumulative
   // work for this new side chain is not enough to make it the new chain.
   if node.workSum.Cmp(tip.workSum) <= 0 {
      // Log information about how the block is forking the chain.
      fork := b.bestChain.FindFork(node)
      if fork.hash == parentHash {
         log.Infof("FORK: Block %v (height %v) forks the chain at height "+
            "%d/block %v, but does not cause a reorganize",
            node.hash, node.height, fork.height, fork.hash)
      } else {
         log.Infof("EXTEND FORK: Block %v (height %v) extends a side chain "+
            "which forks the chain at height %d/block %v", node.hash,
            node.height, fork.height, fork.hash)
      }

      forkLen := node.height - fork.height
      return forkLen, nil
   }

   // We're extending (or creating) a side chain and the cumulative work
   // for this new side chain is more than the old best chain, so this side
   // chain needs to become the main chain.  In order to accomplish that,
   // find the common ancestor of both sides of the fork, disconnect the
   // blocks that form the (now) old fork from the main chain, and attach
   // the blocks that form the new chain to the main chain starting at the
   // common ancestor (the point where the chain forked).
   //
   // Reorganize the chain and flush any potential unsaved changes to the
   // block index to the database.  It is safe to ignore any flushing
   // errors here as the only time the index will be modified is if the
   // block failed to connect.
   log.Infof("REORGANIZE: Block %v is causing a reorganize.", node.hash)
   err := b.reorganizeChain(node)
   b.flushBlockIndexWarnOnly()
   if err != nil {
      return 0, err
   }

   // The fork length is zero since the block is now the tip of the best
   // chain.
   return 0, nil
}

```
通过代码可以得出, 入主链要经过两步:

1. checkConnectBlock()
2. connectBlock()

接下来一一讲解.
## checkConnectBlock() 
checkConnectBlock()是最核心的代码, 它的主要功能就是Utxo数据库的操作, 判断交易是否合法, 操作utxto数据库, 删除引用的输出, 将新的输出加到utxo中.
并且, 为了以后作undo方便, 还将该block所有引用的输出写到stxos中.
checkConnectBlock()前先创建NewUtxoViewpoint{}和stxos(已花费交易列表)
```
view := NewUtxoViewpoint()
view.SetBestHash(parentHash)
var stxos []spentTxOut
b.checkConnectBlock(node, block, parent, view,
      &amp;stxos)

```
我们看checkConnectBlock()代码:
```
func (b BlockChain) checkConnectBlock(node blockNode, block, parent dcrutil.Block, view UtxoViewpoint, stxos []spentTxOut) error {
   // If the side chain blocks end up in the database, a call to
   // CheckBlockSanity should be done here in case a previous version
   // allowed a block that is no longer valid.  However, since the
   // implementation only currently uses memory for the side chain blocks,
   // it isn't currently necessary.

   // (1) 再一次判断是在主链上
   // Ensure the view is for the node being checked.
   parentHash := &amp;block.MsgBlock().Header.PrevBlock
   if !view.BestHash().IsEqual(parentHash) {
      return AssertError(fmt.Sprintf("inconsistent view when "+
         "checking block connection: best hash is %v instead "+
         "of expected %v", view.BestHash(), parentHash))
   }

   // coinbase交易是否有团队的输出
   // Check that the coinbase pays the tax, if applicable.
   err := CoinbasePaysTax(b.subsidyCache, block.Transactions()[0], node.height,
      node.voters, b.chainParams)
   if err != nil {
      return err
   }

   // Don't run scripts if this node is before the latest known good
   // checkpoint since the validity is verified via the checkpoints (all
   // transactions are included in the merkle root hash and any changes
   // will therefore be detected by the next checkpoint).  This is a huge
   // optimization because running the scripts is the most time consuming
   // portion of block handling.
   checkpoint := b.latestCheckpoint()
   runScripts := !b.noVerify
   if checkpoint != nil &amp;&amp; node.height <= checkpoint.Height {
      runScripts = false
   }
   var scriptFlags txscript.ScriptFlags
   if runScripts {
      var err error
      scriptFlags, err = b.consensusScriptVerifyFlags(node)
      if err != nil {
         return err
      }
   }

   // Create a view which preserves the expected consensus semantics for
   // relative lock times via sequence numbers once the stake vote for the
   // agenda is active.
   legacySeqLockView := view
   lnFeaturesActive, err := b.isLNFeaturesAgendaActive(node.parent)
   if err != nil {
      return err
   }
   fixSeqLocksActive, err := b.isFixSeqLocksAgendaActive(node.parent)
   if err != nil {
      return err
   }
   if lnFeaturesActive &amp;&amp; !fixSeqLocksActive {
      var err error
      legacySeqLockView, err = b.createLegacySeqLockView(block, parent,
         view)
      if err != nil {
         return err
      }
   }

   // 如果本块对上一票投票结果是非法, 则要将父块移除出去
   // 注意, 这里只是将普通交易还原
   // Disconnect all of the transactions in the regular transaction tree of
   // the parent if the block being checked votes against it.
   if node.height > 1 &amp;&amp; !voteBitsApproveParent(node.voteBits) {
      err := view.disconnectDisapprovedBlock(b.db, parent)
      if err != nil {
         return err
      }
   }

   // 判断Stake交易是否合法
   // Ensure the stake transaction tree does not contain any transactions
   // that 'overwrite' older transactions which are not fully spent.
   err = b.checkDupTxs(block.STransactions(), view)
   if err != nil {
      log.Tracef("checkDupTxs failed for cur TxTreeStake: %v", err)
      return err
   }

   // Load all of the utxos referenced by the inputs for all transactions
   // in the block don't already exist in the utxo view from the database.
   //
   // These utxo entries are needed for verification of things such as
   // transaction inputs, counting pay-to-script-hashes, and scripts.
   err = view.fetchInputUtxos(b.db, block)
   if err != nil {
      return err
   }
   // 处理stake交易, stake交易上链
   err = b.checkTransactionsAndConnect(b.subsidyCache, 0, node,
      block.STransactions(), view, stxos, false)
   if err != nil {
      log.Tracef("checkTransactionsAndConnect failed for "+
         "TxTreeStake: %v", err)
      return err
   }

   stakeTreeFees, err := getStakeTreeFees(b.subsidyCache, node.height,
      b.chainParams, block.STransactions(), view)
   if err != nil {
      log.Tracef("getStakeTreeFees failed for TxTreeStake: %v", err)
      return err
   }

   // Enforce all relative lock times via sequence numbers for the stake
   // transaction tree once the stake vote for the agenda is active.
   var prevMedianTime time.Time
   if lnFeaturesActive {
      // Use the past median time of the previous block in order
      // to determine if the transactions in the current block are
      // final.
      prevMedianTime = node.parent.CalcPastMedianTime()

      for _, stx := range block.STransactions() {
         sequenceLock, err := b.calcSequenceLock(node, stx,
            view, true)
         if err != nil {
            return err
         }
         if !SequenceLockActive(sequenceLock, node.height,
            prevMedianTime) {

            str := fmt.Sprintf("block contains " +
               "stake transaction whose input " +
               "sequence locks are not met")
            return ruleError(ErrUnfinalizedTx, str)
         }
      }
   }

   if runScripts {
      err = checkBlockScripts(block, view, false, scriptFlags,
         b.sigCache)
      if err != nil {
         log.Tracef("checkBlockScripts failed; error returned "+
            "on txtreestake of cur block: %v", err)
         return err
      }
   }

   // 普通交易验证
   // Ensure the regular transaction tree does not contain any transactions
   // that 'overwrite' older transactions which are not fully spent.
   err = b.checkDupTxs(block.Transactions(), view)
   if err != nil {
      log.Tracef("checkDupTxs failed for cur TxTreeRegular: %v", err)
      return err
   }
   // 普通交易上链
   err = b.checkTransactionsAndConnect(b.subsidyCache, stakeTreeFees, node,
      block.Transactions(), view, stxos, true)
   if err != nil {
      log.Tracef("checkTransactionsAndConnect failed for cur "+
         "TxTreeRegular: %v", err)
      return err
   }

   // Enforce all relative lock times via sequence numbers for the regular
   // transaction tree once the stake vote for the agenda is active.
   if lnFeaturesActive {
      // Skip the coinbase since it does not have any inputs and thus
      // lock times do not apply.
      for _, tx := range block.Transactions()[1:] {
         sequenceLock, err := b.calcSequenceLock(node, tx,
            legacySeqLockView, true)
         if err != nil {
            return err
         }
         if !SequenceLockActive(sequenceLock, node.height,
            prevMedianTime) {

            str := fmt.Sprintf("block contains " +
               "transaction whose input sequence " +
               "locks are not met")
            return ruleError(ErrUnfinalizedTx, str)
         }
      }
   }

   if runScripts {
      err = checkBlockScripts(block, view, true, scriptFlags,
         b.sigCache)
      if err != nil {
         log.Tracef("checkBlockScripts failed; error returned "+
            "on txtreeregular of cur block: %v", err)
         return err
      }
   }

   // First block has special rules concerning the ledger.
   if node.height == 1 {
      err := BlockOneCoinbasePaysTokens(block.Transactions()[0],
         b.chainParams)
      if err != nil {
         return err
      }
   }

   // Update the best hash for view to include this block since all of its
   // transactions have been connected.
   view.SetBestHash(&amp;node.hash)

   return nil
}

```
主要逻辑如下:

* 检查coinbase交易是否有合法的团队奖励
* 如果本块对上一票投票结果是非法, 则要将父块移除出去
* 检查stake交易
* stake交易上链
* 检查普通交易
* 普通交易上链



### 检查coinbase交易是否有合法的团队奖励
团队的奖励是所有奖励的 10% (60% PoW, 30% PoS, 10% 团队).
![image.png](https://upload-images.jianshu.io/upload_images/422094-37e9a12dc6451c37.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

coinbase至少3个output, 第一个output是给团队的奖励(地址 Dcur2mcGjmENx4DhNqDctW5wJCVyT3Qeqkx), 第2个OP_RETURN包含当前height, 如果不一致, 则不合法,  第3个output是给矿工自己的.
这里只判断给团队的奖励是否合法, 并没有判断给矿工的奖励是否合法.
对矿工的奖励验证在之后的普通交易检查中.
### UTXO操作
接下来的移除父块, 对stake和普通交易的上链主要就是对utxo数据的操作, 我们再来复习下, uxto是怎么表示的:
所有的utxoentry是存在名为 “utxoset” 的Bucket中, 其中的Key为交易的tx Hash, Value为utxoentry的序列化结果
{txHash =&gt; utxoentry} (有字段[]utxo)
可以理解为 {txHash =&gt; map[uint32]utxoOutput}, uint32 是 outputIndex
即每一条记录是该tx中所有未花费的记录.
```
type UtxoEntry struct {
   sparseOutputs map[uint32]utxoOutput // Sparse map of unspent outputs.
   stakeExtra    []byte                 // Extra data for the staking system.

   txType    stake.TxType // The stake type of the transaction.
   height    uint32       // Height of block containing tx.
   index     uint32       // Index of containing tx in block.
   txVersion uint16       // The tx version of this tx.

   isCoinBase bool // Whether entry is a coinbase tx.
   hasExpiry  bool // Whether entry has an expiry.
   modified   bool // Entry changed since load.
}
type utxoOutput struct {
   pkScript      []byte // The public key script for the output.
   amount        int64  // The amount of the output.
   scriptVersion uint16 // The script version
   compressed    bool   // The public key script is compressed.
   spent         bool   // Output is spent.
}

```
要对块进行undo, 我们需要知道该块消费了什么, SpendJournal 数据库保存了该块所有引用的输出, 即消费了哪些输出.
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
tips: 其实SpendJournal只是一个优化数据库, 完全可以启动块的交易获得所引用的输出.
#### 如果本块对上一票投票结果是非法, 则要将父块普通交易移除出去
假设有以下链
```
1--2-->3-->4

```
utxo数据库中表示了这4个块所有未花费的输出, 现在要上块5, 如果块5对块4验证失败, 则必须先将块4先移除, 即:

* 对块4引用的输出要还原
* 对块4新增的输出要删除

disconnectDisapprovedBlock(), 步骤如下:

* 加载该块的[]spentTxOut stxos 
* 加载该块引用的所有交易的utxo
* 对普通交易进行undo

```
func (view UtxoViewpoint) disconnectDisapprovedBlock(db database.DB, block dcrutil.Block) error {
   // Load all of the spent txos for the block from the database spend journal.
   var stxos []spentTxOut
   err := db.View(func(dbTx database.Tx) error {
      var err error
      stxos, err = dbFetchSpendJournalEntry(dbTx, block)
      return err
   })
   if err != nil {
      return err
   }

   // Load all of the utxos referenced by the inputs for all transactions in
   // the block that don't already exist in the utxo view from the database.
   err = view.fetchRegularInputUtxos(db, block)
   if err != nil {
      return err
   }

   // Sanity check the correct number of stxos are provided.
   if len(stxos) != countSpentOutputs(block) {
      panicf("provided %v stxos for block %v (height %v) which spends %v "+
         "outputs", len(stxos), block.Hash(), block.MsgBlock().Header.Height,
         countSpentOutputs(block))
   }

   return view.disconnectRegularTransactions(block, stxos)
}

```
disconnectTransactions():

* 交易从后往前, 删除自己的输出(将tx记录从utxo中删除)
* 交易从后往前, 还原引用的输出(重新加到utxo中)

为才能要从后往前呢?
因为在同一块中, 后面的交易可能引用了前面的交易的输出, 如下图:
![image.png](https://upload-images.jianshu.io/upload_images/422094-55e2d409dbcff55a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在我们将block2删除, 那么我们希望删除后的utxo中要有 out1, out4, 不能有out5, out6, out7.
在block2中, in3引用了out5.
将block2 disconnected(), 如果从前往后操作, :

1. block2第1个交易的out5从utxo删除, in1所引用的out1还原
2. block2第2个交易的out6从utxo删除, in2所引用的out4还原
3. block2第3个交易的out7从utxo删除, in3所引用的out5还原, 结果将out5还原了

如果从后往前操作:

1. block2第3个交易的out7从utxo删除, in3所引用的out5还原
2. block2第2个交易的out6从utxo删除, in2所引用的out4还原
3. block2第1个交易的out5从utxo删除, in1所引用的out1还原. 虽然在第1步中将out5还原了, 但这一步还是将其删除了.

```
func (view UtxoViewpoint) disconnectTransactions(block dcrutil.Block, stxos []spentTxOut, stakeTree bool) error {
   // Choose which transaction tree to use and the appropriate offset into the
   // spent transaction outputs that corresponds to them depending on the flag.
   // Transactions in the stake tree are spent before transactions in the
   // regular tree, thus skip all of the outputs spent by the regular tree when
   // disconnecting stake transactions.
   stxoIdx := len(stxos) - 1
   transactions := block.Transactions()
   if stakeTree {
      stxoIdx = len(stxos) - countSpentRegularOutputs(block) - 1
      transactions = block.STransactions()
   }
   // 从后往前
   for txIdx := len(transactions) - 1; txIdx > -1; txIdx-- {
      tx := transactions[txIdx]
      msgTx := tx.MsgTx()
      txType := stake.TxTypeRegular
      if stakeTree {
         txType = stake.DetermineTxType(msgTx)
      }
      isVote := txType == stake.TxTypeSSGen

      // 删除自己创建的输出
      // Clear this transaction from the view if it already exists or create a
      // new empty entry for when it does not.  This is done because the code
      // relies on its existence in the view in order to signal modifications
      // have happened.
      isCoinbase := !stakeTree &amp;&amp; txIdx == 0
      entry := view.entries[tx.Hash()]
      if entry == nil {
         entry = newUtxoEntry(msgTx.Version, uint32(block.Height()),
            uint32(txIdx), isCoinbase, msgTx.Expiry != 0, txType)
         view.entries[tx.Hash()] = entry
      }
      entry.modified = true
      entry.sparseOutputs = make(map[uint32]utxoOutput)

      // Loop backwards through all of the transaction inputs (except for the
      // coinbase which has no inputs) and unspend the referenced txos.  This
      // is necessary to match the order of the spent txout entries.
      if isCoinbase {
         continue
      }
      // 从后往前, 还原自己引用的输出
      for txInIdx := len(msgTx.TxIn) - 1; txInIdx > -1; txInIdx-- {
         // Ignore stakebase since it has no input.
         if isVote &amp;&amp; txInIdx == 0 {
            continue
         }

         // Ensure the spent txout index is decremented to stay in sync with
         // the transaction input.
         stxo := &amp;stxos[stxoIdx]
         stxoIdx--

         // When there is not already an entry for the referenced transaction
         // in the view, it means it was fully spent, so create a new utxo
         // entry in order to resurrect it.
         txIn := msgTx.TxIn[txInIdx]
         originHash := &amp;txIn.PreviousOutPoint.Hash
         originIndex := txIn.PreviousOutPoint.Index
         entry := view.entries[originHash]
         if entry == nil {
            if !stxo.txFullySpent {
               return AssertError(fmt.Sprintf("tried to revive unspent "+
                  "tx %v from non-fully spent stx entry", originHash))
            }
            entry = newUtxoEntry(msgTx.Version, stxo.height, stxo.index,
               stxo.isCoinBase, stxo.hasExpiry, stxo.txType)
            if stxo.txType == stake.TxTypeSStx {
               entry.stakeExtra = stxo.stakeExtra
            }
            view.entries[originHash] = entry
         }

         // Mark the entry as modified since it is either new or will be
         // changed below.
         entry.modified = true

         // Restore the specific utxo using the stxo data from the spend
         // journal if it doesn't already exist in the view.
         output, ok := entry.sparseOutputs[originIndex]
         if !ok {
            // Add the unspent transaction output.
            entry.sparseOutputs[originIndex] = &amp;utxoOutput{
               compressed:    stxo.compressed,
               spent:         false,
               amount:        txIn.ValueIn,
               scriptVersion: stxo.scriptVersion,
               pkScript:      stxo.pkScript,
            }
            continue
         }

         // Mark the existing referenced transaction output as unspent.
         output.spent = false
      }
   }

   return nil
}

```
注意, 这里只是对普通交易undo, 这一步我们没并没有将块的stake交易还原, 其实并不是完全将块移除, 为什么? 因为, 虽然子块对父块验证结果是非法, 但其实只是对父块的普通交易进行验证, 而其stake交易是不验证的.
什么时候也会将stake交易还原, 在之后的侧链转主链过程会提到, 将整块删除.
问题, 既然stake交易不会还原, 矿工会不会对stake交易作恶呢?
#### stake交易/普通交易检查并上链
通过查看代码, stake交易检查并上链要经过以下3个阶段
```
err = b.checkDupTxs(block.STransactions(), view)
err = view.fetchInputUtxos(b.db, block)
b.checkTransactionsAndConnect(b.subsidyCache, 0, node,
   block.STransactions(), view, stxos, false)

```
checkDupTxs()检查是否创建了和之前相同的交易, 即相同的tx之前也创建了, 而且之前这个交易还有没花的输出, 现在创建了和之前一模一样的交易. 这个代码应该是人BTCD复制过来的, 在BTC中这种情况经常发生, 因为coinbase的交易很可能一样, 比如这几个 coinbase交易:  [coinbase d5d279…](https://blockexplorer.com/tx/d5d27987d2a3dfc724e359870c6644b40e497bdc0589a033220fe15429d88599) [block af0aed…](https://blockexplorer.com/block/00000000000af0aed4792b1acee3d966af36cf5def14935db8de83d6f9306f2f) [block a4d0a3…](https://blockexplorer.com/block/00000000000a4d0a398161ffc163c503763b1f4360639393e0e4c8e300e0caec). , 但在DCR中,  coinbase交易的输出是包含了height, 所以不可能有相同的交易. 所以这个代码其实是没用的. 可能的原因这里有这个检查是因为不同的交易可能也会有相同的hash(概率上几乎为0).
fetchInputUtxos() 将普通交易和stake交易引用的utxo从数据库加载进来, 即将引用的utxo加载进来, 为之后判断是否双花.
我们着重讲checkTransactionsAndConnect()
#### checkTransactionsAndConnect()
该方法主要包含两步:

1. 对每个交易调用CheckTransactionInputs()检查每一个交易
2. 对每个交易调用connectTransaction(), 操作utxoview (即删除它引用, 添加它的输出到utxo中)
3. 检查coinbase交易的奖励是否合法, 检查中票交易的奖励是否合法

validate.go CheckTransactionInputs() 检查每一个交易:

* 如果是买票SStx交易, 则要根据买票交易的规则检查, 如 第一个输出必须是票价, 输出的数量必须是 2inputNum + 1
* 如果是投票SSGen交易, 则要检查投票奖励分配是否合法, 引用的交易是否是买票的第一个输出等
* 如果是退票交易, 则要检查引用的交易是否是买票的第一个输出, 票是否已成熟, 输出和买票的输出一致等
* 对于所有交易的引用判断:

    * 是否双花: 引用的utxo是否存在, 是否它的输出为0, 是否已过期, 它花的值是否和引用的输出的值一致
    * 如果引用的是coinbase输出, 则要经过256块
    * 如果是tx是statke交易, 它的引用检查有一些特定的规则, 如:

        * OP_SSTX(买票)的输出, 只能被中票和退票交易花费
        * OP_SSGEN和OP_SSRTX(中票和退票)的输出, 只能等成熟后(过了256块)才能被花费
        * 买票的sstxchange的输出, 只能等成熟后(过了256块)才能被花费


* 最后检查输入是否&gt;=输出, 并返回交易费

utxoviewpoint.go connectTransaction() 操作utxoview (即删除它引用的utxo, 添加它的输出到utxoview中), 代码比较简单.
与 connectTransaction()相反的操作就是 disconnectTransactions(), 即通过stxos来undo, 这会在侧链转主链将块先从主链移除时用到.
在checkTransactionsAndConnect()的最后一步就是检查coinbase交易的奖励是否合法, 检查中票交易的奖励是否合法. 具体逻辑会单独在之后的章节中提到.
stake交易的规则很多, 具体之后会单独出一个文章讲述stake交易的规则.
## connectBlock() 入库
在之前的checkConnectBlock()代码检查了一大堆产生的结果就是操作了utxto, 生成了stxos, 所以数据都已理好, 现在就该入库了.
主链入库要存哪些? 这里仅是主链入库, 所以操作的都是主链相关的:

1. block在此之前就已经入库了, 但blockNode没有入库, 所以它要入库
2. 因为加了一块, 所以block state会变化, 它也要更新
3. utxo数据库有修改, 它也要更新
4. 新的块的stxos, 要加上
5. stake数据库要更新

    1. liveTickets
    2. missedTickets
    3. revokeTickes
    4. undoData
    5. newTickets
    6. 还有stake的状态

1. 如果有一些优化的索引indexManager会调用它们进行connectBlock()

以上就是connectBlock()要操作的数据库, 大家可以看看代码, 很简单的逻辑.
## 总结
之前的一大堆检查, 在这章中终于要主链入库了, BTC/DCR数据库中最重要的就是utxo数据库, 所以本章的重点就操作它.
connectBestChain()最后一步就是侧链转主链.
下一节会讲侧链转主链, 如果新的块加到了侧链, 什么时候它会转成主链呢?

# 侧链转主链
如果一个块的父块是侧链, 它不会有一章入主链的逻辑.
如果我们加的块是侧链, 那么加进去后计算下workSum, 如果难度比之前的主链大,  那么侧链就要转为主链. 关于workSum的计算请至 [工作量证明](https://teakki.com/p/5c7f7022b1029f607607feae)
connectBestChain() 后部分代码:
```
// We're extending (or creating) a side chain, but the cumulative
// work for this new side chain is not enough to make it the new chain.
// 难度比主链小, 则不要转主链, 返回侧链的长度
if node.workSum.Cmp(tip.workSum) <= 0 {
   // Log information about how the block is forking the chain.
   fork := b.bestChain.FindFork(node)
   if fork.hash == parentHash {
      log.Infof("FORK: Block %v (height %v) forks the chain at height "+
         "%d/block %v, but does not cause a reorganize",
         node.hash, node.height, fork.height, fork.hash)
   } else {
      log.Infof("EXTEND FORK: Block %v (height %v) extends a side chain "+
         "which forks the chain at height %d/block %v", node.hash,
         node.height, fork.height, fork.hash)
   }

   forkLen := node.height - fork.height
   return forkLen, nil
}

// We're extending (or creating) a side chain and the cumulative work
// for this new side chain is more than the old best chain, so this side
// chain needs to become the main chain.  In order to accomplish that,
// find the common ancestor of both sides of the fork, disconnect the
// blocks that form the (now) old fork from the main chain, and attach
// the blocks that form the new chain to the main chain starting at the
// common ancestor (the point where the chain forked).
//
// Reorganize the chain and flush any potential unsaved changes to the
// block index to the database.  It is safe to ignore any flushing
// errors here as the only time the index will be modified is if the
// block failed to connect.
log.Infof("REORGANIZE: Block %v is causing a reorganize.", node.hash)
err := b.reorganizeChain(node)
b.flushBlockIndexWarnOnly()
if err != nil {
   return 0, err
}

```
b.bestChain.FindFork(node) 就是找到侧链和主链共同的结点, 如下图:
![image.png](https://upload-images.jianshu.io/upload_images/422094-b0457a3178659413.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在侧链结点6加上后, 侧链难度&gt;主链, 它们共同的祖先结点就是2. 怎么找到2呢? 这是一个简单的数据结构算法:

1. 两条链两个指针, 先让两个指针指向相同高度的结点, 如上, 一个指向5, 一个指向5.1
2. 然后两个指针同时往前遍历, 每定位一个就判断指向的结点是否相等

DCR实现的代码如下:
```
func (c chainView) findFork(node blockNode) blockNode {
   // No fork point for node that doesn't exist.
   if node == nil {
      return nil
   }

   // When the height of the passed node is higher than the height of the
   // tip of the current chain view, walk backwards through the nodes of
   // the other chain until the heights match (or there or no more nodes in
   // which case there is no common node between the two).
   //
   // NOTE: This isn't strictly necessary as the following section will
   // find the node as well, however, it is more efficient to avoid the
   // contains check since it is already known that the common node can't
   // possibly be past the end of the current chain view.  It also allows
   // this code to take advantage of any potential future optimizations to
   // the Ancestor function such as using an O(log n) skip list.
   chainHeight := c.height()
   if node.height > chainHeight {
      node = node.Ancestor(chainHeight)
   }

   // Walk the other chain backwards as long as the current one does not
   // contain the node or there are no more nodes in which case there is no
   // common node between the two.
   for node != nil &amp;&amp; !c.contains(node) {
      node = node.parent
   }

   return node
}


```

## reorganizeChain() 侧链转主链
如果侧链难度比主链难度大, 就要侧链转主链了.
我们仍然看这个图, 怎么转主链呢?
![image.png](https://upload-images.jianshu.io/upload_images/422094-f16b34aa19e74523.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

分两步:

1. 删除主链: 将5, 4, 3 依次删除 (从后往前)
2. 侧链入主链: 将3.1, 4.1, 5.1, 6 依次入主链 (从前往后)

reorganizeChain() 核心代码如下:
```
// 1. 尝试转主链
reorgErr := b.reorganizeChainInternal(targetTip)
// 2. 如果转失败了, 再还原
if reorgErr != nil {
   if err := b.reorganizeChainInternal(origTip); err != nil {
      panicf("failed to reorganize back to known good chain tip %s "+
         "(height %d): %v -- probable database corruption", origTip.hash,
         origTip.height, err)
   }

   return reorgErr
}

```
我们来看reorganizeChainInternal()代码:
```
func (b BlockChain) reorganizeChainInternal(targetTip blockNode) error {
   // Find the fork point adding each block to a slice of blocks to attach
   // below once the current best chain has been disconnected.  They are added
   // to the slice from back to front so that so they are attached in the
   // appropriate order when iterating the slice later.
   //
   // In the case a known invalid block is detected while constructing this
   // list, mark all of its descendants as having an invalid ancestor and
   // prevent the reorganize.
   // 1. 找到共同结点(2)
   fork := b.bestChain.FindFork(targetTip)
   // 2. 侧链中需要添加的结点, 即(3.1, 4.1, 5.1, 6 )
   attachNodes := make([]blockNode, targetTip.height-fork.height)
   for n := targetTip; n != nil &amp;&amp; n != fork; n = n.parent {
      if b.index.NodeStatus(n).KnownInvalid() {
         for _, dn := range attachNodes[n.height-fork.height:] {
            b.index.SetStatusFlags(dn, statusInvalidAncestor)
         }

         str := fmt.Sprintf("block %s is known to be invalid or a "+
            "descendant of an invalid block", n.hash)
         return ruleError(ErrKnownInvalidBlock, str)
      }

      attachNodes[n.height-fork.height-1] = n
   }

   // 3. 删除主链: 将5, 4, 3 依次删除 (从后往前)
   // Disconnect all of the blocks back to the point of the fork.  This entails
   // loading the blocks and their associated spent txos from the database and
   // using that information to unspend all of the spent txos and remove the
   // utxos created by the blocks.  In addition, if a block votes against its
   // parent, the regular transactions are reconnected.
   tip := b.bestChain.Tip()
   view := NewUtxoViewpoint()
   view.SetBestHash(&amp;tip.hash)
   var nextBlockToDetach dcrutil.Block
   for tip != nil &amp;&amp; tip != fork {
      // Grab the block to detach based on the node.  Use the fact that the
      // blocks are being detached in reverse order, so the parent of the
      // current block being detached is the next one being detached.
      n := tip
      block := nextBlockToDetach
      if block == nil {
         var err error
         block, err = b.fetchMainChainBlockByNode(n)
         if err != nil {
            return err
         }
      }
      if n.hash != block.Hash() {
         panicf("detach block node hash %v (height %v) does not match "+
            "previous parent block hash %v", &amp;n.hash, n.height,
            block.Hash())
      }

      // Grab the parent of the current block and also save a reference to it
      // as the next block to detach so it doesn't need to be loaded again on
      // the next iteration.
      parent, err := b.fetchMainChainBlockByNode(n.parent)
      if err != nil {
         return err
      }
      nextBlockToDetach = parent

      // Load all of the spent txos for the block from the spend journal.
      var stxos []spentTxOut
      err = b.db.View(func(dbTx database.Tx) error {
         stxos, err = dbFetchSpendJournalEntry(dbTx, block)
         return err
      })
      if err != nil {
         return err
      }

      // 3.1 utxo undo
      // Update the view to unspend all of the spent txos and remove the utxos
      // created by the block.  Also, if the block votes against its parent,
      // reconnect all of the regular transactions.
      err = view.disconnectBlock(b.db, block, parent, stxos)
      if err != nil {
         return err
      }

      // 3.2 每删除一个节点就更新一次数据库
      // Update the database and chain state.
      err = b.disconnectBlock(n, block, parent, view)
      if err != nil {
         return err
      }

      tip = n.parent
   }

   // 现在 tip == 结点2了, forkBlock == 2
   // 4, 侧链入主链: 将3.1, 4.1, 5.1, 6 依次入主链 (从前往后)
   // Load the fork block if there are blocks to attach and its not already
   // loaded which will be the case if no nodes were detached.  The fork block
   // is used as the parent to the first node to be attached below.
   forkBlock := nextBlockToDetach
   if len(attachNodes) > 0 &amp;&amp; forkBlock == nil {
      var err error
      forkBlock, err = b.fetchMainChainBlockByNode(tip)
      if err != nil {
         return err
      }
   }

   // Attempt to connect each block that needs to be attached to the main
   // chain.  This entails performing several checks to verify each block can
   // be connected without violating any consensus rules and updating the
   // relevant information related to the current chain state.
   var prevBlockAttached dcrutil.Block
   for i, n := range attachNodes {
      // Grab the block to attach based on the node.  Use the fact that the
      // parent of the block is either the fork point for the first node being
      // attached or the previous one that was attached for subsequent blocks
      // to optimize.
      block, err := b.fetchBlockByNode(n)
      if err != nil {
         return err
      }
      parent := forkBlock
      if i > 0 {
         parent = prevBlockAttached
      }
      if n.parent.hash != parent.Hash() {
         panicf("attach block node hash %v (height %v) parent hash %v does "+
            "not match previous parent block hash %v", &amp;n.hash, n.height,
            &amp;n.parent.hash, parent.Hash())
      }

      // Store the loaded block as parent of next iteration.
      prevBlockAttached = block

      // 和入主链的逻辑一样, utxo操作 view.connectBlock() , 然后再b.connectBlock()入库
      // Skip validation if the block is already known to be valid.  However,
      // the utxo view still needs to be updated and the stxos are still
      // needed.
      stxos := make([]spentTxOut, 0, countSpentOutputs(block))
      if b.index.NodeStatus(n).KnownValid() {
         // Update the view to mark all utxos referenced by the block as
         // spent and add all transactions being created by this block to it.
         // In the case the block votes against the parent, also disconnect
         // all of the regular transactions in the parent block.  Finally,
         // provide an stxo slice so the spent txout details are generated.
         err := view.connectBlock(b.db, block, parent, &amp;stxos)
         if err != nil {
            return err
         }
      } else {
         // In the case the block is determined to be invalid due to a rule
         // violation, mark it as invalid and mark all of its descendants as
         // having an invalid ancestor.
         err = b.checkConnectBlock(n, block, parent, view, &amp;stxos)
         if err != nil {
            if _, ok := err.(RuleError); ok {
               b.index.SetStatusFlags(n, statusValidateFailed)
               for _, dn := range attachNodes[i+1:] {
                  b.index.SetStatusFlags(dn, statusInvalidAncestor)
               }
            }
            return err
         }
         b.index.SetStatusFlags(n, statusValid)
      }

      // Update the database and chain state.
      err = b.connectBlock(n, block, parent, view, stxos)
      if err != nil {
         return err
      }

      tip = n
   }

   return nil
}

```
我们从后往前删块, 在DCR中, 子块对父块进行验证, 如果我们将子块删除且子块对父块验证非法, 现在删除了子块, 证明这个验证无效, 所以我们需要connect父块的transactions, 这个逻辑我们可以在view.disconnectBlock()看到.
## 总结
纵观DCR代码, 因为块会添加也会删除, 所以有入链的代码, 必然也有出链的代码, 它们是一一对应的, 反映到代码上就是:
utxoviewpoint的utxo的操作: (uxtoviewpoint.go)

* connectBlock()
* disconnectBlock()

BlockChain的入库操作: (chain.go)

* connectBlock()
* disconnectBlock()

其它的索引数据库 indexers/下的也有

* connectBlock()
* disconnectBlock()

stake相关的: (stakenode.go)
这些操作都由fetchStakeNode()调用, fetchStakeNode()由b.connectBlock()和b.disconnectBlock()调用

* connectBlock()
* disconnectBlock()

stake/tickets.go

* WriteConnectedBestNode() (b.connectBlock() 调用)
* WriteDisconnectedBestNode() (b.disconnectBlock() 调用)

注意: stakenode结点的数据包含

* liveTickets
* missedTickets
* revokedTickets
* databaseUndoUpdate
* databaseBlockTickets

每个stakenode的数据都是计算出来的, stake数据库只会保存最新的stake信息, 其它的节点的stakenode都是通过undo, redo来构造.
只要这个stakenode存在, 这些数据就会存在, fetchStakeNode()就是为了构造stakenode的数据, 它里面有undo, redo的逻辑, 这些都在 [入链检查](https://teakki.com/p/5c7a98b7b1029f60760685c7) 中详细讲解过.


