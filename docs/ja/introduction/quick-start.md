# クイックスタート

## 概要

これはクイックスタートガイドです。 テンダーミントに興味のある方
有効で、すぐに開始したい場合は、続行してください。 バイナリがインストールされていることを確認してください。
そうでない場合は、[インストール](./install.md)を確認してください。

## 初期化

走る:

```sh
tendermint init validator
```

必要なファイルは、単一のローカルノード用に作成されます。

これらのファイルは `$ HOME/.tendermin`にあります。

```sh
$ ls $HOME/.tendermint

config  data

$ ls $HOME/.tendermint/config/

config.toml  genesis.json  node_key.json  priv_validator.json
```

単一のローカルノードの場合、それ以上の構成は必要ありません。 次に、クラスターの構成について詳しく説明します。

## ローカルノード

単純なインプロセスアプリケーションを使用して、Tendermintを起動します。
```sh
tendermint start --proxy-app=kvstore
```

> Note: `kvstore` is a non persistent app, if you would like to run an application with persistence run `--proxy-app=persistent_kvstore`

そして、ブロックが流入し始めます:

```sh
I[01-06|01:45:15.592] Executed block                               module=state height=1 validTxs=0 invalidTxs=0
I[01-06|01:45:15.624] Committed state                              module=state height=1 txs=0 appHash=
```

ステータスの確認:

```sh
curl -s localhost:26657/status
```

###トランザクションを送信する

KVstoreアプリケーションを実行した後、トランザクションを送信できます。

```sh
curl -s 'localhost:26657/broadcast_tx_commit?tx="abcd"'
```

そして、それが以下に適用されるかどうかを確認します。

```sh
curl -s 'localhost:26657/abci_query?data="abcd"'
```

キーと値を使用してトランザクションを送信することもできます。

```sh
curl -s 'localhost:26657/broadcast_tx_commit?tx="name=satoshi"'
```

そして、キーを照会します。

```sh
curl -s 'localhost:26657/abci_query?data="name"'
```

値は16進数で返されます。

## ノードクラスター

まず、4台のUbuntuクラウドマシンを作成します。 以下は数値的にテストされています
Ocean Ubuntu 16.04 x64(3GB/1CPU、20GB SSD)。 それぞれのIPを参照します
次のアドレスは、IP1、IP2、IP3、およびIP4です。

次に、 `ssh`が各マシンに入り、[このスクリプト](https://git.io/fFfOR)を実行します。
```sh
curl -L https://git.io/fFfOR | bash
source ~/.profile
```

これにより、 `go`およびその他の依存関係がインストールされ、Tendermintソースコードが取得されてから、` tendermint`バイナリがコンパイルされます。

次に、 `tendermint testnet`コマンドを使用して構成ファイルの4つのディレクトリ(`。/mytestnet`にあります)を作成し、各ディレクトリをクラウド内の関連するマシンにコピーして、各マシンが `$ HOME/mytestnet/nodeを持つようにします。 [0-3] `ディレクトリ。

ネットワークを開始する前に、ピア識別子が必要です(IPは十分ではなく、変更できます)。 それらをID1、ID2、ID3、ID4と呼びます。

```sh
tendermint show_node_id --home ./mytestnet/node0
tendermint show_node_id --home ./mytestnet/node1
tendermint show_node_id --home ./mytestnet/node2
tendermint show_node_id --home ./mytestnet/node3
```

最後に、各マシンで実行します。

```sh
tendermint start --home ./mytestnet/node0 --proxy-app=kvstore --p2p.persistent-peers="ID1@IP1:26656,ID2@IP2:26656,ID3@IP3:26656,ID4@IP4:26656"
tendermint start --home ./mytestnet/node1 --proxy-app=kvstore --p2p.persistent-peers="ID1@IP1:26656,ID2@IP2:26656,ID3@IP3:26656,ID4@IP4:26656"
tendermint start --home ./mytestnet/node2 --proxy-app=kvstore --p2p.persistent-peers="ID1@IP1:26656,ID2@IP2:26656,ID3@IP3:26656,ID4@IP4:26656"
tendermint start --home ./mytestnet/node3 --proxy-app=kvstore --p2p.persistent-peers="ID1@IP1:26656,ID2@IP2:26656,ID3@IP3:26656,ID4@IP4:26656"
```

3番目のノードが開始された後、ブロックが流入し始めることに注意してください
バリデーターの2/3以上( `genesis.json`で定義)がすでにオンラインになっているためです。
永続ピアは `config.toml`でも指定できます。 構成オプションの詳細については、[ここ](../tendermint-core/configuration.md)を参照してください。

次に、上記の単一のローカルノードの例で説明されているように、トランザクションを送信できます。
