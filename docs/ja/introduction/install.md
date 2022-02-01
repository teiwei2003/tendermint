# Tendermintをインストールする

## バイナリから

ビルド済みのバイナリをダウンロードするには、[リリースページ](https://github.com/tendermint/tendermint/releases)を参照してください.

## 自作を使用する

自作ソフトウェアを使用するだけでTendermintバイナリをインストールすることもできます.

```
brew install tendermint
```

## ソースコードから

`go` [インストール済み](https://golang.org/doc/install)と必要なものが必要です
環境変数を設定するには、次のコマンドを使用して完了します.

```sh
echo export GOPATH=\"\$HOME/go\" >> ~/.bash_profile
echo export PATH=\"\$PATH:\$GOPATH/bin\" >> ~/.bash_profile
```

ソースコードを取得します.

```sh
git clone https://github.com/tendermint/tendermint.git
cd tendermint
```

次に、以下を実行します.

```sh
make install
```

バイナリファイルを `$ GOPATH/bin`に入れるか、次を使用します.

```sh
make build
```

バイナリファイルを `./build`に置きます.

_免責事項_Tendermintのバイナリは、DWARFなしでビルド/インストールされます
シンボルテーブル. DWARFを使用してTendermintをビルド/インストールする場合
シンボルとデバッグ情報、makeの `BUILD_FLAGS`から` -s-w`を削除します
資料.

最新のテンダーミントがインストールされました. インストールを確認するには、
ランニング:

```sh
tendermint version
```

## 走る

単純なインプロセスアプリケーションを使用してシングルノードブロックチェーンを開始するには:

```sh
tendermint init validator
tendermint start --proxy-app=kvstore
```

##再インストール

すでにTendermintをインストールしていて、更新している場合は、

```sh
make install
```

アップグレードするには、

```sh
git pull origin master
make install
```

## コンパイルをサポートするためにCLevelDBを使用する

[LevelDB](https://github.com/google/leveldb)をインストールします(最小バージョンは1.7です).

snappy(オプション)を使用してLevelDBをインストールします. 以下はUbuntuのコマンドです.

```sh
sudo apt-get update
sudo apt install build-essential

sudo apt-get install libsnappy-dev

wget https://github.com/google/leveldb/archive/v1.20.tar.gz && \
  tar -zxvf v1.20.tar.gz && \
  cd leveldb-1.20/ && \
  make && \
  sudo cp -r out-static/lib* out-shared/lib* /usr/local/lib/ && \
  cd include/ && \
  sudo cp -r leveldb /usr/local/include/ && \
  sudo ldconfig && \
  rm -f v1.20.tar.gz
```

データベースバックエンドを `cleveldb`に設定します.

```toml
# config/config.toml
db_backend = "cleveldb"
```

Tendermintをインストールするには、次のコマンドを実行します.

```sh
CGO_LDFLAGS="-lsnappy" make install TENDERMINT_BUILD_OPTIONS=cleveldb
```

または実行:

```sh
CGO_LDFLAGS="-lsnappy" make build TENDERMINT_BUILD_OPTIONS=cleveldb
```

which puts the binary in `./build`.
