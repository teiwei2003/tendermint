# ADR 037:配信ブロック

著者:ダニエル・ラシーン(@ danil-lashin)

## 変更ログ

2019年3月13日:最初のドラフト

## 環境

最初のダイアログ:https://github.com/tendermint/tendermint/issues/2901

一部のアプリケーションは、トランザクションを並行して処理できます。または、少なくとも一部のアプリケーションは処理できます。
tx処理の一部を並列化できます。今では開発者には不可能です
Tendermintがそれに応じて提供するため、txを並行して実行します。

## 決定

これで、Tendermintには `BeginBlock`、` EndBlock`、 `Commit`、` DeliverTx`ステップがあります
ブロックの実行中。このドキュメントでは、これらの手順を1つのDeliverBlockに結合することを推奨しています
ステップ。これにより、アプリ開発者は希望する方法を決定できます
トランザクションを実行します(並列または連続)。それはまた単純化し、
アプリケーションとTendermint間の通信を高速化します。

@jaekwonとして[言及](https://github.com/tendermint/tendermint/issues/2901#issuecomment-477746128)
議論では、すべてのアプリケーションがこのソリューションの恩恵を受けるわけではありません。特定の状況下で、
アプリケーションがそれに応じてトランザクションを処理すると、ブロックチェーンの速度が低下します。
開始するには、ブロック全体がアプリケーションに転送されるまで待機する必要があるためです。
それに対処します。さらに、ABCIが完全に変更された場合は、すべてのアプリケーションを強制する必要があります
それらの実装を完全に変更します。これが私が別のABCIを紹介することを提案する理由です
タイプ。

# 変更を実装する

現在この構造になっているデフォルトのアプリケーションインターフェイスに加えて

```go
type Application interface {
    // Info and Mempool methods...

    // Consensus Connection
    InitChain(RequestInitChain) ResponseInitChain    // Initialize blockchain with validators and other info from TendermintCore
    BeginBlock(RequestBeginBlock) ResponseBeginBlock // Signals the beginning of a block
    DeliverTx(tx []byte) ResponseDeliverTx           // Deliver a tx for full processing
    EndBlock(RequestEndBlock) ResponseEndBlock       // Signals the end of a block, returns changes to the validator set
    Commit() ResponseCommit                          // Commit the state and return the application Merkle root hash
}
```

this doc proposes to add one more:

```go
type Application interface {
    // Info and Mempool methods...

    // Consensus Connection
    InitChain(RequestInitChain) ResponseInitChain           // Initialize blockchain with validators and other info from TendermintCore
    DeliverBlock(RequestDeliverBlock) ResponseDeliverBlock  // Deliver full block
    Commit() ResponseCommit                                 // Commit the state and return the application Merkle root hash
}

type RequestDeliverBlock struct {
    Hash                 []byte
    Header               Header
    Txs                  Txs
    LastCommitInfo       LastCommitInfo
    ByzantineValidators  []Evidence
}

type ResponseDeliverBlock struct {
    ValidatorUpdates      []ValidatorUpdate
    ConsensusParamUpdates *ConsensusParams
    Tags                  []kv.Pair
    TxResults             []ResponseDeliverTx
}

```

さらに、ABCIアプリケーションで使用されるタイプを指定する新しい構成パラメーターを追加する必要があります。
たとえば、 `abci_type`にすることができます。 次に、2つのタイプがあります。
-`Advanced`-現在のABCI
-`Simple`-推奨される実装

## ステータス

レビュー中

## 結果

### ポジティブ

-5つのメソッドホエイを実装する代わりに、新しい開発者向けに簡単な紹介とチュートリアルを提供します
実装する必要があるのは3)だけです
-txsは並行して処理できます
-よりシンプルなインターフェース
-Tendermintとアプリ間のより高速な通信

### ネガティブ

-Tendermintは2種類のABCIをサポートするようになりました
