# テンダーミントを使用する

これは、コマンドラインから「tendermint」プログラムを使用するためのガイドです.
テンダーミントバイナリがインストールされていることを前提としています.
TendermintとABCIとは何かに関するいくつかの基本的な概念.

`tendermint --help`を使用して、ヘルプメニューとバージョンを表示できます
ナンバー「テンダーミントバージョン」付き.

## ディレクトリルート

ブロックチェーンデータのデフォルトディレクトリは `〜/.tendermint`です. カバー
これは、 `TMHOME`環境変数を設定することで実現されます.

## 初期化

次のコマンドを実行して、ルートディレクトリを初期化します.

```sh
tendermint init validator
```

これにより、新しい秘密鍵( `priv_validator_key.json`)が作成され、
関連する公開鍵を含むジェネシスファイル( `genesis.json`)
`$ TMHOME/config`. ローカルテストネットを実行するために必要なのはこれだけです
バリデーターがあります.

初期化の詳細については、testnetコマンドを参照してください.
```sh
tendermint testnet --help
```

### 創世記

`$ TMHOME/config/`の `genesis.json`ファイルはイニシャルを定義します
ブロックチェーンの起点でのTendermintCoreの状態([参照
定義](https://github.com/tendermint/tendermint/blob/master/types/genesis.go)).

#### 分野

-`genesis_time`:ブロックチェーンが開始された公式の時刻.
-`chain_id`:ブロックチェーンのID. **これは一意である必要があります
  すべてのブロックチェーン. **テストネットブロックチェーンが一意でない場合
  チェーンID、あなたは悪い時間を過ごすでしょう. ChainIDは50シンボル未満である必要があります.
-`initial_height`:テンダーミントが開始する高さ.ブロックチェーンでネットワークのアップグレードが行われている場合は、
    ストップの高さから開始すると、前の高さに独自性がもたらされます.
-`consensus_params` [仕様](https://github.com/tendermint/spec/blob/master/spec/core/state.md#consensusparams)
    -`ブロック `
        -`max_bytes`:最大ブロックサイズ(バイト単位).
        -`max_gas`:各ブロックの最大ガス.
        -`time_iota_ms`:使用されません.これは非推奨であり、将来のバージョンで削除される予定です.
    -`証拠 `
        -`max_age_num_blocks`:証拠の最大年齢(ブロック単位).基本式
      これを計算すると、MaxAgeDuration/{平均ブロック時間}になります.
        -`max_age_duration`:証拠の最大年齢、タイムリー.対応する必要があります
      アプリケーションまたは他の同様の処理メカニズムの「バインド解除期間」を使用する
      [興味なし
      攻撃](https://github.com/ethereum/wiki/wiki/Proof-of-Stake-FAQ#what-is-the-nothing-at-stake-problem-and-how-can-it-be-fixed ).
        -`max_num`:提出できる証拠の最大数を設定します
      単一のブロックで.そして、最大のブロックの下に快適に収まるはずです
      各証拠のサイズを検討するとき.
    -`ベリファイア `
        -`pub_key_types`:オーセンティケーターが使用できる公開鍵のタイプ.
    -`バージョン `
        -`app_version`:ABCIアプリケーションのバージョン.
-`validators`:初期バリデーターリスト.これは完全にカバーされている可能性があることに注意してください
  アプリケーション、およびそれを明確にするために空白のままにすることができます
  アプリケーションはResponseInitChainを使用してバリデーターセットを初期化します.
    -`pub_key`:最初の要素は `pub_key`のタイプを指定します. 1
    == Ed25519. 2番目の要素は公開鍵バイトです.
    -`power`:検証者の投票権.
    -`name`:オーセンティケーターの名前(オプション).
-`app_hash`:期待されるアプリケーションハッシュ値(
  作成時の `ResponseInfo` ABCIメッセージ).アプリケーションのハッシュ値の場合
  一致しない場合、テンダーミントはパニックになります.
-`app_state`:アプリケーションの状態(例:初期配布
  トークン).

>:warning:** ChainIDはブロックチェーンごとに一意である必要があります.古いchainIDを再利用すると、問題が発生する可能性があります**

#### 例genesis.json

```json
{
  "genesis_time": "2020-04-21T11:17:42.341227868Z",
  "chain_id": "test-chain-ROp9KF",
  "initial_height": "0",
  "consensus_params": {
    "block": {
      "max_bytes": "22020096",
      "max_gas": "-1",
      "time_iota_ms": "1000"
    },
    "evidence": {
      "max_age_num_blocks": "100000",
      "max_age_duration": "172800000000000",
      "max_num": 50,
    },
    "validator": {
      "pub_key_types": [
        "ed25519"
      ]
    }
  },
  "validators": [
    {
      "address": "B547AB87E79F75A4A3198C57A8C2FDAF8628CB47",
      "pub_key": {
        "type": "tendermint/PubKeyEd25519",
        "value": "P/V6GHuZrb8rs/k1oBorxc6vyXMlnzhJmv7LmjELDys="
      },
      "power": "10",
      "name": ""
    }
  ],
  "app_hash": ""
}
```

## Run

Tendermintノードを実行するには、次を使用します.

```bash
tendermint start
```

デフォルトでは、TendermintはABCIアプリケーションへの接続を試みます
`127.0.0.1:26658`. `kvstore` ABCIアプリケーションをインストールしている場合は、次のWebサイトにアクセスしてください.
別のウィンドウ. そうでない場合は、Tendermintを強制終了し、次のインプロセスバージョンを実行してください.
`kvstore`アプリケーション:

```bash
tendermint start --proxy-app=kvstore
```

数秒後、ブロックが流入し始めるのが見えるはずです. 注意ブロック
取引がなくても定期的に生産されます. _NoEmptyを参照してください
この設定を変更するには、次のBlocks_を使用します.

Tendermintは、 `counter`、` kvstore`、および `noop`のインプロセスバージョンをサポートします
例として `abci-cli`を使用してリリースされたアプリケーション. アプリケーションをコンパイルするのは簡単です
TendermintがGoで書かれている場合、それは進行中です. アプリケーションが書き込みを行わない場合
移動して、別のプロセスで実行し、 `--proxy-app`フラグを使用して指定します
リッスンしているソケットのアドレス.例:

```bash
tendermint start --proxy-app=/var/run/abci.sock
```

`tendermint start --help`を実行して、サポートされているフラグを確認できます.

## トレード

トランザクションを送信するには、 `curl`を使用してTendermintRPCにリクエストを送信します
サーバー、例:

```sh
curl http://localhost:26657/broadcast_tx_commit?tx=\"abcd\"
```

`/status`エンドポイントでチェーンのステータスを確認できます.

```sh
curl http://localhost:26657/status | json_pp
```

and the `latest_app_hash` in particular:

```sh
curl http://localhost:26657/status | json_pp | grep latest_app_hash
```

他のリストを表示するには、ブラウザで `http://localhost:26657`にアクセスしてください
終点. パラメータを受け取らないもの( `/status`など)もあれば、指定するものもあります
パラメータ名とプレースホルダーとして `_`を使用します.


>ヒント:RPCドキュメントを[ここ](https://docs.tendermint.com/master/rpc/)で検索します

### フォーマット

トランザクションをRPCインターフェースに送信する場合、次のフォーマット規則が適用されます
従わなければなりません:

`GET`を使用します(URLのパラメーターを使用):

UTF8文字列をトランザクションデータとして送信するには、 `tx`の値を囲んでください
二重引用符で囲まれたパラメーター:

```sh
curl 'http://localhost:26657/broadcast_tx_commit?tx="hello"'
```

5バイトのトランザクション "h e l l o" \ [68 65 6c 6c 6f \]を送信します.

この例のURLは、防止のために一重引用符で囲まれていることに注意してください
シェルは二重引用符を解釈します. またはあなたは逃げることができます
バックスラッシュ付きの二重引用符:

```sh
curl http://localhost:26657/broadcast_tx_commit?tx=\"hello\"
```

二重引用符形式は、マルチバイト文字である限り、それらに適しています.
有効なUTF8です.例:

```sh
curl 'http://localhost:26657/broadcast_tx_commit?tx="€5"'
```

4バイトのトランザクションを送信します: "€5"(UTF8)\ [e2 82 ac 35 \].

(UTF8以外の)トランザクションデータも文字列としてエンコードできます
16進数(1バイトあたり2桁). これを行うには、引用符を省略します
そして、16進文字列の前に `0x`を追加します.

```sh
curl http://localhost:26657/broadcast_tx_commit?tx=0x68656C6C6F
```

5バイトのトランザクションを送信します:\ [68 65 6c 6c 6f \].

「POST」(JSONのパラメーターを使用)を使用して、トランザクションデータはJSONとして送信されます
Base64でエンコードされた文字列:

```sh
curl http://localhost:26657 -H 'Content-Type: application/json' --data-binary '{
  "jsonrpc": "2.0",
  "id": "anything",
  "method": "broadcast_tx_commit",
  "params": {
    "tx": "aGVsbG8="
  }
}'
```

同じ5バイトのトランザクション\ [68 65 6c 6c 6f \]を送信します.

トランザクションデータの16進エンコーディングはサポートされていないことに注意してください
JSON( `POST`)リクエスト.

## 再起動

>:warning:**安全ではない**これは開発時にのみ、可能な場合にのみ行ってください
すべてのブロックチェーンデータを失うコストを負担してください！


ブロックチェーンをリセットするには、ノードを停止して次のコマンドを実行します.
```sh
tendermint unsafe_reset_all
```

このコマンドは、データディレクトリを削除し、プライベートバリデーターをリセットします.
名簿ファイル.

## 構成

Tendermintは設定に `config.toml`を使用します. 詳細については、[
構成仕様](./configuration.md).

注目すべきオプションには、アプリケーションのソケットアドレスが含まれます
( `proxy-app`)、Tendermintピアのリスニングアドレス
( `p2p.laddr`)、およびRPCサーバーのリスニングアドレス
( `rpc.laddr`).

構成ファイルの一部のフィールドは、フラグでオーバーライドできます.

## 空のブロックはありません

`tendermint`のデフォルトの動作はまだブロックを作成することですが
約1秒に1回、空のブロックを無効にするか、
ブロックの作成間隔を設定します. 前者の場合、ブロックは
新しいトランザクションがあるか、AppHashが変更されたときに作成されます.

存在しない限り空のブロックを生成しないようにTendermintを構成します
トランザクションまたはアプリケーションのハッシュ変更.これを使用してTendermintを実行します
追加の兆候:

```sh
tendermint start --consensus.create_empty_blocks=false
```

or set the configuration via the `config.toml` file:

```toml
[consensus]
create_empty_blocks = false
```

记住:因为默认是_创建空块_，避免
空块需要将配置选项设置为 `false`.

块间隔设置允许延迟(以 time.Duration 格式 [ParseDuration](https://golang.org/pkg/time/#ParseDuration))
创建每个新的空块. 可以使用此附加标志设置它:

```sh
--consensus.create_empty_blocks_interval="5s"
```

或通过 `config.toml` 文件设置配置:

```toml
[consensus]
create_empty_blocks_interval = "5s"
```

この設定では、ブロックがない場合、5秒ごとに空のブロックが生成されます
その価値に関係なく、他の方法で生産された
`create_empty_blocks`.

## ブロードキャストAPI

以前は、 `broadcast_tx_commit`エンドポイントを使用して
トレード. トランザクションがTendermintノードに送信されると、
「CheckTx」を介してアプリケーションに対して実行します. `CheckTx`に合格すると、
メモリプールに含まれ、他のノードにブロードキャストされ、
最後にブロックに含まれます.

取引の処理には複数の段階があるため、
ブロードキャストトランザクションの複数のエンドポイント:

```md
/broadcast_tx_async
/broadcast_tx_sync
/broadcast_tx_commit
```

これらは、処理なし、メモリプールを介した処理、および
それぞれがブロックを介して処理されます.言い換えれば、 `broadcast_tx_async`、
トランザクションが成功したかどうかを聞くのを待たずにすぐに戻ります
でも機能し、 `broadcast_tx_sync`は結果を返します
「CheckTx」を介してトランザクションを実行します. `broadcast_tx_commit`を使用します
トランザクションがブロックでコミットされるまで、または一部のトランザクションがコミットされるまで待機します
タイムアウトに達しましたが、トランザクションが完了すると、すぐに戻ります
`CheckTx`を渡さないでください. `broadcast_tx_commit`の戻り値には次のものが含まれます
`check_tx`と` deliver_tx`の2つのフィールドが結果に関連しています
これらのABCIメッセージを介してトランザクションを実行します.

`broadcast_tx_commit`を使用する利点は、リクエストが
トランザクションがコミットされた後(つまり、ブロックに含まれた後)、ただし
1秒のオーダーを取ることができます.迅速な結果を得るには、
`broadcast_tx_sync`、ただしトランザクションまでコミットしません
その後、状態への影響はそれまでに変わる可能性があります.

txが渡されたという理由だけで、メモリプールは強力な保証を提供しないことに注意してください
CheckTx(つまり、メモリプールに受け入れられる)は、送信されることを意味するものではありません.
メモリプールにtxがあるノードは、推奨を行う前にクラッシュする可能性があるためです.
詳細については、[mempool
ログ先行書き込み](../tendermint-core/running-in-production.md#mempool-wal)

## テンダーミントネットワーク

`tendermint init`が実行されると、` genesis.json`と
`priv_validator_key.json`は`〜/.tendermint/config`に作成されます.この
`genesis.json`は次のようになります.

```json
{
  "validators" : [
    {
      "pub_key" : {
        "value" : "h3hk+QE8c6QLTySp8TcfzclJw/BG79ziGB/pIA+DfPE=",
        "type" : "tendermint/PubKeyEd25519"
      },
      "power" : 10,
      "name" : ""
    }
  ],
  "app_hash" : "",
  "chain_id" : "test-chain-rDlYSN",
  "genesis_time" : "0001-01-01T00:00:00Z"
}
```

And the `priv_validator_key.json`:

```json
{
  "last_step" : 0,
  "last_round" : "0",
  "address" : "B788DEDE4F50AD8BC9462DE76741CCAFF87D51E2",
  "pub_key" : {
    "value" : "h3hk+QE8c6QLTySp8TcfzclJw/BG79ziGB/pIA+DfPE=",
    "type" : "tendermint/PubKeyEd25519"
  },
  "last_height" : "0",
  "priv_key" : {
    "value" : "JPivl82x+LfVkp8i3ztoTjY6c6GJ4pBxQexErOCyhwqHeGT5ATxzpAtPJKnxNx/NyUnD8Ebv3OIYH+kgD4N88Q==",
    "type" : "tendermint/PrivKeyEd25519"
  }
}
```

`priv_validator_key.json`には実際には秘密鍵が含まれているため、
したがって、これは完全に機密です.現在はプレーンテキストを使用しています.
署名できないように、 `last_`フィールドに注意してください
相反するニュース.

また、 `pub_key`(公開鍵)が
`priv_validator_key.json`は` genesis.json`にも存在します.

ジェネシスファイルには、参加する可能性のある公開鍵のリストが含まれています
コンセンサス、およびそれらに対応する議決権. 2/3より大きい
の議決権は有効である必要があります(つまり、対応する秘密鍵
進展を遂げるためには、合意に達するために署名を生成する必要があります.私たちの中で
この場合、ジェネシスファイルには公開鍵が含まれています
`priv_validator_key.json`であるため、Tendermintノードはデフォルト値で開始します
ルートディレクトリを入力できます.議決権はint64を使用します
ただし、正の数である必要があるため、範囲は0〜9223372036854775807です.
現在の提案者はアルゴリズムの動作方法を選択するため、
議決権は10 \ ^ 12(つまり、1兆)を超えることをお勧めします.

ネットワークにノードを追加する場合は、2つのオプションがあります.
新しいバリデーターノードを追加します.これも次の方法でコンセンサスに参加します
ブロックを提案して投票するか、新しい非バリデーターを追加できます
ノード、直接参加しませんが、検証して維持します
コンセンサスとの合意.

### ピア

#### シード

シードノードは、知っている他のピアのアドレスを中継するノードです.
の.これらのノードは、より多くのピアを取得しようとして、ネットワークを継続的にクロールします.この
シードノードによって中継されたアドレスは、ローカルアドレスブックに保存されます.一度
これらはアドレスブックにあり、これらのアドレスに直接接続されます.
基本的に、シードノードの仕事は全員のアドレスを中継することです.あなたはしません
十分なアドレスを受け取った後、シードノードに接続するため、通常は
これらは最初の起動時にのみ必要です.シードノードはすぐに切断されます
あなたにいくつかのアドレスを送った後あなたから.

####永続的なピア

パーマネントピアは、連絡を取り合いたい相手です.もし、あんたが
切断します.使用する代わりに、直接接続を試みます.
名簿の別の住所.再起動時に、常に試してみます
名簿のサイズに関係なく、これらのピアに接続できます.

デフォルトでは、すべてのピアが知っているピアを中継します.これはピアツーピア交換と呼ばれます
合意(PeX). PeXを使用すると、ピアはゴシップの既知のピアを形成します
addrbookにピアアドレスを格納するネットワーク.このため、あなたはしません
リアルタイムの永続ピアがある場合は、シードノードを使用する必要があります.

####ピアに接続する

起動時にピアに接続するには、
`$ TMHOME/config/config.toml`またはコマンドライン. 「シード」を使用して
シードノードを指定し、
`persistent-peers`は、ノードが維持するピアを指定します
との持続的接続.

例えば、

```sh
tendermint start --p2p.seeds "f9baeaa15fedf5e1ef7448dd60f46c01f1a9e9c4@1.2.3.4:26656,0491d373a8e0fcf1023aaf18c51d6a1d0d4f31bd@5.6.7.8:26656"
```

または、RPC `/dial_seeds`エンドポイントを使用して
実行中のノードの接続先のシードを指定します.

```sh
curl 'localhost:26657/dial_seeds?seeds=\["f9baeaa15fedf5e1ef7448dd60f46c01f1a9e9c4@1.2.3.4:26656","0491d373a8e0fcf1023aaf18c51d6a1d0d4f31bd@5.6.7.8:26656"\]'
```

PeXを有効にした後、
最初の起動後にシードは必要ありません.

Tendermintを特定のアドレスのセットに接続する場合
それぞれとの永続的な接続を維持するために、次を使用できます
`--p2p.persistent-peers`フラグまたは対応する設定
`config.toml`または`/dial_peers`RPCエンドポイントは
Tendermintコアインスタンスを停止します.

```sh
tendermint start --p2p.persistent-peers "429fcf25974313b95673f58d77eacdd434402665@10.11.12.13:26656,96663a3dd0d7b9d17d4c8211b191af259621c693@10.11.12.14:26656"

curl 'localhost:26657/dial_peers?persistent=true&peers=\["429fcf25974313b95673f58d77eacdd434402665@10.11.12.13:26656","96663a3dd0d7b9d17d4c8211b191af259621c693@10.11.12.14:26656"\]'
```

### 非バリデーターを追加する

非バリデーターの追加は簡単です. 元の `genesis.json`をコピーするだけです
新しいマシンで `〜/.tendermint/config`に移動し、ノードを起動します.
必要に応じてシードまたは永続ピアを指定します. シードがない場合または
永続ピアが指定されている場合、ノードはブロックを生成しません.
バリデーターではなく、ブロックニュースは聞こえません.
他のピアに接続されていません.

### バリデーターを追加

新しいバリデーターを追加する最も簡単な方法は、 `genesis.json`で追加することです.
ネットワークを開始する前. たとえば、新しいものを作成できます
`priv_validator_key.json`を作成し、その` pub_key`を上記のジェネシスにコピーします.

次のコマンドを使用して、新しい `priv_validator_key.json`を生成できます.

```sh
tendermint gen_validator
```

これで、ジェネシスファイルを更新できます. たとえば、新しい場合
`priv_validator_key.json`は次のようになります.

```json
{
  "address" : "5AF49D2A2D4F5AD4C7C8C4CC2FB020131E9C4902",
  "pub_key" : {
    "value" : "l9X9+fjkeBzDfPGbUM7AMIRE6uJN78zN5+lk5OYotek=",
    "type" : "tendermint/PubKeyEd25519"
  },
  "priv_key" : {
    "value" : "EDJY9W6zlAw+su6ITgTKg2nTZcHAH1NMTW5iwlgmNDuX1f35+OR4HMN88ZtQzsAwhETq4k3vzM3n6WTk5ii16Q==",
    "type" : "tendermint/PrivKeyEd25519"
  },
  "last_step" : 0,
  "last_round" : "0",
  "last_height" : "0"
}
```

次に、新しいgenesis.jsonは次のようになります.

```json
{
  "validators" : [
    {
      "pub_key" : {
        "value" : "h3hk+QE8c6QLTySp8TcfzclJw/BG79ziGB/pIA+DfPE=",
        "type" : "tendermint/PubKeyEd25519"
      },
      "power" : 10,
      "name" : ""
    },
    {
      "pub_key" : {
        "value" : "l9X9+fjkeBzDfPGbUM7AMIRE6uJN78zN5+lk5OYotek=",
        "type" : "tendermint/PubKeyEd25519"
      },
      "power" : 10,
      "name" : ""
    }
  ],
  "app_hash" : "",
  "chain_id" : "test-chain-rDlYSN",
  "genesis_time" : "0001-01-01T00:00:00Z"
}
```

`〜/.tendermint/config`の` genesis.json`を更新します.コピー元
ファイルと新しい `priv_validator_key.json`を`〜/.tendermint/config`に
新しいマシン.

次に、両方のマシンで `tendermint start`を実行し、どちらかを使用します
`--p2p.persistent-peers`または`/dial_peers`を使用してピアにします.
彼らはブロックを作り始めるべきであり、そうし続けるだけです
それらはすべてオンラインだからです.

バリデーターの1つに耐えることができるTendermintネットワークを作成します
失敗するには、少なくとも4つのバリデーターノード(たとえば、2/3)が必要です.

リアルタイムネットワークでバリデーターを更新するためのサポートが必要ですが、
アプリケーション開発者によって明確にプログラムされています.

### 地元のネットワーク

マシンなどでネットワークをローカルで実行するには、 `_laddr`を変更する必要があります
監視用の `config.toml`のフィールド(またはフラグを使用)
さまざまなソケットのアドレスは競合しません.さらに、設定する必要があります
`config.toml`の` addr_book_strict = false`、それ以外の場合はTendermintのp2p
ライブラリは、同じIPアドレスを持つピアとの接続の確立を拒否します.

### アップグレード

見る
[Upgrade.md](https://github.com/tendermint/tendermint/blob/master/UPGRADING.md)
ガイド.主要な画期的なバージョン間でチェーンをリセットする必要があるかもしれません.
ただし、Tendermintの画期的なバージョンは将来的に少なくなると予想されます.
(特にバージョン1.0以降)./
