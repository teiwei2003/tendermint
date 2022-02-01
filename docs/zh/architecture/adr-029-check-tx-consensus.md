# ADR 029:在投票前检查区块交易

## 变更日志

2018 年 4 月 10 日:更新并提供问题链接
[#2384](https://github.com/tendermint/tendermint/issues/2384) 和拒绝原因
19-09-2018:初稿

## 语境

我们目前通过 2 种方式检查 tx 的有效性.

1.通过mempool连接中的checkTx.
2. 通过deliverTx 共识连接.

第一个是在外部 tx 进来时调用，所以这次节点应该是一个提议者.当外部块进入并到达提交阶段时调用第二个，节点不需要是块的提议者，但是它应该检查该块中的 txs.

第二种情况，如果区块中有很多无效的交易，那么所有节点发现区块中的大部分交易都是无效的就为时已晚，我们最好也不要在区块链中记录无效的交易.

## 建议的解决方案

因此，我们应该在发出预投票之前找到一种方法来检查 txs 的有效性.目前我们有 cs.isProposalComplete() 来判断一个区块是否完整.我们可以有

```
func (blockExec *BlockExecutor) CheckBlock(block *types.Block) error {
   // check txs of block.
   for _, tx := range block.Txs {
      reqRes := blockExec.proxyApp.CheckTxAsync(tx)
      reqRes.Wait()
      if reqRes.Response == nil || reqRes.Response.GetCheckTx() == nil || reqRes.Response.GetCheckTx().Code != abci.CodeTypeOK {
         return errors.Errorf("tx %v check failed. response: %v", tx, reqRes.Response)
      }
   }
   return nil
}
```

BlockExecutor 中的这种方法来检查该块中所有交易的有效性.

但是，这种方法不应该这样实现，因为 checkTx 将共享应用程序内存池中使用的相同状态. 所以我们应该在Application中定义一个新的接口方法checkBlock来指示它使用与deliverTx相同的状态.

```
type Application interface {
   // Info/Query Connection
   Info(RequestInfo) ResponseInfo                // Return application info
   Query(RequestQuery) ResponseQuery             // Query for state

   // Mempool Connection
   CheckTx(tx []byte) ResponseCheckTx // Validate a tx for the mempool

   // Consensus Connection
   InitChain(RequestInitChain) ResponseInitChain // Initialize blockchain with validators and other info from TendermintCore
   CheckBlock(RequestCheckBlock) ResponseCheckBlock
   BeginBlock(RequestBeginBlock) ResponseBeginBlock // Signals the beginning of a block
   DeliverTx(tx []byte) ResponseDeliverTx           // Deliver a tx for full processing
   EndBlock(RequestEndBlock) ResponseEndBlock       // Signals the end of a block, returns changes to the validator set
   Commit() ResponseCommit                          // Commit the state and return the application Merkle root hash
}
```

所有应用程序都应该实现该方法. 例如，计数器:

```
func (app *CounterApplication) CheckBlock(block types.Request_CheckBlock) types.ResponseCheckBlock {
   if app.serial {
   	  app.originalTxCount = app.txCount   //backup the txCount state
      for _, tx := range block.CheckBlock.Block.Txs {
         if len(tx) > 8 {
            return types.ResponseCheckBlock{
               Code: code.CodeTypeEncodingError,
               Log:  fmt.Sprintf("Max tx size is 8 bytes, got %d", len(tx))}
         }
         tx8 := make([]byte, 8)
         copy(tx8[len(tx8)-len(tx):], tx)
         txValue := binary.BigEndian.Uint64(tx8)
         if txValue < uint64(app.txCount) {
            return types.ResponseCheckBlock{
               Code: code.CodeTypeBadNonce,
               Log:  fmt.Sprintf("Invalid nonce. Expected >= %v, got %v", app.txCount, txValue)}
         }
         app.txCount++
      }
   }
   return types.ResponseCheckBlock{Code: code.CodeTypeOK}
}
```

在 Begin Block 中，应用应在检查块之前将状态恢复到原始状态:

```
func (app *CounterApplication) DeliverTx(tx []byte) types.ResponseDeliverTx {
   if app.serial {
      app.txCount = app.originalTxCount   //restore the txCount state
   }
   app.txCount++
   return types.ResponseDeliverTx{Code: code.CodeTypeOK}
}
```

txCount 就像ethermint 中的nonce，在进入deliverTx 阶段时应该恢复.而一些操作，如检查 tx 签名不需要再次执行.所以deliverTx可以专注于如何应用一个tx，而忽略对tx的检查，因为之前所有的检查都已经在checkBlock阶段完成了.

一个可选的优化是将 deliveryTx 更改为 deliveryBlock.因为块已经被checkBlock检查过，所以里面的所有交易都是有效的.所以app可以缓存block，在deliverBlock阶段，只需要在缓存中应用block即可.这种优化可以节省deliverTx 中的网络电流.



## 状态

拒绝

## 决定

性能影响被认为太大.见[#2384](https://github.com/tendermint/tendermint/issues/2384)

## 结果

### 积极的

- 更稳健地保护对手提出一个充满无效交易的区块.

### 消极的

- 添加新的接口方法.应用程序逻辑需要调整以吸引它.
- 通过 ABCI 发送所有 tx 数据两次
- 潜在的冗余验证(例如 CheckBlock 和
  DeliverTx)

### 中性的
