## 記録

ロギングにより詳細が追加され、ノードオペレーターは探しているものをより適切に識別できます. Tendermintは、グローバルおよびモジュールごとのログレベルをサポートします. これにより、ノードオペレーターは必要な情報のみを確認でき、開発者は作業中の特定の変更に集中できます.

## ログレベルを設定する

ログレベルには、 `info`、` debug`、 `error`の3つがあります. これらは、コマンドラインから `tendermint start --log-level" "`または `config.toml`ファイルで設定できます.

-`info`情報は情報メッセージを意味します. これは、モジュールが開始、停止、およびそれらがどのように実行されているかを示すために使用されます.
-`debug`デバッグは、さまざまな呼び出しや問題を追跡するために使用されます. デバッグはコードベース全体で広く使用されており、ロギングが非常に長くなる可能性があります.
-`error`エラーは、エラーが発生したことを意味します. エラーログは、ノードを停止させる可能性のある潜在的な問題を表している可能性があります.

在 `config.toml` 中:

```toml
# Output level for logging, including package level options
log-level = "info"
```

Via the command line:

```sh
tendermint start --log-level "info"
```

## モジュールリスト

以下は、Tendermintログで遭遇する可能性のあるモジュールのリストです.
彼らが何をしているのかを概説することはめったにありません.

-`abci-client` [アプリケーションアーキテクチャガイド](../app-dev/app-architecture.md)で説明されているように、TendermintはABCIとして機能します
  クライアントはアプリケーションに関連しており、次の3つの接続を維持します.
  メモリプール、コンセンサス、クエリ. TendermintCoreで使用されるコードは
  ここ(https://github.com/tendermint/tendermint/tree/master/abci/client)にあります.
-`blockchain`は、ストレージ、プール(ピアのセット)、およびリアクターを提供します
  ピア間でブロックを保存および交換するために使用されます.
-`コンセンサス `テンダーミントコアのコア、
  コンセンサスアルゴリズムの実装. 2つ含まれています
  「サブモジュール」: `wal`(先行書き込みロギング)はデータを確保するために使用されます
  整合性と「再生」再生ブロックおよび回復時のメッセージ
  クラッシュから.
  [こちら](https://github.com/tendermint/tendermint/blob/master/types/events.go).
  `subscribe`RPCメソッドを呼び出すことでそれらをサブスクライブできます.参照する
  詳細については、[RPCドキュメント](../tendermint-core/rpc.md)にアクセスしてください.
-`mempool`メモリプールモジュールは、いつでもすべての着信トランザクションを処理します
  それらはピアまたはアプリケーションから来ています.
-`p2p`は、ピアツーピア通信の抽象化を提供します.にとって
  詳しくはご確認ください
  [Readme](https://github.com/tendermint/spec/tree/master/spec/p2p).
-`rpc-server`RPCサーバー.実装の詳細については、以下をお読みください
  [doc.go](https://github.com/tendermint/tendermint/blob/master/rpc/jsonrpc/doc.go).
-`state`は、最新の状態と実行サブモジュールを表します.ここで、
  アプリケーションでブロックを実行します.
-`statesync`は、ノードをプルーニングされた履歴とすばやく同期する方法を提供します.

### ウォーキングの例

まず、3つの接続(メモリプール、コンセンサス、クエリ)を作成します.
アプリケーション(この場合、ローカルで `kvstore`を実行します).

```sh
I[10-04|13:54:27.364] Starting multiAppConn                        module=proxy impl=multiAppConn
I[10-04|13:54:27.366] Starting localClient                         module=abci-client connection=query impl=localClient
I[10-04|13:54:27.366] Starting localClient                         module=abci-client connection=mempool impl=localClient
I[10-04|13:54:27.367] Starting localClient                         module=abci-client connection=consensus impl=localClient
```

次に、TendermintCoreとアプリケーションがハンドシェイクを実行します.

```sh
I[10-04|13:54:27.367] ABCI Handshake                               module=consensus appHeight=90 appHash=E0FBAFBF6FCED8B9786DDFEB1A0D4FA2501BADAD
I[10-04|13:54:27.368] ABCI Replay Blocks                           module=consensus appHeight=90 storeHeight=90 stateHeight=90
I[10-04|13:54:27.368] Completed ABCI Handshake - Tendermint and App are synced module=consensus appHeight=90 appHash=E0FBAFBF6FCED8B9786DDFEB1A0D4FA2501BADAD
```

その後、イベントスイッチ、リアクター、
そして、UPNPディスカバリーを実行してIPアドレスを検出します.

```sh
I[10-04|13:54:27.374] Starting EventSwitch                         module=types impl=EventSwitch
I[10-04|13:54:27.375] This node is a validator                     module=consensus
I[10-04|13:54:27.379] Starting Node                                module=main impl=Node
I[10-04|13:54:27.381] Local listener                               module=p2p ip=:: port=26656
I[10-04|13:54:27.382] Getting UPNP external address                module=p2p
I[10-04|13:54:30.386] Could not perform UPNP discover              module=p2p err="write udp4 0.0.0.0:38238->239.255.255.250:1900: i/o timeout"
I[10-04|13:54:30.386] Starting DefaultListener                     module=p2p impl=Listener(@10.0.2.15:26656)
I[10-04|13:54:30.387] Starting P2P Switch                          module=p2p impl="P2P Switch"
I[10-04|13:54:30.387] Starting MempoolReactor                      module=mempool impl=MempoolReactor
I[10-04|13:54:30.387] Starting BlockchainReactor                   module=blockchain impl=BlockchainReactor
I[10-04|13:54:30.387] Starting ConsensusReactor                    module=consensus impl=ConsensusReactor
I[10-04|13:54:30.387] ConsensusReactor                             module=consensus fastSync=false
I[10-04|13:54:30.387] Starting ConsensusState                      module=consensus impl=ConsensusState
I[10-04|13:54:30.387] Starting WAL                                 module=consensus wal=/home/vagrant/.tendermint/data/cs.wal/wal impl=WAL
I[10-04|13:54:30.388] Starting TimeoutTicker                       module=consensus impl=TimeoutTicker
```

TendermintCoreレポートの2行目に注意してください.「このノードは
バリデーター」.単なるオブザーバー(通常のノード)にすることもできます.

次に、WALからのすべてのメッセージを再生します.

```sh
I[10-04|13:54:30.390] Catchup by replaying consensus messages      module=consensus height=91
I[10-04|13:54:30.390] Replay: New Step                             module=consensus height=91 round=0 step=RoundStepNewHeight
I[10-04|13:54:30.390] Replay: Done                                 module=consensus
```

「開始ノード」メッセージは、すべての準備ができていることを意味します.

```sh
I[10-04|13:54:30.391] Starting RPC HTTP server on tcp socket 0.0.0.0:26657 module=rpc-server
I[10-04|13:54:30.392] Started node                                 module=main nodeInfo="NodeInfo{id: DF22D7C92C91082324A1312F092AA1DA197FA598DBBFB6526E, moniker: anonymous, network: test-chain-3MNw2N [remote , listen 10.0.2.15:26656], version: 0.11.0-10f361fc ([wire_version=0.6.2 p2p_version=0.5.0 consensus_version=v1/0.2.2 rpc_version=0.7.0/3 tx_index=on rpc_addr=tcp://0.0.0.0:26657])}"
```

次は標準のブロック作成サイクルです.新しいブロックを入力します
ラウンドし、ブロックを提案し、投票の2/3以上を獲得してから、
事前にコミットし、最終的にブロックを送信する機会があります. 詳細については、
[ビザンチンのコンセンサスを参照してください
アルゴリズム](https://github.com/tendermint/spec/blob/master/spec/consensus/consensus.md).

```sh
I[10-04|13:54:30.393] enterNewRound(91/0). Current: 91/0/RoundStepNewHeight module=consensus
I[10-04|13:54:30.393] enterPropose(91/0). Current: 91/0/RoundStepNewRound module=consensus
I[10-04|13:54:30.393] enterPropose: Our turn to propose            module=consensus proposer=125B0E3C5512F5C2B0E1109E31885C4511570C42 privValidator="PrivValidator{125B0E3C5512F5C2B0E1109E31885C4511570C42 LH:90, LR:0, LS:3}"
I[10-04|13:54:30.394] Signed proposal                              module=consensus height=91 round=0 proposal="Proposal{91/0 1:21B79872514F (-1,:0:000000000000) {/10EDEDD7C84E.../}}"
I[10-04|13:54:30.397] Received complete proposal block             module=consensus height=91 hash=F671D562C7B9242900A286E1882EE64E5556FE9E
I[10-04|13:54:30.397] enterPrevote(91/0). Current: 91/0/RoundStepPropose module=consensus
I[10-04|13:54:30.397] enterPrevote: ProposalBlock is valid         module=consensus height=91 round=0
I[10-04|13:54:30.398] Signed and pushed vote                       module=consensus height=91 round=0 vote="Vote{0:125B0E3C5512 91/00/1(Prevote) F671D562C7B9 {/89047FFC21D8.../}}" err=null
I[10-04|13:54:30.401] Added to prevote                             module=consensus vote="Vote{0:125B0E3C5512 91/00/1(Prevote) F671D562C7B9 {/89047FFC21D8.../}}" prevotes="VoteSet{H:91 R:0 T:1 +2/3:F671D562C7B9242900A286E1882EE64E5556FE9E:1:21B79872514F BA{1:X} map[]}"
I[10-04|13:54:30.401] enterPrecommit(91/0). Current: 91/0/RoundStepPrevote module=consensus
I[10-04|13:54:30.401] enterPrecommit: +2/3 prevoted proposal block. Locking module=consensus hash=F671D562C7B9242900A286E1882EE64E5556FE9E
I[10-04|13:54:30.402] Signed and pushed vote                       module=consensus height=91 round=0 vote="Vote{0:125B0E3C5512 91/00/2(Precommit) F671D562C7B9 {/80533478E41A.../}}" err=null
I[10-04|13:54:30.404] Added to precommit                           module=consensus vote="Vote{0:125B0E3C5512 91/00/2(Precommit) F671D562C7B9 {/80533478E41A.../}}" precommits="VoteSet{H:91 R:0 T:2 +2/3:F671D562C7B9242900A286E1882EE64E5556FE9E:1:21B79872514F BA{1:X} map[]}"
I[10-04|13:54:30.404] enterCommit(91/0). Current: 91/0/RoundStepPrecommit module=consensus
I[10-04|13:54:30.405] Finalizing commit of block with 0 txs        module=consensus height=91 hash=F671D562C7B9242900A286E1882EE64E5556FE9E root=E0FBAFBF6FCED8B9786DDFEB1A0D4FA2501BADAD
I[10-04|13:54:30.405] Block{
  Header{
    ChainID:        test-chain-3MNw2N
    Height:         91
    Time:           2017-10-04 13:54:30.393 +0000 UTC
    NumTxs:         0
    LastBlockID:    F15AB8BEF9A6AAB07E457A6E16BC410546AA4DC6:1:D505DA273544
    LastCommit:     56FEF2EFDB8B37E9C6E6D635749DF3169D5F005D
    Data:
    Validators:     CE25FBFF2E10C0D51AA1A07C064A96931BC8B297
    App:            E0FBAFBF6FCED8B9786DDFEB1A0D4FA2501BADAD
  }#F671D562C7B9242900A286E1882EE64E5556FE9E
  Data{

  }#
  Commit{
    BlockID:    F15AB8BEF9A6AAB07E457A6E16BC410546AA4DC6:1:D505DA273544
    Precommits: Vote{0:125B0E3C5512 90/00/2(Precommit) F15AB8BEF9A6 {/FE98E2B956F0.../}}
  }#56FEF2EFDB8B37E9C6E6D635749DF3169D5F005D
}#F671D562C7B9242900A286E1882EE64E5556FE9E module=consensus
I[10-04|13:54:30.408] Executed block                               module=state height=91 validTxs=0 invalidTxs=0
I[10-04|13:54:30.410] Committed state                              module=state height=91 txs=0 hash=E0FBAFBF6FCED8B9786DDFEB1A0D4FA2501BADAD
I[10-04|13:54:30.410] Recheck txs                                  module=mempool numtxs=0 height=91
```
