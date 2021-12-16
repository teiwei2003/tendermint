# 在 Go 中创建一个应用程序

## 指南假设

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

## 内置应用与外部应用

为了获得最佳性能，最好与您的应用程序一起运行
Tendermint 核心。 [Cosmos SDK](https://github.com/cosmos/cosmos-sdk) 编写
这边走。请参考[编写内置 Tendermint Core 应用程序]
Go](./go-built-in.md) 指南了解详情。

拥有一个单独的应用程序可能会给你更好的安全保证
进程将通过已建立的二进制协议进行通信。嫩肤
核心将无法访问应用程序的状态。

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
```

在示例目录中创建一个包含以下内容的 `main.go` 文件:

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
go run main.go
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

Now I will go through each method explaining when it's called and adding
required business logic.

### 1.3.1 CheckTx

When a new transaction is added to the Tendermint Core, it will ask the
application to check it (validate the format, signatures, etc.).

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

当 Tendermint Core 决定区块时，它会被转移到
应用程序分为 3 个部分:`BeginBlock`，每笔交易一个 `DeliverTx` 和
最后是`EndBlock`。 DeliverTx 正在异步传输，但
预计响应将有序进行。

```go
func (app *KVStoreApplication) BeginBlock(req abcitypes.RequestBeginBlock) abcitypes.ResponseBeginBlock {
 app.currentBatch = app.db.NewTransaction(true)
 return abcitypes.ResponseBeginBlock{}
}
```

在这里我们创建一个批处理，它将存储块的交易。

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

The complete specification can be found
[here](https://docs.tendermint.com/master/spec/abci/).

## 1.4 启动应用程序和 Tendermint Core 实例

将以下代码放入“main.go”文件中:

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

然后我们启动 ABCI 服务器并添加一些信号处理以优雅地停止
它在收到 SIGTERM 或 Ctrl-C 后。 Tendermint Core 将充当客户端，
它连接到我们的服务器并向我们发送交易和其他消息。

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

## 1.5 启动和运行

我们将使用 [Go modules](https://github.com/golang/go/wiki/Modules)
依赖管理。

```bash
export GO111MODULE=on
go mod init github.com/me/example
```

这应该创建一个 `go.mod` 文件。 当前教程仅适用于
Tendermint 的 master 分支，所以让我们确保我们使用的是最新版本:

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

要创建默认配置、nodeKey 和私有验证器文件，让我们
执行 `tendermint init 验证器`。 但在我们这样做之前，我们需要安装
Tendermint 核心。 请参考[官方
指南](https://docs.tendermint.com/master/introduction/install.html)。 如果你是
从源代码安装，不要忘记检查最新版本(`git
结帐 vX.Y.Z`)。 不要忘记检查应用程序是否使用相同的
主要版本。

```bash
rm -rf /tmp/example
TMHOME="/tmp/example" tendermint init validator

I[2019-07-16|18:20:36.480] Generated private validator                  module=main keyFile=/tmp/example/config/priv_validator_key.json stateFile=/tmp/example2/data/priv_validator_state.json
I[2019-07-16|18:20:36.481] Generated node key                           module=main path=/tmp/example/config/node_key.json
I[2019-07-16|18:20:36.482] Generated genesis file                       module=main path=/tmp/example/config/genesis.json
I[2019-07-16|18:20:36.483] Generated config                             module=main mode=validator
```

随意探索生成的文件，可以在
`/tmp/example/config` 目录。 可以找到有关配置的文档
[此处](https://docs.tendermint.com/master/tendermint-core/configuration.html)。

我们准备开始我们的应用程序:

```bash
rm example.sock
./example

badger 2019/07/16 18:25:11 INFO: All 0 tables opened in 0s
badger 2019/07/16 18:25:11 INFO: Replaying file id: 0 at offset: 0
badger 2019/07/16 18:25:11 INFO: Replay took: 300.4s
I[2019-07-16|18:25:11.523] Starting ABCIServer                          impl=ABCIServ
```

然后我们需要启动 Tendermint Core 并将其指向我们的应用程序。 住宿
在应用程序目录中执行:

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

响应应包含提交此事务的高度。

现在让我们检查给定的键现在是否存在及其值:

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

"dGVuZGVybWludA==" 和 "cm9ja3M=" 是 ASCII 的 base64 编码
相应地“tendermint”和“rocks”。

##结尾

我希望一切顺利，你的第一个，但希望不是最后一个，
Tendermint Core 应用程序已启动并正在运行。 如果没有，请[打开一个问题
Github](https://github.com/tendermint/tendermint/issues/new/choose)。 挖
更深入地阅读 [文档](https://docs.tendermint.com/master/)。
