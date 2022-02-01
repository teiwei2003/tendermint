# Goで組み込みアプリケーションを作成する

## ガイドの仮定

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

>注:このガイドで公開されているバージョンのTendermintを使用してください.これらのガイドラインは最新バージョンに適用されます.マスターは使用しないでください.

## 組み込みアプリケーションと外部アプリケーション

TendermintCoreと同じプロセスでアプリケーションを実行すると
あなたの最高のパフォーマンス.

他の言語の場合、アプリケーションはTendermintCoreと通信する必要があります
TCP、Unixドメインソケット、またはgRPC経由.

## 1.1Goのインストール

[公式インストールガイド]を参照してください
Go](https://golang.org/doc/install).

Goの最新バージョンを使用していることを確認します.

```bash
$ go version
go version go1.16.x darwin/amd64
```

## 1.2新しいGoプロジェクトを作成する

まず、新しいGoプロジェクトを作成します.

```bash
mkdir kvstore
cd kvstore
go mod init github.com/<github_username>/<repo_name>
```

次の内容のmain.goファイルをサンプルディレクトリに作成します.

>注:このチュートリアルでは、Tendermintのクローンを作成したりフォークしたりする必要はありません.

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
$ go run main.go
Hello, Tendermint Core
```

## 1.3 编写 Tendermint Core 应用程序

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

次に、各メソッドによって呼び出されるタイミングを説明し、追加します
必要なビジネスロジック.

### 1.3.1 CheckTx

新しいトランザクションがTendermintCoreに追加されると、
それをチェックするためのアプリケーション(フォーマット、署名などを確認します).

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

これがまだコンパイルされていない場合でも、心配する必要はありません.

トランザクションの形式が `{bytes} = {bytes}`でない場合は、 `1`を返します.
コード. 同じkey = valueがすでに存在する場合(同じkeyとvalue)、 `2`を返します
コード. その他の場合は、有効であることを示すゼロコードを返します.

ゼロ以外のコードを含むコンテンツは無効と見なされることに注意してください( `-1`、` 100`、
など)テンダーミントコアによる.

有効なトランザクションは、大きすぎず、
十分なガス. 天然ガスの詳細については、["
仕様 "](https://docs.tendermint.com/master/spec/abci/apps.html#gas).

基になるKey-Valueストアには、
[アナグマ](https://github.com/dgraph-io/badger)、これは埋め込み可能です、
耐久性があり高速なKey-Value(KV)データベース.

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

Tendermint Coreがブロックを決定すると、次の場所に移動します
アプリケーションは3つの部分に分かれています: `BeginBlock`、トランザクションごとに1つの` DeliverTx`、
最後は `EndBlock`です. DeliverTxは非同期で送信していますが、
整然とした対応が期待されます.

```go
func (app *KVStoreApplication) BeginBlock(req abcitypes.RequestBeginBlock) abcitypes.ResponseBeginBlock {
 app.currentBatch = app.db.NewTransaction(true)
 return abcitypes.ResponseBeginBlock{}
}

```

ここでは、ブロックのトランザクションを格納するバッチを作成します.

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

完全な仕様は見つけることができます
[こちら](https://docs.tendermint.com/master/spec/abci/).

## 1.4同じプロセスでアプリケーションとTendermintCoreインスタンスを起動します

次のコードを「main.go」ファイルに入れます.

```go
package main

import (
 "flag"
 "fmt"
 "os"
 "os/signal"
 "path/filepath"
 "syscall"

 "github.com/dgraph-io/badger"
 "github.com/spf13/viper"

 abci "github.com/tendermint/tendermint/abci/types"
 cfg "github.com/tendermint/tendermint/config"
 tmflags "github.com/tendermint/tendermint/libs/cli/flags"
 "github.com/tendermint/tendermint/libs/log"
 nm "github.com/tendermint/tendermint/node"
 "github.com/tendermint/tendermint/internal/p2p"
 "github.com/tendermint/tendermint/privval"
 "github.com/tendermint/tendermint/proxy"
)

var configFile string

func init() {
 flag.StringVar(&configFile, "config", "$HOME/.tendermint/config/config.toml", "Path to config.toml")
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

 node, err := newTendermint(app, configFile)
 if err != nil {
  fmt.Fprintf(os.Stderr, "%v", err)
  os.Exit(2)
 }

 node.Start()
 defer func() {
  node.Stop()
  node.Wait()
 }()

 c := make(chan os.Signal, 1)
 signal.Notify(c, os.Interrupt, syscall.SIGTERM)
 <-c
}

func newTendermint(app abci.Application, configFile string) (*nm.Node, error) {
//read config
 config := cfg.DefaultValidatorConfig()
 config.RootDir = filepath.Dir(filepath.Dir(configFile))
 viper.SetConfigFile(configFile)
 if err := viper.ReadInConfig(); err != nil {
  return nil, fmt.Errorf("viper failed to read config file: %w", err)
 }
 if err := viper.Unmarshal(config); err != nil {
  return nil, fmt.Errorf("viper failed to unmarshal config: %w", err)
 }
 if err := config.ValidateBasic(); err != nil {
  return nil, fmt.Errorf("config is invalid: %w", err)
 }

//create logger
 logger := log.NewTMLogger(log.NewSyncWriter(os.Stdout))
 var err error
 logger, err = tmflags.ParseLogLevel(config.LogLevel, logger, cfg.DefaultLogLevel)
 if err != nil {
  return nil, fmt.Errorf("failed to parse log level: %w", err)
 }

//read private validator
 pv := privval.LoadFilePV(
  config.PrivValidatorKeyFile(),
  config.PrivValidatorStateFile(),
 )

//read node key
 nodeKey, err := p2p.LoadNodeKey(config.NodeKeyFile())
 if err != nil {
  return nil, fmt.Errorf("failed to load node's key: %w", err)
 }

//create node
 node, err := nm.NewNode(
  config,
  pv,
  nodeKey,
  abcicli.NewLocalClientCreator(app),
  nm.DefaultGenesisDocProviderFunc(config),
  nm.DefaultDBProvider,
  nm.DefaultMetricsProvider(config.Instrumentation),
  logger)
 if err != nil {
  return nil, fmt.Errorf("failed to create new Tendermint node: %w", err)
 }

 return node, nil
}
```

はますのコードです. かの部にててしょしょう.

アナグマを最初に、リストデータベースを作成します.

```go
db, err := badger.Open(badger.DefaultOptions("/tmp/badger"))
if err != nil {
 fmt.Fprintf(os.Stderr, "failed to open badger db: %v", err)
 os.Exit(1)
}
defer db.Close()
app := NewKVStoreApplication(db)
```

** Windows **アプリケーションの機会、このアプリケーションを再見ました、Bedgerはエラーをスローします. 詳細とは、[モザイク](https://github.com/dgraph-io/badger/issues/744)にないしてください.
は、次のように切り捨てるオプションを真に設定することで回避できます.

```go
db, err := badger.Open(badger.DefaultOptions("/tmp/badger").WithTruncate(true))
```

Then we use it to create a Tendermint Core `Node` instance:

```go
flag.Parse()

node, err := newTendermint(app, configFile)
if err != nil {
 fmt.Fprintf(os.Stderr, "%v", err)
 os.Exit(2)
}

...

//create node
node, err := nm.NewNode(
 config,
 pv,
 nodeKey,
 abcicli.NewLocalClientCreator(app),
 nm.DefaultGenesisDocProviderFunc(config),
 nm.DefaultDBProvider,
 nm.DefaultMetricsProvider(config.Instrumentation),
 logger)
if err != nil {
 return nil, fmt.Errorf("failed to create new Tendermint node: %w", err)
}
```

`NewNode`には、構成ファイル、プライベートなど、いくつかのものが必要です.
完全なノードを構築するためのバリデーター、ノードキー、およびその他のいくつか.

ここでは `abcicli.NewLocalClientCreator`を使用してローカルクライアントを作成していることに注意してください
ソケットまたはgRPCを介した通信の一種.

[viper](https://github.com/spf13/viper)を使用して構成を読み取り、
後で `tendermintinit`コマンドを使用して生成します.

```go
config := cfg.DefaultValidatorConfig()
config.RootDir = filepath.Dir(filepath.Dir(configFile))
viper.SetConfigFile(configFile)
if err := viper.ReadInConfig(); err != nil {
 return nil, fmt.Errorf("viper failed to read config file: %w", err)
}
if err := viper.Unmarshal(config); err != nil {
 return nil, fmt.Errorf("viper failed to unmarshal config: %w", err)
}
if err := config.ValidateBasic(); err != nil {
 return nil, fmt.Errorf("config is invalid: %w", err)
}
```

プライベートバリデーター(つまり、コンセンサスに署名するもの)である `FilePV`を使用します
情報). 通常は、 `SignerRemote`を使用して外部に接続します
[HSM](https://kb.certus.one/hsm.html).

```go
pv := privval.LoadFilePV(
 config.PrivValidatorKeyFile(),
 config.PrivValidatorStateFile(),
)

```

`nodeKey` is needed to identify the node in a p2p network.

```go
nodeKey, err := p2p.LoadNodeKey(config.NodeKeyFile())
if err != nil {
 return nil, fmt.Errorf("failed to load node's key: %w", err)
}
```

As for the logger, we use the build-in library, which provides a nice
abstraction over [go-kit's
logger](https://github.com/go-kit/kit/tree/master/log).

```go
logger := log.NewTMLogger(log.NewSyncWriter(os.Stdout))
var err error
logger, err = tmflags.ParseLogLevel(config.LogLevel, logger, cfg.DefaultLogLevel())
if err != nil {
 return nil, fmt.Errorf("failed to parse log level: %w", err)
}
```

最後に、ノードを開始し、信号処理を追加して、ノードを正常に停止します
SIGTERMまたはCtrl-Cを受け取った後.

```go
node.Start()
defer func() {
 node.Stop()
 node.Wait()
}()

c := make(chan os.Signal, 1)
signal.Notify(c, os.Interrupt, syscall.SIGTERM)
<-c
```

## 1.5 Getting Up and Running

We are going to use [Go modules](https://github.com/golang/go/wiki/Modules) for
dependency management.

```bash
export GO111MODULE=on
go mod init github.com/me/example
```

This should create a `go.mod` file. The current tutorial only works with
the master branch of Tendermint. so let's make sure we're using the latest version:

```sh
go get github.com/tendermint/tendermint@master
```

This will populate the `go.mod` with a release number followed by a hash for Tendermint.

```go
module github.com/me/example

go 1.15

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
$ rm -rf/tmp/example
$ TMHOME="/tmp/example" tendermint init validator

I[2019-07-16|18:40:36.480] Generated private validator                  module=main keyFile=/tmp/example/config/priv_validator_key.json stateFile=/tmp/example2/data/priv_validator_state.json
I[2019-07-16|18:40:36.481] Generated node key                           module=main path=/tmp/example/config/node_key.json
I[2019-07-16|18:40:36.482] Generated genesis file                       module=main path=/tmp/example/config/genesis.json
I[2019-07-16|18:40:36.483] Generated config                             module=main mode=validator
```

We are ready to start our application:

```bash
$ ./example -config "/tmp/example/config/config.toml"

badger 2019/07/16 18:42:25 INFO: All 0 tables opened in 0s
badger 2019/07/16 18:42:25 INFO: Replaying file id: 0 at offset: 0
badger 2019/07/16 18:42:25 INFO: Replay took: 695.227s
E[2019-07-16|18:42:25.818] Couldn't connect to any seeds                module=p2p
I[2019-07-16|18:42:26.853] Executed block                               module=state height=1 validTxs=0 invalidTxs=0
I[2019-07-16|18:42:26.865] Committed state                              module=state height=1 txs=0 appHash=
```

Now open another tab in your terminal and try sending a transaction:

```bash
$ curl -s 'localhost:26657/broadcast_tx_commit?tx="tendermint=rocks"'
{
  "jsonrpc": "2.0",
  "id": "",
  "result": {
    "check_tx": {
      "gasWanted": "1"
    },
    "deliver_tx": {},
    "hash": "1B3C5A1093DB952C331B1749A21DCCBB0F6C7F4E0055CD04D16346472FC60EC6",
    "height": "128"
  }
}
```

応答には、トランザクションがコミットされた高さを含める必要があります.

次に、指定されたキーが存在するかどうかとその値を確認しましょう.

```json
$ curl -s 'localhost:26657/abci_query?data="tendermint"'
{
  "jsonrpc": "2.0",
  "id": "",
  "result": {
    "response": {
      "log": "exists",
      "key": "dGVuZGVybWludA==",
      "value": "cm9ja3M="
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
