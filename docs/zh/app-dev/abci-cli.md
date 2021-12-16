# 使用 ABCI-CLI

为了方便 ABCI 服务器和简单应用程序的测试和调试，我们
构建了一个 CLI，`abci-cli`，用于从命令发送 ABCI 消息
线。

## 安装

确保您 [已安装 Go](https://golang.org/doc/install)。

接下来，安装 `abci-cli` 工具和示例应用程序:

```sh
git clone https://github.com/tendermint/tendermint.git
cd tendermint
make install_abci
```

现在运行 `abci-cli` 来查看命令列表:

```sh
Usage:
  abci-cli [command]

Available Commands:
  batch       Run a batch of abci commands against an application
  check_tx    Validate a tx
  commit      Commit the application state and return the Merkle root hash
  console     Start an interactive abci console for multiple commands
  deliver_tx  Deliver a new tx to the application
  kvstore     ABCI demo example
  echo        Have the application echo a message
  help        Help about any command
  info        Get some info about the application
  query       Query the application state
  set_option  Set an options on the application

Flags:
      --abci string      socket or grpc (default "socket")
      --address string   address of application socket (default "tcp://127.0.0.1:26658")
  -h, --help             help for abci-cli
  -v, --verbose          print the command and results as if it were a console session

Use "abci-cli [command] --help" for more information about a command.
```

## KVStore - 第一个例子

`abci-cli` 工具允许我们将 ABCI 消息发送到我们的应用程序，
帮助构建和调试它们。

最重要的消息是`deliver_tx`、`check_tx`和`commit`，
但还有其他的方便、配置和信息
目的。

我们将启动一个 kvstore 应用程序，该应用程序同时安装
如上面的“abci-cli”。 kvstore 只是将交易存储在 merkle 中
树。

可以找到它的代码
[这里](https://github.com/tendermint/tendermint/blob/master/abci/cmd/abci-cli/abci-cli.go)
看起来像:

```go
func cmdKVStore(cmd *cobra.Command, args []string) error {
    logger := log.NewTMLogger(log.NewSyncWriter(os.Stdout))

    // Create the application - in memory or persisted to disk
    var app types.Application
    if flagPersist == "" {
        app = kvstore.NewKVStoreApplication()
    } else {
        app = kvstore.NewPersistentKVStoreApplication(flagPersist)
        app.(*kvstore.PersistentKVStoreApplication).SetLogger(logger.With("module", "kvstore"))
    }

    // Start the listener
    srv, err := server.NewServer(flagAddrD, flagAbci, app)
    if err != nil {
        return err
    }
    srv.SetLogger(logger.With("module", "abci-server"))
    if err := srv.Start(); err != nil {
        return err
    }

    // Stop upon receiving SIGTERM or CTRL-C.
    tmos.TrapSignal(logger, func() {
        // Cleanup
        srv.Stop()
    })

    // Run forever.
    select {}
}
```

Start by running:

```sh
abci-cli kvstore
```

在另一个终端中，运行

```sh
abci-cli echo hello
abci-cli info
```

你会看到类似的东西:

```sh
-> data: hello
-> data.hex: 68656C6C6F
```

and:

```sh
-> data: {"size":0}
-> data.hex: 7B2273697A65223A307D
```

ABCI 应用程序必须提供两件事:

- 一个套接字服务器
- ABCI 消息的处理程序

当我们运行 `abci-cli` 工具时，我们会打开一个到
应用程序的套接字服务器，发送给定的 ABCI 消息，并等待
回复。

服务器对于特定语言可能是通用的，我们提供了一个
[参考实现在
Golang](https://github.com/tendermint/tendermint/tree/master/abci/server)。见
[其他 ABCI 实现列表](https://github.com/tendermint/awesome#ecosystem) 用于服务器
其他语言。

处理程序特定于应用程序，并且可能是任意的，因此
只要它是确定性的并且符合 ABCI 接口
规格。

因此，当我们运行 `abci-cli info` 时，我们会打开一个到 ABCI 的新连接
服务器，它调用应用程序上的 `Info()` 方法，它告诉
我们是 Merkle 树中的交易数量。

现在，由于每个命令都会打开一个新连接，我们提供
`abci-cli console` 和 `abci-cli batch` 命令，允许多个 ABCI
要通过单个连接发送的消息。

运行 `abci-cli console` 应该会让你进入一个交互式控制台
向您的应用程序讲述 ABCI 消息。

尝试运行这些命令:

```sh
> echo hello
-> code: OK
-> data: hello
-> data.hex: 0x68656C6C6F

> info
-> code: OK
-> data: {"size":0}
-> data.hex: 0x7B2273697A65223A307D

> commit
-> code: OK
-> data.hex: 0x0000000000000000

> deliver_tx "abc"
-> code: OK

> info
-> code: OK
-> data: {"size":1}
-> data.hex: 0x7B2273697A65223A317D

> commit
-> code: OK
-> data.hex: 0x0200000000000000

> query "abc"
-> code: OK
-> log: exists
-> height: 2
-> value: abc
-> value.hex: 616263

> deliver_tx "def=xyz"
-> code: OK

> commit
-> code: OK
-> data.hex: 0x0400000000000000

> query "def"
-> code: OK
-> log: exists
-> height: 3
-> value: xyz
-> value.hex: 78797A
```

请注意，如果我们执行 `deliver_tx "abc"` 它将存储 `(abc, abc)`，但是如果
我们做`deliver_tx "abc=efg"`它会存储`(abc, efg)`。

同样，您可以将命令放在一个文件中并运行
`abci-cli --verbose batch < myfile`。

## 赏金

想用您最喜欢的语言编写应用程序吗？！ 我们会很高兴
将您添加到我们的 [生态系统](https://github.com/tendermint/awesome#ecosystem)！
请参阅 [funding](https://github.com/interchainio/funding) 机会来自
[Interchain Foundation](https://interchain.io/) 用于新语言等的实现。

`abci-cli` 是专为测试和调试而设计的。 在一个真实的
部署，发送消息的角色由 Tendermint 承担，它
使用三个独立的连接连接到应用程序，每个连接都有自己的
消息模式。

有关运行 ABCI 应用程序的示例
Tendermint，请参阅[入门指南](./getting-started.md)。
接下来是 ABCI 规范。
