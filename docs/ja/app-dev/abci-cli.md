# ABCI-CLIを使用する

ABCIサーバーと単純なアプリケーションのテストとデバッグを容易にするために、
コマンドからABCIメッセージを送信するために使用されるCLI`abci-cli`を構築しました
弦.

## インストール

[Goがインストールされている](https://golang.org/doc/install)ことを確認してください.

次に、 `abci-cli`ツールとサンプルアプリケーションをインストールします.

```sh
git clone https://github.com/tendermint/tendermint.git
cd tendermint
make install_abci
```

次に、 `abci-cli`を実行して、コマンドのリストを表示します.

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

## KVStore-最初の例

`abci-cli`ツールを使用すると、アプリケーションにABCIメッセージを送信できます.
それらの構築とデバッグを支援します.

最も重要なメッセージは、 `deliver_tx`、` check_tx`、および `commit`です.
しかし、他にも便利な構成と情報があります
目的.

同時にインストールされるkvstoreアプリケーションを起動します
上記の「abci-cli」など. kvstoreはトランザクションをmerkleに保存するだけです
木.

あなたはそのコードを見つけることができます
[こちら](https://github.com/tendermint/tendermint/blob/master/abci/cmd/abci-cli/abci-cli.go)
のように見える:

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

実行することから始めます:

```sh
abci-cli kvstore
```

別のターミナルで、

```sh
abci-cli echo hello
abci-cli info
```

次のようなものが表示されます.

```sh
-> data: hello
-> data.hex: 68656C6C6F
```

and:

```sh
-> data: {"size":0}
-> data.hex: 7B2273697A65223A307D
```

ABCIアプリケーションは2つのものを提供する必要があります.

-ソケットサーバー
-ABCIメッセージハンドラ

`abci-cli`ツールを実行すると、
アプリケーションのソケットサーバーは、指定されたABCIメッセージを送信し、待機します
返事.

サーバーは特定の言語に対応している場合があります.
[リファレンスはで実装されています
Golang](https://github.com/tendermint/tendermint/tree/master/abci/server).見る
[他のABCI実装のリスト](https://github.com/tendermint/awesome#ecosystem)サーバー用
他の言語.

ハンドラーはアプリケーション固有であり、任意である可能性があるため、
それが決定論的であり、ABCIインターフェースに準拠している限り
仕様.

したがって、 `abci-cli info`を実行すると、ABCIへの新しい接続が開きます.
サーバー、それはアプリケーションのInfo()メソッドを呼び出します、それは伝えます
マークルツリーのトランザクション数です.

今、すべてのコマンドが新しい接続を開くので、私たちは提供します
`abci-cliconsole`および` abci-clibatch`コマンドは複数のABCIを許可します
単一の接続を介して送信されるメッセージ.

`abci-cli console`を実行すると、インタラクティブコンソールが表示されます
ABCIニュースについてアプリケーションに伝えます.

次のコマンドを実行してみてください.

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

`deliver_tx" abc "`を実行すると、 `(abc、abc)`が保存されますが、
`deliver_tx" abc = efg "`を実行すると、 `(abc、efg)`が格納されます.

同様に、コマンドをファイルに入れて実行することができます
`abci-cli --verbose batch <myfile`.

## 報奨金

あなたの好きな言語でアプリケーションを書きたいですか？ ！ 私たちは幸せになります
私たちの[エコシステム](https://github.com/tendermint/awesome#ecosystem)にあなたを追加してください！
[資金](https://github.com/interchainio/funding)を参照してください.機会は
[InterchainFoundation](https://interchain.io/)は、新しい言語の実現などに利用されています.

`abci-cli`は、テストとデバッグ用に設計されています. 実際には
展開、メッセージ送信の役割はTendermintによって引き継がれます.
3つの別々の接続を使用して、それぞれが独自のアプリケーションに接続します
メッセージモード.

ABCIアプリケーションの実行例
Tendermintについては、[Getting Started Guide](./ getting-started.md)を参照してください.
次はABCI仕様です.
