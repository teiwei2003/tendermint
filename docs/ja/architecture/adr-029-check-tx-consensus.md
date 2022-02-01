# ADR 029:投票する前にブロックトランザクションを確認してください

## 変更ログ

2018年4月10日:更新して質問へのリンクを提供
[#2384](https://github.com/tendermint/tendermint/issues/2384)と拒否の理由
2018年9月19日:最初のドラフト

## 環境

現在、txの有効性を2つの方法でチェックしています.

1.mempool接続のcheckTxを介して.
2.deliverTxコンセンサスを介して接続します.

最初のものは外部txが入ってくるときに呼び出されるので、今回はノードがプロポーザーである必要があります. 2つ目は、外部ブロックがコミットフェーズに入り、コミットフェーズに到達したときに呼び出されます.ノードはブロックの提案者である必要はありませんが、ブロック内のtxをチェックする必要があります.

2番目のケースでは、ブロック内に無効なトランザクションが多数ある場合、すべてのノードがブロック内のほとんどのトランザクションが無効であると判断するには遅すぎます.無効なトランザクションをブロックチェーンに記録しないことをお勧めします.

## 推奨される解決策

したがって、事前投票を発行する前に、txsの有効性を確認する方法を見つける必要があります. 現在、ブロックが完了したかどうかを判断するためのcs.isProposalComplete()があります. 我々は持つことができる

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

BlockExecutorのこのメソッドは、ブロック内のすべてのトランザクションの有効性をチェックします.

ただし、checkTxはアプリケーションのメモリプールで使用されるのと同じ状態を共有するため、このメソッドはこの方法で実装しないでください. したがって、アプリケーションで新しいインターフェイスメソッドcheckBlockを定義して、deliverTxと同じ状態を使用するように指示する必要があります.

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

すべてのアプリケーションはこのメソッドを実装する必要があります. たとえば、カウンター:

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

Begin Blockでは、アプリケーションはブロックをチェックする前に状態を元の状態に復元する必要があります.

```
func (app *CounterApplication) DeliverTx(tx []byte) types.ResponseDeliverTx {
   if app.serial {
      app.txCount = app.originalTxCount   //restore the txCount state
   }
   app.txCount++
   return types.ResponseDeliverTx{Code: code.CodeTypeOK}
}
```

txCountは、ethermintのナンスに似ており、deliverTxフェーズに入るときに復元する必要があります. tx署名のチェックなどの一部の操作は、再度実行する必要はありません.したがって、deliverTxは、txの適用方法に焦点を合わせ、txチェックを無視できます.これは、以前のすべてのチェックがcheckBlockステージで完了しているためです.

オプションの最適化は、deliveryTxをdeliveryBlockに変更することです.ブロックはcheckBlockによってチェックされているため、その中のすべてのトランザクションが有効です.そのため、アプリはブロックをキャッシュできます.deliverBlockステージでは、キャッシュにブロックを適用するだけで済みます.この最適化により、deliverTxのネットワーク電流を節約できます.



## ステータス

ごみ

## 決定

パフォーマンスへの影響は大きすぎると見なされます. [#2384](https://github.com/tendermint/tendermint/issues/2384)を参照してください

## 結果

### ポジティブ

-無効なトランザクションでいっぱいのブロックを提案することから、より強力に対戦相手を保護します.

### ネガティブ

-新しいインターフェイスメソッドを追加します.アプリケーションロジックは、それを引き付けるために調整する必要があります.
-ABCIを介してすべてのtxデータを2回送信します
-潜在的な冗長検証(例:CheckBlockおよび
  DeliverTx)

### ニュートラル
