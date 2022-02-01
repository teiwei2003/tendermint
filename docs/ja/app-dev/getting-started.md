# 入門

## 最初のTendermintアプリケーション

一般的なブロックチェーンエンジンとして、Tendermintは不明です
実行するアプリケーション. したがって、完全なブロックチェーンを実行するには、
何か便利なものを得るには、2つのプログラムを開始する必要があります.1つはTendermint Coreで、もう1つはTendermintCoreです.
もう1つはアプリケーションで、どのプログラムでも作成できます.
言語. リコール[に導入
ABCI](../ Introduction/what-is-tendermint.md#abci-overview)Tendermint Coreは、すべてのp2pとコンセンサスを処理し、トランザクションをに転送するだけです.
アプリケーションを検証する必要がある場合、またはアプリケーションの準備ができている場合
ブロックに送信します.

このガイドでは、アプリケーションの実行方法の例をいくつか示します.
テンダーミントを使用してください.

### インストール

最初に使用するアプリケーションはGoで記述されています. それらをインストールするには、
[Goをインストール](https://golang.org/doc/install)する必要があります
`$ GOPATH/bin`を` $ PATH`に追加し、次の手順を使用してgoモジュールを有効にします.

```bash
echo export GOPATH=\"\$HOME/go\" >> ~/.bash_profile
echo export PATH=\"\$PATH:\$GOPATH/bin\" >> ~/.bash_profile
```

Then run

```sh
go get github.com/tendermint/tendermint
cd $GOPATH/src/github.com/tendermint/tendermint
make install_abci
```

これで、 `abci-cli`がインストールされているはずです.`kvstore`に気付くでしょう.
コマンド、記述されたサンプルアプリケーション
進行中. JavaScriptで記述されたアプリケーションについては、以下を参照してください.

それでは、いくつかのアプリケーションを実行しましょう！

## KVStore-最初の例


kvstoreアプリケーションは[Merkle
ツリー](https://en.wikipedia.org/wiki/Merkle_tree)はすべてを保存するだけです
トレード. トランザクションに `key = value`などの` = `が含まれている場合、
`value`はMerkleツリーの` key`の下に保存されます. それ以外は、
完全なトランザクションバイトは、キーと値として保存されます.

kvstoreアプリケーションを起動してみましょう.

```sh
abci-cli kvstore
```

別のターミナルで、Tendermintを起動できます. あなたはすでに持っている必要があります
Tendermintバイナリがインストールされます. そうでない場合は、以下の手順に従ってください
[ここ](../ Introduction/install.md). Tendermintを実行したことがない場合
使用前:

```sh
tendermint init validator
tendermint start
```

Tendermintを使用したことがある場合は、新しいデータのデータをリセットする必要がある場合があります
「tendermintunsafe_reset_all」を実行して、ブロックチェーンを実現します. その後、実行することができます
`tendermint start`はTendermintを起動し、アプリケーションに接続します. もっと
詳細については、[Tendermintの使用ガイド](../ tendermint-core/using-tendermint.md)を参照してください.

テンダーミントがブロックを作っているのが見えるはずです！ ステータスを取得できます
Tendermintノードは次のとおりです.

```sh
curl -s localhost:26657/status
```

`-s`は` curl`を沈黙させます. より良い出力のために、結果をにパイプします
[jq](https://stedolan.github.io/jq/)や `json_pp`などのツール.

それでは、いくつかのトランザクションをkvstoreに送信しましょ

```sh
curl -s 'localhost:26657/broadcast_tx_commit?tx="abcd"'
```

URLを一重引用符( `'`)で囲んでいることに注意してください.
二重引用符( `" `)はbashでエスケープされません.このコマンドは
バイト `abcd`とのトランザクション、つまり` abcd`は2つのキーとして保存されます
そして、マークルツリーの値. 応答は何かに見えるはずです
お気に入り:

```json
{
  "jsonrpc": "2.0",
  "id": "",
  "result": {
    "check_tx": {},
    "deliver_tx": {
      "tags": [
        {
          "key": "YXBwLmNyZWF0b3I=",
          "value": "amFl"
        },
        {
          "key": "YXBwLmtleQ==",
          "value": "YWJjZA=="
        }
      ]
    },
    "hash": "9DF66553F98DE3C26E3C3317A3E4CED54F714E39",
    "height": 14
  }
}
```

トランザクションが有効であり、値が保存されていることを確認できます
クエリアプリケーション:

```sh
curl -s 'localhost:26657/abci_query?data="abcd"'
```

The result should look like:

```json
{
  "jsonrpc": "2.0",
  "id": "",
  "result": {
    "response": {
      "log": "exists",
      "index": "-1",
      "key": "YWJjZA==",
      "value": "YWJjZA=="
    }
  }
}
```

結果の `value`(` YWJjZA == `)に注意してください.これはbase64エンコーディングです.
`abcd`のASCIIコード. これは、Python2シェルで次の方法で確認できます.
`" YWJjZA == ".decode( 'base64')`を実行するか、Python3シェルで実行します
`コーデックをインポートします; codecs.decode(b" YWJjZA == "、 'base64').decode( 'ascii')`.
しばらくお待ちください[この出力をもっと作成してください
人間が読める形式](https://github.com/tendermint/tendermint/issues/1794).

次に、さまざまなキーと値を設定してみましょう.

```sh
curl -s 'localhost:26657/broadcast_tx_commit?tx="name=satoshi"'
```

ここで、 `name`を照会すると、` satoshi`または `c2F0b3NoaQ ==`を取得する必要があります.
base64の場合:

```sh
curl -s 'localhost:26657/abci_query?data="name"'
```

他のいくつかのトランザクションとクエリを試して、すべてが正常であることを確認してください
サービング！


## CounterJS-別の言語での例

また、アプリケーションを別の言語で実行する必要があります.この場合は、
Javascriptバージョンの `counter`を実行します. それを実行するには、
[ノードのインストール](https://nodejs.org/en/download/).

あなたも
[ここ](https://github.com/tendermint/js-abci)、次にインストールします:

```sh
git clone https://github.com/tendermint/js-abci.git
cd js-abci
npm install abci
```

以前の `counter`および` tendermint`プロセスを強制終了します. 次に、アプリケーションを実行します.

```sh
node example/counter.js
```

別のウィンドウで、「tendermint」をリセットして開始します.

```sh
tendermint unsafe_reset_all
tendermint start
```

もう一度、ブロックの流れを確認する必要がありますが、今では
アプリケーションはJavascriptで書かれています！ いくつかのトランザクションを送信してから
以前と同じように-結果は同じになるはずです:

```sh
# ok
curl localhost:26657/broadcast_tx_commit?tx=0x00
# invalid nonce
curl localhost:26657/broadcast_tx_commit?tx=0x05
# ok
curl localhost:26657/broadcast_tx_commit?tx=0x01
```

Neat, eh?
