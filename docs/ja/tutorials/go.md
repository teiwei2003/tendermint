# Goでアプリケーションを作成する

## ガイド仮説

このガイドは、テンダーミントを使い始めたい初心者を対象としています.
ゼロからのコアアプリケーション.以前に持っていることを前提とはしていません
TendermintCoreの使用経験.

Tendermint Coreは、状態を採用するビザンチンフォールトトレラント(BFT)ミドルウェアです.
翻訳者-任意のプログラミング言語で書かれている-そして安全
多くのマシンにコピーします.

Tendermint CoreはGolangプログラミング言語で書かれていますが、以前は
このガイドはそれを理解する必要はありません.あなたは私たちが期日になるときにそれを学ぶことができます
そのシンプルさ.ただし、[Y分でXを学ぶ
X = Goの場合](https://learnxinyminutes.com/docs/go/)最初に慣れましょう
独自の文法.

このガイドに従うことで、Tendermintコアプロジェクトを作成します
kvstoreと呼ばれる、(非常に)単純な分散BFTキー値ストア.

## 組み込みアプリケーションと外部アプリケーション

最高のパフォーマンスを得るには、アプリケーションで実行するのが最適です
テンダーミントコア. [Cosmos SDK](https://github.com/cosmos/cosmos-sdk)
こちらです. [組み込みのTendermintCoreアプリケーションを作成する]を参照してください.
詳細については、Go](./go-built-in.md)ガイドを参照してください.

別のアプリケーションを使用すると、セキュリティがより確実に保証される場合があります
プロセスは、確立されたバイナリプロトコルを介して通信します.肌の若返り
コアはアプリケーションの状態にアクセスできなくなります.

## 1.1Goのインストール

[公式インストールガイド]を参照してください
Go](https://golang.org/doc/install).

Goの最新バージョンを使用していることを確認します.

```bash
$ go version
go version go1.16.x darwin/amd64
```

## 1.2 创建一个新的 Go 项目

我们将首先创建一个新的 Go 项目.

```bash
mkdir kvstore
cd kvstore
```

次の内容のmain.goファイルをサンプルディレクトリに作成します.

```go
package main

import (
 "fmt"
)

func main() {
 fmt.Println("Hello, Tendermint Core")
}
```

実行時に、これは「Hello、TendermintCore」を標準出力に出力するはずです.

```bash
go run main.go
Hello, Tendermint Core
```

## 1.3Tendermintコアアプリケーションの作成

Tendermint Coreは、アプリケーションを介してアプリケーションと通信します
ブロックリンクポート(ABCI). すべてのメッセージタイプは[protobuf
ファイル](https://github.com/tendermint/tendermint/blob/master/proto/tendermint/abci/types.proto).
これにより、TendermintCoreはプログラムで記述されたアプリケーションを実行できます.
言語.

次の内容の「app.go」という名前のファイルを作成します.

```go
package main

import (
 abcitypes "github.com/tendermint/tendermint/abci/types"
)

type KVStoreApplication struct {}

var _ abcitypes.Application = (*KVStoreApplication)(nil)

func NewKVStoreApplication() *KVStoreApplication {
 return &KVStoreApplication{}
}

func (KVStoreApplication) Info(req abcitypes.RequestInfo) abcitypes.ResponseInfo {
 return abcitypes.ResponseInfo{}
}

func (KVStoreApplication) DeliverTx(req abcitypes.RequestDeliverTx) abcitypes.ResponseDeliverTx {
 return abcitypes.ResponseDeliverTx{Code: 0}
}

func (KVStoreApplication) CheckTx(req abcitypes.RequestCheckTx) abcitypes.ResponseCheckTx {
 return abcitypes.ResponseCheckTx{Code: 0}
}

func (KVStoreApplication) Commit() abcitypes.ResponseCommit {
 return abcitypes.ResponseCommit{}
}

func (KVStoreApplication) Query(req abcitypes.RequestQuery) abcitypes.ResponseQuery {
 return abcitypes.ResponseQuery{Code: 0}
}

func (KVStoreApplication) InitChain(req abcitypes.RequestInitChain) abcitypes.ResponseInitChain {
 return abcitypes.ResponseInitChain{}
}

func (KVStoreApplication) BeginBlock(req abcitypes.RequestBeginBlock) abcitypes.ResponseBeginBlock {
 return abcitypes.ResponseBeginBlock{}
}

func (KVStoreApplication) EndBlock(req abcitypes.RequestEndBlock) abcitypes.ResponseEndBlock {
 return abcitypes.ResponseEndBlock{}
}

func (KVStoreApplication) ListSnapshots(abcitypes.RequestListSnapshots) abcitypes.ResponseListSnapshots {
 return abcitypes.ResponseListSnapshots{}
}

func (KVStoreApplication) OfferSnapshot(abcitypes.RequestOfferSnapshot) abcitypes.ResponseOfferSnapshot {
 return abcitypes.ResponseOfferSnapshot{}
}

func (KVStoreApplication) LoadSnapshotChunk(abcitypes.RequestLoadSnapshotChunk) abcitypes.ResponseLoadSnapshotChunk {
 return abcitypes.ResponseLoadSnapshotChunk{}
}

func (KVStoreApplication) ApplySnapshotChunk(abcitypes.RequestApplySnapshotChunk) abcitypes.ResponseApplySnapshotChunk {
 return abcitypes.ResponseApplySnapshotChunk{}
}
```

Now I will go through each method explaining when it's called and adding
required business logic.

### 1.3.1 CheckTx

When a new transaction is added to the Tendermint Core, it will ask the
application to check it (validate the format, signatures, etc.).

```go
import "bytes"

func (app *KVStoreApplication) isValid(tx []byte) (code uint32) {
//check format
 parts := bytes.Split(tx, []byte("="))
 if len(parts) != 2 {
  return 1
 }

 key, value := parts[0], parts[1]

//check if the same key=value already exists
 err := app.db.View(func(txn *badger.Txn) error {
  item, err := txn.Get(key)
  if err != nil && err != badger.ErrKeyNotFound {
   return err
  }
  if err == nil {
   return item.Value(func(val []byte) error {
    if bytes.Equal(val, value) {
     code = 2
    }
    return nil
   })
  }
  return nil
 })
 if err != nil {
  panic(err)
 }

 return code
}

func (app *KVStoreApplication) CheckTx(req abcitypes.RequestCheckTx) abcitypes.ResponseCheckTx {
 code := app.isValid(req.Tx)
 return abcitypes.ResponseCheckTx{Code: code, GasWanted: 1}
}
```

Don't worry if this does not compile yet.

If the transaction does not have a form of `{bytes}={bytes}`, we return `1`
code. When the same key=value already exist (same key and value), we return `2`
code. For others, we return a zero code indicating that they are valid.

Note that anything with non-zero code will be considered invalid (`-1`, `100`,
etc.) by Tendermint Core.

Valid transactions will eventually be committed given they are not too big and
have enough gas. To learn more about gas, check out ["the
specification"](https://docs.tendermint.com/master/spec/abci/apps.html#gas).

For the underlying key-value store we'll use
[badger](https://github.com/dgraph-io/badger), which is an embeddable,
persistent and fast key-value (KV) database.

```go
import "github.com/dgraph-io/badger"

type KVStoreApplication struct {
 db           *badger.DB
 currentBatch *badger.Txn
}

func NewKVStoreApplication(db *badger.DB) *KVStoreApplication {
 return &KVStoreApplication{
  db: db,
 }
}
```

### 1.3.2 BeginBlock -> DeliverTx -> EndBlock -> Commit

Tendermint Coreがブロックを決定すると、ブロックはに転送されます
アプリケーションは3つの部分に分かれています: `BeginBlock`、トランザクションごとに1つの` DeliverTx`、
最後は `EndBlock`です. DeliverTxは非同期で送信していますが、
整然とした対応が期待されます.

```go
func (app *KVStoreApplication) BeginBlock(req abcitypes.RequestBeginBlock) abcitypes.ResponseBeginBlock {
 app.currentBatch = app.db.NewTransaction(true)
 return abcitypes.ResponseBeginBlock{}
}
```

在这里我们创建一个批处理，它将存储块的交易.

```go
func (app *KVStoreApplication) DeliverTx(req abcitypes.RequestDeliverTx) abcitypes.ResponseDeliverTx {
 code := app.isValid(req.Tx)
 if code != 0 {
  return abcitypes.ResponseDeliverTx{Code: code}
 }

 parts := bytes.Split(req.Tx, []byte("="))
 key, value := parts[0], parts[1]

 err := app.currentBatch.Set(key, value)
 if err != nil {
  panic(err)
 }

 return abcitypes.ResponseDeliverTx{Code: 0}
}
```

トランザクション形式が間違っているか、同じkey = valueがすでに存在する場合、
ゼロ以外のコードが再び返されます. それ以外の場合は、現在のバッチに追加します.

現在の設計では、ブロックに誤ったトランザクションが含まれている可能性があります(これらのトランザクション
CheckTxに合格しましたが、DeliverTxまたは提案者に含まれるトランザクションに合格しませんでした
直接). これは、パフォーマンス上の理由から行われます.

この場合、 `DeliverTx`内でトランザクションをコミットできないことに注意してください
並行して呼び出すことができるクエリは、一貫性のないデータを返します(つまり、
実際のブロックが存在しない場合でも、特定の値がすでに存在していることが報告されます
まだ提出されていません).

`Commit`は、新しい状態を維持するようにアプリケーションに指示します.

```go
func (app *KVStoreApplication) Commit() abcitypes.ResponseCommit {
 app.currentBatch.Commit()
 return abcitypes.ResponseCommit{Data: []byte{}}
}
```

### 1.3.3クエリ

これで、クライアントが特定のキー/値がいつ存在するかを知りたい場合、
Tendermint Core RPC `/abci_query`エンドポイントを呼び出します.エンドポイントは次に呼び出します
アプリケーションの `Query`メソッド.

アプリケーションは独自のAPIを無料で提供できます. しかし、テンダーミントコアを使用することによって
プロキシとして、クライアント([ライトクライアントを含む
パッケージ](https://godoc.org/github.com/tendermint/tendermint/light))が利用可能
さまざまなアプリケーションにまたがる統合API. さらに、彼らは電話する必要はありません
それ以外の場合は、追加の証明のために別のTendermintコアAPIが使用されます.

ここには証拠が含まれていないことに注意してください.

```go
func (app *KVStoreApplication) Query(reqQuery abcitypes.RequestQuery) (resQuery abcitypes.ResponseQuery) {
 resQuery.Key = reqQuery.Data
 err := app.db.View(func(txn *badger.Txn) error {
  item, err := txn.Get(reqQuery.Data)
  if err != nil && err != badger.ErrKeyNotFound {
   return err
  }
  if err == badger.ErrKeyNotFound {
   resQuery.Log = "does not exist"
  } else {
   return item.Value(func(val []byte) error {
    resQuery.Log = "exists"
    resQuery.Value = val
    return nil
   })
  }
  return nil
 })
 if err != nil {
  panic(err)
 }
 return
}
```

The complete specification can be found
[here](https://docs.tendermint.com/master/spec/abci/).

## 1.4アプリケーションとTendermintCoreインスタンスを起動します

次のコードを「main.go」ファイルに入れます.

```go
package main

import (
 "flag"
 "fmt"
 "os"
 "os/signal"
 "syscall"

 "github.com/dgraph-io/badger"

 abciserver "github.com/tendermint/tendermint/abci/server"
 "github.com/tendermint/tendermint/libs/log"
)

var socketAddr string

func init() {
 flag.StringVar(&socketAddr, "socket-addr", "unix://example.sock", "Unix domain socket address")
}

func main() {
 db, err := badger.Open(badger.DefaultOptions("/tmp/badger"))
 if err != nil {
  fmt.Fprintf(os.Stderr, "failed to open badger db: %v", err)
  os.Exit(1)
 }
 defer db.Close()
 app := NewKVStoreApplication(db)

 flag.Parse()

 logger := log.MustNewDefaultLogger(log.LogFormatPlain, log.LogLevelInfo, false)

 server := abciserver.NewSocketServer(socketAddr, app)
 server.SetLogger(logger)
 if err := server.Start(); err != nil {
  fmt.Fprintf(os.Stderr, "error starting socket server: %v", err)
  os.Exit(1)
 }
 defer server.Stop()

 c := make(chan os.Signal, 1)
 signal.Notify(c, os.Interrupt, syscall.SIGTERM)
 <-c
 os.Exit(0)
}
```

これはたくさんのコードです.いくつかの部分に分けてみましょう.

まず、Badgerデータベースを初期化し、アプリケーションインスタンスを作成します.

```go
db, err := badger.Open(badger.DefaultOptions("/tmp/badger"))
if err != nil {
 fmt.Fprintf(os.Stderr, "failed to open badger db: %v", err)
 os.Exit(1)
}
defer db.Close()
app := NewKVStoreApplication(db)
```

** Windows **ユーザーの場合、このアプリケーションを再起動すると、値ログを切り捨てる必要があるため、Badgerはエラーをスローします. 詳細については、[こちら](https://github.com/dgraph-io/badger/issues/744)にアクセスしてください.
これは、次のようにtruncateオプションをtrueに設定することで回避できます.

```go
db, err := badger.Open(badger.DefaultOptions("/tmp/badger").WithTruncate(true))
```

次に、ABCIサーバーを起動し、信号処理を追加して正常に停止します
SIGTERMまたはCtrl-Cを受け取った後です. TendermintCoreはクライアントとして機能します.
サーバーに接続し、トランザクションやその他のメッセージを送信します.

```go
server := abciserver.NewSocketServer(socketAddr, app)
server.SetLogger(logger)
if err := server.Start(); err != nil {
 fmt.Fprintf(os.Stderr, "error starting socket server: %v", err)
 os.Exit(1)
}
defer server.Stop()

c := make(chan os.Signal, 1)
signal.Notify(c, os.Interrupt, syscall.SIGTERM)
<-c
os.Exit(0)
```

## 1.5起動して実行

[Goモジュール](https://github.com/golang/go/wiki/Modules)を使用します
依存管理.

```bash
export GO111MODULE=on
go mod init github.com/me/example
```

これにより、 `go.mod`ファイルが作成されます. 現在のチュートリアルはにのみ適用されます
Tendermintのマスターブランチなので、最新バージョンを使用していることを確認しましょう.

```sh
go get github.com/tendermint/tendermint@97a3e44e0724f2017079ce24d36433f03124c09e
```

This will populate the `go.mod` with a release number followed by a hash for Tendermint.

```go
module github.com/me/example

go 1.16

require (
 github.com/dgraph-io/badger v1.6.2
 github.com/tendermint/tendermint <vX>
)
```

Now we can build the binary:

```bash
go build
```

デフォルト構成、nodeKey、およびプライベートバリデーターファイルを作成するには、
`tendermintinitvalidator`を実行します. ただし、その前に、インストールする必要があります
テンダーミントコア. [公式
ガイド](https://docs.tendermint.com/master/introduction/install.html). もしあなたが
ソースからインストールします.最新バージョンを確認することを忘れないでください( `git
vX.Y.Z`をチェックしてください). アプリが同じものを使用しているかどうかを確認することを忘れないでください
メジャーバージョン.

```bash
rm -rf/tmp/example
TMHOME="/tmp/example" tendermint init validator

I[2019-07-16|18:20:36.480] Generated private validator                  module=main keyFile=/tmp/example/config/priv_validator_key.json stateFile=/tmp/example2/data/priv_validator_state.json
I[2019-07-16|18:20:36.481] Generated node key                           module=main path=/tmp/example/config/node_key.json
I[2019-07-16|18:20:36.482] Generated genesis file                       module=main path=/tmp/example/config/genesis.json
I[2019-07-16|18:20:36.483] Generated config                             module=main mode=validator
```

随意探索生成的文件，可以在
`/tmp/example/config` 目录. 可以找到有关配置的文档
[此处](https://docs.tendermint.com/master/tendermint-core/configuration.html).

我们准备开始我们的应用程序:

```bash
rm example.sock
./example

badger 2019/07/16 18:25:11 INFO: All 0 tables opened in 0s
badger 2019/07/16 18:25:11 INFO: Replaying file id: 0 at offset: 0
badger 2019/07/16 18:25:11 INFO: Replay took: 300.4s
I[2019-07-16|18:25:11.523] Starting ABCIServer                          impl=ABCIServ
```

次に、Tendermint Coreを起動して、アプリケーションをポイントする必要があります. 止まる
アプリケーションディレクトリで実行します.

```bash
TMHOME="/tmp/example" tendermint node --proxy-app=unix://example.sock

I[2019-07-16|18:26:20.362] Version info                                 module=main software=0.32.1 block=10 p2p=7
I[2019-07-16|18:26:20.383] Starting Node                                module=main impl=Node
E[2019-07-16|18:26:20.392] Couldn't connect to any seeds                module=p2p
I[2019-07-16|18:26:20.394] Started node                                 module=main nodeInfo="{ProtocolVersion:{P2P:7 Block:10 App:0} ID_:8dab80770ae8e295d4ce905d86af78c4ff634b79 ListenAddr:tcp://0.0.0.0:26656 Network:test-chain-nIO96P Version:0.32.1 Channels:4020212223303800 Moniker:app48.fun-box.ru Other:{TxIndex:on RPCAddress:tcp://127.0.0.1:26657}}"
I[2019-07-16|18:26:21.440] Executed block                               module=state height=1 validTxs=0 invalidTxs=0
I[2019-07-16|18:26:21.446] Committed state                              module=state height=1 txs=0 appHash=
```

This should start the full node and connect to our ABCI application.

```sh
I[2019-07-16|18:25:11.525] Waiting for new connection...
I[2019-07-16|18:26:20.329] Accepted a new connection
I[2019-07-16|18:26:20.329] Waiting for new connection...
I[2019-07-16|18:26:20.330] Accepted a new connection
I[2019-07-16|18:26:20.330] Waiting for new connection...
I[2019-07-16|18:26:20.330] Accepted a new connection
```

Now open another tab in your terminal and try sending a transaction:

```json
curl -s 'localhost:26657/broadcast_tx_commit?tx="tendermint=rocks"'
{
  "jsonrpc": "2.0",
  "id": "",
  "result": {
    "check_tx": {
      "gasWanted": "1"
    },
    "deliver_tx": {},
    "hash": "CDD3C6DFA0A08CAEDF546F9938A2EEC232209C24AA0E4201194E0AFB78A2C2BB",
    "height": "33"
}
```

応答には、トランザクションがコミットされた高さを含める必要があります.

次に、指定されたキーが存在するかどうかとその値を確認しましょう.

```json
curl -s 'localhost:26657/abci_query?data="tendermint"'
{
  "jsonrpc": "2.0",
  "id": "",
  "result": {
    "response": {
      "log": "exists",
      "key": "dGVuZGVybWludA==",
      "value": "cm9ja3My"
    }
  }
}
```

「dGVuZGVybWludA ==」および「cm9ja3M =」はASCIIbase64エンコーディングです
対応して「テンダーミント」と「ロック」.

## 終わり

私はすべてがうまくいくことを願っています、あなたの最初ですが、最後ではないことを願っています、
TendermintCoreアプリケーションが稼働しています. そうでない場合は、[質問を開いてください
Github](https://github.com/tendermint/tendermint/issues/new/choose). 掘る
[ドキュメント](https://docs.tendermint.com/master/)を詳しく読んでください.
