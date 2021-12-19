# Docker構成

Docker Composeを使用すると、1つのコマンドでローカルテストネットを起動できます。

## 必須

1. [テンダーミントのインストール](../Introduction/install.md)
2. [Dockerのインストール](https://docs.docker.com/engine/installation/)
3. [docker-composeをインストール](https://docs.docker.com/compose/install/)

## 我慢する

`tendermint`バイナリとオプションの` tendermint/localnode`をビルドします
Dockerイメージ。

バイナリファイルはコンテナにマウントされるため、更新は必要ありません。
画像を再構成します。

```sh
# Build the linux binary in ./build
make build-linux

# (optionally) Build tendermint/localnode image
make build-docker-localnode
```

## Run a testnet

To start a 4 node testnet run:

```sh
make localnet-start
```

ノードは、RPCサーバーをポート26657、26660、26662、および26664にバインドします
ホスト。

このファイルは、ローカルノードイメージを使用して4ノードネットワークを作成します。

ネットワークノードは、P2Pお​​よびRPCエンドポイントをホストに公開します
ポート26656-26657、26659-26660、26661-26662、および26663-26664でそれぞれ。

最初のノード( `node0`)は、2つの追加ポートを公開します。分析用の6060
[`pprof`](https://golang.org/pkg/net/http/pprof)および` 9090`-Prometheusの場合
サーバー(チェックアウトの開始方法がわからない場合["最初のステップ|
Prometheus "](https://prometheus.io/docs/introduction/first_steps/))。

バイナリを更新するには、バイナリを再構築してノードを再起動します。

```sh
make build-linux
make localnet-start
```

## 構成

`make localnet-start`は、`。/build`に4ノードのテストネット用のファイルを作成します
`tenderminttestnet`コマンドを呼び出します。

`。/build`ディレクトリを`/tendermint`マウントポイントにマウントしてアタッチします
コンテナへのバイナリファイルと構成ファイル。

バリデーター/非バリデーターの数を変更するには、 `localnet-start` Makefileターゲットを変更してください[ここ](../../Makefile):

```makefile
localnet-start: localnet-stop
  @if ! [ -f build/node0/config/genesis.json ]; then docker run --rm -v $(CURDIR)/build:/tendermint:Z tendermint/localnode testnet --v 5 --n 3 --o . --populate-persistent-peers --starting-ip-address 192.167.10.2 ; fi
  docker-compose up
```

このコマンドは、5つのバリデーターと3つのバリデーターの構成ファイルを生成します。
非検証者。 新しい構成ファイルを生成することに加えて、docker-composeファイルも編集する必要があります。
生成された構成ファイルを最大限に活用するには、さらに4つのノードを追加する必要があります。

```yml
  node3: # bump by 1 for every node
    container_name: node3 # bump by 1 for every node
    image: "tendermint/localnode"
    environment:
      - ID=3
      - LOG=${LOG:-tendermint.log}
    ports:
      - "26663-26664:26656-26657" # Bump 26663-26664 by one for every node
    volumes:
      - ./build:/tendermint:Z
    networks:
      localnet:
        ipv4_address: 192.167.10.5 # bump the final digit by 1 for every node
```

Before running it, don't forget to cleanup the old files:

```sh
# Clear the build folder
rm -rf ./build/node*
```

## ABCIコンテナを構成する

4ノードのセットアップで独自のABCIアプリケーションを使用するには、[docker-compose.yaml](https://github.com/tendermint/tendermint/blob/master/docker-compose.yml)ファイルを編集して画像を追加しますABCIアプリケーションに追加します。

```yml
 abci0:
    container_name: abci0
    image: "abci-image"
    build:
      context: .
      dockerfile: abci.Dockerfile
    command: <insert command to run your abci application>
    networks:
      localnet:
        ipv4_address: 192.167.10.6

  abci1:
    container_name: abci1
    image: "abci-image"
    build:
      context: .
      dockerfile: abci.Dockerfile
    command: <insert command to run your abci application>
    networks:
      localnet:
        ipv4_address: 192.167.10.7

  abci2:
    container_name: abci2
    image: "abci-image"
    build:
      context: .
      dockerfile: abci.Dockerfile
    command: <insert command to run your abci application>
    networks:
      localnet:
        ipv4_address: 192.167.10.8

  abci3:
    container_name: abci3
    image: "abci-image"
    build:
      context: .
      dockerfile: abci.Dockerfile
    command: <insert command to run your abci application>
    networks:
      localnet:
        ipv4_address: 192.167.10.9

```

各ノードの[コマンド](https://github.com/tendermint/tendermint/blob/master/networks/local/localnode/Dockerfile#L12)を上書きして、そのABCIに接続します。

```yml
  node0:
    container_name: node0
    image: "tendermint/localnode"
    ports:
      - "26656-26657:26656-26657"
    environment:
      - ID=0
      - LOG=$${LOG:-tendermint.log}
    volumes:
      - ./build:/tendermint:Z
    command: node --proxy-app=tcp://abci0:26658
    networks:
      localnet:
        ipv4_address: 192.167.10.2
```

node1、node2、node3についても同じことを行い、[run testnet](https://github.com/tendermint/tendermint/blob/master/docs/networks/docker-compose.md#run-a-testnet)

## 記録

ログは、添付ボリュームの下の「tendermint.log」ファイルに保存されます。 もしも
起動時に `LOG`環境変数は` stdout`に設定され、ログは保存されません。
しかし、画面に印刷されます。

## 特別なバイナリファイル

異なる名前のバイナリファイルが複数ある場合は、どれを指定できますか
`BINARY`環境変数を使用して実行します。 バイナリパスは相対的です
付属のボリュームに。
