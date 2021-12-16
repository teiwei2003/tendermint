# 在 Go 中创建一个内置应用程序

## 指导假设

本指南专为想要开始使用 Tendermint 的初学者而设计
从头开始的核心应用程序。它并不假设您有任何先前
使用 Tendermint Core 的经验。

Tendermint Core 是采用状态的拜占庭容错 (BFT) 中间件
转换机 - 用任何编程语言编写 - 并且安全
在许多机器上复制它。

虽然 Tendermint Core 是用 Golang 编程语言编写的，但之前
本指南不需要了解它。你可以在我们到期时学习它
它的简单性。但是，您可能希望通过 [Learn X in Y 分钟
Where X=Go](https://learnxinyminutes.com/docs/go/) 先来熟悉一下
自己的语法。

通过遵循本指南，您将创建一个 Tendermint 核心项目
称为 kvstore，一个(非常)简单的分布式 BFT 键值存储。

> 注意:请在本指南中使用已发布的 Tendermint 版本。这些指南适用于最新版本。请不要使用大师。

## 内置应用与外部应用

在与 Tendermint Core 相同的进程中运行您的应用程序将提供
你最好的表现。

对于其他语言，您的应用程序必须与 Tendermint Core 通信
通过 TCP、Unix 域套接字或 gRPC。

## 1.1 安装 Go

请参考[官方安装指南]
去](https://golang.org/doc/install)。

验证您是否安装了最新版本的 Go:

```bash
$ go version
go version go1.16.x darwin/amd64
```

## 1.2 创建一个新的 Go 项目

我们将首先创建一个新的 Go 项目。

```bash
mkdir kvstore
cd kvstore
go mod init github.com/<github_username>/<repo_name>
```

在示例目录中创建一个包含以下内容的 `main.go` 文件:

> 注意:本教程无需克隆或分叉 Tendermint。

```go
package main

import (
 "fmt"
)

func main() {
 fmt.Println("Hello, Tendermint Core")
}
```

运行时，这应该将“Hello, Tendermint Core”打印到标准输出。

```bash
$ go run main.go
Hello, Tendermint Core
```

## 1.3 编写 Tendermint Core 应用程序

Tendermint Core 通过应用程序与应用程序通信
区块链接口(ABCI)。 所有消息类型都在 [protobuf
文件](https://github.com/tendermint/tendermint/blob/master/proto/tendermint/abci/types.proto)。
这允许 Tendermint Core 运行以任何编程方式编写的应用程序
语。

创建一个名为“app.go”的文件，内容如下:

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

现在我将通过每种方法解释它何时被调用并添加
所需的业务逻辑。

### 1.3.1 CheckTx

当一个新的交易被添加到 Tendermint Core 时，它会询问
应用程序来检查它(验证格式、签名等)。

```go
import "bytes"

func (app *KVStoreApplication) isValid(tx []byte) (code uint32) {
 // check format
 parts := bytes.Split(tx, []byte("="))
 if len(parts) != 2 {
  return 1
 }

 key, value := parts[0], parts[1]

 // check if the same key=value already exists
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

如果这还没有编译，请不要担心。

如果交易没有`{bytes}={bytes}`的形式，我们返回`1`
代码。 当相同的 key=value 已经存在(相同的 key 和 value)时，我们返回 `2`
代码。 对于其他人，我们返回一个零代码，表明它们是有效的。

请注意，任何具有非零代码的内容都将被视为无效(`-1`、`100`、
等)由 Tendermint 核心。

有效的交易最终将被提交，因为它们不是太大并且
有足够的气。 要了解有关天然气的更多信息，请查看 [“
规范"](https://docs.tendermint.com/master/spec/abci/apps.html#gas)。

对于我们将使用的底层键值存储
[badger](https://github.com/dgraph-io/badger)，这是一个可嵌入的，
持久且快速的键值 (KV) 数据库。

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

当 Tendermint Core 决定区块时，它会转移到
应用程序分为 3 个部分:`BeginBlock`，每笔交易一个 `DeliverTx` 和
最后是`EndBlock`。 DeliverTx 正在异步传输，但
预计响应将有序进行。

```go
func (app *KVStoreApplication) BeginBlock(req abcitypes.RequestBeginBlock) abcitypes.ResponseBeginBlock {
 app.currentBatch = app.db.NewTransaction(true)
 return abcitypes.ResponseBeginBlock{}
}

```

Here we create a batch, which will store block's transactions.

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

如果交易格式错误或相同的 key=value 已经存在，我们
再次返回非零代码。 否则，我们将其添加到当前批次中。

在当前的设计中，一个区块可能包含不正确的交易(那些
通过 CheckTx，但未通过 DeliverTx 或提议者包含的交易
直接地)。 这样做是出于性能原因。

请注意，我们不能在 `DeliverTx` 内提交事务，因为在这种情况下
可以并行调用的 `Query` 将返回不一致的数据(即
即使实际块不存在，它也会报告某些值已经存在
尚未提交)。

`Commit` 指示应用程序保持新状态。

```go
func (app *KVStoreApplication) Commit() abcitypes.ResponseCommit {
 app.currentBatch.Commit()
 return abcitypes.ResponseCommit{Data: []byte{}}
}
```

### 1.3.3 查询

现在，当客户端想知道特定键/值何时存在时，它
将调用 Tendermint Core RPC `/abci_query` 端点，后者又会调用
应用程序的`Query` 方法。

应用程序可以免费提供自己的 API。 但是通过使用 Tendermint Core
作为代理，客户端(包括[轻客户端
包](https://godoc.org/github.com/tendermint/tendermint/light)) 可以利用
跨不同应用程序的统一 API。 此外，他们将不必致电
否则单独的 Tendermint 核心 API 用于额外的证明。

请注意，我们在此处不包含证明。

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

可以找到完整的规范
[此处](https://docs.tendermint.com/master/spec/abci/)。

## 1.4 在同一个进程中启动一个应用程序和一个 Tendermint Core 实例

将以下代码放入“main.go”文件中:

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
 // read config
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

 // create logger
 logger := log.NewTMLogger(log.NewSyncWriter(os.Stdout))
 var err error
 logger, err = tmflags.ParseLogLevel(config.LogLevel, logger, cfg.DefaultLogLevel)
 if err != nil {
  return nil, fmt.Errorf("failed to parse log level: %w", err)
 }

 // read private validator
 pv := privval.LoadFilePV(
  config.PrivValidatorKeyFile(),
  config.PrivValidatorStateFile(),
 )

 // read node key
 nodeKey, err := p2p.LoadNodeKey(config.NodeKeyFile())
 if err != nil {
  return nil, fmt.Errorf("failed to load node's key: %w", err)
 }

 // create node
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

这是一大堆代码，让我们把它分解成几部分。

首先，我们初始化 Badger 数据库并创建一个应用程序实例:

```go
db, err := badger.Open(badger.DefaultOptions("/tmp/badger"))
if err != nil {
 fmt.Fprintf(os.Stderr, "failed to open badger db: %v", err)
 os.Exit(1)
}
defer db.Close()
app := NewKVStoreApplication(db)
```

对于 **Windows** 用户，重新启动此应用程序将使獾抛出错误，因为它需要截断值日志。 有关这方面的更多信息，请访问 [此处](https://github.com/dgraph-io/badger/issues/744)。
这可以通过将 truncate 选项设置为 true 来避免，如下所示:

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

// create node
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

`NewNode` 需要一些东西，包括一个配置文件、一个私有的
验证器、节点密钥和其他一些以构建完整节点。

注意我们在这里使用 `abcicli.NewLocalClientCreator` 来创建一个本地客户端
通过套接字或 gRPC 进行通信的一种。

[viper](https://github.com/spf13/viper) 用于读取配置，
我们稍后将使用 `tendermint init` 命令生成它。

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

我们使用`FilePV`，它是一个私有验证器(即签署共识的东西
消息)。 通常，您会使用 `SignerRemote` 连接到外部
[HSM](https://kb.certus.one/hsm.html)。

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

最后，我们启动节点并添加一些信号处理以优雅地停止它
收到 SIGTERM 或 Ctrl-C 后。

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

要创建默认配置、nodeKey 和私有验证器文件，让我们
执行 `tendermint init 验证器`。 但在我们这样做之前，我们需要安装
Tendermint 核心。 请参考[官方
指南](https://docs.tendermint.com/master/introduction/install.html)。 如果你是
从源代码安装，不要忘记检查最新版本(`git
结帐 vX.Y.Z`)。 不要忘记检查应用程序是否使用相同的
主要版本。

```bash
$ rm -rf /tmp/example
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

响应应包含提交此事务的高度。

现在让我们检查给定的键现在是否存在及其值:

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

"dGVuZGVybWludA==" 和 "cm9ja3M=" 是 ASCII 的 base64 编码
相应地“tendermint”和“rocks”。

## 结尾

我希望一切顺利，你的第一个，但希望不是最后一个，
Tendermint Core 应用程序已启动并正在运行。 如果没有，请[打开一个问题
Github](https://github.com/tendermint/tendermint/issues/new/choose)。 挖
更深入地阅读 [文档](https://docs.tendermint.com/master/)。
